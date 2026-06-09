🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.2 — Informations système

> 🩺 Les premiers réflexes pour observer un serveur en marche : session, connexions, moteurs et métriques.

Cette section regroupe les commandes d'inspection que l'on dégaine en premier face à un serveur à diagnostiquer : obtenir un instantané de sa session, voir qui est connecté et ce qui s'exécute, plonger dans l'état interne d'un moteur, et distinguer les métriques live de la configuration. Toutes ces commandes sont en lecture seule : elles observent sans rien modifier.

---

## Instantané de la session : `STATUS` (`\s`)

Déjà rencontrée en B.1, l'instruction `STATUS` (ou sa forme client `\s`) fournit un récapitulatif condensé de la session et du serveur : versions du client et du serveur, identifiant de connexion, utilisateur, base courante, mode de connexion (socket ou TCP), jeux de caractères, et quelques compteurs globaux.

```text
Connection id:        42
Current database:     boutique
Current user:         app_user@localhost
Server version:       12.3.x-MariaDB
Connection:           Localhost via UNIX socket
Server characterset:  utf8mb4
Uptime:               3 days 4 hours 12 min
Threads: 5   Questions: 128943   Slow queries: 2   Opens: 80   Queries per second avg: 0.41
```

Les champs `Uptime`, `Threads` (connexions actives), `Questions` (requêtes traitées depuis le démarrage) et `Slow queries` donnent une première idée de la charge. Pour aller au-delà de cet aperçu, on passe aux commandes ci-dessous.

## Connexions et requêtes en cours : `SHOW PROCESSLIST`

`SHOW PROCESSLIST` liste les threads (connexions) du serveur, ce qui en fait l'outil de référence pour repérer une requête bloquée ou trop longue.

```sql
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;   -- texte complet des requêtes (sinon tronqué à ~100 caractères)
```

Les colonnes principales sont `Id` (identifiant du thread), `User`, `Host`, `db`, `Command` (état général : `Query`, `Sleep`…), `Time` (durée en secondes dans l'état courant), `State` (étape interne) et `Info` (la requête en cours). MariaDB ajoute une colonne `Progress` pour certaines opérations longues.

Pour filtrer et trier — par exemple isoler les requêtes actives les plus longues —, mieux vaut interroger la vue correspondante d'`INFORMATION_SCHEMA` plutôt que la version brute :

```sql
SELECT id, user, host, db, command, time, state, info
FROM   information_schema.PROCESSLIST
WHERE  command <> 'Sleep'
ORDER  BY time DESC;
```

Une fois le coupable identifié, deux commandes permettent d'intervenir : `KILL <id>` ferme la connexion, tandis que `KILL QUERY <id>` interrompt uniquement la requête en cours en laissant la connexion ouverte.

```sql
KILL QUERY 42;   -- annule la requête du thread 42
KILL 42;         -- ferme la connexion 42
```

*Voir Ch. 6 pour les verrous et deadlocks, et l'Annexe C.1 pour des requêtes d'administration prêtes à l'emploi (locks, processlist).*

## État interne d'un moteur : `SHOW ENGINE`

Au singulier, `SHOW ENGINE` détaille le fonctionnement interne d'un moteur. La variante la plus précieuse concerne InnoDB :

```sql
SHOW ENGINE INNODB STATUS\G
```

Le rapport, volumineux, se lit par sections. Les plus actionnables :

- **TRANSACTIONS** — transactions actives et attentes de verrous (rechercher les mentions de type *waiting for this lock*), pour diagnostiquer la contention.
- **LATEST DETECTED DEADLOCK** — détails du dernier interblocage détecté : transactions impliquées, verrous en cause et victime choisie.
- **BUFFER POOL AND MEMORY** — taille du buffer pool, pages libres et taux de succès (*buffer pool hit rate*), révélateur de la pression mémoire.
- **FILE I/O**, **LOG**, **ROW OPERATIONS**, **SEMAPHORES** — activité disque, journaux et statistiques d'exécution.

Au pluriel, `SHOW ENGINES` liste les moteurs disponibles et leur statut de prise en charge :

```sql
SHOW ENGINES;
```

| Engine | Support | Transactions | XA | Savepoints |
|--------|---------|--------------|----|------------|
| InnoDB | DEFAULT | YES | YES | YES |
| Aria | YES | NO | NO | NO |
| MyISAM | YES | NO | NO | NO |
| MEMORY | YES | NO | NO | NO |
| CSV | YES | NO | NO | NO |

La colonne `Support` indique notamment le moteur par défaut (`DEFAULT`) et les moteurs présents mais désactivés (`DISABLED`). *Voir Ch. 7 pour le détail des moteurs et de leurs structures internes (buffer pool, redo/undo log).*

## Compteurs et configuration : `SHOW STATUS` et `SHOW VARIABLES`

Ces deux commandes se ressemblent mais répondent à des questions opposées, et les confondre est une source d'erreurs fréquente.

`SHOW STATUS` expose des **compteurs et états live**, en lecture seule, qui évoluent au fil de l'activité (nombre de connexions, requêtes lentes, lectures du buffer pool…). `SHOW VARIABLES` expose la **configuration** du serveur (taille du buffer pool, nombre maximal de connexions…), dont une grande partie est modifiable par `SET`.

L'un comme l'autre acceptent une portée `GLOBAL` (tout le serveur) ou `SESSION` (la connexion courante), ainsi qu'un filtre `LIKE` :

```sql
-- Métriques (compteurs)
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL STATUS LIKE 'Aborted_connects';

-- Configuration (réglages)
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

On peut aussi lire une variable ponctuelle via la syntaxe `@@`, pratique dans une expression :

```sql
SELECT @@hostname, @@version, @@global.max_connections;
```

*Voir Ch. 11.2 pour les variables système et leur portée, et Ch. 11.9 / Ch. 15 pour le choix des métriques à surveiller.*

## Autres commandes utiles

`SHOW WARNINGS` et `SHOW ERRORS` affichent les avertissements ou erreurs de la dernière instruction — utiles juste après un import ou une conversion. `SHOW PLUGINS` liste les greffons chargés (moteurs, authentification, audit). Enfin, l'état de la réplication ne se consulte pas ici mais via `SHOW REPLICA STATUS` (*voir Ch. 13*).

## Pour aller plus loin

Cet aide-mémoire couvre l'inspection ponctuelle depuis le client. Pour une observation continue (tableaux de bord, alerting), la formation s'appuie sur `PERFORMANCE_SCHEMA`, le slow query log et un empilement Prometheus/Grafana, traités au Ch. 15 et au Ch. 16. Des requêtes de monitoring prêtes à copier figurent à l'Annexe C.2.

---

⬅️ [B.1 — Connexion et navigation](01-connexion-navigation.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [B.3 — Export/Import](03-export-import.md)

⏭️ [Export/Import (SOURCE avec --script-dir, TEE, mariadb-dump)](/annexes/b-commandes-cli/03-export-import.md)

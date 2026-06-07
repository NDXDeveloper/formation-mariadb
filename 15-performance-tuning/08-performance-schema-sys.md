🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.8 Performance Schema et sys schema

Le Performance Schema **instrumente le fonctionnement interne du serveur** — requêtes, attentes, étapes d'exécution, E/S, verrous, mémoire — en mémoire et à faible coût. Il complète le *slow query log* (§15.7) par une vue vivante, continue et agrégée. Le **sys schema**, intégré à MariaDB depuis la 10.6, rend ces données directement lisibles. Cette section se concentre sur l'**usage** de ces deux outils pour l'optimisation ; pour la référence structurelle du Performance Schema en tant que schéma système (liste des tables, etc.), on se reportera au §9.7.2.

---

## Le Performance Schema : un instrument intégré

Le Performance Schema est une base de données (`performance_schema`) dont les tables, interrogeables en SQL ordinaire, exposent des informations détaillées sur l'activité du serveur : instructions exécutées, événements d'attente, étapes d'exécution, E/S sur fichiers et sur tables, verrous (de métadonnées comme de table), allocations mémoire, threads, etc. Depuis MariaDB 10.5, il comporte environ 80 tables (cf. §9.7.2 pour la liste).

Par rapport au *slow query log* (§15.7), l'approche est différente et complémentaire : là où le journal est un fichier rétrospectif, le Performance Schema agrège les statistiques **en mémoire, en continu et à faible surcoût**, et capture **toutes** les requêtes — y compris les rapides — ce qui en fait l'outil idéal pour un classement par coût agrégé.

---

## Activer le Performance Schema en MariaDB

Deux particularités propres à MariaDB sont à connaître.

D'abord, le Performance Schema est **désactivé par défaut** (contrairement à MySQL, où il est actif), pour des raisons de performance. Ensuite, c'est une **variable statique** : elle ne peut pas être activée à chaud et doit être positionnée au démarrage. On l'active donc dans le fichier de configuration, puis on redémarre le serveur :

```ini
[mariadb]
performance_schema = ON
```

On vérifie son état avec :

```sql
SHOW VARIABLES LIKE 'performance_schema';
```

Le dimensionnement de ses tables est réglé par les variables `performance_schema_*` (par exemple `performance_schema_digests_size` pour la table des empreintes, cf. §15.7.1) : la valeur `-1`, par défaut, active un **dimensionnement automatique**, et `0` désactive la table concernée.

Une fois le Performance Schema actif, l'instrumentation fine s'active ou se désactive **à chaud** via les tables de configuration `setup_instruments` et `setup_consumers` — ce qui permet de n'instrumenter que ce dont on a besoin et de limiter le surcoût :

```sql
UPDATE performance_schema.setup_consumers  SET ENABLED = 'YES';  
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';  
```

Les tables `setup_actors` et `setup_objects` permettent en outre de filtrer l'instrumentation par utilisateur ou par objet.

---

## Le défi : des données brutes

Le Performance Schema est puissant, mais ses tables sont **peu lisibles** telles quelles : les durées y sont exprimées en picosecondes, les noms d'événements sont techniques, et les colonnes nombreuses. Interroger directement `events_statements_summary_by_digest` ou `table_io_waits_summary_by_index_usage` est fastidieux. C'est précisément le problème que résout le sys schema.

---

## Le sys schema : la couche lisible

Le sys schema est un ensemble de **vues, fonctions et procédures** construites au-dessus du Performance Schema et d'INFORMATION_SCHEMA, conçu pour offrir un accès intuitif aux DBA et aux équipes DevOps. **Intégré à MariaDB depuis la 10.6** (auparavant, il fallait l'installer manuellement), il est disponible d'office en 12.3. Comme ses vues lisent les tables du Performance Schema, il suppose que ce dernier est activé.

Il apporte trois choses :

- des **fonctions de formatage** qui transforment les valeurs brutes en valeurs lisibles : `format_bytes()`, `format_time()` (côté MySQL 8.0, cette fonction a été renommée `format_pico_time()` — mais MariaDB conserve le nom `format_time`), `format_path()` ;
- des **vues** en deux variantes : une forme lisible (par exemple `statement_analysis`) et une forme brute préfixée par `x$` (par exemple `x$statement_analysis`), destinée au traitement automatisé ;
- des **procédures** utilitaires, notamment la famille `ps_setup_*` pour activer ou configurer facilement l'instrumentation, et `ps_truncate_all_tables()` pour réinitialiser les statistiques.

---

## Cas d'usage pour le tuning

L'intérêt pratique du couple Performance Schema / sys schema est de répondre directement à la plupart des questions d'optimisation. Le tableau suivant associe une question à la vue (ou table) appropriée :

| Question de tuning | Vue `sys` (ou table Performance Schema) |
|---|---|
| Requêtes les plus coûteuses | `sys.statement_analysis` |
| Requêtes faisant un balayage complet | `sys.statements_with_full_table_scans` |
| Requêtes avec tri sur disque (*filesort*) | `sys.statements_with_sorting` |
| Requêtes avec tables temporaires | `sys.statements_with_temp_tables` |
| Pires latences (95ᵉ centile) | `sys.statements_with_runtimes_in_95th_percentile` |
| Index jamais utilisés | `sys.schema_unused_indexes` |
| Index redondants | `sys.schema_redundant_indexes` |
| Tables les plus sollicitées | `sys.schema_table_statistics` |
| E/S par fichier | `sys.io_global_by_file_by_bytes` |
| Attentes les plus longues | `sys.waits_global_by_latency` |

Quelques usages méritent d'être détaillés.

**Identifier les requêtes coûteuses.** La table `events_statements_summary_by_digest`, exposée de façon lisible par `sys.statement_analysis` (requête, base, nombre d'exécutions, latence totale et moyenne, lignes examinées et renvoyées…), classe les requêtes par coût agrégé. C'est l'équivalent intégré et à faible surcoût du couple *slow query log* + `pt-query-digest` (§15.7).

**Repérer les requêtes problématiques par symptôme.** Les vues `sys.statements_with_full_table_scans`, `sys.statements_with_sorting` et `sys.statements_with_temp_tables` (avec le pourcentage de tables temporaires basculées sur disque) orientent directement vers les leviers de correction — le plus souvent l'indexation (§5.5).

**Auditer les index.** La vue `sys.schema_unused_indexes` liste les index pour lesquels aucun événement de lecture n'a été enregistré, donc candidats à la suppression — sachant que ce résultat n'est significatif qu'après une période d'activité représentative. `sys.schema_redundant_indexes` repère les index doublons. C'est une étape clé de l'audit de schéma (cf. §5 et l'annexe E.2) : un index inutilisé ou redondant gaspille de l'espace et **ralentit les écritures**, puisqu'il doit être maintenu à chaque modification.

**Analyser les E/S et la contention.** `sys.schema_table_statistics` et `sys.io_global_by_file_by_bytes` mettent en évidence les tables et fichiers les plus sollicités (à rapprocher du §15.4), tandis que `sys.waits_global_by_latency` et l'instrumentation des verrous (de métadonnées et de table) aident à diagnostiquer la contention (cf. chapitre 6).

**Examiner les requêtes récentes ou en cours.** Les tables `events_statements_current` et `events_statements_history` (que l'on relie à une connexion via la table `threads`) montrent ce qu'exécute une session à l'instant *t* ou vient d'exécuter — un complément utile à `SHOW PROCESSLIST` (§15.7).

---

## Surcoût et bonnes pratiques

Le surcoût du Performance Schema est minime et n'est généralement perceptible que sur des serveurs à très fort débit (plusieurs centaines de requêtes par seconde). On le maîtrise en n'activant que les instruments et consommateurs nécessaires (via les tables `setup_*`) et en dimensionnant ses tables.

Comme son activation impose un redémarrage, elle se planifie ; mais de nombreux serveurs de production le laissent **actif en permanence**, son coût modéré étant préférable à un *slow query log* à seuil bas maintenu en continu, lorsqu'on recherche une observabilité durable (§15.7). Enfin, avant une fenêtre de mesure, on peut réinitialiser les statistiques avec un `TRUNCATE` sur les tables de synthèse concernées, ou via `sys.ps_truncate_all_tables()`.

---

## Points clés à retenir

- Le **Performance Schema** instrumente le serveur en mémoire, à faible coût, et capture **toutes** les requêtes — complément vivant du *slow query log* (§15.7) ; voir §9.7.2 pour sa référence structurelle.
- En MariaDB, il est **désactivé par défaut** et **statique** : on l'active via `performance_schema=ON` dans `my.cnf` **avec redémarrage**, l'instrumentation fine se réglant ensuite à chaud par `setup_instruments` / `setup_consumers`.
- Le **sys schema** (intégré depuis la **10.6**) rend ces données lisibles via des **vues** (`statement_analysis`, `schema_unused_indexes`…), des **fonctions de formatage** (`format_bytes`, `format_time`) et des **procédures** (`ps_setup_*`).
- Il répond directement aux questions de tuning : requêtes coûteuses, balayages complets, tris et tables temporaires, **index inutilisés ou redondants**, tables chaudes, E/S, attentes.
- Le surcoût est minime ; on instrumente sélectivement, et l'on peut laisser le Performance Schema **actif en permanence** pour une observabilité continue, préférable à un *slow query log* à seuil bas permanent.

⏭️ [Partitionnement de tables](/15-performance-tuning/09-partitionnement-tables.md)

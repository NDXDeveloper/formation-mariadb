🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.9 — Monitoring et métriques importantes

Superviser un serveur MariaDB ne consiste pas à regarder des chiffres après coup, mais à **détecter les problèmes avant qu'ils ne deviennent des incidents**, à planifier la capacité et à disposer des éléments de diagnostic quand une lenteur survient. Cette discipline repose sur trois activités complémentaires : **collecter** les métriques, établir une **ligne de référence** (baseline) du comportement normal, et **alerter** sur les écarts significatifs.

Cette section présente les **sources** de métriques offertes par MariaDB et les **métriques essentielles** à surveiller, organisées par domaine. La mise en place d'une chaîne de collecte et de tableaux de bord (Prometheus/Grafana) est traitée au chapitre [16.9](../16-devops-automatisation/09-monitoring-prometheus-grafana.md), l'observabilité au sens large en [16.10](../16-devops-automatisation/10-observabilite.md), et l'alerting en [16.11](../16-devops-automatisation/11-alerting-incident-response.md). Les requêtes SQL de monitoring prêtes à l'emploi figurent en [Annexe C.2](../annexes/c-requetes-sql-reference/02-requetes-monitoring.md).

---

## Les sources de métriques dans MariaDB

MariaDB expose son état interne à travers plusieurs interfaces, du plus simple au plus détaillé :

- **Variables d'état** (`SHOW [GLOBAL] STATUS`) : les compteurs et jauges fondamentaux du serveur. C'est la source la plus utilisée pour le monitoring continu.
- **Variables système** (`SHOW [GLOBAL] VARIABLES`) : la configuration, indispensable pour **contextualiser** les métriques (par exemple comparer `Max_used_connections` à `max_connections`).
- **`SHOW ENGINE INNODB STATUS`** : une « radiographie » détaillée du moteur InnoDB (verrous, transactions, buffer pool, journaux).
- **`SHOW [FULL] PROCESSLIST`** et `INFORMATION_SCHEMA.PROCESSLIST` : l'activité en cours, session par session.
- **`INFORMATION_SCHEMA`** : tailles des tables, `INNODB_METRICS`, tablespaces, etc. (voir [9.7.1](../09-vues-et-donnees-virtuelles/07.1-information-schema.md)).
- **`PERFORMANCE_SCHEMA`** : l'instrumentation bas niveau des événements (requêtes, attentes, étapes). Implémentée comme un moteur de stockage, elle est **désactivée par défaut** et s'active au démarrage (voir [9.7.2](../09-vues-et-donnees-virtuelles/07.2-performance-schema.md)).
- **Schéma `sys`** : des vues lisibles construites au-dessus de `PERFORMANCE_SCHEMA` et d'`INFORMATION_SCHEMA`, disponibles depuis MariaDB 10.6 (donc présentes en 12.3).
- **Niveau système d'exploitation** : CPU, mémoire, I/O disque et **espace disque** (voir [11.7](07-gestion-espace-disque.md)). Aucun monitoring de base de données n'est complet sans la supervision de l'OS sous-jacent.

---

## Compteurs cumulatifs ou jauges : lire correctement les variables d'état

C'est l'erreur la plus fréquente en monitoring MariaDB, et elle invalide toute interprétation si on l'ignore : **la plupart des variables d'état sont des compteurs cumulatifs** depuis le démarrage du serveur (ou la dernière remise à zéro via `FLUSH STATUS`). Leur valeur brute, prise isolément, ne signifie rien.

Ce qui compte est le **taux**, c'est-à-dire la variation (delta) sur un intervalle de temps. Par exemple, `Questions` totalise toutes les requêtes depuis le démarrage ; le débit (requêtes par seconde) se calcule comme `Δ(Questions) / Δ(temps)` entre deux relevés. Un serveur affichant `Questions = 5 000 000 000` n'apprend rien ; savoir qu'il traite **3 000 requêtes par seconde en moyenne** est exploitable.

À l'inverse, certaines variables sont des **jauges**, qui reflètent un état instantané : `Threads_connected`, `Threads_running`, `Innodb_buffer_pool_pages_free`. Celles-ci se lisent directement.

Cette distinction est précisément ce qu'un système de collecte time-series (chapitre [16.9](../16-devops-automatisation/09-monitoring-prometheus-grafana.md)) gère automatiquement en calculant les taux. C'est aussi la raison pour laquelle on ne doit **pas** piloter la supervision de production par des `SELECT` manuels ponctuels : ils ne donnent qu'un instantané sans tendance.

---

## Les métriques essentielles par domaine

### Disponibilité et connexions

`Uptime` indique la durée de fonctionnement depuis le dernier démarrage ; une chute signale un redémarrage (planifié ou crash). `Threads_connected` (jauge) compte les connexions ouvertes, mais le vrai signal de charge est `Threads_running`, le nombre de connexions **en train d'exécuter** une requête : une montée soudaine traduit une saturation ou un blocage. `Max_used_connections`, comparé à `max_connections`, révèle si l'on approche de la limite de connexions.

Côté anomalies, `Aborted_connects` compte les tentatives de connexion **échouées** (authentification, réseau, TLS), tandis que `Aborted_clients` compte les connexions **interrompues** parce que le client ne s'est pas déconnecté proprement. Enfin, `Threads_created` rapproché du nombre de connexions mesure l'efficacité du cache de threads (`thread_cache_size`) — à interpréter différemment lorsque le Thread Pool est actif (voir [11.10](10-thread-pool.md)).

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN
  ('Uptime', 'Threads_connected', 'Threads_running', 'Max_used_connections',
   'Aborted_connects', 'Aborted_clients', 'Threads_created');
```

### Débit (throughput)

`Questions` et `Queries` totalisent les requêtes reçues ; leur delta donne le débit. Les compteurs `Com_select`, `Com_insert`, `Com_update` et `Com_delete` détaillent la **nature** de la charge (proportion lecture/écriture), une information clé pour le dimensionnement et le choix d'architecture. `Slow_queries` compte les requêtes ayant dépassé `long_query_time` : c'est un indicateur de premier ordre, à analyser ensuite via le slow query log ([15.7](../15-performance-tuning/07-analyse-requetes-lentes.md)).

### Buffer pool InnoDB

C'est le domaine le plus déterminant pour les performances d'un serveur InnoDB. Le **taux de hit** du buffer pool mesure la part des lectures servies depuis la mémoire plutôt que depuis le disque. Il se calcule à partir de `Innodb_buffer_pool_read_requests` (lectures logiques) et `Innodb_buffer_pool_reads` (lectures physiques sur disque) :

```sql
SELECT ROUND(
  (1 - (
    (SELECT variable_value FROM information_schema.global_status
       WHERE variable_name = 'Innodb_buffer_pool_reads')
    /
    (SELECT variable_value FROM information_schema.global_status
       WHERE variable_name = 'Innodb_buffer_pool_read_requests')
  )) * 100, 4) AS buffer_pool_hit_ratio_pct;
```

Sur une charge bien dimensionnée, ce ratio doit être **très élevé** (de l'ordre de 99,9 %). Un ratio qui se dégrade durablement signale un buffer pool sous-dimensionné par rapport au jeu de données actif. Deux autres signaux complètent le tableau : `Innodb_buffer_pool_pages_free` / `_dirty` / `_total` renseignent sur l'occupation, et surtout `Innodb_buffer_pool_wait_free` doit rester à **zéro** — toute valeur positive indique que des requêtes ont attendu qu'une page se libère, symptôme d'un buffer pool trop petit ou d'un flush qui ne suit pas. Le dimensionnement est traité en [15.2.1](../15-performance-tuning/02.1-innodb-buffer-pool.md).

### I/O et journalisation InnoDB

`Innodb_data_reads` / `_writes` et `Innodb_data_read` / `_written` quantifient l'activité disque d'InnoDB. `Innodb_os_log_written` mesure le volume écrit dans le redo log. `Innodb_log_waits` doit rester à **zéro** : une valeur positive révèle un `innodb_log_buffer_size` insuffisant. Du côté de la concurrence, `Innodb_row_lock_waits`, `Innodb_row_lock_time` et `Innodb_row_lock_current_waits` mesurent la contention sur les verrous de ligne — une contention élevée renvoie à l'analyse des verrous et des deadlocks ([6.4](../06-transactions-et-concurrence/04-verrous.md), [6.5](../06-transactions-et-concurrence/05-deadlocks-resolution.md)).

### Tables temporaires et tris

`Created_tmp_tables` compte les tables temporaires internes créées **en mémoire**, ce qui est normal. Le signal à surveiller est `Created_tmp_disk_tables` : les tables temporaires ayant **débordé sur disque**, faute de tenir dans `tmp_memory_table_size`. Un taux élevé de tables temporaires sur disque trahit des requêtes mal optimisées ou un plafond mémoire trop bas. Côté tris, `Sort_merge_passes` doit rester faible — une valeur élevée indique un `sort_buffer_size` insuffisant ou des tris non indexés.

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN
  ('Created_tmp_tables', 'Created_tmp_disk_tables', 'Created_tmp_files',
   'Sort_merge_passes', 'Sort_scan', 'Sort_rows');
```

Ces métriques se complètent des variables d'état `tmp_space_used` et `max_tmp_space_used`, ainsi que de la colonne `TMP_SPACE_USED` d'`INFORMATION_SCHEMA.PROCESSLIST`, qui permettent de suivre et de borner l'espace disque temporaire (voir la section [11.8](08-controle-espace-temporaire.md)).

### Utilisation des index et balayages complets

Le rapport entre `Handler_read_rnd_next` (lecture séquentielle de la ligne suivante, typique d'un balayage de table) et `Handler_read_key` (lecture via un index) donne une indication globale sur l'usage des index : une prédominance de `Handler_read_rnd_next` suggère de nombreux balayages complets et donc des index manquants. Plus précis encore, `Select_scan` compte les requêtes qui balaient entièrement la première table, et `Select_full_join` les jointures effectuées **sans index** — cette dernière valeur devrait rester proche de zéro, car une jointure sans index est l'une des causes les plus sérieuses de lenteur. L'identification des requêtes fautives se fait ensuite via le schéma `sys` (plus bas) et le chapitre [5](../05-index-et-performance/README.md).

### Verrous au niveau table

`Table_locks_waited` rapporté à `Table_locks_immediate` mesure la contention sur les verrous de **table** (principalement MyISAM/Aria). Pour InnoDB, qui pratique le verrouillage au niveau ligne, on s'appuie sur les compteurs `Innodb_row_lock_*` vus plus haut et sur `SHOW ENGINE INNODB STATUS`.

### Cache de tables et fichiers ouverts

Un taux de croissance élevé de `Opened_tables` par rapport à `Open_tables` indique un `table_open_cache` sous-dimensionné : le serveur rouvre sans cesse des descripteurs de table. `Open_files` permet de surveiller la consommation de descripteurs de fichiers au regard de la limite système.

### Journal binaire et réplication

`Binlog_cache_use` et `Binlog_cache_disk_use` mesurent l'usage du cache de binlog par transaction : un `Binlog_cache_disk_use` fréquemment positif signale un `binlog_cache_size` trop petit. Pour la réplication, le retard du réplica (`Seconds_Behind_Master`, et la sortie de `SHOW REPLICA STATUS`) est la métrique critique, traitée en détail en [13.7](../13-replication/07-monitoring-troubleshooting.md).

> 🔔 **Note 12.3** : avec le nouveau journal binaire intégré à InnoDB (fichiers `.ibb`, voir [11.5.4](05.4-binlog-innodb-performance.md)), l'instrumentation dans `performance_schema` n'est pas encore complète — les tables d'instances de fichiers ne reflètent pas fidèlement ces fichiers. Par ailleurs, `sync_binlog` est silencieusement ignoré avec ce format : il ne doit donc **pas** être utilisé comme indicateur de durabilité du binlog dans la supervision.

---

## `SHOW ENGINE INNODB STATUS` : la radiographie d'InnoDB

Cette commande fournit un cliché détaillé du moteur, irremplaçable pour le diagnostic ponctuel. On y trouve notamment : les attentes sur sémaphores et mutex (contention interne), le **dernier deadlock** détecté avec les transactions impliquées, la liste des transactions actives, l'état du buffer pool et de la mémoire, le détail des opérations sur les lignes, ainsi que l'état des journaux et du checkpoint. La **history list length** y figure également : une valeur qui croît durablement signale une purge en retard, souvent due à une transaction restée ouverte, avec un impact direct sur la taille des logs undo (voir [11.7](07-gestion-espace-disque.md)).

On la consulte typiquement lors d'un incident (contention, deadlock, ralentissement soudain), en complément des compteurs continus.

---

## Performance Schema et schéma `sys`

`PERFORMANCE_SCHEMA` instrumente finement les événements internes (requêtes, attentes, étapes d'exécution). Désactivé par défaut, il s'active au démarrage par `performance_schema=ON` ; son surcoût est modéré et il permet une instrumentation **sélective** (n'activer que les instruments utiles). Sa fonctionnalité la plus précieuse pour le DBA est l'**agrégation des requêtes par empreinte** (digests), qui révèle les *familles* de requêtes les plus coûteuses, indépendamment de leurs valeurs littérales :

```sql
SELECT
    digest_text                              AS empreinte,
    count_star                               AS executions,
    ROUND(sum_timer_wait / 1e12, 2)          AS total_s,
    ROUND(avg_timer_wait / 1e9, 2)           AS moyenne_ms,
    sum_rows_examined                        AS lignes_examinees,
    sum_rows_sent                            AS lignes_renvoyees
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC
LIMIT 10;
```

Le schéma `sys` (disponible depuis la 10.6) rend ces données plus accessibles via des vues prêtes à l'emploi. Parmi les plus utiles : `sys.metrics`, qui **consolide** en une seule table les variables d'état globales, les métriques InnoDB et les statistiques mémoire ; `sys.statements_with_full_table_scans`, qui liste directement les requêtes effectuant des balayages complets ; `sys.statement_analysis` pour une vue d'ensemble des requêtes ; et `sys.innodb_lock_waits` pour les attentes de verrous InnoDB en cours.

```sql
-- Métriques unifiées (status global + InnoDB + mémoire)
SELECT * FROM sys.metrics WHERE Variable_name LIKE 'innodb_buffer_pool%';

-- Requêtes effectuant des balayages complets de table
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;

-- Attentes de verrous InnoDB en cours
SELECT * FROM sys.innodb_lock_waits\G
```

L'exploitation approfondie de `PERFORMANCE_SCHEMA` et de `sys` pour l'optimisation est développée en [15.8](../15-performance-tuning/08-performance-schema-sys.md).

---

## Méthode : ligne de référence, signaux d'alerte et tendances

Disposer des métriques ne suffit pas ; encore faut-il les exploiter avec méthode.

D'abord, établir une **ligne de référence**. Une anomalie se définit par rapport à un comportement normal : sans connaître le débit, la latence et les ratios habituels du serveur, on ne peut pas qualifier un écart. La baseline se capture sur plusieurs jours représentatifs, en tenant compte des cycles (heures de pointe, traitements nocturnes).

Ensuite, raisonner en **taux et en tendances**, jamais en valeurs absolues pour les compteurs cumulatifs. Une alerte pertinente combine généralement un **seuil** et une **tendance**, avec plusieurs niveaux de gravité.

Les signaux à privilégier pour l'alerting sont, en synthèse :

- l'**espace disque** sur tous les volumes concernés (voir [11.7](07-gestion-espace-disque.md)), avant tout ;
- une montée anormale de `Threads_running` (saturation ou blocage) ;
- le **retard de réplication** ([13.7](../13-replication/07-monitoring-troubleshooting.md)) ;
- une dégradation du taux de hit du buffer pool, *a fortiori* couplée à `Innodb_buffer_pool_wait_free > 0` ;
- un taux élevé de `Created_tmp_disk_tables` ou de `Slow_queries` ;
- un pic d'`Aborted_connects` (problème d'authentification, de réseau ou attaque) ;
- l'approche de la saturation des connexions (`Max_used_connections` proche de `max_connections`).

Enfin, **externaliser la collecte**. La supervision de production doit reposer sur un système de collecte time-series qui historise, calcule les taux et déclenche les alertes — Prometheus/Grafana via `mysqld_exporter`, ou une solution intégrée type PMM (voir [16.9](../16-devops-automatisation/09-monitoring-prometheus-grafana.md) et [16.10](../16-devops-automatisation/10-observabilite.md)). Les relevés SQL manuels restent un outil de diagnostic ponctuel, pas un dispositif de surveillance.

En combinant ces métriques avec le contrôle de l'espace disque ([11.7](07-gestion-espace-disque.md)) et de l'espace temporaire ([11.8](08-controle-espace-temporaire.md)), puis en les intégrant à une chaîne d'observabilité et d'alerting ([16.9](../16-devops-automatisation/09-monitoring-prometheus-grafana.md)–[16.11](../16-devops-automatisation/11-alerting-incident-response.md)), on transforme un ensemble de compteurs en une véritable capacité à anticiper et à diagnostiquer — l'objectif final du monitoring.

⏭️ [Thread Pool et gestion de la concurrence](/11-administration-configuration/10-thread-pool.md)

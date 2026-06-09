🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.2 — Requêtes de monitoring

> 📊 Le socle d'un suivi régulier : efficacité du buffer pool, requêtes coûteuses, retard de réplication.

Cette section regroupe des requêtes orientées santé continue du serveur. Toutes sont en lecture seule ; certaines supposent que le `PERFORMANCE_SCHEMA` soit activé (`performance_schema=ON`) ou que la journalisation des requêtes lentes pointe vers une table.

> 💡 **Atout MariaDB.** MariaDB conserve `information_schema.GLOBAL_STATUS` et `GLOBAL_VARIABLES` interrogeables — là où MySQL 8 les a déplacées vers `PERFORMANCE_SCHEMA`. On peut donc écrire les contrôles de statut sous forme de `SELECT`, plutôt que d'analyser la sortie d'un `SHOW`.

---

## Efficacité du buffer pool

Les compteurs du buffer pool s'obtiennent directement par requête sur `GLOBAL_STATUS` :

```sql
SELECT variable_name, variable_value
FROM   information_schema.GLOBAL_STATUS
WHERE  variable_name IN (
  'Innodb_buffer_pool_pages_total',
  'Innodb_buffer_pool_pages_free',
  'Innodb_buffer_pool_pages_data',
  'Innodb_buffer_pool_pages_dirty',
  'Innodb_buffer_pool_read_requests',
  'Innodb_buffer_pool_reads',
  'Innodb_buffer_pool_wait_free'
);
```

L'indicateur clé est le **taux de succès** (*hit ratio*) : la proportion de lectures servies depuis la mémoire plutôt que depuis le disque. Il se calcule à partir des lectures logiques (`read_requests`) et des lectures physiques (`reads`) :

```sql
SELECT ROUND(100 * (1 - rd.v / NULLIF(req.v, 0)), 2) AS hit_ratio_pct
FROM
  (SELECT variable_value + 0 AS v FROM information_schema.GLOBAL_STATUS
   WHERE variable_name = 'Innodb_buffer_pool_reads')          AS rd,
  (SELECT variable_value + 0 AS v FROM information_schema.GLOBAL_STATUS
   WHERE variable_name = 'Innodb_buffer_pool_read_requests')  AS req;
```

Sur un serveur OLTP correctement dimensionné, ce taux dépasse généralement 99 % ; une valeur durablement basse, ou un `Innodb_buffer_pool_wait_free` qui grimpe, suggère un buffer pool trop petit. La table `information_schema.INNODB_BUFFER_POOL_STATS` fournit par ailleurs un résumé sur une seule ligne (taille du pool, pages libres, pages modifiées). *Voir Ch. 15.2 pour le dimensionnement.*

## Requêtes lentes et coûteuses

Le simple décompte des requêtes lentes se lit dans le statut :

```sql
SELECT variable_value AS slow_queries
FROM   information_schema.GLOBAL_STATUS
WHERE  variable_name = 'Slow_queries';
```

Si la journalisation des requêtes lentes écrit dans une table (`log_output` incluant `TABLE`), on interroge directement `mysql.slow_log` :

```sql
SELECT start_time, user_host, query_time, lock_time,
       rows_sent, rows_examined, db, sql_text
FROM   mysql.slow_log
ORDER  BY query_time DESC
LIMIT  20;
```

Pour une vue agrégée par **type de requête** plutôt que par occurrence, le Performance Schema (activé) propose la table des digests, qui regroupe les requêtes similaires :

```sql
SELECT
  schema_name,
  digest_text,
  count_star                       AS executions,
  ROUND(sum_timer_wait / 1e12, 2)  AS total_s,
  ROUND(avg_timer_wait / 1e9,  2)  AS moy_ms,
  sum_rows_examined                AS lignes_examinees,
  sum_rows_sent                    AS lignes_envoyees
FROM   performance_schema.events_statements_summary_by_digest
ORDER  BY sum_timer_wait DESC
LIMIT  20;
```

> ⏱️ Attention aux unités : les colonnes `*_TIMER_WAIT` sont exprimées en **picosecondes**. D'où les divisions par `1e12` (vers les secondes) et `1e9` (vers les millisecondes).

L'analyse approfondie des requêtes lentes (slow query log détaillé, `pt-query-digest`) est traitée au [Ch. 15.7 — Analyse des requêtes lentes](../../15-performance-tuning/README.md).

## Réplication

En MariaDB, l'état de la réplication s'obtient par la commande `SHOW REPLICA STATUS` (alias de `SHOW SLAVE STATUS` depuis la 10.5), et non via des tables `PERFORMANCE_SCHEMA` interrogeables comme dans MySQL 8. L'affichage vertical est ici indispensable :

```sql
SHOW REPLICA STATUS\G
```

Les champs à surveiller en priorité :

| Champ | Ce qu'il indique |
|-------|------------------|
| `Slave_IO_Running` / `Slave_SQL_Running` | Les deux fils de réplication doivent être à `Yes` |
| `Seconds_Behind_Master` | Le retard estimé du réplica, en secondes |
| `Last_IO_Error` / `Last_SQL_Error` | La dernière erreur rencontrée (vide si tout va bien) |
| `Read_Master_Log_Pos` vs `Exec_Master_Log_Pos` | L'écart entre ce qui est reçu et ce qui est appliqué |
| `Using_Gtid` / `Gtid_IO_Pos` | Le mode GTID et la position courante |

Pour une réplication multi-source, la variante MariaDB liste tous les canaux :

```sql
SHOW ALL SLAVES STATUS\G
```

Enfin, les positions GTID se lisent directement dans les variables :

```sql
SELECT @@gtid_current_pos, @@gtid_slave_pos, @@gtid_binlog_pos;
```

`Seconds_Behind_Master` est commode mais à interpréter avec prudence : il vaut `NULL` lorsque la réplication est arrêtée et peut paraître nul sur un primaire inactif. Les subtilités du lag et le troubleshooting sont détaillés au [Ch. 13.7 — Monitoring et troubleshooting](../../13-replication/README.md).

## Pour aller plus loin

Le dimensionnement mémoire est traité au [Ch. 15.2](../../15-performance-tuning/README.md), l'analyse des requêtes lentes au [Ch. 15.7](../../15-performance-tuning/README.md), et le troubleshooting de la réplication au [Ch. 13.7](../../13-replication/README.md). Pour une supervision continue avec tableaux de bord et alerting (`mysqld_exporter`, Prometheus, Grafana), voir le [Ch. 16.9](../../16-devops-automatisation/README.md). Pour un instantané rapide en ligne de commande, voir l'[Annexe B.2](../b-commandes-cli/02-informations-systeme.md).

---

⬅️ [C.1 — Requêtes d'administration](01-requetes-administration.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [C.3 — Requêtes d'analyse](03-requetes-analyse.md)

⏭️ [Requêtes d'analyse (statistiques, index usage)](/annexes/c-requetes-sql-reference/03-requetes-analyse.md)

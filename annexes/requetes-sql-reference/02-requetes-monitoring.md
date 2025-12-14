🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.2 - Requêtes de Monitoring

> **Niveau** : Intermédiaire à Avancé  
> **Type** : Requêtes de surveillance et métriques  
> **Usage** : Monitoring continu, alerting, dashboards

---

## 📖 Introduction

Cette section fournit des **requêtes SQL de monitoring** essentielles pour surveiller la santé, la performance et la disponibilité de MariaDB. Ces requêtes sont conçues pour être intégrées dans des systèmes de monitoring automatisés.

### 🎯 Catégories Couvertes

- 💾 **Buffer Pool** : Cache InnoDB et hit rate
- 🐌 **Slow Queries** : Identification requêtes lentes
- 🔄 **Réplication** : État et lag
- 📈 **Métriques Performance** : QPS, TPS, connexions
- 💿 **I/O et Disque** : Statistiques lecture/écriture
- 🔌 **Connexions** : Utilisation et limites

---

## 💾 BUFFER POOL

### 1.1 - Buffer Pool Hit Rate
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Taux de succès du buffer pool (Hit Rate)
-- ============================================================
-- Description :
-- Mesure l'efficacité du cache InnoDB
-- Hit Rate = (requêtes depuis cache / total requêtes) * 100
--
-- Interprétation :
-- - > 99% : Excellent (objectif production)
-- - 95-99% : Bon
-- - < 95% : Buffer pool trop petit ou workload cache-hostile
--
-- Formule :
-- Hit Rate = 1 - (reads / read_requests)
-- ============================================================

SELECT 
  ROUND(
    (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100, 4
  ) AS buffer_pool_hit_rate_pct,
  
  -- Valeurs brutes pour analyse
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS total_read_requests,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') AS disk_reads,
  
  -- Interprétation automatique
  CASE 
    WHEN (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100 >= 99 THEN '✓ Excellent'
    WHEN (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100 >= 95 THEN '⚠️ Bon'
    ELSE '❌ Insuffisant - Augmenter buffer pool'
  END AS status;
```

**Résultat exemple** :
```
+--------------------------+--------------------+------------+---------------+
| buffer_pool_hit_rate_pct | total_read_requests| disk_reads | status        |
+--------------------------+--------------------+------------+---------------+
|                  99.8756 |        98765432100 |  123456789 | ✓ Excellent   |
+--------------------------+--------------------+------------+---------------+
```

**Action si hit rate < 99%** :
```sql
-- Vérifier taille actuelle
SELECT @@innodb_buffer_pool_size / 1024 / 1024 / 1024 AS buffer_pool_gb;

-- Augmenter (exemple : à 16 GB)
SET GLOBAL innodb_buffer_pool_size = 17179869184;  -- 16 GB
```

---

### 1.2 - Statistiques Détaillées Buffer Pool
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Vue complète du buffer pool InnoDB
-- ============================================================

SELECT 
  -- Taille et utilisation
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME = 'innodb_buffer_pool_size') / 1024 / 1024 / 1024, 2
  ) AS buffer_pool_size_gb,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'innodb_buffer_pool_instances') AS bp_instances,
  
  -- Pages
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') AS total_pages,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') AS data_pages,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_free') AS free_pages,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') AS dirty_pages,
  
  -- Ratio utilisation
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') * 100, 2
  ) AS usage_pct,
  
  -- Ratio dirty
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') * 100, 2
  ) AS dirty_ratio_pct;
```

**Interprétation** :
- `usage_pct` proche 100% : Normal (buffer pool bien utilisé)
- `dirty_ratio_pct` > 10% persistant : Écritures en attente, vérifier I/O

---

### 1.3 - Évolution Buffer Pool (Période)
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Calculer métriques buffer pool sur intervalle
-- ============================================================
-- Description :
-- Compare valeurs actuelles avec snapshot précédent
-- pour calculer activité sur période
--
-- Méthode :
-- 1. Capturer snapshot initial
-- 2. Attendre N secondes
-- 3. Comparer avec valeurs actuelles
-- ============================================================

-- Créer table temporaire pour snapshot
CREATE TEMPORARY TABLE IF NOT EXISTS bp_snapshot (
  metric VARCHAR(100),
  value BIGINT,
  captured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Snapshot initial
TRUNCATE bp_snapshot;
INSERT INTO bp_snapshot (metric, value)
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
  'Innodb_buffer_pool_read_requests',
  'Innodb_buffer_pool_reads',
  'Innodb_buffer_pool_write_requests',
  'Innodb_pages_read',
  'Innodb_pages_written'
);

-- Attendre 60 secondes (ajuster selon besoin)
-- DO SLEEP(60);

-- Comparer (exécuter après délai)
SELECT 
  s.metric,
  s.value AS value_before,
  c.VARIABLE_VALUE AS value_now,
  (c.VARIABLE_VALUE - s.value) AS delta,
  ROUND(
    (c.VARIABLE_VALUE - s.value) / 
    TIMESTAMPDIFF(SECOND, s.captured_at, NOW()), 2
  ) AS per_second
FROM bp_snapshot s
JOIN information_schema.GLOBAL_STATUS c 
  ON s.metric = c.VARIABLE_NAME
ORDER BY s.metric;

-- Calculer hit rate sur période
SELECT 
  ROUND(
    (1 - 
      (SELECT (c.VARIABLE_VALUE - s.value) 
       FROM bp_snapshot s
       JOIN information_schema.GLOBAL_STATUS c ON s.metric = c.VARIABLE_NAME
       WHERE s.metric = 'Innodb_buffer_pool_reads') /
      (SELECT (c.VARIABLE_VALUE - s.value)
       FROM bp_snapshot s
       JOIN information_schema.GLOBAL_STATUS c ON s.metric = c.VARIABLE_NAME
       WHERE s.metric = 'Innodb_buffer_pool_read_requests')
    ) * 100, 2
  ) AS hit_rate_last_period_pct;
```

---

### 1.4 - Taux de Remplacement Pages (Churn Rate)
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Analyse du churn rate (remplacement de pages)
-- ============================================================
-- Description :
-- Mesure à quelle vitesse les pages sont évincées du cache
--
-- young pages : Nouvelles pages chargées
-- not young : Pages déjà en cache accédées
--
-- High churn = buffer pool trop petit pour working set
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_made_young') AS pages_made_young,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_not_made_young') AS pages_not_young,
  
  -- Ratio young/total
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_made_young') /
    ((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_made_young') +
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_not_made_young')) * 100, 2
  ) AS young_ratio_pct,
  
  -- Interprétation
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_made_young') /
         ((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
           WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_made_young') +
          (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
           WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_not_made_young')) > 0.5 
    THEN '⚠️ High churn - Consider increasing buffer pool'
    ELSE '✓ Normal churn rate'
  END AS status;
```

---

## 🐌 SLOW QUERIES

### 2.1 - Ratio Slow Queries
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Pourcentage de requêtes lentes
-- ============================================================
-- Description :
-- Calcule le ratio de requêtes dépassant long_query_time
--
-- Objectif : < 1% (idéalement < 0.1%)
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Slow_queries') AS slow_queries_total,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Questions') AS total_queries,
  
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Slow_queries') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') * 100, 4
  ) AS slow_query_ratio_pct,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'long_query_time') AS long_query_time_sec,
  
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Slow_queries') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Questions') * 100 < 0.1 
    THEN '✓ Excellent'
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Slow_queries') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Questions') * 100 < 1 
    THEN '⚠️ Acceptable'
    ELSE '❌ Trop élevé - Optimiser requêtes'
  END AS status;
```

---

### 2.2 - Slow Queries Actuellement en Cours
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Requêtes lentes en cours d'exécution
-- ============================================================
-- Description :
-- Identifie requêtes actuellement actives dépassant le seuil
--
-- Paramètre :
-- Modifier TIME > value selon long_query_time
-- ============================================================

SELECT 
  pl.ID AS process_id,
  pl.USER AS user,
  pl.HOST AS host,
  pl.DB AS database_name,
  pl.TIME AS duration_seconds,
  ROUND(pl.TIME / 60, 2) AS duration_minutes,
  pl.STATE AS state,
  pl.INFO AS query_text,
  
  -- Estimation progression (si disponible)
  COALESCE(
    (SELECT PROGRESS 
     FROM information_schema.PROCESSLIST p2 
     WHERE p2.ID = pl.ID), 
    0
  ) AS progress_pct
FROM information_schema.PROCESSLIST pl
WHERE pl.COMMAND != 'Sleep'
  AND pl.TIME > (
    SELECT VARIABLE_VALUE 
    FROM information_schema.GLOBAL_VARIABLES 
    WHERE VARIABLE_NAME = 'long_query_time'
  )
ORDER BY pl.TIME DESC;
```

**Action** :
```sql
-- Analyser requête lente
EXPLAIN <query>;

-- Si vraiment problématique, tuer
KILL QUERY <process_id>;
```

---

### 2.3 - Statistiques Slow Query Log
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Analyse du slow query log
-- ============================================================
-- Description :
-- Vérifie configuration et utilisation du slow query log
--
-- Note :
-- Utiliser pt-query-digest pour analyse détaillée du fichier
-- ============================================================

SELECT 
  'Slow Query Log Status' AS metric,
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'slow_query_log') AS value
UNION ALL
SELECT 
  'Slow Query Log File',
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'slow_query_log_file')
UNION ALL
SELECT 
  'Long Query Time (sec)',
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'long_query_time')
UNION ALL
SELECT 
  'Log Queries Not Using Indexes',
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'log_queries_not_using_indexes')
UNION ALL
SELECT 
  'Total Slow Queries',
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Slow_queries');
```

**Activer slow query log si désactivé** :
```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- 2 secondes
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

---

### 2.4 - Requêtes Sans Index en Cours
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Identifier requêtes actives n'utilisant pas d'index
-- ============================================================
-- Description :
-- Détecte table scans complets qui ralentissent le serveur
--
-- Note :
-- Combiner avec EXPLAIN pour optimiser
-- ============================================================

-- Via PROCESSLIST (requêtes en cours)
SELECT 
  ID,
  USER,
  DB,
  TIME,
  STATE,
  INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND (STATE LIKE '%table%' OR STATE LIKE '%Sending data%')
  AND INFO IS NOT NULL
ORDER BY TIME DESC;

-- Vérifier si table scans
SELECT 
  VARIABLE_NAME,
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
  'Select_scan',          -- Full table scans
  'Select_full_join',     -- Joins sans index
  'Select_range_check'    -- Range checks
);
```

---

## 🔄 RÉPLICATION

### 3.1 - État Réplication (Replica)
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Statut complet réplication
-- ============================================================
-- Description :
-- À exécuter sur le replica
--
-- Colonnes critiques :
-- - Slave_IO_Running : Réception binlog (doit être Yes)
-- - Slave_SQL_Running : Application binlog (doit être Yes)
-- - Seconds_Behind_Master : Lag en secondes (objectif: < 5s)
-- - Last_Error : Dernière erreur rencontrée
-- ============================================================

SHOW SLAVE STATUS\G

-- Version condensée (colonnes essentielles)
SELECT 
  Slave_IO_Running AS io_running,
  Slave_SQL_Running AS sql_running,
  Seconds_Behind_Master AS lag_seconds,
  Master_Host AS master_host,
  Master_Port AS master_port,
  Master_Log_File AS current_binlog_file,
  Read_Master_Log_Pos AS read_position,
  Relay_Log_File AS relay_log_file,
  Relay_Log_Pos AS relay_position,
  Last_Errno AS last_error_number,
  Last_Error AS last_error_message
FROM (SHOW SLAVE STATUS) AS s\G
```

**Interprétation résultat** :
```
Slave_IO_Running: Yes         ✓ OK
Slave_SQL_Running: Yes        ✓ OK
Seconds_Behind_Master: 0      ✓ OK (pas de lag)

Si Seconds_Behind_Master: NULL → Réplication arrêtée
Si Seconds_Behind_Master: > 60 → Lag important, investiguer
```

---

### 3.2 - Détection Lag Réplication
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Monitorer le lag réplication avec alerting
-- ============================================================
-- Description :
-- Calcule et catégorise le lag réplication
--
-- Seuils :
-- - < 5s : Normal
-- - 5-30s : Attention
-- - 30-60s : Warning
-- - > 60s : Critique
-- ============================================================

SELECT 
  'Replication Lag' AS metric,
  COALESCE(Seconds_Behind_Master, -1) AS lag_seconds,
  CASE 
    WHEN Seconds_Behind_Master IS NULL THEN '🔴 REPLICATION STOPPED'
    WHEN Seconds_Behind_Master = 0 THEN '✓ No lag'
    WHEN Seconds_Behind_Master < 5 THEN '✓ Normal lag'
    WHEN Seconds_Behind_Master < 30 THEN '⚠️ Attention'
    WHEN Seconds_Behind_Master < 60 THEN '⚠️ Warning'
    ELSE '🔴 CRITICAL LAG'
  END AS status,
  Slave_IO_Running AS io_thread,
  Slave_SQL_Running AS sql_thread
FROM (SHOW SLAVE STATUS) AS s\G

-- Requête utilisable dans scripts monitoring
SELECT 
  CASE 
    WHEN (SELECT Slave_IO_Running FROM (SHOW SLAVE STATUS) s) != 'Yes' THEN 1
    WHEN (SELECT Slave_SQL_Running FROM (SHOW SLAVE STATUS) s) != 'Yes' THEN 1
    WHEN (SELECT Seconds_Behind_Master FROM (SHOW SLAVE STATUS) s) > 60 THEN 1
    ELSE 0
  END AS alert_flag;
```

---

### 3.3 - Position Binlog Master
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Position actuelle master (binlog)
-- ============================================================
-- Description :
-- À exécuter sur le master
-- Utilisé pour configurer/vérifier réplication
-- ============================================================

SHOW MASTER STATUS;

-- Résultat :
-- +------------------+----------+--------------+------------------+
-- | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +------------------+----------+--------------+------------------+
-- | mysql-bin.000042 |   123456 |              |                  |
-- +------------------+----------+--------------+------------------+

-- Lister tous les binlogs
SHOW BINARY LOGS;
```

---

### 3.4 - GTID Status
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- État GTID (Global Transaction ID)
-- ============================================================
-- Description :
-- Si réplication basée GTID activée
-- ============================================================

-- Vérifier si GTID activé
SELECT @@gtid_mode, @@gtid_strict_mode;

-- Position GTID executée
SELECT @@gtid_executed;

-- Sur replica : GTIDs récupérés mais pas encore exécutés
SELECT @@gtid_slave_pos;

-- Comparaison master/slave
-- Sur master :
SELECT @@gtid_executed AS master_gtid;

-- Sur replica :
SELECT 
  @@gtid_slave_pos AS replica_current_gtid,
  @@gtid_executed AS replica_executed_gtid;
```

---

### 3.5 - Erreurs Réplication
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Historique erreurs réplication
-- ============================================================

-- Dernière erreur
SELECT 
  Last_Errno AS error_number,
  Last_Error AS error_message,
  Last_SQL_Errno AS sql_error_number,
  Last_SQL_Error AS sql_error_message,
  Last_IO_Errno AS io_error_number,
  Last_IO_Error AS io_error_message
FROM (SHOW SLAVE STATUS) AS s\G

-- Commandes utiles en cas d'erreur
-- Ignorer une erreur (avec EXTRÊME prudence)
-- SET GLOBAL sql_slave_skip_counter = 1;
-- START SLAVE;

-- Ou restart depuis position spécifique
-- STOP SLAVE;
-- CHANGE MASTER TO MASTER_LOG_FILE='...', MASTER_LOG_POS=...;
-- START SLAVE;
```

---

## 📈 MÉTRIQUES PERFORMANCE

### 4.1 - QPS et TPS
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Queries Per Second et Transactions Per Second
-- ============================================================

SELECT 
  -- QPS (Questions Per Second)
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime'), 2
  ) AS avg_qps,
  
  -- TPS (Transactions Per Second)
  ROUND(
    ((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Com_commit') +
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Com_rollback')) /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime'), 2
  ) AS avg_tps,
  
  -- Total questions
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Questions') AS total_questions,
  
  -- Total commits
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Com_commit') AS total_commits,
  
  -- Total rollbacks
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Com_rollback') AS total_rollbacks,
  
  -- Uptime
  CONCAT(
    FLOOR((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
           WHERE VARIABLE_NAME = 'Uptime') / 86400), 'd ',
    FLOOR(((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
            WHERE VARIABLE_NAME = 'Uptime') % 86400) / 3600), 'h'
  ) AS uptime;
```

---

### 4.2 - Utilisation Connexions
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Analyse utilisation des connexions
-- ============================================================
-- Description :
-- Surveille utilisation vs limite max_connections
--
-- Alerte si > 80% utilisé
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Threads_connected') AS current_connections,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Max_used_connections') AS max_used_connections,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'max_connections') AS max_connections_limit,
  
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_connected') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME = 'max_connections') * 100, 2
  ) AS usage_pct,
  
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Threads_connected') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
          WHERE VARIABLE_NAME = 'max_connections') > 0.8 
    THEN '🔴 CRITICAL - Increase max_connections'
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Threads_connected') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
          WHERE VARIABLE_NAME = 'max_connections') > 0.6 
    THEN '⚠️ Warning - Monitor closely'
    ELSE '✓ OK'
  END AS status;
```

---

### 4.3 - Thread Cache Efficiency
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Efficacité du thread cache
-- ============================================================
-- Description :
-- Thread cache réduit overhead création/destruction threads
--
-- Hit rate élevé = bon (réutilisation threads)
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Threads_created') AS threads_created,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Connections') AS total_connections,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'thread_cache_size') AS thread_cache_size,
  
  -- Thread cache hit rate
  ROUND(
    (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Threads_created') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Connections')
    ) * 100, 2
  ) AS thread_cache_hit_rate_pct,
  
  CASE 
    WHEN (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Threads_created') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Connections')
    ) * 100 > 90 THEN '✓ Excellent'
    WHEN (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Threads_created') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Connections')
    ) * 100 > 80 THEN '⚠️ Good'
    ELSE '❌ Increase thread_cache_size'
  END AS status;
```

---

### 4.4 - Table Cache Hit Rate
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Efficacité du table cache
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Open_tables') AS open_tables,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Opened_tables') AS opened_tables_total,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'table_open_cache') AS table_open_cache_limit,
  
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Open_tables') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Opened_tables') * 100, 2
  ) AS table_cache_hit_rate_pct,
  
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Open_tables') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Opened_tables') > 0.95 
    THEN '✓ Excellent'
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Open_tables') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Opened_tables') > 0.85 
    THEN '⚠️ Good'
    ELSE '❌ Consider increasing table_open_cache'
  END AS status;
```

---

## 💿 I/O ET DISQUE

### 5.1 - Statistiques I/O InnoDB
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Activité I/O InnoDB
-- ============================================================

SELECT 
  -- Lecture
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_data_read') AS total_bytes_read,
  
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_read') / 1024 / 1024 / 1024, 2
  ) AS total_gb_read,
  
  -- Écriture
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_data_written') AS total_bytes_written,
  
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_written') / 1024 / 1024 / 1024, 2
  ) AS total_gb_written,
  
  -- Pages
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_pages_read') AS pages_read,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_pages_written') AS pages_written,
  
  -- Moyenne par seconde
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_read') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime') / 1024 / 1024, 2
  ) AS avg_mb_read_per_sec,
  
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_written') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime') / 1024 / 1024, 2
  ) AS avg_mb_written_per_sec;
```

---

### 5.2 - Pending I/O Operations
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Opérations I/O en attente
-- ============================================================
-- Description :
-- Détecte goulots d'étranglement I/O
--
-- Si valeurs constamment élevées → I/O saturé
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_data_pending_reads') AS pending_reads,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_data_pending_writes') AS pending_writes,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Innodb_data_pending_fsyncs') AS pending_fsyncs,
  
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Innodb_data_pending_reads') > 10 OR
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Innodb_data_pending_writes') > 10 OR
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Innodb_data_pending_fsyncs') > 10
    THEN '⚠️ I/O congestion detected'
    ELSE '✓ OK'
  END AS status;
```

**Actions si I/O saturé** :
- Vérifier `innodb_io_capacity` et `innodb_io_capacity_max`
- Analyser avec `iostat` au niveau système
- Considérer upgrade disques (SSD/NVMe)

---

## 🔌 CONNEXIONS ET ERREURS

### 6.1 - Erreurs de Connexion
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Statistiques erreurs connexion
-- ============================================================

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Aborted_connects') AS aborted_connects,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Aborted_clients') AS aborted_clients,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Connection_errors_max_connections') AS max_conn_errors,
  
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Connections') AS total_connections,
  
  -- Ratio aborted/total
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Aborted_connects') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Connections') * 100, 2
  ) AS aborted_connect_ratio_pct;
```

**Interprétation** :
- `Aborted_connects` élevé : Échecs authentification, timeout réseau
- `Aborted_clients` élevé : Clients déconnectés sans fermer proprement
- `max_conn_errors` > 0 : Limite max_connections atteinte

---

### 6.2 - Temps Moyen Requête
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Performance moyenne des requêtes
-- ============================================================

-- Via Performance Schema (si activé)
SELECT 
  DIGEST_TEXT AS query_pattern,
  COUNT_STAR AS exec_count,
  ROUND(AVG_TIMER_WAIT / 1000000000, 3) AS avg_time_sec,
  ROUND(MAX_TIMER_WAIT / 1000000000, 3) AS max_time_sec,
  ROUND(SUM_ROWS_EXAMINED / COUNT_STAR, 0) AS avg_rows_examined,
  ROUND(SUM_ROWS_SENT / COUNT_STAR, 0) AS avg_rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
  AND DIGEST_TEXT NOT LIKE '%performance_schema%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

---

## ✅ Dashboard SQL Complet

### 7.1 - Vue d'Ensemble Santé Serveur
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Dashboard complet monitoring
-- ============================================================
-- Description :
-- Vue unifiée des métriques critiques
-- Utilisable pour rapports automatisés
-- ============================================================

SELECT 'SERVER INFO' AS category, '' AS metric, '' AS value, '' AS status
UNION ALL
SELECT '', 'Version', @@version, '✓'
UNION ALL
SELECT '', 'Uptime', 
  CONCAT(
    FLOOR((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
           WHERE VARIABLE_NAME = 'Uptime') / 86400), 'd ',
    FLOOR(((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
            WHERE VARIABLE_NAME = 'Uptime') % 86400) / 3600), 'h'
  ), '✓'

UNION ALL SELECT '', '', '', ''
UNION ALL SELECT 'PERFORMANCE', '', '', ''

UNION ALL
SELECT '', 'QPS', 
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime'), 2
  ),
  '✓'

UNION ALL
SELECT '', 'Buffer Pool Hit Rate %',
  ROUND(
    (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100, 2
  ),
  CASE 
    WHEN (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100 >= 99 THEN '✓'
    ELSE '⚠️'
  END

UNION ALL
SELECT '', 'Slow Query Ratio %',
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Slow_queries') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') * 100, 4
  ),
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Slow_queries') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Questions') * 100 < 1 
    THEN '✓'
    ELSE '⚠️'
  END

UNION ALL SELECT '', '', '', ''
UNION ALL SELECT 'CONNECTIONS', '', '', ''

UNION ALL
SELECT '', 'Current / Max',
  CONCAT(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_connected'), ' / ',
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME = 'max_connections')
  ),
  CASE 
    WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Threads_connected') /
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
          WHERE VARIABLE_NAME = 'max_connections') > 0.8 
    THEN '⚠️'
    ELSE '✓'
  END

ORDER BY category, metric;
```

---

## 🔗 Ressources

### Liens Vers Autres Sections
- **[← C.1 Requêtes d'Administration](01-requetes-administration.md)** : Locks, processlist, table sizes
- **[→ C.3 Requêtes d'Analyse](03-requetes-analyse.md)** : Index usage, fragmentation

### Documentation
- [InnoDB Monitors](https://mariadb.com/kb/en/innodb-monitors/)
- [Server Status Variables](https://mariadb.com/kb/en/server-status-variables/)
- [Replication](https://mariadb.com/kb/en/replication/)

---

**MariaDB** : 11.8 LTS

⏭️ [Requêtes d'analyse (statistiques, index usage)](/annexes/requetes-sql-reference/03-requetes-analyse.md)

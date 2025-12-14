🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.8 Performance Schema et sys schema

> **Niveau** : Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : 
> - Sections 15.1-15.7 (Méthodologie, Mémoire, I/O, InnoDB, Analyse requêtes)
> - Compréhension de l'architecture MariaDB/MySQL
> - Expérience en diagnostic de performance
> - Connaissance SQL avancée

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre l'architecture** de Performance Schema et son fonctionnement
- **Activer et configurer** Performance Schema avec overhead minimal
- **Exploiter les tables critiques** pour le diagnostic de performance
- **Utiliser sys schema** pour des analyses rapides et efficaces
- **Identifier les requêtes problématiques** en temps réel
- **Diagnostiquer les locks** et contentions
- **Analyser les I/O** au niveau table et fichier
- **Monitorer la consommation mémoire** par composant
- **Appliquer les best practices** pour un monitoring production
- **Automatiser les rapports** de performance

---

## Introduction

**Performance Schema** est le système d'instrumentation natif de MariaDB/MySQL qui permet de **monitorer en temps réel** l'activité interne du serveur avec un overhead minimal.

**sys schema** est une couche de vues et procédures construites au-dessus de Performance Schema pour **simplifier et accélérer** l'analyse.

### Pourquoi Performance Schema ?

```
┌────────────────────────────────────────────────────┐
│  AVANT Performance Schema                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  Outils de diagnostic :                            │
│  • SHOW STATUS (métriques globales seulement)      │
│  • SHOW PROCESSLIST (snapshot instantané)          │
│  • Slow query log (post-mortem, pas temps réel)    │
│  • SHOW ENGINE INNODB STATUS (complexe)            │
│                                                    │
│  Limitations :                                     │
│  ❌ Pas de drill-down détaillé                     │
│  ❌ Pas d'historique                               │
│  ❌ Agrégations difficiles                         │
│  ❌ Pas de visibilité fine (locks, I/O, memory)    │
│                                                    │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  AVEC Performance Schema                           │
├────────────────────────────────────────────────────┤
│                                                    │
│  Capacités :                                       │
│  ✅ Instrumentation fine-grained                   │
│  ✅ Historique des événements                      │
│  ✅ Agrégations automatiques                       │
│  ✅ Drill-down par : requête, table, user, host    │
│  ✅ Métriques : temps, locks, I/O, memory, réseau  │
│  ✅ Overhead <5% (souvent <2%)                     │
│  ✅ Requêtes SQL standard                          │
│                                                    │
│  + sys schema :                                    │
│  ✅ Vues pré-construites lisibles                  │
│  ✅ Rapports formatés                              │
│  ✅ Procédures d'analyse                           │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Architecture de Performance Schema

### Composants principaux

```
┌──────────────────────────────────────────────────────┐
│          ARCHITECTURE PERFORMANCE SCHEMA             │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────────────────────────────────┐         │
│  │   INSTRUMENTATION POINTS                │         │
│  │   (Code MariaDB instrumenté)            │         │
│  │                                         │         │
│  │   • Statements (requêtes SQL)           │         │
│  │   • Stages (étapes exécution)           │         │
│  │   • Waits (attentes)                    │         │
│  │   • Locks (verrous)                     │         │
│  │   • I/O operations                      │         │
│  │   • Memory allocation                   │         │
│  │   • Transactions                        │         │
│  └────────────┬────────────────────────────┘         │
│               │                                      │
│               ▼                                      │
│  ┌─────────────────────────────────────────┐         │
│  │   CONSUMERS (Collecteurs)               │         │
│  │                                         │         │
│  │   • events_statements_current           │         │
│  │   • events_statements_history           │         │
│  │   • events_waits_current                │         │
│  │   • events_stages_current               │         │
│  │   • Global stats                        │         │
│  └────────────┬────────────────────────────┘         │
│               │                                      │
│               ▼                                      │
│  ┌─────────────────────────────────────────┐         │
│  │   TABLES (Stockage)                     │         │
│  │                                         │         │
│  │   • events_* (événements bruts)         │         │
│  │   • *_summary_* (agrégations)           │         │
│  │   • setup_* (configuration)             │         │
│  └─────────────────────────────────────────┘         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Catégories de tables

```sql
-- Performance Schema contient 100+ tables organisées en catégories

-- 1. SETUP (Configuration)
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'performance_schema'
AND TABLE_NAME LIKE 'setup_%'
ORDER BY TABLE_NAME;
/*
setup_actors          -- Quels users monitorer
setup_consumers       -- Quels consumers activer
setup_instruments     -- Quels instruments activer
setup_objects         -- Quels objets monitorer
setup_threads         -- Configuration threads
*/

-- 2. EVENTS (Événements en cours et historique)
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'performance_schema'
AND TABLE_NAME LIKE 'events_%'
ORDER BY TABLE_NAME;
/*
events_statements_current     -- Requêtes en cours
events_statements_history     -- Dernières requêtes par thread
events_statements_history_long -- Historique global
events_waits_current          -- Attentes en cours
events_stages_current         -- Étapes en cours
events_transactions_current   -- Transactions en cours
*/

-- 3. SUMMARY (Agrégations)
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'performance_schema'
AND TABLE_NAME LIKE '%summary%'
ORDER BY TABLE_NAME;
/*
events_statements_summary_by_digest  -- Par pattern de requête
events_statements_summary_by_thread  -- Par thread
table_io_waits_summary_by_table      -- I/O par table
table_lock_waits_summary_by_table    -- Locks par table
file_summary_by_instance             -- I/O par fichier
memory_summary_by_thread_by_event    -- Mémoire par thread
*/

-- 4. INSTANCES (Objets système)
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'performance_schema'
AND TABLE_NAME LIKE '%instances'
ORDER BY TABLE_NAME;
/*
file_instances       -- Fichiers ouverts
mutex_instances      -- Mutex
rwlock_instances     -- Read-write locks
socket_instances     -- Connexions réseau
*/
```

---

## Activation et Configuration

### Vérifier l'état de Performance Schema

```sql
-- Performance Schema est-il activé ?
SHOW VARIABLES LIKE 'performance_schema';
-- ON = activé, OFF = désactivé

-- Mémoire allouée
SHOW VARIABLES LIKE 'performance_schema%';

-- Version MariaDB récente : Activé par défaut
-- Si désactivé, nécessite redémarrage pour activer
```

### Activer Performance Schema (si désactivé)

```ini
# /etc/mysql/my.cnf ou /etc/my.cnf.d/server.cnf

[mariadb]
# Activer Performance Schema
performance_schema = ON

# Configuration mémoire (optionnel, auto-sizing généralement bon)
performance_schema_max_table_instances = 12500
performance_schema_max_table_handles = 4000
performance_schema_events_statements_history_long_size = 10000

# Après modification : Redémarrage requis
# systemctl restart mariadb
```

### Configuration des instruments et consumers

```sql
-- Voir tous les instruments disponibles
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
ORDER BY NAME;

-- Activer/désactiver des instruments spécifiques
-- Exemple : Activer instrumentation des statements
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';

-- Voir les consumers actifs
SELECT * FROM performance_schema.setup_consumers;

-- Activer consumer pour historique long
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME = 'events_statements_history_long';

-- Configuration recommandée pour production
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME IN (
    'events_statements_current',
    'events_statements_history',
    'events_statements_history_long',
    'statements_digest'
);
```

### Configuration optimale pour production

```sql
-- Script de configuration initiale

-- 1. Activer consumers essentiels
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME IN (
    'events_statements_current',
    'events_statements_history_long',
    'events_waits_current',
    'global_instrumentation',
    'thread_instrumentation',
    'statements_digest'
);

-- 2. Activer instruments critiques uniquement (réduire overhead)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%'
   OR NAME LIKE 'wait/io/%'
   OR NAME LIKE 'wait/lock/%';

-- 3. Désactiver instruments moins critiques
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO'
WHERE NAME LIKE 'wait/synch/%'  -- Moins critique
   OR NAME LIKE 'stage/%';      -- Overhead élevé

-- 4. Configurer actors (tous les users)
TRUNCATE TABLE performance_schema.setup_actors;
INSERT INTO performance_schema.setup_actors 
VALUES ('%', '%', '%', 'YES', 'YES');
```

### Mesurer l'overhead

```sql
-- Tester l'overhead de Performance Schema

-- Benchmark sans Performance Schema
SET GLOBAL performance_schema = OFF;
-- Redémarrer MariaDB
-- Exécuter workload pendant 10 minutes
-- Noter : QPS, latence p95, CPU%

-- Benchmark avec Performance Schema
SET GLOBAL performance_schema = ON;
-- Redémarrer MariaDB
-- Configuration comme ci-dessus
-- Exécuter même workload
-- Comparer métriques

-- Overhead typique :
-- Configuration minimale : <2%
-- Configuration standard : 2-5%
-- Configuration maximale : 5-10%
```

---

## Tables critiques de Performance Schema

### 1. events_statements_summary_by_digest

**La table la plus importante** pour identifier les requêtes problématiques.

```sql
-- Top 10 requêtes par temps cumulé
SELECT 
    SCHEMA_NAME as db,
    SUBSTRING(DIGEST_TEXT, 1, 80) as query_pattern,
    COUNT_STAR as exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) as avg_time_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) as total_time_sec,
    ROUND(SUM_LOCK_TIME / 1000000000000, 3) as total_lock_time_sec,
    SUM_ROWS_EXAMINED as total_rows_examined,
    SUM_ROWS_SENT as total_rows_sent,
    SUM_ROWS_AFFECTED as total_rows_affected,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 2) as exam_to_sent_ratio,
    FIRST_SEEN,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Résultat exemple :
/*
+-------+------------------------------------------+-----------+--------------+----------------+---------------------+
| db    | query_pattern                            | exec_count| avg_time_sec | total_time_sec | exam_to_sent_ratio  |
+-------+------------------------------------------+-----------+--------------+----------------+---------------------+
| shop  | SELECT * FROM orders WHERE customer_id=? | 125000    | 0.852        | 106500         | 5234.5 ← PROBLÈME!  |
| shop  | SELECT * FROM products WHERE categ...    | 50000     | 0.125        | 6250           | 1.2                 |
+-------+------------------------------------------+-----------+--------------+----------------+---------------------+
*/
```

**Métriques clés** :
- `COUNT_STAR` : Nombre d'exécutions
- `AVG_TIMER_WAIT` : Temps moyen (nanosecondes)
- `SUM_TIMER_WAIT` : Temps cumulé total
- `SUM_ROWS_EXAMINED` / `SUM_ROWS_SENT` : Ratio efficacité

### 2. events_statements_current

**Requêtes en cours d'exécution** en ce moment.

```sql
-- Voir toutes les requêtes en cours
SELECT 
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_HOST,
    t.PROCESSLIST_DB,
    t.PROCESSLIST_COMMAND,
    t.PROCESSLIST_TIME as duration_sec,
    SUBSTRING(s.SQL_TEXT, 1, 100) as query_text,
    s.TIMER_WAIT / 1000000000000 as elapsed_sec,
    s.LOCK_TIME / 1000000000000 as lock_time_sec,
    s.ROWS_EXAMINED,
    s.ROWS_SENT
FROM performance_schema.threads t
JOIN performance_schema.events_statements_current s 
    ON t.THREAD_ID = s.THREAD_ID
WHERE t.PROCESSLIST_COMMAND != 'Sleep'
AND t.PROCESSLIST_ID IS NOT NULL
ORDER BY s.TIMER_WAIT DESC;

-- Identifier la requête la plus lente en cours
SELECT 
    THREAD_ID,
    SUBSTRING(SQL_TEXT, 1, 200) as query,
    TIMER_WAIT / 1000000000000 as elapsed_sec,
    LOCK_TIME / 1000000000000 as lock_sec,
    ROWS_EXAMINED
FROM performance_schema.events_statements_current
WHERE TIMER_WAIT IS NOT NULL
ORDER BY TIMER_WAIT DESC
LIMIT 1;
```

### 3. table_io_waits_summary_by_table

**I/O par table** : Identifier les tables les plus sollicitées.

```sql
-- Top 10 tables par I/O total
SELECT 
    OBJECT_SCHEMA as db,
    OBJECT_NAME as table_name,
    COUNT_STAR as total_io_ops,
    COUNT_READ as read_ops,
    COUNT_WRITE as write_ops,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) as total_io_time_sec,
    ROUND(SUM_TIMER_READ / 1000000000000, 3) as total_read_time_sec,
    ROUND(SUM_TIMER_WRITE / 1000000000000, 3) as total_write_time_sec,
    ROUND((SUM_TIMER_WAIT / COUNT_STAR) / 1000000000, 3) as avg_io_time_ms
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Détail read vs write par table
SELECT 
    OBJECT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    ROUND(COUNT_READ * 100.0 / NULLIF(COUNT_STAR, 0), 2) as read_pct,
    ROUND(COUNT_WRITE * 100.0 / NULLIF(COUNT_STAR, 0), 2) as write_pct
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA = 'mydb'
ORDER BY COUNT_STAR DESC
LIMIT 20;
```

### 4. table_lock_waits_summary_by_table

**Locks par table** : Identifier contention.

```sql
-- Tables avec le plus de contention de locks
SELECT 
    OBJECT_SCHEMA as db,
    OBJECT_NAME as table_name,
    COUNT_STAR as total_locks,
    COUNT_READ as read_locks,
    COUNT_WRITE as write_locks,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) as total_wait_sec,
    ROUND(SUM_TIMER_READ_WITH_SHARED_LOCKS / 1000000000000, 3) as shared_lock_wait_sec,
    ROUND(SUM_TIMER_WRITE / 1000000000000, 3) as write_lock_wait_sec
FROM performance_schema.table_lock_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema')
AND SUM_TIMER_WAIT > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### 5. file_summary_by_instance

**I/O fichiers** : Performance au niveau système de fichiers.

```sql
-- Top 10 fichiers par I/O
SELECT 
    FILE_NAME,
    EVENT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    ROUND(SUM_NUMBER_OF_BYTES_READ / 1024 / 1024, 2) as read_mb,
    ROUND(SUM_NUMBER_OF_BYTES_WRITE / 1024 / 1024, 2) as write_mb,
    ROUND((SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE) / 1024 / 1024, 2) as total_mb,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) as total_wait_sec
FROM performance_schema.file_summary_by_instance
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- I/O par type de fichier
SELECT 
    SUBSTRING_INDEX(FILE_NAME, '/', -1) as filename,
    CASE 
        WHEN FILE_NAME LIKE '%.ibd' THEN 'InnoDB Data'
        WHEN FILE_NAME LIKE 'ib_logfile%' THEN 'InnoDB Redo Log'
        WHEN FILE_NAME LIKE 'binlog%' THEN 'Binary Log'
        WHEN FILE_NAME LIKE '%slow.log%' THEN 'Slow Query Log'
        ELSE 'Other'
    END as file_type,
    COUNT_READ + COUNT_WRITE as total_ops,
    ROUND((SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE) / 1024 / 1024, 2) as total_mb
FROM performance_schema.file_summary_by_instance
ORDER BY total_mb DESC
LIMIT 20;
```

### 6. memory_summary_by_thread_by_event_name

**Consommation mémoire** par thread et type d'événement.

```sql
-- Top consommateurs de mémoire
SELECT 
    THREAD_ID,
    EVENT_NAME,
    CURRENT_COUNT_USED as current_allocations,
    ROUND(CURRENT_NUMBER_OF_BYTES_USED / 1024 / 1024, 2) as current_mb,
    HIGH_COUNT_USED as peak_allocations,
    ROUND(HIGH_NUMBER_OF_BYTES_USED / 1024 / 1024, 2) as peak_mb
FROM performance_schema.memory_summary_by_thread_by_event_name
WHERE CURRENT_NUMBER_OF_BYTES_USED > 0
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC
LIMIT 20;

-- Mémoire totale par type d'événement
SELECT 
    EVENT_NAME,
    SUM(CURRENT_COUNT_USED) as total_allocations,
    ROUND(SUM(CURRENT_NUMBER_OF_BYTES_USED) / 1024 / 1024, 2) as total_mb
FROM performance_schema.memory_summary_by_thread_by_event_name
GROUP BY EVENT_NAME
ORDER BY total_mb DESC
LIMIT 20;
```

---

## sys Schema : Vues simplifiées

### Introduction à sys schema

```sql
-- sys schema fournit des vues lisibles sur Performance Schema

-- Lister toutes les vues sys
SELECT TABLE_NAME 
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'sys'
AND TABLE_TYPE = 'VIEW'
ORDER BY TABLE_NAME;

-- 100+ vues organisées par catégories :
-- • statement_analysis : Analyse requêtes
-- • schema_* : Analyse schéma (tables, index)
-- • io_* : Analyse I/O
-- • memory_* : Analyse mémoire
-- • user_* : Analyse par utilisateur
-- • host_* : Analyse par host
```

### Vues critiques sys schema

#### statement_analysis

```sql
-- Vue la plus utilisée : Analyse complète des requêtes
SELECT 
    query,
    db,
    exec_count,
    total_latency,
    avg_latency,
    lock_latency,
    rows_sent,
    rows_examined,
    rows_affected,
    full_scan
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 10;

-- Résultat formaté lisible (latency en format humain)
/*
+----------------------------------------+-------+------------+----------------+--------------+
| query                                  | db    | exec_count | total_latency  | avg_latency  |
+----------------------------------------+-------+------------+----------------+--------------+
| SELECT * FROM orders WHERE customer... | shop  | 12500      | 2.5 h          | 720.5 ms     |
| SELECT p . * FROM products p JOIN ...  | shop  | 500        | 52.1 min       | 6.25 s       |
+----------------------------------------+-------+------------+----------------+--------------+
*/
```

#### statements_with_full_table_scans

```sql
-- Requêtes faisant des full table scans
SELECT 
    query,
    db,
    exec_count,
    total_latency,
    no_index_used_count,
    no_good_index_used_count,
    rows_sent,
    rows_examined
FROM sys.statements_with_full_table_scans
ORDER BY total_latency DESC
LIMIT 20;

-- Indicateur fort de problème de performance !
```

#### statements_with_temp_tables

```sql
-- Requêtes créant des tables temporaires
SELECT 
    query,
    db,
    exec_count,
    total_latency,
    memory_tmp_tables,
    disk_tmp_tables,
    avg_tmp_tables_per_query,
    tmp_tables_to_disk_pct
FROM sys.statements_with_temp_tables
WHERE tmp_tables_to_disk_pct > 0
ORDER BY disk_tmp_tables DESC
LIMIT 20;

-- disk_tmp_tables > 0 = Problème !
-- Tables temp sur disque sont très lentes
```

#### schema_table_statistics

```sql
-- Statistiques détaillées par table
SELECT 
    table_schema,
    table_name,
    rows_fetched,
    rows_inserted,
    rows_updated,
    rows_deleted,
    io_read,
    io_write,
    io_read_latency,
    io_write_latency
FROM sys.schema_table_statistics
WHERE table_schema = 'mydb'
ORDER BY io_read_latency + io_write_latency DESC
LIMIT 20;
```

#### schema_unused_indexes

```sql
-- Index jamais utilisés (candidats à suppression)
SELECT 
    object_schema as db,
    object_name as table_name,
    index_name
FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY object_schema, object_name;

-- Attention : Vérifier sur période longue (30+ jours)
-- Ne pas supprimer index utilisés périodiquement (fin de mois, etc.)
```

#### schema_redundant_indexes

```sql
-- Index redondants (waste de ressources)
SELECT 
    table_schema,
    table_name,
    redundant_index_name,
    redundant_index_columns,
    dominant_index_name,
    dominant_index_columns,
    sql_drop_index
FROM sys.schema_redundant_indexes
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'sys');

-- Exemple :
-- Index (col1, col2) rend redondant index (col1)
-- Supprimer index (col1)
```

#### io_global_by_file_by_latency

```sql
-- I/O par fichier avec latence
SELECT 
    file,
    total,
    total_latency,
    read_latency,
    write_latency
FROM sys.io_global_by_file_by_latency
LIMIT 20;

-- Identifier fichiers avec I/O lents
```

#### user_summary

```sql
-- Activité par utilisateur
SELECT 
    user,
    statements,
    statement_latency,
    table_scans,
    file_ios,
    file_io_latency,
    current_connections,
    total_connections
FROM sys.user_summary
ORDER BY statement_latency DESC;
```

#### memory_by_thread_by_current_bytes

```sql
-- Mémoire utilisée par thread
SELECT 
    thread_id,
    user,
    current_allocated,
    current_max_alloc,
    total_allocated
FROM sys.memory_by_thread_by_current_bytes
ORDER BY current_allocated DESC
LIMIT 20;
```

---

## Procédures sys schema

### ps_setup_show_enabled

```sql
-- Voir la configuration actuelle de Performance Schema
CALL sys.ps_setup_show_enabled(TRUE, TRUE);

-- Affiche :
-- • Instruments activés
-- • Consumers activés
-- • Configuration actors
```

### ps_trace_thread

```sql
-- Tracer l'activité d'un thread spécifique

-- 1. Identifier le thread
SELECT thread_id, processlist_id 
FROM performance_schema.threads 
WHERE processlist_user = 'myapp';

-- 2. Activer tracing
CALL sys.ps_trace_thread(123, '/tmp/trace.txt', NULL, NULL, TRUE, TRUE, TRUE, TRUE);
-- 123 = thread_id
-- Génère fichier avec détail complet des opérations

-- 3. Analyser le fichier trace
-- cat /tmp/trace.txt
```

### ps_statement_avg_latency_histogram

```sql
-- Histogramme de latence des requêtes
CALL sys.ps_statement_avg_latency_histogram();

-- Affiche distribution :
-- 0-1ms : ████████████████ 65%
-- 1-10ms : ████████ 25%
-- 10-100ms : ███ 8%
-- >100ms : █ 2%
```

---

## Cas d'usage pratiques

### Diagnostic 1 : Identifier requête lente en cours

```sql
-- Scénario : "Le serveur est lent en ce moment"

-- Étape 1 : Requêtes en cours triées par durée
SELECT 
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_DB,
    t.PROCESSLIST_TIME as running_sec,
    SUBSTRING(s.SQL_TEXT, 1, 200) as query,
    s.ROWS_EXAMINED,
    s.ROWS_SENT,
    s.LOCK_TIME / 1000000000000 as lock_sec
FROM performance_schema.threads t
JOIN performance_schema.events_statements_current s 
    ON t.THREAD_ID = s.THREAD_ID
WHERE t.PROCESSLIST_COMMAND != 'Sleep'
ORDER BY t.PROCESSLIST_TIME DESC
LIMIT 5;

-- Étape 2 : Si lock_sec élevé, vérifier les locks
SELECT 
    r.trx_id as blocking_trx,
    r.trx_mysql_thread_id as blocking_thread,
    SUBSTRING(r.trx_query, 1, 100) as blocking_query,
    b.trx_id as blocked_trx,
    b.trx_mysql_thread_id as blocked_thread,
    SUBSTRING(b.trx_query, 1, 100) as blocked_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.requesting_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.blocking_trx_id;

-- Étape 3 : Tuer requête si nécessaire
KILL 12345;  -- PROCESSLIST_ID de la requête problématique
```

### Diagnostic 2 : Analyser performance sur 24h

```sql
-- Collecter métriques sur 24h puis analyser

-- Début de journée : Réinitialiser compteurs
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;

-- Fin de journée : Analyser
SELECT 
    SCHEMA_NAME,
    SUBSTRING(DIGEST_TEXT, 1, 100) as query,
    COUNT_STAR as executions,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) as avg_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) as total_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000 / 86400 * 100, 2) as pct_of_day,
    SUM_ROWS_EXAMINED as rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Exporter dans rapport
SELECT * FROM sys.statement_analysis
INTO OUTFILE '/tmp/daily_report.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

### Diagnostic 3 : Table la plus lente

```sql
-- Identifier la table causant le plus de temps I/O

SELECT 
    OBJECT_SCHEMA as db,
    OBJECT_NAME as table_name,
    COUNT_STAR as io_ops,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) as total_io_sec,
    ROUND(SUM_TIMER_WAIT / COUNT_STAR / 1000000000, 3) as avg_io_ms,
    COUNT_READ,
    COUNT_WRITE
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 1;

-- Analyser index sur cette table
SELECT * FROM sys.schema_redundant_indexes
WHERE table_schema = 'mydb' AND table_name = 'orders';

SELECT * FROM sys.schema_unused_indexes
WHERE object_schema = 'mydb' AND object_name = 'orders';

-- Analyser requêtes sur cette table
SELECT 
    SUBSTRING(DIGEST_TEXT, 1, 200) as query,
    COUNT_STAR as exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) as avg_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%orders%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### Diagnostic 4 : Memory leak detection

```sql
-- Identifier threads consommant trop de mémoire

SELECT 
    t.THREAD_ID,
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_HOST,
    t.PROCESSLIST_TIME as duration_sec,
    ROUND(SUM(m.CURRENT_NUMBER_OF_BYTES_USED) / 1024 / 1024, 2) as memory_mb,
    SUBSTRING(s.SQL_TEXT, 1, 100) as current_query
FROM performance_schema.threads t
LEFT JOIN performance_schema.memory_summary_by_thread_by_event_name m 
    ON t.THREAD_ID = m.THREAD_ID
LEFT JOIN performance_schema.events_statements_current s
    ON t.THREAD_ID = s.THREAD_ID
WHERE t.PROCESSLIST_ID IS NOT NULL
GROUP BY t.THREAD_ID
HAVING memory_mb > 100  -- Seuil : 100 MB
ORDER BY memory_mb DESC;
```

---

## Automatisation et alerting

### Créer vues de monitoring personnalisées

```sql
-- Vue : Top 10 requêtes lentes actuelles
CREATE OR REPLACE VIEW monitoring.current_slow_queries AS
SELECT 
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_TIME as duration_sec,
    SUBSTRING(s.SQL_TEXT, 1, 200) as query,
    s.TIMER_WAIT / 1000000000000 as elapsed_sec,
    s.ROWS_EXAMINED
FROM performance_schema.threads t
JOIN performance_schema.events_statements_current s 
    ON t.THREAD_ID = s.THREAD_ID
WHERE t.PROCESSLIST_COMMAND != 'Sleep'
AND s.TIMER_WAIT / 1000000000000 > 1  -- > 1 seconde
ORDER BY s.TIMER_WAIT DESC
LIMIT 10;

-- Utiliser
SELECT * FROM monitoring.current_slow_queries;
```

### Script d'alerting

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE monitoring.alert_performance_issues()
BEGIN
    DECLARE v_slow_count INT;
    DECLARE v_full_scans BIGINT;
    DECLARE v_disk_tmp BIGINT;
    
    -- 1. Vérifier requêtes lentes en cours
    SELECT COUNT(*) INTO v_slow_count
    FROM performance_schema.events_statements_current
    WHERE TIMER_WAIT / 1000000000000 > 5;
    
    IF v_slow_count > 10 THEN
        SELECT CONCAT('ALERT: ', v_slow_count, ' requêtes > 5s en cours') as alert;
    END IF;
    
    -- 2. Vérifier full table scans
    SELECT SUM(exec_count) INTO v_full_scans
    FROM sys.statements_with_full_table_scans
    WHERE last_seen > DATE_SUB(NOW(), INTERVAL 1 HOUR);
    
    IF v_full_scans > 1000 THEN
        SELECT CONCAT('ALERT: ', v_full_scans, ' full scans dernière heure') as alert;
    END IF;
    
    -- 3. Vérifier temp tables sur disque
    SELECT SUM(disk_tmp_tables) INTO v_disk_tmp
    FROM sys.statements_with_temp_tables
    WHERE last_seen > DATE_SUB(NOW(), INTERVAL 1 HOUR);
    
    IF v_disk_tmp > 100 THEN
        SELECT CONCAT('ALERT: ', v_disk_tmp, ' temp tables sur disque dernière heure') as alert;
    END IF;
END //
DELIMITER ;

-- Appeler toutes les 5 minutes via cron
-- */5 * * * * mysql -e "CALL monitoring.alert_performance_issues();"
```

### Rapport quotidien automatique

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE monitoring.daily_performance_report()
BEGIN
    -- Rapport journalier de performance
    
    SELECT '=== TOP 10 REQUÊTES PAR TEMPS CUMULÉ ===' as section;
    SELECT 
        query,
        exec_count,
        total_latency,
        avg_latency,
        rows_examined
    FROM sys.statement_analysis
    ORDER BY total_latency DESC
    LIMIT 10;
    
    SELECT '=== REQUÊTES AVEC FULL TABLE SCANS ===' as section;
    SELECT 
        query,
        exec_count,
        total_latency,
        no_index_used_count
    FROM sys.statements_with_full_table_scans
    ORDER BY exec_count DESC
    LIMIT 10;
    
    SELECT '=== TOP 10 TABLES PAR I/O ===' as section;
    SELECT 
        table_schema,
        table_name,
        io_read + io_write as total_io,
        io_read_latency,
        io_write_latency
    FROM sys.schema_table_statistics
    ORDER BY io_read_latency + io_write_latency DESC
    LIMIT 10;
    
    SELECT '=== INDEX INUTILISÉS ===' as section;
    SELECT 
        object_schema,
        object_name,
        index_name
    FROM sys.schema_unused_indexes;
    
    SELECT '=== CONSOMMATION MÉMOIRE PAR UTILISATEUR ===' as section;
    SELECT 
        user,
        current_allocated,
        total_allocated
    FROM sys.memory_by_user_by_current_bytes;
END //
DELIMITER ;

-- Exécuter quotidiennement
-- 0 6 * * * mysql -e "CALL monitoring.daily_performance_report();" > /var/log/mysql/daily_report_$(date +\%Y\%m\%d).txt
```

---

## Best Practices

### 1. Configuration graduée selon environnement

```sql
-- DÉVELOPPEMENT : Instrumentation maximale
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES';
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';

-- STAGING : Instrumentation complète mais ciblée
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES'
WHERE NAME IN ('events_statements_current', 'events_statements_history_long', 'statements_digest');
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%' OR NAME LIKE 'wait/io/%';

-- PRODUCTION : Instrumentation minimale
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES'
WHERE NAME IN ('events_statements_current', 'statements_digest');
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';
```

### 2. Purge régulière des données historiques

```sql
-- Éviter accumulation excessive

-- Réinitialiser digest hebdomadairement
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;

-- Ou créer procédure de rotation
DELIMITER //
CREATE OR REPLACE PROCEDURE monitoring.rotate_perf_schema()
BEGIN
    -- Sauvegarder dans table historique
    INSERT INTO monitoring.statements_history
    SELECT NOW(), s.*
    FROM performance_schema.events_statements_summary_by_digest s
    WHERE LAST_SEEN < DATE_SUB(NOW(), INTERVAL 7 DAY);
    
    -- Purger anciennes données
    TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
END //
DELIMITER ;

-- Exécuter hebdomadairement
```

### 3. Monitoring continu léger

```sql
-- Créer événement (scheduler) pour monitoring

SET GLOBAL event_scheduler = ON;

DELIMITER //
CREATE EVENT IF NOT EXISTS monitoring.hourly_check
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    -- Vérifier et alerter si problème
    CALL monitoring.alert_performance_issues();
END //
DELIMITER ;
```

### 4. Documentation des changements

```sql
-- Logger toute modification de configuration

CREATE TABLE monitoring.perf_schema_config_log (
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100),
    change_type ENUM('instrument', 'consumer', 'actor'),
    object_name VARCHAR(255),
    old_value VARCHAR(10),
    new_value VARCHAR(10),
    reason TEXT
);

-- Trigger sur modifications (exemple)
-- À adapter selon besoins
```

---

## ✅ Points clés à retenir

- 🎯 **Performance Schema = monitoring temps réel** avec overhead minimal (<5%)
- 📊 **sys schema = simplification** de Performance Schema pour analyses rapides
- 🔍 **events_statements_summary_by_digest** = table la plus importante
- ⚡ **Temps cumulé** = métrique #1 (exec_count × avg_time)
- 📈 **Full table scans** = indicateur critique à surveiller
- 💾 **Index inutilisés** = sys.schema_unused_indexes pour optimisation
- 🔧 **Configuration graduée** : Dev (max) → Staging (complet) → Prod (minimal)
- 🔄 **Purge régulière** : Éviter accumulation de données
- 📝 **Automatisation** : Rapports et alerting quotidiens
- ✅ **Complémentaire** : PS temps réel + Slow log post-mortem

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [📖 sys Schema](https://mariadb.com/kb/en/sys-schema/)
- [📖 Performance Schema Tables](https://mariadb.com/kb/en/performance-schema-tables/)

### Documentation MySQL (compatible)

- [MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)
- [MySQL sys Schema](https://dev.mysql.com/doc/refman/8.0/en/sys-schema.html)

### Blogs et tutoriels

- [Percona Blog - Performance Schema](https://www.percona.com/blog/category/mysql/performance-schema/)
- [MySQL Performance Blog](https://www.percona.com/blog/tag/performance-schema/)

---

## ➡️ Section suivante

**[15.9 Partitionnement](/15-performance-tuning/09-partitionnement-tables.md)** : Techniques avancées de partitionnement (RANGE, LIST, HASH) pour gérer de très grandes tables et améliorer les performances des requêtes ciblées.

---

*Performance Schema et sys schema sont des outils puissants et sous-utilisés. Maîtriser leur utilisation transforme le diagnostic de performance de "divination" en science exacte.*

⏭️ [Partitionnement de tables](/15-performance-tuning/09-partitionnement-tables.md)

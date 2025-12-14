🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.1 Audit de Configuration

> **Type** : Checklist d'audit serveur  
> **Focus** : Paramètres my.cnf, ressources système, durabilité  
> **Durée** : 30-60 minutes  
> **Prérequis** : Accès serveur, privilèges SUPER

---

## 🎯 Objectifs de cet audit

Vérifier que la **configuration serveur MariaDB** est optimale pour :
- ✅ Utilisation efficace des ressources (RAM, CPU, I/O)
- ✅ Performance adaptée au cas d'usage (OLTP, OLAP, Mixed)
- ✅ Durabilité et fiabilité des données
- ✅ Évolutivité et scalabilité

**Résultat attendu :** Liste priorisée d'actions correctives avec impact estimé.

---

## 📋 Checklist complète (20 points)

### Catégorie 1 : Mémoire et Buffer Pool

#### ✅ 1.1 Taille du Buffer Pool InnoDB

**🔍 Diagnostic**

```sql
-- Taille actuelle buffer pool
SELECT 
    @@innodb_buffer_pool_size / 1024 / 1024 / 1024 AS buffer_pool_gb,
    (SELECT SUM(data_length + index_length) / 1024 / 1024 / 1024 
     FROM information_schema.tables 
     WHERE engine = 'InnoDB') AS innodb_data_gb,
    @@innodb_buffer_pool_instances AS instances;

-- Utilisation buffer pool
SELECT 
    ROUND(100.0 * (1 - (
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_reads') / 
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_read_requests')
    )), 4) AS buffer_pool_hit_rate_pct;

-- RAM système disponible
-- Via shell : free -h
```

**⚠️ Seuils critiques**

| Métrique | 🟢 Optimal | 🟡 Acceptable | 🔴 Critique |
|----------|-----------|---------------|-------------|
| **Buffer pool / RAM totale** | 60-70% (OLTP)<br>70-80% (OLAP) | 50-60% | < 50% ou > 85% |
| **Hit rate** | > 99% (OLTP)<br>> 95% (OLAP) | 95-99% | < 95% |
| **Buffer pool / Données InnoDB** | > 80% | 50-80% | < 50% |

**🔧 Actions correctives**

1. **Hit rate < 95%** : Augmenter `innodb_buffer_pool_size`
   ```ini
   # my.cnf
   innodb_buffer_pool_size = 20G  # Ajuster selon RAM
   
   # Ou dynamique (MariaDB 10.2.2+)
   SET GLOBAL innodb_buffer_pool_size = 21474836480;  -- 20GB
   ```

2. **Buffer pool > 85% RAM** : Risque OOM (Out of Memory)
   ```ini
   # Réduire buffer pool OU augmenter RAM serveur
   innodb_buffer_pool_size = 16G  # Si RAM = 24GB
   ```

3. **Instances sub-optimales** :
   ```ini
   # Règle : 1 instance par 1-2GB, max = nb CPU cores
   innodb_buffer_pool_instances = 16  # Pour 32GB buffer pool
   ```

**📊 Validation**

```sql
-- Après changement, surveiller 24-48h
SELECT 
    NOW() AS check_time,
    ROUND(100.0 * (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)), 2) AS hit_rate
FROM (
    SELECT 
        CAST((SELECT variable_value FROM information_schema.global_status 
              WHERE variable_name = 'Innodb_buffer_pool_reads') AS UNSIGNED) AS Innodb_buffer_pool_reads,
        CAST((SELECT variable_value FROM information_schema.global_status 
              WHERE variable_name = 'Innodb_buffer_pool_read_requests') AS UNSIGNED) AS Innodb_buffer_pool_read_requests
) AS stats;
```

---

#### ✅ 1.2 Buffers par connexion

**🔍 Diagnostic**

```sql
-- Mémoire allouée par connexion
SELECT 
    @@max_connections AS max_conn,
    @@read_buffer_size / 1024 / 1024 AS read_buffer_mb,
    @@sort_buffer_size / 1024 / 1024 AS sort_buffer_mb,
    @@join_buffer_size / 1024 / 1024 AS join_buffer_mb,
    @@read_rnd_buffer_size / 1024 / 1024 AS read_rnd_buffer_mb,
    ((@@read_buffer_size + @@sort_buffer_size + @@join_buffer_size + @@read_rnd_buffer_size) 
     * @@max_connections / 1024 / 1024 / 1024) AS max_memory_for_connections_gb;

-- Connexions actuelles vs max
SELECT 
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Threads_connected') AS current_connections,
    @@max_connections AS max_connections,
    ROUND(100.0 * (SELECT variable_value FROM information_schema.global_status 
                   WHERE variable_name = 'Threads_connected') / @@max_connections, 2) AS pct_used;
```

**⚠️ Seuils critiques**

| Paramètre | 🟢 OLTP | 🟢 OLAP | 🟢 Dev | 🔴 Problème |
|-----------|---------|---------|--------|-------------|
| `sort_buffer_size` | 256K-512K | 32M-64M | 256K | > 128M |
| `join_buffer_size` | 256K | 32M-64M | 128K | > 128M |
| `read_buffer_size` | 256K | 4M-8M | 128K | > 16M |
| `max_connections` | 200-500 | 50-100 | 50 | > 1000 |

**🔧 Actions correctives**

1. **Buffers trop élevés** (mémoire gaspillée) :
   ```ini
   # OLTP optimisé
   sort_buffer_size = 512K
   join_buffer_size = 256K
   read_buffer_size = 256K
   read_rnd_buffer_size = 512K
   ```

2. **max_connections surdimensionné** :
   ```sql
   -- Analyser pic réel connexions
   SELECT MAX(variable_value) AS max_seen
   FROM (
       SELECT variable_value 
       FROM information_schema.global_status 
       WHERE variable_name = 'Threads_connected'
   ) AS conn_history;
   
   -- Ajuster max_connections = max_seen × 1.5
   SET GLOBAL max_connections = 300;
   ```

3. **OLAP avec buffers OLTP** :
   ```ini
   # OLAP nécessite gros buffers
   sort_buffer_size = 64M
   join_buffer_size = 64M
   tmp_table_size = 1G
   ```

**📊 Validation**

```sql
-- Utilisation réelle buffers (Performance Schema requis)
SELECT 
    EVENT_NAME,
    COUNT_ALLOC AS allocations,
    ROUND(SUM_NUMBER_OF_BYTES_ALLOC / 1024 / 1024, 2) AS allocated_mb
FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE '%sort%' OR EVENT_NAME LIKE '%join%'
ORDER BY SUM_NUMBER_OF_BYTES_ALLOC DESC
LIMIT 10;
```

---

### Catégorie 2 : I/O et Disques

#### ✅ 2.1 I/O Capacity (IOPS)

**🔍 Diagnostic**

```sql
-- Configuration actuelle
SELECT 
    @@innodb_io_capacity AS io_capacity,
    @@innodb_io_capacity_max AS io_capacity_max,
    @@innodb_flush_method AS flush_method,
    @@innodb_flush_neighbors AS flush_neighbors;

-- Activité I/O actuelle
SHOW GLOBAL STATUS LIKE 'Innodb_data%';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_flushed';
```

**⚠️ Seuils critiques**

| Type disque | `innodb_io_capacity` | `innodb_io_capacity_max` |
|-------------|----------------------|--------------------------|
| **HDD 7200rpm** | 100-200 | 200-400 |
| **SSD SATA** | 2000-5000 | 4000-10000 |
| **NVMe Gen3** | 10000-20000 | 20000-40000 |
| **NVMe Gen4** | 30000-50000 | 60000-100000 |

**🔧 Actions correctives**

1. **Mesurer IOPS réels du disque** :
   ```bash
   # Test avec fio
   fio --name=random-write --ioengine=libaio --iodepth=32 \
       --rw=randwrite --bs=16k --direct=1 --size=4G \
       --numjobs=4 --runtime=60 --group_reporting
   
   # Observer IOPS dans résultats
   # write: IOPS=15234, BW=237MiB/s (249MB/s)
   ```

2. **Ajuster selon disque** :
   ```ini
   # SSD NVMe détecté avec 15000 IOPS
   innodb_io_capacity = 15000
   innodb_io_capacity_max = 30000
   
   # 🆕 MariaDB 11.8 : Optimisation SSD
   innodb_flush_neighbors = 0  # SSD : pas de flush voisins
   innodb_flush_method = O_DIRECT
   ```

3. **HDD en production (⚠️ à éviter)** :
   ```ini
   # Si HDD obligatoire (non recommandé OLTP)
   innodb_io_capacity = 200
   innodb_io_capacity_max = 400
   innodb_flush_neighbors = 1  # HDD bénéficie flush séquentiel
   ```

**📊 Validation**

```bash
# Monitorer I/O wait
iostat -x 5 3

# Si %iowait > 20% constant : disques saturés
# Action : Upgrade disque ou réduire charge
```

---

#### ✅ 2.2 Redo Log

**🔍 Diagnostic**

```sql
-- Configuration redo log
SELECT 
    @@innodb_log_file_size / 1024 / 1024 AS log_file_size_mb,
    @@innodb_log_files_in_group AS log_files,
    (@@innodb_log_file_size * @@innodb_log_files_in_group / 1024 / 1024) AS total_redo_log_mb,
    @@innodb_log_buffer_size / 1024 / 1024 AS log_buffer_mb;

-- Fréquence checkpoints
SHOW GLOBAL STATUS LIKE 'Innodb_checkpoint%';
```

**⚠️ Seuils critiques**

| Cas d'usage | Total redo log | Log buffer |
|-------------|----------------|------------|
| **OLTP** | 1-2 GB | 64-128 MB |
| **OLAP** | 2-4 GB | 128-256 MB |
| **Dev** | 256-512 MB | 8-16 MB |

**🔧 Actions correctives**

1. **Redo log trop petit** (checkpoints fréquents) :
   ```ini
   # OLTP production
   innodb_log_file_size = 1G
   innodb_log_files_in_group = 2
   # Total = 2GB
   
   # ⚠️ Redémarrage requis pour changer log_file_size
   ```

2. **Redo log trop grand** (recovery lent) :
   ```ini
   # Si recovery > 5 minutes inacceptable
   innodb_log_file_size = 512M
   # Compromis : écriture vs recovery
   ```

3. **Log buffer sous-dimensionné** :
   ```sql
   -- Vérifier log waits
   SHOW GLOBAL STATUS LIKE 'Innodb_log_waits';
   -- Si > 0 : augmenter log_buffer_size
   
   SET GLOBAL innodb_log_buffer_size = 134217728;  -- 128MB
   ```

**📊 Validation**

```sql
-- Checkpoints toutes les X minutes (objectif : 15-30 min)
-- Calculer via logs ou monitoring
SHOW ENGINE INNODB STATUS\G
-- Regarder "LOG" section
```

---

### Catégorie 3 : Durabilité et Logs

#### ✅ 3.1 Flush Log at Transaction Commit

**🔍 Diagnostic**

```sql
SELECT 
    @@innodb_flush_log_at_trx_commit AS flush_log,
    @@sync_binlog AS sync_binlog,
    CASE @@innodb_flush_log_at_trx_commit
        WHEN 0 THEN 'DANGER: No ACID (flush 1×/sec)'
        WHEN 1 THEN 'SAFE: Full ACID'
        WHEN 2 THEN 'RISK: OS cache (crash OS = perte)'
    END AS durability_level;
```

**⚠️ Seuils critiques**

| Environnement | `flush_log` | `sync_binlog` | Durabilité |
|---------------|-------------|---------------|------------|
| **Production OLTP** | 1 | 1 | ✅ ACID strict |
| **Production OLAP** | 1-2 | 0-1 | ✅ Acceptable |
| **Staging** | 2 | 0 | ⚠️ Compromis |
| **Dev** | 0 | 0 | ❌ Perf > Durabilité |

**🔧 Actions correctives**

1. **Production avec flush_log = 0 ou 2** (🔴 DANGER) :
   ```ini
   # TOUJOURS = 1 en production
   innodb_flush_log_at_trx_commit = 1
   sync_binlog = 1
   
   # Perte performance : 30-40%
   # MAIS : Aucune perte données en cas crash
   ```

2. **Dev avec flush_log = 1** (gaspillage) :
   ```ini
   # Dev : Performance > Durabilité
   innodb_flush_log_at_trx_commit = 0
   sync_binlog = 0
   skip-log-bin = 1  # Pas de binlog dev
   ```

3. **OLAP avec données rechargeable** :
   ```ini
   # Si crash → Re-ETL acceptable
   innodb_flush_log_at_trx_commit = 2
   # Gain : 15-25% perf writes
   ```

**📊 Validation**

```bash
# Benchmark impact
sysbench oltp_write_only \
  --mysql-user=root --mysql-password=xxx \
  --table-size=100000 --threads=4 --time=60 \
  prepare

# Test avec flush_log=1
sysbench oltp_write_only ... run

# Test avec flush_log=2
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
sysbench oltp_write_only ... run

# Comparer TPS (transactions/sec)
```

---

#### ✅ 3.2 Binary Logs

**🔍 Diagnostic**

```sql
-- Binlog activé ?
SELECT 
    @@log_bin AS binlog_enabled,
    @@binlog_format AS format,
    @@max_binlog_size / 1024 / 1024 AS max_binlog_mb,
    @@expire_logs_days AS retention_days;

-- Espace utilisé binlog
SELECT 
    ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2) AS binlog_total_gb
FROM information_schema.binary_logs;

-- Liste binlogs
SHOW BINARY LOGS;
```

**⚠️ Seuils critiques**

| Métrique | 🟢 Optimal | 🟡 Acceptable | 🔴 Critique |
|----------|-----------|---------------|-------------|
| **Espace binlog** | < 10% disque | 10-20% | > 20% disque |
| **Retention** | 3-7 jours (prod) | 7-14 jours | > 30 jours |
| **Format** | ROW | MIXED | STATEMENT |

**🔧 Actions correctives**

1. **Binlog consomme trop d'espace** :
   ```sql
   -- Purger manuellement
   PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);
   
   -- Ou automatique
   SET GLOBAL expire_logs_days = 3;
   ```

2. **Format STATEMENT** (risqué réplication) :
   ```ini
   # my.cnf
   binlog_format = ROW  # Plus sûr pour réplication
   binlog_row_image = MINIMAL  # Économie espace
   ```

3. **Binlog inutile** (pas de réplication, pas de PITR) :
   ```ini
   # Dev ou OLAP standalone
   skip-log-bin = 1
   # Gain : 10-20% perf writes + espace disque
   ```

**📊 Validation**

```sql
-- Croissance binlog par jour
SELECT 
    DATE(FROM_UNIXTIME(file_time)) AS day,
    ROUND(SUM(file_size) / 1024 / 1024 / 1024, 2) AS gb_per_day
FROM (
    SELECT 
        file_name,
        file_size,
        UNIX_TIMESTAMP(STR_TO_DATE(
            SUBSTRING_INDEX(file_name, '.', -1), 
            '%Y%m%d'
        )) AS file_time
    FROM information_schema.binary_logs
) AS binlog_info
GROUP BY day
ORDER BY day DESC;
```

---

### Catégorie 4 : Connexions et Threads

#### ✅ 4.1 Thread Pool vs One-Thread-Per-Connection

**🔍 Diagnostic**

```sql
SELECT 
    @@thread_handling AS handling,
    @@thread_pool_size AS pool_size,
    @@max_connections AS max_conn,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Threads_connected') AS current_conn,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Threads_running') AS threads_running;
```

**⚠️ Seuils critiques**

| Cas d'usage | `thread_handling` | `thread_pool_size` | `max_connections` |
|-------------|-------------------|---------------------|-------------------|
| **OLTP** | pool-of-threads | = nb CPU cores | 200-500 |
| **OLAP** | one-thread-per-connection | N/A | 50-100 |
| **Mixed** | pool-of-threads | = nb CPU cores | 200-300 |
| **Dev** | one-thread-per-connection | N/A | 50 |

**🔧 Actions correctives**

1. **OLTP sans thread pool** (scalabilité limitée) :
   ```ini
   # Activer thread pool (redémarrage requis)
   thread_handling = pool-of-threads
   thread_pool_size = 8  # = nb CPU cores
   thread_pool_max_threads = 1000
   thread_pool_stall_limit = 500
   ```

2. **thread_pool_size inadapté** :
   ```bash
   # Vérifier nb cores
   lscpu | grep "^CPU(s):"
   # CPU(s): 16
   
   # Ajuster
   SET GLOBAL thread_pool_size = 16;
   ```

3. **max_connections trop élevé** :
   ```sql
   -- Analyser historique
   SELECT MAX(variable_value) AS peak_connections
   FROM mysql.global_status_history  -- Si activé
   WHERE variable_name = 'Threads_connected';
   
   -- Ajuster max_connections = peak × 1.5
   ```

**📊 Validation**

```sql
-- Thread pool efficiency
SELECT 
    tp.pool_id,
    tp.num_threads,
    tp.active_threads,
    tp.stalled_queries
FROM information_schema.thread_pool_queues tp;

-- Si stalled_queries élevé : augmenter thread_pool_max_threads
```

---

#### ✅ 4.2 Table Open Cache

**🔍 Diagnostic**

```sql
SELECT 
    @@table_open_cache AS cache_size,
    @@table_definition_cache AS def_cache,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Open_tables') AS open_tables,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Opened_tables') AS opened_tables,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Table_open_cache_hits') AS cache_hits,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Table_open_cache_misses') AS cache_misses;

-- Hit rate
SELECT ROUND(100.0 * cache_hits / (cache_hits + cache_misses), 2) AS hit_rate_pct
FROM (
    SELECT 
        CAST((SELECT variable_value FROM information_schema.global_status 
              WHERE variable_name = 'Table_open_cache_hits') AS UNSIGNED) AS cache_hits,
        CAST((SELECT variable_value FROM information_schema.global_status 
              WHERE variable_name = 'Table_open_cache_misses') AS UNSIGNED) AS cache_misses
) AS cache_stats;
```

**⚠️ Seuils critiques**

| Métrique | 🟢 Optimal | 🟡 Acceptable | 🔴 Critique |
|----------|-----------|---------------|-------------|
| **Hit rate** | > 95% | 90-95% | < 90% |
| **table_open_cache** | 2000-4000 (OLTP) | 1000-2000 | < 400 |
| **Open tables vs cache** | 60-80% | 80-95% | > 95% (saturation) |

**🔧 Actions correctives**

1. **Hit rate < 90%** :
   ```ini
   # Augmenter cache
   table_open_cache = 4000
   table_definition_cache = 2000
   
   # Formule : max_connections × nb_tables_par_query_moyen
   # Exemple : 500 conn × 8 tables = 4000
   ```

2. **Open tables > 95% cache** :
   ```sql
   -- Cache saturé, augmenter
   SET GLOBAL table_open_cache = 6000;
   ```

3. **Trop de fichiers ouverts** :
   ```ini
   # Limite OS doit être > table_open_cache
   open_files_limit = 10000
   
   # Vérifier limite OS
   # ulimit -n
   ```

**📊 Validation**

```sql
-- Surveiller après changement
SHOW GLOBAL STATUS LIKE 'Table_open_cache%';
-- Hit rate doit augmenter progressivement
```

---

### Catégorie 5 : Sécurité et Réseau

#### ✅ 5.1 TLS/SSL Connexions

**🔍 Diagnostic**

```sql
-- TLS disponible ?
SHOW VARIABLES LIKE 'have_ssl';
SHOW VARIABLES LIKE 'ssl%';

-- Connexions actuelles TLS
SELECT 
    user,
    host,
    ssl_type,
    ssl_cipher
FROM information_schema.processlist
WHERE ssl_cipher != '';
```

**⚠️ Seuils critiques**

| Environnement | TLS | Action |
|---------------|-----|--------|
| **Production externe** | Obligatoire | ✅ have_ssl = YES |
| **Production interne** | Recommandé | ✅ Optionnel |
| **Dev local** | Optionnel | ⚪ Non requis |

**🔧 Actions correctives**

1. **🆕 MariaDB 11.8 : TLS par défaut** :
   ```ini
   # TLS activé automatiquement si certificats présents
   ssl_cert = /etc/mysql/ssl/server-cert.pem
   ssl_key = /etc/mysql/ssl/server-key.pem
   ssl_ca = /etc/mysql/ssl/ca-cert.pem
   
   # Forcer TLS pour connexions distantes
   require_secure_transport = ON
   ```

2. **Générer certificats auto-signés (dev/staging)** :
   ```bash
   # Générer CA
   openssl genrsa 2048 > ca-key.pem
   openssl req -new -x509 -nodes -days 3650 \
     -key ca-key.pem -out ca-cert.pem
   
   # Générer certificat serveur
   openssl req -newkey rsa:2048 -days 3650 \
     -nodes -keyout server-key.pem -out server-req.pem
   
   openssl x509 -req -in server-req.pem -days 3650 \
     -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 \
     -out server-cert.pem
   
   # Copier dans /etc/mysql/ssl/
   ```

3. **Vérifier connexion TLS** :
   ```sql
   -- Depuis client
   \s
   -- SSL: Cipher in use is AES256-SHA
   ```

**📊 Validation**

```sql
-- Statistiques TLS
SELECT 
    COUNT(*) AS total_connections,
    SUM(CASE WHEN ssl_cipher != '' THEN 1 ELSE 0 END) AS tls_connections,
    ROUND(100.0 * SUM(CASE WHEN ssl_cipher != '' THEN 1 ELSE 0 END) / COUNT(*), 2) AS tls_pct
FROM information_schema.processlist;
```

---

### Catégorie 6 : Logs et Debugging

#### ✅ 6.1 Slow Query Log

**🔍 Diagnostic**

```sql
SELECT 
    @@slow_query_log AS enabled,
    @@long_query_time AS threshold_sec,
    @@log_queries_not_using_indexes AS log_no_index,
    @@slow_query_log_file AS log_file,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Slow_queries') AS total_slow_queries;

-- Ratio slow queries
SELECT 
    ROUND(100.0 * slow / total, 2) AS slow_query_pct
FROM (
    SELECT 
        CAST((SELECT variable_value FROM information_schema.global_status 
              WHERE variable_name = 'Slow_queries') AS UNSIGNED) AS slow,
        CAST((SELECT variable_value FROM information_schema.global_status 
              WHERE variable_name = 'Queries') AS UNSIGNED) AS total
) AS query_stats;
```

**⚠️ Seuils critiques**

| Environnement | `long_query_time` | `log_no_index` | Ratio slow |
|---------------|-------------------|----------------|------------|
| **OLTP prod** | 0.5 - 1s | ON | < 1% |
| **OLAP prod** | 10 - 30s | OFF | < 10% |
| **Dev** | 0.1s | ON | N/A |

**🔧 Actions correctives**

1. **Slow query log désactivé en prod** :
   ```ini
   # Activer immédiatement
   slow_query_log = ON
   long_query_time = 1  # 1 seconde OLTP
   log_queries_not_using_indexes = ON
   slow_query_log_file = /var/log/mysql/slow-query.log
   log_slow_verbosity = query_plan,explain
   ```

2. **Ratio slow > 5%** (problème performance) :
   ```bash
   # Analyser slow query log
   pt-query-digest /var/log/mysql/slow-query.log
   
   # Identifier top 10 queries lentes
   # → Voir Checklist E.3 Audit Requêtes
   ```

3. **Log trop verbeux** (OLAP) :
   ```ini
   # OLAP : Full scans normaux
   log_queries_not_using_indexes = OFF
   long_query_time = 30  # 30 secondes
   ```

**📊 Validation**

```bash
# Surveiller croissance log
ls -lh /var/log/mysql/slow-query.log

# Analyser périodiquement (hebdo/mensuel)
pt-query-digest slow-query.log > slow-report-$(date +%Y%m%d).txt
```

---

#### ✅ 6.2 Performance Schema

**🔍 Diagnostic**

```sql
SELECT 
    @@performance_schema AS enabled,
    COUNT(*) AS instruments_count
FROM performance_schema.setup_instruments
WHERE enabled = 'YES';

-- Overhead mémoire
SELECT 
    ROUND(SUM(bytes) / 1024 / 1024, 2) AS perf_schema_memory_mb
FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'memory/performance_schema/%';
```

**⚠️ Seuils critiques**

| Environnement | Performance Schema | Overhead |
|---------------|--------------------|----------|
| **Production** | ON (recommandé) | 5-10% RAM |
| **Dev** | ON (debug) | Acceptable |
| **Benchmark** | OFF (précision) | 0% |

**🔧 Actions correctives**

1. **Performance Schema désactivé** :
   ```ini
   # Activer (redémarrage requis)
   performance_schema = ON
   
   # Instruments par défaut suffisants
   # Ajuster si profiling spécifique requis
   ```

2. **Overhead trop élevé** :
   ```sql
   -- Désactiver instruments non utilisés
   UPDATE performance_schema.setup_instruments
   SET ENABLED = 'NO'
   WHERE NAME LIKE 'wait/io/file/%'
     AND NAME NOT LIKE '%innodb%';
   ```

3. **Tables historiques saturées** :
   ```sql
   -- Tronquer périodiquement
   TRUNCATE TABLE performance_schema.events_statements_history_long;
   TRUNCATE TABLE performance_schema.events_waits_history_long;
   ```

**📊 Validation**

```sql
-- Top queries par temps total
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME NOT IN ('performance_schema', 'information_schema')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

## 📊 Tableau récapitulatif des seuils

### Mémoire

| Paramètre | OLTP Optimal | OLAP Optimal | Dev Optimal |
|-----------|--------------|--------------|-------------|
| `innodb_buffer_pool_size` | 60-70% RAM | 70-80% RAM | 12-25% RAM |
| `sort_buffer_size` | 256K-512K | 32M-64M | 256K |
| `join_buffer_size` | 256K | 32M-64M | 128K |
| `tmp_table_size` | 64M | 512M-1G | 32M |
| `max_connections` | 200-500 | 50-100 | 50 |

### I/O et Durabilité

| Paramètre | Production | Dev | Notes |
|-----------|------------|-----|-------|
| `innodb_flush_log_at_trx_commit` | 1 | 0 | 1 = ACID, 0 = Perf |
| `sync_binlog` | 1 | 0 | 1 = Sécurité |
| `innodb_io_capacity` | Selon disque | 200 | Mesurer IOPS réels |
| `innodb_log_file_size` | 1-2G | 256M | OLTP vs Dev |

### Logs et Debug

| Paramètre | Production | Dev |
|-----------|------------|-----|
| `slow_query_log` | ON | ON |
| `long_query_time` | 1s (OLTP)<br>10s (OLAP) | 0.1s |
| `log_queries_not_using_indexes` | ON | ON |
| `performance_schema` | ON | ON |

---

## 🤖 Script d'audit automatisé

### audit-config.sql

```sql
-- ============================================================================
-- SCRIPT D'AUDIT CONFIGURATION MARIADB 11.8
-- ============================================================================
-- Usage : mariadb < audit-config.sql > audit-report.txt
-- ============================================================================

SELECT '========================================' AS '';
SELECT 'AUDIT CONFIGURATION MARIADB' AS '';
SELECT CONCAT('Date: ', NOW()) AS '';
SELECT CONCAT('Version: ', @@version) AS '';
SELECT '========================================' AS '';

-- 1. BUFFER POOL
SELECT '\n1. BUFFER POOL' AS '';
SELECT 
    CONCAT(ROUND(@@innodb_buffer_pool_size / 1024 / 1024 / 1024, 2), ' GB') AS buffer_pool_size,
    @@innodb_buffer_pool_instances AS instances,
    CONCAT(ROUND(100.0 * (1 - (
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_reads') / 
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_read_requests')
    )), 2), '%') AS hit_rate,
    CASE 
        WHEN (100.0 * (1 - (
            (SELECT variable_value FROM information_schema.global_status 
             WHERE variable_name = 'Innodb_buffer_pool_reads') / 
            (SELECT variable_value FROM information_schema.global_status 
             WHERE variable_name = 'Innodb_buffer_pool_read_requests')
        ))) > 99 THEN '🟢 OPTIMAL'
        WHEN (100.0 * (1 - (
            (SELECT variable_value FROM information_schema.global_status 
             WHERE variable_name = 'Innodb_buffer_pool_reads') / 
            (SELECT variable_value FROM information_schema.global_status 
             WHERE variable_name = 'Innodb_buffer_pool_read_requests')
        ))) > 95 THEN '🟡 ACCEPTABLE'
        ELSE '🔴 CRITIQUE'
    END AS status;

-- 2. CONNEXIONS
SELECT '\n2. CONNEXIONS' AS '';
SELECT 
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Threads_connected') AS current,
    @@max_connections AS max,
    CONCAT(ROUND(100.0 * (SELECT variable_value FROM information_schema.global_status 
                          WHERE variable_name = 'Threads_connected') / @@max_connections, 2), '%') AS usage,
    @@thread_handling AS thread_model;

-- 3. I/O
SELECT '\n3. I/O CONFIGURATION' AS '';
SELECT 
    @@innodb_io_capacity AS io_capacity,
    @@innodb_io_capacity_max AS io_capacity_max,
    @@innodb_flush_method AS flush_method,
    @@innodb_flush_neighbors AS flush_neighbors;

-- 4. DURABILITÉ
SELECT '\n4. DURABILITÉ' AS '';
SELECT 
    @@innodb_flush_log_at_trx_commit AS flush_log,
    @@sync_binlog AS sync_binlog,
    CASE @@innodb_flush_log_at_trx_commit
        WHEN 0 THEN '🔴 DANGER: Pas ACID'
        WHEN 1 THEN '🟢 SAFE: Full ACID'
        WHEN 2 THEN '🟡 RISK: OS cache'
    END AS durability_status;

-- 5. LOGS
SELECT '\n5. SLOW QUERY LOG' AS '';
SELECT 
    @@slow_query_log AS enabled,
    @@long_query_time AS threshold_sec,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Slow_queries') AS total_slow,
    CONCAT(ROUND(100.0 * 
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Slow_queries') / 
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Queries'), 2), '%') AS slow_pct;

-- 6. REDO LOG
SELECT '\n6. REDO LOG' AS '';
SELECT 
    CONCAT(ROUND(@@innodb_log_file_size / 1024 / 1024, 0), ' MB') AS log_file_size,
    @@innodb_log_files_in_group AS log_files,
    CONCAT(ROUND(@@innodb_log_file_size * @@innodb_log_files_in_group / 1024 / 1024, 0), ' MB') AS total_redo,
    CONCAT(ROUND(@@innodb_log_buffer_size / 1024 / 1024, 0), ' MB') AS log_buffer;

SELECT '\n========================================' AS '';
SELECT 'FIN AUDIT' AS '';
SELECT '========================================' AS '';
```

### Exécution

```bash
# Audit complet
mariadb -u root -p < audit-config.sql > audit-report-$(date +%Y%m%d).txt

# Ou via script shell
#!/bin/bash
REPORT="audit-$(date +%Y%m%d-%H%M%S).txt"

echo "=== MariaDB Configuration Audit ===" > $REPORT
echo "Date: $(date)" >> $REPORT
echo "" >> $REPORT

mariadb -u root -p -e "SOURCE audit-config.sql" >> $REPORT

echo "Rapport généré : $REPORT"
```

---

## ✅ Points clés à retenir

- 🎯 **Buffer pool #1** : 60-70% RAM OLTP, 70-80% OLAP, hit rate > 99%
- 🔒 **Durabilité prod** : flush_log = 1, sync_binlog = 1 (ACID strict)
- 💾 **I/O adapté disque** : Mesurer IOPS réels, ajuster io_capacity
- 🧵 **Thread pool OLTP** : Scalabilité haute concurrence
- 📊 **Slow query log ON** : Monitoring continu problèmes perf
- 🔍 **Performance Schema** : Overhead 5-10% acceptable pour debug
- 🆕 **MariaDB 11.8** : TLS défaut, flush_neighbors=0 SSD, UCA 14.0.0
- 📝 **Script audit** : Automatiser vérifications périodiques

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Server System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [Performance Schema](https://mariadb.com/kb/en/performance-schema/)

### Outils
- [MySQLTuner](https://github.com/major/MySQLTuner-perl)
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)

### Autres checklists
- [E.2 - Audit d'indexation](./02-audit-indexation.md)
- [E.3 - Audit de requêtes](./03-audit-requetes.md)
- [E.4 - Audit de schéma](./04-audit-schema.md)

### Annexes complémentaires
- [Annexe D - Configurations de référence](/annexes/configuration-reference/README.md)
- [Section 15 - Performance et Tuning](/15-performance-tuning/README.md)

---

## ➡️ Section suivante

**[E.2 - Audit d'indexation](./02-audit-indexation.md)** : Missing indexes, duplicates, stratégie indexation

---

**MariaDB** : Version 11.8 LTS 

⏭️ [Audit d'indexation](/annexes/checklist-performance/02-audit-indexation.md)

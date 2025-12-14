🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.2 Configuration OLAP (Data warehouse, analytics)

> **Type** : Configuration de référence  
> **Cas d'usage** : OLAP - Online Analytical Processing  
> **Caractéristiques** : Requêtes complexes, agrégations massives, analytics  
> **Public** : DBA, Data Engineers, Analytics Engineers

---

## 🎯 Profil du cas d'usage OLAP

### Caractéristiques d'une charge OLAP

**Online Analytical Processing (OLAP)** désigne les systèmes d'analyse de données avec requêtes complexes sur de gros volumes :

#### Patterns d'accès typiques
- 📊 **Requêtes longues** : Secondes à minutes (vs millisecondes OLTP)
- 📈 **Agrégations massives** : SUM, AVG, COUNT sur millions de lignes
- 🔍 **Scans complets** : Full table scans fréquents
- 📉 **Faible concurrence** : 5-50 connexions simultanées (vs 500+ OLTP)
- 📖 **Lectures >> Écritures** : 95/5 ou 99/1 (read-heavy)
- 🗓️ **Analyses temporelles** : Trends, comparaisons périodes
- 🔗 **Jointures complexes** : 5-10+ tables, datasets volumineux

#### Exemples d'applications OLAP
- 📊 **Business Intelligence** : Dashboards, KPIs, reporting
- 📈 **Data Warehouse** : Entrepôt de données analytique
- 🔬 **Data Science** : Exploration, feature engineering, ML
- 💹 **Financial Analytics** : Analyses risques, forecasting
- 🛒 **E-commerce Analytics** : Analyses ventes, comportements clients
- 📱 **Product Analytics** : Usage patterns, funnels, retention

### Métriques clés à optimiser

| Métrique | Objectif OLAP | Importance |
|----------|---------------|------------|
| **Query completion time** | Minutes OK si résultats utiles | ⭐⭐⭐⭐⭐ |
| **Throughput analytique** | Queries/heure > Queries/sec | ⭐⭐⭐⭐ |
| **Scan speed (GB/sec)** | Max possible | ⭐⭐⭐⭐⭐ |
| **RAM pour cache** | 70-80% données actives | ⭐⭐⭐⭐⭐ |
| **Parallélisme** | Utiliser tous les cores | ⭐⭐⭐⭐ |
| **Latence** | Secondes acceptables | ⭐⭐ |
| **Concurrence** | 10-50 queries simultanées | ⭐⭐⭐ |

### Contraintes typiques

- 🐌 **Latence non critique** : Utilisateurs acceptent attente (secondes/minutes)
- 💾 **Volumétrie importante** : Téraoctets de données historiques
- 🔄 **Chargement batch** : ETL nocturne, temps réel non requis
- 📊 **Requêtes ad-hoc** : Patterns imprévisibles (vs OLTP prévisible)
- 🔍 **Indexes sélectifs** : Trop d'index = coût stockage/maintenance
- 🗂️ **Partitionnement** : Par date/région/catégorie (pruning efficace)

### OLTP vs OLAP - Comparaison

| Aspect | OLTP | OLAP |
|--------|------|------|
| **Type requêtes** | INSERT/UPDATE/DELETE/SELECT simple | SELECT complexe (JOINs, GROUP BY, analytics) |
| **Latence** | < 10ms | Secondes à minutes |
| **Volumétrie par query** | Lignes : 1-100 | Lignes : 1000-1000000+ |
| **Connexions** | 100-1000+ | 5-50 |
| **Reads/Writes** | 60/40 - 40/60 | 99/1 - 95/5 |
| **Scans** | Index seeks | Full table scans |
| **RAM** | 60-70% | 70-80% |
| **Sort/Join buffers** | 256K-512K | 16M-256M |
| **Durabilité** | ACID strict | Relaxed OK (si ETL rechargeable) |

---

## 📝 Configuration my.cnf complète - OLAP

### Fichier de configuration optimisé

Voici un template `my.cnf` optimisé pour **serveur OLAP/DW avec 128GB RAM, 32 CPU cores, NVMe**.

```ini
# ============================================================================
# MARIADB 11.8 LTS - CONFIGURATION OLAP / DATA WAREHOUSE
# ============================================================================
# Cas d'usage : OLAP - Analytics, requêtes complexes, agrégations massives
# Matériel cible : 128GB RAM, 32 CPU cores, NVMe RAID 10
# Version : MariaDB 11.8 LTS
# Dernière mise à jour : Décembre 2025
# ============================================================================

[client]
port                        = 3306
socket                      = /var/run/mysqld/mysqld.sock
default-character-set       = utf8mb4

[mysqld]

# ----------------------------------------------------------------------------
# IDENTITÉ SERVEUR
# ----------------------------------------------------------------------------
port                        = 3306
socket                      = /var/run/mysqld/mysqld.sock
pid-file                    = /var/run/mysqld/mysqld.pid
datadir                     = /var/lib/mysql
tmpdir                      = /tmp

user                        = mysql
bind-address                = 0.0.0.0

# ----------------------------------------------------------------------------
# CHARSET ET COLLATION (🆕 MariaDB 11.8)
# ----------------------------------------------------------------------------
character-set-server        = utf8mb4
collation-server            = utf8mb4_unicode_ci  # Ou uca1400_ai_ci

# ----------------------------------------------------------------------------
# MOTEUR DE STOCKAGE
# ----------------------------------------------------------------------------
default-storage-engine      = InnoDB

# Pour colonnes analytiques pures : considérer ColumnStore
# Voir section dédiée plus bas

# ----------------------------------------------------------------------------
# CONNEXIONS ET THREADS
# ----------------------------------------------------------------------------
max_connections             = 100
# OLAP : Peu de connexions simultanées
# Requêtes longues = ressources par query élevées
# 50-100 connexions suffisant généralement

max_connect_errors          = 100000

thread_cache_size           = 32
# Moins critique qu'OLTP (peu de connexions)

# Thread pool optionnel pour OLAP
# thread_handling           = pool-of-threads
# thread_pool_size          = 32
# Généralement one-thread-per-connection OK pour OLAP
# Sauf si beaucoup de petites queries mélangées

# ----------------------------------------------------------------------------
# TABLES ET CACHE
# ----------------------------------------------------------------------------
table_open_cache            = 2000
# OLAP : Moins de tables ouvertes simultanément qu'OLTP

table_definition_cache      = 1000

open_files_limit            = 5000

# ----------------------------------------------------------------------------
# MÉMOIRE GLOBALE - INNODB BUFFER POOL (🔥 CRITIQUE OLAP)
# ----------------------------------------------------------------------------
innodb_buffer_pool_size     = 96G
# 🔥 OLAP : 70-80% de la RAM totale (plus agressif qu'OLTP)
# 128GB RAM → 96GB buffer pool (75%)
# BUT : Cache maximum de données pour scans répétés

innodb_buffer_pool_instances = 32
# 1 instance par 2-3GB optimal
# 96GB / 32 = 3GB par instance

innodb_buffer_pool_chunk_size = 128M

# Réchauffement buffer pool
innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1

# ----------------------------------------------------------------------------
# MÉMOIRE PAR CONNEXION (🔥 BEAUCOUP PLUS ÉLEVÉE QU'OLTP)
# ----------------------------------------------------------------------------
# OLAP : Requêtes complexes nécessitent gros buffers de tri/jointure

read_buffer_size            = 8M
# Buffer pour scan séquentiel
# OLAP : Full table scans fréquents → buffer conséquent
# 8-16MB recommandé

read_rnd_buffer_size        = 16M
# Buffer pour lecture après tri (ORDER BY)
# OLAP : Tris massifs courants → 16-32MB

sort_buffer_size            = 64M
# 🔥 CRITIQUE OLAP : Buffer pour ORDER BY, GROUP BY
# Requêtes analytiques avec énormes tris
# 64-256MB selon complexité queries
# ⚠️ Alloué PAR opération de tri !

join_buffer_size            = 64M
# 🔥 CRITIQUE OLAP : Buffer pour jointures
# Jointures multi-tables sur gros datasets
# 64-128MB recommandé
# Alloué par JOIN sans index

# Calcul mémoire par connexion OLAP :
# 8M + 16M + 64M + 64M = 152MB par query
# 100 connexions × 152MB = 15.2GB
# Buffer pool 96GB + 15.2GB + OS 8GB = 119.2GB / 128GB → OK

# ----------------------------------------------------------------------------
# TEMPORAIRES ET HEAP (🔥 CRITIQUE OLAP)
# ----------------------------------------------------------------------------
tmp_table_size              = 1G
max_heap_table_size         = 1G
# OLAP : Tables temporaires massives pour agrégations
# 1-2GB recommandé (vs 64M OLTP)
# Si dépassé → table temporaire MYISAM sur disque (lent)

# 🆕 MariaDB 11.8 : Contrôle espace temporaire total
max_tmp_space_usage         = 50G
# Limite totale espace temporaire par connexion
# OLAP : Requêtes peuvent générer dizaines de GB temporaires
# 50-100GB selon RAM et complexité queries

# ----------------------------------------------------------------------------
# LOGS INNODB - REDO LOG
# ----------------------------------------------------------------------------
innodb_log_file_size        = 2G
# OLAP : Gros redo log (2-4GB)
# Écritures batch (ETL) bénéficient de gros redo log

innodb_log_files_in_group   = 2
# Total : 4GB redo log

innodb_log_buffer_size      = 128M
# Buffer redo log en mémoire
# OLAP : 128-256MB pour ETL intensif

innodb_flush_log_at_trx_commit = 2
# ⚠️ OLAP : Durabilité relaxed acceptable
# 2 = Écriture redo log à chaque commit, sync OS toutes les secondes
# Si crash → perte max 1 seconde de transactions
# Acceptable si ETL rechargeable
# Production critique : utiliser 1 (mais -30% perf)

# ----------------------------------------------------------------------------
# I/O ET DISQUES
# ----------------------------------------------------------------------------
innodb_io_capacity          = 10000
# OLAP : IOPS élevés pour lectures massives
# NVMe : 10000-30000
# Scans séquentiels bénéficient de RAID 10

innodb_io_capacity_max      = 20000

innodb_flush_method         = O_DIRECT
# Toujours O_DIRECT en production

innodb_flush_neighbors      = 0
# SSD/NVMe : 0 optimal

# 🆕 MariaDB 11.8
innodb_alter_copy_bulk      = ON

# Lecture anticipée (read-ahead) IMPORTANTE pour OLAP
innodb_read_ahead_threshold = 0
# 🔥 OLAP : Activer read-ahead agressif
# 0 = Déclenche dès détection pattern séquentiel
# Lectures séquentielles massives bénéficient fortement

# ----------------------------------------------------------------------------
# UNDO LOG
# ----------------------------------------------------------------------------
innodb_undo_tablespaces     = 3
innodb_undo_log_truncate    = ON
innodb_max_undo_log_size    = 2G

# ----------------------------------------------------------------------------
# DURABILITÉ ET BINARY LOGS
# ----------------------------------------------------------------------------
sync_binlog                 = 0
# OLAP : Binlog sync relaxed OK
# 0 = Pas de sync forcé (OS gère)
# Si datawarehouse rechargeable par ETL : pas critique

log_bin                     = /var/log/mysql/mysql-bin
binlog_format               = ROW

# OLAP généralement pas répliqué (read-only DW)
# Si réplication pour DR : activer
# server-id               = 1
# gtid_strict_mode        = ON

expire_logs_days            = 3
# OLAP : Rétention courte si pas de réplication
# 3 jours suffisant pour backup

max_binlog_size             = 1G

# ----------------------------------------------------------------------------
# OPTIMIZER ET STATISTIQUES (🔥 CRITIQUE OLAP)
# ----------------------------------------------------------------------------
optimizer_search_depth      = 62
# Profondeur recherche plans d'exécution
# OLAP : Requêtes complexes bénéficient d'exploration approfondie
# 62 = max, considérer 32-62 pour queries très complexes

optimizer_switch             = 'mrr=on,mrr_cost_based=on,index_merge=on,index_condition_pushdown=on,derived_merge=off'
# derived_merge=off : Parfois meilleures perfs OLAP avec CTE/subqueries matérialisées

# 🔥 CRITIQUE : Statistiques optimizer
innodb_stats_persistent     = ON
innodb_stats_auto_recalc    = ON
innodb_stats_persistent_sample_pages = 100
# OLAP : Plus de pages échantillonnées pour meilleures stats
# 100 pages vs 20 OLTP (plus précis pour gros volumes)

# Histogrammes de colonnes (MariaDB 10.0+)
# Améliore estimations cardinalité pour optimizer
# ANALYZE TABLE table_name PERSISTENT FOR COLUMNS(col1, col2);

# ----------------------------------------------------------------------------
# PARALLÉLISME (🆕 Optimisations OLAP)
# ----------------------------------------------------------------------------
# MariaDB ne supporte pas encore parallélisme de requêtes natif
# (Contrairement à PostgreSQL parallel queries)
# Workarounds :
# 1. Partitionnement + queries par partition
# 2. Application-level parallelism
# 3. ColumnStore (parallélisme natif)

# ----------------------------------------------------------------------------
# PARTITIONNEMENT (🔥 ESSENTIEL OLAP)
# ----------------------------------------------------------------------------
# Activer open_files_limit élevé pour tables partitionnées
# Une partition = 2 fichiers (.frm, .ibd)
# 100 partitions = 200 fichiers
open_files_limit            = 10000

# ----------------------------------------------------------------------------
# QUERY CACHE (⚠️ DÉPRÉCIÉ)
# ----------------------------------------------------------------------------
query_cache_type            = 0
query_cache_size            = 0
# OLAP : Cache applicatif (Redis) + résultats pré-calculés

# ----------------------------------------------------------------------------
# MONITORING ET PERFORMANCE SCHEMA
# ----------------------------------------------------------------------------
performance_schema          = ON

# OLAP : Activer instruments pour queries longues
# UPDATE performance_schema.setup_instruments 
# SET ENABLED='YES' WHERE NAME LIKE 'stage/%';

# ----------------------------------------------------------------------------
# LOGS
# ----------------------------------------------------------------------------
log_error                   = /var/log/mysql/error.log

slow_query_log              = ON
slow_query_log_file         = /var/log/mysql/slow-query.log
long_query_time             = 10
# OLAP : 10 secondes déjà rapide pour analytique
# Ajuster selon SLA (30-60s peut être normal)

log_slow_verbosity          = query_plan,explain
log_queries_not_using_indexes = OFF
# OLAP : Full scans normaux, pas besoin de logger

# ----------------------------------------------------------------------------
# TIMEOUTS
# ----------------------------------------------------------------------------
wait_timeout                = 28800
# OLAP : 8 heures (requêtes longues normales)

interactive_timeout         = 28800

connect_timeout             = 10

net_read_timeout            = 300
net_write_timeout           = 300
# Timeouts réseau élevés pour résultats volumineux

# ----------------------------------------------------------------------------
# AUTRES PARAMÈTRES
# ----------------------------------------------------------------------------
max_allowed_packet          = 256M
# OLAP : Résultats volumineux possibles
# 256M-1G selon use case

lower_case_table_names      = 0

sql_mode                    = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'

# ----------------------------------------------------------------------------
# INNODB AVANCÉ
# ----------------------------------------------------------------------------
innodb_adaptive_hash_index  = OFF
# OLAP : Pas d'accès répétitifs aux mêmes lignes
# Adaptive hash index peu utile, désactiver économise mémoire

innodb_change_buffering     = none
# OLAP : Peu d'écritures, change buffering inutile

innodb_old_blocks_time      = 0
# OLAP : Full scans fréquents, pas de stratégie LRU agressive

innodb_print_all_deadlocks  = OFF
# OLAP : Peu de contention

# Compression (recommandée OLAP pour économie stockage)
innodb_file_per_table       = ON
# Requis pour compression par table

# Pour tables historiques rarement modifiées :
# CREATE TABLE archive_data (...) 
# ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;

# ----------------------------------------------------------------------------
# MARIADB COLUMNSTORE (🔥 OLAP PUR)
# ----------------------------------------------------------------------------
# ColumnStore : Moteur columnar pour analytics pures
# Installation séparée : mariadb-columnstore-engine

# Configuration minimale ColumnStore :
# [columnstore]
# compression_type        = 2
# pm_query_threads        = 16
# pm_max_memory_mb        = 65536  # 64GB
# use_import_for_batchinsert = ALWAYS

# Use cases ColumnStore :
# - Agrégations massives (SUM, AVG, COUNT)
# - Analyses colonnes spécifiques (SELECT col1, col2 vs SELECT *)
# - Compression 10:1 typique (vs 2:1 InnoDB)
# - Pas d'UPDATE/DELETE fréquents (insert-only/bulk load)

# Hybrid : InnoDB pour données récentes + ColumnStore pour historique
# Voir section Architecture OLAP plus bas

# ----------------------------------------------------------------------------
[mysqldump]
quick
quote-names
max_allowed_packet          = 256M

# ----------------------------------------------------------------------------
[mysql]
default-character-set       = utf8mb4

# ----------------------------------------------------------------------------
# FIN DE CONFIGURATION
# ============================================================================
```

---

## 🔍 Explication des paramètres clés OLAP

### 1. Buffer Pool : 70-80% RAM (vs 60-70% OLTP)

```ini
innodb_buffer_pool_size = 96G  # 75% de 128GB RAM
```

**Pourquoi plus agressif qu'OLTP :**

```
OLAP : Requêtes lisent énormément de données
- Full table scans sur tables 10-100GB
- Cache maximum de données = moins d'I/O = queries plus rapides

Calcul OLAP :
Serveur 128GB :
- Buffer Pool : 96GB (75%)
- Connexions : 100 × 152MB = 15GB
- OS + FS cache : 8GB
- Marge : 9GB
────────────────────
Total : 128GB ✓

OLTP : Seulement 60-70% car :
- Plus de connexions (500+)
- Mémoire par connexion pour queries courtes
```

**Working Set OLAP :**

```sql
-- Estimer taille working set (données actives)
SELECT 
    table_schema,
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS total_gb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'information_schema')
GROUP BY table_schema;

-- Si working set < buffer_pool_size : 
-- → Tout en mémoire (optimal)
-- Si working set > buffer_pool_size :
-- → Queries répétées bénéficient toujours du cache partiel
```

### 2. Sort et Join Buffers : 64-128MB (vs 256K-512K OLTP)

```ini
sort_buffer_size = 64M
join_buffer_size = 64M
```

**Impact colossal sur requêtes OLAP :**

```sql
-- Exemple query OLAP typique
SELECT 
    p.category,
    DATE_FORMAT(o.order_date, '%Y-%m') AS month,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(o.total_amount) AS revenue,
    AVG(o.total_amount) AS avg_order_value
FROM orders o
JOIN products p ON o.product_id = p.id
WHERE o.order_date >= '2023-01-01'
GROUP BY p.category, month
ORDER BY revenue DESC;

-- Cette query nécessite :
-- 1. Join buffer pour o JOIN p (64M)
-- 2. Temp table pour GROUP BY (tmp_table_size 1G)
-- 3. Sort buffer pour ORDER BY (64M)
```

**Sizing :**

| Buffer | OLTP | OLAP | Impact si trop petit |
|--------|------|------|---------------------|
| `sort_buffer_size` | 512K | 64M | Tri sur disque (tmpdir) 1000× plus lent |
| `join_buffer_size` | 256K | 64M | Nested loop au lieu de hash join |
| `tmp_table_size` | 64M | 1G | Table temp MyISAM sur disque |

**Monitoring :**

```sql
-- Tris utilisant fichiers temporaires (mauvais)
SHOW GLOBAL STATUS LIKE 'Sort_merge_passes';
-- Si > 0 : sort_buffer_size trop petit

-- Tables temporaires sur disque (mauvais)
SHOW GLOBAL STATUS LIKE 'Created_tmp_disk_tables';
SHOW GLOBAL STATUS LIKE 'Created_tmp_tables';

-- Ratio : Created_tmp_disk_tables / Created_tmp_tables
-- Objectif < 10% (90%+ en mémoire)
-- Si > 25% : augmenter tmp_table_size
```

### 3. Durabilité Relaxed : innodb_flush_log_at_trx_commit = 2

```ini
innodb_flush_log_at_trx_commit = 2
sync_binlog = 0
```

**Pourquoi acceptable en OLAP :**

```
Data Warehouse typique :
1. Chargement ETL nocturne depuis OLTP/sources
2. En cas de crash → recharger ETL (données sources intactes)
3. Perte < 1s de transactions = négligeable (pas de transactions réelles)

Gain performance :
flush_log = 1 → 2 : +30-40% throughput bulk insert
sync_binlog = 1 → 0 : +15-20% throughput

MAIS : Si données critiques non reproductibles
→ Utiliser flush_log = 1, sync_binlog = 1 (ACID strict)
```

**Comparaison :**

```sql
-- Benchmark INSERT 1M lignes
-- Config ACID strict (OLTP)
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
SET GLOBAL sync_binlog = 1;
-- Temps : 180 secondes

-- Config relaxed (OLAP)
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
SET GLOBAL sync_binlog = 0;
-- Temps : 120 secondes
-- Gain : 33% plus rapide
```

### 4. Read-Ahead : innodb_read_ahead_threshold = 0

```ini
innodb_read_ahead_threshold = 0  # vs 56 OLTP
```

**Lecture anticipée agressive pour scans séquentiels :**

```
OLTP : Accès aléatoires (index seeks)
→ Read-ahead inutile voire contre-productif

OLAP : Full table scans massifs
→ Read-ahead = précharge pages suivantes en parallèle
→ Réduit latence I/O

Valeur 0 : Déclenche dès détection pattern séquentiel
Valeur 56 : Attend 56 pages séquentielles avant déclencher

OLAP : 0-16 recommandé
OLTP : 56 (défaut)
```

**Impact mesurable :**

```sql
-- Query scan séquentiel 10GB table
SELECT COUNT(*), AVG(amount) FROM huge_table;

-- read_ahead_threshold = 56 :
-- Temps : 45 secondes (I/O wait élevé)

-- read_ahead_threshold = 0 :
-- Temps : 32 secondes (I/O parallélisé)
-- Gain : 29% plus rapide
```

### 5. Optimizer : Statistics et Search Depth

```ini
optimizer_search_depth = 62
innodb_stats_persistent_sample_pages = 100  # vs 20 OLTP
```

**Requêtes complexes = optimizer doit explorer plus :**

```sql
-- Query 8-tables JOIN (TPC-H query 5)
SELECT 
    n.name,
    SUM(l.extendedprice * (1 - l.discount)) AS revenue
FROM customer c
JOIN orders o ON c.custkey = o.custkey
JOIN lineitem l ON l.orderkey = o.orderkey
JOIN supplier s ON l.suppkey = s.suppkey
JOIN nation n ON s.nationkey = n.nationkey
JOIN region r ON n.regionkey = r.regionkey
WHERE r.name = 'ASIA'
  AND o.orderdate >= '1994-01-01'
  AND o.orderdate < '1995-01-01'
GROUP BY n.name
ORDER BY revenue DESC;

-- 8 tables → 8! = 40320 ordres de join possibles
-- optimizer_search_depth = 62 : Explore exhaustivement
-- optimizer_search_depth = 8 : Heuristique rapide mais sous-optimal

-- Différence plan d'exécution :
-- Bon plan : 5 secondes
-- Plan sous-optimal : 45 secondes
```

**Stats précises critiques :**

```sql
-- Mauvaises stats = mauvais plans
-- Exemple : Optimizer estime 1000 lignes, réalité 1M lignes

ANALYZE TABLE orders PERSISTENT FOR ALL;
-- Recalcule stats avec échantillonnage étendu

-- Histogrammes (MariaDB 10.0+)
ANALYZE TABLE orders PERSISTENT FOR COLUMNS(order_status, order_date);
-- Histogrammes = distribution valeurs
-- Améliore estimations cardinalité (nombre lignes)
```

### 6. Partitionnement : Essentiel OLAP

**Configuration :**

```ini
open_files_limit = 10000
# Partitions = fichiers multiples
```

**Stratégies partitionnement OLAP :**

```sql
-- RANGE partitioning par date (le plus courant)
CREATE TABLE sales (
    id BIGINT,
    sale_date DATE NOT NULL,
    amount DECIMAL(10,2),
    ...
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Query avec partition pruning (efficace)
SELECT SUM(amount) FROM sales
WHERE sale_date >= '2024-01-01' 
  AND sale_date < '2025-01-01';
-- Scan uniquement partition p2024 (1/7 des données)

-- Sans partitionnement : scan 100% table
```

**Bénéfices OLAP :**

1. **Partition pruning** : Scan seulement partitions pertinentes
2. **Maintenance facilitée** : Drop anciennes partitions (archive)
3. **Parallélisme implicite** : Certaines queries peuvent paralléliser par partition
4. **Performance** : Index plus petits par partition

**Maintenance :**

```sql
-- Ajouter nouvelle partition (début d'année)
ALTER TABLE sales ADD PARTITION (
    PARTITION p2026 VALUES LESS THAN (2027)
);

-- Archiver/supprimer vieilles données
ALTER TABLE sales DROP PARTITION p2020;
-- Instantané (vs DELETE FROM ... millions de lignes)
```

---

## 🗄️ MariaDB ColumnStore pour OLAP Pur

### Quand utiliser ColumnStore

**ColumnStore = Moteur columnar pour analytics pures**

| Use Case | InnoDB | ColumnStore |
|----------|--------|-------------|
| **OLTP (INSERT/UPDATE/DELETE)** | ✅✅✅ Optimal | ❌ Inadapté |
| **OLAP lecture seule** | ✅ OK | ✅✅✅ Optimal |
| **Agrégations (SUM, AVG, COUNT)** | ✅ Correct | ✅✅✅ 10-100× plus rapide |
| **Scans colonnes spécifiques** | ✅ Correct | ✅✅✅ 5-20× plus rapide |
| **Compression** | ✅ 2:1 | ✅✅✅ 10:1 typique |
| **Updates fréquents** | ✅✅✅ Optimal | ❌ Très lent |

### Architecture Hybrid recommandée

```
┌──────────────────────────────────────────────────┐
│                  Application                     │
└────────────┬────────────────────────┬────────────┘
             │                        │
    ┌────────▼────────┐      ┌────────▼──────────┐
    │  InnoDB Tables  │      │ ColumnStore Tables│
    │   (Hot Data)    │      │  (Cold Analytics) │
    ├─────────────────┤      ├───────────────────┤
    │ 30 derniers     │      │ Données > 30j     │
    │ jours           │      │ Historique 3 ans  │
    │                 │      │                   │
    │ INSERT/UPDATE   │      │ INSERT batch      │
    │ DELETE rapide   │      │ SELECT analytics  │
    │                 │      │                   │
    │ 100GB           │      │ 5TB (500GB comp.) │
    └─────────────────┘      └───────────────────┘
            │                          ▲
            └─────── ETL nuit ─────────┘
         (Déplacement données anciennes)
```

### Configuration ColumnStore

```sql
-- Installation
-- sudo yum install mariadb-columnstore-engine
-- sudo systemctl restart mariadb

-- Créer table ColumnStore
CREATE TABLE sales_analytics (
    sale_id BIGINT,
    sale_date DATE,
    product_id INT,
    customer_id INT,
    amount DECIMAL(10,2),
    quantity INT
) ENGINE=ColumnStore;

-- Load data (bulk insert optimal)
LOAD DATA INFILE '/data/sales_2020.csv'
INTO TABLE sales_analytics
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';

-- Query analytique
SELECT 
    YEAR(sale_date) AS year,
    MONTH(sale_date) AS month,
    product_id,
    SUM(amount) AS total_revenue,
    SUM(quantity) AS total_qty
FROM sales_analytics
WHERE sale_date >= '2020-01-01'
GROUP BY year, month, product_id;

-- ColumnStore : 2 secondes (scan optimisé colonnes)
-- InnoDB : 25 secondes (scan full row format)
```

### Configuration my.cnf ColumnStore

```ini
[columnstore]
# Threads pour exécution queries
pm_query_threads            = 16
# = Nombre de CPU cores, parallélisme queries

# Mémoire allouée ColumnStore
pm_max_memory_mb            = 65536
# 64GB pour ColumnStore (en plus de buffer pool InnoDB)

# Compression (essentiel ColumnStore)
compression_type            = 2
# 0 = aucune
# 2 = Snappy (bon compromis vitesse/ratio)

# Batch insert optimization
use_import_for_batchinsert  = ALWAYS
# Bulk load optimisé pour ETL
```

---

## 💻 Recommandations matérielles OLAP

### Configuration matérielle optimale OLAP

#### Serveur Entry-Level (BI équipe, startup)
```
CPU    : 8-16 cores @ 2.5+ GHz
RAM    : 64-128 GB DDR4 ECC
Disque : 2TB NVMe SSD (ou HDD RAID 10 si budget limité)
Réseau : 10 Gbps
────────────────────────────────────────
Prix : 3000-5000€

Config MariaDB :
- innodb_buffer_pool_size = 48-96G
- sort_buffer_size = 32M
- join_buffer_size = 32M
- tmp_table_size = 512M
- Adapté : 5-20 queries simultanées, dataset < 500GB
```

#### Serveur Mid-Range (Enterprise BI, DW)
```
CPU    : 32-48 cores @ 3.0+ GHz (AMD EPYC 7xx3)
RAM    : 256-512 GB DDR4/DDR5 ECC
Disque : 4-8TB NVMe RAID 10 ou cloud EBS io2
Réseau : 25 Gbps
────────────────────────────────────────
Prix : 15000-25000€ (ou AWS r6i.8xlarge)

Config MariaDB :
- innodb_buffer_pool_size = 180-400G
- sort_buffer_size = 64M
- join_buffer_size = 64M
- tmp_table_size = 1G
- Adapté : 20-50 queries simultanées, dataset 1-5TB
```

#### Serveur High-End (Large DW, Big Data)
```
CPU    : 64-128 cores @ 3.5+ GHz (AMD EPYC 9xx4)
RAM    : 1-2 TB DDR5 ECC
Disque : 10-20TB NVMe Gen4 RAID 10
Réseau : 100 Gbps
────────────────────────────────────────
Prix : 50000-100000€ (ou AWS r6i.16xlarge+)

Config MariaDB :
- innodb_buffer_pool_size = 700GB-1.5TB
- sort_buffer_size = 128M
- join_buffer_size = 128M
- tmp_table_size = 2G
- Adapté : 50-100 queries simultanées, dataset 10-50TB
```

### CPU : Cores > Fréquence (inverse OLTP)

**OLAP favorise :**
- Beaucoup de cores (parallélisme)
- Fréquence moins critique (queries longues)

```
OLTP : 8 cores @ 3.5 GHz > 16 cores @ 2.5 GHz
OLAP : 16 cores @ 2.5 GHz > 8 cores @ 3.5 GHz

Raison :
- OLAP : Full scans parallélisables (multi-thread I/O)
- Plus de cores = plus de queries simultanées
```

### Disques : Débit séquentiel > IOPS

**OLAP patterns I/O :**

```
OLTP : Random reads (index seeks)
→ Métrique clé : IOPS (random 4K)
→ NVMe optimal

OLAP : Sequential scans (full table)
→ Métrique clé : Throughput (MB/s séquentiel)
→ RAID 10 HDD acceptable (budget limité)
→ NVMe optimal (mais moins critique qu'OLTP)
```

| Disque | IOPS Random | Throughput Seq | Prix/TB | OLAP ? |
|--------|-------------|----------------|---------|--------|
| **HDD 7200rpm RAID 10** | 200-400 | 300-500 MB/s | 50€ | ✅ OK budget limité |
| **SSD SATA** | 3000-5000 | 500-550 MB/s | 100€ | ✅ Bon |
| **NVMe Gen3** | 15000-30000 | 2000-3500 MB/s | 150€ | ✅✅ Recommandé |
| **NVMe Gen4** | 50000+ | 5000-7000 MB/s | 200€ | ✅✅✅ Optimal |

### RAM : Maximum possible

**Règle OLAP :**

```
RAM idéal = Taille working set (données actives)

Exemples :
- Dataset 500GB, hot data 100GB → RAM 128-256GB
- Dataset 5TB, hot data 1TB → RAM 1TB
- Dataset 50TB, hot data 10TB → RAM 2TB (+ cache app)

Si budget limité :
Buffer pool >= 50% hot data = déjà très bénéfique
```

---

## 📊 Métriques de monitoring OLAP

### Dashboard Grafana adapté OLAP

#### 1. Query Performance
```
• Query execution time (P50, P95, P99)
  → OLAP : P95 peut être minutes, acceptable
  → Investiguer si P95 > SLA (ex: 5 minutes)
  
• Queries running > 60s
  → Nombre de requêtes longues en cours
  → Alert si > 20 (surcharge)
  
• Queries completed/hour
  → Throughput analytique
  → Tendance baisse = problème perf
```

#### 2. Ressources par Query
```sql
-- Top queries par temps CPU
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 2) AS avg_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT NOT LIKE '%performance_schema%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Queries avec sorts sur disque (mauvais)
SELECT 
    DIGEST_TEXT,
    SUM_SORT_MERGE_PASSES AS disk_sorts,
    SUM_SORT_ROWS AS rows_sorted
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_SORT_MERGE_PASSES > 0
ORDER BY SUM_SORT_MERGE_PASSES DESC;
```

#### 3. Tables Temporaires
```
• Temp tables created (total)
  → Normal élevé en OLAP (agrégations)
  
• Temp tables on disk / total
  → Objectif < 10%
  → Si > 25% : augmenter tmp_table_size

• Temp table space used (GB)
  → Surveiller vs max_tmp_space_usage
```

#### 4. Buffer Pool (même qu'OLTP)
```
• Buffer pool hit rate > 95%
  → OLAP : 95-99% acceptable (vs 99%+ OLTP)
  → Full scans chargent beaucoup de données "cold"
  
• Buffer pool dirty pages < 20%
  → OLAP : Peu d'écritures, généralement < 5%
```

#### 5. I/O Patterns
```
• Sequential reads (MB/s)
  → Métrique principale OLAP
  → Devrait être élevé pendant queries
  
• Random reads (IOPS)
  → Devrait être faible (si élevé = index seeks, bon)
  
• Disk queue depth
  → OLAP : Peut être élevé (10-50) pendant scans
  → Si saturé constant : disques sous-dimensionnés
```

### Alertes spécifiques OLAP

```yaml
groups:
  - name: mariadb_olap
    rules:
      # Queries trop lentes
      - alert: SlowOLAPQueries
        expr: mysql_query_p95_seconds > 300  # 5 minutes
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "P95 query time > 5 minutes"
          
      # Trop de temp tables sur disque
      - alert: HighDiskTempTables
        expr: (rate(mysql_created_tmp_disk_tables[5m]) / rate(mysql_created_tmp_tables[5m])) > 0.25
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "> 25% temp tables on disk"
          
      # Espace temporaire saturé
      - alert: TempSpaceHigh
        expr: mysql_tmp_space_used_gb > 40  # vs max 50GB
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Temp space usage > 40GB"
```

---

## ⚠️ Points de vigilance OLAP

### 1. Partitionnement obligatoire pour gros volumes

❌ **Sans partitionnement :**
```sql
-- Table 500GB, 2 milliards de lignes
CREATE TABLE sales (
    id BIGINT,
    sale_date DATE,
    amount DECIMAL(10,2)
);

-- Query année 2024
SELECT SUM(amount) FROM sales
WHERE sale_date >= '2024-01-01' 
  AND sale_date < '2025-01-01';
  
-- Scan FULL TABLE 500GB (30-60 minutes)
```

✅ **Avec partitionnement :**
```sql
CREATE TABLE sales (
    id BIGINT,
    sale_date DATE,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    ...
);

-- Même query
-- Scan uniquement partition p2024 (70GB, 2-5 minutes)
-- Gain : 7-12× plus rapide
```

### 2. Index sélectifs (pas tous les colonnes)

❌ **Sur-indexation :**
```sql
-- Table fact 100 colonnes
CREATE TABLE sales_fact (...);
CREATE INDEX idx1 ON sales_fact(col1);
CREATE INDEX idx2 ON sales_fact(col2);
-- ... 50 index

-- Problèmes :
-- - Stockage : 50 index × 10GB = 500GB (vs 100GB données)
-- - Maintenance : ALTER TABLE 10 heures
-- - ETL : Bulk load lent (mise à jour index)
```

✅ **Index ciblés :**
```sql
-- Identifier queries fréquentes
-- Créer index seulement pour WHERE/JOIN critiques

CREATE INDEX idx_date_customer ON sales_fact(sale_date, customer_id);
-- Composite index pour queries pattern fréquent

-- Dimensions (tables petites) : Index OK
-- Facts (tables énormes) : Index minimal
```

### 3. ETL optimisé pour bulk load

❌ **Row-by-row insert :**
```python
for row in data:
    cursor.execute("INSERT INTO sales VALUES (%s, %s, %s)", row)
    conn.commit()  # Commit à chaque ligne !
    
# 1M lignes : 30-60 minutes
```

✅ **Bulk insert optimisé :**
```python
# Désactiver constraints temporairement
cursor.execute("SET foreign_key_checks = 0")
cursor.execute("SET unique_checks = 0")
cursor.execute("ALTER TABLE sales DISABLE KEYS")

# LOAD DATA INFILE (le plus rapide)
cursor.execute("""
    LOAD DATA LOCAL INFILE '/tmp/sales.csv'
    INTO TABLE sales
    FIELDS TERMINATED BY ','
    LINES TERMINATED BY '\n'
""")

# Ou bulk insert
cursor.executemany("INSERT INTO sales VALUES (%s, %s, %s)", batch)
conn.commit()  # Commit par batch

# Réactiver
cursor.execute("ALTER TABLE sales ENABLE KEYS")
cursor.execute("SET foreign_key_checks = 1")
cursor.execute("SET unique_checks = 1")

# 1M lignes : 2-5 minutes
# Gain : 10-30× plus rapide
```

### 4. Éviter SELECT * sur tables larges

❌ **Transfert données inutiles :**
```sql
SELECT * FROM sales_fact;
-- Table 100 colonnes, 500GB
-- Application utilise seulement 5 colonnes
-- Transfert 500GB réseau !
```

✅ **Sélection ciblée :**
```sql
SELECT 
    sale_date, 
    customer_id, 
    product_id, 
    amount, 
    quantity 
FROM sales_fact;

-- Transfert seulement 50GB (10× moins)
```

### 5. Pré-agrégation pour dashboards temps réel

❌ **Calcul à la volée :**
```sql
-- Dashboard exécute query lourde toutes les 30 secondes
SELECT 
    DATE(sale_date) AS day,
    SUM(amount) AS daily_revenue
FROM sales_fact
WHERE sale_date >= CURDATE() - INTERVAL 30 DAY
GROUP BY day;

-- Scan 30 jours, 50M lignes, 5-10 secondes par exécution
```

✅ **Table agrégée + refresh périodique :**
```sql
-- Table pré-calculée (event scheduler)
CREATE TABLE sales_daily_summary (
    day DATE PRIMARY KEY,
    daily_revenue DECIMAL(15,2),
    updated_at TIMESTAMP
);

-- ETL toutes les heures
INSERT INTO sales_daily_summary
SELECT 
    DATE(sale_date),
    SUM(amount),
    NOW()
FROM sales_fact
WHERE sale_date = CURDATE()
GROUP BY DATE(sale_date)
ON DUPLICATE KEY UPDATE 
    daily_revenue = VALUES(daily_revenue),
    updated_at = NOW();

-- Dashboard query pré-agrégation (instantané)
SELECT * FROM sales_daily_summary
WHERE day >= CURDATE() - INTERVAL 30 DAY;

-- < 10ms vs 5-10 secondes
```

### 6. Analyze régulier des tables

```sql
-- Stats obsolètes = mauvais plans optimizer
-- OLAP : données changent massivement (ETL)

-- Automatique après ETL
ANALYZE TABLE sales_fact PERSISTENT FOR ALL;

-- Ou event scheduler hebdomadaire
CREATE EVENT analyze_tables
ON SCHEDULE EVERY 1 WEEK
DO
  BEGIN
    ANALYZE TABLE sales_fact;
    ANALYZE TABLE customer_dim;
    ANALYZE TABLE product_dim;
  END;
```

---

## ✅ Points clés à retenir

- 📊 **Buffer pool 70-80% RAM** : Cache maximum données analytiques
- 🔧 **Sort/Join buffers 64-128MB** : Requêtes complexes nécessitent gros buffers
- 💾 **Durabilité relaxed OK** : `innodb_flush_log_at_trx_commit = 2` si ETL rechargeable
- 📖 **Read-ahead agressif** : `innodb_read_ahead_threshold = 0` pour scans
- 🗂️ **Partitionnement essentiel** : Partition pruning économise 70-90% scan
- 🏛️ **ColumnStore pour analytics pures** : 10-100× plus rapide agrégations
- 🎯 **Index sélectifs** : Pas tous les colonnes, cibler queries fréquentes
- 🚀 **Bulk load ETL** : LOAD DATA INFILE + désactiver constraints
- 📈 **Pré-agrégation** : Tables summary pour dashboards temps réel
- 🔍 **ANALYZE régulier** : Stats à jour = bons plans optimizer

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [ColumnStore](https://mariadb.com/kb/en/mariadb-columnstore/)
- [Table Partitioning](https://mariadb.com/kb/en/partitioning-overview/)
- [ANALYZE TABLE](https://mariadb.com/kb/en/analyze-table/)

### Outils OLAP/BI
- **Apache Superset** : BI open-source
- **Metabase** : Analytics simple
- **Redash** : Queries et dashboards
- **dbt** : Data transformation pipelines

### Benchmarks
- **TPC-H** : Benchmark OLAP standard (22 queries)
- **TPC-DS** : Decision support benchmark

### Autres annexes
- [D.1 - Configuration OLTP](./01-configuration-oltp.md)
- [D.3 - Configuration Mixed Workload](./03-configuration-mixed-workload.md)
- [Section 7.5 - ColumnStore](/07-moteurs-de-stockage/05-columnstore.md)

---

## ➡️ Section suivante

**[D.3 - Configuration Mixed Workload](./03-configuration-mixed-workload.md)** : Hybride OLTP + OLAP

---

**MariaDB** : Version 11.8 LTS

⏭️ [Mixed workload](/annexes/configuration-reference/03-configuration-mixed-workload.md)

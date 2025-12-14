🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.3 Configuration Mixed Workload (Hybride OLTP + OLAP)

> **Type** : Configuration de référence  
> **Cas d'usage** : Mixed Workload - Hybride transactionnel + analytique  
> **Caractéristiques** : Transactions ET analytics sur même infrastructure  
> **Public** : DBA, Architectes, DevOps

---

## 🎯 Profil du cas d'usage Mixed Workload

### Caractéristiques d'une charge mixte

**Mixed Workload** désigne les systèmes combinant transactions OLTP et requêtes analytiques OLAP simultanément :

#### Patterns d'accès typiques
- 🔄 **Transactions courtes** : INSERT/UPDATE/DELETE temps réel
- 📊 **Analytics périodiques** : Dashboards, rapports, agrégations
- ⚖️ **Équilibrage dynamique** : Ratio OLTP/OLAP variable dans le temps
- 🕐 **Pics mixtes** : Transactions pointe journée + rapports matinaux
- 📈 **Croissance continue** : Données historiques + activité courante

#### Exemples d'applications Mixed
- 🛒 **E-commerce moderne** : 
  - OLTP : Commandes, paiements, inventory en temps réel
  - OLAP : Dashboards ventes, analytics comportement clients
- 💼 **ERP/CRM** :
  - OLTP : Saisie données métier, workflows
  - OLAP : Reporting managérial, forecasting
- 📱 **SaaS Applications** :
  - OLTP : Fonctionnalités applicatives utilisateur
  - OLAP : Analytics usage, billing, metrics produit
- 🏦 **Banking** :
  - OLTP : Transactions bancaires instantanées
  - OLAP : Analyses risques, détection fraudes, compliance

### Défis du Mixed Workload

| Défi | Impact | Solution |
|------|--------|----------|
| **Contention ressources** | Analytics ralentit transactions | Séparation read replicas |
| **Configuration compromis** | Ni optimal OLTP ni OLAP | Tuning équilibré |
| **Buffer pool pollution** | Scans OLAP évincent données OLTP | InnoDB old blocks time |
| **Lock contention** | Requêtes longues bloquent writes | READ COMMITTED isolation |
| **I/O spikes** | Analytics sature disques | QoS I/O ou séparation physique |

### Métriques clés Mixed Workload

| Métrique | Objectif | Importance |
|----------|----------|------------|
| **Latence P95 OLTP** | < 50ms | ⭐⭐⭐⭐⭐ |
| **Completion time OLAP** | < 5 min | ⭐⭐⭐⭐ |
| **Buffer pool hit rate** | > 97% | ⭐⭐⭐⭐⭐ |
| **Lock waits/sec** | < 10 | ⭐⭐⭐⭐ |
| **Connexions OLTP vs OLAP** | 70/30 à 50/50 | ⭐⭐⭐⭐ |

---

## 📝 Configuration my.cnf complète - Mixed Workload

### Approche 1 : Configuration équilibrée (serveur unique)

Configuration pour **serveur 64GB RAM, 16 CPU cores, NVMe** gérant OLTP + OLAP.

```ini
# ============================================================================
# MARIADB 11.8 LTS - CONFIGURATION MIXED WORKLOAD
# ============================================================================
# Cas d'usage : Hybride OLTP + OLAP sur infrastructure partagée
# Matériel cible : 64GB RAM, 16 CPU cores, NVMe
# Ratio estimé : 60% OLTP / 40% OLAP
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
collation-server            = utf8mb4_unicode_ci

# ----------------------------------------------------------------------------
# MOTEUR DE STOCKAGE
# ----------------------------------------------------------------------------
default-storage-engine      = InnoDB

# ----------------------------------------------------------------------------
# CONNEXIONS ET THREADS (COMPROMIS OLTP/OLAP)
# ----------------------------------------------------------------------------
max_connections             = 300
# Mixed : 200-300 connexions
# OLTP : 200 connexions rapides
# OLAP : 50-100 connexions longues
# Moins qu'OLTP pur (500) mais plus qu'OLAP pur (100)

max_connect_errors          = 100000

thread_cache_size           = 64
# Compromis entre OLTP (128) et OLAP (32)

thread_handling             = pool-of-threads
# 🔥 Thread pool bénéfique même en Mixed
# Gère mieux mix de queries courtes et longues

thread_pool_size            = 16
# = Nombre CPU cores

thread_pool_max_threads     = 500

thread_pool_stall_limit     = 500
# Équilibre OLTP (latence) et OLAP (throughput)

# ----------------------------------------------------------------------------
# TABLES ET CACHE
# ----------------------------------------------------------------------------
table_open_cache            = 3000
# Compromis OLTP (4000) et OLAP (2000)

table_definition_cache      = 1500

open_files_limit            = 8000

# ----------------------------------------------------------------------------
# MÉMOIRE GLOBALE - INNODB BUFFER POOL
# ----------------------------------------------------------------------------
innodb_buffer_pool_size     = 44G
# Mixed : 68% RAM (compromis 62.5% OLTP et 75% OLAP)
# 64GB × 0.68 ≈ 44GB
# Laisse plus de marge pour connexions et OS

innodb_buffer_pool_instances = 16
# 44GB / 16 = 2.75GB par instance

innodb_buffer_pool_chunk_size = 128M

innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1

# 🔥 CRITIQUE MIXED : Protection pollution buffer pool
innodb_old_blocks_time      = 1000
# Délai avant promouvoir page vers "young" list
# Évite que scans OLAP évincent données OLTP chaudes
# 1000ms = bon compromis

innodb_old_blocks_pct       = 37
# 37% buffer pool = "old" sublist (défaut)
# Scans OLAP remplissent d'abord old sublist

# ----------------------------------------------------------------------------
# MÉMOIRE PAR CONNEXION (COMPROMIS)
# ----------------------------------------------------------------------------
read_buffer_size            = 2M
# Compromis 256K (OLTP) et 8M (OLAP)
# Suffisant pour scans légers sans gaspillage

read_rnd_buffer_size        = 4M
# Compromis 512K (OLTP) et 16M (OLAP)

sort_buffer_size            = 16M
# 🔥 Compromis crucial : 512K (OLTP) vs 64M (OLAP)
# 16M permet tris moyens sans exploser mémoire
# Requêtes OLAP complexes peuvent allouer plus via SET SESSION

join_buffer_size            = 16M
# Même logique que sort_buffer_size

# Calcul mémoire :
# 2M + 4M + 16M + 16M = 38MB par connexion
# 300 connexions × 38MB = 11.4GB
# 44GB buffer + 11.4GB conn + 4GB OS = 59.4GB / 64GB → OK

# ----------------------------------------------------------------------------
# TEMPORAIRES ET HEAP
# ----------------------------------------------------------------------------
tmp_table_size              = 256M
max_heap_table_size         = 256M
# Compromis 64M (OLTP) et 1G (OLAP)
# Agrégations moyennes en mémoire

# 🆕 MariaDB 11.8
max_tmp_space_usage         = 20G
# Limite espace temporaire par connexion
# Protège contre requêtes OLAP explosives

# ----------------------------------------------------------------------------
# LOGS INNODB - REDO LOG
# ----------------------------------------------------------------------------
innodb_log_file_size        = 1536M
# Compromis 1G (OLTP) et 2G (OLAP)
# 1.5G × 2 = 3GB total redo log

innodb_log_files_in_group   = 2

innodb_log_buffer_size      = 96M
# Compromis 64M (OLTP) et 128M (OLAP)

innodb_flush_log_at_trx_commit = 1
# 🔥 Mixed : ACID strict recommandé
# Si OLAP rechargeable : considérer = 2
# Mais transactions OLTP généralement critiques

# ----------------------------------------------------------------------------
# I/O ET DISQUES
# ----------------------------------------------------------------------------
innodb_io_capacity          = 6000
# NVMe : Bon pour OLTP et scans OLAP

innodb_io_capacity_max      = 12000

innodb_flush_method         = O_DIRECT

innodb_flush_neighbors      = 0
# SSD/NVMe optimal

# 🆕 MariaDB 11.8
innodb_alter_copy_bulk      = ON

# Read-ahead : Compromis
innodb_read_ahead_threshold = 32
# Entre 0 (OLAP agressif) et 56 (OLTP conservateur)
# Déclenche read-ahead pour scans moyens

# ----------------------------------------------------------------------------
# UNDO LOG
# ----------------------------------------------------------------------------
innodb_undo_tablespaces     = 2
innodb_undo_log_truncate    = ON
innodb_max_undo_log_size    = 1536M

# ----------------------------------------------------------------------------
# DURABILITÉ ET BINARY LOGS
# ----------------------------------------------------------------------------
sync_binlog                 = 1
# Mixed : Transactions OLTP critiques généralement

log_bin                     = /var/log/mysql/mysql-bin
binlog_format               = ROW
binlog_row_image            = MINIMAL

expire_logs_days            = 5
# Compromis 7j (OLTP) et 3j (OLAP)

max_binlog_size             = 768M

# ----------------------------------------------------------------------------
# RÉPLICATION (si applicable)
# ----------------------------------------------------------------------------
server-id                   = 1
log_slave_updates           = ON
gtid_domain_id              = 0
gtid_strict_mode            = ON
relay_log                   = /var/log/mysql/relay-bin
relay_log_recovery          = ON

# ----------------------------------------------------------------------------
# SÉCURITÉ (🆕 MariaDB 11.8)
# ----------------------------------------------------------------------------
# TLS activé par défaut
# ssl_cert = /etc/mysql/ssl/server-cert.pem
# ssl_key = /etc/mysql/ssl/server-key.pem
# ssl_ca = /etc/mysql/ssl/ca-cert.pem

# ----------------------------------------------------------------------------
# QUERY CACHE (⚠️ DÉPRÉCIÉ)
# ----------------------------------------------------------------------------
query_cache_type            = 0
query_cache_size            = 0

# ----------------------------------------------------------------------------
# OPTIMIZER ET STATISTIQUES
# ----------------------------------------------------------------------------
optimizer_search_depth      = 32
# Compromis 62 (OLTP strict) et 62 (OLAP complet)
# 32 bon équilibre (queries moyennement complexes)

optimizer_switch             = 'mrr=on,mrr_cost_based=on,index_condition_pushdown=on'

innodb_stats_persistent     = ON
innodb_stats_auto_recalc    = ON
innodb_stats_persistent_sample_pages = 50
# Compromis 20 (OLTP) et 100 (OLAP)

# ----------------------------------------------------------------------------
# MONITORING ET PERFORMANCE SCHEMA
# ----------------------------------------------------------------------------
performance_schema          = ON

# ----------------------------------------------------------------------------
# LOGS
# ----------------------------------------------------------------------------
log_error                   = /var/log/mysql/error.log

slow_query_log              = ON
slow_query_log_file         = /var/log/mysql/slow-query.log
long_query_time             = 2
# Mixed : 2 secondes
# Capture requêtes OLTP anormalement lentes (> 0.5s)
# Et requêtes OLAP problématiques (> 10s normal)

log_slow_verbosity          = query_plan,explain
log_queries_not_using_indexes = ON
# Utile pour identifier requêtes OLAP mal optimisées

# ----------------------------------------------------------------------------
# TIMEOUTS
# ----------------------------------------------------------------------------
wait_timeout                = 3600
# 1 heure : Compromis OLTP (600s) et OLAP (28800s)

interactive_timeout         = 3600

connect_timeout             = 10

net_read_timeout            = 60
net_write_timeout           = 120

# ----------------------------------------------------------------------------
# AUTRES PARAMÈTRES
# ----------------------------------------------------------------------------
max_allowed_packet          = 128M
# Compromis 64M (OLTP) et 256M (OLAP)

lower_case_table_names      = 0

sql_mode                    = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'

# ----------------------------------------------------------------------------
# INNODB AVANCÉ
# ----------------------------------------------------------------------------
innodb_adaptive_hash_index  = ON
# Mixed : Utile pour requêtes OLTP répétitives
# Peu d'impact négatif sur OLAP

innodb_change_buffering     = all
# Utile pour écritures OLTP

innodb_print_all_deadlocks  = ON
# Debug contention entre OLTP et OLAP

# ----------------------------------------------------------------------------
# ISOLATION (🔥 IMPORTANT MIXED)
# ----------------------------------------------------------------------------
transaction_isolation       = 'READ-COMMITTED'
# 🔥 Plutôt que REPEATABLE-READ (défaut)
# Réduit contention entre transactions OLTP et queries OLAP longues
# OLAP lit généralement données historiques (cohérence moins critique)

# Si REPEATABLE-READ requis pour OLTP :
# Garder défaut, mais surveiller lock waits

# ----------------------------------------------------------------------------
[mysqldump]
quick
quote-names
max_allowed_packet          = 128M

# ----------------------------------------------------------------------------
[mysql]
default-character-set       = utf8mb4

# ----------------------------------------------------------------------------
# FIN DE CONFIGURATION
# ============================================================================
```

---

## 🏗️ Approche 2 : Architecture séparation Read/Write (Recommandée)

### Architecture avec Read Replicas

**La solution optimale pour Mixed Workload :** séparer physiquement OLTP et OLAP.

```
┌─────────────────────────────────────────────────────┐
│                    Application                      │
│                                                     │
│  ┌─────────────────┐        ┌────────────────────┐  │
│  │  OLTP Requests  │        │  OLAP Requests     │  │
│  │  (Transactions) │        │  (Analytics)       │  │
│  └────────┬────────┘        └─────────────┬──────┘  │
└───────────┼───────────────────────────────┼─────────┘
            │                               │
            │                               │
    ┌───────▼────────┐             ┌────────▼───────────┐
    │   PRIMARY      │             │   READ REPLICA     │
    │   (Master)     │◄────────────│   (Analytics)      │
    │                │ Réplication │                    │
    ├────────────────┤             ├────────────────────┤
    │ Config OLTP    │             │ Config OLAP        │
    │ - 32GB RAM     │             │ - 128GB RAM        │
    │ - Thread pool  │             │ - Big buffers      │
    │ - Flush = 1    │             │ - Read-ahead = 0   │
    │ - Max conn 500 │             │ - Max conn 50      │
    └────────────────┘             └────────────────────┘
         WRITE                           READ ONLY
```

### Configuration Primary (OLTP optimisé)

```ini
# PRIMARY - OLTP Configuration
[mysqld]
server-id                   = 1

# Optimisations OLTP (voir D.1)
innodb_buffer_pool_size     = 20G  # 32GB serveur
max_connections             = 500
thread_handling             = pool-of-threads
sort_buffer_size            = 512K
join_buffer_size            = 256K

innodb_flush_log_at_trx_commit = 1
sync_binlog                 = 1

# Binlog pour réplication
log_bin                     = /var/log/mysql/mysql-bin
binlog_format               = ROW
gtid_strict_mode            = ON
```

### Configuration Read Replica (OLAP optimisé)

```ini
# READ REPLICA - OLAP Configuration
[mysqld]
server-id                   = 2
read_only                   = ON  # 🔥 Sécurité : read-only strict

# Optimisations OLAP (voir D.2)
innodb_buffer_pool_size     = 96G  # 128GB serveur
max_connections             = 100
sort_buffer_size            = 64M
join_buffer_size            = 64M
tmp_table_size              = 1G

innodb_flush_log_at_trx_commit = 2  # Relaxed OK (replica)
sync_binlog                 = 0

innodb_read_ahead_threshold = 0

# Réplication
relay_log                   = /var/log/mysql/relay-bin
relay_log_recovery          = ON
log_slave_updates           = OFF  # Pas de cascade
```

---

## 🔀 MaxScale : Routing intelligent Read/Write

### Architecture avec MaxScale

```
                        ┌─────────────────┐
                        │   Application   │
                        └────────┬────────┘
                                 │
                                 │ Port 3306
                        ┌────────▼────────┐
                        │    MaxScale     │
                        │  (Read/Write    │
                        │    Splitting)   │
                        └────┬──────┬─────┘
                             │      │
                ┌────────────┘      └────────────┐
                │                                │
         ┌──────▼──────┐                 ┌───────▼─────┐
         │   PRIMARY   │                 │   REPLICA   │
         │   (Write)   │────Replication─►│   (Read)    │
         └─────────────┘                 └─────────────┘
```

### Configuration MaxScale

```ini
# /etc/maxscale.cnf

[maxscale]
threads                 = 4
log_info                = true

# Serveurs backend
[primary]
type                    = server
address                 = 10.0.1.10
port                    = 3306
protocol                = MariaDBBackend

[replica1]
type                    = server
address                 = 10.0.1.11
port                    = 3306
protocol                = MariaDBBackend

# Monitoring
[MariaDB-Monitor]
type                    = monitor
module                  = mariadbmon
servers                 = primary,replica1
user                    = maxscale_monitor
password                = monitor_password
monitor_interval        = 2000
auto_failover           = true
auto_rejoin             = true

# Service Read/Write Split
[Read-Write-Service]
type                    = service
router                  = readwritesplit
servers                 = primary,replica1
user                    = maxscale_user
password                = maxscale_password

# 🔥 CONFIGURATION CRITIQUE MIXED WORKLOAD
master_accept_reads     = false
# Forcer lectures vers replicas (sauf transactions)

transaction_replay      = true
# Rejouer transaction si replica échoue

delayed_retry           = true
delayed_retry_timeout   = 60
# Retry si replica saturé

causal_reads            = fast
# Garantir lecture propres données écrites

max_slave_connections   = 100
# Limite connexions vers replicas

# Critères routage
[Read-Write-Listener]
type                    = listener
service                 = Read-Write-Service
protocol                = MariaDBClient
port                    = 3306
address                 = 0.0.0.0
```

### Utilisation application

```python
# L'application connecte MaxScale (transparent)
import mariadb

# Connexion unique vers MaxScale
conn = mariadb.connect(
    host = "maxscale-host",
    port = 3306,
    user = "app_user",
    password = "app_password",
    database = "mydb"
)

cursor = conn.cursor()

# Requête OLTP (routée vers PRIMARY)
cursor.execute("INSERT INTO orders (customer_id, amount) VALUES (123, 99.99)")
conn.commit()

# Requête OLAP (routée vers REPLICA automatiquement)
cursor.execute("""
    SELECT 
        DATE(order_date) AS day,
        SUM(amount) AS revenue
    FROM orders
    WHERE order_date >= CURDATE() - INTERVAL 30 DAY
    GROUP BY day
""")
results = cursor.fetchall()

# MaxScale route intelligemment sans changement code application !
```

### 🆕 MaxScale 25.01 - Nouvelles fonctionnalités

```ini
# Workload Capture & Replay (test charge réaliste)
[Capture-Filter]
type                    = filter
module                  = workloadcapture
output_file             = /var/log/maxscale/workload.sql
output_format           = sql

# Diff Router (comparer versions MariaDB)
[Diff-Service]
type                    = service
router                  = diffsplit
servers                 = mariadb-11.4,mariadb-11.8
user                    = diff_user
password                = diff_password
```

---

## 🔍 Explication des compromis clés

### 1. Buffer Pool : 68% RAM (entre 62.5% et 75%)

```ini
# OLTP pur : 62.5% (20G / 32G)
# OLAP pur : 75% (96G / 128G)
# Mixed : 68% (44G / 64G)
```

**Raison du compromis :**
- Plus que OLTP : Cache données analytiques répétées
- Moins qu'OLAP : Marge pour connexions et OS
- Buffer pool pollution géré par `innodb_old_blocks_time`

### 2. Sort/Join Buffers : 16MB (entre 512K et 64M)

```ini
# OLTP : 512K
# OLAP : 64M
# Mixed : 16M
```

**Impact par use case :**

| Scénario | Buffer 512K | Buffer 16M | Buffer 64M |
|----------|------------|------------|------------|
| **OLTP simple** | ✅ Optimal | ✅ OK (gaspillage) | ❌ Mémoire excessive |
| **OLAP léger** | ❌ Tri sur disque | ✅ Optimal | ✅ Marge |
| **OLAP complexe** | ❌ Très lent | ⚠️ Limite | ✅ Optimal |

**Flexibilité session-level :**
```sql
-- Requête OLAP ponctuelle nécessitant plus
SET SESSION sort_buffer_size = 67108864;  -- 64M
SET SESSION join_buffer_size = 67108864;

SELECT /* requête complexe */ ...;

-- Revient à défaut après déconnexion
```

### 3. Isolation READ-COMMITTED vs REPEATABLE-READ

```ini
transaction_isolation = 'READ-COMMITTED'
```

**Pourquoi changer l'isolation par défaut :**

| Aspect | REPEATABLE-READ | READ-COMMITTED |
|--------|-----------------|----------------|
| **Cohérence** | Snapshot transaction | Read latest committed |
| **Gap locks** | Oui (contention++) | Non (moins de locks) |
| **Phantom reads** | Protégé | Possible |
| **Requêtes longues OLAP** | Bloque OLTP | Impact minimal |

**Exemple contention :**
```sql
-- Session 1 (OLAP) - REPEATABLE-READ
START TRANSACTION;
SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- Acquiert gap locks sur plage status='pending'
-- Requête prend 30 secondes...

-- Session 2 (OLTP) - Bloquée !
INSERT INTO orders (status) VALUES ('pending');
-- ATTEND gap lock release (30 secondes) ❌

-- Avec READ-COMMITTED :
-- Pas de gap locks, INSERT immédiat ✅
```

**Trade-off :**
- ✅ Moins de contention OLTP/OLAP
- ⚠️ Phantom reads possibles (généralement acceptable analytics)

### 4. Protection Buffer Pool : innodb_old_blocks_time

```ini
innodb_old_blocks_time = 1000  # 1 seconde
innodb_old_blocks_pct = 37     # 37% buffer pool
```

**Mécanisme LRU protection :**

```
Buffer Pool divisé :
├─ Young Sublist (63%) : Données chaudes OLTP
└─ Old Sublist (37%) : Nouvelles pages

Scan OLAP charge 10GB données :
1. Pages vont dans Old Sublist
2. Si accédées > 1000ms → Promotion vers Young
3. Si scan unique → Restent Old, évincées rapidement

Résultat :
- Données OLTP chaudes protégées
- Scans OLAP ne "polluent" pas cache
```

**Sans protection :**
```
Scan OLAP 10GB :
→ Éjecte 10GB données OLTP du buffer pool
→ Queries OLTP lisent disque (1000× plus lent)
→ Latence OLTP dégradée ❌
```

### 5. Read-Ahead : 32 (entre 0 et 56)

```ini
# OLAP : 0 (agressif)
# OLTP : 56 (conservateur)
# Mixed : 32 (modéré)
```

**Comportement :**
- Déclenche read-ahead après 32 pages séquentielles
- Bénéficie rapports/analytics moyens
- N'impacte pas trop queries OLTP ponctuelles

---

## 💻 Recommandations matérielles Mixed

### Approche 1 : Serveur unique (budget contraint)

```
Configuration compromis :
────────────────────────────
CPU    : 16-24 cores @ 3.0 GHz
RAM    : 64-128 GB DDR4 ECC
Disque : 2TB NVMe (bonne IOPS + throughput)
Réseau : 10 Gbps

Prix : 5000-8000€
────────────────────────────

Limites :
- Contention CPU/RAM entre OLTP et OLAP
- Pas d'isolation workload
- Scaling limité

Adapté si :
- Charge OLAP légère (< 20% total)
- Budget serré
- Volumétrie modérée (< 1TB)
```

### Approche 2 : Séparation Primary + Replica (Recommandée)

```
PRIMARY (OLTP) :
────────────────────────────
CPU    : 8-16 cores @ 3.5 GHz (haute fréquence)
RAM    : 32-64 GB
Disque : 1TB NVMe (IOPS critiques)
Prix   : 3000-5000€

READ REPLICA (OLAP) :
────────────────────────────
CPU    : 32-48 cores @ 2.5 GHz (beaucoup cores)
RAM    : 128-256 GB
Disque : 4TB NVMe (débit séquentiel)
Prix   : 8000-12000€

TOTAL  : 11000-17000€
────────────────────────────

Avantages :
✅ Isolation complète workloads
✅ Tuning spécialisé par serveur
✅ Scaling indépendant
✅ Pas de contention

Adapté si :
- Charge OLAP significative (> 30%)
- SLA OLTP strict
- Croissance prévue
```

### Approche 3 : Multi-replicas (Production)

```
┌──────────────┐
│   PRIMARY    │ OLTP optimisé
└──────┬───────┘
       │
       ├─────► REPLICA 1  Analytics générale
       ├─────► REPLICA 2  Dashboards temps réel
       └─────► REPLICA 3  Data Science / ML

Coût : 15000-30000€
Bénéfices :
- Isolation par use case
- Haute disponibilité
- Scaling horizontal reads
```

---

## 📊 Stratégies de séparation Hot/Cold Data

### Pattern 1 : Partitionnement temporel

```sql
-- Données récentes (hot) : InnoDB optimisé OLTP
CREATE TABLE orders_recent (
    id BIGINT PRIMARY KEY,
    order_date DATE NOT NULL,
    customer_id INT,
    amount DECIMAL(10,2),
    ...
) ENGINE=InnoDB;

-- Données anciennes (cold) : ColumnStore optimisé OLAP
CREATE TABLE orders_archive (
    id BIGINT,
    order_date DATE NOT NULL,
    customer_id INT,
    amount DECIMAL(10,2),
    ...
) ENGINE=ColumnStore;

-- ETL nocturne : Déplacement données > 30 jours
INSERT INTO orders_archive
SELECT * FROM orders_recent
WHERE order_date < CURDATE() - INTERVAL 30 DAY;

DELETE FROM orders_recent
WHERE order_date < CURDATE() - INTERVAL 30 DAY;
```

### Pattern 2 : Tables dénormalisées pour analytics

```sql
-- Table normalisée (OLTP)
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    order_date DATE,
    amount DECIMAL(10,2)
) ENGINE=InnoDB;

-- Table dénormalisée pré-calculée (OLAP)
CREATE TABLE orders_analytics (
    day DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(15,2),
    unique_customers INT,
    avg_order_value DECIMAL(10,2),
    top_product_id INT,
    updated_at TIMESTAMP
) ENGINE=InnoDB;

-- Event scheduler : Refresh toutes les heures
CREATE EVENT refresh_analytics
ON SCHEDULE EVERY 1 HOUR
DO
  INSERT INTO orders_analytics
  SELECT 
      DATE(order_date),
      COUNT(*),
      SUM(amount),
      COUNT(DISTINCT customer_id),
      AVG(amount),
      (SELECT product_id FROM orders o2 
       WHERE DATE(o2.order_date) = DATE(o.order_date)
       GROUP BY product_id ORDER BY COUNT(*) DESC LIMIT 1),
      NOW()
  FROM orders o
  WHERE DATE(order_date) >= CURDATE() - INTERVAL 1 DAY
  GROUP BY DATE(order_date)
  ON DUPLICATE KEY UPDATE
      total_orders = VALUES(total_orders),
      total_revenue = VALUES(total_revenue),
      unique_customers = VALUES(unique_customers),
      avg_order_value = VALUES(avg_order_value),
      top_product_id = VALUES(top_product_id),
      updated_at = NOW();
```

### Pattern 3 : Indexes différenciés

```sql
-- Table mixte avec indexes ciblés
CREATE TABLE user_events (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_date DATETIME NOT NULL,
    properties JSON,
    
    -- Index OLTP (point lookups)
    INDEX idx_user_recent (user_id, event_date),
    
    -- Index OLAP (analytics)
    INDEX idx_analytics (event_date, event_type)
    
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(event_date)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    ...
);
```

---

## 📈 Monitoring spécifique Mixed Workload

### Métriques essentielles

```sql
-- Ratio OLTP vs OLAP queries
SELECT 
    CASE 
        WHEN AVG_TIMER_WAIT / 1000000000000 < 1 THEN 'OLTP'
        ELSE 'OLAP'
    END AS query_type,
    COUNT(*) AS count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER(), 2) AS pct,
    ROUND(AVG(AVG_TIMER_WAIT / 1000000000000), 2) AS avg_seconds
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT NOT LIKE '%performance_schema%'
GROUP BY query_type;

-- Exemple résultat :
-- query_type | count | pct   | avg_seconds
-- OLTP       | 15240 | 68.5% | 0.08
-- OLAP       | 7010  | 31.5% | 3.45
```

### Dashboard Grafana Mixed

```yaml
# Panel 1 : Latence OLTP vs OLAP
queries:
  - OLTP P95: mysql_query_p95_milliseconds{query_type="oltp"}
  - OLAP P95: mysql_query_p95_seconds{query_type="olap"}
    
# Panel 2 : Buffer Pool Pressure
queries:
  - Hit Rate: mysql_buffer_pool_hit_rate
  - Dirty Pages %: mysql_buffer_pool_dirty_pages_pct
  - Pages Read/sec: mysql_innodb_buffer_pool_reads
    
# Panel 3 : Lock Contention
queries:
  - Row Lock Waits/sec: rate(mysql_innodb_row_lock_waits[5m])
  - Lock Wait Time: mysql_innodb_row_lock_time_avg
    
# Panel 4 : Temp Tables
queries:
  - Temp on Disk %: mysql_tmp_disk_tables_pct
  - Temp Space Used: mysql_tmp_space_used_gb
```

### Alertes spécifiques

```yaml
groups:
  - name: mariadb_mixed
    rules:
      # OLTP dégradé
      - alert: OLTPLatencyHigh
        expr: mysql_query_p95_milliseconds{type="oltp"} > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "OLTP queries P95 > 100ms"
          
      # Contention locks
      - alert: HighLockContention
        expr: rate(mysql_innodb_row_lock_waits[5m]) > 50
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Lock waits > 50/sec (OLTP/OLAP contention?)"
          
      # Buffer pool pressure
      - alert: BufferPoolThrashing
        expr: rate(mysql_innodb_buffer_pool_reads[5m]) > 1000
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Buffer pool thrashing (OLAP evicting OLTP data?)"
```

---

## ⚠️ Points de vigilance Mixed Workload

### 1. Identifier et séparer workloads

❌ **Tout mélangé :**
```python
# Application fait OLTP et OLAP sur même connexion
def get_user_orders(user_id):
    # OLTP
    conn.execute("SELECT * FROM orders WHERE user_id = ?", user_id)
    
def generate_report():
    # OLAP (même connexion pool !)
    conn.execute("""
        SELECT DATE(order_date), SUM(amount), COUNT(*)
        FROM orders
        WHERE order_date >= CURDATE() - INTERVAL 90 DAY
        GROUP BY DATE(order_date)
    """)
```

✅ **Séparation explicite :**
```python
# Pools séparés (ou MaxScale routing)
oltp_pool = create_pool(host="primary", max_connections=100)
olap_pool = create_pool(host="replica", max_connections=20)

def get_user_orders(user_id):
    conn = oltp_pool.get_connection()
    return conn.execute("SELECT * FROM orders WHERE user_id = ?", user_id)
    
def generate_report():
    conn = olap_pool.get_connection()
    return conn.execute("SELECT ... analytics query ...")
```

### 2. Gérer priorités avec resource groups

```sql
-- MariaDB 10.5+ : Resource Groups (expérimental)
-- Limiter ressources queries OLAP

-- Créer resource group OLAP
CREATE RESOURCE GROUP olap_group
  TYPE = USER
  VCPU = 0-15  -- Cores 0-15
  THREAD_PRIORITY = 10;  -- Basse priorité

-- Assigner user analytics
CREATE USER 'analytics'@'%' IDENTIFIED BY 'password';
GRANT SELECT ON mydb.* TO 'analytics'@'%';
SET RESOURCE GROUP olap_group FOR 'analytics'@'%';

-- User OLTP (default group, haute priorité)
CREATE USER 'app'@'%' IDENTIFIED BY 'password';
```

### 3. Scheduler queries OLAP hors pics

❌ **Analytics en journée (pics OLTP) :**
```python
# Rapport génération 10h-16h (pic OLTP)
def daily_report():
    # Requête lourde pendant activité max utilisateurs
    # → Dégrade latence OLTP ❌
```

✅ **Fenêtres analytiques optimisées :**
```python
import schedule

# Rapports lourds : 1h-6h du matin (faible OLTP)
schedule.every().day.at("02:00").do(heavy_analytics)

# Dashboards temps réel : requêtes légères ou pré-calculées
def realtime_dashboard():
    # Requête sur table summary (rapide)
    return fetch_from_summary_table()
```

### 4. Limiter durée queries OLAP

```sql
-- Définir timeout global ou par session
SET GLOBAL max_statement_time = 300;  -- 5 minutes max

-- Ou session-level pour user analytics
SET SESSION max_statement_time = 600;  -- 10 minutes

-- Requête dépassant timeout → Annulée automatiquement
-- Évite queries runaway consommant ressources indéfiniment
```

### 5. Monitoring lag réplication

```sql
-- Replica lag (critique si read replica OLAP)
SHOW SLAVE STATUS\G

-- Seconds_Behind_Master doit rester < 5s
-- Si > 30s :
-- → Replica surchargée par queries OLAP
-- → Scaler replica ou limiter charge

-- Monitoring automated
SELECT 
    @@hostname AS replica,
    TIMESTAMPDIFF(SECOND, 
        MASTER_POS_WAIT(master_log_file, master_log_pos, 1),
        NOW()
    ) AS lag_seconds
FROM (
    SELECT 
        Master_Log_File AS master_log_file,
        Read_Master_Log_Pos AS master_log_pos
    FROM SHOW SLAVE STATUS
) AS repl_status;
```

### 6. Planifier maintenance différenciée

```bash
# PRIMARY (OLTP) : Maintenance légère, fenêtre courte
# 3h-5h dimanche (2h fenêtre)
0 3 * * 0 /usr/bin/mariadb -e "OPTIMIZE TABLE hot_tables"

# REPLICA (OLAP) : Maintenance lourde possible
# Peut prendre plus de temps, read-only donc moins critique
0 2 * * 0 /usr/bin/mariadb -e "ANALYZE TABLE all_tables PERSISTENT FOR ALL"
0 4 * * 0 /usr/bin/mariadb -e "OPTIMIZE TABLE large_analytics_tables"
```

---

## ✅ Points clés à retenir

- ⚖️ **Compromis équilibré** : Config entre OLTP (D.1) et OLAP (D.2)
- 🏗️ **Séparation physique recommandée** : Primary OLTP + Replica OLAP
- 🔀 **MaxScale essentiel** : Routing intelligent read/write split
- 🛡️ **Protection buffer pool** : innodb_old_blocks_time évite pollution
- 🔓 **READ-COMMITTED** : Réduit contention locks OLTP/OLAP
- 📊 **Pré-agrégation** : Tables summary pour analytics fréquentes
- ⏰ **Scheduler intelligent** : OLAP hors pics OLTP (1h-6h)
- 🔍 **Monitoring dédié** : Métriques séparées OLTP vs OLAP
- 🗄️ **Hot/Cold data** : Partitionnement + ColumnStore pour historique
- 📈 **Scaling progressif** : Commencer unique → Ajouter replicas selon croissance

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Read/Write Splitting](https://mariadb.com/kb/en/mariadb-maxscale-6-readwritesplit/)
- [Transaction Isolation Levels](https://mariadb.com/kb/en/set-transaction/)
- [InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [Resource Groups](https://mariadb.com/kb/en/create-resource-group/)

### MaxScale
- [MaxScale Documentation](https://mariadb.com/kb/en/mariadb-maxscale/)
- [ReadWriteSplit Router](https://mariadb.com/kb/en/mariadb-maxscale-6-readwritesplit/)
- [🆕 MaxScale 25.01](https://mariadb.com/docs/server/release-notes/mariadb-maxscale-25-01/)

### Architectures
- [Scaling Reads with Replicas](https://mariadb.com/kb/en/read-write-splitting-with-mariadb-maxscale/)
- [ColumnStore for Analytics](https://mariadb.com/kb/en/mariadb-columnstore/)

### Autres annexes
- [D.1 - Configuration OLTP](./01-configuration-oltp.md)
- [D.2 - Configuration OLAP](./02-configuration-olap.md)
- [Section 14.4 - MaxScale](/14-haute-disponibilite/04-maxscale.md)

---

## ➡️ Section suivante

**[D.4 - Configuration Développement Local](./04-configuration-developpement-local.md)** : Dev, testing, CI/CD

---

**MariaDB** : Version 11.8 LTS

⏭️ [Développement local](/annexes/configuration-reference/04-configuration-developpement-local.md)

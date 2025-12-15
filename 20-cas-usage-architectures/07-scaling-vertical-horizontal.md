🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.7 Scaling Vertical vs Horizontal

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Chapitres 13-14 (Réplication, Haute Disponibilité), Section 20.1 (OLTP vs OLAP), notions de capacity planning

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les différences fondamentales entre scaling vertical et horizontal
- Identifier les indicateurs qui signalent le besoin de scaling
- Choisir la stratégie de scaling adaptée selon le contexte
- Implémenter le scaling horizontal avec read replicas et sharding
- Utiliser Galera Cluster pour le scaling en écriture
- Optimiser les coûts en combinant les approches de scaling
- Planifier la capacité et anticiper les besoins de croissance

---

## Introduction

Le **scaling** (mise à l'échelle) est la capacité d'un système à gérer une charge croissante. C'est l'un des défis majeurs pour toute application en croissance. Deux approches fondamentales existent : **vertical** (scale-up) et **horizontal** (scale-out).

MariaDB 11.8 LTS offre des mécanismes robustes pour les deux approches : optimisation des ressources pour le scaling vertical, et réplication/clustering pour le scaling horizontal.

### Définitions

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    SCALING VERTICAL VS HORIZONTAL                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  SCALING VERTICAL (Scale-Up)                                               │
│  ═══════════════════════════                                               │
│  Augmenter les ressources d'un serveur unique                              │
│                                                                            │
│  Avant :                     Après :                                       │
│  ┌─────────────────┐         ┌─────────────────┐                           │
│  │    Serveur      │         │    Serveur      │                           │
│  │   ┌─────────┐   │         │   ┌─────────┐   │                           │
│  │   │ 16 CPU  │   │  ──►    │   │ 64 CPU  │   │                           │
│  │   │ 64 GB   │   │         │   │ 256 GB  │   │                           │
│  │   │ 1 TB SSD│   │         │   │ 4 TB NVMe│  │                           │
│  │   └─────────┘   │         │   └─────────┘   │                           │
│  └─────────────────┘         └─────────────────┘                           │
│                                                                            │
│  ✅ Simple : pas de changement d'architecture                              │
│  ✅ Pas de complexité distribuée                                           │
│  ❌ Limite physique (plus gros serveur disponible)                         │
│  ❌ Point unique de défaillance                                            │
│  ❌ Coût exponentiel aux hautes specs                                      │
│                                                                            │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                            │
│  SCALING HORIZONTAL (Scale-Out)                                            │
│  ══════════════════════════════                                            │
│  Ajouter plus de serveurs au cluster                                       │
│                                                                            │
│  Avant :                     Après :                                       │
│  ┌─────────────────┐         ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │
│  │    Serveur      │         │Node 1│ │Node 2│ │Node 3│ │Node 4│           │
│  │   ┌─────────┐   │  ──►    │      │ │      │ │      │ │      │           │
│  │   │ 16 CPU  │   │         │16 CPU│ │16 CPU│ │16 CPU│ │16 CPU│           │
│  │   │ 64 GB   │   │         │64 GB │ │64 GB │ │64 GB │ │64 GB │           │
│  │   └─────────┘   │         └──────┘ └──────┘ └──────┘ └──────┘           │
│  └─────────────────┘         Total: 64 CPU, 256 GB (distribué)             │
│                                                                            │
│  ✅ Scalabilité quasi-illimitée                                            │
│  ✅ Haute disponibilité native                                             │
│  ✅ Coût linéaire                                                          │
│  ❌ Complexité de l'architecture distribuée                                │
│  ❌ Latence réseau entre nœuds                                             │
│  ❌ Cohérence des données (CAP theorem)                                    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Quand scaler ?

### Indicateurs de besoin de scaling

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- INDICATEURS DE SATURATION - QUAND SCALER ?
-- ═══════════════════════════════════════════════════════════════════════════

-- 1. CPU : Utilisation élevée
-- Alerte si > 70% sur période prolongée
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'Threads_running',      -- Threads actifs (vs max_connections)
    'Slow_queries',         -- Requêtes lentes
    'Handler_read_rnd_next' -- Full table scans (CPU intensive)
);

-- 2. Mémoire : Buffer Pool sous pression
-- Objectif : Hit ratio > 99%
SELECT 
    (1 - (
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_ratio;

-- Si < 99%, le buffer pool est trop petit → Scale vertical (RAM)

-- 3. I/O : Disque saturé
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'Innodb_data_reads',
    'Innodb_data_writes',
    'Innodb_data_pending_reads',
    'Innodb_data_pending_writes'  -- > 0 = I/O saturé
);

-- 4. Connexions : Pool épuisé
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_connected') AS current_connections,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME = 'max_connections') AS max_connections,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Max_used_connections') AS peak_connections,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Threads_connected') * 100.0 /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'max_connections')
    , 1) AS connection_usage_pct;

-- Si > 80% → Scale horizontal (read replicas) ou vertical (RAM)

-- 5. Réplication : Lag croissant
-- Si le replica ne suit plus → Scale vertical (replica) ou ajouter replicas
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master croissant = replica sous-dimensionné

-- 6. Temps de réponse dégradé
SELECT 
    schema_name,
    ROUND(SUM(sum_timer_wait) / SUM(count_star) / 1000000000, 3) AS avg_query_time_ms,
    SUM(count_star) AS total_queries
FROM performance_schema.events_statements_summary_by_digest
WHERE schema_name NOT IN ('mysql', 'information_schema', 'performance_schema')
GROUP BY schema_name
HAVING avg_query_time_ms > 100  -- Alerte si > 100ms moyenne
ORDER BY avg_query_time_ms DESC;
```

### Matrice de décision

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MATRICE DE DÉCISION SCALING                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Problème observé          │ Scale Vertical │ Scale Horizontal              │
│  ══════════════════════════│════════════════│═══════════════════════════    │
│  CPU saturé                │ ✅ Plus cores  │ ✅ Read replicas (reads)      │
│  Mémoire insuffisante      │ ✅ Plus RAM    │ ⚠️ Sharding (si > 1TB)        │
│  I/O disque saturé         │ ✅ NVMe/RAID   │ ✅ Sharding                   │
│  Connexions épuisées       │ ⚠️ Limité     │ ✅ Read replicas + ProxySQL    │
│  Writes saturent           │ ✅ Disque+CPU  │ ✅ Galera / Sharding          │
│  Reads saturent            │ ⚠️ Limité     │ ✅ Read replicas               │
│  Latence géographique      │ ❌ N/A        │ ✅ Replicas régionaux          │
│  Taille base > RAM         │ ⚠️ Coûteux   │ ✅ Sharding                     │
│  HA requise                │ ❌ SPOF       │ ✅ Cluster/Replicas            │
│                                                                             │
│  Légende :                                                                  │
│  ✅ Solution recommandée                                                    │
│  ⚠️ Possible mais limité                                                    │
│  ❌ Non applicable                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Scaling Vertical

### Optimisation des ressources MariaDB

```ini
# ═══════════════════════════════════════════════════════════════════════════
# CONFIGURATION OPTIMISÉE - SCALING VERTICAL
# /etc/mysql/mariadb.conf.d/scaled-vertical.cnf
# ═══════════════════════════════════════════════════════════════════════════

[mysqld]
# ═══════════════════════════════════════════════════════════════════════════
# MÉMOIRE (Exemple : Serveur 256 GB RAM)
# ═══════════════════════════════════════════════════════════════════════════

# Buffer Pool : 70-80% de la RAM pour DB dédiée
innodb_buffer_pool_size = 180G
innodb_buffer_pool_instances = 16  # 1 instance par 1-2 GB
innodb_buffer_pool_chunk_size = 1G

# Chargement à chaud au démarrage
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON

# Log Buffer (pour gros batch inserts)
innodb_log_buffer_size = 256M

# 🆕 MariaDB 11.8 : Contrôle de l'espace temporaire
max_tmp_space_usage = 16G
max_total_tmp_space_usage = 64G

# ═══════════════════════════════════════════════════════════════════════════
# CPU (Exemple : 64 cores)
# ═══════════════════════════════════════════════════════════════════════════

# Parallélisme InnoDB
innodb_read_io_threads = 16
innodb_write_io_threads = 16
innodb_purge_threads = 8

# Thread pool (mieux que thread-per-connection pour > 100 connexions)
thread_handling = pool-of-threads
thread_pool_size = 32  # ~50% des cores
thread_pool_max_threads = 1000
thread_pool_idle_timeout = 60

# Concurrence
innodb_thread_concurrency = 0  # Auto

# 🆕 MariaDB 11.8 : Optimisation pour NVMe
optimizer_disk_read_ratio = 0.0002

# ═══════════════════════════════════════════════════════════════════════════
# STOCKAGE (Exemple : NVMe RAID 10)
# ═══════════════════════════════════════════════════════════════════════════

# I/O Capacity (ajuster selon benchmark)
innodb_io_capacity = 20000       # NVMe haute performance
innodb_io_capacity_max = 40000

# Flush method (O_DIRECT pour éviter double buffering)
innodb_flush_method = O_DIRECT

# Redo Log (2-4 GB pour gros volumes)
innodb_log_file_size = 4G
innodb_log_files_in_group = 2

# Checkpointing
innodb_max_dirty_pages_pct = 75
innodb_max_dirty_pages_pct_lwm = 10

# Compression (économise I/O au prix de CPU)
# innodb_compression_default = ON  # Si CPU disponible

# ═══════════════════════════════════════════════════════════════════════════
# CONNEXIONS
# ═══════════════════════════════════════════════════════════════════════════

max_connections = 2000  # Augmenté grâce au thread pool
max_user_connections = 1800

# Timeouts pour libérer les connexions inactives
wait_timeout = 600
interactive_timeout = 3600

# ═══════════════════════════════════════════════════════════════════════════
# CACHES
# ═══════════════════════════════════════════════════════════════════════════

# Table cache
table_open_cache = 8000
table_definition_cache = 4000

# Query cache (désactivé en 11.8, utiliser ProxySQL pour caching)
# query_cache_type = OFF

# Join buffer (par connexion, attention à la RAM totale)
join_buffer_size = 4M
sort_buffer_size = 4M
read_buffer_size = 2M
read_rnd_buffer_size = 2M
```

### Limites du scaling vertical

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LIMITES DU SCALING VERTICAL                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Ressource    │ Limite pratique      │ Limite absolue                       │
│  ════════════ │ ════════════════════ │ ══════════════════════════════════   │
│  RAM          │ 2-4 TB (coût)        │ 12 TB (plus gros serveurs)           │
│  CPU          │ 128-256 cores        │ 512 cores (multi-socket)             │
│  Stockage     │ 100 TB SSD (coût)    │ Pétaoctets (SAN)                     │
│  IOPS         │ 1M IOPS NVMe         │ 10M IOPS (NVMe array)                │
│  Réseau       │ 100 Gbps             │ 400 Gbps                             │
│                                                                             │
│  Problèmes au-delà des limites pratiques :                                  │
│  ════════════════════════════════════════                                   │
│                                                                             │
│  💰 COÛT EXPONENTIEL                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │  Exemple AWS (EC2 + EBS) :                                        │      │
│  │  • r6g.xlarge  (4 CPU, 32 GB)   : ~$250/mois                      │      │
│  │  • r6g.4xlarge (16 CPU, 128 GB) : ~$1,000/mois (4x specs = 4x $)  │      │
│  │  • r6g.16xlarge (64 CPU, 512 GB): ~$4,500/mois (16x specs = 18x $)│      │
│  │                                                                   │      │
│  │  Le coût croît plus vite que les specs !                          │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ⏱️ DOWNTIME POUR UPGRADE                                                   │
│  • Migration vers instance plus grande = downtime                           │
│  • Exceptions : AWS Graviton, Azure Flex (online resize limité)             │
│                                                                             │
│  🎯 POINT UNIQUE DE DÉFAILLANCE                                             │
│  • Un seul serveur = si crash, tout tombe                                   │
│  • Même avec HA, le failover prend du temps                                 │
│                                                                             │
│  📈 RENDEMENTS DÉCROISSANTS                                                 │
│  • Locks, contention, latence interne                                       │
│  • Doubler la RAM ≠ doubler les performances                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Scaling Horizontal

### 1. Read Replicas

La méthode la plus simple pour scaler les lectures.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE READ REPLICAS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                            Applications                                     │
│                                │                                            │
│                                ▼                                            │
│                    ┌─────────────────────┐                                  │
│                    │      MaxScale /     │                                  │
│                    │      ProxySQL       │                                  │
│                    │   (Read/Write Split)│                                  │
│                    └──────────┬──────────┘                                  │
│                               │                                             │
│              ┌────────────────┼────────────────┐                            │
│              │                │                │                            │
│        Writes│          Reads │                │ Reads                      │
│              ▼                ▼                ▼                            │
│     ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                     │
│     │   Primary   │   │  Replica 1  │   │  Replica 2  │                     │
│     │    (RW)     │──►│    (RO)     │   │    (RO)     │                     │
│     │             │   │             │   │             │                     │
│     │  Writes +   │   │   Reads     │   │   Reads     │                     │
│     │  Reads      │   │   only      │   │   only      │                     │
│     └─────────────┘   └─────────────┘   └─────────────┘                     │
│            │                ▲                 ▲                             │
│            │                │                 │                             │
│            └────────────────┴─────────────────┘                             │
│                    Async Replication                                        │
│                                                                             │
│  Scaling :                                                                  │
│  • Ajouter des replicas = multiplier la capacité de lecture                 │
│  • Lectures : N replicas → N × capacité                                     │
│  • Écritures : 1 primary → pas de scaling                                   │
│                                                                             │
│  Ratio typique : 80-90% reads → 3-5 replicas suffisent                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```yaml
# maxscale-read-replicas.cnf
# Configuration MaxScale pour read/write split

[maxscale]
threads = auto

[primary]
type = server
address = primary.db.internal
port = 3306
protocol = MariaDBBackend

[replica1]
type = server
address = replica1.db.internal
port = 3306
protocol = MariaDBBackend

[replica2]
type = server
address = replica2.db.internal
port = 3306
protocol = MariaDBBackend

[replica3]
type = server
address = replica3.db.internal
port = 3306
protocol = MariaDBBackend

[replication-monitor]
type = monitor
module = mariadbmon
servers = primary, replica1, replica2, replica3
user = maxscale_monitor
password = secure_password
monitor_interval = 2000ms
auto_failover = true
auto_rejoin = true

[rw-split-service]
type = service
router = readwritesplit
servers = primary, replica1, replica2, replica3
user = maxscale_user
password = secure_password

# Politique de routing
master_accept_reads = false           # Writes seulement sur primary
slave_selection_criteria = ADAPTIVE   # Choix intelligent du replica
max_slave_replication_lag = 10s       # Exclure replicas en retard

# Retry sur erreur
transaction_replay = true
transaction_replay_max_size = 10Mi

[rw-split-listener]
type = listener
service = rw-split-service
protocol = MariaDBClient
port = 3306
```

### 2. Galera Cluster (Multi-Primary)

Scaling des écritures avec synchronisation synchrone.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GALERA CLUSTER - MULTI-PRIMARY                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                            Applications                                     │
│                                │                                            │
│                                ▼                                            │
│                    ┌─────────────────────┐                                  │
│                    │   Load Balancer     │                                  │
│                    │   (Round-robin)     │                                  │
│                    └──────────┬──────────┘                                  │
│                               │                                             │
│              ┌────────────────┼────────────────┐                            │
│              │                │                │                            │
│              ▼                ▼                ▼                            │
│     ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                     │
│     │   Node 1    │◄─►│   Node 2    │◄─►│   Node 3    │                     │
│     │    (RW)     │   │    (RW)     │   │    (RW)     │                     │
│     │             │   │             │   │             │                     │
│     │  Writes +   │   │  Writes +   │   │  Writes +   │                     │
│     │  Reads      │   │  Reads      │   │  Reads      │                     │
│     └─────────────┘   └─────────────┘   └─────────────┘                     │
│            ▲                ▲                 ▲                             │
│            │                │                 │                             │
│            └────────────────┴─────────────────┘                             │
│                 Synchronous Replication                                     │
│                 (Certification-based)                                       │
│                                                                             │
│  Caractéristiques :                                                         │
│  • Tous les nœuds acceptent les écritures                                   │
│  • Réplication synchrone (pas de perte de données)                          │
│  • Détection automatique des conflits                                       │
│                                                                             │
│  Attention :                                                                │
│  • Latence d'écriture = RTT réseau (tous les nœuds doivent certifier)       │
│  • Conflicts possibles (même row modifiée simultanément)                    │
│  • Scaling limité (~5-7 nœuds pour writes intensifs)                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```ini
# /etc/mysql/mariadb.conf.d/galera.cnf
# Configuration Galera Cluster optimisée pour scaling

[mysqld]
# Galera Provider
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "production-cluster"
wsrep_cluster_address = "gcomm://node1,node2,node3"
wsrep_node_name = "node1"
wsrep_node_address = "10.0.1.10"

# Réplication
wsrep_sst_method = mariabackup
wsrep_sst_auth = sst_user:sst_password

# Performance
wsrep_slave_threads = 8              # Parallélisme applyer
wsrep_provider_options = "gcache.size=2G; gcs.fc_limit=256"

# Tolérance aux conflits
wsrep_retry_autocommit = 3           # Retry sur deadlock

# InnoDB settings pour Galera
innodb_autoinc_lock_mode = 2         # Requis pour Galera
innodb_flush_log_at_trx_commit = 2   # Performance (Galera garantit durabilité)

# Binlog pour replicas asynchrones additionnels
log_bin = mysql-bin
binlog_format = ROW
```

### 3. Sharding

Division horizontale des données sur plusieurs serveurs.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SHARDING - DIVISION DES DONNÉES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                            Applications                                     │
│                                │                                            │
│                                ▼                                            │
│                    ┌─────────────────────┐                                  │
│                    │    Shard Router     │                                  │
│                    │   (ProxySQL /       │                                  │
│                    │    Application)     │                                  │
│                    └──────────┬──────────┘                                  │
│                               │                                             │
│         ┌─────────────────────┼─────────────────────┐                       │
│         │                     │                     │                       │
│         ▼                     ▼                     ▼                       │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐                │
│  │   Shard 1   │       │   Shard 2   │       │   Shard 3   │                │
│  │             │       │             │       │             │                │
│  │ user_id     │       │ user_id     │       │ user_id     │                │
│  │  1-1M       │       │  1M-2M      │       │  2M-3M      │                │
│  │             │       │             │       │             │                │
│  │ ┌─────────┐ │       │ ┌─────────┐ │       │ ┌─────────┐ │                │
│  │ │ Primary │ │       │ │ Primary │ │       │ │ Primary │ │                │
│  │ └────┬────┘ │       │ └────┬────┘ │       │ └────┬────┘ │                │
│  │      │      │       │      │      │       │      │      │                │
│  │ ┌────▼────┐ │       │ ┌────▼────┐ │       │ ┌────▼────┐ │                │
│  │ │ Replica │ │       │ │ Replica │ │       │ │ Replica │ │                │
│  │ └─────────┘ │       │ └─────────┘ │       │ └─────────┘ │                │
│  └─────────────┘       └─────────────┘       └─────────────┘                │
│                                                                             │
│  Stratégies de sharding :                                                   │
│  ═══════════════════════                                                    │
│                                                                             │
│  1. Range-based : user_id 1-1M → Shard 1, 1M-2M → Shard 2                   │
│     ✅ Simple, requêtes de range efficaces                                  │
│     ❌ Hotspots si distribution inégale                                     │
│                                                                             │
│  2. Hash-based : shard = hash(user_id) % N                                  │
│     ✅ Distribution uniforme                                                │
│     ❌ Requêtes de range sur tous les shards                                │
│                                                                             │
│  3. Directory-based : Table de mapping user_id → shard                      │
│     ✅ Flexible, migration facile                                           │
│     ❌ Lookup additionnel                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implémentation du sharding

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SHARDING : EXEMPLE D'IMPLÉMENTATION
-- ═══════════════════════════════════════════════════════════════════════════

-- Table de routage des shards (sur un serveur de configuration)
CREATE TABLE shard_config (
    shard_id INT PRIMARY KEY,
    shard_name VARCHAR(50) NOT NULL,
    host VARCHAR(255) NOT NULL,
    port INT DEFAULT 3306,
    min_key BIGINT NOT NULL,
    max_key BIGINT NOT NULL,
    status ENUM('active', 'migrating', 'readonly', 'offline') DEFAULT 'active',
    
    INDEX idx_key_range (min_key, max_key)
) ENGINE=InnoDB;

INSERT INTO shard_config VALUES
    (1, 'shard_1', 'shard1.db.internal', 3306, 1, 1000000, 'active'),
    (2, 'shard_2', 'shard2.db.internal', 3306, 1000001, 2000000, 'active'),
    (3, 'shard_3', 'shard3.db.internal', 3306, 2000001, 3000000, 'active');

-- Fonction de routing (côté application ou proxy)
DELIMITER //

CREATE FUNCTION get_shard_for_user(p_user_id BIGINT)
RETURNS INT
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_shard_id INT;
    
    SELECT shard_id INTO v_shard_id
    FROM shard_config
    WHERE p_user_id BETWEEN min_key AND max_key
      AND status = 'active'
    LIMIT 1;
    
    RETURN v_shard_id;
END //

DELIMITER ;

-- Utilisation
SELECT get_shard_for_user(1500000);  -- Retourne 2

-- ═══════════════════════════════════════════════════════════════════════════
-- SCHÉMA SUR CHAQUE SHARD
-- ═══════════════════════════════════════════════════════════════════════════

-- Chaque shard contient une partie des données
-- La shard_key (user_id) est incluse dans les PKs pour unicité globale

CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,  -- Unique globalement
    email VARCHAR(255) NOT NULL,
    name VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Données locales au shard
    UNIQUE KEY uk_email (email)
) ENGINE=InnoDB;

CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT,
    user_id BIGINT NOT NULL,  -- Shard key
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(12,2),
    
    PRIMARY KEY (order_id, user_id),  -- Include shard key in PK
    INDEX idx_user (user_id, order_date),
    
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) ENGINE=InnoDB;

-- Note: Les FK cross-shard sont impossibles
-- → Dénormalisation ou vérification applicative
```

### Configuration ProxySQL pour sharding

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- PROXYSQL : CONFIGURATION SHARDING
-- ═══════════════════════════════════════════════════════════════════════════

-- Connexion à l'admin ProxySQL
-- mysql -u admin -p -h 127.0.0.1 -P 6032

-- Définir les hostgroups (un par shard)
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES
    (10, 'shard1-primary.db.internal', 3306, 1000),
    (11, 'shard1-replica.db.internal', 3306, 1000),
    (20, 'shard2-primary.db.internal', 3306, 1000),
    (21, 'shard2-replica.db.internal', 3306, 1000),
    (30, 'shard3-primary.db.internal', 3306, 1000),
    (31, 'shard3-replica.db.internal', 3306, 1000);

-- Règles de routing basées sur user_id
-- Utilise une regex pour extraire user_id et router vers le bon shard

-- Shard 1 : user_id 1-1000000
INSERT INTO mysql_query_rules (
    rule_id, active, match_pattern, destination_hostgroup, apply
) VALUES (
    100, 1, 
    'WHERE.*user_id\s*=\s*([1-9][0-9]{0,5}|1000000)\b',
    10, 1
);

-- Shard 2 : user_id 1000001-2000000
INSERT INTO mysql_query_rules (
    rule_id, active, match_pattern, destination_hostgroup, apply
) VALUES (
    200, 1,
    'WHERE.*user_id\s*=\s*(100000[1-9]|10000[1-9][0-9]|1000[1-9][0-9]{2}|100[1-9][0-9]{3}|10[1-9][0-9]{4}|1[1-9][0-9]{5}|2000000)\b',
    20, 1
);

-- Shard 3 : user_id 2000001-3000000
INSERT INTO mysql_query_rules (
    rule_id, active, match_pattern, destination_hostgroup, apply
) VALUES (
    300, 1,
    'WHERE.*user_id\s*=\s*(200000[1-9]|20000[1-9][0-9]|2000[1-9][0-9]{2}|200[1-9][0-9]{3}|20[1-9][0-9]{4}|2[1-9][0-9]{5}|3000000)\b',
    30, 1
);

-- Read/Write split par shard
INSERT INTO mysql_query_rules (
    rule_id, active, match_pattern, destination_hostgroup, apply
) VALUES
    (101, 1, '^SELECT.*FOR UPDATE', 10, 1),  -- Shard 1 writes
    (102, 1, '^SELECT', 11, 1);              -- Shard 1 reads

-- Appliquer les changements
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;
```

---

## Comparaison des stratégies

### Comparaison des Stratégies de Scaling

#### Tableau comparatif

| Stratégie | Reads | Writes | Complexité | Coût | HA | Consistency |
|-----------|:-----:|:------:|:----------:|:----:|:--:|:-----------:|
| **Vertical** | ⭐⭐ | ⭐⭐ | ⭐ | $$$$ | ❌ | ✅ |
| **Read Replicas** | ⭐⭐⭐⭐ | ⭐ | ⭐⭐ | $$ | ✅ | ⚠️ * |
| **Galera Cluster** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | $$$ | ✅✅ | ✅ |
| **Sharding** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | $$ | ✅ | ✅ |
| **Sharding + Galera** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | $$$ | ✅✅ | ✅ |

> \* Eventual consistency avec async replication

---

#### Légende

##### Performance (⭐)
| Symbole | Signification |
|:-------:|---------------|
| ⭐ | Limité |
| ⭐⭐ | Basique |
| ⭐⭐⭐ | Bon |
| ⭐⭐⭐⭐ | Excellent |
| ⭐⭐⭐⭐⭐ | Illimité |

##### Coût ($)
| Symbole | Signification |
|:-------:|---------------|
| $ | Économique |
| $$ | Modéré |
| $$$ | Élevé |
| $$$$ | Très élevé |

---

#### Recommandations par Use Case

| Use Case | Stratégie recommandée |
|----------|----------------------|
| 🚀 **Startup / MVP** | Vertical *(simple, rapide)* |
| 📊 **SaaS read-heavy** (80%+ reads) | Read Replicas *(3-5 replicas)* |
| 🛒 **E-commerce transactionnel** | Galera Cluster *(3-5 nodes)* |
| 📱 **Social media / Big data** | Sharding *(par user_id)* |
| 💰 **Finance / High write volume** | Sharding + Galera par shard |
| 🎮 **Gaming global** | Sharding géographique + Galera |

---

## Scaling automatique

### Auto-scaling des replicas

```yaml
# kubernetes-mariadb-autoscaling.yaml
# HPA pour pods read replicas MariaDB

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mariadb-replicas-hpa
  namespace: database
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: mariadb-replicas
  
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
    # Scale sur CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Scale sur connexions (métrique custom)
    - type: Pods
      pods:
        metric:
          name: mysql_connections_active
        target:
          type: AverageValue
          averageValue: 100
    
    # Scale sur lag de réplication
    - type: Pods
      pods:
        metric:
          name: mysql_slave_lag_seconds
        target:
          type: AverageValue
          averageValue: 5  # Scale si lag > 5s
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

```bash
#!/bin/bash
# auto-scale-replicas.sh
# Script d'auto-scaling pour environnements non-Kubernetes

METRICS_HOST="prometheus.internal:9090"
MIN_REPLICAS=2
MAX_REPLICAS=10
SCALE_UP_THRESHOLD=70    # CPU %
SCALE_DOWN_THRESHOLD=30  # CPU %

# Récupérer le nombre actuel de replicas
current_replicas=$(terraform output -raw replica_count)

# Récupérer la charge CPU moyenne
avg_cpu=$(curl -s "$METRICS_HOST/api/v1/query?query=avg(mysql_cpu_usage_percent)" | jq -r '.data.result[0].value[1]')

echo "Current replicas: $current_replicas, Avg CPU: $avg_cpu%"

if (( $(echo "$avg_cpu > $SCALE_UP_THRESHOLD" | bc -l) )); then
    if [ "$current_replicas" -lt "$MAX_REPLICAS" ]; then
        new_count=$((current_replicas + 1))
        echo "Scaling UP to $new_count replicas"
        terraform apply -var="replica_count=$new_count" -auto-approve
    fi
elif (( $(echo "$avg_cpu < $SCALE_DOWN_THRESHOLD" | bc -l) )); then
    if [ "$current_replicas" -gt "$MIN_REPLICAS" ]; then
        new_count=$((current_replicas - 1))
        echo "Scaling DOWN to $new_count replicas"
        terraform apply -var="replica_count=$new_count" -auto-approve
    fi
else
    echo "No scaling needed"
fi
```

---

## Coûts et optimisation

### Analyse coût/performance

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ANALYSE COÛT / PERFORMANCE                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Scénario : 10K QPS, 80% reads, 20% writes, 500 GB données                 │
│                                                                            │
│  Option 1 : SCALE VERTICAL                                                 │
│  ══════════════════════════                                                │
│  • 1 × r6g.16xlarge (64 CPU, 512 GB) + 1 TB gp3                            │
│  • Coût : ~$5,000/mois                                                     │
│  • Capacité : ~15K QPS max                                                 │
│  • HA : Besoin d'un replica → $10,000/mois                                 │
│                                                                            │
│  Option 2 : SCALE HORIZONTAL (Read Replicas)                               │
│  ════════════════════════════════════════════                              │
│  • 1 × r6g.4xlarge Primary (writes)                                        │
│  • 3 × r6g.2xlarge Replicas (reads)                                        │
│  • Coût : $1,000 + 3×$500 = $2,500/mois                                    │
│  • Capacité : 2K writes + 24K reads = OK                                   │
│  • HA : Native (failover vers replica)                                     │
│                                                                            │
│  Option 3 : GALERA CLUSTER                                                 │
│  ══════════════════════════                                                │
│  • 3 × r6g.4xlarge (Galera multi-master)                                   │
│  • Coût : 3×$1,000 = $3,000/mois                                           │
│  • Capacité : ~8K QPS distribué                                            │
│  • HA : Native (n'importe quel node peut être primary)                     │
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  Comparaison (10K QPS)                                             │    │
│  │                                                                    │    │
│  │  Option        Coût/mois  Capacité   HA    Complexité              │    │
│  │  ──────────────────────────────────────────────────────            │    │
│  │  Vertical HA    $10,000    15K QPS   ⚠️     ⭐                     │    │
│  │  Replicas       $2,500     26K QPS   ✅     ⭐⭐                   │    │
│  │  Galera         $3,000     8K QPS    ✅✅    ⭐⭐⭐                │    │
│  │                                                                    │    │
│  │  Winner pour ce scénario : Read Replicas ($2,500)                  │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                            │
│  Conseil : Commencer par vertical, migrer vers horizontal à 70% capacité   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Étude de cas : Migration de scaling

```
┌────────────────────────────────────────────────────────────────────────────┐
│               ÉTUDE DE CAS : ÉVOLUTION D'UNE STARTUP                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Phase 1 : Lancement (0-10K users)                                         │
│  ══════════════════════════════════                                        │
│                                                                            │
│  ┌─────────────────┐                                                       │
│  │  Single Node    │  r6g.xlarge (4 CPU, 32 GB)                            │
│  │  MariaDB        │  $250/mois                                            │
│  │                 │  Capacité : 500 QPS                                   │
│  └─────────────────┘                                                       │
│                                                                            │
│  Phase 2 : Croissance (10K-100K users)                                     │
│  ═════════════════════════════════════                                     │
│                                                                            │
│  ┌─────────────────┐                                                       │
│  │  Scaled Up      │  r6g.4xlarge (16 CPU, 128 GB)                         │
│  │  MariaDB        │  $1,000/mois                                          │
│  │                 │  Capacité : 3K QPS                                    │
│  └─────────────────┘                                                       │
│                                                                            │
│  Phase 3 : Traction (100K-500K users)                                      │
│  ═════════════════════════════════════                                     │
│                                                                            │
│  ┌─────────────────┐    ┌─────────────────┐                                │
│  │    Primary      │───►│    Replica      │                                │
│  │  r6g.4xlarge    │    │  r6g.2xlarge    │                                │
│  │  (Writes)       │    │  (Reads)        │                                │
│  └─────────────────┘    └─────────────────┘                                │
│                                                                            │
│  $1,500/mois, Capacité : 5K QPS, HA: ✅                                    │
│                                                                            │
│  Phase 4 : Scale (500K-2M users)                                           │
│  ═══════════════════════════════                                           │
│                                                                            │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │    Primary      │───►│   Replica 1     │    │   Replica 2     │         │
│  │  r6g.4xlarge    │    │  r6g.2xlarge    │    │  r6g.2xlarge    │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                   │                    │                   │
│                                   └──────┬─────────────┘                   │
│                                          │                                 │
│                              ┌───────────▼───────────┐                     │
│                              │      MaxScale         │                     │
│                              │   (R/W Split + LB)    │                     │
│                              └───────────────────────┘                     │
│                                                                            │
│  $2,500/mois, Capacité : 15K QPS, HA: ✅✅                                 │
│                                                                            │
│  Phase 5 : Explosion (2M+ users)                                           │
│  ════════════════════════════════                                          │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         ProxySQL (Sharding)                         │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│       ┌──────────────────────────┼──────────────────────────┐              │
│       │                          │                          │              │
│       ▼                          ▼                          ▼              │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐       │
│  │  Shard 1    │           │  Shard 2    │           │  Shard 3    │       │
│  │  Galera 3   │           │  Galera 3   │           │  Galera 3   │       │
│  │  (US users) │           │  (EU users) │           │  (APAC)     │       │
│  └─────────────┘           └─────────────┘           └─────────────┘       │
│                                                                            │
│  $12,000/mois, Capacité : 100K+ QPS, HA: ✅✅✅                            │
│                                                                            │
│  Leçons :                                                                  │
│  ════════                                                                  │
│  • Commencer simple (vertical)                                             │
│  • Ajouter des replicas avant le scaling complexe                          │
│  • Sharding seulement quand nécessaire (complexité ++)                     │
│  • Chaque transition = refactoring applicatif                              │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Points clés à retenir

- **Scaling vertical** est simple mais limité — commencer par là, migrer quand on atteint 70% capacité
- **Read replicas** sont la première étape du scaling horizontal — efficace pour les workloads read-heavy (80%+ reads)
- **Galera Cluster** permet le scaling des écritures avec consistance forte — limité à ~5-7 nœuds pour performance optimale
- **Sharding** offre un scaling quasi-illimité mais au prix d'une complexité significative — réserver pour > 1TB ou > 50K QPS
- **MaxScale/ProxySQL** sont essentiels pour le routing intelligent (R/W split, load balancing, sharding)
- **Auto-scaling** permet d'ajuster dynamiquement la capacité selon la charge
- **Le coût par QPS** diminue avec le scaling horizontal — mais la complexité opérationnelle augmente
- 🆕 **MariaDB 11.8** avec `optimizer_disk_read_ratio` optimise automatiquement pour le stockage NVMe

---

## 🔗 Ressources et références

- [📖 MariaDB Replication](https://mariadb.com/kb/en/replication/)
- [📖 Galera Cluster Documentation](https://galeracluster.com/library/documentation/)
- [📖 MaxScale Readwritesplit](https://mariadb.com/kb/en/maxscale-readwritesplit/)
- [📖 ProxySQL Configuration](https://proxysql.com/documentation/)
- [📖 Database Sharding Patterns](https://www.citusdata.com/blog/2018/01/10/sharding-in-plain-english/)
- [📖 Scaling MySQL - O'Reilly](https://www.oreilly.com/library/view/high-performance-mysql/9781449332471/)

---

## ➡️ Section suivante

[20.8 Architectures Event-Driven](./08-architectures-event-driven.md) : Découvrez comment intégrer MariaDB dans des architectures événementielles avec CDC, Kafka et Debezium.

⏭️ [MariaDB dans les architectures Event-Driven](/20-cas-usage-architectures/08-architectures-event-driven.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.1 Configuration OLTP (High concurrency, low latency)

> **Type** : Configuration de référence  
> **Cas d'usage** : OLTP - Online Transaction Processing  
> **Caractéristiques** : Haute concurrence, faible latence, transactions courtes  
> **Public** : DBA, Administrateurs système, DevOps

---

## 🎯 Profil du cas d'usage OLTP

### Caractéristiques d'une charge OLTP

**Online Transaction Processing (OLTP)** désigne les systèmes transactionnels en ligne gérant de nombreuses opérations courtes et concurrentes :

#### Patterns d'accès typiques
- ✅ **Transactions courtes** : < 100ms en moyenne
- ✅ **Haute concurrence** : 100-1000+ connexions simultanées
- ✅ **Lectures/écritures équilibrées** : 60/40 à 40/60 selon l'application
- ✅ **Opérations ciblées** : Requêtes sur index, peu de scans complets
- ✅ **Faible latence critique** : Temps de réponse < 10ms souhaité
- ✅ **Pics de charge** : Variabilité importante (heures de pointe)

#### Exemples d'applications OLTP
- 🛒 **E-commerce** : Panier, commandes, paiements, inventory
- 💳 **Banking** : Transactions bancaires, virements, soldes
- 🎫 **Booking** : Réservations (vols, hôtels, événements)
- 📱 **SaaS Applications** : CRM, ERP, applications métier
- 🎮 **Gaming** : Gestion comptes joueurs, inventaires, scores
- 📧 **Messaging** : Emails, messagerie instantanée

### Métriques clés à optimiser

| Métrique | Objectif OLTP | Importance |
|----------|---------------|------------|
| **Latence (P95)** | < 10ms | ⭐⭐⭐⭐⭐ |
| **Throughput (TPS)** | Max (5000+ TPS) | ⭐⭐⭐⭐⭐ |
| **Connexions concurrentes** | 200-1000+ | ⭐⭐⭐⭐ |
| **CPU Usage** | 60-80% optimal | ⭐⭐⭐⭐ |
| **Buffer Pool Hit Rate** | > 99% | ⭐⭐⭐⭐⭐ |
| **IOPS** | Max disponible | ⭐⭐⭐⭐ |
| **Disk Latency** | < 5ms (SSD/NVMe) | ⭐⭐⭐⭐ |

### Contraintes typiques

- ⚡ **Latence** : Chaque milliseconde compte
- 🔒 **ACID strict** : Durabilité des transactions obligatoire
- 📊 **Disponibilité** : 99.9%+ (downtime très coûteux)
- 🔄 **Réplication** : Souvent master-slave pour lecture scaling
- 💾 **SSD/NVMe requis** : HDD inadaptés pour OLTP moderne

---

## 📝 Configuration my.cnf complète - OLTP

### Fichier de configuration optimisé

Voici un template `my.cnf` optimisé pour **serveur OLTP avec 32GB RAM, 8 CPU cores, SSD**.

```ini
# ============================================================================
# MARIADB 11.8 LTS - CONFIGURATION OLTP
# ============================================================================
# Cas d'usage : OLTP - Haute concurrence, faible latence
# Matériel cible : 32GB RAM, 8 CPU cores, SSD (5000+ IOPS)
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
bind-address                = 0.0.0.0  # Ajuster selon environnement

# ----------------------------------------------------------------------------
# CHARSET ET COLLATION (🆕 MariaDB 11.8 - utf8mb4 par défaut)
# ----------------------------------------------------------------------------
character-set-server        = utf8mb4
collation-server            = utf8mb4_unicode_ci  # Ou uca1400_ai_ci pour UCA 14.0.0

# Support complet Unicode, emojis, tous caractères internationaux
# UCA 14.0.0 offre un meilleur tri multilingue

# ----------------------------------------------------------------------------
# MOTEUR DE STOCKAGE
# ----------------------------------------------------------------------------
default-storage-engine      = InnoDB

# InnoDB est le seul choix pour OLTP moderne
# Support ACID, row-level locking, MVCC, transactions

# ----------------------------------------------------------------------------
# CONNEXIONS ET THREADS
# ----------------------------------------------------------------------------
max_connections             = 500
# Nombre max de connexions simultanées
# Formule : max_connections = (RAM - Buffer Pool - OS) / thread_memory
# Pour OLTP : prévoir pics de charge (x1.5 à x2 la moyenne)

max_connect_errors          = 100000
# Éviter blocage après tentatives échouées répétées

thread_cache_size           = 128
# Cache de threads pour réutilisation
# Formule : thread_cache_size = 8 + (max_connections / 100)
# Réduit overhead création/destruction threads

thread_handling             = pool-of-threads
# 🔥 CRITIQUE OLTP : Thread pool plutôt que one-thread-per-connection
# Meilleure scalabilité avec haute concurrence

thread_pool_size            = 8
# Nombre de thread groups = nombre de CPU cores
# Optimise parallélisme et affinité CPU

thread_pool_max_threads     = 1000
# Maximum threads dans le pool
# Évite épuisement threads lors de pics

thread_pool_stall_limit     = 500
# Milliseconds avant création nouveau thread si stall
# 500ms bon compromis OLTP

# ----------------------------------------------------------------------------
# TABLES ET CACHE
# ----------------------------------------------------------------------------
table_open_cache            = 4000
# Cache des descripteurs de fichiers de tables
# Formule : max_connections × nb_tables_par_query_moyen
# OLTP : beaucoup de tables différentes accédées

table_definition_cache      = 2000
# Cache des définitions de tables (.frm)
# Devrait être >= nombre de tables dans la base

open_files_limit            = 10000
# Limite OS des fichiers ouverts
# Doit être > table_open_cache

# ----------------------------------------------------------------------------
# MÉMOIRE GLOBALE - INNODB BUFFER POOL (🔥 PARAMÈTRE LE PLUS CRITIQUE)
# ----------------------------------------------------------------------------
innodb_buffer_pool_size     = 20G
# 🔥 PARAMÈTRE #1 : Cache des données et index InnoDB
# OLTP : 60-70% de la RAM totale
# 32GB RAM → 20GB buffer pool (62.5%)
# BUT : Maximiser données en mémoire, minimiser I/O disque

innodb_buffer_pool_instances = 8
# Nombre d'instances du buffer pool (parallélisme)
# Règle : 1 instance par GB si >= 8GB, max = nb CPU cores
# 20GB / 8 = 2.5GB par instance → bon équilibre

innodb_buffer_pool_chunk_size = 128M
# Taille des chunks pour resize dynamique
# buffer_pool_size doit être multiple de (chunk_size × instances)
# 20GB = 160 chunks × 128M ✓

# Réchauffement du buffer pool au démarrage
innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1
# Sauvegarde/restaure liste des pages en mémoire
# Accélère redémarrage en rechargeant pages fréquentes

# ----------------------------------------------------------------------------
# MÉMOIRE PAR CONNEXION (⚠️ ATTENTION : × max_connections)
# ----------------------------------------------------------------------------
# Ces buffers sont alloués PAR THREAD actif
# Mémoire totale = max_connections × (read_buffer + sort_buffer + join_buffer + ...)

read_buffer_size            = 256K
# Buffer pour scan séquentiel de table
# OLTP : petite valeur (requêtes ciblées sur index)
# 256K-512K suffisant

read_rnd_buffer_size        = 512K
# Buffer pour lecture après tri
# OLTP : requêtes généralement simples, 512K OK

sort_buffer_size            = 512K
# Buffer pour opérations ORDER BY, GROUP BY
# OLTP : tris généralement petits
# ⚠️ Alloué par query nécessitant un tri !

join_buffer_size            = 256K
# Buffer pour jointures sans index
# OLTP : jointures doivent utiliser index (si non, revoir schema)
# Valeur faible acceptable

# Calcul mémoire par connexion :
# 256K + 512K + 512K + 256K = ~1.5MB par thread
# 500 connexions × 1.5MB = 750MB → acceptable

# ----------------------------------------------------------------------------
# LOGS INNODB - REDO LOG (Durabilité vs Performance)
# ----------------------------------------------------------------------------
innodb_log_file_size        = 1G
# Taille de chaque fichier redo log
# OLTP : 1-2GB par fichier (balance durabilité/performance)
# Plus grand = moins de checkpoints = meilleure perf écriture
# Mais recovery plus long en cas de crash

innodb_log_files_in_group   = 2
# Nombre de fichiers redo log (circular)
# Total redo log = 2 × 1G = 2GB
# Doit contenir au moins 1 heure d'activité

innodb_log_buffer_size      = 64M
# Buffer en mémoire avant flush vers fichiers redo log
# OLTP haute concurrence : 64-128MB

innodb_flush_log_at_trx_commit = 1
# 🔥 CRITIQUE DURABILITÉ :
# 0 = flush toutes les secondes (DANGEREUX, perte possible)
# 1 = flush à chaque commit (ACID strict, recommandé OLTP)
# 2 = flush à chaque commit mais sync OS chaque seconde (compromis)
# OLTP production : TOUJOURS = 1

# ----------------------------------------------------------------------------
# I/O ET DISQUES (🆕 Optimisations SSD MariaDB 11.8)
# ----------------------------------------------------------------------------
innodb_io_capacity          = 4000
# IOPS baseline que le serveur peut utiliser
# HDD 7200rpm : 100-200
# SSD SATA : 2000-5000
# NVMe : 10000-50000+
# 🔥 Mesurer avec : fio ou sysbench fileio

innodb_io_capacity_max      = 8000
# IOPS max en cas de flush intensif (background tasks)
# Généralement 2× innodb_io_capacity

innodb_flush_method         = O_DIRECT
# 🔥 CRITIQUE PRODUCTION :
# O_DIRECT = bypass filesystem cache (évite double buffering)
# Recommandé pour production avec InnoDB buffer pool bien dimensionné

innodb_flush_neighbors      = 0
# 🆕 Optimisation SSD :
# 0 = ne pas flusher pages voisines (optimal SSD/NVMe)
# 1 = flush pages adjacentes (utile HDD, inutile SSD)
# SSD : accès aléatoire rapide, flush neighbors inutile

# 🆕 MariaDB 11.8 : Construction d'index optimisée
innodb_alter_copy_bulk      = ON
# Améliore performances ALTER TABLE ADD INDEX
# Utilise bulk load pour construire index plus rapidement

# Lecture anticipée (read-ahead)
innodb_read_ahead_threshold = 56
# Nombre de pages séquentielles avant déclencher read-ahead
# OLTP : accès aléatoires, read-ahead peu utile
# 56 = valeur par défaut, OK pour OLTP

# ----------------------------------------------------------------------------
# UNDO LOG
# ----------------------------------------------------------------------------
innodb_undo_tablespaces     = 2
# Nombre de tablespaces undo séparés
# Permet truncate/purge plus efficace

innodb_undo_log_truncate    = ON
# Tronque automatiquement undo logs trop gros

innodb_max_undo_log_size    = 1G
# Taille max avant truncate

# ----------------------------------------------------------------------------
# DURABILITÉ ET BINARY LOGS
# ----------------------------------------------------------------------------
sync_binlog                 = 1
# 🔥 CRITIQUE DURABILITÉ :
# 1 = sync binlog à chaque transaction (ACID strict)
# 0 ou N = sync tous les N commits (plus rapide, risque perte)
# OLTP production : = 1 pour cohérence réplication

log_bin                     = /var/log/mysql/mysql-bin
# Activation binary logs (requis pour réplication/PITR)

binlog_format               = ROW
# ROW : plus sûr, meilleur pour réplication
# MIXED : auto-sélection
# STATEMENT : legacy, éviter
# OLTP : ROW recommandé

binlog_row_image            = MINIMAL
# MINIMAL : log seulement colonnes modifiées (économie espace)
# FULL : toutes les colonnes (défaut)
# OLTP avec réplication : MINIMAL si schema stable

expire_logs_days            = 7
# Rétention binary logs (7 jours = 1 semaine)
# Ajuster selon stratégie backup/PITR

max_binlog_size             = 512M
# Taille max d'un fichier binlog avant rotation

# ----------------------------------------------------------------------------
# RÉPLICATION (si applicable)
# ----------------------------------------------------------------------------
server-id                   = 1
# ID unique du serveur (requis pour réplication)
# Master = 1, Slaves = 2, 3, 4...

log_slave_updates           = ON
# Slave log ses propres mises à jour (cascade replication)

gtid_domain_id              = 0
gtid_strict_mode            = ON
# GTID : Global Transaction ID (facilite failover)
# strict_mode = ON : sécurité maximale

relay_log                   = /var/log/mysql/relay-bin
relay_log_recovery          = ON
# Auto-récupération relay logs en cas de crash slave

# ----------------------------------------------------------------------------
# SÉCURITÉ (🆕 MariaDB 11.8)
# ----------------------------------------------------------------------------
# TLS activé par défaut en 11.8
# Si certificats disponibles, décommenter :
# ssl_cert = /etc/mysql/ssl/server-cert.pem
# ssl_key = /etc/mysql/ssl/server-key.pem
# ssl_ca = /etc/mysql/ssl/ca-cert.pem

# Forcer TLS pour connexions distantes
# require_secure_transport = ON

# Plugin d'authentification moderne
# default_authentication_plugin = ed25519

# ----------------------------------------------------------------------------
# QUERY CACHE (⚠️ DÉPRÉCIÉ - NE PAS UTILISER)
# ----------------------------------------------------------------------------
# Query cache est DÉPRÉCIÉ et désactivé par défaut
# Performance dégradée en haute concurrence
# Utiliser cache applicatif (Redis, Memcached) à la place

query_cache_type            = 0
query_cache_size            = 0

# ----------------------------------------------------------------------------
# TEMPORAIRES ET HEAP
# ----------------------------------------------------------------------------
tmp_table_size              = 64M
max_heap_table_size         = 64M
# Taille max tables temporaires en mémoire
# OLTP : requêtes simples, 64M suffisant
# Si dépassé → création table temporaire sur disque (slow)

# 🆕 MariaDB 11.8 : Contrôle espace temporaire
max_tmp_space_usage         = 10G
# Limite totale espace temporaire par connexion
# Protège contre requêtes mal optimisées

# ----------------------------------------------------------------------------
# OPTIMIZER ET STATISTIQUES
# ----------------------------------------------------------------------------
# 🆕 MariaDB 11.8 : Cost optimizer amélioré pour SSD
# Pas de paramètre spécial, automatiquement optimisé

optimizer_search_depth      = 62
# Profondeur recherche plans d'exécution
# OLTP : requêtes simples, défaut OK

optimizer_switch             = 'mrr=on,mrr_cost_based=on,index_condition_pushdown=on'
# Optimisations modernes activées
# MRR (Multi-Range Read) utile pour OLTP

# Statistiques persistantes (meilleur optimizer)
innodb_stats_persistent     = ON
innodb_stats_auto_recalc    = ON
innodb_stats_persistent_sample_pages = 20

# ----------------------------------------------------------------------------
# MONITORING ET PERFORMANCE SCHEMA
# ----------------------------------------------------------------------------
performance_schema          = ON
# Active Performance Schema (monitoring détaillé)
# Overhead ~5-10% mais indispensable production

# Instruments activés par défaut
# Ajuster si besoin via :
# UPDATE performance_schema.setup_instruments SET ENABLED='YES' WHERE NAME LIKE '%wait%';

# ----------------------------------------------------------------------------
# LOGS (Debugging et monitoring)
# ----------------------------------------------------------------------------
log_error                   = /var/log/mysql/error.log

# Slow query log (requêtes lentes)
slow_query_log              = ON
slow_query_log_file         = /var/log/mysql/slow-query.log
long_query_time             = 0.5
# OLTP : 500ms est déjà lent, investiguer
log_slow_verbosity          = query_plan,explain
# Log le plan d'exécution pour analyse

log_queries_not_using_indexes = ON
# Log requêtes sans index (code smell OLTP)

# General log (⚠️ NE PAS ACTIVER EN PRODUCTION - OVERHEAD ÉNORME)
general_log                 = OFF
# Seulement pour debug ponctuel

# ----------------------------------------------------------------------------
# TIMEOUTS
# ----------------------------------------------------------------------------
wait_timeout                = 600
# Timeout connexion inactive (10 min)
# OLTP : connections pools maintiennent connexions actives

interactive_timeout         = 600
# Timeout session interactive

connect_timeout             = 10
# Timeout connexion initiale

net_read_timeout            = 30
net_write_timeout           = 60

# ----------------------------------------------------------------------------
# AUTRES PARAMÈTRES
# ----------------------------------------------------------------------------
max_allowed_packet          = 64M
# Taille max paquet réseau
# OLTP : requêtes généralement petites, 64M largement suffisant

lower_case_table_names      = 0
# 0 = case-sensitive (Linux)
# 1 = case-insensitive (Windows)
# Garder 0 sur Linux pour éviter problèmes portabilité

sql_mode                    = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'
# Mode strict recommandé
# Évite insertions silencieuses de données invalides

# ----------------------------------------------------------------------------
# INNODB AVANCÉ
# ----------------------------------------------------------------------------
innodb_adaptive_hash_index  = ON
# Index hash adaptatif (auto-optimisation)
# OLTP : très bénéfique pour patterns d'accès répétitifs

innodb_change_buffering     = all
# Buffer changements index secondaires
# all = inserts, deletes, purges, changes
# OLTP : utile pour écritures intensives

innodb_old_blocks_time      = 1000
# Millisecondes avant promouvoir page vers young list
# Anti-scan optimization (évite pollution buffer pool par scans)

innodb_print_all_deadlocks  = ON
# Log tous les deadlocks dans error log
# OLTP : utile pour debug contention

# Compression (optionnel)
# innodb_compression_level = 6
# innodb_compression_algorithm = zlib
# OLTP : compression généralement pas nécessaire (overhead CPU)

# ----------------------------------------------------------------------------
[mysqldump]
quick
quote-names
max_allowed_packet          = 64M

# ----------------------------------------------------------------------------
[mysql]
default-character-set       = utf8mb4

# ----------------------------------------------------------------------------
# FIN DE CONFIGURATION
# ============================================================================
```

---

## 🔍 Explication des paramètres clés

### 1. InnoDB Buffer Pool (Mémoire - CRITIQUE)

```ini
innodb_buffer_pool_size = 20G
innodb_buffer_pool_instances = 8
```

**Pourquoi c'est le paramètre #1 :**
- Cache **TOUTES** les données et index InnoDB en mémoire
- Évite les I/O disque (1000× plus lents que RAM)
- Buffer pool hit rate > 99% = objectif OLTP

**Dimensionnement OLTP :**
```
OLTP : 60-70% RAM totale

Serveur 32GB :
- Buffer Pool : 20GB (62.5%)
- Connexions : 500 × 1.5MB = 750MB
- OS + autres : 3-4GB
- Marge sécurité : 7GB
───────────────────────
Total : 32GB ✓
```

**Instances :**
- 1 instance par GB si >= 8GB
- Maximum = nombre de CPU cores
- 20GB / 8 instances = 2.5GB par instance (optimal)
- Réduit contention mutex en haute concurrence

**Monitoring :**
```sql
-- Taux de hit (objectif > 99%)
SELECT 
    ROUND(100.0 * (1 - (
        (SELECT variable_value FROM information_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_reads') / 
        (SELECT variable_value FROM information_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_read_requests')
    )), 4) AS buffer_pool_hit_rate_pct;

-- Si < 99% : augmenter buffer_pool_size
-- Si > 99.5% et RAM libre : OK
-- Si > 99.9% : peut-être trop gros (RAM gaspillée)
```

### 2. Thread Pool (Concurrence - CRITIQUE OLTP)

```ini
thread_handling = pool-of-threads
thread_pool_size = 8
thread_pool_max_threads = 1000
```

**Pourquoi thread pool pour OLTP :**

| One-Thread-Per-Connection | Thread Pool |
|----------------------------|-------------|
| 1 thread OS par connexion | N thread groups (= CPU cores) |
| Context switching intensif | Moins de context switches |
| CPU thrashing à 1000+ conn | Scalable 10000+ connexions |
| Overhead mémoire élevé | Overhead faible |

**Configuration optimale :**
```ini
thread_pool_size = [nombre_cpu_cores]
# 8 cores → 8 thread groups
# Chaque group gère ~500/8 = 62 connexions

thread_pool_stall_limit = 500
# Si thread bloqué > 500ms → nouveau thread
# Balance latence vs prolifération threads
```

**Impact performance :**
- **Sans thread pool** : Dégradation linéaire après 200-300 connexions
- **Avec thread pool** : Scalable jusqu'à 10000+ connexions
- **Gain OLTP** : 30-50% throughput en haute concurrence

### 3. Durabilité ACID (innodb_flush_log_at_trx_commit)

```ini
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

**Options et compromis :**

| Valeur | Durabilité | Performance | Risque |
|--------|------------|-------------|--------|
| **0** | ❌ Flush 1×/sec | ⚡ Maximum | ⚠️ Perte jusqu'à 1s de transactions |
| **1** | ✅ ACID strict | ⚡⚡ -30% | ✅ Aucune perte |
| **2** | ⚡ OS cache | ⚡⚡⚡ -15% | ⚠️ Perte si crash OS |

**Recommandation OLTP production :**
```ini
# TOUJOURS = 1 en production
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

# = 2 acceptable uniquement si :
# - Réplication synchrone active (Galera)
# - Données non critiques (cache, sessions)
# - Infrastructure HA avec UPS
```

**Impact performance :**
- Perte ~30% throughput entre 0 et 1
- **Mais** : garantie ACID indispensable OLTP
- Compenser avec SSD/NVMe performants

### 4. I/O et SSD (innodb_io_capacity)

```ini
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000
innodb_flush_method = O_DIRECT
innodb_flush_neighbors = 0  # 🆕 Optimisation SSD
```

**Mesurer IOPS réels :**

```bash
# Test avec fio
fio --name=random-write --ioengine=libaio --iodepth=32 \
    --rw=randwrite --bs=16k --direct=1 --size=1G \
    --numjobs=4 --runtime=60 --group_reporting

# Ou avec sysbench
sysbench fileio --file-total-size=4G --file-test-mode=rndrw \
    --max-requests=0 --time=60 --threads=4 run

# Observer IOPS dans résultats
```

**Valeurs typiques :**
```ini
# HDD 7200rpm
innodb_io_capacity = 200
innodb_io_capacity_max = 400

# SSD SATA (SATA III 6Gb/s)
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000

# NVMe Gen3
innodb_io_capacity = 15000
innodb_io_capacity_max = 30000

# NVMe Gen4
innodb_io_capacity = 30000
innodb_io_capacity_max = 60000
```

**🆕 MariaDB 11.8 - Optimisations SSD :**
```ini
innodb_flush_neighbors = 0
# SSD : accès aléatoire rapide
# Flusher pages voisines inutile (HDD legacy)

innodb_alter_copy_bulk = ON
# Construction index plus rapide avec bulk load
```

### 5. Connexions et mémoire par thread

```ini
max_connections = 500
read_buffer_size = 256K
sort_buffer_size = 512K
join_buffer_size = 256K
```

**Calcul mémoire totale :**

```
Mémoire globale :
├─ Buffer Pool : 20GB (fixe)
├─ Redo Log Buffer : 64MB
├─ Key Buffer : 32MB (MyISAM, si utilisé)
└─ Thread Cache : 128 × 256KB ≈ 32MB

Mémoire par connexion active :
├─ read_buffer_size : 256KB
├─ read_rnd_buffer_size : 512KB
├─ sort_buffer_size : 512KB (si ORDER BY)
├─ join_buffer_size : 256KB (si JOIN sans index)
└─ Thread stack : 256KB
    ─────────────────
    Total : ~1.8MB par connexion

Max memory (500 connexions) :
20GB + (500 × 1.8MB) + 128MB ≈ 21GB

+ OS + cache FS : 3-4GB
───────────────────────────
Total : 25GB / 32GB → ✅ OK avec marge
```

**⚠️ Attention :**
```sql
-- Requête mal optimisée avec tri énorme
SELECT * FROM huge_table ORDER BY random_column;
-- Alloue sort_buffer_size (512K)
-- Si insuffisant → table temporaire sur DISQUE (lent !)

-- Si 100 requêtes simultanées avec tri :
-- 100 × 512KB = 50MB supplémentaires
```

**Ajustement dynamique possible :**
```sql
-- Session-level (ne persiste pas)
SET SESSION sort_buffer_size = 2097152;  -- 2MB pour cette connexion
```

### 6. Redo Log et Checkpoints

```ini
innodb_log_file_size = 1G
innodb_log_files_in_group = 2
# Total : 2GB redo log
```

**Dimensionnement :**

```
Règle empirique :
Redo Log Total = 1-2 heures d'activité écriture

Trop petit :
- Checkpoints fréquents
- Flush intensif (ralentit écritures)
- Pics de latence

Trop grand :
- Recovery lent en cas de crash
- Espace disque gaspillé

OLTP typique :
1-2GB total = bon compromis
```

**Monitoring :**
```sql
-- Fréquence des checkpoints
SHOW GLOBAL STATUS LIKE 'Innodb_checkpoint%';

-- Si checkpoints toutes les 5 minutes :
-- → Redo log trop petit, augmenter
-- Objectif : checkpoints toutes les 15-30 minutes
```

**🆕 MariaDB 11.8 :**
```ini
innodb_log_buffer_size = 64M
# Buffer en mémoire avant écriture fichiers
# OLTP haute concurrence : 64-128MB recommandé
```

### 7. Monitoring et Performance Schema

```ini
performance_schema = ON
```

**Overhead vs bénéfices :**
- Overhead : 5-10% CPU/mémoire
- **Indispensable** pour diagnostiquer problèmes production
- Alternative : désactiver et utiliser outils externes (PMM, pt-query-digest)

**Requêtes utiles :**
```sql
-- Top 10 requêtes lentes
SELECT 
    DIGEST_TEXT AS query,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_latency_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) AS total_latency_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Index non utilisés (candidats à suppression)
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NOT NULL
  AND COUNT_STAR = 0
  AND OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema');

-- Contention mutex (hotspots)
SELECT 
    EVENT_NAME,
    COUNT_STAR AS count,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) AS total_wait_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'wait/synch/mutex/%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

## 💻 Recommandations matérielles

### Configuration matérielle optimale OLTP

#### Serveur Entry-Level (PME, startup)
```
CPU    : 4-8 cores (Intel Xeon, AMD EPYC)
RAM    : 16-32 GB DDR4/DDR5
Disque : 500GB SSD SATA (min 3000 IOPS)
Réseau : 1 Gbps
────────────────────────────────────────
Prix : 1000-2000€ (cloud ou on-premise)

Config MariaDB :
- innodb_buffer_pool_size = 10-20G
- thread_pool_size = 4-8
- max_connections = 200-300
- Adapté : 100-500 TPS
```

#### Serveur Mid-Range (Production standard)
```
CPU    : 8-16 cores @ 3.0+ GHz
RAM    : 32-64 GB DDR4 ECC
Disque : 1TB NVMe SSD (10000+ IOPS)
Réseau : 10 Gbps
RAID   : RAID 10 ou cloud EBS io2
────────────────────────────────────────
Prix : 3000-6000€ (ou AWS r6i.2xlarge ~300€/mois)

Config MariaDB :
- innodb_buffer_pool_size = 20-40G
- thread_pool_size = 8-16
- max_connections = 500-1000
- Adapté : 1000-5000 TPS
```

#### Serveur High-End (Enterprise, forte charge)
```
CPU    : 32-64 cores @ 3.5+ GHz (AMD EPYC 7xx3)
RAM    : 128-512 GB DDR4/DDR5 ECC
Disque : 2-4TB NVMe Gen4 (50000+ IOPS)
Réseau : 25-100 Gbps
RAID   : RAID 10 NVMe ou cloud io2 Block Express
────────────────────────────────────────
Prix : 15000-50000€ (ou AWS r6i.8xlarge ~1200€/mois)

Config MariaDB :
- innodb_buffer_pool_size = 80-400G
- thread_pool_size = 32-64
- max_connections = 1000-5000
- Adapté : 10000-50000+ TPS
```

### Disques : SSD/NVMe obligatoire OLTP

| Type | IOPS | Latence | Prix/TB | OLTP ? |
|------|------|---------|---------|--------|
| **HDD 7200rpm** | 100-200 | 10-20ms | 20€ | ❌ NON |
| **SSD SATA** | 3000-5000 | 1-5ms | 100€ | ✅ OK |
| **NVMe Gen3** | 15000-30000 | 0.1-1ms | 150€ | ✅✅ Recommandé |
| **NVMe Gen4** | 50000-100000 | 0.05-0.5ms | 200€ | ✅✅✅ Optimal |

**Cloud equivalents :**
- **AWS** : io2 Block Express (64000 IOPS, 4GB/s)
- **Azure** : Premium SSD v2
- **GCP** : Hyperdisk Extreme

### RAM : Plus = Mieux (cache données)

**Règle d'or OLTP :**
```
RAM >= Working Set Size (données actives)

Idéal : 
Buffer Pool Size >= Taille base de données

Réalité :
Buffer Pool Size >= 80% des données fréquentes (hot data)
```

**Exemple :**
```
Base de données : 100GB
Données actives (80/20) : 20GB
────────────────────────────
RAM minimum : 32GB (20GB buffer pool + OS)
RAM recommandée : 64GB (40GB buffer pool = marge)
RAM optimale : 128GB (buffer pool = base entière)
```

### CPU : Fréquence > Nombre de cores

**OLTP favorise :**
- Haute fréquence (3.0+ GHz) : latence par transaction
- Moins de cores rapides > beaucoup de cores lents

**Exemple :**
- **Mieux** : 8 cores @ 3.5 GHz (Intel Xeon Gold)
- **Moins bien** : 16 cores @ 2.0 GHz

**Thread pool utilise efficacement les cores :**
- `thread_pool_size = nombre de cores`
- Affinité CPU pour réduire context switches

### Réseau : 10Gbps recommandé

```
1 Gbps = 125 MB/s théorique
→ 80-100 MB/s réel
→ OK pour <500 connexions

10 Gbps = 1250 MB/s théorique
→ 800-1000 MB/s réel
→ Requis pour 1000+ connexions ou réplication intensive
```

---

## 📊 Métriques de monitoring OLTP

### Dashboard essentiel (Grafana + Prometheus)

#### 1. Latence et throughput
```
• Queries per second (QPS)
  → Objectif : stable, pas de chutes brutales
  
• Transactions per second (TPS)
  → OLTP : 1000-10000+ TPS typique
  
• Average query latency (P50, P95, P99)
  → P95 < 10ms = excellent OLTP
  → P99 < 50ms = acceptable
  
• Slow queries/sec
  → Tendance à 0, investiguer si >10/sec
```

#### 2. Connexions
```
• Threads connected vs max_connections
  → Si > 80% : risque épuisement
  
• Threads running (actifs)
  → OLTP : généralement < 50% de connected
  → Si > connected : problème (requêtes lentes bloquent)
  
• Aborted connections
  → Doit rester ~0
  → Si élevé : problème réseau ou timeout
```

#### 3. InnoDB Buffer Pool
```sql
-- Taux de hit
SELECT ROUND(100.0 * (1 - (
    innodb_buffer_pool_reads / innodb_buffer_pool_read_requests
)), 4) AS hit_rate_pct
FROM (
    SELECT 
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_reads') AS innodb_buffer_pool_reads,
        (SELECT variable_value FROM information_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_read_requests') AS innodb_buffer_pool_read_requests
) AS stats;

-- Objectif : > 99%
-- Si < 99% : augmenter buffer_pool_size
```

```
• Buffer pool utilization (% rempli)
  → 80-95% = optimal (données chaudes en cache)
  
• Buffer pool dirty pages (% modifiées non écrites)
  → < 10% = normal
  → > 50% = flush trop lent, augmenter io_capacity
  
• Buffer pool pages read/written per second
  → Tendance I/O physique
```

#### 4. I/O et disques
```
• IOPS (read + write)
  → Comparer avec innodb_io_capacity
  → Si saturé : goulot disque
  
• Disk latency (ms)
  → SSD : < 5ms
  → NVMe : < 1ms
  
• Disk queue depth
  → < 10 = bon
  → > 50 = saturation
  
• Redo log writes/sec
  → Corrélé au volume de transactions
```

#### 5. Locks et contentions
```sql
-- Deadlocks
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';
-- Si > 0 : analyser requêtes conflictuelles

-- Row lock waits
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_waits';
-- Élevé = contention, optimiser index ou transactions
```

```
• Lock waits/sec
  → Objectif : proche de 0
  → Si élevé : requêtes concurrentes sur mêmes lignes
  
• Lock wait time (ms)
  → Temps d'attente moyen pour acquérir locks
  → Si > 100ms : problème design transactionnel
```

### Alertes critiques

```yaml
# Exemple Prometheus alert rules
groups:
  - name: mariadb_oltp
    rules:
      # Latence P95 élevée
      - alert: HighQueryLatency
        expr: mysql_global_status_queries_p95_milliseconds > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 query latency > 50ms"
          
      # Buffer pool hit rate faible
      - alert: LowBufferPoolHitRate
        expr: mysql_buffer_pool_hit_rate < 0.99
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Buffer pool hit rate < 99%"
          
      # Connexions près du max
      - alert: HighConnectionUsage
        expr: (mysql_global_status_threads_connected / mysql_global_variables_max_connections) > 0.8
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Connections > 80% of max"
          
      # Trop de slow queries
      - alert: TooManySlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow queries rate > 10/sec"
```

---

## ⚠️ Points de vigilance OLTP

### 1. Ne pas sur-dimensionner max_connections

❌ **Mauvaise pratique :**
```ini
max_connections = 10000
# "On ne sait jamais, mettons large"
```

**Problèmes :**
- Épuisement RAM si toutes actives simultanément
- Thread pool inefficace avec trop de threads
- Context switching excessif

✅ **Bonne pratique :**
```ini
max_connections = 500
# Basé sur :
# - Monitoring : max observé × 1.5
# - Pool applicatif : 10 app servers × 50 conn = 500
# - RAM disponible : (RAM - buffer_pool) / thread_memory
```

### 2. Transactions courtes obligatoires

❌ **Anti-pattern OLTP :**
```sql
START TRANSACTION;
SELECT * FROM orders WHERE user_id = 123 FOR UPDATE;
-- Application fait traitement long (5 secondes)
-- Envoie email, appelle API externe...
UPDATE orders SET status = 'processed' WHERE id = 456;
COMMIT;
```

**Impact :**
- Locks maintenus pendant traitement applicatif
- Autres transactions bloquées
- Timeouts, deadlocks

✅ **Pattern OLTP correct :**
```sql
-- Lire données SANS lock
SELECT * FROM orders WHERE user_id = 123;

-- Traitement applicatif (hors transaction)
-- ...

-- Transaction ultra-courte pour écriture
START TRANSACTION;
UPDATE orders SET status = 'processed' WHERE id = 456 AND version = 1;
-- Vérifier affected_rows = 1 (optimistic locking)
COMMIT;
```

**Règle d'or :**
```
Durée transaction < 100ms
Pas d'I/O externe dans transaction (API, email, fichiers)
Lock seulement au dernier moment
```

### 3. Index obligatoires pour toutes les WHERE clauses

❌ **Killer OLTP :**
```sql
SELECT * FROM users WHERE email = 'john@example.com';
-- Table 10M lignes, PAS D'INDEX sur email
-- → Full table scan, 5-10 secondes
-- × 100 requêtes simultanées = catastrophe
```

✅ **OLTP correct :**
```sql
CREATE INDEX idx_users_email ON users(email);
-- Lookup O(log n), < 1ms
```

**Vérification :**
```sql
-- Activer logging requêtes sans index
SET GLOBAL log_queries_not_using_indexes = ON;

-- Analyser slow query log
-- Toute requête sans index est un BUG en OLTP
```

### 4. Éviter SELECT * (surtout avec BLOB/TEXT)

❌ **Gaspillage réseau/mémoire :**
```sql
SELECT * FROM products WHERE id = 123;
-- Table avec colonnes :
-- - id, name, price (léger)
-- - description TEXT 10KB
-- - image BLOB 500KB
-- → Transfert 510KB pour lire 1 prix !
```

✅ **Sélection ciblée :**
```sql
SELECT id, name, price FROM products WHERE id = 123;
-- Transfert 100 bytes au lieu de 510KB
-- × 1000 requêtes/sec = économie massive
```

### 5. Connection pooling applicatif obligatoire

❌ **Sans pool :**
```python
# À chaque requête HTTP
def handle_request():
    conn = mariadb.connect(host='db', user='app', password='xxx')
    # Overhead connexion : 10-50ms
    cursor = conn.cursor()
    cursor.execute("SELECT ...")
    conn.close()
```

**Problèmes :**
- Overhead connexion TCP + auth : 10-50ms
- Épuise max_connections rapidement
- Performance dégradée

✅ **Avec pool :**
```python
# Pool global
pool = mariadb.ConnectionPool(
    pool_name = "app_pool",
    pool_size = 50,  # Par instance app
    host = 'db',
    user = 'app',
    password = 'xxx'
)

def handle_request():
    conn = pool.get_connection()  # Réutilise connexion, < 1ms
    cursor = conn.cursor()
    cursor.execute("SELECT ...")
    conn.close()  # Retourne au pool, ne ferme pas vraiment
```

**Sizing pool applicatif :**
```
Formule :
pool_size_per_app_instance = max_connections_db / nb_app_instances / 2

Exemple :
max_connections = 500
10 app servers
────────────────
pool_size = 500 / 10 / 2 = 25 connexions par app server
```

### 6. Monitoring proactif requis

❌ **Réactif :**
```
"Le site est lent !"
→ Panique, investigations...
→ Découverte : buffer pool saturé depuis 1 semaine
```

✅ **Proactif :**
```
Grafana dashboard :
- Buffer pool hit rate < 99% depuis 3 jours
- Alert Slack/PagerDuty
- Planification upgrade RAM avant impact utilisateur
```

**Minimum monitoring OLTP :**
- Grafana + Prometheus (ou équivalent)
- Alertes sur métriques critiques
- Review hebdomadaire slow query log
- Capacity planning mensuel

---

## ✅ Points clés à retenir

- 🔥 **Buffer pool = paramètre #1** : 60-70% RAM pour cache données
- ⚡ **Thread pool obligatoire** : Scalabilité haute concurrence
- 💾 **SSD/NVMe requis** : HDD inadaptés OLTP moderne
- 🔒 **ACID strict** : `innodb_flush_log_at_trx_commit = 1` en production
- 🎯 **Transactions courtes** : < 100ms, pas d'I/O externe
- 📊 **Index partout** : Toutes les colonnes WHERE doivent avoir index
- 🔌 **Connection pooling** : Côté application obligatoire
- 📈 **Monitoring proactif** : Grafana + alertes critiques
- 🆕 **MariaDB 11.8** : Thread pool, innodb_alter_copy_bulk, optimizer SSD
- 🧮 **Sizing RAM** : Working set + connexions + OS + marge 20%

---

## 🔗 Ressources complémentaires

### Documentation officielle MariaDB
- [InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [Thread Pool](https://mariadb.com/kb/en/thread-pool-in-mariadb/)
- [Optimization and Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)

### Outils de monitoring
- **Percona Monitoring and Management (PMM)** : Suite complète
- **Prometheus + mysqld_exporter** : Metrics collection
- **Grafana** : Dashboards ([templates officiels](https://grafana.com/grafana/dashboards/7362))
- **pt-query-digest** : Analyse slow query log

### Benchmarking
- **sysbench** : Benchmark OLTP standard
- **mysqlslap** : Tool officiel MariaDB
- **HammerDB** : TPC-C/TPC-H workloads

### Autres annexes
- [D.2 - Configuration OLAP](./02-configuration-olap.md)
- [D.3 - Configuration Mixed Workload](./03-configuration-mixed-workload.md)
- [E - Checklist Performance](/annexes/checklist-performance/README.md)

---

## ➡️ Section suivante

**[D.2 - Configuration OLAP](./02-configuration-olap.md)** : Data warehousing, analytics, requêtes complexes

---

**MariaDB** : Version 11.8 LTS

⏭️ [OLAP (Data warehouse, analytics)](/annexes/configuration-reference/02-configuration-olap.md)

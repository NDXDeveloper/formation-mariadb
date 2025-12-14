🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.4 Configuration I/O et disques

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Sections 15.1-15.3 (Méthodologie, Mémoire, Query Cache)
> - Compréhension de l'architecture InnoDB
> - Connaissances en stockage (SSD, NVMe, RAID)
> - Bases en administration système Linux (iostat, blktrace)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre l'architecture I/O** de MariaDB/InnoDB et les patterns d'accès
- **Différencier les types de stockage** (HDD, SSD SATA, NVMe) et leurs caractéristiques
- **Configurer les paramètres I/O critiques** pour chaque type de stockage
- **Exploiter les nouveautés MariaDB 11.8** pour SSD/NVMe (cost optimizer, alter_copy_bulk)
- **Monitorer les performances I/O** au niveau système et MariaDB
- **Diagnostiquer les bottlenecks I/O** et appliquer les corrections appropriées
- **Optimiser pour différents workloads** (OLTP, OLAP, mixte)
- **Adapter la configuration** lors de migration HDD → SSD

---

## Introduction

La configuration I/O est **le deuxième levier de performance le plus important** après la mémoire. Même avec un buffer pool parfaitement dimensionné, les performances dépendent fondamentalement de la capacité du sous-système de stockage à gérer :

- **Lectures aléatoires** (random reads) : Requêtes SELECT sur données non cachées
- **Écritures aléatoires** (random writes) : INSERT, UPDATE, DELETE
- **Écritures séquentielles** (sequential writes) : Redo logs, binary logs
- **Flush operations** : Dirty pages → disque
- **Checkpoints** : Synchronisation périodique

### L'évolution du stockage change tout

```
┌──────────────────────────────────────────────────────────┐
│        ÉVOLUTION DU STOCKAGE (2000-2024)                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  2000-2010 : ÈRE HDD                                     │
│  • IOPS : 100-200 (rotation 7200 RPM)                    │
│  • Latence : 5-10 ms                                     │
│  • Stratégie : Minimiser seeks, maximiser séquentiel     │
│  • MariaDB tuné pour HDD                                 │
│                                                          │
│  2010-2018 : TRANSITION SSD SATA                         │
│  • IOPS : 50,000-100,000                                 │
│  • Latence : 0.1-0.2 ms                                  │
│  • Stratégie : Seeks acceptables, mais pas optimaux      │
│  • MariaDB encore optimisé pour HDD                      │
│                                                          │
│  2018-2024 : ÈRE NVMe                                    │
│  • IOPS : 500,000-1,000,000+                             │
│  • Latence : 0.02-0.05 ms (20-50 µs)                     │
│  • Stratégie : Random access quasi-gratuit               │
│  🆕 MariaDB 11.8 : Premier LTS optimisé pour NVMe        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Impact sur MariaDB 11.8

🆕 **Nouveautés MariaDB 11.8 pour stockage moderne** :

1. **Cost-based optimizer SSD-aware**
   - Détection automatique du type de stockage
   - Coûts I/O ajustés pour SSD/NVMe
   - Plans d'exécution optimisés pour random access rapide

2. **innodb_alter_copy_bulk**
   - Construction d'index jusqu'à 40% plus rapide
   - Optimisé pour I/O parallèles sur SSD
   - Exploitation des IOPS élevés des NVMe

3. **Recommandations de configuration adaptées**
   - `innodb_io_capacity` : valeurs réalistes pour SSD
   - `innodb_flush_method` : optimisations modernes
   - Thread pool ajusté pour haute concurrence I/O

---

## Architecture I/O de MariaDB/InnoDB

### Les différents flux I/O

```
┌───────────────────────────────────────────────────────┐
│           ARCHITECTURE I/O InnoDB                     │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────┐          │
│  │      BUFFER POOL (RAM)                  │          │
│  │  • Pages de données                     │          │
│  │  • Index                                │          │
│  │  • Dirty pages (modifiées)              │          │
│  └──────────┬──────────────────────────────┘          │
│             │                                         │
│             │ (1) READ I/O                            │
│             │     • Buffer pool misses                │
│             │     • Read-ahead                        │
│             │     • Préchargement                     │
│             │                                         │
│             │ (2) WRITE I/O (Background threads)      │
│             │     • Flush dirty pages                 │
│             │     • Merge insert buffer               │
│             │     • Purge                             │
│             ▼                                         │
│  ┌─────────────────────────────────────────┐          │
│  │      REDO LOG BUFFER (RAM)              │          │
│  │  • Modifications récentes               │          │
│  └──────────┬──────────────────────────────┘          │
│             │                                         │
│             │ (3) LOG WRITES (Critical path)          │
│             │     • Commit = flush log                │
│             │     • Séquentiel                        │
│             │     • Latence critique                  │
│             ▼                                         │
│  ┌──────────────────────────────────────────┐         │
│  │           DISQUE                         │         │
│  │                                          │         │
│  │  • Data files (.ibd)                     │         │
│  │  • Redo logs (ib_logfile*)               │         │
│  │  • Binary logs (binlog.*)                │         │
│  │  • Temp files                            │         │
│  └──────────────────────────────────────────┘         │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### Patterns d'I/O par type d'opération

```sql
-- 1. SELECT (lectures)
-- Pattern : Random reads si données pas en cache
SELECT * FROM orders WHERE customer_id = 12345;
-- I/O : Lecture aléatoire pages de données + index

-- 2. INSERT (écritures)
-- Pattern : Écriture séquentielle log + écriture aléatoire data (background)
INSERT INTO orders (customer_id, amount) VALUES (12345, 99.99);
-- I/O immédiate : Redo log (séquentiel)
-- I/O différée : Page de données (aléatoire, background)

-- 3. UPDATE (écritures + lectures)
-- Pattern : Lecture aléatoire + écriture log + écriture data
UPDATE orders SET status = 'shipped' WHERE id = 98765;
-- I/O : Read page → Modify → Write log → Write page (background)

-- 4. Full table scan (lecture séquentielle)
-- Pattern : Lectures séquentielles massives
SELECT COUNT(*) FROM orders WHERE order_date > '2024-01-01';
-- I/O : Scan séquentiel (peut saturer I/O)
```

### Les 3 types d'I/O critiques

```
┌────────────────────────────────────────────────────┐
│  TYPE I/O         CARACTÉRISTIQUE    CRITICITÉ     │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. REDO LOG      • Séquentiel       CRITIQUE      │
│                   • Synchrone        (commit path) │
│                   • Latence = user   ⚠️            │
│                     latency                        │
│                                                    │
│  2. DATA READS    • Aléatoire        HAUTE         │
│                   • Cache misses     (queries)     │
│                   • Affecte queries                │
│                                                    │
│  3. DATA WRITES   • Aléatoire        MOYENNE       │
│    (background)   • Asynchrone       (background)  │
│                   • Buffer pool      ✓             │
│                     flush                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

💡 **Point critique** : Les écritures dans les **redo logs** sont sur le chemin critique des commits. Leur latence impacte **directement** la latence utilisateur.

---

## Comparaison HDD vs SSD vs NVMe

### Tableau comparatif détaillé

| Critère | HDD (7200 RPM) | SSD SATA | NVMe Gen3 | NVMe Gen4 |
|---------|----------------|----------|-----------|-----------|
| **IOPS Random Read 4K** | 100-200 | 50,000-100,000 | 500,000-600,000 | 800,000-1,000,000 |
| **IOPS Random Write 4K** | 100-200 | 40,000-90,000 | 400,000-500,000 | 700,000-900,000 |
| **Latence lecture** | 5-10 ms | 0.1-0.2 ms | 20-50 µs | 10-30 µs |
| **Bande passante seq** | 150-200 MB/s | 500-600 MB/s | 3,000-3,500 MB/s | 5,000-7,000 MB/s |
| **Parallélisme** | Très limité | Moyen | Élevé | Très élevé |
| **Prix/GB (2024)** | $0.02 | $0.10 | $0.15 | $0.20 |
| **Durabilité (TBW)** | N/A | 200-600 TBW | 600-1,800 TBW | 1,000-3,000 TBW |

### Impact sur les configurations MariaDB

```ini
# ──────────────────────────────────────────────────────
# CONFIGURATION HDD (legacy, 2000-2015)
# ──────────────────────────────────────────────────────
[mariadb-hdd]
# I/O limité : minimiser les accès disque
innodb_buffer_pool_size = 90%_RAM  # Cache maximal
innodb_io_capacity = 200           # IOPS réalistes HDD
innodb_io_capacity_max = 2000      # Burst limité
innodb_flush_method = O_DIRECT     # OK
innodb_flush_neighbors = 1         # Utile (écritures adjacentes groupées)
innodb_read_ahead_threshold = 56   # Read-ahead agressif

# ──────────────────────────────────────────────────────
# CONFIGURATION SSD SATA (transition, 2015-2020)
# ──────────────────────────────────────────────────────
[mariadb-ssd-sata]
# I/O bon : équilibre cache/performance
innodb_buffer_pool_size = 75%_RAM  # Cache important mais pas max
innodb_io_capacity = 2000          # IOPS réalistes SSD SATA
innodb_io_capacity_max = 4000      # Burst correct
innodb_flush_method = O_DIRECT     # Recommandé
innodb_flush_neighbors = 0         # Inutile sur SSD
innodb_read_ahead_threshold = 0    # Read-ahead moins utile

# ──────────────────────────────────────────────────────
# CONFIGURATION NVMe (moderne, 2020+, MariaDB 11.8) 🆕
# ──────────────────────────────────────────────────────
[mariadb-nvme]
# I/O excellent : équilibre RAM/OS cache
innodb_buffer_pool_size = 70%_RAM  # Laisser RAM pour OS cache
innodb_io_capacity = 10000         # IOPS réalistes NVMe
innodb_io_capacity_max = 20000     # Burst élevé
innodb_flush_method = O_DIRECT     # Essentiel
innodb_flush_neighbors = 0         # Contre-productif sur NVMe
innodb_read_ahead_threshold = 0    # Inutile
innodb_alter_copy_bulk = ON        # 🆕 11.8 : DDL rapides

# Cost optimizer détecte automatiquement NVMe 🆕
```

### Évolution des stratégies d'optimisation

```
┌──────────────────────────────────────────────────────┐
│         PHILOSOPHIE D'OPTIMISATION                   │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ÈRE HDD (avant 2015)                                │
│  ────────────────────                                │
│  Objectif : ÉVITER les I/O disque à tout prix        │
│  Stratégie : Cache maximal, reads-ahead agressif     │
│  Limitation : ~200 IOPS totales                      │
│                                                      │
│  ÈRE SSD SATA (2015-2020)                            │
│  ─────────────────────                               │
│  Objectif : ÉQUILIBRER cache et I/O                  │
│  Stratégie : Cache important, I/O acceptables        │
│  Limitation : ~100k IOPS                             │
│                                                      │
│  ÈRE NVMe + MariaDB 11.8 (2020+) 🆕                  │
│  ───────────────────────────────────                 │
│  Objectif : EXPLOITER les I/O ultra-rapides          │
│  Stratégie : Balance RAM/OS, optimizer aware         │
│  Limitation : Quasi-aucune (<1M IOPS)                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## Paramètres I/O critiques : Vue d'ensemble

### Les 8 paramètres essentiels

```sql
-- Vue d'ensemble des paramètres I/O actuels
SELECT 
    '1. innodb_io_capacity' as parameter,
    @@innodb_io_capacity as current_value,
    'Background I/O rate' as description
UNION ALL
SELECT '2. innodb_io_capacity_max',
    @@innodb_io_capacity_max,
    'Max burst I/O rate'
UNION ALL
SELECT '3. innodb_flush_method',
    @@innodb_flush_method,
    'Comment MariaDB écrit sur disque'
UNION ALL
SELECT '4. innodb_flush_neighbors',
    @@innodb_flush_neighbors,
    '0=SSD, 1=HDD (flush pages adjacentes)'
UNION ALL
SELECT '5. innodb_read_io_threads',
    @@innodb_read_io_threads,
    'Threads lecture asynchrone'
UNION ALL
SELECT '6. innodb_write_io_threads',
    @@innodb_write_io_threads,
    'Threads écriture asynchrone'
UNION ALL
SELECT '7. innodb_flush_log_at_trx_commit',
    @@innodb_flush_log_at_trx_commit,
    'Durabilité vs performance'
UNION ALL
SELECT '8. innodb_log_file_size',
    @@innodb_log_file_size,
    'Taille redo log';
```

### Hiérarchie d'impact

```
┌──────────────────────────────────────────────────────┐
│  IMPACT SUR PERFORMANCES (ordre décroissant)         │
├──────────────────────────────────────────────────────┤
│                                                      │
│  🔴 CRITIQUE (impact 10x-100x)                       │
│  ───────────────────────────                         │
│  1. innodb_flush_method                              │
│     • O_DIRECT vs défaut = 2-5x performance          │
│     • Évite double buffering                         │
│                                                      │
│  2. Type de disque (HDD vs SSD vs NVMe)              │
│     • Impact matériel : 10-100x IOPS                 │
│     • Prérequis à toute optimisation                 │
│                                                      │
│  🟡 IMPORTANT (impact 2x-10x)                        │
│  ───────────────────────                             │
│  3. innodb_io_capacity + innodb_io_capacity_max      │
│     • Doit matcher les capacités réelles du disque   │
│     • Mal configuré = sous-utilisation ou saturation │
│                                                      │
│  4. innodb_flush_log_at_trx_commit                   │
│     • Durabilité vs latence des commits              │
│     • Impact direct sur throughput write             │
│                                                      │
│  5. innodb_log_file_size                             │
│     • Trop petit = checkpoints fréquents             │
│     • Trop grand = recovery lent                     │
│                                                      │
│  🟢 MODÉRÉ (impact 1.2x-2x)                          │
│  ────────────────────                                │
│  6. innodb_flush_neighbors                           │
│     • 0 pour SSD (important)                         │
│     • 1 pour HDD (utile)                             │
│                                                      │
│  7. innodb_read_io_threads / innodb_write_io_threads │
│     • 4-16 threads selon workload                    │
│                                                      │
│  8. innodb_read_ahead_threshold                      │
│     • Inutile sur SSD/NVMe                           │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Configurations recommandées par type de stockage

#### Configuration HDD (legacy)

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION HDD 7200 RPM (déconseillé en 2024)
# ─────────────────────────────────────────────────────

# I/O capacity : Conservateur
innodb_io_capacity = 200        # ~100-200 IOPS réalistes
innodb_io_capacity_max = 2000   # Burst limité

# Flush method
innodb_flush_method = O_DIRECT  # OK, évite double buffering

# Flush neighbors : IMPORTANT sur HDD
innodb_flush_neighbors = 1      # Grouper écritures adjacentes
innodb_flush_sync = 1           # Sync lors des checkpoints

# Read-ahead : Utile sur HDD
innodb_read_ahead_threshold = 56  # Lectures anticipées agressives

# Threads I/O
innodb_read_io_threads = 4      # Limité par I/O disque
innodb_write_io_threads = 4

# Log : Équilibre performance/durabilité
innodb_flush_log_at_trx_commit = 1   # Durabilité max
innodb_log_file_size = 512M          # Modéré

# Buffer pool : Maximal pour éviter I/O
innodb_buffer_pool_size = 90%_RAM
```

#### Configuration SSD SATA

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION SSD SATA (acceptable en 2024)
# ─────────────────────────────────────────────────────

# I/O capacity : Réaliste pour SSD SATA
innodb_io_capacity = 2000       # ~2000-4000 IOPS moyennes
innodb_io_capacity_max = 4000   # Burst correct

# Flush method
innodb_flush_method = O_DIRECT  # ESSENTIEL sur SSD

# Flush neighbors : INUTILE sur SSD
innodb_flush_neighbors = 0      # Random access rapide

# Read-ahead : Moins utile
innodb_read_ahead_threshold = 0   # Désactivé

# Threads I/O : Plus élevés
innodb_read_io_threads = 8      # SSD peut paralléliser
innodb_write_io_threads = 8

# Log : Performance améliorée
innodb_flush_log_at_trx_commit = 1   # Ou 2 si acceptable
innodb_log_file_size = 1G            # Plus grand

# Buffer pool : Équilibré
innodb_buffer_pool_size = 75%_RAM
```

#### Configuration NVMe (recommandé 2024+) 🆕

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION NVMe + MariaDB 11.8 (optimal 2024) 🆕
# ─────────────────────────────────────────────────────

# I/O capacity : Élevé pour NVMe
innodb_io_capacity = 10000       # ~10k-15k IOPS moyennes
innodb_io_capacity_max = 20000   # Burst très élevé

# Flush method
innodb_flush_method = O_DIRECT   # CRITIQUE sur NVMe

# Flush neighbors : CONTRE-PRODUCTIF
innodb_flush_neighbors = 0       # Ne jamais activer sur NVMe

# Read-ahead : Inutile
innodb_read_ahead_threshold = 0  # Random access quasi-gratuit

# Threads I/O : Parallélisme élevé
innodb_read_io_threads = 16      # NVMe très parallèle
innodb_write_io_threads = 16

# Log : Performance maximale
innodb_flush_log_at_trx_commit = 1   # Ou 2 selon tolérance
innodb_log_file_size = 2G            # Grand pour NVMe

# Buffer pool : Balance avec OS cache
innodb_buffer_pool_size = 70%_RAM   # Laisser RAM pour OS

# 🆕 Nouveautés MariaDB 11.8
innodb_alter_copy_bulk = ON      # DDL rapides sur NVMe
# Cost optimizer SSD-aware activé automatiquement
```

---

## 🆕 Nouveautés MariaDB 11.8 pour I/O

### 1. Cost-based Optimizer SSD-aware

**Innovation majeure** : L'optimiseur détecte automatiquement le type de stockage et ajuste les coûts I/O.

```sql
-- Avant MariaDB 11.8 (optimizer "aveugle")
-- ─────────────────────────────────────────────────
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
/*
L'optimizer utilise des coûts fixes :
- Random read = 10.0 (calibré pour HDD)
- Sequential read = 1.0
→ Favorise systématiquement le séquentiel

Résultat : Plans sous-optimaux sur SSD
*/

-- MariaDB 11.8 🆕 (optimizer SSD-aware)
-- ─────────────────────────────────────────────────
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
/*
L'optimizer détecte NVMe :
- Random read = 1.2 (ajusté pour NVMe)
- Sequential read = 1.0
→ Random access presque aussi efficace

Résultat : Plans optimaux pour SSD/NVMe
*/
```

**Impact concret** :

```sql
-- Table : orders (10M lignes)
-- Index : idx_customer_id sur customer_id
-- Requête : Récupérer commandes d'un client (50 lignes attendues)

-- Avant 11.8 sur HDD :
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;
-- Plan choisi : INDEX SCAN (utilise l'index) ✓
-- Coût estimé : 500 (50 random reads × 10.0)

-- Avant 11.8 sur NVMe :
-- Plan choisi : TABLE SCAN (ignore l'index!) ✗
-- Coût estimé : 800 (séquentiel paraissait moins cher)
-- Problème : Optimizer calibré pour HDD!

-- Avec 11.8 sur NVMe 🆕 :
-- Plan choisi : INDEX SCAN (utilise l'index) ✓
-- Coût estimé : 60 (50 random reads × 1.2)
-- Optimizer comprend que random access est rapide!
```

**Vérifier la détection** :

```sql
-- MariaDB 11.8 log les informations de stockage au démarrage
SHOW WARNINGS;

-- Vérifier les coûts optimizer actuels
SELECT 
    @@optimizer_disk_read_cost as disk_read_cost,
    @@optimizer_disk_write_cost as disk_write_cost;

-- Sur NVMe détecté, ces valeurs sont ajustées automatiquement
```

### 2. innodb_alter_copy_bulk 🆕

**Problème résolu** : Les ALTER TABLE avec reconstruction étaient lents, même sur SSD rapides.

```sql
-- Activer la fonctionnalité (MariaDB 11.8+)
SET GLOBAL innodb_alter_copy_bulk = ON;

-- Exemple : Ajout d'index sur grande table
CREATE TABLE large_table (
    id INT PRIMARY KEY,
    data VARCHAR(1000),
    created_at TIMESTAMP
) ENGINE=InnoDB;

-- Insérer 10M lignes
INSERT INTO large_table SELECT ... ;  -- 10M rows

-- ALTER TABLE avec reconstruction d'index
ALTER TABLE large_table ADD INDEX idx_created (created_at);

/*
Sans innodb_alter_copy_bulk :
- Temps : ~180 secondes
- Méthode : Construction index ligne par ligne

Avec innodb_alter_copy_bulk = ON 🆕 :
- Temps : ~108 secondes (40% plus rapide!)
- Méthode : Construction bulk optimisée
- Exploitation : I/O parallèles sur SSD
*/
```

**Fonctionnement** :

```
CONSTRUCTION INDEX TRADITIONNELLE :
┌──────────────────────────────────────┐
│  1. Trier données                    │
│  2. Construire index page par page   │
│  3. Écrire chaque page séparément    │
│  → Beaucoup d'I/O petites            │
└──────────────────────────────────────┘

AVEC innodb_alter_copy_bulk 🆕 :
┌──────────────────────────────────────┐
│  1. Trier données                    │
│  2. Construire gros blocs en mémoire │
│  3. Écrire blocs en bulk (parallèle) │
│  → Moins d'I/O, plus grosses, //     │
│  → Exploite IOPS élevés SSD/NVMe     │
└──────────────────────────────────────┘
```

**Quand l'utiliser** :

✅ **Recommandé** :
- Stockage SSD ou NVMe
- Grandes tables (>1M lignes)
- Migrations de schéma
- Ajout d'index multiples
- Rebuild après suppression massive

⚠️ **Déconseillé** :
- Stockage HDD (peu d'impact)
- Tables petites (<10k lignes)
- Serveurs avec RAM limitée

```sql
-- Monitoring du progrès
SHOW PROCESSLIST;
-- Regarder "Stage" : "copy to tmp table" → "building index"

-- Métriques
SELECT 
    COUNT(*) as table_rows,
    ROUND(data_length / 1024 / 1024, 2) as data_mb,
    ROUND(index_length / 1024 / 1024, 2) as index_mb
FROM information_schema.tables
WHERE table_name = 'large_table';
```

### 3. Paramètres recommandés pour NVMe (11.8)

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION OPTIMALE MariaDB 11.8 + NVMe Gen4
# ─────────────────────────────────────────────────────

# I/O capacity ajusté pour NVMe haute performance
innodb_io_capacity = 15000           # 🆕 Recommandation 11.8
innodb_io_capacity_max = 30000       # Burst très élevé

# Flush optimisé
innodb_flush_method = O_DIRECT
innodb_flush_neighbors = 0           # JAMAIS sur NVMe
innodb_flush_sync = 1

# Threads I/O : Haute parallélisation
innodb_read_io_threads = 16          # 🆕 Augmenté pour NVMe
innodb_write_io_threads = 16

# DDL rapides
innodb_alter_copy_bulk = ON          # 🆕 MariaDB 11.8

# Log sizing moderne
innodb_log_file_size = 2G            # 🆕 Plus grand acceptable
innodb_log_files_in_group = 2
innodb_log_buffer_size = 64M

# Checkpointing agressif (NVMe peut suivre)
innodb_adaptive_flushing = ON
innodb_adaptive_flushing_lwm = 10    # 🆕 Plus agressif
innodb_max_dirty_pages_pct = 75      # 🆕 Tolérance plus élevée
innodb_max_dirty_pages_pct_lwm = 50

# Buffer pool équilibré
innodb_buffer_pool_size = 70%_RAM    # 🆕 Laisser RAM pour OS cache

# Read-ahead désactivé
innodb_read_ahead_threshold = 0
innodb_random_read_ahead = 0

# Page cleaner threads
innodb_page_cleaners = 4             # Paralléliser le flushing
```

---

## Monitoring I/O : Système et MariaDB

### 1. Monitoring système (Linux)

#### iostat : L'outil essentiel

```bash
# Installation
sudo apt-get install sysstat  # Debian/Ubuntu
sudo yum install sysstat      # RHEL/CentOS

# Monitoring temps réel
iostat -x 1 10
# -x : Extended stats
# 1  : Intervalle 1 seconde
# 10 : 10 itérations

# Output exemple :
# Device    r/s    w/s    rkB/s    wkB/s  %util  await  svctm
# nvme0n1   2500   1800   40000    28800  65.2   2.1    0.3
#           ↑      ↑      ↑        ↑      ↑      ↑      ↑
#           reads  writes read MB  write  satur  wait   service
#           /sec   /sec   /sec     MB/sec -ation time   time
```

**Interprétation** :

```
┌──────────────────────────────────────────────────────┐
│  MÉTRIQUE      BON         ATTENTION    CRITIQUE     │
├──────────────────────────────────────────────────────┤
│  %util         < 70%       70-90%       > 90%        │
│  await (ms)    < 5         5-20         > 20         │
│  svctm (ms)    < 1         1-5          > 5          │
│                                                      │
│  Spécifique NVMe :                                   │
│  await         < 0.5 ms    0.5-2 ms     > 2 ms       │
│  %util         < 80%       80-95%       > 95%        │
└──────────────────────────────────────────────────────┘
```

#### Autres outils utiles

```bash
# iotop : Top des processus I/O
sudo iotop -o
# -o : Seulement processus actifs

# blktrace : Tracing I/O détaillé
sudo blktrace -d /dev/nvme0n1 -o trace
sudo blkparse trace.blktrace.0

# fio : Benchmarking I/O
fio --name=randread --ioengine=libaio --iodepth=16 --rw=randread \
    --bs=4k --direct=1 --size=4G --numjobs=4 --runtime=60 \
    --group_reporting
```

### 2. Monitoring MariaDB

#### Métriques InnoDB I/O

```sql
-- Dashboard I/O complet
SELECT 
    -- Reads
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_reads') as data_reads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_read') / 1024 / 1024 as data_read_mb,
    
    -- Writes
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_writes') as data_writes,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_written') / 1024 / 1024 as data_written_mb,
    
    -- Fsyncs (commits)
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_fsyncs') as fsyncs,
    
    -- Pending I/O
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_pending_reads') as pending_reads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_pending_writes') as pending_writes,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_pending_fsyncs') as pending_fsyncs,
    
    -- Buffer pool I/O
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') as bp_disk_reads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') as bp_read_requests,
    
    -- Hit rate
    ROUND(
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        ), 4
    ) as buffer_pool_hit_rate_pct;
```

#### Calcul des taux I/O

```sql
-- Procédure pour calculer taux I/O (reads/writes par seconde)
DELIMITER //
CREATE OR REPLACE PROCEDURE show_io_rates()
BEGIN
    DECLARE v_uptime INT;
    
    SELECT VARIABLE_VALUE INTO v_uptime
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Uptime';
    
    SELECT 
        'I/O Rates (per second)' as metric_category,
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_data_reads') / v_uptime, 2
        ) as reads_per_sec,
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_data_writes') / v_uptime, 2
        ) as writes_per_sec,
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_data_fsyncs') / v_uptime, 2
        ) as fsyncs_per_sec,
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_data_read') / v_uptime / 1024 / 1024, 2
        ) as mb_read_per_sec,
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_data_written') / v_uptime / 1024 / 1024, 2
        ) as mb_written_per_sec;
END //
DELIMITER ;

-- Utiliser
CALL show_io_rates();
```

#### Dirty pages et flushing

```sql
-- État du flushing
SELECT 
    -- Dirty pages
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') as dirty_pages,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') as total_pages,
    
    -- Ratio
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100, 2
    ) as dirty_pages_pct,
    
    -- Seuils configurés
    @@innodb_max_dirty_pages_pct as max_dirty_pct,
    @@innodb_max_dirty_pages_pct_lwm as lwm_dirty_pct,
    
    -- Flushing stats
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_flushed') as pages_flushed;
```

**Interprétation** :

```
Dirty pages % :
  < 25%    : Normal, flushing efficace
  25-50%   : Acceptable, surveiller
  50-75%   : Attention, flushing en retard
  > 75%    : Problème, I/O saturés ou config inadaptée

Action si > 75% :
  1. Vérifier iostat : Disques saturés ?
  2. Augmenter innodb_io_capacity
  3. Vérifier innodb_max_dirty_pages_pct
  4. Considérer plus de page_cleaners
```

---

## Diagnostic des problèmes I/O

### Symptômes courants

```sql
-- 1. Requêtes lentes malgré bon buffer pool hit rate
SELECT 
    ROUND(
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        ), 2
    ) as bp_hit_rate_pct,  -- > 99%
    
    (SELECT AVG(timer_wait) / 1000000000000 
     FROM performance_schema.events_statements_history
    ) as avg_query_time_sec;  -- Mais > 1 seconde ?
    
-- Diagnostic : Problème I/O sur les quelques reads qui vont au disque

-- 2. Dirty pages élevées
SELECT 
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100, 2
    ) as dirty_pct;  -- > 75% ?
    
-- Diagnostic : Flushing ne suit pas, innodb_io_capacity trop bas

-- 3. Pending I/O élevé
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_pending_reads') as pending_reads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_pending_writes') as pending_writes;
-- > 100 pendant longtemps ?

-- Diagnostic : I/O disque saturés
```

### Arbre de décision diagnostic

```
Problème de performance suspecté lié I/O
│
├─ iostat montre %util > 90% ?
│  │
│  ├─ OUI → Disques saturés
│  │  │
│  │  ├─ await > 20ms ?
│  │  │  ├─ OUI → Disques lents (HDD ?) ou RAID dégradé
│  │  │  │  → Upgrade vers SSD ou réparer RAID
│  │  │  │
│  │  │  └─ NON → Trop d'IOPS demandées
│  │  │     → Optimiser requêtes (voir section 15.7)
│  │  │     → Ajouter index
│  │  │     → Augmenter buffer pool
│  │  │
│  │  └─ Ratio read/write ?
│  │     ├─ Majoritairement READS →
│  │     │  → Buffer pool trop petit
│  │     │  → Queries non optimisées
│  │     │
│  │     └─ Majoritairement WRITES →
│  │        → innodb_io_capacity trop bas
│  │        → Workload très write-heavy
│  │        → Considérer write caching RAID
│  │
│  └─ NON → Disques OK, problème ailleurs
│     │
│     ├─ Dirty pages > 75% ?
│     │  └─ OUI → Augmenter innodb_io_capacity
│     │
│     ├─ Buffer pool hit rate < 95% ?
│     │  └─ OUI → Augmenter buffer pool
│     │
│     └─ Requêtes lentes individuelles ?
│        └─ OUI → Problème requêtes (section 15.7)
```

---

## ✅ Points clés à retenir

- 💾 **Type de stockage = fondamental** : NVMe >> SSD SATA >> HDD (100x IOPS)
- 🆕 **MariaDB 11.8 optimisé pour SSD** : Cost optimizer, alter_copy_bulk
- 🔧 **innodb_io_capacity** : DOIT matcher capacités réelles (10k-20k pour NVMe)
- 📁 **innodb_flush_method** : O_DIRECT = essentiel (évite double buffering)
- 🚫 **innodb_flush_neighbors** : 0 sur SSD, 1 sur HDD uniquement
- 📊 **Monitoring = clé** : iostat (système) + métriques InnoDB
- ⚡ **NVMe change tout** : 70% RAM buffer pool suffisant, OS cache important
- 🎯 **Redo logs = critique** : Sur chemin commit, impacte latence directement
- 🔍 **Diagnostic méthodique** : iostat → métriques MariaDB → requêtes
- ✅ **Upgrade SSD/NVMe** : ROI énorme, 1er investissement à faire

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 InnoDB I/O Configuration](https://mariadb.com/kb/en/innodb-system-variables/#innodb-io-capacity)
- [📖 InnoDB Startup Options](https://mariadb.com/kb/en/innodb-startup-options-and-system-variables/)
- [📖 Optimizing InnoDB Disk I/O](https://mariadb.com/kb/en/optimizing-innodb-disk-io/)

### Nouveautés MariaDB 11.8

- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/changes-improvements-in-mariadb-118/)
- [📖 innodb_alter_copy_bulk](https://mariadb.com/kb/en/innodb-system-variables/#innodb_alter_copy_bulk)

### Benchmarking et tuning

- [Percona Blog - InnoDB Performance](https://www.percona.com/blog/category/mysql/innodb/)
- [Intel - NVMe Performance Guide](https://www.intel.com/content/www/us/en/support/articles/000055862/)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent chaque paramètre I/O :

### **Section 15.4.1** : [innodb_io_capacity](/15-performance-tuning/04.1-innodb-io-capacity.md)
*Dimensionnement précis de la capacité I/O selon le type de disque, avec benchmarks et formules de calcul.*

### **Section 15.4.2** : [innodb_flush_method](/15-performance-tuning/04.2-innodb-flush-method.md)
*Comparaison des méthodes de flush (O_DIRECT, fsync, etc.), impact performance, et choix optimal.*

### **Section 15.4.3** : [Optimisations SSD modernes](/15-performance-tuning/04.3-optimisations-ssd.md)
*Techniques avancées spécifiques SSD/NVMe : alignment, TRIM, over-provisioning, nouveautés 11.8.*

---

*La configuration I/O est le deuxième pilier des performances MariaDB. Avec le stockage moderne NVMe et MariaDB 11.8, les gains peuvent être spectaculaires.*

⏭️ [innodb_io_capacity](/15-performance-tuning/04.1-innodb-io-capacity.md)

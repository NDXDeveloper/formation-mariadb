🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.2 Configuration mémoire

> **Niveau** : Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Section 15.1 (Méthodologie d'optimisation)
> - Compréhension de l'architecture InnoDB
> - Connaissance des types de workload (OLTP, OLAP, mixte)
> - Bases en administration système Linux

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Dimensionner correctement** le buffer pool InnoDB selon le workload et le matériel
- **Configurer les buffer pool instances** pour optimiser la concurrence
- **Gérer efficacement** les autres buffers mémoire (key buffer, sort buffer, etc.)
- **Calculer l'empreinte mémoire totale** de MariaDB et éviter le swap
- **Adapter la configuration** selon le type de stockage (SSD vs HDD)
- **Monitorer l'utilisation mémoire** et identifier les problèmes
- **Optimiser pour MariaDB 11.8** avec les nouvelles capacités SSD-aware

---

## Introduction

La configuration mémoire est **l'un des leviers les plus impactants** pour les performances de MariaDB. Une configuration optimale peut multiplier les performances par 10 ou plus, tandis qu'une mauvaise configuration peut causer :

- ❌ Swap actif (pire scénario : performances divisées par 100)
- ❌ Buffer pool trop petit (augmentation massive des I/O disque)
- ❌ Buffer pool trop grand (RAM insuffisante pour l'OS et les connexions)
- ❌ Contention sur les mutex (trop peu d'instances)

### L'architecture mémoire de MariaDB

```
┌────────────────────────────────────────────────────────────┐
│                    RAM SYSTÈME (exemple 64 GB)             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         SYSTÈME D'EXPLOITATION                     │    │
│  │  • Filesystem cache (8-12 GB)                      │    │
│  │  • Kernel + processus système (2-4 GB)             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         MARIADB - MÉMOIRE GLOBALE                  │    │
│  │  ┌──────────────────────────────────────────┐      │    │
│  │  │  InnoDB Buffer Pool (40-45 GB)           │      │    │
│  │  │  • Pages de données                      │      │    │
│  │  │  • Index                                 │      │    │
│  │  │  • Change buffer                         │      │    │
│  │  │  • Adaptive hash index                   │      │    │
│  │  └──────────────────────────────────────────┘      │    │
│  │                                                    │    │
│  │  • Key buffer (MyISAM) : 128-256 MB                │    │
│  │  • Query cache (DEPRECATED) : 0 MB                 │    │
│  │  • InnoDB log buffer : 16-64 MB                    │    │
│  │  • Table cache : 2000-4000 tables                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │      MARIADB - MÉMOIRE PAR CONNEXION               │    │
│  │  Chaque connexion utilise :                        │    │
│  │  • Thread stack : 292 KB (default)                 │    │
│  │  • Sort buffer : 2 MB (si ORDER BY/GROUP BY)       │    │
│  │  • Join buffer : 256 KB (par join sans index)      │    │
│  │  • Read buffer : 128 KB (scan séquentiel)          │    │
│  │  • Binlog cache : 32 KB (si binlog actif)          │    │
│  │                                                    │    │
│  │  Exemple : 500 connexions simultanées              │    │
│  │  → 500 × (292 KB + buffers) ≈ 1-3 GB               │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
└────────────────────────────────────────────────────────────┘

Total recommandé pour MariaDB : 70-80% de la RAM sur SSD
                                80-90% de la RAM sur HDD (legacy)
```

### Évolution avec MariaDB 11.8 et stockage moderne

🆕 **Changements importants avec MariaDB 11.8 + SSD/NVMe** :

Avant (HDD + MariaDB 10.x) :
- Buffer pool : 80-90% RAM (maximiser le cache pour éviter I/O lents)
- Filesystem cache : minimal
- Philosophie : "Cache everything in RAM"

Maintenant (SSD/NVMe + MariaDB 11.8) :
- Buffer pool : 70-80% RAM (I/O rapides, moins critique de tout cacher)
- Filesystem cache : plus important (8-12 GB)
- Philosophie : "Balance RAM between buffer pool and OS cache"

**Raisons** :
- Les SSD ont des latences 100x plus faibles que les HDD
- Le filesystem cache peut servir efficacement les I/O non cachés
- Le cost optimizer 11.8 gère mieux les accès disque sur SSD
- Moins de pression pour avoir un buffer pool gigantesque

---

## Vue d'ensemble des paramètres mémoire

### Hiérarchie des paramètres par impact

```
┌──────────────────────────────────────────────────────┐
│  IMPACT CRITIQUE (>80% de l'utilisation mémoire)     │
├──────────────────────────────────────────────────────┤
│  1. innodb_buffer_pool_size                          │
│     • 70-80% RAM (SSD)                               │
│     • 40-45 GB pour serveur 64 GB                    │
│     • LE paramètre le plus important                 │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  IMPACT IMPORTANT (5-15% de l'utilisation)           │
├──────────────────────────────────────────────────────┤
│  2. Mémoire par connexion × max_connections          │
│     • thread_stack : 292 KB                          │
│     • sort_buffer_size : 2 MB                        │
│     • join_buffer_size : 256 KB                      │
│     • read_buffer_size : 128 KB                      │
│     • read_rnd_buffer_size : 256 KB                  │
│                                                      │
│  3. innodb_log_buffer_size                           │
│     • 16-64 MB selon write workload                  │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  IMPACT MODÉRÉ (1-5% de l'utilisation)               │
├──────────────────────────────────────────────────────┤
│  4. key_buffer_size (MyISAM)                         │
│     • 128-256 MB si tables InnoDB uniquement         │
│     • Plus si utilisation MyISAM                     │
│                                                      │
│  5. tmp_table_size / max_heap_table_size             │
│     • 64-128 MB par défaut                           │
│     • Plus si beaucoup de temp tables                │
└──────────────────────────────────────────────────────┘
```

### Formule de calcul de l'empreinte mémoire totale

```sql
-- Calcul estimé de l'utilisation mémoire MariaDB
SELECT 
    -- Mémoire globale
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'innodb_buffer_pool_size') / 1024 / 1024 / 1024, 2
    ) as buffer_pool_gb,
    
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'innodb_log_buffer_size') / 1024 / 1024, 2
    ) as log_buffer_mb,
    
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'key_buffer_size') / 1024 / 1024, 2
    ) as key_buffer_mb,
    
    -- Mémoire par connexion
    ROUND(
        (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'thread_stack') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'sort_buffer_size') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'join_buffer_size') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'read_buffer_size') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'read_rnd_buffer_size')
        ) / 1024 / 1024, 2
    ) as per_connection_mb,
    
    -- Max connexions
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME = 'max_connections') as max_connections,
    
    -- Estimation mémoire totale (formule simplifiée)
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'innodb_buffer_pool_size') / 1024 / 1024 / 1024 +
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'key_buffer_size') / 1024 / 1024 / 1024 +
        (
            (
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'thread_stack') +
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'sort_buffer_size') +
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'join_buffer_size') +
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'read_buffer_size') +
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'read_rnd_buffer_size')
            ) *
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'max_connections')
        ) / 1024 / 1024 / 1024,
        2
    ) as estimated_total_gb;
```

**Formule manuelle** :
```
Total RAM MariaDB ≈ 
    innodb_buffer_pool_size +
    key_buffer_size +
    (per_thread_buffers × max_connections) +
    innodb_log_buffer_size +
    tmp_table_size +
    overhead (~1-2 GB)

Où per_thread_buffers = 
    thread_stack + 
    sort_buffer_size + 
    join_buffer_size + 
    read_buffer_size + 
    read_rnd_buffer_size + 
    binlog_cache_size
```

---

## InnoDB Buffer Pool : Vue d'ensemble

Le buffer pool est **de loin le paramètre mémoire le plus important** de MariaDB. Il cache :

1. **Pages de données** : Les lignes des tables InnoDB
2. **Pages d'index** : Les structures d'index B-Tree
3. **Change buffer** : Modifications secondaires d'index en attente
4. **Adaptive Hash Index** : Hash index automatique pour hot data

### Métriques clés du buffer pool

```sql
-- Dashboard complet du buffer pool
SELECT 
    -- Taille totale
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'innodb_buffer_pool_size') / 1024 / 1024 / 1024, 2
    ) as configured_size_gb,
    
    -- Pages totales
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') as total_pages,
    
    -- Pages utilisées (data)
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') as data_pages,
    
    -- Pages libres
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_free') as free_pages,
    
    -- Pages dirty (modifiées, pas encore écrites)
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') as dirty_pages,
    
    -- Pourcentage utilisé
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100, 2
    ) as utilization_pct,
    
    -- Hit rate (métrique CRITIQUE)
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'), 0), 2
    ) as hit_rate_ratio,
    
    ROUND(
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        ), 4
    ) as hit_rate_pct;
```

### Interprétation du buffer pool hit rate

```
┌────────────────────────────────────────────────────┐
│  HIT RATE          ÉTAT            ACTION          │
├────────────────────────────────────────────────────┤
│  > 99.5%           Excellent       RAS             │
│  99-99.5%          Très bon        RAS             │
│  95-99%            Bon             Monitorer       │
│  90-95%            Acceptable      Considérer ↑    │
│  80-90%            Problématique   Augmenter BP    │
│  < 80%             Critique        Urgence         │
└────────────────────────────────────────────────────┘

⚠️ IMPORTANT sur SSD/NVMe + MariaDB 11.8 :
Un hit rate de 95-97% peut être acceptable si :
- Les I/O disque restent rapides (<5ms latence)
- Le throughput est satisfaisant
- Pas de contention sur le buffer pool

Sur HDD, visez toujours >99%
```

💡 **Conseil** : Le hit rate seul ne suffit pas. Regardez aussi :
- La latence des I/O disque (`iostat -x`)
- Le throughput global (`queries/sec`)
- Les pages dirty ratio (flush performance)

---

## Buffer Pool Instances : Optimisation de la concurrence

### Pourquoi plusieurs instances ?

Sur les systèmes multi-cœurs modernes, un seul buffer pool peut devenir un **point de contention** :

```
┌──────────────────────────────────────────────────┐
│  1 INSTANCE (mauvais sur 32+ cores)              │
├──────────────────────────────────────────────────┤
│                                                  │
│   Thread 1 ──┐                                   │
│   Thread 2 ──┤                                   │
│   Thread 3 ──┤──→ [MUTEX] ──→ [Buffer Pool]      │
│   Thread 4 ──┤                                   │
│   ...        ┘                                   │
│   Thread 32 ─┘    ↑ Contention !                 │
│                                                  │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│  PLUSIEURS INSTANCES (bon sur 32+ cores)         │
├──────────────────────────────────────────────────┤
│                                                  │
│   Thread 1-8  ──→ [Buffer Pool Instance 1]       │
│   Thread 9-16 ──→ [Buffer Pool Instance 2]       │
│   Thread 17-24 ──→ [Buffer Pool Instance 3]      │
│   Thread 25-32 ──→ [Buffer Pool Instance 4]      │
│                                                  │
│   ↑ Moins de contention, meilleure concurrence   │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Configuration des instances

```ini
[mariadb]
# Règle générale : 1 instance par GB de buffer pool, max 64 instances

# Exemple 1 : Serveur 64 GB RAM, buffer pool 48 GB
innodb_buffer_pool_size = 48G
innodb_buffer_pool_instances = 48  # 1 par GB

# Exemple 2 : Serveur 128 GB RAM, buffer pool 96 GB
innodb_buffer_pool_size = 96G
innodb_buffer_pool_instances = 64  # Max 64 instances

# Exemple 3 : Serveur 16 GB RAM, buffer pool 12 GB
innodb_buffer_pool_size = 12G
innodb_buffer_pool_instances = 12  # 1 par GB
```

**Règles de dimensionnement** :

| RAM totale | Buffer Pool | Instances | Commentaire |
|------------|-------------|-----------|-------------|
| 8 GB | 6 GB | 6 | Petit serveur |
| 16 GB | 12 GB | 12 | Serveur moyen |
| 32 GB | 24 GB | 24 | Serveur standard |
| 64 GB | 48 GB | 48 | Production OLTP |
| 128 GB | 96 GB | 64 | Max instances (limite) |
| 256 GB | 192 GB | 64 | High-end (limite 64) |

⚠️ **Contraintes** :
- Chaque instance doit faire au minimum 1 GB
- Maximum 64 instances (limite MariaDB)
- Le buffer pool est divisé équitablement entre instances

```sql
-- Vérifier la configuration actuelle des instances
SELECT 
    @@innodb_buffer_pool_size / 1024 / 1024 / 1024 as buffer_pool_gb,
    @@innodb_buffer_pool_instances as instances,
    ROUND(
        (@@innodb_buffer_pool_size / 1024 / 1024 / 1024) / 
        @@innodb_buffer_pool_instances, 2
    ) as gb_per_instance;

-- Vérifier l'utilisation par instance
SELECT 
    pool_id,
    pool_size as pages_total,
    free_buffers as pages_free,
    database_pages as pages_data,
    ROUND(database_pages / pool_size * 100, 2) as utilization_pct
FROM information_schema.INNODB_BUFFER_POOL_STATS
ORDER BY pool_id;
```

---

## Autres buffers critiques

### InnoDB Log Buffer

Le log buffer stocke les modifications en attente d'écriture dans les redo logs.

```ini
[mariadb]
# Dimensionnement selon workload

# OLTP léger (faible write rate)
innodb_log_buffer_size = 16M

# OLTP standard
innodb_log_buffer_size = 32M

# OLTP intense ou batch updates
innodb_log_buffer_size = 64M

# Très haute volumétrie (rare)
innodb_log_buffer_size = 128M
```

**Comment dimensionner** :

```sql
-- Vérifier si le log buffer est suffisant
-- Si Log_waits > 0, augmenter innodb_log_buffer_size
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'Innodb_log_waits',           -- Doit être 0
    'Innodb_log_write_requests',  -- Écritures dans le buffer
    'Innodb_log_writes'            -- Flush sur disque
);
```

Si `Innodb_log_waits > 0` :
- Le log buffer est trop petit
- Les transactions attendent que le buffer se libère
- **Action** : Doubler `innodb_log_buffer_size`

---

## Buffers par connexion

Chaque connexion alloue sa propre mémoire pour les opérations. Ces buffers sont alloués **à la demande** (pas au moment de la connexion).

### Sort Buffer

Utilisé pour les opérations `ORDER BY` et `GROUP BY`.

```ini
[mariadb]
# Taille par défaut
sort_buffer_size = 2M  # 2 MB par connexion (quand utilisé)

# ⚠️ Ne pas augmenter excessivement !
# Si queries complexes avec gros sorts :
sort_buffer_size = 4M  # Max recommandé
```

💡 **Important** : Ce buffer est alloué **entièrement en RAM** même si le sort est petit. Ne pas mettre 256 MB "au cas où" !

```sql
-- Vérifier l'utilisation du sort buffer
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Sort_scan') as sort_scan,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Sort_range') as sort_range,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Sort_merge_passes') as merge_passes,
    -- Si merge_passes élevé, considérer augmenter sort_buffer_size
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Sort_merge_passes') /
        NULLIF(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Sort_scan') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Sort_range'), 0
        ) * 100, 4
    ) as merge_passes_pct;
```

### Join Buffer

Utilisé pour les jointures **sans index** (full table scan joins).

```ini
[mariadb]
join_buffer_size = 256K  # Défaut, généralement suffisant

# Si beaucoup de joins sans index :
join_buffer_size = 512K  # Ou 1M maximum

# ⚠️ La vraie solution : CRÉER LES INDEX APPROPRIÉS
```

```sql
-- Identifier les requêtes qui utilisent join buffers
SELECT 
    digest_text,
    count_star,
    sum_no_index_used,
    sum_no_good_index_used
FROM performance_schema.events_statements_summary_by_digest
WHERE sum_no_index_used > 0 OR sum_no_good_index_used > 0
ORDER BY sum_no_index_used DESC
LIMIT 20;
```

💡 **Best practice** : Si `join_buffer_size` doit être augmenté, c'est souvent le signe d'index manquants.

### Read Buffers

```ini
[mariadb]
# Scans séquentiels
read_buffer_size = 128K  # Défaut OK pour OLTP
read_buffer_size = 1M    # Si beaucoup de full table scans (OLAP)

# Scans après ORDER BY
read_rnd_buffer_size = 256K  # Défaut OK
read_rnd_buffer_size = 512K  # Si ORDER BY fréquents sur gros datasets
```

---

## Temporary Tables

Les tables temporaires sont créées pour :
- Requêtes avec `GROUP BY`, `ORDER BY`, `DISTINCT`
- Sous-requêtes complexes
- Unions

### Configuration

```ini
[mariadb]
# Taille max temp table en mémoire (MEMORY engine)
tmp_table_size = 64M
max_heap_table_size = 64M  # Doit être identique

# Pour workload analytique :
tmp_table_size = 256M
max_heap_table_size = 256M

# ⚠️ Ne pas mettre trop haut : risque OOM si beaucoup de connexions
```

### Monitoring des temp tables

```sql
-- Ratio de temp tables créées sur disque (à minimiser)
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') as tmp_disk,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Created_tmp_tables') as tmp_total,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_tables'), 0) * 100, 2
    ) as disk_pct;
```

**Interprétation** :

| disk_pct | État | Action |
|----------|------|--------|
| < 10% | Excellent | RAS |
| 10-25% | Bon | Monitorer |
| 25-50% | Attention | Augmenter tmp_table_size |
| > 50% | Problème | Optimiser requêtes + augmenter config |

💡 **Attention** : Si trop de temp tables sur disque malgré tmp_table_size élevé, vérifier :
- Présence de colonnes BLOB/TEXT (forcent disque)
- Requêtes avec large resultset
- Besoin d'index appropriés

---

## Key Buffer (MyISAM)

Si vous utilisez **uniquement InnoDB** (recommandé), le key buffer peut être minimal.

```ini
[mariadb]
# Serveur InnoDB-only
key_buffer_size = 128M  # Minimal pour tables système

# Serveur avec tables MyISAM
key_buffer_size = 2G    # 20-30% du buffer pool InnoDB
```

```sql
-- Utilisation du key buffer
SELECT 
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
         WHERE VARIABLE_NAME = 'key_buffer_size') / 1024 / 1024, 2
    ) as key_buffer_mb,
    
    ROUND(
        (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Key_blocks_used') * 1024
        ) / 1024 / 1024, 2
    ) as used_mb,
    
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Key_read_requests') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Key_reads'), 0), 2
    ) as hit_ratio;
```

---

## Calcul de la mémoire totale et éviter le swap

### Méthode de calcul complète

```bash
# 1. RAM totale disponible
free -g | grep Mem | awk '{print $2}'
# Exemple : 64 GB

# 2. Réserver pour l'OS et le filesystem cache
# Formule : RAM_OS = max(4 GB, RAM_total * 0.15)
# Pour 64 GB : max(4, 64 * 0.15) = max(4, 9.6) = 9.6 GB → arrondir à 10 GB

# 3. RAM disponible pour MariaDB
# 64 GB - 10 GB = 54 GB

# 4. Buffer pool (70-80% de la RAM MariaDB)
# 54 GB × 0.75 = 40.5 GB → arrondir à 40 GB

# 5. Mémoire connexions
# max_connections × per_thread_buffers
# Exemple : 500 connexions × 3 MB = 1.5 GB

# 6. Total MariaDB
# 40 GB (buffer pool) + 1.5 GB (connexions) + 0.5 GB (autres) = 42 GB

# 7. Vérification
# 42 GB < 54 GB ✓ OK
```

### Script de vérification mémoire

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE check_memory_config()
BEGIN
    DECLARE v_buffer_pool_gb DECIMAL(10,2);
    DECLARE v_max_conn INT;
    DECLARE v_per_thread_mb DECIMAL(10,2);
    DECLARE v_global_buffers_gb DECIMAL(10,2);
    DECLARE v_estimated_total_gb DECIMAL(10,2);
    
    -- Buffer pool
    SELECT @@innodb_buffer_pool_size / 1024 / 1024 / 1024 
    INTO v_buffer_pool_gb;
    
    -- Max connexions
    SELECT @@max_connections INTO v_max_conn;
    
    -- Mémoire par thread
    SELECT (
        @@thread_stack +
        @@sort_buffer_size +
        @@join_buffer_size +
        @@read_buffer_size +
        @@read_rnd_buffer_size
    ) / 1024 / 1024 INTO v_per_thread_mb;
    
    -- Buffers globaux
    SELECT (
        @@key_buffer_size +
        @@innodb_log_buffer_size +
        @@tmp_table_size
    ) / 1024 / 1024 / 1024 INTO v_global_buffers_gb;
    
    -- Total estimé
    SET v_estimated_total_gb = 
        v_buffer_pool_gb + 
        v_global_buffers_gb +
        (v_per_thread_mb * v_max_conn / 1024);
    
    -- Affichage
    SELECT 
        v_buffer_pool_gb as buffer_pool_gb,
        v_global_buffers_gb as other_global_gb,
        v_per_thread_mb as per_thread_mb,
        v_max_conn as max_connections,
        ROUND(v_per_thread_mb * v_max_conn / 1024, 2) as max_threads_gb,
        ROUND(v_estimated_total_gb, 2) as estimated_total_gb,
        '---' as separator,
        CASE 
            WHEN v_estimated_total_gb > 100 THEN 
                'WARNING: >100GB, vérifier RAM système'
            WHEN v_estimated_total_gb > 80 THEN 
                'ATTENTION: Haute utilisation mémoire'
            ELSE 
                'OK'
        END as status;
END //
DELIMITER ;

-- Exécuter
CALL check_memory_config();
```

### Monitoring du swap

```bash
# Vérifier si swap actif
free -h
#               total        used        free      shared  buff/cache   available
# Mem:            62Gi        45Gi       2.1Gi       1.0Gi        15Gi        15Gi
# Swap:          8.0Gi          0B       8.0Gi  ← Doit être 0B !

# Vérifier l'activité swap
vmstat 1 10
# si : swap in (doit être 0)
# so : swap out (doit être 0)

# Surveiller en continu
watch -n 1 'free -h; echo; vmstat 1 2 | tail -1'
```

⚠️ **CRITIQUE** : Si swap actif (`si` ou `so` > 0), les performances peuvent chuter de 100x !

**Solutions si swap actif** :
1. Réduire `innodb_buffer_pool_size`
2. Réduire `max_connections`
3. Ajouter RAM physique
4. Désactiver le swap (environnement dédié uniquement)

```bash
# Désactiver swap (serveur dédié MariaDB uniquement)
sudo swapoff -a
# Rendre permanent
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

---

## Configurations types par cas d'usage

### 1. OLTP Haute Performance (64 GB RAM, SSD)

```ini
[mariadb]
# Architecture moderne : 16 cores, 64 GB RAM, NVMe SSD

# Buffer pool : 70% RAM
innodb_buffer_pool_size = 45G
innodb_buffer_pool_instances = 45

# Log buffer
innodb_log_buffer_size = 32M

# Connexions (500 max)
max_connections = 500
thread_cache_size = 100

# Buffers par thread (conservateurs)
sort_buffer_size = 2M
join_buffer_size = 256K
read_buffer_size = 128K
read_rnd_buffer_size = 256K

# Temp tables
tmp_table_size = 64M
max_heap_table_size = 64M

# MyISAM minimal
key_buffer_size = 128M

# Table cache
table_open_cache = 4000
table_definition_cache = 2000
```

### 2. OLAP / Analytics (128 GB RAM, SSD)

```ini
[mariadb]
# Serveur analytique : 32 cores, 128 GB RAM, SSD

# Buffer pool : 75% RAM
innodb_buffer_pool_size = 96G
innodb_buffer_pool_instances = 64  # Max

# Log buffer (gros batch updates)
innodb_log_buffer_size = 64M

# Moins de connexions mais plus de ressources par query
max_connections = 200
thread_cache_size = 50

# Buffers généreux pour sorts/joins complexes
sort_buffer_size = 4M
join_buffer_size = 1M
read_buffer_size = 1M
read_rnd_buffer_size = 512K

# Temp tables larges
tmp_table_size = 256M
max_heap_table_size = 256M

# MyISAM si besoin
key_buffer_size = 2G
```

### 3. Workload mixte (32 GB RAM, SSD)

```ini
[mariadb]
# Serveur polyvalent : 8 cores, 32 GB RAM, SSD

# Buffer pool : 75% RAM
innodb_buffer_pool_size = 24G
innodb_buffer_pool_instances = 24

# Log buffer équilibré
innodb_log_buffer_size = 32M

# Connexions modérées
max_connections = 300
thread_cache_size = 75

# Buffers standards
sort_buffer_size = 2M
join_buffer_size = 256K
read_buffer_size = 256K
read_rnd_buffer_size = 256K

# Temp tables moyennes
tmp_table_size = 128M
max_heap_table_size = 128M

# MyISAM
key_buffer_size = 256M
```

### 4. Petit serveur (8 GB RAM)

```ini
[mariadb]
# Petit serveur : 4 cores, 8 GB RAM

# Buffer pool : 70% RAM
innodb_buffer_pool_size = 5G
innodb_buffer_pool_instances = 5

# Log buffer minimal
innodb_log_buffer_size = 16M

# Connexions limitées
max_connections = 150
thread_cache_size = 50

# Buffers minimaux
sort_buffer_size = 1M
join_buffer_size = 256K
read_buffer_size = 128K
read_rnd_buffer_size = 128K

# Temp tables petites
tmp_table_size = 32M
max_heap_table_size = 32M

# MyISAM
key_buffer_size = 128M
```

---

## Ajustement dynamique du buffer pool

🆕 **Depuis MariaDB 10.2+**, le buffer pool peut être redimensionné **sans restart** :

```sql
-- Vérifier la taille actuelle
SELECT @@innodb_buffer_pool_size / 1024 / 1024 / 1024 as current_gb;

-- Augmenter le buffer pool (exemple : 32G → 48G)
SET GLOBAL innodb_buffer_pool_size = 48 * 1024 * 1024 * 1024;
-- L'opération peut prendre quelques secondes/minutes

-- Suivre la progression
SHOW STATUS LIKE 'Innodb_buffer_pool_resize_status';
```

⚠️ **Important** :
- Le resize se fait par chunks de 128 MB (valeur par défaut)
- L'opération peut prendre du temps sur gros buffer pools
- Les performances peuvent être impactées pendant le resize
- **Toujours tester en non-prod d'abord**

```sql
-- Configurer la taille des chunks
-- (doit être fait AVANT le premier resize, nécessite restart)
SET GLOBAL innodb_buffer_pool_chunk_size = 134217728;  -- 128 MB (défaut)
```

---

## Monitoring continu de la mémoire

### Dashboard de surveillance

```sql
-- Vue de monitoring quotidien
CREATE OR REPLACE VIEW v_memory_health AS
SELECT 
    -- Buffer pool
    ROUND(@@innodb_buffer_pool_size / 1024 / 1024 / 1024, 2) as bp_config_gb,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100, 2
    ) as bp_usage_pct,
    ROUND(
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        ), 4
    ) as bp_hit_rate_pct,
    
    -- Connexions
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_connected') as conn_current,
    @@max_connections as conn_max,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Threads_connected') / 
        @@max_connections * 100, 2
    ) as conn_usage_pct,
    
    -- Temp tables
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_tables'), 0) * 100, 2
    ) as tmp_disk_pct;

-- Utiliser
SELECT * FROM v_memory_health;
```

### Alertes automatiques

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE alert_memory_issues()
BEGIN
    DECLARE v_bp_hit_rate DECIMAL(10,4);
    DECLARE v_tmp_disk_pct DECIMAL(10,2);
    DECLARE v_conn_pct DECIMAL(10,2);
    
    -- Buffer pool hit rate
    SELECT 
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        )
    INTO v_bp_hit_rate;
    
    -- Temp disk ratio
    SELECT 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_tables'), 0) * 100
    INTO v_tmp_disk_pct;
    
    -- Connexions ratio
    SELECT 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Threads_connected') / 
        @@max_connections * 100
    INTO v_conn_pct;
    
    -- Alertes
    IF v_bp_hit_rate < 95 THEN
        SELECT CONCAT('ALERT: Buffer pool hit rate faible: ', 
                     v_bp_hit_rate, '%') as alert;
    END IF;
    
    IF v_tmp_disk_pct > 25 THEN
        SELECT CONCAT('ALERT: Trop de temp tables sur disque: ', 
                     v_tmp_disk_pct, '%') as alert;
    END IF;
    
    IF v_conn_pct > 80 THEN
        SELECT CONCAT('ALERT: Connexions élevées: ', 
                     v_conn_pct, '% de max_connections') as alert;
    END IF;
END //
DELIMITER ;
```

---

## ✅ Points clés à retenir

- 🎯 **Buffer pool = paramètre #1** : 70-80% RAM sur SSD, le plus important de tous
- 📊 **Hit rate > 95%** : Objectif minimum, >99% excellent (mais 95-97% OK sur SSD rapide)
- 🔢 **Instances = 1 par GB** : Maximum 64, optimise la concurrence multi-cœurs
- 💾 **Formule mémoire** : BP + (per_thread × max_conn) + global_buffers < 80% RAM
- ⚠️ **Swap = ennemi #1** : Doit être à 0, sinon performances catastrophiques
- 🔄 **Resize dynamique** : Buffer pool ajustable sans restart depuis 10.2+
- 📈 **Monitoring continu** : Surveiller hit rate, dirty pages, temp tables disk%
- 🆕 **11.8 + SSD** : 70-80% RAM suffisant (vs 80-90% sur HDD), OS cache important
- 🚫 **Éviter** : Query cache (déprécié), buffers thread trop gros, over-provisioning
- ✅ **Valider** : Toujours mesurer avant/après un changement de configuration

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [📖 InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [📖 Server System Variables](https://mariadb.com/kb/en/server-system-variables/)

### Outils de tuning

- [MySQLTuner](https://github.com/major/MySQLTuner-perl) - Analyse et recommandations
- [Percona Monitoring Plugins](https://www.percona.com/software/database-tools/percona-monitoring-and-management)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent chaque aspect de la configuration mémoire :

### **Section 15.2.1** : [InnoDB Buffer Pool : Dimensionnement](/15-performance-tuning/02.1-innodb-buffer-pool.md)
*Méthodologie complète de dimensionnement du buffer pool selon le workload, avec formules de calcul, benchmarks et optimisations avancées.*

### **Section 15.2.2** : [Buffer Pool Instances](/15-performance-tuning/02.2-buffer-pool-instances.md)
*Configuration multi-instances pour systèmes multi-cœurs, préchargement, et gestion de la mémoire InnoDB.*

### **Section 15.2.3** : [Key Buffer (MyISAM)](/15-performance-tuning/02.3-key-buffer.md)
*Configuration du key buffer pour tables MyISAM, migration vers InnoDB, et cas d'usage legacy.*

---

*La configuration mémoire est la pierre angulaire des performances MariaDB. Prenez le temps de bien dimensionner et monitorer ces paramètres pour des résultats optimaux.*

⏭️ [InnoDB Buffer Pool : Dimensionnement](/15-performance-tuning/02.1-innodb-buffer-pool.md)

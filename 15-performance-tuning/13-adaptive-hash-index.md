🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.13 Adaptive Hash Index et Buffer Pool optimizations

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Sections 15.1-15.5 (Méthodologie, Mémoire, InnoDB)
> - Compréhension profonde de l'architecture InnoDB
> - Connaissance des index B-tree
> - Expérience en tuning de performance

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre l'Adaptive Hash Index** et son fonctionnement interne
- **Identifier les workloads** bénéficiant de l'AHI
- **Configurer et optimiser** l'AHI pour votre environnement
- **Maîtriser les optimisations** avancées du Buffer Pool
- **Optimiser les instances** multiples du Buffer Pool
- **Configurer le prefetching** et le read-ahead
- **Monitorer les performances** de l'AHI et du Buffer Pool
- **Diagnostiquer les problèmes** de contention
- **Appliquer les best practices** de production
- **Exploiter les nouveautés** MariaDB 11.8

---

## Introduction

L'**Adaptive Hash Index (AHI)** et le **Buffer Pool** sont deux mécanismes internes d'InnoDB qui ont un **impact majeur** sur les performances :

```
┌────────────────────────────────────────────────────┐
│  ARCHITECTURE INNODB : COMPOSANTS CRITIQUES        │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │         BUFFER POOL                          │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  LRU List (pages fréquentes)           │  │  │
│  │  │  Free List (pages libres)              │  │  │
│  │  │  Flush List (pages dirty)              │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  │                                              │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  ADAPTIVE HASH INDEX                   │  │  │
│  │  │  (Hash index in-memory)                │  │  │
│  │  │  • Court-circuite B-tree               │  │  │
│  │  │  • Accès O(1) vs O(log n)              │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
│                    ↓                               │
│  ┌──────────────────────────────────────────────┐  │
│  │         TABLESPACE (.ibd files)              │  │
│  │  • Index B-tree (primaire et secondaires)    │  │
│  │  • Data pages (16 KB)                        │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  Flux normal :                                     │
│  Query → B-tree index (log n lookups) → Data       │
│                                                    │
│  Avec AHI :                                        │
│  Query → AHI (1 lookup) → Data ✅ RAPIDE           │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Impact sur les performances

```
Gain typique avec AHI :
• Workload read-heavy avec patterns répétitifs : +20-40%
• OLTP avec point selects : +15-30%
• Workload avec full table scans : 0% (aucun gain)
• Workload avec contention élevée : Négatif (ralentit)
```

---

## Adaptive Hash Index (AHI)

### Principe de fonctionnement

L'**Adaptive Hash Index** est un hash index automatique construit **en mémoire** par InnoDB pour accélérer les accès fréquents.

```
┌────────────────────────────────────────────────────┐
│  FONCTIONNEMENT DE L'AHI                           │
├────────────────────────────────────────────────────┤
│                                                    │
│  Requête typique sans AHI :                        │
│  ─────────────────────────                         │
│  SELECT * FROM users WHERE id = 12345;             │
│                                                    │
│  1. Recherche dans B-tree index                    │
│     ├─ Root node                                   │
│     ├─ Internal node(s)                            │
│     └─ Leaf node → Data                            │
│     Complexité : O(log n) = ~4-5 lookups           │
│                                                    │
│  Requête avec AHI activé :                         │
│  ─────────────────────                             │
│  SELECT * FROM users WHERE id = 12345;             │
│                                                    │
│  InnoDB détecte pattern répétitif :                │
│  "WHERE id = ?" utilisé fréquemment                │
│                                                    │
│  Construit automatiquement hash index :            │
│  ┌────────────────────────────────────┐            │
│  │  AHI Hash Table                    │            │
│  │  ───────────────                   │            │
│  │  12345 → Pointeur direct vers page │            │
│  │  67890 → Pointeur direct vers page │            │
│  │  ...                               │            │
│  └────────────────────────────────────┘            │
│                                                    │
│  1. Hash lookup : HASH(12345) → Page directement   │
│     Complexité : O(1) = 1 lookup ✅                │
│                                                    │
│  Gain : 4-5 lookups → 1 lookup = 75-80% plus vite  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Quand l'AHI est utile ✅

```
┌────────────────────────────────────────────────────┐
│  WORKLOADS BÉNÉFICIANT DE L'AHI                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. OLTP avec POINT SELECTS répétitifs             │
│     SELECT * FROM orders WHERE order_id = ?        │
│     • Même pattern répété des millions de fois     │
│     • Valeurs différentes mais même colonne        │
│     → Gain : +20-40%                               │
│                                                    │
│  2. PRIMARY KEY lookups fréquents                  │
│     SELECT * FROM users WHERE id = ?               │
│     • Accès direct par clé primaire                │
│     • Très fréquent en OLTP                        │
│     → Gain : +15-30%                               │
│                                                    │
│  3. SECONDARY INDEX accès répétitifs               │
│     SELECT * FROM products WHERE sku = ?           │
│     • Index secondaire utilisé fréquemment         │
│     • Patterns stables                             │
│     → Gain : +10-25%                               │
│                                                    │
│  4. HIGH READ / LOW WRITE ratio                    │
│     • 80%+ reads, <20% writes                      │
│     • Données relativement stables                 │
│     → AHI reste pertinent longtemps                │
│                                                    │
│  5. WORKING SET en mémoire                         │
│     • Dataset tient dans buffer pool               │
│     • Pas de disk I/O fréquent                     │
│     → AHI maximise gains CPU                       │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Quand l'AHI est contre-productif ❌

```
┌────────────────────────────────────────────────────┐
│  WORKLOADS OÙ DÉSACTIVER L'AHI                     │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. FULL TABLE SCANS fréquents                     │
│     SELECT * FROM logs WHERE date > ?              │
│     • Pas de patterns répétitifs                   │
│     • AHI non applicable                           │
│     → Overhead inutile                             │
│                                                    │
│  2. WRITE-HEAVY workload                           │
│     • >50% writes (INSERT, UPDATE, DELETE)         │
│     • AHI invalidé constamment                     │
│     • Overhead de reconstruction                   │
│     → Performance dégradée                         │
│                                                    │
│  3. HAUTES CONTENTIONS                             │
│     • Beaucoup de threads concurrents              │
│     • Contention sur AHI mutex/latches             │
│     • MariaDB <10.5 : Mutex global (bottleneck)    │
│     → Ralentissement significatif                  │
│                                                    │
│  4. PATTERNS D'ACCÈS ALÉATOIRES                    │
│     SELECT * FROM table WHERE random_column = ?    │
│     • Pas de répétition                            │
│     • AHI ne trouve pas de patterns                │
│     → Pas de construction, overhead uniquement     │
│                                                    │
│  5. MÉMOIRE LIMITÉE                                │
│     • RAM insuffisante                             │
│     • AHI consomme précieuse mémoire               │
│     • Mieux utilisée pour buffer pool              │
│     → Désactiver pour libérer RAM                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Configuration de l'AHI

```sql
-- Vérifier si AHI est activé
SHOW VARIABLES LIKE 'innodb_adaptive_hash_index';
-- ON par défaut

-- Désactiver AHI (si contre-productif)
SET GLOBAL innodb_adaptive_hash_index = OFF;

-- Réactiver
SET GLOBAL innodb_adaptive_hash_index = ON;

-- Configuration permanente dans my.cnf
[mariadb]
innodb_adaptive_hash_index = ON  # ou OFF
```

### Partitionnement de l'AHI (MariaDB 10.5+)

**Problème historique** : Avant MariaDB 10.5, l'AHI utilisait un mutex global, créant une **contention massive** sur systèmes multi-cores.

**Solution** : Partitionnement de l'AHI en multiples structures indépendantes.

```sql
-- MariaDB 10.5+ : AHI partitionné par défaut
-- Nombre de partitions = innodb_adaptive_hash_index_parts

SHOW VARIABLES LIKE 'innodb_adaptive_hash_index_parts';
-- Valeur par défaut : 8

-- Augmenter pour réduire contention (serveurs >32 cores)
SET GLOBAL innodb_adaptive_hash_index_parts = 16;

-- Configuration my.cnf
[mariadb]
innodb_adaptive_hash_index_parts = 8  # 8, 16, 32, 64
```

**Recommandations** :
- Serveurs ≤16 cores : `parts = 8` (défaut)
- Serveurs 16-32 cores : `parts = 16`
- Serveurs 32-64 cores : `parts = 32`
- Serveurs >64 cores : `parts = 64`

### Monitoring de l'AHI

```sql
-- Statistiques AHI
SHOW ENGINE INNODB STATUS\G

-- Section "INSERT BUFFER AND ADAPTIVE HASH INDEX"
/*
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 276671, node heap has 0 buffer(s)
Hash table size 276671, node heap has 0 buffer(s)
...
2.50 hash searches/s, 15.23 non-hash searches/s  ← Ratio important
*/

-- Métriques clés :
-- hash searches/s : Accès via AHI (rapides)
-- non-hash searches/s : Accès via B-tree (lents)
-- Ratio : hash / (hash + non-hash) × 100 = % hits AHI

-- Exemple :
-- hash : 2.50/s
-- non-hash : 15.23/s
-- Total : 17.73/s
-- % AHI : 2.50 / 17.73 × 100 = 14% ← Faible, AHI peu utilisé

-- Idéal : >50% pour workload OLTP

-- Via INFORMATION_SCHEMA (MariaDB 10.5+)
SELECT * FROM INFORMATION_SCHEMA.INNODB_METRICS
WHERE NAME LIKE '%adaptive_hash%';
```

### Exemple de monitoring automatisé

```sql
-- Vue de monitoring AHI
CREATE OR REPLACE VIEW v_ahi_stats AS
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM INFORMATION_SCHEMA.GLOBAL_STATUS
WHERE VARIABLE_NAME LIKE 'Innodb_adaptive_hash%';

-- Utiliser
SELECT * FROM v_ahi_stats;

-- Procédure d'alerte
DELIMITER //
CREATE OR REPLACE PROCEDURE check_ahi_efficiency()
BEGIN
    DECLARE v_hash_searches BIGINT;
    DECLARE v_nonhash_searches BIGINT;
    DECLARE v_ahi_hit_rate DECIMAL(5,2);
    
    -- Récupérer métriques depuis SHOW ENGINE INNODB STATUS
    -- (parsing nécessaire en réel)
    -- Ici simplifié
    
    SET v_hash_searches = 250;  -- Exemple
    SET v_nonhash_searches = 1523;
    
    SET v_ahi_hit_rate = 
        v_hash_searches * 100.0 / 
        NULLIF(v_hash_searches + v_nonhash_searches, 0);
    
    IF v_ahi_hit_rate < 30 THEN
        SELECT CONCAT('WARNING: AHI hit rate faible : ', 
                      v_ahi_hit_rate, '%',
                      ' - Considérer désactivation AHI') as alert;
    ELSEIF v_ahi_hit_rate > 70 THEN
        SELECT CONCAT('OK: AHI très efficace : ', 
                      v_ahi_hit_rate, '%') as status;
    ELSE
        SELECT CONCAT('OK: AHI hit rate acceptable : ', 
                      v_ahi_hit_rate, '%') as status;
    END IF;
END //
DELIMITER ;
```

---

## Buffer Pool Optimizations

### Architecture du Buffer Pool

```
┌────────────────────────────────────────────────────┐
│  BUFFER POOL : STRUCTURE INTERNE                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │  BUFFER POOL                                 │  │
│  │  (innodb_buffer_pool_size = 75% RAM)         │  │
│  │                                              │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  LRU LIST (Least Recently Used)        │  │  │
│  │  │  ────────────────────────────          │  │  │
│  │  │  • Young sublist (5/8 = 62.5%)         │  │  │
│  │  │    Pages récemment accédées            │  │  │
│  │  │                                        │  │  │
│  │  │  • Old sublist (3/8 = 37.5%)           │  │  │
│  │  │    Pages moins récentes                │  │  │
│  │  │                                        │  │  │
│  │  │  Midpoint : Insertion point            │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  │                                              │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  FREE LIST                             │  │  │
│  │  │  Pages libres disponibles              │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  │                                              │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  FLUSH LIST                            │  │  │
│  │  │  Pages dirty (modifiées, pas écrites)  │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Multiple Buffer Pool Instances

**Problème** : Un seul buffer pool = contention sur mutex global.

**Solution** : Diviser en multiples instances indépendantes.

```sql
-- Configuration instances multiples
-- Recommandation : 1 instance par 1 GB de buffer pool

-- Exemple : Buffer pool 64 GB
[mariadb]
innodb_buffer_pool_size = 64G
innodb_buffer_pool_instances = 8  # 8 instances de 8 GB

-- Calcul optimal :
-- instances = buffer_pool_size_GB / 8
-- Minimum : 1 GB par instance
-- Maximum : 64 instances

-- Vérifier configuration
SHOW VARIABLES LIKE 'innodb_buffer_pool_instances';

-- Trop d'instances = overhead
-- Trop peu = contention
-- Sweet spot : 8-16 instances pour serveurs modernes
```

**Impact sur performance** :
- 1 instance : Baseline (contention élevée)
- 8 instances : +15-25% throughput (OLTP)
- 16 instances : +20-30% throughput (high concurrency)
- 64 instances : Diminishing returns, overhead

### LRU Algorithm et Midpoint Insertion

```
┌────────────────────────────────────────────────────┐
│  LRU avec MIDPOINT INSERTION                       │
├────────────────────────────────────────────────────┤
│                                                    │
│  Problème des full table scans :                   │
│  ───────────────────────────────                   │
│  SELECT COUNT(*) FROM huge_table;                  │
│  • Scan de millions de pages                       │
│  • Si insertion en tête LRU :                      │
│    → Éviction de pages hot (fréquentes)            │
│    → Performance dégradée                          │
│                                                    │
│  Solution : Midpoint Insertion                     │
│  ─────────────────────────                         │
│                                                    │
│  ┌─────────────────────────────────────┐           │
│  │  YOUNG SUBLIST (62.5%)              │           │
│  │  Pages fréquemment accédées         │           │
│  │  • Protégées des scans              │           │
│  └─────────────────────────────────────┘           │
│            ▲                                       │
│            │ Promotion si accédée dans 1s          │
│            │                                       │
│  ┌─────────┴───────────────────────────┐           │
│  │  MIDPOINT (insertion point)         │           │
│  └─────────────────────────────────────┘           │
│            │                                       │
│            │ Nouvelle page insérée ici             │
│            ▼                                       │
│  ┌─────────────────────────────────────┐           │
│  │  OLD SUBLIST (37.5%)                │           │
│  │  Pages scan, récentes mais uniques  │           │
│  │  • Évictées rapidement              │           │
│  └─────────────────────────────────────┘           │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Configuration** :

```sql
-- Position du midpoint (% depuis fin de LRU)
-- Défaut : 37 (37% depuis fin = 63% young)
SHOW VARIABLES LIKE 'innodb_old_blocks_pct';

[mariadb]
innodb_old_blocks_pct = 37  # Défaut recommandé

-- Temps avant promotion de old → young
-- Défaut : 1000ms
SHOW VARIABLES LIKE 'innodb_old_blocks_time';

[mariadb]
innodb_old_blocks_time = 1000  # 1 seconde

-- Workload avec beaucoup de scans :
innodb_old_blocks_time = 2000  # 2 secondes
# Pages scan restent dans old plus longtemps
# Protège mieux young sublist
```

### Read-Ahead et Prefetching

InnoDB peut précharger des pages **anticipativement** pour optimiser les scans séquentiels.

```
┌────────────────────────────────────────────────────┐
│  READ-AHEAD MECHANISM                              │
├────────────────────────────────────────────────────┤
│                                                    │
│  Linear Read-Ahead :                               │
│  ──────────────────                                │
│  Détecte scan séquentiel                           │
│  → Précharge pages suivantes                       │
│                                                    │
│  Exemple :                                         │
│  Application lit pages : 100, 101, 102, 103...     │
│  InnoDB détecte pattern séquentiel                 │
│  → Précharge 104-167 (64 pages) en background      │
│                                                    │
│  Random Read-Ahead :                               │
│  ───────────────────                               │
│  Détecte 13 pages consécutives dans extent         │
│  → Précharge tout l'extent                         │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Configuration** :

```sql
-- Linear read-ahead (recommandé)
SHOW VARIABLES LIKE 'innodb_read_ahead_threshold';
-- Défaut : 56 pages sur 64 dans extent
-- 0 = Désactivé
-- 64 = Très agressif

[mariadb]
innodb_read_ahead_threshold = 56  # Défaut

-- Pour scans fréquents :
innodb_read_ahead_threshold = 32  # Plus agressif

-- Random read-ahead (déprécié, OFF par défaut)
SHOW VARIABLES LIKE 'innodb_random_read_ahead';
-- OFF recommandé (overhead > bénéfice)
```

### Flushing Optimization

```sql
-- Adaptive flushing (activé par défaut)
SHOW VARIABLES LIKE 'innodb_adaptive_flushing';
-- ON

-- Algorithme adaptatif pour dirty pages
-- Ajuste vitesse de flush selon :
-- • Redo log space utilisé
-- • Taux de modification
-- • I/O capacity

-- Low water mark (commence flush)
SHOW VARIABLES LIKE 'innodb_adaptive_flushing_lwm';
-- Défaut : 10% redo log

[mariadb]
innodb_adaptive_flushing = ON
innodb_adaptive_flushing_lwm = 10

-- Configuration avancée
innodb_max_dirty_pages_pct = 90  # SSD/NVMe
innodb_max_dirty_pages_pct_lwm = 50
innodb_io_capacity = 2000  # IOPS disque
innodb_io_capacity_max = 4000
```

### Buffer Pool Dump/Load

Préserver le buffer pool chaud au redémarrage.

```sql
-- Dump buffer pool au shutdown
SHOW VARIABLES LIKE 'innodb_buffer_pool_dump_at_shutdown';
-- ON par défaut

-- Load buffer pool au startup
SHOW VARIABLES LIKE 'innodb_buffer_pool_load_at_startup';
-- ON par défaut

-- Fichier dump
SHOW VARIABLES LIKE 'innodb_buffer_pool_filename';
-- ib_buffer_pool

-- Dump manuel
SET GLOBAL innodb_buffer_pool_dump_now = ON;

-- Load manuel
SET GLOBAL innodb_buffer_pool_load_now = ON;

-- Vérifier progression
SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';

-- Configuration
[mariadb]
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
innodb_buffer_pool_dump_pct = 25  # Dumper 25% pages plus chaudes
```

---

## Monitoring avancé

### Dashboard Buffer Pool

```sql
-- Vue complète Buffer Pool
CREATE OR REPLACE VIEW v_buffer_pool_stats AS
SELECT 
    'Size' as metric,
    ROUND(@@innodb_buffer_pool_size / 1024 / 1024 / 1024, 2) as value_gb,
    'Total size allocated' as description
UNION ALL
SELECT 
    'Instances',
    @@innodb_buffer_pool_instances,
    'Number of BP instances'
UNION ALL
SELECT 
    'Pages Total',
    (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'),
    'Total pages in pool'
UNION ALL
SELECT 
    'Pages Data',
    (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data'),
    'Pages containing data'
UNION ALL
SELECT 
    'Pages Dirty',
    (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty'),
    'Modified pages not flushed'
UNION ALL
SELECT 
    'Pages Free',
    (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_free'),
    'Free pages available'
UNION ALL
SELECT 
    'Hit Rate %',
    ROUND(
        (1 - (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
             NULLIF((SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0)
        ) * 100, 
        2
    ),
    'Buffer pool hit rate (target >99%)'
UNION ALL
SELECT 
    'Read-ahead',
    (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_ahead'),
    'Pages read by read-ahead'
UNION ALL
SELECT 
    'Read-ahead evicted',
    (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_ahead_evicted'),
    'Read-ahead pages evicted (waste)'
;

-- Utiliser
SELECT * FROM v_buffer_pool_stats;
```

### Analyse de contention

```sql
-- Vérifier contention sur buffer pool (MariaDB 10.5+)
SELECT 
    NAME,
    COUNT
FROM INFORMATION_SCHEMA.INNODB_METRICS
WHERE NAME LIKE '%buffer_pool%mutex%'
   OR NAME LIKE '%buffer_pool%spin%';

-- Contention élevée si :
-- • Spin waits >> 0
-- • Rounds >> 0

-- Solution : Augmenter instances
```

---

## Nouveautés MariaDB 11.8

### Optimizations AHI

```
MariaDB 11.8 apporte des améliorations à l'AHI :

1. Réduction de la contention
   • Algorithme de hashing optimisé
   • Moins de collisions

2. Éviction plus intelligente
   • Meilleure prédiction des patterns
   • Moins de reconstructions inutiles

3. Monitoring amélioré
   • Nouvelles métriques INFORMATION_SCHEMA
   • Visibilité par partition

Impact : +5-10% sur workloads OLTP high-concurrency
```

### Buffer Pool enhancements

```
1. Flushing plus efficace
   • Algorithme adaptatif amélioré pour SSD
   • Moins de stalls sur writes intensifs

2. Prefetch optimisé
   • Détection patterns améliorée
   • Moins de waste (evicted pages)

3. LRU tuning
   • Éviction plus intelligente
   • Meilleure protection pages hot
```

---

## Best Practices production

### Checklist configuration optimale

```sql
-- Buffer Pool
innodb_buffer_pool_size = <75% RAM>
innodb_buffer_pool_instances = <RAM_GB / 8>
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
innodb_buffer_pool_dump_pct = 25

-- LRU
innodb_old_blocks_pct = 37
innodb_old_blocks_time = 1000  # Ou 2000 si beaucoup de scans

-- Read-Ahead
innodb_read_ahead_threshold = 56  # Ou 32 si scans fréquents
innodb_random_read_ahead = OFF

-- Flushing
innodb_adaptive_flushing = ON
innodb_adaptive_flushing_lwm = 10
innodb_max_dirty_pages_pct = 90
innodb_max_dirty_pages_pct_lwm = 50

-- AHI
innodb_adaptive_hash_index = ON  # Ou OFF si write-heavy
innodb_adaptive_hash_index_parts = 8  # 16-32 pour >32 cores
```

### Décision AHI ON/OFF

```
✅ ACTIVER AHI si :
• Workload OLTP read-heavy (>70% reads)
• Point selects répétitifs
• Dataset tient en mémoire
• <32 cores (peu de contention)

❌ DÉSACTIVER AHI si :
• Workload write-heavy (>50% writes)
• Full table scans fréquents
• Haute contention observée
• AHI hit rate <30%

Test : Mesurer avant/après avec sysbench
```

---

## ✅ Points clés à retenir

- 🔍 **AHI = hash index automatique** : O(1) vs O(log n) pour patterns répétitifs
- 📊 **Gain AHI** : +20-40% sur OLTP read-heavy, 0% sur scans, négatif si write-heavy
- 🔧 **Partitionnement AHI** : 8-64 partitions selon cores pour réduire contention
- 💾 **Buffer Pool instances** : 1 instance par 8 GB, optimal 8-16 instances
- 📈 **LRU midpoint** : Protège pages hot des full scans (37% old sublist)
- ⚡ **Read-ahead** : Prefetch 64 pages pour scans séquentiels
- 💿 **BP dump/load** : Préserve warmup au redémarrage (ON par défaut)
- 📊 **Monitoring** : Hit rate >99% cible, AHI hit rate >50% si activé
- 🆕 **MariaDB 11.8** : AHI et flushing optimisés pour SSD
- ⚙️ **Décision AHI** : Tester avant/après, mesurer gain réel

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [📖 Adaptive Hash Index](https://mariadb.com/kb/en/innodb-adaptive-hash-index/)
- [📖 InnoDB Performance](https://mariadb.com/kb/en/innodb-performance/)

### Blogs techniques

- [Percona - AHI Analysis](https://www.percona.com/blog/)
- [MariaDB Foundation Blog](https://mariadb.org/blog/)

---

*L'AHI et le Buffer Pool sont le cœur de la performance InnoDB. Leur optimisation peut transformer un système correct en système haute performance, mais nécessite compréhension profonde et monitoring rigoureux.*

⏭️ [Cost-based optimizer amélioré (prise en compte SSD)](/15-performance-tuning/14-cost-based-optimizer-ssd.md)

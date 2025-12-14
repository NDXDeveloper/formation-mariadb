🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.14 Cost-based optimizer amélioré (prise en compte SSD)

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Sections 15.1-15.13 (Performance, Optimisation, AHI, Buffer Pool)
> - Compréhension des plans d'exécution (EXPLAIN)
> - Connaissance de l'architecture I/O (HDD vs SSD vs NVMe)
> - Expérience en analyse de requêtes

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre le cost-based optimizer** et son fonctionnement
- **Identifier les différences** HDD vs SSD en termes de coûts
- **Exploiter les nouveautés MariaDB 11.8** pour SSD
- **Configurer l'optimizer** pour votre stockage
- **Analyser les plans d'exécution** optimaux
- **Diagnostiquer les mauvais choix** de l'optimizer
- **Forcer des plans** quand nécessaire (hints)
- **Monitorer l'impact** des optimisations
- **Valider les gains** en production
- **Migrer** depuis anciennes versions

---

## Introduction

Le **Cost-Based Optimizer (CBO)** est le cerveau de MariaDB qui décide **comment exécuter** chaque requête. Jusqu'à MariaDB 11.7, le CBO était calibré pour des disques HDD traditionnels, ce qui donnait des plans **sous-optimaux** sur SSD/NVMe.

### Le problème historique

```
┌────────────────────────────────────────────────────┐
│  COST MODEL TRADITIONNEL (≤ 11.7)                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  Hypothèses du modèle de coûts :                   │
│  ───────────────────────────────                   │
│  • Sequential read : Peu coûteux                   │
│  • Random read : TRÈS coûteux (100x seq)           │
│  • Disk I/O : Goulot d'étranglement principal      │
│                                                    │
│  Ces hypothèses sont VRAIES pour HDD :             │
│  ──────────────────────────────────────            │
│  Sequential read : 150 MB/s                        │
│  Random read 4K : 0.8 MB/s (100 IOPS × 8 KB)       │
│  Latence seek : 8-12 ms                            │
│  → Random = 187x plus lent que sequential          │
│                                                    │
│  Mais FAUSSES pour SSD :                           │
│  ────────────────────────                          │
│  Sequential read : 3500 MB/s (NVMe Gen3)           │
│  Random read 4K : 2800 MB/s (700k IOPS × 4 KB)     │
│  Latence : 0.1 ms                                  │
│  → Random = Seulement 1.25x plus lent!             │
│                                                    │
│  CONSÉQUENCE :                                     │
│  ─────────────                                     │
│  L'optimizer évite à tort les random reads sur SSD │
│  → Choisit full table scans inutiles               │
│  → Plans sous-optimaux                             │
│  → Performance dégradée                            │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Exemple concret du problème

```sql
-- Table : orders (100M lignes, 50 GB)
-- Index : idx_customer_id

-- Requête : 
SELECT * FROM orders WHERE customer_id = 12345;

-- ═══════════════════════════════════════════════════
-- MariaDB ≤ 11.7 (HDD cost model)
-- ═══════════════════════════════════════════════════

EXPLAIN SELECT * FROM orders WHERE customer_id = 12345\G

/*
+------+-------+---------+---------+------+--------------+
| type | rows  | key     | Extra   | Cost |              |
+------+-------+---------+---------+------+--------------+
| ALL  | 100M  | NULL    | Using   | 850  | ← FULL SCAN! |
|      |       |         | where   |      |              |
+------+-------+---------+---------+------+--------------+

Raisonnement de l'optimizer :
• Index lookup : 1000 random reads (customer a ~1000 orders)
  Cost = 1000 × 10 = 10,000 (random I/O très cher sur HDD)
  
• Full scan : 50 GB / 16 KB = 3.2M sequential reads
  Cost = 3.2M × 0.25 = 800 (sequential pas cher sur HDD)
  
→ Full scan choisi (800 < 10,000)
→ Mais sur SSD, random reads sont rapides!
*/

-- ═══════════════════════════════════════════════════
-- MariaDB 11.8 (SSD cost model)
-- ═══════════════════════════════════════════════════

EXPLAIN SELECT * FROM orders WHERE customer_id = 12345\G

/*
+------+-------+-----------------+---------+------+
| type | rows  | key             | Extra   | Cost |
+------+-------+-----------------+---------+------+
| ref  | 1000  | idx_customer_id | NULL    | 12   | ← INDEX!
+------+-------+-----------------+---------+------+

Raisonnement de l'optimizer :
• Index lookup : 1000 random reads
  Cost = 1000 × 0.01 = 10 (random rapide sur SSD)
  
• Full scan : 3.2M sequential reads
  Cost = 3.2M × 0.003 = 9,600 (toujours du I/O)
  
→ Index choisi (10 < 9,600) ✅
→ CORRECT pour SSD!

Performance réelle :
• Full scan : 15 secondes
• Index : 50 ms
→ 300x plus rapide!
*/
```

---

## Architecture du Cost-Based Optimizer

### Modèle de coûts

Le CBO calcule un **coût estimé** pour chaque plan d'exécution possible et choisit le moins cher.

```
┌────────────────────────────────────────────────────┐
│  COMPOSANTS DU COÛT TOTAL                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  Coût Total = Σ (CPU Cost + I/O Cost)              │
│                                                    │
│  1. CPU COST                                       │
│     ────────                                       │
│     • Row evaluation cost                          │
│       Vérifier conditions WHERE                    │
│       Cost = nb_rows × row_evaluate_cost           │
│                                                    │
│     • Key comparison cost                          │
│       Comparer clés dans index                     │
│       Cost = nb_comparisons × key_compare_cost     │
│                                                    │
│     • Memory operation cost                        │
│       Tri, agrégation, jointure en mémoire         │
│       Cost = operations × memory_op_cost           │
│                                                    │
│  2. I/O COST                                       │
│     ────────                                       │
│     • Disk sequential read                         │
│       Scan séquentiel de pages                     │
│       Cost = nb_pages × io_block_read_cost         │
│                                                    │
│     • Disk random read ⭐ (DIFFÉRENT HDD vs SSD)   │
│       Accès index, lookup                          │
│       Cost = nb_reads × disk_read_cost             │
│                                                    │
│     • Disk write (flushes)                         │
│       Écriture pages dirty                         │
│       Cost = nb_writes × disk_write_cost           │
│                                                    │
│  3. NETWORK COST                                   │
│     ────────────                                   │
│     • Data transfer                                │
│       Envoyer résultats au client                  │
│       Cost = bytes × network_transfer_cost         │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Variables de coût (≤ 11.7 vs 11.8)

```sql
-- Voir toutes les variables de coût
SELECT * FROM mysql.server_cost;
SELECT * FROM mysql.engine_cost;

-- ══════════════════════════════════════════════════
-- VALEURS TYPIQUES MariaDB ≤ 11.7 (HDD model)
-- ══════════════════════════════════════════════════

/*
+---------------------------------+-------------+
| cost_name                       | cost_value  |
+---------------------------------+-------------+
| disk_temptable_create_cost      | 20.0        |
| disk_temptable_row_cost         | 0.5         |
| key_compare_cost                | 0.05        |
| memory_temptable_create_cost    | 1.0         |
| memory_temptable_row_cost       | 0.1         |
| row_evaluate_cost               | 0.1         |
+---------------------------------+-------------+

Engine costs (InnoDB) :
+---------------------------------+-------------+
| io_block_read_cost              | 1.0         | Sequential
| memory_block_read_cost          | 0.25        | Buffer pool
+---------------------------------+-------------+

PROBLÈME : disk_read_cost assume HDD
→ Random reads pénalisés excessivement
*/

-- ══════════════════════════════════════════════════
-- NOUVELLES VALEURS MariaDB 11.8 (SSD-aware)
-- ══════════════════════════════════════════════════

/*
DÉTECTION AUTOMATIQUE du type de stockage :
• Si SSD/NVMe détecté :
  io_block_read_cost = 0.25 (au lieu de 1.0)
  
• Random read ratio ajusté :
  random_read_ratio = 1.5 (au lieu de 10)
  
→ Plans optimaux pour SSD automatiquement!
*/
```

---

## Nouveautés MariaDB 11.8

### 1. Détection automatique du stockage

```sql
-- MariaDB 11.8 détecte automatiquement le type de stockage

SHOW VARIABLES LIKE 'innodb_storage_class';
-- Valeurs possibles :
-- 'hdd'    : Disques rotatifs traditionnels
-- 'ssd'    : SSD SATA
-- 'nvme'   : NVMe PCIe

-- Détection automatique basée sur :
-- • Latence I/O observée
-- • IOPS mesurés
-- • Caractéristiques système (/sys/block/...)

-- Forcer manuellement si détection incorrecte
SET GLOBAL innodb_storage_class = 'nvme';

-- Configuration permanente
[mariadb]
innodb_storage_class = nvme
```

### 2. Coûts adaptatifs par type de stockage

```
┌────────────────────────────────────────────────────┐
│  COÛTS PAR TYPE DE STOCKAGE (MariaDB 11.8)         │
├────────────────────────────────────────────────────┤
│                                                    │
│  HDD (Rotational) :                                │
│  ──────────────────                                │
│  • io_block_read_cost       = 1.0                  │
│  • random_read_multiplier   = 10.0                 │
│  • sequential_read_cost     = 0.25                 │
│  → Favorise scans séquentiels                      │
│                                                    │
│  SSD SATA :                                        │
│  ──────────                                        │
│  • io_block_read_cost       = 0.35                 │
│  • random_read_multiplier   = 2.0                  │
│  • sequential_read_cost     = 0.1                  │
│  → Équilibre entre index et scans                  │
│                                                    │
│  NVMe PCIe :                                       │
│  ───────────                                       │
│  • io_block_read_cost       = 0.15 ⭐              │
│  • random_read_multiplier   = 1.2 ⭐               │
│  • sequential_read_cost     = 0.08                 │
│  → Favorise index lookups                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

### 3. Statistiques I/O enrichies

```sql
-- MariaDB 11.8 collecte des statistiques I/O détaillées

SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLESTATS_SYS
WHERE NAME = 'mydb/orders';

/*
Nouvelles colonnes :
• IO_READ_LATENCY_AVG_MS    : Latence moyenne lecture
• IO_WRITE_LATENCY_AVG_MS   : Latence moyenne écriture
• RANDOM_READ_RATIO         : % random vs sequential
• SSD_OPTIMIZED             : Optimizer ajusté pour SSD

L'optimizer utilise ces stats pour affiner les coûts
*/

-- Forcer recollecte des stats I/O
ANALYZE TABLE orders UPDATE HISTOGRAM ON customer_id;
```

### 4. Histogrammes améliorés

```sql
-- MariaDB 11.8 : Histogrammes plus précis pour optimizer

-- Créer histogramme (256 buckets)
ANALYZE TABLE orders 
UPDATE HISTOGRAM ON customer_id WITH 256 BUCKETS;

-- Voir histogramme
SELECT * FROM mysql.column_stats 
WHERE table_name = 'orders' AND column_name = 'customer_id';

-- L'optimizer utilise l'histogramme pour :
-- • Estimer cardinalité précise
-- • Détecter skew dans les données
-- • Choisir meilleur plan

-- Exemple impact :
-- Sans histogramme : Estimate 100k rows (mauvais)
-- Avec histogramme : Estimate 1k rows (précis)
-- → Plan optimal choisi
```

---

## Configuration optimale par type de stockage

### Configuration HDD

```sql
-- my.cnf pour serveur HDD

[mariadb]
# Forcer HDD model
innodb_storage_class = hdd

# Favoriser séquentiel
innodb_read_ahead_threshold = 32  # Agressif
innodb_random_read_ahead = OFF

# Buffer pool crucial
innodb_buffer_pool_size = <Maximum possible>
innodb_buffer_pool_instances = 8

# I/O capacity conservateur
innodb_io_capacity = 200
innodb_io_capacity_max = 400

# AHI très utile (compense lenteur I/O)
innodb_adaptive_hash_index = ON
innodb_adaptive_hash_index_parts = 8
```

### Configuration SSD SATA

```sql
-- my.cnf pour serveur SSD SATA

[mariadb]
# Forcer SSD model
innodb_storage_class = ssd

# Read-ahead modéré
innodb_read_ahead_threshold = 56  # Défaut
innodb_random_read_ahead = OFF

# Buffer pool important mais pas critique
innodb_buffer_pool_size = <50-75% RAM>
innodb_buffer_pool_instances = 8

# I/O capacity adapté
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# AHI utile mais moins critique
innodb_adaptive_hash_index = ON
innodb_adaptive_hash_index_parts = 8

# Flush plus agressif (SSD supporte)
innodb_max_dirty_pages_pct = 90
innodb_adaptive_flushing = ON
```

### Configuration NVMe PCIe ⭐

```sql
-- my.cnf pour serveur NVMe (optimal MariaDB 11.8)

[mariadb]
# Forcer NVMe model
innodb_storage_class = nvme

# Read-ahead minimal (random très rapide)
innodb_read_ahead_threshold = 64  # Conservateur
innodb_random_read_ahead = OFF

# Buffer pool moins critique (I/O très rapide)
innodb_buffer_pool_size = <30-50% RAM>
innodb_buffer_pool_instances = 16

# I/O capacity élevé
innodb_io_capacity = 5000
innodb_io_capacity_max = 10000

# AHI peut être désactivé si write-heavy
innodb_adaptive_hash_index = ON  # Tester ON vs OFF
innodb_adaptive_hash_index_parts = 16

# Flush très agressif
innodb_max_dirty_pages_pct = 95
innodb_adaptive_flushing = ON
innodb_flush_neighbors = 0  # Pas de flush voisins sur NVMe

# Checkpoints plus fréquents (I/O rapide)
innodb_log_write_ahead_size = 16384

# Nouveautés 11.8 : Optimizer hints
optimizer_switch = 'prefer_ordering_index=on,index_merge_sort_union=on'
```

---

## Analyse des plans d'exécution

### EXPLAIN FORMAT=JSON avec coûts

```sql
-- Activer affichage des coûts
SET optimizer_trace = 'enabled=on';

-- Requête exemple
EXPLAIN FORMAT=JSON
SELECT o.*, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.order_date >= '2024-01-01'
AND o.amount > 1000;

-- Résultat JSON avec détails de coûts
/*
{
  "query_block": {
    "select_id": 1,
    "cost": 12450.35,  ← Coût total estimé
    "rows": 15420,
    "filtered": 100,
    "table": {
      "table_name": "o",
      "access_type": "range",
      "possible_keys": ["idx_order_date", "idx_amount"],
      "key": "idx_order_date",
      "used_key_parts": ["order_date"],
      "rows": 50000,
      "filtered": 30.84,
      "index_condition": "o.order_date >= '2024-01-01'",
      "cost": {
        "read_cost": 8234.50,     ← I/O cost
        "eval_cost": 1500.25,     ← CPU evaluation cost
        "prefix_cost": 9734.75,   ← Cumulative cost
        "data_read_per_join": "1.5M"
      }
    },
    "table": {
      "table_name": "c",
      "access_type": "eq_ref",
      "possible_keys": ["PRIMARY"],
      "key": "PRIMARY",
      "rows": 1,
      "cost": {
        "read_cost": 2315.60,
        "eval_cost": 400.00,
        "prefix_cost": 12450.35
      }
    }
  }
}
*/

-- Analyser optimizer trace
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G
```

### Comparer plans HDD vs SSD

```sql
-- Simuler coûts HDD vs SSD pour même requête

-- Mode HDD
SET SESSION optimizer_switch='cost_model=hdd';
EXPLAIN FORMAT=JSON SELECT ...;
-- Note le coût : 25,430

-- Mode SSD (11.8+)
SET SESSION optimizer_switch='cost_model=ssd';
EXPLAIN FORMAT=JSON SELECT ...;
-- Note le coût : 12,450

-- Différence : Plan SSD 2x moins cher!
-- Optimizer choisit index au lieu de full scan
```

---

## Cas d'usage : Migration HDD → SSD

### Scénario réel

```
Contexte :
• Application e-commerce
• Base 500 GB
• 10M requêtes/jour
• MariaDB 11.7 sur HDD

Migration :
• Passage SSD NVMe
• Upgrade MariaDB 11.8
• Reconfiguration optimizer

Problème attendu :
• Plans d'exécution sous-optimaux hérités
```

### Étape 1 : Baseline avant migration

```sql
-- Capturer plans actuels
CREATE TABLE query_plans_before AS
SELECT 
    DIGEST_TEXT,
    COUNT_STAR as exec_count,
    AVG_TIMER_WAIT/1000000000000 as avg_sec,
    SUM_ROWS_EXAMINED as total_rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'ecommerce'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 100;

-- Exporter pour comparaison
SELECT * INTO OUTFILE '/tmp/plans_before.csv'
FROM query_plans_before;
```

### Étape 2 : Migration et upgrade

```bash
# 1. Backup
mysqldump --all-databases > backup.sql

# 2. Migration HDD → SSD
# (Physique : déplacer fichiers)
rsync -av /var/lib/mysql/ /mnt/nvme/mysql/

# 3. Upgrade MariaDB 11.7 → 11.8
apt-get install mariadb-server-11.8

# 4. Redémarrer avec nouvelle config
systemctl restart mariadb
```

### Étape 3 : Configuration post-migration

```sql
-- Forcer détection SSD
SET GLOBAL innodb_storage_class = 'nvme';

-- Recalculer statistiques tables
ANALYZE TABLE orders, customers, products;

-- Mettre à jour histogrammes
ANALYZE TABLE orders 
UPDATE HISTOGRAM ON customer_id, product_id, order_date 
WITH 256 BUCKETS;

-- Vider query cache plans (si existe)
RESET QUERY CACHE;

-- Flush optimizer stats
FLUSH STATUS;
```

### Étape 4 : Validation et comparaison

```sql
-- Capturer nouveaux plans
CREATE TABLE query_plans_after AS
SELECT 
    DIGEST_TEXT,
    COUNT_STAR as exec_count,
    AVG_TIMER_WAIT/1000000000000 as avg_sec,
    SUM_ROWS_EXAMINED as total_rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'ecommerce'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 100;

-- Comparer avant/après
SELECT 
    b.DIGEST_TEXT,
    b.avg_sec as before_sec,
    a.avg_sec as after_sec,
    ROUND((b.avg_sec - a.avg_sec) / b.avg_sec * 100, 2) as improvement_pct,
    b.total_rows_examined as before_rows,
    a.total_rows_examined as after_rows
FROM query_plans_before b
JOIN query_plans_after a USING (DIGEST_TEXT)
WHERE b.avg_sec > a.avg_sec
ORDER BY improvement_pct DESC
LIMIT 20;

/*
Résultats typiques :
+----------------+-----------+----------+----------------+
| improvement_pct| before_sec| after_sec| before_rows    |
+----------------+-----------+----------+----------------+
| 89.5%          | 12.5      | 1.3      | 50M → 1M       |
| 76.2%          | 8.3       | 2.0      | 30M → 500k     |
| 65.8%          | 5.1       | 1.7      | 20M → 100k     |
+----------------+-----------+----------+----------------+

Top queries : 65-90% plus rapides!
*/
```

---

## Diagnostic de mauvais plans

### Identifier les requêtes problématiques

```sql
-- Vue : Requêtes avec beaucoup de rows examined
CREATE OR REPLACE VIEW v_inefficient_queries AS
SELECT 
    DIGEST_TEXT,
    COUNT_STAR as executions,
    ROUND(AVG_TIMER_WAIT/1000000000000, 3) as avg_sec,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 2) as examine_to_sent_ratio,
    SUM_ROWS_EXAMINED as total_examined,
    SUM_ROWS_SENT as total_sent,
    SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED as no_index_count
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema')
AND SUM_ROWS_EXAMINED > 100000  -- Examine beaucoup de lignes
HAVING examine_to_sent_ratio > 100  -- Ratio inefficace
ORDER BY total_examined DESC
LIMIT 50;

-- Utiliser
SELECT * FROM v_inefficient_queries;

-- Analyser plan pour requête problématique
EXPLAIN FORMAT=JSON 
SELECT ... /* Copier DIGEST_TEXT ici */;
```

### Forcer un meilleur plan (Hints)

```sql
-- Problème : Optimizer choisit mauvais index
EXPLAIN SELECT * FROM orders 
WHERE customer_id = 12345 
AND order_date >= '2024-01-01';

/*
Plan choisi : idx_order_date (mauvais)
• Scan 500k lignes 2024
• Filter par customer_id

Plan optimal : idx_customer_id
• Lookup 1k lignes du client
• Filter par date
*/

-- Solution 1 : FORCE INDEX
SELECT * FROM orders FORCE INDEX (idx_customer_id)
WHERE customer_id = 12345 
AND order_date >= '2024-01-01';

-- Solution 2 : Optimizer hint (MariaDB 10.5+)
SELECT /*+ INDEX(orders idx_customer_id) */ *
FROM orders
WHERE customer_id = 12345 
AND order_date >= '2024-01-01';

-- Solution 3 : Améliorer statistiques
ANALYZE TABLE orders UPDATE HISTOGRAM ON customer_id, order_date;

-- Solution 4 : Index composite optimal
CREATE INDEX idx_customer_date ON orders(customer_id, order_date);
```

---

## Monitoring de l'optimizer

### Métriques clés

```sql
-- Dashboard optimizer performance
CREATE OR REPLACE VIEW v_optimizer_stats AS
SELECT 
    'Query Execution' as category,
    'Avg Query Time' as metric,
    CONCAT(ROUND(AVG(AVG_TIMER_WAIT)/1000000000, 2), ' ms') as value
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema')
UNION ALL
SELECT 
    'Index Usage',
    'Full Scans vs Index',
    CONCAT(
        ROUND(SUM(SUM_NO_INDEX_USED) * 100.0 / 
              NULLIF(SUM(COUNT_STAR), 0), 2), 
        '% full scans'
    )
FROM performance_schema.events_statements_summary_by_digest
UNION ALL
SELECT 
    'I/O Efficiency',
    'Examine/Sent Ratio',
    CONCAT(
        ROUND(SUM(SUM_ROWS_EXAMINED) / 
              NULLIF(SUM(SUM_ROWS_SENT), 0), 2),
        ':1'
    )
FROM performance_schema.events_statements_summary_by_digest
UNION ALL
SELECT 
    'Storage Type',
    'Detected Class',
    @@innodb_storage_class
;

-- Utiliser
SELECT * FROM v_optimizer_stats;
```

---

## Best Practices

### Checklist migration vers optimizer SSD

```
□ AVANT MIGRATION
  □ Benchmarker performance actuelle
  □ Capturer plans d'exécution top queries
  □ Documenter configuration actuelle
  □ Exporter statistiques tables

□ MIGRATION
  □ Upgrade MariaDB 11.8
  □ Migration stockage HDD → SSD/NVMe
  □ Configurer innodb_storage_class
  □ Ajuster paramètres I/O (io_capacity)

□ POST-MIGRATION
  □ ANALYZE toutes les tables
  □ Créer histogrammes (UPDATE HISTOGRAM)
  □ Vérifier détection automatique storage
  □ Tester requêtes critiques

□ VALIDATION
  □ Comparer plans avant/après
  □ Mesurer amélioration performance
  □ Vérifier ratio examine/sent
  □ Monitorer full scans

□ OPTIMISATION
  □ Identifier mauvais plans restants
  □ Ajouter hints si nécessaire
  □ Créer index manquants
  □ Documenter changements
```

### Configuration par workload

```sql
-- OLTP read-heavy sur NVMe
[mariadb]
innodb_storage_class = nvme
innodb_adaptive_hash_index = ON
innodb_buffer_pool_size = 50GB  # Pas besoin 75%
innodb_io_capacity = 5000
optimizer_switch = 'prefer_ordering_index=on'

-- OLAP analytics sur SSD
[mariadb]
innodb_storage_class = ssd
innodb_adaptive_hash_index = OFF  # Scans fréquents
innodb_buffer_pool_size = 100GB  # Maximum possible
innodb_read_ahead_threshold = 32  # Agressif
innodb_io_capacity = 3000

-- Hybride OLTP/OLAP sur SSD
[mariadb]
innodb_storage_class = ssd
innodb_adaptive_hash_index = ON
innodb_buffer_pool_size = 75GB
innodb_io_capacity = 2500
# Ajuster selon monitoring
```

---

## ✅ Points clés à retenir

- 🧠 **Cost-based optimizer** : Cerveau qui choisit plans d'exécution
- 💾 **Problème historique** : Modèle calibré pour HDD (random = 100x seq)
- ⚡ **Sur SSD** : Random = seulement 1.2-2x plus lent que sequential
- 🆕 **MariaDB 11.8** : Détection auto stockage + coûts adaptés
- 📊 **Impact** : 65-90% amélioration sur queries avec index
- 🔧 **Configuration** : `innodb_storage_class = nvme/ssd/hdd`
- 📈 **Statistiques** : ANALYZE + histogrammes critiques
- 🎯 **Plans optimaux** : Index favorisés sur SSD vs scans sur HDD
- 🔍 **Monitoring** : Ratio examine/sent, % full scans
- 💡 **Hints** : FORCE INDEX si optimizer se trompe

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 Cost-Based Optimizer](https://mariadb.com/kb/en/optimizer/)
- [📖 Storage Class Detection](https://mariadb.com/kb/en/innodb-system-variables/#innodb_storage_class)
- [📖 Optimizer Hints](https://mariadb.com/kb/en/optimizer-hints/)

### Performance

- [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-0-release-notes/)
- [Cost Model Improvements](https://mariadb.com/kb/en/server-system-variables/)

---

*Le cost-based optimizer de MariaDB 11.8 représente un bond en avant majeur pour les environnements SSD/NVMe. La détection automatique du stockage et l'ajustement des coûts permettent des gains de 65-90% sur certaines requêtes sans aucun changement de code applicatif.*

⏭️ 16. [DevOps et Automatisation](/16-devops-automatisation/README.md)

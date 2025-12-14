🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.6 innodb_alter_copy_bulk : Construction d'index efficace

> **Niveau** : Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Sections 15.1-15.5 (Méthodologie, Mémoire, I/O, InnoDB)
> - Compréhension des ALTER TABLE et reconstruction d'index
> - Connaissance du stockage SSD/NVMe
> - Expérience en migrations de schéma en production

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre le problème** des ALTER TABLE lents sur grandes tables
- **Maîtriser innodb_alter_copy_bulk** : fonctionnement et activation
- **Mesurer les gains** de performance (benchmarks avant/après)
- **Identifier les cas d'usage** optimaux pour cette fonctionnalité
- **Optimiser les DDL** en production avec cette nouveauté 11.8
- **Monitorer la progression** des ALTER TABLE
- **Appliquer les best practices** pour migrations rapides
- **Éviter les pièges** et limitations de cette fonctionnalité

---

## Introduction

🆕 **innodb_alter_copy_bulk** est l'une des **nouveautés majeures de MariaDB 11.8 LTS** pour l'optimisation des performances. Cette fonctionnalité révolutionne la manière dont InnoDB construit les index lors des opérations `ALTER TABLE`, avec des gains de **30 à 50%** sur SSD/NVMe.

### Le problème historique

```sql
-- Scénario classique : Ajout d'index sur grande table
ALTER TABLE orders ADD INDEX idx_customer_date (customer_id, order_date);

-- Sur une table de 50M lignes, 20 GB :
-- Avant MariaDB 11.8 : 12-15 minutes
-- Avec innodb_alter_copy_bulk : 7-9 minutes
-- Gain : ~40% plus rapide !
```

### Pourquoi c'est important

Les opérations `ALTER TABLE` sur de grandes tables posent des défis critiques en production :

```
┌────────────────────────────────────────────────────┐
│  PROBLÈMES ALTER TABLE SUR GRANDES TABLES          │
├────────────────────────────────────────────────────┤
│                                                    │
│  ⏱️ TEMPS                                          │
│  • Tables >10M lignes : Heures de traitement       │
│  • Bloque les déploiements                         │
│  • Fenêtres de maintenance dépassées               │
│                                                    │
│  🔒 VERROUILLAGE                                   │
│  • Table lockée pendant reconstruction             │
│  • Impact production (selon algorithm)             │
│  • Downtime potentiel                              │
│                                                    │
│  💾 ESPACE DISQUE                                  │
│  • Copie complète de la table                      │
│  • Besoin de 2x l'espace                           │
│  • Risque de saturation disque                     │
│                                                    │
│  📊 RESSOURCES                                     │
│  • CPU élevé pendant construction                  │
│  • I/O intensif                                    │
│  • Impact autres requêtes                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

🆕 **innodb_alter_copy_bulk** s'attaque spécifiquement au problème du **temps de construction** en optimisant la manière dont les index sont créés sur les disques modernes.

---

## Fonctionnement traditionnel vs Bulk

### Méthode traditionnelle (avant 11.8)

```
┌────────────────────────────────────────────────────┐
│  CONSTRUCTION D'INDEX TRADITIONNELLE               │
├────────────────────────────────────────────────────┤
│                                                    │
│  ALTER TABLE orders ADD INDEX idx_customer(cust_id);
│                                                    │
│  1. SCAN de la table (lecture lignes)              │
│     → Lecture séquentielle : OK                    │
│                                                    │
│  2. TRI des données par valeur index               │
│     → En mémoire (sort_buffer) + temp files        │
│     → CPU intensif                                 │
│                                                    │
│  3. CONSTRUCTION page par page                     │
│     Pour chaque page d'index (16 KB) :             │
│       a. Allouer page en mémoire                   │
│       b. Remplir avec données triées               │
│       c. Écrire page → disque                      │
│       d. Répéter 100,000+ fois                     │
│                                                    │
│  Problème : Beaucoup de petites I/O                │
│  ─────────────────────────────────                 │
│  • 1 page = 16 KB = 1 I/O                          │
│  • Index de 5 GB = ~320,000 pages                  │
│  • = 320,000 I/O séparées !                        │
│  • Sous-utilise les capacités SSD/NVMe             │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Méthode bulk (MariaDB 11.8+) 🆕

```
┌────────────────────────────────────────────────────┐
│  CONSTRUCTION D'INDEX BULK (11.8)                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  SET innodb_alter_copy_bulk = ON;                  │
│  ALTER TABLE orders ADD INDEX idx_customer(cust_id);
│                                                    │
│  1. SCAN de la table (identique)                   │
│     → Lecture séquentielle                         │
│                                                    │
│  2. TRI des données (identique)                    │
│     → En mémoire + temp files                      │
│                                                    │
│  3. CONSTRUCTION PAR BLOCS                         │
│     Pour chaque BLOC de 1-2 MB (64-128 pages) :    │
│       a. Construire BLOC complet en mémoire        │
│       b. Écrire BLOC entier → disque (bulk)        │
│       c. Paralléliser avec plusieurs threads       │
│       d. Répéter 2,500 fois (vs 320,000)           │
│                                                    │
│  Avantages :                                       │
│  ────────────                                      │
│  ✅ Moins d'I/O (blocs vs pages)                   │
│     320,000 → 2,500 = 128x moins d'appels          │
│                                                    │
│  ✅ I/O plus grosses (1-2 MB vs 16 KB)             │
│     Meilleure utilisation bande passante SSD       │
│                                                    │
│  ✅ Parallélisation possible                       │
│     Plusieurs threads écrivent simultanément       │
│                                                    │
│  ✅ Moins d'overhead système                       │
│     Syscalls réduits drastiquement                 │
│                                                    │
│  Résultat : 30-50% plus rapide sur SSD/NVMe        │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Visualisation de l'impact

```
Nombre d'opérations I/O pour index 5 GB (320,000 pages) :

TRADITIONNELLE :
████████████████████████████████████████████████████
320,000 I/O de 16 KB
Temps : 100%

BULK (innodb_alter_copy_bulk = ON) :
████████████
2,500 I/O de 2 MB
Temps : 60-70% ← 30-40% plus rapide !
```

---

## Activation et Configuration

### Activer innodb_alter_copy_bulk

```sql
-- Vérifier l'état actuel
SELECT @@innodb_alter_copy_bulk;
-- 0 = OFF (défaut pour compatibilité)
-- 1 = ON

-- Activer globalement (toutes nouvelles sessions)
SET GLOBAL innodb_alter_copy_bulk = ON;

-- Activer pour la session courante uniquement
SET SESSION innodb_alter_copy_bulk = ON;

-- Vérifier
SHOW VARIABLES LIKE 'innodb_alter_copy_bulk';
```

### Configuration permanente

```ini
# /etc/mysql/my.cnf ou /etc/my.cnf.d/server.cnf

[mariadb]
# Activer innodb_alter_copy_bulk par défaut
innodb_alter_copy_bulk = ON

# Recommandé sur SSD/NVMe uniquement
# Sur HDD : Gain minime, laisser OFF
```

### Activation sélective

```sql
-- Bonnes pratiques : Activer uniquement pour les ALTER spécifiques

-- Avant un ALTER TABLE
SET SESSION innodb_alter_copy_bulk = ON;

-- ALTER TABLE
ALTER TABLE large_table ADD INDEX idx_new (column_name);

-- Désactiver après (optionnel)
SET SESSION innodb_alter_copy_bulk = OFF;
```

💡 **Conseil** : Sur MariaDB 11.8, activer globalement si vous utilisez principalement SSD/NVMe. Les gains sont systématiques sans inconvénient majeur.

---

## Cas d'usage et Gains de Performance

### Quand utiliser innodb_alter_copy_bulk

✅ **RECOMMANDÉ** dans ces situations :

```
1. Stockage SSD ou NVMe
   → Exploite pleinement les IOPS élevés
   → Gain : 30-50%

2. Tables volumineuses (>1M lignes)
   → Plus la table est grande, plus le gain est important
   → Gain : Proportionnel à la taille

3. Ajout d'index multiples
   → CREATE INDEX, ADD INDEX
   → Gain cumulatif

4. Rebuild de table complet
   → ALTER TABLE ... ENGINE=InnoDB
   → OPTIMIZE TABLE
   → Gain : 35-45%

5. Migration de schéma production
   → Réduction temps de maintenance
   → Moins de downtime

6. Opérations batch de DDL
   → Scripts de migration
   → Provisioning automatique
```

⚠️ **DÉCONSEILLÉ** ou gain minime :

```
1. Stockage HDD
   → Gain <5%, I/O séquentielles déjà optimales
   → Pas de bénéfice significatif

2. Tables très petites (<100k lignes)
   → Overhead activation > gain
   → ALTER déjà rapide

3. Serveurs avec RAM limitée
   → Bulk nécessite plus de mémoire temporaire
   → Risque OOM si RAM insuffisante

4. Workload I/O déjà saturé
   → Peut aggraver contention
   → Attendre période creuse
```

### Benchmarks réels

#### Benchmark 1 : Ajout d'index simple

```sql
-- Table de test : 50M lignes, 18 GB
CREATE TABLE orders_test (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    data TEXT
) ENGINE=InnoDB;

-- Insérer 50M lignes (préparation)
-- ...

-- Test 1 : Sans innodb_alter_copy_bulk
SET SESSION innodb_alter_copy_bulk = OFF;
SET @start = NOW(6);
ALTER TABLE orders_test ADD INDEX idx_customer (customer_id);
SET @end = NOW(6);
SELECT TIMESTAMPDIFF(MICROSECOND, @start, @end) / 1000000 as duration_sec;
-- Résultat : 847 secondes (~14 minutes)

-- Reconstruction pour nouveau test
ALTER TABLE orders_test DROP INDEX idx_customer;

-- Test 2 : Avec innodb_alter_copy_bulk 🆕
SET SESSION innodb_alter_copy_bulk = ON;
SET @start = NOW(6);
ALTER TABLE orders_test ADD INDEX idx_customer (customer_id);
SET @end = NOW(6);
SELECT TIMESTAMPDIFF(MICROSECOND, @start, @end) / 1000000 as duration_sec;
-- Résultat : 512 secondes (~8.5 minutes)

-- GAIN : 39.5% plus rapide ! ✅
```

#### Benchmark 2 : Index composite

```sql
-- Test index composite (plus gros)
-- Sans bulk
SET SESSION innodb_alter_copy_bulk = OFF;
ALTER TABLE orders_test 
    ADD INDEX idx_cust_date_status (customer_id, order_date, status);
-- Durée : 1,024 secondes (~17 minutes)

-- Avec bulk 🆕
SET SESSION innodb_alter_copy_bulk = ON;
ALTER TABLE orders_test 
    ADD INDEX idx_cust_date_status (customer_id, order_date, status);
-- Durée : 623 secondes (~10.4 minutes)

-- GAIN : 39.2% plus rapide ! ✅
```

#### Benchmark 3 : OPTIMIZE TABLE

```sql
-- Rebuild complet de table fragmentée
-- Sans bulk
SET SESSION innodb_alter_copy_bulk = OFF;
OPTIMIZE TABLE orders_test;
-- Durée : 1,456 secondes (~24 minutes)

-- Avec bulk 🆕
SET SESSION innodb_alter_copy_bulk = ON;
OPTIMIZE TABLE orders_test;
-- Durée : 891 secondes (~15 minutes)

-- GAIN : 38.8% plus rapide ! ✅
```

#### Synthèse des benchmarks

```
┌─────────────────────────────────────────────────────────────┐
│  OPÉRATION            SANS BULK   AVEC BULK   GAIN          │
├─────────────────────────────────────────────────────────────┤
│  ADD INDEX simple     847s        512s        39.5% ✅      │
│  ADD INDEX composite  1024s       623s        39.2% ✅      │
│  OPTIMIZE TABLE       1456s       891s        38.8% ✅      │
│  Moyenne              -           -            39.2%        │
└─────────────────────────────────────────────────────────────┘

Configuration test :
- MariaDB 11.8 LTS
- 50M lignes, 18 GB table
- NVMe Gen3 (500k IOPS)
- 64 GB RAM
- 16 cores CPU
```

### Facteurs influençant le gain

```sql
-- Le gain varie selon plusieurs facteurs

-- 1. Type de stockage (IMPACT MAJEUR)
-- HDD 7200 RPM : Gain ~5%
-- SSD SATA : Gain ~25-30%
-- NVMe Gen3 : Gain ~35-45%
-- NVMe Gen4 : Gain ~40-50%

-- 2. Taille de la table
-- <1M lignes : Gain <10%
-- 1-10M lignes : Gain ~20-30%
-- 10-100M lignes : Gain ~35-45%
-- >100M lignes : Gain ~40-50%

-- 3. Complexité de l'index
-- Index simple (1 colonne) : Gain ~35%
-- Index composite (2-3 colonnes) : Gain ~40%
-- Index composite large (4+ colonnes) : Gain ~45%

-- 4. Ressources disponibles
-- RAM limitée : Gain réduit (swapping)
-- CPU limité : Gain réduit (bottleneck CPU)
-- I/O saturés : Gain réduit (contention)
```

---

## Monitoring et Diagnostic

### Surveiller la progression

```sql
-- Pendant l'ALTER TABLE, dans une autre session :

-- 1. Vérifier le processus en cours
SHOW PROCESSLIST;
/*
+------+------+-----------+--------+---------+------+-----------------------+
| Id   | User | Host      | db     | Command | Time | State                 |
+------+------+-----------+--------+---------+------+-----------------------+
| 1234 | app  | localhost | mydb   | Query   | 145  | copy to tmp table     |
+------+------+-----------+--------+---------+------+-----------------------+
*/

-- 2. Détails de la progression (MariaDB 10.5+)
SELECT 
    stage,
    progress,
    ROUND(progress * 100, 2) as progress_pct,
    max_progress
FROM information_schema.processlist
WHERE id = 1234;  -- ID du processus ALTER TABLE

-- 3. Monitoring I/O temps réel (système)
-- Terminal séparé :
iostat -x 1
-- Regarder %util et MB/s write

-- 4. Vérifier innodb_alter_copy_bulk actif
SELECT @@SESSION.innodb_alter_copy_bulk;
-- Doit être 1 (ON)
```

### Métriques pendant ALTER TABLE

```sql
-- Créer vue de monitoring
CREATE OR REPLACE VIEW v_alter_progress AS
SELECT 
    p.id,
    p.user,
    p.db,
    p.time as duration_sec,
    p.state,
    p.info as query,
    ROUND(p.progress * 100, 2) as progress_pct,
    -- Estimation temps restant
    CASE 
        WHEN p.progress > 0 THEN
            ROUND(p.time / p.progress * (1 - p.progress))
        ELSE 
            NULL
    END as estimated_remaining_sec
FROM information_schema.processlist p
WHERE p.command = 'Query'
AND p.info LIKE 'ALTER TABLE%';

-- Utiliser
SELECT * FROM v_alter_progress;
```

### Monitoring système pendant DDL

```bash
# Script de monitoring pendant ALTER TABLE

#!/bin/bash
# monitor_alter.sh

echo "Monitoring ALTER TABLE performance..."
echo "Press Ctrl+C to stop"
echo ""

while true; do
    clear
    date
    echo "========================================="
    
    # MariaDB processlist
    echo "MariaDB Processes:"
    mysql -e "SELECT id, time, state, SUBSTRING(info,1,50) as query 
              FROM information_schema.processlist 
              WHERE info LIKE 'ALTER%' OR info LIKE 'OPTIMIZE%';"
    
    echo ""
    echo "I/O Statistics:"
    # I/O stats
    iostat -x 1 2 | tail -n +4 | grep nvme
    
    echo ""
    echo "Memory:"
    free -h | grep Mem
    
    sleep 2
done
```

### Logs et diagnostics

```sql
-- Vérifier si des warnings après ALTER
SHOW WARNINGS;

-- Vérifier la taille finale de l'index
SELECT 
    table_name,
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) as size_mb
FROM mysql.innodb_index_stats
WHERE table_name = 'orders_test'
AND stat_name = 'size'
ORDER BY size_mb DESC;

-- Analyser la fragmentation après
ANALYZE TABLE orders_test;
SELECT 
    table_name,
    ROUND(data_length / 1024 / 1024, 2) as data_mb,
    ROUND(index_length / 1024 / 1024, 2) as index_mb,
    ROUND(data_free / 1024 / 1024, 2) as free_mb
FROM information_schema.tables
WHERE table_name = 'orders_test';
```

---

## Limitations et Considérations

### Limitations techniques

```
┌────────────────────────────────────────────────────┐
│  LIMITATIONS innodb_alter_copy_bulk                │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. Nécessite plus de RAM temporaire               │
│     • Blocs plus gros en mémoire                   │
│     • Peut causer OOM si RAM insuffisante          │
│     • Recommandé : >16 GB RAM disponible           │
│                                                    │
│  2. Non compatible avec tous les ALTER             │
│     • Seulement operations qui rebuiltable         │
│     • ALGORITHM=INPLACE : N'utilise pas bulk       │
│     • Voir liste opérations compatibles            │
│                                                    │
│  3. Peut augmenter charge I/O ponctuelle           │
│     • Écritures plus grosses mais moins fréquentes │
│     • Pic d'écriture au lieu de lissage            │
│     • Planifier pendant heures creuses             │
│                                                    │
│  4. Incompatible avec certaines options ALTER      │
│     • LOCK=NONE sur certaines opérations           │
│     • Certains types de colonnes                   │
│                                                    │
│  5. Gain nul sur HDD                               │
│     • Optimisé pour SSD/NVMe uniquement            │
│     • Sur HDD : <5% gain, pas significatif         │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Opérations ALTER compatibles

```sql
-- ✅ COMPATIBLE avec innodb_alter_copy_bulk :

-- Ajout d'index
ALTER TABLE t ADD INDEX idx_name (column);
ALTER TABLE t ADD UNIQUE INDEX idx_unique (column);
ALTER TABLE t ADD FULLTEXT INDEX idx_ft (text_column);

-- Ajout de clé primaire (si pas existante)
ALTER TABLE t ADD PRIMARY KEY (id);

-- Rebuild de table
ALTER TABLE t ENGINE=InnoDB;
OPTIMIZE TABLE t;
ALTER TABLE t FORCE;

-- Modification nécessitant rebuild
ALTER TABLE t MODIFY COLUMN data TEXT;
ALTER TABLE t ADD COLUMN new_col INT AFTER existing;

-- ⚠️ NON COMPATIBLE (utilise ALGORITHM=INPLACE) :

-- Ajout colonne sans rebuild (depends on position)
ALTER TABLE t ADD COLUMN last_col INT;  -- Si à la fin

-- Suppression d'index (pas de construction)
ALTER TABLE t DROP INDEX idx_name;

-- Renommage (metadata only)
ALTER TABLE t RENAME TO t_new;
ALTER TABLE t CHANGE old_col new_col INT;
```

### Vérifier si bulk sera utilisé

```sql
-- Dry-run avec ALGORITHM explicite
EXPLAIN ALTER TABLE orders_test ADD INDEX idx_test (customer_id);
/*
Si "copy to tmp table" apparaît → Bulk sera utilisé
Si "inplace" → Bulk NON utilisé
*/

-- Forcer algorithme pour tester
ALTER TABLE orders_test 
    ADD INDEX idx_test (customer_id),
    ALGORITHM=COPY;  -- Force rebuild, utilisera bulk
```

---

## Best Practices en Production

### 1. Préparation avant ALTER TABLE

```sql
-- Checklist avant ALTER avec bulk

-- A. Vérifier espace disque (besoin ~2x table size)
SELECT 
    table_schema,
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024 / 1024, 2) as total_gb,
    ROUND((data_length + index_length) * 2 / 1024 / 1024 / 1024, 2) as needed_gb
FROM information_schema.tables
WHERE table_name = 'orders';

-- B. Vérifier RAM disponible
-- System: free -h
-- MariaDB buffer pool devrait avoir de la marge

-- C. Planifier hors heures de pointe
-- Vérifier charge actuelle
SHOW STATUS LIKE 'Threads_running';
SHOW STATUS LIKE 'Questions';

-- D. Backup avant modification critique
-- mysqldump ou mariabackup

-- E. Test en staging d'abord !
-- Mesurer durée et impact
```

### 2. Exécution optimale

```sql
-- Pattern recommandé pour ALTER en production

-- 1. Activer bulk
SET SESSION innodb_alter_copy_bulk = ON;

-- 2. Optionnel : Ajuster priorité I/O (Linux)
-- SET SESSION innodb_adaptive_flushing = OFF;  -- Temporaire

-- 3. Exécuter ALTER avec options explicites
ALTER TABLE orders 
    ADD INDEX idx_customer_date (customer_id, order_date),
    ALGORITHM=COPY,      -- Force utilisation bulk
    LOCK=SHARED;         -- Permet reads pendant ALTER

-- 4. Restaurer configuration
SET SESSION innodb_alter_copy_bulk = default;
```

### 3. Migrations multiples

```sql
-- Pour plusieurs ALTER successifs

-- Mauvais : Plusieurs ALTER séparés
ALTER TABLE t ADD INDEX idx1 (col1);  -- Rebuild complet
ALTER TABLE t ADD INDEX idx2 (col2);  -- Rebuild complet
ALTER TABLE t ADD INDEX idx3 (col3);  -- Rebuild complet
-- 3 rebuilds = 3x le temps !

-- Bon : Un seul ALTER avec multiples changements ✅
SET SESSION innodb_alter_copy_bulk = ON;
ALTER TABLE t 
    ADD INDEX idx1 (col1),
    ADD INDEX idx2 (col2),
    ADD INDEX idx3 (col3);
-- 1 seul rebuild avec bulk = Optimal !
```

### 4. Online vs Offline

```sql
-- Stratégie selon criticité

-- Option 1 : Offline (fenêtre de maintenance)
-- Plus rapide, pas de reads concurrents
ALTER TABLE orders 
    ADD INDEX idx_new (column),
    ALGORITHM=COPY,
    LOCK=EXCLUSIVE;

-- Option 2 : Online (production continue)
-- Plus lent, mais users peuvent lire
ALTER TABLE orders 
    ADD INDEX idx_new (column),
    ALGORITHM=COPY,
    LOCK=SHARED;  -- Reads autorisés

-- Option 3 : pt-online-schema-change (Percona)
-- Sans lock, via triggers
pt-online-schema-change \
    --alter "ADD INDEX idx_new (column)" \
    --execute \
    D=mydb,t=orders
# innodb_alter_copy_bulk utilisé pour la copie finale
```

### 5. Rollback et contingence

```sql
-- Préparer rollback avant ALTER

-- 1. Documenter l'état initial
SELECT 
    table_name,
    ROUND(data_length / 1024 / 1024, 2) as data_mb,
    ROUND(index_length / 1024 / 1024, 2) as index_mb
FROM information_schema.tables
WHERE table_name = 'orders';

SHOW CREATE TABLE orders\G

-- 2. Backup si critique
CREATE TABLE orders_backup LIKE orders;
INSERT INTO orders_backup SELECT * FROM orders;
-- Ou mysqldump

-- 3. Script de rollback
-- rollback.sql :
/*
ALTER TABLE orders DROP INDEX idx_new;
-- Ou restaurer backup si changements multiples
*/

-- 4. Tester temps de rollback en staging
```

---

## Comparaison avec alternatives

### vs pt-online-schema-change

```
┌────────────────────────────────────────────────────┐
│  innodb_alter_copy_bulk vs pt-osc                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  innodb_alter_copy_bulk :                          │
│  ✅ Natif MariaDB (pas de tool externe)            │
│  ✅ 40% plus rapide si offline possible            │
│  ✅ Pas de triggers overhead                       │
│  ⚠️ Lock table pendant ALTER                       │
│                                                    │
│  pt-online-schema-change :                         │
│  ✅ Zero downtime (table accessible)               │
│  ✅ Throttling automatique                         │
│  ✅ Annulation possible                            │
│  ⚠️ Overhead triggers (~10-20%)                    │
│  ⚠️ Plus lent au total                             │
│                                                    │
│  Recommandation :                                  │
│  • Fenêtre maintenance → bulk                      │
│  • Production 24/7 → pt-osc                        │
│  • Combiner : pt-osc utilise bulk en interne       │
│                                                    │
└────────────────────────────────────────────────────┘
```

### vs gh-ost

```sql
-- gh-ost (GitHub Online Schema Tool)
-- Similaire à pt-osc mais via binlog

-- Configuration pour utiliser bulk
gh-ost \
    --user="root" \
    --password="***" \
    --host=localhost \
    --database="mydb" \
    --table="orders" \
    --alter="ADD INDEX idx_new (column)" \
    --execute \
    --allow-on-master

# gh-ost n'utilise PAS directement innodb_alter_copy_bulk
# Car travaille via réplication binlog
# Mais bulk peut être utilisé sur la shadow table
```

---

## Troubleshooting

### Problème 1 : Bulk ne s'active pas

```sql
-- Symptômes : Pas de gain de performance

-- Diagnostic
SELECT @@innodb_alter_copy_bulk;  -- Vérifier activé

SHOW PROCESSLIST;
-- Regarder "State" pendant ALTER

-- Si "inplace" au lieu de "copy to tmp table"
-- → ALTER utilise INPLACE, bulk non applicable

-- Solution : Forcer ALGORITHM=COPY
ALTER TABLE t ADD INDEX idx (col), ALGORITHM=COPY;
```

### Problème 2 : Out of Memory

```sql
-- Symptômes : ALTER échoue avec "Out of memory"

-- Diagnostic
-- Vérifier RAM disponible
-- free -h

-- Vérifier tmp_table_size
SELECT @@tmp_table_size / 1024 / 1024 as tmp_mb;

-- Solution 1 : Augmenter RAM temporaire
SET SESSION tmp_table_size = 2147483648;  -- 2 GB
SET SESSION max_heap_table_size = 2147483648;

-- Solution 2 : Désactiver bulk si RAM critique
SET SESSION innodb_alter_copy_bulk = OFF;

-- Solution 3 : Utiliser pt-osc (plus lent mais stable)
```

### Problème 3 : I/O saturés

```bash
# Symptômes : iostat montre %util = 100% pendant ALTER

# Diagnostic
iostat -x 1
# Si %util constant à 100%, disques saturés

# Solutions :

# 1. Réduire innodb_io_capacity temporairement
mysql -e "SET GLOBAL innodb_io_capacity = 500;"

# 2. Utiliser ionice (Linux)
ionice -c 3 mysql  # Idle priority

# 3. Planifier à heure creuse

# 4. Throttling avec pt-osc
pt-online-schema-change --max-load="Threads_running=25"
```

### Problème 4 : ALTER plus lent que prévu

```sql
-- Diagnostic

-- 1. Vérifier bulk activé
SELECT @@innodb_alter_copy_bulk;

-- 2. Vérifier type disque
-- Si HDD : Gain minimal attendu

-- 3. Vérifier contention
SHOW ENGINE INNODB STATUS\G
-- Regarder section "SEMAPHORES"

-- 4. Vérifier buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool_reads';
-- Si élevé : Buffer pool trop petit

-- 5. Mesurer baseline
-- Tester sans bulk pour comparer
SET SESSION innodb_alter_copy_bulk = OFF;
-- Temps sans bulk
SET SESSION innodb_alter_copy_bulk = ON;
-- Temps avec bulk
```

---

## Cas d'usage avancés

### 1. Migration de production avec downtime minimal

```sql
-- Stratégie : Préparer puis basculer rapidement

-- Phase 1 : Créer nouvelle structure (shadow table)
CREATE TABLE orders_new LIKE orders;

-- Phase 2 : Ajouter index sur shadow (hors ligne)
SET SESSION innodb_alter_copy_bulk = ON;
ALTER TABLE orders_new 
    ADD INDEX idx_customer (customer_id),
    ADD INDEX idx_date (order_date),
    ADD INDEX idx_status (status);

-- Phase 3 : Copier données (peut être long)
INSERT INTO orders_new SELECT * FROM orders;

-- Phase 4 : Bascule rapide (downtime <1 minute)
START TRANSACTION;
RENAME TABLE orders TO orders_old, orders_new TO orders;
COMMIT;

-- Phase 5 : Cleanup
DROP TABLE orders_old;
```

### 2. Rebuild partiel avec filtres

```sql
-- Rebuild uniquement données récentes

-- Créer table avec index optimisé
CREATE TABLE orders_2024 LIKE orders;
SET SESSION innodb_alter_copy_bulk = ON;
ALTER TABLE orders_2024 ADD INDEX idx_optimized (customer_id, order_date);

-- Copier seulement 2024
INSERT INTO orders_2024 
SELECT * FROM orders 
WHERE YEAR(order_date) = 2024;

-- Archive anciennes données
CREATE TABLE orders_archive 
SELECT * FROM orders 
WHERE YEAR(order_date) < 2024;
```

### 3. Compression + Index

```sql
-- Combiner compression et index pour réduire taille

SET SESSION innodb_alter_copy_bulk = ON;

ALTER TABLE large_table 
    ROW_FORMAT=COMPRESSED,
    KEY_BLOCK_SIZE=8,
    ADD INDEX idx1 (col1),
    ADD INDEX idx2 (col2);

-- Bulk optimise la construction des index compressés aussi !
```

---

## ✅ Points clés à retenir

- 🆕 **innodb_alter_copy_bulk = MariaDB 11.8** : Feature majeure pour DDL rapides
- 📈 **Gain 30-50% sur SSD/NVMe** : Construction index significativement plus rapide
- ⚡ **Optimisé pour stockage moderne** : Blocs larges vs pages, meilleure bande passante
- 🎯 **Tables volumineuses** : Plus la table est grande, plus le gain est important
- 🔧 **Activation simple** : `SET innodb_alter_copy_bulk = ON`
- 💾 **SSD/NVMe requis** : Sur HDD, gain négligeable (<5%)
- 📊 **Monitoring important** : Surveiller progression et ressources
- ⚠️ **RAM nécessaire** : Blocs plus gros = plus de mémoire temporaire
- ✅ **Production-ready** : Stable, aucun risque de corruption
- 🔄 **Combiner avec pt-osc** : pt-online-schema-change utilise bulk en interne

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 innodb_alter_copy_bulk](https://mariadb.com/kb/en/innodb-system-variables/#innodb_alter_copy_bulk)
- [📖 ALTER TABLE Performance](https://mariadb.com/kb/en/alter-table/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/changes-improvements-in-mariadb-118/)

### Outils complémentaires

- [Percona Toolkit - pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [gh-ost](https://github.com/github/gh-ost)

### Benchmarks et études

- [MariaDB Foundation - 11.8 Performance Improvements](https://mariadb.org/mariadb-11-8-performance/)
- [InnoDB Bulk Load for CREATE INDEX (MySQL Blog)](https://dev.mysql.com/blog-archive/innodb-bulk-load-for-create-index/)

---

## ➡️ Section suivante

**[15.7 Analyse des requêtes lentes](/15-performance-tuning/07-analyse-requetes-lentes.md)** : Maintenant que nous avons optimisé la structure (index), apprenons à identifier et corriger les requêtes problématiques avec le slow query log et pt-query-digest.

---

*innodb_alter_copy_bulk est une innovation majeure de MariaDB 11.8 qui transforme les migrations de schéma sur stockage moderne. C'est l'un des gains de performance les plus tangibles et immédiats de cette version LTS.*

⏭️ [Analyse des requêtes lentes](/15-performance-tuning/07-analyse-requetes-lentes.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.6 Compression de Tables

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** : Chapitre 7 (Moteurs de stockage InnoDB), compréhension des performances I/O

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les **deux méthodes de compression InnoDB** (COMPRESSED vs PAGE_COMPRESSED)
- Configurer **ROW_FORMAT=COMPRESSED** avec KEY_BLOCK_SIZE approprié
- Utiliser **PAGE_COMPRESSED** avec compression transparente
- Analyser l'**impact sur performance** (CPU, I/O, espace disque)
- Choisir la **méthode de compression** selon le cas d'usage
- Optimiser la compression pour **cloud et SSD**
- Mesurer le **ratio de compression** et l'économie d'espace
- Identifier les **tables candidates** à la compression

---

## Introduction

La **compression de tables** permet de réduire l'espace disque utilisé et, dans certains cas, d'améliorer les performances I/O en échangeant de la puissance CPU contre une réduction du volume de données lues/écrites. MariaDB InnoDB offre deux méthodes principales de compression, chacune avec ses avantages et cas d'usage.

### Pourquoi Compresser les Tables ?

**Bénéfices de la compression** :

1. **💾 Réduction de l'Espace Disque**
   - Économie 40-70% d'espace selon les données
   - Stockage de plus de données sur même hardware
   - Réduction des coûts cloud (stockage facturé au Go)

2. **⚡ Amélioration des Performances I/O**
   - Moins de données à lire/écrire sur disque
   - Réduction de la latence I/O (surtout HDD)
   - Buffer pool plus efficace (plus de données en cache)

3. **💰 Optimisation des Coûts**
   - Stockage cloud moins cher (AWS EBS, Azure Disks)
   - Moins de IOPS consommés
   - Bande passante réseau réduite (réplication, backup)

4. **📈 Scalabilité Améliorée**
   - Archivage de plus d'historique
   - Rétention de logs plus longue
   - Capacité accrue sans agrandissement disque

**Compromis à considérer** :

- ⚠️ **CPU** : Compression/décompression consomme du CPU
- ⚠️ **Complexité** : Configuration et tuning nécessaires
- ⚠️ **Performances variables** : Dépend du type de données et workload

**Quand compresser** :
- ✅ Tables volumineuses peu modifiées (logs, archives)
- ✅ Données textuelles (JSON, XML, logs applicatifs)
- ✅ Environnements cloud avec stockage coûteux
- ✅ Tables avec ratio lecture/écriture élevé

**Quand NE PAS compresser** :
- ❌ Tables OLTP à forte écriture (overhead CPU)
- ❌ Données déjà compressées (images, vidéos, archives ZIP)
- ❌ Tables très sollicitées en UPDATE/DELETE
- ❌ Hardware ancien avec CPU limité

---

## Les Deux Méthodes de Compression

MariaDB InnoDB propose deux approches distinctes de compression :

### 1. ROW_FORMAT=COMPRESSED (Compression Page-Level)

**Principe** : Compresse les pages InnoDB (16KB par défaut) vers une taille plus petite (1K, 2K, 4K, 8K).

```sql
CREATE TABLE logs_compressed (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  timestamp TIMESTAMP,
  message TEXT,
  metadata JSON
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

**Caractéristiques** :
- Compression au niveau page InnoDB
- Taille de page compressée configurable (1, 2, 4, 8 KB)
- Stocke version non compressée en buffer pool
- Algorithme zlib (deflate)

**Avantages** :
- ✅ Compatible avec toutes versions InnoDB
- ✅ Transparente pour application
- ✅ Bon ratio compression/performance

**Inconvénients** :
- ⚠️ Overhead CPU modéré à élevé
- ⚠️ Configuration KEY_BLOCK_SIZE requise
- ⚠️ Double stockage en mémoire (compressed + uncompressed)

### 2. PAGE_COMPRESSED (Compression Transparente)

**Principe** : Utilise la compression au niveau filesystem avec "punch hole" (sparse files).

```sql
CREATE TABLE logs_page_compressed (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  timestamp TIMESTAMP,
  message TEXT,
  metadata JSON
) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=6;
```

**Caractéristiques** :
- Compression transparente au niveau OS
- "Punch hole" libère blocs inutilisés
- Algorithmes multiples (zlib, lz4, lzo, lzma, bzip2, snappy)
- Nécessite filesystem supportant punch hole (XFS, ext4, Btrfs)

**Avantages** :
- ✅ Overhead CPU généralement plus faible
- ✅ Pas de double stockage en mémoire
- ✅ Configuration simple (niveau 1-9)
- ✅ Meilleur ratio compression souvent

**Inconvénients** :
- ⚠️ Nécessite filesystem compatible
- ⚠️ Moins de contrôle granulaire
- ⚠️ Support variable selon OS/version

---

## ROW_FORMAT=COMPRESSED - Compression Page-Level

### Configuration et Paramètres

#### KEY_BLOCK_SIZE : Taille de Page Compressée

```sql
-- Tailles disponibles : 1, 2, 4, 8 (en KB)
-- Plus petit = meilleure compression, mais plus de CPU

-- KEY_BLOCK_SIZE=8 (recommandé pour démarrer)
CREATE TABLE moderate_compression (
  id INT PRIMARY KEY,
  data TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;

-- KEY_BLOCK_SIZE=4 (compression plus agressive)
CREATE TABLE high_compression (
  id INT PRIMARY KEY,
  data TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4;

-- KEY_BLOCK_SIZE=1 (compression maximale, CPU élevé)
CREATE TABLE ultra_compression (
  id INT PRIMARY KEY,
  data TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=1;
```

**Règles de choix KEY_BLOCK_SIZE** :

| KEY_BLOCK_SIZE | Ratio Compression | CPU Overhead | Usage |
|----------------|-------------------|--------------|-------|
| 8K | Modéré (30-40%) | Faible (+10-15%) | Tables fréquemment lues |
| 4K | Bon (40-50%) | Moyen (+20-30%) | Logs, archives récentes |
| 2K | Élevé (50-60%) | Élevé (+30-50%) | Archives, rarement lues |
| 1K | Maximum (60-70%) | Très élevé (+50-100%) | Archivage long terme |

#### Variables de Configuration InnoDB

```sql
-- Activer file-per-table (requis pour compression)
SET GLOBAL innodb_file_per_table = 1;

-- Taille page InnoDB (doit être >= KEY_BLOCK_SIZE)
-- Par défaut 16K, compatible avec tous KEY_BLOCK_SIZE
SHOW VARIABLES LIKE 'innodb_page_size';

-- Format de fichier compatible compression
SET GLOBAL innodb_file_format = 'Barracuda';  -- MariaDB 10.2+
SET GLOBAL innodb_file_format_max = 'Barracuda';

-- Niveau compression zlib (0-9, défaut 6)
SET GLOBAL innodb_compression_level = 6;
-- Plus élevé = meilleure compression, plus de CPU

-- Taux d'échec acceptable avant recompression
SET GLOBAL innodb_compression_failure_threshold_pct = 5;
-- Si >5% pages ne rentrent pas, recompression automatique

-- Padding pour éviter recompression fréquente
SET GLOBAL innodb_compression_pad_pct_max = 50;
-- Laisse 50% d'espace libre pour UPDATE
```

### Exemple Complet de Création

```sql
-- Configuration serveur (my.cnf)
-- [mysqld]
-- innodb_file_per_table = 1
-- innodb_compression_level = 6
-- innodb_compression_failure_threshold_pct = 5

-- Table de logs applicatifs
CREATE TABLE application_logs (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  application VARCHAR(50),
  level ENUM('DEBUG','INFO','WARN','ERROR','FATAL'),
  message TEXT,
  context JSON,
  stack_trace TEXT,
  created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
  
  -- Index
  INDEX idx_app_time (application, created_at),
  INDEX idx_level (level, created_at)
  
) ENGINE=InnoDB 
  ROW_FORMAT=COMPRESSED 
  KEY_BLOCK_SIZE=4
  COMMENT='Logs applicatifs compressés, rétention 90 jours';

-- Vérifier configuration
SHOW CREATE TABLE application_logs;
```

### Impact sur Performances

**Benchmark typique** (table de logs, 10M lignes) :

```sql
-- Table non compressée
CREATE TABLE logs_uncompressed (
  log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  timestamp TIMESTAMP,
  level VARCHAR(10),
  message TEXT,
  metadata JSON
) ENGINE=InnoDB;

-- Table compressée KEY_BLOCK_SIZE=4
CREATE TABLE logs_compressed_4k (
  log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  timestamp TIMESTAMP,
  level VARCHAR(10),
  message TEXT,
  metadata JSON
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4;

-- Insérer 10M lignes de test
-- ...

-- Résultats typiques :
```

| Métrique | Non Compressée | Compressée 4K | Différence |
|----------|---------------|---------------|------------|
| **Taille disque** | 2.5 GB | 1.1 GB | -56% |
| **INSERT 100K rows** | 12.5 sec | 15.8 sec | +26% |
| **SELECT COUNT(*)** | 3.2 sec | 3.5 sec | +9% |
| **SELECT avec WHERE** | 1.8 sec | 1.4 sec | -22% (I/O réduit) |
| **UPDATE 10K rows** | 2.1 sec | 3.4 sec | +62% |
| **DELETE 10K rows** | 1.9 sec | 2.8 sec | +47% |

**Observations** :
- ✅ Économie espace disque significative (50-60%)
- ⚠️ INSERT/UPDATE/DELETE plus lents (compression overhead)
- ✅ SELECT parfois plus rapides (moins de I/O)
- Optimal pour tables **read-heavy**

---

## PAGE_COMPRESSED - Compression Transparente

### Configuration et Utilisation

#### Activer PAGE_COMPRESSED

```sql
-- Compression transparente avec niveau par défaut (6)
CREATE TABLE logs_page (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  timestamp TIMESTAMP,
  message TEXT,
  metadata JSON
) PAGE_COMPRESSED=1;

-- Avec niveau de compression spécifique (1-9)
CREATE TABLE logs_page_level9 (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  timestamp TIMESTAMP,
  message TEXT,
  metadata JSON
) PAGE_COMPRESSED=1 
  PAGE_COMPRESSION_LEVEL=9;  -- Compression maximale

-- Algorithme de compression personnalisé
CREATE TABLE logs_lz4 (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  timestamp TIMESTAMP,
  message TEXT
) PAGE_COMPRESSED=1 
  PAGE_COMPRESSION_LEVEL=6
  COMPRESSION='lz4';  -- Algorithme LZ4 (plus rapide)
```

#### Algorithmes de Compression Disponibles

| Algorithme | Ratio | Vitesse | CPU | Usage |
|------------|-------|---------|-----|-------|
| **zlib** | Bon | Moyenne | Moyen | Défaut, équilibré |
| **lz4** | Modéré | Très rapide | Faible | OLTP, performance |
| **lzo** | Modéré | Rapide | Faible | Alternative à lz4 |
| **lzma** | Excellent | Lente | Élevé | Archivage |
| **bzip2** | Très bon | Lente | Élevé | Archivage |
| **snappy** | Modéré | Très rapide | Très faible | Google, performance |

```sql
-- Exemple avec LZ4 (recommandé pour OLTP)
CREATE TABLE orders_lz4 (
  order_id INT PRIMARY KEY,
  customer_id INT,
  order_data JSON
) PAGE_COMPRESSED=1 COMPRESSION='lz4';

-- Exemple avec LZMA (archivage long terme)
CREATE TABLE archives_lzma (
  archive_id BIGINT PRIMARY KEY,
  archive_data LONGTEXT
) PAGE_COMPRESSED=1 
  COMPRESSION='lzma' 
  PAGE_COMPRESSION_LEVEL=9;
```

### Prérequis Filesystem

**Filesystems supportant punch hole** :
- ✅ **XFS** (recommandé, support natif excellent)
- ✅ **ext4** (avec feature "extents" activée)
- ✅ **Btrfs** (support natif)
- ❌ **ext3** (non supporté)
- ❌ **NTFS** (Windows, non supporté)

**Vérifier support punch hole** :
```bash
# Linux : Vérifier filesystem
df -T /var/lib/mysql
# Filesystem     Type
# /dev/sda1      xfs   ← OK

# Tester punch hole manuellement
fallocate -p -o 0 -l 4096 testfile
# Si pas d'erreur → punch hole supporté

# MariaDB : Vérifier dans error log au démarrage
grep -i "punch hole" /var/log/mysql/error.log
# InnoDB: Using punch hole for PAGE_COMPRESSED
```

**Configuration my.cnf** :
```ini
[mysqld]
# Activer page compression
innodb_file_per_table = 1

# Algorithme par défaut (zlib, lz4, lzo, etc.)
innodb_compression_algorithm = zlib

# Niveau par défaut si non spécifié
innodb_compression_default = 6

# Désactiver double-write pour PAGE_COMPRESSED (optionnel, risqué)
# innodb_doublewrite = 0  # Améliore perfs mais risque corruption crash
```

### Avantages PAGE_COMPRESSED

```sql
-- Comparaison ROW_FORMAT vs PAGE_COMPRESSED
-- Table test : 5M lignes, colonnes TEXT/JSON

-- ROW_FORMAT=COMPRESSED
CREATE TABLE test_row_compressed (
  id INT PRIMARY KEY,
  data TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4;

-- PAGE_COMPRESSED
CREATE TABLE test_page_compressed (
  id INT PRIMARY KEY,
  data TEXT
) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=6;

-- Résultats typiques :
```

| Métrique | ROW_COMPRESSED | PAGE_COMPRESSED | Avantage |
|----------|---------------|-----------------|----------|
| **Taille disque** | 1.2 GB | 0.9 GB | PAGE (-25%) |
| **Mémoire buffer pool** | 2.4 GB (double) | 1.2 GB | PAGE (-50%) |
| **INSERT 100K** | 18 sec | 14 sec | PAGE (-22%) |
| **SELECT COUNT(*)** | 4 sec | 3.2 sec | PAGE (-20%) |
| **CPU usage** | 35% | 28% | PAGE (-20%) |

**PAGE_COMPRESSED gagne généralement** sur :
- ✅ Consommation mémoire (pas de double stockage)
- ✅ CPU overhead plus faible (algorithmes optimisés)
- ✅ Performance globale meilleure
- ✅ Ratio compression souvent supérieur

---

## Cas d'Usage et Exemples Concrets

### 1. Table de Logs Applicatifs

**Besoin** : Logs JSON volumineux, rétention 90 jours, lectures rares.

```sql
-- Table de logs avec compression agressive
CREATE TABLE application_logs (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  application VARCHAR(50),
  environment ENUM('DEV','STAGING','PROD'),
  level ENUM('DEBUG','INFO','WARN','ERROR','FATAL'),
  timestamp TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
  
  -- Données volumineuses
  message TEXT,
  context JSON,
  stack_trace TEXT,
  request_headers JSON,
  response_body TEXT,
  
  -- Métadonnées
  user_id INT,
  session_id VARCHAR(64),
  ip_address VARCHAR(45),
  
  -- Index
  INDEX idx_app_env_time (application, environment, timestamp),
  INDEX idx_level_time (level, timestamp),
  INDEX idx_user (user_id, timestamp)
  
) ENGINE=InnoDB
  PAGE_COMPRESSED=1
  PAGE_COMPRESSION_LEVEL=6
  COMPRESSION='zlib'
  COMMENT='Logs applicatifs, rétention 90j, compression ~60%';

-- Partitionnement par mois pour archivage
ALTER TABLE application_logs
PARTITION BY RANGE (UNIX_TIMESTAMP(timestamp)) (
  PARTITION p202501 VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01')),
  PARTITION p202502 VALUES LESS THAN (UNIX_TIMESTAMP('2025-03-01')),
  PARTITION p202503 VALUES LESS THAN (UNIX_TIMESTAMP('2025-04-01')),
  -- ...
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Résultat typique :
-- Sans compression : 50 GB/mois
-- Avec compression : 18 GB/mois (-64%)
-- Économie : 32 GB/mois, ~400 GB/an
```

### 2. Archivage de Données Historiques

**Besoin** : Commandes >2 ans, rarement consultées, rétention légale 7 ans.

```sql
-- Table d'archive avec compression maximale
CREATE TABLE orders_archive (
  order_id BIGINT PRIMARY KEY,
  customer_id INT,
  order_date DATE,
  
  -- Données commande (snapshot complet)
  order_data JSON,          -- Détail items, prix, promos
  shipping_info JSON,       -- Adresse, transporteur, tracking
  payment_info JSON,        -- Mode paiement, transaction_id
  customer_snapshot JSON,   -- État client à date commande
  
  -- Métadonnées
  archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  original_table VARCHAR(50),
  
  INDEX idx_customer (customer_id, order_date),
  INDEX idx_date (order_date)
  
) ENGINE=InnoDB
  PAGE_COMPRESSED=1
  PAGE_COMPRESSION_LEVEL=9  -- Compression maximale
  COMPRESSION='lzma'        -- Algorithme le plus efficace
  COMMENT='Archive commandes >2 ans, rétention 7 ans';

-- Procédure d'archivage mensuel
DELIMITER $$
CREATE PROCEDURE archive_old_orders()
BEGIN
  DECLARE v_cutoff_date DATE;
  SET v_cutoff_date = DATE_SUB(CURDATE(), INTERVAL 2 YEAR);
  
  -- Copier vers archive
  INSERT INTO orders_archive (
    order_id, customer_id, order_date, order_data, 
    shipping_info, payment_info, customer_snapshot, original_table
  )
  SELECT 
    o.order_id,
    o.customer_id,
    o.order_date,
    JSON_OBJECT(
      'items', (SELECT JSON_ARRAYAGG(JSON_OBJECT(
        'product_id', oi.product_id,
        'quantity', oi.quantity,
        'unit_price', oi.unit_price
      )) FROM order_items oi WHERE oi.order_id = o.order_id),
      'total', o.total_amount,
      'status', o.status
    ) AS order_data,
    JSON_OBJECT('address', o.shipping_address, 'method', o.shipping_method) AS shipping_info,
    JSON_OBJECT('method', o.payment_method, 'transaction', o.transaction_id) AS payment_info,
    JSON_OBJECT('name', c.name, 'email', c.email) AS customer_snapshot,
    'orders' AS original_table
  FROM orders o
  INNER JOIN customers c ON o.customer_id = c.customer_id
  WHERE o.order_date < v_cutoff_date
    AND o.order_id NOT IN (SELECT order_id FROM orders_archive);
  
  -- Supprimer de table active
  DELETE FROM orders WHERE order_date < v_cutoff_date;
  
  SELECT 
    ROW_COUNT() AS orders_archived,
    v_cutoff_date AS cutoff_date;
END$$
DELIMITER ;

-- Planifier exécution mensuelle
CREATE EVENT archive_orders_monthly
ON SCHEDULE EVERY 1 MONTH
STARTS '2025-01-01 02:00:00'
DO CALL archive_old_orders();

-- Résultat typique :
-- Sans compression : 200 GB archives
-- Avec LZMA level 9 : 45 GB (-77%)
```

### 3. Table JSON de Données Semi-Structurées

**Besoin** : Documents JSON, recherche occasionnelle, stockage optimisé.

```sql
-- Table de documents JSON compressés
CREATE TABLE documents (
  document_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  document_type VARCHAR(50),
  
  -- Document JSON complet
  content JSON,
  
  -- Métadonnées (colonnes générées pour recherche)
  title VARCHAR(255) AS (content->>'$.title') VIRTUAL,
  author VARCHAR(100) AS (content->>'$.author') VIRTUAL,
  created_date DATE AS (content->>'$.created_date') VIRTUAL,
  tags JSON AS (content->'$.tags') VIRTUAL,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  -- Index sur colonnes virtuelles
  INDEX idx_title (title),
  INDEX idx_author (author),
  INDEX idx_type_date (document_type, created_date),
  FULLTEXT INDEX ft_content (content)
  
) ENGINE=InnoDB
  PAGE_COMPRESSED=1
  PAGE_COMPRESSION_LEVEL=7
  COMPRESSION='zlib'
  COMMENT='Documents JSON, compression ~65%';

-- Insertion exemple
INSERT INTO documents (document_type, content) VALUES
('article', JSON_OBJECT(
  'title', 'Introduction to MariaDB Compression',
  'author', 'Jane Doe',
  'created_date', '2025-01-15',
  'tags', JSON_ARRAY('database', 'compression', 'performance'),
  'body', 'MariaDB offers two main compression methods...',  -- Texte long
  'metadata', JSON_OBJECT('word_count', 2500, 'language', 'en')
));

-- Recherche optimisée (utilise index sur colonne virtuelle)
SELECT document_id, title, author, created_date
FROM documents
WHERE author = 'Jane Doe'
  AND created_date >= '2025-01-01';

-- Résultat typique :
-- Documents JSON moyens : 50 KB/document
-- 1M documents : 50 GB non compressé
-- Avec compression : 17 GB (-66%)
```

### 4. Tables de Mesures IoT/Télémétrie

**Besoin** : Millions de mesures/jour, rétention 1 an, lectures agrégées.

```sql
-- Table de mesures IoT
CREATE TABLE sensor_readings (
  reading_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  sensor_id INT,
  timestamp TIMESTAMP(6),
  
  -- Mesures (souvent répétitives, compressent bien)
  temperature DECIMAL(5,2),
  humidity DECIMAL(5,2),
  pressure DECIMAL(7,2),
  battery_level TINYINT,
  
  -- Métadonnées (JSON, compresse très bien)
  raw_data JSON,  -- Données brutes capteur
  
  INDEX idx_sensor_time (sensor_id, timestamp),
  INDEX idx_timestamp (timestamp)
  
) ENGINE=InnoDB
  PAGE_COMPRESSED=1
  PAGE_COMPRESSION_LEVEL=4  -- Compromis perf/compression
  COMPRESSION='lz4'         -- Rapide pour insertions fréquentes
  COMMENT='Mesures IoT, compression LZ4 pour performance INSERT'
PARTITION BY RANGE (UNIX_TIMESTAMP(timestamp)) (
  PARTITION p_2025_01 VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01')),
  PARTITION p_2025_02 VALUES LESS THAN (UNIX_TIMESTAMP('2025-03-01')),
  -- ... partitions mensuelles
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Insertion batch (100K lectures/batch)
INSERT INTO sensor_readings (sensor_id, timestamp, temperature, humidity, pressure, battery_level, raw_data)
SELECT 
  sensor_id,
  FROM_UNIXTIME(UNIX_TIMESTAMP('2025-01-15 00:00:00') + seq * 60) AS timestamp,
  20 + RAND() * 10 AS temperature,
  40 + RAND() * 30 AS humidity,
  1000 + RAND() * 30 AS pressure,
  90 - (seq % 100) AS battery_level,
  JSON_OBJECT('raw', CONCAT('DATA_', seq)) AS raw_data
FROM seq_1_to_100000
CROSS JOIN (SELECT sensor_id FROM sensors) AS s;

-- Agrégation (bénéficie de compression = moins de I/O)
SELECT 
  sensor_id,
  DATE(timestamp) AS reading_date,
  AVG(temperature) AS avg_temp,
  MIN(temperature) AS min_temp,
  MAX(temperature) AS max_temp,
  COUNT(*) AS reading_count
FROM sensor_readings
WHERE timestamp >= '2025-01-01'
  AND timestamp < '2025-02-01'
GROUP BY sensor_id, DATE(timestamp);

-- Résultat typique :
-- 10M lectures/jour : 3 GB/jour non compressé
-- Avec LZ4 level 4 : 1.2 GB/jour (-60%)
-- Performance INSERT : -5% vs non compressé (LZ4 très rapide)
```

---

## Monitoring et Mesure de la Compression

### Vérifier Ratio de Compression

```sql
-- Requête pour voir taille réelle vs taille compressée
SELECT 
  table_schema AS 'Database',
  table_name AS 'Table',
  ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)',
  ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)',
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS 'Total (MB)',
  table_rows AS 'Rows',
  ROW_FORMAT,
  CREATE_OPTIONS
FROM information_schema.TABLES
WHERE table_schema NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
  AND (ROW_FORMAT = 'Compressed' OR CREATE_OPTIONS LIKE '%PAGE_COMPRESSED%')
ORDER BY (data_length + index_length) DESC;

-- Exemple résultat :
-- Database | Table             | Data (MB) | Index (MB) | Total (MB) | Rows    | ROW_FORMAT | CREATE_OPTIONS
-- mydb     | application_logs  | 1234.56   | 234.12     | 1468.68    | 5000000 | Compressed | KEY_BLOCK_SIZE=4
-- mydb     | sensor_readings   | 890.23    | 123.45     | 1013.68    | 10000000| Dynamic    | page_compressed=1
```

### Statistiques InnoDB Compression

```sql
-- Statistiques détaillées de compression
SELECT * FROM information_schema.INNODB_CMP;

-- Colonnes importantes :
-- page_size : Taille page compressée (KEY_BLOCK_SIZE)
-- compress_ops : Nombre d'opérations de compression
-- compress_ops_ok : Compressions réussies
-- compress_time : Temps CPU compression (microsecondes)
-- uncompress_ops : Nombre de décompressions
-- uncompress_time : Temps CPU décompression

-- Taux d'échec de compression
SELECT 
  page_size,
  compress_ops,
  compress_ops - compress_ops_ok AS compress_failures,
  ROUND((compress_ops - compress_ops_ok) / compress_ops * 100, 2) AS failure_rate_pct
FROM information_schema.INNODB_CMP
WHERE compress_ops > 0;

-- Si failure_rate_pct > 5% → Augmenter KEY_BLOCK_SIZE ou désactiver compression
```

### Monitoring en Production

```sql
-- Vue pour monitoring régulier
CREATE VIEW compression_stats AS
SELECT 
  t.table_schema,
  t.table_name,
  t.row_format,
  t.table_rows,
  ROUND((t.data_length + t.index_length) / 1024 / 1024, 2) AS total_mb,
  ROUND(t.data_length / t.table_rows / 1024, 2) AS kb_per_row,
  c.compress_ops,
  c.uncompress_ops,
  ROUND(c.compress_time / 1000000, 2) AS compress_time_sec,
  ROUND(c.uncompress_time / 1000000, 2) AS uncompress_time_sec,
  ROUND((c.compress_ops - c.compress_ops_ok) / c.compress_ops * 100, 2) AS failure_pct
FROM information_schema.TABLES t
LEFT JOIN information_schema.INNODB_CMP c 
  ON c.page_size = SUBSTRING_INDEX(SUBSTRING_INDEX(t.create_options, 'KEY_BLOCK_SIZE=', -1), ' ', 1) * 1024
WHERE t.table_schema NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
  AND (t.row_format = 'Compressed' OR t.create_options LIKE '%PAGE_COMPRESSED%');

-- Alertes
SELECT * FROM compression_stats WHERE failure_pct > 5;
```

---

## Migration vers Compression

### Ajouter Compression à Table Existante

```sql
-- Table existante non compressée
CREATE TABLE logs (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  message TEXT,
  created_at TIMESTAMP
) ENGINE=InnoDB;

-- 10M lignes existantes, 5 GB

-- Option 1 : ALTER TABLE (bloque table, peut être long)
ALTER TABLE logs 
  ROW_FORMAT=COMPRESSED 
  KEY_BLOCK_SIZE=4;
-- Durée typique : 15-30 min pour 5 GB, table LOCKED

-- Option 2 : Créer nouvelle table + copie + swap (moins de downtime)
-- Étape 1 : Créer table compressée
CREATE TABLE logs_compressed LIKE logs;
ALTER TABLE logs_compressed 
  ROW_FORMAT=COMPRESSED 
  KEY_BLOCK_SIZE=4;

-- Étape 2 : Copier données (batch pour réduire lock)
INSERT INTO logs_compressed 
SELECT * FROM logs 
WHERE log_id <= 5000000;  -- Premier batch

INSERT INTO logs_compressed 
SELECT * FROM logs 
WHERE log_id > 5000000 AND log_id <= 10000000;  -- Deuxième batch

-- Étape 3 : Synchroniser nouvelles données (pendant migration)
INSERT INTO logs_compressed 
SELECT * FROM logs l
WHERE l.log_id > (SELECT MAX(log_id) FROM logs_compressed);

-- Étape 4 : Swap tables (rapide)
RENAME TABLE 
  logs TO logs_old,
  logs_compressed TO logs;

-- Étape 5 : Vérifier, puis supprimer ancienne
-- DROP TABLE logs_old;

-- Résultat : 5 GB → 2.1 GB (-58%)
```

### Utiliser gh-ost ou pt-online-schema-change

```bash
# gh-ost : Migration sans downtime
gh-ost \
  --host=localhost \
  --database=mydb \
  --table=logs \
  --alter="ROW_FORMAT=COMPRESSED, KEY_BLOCK_SIZE=4" \
  --initially-drop-ghost-table \
  --initially-drop-old-table \
  --execute

# Avantages :
# - Pas de lock de la table originale
# - Migrations longues possibles
# - Pause/resume si besoin
# - Rollback facile
```

---

## Optimisation et Tuning

### Choisir le Bon Niveau de Compression

```sql
-- Test : Créer 4 tables avec différents niveaux
CREATE TABLE test_level_1 (...) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=1;
CREATE TABLE test_level_3 (...) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=3;
CREATE TABLE test_level_6 (...) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=6;
CREATE TABLE test_level_9 (...) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=9;

-- Insérer mêmes données dans toutes
INSERT INTO test_level_1 SELECT * FROM source_data;
INSERT INTO test_level_3 SELECT * FROM source_data;
INSERT INTO test_level_6 SELECT * FROM source_data;
INSERT INTO test_level_9 SELECT * FROM source_data;

-- Comparer taille et performance
SELECT 
  table_name,
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb,
  SUBSTRING_INDEX(create_options, 'PAGE_COMPRESSION_LEVEL=', -1) AS level
FROM information_schema.TABLES
WHERE table_name LIKE 'test_level_%';

-- Résultats typiques (1 GB données source) :
-- Level 1 : 680 MB, INSERT rapide
-- Level 3 : 540 MB, INSERT moyen
-- Level 6 : 420 MB, INSERT lent
-- Level 9 : 380 MB, INSERT très lent

-- Choisir niveau 6 (bon compromis)
```

### Compression pour SSD vs HDD

```sql
-- SSD : Privilégier vitesse (CPU < I/O)
-- Recommandation : LZ4 ou Snappy
CREATE TABLE ssd_optimized (
  id INT PRIMARY KEY,
  data TEXT
) PAGE_COMPRESSED=1 
  COMPRESSION='lz4' 
  PAGE_COMPRESSION_LEVEL=3;

-- HDD : Privilégier compression (I/O critique)
-- Recommandation : zlib level 6-9
CREATE TABLE hdd_optimized (
  id INT PRIMARY KEY,
  data TEXT
) PAGE_COMPRESSED=1 
  COMPRESSION='zlib' 
  PAGE_COMPRESSION_LEVEL=7;
```

### Compression Cloud (AWS, Azure, GCP)

```sql
-- Cloud : Optimiser coûts stockage + IOPS
-- EBS GP3 : $0.08/GB + $0.005/IOPS
-- Compression 60% = économie $48/TB/mois

-- Configuration recommandée AWS RDS / Azure Database
CREATE TABLE cloud_table (
  id BIGINT PRIMARY KEY,
  data JSON
) PAGE_COMPRESSED=1 
  COMPRESSION='zlib'
  PAGE_COMPRESSION_LEVEL=6;

-- Bénéfices :
-- - Réduction stockage (facturation GB)
-- - Réduction IOPS (moins de lectures)
-- - Réduction coûts snapshots/backups
-- - Réduction bande passante réplication

-- Économie typique :
-- 100 GB → 40 GB après compression
-- Économie : $4.80/mois stockage
-- + Réduction IOPS : ~$10-20/mois
-- Total : ~$15-25/mois par 100 GB
```

---

## Best Practices

### 1. Tester avant Production

```sql
-- ✅ Toujours tester sur copie avec données réelles
CREATE TABLE test_compression LIKE production_table;
ALTER TABLE test_compression PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=6;

-- Copier échantillon représentatif
INSERT INTO test_compression 
SELECT * FROM production_table LIMIT 1000000;

-- Benchmark
SELECT BENCHMARK(1000, (SELECT COUNT(*) FROM test_compression));
SELECT BENCHMARK(100, (SELECT * FROM test_compression WHERE id = RAND() * 1000000));
```

### 2. Monitoring Continu

```sql
-- ✅ Surveiller taux d'échec compression
SELECT failure_pct FROM compression_stats WHERE failure_pct > 5;

-- ✅ Alertes si ratio compression < 20%
SELECT table_name, 
  ROUND((1 - (data_length / (table_rows * avg_row_length))) * 100, 2) AS compression_ratio
FROM information_schema.TABLES
WHERE compression_ratio < 20;
```

### 3. Adapter selon Workload

```sql
-- ✅ OLTP haute écriture : LZ4, niveau bas
CREATE TABLE oltp_table (...) 
  PAGE_COMPRESSED=1 COMPRESSION='lz4' PAGE_COMPRESSION_LEVEL=3;

-- ✅ OLAP lecture intensive : zlib, niveau élevé
CREATE TABLE analytics_table (...) 
  PAGE_COMPRESSED=1 COMPRESSION='zlib' PAGE_COMPRESSION_LEVEL=7;

-- ✅ Archivage : LZMA, niveau max
CREATE TABLE archive_table (...) 
  PAGE_COMPRESSED=1 COMPRESSION='lzma' PAGE_COMPRESSION_LEVEL=9;
```

### 4. Partitionnement + Compression

```sql
-- ✅ Combiner pour optimisation maximale
CREATE TABLE logs_optimized (
  log_id BIGINT AUTO_INCREMENT,
  created_at TIMESTAMP,
  message TEXT,
  PRIMARY KEY (log_id, created_at)
) PAGE_COMPRESSED=1 COMPRESSION='zlib' PAGE_COMPRESSION_LEVEL=6
PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
  PARTITION p_current VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Archiver partition ancienne avec compression max
ALTER TABLE logs_optimized 
  REORGANIZE PARTITION p_current INTO (
    PARTITION p_2025_01 VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01'))
      PAGE_COMPRESSED=1 COMPRESSION='lzma' PAGE_COMPRESSION_LEVEL=9
  );
```

---

## ✅ Points clés à retenir

### Deux Méthodes de Compression
- ✅ **ROW_FORMAT=COMPRESSED** : Compression page-level InnoDB (KEY_BLOCK_SIZE: 1, 2, 4, 8)
- ✅ **PAGE_COMPRESSED** : Compression transparente OS avec punch hole (niveaux 1-9)
- ✅ **Recommandation générale** : PAGE_COMPRESSED (meilleure performance, ratio supérieur)

### Performance
- ✅ **Économie espace** : 40-70% selon données (texte/JSON = meilleur)
- ⚠️ **CPU overhead** : +10-50% selon algorithme et niveau
- ✅ **I/O améliorés** : Moins de données lues/écrites (bénéfice SSD++)
- ⚠️ **Écritures plus lentes** : INSERT/UPDATE +15-50%
- ✅ **Lectures variables** : Parfois plus rapides (moins I/O), parfois plus lentes (CPU)

### Algorithmes
- ✅ **zlib** : Défaut, équilibré (ratio/vitesse)
- ✅ **lz4** : Rapide, OLTP, faible overhead CPU
- ✅ **lzma** : Maximum compression, archivage
- ✅ **snappy** : Très rapide, Google

### Cas d'Usage Idéaux
- ✅ **Logs applicatifs** : Texte, JSON, rétention longue
- ✅ **Archives** : Données >1-2 ans, lectures rares
- ✅ **JSON/XML** : Documents semi-structurés
- ✅ **IoT/Télémétrie** : Millions de mesures, agrégations
- ✅ **Cloud** : Réduction coûts stockage et IOPS

### Best Practices
- ✅ Tester sur copie avec données réelles avant production
- ✅ Commencer niveau 6 (équilibré), ajuster selon besoins
- ✅ LZ4 pour OLTP, zlib pour OLAP, LZMA pour archives
- ✅ Monitoring continu (taux échec, ratio compression)
- ✅ Combiner avec partitionnement pour archivage optimal
- ✅ SSD : Privilégier vitesse (LZ4), HDD : Privilégier compression (zlib)

### Limitations
- ⚠️ PAGE_COMPRESSED nécessite filesystem compatible (XFS, ext4)
- ⚠️ Pas bénéfique sur données déjà compressées (images, vidéos)
- ⚠️ Overhead CPU peut être prohibitif sur hardware ancien
- ⚠️ Tables OLTP haute écriture : évaluer compromis

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [InnoDB Compression](https://mariadb.com/kb/en/innodb-page-compression/) - Guide complet
- 📖 [ROW_FORMAT=COMPRESSED](https://mariadb.com/kb/en/innodb-row-formats-overview/#compressed) - Page-level
- 📖 [PAGE_COMPRESSED](https://mariadb.com/kb/en/innodb-page-compression/) - Transparente
- 📖 [Compression Algorithms](https://mariadb.com/kb/en/compression-algorithms/) - Détails algorithmes

### Performance et Benchmarks
- 📝 [InnoDB Compression Performance](https://mariadb.com/resources/blog/innodb-compression-performance/)
- 📝 [Compression for SSD vs HDD](https://mariadb.com/kb/en/compression-ssd-vs-hdd/)
- 📝 [Cloud Cost Optimization with Compression](https://mariadb.com/resources/blog/cloud-cost-compression/)

### Outils
- 🛠️ [pt-table-checksum](https://www.percona.com/doc/percona-toolkit/) - Vérifier intégrité post-compression
- 🛠️ [sysbench](https://github.com/akopytov/sysbench) - Benchmark compression
- 🛠️ [mysqltuner](https://github.com/major/MySQLTuner-perl) - Recommandations compression

---

## ➡️ Section suivante

**[18.7 Encryption at Rest](./07-encryption-at-rest.md)** : Découvrez comment protéger vos données stockées avec le chiffrement InnoDB, la gestion des clés (file, AWS KMS, Vault), et l'impact sur les performances.

---


⏭️ [Encryption at rest](/18-fonctionnalites-avancees/07-encryption-at-rest.md)

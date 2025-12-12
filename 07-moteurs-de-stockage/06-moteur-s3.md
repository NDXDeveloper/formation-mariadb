🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.6 Moteur S3 : Archivage données froides sur AWS S3/MinIO

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 7.1-7.2 (Architecture, InnoDB), concepts cloud storage, AWS S3 ou MinIO

> **Public cible** : DBA, Architectes cloud, DevOps, Data Engineers

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture du moteur S3 et son intégration avec le stockage objet
- Configurer MariaDB pour utiliser AWS S3 ou MinIO comme backend de stockage
- Identifier les cas d'usage appropriés pour l'archivage de données froides
- Implémenter une stratégie de tiering (données chaudes/froides)
- Optimiser les coûts de stockage avec le moteur S3
- Gérer les performances et limitations du stockage objet
- Intégrer S3 dans une architecture de données multi-tiers
- Comprendre les aspects de sécurité et de conformité

---

## Introduction

Le **moteur S3** (introduit dans MariaDB 10.5) permet de stocker des tables MariaDB directement sur du **stockage objet** compatible S3 (AWS S3, MinIO, Ceph, etc.). C'est une innovation majeure pour l'**archivage économique** de données froides rarement accédées.

### Problématique : Le coût du stockage chaud

```
Évolution typique d'une base de données :
┌─────────────────────────────────────────────────────────┐
│ Année 1 : 100 GB (données actives, accès fréquent)      │
│   • Stockage SSD local : 100 GB × $0.10/GB = $10/mois   │
│   • Total : $10/mois                                    │
├─────────────────────────────────────────────────────────┤
│ Année 2 : 500 GB (400 GB historique rarement accédé)    │
│   • Tout sur SSD : 500 GB × $0.10/GB = $50/mois         │
│   • Gaspillage : 400 GB × $0.09/GB = $36/mois           │
├─────────────────────────────────────────────────────────┤
│ Année 5 : 5 TB (4.5 TB archives froides)                │
│   • Tout sur SSD : 5 TB × $0.10/GB = $500/mois          │
│   • Gaspillage : 4.5 TB × $0.09/GB = $450/mois ⚠️       │
└─────────────────────────────────────────────────────────┘

Solution avec moteur S3 :
┌─────────────────────────────────────────────────────────┐
│ Année 5 : 5 TB total                                    │
│   • Données chaudes (InnoDB SSD) : 500 GB × $0.10       │
│     = $50/mois                                          │
│   • Données froides (S3) : 4.5 TB × $0.01               │
│     = $45/mois                                          │
│   • Total : $95/mois (économie de 81% !)                │
└─────────────────────────────────────────────────────────┘
```

### Qu'est-ce que le stockage objet ?

**Stockage bloc** (traditionnel - SSD/HDD) :
```
Accès : Direct via filesystem
Performance : Très rapide (latence µs)
Coût : Élevé ($0.10-0.20/GB/mois)
Scalabilité : Limitée au serveur
```

**Stockage objet** (S3, MinIO) :
```
Accès : HTTP/HTTPS API REST
Performance : Moyen (latence ms)
Coût : Très faible ($0.01-0.03/GB/mois)
Scalabilité : Illimitée (pétabytes)
```

**Comparaison visuelle** :

```
┌─────────────────────────────────────────────────────────┐
│         Stockage Bloc (InnoDB classique)                │
│                                                         │
│  MariaDB ──► Filesystem ──► Disque local (SSD/HDD)      │
│             (ext4, xfs)     (/var/lib/mysql)            │
│                                                         │
│  • Latence : 0.1-1 ms                                   │
│  • Throughput : 500-5000 MB/s                           │
│  • Coût : Élevé                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│         Stockage Objet (Moteur S3)                      │
│                                                         │
│  MariaDB ──► S3 API ──► AWS S3 / MinIO                  │
│             (HTTPS)      (stockage objet cloud)         │
│                                                         │
│  • Latence : 10-100 ms                                  │
│  • Throughput : 100-500 MB/s                            │
│  • Coût : Très faible                                   │
│  • Scalabilité : Illimitée                              │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture du moteur S3

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────┐
│                   MariaDB Server                        │
│  ┌────────────────────────────────────────────────────┐ │
│  │           SQL Layer (Parser, Optimizer)            │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                         ↓ Handler API
┌─────────────────────────────────────────────────────────┐
│                  S3 Storage Engine                      │
│                                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │         Table Metadata (local)                     │ │
│  │  • Structure de table (.frm)                       │ │
│  │  • Index metadata                                  │ │
│  │  • Statistiques                                    │ │
│  └────────────────────────────────────────────────────┘ │
│                         ↓                               │
│  ┌────────────────────────────────────────────────────┐ │
│  │      S3 Client (libcurl + AWS SDK)                 │ │
│  │  • HTTP/HTTPS connexion                            │ │
│  │  • Authentication (Access Key/Secret)              │ │
│  │  • Retry logic                                     │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                         ↓ HTTPS
┌─────────────────────────────────────────────────────────┐
│              Stockage Objet S3                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Bucket: my-mariadb-archive                        │ │
│  │  ├── database1/                                    │ │
│  │  │   ├── table1.frm                                │ │
│  │  │   ├── table1/                                   │ │
│  │  │   │   ├── block-000001                          │ │
│  │  │   │   ├── block-000002                          │ │
│  │  │   │   └── ...                                   │ │
│  │  │   └── table2/                                   │ │
│  │  └── database2/                                    │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  • AWS S3, S3-compatible (MinIO, Ceph)                  │
│  • Réplication automatique (durabilité 99.999999999%)   │
│  • Versioning, encryption                               │
└─────────────────────────────────────────────────────────┘
```

### Structure des données sur S3

Les tables S3 sont stockées sous forme de **blocs compressés** dans des objets S3 :

```
Organisation sur S3 :
┌────────────────────────────────────────────────────────┐
│ s3://my-bucket/mariadb/mydb/orders_archive/            │
│                                                        │
│ ├── aria                          # Metadata Aria      │
│ ├── frm                           # Structure table    │
│ ├── index                         # Index data         │
│ └── data/                         # Données par blocs  │
│     ├── 000001                    # Block 1 (1 MB)     │
│     ├── 000002                    # Block 2 (1 MB)     │
│     ├── 000003                                         │
│     └── ...                                            │
└────────────────────────────────────────────────────────┘

Chaque bloc :
• Taille : 1-16 MB (configurable)
• Format : Compressed Aria format
• Immuable : Read-only après création
```

### Caractéristiques clés

**1. Read-Only** : Les tables S3 sont en **lecture seule**

```sql
-- Créer une table S3
CREATE TABLE archive_2023 (
    id INT,
    data TEXT
) ENGINE=S3;

-- ✅ Lecture OK
SELECT * FROM archive_2023 WHERE id = 42;

-- ❌ Écriture interdite
INSERT INTO archive_2023 VALUES (1, 'Data');
-- ERROR: Table 'archive_2023' is read only

UPDATE archive_2023 SET data = 'New' WHERE id = 1;
-- ERROR: Table 'archive_2023' is read only

DELETE FROM archive_2023 WHERE id = 1;
-- ERROR: Table 'archive_2023' is read only
```

💡 **Raison** : Immutabilité garantit la cohérence avec le stockage objet distribué.

**2. Compression automatique**

Toutes les données sont compressées avant envoi sur S3 :

```
Pipeline de stockage S3 :
┌───────────────────────────────────────────────┐
│ 1. Données d'origine (table Aria/InnoDB)      │
│    1 GB non compressé                         │
│    ↓                                          │
│ 2. Compression (zlib/lz4)                     │
│    300 MB compressé (ratio 3.3×)              │
│    ↓                                          │
│ 3. Découpage en blocs (1 MB chacun)           │
│    300 blocs de 1 MB                          │
│    ↓                                          │
│ 4. Upload vers S3                             │
│    PUT s3://bucket/table/data/000001          │
│    PUT s3://bucket/table/data/000002          │
│    ...                                        │
└───────────────────────────────────────────────┘

Bénéfices :
• Réduction coûts stockage (3-5×)
• Réduction transferts réseau
• Décompression à la volée lors des lectures
```

**3. Discovery automatique**

MariaDB peut découvrir automatiquement les tables S3 :

```sql
-- Découvrir toutes les tables dans un bucket S3
CALL mysql.discover_s3_tables('my-bucket', 'mariadb/');

-- Résultat : Création automatique des définitions de tables
-- dans le schema courant, pointant vers S3
```

---

## Configuration

### Prérequis : Bucket S3

**AWS S3** :

```bash
# Créer un bucket S3 (AWS CLI)
aws s3api create-bucket \
    --bucket my-mariadb-archive \
    --region eu-west-1 \
    --create-bucket-configuration LocationConstraint=eu-west-1

# Configurer lifecycle (optionnel : transition vers Glacier après 90 jours)
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-mariadb-archive \
    --lifecycle-configuration file://lifecycle.json

# lifecycle.json :
{
  "Rules": [
    {
      "Id": "Archive-to-Glacier",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ]
    }
  ]
}

# Permissions IAM (créer un utilisateur avec accès S3)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-mariadb-archive",
        "arn:aws:s3:::my-mariadb-archive/*"
      ]
    }
  ]
}
```

**MinIO (alternative open-source)** :

```bash
# Déployer MinIO (Docker)
docker run -d \
    -p 9000:9000 \
    -p 9001:9001 \
    --name minio \
    -e "MINIO_ROOT_USER=minioadmin" \
    -e "MINIO_ROOT_PASSWORD=minioadmin123" \
    -v /data/minio:/data \
    quay.io/minio/minio server /data --console-address ":9001"

# Créer un bucket via mc (MinIO Client)
mc alias set myminio http://localhost:9000 minioadmin minioadmin123
mc mb myminio/mariadb-archive
mc policy set download myminio/mariadb-archive

# Ou via interface web : http://localhost:9001
```

### Configuration MariaDB

```ini
# /etc/mysql/mariadb.conf.d/s3.cnf
[mysqld]
# ════════════════════════════════════════════════════════
# S3 STORAGE ENGINE CONFIGURATION
# ════════════════════════════════════════════════════════

# Activer le moteur S3
plugin_load_add = ha_s3

# ──────────────────────────────────────────────────────
# Connexion S3
# ──────────────────────────────────────────────────────

# AWS S3 (production)
s3_access_key = AKIAIOSFODNN7EXAMPLE
s3_secret_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
s3_region = eu-west-1
s3_bucket = my-mariadb-archive
s3_host_name = s3.eu-west-1.amazonaws.com

# MinIO (alternative - commenter AWS ci-dessus si MinIO)
# s3_access_key = minioadmin
# s3_secret_key = minioadmin123
# s3_region = us-east-1
# s3_bucket = mariadb-archive
# s3_host_name = localhost:9000
# s3_protocol_version = Amazon    # ou Original pour MinIO ancien
# s3_use_http = ON                # Pour MinIO local sans SSL

# ──────────────────────────────────────────────────────
# Performance et optimisations
# ──────────────────────────────────────────────────────

# Taille des blocs (1-16 MB)
# Plus grand = moins d'objets S3, mais moins de granularité
s3_block_size = 4M               # Défaut : 4 MB, recommandé 4-8 MB

# Nombre de threads de lecture parallèles
s3_pagecache_buffer_size = 128M  # Cache local pour blocs S3

# Retry en cas d'erreur réseau
s3_protocol_version = Auto       # Auto, Amazon, Original

# Compression
s3_compression_algorithm = zlib  # zlib (défaut) ou none

# ──────────────────────────────────────────────────────
# Sécurité
# ──────────────────────────────────────────────────────

# SSL/TLS
s3_use_http = OFF                # ON = HTTP, OFF = HTTPS (recommandé)

# Vérifier certificats SSL
# s3_ssl_verify_server = ON

# Path vers certificats CA (si nécessaire)
# s3_ssl_ca_file = /etc/ssl/certs/ca-bundle.crt
```

**Configuration alternative : Fichiers de configuration** :

```ini
# Alternative : Stocker credentials dans ~/.aws/credentials
# (plus sécurisé que my.cnf)
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Dans my.cnf :
# s3_use_aws_credentials_file = ON
```

### Vérification

```sql
-- Vérifier que S3 est chargé
SHOW ENGINES;
-- +---------+---------+---------------------------+
-- | Engine  | Support | Comment                   |
-- +---------+---------+---------------------------+
-- | S3      | YES     | S3 storage engine         |
-- +---------+---------+---------------------------+

-- Vérifier la configuration
SHOW VARIABLES LIKE 's3%';
-- +---------------------------+---------------------------+
-- | Variable_name             | Value                     |
-- +---------------------------+---------------------------+
-- | s3_access_key             | AKIA...                   |
-- | s3_block_size             | 4194304                   |
-- | s3_bucket                 | my-mariadb-archive        |
-- | s3_host_name              | s3.eu-west-1.amazonaws.com|
-- | s3_region                 | eu-west-1                 |
-- +---------------------------+---------------------------+

-- Tester la connexion
CREATE TABLE s3_test (id INT, data VARCHAR(100)) ENGINE=S3;
-- Si erreur → Vérifier credentials, network, bucket
```

---

## Utilisation du moteur S3

### Pattern 1 : Archivage de tables existantes

**Cas d'usage** : Archiver des données anciennes (>1 an) rarement consultées.

```sql
-- Étape 1 : Table existante InnoDB
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    order_date DATE,
    customer_id INT,
    amount DECIMAL(10,2)
) ENGINE=InnoDB;

-- 10 millions de commandes sur 5 ans

-- Étape 2 : Identifier les données à archiver
SELECT
    YEAR(order_date) AS year,
    COUNT(*) AS num_orders,
    SUM(amount) AS total_amount
FROM orders
GROUP BY YEAR(order_date);
-- +------+------------+--------------+
-- | year | num_orders | total_amount |
-- +------+------------+--------------+
-- | 2020 |    500000  | 5000000.00   |  ← À archiver
-- | 2021 |    800000  | 8000000.00   |  ← À archiver
-- | 2022 |   1200000  | 12000000.00  |  ← À archiver
-- | 2023 |   2000000  | 20000000.00  |  ← Peut-être
-- | 2024 |   3500000  | 35000000.00  |  ← Garder InnoDB (actif)
-- | 2025 |   2000000  | 20000000.00  |  ← Garder InnoDB (actif)
-- +------+------------+--------------+

-- Étape 3 : Créer une table temporaire Aria avec données à archiver
CREATE TABLE orders_to_archive (
    order_id INT PRIMARY KEY,
    order_date DATE,
    customer_id INT,
    amount DECIMAL(10,2)
) ENGINE=Aria TRANSACTIONAL=0
SELECT * FROM orders
WHERE order_date < '2023-01-01';
-- 3.5 millions de lignes copiées

-- Étape 4 : Convertir en table S3 (upload vers S3)
ALTER TABLE orders_to_archive ENGINE=S3;
-- MariaDB :
-- 1. Compresse les données
-- 2. Upload vers s3://my-mariadb-archive/mydb/orders_to_archive/
-- 3. Transforme la table en lecture seule

-- Étape 5 : Renommer pour usage final
RENAME TABLE
    orders TO orders_current,
    orders_to_archive TO orders_archive;

-- Étape 6 : Supprimer les anciennes données de la table active
DELETE FROM orders_current
WHERE order_date < '2023-01-01';
-- Libère ~2 GB d'espace InnoDB

-- Résultat :
-- • orders_current (InnoDB) : 5.5M lignes, données 2023-2025
-- • orders_archive (S3) : 3.5M lignes, données 2020-2022
-- • Économie : ~2 GB SSD → S3 (500 MB compressé)
-- • Coût : $20/mois → $0.50/mois (économie 97.5%)
```

### Pattern 2 : Archivage incrémental annuel

**Cas d'usage** : Archiver automatiquement chaque année les données N-2.

```sql
-- Procédure stockée d'archivage annuel
DELIMITER $$

CREATE PROCEDURE archive_old_orders()
BEGIN
    DECLARE archive_year INT;
    DECLARE archive_table_name VARCHAR(100);

    -- Année à archiver (2 ans en arrière)
    SET archive_year = YEAR(CURDATE()) - 2;
    SET archive_table_name = CONCAT('orders_', archive_year);

    -- Créer table temporaire avec données de l'année
    SET @sql = CONCAT(
        'CREATE TABLE ', archive_table_name, '_temp ',
        'ENGINE=Aria TRANSACTIONAL=0 ',
        'SELECT * FROM orders ',
        'WHERE YEAR(order_date) = ', archive_year
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    -- Convertir en S3
    SET @sql = CONCAT(
        'ALTER TABLE ', archive_table_name, '_temp ENGINE=S3'
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    -- Renommer
    SET @sql = CONCAT(
        'RENAME TABLE ', archive_table_name, '_temp TO ', archive_table_name
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    -- Supprimer de la table principale
    DELETE FROM orders WHERE YEAR(order_date) = archive_year;

    SELECT CONCAT('Archived ', archive_year, ' to S3') AS result;
END$$

DELIMITER ;

-- Exécuter manuellement ou via cron
CALL archive_old_orders();
-- Archived 2023 to S3

-- Planifier avec Event Scheduler
CREATE EVENT annual_archive
ON SCHEDULE EVERY 1 YEAR
STARTS '2026-01-01 02:00:00'
DO CALL archive_old_orders();
```

### Pattern 3 : Vue unifiée (données chaudes + froides)

**Cas d'usage** : Requêtes transparentes sur données actuelles et archivées.

```sql
-- Créer une vue UNION ALL
CREATE VIEW orders_all AS
SELECT * FROM orders_current     -- InnoDB (données chaudes)
UNION ALL
SELECT * FROM orders_2023        -- S3 (archives)
UNION ALL
SELECT * FROM orders_2022        -- S3
UNION ALL
SELECT * FROM orders_2021;       -- S3

-- Requête transparente sur toutes les données
SELECT
    YEAR(order_date) AS year,
    COUNT(*) AS num_orders,
    SUM(amount) AS total_amount
FROM orders_all
WHERE customer_id = 12345
GROUP BY YEAR(order_date)
ORDER BY year;

-- MariaDB optimise :
-- • Accède à orders_current (InnoDB, rapide)
-- • Accède aux archives S3 seulement si nécessaire
```

### Pattern 4 : Import direct depuis S3

**Cas d'usage** : Partager des tables entre plusieurs serveurs MariaDB.

```sql
-- Serveur A : Créer et peupler une table S3
CREATE TABLE shared_reference_data (
    code VARCHAR(50),
    description TEXT,
    category VARCHAR(100)
) ENGINE=Aria;

INSERT INTO shared_reference_data VALUES
    ('PROD001', 'Product 1', 'Category A'),
    ('PROD002', 'Product 2', 'Category B'),
    -- ... millions de lignes ...
;

-- Convertir en S3
ALTER TABLE shared_reference_data ENGINE=S3;
-- Upload vers s3://my-bucket/mydb/shared_reference_data/

-- Serveur B : Importer la même table depuis S3
CREATE TABLE shared_reference_data (
    code VARCHAR(50),
    description TEXT,
    category VARCHAR(100)
) ENGINE=S3
CONNECTION='s3://my-bucket/mydb/shared_reference_data/';

-- Serveur C : Idem
-- Tous les serveurs partagent les mêmes données S3 (read-only)
```

---

## Performance et optimisations

### Latence et throughput

**Comparaison des performances** :

| Opération | InnoDB (SSD local) | S3 (AWS) | S3 (MinIO local) |
|-----------|-------------------|----------|------------------|
| SELECT 1 row | 0.1 ms | 15-50 ms | 5-15 ms |
| SELECT 1000 rows | 1 ms | 50-150 ms | 20-80 ms |
| Full table scan (1 GB) | 2 sec | 20-60 sec | 10-30 sec |
| Sequential read throughput | 500 MB/s | 50-100 MB/s | 100-300 MB/s |

💡 **Conclusion** : S3 est 10-50× plus lent qu'InnoDB pour lectures ponctuelles, mais acceptable pour scans de grandes tables rarement accédées.

### Optimisations

#### 1. Taille de bloc optimale

```ini
# my.cnf
# Blocs plus grands = moins d'objets S3 = moins de latence réseau
s3_block_size = 8M  # Au lieu de 4M par défaut

# Compromis :
# • 1-2 MB : Granularité fine, plus de requests
# • 4-8 MB : Équilibré (recommandé)
# • 16 MB : Moins de requests, mais over-fetch
```

**Exemple** :

```sql
-- Table 1 GB avec block_size=1MB :
-- • 1000 objets S3
-- • Lecture 100 MB → 100 GET requests

-- Table 1 GB avec block_size=8MB :
-- • 125 objets S3
-- • Lecture 100 MB → 13 GET requests (8× moins)
-- → Latence réduite
```

#### 2. Cache local (s3_pagecache_buffer_size)

```ini
# Cache local pour blocs S3
s3_pagecache_buffer_size = 512M  # Augmenter si RAM disponible

# Effet : Blocs S3 fréquemment accédés restent en cache
# → Pas de re-téléchargement
```

#### 3. Partitionnement logique

```sql
-- Au lieu d'une énorme table S3 :
-- orders_archive (10 ans, 100 GB, lent)

-- Préférer plusieurs petites tables :
-- orders_2020 (10 GB, S3)
-- orders_2021 (12 GB, S3)
-- orders_2022 (15 GB, S3)
-- ...

-- Avantages :
-- • Requêtes filtrent par année → Accès 1 seule table
-- • Moins de données scannées
-- • Meilleure performance
```

#### 4. Compression

```sql
-- Vérifier le ratio de compression
SELECT
    table_name,
    data_length / 1024 / 1024 AS original_mb,
    data_free / 1024 / 1024 AS compressed_mb,
    ROUND(data_length / data_free, 2) AS compression_ratio
FROM information_schema.tables
WHERE engine = 'S3';

-- Si ratio faible (<2×) :
-- • Données déjà compressées (JPEG, vidéos, etc.)
-- • Colonnes aléatoires (UUID, hashes)
-- → S3 moins intéressant (coût vs performance)
```

### Monitoring

```sql
-- Statistiques S3
SHOW STATUS LIKE 's3%';
-- +---------------------------+----------+
-- | Variable_name             | Value    |
-- +---------------------------+----------+
-- | S3_pagecache_read_requests| 1000000  |
-- | S3_pagecache_reads        | 50000    |  ← Cache misses
-- | S3_objects_loaded         | 45000    |  ← GET requests
-- +---------------------------+----------+

-- Hit ratio cache
SELECT
    (1 - (S3_pagecache_reads / S3_pagecache_read_requests)) * 100
    AS cache_hit_ratio;
-- Objectif : > 90% pour tables fréquemment accédées

-- Coût AWS (approximatif)
-- GET requests : 50000 × $0.0004 / 1000 = $0.02
-- Transfert sortant : 50 GB × $0.09 = $4.50
-- Total : $4.52
```

---

## Cas d'usage et stratégies

### 1. Archivage conformité (RGPD, SOX, HIPAA)

```sql
-- Conserver 7 ans d'historique pour conformité
-- Années N-1 à N-7 sur S3 (read-only = immuable)

-- orders_current (InnoDB) : Année N
-- orders_2024 (S3) : Année N-1
-- orders_2023 (S3) : Année N-2
-- ...
-- orders_2018 (S3) : Année N-7

-- Après 7 ans : Suppression automatique
DROP TABLE orders_2017;  -- Supprime aussi de S3
```

### 2. Data Lake / Data Warehouse hybride

```sql
-- Architecture 3-tiers :

-- Tier 1 : Hot data (InnoDB SSD)
-- • Derniers 3 mois
-- • Accès temps réel
-- • Coût : $$$

-- Tier 2 : Warm data (InnoDB HDD ou S3 MinIO local)
-- • 3 mois à 2 ans
-- • Accès occasionnel
-- • Coût : $$

-- Tier 3 : Cold data (S3 AWS)
-- • Plus de 2 ans
-- • Accès rare (compliance, analytics)
-- • Coût : $

-- + Tier 4 : Glacier (optionnel)
-- • Plus de 5 ans
-- • Archivage long terme
-- • Coût : ¢
```

### 3. Multi-region disaster recovery

```sql
-- Configuration multi-region S3
-- my.cnf sur serveur principal (EU) :
s3_region = eu-west-1
s3_bucket = mariadb-eu

-- Réplication S3 cross-region (AWS Console) :
-- mariadb-eu → mariadb-us (replica)

-- my.cnf sur serveur DR (US) :
s3_region = us-east-1
s3_bucket = mariadb-us

-- En cas de disaster :
-- Serveur US lit depuis mariadb-us (réplica)
-- RTO : Quelques minutes (pas de restore nécessaire)
```

### 4. Partage de données entre environnements

```sql
-- Production : Créer dataset de référence
CREATE TABLE product_catalog_2024 (...) ENGINE=Aria;
-- Peupler avec données
ALTER TABLE product_catalog_2024 ENGINE=S3;

-- QA : Importer depuis S3
CREATE TABLE product_catalog_2024 (...) ENGINE=S3
    CONNECTION='s3://prod-bucket/catalog/';

-- Staging : Idem
CREATE TABLE product_catalog_2024 (...) ENGINE=S3
    CONNECTION='s3://prod-bucket/catalog/';

-- Avantages :
-- ✅ Pas de duplication de données
-- ✅ Synchronisation automatique
-- ✅ Read-only (pas de corruption par QA)
```

---

## Sécurité et conformité

### Chiffrement

```ini
# my.cnf - Chiffrement en transit
s3_use_http = OFF  # Force HTTPS

# Chiffrement au repos (AWS S3)
# Activé par défaut sur AWS S3 (AES-256)
# Ou configurer SSE-KMS pour gestion de clés avancée
```

**Configuration AWS S3 SSE-KMS** :

```bash
# Créer une clé KMS
aws kms create-key --description "MariaDB S3 encryption key"

# Activer chiffrement par défaut sur bucket
aws s3api put-bucket-encryption \
    --bucket my-mariadb-archive \
    --server-side-encryption-configuration '{
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "aws:kms",
                    "KMSMasterKeyID": "arn:aws:kms:eu-west-1:123456789:key/abc-def"
                }
            }
        ]
    }'
```

### Contrôle d'accès

```sql
-- Limiter l'accès aux tables S3
GRANT SELECT ON archive.orders_2020 TO 'readonly_user'@'%';
REVOKE INSERT, UPDATE, DELETE ON archive.* FROM 'readonly_user'@'%';

-- Utilisateur ne peut que lire les archives S3
-- (déjà read-only au niveau moteur, mais double sécurité)
```

**IAM AWS** :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MariaDBReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-mariadb-archive",
        "arn:aws:s3:::my-mariadb-archive/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### Audit et logging

```sql
-- Activer l'audit des accès S3
SET GLOBAL general_log = ON;
SET GLOBAL log_output = 'TABLE';

-- Requêtes sur tables S3 loggées dans mysql.general_log
SELECT * FROM mysql.general_log
WHERE argument LIKE '%orders_2020%'
  AND command_type = 'Query';
```

**CloudTrail (AWS)** :

```bash
# Activer CloudTrail pour auditer accès S3
aws cloudtrail create-trail \
    --name mariadb-s3-audit \
    --s3-bucket-name audit-logs-bucket

# Logs de tous les accès aux objets S3
# GET, PUT, DELETE, LIST → Audit trail
```

---

## Limitations et considérations

### Limitations techniques

| Limitation | Description | Workaround |
|------------|-------------|------------|
| **Read-only** | Pas d'INSERT/UPDATE/DELETE | Utiliser Aria intermédiaire puis ALTER |
| **Latence** | 10-100× plus lent qu'InnoDB | Réserver aux données froides |
| **Pas de transactions** | Pas de rollback | Inapplicable (read-only) |
| **Pas d'index** | Pas d'index secondaires | Full table scan uniquement |
| **Taille max bloc** | 16 MB par bloc | OK pour la plupart des cas |

### Cas où S3 n'est PAS approprié

```sql
-- ❌ Mauvais cas 1 : Données actives avec modifications fréquentes
-- NE PAS FAIRE :
CREATE TABLE users ENGINE=S3;  -- Users changent constamment

-- ❌ Mauvais cas 2 : Petites tables (< 100 MB)
-- Overhead S3 non justifié
CREATE TABLE countries (id INT, name VARCHAR(100)) ENGINE=S3;

-- ❌ Mauvais cas 3 : Requêtes point-lookup fréquentes
SELECT * FROM huge_s3_table WHERE id = 12345;  -- Lent
-- Latence S3 (50 ms) vs InnoDB (0.1 ms) = 500× plus lent

-- ❌ Mauvais cas 4 : JOINs complexes
SELECT *
FROM s3_table_a a
JOIN s3_table_b b ON a.id = b.id
JOIN s3_table_c c ON b.id = c.id;
-- Latence cumulée, très lent
```

### Coûts cachés AWS S3

```
Coûts AWS S3 à considérer :
┌───────────────────────────────────────────────────────┐
│ 1. Stockage                                           │
│    • Standard : $0.023/GB/mois (premiers 50 TB)       │
│    • Infrequent Access : $0.0125/GB/mois              │
│    • Glacier : $0.004/GB/mois                         │
│                                                       │
│ 2. Requêtes                                           │
│    • GET : $0.0004 / 1000 requests                    │
│    • PUT : $0.005 / 1000 requests                     │
│    • LIST : $0.005 / 1000 requests                    │
│                                                       │
│ 3. Transfert sortant                                  │
│    • Premiers 10 TB : $0.09/GB                        │
│    • Au-delà : dégressif                              │
│                                                       │
│ 4. Réplication cross-region (optionnelle)             │
│    • $0.02/GB transféré                               │
└───────────────────────────────────────────────────────┘

Exemple pour 1 TB, 10 000 requêtes/jour :
• Stockage : 1000 GB × $0.023 = $23/mois
• Requêtes : 300 000 GET × $0.0004/1000 = $0.12/mois
• Transfert : 50 GB × $0.09 = $4.50/mois
• Total : ~$28/mois
(vs $100-200/mois pour SSD local équivalent)
```

---

## Exemples pratiques

### Exemple 1 : E-commerce - Archivage commandes

```sql
-- Configuration annuelle : Commandes > 2 ans → S3

-- 1. Créer table intermédiaire
CREATE TABLE orders_2022_staging ENGINE=Aria
SELECT * FROM orders
WHERE order_date BETWEEN '2022-01-01' AND '2022-12-31';
-- 2.5 millions de commandes, 5 GB

-- 2. Convertir en S3
ALTER TABLE orders_2022_staging ENGINE=S3;
-- Upload vers S3 : ~1.5 GB compressé
-- Durée : 2-5 minutes

-- 3. Valider
SELECT COUNT(*) FROM orders_2022_staging;
-- 2500000

SELECT * FROM orders_2022_staging LIMIT 10;
-- Fonctionne, read-only

-- 4. Finaliser
RENAME TABLE orders_2022_staging TO orders_2022;
DELETE FROM orders WHERE order_date BETWEEN '2022-01-01' AND '2022-12-31';
OPTIMIZE TABLE orders;  -- Récupérer espace

-- Économie : 5 GB SSD → 1.5 GB S3
-- Coût : $0.50/mois → $0.035/mois (économie 93%)
```

### Exemple 2 : Logs applicatifs - Archivage mensuel

```sql
-- Logs applicatifs : 100 GB/mois
-- Conservation : 12 mois actifs (InnoDB), puis S3

-- Créer tables S3 mensuelles
CREATE TABLE app_logs_202401 ENGINE=Aria
SELECT * FROM app_logs
WHERE log_date BETWEEN '2024-01-01' AND '2024-01-31';

ALTER TABLE app_logs_202401 ENGINE=S3;

-- Répéter pour chaque mois

-- Vue unifiée
CREATE VIEW app_logs_all AS
SELECT * FROM app_logs              -- InnoDB (2025)
UNION ALL SELECT * FROM app_logs_202412  -- S3 (déc 2024)
UNION ALL SELECT * FROM app_logs_202411  -- S3 (nov 2024)
-- ... 10 autres mois

-- Requête analyse
SELECT
    DATE_FORMAT(log_date, '%Y-%m') AS month,
    severity,
    COUNT(*) AS num_logs
FROM app_logs_all
WHERE severity IN ('ERROR', 'CRITICAL')
  AND log_date >= '2024-06-01'
GROUP BY month, severity
ORDER BY month DESC;

-- Performance acceptable :
-- • Scan InnoDB actuel : 0.5 sec
-- • Scan 6 mois S3 : 10-15 sec
-- Total : ~15 sec (acceptable pour analytics)
```

### Exemple 3 : IoT sensors - Tiering automatique

```sql
-- Sensors data : 1 TB/mois
-- Tier 1 (InnoDB) : Derniers 7 jours (200 GB)
-- Tier 2 (S3) : 7 jours à 1 an (11 TB)
-- Tier 3 (Glacier) : > 1 an (archivage long terme)

-- Event quotidien
CREATE EVENT daily_sensor_archiving
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 03:00:00'
DO
BEGIN
    DECLARE archive_date DATE;
    DECLARE archive_table VARCHAR(100);

    -- Archiver données d'il y a 7 jours
    SET archive_date = CURDATE() - INTERVAL 7 DAY;
    SET archive_table = CONCAT('sensors_', DATE_FORMAT(archive_date, '%Y%m%d'));

    -- Créer table quotidienne
    SET @sql = CONCAT(
        'CREATE TABLE ', archive_table, ' ENGINE=Aria ',
        'SELECT * FROM sensor_readings ',
        'WHERE reading_date = ''', archive_date, ''''
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;

    -- Convertir en S3
    SET @sql = CONCAT('ALTER TABLE ', archive_table, ' ENGINE=S3');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;

    -- Supprimer de table principale
    DELETE FROM sensor_readings WHERE reading_date = archive_date;
END;

-- Configuration AWS Lifecycle : S3 → Glacier après 365 jours
```

---

## Migration et maintenance

### Migration InnoDB → S3

```sql
-- Checklist migration :

-- 1. ✅ Vérifier que table est bien read-only après migration
-- 2. ✅ Tester sur copie avant production
-- 3. ✅ Vérifier espace disque local suffisant (conversion via Aria)
-- 4. ✅ Valider credentials et bucket S3
-- 5. ✅ Prévoir fenêtre maintenance (conversion peut être longue)

-- Étapes :
-- 1. Backup
mysqldump mydb orders_2022 > orders_2022_backup.sql

-- 2. Créer table Aria intermédiaire
CREATE TABLE orders_2022_aria ENGINE=Aria
SELECT * FROM orders_2022;

-- 3. Convertir en S3
ALTER TABLE orders_2022_aria ENGINE=S3;

-- 4. Vérifier
SELECT COUNT(*) FROM orders_2022_aria;
SELECT * FROM orders_2022_aria LIMIT 100;

-- 5. Swap
RENAME TABLE
    orders_2022 TO orders_2022_old_innodb,
    orders_2022_aria TO orders_2022;

-- 6. Tester en production pendant 1-7 jours

-- 7. Supprimer ancienne table InnoDB
DROP TABLE orders_2022_old_innodb;
```

### Restauration S3 → InnoDB

```sql
-- Cas : Besoin de modifier des données archivées

-- 1. Créer copie InnoDB depuis S3
CREATE TABLE orders_2022_restore ENGINE=InnoDB
SELECT * FROM orders_2022;  -- S3 (read-only)

-- 2. Modifier
UPDATE orders_2022_restore
SET status = 'corrected'
WHERE order_id = 12345;

-- 3. Re-archiver si nécessaire
CREATE TABLE orders_2022_new ENGINE=Aria
SELECT * FROM orders_2022_restore;

ALTER TABLE orders_2022_new ENGINE=S3;

RENAME TABLE
    orders_2022 TO orders_2022_old,
    orders_2022_new TO orders_2022;

DROP TABLE orders_2022_old;  -- Supprime aussi de S3
```

---

## ✅ Points clés à retenir

1. **Read-only** : Tables S3 sont immuables après création (lecture seule).

2. **Coût réduit** : Stockage S3 10-20× moins cher que SSD local ($0.01 vs $0.10/GB/mois).

3. **Compression automatique** : Ratio 3-5× réduit encore les coûts et transferts.

4. **Latence élevée** : 10-100× plus lent qu'InnoDB, réservé aux données froides.

5. **Cas d'usage** : Archivage conformité, data warehouse, logs historiques, partage données.

6. **Pattern Aria → S3** : Créer table Aria temporaire, peupler, puis ALTER ENGINE=S3.

7. **Architecture hybride** : InnoDB (hot data) + S3 (cold data) + vue UNION ALL.

8. **AWS S3 ou MinIO** : AWS pour production cloud, MinIO pour on-premise ou dev.

9. **Sécurité** : HTTPS, chiffrement AES-256, IAM policies, audit CloudTrail.

10. **Limitations** : Pas d'UPDATE/DELETE, pas d'index secondaires, full scan uniquement.

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 S3 Storage Engine](https://mariadb.com/kb/en/s3-storage-engine/)
- [📖 S3 Storage Engine System Variables](https://mariadb.com/kb/en/s3-system-variables/)
- [📖 Using the S3 Storage Engine](https://mariadb.com/kb/en/using-the-s3-storage-engine/)
- [📖 S3 API](https://mariadb.com/kb/en/s3-api/)

### Documentation AWS S3

- [📖 Amazon S3 Documentation](https://docs.aws.amazon.com/s3/)
- [📖 S3 Pricing](https://aws.amazon.com/s3/pricing/)
- [📖 S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [📖 S3 Lifecycle Policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)

### MinIO (alternative open-source)

- [📖 MinIO Documentation](https://min.io/docs/)
- [📖 MinIO S3 Compatibility](https://min.io/product/s3-compatibility)
- [GitHub MinIO](https://github.com/minio/minio)

### Articles et guides

- [MariaDB S3 Engine for Archival](https://mariadb.org/s3-archival/)
- [Cost Optimization with S3 Engine](https://mariadb.com/kb/en/s3-cost-optimization/)
- [Data Tiering Strategies](https://mariadb.com/resources/blog/data-tiering/)

---

## ➡️ Section suivante

**[7.7 Moteur Vector/HNSW : Recherche vectorielle pour l'IA 🆕](/07-moteurs-de-stockage/07-moteur-vector-hnsw.md)** : Découverte du nouveau moteur Vector/HNSW pour la recherche sémantique et les applications d'IA générative (RAG, embeddings).

Puis nous terminerons avec :
- **7.8** : Comparaison et choix du moteur approprié
- **7.9** : Conversion entre moteurs (stratégies détaillées)

---

**📌 Mémo DBA** : "S3 = archivage économique de données froides. Read-only, latence élevée, mais coûts 10-20× inférieurs. Parfait pour conformité, logs historiques, data warehouse tier 3."

**🎯 Règle d'or** : Si vos données n'ont pas été consultées depuis 6+ mois → S3. Si consultation quotidienne → InnoDB. Entre les deux → Analyse coût/bénéfice.

**💰 ROI** : Pour 1 TB de données historiques, économie de ~$100/mois ($1200/an). Amortissement immédiat pour archives de plusieurs TB.

⏭️ [Moteur Vector/HNSW : Recherche vectorielle pour IA](/07-moteurs-de-stockage/07-moteur-vector-hnsw.md)

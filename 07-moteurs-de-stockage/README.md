🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7. Moteurs de Stockage

> **Niveau** : Avancé
> **Durée estimée** : 8-10 heures
> **Prérequis** : Chapitres 1-6, connaissances solides en SQL, administration système

> **Public cible** : DBA, Architectes de bases de données, DevOps avancés

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :
- Comprendre l'architecture Pluggable Storage Engine de MariaDB
- Maîtriser les caractéristiques et cas d'usage de chaque moteur (InnoDB, MyISAM, Aria, ColumnStore, S3, Vector/HNSW)
- Configurer et optimiser InnoDB pour la production (Buffer Pool, logs, paramètres avancés)
- Choisir le moteur approprié selon les besoins métier et techniques
- Effectuer des conversions entre moteurs en toute sécurité
- Exploiter le nouveau moteur Vector/HNSW pour les cas d'usage IA

---

## Introduction

Les moteurs de stockage sont au cœur de l'architecture modulaire de MariaDB. Contrairement à de nombreux SGBD qui imposent un seul moteur, MariaDB offre une **architecture pluggable** permettant d'utiliser différents moteurs selon les besoins spécifiques de chaque table.

### Pourquoi plusieurs moteurs ?

Chaque moteur de stockage est optimisé pour des cas d'usage particuliers :
- **OLTP haute performance** : InnoDB avec transactions ACID
- **Analytique et data warehouse** : ColumnStore avec compression columnar
- **Archivage à froid** : S3 pour stockage objet économique
- **Recherche vectorielle IA** : Vector/HNSW pour les embeddings
- **Tables temporaires** : Memory pour performances extrêmes
- **Compatibilité legacy** : MyISAM et Aria

Cette flexibilité permet d'optimiser **chaque table individuellement** selon son profil d'accès, son volume, et ses contraintes de cohérence.

---

## Architecture Pluggable Storage Engine

### Principe de fonctionnement

MariaDB implémente une architecture en couches séparant le **moteur SQL** (parser, optimizer, query executor) des **moteurs de stockage** (gestion physique des données).

```
┌─────────────────────────────────────────┐
│      Couche Application/Client          │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│         Couche SQL (Server Layer)       │
│  - Parser SQL                           │
│  - Query Optimizer                      │
│  - Query Executor                       │
│  - Cache & Buffers                      │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│    Storage Engine API (Handler API)     │
└─────────────────────────────────────────┘
                   ↓
┌──────────┬──────────┬───────────┬─────────┐
│ InnoDB   │ Aria     │ColumnStore│ Vector  │
└──────────┴──────────┴───────────┴─────────┘
```

### Interface Handler API

Tous les moteurs implémentent une interface commune qui expose des méthodes standardisées :

```cpp
// Opérations de base (simplifiées)
handler::open()       // Ouvrir une table
handler::close()      // Fermer une table
handler::write_row()  // Insérer un enregistrement
handler::update_row() // Mettre à jour un enregistrement
handler::delete_row() // Supprimer un enregistrement
handler::rnd_next()   // Lecture séquentielle
handler::index_read() // Lecture via index
handler::start_stmt() // Début de transaction
handler::commit()     // Valider transaction
```

💡 **Avantage** : Le moteur SQL n'a pas besoin de connaître les détails d'implémentation de chaque moteur. Il suffit d'appeler les méthodes de l'API Handler.

### Consultation des moteurs disponibles

```sql
-- Lister tous les moteurs disponibles
SHOW ENGINES;

-- Résultat type
+--------------------+---------+------------+----------+---------+
| Engine             | Support | Comment    | Trans... | XA      |
+--------------------+---------+------------+----------+---------+
| InnoDB             | DEFAULT | ACID...    | YES      | YES     |
| Aria               | YES     | Crash-safe | NO       | NO      |
| MyISAM             | YES     | Non-trans  | NO       | NO      |
| MEMORY             | YES     | Hash/RAM   | NO       | NO      |
| CSV                | YES     | CSV files  | NO       | NO      |
| ColumnStore        | YES     | Columnar   | YES      | NO      |
| S3                 | YES     | AWS S3     | NO       | NO      |
+--------------------+---------+------------+----------+---------+

-- Détails sur un moteur spécifique
SHOW ENGINE InnoDB STATUS\G
```

### Spécification du moteur par table

```sql
-- Lors de la création
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB;

-- Vérifier le moteur d'une table
SHOW CREATE TABLE users;

-- Consulter les moteurs utilisés
SELECT
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    DATA_LENGTH,
    INDEX_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb';
```

⚠️ **Important** : Le moteur par défaut depuis MariaDB 10.2 est **InnoDB**. Il est fortement recommandé de l'utiliser pour les nouvelles applications.

---

## Vue d'ensemble des moteurs principaux

### Tableau comparatif

| Moteur | Transactions | Foreign Keys | MVCC | Index Type | Cas d'usage | Nouveauté 11.8 |
|--------|--------------|--------------|------|------------|-------------|----------------|
| **InnoDB** | ✅ ACID | ✅ | ✅ | B-Tree | **OLTP**, production | Optimisations SSD |
| **MyISAM** | ❌ | ❌ | ❌ | B-Tree | Legacy, lecture seule | 🔄 Déprécié |
| **Aria** | ❌ | ❌ | ❌ | B-Tree | Tables système, crash-safe | - |
| **ColumnStore** | ✅ | ❌ | ❌ | Bitmap | **OLAP**, analytics | - |
| **S3** | ❌ | ❌ | ❌ | Read-only | **Archivage** froid | Améliorations perf |
| **Vector/HNSW** | ✅ | ✅ | ✅ | HNSW | **IA/RAG**, embeddings | 🆕 Nouveau |
| **Memory** | ❌ | ❌ | ❌ | Hash/B-Tree | Tables temporaires | - |
| **Spider** | ✅ | ✅ | ❌ | - | **Sharding** distribué | - |

### Critères de sélection

#### 1. **Besoin de transactions ACID ?**
- **OUI** → InnoDB, ColumnStore, Vector/HNSW
- **NON** → MyISAM, Aria, S3, Memory

#### 2. **Type de charge de travail ?**
- **OLTP** (lecture/écriture équilibrée, transactions courtes) → **InnoDB**
- **OLAP** (analytics, agrégations massives) → **ColumnStore**
- **Recherche vectorielle** (similarité, IA) → **Vector/HNSW**
- **Archivage** (lectures rares, coût minimal) → **S3**

#### 3. **Volume de données ?**
- **< 1 TB** → InnoDB (excellent pour la plupart des cas)
- **> 1 TB, lectures intensives** → ColumnStore
- **> 10 TB, données froides** → S3

#### 4. **Contraintes de cohérence ?**
- **Foreign Keys requises** → InnoDB ou Vector/HNSW
- **Pas de FK** → Tous les autres moteurs

---

## InnoDB : Le moteur par défaut

InnoDB est le moteur **transactionnel** par excellence de MariaDB, optimisé pour les charges OLTP avec des garanties ACID complètes.

### Caractéristiques principales

#### ✅ Transactions ACID complètes

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Garanties ACID :
-- Atomicity : Les deux UPDATE ou aucun
-- Consistency : Contraintes respectées
-- Isolation : Autres transactions ne voient pas état intermédiaire
-- Durability : Persistance garantie après COMMIT

COMMIT;
```

#### ✅ Row-level locking

InnoDB utilise des verrous au niveau **ligne** (et non table comme MyISAM), permettant une concurrence maximale :

```sql
-- Session 1
BEGIN;
UPDATE orders SET status = 'shipped' WHERE id = 100;
-- Seule la ligne id=100 est verrouillée

-- Session 2 (exécution simultanée)
UPDATE orders SET status = 'shipped' WHERE id = 200;
-- Succès immédiat : ligne différente
```

#### ✅ Foreign Keys et intégrité référentielle

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
) ENGINE=InnoDB;

-- Intégrité garantie par le moteur
DELETE FROM users WHERE id = 1;
-- Supprime automatiquement les orders associées
```

#### ✅ Crash Recovery automatique

En cas de crash serveur, InnoDB utilise ses logs pour :
1. **Rejouer** les transactions validées non encore écrites (redo log)
2. **Annuler** les transactions non validées (undo log)

```sql
-- Après un crash et redémarrage
-- InnoDB affiche dans l'error log :
[Note] InnoDB: Starting crash recovery.
[Note] InnoDB: Last MySQL binlog file position 0 1234567
[Note] InnoDB: Recovery completed
```

### Architecture InnoDB en détail

#### Buffer Pool : Le cache mémoire central

Le **Buffer Pool** est la zone mémoire où InnoDB met en cache les pages de données et d'index.

```sql
-- Configuration Buffer Pool (my.cnf)
[mysqld]
# Règle générale : 70-80% de la RAM disponible
innodb_buffer_pool_size = 16G

# Instances multiples pour réduire contention (1 par 1GB, max 64)
innodb_buffer_pool_instances = 16

# Préchargement au démarrage (après crash)
innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1
```

**Fonctionnement :**

```
┌───────────────── Buffer Pool (RAM) ────────────────┐
│                                                    │
│  ┌──────────────────────────────────────────┐      │
│  │   Pages de données (16KB chacune)        │      │
│  │   ┌────┐ ┌────┐ ┌────┐ ┌────┐            │      │
│  │   │Pg 1│ │Pg 2│ │Pg 3│ │Pg N│   ...      │      │
│  │   └────┘ └────┘ └────┘ └────┘            │      │
│  └──────────────────────────────────────────┘      │
│                                                    │
│  ┌──────────────────────────────────────────┐      │
│  │   Pages d'index                          │      │
│  └──────────────────────────────────────────┘      │
│                                                    │
│  ┌──────────────────────────────────────────┐      │
│  │   Adaptive Hash Index (AHI)              │      │
│  └──────────────────────────────────────────┘      │
└────────────────────────────────────────────────────┘
                    ↕ Flush/Load
┌────────────────────────────────────────────────────┐
│              Fichiers .ibd (disque)                │
└────────────────────────────────────────────────────┘
```

**Algorithme LRU (Least Recently Used) :**

InnoDB utilise une variante du LRU pour gérer les pages :
- **Young sublist** (37% du pool) : Pages récemment accédées
- **Old sublist** (63% du pool) : Pages candidates à l'éviction

```sql
-- Monitoring du Buffer Pool
SHOW ENGINE InnoDB STATUS\G

-- Statistiques détaillées
SELECT
    POOL_ID,
    POOL_SIZE,
    FREE_BUFFERS,
    DATABASE_PAGES,
    OLD_DATABASE_PAGES,
    MODIFIED_DATABASE_PAGES,
    PENDING_READS,
    PENDING_DECOMPRESS
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- Hit ratio (objectif > 99%)
SELECT
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS buffer_pool_hit_ratio
FROM
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_reads
     FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') AS reads,
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
     FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS requests;
```

💡 **Conseil** : Un hit ratio < 95% indique que le Buffer Pool est sous-dimensionné ou que le workload dépasse la capacité mémoire.

#### Redo Log : Garantie de durabilité

Le **Redo Log** (journal de transactions) enregistre **toutes les modifications** avant leur écriture sur disque.

```sql
-- Configuration Redo Log (my.cnf)
[mysqld]
# Taille totale des fichiers redo log
# Plus grand = moins de checkpoints, mais recovery plus lente
innodb_log_file_size = 512M
innodb_log_files_in_group = 2

# Flush policy (balance performance/durabilité)
innodb_flush_log_at_trx_commit = 1
# 0 = flush toutes les secondes (risque de perte)
# 1 = flush à chaque commit (sûr, par défaut)
# 2 = flush en cache OS à chaque commit (compromis)

# Taille du buffer en mémoire
innodb_log_buffer_size = 16M
```

**Cycle de vie d'une transaction :**

```
1. BEGIN
2. UPDATE table SET column = value WHERE id = 1
   ↓
   Écriture dans Redo Log Buffer (RAM)
   ↓
3. COMMIT
   ↓
   Flush Redo Log sur disque (fsync)
   ↓
   Transaction durable
   ↓
4. Écriture asynchrone des pages modifiées (Buffer Pool → Disque)
```

**Checkpoints :**

Périodiquement, InnoDB effectue un **checkpoint** :
- Écrit les pages modifiées (dirty pages) sur disque
- Permet de libérer de l'espace dans le Redo Log
- Réduit le temps de recovery

```sql
-- Forcer un checkpoint (rarement nécessaire)
SET GLOBAL innodb_fast_shutdown = 0;
SHUTDOWN;
```

⚠️ **Attention** : Ne jamais supprimer manuellement les fichiers `ib_logfile*` lorsque le serveur est arrêté sans avoir fait un shutdown propre.

#### Undo Log : Support des transactions et MVCC

Le **Undo Log** permet :
1. **Rollback** des transactions
2. **MVCC** (Multi-Version Concurrency Control) pour l'isolation

```sql
-- Configuration Undo Log
[mysqld]
# Emplacement des tablespaces undo (MariaDB 10.0+)
innodb_undo_directory = /var/lib/mysql/undo
innodb_undo_tablespaces = 2

# Purge automatique des anciennes versions
innodb_purge_threads = 4
innodb_max_purge_lag = 0
```

**MVCC en action :**

```sql
-- Transaction 1 (long running read)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;
-- Lit 1000€

-- Transaction 2 (en parallèle)
UPDATE accounts SET balance = 2000 WHERE id = 1;
COMMIT;

-- Transaction 1 (reprend)
SELECT balance FROM accounts WHERE id = 1;
-- Lit toujours 1000€ grâce au Undo Log
-- (Isolation REPEATABLE READ)

COMMIT;
```

Le Undo Log conserve les **anciennes versions** des lignes pour permettre aux transactions en cours de lire des données cohérentes.

### Configuration avancée InnoDB

#### File-per-table vs System Tablespace

```sql
-- Recommandé : un fichier .ibd par table
[mysqld]
innodb_file_per_table = 1

-- Avantages :
-- - OPTIMIZE TABLE libère vraiment l'espace disque
-- - DROP TABLE plus rapide
-- - Facilite les backups/restaurations sélectives
```

```sql
-- Vérifier l'emplacement des fichiers
SELECT
    TABLE_NAME,
    ENGINE,
    CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND ENGINE = 'InnoDB';

-- Les fichiers .ibd sont dans /var/lib/mysql/mydb/
```

#### I/O et performances disque

```sql
[mysqld]
# Threads I/O pour lecture/écriture
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# Capacité I/O (IOPS) - ajuster selon disques
# HDD : 100-200, SSD : 2000-10000, NVMe : 10000+
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# Méthode de flush (Linux)
innodb_flush_method = O_DIRECT
# O_DIRECT : bypasse le cache OS (recommandé SSD)
# fsync : utilise le cache OS
# O_DSYNC : sync après chaque écriture
```

🆕 **MariaDB 11.8** : L'optimiseur de coût prend désormais mieux en compte les caractéristiques des SSD modernes pour ajuster les plans d'exécution.

#### Gestion de la concurrence

```sql
[mysqld]
# Nombre de threads pouvant entrer dans InnoDB simultanément
# Formule : 2 × (nb_CPU + nb_disques)
innodb_thread_concurrency = 0  # 0 = illimité (recommandé)

# Durée d'attente avant de dormir si concurrency atteinte
innodb_thread_sleep_delay = 10000  # microsecondes

# Nombre de spin loops avant de bloquer un thread
innodb_sync_spin_loops = 30
```

#### Online DDL et ALTER TABLE

🆕 **MariaDB 11.8** : `innodb_alter_copy_bulk` améliore les performances de reconstruction d'index.

```sql
-- Configuration
SET GLOBAL innodb_alter_copy_bulk = ON;

-- ALTER TABLE avec algorithme online
ALTER TABLE large_table
    ADD INDEX idx_email (email),
    ALGORITHM=INPLACE,  -- Sans copie de table
    LOCK=NONE;          -- Pas de lock lecture/écriture

-- Vérifier la progression
SELECT * FROM information_schema.INNODB_TRX;
```

#### Compression de données

```sql
-- Compression au niveau page (ROW_FORMAT)
CREATE TABLE compressed_data (
    id INT PRIMARY KEY,
    data TEXT
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED
  KEY_BLOCK_SIZE=8;  -- 1, 2, 4, 8, 16 KB

-- Vérifier la compression
SELECT
    TABLE_NAME,
    ROW_FORMAT,
    CREATE_OPTIONS,
    (DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 AS size_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb';
```

💡 **Conseil** : La compression réduit l'I/O mais augmente la charge CPU. À utiliser pour des données peu modifiées.

---

## MyISAM : Le moteur legacy

### Caractéristiques

MyISAM était le moteur par défaut avant MariaDB 10.2. Il est désormais **déprécié** et ne devrait être utilisé que pour :
- **Compatibilité** avec anciens systèmes
- **Tables en lecture seule** (sans écriture)
- **Full-text search** legacy (InnoDB supporte FTS depuis 10.0)

#### ❌ Limitations majeures

- **Pas de transactions**
- **Pas de Foreign Keys**
- **Table-level locking** (une seule écriture à la fois)
- **Pas de crash recovery** automatique
- **Corruption possible** après crash

```sql
-- Exemple MyISAM (à éviter)
CREATE TABLE legacy_data (
    id INT PRIMARY KEY,
    content TEXT,
    INDEX(content(100))
) ENGINE=MyISAM;

-- Lock complet de la table à chaque écriture
INSERT INTO legacy_data VALUES (1, 'data');
-- Toute la table est verrouillée !
```

### Quand utiliser MyISAM (rarement)

1. **Tables temporaires** en lecture seule
2. **Export/Import** rapide (pas de transactions = moins d'overhead)
3. **Logs** où l'ordre strict n'importe pas

🔄 **Recommandation** : Migrer vers InnoDB ou Aria pour les nouveaux projets.

---

## Aria : Le successeur crash-safe de MyISAM

### Caractéristiques

Aria est développé par MariaDB et utilisé pour les **tables système internes**. Il offre :

- ✅ **Crash recovery** (contrairement à MyISAM)
- ✅ **Checksum automatique** des données
- ✅ Meilleure **gestion du cache**
- ❌ Pas de transactions ACID
- ❌ Pas de Foreign Keys

```sql
-- Configuration Aria
[mysqld]
aria_pagecache_buffer_size = 128M  # Cache pour Aria
aria_log_file_size = 1G
aria_checkpoint_interval = 30      # Secondes

-- Utilisation
CREATE TABLE system_config (
    param VARCHAR(50) PRIMARY KEY,
    value TEXT
) ENGINE=Aria;
```

### Cas d'usage

- **Tables système** MariaDB (mysql.*)
- **Tables temporaires** (meilleures que MyISAM)
- **Charges mixtes lecture/écriture** sans besoin de transactions

💡 **Note** : Aria est utilisé automatiquement pour les tables système comme `mysql.user`, `mysql.db`, etc.

---

## ColumnStore : Analytics et OLAP

### Principe columnar

Contrairement aux moteurs row-based (InnoDB), ColumnStore stocke les données **par colonne** :

```
Row-based (InnoDB) :
[id=1, name='Alice', age=30] [id=2, name='Bob', age=25] ...

Columnar (ColumnStore) :
Column 'id':   [1, 2, 3, ...]
Column 'name': ['Alice', 'Bob', 'Charlie', ...]
Column 'age':  [30, 25, 35, ...]
```

#### ✅ Avantages pour l'analytique

1. **Compression massive** : colonnes homogènes compressent mieux
2. **I/O réduit** : lecture uniquement des colonnes nécessaires
3. **Agrégations rapides** : CPU optimisé pour opérations vectorielles

```sql
-- Création d'une table ColumnStore
CREATE TABLE sales_analytics (
    date DATE,
    product_id INT,
    region VARCHAR(50),
    revenue DECIMAL(10,2),
    quantity INT
) ENGINE=ColumnStore;

-- Requête analytique optimale
SELECT
    region,
    SUM(revenue) AS total_revenue,
    AVG(quantity) AS avg_quantity
FROM sales_analytics
WHERE date >= '2025-01-01'
GROUP BY region;
-- ColumnStore lit uniquement region, revenue, quantity, date
-- InnoDB devrait lire toutes les colonnes
```

### Configuration et gestion

```sql
-- Installation (si non incluse)
INSTALL SONAME 'ha_columnstore';

-- Monitoring
SELECT * FROM information_schema.COLUMNSTORE_TABLES;
SELECT * FROM information_schema.COLUMNSTORE_COLUMNS;

-- Maintenance
-- ColumnStore n'a pas besoin d'OPTIMIZE TABLE
-- Compression automatique en arrière-plan
```

⚠️ **Limitations** :
- Pas de Foreign Keys
- Pas d'index secondaires (scan complet optimisé)
- Mises à jour lentes (optimisé pour lectures)

### Cas d'usage typiques

- **Data Warehouse**
- **Business Intelligence** (reporting, dashboards)
- **Log analytics** (Clickhouse-like)
- **IoT time-series data**

💡 **Conseil** : Utiliser ColumnStore pour tables > 10 millions de lignes avec agrégations fréquentes.

---

## S3 : Stockage objet et archivage

🆕 **Améliorations 11.8** : Performances accrues pour les requêtes sur S3.

### Principe

Le moteur S3 permet de stocker des données MariaDB sur **AWS S3** ou **MinIO** (S3-compatible). Idéal pour :
- **Archivage de données froides**
- **Réduction des coûts** (stockage objet < SSD)
- **Compliance** (données immuables)

```sql
-- Configuration S3 (my.cnf)
[mysqld]
s3_access_key = 'YOUR_ACCESS_KEY'
s3_secret_key = 'YOUR_SECRET_KEY'
s3_region = 'us-east-1'
s3_bucket = 'mariadb-archive'
s3_host_name = 's3.amazonaws.com'  -- ou MinIO endpoint

-- Création d'une table S3
CREATE TABLE archived_orders (
    id INT,
    order_date DATE,
    customer_id INT,
    amount DECIMAL(10,2)
) ENGINE=S3;

-- Ou migration depuis InnoDB
ALTER TABLE old_orders ENGINE=S3;
```

### Caractéristiques

- ✅ **Read-only** après écriture initiale
- ✅ **Compression automatique**
- ✅ **Coût stockage minimal**
- ❌ **Latence élevée** (réseau S3)
- ❌ **Pas de transactions**
- ❌ **Pas de mises à jour** (immutable)

```sql
-- Requêtes SELECT normales
SELECT * FROM archived_orders WHERE order_date = '2024-01-01';

-- Pas d'UPDATE/DELETE possible
UPDATE archived_orders SET amount = 100;
-- ERROR: Table is read-only

-- Pour modifier : recréer la table
CREATE TABLE temp_orders ENGINE=InnoDB
    SELECT * FROM archived_orders;
-- Modifier temp_orders
ALTER TABLE temp_orders ENGINE=S3;
DROP TABLE archived_orders;
RENAME TABLE temp_orders TO archived_orders;
```

### Cas d'usage

- **Logs anciens** (> 1 an)
- **Données de conformité** (audit trails)
- **Snapshots historiques**
- **Cold data** rarement accédé

💡 **Conseil** : Partitionner par date et archiver les partitions anciennes en S3.

```sql
-- Exemple avec partitionnement
CREATE TABLE orders (
    id INT,
    order_date DATE,
    data TEXT
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2023 VALUES LESS THAN (2024) ENGINE=S3,
    PARTITION p2024 VALUES LESS THAN (2025) ENGINE=InnoDB,
    PARTITION p2025 VALUES LESS THAN (2026) ENGINE=InnoDB,
    PARTITION pmax VALUES LESS THAN MAXVALUE ENGINE=InnoDB
);
```

---

## Vector/HNSW : Recherche vectorielle pour l'IA

🆕 **Nouveauté majeure MariaDB 11.8** : Support natif des embeddings et recherche vectorielle.

### Introduction

Avec l'essor de l'IA générative et des LLMs (Large Language Models), le besoin de stocker et rechercher des **vecteurs d'embeddings** est devenu crucial pour :
- **RAG (Retrieval-Augmented Generation)**
- **Recherche sémantique**
- **Systèmes de recommandation**
- **Détection d'anomalies**

MariaDB 11.8 introduit :
- Type de données **VECTOR**
- Index **HNSW** (Hierarchical Navigable Small Worlds)
- Fonctions de distance vectorielle
- Optimisations **SIMD** (AVX2, AVX512, ARM NEON)

### Type de données VECTOR

```sql
-- Création d'une table avec embeddings
CREATE TABLE documents (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(200),
    content TEXT,
    -- Embedding OpenAI text-embedding-3-small (1536 dimensions)
    embedding VECTOR(1536) NOT NULL
) ENGINE=InnoDB;

-- Insertion avec embedding
INSERT INTO documents (title, content, embedding) VALUES (
    'Guide MariaDB',
    'MariaDB est un SGBD relationnel...',
    VEC_FromText('[0.123, -0.456, 0.789, ...]')  -- 1536 valeurs
);
```

### Index HNSW pour recherche rapide

HNSW est un algorithme de **Approximate Nearest Neighbor** (ANN) qui permet des recherches vectorielles en temps logarithmique.

```sql
-- Création d'un index HNSW
CREATE INDEX idx_embedding ON documents(embedding)
    USING HNSW
    WITH (
        M = 16,              -- Nombre de connexions par nœud (8-48)
        ef_construction = 200 -- Effort à la construction (100-500)
    );

-- Configuration runtime
SET SESSION hnsw.ef_search = 100;  -- Précision recherche (50-200)
```

**Paramètres clés :**
- **M** : Plus grand = meilleure précision, index plus volumineux
- **ef_construction** : Plus grand = construction plus lente, meilleure qualité
- **ef_search** : Plus grand = recherche plus lente, meilleure précision

### Fonctions de distance

```sql
-- Distance euclidienne (L2)
SELECT
    id,
    title,
    VEC_DISTANCE_EUCLIDEAN(embedding,
        VEC_FromText('[0.1, 0.2, ...]')) AS distance
FROM documents
ORDER BY distance
LIMIT 10;

-- Distance cosinus (similitude angulaire)
SELECT
    id,
    title,
    VEC_DISTANCE_COSINE(embedding,
        VEC_FromText('[0.1, 0.2, ...]')) AS similarity
FROM documents
ORDER BY similarity
LIMIT 10;

-- Distance Manhattan (L1)
SELECT
    id,
    title,
    VEC_DISTANCE_MANHATTAN(embedding, query_vector) AS distance
FROM documents
ORDER BY distance;
```

💡 **Conseil** : Utiliser la **distance cosinus** pour les embeddings normalisés (comme ceux d'OpenAI).

### Optimisations SIMD

🆕 **MariaDB 11.8** inclut des optimisations SIMD pour accélérer les calculs vectoriels :

- **AVX2** : Intel/AMD moderne (jusqu'à 8× plus rapide)
- **AVX512** : Intel Xeon récent (jusqu'à 16× plus rapide)
- **ARM NEON** : ARM64 (Graviton, Apple Silicon)
- **IBM Power10** : POWER10 VSX

```sql
-- Vérifier le support SIMD
SHOW VARIABLES LIKE 'have_simd%';

-- Résultat type
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| have_simd_avx2   | YES   |
| have_simd_avx512 | NO    |
| have_simd_neon   | NO    |
+------------------+-------+
```

### Cas d'usage IA/RAG

#### 1. **Recherche sémantique**

```sql
-- Application : moteur de recherche documentaire
-- 1. L'utilisateur saisit une requête
-- 2. On génère l'embedding via API (ex: OpenAI)
-- 3. On recherche les documents similaires

SELECT
    id,
    title,
    content,
    VEC_DISTANCE_COSINE(embedding, :query_embedding) AS score
FROM documents
WHERE VEC_DISTANCE_COSINE(embedding, :query_embedding) < 0.3
ORDER BY score
LIMIT 5;
```

#### 2. **RAG (Retrieval-Augmented Generation)**

```sql
-- Schéma type pour chatbot RAG
CREATE TABLE knowledge_base (
    id INT PRIMARY KEY AUTO_INCREMENT,
    source VARCHAR(200),         -- Origine du contenu
    chunk TEXT,                  -- Fragment de texte (200-500 tokens)
    embedding VECTOR(1536),      -- Embedding OpenAI
    metadata JSON,               -- Métadonnées (date, auteur, etc.)
    INDEX idx_emb(embedding) USING HNSW
) ENGINE=InnoDB;

-- Pipeline RAG :
-- 1. Recherche des chunks pertinents
-- 2. Injection dans le prompt LLM
-- 3. Génération de la réponse
```

#### 3. **Recherche d'images par similarité**

```sql
CREATE TABLE images (
    id INT PRIMARY KEY,
    filename VARCHAR(255),
    embedding VECTOR(512),  -- CLIP embeddings
    INDEX idx_img(embedding) USING HNSW
) ENGINE=InnoDB;

-- Recherche d'images similaires
SELECT id, filename
FROM images
ORDER BY VEC_DISTANCE_COSINE(embedding, :query_image_embedding)
LIMIT 20;
```

### Intégration avec LLMs

```python
# Exemple Python avec OpenAI
import openai
import mysql.connector

# Générer embedding
response = openai.Embedding.create(
    input="MariaDB est un SGBD relationnel",
    model="text-embedding-3-small"
)
embedding = response['data'][0]['embedding']

# Stocker dans MariaDB
conn = mysql.connector.connect(...)
cursor = conn.cursor()
cursor.execute(
    "INSERT INTO documents (title, embedding) VALUES (%s, VEC_FromText(%s))",
    ("Title", str(embedding))
)
conn.commit()
```

💡 **Conseil** : Normaliser les embeddings avant stockage pour optimiser la distance cosinus.

---

## Conversion entre moteurs

### Méthode ALTER TABLE

```sql
-- Conversion simple
ALTER TABLE my_table ENGINE=InnoDB;

-- Avec options
ALTER TABLE my_table
    ENGINE=InnoDB,
    ROW_FORMAT=DYNAMIC;

-- Vérifier la progression (sessions parallèles)
SHOW PROCESSLIST;
SELECT * FROM information_schema.INNODB_TRX;
```

⚠️ **Attention** : `ALTER TABLE ENGINE` effectue une **copie complète** de la table. Pour les grandes tables (> 100 GB), privilégier d'autres méthodes.

### Migration progressive avec partitionnement

```sql
-- 1. Créer une table partitionnée mixte
CREATE TABLE orders_new (
    id INT,
    order_date DATE,
    data TEXT
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023) ENGINE=S3,
    PARTITION p2023 VALUES LESS THAN (2024) ENGINE=Aria,
    PARTITION p2024 VALUES LESS THAN (2025) ENGINE=InnoDB,
    PARTITION p2025 VALUES LESS THAN (2026) ENGINE=InnoDB,
    PARTITION pmax VALUES LESS THAN MAXVALUE ENGINE=InnoDB
);

-- 2. Copier les données
INSERT INTO orders_new SELECT * FROM orders_old;

-- 3. Swap atomique
RENAME TABLE
    orders_old TO orders_backup,
    orders_new TO orders;
```

### Online Schema Change avec gh-ost

Pour les très grandes tables en production, utiliser **gh-ost** :

```bash
# Installation gh-ost
wget https://github.com/github/gh-ost/releases/download/v1.1.5/gh-ost_linux_amd64
chmod +x gh-ost_linux_amd64

# Conversion sans downtime
./gh-ost \
    --host=localhost \
    --user=admin \
    --password=secret \
    --database=mydb \
    --table=huge_table \
    --alter="ENGINE=InnoDB ROW_FORMAT=DYNAMIC" \
    --execute \
    --allow-on-master \
    --cut-over=default
```

### Migration MyISAM → InnoDB : Checklist

1. **Vérifier les dépendances**
```sql
-- Tables avec Foreign Keys (InnoDB uniquement)
SELECT
    TABLE_NAME,
    CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'mydb'
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

2. **Analyser l'espace disque**
```sql
-- InnoDB utilise plus d'espace (index + undo log)
SELECT
    TABLE_NAME,
    ENGINE,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND ENGINE = 'MyISAM';
```

3. **Tester sur une réplique**
```sql
-- Sur une réplique secondaire
ALTER TABLE test_table ENGINE=InnoDB;
-- Valider les performances
-- Puis basculer sur le master
```

4. **Convertir par lots**
```bash
# Script de conversion progressive
for table in $(mysql -Nse "SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA='mydb' AND ENGINE='MyISAM'"); do
    echo "Converting $table..."
    mysql mydb -e "ALTER TABLE $table ENGINE=InnoDB;"
done
```

### Critères de décision pour la conversion

| Depuis | Vers | Quand ? | Méthode |
|--------|------|---------|---------|
| MyISAM | InnoDB | **Toujours** (sauf legacy read-only) | ALTER TABLE |
| MyISAM | Aria | Tables système, temporaires | ALTER TABLE |
| InnoDB | ColumnStore | Analytics, agrégations (> 10M rows) | CREATE TABLE AS SELECT |
| InnoDB | S3 | Archivage données froides (> 1 an) | ALTER TABLE |
| InnoDB | Vector/HNSW | Ajout recherche vectorielle | ADD COLUMN + INDEX |

---

## Moteurs spécialisés

### Memory : Tables en RAM

```sql
CREATE TABLE session_data (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id INT,
    data TEXT,
    expires_at TIMESTAMP
) ENGINE=Memory;

-- ❌ Données volatiles (perdues au restart)
-- ✅ Performance extrême (RAM)
-- ✅ Idéal pour caches applicatifs temporaires
```

### Archive : Compression maximale

```sql
CREATE TABLE audit_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp TIMESTAMP,
    action VARCHAR(100),
    details TEXT
) ENGINE=Archive;

-- ✅ Compression jusqu'à 75%
-- ❌ INSERT only (pas d'UPDATE/DELETE)
-- ✅ Idéal pour logs append-only
```

### Spider : Sharding distribué

```sql
-- Nœud Spider (proxy)
CREATE TABLE distributed_users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=Spider
COMMENT='wrapper "mysql",
         srv "shard1 shard2 shard3",
         table "users"'
PARTITION BY KEY(id) PARTITIONS 3;

-- ✅ Sharding transparent
-- ✅ Agrégations cross-shard
-- ❌ Complexité opérationnelle élevée
```

### CONNECT : Accès données externes

```sql
-- Accès CSV
CREATE TABLE external_csv (
    col1 INT,
    col2 VARCHAR(50)
) ENGINE=CONNECT
TABLE_TYPE=CSV
FILE_NAME='/data/file.csv'
SEP_CHAR=','
HEADER=1;

-- Accès autre SGBD (ODBC)
CREATE TABLE external_oracle (
    id INT,
    name VARCHAR(100)
) ENGINE=CONNECT
TABLE_TYPE=ODBC
TABNAME='REMOTE_TABLE'
CONNECTION='DSN=OracleDB;UID=user;PWD=pass';
```

---

## ✅ Points clés à retenir

1. **Architecture Pluggable Storage Engine** : MariaDB permet de choisir un moteur différent par table selon les besoins.

2. **InnoDB est le choix par défaut** pour 95% des cas d'usage (OLTP, production, transactions ACID).

3. **Buffer Pool** : Zone mémoire critique d'InnoDB à dimensionner à 70-80% de la RAM disponible.

4. **Redo/Undo Logs** : Garantissent la durabilité (redo) et l'isolation (undo) des transactions.

5. **ColumnStore** : Moteur columnar idéal pour l'analytique (OLAP) et les agrégations massives sur grandes tables.

6. **S3** : Permet l'archivage économique de données froides sur stockage objet (AWS S3, MinIO).

7. 🆕 **Vector/HNSW** : Nouveauté 11.8 majeure pour la recherche vectorielle et les cas d'usage IA/RAG.

8. **Conversion entre moteurs** : Possible via `ALTER TABLE ENGINE`, mais nécessite une copie complète pour les grandes tables.

9. **MyISAM est déprécié** : Migrer vers InnoDB ou Aria pour bénéficier du crash recovery et des transactions.

10. **Choix du moteur** : Basé sur les contraintes (transactions, FK, volume, type de workload, coût stockage).

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Storage Engines Overview](https://mariadb.com/kb/en/storage-engines/)
- [📖 InnoDB Storage Engine](https://mariadb.com/kb/en/innodb/)
- [📖 ColumnStore](https://mariadb.com/kb/en/columnstore/)
- [📖 S3 Storage Engine](https://mariadb.com/kb/en/s3-storage-engine/)
- [📖 MariaDB Vector](https://mariadb.com/kb/en/vector-overview/) 🆕
- [📖 HNSW Index](https://mariadb.com/kb/en/hnsw-index/) 🆕
- [📖 Storage Engine API](https://mariadb.com/kb/en/storage-engine-api/)

### Articles techniques

- [InnoDB Buffer Pool Optimization](https://www.percona.com/blog/innodb-buffer-pool-optimization/)
- [Understanding MVCC in InnoDB](https://blog.jcole.us/innodb/)
- [Vector Search with MariaDB 11.8](https://mariadb.org/vector-search-rag/) 🆕
- [HNSW Algorithm Explained](https://arxiv.org/abs/1603.09320)

### Outils recommandés

- **gh-ost** : [GitHub](https://github.com/github/gh-ost) - Online schema change
- **pt-online-schema-change** : Percona Toolkit
- **mysqltuner** : Script d'audit de configuration

---

## ➡️ Section suivante

**[7.1 Vue d'ensemble : Architecture Pluggable Storage Engine](/07-moteurs-de-stockage/01-vue-ensemble-architecture.md)** : Détails techniques sur l'interface Handler API, le cycle de vie d'une requête à travers les moteurs, et l'implémentation d'un moteur custom.

Puis nous approfondirons chaque moteur dans les sections suivantes :
- **7.2** : InnoDB en profondeur
- **7.3-7.4** : MyISAM et Aria
- **7.5** : ColumnStore et analytics
- **7.6** : S3 et archivage cloud
- **7.7** : Vector/HNSW pour l'IA 🆕

---

**📌 Mémo DBA** : Pour 95% des tables, utilisez InnoDB. Pour l'analytique, ColumnStore. Pour l'IA, Vector/HNSW. Le reste est pour des cas très spécifiques.

⏭️ [Vue d'ensemble : Architecture Pluggable Storage Engine](/07-moteurs-de-stockage/01-vue-ensemble-architecture.md)

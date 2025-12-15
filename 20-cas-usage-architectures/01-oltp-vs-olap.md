🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.1 OLTP vs OLAP

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Chapitres 5-7 (Index, Transactions, Moteurs de stockage), notions d'architecture système

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Distinguer les caractéristiques fondamentales des workloads OLTP et OLAP
- Choisir le moteur de stockage adapté (InnoDB vs ColumnStore) selon le cas d'usage
- Configurer MariaDB 11.8 de manière optimale pour chaque type de workload
- Concevoir une architecture HTAP (Hybrid) combinant les deux approches
- Dimensionner et monitorer les performances selon le profil de charge

---

## Introduction

La distinction entre OLTP (Online Transaction Processing) et OLAP (Online Analytical Processing) est fondamentale dans le monde des bases de données. Ces deux paradigmes répondent à des besoins radicalement différents et nécessitent des architectures, des configurations et souvent des moteurs de stockage distincts.

MariaDB 11.8 LTS excelle dans les deux domaines grâce à son architecture pluggable de moteurs de stockage : **InnoDB** pour l'OLTP et **ColumnStore** pour l'OLAP. Comprendre quand et comment utiliser chacun est essentiel pour concevoir des systèmes performants.

### Le spectre des workloads

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SPECTRE DES WORKLOADS DATA                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  OLTP                      HTAP                        OLAP                 │
│  ◄────────────────────────────────────────────────────────────────────────► │
│                                                                             │
│  Transactions              Hybride                     Analytique           │
│  ───────────              ────────                     ──────────           │
│  • E-commerce             • Real-time                  • Data Warehouse     │
│  • Banking                  dashboards                 • Business Intel.    │
│  • Reservations           • Operational                • Reporting          │
│  • Gaming                   analytics                  • Data Mining        │
│  • IoT ingestion          • Monitoring                 • Machine Learning   │
│                                                                             │
│  Moteur : InnoDB          Moteur : InnoDB +            Moteur : ColumnStore │
│                           Replica ColumnStore                               │
│                                                                             │
│  Latence : < 10ms         Latence : 10ms - 1s          Latence : 1s - 10min │
│  QPS : 10K - 500K         QPS : 1K - 10K               QPS : 1 - 100        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Caractéristiques fondamentales

### Comparaison détaillée OLTP vs OLAP

| Dimension | OLTP | OLAP |
|-----------|------|------|
| **Objectif** | Gérer les opérations quotidiennes | Analyser les données historiques |
| **Opérations** | INSERT, UPDATE, DELETE, SELECT par clé | SELECT avec agrégations massives |
| **Transactions** | Courtes, nombreuses, concurrentes | Longues, peu nombreuses |
| **Données** | État actuel, operationnel | Historique, consolidé |
| **Modèle** | Normalisé (3NF, BCNF) | Dénormalisé (Star, Snowflake) |
| **Volume accédé** | Quelques lignes par requête | Millions de lignes par requête |
| **Utilisateurs** | Milliers (applications) | Dizaines (analystes, BI) |
| **Disponibilité** | 99.99%+ critique | 99.9% acceptable |
| **Fraîcheur** | Temps réel | Minutes à heures acceptable |
| **Indexation** | B-Tree sur colonnes filtrées | Peu d'index, scans massifs |

### Profils de requêtes typiques

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- OLTP : Requêtes transactionnelles
-- ═══════════════════════════════════════════════════════════════════════════

-- Lecture par clé primaire (< 1ms)
SELECT id, name, email, balance 
FROM customers 
WHERE id = 12345;

-- Insertion transactionnelle
START TRANSACTION;
INSERT INTO orders (customer_id, total, status) VALUES (12345, 99.99, 'pending');
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 789;
INSERT INTO order_items (order_id, product_id, quantity, price) 
VALUES (LAST_INSERT_ID(), 789, 1, 99.99);
COMMIT;

-- Mise à jour ciblée
UPDATE customers SET last_login = NOW() WHERE id = 12345;

-- Requête paginée avec index
SELECT id, name, created_at 
FROM products 
WHERE category_id = 5 AND active = 1
ORDER BY created_at DESC 
LIMIT 20 OFFSET 40;

-- ═══════════════════════════════════════════════════════════════════════════
-- OLAP : Requêtes analytiques
-- ═══════════════════════════════════════════════════════════════════════════

-- Agrégation sur millions de lignes (secondes à minutes)
SELECT 
    d.year,
    d.quarter,
    p.category,
    c.region,
    SUM(f.quantity) AS total_quantity,
    SUM(f.revenue) AS total_revenue,
    SUM(f.profit) AS total_profit,
    COUNT(DISTINCT f.customer_key) AS unique_customers
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_customer c ON f.customer_key = c.customer_key
WHERE d.year BETWEEN 2023 AND 2025
GROUP BY d.year, d.quarter, p.category, c.region
WITH ROLLUP;

-- Analyse de tendance avec window functions
SELECT 
    month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY month) AS revenue_year_ago,
    (revenue - LAG(revenue, 12) OVER (ORDER BY month)) / 
        NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0) * 100 AS yoy_growth
FROM monthly_revenue
ORDER BY month;

-- Segmentation client (RFM)
WITH customer_metrics AS (
    SELECT 
        customer_id,
        DATEDIFF(CURRENT_DATE, MAX(order_date)) AS recency,
        COUNT(*) AS frequency,
        SUM(total_amount) AS monetary
    FROM orders
    WHERE order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 2 YEAR)
    GROUP BY customer_id
)
SELECT 
    NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
    NTILE(5) OVER (ORDER BY frequency) AS f_score,
    NTILE(5) OVER (ORDER BY monetary) AS m_score,
    COUNT(*) AS customer_count
FROM customer_metrics
GROUP BY r_score, f_score, m_score;
```

---

## Architecture OLTP avec InnoDB

### Pourquoi InnoDB pour l'OLTP ?

InnoDB est le moteur de stockage par défaut de MariaDB et le choix optimal pour les workloads transactionnels :

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        ARCHITECTURE INNODB OLTP                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Buffer Pool                                 │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │     Data Pages      │    Index Pages    │   Undo Pages      │    │   │
│  │  │   (Tables cached)   │  (B-Tree cached)  │  (MVCC versions)  │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │  • LRU + Midpoint insertion                                         │   │
│  │  • Change Buffer pour secondary indexes                             │   │
│  │  • Adaptive Hash Index (optionnel)                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                             │
│              ┌───────────────┼───────────────┐                             │
│              ▼               ▼               ▼                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────────────┐   │
│  │   Redo Log    │  │   Undo Log    │  │      Tablespace Files         │   │
│  │  (Durability) │  │    (MVCC)     │  │   (.ibd per table)            │   │
│  │               │  │               │  │                               │   │
│  │ Write-ahead   │  │ Rollback      │  │  Clustered Index              │   │
│  │ logging       │  │ Isolation     │  │  (PK = row location)          │   │
│  └───────────────┘  └───────────────┘  └───────────────────────────────┘   │
│                                                                            │
│  Propriétés ACID :                                                         │
│  ✓ Atomicity    : Undo log pour rollback                                   │
│  ✓ Consistency  : Contraintes FK, CHECK                                    │
│  ✓ Isolation    : MVCC + Row-level locking                                 │
│  ✓ Durability   : Redo log + double write buffer                           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Configuration OLTP optimisée

```ini
# /etc/mysql/mariadb.conf.d/oltp-optimized.cnf
# Configuration pour serveur dédié OLTP - 64GB RAM, SSD NVMe

[mysqld]
# ═══════════════════════════════════════════════════════════════════════════
# MOTEUR DE STOCKAGE
# ═══════════════════════════════════════════════════════════════════════════
default_storage_engine = InnoDB
innodb_file_per_table = ON

# ═══════════════════════════════════════════════════════════════════════════
# BUFFER POOL - Cœur de la performance OLTP
# ═══════════════════════════════════════════════════════════════════════════
# Règle : 70-80% de la RAM disponible
innodb_buffer_pool_size = 48G

# Instances multiples pour réduire la contention (1 par GB jusqu'à 8)
innodb_buffer_pool_instances = 8

# Taille des chunks pour redimensionnement dynamique
innodb_buffer_pool_chunk_size = 1G

# Dump/restore du buffer pool au redémarrage
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
innodb_buffer_pool_dump_pct = 75

# ═══════════════════════════════════════════════════════════════════════════
# REDO LOG - Durabilité et performance écriture
# ═══════════════════════════════════════════════════════════════════════════
# Taille totale = innodb_log_file_size × innodb_log_files_in_group
# Recommandation : 1-2 heures de writes
innodb_log_file_size = 2G
innodb_log_files_in_group = 2
innodb_log_buffer_size = 64M

# Durabilité maximale (1 = flush à chaque commit)
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

# ═══════════════════════════════════════════════════════════════════════════
# I/O ET DISQUES
# ═══════════════════════════════════════════════════════════════════════════
# Capacité I/O du disque (IOPS)
innodb_io_capacity = 4000           # SSD SATA standard
innodb_io_capacity_max = 8000       # Burst capacity

# Méthode de flush (O_DIRECT évite double buffering OS)
innodb_flush_method = O_DIRECT

# Threads I/O
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# 🆕 MariaDB 11.8 : Optimizer prenant en compte SSD
optimizer_disk_read_ratio = 0.0002  # Ratio pour SSD NVMe rapide

# ═══════════════════════════════════════════════════════════════════════════
# CONCURRENCE ET THREADS
# ═══════════════════════════════════════════════════════════════════════════
# 0 = auto-détection basée sur CPU
innodb_thread_concurrency = 0

# Purge threads pour MVCC cleanup
innodb_purge_threads = 4

# Adaptive Hash Index (bénéfique pour lookups répétitifs)
innodb_adaptive_hash_index = ON

# ═══════════════════════════════════════════════════════════════════════════
# CONNEXIONS
# ═══════════════════════════════════════════════════════════════════════════
max_connections = 500
thread_cache_size = 100
table_open_cache = 4000
table_definition_cache = 2000

# Thread pool (Enterprise ou MariaDB Community)
thread_handling = pool-of-threads
thread_pool_size = 16
thread_pool_max_threads = 1000

# ═══════════════════════════════════════════════════════════════════════════
# REQUÊTES ET MÉMOIRE SESSION
# ═══════════════════════════════════════════════════════════════════════════
# Garder ces valeurs modestes (allouées par connexion)
sort_buffer_size = 2M
join_buffer_size = 2M
read_buffer_size = 256K
read_rnd_buffer_size = 512K

# Tables temporaires
tmp_table_size = 256M
max_heap_table_size = 256M

# 🆕 MariaDB 11.8 : Contrôle espace temporaire
max_tmp_space_usage = 8G
max_total_tmp_space_usage = 32G

# ═══════════════════════════════════════════════════════════════════════════
# LOGGING
# ═══════════════════════════════════════════════════════════════════════════
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0.5
log_queries_not_using_indexes = ON
min_examined_row_limit = 1000

# Binary log pour réplication
log_bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 1G
```

### Schéma OLTP : Normalisation et indexation

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SCHÉMA OLTP E-COMMERCE NORMALISÉ
-- ═══════════════════════════════════════════════════════════════════════════

-- Customers : Table principale utilisateurs
CREATE TABLE customers (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    status ENUM('active', 'suspended', 'deleted') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_login TIMESTAMP NULL,
    
    -- Index pour lookups fréquents
    UNIQUE KEY uk_email (email),
    KEY idx_status_created (status, created_at),
    KEY idx_last_login (last_login)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4 
  COLLATE=utf8mb4_unicode_ci;

-- Addresses : Relation 1-N avec customers
CREATE TABLE addresses (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT UNSIGNED NOT NULL,
    type ENUM('billing', 'shipping') NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    street_line1 VARCHAR(255) NOT NULL,
    street_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    KEY idx_customer (customer_id),
    KEY idx_customer_type_default (customer_id, type, is_default),
    CONSTRAINT fk_address_customer 
        FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Products : Catalogue produits
CREATE TABLE products (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id INT UNSIGNED NOT NULL,
    brand_id INT UNSIGNED,
    price DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2),
    weight_kg DECIMAL(8,3),
    status ENUM('active', 'inactive', 'discontinued') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_sku (sku),
    KEY idx_category_status (category_id, status),
    KEY idx_brand (brand_id),
    KEY idx_price (price),
    
    -- 🆕 Full-text pour recherche produits
    FULLTEXT KEY ft_name_desc (name, description)
) ENGINE=InnoDB;

-- Inventory : Stock par entrepôt (haute fréquence UPDATE)
CREATE TABLE inventory (
    product_id BIGINT UNSIGNED NOT NULL,
    warehouse_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
    reserved_quantity INT NOT NULL DEFAULT 0,
    reorder_point INT DEFAULT 10,
    last_restocked TIMESTAMP NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (product_id, warehouse_id),
    KEY idx_low_stock (warehouse_id, quantity, reorder_point),
    
    CONSTRAINT fk_inventory_product 
        FOREIGN KEY (product_id) REFERENCES products(id)
) ENGINE=InnoDB;

-- Orders : Commandes (haute fréquence INSERT)
CREATE TABLE orders (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT UNSIGNED NOT NULL,
    order_number VARCHAR(20) NOT NULL,
    status ENUM('pending', 'confirmed', 'processing', 'shipped', 
                'delivered', 'cancelled', 'refunded') DEFAULT 'pending',
    shipping_address_id BIGINT UNSIGNED,
    billing_address_id BIGINT UNSIGNED,
    subtotal DECIMAL(12,2) NOT NULL,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    shipping_amount DECIMAL(10,2) DEFAULT 0,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(12,2) NOT NULL,
    currency_code CHAR(3) DEFAULT 'EUR',
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    shipped_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    
    UNIQUE KEY uk_order_number (order_number),
    KEY idx_customer_created (customer_id, created_at DESC),
    KEY idx_status_created (status, created_at),
    KEY idx_created (created_at),
    
    CONSTRAINT fk_order_customer 
        FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB;

-- Order Items : Lignes de commande
CREATE TABLE order_items (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_percent DECIMAL(5,2) DEFAULT 0,
    line_total DECIMAL(12,2) GENERATED ALWAYS AS 
        (quantity * unit_price * (1 - discount_percent/100)) STORED,
    
    KEY idx_order (order_id),
    KEY idx_product (product_id),
    
    CONSTRAINT fk_item_order 
        FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_item_product 
        FOREIGN KEY (product_id) REFERENCES products(id)
) ENGINE=InnoDB;

-- ═══════════════════════════════════════════════════════════════════════════
-- INDEX COVERING POUR REQUÊTES FRÉQUENTES
-- ═══════════════════════════════════════════════════════════════════════════

-- Index covering pour listing commandes client
ALTER TABLE orders ADD INDEX idx_customer_listing 
    (customer_id, created_at DESC, id, status, total_amount);

-- Index covering pour dashboard produits
ALTER TABLE products ADD INDEX idx_category_listing
    (category_id, status, id, name, price);
```

💡 **Conseil** : En OLTP, privilégiez des index ciblés sur les colonnes WHERE, JOIN et ORDER BY. Évitez la sur-indexation qui pénalise les écritures.

---

## Architecture OLAP avec ColumnStore

### Pourquoi ColumnStore pour l'OLAP ?

ColumnStore stocke les données par colonne plutôt que par ligne, offrant des avantages majeurs pour l'analytique :

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STOCKAGE ROW vs COLUMN                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ROW STORE (InnoDB)                  COLUMN STORE (ColumnStore)             │
│  ══════════════════                  ══════════════════════════             │
│                                                                             │
│  Table: sales                        Table: sales                           │
│  ┌────┬─────────┬────────┬──────┐   ┌─────────────────────────────────────┐ │
│  │ id │ product │  date  │amount│   │ id:     [1, 2, 3, 4, 5, ...]        │ │
│  ├────┼─────────┼────────┼──────┤   │ product:[A, B, A, C, B, ...]        │ │
│  │ 1  │    A    │2025-01 │ 100  │   │ date:   [2025-01, 2025-01, ...]     │ │
│  │ 2  │    B    │2025-01 │ 200  │   │ amount: [100, 200, 150, 300, ...]   │ │
│  │ 3  │    A    │2025-02 │ 150  │   └─────────────────────────────────────┘ │
│  │ 4  │    C    │2025-02 │ 300  │                                           │
│  │ 5  │    B    │2025-03 │ 250  │                                           │
│  └────┴─────────┴────────┴──────┘                                           │
│                                                                             │
│  SELECT SUM(amount)                  SELECT SUM(amount)                     │
│  FROM sales                          FROM sales                             │
│  WHERE date = '2025-01'              WHERE date = '2025-01'                 │
│                                                                             │
│  → Lit TOUTES les colonnes           → Lit SEULEMENT date + amount          │
│  → I/O : 100%                        → I/O : 25% (2 colonnes sur 4)         │
│                                                                             │
│  Avantages ColumnStore :                                                    │
│  ✓ Compression 5-10x (valeurs similaires par colonne)                       │
│  ✓ Vectorized execution (SIMD sur colonnes)                                 │
│  ✓ Scan efficace sur sous-ensemble de colonnes                              │
│  ✓ Agrégations massives optimisées                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configuration OLAP avec ColumnStore

```ini
# /etc/mysql/mariadb.conf.d/olap-columnstore.cnf
# Configuration pour serveur analytique - 128GB RAM

[mysqld]
# ═══════════════════════════════════════════════════════════════════════════
# MOTEUR PAR DÉFAUT
# ═══════════════════════════════════════════════════════════════════════════
default_storage_engine = Columnstore

# ═══════════════════════════════════════════════════════════════════════════
# COLUMNSTORE SPÉCIFIQUE
# ═══════════════════════════════════════════════════════════════════════════
# Import batch optimisé
columnstore_use_import_for_batchinsert = ON

# Seuil pour scan de chaînes
columnstore_string_scan_threshold = 10

# Compression (snappy par défaut, lz4 plus rapide)
columnstore_compression_type = snappy

# ═══════════════════════════════════════════════════════════════════════════
# MÉMOIRE POUR AGRÉGATIONS ET JOINTURES
# ═══════════════════════════════════════════════════════════════════════════
# Tables temporaires en mémoire (agrégations)
tmp_table_size = 8G
max_heap_table_size = 8G

# Buffers pour jointures et tris massifs
join_buffer_size = 512M
sort_buffer_size = 512M
read_buffer_size = 4M

# 🆕 MariaDB 11.8 : Limites espace temporaire
max_tmp_space_usage = 64G
max_total_tmp_space_usage = 128G

# ═══════════════════════════════════════════════════════════════════════════
# PARALLÉLISME
# ═══════════════════════════════════════════════════════════════════════════
# ColumnStore utilise ses propres mécanismes de parallélisme
# Configurer via Columnstore.xml pour le cluster

# ═══════════════════════════════════════════════════════════════════════════
# CONNEXIONS (moins nombreuses qu'en OLTP)
# ═══════════════════════════════════════════════════════════════════════════
max_connections = 100
wait_timeout = 28800        # 8 heures pour requêtes longues
interactive_timeout = 28800

# ═══════════════════════════════════════════════════════════════════════════
# OPTIMISEUR
# ═══════════════════════════════════════════════════════════════════════════
optimizer_switch = 'join_cache_hashed=on,join_cache_bka=on'

# Statistiques histogrammes pour meilleures estimations
histogram_type = DOUBLE_PREC_HB
histogram_size = 254
```

### Schéma OLAP : Star Schema

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SCHÉMA OLAP DATA WAREHOUSE - STAR SCHEMA
-- ═══════════════════════════════════════════════════════════════════════════

-- ═══════════════════════════════════════════════════════════════════════════
-- TABLES DE DIMENSION (petites, dénormalisées)
-- ═══════════════════════════════════════════════════════════════════════════

-- Dimension Temps (pré-générée pour 20 ans)
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,                    -- Format YYYYMMDD
    full_date DATE NOT NULL,
    day_of_week TINYINT NOT NULL,                -- 1-7
    day_name VARCHAR(10) NOT NULL,               -- Monday, Tuesday...
    day_of_month TINYINT NOT NULL,               -- 1-31
    day_of_year SMALLINT NOT NULL,               -- 1-366
    week_of_year TINYINT NOT NULL,               -- 1-53
    month TINYINT NOT NULL,                      -- 1-12
    month_name VARCHAR(10) NOT NULL,             -- January, February...
    quarter TINYINT NOT NULL,                    -- 1-4
    quarter_name CHAR(2) NOT NULL,               -- Q1, Q2, Q3, Q4
    year SMALLINT NOT NULL,
    year_month CHAR(7) NOT NULL,                 -- 2025-01
    year_quarter CHAR(7) NOT NULL,               -- 2025-Q1
    is_weekend BOOLEAN NOT NULL,
    is_holiday BOOLEAN DEFAULT FALSE,
    holiday_name VARCHAR(50),
    fiscal_year SMALLINT,
    fiscal_quarter TINYINT
) ENGINE=Columnstore;

-- Dimension Produit (dénormalisée avec hiérarchie)
CREATE TABLE dim_product (
    product_key INT PRIMARY KEY AUTO_INCREMENT,
    product_id VARCHAR(50) NOT NULL,             -- Business key
    sku VARCHAR(50) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    product_description TEXT,
    
    -- Hiérarchie dénormalisée
    category_id INT,
    category_name VARCHAR(100),
    subcategory_id INT,
    subcategory_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),
    
    brand_id INT,
    brand_name VARCHAR(100),
    supplier_id INT,
    supplier_name VARCHAR(200),
    
    -- Attributs
    unit_cost DECIMAL(10,2),
    unit_price DECIMAL(10,2),
    weight_kg DECIMAL(8,3),
    is_active BOOLEAN DEFAULT TRUE,
    
    -- SCD Type 2 (Slowly Changing Dimension)
    effective_date DATE NOT NULL,
    expiration_date DATE DEFAULT '9999-12-31',
    is_current BOOLEAN DEFAULT TRUE,
    
    KEY idx_product_id (product_id),
    KEY idx_current (is_current, product_id)
) ENGINE=Columnstore;

-- Dimension Client
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY AUTO_INCREMENT,
    customer_id VARCHAR(50) NOT NULL,            -- Business key
    
    -- Identité
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    full_name VARCHAR(200),
    email VARCHAR(255),
    
    -- Segmentation
    customer_type ENUM('B2C', 'B2B', 'Enterprise'),
    segment VARCHAR(50),                         -- VIP, Regular, New...
    acquisition_channel VARCHAR(50),             -- Organic, Paid, Referral...
    
    -- Géographie (dénormalisée)
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100),
    country_code CHAR(2),
    region VARCHAR(50),                          -- EMEA, APAC, Americas
    postal_code VARCHAR(20),
    
    -- Métriques pré-calculées
    lifetime_value DECIMAL(12,2),
    first_order_date DATE,
    total_orders INT,
    
    -- SCD Type 2
    effective_date DATE NOT NULL,
    expiration_date DATE DEFAULT '9999-12-31',
    is_current BOOLEAN DEFAULT TRUE,
    
    KEY idx_customer_id (customer_id),
    KEY idx_current (is_current, customer_id)
) ENGINE=Columnstore;

-- Dimension Store/Channel
CREATE TABLE dim_store (
    store_key INT PRIMARY KEY AUTO_INCREMENT,
    store_id VARCHAR(20) NOT NULL,
    store_name VARCHAR(100) NOT NULL,
    store_type ENUM('Physical', 'Online', 'Marketplace', 'Wholesale'),
    
    -- Localisation
    address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100),
    region VARCHAR(50),
    
    -- Attributs
    size_sqm INT,
    employee_count INT,
    opening_date DATE,
    manager_name VARCHAR(100),
    
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=Columnstore;

-- ═══════════════════════════════════════════════════════════════════════════
-- TABLE DE FAITS (très grande, nombreuses mesures)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE fact_sales (
    -- Clés de dimension (pas de FK en ColumnStore)
    date_key INT NOT NULL,
    product_key INT NOT NULL,
    customer_key INT NOT NULL,
    store_key INT NOT NULL,
    
    -- Clé dégénérée (référence sans dimension)
    order_id BIGINT NOT NULL,
    order_line_id BIGINT NOT NULL,
    
    -- Mesures quantitatives
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    unit_cost DECIMAL(10,2),
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    shipping_amount DECIMAL(10,2) DEFAULT 0,
    
    -- Mesures calculées (pré-agrégées pour performance)
    gross_revenue DECIMAL(12,2) NOT NULL,        -- quantity * unit_price
    net_revenue DECIMAL(12,2) NOT NULL,          -- gross - discount
    total_cost DECIMAL(12,2),                    -- quantity * unit_cost
    profit DECIMAL(12,2),                        -- net_revenue - total_cost
    
    -- Pas de clé primaire composite en ColumnStore
    -- L'order_line_id garantit l'unicité logique
    
    KEY idx_date (date_key),
    KEY idx_product (product_key),
    KEY idx_customer (customer_key)
) ENGINE=Columnstore;

-- Table de faits agrégée (pour dashboards rapides)
CREATE TABLE fact_daily_sales_summary (
    date_key INT NOT NULL,
    store_key INT NOT NULL,
    category_id INT NOT NULL,
    
    -- Mesures agrégées
    order_count INT NOT NULL,
    item_count INT NOT NULL,
    customer_count INT NOT NULL,
    gross_revenue DECIMAL(14,2) NOT NULL,
    net_revenue DECIMAL(14,2) NOT NULL,
    total_cost DECIMAL(14,2),
    profit DECIMAL(14,2),
    avg_order_value DECIMAL(10,2),
    
    KEY idx_date_store (date_key, store_key)
) ENGINE=Columnstore;

-- ═══════════════════════════════════════════════════════════════════════════
-- REQUÊTES ANALYTIQUES OPTIMISÉES
-- ═══════════════════════════════════════════════════════════════════════════

-- Analyse des ventes par trimestre et catégorie
SELECT 
    d.year,
    d.quarter_name,
    p.category_name,
    p.brand_name,
    COUNT(DISTINCT f.order_id) AS orders,
    SUM(f.quantity) AS units_sold,
    SUM(f.net_revenue) AS revenue,
    SUM(f.profit) AS profit,
    SUM(f.profit) / NULLIF(SUM(f.net_revenue), 0) * 100 AS profit_margin_pct
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year = 2025
  AND p.is_current = TRUE
GROUP BY d.year, d.quarter_name, p.category_name, p.brand_name
ORDER BY d.quarter_name, revenue DESC;

-- Top 10 clients par région avec comparaison N-1
WITH current_year AS (
    SELECT 
        c.region,
        c.customer_id,
        c.full_name,
        SUM(f.net_revenue) AS revenue_2025
    FROM fact_sales f
    JOIN dim_customer c ON f.customer_key = c.customer_key
    JOIN dim_date d ON f.date_key = d.date_key
    WHERE d.year = 2025 AND c.is_current = TRUE
    GROUP BY c.region, c.customer_id, c.full_name
),
previous_year AS (
    SELECT 
        c.customer_id,
        SUM(f.net_revenue) AS revenue_2024
    FROM fact_sales f
    JOIN dim_customer c ON f.customer_key = c.customer_key
    JOIN dim_date d ON f.date_key = d.date_key
    WHERE d.year = 2024
    GROUP BY c.customer_id
),
ranked AS (
    SELECT 
        cy.*,
        py.revenue_2024,
        (cy.revenue_2025 - COALESCE(py.revenue_2024, 0)) / 
            NULLIF(py.revenue_2024, 0) * 100 AS yoy_growth,
        ROW_NUMBER() OVER (PARTITION BY cy.region ORDER BY cy.revenue_2025 DESC) AS rank_in_region
    FROM current_year cy
    LEFT JOIN previous_year py ON cy.customer_id = py.customer_id
)
SELECT region, customer_id, full_name, revenue_2025, revenue_2024, yoy_growth
FROM ranked
WHERE rank_in_region <= 10
ORDER BY region, rank_in_region;
```

---

## Architecture HTAP (Hybrid)

Pour les cas nécessitant à la fois transactions et analytics en quasi-temps réel, MariaDB permet une architecture hybride :

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ARCHITECTURE HTAP MARIADB                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                           Applications                                      │
│                               │                                             │
│            ┌──────────────────┴───────────────────┐                         │
│            │             MaxScale                 │                         │
│            │         (Read/Write Split)           │                         │
│            └──────────────┬───────────────────────┘                         │
│                           │                                                 │
│          Writes ──────────┼────────── Reads (Analytics)                     │
│                           │                                                 │
│            ┌──────────────┴──────────────────────┐                          │
│            ▼                                     ▼                          │
│  ┌──────────────────────┐         ┌──────────────────────────┐              │
│  │   PRIMARY (InnoDB)   │         │   ANALYTICS REPLICA      │              │
│  │                      │         │   (ColumnStore)          │              │
│  │  • Transactions      │ ──────► │                          │              │
│  │  • CRUD              │  Repli- │  • Dashboards            │              │
│  │  • OLTP workload     │  cation │  • Reports               │              │
│  │                      │  Async  │  • BI Tools              │              │
│  │  Tables InnoDB       │         │  Tables ColumnStore      │              │
│  └──────────────────────┘         └──────────────────────────┘              │
│                                                                             │
│  Latence réplication : 1-60 secondes (configurable)                         │
│                                                                             │
│  Configuration requise :                                                    │
│  • binlog_format = ROW sur le Primary                                       │
│  • Replica avec tables ColumnStore                                          │
│  • MaxScale readwritesplit router                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configuration HTAP

```ini
# Primary (InnoDB) - my.cnf
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
gtid_strict_mode = ON

# Replica Analytics (ColumnStore) - my.cnf
[mysqld]
server_id = 2
read_only = ON
default_storage_engine = Columnstore
log_slave_updates = ON

# Conversion automatique InnoDB → ColumnStore à la réplication
# Utiliser un script de transformation ou Debezium
```

### Script ETL InnoDB → ColumnStore

```sql
-- Procédure de synchronisation incrementale
DELIMITER //

CREATE PROCEDURE sync_sales_to_analytics(IN p_batch_size INT)
BEGIN
    DECLARE v_last_sync TIMESTAMP;
    DECLARE v_rows_synced INT DEFAULT 0;
    
    -- Récupérer le dernier timestamp synchronisé
    SELECT COALESCE(MAX(synced_at), '1970-01-01') INTO v_last_sync
    FROM analytics.sync_log
    WHERE table_name = 'fact_sales';
    
    -- Insérer les nouvelles données
    INSERT INTO analytics.fact_sales (
        date_key, product_key, customer_key, store_key,
        order_id, order_line_id, quantity, unit_price,
        gross_revenue, net_revenue, profit
    )
    SELECT 
        CAST(DATE_FORMAT(o.created_at, '%Y%m%d') AS INT),
        p.product_key,
        c.customer_key,
        s.store_key,
        o.id,
        oi.id,
        oi.quantity,
        oi.unit_price,
        oi.line_total,
        oi.line_total,
        oi.line_total - (oi.quantity * COALESCE(p.unit_cost, 0))
    FROM production.orders o
    JOIN production.order_items oi ON o.id = oi.order_id
    JOIN analytics.dim_product p ON oi.product_id = p.product_id AND p.is_current = TRUE
    JOIN analytics.dim_customer c ON o.customer_id = c.customer_id AND c.is_current = TRUE
    JOIN analytics.dim_store s ON o.store_id = s.store_id
    WHERE o.updated_at > v_last_sync
      AND o.status NOT IN ('cancelled', 'pending')
    LIMIT p_batch_size;
    
    SET v_rows_synced = ROW_COUNT();
    
    -- Logger la synchronisation
    INSERT INTO analytics.sync_log (table_name, rows_synced, synced_at)
    VALUES ('fact_sales', v_rows_synced, NOW());
    
    SELECT v_rows_synced AS rows_synchronized;
END //

DELIMITER ;

-- Exécuter toutes les 5 minutes via Event
CREATE EVENT sync_sales_event
ON SCHEDULE EVERY 5 MINUTE
DO CALL sync_sales_to_analytics(10000);
```

---

## Métriques et monitoring

### KPIs par type de workload

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- MONITORING OLTP
-- ═══════════════════════════════════════════════════════════════════════════

-- Buffer Pool Hit Ratio (cible > 99%)
SELECT 
    (1 - (
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_ratio;

-- Transactions par seconde
SELECT 
    VARIABLE_VALUE AS transactions_per_sec
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Com_commit';

-- Requêtes par seconde
SELECT 
    SUM(VARIABLE_VALUE) AS qps
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN ('Com_select', 'Com_insert', 'Com_update', 'Com_delete');

-- Lock waits (doit être proche de 0)
SELECT 
    VARIABLE_VALUE AS row_lock_waits
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Innodb_row_lock_waits';

-- Connexions actives vs max
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_connected') AS current_connections,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME = 'max_connections') AS max_connections;

-- ═══════════════════════════════════════════════════════════════════════════
-- MONITORING OLAP (ColumnStore)
-- ═══════════════════════════════════════════════════════════════════════════

-- Queries en cours
SELECT 
    ID,
    USER,
    HOST,
    DB,
    TIME,
    STATE,
    SUBSTRING(INFO, 1, 100) AS query_preview
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND TIME > 10  -- Plus de 10 secondes
ORDER BY TIME DESC;

-- Utilisation mémoire temporaire
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') AS tmp_disk_tables,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Created_tmp_tables') AS tmp_memory_tables;
```

### Dashboard Grafana (Prometheus queries)

```yaml
# prometheus-mariadb-alerts.yaml
groups:
  - name: mariadb-oltp
    rules:
      - alert: LowBufferPoolHitRatio
        expr: |
          (1 - (mysql_global_status_innodb_buffer_pool_reads / 
                mysql_global_status_innodb_buffer_pool_read_requests)) < 0.99
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Buffer pool hit ratio below 99%"
          
      - alert: HighLockWaits
        expr: rate(mysql_global_status_innodb_row_lock_waits[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High InnoDB row lock waits"
          
      - alert: SlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 1
        for: 5m
        labels:
          severity: info
        annotations:
          summary: "Slow queries detected"

  - name: mariadb-olap
    rules:
      - alert: LongRunningQuery
        expr: mysql_info_schema_processlist_seconds{state!="Sleep"} > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Query running for more than 1 hour"
          
      - alert: HighTempDiskUsage
        expr: |
          mysql_global_status_created_tmp_disk_tables / 
          (mysql_global_status_created_tmp_tables + 0.001) > 0.25
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "More than 25% of temp tables on disk"
```

---

## Cas d'usage et décisions architecturales

### Arbre de décision

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARBRE DE DÉCISION OLTP vs OLAP                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Quel est le pattern d'accès principal ?                                    │
│  ├── Transactions fréquentes (INSERT/UPDATE/DELETE) ──► OLTP (InnoDB)       │
│  │   ├── Latence critique < 10ms ? ──► InnoDB optimisé                      │
│  │   └── Haute disponibilité requise ? ──► InnoDB + Galera                  │
│  │                                                                          │
│  ├── Lectures analytiques (agrégations) ──► OLAP (ColumnStore)              │
│  │   ├── Volume > 100M lignes ? ──► ColumnStore distribué                   │
│  │   └── Données froides archivage ? ──► ColumnStore + S3                   │
│  │                                                                          │
│  └── Mix Transactions + Analytics ──► Architecture HTAP                     │
│      ├── Analytics temps réel (< 1min) ? ──► InnoDB + Read Replicas         │
│      └── Analytics quasi temps réel (< 1h) ? ──► InnoDB → ColumnStore       │
│                                                                             │
│  Volume de données ?                                                        │
│  ├── < 100 GB ──► Single node suffisant                                     │
│  ├── 100 GB - 1 TB ──► Considérer partitionnement                           │
│  └── > 1 TB ──► ColumnStore ou sharding                                     │
│                                                                             │
│  Ratio Lecture/Écriture ?                                                   │
│  ├── Write-heavy (> 50% writes) ──► InnoDB, attention aux index             │
│  ├── Read-heavy (> 80% reads) ──► Read replicas, index covering             │
│  └── Balanced ──► Configuration équilibrée                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tableau récapitulatif

| Critère | OLTP (InnoDB) | OLAP (ColumnStore) | HTAP |
|---------|---------------|-------------------|------|
| **Use case** | E-commerce, Banking, Gaming | BI, Reporting, Data Science | Dashboards temps réel |
| **Latence cible** | < 10ms | 1s - 10min | 10ms - 1min |
| **Transactions/sec** | 1K - 100K | N/A | 1K - 10K |
| **Volume optimal** | 1GB - 500GB | 100GB - Pétaoctets | 1GB - 1TB |
| **Modèle données** | 3NF normalisé | Star/Snowflake | Dépend du besoin |
| **Index** | B-Tree, covering | Minimal | B-Tree + agrégats |
| **Compression** | 2x (page) | 5-10x (colonne) | Variable |
| **RAM/Data ratio** | 70-100% buffer pool | 10-30% pour cache | 50-70% |
| **Coût/performance** | $$$$ pour gros volumes | $ pour gros volumes | $$$ |

---

## ✅ Points clés à retenir

- **OLTP** = Transactions courtes, haute concurrence, accès par clé → **InnoDB** avec Buffer Pool optimisé (70-80% RAM)
- **OLAP** = Agrégations massives, peu de requêtes, scans de colonnes → **ColumnStore** avec modèle Star Schema
- **La normalisation** convient à l'OLTP (évite les anomalies), la **dénormalisation** convient à l'OLAP (évite les jointures)
- **Buffer Pool Hit Ratio > 99%** est l'indicateur clé de performance OLTP
- **ColumnStore** excelle sur les requêtes touchant peu de colonnes mais beaucoup de lignes
- **L'architecture HTAP** (InnoDB + réplication vers ColumnStore) permet d'avoir le meilleur des deux mondes
- 🆕 **MariaDB 11.8** améliore l'optimizer pour SSD avec `optimizer_disk_read_ratio`
- Les **tables de faits agrégées** (daily/weekly summaries) accélèrent drastiquement les dashboards OLAP

---

## 🔗 Ressources et références

- [📖 InnoDB Storage Engine - MariaDB KB](https://mariadb.com/kb/en/innodb/)
- [📖 MariaDB ColumnStore](https://mariadb.com/kb/en/mariadb-columnstore/)
- [📖 Star Schema - Kimball Group](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
- [📖 Buffer Pool Tuning](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [📖 HTAP with MariaDB](https://mariadb.com/resources/blog/hybrid-transactional-analytical-processing-htap/)

---

## ➡️ Section suivante

[20.2 Architecture Microservices](./02-architecture-microservices.md) : Découvrez les patterns Database-per-Service et Shared Database pour les architectures distribuées modernes.

⏭️ [Architecture microservices](/20-cas-usage-architectures/02-architecture-microservices.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.3 Data Warehousing avec ColumnStore

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Section 20.1 (OLTP vs OLAP), Chapitre 7 (Moteurs de stockage), notions de modélisation dimensionnelle

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre l'architecture distribuée de MariaDB ColumnStore et ses composants
- Concevoir des modèles dimensionnels (Star Schema, Snowflake) optimisés pour l'analytique
- Implémenter des pipelines ETL/ELT performants vers ColumnStore
- Optimiser les requêtes analytiques pour des performances maximales
- Dimensionner et configurer un cluster ColumnStore pour des volumes de données massifs
- Intégrer ColumnStore dans une architecture data moderne (Data Lake, Lakehouse)

---

## Introduction

Le Data Warehousing représente le cœur de la Business Intelligence et de l'analytique d'entreprise. Alors que les bases transactionnelles (OLTP) gèrent les opérations quotidiennes, le Data Warehouse consolidé permet d'analyser l'historique, de détecter des tendances et de prendre des décisions stratégiques basées sur les données.

**MariaDB ColumnStore** est un moteur de stockage colonnaire distribué, conçu spécifiquement pour les workloads analytiques (OLAP). Il permet d'exécuter des requêtes complexes sur des téraoctets voire des pétaoctets de données avec des performances exceptionnelles.

### Pourquoi ColumnStore pour le Data Warehousing ?

```
┌────────────────────────────────────────────────────────────────────────────┐
│                 POURQUOI COLUMNSTORE POUR L'ANALYTIQUE ?                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Requête analytique typique :                                              │
│  ══════════════════════════                                                │
│                                                                            │
│  SELECT region, SUM(revenue), AVG(profit_margin)                           │
│  FROM sales                           -- 500 millions de lignes            │
│  WHERE year = 2024                    -- Filtre sur 1 colonne              │
│  GROUP BY region;                     -- Agrégation sur 3 colonnes         │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ROW STORE (InnoDB)                                                 │   │
│  │  ═══════════════════                                                │   │
│  │                                                                     │   │
│  │  • Lit TOUTES les colonnes de chaque ligne                          │   │
│  │  • I/O : 100% des données (50 colonnes × 500M lignes)               │   │
│  │  • Pas de compression efficace (données hétérogènes par ligne)      │   │
│  │  • Temps : minutes à heures                                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  COLUMN STORE (ColumnStore)                                         │   │
│  │  ══════════════════════════                                         │   │
│  │                                                                     │   │
│  │  • Lit SEULEMENT les colonnes nécessaires (4 sur 50)                │   │
│  │  • I/O : 8% des données                                             │   │
│  │  • Compression 5-15x (valeurs similaires par colonne)               │   │
│  │  • Exécution vectorisée (SIMD)                                      │   │
│  │  • Temps : secondes                                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Gain typique : 10x à 100x sur les requêtes analytiques                    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture de MariaDB ColumnStore

### Composants du système

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE COLUMNSTORE DISTRIBUÉE                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                              Clients                                       │
│                                 │                                          │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      User Module (UM)                               │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  MariaDB Server                                             │    │   │
│  │  │  • Parser SQL                                               │    │   │
│  │  │  • Query Optimizer                                          │    │   │
│  │  │  • Result Aggregation                                       │    │   │
│  │  │  • Client Protocol                                          │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  ExeMgr (Execution Manager)                                 │    │   │
│  │  │  • Query Decomposition                                      │    │   │
│  │  │  • Work Distribution                                        │    │   │
│  │  │  • Result Merge                                             │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                       │
│                    Distribution du travail                                 │
│                                    │                                       │
│          ┌─────────────────────────┼─────────────────────────┐             │
│          │                         │                         │             │
│          ▼                         ▼                         ▼             │
│  ┌───────────────┐        ┌───────────────┐        ┌───────────────┐       │
│  │ Performance   │        │ Performance   │        │ Performance   │       │
│  │ Module (PM1)  │        │ Module (PM2)  │        │ Module (PM3)  │       │
│  │               │        │               │        │               │       │
│  │ ┌───────────┐ │        │ ┌───────────┐ │        │ ┌───────────┐ │       │
│  │ │ PrimProc  │ │        │ │ PrimProc  │ │        │ │ PrimProc  │ │       │
│  │ │ • Scan    │ │        │ │ • Scan    │ │        │ │ • Scan    │ │       │
│  │ │ • Filter  │ │        │ │ • Filter  │ │        │ │ • Filter  │ │       │
│  │ │ • Project │ │        │ │ • Project │ │        │ │ • Project │ │       │
│  │ └───────────┘ │        │ └───────────┘ │        │ └───────────┘ │       │
│  │               │        │               │        │               │       │
│  │ ┌───────────┐ │        │ ┌───────────┐ │        │ ┌───────────┐ │       │
│  │ │ WriteEngn │ │        │ │ WriteEngn │ │        │ │ WriteEngn │ │       │
│  │ │ • Insert  │ │        │ │ • Insert  │ │        │ │ • Insert  │ │       │
│  │ │ • Bulk    │ │        │ │ • Bulk    │ │        │ │ • Bulk    │ │       │
│  │ └───────────┘ │        │ └───────────┘ │        │ └───────────┘ │       │
│  │               │        │               │        │               │       │
│  │ ┌───────────┐ │        │ ┌───────────┐ │        │ ┌───────────┐ │       │
│  │ │ Storage   │ │        │ │ Storage   │ │        │ │ Storage   │ │       │
│  │ │ Extents   │ │        │ │ Extents   │ │        │ │ Extents   │ │       │
│  │ │ (8M rows) │ │        │ │ (8M rows) │ │        │ │ (8M rows) │ │       │
│  │ └───────────┘ │        │ └───────────┘ │        │ └───────────┘ │       │
│  └───────────────┘        └───────────────┘        └───────────────┘       │
│                                                                            │
│  Stockage partagé (optionnel) : S3, GlusterFS, ou local                    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Organisation des données

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ORGANISATION DU STOCKAGE COLUMNSTORE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Table                                                                      │
│    │                                                                        │
│    ├── Colonne 1 (ex: customer_id)                                          │
│    │     ├── Extent 1 (8M valeurs, compressé)                               │
│    │     │     ├── Block 1 (8K valeurs)                                     │
│    │     │     ├── Block 2                                                  │
│    │     │     └── ...                                                      │
│    │     ├── Extent 2                                                       │
│    │     └── ...                                                            │
│    │                                                                        │
│    ├── Colonne 2 (ex: order_date)                                           │
│    │     ├── Extent 1                                                       │
│    │     ├── Extent 2                                                       │
│    │     └── ...                                                            │
│    │                                                                        │
│    └── Colonne N                                                            │
│          └── ...                                                            │
│                                                                             │
│  Métadonnées par Extent :                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Min/Max values (Extent Map)  → Élimination d'extents              │    │
│  │ • Compression type             → Snappy, LZ4, ou None               │    │
│  │ • Row count                    → Jusqu'à 8 millions                 │    │
│  │ • Dictionary (si applicable)   → Pour chaînes répétitives           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Extent Elimination :                                                       │
│  ══════════════════                                                         │
│  WHERE order_date = '2025-01-15'                                            │
│  → ColumnStore vérifie le min/max de chaque extent                          │
│  → Skip les extents où min > '2025-01-15' OR max < '2025-01-15'             │
│  → Peut éliminer 90%+ des données sans les lire                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Modélisation dimensionnelle

### Star Schema

Le Star Schema est le modèle de référence pour le Data Warehousing :

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           STAR SCHEMA                                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                          ┌─────────────────┐                               │
│                          │   dim_date      │                               │
│                          │ ─────────────── │                               │
│                          │ date_key (PK)   │                               │
│                          │ full_date       │                               │
│                          │ day_name        │                               │
│                          │ month           │                               │
│                          │ quarter         │                               │
│                          │ year            │                               │
│                          └────────┬────────┘                               │
│                                   │                                        │
│  ┌─────────────────┐              │              ┌─────────────────┐       │
│  │  dim_product    │              │              │  dim_customer   │       │
│  │ ─────────────── │              │              │ ─────────────── │       │
│  │ product_key(PK) │              │              │ customer_key(PK)│       │
│  │ sku             │              │              │ customer_id     │       │
│  │ name            │    ┌─────────┴─────────┐    │ name            │       │
│  │ category        │    │                   │    │ segment         │       │
│  │ brand           │────┤    fact_sales     │────│ region          │       │
│  │ price           │    │ ───────────────── │    │ country         │       │
│  └─────────────────┘    │ date_key (FK)     │    └─────────────────┘       │
│                         │ product_key (FK)  │                              │
│                         │ customer_key (FK) │                              │
│                         │ store_key (FK)    │                              │
│  ┌─────────────────┐    │ ───────────────── │                              │
│  │   dim_store     │    │ quantity          │                              │
│  │ ─────────────── │────│ unit_price        │                              │
│  │ store_key (PK)  │    │ revenue           │                              │
│  │ store_name      │    │ cost              │                              │
│  │ city            │    │ profit            │                              │
│  │ region          │    │ discount          │                              │
│  └─────────────────┘    └───────────────────┘                              │
│                                                                            │
│  Caractéristiques :                                                        │
│  • Une table de faits centrale avec les mesures                            │
│  • Tables de dimensions autour (dénormalisées)                             │
│  • Jointures simples (1 niveau)                                            │
│  • Optimal pour les requêtes analytiques                                   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Implémentation complète

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- DATA WAREHOUSE E-COMMERCE - STAR SCHEMA AVEC COLUMNSTORE
-- ═══════════════════════════════════════════════════════════════════════════

-- Création de la base de données
CREATE DATABASE IF NOT EXISTS ecommerce_dw 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;
USE ecommerce_dw;

-- ═══════════════════════════════════════════════════════════════════════════
-- DIMENSION : DATE (générée pour 20 ans)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE dim_date (
    date_key INT NOT NULL COMMENT 'Format YYYYMMDD, ex: 20250115',
    full_date DATE NOT NULL,
    
    -- Attributs jour
    day_of_week TINYINT NOT NULL COMMENT '1=Lundi, 7=Dimanche',
    day_of_week_name VARCHAR(10) NOT NULL,
    day_of_month TINYINT NOT NULL,
    day_of_year SMALLINT NOT NULL,
    
    -- Attributs semaine
    week_of_year TINYINT NOT NULL,
    week_start_date DATE NOT NULL,
    
    -- Attributs mois
    month_number TINYINT NOT NULL,
    month_name VARCHAR(15) NOT NULL,
    month_short VARCHAR(3) NOT NULL,
    
    -- Attributs trimestre
    quarter_number TINYINT NOT NULL,
    quarter_name CHAR(2) NOT NULL COMMENT 'Q1, Q2, Q3, Q4',
    
    -- Attributs année
    year_number SMALLINT NOT NULL,
    
    -- Combinaisons utiles
    year_month CHAR(7) NOT NULL COMMENT 'YYYY-MM',
    year_quarter CHAR(7) NOT NULL COMMENT 'YYYY-Q1',
    year_week CHAR(8) NOT NULL COMMENT 'YYYY-W01',
    
    -- Flags
    is_weekend BOOLEAN NOT NULL,
    is_holiday BOOLEAN NOT NULL DEFAULT FALSE,
    holiday_name VARCHAR(50),
    is_last_day_of_month BOOLEAN NOT NULL,
    
    -- Fiscal (personnalisable selon l'entreprise)
    fiscal_year SMALLINT,
    fiscal_quarter TINYINT,
    fiscal_month TINYINT,
    
    PRIMARY KEY (date_key)
) ENGINE=Columnstore
  COMMENT='Dimension temporelle - génération: 2015-01-01 à 2035-12-31';

-- Procédure de génération de la dimension date
DELIMITER //

CREATE PROCEDURE generate_dim_date(
    IN start_date DATE,
    IN end_date DATE
)
BEGIN
    DECLARE current_date_val DATE;
    
    SET current_date_val = start_date;
    
    -- Désactiver temporairement pour insertion bulk
    SET SESSION sql_mode = '';
    
    WHILE current_date_val <= end_date DO
        INSERT INTO dim_date (
            date_key, full_date,
            day_of_week, day_of_week_name, day_of_month, day_of_year,
            week_of_year, week_start_date,
            month_number, month_name, month_short,
            quarter_number, quarter_name,
            year_number,
            year_month, year_quarter, year_week,
            is_weekend, is_last_day_of_month,
            fiscal_year, fiscal_quarter, fiscal_month
        )
        VALUES (
            CAST(DATE_FORMAT(current_date_val, '%Y%m%d') AS UNSIGNED),
            current_date_val,
            DAYOFWEEK(current_date_val),
            DAYNAME(current_date_val),
            DAYOFMONTH(current_date_val),
            DAYOFYEAR(current_date_val),
            WEEKOFYEAR(current_date_val),
            DATE_SUB(current_date_val, INTERVAL (DAYOFWEEK(current_date_val) - 2) DAY),
            MONTH(current_date_val),
            MONTHNAME(current_date_val),
            DATE_FORMAT(current_date_val, '%b'),
            QUARTER(current_date_val),
            CONCAT('Q', QUARTER(current_date_val)),
            YEAR(current_date_val),
            DATE_FORMAT(current_date_val, '%Y-%m'),
            CONCAT(YEAR(current_date_val), '-Q', QUARTER(current_date_val)),
            CONCAT(YEAR(current_date_val), '-W', LPAD(WEEKOFYEAR(current_date_val), 2, '0')),
            DAYOFWEEK(current_date_val) IN (1, 7),
            current_date_val = LAST_DAY(current_date_val),
            -- Fiscal year (exemple: commence en Avril)
            IF(MONTH(current_date_val) >= 4, 
               YEAR(current_date_val), 
               YEAR(current_date_val) - 1),
            CEIL((MONTH(current_date_val) - 3) / 3),
            MOD(MONTH(current_date_val) + 8, 12) + 1
        );
        
        SET current_date_val = DATE_ADD(current_date_val, INTERVAL 1 DAY);
    END WHILE;
END //

DELIMITER ;

-- Générer 20 ans de dates
CALL generate_dim_date('2015-01-01', '2035-12-31');

-- ═══════════════════════════════════════════════════════════════════════════
-- DIMENSION : PRODUIT (avec hiérarchie dénormalisée)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE dim_product (
    product_key INT NOT NULL AUTO_INCREMENT,
    
    -- Clé métier
    product_id VARCHAR(50) NOT NULL COMMENT 'ID source OLTP',
    sku VARCHAR(50) NOT NULL,
    
    -- Attributs produit
    product_name VARCHAR(255) NOT NULL,
    product_description TEXT,
    
    -- Hiérarchie de catégories (dénormalisée pour performances)
    category_id INT,
    category_name VARCHAR(100),
    subcategory_id INT,
    subcategory_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),
    
    -- Marque et fournisseur
    brand_id INT,
    brand_name VARCHAR(100),
    supplier_id INT,
    supplier_name VARCHAR(200),
    supplier_country VARCHAR(100),
    
    -- Attributs commerciaux
    unit_cost DECIMAL(10,2) COMMENT 'Coût d''achat',
    list_price DECIMAL(10,2) COMMENT 'Prix catalogue',
    weight_kg DECIMAL(8,3),
    
    -- Statut
    status VARCHAR(20) DEFAULT 'Active' COMMENT 'Active, Discontinued, Seasonal',
    introduction_date DATE,
    discontinuation_date DATE,
    
    -- SCD Type 2 : Gestion des changements
    effective_start_date DATETIME NOT NULL,
    effective_end_date DATETIME DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE,
    version_number INT DEFAULT 1,
    
    PRIMARY KEY (product_key)
) ENGINE=Columnstore;

-- Index sur la clé métier pour les lookups ETL
-- Note: ColumnStore gère les index différemment, on utilise extent elimination

-- ═══════════════════════════════════════════════════════════════════════════
-- DIMENSION : CLIENT (avec segmentation)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE dim_customer (
    customer_key INT NOT NULL AUTO_INCREMENT,
    
    -- Clé métier
    customer_id VARCHAR(50) NOT NULL,
    
    -- Identité
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    full_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(50),
    
    -- Type de client
    customer_type VARCHAR(20) COMMENT 'B2C, B2B, Wholesale, Partner',
    
    -- Géographie (dénormalisée)
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100),
    country_code CHAR(2),
    region VARCHAR(50) COMMENT 'EMEA, APAC, Americas, etc.',
    
    -- Segmentation analytique
    segment VARCHAR(50) COMMENT 'Premium, Standard, Basic, Churned',
    acquisition_channel VARCHAR(50) COMMENT 'Organic, Paid, Referral, Partner',
    acquisition_campaign VARCHAR(100),
    acquisition_date DATE,
    
    -- Métriques pré-calculées (mises à jour périodiquement)
    lifetime_value DECIMAL(12,2),
    total_orders INT,
    avg_order_value DECIMAL(10,2),
    first_order_date DATE,
    last_order_date DATE,
    days_since_last_order INT,
    
    -- RFM Score (calculé)
    rfm_recency_score TINYINT,
    rfm_frequency_score TINYINT,
    rfm_monetary_score TINYINT,
    rfm_segment VARCHAR(50),
    
    -- SCD Type 2
    effective_start_date DATETIME NOT NULL,
    effective_end_date DATETIME DEFAULT '9999-12-31 23:59:59',
    is_current BOOLEAN DEFAULT TRUE,
    
    PRIMARY KEY (customer_key)
) ENGINE=Columnstore;

-- ═══════════════════════════════════════════════════════════════════════════
-- DIMENSION : MAGASIN / CANAL
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE dim_store (
    store_key INT NOT NULL AUTO_INCREMENT,
    
    store_id VARCHAR(20) NOT NULL,
    store_name VARCHAR(100) NOT NULL,
    store_type VARCHAR(30) COMMENT 'Physical, Online, Marketplace, Franchise',
    
    -- Localisation physique (si applicable)
    address VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    country VARCHAR(100),
    region VARCHAR(50),
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    timezone VARCHAR(50),
    
    -- Caractéristiques
    size_sqm INT COMMENT 'Surface en m²',
    employee_count INT,
    parking_spaces INT,
    
    -- Dates clés
    opening_date DATE,
    renovation_date DATE,
    closing_date DATE,
    
    -- Gestion
    district_manager VARCHAR(100),
    regional_manager VARCHAR(100),
    
    -- Statut
    status VARCHAR(20) DEFAULT 'Open' COMMENT 'Open, Closed, Renovating',
    is_flagship BOOLEAN DEFAULT FALSE,
    
    PRIMARY KEY (store_key)
) ENGINE=Columnstore;

-- ═══════════════════════════════════════════════════════════════════════════
-- TABLE DE FAITS : VENTES (grain = ligne de commande)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE fact_sales (
    -- Clés de dimension
    date_key INT NOT NULL,
    product_key INT NOT NULL,
    customer_key INT NOT NULL,
    store_key INT NOT NULL,
    
    -- Clés dégénérées (info sans dimension dédiée)
    order_id BIGINT NOT NULL,
    order_line_id BIGINT NOT NULL,
    
    -- Attributs de la transaction
    order_datetime DATETIME NOT NULL,
    order_status VARCHAR(20),
    payment_method VARCHAR(30),
    shipping_method VARCHAR(30),
    
    -- Mesures additives
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    unit_cost DECIMAL(10,2),
    
    -- Montants
    gross_amount DECIMAL(12,2) NOT NULL COMMENT 'quantity * unit_price',
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    shipping_amount DECIMAL(10,2) DEFAULT 0,
    net_amount DECIMAL(12,2) NOT NULL COMMENT 'gross - discount + tax + shipping',
    
    -- Coûts et profits
    cost_amount DECIMAL(12,2) COMMENT 'quantity * unit_cost',
    profit_amount DECIMAL(12,2) COMMENT 'net_amount - cost_amount',
    profit_margin_pct DECIMAL(5,2) COMMENT 'profit / net_amount * 100',
    
    -- Flags analytiques
    is_return BOOLEAN DEFAULT FALSE,
    is_promotion BOOLEAN DEFAULT FALSE,
    promotion_id INT,
    
    -- Timestamps ETL
    etl_loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    
    -- Note: Pas de PRIMARY KEY composite en ColumnStore
    -- L'unicité est garantie par order_line_id
) ENGINE=Columnstore
  COMMENT='Table de faits des ventes - grain: ligne de commande';

-- ═══════════════════════════════════════════════════════════════════════════
-- TABLES DE FAITS AGRÉGÉES (pour performances dashboard)
-- ═══════════════════════════════════════════════════════════════════════════

-- Agrégation quotidienne par magasin et catégorie
CREATE TABLE fact_daily_sales (
    date_key INT NOT NULL,
    store_key INT NOT NULL,
    category_id INT NOT NULL,
    
    -- Comptages
    order_count INT NOT NULL,
    line_count INT NOT NULL,
    customer_count INT NOT NULL,
    
    -- Mesures agrégées
    total_quantity INT NOT NULL,
    gross_revenue DECIMAL(14,2) NOT NULL,
    discount_total DECIMAL(12,2) NOT NULL,
    tax_total DECIMAL(12,2) NOT NULL,
    net_revenue DECIMAL(14,2) NOT NULL,
    cost_total DECIMAL(14,2),
    profit_total DECIMAL(14,2),
    
    -- Métriques calculées
    avg_order_value DECIMAL(10,2),
    avg_items_per_order DECIMAL(6,2),
    avg_selling_price DECIMAL(10,2),
    
    -- ETL
    etl_loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=Columnstore;

-- Agrégation mensuelle (pour trends long terme)
CREATE TABLE fact_monthly_sales (
    year_month CHAR(7) NOT NULL COMMENT 'YYYY-MM',
    year_number SMALLINT NOT NULL,
    month_number TINYINT NOT NULL,
    store_key INT NOT NULL,
    category_id INT NOT NULL,
    customer_segment VARCHAR(50),
    
    -- Mêmes mesures que daily mais agrégées
    order_count INT NOT NULL,
    unique_customers INT NOT NULL,
    new_customers INT NOT NULL,
    returning_customers INT NOT NULL,
    
    gross_revenue DECIMAL(16,2) NOT NULL,
    net_revenue DECIMAL(16,2) NOT NULL,
    profit_total DECIMAL(16,2),
    
    -- Comparaisons
    revenue_vs_prev_month DECIMAL(16,2),
    revenue_vs_prev_year DECIMAL(16,2),
    yoy_growth_pct DECIMAL(6,2),
    mom_growth_pct DECIMAL(6,2)
) ENGINE=Columnstore;
```

---

## Pipelines ETL/ELT

### Architecture ETL moderne

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PIPELINE ETL VERS COLUMNSTORE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sources OLTP                   Transformation              Data Warehouse  │
│  ════════════                   ══════════════              ══════════════  │
│                                                                             │
│  ┌─────────────┐                                           ┌─────────────┐  │
│  │  MariaDB    │──┐                                        │ ColumnStore │  │
│  │  (Orders)   │  │      ┌─────────────────────┐           │             │  │
│  └─────────────┘  │      │                     │           │ ┌─────────┐ │  │
│                   │      │    ETL Engine       │           │ │dim_date │ │  │
│  ┌─────────────┐  │      │                     │           │ │dim_prod │ │  │
│  │  MariaDB    │──┼─────►│  • Extract (CDC)    │──────────►│ │dim_cust │ │  │
│  │  (Products) │  │      │  • Transform        │           │ │         │ │  │
│  └─────────────┘  │      │  • Load (cpimport)  │           │ │fact_    │ │  │
│                   │      │                     │           │ │sales    │ │  │
│  ┌─────────────┐  │      │  Outils:            │           │ └─────────┘ │  │
│  │  PostgreSQL │──┤      │  • Debezium/Kafka   │           │             │  │
│  │  (Legacy)   │  │      │  • Apache Airflow   │           └─────────────┘  │
│  └─────────────┘  │      │  • dbt              │                            │
│                   │      │  • Custom Python    │                            │
│  ┌─────────────┐  │      │                     │                            │
│  │  APIs       │──┘      └─────────────────────┘                            │
│  │  (External) │                                                            │
│  └─────────────┘                                                            │
│                                                                             │
│  Patterns de chargement :                                                   │
│  ═══════════════════════                                                    │
│  • Full Load     : Rechargement complet (dimensions petites)                │
│  • Incremental   : Delta via timestamp/CDC (faits)                          │
│  • Merge (SCD)   : Update + Insert pour dimensions                          │
│  • Bulk (cpimport): Chargement optimisé pour gros volumes                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Chargement bulk avec cpimport

```bash
#!/bin/bash
# etl-load-sales.sh
# Chargement optimisé des ventes via cpimport

# Configuration
DW_HOST="columnstore-um"
DW_DB="ecommerce_dw"
DATA_DIR="/data/staging"
LOG_DIR="/var/log/etl"
DATE=$(date +%Y%m%d)

# Créer le répertoire de logs
mkdir -p "$LOG_DIR"

# Fonction de chargement bulk
load_table() {
    local table=$1
    local file=$2
    local delimiter=${3:-','}
    
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Loading $table from $file..."
    
    cpimport \
        -h "$DW_HOST" \
        -u etl_user \
        -p "$ETL_PASSWORD" \
        -s "$delimiter" \
        -E '"' \
        -C 'YYYY-MM-DD HH:MI:SS' \
        -n 1 \
        "$DW_DB" \
        "$table" \
        "$file" \
        >> "$LOG_DIR/cpimport_${DATE}.log" 2>&1
    
    local status=$?
    if [ $status -eq 0 ]; then
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] $table loaded successfully"
    else
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR loading $table (exit code: $status)"
        exit 1
    fi
}

# Chargement des dimensions (full refresh pour les petites tables)
echo "=== Loading dimensions ==="
load_table "dim_store" "$DATA_DIR/dim_store_${DATE}.csv"
load_table "dim_product" "$DATA_DIR/dim_product_${DATE}.csv"
load_table "dim_customer" "$DATA_DIR/dim_customer_${DATE}.csv"

# Chargement des faits (incrémental)
echo "=== Loading fact tables ==="
load_table "fact_sales" "$DATA_DIR/fact_sales_delta_${DATE}.csv"

# Mise à jour des agrégats
echo "=== Refreshing aggregates ==="
mariadb -h "$DW_HOST" -u etl_user -p"$ETL_PASSWORD" "$DW_DB" << 'EOF'
-- Recalculer les agrégations quotidiennes pour les données chargées
INSERT INTO fact_daily_sales
SELECT 
    date_key,
    store_key,
    p.category_id,
    COUNT(DISTINCT f.order_id),
    COUNT(*),
    COUNT(DISTINCT f.customer_key),
    SUM(quantity),
    SUM(gross_amount),
    SUM(discount_amount),
    SUM(tax_amount),
    SUM(net_amount),
    SUM(cost_amount),
    SUM(profit_amount),
    SUM(net_amount) / NULLIF(COUNT(DISTINCT f.order_id), 0),
    SUM(quantity) / NULLIF(COUNT(DISTINCT f.order_id), 0),
    SUM(net_amount) / NULLIF(SUM(quantity), 0),
    NOW()
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key AND p.is_current = TRUE
WHERE f.etl_loaded_at >= CURDATE()
GROUP BY date_key, store_key, p.category_id
ON DUPLICATE KEY UPDATE
    order_count = VALUES(order_count),
    line_count = VALUES(line_count),
    customer_count = VALUES(customer_count),
    total_quantity = VALUES(total_quantity),
    gross_revenue = VALUES(gross_revenue),
    net_revenue = VALUES(net_revenue),
    etl_loaded_at = NOW();
EOF

echo "=== ETL completed successfully ==="
```

### ETL avec Python et pandas

```python
#!/usr/bin/env python3
"""
etl_pipeline.py
Pipeline ETL pour charger les données OLTP vers ColumnStore
"""

import pandas as pd
import mariadb
from datetime import datetime, timedelta
import logging
from typing import Optional
import os

# Configuration logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

class ColumnStoreETL:
    """Pipeline ETL vers MariaDB ColumnStore"""
    
    def __init__(self, source_config: dict, target_config: dict):
        self.source_config = source_config
        self.target_config = target_config
        self.source_conn = None
        self.target_conn = None
    
    def connect(self):
        """Établir les connexions source et cible"""
        logger.info("Connecting to databases...")
        
        self.source_conn = mariadb.connect(**self.source_config)
        self.target_conn = mariadb.connect(**self.target_config)
        
        logger.info("Connections established")
    
    def close(self):
        """Fermer les connexions"""
        if self.source_conn:
            self.source_conn.close()
        if self.target_conn:
            self.target_conn.close()
    
    def extract_incremental(
        self, 
        table: str, 
        timestamp_col: str,
        last_load: datetime
    ) -> pd.DataFrame:
        """Extraction incrémentale depuis la source"""
        
        query = f"""
        SELECT * FROM {table}
        WHERE {timestamp_col} > %s
        ORDER BY {timestamp_col}
        """
        
        logger.info(f"Extracting from {table} since {last_load}")
        
        df = pd.read_sql(query, self.source_conn, params=[last_load])
        logger.info(f"Extracted {len(df)} rows from {table}")
        
        return df
    
    def transform_sales(self, df: pd.DataFrame) -> pd.DataFrame:
        """Transformation des données de ventes"""
        
        logger.info("Transforming sales data...")
        
        # Calcul des métriques
        df['gross_amount'] = df['quantity'] * df['unit_price']
        df['net_amount'] = df['gross_amount'] - df['discount_amount'] + df['tax_amount']
        df['cost_amount'] = df['quantity'] * df['unit_cost']
        df['profit_amount'] = df['net_amount'] - df['cost_amount']
        df['profit_margin_pct'] = (
            df['profit_amount'] / df['net_amount'].replace(0, float('nan')) * 100
        ).fillna(0)
        
        # Clé de date (YYYYMMDD)
        df['date_key'] = pd.to_datetime(df['order_datetime']).dt.strftime('%Y%m%d').astype(int)
        
        # Colonnes finales
        columns = [
            'date_key', 'product_key', 'customer_key', 'store_key',
            'order_id', 'order_line_id', 'order_datetime', 'order_status',
            'payment_method', 'shipping_method',
            'quantity', 'unit_price', 'unit_cost',
            'gross_amount', 'discount_amount', 'tax_amount', 'shipping_amount',
            'net_amount', 'cost_amount', 'profit_amount', 'profit_margin_pct',
            'is_return', 'is_promotion', 'promotion_id'
        ]
        
        return df[columns]
    
    def lookup_dimension_keys(
        self, 
        df: pd.DataFrame,
        dim_table: str,
        business_key_col: str,
        surrogate_key_col: str
    ) -> pd.DataFrame:
        """Lookup des clés de dimension (surrogate keys)"""
        
        # Récupérer le mapping depuis la dimension
        lookup_query = f"""
        SELECT {business_key_col}, {surrogate_key_col}
        FROM {dim_table}
        WHERE is_current = TRUE
        """
        
        lookup_df = pd.read_sql(lookup_query, self.target_conn)
        lookup_dict = dict(zip(
            lookup_df[business_key_col], 
            lookup_df[surrogate_key_col]
        ))
        
        # Appliquer le mapping
        source_col = business_key_col.replace('_id', '_key').replace('_key_key', '_key')
        df[f'{source_col}'] = df[business_key_col].map(lookup_dict)
        
        # Vérifier les clés manquantes
        missing = df[df[f'{source_col}'].isna()][business_key_col].unique()
        if len(missing) > 0:
            logger.warning(f"Missing dimension keys for {dim_table}: {missing[:10]}...")
        
        return df
    
    def load_to_columnstore(
        self, 
        df: pd.DataFrame, 
        table: str,
        batch_size: int = 10000
    ):
        """Chargement vers ColumnStore par batch"""
        
        logger.info(f"Loading {len(df)} rows to {table}...")
        
        cursor = self.target_conn.cursor()
        
        # Préparer la requête INSERT
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
        
        # Insertion par batch
        total_loaded = 0
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            values = [tuple(row) for row in batch.values]
            
            cursor.executemany(insert_query, values)
            self.target_conn.commit()
            
            total_loaded += len(batch)
            logger.info(f"Loaded {total_loaded}/{len(df)} rows")
        
        cursor.close()
        logger.info(f"Successfully loaded {total_loaded} rows to {table}")
    
    def run_full_pipeline(self, last_load: datetime):
        """Exécuter le pipeline complet"""
        
        try:
            self.connect()
            
            # 1. Extract
            orders_df = self.extract_incremental(
                'order_items', 
                'updated_at',
                last_load
            )
            
            if orders_df.empty:
                logger.info("No new data to process")
                return
            
            # 2. Transform
            sales_df = self.transform_sales(orders_df)
            
            # 3. Dimension lookups
            sales_df = self.lookup_dimension_keys(
                sales_df, 'dim_product', 'product_id', 'product_key'
            )
            sales_df = self.lookup_dimension_keys(
                sales_df, 'dim_customer', 'customer_id', 'customer_key'
            )
            
            # 4. Load
            self.load_to_columnstore(sales_df, 'fact_sales')
            
            logger.info("Pipeline completed successfully")
            
        finally:
            self.close()


# Point d'entrée
if __name__ == '__main__':
    source_config = {
        'host': os.environ.get('SOURCE_HOST', 'oltp-db'),
        'port': 3306,
        'user': 'etl_reader',
        'password': os.environ['SOURCE_PASSWORD'],
        'database': 'ecommerce'
    }
    
    target_config = {
        'host': os.environ.get('TARGET_HOST', 'columnstore-um'),
        'port': 3306,
        'user': 'etl_writer',
        'password': os.environ['TARGET_PASSWORD'],
        'database': 'ecommerce_dw'
    }
    
    # Dernière exécution (à stocker en base ou fichier)
    last_load = datetime.now() - timedelta(hours=1)
    
    etl = ColumnStoreETL(source_config, target_config)
    etl.run_full_pipeline(last_load)
```

---

## Optimisation des requêtes analytiques

### Bonnes pratiques

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- OPTIMISATION DES REQUÊTES COLUMNSTORE
-- ═══════════════════════════════════════════════════════════════════════════

-- ✅ BON : Filtrer sur les colonnes pour extent elimination
-- Le filtre sur date_key permet d'éliminer les extents non pertinents
SELECT 
    d.month_name,
    p.category_name,
    SUM(f.net_amount) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year_number = 2025           -- Extent elimination efficace
  AND d.quarter_number = 1
GROUP BY d.month_name, p.category_name
ORDER BY revenue DESC;

-- ❌ MAUVAIS : Fonction sur la colonne filtrée (pas d'extent elimination)
SELECT 
    d.month_name,
    SUM(f.net_amount) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
WHERE YEAR(d.full_date) = 2025      -- Fonction empêche l'optimisation
GROUP BY d.month_name;

-- ✅ BON : Pré-agréger pour les dashboards fréquents
-- Utiliser les tables agrégées pour les requêtes répétitives
SELECT 
    year_month,
    SUM(net_revenue) AS revenue,
    SUM(profit_total) AS profit
FROM fact_monthly_sales
WHERE year_number = 2025
GROUP BY year_month
ORDER BY year_month;

-- ✅ BON : Limiter les colonnes sélectionnées
-- ColumnStore lit seulement les colonnes demandées
SELECT 
    date_key,
    SUM(quantity),
    SUM(net_amount)
FROM fact_sales
WHERE date_key BETWEEN 20250101 AND 20250131
GROUP BY date_key;

-- ❌ MAUVAIS : SELECT * (lit toutes les colonnes)
SELECT *
FROM fact_sales
WHERE date_key BETWEEN 20250101 AND 20250131;

-- ✅ BON : ORDER BY après les agrégations
SELECT 
    p.category_name,
    SUM(f.net_amount) AS revenue
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
WHERE f.date_key >= 20250101
GROUP BY p.category_name
ORDER BY revenue DESC              -- Tri sur résultat agrégé (petit volume)
LIMIT 10;

-- ✅ BON : Utiliser des CTE pour la lisibilité et l'optimisation
WITH monthly_sales AS (
    SELECT 
        d.year_month,
        d.year_number,
        d.month_number,
        SUM(f.net_amount) AS revenue
    FROM fact_sales f
    JOIN dim_date d ON f.date_key = d.date_key
    WHERE d.year_number IN (2024, 2025)
    GROUP BY d.year_month, d.year_number, d.month_number
),
with_comparison AS (
    SELECT 
        year_month,
        revenue,
        LAG(revenue, 12) OVER (ORDER BY year_month) AS revenue_prev_year
    FROM monthly_sales
)
SELECT 
    year_month,
    revenue,
    revenue_prev_year,
    (revenue - revenue_prev_year) / NULLIF(revenue_prev_year, 0) * 100 AS yoy_growth
FROM with_comparison
WHERE year_month >= '2025-01'
ORDER BY year_month;
```

### Analyser les performances

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- ANALYSE DES PERFORMANCES COLUMNSTORE
-- ═══════════════════════════════════════════════════════════════════════════

-- Vérifier les statistiques de la requête
SET infinidb_vtable_mode = 1;  -- Mode verbose

EXPLAIN SELECT 
    d.year_month,
    SUM(f.net_amount)
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year_number = 2025
GROUP BY d.year_month;

-- Statistiques d'extent elimination
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    COLUMN_NAME,
    OBJECT_ID,
    EXTENT_COUNT,
    EXTENT_ROWS
FROM information_schema.COLUMNSTORE_EXTENTS
WHERE TABLE_SCHEMA = 'ecommerce_dw'
  AND TABLE_NAME = 'fact_sales';

-- Requêtes en cours d'exécution
SELECT 
    ID,
    USER,
    HOST,
    DB,
    TIME,
    STATE,
    SUBSTRING(INFO, 1, 200) AS query_preview
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND TIME > 5
ORDER BY TIME DESC;

-- Utilisation mémoire
SHOW STATUS LIKE 'columnstore%';
```

---

## Déploiement et dimensionnement

### Configuration cluster multi-nœuds

```yaml
# docker-compose-columnstore-cluster.yml
version: '3.8'

services:
  # User Module (point d'entrée SQL)
  columnstore-um:
    image: mariadb/columnstore:latest
    hostname: um1
    container_name: columnstore-um
    environment:
      - MARIADB_ROOT_PASSWORD=${ROOT_PASSWORD}
      - MARIADB_USER=dw_user
      - MARIADB_PASSWORD=${DW_PASSWORD}
      - MARIADB_DATABASE=ecommerce_dw
      - COLUMNSTORE_MODULE_TYPE=um
      - COLUMNSTORE_UM_COUNT=1
      - COLUMNSTORE_PM_COUNT=3
    ports:
      - "3306:3306"
    volumes:
      - um-data:/var/lib/columnstore
      - ./config/Columnstore.xml:/etc/columnstore/Columnstore.xml
    networks:
      - columnstore-net
    depends_on:
      - columnstore-pm1
      - columnstore-pm2
      - columnstore-pm3

  # Performance Module 1
  columnstore-pm1:
    image: mariadb/columnstore:latest
    hostname: pm1
    container_name: columnstore-pm1
    environment:
      - COLUMNSTORE_MODULE_TYPE=pm
      - COLUMNSTORE_PM_ID=1
    volumes:
      - pm1-data:/var/lib/columnstore
      - ./data/pm1:/data/columnstore
    networks:
      - columnstore-net

  # Performance Module 2
  columnstore-pm2:
    image: mariadb/columnstore:latest
    hostname: pm2
    container_name: columnstore-pm2
    environment:
      - COLUMNSTORE_MODULE_TYPE=pm
      - COLUMNSTORE_PM_ID=2
    volumes:
      - pm2-data:/var/lib/columnstore
      - ./data/pm2:/data/columnstore
    networks:
      - columnstore-net

  # Performance Module 3
  columnstore-pm3:
    image: mariadb/columnstore:latest
    hostname: pm3
    container_name: columnstore-pm3
    environment:
      - COLUMNSTORE_MODULE_TYPE=pm
      - COLUMNSTORE_PM_ID=3
    volumes:
      - pm3-data:/var/lib/columnstore
      - ./data/pm3:/data/columnstore
    networks:
      - columnstore-net

volumes:
  um-data:
  pm1-data:
  pm2-data:
  pm3-data:

networks:
  columnstore-net:
    driver: bridge
```

### Dimensionnement

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GUIDE DE DIMENSIONNEMENT COLUMNSTORE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Volume données    │  Config recommandée                                    │
│  ══════════════════│════════════════════                                    │
│  < 100 GB          │  Single node (UM+PM combiné)                           │
│                    │  • 16 GB RAM, 4 cores                                  │
│                    │  • SSD 500 GB                                          │
│                    │                                                        │
│  100 GB - 1 TB     │  1 UM + 2-3 PM                                         │
│                    │  • UM: 16 GB RAM, 4 cores                              │
│                    │  • PM: 32 GB RAM, 8 cores chacun                       │
│                    │  • SSD 1-2 TB par PM                                   │
│                    │                                                        │
│  1 TB - 10 TB      │  1-2 UM + 5-10 PM                                      │
│                    │  • UM: 32 GB RAM, 8 cores                              │
│                    │  • PM: 64 GB RAM, 16 cores chacun                      │
│                    │  • SSD NVMe 2-4 TB par PM                              │
│                    │                                                        │
│  > 10 TB           │  2+ UM (HA) + 10+ PM                                   │
│                    │  • UM: 64 GB RAM, 16 cores                             │
│                    │  • PM: 128 GB RAM, 32 cores chacun                     │
│                    │  • Storage distribué (S3, GlusterFS)                   │
│                                                                             │
│  Règles générales :                                                         │
│  ════════════════                                                           │
│  • RAM PM : 2-4 GB par TB de données compressées                            │
│  • Compression typique : 5-10x (10 TB raw → 1-2 TB stocké)                  │
│  • CPU : Plus de cores = meilleur parallélisme sur grosses requêtes         │
│  • Réseau : 10 Gbps minimum entre UM et PM pour gros volumes                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Intégration dans une architecture data moderne

### Data Lakehouse avec ColumnStore

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE DATA LAKEHOUSE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sources                  Data Lake              Data Warehouse             │
│  ═══════                  ═════════              ══════════════             │
│                                                                             │
│  ┌─────────┐                                                                │
│  │ OLTP    │──┐                                                             │
│  │ Systems │  │          ┌─────────────────┐                                │
│  └─────────┘  │          │                 │    ┌─────────────────┐         │
│               │          │   S3 / MinIO    │    │   ColumnStore   │         │
│  ┌─────────┐  │          │   (Raw Zone)    │    │   (Gold Zone)   │         │
│  │ APIs    │──┼─────────►│                 │───►│                 │         │
│  │         │  │  Ingest  │   ┌──────────┐  │ETL │   Star Schema   │         │
│  └─────────┘  │          │   │ Parquet  │  │    │   Optimisé      │         │
│               │          │   │ CSV      │  │    │                 │         │
│  ┌─────────┐  │          │   │ JSON     │  │    │   fact_sales    │         │
│  │ Streams │──┤          │   └──────────┘  │    │   dim_*         │         │
│  │ (Kafka) │  │          │                 │    │                 │         │
│  └─────────┘  │          └────────┬────────┘    └────────┬────────┘         │
│               │                   │                      │                  │
│  ┌─────────┐  │          ┌────────▼────────┐    ┌────────▼────────┐         │
│  │ Files   │──┘          │   Spark/dbt     │    │   BI Tools      │         │
│  │ (S3)    │             │   Processing    │    │   (Tableau,     │         │
│  └─────────┘             │                 │    │    Metabase,    │         │
│                          └─────────────────┘    │    Superset)    │         │
│                                                 └─────────────────┘         │
│                                                                             │
│  Avantages de cette architecture :                                          │
│  • Données brutes préservées dans le Data Lake (S3)                         │
│  • Traitement flexible avec Spark/dbt                                       │
│  • ColumnStore pour les requêtes SQL haute performance                      │
│  • Séparation stockage (S3) et compute (ColumnStore)                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Requêtes analytiques avancées

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- REQUÊTES ANALYTIQUES COMPLEXES
-- ═══════════════════════════════════════════════════════════════════════════

-- Analyse de cohortes (rétention client)
WITH first_purchase AS (
    SELECT 
        customer_key,
        MIN(date_key) AS cohort_date_key
    FROM fact_sales
    GROUP BY customer_key
),
cohort_data AS (
    SELECT 
        fp.cohort_date_key,
        d_cohort.year_month AS cohort_month,
        d_order.year_month AS order_month,
        PERIOD_DIFF(
            EXTRACT(YEAR_MONTH FROM d_order.full_date),
            EXTRACT(YEAR_MONTH FROM d_cohort.full_date)
        ) AS months_since_first,
        f.customer_key
    FROM fact_sales f
    JOIN first_purchase fp ON f.customer_key = fp.customer_key
    JOIN dim_date d_cohort ON fp.cohort_date_key = d_cohort.date_key
    JOIN dim_date d_order ON f.date_key = d_order.date_key
)
SELECT 
    cohort_month,
    months_since_first,
    COUNT(DISTINCT customer_key) AS customers,
    FIRST_VALUE(COUNT(DISTINCT customer_key)) OVER (
        PARTITION BY cohort_month ORDER BY months_since_first
    ) AS cohort_size,
    COUNT(DISTINCT customer_key) * 100.0 / 
        FIRST_VALUE(COUNT(DISTINCT customer_key)) OVER (
            PARTITION BY cohort_month ORDER BY months_since_first
        ) AS retention_pct
FROM cohort_data
WHERE cohort_month >= '2024-01'
  AND months_since_first <= 12
GROUP BY cohort_month, months_since_first
ORDER BY cohort_month, months_since_first;

-- Analyse RFM (Recency, Frequency, Monetary)
WITH customer_rfm AS (
    SELECT 
        c.customer_key,
        c.customer_id,
        c.full_name,
        DATEDIFF(CURRENT_DATE, MAX(d.full_date)) AS recency_days,
        COUNT(DISTINCT f.order_id) AS frequency,
        SUM(f.net_amount) AS monetary
    FROM fact_sales f
    JOIN dim_customer c ON f.customer_key = c.customer_key
    JOIN dim_date d ON f.date_key = d.date_key
    WHERE d.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 2 YEAR)
      AND c.is_current = TRUE
    GROUP BY c.customer_key, c.customer_id, c.full_name
),
rfm_scores AS (
    SELECT 
        *,
        NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,  -- Lower is better
        NTILE(5) OVER (ORDER BY frequency) AS f_score,
        NTILE(5) OVER (ORDER BY monetary) AS m_score
    FROM customer_rfm
)
SELECT 
    customer_key,
    customer_id,
    full_name,
    recency_days,
    frequency,
    monetary,
    r_score,
    f_score,
    m_score,
    CONCAT(r_score, f_score, m_score) AS rfm_cell,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
        WHEN r_score >= 4 AND f_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 4 AND f_score <= 2 THEN 'Recent Customers'
        WHEN r_score >= 3 AND f_score >= 3 THEN 'Promising'
        WHEN r_score <= 2 AND f_score >= 4 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score >= 3 THEN 'Cant Lose Them'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost'
        ELSE 'Others'
    END AS rfm_segment
FROM rfm_scores
ORDER BY monetary DESC;

-- Analyse panier (Market Basket Analysis - Produits fréquemment achetés ensemble)
WITH order_products AS (
    SELECT 
        order_id,
        product_key
    FROM fact_sales
    WHERE date_key >= 20240101
),
product_pairs AS (
    SELECT 
        a.product_key AS product_a,
        b.product_key AS product_b,
        COUNT(DISTINCT a.order_id) AS pair_count
    FROM order_products a
    JOIN order_products b ON a.order_id = b.order_id AND a.product_key < b.product_key
    GROUP BY a.product_key, b.product_key
    HAVING pair_count >= 100
),
product_stats AS (
    SELECT 
        product_key,
        COUNT(DISTINCT order_id) AS product_orders
    FROM order_products
    GROUP BY product_key
)
SELECT 
    pa.product_name AS product_a_name,
    pb.product_name AS product_b_name,
    pp.pair_count,
    ps_a.product_orders AS orders_with_a,
    ps_b.product_orders AS orders_with_b,
    pp.pair_count * 1.0 / ps_a.product_orders AS support_a,
    pp.pair_count * 1.0 / ps_b.product_orders AS support_b,
    (pp.pair_count * 1.0 / ps_a.product_orders) / 
        (ps_b.product_orders * 1.0 / (SELECT COUNT(DISTINCT order_id) FROM order_products)) AS lift
FROM product_pairs pp
JOIN dim_product pa ON pp.product_a = pa.product_key AND pa.is_current = TRUE
JOIN dim_product pb ON pp.product_b = pb.product_key AND pb.is_current = TRUE
JOIN product_stats ps_a ON pp.product_a = ps_a.product_key
JOIN product_stats ps_b ON pp.product_b = ps_b.product_key
WHERE lift > 1.5
ORDER BY lift DESC
LIMIT 50;
```

---

## ✅ Points clés à retenir

- **ColumnStore** est optimisé pour les requêtes analytiques lisant peu de colonnes mais beaucoup de lignes — gains de 10x à 100x vs InnoDB
- **Le Star Schema** est le modèle de référence : une table de faits centrale entourée de dimensions dénormalisées
- **L'Extent Elimination** est la clé de performance — filtrer sur des colonnes avec bonne distribution min/max
- **cpimport** est l'outil de chargement bulk optimal — bien plus rapide que INSERT
- **Les tables agrégées** (daily, monthly) accélèrent drastiquement les dashboards fréquents
- **Dimensionnement** : 2-4 GB RAM par TB de données compressées sur les Performance Modules
- **SCD Type 2** permet de tracer l'historique des dimensions (effective_start_date, effective_end_date, is_current)
- **L'architecture Lakehouse** combine le meilleur du Data Lake (S3, flexibilité) et du Data Warehouse (ColumnStore, SQL)

---

## 🔗 Ressources et références

- [📖 MariaDB ColumnStore Documentation](https://mariadb.com/kb/en/mariadb-columnstore/)
- [📖 The Data Warehouse Toolkit - Ralph Kimball](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/)
- [📖 cpimport User Guide](https://mariadb.com/kb/en/columnstore-bulk-data-loading/)
- [📖 ColumnStore System Variables](https://mariadb.com/kb/en/columnstore-system-variables/)
- [📖 Star Schema Best Practices](https://mariadb.com/kb/en/columnstore-star-schema/)

---

## ➡️ Section suivante

[20.4 Architecture Multi-Tenant](./04-architecture-multi-tenant.md) : Découvrez les stratégies d'isolation des données pour les applications SaaS avec MariaDB.

⏭️[Architecture multi-tenant](/20-cas-usage-architectures/04-architecture-multi-tenant.md)

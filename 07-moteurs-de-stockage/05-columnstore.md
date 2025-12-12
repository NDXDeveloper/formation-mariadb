🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5 ColumnStore : Analytique et OLAP

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 7.1-7.2 (Architecture, InnoDB), concepts data warehousing et OLAP

> **Public cible** : DBA, Architectes data, Data Engineers, Analystes BI

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture columnar et ses avantages pour l'analytique
- Maîtriser le déploiement et la configuration de ColumnStore
- Optimiser les requêtes OLAP pour tirer parti du stockage columnar
- Concevoir des schémas de data warehouse avec ColumnStore
- Intégrer ColumnStore avec InnoDB dans une architecture hybride OLTP/OLAP
- Identifier les cas d'usage appropriés et les limitations
- Configurer le MPP (Massively Parallel Processing) multi-nœuds
- Monitorer et optimiser les performances ColumnStore

---

## Introduction

**MariaDB ColumnStore** est un moteur de stockage **columnar** (orienté colonnes) conçu spécifiquement pour les charges de travail **OLAP** (Online Analytical Processing) et le **data warehousing**. Il a été développé à partir de la technologie InfiniDB, acquise par MariaDB en 2016.

### OLTP vs OLAP : Paradigmes opposés

```
┌────────────────────────────────────────────────────────────┐
│                    OLTP (InnoDB)                           │
│  • Transactions courtes                                    │
│  • Lectures/écritures de quelques lignes                   │
│  • Forte concurrence                                       │
│  • Latence faible (ms)                                     │
│  • Exemple : SELECT * FROM users WHERE id = 42             │
│                                                            │
│  Stockage ROW-BASED (orienté lignes) :                     │
│  [Row1: id=1, name='Alice', age=30, city='Paris']          │
│  [Row2: id=2, name='Bob', age=25, city='Lyon']             │
│  [Row3: id=3, name='Charlie', age=35, city='Paris']        │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                  OLAP (ColumnStore)                        │
│  • Requêtes analytiques longues                            │
│  • Scans de millions/milliards de lignes                   │
│  • Agrégations (SUM, AVG, COUNT)                           │
│  • Throughput élevé (pas latence)                          │
│  • Exemple : SELECT city, AVG(age)                         │
│             FROM users GROUP BY city                       │
│                                                            │
│  Stockage COLUMNAR (orienté colonnes) :                    │
│  Column 'id':     [1, 2, 3, ...]                           │
│  Column 'name':   ['Alice', 'Bob', 'Charlie', ...]         │
│  Column 'age':    [30, 25, 35, ...]                        │
│  Column 'city':   ['Paris', 'Lyon', 'Paris', ...]          │
└────────────────────────────────────────────────────────────┘
```

### Pourquoi le stockage columnar pour l'analytique ?

**Problème avec le stockage row-based** :

```sql
-- Requête analytique typique
SELECT
    region,
    SUM(revenue) AS total_revenue,
    AVG(quantity) AS avg_quantity
FROM sales  -- 1 milliard de lignes
WHERE date >= '2024-01-01'
GROUP BY region;

-- Avec stockage row-based (InnoDB) :
-- • Lit TOUTES les colonnes de TOUTES les lignes
-- • Même si on n'utilise que 3 colonnes (region, revenue, quantity)
-- • Lecture de 100+ colonnes inutiles
-- • I/O massif, performances dégradées
```

**Solution avec stockage columnar** :

```sql
-- Même requête avec ColumnStore :
-- • Lit UNIQUEMENT les colonnes nécessaires :
--   - date (filtrage WHERE)
--   - region (GROUP BY)
--   - revenue (SUM)
--   - quantity (AVG)
-- • Ignore les 100+ autres colonnes
-- • I/O réduit de 95%
-- • Compression très efficace (colonnes homogènes)
-- • Performance 10-100× supérieure
```

**Comparaison visuelle** :

```
Table avec 10 colonnes, requête sur 2 colonnes :

Row-based (InnoDB) :
Disque → [Row1: C1|C2|C3|C4|C5|C6|C7|C8|C9|C10]
         [Row2: C1|C2|C3|C4|C5|C6|C7|C8|C9|C10]
         [Row3: C1|C2|C3|C4|C5|C6|C7|C8|C9|C10]
         → Lit 100% des données pour utiliser 20%

Columnar (ColumnStore) :
Disque → [C1: val1, val2, val3, ...]
         [C2: val1, val2, val3, ...]  ← Lit uniquement C1 et C2
         [C3: ...] ← Ignoré
         [C4: ...] ← Ignoré
         ...
         → Lit 20% des données (10× moins d'I/O)
```

---

## Architecture de ColumnStore

### Vue d'ensemble du système

```
┌─────────────────────────────────────────────────────────────┐
│                   MariaDB Server (SQL Layer)                │
│  • Parser, Optimizer, Executor                              │
│  • InnoDB tables (OLTP)                                     │
└─────────────────────────────────────────────────────────────┘
                          ↓ Handler API
┌─────────────────────────────────────────────────────────────┐
│               ColumnStore Storage Engine                    │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         User Module (UM)                               │ │
│  │  • Interface SQL                                       │ │
│  │  • Query Parser spécialisé                             │ │
│  │  • Query Optimization (pushdown)                       │ │
│  │  • Résultats agrégés                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                          ↓                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │      Performance Module (PM) - Calcul parallèle        │ │
│  │  • Scan des colonnes                                   │ │
│  │  • Filtrage (WHERE)                                    │ │
│  │  • Agrégations locales (SUM, COUNT, etc.)              │ │
│  │  • Parallélisme massif                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                          ↓                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Extent Map & Block Manager                     │ │
│  │  • Métadonnées des extents                             │ │
│  │  • Distribution des données                            │ │
│  │  • Gestion des blocks (8 MB chacun)                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                 Stockage Physique (Disque)                  │
│  • Colonnes stockées séparément                             │
│  • Compression avancée (Snappy)                             │
│  • Blocks de 8 MB                                           │
│  • Métadonnées (min/max, histogrammes)                      │
└─────────────────────────────────────────────────────────────┘
```

### Composants principaux

#### 1. User Module (UM)

Le **User Module** est le point d'entrée pour les requêtes SQL :

```
Rôle du UM :
┌───────────────────────────────────────────┐
│ 1. Reçoit requête SQL du MariaDB Server   │
│    ↓                                      │
│ 2. Parse et analyse la requête            │
│    ↓                                      │
│ 3. Optimise pour ColumnStore              │
│    • Pushdown des filtres                 │
│    • Pushdown des agrégations             │
│    ↓                                      │
│ 4. Distribue aux Performance Modules      │
│    ↓                                      │
│ 5. Agrège les résultats finaux            │
│    ↓                                      │
│ 6. Retourne au MariaDB Server             │
└───────────────────────────────────────────┘
```

**Query Pushdown** : Optimisation clé de ColumnStore

```sql
-- Requête SQL
SELECT
    region,
    SUM(revenue) AS total
FROM sales
WHERE date >= '2024-01-01'
  AND status = 'completed'
GROUP BY region;

-- Sans pushdown (inefficace) :
-- 1. PM lit toutes les lignes
-- 2. Transfert de millions de lignes vers UM
-- 3. UM applique WHERE et GROUP BY
-- → Réseau saturé, lent

-- Avec pushdown (optimal) :
-- 1. UM envoie les filtres aux PM
-- 2. PM appliquent WHERE localement
-- 3. PM calculent SUM() localement par région
-- 4. PM envoient résultats agrégés à UM (quelques lignes)
-- 5. UM fait agrégation finale
-- → Réseau minimal, rapide
```

#### 2. Performance Module (PM)

Le **Performance Module** effectue le traitement parallèle des données :

```
Architecture MPP (Massively Parallel Processing) :
┌────────────────────────────────────────────────────────────┐
│                      User Module                           │
└────────────────────────────────────────────────────────────┘
          ↓                ↓                ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PM Node 1   │  │  PM Node 2   │  │  PM Node 3   │
│              │  │              │  │              │
│ Scan col X   │  │ Scan col X   │  │ Scan col X   │
│ WHERE filter │  │ WHERE filter │  │ WHERE filter │
│ SUM(revenue) │  │ SUM(revenue) │  │ SUM(revenue) │
│              │  │              │  │              │
│ Partition 1  │  │ Partition 2  │  │ Partition 3  │
└──────────────┘  └──────────────┘  └──────────────┘
          ↓                ↓                ↓
     [Résultat 1]    [Résultat 2]    [Résultat 3]
          ↓                ↓                ↓
          └────────────────┴────────────────┘
                           ↓
               Agrégation finale (UM)
```

**Parallélisme multi-niveaux** :
- **Inter-nœuds** : Plusieurs PM travaillent en parallèle
- **Intra-nœud** : Chaque PM utilise plusieurs threads
- **Vectorisation** : Traitement SIMD des données

#### 3. Extent Map & Metadata

ColumnStore utilise une structure hiérarchique pour organiser les données :

```
Hiérarchie ColumnStore :
┌───────────────────────────────────────────┐
│              Table                        │
│  (ex: sales)                              │
└───────────────────────────────────────────┘
                 ↓
┌───────────────────────────────────────────┐
│            Colonnes                       │
│  • date                                   │
│  • region                                 │
│  • revenue                                │
│  • quantity                               │
└───────────────────────────────────────────┘
                 ↓
┌───────────────────────────────────────────┐
│            Extents (64 MB)                │
│  • Unité de distribution                  │
│  • Répartis sur les PM                    │
│  • Métadonnées : min, max, histogramme    │
└───────────────────────────────────────────┘
                 ↓
┌───────────────────────────────────────────┐
│            Blocks (8 MB)                  │
│  • Unité de compression                   │
│  • Unité de lecture                       │
│  • ~8 millions de valeurs par block       │
└───────────────────────────────────────────┘
```

**Exemple de métadonnées d'Extent** :

```
Extent #42 de la colonne 'revenue' :
{
    extent_id: 42,
    column: "revenue",
    min_value: 10.00,
    max_value: 9999.99,
    num_rows: 8000000,
    compression: "Snappy",
    blocks: [
        { block_id: 1, offset: 0, size: 8388608 },
        { block_id: 2, offset: 8388608, size: 8388608 },
        ...
    ]
}
```

**Optimisation : Extent Elimination** :

```sql
SELECT SUM(revenue)
FROM sales
WHERE revenue > 10000;

-- ColumnStore lit l'Extent Map :
-- Extent #1 : min=10, max=500 → SKIP (max < 10000)
-- Extent #2 : min=100, max=5000 → SKIP
-- Extent #3 : min=5000, max=15000 → SCAN (contient valeurs > 10000)
-- Extent #4 : min=12000, max=50000 → SCAN
-- ...

-- Résultat : Scanne seulement 30% des extents
-- I/O réduit de 70%
```

### Compression

ColumnStore applique une **compression agressive** sur chaque colonne :

```
Pipeline de compression :
┌───────────────────────────────────────────┐
│ 1. Données brutes (ex: colonne 'city')    │
│    ['Paris', 'Lyon', 'Paris', ...]        │
│    ↓                                      │
│ 2. Encodage (Dictionary / RLE / etc.)     │
│    Paris → ID 1                           │
│    Lyon → ID 2                            │
│    [1, 2, 1, 1, 2, 1, ...]                │
│    ↓                                      │
│ 3. Compression (Snappy)                   │
│    Algorithme LZ77                        │
│    [compressed binary data]               │
│    ↓                                      │
│ 4. Stockage (8 MB blocks)                 │
│    Ratio compression : 5-20×              │
└───────────────────────────────────────────┘
```

**Techniques de compression** :

| Technique | Cas d'usage | Ratio typique |
|-----------|-------------|---------------|
| **Dictionary** | Colonnes avec cardinalité faible (ex: pays, statuts) | 10-50× |
| **Run-Length Encoding (RLE)** | Colonnes triées avec valeurs répétées | 5-100× |
| **Delta Encoding** | Colonnes avec valeurs incrémentales (timestamps, IDs) | 3-10× |
| **Snappy** | Compression finale, rapide décompression | 2-5× |

**Exemple de compression** :

```sql
-- Colonne 'country' : 100 millions de lignes
-- Valeurs : 'France', 'Germany', 'Spain', 'Italy', 'UK'

-- Sans compression :
-- 100M lignes × 7 bytes (moyenne) = 700 MB

-- Avec Dictionary Encoding :
-- Dictionary : 5 entrées × 10 bytes = 50 bytes
-- IDs : 100M × 1 byte = 100 MB
-- Total : 100 MB (compression 7×)

-- Avec Snappy en plus :
-- 100 MB → 25 MB (compression finale 28×)
```

💡 **Avantage clé** : Plus de données en cache = moins d'I/O = performances supérieures.

---

## Déploiement et installation

### Architecture de déploiement

ColumnStore peut être déployé en plusieurs modes :

**1. Single-node (développement, petits volumes)** :

```
┌────────────────────────────────┐
│     MariaDB Server             │
│  ┌──────────────────────────┐  │
│  │  InnoDB (OLTP tables)    │  │
│  └──────────────────────────┘  │
│  ┌──────────────────────────┐  │
│  │  ColumnStore             │  │
│  │  • UM + PM combinés      │  │
│  │  • OLAP tables           │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
```

**2. Multi-node MPP (production, gros volumes)** :

```
┌──────────────────────────────────────────────────────────┐
│              MariaDB UM Node (Query Coordinator)         │
│  • Reçoit requêtes                                       │
│  • Distribue aux PM                                      │
│  • Agrège résultats                                      │
└──────────────────────────────────────────────────────────┘
                          ↓
    ┌─────────────────────┴───────────────────┐
    ↓                     ↓                   ↓
┌─────────┐         ┌─────────┐         ┌─────────┐
│ PM Node1│         │ PM Node2│         │ PM Node3│
│         │         │         │         │         │
│ Disks   │         │ Disks   │         │ Disks   │
│ (RAID)  │         │ (RAID)  │         │ (RAID)  │
└─────────┘         └─────────┘         └─────────┘
 Partition 1        Partition 2        Partition 3
```

### Installation

```bash
# Installation via package manager (RHEL/CentOS/Rocky)
sudo yum install MariaDB-server MariaDB-columnstore-engine

# Debian/Ubuntu
sudo apt-get install mariadb-server mariadb-plugin-columnstore

# Initialisation ColumnStore
sudo /usr/bin/columnstore-post-install

# Configuration interactive
# 1. Single-node ou Multi-node ?
# 2. Répertoire de stockage ?
# 3. Taille du cache ?
# etc.
```

### Configuration de base

```ini
# /etc/mysql/mariadb.conf.d/columnstore.cnf
[mysqld]
# Activer ColumnStore
plugin_load_add = ha_columnstore

# Mémoire pour ColumnStore (30-70% RAM)
columnstore_cache_size = 32G

# Nombre de threads par PM
columnstore_max_threads = 16

# Taille du buffer d'importation
columnstore_import_buffer_size = 2G

# Mode compression
columnstore_compression_type = 2  # 0=none, 1=snappy(défaut), 2=lz4

# Logging
columnstore_use_import_for_batchinsert = ON
```

### Vérification

```sql
-- Vérifier que ColumnStore est chargé
SHOW ENGINES;
-- +--------------------+---------+-------------------------------+
-- | Engine             | Support | Comment                       |
-- +--------------------+---------+-------------------------------+
-- | ColumnStore        | YES     | ColumnStore storage engine    |
-- +--------------------+---------+-------------------------------+

-- Afficher les tables ColumnStore
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE
FROM information_schema.TABLES
WHERE ENGINE = 'ColumnStore';

-- Vérifier le statut du système
SELECT calgetstats();
```

---

## Utilisation de ColumnStore

### Création de tables

```sql
-- Créer une table ColumnStore
CREATE TABLE sales_fact (
    sale_id BIGINT,
    sale_date DATE,
    customer_id INT,
    product_id INT,
    region VARCHAR(50),
    revenue DECIMAL(10,2),
    quantity INT,
    cost DECIMAL(10,2)
) ENGINE=ColumnStore;

-- Pas d'index nécessaire ! (scan columnar optimisé)
-- Pas de clé primaire obligatoire
```

**Bonnes pratiques de design** :

```sql
-- ✅ BON : Table dénormalisée (star schema)
CREATE TABLE sales_analytics (
    date DATE,
    year INT,           -- Dénormalisation temporelle
    quarter INT,
    month INT,
    customer_name VARCHAR(100),
    customer_country VARCHAR(50),
    product_name VARCHAR(200),
    product_category VARCHAR(50),
    revenue DECIMAL(15,2),
    quantity INT
) ENGINE=ColumnStore;

-- ❌ ÉVITER : Tables normalisées avec JOINs complexes
-- ColumnStore préfère tables larges dénormalisées
```

### Chargement de données

#### 1. LOAD DATA (recommandé pour gros volumes)

```sql
-- Chargement depuis CSV
LOAD DATA INFILE '/data/sales.csv'
INTO TABLE sales_fact
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Performance : 500 000 - 2 000 000 lignes/seconde
-- (dépend du matériel et taille des lignes)
```

**Optimisations pour LOAD DATA** :

```sql
-- Désactiver temporairement les autocommits
SET autocommit = 0;
SET unique_checks = 0;
SET foreign_key_checks = 0;

-- Charger les données
LOAD DATA INFILE '/data/huge_file.csv' INTO TABLE sales_fact;

-- Réactiver
SET autocommit = 1;
SET unique_checks = 1;
SET foreign_key_checks = 1;
```

#### 2. cpimport (outil ColumnStore natif)

```bash
# Chargement ultra-rapide via outil natif
cpimport mydb sales_fact /data/sales.csv

# Options avancées
cpimport mydb sales_fact /data/sales.csv \
    -s ',' \              # Séparateur
    -E '"' \              # Enclosure
    -c 50000 \            # Batch size
    -j 8                  # Nombre de threads parallèles

# Performance : 5-10× plus rapide que LOAD DATA
# Jusqu'à 10 millions de lignes/seconde sur matériel moderne
```

#### 3. INSERT batch (pour petits volumes)

```sql
-- INSERT multiple rows (par batch de 10 000+)
INSERT INTO sales_fact VALUES
    (1, '2025-01-01', 100, 200, 'Paris', 1500.00, 10, 1000.00),
    (2, '2025-01-01', 101, 201, 'Lyon', 2500.00, 15, 1800.00),
    -- ... 9998 autres lignes ...
    (10000, '2025-01-01', 999, 299, 'Nice', 3500.00, 20, 2500.00);

-- Performance : 10 000 - 100 000 lignes/seconde
-- Acceptable pour flux continu, mais LOAD DATA/cpimport meilleurs pour bulk
```

### Requêtes analytiques

#### Agrégations simples

```sql
-- COUNT, SUM, AVG optimisés par ColumnStore
SELECT
    COUNT(*) AS total_sales,
    SUM(revenue) AS total_revenue,
    AVG(quantity) AS avg_quantity
FROM sales_fact
WHERE sale_date >= '2024-01-01';

-- Execution : < 1 seconde sur 100 millions de lignes
-- ColumnStore lit uniquement colonnes revenue, quantity, sale_date
```

#### GROUP BY et agrégations

```sql
-- Agrégation par région
SELECT
    region,
    COUNT(*) AS num_sales,
    SUM(revenue) AS total_revenue,
    AVG(revenue) AS avg_revenue,
    MIN(revenue) AS min_revenue,
    MAX(revenue) AS max_revenue
FROM sales_fact
WHERE sale_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY region
ORDER BY total_revenue DESC;

-- ColumnStore applique pushdown :
-- 1. PM filtrent par date (WHERE)
-- 2. PM calculent agrégations locales par région
-- 3. UM agrège les résultats finaux
-- → Très rapide même sur milliards de lignes
```

#### Window Functions

```sql
-- Classement des produits par revenue
SELECT
    product_id,
    product_name,
    SUM(revenue) AS total_revenue,
    RANK() OVER (ORDER BY SUM(revenue) DESC) AS revenue_rank,
    SUM(SUM(revenue)) OVER (ORDER BY SUM(revenue) DESC) AS cumulative_revenue
FROM sales_fact
WHERE sale_date >= '2024-01-01'
GROUP BY product_id, product_name;

-- Window functions bénéficient du stockage columnar
```

#### Requêtes complexes

```sql
-- Analyse multi-dimensionnelle
SELECT
    YEAR(sale_date) AS year,
    QUARTER(sale_date) AS quarter,
    region,
    product_category,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(revenue) AS total_revenue,
    SUM(quantity) AS total_quantity,
    SUM(revenue - cost) AS total_profit,
    AVG(revenue - cost) AS avg_profit_per_sale
FROM sales_fact
WHERE sale_date BETWEEN '2020-01-01' AND '2024-12-31'
GROUP BY
    YEAR(sale_date),
    QUARTER(sale_date),
    region,
    product_category
HAVING SUM(revenue) > 100000
ORDER BY year, quarter, total_revenue DESC;

-- Performance : 2-5 secondes sur 1 milliard de lignes
-- (vs plusieurs minutes avec InnoDB)
```

---

## Architecture hybride OLTP/OLAP

### Pattern ETL classique

```
┌────────────────────────────────────────────────────────┐
│                 Application OLTP                       │
│  ┌─────────────────────────────────────────────────┐   │
│  │          InnoDB Tables                          │   │
│  │  • orders (transactionnel)                      │   │
│  │  • customers                                    │   │
│  │  • products                                     │   │
│  │  → Optimisé pour INSERT/UPDATE/DELETE           │   │
│  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
                      ↓ ETL (Extract-Transform-Load)
                      ↓ (Batch nightly ou temps réel)
┌────────────────────────────────────────────────────────┐
│              Data Warehouse (Analytique)               │
│  ┌─────────────────────────────────────────────────┐   │
│  │        ColumnStore Tables                       │   │
│  │  • sales_fact (dénormalisée)                    │   │
│  │  • customer_dimension                           │   │
│  │  • product_dimension                            │   │
│  │  → Optimisé pour SELECT/agrégations             │   │
│  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
                      ↓
              ┌───────────────┐
              │  BI Tools     │
              │  • Tableau    │
              │  • PowerBI    │
              │  • Metabase   │
              └───────────────┘
```

### ETL avec MariaDB

```sql
-- Exemple : ETL quotidien OLTP → OLAP

-- 1. Extraction depuis InnoDB
CREATE TEMPORARY TABLE daily_sales AS
SELECT
    o.order_date,
    o.order_id,
    c.customer_name,
    c.customer_country,
    p.product_name,
    p.product_category,
    o.quantity,
    o.unit_price * o.quantity AS revenue,
    o.unit_cost * o.quantity AS cost
FROM orders o  -- InnoDB
JOIN customers c ON o.customer_id = c.customer_id  -- InnoDB
JOIN products p ON o.product_id = p.product_id     -- InnoDB
WHERE o.order_date = CURDATE() - INTERVAL 1 DAY;

-- 2. Transformation (déjà faite ci-dessus via JOIN)

-- 3. Chargement dans ColumnStore
INSERT INTO sales_fact_columnstore  -- ColumnStore
SELECT * FROM daily_sales;

-- Alternative : Export/Import pour meilleure performance
SELECT * FROM daily_sales
INTO OUTFILE '/tmp/daily_sales.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"';

-- Puis cpimport :
-- shell> cpimport warehouse sales_fact /tmp/daily_sales.csv
```

### Requêtes hybrides (Cross-Engine JOIN)

MariaDB permet de joindre tables InnoDB et ColumnStore :

```sql
-- JOIN entre InnoDB (OLTP) et ColumnStore (OLAP)
SELECT
    c.customer_name,           -- InnoDB (actuel)
    c.email,                   -- InnoDB
    SUM(s.revenue) AS lifetime_value  -- ColumnStore (historique)
FROM customers c               -- InnoDB (10 000 lignes)
JOIN sales_fact s ON c.customer_id = s.customer_id  -- ColumnStore (1B lignes)
WHERE s.sale_date >= '2020-01-01'
GROUP BY c.customer_id, c.customer_name, c.email
ORDER BY lifetime_value DESC
LIMIT 100;

-- ⚠️ Performance variable selon :
-- • Taille de la table InnoDB (petite = OK)
-- • Cardinalité du JOIN
-- • Préférer matérialiser les dimensions dans ColumnStore si possible
```

**Recommandation** : Dupliquer les dimensions dans ColumnStore pour éviter cross-engine JOINs :

```sql
-- Au lieu de :
-- customers (InnoDB) JOIN sales_fact (ColumnStore)

-- Préférer :
-- customer_dimension (ColumnStore) JOIN sales_fact (ColumnStore)
-- Sync via ETL quotidien
```

---

## Schéma en étoile (Star Schema)

ColumnStore est idéal pour les schémas de data warehouse classiques :

```
                    ┌──────────────────┐
                    │  Date Dimension  │
                    │  ──────────────  │
                    │  date_key (PK)   │
                    │  full_date       │
                    │  year            │
                    │  quarter         │
                    │  month           │
                    │  day_of_week     │
                    └──────────────────┘
                             ↑
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────────────┐   ┌──────────────┐   ┌───────────────┐
│   Customer     │   │   Fact Table │   │   Product     │
│   Dimension    │   │   (Sales)    │   │   Dimension   │
│   ────────     │   │   ────────   │   │   ────────    │
│customer_key(PK)│←──│date_key (FK) │──→│product_key(PK)│
│name            │   │customer_key  │   │name           │
│country         │   │product_key   │   │category       │
│segment         │   │quantity      │   │brand          │
└────────────────┘   │revenue       │   └───────────────┘
                     │cost          │
                     │profit        │
                     └──────────────┘
                             ↓
                    ┌──────────────────┐
                    │   Region         │
                    │   Dimension      │
                    │   ────────────   │
                    │   region_key(PK) │
                    │   region_name    │
                    │   country        │
                    │   continent      │
                    └──────────────────┘
```

**Implémentation en SQL** :

```sql
-- Fact Table (centre de l'étoile)
CREATE TABLE sales_fact (
    date_key INT,
    customer_key INT,
    product_key INT,
    region_key INT,
    quantity INT,
    revenue DECIMAL(15,2),
    cost DECIMAL(15,2),
    profit DECIMAL(15,2)
) ENGINE=ColumnStore;

-- Dimensions
CREATE TABLE date_dimension (
    date_key INT PRIMARY KEY,
    full_date DATE,
    year INT,
    quarter INT,
    month INT,
    day_of_week VARCHAR(10),
    is_weekend BOOLEAN
) ENGINE=ColumnStore;

CREATE TABLE customer_dimension (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    customer_name VARCHAR(200),
    country VARCHAR(50),
    segment VARCHAR(50)
) ENGINE=ColumnStore;

-- etc. pour product_dimension, region_dimension

-- Requête analytique typique (drill-down)
SELECT
    d.year,
    d.quarter,
    c.country,
    p.category,
    SUM(f.revenue) AS total_revenue,
    SUM(f.profit) AS total_profit
FROM sales_fact f
JOIN date_dimension d ON f.date_key = d.date_key
JOIN customer_dimension c ON f.customer_key = c.customer_key
JOIN product_dimension p ON f.product_key = p.product_key
WHERE d.year = 2024
GROUP BY d.year, d.quarter, c.country, p.category
ORDER BY total_revenue DESC;
```

---

## Configuration et tuning

### Paramètres critiques

```ini
[mysqld]
# ═══════════════════════════════════════════════════════
# COLUMNSTORE PERFORMANCE TUNING
# ═══════════════════════════════════════════════════════

# Mémoire cache (30-70% RAM système)
# Plus = plus de données en cache = moins d'I/O
columnstore_cache_size = 64G

# Threads de traitement par PM
# Recommandation : 1-2× nombre de CPU cores
columnstore_max_threads = 32

# Taille buffer pour LOAD DATA / cpimport
columnstore_import_buffer_size = 4G

# Mode compression (1=Snappy recommandé, 2=LZ4 plus rapide)
columnstore_compression_type = 1

# Utiliser cpimport pour batch inserts (plus rapide)
columnstore_use_import_for_batchinsert = ON

# Nombre de blocks à préfetch (I/O anticipé)
columnstore_num_blocks_pct = 50

# Hash join buffer size
columnstore_hash_join_size = 100M

# Taille disk-based join (si dépassement mémoire)
columnstore_disk_based_join = ON
columnstore_um_mem_limit = 0  # 0 = illimité (utilise columnstore_cache_size)
```

### Optimisations de requêtes

#### 1. Préférer les colonnes dans ORDER BY présentes dans GROUP BY

```sql
-- ✅ BON (évite tri final)
SELECT
    region,
    SUM(revenue) AS total
FROM sales_fact
GROUP BY region
ORDER BY region;  -- Déjà groupé par region

-- ⚠️ Moins optimal (tri supplémentaire)
SELECT
    region,
    SUM(revenue) AS total
FROM sales_fact
GROUP BY region
ORDER BY total DESC;  -- Tri sur agrégat
```

#### 2. Utiliser WHERE plutôt que HAVING quand possible

```sql
-- ✅ BON (filtrage précoce)
SELECT
    product_category,
    SUM(revenue) AS total
FROM sales_fact
WHERE revenue > 100  -- Pushdown vers PM
GROUP BY product_category;

-- ⚠️ Moins optimal (filtrage tardif)
SELECT
    product_category,
    SUM(revenue) AS total
FROM sales_fact
GROUP BY product_category
HAVING SUM(revenue) > 10000;  -- Après agrégation
```

#### 3. Limiter les JOINs cross-engine

```sql
-- ❌ À ÉVITER
SELECT *
FROM big_innodb_table i      -- InnoDB, 10M lignes
JOIN sales_fact c ON i.id = c.customer_id  -- ColumnStore, 1B lignes

-- ✅ PRÉFÉRER : Matérialiser dimensions dans ColumnStore
CREATE TABLE customer_dim_cs ENGINE=ColumnStore
    SELECT * FROM big_innodb_table;

SELECT *
FROM customer_dim_cs c
JOIN sales_fact s ON c.id = s.customer_id;
```

### Partitionnement (implicite)

ColumnStore partitionne automatiquement les données par **extent** (64 MB) :

```sql
-- Pas de PARTITION BY explicite nécessaire
-- ColumnStore distribue automatiquement les extents sur les PM

-- Requête bénéficie de l'extent elimination automatique
SELECT SUM(revenue)
FROM sales_fact
WHERE sale_date = '2024-06-15';

-- ColumnStore :
-- 1. Identifie les extents contenant des données de juin 2024
-- 2. Skip tous les autres extents (metadata min/max)
-- 3. Scan parallèle des extents pertinents seulement
```

---

## Monitoring et diagnostics

### Métriques système

```sql
-- Vue globale du système ColumnStore
SELECT calgetstats();
-- Retourne JSON avec :
-- • Nombre de PM actifs
-- • Mémoire utilisée
-- • Nombre de requêtes en cours
-- • Statistiques I/O

-- Version détaillée
SELECT calgetstats('verbose');

-- Statistiques par table
SELECT
    table_schema,
    table_name,
    table_rows,
    data_length / 1024 / 1024 / 1024 AS size_gb
FROM information_schema.tables
WHERE engine = 'ColumnStore';
```

### Analyse de requêtes

```sql
-- EXPLAIN pour ColumnStore
EXPLAIN
SELECT
    region,
    SUM(revenue)
FROM sales_fact
WHERE sale_date >= '2024-01-01'
GROUP BY region;

-- Résultat montre :
-- • Colonnes accédées
-- • Filtres appliqués (pushdown)
-- • Agrégations pushdown
```

### Logs et debug

```bash
# Logs ColumnStore
tail -f /var/log/mariadb/columnstore/debug.log

# Activer logging détaillé
calsetlogconfig --debug

# Statistiques en temps réel
watch -n 1 'echo "SELECT calgetstats();" | mysql -u root -p'
```

---

## Limitations et cas d'usage

### ✅ Idéal pour

1. **Data Warehouse et OLAP**
   - Tables de faits avec milliards de lignes
   - Requêtes d'agrégation (SUM, AVG, COUNT)
   - Analyses multi-dimensionnelles
   - Rapports BI

2. **Analytics en temps réel**
   - Dashboards avec requêtes complexes
   - Analyses ad-hoc
   - Exploration de données

3. **IoT et time-series**
   - Logs applicatifs (millions de lignes/jour)
   - Données de capteurs
   - Métriques système

4. **ETL et data pipelines**
   - Transformation de données massives
   - Agrégations intermédiaires

### ❌ Ne PAS utiliser pour

1. **OLTP (transactions courtes)**
   ```sql
   -- ❌ Mauvais cas d'usage pour ColumnStore
   UPDATE orders SET status = 'shipped' WHERE order_id = 12345;
   -- Utilisez InnoDB !
   ```

2. **Petites tables (< 1 million de lignes)**
   - Overhead de ColumnStore non justifié
   - InnoDB plus rapide pour petits volumes

3. **Requêtes point-lookup**
   ```sql
   -- ❌ Inefficace avec ColumnStore
   SELECT * FROM users WHERE id = 42;
   -- InnoDB avec index primaire est 1000× plus rapide
   ```

4. **Forte concurrence en écriture**
   - ColumnStore optimisé pour lecture
   - Écritures plus lentes qu'InnoDB
   - Batch inserts recommandés

5. **Tables avec modifications fréquentes (UPDATE/DELETE)**
   - ColumnStore n'a pas de vrai UPDATE en place
   - UPDATE = DELETE + INSERT
   - Performance dégradée

### Tableau comparatif : InnoDB vs ColumnStore

| Critère | InnoDB | ColumnStore |
|---------|--------|-------------|
| **Cas d'usage** | OLTP | OLAP |
| **Taille typique** | 1 KB - 100 GB | 100 GB - 100 TB |
| **Requêtes** | Point-lookup, range | Agrégations, scans |
| **INSERT** | ⭐⭐⭐⭐⭐ (rapide) | ⭐⭐⭐ (batch OK) |
| **UPDATE/DELETE** | ⭐⭐⭐⭐⭐ | ⭐ (lent) |
| **SELECT (row)** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **SELECT (agrégat)** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Compression** | 2-3× | 10-50× |
| **Transactions ACID** | ✅ Complet | ⚠️ Limité |
| **Index** | B-Tree, Hash, FTS | ❌ Pas d'index (scan optimisé) |
| **Concurrence** | Haute (row-lock) | Moyenne (extent-lock) |
| **Parallélisme** | Thread-pool | MPP natif |

---

## Exemples réels

### Exemple 1 : E-commerce analytics

```sql
-- Schéma
CREATE TABLE order_facts (
    order_date DATE,
    customer_country VARCHAR(50),
    product_category VARCHAR(100),
    product_sku VARCHAR(50),
    quantity INT,
    revenue DECIMAL(12,2),
    cost DECIMAL(12,2),
    shipping_cost DECIMAL(8,2)
) ENGINE=ColumnStore;

-- Chargement : 500 millions de commandes historiques
-- Taille : 200 GB non compressé → 15 GB compressé (13× compression)

-- Requête 1 : Top 10 produits par revenue
SELECT
    product_sku,
    SUM(revenue) AS total_revenue,
    SUM(quantity) AS total_quantity
FROM order_facts
WHERE order_date >= '2024-01-01'
GROUP BY product_sku
ORDER BY total_revenue DESC
LIMIT 10;
-- Temps : 0.8 secondes (vs 45 sec avec InnoDB)

-- Requête 2 : Analyse géographique mensuelle
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    customer_country,
    COUNT(DISTINCT product_sku) AS unique_products,
    SUM(revenue - cost - shipping_cost) AS net_profit
FROM order_facts
WHERE order_date BETWEEN '2023-01-01' AND '2024-12-31'
GROUP BY month, customer_country
ORDER BY month, net_profit DESC;
-- Temps : 2.3 secondes
```

### Exemple 2 : Logs applicatifs

```sql
-- Schéma
CREATE TABLE application_logs (
    log_timestamp DATETIME(6),
    application VARCHAR(50),
    severity VARCHAR(10),
    message TEXT,
    user_id INT,
    request_id VARCHAR(36),
    duration_ms INT
) ENGINE=ColumnStore;

-- Volume : 10 milliards de lignes, 5 TB compressé

-- Requête : Analyse des erreurs par application
SELECT
    application,
    severity,
    COUNT(*) AS error_count,
    AVG(duration_ms) AS avg_duration
FROM application_logs
WHERE log_timestamp >= NOW() - INTERVAL 7 DAY
  AND severity IN ('ERROR', 'CRITICAL')
GROUP BY application, severity
ORDER BY error_count DESC;
-- Temps : 3.5 secondes sur 7 jours de logs
```

### Exemple 3 : IoT time-series

```sql
-- Schéma
CREATE TABLE sensor_readings (
    reading_timestamp DATETIME(6),
    device_id VARCHAR(50),
    sensor_type VARCHAR(50),
    location VARCHAR(100),
    value DOUBLE,
    unit VARCHAR(20)
) ENGINE=ColumnStore;

-- Volume : 1 billion de mesures, 8 TB

-- Requête : Moyenne mobile par capteur et location
SELECT
    device_id,
    location,
    DATE(reading_timestamp) AS day,
    AVG(value) AS daily_avg,
    MIN(value) AS daily_min,
    MAX(value) AS daily_max,
    STDDEV(value) AS daily_stddev
FROM sensor_readings
WHERE sensor_type = 'temperature'
  AND reading_timestamp >= '2024-01-01'
GROUP BY device_id, location, day
ORDER BY device_id, day;
-- Temps : 12 secondes sur 1 milliard de lectures
```

---

## ✅ Points clés à retenir

1. **Stockage columnar** : Colonnes stockées séparément, optimise les requêtes analytiques (lecture seulement colonnes nécessaires).

2. **Compression massive** : Ratio 10-50× grâce à l'homogénéité des colonnes (Dictionary, RLE, Snappy).

3. **MPP (Massively Parallel Processing)** : Architecture distribuée avec User Module (UM) et Performance Modules (PM).

4. **Pas d'index** : Scan columnar optimisé avec extent elimination (métadonnées min/max).

5. **OLAP idéal** : Agrégations (SUM, AVG, COUNT), GROUP BY, analytique sur milliards de lignes.

6. **Pushdown optimization** : Filtres et agrégations exécutés sur les PM (réduction transferts réseau).

7. **Architecture hybride** : Combiner InnoDB (OLTP) et ColumnStore (OLAP) dans la même base.

8. **Chargement batch** : cpimport ou LOAD DATA pour insertion rapide (millions lignes/seconde).

9. **Limitations** : Pas pour OLTP, UPDATE/DELETE lents, pas d'index traditionnels.

10. **Star Schema** : Idéal pour schémas en étoile (fact table + dimensions).

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 ColumnStore Overview](https://mariadb.com/kb/en/columnstore/)
- [📖 ColumnStore Architecture](https://mariadb.com/kb/en/columnstore-architecture/)
- [📖 ColumnStore Installation](https://mariadb.com/kb/en/installing-and-configuring-columnstore/)
- [📖 ColumnStore Functions](https://mariadb.com/kb/en/columnstore-sql-structure-and-commands/)
- [📖 cpimport Tool](https://mariadb.com/kb/en/columnstore-bulk-data-loading/)

### Articles techniques

- [ColumnStore Performance Tuning](https://mariadb.com/kb/en/columnstore-performance-tuning/)
- [OLAP vs OLTP: Understanding the Difference](https://mariadb.org/olap-vs-oltp/)
- [Star Schema Design for ColumnStore](https://mariadb.com/kb/en/columnstore-best-practices/)

### Outils et intégration

- **BI Tools** : Tableau, PowerBI, Metabase, Grafana
- **ETL** : Apache Airflow, Pentaho, Talend
- **Data Science** : Python (pandas, SQLAlchemy), R (RMariaDB)

---

## ➡️ Section suivante

**[7.6 Moteur S3 : Archivage données froides sur AWS S3/MinIO](/07-moteurs-de-stockage/06-moteur-s3.md)** : Découverte du moteur S3 pour archivage économique de données froides sur stockage objet.

Puis nous continuerons avec :
- **7.7** : Vector/HNSW pour recherche vectorielle IA 🆕
- **7.8** : Comparaison et choix du moteur approprié
- **7.9** : Conversion entre moteurs

---

**📌 Mémo Architecte** : "ColumnStore = OLAP, InnoDB = OLTP. Ne mélangez pas. Utilisez ETL pour passer de l'un à l'autre. Tables dénormalisées, star schema, et batch inserts sont vos amis."

**🎯 Règle d'or** : Si votre requête fait `SELECT COUNT(*), SUM(), AVG(), GROUP BY` sur des millions de lignes → ColumnStore. Si c'est `SELECT * WHERE id = X` ou `UPDATE` → InnoDB.

⏭️ [Moteur S3 : Archivage données froides sur AWS S3/MinIO](/07-moteurs-de-stockage/06-moteur-s3.md)

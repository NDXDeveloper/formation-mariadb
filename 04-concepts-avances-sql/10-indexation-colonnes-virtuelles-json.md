🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.10 Indexation de colonnes virtuelles extraites du JSON

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Connaissance JSON (4.7-4.9), index (5.1-5.2), colonnes générées

> **Recommandé** : MariaDB 11.4+ (améliorations performance 11.8)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les **colonnes virtuelles** (VIRTUAL vs STORED)
- Extraire des **données JSON** dans des colonnes traditionnelles
- Créer des **index efficaces** sur colonnes extraites
- Mesurer l'**impact performance** avec EXPLAIN
- Optimiser les **requêtes JSON** fréquentes
- Appliquer les **best practices** d'indexation JSON
- Résoudre des **problèmes de performance** concrets
- Choisir la **bonne stratégie** selon les cas d'usage

---

## Introduction

### Le problème : Performance des requêtes JSON

Sans index, MariaDB doit **scanner toutes les lignes** et parser le JSON à chaque requête.

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON
);

-- Insérer 100,000 produits
INSERT INTO products ...;

-- ❌ Requête LENTE : Full table scan + parsing JSON
SELECT id, JSON_EXTRACT(data, '$.name')
FROM products
WHERE JSON_EXTRACT(data, '$.price') < 100;

-- EXPLAIN montre : type=ALL, rows=100000 ⚠️
```

**Problèmes** :
- 🔴 **Full table scan** : Toutes les lignes lues
- 🔴 **Parsing JSON répété** : Pour chaque ligne
- 🔴 **Pas d'index** : Impossible d'indexer directement JSON
- 🔴 **Performances dégradées** : Linéaire O(n) avec la taille de la table

### La solution : Colonnes virtuelles indexées

```sql
-- ✅ Extraire dans une colonne, puis indexer
ALTER TABLE products
ADD COLUMN price DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) STORED,
ADD COLUMN product_name VARCHAR(200)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.name'))) STORED;

CREATE INDEX idx_price ON products(price);
CREATE INDEX idx_name ON products(product_name);

-- ✅ Requête RAPIDE : Utilise l'index
SELECT id, product_name
FROM products
WHERE price < 100;

-- EXPLAIN montre : type=range, key=idx_price, rows=50 ✅
```

**Gains** :
- ✅ **Index utilisable** : Requêtes logarithmiques O(log n)
- ✅ **Pas de parsing répété** : Données extraites une fois
- ✅ **Performance 10-100x** : Selon la sélectivité
- ✅ **Tri rapide** : ORDER BY utilise l'index

---

## Colonnes générées : VIRTUAL vs STORED

### Définitions

**Colonne générée (Generated Column)** : Colonne dont la valeur est calculée automatiquement à partir d'une expression.

| Type | Stockage | Calcul | Index | Performance |
|------|----------|--------|-------|-------------|
| **VIRTUAL** | ❌ Non stocké | À chaque lecture | ✅ Possible | Calcul répété |
| **STORED** | ✅ Stocké sur disque | À l'insertion/MAJ | ✅ Possible | Plus rapide |

### VIRTUAL : Calcul à la volée

```sql
ALTER TABLE products
ADD COLUMN price_virtual DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) VIRTUAL;

-- Stockage : 0 bytes supplémentaires
-- Calcul : À chaque SELECT
-- Index : Possible mais avec overhead
```

**Avantages** :
- ✅ Pas de stockage supplémentaire
- ✅ Toujours synchronisé avec data

**Inconvénients** :
- ❌ Calcul répété à chaque lecture
- ❌ Index VIRTUAL plus lent que STORED

### STORED : Valeur persistée

```sql
ALTER TABLE products
ADD COLUMN price_stored DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) STORED;

-- Stockage : Oui (comme une colonne normale)
-- Calcul : Seulement à INSERT/UPDATE
-- Index : Rapide comme une colonne normale
```

**Avantages** :
- ✅ Pas de recalcul à la lecture
- ✅ Index très performants
- ✅ Utilisable dans contraintes FOREIGN KEY

**Inconvénients** :
- ❌ Espace disque supplémentaire
- ❌ Ralentit légèrement INSERT/UPDATE

### Comparaison de performance

```sql
-- Benchmark : 100,000 lignes
-- Requête : SELECT WHERE price < 100

-- Sans colonne générée (baseline)
SELECT COUNT(*) FROM products
WHERE JSON_EXTRACT(data, '$.price') < 100;
-- Temps : 2500ms (full scan + parsing)

-- Avec colonne VIRTUAL indexée
SELECT COUNT(*) FROM products_virtual
WHERE price_virtual < 100;
-- Temps : 150ms (index + calcul à la lecture)

-- Avec colonne STORED indexée
SELECT COUNT(*) FROM products_stored
WHERE price_stored < 100;
-- Temps : 5ms (index pur, pas de calcul)
```

💡 **Recommandation générale** :
- **STORED** pour colonnes fréquemment utilisées en WHERE/ORDER BY
- **VIRTUAL** pour colonnes rarement utilisées ou espace limité

---

## Création de colonnes virtuelles

### Syntaxe de base

```sql
ALTER TABLE table_name
ADD COLUMN column_name data_type
    AS (expression) [VIRTUAL | STORED];
```

### Extraction de valeurs simples

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON
);

-- Prix (number)
ALTER TABLE products
ADD COLUMN price DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) STORED;

-- Nom (string) - ATTENTION au UNQUOTE
ALTER TABLE products
ADD COLUMN name VARCHAR(200)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.name'))) STORED;

-- Ou avec l'opérateur ->>
ALTER TABLE products
ADD COLUMN name_v2 VARCHAR(200)
    AS (data->>'$.name') STORED;

-- Status (enum)
ALTER TABLE products
ADD COLUMN status VARCHAR(20)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.status'))) STORED;

-- En stock (boolean)
ALTER TABLE products
ADD COLUMN in_stock BOOLEAN
    AS (JSON_EXTRACT(data, '$.stock.quantity') > 0) STORED;
```

### Extraction de valeurs imbriquées

```sql
-- data.specs.cpu
ALTER TABLE products
ADD COLUMN cpu VARCHAR(100)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.specs.cpu'))) STORED;

-- data.dimensions.weight
ALTER TABLE products
ADD COLUMN weight DECIMAL(8,3)
    AS (JSON_EXTRACT(data, '$.dimensions.weight')) STORED;

-- Chemin profond
ALTER TABLE products
ADD COLUMN warehouse_location VARCHAR(50)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.stock.warehouses[0].location'))) STORED;
```

### Extraction d'éléments d'array

```sql
-- Premier tag
ALTER TABLE products
ADD COLUMN first_tag VARCHAR(50)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.tags[0]'))) STORED;

-- Nombre de reviews
ALTER TABLE products
ADD COLUMN review_count INT
    AS (JSON_LENGTH(JSON_EXTRACT(data, '$.reviews'))) STORED;
```

### Calculs sur données JSON

```sql
-- Prix TTC (price * 1.20)
ALTER TABLE products
ADD COLUMN price_ttc DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price') * 1.20) STORED;

-- Moyenne des ratings
ALTER TABLE products
ADD COLUMN avg_rating DECIMAL(3,2)
    AS (
        (SELECT AVG(rating)
         FROM JSON_TABLE(
             JSON_EXTRACT(data, '$.reviews'),
             '$[*]' COLUMNS(rating INT PATH '$.rating')
         ) AS r)
    ) STORED;

-- Texte concaténé
ALTER TABLE products
ADD COLUMN full_name VARCHAR(300)
    AS (CONCAT(
        JSON_UNQUOTE(JSON_EXTRACT(data, '$.brand')),
        ' - ',
        JSON_UNQUOTE(JSON_EXTRACT(data, '$.name'))
    )) STORED;
```

---

## Création d'index sur colonnes virtuelles

### Index simples

```sql
-- Index sur prix
CREATE INDEX idx_price ON products(price);

-- Index sur nom
CREATE INDEX idx_name ON products(name);

-- Index sur status
CREATE INDEX idx_status ON products(status);
```

### Index composites

```sql
-- Recherche par catégorie et prix
ALTER TABLE products
ADD COLUMN category VARCHAR(50)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.category'))) STORED;

CREATE INDEX idx_category_price ON products(category, price);

-- Requête utilise l'index composite
SELECT name, price
FROM products
WHERE category = 'electronics'
  AND price BETWEEN 100 AND 500
ORDER BY price;
```

### Index FULLTEXT pour recherche

```sql
-- Extraction du texte de description
ALTER TABLE products
ADD COLUMN description TEXT
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.description'))) STORED;

-- Index FULLTEXT pour recherche textuelle
CREATE FULLTEXT INDEX idx_ft_description ON products(description);

-- Recherche full-text
SELECT name, price
FROM products
WHERE MATCH(description) AGAINST('laptop gaming' IN BOOLEAN MODE);
```

### Index sur expressions

```sql
-- Index sur année extraite de date
ALTER TABLE orders
ADD COLUMN order_year INT
    AS (YEAR(JSON_UNQUOTE(JSON_EXTRACT(data, '$.order_date')))) STORED;

CREATE INDEX idx_order_year ON orders(order_year);

-- Requête par année
SELECT COUNT(*), order_year
FROM orders
GROUP BY order_year;
```

---

## Analyse de performance

### Mesurer l'impact avec EXPLAIN

```sql
-- AVANT : Sans colonne virtuelle
EXPLAIN SELECT id, JSON_EXTRACT(data, '$.name') AS name
FROM products
WHERE JSON_EXTRACT(data, '$.price') < 100;
```

**Résultat** :
```
+------+-------------+----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table    | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | products | ALL  | NULL          | NULL | NULL    | NULL | 100000 | Using where |
+------+-------------+----------+------+---------------+------+---------+------+--------+-------------+
```

⚠️ **Problèmes** :
- `type: ALL` = Full table scan
- `rows: 100000` = Toutes les lignes scannées
- `key: NULL` = Aucun index utilisé

```sql
-- APRÈS : Avec colonne virtuelle indexée
EXPLAIN SELECT id, name
FROM products
WHERE price < 100;
```

**Résultat** :
```
+------+-------------+----------+-------+---------------+-----------+---------+------+------+-----------------------+
| id   | select_type | table    | type  | possible_keys | key       | key_len | ref  | rows | Extra                 |
+------+-------------+----------+-------+---------------+-----------+---------+------+------+-----------------------+
|    1 | SIMPLE      | products | range | idx_price     | idx_price | 6       | NULL |   50 | Using index condition |
+------+-------------+----------+-------+---------------+-----------+---------+------+------+-----------------------+
```

✅ **Améliorations** :
- `type: range` = Range scan sur index
- `key: idx_price` = Index utilisé
- `rows: 50` = Seulement 50 lignes (au lieu de 100k)
- **Gain : 2000x moins de lignes lues**

### Benchmark réel

```sql
-- Créer table de test avec 100,000 produits
CREATE TABLE products_test (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data JSON
);

-- Script d'insertion (simplifié)
INSERT INTO products_test (data)
SELECT JSON_OBJECT(
    'name', CONCAT('Product ', n),
    'price', ROUND(RAND() * 2000, 2),
    'category', ELT(FLOOR(RAND() * 5) + 1, 'electronics', 'books', 'clothing', 'home', 'sports'),
    'stock', JSON_OBJECT('quantity', FLOOR(RAND() * 100))
)
FROM (
    -- Générer 100,000 nombres
    SELECT @row := @row + 1 AS n
    FROM (SELECT 0 UNION ALL SELECT 1 /* ... */) t1,
         (SELECT 0 UNION ALL SELECT 1 /* ... */) t2,
         -- ...
         (SELECT @row := 0) init
    LIMIT 100000
) numbers;

-- Test 1 : Sans index
SET @start = NOW(6);
SELECT COUNT(*) FROM products_test
WHERE JSON_EXTRACT(data, '$.price') < 100;
SET @time_no_index = TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000;

-- Test 2 : Avec colonne + index
ALTER TABLE products_test
ADD COLUMN price DECIMAL(10,2) AS (JSON_EXTRACT(data, '$.price')) STORED;
CREATE INDEX idx_price ON products_test(price);

SET @start = NOW(6);
SELECT COUNT(*) FROM products_test WHERE price < 100;
SET @time_with_index = TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000;

-- Comparaison
SELECT
    @time_no_index AS ms_without_index,
    @time_with_index AS ms_with_index,
    ROUND(@time_no_index / @time_with_index, 1) AS speedup;
```

**Résultats typiques** :
```
+-------------------+------------------+---------+
| ms_without_index  | ms_with_index    | speedup |
+-------------------+------------------+---------+
|           2847.3  |             4.2  |   677.9 |
+-------------------+------------------+---------+
```

💡 **Gain : 680x plus rapide !**

---

## Cas d'usage pratiques

### Exemple 1 : E-commerce - Recherche produits

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Colonnes extraites pour performance
    product_name VARCHAR(200) AS (data->>'$.name') STORED,
    price DECIMAL(10,2) AS (JSON_EXTRACT(data, '$.price')) STORED,
    category VARCHAR(50) AS (data->>'$.category') STORED,
    brand VARCHAR(100) AS (data->>'$.brand') STORED,
    in_stock BOOLEAN AS (JSON_EXTRACT(data, '$.stock.quantity') > 0) STORED,
    rating DECIMAL(3,2) AS (JSON_EXTRACT(data, '$.avg_rating')) STORED,

    -- Index pour requêtes fréquentes
    INDEX idx_category_price (category, price),
    INDEX idx_brand (brand),
    INDEX idx_in_stock (in_stock),
    INDEX idx_rating (rating),
    FULLTEXT INDEX idx_ft_name (product_name)
);

-- Requête 1 : Produits en stock dans une catégorie
SELECT product_name, price, brand
FROM products
WHERE category = 'electronics'
  AND in_stock = 1
  AND price < 1000
ORDER BY rating DESC
LIMIT 20;
-- Utilise idx_category_price + idx_in_stock

-- Requête 2 : Recherche par nom
SELECT product_name, price, category
FROM products
WHERE MATCH(product_name) AGAINST('+laptop +gaming' IN BOOLEAN MODE)
  AND in_stock = 1
ORDER BY rating DESC;
-- Utilise idx_ft_name + idx_in_stock

-- Requête 3 : Top produits par catégorie
SELECT
    category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price,
    MAX(rating) AS max_rating
FROM products
WHERE in_stock = 1
GROUP BY category
ORDER BY product_count DESC;
-- Utilise idx_category_price
```

### Exemple 2 : Analytics - Logs d'événements

```sql
CREATE TABLE user_events (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    event_data JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Extraction pour analytics
    event_type VARCHAR(50) AS (event_data->>'$.type') STORED,
    event_date DATE AS (DATE(created_at)) STORED,
    session_id VARCHAR(100) AS (event_data->>'$.session_id') STORED,
    page_url TEXT AS (event_data->>'$.page_url') STORED,

    -- Index analytics
    INDEX idx_user_date (user_id, event_date),
    INDEX idx_type_date (event_type, event_date),
    INDEX idx_session (session_id)
);

-- Analytics : Événements par type et jour
SELECT
    event_type,
    event_date,
    COUNT(*) AS event_count,
    COUNT(DISTINCT user_id) AS unique_users
FROM user_events
WHERE event_date BETWEEN '2025-01-01' AND '2025-01-31'
GROUP BY event_type, event_date
ORDER BY event_date, event_count DESC;
-- Utilise idx_type_date efficacement

-- Analytics : Parcours utilisateur
SELECT
    session_id,
    GROUP_CONCAT(event_type ORDER BY created_at) AS event_sequence
FROM user_events
WHERE user_id = 12345
  AND event_date = '2025-01-15'
GROUP BY session_id;
-- Utilise idx_user_date
```

### Exemple 3 : API - Configuration dynamique

```sql
CREATE TABLE api_configs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    service_name VARCHAR(100) UNIQUE NOT NULL,
    config JSON NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Extraction pour monitoring
    is_enabled BOOLEAN AS (JSON_EXTRACT(config, '$.enabled')) STORED,
    rate_limit INT AS (JSON_EXTRACT(config, '$.rate_limit.requests')) STORED,
    timeout_ms INT AS (JSON_EXTRACT(config, '$.timeout_ms')) STORED,
    environment VARCHAR(20) AS (config->>'$.environment') STORED,

    -- Index monitoring
    INDEX idx_enabled (is_enabled),
    INDEX idx_environment (environment),
    INDEX idx_rate_limit (rate_limit)
);

-- Monitoring : Services actifs avec rate limit élevé
SELECT
    service_name,
    rate_limit,
    timeout_ms,
    environment
FROM api_configs
WHERE is_enabled = 1
  AND rate_limit > 1000
  AND environment = 'production'
ORDER BY rate_limit DESC;
-- Utilise idx_enabled + idx_environment + idx_rate_limit
```

---

## Stratégies d'indexation

### Choisir quelles colonnes extraire

**Critères de décision** :

✅ **Extraire et indexer si** :
- Utilisé fréquemment dans WHERE
- Utilisé dans ORDER BY / GROUP BY
- Utilisé dans JOIN
- Colonne à faible cardinalité (enum, boolean)
- Requêtes critiques pour la performance

❌ **NE PAS extraire si** :
- Utilisé rarement (< 1% des requêtes)
- Données très volumineuses (LONGTEXT)
- Mise à jour très fréquente
- Espace disque limité

```sql
-- ✅ BON : Colonnes fréquemment filtrées
ALTER TABLE products
ADD COLUMN category VARCHAR(50) AS (...) STORED,
ADD COLUMN price DECIMAL(10,2) AS (...) STORED,
ADD COLUMN in_stock BOOLEAN AS (...) STORED;

CREATE INDEX idx_category ON products(category);
CREATE INDEX idx_price ON products(price);

-- ❌ MAUVAIS : Extraire tout "au cas où"
ALTER TABLE products
ADD COLUMN field1 VARCHAR(100) AS (...) STORED,
ADD COLUMN field2 VARCHAR(100) AS (...) STORED,
-- ... 50 colonnes inutilisées
```

### Index composites vs simples

```sql
-- Analyser les requêtes fréquentes
-- Requête A : WHERE category = ? AND price < ?
-- Requête B : WHERE category = ? ORDER BY price
-- Requête C : WHERE price < ?

-- ✅ Solution optimale
CREATE INDEX idx_category_price ON products(category, price);
-- Couvre requêtes A et B

CREATE INDEX idx_price ON products(price);
-- Couvre requête C

-- ❌ MAUVAIS : Index redondants
CREATE INDEX idx_category ON products(category);  -- Redondant avec idx_category_price
CREATE INDEX idx_price_category ON products(price, category);  -- Rarement utilisé
```

### Surveiller l'utilisation des index

```sql
-- Activer statistiques d'index (MariaDB 10.5+)
SET GLOBAL userstat = 1;

-- Après quelques jours en production
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    INDEX_NAME,
    ROWS_READ
FROM information_schema.INDEX_STATISTICS
WHERE TABLE_NAME = 'products'
ORDER BY ROWS_READ DESC;

-- Identifier les index inutilisés
SELECT
    INDEX_NAME,
    ROWS_READ
FROM information_schema.INDEX_STATISTICS
WHERE TABLE_NAME = 'products'
  AND ROWS_READ = 0;
-- Supprimer ces index pour économiser espace/performance INSERT
```

---

## Maintenance et mise à jour

### Ajouter des colonnes à une table existante

```sql
-- Table existante avec 1M lignes
CREATE TABLE products_existing (
    id INT PRIMARY KEY,
    data JSON
);

-- ⚠️ ATTENTION : ALTER sur grosse table peut être long
-- Estimer le temps avec EXPLAIN FORMAT=JSON

-- Option 1 : Ajout direct (bloquant)
ALTER TABLE products_existing
ADD COLUMN price DECIMAL(10,2) AS (JSON_EXTRACT(data, '$.price')) STORED;
-- Peut prendre plusieurs minutes sur grosse table

-- Option 2 : pt-online-schema-change (non-bloquant)
-- pt-online-schema-change --alter "ADD COLUMN price ..." D=mydb,t=products_existing

-- Puis créer l'index
CREATE INDEX idx_price ON products_existing(price);
```

### Recalculer les valeurs

Les colonnes STORED sont automatiquement mises à jour lors des UPDATE.

```sql
-- MAJ du JSON → Colonne virtuelle mise à jour automatiquement
UPDATE products
SET data = JSON_SET(data, '$.price', 150)
WHERE id = 123;

-- La colonne 'price' est automatiquement recalculée ✅
SELECT id, price FROM products WHERE id = 123;
-- price = 150
```

### Changer l'expression d'une colonne

```sql
-- Impossible de modifier directement
-- ❌ ALTER TABLE products MODIFY COLUMN price AS (...);

-- ✅ Solution : DROP puis ADD
ALTER TABLE products
DROP COLUMN price,
ADD COLUMN price DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price') * 1.20) STORED;  -- Nouvelle expression

-- Recréer l'index
CREATE INDEX idx_price ON products(price);
```

---

## Optimisations avancées

### Colonnes calculées complexes

```sql
-- Moyenne des ratings avec gestion NULL
ALTER TABLE products
ADD COLUMN avg_rating DECIMAL(3,2) AS (
    COALESCE(
        (SELECT AVG(rating)
         FROM JSON_TABLE(
             JSON_EXTRACT(data, '$.reviews'),
             '$[*]' COLUMNS(rating INT PATH '$.rating')
         ) AS r
        ),
        0
    )
) STORED;

-- Concaténation pour recherche full-text
ALTER TABLE products
ADD COLUMN search_text TEXT AS (
    CONCAT_WS(' ',
        JSON_UNQUOTE(JSON_EXTRACT(data, '$.name')),
        JSON_UNQUOTE(JSON_EXTRACT(data, '$.brand')),
        JSON_UNQUOTE(JSON_EXTRACT(data, '$.category')),
        JSON_UNQUOTE(JSON_EXTRACT(data, '$.description'))
    )
) STORED;

CREATE FULLTEXT INDEX idx_ft_search ON products(search_text);
```

### Covering indexes

```sql
-- Index covering pour éviter le retour à la table
CREATE INDEX idx_category_price_name
ON products(category, price, product_name);

-- Cette requête est ultra-rapide (index-only scan)
SELECT category, price, product_name
FROM products
WHERE category = 'electronics'
  AND price < 1000;
-- EXPLAIN montre "Using index" ✅
```

### Partitionnement + Colonnes virtuelles

```sql
-- Extraire l'année pour partitionner
ALTER TABLE orders
ADD COLUMN order_year INT
    AS (YEAR(JSON_UNQUOTE(JSON_EXTRACT(data, '$.order_date')))) STORED;

-- Partitionner par année
ALTER TABLE orders
PARTITION BY RANGE (order_year) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Requête utilise la partition pruning
SELECT * FROM orders WHERE order_year = 2024;
-- Scan seulement partition p2024
```

---

## Bonnes pratiques

### 1. Nommer clairement les colonnes

```sql
-- ❌ Noms vagues
ALTER TABLE t ADD COLUMN c1 VARCHAR(50) AS (...) STORED;
ALTER TABLE t ADD COLUMN c2 INT AS (...) STORED;

-- ✅ Noms descriptifs
ALTER TABLE products
ADD COLUMN product_category VARCHAR(50) AS (...) STORED,
ADD COLUMN unit_price_eur DECIMAL(10,2) AS (...) STORED,
ADD COLUMN is_available BOOLEAN AS (...) STORED;
```

### 2. Documenter les expressions

```sql
ALTER TABLE products
-- Extrait le prix en euros depuis le JSON
-- Utilisé pour : recherche par prix, tri, analytics
ADD COLUMN price_eur DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) STORED,

-- Extrait le statut de stock
-- Utilisé pour : filtrer produits disponibles
ADD COLUMN in_stock BOOLEAN
    AS (JSON_EXTRACT(data, '$.stock.quantity') > 0) STORED;
```

### 3. Tester avant production

```sql
-- 1. Créer table de test
CREATE TABLE products_test LIKE products;
INSERT INTO products_test SELECT * FROM products LIMIT 10000;

-- 2. Ajouter colonnes + index
ALTER TABLE products_test ADD COLUMN price ...;
CREATE INDEX idx_price ON products_test(price);

-- 3. Tester les requêtes
EXPLAIN SELECT ... FROM products_test WHERE price < 100;

-- 4. Benchmark
-- Comparer temps d'exécution

-- 5. Déployer en production
```

### 4. Surveiller l'espace disque

```sql
-- Taille avant
SELECT
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_NAME = 'products';

-- Ajouter colonnes STORED + index
-- ...

-- Taille après
-- Vérifier l'augmentation
```

### 5. Limiter le nombre de colonnes extraites

```sql
-- ⚠️ MAUVAIS : Extraire tout
ALTER TABLE products
ADD COLUMN field1 ... AS (...) STORED,
ADD COLUMN field2 ... AS (...) STORED,
-- ... 50 colonnes

-- ✅ BON : Seulement l'essentiel
ALTER TABLE products
ADD COLUMN price ... AS (...) STORED,      -- Utilisé dans 80% requêtes
ADD COLUMN category ... AS (...) STORED,   -- Utilisé dans 60% requêtes
ADD COLUMN in_stock ... AS (...) STORED;   -- Utilisé dans 50% requêtes
```

---

## Limitations et considérations

### Limitations des colonnes générées

⚠️ **Restrictions** :
- ❌ Pas de sous-requêtes corrélées (sauf dans STORED avec workarounds)
- ❌ Pas de fonctions non-déterministes (RAND(), NOW()) dans VIRTUAL
- ❌ Pas de variables utilisateur (@var)
- ❌ Expression doit être déterministe

```sql
-- ❌ ERREUR : Fonction non-déterministe
ALTER TABLE products
ADD COLUMN created_now TIMESTAMP
    AS (NOW()) STORED;
-- ERROR

-- ✅ OK : Utiliser une colonne normale avec DEFAULT
ALTER TABLE products
ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
```

### Impact sur INSERT/UPDATE

Les colonnes STORED **ralentissent légèrement** INSERT et UPDATE.

```sql
-- Benchmark INSERT : 10,000 lignes
-- Sans colonnes STORED : 1.2s
-- Avec 5 colonnes STORED : 1.8s (+50%)
-- Avec 20 colonnes STORED : 3.5s (+190%)
```

💡 **Recommandation** : Maximum 5-10 colonnes STORED par table.

### Espace disque

```sql
-- Estimer l'espace supplémentaire
-- Exemple : 1M lignes, 10 colonnes STORED moyenne 50 bytes
-- Espace = 1,000,000 * 50 * 10 = 500 MB données
-- + 30-50% pour les index = 650-750 MB total
```

---

## Migration depuis tables sans colonnes virtuelles

### Étape par étape

```sql
-- 1. Analyser les requêtes actuelles
-- Identifier les JSON_EXTRACT() fréquents
SELECT
    SUBSTRING(argument, 1, 100) AS query_pattern,
    COUNT(*) AS count
FROM mysql.slow_log
WHERE argument LIKE '%JSON_EXTRACT%'
GROUP BY query_pattern
ORDER BY count DESC;

-- 2. Créer table de test
CREATE TABLE products_optimized LIKE products;

-- 3. Ajouter colonnes + index
ALTER TABLE products_optimized
ADD COLUMN price DECIMAL(10,2) AS (...) STORED,
ADD COLUMN category VARCHAR(50) AS (...) STORED;

CREATE INDEX idx_category_price ON products_optimized(category, price);

-- 4. Copier données
INSERT INTO products_optimized SELECT * FROM products;

-- 5. Tester requêtes
-- Comparer EXPLAIN et temps

-- 6. Switcher en production (downtime minimal)
START TRANSACTION;
RENAME TABLE products TO products_old, products_optimized TO products;
COMMIT;

-- 7. Vérifier, puis supprimer l'ancienne
DROP TABLE products_old;
```

---

## ✅ Points clés à retenir

- 🚀 **Performance** : Index sur colonnes extraites = 10-1000x plus rapide
- 💾 **STORED vs VIRTUAL** : STORED pour performance, VIRTUAL pour espace
- 📊 **Sélectivité** : Extraire seulement colonnes fréquemment utilisées (WHERE, ORDER BY)
- 🔍 **EXPLAIN** : Toujours vérifier que l'index est utilisé
- 🏗️ **Index composites** : Mieux que plusieurs index simples
- ⚙️ **Maintenance** : Colonnes STORED automatiquement mises à jour
- 📈 **Cas d'usage** : E-commerce (recherche), analytics (filtrage), API (monitoring)
- ⚠️ **Limitations** : Ralentit INSERT/UPDATE, espace disque, expressions déterministes seulement
- 📏 **Best practice** : Max 5-10 colonnes STORED, documenter, tester avant prod
- 🔄 **Migration** : Table test → Benchmark → Switch en prod

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Generated (Virtual and Persistent/Stored) Columns](https://mariadb.com/kb/en/generated-columns/)
- [📖 JSON Functions](https://mariadb.com/kb/en/json-functions/)
- [📖 CREATE INDEX](https://mariadb.com/kb/en/create-index/)
- [📖 EXPLAIN](https://mariadb.com/kb/en/explain/)

### Optimisation
- [📖 Optimization and Indexes](https://mariadb.com/kb/en/optimization-and-indexes/)
- [Use The Index, Luke](https://use-the-index-luke.com/) - Guide complet indexation

### Outils
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit) - pt-online-schema-change
- [MySQLTuner](https://github.com/major/MySQLTuner-perl) - Analyse configuration

---


⏭️ [Expressions régulières (REGEXP, REGEXP_REPLACE, REGEXP_SUBSTR)](/04-concepts-avances-sql/11-expressions-regulieres.md)

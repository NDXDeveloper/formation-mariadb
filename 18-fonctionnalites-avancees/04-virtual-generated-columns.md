🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.4 Colonnes Virtuelles et Générées

> **Niveau** : Avancé  
> **Durée estimée** : 2-2.5 heures  
> **Prérequis** : Chapitres 2-5, compréhension des index et performance

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le concept de **colonnes générées** et leur utilité
- Maîtriser la différence fondamentale entre **VIRTUAL** et **STORED**
- Créer des colonnes générées avec **syntaxe AS (expression)**
- **Indexer des colonnes virtuelles** pour optimiser les requêtes
- Utiliser les colonnes générées pour **extraire des données JSON**
- Implémenter des **calculs dérivés** automatiques
- Optimiser les performances avec le bon choix VIRTUAL/STORED
- Comprendre les **limitations** et contraintes

---

## Introduction

Les **colonnes générées** (ou colonnes calculées) sont des colonnes dont la valeur est **automatiquement calculée** à partir d'autres colonnes de la même table. Elles offrent un moyen élégant de dénormaliser des données, d'extraire des informations, ou de précalculer des valeurs dérivées sans redondance manuelle.

### Qu'est-ce qu'une Colonne Générée ?

Une colonne générée :
1. ✅ **Ne stocke pas de valeur directe** (VIRTUAL) ou **stocke le résultat du calcul** (STORED)
2. ✅ **Se met à jour automatiquement** quand les colonnes source changent
3. ✅ **Peut être indexée** (crucial pour performance !)
4. ✅ **Est en lecture seule** - impossible d'assigner une valeur manuellement
5. ✅ **Suit une expression SQL déterministe**

**Métaphore** : Une colonne générée est comme une **formule Excel** - elle se recalcule automatiquement quand les cellules référencées changent.

### Pourquoi Utiliser des Colonnes Générées ?

**Problématiques résolues** :

1. **📊 Calculs Dérivés Automatiques**
   - Prix TTC à partir de prix HT et taux TVA
   - Nom complet à partir de prénom et nom
   - Age à partir de date de naissance
   
2. **🔍 Extraction de Données JSON**
   - Extraire attributs JSON en colonnes relationnelles
   - Indexer des champs JSON pour performance
   - Requêtes SQL standards sur données semi-structurées

3. **⚡ Optimisation de Requêtes**
   - Précalculer expressions complexes
   - Créer index sur valeurs dérivées
   - Éviter calculs répétés dans WHERE/ORDER BY

4. **🔄 Dénormalisation Contrôlée**
   - Maintenir cohérence automatique
   - Pas de triggers complexes
   - Pas de risque de données obsolètes

5. **🌐 Fonctions Métier Complexes**
   - Calculs de distance géographique
   - Transformations de formats
   - Validation et normalisation

**Avant les colonnes générées** :
```sql
-- Option 1 : Recalculer à chaque requête (lent)
SELECT 
  first_name,
  last_name,
  CONCAT(first_name, ' ', last_name) AS full_name
FROM users
WHERE CONCAT(first_name, ' ', last_name) LIKE '%Martin%';
-- Expression calculée 2 fois !

-- Option 2 : Stocker manuellement (risque incohérence)
UPDATE users 
SET full_name = CONCAT(first_name, ' ', last_name)
WHERE user_id = 123;
-- Si on oublie lors d'UPDATE de first_name → données incohérentes
```

**Avec colonnes générées** :
```sql
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  first_name VARCHAR(50),
  last_name VARCHAR(50),
  full_name VARCHAR(101) AS (CONCAT(first_name, ' ', last_name)) STORED,
  INDEX idx_full_name (full_name)
);
-- Mise à jour automatique, index possible, cohérence garantie !
```

---

## Concepts Fondamentaux

### Deux Types de Colonnes Générées

MariaDB supporte deux types de colonnes générées, avec des caractéristiques **radicalement différentes** :

#### 1. VIRTUAL (Colonnes Virtuelles)

```sql
column_name data_type AS (expression) VIRTUAL
```

**Caractéristiques** :
- ❌ **Pas de stockage physique** - valeur calculée à la lecture
- ✅ **Économie d'espace disque** - aucun octet utilisé
- ⚡ **Calcul à chaque SELECT** - peut être coûteux CPU
- ✅ **Écritures rapides** (INSERT/UPDATE) - pas de calcul
- ✅ **Indexable** (crucial !) - MariaDB stocke l'index, pas la colonne

**Quand utiliser VIRTUAL** :
- Colonnes rarement lues
- Expressions légères (CONCAT, UPPER, simple arithmétique)
- Tables avec beaucoup d'écritures
- Économie d'espace prioritaire

#### 2. STORED (Colonnes Stockées)

```sql
column_name data_type AS (expression) STORED
```

**Caractéristiques** :
- ✅ **Stockage physique** - valeur calculée et enregistrée sur disque
- ❌ **Consomme espace disque** - comme une colonne normale
- ✅ **Lecture rapide** - pas de calcul, valeur déjà disponible
- ⚡ **Calcul à l'écriture** (INSERT/UPDATE) - coût CPU
- ✅ **Indexable** - index sur valeur stockée

**Quand utiliser STORED** :
- Colonnes fréquemment lues
- Expressions coûteuses (fonctions complexes, JSON, REGEXP)
- Tables avec beaucoup de lectures
- Performance lecture prioritaire

### Syntaxe de Base

```sql
CREATE TABLE table_name (
  -- Colonnes normales
  column1 data_type,
  column2 data_type,
  
  -- Colonne générée VIRTUAL (défaut)
  generated_col1 data_type AS (expression),
  
  -- Colonne générée VIRTUAL (explicite)
  generated_col2 data_type AS (expression) VIRTUAL,
  
  -- Colonne générée STORED
  generated_col3 data_type AS (expression) STORED
);
```

**Points importants** :
- Si ni VIRTUAL ni STORED spécifié → **VIRTUAL par défaut**
- `AS (expression)` ou `GENERATED ALWAYS AS (expression)` (équivalent, SQL standard)
- Expression doit être **déterministe** (même input → même output)
- Expression peut référencer **uniquement des colonnes de la même table**

---

## Comparaison VIRTUAL vs STORED

### Tableau Comparatif Détaillé

| Aspect | VIRTUAL | STORED |
|--------|---------|--------|
| **Stockage disque** | 0 bytes | Selon type de données |
| **Calcul** | À chaque SELECT | À chaque INSERT/UPDATE |
| **Performance SELECT** | Plus lent (calcul) | Rapide (valeur stockée) |
| **Performance INSERT** | Rapide | Plus lent (calcul) |
| **Performance UPDATE** | Rapide | Plus lent (calcul) |
| **Indexation** | ✅ Possible | ✅ Possible |
| **Taille index** | Normale | Normale |
| **Usage mémoire** | Buffer pool minimal | Buffer pool normal |
| **Expressions complexes** | ⚠️ Impact performance | ✅ Recommandé |
| **Cas d'usage** | Lectures rares, écritures fréquentes | Lectures fréquentes |

### Benchmark Performance

```sql
-- Table de test : 1 million de lignes
CREATE TABLE perf_test (
  id INT PRIMARY KEY AUTO_INCREMENT,
  price DECIMAL(10,2),
  quantity INT,
  
  -- VIRTUAL : Calcul simple
  total_virtual DECIMAL(12,2) AS (price * quantity) VIRTUAL,
  
  -- STORED : Même calcul
  total_stored DECIMAL(12,2) AS (price * quantity) STORED
);

-- Remplir avec données
INSERT INTO perf_test (price, quantity)
SELECT 
  ROUND(RAND() * 1000, 2),
  FLOOR(RAND() * 100)
FROM seq_1_to_1000000;
```

**Résultats typiques** :

| Opération | VIRTUAL | STORED | Écart |
|-----------|---------|--------|-------|
| INSERT 1M rows | 45 sec | 52 sec | +15% |
| SELECT SUM(total) | 2.8 sec | 0.9 sec | -68% |
| SELECT WHERE total > 5000 | 3.1 sec | 0.3 sec | -90% |
| UPDATE 10K rows | 1.2 sec | 1.8 sec | +50% |
| Taille table | 50 MB | 58 MB | +16% |

💡 **Conclusion** : 
- VIRTUAL = Meilleure écriture, lecture plus lente (sauf si indexée)
- STORED = Écriture plus lente, lecture rapide
- **VIRTUAL indexée** = Compromis optimal dans beaucoup de cas !

---

## Syntaxe et Expressions

### Expressions Valides

```sql
-- Arithmétique
CREATE TABLE products (
  price DECIMAL(10,2),
  tax_rate DECIMAL(4,2),
  price_with_tax DECIMAL(10,2) AS (price * (1 + tax_rate)) STORED
);

-- Concaténation de chaînes
CREATE TABLE contacts (
  first_name VARCHAR(50),
  last_name VARCHAR(50),
  full_name VARCHAR(101) AS (CONCAT(first_name, ' ', last_name)) VIRTUAL
);

-- Fonctions de dates
CREATE TABLE people (
  birth_date DATE,
  age INT AS (YEAR(CURDATE()) - YEAR(birth_date)) VIRTUAL
);

-- Expressions conditionnelles
CREATE TABLE orders (
  amount DECIMAL(10,2),
  status ENUM('PENDING','PAID','CANCELLED'),
  is_valid BOOLEAN AS (status = 'PAID' AND amount > 0) VIRTUAL
);

-- Fonctions de chaînes
CREATE TABLE users (
  email VARCHAR(255),
  email_lowercase VARCHAR(255) AS (LOWER(email)) VIRTUAL,
  email_domain VARCHAR(100) AS (SUBSTRING_INDEX(email, '@', -1)) VIRTUAL
);

-- Extraction JSON
CREATE TABLE user_profiles (
  profile JSON,
  age INT AS (JSON_UNQUOTE(JSON_EXTRACT(profile, '$.age'))) VIRTUAL,
  city VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(profile, '$.address.city'))) VIRTUAL
);

-- Fonctions géographiques
CREATE TABLE locations (
  latitude DECIMAL(10,8),
  longitude DECIMAL(11,8),
  geo_point POINT AS (POINT(longitude, latitude)) STORED
);
```

### Expressions NON Permises

❌ **Fonctions non déterministes** :
```sql
-- INTERDIT : Résultat change à chaque appel
created_time TIMESTAMP AS (NOW()) VIRTUAL;
random_value INT AS (RAND() * 100) VIRTUAL;

-- Exception : Certaines fonctions de date sont permises
current_year INT AS (YEAR(CURDATE())) VIRTUAL;  -- OK
```

❌ **Sous-requêtes** :
```sql
-- INTERDIT : Pas de SELECT dans expression
total_orders INT AS (SELECT COUNT(*) FROM orders WHERE customer_id = id) VIRTUAL;
```

❌ **Colonnes d'autres tables** :
```sql
-- INTERDIT : Seulement colonnes de la même table
customer_name VARCHAR(100) AS (SELECT name FROM customers WHERE id = customer_id) VIRTUAL;
```

❌ **Variables et paramètres** :
```sql
-- INTERDIT : Pas de variables
calculated DECIMAL AS (price * @discount_rate) VIRTUAL;
```

✅ **Solution pour sous-requêtes** : Utiliser une vue au lieu d'une colonne générée.

---

## Indexation de Colonnes Générées

L'**indexation de colonnes générées** est une fonctionnalité **extrêmement puissante** qui transforme les colonnes VIRTUAL en outils d'optimisation.

### Pourquoi Indexer une Colonne Générée ?

**Problème** : Expression dans WHERE non indexable
```sql
-- Sans index : Full table scan
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- Scan 1M rows pour appliquer LOWER() à chaque ligne
```

**Solution** : Colonne générée indexée
```sql
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  email VARCHAR(255),
  email_lower VARCHAR(255) AS (LOWER(email)) VIRTUAL,
  INDEX idx_email_lower (email_lower)
);

-- Avec index : Index seek (rapide)
SELECT * FROM users WHERE email_lower = 'alice@example.com';
-- Utilise idx_email_lower → temps de recherche logarithmique
```

### Comment Indexer une Colonne Générée

```sql
-- Méthode 1 : Index dans CREATE TABLE
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  name_upper VARCHAR(100) AS (UPPER(name)) VIRTUAL,
  INDEX idx_name_upper (name_upper)
);

-- Méthode 2 : ALTER TABLE après création
ALTER TABLE products ADD INDEX idx_name_upper (name_upper);

-- Méthode 3 : CREATE INDEX
CREATE INDEX idx_name_upper ON products(name_upper);
```

### Index sur Colonnes VIRTUAL vs STORED

**Point crucial** : L'index d'une colonne VIRTUAL **stocke les valeurs calculées** dans l'index, même si la colonne elle-même ne stocke rien !

```sql
-- Colonne VIRTUAL avec index
CREATE TABLE test_virtual (
  id INT PRIMARY KEY,
  value VARCHAR(100),
  value_upper VARCHAR(100) AS (UPPER(value)) VIRTUAL,
  INDEX idx_upper (value_upper)
);
-- Stockage :
-- - Colonne value_upper : 0 bytes (VIRTUAL)
-- - Index idx_upper : stocke UPPER(value) pour chaque ligne
-- - Calcul : À l'INSERT/UPDATE (pour peupler l'index)

-- Colonne STORED avec index
CREATE TABLE test_stored (
  id INT PRIMARY KEY,
  value VARCHAR(100),
  value_upper VARCHAR(100) AS (UPPER(value)) STORED,
  INDEX idx_upper (value_upper)
);
-- Stockage :
-- - Colonne value_upper : X bytes (STORED)
-- - Index idx_upper : stocke valeur déjà stockée
-- - Calcul : À l'INSERT/UPDATE (pour colonne ET index)
```

**Comparaison** :

| Aspect | VIRTUAL indexée | STORED indexée |
|--------|----------------|----------------|
| Espace colonne | 0 | X bytes |
| Espace index | Y bytes | Y bytes |
| Total stockage | Y | X + Y |
| Performance SELECT | Rapide (index) | Rapide (index) |
| Performance INSERT | Modéré (calcul index) | Modéré (calcul colonne + index) |
| Économie d'espace | ✅ Meilleure | ⚠️ Stockage double |

💡 **Recommandation** : 
- Si colonne rarement lue SANS index → VIRTUAL sans index
- Si colonne filtrée/triée → **VIRTUAL avec index** (meilleur compromis)
- Si colonne fréquemment lue AVEC et SANS index → STORED avec index

---

## Cas d'Usage Détaillés

### 1. Extraction et Indexation de Champs JSON

**Problème** : Requêtes sur attributs JSON sont lentes sans index.

```sql
-- Table avec données JSON
CREATE TABLE user_profiles (
  user_id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50),
  profile JSON
);

-- Données exemple
INSERT INTO user_profiles (username, profile) VALUES
  ('alice', '{"age": 28, "city": "Paris", "interests": ["music", "sports"]}'),
  ('bob', '{"age": 35, "city": "Lyon", "interests": ["tech", "gaming"]}'),
  ('charlie', '{"age": 42, "city": "Paris", "interests": ["art", "culture"]}');

-- Requête lente : Extraction JSON pour chaque ligne
SELECT * FROM user_profiles
WHERE JSON_UNQUOTE(JSON_EXTRACT(profile, '$.city')) = 'Paris';
-- Full table scan + calcul JSON pour chaque ligne

-- Solution : Colonnes générées pour attributs fréquemment requêtés
ALTER TABLE user_profiles
  ADD COLUMN age INT AS (JSON_UNQUOTE(JSON_EXTRACT(profile, '$.age'))) VIRTUAL,
  ADD COLUMN city VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(profile, '$.city'))) VIRTUAL,
  ADD INDEX idx_city (city),
  ADD INDEX idx_age (age);

-- Requête rapide : Utilise index
SELECT * FROM user_profiles WHERE city = 'Paris';
EXPLAIN SELECT * FROM user_profiles WHERE city = 'Paris';
-- key: idx_city, Extra: Using index condition

-- Requête complexe optimisée
SELECT username, age, city
FROM user_profiles
WHERE city = 'Paris' AND age > 30
ORDER BY age;
-- Utilise idx_city + idx_age
```

**Cas d'usage avancé : Opérateur ->>** (MariaDB 10.2.3+)
```sql
-- Syntaxe raccourcie pour extraction JSON
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  data JSON,
  category VARCHAR(50) AS (data->>'$.category') VIRTUAL,
  price DECIMAL(10,2) AS (data->>'$.price') VIRTUAL,
  INDEX idx_category (category),
  INDEX idx_price (price)
);

-- ->>' équivalent à JSON_UNQUOTE(JSON_EXTRACT(...))
```

### 2. Calculs Dérivés et Business Logic

**Scénario** : Table de commandes avec calculs TTC, remises, totaux.

```sql
CREATE TABLE order_items (
  item_id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT,
  product_name VARCHAR(100),
  
  -- Données brutes
  quantity INT,
  unit_price DECIMAL(10,2),
  discount_percent DECIMAL(5,2) DEFAULT 0,
  tax_rate DECIMAL(4,2) DEFAULT 0.20,
  
  -- Calculs dérivés (STORED car fréquemment utilisés)
  subtotal DECIMAL(12,2) AS (quantity * unit_price) STORED,
  discount_amount DECIMAL(12,2) AS (subtotal * discount_percent / 100) STORED,
  subtotal_after_discount DECIMAL(12,2) AS (subtotal - discount_amount) STORED,
  tax_amount DECIMAL(12,2) AS (subtotal_after_discount * tax_rate) STORED,
  total DECIMAL(12,2) AS (subtotal_after_discount + tax_amount) STORED,
  
  -- Index sur total pour rapports
  INDEX idx_total (total),
  FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Insertion simple (calculs automatiques)
INSERT INTO order_items (order_id, product_name, quantity, unit_price, discount_percent)
VALUES (1, 'Widget', 10, 25.00, 5);

-- Résultat automatique :
-- subtotal = 250.00
-- discount_amount = 12.50
-- subtotal_after_discount = 237.50
-- tax_amount = 47.50
-- total = 285.00

-- Rapports optimisés
SELECT 
  order_id,
  COUNT(*) AS items_count,
  SUM(total) AS order_total
FROM order_items
GROUP BY order_id;
-- Utilise valeurs précalculées, très rapide

-- Recherche par montant (utilise index)
SELECT * FROM order_items WHERE total > 500 ORDER BY total DESC;
```

### 3. Normalisation et Validation de Données

**Scénario** : Garantir cohérence des formats (email, téléphone, codes postaux).

```sql
CREATE TABLE customers (
  customer_id INT PRIMARY KEY AUTO_INCREMENT,
  
  -- Données brutes (saisies utilisateur)
  email VARCHAR(255),
  phone VARCHAR(20),
  postal_code VARCHAR(10),
  
  -- Versions normalisées (VIRTUAL pour économie espace)
  email_normalized VARCHAR(255) AS (LOWER(TRIM(email))) VIRTUAL,
  phone_digits VARCHAR(20) AS (REGEXP_REPLACE(phone, '[^0-9]', '')) VIRTUAL,
  postal_code_normalized VARCHAR(5) AS (LPAD(postal_code, 5, '0')) VIRTUAL,
  
  -- Index sur versions normalisées pour recherche
  UNIQUE INDEX idx_email (email_normalized),
  INDEX idx_phone (phone_digits),
  INDEX idx_postal (postal_code_normalized)
);

-- Insertions avec formats variés
INSERT INTO customers (email, phone, postal_code) VALUES
  ('Alice@Example.COM  ', '01-23-45-67-89', '75001'),
  ('  bob@test.fr', '06 12 34 56 78', '1234'),
  ('charlie@DEMO.org', '+33 9 87 65 43 21', '69100');

-- Recherche normalisée (insensible à la casse, espaces, format)
SELECT * FROM customers WHERE email_normalized = 'alice@example.com';
-- Trouve 'Alice@Example.COM  '

SELECT * FROM customers WHERE phone_digits = '0123456789';
-- Trouve '01-23-45-67-89'

-- Vue pour exposition normalisée
CREATE VIEW customers_clean AS
SELECT 
  customer_id,
  email_normalized AS email,
  phone_digits AS phone,
  postal_code_normalized AS postal_code
FROM customers;
```

### 4. Dénormalisation Contrôlée pour Performance

**Scénario** : Éviter jointures coûteuses en précalculant agrégations.

```sql
-- Table orders avec compteurs/totaux calculés
CREATE TABLE orders (
  order_id INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT,
  order_date DATE,
  status ENUM('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED')
);

-- Table order_items (détails)
CREATE TABLE order_items (
  item_id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT,
  product_id INT,
  quantity INT,
  unit_price DECIMAL(10,2),
  FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Vue matérialisée via colonnes générées (alternative à table summary)
CREATE TABLE order_summary (
  order_id INT PRIMARY KEY,
  
  -- Comptages (nécessite trigger car sous-requête interdite)
  -- Voir exemple triggers ci-dessous
  items_count INT,
  total_amount DECIMAL(12,2),
  
  FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Alternative : Trigger pour maintenir cohérence
DELIMITER $$
CREATE TRIGGER update_order_summary_after_item_insert
AFTER INSERT ON order_items
FOR EACH ROW
BEGIN
  INSERT INTO order_summary (order_id, items_count, total_amount)
  VALUES (
    NEW.order_id,
    1,
    NEW.quantity * NEW.unit_price
  )
  ON DUPLICATE KEY UPDATE
    items_count = items_count + 1,
    total_amount = total_amount + (NEW.quantity * NEW.unit_price);
END$$

CREATE TRIGGER update_order_summary_after_item_delete
AFTER DELETE ON order_items
FOR EACH ROW
BEGIN
  UPDATE order_summary
  SET 
    items_count = items_count - 1,
    total_amount = total_amount - (OLD.quantity * OLD.unit_price)
  WHERE order_id = OLD.order_id;
END$$
DELIMITER ;
```

**Meilleure approche pour comptages** : Utiliser vues si performance acceptable, ou table summary avec triggers.

### 5. Fonctions Géographiques et Spatiales

**Scénario** : Recherche de proximité géographique.

```sql
CREATE TABLE locations (
  location_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  
  -- Coordonnées stockées séparément
  latitude DECIMAL(10,8),
  longitude DECIMAL(11,8),
  
  -- Point géographique généré (STORED car utilisé pour calculs spatiaux)
  coordinates POINT AS (POINT(longitude, latitude)) STORED,
  
  -- Index spatial
  SPATIAL INDEX idx_coordinates (coordinates)
);

-- Insertions
INSERT INTO locations (name, latitude, longitude) VALUES
  ('Tour Eiffel', 48.858844, 2.294351),
  ('Arc de Triomphe', 48.873792, 2.295028),
  ('Notre-Dame', 48.852968, 2.349902),
  ('Sacré-Cœur', 48.886705, 2.343104);

-- Recherche dans rayon (ex: 1 km autour d'un point)
SET @center = POINT(2.3, 48.86);  -- Approximativement centre Paris
SET @radius = 1000;  -- 1 km en mètres

SELECT 
  name,
  latitude,
  longitude,
  ST_Distance_Sphere(coordinates, @center) AS distance_meters
FROM locations
WHERE ST_Distance_Sphere(coordinates, @center) <= @radius
ORDER BY distance_meters;

-- Recherche optimisée avec MBR (Minimum Bounding Rectangle)
SELECT name
FROM locations
WHERE MBRContains(
  ST_Buffer(@center, @radius / 111320),  -- ~111320m par degré
  coordinates
);
```

### 6. Expressions Régulières et Validation

**Scénario** : Extraction de parties d'identifiants ou codes structurés.

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  
  -- SKU format: CAT-YYYY-NNNN (ex: ELC-2025-0123)
  sku VARCHAR(20) UNIQUE,
  
  -- Extraction automatique des composants
  category_code VARCHAR(3) AS (SUBSTRING_INDEX(sku, '-', 1)) VIRTUAL,
  year_code INT AS (CAST(SUBSTRING_INDEX(SUBSTRING_INDEX(sku, '-', 2), '-', -1) AS UNSIGNED)) VIRTUAL,
  sequence_number INT AS (CAST(SUBSTRING_INDEX(sku, '-', -1) AS UNSIGNED)) VIRTUAL,
  
  -- Validation (retourne 1 si valide, 0 sinon)
  sku_is_valid BOOLEAN AS (
    sku REGEXP '^[A-Z]{3}-[0-9]{4}-[0-9]{4}$'
  ) VIRTUAL,
  
  -- Index pour recherche par catégorie
  INDEX idx_category (category_code),
  INDEX idx_year (year_code)
);

-- Insertions
INSERT INTO products (sku) VALUES
  ('ELC-2025-0001'),
  ('FUR-2025-0042'),
  ('ELC-2024-9999');

-- Requêtes optimisées
SELECT * FROM products WHERE category_code = 'ELC';
SELECT * FROM products WHERE year_code = 2025;

-- Validation
SELECT sku, sku_is_valid FROM products;
```

---

## Performance et Considérations

### Choix VIRTUAL vs STORED : Arbre de Décision

```
Est-ce que la colonne sera souvent lue ?
│
├─ NON → VIRTUAL sans index
│         (économie maximale d'espace)
│
└─ OUI → La colonne sera-t-elle utilisée en WHERE/ORDER BY ?
          │
          ├─ OUI → Expression est-elle coûteuse (JSON, REGEXP, calculs complexes) ?
          │        │
          │        ├─ OUI → STORED avec index
          │        │        (évite recalcul fréquent)
          │        │
          │        └─ NON → VIRTUAL avec index
          │                 (compromis optimal)
          │
          └─ NON → Expression est-elle très coûteuse ?
                   │
                   ├─ OUI → STORED
                   │        (calcul à l'écriture uniquement)
                   │
                   └─ NON → VIRTUAL
                            (économie d'espace)
```

### Impact sur INSERT/UPDATE

```sql
-- Test : Impact de colonnes générées sur écritures
CREATE TABLE write_test_baseline (
  id INT PRIMARY KEY AUTO_INCREMENT,
  value1 VARCHAR(100),
  value2 VARCHAR(100),
  value3 VARCHAR(100)
);

CREATE TABLE write_test_virtual (
  id INT PRIMARY KEY AUTO_INCREMENT,
  value1 VARCHAR(100),
  value2 VARCHAR(100),
  value3 VARCHAR(100),
  computed VARCHAR(303) AS (CONCAT(value1, value2, value3)) VIRTUAL
);

CREATE TABLE write_test_stored (
  id INT PRIMARY KEY AUTO_INCREMENT,
  value1 VARCHAR(100),
  value2 VARCHAR(100),
  value3 VARCHAR(100),
  computed VARCHAR(303) AS (CONCAT(value1, value2, value3)) STORED
);

-- Benchmark INSERT 100K rows
-- Résultats typiques :
-- Baseline: 8.2 sec
-- VIRTUAL:  8.3 sec (+1%)
-- STORED:   9.5 sec (+16%)
```

**Overhead typique** :

| Opération | VIRTUAL | STORED |
|-----------|---------|--------|
| INSERT | +0-5% | +10-20% |
| UPDATE (colonne source) | +0-5% | +10-20% |
| UPDATE (autre colonne) | 0% | 0% |
| SELECT sans colonne générée | 0% | 0% |
| SELECT avec colonne générée | Variable | 0% |

### Limitations de Longueur d'Expression

```sql
-- Limitation : Expression max ~65,535 caractères
-- Mais en pratique, gardez expressions courtes et lisibles

-- ✅ Bon : Expression simple
full_name VARCHAR(101) AS (CONCAT(first_name, ' ', last_name)) VIRTUAL

-- ⚠️ Acceptable : Expression modérée
address_full TEXT AS (
  CONCAT_WS(', ', 
    street, 
    city, 
    postal_code, 
    country
  )
) VIRTUAL

-- ❌ Mauvais : Expression trop complexe (utiliser fonction stockée)
complex_calc DECIMAL AS (
  CASE 
    WHEN ... THEN ...
    WHEN ... THEN ...
    -- 50 lignes de CASE/WHEN
  END
) STORED
-- → Refactorer en fonction stockée, puis appeler dans colonne générée
```

---

## Limitations et Contraintes

### Ce Qui N'est PAS Supporté

❌ **Modification directe** :
```sql
-- INTERDIT : Colonnes générées en lecture seule
UPDATE users SET full_name = 'John Doe' WHERE user_id = 1;
-- ERROR: 'full_name' is generated always
```

❌ **Valeurs par défaut** :
```sql
-- INTERDIT : DEFAULT incompatible avec GENERATED
name_upper VARCHAR(100) AS (UPPER(name)) DEFAULT 'UNKNOWN';
-- ERROR: Cannot use DEFAULT with generated column
```

❌ **Colonnes générées référençant d'autres colonnes générées** :
```sql
-- INTERDIT en VIRTUAL (mais STORED possible sous conditions)
CREATE TABLE test (
  a INT,
  b INT AS (a * 2) VIRTUAL,
  c INT AS (b * 2) VIRTUAL  -- ERROR
);

-- STORED peut référencer STORED précédent
CREATE TABLE test (
  a INT,
  b INT AS (a * 2) STORED,
  c INT AS (b * 2) STORED  -- OK
);
```

❌ **AUTO_INCREMENT** :
```sql
-- INTERDIT
id INT AS (...) AUTO_INCREMENT  -- ERROR
```

### Contraintes sur Expressions

**Expressions déterministes uniquement** :
```sql
-- ✅ OK : Déterministe
price_double DECIMAL AS (price * 2) VIRTUAL

-- ❌ INTERDIT : Non déterministe
created_at TIMESTAMP AS (NOW()) VIRTUAL
random_id INT AS (FLOOR(RAND() * 1000000)) VIRTUAL
```

**Exception** : Quelques fonctions de date considérées déterministes :
```sql
-- ✅ OK : CURDATE() dans contexte approprié
current_year INT AS (YEAR(CURDATE())) VIRTUAL
-- Mais sera recalculé à chaque SELECT
```

---

## Best Practices

### 1. Préférer VIRTUAL avec Index quand Possible

```sql
-- ✅ Meilleur : VIRTUAL indexée (économie espace)
CREATE TABLE users (
  email VARCHAR(255),
  email_lower VARCHAR(255) AS (LOWER(email)) VIRTUAL,
  INDEX idx_email_lower (email_lower)
);

-- ⚠️ Acceptable : STORED si vraiment beaucoup de lectures
CREATE TABLE high_read_table (
  complex_field TEXT,
  extracted VARCHAR(100) AS (/* expression coûteuse */) STORED
);
```

### 2. Documenter les Expressions Complexes

```sql
-- ✅ Bon : Commenter logique métier
CREATE TABLE invoices (
  amount DECIMAL(10,2),
  tax_rate DECIMAL(4,2),
  
  -- Calcul TTC : montant HT * (1 + taux TVA)
  -- Exemple : 100 € HT avec TVA 20% = 120 € TTC
  amount_incl_tax DECIMAL(10,2) AS (amount * (1 + tax_rate)) STORED
);
```

### 3. Utiliser pour Extraction JSON Systématique

```sql
-- ✅ Pattern recommandé : Exposer attributs JSON fréquents
CREATE TABLE events (
  event_id INT PRIMARY KEY,
  event_data JSON,
  
  -- Extraction attributs critiques
  event_type VARCHAR(50) AS (event_data->>'$.type') VIRTUAL,
  event_timestamp DATETIME AS (event_data->>'$.timestamp') VIRTUAL,
  user_id INT AS (event_data->>'$.user_id') VIRTUAL,
  
  -- Index sur attributs exposés
  INDEX idx_type (event_type),
  INDEX idx_timestamp (event_timestamp),
  INDEX idx_user (user_id)
);
```

### 4. Éviter Expressions Trop Complexes

```sql
-- ❌ Mauvais : Logique trop complexe dans colonne
status VARCHAR(20) AS (
  CASE
    WHEN condition1 AND condition2 THEN 'STATUS_A'
    WHEN condition3 OR (condition4 AND condition5) THEN 'STATUS_B'
    WHEN EXISTS(/* sous-requête impossible */) THEN 'STATUS_C'
    -- ... 20 lignes supplémentaires
  END
) STORED;

-- ✅ Meilleur : Fonction stockée + colonne générée simple
DELIMITER $$
CREATE FUNCTION calculate_status(/* params */)
RETURNS VARCHAR(20)
DETERMINISTIC
BEGIN
  -- Logique complexe ici
  RETURN computed_status;
END$$
DELIMITER ;

CREATE TABLE items (
  -- ...
  status VARCHAR(20) AS (calculate_status(col1, col2, col3)) STORED
);
```

### 5. Tester Performance avant Production

```sql
-- Toujours benchmarker sur données réelles
-- Créer table de test avec volume réaliste
CREATE TABLE test_generated LIKE production_table;
ALTER TABLE test_generated 
  ADD COLUMN new_computed AS (...) VIRTUAL,
  ADD INDEX idx_computed (new_computed);

-- Insérer données de test
INSERT INTO test_generated SELECT * FROM production_table LIMIT 100000;

-- Mesurer performance
EXPLAIN SELECT * FROM test_generated WHERE new_computed = 'value';

-- Comparer temps d'exécution
SELECT BENCHMARK(10000, (SELECT COUNT(*) FROM test_generated WHERE new_computed = 'X'));
```

---

## Migration et Adoption

### Ajouter Colonne Générée à Table Existante

```sql
-- Table existante
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  price DECIMAL(10,2)
);

-- Données existantes
INSERT INTO products VALUES
  (1, 'Widget', 29.99),
  (2, 'Gadget', 49.99);

-- Ajouter colonne générée
ALTER TABLE products
  ADD COLUMN name_upper VARCHAR(100) AS (UPPER(name)) VIRTUAL;

-- Vérification : Calcul automatique pour lignes existantes
SELECT product_id, name, name_upper FROM products;
-- 1 | Widget | WIDGET
-- 2 | Gadget | GADGET

-- Ajouter index
ALTER TABLE products ADD INDEX idx_name_upper (name_upper);
```

### Migration VIRTUAL → STORED (ou inverse)

```sql
-- Impossible de modifier VIRTUAL ↔ STORED directement
-- Il faut supprimer et recréer

-- Étape 1 : Supprimer colonne VIRTUAL
ALTER TABLE products DROP COLUMN name_upper;

-- Étape 2 : Recréer en STORED
ALTER TABLE products 
  ADD COLUMN name_upper VARCHAR(100) AS (UPPER(name)) STORED;

-- Note : Opération potentiellement longue sur grande table
-- MariaDB doit calculer et stocker toutes les valeurs
```

**Alternative avec downtime minimal** :
```sql
-- Créer nouvelle colonne temporaire
ALTER TABLE products 
  ADD COLUMN name_upper_new VARCHAR(100) AS (UPPER(name)) STORED;

-- Copier index
CREATE INDEX idx_name_upper_new ON products(name_upper_new);

-- Basculer application vers nouvelle colonne
-- Puis supprimer ancienne
ALTER TABLE products DROP COLUMN name_upper;
ALTER TABLE products CHANGE name_upper_new name_upper VARCHAR(100) AS (UPPER(name)) STORED;
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **Colonnes générées** : Calculées automatiquement à partir d'autres colonnes
- ✅ **Deux types** : VIRTUAL (calcul à la lecture) vs STORED (calcul à l'écriture)
- ✅ **Syntaxe** : `column_name type AS (expression) [VIRTUAL|STORED]`
- ✅ **Lecture seule** : Impossible d'assigner valeur manuellement
- ✅ **Expressions déterministes** : Même input → même output

### VIRTUAL vs STORED
- ✅ **VIRTUAL** : 0 bytes stockage, calcul SELECT, écritures rapides
- ✅ **STORED** : Stockage disque, lectures rapides, écritures lentes
- ✅ **VIRTUAL indexée** : Compromis optimal dans la majorité des cas
- ✅ **Overhead** : VIRTUAL +0-5% INSERT, STORED +10-20% INSERT

### Indexation (CRUCIAL)
- ✅ **Colonnes VIRTUAL indexables** : Index stocke valeurs calculées
- ✅ **Performance** : VIRTUAL + index = rapidité STORED pour requêtes
- ✅ **Économie** : VIRTUAL + index < STORED en espace disque
- ✅ **Pattern optimal** : Colonne VIRTUAL avec index pour filtres/tris

### Cas d'Usage Principaux
- ✅ **Extraction JSON** : Exposer attributs JSON en colonnes relationnelles indexables
- ✅ **Calculs dérivés** : Prix TTC, totaux, sous-totaux automatiques
- ✅ **Normalisation** : email_lower, phone_digits, formats cohérents
- ✅ **Dénormalisation** : Précalculer agrégations (avec triggers si sous-requêtes)
- ✅ **Géo-spatial** : POINT généré à partir de lat/long

### Best Practices
- ✅ Préférer VIRTUAL + index sauf si vraiment beaucoup de lectures sans index
- ✅ Documenter expressions complexes (commentaires)
- ✅ Utiliser pour JSON systématiquement (performance ++)
- ✅ Éviter expressions trop complexes (fonction stockée à la place)
- ✅ Toujours benchmarker sur données réelles avant production

### Limitations
- ❌ Pas de modification directe (lecture seule)
- ❌ Expressions déterministes uniquement (pas NOW(), RAND())
- ❌ Pas de sous-requêtes
- ❌ Pas de colonnes d'autres tables
- ❌ Impossible AUTO_INCREMENT ou DEFAULT

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [Generated (Virtual and Persistent/Stored) Columns](https://mariadb.com/kb/en/generated-columns/) - Guide complet
- 📖 [CREATE TABLE - Generated Columns](https://mariadb.com/kb/en/create-table/#generated-columns) - Syntaxe
- 📖 [Invisible Columns](https://mariadb.com/kb/en/invisible-columns/) - Colonnes invisibles
- 📖 [JSON Functions](https://mariadb.com/kb/en/json-functions/) - Extraction JSON

### Performance et Optimisation
- 📝 [Indexing Generated Columns](https://mariadb.com/kb/en/generated-columns-index/)
- 📝 [Virtual vs Stored Columns Performance](https://mariadb.com/resources/blog/virtual-stored-columns-performance/)
- 📝 [JSON Indexing Best Practices](https://mariadb.com/kb/en/json-index-best-practices/)

### Cas d'Usage Avancés
- 📝 [Using Generated Columns with JSON](https://mariadb.com/resources/blog/generated-columns-json/)
- 📝 [Functional Indexes Alternative](https://mariadb.com/kb/en/functional-indexes/)
- 📝 [Spatial Data with Generated Columns](https://mariadb.com/kb/en/spatial-generated-columns/)

### Comparaison avec Autres SGBD
- 🔄 [MySQL Generated Columns](https://dev.mysql.com/doc/refman/8.0/en/create-table-generated-columns.html) - Quasi-identique
- 🔄 [PostgreSQL Generated Columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html) - Depuis v12
- 🔄 [SQL Server Computed Columns](https://docs.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table)

---

## ➡️ Sous-sections suivantes

### **18.4.1 VIRTUAL vs STORED**
Comparaison approfondie avec benchmarks détaillés, arbres de décision, et recommandations par cas d'usage.

### **18.4.2 Indexation de Colonnes Générées**
Techniques avancées d'indexation, index composites, covering indexes sur colonnes générées.

---


⏭️ [VIRTUAL vs STORED](/18-fonctionnalites-avancees/04.1-virtual-vs-stored.md)

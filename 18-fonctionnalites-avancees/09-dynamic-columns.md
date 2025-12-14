🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.9 Dynamic Columns

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** : Chapitre 6 (Types de données), compréhension JSON de base

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le concept de **dynamic columns** vs schéma fixe
- Utiliser les **fonctions COLUMN_*** (CREATE, GET, ADD, DELETE, LIST)
- Stocker des **données semi-structurées** avec flexibilité
- Comparer **dynamic columns vs JSON** (avantages/inconvénients)
- Implémenter des **modèles hybrides** SQL/NoSQL
- Optimiser les **performances** (indexation, requêtes)
- Migrer progressivement vers/depuis dynamic columns
- Résoudre les **limitations** et pièges courants

---

## Introduction

Les **dynamic columns** permettent de stocker des paires clé-valeur dans une seule colonne BLOB, offrant une flexibilité de schéma similaire aux bases NoSQL tout en conservant les avantages du SQL relationnel.

### Qu'est-ce qu'une Dynamic Column ?

**Définition** : Une colonne BLOB contenant des **données structurées** encodées au format binaire, où chaque enregistrement peut avoir un ensemble différent d'attributs.

```sql
-- Schéma fixe traditionnel
CREATE TABLE products_fixed (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  price DECIMAL(10,2),
  category VARCHAR(50),
  brand VARCHAR(50),
  -- Problème : Attributs spécifiques à certains produits
  -- Electronics : warranty_years, voltage
  -- Clothing : size, color, material
  -- Books : author, isbn, pages
);

-- Dynamic columns : Flexibilité
CREATE TABLE products_dynamic (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  price DECIMAL(10,2),
  category VARCHAR(50),
  -- Colonne dynamique pour attributs variables
  attributes BLOB
);

-- Exemple de contenu attributes (format binaire interne) :
-- Product Electronics : {warranty_years: 2, voltage: "220V", energy_class: "A++"}
-- Product Clothing : {size: "M", color: "blue", material: "cotton"}
-- Product Book : {author: "John Doe", isbn: "978-...", pages: 350}
```

### Pourquoi Utiliser Dynamic Columns ?

**Avantages** :

1. **🔄 Flexibilité de Schéma**
   - Attributs différents par enregistrement
   - Ajout d'attributs sans ALTER TABLE
   - Adaptation rapide aux besoins métier

2. **📦 Densité de Stockage**
   - Pas de colonnes NULL inutiles
   - Format binaire compact
   - Économie d'espace pour données éparses

3. **🚀 Rapidité de Développement**
   - Prototype rapide
   - Évolution sans migration lourde
   - Modèle hybride SQL/NoSQL

4. **🔧 Use Cases Spécifiques**
   - Données multi-tenant avec attributs variables
   - Métadonnées extensibles
   - Logs avec champs personnalisés
   - E-commerce avec produits hétérogènes

**Compromis** :
- ⚠️ Pas d'index direct sur attributs dynamiques
- ⚠️ Requêtes moins optimisées que colonnes natives
- ⚠️ Validation de schéma au niveau application
- ⚠️ Typage moins strict

**Quand utiliser** :
- ✅ Attributs très variables entre enregistrements
- ✅ Évolution fréquente du schéma
- ✅ Données éparses (< 30% colonnes remplies)
- ✅ Multi-tenant avec personnalisation

**Quand NE PAS utiliser** :
- ❌ Attributs stables et communs à tous
- ❌ Besoin d'index performants sur tous attributs
- ❌ Requêtes complexes sur attributs dynamiques
- ❌ Contraintes d'intégrité strictes requises

---

## Fonctions Dynamic Columns

### COLUMN_CREATE - Créer Dynamic Columns

**Syntaxe** :
```sql
COLUMN_CREATE(name1, value1 [AS type], name2, value2 [AS type], ...)
```

**Exemples** :
```sql
-- Créer dynamic column simple
SELECT COLUMN_CREATE('brand', 'Samsung', 'warranty', 2);
-- Retourne BLOB binaire

-- Avec types explicites
SELECT COLUMN_CREATE(
  'brand', 'Samsung' AS CHAR,
  'warranty_years', 2 AS INTEGER,
  'price', 599.99 AS DECIMAL,
  'release_date', '2025-01-15' AS DATE,
  'in_stock', TRUE AS INTEGER  -- Boolean = INTEGER
);

-- Insérer dans table
INSERT INTO products_dynamic (id, name, price, category, attributes)
VALUES (
  1,
  'Galaxy S25',
  799.99,
  'Electronics',
  COLUMN_CREATE(
    'brand', 'Samsung',
    'screen_size', 6.5 AS DECIMAL,
    'storage_gb', 256 AS INTEGER,
    'color', 'Black',
    'warranty_years', 2 AS INTEGER
  )
);

-- Produit vêtement avec attributs différents
INSERT INTO products_dynamic (id, name, price, category, attributes)
VALUES (
  2,
  'T-Shirt Classic',
  29.99,
  'Clothing',
  COLUMN_CREATE(
    'size', 'M',
    'color', 'Blue',
    'material', 'Cotton',
    'brand', 'Nike',
    'eco_friendly', 1 AS INTEGER  -- Boolean
  )
);
```

### COLUMN_GET - Lire Valeur

**Syntaxe** :
```sql
COLUMN_GET(dynamic_column_blob, name AS type)
```

**Exemples** :
```sql
-- Lire attribut simple
SELECT 
  id,
  name,
  COLUMN_GET(attributes, 'brand' AS CHAR) AS brand,
  COLUMN_GET(attributes, 'warranty_years' AS INTEGER) AS warranty
FROM products_dynamic
WHERE id = 1;

-- Résultat :
-- id | name        | brand   | warranty
--  1 | Galaxy S25  | Samsung | 2

-- Lire multiple attributs
SELECT 
  id,
  name,
  category,
  COLUMN_GET(attributes, 'brand' AS CHAR) AS brand,
  COLUMN_GET(attributes, 'size' AS CHAR) AS size,
  COLUMN_GET(attributes, 'color' AS CHAR) AS color,
  COLUMN_GET(attributes, 'storage_gb' AS INTEGER) AS storage
FROM products_dynamic;

-- NULL pour attributs inexistants
-- id | name           | category    | brand   | size | color | storage
--  1 | Galaxy S25     | Electronics | Samsung | NULL | Black | 256
--  2 | T-Shirt Classic| Clothing    | Nike    | M    | Blue  | NULL
```

### COLUMN_ADD - Ajouter/Modifier Attributs

**Syntaxe** :
```sql
COLUMN_ADD(dynamic_column_blob, name1, value1 [AS type], ...)
```

**Exemples** :
```sql
-- Ajouter nouvel attribut à produit existant
UPDATE products_dynamic
SET attributes = COLUMN_ADD(
  attributes,
  'special_offer', 1 AS INTEGER,
  'discount_pct', 15 AS INTEGER
)
WHERE id = 1;

-- Modifier attribut existant
UPDATE products_dynamic
SET attributes = COLUMN_ADD(
  attributes,
  'warranty_years', 3 AS INTEGER  -- Remplace 2 par 3
)
WHERE id = 1;

-- Ajouter multiple attributs en une fois
UPDATE products_dynamic
SET attributes = COLUMN_ADD(
  attributes,
  '5g_compatible', 1 AS INTEGER,
  'fast_charging', 1 AS INTEGER,
  'water_resistant', 'IP68' AS CHAR
)
WHERE category = 'Electronics';
```

### COLUMN_DELETE - Supprimer Attributs

**Syntaxe** :
```sql
COLUMN_DELETE(dynamic_column_blob, name1, name2, ...)
```

**Exemples** :
```sql
-- Supprimer un attribut
UPDATE products_dynamic
SET attributes = COLUMN_DELETE(attributes, 'special_offer')
WHERE id = 1;

-- Supprimer multiple attributs
UPDATE products_dynamic
SET attributes = COLUMN_DELETE(
  attributes,
  'special_offer',
  'discount_pct'
)
WHERE category = 'Electronics';

-- Nettoyer tous les attributs temporaires
UPDATE products_dynamic
SET attributes = COLUMN_DELETE(
  attributes,
  'temp_field1',
  'temp_field2',
  'debug_info'
);
```

### COLUMN_LIST - Lister Attributs

**Syntaxe** :
```sql
COLUMN_LIST(dynamic_column_blob)
```

**Exemples** :
```sql
-- Lister tous les attributs d'un produit
SELECT 
  id,
  name,
  COLUMN_LIST(attributes) AS attribute_names
FROM products_dynamic
WHERE id = 1;

-- Résultat :
-- id | name       | attribute_names
--  1 | Galaxy S25 | brand,screen_size,storage_gb,color,warranty_years

-- Compter nombre d'attributs
SELECT 
  id,
  name,
  LENGTH(COLUMN_LIST(attributes)) - LENGTH(REPLACE(COLUMN_LIST(attributes), ',', '')) + 1 AS attr_count
FROM products_dynamic;

-- Trouver produits avec attribut spécifique
SELECT id, name
FROM products_dynamic
WHERE COLUMN_LIST(attributes) LIKE '%warranty_years%';
```

### COLUMN_EXISTS - Vérifier Existence

**Syntaxe** :
```sql
COLUMN_EXISTS(dynamic_column_blob, name)
```

**Exemples** :
```sql
-- Vérifier si attribut existe
SELECT 
  id,
  name,
  COLUMN_EXISTS(attributes, 'warranty_years') AS has_warranty,
  COLUMN_EXISTS(attributes, 'size') AS has_size
FROM products_dynamic;

-- Résultat :
-- id | name            | has_warranty | has_size
--  1 | Galaxy S25      | 1            | 0
--  2 | T-Shirt Classic | 0            | 1

-- Filtrer sur existence attribut
SELECT id, name
FROM products_dynamic
WHERE COLUMN_EXISTS(attributes, 'eco_friendly') = 1;

-- Requête conditionnelle selon existence
SELECT 
  id,
  name,
  CASE 
    WHEN COLUMN_EXISTS(attributes, 'warranty_years') = 1
    THEN COLUMN_GET(attributes, 'warranty_years' AS INTEGER)
    ELSE 0
  END AS warranty
FROM products_dynamic;
```

### COLUMN_CHECK - Valider Format

**Syntaxe** :
```sql
COLUMN_CHECK(dynamic_column_blob)
```

**Exemples** :
```sql
-- Vérifier intégrité colonne dynamique
SELECT 
  id,
  name,
  COLUMN_CHECK(attributes) AS is_valid
FROM products_dynamic;

-- Retourne 1 si valide, 0 si corrompu

-- Filtrer enregistrements corrompus
SELECT id, name
FROM products_dynamic
WHERE COLUMN_CHECK(attributes) = 0;

-- Utiliser dans contrainte CHECK (MariaDB 10.2+)
ALTER TABLE products_dynamic
ADD CONSTRAINT chk_attributes CHECK (COLUMN_CHECK(attributes) = 1);
```

---

## Types de Données Supportés

### Types Primitifs

```sql
-- Types supportés dans COLUMN_CREATE / COLUMN_GET

-- Chaînes de caractères
COLUMN_CREATE('name', 'Value' AS CHAR)
COLUMN_GET(col, 'name' AS CHAR)

-- Entiers
COLUMN_CREATE('count', 42 AS INTEGER)
COLUMN_GET(col, 'count' AS INTEGER)

-- Décimaux
COLUMN_CREATE('price', 99.99 AS DECIMAL)
COLUMN_GET(col, 'price' AS DECIMAL)

-- Dates et temps
COLUMN_CREATE('created', '2025-01-15' AS DATE)
COLUMN_CREATE('updated', '2025-01-15 14:30:00' AS DATETIME)
COLUMN_CREATE('duration', '12:30:00' AS TIME)
COLUMN_GET(col, 'created' AS DATE)

-- Booléens (stockés comme INTEGER)
COLUMN_CREATE('active', 1 AS INTEGER)  -- TRUE
COLUMN_CREATE('deleted', 0 AS INTEGER) -- FALSE

-- Double (float)
COLUMN_CREATE('rating', 4.5 AS DOUBLE)
COLUMN_GET(col, 'rating' AS DOUBLE)
```

### Conversion de Types

```sql
-- Lecture avec conversion automatique
SELECT 
  COLUMN_GET(attributes, 'price' AS CHAR) AS price_str,      -- "99.99"
  COLUMN_GET(attributes, 'price' AS DECIMAL) AS price_num,   -- 99.99
  COLUMN_GET(attributes, 'count' AS CHAR) AS count_str,      -- "42"
  COLUMN_GET(attributes, 'count' AS INTEGER) AS count_num    -- 42
FROM products_dynamic;

-- NULL si conversion impossible
SELECT COLUMN_GET(
  COLUMN_CREATE('text', 'not a number'),
  'text' AS INTEGER
);
-- Résultat : NULL (conversion échoue silencieusement)
```

### Structures Imbriquées (Limité)

```sql
-- ⚠️ Dynamic columns ne supportent PAS structures imbriquées natives
-- Workaround : JSON dans CHAR

-- Stocker JSON comme chaîne
INSERT INTO products_dynamic (id, name, attributes)
VALUES (
  3,
  'Complex Product',
  COLUMN_CREATE(
    'specs', '{"cpu": "Intel i7", "ram": "16GB"}' AS CHAR,
    'features', '["wifi", "bluetooth", "usb-c"]' AS CHAR
  )
);

-- Lire et parser JSON
SELECT 
  id,
  name,
  JSON_EXTRACT(
    COLUMN_GET(attributes, 'specs' AS CHAR),
    '$.cpu'
  ) AS cpu
FROM products_dynamic
WHERE id = 3;
```

---

## Cas d'Usage Concrets

### 1. E-commerce Multi-Catégories

**Besoin** : Produits avec attributs variables selon catégorie.

```sql
-- Table produits avec attributs dynamiques
CREATE TABLE products (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  
  -- Attributs communs (colonnes fixes)
  name VARCHAR(255) NOT NULL,
  sku VARCHAR(50) UNIQUE,
  price DECIMAL(10,2) NOT NULL,
  category VARCHAR(50) NOT NULL,
  
  -- Attributs spécifiques catégorie (dynamic columns)
  category_attributes BLOB,
  
  -- Métadonnées
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  INDEX idx_category (category),
  INDEX idx_price (price)
);

-- Insérer produits de différentes catégories

-- Electronics
INSERT INTO products (name, sku, price, category, category_attributes)
VALUES (
  'Laptop Dell XPS 15',
  'DELL-XPS15-001',
  1499.99,
  'Electronics',
  COLUMN_CREATE(
    'brand', 'Dell',
    'processor', 'Intel Core i7-13700H',
    'ram_gb', 16 AS INTEGER,
    'storage_gb', 512 AS INTEGER,
    'screen_size', 15.6 AS DECIMAL,
    'warranty_years', 2 AS INTEGER,
    'energy_rating', 'A+' AS CHAR
  )
);

-- Clothing
INSERT INTO products (name, sku, price, category, category_attributes)
VALUES (
  'Jeans Slim Fit',
  'LEVI-511-BLU-32',
  79.99,
  'Clothing',
  COLUMN_CREATE(
    'brand', 'Levi''s',
    'size', '32x34',
    'color', 'Blue',
    'material', '98% Cotton, 2% Elastane',
    'fit', 'Slim',
    'care', 'Machine wash cold'
  )
);

-- Books
INSERT INTO products (name, sku, price, category, category_attributes)
VALUES (
  'Database Systems: The Complete Book',
  'BOOK-978-0131873254',
  89.99,
  'Books',
  COLUMN_CREATE(
    'author', 'Hector Garcia-Molina',
    'isbn', '978-0131873254',
    'publisher', 'Pearson',
    'pages', 1248 AS INTEGER,
    'edition', '2nd',
    'language', 'English',
    'publication_year', 2008 AS INTEGER
  )
);

-- Recherche par attributs dynamiques
-- Electronics avec RAM >= 16GB
SELECT 
  product_id,
  name,
  price,
  COLUMN_GET(category_attributes, 'brand' AS CHAR) AS brand,
  COLUMN_GET(category_attributes, 'ram_gb' AS INTEGER) AS ram_gb
FROM products
WHERE category = 'Electronics'
  AND COLUMN_GET(category_attributes, 'ram_gb' AS INTEGER) >= 16;

-- Clothing par taille
SELECT 
  product_id,
  name,
  COLUMN_GET(category_attributes, 'size' AS CHAR) AS size,
  COLUMN_GET(category_attributes, 'color' AS CHAR) AS color
FROM products
WHERE category = 'Clothing'
  AND COLUMN_GET(category_attributes, 'size' AS CHAR) = '32x34';

-- Books par auteur
SELECT 
  product_id,
  name,
  price,
  COLUMN_GET(category_attributes, 'author' AS CHAR) AS author
FROM products
WHERE category = 'Books'
  AND COLUMN_GET(category_attributes, 'author' AS CHAR) LIKE '%Garcia%';
```

### 2. Application Multi-Tenant

**Besoin** : Chaque tenant personnalise ses champs métier.

```sql
-- Table clients multi-tenant
CREATE TABLE customers (
  customer_id INT PRIMARY KEY AUTO_INCREMENT,
  tenant_id INT NOT NULL,
  
  -- Champs communs
  email VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  
  -- Champs personnalisés par tenant
  custom_fields BLOB,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE KEY uk_tenant_email (tenant_id, email),
  INDEX idx_tenant (tenant_id)
);

-- Tenant 1 (SaaS B2B) : Champs entreprise
INSERT INTO customers (tenant_id, email, first_name, last_name, custom_fields)
VALUES (
  1,
  'john@company.com',
  'John',
  'Doe',
  COLUMN_CREATE(
    'company_name', 'Acme Corp',
    'industry', 'Technology',
    'employee_count', 500 AS INTEGER,
    'annual_revenue', 5000000 AS DECIMAL,
    'account_manager', 'Alice Smith'
  )
);

-- Tenant 2 (E-commerce B2C) : Champs consommateur
INSERT INTO customers (tenant_id, email, first_name, last_name, custom_fields)
VALUES (
  2,
  'jane@example.com',
  'Jane',
  'Smith',
  COLUMN_CREATE(
    'birthdate', '1990-05-15' AS DATE,
    'gender', 'F',
    'loyalty_points', 1250 AS INTEGER,
    'preferred_category', 'Fashion',
    'newsletter_subscribed', 1 AS INTEGER
  )
);

-- Tenant 3 (Healthcare) : Champs patient
INSERT INTO customers (tenant_id, email, first_name, last_name, custom_fields)
VALUES (
  3,
  'patient@hospital.com',
  'Bob',
  'Johnson',
  COLUMN_CREATE(
    'patient_id', 'P-12345',
    'blood_type', 'A+',
    'allergies', 'Penicillin',
    'emergency_contact', 'Mary Johnson - 555-1234',
    'insurance_provider', 'Blue Cross'
  )
);

-- Requête tenant-specific
-- B2B : Companies avec > 100 employés
SELECT 
  customer_id,
  CONCAT(first_name, ' ', last_name) AS name,
  email,
  COLUMN_GET(custom_fields, 'company_name' AS CHAR) AS company,
  COLUMN_GET(custom_fields, 'employee_count' AS INTEGER) AS employees
FROM customers
WHERE tenant_id = 1
  AND COLUMN_GET(custom_fields, 'employee_count' AS INTEGER) > 100;

-- B2C : Clients avec points de fidélité
SELECT 
  customer_id,
  CONCAT(first_name, ' ', last_name) AS name,
  COLUMN_GET(custom_fields, 'loyalty_points' AS INTEGER) AS points
FROM customers
WHERE tenant_id = 2
  AND COLUMN_GET(custom_fields, 'loyalty_points' AS INTEGER) > 1000
ORDER BY points DESC;
```

### 3. Logs Applicatifs avec Métadonnées Variables

**Besoin** : Logs avec contexte variable selon type d'événement.

```sql
CREATE TABLE application_logs (
  log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  
  -- Champs fixes
  timestamp TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
  application VARCHAR(50),
  level ENUM('DEBUG','INFO','WARN','ERROR','FATAL'),
  message TEXT,
  
  -- Contexte variable selon type log
  context BLOB,
  
  INDEX idx_timestamp (timestamp),
  INDEX idx_app_level (application, level)
);

-- Log authentification
INSERT INTO application_logs (application, level, message, context)
VALUES (
  'auth-service',
  'INFO',
  'User login successful',
  COLUMN_CREATE(
    'user_id', 12345 AS INTEGER,
    'username', 'john.doe',
    'ip_address', '192.168.1.100',
    'user_agent', 'Mozilla/5.0...',
    'session_id', 'sess_abc123',
    'login_method', '2FA'
  )
);

-- Log erreur API
INSERT INTO application_logs (application, level, message, context)
VALUES (
  'api-gateway',
  'ERROR',
  'HTTP 500 Internal Server Error',
  COLUMN_CREATE(
    'endpoint', '/api/v1/orders',
    'method', 'POST',
    'status_code', 500 AS INTEGER,
    'response_time_ms', 1250 AS INTEGER,
    'error_type', 'DatabaseConnectionTimeout',
    'stack_trace', 'Error at line 42...'
  )
);

-- Log transaction e-commerce
INSERT INTO application_logs (application, level, message, context)
VALUES (
  'payment-service',
  'INFO',
  'Payment processed',
  COLUMN_CREATE(
    'order_id', 98765 AS INTEGER,
    'amount', 159.99 AS DECIMAL,
    'currency', 'EUR',
    'payment_method', 'credit_card',
    'transaction_id', 'txn_xyz789',
    'processor_response_code', '00'
  )
);

-- Analyse : Erreurs API par endpoint
SELECT 
  COUNT(*) AS error_count,
  COLUMN_GET(context, 'endpoint' AS CHAR) AS endpoint,
  AVG(COLUMN_GET(context, 'response_time_ms' AS INTEGER)) AS avg_response_ms
FROM application_logs
WHERE application = 'api-gateway'
  AND level = 'ERROR'
  AND timestamp >= NOW() - INTERVAL 1 DAY
GROUP BY endpoint
ORDER BY error_count DESC;

-- Analyse : Logins par méthode
SELECT 
  DATE(timestamp) AS login_date,
  COLUMN_GET(context, 'login_method' AS CHAR) AS method,
  COUNT(*) AS login_count
FROM application_logs
WHERE application = 'auth-service'
  AND message = 'User login successful'
  AND timestamp >= NOW() - INTERVAL 7 DAY
GROUP BY login_date, method
ORDER BY login_date, login_count DESC;
```

### 4. Configuration Utilisateur Extensible

**Besoin** : Préférences utilisateur évolutives sans migration.

```sql
CREATE TABLE user_preferences (
  user_id INT PRIMARY KEY,
  
  -- Préférences dynamiques
  settings BLOB,
  
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Utilisateur basique
INSERT INTO user_preferences (user_id, settings)
VALUES (
  1,
  COLUMN_CREATE(
    'theme', 'dark',
    'language', 'en',
    'notifications_email', 1 AS INTEGER
  )
);

-- Utilisateur avancé
INSERT INTO user_preferences (user_id, settings)
VALUES (
  2,
  COLUMN_CREATE(
    'theme', 'light',
    'language', 'fr',
    'notifications_email', 1 AS INTEGER,
    'notifications_sms', 0 AS INTEGER,
    'notifications_push', 1 AS INTEGER,
    'timezone', 'Europe/Paris',
    'date_format', 'DD/MM/YYYY',
    'currency', 'EUR',
    'items_per_page', 50 AS INTEGER,
    'default_view', 'grid',
    'keyboard_shortcuts', 1 AS INTEGER
  )
);

-- Ajouter nouvelle préférence à tous utilisateurs existants
UPDATE user_preferences
SET settings = COLUMN_ADD(
  settings,
  'dark_mode_auto', 1 AS INTEGER  -- Nouveau : Mode sombre automatique
)
WHERE COLUMN_EXISTS(settings, 'theme') = 1;

-- Lire préférences avec valeurs par défaut
SELECT 
  user_id,
  COALESCE(COLUMN_GET(settings, 'theme' AS CHAR), 'light') AS theme,
  COALESCE(COLUMN_GET(settings, 'language' AS CHAR), 'en') AS language,
  COALESCE(COLUMN_GET(settings, 'items_per_page' AS INTEGER), 25) AS items_per_page
FROM user_preferences
WHERE user_id = 1;

-- Fonction pour récupérer préférence avec défaut
DELIMITER $$
CREATE FUNCTION get_user_preference(
  p_user_id INT,
  p_key VARCHAR(100),
  p_default VARCHAR(255)
)
RETURNS VARCHAR(255)
DETERMINISTIC
READS SQL DATA
BEGIN
  DECLARE v_value VARCHAR(255);
  
  SELECT COLUMN_GET(settings, p_key AS CHAR)
  INTO v_value
  FROM user_preferences
  WHERE user_id = p_user_id;
  
  RETURN COALESCE(v_value, p_default);
END$$
DELIMITER ;

-- Utilisation
SELECT get_user_preference(1, 'theme', 'light') AS theme;
```

---

## Dynamic Columns vs JSON

### Comparaison Détaillée

| Aspect | Dynamic Columns | JSON |
|--------|----------------|------|
| **Format stockage** | Binaire compact | Texte (UTF-8) |
| **Taille** | Plus compact (~30% moins) | Plus volumineux |
| **Performance lecture** | Rapide (accès binaire) | Moyen (parsing texte) |
| **Performance écriture** | Moyen | Rapide |
| **Indexation** | Aucune (colonnes générées) | Native (MariaDB 10.2+) |
| **Validation** | Application | Native (JSON_VALID) |
| **Structures imbriquées** | Non supportées | ✅ Arrays, objets |
| **Fonctions** | COLUMN_* (10+) | JSON_* (50+) |
| **Standard** | MariaDB only | Standard (toutes BDD) |
| **Portabilité** | ❌ Faible | ✅ Élevée |
| **Lisibilité** | ❌ Binaire illisible | ✅ Texte lisible |

### Benchmarks

```sql
-- Test : 1M lignes, 10 attributs par ligne

-- Taille stockage
-- Dynamic Columns : 250 MB
-- JSON : 380 MB (-34%)

-- SELECT : Lire 3 attributs
-- Dynamic Columns : 1.2 sec
-- JSON : 1.8 sec (-33%)

-- INSERT : Créer 10K lignes
-- Dynamic Columns : 4.5 sec
-- JSON : 3.2 sec (+40%)

-- UPDATE : Modifier 1 attribut
-- Dynamic Columns : 5.8 sec (re-créer BLOB)
-- JSON : 2.1 sec (JSON_SET) (+176%)
```

**Conclusion** :
- **Dynamic Columns** : Meilleur pour **lecture intensive**, stockage compact
- **JSON** : Meilleur pour **écriture/modification**, structures complexes, portabilité

### Conversion Dynamic Columns ↔ JSON

```sql
-- Dynamic Columns → JSON (via procédure)
DELIMITER $$
CREATE FUNCTION dyncol_to_json(dyncol BLOB)
RETURNS TEXT
DETERMINISTIC
BEGIN
  DECLARE json_result TEXT DEFAULT '{}';
  DECLARE attr_list TEXT;
  DECLARE attr_name VARCHAR(255);
  DECLARE attr_value TEXT;
  DECLARE pos INT DEFAULT 1;
  DECLARE comma_pos INT;
  
  SET attr_list = COLUMN_LIST(dyncol);
  
  IF attr_list IS NULL OR attr_list = '' THEN
    RETURN '{}';
  END IF;
  
  SET json_result = '{';
  
  WHILE pos <= LENGTH(attr_list) DO
    SET comma_pos = LOCATE(',', attr_list, pos);
    
    IF comma_pos = 0 THEN
      SET attr_name = SUBSTRING(attr_list, pos);
      SET pos = LENGTH(attr_list) + 1;
    ELSE
      SET attr_name = SUBSTRING(attr_list, pos, comma_pos - pos);
      SET pos = comma_pos + 1;
    END IF;
    
    SET attr_value = COLUMN_GET(dyncol, attr_name AS CHAR);
    
    IF json_result != '{' THEN
      SET json_result = CONCAT(json_result, ',');
    END IF;
    
    SET json_result = CONCAT(
      json_result,
      '"', attr_name, '":"',
      IFNULL(attr_value, 'null'),
      '"'
    );
  END WHILE;
  
  RETURN CONCAT(json_result, '}');
END$$
DELIMITER ;

-- Test conversion
SELECT dyncol_to_json(
  COLUMN_CREATE('name', 'John', 'age', 30 AS INTEGER)
);
-- Résultat : {"name":"John","age":"30"}

-- JSON → Dynamic Columns (limité, sans imbrication)
-- Utiliser JSON_EXTRACT pour chaque clé connue
```

### Quand Choisir l'un ou l'autre ?

**Utiliser Dynamic Columns si** :
- ✅ Lecture intensive, peu de modifications
- ✅ Stockage compact critique (coûts cloud)
- ✅ Attributs simples (pas d'imbrication)
- ✅ Pas besoin de portabilité
- ✅ Schéma évolue rarement

**Utiliser JSON si** :
- ✅ Modifications fréquentes (JSON_SET rapide)
- ✅ Structures complexes (arrays, objets imbriqués)
- ✅ Index nécessaires (virtual columns + index)
- ✅ Portabilité entre BDD (PostgreSQL, MySQL)
- ✅ Lisibilité importante (debug, export)
- ✅ Intégration avec applications web (native JSON)

**Utiliser les deux (hybride)** :
```sql
-- Exemple : Métadonnées fréquentes (JSON) + Archive (Dynamic Columns)
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  
  -- Métadonnées actives (JSON, indexé)
  metadata JSON,
  
  -- Archive historique (Dynamic Columns, compact)
  historical_data BLOB,
  
  -- Index sur JSON
  meta_customer_id INT AS (metadata->>'$.customer_id') VIRTUAL,
  INDEX idx_customer (meta_customer_id)
);
```

---

## Performance et Optimisation

### Indexation Indirecte

**Problème** : Pas d'index direct sur dynamic columns.

**Solution** : Colonnes générées VIRTUAL + index.

```sql
-- Sans index : Scan complet
SELECT * FROM products
WHERE COLUMN_GET(category_attributes, 'brand' AS CHAR) = 'Samsung';
-- EXPLAIN : Full table scan, lent sur grandes tables

-- Avec colonne générée + index
ALTER TABLE products
ADD COLUMN brand_virtual VARCHAR(100) AS (
  COLUMN_GET(category_attributes, 'brand' AS CHAR)
) VIRTUAL,
ADD INDEX idx_brand (brand_virtual);

-- Maintenant indexé, rapide
SELECT * FROM products
WHERE brand_virtual = 'Samsung';
-- EXPLAIN : Index range scan

-- Colonnes générées multiples
ALTER TABLE products
ADD COLUMN warranty_virtual INT AS (
  COLUMN_GET(category_attributes, 'warranty_years' AS INTEGER)
) VIRTUAL,
ADD INDEX idx_warranty (warranty_virtual);

ALTER TABLE products
ADD COLUMN ram_virtual INT AS (
  COLUMN_GET(category_attributes, 'ram_gb' AS INTEGER)
) VIRTUAL,
ADD INDEX idx_ram (ram_virtual);

-- Requête combinée (utilise index)
SELECT * FROM products
WHERE category = 'Electronics'
  AND brand_virtual = 'Dell'
  AND ram_virtual >= 16;
```

### Éviter Requêtes Lourdes

```sql
-- ❌ Lent : COLUMN_GET dans SELECT pour toutes lignes
SELECT 
  product_id,
  COLUMN_GET(category_attributes, 'attr1' AS CHAR) AS attr1,
  COLUMN_GET(category_attributes, 'attr2' AS CHAR) AS attr2,
  COLUMN_GET(category_attributes, 'attr3' AS CHAR) AS attr3,
  -- ... 20 attributs
FROM products;
-- Parsing BLOB 20 fois par ligne = très lent

-- ✅ Rapide : Filtrer d'abord, puis extraire
SELECT 
  product_id,
  COLUMN_GET(category_attributes, 'brand' AS CHAR) AS brand,
  COLUMN_GET(category_attributes, 'warranty_years' AS INTEGER) AS warranty
FROM products
WHERE category = 'Electronics'  -- Index category
  AND price BETWEEN 500 AND 1500  -- Index price
LIMIT 100;
-- Moins de lignes à parser

-- ✅ Ou utiliser colonnes générées
SELECT 
  product_id,
  brand_virtual AS brand,
  warranty_virtual AS warranty
FROM products
WHERE category = 'Electronics'
  AND price BETWEEN 500 AND 1500;
```

### Batch Updates

```sql
-- ❌ Lent : UPDATE ligne par ligne
UPDATE products
SET category_attributes = COLUMN_ADD(
  category_attributes,
  'new_attr', 'value' AS CHAR
)
WHERE product_id = 1;
-- Répété 10000 fois = très lent

-- ✅ Rapide : Batch update
UPDATE products
SET category_attributes = COLUMN_ADD(
  category_attributes,
  'new_attr', 'default_value' AS CHAR
)
WHERE category = 'Electronics';
-- 10000 lignes en une requête
```

---

## Limitations et Pièges

### 1. Pas de Contraintes d'Intégrité

```sql
-- ❌ Impossible de définir contraintes sur dynamic columns
-- Pas de NOT NULL, UNIQUE, FOREIGN KEY, CHECK

-- Workaround : Validation application + triggers
DELIMITER $$
CREATE TRIGGER validate_product_attributes
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
  -- Validation personnalisée
  IF NEW.category = 'Electronics' THEN
    IF COLUMN_EXISTS(NEW.category_attributes, 'warranty_years') = 0 THEN
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Electronics must have warranty_years';
    END IF;
    
    IF COLUMN_GET(NEW.category_attributes, 'warranty_years' AS INTEGER) < 1 THEN
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Warranty must be at least 1 year';
    END IF;
  END IF;
END$$
DELIMITER ;
```

### 2. Typage Faible

```sql
-- ⚠️ Pas de vérification de type stricte
INSERT INTO products (name, category_attributes)
VALUES (
  'Test',
  COLUMN_CREATE('price', 'not a number')  -- Accepté !
);

-- Lecture échoue silencieusement
SELECT COLUMN_GET(category_attributes, 'price' AS DECIMAL) FROM products;
-- Résultat : NULL (au lieu d'erreur)

-- Solution : Validation explicite
SELECT 
  CASE 
    WHEN COLUMN_GET(category_attributes, 'price' AS DECIMAL) IS NULL
      AND COLUMN_EXISTS(category_attributes, 'price') = 1
    THEN 'INVALID TYPE'
    ELSE 'OK'
  END AS validation
FROM products;
```

### 3. Taille Maximale

```sql
-- BLOB limite : 64 KB (BLOB), 16 MB (MEDIUMBLOB), 4 GB (LONGBLOB)

-- Recommandation : < 1000 attributs, < 64 KB total
-- Si dépassé → Fragmenter en multiple colonnes dynamic

CREATE TABLE large_metadata (
  id INT PRIMARY KEY,
  
  -- Catégorie 1 d'attributs
  attributes_general BLOB,
  
  -- Catégorie 2 d'attributs
  attributes_technical BLOB,
  
  -- Catégorie 3 d'attributs
  attributes_marketing MEDIUMBLOB  -- Si vraiment beaucoup
);
```

### 4. Performance Bulk Operations

```sql
-- ⚠️ LOAD DATA INFILE complexe avec dynamic columns
-- Nécessite pre-processing

-- Workaround : INSERT ... SELECT avec COLUMN_CREATE
INSERT INTO products_dynamic (name, price, attributes)
SELECT 
  name,
  price,
  COLUMN_CREATE(
    'brand', brand,
    'category', category,
    'stock', stock AS INTEGER
  )
FROM products_staging;
```

---

## Migration et Évolution

### Migration Colonnes Fixes → Dynamic Columns

```sql
-- Schéma initial (colonnes fixes)
CREATE TABLE products_old (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  price DECIMAL(10,2),
  brand VARCHAR(100),
  warranty_years INT,
  screen_size DECIMAL(4,2),
  storage_gb INT
);

-- Nouvelle structure (dynamic columns)
CREATE TABLE products_new (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  price DECIMAL(10,2),
  attributes BLOB
);

-- Migration des données
INSERT INTO products_new (id, name, price, attributes)
SELECT 
  id,
  name,
  price,
  COLUMN_CREATE(
    'brand', brand,
    'warranty_years', warranty_years AS INTEGER,
    'screen_size', screen_size AS DECIMAL,
    'storage_gb', storage_gb AS INTEGER
  )
FROM products_old;

-- Vérifier migration
SELECT 
  id,
  name,
  COLUMN_GET(attributes, 'brand' AS CHAR) AS brand,
  COLUMN_GET(attributes, 'warranty_years' AS INTEGER) AS warranty
FROM products_new
LIMIT 10;

-- Swap tables
RENAME TABLE 
  products_old TO products_backup,
  products_new TO products_old;
```

### Évolution Progressive

```sql
-- Étape 1 : Ajouter colonne dynamic (garder anciennes)
ALTER TABLE products
ADD COLUMN new_attributes BLOB;

-- Étape 2 : Synchroniser données (background)
UPDATE products
SET new_attributes = COLUMN_CREATE(
  'brand', brand,
  'warranty_years', warranty_years AS INTEGER
)
WHERE new_attributes IS NULL
LIMIT 10000;
-- Répéter jusqu'à complète

-- Étape 3 : Applications utilisent new_attributes

-- Étape 4 : Supprimer anciennes colonnes (après validation)
ALTER TABLE products
DROP COLUMN brand,
DROP COLUMN warranty_years;
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **Dynamic columns** : Paires clé-valeur dans BLOB binaire
- ✅ **Flexibilité schéma** : Attributs différents par enregistrement
- ✅ **Format binaire** : Compact, rapide en lecture
- ✅ **Use case principal** : Données semi-structurées, multi-tenant

### Fonctions Essentielles
- ✅ `COLUMN_CREATE` : Créer dynamic column
- ✅ `COLUMN_GET` : Lire valeur (avec type)
- ✅ `COLUMN_ADD` : Ajouter/modifier attributs
- ✅ `COLUMN_DELETE` : Supprimer attributs
- ✅ `COLUMN_LIST` : Lister noms attributs
- ✅ `COLUMN_EXISTS` : Vérifier existence
- ✅ `COLUMN_CHECK` : Valider intégrité

### Types de Données
- ✅ Supportés : CHAR, INTEGER, DECIMAL, DATE, DATETIME, DOUBLE
- ✅ Booléens : INTEGER (0/1)
- ⚠️ Pas de structures imbriquées (utiliser JSON dans CHAR)

### Performance
- ✅ **Indexation** : Colonnes générées VIRTUAL + index
- ✅ **Lecture** : 30% plus rapide que JSON
- ⚠️ **Écriture** : Plus lent que JSON (re-créer BLOB)
- ✅ **Stockage** : 30-40% plus compact que JSON

### Dynamic Columns vs JSON
- **Dynamic Columns** : Compact, rapide lecture, MariaDB only
- **JSON** : Standard, modification rapide, structures complexes
- **Choix** : Dynamic pour lecture intensive, JSON pour écriture

### Limitations
- ❌ Pas d'index direct (utiliser colonnes générées)
- ❌ Pas de contraintes (NOT NULL, UNIQUE, FK)
- ❌ Typage faible (conversion silencieuse)
- ❌ Pas de structures imbriquées natives
- ⚠️ Validation au niveau application nécessaire

### Best Practices
- ✅ Utiliser pour attributs variables (< 30% communs)
- ✅ Créer colonnes générées pour attributs fréquemment filtrés
- ✅ Limiter à < 1000 attributs, < 64 KB par BLOB
- ✅ Valider données avec triggers
- ✅ Monitorer taille BLOB
- ✅ Documenter schéma logique (JSON schema)

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [Dynamic Columns](https://mariadb.com/kb/en/dynamic-columns/) - Guide complet
- 📖 [COLUMN_CREATE](https://mariadb.com/kb/en/column_create/)
- 📖 [COLUMN_GET](https://mariadb.com/kb/en/column_get/)
- 📖 [COLUMN Functions](https://mariadb.com/kb/en/dynamic-columns-functions/) - Toutes fonctions

### Comparaisons et Alternatives
- 🔄 [JSON vs Dynamic Columns](https://mariadb.com/kb/en/json-vs-dynamic-columns/)
- 🔄 [Dynamic Columns in MariaDB](https://mariadb.com/resources/blog/dynamic-columns-mariadb/)

### Cas d'Usage
- 📝 [E-commerce with Dynamic Columns](https://mariadb.com/resources/blog/ecommerce-dynamic-columns/)
- 📝 [Multi-Tenant Applications](https://mariadb.com/kb/en/multi-tenant-database-design/)

---

## ➡️ Section suivante

**[18.10 MariaDB Vector](./10-mariadb-vector.md)** : Découvrez le support natif des vecteurs pour l'IA et le machine learning, avec le type VECTOR, index HNSW, fonctions de distance, et intégration LLMs.

---


⏭️ [MariaDB Vector : Recherche vectorielle pour l'IA/RAG](/18-fonctionnalites-avancees/10-mariadb-vector.md)

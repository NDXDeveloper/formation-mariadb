🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.5 Invisible Columns

> **Niveau** : Avancé  
> **Durée estimée** : 1-1.5 heures  
> **Prérequis** : Chapitres 2-4, compréhension ALTER TABLE et évolution de schéma

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le concept de **colonnes invisibles** et leur utilité
- Créer des colonnes invisibles avec **INVISIBLE**
- Utiliser les colonnes invisibles pour **migrations sans downtime**
- Implémenter des **colonnes d'audit** transparentes
- Maîtriser **SET VISIBLE / SET INVISIBLE** pour évolutions progressives
- Combiner colonnes invisibles avec **colonnes générées**
- Concevoir des **stratégies de migration** en plusieurs phases
- Comprendre les **limitations** et comportements spécifiques

---

## Introduction

Les **colonnes invisibles** (invisible columns) sont des colonnes qui existent physiquement dans une table mais qui **ne sont pas retournées par SELECT *** sauf si elles sont explicitement mentionnées. Cette fonctionnalité permet d'ajouter des colonnes à une table existante **sans impacter les applications qui utilisent SELECT ***.

### Qu'est-ce qu'une Colonne Invisible ?

Une colonne invisible :
1. ✅ **Existe physiquement** dans la table (stockage normal)
2. ❌ **N'apparaît pas dans SELECT *** (sauf mention explicite)
3. ✅ **Apparaît dans DESCRIBE/SHOW COLUMNS** avec attribut INVISIBLE
4. ✅ **Accessible explicitement** : `SELECT invisible_col FROM table`
5. ✅ **Peut être indexée** normalement
6. ✅ **Peut être modifiée** entre VISIBLE et INVISIBLE à tout moment

**Métaphore** : Une colonne invisible est comme un **champ masqué dans un formulaire HTML** - elle existe et peut être utilisée, mais n'est pas affichée par défaut.

### Pourquoi Utiliser des Colonnes Invisibles ?

**Problématiques résolues** :

1. **🔄 Migration Progressive sans Downtime**
   - Ajouter colonnes sans casser applications existantes utilisant SELECT *
   - Déploiement progressif sur plusieurs versions
   - Rollback facile (SET INVISIBLE si problème)

2. **📊 Colonnes d'Audit Transparentes**
   - Timestamps de création/modification
   - Informations de traçabilité (user_id, IP, session)
   - Métadonnées système non exposées aux utilisateurs

3. **🔧 Colonnes Techniques et Métadonnées**
   - Colonnes internes (hash, checksums, versions)
   - Données de réplication ou partitionnement
   - Colonnes de debug/diagnostique

4. **📈 Dénormalisation Future-Proof**
   - Préparer colonnes pour futures fonctionnalités
   - Tester en production sans impact
   - Migration graduelle vers nouveau schéma

5. **🛡️ Compatibilité Descendante**
   - Maintenir compatibilité avec anciennes versions d'applications
   - Éviter regression lors de montée de version
   - Dépréciation progressive de colonnes obsolètes

**Scénario classique - Le problème** :
```sql
-- Application legacy utilise SELECT *
SELECT * FROM users;  -- Retourne : id, username, email

-- Ajout d'une nouvelle colonne
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Maintenant SELECT * retourne : id, username, email, phone
-- ❌ Application legacy casse si elle parse résultat par position !
-- ❌ Ou consomme plus de bande passante inutilement
```

**Solution avec colonnes invisibles** :
```sql
-- Ajouter colonne en invisible
ALTER TABLE users ADD COLUMN phone VARCHAR(20) INVISIBLE;

-- SELECT * continue de retourner : id, username, email
-- ✅ Application legacy fonctionne sans modification

-- Nouvelle version application accède explicitement
SELECT id, username, email, phone FROM users;
-- ✅ Nouvelle application voit la colonne phone
```

---

## Syntaxe et Utilisation

### Créer une Colonne Invisible

```sql
-- Méthode 1 : Dans CREATE TABLE
CREATE TABLE users (
  user_id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50),
  email VARCHAR(255),
  
  -- Colonnes invisibles
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP INVISIBLE,
  created_by VARCHAR(50) INVISIBLE,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP INVISIBLE
);

-- Méthode 2 : Ajouter avec ALTER TABLE
ALTER TABLE users 
  ADD COLUMN phone VARCHAR(20) INVISIBLE;

-- Méthode 3 : Ajouter après une colonne spécifique
ALTER TABLE users 
  ADD COLUMN last_login TIMESTAMP INVISIBLE AFTER email;
```

### Rendre Visible/Invisible

```sql
-- Rendre une colonne invisible
ALTER TABLE users 
  ALTER COLUMN phone SET INVISIBLE;

-- Rendre une colonne visible
ALTER TABLE users 
  ALTER COLUMN phone SET VISIBLE;

-- Basculement instantané (metadata only, très rapide)
```

### Accès aux Colonnes Invisibles

```sql
-- SELECT * : Ne retourne PAS les colonnes invisibles
SELECT * FROM users;
-- Résultat : user_id, username, email

-- SELECT explicite : Retourne la colonne invisible
SELECT user_id, username, email, created_at FROM users;
-- Résultat : user_id, username, email, created_at

-- SELECT avec toutes colonnes (y compris invisibles)
SELECT user_id, username, email, created_at, created_by, updated_at FROM users;

-- COUNT(*) : Compte toutes lignes, ignorant visibilité colonnes
SELECT COUNT(*) FROM users;  -- Fonctionne normalement
```

### Inspection des Colonnes

```sql
-- DESCRIBE : Montre toutes colonnes avec indicateur INVISIBLE
DESCRIBE users;
-- Field      | Type         | Null | Key | Default | Extra | Invisible
-- user_id    | int(11)      | NO   | PRI | NULL    | auto  | NO
-- username   | varchar(50)  | YES  |     | NULL    |       | NO
-- email      | varchar(255) | YES  |     | NULL    |       | NO
-- created_at | timestamp    | NO   |     | CURRENT | on up | YES
-- created_by | varchar(50)  | YES  |     | NULL    |       | YES

-- SHOW COLUMNS : Équivalent à DESCRIBE
SHOW COLUMNS FROM users;

-- SHOW CREATE TABLE : Montre structure complète
SHOW CREATE TABLE users;
-- CREATE TABLE `users` (
--   `user_id` int(11) NOT NULL AUTO_INCREMENT,
--   `username` varchar(50) DEFAULT NULL,
--   `email` varchar(255) DEFAULT NULL,
--   `created_at` timestamp INVISIBLE NOT NULL DEFAULT current_timestamp(),
--   ...
```

---

## Comportements et Règles

### INSERT : Colonnes Invisibles Ignorées par Défaut

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  price DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP INVISIBLE
);

-- INSERT sans spécifier colonnes : Ignore invisible
INSERT INTO products VALUES (1, 'Widget', 29.99);
-- ✅ OK : created_at prend DEFAULT (CURRENT_TIMESTAMP)

-- INSERT explicite : Peut inclure invisible
INSERT INTO products (product_id, name, price, created_at)
VALUES (2, 'Gadget', 49.99, '2025-01-01 10:00:00');
-- ✅ OK : created_at = '2025-01-01 10:00:00'

-- INSERT ... SELECT * : Ignore invisible (source et destination)
INSERT INTO products_backup SELECT * FROM products;
-- Copie uniquement colonnes visibles
```

### UPDATE : Colonnes Invisibles Modifiables

```sql
-- UPDATE fonctionne normalement sur colonnes invisibles
UPDATE users 
SET created_by = 'admin' 
WHERE user_id = 1;
-- ✅ OK

-- Triggers peuvent accéder colonnes invisibles
CREATE TRIGGER audit_update
BEFORE UPDATE ON users
FOR EACH ROW
SET NEW.updated_at = NOW();
-- ✅ OK même si updated_at INVISIBLE
```

### DELETE : Pas d'impact de la Visibilité

```sql
-- DELETE fonctionne normalement
DELETE FROM users WHERE user_id = 1;
-- Visibilité colonnes n'a aucun impact
```

---

## Cas d'Usage Détaillés

### 1. Migration Progressive sans Downtime

**Scénario** : Ajouter colonne `phone` à table `users` avec 10M lignes, utilisée par 5 applications différentes.

**Phase 1 : Ajouter colonne invisible**
```sql
-- Étape 1 : Ajouter colonne en invisible (rapide, metadata only)
ALTER TABLE users 
  ADD COLUMN phone VARCHAR(20) INVISIBLE;

-- Applications legacy continuent de fonctionner
-- SELECT * FROM users; → retourne colonnes originales
```

**Phase 2 : Créer index (sans impact applications)**
```sql
-- Étape 2 : Indexer la nouvelle colonne
CREATE INDEX idx_phone ON users(phone);

-- Applications toujours non impactées
```

**Phase 3 : Déployer nouvelle version application**
```sql
-- Nouvelle version application utilise explicitement phone
-- Code application v2.0 :
-- SELECT id, username, email, phone FROM users WHERE phone = '+33123456789';

-- Anciennes versions (v1.x) continuent avec SELECT *
-- Coexistence des versions sans problème
```

**Phase 4 : Peupler données progressivement**
```sql
-- Batch de mise à jour en arrière-plan
UPDATE users 
SET phone = normalize_phone(legacy_contact_field)
WHERE phone IS NULL 
LIMIT 10000;

-- Applications fonctionnent pendant le remplissage
```

**Phase 5 : Rendre visible (une fois toutes apps migrées)**
```sql
-- Une fois toutes applications en v2.0+
ALTER TABLE users 
  ALTER COLUMN phone SET VISIBLE;

-- Désormais SELECT * inclut phone
-- Toutes applications supportent déjà la colonne → pas de problème
```

**Phase 6 (optionnelle) : Rendre obligatoire**
```sql
-- Si besoin, rendre NOT NULL après validation
ALTER TABLE users 
  MODIFY COLUMN phone VARCHAR(20) NOT NULL;
```

### 2. Colonnes d'Audit Transparentes

**Besoin** : Tracer toutes modifications sans polluer données métier.

```sql
CREATE TABLE orders (
  -- Colonnes métier (visibles)
  order_id INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT,
  total_amount DECIMAL(10,2),
  status ENUM('PENDING','PAID','SHIPPED','DELIVERED'),
  
  -- Colonnes d'audit (invisibles)
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP INVISIBLE,
  created_by VARCHAR(50) INVISIBLE,
  created_ip VARCHAR(45) INVISIBLE,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP INVISIBLE,
  updated_by VARCHAR(50) INVISIBLE,
  updated_ip VARCHAR(45) INVISIBLE,
  row_version INT DEFAULT 1 INVISIBLE,
  
  -- Index sur colonnes d'audit pour requêtes forensiques
  INDEX idx_created_at (created_at),
  INDEX idx_created_by (created_by)
);

-- Application métier : SELECT * retourne uniquement données business
SELECT * FROM orders WHERE customer_id = 123;
-- Résultat : order_id, customer_id, total_amount, status

-- Équipe audit : Accès complet pour investigation
SELECT 
  order_id,
  customer_id,
  total_amount,
  status,
  created_at,
  created_by,
  created_ip,
  updated_at,
  updated_by,
  row_version
FROM orders
WHERE created_at >= '2025-01-01'
ORDER BY created_at DESC;

-- Trigger pour peupler automatiquement colonnes audit
DELIMITER $$
CREATE TRIGGER orders_audit_insert
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
  SET NEW.created_by = COALESCE(NEW.created_by, USER());
  SET NEW.created_ip = COALESCE(NEW.created_ip, @client_ip);
END$$

CREATE TRIGGER orders_audit_update
BEFORE UPDATE ON orders
FOR EACH ROW
BEGIN
  SET NEW.updated_by = USER();
  SET NEW.updated_ip = @client_ip;
  SET NEW.row_version = OLD.row_version + 1;
END$$
DELIMITER ;

-- Application définit @client_ip avant INSERT/UPDATE
SET @client_ip = '192.168.1.100';
INSERT INTO orders (customer_id, total_amount, status)
VALUES (123, 599.99, 'PENDING');
```

### 3. Dénormalisation Préparée

**Scénario** : Préparer optimisation future sans impacter prod actuelle.

```sql
-- Table actuelle avec jointures lourdes
CREATE TABLE order_items (
  item_id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT,
  product_id INT,
  quantity INT,
  unit_price DECIMAL(10,2),
  
  -- Colonnes dénormalisées futures (invisibles pour l'instant)
  product_name VARCHAR(100) INVISIBLE,
  product_category VARCHAR(50) INVISIBLE,
  customer_name VARCHAR(100) INVISIBLE,
  
  FOREIGN KEY (order_id) REFERENCES orders(order_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Phase 1 : Peupler progressivement en batch arrière-plan
UPDATE order_items oi
INNER JOIN products p ON oi.product_id = p.product_id
INNER JOIN orders o ON oi.order_id = o.order_id
INNER JOIN customers c ON o.customer_id = c.customer_id
SET 
  oi.product_name = p.name,
  oi.product_category = p.category,
  oi.customer_name = c.name
WHERE oi.product_name IS NULL
LIMIT 10000;

-- Phase 2 : Trigger pour maintenir cohérence sur nouvelles lignes
DELIMITER $$
CREATE TRIGGER order_items_denormalize
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
  SELECT p.name, p.category INTO NEW.product_name, NEW.product_category
  FROM products p WHERE p.product_id = NEW.product_id;
  
  SELECT c.name INTO NEW.customer_name
  FROM orders o
  INNER JOIN customers c ON o.customer_id = c.customer_id
  WHERE o.order_id = NEW.order_id;
END$$
DELIMITER ;

-- Phase 3 : Benchmarker performance
-- Requête ancienne (avec jointures)
SELECT 
  oi.item_id,
  p.name AS product_name,
  p.category,
  c.name AS customer_name,
  oi.quantity
FROM order_items oi
INNER JOIN products p ON oi.product_id = p.product_id
INNER JOIN orders o ON oi.order_id = o.order_id
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE oi.order_id = 12345;
-- Temps : ~50ms

-- Requête nouvelle (sans jointures, colonnes invisibles)
SELECT 
  item_id,
  product_name,
  product_category,
  customer_name,
  quantity
FROM order_items
WHERE order_id = 12345;
-- Temps : ~5ms (10x plus rapide)

-- Phase 4 : Si performance validée → Rendre visible
ALTER TABLE order_items 
  ALTER COLUMN product_name SET VISIBLE,
  ALTER COLUMN product_category SET VISIBLE,
  ALTER COLUMN customer_name SET VISIBLE;

-- Phase 5 : Migrer application pour utiliser colonnes dénormalisées
```

### 4. Colonnes de Debug et Diagnostique

**Besoin** : Colonnes temporaires pour diagnostique sans polluer schéma production.

```sql
CREATE TABLE api_requests (
  request_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  endpoint VARCHAR(255),
  method ENUM('GET','POST','PUT','DELETE'),
  response_time_ms INT,
  status_code INT,
  
  -- Colonnes de debug (invisibles en prod, visibles en dev/staging)
  request_headers JSON INVISIBLE,
  request_body TEXT INVISIBLE,
  response_body TEXT INVISIBLE,
  stack_trace TEXT INVISIBLE,
  debug_info JSON INVISIBLE
);

-- En production : Debug désactivé
-- SELECT * ne retourne pas données volumineuses (headers, bodies)

-- En cas d'incident : Activer temporairement
ALTER TABLE api_requests 
  ALTER COLUMN request_headers SET VISIBLE,
  ALTER COLUMN response_body SET VISIBLE,
  ALTER COLUMN stack_trace SET VISIBLE;

-- Debugging
SELECT 
  request_id,
  endpoint,
  status_code,
  request_headers,
  response_body,
  stack_trace
FROM api_requests
WHERE status_code >= 500
  AND created_at >= NOW() - INTERVAL 1 HOUR;

-- Après résolution : Remettre invisible
ALTER TABLE api_requests 
  ALTER COLUMN request_headers SET INVISIBLE,
  ALTER COLUMN response_body SET INVISIBLE,
  ALTER COLUMN stack_trace SET INVISIBLE;
```

### 5. Migration de Schéma avec Colonnes Obsolètes

**Scénario** : Déprécier colonne `old_format` vers `new_format`.

```sql
CREATE TABLE documents (
  doc_id INT PRIMARY KEY,
  title VARCHAR(255),
  
  -- Ancienne colonne (à déprécier)
  old_format TEXT,
  
  -- Nouvelle colonne (invisible initialement)
  new_format JSON INVISIBLE
);

-- Stratégie de migration en 4 phases

-- Phase 1 : Ajouter nouvelle colonne invisible + trigger sync
DELIMITER $$
CREATE TRIGGER documents_sync_format
BEFORE INSERT ON documents
FOR EACH ROW
BEGIN
  -- Convertir old_format → new_format automatiquement
  IF NEW.old_format IS NOT NULL THEN
    SET NEW.new_format = JSON_OBJECT('content', NEW.old_format, 'version', 2);
  END IF;
END$$

CREATE TRIGGER documents_sync_format_update
BEFORE UPDATE ON documents
FOR EACH ROW
BEGIN
  IF NEW.old_format != OLD.old_format THEN
    SET NEW.new_format = JSON_OBJECT('content', NEW.old_format, 'version', 2);
  END IF;
END$$
DELIMITER ;

-- Phase 2 : Migrer données existantes en arrière-plan
UPDATE documents
SET new_format = JSON_OBJECT('content', old_format, 'version', 2)
WHERE new_format IS NULL
LIMIT 10000;

-- Phase 3 : Déployer nouvelle version app utilisant new_format
-- Code app v2.0 : SELECT doc_id, title, new_format FROM documents

-- Phase 4 : Basculer visibilité (une fois migration complète)
ALTER TABLE documents 
  ALTER COLUMN new_format SET VISIBLE,
  ALTER COLUMN old_format SET INVISIBLE;

-- Phase 5 : Supprimer old_format (après période de grâce)
ALTER TABLE documents DROP COLUMN old_format;
```

---

## Combinaison avec Autres Fonctionnalités

### Colonnes Générées Invisibles

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  price DECIMAL(10,2),
  tax_rate DECIMAL(4,2),
  
  -- Colonne générée invisible
  price_with_tax DECIMAL(10,2) AS (price * (1 + tax_rate)) STORED INVISIBLE,
  
  -- Index sur colonne générée invisible
  INDEX idx_price_tax (price_with_tax)
);

-- SELECT * : Ne retourne pas price_with_tax
SELECT * FROM products;
-- Résultat : product_id, name, price, tax_rate

-- Requête optimisée utilisant index invisible
SELECT product_id, name, price, price_with_tax
FROM products
WHERE price_with_tax > 100
ORDER BY price_with_tax;
-- Utilise idx_price_tax même si colonne INVISIBLE
```

**Cas d'usage** : Préparer index sur valeur dérivée pour migration future.

### System-Versioned avec Colonnes Invisibles

```sql
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  name VARCHAR(100),
  salary DECIMAL(10,2),
  department VARCHAR(50),
  
  -- Colonnes de versioning invisibles (en plus row_start/row_end automatiques)
  last_modified_by VARCHAR(50) INVISIBLE,
  change_reason VARCHAR(255) INVISIBLE
  
) WITH SYSTEM VERSIONING;

-- SELECT * : Ne retourne que colonnes métier
-- row_start, row_end déjà cachées par système
-- last_modified_by, change_reason aussi invisibles

-- Audit complet
SELECT 
  employee_id,
  name,
  salary,
  department,
  row_start,
  row_end,
  last_modified_by,
  change_reason
FROM employees
FOR SYSTEM_TIME ALL
WHERE employee_id = 123
ORDER BY row_start;
```

### Application Time Period avec Colonnes Invisibles

```sql
CREATE TABLE contracts (
  contract_id INT PRIMARY KEY,
  customer_id INT,
  
  -- Période applicative (visible)
  valid_from DATE,
  valid_until DATE,
  
  amount DECIMAL(10,2),
  
  -- Métadonnées invisibles
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP INVISIBLE,
  approved_by VARCHAR(50) INVISIBLE,
  approval_date DATE INVISIBLE,
  
  PERIOD FOR validity_period (valid_from, valid_until),
  UNIQUE (customer_id, validity_period WITHOUT OVERLAPS)
);

-- Requête métier : Colonnes business seulement
SELECT * FROM contracts WHERE customer_id = 123;

-- Audit : Colonnes métadonnées incluses
SELECT 
  contract_id,
  customer_id,
  valid_from,
  valid_until,
  amount,
  approved_by,
  approval_date
FROM contracts
WHERE created_at >= '2025-01-01';
```

---

## Performance et Considérations

### Impact sur Performances

```sql
-- Colonnes invisibles : AUCUN impact performance
-- Stockage identique, indexation identique, requêtes identiques

-- Benchmark : Table avec colonnes invisibles
CREATE TABLE perf_test (
  id INT PRIMARY KEY,
  col1 VARCHAR(100),
  col2 VARCHAR(100),
  col3 VARCHAR(100) INVISIBLE,
  col4 VARCHAR(100) INVISIBLE,
  col5 VARCHAR(100) INVISIBLE
);

-- INSERT 1M rows
-- Temps : Identique avec ou sans INVISIBLE
-- L'attribut INVISIBLE est purement metadata
```

**Impact** :
- ✅ **SELECT *** : Légèrement plus rapide (moins de colonnes transférées)
- ✅ **INSERT/UPDATE** : Aucune différence
- ✅ **Index** : Fonctionne normalement sur colonnes invisibles
- ✅ **Stockage** : Identique (INVISIBLE n'économise pas d'espace)

### Espace Disque

```sql
-- INVISIBLE ne réduit PAS l'espace disque
-- Pour économie espace, utiliser colonnes VIRTUAL (section 18.4)

-- Colonne INVISIBLE : Stockage normal
CREATE TABLE test1 (
  id INT,
  data TEXT INVISIBLE
);
-- data est stockée normalement sur disque

-- Colonne VIRTUAL : Pas de stockage (sauf index)
CREATE TABLE test2 (
  id INT,
  source TEXT,
  computed VARCHAR(100) AS (UPPER(source)) VIRTUAL
);
-- computed n'occupe aucun espace disque
```

---

## Best Practices

### 1. Utiliser pour Migrations Progressives

```sql
-- ✅ Pattern recommandé : INVISIBLE → populate → VISIBLE
-- Étape 1 : Ajouter invisible
ALTER TABLE users ADD COLUMN new_col VARCHAR(100) INVISIBLE;

-- Étape 2 : Peupler
UPDATE users SET new_col = transform(old_col);

-- Étape 3 : Indexer
CREATE INDEX idx_new ON users(new_col);

-- Étape 4 : Déployer app
-- Code app accède explicitement new_col

-- Étape 5 : Rendre visible
ALTER TABLE users ALTER COLUMN new_col SET VISIBLE;

-- Étape 6 : Supprimer old_col
ALTER TABLE users DROP COLUMN old_col;
```

### 2. Documenter l'Intention

```sql
-- ✅ Commenter pourquoi colonne est invisible
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  -- ... colonnes business ...
  
  -- INVISIBLE : Audit uniquement, pas exposé aux apps métier
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP INVISIBLE,
  created_by VARCHAR(50) INVISIBLE
) COMMENT='Colonnes audit invisibles pour ne pas impacter SELECT * legacy';
```

### 3. Combiner avec Colonnes Générées

```sql
-- ✅ Pattern : Colonne générée INVISIBLE pour préparer index futur
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  description TEXT,
  
  -- Préparer index full-text pour future recherche
  searchable TEXT AS (CONCAT(name, ' ', description)) STORED INVISIBLE,
  FULLTEXT INDEX ft_search (searchable)
);

-- Actuellement : App n'utilise pas recherche full-text
-- Futur : Rendre visible quand feature recherche déployée
```

### 4. Stratégie de Rollback

```sql
-- ✅ Rollback instantané en cas de problème
-- Migration en cours, problème détecté

-- Rollback : Remettre invisible
ALTER TABLE users ALTER COLUMN new_col SET INVISIBLE;
-- Instantané (metadata only), app legacy continue de fonctionner

-- Analyser problème, corriger, puis re-déployer
```

### 5. Éviter Abus

```sql
-- ❌ Mauvais : Trop de colonnes invisibles
CREATE TABLE bad_table (
  id INT,
  col1 VARCHAR(100),
  col2 VARCHAR(100),
  col3 VARCHAR(100) INVISIBLE,
  col4 VARCHAR(100) INVISIBLE,
  col5 VARCHAR(100) INVISIBLE,
  col6 VARCHAR(100) INVISIBLE,
  col7 VARCHAR(100) INVISIBLE,
  col8 VARCHAR(100) INVISIBLE
  -- ... 20 colonnes invisibles
);
-- Confusion : Quelle est la vraie structure ?

-- ✅ Bon : INVISIBLE pour transition temporaire uniquement
-- Pas pour cacher définitivement des données
```

---

## Limitations et Pièges

### Limitations Techniques

❌ **PRIMARY KEY ne peut être invisible** :
```sql
-- INTERDIT
CREATE TABLE test (
  id INT PRIMARY KEY INVISIBLE  -- ERROR
);
-- PRIMARY KEY doit toujours être visible
```

❌ **Pas de colonne invisible uniquement** :
```sql
-- INTERDIT : Au moins une colonne visible requise
CREATE TABLE test (
  col1 INT INVISIBLE,
  col2 VARCHAR(100) INVISIBLE
);
-- ERROR: Table must have at least one visible column
```

✅ **AUTO_INCREMENT peut être invisible** :
```sql
-- OK : AUTO_INCREMENT invisible possible (rare)
CREATE TABLE logs (
  log_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Visible
  message TEXT,
  internal_id BIGINT AUTO_INCREMENT INVISIBLE  -- OK mais inhabituel
);
```

### Pièges Courants

⚠️ **INSERT avec valeurs positionnelles** :
```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255),
  phone VARCHAR(20) INVISIBLE
);

-- ⚠️ Piège : INSERT positionnel ignore invisible
INSERT INTO users VALUES (1, 'Alice', 'alice@example.com', '+33123');
-- ERROR: Column count doesn't match value count

-- ✅ Solution : Spécifier colonnes
INSERT INTO users (id, name, email, phone)
VALUES (1, 'Alice', 'alice@example.com', '+33123');
```

⚠️ **CREATE TABLE ... SELECT *** :
```sql
-- Source avec colonnes invisibles
CREATE TABLE source (
  id INT,
  name VARCHAR(100),
  hidden TEXT INVISIBLE
);

-- CREATE TABLE ... SELECT * : Ignore invisibles
CREATE TABLE destination AS SELECT * FROM source;
-- destination n'aura pas la colonne 'hidden'

-- ✅ Solution : Spécifier explicitement
CREATE TABLE destination AS 
SELECT id, name, hidden FROM source;
```

⚠️ **ALTER TABLE ... CHANGE avec INVISIBLE** :
```sql
-- Comportement : CHANGE ne préserve pas INVISIBLE
ALTER TABLE users CHANGE COLUMN phone mobile VARCHAR(25);
-- mobile sera VISIBLE même si phone était INVISIBLE

-- ✅ Solution : Réappliquer INVISIBLE
ALTER TABLE users 
  CHANGE COLUMN phone mobile VARCHAR(25),
  ALTER COLUMN mobile SET INVISIBLE;
```

---

## Comparaison avec Alternatives

### INVISIBLE vs Vues

| Aspect | INVISIBLE Columns | Vues |
|--------|------------------|------|
| **Stockage** | Physique (table) | Virtuel (query) |
| **Performance écriture** | Normale | Dépend (updatable view) |
| **Performance lecture** | Rapide | Variable (MERGE vs TEMPTABLE) |
| **Flexibilité** | Modérée | Élevée |
| **Complexité** | Faible | Moyenne |
| **Index** | Directs | Sur table sous-jacente |
| **Cas d'usage** | Migration colonne | Masquage données métier |

**Quand utiliser INVISIBLE** :
- Migration progressive d'applications
- Colonnes d'audit/métadonnées
- Évolution de schéma sans downtime

**Quand utiliser Vues** :
- Masquage complexe de données sensibles
- Jointures ou transformations
- Différents niveaux d'accès (row-level security)

### INVISIBLE vs Permissions

```sql
-- Alternative 1 : INVISIBLE
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  username VARCHAR(50),
  password_hash VARCHAR(255) INVISIBLE
);
-- SELECT * ne retourne pas password_hash

-- Alternative 2 : Permissions + Vue
CREATE TABLE users_internal (
  user_id INT PRIMARY KEY,
  username VARCHAR(50),
  password_hash VARCHAR(255)
);

CREATE VIEW users AS
SELECT user_id, username FROM users_internal;

GRANT SELECT ON users TO 'app_user'@'%';
-- app_user ne voit que user_id, username
```

**INVISIBLE** : Plus simple pour masquer quelques colonnes  
**Vues + Permissions** : Meilleur contrôle d'accès granulaire

---

## Monitoring et Administration

### Détecter Colonnes Invisibles

```sql
-- Lister toutes colonnes invisibles d'une base
SELECT 
  TABLE_NAME,
  COLUMN_NAME,
  COLUMN_TYPE,
  IS_NULLABLE,
  COLUMN_DEFAULT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND EXTRA LIKE '%INVISIBLE%'
ORDER BY TABLE_NAME, ORDINAL_POSITION;

-- Compter colonnes invisibles par table
SELECT 
  TABLE_NAME,
  COUNT(*) AS invisible_columns_count
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND EXTRA LIKE '%INVISIBLE%'
GROUP BY TABLE_NAME
ORDER BY invisible_columns_count DESC;
```

### Script de Migration Automatique

```sql
-- Procédure pour rendre toutes colonnes d'audit invisibles
DELIMITER $$
CREATE PROCEDURE make_audit_columns_invisible()
BEGIN
  DECLARE done INT DEFAULT FALSE;
  DECLARE tbl VARCHAR(64);
  DECLARE col VARCHAR(64);
  DECLARE cur CURSOR FOR 
    SELECT TABLE_NAME, COLUMN_NAME
    FROM information_schema.COLUMNS
    WHERE TABLE_SCHEMA = DATABASE()
      AND COLUMN_NAME IN ('created_at', 'created_by', 'updated_at', 'updated_by')
      AND EXTRA NOT LIKE '%INVISIBLE%';
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
  
  OPEN cur;
  
  read_loop: LOOP
    FETCH cur INTO tbl, col;
    IF done THEN
      LEAVE read_loop;
    END IF;
    
    SET @sql = CONCAT('ALTER TABLE ', tbl, ' ALTER COLUMN ', col, ' SET INVISIBLE');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SELECT CONCAT('Made invisible: ', tbl, '.', col) AS status;
  END LOOP;
  
  CLOSE cur;
END$$
DELIMITER ;

-- Exécution
CALL make_audit_columns_invisible();
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **Colonnes invisibles** : Existent physiquement mais absentes de SELECT *
- ✅ **INVISIBLE** : Attribut de colonne, basculable avec SET VISIBLE/INVISIBLE
- ✅ **Accès explicite** : Toujours accessible si mentionnée dans SELECT
- ✅ **Performance** : Aucun overhead, purement metadata

### Syntaxe
- ✅ `CREATE TABLE ... col_name type INVISIBLE`
- ✅ `ALTER TABLE ... ADD COLUMN col_name type INVISIBLE`
- ✅ `ALTER TABLE ... ALTER COLUMN col_name SET INVISIBLE`
- ✅ `ALTER TABLE ... ALTER COLUMN col_name SET VISIBLE`

### Cas d'Usage Principaux
- ✅ **Migration progressive** : Ajouter colonnes sans casser SELECT * legacy
- ✅ **Colonnes d'audit** : Timestamps, user_id, IP transparents pour apps métier
- ✅ **Dénormalisation préparée** : Tester optimisations sans impact
- ✅ **Debug/diagnostique** : Colonnes temporaires activables à la demande
- ✅ **Dépréciation** : Transition old_col → new_col progressive

### Stratégie de Migration
1. Ajouter colonne INVISIBLE
2. Indexer si nécessaire
3. Peupler données (batch arrière-plan)
4. Déployer nouvelle version app
5. Rendre VISIBLE une fois toutes apps migrées
6. Supprimer ancienne colonne

### Best Practices
- ✅ Utiliser pour migrations temporaires, pas masquage permanent
- ✅ Documenter intention (commentaires)
- ✅ Combiner avec colonnes générées pour optimisations futures
- ✅ Rollback instantané possible (SET INVISIBLE)
- ✅ Éviter trop de colonnes invisibles (confusion)

### Limitations
- ❌ PRIMARY KEY ne peut être invisible
- ❌ Au moins une colonne visible requise par table
- ⚠️ INSERT positionnel ignore colonnes invisibles
- ⚠️ CREATE TABLE ... SELECT * ignore invisibles
- ⚠️ ALTER CHANGE ne préserve pas INVISIBLE

### Combinaisons
- ✅ **+ Colonnes générées** : Index futurs sans impact actuel
- ✅ **+ System-Versioned** : Métadonnées audit en plus row_start/row_end
- ✅ **+ Application Time** : Informations approbation/validation cachées

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [Invisible Columns](https://mariadb.com/kb/en/invisible-columns/) - Guide complet
- 📖 [ALTER TABLE - Column Visibility](https://mariadb.com/kb/en/alter-table/#column-visibility) - Syntaxe
- 📖 [INFORMATION_SCHEMA.COLUMNS](https://mariadb.com/kb/en/information-schema-columns-table/) - Métadonnées

### Articles et Best Practices
- 📝 [Zero-Downtime Schema Migrations](https://mariadb.com/resources/blog/zero-downtime-migrations/)
- 📝 [Invisible Columns for Backward Compatibility](https://mariadb.com/kb/en/invisible-columns-backward-compatibility/)
- 📝 [Progressive Database Migrations](https://mariadb.com/resources/blog/progressive-migrations/)

### Outils de Migration
- 🛠️ [gh-ost](https://github.com/github/gh-ost) - Online schema migrations
- 🛠️ [pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/) - Percona Toolkit
- 🛠️ [Flyway](https://flywaydb.org/) - Version control pour databases

### Comparaison avec Autres SGBD
- 🔄 [MySQL Invisible Columns](https://dev.mysql.com/doc/refman/8.0/en/invisible-columns.html) - Depuis MySQL 8.0.23
- 🔄 [PostgreSQL Generated Columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html) - Pas d'équivalent INVISIBLE

---

## ➡️ Section suivante

**[18.6 Compression de Tables](./06-compression-tables.md)** : Découvrez comment réduire l'espace disque et optimiser les I/O avec ROW_FORMAT=COMPRESSED et PAGE_COMPRESSED, avec benchmarks et recommandations par cas d'usage.

---


⏭️ [Compression de tables](/18-fonctionnalites-avancees/06-compression-tables.md)

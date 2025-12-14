🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.4 Audit de Schéma

> **Type** : Checklist d'audit design base de données  
> **Focus** : Structure tables, types de données, normalisation, partitionnement  
> **Durée** : 1-3 heures  
> **Prérequis** : Accès schéma, connaissance métier

---

## 🎯 Objectifs de cet audit

Vérifier que le **design du schéma** est optimal pour :
- ✅ Types de données appropriés (espace, performance)
- ✅ Normalisation adaptée au cas d'usage
- ✅ Clés primaires et contraintes efficaces
- ✅ Partitionnement pour tables volumineuses
- ✅ Stratégie d'archivage et rétention
- ✅ Évolutivité et maintenance

**Résultat attendu :** Plan de refactoring schéma avec migrations prioritaires et impact estimé.

---

## 📋 Checklist complète (16 points)

### Catégorie 1 : Types de Données

#### ✅ 1.1 Taille des entiers (INT vs BIGINT)

**🔍 Diagnostic**

```sql
-- Identifier colonnes INT sous-utilisées ou surchargées
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    CASE DATA_TYPE
        WHEN 'tinyint' THEN 127
        WHEN 'smallint' THEN 32767
        WHEN 'mediumint' THEN 8388607
        WHEN 'int' THEN 2147483647
        WHEN 'bigint' THEN 9223372036854775807
    END AS max_value,
    (SELECT MAX(COLUMN_NAME) FROM TABLE_NAME) AS current_max,
    ROUND(100.0 * (SELECT MAX(COLUMN_NAME) FROM TABLE_NAME) / 
          CASE DATA_TYPE
              WHEN 'int' THEN 2147483647
              WHEN 'bigint' THEN 9223372036854775807
          END, 2) AS usage_pct
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE IN ('tinyint', 'smallint', 'mediumint', 'int', 'bigint')
  AND COLUMN_NAME LIKE '%id%'
ORDER BY TABLE_NAME, COLUMN_NAME;
```

**Tailles des types entiers :**

| Type | Bytes | Range (UNSIGNED) | Use case |
|------|-------|------------------|----------|
| **TINYINT** | 1 | 0 - 255 | Statuts, flags, âges |
| **SMALLINT** | 2 | 0 - 65,535 | IDs tables petites |
| **MEDIUMINT** | 3 | 0 - 16,777,215 | IDs tables moyennes |
| **INT** | 4 | 0 - 4,294,967,295 | IDs standard (~4B) |
| **BIGINT** | 8 | 0 - 18,446,744,073,709,551,615 | Twitter-scale |

**⚠️ Seuils critiques**

| Situation | Problème | Action |
|-----------|----------|--------|
| **BIGINT avec max < 1M** | Gaspillage 4 bytes/row | Downgrade à INT |
| **INT proche 2B (80%+)** | Risque overflow | Upgrade à BIGINT |
| **Status en INT** | Gaspillage 3 bytes | TINYINT ou ENUM |

**🔧 Actions correctives**

1. **Downgrade BIGINT inutile** :
   ```sql
   -- Table avec 100K lignes, max ID = 100000
   -- BIGINT gaspille 4 bytes × 100K = 400 KB
   
   -- ❌ Avant
   CREATE TABLE users (
       id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
       -- Max actuel : 100000 (< 4B)
   );
   
   -- ✅ Après
   ALTER TABLE users MODIFY id INT UNSIGNED AUTO_INCREMENT;
   -- Économie : 4 bytes × 100K lignes = 400 KB
   -- Sur table 10M lignes = 40 MB économisés
   ```

2. **Upgrade INT proche limite** :
   ```sql
   -- Table approche 2 milliards
   SELECT MAX(id) FROM orders;
   -- 1,850,000,000 (85% du max INT)
   
   -- Upgrade AVANT overflow
   ALTER TABLE orders MODIFY id BIGINT UNSIGNED AUTO_INCREMENT;
   
   -- ⚠️ Migration lourde, planifier maintenance window
   ```

3. **Status/Flags en TINYINT ou ENUM** :
   ```sql
   -- ❌ Gaspillage
   status INT  -- 4 bytes pour 3 valeurs
   
   -- ✅ Optimisé
   status TINYINT UNSIGNED  -- 1 byte (0-255)
   -- Ou
   status ENUM('pending', 'completed', 'cancelled')  -- 1-2 bytes
   ```

**📊 Validation**

```sql
-- Vérifier espace récupéré
SELECT 
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb_before
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- Après ALTER TABLE
-- size_mb_after < size_mb_before
```

---

#### ✅ 1.2 VARCHAR size optimal

**🔍 Diagnostic**

```sql
-- Analyser longueur réelle des VARCHAR
SELECT 
    c.TABLE_NAME,
    c.COLUMN_NAME,
    c.CHARACTER_MAXIMUM_LENGTH AS declared_length,
    ROUND(AVG(LENGTH(t.COLUMN_NAME)), 0) AS avg_actual_length,
    MAX(LENGTH(t.COLUMN_NAME)) AS max_actual_length,
    ROUND(100.0 * MAX(LENGTH(t.COLUMN_NAME)) / c.CHARACTER_MAXIMUM_LENGTH, 2) AS usage_pct
FROM information_schema.columns c
JOIN information_schema.tables tbl 
    ON c.TABLE_SCHEMA = tbl.TABLE_SCHEMA 
    AND c.TABLE_NAME = tbl.TABLE_NAME
-- Remplacer par query dynamique réelle sur chaque table
WHERE c.TABLE_SCHEMA = DATABASE()
  AND c.DATA_TYPE = 'varchar'
ORDER BY c.CHARACTER_MAXIMUM_LENGTH DESC;
```

**Impacts taille VARCHAR :**

```
VARCHAR(50)  : Longueur stockée + données (ex: "Bob" = 1 byte length + 3 bytes = 4 bytes)
VARCHAR(255) : Idem (pas de pénalité si données courtes)
VARCHAR(500) : Peut forcer tmp tables sur disque si total row > 65KB
VARCHAR(5000): ❌ Problème performance (sorting, tmp tables)

Règle : Déclarer taille réaliste (max observé × 1.5)
```

**⚠️ Seuils critiques**

| Déclaré | Réel max | Problème | Action |
|---------|----------|----------|--------|
| VARCHAR(255) | < 50 | Pas grave | OK (threshold index) |
| VARCHAR(1000) | < 100 | Gaspillage metadata | Réduire à 255 |
| VARCHAR(5000) | Variable | Tmp tables disque | TEXT + index prefix |

**🔧 Actions correctives**

1. **Réduire VARCHAR surdimensionné** :
   ```sql
   -- Colonne déclarée 1000, utilisée max 80
   ALTER TABLE products 
   MODIFY name VARCHAR(100) NOT NULL;
   
   -- ⚠️ Vérifier AVANT qu'aucune donnée > 100
   SELECT MAX(LENGTH(name)) FROM products;
   -- Si > 100 → ajuster limite
   ```

2. **VARCHAR(255) = threshold magique** :
   ```sql
   -- InnoDB : VARCHAR <= 255 utilise 1 byte pour length
   -- VARCHAR > 255 utilise 2 bytes pour length
   
   -- Préférer VARCHAR(255) si longueur variable < 255
   email VARCHAR(255)  -- vs VARCHAR(320) (RFC 5321)
   ```

3. **TEXT pour contenu long variable** :
   ```sql
   -- ❌ VARCHAR très long
   description VARCHAR(5000)  -- Problème sort, group by
   
   -- ✅ TEXT avec index prefix
   description TEXT,
   INDEX idx_description_prefix (description(100))
   
   -- TEXT stocké hors page (row pointer)
   ```

**📊 Validation**

```sql
-- Vérifier distribution longueurs
SELECT 
    LENGTH(name) AS length,
    COUNT(*) AS count
FROM products
GROUP BY LENGTH(name)
ORDER BY length DESC
LIMIT 10;

-- Si 99% < 100 chars → VARCHAR(100) optimal
```

---

#### ✅ 1.3 ENUM vs VARCHAR vs TINYINT

**🔍 Diagnostic**

```sql
-- Colonnes candidates pour ENUM
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    COUNT(DISTINCT COLUMN_NAME) AS distinct_values
FROM information_schema.columns c
-- Joindre avec données réelles (requête par table)
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE = 'varchar'
-- HAVING distinct_values < 10  -- Candidat ENUM
ORDER BY distinct_values;
```

**Comparaison types pour valeurs fixes :**

| Type | Stockage | Avantages | Inconvénients |
|------|----------|-----------|---------------|
| **ENUM** | 1-2 bytes | Compact, lisible SQL | Schema change pour nouvelle valeur |
| **VARCHAR** | Length + data | Flexible | 5-20× plus gros |
| **TINYINT + lookup** | 1 byte | Le plus compact | Moins lisible, JOIN requis |

**⚠️ Seuils critiques**

| Valeurs distinctes | Type recommandé | Raison |
|-------------------|-----------------|--------|
| **2-3 (fixe)** | ENUM | Optimal stockage + lisibilité |
| **4-10 (fixe)** | ENUM ou TINYINT | Selon fréquence changement |
| **10+ (évolutif)** | VARCHAR ou lookup table | Flexibilité |

**🔧 Actions correctives**

1. **VARCHAR → ENUM pour valeurs fixes** :
   ```sql
   -- ❌ Avant : status VARCHAR(20)
   -- Valeurs : 'pending', 'completed', 'cancelled'
   -- Stockage : 7-9 bytes par row
   
   -- ✅ Après : status ENUM
   ALTER TABLE orders 
   MODIFY status ENUM('pending', 'processing', 'shipped', 'completed', 'cancelled') 
   NOT NULL DEFAULT 'pending';
   -- Stockage : 1 byte par row (5 valeurs = 1 byte ENUM)
   
   -- Économie : Table 10M lignes = 10M × 8 bytes = 80 MB
   ```

2. **ENUM avec valeurs fréquentes** :
   ```sql
   -- Use case : 99% des valeurs dans 5 catégories
   -- 1% autres
   
   -- ✅ ENUM + 'other' + colonne description
   category ENUM('electronics', 'clothing', 'food', 'books', 'other'),
   category_other VARCHAR(50) NULL  -- Seulement si category = 'other'
   ```

3. **Lookup table pour évolutivité** :
   ```sql
   -- Si nouvelles valeurs fréquentes (sans ALTER TABLE)
   CREATE TABLE order_statuses (
       id TINYINT UNSIGNED PRIMARY KEY,
       name VARCHAR(50) NOT NULL UNIQUE
   );
   
   CREATE TABLE orders (
       id BIGINT PRIMARY KEY,
       status_id TINYINT UNSIGNED NOT NULL,
       FOREIGN KEY (status_id) REFERENCES order_statuses(id)
   );
   
   -- Ajout nouvelle valeur = INSERT (pas de schema change)
   ```

**📊 Validation**

```sql
-- Comparer taille table avant/après
SELECT 
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders';

-- Après VARCHAR → ENUM : data_mb devrait diminuer
```

---

#### ✅ 1.4 JSON vs colonnes normalisées

**🔍 Diagnostic**

```sql
-- Tables avec colonnes JSON
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE = 'json';

-- Analyser taille JSON
SELECT 
    AVG(LENGTH(json_column)) AS avg_json_size_bytes,
    MAX(LENGTH(json_column)) AS max_json_size_bytes,
    COUNT(DISTINCT JSON_KEYS(json_column)) AS distinct_key_sets
FROM table_with_json;
```

**JSON : Quand utiliser ?**

```
✅ Utiliser JSON :
- Schéma flexible (user preferences, metadata)
- Données semi-structurées (API responses)
- Évolution fréquente du schéma
- Queries rares sur attributs JSON

❌ Éviter JSON :
- Queries fréquentes sur attributs
- Agrégations (SUM, AVG sur attribut JSON)
- Index sur sous-attribut requis
- Données volumineuses
```

**⚠️ Seuils critiques**

| Use case | JSON | Colonnes normales |
|----------|------|-------------------|
| **User preferences** | ✅ Optimal | ❌ Rigide |
| **Product attributes (query fréquent)** | ❌ Lent | ✅ Optimal |
| **Logs non-structurés** | ✅ OK | ❌ Complexe |
| **Analytics queries** | ❌ Très lent | ✅ Rapide |

**🔧 Actions correctives**

1. **JSON → Colonnes pour queries fréquentes** :
   ```sql
   -- ❌ JSON queryé souvent
   CREATE TABLE products (
       id BIGINT PRIMARY KEY,
       attributes JSON
       -- {"price": 99.99, "stock": 50, "category": "electronics"}
   );
   
   SELECT * FROM products 
   WHERE JSON_EXTRACT(attributes, '$.price') > 50;
   -- Très lent, pas d'index possible
   
   -- ✅ Colonnes normalisées
   ALTER TABLE products 
   ADD COLUMN price DECIMAL(10,2),
   ADD COLUMN stock INT UNSIGNED,
   ADD COLUMN category VARCHAR(50);
   
   CREATE INDEX idx_price ON products(price);
   CREATE INDEX idx_category ON products(category);
   
   -- Queries 100-1000× plus rapides
   ```

2. **Colonnes → JSON pour flexibilité** :
   ```sql
   -- Si schéma change souvent (metadata, preferences)
   -- 50+ colonnes optionnelles → JSON
   
   -- Avant : 50 colonnes NULL
   user_pref_1 VARCHAR(50),
   user_pref_2 VARCHAR(50),
   -- ...
   user_pref_50 VARCHAR(50)
   
   -- Après : 1 colonne JSON
   preferences JSON
   -- {"theme": "dark", "language": "fr", "notifications": true}
   ```

3. **🆕 MariaDB 11.8 : Generated columns + index** :
   ```sql
   -- JSON avec index sur sous-attribut
   CREATE TABLE products (
       id BIGINT PRIMARY KEY,
       attributes JSON,
       
       -- Colonne générée pour attribut queryé
       price DECIMAL(10,2) GENERATED ALWAYS AS (JSON_VALUE(attributes, '$.price')) STORED,
       
       INDEX idx_price (price)
   );
   
   -- Query rapide avec index
   SELECT * FROM products WHERE price > 50;
   -- Utilise idx_price ✅
   ```

**📊 Validation**

```sql
-- Comparer performance
-- JSON query
EXPLAIN SELECT * FROM products 
WHERE JSON_EXTRACT(attributes, '$.category') = 'electronics';
-- type: ALL (full scan)

-- Colonne normale
EXPLAIN SELECT * FROM products WHERE category = 'electronics';
-- type: ref, key: idx_category ✅
```

---

### Catégorie 2 : Normalisation et Design

#### ✅ 2.1 Niveau de normalisation (3NF vs dénormalisation)

**🔍 Diagnostic**

```sql
-- Détecter colonnes dupliquées (candidates dénormalisation inverse)
SELECT 
    c1.TABLE_NAME AS table1,
    c1.COLUMN_NAME,
    c2.TABLE_NAME AS table2
FROM information_schema.columns c1
JOIN information_schema.columns c2
  ON c1.COLUMN_NAME = c2.COLUMN_NAME
  AND c1.TABLE_NAME < c2.TABLE_NAME
WHERE c1.TABLE_SCHEMA = DATABASE()
  AND c2.TABLE_SCHEMA = DATABASE()
  AND c1.COLUMN_NAME NOT IN ('id', 'created_at', 'updated_at')
ORDER BY c1.COLUMN_NAME;
```

**Niveaux normalisation :**

```
1NF : Valeurs atomiques (pas de listes dans colonnes)
2NF : Pas de dépendances partielles
3NF : Pas de dépendances transitives

Optimal OLTP : 3NF (pas de redondance)
Optimal OLAP : Dénormalisé (star schema, agrégats pré-calculés)
```

**⚠️ Seuils critiques**

| Cas d'usage | Normalisation | Raison |
|-------------|---------------|--------|
| **OLTP transactionnel** | 3NF strict | Éviter anomalies update |
| **OLTP lectures >> writes** | 2NF + dénorm ciblée | Optimiser JOINs fréquents |
| **OLAP reporting** | Star schema | Éviter JOINs complexes |
| **Mixed workload** | 3NF + tables summary | Compromis |

**🔧 Actions correctives**

1. **Normaliser données redondantes** :
   ```sql
   -- ❌ Dénormalisé (anomalies update)
   CREATE TABLE orders (
       id BIGINT PRIMARY KEY,
       customer_name VARCHAR(100),
       customer_email VARCHAR(255),
       customer_phone VARCHAR(20)
       -- Si customer change email → UPDATE tous orders ❌
   );
   
   -- ✅ Normalisé 3NF
   CREATE TABLE customers (
       id BIGINT PRIMARY KEY,
       name VARCHAR(100),
       email VARCHAR(255),
       phone VARCHAR(20)
   );
   
   CREATE TABLE orders (
       id BIGINT PRIMARY KEY,
       customer_id BIGINT NOT NULL,
       FOREIGN KEY (customer_id) REFERENCES customers(id)
   );
   ```

2. **Dénormaliser pour performance (OLTP lecture)** :
   ```sql
   -- Si query fréquente nécessite JOIN
   SELECT o.id, o.total, c.name
   FROM orders o
   JOIN customers c ON o.customer_id = c.id;
   -- Exécutée 10000×/sec
   
   -- Dénormaliser colonne rarement modifiée
   ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);
   
   -- Maintenir avec trigger
   CREATE TRIGGER update_customer_name
   AFTER UPDATE ON customers
   FOR EACH ROW
   UPDATE orders SET customer_name = NEW.name WHERE customer_id = NEW.id;
   
   -- Query sans JOIN (10× plus rapide)
   SELECT id, total, customer_name FROM orders;
   ```

3. **OLAP : Star schema** :
   ```sql
   -- Fact table (centre)
   CREATE TABLE fact_sales (
       sale_id BIGINT PRIMARY KEY,
       date_id INT,           -- FK → dim_date
       product_id INT,        -- FK → dim_product
       customer_id INT,       -- FK → dim_customer
       amount DECIMAL(10,2),
       quantity INT
   );
   
   -- Dimension tables (dénormalisées)
   CREATE TABLE dim_product (
       product_id INT PRIMARY KEY,
       name VARCHAR(100),
       category VARCHAR(50),
       brand VARCHAR(50),
       -- Toutes infos product (pas de JOIN sous-catégories)
   );
   
   -- Query OLAP rapide (1 ou 2 JOINs max)
   ```

**📊 Validation**

```sql
-- Comparer performance avant/après dénormalisation
-- Avant (JOIN)
EXPLAIN SELECT o.id, c.name FROM orders o JOIN customers c ON o.customer_id = c.id;
-- rows examined: 1M + 100K

-- Après (dénormalisé)
EXPLAIN SELECT id, customer_name FROM orders;
-- rows examined: 1M (pas de JOIN)
```

---

#### ✅ 2.2 Clés primaires (choix et type)

**🔍 Diagnostic**

```sql
-- Tables sans PRIMARY KEY
SELECT 
    t.TABLE_NAME,
    t.ENGINE,
    t.TABLE_ROWS
FROM information_schema.tables t
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND NOT EXISTS (
      SELECT 1 FROM information_schema.statistics s
      WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
        AND s.TABLE_NAME = t.TABLE_NAME
        AND s.INDEX_NAME = 'PRIMARY'
  );

-- PRIMARY KEY composites
SELECT 
    TABLE_NAME,
    GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS pk_columns,
    COUNT(*) AS pk_column_count
FROM information_schema.statistics
WHERE TABLE_SCHEMA = DATABASE()
  AND INDEX_NAME = 'PRIMARY'
GROUP BY TABLE_NAME
HAVING pk_column_count > 1;
```

**Types PRIMARY KEY :**

| Type | Avantages | Inconvénients | Use case |
|------|-----------|---------------|----------|
| **AUTO_INCREMENT INT** | Simple, séquentiel | Prédictible | Standard OLTP |
| **AUTO_INCREMENT BIGINT** | Scalable | +4 bytes | Twitter-scale |
| **UUID/GUID** | Distribué, unique globalement | 16 bytes, random I/O | Microservices |
| **Composite** | Logique métier | Index secondaires gros | Many-to-many |

**⚠️ Seuils critiques**

| Problème | Impact | Action |
|----------|--------|--------|
| **Pas de PRIMARY KEY** | 🔴 Critique | Créer immédiatement |
| **UUID sur grosse table** | Fragmentation index | INT + UUID unique |
| **PK composite 3+ colonnes** | Index énormes | Surrogate key |

**🔧 Actions correctives**

1. **Ajouter PRIMARY KEY manquant** :
   ```sql
   -- ❌ Table sans PK
   CREATE TABLE logs (
       message TEXT,
       created_at TIMESTAMP
   );
   -- InnoDB crée hidden PK (6 bytes) + pas de contrainte unicité
   
   -- ✅ Ajouter PK
   ALTER TABLE logs ADD COLUMN id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY FIRST;
   ```

2. **UUID : Compromis performance** :
   ```sql
   -- ❌ UUID PRIMARY KEY sur table volumineuse
   CREATE TABLE events (
       id BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID())),
       -- Random I/O, fragmentation index
   );
   
   -- ✅ INT AUTO_INCREMENT + UUID UNIQUE
   CREATE TABLE events (
       id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- Séquentiel
       uuid BINARY(16) UNIQUE NOT NULL DEFAULT (UUID_TO_BIN(UUID())),
       INDEX idx_uuid (uuid)
   );
   -- Utiliser id en interne, uuid pour API externe
   ```

3. **Composite PK → Surrogate key** :
   ```sql
   -- ❌ PK composite lourd
   CREATE TABLE user_roles (
       user_id BIGINT,
       role_id BIGINT,
       organization_id BIGINT,
       PRIMARY KEY (user_id, role_id, organization_id)
   );
   -- Index secondaires incluent les 3 colonnes (24 bytes)
   
   -- ✅ Surrogate key + UNIQUE composite
   CREATE TABLE user_roles (
       id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
       user_id BIGINT,
       role_id BIGINT,
       organization_id BIGINT,
       UNIQUE KEY uk_user_role_org (user_id, role_id, organization_id)
   );
   -- Index secondaires utilisent id (8 bytes)
   ```

**📊 Validation**

```sql
-- Vérifier taille index après changement PK
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    ROUND(STAT_VALUE * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE DATABASE_NAME = 'mydb'
  AND TABLE_NAME = 'events'
ORDER BY STAT_VALUE DESC;
```

---

### Catégorie 3 : Partitionnement

#### ✅ 3.1 Candidates au partitionnement

**🔍 Diagnostic**

```sql
-- Tables volumineuses (> 10M lignes ou > 10GB)
SELECT 
    TABLE_NAME,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS size_gb,
    ROUND(DATA_LENGTH / TABLE_ROWS, 0) AS avg_row_bytes,
    CREATE_TIME,
    CASE 
        WHEN TABLE_ROWS > 100000000 THEN '🔴 CRITIQUE (>100M)'
        WHEN TABLE_ROWS > 10000000 THEN '🟡 Candidat (>10M)'
        ELSE '🟢 OK'
    END AS partitioning_status
FROM information_schema.tables
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_ROWS DESC;

-- Tables avec pattern temporel (time-series)
SELECT 
    TABLE_NAME,
    COLUMN_NAME
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND (COLUMN_NAME LIKE '%date%' OR COLUMN_NAME LIKE '%created%' OR DATA_TYPE IN ('date', 'datetime', 'timestamp'))
ORDER BY TABLE_NAME;
```

**Candidats partitionnement :**

```
✅ Partitionner si :
1. Table > 10M lignes ou > 10GB
2. Queries filtrées par date/range (WHERE created_at > '2024-01-01')
3. Archivage périodique (DROP partition vs DELETE)
4. Données time-series (logs, events, metrics)

❌ Ne PAS partitionner :
- Table < 1M lignes (overhead inutile)
- Queries sans filtre partition key
- Nombreux index (multiplication par nb partitions)
```

**⚠️ Seuils critiques**

| Taille table | Partitionnement | Raison |
|--------------|-----------------|--------|
| **< 1M lignes** | ❌ Non | Overhead > bénéfice |
| **1-10M lignes** | 🟡 Optionnel | Selon use case |
| **> 10M lignes** | ✅ Recommandé | Maintenance + perf |
| **> 100M lignes** | 🔴 Obligatoire | Management impossible sans |

**🔧 Actions correctives**

1. **RANGE partitioning (time-series)** :
   ```sql
   -- Table logs 100M lignes, requêtes sur created_at
   
   -- Partitionner par mois
   ALTER TABLE logs PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
       PARTITION p202401 VALUES LESS THAN (202402),  -- Jan 2024
       PARTITION p202402 VALUES LESS THAN (202403),  -- Feb 2024
       PARTITION p202403 VALUES LESS THAN (202404),  -- Mar 2024
       -- ...
       PARTITION p202412 VALUES LESS THAN (202501),  -- Dec 2024
       PARTITION p_future VALUES LESS THAN MAXVALUE
   );
   
   -- Query sur 1 mois = scan 1 partition (vs table entière)
   SELECT * FROM logs WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
   -- Partition pruning : Scan seulement p202403 ✅
   ```

2. **Archivage avec DROP PARTITION** :
   ```sql
   -- Supprimer données > 1 an
   
   -- ❌ Sans partition : DELETE (très lent)
   DELETE FROM logs WHERE created_at < '2023-01-01';
   -- Sur 100M lignes = heures, lock table
   
   -- ✅ Avec partition : DROP (instantané)
   ALTER TABLE logs DROP PARTITION p202301, p202302, p202303;
   -- Millisecondes, pas de lock table ✅
   ```

3. **LIST partitioning (géographie, catégorie)** :
   ```sql
   -- Partitionner par région
   ALTER TABLE orders PARTITION BY LIST (country_code) (
       PARTITION p_us VALUES IN ('US'),
       PARTITION p_eu VALUES IN ('FR', 'DE', 'UK', 'ES', 'IT'),
       PARTITION p_asia VALUES IN ('JP', 'CN', 'KR', 'IN'),
       PARTITION p_other VALUES IN (DEFAULT)
   );
   
   -- Query filtrée par pays = 1 partition
   SELECT * FROM orders WHERE country_code = 'FR';
   -- Scan seulement p_eu
   ```

**📊 Validation**

```sql
-- Vérifier partition pruning
EXPLAIN PARTITIONS
SELECT * FROM logs WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
-- partitions: p202403 (1 seule partition scannée ✅)

-- Statistiques par partition
SELECT 
    PARTITION_NAME,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.partitions
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'logs'
ORDER BY PARTITION_ORDINAL_POSITION;
```

---

#### ✅ 3.2 Stratégie d'archivage

**🔍 Diagnostic**

```sql
-- Distribution temporelle des données
SELECT 
    DATE_FORMAT(created_at, '%Y-%m') AS month,
    COUNT(*) AS row_count,
    ROUND(COUNT(*) * AVG_ROW_LENGTH / 1024 / 1024, 2) AS estimated_mb
FROM large_table
GROUP BY DATE_FORMAT(created_at, '%Y-%m')
ORDER BY month DESC;

-- Données anciennes jamais accédées
SELECT 
    MIN(created_at) AS oldest_record,
    MAX(created_at) AS newest_record,
    DATEDIFF(NOW(), MIN(created_at)) AS days_retention
FROM large_table;
```

**Stratégies archivage :**

```
1. Hard delete : DELETE (lent, lock)
2. Soft delete : is_archived = 1 (espace pas libéré)
3. Archive table : INSERT INTO archive + DELETE FROM main
4. Partition DROP : ALTER TABLE DROP PARTITION (instantané ✅)
5. External storage : Export S3/parquet + DELETE
```

**⚠️ Seuils critiques**

| Rétention | Stratégie | Raison |
|-----------|-----------|--------|
| **< 30 jours** | Partition RANGE journalière | Archivage fréquent |
| **30-365 jours** | Partition RANGE mensuelle | Équilibre |
| **1-3 ans** | Archive table | Queries rares |
| **> 3 ans** | Cold storage (S3) | Compliance, coût |

**🔧 Actions correctives**

1. **Archive table automatisée** :
   ```sql
   -- Table principale (données chaudes)
   CREATE TABLE orders (
       id BIGINT PRIMARY KEY,
       created_at DATETIME NOT NULL,
       -- ...
   ) PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
       -- Partitions récentes
   );
   
   -- Table archive (données froides)
   CREATE TABLE orders_archive LIKE orders;
   
   -- Event Scheduler : Archivage automatique mensuel
   CREATE EVENT archive_old_orders
   ON SCHEDULE EVERY 1 MONTH
   DO
   BEGIN
       -- Calculer partition à archiver (> 1 an)
       SET @old_partition = CONCAT('p', DATE_FORMAT(DATE_SUB(NOW(), INTERVAL 13 MONTH), '%Y%m'));
       
       -- Copier vers archive
       SET @sql = CONCAT('INSERT INTO orders_archive SELECT * FROM orders PARTITION (', @old_partition, ')');
       PREPARE stmt FROM @sql;
       EXECUTE stmt;
       DEALLOCATE PREPARE stmt;
       
       -- Supprimer partition
       SET @sql = CONCAT('ALTER TABLE orders DROP PARTITION ', @old_partition);
       PREPARE stmt FROM @sql;
       EXECUTE stmt;
       DEALLOCATE PREPARE stmt;
   END;
   ```

2. **Soft delete avec filtre applicatif** :
   ```sql
   -- Colonne archived_at
   ALTER TABLE orders ADD COLUMN archived_at DATETIME NULL;
   CREATE INDEX idx_archived_at ON orders(archived_at);
   
   -- Archiver logiquement (pas de suppression)
   UPDATE orders 
   SET archived_at = NOW() 
   WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);
   
   -- Application filtre automatique
   SELECT * FROM orders WHERE archived_at IS NULL;
   
   -- Purge physique périodique (batch)
   DELETE FROM orders WHERE archived_at < DATE_SUB(NOW(), INTERVAL 3 YEAR) LIMIT 10000;
   ```

3. **ColumnStore pour cold data** :
   ```sql
   -- 🆕 MariaDB 11.8 : Données anciennes → ColumnStore (compression)
   
   -- Table hot (InnoDB)
   CREATE TABLE metrics_hot (
       id BIGINT AUTO_INCREMENT PRIMARY KEY,
       created_at DATETIME,
       value DOUBLE
   ) ENGINE=InnoDB;
   
   -- Table cold (ColumnStore - compression 10×)
   CREATE TABLE metrics_cold (
       id BIGINT,
       created_at DATETIME,
       value DOUBLE
   ) ENGINE=ColumnStore;
   
   -- Migration mensuelle hot → cold
   INSERT INTO metrics_cold 
   SELECT * FROM metrics_hot 
   WHERE created_at < DATE_SUB(NOW(), INTERVAL 3 MONTH);
   
   DELETE FROM metrics_hot 
   WHERE created_at < DATE_SUB(NOW(), INTERVAL 3 MONTH);
   ```

**📊 Validation**

```sql
-- Vérifier espace récupéré
SELECT 
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS size_gb
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME IN ('orders', 'orders_archive');

-- orders devrait diminuer après archivage
```

---

### Catégorie 4 : Contraintes et Relations

#### ✅ 4.1 Foreign Keys (performance vs intégrité)

**🔍 Diagnostic**

```sql
-- Foreign Keys définies
SELECT 
    kcu.TABLE_NAME,
    kcu.COLUMN_NAME,
    kcu.REFERENCED_TABLE_NAME,
    kcu.REFERENCED_COLUMN_NAME,
    rc.UPDATE_RULE,
    rc.DELETE_RULE
FROM information_schema.key_column_usage kcu
JOIN information_schema.referential_constraints rc
  ON kcu.CONSTRAINT_NAME = rc.CONSTRAINT_NAME
  AND kcu.TABLE_SCHEMA = rc.CONSTRAINT_SCHEMA
WHERE kcu.TABLE_SCHEMA = DATABASE()
  AND kcu.REFERENCED_TABLE_NAME IS NOT NULL;

-- Tables sans FK mais devraient en avoir
-- (Détection manuelle via nommage : *_id)
SELECT 
    TABLE_NAME,
    COLUMN_NAME
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND COLUMN_NAME LIKE '%\_id'
  AND COLUMN_NAME != 'id'
  AND NOT EXISTS (
      SELECT 1 FROM information_schema.key_column_usage kcu
      WHERE kcu.TABLE_SCHEMA = DATABASE()
        AND kcu.TABLE_NAME = columns.TABLE_NAME
        AND kcu.COLUMN_NAME = columns.COLUMN_NAME
        AND kcu.REFERENCED_TABLE_NAME IS NOT NULL
  );
```

**Foreign Keys : Trade-offs**

| Aspect | Avec FK | Sans FK |
|--------|---------|---------|
| **Intégrité** | ✅ Garantie | ❌ Doit vérifier en app |
| **Performance INSERT** | 🟡 -5-10% | ✅ Plus rapide |
| **Performance DELETE** | 🟡 Cascade check | ✅ Plus rapide |
| **Développement** | ✅ Sécurisé | ❌ Bugs possibles |

**⚠️ Seuils critiques**

| Cas d'usage | FK | Raison |
|-------------|-----|--------|
| **OLTP standard** | ✅ Activer | Intégrité critique |
| **Bulk loading** | ⚠️ Désactiver temporairement | Performance import |
| **OLAP read-only** | 🟡 Optionnel | Pas de writes |
| **Sharded tables** | ❌ Impossible | FK cross-shard |

**🔧 Actions correctives**

1. **Ajouter FK manquantes** :
   ```sql
   -- Colonne orders.customer_id sans FK
   ALTER TABLE orders
   ADD CONSTRAINT fk_orders_customer
   FOREIGN KEY (customer_id) REFERENCES customers(id)
   ON DELETE RESTRICT  -- Empêcher suppression client avec orders
   ON UPDATE CASCADE;  -- Update ID propage
   
   -- ⚠️ Vérifier données valides AVANT
   SELECT DISTINCT o.customer_id
   FROM orders o
   LEFT JOIN customers c ON o.customer_id = c.id
   WHERE c.id IS NULL;
   -- Si résultat non vide : nettoyer données orphelines
   ```

2. **Désactiver FK pour bulk import** :
   ```sql
   -- Import massif (100M lignes)
   SET foreign_key_checks = 0;
   
   LOAD DATA INFILE '/tmp/orders.csv'
   INTO TABLE orders
   FIELDS TERMINATED BY ',';
   
   SET foreign_key_checks = 1;
   
   -- Gain : 20-40% performance import
   ```

3. **CASCADE DELETE avec prudence** :
   ```sql
   -- ⚠️ Dangereux
   FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
   -- DELETE 1 user → DELETE tous ses orders/posts/comments
   
   -- ✅ Plus sûr : RESTRICT
   FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT
   -- Empêche suppression user si données liées
   
   -- Ou soft delete
   UPDATE users SET deleted_at = NOW() WHERE id = 123;
   ```

**📊 Validation**

```sql
-- Vérifier intégrité référentielle sans FK
SELECT 'orders → customers' AS relation, COUNT(*) AS orphans
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL

UNION ALL

SELECT 'posts → users', COUNT(*)
FROM posts p
LEFT JOIN users u ON p.user_id = u.id
WHERE u.id IS NULL;

-- orphans devrait être 0
```

---

## 📊 Script d'audit automatisé

### audit-schema.sql

```sql
-- ============================================================================
-- SCRIPT D'AUDIT SCHÉMA MARIADB 11.8
-- ============================================================================

SELECT '=' AS '';
SELECT 'AUDIT SCHÉMA DATABASE' AS '';
SELECT '=' AS '';

-- 1. TYPES DE DONNÉES SUBOPTIMAUX
SELECT '\n1. COLONNES BIGINT SOUS-UTILISÉES' AS '';
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    '🟡 Downgrade à INT ?' AS recommendation
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE = 'bigint'
  AND COLUMN_NAME LIKE '%id%'
LIMIT 10;

-- 2. VARCHAR SURDIMENSIONNÉS
SELECT '\n2. VARCHAR POTENTIELLEMENT SURDIMENSIONNÉS' AS '';
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    CHARACTER_MAXIMUM_LENGTH AS declared_length,
    '🟡 Analyser usage réel' AS recommendation
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE = 'varchar'
  AND CHARACTER_MAXIMUM_LENGTH > 500
ORDER BY CHARACTER_MAXIMUM_LENGTH DESC
LIMIT 10;

-- 3. TABLES SANS PRIMARY KEY
SELECT '\n3. TABLES SANS PRIMARY KEY (🔴 CRITIQUE)' AS '';
SELECT 
    t.TABLE_NAME,
    t.TABLE_ROWS,
    '🔴 AJOUTER PK' AS action
FROM information_schema.tables t
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND NOT EXISTS (
      SELECT 1 FROM information_schema.statistics s
      WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
        AND s.TABLE_NAME = t.TABLE_NAME
        AND s.INDEX_NAME = 'PRIMARY'
  );

-- 4. CANDIDATES PARTITIONNEMENT
SELECT '\n4. TABLES CANDIDATES PARTITIONNEMENT' AS '';
SELECT 
    TABLE_NAME,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS size_gb,
    CASE 
        WHEN TABLE_ROWS > 100000000 THEN '🔴 URGENT'
        WHEN TABLE_ROWS > 10000000 THEN '🟡 RECOMMANDÉ'
        ELSE '🟢 OK'
    END AS status
FROM information_schema.tables
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_TYPE = 'BASE TABLE'
  AND TABLE_ROWS > 1000000
ORDER BY TABLE_ROWS DESC
LIMIT 10;

-- 5. COLONNES JSON
SELECT '\n5. COLONNES JSON (Analyser use case)' AS '';
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    '🟡 Vérifier queries fréquentes' AS note
FROM information_schema.columns
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE = 'json';

-- 6. FOREIGN KEYS MANQUANTES
SELECT '\n6. COLONNES *_ID SANS FOREIGN KEY' AS '';
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    '🟡 Vérifier si FK requise' AS recommendation
FROM information_schema.columns c
WHERE TABLE_SCHEMA = DATABASE()
  AND COLUMN_NAME LIKE '%\_id'
  AND COLUMN_NAME != 'id'
  AND NOT EXISTS (
      SELECT 1 FROM information_schema.key_column_usage kcu
      WHERE kcu.TABLE_SCHEMA = DATABASE()
        AND kcu.TABLE_NAME = c.TABLE_NAME
        AND kcu.COLUMN_NAME = c.COLUMN_NAME
        AND kcu.REFERENCED_TABLE_NAME IS NOT NULL
  )
LIMIT 20;

SELECT '=' AS '';
SELECT 'FIN AUDIT SCHÉMA' AS '';
```

---

## ✅ Points clés à retenir

- 🔢 **Types entiers** : INT pour <4B, BIGINT seulement si nécessaire (économie 4 bytes/row)
- 📏 **VARCHAR(255)** : Threshold magique (1 byte vs 2 bytes pour length)
- 🔤 **ENUM** : 1-2 bytes vs 5-20 bytes pour VARCHAR (valeurs fixes)
- 📦 **JSON** : Flexible mais lent, colonnes générées pour queries fréquentes
- 📐 **3NF** : Optimal OLTP, dénormaliser seulement si prouvé nécessaire
- 🔑 **PRIMARY KEY** : Toujours AUTO_INCREMENT INT/BIGINT (UUID = problème I/O)
- 📊 **Partitionnement** : Tables >10M lignes + filtre temporel = RANGE partition
- 🗄️ **Archivage** : DROP PARTITION >> DELETE (instantané vs heures)
- 🔗 **Foreign Keys** : Activer sauf cas spéciaux (bulk import, sharding)
- 🆕 **MariaDB 11.8** : ColumnStore pour cold data (compression 10×)

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Data Types](https://mariadb.com/kb/en/data-types/)
- [Partitioning](https://mariadb.com/kb/en/partitioning-overview/)
- [Foreign Keys](https://mariadb.com/kb/en/foreign-keys/)

### Autres checklists
- [E.1 - Audit de configuration](./01-audit-configuration.md)
- [E.2 - Audit d'indexation](./02-audit-indexation.md)
- [E.3 - Audit de requêtes](./03-audit-requetes.md)

### Annexes complémentaires
- [Annexe D - Configurations de référence](/annexes/configuration-reference/README.md)
- [Section 15 - Performance et Tuning](/15-performance-tuning/README.md)

---

## 🎉 Annexe E - Checklist Performance : COMPLÈTE

Vous disposez maintenant de **4 checklists complètes** :
- ✅ **E.0** : Méthodologie audit (6 étapes)
- ✅ **E.1** : Configuration serveur (20 points)
- ✅ **E.2** : Indexation (15 points)
- ✅ **E.3** : Requêtes SQL (18 points)
- ✅ **E.4** : Schéma database (16 points)

**Total : 69 points de vérification** pour optimiser MariaDB 11.8 de A à Z.

---

**MariaDB** : Version 11.8 LTS  

⏭️ [Nouveautés MariaDB 11.8 LTS en un Coup d'Œil](/annexes/nouveautes-11-8/README.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 Création et gestion des index

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures

> **Prérequis** :
> - Sections 5.1 à 5.3 - Tous les types d'index
> - Compréhension des requêtes SQL
> - Notions d'administration de base de données

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser les différentes syntaxes de création d'index (CREATE INDEX, ALTER TABLE)
- Utiliser Online DDL pour créer des index sans bloquer la production
- Gérer le cycle de vie des index (création, modification, suppression)
- Choisir les bonnes options selon le type d'index et le moteur de stockage
- Maintenir et optimiser les index existants
- Diagnostiquer et résoudre les problèmes d'index
- Appliquer les bonnes pratiques de gestion d'index en production

---

## Introduction

La **gestion des index** est cruciale pour maintenir des performances optimales. Un index mal créé ou mal maintenu peut dégrader les performances au lieu de les améliorer.

### Le cycle de vie d'un index

```
1. PLANIFICATION
   - Analyser les requêtes
   - Identifier les colonnes candidates
   - Choisir le type d'index

2. CRÉATION
   - CREATE INDEX ou ALTER TABLE
   - Options et paramètres
   - Online DDL si production

3. MONITORING
   - Vérifier l'utilisation
   - Mesurer l'impact
   - Analyser les performances

4. MAINTENANCE
   - OPTIMIZE TABLE
   - ANALYZE TABLE
   - Rebuild si fragmentation

5. ÉVOLUTION
   - Modifier si nécessaire
   - Supprimer si inutilisé
   - Ajuster les paramètres
```

---

## Syntaxes de création d'index

### CREATE INDEX : La méthode dédiée

```sql
-- Syntaxe générale
CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX index_name
ON table_name (column1 [ASC|DESC], column2, ...)
[USING {BTREE | HASH | HNSW}]
[WITH (option1=value1, option2=value2, ...)]
[ALGORITHM={DEFAULT|INPLACE|COPY}]
[LOCK={DEFAULT|NONE|SHARED|EXCLUSIVE}];

-- Exemples basiques
-- Index B-Tree simple
CREATE INDEX idx_lastname ON users(last_name);

-- Index B-Tree composite
CREATE INDEX idx_name_email ON users(last_name, first_name, email);

-- Index unique
CREATE UNIQUE INDEX idx_email_unique ON users(email);

-- Index avec ordre descendant
CREATE INDEX idx_created_desc ON posts(created_at DESC);

-- Index sur préfixe de colonne
CREATE INDEX idx_url_prefix ON pages(url(100));
```

### ALTER TABLE : La méthode alternative

```sql
-- Syntaxe
ALTER TABLE table_name
ADD [UNIQUE|FULLTEXT|SPATIAL] INDEX index_name (columns)
[USING {BTREE | HASH | HNSW}]
[WITH (...)];

-- Exemples
-- Index B-Tree
ALTER TABLE users ADD INDEX idx_city (city);

-- Index unique
ALTER TABLE users ADD UNIQUE INDEX idx_username (username);

-- Index composite
ALTER TABLE orders ADD INDEX idx_customer_date (customer_id, order_date);

-- Plusieurs index en une commande
ALTER TABLE products
ADD INDEX idx_category (category_id),
ADD INDEX idx_price (price),
ADD FULLTEXT INDEX idx_search (name, description);
```

### CREATE TABLE : Définition à la création

```sql
-- Index définis lors de la création de table
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE,  -- Index unique implicite
    email VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    city VARCHAR(100),
    created_at DATETIME,

    -- Index explicites
    INDEX idx_name (last_name, first_name),
    INDEX idx_city (city),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;
```

### Différences et quand utiliser quoi

| Méthode | Avantages | Inconvénients | Usage |
|---------|-----------|---------------|-------|
| **CREATE INDEX** | Syntaxe claire, lisible | Bloque la table | Tables existantes |
| **ALTER TABLE** | Peut créer plusieurs index | Plus verbeux | Modifications multiples |
| **CREATE TABLE** | Tout défini d'un coup | Pas flexible | Création initiale |

---

## Types d'index : Syntaxes spécifiques

### Index B-Tree (par défaut)

```sql
-- Implicite (BTREE par défaut)
CREATE INDEX idx_status ON orders(status);

-- Explicite
CREATE INDEX idx_status ON orders(status) USING BTREE;

-- Index composite
CREATE INDEX idx_customer_status ON orders(customer_id, status, order_date);

-- Index avec ordre mixte
CREATE INDEX idx_mixed ON logs(severity ASC, timestamp DESC);

-- Index partiel sur préfixe
CREATE INDEX idx_description ON products(description(200));
```

### Index Hash (MEMORY engine)

```sql
-- Index Hash sur table MEMORY
CREATE TABLE sessions (
    session_id CHAR(64) PRIMARY KEY,
    user_id INT,
    data BLOB,
    INDEX idx_session (session_id) USING HASH
) ENGINE=MEMORY;

-- ⚠️ Sur InnoDB, USING HASH est ignoré et converti en BTREE
CREATE TABLE cache (
    key_id VARCHAR(255) PRIMARY KEY,
    value TEXT,
    INDEX idx_key (key_id) USING HASH  -- Converti en BTREE !
) ENGINE=InnoDB;
```

### Index Full-Text

```sql
-- Index Full-Text sur une colonne
CREATE FULLTEXT INDEX idx_content ON articles(content);

-- Index Full-Text sur plusieurs colonnes
CREATE FULLTEXT INDEX idx_search ON articles(title, content, tags);

-- Avec ALTER TABLE
ALTER TABLE blog_posts
ADD FULLTEXT INDEX idx_full_search (title, body);

-- Lors de la création de table
CREATE TABLE documentation (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    content LONGTEXT,
    FULLTEXT INDEX idx_docs (title, content)
) ENGINE=InnoDB;

-- ⚠️ Contraintes
-- - Colonnes TEXT ou VARCHAR uniquement
-- - InnoDB, MyISAM, ou Aria
-- - Pas de colonnes NULL (recommandé NOT NULL)
```

### Index Spatial

```sql
-- Index Spatial sur POINT
CREATE SPATIAL INDEX idx_location ON stores(location);

-- Avec ALTER TABLE
ALTER TABLE restaurants
ADD SPATIAL INDEX idx_coordinates (coordinates);

-- Lors de la création de table
CREATE TABLE properties (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    location POINT NOT NULL SRID 4326,
    SPATIAL INDEX idx_location (location)
) ENGINE=InnoDB;

-- ⚠️ Contraintes
-- - Colonne NOT NULL obligatoire
-- - Types géométriques uniquement
-- - Une seule colonne par index Spatial
```

### Index VECTOR (HNSW) 🆕

```sql
-- Index VECTOR avec paramètres HNSW
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (
    m = 16,                 -- Connexions par nœud
    ef_construction = 200,  -- Qualité construction
    metric = 'cosine'       -- Métrique de distance
);

-- Variations de paramètres
-- Haute performance (recall ~94%)
CREATE INDEX idx_fast ON docs(embedding)
USING HNSW
WITH (m=12, ef_construction=150, metric='cosine');

-- Haute précision (recall ~97%)
CREATE INDEX idx_precise ON docs(embedding)
USING HNSW
WITH (m=32, ef_construction=400, metric='cosine');

-- Pour recommandations (dot product)
CREATE INDEX idx_reco ON products(features)
USING HNSW
WITH (m=24, ef_construction=300, metric='dot');

-- Pour images (euclidean)
CREATE INDEX idx_images ON photos(visual_embedding)
USING HNSW
WITH (m=32, ef_construction=400, metric='euclidean');

-- Avec ALTER TABLE
ALTER TABLE knowledge_base
ADD INDEX idx_vector (embedding)
USING HNSW
WITH (m=16, ef_construction=200, metric='cosine');

-- ⚠️ Contraintes
-- - Type VECTOR(dimensions) obligatoire
-- - Colonne NOT NULL
-- - InnoDB uniquement
-- - MariaDB 11.8+ requis
```

---

## Online DDL : Créer des index sans downtime

### Concept d'Online DDL

**Online DDL** permet de modifier la structure d'une table (dont créer des index) **sans bloquer** les opérations DML (INSERT, UPDATE, DELETE).

```
Modes de verrouillage :

LOCK=NONE (Online)
├─ Lectures : ✅ Autorisées
├─ Écritures : ✅ Autorisées
└─ Durée : Longue (mais non bloquant)

LOCK=SHARED
├─ Lectures : ✅ Autorisées
├─ Écritures : ❌ Bloquées
└─ Durée : Moyenne

LOCK=EXCLUSIVE
├─ Lectures : ❌ Bloquées
├─ Écritures : ❌ Bloquées
└─ Durée : Courte (fast)
```

### Algorithmes de création d'index

```sql
-- ALGORITHM=INPLACE : Online, pas de copie de table
CREATE INDEX idx_status ON orders(status)
ALGORITHM=INPLACE, LOCK=NONE;

-- ALGORITHM=COPY : Copie toute la table (lent, bloquant)
CREATE INDEX idx_category ON products(category)
ALGORITHM=COPY;

-- DEFAULT : MariaDB choisit automatiquement
CREATE INDEX idx_price ON products(price)
ALGORITHM=DEFAULT;
```

**Tableau de compatibilité** :

| Type d'index | ALGORITHM=INPLACE | LOCK=NONE | Notes |
|--------------|-------------------|-----------|-------|
| **B-Tree** | ✅ Oui | ✅ Oui | Optimal |
| **Hash** | ✅ Oui (MEMORY) | ✅ Oui | MEMORY uniquement |
| **Full-Text** | ✅ Oui (InnoDB) | ✅ Oui | InnoDB depuis 10.0 |
| **Spatial** | ✅ Oui | ✅ Oui | InnoDB |
| **VECTOR** 🆕 | ✅ Oui | ✅ Oui | InnoDB 11.8+ |

### Création d'index en production

**Scénario 1 : Petite table (< 100 000 lignes)**

```sql
-- Création simple et rapide
CREATE INDEX idx_category ON products(category);
-- Temps : quelques secondes, impact minimal
```

**Scénario 2 : Grande table (> 10 millions de lignes)**

```sql
-- Utiliser Online DDL explicitement
CREATE INDEX idx_customer ON orders(customer_id)
ALGORITHM=INPLACE,
LOCK=NONE;

-- Surveiller la progression
SHOW PROCESSLIST;

-- Vérifier l'état
SELECT
    STAGE,
    WORK_COMPLETED,
    WORK_ESTIMATED,
    ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 2) AS progress_pct
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE 'stage/innodb/alter%';
```

**Scénario 3 : Index Full-Text sur table volumineuse**

```sql
-- Full-Text peut être long, préparer l'environnement
SET GLOBAL innodb_ft_cache_size = 134217728;  -- 128 MB

CREATE FULLTEXT INDEX idx_content ON articles(title, content)
ALGORITHM=INPLACE,
LOCK=NONE;

-- Temps typique : 1-5 minutes par million de lignes
```

**Scénario 4 : Index VECTOR HNSW** 🆕

```sql
-- Configuration préalable
SET GLOBAL innodb_vector_cache_size = 2147483648;  -- 2 GB
SET GLOBAL innodb_vector_threads = 8;

-- Création Online
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (m=16, ef_construction=200, metric='cosine')
ALGORITHM=INPLACE,
LOCK=NONE;

-- Temps typique : 2-10 minutes par million de vecteurs
-- Dépend de : dimensions, m, ef_construction, CPU/RAM
```

### Limites et contraintes Online DDL

```sql
-- ❌ Pas possible en Online :
-- 1. Changer le type de colonne
ALTER TABLE users MODIFY COLUMN age SMALLINT;  -- Bloquant

-- 2. Ajouter PRIMARY KEY si table sans PK
ALTER TABLE logs ADD PRIMARY KEY (id);  -- Bloquant

-- 3. Changer le moteur de stockage
ALTER TABLE data ENGINE=MyISAM;  -- Bloquant

-- ✅ Possible en Online :
-- 1. Ajouter/supprimer index secondaires
-- 2. Ajouter colonnes (avec DEFAULT)
-- 3. Renommer colonnes (metadata only)
-- 4. Modifier AUTO_INCREMENT
```

---

## Options et paramètres d'index

### Ordre de tri : ASC vs DESC

```sql
-- Index avec ordre par défaut (ASC)
CREATE INDEX idx_created ON posts(created_at);

-- Index avec ordre descendant
CREATE INDEX idx_created_desc ON posts(created_at DESC);

-- Index composite avec ordres mixtes
CREATE INDEX idx_user_posts ON posts(user_id ASC, created_at DESC);

-- Impact :
-- ORDER BY created_at DESC utilise idx_created_desc sans tri
-- ORDER BY created_at ASC peut utiliser idx_created ou idx_created_desc
```

### Index sur préfixe de colonne

```sql
-- Index sur préfixe (premiers N caractères)
CREATE INDEX idx_url ON pages(url(100));  -- 100 premiers caractères

-- Calculer la longueur optimale
SELECT
    COUNT(DISTINCT url) AS total_distinct,
    COUNT(DISTINCT LEFT(url, 50)) AS distinct_50,
    COUNT(DISTINCT LEFT(url, 100)) AS distinct_100,
    COUNT(DISTINCT LEFT(url, 200)) AS distinct_200
FROM pages;

-- Si distinct_100 ≈ total_distinct : 100 est suffisant
```

**Trade-offs** :

| Longueur préfixe | Taille index | Sélectivité | Faux positifs |
|------------------|--------------|-------------|---------------|
| 50 caractères | Petit | Faible | Élevé |
| 100 caractères | Moyen | Bonne | Faible |
| 200 caractères | Grand | Excellente | Très faible |
| Full (pas de préfixe) | Très grand | Parfaite | Aucun |

### Index invisibles

```sql
-- Créer un index invisible (test avant production)
CREATE INDEX idx_test ON orders(status) INVISIBLE;

-- L'index existe mais n'est pas utilisé par l'optimiseur
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- N'utilise pas idx_test

-- Activer pour une session de test
SET optimizer_switch='use_invisible_indexes=on';
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- Utilise maintenant idx_test

-- Si bénéfique : rendre visible
ALTER TABLE orders ALTER INDEX idx_test VISIBLE;

-- Sinon : supprimer
DROP INDEX idx_test ON orders;
```

### Index conditionnels (workaround)

MariaDB ne supporte pas nativement `WHERE` dans les index, mais on peut simuler :

```sql
-- PostgreSQL (pas supporté MariaDB) :
-- CREATE INDEX idx_active ON users(email) WHERE active = TRUE;

-- Workaround MariaDB : colonne générée
ALTER TABLE users
ADD COLUMN email_if_active VARCHAR(255) AS (
    IF(active = TRUE, email, NULL)
) STORED;

CREATE INDEX idx_email_active ON users(email_if_active);

-- Requête utilise l'index
SELECT * FROM users WHERE email_if_active = 'user@example.com';
```

---

## Modification d'index

### Renommer un index

```sql
-- ALTER TABLE ... RENAME INDEX (MariaDB 10.5+)
ALTER TABLE users RENAME INDEX old_idx_name TO new_idx_name;

-- Avant 10.5 : Supprimer et recréer
DROP INDEX old_idx_name ON users;
CREATE INDEX new_idx_name ON users(column);
```

### Changer de visible à invisible

```sql
-- Rendre invisible
ALTER TABLE orders ALTER INDEX idx_status INVISIBLE;

-- Rendre visible
ALTER TABLE orders ALTER INDEX idx_status VISIBLE;

-- Vérifier l'état
SHOW INDEX FROM orders;
-- Colonne Visible: YES ou NO
```

### Reconstruire un index

```sql
-- Méthode 1 : DROP + CREATE
DROP INDEX idx_category ON products;
CREATE INDEX idx_category ON products(category);

-- Méthode 2 : ALTER TABLE ... DROP ... ADD (atomic)
ALTER TABLE products
DROP INDEX idx_category,
ADD INDEX idx_category (category);

-- Méthode 3 : OPTIMIZE TABLE (reconstruit tous les index)
OPTIMIZE TABLE products;
```

### Changer les paramètres HNSW 🆕

```sql
-- Les paramètres HNSW ne peuvent pas être modifiés après création
-- Il faut recréer l'index

-- Recréer avec nouveaux paramètres
DROP INDEX idx_embedding ON documents;

CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (
    m = 24,                 -- Changé de 16 à 24
    ef_construction = 300,  -- Changé de 200 à 300
    metric = 'cosine'
)
ALGORITHM=INPLACE,
LOCK=NONE;

-- ⚠️ Peut prendre du temps sur grandes tables
-- Planifier en fenêtre de maintenance si possible
```

---

## Suppression d'index

### Syntaxes de suppression

```sql
-- DROP INDEX
DROP INDEX index_name ON table_name;

-- Exemples
DROP INDEX idx_category ON products;
DROP INDEX idx_email_unique ON users;

-- ALTER TABLE (peut supprimer plusieurs index)
ALTER TABLE products
DROP INDEX idx_category,
DROP INDEX idx_price;

-- Supprimer PRIMARY KEY (attention !)
ALTER TABLE temp_table DROP PRIMARY KEY;
```

### Identifier les index à supprimer

```sql
-- 1. Index inutilisés (Performance Schema)
SELECT
    object_schema AS db,
    object_name AS table_name,
    index_name,
    count_star AS accesses
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'mydb'
AND index_name IS NOT NULL
AND index_name != 'PRIMARY'
AND count_star = 0
ORDER BY object_name;

-- 2. Index redondants
-- idx(a) est redondant si idx(a,b) existe
SELECT
    t.table_name,
    t.index_name,
    GROUP_CONCAT(t.column_name ORDER BY t.seq_in_index) AS columns
FROM information_schema.STATISTICS t
WHERE t.table_schema = 'mydb'
GROUP BY t.table_name, t.index_name
ORDER BY t.table_name, columns;

-- Analyser manuellement pour détecter redondance
-- idx_customer (customer_id)
-- idx_customer_date (customer_id, order_date)
-- → idx_customer est redondant !

-- 3. Index dupliqués (exactement identiques)
SELECT
    t1.table_name,
    t1.index_name AS index1,
    t2.index_name AS index2,
    GROUP_CONCAT(t1.column_name) AS columns
FROM information_schema.STATISTICS t1
JOIN information_schema.STATISTICS t2
    ON t1.table_schema = t2.table_schema
    AND t1.table_name = t2.table_name
    AND t1.column_name = t2.column_name
    AND t1.seq_in_index = t2.seq_in_index
    AND t1.index_name < t2.index_name
WHERE t1.table_schema = 'mydb'
GROUP BY t1.table_name, t1.index_name, t2.index_name;
```

### Procédure de suppression sécurisée

```sql
-- Étape 1 : Identifier l'index candidat
SELECT * FROM information_schema.STATISTICS
WHERE table_schema = 'mydb'
AND table_name = 'orders'
AND index_name = 'idx_old';

-- Étape 2 : Rendre invisible pendant 1-2 semaines
ALTER TABLE orders ALTER INDEX idx_old INVISIBLE;

-- Étape 3 : Monitorer l'impact
-- Si aucune dégradation après 2 semaines → supprimer

-- Étape 4 : Suppression définitive
DROP INDEX idx_old ON orders;

-- Rollback possible si problème détecté :
CREATE INDEX idx_old ON orders(column_name);
```

---

## Maintenance des index

### ANALYZE TABLE : Mise à jour des statistiques

```sql
-- Met à jour les statistiques de cardinalité
ANALYZE TABLE orders;

-- Plusieurs tables en une commande
ANALYZE TABLE orders, customers, products;

-- Vérifier les statistiques
SHOW INDEX FROM orders;
-- Colonne Cardinality : nombre de valeurs uniques estimées

-- Quand exécuter ANALYZE ?
-- - Après imports massifs
-- - Après suppressions importantes
-- - Si les plans d'exécution semblent sous-optimaux
-- - Périodiquement (hebdomadaire/mensuel)
```

### OPTIMIZE TABLE : Défragmentation

```sql
-- Défragmente et reconstruit la table + tous les index
OPTIMIZE TABLE orders;

-- ⚠️ Attention
-- - Verrouille la table (peut être long)
-- - Nécessite espace disque libre (2x taille table)
-- - Équivalent à ALTER TABLE ... ENGINE=InnoDB

-- Alternative Online (InnoDB)
ALTER TABLE orders ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;

-- Quand exécuter OPTIMIZE ?
-- - Fragmentation > 20-30%
-- - Après beaucoup de DELETE/UPDATE
-- - Performances dégradées
```

**Vérifier la fragmentation** :

```sql
-- Taux de fragmentation
SELECT
    table_name,
    data_length AS data_size,
    data_free AS wasted_space,
    ROUND(data_free / (data_length + data_free) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE table_schema = 'mydb'
AND engine = 'InnoDB'
AND data_length > 0
ORDER BY fragmentation_pct DESC;

-- Fragmentation > 30% : OPTIMIZE recommandé
```

### CHECK TABLE et REPAIR TABLE

```sql
-- Vérifier l'intégrité de la table et des index
CHECK TABLE orders;

-- Options
CHECK TABLE orders QUICK;      -- Rapide, scan partiel
CHECK TABLE orders FAST;       -- Seulement si non fermée proprement
CHECK TABLE orders CHANGED;    -- Si modifiée depuis dernier check
CHECK TABLE orders EXTENDED;   -- Vérification approfondie (lent)

-- Réparer une table corrompue (MyISAM principalement)
REPAIR TABLE orders;
REPAIR TABLE orders QUICK;     -- Réparation rapide
REPAIR TABLE orders EXTENDED;  -- Réparation complète

-- ⚠️ InnoDB
-- CHECK TABLE : fonctionne
-- REPAIR TABLE : pas supporté (utiliser mariabackup restore)
```

### Maintenance des index Full-Text

```sql
-- Optimiser les index Full-Text
OPTIMIZE TABLE articles;

-- Purger les caches Full-Text (InnoDB)
SET GLOBAL innodb_ft_aux_table = 'mydb/articles';
SELECT * FROM information_schema.INNODB_FT_INDEX_CACHE;
-- Voir les tokens en cache

-- Forcer la mise à jour de l'index Full-Text
ALTER TABLE articles DROP INDEX idx_content;
ALTER TABLE articles ADD FULLTEXT INDEX idx_content (title, content);
```

### Maintenance des index VECTOR 🆕

```sql
-- Les index HNSW se maintiennent automatiquement
-- Mais peuvent nécessiter rebuild si :
-- - Changement de paramètres souhaité
-- - Fragmentation importante
-- - Performances dégradées

-- Vérifier la taille de l'index
SELECT
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb'
AND table_name = 'documents'
AND index_name = 'idx_embedding'
AND stat_name = 'size';

-- Rebuild si nécessaire
DROP INDEX idx_embedding ON documents;
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (m=16, ef_construction=200, metric='cosine')
ALGORITHM=INPLACE, LOCK=NONE;
```

---

## Stratégies de maintenance automatisée

### Script de maintenance périodique

```sql
-- Procédure stockée pour maintenance hebdomadaire
DELIMITER //

CREATE PROCEDURE maintain_indexes(IN schema_name VARCHAR(64))
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE tbl VARCHAR(64);
    DECLARE frag DECIMAL(5,2);

    -- Curseur pour tables fragmentées
    DECLARE cur CURSOR FOR
        SELECT table_name,
               ROUND(data_free / (data_length + data_free) * 100, 2) AS fragmentation
        FROM information_schema.TABLES
        WHERE table_schema = schema_name
        AND engine = 'InnoDB'
        AND data_length > 0
        AND data_free / (data_length + data_free) > 0.20;  -- > 20%

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    -- Mise à jour des statistiques pour toutes les tables
    FOR tbl IN (SELECT table_name FROM information_schema.TABLES
                WHERE table_schema = schema_name)
    DO
        SET @sql = CONCAT('ANALYZE TABLE ', schema_name, '.', tbl);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END FOR;

    -- Optimisation des tables fragmentées
    OPEN cur;
    read_loop: LOOP
        FETCH cur INTO tbl, frag;
        IF done THEN
            LEAVE read_loop;
        END IF;

        SET @sql = CONCAT('OPTIMIZE TABLE ', schema_name, '.', tbl);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;

        SELECT CONCAT('Optimized ', tbl, ' (fragmentation: ', frag, '%)') AS status;
    END LOOP;
    CLOSE cur;
END//

DELIMITER ;

-- Exécuter manuellement
CALL maintain_indexes('mydb');

-- Ou planifier avec Event Scheduler
CREATE EVENT weekly_maintenance
ON SCHEDULE EVERY 1 WEEK
STARTS '2025-01-06 02:00:00'  -- Dimanche 2h du matin
DO CALL maintain_indexes('mydb');
```

### Monitoring automatique des index

```sql
-- Vue pour index inutilisés
CREATE VIEW v_unused_indexes AS
SELECT
    object_schema AS database_name,
    object_name AS table_name,
    index_name,
    'Unused - Consider dropping' AS recommendation
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema NOT IN ('mysql', 'performance_schema', 'information_schema')
AND index_name IS NOT NULL
AND index_name != 'PRIMARY'
AND count_star = 0;

-- Vue pour fragmentation
CREATE VIEW v_fragmented_tables AS
SELECT
    table_schema,
    table_name,
    ROUND(data_free / (data_length + data_free) * 100, 2) AS fragmentation_pct,
    CASE
        WHEN data_free / (data_length + data_free) > 0.30 THEN 'Critical - OPTIMIZE needed'
        WHEN data_free / (data_length + data_free) > 0.20 THEN 'High - OPTIMIZE recommended'
        WHEN data_free / (data_length + data_free) > 0.10 THEN 'Moderate - Monitor'
        ELSE 'Good'
    END AS status
FROM information_schema.TABLES
WHERE engine = 'InnoDB'
AND data_length > 0
ORDER BY fragmentation_pct DESC;

-- Requêtes de monitoring
SELECT * FROM v_unused_indexes;
SELECT * FROM v_fragmented_tables WHERE fragmentation_pct > 20;
```

---

## Bonnes pratiques de gestion d'index

### 1. Planification avant création

```sql
-- ❌ Mauvais : Créer des index au hasard
CREATE INDEX idx_col1 ON table(col1);
CREATE INDEX idx_col2 ON table(col2);
CREATE INDEX idx_col3 ON table(col3);
-- Résultat : Trop d'index, performances d'écriture dégradées

-- ✅ Bon : Analyser les requêtes d'abord
-- 1. Identifier les requêtes lentes
SELECT * FROM mysql.slow_log
WHERE query_time > 1
ORDER BY query_time DESC
LIMIT 20;

-- 2. Analyser avec EXPLAIN
EXPLAIN SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending';

-- 3. Créer un index ciblé
CREATE INDEX idx_customer_status ON orders(customer_id, status);
```

### 2. Nommer les index de façon cohérente

```sql
-- Convention de nommage
-- idx_{table}_{colonne(s)}
-- idx_{type}_{table}_{colonne(s)}

-- ✅ Bonnes pratiques
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
CREATE FULLTEXT INDEX idx_ft_articles_content ON articles(title, content);
CREATE SPATIAL INDEX idx_sp_stores_location ON stores(location);
CREATE INDEX idx_vec_docs_embedding ON documents(embedding) USING HNSW;

-- ❌ À éviter
CREATE INDEX index1 ON users(email);  -- Nom non descriptif
CREATE INDEX email ON users(email);   -- Confusion avec nom de colonne
```

### 3. Limiter le nombre d'index

```sql
-- Règle empirique : 3-5 index par table OLTP
-- Plus d'index = écritures plus lentes

-- Exemple équilibré
CREATE TABLE orders (
    order_id INT PRIMARY KEY,           -- Index 1 (clustered)
    customer_id INT,
    status VARCHAR(20),
    order_date DATETIME,
    total DECIMAL(10,2),

    INDEX idx_customer (customer_id),   -- Index 2 (FK)
    INDEX idx_status_date (status, order_date),  -- Index 3 (requêtes fréquentes)
    INDEX idx_date (order_date)         -- Index 4 (rapports)
) ENGINE=InnoDB;

-- 4 index total = Bon équilibre lecture/écriture
```

### 4. Tester l'impact avant déploiement

```sql
-- 1. Créer l'index en invisible
CREATE INDEX idx_test ON large_table(column) INVISIBLE;

-- 2. Activer pour tests
SET optimizer_switch='use_invisible_indexes=on';

-- 3. Tester les requêtes
EXPLAIN SELECT ... ;
-- Mesurer les performances

-- 4. Décision
-- Si amélioration > 30% : ALTER INDEX VISIBLE
-- Sinon : DROP INDEX
```

### 5. Documenter les index

```sql
-- Utiliser les commentaires de table/colonnes
ALTER TABLE orders COMMENT = 'Table commandes - Index:
idx_customer pour recherches par client,
idx_status_date pour dashboard admin,
idx_date pour rapports mensuels';

-- Maintenir un document de référence
-- Tableau : Nom Index | Colonnes | Justification | Date création
```

### 6. Maintenance régulière planifiée

```
Calendrier recommandé :

Quotidien :
- Monitoring utilisation CPU/IO
- Vérification slow query log

Hebdomadaire :
- ANALYZE TABLE sur tables actives
- Revue des index inutilisés (Performance Schema)

Mensuel :
- OPTIMIZE TABLE sur tables fragmentées
- Revue de la stratégie d'indexation
- Ajustement des index selon évolution usage

Trimestriel :
- Audit complet de tous les index
- Benchmark des requêtes critiques
- Planification des optimisations
```

---

## Cas pratiques de gestion d'index

### Cas 1 : Ajout d'index sur table de production volumineuse

```sql
-- Contexte : Table orders avec 50 millions de lignes
-- Besoin : Index sur customer_id pour améliorer les requêtes

-- Étape 1 : Vérifier l'espace disque disponible
SELECT
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024 / 1024, 2) AS size_gb
FROM information_schema.TABLES
WHERE table_name = 'orders';
-- Résultat : 45 GB

-- Besoin estimé : ~5-10 GB pour nouvel index
-- Vérifier : df -h sur serveur

-- Étape 2 : Création Online pendant heures creuses
-- Lancer à 2h du matin
CREATE INDEX idx_customer ON orders(customer_id)
ALGORITHM=INPLACE,
LOCK=NONE;

-- Étape 3 : Monitoring pendant création
SELECT
    STAGE,
    WORK_COMPLETED,
    WORK_ESTIMATED,
    ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 2) AS progress_pct,
    TIME_ELAPSED_SECONDS / 60 AS elapsed_minutes
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE 'stage/innodb/alter%';

-- Étape 4 : Validation post-création
SHOW INDEX FROM orders WHERE key_name = 'idx_customer';
ANALYZE TABLE orders;

-- Étape 5 : Vérifier utilisation
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
-- type: ref, key: idx_customer ← OK !
```

### Cas 2 : Migration d'index pour améliorer les performances

```sql
-- Contexte : Index existant peu optimal
-- Avant : idx_status (status)
-- Requêtes fréquentes filtrent sur status ET order_date

-- Étape 1 : Créer nouvel index optimisé
CREATE INDEX idx_status_date_new ON orders(status, order_date)
ALGORITHM=INPLACE, LOCK=NONE;

-- Étape 2 : Tester avec les deux index
-- Les deux index coexistent, pas de problème

SET optimizer_switch='use_invisible_indexes=on';
EXPLAIN SELECT * FROM orders
WHERE status = 'pending'
ORDER BY order_date DESC
LIMIT 100;

-- Vérifier que idx_status_date_new est utilisé

-- Étape 3 : Rendre ancien index invisible
ALTER TABLE orders ALTER INDEX idx_status INVISIBLE;

-- Étape 4 : Monitoring 1-2 semaines
-- Vérifier qu'aucune régression

-- Étape 5 : Supprimer ancien index
DROP INDEX idx_status ON orders;
```

### Cas 3 : Correction d'un problème de fragmentation

```sql
-- Contexte : Table avec 40% de fragmentation après purge annuelle

-- Étape 1 : Identifier
SELECT
    table_name,
    ROUND(data_free / (data_length + data_free) * 100, 2) AS frag_pct,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb,
    ROUND(data_free / 1024 / 1024, 2) AS wasted_mb
FROM information_schema.TABLES
WHERE table_name = 'logs';

-- Résultat :
-- frag_pct: 42%
-- total_mb: 15000 MB
-- wasted_mb: 6300 MB (!)

-- Étape 2 : Planifier OPTIMIZE en fenêtre de maintenance
-- Samedi 3h du matin, faible trafic

-- Étape 3 : Exécuter OPTIMIZE
OPTIMIZE TABLE logs;
-- Durée : ~2 heures pour 15 GB

-- Étape 4 : Vérifier l'amélioration
SELECT
    ROUND(data_free / (data_length + data_free) * 100, 2) AS frag_pct,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
WHERE table_name = 'logs';

-- Résultat :
-- frag_pct: 2%
-- total_mb: 8700 MB (gain de 6.3 GB !)
```

### Cas 4 : Déploiement d'index VECTOR sur chatbot 🆕

```sql
-- Contexte : Migration knowledge base vers recherche sémantique
-- Table : 500 000 documents, embedding 1536 dimensions

-- Étape 1 : Préparer la configuration
SET GLOBAL innodb_vector_cache_size = 2147483648;  -- 2 GB
SET GLOBAL innodb_vector_threads = 8;
SET GLOBAL innodb_buffer_pool_size = 17179869184; -- 16 GB

-- Étape 2 : Créer l'index (heure creuse)
CREATE INDEX idx_embedding ON knowledge_base(embedding)
USING HNSW
WITH (
    m = 16,
    ef_construction = 200,
    metric = 'cosine'
)
ALGORITHM=INPLACE,
LOCK=NONE;

-- Durée estimée : ~5-8 minutes

-- Étape 3 : Tester les recherches
SET @query_vec = VEC_FromText('[...]');
SET @@session.hnsw_ef_search = 100;

SELECT id, title,
       VEC_DISTANCE_COSINE(embedding, @query_vec) AS similarity
FROM knowledge_base
ORDER BY similarity ASC
LIMIT 10;

-- Mesurer le temps : < 10 ms = OK

-- Étape 4 : Ajuster ef_search selon besoin
-- ef=50  : 3-5 ms,  recall ~88%
-- ef=100 : 5-10 ms, recall ~94% ← Production
-- ef=200 : 10-20 ms, recall ~97% ← Si précision critique

-- Étape 5 : Monitoring continu
SELECT
    COUNT(*) as total_searches,
    AVG(query_time) as avg_time_sec
FROM mysql.slow_log
WHERE sql_text LIKE '%VEC_DISTANCE%'
AND start_time > DATE_SUB(NOW(), INTERVAL 24 HOUR);
```

---

## Troubleshooting : Problèmes courants

### Problème 1 : Création d'index très lente

```sql
-- Symptômes : Index prend des heures à créer

-- Diagnostic :
-- 1. Vérifier I/O disque
SHOW ENGINE INNODB STATUS\G
-- Section "BACKGROUND THREAD"

-- 2. Vérifier innodb_io_capacity
SHOW VARIABLES LIKE 'innodb_io_capacity%';

-- Solution : Augmenter temporairement
SET GLOBAL innodb_io_capacity = 2000;
SET GLOBAL innodb_io_capacity_max = 4000;

-- Relancer la création
CREATE INDEX idx_name ON table(column)
ALGORITHM=INPLACE, LOCK=NONE;
```

### Problème 2 : Index non utilisé par l'optimiseur

```sql
-- Symptômes : EXPLAIN n'utilise pas l'index créé

-- Diagnostic :
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
-- possible_keys: idx_customer
-- key: NULL ← Index pas utilisé !

-- Causes possibles :

-- 1. Statistiques obsolètes
ANALYZE TABLE orders;

-- 2. Index moins sélectif que full scan
-- Vérifier cardinalité
SHOW INDEX FROM orders WHERE key_name = 'idx_customer';

-- 3. Fonction sur colonne indexée
-- ❌ Mauvais
WHERE YEAR(order_date) = 2025

-- ✅ Bon
WHERE order_date >= '2025-01-01' AND order_date < '2026-01-01'

-- 4. Type de données incompatible
-- ❌ customer_id INT dans table, VARCHAR dans query
WHERE customer_id = '123'

-- ✅ Type cohérent
WHERE customer_id = 123
```

### Problème 3 : Erreur lors de création d'index Spatial

```sql
-- Erreur : "All spatial columns must be NOT NULL"

-- ❌ Échoue
CREATE SPATIAL INDEX idx_loc ON stores(location);
-- ERROR 1252

-- Diagnostic
DESC stores;
-- location POINT DEFAULT NULL ← Problème !

-- Solution
ALTER TABLE stores MODIFY COLUMN location POINT NOT NULL;
CREATE SPATIAL INDEX idx_loc ON stores(location);
```

### Problème 4 : Erreur HNSW dimensions incompatibles 🆕

```sql
-- Erreur : Vector dimension mismatch

-- ❌ Échoue
INSERT INTO docs VALUES (1, 'text', VEC_FromText('[0.1, 0.2, 0.3]'));
-- ERROR: Expected 1536 dimensions, got 3

-- Diagnostic
DESC docs;
-- embedding VECTOR(1536)

-- Solution : Vérifier dimensions
SELECT VEC_Dimension(VEC_FromText('[0.1, 0.2, 0.3]'));
-- 3 dimensions

-- Utiliser embedding avec bonnes dimensions
-- Généré par OpenAI text-embedding-3-small (1536 dims)
```

---

## ✅ Points clés à retenir

- 📝 **3 syntaxes** : CREATE INDEX, ALTER TABLE, CREATE TABLE
- 🔄 **Online DDL** : ALGORITHM=INPLACE, LOCK=NONE pour production
- 🎯 **Types variés** : B-Tree, Hash, Full-Text, Spatial, VECTOR 🆕
- 👁️ **Index invisibles** : Tester avant activer (INVISIBLE/VISIBLE)
- 🔧 **Maintenance** : ANALYZE (stats), OPTIMIZE (défrag), CHECK (intégrité)
- 📊 **Monitoring** : Performance Schema pour détecter index inutilisés
- 🚫 **Supprimer** : Index redondants, dupliqués, jamais utilisés
- ⚙️ **HNSW paramètres** 🆕 : m, ef_construction (création), ef_search (runtime)
- 📏 **Limiter** : 3-5 index par table OLTP pour équilibre lecture/écriture
- 🔍 **Ordre important** : Index composites (leftmost prefix rule)
- 📅 **Calendrier** : Maintenance hebdomadaire (ANALYZE), mensuelle (OPTIMIZE)
- 💡 **Nommage cohérent** : idx_{table}_{colonnes} ou idx_{type}_{table}_{colonnes}

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 CREATE INDEX](https://mariadb.com/kb/en/create-index/)
- [📖 ALTER TABLE](https://mariadb.com/kb/en/alter-table/)
- [📖 Online DDL](https://mariadb.com/kb/en/innodb-online-ddl/)
- [📖 DROP INDEX](https://mariadb.com/kb/en/drop-index/)
- [📖 OPTIMIZE TABLE](https://mariadb.com/kb/en/optimize-table/)
- [📖 ANALYZE TABLE](https://mariadb.com/kb/en/analyze-table/)
- [📖 Index Maintenance](https://mariadb.com/kb/en/index-maintenance/)

### Outils de gestion

- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)
  - pt-online-schema-change
  - pt-index-usage
  - pt-duplicate-key-checker
- [gh-ost](https://github.com/github/gh-ost) - GitHub's online schema migration

### Articles techniques

- [InnoDB Online DDL Performance](https://www.percona.com/blog/)
- [Index Management Best Practices](https://mariadb.org/best-practices/)
- [HNSW Index Tuning](https://mariadb.org/vector-index-tuning/) 🆕

---

## ➡️ Section suivante

**[5.5 Stratégies d'indexation](./05-strategies-indexation.md)**

Approfondissez les stratégies d'indexation avancées : index sur colonnes fréquemment filtrées, index sur clés étrangères, optimisation pour ORDER BY et GROUP BY, choix entre index simple et composite, et patterns d'indexation pour différents types de requêtes et workloads.

---


⏭️ [Stratégies d'indexation](/05-index-et-performance/05-strategies-indexation.md)

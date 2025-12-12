🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 Stratégies d'indexation

> **Niveau** : Intermédiaire
> **Durée estimée** : 2 heures
> **Prérequis** : Section 5.1 à 5.4 (Types d'index et fonctionnement B-Tree)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Déterminer quand créer un index et quel type utiliser
- Analyser les patterns d'accès aux données pour optimiser l'indexation
- Appliquer les meilleures pratiques d'indexation selon les cas d'usage
- Équilibrer les bénéfices et coûts des index en production
- Utiliser une méthodologie pour identifier les besoins d'indexation

---

## Introduction

L'indexation est un art d'équilibre : trop peu d'index ralentit les requêtes de lecture, trop d'index dégrade les performances d'écriture et consomme inutilement de l'espace disque. Une stratégie d'indexation efficace repose sur la compréhension des **patterns d'accès aux données** et l'analyse méthodique des requêtes réelles de votre application.

Dans cette section, nous explorerons les **principes fondamentaux** qui guident la création d'index pertinents, ainsi que les **méthodologies d'analyse** pour identifier les opportunités d'optimisation.

💡 **Principe clé** : Un index doit être créé en fonction des **requêtes réelles** de votre application, pas de manière spéculative.

---

## Méthodologie d'analyse des besoins d'indexation

### 1. Identifier les requêtes critiques

La première étape consiste à identifier les requêtes qui ont le plus d'impact sur votre application :

```sql
-- Activer le slow query log pour capturer les requêtes lentes
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- Requêtes > 1 seconde
SET GLOBAL log_queries_not_using_indexes = 'ON'; -- Capturer les full scans

-- Analyser le fichier slow query log avec pt-query-digest (Percona Toolkit)
-- $ pt-query-digest /var/log/mysql/slow.log
```

**Critères de priorisation** :
- **Fréquence d'exécution** : Requêtes exécutées des milliers de fois par jour
- **Temps d'exécution** : Requêtes prenant plusieurs secondes
- **Impact métier** : Requêtes critiques pour l'expérience utilisateur
- **Full table scans** : Requêtes qui parcourent des tables entières

### 2. Analyser les plans d'exécution

Pour chaque requête critique identifiée, utilisez `EXPLAIN` pour comprendre comment MariaDB l'exécute :

```sql
-- Analyse d'une requête typique de recherche utilisateur
EXPLAIN
SELECT u.user_id, u.username, u.email, p.profile_picture
FROM users u
LEFT JOIN user_profiles p ON u.user_id = p.user_id
WHERE u.status = 'active'
  AND u.country = 'FR'
  AND u.created_at > '2024-01-01'
ORDER BY u.last_login DESC
LIMIT 20;
```

**Indicateurs d'alerte dans EXPLAIN** :
- `type: ALL` → Full table scan (critique sur grandes tables)
- `rows: 1000000` → Trop de lignes examinées
- `Extra: Using filesort` → Tri en mémoire/disque (peut être optimisé)
- `Extra: Using temporary` → Table temporaire créée
- `key: NULL` → Aucun index utilisé

💡 **Astuce** : `EXPLAIN ANALYZE` (MariaDB 10.6+) fournit des **statistiques réelles** d'exécution, pas seulement des estimations.

```sql
-- EXPLAIN ANALYZE donne les temps réels d'exécution
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = 12345
AND order_date BETWEEN '2024-01-01' AND '2024-12-31';
```

### 3. Calculer la sélectivité des colonnes

La **sélectivité** d'une colonne détermine son efficacité pour un index :

```sql
-- Calcul de la sélectivité : ratio valeurs distinctes / total lignes
SELECT
    COUNT(DISTINCT country) / COUNT(*) AS selectivity_country,
    COUNT(DISTINCT status) / COUNT(*) AS selectivity_status,
    COUNT(DISTINCT email) / COUNT(*) AS selectivity_email
FROM users;

-- Résultat exemple :
-- selectivity_country: 0.015  (195 pays / ~13000 lignes = faible sélectivité)
-- selectivity_status:  0.002  (3 statuts / ~13000 lignes = très faible)
-- selectivity_email:   0.999  (emails uniques = excellente sélectivité)
```

**Règle empirique** :
- **Sélectivité > 0.5** → Excellent candidat pour un index
- **Sélectivité 0.1-0.5** → Index probablement utile selon la volumétrie
- **Sélectivité < 0.05** → Index peu efficace (optimizer peut choisir full scan)

⚠️ **Attention** : Une colonne à faible sélectivité (ex: `status` avec 3 valeurs) peut quand même bénéficier d'un index dans un **index composite** ou si elle filtre significativement le dataset.

---

## Principes fondamentaux d'indexation

### Principe 1 : Indexer les colonnes des clauses WHERE

Les colonnes utilisées pour **filtrer** les données sont les candidates prioritaires :

```sql
-- Requête fréquente : recherche par email
SELECT * FROM users WHERE email = 'john@example.com';

-- Index recommandé
CREATE INDEX idx_users_email ON users(email);
```

**Cas particulier** : opérateurs qui **n'utilisent pas les index** :
- `NOT IN`, `<>`, `!=` → Full scan probable
- `LIKE '%pattern'` → Wildcard au début empêche l'utilisation de l'index
- Fonctions sur colonnes indexées : `WHERE YEAR(created_at) = 2024`

```sql
-- ❌ Mauvais : fonction sur colonne indexée
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- ✅ Bon : condition directe sur la colonne
SELECT * FROM orders
WHERE order_date >= '2024-01-01'
  AND order_date < '2025-01-01';
```

### Principe 2 : Ordre des colonnes dans les index composites

Pour les **index multi-colonnes**, l'ordre est crucial. Règle générale :

1. **Colonnes d'égalité** (`=`) en premier
2. **Colonnes de plage** (`>`, `<`, `BETWEEN`) ensuite
3. **Colonnes d'ordre** (`ORDER BY`) en dernier

```sql
-- Requête analysée
SELECT * FROM orders
WHERE customer_id = 123        -- Égalité
  AND status = 'pending'       -- Égalité
  AND order_date > '2024-01-01' -- Plage
ORDER BY order_date DESC;

-- Index optimal
CREATE INDEX idx_orders_composite
ON orders(customer_id, status, order_date);
```

**Explication** :
- `customer_id` en premier : filtre le plus sélectif (réduit drastiquement le dataset)
- `status` en second : affine encore le filtrage
- `order_date` en dernier : utilisé pour la plage ET l'ordre

💡 **Règle du préfixe gauche** : Un index composite peut servir plusieurs requêtes :
```sql
-- Un index (col_a, col_b, col_c) peut être utilisé pour :
WHERE col_a = 10;                      -- ✅ Utilise l'index
WHERE col_a = 10 AND col_b = 20;      -- ✅ Utilise l'index
WHERE col_a = 10 AND col_b = 20 AND col_c = 30; -- ✅ Utilise l'index complet

-- Mais PAS pour :
WHERE col_b = 20;                      -- ❌ Ne peut pas utiliser l'index
WHERE col_c = 30;                      -- ❌ Ne peut pas utiliser l'index
WHERE col_b = 20 AND col_c = 30;      -- ❌ Ne peut pas utiliser l'index
```

### Principe 3 : Index covering (couvrants)

Un **index covering** contient **toutes les colonnes** nécessaires à la requête, évitant ainsi la lecture de la table :

```sql
-- Requête fréquente
SELECT user_id, username, email
FROM users
WHERE status = 'active';

-- Index covering : contient toutes les colonnes sélectionnées
CREATE INDEX idx_users_covering
ON users(status, user_id, username, email);

-- Vérification dans EXPLAIN
EXPLAIN SELECT user_id, username, email FROM users WHERE status = 'active';
-- Extra: Using index  ← Indique un index covering (index-only scan)
```

**Avantages** :
- ✅ **Performance maximale** : lecture uniquement de l'index (plus compact que la table)
- ✅ **Moins d'I/O** : pas d'accès à la table de données
- ✅ **Buffer pool efficace** : plus de données d'index en cache

⚠️ **Compromis** : index plus volumineux, coût d'écriture accru.

### Principe 4 : Équilibrer lecture vs écriture

Chaque index a un **coût en écriture** (INSERT, UPDATE, DELETE) :

```sql
-- Scénario : table de logs avec millions d'écritures/jour
CREATE TABLE application_logs (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME NOT NULL,
    level VARCHAR(10),
    message TEXT,
    user_id INT
);

-- ❌ Mauvais : trop d'index pour une table en écriture intensive
CREATE INDEX idx_timestamp ON application_logs(timestamp);
CREATE INDEX idx_level ON application_logs(level);
CREATE INDEX idx_user_id ON application_logs(user_id);
CREATE INDEX idx_level_timestamp ON application_logs(level, timestamp);

-- ✅ Bon : index ciblés selon les requêtes réelles
CREATE INDEX idx_logs_query ON application_logs(level, timestamp, user_id);
-- Un seul index composite répond à plusieurs patterns de requêtes
```

**Recommandations** :
- **Tables OLTP** (50%+ d'écritures) : limiter à 3-5 index maximum
- **Tables analytiques** (95%+ de lectures) : peut supporter 10-15 index
- **Tables de logs** : privilégier partitionnement sur indexation massive

---

## Stratégies par type de requête

### Requêtes de recherche exacte

```sql
-- Pattern : recherche par identifiant unique
SELECT * FROM products WHERE sku = 'PROD-12345';

-- Stratégie : index unique sur colonne sélective
CREATE UNIQUE INDEX idx_products_sku ON products(sku);
```

### Requêtes avec plages de dates

```sql
-- Pattern : rapports sur période
SELECT SUM(amount) FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND status = 'completed';

-- Stratégie : index composite avec égalité avant plage
CREATE INDEX idx_orders_reporting
ON orders(status, order_date);
```

### Requêtes avec jointures

```sql
-- Pattern : jointure fréquente
SELECT o.order_id, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending';

-- Stratégie :
-- 1. Index sur clé étrangère (côté orders)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- 2. Index composite pour filtrage
CREATE INDEX idx_orders_status_customer
ON orders(status, customer_id);

-- 3. Primary key déjà présent sur customers(customer_id)
```

💡 **Règle** : Toujours indexer les **clés étrangères** utilisées dans les jointures.

### Requêtes avec ORDER BY / GROUP BY

```sql
-- Pattern : liste paginée triée
SELECT * FROM products
WHERE category_id = 5
ORDER BY price DESC
LIMIT 20 OFFSET 100;

-- Stratégie : index composite permettant le tri
CREATE INDEX idx_products_category_price
ON products(category_id, price);
-- Évite "Using filesort" dans EXPLAIN
```

### Requêtes full-text

```sql
-- Pattern : recherche textuelle
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('MariaDB performance' IN NATURAL LANGUAGE MODE);

-- Stratégie : index FULLTEXT
CREATE FULLTEXT INDEX idx_articles_fulltext
ON articles(title, content);
```

**Cas d'usage** :
- ✅ Recherche de mots-clés dans textes longs
- ✅ Moteurs de recherche internes
- ❌ Requêtes LIKE simples (préférer index B-Tree)

### Requêtes géospatiales

```sql
-- Pattern : recherche par proximité
SELECT * FROM stores
WHERE ST_Distance_Sphere(
    location,
    POINT(2.3522, 48.8566) -- Paris
) < 5000; -- 5km

-- Stratégie : index SPATIAL
CREATE SPATIAL INDEX idx_stores_location ON stores(location);
```

### 🆕 Requêtes de recherche vectorielle (MariaDB 11.8)

```sql
-- Pattern : recherche sémantique par similarité
SELECT product_id, name,
       VEC_DISTANCE_COSINE(embedding, @query_vector) AS similarity
FROM products
ORDER BY similarity
LIMIT 10;

-- Stratégie : index HNSW pour vecteurs
CREATE INDEX idx_products_embedding
ON products(embedding)
USING HNSW;
```

**Cas d'usage MariaDB Vector** :
- ✅ Recherche sémantique de produits
- ✅ RAG (Retrieval-Augmented Generation) pour LLMs
- ✅ Systèmes de recommandation basés sur embeddings
- ✅ Détection d'anomalies par distance vectorielle

💡 **Note** : Les index HNSW utilisent des optimisations SIMD (AVX2, AVX512) pour des performances maximales sur architectures modernes.

---

## Éviter les anti-patterns

### Anti-pattern 1 : Index sur colonnes à faible cardinalité seules

```sql
-- ❌ Mauvais : colonne avec seulement 2-3 valeurs distinctes
CREATE INDEX idx_users_gender ON users(gender); -- 'M', 'F', 'Other'
CREATE INDEX idx_orders_status ON orders(status); -- 'pending', 'completed', 'cancelled'

-- L'optimizer MariaDB préférera souvent un full scan
```

**Solution** : Utiliser ces colonnes dans des **index composites** où elles filtrent en amont :
```sql
-- ✅ Bon : faible cardinalité en premier d'un composite
CREATE INDEX idx_orders_composite
ON orders(status, customer_id, order_date);
```

### Anti-pattern 2 : Index redondants

```sql
-- ❌ Mauvais : index redondants
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_username ON users(email, username);
-- Le premier index est redondant (règle du préfixe gauche)

-- ✅ Bon : garder uniquement le plus complet
CREATE INDEX idx_users_email_username ON users(email, username);
-- Peut servir pour WHERE email = ... ET WHERE email = ... AND username = ...
```

**Détection des index redondants** :
```sql
-- Requête pour identifier les doublons potentiels
SELECT
    TABLE_NAME,
    INDEX_NAME,
    GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS columns
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
GROUP BY TABLE_NAME, INDEX_NAME
ORDER BY TABLE_NAME, columns;
```

### Anti-pattern 3 : Indexer sans analyser

```sql
-- ❌ Mauvais : création d'index "au cas où"
CREATE INDEX idx_users_lastname ON users(lastname);
CREATE INDEX idx_users_firstname ON users(firstname);
CREATE INDEX idx_users_phone ON users(phone);
-- Sans analyse des requêtes réelles !

-- ✅ Bon : analyser d'abord les requêtes, puis créer des index ciblés
-- Exemple : si 80% des recherches sont par email, commencer par là
CREATE INDEX idx_users_email ON users(email);
```

💡 **Méthodologie** :
1. Collecter les requêtes réelles (slow query log, APM)
2. Analyser les patterns d'accès
3. Créer des index pour les 20% de requêtes qui représentent 80% de la charge
4. Mesurer l'impact avant/après

### Anti-pattern 4 : Fonctions sur colonnes indexées

```sql
-- ❌ Mauvais : fonction empêche l'utilisation de l'index
CREATE INDEX idx_users_email ON users(email);

SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
-- Index non utilisé !

-- ✅ Solution 1 : Colonne générée (MariaDB 10.2+)
ALTER TABLE users
ADD COLUMN email_lower VARCHAR(255)
AS (LOWER(email)) VIRTUAL;

CREATE INDEX idx_users_email_lower ON users(email_lower);

-- ✅ Solution 2 : Normaliser les données à l'insertion
-- Stocker toujours les emails en minuscules
```

---

## Analyse et validation des index

### Vérifier l'utilisation effective des index

```sql
-- 1. Vérifier le plan d'exécution
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- 2. Vérifier les statistiques d'utilisation (MariaDB 10.5+)
SELECT
    TABLE_NAME,
    INDEX_NAME,
    ROWS_READ
FROM sys.schema_index_statistics
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY ROWS_READ DESC;
```

### Identifier les index inutilisés

```sql
-- Index jamais utilisés depuis le dernier redémarrage
SELECT
    s.TABLE_SCHEMA,
    s.TABLE_NAME,
    s.INDEX_NAME,
    s.CARDINALITY
FROM INFORMATION_SCHEMA.STATISTICS s
LEFT JOIN sys.schema_index_statistics sis
    ON s.TABLE_SCHEMA = sis.TABLE_SCHEMA
    AND s.TABLE_NAME = sis.TABLE_NAME
    AND s.INDEX_NAME = sis.INDEX_NAME
WHERE s.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
    AND sis.INDEX_NAME IS NULL
    AND s.INDEX_NAME != 'PRIMARY'
ORDER BY s.TABLE_SCHEMA, s.TABLE_NAME, s.INDEX_NAME;
```

⚠️ **Attention** : Ne supprimez pas immédiatement un index inutilisé. Il peut servir pour :
- Des requêtes batch nocturnes
- Des rapports mensuels
- Des processus exceptionnels (migration, audit)

💡 **Bonne pratique** : Utiliser les **invisible indexes** (MariaDB 10.0+) pour tester l'impact de la suppression :

```sql
-- Rendre un index invisible (non utilisé par l'optimizer)
ALTER TABLE users ALTER INDEX idx_users_phone INVISIBLE;

-- Surveiller les performances pendant 1-2 semaines

-- Si OK, supprimer définitivement
DROP INDEX idx_users_phone ON users;

-- Sinon, rendre visible à nouveau
ALTER TABLE users ALTER INDEX idx_users_phone VISIBLE;
```

---

## Stratégies selon la volumétrie

### Petites tables (< 10 000 lignes)

```sql
-- Sur petites tables, l'index peut être contre-productif
-- Full scan en mémoire = plus rapide que parcours d'index

-- ✅ Indexer uniquement :
-- - Clés primaires et uniques (intégrité référentielle)
-- - Clés étrangères (jointures)
```

### Tables moyennes (10k - 1M lignes)

```sql
-- Stratégie équilibrée :
-- - Index sur colonnes fréquemment filtrées (WHERE)
-- - Index sur clés étrangères
-- - 1-2 index composites pour requêtes critiques
-- - Total : 3-6 index par table
```

### Grandes tables (> 1M lignes)

```sql
-- Stratégie agressive :
-- - Index covering pour requêtes principales
-- - Index composites optimisés (ordre crucial)
-- - Considérer le partitionnement en complément
-- - Total : 5-10 index par table maximum

-- Exemple : table orders avec 50M lignes
CREATE INDEX idx_orders_customer_status_date
ON orders(customer_id, status, order_date);

CREATE INDEX idx_orders_reporting
ON orders(order_date, status, total_amount);
```

### Tables très grandes (> 100M lignes)

```sql
-- Combiner indexation + partitionnement
CREATE TABLE logs (
    log_id BIGINT AUTO_INCREMENT,
    log_date DATE NOT NULL,
    level VARCHAR(10),
    message TEXT,
    PRIMARY KEY (log_id, log_date)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(log_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Index par partition (partition pruning)
CREATE INDEX idx_logs_level ON logs(level);
```

---

## Workflow de gestion des index

### 1. Phase d'audit initiale

```bash
# Utiliser pt-duplicate-key-checker (Percona Toolkit)
pt-duplicate-key-checker --host=localhost --user=admin --password=xxx

# Analyser les tables volumineuses
pt-table-checksum --host=localhost
```

### 2. Cycle d'optimisation continue

```sql
-- Étape 1 : Identifier les requêtes lentes (automatique)
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- Étape 2 : Analyser mensuellement avec pt-query-digest
-- $ pt-query-digest /var/log/mysql/slow.log --since '1 month ago'

-- Étape 3 : Tester les nouveaux index sur environnement de staging

-- Étape 4 : Déployer progressivement en production
CREATE INDEX idx_new ONLINE ON large_table(col1, col2);

-- Étape 5 : Monitorer l'impact (Performance Schema)
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
```

### 3. Maintenance régulière

```sql
-- Trimestre : Analyser les statistiques d'index
ANALYZE TABLE orders;
ANALYZE TABLE customers;

-- Semestre : Identifier et supprimer les index inutilisés
-- (Après analyse sur plusieurs mois)

-- Annuel : Audit complet de l'architecture d'indexation
```

---

## ✅ Points clés à retenir

- **Analyser avant d'indexer** : Slow query log + EXPLAIN sont vos meilleurs outils
- **La sélectivité compte** : Privilégiez les colonnes avec forte cardinalité
- **L'ordre des colonnes** dans un index composite est crucial (égalité → plage → tri)
- **Index covering = performance maximale** : Incluez toutes les colonnes SELECT
- **Équilibrer lecture/écriture** : Trop d'index nuit aux INSERT/UPDATE
- **Éviter les fonctions** sur colonnes indexées (utilisez colonnes générées)
- **Invisible indexes** : Testez l'impact avant de supprimer un index
- **🆕 MariaDB Vector** : Index HNSW pour recherche sémantique et IA

---

## 🔗 Ressources et références

- [📖 MariaDB Optimization and Indexes](https://mariadb.com/kb/en/optimization-and-indexes/)
- [📖 MariaDB EXPLAIN Documentation](https://mariadb.com/kb/en/explain/)
- [📖 Index Hints and Optimizer Control](https://mariadb.com/kb/en/index-hints-how-to-force-query-plans/)
- [🆕 MariaDB Vector Indexes](https://mariadb.com/kb/en/vector/)
- [🛠️ Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)
- [📊 sys Schema Documentation](https://mariadb.com/kb/en/sys-schema/)

---

## ➡️ Section suivante

**5.5.1 Index sur colonnes fréquemment filtrées** : Approfondir les stratégies d'indexation pour les clauses WHERE avec exemples concrets par type de données et patterns de requêtes.

⏭️ [Index sur colonnes fréquemment filtrées](/05-index-et-performance/05.1-index-colonnes-filtrees.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.9 Index covering et index-only scans

> **Niveau** : Intermédiaire
> **Durée estimée** : 2 heures
> **Prérequis** : Section 5.1 à 5.8 (Index B-Tree, composites et optimisation)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le concept d'index covering et son impact sur les performances
- Identifier quand un index-only scan est possible
- Créer des index covering stratégiques sans sur-indexer
- Analyser avec EXPLAIN la présence d'un index covering
- Mesurer le gain de performance des index-only scans
- Équilibrer la taille des index avec les bénéfices en lecture
- Appliquer les bonnes pratiques pour maximiser l'utilisation des index covering

---

## Introduction

Un **index covering** (index couvrant) est un index qui contient **toutes les colonnes** nécessaires pour répondre à une requête, éliminant ainsi complètement le besoin d'accéder à la table de données. Cette technique peut améliorer les performances de **3 à 10 fois** en réduisant drastiquement les opérations d'I/O.

💡 **Analogie** : Imaginez chercher un numéro de téléphone dans un annuaire. Un index normal vous donne la page du livre où se trouve l'information complète. Un index covering vous donne directement le numéro dans l'annuaire lui-même, sans avoir à ouvrir le livre.

**Bénéfice clé** : Éviter l'accès à la table principale = **moins d'I/O disque** = **plus de données en cache** = **performances accrues**.

---

## Comprendre l'accès aux données avec et sans index covering

### Processus normal : Index → Table

Quand MariaDB utilise un index **non-covering**, le processus est :

```sql
-- Table exemple
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category_id INT,
    price DECIMAL(10,2),
    description TEXT,
    stock_quantity INT,
    created_at DATETIME,
    INDEX idx_category (category_id)
) ENGINE=InnoDB;

-- Requête
SELECT product_id, name, price
FROM products
WHERE category_id = 5;
```

**Étapes d'exécution** :

1. **Lecture de l'index** `idx_category` :
   ```
   Index idx_category:
   category_id=5 → [product_id: 101, 205, 312, 445, ...]
   ```

2. **Pour chaque product_id trouvé**, accès à la table :
   ```
   Table products:
   product_id=101 → Lire toute la ligne (name, price, description, etc.)
   product_id=205 → Lire toute la ligne
   product_id=312 → Lire toute la ligne
   ...
   ```

3. **Filtrer les colonnes** SELECT (name, price)

**Coût** :
- Lecture index : 1 opération I/O
- Lecture table : N opérations I/O (N = nombre de lignes)
- **Total : 1 + N opérations I/O**

### Processus optimisé : Index-only scan

Avec un index covering contenant toutes les colonnes nécessaires :

```sql
-- Index covering
CREATE INDEX idx_category_covering
ON products(category_id, product_id, name, price);

-- Même requête
SELECT product_id, name, price
FROM products
WHERE category_id = 5;
```

**Étapes d'exécution** :

1. **Lecture de l'index** `idx_category_covering` :
   ```
   Index idx_category_covering:
   category_id=5, product_id=101, name='Laptop XPS', price=1299.99
   category_id=5, product_id=205, name='MacBook Pro', price=2499.99
   category_id=5, product_id=312, name='ThinkPad X1', price=1599.99
   ```

2. **C'est tout !** Toutes les données nécessaires sont dans l'index.

**Coût** :
- Lecture index : 1 opération I/O
- Lecture table : 0 opération
- **Total : 1 opération I/O**

**Amélioration** : Si N=1000 lignes, gain de **999 I/O** → **x1000 plus rapide** dans le meilleur cas.

---

## Identifier un index covering avec EXPLAIN

### Indicateur "Using index"

```sql
-- Index covering
CREATE INDEX idx_users_country_info
ON users(country, user_id, username, email);

-- Requête couverte
EXPLAIN
SELECT user_id, username, email
FROM users
WHERE country = 'FR';

-- Résultat :
-- id | type | key                    | Extra
-- 1  | ref  | idx_users_country_info | Using index
--
-- ✅ "Using index" = index-only scan (optimal)
```

**Signification "Using index"** :
- ✅ Toutes les colonnes SELECT sont dans l'index
- ✅ Toutes les colonnes WHERE sont dans l'index
- ✅ Aucun accès à la table de données
- ✅ Performance maximale

### Comparaison avec index non-covering

```sql
-- Index simple (non-covering)
CREATE INDEX idx_users_country ON users(country);

EXPLAIN
SELECT user_id, username, email
FROM users
WHERE country = 'FR';

-- Résultat :
-- id | type | key               | Extra
-- 1  | ref  | idx_users_country | Using index condition
--
-- ⚠️ "Using index condition" = utilise index PUIS accède table
-- (pas "Using index" seul)
```

### EXPLAIN ANALYZE : mesurer l'impact réel

```sql
-- Sans index covering
EXPLAIN ANALYZE
SELECT user_id, username, email
FROM users
WHERE country = 'FR';

-- Résultat :
-- -> Index lookup on users using idx_users_country
--    (cost=2500 rows=5000)
--    (actual time=0.5..85.3 rows=5000 loops=1)
--
-- Temps : 85ms (lecture index + 5000 accès table)

-- Avec index covering
CREATE INDEX idx_users_country_covering
ON users(country, user_id, username, email);

EXPLAIN ANALYZE
SELECT user_id, username, email
FROM users
WHERE country = 'FR';

-- Résultat :
-- -> Index lookup on users using idx_users_country_covering
--    (cost=1250 rows=5000)
--    (actual time=0.3..12.5 rows=5000 loops=1)
--    Extra: Using index
--
-- Temps : 12.5ms (lecture index uniquement)
-- Amélioration : x7
```

---

## Stratégies pour créer des index covering

### Principe 1 : Colonnes WHERE + SELECT

Un index covering doit contenir :
1. **Colonnes de filtrage** (WHERE, JOIN ON)
2. **Colonnes de tri** (ORDER BY) si applicable
3. **Colonnes sélectionnées** (SELECT)

```sql
-- Requête analysée
SELECT order_id, customer_name, total_amount
FROM orders
WHERE status = 'completed'
  AND order_date >= '2024-01-01'
ORDER BY order_date DESC
LIMIT 100;

-- Index covering optimal
CREATE INDEX idx_orders_covering
ON orders(
    status,           -- WHERE (égalité)
    order_date,       -- WHERE (plage) + ORDER BY
    order_id,         -- SELECT
    customer_name,    -- SELECT
    total_amount      -- SELECT
);

EXPLAIN
-- Extra: Using index
-- Index-only scan réussi
```

### Principe 2 : Ordre optimal des colonnes

**Règle générale** :
1. **Colonnes de filtrage** (WHERE) en premier
2. **Colonnes de tri** (ORDER BY) ensuite
3. **Colonnes additionnelles** (SELECT) à la fin

```sql
-- ✅ Bon ordre : filtrage → tri → données
CREATE INDEX idx_articles_optimal
ON articles(
    category_id,      -- WHERE égalité
    published_at,     -- WHERE plage + ORDER BY
    article_id,       -- SELECT (+ garantit unicité)
    title,            -- SELECT
    author_name       -- SELECT
);

-- ❌ Mauvais ordre : données avant filtrage
CREATE INDEX idx_articles_bad
ON articles(
    title,            -- SELECT (faible sélectivité)
    category_id,      -- WHERE
    published_at      -- WHERE
);
-- Ne peut pas être utilisé efficacement
```

### Principe 3 : Ne pas surcharger l'index

⚠️ **Compromis** : Plus l'index contient de colonnes, plus il est volumineux et coûteux à maintenir.

```sql
-- ❌ Index trop large
CREATE INDEX idx_products_monster
ON products(
    category_id,
    brand_id,
    product_id,
    name,
    description,      -- TEXT (très volumineux)
    price,
    stock_quantity,
    created_at,
    updated_at
);
-- Index de plusieurs Go, coût d'écriture énorme

-- ✅ Index ciblé sur colonnes réellement nécessaires
CREATE INDEX idx_products_targeted
ON products(
    category_id,
    brand_id,
    product_id,       -- Suffisant si description rarement sélectionnée
    name,
    price
);
-- Index compact, performances d'écriture acceptables
```

**Décision** :
- Requête exécutée **millions de fois/jour** → index covering même si large
- Requête exécutée **quelques fois/jour** → index simple, accepter accès table

### Principe 4 : Colonnes de taille limitée

```sql
-- ❌ Problème : colonnes TEXT/BLOB dans index covering
CREATE INDEX idx_articles_bad
ON articles(category_id, title, content); -- content = TEXT(64KB)
-- Index très volumineux, peu de lignes en cache

-- ✅ Solution 1 : Exclure colonnes volumineuses
CREATE INDEX idx_articles_good
ON articles(category_id, article_id, title);
-- Accepter accès table pour 'content' uniquement

-- ✅ Solution 2 : Préfixe de colonne
CREATE INDEX idx_articles_prefix
ON articles(category_id, title, content(100));
-- Index sur premiers 100 caractères de content seulement
```

---

## Cas d'usage optimaux pour index covering

### Cas 1 : Requêtes analytiques

Les requêtes d'agrégation bénéficient énormément des index covering.

```sql
-- Requête : statistiques par catégorie
SELECT
    category_id,
    COUNT(*) as product_count,
    AVG(price) as avg_price,
    MAX(price) as max_price
FROM products
WHERE is_active = 1
GROUP BY category_id;

-- Index covering optimal
CREATE INDEX idx_products_analytics
ON products(is_active, category_id, price);

EXPLAIN
-- Extra: Using index
--
-- Lit uniquement l'index pour faire les agrégations
-- Pas d'accès à la table (évite colonnes name, description, etc.)
```

**Amélioration mesurée** :
- Sans index covering : 800-2000ms (accès table pour chaque ligne)
- Avec index covering : 50-150ms (index-only scan)
- **Gain : x10-20**

### Cas 2 : Recherche paginée

```sql
-- Requête : liste de produits paginée
SELECT product_id, name, price
FROM products
WHERE category_id = 5
ORDER BY price ASC
LIMIT 20 OFFSET 1000;

-- Index covering élimine accès table
CREATE INDEX idx_products_listing
ON products(category_id, price, product_id, name);

EXPLAIN
-- Extra: Using index
--
-- Lit 1020 lignes de l'index (ignore 1000, retourne 20)
-- Aucun accès table
```

**Bénéfice majeur** : Même avec offset élevé, pas d'accès table pour les 1000 lignes ignorées.

### Cas 3 : Jointures avec colonnes limitées

```sql
-- Requête : commandes avec infos clients essentielles
SELECT
    o.order_id,
    o.order_date,
    o.total_amount,
    c.customer_name,
    c.email
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending';

-- Index covering sur orders
CREATE INDEX idx_orders_pending_info
ON orders(status, customer_id, order_id, order_date, total_amount);

-- Index covering sur customers
CREATE INDEX idx_customers_contact
ON customers(customer_id, customer_name, email);

EXPLAIN
-- Table orders : Extra: Using index
-- Table customers : Extra: Using index
--
-- Les deux tables utilisent index-only scan !
```

### Cas 4 : COUNT(*) optimisé

```sql
-- Requête : compter les articles par statut
SELECT status, COUNT(*)
FROM articles
GROUP BY status;

-- Index covering minimal
CREATE INDEX idx_articles_status ON articles(status);

EXPLAIN
-- Extra: Using index
--
-- COUNT(*) sur index est x5-10 plus rapide que sur table
```

💡 **Optimisation InnoDB** : `COUNT(*)` sans WHERE sur toute la table est optimisé (métadonnées), mais avec WHERE/GROUP BY, index covering est crucial.

### Cas 5 : Vérification d'existence

```sql
-- Requête : vérifier si un email existe
SELECT EXISTS(
    SELECT 1 FROM users WHERE email = 'john@example.com'
) as email_exists;

-- Index covering simple
CREATE INDEX idx_users_email ON users(email);

EXPLAIN
-- Extra: Using index
--
-- Vérifie existence sans accéder à la table
-- Arrêt dès première correspondance
```

---

## Index covering avec contraintes spéciales

### Primary Key clustered (InnoDB)

InnoDB stocke les données dans l'index de la clé primaire (**clustered index**). Cela a des implications pour les index covering.

```sql
-- Table InnoDB
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    order_date DATETIME,
    status VARCHAR(20),
    INDEX idx_customer (customer_id)
) ENGINE=InnoDB;
```

**Propriété importante** : Tous les index secondaires incluent **automatiquement** la PRIMARY KEY.

```sql
-- Index déclaré
CREATE INDEX idx_orders_status ON orders(status);

-- Index réel en mémoire (InnoDB ajoute PK)
-- idx_orders_status : (status, order_id)

-- Requête couverte automatiquement
SELECT order_id, status FROM orders WHERE status = 'pending';

EXPLAIN
-- Extra: Using index
-- Même sans inclure order_id dans l'index !
```

💡 **Astuce** : Exploiter cette propriété pour créer des index covering légers.

```sql
-- Au lieu de :
CREATE INDEX idx_verbose
ON orders(status, order_id, customer_id);

-- Suffit de créer :
CREATE INDEX idx_compact
ON orders(status, customer_id);
-- order_id est déjà inclus automatiquement par InnoDB
```

### Colonnes NULL et index covering

```sql
-- Table avec colonnes nullable
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category_id INT,
    optional_code VARCHAR(50),  -- Peut être NULL
    INDEX idx_category_code (category_id, optional_code)
);

-- Requête
SELECT product_id, optional_code
FROM products
WHERE category_id = 5;

EXPLAIN
-- Extra: Using index
-- ✅ Index covering fonctionne même avec NULL
```

**Note** : Les valeurs NULL sont stockées dans l'index B-Tree, donc pas de problème pour index covering.

### Index UNIQUE et covering

```sql
-- Index UNIQUE peut aussi être covering
CREATE UNIQUE INDEX idx_users_email_info
ON users(email, user_id, username);

-- Requête couverte
SELECT user_id, username
FROM users
WHERE email = 'john@example.com';

EXPLAIN
-- type: const (encore mieux que ref)
-- Extra: Using index
```

---

## Mesurer l'impact des index covering

### Méthodologie de mesure

```sql
-- 1. Mesurer AVANT index covering
EXPLAIN ANALYZE
SELECT product_id, name, price
FROM products
WHERE category_id = 5
ORDER BY price;

-- Résultat :
-- actual time: 0.5..120.3 rows=2500
-- (120ms)

-- 2. Créer index covering
CREATE INDEX idx_products_covering
ON products(category_id, price, product_id, name);

-- 3. Mesurer APRÈS
EXPLAIN ANALYZE
SELECT product_id, name, price
FROM products
WHERE category_id = 5
ORDER BY price;

-- Résultat :
-- actual time: 0.3..18.7 rows=2500
-- Extra: Using index
-- (18ms)
--
-- Amélioration : x6.4
```

### Impact sur le buffer pool

```sql
-- Vérifier utilisation buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Avant index covering (table accédée)
-- Innodb_buffer_pool_read_requests: 1,000,000
-- Innodb_buffer_pool_reads: 50,000 (lectures disque)
-- Taux de hit : 95%

-- Après index covering (index seul)
-- Innodb_buffer_pool_read_requests: 1,000,000
-- Innodb_buffer_pool_reads: 5,000 (lectures disque)
-- Taux de hit : 99.5%
```

**Bénéfice** : Plus de données en cache = moins d'I/O disque = performances globales améliorées.

### Taille des index : compromis

```sql
-- Vérifier taille des index
SELECT
    TABLE_NAME,
    INDEX_NAME,
    ROUND(SUM(STAT_VALUE * @@innodb_page_size) / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE DATABASE_NAME = 'mydb'
  AND TABLE_NAME = 'products'
GROUP BY TABLE_NAME, INDEX_NAME;

-- Résultat :
-- PRIMARY               : 250 MB
-- idx_category          : 15 MB (index simple)
-- idx_category_covering : 85 MB (index covering)
```

**Analyse du compromis** :

| Métrique | Index simple | Index covering | Différence |
|----------|--------------|----------------|------------|
| Taille | 15 MB | 85 MB | +70 MB |
| Temps lecture | 120 ms | 18 ms | x6.7 plus rapide |
| I/O par requête | 2501 | 1 | x2501 moins |

**Décision** :
- Requête exécutée 1M fois/jour → +70MB largement justifiés
- Requête exécutée 10 fois/jour → coût probablement non justifié

---

## Limitations et considérations

### Limitation 1 : Toutes les colonnes SELECT doivent être dans l'index

```sql
CREATE INDEX idx_users_partial
ON users(country, user_id, username);
-- N'inclut pas 'email'

-- ❌ Ne peut PAS être index-only
SELECT user_id, username, email
FROM users
WHERE country = 'FR';

EXPLAIN
-- Extra: (pas "Using index")
-- Doit accéder table pour 'email'

-- ✅ Solution : ajouter email à l'index
CREATE INDEX idx_users_complete
ON users(country, user_id, username, email);
```

### Limitation 2 : Colonnes calculées

```sql
-- ❌ Expression calculée empêche index-only
SELECT
    product_id,
    name,
    price * 1.20 as price_with_tax
FROM products
WHERE category_id = 5;

-- Même avec index covering sur (category_id, product_id, name, price),
-- MariaDB doit accéder à la table pour vérifier les données originales

-- ✅ Solution : colonne générée
ALTER TABLE products
ADD COLUMN price_with_tax DECIMAL(10,2)
    AS (price * 1.20) VIRTUAL;

CREATE INDEX idx_products_with_tax
ON products(category_id, product_id, name, price_with_tax);
```

### Limitation 3 : Certaines fonctions forcent accès table

```sql
-- Requête avec fonction
SELECT COUNT(DISTINCT customer_id)
FROM orders
WHERE status = 'completed';

-- Même avec index sur (status, customer_id),
-- COUNT(DISTINCT) peut nécessiter accès table selon implémentation
```

### Limitation 4 : Coût de maintenance en écriture

```sql
-- Impact sur INSERT/UPDATE/DELETE

-- Sans index covering (index simple)
INSERT INTO products VALUES (...);
-- Met à jour : PRIMARY + 1 index simple
-- Temps : 0.5ms

-- Avec index covering (5 colonnes)
INSERT INTO products VALUES (...);
-- Met à jour : PRIMARY + 1 index large
-- Temps : 1.2ms
--
-- Coût d'écriture × 2.4
```

**Recommandation** :
- Tables **read-heavy** (95%+ SELECT) → index covering pertinents
- Tables **write-heavy** (50%+ INSERT/UPDATE) → limiter les index covering

---

## Stratégies avancées

### Stratégie 1 : Index covering partiels

Pour requêtes avec plusieurs SELECT différents, créer des index covering pour les plus fréquents uniquement.

```sql
-- Scénario : 3 types de requêtes

-- Requête A (80% du trafic) : colonnes essentielles
SELECT product_id, name, price
FROM products
WHERE category_id = 5;

-- Requête B (15% du trafic) : colonnes étendues
SELECT product_id, name, price, description
FROM products
WHERE category_id = 5;

-- Requête C (5% du trafic) : toutes colonnes
SELECT * FROM products WHERE category_id = 5;

-- ✅ Stratégie optimale : 1 index covering pour A
CREATE INDEX idx_products_essential
ON products(category_id, product_id, name, price);
-- Couvre 80% avec index-only
-- Requêtes B et C acceptent accès table (moins fréquentes)
```

### Stratégie 2 : Index covering pour agrégations

```sql
-- Requête analytique fréquente
SELECT
    DATE(order_date) as day,
    status,
    COUNT(*) as order_count,
    SUM(total_amount) as daily_revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE(order_date), status;

-- Index covering pour agrégation
CREATE INDEX idx_orders_daily_stats
ON orders(order_date, status, total_amount);

EXPLAIN
-- Extra: Using index; Using temporary
-- "Using index" présent malgré "Using temporary"
-- Agrégation faite sur index, pas sur table
```

### Stratégie 3 : Combiner avec partition pruning

```sql
-- Table partitionnée
CREATE TABLE logs (
    log_id BIGINT AUTO_INCREMENT,
    log_date DATE,
    level VARCHAR(10),
    message TEXT,
    PRIMARY KEY (log_id, log_date)
) PARTITION BY RANGE (YEAR(log_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Index covering par partition
CREATE INDEX idx_logs_level
ON logs(log_date, level, log_id);

-- Requête
SELECT log_id, level
FROM logs
WHERE log_date = '2024-06-15' AND level = 'ERROR';

EXPLAIN
-- partitions: p2024 (partition pruning)
-- Extra: Using index (index covering)
--
-- Double optimisation : partition + index-only
```

---

## Checklist pour index covering

### ✅ Quand créer un index covering

- [ ] Requête exécutée **fréquemment** (milliers de fois/jour)
- [ ] Requête SELECT un **nombre limité de colonnes** (3-6)
- [ ] Colonnes sélectionnées sont de **taille raisonnable** (pas TEXT/BLOB volumineux)
- [ ] Table est **read-heavy** (>80% SELECT)
- [ ] Gain de performance mesurable avec EXPLAIN ANALYZE (>x3)
- [ ] Taille d'index additionnelle acceptable (<10% de la table)

### ⚠️ Quand éviter

- [ ] Requête SELECT * ou beaucoup de colonnes
- [ ] Colonnes incluent TEXT/BLOB volumineux
- [ ] Table write-heavy (>30% INSERT/UPDATE)
- [ ] Requête rarement exécutée (<100 fois/jour)
- [ ] Index résultant > 2× taille table originale

### 🔍 Vérification

1. **Avant création** : `EXPLAIN` montre type=ref ou range, mais pas "Using index"
2. **Après création** : `EXPLAIN` montre "Using index"
3. **Mesure** : `EXPLAIN ANALYZE` montre amélioration ≥ x3
4. **Production** : Monitoring montre réduction I/O disque

---

## Exemples complets d'optimisation

### Exemple 1 : Tableau de bord e-commerce

**Requête initiale (250ms)** :

```sql
-- Requête : top produits par catégorie
SELECT
    p.product_id,
    p.name,
    p.price,
    COUNT(oi.item_id) as sales_count
FROM products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
WHERE p.category_id = 5
  AND oi.order_date >= '2024-01-01'
GROUP BY p.product_id, p.name, p.price
ORDER BY sales_count DESC
LIMIT 10;

EXPLAIN ANALYZE
-- actual time: 0.8..250.5ms
-- Accès table products pour chaque ligne
```

**Optimisation avec index covering** :

```sql
-- Index covering sur products
CREATE INDEX idx_products_category_info
ON products(category_id, product_id, name, price);

-- Index covering sur order_items
CREATE INDEX idx_order_items_product_date
ON order_items(product_id, order_date, item_id);

EXPLAIN ANALYZE
-- actual time: 0.5..35.2ms
-- Extra: Using index (sur les deux tables)
--
-- Amélioration : x7
```

### Exemple 2 : Recherche utilisateurs

**Requête initiale (180ms)** :

```sql
-- Recherche paginée
SELECT user_id, username, email, country
FROM users
WHERE status = 'active'
  AND last_login >= NOW() - INTERVAL 30 DAY
ORDER BY last_login DESC
LIMIT 20 OFFSET 500;

EXPLAIN ANALYZE
-- actual time: 0.6..180.3ms
-- 520 accès table (500 ignorés + 20 retournés)
```

**Optimisation** :

```sql
CREATE INDEX idx_users_active_covering
ON users(
    status,
    last_login,
    user_id,
    username,
    email,
    country
);

EXPLAIN ANALYZE
-- actual time: 0.4..22.1ms
-- Extra: Using index
--
-- Lit 520 lignes d'index, 0 accès table
-- Amélioration : x8
```

### Exemple 3 : Statistiques en temps réel

**Requête initiale (450ms)** :

```sql
-- Dashboard : stats par région
SELECT
    region,
    COUNT(*) as order_count,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as avg_order
FROM orders
WHERE order_date >= CURDATE()
  AND status IN ('completed', 'processing')
GROUP BY region;

EXPLAIN ANALYZE
-- actual time: 1.2..450.8ms
-- Scanne toute la table pour aujourd'hui
```

**Optimisation** :

```sql
CREATE INDEX idx_orders_daily_region_stats
ON orders(order_date, status, region, total_amount);

EXPLAIN ANALYZE
-- actual time: 0.8..45.3ms
-- Extra: Using index
--
-- Agrégation sur index uniquement
-- Amélioration : x10
```

---

## ✅ Points clés à retenir

- **Index covering** = toutes colonnes nécessaires dans l'index → aucun accès table
- **"Using index" dans EXPLAIN** = indicateur d'index-only scan réussi
- **Gain typique** : x3 à x10 en réduction I/O et temps d'exécution
- **Ordre colonnes** : filtrage → tri → données SELECT
- **InnoDB** : PRIMARY KEY automatiquement incluse dans index secondaires
- **Compromis** : taille index vs performance lecture
- **Idéal pour** : requêtes fréquentes, read-heavy, colonnes limitées
- **Éviter pour** : SELECT *, colonnes volumineuses, write-heavy
- **Toujours mesurer** : EXPLAIN ANALYZE avant/après création

---

## 🔗 Ressources et références

- [📖 MariaDB Covering Indexes](https://mariadb.com/kb/en/covering-indexes/)
- [📖 InnoDB Clustered and Secondary Indexes](https://mariadb.com/kb/en/innodb-storage-engine/)
- [📖 EXPLAIN Output](https://mariadb.com/kb/en/explain/)
- [📖 Index Optimization](https://mariadb.com/kb/en/optimization-and-indexes/)
- [📊 Buffer Pool Statistics](https://mariadb.com/kb/en/innodb-buffer-pool/)

---

## ➡️ Section suivante

**5.10 Invisible indexes et Progressive indexes** : Techniques avancées pour tester l'impact des index en production sans risque et construire des index progressivement pour minimiser l'impact sur les opérations en cours.

⏭️ [Invisible indexes et Progressive indexes](/05-index-et-performance/10-invisible-progressive-indexes.md)

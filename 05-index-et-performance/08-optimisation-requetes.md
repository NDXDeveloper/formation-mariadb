🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.8 Optimisation des requêtes

> **Niveau** : Intermédiaire
> **Durée estimée** : 2.5 heures
> **Prérequis** : Section 5.1 à 5.7 (Index et analyse EXPLAIN)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Identifier les anti-patterns SQL courants et les corriger
- Réécrire des requêtes inefficaces pour améliorer leurs performances
- Optimiser les sous-requêtes et choisir entre sous-requêtes et jointures
- Utiliser efficacement les CTE (Common Table Expressions)
- Éviter les fonctions sur colonnes indexées dans les clauses WHERE
- Optimiser les requêtes avec agrégations complexes
- Appliquer les bonnes pratiques de rédaction SQL pour la performance
- Mesurer l'impact réel des optimisations

---

## Introduction

L'indexation résout 80% des problèmes de performance, mais les 20% restants nécessitent une **optimisation au niveau des requêtes** elles-mêmes. Une requête mal écrite peut être lente même avec des index parfaits. À l'inverse, une requête bien conçue peut parfois compenser un indexage sous-optimal.

💡 **Principe clé** : L'optimiseur MariaDB est intelligent, mais il ne peut pas deviner vos intentions. Une requête claire et bien structurée permet à l'optimiseur de faire son travail efficacement.

⚠️ **Attention** : Toute optimisation doit être **mesurée** avec EXPLAIN ANALYZE. Une optimisation théoriquement meilleure peut parfois être moins performante en pratique selon vos données réelles.

---

## Anti-patterns SQL et corrections

### Anti-pattern 1 : Fonctions sur colonnes indexées

**Problème** : Appliquer une fonction à une colonne indexée empêche l'utilisation de l'index.

```sql
-- ❌ MAUVAIS : fonction sur colonne indexée
SELECT * FROM orders
WHERE YEAR(order_date) = 2024;

EXPLAIN
-- type: ALL
-- rows: 1000000
-- Extra: Using where
-- Index sur order_date NON utilisé !

-- ❌ Autres exemples problématiques
WHERE UPPER(email) = 'JOHN@EXAMPLE.COM'
WHERE DATE(created_at) = '2024-01-15'
WHERE price * 1.2 > 100
```

**Solution 1 : Réécrire la condition** :

```sql
-- ✅ BON : condition directe sur la colonne
SELECT * FROM orders
WHERE order_date >= '2024-01-01'
  AND order_date < '2025-01-01';

EXPLAIN
-- type: range
-- key: idx_order_date
-- rows: 150000
-- Index utilisé efficacement
```

**Solution 2 : Colonne générée** :

```sql
-- Créer une colonne calculée indexable
ALTER TABLE orders
ADD COLUMN order_year SMALLINT AS (YEAR(order_date)) VIRTUAL,
ADD INDEX idx_order_year (order_year);

-- Requête optimisée
SELECT * FROM orders WHERE order_year = 2024;

EXPLAIN
-- type: ref
-- key: idx_order_year
-- rows: 150000
```

**Solution 3 : Expression index** (MariaDB 10.2+) :

```sql
-- Index sur expression calculée
CREATE INDEX idx_orders_year ON orders((YEAR(order_date)));

-- La requête originale peut maintenant utiliser l'index
SELECT * FROM orders WHERE YEAR(order_date) = 2024;
```

### Anti-pattern 2 : SELECT * au lieu de colonnes spécifiques

**Problème** : `SELECT *` empêche l'utilisation d'index covering et charge des données inutiles.

```sql
-- ❌ MAUVAIS : sélectionne toutes les colonnes
SELECT * FROM users WHERE country = 'FR';

-- Table users : 50 colonnes, certaines TEXT/BLOB volumineuses
-- Charge plusieurs Mo de données alors qu'on n'utilise que 3 colonnes

-- ❌ Empêche index covering même si possible
CREATE INDEX idx_users_country_email ON users(country, email);
-- Index covering potentiel, mais SELECT * force l'accès table
```

**Solution** :

```sql
-- ✅ BON : sélectionner uniquement les colonnes nécessaires
SELECT user_id, username, email
FROM users
WHERE country = 'FR';

-- Permet index covering si approprié
CREATE INDEX idx_users_covering
ON users(country, user_id, username, email);

EXPLAIN
-- Extra: Using index (index-only scan, optimal)
```

**Impact mesuré** :
- Réduction bande passante : 90%+ si colonnes TEXT/BLOB
- Réduction mémoire buffer pool
- Possibilité d'index covering

### Anti-pattern 3 : OR sur colonnes différentes

**Problème** : `OR` sur colonnes non indexées ensemble rend l'indexation inefficace.

```sql
-- ❌ MAUVAIS : OR sur colonnes différentes
SELECT * FROM products
WHERE category_id = 5 OR brand_id = 10;

EXPLAIN
-- type: index_merge (ou pire: ALL)
-- Extra: Using union(idx_category,idx_brand); Using where
-- Moins efficace qu'un index composite
```

**Solution 1 : UNION ALL si sélectif** :

```sql
-- ✅ BON : UNION ALL si chaque branche est sélective
SELECT * FROM products WHERE category_id = 5
UNION ALL
SELECT * FROM products WHERE brand_id = 10 AND category_id != 5;

-- Chaque requête utilise son index optimal
-- Évite les doublons avec condition AND category_id != 5
```

**Solution 2 : Restructurer la logique** :

```sql
-- Si la logique métier le permet
CREATE TABLE product_filters (
    product_id INT,
    filter_type ENUM('category', 'brand'),
    filter_value INT,
    INDEX idx_filter (filter_type, filter_value, product_id)
);

-- Requête unifiée
SELECT DISTINCT p.*
FROM products p
INNER JOIN product_filters pf ON p.product_id = pf.product_id
WHERE (pf.filter_type = 'category' AND pf.filter_value = 5)
   OR (pf.filter_type = 'brand' AND pf.filter_value = 10);
```

### Anti-pattern 4 : NOT IN avec sous-requête

**Problème** : `NOT IN` avec sous-requête peut être très lent et a un comportement contre-intuitif avec NULL.

```sql
-- ❌ MAUVAIS : NOT IN avec sous-requête
SELECT * FROM products
WHERE product_id NOT IN (
    SELECT product_id FROM archived_products
);

-- Problèmes :
-- 1. Lent sur grandes tables
-- 2. Si archived_products.product_id contient NULL, retourne 0 ligne !
-- 3. Sous-requête peut être exécutée pour chaque ligne
```

**Solution 1 : LEFT JOIN avec IS NULL** :

```sql
-- ✅ BON : LEFT JOIN anti-pattern
SELECT p.*
FROM products p
LEFT JOIN archived_products ap ON p.product_id = ap.product_id
WHERE ap.product_id IS NULL;

EXPLAIN
-- Utilise les index efficacement
-- Logique claire et prévisible
```

**Solution 2 : NOT EXISTS** :

```sql
-- ✅ BON : NOT EXISTS (souvent plus rapide que NOT IN)
SELECT * FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM archived_products ap
    WHERE ap.product_id = p.product_id
);

-- Avantage : s'arrête dès qu'une correspondance est trouvée
```

### Anti-pattern 5 : LIMIT sans ORDER BY

**Problème** : Résultats non déterministes et impossibilité d'optimiser avec index.

```sql
-- ❌ MAUVAIS : LIMIT sans ORDER BY
SELECT * FROM articles LIMIT 10;

-- Problèmes :
-- 1. Résultats différents à chaque exécution
-- 2. Impossible de paginer correctement
-- 3. Ne peut pas utiliser d'index pour optimiser
```

**Solution** :

```sql
-- ✅ BON : Toujours avec ORDER BY
SELECT * FROM articles
ORDER BY article_id
LIMIT 10;

-- Bénéfices :
-- 1. Résultats déterministes
-- 2. Peut utiliser PRIMARY KEY pour optimisation
-- 3. Pagination cohérente possible
```

---

## Optimisation des sous-requêtes

### Sous-requêtes scalaires : quand éviter

**Problème** : Sous-requêtes scalaires dans SELECT peuvent être exécutées N fois.

```sql
-- ❌ PROBLÉMATIQUE : sous-requête dans SELECT
SELECT
    c.customer_id,
    c.name,
    (SELECT COUNT(*)
     FROM orders o
     WHERE o.customer_id = c.customer_id) as order_count,
    (SELECT SUM(total_amount)
     FROM orders o
     WHERE o.customer_id = c.customer_id) as total_spent
FROM customers c
WHERE c.country = 'FR';

-- Problème : 2 sous-requêtes × 5000 clients = 10000 exécutions !
-- Temps : 2-5 secondes
```

**Solution : JOIN avec agrégation** :

```sql
-- ✅ OPTIMISÉ : une seule jointure
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id) as order_count,
    COALESCE(SUM(o.total_amount), 0) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.country = 'FR'
GROUP BY c.customer_id, c.name;

-- Une seule passe sur les données
-- Temps : 50-200ms
-- Amélioration : x10-25
```

### IN vs EXISTS : choisir le bon opérateur

**Règle générale** :
- `IN` : Efficace si la sous-requête retourne **peu de lignes**
- `EXISTS` : Efficace si la sous-requête retourne **beaucoup de lignes**

```sql
-- Scénario 1 : Sous-requête retourne peu de lignes (< 100)
-- ✅ IN est efficace
SELECT * FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM vip_customers
);
-- vip_customers : 50 lignes → liste en mémoire

-- Scénario 2 : Sous-requête retourne beaucoup de lignes (> 10000)
-- ✅ EXISTS est plus efficace
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.order_id = o.order_id
    AND oi.product_id = 12345
);
-- EXISTS s'arrête dès la première correspondance trouvée
```

**Exemple comparatif** :

```sql
-- Cas : trouver produits jamais commandés

-- Option 1 : NOT IN
SELECT * FROM products p
WHERE p.product_id NOT IN (
    SELECT product_id FROM order_items
);
-- Charge toute la sous-requête en mémoire
-- Temps : 2-5 secondes si order_items volumineux

-- Option 2 : NOT EXISTS (recommandé)
SELECT * FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.product_id
);
-- Arrêt dès première correspondance
-- Temps : 200-800ms
-- Amélioration : x3-10
```

### Sous-requêtes corrélées : optimiser ou éliminer

**Sous-requête corrélée** : référence la table externe, donc potentiellement exécutée N fois.

```sql
-- ❌ LENT : sous-requête corrélée
SELECT
    p.product_name,
    p.price,
    (SELECT AVG(price)
     FROM products p2
     WHERE p2.category_id = p.category_id) as category_avg_price
FROM products p;

-- Exécuté une fois par ligne de products
-- Si 100k produits, 100k calculs de moyenne
```

**Solution : Pré-calculer avec JOIN** :

```sql
-- ✅ OPTIMISÉ : calculer une seule fois par catégorie
WITH category_averages AS (
    SELECT
        category_id,
        AVG(price) as avg_price
    FROM products
    GROUP BY category_id
)
SELECT
    p.product_name,
    p.price,
    ca.avg_price as category_avg_price
FROM products p
INNER JOIN category_averages ca ON p.category_id = ca.category_id;

-- Calcul une fois par catégorie, pas par produit
-- Amélioration : x100-1000 si beaucoup de produits par catégorie
```

---

## CTE (Common Table Expressions) : bonnes pratiques

### Quand utiliser les CTE

**Avantages des CTE** :
- ✅ Lisibilité : code modulaire et structuré
- ✅ Réutilisation : référencer plusieurs fois sans recalcul (si matérialisé)
- ✅ Récursivité : WITH RECURSIVE pour données hiérarchiques

```sql
-- CTE pour clarifier logique complexe
WITH
    active_customers AS (
        SELECT customer_id, name, country
        FROM customers
        WHERE status = 'active'
          AND last_order_date >= NOW() - INTERVAL 90 DAY
    ),
    high_value_orders AS (
        SELECT customer_id, COUNT(*) as order_count, SUM(total) as total_spent
        FROM orders
        WHERE order_date >= NOW() - INTERVAL 90 DAY
          AND total > 100
        GROUP BY customer_id
    )
SELECT
    ac.customer_id,
    ac.name,
    ac.country,
    COALESCE(hvo.order_count, 0) as high_value_orders,
    COALESCE(hvo.total_spent, 0) as total_spent
FROM active_customers ac
LEFT JOIN high_value_orders hvo ON ac.customer_id = hvo.customer_id
ORDER BY total_spent DESC;
```

### CTE vs sous-requêtes vs tables temporaires

**Comparaison** :

| Critère | CTE | Sous-requête | Table temp |
|---------|-----|--------------|------------|
| Lisibilité | ✅ Excellente | ⚠️ Moyenne | ✅ Bonne |
| Performance | ⚠️ Variable | ⚠️ Variable | ✅ Prévisible |
| Réutilisation | ✅ Oui | ❌ Non | ✅ Oui |
| Index | ❌ Non | ❌ Non | ✅ Oui |
| Scope | Requête | Clause | Session |

**Choisir selon le cas** :

```sql
-- Cas 1 : Logique simple, une utilisation → Sous-requête
SELECT * FROM orders
WHERE customer_id IN (SELECT customer_id FROM vip_customers);

-- Cas 2 : Logique complexe, utilisée plusieurs fois → CTE
WITH filtered_orders AS (
    SELECT * FROM orders WHERE status = 'completed'
)
SELECT
    (SELECT COUNT(*) FROM filtered_orders WHERE region = 'EU') as eu_count,
    (SELECT COUNT(*) FROM filtered_orders WHERE region = 'US') as us_count;

-- Cas 3 : Dataset intermédiaire volumineux, besoin d'index → Table temp
CREATE TEMPORARY TABLE temp_large_dataset AS
SELECT /* requête complexe */ ... FROM ... WHERE ...;

CREATE INDEX idx_temp ON temp_large_dataset(key_column);

SELECT * FROM temp_large_dataset WHERE ...;
```

### CTE récursives : hiérarchies et graphes

```sql
-- Exemple : hiérarchie de catégories
WITH RECURSIVE category_tree AS (
    -- Ancre : catégories racines
    SELECT
        category_id,
        name,
        parent_category_id,
        0 as level,
        CAST(name AS CHAR(500)) as path
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    -- Récursion : sous-catégories
    SELECT
        c.category_id,
        c.name,
        c.parent_category_id,
        ct.level + 1,
        CONCAT(ct.path, ' > ', c.name)
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_category_id = ct.category_id
    WHERE ct.level < 10  -- Limite pour éviter boucles infinies
)
SELECT
    category_id,
    CONCAT(REPEAT('  ', level), name) as indented_name,
    level,
    path
FROM category_tree
ORDER BY path;

-- Résultat :
-- Electronics
--   Computers
--     Laptops
--     Desktops
--   Mobile
--     Smartphones
--     Tablets
```

⚠️ **Attention** : Toujours inclure une **condition d'arrêt** (ex: `level < 10`) pour éviter les boucles infinies.

---

## Optimisation des jointures

### Ordre des jointures : laisser l'optimiseur décider

MariaDB réorganise automatiquement les jointures, mais on peut influencer avec des hints si nécessaire.

```sql
-- MariaDB optimise automatiquement l'ordre
SELECT *
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
WHERE c.country = 'FR';

-- L'optimiseur choisit l'ordre optimal basé sur :
-- 1. Taille des tables
-- 2. Sélectivité des conditions WHERE
-- 3. Index disponibles
-- 4. Statistiques

EXPLAIN
-- Montre l'ordre choisi par l'optimiseur
```

**Forcer l'ordre si nécessaire** (rare) :

```sql
-- STRAIGHT_JOIN force l'ordre d'écriture
SELECT STRAIGHT_JOIN *
FROM small_table s
INNER JOIN large_table l ON s.id = l.small_id
WHERE s.filter_column = 'value';

-- Utiliser uniquement si l'optimiseur fait un mauvais choix
-- Toujours mesurer avec EXPLAIN ANALYZE
```

### Éviter les jointures cartésiennes accidentelles

```sql
-- ❌ DANGER : jointure cartésienne accidentelle
SELECT *
FROM orders o, customers c
WHERE o.status = 'pending';
-- Oubli de la condition de jointure !
-- Résultat : orders × customers lignes (explosion combinatoire)

-- ✅ BON : toujours spécifier la condition de jointure
SELECT *
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending';
```

**Détecter dans EXPLAIN** :

```
-- Signe d'alerte :
type: ALL
rows: 1000000
Extra: Using join buffer (Block Nested Loop)

-- Si type=ALL sur table jointe sans condition appropriée
```

### Optimiser les LEFT JOIN

**Règle** : Placer les conditions sur la table de gauche dans WHERE, sur la table de droite dans ON.

```sql
-- ❌ INCORRECT : filtre sur table RIGHT dans WHERE
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01';
-- Convertit le LEFT JOIN en INNER JOIN !
-- Ne retourne que les clients avec commandes en 2024

-- ✅ CORRECT : filtre sur RIGHT table dans ON
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
    AND o.order_date >= '2024-01-01';
-- Retourne tous les clients, avec commandes 2024 si elles existent

-- ✅ CORRECT : filtre sur LEFT table dans WHERE
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.country = 'FR';
-- Filtre les clients français, puis leurs commandes
```

---

## Optimisation des agrégations

### GROUP BY : minimiser les données avant agrégation

```sql
-- ❌ INEFFICACE : agrège puis filtre
SELECT
    customer_id,
    COUNT(*) as order_count
FROM orders
GROUP BY customer_id
HAVING order_count > 5;

-- Agrège d'abord TOUS les clients, puis filtre
-- Beaucoup de travail inutile

-- ✅ OPTIMISÉ : filtre puis agrège (si possible)
SELECT
    customer_id,
    COUNT(*) as order_count
FROM orders
WHERE order_date >= '2024-01-01'  -- Filtre AVANT agrégation
GROUP BY customer_id
HAVING COUNT(*) > 5;  -- Filtre APRÈS agrégation (nécessaire ici)
```

**Principe** :
- `WHERE` filtre **avant** GROUP BY → réduit les données à agréger
- `HAVING` filtre **après** GROUP BY → ne réduit pas le travail d'agrégation

### Éviter DISTINCT si possible

```sql
-- ❌ LENT : DISTINCT sur grosse table
SELECT DISTINCT customer_id
FROM orders
WHERE order_date >= '2024-01-01';

-- Doit trier/hasher toutes les lignes pour éliminer doublons
-- Temps : 500-2000ms

-- ✅ PLUS RAPIDE : GROUP BY (souvent)
SELECT customer_id
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id;

-- Utilise index si disponible sur (order_date, customer_id)
-- Temps : 50-200ms
-- Amélioration : x5-10
```

**Cas où DISTINCT est approprié** :

```sql
-- DISTINCT nécessaire pour logique métier
SELECT DISTINCT
    c.customer_id,
    c.name
FROM customers c
INNER JOIN orders o1 ON c.customer_id = o1.customer_id
INNER JOIN orders o2 ON c.customer_id = o2.customer_id
WHERE o1.product_id = 100
  AND o2.product_id = 200;
-- Client doit avoir commandé les deux produits
```

### Optimiser COUNT(*)

```sql
-- COUNT(*) est optimisé par MariaDB sur table entière
SELECT COUNT(*) FROM large_table;
-- Utilise métadonnées si possible (pas de WHERE)

-- Avec WHERE, nécessite parcours
SELECT COUNT(*) FROM orders WHERE status = 'completed';
-- Utilise index sur status si disponible

-- ✅ Astuce : COUNT approximatif pour gros volumes
SELECT TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders';
-- Approximation rapide (peut être inexacte)
```

---

## Pagination efficace

### Problème : OFFSET élevé

```sql
-- ❌ INEFFICACE : pagination classique avec OFFSET
SELECT * FROM articles
ORDER BY article_id
LIMIT 10 OFFSET 50000;

-- Problème : doit lire et IGNORER 50000 lignes
-- Temps : augmente linéairement avec l'offset
-- Page 1 : 10ms
-- Page 1000 : 500ms
-- Page 10000 : 5000ms
```

### Solution : Keyset Pagination

```sql
-- ✅ EFFICACE : pagination par curseur (keyset)
-- Page 1 : obtenir les 10 premiers
SELECT * FROM articles
ORDER BY article_id
LIMIT 10;
-- Dernier article_id de la page : 10

-- Page 2 : utiliser le dernier ID
SELECT * FROM articles
WHERE article_id > 10
ORDER BY article_id
LIMIT 10;
-- Dernier article_id : 20

-- Page 3
SELECT * FROM articles
WHERE article_id > 20
ORDER BY article_id
LIMIT 10;

-- Performance constante : toujours ~10ms
-- Indépendant de la profondeur de pagination
```

**Avec tri composite** :

```sql
-- Tri par date puis ID (pour stabilité)
-- Page 1
SELECT * FROM articles
ORDER BY published_at DESC, article_id DESC
LIMIT 10;
-- Dernière ligne : published_at='2024-06-15 10:30:00', article_id=12345

-- Page 2
SELECT * FROM articles
WHERE (published_at, article_id) < ('2024-06-15 10:30:00', 12345)
ORDER BY published_at DESC, article_id DESC
LIMIT 10;

-- Index requis
CREATE INDEX idx_articles_published_id
ON articles(published_at DESC, article_id DESC);
```

---

## Requêtes avec UNION : bonnes pratiques

### UNION vs UNION ALL

```sql
-- UNION : élimine les doublons (tri/hash requis)
SELECT customer_id FROM orders WHERE region = 'EU'
UNION
SELECT customer_id FROM orders WHERE status = 'vip';
-- Doit éliminer doublons → coûteux

-- UNION ALL : conserve les doublons (plus rapide)
SELECT customer_id FROM orders WHERE region = 'EU'
UNION ALL
SELECT customer_id FROM orders WHERE region = 'US';
-- Pas de déduplication → x2-5 plus rapide
```

**Règle** : Utiliser `UNION ALL` si :
- Vous êtes sûr qu'il n'y a pas de doublons
- Les doublons sont acceptables pour votre logique métier

### Optimiser les UNION multiples

```sql
-- ❌ INEFFICACE : multiples full scans
SELECT product_id, 'Electronics' as category FROM products WHERE category_id = 1
UNION ALL
SELECT product_id, 'Books' as category FROM products WHERE category_id = 2
UNION ALL
SELECT product_id, 'Clothing' as category FROM products WHERE category_id = 3;
-- Scanne la table 3 fois !

-- ✅ OPTIMISÉ : un seul scan avec CASE
SELECT
    product_id,
    CASE
        WHEN category_id = 1 THEN 'Electronics'
        WHEN category_id = 2 THEN 'Books'
        WHEN category_id = 3 THEN 'Clothing'
    END as category
FROM products
WHERE category_id IN (1, 2, 3);
-- Un seul scan
-- Amélioration : x3
```

---

## Gestion des expressions et calculs

### Précalculer plutôt que calculer à chaque requête

```sql
-- ❌ INEFFICACE : calcul dans chaque requête
SELECT
    product_id,
    name,
    price,
    price * 1.20 as price_with_tax
FROM products;
-- Calcul répété à chaque SELECT

-- ✅ OPTIMISÉ : colonne générée
ALTER TABLE products
ADD COLUMN price_with_tax DECIMAL(10,2)
    AS (price * 1.20) STORED;

-- Ou colonne calculée mise à jour par trigger
ALTER TABLE products ADD COLUMN price_with_tax DECIMAL(10,2);

CREATE TRIGGER trg_products_tax
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
BEGIN
    SET NEW.price_with_tax = NEW.price * 1.20;
END;
```

### Éviter les calculs répétés dans JOIN

```sql
-- ❌ INEFFICACE : calcul dans condition de jointure
SELECT *
FROM orders o
INNER JOIN order_archive oa
    ON YEAR(o.order_date) = YEAR(oa.archive_date)
   AND MONTH(o.order_date) = MONTH(oa.archive_date);
-- Calculs YEAR/MONTH pour chaque combinaison

-- ✅ OPTIMISÉ : colonnes pré-calculées
ALTER TABLE orders
ADD COLUMN order_year_month CHAR(7)
    AS (DATE_FORMAT(order_date, '%Y-%m')) VIRTUAL,
ADD INDEX idx_orders_ym (order_year_month);

ALTER TABLE order_archive
ADD COLUMN archive_year_month CHAR(7)
    AS (DATE_FORMAT(archive_date, '%Y-%m')) VIRTUAL,
ADD INDEX idx_archive_ym (archive_year_month);

SELECT *
FROM orders o
INNER JOIN order_archive oa ON o.order_year_month = oa.archive_year_month;
-- Jointure directe sur colonnes indexées
```

---

## Techniques avancées d'optimisation

### Dénormalisation sélective

Pour des requêtes très fréquentes, répliquer des données peut améliorer drastiquement les performances.

```sql
-- Scénario : afficher commande avec nom client (millions de fois/jour)

-- ❌ Approche normalisée : JOIN à chaque fois
SELECT o.order_id, o.total_amount, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_id = 12345;
-- 10ms × 1M requêtes/jour = 3h CPU

-- ✅ Dénormalisation : stocker customer_name dans orders
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);

-- Mise à jour par trigger
CREATE TRIGGER trg_orders_customer_name
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    SELECT name INTO NEW.customer_name
    FROM customers
    WHERE customer_id = NEW.customer_id;
END;

-- Requête simple
SELECT order_id, total_amount, customer_name
FROM orders
WHERE order_id = 12345;
-- 1ms × 1M = 17min CPU
-- Économie : 2h40 CPU/jour
```

⚠️ **Compromis** :
- ✅ Performance lecture × 5-10
- ❌ Redondance des données
- ❌ Maintenance UPDATE si customer_name change

### Requêtes pré-agrégées : tables de matérialisation

```sql
-- Scénario : rapports analytics quotidiens

-- ❌ Calcul en temps réel (lent)
SELECT
    DATE(order_date) as day,
    COUNT(*) as order_count,
    SUM(total_amount) as daily_revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE(order_date);
-- 2-5 secondes sur millions de commandes

-- ✅ Table pré-agrégée
CREATE TABLE daily_stats (
    stat_date DATE PRIMARY KEY,
    order_count INT,
    daily_revenue DECIMAL(12,2),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Mise à jour nocturne (batch)
INSERT INTO daily_stats (stat_date, order_count, daily_revenue)
SELECT
    DATE(order_date),
    COUNT(*),
    SUM(total_amount)
FROM orders
WHERE DATE(order_date) = CURDATE() - INTERVAL 1 DAY
GROUP BY DATE(order_date)
ON DUPLICATE KEY UPDATE
    order_count = VALUES(order_count),
    daily_revenue = VALUES(daily_revenue),
    updated_at = CURRENT_TIMESTAMP;

-- Requête rapide
SELECT * FROM daily_stats
WHERE stat_date >= '2024-01-01'
ORDER BY stat_date;
-- 10-50ms
-- Amélioration : x40-500
```

### Batch processing pour mises à jour massives

```sql
-- ❌ INEFFICACE : mise à jour ligne par ligne
UPDATE products SET status = 'inactive' WHERE last_sale < '2023-01-01';
-- Peut prendre plusieurs minutes sur millions de lignes
-- Verrouille la table, bloque autres transactions

-- ✅ OPTIMISÉ : batch processing
SET @batch_size = 1000;
SET @rows_updated = 1;

WHILE @rows_updated > 0 DO
    UPDATE products
    SET status = 'inactive'
    WHERE last_sale < '2023-01-01'
      AND status != 'inactive'
    LIMIT @batch_size;

    SET @rows_updated = ROW_COUNT();

    -- Pause entre batches pour éviter verrouillage prolongé
    DO SLEEP(0.1);
END WHILE;
```

---

## Checklist d'optimisation de requêtes

### ✅ Avant d'optimiser

1. **Mesurer les performances actuelles** : EXPLAIN ANALYZE
2. **Identifier le goulot** : type=ALL, filesort, temporary ?
3. **Vérifier les statistiques** : ANALYZE TABLE si estimations erronées
4. **Indexer si nécessaire** : 80% des problèmes résolus ici

### ✅ Optimisations SQL

- [ ] **SELECT** : colonnes spécifiques, pas `*`
- [ ] **WHERE** : pas de fonctions sur colonnes indexées
- [ ] **JOIN** : index sur colonnes de jointure (FK)
- [ ] **ORDER BY** : index covering pour éviter filesort
- [ ] **GROUP BY** : index sur colonnes groupées
- [ ] **LIMIT** : toujours avec ORDER BY
- [ ] **Sous-requêtes** : remplacer par JOIN si plus efficace
- [ ] **DISTINCT** : remplacer par GROUP BY si possible
- [ ] **OR** : envisager UNION ALL si sélectif
- [ ] **NOT IN** : remplacer par LEFT JOIN ou NOT EXISTS

### ✅ Après optimisation

1. **Re-mesurer** : EXPLAIN ANALYZE sur requête optimisée
2. **Comparer** : temps avant vs après
3. **Valider** : résultats identiques
4. **Documenter** : pourquoi cette optimisation

---

## Exemples complets d'optimisation

### Exemple 1 : Rapport des ventes

**Requête initiale (8 secondes)** :

```sql
SELECT
    p.product_name,
    c.category_name,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(oi.quantity) as total_quantity,
    SUM(oi.quantity * oi.unit_price) as revenue
FROM order_items oi
INNER JOIN products p ON oi.product_id = p.product_id
INNER JOIN categories c ON p.category_id = c.category_id
INNER JOIN orders o ON oi.order_id = o.order_id
WHERE YEAR(o.order_date) = 2024
  AND o.status = 'completed'
GROUP BY p.product_id, p.product_name, c.category_name
ORDER BY revenue DESC;

EXPLAIN ANALYZE
-- type: ALL (orders)
-- Extra: Using temporary; Using filesort
-- actual time: 8200ms
```

**Problèmes identifiés** :
1. `YEAR(o.order_date)` empêche utilisation index
2. Pas d'index sur order_items.order_id
3. Tri sur revenue calculée (pas indexable)

**Optimisation** :

```sql
-- 1. Créer index manquants
CREATE INDEX idx_orders_date_status
ON orders(order_date, status);

CREATE INDEX idx_order_items_order_product
ON order_items(order_id, product_id, quantity, unit_price);

-- 2. Réécrire requête
SELECT
    p.product_name,
    c.category_name,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(oi.quantity) as total_quantity,
    SUM(oi.quantity * oi.unit_price) as revenue
FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
INNER JOIN categories c ON p.category_id = c.category_id
WHERE o.order_date >= '2024-01-01'
  AND o.order_date < '2025-01-01'
  AND o.status = 'completed'
GROUP BY p.product_id, p.product_name, c.category_name
ORDER BY revenue DESC;

EXPLAIN ANALYZE
-- type: range (orders), ref (autres)
-- key: idx_orders_date_status, idx_order_items_order_product
-- actual time: 320ms
-- Amélioration : x25
```

### Exemple 2 : Top clients du mois

**Requête initiale (5 secondes)** :

```sql
SELECT
    c.customer_id,
    c.name,
    (SELECT COUNT(*)
     FROM orders o
     WHERE o.customer_id = c.customer_id
       AND o.order_date >= DATE_FORMAT(NOW(), '%Y-%m-01')) as month_orders,
    (SELECT SUM(total_amount)
     FROM orders o
     WHERE o.customer_id = c.customer_id
       AND o.order_date >= DATE_FORMAT(NOW(), '%Y-%m-01')) as month_revenue
FROM customers c
WHERE c.status = 'active'
ORDER BY month_revenue DESC
LIMIT 100;

EXPLAIN ANALYZE
-- 2 sous-requêtes × 50k customers = 100k exécutions
-- actual time: 5200ms
```

**Optimisation** :

```sql
-- Réécrire avec JOIN
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id) as month_orders,
    COALESCE(SUM(o.total_amount), 0) as month_revenue
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
   AND o.order_date >= DATE_FORMAT(NOW(), '%Y-%m-01')
WHERE c.status = 'active'
GROUP BY c.customer_id, c.name
ORDER BY month_revenue DESC
LIMIT 100;

-- Index optimal
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date, total_amount);

EXPLAIN ANALYZE
-- type: ref (customers), ref (orders)
-- actual time: 180ms
-- Amélioration : x29
```

---

## ✅ Points clés à retenir

- **Éviter fonctions sur colonnes indexées** dans WHERE (utiliser colonnes générées)
- **SELECT colonnes spécifiques**, pas `*` (permet index covering)
- **Réécrire sous-requêtes** en JOIN si plus efficace
- **EXISTS vs IN** : EXISTS pour gros datasets
- **NOT IN** : remplacer par LEFT JOIN IS NULL ou NOT EXISTS
- **CTE pour lisibilité**, tables temporaires si besoin d'index
- **Pagination** : keyset pagination, pas OFFSET élevé
- **UNION ALL** plus rapide que UNION (si doublons acceptables)
- **Pré-calculer** valeurs fréquemment utilisées (colonnes générées)
- **Toujours mesurer** avec EXPLAIN ANALYZE avant/après

---

## 🔗 Ressources et références

- [📖 MariaDB Query Optimization](https://mariadb.com/kb/en/query-optimizations/)
- [📖 Subquery Optimization](https://mariadb.com/kb/en/subquery-optimizations/)
- [📖 WITH (Common Table Expressions)](https://mariadb.com/kb/en/with/)
- [📖 Generated Columns](https://mariadb.com/kb/en/generated-columns/)
- [📖 Optimizer Hints](https://mariadb.com/kb/en/optimizer-hints/)
- [🛠️ pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)

---

## ➡️ Section suivante

**5.9 Index covering et index-only scans** : Maximiser les performances en éliminant complètement les accès à la table de données, stratégies pour créer des index covering efficaces.

⏭️ [Index covering et index-only scans](/05-index-et-performance/09-index-covering.md)

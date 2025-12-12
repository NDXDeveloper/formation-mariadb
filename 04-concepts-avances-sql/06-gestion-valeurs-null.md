🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6 Gestion des valeurs NULL : Logique ternaire

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Maîtrise des requêtes de base, jointures, agrégations

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre la **logique ternaire SQL** (TRUE/FALSE/NULL)
- Maîtriser le comportement de **NULL** dans toutes les situations
- Utiliser correctement les **fonctions de gestion NULL** (COALESCE, NULLIF, IFNULL)
- Éviter les **pièges courants** liés aux valeurs NULL
- Appliquer les **best practices** pour la gestion de NULL
- Distinguer NULL de valeurs vides, zéro, ou chaînes vides
- Optimiser les requêtes impliquant des valeurs NULL

---

## Introduction

### Qu'est-ce que NULL en SQL ?

**NULL** n'est pas une valeur, c'est l'**absence de valeur**. C'est un marqueur spécial qui signifie :
- "Je ne sais pas"
- "Non applicable"
- "Donnée manquante"
- "Indéfini"

⚠️ **NULL ≠ 0** et **NULL ≠ ''** (chaîne vide)

```sql
-- NULL n'est pas égal à zéro
SELECT NULL = 0;        -- NULL (pas TRUE, pas FALSE)

-- NULL n'est pas égal à une chaîne vide
SELECT NULL = '';       -- NULL

-- NULL n'est même pas égal à lui-même !
SELECT NULL = NULL;     -- NULL (pas TRUE !)
```

### Pourquoi NULL est-il important ?

🔴 **Problèmes causés par une mauvaise gestion de NULL** :
- Résultats de requêtes incorrects
- Bugs subtils dans les jointures
- Agrégations faussées
- Conditions WHERE qui ne fonctionnent pas comme prévu
- Pertes de données dans les applications

✅ **Maîtriser NULL permet** :
- D'écrire des requêtes robustes
- D'éviter les bugs de production
- De gérer correctement les données manquantes
- D'améliorer la qualité des données

---

## La logique ternaire SQL

Le SQL utilise une **logique à trois valeurs** (ternaire) au lieu de la logique booléenne classique à deux valeurs.

### Les trois valeurs logiques

| Valeur | Signification |
|--------|---------------|
| **TRUE** | Vrai |
| **FALSE** | Faux |
| **NULL** | Inconnu / Indéterminé |

### Tables de vérité

#### Opérateur AND

| A | B | A AND B |
|---|---|---------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| TRUE | NULL | **NULL** |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |
| FALSE | NULL | **FALSE** |
| NULL | TRUE | **NULL** |
| NULL | FALSE | **FALSE** |
| NULL | NULL | **NULL** |

💡 **Règle** : AND retourne FALSE si un des opérandes est FALSE, sinon NULL si un est NULL, sinon TRUE.

```sql
SELECT
    TRUE AND TRUE,      -- TRUE
    TRUE AND FALSE,     -- FALSE
    TRUE AND NULL,      -- NULL ⚠️
    FALSE AND NULL,     -- FALSE (FALSE domine)
    NULL AND NULL;      -- NULL
```

#### Opérateur OR

| A | B | A OR B |
|---|---|--------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE |
| TRUE | NULL | **TRUE** |
| FALSE | TRUE | TRUE |
| FALSE | FALSE | FALSE |
| FALSE | NULL | **NULL** |
| NULL | TRUE | **TRUE** |
| NULL | FALSE | **NULL** |
| NULL | NULL | **NULL** |

💡 **Règle** : OR retourne TRUE si un des opérandes est TRUE, sinon NULL si un est NULL, sinon FALSE.

```sql
SELECT
    TRUE OR FALSE,      -- TRUE
    TRUE OR NULL,       -- TRUE (TRUE domine) ✅
    FALSE OR NULL,      -- NULL ⚠️
    NULL OR NULL;       -- NULL
```

#### Opérateur NOT

| A | NOT A |
|---|-------|
| TRUE | FALSE |
| FALSE | TRUE |
| NULL | **NULL** |

```sql
SELECT
    NOT TRUE,           -- FALSE
    NOT FALSE,          -- TRUE
    NOT NULL;           -- NULL ⚠️
```

---

## NULL dans les comparaisons

### Comparaisons classiques (=, !=, <, >)

**Toute comparaison impliquant NULL retourne NULL.**

```sql
-- Comparaisons d'égalité
SELECT
    NULL = NULL,        -- NULL (pas TRUE !)
    NULL = 5,           -- NULL
    5 = NULL,           -- NULL
    NULL != NULL,       -- NULL
    NULL <> 5;          -- NULL

-- Comparaisons d'ordre
SELECT
    NULL > 5,           -- NULL
    NULL < 5,           -- NULL
    NULL >= 5,          -- NULL
    NULL <= 5;          -- NULL
```

⚠️ **Conséquence critique** : `WHERE column = NULL` ne fonctionne JAMAIS !

```sql
CREATE TABLE test (id INT, value INT);
INSERT INTO test VALUES (1, 10), (2, NULL), (3, 20);

-- ❌ INCORRECT : Ne trouve RIEN (même pas les NULL)
SELECT * FROM test WHERE value = NULL;
-- Résultat : 0 lignes

-- ✅ CORRECT : Utiliser IS NULL
SELECT * FROM test WHERE value IS NULL;
-- Résultat : (2, NULL)
```

### Opérateurs IS NULL et IS NOT NULL

Les **seuls** opérateurs pour tester NULL correctement.

```sql
-- IS NULL : teste si la valeur est NULL
SELECT * FROM test WHERE value IS NULL;

-- IS NOT NULL : teste si la valeur n'est pas NULL
SELECT * FROM test WHERE value IS NOT NULL;

-- ❌ ERREUR FRÉQUENTE
SELECT * FROM test WHERE value != NULL;  -- ⚠️ Retourne 0 lignes !

-- ✅ CORRECT
SELECT * FROM test WHERE value IS NOT NULL;
```

### Opérateur <=> (NULL-safe equal)

MariaDB fournit un opérateur spécial pour comparer avec NULL.

```sql
-- <=> : Comparaison NULL-safe
SELECT
    NULL <=> NULL,      -- TRUE ✅
    NULL <=> 5,         -- FALSE
    5 <=> 5,            -- TRUE
    5 <=> NULL;         -- FALSE

-- Cas d'usage : comparaison de colonnes incluant NULL
SELECT *
FROM table1 t1
JOIN table2 t2 ON t1.col <=> t2.col;  -- ✅ Match aussi les NULL
```

💡 **Différence** :
- `col1 = col2` : NULL si l'un des deux est NULL
- `col1 <=> col2` : TRUE si les deux sont NULL, FALSE si un seul est NULL

---

## NULL dans les agrégations

### Comportement par défaut : NULL est ignoré

Les fonctions d'agrégation **ignorent les valeurs NULL**.

```sql
CREATE TABLE sales (id INT, amount DECIMAL(10,2));
INSERT INTO sales VALUES
    (1, 100.00),
    (2, NULL),
    (3, 200.00),
    (4, NULL),
    (5, 150.00);

SELECT
    COUNT(*) AS total_rows,              -- 5 (compte toutes les lignes)
    COUNT(amount) AS non_null_amounts,   -- 3 (ignore NULL) ⚠️
    SUM(amount) AS total,                -- 450.00 (100+200+150)
    AVG(amount) AS average,              -- 150.00 (450/3, pas 450/5) ⚠️
    MIN(amount) AS minimum,              -- 100.00
    MAX(amount) AS maximum;              -- 200.00
```

**Résultat** :
```
+------------+-------------------+--------+---------+---------+---------+
| total_rows | non_null_amounts  | total  | average | minimum | maximum |
+------------+-------------------+--------+---------+---------+---------+
|          5 |                 3 | 450.00 |  150.00 |  100.00 |  200.00 |
+------------+-------------------+--------+---------+---------+---------+
```

⚠️ **Attention** :
- `COUNT(*)` compte toutes les lignes (y compris celles avec NULL)
- `COUNT(column)` ignore les NULL
- `AVG(column)` = `SUM(column) / COUNT(column)`, donc ignore les NULL

### Piège : Moyenne vs moyenne incluant NULL

```sql
-- Moyenne "pure" (ignore NULL)
SELECT AVG(amount) FROM sales;  -- 150.00

-- Moyenne incluant NULL comme zéro
SELECT SUM(amount) / COUNT(*) FROM sales;  -- 90.00 (450/5)

-- Ou avec COALESCE
SELECT AVG(COALESCE(amount, 0)) FROM sales;  -- 90.00
```

💡 **Choisir selon le contexte métier** :
- Moyenne de ventes réalisées → `AVG(amount)` (ignore NULL)
- Moyenne de performance quotidienne → traiter NULL comme 0

---

## NULL dans les jointures

### LEFT JOIN et NULL

Les LEFT JOIN introduisent des NULL pour les lignes sans correspondance.

```sql
CREATE TABLE customers (id INT, name VARCHAR(50));
CREATE TABLE orders (id INT, customer_id INT, amount DECIMAL(10,2));

INSERT INTO customers VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol');
INSERT INTO orders VALUES (1, 1, 100), (2, 1, 150), (3, 2, 200);

-- LEFT JOIN : Carol n'a pas de commandes
SELECT
    c.id,
    c.name,
    o.id AS order_id,
    o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

**Résultat** :
```
+----+-------+----------+--------+
| id | name  | order_id | amount |
+----+-------+----------+--------+
|  1 | Alice |        1 | 100.00 |
|  1 | Alice |        2 | 150.00 |
|  2 | Bob   |        3 | 200.00 |
|  3 | Carol |     NULL |   NULL |  ⚠️
+----+-------+----------+--------+
```

### Filtrer après LEFT JOIN : Piège courant

```sql
-- ❌ ERREUR : Le WHERE transforme le LEFT JOIN en INNER JOIN
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.amount > 100;  -- ⚠️ Exclut Carol car o.amount est NULL

-- Résultat : Bob (200) et Alice (150) seulement, Carol disparaît !

-- ✅ CORRECT : Utiliser IS NOT NULL ou filtrer dans le ON
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.amount > 100 OR o.amount IS NULL;  -- ✅ Inclut Carol

-- OU mieux : filtrer dans le ON
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.amount > 100;
```

💡 **Règle** : Après un LEFT JOIN, toujours considérer les NULL dans les WHERE.

---

## Fonctions de gestion de NULL

### IFNULL(expr1, expr2)

Retourne `expr2` si `expr1` est NULL, sinon `expr1`.

```sql
SELECT
    IFNULL(NULL, 'default'),        -- 'default'
    IFNULL(10, 'default'),          -- 10
    IFNULL('value', 'default');     -- 'value'

-- Cas d'usage : remplacer NULL par 0 dans les calculs
SELECT
    id,
    amount,
    IFNULL(amount, 0) AS amount_with_default
FROM sales;
```

**Résultat** :
```
+----+--------+----------------------+
| id | amount | amount_with_default  |
+----+--------+----------------------+
|  1 | 100.00 |               100.00 |
|  2 |   NULL |                 0.00 |
|  3 | 200.00 |               200.00 |
+----+--------+----------------------+
```

### COALESCE(expr1, expr2, ..., exprN)

Retourne la première valeur non-NULL.

```sql
SELECT
    COALESCE(NULL, NULL, 'third', 'fourth'),  -- 'third'
    COALESCE(NULL, 10, 20),                   -- 10
    COALESCE('first', 'second');              -- 'first'

-- Cas d'usage : cascade de valeurs par défaut
SELECT
    id,
    COALESCE(
        mobile_phone,
        home_phone,
        work_phone,
        'No phone'
    ) AS contact_phone
FROM contacts;
```

💡 **COALESCE vs IFNULL** :
- `IFNULL(a, b)` : exactement 2 arguments
- `COALESCE(a, b, c, ...)` : N arguments, plus flexible

### NULLIF(expr1, expr2)

Retourne NULL si `expr1 = expr2`, sinon `expr1`.

```sql
SELECT
    NULLIF(10, 10),         -- NULL (égaux)
    NULLIF(10, 20),         -- 10 (différents)
    NULLIF('', '');         -- NULL

-- Cas d'usage : éviter division par zéro
SELECT
    revenue,
    cost,
    revenue / NULLIF(cost, 0) AS profit_ratio  -- ✅ NULL si cost=0
FROM finances;

-- Autre cas : convertir chaînes vides en NULL
SELECT
    NULLIF(TRIM(user_input), '') AS cleaned_value;
```

### IF(condition, valeur_si_vrai, valeur_si_faux)

Évaluation conditionnelle (similaire à CASE).

```sql
SELECT
    IF(amount IS NULL, 'No data', 'Has data') AS status,
    IF(amount > 100, 'High', 'Low') AS category
FROM sales;
```

---

## NULL dans les opérations arithmétiques

### Propagation de NULL

**Toute opération arithmétique avec NULL retourne NULL.**

```sql
SELECT
    NULL + 10,          -- NULL
    100 - NULL,         -- NULL
    NULL * 5,           -- NULL
    20 / NULL,          -- NULL
    NULL % 3;           -- NULL

-- Conséquence dans les calculs
SELECT
    quantity,
    price,
    quantity * price AS total  -- NULL si quantity OU price est NULL
FROM order_items;
```

### Solution : Remplacer NULL avant calcul

```sql
-- ✅ Remplacer NULL par une valeur par défaut
SELECT
    COALESCE(quantity, 0) * COALESCE(price, 0) AS total
FROM order_items;

-- Ou décider du comportement :
SELECT
    CASE
        WHEN quantity IS NULL OR price IS NULL THEN NULL
        ELSE quantity * price
    END AS total
FROM order_items;
```

---

## NULL dans les chaînes de caractères

### Concaténation avec NULL

```sql
-- CONCAT : retourne NULL si un argument est NULL
SELECT CONCAT('Hello', NULL, 'World');  -- NULL ⚠️

-- CONCAT_WS : ignore les NULL
SELECT CONCAT_WS(' ', 'Hello', NULL, 'World');  -- 'Hello World' ✅

-- Cas d'usage : construction d'adresses
SELECT
    CONCAT_WS(', ',
        street,
        NULLIF(apartment, ''),
        city,
        zipcode
    ) AS full_address
FROM addresses;
```

### Comparaison de chaînes

```sql
-- NULL n'est pas une chaîne vide
SELECT
    NULL = '',              -- NULL
    NULL <=> '',            -- FALSE
    COALESCE(NULL, '') = '' -- TRUE (NULL devient '')
;

-- Longueur de NULL
SELECT
    LENGTH(NULL),           -- NULL
    LENGTH(''),             -- 0
    LENGTH('abc');          -- 3
```

---

## NULL dans GROUP BY et DISTINCT

### GROUP BY avec NULL

Les valeurs NULL sont **regroupées ensemble**.

```sql
CREATE TABLE products (category VARCHAR(50), price DECIMAL(10,2));
INSERT INTO products VALUES
    ('Electronics', 100),
    ('Electronics', 150),
    (NULL, 200),
    (NULL, 250),
    ('Books', 50);

SELECT
    category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price
FROM products
GROUP BY category;
```

**Résultat** :
```
+-------------+---------------+-----------+
| category    | product_count | avg_price |
+-------------+---------------+-----------+
| Electronics |             2 |    125.00 |
| NULL        |             2 |    225.00 |  ⚠️ NULL groupé
| Books       |             1 |     50.00 |
+-------------+---------------+-----------+
```

💡 **NULL forme son propre groupe.**

### DISTINCT avec NULL

```sql
SELECT DISTINCT category FROM products;
```

**Résultat** :
```
+-------------+
| category    |
+-------------+
| Electronics |
| NULL        |  ⚠️ NULL considéré comme une valeur distincte
| Books       |
+-------------+
```

---

## NULL dans ORDER BY

### Ordre de tri par défaut

MariaDB place les NULL **en premier** par défaut (ASC) ou **en dernier** (DESC).

```sql
CREATE TABLE items (id INT, priority INT);
INSERT INTO items VALUES (1, 10), (2, NULL), (3, 20), (4, NULL), (5, 5);

-- ASC : NULL en premier
SELECT * FROM items ORDER BY priority ASC;
```

**Résultat** :
```
+----+----------+
| id | priority |
+----+----------+
|  2 |     NULL |  ⚠️
|  4 |     NULL |  ⚠️
|  5 |        5 |
|  1 |       10 |
|  3 |       20 |
+----+----------+
```

```sql
-- DESC : NULL en dernier
SELECT * FROM items ORDER BY priority DESC;
```

**Résultat** :
```
+----+----------+
| id | priority |
+----+----------+
|  3 |       20 |
|  1 |       10 |
|  5 |        5 |
|  2 |     NULL |  ⚠️
|  4 |     NULL |  ⚠️
+----+----------+
```

### Contrôler l'ordre des NULL

```sql
-- Forcer NULL en dernier avec ASC
SELECT *
FROM items
ORDER BY priority IS NULL, priority ASC;

-- Forcer NULL en premier avec DESC
SELECT *
FROM items
ORDER BY priority IS NULL DESC, priority DESC;

-- Ou avec CASE
SELECT *
FROM items
ORDER BY CASE WHEN priority IS NULL THEN 1 ELSE 0 END, priority;
```

---

## Pièges courants et solutions

### Piège 1 : WHERE avec NULL

```sql
-- ❌ INCORRECT
SELECT * FROM users WHERE age = NULL;  -- 0 résultats
SELECT * FROM users WHERE age != NULL; -- 0 résultats

-- ✅ CORRECT
SELECT * FROM users WHERE age IS NULL;
SELECT * FROM users WHERE age IS NOT NULL;
```

### Piège 2 : NOT IN avec NULL

```sql
CREATE TABLE allowed (value INT);
INSERT INTO allowed VALUES (1), (2), (NULL);

-- ❌ PROBLÈME : retourne 0 lignes si NULL présent
SELECT * FROM items WHERE id NOT IN (SELECT value FROM allowed);

-- Pourquoi ?
-- id NOT IN (1, 2, NULL)
-- ≡ id != 1 AND id != 2 AND id != NULL
-- ≡ id != 1 AND id != 2 AND NULL
-- ≡ NULL (toujours)

-- ✅ SOLUTION 1 : Filtrer les NULL
SELECT * FROM items
WHERE id NOT IN (SELECT value FROM allowed WHERE value IS NOT NULL);

-- ✅ SOLUTION 2 : Utiliser NOT EXISTS
SELECT * FROM items i
WHERE NOT EXISTS (
    SELECT 1 FROM allowed a WHERE a.value = i.id
);
```

💡 **Règle** : Éviter `NOT IN` avec sous-requêtes qui peuvent contenir NULL.

### Piège 3 : COUNT(*) vs COUNT(column)

```sql
-- Comptage différent
SELECT
    COUNT(*) AS all_rows,        -- 5
    COUNT(amount) AS non_null    -- 3
FROM sales;

-- Attention dans les pourcentages
SELECT
    -- ❌ INCORRECT : division par COUNT(*)
    100.0 * COUNT(amount) / COUNT(*) AS pct_non_null,  -- 60%

    -- ✅ CORRECT : comprendre ce qu'on mesure
    100.0 * COUNT(amount) / COUNT(*) AS pct_with_amount,  -- 60%
    100.0 * (COUNT(*) - COUNT(amount)) / COUNT(*) AS pct_null  -- 40%
FROM sales;
```

### Piège 4 : Agrégation sur colonnes avec NULL

```sql
-- Moyenne biaisée si on ignore les NULL
SELECT AVG(satisfaction_score) FROM surveys;  -- Moyenne de ceux qui ont répondu

-- Pour inclure les NULL comme "0" ou "neutre"
SELECT AVG(COALESCE(satisfaction_score, 0)) FROM surveys;

-- Ou être explicite
SELECT
    AVG(satisfaction_score) AS avg_responders,
    COUNT(*) AS total_surveys,
    COUNT(satisfaction_score) AS completed_surveys,
    COUNT(*) - COUNT(satisfaction_score) AS no_response
FROM surveys;
```

### Piège 5 : UNIQUE avec NULL

En SQL standard, **NULL n'est pas égal à NULL**, donc on peut avoir **plusieurs NULL** dans une colonne UNIQUE.

```sql
CREATE TABLE emails (
    user_id INT,
    email VARCHAR(100) UNIQUE
);

INSERT INTO emails VALUES
    (1, 'alice@example.com'),
    (2, NULL),
    (3, NULL);  -- ✅ Accepté ! Plusieurs NULL possibles

-- Les deux INSERT de NULL passent car NULL != NULL
```

💡 **Comportement MariaDB** : Plusieurs NULL autorisés dans UNIQUE (conforme SQL standard).

### Piège 6 : CASE WHEN avec NULL

```sql
-- ❌ INCORRECT : NULL n'est jamais égal à NULL
SELECT
    CASE status
        WHEN NULL THEN 'Unknown'  -- ⚠️ Ne match jamais
        WHEN 'active' THEN 'Active'
        ELSE 'Other'
    END AS status_label
FROM users;

-- ✅ CORRECT : Utiliser IS NULL
SELECT
    CASE
        WHEN status IS NULL THEN 'Unknown'  -- ✅
        WHEN status = 'active' THEN 'Active'
        ELSE 'Other'
    END AS status_label
FROM users;
```

---

## Best practices

### 1. Conception de schéma : Éviter NULL quand possible

```sql
-- ❌ Conception avec beaucoup de NULL
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,  -- Peut être NULL ?
    status VARCHAR(20),  -- Peut être NULL ?
    total DECIMAL(10,2)  -- Peut être NULL ?
);

-- ✅ Meilleure conception : NOT NULL + valeurs par défaut
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT NOT NULL,  -- ✅ Obligatoire
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- ✅ Défaut
    total DECIMAL(10,2) NOT NULL DEFAULT 0.00  -- ✅ Défaut à 0
);
```

💡 **Règle** : Utiliser NULL **uniquement quand l'absence de valeur a un sens métier**.

### 2. Documenter la signification de NULL

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    -- NULL = démission non encore validée
    resignation_date DATE NULL,
    -- NULL = salaire non négocié (stagiaire/bénévole)
    salary DECIMAL(10,2) NULL,
    -- NULL = pas de manager (CEO)
    manager_id INT NULL
);
```

### 3. Utiliser COALESCE pour les valeurs par défaut

```sql
-- ✅ Valeurs par défaut cohérentes
SELECT
    id,
    COALESCE(name, 'Unknown') AS name,
    COALESCE(email, 'no-email@example.com') AS email,
    COALESCE(phone, 'N/A') AS phone
FROM contacts;
```

### 4. Filtrer les NULL explicitement

```sql
-- ✅ Être explicite sur la gestion de NULL
SELECT * FROM products
WHERE price > 100
  AND category IS NOT NULL;  -- ✅ Explicite

-- Plutôt que de compter sur le comportement implicite
```

### 5. Comprendre les agrégations

```sql
-- ✅ Documenter l'intention
SELECT
    COUNT(*) AS total_customers,
    COUNT(email) AS customers_with_email,
    COUNT(*) - COUNT(email) AS customers_without_email,
    ROUND(100.0 * COUNT(email) / COUNT(*), 2) AS email_completion_pct
FROM customers;
```

### 6. Tester les cas limites

```sql
-- ✅ Tester explicitement avec NULL
SELECT
    id,
    amount,
    CASE
        WHEN amount IS NULL THEN 'Missing'
        WHEN amount = 0 THEN 'Zero'
        WHEN amount > 0 THEN 'Positive'
        ELSE 'Negative'
    END AS amount_status
FROM transactions;
```

---

## Cas d'usage pratiques

### Exemple 1 : Rapport avec valeurs manquantes

```sql
WITH customer_stats AS (
    SELECT
        c.id,
        c.name,
        COUNT(o.id) AS order_count,
        SUM(o.total) AS total_spent,
        MAX(o.order_date) AS last_order_date
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, c.name
)
SELECT
    name,
    order_count,
    COALESCE(total_spent, 0) AS lifetime_value,
    COALESCE(
        DATE_FORMAT(last_order_date, '%Y-%m-%d'),
        'Never ordered'
    ) AS last_order,
    CASE
        WHEN last_order_date IS NULL THEN 'Never purchased'
        WHEN last_order_date < DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY) THEN 'Inactive'
        WHEN last_order_date < DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY) THEN 'At risk'
        ELSE 'Active'
    END AS status
FROM customer_stats
ORDER BY total_spent DESC NULLS LAST;
```

### Exemple 2 : Calcul de KPI avec données incomplètes

```sql
SELECT
    product_id,
    product_name,

    -- Revenue (NULL si aucune vente)
    SUM(quantity * price) AS revenue,

    -- Unités vendues (0 si NULL)
    COALESCE(SUM(quantity), 0) AS units_sold,

    -- Prix moyen (NULL si aucune vente)
    AVG(price) AS avg_price,

    -- Completion rate (% de champs renseignés)
    100.0 * (
        COUNT(product_name) +
        COUNT(description) +
        COUNT(image_url)
    ) / (3 * COUNT(*)) AS data_completeness_pct
FROM products
LEFT JOIN sales ON products.id = sales.product_id
GROUP BY product_id, product_name;
```

### Exemple 3 : Gestion des références optionnelles

```sql
-- Commandes avec shipping optionnel
SELECT
    o.id AS order_id,
    o.customer_id,
    o.total,

    -- Adresse de livraison (peut être NULL = retrait en magasin)
    COALESCE(
        CONCAT_WS(', ',
            sa.street,
            sa.city,
            sa.zipcode
        ),
        'Store pickup'
    ) AS shipping_address,

    -- Frais de port (NULL = gratuit ou retrait)
    COALESCE(o.shipping_cost, 0) AS shipping_cost,

    -- Total avec frais
    o.total + COALESCE(o.shipping_cost, 0) AS total_with_shipping
FROM orders o
LEFT JOIN shipping_addresses sa ON o.shipping_address_id = sa.id;
```

---

## Performance et index avec NULL

### NULL et index B-Tree

Les valeurs NULL **sont indexées** dans MariaDB (contrairement à certains autres SGBD).

```sql
-- Index inclut les NULL
CREATE INDEX idx_email ON users(email);

-- Cette requête utilise l'index
SELECT * FROM users WHERE email IS NULL;

-- Cette requête aussi
SELECT * FROM users WHERE email IS NOT NULL;
```

### Optimisation : Filtrer NULL avec index

```sql
-- ✅ Index utilisé
EXPLAIN SELECT * FROM orders WHERE shipping_date IS NULL;

-- Index partiel (MariaDB 10.3+) pour exclure NULL
-- Note : Pas supporté directement, mais on peut simuler avec colonnes générées
ALTER TABLE orders
ADD COLUMN has_shipping BOOLEAN AS (shipping_date IS NOT NULL) STORED;

CREATE INDEX idx_has_shipping ON orders(has_shipping);
```

---

## ✅ Points clés à retenir

- 🎯 **NULL ≠ valeur** : NULL représente l'absence de valeur, pas zéro ni chaîne vide
- 🔺 **Logique ternaire** : TRUE, FALSE, NULL → Toute comparaison avec NULL retourne NULL
- ⚠️ **NULL = NULL retourne NULL** : Utiliser `IS NULL` ou `<=>` pour tester NULL
- 🔢 **Agrégations** : `COUNT(*)` vs `COUNT(column)` → COUNT ignore les NULL
- 🔗 **Jointures** : LEFT JOIN introduit NULL, attention aux filtres WHERE ensuite
- 📊 **GROUP BY** : NULL forme son propre groupe
- 🛠️ **Fonctions** : `COALESCE`, `IFNULL`, `NULLIF` pour gérer NULL
- ➕ **Arithmétique** : Toute opération avec NULL → NULL
- 🚫 **NOT IN** : Éviter avec sous-requêtes pouvant contenir NULL
- 📐 **Best practice** : Éviter NULL quand possible, documenter sa signification

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 NULL Values](https://mariadb.com/kb/en/null-values/)
- [📖 IS NULL Operator](https://mariadb.com/kb/en/is-null/)
- [📖 COALESCE](https://mariadb.com/kb/en/coalesce/)
- [📖 IFNULL](https://mariadb.com/kb/en/ifnull/)
- [📖 NULLIF](https://mariadb.com/kb/en/nullif/)

### Standards SQL
- [SQL:1999](https://en.wikipedia.org/wiki/SQL:1999) - Définition de la logique ternaire

### Articles recommandés
- [Modern SQL: NULL](https://modern-sql.com/concept/null) - Explications détaillées
- [Use The Index, Luke: NULL](https://use-the-index-luke.com/sql/where-clause/null) - Perspective performance

---

## ➡️ Section suivante

**[4.7 JSON dans MariaDB](./07-json-mariadb.md)** : Découvrez comment stocker et manipuler des données JSON dans MariaDB, avec les fonctions natives, l'indexation de colonnes virtuelles, et les cas d'usage pratiques.

---


⏭️ [JSON dans MariaDB](/04-concepts-avances-sql/07-json-mariadb.md)

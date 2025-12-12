🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.7 Analyse des plans d'exécution (EXPLAIN, EXPLAIN ANALYZE)

> **Niveau** : Intermédiaire
> **Durée estimée** : 2.5 heures
> **Prérequis** : Section 5.1 à 5.6 (Index et stratégies d'indexation)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Interpréter tous les champs d'un plan d'exécution EXPLAIN
- Distinguer les différents types d'accès (ALL, index, range, ref, eq_ref, const)
- Analyser la colonne Extra pour identifier les optimisations et problèmes
- Utiliser EXPLAIN ANALYZE pour obtenir des métriques réelles d'exécution
- Identifier les goulots d'étranglement de performance dans les requêtes
- Comparer différentes stratégies d'optimisation avec des données chiffrées
- Diagnostiquer les problèmes courants (full scans, filesort, temporary tables)

---

## Introduction

L'analyse des **plans d'exécution** est l'outil fondamental pour comprendre comment MariaDB traite vos requêtes et identifier les opportunités d'optimisation. `EXPLAIN` révèle la stratégie choisie par l'optimiseur (optimizer) pour exécuter une requête, tandis que `EXPLAIN ANALYZE` (MariaDB 10.6+) fournit en plus les **temps réels** d'exécution.

💡 **Analogie** : EXPLAIN est comme un GPS qui vous montre l'itinéraire prévu avant de partir, tandis qu'EXPLAIN ANALYZE est le trajet réel avec les temps de parcours effectifs.

⚠️ **Point clé** : Un plan d'exécution n'est qu'une **estimation** basée sur les statistiques. Les performances réelles peuvent varier, d'où l'importance d'EXPLAIN ANALYZE.

---

## EXPLAIN : Structure et colonnes

### Syntaxe de base

```sql
-- Syntaxe standard
EXPLAIN SELECT * FROM users WHERE country = 'FR';

-- Format étendu (plus d'informations)
EXPLAIN EXTENDED SELECT * FROM users WHERE country = 'FR';
SHOW WARNINGS; -- Affiche la requête réécrite par l'optimizer

-- Format JSON (MariaDB 10.1+)
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE country = 'FR';
```

### Structure du résultat EXPLAIN

```sql
-- Exemple de requête
EXPLAIN
SELECT u.user_id, u.username, o.order_id, o.total_amount
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
WHERE u.country = 'FR'
  AND o.order_date >= '2024-01-01'
ORDER BY o.order_date DESC
LIMIT 10;
```

**Colonnes du résultat** :

| Colonne | Description |
|---------|-------------|
| `id` | Identifiant de la partie SELECT (requêtes imbriquées) |
| `select_type` | Type de SELECT (SIMPLE, PRIMARY, SUBQUERY, etc.) |
| `table` | Table accédée |
| `type` | Type d'accès (ALL, index, range, ref, eq_ref, const) |
| `possible_keys` | Index potentiellement utilisables |
| `key` | Index réellement utilisé (NULL si aucun) |
| `key_len` | Longueur de la clé d'index utilisée |
| `ref` | Colonnes/constantes comparées à l'index |
| `rows` | Estimation du nombre de lignes examinées |
| `filtered` | Pourcentage de lignes filtrées (MariaDB 10.0+) |
| `Extra` | Informations supplémentaires importantes |

---

## Colonne `type` : Types d'accès aux données

La colonne `type` indique **comment** MariaDB accède aux données. C'est l'indicateur le plus important pour évaluer les performances.

### Hiérarchie des types (du meilleur au pire)

```
system > const > eq_ref > ref > fulltext > ref_or_null >
index_merge > unique_subquery > index_subquery > range >
index > ALL
```

### `system` : Table système avec 1 seule ligne

```sql
-- Rare en pratique, tables système ou tables vides
EXPLAIN SELECT * FROM (SELECT 1) AS t;

-- type: system
-- rows: 1
-- Le plus rapide possible (table en mémoire, 1 ligne)
```

### `const` : Recherche par clé primaire ou unique

```sql
-- Accès par PRIMARY KEY
EXPLAIN SELECT * FROM users WHERE user_id = 12345;

-- Résultat :
-- type: const
-- key: PRIMARY
-- rows: 1
--
-- ✅ Optimal : accès direct à 1 ligne unique
```

**Caractéristiques** :
- Recherche sur PRIMARY KEY ou UNIQUE avec valeur constante
- MariaDB lit **exactement 1 ligne**
- Temps d'accès : O(1) via l'index clustered

### `eq_ref` : Jointure sur clé primaire/unique

```sql
-- Jointure sur PRIMARY KEY
EXPLAIN
SELECT u.username, o.order_id
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id
WHERE o.order_id = 1000;

-- Résultat :
-- id | table | type   | key     | rows
-- 1  | o     | const  | PRIMARY | 1
-- 1  | u     | eq_ref | PRIMARY | 1
--
-- ✅ Excellent : une seule ligne lue de 'users' par ligne de 'orders'
```

**Caractéristiques** :
- Jointure où **au plus 1 ligne** correspond dans la table jointe
- Utilise PRIMARY KEY ou UNIQUE INDEX
- Performance quasi-optimale pour jointures

### `ref` : Recherche par index non-unique

```sql
-- Index non-unique
CREATE INDEX idx_users_country ON users(country);

EXPLAIN SELECT * FROM users WHERE country = 'FR';

-- Résultat :
-- type: ref
-- key: idx_users_country
-- rows: 5000
--
-- ✅ Bon : utilise index, multiple lignes correspondantes
```

**Caractéristiques** :
- Recherche sur index non-unique
- Retourne **plusieurs lignes** avec la même valeur
- Très courant pour colonnes comme status, country, category_id

### `ref_or_null` : Recherche avec NULL inclus

```sql
EXPLAIN
SELECT * FROM orders
WHERE customer_id = 123 OR customer_id IS NULL;

-- type: ref_or_null
-- Recherche index + scan pour NULL
```

### `range` : Recherche par plage de valeurs

```sql
-- Plages : >, <, BETWEEN, IN, LIKE 'prefix%'
EXPLAIN
SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

-- Résultat :
-- type: range
-- key: idx_orders_date
-- rows: 50000
--
-- ✅ Bon : parcours d'une portion de l'index
```

**Opérateurs déclenchant `range`** :
- `>`, `>=`, `<`, `<=`
- `BETWEEN`
- `IN (val1, val2, ...)`
- `LIKE 'prefix%'` (wildcard à la fin)

### `index` : Full index scan

```sql
-- Scan complet de l'index (pas de la table)
EXPLAIN
SELECT user_id FROM users ORDER BY user_id;

-- Résultat :
-- type: index
-- key: PRIMARY
-- rows: 1000000
-- Extra: Using index
--
-- ⚠️ Acceptable si index est petit et covering
```

**Caractéristiques** :
- Parcourt **tout l'index** (pas la table)
- Plus rapide que `ALL` si index couvre les colonnes SELECT
- Peut être lent sur très grands index

### `ALL` : Full table scan

```sql
-- Aucun index utilisable
EXPLAIN SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- Résultat :
-- type: ALL
-- key: NULL
-- rows: 1000000
--
-- ❌ Problème : scan de toute la table
```

**Caractéristiques** :
- Lit **toutes les lignes** de la table
- Très lent sur grandes tables (>100k lignes)
- Acceptable uniquement sur petites tables (<1000 lignes)

💡 **Règle** : `ALL` sur table de plus de 10 000 lignes = opportunité d'optimisation urgente.

### `index_merge` : Combinaison de plusieurs index

```sql
-- Utilise plusieurs index et fusionne les résultats
CREATE INDEX idx_status ON orders(status);
CREATE INDEX idx_customer ON orders(customer_id);

EXPLAIN
SELECT * FROM orders
WHERE status = 'pending' OR customer_id = 123;

-- Résultat :
-- type: index_merge
-- key: idx_status,idx_customer
-- Extra: Using union(idx_status,idx_customer); Using where
--
-- ⚠️ Fonctionne mais moins efficace qu'un index composite
```

**Types d'index merge** :
- `union` : OR entre conditions
- `intersection` : AND entre conditions
- `sort_union` : OR avec tri préalable

💡 **Optimisation** : Remplacer par un index composite si la combinaison est fréquente.

---

## Colonne `Extra` : Informations critiques

La colonne `Extra` contient des indicateurs sur le traitement de la requête. C'est souvent là que se cachent les problèmes de performance.

### Indicateurs positifs (✅)

#### `Using index` : Index covering (optimal)

```sql
CREATE INDEX idx_users_email ON users(email);

EXPLAIN SELECT email FROM users WHERE email LIKE 'john%';

-- Extra: Using index
-- ✅ Parfait : lecture uniquement de l'index, pas d'accès table
```

**Signification** : Toutes les colonnes nécessaires sont dans l'index → **index-only scan**.

#### `Using index condition` : Index Condition Pushdown (ICP)

```sql
CREATE INDEX idx_users_country_city ON users(country, city);

EXPLAIN
SELECT * FROM users
WHERE country = 'FR' AND city LIKE 'Par%';

-- Extra: Using index condition
-- ✅ Bien : filtrage poussé au niveau de l'index
```

**Signification** : MariaDB évalue les conditions WHERE directement au niveau de l'index, réduisant les accès table.

### Indicateurs neutres (⚠️)

#### `Using where` : Filtrage après lecture

```sql
EXPLAIN
SELECT * FROM users
WHERE country = 'FR' AND age > 25;
-- Index seulement sur country

-- Extra: Using where
-- ⚠️ Filtre 'age' appliqué après lecture via index 'country'
```

**Signification** : Certaines conditions WHERE ne peuvent pas être évaluées via l'index.

### Indicateurs problématiques (❌)

#### `Using filesort` : Tri en mémoire/disque

```sql
EXPLAIN
SELECT * FROM articles
WHERE category_id = 5
ORDER BY published_at DESC;
-- Pas d'index sur (category_id, published_at)

-- Extra: Using filesort
-- ❌ Problème : tri coûteux de potentiellement milliers de lignes
```

**Impact** :
- Copie des données dans sort buffer
- Si trop volumineux → tri sur disque (très lent)
- Temps : +500ms à +5s selon volumétrie

**Solution** : Créer index couvrant WHERE + ORDER BY.

#### `Using temporary` : Table temporaire créée

```sql
EXPLAIN
SELECT category_id, COUNT(*)
FROM articles
GROUP BY category_id
ORDER BY COUNT(*) DESC;

-- Extra: Using temporary; Using filesort
-- ❌ Problème : création table temporaire + tri
```

**Impact** :
- Copie des données dans table temporaire en mémoire/disque
- Puis tri de cette table
- Doublement coûteux

**Causes fréquentes** :
- GROUP BY sur colonnes non indexées
- DISTINCT sur plusieurs colonnes
- ORDER BY différent de GROUP BY

#### `Using join buffer` : Jointure sans index

```sql
EXPLAIN
SELECT * FROM orders o
INNER JOIN customers c ON o.customer_email = c.email
WHERE o.status = 'pending';
-- Pas d'index sur orders.customer_email

-- Extra: Using join buffer (Block Nested Loop)
-- ❌ Problème : jointure inefficace, parcours cartésien partiel
```

**Impact** : Algorithme de jointure lent (Block Nested Loop).

**Solution** : Créer index sur colonne de jointure.

#### `Impossible WHERE` : Condition toujours fausse

```sql
EXPLAIN SELECT * FROM users WHERE 1 = 0;

-- Extra: Impossible WHERE
-- Requête n'est pas exécutée (optimiseur détecte)
```

#### `No matching rows after partition pruning`

```sql
-- Table partitionnée par année
EXPLAIN SELECT * FROM logs WHERE log_date = '2020-01-01';

-- Extra: No matching rows after partition pruning
-- Aucune partition ne contient cette date
```

---

## EXPLAIN ANALYZE : Métriques réelles

**Disponible depuis MariaDB 10.6**, `EXPLAIN ANALYZE` exécute réellement la requête et fournit des **temps réels** d'exécution.

### Syntaxe et exemple

```sql
-- Syntaxe
EXPLAIN ANALYZE
SELECT u.username, COUNT(o.order_id) as order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE u.country = 'FR'
GROUP BY u.user_id, u.username
ORDER BY order_count DESC
LIMIT 10;
```

**Résultat (format tree)** :

```
-> Limit: 10 row(s)
    (cost=25630 rows=10)
    (actual time=45.2..45.4 rows=10 loops=1)
  -> Sort: order_count DESC, limit input to 10 row(s) per chunk
      (cost=25630 rows=5000)
      (actual time=45.2..45.3 rows=10 loops=1)
    -> Table scan on <temporary>
        (cost=2798..8298 rows=5000)
        (actual time=43.1..44.8 rows=5000 loops=1)
      -> Aggregate using temporary table
          (cost=28428 rows=5000)
          (actual time=43.1..44.5 rows=5000 loops=1)
        -> Nested loop left join
            (cost=23428 rows=5000)
            (actual time=0.156..32.4 rows=45000 loops=1)
          -> Index lookup on u using idx_users_country (country='FR')
              (cost=2178 rows=5000)
              (actual time=0.124..8.5 rows=5000 loops=1)
          -> Index lookup on o using idx_orders_user (user_id=u.user_id)
              (cost=3.25 rows=9)
              (actual time=0.003..0.004 rows=9 loops=5000)
```

### Métriques clés

| Métrique | Description |
|----------|-------------|
| `cost` | Coût estimé par l'optimiseur (unités arbitraires) |
| `rows` | Nombre de lignes **estimé** |
| `actual time` | **Temps réel** en millisecondes (format: début..fin) |
| `actual rows` | Nombre de lignes **réellement** traitées |
| `loops` | Nombre de fois que l'opération est exécutée |

### Interpréter les métriques

```sql
-> Index lookup on orders using idx_customer (customer_id=123)
    (cost=125 rows=500)
    (actual time=0.15..2.3 rows=450 loops=1)
```

**Analyse** :
- **cost=125** : Coût estimé
- **rows=500** : Optimiseur s'attend à 500 lignes
- **actual time=0.15..2.3** :
  - `0.15ms` : temps pour trouver la première ligne
  - `2.3ms` : temps total pour lire toutes les lignes
- **actual rows=450** : 450 lignes réellement trouvées (assez proche de l'estimation)
- **loops=1** : Opération exécutée 1 fois

💡 **Temps total réel** : `actual time × loops` = `2.3ms × 1` = **2.3ms**

### Détecter les problèmes avec EXPLAIN ANALYZE

#### Problème 1 : Estimations erronées

```sql
-> Table scan on orders
    (cost=10000 rows=100)
    (actual time=0.5..850 rows=500000 loops=1)
```

⚠️ **Alerte** : `rows=100` vs `actual rows=500000` → estimation **5000× incorrecte** !

**Cause** : Statistiques d'index obsolètes.

**Solution** :
```sql
ANALYZE TABLE orders;
-- Recalcule les statistiques
```

#### Problème 2 : Boucles imbriquées excessives

```sql
-> Nested loop inner join
    (cost=5000 rows=100)
    (actual time=0.1..15000 rows=100 loops=1)
  -> Table scan on orders (rows=10000 loops=1)
  -> Index lookup on customers
      (actual time=0.05..1.5 rows=0.01 loops=10000)
```

⚠️ **Alerte** : `loops=10000` → opération répétée 10 000 fois !

**Temps total** : `1.5ms × 10000` = **15 secondes**

**Solution** : Créer index sur colonne de jointure pour réduire les loops.

#### Problème 3 : Filesort sur gros dataset

```sql
-> Sort: order_date DESC
    (cost=50000 rows=100000)
    (actual time=2500..2800 rows=100000 loops=1)
  -> Table scan on orders
      (actual time=0.5..450 rows=100000 loops=1)
```

⚠️ **Alerte** : `Sort` prend **2.8 secondes** sur 100k lignes.

**Solution** : Index sur `order_date DESC` pour éliminer le tri.

---

## Patterns d'optimisation guidés par EXPLAIN

### Pattern 1 : Full table scan → Index

**Avant optimisation** :

```sql
EXPLAIN SELECT * FROM products WHERE category_id = 5;

-- type: ALL
-- rows: 500000
-- Extra: Using where
-- Temps estimé : 500-2000ms
```

**Après optimisation** :

```sql
CREATE INDEX idx_products_category ON products(category_id);

EXPLAIN SELECT * FROM products WHERE category_id = 5;

-- type: ref
-- key: idx_products_category
-- rows: 2500
-- Temps estimé : 10-50ms
-- Amélioration : x50-100
```

### Pattern 2 : Filesort → Index covering

**Avant optimisation** :

```sql
EXPLAIN
SELECT customer_id, order_date, total_amount
FROM orders
WHERE status = 'pending'
ORDER BY order_date DESC
LIMIT 20;

-- type: ref
-- key: idx_status
-- rows: 50000
-- Extra: Using filesort
-- Temps : 300-800ms (tri de 50k lignes)
```

**Après optimisation** :

```sql
CREATE INDEX idx_orders_status_date_amount
ON orders(status, order_date DESC, total_amount);

EXPLAIN
SELECT customer_id, order_date, total_amount
FROM orders
WHERE status = 'pending'
ORDER BY order_date DESC
LIMIT 20;

-- type: ref
-- key: idx_orders_status_date_amount
-- rows: 20
-- Extra: Using index condition
-- Temps : 5-15ms
-- Amélioration : x30-50
```

### Pattern 3 : Using temporary → Index sur GROUP BY

**Avant optimisation** :

```sql
EXPLAIN
SELECT author_id, COUNT(*) as cnt
FROM articles
WHERE category_id = 5
GROUP BY author_id;

-- type: ref
-- rows: 50000
-- Extra: Using temporary; Using filesort
-- Temps : 400-1000ms
```

**Après optimisation** :

```sql
CREATE INDEX idx_articles_category_author
ON articles(category_id, author_id);

EXPLAIN
SELECT author_id, COUNT(*) as cnt
FROM articles
WHERE category_id = 5
GROUP BY author_id;

-- type: ref
-- key: idx_articles_category_author
-- rows: 50000
-- Extra: Using index
-- Temps : 50-150ms
-- Amélioration : x7-10
```

### Pattern 4 : Jointure inefficace → Index sur FK

**Avant optimisation** :

```sql
EXPLAIN
SELECT o.order_id, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending';

-- id | table | type | key        | rows   | Extra
-- 1  | o     | ref  | idx_status | 10000  | NULL
-- 1  | c     | ALL  | NULL       | 100000 | Using where; Using join buffer
--
-- ❌ Full scan sur customers pour chaque commande !
-- Temps : 5-15 secondes
```

**Après optimisation** :

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);

EXPLAIN
SELECT o.order_id, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending';

-- id | table | type   | key                | rows  | Extra
-- 1  | o     | ref    | idx_status         | 10000 | NULL
-- 1  | c     | eq_ref | PRIMARY            | 1     | NULL
--
-- ✅ Accès direct via PRIMARY KEY
-- Temps : 50-200ms
-- Amélioration : x50-100
```

---

## Cas d'usage avancés

### Comparer plusieurs stratégies d'index

```sql
-- Scénario : optimiser une requête avec plusieurs options d'index

-- Option 1 : Index simple sur date
CREATE INDEX idx_v1 ON orders(order_date);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE order_date >= '2024-01-01'
  AND status = 'completed'
ORDER BY order_date DESC
LIMIT 10;
-- Résultat : actual time=150..180ms (scan puis filtre status)

-- Option 2 : Index composite (status, order_date)
DROP INDEX idx_v1 ON orders;
CREATE INDEX idx_v2 ON orders(status, order_date DESC);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE order_date >= '2024-01-01'
  AND status = 'completed'
ORDER BY order_date DESC
LIMIT 10;
-- Résultat : actual time=5..8ms

-- Conclusion : Option 2 est x20-30 plus rapide
```

### Analyser les sous-requêtes

```sql
EXPLAIN
SELECT u.username
FROM users u
WHERE u.user_id IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE order_date >= '2024-01-01'
);

-- Résultat EXPLAIN :
-- id | select_type        | table  | type  | Extra
-- 1  | PRIMARY            | u      | ALL   | Using where
-- 2  | DEPENDENT SUBQUERY | orders | range | Using where; Using index
--
-- ⚠️ DEPENDENT SUBQUERY : sous-requête exécutée pour chaque ligne de 'users'
-- Très inefficace sur grandes tables
```

**Optimisation avec JOIN** :

```sql
EXPLAIN
SELECT DISTINCT u.username
FROM users u
INNER JOIN orders o ON u.user_id = o.customer_id
WHERE o.order_date >= '2024-01-01';

-- id | select_type | table | type  | Extra
-- 1  | SIMPLE      | o     | range | Using index
-- 1  | SIMPLE      | u     | eq_ref| NULL
--
-- ✅ SIMPLE : pas de sous-requête, plus efficace
```

### Partitionnement et partition pruning

```sql
-- Table partitionnée par année
CREATE TABLE logs (
    log_id BIGINT PRIMARY KEY,
    log_date DATE NOT NULL,
    message TEXT
) PARTITION BY RANGE (YEAR(log_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

EXPLAIN
SELECT * FROM logs
WHERE log_date = '2024-06-15';

-- Extra: Using where
-- partitions: p2024
--
-- ✅ Partition pruning : seule p2024 est scannée
```

---

## Outils complémentaires

### SHOW WARNINGS après EXPLAIN EXTENDED

```sql
EXPLAIN EXTENDED
SELECT * FROM users WHERE age > 25 AND country = 'FR';

SHOW WARNINGS;

-- Message: Requête réécrite par l'optimiseur
-- select `db`.`users`.`user_id` AS `user_id`, ...
-- from `db`.`users`
-- where (`db`.`users`.`country` = 'FR' and `db`.`users`.`age` > 25)
```

Montre la requête **après optimisation** par MariaDB (prédicates pushdown, simplifications, etc.).

### OPTIMIZER_TRACE pour analyse détaillée

```sql
-- Activer le traçage de l'optimiseur
SET optimizer_trace='enabled=on';

SELECT * FROM orders WHERE customer_id = 123;

-- Voir le processus de décision de l'optimiseur
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G

-- Désactiver
SET optimizer_trace='enabled=off';
```

Fournit un JSON détaillé du raisonnement de l'optimiseur : index considérés, coûts calculés, décision finale.

### SHOW PROFILE pour profiling détaillé

```sql
-- Activer le profiling
SET profiling = 1;

-- Exécuter requête
SELECT * FROM orders WHERE customer_id = 123;

-- Voir le profil
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;

-- Résultat : temps par étape
-- Stage                          | Duration
-- starting                       | 0.000050
-- checking permissions           | 0.000010
-- Opening tables                 | 0.000025
-- init                           | 0.000015
-- System lock                    | 0.000008
-- optimizing                     | 0.000012
-- statistics                     | 0.000028
-- preparing                      | 0.000015
-- executing                      | 0.000005
-- Sending data                   | 0.002450
-- end                            | 0.000008
-- query end                      | 0.000005
-- closing tables                 | 0.000008
-- freeing items                  | 0.000012
-- cleaning up                    | 0.000010
```

---

## Checklist d'analyse de performance

### ✅ Étapes d'analyse systématique

1. **Exécuter EXPLAIN** sur la requête lente
2. **Vérifier la colonne `type`** :
   - ❌ `ALL` sur table >10k lignes → créer index
   - ⚠️ `index` → vérifier si index covering possible
   - ✅ `range`, `ref`, `eq_ref`, `const` → bon
3. **Vérifier la colonne `key`** :
   - `NULL` → aucun index utilisé, problème !
4. **Analyser `Extra`** :
   - ❌ `Using filesort` → créer index ORDER BY
   - ❌ `Using temporary` → optimiser GROUP BY
   - ❌ `Using join buffer` → indexer colonne de jointure
   - ✅ `Using index` → optimal (index covering)
5. **Comparer `rows` estimé vs réel** :
   - Si très différent → `ANALYZE TABLE`
6. **Utiliser EXPLAIN ANALYZE** pour temps réels
7. **Tester les optimisations** : créer index, comparer avant/après

### ⚠️ Signaux d'alarme

- [ ] `type: ALL` sur table de plus de 10 000 lignes
- [ ] `key: NULL` dans une requête fréquente
- [ ] `rows: 500000+` pour retourner 10-100 lignes
- [ ] `Extra: Using filesort` sur ORDER BY fréquent
- [ ] `Extra: Using temporary; Using filesort`
- [ ] Temps d'exécution > 100ms pour requête simple
- [ ] `actual rows` 10× différent de `rows` estimé

---

## Exemples complets d'optimisation

### Exemple 1 : Dashboard e-commerce

**Requête initiale (lente)** :

```sql
EXPLAIN ANALYZE
SELECT
    p.product_id,
    p.name,
    p.price,
    COUNT(oi.item_id) as times_ordered,
    SUM(oi.quantity) as total_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE p.category_id = 5
  AND p.is_active = 1
GROUP BY p.product_id, p.name, p.price
ORDER BY times_ordered DESC
LIMIT 10;

-- Résultat EXPLAIN :
-- type: ALL (products), ref (order_items)
-- rows: 50000 (products)
-- Extra: Using temporary; Using filesort
-- actual time: 0.5..3500ms
-- Temps total : 3.5 secondes
```

**Optimisation** :

```sql
-- Index 1 : WHERE sur products
CREATE INDEX idx_products_category_active
ON products(category_id, is_active);

-- Index 2 : Jointure
CREATE INDEX idx_order_items_product
ON order_items(product_id, quantity);

EXPLAIN ANALYZE
-- Même requête

-- Résultat optimisé :
-- type: ref (products), ref (order_items)
-- key: idx_products_category_active, idx_order_items_product
-- rows: 500 (products)
-- Extra: Using temporary; Using filesort (reste mais sur dataset réduit)
-- actual time: 0.3..85ms
-- Temps total : 85ms
-- Amélioration : x40
```

### Exemple 2 : Recherche utilisateurs

**Requête initiale** :

```sql
EXPLAIN ANALYZE
SELECT user_id, username, email, last_login
FROM users
WHERE country = 'FR'
  AND age BETWEEN 25 AND 35
  AND status = 'active'
ORDER BY last_login DESC
LIMIT 20;

-- type: ALL
-- rows: 1000000
-- Extra: Using where; Using filesort
-- actual time: 0.5..8500ms
-- Temps : 8.5 secondes
```

**Optimisation progressive** :

```sql
-- Étape 1 : Index sur WHERE
CREATE INDEX idx_users_country_status_age
ON users(country, status, age);

EXPLAIN ANALYZE
-- type: range
-- rows: 5000
-- Extra: Using index condition; Using filesort
-- actual time: 0.2..450ms
-- Amélioration : x19

-- Étape 2 : Ajouter ORDER BY à l'index
DROP INDEX idx_users_country_status_age ON users;
CREATE INDEX idx_users_complete
ON users(country, status, age, last_login DESC);

EXPLAIN ANALYZE
-- type: range
-- rows: 20
-- Extra: Using index condition
-- actual time: 0.15..12ms
-- Amélioration totale : x700
```

---

## ✅ Points clés à retenir

- **EXPLAIN révèle la stratégie** de l'optimiseur, EXPLAIN ANALYZE donne les **temps réels**
- **Colonne `type`** : système > const > eq_ref > ref > range > index > ALL
- **`type: ALL`** sur table >10k lignes = problème urgent
- **Extra: Using filesort** = opportunité d'index ORDER BY
- **Extra: Using index** = optimal (index-only scan)
- **Estimations vs réalité** : si `rows` estimé ≠ `actual rows`, exécuter `ANALYZE TABLE`
- **Loops multiples** : attention aux nested loops avec `loops=10000+`
- **Tester systématiquement** : EXPLAIN ANALYZE avant/après optimisation
- **Index covering** élimine accès table → gain majeur de performance

---

## 🔗 Ressources et références

- [📖 MariaDB EXPLAIN](https://mariadb.com/kb/en/explain/)
- [📖 EXPLAIN ANALYZE](https://mariadb.com/kb/en/explain-analyze/)
- [📖 EXPLAIN FORMAT=JSON](https://mariadb.com/kb/en/explain-format-json/)
- [📖 OPTIMIZER_TRACE](https://mariadb.com/kb/en/optimizer-trace/)
- [📖 Understanding EXPLAIN Output](https://mariadb.com/kb/en/explain-output/)
- [🛠️ pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)

---

## ➡️ Section suivante

**5.8 Optimisation des requêtes** : Techniques avancées de réécriture de requêtes, optimisation des sous-requêtes, utilisation de CTE, et stratégies pour améliorer les performances au-delà de l'indexation.

⏭️ [Optimisation des requêtes](/05-index-et-performance/08-optimisation-requetes.md)

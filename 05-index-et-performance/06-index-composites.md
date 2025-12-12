🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6 Index composites et ordre des colonnes

> **Niveau** : Intermédiaire
> **Durée estimée** : 2.5 heures
> **Prérequis** : Section 5.1 à 5.5.3 (Fonctionnement B-Tree et stratégies d'indexation)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le fonctionnement interne des index composites (multi-colonnes)
- Maîtriser la règle du préfixe gauche et ses implications pratiques
- Déterminer l'ordre optimal des colonnes dans un index composite
- Analyser quand utiliser un index composite vs plusieurs index simples
- Optimiser les index pour couvrir plusieurs patterns de requêtes
- Éviter les erreurs courantes dans la conception d'index composites
- Mesurer l'impact de l'ordre des colonnes sur les performances

---

## Introduction

Les **index composites** (ou index multi-colonnes) sont l'un des outils les plus puissants pour optimiser les performances, mais aussi l'un des plus mal compris. Un index composite bien conçu peut remplacer plusieurs index simples et améliorer drastiquement les performances. À l'inverse, un ordre de colonnes inapproprié peut rendre un index composite totalement inutile.

💡 **Principe clé** : Un index composite est comme un **annuaire téléphonique** trié d'abord par nom de famille, puis par prénom. Vous pouvez chercher rapidement "Dupont" ou "Dupont, Jean", mais pas "Jean" seul sans parcourir tout l'annuaire.

⚠️ **Erreur fréquente** : Créer un index composite sans réfléchir à l'ordre des colonnes, rendant l'index inefficace pour certaines requêtes pourtant importantes.

---

## Comprendre les index composites

### Structure interne : Index B-Tree multi-colonnes

Un index composite sur `(col_a, col_b, col_c)` crée un arbre B-Tree trié **d'abord** par `col_a`, **puis** par `col_b` pour chaque valeur de `col_a`, **puis** par `col_c` pour chaque combinaison `(col_a, col_b)`.

```sql
-- Exemple : index composite sur table utilisateurs
CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    country VARCHAR(2),
    city VARCHAR(100),
    age INT,
    email VARCHAR(255) UNIQUE
) ENGINE=InnoDB;

-- Index composite
CREATE INDEX idx_users_location_age
ON users(country, city, age);
```

**Visualisation de la structure interne** :

```
Index idx_users_location_age (country, city, age)
│
├─ 'FR'
│  ├─ 'Paris'
│  │  ├─ age: 25 → [user_id: 1001]
│  │  ├─ age: 28 → [user_id: 1005]
│  │  └─ age: 35 → [user_id: 1012]
│  └─ 'Lyon'
│     ├─ age: 30 → [user_id: 1020]
│     └─ age: 42 → [user_id: 1025]
│
├─ 'US'
│  ├─ 'New York'
│  │  ├─ age: 27 → [user_id: 2001]
│  │  └─ age: 33 → [user_id: 2008]
│  └─ 'San Francisco'
│     └─ age: 29 → [user_id: 2015]
│
└─ ...
```

**Propriété importante** : Les données sont triées de manière **hiérarchique**, créant une structure ordonnée à plusieurs niveaux.

---

## La règle du préfixe gauche

### Principe fondamental

Un index composite `(col_a, col_b, col_c)` peut être utilisé pour des requêtes filtrant sur :
- ✅ `col_a` seule
- ✅ `col_a, col_b`
- ✅ `col_a, col_b, col_c`

Mais **PAS** pour :
- ❌ `col_b` seule
- ❌ `col_c` seule
- ❌ `col_b, col_c`

**Analogie** : Dans un annuaire (Nom, Prénom) :
- ✅ Vous pouvez chercher "Dupont" (nom seul)
- ✅ Vous pouvez chercher "Dupont, Jean" (nom + prénom)
- ❌ Vous ne pouvez pas chercher "Jean" efficacement sans nom

### Exemples pratiques

```sql
-- Index : (country, city, age)
CREATE INDEX idx_location_age ON users(country, city, age);

-- ✅ Requête 1 : utilise l'index (préfixe: country)
SELECT * FROM users WHERE country = 'FR';
EXPLAIN: type=ref, key=idx_location_age, rows=50000

-- ✅ Requête 2 : utilise l'index (préfixe: country, city)
SELECT * FROM users WHERE country = 'FR' AND city = 'Paris';
EXPLAIN: type=ref, key=idx_location_age, rows=5000

-- ✅ Requête 3 : utilise l'index complet
SELECT * FROM users WHERE country = 'FR' AND city = 'Paris' AND age > 25;
EXPLAIN: type=range, key=idx_location_age, rows=500

-- ❌ Requête 4 : N'utilise PAS l'index (commence par city)
SELECT * FROM users WHERE city = 'Paris';
EXPLAIN: type=ALL, key=NULL, rows=1000000

-- ❌ Requête 5 : N'utilise PAS l'index (commence par age)
SELECT * FROM users WHERE age > 25;
EXPLAIN: type=ALL, key=NULL, rows=1000000

-- ⚠️ Requête 6 : utilise partiellement l'index
SELECT * FROM users WHERE country = 'FR' AND age > 25;
EXPLAIN: type=ref, key=idx_location_age, rows=50000
-- Utilise seulement 'country', ignore 'age' car 'city' manque
```

💡 **Important** : La requête 6 utilise l'index pour filtrer par `country`, mais doit ensuite scanner toutes les lignes françaises pour filtrer par `age`, car `city` (la colonne intermédiaire) est absente.

### Cas particulier : plages de valeurs

Quand une colonne utilise une **plage** (`>`, `<`, `BETWEEN`, `LIKE 'prefix%'`), les colonnes suivantes dans l'index ne peuvent plus être utilisées efficacement.

```sql
-- Index : (country, city, age)
CREATE INDEX idx_location_age ON users(country, city, age);

-- ✅ Égalité sur country, puis plage sur city
SELECT * FROM users
WHERE country = 'FR'
  AND city > 'L'
  AND age = 30;
-- Utilise (country, city) de l'index
-- age ne peut pas être utilisé car city est une plage

-- ✅ Égalités sur country et city, puis plage sur age
SELECT * FROM users
WHERE country = 'FR'
  AND city = 'Paris'
  AND age BETWEEN 25 AND 35;
-- Utilise l'index complet (country, city, age)
```

**Règle d'ordre optimal** :
1. **Colonnes avec égalité** (`=`, `IN`) en premier
2. **Colonnes avec plage** (`>`, `<`, `BETWEEN`, `LIKE`) ensuite
3. **Colonnes de tri** (`ORDER BY`) en dernier

---

## Déterminer l'ordre optimal des colonnes

### Critère 1 : Type de condition (égalité vs plage)

```sql
-- Requête analysée
SELECT * FROM orders
WHERE status = 'pending'           -- Égalité
  AND customer_id IN (100, 200)    -- Égalité (IN)
  AND order_date > '2024-01-01'    -- Plage
ORDER BY order_date;

-- ✅ Ordre optimal : égalités avant plages
CREATE INDEX idx_orders_optimal
ON orders(
    status,        -- Égalité
    customer_id,   -- Égalité (IN)
    order_date     -- Plage + ORDER BY
);
```

### Critère 2 : Cardinalité et sélectivité

**Cardinalité** : Nombre de valeurs distinctes dans une colonne.
**Sélectivité** : Ratio de valeurs distinctes / total de lignes.

**Règle générale** : Placer les colonnes à **haute cardinalité** (forte sélectivité) **en premier** parmi les égalités.

```sql
-- Analyse de cardinalité
SELECT
    COUNT(DISTINCT status) as status_cardinality,
    COUNT(DISTINCT country) as country_cardinality,
    COUNT(DISTINCT email) as email_cardinality,
    COUNT(*) as total_rows
FROM users;

-- Résultats exemple :
-- status_cardinality: 3 (active, inactive, suspended)
-- country_cardinality: 195
-- email_cardinality: 1000000 (unique)
-- total_rows: 1000000
```

**Sélectivité** :
- `email` : 1000000/1000000 = 1.0 (excellente)
- `country` : 195/1000000 = 0.000195 (faible)
- `status` : 3/1000000 = 0.000003 (très faible)

**Index optimal selon sélectivité** :

```sql
-- Requête fréquente
SELECT * FROM users
WHERE status = 'active'
  AND country = 'FR';

-- ❌ Ordre non optimal : faible sélectivité en premier
CREATE INDEX idx_bad ON users(status, country);
-- status = 'active' filtre 900k lignes sur 1M (90%)
-- Puis country affine sur ces 900k

-- ✅ Ordre optimal : haute sélectivité en premier
CREATE INDEX idx_good ON users(country, status);
-- country = 'FR' filtre à ~5000 lignes (0.5%)
-- Puis status affine sur ces 5000
-- Beaucoup plus efficace !
```

💡 **Exception** : Si une colonne à faible cardinalité apparaît dans **100% des requêtes** et filtre significativement, elle peut être placée en premier malgré sa faible sélectivité.

### Critère 3 : Fréquence d'utilisation des colonnes

```sql
-- Analyse des patterns de requêtes

-- 80% des requêtes : filtrent par category_id
-- 60% des requêtes : filtrent par status
-- 20% des requêtes : filtrent par brand_id

-- ✅ Ordre basé sur fréquence (si cardinalités similaires)
CREATE INDEX idx_products_usage
ON products(
    category_id,  -- 80% des requêtes
    status,       -- 60% des requêtes
    brand_id      -- 20% des requêtes
);
```

Cet index peut servir :
- 80% des requêtes (filtrant par category_id)
- 48% des requêtes (filtrant par category_id ET status)
- 9.6% des requêtes (filtrant par les 3)

### Critère 4 : Couverture de plusieurs patterns

L'ordre doit maximiser le nombre de patterns de requêtes couverts.

```sql
-- Patterns de requêtes identifiés :
-- Pattern A (50% trafic) : WHERE country = X
-- Pattern B (30% trafic) : WHERE country = X AND city = Y
-- Pattern C (20% trafic) : WHERE country = X AND city = Y AND age > Z

-- ✅ Un seul index composite couvre tous les patterns
CREATE INDEX idx_users_location_age ON users(country, city, age);
-- Couvre 100% du trafic avec un seul index

-- ❌ Alternative moins efficace : 3 index séparés
CREATE INDEX idx_country ON users(country);
CREATE INDEX idx_country_city ON users(country, city);
CREATE INDEX idx_country_city_age ON users(country, city, age);
-- Coût de maintenance × 3, redondance
```

---

## Stratégies d'optimisation

### Index composites vs index simples multiples

```sql
-- Scénario : requêtes variées sur table products

-- Option 1 : Index simples séparés
CREATE INDEX idx_category ON products(category_id);
CREATE INDEX idx_brand ON products(brand_id);
CREATE INDEX idx_price ON products(price);
```

**Avantages** :
- ✅ Chaque requête simple est optimisée
- ✅ Flexibilité maximale

**Inconvénients** :
- ❌ Requêtes multi-colonnes utilisent index merge (lent)
- ❌ Coût de maintenance × 3

```sql
-- Option 2 : Index composite unique
CREATE INDEX idx_products_composite
ON products(category_id, brand_id, price);
```

**Avantages** :
- ✅ Requêtes multi-colonnes très rapides
- ✅ Un seul index à maintenir

**Inconvénients** :
- ❌ Requêtes filtrant seulement par `brand_id` ou `price` n'utilisent pas l'index

**Décision** :
```sql
-- Si requêtes majoritairement multi-colonnes → Option 2
-- Si requêtes majoritairement sur colonnes isolées → Option 1
-- Si mix équilibré → Compromis avec 1-2 index composites ciblés
```

### Index merge : quand MariaDB combine plusieurs index

MariaDB peut **combiner plusieurs index simples** avec l'algorithme **index merge**, mais c'est généralement moins efficace qu'un index composite.

```sql
-- Table avec 2 index simples
CREATE INDEX idx_category ON products(category_id);
CREATE INDEX idx_brand ON products(brand_id);

-- Requête utilisant les deux colonnes
SELECT * FROM products
WHERE category_id = 5
  AND brand_id = 10;

EXPLAIN
-- type: index_merge
-- key: idx_category, idx_brand
-- Extra: Using intersect(idx_category, idx_brand); Using where
```

**Processus index merge** :
1. Lit l'index `idx_category` → liste d'IDs (ex: 10000 lignes)
2. Lit l'index `idx_brand` → liste d'IDs (ex: 5000 lignes)
3. Intersection des deux listes → IDs communs (ex: 50 lignes)
4. Accède aux 50 lignes dans la table

**Coût** : 3 opérations d'index + accès table.

**Avec index composite** :
```sql
CREATE INDEX idx_category_brand ON products(category_id, brand_id);

EXPLAIN
-- type: ref
-- key: idx_category_brand
-- Extra: Using index condition
```

**Processus optimisé** :
1. Lit directement les 50 lignes via l'index composite
2. Accède aux lignes dans la table

**Coût** : 1 opération d'index + accès table (**2-3× plus rapide**).

💡 **Recommandation** : Préférer un index composite pour les combinaisons de colonnes fréquentes plutôt que se reposer sur l'index merge.

### Ordre et ORDER BY : double utilisation

Un index composite peut servir **à la fois** pour filtrage (WHERE) et tri (ORDER BY).

```sql
-- Requête avec WHERE + ORDER BY
SELECT * FROM articles
WHERE category_id = 5
ORDER BY published_at DESC
LIMIT 10;

-- ✅ Index optimal : WHERE + ORDER BY
CREATE INDEX idx_articles_category_published
ON articles(category_id, published_at DESC);

EXPLAIN
-- type: ref
-- key: idx_articles_category_published
-- rows: 10
-- Extra: Using index condition (PAS de "Using filesort")
```

**Bénéfices** :
1. Filtre via `category_id` (égalité)
2. Lit les données déjà triées par `published_at DESC`
3. S'arrête après 10 lignes (LIMIT)

**Performance** : Lecture de seulement 10 lignes au lieu de 50 000.

### Longueur des index composites

**Recommandation** : Limiter à **3-4 colonnes** maximum dans un index composite.

```sql
-- ✅ Acceptable : 3 colonnes
CREATE INDEX idx_orders ON orders(customer_id, status, order_date);

-- ⚠️ Limite : 4 colonnes
CREATE INDEX idx_products ON products(category_id, brand_id, status, price);

-- ❌ Trop long : 6+ colonnes
CREATE INDEX idx_too_long ON table_name(col1, col2, col3, col4, col5, col6);
-- Index volumineux, coût de maintenance élevé, rarement utilisé complètement
```

**Compromis** :
- Index courts (2-3 colonnes) : plus flexibles, maintenance légère
- Index longs (4-5 colonnes) : très spécialisés, coût de maintenance élevé

---

## Cas d'usage avancés

### Index covering avec colonnes supplémentaires

Ajouter des colonnes SELECT à la fin d'un index composite pour créer un **index covering** (index-only scan).

```sql
-- Requête fréquente
SELECT customer_id, order_date, total_amount
FROM orders
WHERE status = 'completed'
  AND order_date >= '2024-01-01'
ORDER BY order_date DESC;

-- ✅ Index covering : WHERE + ORDER BY + SELECT
CREATE INDEX idx_orders_covering
ON orders(
    status,        -- WHERE égalité
    order_date,    -- WHERE plage + ORDER BY
    total_amount   -- Colonne SELECT (covering)
);

EXPLAIN
-- Extra: Using index (index-only scan, optimal !)
```

**Avantage** : MariaDB lit **uniquement l'index**, sans accéder à la table des données. Économie d'I/O considérable.

⚠️ **Compromis** : Index plus volumineux, coût d'écriture accru.

### Index avec préfixes de chaînes

Pour colonnes VARCHAR longues, utiliser un **préfixe** dans l'index composite.

```sql
-- Colonne URL très longue (2000 caractères)
CREATE TABLE pages (
    page_id INT PRIMARY KEY,
    url VARCHAR(2000),
    domain VARCHAR(100),
    status VARCHAR(20)
);

-- ❌ Index complet : trop volumineux
CREATE INDEX idx_bad ON pages(domain, url);
-- Chaque clé d'index peut faire 2100 octets !

-- ✅ Index avec préfixe sur URL
CREATE INDEX idx_good ON pages(domain, url(100));
-- Préfixe de 100 caractères suffit souvent

-- Vérifier la sélectivité du préfixe
SELECT
    COUNT(DISTINCT url) as full_url,
    COUNT(DISTINCT LEFT(url, 50)) as prefix_50,
    COUNT(DISTINCT LEFT(url, 100)) as prefix_100
FROM pages;
```

💡 **Règle** : Choisir une longueur de préfixe qui capte **95%+ de la sélectivité** de la colonne complète.

### Ordre inversé pour DESC

Pour ORDER BY descendant fréquent, spécifier explicitement la direction dans l'index.

```sql
-- Requête : derniers articles
SELECT * FROM articles
ORDER BY published_at DESC
LIMIT 20;

-- ✅ Index avec direction explicite DESC
CREATE INDEX idx_articles_published_desc
ON articles(published_at DESC);

-- MariaDB lit l'index directement de la fin
-- Plus efficace que scan backward sur index ASC
```

**Depuis MariaDB 10.8** : Support complet des directions mixtes ASC/DESC dans index composites.

```sql
-- Tri mixte
SELECT * FROM leaderboard
ORDER BY score DESC, username ASC;

-- ✅ Index avec directions correspondantes
CREATE INDEX idx_leaderboard_sort
ON leaderboard(score DESC, username ASC);
```

### Index partiels simulés (colonnes générées)

MariaDB ne supporte pas les **index partiels** (WHERE clause dans l'index) comme PostgreSQL, mais on peut utiliser des **colonnes générées**.

```sql
-- Cas d'usage : indexer seulement les lignes actives

-- ❌ Index complet (indexe aussi lignes deleted)
CREATE INDEX idx_products_all ON products(category_id);
-- 90% des requêtes filtrent aussi par deleted_at IS NULL

-- ✅ Solution : colonne générée
ALTER TABLE products
ADD COLUMN is_active TINYINT(1)
    AS (deleted_at IS NULL) VIRTUAL;

CREATE INDEX idx_products_active
ON products(is_active, category_id);

-- Requête optimisée
SELECT * FROM products
WHERE is_active = 1
  AND category_id = 5;
-- Index utilisé efficacement pour lignes actives uniquement
```

---

## Analyse et validation

### Tester différents ordres de colonnes

```sql
-- Scénario : déterminer le meilleur ordre pour (status, customer_id, order_date)

-- Option 1
CREATE INDEX idx_v1 ON orders(status, customer_id, order_date);

-- Option 2
CREATE INDEX idx_v2 ON orders(customer_id, status, order_date);

-- Tester avec requêtes réelles
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'pending'
  AND customer_id = 12345
  AND order_date >= '2024-01-01';

-- Comparer actual time des deux options
-- Choisir l'option avec le temps le plus court
```

### Analyser l'utilisation avec EXPLAIN

```sql
EXPLAIN
SELECT * FROM users
WHERE country = 'FR'
  AND age > 25;

-- Index : (country, city, age)
-- Résultat EXPLAIN :
-- type: ref (utilise country)
-- key: idx_location_age
-- rows: 50000
-- Extra: Using index condition
--
-- ⚠️ 'city' manque → 'age' n'est pas utilisé efficacement
```

**Indicateur clé** : Colonne `rows` estimée.
- Si `rows` est proche du résultat réel → index efficace
- Si `rows` >> résultat réel → opportunité d'optimisation

### Détecter les index composites sous-utilisés

```sql
-- Requête : index utilisant seulement la première colonne
SELECT
    TABLE_NAME,
    INDEX_NAME,
    COLUMN_NAME,
    SEQ_IN_INDEX
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
  AND INDEX_NAME LIKE '%composite%'
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;

-- Analyser manuellement si les colonnes 2+ sont réellement utilisées
```

**Signes d'index composite mal conçu** :
- Index jamais utilisé au-delà de la 1ère colonne
- Performances identiques avec index simple sur 1ère colonne
- Coût de maintenance sans bénéfice

**Action** : Supprimer colonnes inutiles ou réorganiser l'ordre.

---

## Anti-patterns à éviter

### Anti-pattern 1 : Ordre alphabétique

```sql
-- ❌ MAUVAIS : ordre alphabétique des colonnes
CREATE INDEX idx_bad ON orders(customer_id, order_date, status);
-- Aucune logique de requête

-- ✅ BON : ordre basé sur les requêtes réelles
CREATE INDEX idx_good ON orders(status, customer_id, order_date);
-- status (égalité) → customer_id (égalité) → order_date (plage)
```

### Anti-pattern 2 : Dupliquer les préfixes

```sql
-- ❌ MAUVAIS : index redondants
CREATE INDEX idx_1 ON users(country);
CREATE INDEX idx_2 ON users(country, city);
CREATE INDEX idx_3 ON users(country, city, age);
-- idx_1 est redondant (couvert par idx_2 et idx_3)

-- ✅ BON : garder seulement les index nécessaires
CREATE INDEX idx_location ON users(country, city);
CREATE INDEX idx_location_age ON users(country, city, age);
-- Ou simplement un seul si idx_location_age couvre tous les cas
```

### Anti-pattern 3 : Ignorer la cardinalité

```sql
-- ❌ MAUVAIS : faible cardinalité en premier
CREATE INDEX idx_bad ON products(is_active, category_id);
-- is_active : 2 valeurs (0, 1)
-- category_id : 500 valeurs

-- ✅ BON : haute cardinalité en premier
CREATE INDEX idx_good ON products(category_id, is_active);
-- Filtre d'abord par category (plus sélectif)
```

### Anti-pattern 4 : Index trop long sans justification

```sql
-- ❌ MAUVAIS : 7 colonnes sans analyse
CREATE INDEX idx_monster ON orders(
    customer_id,
    status,
    order_date,
    total_amount,
    shipping_method,
    payment_method,
    coupon_code
);
-- Index volumineux, coût élevé, probablement jamais utilisé au-delà de 3-4 colonnes

-- ✅ BON : limiter à 3-4 colonnes vraiment nécessaires
CREATE INDEX idx_reasonable ON orders(customer_id, status, order_date);
```

---

## Checklist de conception d'index composites

### ✅ Étapes de conception

1. **Analyser les requêtes** : Identifier les patterns WHERE + ORDER BY
2. **Lister les colonnes** : Extraire toutes les colonnes impliquées
3. **Calculer la cardinalité** : Mesurer la sélectivité de chaque colonne
4. **Déterminer l'ordre** :
   - Égalités avant plages
   - Haute cardinalité avant basse cardinalité (parmi égalités)
   - Colonnes fréquentes avant colonnes rares
5. **Ajouter colonnes covering** : Si bénéfice > coût
6. **Tester avec EXPLAIN** : Vérifier utilisation effective
7. **Comparer alternatives** : Tester différents ordres
8. **Monitorer en production** : Mesurer l'impact réel

### ⚠️ Points de validation

- [ ] Ordre basé sur **logique de requêtes**, pas alphabétique
- [ ] Colonnes d'égalité **avant** colonnes de plage
- [ ] Haute cardinalité **avant** basse cardinalité
- [ ] Maximum **3-4 colonnes** (sauf cas justifié)
- [ ] Pas de préfixes redondants (vérifier index existants)
- [ ] Validé avec **EXPLAIN** sur requêtes réelles
- [ ] Impact mesuré avec **EXPLAIN ANALYZE**

---

## Exemples complets par cas d'usage

### E-commerce : recherche produits

```sql
-- Requêtes analysées :
-- 1. Par catégorie + prix (60%)
-- 2. Par catégorie + marque (30%)
-- 3. Par catégorie seule (10%)

-- Solution : un index composite couvre les 3 patterns
CREATE INDEX idx_products_category_price_brand
ON products(
    category_id,  -- 100% des requêtes
    price,        -- 60% + ORDER BY fréquent
    brand_id      -- 30%
);

-- Requête 1 : utilise (category_id, price)
SELECT * FROM products
WHERE category_id = 5 AND price BETWEEN 100 AND 500;

-- Requête 2 : utilise (category_id) puis scan sur brand_id
SELECT * FROM products
WHERE category_id = 5 AND brand_id = 10;
-- Moins optimal mais acceptable

-- Alternative si requête 2 est critique :
CREATE INDEX idx_products_category_brand
ON products(category_id, brand_id);
-- 2 index au lieu d'un, mais optimise les 2 patterns principaux
```

### Réseaux sociaux : fil d'actualité

```sql
-- Requête principale : posts d'amis récents
SELECT p.post_id, p.content, p.created_at, u.username
FROM posts p
INNER JOIN friendships f ON p.user_id = f.friend_id
INNER JOIN users u ON p.user_id = u.user_id
WHERE f.user_id = 12345
  AND p.is_deleted = 0
ORDER BY p.created_at DESC
LIMIT 20;

-- Index optimal sur posts
CREATE INDEX idx_posts_user_active_created
ON posts(
    user_id,      -- Jointure + WHERE fréquent
    is_deleted,   -- Filtrage actifs
    created_at DESC  -- ORDER BY
);

-- Index covering sur friendships
CREATE INDEX idx_friendships_user_friend
ON friendships(user_id, friend_id);
```

### Analytics : rapports agrégés

```sql
-- Requête : ventes par catégorie et région sur période
SELECT
    category_id,
    region,
    DATE_FORMAT(order_date, '%Y-%m') as month,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue
FROM orders
WHERE order_date >= '2024-01-01'
  AND status = 'completed'
GROUP BY category_id, region, month
ORDER BY month DESC, revenue DESC;

-- Index optimal covering
CREATE INDEX idx_orders_analytics
ON orders(
    status,           -- WHERE égalité
    order_date,       -- WHERE plage + GROUP BY (via DATE_FORMAT)
    category_id,      -- GROUP BY
    region,           -- GROUP BY
    total_amount      -- SUM (covering)
);
```

---

## ✅ Points clés à retenir

- **Règle du préfixe gauche** : index (A,B,C) utilisable pour A, AB, ABC mais pas B, C, BC
- **Ordre optimal** : égalités → plages → tri → colonnes covering
- **Cardinalité compte** : haute sélectivité avant basse (parmi égalités)
- **Un bon index composite** peut remplacer plusieurs index simples
- **Index merge** moins efficace qu'index composite pour combinaisons fréquentes
- **Limiter à 3-4 colonnes** : au-delà, efficacité décroissante
- **Directions ASC/DESC** : doivent correspondre pour ORDER BY multi-colonnes
- **Tester avec EXPLAIN** : valider utilisation effective de l'index
- **Monitoring continu** : mesurer l'impact réel en production

---

## 🔗 Ressources et références

- [📖 MariaDB Composite Indexes](https://mariadb.com/kb/en/composite-indexes/)
- [📖 Index Prefixes](https://mariadb.com/kb/en/create-index/#index-prefixes)
- [📖 EXPLAIN Documentation](https://mariadb.com/kb/en/explain/)
- [📖 Index Merge Optimization](https://mariadb.com/kb/en/index-merge-optimization/)
- [📖 Generated Columns](https://mariadb.com/kb/en/generated-columns/)
- [🛠️ pt-duplicate-key-checker](https://www.percona.com/doc/percona-toolkit/LATEST/pt-duplicate-key-checker.html)

---

## ➡️ Section suivante

**5.7 Analyse des plans d'exécution (EXPLAIN, EXPLAIN ANALYZE)** : Maîtriser l'interprétation détaillée des plans d'exécution, comprendre les différents types d'accès (ALL, index, range, ref, eq_ref, const) et utiliser EXPLAIN ANALYZE pour mesurer les performances réelles.

⏭️ [Analyse des plans d'exécution (EXPLAIN, EXPLAIN ANALYZE)](/05-index-et-performance/07-analyse-plans-execution.md)

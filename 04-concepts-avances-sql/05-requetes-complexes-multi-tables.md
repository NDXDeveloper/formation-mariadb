🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Requêtes complexes multi-tables

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Maîtrise des jointures, CTE, Window Functions, agrégations

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Construire des **requêtes analytiques sophistiquées** impliquant 5+ tables
- Maîtriser les **stratégies de jointure** complexes (multiple JOINs, self-joins, cross joins)
- Combiner **CTE, Window Functions et agrégations** dans une même requête
- Optimiser les **performances** des requêtes multi-tables
- Structurer et **documenter** des requêtes complexes pour la maintenabilité
- Résoudre des **problèmes métier réels** nécessitant plusieurs sources de données
- Appliquer les **best practices** pour éviter les pièges de performance

---

## Introduction

### Qu'est-ce qu'une requête complexe multi-tables ?

Une **requête complexe multi-tables** combine :
- **Plusieurs tables** (généralement 3 à 10+)
- **Plusieurs types d'opérations** (jointures, agrégations, window functions)
- **Logique métier sophistiquée** (calculs, conditions multiples, transformations)
- **Volume de données significatif** (performance critique)

**Exemple typique** : Générer un rapport de ventes consolidé incluant clients, produits, commandes, stocks, promotions, et historique.

### Pourquoi c'est complexe ?

🔴 **Défis** :
- **Explosion cartésienne** : Jointures mal conçues → millions de lignes
- **Performance** : Chaque jointure multiplie le coût
- **Maintenabilité** : Requêtes de 200+ lignes difficiles à comprendre
- **Bugs subtils** : Doublons, valeurs NULL, agrégations incorrectes

✅ **Solutions** :
- **Stratégie de décomposition** : CTE pour diviser le problème
- **Ordre des jointures** : Optimiser le plan d'exécution
- **Filtrage précoce** : Réduire les données avant les jointures
- **Documentation** : Commenter chaque étape

---

## Schéma de base de données exemple

Nous allons utiliser un schéma e-commerce complet pour nos exemples.

```sql
-- Clients
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(100),
    country VARCHAR(50),
    signup_date DATE,
    customer_segment VARCHAR(20) -- VIP, Regular, New
);

-- Produits
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    subcategory VARCHAR(50),
    unit_price DECIMAL(10,2),
    cost DECIMAL(10,2)
);

-- Commandes
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    status VARCHAR(20), -- Completed, Pending, Cancelled
    shipping_country VARCHAR(50),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Détails commandes
CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2), -- Prix au moment de la vente
    discount_pct DECIMAL(5,2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Promotions
CREATE TABLE promotions (
    promotion_id INT PRIMARY KEY,
    promotion_name VARCHAR(100),
    product_id INT,
    start_date DATE,
    end_date DATE,
    discount_pct DECIMAL(5,2),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Reviews produits
CREATE TABLE product_reviews (
    review_id INT PRIMARY KEY,
    product_id INT,
    customer_id INT,
    rating INT, -- 1-5
    review_date DATE,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

**Schéma visuel** :
```
customers (1) ----< orders (1) ----< order_items >---- (1) products
    |                                                        |
    |                                                        |
    +----------------> product_reviews <--------------------+
                                                             |
                                                   promotions
```

---

## Exemple 1 : Rapport de ventes complet

Générer un rapport consolidé avec ventes, profits, ratings par produit et catégorie.

### Requête avec CTE multiples

```sql
WITH
    -- CTE 1 : Calcul des ventes par item
    item_sales AS (
        SELECT
            oi.item_id,
            oi.order_id,
            oi.product_id,
            o.order_date,
            o.customer_id,
            oi.quantity,
            oi.unit_price,
            oi.discount_pct,
            -- Revenue après discount
            oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100) AS item_revenue,
            -- Coût (récupéré depuis products)
            p.cost,
            oi.quantity * p.cost AS item_cost
        FROM order_items oi
        JOIN orders o ON oi.order_id = o.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE o.status = 'Completed'  -- ✅ Filtrage précoce
          AND o.order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    ),

    -- CTE 2 : Agrégation par produit
    product_sales_summary AS (
        SELECT
            product_id,
            COUNT(DISTINCT order_id) AS order_count,
            SUM(quantity) AS total_quantity_sold,
            SUM(item_revenue) AS total_revenue,
            SUM(item_cost) AS total_cost,
            SUM(item_revenue) - SUM(item_cost) AS total_profit,
            AVG(item_revenue / NULLIF(quantity, 0)) AS avg_selling_price
        FROM item_sales
        GROUP BY product_id
    ),

    -- CTE 3 : Ratings moyens par produit
    product_ratings AS (
        SELECT
            product_id,
            COUNT(*) AS review_count,
            AVG(rating) AS avg_rating
        FROM product_reviews
        WHERE review_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
        GROUP BY product_id
    ),

    -- CTE 4 : Informations produit enrichies
    enriched_products AS (
        SELECT
            p.product_id,
            p.product_name,
            p.category,
            p.subcategory,
            p.unit_price AS current_price,
            pss.order_count,
            pss.total_quantity_sold,
            pss.total_revenue,
            pss.total_cost,
            pss.total_profit,
            pss.avg_selling_price,
            pr.review_count,
            pr.avg_rating
        FROM products p
        LEFT JOIN product_sales_summary pss ON p.product_id = pss.product_id
        LEFT JOIN product_ratings pr ON p.product_id = pr.product_id
    )

-- Requête finale : Rapport consolidé avec métriques calculées
SELECT
    category,
    subcategory,
    product_name,
    current_price,
    COALESCE(order_count, 0) AS orders,
    COALESCE(total_quantity_sold, 0) AS units_sold,
    COALESCE(ROUND(total_revenue, 2), 0) AS revenue,
    COALESCE(ROUND(total_profit, 2), 0) AS profit,
    -- Marge bénéficiaire
    CASE
        WHEN total_revenue > 0 THEN ROUND(100.0 * total_profit / total_revenue, 1)
        ELSE 0
    END AS profit_margin_pct,
    -- Metrics qualité
    COALESCE(review_count, 0) AS reviews,
    COALESCE(ROUND(avg_rating, 2), 0) AS avg_rating,
    -- Performance vs prix courant
    CASE
        WHEN avg_selling_price IS NOT NULL
        THEN ROUND(100.0 * (avg_selling_price - current_price) / current_price, 1)
        ELSE NULL
    END AS avg_discount_pct
FROM enriched_products
WHERE total_revenue IS NOT NULL  -- Seulement produits avec ventes
ORDER BY total_revenue DESC
LIMIT 20;
```

**Résultat** :
```
+-------------+-------------+------------------+---------------+--------+------------+-----------+----------+-------------------+---------+------------+------------------+
| category    | subcategory | product_name     | current_price | orders | units_sold | revenue   | profit   | profit_margin_pct | reviews | avg_rating | avg_discount_pct |
+-------------+-------------+------------------+---------------+--------+------------+-----------+----------+-------------------+---------+------------+------------------+
| Electronics | Laptops     | Laptop Pro 15"   |       1500.00 |     45 |         52 | 72000.00  | 26000.00 |              36.1 |      23 |       4.65 |             -8.5 |
| Electronics | Laptops     | Laptop Air 13"   |       1200.00 |     38 |         42 | 48000.00  | 16800.00 |              35.0 |      19 |       4.42 |             -5.2 |
| Electronics | Monitors    | Monitor 4K 27"   |        600.00 |     32 |         35 | 20000.00  |  7000.00 |              35.0 |      15 |       4.53 |             -4.7 |
| Accessories | Keyboards   | Keyboard Mech    |        150.00 |     28 |         31 |  4500.00  |  1860.00 |              41.3 |      12 |       4.75 |             -3.3 |
+-------------+-------------+------------------+---------------+--------+------------+-----------+----------+-------------------+---------+------------+------------------+
```

💡 **Points clés** :
- **4 CTE** pour décomposer le problème en étapes logiques
- **LEFT JOIN** pour inclure produits sans ventes/reviews
- **COALESCE** pour gérer les NULL proprement
- **Filtrage précoce** sur les dates et status

---

## Exemple 2 : Analyse de cohortes clients

Analyser la rétention et la valeur vie client (LTV) par cohorte mensuelle.

```sql
WITH
    -- CTE 1 : Première commande par client (définit la cohorte)
    customer_first_order AS (
        SELECT
            customer_id,
            MIN(order_date) AS first_order_date,
            DATE_FORMAT(MIN(order_date), '%Y-%m') AS cohort_month
        FROM orders
        WHERE status = 'Completed'
        GROUP BY customer_id
    ),

    -- CTE 2 : Toutes les commandes avec info cohorte
    orders_with_cohort AS (
        SELECT
            o.order_id,
            o.customer_id,
            o.order_date,
            cfo.cohort_month,
            -- Mois depuis la première commande (M0, M1, M2...)
            PERIOD_DIFF(
                DATE_FORMAT(o.order_date, '%Y%m'),
                DATE_FORMAT(cfo.first_order_date, '%Y%m')
            ) AS months_since_first_order
        FROM orders o
        JOIN customer_first_order cfo ON o.customer_id = cfo.customer_id
        WHERE o.status = 'Completed'
    ),

    -- CTE 3 : Revenue par commande
    order_revenue AS (
        SELECT
            oi.order_id,
            SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS order_total
        FROM order_items oi
        GROUP BY oi.order_id
    ),

    -- CTE 4 : Métriques par client et cohorte
    customer_metrics AS (
        SELECT
            owc.customer_id,
            owc.cohort_month,
            owc.months_since_first_order,
            COUNT(DISTINCT owc.order_id) AS order_count,
            SUM(orv.order_total) AS total_spent
        FROM orders_with_cohort owc
        JOIN order_revenue orv ON owc.order_id = orv.order_id
        GROUP BY owc.customer_id, owc.cohort_month, owc.months_since_first_order
    ),

    -- CTE 5 : Agrégation par cohorte et période
    cohort_analysis AS (
        SELECT
            cohort_month,
            months_since_first_order,
            COUNT(DISTINCT customer_id) AS active_customers,
            SUM(order_count) AS total_orders,
            SUM(total_spent) AS total_revenue,
            AVG(total_spent) AS avg_revenue_per_customer
        FROM customer_metrics
        GROUP BY cohort_month, months_since_first_order
    ),

    -- CTE 6 : Taille initiale de chaque cohorte (M0)
    cohort_sizes AS (
        SELECT
            cohort_month,
            active_customers AS cohort_size
        FROM cohort_analysis
        WHERE months_since_first_order = 0
    )

-- Requête finale : Tableau de rétention avec LTV
SELECT
    ca.cohort_month,
    ca.months_since_first_order AS month_number,
    cs.cohort_size AS initial_size,
    ca.active_customers,
    -- Taux de rétention
    ROUND(100.0 * ca.active_customers / cs.cohort_size, 1) AS retention_pct,
    -- Revenue metrics
    ROUND(ca.total_revenue, 2) AS period_revenue,
    ROUND(ca.avg_revenue_per_customer, 2) AS avg_revenue_per_active_customer,
    -- LTV cumulé (simplifié - somme depuis M0)
    ROUND(SUM(ca.total_revenue) OVER (
        PARTITION BY ca.cohort_month
        ORDER BY ca.months_since_first_order
    ) / cs.cohort_size, 2) AS cumulative_ltv_per_customer
FROM cohort_analysis ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
WHERE ca.cohort_month >= DATE_FORMAT(DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH), '%Y-%m')
ORDER BY ca.cohort_month DESC, ca.months_since_first_order;
```

**Résultat** :
```
+--------------+--------------+--------------+------------------+---------------+----------------+----------------------------------+-------------------------------+
| cohort_month | month_number | initial_size | active_customers | retention_pct | period_revenue | avg_revenue_per_active_customer  | cumulative_ltv_per_customer   |
+--------------+--------------+--------------+------------------+---------------+----------------+----------------------------------+-------------------------------+
| 2025-01      |            0 |          150 |              150 |         100.0 |      75000.00  |                          500.00  |                       500.00  |
| 2025-01      |            1 |          150 |              120 |          80.0 |      60000.00  |                          500.00  |                       900.00  |
| 2025-01      |            2 |          150 |               90 |          60.0 |      45000.00  |                          500.00  |                      1200.00  |
| 2025-02      |            0 |          180 |              180 |         100.0 |      90000.00  |                          500.00  |                       500.00  |
| 2025-02      |            1 |          180 |              144 |          80.0 |      72000.00  |                          500.00  |                       900.00  |
+--------------+--------------+--------------+------------------+---------------+----------------+----------------------------------+-------------------------------+
```

💡 **Insights** :
- Rétention de 80% à M1, 60% à M2
- LTV moyen de ~1200€ après 3 mois
- Window function pour calculer le cumul par cohorte

---

## Exemple 3 : Analyse géographique multi-niveaux

Ventes par pays, avec drill-down et comparaisons.

```sql
WITH
    -- CTE 1 : Base de données des ventes enrichies
    sales_base AS (
        SELECT
            o.order_id,
            o.order_date,
            o.shipping_country,
            c.customer_segment,
            oi.product_id,
            p.category,
            oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100) AS item_revenue,
            oi.quantity * p.cost AS item_cost
        FROM orders o
        JOIN customers c ON o.customer_id = c.customer_id
        JOIN order_items oi ON o.order_id = oi.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE o.status = 'Completed'
          AND o.order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    ),

    -- CTE 2 : Agrégations par pays et catégorie
    country_category_sales AS (
        SELECT
            shipping_country,
            category,
            COUNT(DISTINCT order_id) AS order_count,
            SUM(item_revenue) AS revenue,
            SUM(item_cost) AS cost,
            SUM(item_revenue) - SUM(item_cost) AS profit
        FROM sales_base
        GROUP BY shipping_country, category
    ),

    -- CTE 3 : Totaux par pays (tous produits)
    country_totals AS (
        SELECT
            shipping_country,
            COUNT(DISTINCT order_id) AS total_orders,
            SUM(item_revenue) AS total_revenue,
            SUM(item_cost) AS total_cost,
            SUM(item_revenue) - SUM(item_cost) AS total_profit
        FROM sales_base
        GROUP BY shipping_country
    ),

    -- CTE 4 : Totaux globaux
    global_totals AS (
        SELECT
            SUM(item_revenue) AS global_revenue,
            SUM(item_cost) AS global_cost,
            SUM(item_revenue) - SUM(item_cost) AS global_profit
        FROM sales_base
    )

-- Requête finale : Rapport avec pourcentages et rankings
SELECT
    ccs.shipping_country,
    ccs.category,
    ccs.order_count,
    ROUND(ccs.revenue, 2) AS revenue,
    ROUND(ccs.profit, 2) AS profit,
    -- Marge
    ROUND(100.0 * ccs.profit / NULLIF(ccs.revenue, 0), 1) AS margin_pct,
    -- % du total pays
    ROUND(100.0 * ccs.revenue / ct.total_revenue, 1) AS pct_of_country,
    -- % du total global
    ROUND(100.0 * ccs.revenue / gt.global_revenue, 1) AS pct_of_global,
    -- Ranking dans le pays
    RANK() OVER (
        PARTITION BY ccs.shipping_country
        ORDER BY ccs.revenue DESC
    ) AS rank_in_country
FROM country_category_sales ccs
JOIN country_totals ct ON ccs.shipping_country = ct.shipping_country
CROSS JOIN global_totals gt
WHERE ccs.revenue > 1000  -- Filtrer les petites catégories
ORDER BY ccs.shipping_country, ccs.revenue DESC;
```

**Résultat** :
```
+------------------+-------------+-------------+-----------+----------+------------+----------------+---------------+-----------------+
| shipping_country | category    | order_count | revenue   | profit   | margin_pct | pct_of_country | pct_of_global | rank_in_country |
+------------------+-------------+-------------+-----------+----------+------------+----------------+---------------+-----------------+
| France           | Electronics |         245 | 450000.00 | 157500.00|       35.0 |           65.2 |          22.5 |               1 |
| France           | Accessories |         180 | 150000.00 |  60000.00|       40.0 |           21.7 |           7.5 |               2 |
| France           | Furniture   |          95 |  90000.00 |  27000.00|       30.0 |           13.0 |           4.5 |               3 |
| Germany          | Electronics |         198 | 380000.00 | 133000.00|       35.0 |           67.9 |          19.0 |               1 |
| Germany          | Accessories |         142 | 120000.00 |  48000.00|       40.0 |           21.4 |           6.0 |               2 |
+------------------+-------------+-------------+-----------+----------+------------+----------------+---------------+-----------------+
```

💡 **Techniques avancées** :
- **CROSS JOIN** avec global_totals pour calculer les pourcentages globaux
- **RANK() OVER (PARTITION BY ...)** pour classement par pays
- **Agrégations multiples** (par pays, par catégorie, globale)

---

## Exemple 4 : Impact des promotions

Analyser l'efficacité des promotions sur les ventes.

```sql
WITH
    -- CTE 1 : Périodes de promotion par produit
    promotion_periods AS (
        SELECT
            product_id,
            promotion_id,
            promotion_name,
            start_date,
            end_date,
            discount_pct
        FROM promotions
    ),

    -- CTE 2 : Ventes pendant les promotions
    sales_during_promotion AS (
        SELECT
            oi.product_id,
            pp.promotion_id,
            pp.promotion_name,
            o.order_date,
            oi.quantity,
            oi.unit_price,
            oi.quantity * oi.unit_price AS gross_revenue,
            -- Discount appliqué
            oi.quantity * oi.unit_price * oi.discount_pct / 100 AS discount_amount
        FROM order_items oi
        JOIN orders o ON oi.order_id = o.order_id
        JOIN promotion_periods pp ON oi.product_id = pp.product_id
            AND o.order_date BETWEEN pp.start_date AND pp.end_date
        WHERE o.status = 'Completed'
    ),

    -- CTE 3 : Ventes hors promotion (même produits, périodes comparables)
    sales_without_promotion AS (
        SELECT
            oi.product_id,
            o.order_date,
            oi.quantity,
            oi.unit_price,
            oi.quantity * oi.unit_price AS gross_revenue
        FROM order_items oi
        JOIN orders o ON oi.order_id = o.order_id
        WHERE o.status = 'Completed'
          AND NOT EXISTS (
              SELECT 1 FROM promotion_periods pp
              WHERE oi.product_id = pp.product_id
                AND o.order_date BETWEEN pp.start_date AND pp.end_date
          )
    ),

    -- CTE 4 : Agrégation ventes avec promo
    promo_summary AS (
        SELECT
            promotion_id,
            promotion_name,
            product_id,
            COUNT(*) AS sale_count,
            SUM(quantity) AS total_units_sold,
            SUM(gross_revenue) AS gross_revenue,
            SUM(discount_amount) AS total_discount_given,
            AVG(quantity) AS avg_units_per_sale
        FROM sales_during_promotion
        GROUP BY promotion_id, promotion_name, product_id
    ),

    -- CTE 5 : Agrégation ventes sans promo (baseline)
    baseline_summary AS (
        SELECT
            product_id,
            COUNT(*) AS sale_count,
            SUM(quantity) AS total_units_sold,
            SUM(gross_revenue) AS gross_revenue,
            AVG(quantity) AS avg_units_per_sale
        FROM sales_without_promotion
        GROUP BY product_id
    )

-- Requête finale : Comparaison et ROI
SELECT
    p.product_name,
    ps.promotion_name,
    ps.sale_count AS promo_sales,
    bs.sale_count AS baseline_sales,
    -- Uplift en ventes
    ps.sale_count - bs.sale_count AS sales_uplift,
    ROUND(100.0 * (ps.sale_count - bs.sale_count) / NULLIF(bs.sale_count, 0), 1) AS sales_uplift_pct,
    -- Revenue
    ROUND(ps.gross_revenue, 2) AS promo_revenue,
    ROUND(ps.total_discount_given, 2) AS discount_cost,
    ROUND(ps.gross_revenue - ps.total_discount_given, 2) AS net_revenue,
    -- ROI de la promotion
    ROUND(
        (ps.gross_revenue - ps.total_discount_given - bs.gross_revenue) /
        NULLIF(ps.total_discount_given, 0) * 100,
        1
    ) AS promo_roi_pct
FROM promo_summary ps
JOIN products p ON ps.product_id = p.product_id
LEFT JOIN baseline_summary bs ON ps.product_id = bs.product_id
ORDER BY promo_roi_pct DESC NULLS LAST;
```

**Résultat** :
```
+------------------+------------------+-------------+----------------+--------------+------------------+---------------+--------------+-------------+--------------+
| product_name     | promotion_name   | promo_sales | baseline_sales | sales_uplift | sales_uplift_pct | promo_revenue | discount_cost| net_revenue | promo_roi_pct|
+------------------+------------------+-------------+----------------+--------------+------------------+---------------+--------------+-------------+--------------+
| Laptop Pro 15"   | Black Friday 20% |          85 |             45 |           40 |             88.9 |     127500.00 |     25500.00 |   102000.00 |        294.1 |
| Monitor 4K 27"   | Summer Sale 15%  |          52 |             32 |           20 |             62.5 |      31200.00 |      4680.00 |    26520.00 |        467.5 |
| Keyboard Mech    | New Year 10%     |          38 |             28 |           10 |             35.7 |       5700.00 |       570.00 |     5130.00 |        800.0 |
+------------------+------------------+-------------+----------------+--------------+------------------+---------------+--------------+-------------+--------------+
```

💡 **Analyse** :
- Keyboard Mech : 800% ROI ! Excellent résultat
- Uplift moyen de 60% des ventes pendant promo
- Coût des discounts compensé largement par volume

---

## Stratégies d'optimisation

### 1. Ordre des jointures

MariaDB optimise automatiquement, mais on peut aider :

```sql
-- ❌ Mauvais ordre : grosse table en premier
SELECT ...
FROM big_log_table blt  -- 10M lignes
JOIN small_users su ON blt.user_id = su.id  -- 10k lignes
WHERE su.active = 1;

-- ✅ Meilleur ordre : filtrer d'abord
WITH active_users AS (
    SELECT id FROM small_users WHERE active = 1  -- ✅ 5k lignes
)
SELECT ...
FROM active_users au
JOIN big_log_table blt ON au.id = blt.user_id;  -- ✅ Jointure plus petite
```

### 2. Index appropriés

```sql
-- Analyser le plan d'exécution
EXPLAIN WITH ...
SELECT ...;

-- Créer des index sur les colonnes de jointure
CREATE INDEX idx_orders_customer ON orders(customer_id, order_date);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- Index composites pour filtres fréquents
CREATE INDEX idx_orders_status_date ON orders(status, order_date);
```

### 3. Filtrage précoce

```sql
-- ❌ Filtrage tardif
WITH all_sales AS (
    SELECT * FROM orders  -- ⚠️ Toutes les données
    JOIN order_items ON ...
)
SELECT * FROM all_sales WHERE order_date >= '2025-01-01';

-- ✅ Filtrage précoce
WITH recent_orders AS (
    SELECT * FROM orders
    WHERE order_date >= '2025-01-01'  -- ✅ Réduction immédiate
      AND status = 'Completed'
)
SELECT * FROM recent_orders JOIN order_items ON ...;
```

### 4. Limiter les colonnes

```sql
-- ❌ SELECT * inutile
WITH all_cols AS (
    SELECT * FROM big_table  -- ⚠️ 50 colonnes
)
SELECT col1, col2 FROM all_cols;

-- ✅ Sélection ciblée
WITH needed_cols AS (
    SELECT col1, col2, col3 FROM big_table  -- ✅ 3 colonnes
)
SELECT col1, col2 FROM needed_cols;
```

### 5. Éviter les agrégations multiples dans les CTE intermédiaires

```sql
-- ⚠️ Agrégation multiple peut être coûteuse
WITH summary AS (
    SELECT
        customer_id,
        COUNT(*) AS cnt,
        SUM(amount) AS total,
        AVG(amount) AS avg_amt,
        MIN(date) AS first,
        MAX(date) AS last,
        STDDEV(amount) AS stddev
    FROM orders
    GROUP BY customer_id
)
-- Si on n'utilise que quelques colonnes...

-- ✅ Mieux : calculer seulement ce qui est nécessaire
WITH summary AS (
    SELECT
        customer_id,
        COUNT(*) AS cnt,
        SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
)
```

---

## Stratégies de débogage

### Tester chaque CTE individuellement

```sql
-- Au lieu d'exécuter toute la requête complexe :
WITH cte1 AS (...),
     cte2 AS (...),
     cte3 AS (...)
SELECT * FROM cte3;

-- Tester étape par étape :
-- Étape 1
WITH cte1 AS (...) SELECT * FROM cte1 LIMIT 10;

-- Étape 2
WITH cte1 AS (...), cte2 AS (...) SELECT * FROM cte2 LIMIT 10;

-- Étape 3
WITH cte1 AS (...), cte2 AS (...), cte3 AS (...) SELECT * FROM cte3 LIMIT 10;
```

### Compter les lignes à chaque étape

```sql
WITH
    step1 AS (
        SELECT * FROM table1 WHERE condition
    ),
    step1_count AS (
        SELECT COUNT(*) AS cnt FROM step1
    ),
    step2 AS (
        SELECT * FROM step1 JOIN table2 ...
    ),
    step2_count AS (
        SELECT COUNT(*) AS cnt FROM step2
    )
SELECT
    'Step1' AS step, cnt FROM step1_count
UNION ALL
SELECT 'Step2' AS step, cnt FROM step2_count;
```

### Utiliser EXPLAIN EXTENDED

```sql
EXPLAIN EXTENDED
WITH ...
SELECT ...;

-- Puis voir les warnings pour plus de détails
SHOW WARNINGS;
```

---

## Bonnes pratiques de documentation

### Commenter chaque CTE

```sql
WITH
    /*
     * active_customers
     * But : Identifier les clients actifs (au moins 1 commande dans les 90 derniers jours)
     * Filtres : status = 'Completed', date >= -90j
     * Output : customer_id, last_order_date, order_count
     */
    active_customers AS (
        SELECT
            customer_id,
            MAX(order_date) AS last_order_date,
            COUNT(*) AS order_count
        FROM orders
        WHERE status = 'Completed'
          AND order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
        GROUP BY customer_id
    ),

    /*
     * customer_ltv
     * But : Calculer la valeur vie (LTV) par client
     * Join : active_customers + order_items pour le revenue total
     * Output : customer_id, total_revenue, avg_order_value
     */
    customer_ltv AS (
        SELECT
            ac.customer_id,
            SUM(oi.quantity * oi.unit_price) AS total_revenue,
            AVG(oi.quantity * oi.unit_price) AS avg_order_value
        FROM active_customers ac
        JOIN orders o ON ac.customer_id = o.customer_id
        JOIN order_items oi ON o.order_id = oi.order_id
        GROUP BY ac.customer_id
    )

SELECT * FROM customer_ltv;
```

### Utiliser des noms explicites

```sql
-- ❌ Noms vagues
WITH
    t1 AS (...),
    t2 AS (...),
    final AS (...)

-- ✅ Noms descriptifs
WITH
    completed_orders_last_year AS (...),
    product_revenue_summary AS (...),
    top_products_with_ratings AS (...)
```

### Indenter et formater

```sql
-- ❌ Difficile à lire
WITH cte1 AS(SELECT * FROM t1 WHERE x=1),cte2 AS(SELECT * FROM t1 JOIN t2 ON t1.id=t2.id)SELECT * FROM cte2;

-- ✅ Bien formaté
WITH
    cte1 AS (
        SELECT *
        FROM t1
        WHERE x = 1
    ),
    cte2 AS (
        SELECT *
        FROM t1
        JOIN t2 ON t1.id = t2.id
    )
SELECT *
FROM cte2;
```

---

## Cas d'usage complexes

### Exemple 5 : Dashboard analytique complet

```sql
WITH
    -- Période de référence
    date_params AS (
        SELECT
            DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY) AS period_start,
            CURRENT_DATE AS period_end,
            DATE_SUB(CURRENT_DATE, INTERVAL 60 DAY) AS comparison_start,
            DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY) AS comparison_end
    ),

    -- Ventes période actuelle
    current_period_sales AS (
        SELECT
            COUNT(DISTINCT o.order_id) AS order_count,
            COUNT(DISTINCT o.customer_id) AS customer_count,
            SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS revenue,
            AVG(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS avg_order_value
        FROM orders o
        CROSS JOIN date_params dp
        JOIN order_items oi ON o.order_id = oi.order_id
        WHERE o.status = 'Completed'
          AND o.order_date BETWEEN dp.period_start AND dp.period_end
    ),

    -- Ventes période précédente (comparaison)
    previous_period_sales AS (
        SELECT
            COUNT(DISTINCT o.order_id) AS order_count,
            COUNT(DISTINCT o.customer_id) AS customer_count,
            SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS revenue,
            AVG(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS avg_order_value
        FROM orders o
        CROSS JOIN date_params dp
        JOIN order_items oi ON o.order_id = oi.order_id
        WHERE o.status = 'Completed'
          AND o.order_date BETWEEN dp.comparison_start AND dp.comparison_end
    ),

    -- Top 5 produits
    top_products AS (
        SELECT
            p.product_name,
            SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS revenue
        FROM orders o
        CROSS JOIN date_params dp
        JOIN order_items oi ON o.order_id = oi.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE o.status = 'Completed'
          AND o.order_date BETWEEN dp.period_start AND dp.period_end
        GROUP BY p.product_name
        ORDER BY revenue DESC
        LIMIT 5
    ),

    -- Répartition par segment client
    segment_distribution AS (
        SELECT
            c.customer_segment,
            COUNT(DISTINCT o.order_id) AS order_count,
            SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct / 100)) AS revenue
        FROM orders o
        CROSS JOIN date_params dp
        JOIN customers c ON o.customer_id = c.customer_id
        JOIN order_items oi ON o.order_id = oi.order_id
        WHERE o.status = 'Completed'
          AND o.order_date BETWEEN dp.period_start AND dp.period_end
        GROUP BY c.customer_segment
    )

-- Résultat : Dashboard consolidé
SELECT
    'KPIs Période Actuelle' AS section,
    NULL AS metric,
    NULL AS value,
    NULL AS change_pct
UNION ALL
SELECT
    '',
    'Orders',
    CAST(cps.order_count AS CHAR),
    CONCAT(
        ROUND(100.0 * (cps.order_count - pps.order_count) / pps.order_count, 1),
        '%'
    )
FROM current_period_sales cps, previous_period_sales pps
UNION ALL
SELECT
    '',
    'Revenue',
    CONCAT('$', ROUND(cps.revenue, 2)),
    CONCAT(
        ROUND(100.0 * (cps.revenue - pps.revenue) / pps.revenue, 1),
        '%'
    )
FROM current_period_sales cps, previous_period_sales pps
UNION ALL
SELECT
    '',
    'Avg Order Value',
    CONCAT('$', ROUND(cps.avg_order_value, 2)),
    CONCAT(
        ROUND(100.0 * (cps.avg_order_value - pps.avg_order_value) / pps.avg_order_value, 1),
        '%'
    )
FROM current_period_sales cps, previous_period_sales pps
UNION ALL
SELECT
    'Top 5 Produits',
    NULL,
    NULL,
    NULL
UNION ALL
SELECT
    '',
    product_name,
    CONCAT('$', ROUND(revenue, 2)),
    NULL
FROM top_products
UNION ALL
SELECT
    'Répartition par Segment',
    NULL,
    NULL,
    NULL
UNION ALL
SELECT
    '',
    customer_segment,
    CONCAT(order_count, ' orders / $', ROUND(revenue, 2)),
    NULL
FROM segment_distribution;
```

**Résultat** : Dashboard complet avec KPIs, top produits, et segmentation.

---

## ⚠️ Pièges courants et solutions

### Piège 1 : Explosion cartésienne

```sql
-- ❌ DANGER : Jointure sans condition → produit cartésien
SELECT *
FROM orders o
JOIN order_items oi  -- ⚠️ Pas de ON clause
JOIN products p;

-- Résultat : orders × order_items × products (millions de lignes!)

-- ✅ SOLUTION : Toujours spécifier la condition
SELECT *
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;
```

### Piège 2 : Doublons dans les agrégations

```sql
-- ⚠️ Risque de doublons
SELECT
    c.customer_id,
    COUNT(*) AS order_count  -- ⚠️ Peut compter plusieurs fois
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id;  -- ⚠️ Multiple items → doublons

-- ✅ SOLUTION : DISTINCT dans les agrégations
SELECT
    c.customer_id,
    COUNT(DISTINCT o.order_id) AS order_count  -- ✅ Compte unique
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id;
```

### Piège 3 : NULL dans les LEFT JOIN

```sql
-- ⚠️ Agrégation incorrecte avec LEFT JOIN
SELECT
    p.product_id,
    COUNT(*) AS review_count  -- ⚠️ Compte aussi les NULL
FROM products p
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id;

-- ✅ SOLUTION : COUNT sur colonne non-NULL ou COALESCE
SELECT
    p.product_id,
    COUNT(pr.review_id) AS review_count  -- ✅ Ignore NULL
FROM products p
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id;
```

### Piège 4 : Performance avec OR

```sql
-- ❌ OR empêche l'utilisation d'index
SELECT *
FROM orders
WHERE customer_id = 123 OR order_date = '2025-01-01';  -- ⚠️ Lent

-- ✅ SOLUTION : UNION avec index séparés
SELECT * FROM orders WHERE customer_id = 123
UNION
SELECT * FROM orders WHERE order_date = '2025-01-01';
```

---

## ✅ Points clés à retenir

- 🏗️ **CTE multiples** : Décomposer les problèmes complexes en étapes logiques
- 🔗 **Ordre des jointures** : Filtrer d'abord, joindre ensuite
- 📊 **Agrégations** : Utiliser DISTINCT pour éviter les doublons
- ⚡ **Index** : Créer des index sur les colonnes de jointure et filtres
- 🎯 **Filtrage précoce** : WHERE dans les CTE, pas à la fin
- 💡 **Documentation** : Commenter chaque CTE et étape complexe
- 🐛 **Débogage** : Tester chaque CTE individuellement
- 📈 **Performance** : Analyser avec EXPLAIN, optimiser l'ordre
- ⚠️ **Pièges** : Attention aux produits cartésiens, NULL, doublons
- 🔄 **Window Functions** : Combiner avec CTE pour analyses avancées

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 JOIN Syntax](https://mariadb.com/kb/en/join-syntax/)
- [📖 SELECT Statement](https://mariadb.com/kb/en/select/)
- [📖 EXPLAIN](https://mariadb.com/kb/en/explain/)
- [📖 Optimization and Indexes](https://mariadb.com/kb/en/optimization-and-indexes/)

### Performance et optimisation
- [Use The Index, Luke](https://use-the-index-luke.com/) - Guide complet sur les index
- [MariaDB Query Optimization](https://mariadb.com/kb/en/query-optimizations/) - Techniques d'optimisation

### Patterns SQL avancés
- [Modern SQL](https://modern-sql.com/) - Patterns SQL modernes
- [SQL Performance Explained](https://sql-performance-explained.com/) - Livre de référence

---

## ➡️ Section suivante

**[4.6 Gestion des valeurs NULL : Logique ternaire](./06-gestion-valeurs-null.md)** : Maîtrisez la logique ternaire SQL (TRUE/FALSE/NULL) et évitez les pièges courants liés aux valeurs NULL dans vos requêtes complexes.

---


⏭️ [Gestion des valeurs NULL : Logique ternaire](/04-concepts-avances-sql/06-gestion-valeurs-null.md)

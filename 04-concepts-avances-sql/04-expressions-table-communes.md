🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 Expressions de table communes (CTE)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Maîtrise des sous-requêtes, jointures, GROUP BY

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les **CTE (Common Table Expressions)** et leurs avantages
- Utiliser la clause `WITH` pour structurer des requêtes complexes
- Créer des **CTE multiples** et les chaîner
- Combiner CTE avec **Window Functions**, **agrégations** et **jointures**
- Améliorer la **lisibilité** et la **maintenabilité** de vos requêtes SQL
- Comprendre les **différences** entre CTE, sous-requêtes et vues temporaires
- Optimiser les performances des requêtes avec CTE

---

## Introduction

### Qu'est-ce qu'une CTE ?

Une **Common Table Expression (CTE)** est un résultat de requête temporaire nommé qui existe uniquement pendant l'exécution d'une requête. C'est comme une "vue temporaire" que vous pouvez référencer dans la requête principale.

**Syntaxe de base** :
```sql
WITH nom_cte AS (
    SELECT ...
    FROM ...
    WHERE ...
)
SELECT * FROM nom_cte;
```

**Analogie** : Imaginez une CTE comme une **variable SQL** contenant un ensemble de résultats que vous pouvez réutiliser.

### Pourquoi utiliser des CTE ?

#### 1. **Lisibilité améliorée** 📖

**Sans CTE** (requête imbriquée complexe) :
```sql
SELECT
    dept,
    avg_salary
FROM (
    SELECT
        department AS dept,
        AVG(salary) AS avg_salary
    FROM (
        SELECT
            e.department,
            e.salary
        FROM employees e
        WHERE e.active = 1
    ) active_emp
    GROUP BY department
) dept_avg
WHERE avg_salary > 50000;
```

**Avec CTE** (beaucoup plus clair) :
```sql
WITH active_employees AS (
    SELECT department, salary
    FROM employees
    WHERE active = 1
),
department_averages AS (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM active_employees
    GROUP BY department
)
SELECT
    department,
    avg_salary
FROM department_averages
WHERE avg_salary > 50000;
```

💡 **Différence** : Avec les CTE, on lit le code de haut en bas, étape par étape, comme un algorithme.

#### 2. **Réutilisabilité** 🔄

Une CTE peut être référencée **plusieurs fois** dans la même requête :

```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 100000
)
SELECT
    (SELECT COUNT(*) FROM high_earners) AS total_high_earners,
    (SELECT AVG(salary) FROM high_earners) AS avg_high_earner_salary,
    (SELECT MAX(salary) FROM high_earners) AS max_salary;
```

Avec une sous-requête classique, il faudrait répéter 3 fois la même sous-requête !

#### 3. **Décomposition de problèmes complexes** 🧩

Les CTE permettent de découper un problème complexe en étapes logiques :

```sql
WITH
    -- Étape 1 : Identifier les clients actifs
    active_customers AS (...),

    -- Étape 2 : Calculer leurs achats totaux
    customer_purchases AS (...),

    -- Étape 3 : Segmenter par valeur
    customer_segments AS (...)

-- Étape 4 : Requête finale
SELECT * FROM customer_segments;
```

---

## Syntaxe et utilisation de base

### CTE simple

```sql
WITH employee_summary AS (
    SELECT
        department,
        COUNT(*) AS employee_count,
        AVG(salary) AS avg_salary,
        MAX(salary) AS max_salary
    FROM employees
    GROUP BY department
)
SELECT
    department,
    employee_count,
    ROUND(avg_salary, 2) AS avg_salary,
    max_salary
FROM employee_summary
WHERE employee_count > 5
ORDER BY avg_salary DESC;
```

**Résultat** :
```
+------------+----------------+------------+------------+
| department | employee_count | avg_salary | max_salary |
+------------+----------------+------------+------------+
| IT         |             15 |   75000.00 |  120000.00 |
| Sales      |             12 |   68000.00 |  105000.00 |
| HR         |              8 |   62000.00 |   95000.00 |
+------------+----------------+------------+------------+
```

### CTE multiples (chaînées)

Plusieurs CTE séparées par des virgules :

```sql
WITH
    -- CTE 1 : Employés actifs
    active_employees AS (
        SELECT id, name, department, salary
        FROM employees
        WHERE active = 1
    ),

    -- CTE 2 : Statistiques par département (utilise CTE 1)
    dept_stats AS (
        SELECT
            department,
            COUNT(*) AS emp_count,
            AVG(salary) AS avg_salary
        FROM active_employees
        GROUP BY department
    ),

    -- CTE 3 : Départements au-dessus de la moyenne
    above_avg_depts AS (
        SELECT department
        FROM dept_stats
        WHERE avg_salary > (SELECT AVG(avg_salary) FROM dept_stats)
    )

-- Requête finale : employés dans les départements performants
SELECT
    ae.name,
    ae.department,
    ae.salary,
    ds.avg_salary AS dept_avg_salary
FROM active_employees ae
JOIN dept_stats ds ON ae.department = ds.department
WHERE ae.department IN (SELECT department FROM above_avg_depts)
ORDER BY ae.department, ae.salary DESC;
```

💡 **Note** : Chaque CTE peut référencer les CTE définies **avant** elle, mais pas celles définies après.

---

## Exemples pratiques détaillés

### Exemple 1 : Analyse de ventes avec CTE multiples

```sql
CREATE TABLE sales (
    sale_id INT PRIMARY KEY,
    sale_date DATE,
    customer_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2)
);

INSERT INTO sales VALUES
(1, '2025-01-15', 101, 1, 2, 500.00),
(2, '2025-01-20', 101, 2, 1, 150.00),
(3, '2025-02-10', 102, 1, 1, 500.00),
(4, '2025-02-15', 101, 3, 3, 100.00),
(5, '2025-03-05', 103, 1, 1, 500.00),
(6, '2025-03-10', 102, 2, 2, 150.00);
```

#### Analyse complète : RFM (Recency, Frequency, Monetary)

```sql
WITH
    -- CTE 1 : Calcul du montant de chaque vente
    sales_with_amount AS (
        SELECT
            sale_id,
            sale_date,
            customer_id,
            quantity * unit_price AS sale_amount
        FROM sales
    ),

    -- CTE 2 : Métriques RFM par client
    customer_rfm AS (
        SELECT
            customer_id,
            -- Recency : jours depuis le dernier achat
            DATEDIFF(CURRENT_DATE, MAX(sale_date)) AS recency_days,
            -- Frequency : nombre d'achats
            COUNT(*) AS frequency,
            -- Monetary : montant total dépensé
            SUM(sale_amount) AS monetary_value,
            -- Panier moyen
            AVG(sale_amount) AS avg_basket_size
        FROM sales_with_amount
        GROUP BY customer_id
    ),

    -- CTE 3 : Segmentation des clients
    customer_segments AS (
        SELECT
            customer_id,
            recency_days,
            frequency,
            monetary_value,
            avg_basket_size,
            -- Segmentation basée sur RFM
            CASE
                WHEN recency_days <= 30 AND frequency >= 3 AND monetary_value >= 1000
                    THEN 'VIP'
                WHEN recency_days <= 60 AND frequency >= 2
                    THEN 'Loyal'
                WHEN recency_days <= 90
                    THEN 'Active'
                WHEN recency_days > 90
                    THEN 'At Risk'
                ELSE 'New'
            END AS segment
        FROM customer_rfm
    )

-- Requête finale : Distribution par segment
SELECT
    segment,
    COUNT(*) AS customer_count,
    ROUND(AVG(monetary_value), 2) AS avg_ltv,
    ROUND(AVG(frequency), 2) AS avg_frequency,
    ROUND(AVG(recency_days), 0) AS avg_recency_days
FROM customer_segments
GROUP BY segment
ORDER BY
    CASE segment
        WHEN 'VIP' THEN 1
        WHEN 'Loyal' THEN 2
        WHEN 'Active' THEN 3
        WHEN 'At Risk' THEN 4
        WHEN 'New' THEN 5
    END;
```

**Résultat** :
```
+---------+----------------+---------+---------------+------------------+
| segment | customer_count | avg_ltv | avg_frequency | avg_recency_days |
+---------+----------------+---------+---------------+------------------+
| VIP     |              1 | 1450.00 |          3.00 |               20 |
| Loyal   |              1 |  800.00 |          2.00 |               55 |
| Active  |              1 |  500.00 |          1.00 |               75 |
+---------+----------------+---------+---------------+------------------+
```

💡 **Avantages de cette approche** :
- Chaque étape est clairement identifiée et nommée
- Facile à déboguer (on peut tester chaque CTE individuellement)
- Maintenable : ajouter une nouvelle étape est simple

### Exemple 2 : Hiérarchie d'organisation avec CTE

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT,
    salary DECIMAL(10,2),
    department VARCHAR(50)
);

INSERT INTO employees VALUES
(1, 'Alice CEO', NULL, 200000, 'Executive'),
(2, 'Bob CTO', 1, 150000, 'Technology'),
(3, 'Carol CFO', 1, 150000, 'Finance'),
(4, 'David Dev Lead', 2, 120000, 'Technology'),
(5, 'Eve Dev', 4, 90000, 'Technology'),
(6, 'Frank Dev', 4, 85000, 'Technology'),
(7, 'Grace Accountant', 3, 80000, 'Finance');
```

#### Analyse avec plusieurs CTE non récursives

```sql
WITH
    -- CTE 1 : Managers avec leurs équipes
    managers_with_teams AS (
        SELECT
            m.id AS manager_id,
            m.name AS manager_name,
            m.department AS manager_dept,
            COUNT(e.id) AS team_size,
            SUM(e.salary) AS team_salary_cost,
            AVG(e.salary) AS avg_team_salary
        FROM employees m
        LEFT JOIN employees e ON e.manager_id = m.id
        GROUP BY m.id, m.name, m.department
    ),

    -- CTE 2 : Ratio salaire manager vs équipe
    manager_analysis AS (
        SELECT
            mwt.manager_name,
            mwt.manager_dept,
            mwt.team_size,
            e.salary AS manager_salary,
            mwt.avg_team_salary,
            -- Ratio du salaire du manager vs moyenne de son équipe
            ROUND(e.salary / NULLIF(mwt.avg_team_salary, 0), 2) AS salary_ratio
        FROM managers_with_teams mwt
        JOIN employees e ON e.id = mwt.manager_id
        WHERE mwt.team_size > 0  -- Seulement les vrais managers
    )

SELECT
    manager_name,
    manager_dept,
    team_size,
    manager_salary,
    ROUND(avg_team_salary, 2) AS avg_team_salary,
    salary_ratio,
    CASE
        WHEN salary_ratio >= 2.0 THEN '⚠️ Écart élevé'
        WHEN salary_ratio >= 1.5 THEN '⚡ Écart normal'
        ELSE '✅ Écart faible'
    END AS salary_gap_status
FROM manager_analysis
ORDER BY salary_ratio DESC;
```

**Résultat** :
```
+-----------------+--------------+-----------+----------------+------------------+--------------+--------------------+
| manager_name    | manager_dept | team_size | manager_salary | avg_team_salary  | salary_ratio | salary_gap_status  |
+-----------------+--------------+-----------+----------------+------------------+--------------+--------------------+
| Alice CEO       | Executive    |         2 |      200000.00 |        150000.00 |         1.33 | ✅ Écart faible    |
| Bob CTO         | Technology   |         1 |      150000.00 |        120000.00 |         1.25 | ✅ Écart faible    |
| David Dev Lead  | Technology   |         2 |      120000.00 |         87500.00 |         1.37 | ✅ Écart faible    |
| Carol CFO       | Finance      |         1 |      150000.00 |         80000.00 |         1.88 | ⚡ Écart normal    |
+-----------------+--------------+-----------+----------------+------------------+--------------+--------------------+
```

### Exemple 3 : Comparaison temporelle (Year-over-Year)

```sql
CREATE TABLE monthly_revenue (
    year INT,
    month INT,
    revenue DECIMAL(12,2)
);

INSERT INTO monthly_revenue VALUES
(2023, 1, 100000), (2023, 2, 110000), (2023, 3, 105000),
(2024, 1, 120000), (2024, 2, 130000), (2024, 3, 125000),
(2025, 1, 140000), (2025, 2, 155000), (2025, 3, 150000);
```

#### Analyse YoY avec CTE

```sql
WITH
    -- CTE 1 : Données formatées avec date complète
    revenue_formatted AS (
        SELECT
            year,
            month,
            revenue,
            CONCAT(year, '-', LPAD(month, 2, '0'), '-01') AS period_date
        FROM monthly_revenue
    ),

    -- CTE 2 : Revenue de l'année précédente
    revenue_with_previous_year AS (
        SELECT
            rf.year,
            rf.month,
            rf.revenue AS current_revenue,
            prev.revenue AS previous_year_revenue
        FROM revenue_formatted rf
        LEFT JOIN revenue_formatted prev
            ON rf.month = prev.month
            AND rf.year = prev.year + 1
    ),

    -- CTE 3 : Calculs de croissance
    growth_metrics AS (
        SELECT
            year,
            month,
            current_revenue,
            previous_year_revenue,
            current_revenue - COALESCE(previous_year_revenue, 0) AS absolute_growth,
            ROUND(
                100.0 * (current_revenue - COALESCE(previous_year_revenue, current_revenue))
                / NULLIF(previous_year_revenue, 0),
                2
            ) AS yoy_growth_pct
        FROM revenue_with_previous_year
    )

SELECT
    year,
    MONTHNAME(CONCAT(year, '-', LPAD(month, 2, '0'), '-01')) AS month_name,
    current_revenue,
    previous_year_revenue,
    absolute_growth,
    CONCAT(yoy_growth_pct, '%') AS yoy_growth,
    CASE
        WHEN yoy_growth_pct >= 20 THEN '🚀 Excellente croissance'
        WHEN yoy_growth_pct >= 10 THEN '📈 Bonne croissance'
        WHEN yoy_growth_pct >= 0 THEN '➡️ Croissance modérée'
        ELSE '📉 Décroissance'
    END AS performance_status
FROM growth_metrics
WHERE year >= 2024  -- Seulement les 2 dernières années
ORDER BY year, month;
```

**Résultat** :
```
+------+------------+-----------------+-----------------------+-----------------+------------+-----------------------+
| year | month_name | current_revenue | previous_year_revenue | absolute_growth | yoy_growth | performance_status    |
+------+------------+-----------------+-----------------------+-----------------+------------+-----------------------+
| 2024 | January    |       120000.00 |             100000.00 |        20000.00 | 20.00%     | 🚀 Excellente crois.  |
| 2024 | February   |       130000.00 |             110000.00 |        20000.00 | 18.18%     | 📈 Bonne croissance   |
| 2024 | March      |       125000.00 |             105000.00 |        20000.00 | 19.05%     | 📈 Bonne croissance   |
| 2025 | January    |       140000.00 |             120000.00 |        20000.00 | 16.67%     | 📈 Bonne croissance   |
| 2025 | February   |       155000.00 |             130000.00 |        25000.00 | 19.23%     | 📈 Bonne croissance   |
| 2025 | March      |       150000.00 |             125000.00 |        25000.00 | 20.00%     | 🚀 Excellente crois.  |
+------+------------+-----------------+-----------------------+-----------------+------------+-----------------------+
```

---

## CTE avec Window Functions

Les CTE sont particulièrement puissantes combinées aux Window Functions.

### Exemple 4 : Top N par catégorie

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2),
    sales INT
);

INSERT INTO products VALUES
(1, 'Laptop Pro', 'Electronics', 1500, 250),
(2, 'Laptop Air', 'Electronics', 1200, 300),
(3, 'Laptop Basic', 'Electronics', 800, 450),
(4, 'Mouse Pro', 'Accessories', 80, 500),
(5, 'Mouse Basic', 'Accessories', 25, 800),
(6, 'Keyboard Mech', 'Accessories', 150, 200),
(7, 'Monitor 4K', 'Electronics', 600, 180),
(8, 'Monitor HD', 'Electronics', 300, 320);
```

#### Top 2 produits par catégorie (revenue)

```sql
WITH
    -- CTE 1 : Calculer le revenue par produit
    product_revenue AS (
        SELECT
            id,
            name,
            category,
            price,
            sales,
            price * sales AS total_revenue
        FROM products
    ),

    -- CTE 2 : Ajouter un rang par catégorie
    ranked_products AS (
        SELECT
            name,
            category,
            total_revenue,
            ROW_NUMBER() OVER (
                PARTITION BY category
                ORDER BY total_revenue DESC
            ) AS rank_in_category
        FROM product_revenue
    )

-- Requête finale : Top 2 par catégorie
SELECT
    category,
    name,
    total_revenue,
    rank_in_category
FROM ranked_products
WHERE rank_in_category <= 2
ORDER BY category, rank_in_category;
```

**Résultat** :
```
+-------------+---------------+---------------+------------------+
| category    | name          | total_revenue | rank_in_category |
+-------------+---------------+---------------+------------------+
| Accessories | Mouse Basic   |     20000.00  |                1 |
| Accessories | Mouse Pro     |     40000.00  |                2 |
| Electronics | Laptop Basic  |    360000.00  |                1 |
| Electronics | Laptop Pro    |    375000.00  |                2 |
+-------------+---------------+---------------+------------------+
```

### Exemple 5 : Cumuls et moyennes mobiles

```sql
WITH
    -- CTE 1 : Revenue mensuel
    monthly_data AS (
        SELECT
            year,
            month,
            revenue,
            -- Ordre chronologique
            ROW_NUMBER() OVER (ORDER BY year, month) AS period_number
        FROM monthly_revenue
    ),

    -- CTE 2 : Calculs de window functions
    revenue_analytics AS (
        SELECT
            year,
            month,
            revenue,
            -- Cumul depuis le début
            SUM(revenue) OVER (ORDER BY year, month) AS cumulative_revenue,
            -- Moyenne mobile 3 mois
            AVG(revenue) OVER (
                ORDER BY year, month
                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
            ) AS moving_avg_3m,
            -- Croissance vs mois précédent
            revenue - LAG(revenue) OVER (ORDER BY year, month) AS mom_growth
        FROM monthly_data
    )

SELECT
    year,
    month,
    revenue,
    ROUND(cumulative_revenue, 2) AS cumulative_revenue,
    ROUND(moving_avg_3m, 2) AS moving_avg_3m,
    ROUND(mom_growth, 2) AS mom_growth,
    ROUND(100.0 * mom_growth / LAG(revenue) OVER (ORDER BY year, month), 2) AS mom_growth_pct
FROM revenue_analytics
ORDER BY year, month;
```

**Résultat** :
```
+------+-------+-----------+--------------------+----------------+------------+----------------+
| year | month | revenue   | cumulative_revenue | moving_avg_3m  | mom_growth | mom_growth_pct |
+------+-------+-----------+--------------------+----------------+------------+----------------+
| 2023 |     1 | 100000.00 |          100000.00 |     100000.00  |       NULL |           NULL |
| 2023 |     2 | 110000.00 |          210000.00 |     105000.00  |   10000.00 |          10.00 |
| 2023 |     3 | 105000.00 |          315000.00 |     105000.00  |   -5000.00 |          -4.55 |
| 2024 |     1 | 120000.00 |          435000.00 |     111666.67  |   15000.00 |          14.29 |
| 2024 |     2 | 130000.00 |          565000.00 |     118333.33  |   10000.00 |           8.33 |
| 2024 |     3 | 125000.00 |          690000.00 |     125000.00  |   -5000.00 |          -3.85 |
+------+-------+-----------+--------------------+----------------+------------+----------------+
```

---

## CTE vs Sous-requêtes vs Vues temporaires

### Comparaison

| Critère | CTE | Sous-requête | Vue temporaire | Vue standard |
|---------|-----|--------------|----------------|--------------|
| **Portée** | Une requête | Une requête | Session | Permanente |
| **Réutilisable** | ✅ Dans la requête | ❌ Non | ✅ Dans la session | ✅ Globale |
| **Lisibilité** | ✅✅ Excellente | ❌ Faible | ✅ Bonne | ✅ Bonne |
| **Performance** | 🟡 Variable | 🟡 Variable | 🟢 Indexable | 🟢 Indexable |
| **Persistance** | ❌ Non | ❌ Non | ❌ Session | ✅ Permanente |
| **Récursif** | ✅ Oui | ❌ Non | ❌ Non | ❌ Non |

### Exemple comparatif

#### Avec sous-requête (moins lisible)

```sql
SELECT
    e.name,
    e.salary,
    dept_avg.avg_salary
FROM employees e
JOIN (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary * 1.2;
```

#### Avec CTE (plus lisible)

```sql
WITH department_averages AS (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT
    e.name,
    e.salary,
    da.avg_salary AS dept_avg_salary
FROM employees e
JOIN department_averages da ON e.department = da.department
WHERE e.salary > da.avg_salary * 1.2;
```

#### Avec vue temporaire (pour réutilisation session)

```sql
CREATE TEMPORARY TABLE department_averages AS
SELECT
    department,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

-- Peut être utilisée dans plusieurs requêtes
SELECT * FROM department_averages WHERE avg_salary > 70000;
SELECT * FROM department_averages WHERE department = 'IT';

DROP TEMPORARY TABLE department_averages;
```

💡 **Quand utiliser quoi ?**

- **CTE** : Requête ponctuelle complexe, besoin de lisibilité
- **Sous-requête** : Logique simple, utilisée une seule fois
- **Vue temporaire** : Résultat réutilisé dans plusieurs requêtes de la session
- **Vue standard** : Logique métier réutilisée fréquemment, nécessite indexation

---

## Performance et optimisations

### Optimisation automatique par MariaDB

MariaDB peut optimiser les CTE de deux façons :

#### 1. **Inline Expansion (Matérialisation inline)**

La CTE est "collapsée" dans la requête principale si possible.

```sql
WITH simple_cte AS (
    SELECT id, name FROM employees WHERE active = 1
)
SELECT * FROM simple_cte WHERE department = 'IT';

-- MariaDB peut optimiser en :
SELECT id, name FROM employees WHERE active = 1 AND department = 'IT';
```

#### 2. **Materialization (Matérialisation)**

La CTE est exécutée une fois, résultat stocké en mémoire, puis réutilisé.

```sql
WITH expensive_cte AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
)
SELECT * FROM expensive_cte e1
JOIN expensive_cte e2 ON e1.avg_sal > e2.avg_sal * 1.1;

-- MariaDB matérialise expensive_cte une fois
```

### Analyse avec EXPLAIN

```sql
EXPLAIN WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT * FROM dept_avg WHERE avg_salary > 50000;
```

**Vérifier** :
- `Using temporary` : La CTE est matérialisée
- Absence de `Using temporary` : Inline expansion
- `rows` : Nombre de lignes traitées

### 💡 Bonnes pratiques de performance

#### 1. **Filtrer tôt**

```sql
-- ❌ Filtrage tardif
WITH all_sales AS (
    SELECT * FROM sales  -- ⚠️ Toutes les lignes
)
SELECT * FROM all_sales WHERE year = 2025;

-- ✅ Filtrage précoce
WITH sales_2025 AS (
    SELECT * FROM sales WHERE year = 2025  -- ✅ Filtrage immédiat
)
SELECT * FROM sales_2025;
```

#### 2. **Limiter les colonnes**

```sql
-- ❌ Sélection inutile
WITH all_columns AS (
    SELECT * FROM big_table  -- ⚠️ Toutes les colonnes
)
SELECT id, name FROM all_columns;

-- ✅ Sélection ciblée
WITH needed_columns AS (
    SELECT id, name FROM big_table  -- ✅ Seulement ce qui est nécessaire
)
SELECT * FROM needed_columns;
```

#### 3. **Index sur les colonnes de jointure**

```sql
-- Si la CTE est jointe souvent, indexer les colonnes de jointure
CREATE INDEX idx_employees_dept ON employees(department);

WITH dept_stats AS (
    SELECT department, COUNT(*) AS cnt
    FROM employees
    GROUP BY department
)
SELECT e.*, ds.cnt
FROM employees e
JOIN dept_stats ds ON e.department = ds.department;  -- ✅ Index utilisé
```

#### 4. **Éviter les CTE trop larges**

```sql
-- ⚠️ CTE avec millions de lignes
WITH huge_cte AS (
    SELECT * FROM log_table  -- 10M lignes
)
SELECT COUNT(*) FROM huge_cte WHERE user_id = 123;

-- ✅ Mieux : filtrer dans la CTE
WITH filtered_logs AS (
    SELECT * FROM log_table WHERE user_id = 123  -- Quelques lignes
)
SELECT COUNT(*) FROM filtered_logs;
```

---

## Cas d'usage avancés

### Exemple 6 : Pipeline de transformation de données

Simuler un ETL (Extract, Transform, Load) avec CTE.

```sql
WITH
    -- EXTRACT : Récupération des données brutes
    raw_data AS (
        SELECT
            order_id,
            customer_id,
            LOWER(TRIM(product_name)) AS product_name,
            quantity,
            unit_price,
            order_date
        FROM raw_orders
        WHERE order_date >= '2025-01-01'
    ),

    -- TRANSFORM 1 : Nettoyage et validation
    cleaned_data AS (
        SELECT
            order_id,
            customer_id,
            product_name,
            CASE
                WHEN quantity <= 0 THEN 1  -- Correction valeurs invalides
                ELSE quantity
            END AS quantity,
            CASE
                WHEN unit_price <= 0 THEN NULL  -- Marquer prix invalides
                ELSE unit_price
            END AS unit_price,
            order_date
        FROM raw_data
    ),

    -- TRANSFORM 2 : Enrichissement
    enriched_data AS (
        SELECT
            cd.*,
            c.customer_name,
            c.customer_segment,
            p.product_category,
            cd.quantity * cd.unit_price AS line_total
        FROM cleaned_data cd
        LEFT JOIN customers c ON cd.customer_id = c.id
        LEFT JOIN products p ON cd.product_name = p.name
    ),

    -- TRANSFORM 3 : Agrégation métier
    business_metrics AS (
        SELECT
            customer_segment,
            product_category,
            COUNT(DISTINCT customer_id) AS unique_customers,
            COUNT(DISTINCT order_id) AS total_orders,
            SUM(line_total) AS total_revenue,
            AVG(line_total) AS avg_order_value
        FROM enriched_data
        WHERE line_total IS NOT NULL  -- Exclure prix invalides
        GROUP BY customer_segment, product_category
    )

-- LOAD : Résultat final
SELECT
    customer_segment,
    product_category,
    unique_customers,
    total_orders,
    ROUND(total_revenue, 2) AS total_revenue,
    ROUND(avg_order_value, 2) AS avg_order_value,
    ROUND(100.0 * total_revenue / SUM(total_revenue) OVER (), 2) AS revenue_pct
FROM business_metrics
ORDER BY total_revenue DESC;
```

### Exemple 7 : Détection d'anomalies

Identifier les transactions suspectes.

```sql
WITH
    -- CTE 1 : Statistiques par client
    customer_stats AS (
        SELECT
            customer_id,
            COUNT(*) AS transaction_count,
            AVG(amount) AS avg_amount,
            STDDEV(amount) AS stddev_amount,
            MAX(amount) AS max_amount
        FROM transactions
        WHERE transaction_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
        GROUP BY customer_id
    ),

    -- CTE 2 : Transactions avec z-score
    transactions_with_zscore AS (
        SELECT
            t.transaction_id,
            t.customer_id,
            t.amount,
            t.transaction_date,
            cs.avg_amount,
            cs.stddev_amount,
            -- Calcul du z-score (écart à la moyenne en écarts-types)
            (t.amount - cs.avg_amount) / NULLIF(cs.stddev_amount, 0) AS zscore
        FROM transactions t
        JOIN customer_stats cs ON t.customer_id = cs.customer_id
        WHERE t.transaction_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
    ),

    -- CTE 3 : Classification des anomalies
    anomaly_detection AS (
        SELECT
            transaction_id,
            customer_id,
            amount,
            transaction_date,
            ROUND(avg_amount, 2) AS customer_avg,
            ROUND(zscore, 2) AS zscore,
            CASE
                WHEN ABS(zscore) > 3 THEN '🔴 Anomalie sévère'
                WHEN ABS(zscore) > 2 THEN '🟡 Anomalie modérée'
                ELSE '✅ Normal'
            END AS anomaly_status
        FROM transactions_with_zscore
    )

SELECT *
FROM anomaly_detection
WHERE anomaly_status IN ('🔴 Anomalie sévère', '🟡 Anomalie modérée')
ORDER BY ABS(zscore) DESC;
```

---

## Combinaison CTE + Récursivité

Les CTE récursives méritent une section dédiée (voir 4.1), mais voici un rappel rapide :

```sql
WITH RECURSIVE hierarchy AS (
    -- Ancre
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    -- Récursion
    SELECT e.id, e.name, e.manager_id, h.level + 1
    FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy ORDER BY level, name;
```

💡 **Différence** : `WITH RECURSIVE` pour les hiérarchies, `WITH` simple pour les étapes de transformation.

---

## ⚠️ Pièges courants et solutions

### Piège 1 : Référence circulaire

```sql
-- ❌ ERREUR : cte2 référence cte1 qui référence cte2
WITH
    cte1 AS (SELECT * FROM cte2),  -- ⚠️ cte2 pas encore définie
    cte2 AS (SELECT * FROM cte1)
SELECT * FROM cte1;

-- ✅ SOLUTION : Ordre correct
WITH
    cte1 AS (SELECT * FROM table1),
    cte2 AS (SELECT * FROM cte1)  -- ✅ cte1 déjà définie
SELECT * FROM cte2;
```

### Piège 2 : Nom de colonne ambigu

```sql
-- ⚠️ Ambiguïté
WITH dept_data AS (
    SELECT id, name FROM departments  -- 'id' et 'name'
)
SELECT id, name  -- ⚠️ De quelle table ?
FROM dept_data
JOIN employees ON dept_data.id = employees.department_id;

-- ✅ SOLUTION : Alias explicites
WITH dept_data AS (
    SELECT id AS dept_id, name AS dept_name FROM departments
)
SELECT
    dd.dept_id,
    dd.dept_name,
    e.id AS emp_id,
    e.name AS emp_name
FROM dept_data dd
JOIN employees e ON dd.dept_id = e.department_id;
```

### Piège 3 : Oublier le nommage des colonnes

```sql
-- ⚠️ Colonne sans nom
WITH summary AS (
    SELECT department, COUNT(*) FROM employees GROUP BY department
)
SELECT * FROM summary;  -- ⚠️ La colonne COUNT(*) s'appelle "COUNT(*)"

-- ✅ SOLUTION : Nommer explicitement
WITH summary AS (
    SELECT department, COUNT(*) AS employee_count
    FROM employees
    GROUP BY department
)
SELECT * FROM summary;
```

### Piège 4 : CTE non utilisée

```sql
-- ⚠️ CTE définie mais jamais utilisée (pas d'erreur, mais inutile)
WITH unused_cte AS (
    SELECT expensive_computation()  -- ⚠️ Calcul inutile
)
SELECT * FROM other_table;  -- unused_cte n'est pas référencée

-- ✅ Supprimer les CTE inutilisées
SELECT * FROM other_table;
```

---

## ✅ Points clés à retenir

- 📖 **Lisibilité** : Les CTE rendent le code SQL beaucoup plus lisible et maintenable
- 🔄 **Réutilisabilité** : Une CTE peut être référencée plusieurs fois dans la même requête
- 🧩 **Décomposition** : Idéal pour découper des problèmes complexes en étapes logiques
- 🔗 **Chaînage** : Les CTE peuvent se référencer entre elles (dans l'ordre de définition)
- 🎯 **Clause WITH** : Syntaxe standard SQL:1999, portable entre SGBD
- 🚀 **Performance** : MariaDB optimise automatiquement (inline ou matérialisation)
- 🔁 **Récursivité** : `WITH RECURSIVE` pour les structures hiérarchiques (voir 4.1)
- 💡 **Alternative** : Préférer les CTE aux sous-requêtes imbriquées complexes
- ⚠️ **Portée** : Une CTE n'existe que pendant l'exécution de la requête
- 📊 **Combinable** : Excellent avec Window Functions, agrégations, jointures

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 WITH Clause (Common Table Expressions)](https://mariadb.com/kb/en/with/)
- [📖 Recursive Common Table Expressions](https://mariadb.com/kb/en/recursive-common-table-expressions-overview/)
- [📖 EXPLAIN Output](https://mariadb.com/kb/en/explain/)

### Standards SQL
- [SQL:1999](https://en.wikipedia.org/wiki/SQL:1999) - Introduction des CTEs dans le standard

### Articles et tutoriels
- [Modern SQL: WITH Clause](https://modern-sql.com/feature/with) - Explications détaillées
- [Use The Index, Luke: CTEs](https://use-the-index-luke.com/sql/testing-where-clause) - Perspective performance

### Comparaisons
- [CTE vs Temporary Tables](https://www.sqlshack.com/difference-between-cte-and-temp-table-and-table-variable/) - Quand utiliser quoi

---

## ➡️ Section suivante

**[4.5 Requêtes complexes multi-tables](./05-requetes-complexes-multi-tables.md)** : Combinez tout ce que vous avez appris (CTE, Window Functions, jointures) pour construire des requêtes analytiques sophistiquées sur plusieurs tables.

---


⏭️ [Requêtes complexes multi-tables](/04-concepts-avances-sql/05-requetes-complexes-multi-tables.md)

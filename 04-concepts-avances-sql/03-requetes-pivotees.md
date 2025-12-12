🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 Requêtes pivotées et transformations

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Maîtrise des GROUP BY, CASE WHEN, agrégations

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les concepts de **PIVOT** (lignes → colonnes) et **UNPIVOT** (colonnes → lignes)
- Implémenter des pivots dans MariaDB avec `CASE WHEN` et `GROUP BY`
- Créer des **rapports croisés** et **tableaux de bord** dynamiques
- Transformer des structures de données pour l'analyse et la présentation
- Utiliser les **agrégations conditionnelles** efficacement
- Appliquer ces techniques à des cas d'usage réels (sales reports, analytics)

---

## Introduction

### Qu'est-ce qu'une requête pivotée ?

Une **requête pivotée** (PIVOT) transforme des données organisées en lignes en colonnes, créant ainsi une vue matricielle des données. C'est l'équivalent SQL d'un tableau croisé dynamique Excel.

**Transformation PIVOT** : Lignes → Colonnes
```
Avant (format ligne)          Après (format pivoté)
+------+-------+-------+       +------+-----+-----+-----+
| Mois | Prod  | Vente |       | Prod | Jan | Fev | Mar |
+------+-------+-------+       +------+-----+-----+-----+
| Jan  | A     |   100 |       | A    | 100 | 150 | 200 |
| Jan  | B     |    80 |  =>   | B    |  80 | 120 | 160 |
| Fev  | A     |   150 |       | C    |  60 |  90 | 120 |
| Fev  | B     |   120 |       +------+-----+-----+-----+
| ...                 |
```

**Transformation UNPIVOT** : Colonnes → Lignes (inverse)
```
Avant (format pivoté)         Après (format normalisé)
+------+-----+-----+-----+     +------+-------+-------+
| Prod | Jan | Fev | Mar |     | Prod | Mois  | Vente |
+------+-----+-----+-----+     +------+-------+-------+
| A    | 100 | 150 | 200 |     | A    | Jan   |   100 |
| B    |  80 | 120 | 160 | =>  | A    | Fev   |   150 |
| C    |  60 |  90 | 120 |     | A    | Mar   |   200 |
+------+-----+-----+-----+     | B    | Jan   |    80 |
                               | ...              ... |
```

### Pourquoi pivoter des données ?

**Avantages du PIVOT** :
- 📊 **Lisibilité** : Vue matricielle plus intuitive pour les humains
- 📈 **Rapports** : Format adapté aux tableaux de bord et reporting
- 🔍 **Comparaisons** : Facilite la comparaison entre catégories
- 📉 **Visualisation** : Format optimal pour les graphiques

**Avantages de l'UNPIVOT** :
- 🗄️ **Normalisation** : Retour à un format relationnel standard
- 🔄 **Traitement** : Plus facile à manipuler programmatiquement
- 📊 **Agrégations** : Simplifie les calculs sur plusieurs colonnes
- 💾 **Stockage** : Souvent plus efficace en espace disque

### MariaDB et le PIVOT

⚠️ **Important** : Contrairement à SQL Server ou Oracle, MariaDB **n'a pas d'opérateur PIVOT/UNPIVOT natif**. Nous devons donc utiliser des techniques alternatives :

- **Pour PIVOT** : `CASE WHEN` + `GROUP BY` + Agrégations
- **Pour UNPIVOT** : `UNION ALL` ou tables temporaires
- **Avancé** : Procédures stockées pour pivots dynamiques

💡 Cette limitation nous force à être plus créatifs, mais offre aussi plus de flexibilité !

---

## PIVOT : Transformer lignes en colonnes

### Exemple 1 : Ventes par mois et produit (Simple)

#### Structure de données

```sql
CREATE TABLE sales (
    sale_date DATE,
    product VARCHAR(50),
    amount DECIMAL(10,2)
);

INSERT INTO sales VALUES
('2025-01-15', 'Laptop', 1200.00),
('2025-01-20', 'Mouse', 25.00),
('2025-01-25', 'Laptop', 1300.00),
('2025-02-10', 'Laptop', 1250.00),
('2025-02-15', 'Mouse', 30.00),
('2025-02-20', 'Keyboard', 75.00),
('2025-03-05', 'Laptop', 1400.00),
('2025-03-10', 'Mouse', 28.00),
('2025-03-15', 'Keyboard', 80.00),
('2025-03-20', 'Monitor', 350.00);
```

#### Données brutes (format ligne)

```sql
SELECT
    MONTHNAME(sale_date) AS mois,
    product,
    SUM(amount) AS total_ventes
FROM sales
GROUP BY MONTHNAME(sale_date), product
ORDER BY MONTH(sale_date), product;
```

**Résultat** :
```
+----------+----------+--------------+
| mois     | product  | total_ventes |
+----------+----------+--------------+
| January  | Laptop   |      2500.00 |
| January  | Mouse    |        25.00 |
| February | Keyboard |        75.00 |
| February | Laptop   |      1250.00 |
| February | Mouse    |        30.00 |
| March    | Keyboard |        80.00 |
| March    | Laptop   |      1400.00 |
| March    | Monitor  |       350.00 |
| March    | Mouse    |        28.00 |
+----------+----------+--------------+
```

#### Requête PIVOT : Produits en colonnes

```sql
SELECT
    MONTHNAME(sale_date) AS mois,

    -- Chaque produit devient une colonne avec CASE WHEN + SUM
    SUM(CASE WHEN product = 'Laptop' THEN amount ELSE 0 END) AS laptop,
    SUM(CASE WHEN product = 'Mouse' THEN amount ELSE 0 END) AS mouse,
    SUM(CASE WHEN product = 'Keyboard' THEN amount ELSE 0 END) AS keyboard,
    SUM(CASE WHEN product = 'Monitor' THEN amount ELSE 0 END) AS monitor,

    -- Total par mois
    SUM(amount) AS total_mois
FROM sales
GROUP BY MONTHNAME(sale_date), MONTH(sale_date)
ORDER BY MONTH(sale_date);
```

**Résultat pivoté** :
```
+----------+---------+-------+----------+---------+------------+
| mois     | laptop  | mouse | keyboard | monitor | total_mois |
+----------+---------+-------+----------+---------+------------+
| January  | 2500.00 | 25.00 |     0.00 |    0.00 |    2525.00 |
| February | 1250.00 | 30.00 |    75.00 |    0.00 |    1355.00 |
| March    | 1400.00 | 28.00 |    80.00 |  350.00 |    1858.00 |
+----------+---------+-------+----------+---------+------------+
```

💡 **Explication** :
- `CASE WHEN product = 'Laptop' THEN amount ELSE 0` : Prend le montant si Laptop, sinon 0
- `SUM(...)` : Agrège tous les montants par mois
- `GROUP BY MONTHNAME(sale_date)` : Une ligne par mois

### Exemple 2 : Comptages conditionnels

Nombre de ventes par produit et par mois.

```sql
SELECT
    MONTHNAME(sale_date) AS mois,

    -- COUNT conditionnel avec CASE
    COUNT(CASE WHEN product = 'Laptop' THEN 1 END) AS nb_laptop,
    COUNT(CASE WHEN product = 'Mouse' THEN 1 END) AS nb_mouse,
    COUNT(CASE WHEN product = 'Keyboard' THEN 1 END) AS nb_keyboard,
    COUNT(CASE WHEN product = 'Monitor' THEN 1 END) AS nb_monitor,

    COUNT(*) AS total_transactions
FROM sales
GROUP BY MONTHNAME(sale_date), MONTH(sale_date)
ORDER BY MONTH(sale_date);
```

**Résultat** :
```
+----------+-----------+----------+-------------+------------+--------------------+
| mois     | nb_laptop | nb_mouse | nb_keyboard | nb_monitor | total_transactions |
+----------+-----------+----------+-------------+------------+--------------------+
| January  |         2 |        1 |           0 |          0 |                  3 |
| February |         1 |        1 |           1 |          0 |                  3 |
| March    |         1 |        1 |           1 |          1 |                  4 |
+----------+-----------+----------+-------------+------------+--------------------+
```

### Exemple 3 : Moyennes et métriques multiples

Calcul de moyenne, min, max par produit et par mois.

```sql
SELECT
    MONTHNAME(sale_date) AS mois,

    -- Moyennes par produit
    AVG(CASE WHEN product = 'Laptop' THEN amount END) AS avg_laptop,
    AVG(CASE WHEN product = 'Mouse' THEN amount END) AS avg_mouse,

    -- Max par produit
    MAX(CASE WHEN product = 'Laptop' THEN amount END) AS max_laptop,
    MAX(CASE WHEN product = 'Mouse' THEN amount END) AS max_mouse,

    -- Min par produit
    MIN(CASE WHEN product = 'Laptop' THEN amount END) AS min_laptop,
    MIN(CASE WHEN product = 'Mouse' THEN amount END) AS min_mouse
FROM sales
GROUP BY MONTHNAME(sale_date), MONTH(sale_date)
ORDER BY MONTH(sale_date);
```

**Résultat** :
```
+----------+------------+-----------+------------+-----------+------------+-----------+
| mois     | avg_laptop | avg_mouse | max_laptop | max_mouse | min_laptop | min_mouse |
+----------+------------+-----------+------------+-----------+------------+-----------+
| January  |    1250.00 |     25.00 |    1300.00 |     25.00 |    1200.00 |     25.00 |
| February |    1250.00 |     30.00 |    1250.00 |     30.00 |    1250.00 |     30.00 |
| March    |    1400.00 |     28.00 |    1400.00 |     28.00 |    1400.00 |     28.00 |
+----------+------------+-----------+------------+-----------+------------+-----------+
```

💡 **Note** : Utiliser `CASE WHEN ... END` sans `ELSE` retourne `NULL` pour les non-correspondances, ce qui est ignoré par les fonctions d'agrégation comme `AVG`, `MIN`, `MAX`.

---

## PIVOT avancé : Cas d'usage complexes

### Exemple 4 : Tableau de bord RH multi-dimensions

Analyser les employés par département, niveau et tranche de salaire.

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    level VARCHAR(20),
    salary DECIMAL(10,2)
);

INSERT INTO employees VALUES
(1, 'Alice', 'IT', 'Senior', 90000),
(2, 'Bob', 'IT', 'Junior', 50000),
(3, 'Charlie', 'IT', 'Senior', 95000),
(4, 'Diana', 'Sales', 'Senior', 85000),
(5, 'Eve', 'Sales', 'Junior', 45000),
(6, 'Frank', 'Sales', 'Mid', 65000),
(7, 'Grace', 'HR', 'Senior', 80000),
(8, 'Henry', 'HR', 'Junior', 48000);
```

#### Pivot : Nombre d'employés par niveau et département

```sql
SELECT
    department,

    COUNT(CASE WHEN level = 'Junior' THEN 1 END) AS junior,
    COUNT(CASE WHEN level = 'Mid' THEN 1 END) AS mid,
    COUNT(CASE WHEN level = 'Senior' THEN 1 END) AS senior,

    COUNT(*) AS total,

    -- Pourcentage de seniors
    ROUND(100.0 * COUNT(CASE WHEN level = 'Senior' THEN 1 END) / COUNT(*), 1) AS pct_senior
FROM employees
GROUP BY department
ORDER BY department;
```

**Résultat** :
```
+------------+--------+-----+--------+-------+------------+
| department | junior | mid | senior | total | pct_senior |
+------------+--------+-----+--------+-------+------------+
| HR         |      1 |   0 |      1 |     2 |       50.0 |
| IT         |      1 |   0 |      2 |     3 |       66.7 |
| Sales      |      1 |   1 |      1 |     3 |       33.3 |
+------------+--------+-----+--------+-------+------------+
```

#### Pivot : Masse salariale par niveau et département

```sql
SELECT
    department,

    SUM(CASE WHEN level = 'Junior' THEN salary ELSE 0 END) AS salaire_junior,
    SUM(CASE WHEN level = 'Mid' THEN salary ELSE 0 END) AS salaire_mid,
    SUM(CASE WHEN level = 'Senior' THEN salary ELSE 0 END) AS salaire_senior,

    SUM(salary) AS masse_salariale_totale,

    -- Salaire moyen par département
    ROUND(AVG(salary), 2) AS salaire_moyen_dept
FROM employees
GROUP BY department
ORDER BY masse_salariale_totale DESC;
```

**Résultat** :
```
+------------+----------------+-------------+----------------+-------------------------+--------------------+
| department | salaire_junior | salaire_mid | salaire_senior | masse_salariale_totale  | salaire_moyen_dept |
+------------+----------------+-------------+----------------+-------------------------+--------------------+
| IT         |       50000.00 |        0.00 |      185000.00 |               235000.00 |           78333.33 |
| Sales      |       45000.00 |    65000.00 |       85000.00 |               195000.00 |           65000.00 |
| HR         |       48000.00 |        0.00 |       80000.00 |               128000.00 |           64000.00 |
+------------+----------------+-------------+----------------+-------------------------+--------------------+
```

### Exemple 5 : Analyse de cohortes (E-commerce)

Analyser le comportement d'achat des clients par cohorte mensuelle.

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    amount DECIMAL(10,2)
);

INSERT INTO orders VALUES
(1, 101, '2025-01-15', 100),
(2, 101, '2025-02-10', 150),
(3, 101, '2025-03-05', 200),
(4, 102, '2025-01-20', 80),
(5, 102, '2025-02-15', 90),
(6, 103, '2025-02-05', 120),
(7, 103, '2025-03-10', 130),
(8, 104, '2025-03-01', 200);
```

#### Pivot : Analyse de rétention par cohorte

```sql
WITH first_purchase AS (
    -- Date du premier achat par client
    SELECT
        customer_id,
        DATE_FORMAT(MIN(order_date), '%Y-%m') AS cohorte
    FROM orders
    GROUP BY customer_id
),
customer_activity AS (
    -- Activité mensuelle de chaque client
    SELECT
        o.customer_id,
        fp.cohorte,
        DATE_FORMAT(o.order_date, '%Y-%m') AS mois_achat,
        -- Mois depuis la première commande (M0, M1, M2...)
        PERIOD_DIFF(
            EXTRACT(YEAR_MONTH FROM o.order_date),
            EXTRACT(YEAR_MONTH FROM MIN(o.order_date) OVER (PARTITION BY o.customer_id))
        ) AS mois_depuis_premiere_commande
    FROM orders o
    JOIN first_purchase fp ON o.customer_id = fp.customer_id
)
SELECT
    cohorte,

    -- Nombre de clients actifs à M0, M1, M2
    COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 0 THEN customer_id END) AS M0,
    COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 1 THEN customer_id END) AS M1,
    COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 2 THEN customer_id END) AS M2,

    -- Taux de rétention
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 1 THEN customer_id END) /
          NULLIF(COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 0 THEN customer_id END), 0), 1) AS retention_M1_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 2 THEN customer_id END) /
          NULLIF(COUNT(DISTINCT CASE WHEN mois_depuis_premiere_commande = 0 THEN customer_id END), 0), 1) AS retention_M2_pct
FROM customer_activity
GROUP BY cohorte
ORDER BY cohorte;
```

**Résultat** :
```
+---------+----+----+----+------------------+------------------+
| cohorte | M0 | M1 | M2 | retention_M1_pct | retention_M2_pct |
+---------+----+----+----+------------------+------------------+
| 2025-01 |  2 |  2 |  1 |            100.0 |             50.0 |
| 2025-02 |  1 |  1 |  0 |            100.0 |              0.0 |
| 2025-03 |  1 |  0 |  0 |              0.0 |              0.0 |
+---------+----+----+----+------------------+------------------+
```

💡 **Interprétation** :
- Cohorte Jan 2025 : 2 clients acquis, 100% reviennent en M1, 50% en M2
- Excellente rétention !

---

## UNPIVOT : Transformer colonnes en lignes

### Exemple 6 : Dénormalisation de données

Vous avez un tableau avec plusieurs colonnes de ventes à transformer en format ligne.

```sql
CREATE TABLE sales_pivot (
    product VARCHAR(50),
    jan DECIMAL(10,2),
    feb DECIMAL(10,2),
    mar DECIMAL(10,2)
);

INSERT INTO sales_pivot VALUES
('Laptop', 2500.00, 1250.00, 1400.00),
('Mouse', 25.00, 30.00, 28.00),
('Keyboard', 0.00, 75.00, 80.00);
```

**Données actuelles (format pivoté)** :
```
+----------+---------+---------+---------+
| product  | jan     | feb     | mar     |
+----------+---------+---------+---------+
| Laptop   | 2500.00 | 1250.00 | 1400.00 |
| Mouse    |   25.00 |   30.00 |   28.00 |
| Keyboard |    0.00 |   75.00 |   80.00 |
+----------+---------+---------+---------+
```

#### Méthode 1 : UNION ALL (Simple mais verbeux)

```sql
SELECT product, 'January' AS mois, jan AS montant FROM sales_pivot
UNION ALL
SELECT product, 'February' AS mois, feb AS montant FROM sales_pivot
UNION ALL
SELECT product, 'March' AS mois, mar AS montant FROM sales_pivot
ORDER BY product,
    CASE mois
        WHEN 'January' THEN 1
        WHEN 'February' THEN 2
        WHEN 'March' THEN 3
    END;
```

**Résultat (format ligne)** :
```
+----------+----------+---------+
| product  | mois     | montant |
+----------+----------+---------+
| Keyboard | January  |    0.00 |
| Keyboard | February |   75.00 |
| Keyboard | March    |   80.00 |
| Laptop   | January  | 2500.00 |
| Laptop   | February | 1250.00 |
| Laptop   | March    | 1400.00 |
| Mouse    | January  |   25.00 |
| Mouse    | February |   30.00 |
| Mouse    | March    |   28.00 |
+----------+----------+---------+
```

#### Méthode 2 : CROSS JOIN avec table de mois (Plus élégant)

```sql
-- Créer une table temporaire des mois
CREATE TEMPORARY TABLE months (
    mois_nom VARCHAR(20),
    mois_col VARCHAR(10)
);

INSERT INTO months VALUES
('January', 'jan'),
('February', 'feb'),
('March', 'mar');

-- UNPIVOT avec CROSS JOIN
SELECT
    sp.product,
    m.mois_nom AS mois,
    CASE m.mois_col
        WHEN 'jan' THEN sp.jan
        WHEN 'feb' THEN sp.feb
        WHEN 'mar' THEN sp.mar
    END AS montant
FROM sales_pivot sp
CROSS JOIN months m
ORDER BY sp.product,
    CASE m.mois_nom
        WHEN 'January' THEN 1
        WHEN 'February' THEN 2
        WHEN 'March' THEN 3
    END;
```

### Exemple 7 : UNPIVOT de metrics multiples

Transformer plusieurs métriques en format ligne pour analyse.

```sql
CREATE TABLE product_metrics (
    product VARCHAR(50),
    revenue DECIMAL(10,2),
    cost DECIMAL(10,2),
    profit DECIMAL(10,2)
);

INSERT INTO product_metrics VALUES
('Laptop', 10000, 7000, 3000),
('Mouse', 500, 300, 200),
('Keyboard', 1500, 900, 600);
```

#### UNPIVOT : Une ligne par métrique

```sql
SELECT product, 'Revenue' AS metric, revenue AS value FROM product_metrics
UNION ALL
SELECT product, 'Cost' AS metric, cost AS value FROM product_metrics
UNION ALL
SELECT product, 'Profit' AS metric, profit AS value FROM product_metrics
ORDER BY product,
    CASE metric
        WHEN 'Revenue' THEN 1
        WHEN 'Cost' THEN 2
        WHEN 'Profit' THEN 3
    END;
```

**Résultat** :
```
+----------+---------+----------+
| product  | metric  | value    |
+----------+---------+----------+
| Keyboard | Revenue |  1500.00 |
| Keyboard | Cost    |   900.00 |
| Keyboard | Profit  |   600.00 |
| Laptop   | Revenue | 10000.00 |
| Laptop   | Cost    |  7000.00 |
| Laptop   | Profit  |  3000.00 |
| Mouse    | Revenue |   500.00 |
| Mouse    | Cost    |   300.00 |
| Mouse    | Profit  |   200.00 |
+----------+---------+----------+
```

💡 **Utilité** : Ce format est idéal pour créer des graphiques multi-séries ou exporter vers des outils BI.

---

## PIVOT dynamique avec procédures stockées

Pour créer des pivots avec un nombre variable de colonnes, nous devons utiliser du SQL dynamique.

### Exemple 8 : Pivot dynamique générique

```sql
DELIMITER //

CREATE PROCEDURE pivot_sales_by_product(IN year_param INT)
BEGIN
    DECLARE sql_query TEXT;
    DECLARE pivot_columns TEXT;

    -- 1. Générer la liste des colonnes dynamiquement
    SELECT GROUP_CONCAT(
        DISTINCT CONCAT(
            'SUM(CASE WHEN product = ''', product,
            ''' THEN amount ELSE 0 END) AS `', product, '`'
        )
    ) INTO pivot_columns
    FROM sales
    WHERE YEAR(sale_date) = year_param;

    -- 2. Construire la requête complète
    SET sql_query = CONCAT(
        'SELECT ',
        '  MONTHNAME(sale_date) AS mois, ',
        pivot_columns, ', ',
        '  SUM(amount) AS total ',
        'FROM sales ',
        'WHERE YEAR(sale_date) = ', year_param, ' ',
        'GROUP BY MONTHNAME(sale_date), MONTH(sale_date) ',
        'ORDER BY MONTH(sale_date)'
    );

    -- 3. Préparer et exécuter
    SET @sql_query = sql_query;
    PREPARE stmt FROM @sql_query;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

-- Utilisation
CALL pivot_sales_by_product(2025);
```

**Résultat** : Génère automatiquement une colonne pour chaque produit trouvé dans les données !

💡 **Avantage** : Pas besoin de modifier la requête quand de nouveaux produits apparaissent.

⚠️ **Limitation** :
- Nécessite des privilèges pour exécuter du SQL dynamique
- Plus difficile à déboguer
- Performance légèrement inférieure aux requêtes statiques

---

## Cas d'usage réels en production

### 1. Dashboard de ventes e-commerce

```sql
-- Ventes par catégorie et par canal (Web, Mobile, Store)
SELECT
    category,

    SUM(CASE WHEN channel = 'Web' THEN amount ELSE 0 END) AS web_sales,
    SUM(CASE WHEN channel = 'Mobile' THEN amount ELSE 0 END) AS mobile_sales,
    SUM(CASE WHEN channel = 'Store' THEN amount ELSE 0 END) AS store_sales,

    SUM(amount) AS total_sales,

    -- Pourcentage par canal
    ROUND(100.0 * SUM(CASE WHEN channel = 'Web' THEN amount ELSE 0 END) / SUM(amount), 1) AS web_pct,
    ROUND(100.0 * SUM(CASE WHEN channel = 'Mobile' THEN amount ELSE 0 END) / SUM(amount), 1) AS mobile_pct,
    ROUND(100.0 * SUM(CASE WHEN channel = 'Store' THEN amount ELSE 0 END) / SUM(amount), 1) AS store_pct
FROM sales_detailed
GROUP BY category
ORDER BY total_sales DESC;
```

### 2. Rapport RH : Répartition des employés

```sql
-- Employés par département et par contrat (CDI, CDD, Stage)
SELECT
    department,

    COUNT(CASE WHEN contract_type = 'CDI' THEN 1 END) AS cdi,
    COUNT(CASE WHEN contract_type = 'CDD' THEN 1 END) AS cdd,
    COUNT(CASE WHEN contract_type = 'Stage' THEN 1 END) AS stage,
    COUNT(CASE WHEN contract_type = 'Freelance' THEN 1 END) AS freelance,

    COUNT(*) AS total,

    -- ETP (Équivalent Temps Plein)
    SUM(CASE WHEN contract_type = 'CDI' THEN 1.0
             WHEN contract_type = 'CDD' THEN 1.0
             WHEN contract_type = 'Stage' THEN 0.5
             WHEN contract_type = 'Freelance' THEN 0.8
             ELSE 0 END) AS etp_total
FROM employees_detailed
GROUP BY department
ORDER BY department;
```

### 3. Analyse financière : P&L par trimestre

```sql
-- Compte de résultat par trimestre
SELECT
    account_type,

    SUM(CASE WHEN QUARTER(date) = 1 THEN amount ELSE 0 END) AS Q1,
    SUM(CASE WHEN QUARTER(date) = 2 THEN amount ELSE 0 END) AS Q2,
    SUM(CASE WHEN QUARTER(date) = 3 THEN amount ELSE 0 END) AS Q3,
    SUM(CASE WHEN QUARTER(date) = 4 THEN amount ELSE 0 END) AS Q4,

    SUM(amount) AS total_year,

    -- Évolution Q4 vs Q1
    ROUND(100.0 * (
        SUM(CASE WHEN QUARTER(date) = 4 THEN amount ELSE 0 END) -
        SUM(CASE WHEN QUARTER(date) = 1 THEN amount ELSE 0 END)
    ) / NULLIF(SUM(CASE WHEN QUARTER(date) = 1 THEN amount ELSE 0 END), 0), 1) AS growth_Q1_Q4_pct
FROM financial_data
WHERE YEAR(date) = 2025
GROUP BY account_type
ORDER BY
    CASE account_type
        WHEN 'Revenue' THEN 1
        WHEN 'Cost' THEN 2
        WHEN 'Operating Expense' THEN 3
        WHEN 'Net Income' THEN 4
    END;
```

### 4. Analyse de satisfaction client par canal

```sql
-- Scores de satisfaction (1-5) par canal de support
SELECT
    support_category,

    -- Nombre de réponses par score
    COUNT(CASE WHEN rating = 5 THEN 1 END) AS score_5_excellent,
    COUNT(CASE WHEN rating = 4 THEN 1 END) AS score_4_good,
    COUNT(CASE WHEN rating = 3 THEN 1 END) AS score_3_ok,
    COUNT(CASE WHEN rating = 2 THEN 1 END) AS score_2_bad,
    COUNT(CASE WHEN rating = 1 THEN 1 END) AS score_1_very_bad,

    COUNT(*) AS total_responses,

    -- Score moyen
    ROUND(AVG(rating), 2) AS avg_rating,

    -- Net Promoter Score (NPS-like)
    ROUND(100.0 * (
        COUNT(CASE WHEN rating >= 4 THEN 1 END) -
        COUNT(CASE WHEN rating <= 2 THEN 1 END)
    ) / COUNT(*), 1) AS nps
FROM customer_feedback
GROUP BY support_category
ORDER BY avg_rating DESC;
```

---

## Performance et optimisations

### 💡 Bonnes pratiques

#### 1. **Index sur les colonnes utilisées**

```sql
-- ✅ Index sur colonnes de filtrage et groupement
CREATE INDEX idx_sales_date_product ON sales(sale_date, product);
CREATE INDEX idx_employees_dept_level ON employees(department, level);
```

#### 2. **Filtrer avant de pivoter**

```sql
-- ❌ Pivot puis filtrage (moins efficace)
SELECT ... FROM (
    SELECT mois, product, SUM(CASE ...) FROM sales GROUP BY ...
) WHERE mois = 'January';

-- ✅ Filtrage puis pivot (plus efficace)
SELECT mois, SUM(CASE WHEN product = 'Laptop' THEN amount ELSE 0 END) ...
FROM sales
WHERE MONTH(sale_date) = 1  -- ✅ Filtre dès le départ
GROUP BY mois;
```

#### 3. **Limiter le nombre de colonnes pivotées**

```sql
-- ⚠️ Trop de colonnes = requête lourde
-- Évitez de pivoter plus de 20-30 colonnes
-- Sinon, considérez une autre approche (JSON, export CSV)
```

#### 4. **Utiliser des vues matérialisées (workaround)**

MariaDB n'a pas de vues matérialisées natives, mais on peut simuler :

```sql
-- Créer une table "cache" du pivot
CREATE TABLE sales_pivot_cache AS
SELECT
    MONTHNAME(sale_date) AS mois,
    SUM(CASE WHEN product = 'Laptop' THEN amount ELSE 0 END) AS laptop,
    SUM(CASE WHEN product = 'Mouse' THEN amount ELSE 0 END) AS mouse
FROM sales
GROUP BY MONTHNAME(sale_date), MONTH(sale_date);

-- Ajouter un index
CREATE INDEX idx_mois ON sales_pivot_cache(mois);

-- Rafraîchir périodiquement (via cron ou event)
CREATE EVENT refresh_sales_pivot
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    TRUNCATE sales_pivot_cache;
    INSERT INTO sales_pivot_cache
    SELECT ... FROM sales GROUP BY ...;
END;
```

### 📊 Analyse EXPLAIN

```sql
EXPLAIN SELECT
    MONTHNAME(sale_date) AS mois,
    SUM(CASE WHEN product = 'Laptop' THEN amount ELSE 0 END) AS laptop
FROM sales
GROUP BY MONTHNAME(sale_date), MONTH(sale_date);
```

**Points à vérifier** :
- ✅ `type: index` ou `ref` (pas `ALL`)
- ✅ `Using index` (covering index)
- ✅ Pas de `Using temporary` si possible
- ✅ `rows` raisonnable (pas des millions)

---

## Alternatives et outils complémentaires

### 1. **Utiliser JSON pour pivots complexes**

```sql
-- Stocker le résultat pivoté en JSON
SELECT
    department,
    JSON_OBJECT(
        'Junior', COUNT(CASE WHEN level = 'Junior' THEN 1 END),
        'Mid', COUNT(CASE WHEN level = 'Mid' THEN 1 END),
        'Senior', COUNT(CASE WHEN level = 'Senior' THEN 1 END)
    ) AS employee_counts
FROM employees
GROUP BY department;
```

**Résultat** :
```
+------------+--------------------------------------------------+
| department | employee_counts                                  |
+------------+--------------------------------------------------+
| HR         | {"Mid": 0, "Junior": 1, "Senior": 1}             |
| IT         | {"Mid": 0, "Junior": 1, "Senior": 2}             |
| Sales      | {"Mid": 1, "Junior": 1, "Senior": 1}             |
+------------+--------------------------------------------------+
```

💡 **Avantage** : Plus flexible, pas de limite de colonnes.

### 2. **Exporter vers Excel/Google Sheets**

Parfois, il est plus simple de faire le pivot côté application :

```sql
-- Requête simple en format ligne
SELECT
    sale_date,
    product,
    amount
FROM sales
ORDER BY sale_date, product;

-- Puis pivoter dans Excel avec un tableau croisé dynamique
```

### 3. **Utiliser des outils BI**

Tableau, Power BI, Metabase, Superset gèrent nativement les pivots :

```sql
-- Fournir les données brutes
SELECT
    DATE_FORMAT(sale_date, '%Y-%m') AS mois,
    product,
    SUM(amount) AS total_ventes
FROM sales
GROUP BY DATE_FORMAT(sale_date, '%Y-%m'), product;

-- L'outil BI fera le pivot visuellement
```

---

## ⚠️ Pièges courants et solutions

### Piège 1 : Oublier le ELSE 0 dans SUM

```sql
-- ❌ ERREUR : NULL au lieu de 0
SELECT
    SUM(CASE WHEN product = 'Laptop' THEN amount END) AS laptop
FROM sales;
-- Si aucun Laptop : résultat = NULL (pas 0)

-- ✅ CORRECT : ELSE 0
SELECT
    SUM(CASE WHEN product = 'Laptop' THEN amount ELSE 0 END) AS laptop
FROM sales;
-- Si aucun Laptop : résultat = 0
```

### Piège 2 : Colonnes hardcodées obsolètes

```sql
-- ⚠️ Si un nouveau produit "Tablet" apparaît, il ne sera pas inclus
SELECT
    SUM(CASE WHEN product = 'Laptop' THEN amount ELSE 0 END) AS laptop,
    SUM(CASE WHEN product = 'Mouse' THEN amount ELSE 0 END) AS mouse
    -- Manque Tablet !
FROM sales;

-- ✅ Solution : Utiliser une procédure stockée dynamique
-- OU accepter que le pivot soit statique et le mettre à jour manuellement
```

### Piège 3 : Ordre de tri incorrect

```sql
-- ❌ Tri alphabétique des mois (incorrect)
SELECT MONTHNAME(sale_date) AS mois, ...
FROM sales
GROUP BY MONTHNAME(sale_date)
ORDER BY mois;  -- ⚠️ April, August, December, February...

-- ✅ Tri numérique correct
SELECT MONTHNAME(sale_date) AS mois, ...
FROM sales
GROUP BY MONTHNAME(sale_date), MONTH(sale_date)
ORDER BY MONTH(sale_date);  -- ✅ January, February, March...
```

### Piège 4 : Agrégations multiples incorrectes

```sql
-- ❌ Double comptage
SELECT
    department,
    COUNT(*) AS total_employees,
    COUNT(CASE WHEN level = 'Senior' THEN 1 END) AS seniors
FROM employees
GROUP BY department;
-- OK

-- ❌ ERREUR : Utiliser COUNT(*) avec LEFT JOIN
SELECT
    d.department,
    COUNT(*) AS total  -- ⚠️ Comptera aussi les NULL du JOIN
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.department;

-- ✅ CORRECT
SELECT
    d.department,
    COUNT(e.id) AS total  -- ✅ Ignore les NULL
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.department;
```

---

## ✅ Points clés à retenir

- 🔄 **PIVOT** = Lignes → Colonnes avec `CASE WHEN` + `SUM/COUNT/AVG`
- 🔄 **UNPIVOT** = Colonnes → Lignes avec `UNION ALL` ou `CROSS JOIN`
- 📊 **Agrégations conditionnelles** : Technique fondamentale pour les pivots
- 🏗️ **MariaDB n'a pas PIVOT natif** : On utilise des techniques alternatives
- 💾 **SQL dynamique** : Nécessaire pour des pivots avec colonnes variables
- 📈 **Cas d'usage** : Rapports, dashboards, analyses comparatives
- ⚡ **Performance** : Indexer colonnes de filtrage, filtrer avant de pivoter
- 🎯 **Toujours utiliser ELSE 0** avec SUM pour éviter les NULL
- 📅 **Tri temporel** : GROUP BY et ORDER BY sur MONTH() numérique, pas alphabétique

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CASE Statement](https://mariadb.com/kb/en/case-statement/)
- [📖 GROUP BY](https://mariadb.com/kb/en/select/#group-by)
- [📖 Aggregate Functions](https://mariadb.com/kb/en/aggregate-functions/)
- [📖 Dynamic SQL](https://mariadb.com/kb/en/dynamic-sql/)

### Articles et tutoriels
- [SQL Pivot Tutorial](https://www.sqlshack.com/sql-pivot-and-unpivot/) - Concepts généraux
- [Conditional Aggregation](https://modern-sql.com/use-case/pivot) - Approche moderne

### Outils
- [Excel Pivot Tables](https://support.microsoft.com/en-us/office/create-a-pivottable-to-analyze-worksheet-data-a9a84538-bfe9-40a9-a8e9-f99134456576) - Référence visuelle
- [Tableau](https://www.tableau.com/) - Outil BI avec pivots natifs

---

## ➡️ Section suivante

**[4.4 Expressions de table communes (CTE)](./04-expressions-table-communes.md)** : Apprenez à structurer vos requêtes complexes avec des CTEs, rendant votre code SQL plus lisible, maintenable et réutilisable.

---


⏭️ [Expressions de table communes (CTE)](/04-concepts-avances-sql/04-expressions-table-communes.md)

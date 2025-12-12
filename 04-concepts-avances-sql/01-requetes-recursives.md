🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Requêtes récursives (WITH RECURSIVE)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Maîtrise des CTE simples, sous-requêtes, UNION

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le concept de **récursivité en SQL** et ses cas d'usage
- Maîtriser la syntaxe `WITH RECURSIVE` de MariaDB
- Parcourir des **structures hiérarchiques** (arbres, graphes)
- Générer des **séries de données** (nombres, dates)
- Éviter les **boucles infinies** et optimiser les performances
- Appliquer les requêtes récursives à des problèmes métier réels

---

## Introduction

### Qu'est-ce qu'une requête récursive ?

Une **requête récursive** est une requête qui se référence elle-même pour traiter des données de manière itérative. En SQL, cela permet de résoudre des problèmes qui nécessiteraient plusieurs requêtes ou des boucles en code applicatif.

### Pourquoi les requêtes récursives ?

**Le problème** : Comment récupérer tous les employés sous un manager, peu importe le nombre de niveaux hiérarchiques ?

```sql
-- ❌ Approche naïve : nombre de requêtes = profondeur de la hiérarchie
SELECT * FROM employees WHERE manager_id = 1;
SELECT * FROM employees WHERE manager_id IN (2, 3, 4);
SELECT * FROM employees WHERE manager_id IN (5, 6, 7, 8);
-- ... et ainsi de suite
```

**La solution** : Une requête récursive qui traverse automatiquement toute la hiérarchie.

```sql
-- ✅ Approche récursive : une seule requête pour toute la hiérarchie
WITH RECURSIVE hierarchy AS (
    -- Point de départ
    SELECT id, name, manager_id, 1 as level
    FROM employees WHERE id = 1

    UNION ALL

    -- Récursion : chercher les employés du niveau suivant
    SELECT e.id, e.name, e.manager_id, h.level + 1
    FROM employees e
    INNER JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy;
```

### Cas d'usage courants

Les requêtes récursives sont particulièrement utiles pour :

- 🌳 **Hiérarchies** : Organigrammes, catégories de produits, structure de fichiers
- 📊 **Graphes** : Réseaux sociaux, relations, dépendances
- 🔢 **Génération de séries** : Dates, nombres, calendriers
- 🗺️ **Chemins** : Itinéraires, routes, parcours
- 🔄 **Cycles de vie** : Workflow, états, processus

---

## Syntaxe et fonctionnement

### Structure de base

```sql
WITH RECURSIVE cte_name AS (
    -- 1️⃣ ANCRE (Anchor member) : requête non-récursive
    SELECT ...
    FROM table
    WHERE condition_initiale

    UNION [ALL]

    -- 2️⃣ RÉCURSION (Recursive member) : requête qui référence la CTE
    SELECT ...
    FROM table
    INNER JOIN cte_name ON ...
)
-- 3️⃣ REQUÊTE FINALE : utilisation de la CTE
SELECT * FROM cte_name;
```

### Les trois parties essentielles

#### 1️⃣ **L'ancre (Anchor member)**
- Requête **non-récursive** qui définit le point de départ
- Exécutée **une seule fois** au début
- Produit le **résultat initial** (R₀)

#### 2️⃣ **La partie récursive (Recursive member)**
- Requête qui **référence la CTE elle-même**
- Exécutée **de manière itérative** jusqu'à ce qu'elle ne retourne plus de lignes
- À chaque itération, travaille sur le résultat de l'itération précédente (Rₙ₋₁ → Rₙ)

#### 3️⃣ **La requête finale**
- Utilise la CTE complète (union de toutes les itérations)
- Peut inclure des filtres, tris, agrégations

### Comment ça fonctionne ? (Algorithme)

```
Itération 0 : Exécuter l'ancre → Résultat R₀
Itération 1 : Exécuter la partie récursive avec R₀ → Résultat R₁
Itération 2 : Exécuter la partie récursive avec R₁ → Résultat R₂
...
Itération n : Exécuter la partie récursive avec Rₙ₋₁ → Résultat Rₙ
              Si Rₙ est vide → STOP

Résultat final = R₀ ∪ R₁ ∪ R₂ ∪ ... ∪ Rₙ
```

💡 **Avec `UNION ALL`** : Conserve les doublons entre itérations
💡 **Avec `UNION`** : Élimine les doublons (plus coûteux, mais évite les boucles infinies dans les graphes cycliques)

---

## Exemple 1 : Génération de séries numériques

Le cas le plus simple pour comprendre le mécanisme.

### Générer les nombres de 1 à 10

```sql
WITH RECURSIVE numbers AS (
    -- Ancre : on commence à 1
    SELECT 1 AS n

    UNION ALL

    -- Récursion : on ajoute 1 à chaque itération
    SELECT n + 1
    FROM numbers
    WHERE n < 10  -- ⚠️ Condition d'arrêt INDISPENSABLE
)
SELECT * FROM numbers;
```

**Résultat** :
```
+----+
| n  |
+----+
|  1 |
|  2 |
|  3 |
|  4 |
|  5 |
|  6 |
|  7 |
|  8 |
|  9 |
| 10 |
+----+
```

**Déroulement** :
```
Itération 0 : SELECT 1 → {1}
Itération 1 : SELECT 1+1 WHERE 1<10 → {2}
Itération 2 : SELECT 2+1 WHERE 2<10 → {3}
...
Itération 9 : SELECT 9+1 WHERE 9<10 → {10}
Itération 10 : SELECT 10+1 WHERE 10<10 → ∅ (vide, arrêt)
```

### Générer une série de dates

Très utile pour créer des calendriers ou remplir des gaps temporels.

```sql
WITH RECURSIVE date_series AS (
    -- Ancre : date de début
    SELECT DATE('2025-01-01') AS date

    UNION ALL

    -- Récursion : ajouter un jour à chaque itération
    SELECT DATE_ADD(date, INTERVAL 1 DAY)
    FROM date_series
    WHERE date < '2025-01-31'  -- Jusqu'au 31 janvier
)
SELECT
    date,
    DAYNAME(date) AS jour_semaine,
    CASE
        WHEN DAYOFWEEK(date) IN (1, 7) THEN '🔴 Week-end'
        ELSE '✅ Semaine'
    END AS type_jour
FROM date_series;
```

**Résultat** :
```
+------------+---------------+--------------+
| date       | jour_semaine  | type_jour    |
+------------+---------------+--------------+
| 2025-01-01 | Wednesday     | ✅ Semaine   |
| 2025-01-02 | Thursday      | ✅ Semaine   |
| 2025-01-03 | Friday        | ✅ Semaine   |
| 2025-01-04 | Saturday      | 🔴 Week-end  |
| 2025-01-05 | Sunday        | 🔴 Week-end  |
...
| 2025-01-31 | Friday        | ✅ Semaine   |
+------------+---------------+--------------+
```

💡 **Cas d'usage** : Générer un rapport avec tous les jours du mois, même ceux sans données.

---

## Exemple 2 : Hiérarchie d'employés (Organigramme)

Le cas d'usage classique et très fréquent en entreprise.

### Structure de données

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT,
    position VARCHAR(100),
    salary DECIMAL(10,2),
    FOREIGN KEY (manager_id) REFERENCES employees(id)
);

INSERT INTO employees VALUES
(1, 'Alice Martin', NULL, 'CEO', 150000),
(2, 'Bob Dupont', 1, 'CTO', 120000),
(3, 'Claire Bernard', 1, 'CFO', 120000),
(4, 'David Petit', 2, 'Dev Manager', 90000),
(5, 'Emma Dubois', 2, 'DevOps Manager', 90000),
(6, 'Frank Moreau', 4, 'Senior Dev', 75000),
(7, 'Gina Laurent', 4, 'Senior Dev', 75000),
(8, 'Hugo Simon', 4, 'Junior Dev', 50000),
(9, 'Iris Roux', 5, 'DevOps Engineer', 70000),
(10, 'Jack Blanc', 3, 'Accountant', 60000);
```

**Hiérarchie visuelle** :
```
Alice Martin (CEO)
├── Bob Dupont (CTO)
│   ├── David Petit (Dev Manager)
│   │   ├── Frank Moreau (Senior Dev)
│   │   ├── Gina Laurent (Senior Dev)
│   │   └── Hugo Simon (Junior Dev)
│   └── Emma Dubois (DevOps Manager)
│       └── Iris Roux (DevOps Engineer)
└── Claire Bernard (CFO)
    └── Jack Blanc (Accountant)
```

### Requête : Tous les subordonnés d'un manager

Récupérer tous les employés sous Bob Dupont (CTO, id=2) :

```sql
WITH RECURSIVE subordinates AS (
    -- Ancre : le manager lui-même
    SELECT
        id,
        name,
        position,
        manager_id,
        salary,
        0 AS level,  -- Niveau hiérarchique
        name AS path  -- Chemin hiérarchique
    FROM employees
    WHERE id = 2  -- Bob Dupont

    UNION ALL

    -- Récursion : employés directs et indirects
    SELECT
        e.id,
        e.name,
        e.position,
        e.manager_id,
        e.salary,
        s.level + 1,
        CONCAT(s.path, ' → ', e.name)  -- Construire le chemin
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.id
)
SELECT
    REPEAT('  ', level) AS indent,  -- Indentation visuelle
    name,
    position,
    salary,
    level,
    path
FROM subordinates
ORDER BY path;
```

**Résultat** :
```
+--------+------------------+------------------+----------+-------+------------------------------------------+
| indent | name             | position         | salary   | level | path                                     |
+--------+------------------+------------------+----------+-------+------------------------------------------+
|        | Bob Dupont       | CTO              | 120000   |     0 | Bob Dupont                               |
|        | David Petit      | Dev Manager      |  90000   |     1 | Bob Dupont → David Petit                 |
|        | Frank Moreau     | Senior Dev       |  75000   |     2 | Bob Dupont → David Petit → Frank Moreau  |
|        | Gina Laurent     | Senior Dev       |  75000   |     2 | Bob Dupont → David Petit → Gina Laurent  |
|        | Hugo Simon       | Junior Dev       |  50000   |     2 | Bob Dupont → David Petit → Hugo Simon    |
|        | Emma Dubois      | DevOps Manager   |  90000   |     1 | Bob Dupont → Emma Dubois                 |
|        | Iris Roux        | DevOps Engineer  |  70000   |     2 | Bob Dupont → Emma Dubois → Iris Roux     |
+--------+------------------+------------------+----------+-------+------------------------------------------+
```

### Agrégations sur la hiérarchie

Calculer la masse salariale de chaque manager (incluant ses subordonnés) :

```sql
WITH RECURSIVE subordinates AS (
    SELECT id, name, position, manager_id, salary, 0 AS level
    FROM employees WHERE id = 2

    UNION ALL

    SELECT e.id, e.name, e.position, e.manager_id, e.salary, s.level + 1
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.id
)
SELECT
    name,
    position,
    salary AS salaire_personnel,
    (SELECT SUM(salary) FROM subordinates s2 WHERE s2.id = s.id OR s2.manager_id = s.id) AS masse_salariale_equipe,
    (SELECT COUNT(*) FROM subordinates s2 WHERE s2.manager_id = s.id) AS nb_subordonnes_directs
FROM subordinates s
WHERE level <= 1  -- Seulement les managers de niveau 0 et 1
ORDER BY level, name;
```

**Résultat** :
```
+---------------+------------------+---------------------+-------------------------+---------------------------+
| name          | position         | salaire_personnel   | masse_salariale_equipe  | nb_subordonnes_directs    |
+---------------+------------------+---------------------+-------------------------+---------------------------+
| Bob Dupont    | CTO              | 120000.00           | 570000.00               |                         2 |
| David Petit   | Dev Manager      |  90000.00           | 290000.00               |                         3 |
| Emma Dubois   | DevOps Manager   |  90000.00           | 160000.00               |                         1 |
+---------------+------------------+---------------------+-------------------------+---------------------------+
```

💡 **Analyse** : Bob gère 570k€ de masse salariale totale (lui + ses 5 subordonnés directs et indirects).

---

## Exemple 3 : Arbre de catégories de produits

Cas typique en e-commerce : catégories et sous-catégories imbriquées.

### Structure de données

```sql
CREATE TABLE categories (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    parent_id INT,
    FOREIGN KEY (parent_id) REFERENCES categories(id)
);

INSERT INTO categories VALUES
(1, 'Électronique', NULL),
(2, 'Informatique', 1),
(3, 'Audio/Vidéo', 1),
(4, 'Ordinateurs', 2),
(5, 'Périphériques', 2),
(6, 'Portables', 4),
(7, 'Desktop', 4),
(8, 'Claviers', 5),
(9, 'Souris', 5),
(10, 'Casques', 3),
(11, 'Enceintes', 3);
```

**Arbre de catégories** :
```
Électronique (1)
├── Informatique (2)
│   ├── Ordinateurs (4)
│   │   ├── Portables (6)
│   │   └── Desktop (7)
│   └── Périphériques (5)
│       ├── Claviers (8)
│       └── Souris (9)
└── Audio/Vidéo (3)
    ├── Casques (10)
    └── Enceintes (11)
```

### Afficher toutes les sous-catégories

```sql
WITH RECURSIVE category_tree AS (
    -- Ancre : catégorie racine
    SELECT
        id,
        name,
        parent_id,
        0 AS depth,
        CAST(name AS CHAR(200)) AS path
    FROM categories
    WHERE id = 1  -- Électronique

    UNION ALL

    -- Récursion : sous-catégories
    SELECT
        c.id,
        c.name,
        c.parent_id,
        ct.depth + 1,
        CONCAT(ct.path, ' > ', c.name)
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT
    CONCAT(REPEAT('  ', depth), '└─ ') AS tree,
    name,
    depth,
    path
FROM category_tree
ORDER BY path;
```

**Résultat** :
```
+---------+------------------+-------+------------------------------------------------------+
| tree    | name             | depth | path                                                 |
+---------+------------------+-------+------------------------------------------------------+
| └─      | Électronique   |     0 | Électronique                                           |
|   └─    | Audio/Vidéo    |     1 | Électronique > Audio/Vidéo                             |
|     └─  | Casques        |     2 | Électronique > Audio/Vidéo > Casques                   |
|     └─  | Enceintes      |     2 | Électronique > Audio/Vidéo > Enceintes                 |
|   └─    | Informatique   |     1 | Électronique > Informatique                            |
|     └─  | Ordinateurs    |     2 | Électronique > Informatique > Ordinateurs              |
|       └─| Desktop        |     3 | Électronique > Informatique > Ordinateurs > Desktop    |
|       └─| Portables      |     3 | Électronique > Informatique > Ordinateurs > Portables  |
|     └─  | Périphériques  |     2 | Électronique > Informatique > Périphériques            |
|       └─| Claviers       |     3 | Électronique > Informatique > Périphériques > Claviers |
|       └─| Souris         |     3 | Électronique > Informatique > Périphériques > Souris   |
+-------+------------------+-------+--------------------------------------------------------+
```

### Remonter la hiérarchie (du bas vers le haut)

Trouver le chemin complet d'une catégorie feuille vers la racine :

```sql
WITH RECURSIVE category_path AS (
    -- Ancre : catégorie de départ (Claviers)
    SELECT
        id,
        name,
        parent_id,
        1 AS level,
        name AS full_path
    FROM categories
    WHERE id = 8  -- Claviers

    UNION ALL

    -- Récursion : remonter vers le parent
    SELECT
        c.id,
        c.name,
        c.parent_id,
        cp.level + 1,
        CONCAT(c.name, ' > ', cp.full_path)
    FROM categories c
    INNER JOIN category_path cp ON c.id = cp.parent_id
)
SELECT
    level,
    name,
    full_path
FROM category_path
ORDER BY level DESC;
```

**Résultat** :
```
+-------+----------------+--------------------------------------------------------+
| level | name           | full_path                                              |
+-------+----------------+--------------------------------------------------------+
|     3 | Électronique   | Électronique > Informatique > Périphériques > Claviers |
|     2 | Informatique   | Informatique > Périphériques > Claviers                |
|     1 | Périphériques  | Périphériques > Claviers                               |
|     0 | Claviers       | Claviers                                               |
+-------+----------------+--------------------------------------------------------+
```

---

## Exemple 4 : Détection de cycles dans un graphe

Les requêtes récursives peuvent aussi détecter les cycles (relations circulaires).

### Structure avec cycle potentiel

```sql
CREATE TABLE graph_nodes (
    from_id INT,
    to_id INT,
    PRIMARY KEY (from_id, to_id)
);

INSERT INTO graph_nodes VALUES
(1, 2), (2, 3), (3, 4), (4, 2);  -- Cycle : 2 → 3 → 4 → 2
```

### Détection avec UNION (évite automatiquement les cycles)

```sql
WITH RECURSIVE paths AS (
    -- Ancre : point de départ
    SELECT
        from_id,
        to_id,
        CAST(from_id AS CHAR(200)) AS path,
        1 AS depth
    FROM graph_nodes
    WHERE from_id = 1

    UNION  -- ⚠️ UNION (pas UNION ALL) élimine les doublons

    -- Récursion
    SELECT
        g.from_id,
        g.to_id,
        CONCAT(p.path, ' → ', g.from_id),
        p.depth + 1
    FROM graph_nodes g
    INNER JOIN paths p ON g.from_id = p.to_id
    WHERE p.depth < 10  -- Limite de sécurité
)
SELECT * FROM paths;
```

### Détection explicite avec suivi du chemin

```sql
WITH RECURSIVE paths AS (
    SELECT
        from_id,
        to_id,
        CAST(CONCAT(',', from_id, ',') AS CHAR(1000)) AS visited,
        CAST(from_id AS CHAR(200)) AS path,
        0 AS has_cycle
    FROM graph_nodes
    WHERE from_id = 1

    UNION ALL

    SELECT
        g.from_id,
        g.to_id,
        CONCAT(p.visited, g.from_id, ','),
        CONCAT(p.path, ' → ', g.from_id),
        -- Détecter si on a déjà visité ce nœud
        CASE
            WHEN p.visited LIKE CONCAT('%,', g.from_id, ',%') THEN 1
            ELSE 0
        END AS has_cycle
    FROM graph_nodes g
    INNER JOIN paths p ON g.from_id = p.to_id
    WHERE p.has_cycle = 0  -- Arrêter dès qu'un cycle est détecté
)
SELECT
    path,
    CASE
        WHEN has_cycle = 1 THEN '🔴 CYCLE DÉTECTÉ'
        ELSE '✅ Pas de cycle'
    END AS status
FROM paths;
```

---

## Cas d'usage avancés

### 1. Bill of Materials (BOM) - Nomenclature produit

Calculer tous les composants nécessaires pour fabriquer un produit.

```sql
CREATE TABLE bom (
    product_id INT,
    component_id INT,
    quantity INT,
    PRIMARY KEY (product_id, component_id)
);

-- Produit 1 = 2x Composant 2 + 3x Composant 3
-- Composant 2 = 4x Composant 4 + 1x Composant 5
INSERT INTO bom VALUES
(1, 2, 2),
(1, 3, 3),
(2, 4, 4),
(2, 5, 1);

WITH RECURSIVE bom_explosion AS (
    -- Ancre : produit final
    SELECT
        product_id,
        component_id,
        quantity,
        1 AS level,
        quantity AS total_qty
    FROM bom
    WHERE product_id = 1

    UNION ALL

    -- Récursion : composants des composants
    SELECT
        b.product_id,
        b.component_id,
        b.quantity,
        be.level + 1,
        be.total_qty * b.quantity  -- Quantité cumulée
    FROM bom b
    INNER JOIN bom_explosion be ON b.product_id = be.component_id
)
SELECT
    REPEAT('  ', level - 1) AS indent,
    component_id,
    quantity AS qty_per_parent,
    total_qty AS qty_total_needed,
    level
FROM bom_explosion
ORDER BY level, component_id;
```

### 2. Génération de rapport hierarchique

Créer un rapport JSON d'une structure hiérarchique.

```sql
WITH RECURSIVE org_tree AS (
    SELECT
        id,
        name,
        manager_id,
        0 AS level,
        JSON_ARRAY(JSON_OBJECT('id', id, 'name', name)) AS subordinates_json
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.id,
        e.name,
        e.manager_id,
        ot.level + 1,
        JSON_ARRAY_APPEND(
            ot.subordinates_json,
            '$',
            JSON_OBJECT('id', e.id, 'name', e.name, 'level', ot.level + 1)
        )
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT
    name AS manager,
    subordinates_json
FROM org_tree
WHERE level = 0;
```

### 3. Calcul de distances dans un graphe

Trouver le chemin le plus court (nombre de sauts) entre deux nœuds.

```sql
CREATE TABLE connections (
    node_a INT,
    node_b INT
);

INSERT INTO connections VALUES
(1, 2), (2, 3), (3, 5), (1, 4), (4, 5), (5, 6);

WITH RECURSIVE shortest_path AS (
    -- Ancre : nœud de départ
    SELECT
        node_a AS current_node,
        node_b AS next_node,
        1 AS distance,
        CAST(CONCAT(node_a, '→', node_b) AS CHAR(200)) AS path
    FROM connections
    WHERE node_a = 1

    UNION

    -- Récursion : explorer les voisins
    SELECT
        c.node_a,
        c.node_b,
        sp.distance + 1,
        CONCAT(sp.path, '→', c.node_b)
    FROM connections c
    INNER JOIN shortest_path sp ON c.node_a = sp.next_node
    WHERE sp.distance < 5  -- Limite de profondeur
)
SELECT
    path,
    distance
FROM shortest_path
WHERE next_node = 6  -- Nœud de destination
ORDER BY distance
LIMIT 1;
```

**Résultat** : Le chemin le plus court de 1 à 6.

---

## Performance et optimisations

### ⚡ Bonnes pratiques de performance

#### 1. **Toujours avoir une condition d'arrêt**

```sql
-- ❌ DANGER : Boucle infinie potentielle
WITH RECURSIVE infinite AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM infinite  -- ⚠️ Pas de WHERE
)
SELECT * FROM infinite;  -- Ne finira jamais !
```

```sql
-- ✅ CORRECT : Condition d'arrêt
WITH RECURSIVE safe AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM safe WHERE n < 1000  -- ✅ Limite claire
)
SELECT * FROM safe;
```

#### 2. **Limiter la profondeur de récursion**

MariaDB a une limite de profondeur configurable :

```sql
-- Voir la limite actuelle
SHOW VARIABLES LIKE 'max_sp_recursion_depth';
-- Valeur par défaut : 255

-- Modifier pour la session (si nécessaire)
SET SESSION max_sp_recursion_depth = 10;
```

💡 **Conseil** : Ajoutez toujours un compteur de niveau dans votre requête et limitez-le.

#### 3. **Utiliser des index appropriés**

Les colonnes utilisées dans les jointures récursives doivent être indexées.

```sql
-- ✅ Index sur les colonnes de jointure
CREATE INDEX idx_manager ON employees(manager_id);
CREATE INDEX idx_parent ON categories(parent_id);
```

#### 4. **UNION vs UNION ALL**

```sql
-- UNION ALL : Plus rapide (pas de vérification de doublons)
-- Utilisez quand vous êtes sûr qu'il n'y a pas de cycles
WITH RECURSIVE fast AS (
    SELECT id FROM start
    UNION ALL  -- ⚡ Plus rapide
    SELECT id FROM next JOIN fast ON ...
)

-- UNION : Plus sûr (élimine les doublons)
-- Utilisez pour les graphes avec cycles potentiels
WITH RECURSIVE safe AS (
    SELECT id FROM start
    UNION  -- 🛡️ Plus sûr mais plus lent
    SELECT id FROM next JOIN safe ON ...
)
```

#### 5. **Limiter le nombre de colonnes dans la récursion**

Ne sélectionnez que ce dont vous avez besoin dans la partie récursive.

```sql
-- ❌ Sélection inutilement large
WITH RECURSIVE all_cols AS (
    SELECT * FROM big_table  -- Toutes les colonnes
    UNION ALL
    SELECT * FROM big_table JOIN all_cols ...
)

-- ✅ Sélection ciblée
WITH RECURSIVE needed_cols AS (
    SELECT id, parent_id, name FROM big_table  -- Seulement l'essentiel
    UNION ALL
    SELECT t.id, t.parent_id, t.name FROM big_table t JOIN needed_cols ...
)
```

### 📊 Analyse de performance

```sql
-- Utiliser EXPLAIN pour analyser
EXPLAIN WITH RECURSIVE hierarchy AS (
    SELECT id, manager_id FROM employees WHERE id = 1
    UNION ALL
    SELECT e.id, e.manager_id FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy;
```

---

## ⚠️ Pièges courants et solutions

### Piège 1 : Boucle infinie

**Symptôme** : La requête ne se termine jamais.

**Cause** : Pas de condition d'arrêt ou cycle dans les données.

**Solution** :
```sql
-- ✅ Toujours ajouter une limite de profondeur
WITH RECURSIVE safe AS (
    SELECT id, parent_id, 0 AS depth FROM table WHERE id = 1
    UNION ALL
    SELECT t.id, t.parent_id, s.depth + 1
    FROM table t
    JOIN safe s ON t.parent_id = s.id
    WHERE s.depth < 100  -- ✅ Limite de sécurité
)
SELECT * FROM safe;
```

### Piège 2 : Explosion mémoire

**Symptôme** : `ERROR 1213: Deadlock found` ou mémoire insuffisante.

**Cause** : Résultat intermédiaire trop volumineux.

**Solution** :
```sql
-- ✅ Ajouter des filtres dès l'ancre
WITH RECURSIVE filtered AS (
    SELECT id, parent_id, name
    FROM categories
    WHERE active = 1  -- ✅ Filtrer tôt
    AND id = 1

    UNION ALL

    SELECT c.id, c.parent_id, c.name
    FROM categories c
    JOIN filtered f ON c.parent_id = f.id
    WHERE c.active = 1  -- ✅ Filtrer à chaque niveau
)
SELECT * FROM filtered;
```

### Piège 3 : Types de données incompatibles

**Symptôme** : `ERROR 1292: Truncated incorrect ... value`.

**Cause** : Types de données différents entre l'ancre et la récursion.

**Solution** :
```sql
-- ❌ Types incompatibles
WITH RECURSIVE mixed AS (
    SELECT id, 'Start' AS path FROM table  -- VARCHAR court
    UNION ALL
    SELECT t.id, CONCAT(m.path, ' → ', t.name) FROM ...  -- ⚠️ Peut dépasser
)

-- ✅ CAST explicite
WITH RECURSIVE typed AS (
    SELECT id, CAST('Start' AS CHAR(500)) AS path FROM table  -- ✅ Taille définie
    UNION ALL
    SELECT t.id, CAST(CONCAT(m.path, ' → ', t.name) AS CHAR(500)) FROM ...
)
```

### Piège 4 : Oublier les index

**Symptôme** : Requête extrêmement lente sur des tables volumineuses.

**Solution** :
```sql
-- ✅ Créer des index sur les colonnes de jointure
CREATE INDEX idx_hierarchy ON employees(manager_id);
CREATE INDEX idx_tree ON categories(parent_id);

-- ✅ Vérifier avec EXPLAIN
EXPLAIN ...
```

---

## 💡 Conseils et bonnes pratiques

### 1. Commencer simple

```sql
-- Étape 1 : Écrire l'ancre seule
SELECT id, manager_id FROM employees WHERE id = 1;

-- Étape 2 : Ajouter une itération manuelle
SELECT e.id, e.manager_id
FROM employees e
WHERE e.manager_id = 1;

-- Étape 3 : Transformer en récursif
WITH RECURSIVE ...
```

### 2. Nommer explicitement les colonnes

```sql
-- ❌ Difficile à maintenir
WITH RECURSIVE cte AS (
    SELECT * FROM table1
    UNION ALL
    SELECT * FROM table1 JOIN cte ...
)

-- ✅ Clair et explicite
WITH RECURSIVE cte AS (
    SELECT id, parent_id, name, level FROM table1
    UNION ALL
    SELECT t.id, t.parent_id, t.name, c.level + 1 FROM table1 t JOIN cte c ...
)
```

### 3. Ajouter des métadonnées utiles

```sql
WITH RECURSIVE enhanced AS (
    SELECT
        id,
        name,
        parent_id,
        0 AS level,                          -- Profondeur
        name AS path,                        -- Chemin complet
        1 AS is_leaf,                        -- Feuille ou pas
        CAST(id AS CHAR(200)) AS ancestors   -- Liste des ancêtres
    FROM tree
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        t.id,
        t.name,
        t.parent_id,
        e.level + 1,
        CONCAT(e.path, ' > ', t.name),
        (SELECT COUNT(*) FROM tree WHERE parent_id = t.id) = 0,
        CONCAT(e.ancestors, ',', t.id)
    FROM tree t
    JOIN enhanced e ON t.parent_id = e.id
)
SELECT * FROM enhanced;
```

### 4. Documenter la requête

```sql
WITH RECURSIVE employee_hierarchy AS (
    /*
     * Récupère tous les employés sous un manager donné
     * @param manager_id : ID du manager racine (ici : 1)
     * @return : Liste complète avec niveau hiérarchique et chemin
     * @note : Limité à 10 niveaux pour éviter les boucles
     */

    -- Ancre : le manager racine
    SELECT id, name, manager_id, 0 AS level
    FROM employees
    WHERE id = 1

    UNION ALL

    -- Récursion : subordonnés directs à chaque niveau
    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
    WHERE eh.level < 10  -- Protection contre boucles infinies
)
SELECT * FROM employee_hierarchy
ORDER BY level, name;
```

---

## ✅ Points clés à retenir

- 🔄 **Structure** : Ancre (point de départ) + Récursion (itération) = Résultat complet
- 🛡️ **Sécurité** : Toujours définir une condition d'arrêt explicite (WHERE niveau < N)
- 🎯 **Cas d'usage** : Hiérarchies, graphes, séries de données, nomenclatures (BOM)
- ⚡ **Performance** : Indexer les colonnes de jointure, limiter les colonnes sélectionnées
- 🔀 **UNION vs UNION ALL** : UNION élimine doublons (cycles), UNION ALL plus rapide
- 📊 **Métadonnées** : Ajouter level, path, depth pour faciliter l'exploitation
- ⚠️ **Limite** : `max_sp_recursion_depth` (défaut 255, configurable par session)
- 🧪 **Tests** : Toujours tester sur un petit dataset avant production

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Recursive Common Table Expressions](https://mariadb.com/kb/en/recursive-common-table-expressions-overview/)
- [📖 WITH Clause](https://mariadb.com/kb/en/with/)
- [📖 max_sp_recursion_depth](https://mariadb.com/kb/en/server-system-variables/#max_sp_recursion_depth)

### Standards SQL
- [SQL:1999](https://en.wikipedia.org/wiki/SQL:1999) - Introduction des CTEs récursives dans le standard

### Articles et tutoriels
- [Modern SQL: WITH Clause](https://modern-sql.com/feature/with) - Excellente explication progressive
- [Hierarchical Data in SQL](https://www.slideshare.net/billkarwin/models-for-hierarchical-data) - Modèles de données hiérarchiques

### Outils
- [SQLFiddle](http://sqlfiddle.com/) - Tester vos requêtes récursives en ligne
- [DB Fiddle](https://dbfiddle.uk/) - Alternative moderne pour MariaDB

---

## 🎓 Exercice de réflexion

**Sans écrire de code, réfléchissez** :

1. Comment récupéreriez-vous tous les "cousins" d'un employé dans un organigramme ?
2. Comment calculeriez-vous le "nombre de Erdős" (distance dans un graphe de collaboration) ?
3. Comment généreriez-vous un calendrier des 52 semaines de l'année avec leurs numéros ?
4. Comment détecteriez-vous les branches "orphelines" dans un arbre de catégories ?

💡 **Indice** : Toutes ces questions se résolvent avec `WITH RECURSIVE` !

---

## ➡️ Section suivante

**[4.2 Window Functions](./02-window-functions.md)** : Découvrez comment effectuer des calculs sophistiqués sur des ensembles de lignes sans GROUP BY, incluant les classements, moyennes mobiles, et comparaisons entre lignes adjacentes.

---


⏭️ [Window Functions](/04-concepts-avances-sql/02-window-functions.md)

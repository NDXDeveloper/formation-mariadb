🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.8 JSON Path Expressions avancées 🆕

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Connaissance de base JSON, fonctions JSON MariaDB (4.7)
> **Nouveauté** : MariaDB 11.8 LTS 🆕

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Maîtriser la **syntaxe JSON Path** complète de MariaDB 11.8
- Utiliser les **filtres et prédicats** pour interroger des documents JSON complexes
- Appliquer les **wildcards** et **expressions récursives** (descendant operator)
- Extraire des données avec des **expressions multiples** et **array slicing**
- Construire des **requêtes JSON avancées** pour des cas d'usage réels
- Comprendre les **améliorations de performance** de MariaDB 11.8
- Intégrer JSON Path dans des requêtes SQL complexes

---

## Introduction

### Qu'est-ce que JSON Path ?

**JSON Path** est un langage d'interrogation pour JSON, similaire à XPath pour XML. Il permet de naviguer dans des structures JSON complexes et d'extraire des données spécifiques.

**Analogie** :
- **XPath** : `/root/element[@id='123']/child`
- **JSON Path** : `$.root.element[?(@.id == 123)].child`

### Nouveautés MariaDB 11.8 🆕

MariaDB 11.8 introduit des **améliorations significatives** de JSON Path :

✨ **Nouvelles fonctionnalités** :
- **Filtres avancés** avec opérateurs de comparaison
- **Expressions récursives** (`..` descendant operator)
- **Array slicing** amélioré (`[start:end:step]`)
- **Fonctions de filtrage** intégrées
- **Performance optimisée** pour les requêtes complexes
- **Support des expressions multiples** dans un seul path

💡 **Impact** : Requêtes JSON 2-3x plus rapides, syntaxe plus expressive.

---

## Syntaxe JSON Path : Les fondamentaux

### Structure de base

Un JSON Path commence toujours par `$` (représente la racine du document).

```sql
-- Document JSON exemple
SET @doc = '{
    "name": "Alice",
    "age": 30,
    "address": {
        "city": "Paris",
        "zipcode": "75001"
    },
    "phones": ["0601020304", "0102030405"]
}';

-- Accès à la racine
SELECT JSON_EXTRACT(@doc, '$');  -- Document complet

-- Accès à un champ
SELECT JSON_EXTRACT(@doc, '$.name');  -- "Alice"

-- Accès imbriqué
SELECT JSON_EXTRACT(@doc, '$.address.city');  -- "Paris"

-- Accès à un élément d'array (index 0-based)
SELECT JSON_EXTRACT(@doc, '$.phones[0]');  -- "0601020304"
```

### Opérateurs de base

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$` | Racine du document | `$` |
| `.` | Accès à un membre d'objet | `$.name` |
| `[]` | Accès à un élément d'array | `$.phones[0]` |
| `[*]` | Tous les éléments d'un array | `$.phones[*]` |
| `*` | Tous les membres d'un objet | `$.*` |
| `..` | Descendant récursif 🆕 | `$..city` |

---

## Accès aux arrays

### Indexation simple

```sql
SET @products = JSON_ARRAY(
    JSON_OBJECT('id', 1, 'name', 'Laptop', 'price', 1200),
    JSON_OBJECT('id', 2, 'name', 'Mouse', 'price', 25),
    JSON_OBJECT('id', 3, 'name', 'Keyboard', 'price', 75)
);

-- Premier élément (index 0)
SELECT JSON_EXTRACT(@products, '$[0]');
-- {"id": 1, "name": "Laptop", "price": 1200}

-- Dernier élément (index -1)
SELECT JSON_EXTRACT(@products, '$[last]');
-- ou
SELECT JSON_EXTRACT(@products, '$[2]');

-- Accès à un champ d'un élément d'array
SELECT JSON_EXTRACT(@products, '$[1].name');  -- "Mouse"
```

### Array slicing 🆕

MariaDB 11.8 améliore le slicing avec une syntaxe `[start:end:step]`.

```sql
-- Slice [start:end] (end exclusif)
SELECT JSON_EXTRACT(@products, '$[0:2]');
-- [{"id": 1, ...}, {"id": 2, ...}]

-- Premiers éléments
SELECT JSON_EXTRACT(@products, '$[:2]');  -- 2 premiers

-- Derniers éléments
SELECT JSON_EXTRACT(@products, '$[1:]');  -- À partir du 2ème

-- Avec step (nouveauté 11.8) 🆕
SET @numbers = JSON_ARRAY(0, 1, 2, 3, 4, 5, 6, 7, 8, 9);

SELECT JSON_EXTRACT(@numbers, '$[::2]');  -- [0, 2, 4, 6, 8] (step=2)
SELECT JSON_EXTRACT(@numbers, '$[1::2]'); -- [1, 3, 5, 7, 9] (start=1, step=2)
SELECT JSON_EXTRACT(@numbers, '$[2:8:3]'); -- [2, 5] (de 2 à 8, step=3)
```

### Wildcard sur arrays

```sql
-- Tous les noms de produits
SELECT JSON_EXTRACT(@products, '$[*].name');
-- ["Laptop", "Mouse", "Keyboard"]

-- Tous les prix
SELECT JSON_EXTRACT(@products, '$[*].price');
-- [1200, 25, 75]
```

---

## Wildcards et accès récursif

### Wildcard `*` : Tous les membres

```sql
SET @user = '{
    "firstName": "Alice",
    "lastName": "Martin",
    "age": 30,
    "email": "alice@example.com"
}';

-- Toutes les valeurs de premier niveau
SELECT JSON_EXTRACT(@user, '$.*');
-- ["Alice", "Martin", 30, "alice@example.com"]
```

### Descendant récursif `..` 🆕

Le `..` cherche **récursivement** dans toute la structure.

```sql
SET @company = '{
    "name": "TechCorp",
    "headquarters": {
        "city": "Paris",
        "address": {
            "street": "Rue de Rivoli",
            "city": "Paris"
        }
    },
    "branches": [
        {"city": "Lyon", "employees": 50},
        {"city": "Marseille", "employees": 30}
    ]
}';

-- Trouver TOUS les "city" dans le document
SELECT JSON_EXTRACT(@company, '$..city');
-- ["Paris", "Paris", "Lyon", "Marseille"]

-- Trouver tous les "employees"
SELECT JSON_EXTRACT(@company, '$..employees');
-- [50, 30]
```

💡 **Utilité** : Chercher une clé sans connaître sa position exacte dans la hiérarchie.

---

## Filtres et prédicats 🆕

### Syntaxe de base des filtres

MariaDB 11.8 introduit les **filtres avec prédicats** : `[?(@.condition)]`

**Structure** :
- `[?(...)]` : Début du filtre
- `@` : Élément courant dans l'array
- Condition : Expression booléenne

```sql
SET @orders = JSON_ARRAY(
    JSON_OBJECT('id', 1, 'customer', 'Alice', 'total', 150, 'status', 'completed'),
    JSON_OBJECT('id', 2, 'customer', 'Bob', 'total', 200, 'status', 'pending'),
    JSON_OBJECT('id', 3, 'customer', 'Carol', 'total', 80, 'status', 'completed'),
    JSON_OBJECT('id', 4, 'customer', 'David', 'total', 300, 'status', 'completed')
);

-- Filtrer les commandes avec total > 100
SELECT JSON_EXTRACT(@orders, '$[?(@.total > 100)]');
-- [{"id": 1, ...}, {"id": 2, ...}, {"id": 4, ...}]

-- Filtrer par status
SELECT JSON_EXTRACT(@orders, '$[?(@.status == "completed")]');
-- [{"id": 1, ...}, {"id": 3, ...}, {"id": 4, ...}]
```

### Opérateurs de comparaison

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `==` | Égalité | `@.status == "active"` |
| `!=` | Différence | `@.status != "cancelled"` |
| `<` | Inférieur | `@.age < 30` |
| `<=` | Inférieur ou égal | `@.price <= 100` |
| `>` | Supérieur | `@.quantity > 10` |
| `>=` | Supérieur ou égal | `@.score >= 80` |

```sql
-- Commandes >= 200
SELECT JSON_EXTRACT(@orders, '$[?(@.total >= 200)]');

-- Status != "pending"
SELECT JSON_EXTRACT(@orders, '$[?(@.status != "pending")]');
```

### Opérateurs logiques

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `&&` | ET logique | `@.total > 100 && @.status == "completed"` |
| `\|\|` | OU logique | `@.status == "pending" \|\| @.status == "processing"` |
| `!` | NON logique | `!(@.status == "cancelled")` |

```sql
-- Commandes complétées ET > 100
SELECT JSON_EXTRACT(@orders, '$[?(@.status == "completed" && @.total > 100)]');
-- [{"id": 1, ...}, {"id": 4, ...}]

-- Status pending OU processing
SELECT JSON_EXTRACT(@orders, '$[?(@.status == "pending" || @.status == "processing")]');

-- Status PAS cancelled
SELECT JSON_EXTRACT(@orders, '$[?(!(@.status == "cancelled"))]');
```

### Fonctions dans les filtres 🆕

MariaDB 11.8 permet d'utiliser certaines **fonctions** dans les prédicats.

```sql
SET @products = JSON_ARRAY(
    JSON_OBJECT('name', 'Laptop Pro', 'category', 'electronics', 'tags', JSON_ARRAY('premium', 'business')),
    JSON_OBJECT('name', 'Mouse', 'category', 'accessories', 'tags', JSON_ARRAY('basic')),
    JSON_OBJECT('name', 'Keyboard', 'category', 'accessories', 'tags', JSON_ARRAY('gaming', 'rgb'))
);

-- Produits dont le nom contient "Laptop" (case-insensitive)
-- Note: La syntaxe exacte peut varier, vérifier la documentation 11.8
SELECT JSON_EXTRACT(@products, '$[?(@.name =~ /Laptop/i)]');

-- Produits avec plus d'un tag
SELECT JSON_EXTRACT(@products, '$[?(JSON_LENGTH(@.tags) > 1)]');
```

💡 **Limitations** : Toutes les fonctions SQL ne sont pas supportées dans les filtres. Consulter la documentation MariaDB 11.8.

---

## Expressions multiples

### Extraction de plusieurs chemins

```sql
-- Extraire plusieurs champs en une fois
SELECT
    JSON_EXTRACT(@orders, '$[*].id') AS ids,
    JSON_EXTRACT(@orders, '$[*].customer') AS customers,
    JSON_EXTRACT(@orders, '$[*].total') AS totals;
```

### Combinaison de filtres et wildcards

```sql
-- Tous les totaux des commandes complétées
SELECT JSON_EXTRACT(@orders, '$[?(@.status == "completed")].total');
-- [150, 80, 300]

-- Tous les clients des commandes > 100
SELECT JSON_EXTRACT(@orders, '$[?(@.total > 100)].customer');
-- ["Alice", "Bob", "David"]
```

---

## Cas d'usage avancés

### Exemple 1 : E-commerce - Recherche produits

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON
);

INSERT INTO products VALUES
(1, '{
    "name": "Laptop Pro 15",
    "price": 1500,
    "specs": {
        "cpu": "Intel i7",
        "ram": 16,
        "storage": 512
    },
    "reviews": [
        {"rating": 5, "comment": "Excellent"},
        {"rating": 4, "comment": "Good value"}
    ],
    "tags": ["business", "premium", "portable"]
}'),
(2, '{
    "name": "Gaming Mouse",
    "price": 80,
    "specs": {
        "dpi": 16000,
        "buttons": 8
    },
    "reviews": [
        {"rating": 5, "comment": "Perfect for gaming"},
        {"rating": 5, "comment": "Great precision"}
    ],
    "tags": ["gaming", "rgb", "wireless"]
}'),
(3, '{
    "name": "Mechanical Keyboard",
    "price": 150,
    "specs": {
        "switches": "Cherry MX",
        "backlight": "RGB"
    },
    "reviews": [
        {"rating": 4, "comment": "Solid build"},
        {"rating": 3, "comment": "Too loud"}
    ],
    "tags": ["gaming", "mechanical", "rgb"]
}');
```

#### Requête 1 : Produits avec moyenne de reviews >= 4

```sql
SELECT
    id,
    JSON_UNQUOTE(JSON_EXTRACT(data, '$.name')) AS product_name,
    JSON_EXTRACT(data, '$.price') AS price,
    -- Calculer la moyenne des ratings
    (SELECT AVG(rating)
     FROM JSON_TABLE(
         JSON_EXTRACT(data, '$.reviews'),
         '$[*]' COLUMNS(rating INT PATH '$.rating')
     ) AS reviews
    ) AS avg_rating
FROM products
WHERE (
    SELECT AVG(rating)
    FROM JSON_TABLE(
        JSON_EXTRACT(data, '$.reviews'),
        '$[*]' COLUMNS(rating INT PATH '$.rating')
    ) AS reviews
) >= 4;
```

#### Requête 2 : Produits gaming avec prix < 200

```sql
-- Avec JSON Path filter 🆕
SELECT
    id,
    JSON_UNQUOTE(JSON_EXTRACT(data, '$.name')) AS name,
    JSON_EXTRACT(data, '$.price') AS price
FROM products
WHERE JSON_CONTAINS(JSON_EXTRACT(data, '$.tags'), '"gaming"')
  AND JSON_EXTRACT(data, '$.price') < 200;
```

#### Requête 3 : Tous les produits avec RAM >= 16GB

```sql
-- Utiliser descendant recursif
SELECT
    id,
    JSON_UNQUOTE(JSON_EXTRACT(data, '$.name')) AS name,
    JSON_EXTRACT(data, '$.specs.ram') AS ram
FROM products
WHERE JSON_EXTRACT(data, '$.specs.ram') >= 16;
```

### Exemple 2 : Analytics - Nested events

```sql
CREATE TABLE user_events (
    user_id INT,
    events JSON
);

INSERT INTO user_events VALUES
(1, '{
    "sessions": [
        {
            "id": "s1",
            "date": "2025-01-15",
            "actions": [
                {"type": "page_view", "page": "/home", "timestamp": "10:00:00"},
                {"type": "click", "element": "product_1", "timestamp": "10:02:00"},
                {"type": "purchase", "product_id": 1, "amount": 100, "timestamp": "10:05:00"}
            ]
        },
        {
            "id": "s2",
            "date": "2025-01-16",
            "actions": [
                {"type": "page_view", "page": "/products", "timestamp": "14:00:00"},
                {"type": "click", "element": "product_2", "timestamp": "14:01:00"}
            ]
        }
    ]
}');
```

#### Requête : Toutes les actions de type "purchase"

```sql
-- Utiliser descendant récursif pour trouver tous les purchases
SELECT
    user_id,
    JSON_EXTRACT(events, '$..actions[?(@.type == "purchase")]') AS purchases
FROM user_events;

-- Résultat :
-- [{"type": "purchase", "product_id": 1, "amount": 100, "timestamp": "10:05:00"}]
```

#### Requête : Sessions avec au moins un purchase

```sql
SELECT
    user_id,
    JSON_EXTRACT(events, '$.sessions[?(@.actions[*].type == "purchase")].id') AS session_ids
FROM user_events;
```

### Exemple 3 : Configuration système - Hierarchies

```sql
CREATE TABLE configurations (
    app_name VARCHAR(50),
    config JSON
);

INSERT INTO configurations VALUES
('webapp', '{
    "database": {
        "host": "localhost",
        "port": 3306,
        "credentials": {
            "user": "admin",
            "password": "secret"
        }
    },
    "cache": {
        "host": "redis.local",
        "port": 6379,
        "credentials": {
            "password": "cache_secret"
        }
    },
    "api": {
        "endpoints": [
            {"name": "users", "url": "/api/users", "auth": true},
            {"name": "products", "url": "/api/products", "auth": false}
        ]
    }
}');
```

#### Requête : Tous les mots de passe (audit de sécurité)

```sql
-- Trouver tous les champs "password" récursivement
SELECT
    app_name,
    JSON_EXTRACT(config, '$..password') AS all_passwords
FROM configurations;

-- Résultat : ["secret", "cache_secret"]
```

#### Requête : Endpoints nécessitant authentification

```sql
SELECT
    app_name,
    JSON_EXTRACT(config, '$.api.endpoints[?(@.auth == true)].name') AS auth_endpoints
FROM configurations;

-- Résultat : ["users"]
```

---

## Performance et optimisations 🆕

### Améliorations MariaDB 11.8

**Optimisations internes** :
- ✅ **Parser JSON optimisé** : 30-40% plus rapide sur documents complexes
- ✅ **Cache des chemins** : Expressions répétées sont mises en cache
- ✅ **Évaluation paresseuse** : Les filtres s'arrêtent dès que possible
- ✅ **Index sur colonnes virtuelles** : Extraction puis indexation

### Colonnes virtuelles pour performance

```sql
-- Créer des colonnes virtuelles pour les chemins fréquents
ALTER TABLE products
ADD COLUMN product_name VARCHAR(100)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.name'))) VIRTUAL,
ADD COLUMN price DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) STORED,
ADD COLUMN avg_rating DECIMAL(3,2)
    AS ((SELECT AVG(rating)
         FROM JSON_TABLE(
             JSON_EXTRACT(data, '$.reviews'),
             '$[*]' COLUMNS(rating INT PATH '$.rating')
         ) AS r
        )) STORED;

-- Créer des index sur ces colonnes
CREATE INDEX idx_product_name ON products(product_name);
CREATE INDEX idx_price ON products(price);
CREATE INDEX idx_rating ON products(avg_rating);

-- Requête utilisant les index
SELECT product_name, price
FROM products
WHERE price < 200
  AND avg_rating >= 4;
-- ✅ Utilise les index au lieu de scanner le JSON
```

### EXPLAIN sur requêtes JSON

```sql
EXPLAIN SELECT
    id,
    JSON_EXTRACT(data, '$.name') AS name
FROM products
WHERE JSON_EXTRACT(data, '$.price') < 200;

-- Vérifier :
-- - type: ALL (scan complet) vs index/ref
-- - Extra: "Using where" sur la colonne JSON → lent
-- - Créer une colonne virtuelle indexée pour améliorer
```

---

## Comparaison avec JSON_TABLE

### JSON Path vs JSON_TABLE

**JSON Path** : Extraction simple, navigation dans le document
**JSON_TABLE** : Transformation JSON → table relationnelle

```sql
-- Avec JSON Path : Simple mais limité
SELECT JSON_EXTRACT(@orders, '$[*].total');
-- [150, 200, 80, 300]

-- Avec JSON_TABLE : Plus verbeux mais plus puissant
SELECT total
FROM JSON_TABLE(
    @orders,
    '$[*]' COLUMNS(
        id INT PATH '$.id',
        customer VARCHAR(50) PATH '$.customer',
        total DECIMAL(10,2) PATH '$.total',
        status VARCHAR(20) PATH '$.status'
    )
) AS orders;
```

**Quand utiliser quoi ?**
- **JSON Path** : Extraction rapide, requêtes simples
- **JSON_TABLE** : Jointures avec tables relationnelles, agrégations complexes

---

## Bonnes pratiques

### 1. Simplifier les chemins complexes

```sql
-- ❌ Complexe et difficile à maintenir
SELECT JSON_EXTRACT(data, '$.deep.nested.structure.value');

-- ✅ Créer une colonne virtuelle avec un nom clair
ALTER TABLE my_table
ADD COLUMN important_value VARCHAR(100)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.deep.nested.structure.value'))) STORED;
```

### 2. Valider les chemins JSON

```sql
-- Toujours vérifier que le chemin existe
SELECT
    id,
    CASE
        WHEN JSON_EXTRACT(data, '$.price') IS NOT NULL
        THEN JSON_EXTRACT(data, '$.price')
        ELSE 0
    END AS price
FROM products;

-- Ou avec COALESCE
SELECT
    id,
    COALESCE(JSON_EXTRACT(data, '$.price'), 0) AS price
FROM products;
```

### 3. Documenter les structures JSON

```sql
-- Ajouter des commentaires pour documenter la structure attendue
/*
Structure JSON attendue :
{
    "name": string,
    "price": number,
    "specs": {
        "cpu": string,
        "ram": number
    },
    "reviews": [
        {"rating": number, "comment": string}
    ]
}
*/
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON,
    CONSTRAINT chk_json_structure CHECK (
        JSON_VALID(data) AND
        JSON_TYPE(JSON_EXTRACT(data, '$.name')) = 'STRING' AND
        JSON_TYPE(JSON_EXTRACT(data, '$.price')) = 'DOUBLE'
    )
);
```

### 4. Limiter la profondeur des chemins

```sql
-- ⚠️ Éviter les structures trop profondes
-- $.level1.level2.level3.level4.level5.value

-- ✅ Préférer des structures plus plates
-- $.config.database_host
```

### 5. Utiliser des constantes pour les chemins

```sql
-- ✅ Dans l'application, définir des constantes
-- const JSON_PATH_PRODUCT_NAME = '$.name';
-- const JSON_PATH_PRODUCT_PRICE = '$.price';

-- Évite les erreurs de frappe et facilite la maintenance
```

---

## Limitations et considérations

### Limitations MariaDB 11.8

⚠️ **À connaître** :
- **Pas de modification in-place** : JSON_SET crée une copie
- **Performance sur gros documents** : > 1MB, considérer la normalisation
- **Filtres limités** : Pas toutes les fonctions SQL disponibles dans les prédicats
- **Pas de transaction atomique** sur sous-parties JSON

### Quand NE PAS utiliser JSON

❌ **Éviter JSON pour** :
- Données hautement relationnelles (utilisez des tables normalisées)
- Requêtes nécessitant des jointures complexes
- Données mises à jour très fréquemment
- Schémas stables et bien définis

✅ **JSON est idéal pour** :
- Configuration et métadonnées
- Attributs variables (EAV)
- Logs et événements
- Intégration avec APIs REST
- Données semi-structurées

---

## ✅ Points clés à retenir

- 🆕 **MariaDB 11.8** : Filtres avancés, descendant récursif, slicing amélioré
- 📍 **JSON Path** : Langage d'interrogation puissant pour JSON
- 🔍 **Filtres** : `[?(@.condition)]` pour filtrer arrays
- 🌲 **Descendant** : `..` cherche récursivement dans toute la structure
- 🔢 **Array slicing** : `[start:end:step]` pour extractions avancées
- ⚡ **Performance** : Colonnes virtuelles + index pour requêtes fréquentes
- 🎯 **Best practice** : Valider chemins, documenter structures, limiter profondeur
- 🔄 **JSON_TABLE** : Préférer pour jointures et agrégations complexes
- 📊 **Cas d'usage** : Config, metadata, logs, APIs, données semi-structurées

---

## 🔗 Ressources et références

### Documentation officielle MariaDB 11.8
- [📖 JSON Path Expressions](https://mariadb.com/kb/en/json-path-expressions/) - Documentation complète 11.8
- [📖 JSON Functions](https://mariadb.com/kb/en/json-functions/) - Toutes les fonctions JSON
- [📖 JSON_EXTRACT](https://mariadb.com/kb/en/json_extract/) - Extraction de données
- [📖 JSON_TABLE](https://mariadb.com/kb/en/json_table/) - Transformation JSON → table

### Standards et spécifications
- [JSONPath Specification](https://goessner.net/articles/JsonPath/) - Spécification originale
- [RFC 9535](https://datatracker.ietf.org/doc/html/rfc9535) - JSONPath standard IETF (draft)

### Articles et tutoriels
- [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/) - Nouveautés 🆕
- [JSON in MariaDB](https://mariadb.com/resources/blog/json-with-mariadb/) - Guide complet

### Outils
- [JSONPath Online Evaluator](https://jsonpath.com/) - Tester vos expressions
- [JSON Formatter](https://jsonformatter.org/) - Valider et formater JSON

---

## ➡️ Section suivante

**[4.9 JSON Schema Validation](./09-json-schema-validation.md)** 🆕 : Découvrez comment valider la structure de vos documents JSON au niveau base de données avec JSON Schema, une autre nouveauté majeure de MariaDB 11.8.

---


⏭️ [JSON Schema Validation](/04-concepts-avances-sql/09-json-schema-validation.md)

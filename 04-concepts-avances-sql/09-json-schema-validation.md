🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.9 JSON Schema Validation 🆕

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Connaissance JSON, JSON Path (4.7, 4.8), contraintes SQL

> **Nouveauté** : MariaDB 11.8 LTS 🆕

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les **avantages de la validation JSON** au niveau base de données
- Maîtriser la **syntaxe JSON Schema** dans MariaDB 11.8
- Définir des **contraintes de validation** pour types, formats, structures
- Valider des **objets et arrays imbriqués** complexes
- Intégrer JSON Schema avec les **contraintes CHECK**
- Appliquer la validation à des **cas d'usage réels** (e-commerce, API, config)
- Optimiser les **performances** de validation
- Comprendre les **limites** et alternatives

---

## Introduction

### Qu'est-ce que JSON Schema ?

**JSON Schema** est un vocabulaire permettant de **décrire et valider** la structure de documents JSON. C'est un standard reconnu (draft-07, draft 2019-09, draft 2020-12) qui définit :

- **Types de données** attendus (string, number, boolean, object, array, null)
- **Contraintes** sur les valeurs (min/max, pattern, enum)
- **Structure** des objets (propriétés requises, additionnelles)
- **Composition** (nested objects, arrays)

**Analogie** : JSON Schema est aux données JSON ce que `CREATE TABLE` est aux données relationnelles.

### Pourquoi valider JSON au niveau base de données ? 🆕

#### Avant MariaDB 11.8 : Validation applicative

```sql
-- ❌ Pas de validation structurelle
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON  -- ⚠️ Accepte N'IMPORTE QUEL JSON valide
);

-- Ces INSERT sont tous acceptés, même incorrects !
INSERT INTO products VALUES (1, '{"name": "Laptop"}');  -- Manque price
INSERT INTO products VALUES (2, '{"price": "expensive"}');  -- Type incorrect
INSERT INTO products VALUES (3, '{"name": 123}');  -- Type incorrect
```

**Problèmes** :
- 🔴 Validation dans chaque application/service
- 🔴 Risque d'incohérence entre applications
- 🔴 Données invalides en base
- 🔴 Bugs découverts tard

#### Avec MariaDB 11.8 : Validation native 🆕

```sql
-- ✅ Validation au niveau base de données
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON,
    CONSTRAINT valid_product CHECK (
        JSON_SCHEMA_VALID('{
            "type": "object",
            "required": ["name", "price"],
            "properties": {
                "name": {"type": "string"},
                "price": {"type": "number", "minimum": 0}
            }
        }', data)
    )
);

-- ✅ INSERT valide accepté
INSERT INTO products VALUES (1, '{"name": "Laptop", "price": 1200}');

-- ❌ INSERT invalides rejetés
INSERT INTO products VALUES (2, '{"name": "Mouse"}');  -- ERROR: price manquant
INSERT INTO products VALUES (3, '{"price": "expensive"}');  -- ERROR: type incorrect
```

**Avantages** :
- ✅ **Source unique de vérité** : Schéma centralisé
- ✅ **Garantie d'intégrité** : Impossible d'insérer des données invalides
- ✅ **Simplification** : Moins de code de validation dans les apps
- ✅ **Documentation** : Le schéma documente la structure attendue
- ✅ **Migration facilitée** : Validation lors du changement de schéma

---

## Syntaxe JSON Schema de base

### Structure minimale

```sql
-- Schéma le plus simple : type obligatoire
SET @schema = '{
    "type": "object"
}';

-- Validation
SELECT JSON_SCHEMA_VALID(@schema, '{"any": "value"}');  -- 1 (TRUE)
SELECT JSON_SCHEMA_VALID(@schema, '[]');  -- 0 (FALSE, array au lieu d'object)
```

### Types de données

| Type JSON Schema | Correspond à | Exemple |
|------------------|--------------|---------|
| `string` | Chaîne de caractères | `"hello"` |
| `number` | Nombre (int ou float) | `42`, `3.14` |
| `integer` | Entier seulement | `42` |
| `boolean` | Booléen | `true`, `false` |
| `object` | Objet JSON | `{"key": "value"}` |
| `array` | Tableau JSON | `[1, 2, 3]` |
| `null` | Valeur null | `null` |

```sql
-- Schéma avec type string
SET @schema_string = '{"type": "string"}';
SELECT JSON_SCHEMA_VALID(@schema_string, '"hello"');  -- 1
SELECT JSON_SCHEMA_VALID(@schema_string, '123');  -- 0

-- Schéma avec type number
SET @schema_number = '{"type": "number"}';
SELECT JSON_SCHEMA_VALID(@schema_number, '42');  -- 1
SELECT JSON_SCHEMA_VALID(@schema_number, '3.14');  -- 1
SELECT JSON_SCHEMA_VALID(@schema_number, '"42"');  -- 0 (string)

-- Schéma avec type integer (plus strict que number)
SET @schema_integer = '{"type": "integer"}';
SELECT JSON_SCHEMA_VALID(@schema_integer, '42');  -- 1
SELECT JSON_SCHEMA_VALID(@schema_integer, '3.14');  -- 0 (float)
```

---

## Validation d'objets JSON

### Propriétés requises

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    profile JSON,
    CONSTRAINT valid_profile CHECK (
        JSON_SCHEMA_VALID('{
            "type": "object",
            "required": ["username", "email"],
            "properties": {
                "username": {"type": "string"},
                "email": {"type": "string"}
            }
        }', profile)
    )
);

-- ✅ Valide
INSERT INTO users VALUES (1, '{
    "username": "alice",
    "email": "alice@example.com"
}');

-- ❌ Invalide : email manquant
INSERT INTO users VALUES (2, '{"username": "bob"}');
-- ERROR: Check constraint 'valid_profile' violated

-- ❌ Invalide : type incorrect
INSERT INTO users VALUES (3, '{
    "username": 123,
    "email": "carol@example.com"
}');
-- ERROR: Check constraint 'valid_profile' violated
```

### Propriétés optionnelles

```sql
-- username requis, bio optionnel
SET @schema = '{
    "type": "object",
    "required": ["username"],
    "properties": {
        "username": {"type": "string"},
        "bio": {"type": "string"}
    }
}';

SELECT JSON_SCHEMA_VALID(@schema, '{"username": "alice"}');  -- 1 (bio absent OK)
SELECT JSON_SCHEMA_VALID(@schema, '{"username": "alice", "bio": "Developer"}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"bio": "Developer"}');  -- 0 (username manquant)
```

### Propriétés additionnelles

```sql
-- Par défaut, propriétés non déclarées sont autorisées
SET @schema_default = '{
    "type": "object",
    "properties": {
        "name": {"type": "string"}
    }
}';

SELECT JSON_SCHEMA_VALID(@schema_default, '{
    "name": "Alice",
    "age": 30
}');  -- 1 (age non déclaré mais accepté)

-- Interdire les propriétés additionnelles
SET @schema_strict = '{
    "type": "object",
    "properties": {
        "name": {"type": "string"}
    },
    "additionalProperties": false
}';

SELECT JSON_SCHEMA_VALID(@schema_strict, '{
    "name": "Alice",
    "age": 30
}');  -- 0 (age non autorisé)

SELECT JSON_SCHEMA_VALID(@schema_strict, '{"name": "Alice"}');  -- 1
```

---

## Contraintes sur les valeurs

### Nombres : min, max, multipleOf

```sql
SET @schema = '{
    "type": "object",
    "properties": {
        "age": {
            "type": "integer",
            "minimum": 0,
            "maximum": 150
        },
        "price": {
            "type": "number",
            "minimum": 0,
            "exclusiveMinimum": true
        },
        "quantity": {
            "type": "integer",
            "multipleOf": 5
        }
    }
}';

-- age : 0-150
SELECT JSON_SCHEMA_VALID(@schema, '{"age": 25}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"age": -1}');  -- 0
SELECT JSON_SCHEMA_VALID(@schema, '{"age": 200}');  -- 0

-- price : > 0 (exclusif)
SELECT JSON_SCHEMA_VALID(@schema, '{"price": 0.01}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"price": 0}');  -- 0 (exclusif)

-- quantity : multiple de 5
SELECT JSON_SCHEMA_VALID(@schema, '{"quantity": 10}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"quantity": 7}');  -- 0
```

### Chaînes : minLength, maxLength, pattern

```sql
SET @schema = '{
    "type": "object",
    "properties": {
        "username": {
            "type": "string",
            "minLength": 3,
            "maxLength": 20,
            "pattern": "^[a-zA-Z0-9_]+$"
        },
        "email": {
            "type": "string",
            "format": "email"
        },
        "zipcode": {
            "type": "string",
            "pattern": "^[0-9]{5}$"
        }
    }
}';

-- username : 3-20 chars, alphanumeric + underscore
SELECT JSON_SCHEMA_VALID(@schema, '{"username": "alice"}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"username": "ab"}');  -- 0 (trop court)
SELECT JSON_SCHEMA_VALID(@schema, '{"username": "alice@"}');  -- 0 (char invalide)

-- zipcode : exactement 5 chiffres
SELECT JSON_SCHEMA_VALID(@schema, '{"zipcode": "75001"}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"zipcode": "7500"}');  -- 0 (4 chiffres)
```

### Enum : valeurs autorisées

```sql
SET @schema = '{
    "type": "object",
    "properties": {
        "status": {
            "type": "string",
            "enum": ["pending", "processing", "completed", "cancelled"]
        },
        "priority": {
            "type": "integer",
            "enum": [1, 2, 3, 4, 5]
        }
    }
}';

SELECT JSON_SCHEMA_VALID(@schema, '{"status": "pending"}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"status": "invalid"}');  -- 0
SELECT JSON_SCHEMA_VALID(@schema, '{"priority": 3}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"priority": 10}');  -- 0
```

---

## Validation d'arrays

### Arrays de types simples

```sql
SET @schema = '{
    "type": "object",
    "properties": {
        "tags": {
            "type": "array",
            "items": {"type": "string"},
            "minItems": 1,
            "maxItems": 10,
            "uniqueItems": true
        }
    }
}';

-- ✅ Valide
SELECT JSON_SCHEMA_VALID(@schema, '{"tags": ["tag1", "tag2"]}');  -- 1

-- ❌ Array vide (minItems: 1)
SELECT JSON_SCHEMA_VALID(@schema, '{"tags": []}');  -- 0

-- ❌ Type incorrect dans array
SELECT JSON_SCHEMA_VALID(@schema, '{"tags": ["tag1", 123]}');  -- 0

-- ❌ Doublons (uniqueItems: true)
SELECT JSON_SCHEMA_VALID(@schema, '{"tags": ["tag1", "tag1"]}');  -- 0
```

### Arrays d'objets

```sql
SET @schema = '{
    "type": "object",
    "properties": {
        "items": {
            "type": "array",
            "items": {
                "type": "object",
                "required": ["id", "quantity"],
                "properties": {
                    "id": {"type": "integer"},
                    "quantity": {"type": "integer", "minimum": 1}
                }
            }
        }
    }
}';

-- ✅ Valide
SELECT JSON_SCHEMA_VALID(@schema, '{
    "items": [
        {"id": 1, "quantity": 2},
        {"id": 2, "quantity": 5}
    ]
}');  -- 1

-- ❌ Objet invalide dans array (quantity manquant)
SELECT JSON_SCHEMA_VALID(@schema, '{
    "items": [
        {"id": 1, "quantity": 2},
        {"id": 2}
    ]
}');  -- 0
```

### Tuples (arrays avec types positionnels)

```sql
-- Array de longueur fixe avec types spécifiques par position
SET @schema = '{
    "type": "array",
    "items": [
        {"type": "string"},
        {"type": "integer"},
        {"type": "number"}
    ],
    "minItems": 3,
    "maxItems": 3
}';

-- ✅ Valide : [string, int, number]
SELECT JSON_SCHEMA_VALID(@schema, '["Alice", 30, 5.7]');  -- 1

-- ❌ Ordre incorrect
SELECT JSON_SCHEMA_VALID(@schema, '[30, "Alice", 5.7]');  -- 0

-- ❌ Trop d'éléments
SELECT JSON_SCHEMA_VALID(@schema, '["Alice", 30, 5.7, "extra"]');  -- 0
```

---

## Structures imbriquées complexes

### Objets dans objets (nested)

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    data JSON,
    CONSTRAINT valid_product CHECK (
        JSON_SCHEMA_VALID('{
            "type": "object",
            "required": ["name", "price", "specs"],
            "properties": {
                "name": {"type": "string"},
                "price": {"type": "number", "minimum": 0},
                "specs": {
                    "type": "object",
                    "required": ["cpu", "ram"],
                    "properties": {
                        "cpu": {"type": "string"},
                        "ram": {"type": "integer", "minimum": 1},
                        "storage": {"type": "integer"}
                    }
                }
            }
        }', data)
    )
);

-- ✅ Valide
INSERT INTO products VALUES (1, '{
    "name": "Laptop",
    "price": 1200,
    "specs": {
        "cpu": "Intel i7",
        "ram": 16,
        "storage": 512
    }
}');

-- ❌ specs.ram manquant
INSERT INTO products VALUES (2, '{
    "name": "Laptop",
    "price": 1200,
    "specs": {
        "cpu": "Intel i7"
    }
}');
-- ERROR
```

### Arrays dans objets dans arrays

```sql
SET @schema = '{
    "type": "object",
    "properties": {
        "orders": {
            "type": "array",
            "items": {
                "type": "object",
                "required": ["order_id", "items"],
                "properties": {
                    "order_id": {"type": "integer"},
                    "items": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "required": ["product_id", "quantity"],
                            "properties": {
                                "product_id": {"type": "integer"},
                                "quantity": {"type": "integer", "minimum": 1}
                            }
                        },
                        "minItems": 1
                    }
                }
            }
        }
    }
}';

-- ✅ Structure valide
SELECT JSON_SCHEMA_VALID(@schema, '{
    "orders": [
        {
            "order_id": 1,
            "items": [
                {"product_id": 101, "quantity": 2},
                {"product_id": 102, "quantity": 1}
            ]
        }
    ]
}');  -- 1
```

---

## Cas d'usage pratiques

### Exemple 1 : E-commerce - Produits

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Contrainte de validation complète
    CONSTRAINT valid_product_schema CHECK (
        JSON_SCHEMA_VALID('{
            "type": "object",
            "required": ["name", "sku", "price", "category", "status"],
            "properties": {
                "name": {
                    "type": "string",
                    "minLength": 3,
                    "maxLength": 200
                },
                "sku": {
                    "type": "string",
                    "pattern": "^[A-Z]{3}-[0-9]{6}$"
                },
                "price": {
                    "type": "number",
                    "minimum": 0,
                    "exclusiveMinimum": true
                },
                "category": {
                    "type": "string",
                    "enum": ["electronics", "clothing", "books", "home", "sports"]
                },
                "status": {
                    "type": "string",
                    "enum": ["draft", "active", "archived"]
                },
                "description": {
                    "type": "string",
                    "maxLength": 2000
                },
                "tags": {
                    "type": "array",
                    "items": {"type": "string"},
                    "uniqueItems": true,
                    "maxItems": 10
                },
                "stock": {
                    "type": "object",
                    "required": ["quantity", "warehouse"],
                    "properties": {
                        "quantity": {
                            "type": "integer",
                            "minimum": 0
                        },
                        "warehouse": {
                            "type": "string"
                        },
                        "reserved": {
                            "type": "integer",
                            "minimum": 0,
                            "default": 0
                        }
                    }
                },
                "dimensions": {
                    "type": "object",
                    "properties": {
                        "length": {"type": "number", "minimum": 0},
                        "width": {"type": "number", "minimum": 0},
                        "height": {"type": "number", "minimum": 0},
                        "weight": {"type": "number", "minimum": 0},
                        "unit": {
                            "type": "string",
                            "enum": ["cm", "in"]
                        }
                    }
                }
            },
            "additionalProperties": false
        }', data)
    )
);

-- ✅ INSERT valide
INSERT INTO products (data) VALUES ('{
    "name": "Laptop Pro 15",
    "sku": "ELC-123456",
    "price": 1299.99,
    "category": "electronics",
    "status": "active",
    "description": "Professional laptop with high performance",
    "tags": ["laptop", "business", "premium"],
    "stock": {
        "quantity": 50,
        "warehouse": "Paris-01"
    },
    "dimensions": {
        "length": 35.0,
        "width": 24.5,
        "height": 2.0,
        "weight": 1.8,
        "unit": "cm"
    }
}');

-- ❌ SKU format invalide
INSERT INTO products (data) VALUES ('{
    "name": "Invalid Product",
    "sku": "INVALID",
    "price": 100,
    "category": "electronics",
    "status": "active"
}');
-- ERROR: Check constraint violated
```

### Exemple 2 : API Configuration

```sql
CREATE TABLE api_endpoints (
    id INT PRIMARY KEY AUTO_INCREMENT,
    endpoint_name VARCHAR(100) UNIQUE,
    config JSON NOT NULL,

    CONSTRAINT valid_api_config CHECK (
        JSON_SCHEMA_VALID('{
            "type": "object",
            "required": ["path", "method", "auth", "rate_limit"],
            "properties": {
                "path": {
                    "type": "string",
                    "pattern": "^/api/v[0-9]+/"
                },
                "method": {
                    "type": "string",
                    "enum": ["GET", "POST", "PUT", "PATCH", "DELETE"]
                },
                "auth": {
                    "type": "object",
                    "required": ["required", "type"],
                    "properties": {
                        "required": {"type": "boolean"},
                        "type": {
                            "type": "string",
                            "enum": ["bearer", "api_key", "basic", "oauth2"]
                        },
                        "scopes": {
                            "type": "array",
                            "items": {"type": "string"}
                        }
                    }
                },
                "rate_limit": {
                    "type": "object",
                    "required": ["requests", "window"],
                    "properties": {
                        "requests": {
                            "type": "integer",
                            "minimum": 1,
                            "maximum": 10000
                        },
                        "window": {
                            "type": "string",
                            "enum": ["second", "minute", "hour", "day"]
                        }
                    }
                },
                "timeout": {
                    "type": "integer",
                    "minimum": 100,
                    "maximum": 60000
                },
                "cache": {
                    "type": "object",
                    "properties": {
                        "enabled": {"type": "boolean"},
                        "ttl": {"type": "integer", "minimum": 0}
                    }
                }
            },
            "additionalProperties": false
        }', config)
    )
);

-- ✅ Configuration valide
INSERT INTO api_endpoints (endpoint_name, config) VALUES
('get_users', '{
    "path": "/api/v1/users",
    "method": "GET",
    "auth": {
        "required": true,
        "type": "bearer",
        "scopes": ["users:read"]
    },
    "rate_limit": {
        "requests": 100,
        "window": "minute"
    },
    "timeout": 5000,
    "cache": {
        "enabled": true,
        "ttl": 300
    }
}');
```

### Exemple 3 : User Preferences

```sql
CREATE TABLE user_preferences (
    user_id INT PRIMARY KEY,
    preferences JSON NOT NULL,

    CONSTRAINT valid_preferences CHECK (
        JSON_SCHEMA_VALID('{
            "type": "object",
            "properties": {
                "theme": {
                    "type": "string",
                    "enum": ["light", "dark", "auto"]
                },
                "language": {
                    "type": "string",
                    "pattern": "^[a-z]{2}(-[A-Z]{2})?$"
                },
                "notifications": {
                    "type": "object",
                    "properties": {
                        "email": {"type": "boolean"},
                        "push": {"type": "boolean"},
                        "sms": {"type": "boolean"},
                        "frequency": {
                            "type": "string",
                            "enum": ["realtime", "daily", "weekly", "never"]
                        }
                    }
                },
                "privacy": {
                    "type": "object",
                    "properties": {
                        "profile_visible": {"type": "boolean"},
                        "show_email": {"type": "boolean"},
                        "allow_messages": {"type": "boolean"}
                    }
                },
                "display": {
                    "type": "object",
                    "properties": {
                        "items_per_page": {
                            "type": "integer",
                            "minimum": 10,
                            "maximum": 100,
                            "multipleOf": 10
                        },
                        "timezone": {"type": "string"}
                    }
                }
            }
        }', preferences)
    )
);
```

---

## Performance et optimisations

### Impact sur les performances

**Coût de validation** :
- ⚡ **Très rapide** pour schémas simples (<1ms)
- 🟡 **Modéré** pour schémas complexes (1-5ms)
- 🔴 **Peut être lent** pour schémas très imbriqués (>10ms)

```sql
-- Benchmark simple
SET @start = UNIX_TIMESTAMP(CURTIME(6));

-- Valider 1000 fois
SELECT COUNT(*) FROM (
    SELECT JSON_SCHEMA_VALID(@simple_schema, @data)
    FROM (
        SELECT 1 UNION SELECT 2 UNION SELECT 3 -- ... x1000
    ) AS numbers
) AS validation;

SET @end = UNIX_TIMESTAMP(CURTIME(6));
SELECT (@end - @start) AS elapsed_seconds;
```

### Bonnes pratiques de performance

#### 1. Simplifier les schémas quand possible

```sql
-- ❌ Schéma trop complexe
{
    "type": "object",
    "properties": {
        "field1": {...deep nested...},
        "field2": {...deep nested...},
        ...
        "field50": {...deep nested...}
    }
}

-- ✅ Séparer en plusieurs tables si possible
CREATE TABLE main_data (...);
CREATE TABLE additional_data (...);
```

#### 2. Valider seulement l'essentiel

```sql
-- ❌ Validation de TOUT (trop strict)
CONSTRAINT check_all CHECK (
    JSON_SCHEMA_VALID(@huge_schema, data)
)

-- ✅ Validation des champs critiques seulement
CONSTRAINT check_critical CHECK (
    JSON_SCHEMA_VALID('{
        "type": "object",
        "required": ["id", "type", "status"],
        "properties": {
            "id": {"type": "integer"},
            "type": {"type": "string"},
            "status": {"enum": ["active", "inactive"]}
        }
    }', data)
)
```

#### 3. Utiliser des index sur colonnes virtuelles

```sql
-- Extraire et indexer les champs fréquemment filtrés
ALTER TABLE products
ADD COLUMN status VARCHAR(20)
    AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.status'))) STORED,
ADD COLUMN price DECIMAL(10,2)
    AS (JSON_EXTRACT(data, '$.price')) STORED;

CREATE INDEX idx_status ON products(status);
CREATE INDEX idx_price ON products(price);

-- Requête utilise les index
SELECT * FROM products WHERE status = 'active' AND price < 1000;
```

---

## Fonctions de validation

### JSON_SCHEMA_VALID()

```sql
-- Retourne 1 (TRUE) ou 0 (FALSE)
SELECT JSON_SCHEMA_VALID(@schema, @data);

-- Usage dans contraintes
CONSTRAINT chk CHECK (JSON_SCHEMA_VALID(@schema, col) = 1)
```

### JSON_SCHEMA_VALIDATION_REPORT() 🆕

Retourne un rapport détaillé de validation (MariaDB 11.8+).

```sql
-- Schéma avec erreurs multiples
SET @schema = '{
    "type": "object",
    "required": ["name", "age"],
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer", "minimum": 0}
    }
}';

SET @invalid_data = '{"age": -5}';

-- Rapport détaillé
SELECT JSON_SCHEMA_VALIDATION_REPORT(@schema, @invalid_data);

-- Résultat (exemple) :
/*
{
    "valid": false,
    "errors": [
        {
            "keyword": "required",
            "instancePath": "",
            "message": "must have required property 'name'"
        },
        {
            "keyword": "minimum",
            "instancePath": "/age",
            "message": "must be >= 0"
        }
    ]
}
*/
```

💡 **Utilité** : Debugging, affichage d'erreurs aux utilisateurs, logging.

---

## Migration et évolution de schéma

### Ajouter validation à une table existante

```sql
-- Table existante sans validation
CREATE TABLE old_products (
    id INT PRIMARY KEY,
    data JSON
);

-- 1. Vérifier les données existantes
SELECT id, data
FROM old_products
WHERE NOT JSON_SCHEMA_VALID('{
    "type": "object",
    "required": ["name", "price"],
    "properties": {
        "name": {"type": "string"},
        "price": {"type": "number"}
    }
}', data);

-- 2. Corriger les données invalides
UPDATE old_products
SET data = JSON_SET(data, '$.price', 0)
WHERE JSON_EXTRACT(data, '$.price') IS NULL;

-- 3. Ajouter la contrainte
ALTER TABLE old_products
ADD CONSTRAINT valid_schema CHECK (
    JSON_SCHEMA_VALID(..., data)
);
```

### Faire évoluer le schéma

```sql
-- Version 1 : name et price requis
-- Version 2 : Ajouter category requis

-- ❌ Problème : Les anciennes données n'ont pas category
ALTER TABLE products
DROP CONSTRAINT old_constraint,
ADD CONSTRAINT new_constraint CHECK (
    JSON_SCHEMA_VALID('{
        "required": ["name", "price", "category"],
        ...
    }', data)
);
-- ERROR: Les anciennes lignes violent la contrainte

-- ✅ Solution 1 : Migration préalable
UPDATE products
SET data = JSON_SET(data, '$.category', 'uncategorized')
WHERE JSON_EXTRACT(data, '$.category') IS NULL;

-- Puis ajouter contrainte

-- ✅ Solution 2 : Schema versionné
ALTER TABLE products
ADD COLUMN schema_version INT DEFAULT 1;

-- Contrainte conditionnelle selon version
CONSTRAINT check_versioned CHECK (
    (schema_version = 1 AND JSON_SCHEMA_VALID(@schema_v1, data)) OR
    (schema_version = 2 AND JSON_SCHEMA_VALID(@schema_v2, data))
)
```

---

## Limitations et considérations

### Limitations MariaDB 11.8

⚠️ **À connaître** :
- **Pas de $ref** : Les références de schéma ne sont pas supportées
- **Formats limités** : Seuls quelques formats sont validés (email, date-time basique)
- **Performance** : Validation coûteuse sur gros documents (>100KB)
- **Messages d'erreur** : Générique "constraint violated" (utiliser validation_report pour debug)

### Standards JSON Schema non supportés

❌ **Non disponibles** :
- `$ref` et `$id` (références de schéma)
- `dependencies` et `dependentSchemas`
- `if/then/else` conditionnel
- `allOf`, `anyOf`, `oneOf` (combinaisons complexes)
- Formats avancés (ipv4, ipv6, uri, uuid avec validation stricte)

✅ **Disponibles** :
- Types de base
- Contraintes simples (min/max, pattern, enum)
- Objects et arrays imbriqués
- required, additionalProperties

### Alternatives et compléments

**1. Validation applicative** : Pour logique complexe

```javascript
// Application Node.js avec Ajv
const Ajv = require('ajv');
const ajv = new Ajv();

const schema = {...complex schema...};
const validate = ajv.compile(schema);
const valid = validate(data);
```

**2. Triggers** : Validation custom

```sql
CREATE TRIGGER validate_product_before_insert
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
    IF JSON_EXTRACT(NEW.data, '$.price') < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Price must be positive';
    END IF;
END;
```

**3. Stored procedures** : Validation complexe

```sql
CREATE PROCEDURE validate_complex_product(IN product_data JSON)
BEGIN
    -- Logique de validation complexe
    DECLARE price DECIMAL(10,2);
    SET price = JSON_EXTRACT(product_data, '$.price');

    IF price < 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid price';
    END IF;

    -- Plus de validations...
END;
```

---

## ✅ Points clés à retenir

- 🆕 **MariaDB 11.8** : Validation JSON Schema native avec CHECK constraints
- ✅ **Avantages** : Source unique de vérité, garantie d'intégrité, documentation
- 📐 **Syntaxe** : Standard JSON Schema (types, required, properties, constraints)
- 🔢 **Contraintes** : min/max, pattern, enum, minLength/maxLength
- 🏗️ **Structures** : Objects imbriqués, arrays d'objets, validation profonde
- 🛠️ **Fonction** : `JSON_SCHEMA_VALID()` dans CHECK constraints
- 📊 **Debug** : `JSON_SCHEMA_VALIDATION_REPORT()` pour rapports détaillés
- ⚡ **Performance** : <1ms pour schémas simples, optimiser avec colonnes virtuelles
- ⚠️ **Limitations** : Pas de $ref, formats limités, messages d'erreur génériques
- 🔄 **Migration** : Valider données existantes avant ajout contrainte

---

## 🔗 Ressources et références

### Documentation officielle MariaDB 11.8
- [📖 JSON_SCHEMA_VALID](https://mariadb.com/kb/en/json_schema_valid/) - Fonction de validation 🆕
- [📖 JSON_SCHEMA_VALIDATION_REPORT](https://mariadb.com/kb/en/json_schema_validation_report/) - Rapports détaillés 🆕
- [📖 CHECK Constraints](https://mariadb.com/kb/en/constraint/) - Contraintes de validation
- [📖 JSON Functions](https://mariadb.com/kb/en/json-functions/) - Toutes fonctions JSON

### Standards JSON Schema
- [JSON Schema Official Site](https://json-schema.org/) - Documentation complète
- [JSON Schema Specification](https://json-schema.org/specification.html) - Specs officielles
- [Understanding JSON Schema](https://json-schema.org/understanding-json-schema/) - Guide complet

### Outils et validateurs
- [JSON Schema Validator](https://www.jsonschemavalidator.net/) - Tester vos schémas en ligne
- [Ajv JSON Schema Validator](https://ajv.js.org/) - Library JavaScript (Node.js)
- [JSON Schema Lint](https://jsonschemalint.com/) - Validation et linting

### Articles et tutoriels
- [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/) - Nouveautés 🆕
- [JSON Schema Best Practices](https://json-schema.org/learn/getting-started-step-by-step) - Bonnes pratiques

---

## ➡️ Section suivante

**[4.10 Indexation de colonnes virtuelles extraites du JSON](./10-indexation-colonnes-virtuelles-json.md)** : Apprenez à optimiser les performances de vos requêtes JSON en créant des colonnes virtuelles indexées et en exploitant au mieux les capacités de MariaDB.

---


⏭️ [Indexation de colonnes virtuelles extraites du JSON](/04-concepts-avances-sql/10-indexation-colonnes-virtuelles-json.md)

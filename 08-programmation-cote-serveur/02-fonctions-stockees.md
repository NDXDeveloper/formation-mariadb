🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.2 Fonctions Stockées (Stored Functions)

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Procédures stockées (8.1), SQL avancé (chapitres 3-4)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les différences fondamentales entre fonctions et procédures stockées
- **Créer** des fonctions stockées avec clause RETURNS appropriée
- **Utiliser** les caractéristiques DETERMINISTIC, NO SQL, READS SQL DATA
- **Intégrer** des fonctions dans les requêtes SELECT, WHERE, ORDER BY
- **Optimiser** les performances avec les déclarations appropriées
- **Éviter** les pièges courants liés aux fonctions stockées
- **Appliquer** les bonnes pratiques de développement de fonctions

---

## Introduction

Les **fonctions stockées** (stored functions) sont des routines serveur qui retournent une valeur unique et peuvent être utilisées directement dans des expressions SQL. Contrairement aux procédures stockées qui sont appelées avec `CALL`, les fonctions s'utilisent comme les fonctions natives de MariaDB.

### 🔍 Qu'est-ce qu'une Fonction Stockée ?

```sql
-- Exemple simple : Fonction de calcul
DELIMITER $$
CREATE FUNCTION calculate_tax(amount DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    RETURN amount * 0.20;  -- TVA 20%
END$$
DELIMITER ;

-- Utilisation directe dans SELECT
SELECT
    product_name,
    price,
    calculate_tax(price) AS tax,
    price + calculate_tax(price) AS total_price
FROM products;
```

**Caractéristiques clés** :
- ✅ Retourne **une seule valeur** via `RETURN`
- ✅ Utilisable dans **expressions SQL** (SELECT, WHERE, ORDER BY, etc.)
- ✅ Paramètres **en lecture seule** (implicitement IN)
- ✅ Ne peut pas utiliser OUT ou INOUT
- ⚠️ Restrictions sur les modifications de données (selon caractéristiques)

---

## Fonctions vs Procédures : Différences Fondamentales

### 📊 Tableau Comparatif

| Aspect | Fonction Stockée | Procédure Stockée |
|--------|------------------|-------------------|
| **Retour de valeur** | Via `RETURN` (obligatoire) | Via paramètres OUT/INOUT |
| **Type de retour** | Scalaire unique | Multiple valeurs possibles |
| **Appel** | Dans expressions SQL | `CALL procedure()` |
| **Paramètres** | IN uniquement (implicite) | IN, OUT, INOUT |
| **Usage dans SELECT** | ✅ Oui | ❌ Non |
| **Modification de données** | ⚠️ Limitée (avec MODIFIES SQL DATA) | ✅ Complète |
| **Transactions** | ❌ Ne peut pas contrôler | ✅ START/COMMIT/ROLLBACK |
| **Jeux de résultats** | ❌ Un seul scalaire | ✅ Multiple SELECT |
| **Caractéristiques requises** | DETERMINISTIC, NO SQL, etc. | Optionnelles |

### 🎯 Quand Utiliser Une Fonction ?

**✅ Utilisez une fonction stockée pour :**
- Calculs réutilisables (TVA, remise, conversion)
- Transformations de données (formatage, normalisation)
- Validations retournant un booléen
- Extraction de valeurs depuis structures complexes (JSON, XML)
- Logique métier pure sans effets de bord

**❌ N'utilisez PAS une fonction pour :**
- Opérations complexes modifiant plusieurs tables
- Logique avec transactions explicites
- Retour de plusieurs valeurs ou jeux de résultats
- Traitement batch de données

```sql
-- ✅ BON : Calcul simple
CREATE FUNCTION get_full_name(first_name VARCHAR(50), last_name VARCHAR(50))
RETURNS VARCHAR(101)
DETERMINISTIC
BEGIN
    RETURN CONCAT(first_name, ' ', last_name);
END;

-- ❌ MAUVAIS : Modification de données (utilisez une procédure)
CREATE FUNCTION bad_update_stock(product_id INT, qty INT)
RETURNS INT
MODIFIES SQL DATA
BEGIN
    UPDATE products SET stock = stock - qty WHERE id = product_id;
    RETURN stock;  -- Anti-pattern !
END;
```

---

## Structure de Base d'une Fonction

### Syntaxe Minimale

```sql
DELIMITER $$

CREATE FUNCTION nom_fonction(param1 type1, param2 type2)
RETURNS type_retour
[DETERMINISTIC | NOT DETERMINISTIC]
[NO SQL | CONTAINS SQL | READS SQL DATA | MODIFIES SQL DATA]
BEGIN
    DECLARE variable type;
    -- Logique
    RETURN valeur;
END$$

DELIMITER ;
```

### Exemple Annoté

```sql
DELIMITER $$

CREATE FUNCTION calculate_discount(
    price DECIMAL(10,2),           -- Paramètre 1 : Prix original
    discount_percent INT            -- Paramètre 2 : Pourcentage de remise
)
RETURNS DECIMAL(10,2)               -- Type de retour
DETERMINISTIC                       -- Toujours le même résultat pour mêmes entrées
NO SQL                              -- N'accède pas à la base de données
COMMENT 'Calcule le prix après remise'
BEGIN
    DECLARE discounted_price DECIMAL(10,2);

    -- Validation : remise entre 0 et 100
    IF discount_percent < 0 OR discount_percent > 100 THEN
        RETURN price;  -- Retour du prix original si invalide
    END IF;

    -- Calcul
    SET discounted_price = price * (1 - discount_percent / 100);

    RETURN discounted_price;
END$$

DELIMITER ;

-- Utilisation
SELECT calculate_discount(100.00, 20);  -- 80.00
SELECT product_name, calculate_discount(price, 15) AS sale_price FROM products;
```

---

## Caractéristiques des Fonctions

Les fonctions stockées doivent déclarer leurs caractéristiques pour des raisons de performance et de sécurité.

### 1️⃣ DETERMINISTIC vs NOT DETERMINISTIC

**DETERMINISTIC** : La fonction retourne toujours le même résultat pour les mêmes paramètres d'entrée.

```sql
-- ✅ DETERMINISTIC : Toujours le même résultat
CREATE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    RETURN a + b;
END;

-- ✅ DETERMINISTIC : Calcul mathématique pur
CREATE FUNCTION calculate_circle_area(radius DECIMAL(10,2))
RETURNS DECIMAL(15,2)
DETERMINISTIC
NO SQL
BEGIN
    RETURN PI() * radius * radius;
END;
```

**NOT DETERMINISTIC** : Le résultat peut varier (date/heure, aléatoire, données de la base).

```sql
-- ❌ NOT DETERMINISTIC : Dépend de la date courante
CREATE FUNCTION get_age(birth_date DATE)
RETURNS INT
NOT DETERMINISTIC
NO SQL
BEGIN
    RETURN TIMESTAMPDIFF(YEAR, birth_date, CURDATE());
END;

-- ❌ NOT DETERMINISTIC : Lit depuis la base
CREATE FUNCTION get_customer_name(customer_id INT)
RETURNS VARCHAR(100)
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE name VARCHAR(100);
    SELECT customer_name INTO name FROM customers WHERE id = customer_id;
    RETURN COALESCE(name, 'Unknown');
END;
```

💡 **Important** : Si vous ne spécifiez pas DETERMINISTIC ou NOT DETERMINISTIC, MariaDB suppose NOT DETERMINISTIC par défaut (comportement sécuritaire).

### 2️⃣ Accès aux Données

#### NO SQL
La fonction **n'accède PAS** à la base de données.

```sql
CREATE FUNCTION format_phone(phone VARCHAR(20))
RETURNS VARCHAR(20)
DETERMINISTIC
NO SQL
BEGIN
    -- Seulement manipulation de chaînes
    RETURN CONCAT('(', SUBSTRING(phone, 1, 3), ') ',
                  SUBSTRING(phone, 4, 3), '-',
                  SUBSTRING(phone, 7, 4));
END;
```

#### CONTAINS SQL
La fonction contient des instructions SQL mais **ne lit ni n'écrit** de données.

```sql
CREATE FUNCTION get_current_timestamp()
RETURNS DATETIME
NOT DETERMINISTIC
CONTAINS SQL
BEGIN
    -- Utilise des fonctions SQL mais ne lit pas de tables
    RETURN NOW();
END;
```

#### READS SQL DATA
La fonction **lit des données** depuis la base de données.

```sql
CREATE FUNCTION get_product_stock(product_id INT)
RETURNS INT
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE stock_qty INT DEFAULT 0;

    SELECT stock INTO stock_qty
    FROM products
    WHERE id = product_id;

    RETURN COALESCE(stock_qty, 0);
END;
```

#### MODIFIES SQL DATA
La fonction **modifie des données** (INSERT, UPDATE, DELETE).

⚠️ **À éviter** : Les fonctions ne devraient généralement pas modifier les données.

```sql
-- ⚠️ ANTI-PATTERN : Ne faites pas cela !
CREATE FUNCTION log_access(user_id INT)
RETURNS INT
NOT DETERMINISTIC
MODIFIES SQL DATA
BEGIN
    INSERT INTO access_log (user_id, access_time)
    VALUES (user_id, NOW());

    RETURN LAST_INSERT_ID();
END;

-- ✅ MIEUX : Utilisez une procédure stockée
CREATE PROCEDURE log_access(IN user_id INT)
BEGIN
    INSERT INTO access_log (user_id, access_time)
    VALUES (user_id, NOW());
END;
```

### 📋 Tableau de Décision

| Type de Fonction | Caractéristiques Recommandées |
|------------------|------------------------------|
| **Calcul mathématique pur** | DETERMINISTIC, NO SQL |
| **Formatage de données** | DETERMINISTIC, NO SQL |
| **Date/heure courante** | NOT DETERMINISTIC, CONTAINS SQL |
| **Lecture depuis tables** | NOT DETERMINISTIC, READS SQL DATA |
| **Validation avec lookup** | NOT DETERMINISTIC, READS SQL DATA |

---

## Exemples Pratiques

### 1. Calculs Mathématiques et Financiers

```sql
DELIMITER $$

-- Calculer l'intérêt composé
CREATE FUNCTION calculate_compound_interest(
    principal DECIMAL(15,2),
    annual_rate DECIMAL(5,4),
    years INT
)
RETURNS DECIMAL(15,2)
DETERMINISTIC
NO SQL
COMMENT 'Calcule l intérêt composé : A = P(1 + r)^t'
BEGIN
    RETURN principal * POWER(1 + annual_rate, years);
END$$

-- Arrondir au centime supérieur
CREATE FUNCTION round_up_to_cent(amount DECIMAL(15,2))
RETURNS DECIMAL(15,2)
DETERMINISTIC
NO SQL
BEGIN
    RETURN CEIL(amount * 100) / 100;
END$$

-- Calculer le taux de marge
CREATE FUNCTION calculate_margin_rate(
    selling_price DECIMAL(10,2),
    cost_price DECIMAL(10,2)
)
RETURNS DECIMAL(5,2)
DETERMINISTIC
NO SQL
BEGIN
    IF cost_price = 0 THEN
        RETURN NULL;  -- Éviter division par zéro
    END IF;

    RETURN ((selling_price - cost_price) / cost_price) * 100;
END$$

DELIMITER ;

-- Utilisation
SELECT
    calculate_compound_interest(1000, 0.05, 10) AS final_amount,    -- 1628.89
    round_up_to_cent(19.994) AS rounded,                            -- 20.00
    calculate_margin_rate(150, 100) AS margin_pct;                  -- 50.00
```

### 2. Formatage et Transformation de Données

```sql
DELIMITER $$

-- Formater un nom (First letter uppercase)
CREATE FUNCTION capitalize_name(name VARCHAR(100))
RETURNS VARCHAR(100)
DETERMINISTIC
NO SQL
BEGIN
    RETURN CONCAT(
        UPPER(SUBSTRING(name, 1, 1)),
        LOWER(SUBSTRING(name, 2))
    );
END$$

-- Masquer les numéros de carte bancaire
CREATE FUNCTION mask_credit_card(card_number VARCHAR(20))
RETURNS VARCHAR(20)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE masked VARCHAR(20);
    DECLARE length INT;

    SET length = LENGTH(card_number);

    IF length < 4 THEN
        RETURN '****';
    END IF;

    -- Afficher seulement les 4 derniers chiffres
    SET masked = CONCAT(
        REPEAT('*', length - 4),
        SUBSTRING(card_number, -4)
    );

    RETURN masked;
END$$

-- Générer un slug pour URL (simple version)
CREATE FUNCTION generate_slug(text VARCHAR(255))
RETURNS VARCHAR(255)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE slug VARCHAR(255);

    -- Convertir en minuscules
    SET slug = LOWER(text);

    -- Remplacer espaces par tirets
    SET slug = REPLACE(slug, ' ', '-');

    -- Supprimer caractères spéciaux (simpliste)
    SET slug = REPLACE(slug, '''', '');
    SET slug = REPLACE(slug, '"', '');
    SET slug = REPLACE(slug, '!', '');
    SET slug = REPLACE(slug, '?', '');

    RETURN slug;
END$$

DELIMITER ;

-- Utilisation
SELECT
    capitalize_name('jean DUPONT') AS formatted_name,               -- Jean Dupont
    mask_credit_card('1234567890123456') AS masked_card,            -- ************3456
    generate_slug('MariaDB 11.8 LTS!') AS url_slug;                 -- mariadb-11.8-lts
```

### 3. Validation de Données

```sql
DELIMITER $$

-- Valider un email (version simple)
CREATE FUNCTION is_valid_email(email VARCHAR(255))
RETURNS BOOLEAN
DETERMINISTIC
NO SQL
BEGIN
    -- Validation basique : contient @ et .
    IF email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$' THEN
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;
END$$

-- Valider un numéro de SIRET français (14 chiffres)
CREATE FUNCTION is_valid_siret(siret VARCHAR(14))
RETURNS BOOLEAN
DETERMINISTIC
NO SQL
BEGIN
    -- Vérifier que c'est 14 chiffres
    IF siret REGEXP '^[0-9]{14}$' THEN
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;

    -- Note : Une validation complète inclurait l'algorithme de Luhn
END$$

-- Vérifier la force d'un mot de passe
CREATE FUNCTION check_password_strength(password VARCHAR(255))
RETURNS VARCHAR(20)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE length INT;
    DECLARE has_upper BOOLEAN DEFAULT FALSE;
    DECLARE has_lower BOOLEAN DEFAULT FALSE;
    DECLARE has_digit BOOLEAN DEFAULT FALSE;
    DECLARE has_special BOOLEAN DEFAULT FALSE;
    DECLARE score INT DEFAULT 0;

    SET length = LENGTH(password);

    -- Critères
    IF password REGEXP '[A-Z]' THEN SET has_upper = TRUE; END IF;
    IF password REGEXP '[a-z]' THEN SET has_lower = TRUE; END IF;
    IF password REGEXP '[0-9]' THEN SET has_digit = TRUE; END IF;
    IF password REGEXP '[^A-Za-z0-9]' THEN SET has_special = TRUE; END IF;

    -- Calcul du score
    IF length >= 8 THEN SET score = score + 1; END IF;
    IF length >= 12 THEN SET score = score + 1; END IF;
    IF has_upper THEN SET score = score + 1; END IF;
    IF has_lower THEN SET score = score + 1; END IF;
    IF has_digit THEN SET score = score + 1; END IF;
    IF has_special THEN SET score = score + 1; END IF;

    -- Évaluation
    CASE
        WHEN score >= 6 THEN RETURN 'STRONG';
        WHEN score >= 4 THEN RETURN 'MEDIUM';
        ELSE RETURN 'WEAK';
    END CASE;
END$$

DELIMITER ;

-- Utilisation
SELECT
    is_valid_email('user@example.com') AS email_valid,              -- 1 (TRUE)
    is_valid_email('invalid-email') AS email_invalid,               -- 0 (FALSE)
    is_valid_siret('12345678901234') AS siret_valid,                -- 1 (TRUE)
    check_password_strength('Pass123!@#') AS pwd_strength;          -- STRONG
```

### 4. Fonctions avec Lecture de Données

```sql
DELIMITER $$

-- Obtenir le nom complet d'un client
CREATE FUNCTION get_customer_fullname(customer_id INT)
RETURNS VARCHAR(200)
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE fullname VARCHAR(200);

    SELECT CONCAT(first_name, ' ', last_name)
    INTO fullname
    FROM customers
    WHERE id = customer_id;

    RETURN COALESCE(fullname, 'Unknown Customer');
END$$

-- Calculer le total des commandes d'un client
CREATE FUNCTION get_customer_total_orders(customer_id INT)
RETURNS DECIMAL(15,2)
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE total DECIMAL(15,2) DEFAULT 0.00;

    SELECT COALESCE(SUM(total_amount), 0.00)
    INTO total
    FROM orders
    WHERE customer_id = customer_id
    AND status = 'completed';

    RETURN total;
END$$

-- Obtenir le niveau de fidélité d'un client
CREATE FUNCTION get_loyalty_tier(customer_id INT)
RETURNS VARCHAR(20)
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE total_spent DECIMAL(15,2);

    -- Récupérer le total dépensé
    SET total_spent = get_customer_total_orders(customer_id);

    -- Déterminer le tier
    IF total_spent >= 10000 THEN
        RETURN 'PLATINUM';
    ELSEIF total_spent >= 5000 THEN
        RETURN 'GOLD';
    ELSEIF total_spent >= 1000 THEN
        RETURN 'SILVER';
    ELSE
        RETURN 'BRONZE';
    END IF;
END$$

-- Vérifier si un produit est en stock
CREATE FUNCTION is_in_stock(product_id INT, required_qty INT)
RETURNS BOOLEAN
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE available_stock INT;

    SELECT stock INTO available_stock
    FROM products
    WHERE id = product_id;

    IF available_stock IS NULL THEN
        RETURN FALSE;
    END IF;

    RETURN available_stock >= required_qty;
END$$

DELIMITER ;

-- Utilisation dans des requêtes complexes
SELECT
    c.id,
    get_customer_fullname(c.id) AS customer_name,
    get_customer_total_orders(c.id) AS total_spent,
    get_loyalty_tier(c.id) AS tier
FROM customers c
WHERE c.is_active = TRUE
ORDER BY get_customer_total_orders(c.id) DESC
LIMIT 10;

-- Utilisation dans WHERE
SELECT
    product_id,
    product_name,
    stock
FROM products
WHERE is_in_stock(id, 10) = TRUE;
```

### 5. Extraction de Données JSON

```sql
DELIMITER $$

-- Extraire une valeur depuis un champ JSON
CREATE FUNCTION json_get_value(
    json_data JSON,
    json_path VARCHAR(255)
)
RETURNS VARCHAR(500)
NOT DETERMINISTIC
CONTAINS SQL
BEGIN
    DECLARE extracted_value VARCHAR(500);

    SET extracted_value = JSON_UNQUOTE(JSON_EXTRACT(json_data, json_path));

    RETURN extracted_value;
END$$

-- Compter les éléments dans un tableau JSON
CREATE FUNCTION json_array_count(json_array JSON)
RETURNS INT
DETERMINISTIC
CONTAINS SQL
BEGIN
    IF JSON_TYPE(json_array) != 'ARRAY' THEN
        RETURN 0;
    END IF;

    RETURN JSON_LENGTH(json_array);
END$$

-- Vérifier si une clé existe dans un objet JSON
CREATE FUNCTION json_has_key(
    json_data JSON,
    key_name VARCHAR(255)
)
RETURNS BOOLEAN
DETERMINISTIC
CONTAINS SQL
BEGIN
    RETURN JSON_EXTRACT(json_data, CONCAT('$.', key_name)) IS NOT NULL;
END$$

DELIMITER ;

-- Utilisation
SELECT
    json_get_value('{"name":"John","age":30}', '$.name') AS name,       -- John
    json_array_count('[1,2,3,4,5]') AS count,                           -- 5
    json_has_key('{"a":1,"b":2}', 'a') AS has_key_a;                    -- 1 (TRUE)
```

---

## Utilisation des Fonctions dans les Requêtes

### Dans SELECT

```sql
-- Calculs sur colonnes
SELECT
    product_name,
    price AS original_price,
    calculate_discount(price, 20) AS discounted_price,
    calculate_tax(calculate_discount(price, 20)) AS tax_on_discounted
FROM products;

-- Formatage de données
SELECT
    customer_id,
    capitalize_name(first_name) AS first_name,
    capitalize_name(last_name) AS last_name,
    mask_credit_card(credit_card) AS masked_card
FROM customers;
```

### Dans WHERE

```sql
-- Filtrer par fonction
SELECT *
FROM customers
WHERE is_valid_email(email) = TRUE
AND get_loyalty_tier(id) IN ('GOLD', 'PLATINUM');

-- Filtrer par calcul
SELECT *
FROM products
WHERE calculate_margin_rate(selling_price, cost_price) > 30;
```

### Dans ORDER BY

```sql
-- Trier par fonction
SELECT
    customer_id,
    get_customer_fullname(id) AS name,
    get_customer_total_orders(id) AS total_spent
FROM customers
ORDER BY get_customer_total_orders(id) DESC;
```

### Dans GROUP BY et HAVING

```sql
-- Grouper par fonction
SELECT
    get_loyalty_tier(customer_id) AS tier,
    COUNT(*) AS customer_count,
    AVG(total_amount) AS avg_order_value
FROM orders
GROUP BY get_loyalty_tier(customer_id)
HAVING AVG(total_amount) > 100;
```

### Dans JOIN

```sql
-- Utiliser dans une condition de jointure
SELECT
    o.id,
    o.customer_id,
    get_customer_fullname(o.customer_id) AS customer_name
FROM orders o
WHERE is_in_stock(o.product_id, o.quantity) = TRUE;
```

---

## Bonnes Pratiques

### 1. ✅ Toujours Spécifier les Caractéristiques

```sql
-- ❌ MAUVAIS : Caractéristiques manquantes
CREATE FUNCTION bad_function(x INT)
RETURNS INT
BEGIN
    RETURN x * 2;
END;

-- ✅ BON : Caractéristiques explicites
CREATE FUNCTION good_function(x INT)
RETURNS INT
DETERMINISTIC
NO SQL
COMMENT 'Double the input value'
BEGIN
    RETURN x * 2;
END;
```

### 2. ✅ Gestion des Valeurs NULL

```sql
-- ✅ BON : Gestion explicite de NULL
CREATE FUNCTION safe_divide(numerator DECIMAL(10,2), denominator DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
BEGIN
    -- Gérer NULL
    IF numerator IS NULL OR denominator IS NULL THEN
        RETURN NULL;
    END IF;

    -- Gérer division par zéro
    IF denominator = 0 THEN
        RETURN NULL;
    END IF;

    RETURN numerator / denominator;
END;
```

### 3. ✅ Fonctions Simples et Ciblées (Single Responsibility)

```sql
-- ✅ BON : Fonction simple avec un seul objectif
CREATE FUNCTION calculate_vat(amount DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
BEGIN
    RETURN amount * 0.20;
END;

-- ❌ MAUVAIS : Fonction qui fait trop de choses
CREATE FUNCTION bad_complex_function(
    amount DECIMAL(10,2),
    customer_id INT,
    apply_loyalty BOOLEAN
)
RETURNS DECIMAL(10,2)
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    -- Trop complexe, devrait être plusieurs fonctions
    DECLARE discount DECIMAL(5,2);
    DECLARE vat DECIMAL(10,2);

    -- Logique complexe imbriquée...
    -- ...
END;
```

### 4. ✅ Éviter les Fonctions qui Modifient les Données

```sql
-- ❌ MAUVAIS : Fonction avec effets de bord
CREATE FUNCTION bad_increment_counter(product_id INT)
RETURNS INT
NOT DETERMINISTIC
MODIFIES SQL DATA
BEGIN
    UPDATE products SET view_count = view_count + 1 WHERE id = product_id;
    RETURN view_count;
END;

-- ✅ BON : Utiliser une procédure à la place
CREATE PROCEDURE increment_counter(IN product_id INT)
BEGIN
    UPDATE products SET view_count = view_count + 1 WHERE id = product_id;
END;
```

### 5. ✅ Documentation et Nommage

```sql
-- ✅ BON : Nom descriptif et documentation
CREATE FUNCTION calculate_shipping_cost(
    weight_kg DECIMAL(8,2),
    distance_km INT
)
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
COMMENT 'Calcule les frais de port : base 5€ + 0.50€/kg + 0.10€/km'
BEGIN
    DECLARE base_cost DECIMAL(10,2) DEFAULT 5.00;
    DECLARE cost_per_kg DECIMAL(10,2) DEFAULT 0.50;
    DECLARE cost_per_km DECIMAL(10,2) DEFAULT 0.10;

    RETURN base_cost + (weight_kg * cost_per_kg) + (distance_km * cost_per_km);
END;
```

### 6. ✅ Performance : Éviter les Appels Répétés

```sql
-- ❌ INEFFICACE : Fonction appelée plusieurs fois
SELECT
    customer_id,
    get_loyalty_tier(customer_id) AS tier,  -- Appel 1
    CASE get_loyalty_tier(customer_id)      -- Appel 2 (même calcul!)
        WHEN 'PLATINUM' THEN 'Premium'
        ELSE 'Standard'
    END AS service_level
FROM customers;

-- ✅ EFFICACE : Calculer une seule fois
SELECT
    customer_id,
    tier,
    CASE tier
        WHEN 'PLATINUM' THEN 'Premium'
        ELSE 'Standard'
    END AS service_level
FROM (
    SELECT
        customer_id,
        get_loyalty_tier(customer_id) AS tier
    FROM customers
) AS customer_tiers;
```

### 7. ✅ Testabilité

```sql
-- ✅ BON : Fonction pure, facile à tester
CREATE FUNCTION add_vat(amount DECIMAL(10,2), vat_rate DECIMAL(5,4))
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
BEGIN
    RETURN amount * (1 + vat_rate);
END;

-- Tests simples
SELECT
    add_vat(100, 0.20) = 120 AS test1_passed,      -- TRUE
    add_vat(50, 0.10) = 55 AS test2_passed,        -- TRUE
    add_vat(0, 0.20) = 0 AS test3_passed;          -- TRUE
```

---

## Limitations et Restrictions

### ❌ Ce qui n'est PAS Possible

```sql
-- 1. PAS de paramètres OUT ou INOUT
-- ❌ IMPOSSIBLE
CREATE FUNCTION impossible_out(IN x INT, OUT y INT)  -- Erreur !
RETURNS INT
BEGIN
    SET y = x * 2;
    RETURN x;
END;

-- 2. PAS de CALL vers une procédure
-- ❌ IMPOSSIBLE
CREATE FUNCTION impossible_call()
RETURNS INT
BEGIN
    CALL some_procedure();  -- Erreur !
    RETURN 1;
END;

-- 3. PAS de contrôle transactionnel
-- ❌ IMPOSSIBLE
CREATE FUNCTION impossible_transaction()
RETURNS INT
BEGIN
    START TRANSACTION;      -- Erreur !
    -- ...
    COMMIT;                 -- Erreur !
    RETURN 1;
END;

-- 4. PAS de retour de jeux de résultats multiples
-- ❌ IMPOSSIBLE
CREATE FUNCTION impossible_resultsets()
RETURNS INT
BEGIN
    SELECT * FROM table1;   -- Erreur si retourne un resultset
    SELECT * FROM table2;   -- Erreur
    RETURN 1;
END;
```

### ⚠️ Restrictions Importantes

1. **Binary Logging** : Les fonctions avec `READS SQL DATA` ou `MODIFIES SQL DATA` doivent être `DETERMINISTIC` ou `NOT DETERMINISTIC` doit être explicitement spécifié si `log_bin_trust_function_creators = 0`.

2. **Réplication** : Les fonctions non-déterministes peuvent causer des problèmes de réplication.

3. **Performance** : Les fonctions appelées dans un WHERE ou JOIN peuvent impacter les performances si la table est grande.

---

## Gestion et Maintenance

### Lister les Fonctions

```sql
-- Toutes les fonctions de la base courante
SHOW FUNCTION STATUS WHERE Db = DATABASE();

-- Afficher le code source
SHOW CREATE FUNCTION nom_fonction;

-- Informations détaillées
SELECT
    ROUTINE_NAME,
    ROUTINE_TYPE,
    DATA_TYPE,
    IS_DETERMINISTIC,
    SQL_DATA_ACCESS,
    SECURITY_TYPE,
    ROUTINE_COMMENT
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = 'mydb'
AND ROUTINE_TYPE = 'FUNCTION';
```

### Modifier une Fonction

```sql
-- MariaDB 10.3+ : CREATE OR REPLACE
CREATE OR REPLACE FUNCTION my_function(x INT)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    RETURN x * 3;  -- Nouvelle version
END;

-- Alternative : DROP puis CREATE
DROP FUNCTION IF EXISTS my_function;
CREATE FUNCTION my_function(x INT)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    RETURN x * 3;
END;
```

### Supprimer une Fonction

```sql
DROP FUNCTION IF EXISTS nom_fonction;
```

### Privilèges

```sql
-- Créer des fonctions
GRANT CREATE ROUTINE ON mydb.* TO 'developer'@'%';

-- Exécuter une fonction
GRANT EXECUTE ON FUNCTION mydb.calculate_tax TO 'app_user'@'%';

-- Modifier
GRANT ALTER ROUTINE ON mydb.* TO 'developer'@'%';
```

---

## ⚠️ Erreurs Courantes

### 1. Oublier RETURN

```sql
-- ❌ ERREUR : Pas de RETURN
CREATE FUNCTION bad_no_return(x INT)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    DECLARE result INT;
    SET result = x * 2;
    -- Manque RETURN result;
END;

-- ✅ CORRECT
CREATE FUNCTION good_with_return(x INT)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    DECLARE result INT;
    SET result = x * 2;
    RETURN result;
END;
```

### 2. Type de Retour Incorrect

```sql
-- ❌ ERREUR : Type incompatible
CREATE FUNCTION bad_type_mismatch()
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    RETURN 'Hello';  -- String retournée au lieu d'INT
END;

-- ✅ CORRECT
CREATE FUNCTION good_type_match()
RETURNS VARCHAR(10)
DETERMINISTIC
NO SQL
BEGIN
    RETURN 'Hello';
END;
```

### 3. Caractéristiques Manquantes en Réplication

```sql
-- ⚠️ PROBLÈME : Peut échouer si log_bin_trust_function_creators = 0
CREATE FUNCTION risky_function(x INT)
RETURNS INT
-- Manque DETERMINISTIC ou NOT DETERMINISTIC
BEGIN
    RETURN x * 2;
END;

-- ✅ SOLUTION : Toujours spécifier
CREATE FUNCTION safe_function(x INT)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    RETURN x * 2;
END;
```

---

## 🔄 Comparaison : MySQL vs MariaDB

Les fonctions stockées sont très similaires entre MySQL et MariaDB, avec quelques nuances :

| Fonctionnalité | MariaDB | MySQL |
|----------------|---------|-------|
| **Syntaxe de base** | ✅ Identique | ✅ Identique |
| **CREATE OR REPLACE** | ✅ Oui (10.3+) | ⚠️ Non (avant 8.0.29) |
| **Caractéristiques** | ✅ Identiques | ✅ Identiques |
| **DETERMINISTIC** | ✅ Oui | ✅ Oui |
| **JSON support** | ✅ Oui | ✅ Oui |

💡 Les fonctions stockées écrites pour MySQL fonctionneront généralement sans modification dans MariaDB.

---

## ✅ Points clés à retenir

- 🎯 **Usage** : Les fonctions retournent UNE valeur scalaire et s'utilisent dans les expressions SQL
- 🔄 **vs Procédures** : Pas de OUT/INOUT, pas de transactions, pas de CALL
- 📊 **Caractéristiques** : DETERMINISTIC, NO SQL, READS SQL DATA doivent être spécifiées
- ⚡ **Performance** : Les fonctions DETERMINISTIC peuvent être optimisées par le moteur
- 🛡️ **Sécurité** : Évitez MODIFIES SQL DATA dans les fonctions
- 📝 **Bonnes pratiques** : Fonctions simples, gestion de NULL, documentation
- 🚫 **Limitations** : Pas de contrôle transactionnel, paramètres IN uniquement
- ✅ **Tests** : Les fonctions pures (NO SQL) sont faciles à tester

---

## 🔗 Ressources et références

- [📖 CREATE FUNCTION - MariaDB Documentation](https://mariadb.com/kb/en/create-function/)
- [📖 Stored Functions - MariaDB Documentation](https://mariadb.com/kb/en/stored-functions/)
- [📖 Function Characteristics](https://mariadb.com/kb/en/stored-routine-characteristics/)
- [📖 Stored Function Limitations](https://mariadb.com/kb/en/stored-function-limitations/)
- [📝 Best Practices for Stored Functions](https://mariadb.com/kb/en/stored-function-best-practices/)

---

## ➡️ Section suivante

**8.2.1 Syntaxe CREATE FUNCTION** : Détails exhaustifs de la syntaxe, options avancées, et exemples de toutes les combinaisons de caractéristiques possibles.

---


⏭️ [Syntaxe CREATE FUNCTION](/08-programmation-cote-serveur/02.1-syntaxe-create-function.md)

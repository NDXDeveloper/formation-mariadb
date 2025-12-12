🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1 Procédures Stockées (Stored Procedures)

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : SQL avancé (chapitres 3-4), Transactions (chapitre 6), Introduction à la programmation serveur (8.0)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et les avantages des procédures stockées dans MariaDB
- **Identifier** les cas d'usage appropriés pour les procédures stockées
- **Distinguer** les différents types de paramètres (IN, OUT, INOUT)
- **Créer** des procédures stockées simples et complexes
- **Gérer** les transactions au sein des procédures
- **Appliquer** les bonnes pratiques de développement et de sécurité
- **Optimiser** les performances des procédures en production

---

## Introduction

Les **procédures stockées** (stored procedures) sont des blocs de code SQL précompilés et stockés dans le serveur de base de données. Elles constituent l'un des piliers de la programmation côté serveur et permettent d'encapsuler une logique métier complexe.

### 🔍 Qu'est-ce qu'une procédure stockée ?

Une procédure stockée est essentiellement :
- Un **programme SQL** avec un nom unique
- **Stocké dans la base de données** (schéma `mysql.proc`)
- **Compilé une seule fois** et réutilisable
- Capable d'accepter des **paramètres** (entrée/sortie)
- Exécutable via la commande `CALL`

```sql
-- Exemple minimal
DELIMITER $$
CREATE PROCEDURE hello_world()
BEGIN
    SELECT 'Hello, World!' AS message;
END$$
DELIMITER ;

-- Appel
CALL hello_world();
```

### 📊 Procédures vs Autres Objets

| Caractéristique | Procédure | Fonction | Trigger |
|----------------|-----------|----------|---------|
| **Retour de valeur** | Via OUT/INOUT | RETURN valeur | N/A |
| **Modification données** | ✅ Oui | ⚠️ Limité | ✅ Oui |
| **Appel** | `CALL proc()` | Dans SELECT | Automatique |
| **Paramètres** | IN/OUT/INOUT | IN seulement | N/A |
| **Transactions** | ✅ Contrôle total | ❌ Non | ⚠️ Contexte parent |
| **Usage dans SELECT** | ❌ Non | ✅ Oui | ❌ Non |

---

## Pourquoi Utiliser les Procédures Stockées ?

### ✅ Avantages

#### 1. **Performance**

```sql
-- ❌ Application : 3 requêtes = 3 allers-retours réseau
-- SELECT * FROM users WHERE id = 1;
-- SELECT * FROM orders WHERE user_id = 1;
-- SELECT COUNT(*) FROM order_items WHERE order_id IN (...);

-- ✅ Procédure : 1 seul aller-retour
DELIMITER $$
CREATE PROCEDURE get_user_summary(IN p_user_id INT)
BEGIN
    -- Toute la logique côté serveur
    SELECT * FROM users WHERE id = p_user_id;
    SELECT * FROM orders WHERE user_id = p_user_id;
    SELECT COUNT(*) AS total_items
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    WHERE o.user_id = p_user_id;
END$$
DELIMITER ;
```

**Gains** :
- 🚀 Réduction latence réseau (1 appel vs N appels)
- 🚀 Plan d'exécution en cache (precompiled)
- 🚀 Moins de parsing SQL

#### 2. **Sécurité**

```sql
-- Encapsulation : L'utilisateur n'accède PAS directement aux tables
DELIMITER $$
CREATE PROCEDURE transfer_money(
    IN p_from_account INT,
    IN p_to_account INT,
    IN p_amount DECIMAL(10,2)
)
BEGIN
    -- Validation des règles métier
    -- Transactions ACID
    -- Logs d'audit
    -- L'application ne peut PAS bypass cette logique
END$$
DELIMITER ;

-- Privilèges granulaires
GRANT EXECUTE ON PROCEDURE mydb.transfer_money TO 'app_user'@'%';
REVOKE ALL ON mydb.accounts FROM 'app_user'@'%';  -- Pas d'accès direct
```

#### 3. **Réutilisabilité**

```sql
-- Une seule définition, multiple applications
DELIMITER $$
CREATE PROCEDURE calculate_invoice_total(
    IN p_invoice_id INT,
    OUT p_subtotal DECIMAL(10,2),
    OUT p_tax DECIMAL(10,2),
    OUT p_total DECIMAL(10,2)
)
BEGIN
    -- Logique métier complexe centralisée
    SELECT
        SUM(quantity * unit_price) INTO p_subtotal
    FROM invoice_items
    WHERE invoice_id = p_invoice_id;

    SET p_tax = p_subtotal * 0.20;  -- TVA 20%
    SET p_total = p_subtotal + p_tax;
END$$
DELIMITER ;

-- Utilisable depuis PHP, Python, Java, Node.js, etc.
```

#### 4. **Maintenance Centralisée**

```sql
-- Modification de la logique métier = 1 seul endroit
DELIMITER $$
CREATE OR REPLACE PROCEDURE apply_discount(
    IN p_customer_id INT,
    IN p_amount DECIMAL(10,2),
    OUT p_final_amount DECIMAL(10,2)
)
BEGIN
    DECLARE v_is_vip BOOLEAN;
    DECLARE v_discount DECIMAL(5,2);

    -- Règle métier : si client VIP, 10% de réduction
    SELECT is_vip INTO v_is_vip FROM customers WHERE id = p_customer_id;

    IF v_is_vip THEN
        SET v_discount = 0.10;
    ELSE
        SET v_discount = 0.00;
    END IF;

    SET p_final_amount = p_amount * (1 - v_discount);
END$$
DELIMITER ;

-- Changement de règle ? Modifier 1 fois, tous les clients en bénéficient
```

### ⚠️ Inconvénients et Limites

| Inconvénient | Impact | Mitigation |
|--------------|--------|------------|
| **Portabilité** | Code MariaDB-specific | Abstraire via ORM/DAL |
| **Debugging** | Outils limités | Logging, tables debug |
| **Version control** | Difficile dans Git | Scripts de migration |
| **Testing** | Complexe à tester | Frameworks de tests DB |
| **Scalabilité** | Charge CPU serveur | Monitoring, load balancing |
| **Complexité** | Maintenance difficile | Documentation, conventions |

💡 **Conseil** : Utilisez les procédures stockées pour la logique proche des données (transactions, calculs complexes), mais gardez la logique métier évolutive dans l'application.

---

## Types de Paramètres

MariaDB supporte trois types de paramètres pour les procédures stockées :

### 1️⃣ Paramètres IN (Entrée)

**Définition** : Valeur passée À la procédure, en lecture seule.

```sql
DELIMITER $$
CREATE PROCEDURE get_customer_info(IN p_customer_id INT)
BEGIN
    SELECT
        id,
        name,
        email,
        created_at
    FROM customers
    WHERE id = p_customer_id;

    -- p_customer_id est en lecture seule
    -- SET p_customer_id = 999;  ❌ Sans effet à l'extérieur
END$$
DELIMITER ;

-- Utilisation
CALL get_customer_info(42);
```

**Caractéristiques** :
- ✅ Valeur passée par l'appelant
- ✅ Peut être utilisée dans la procédure
- ❌ Modification locale uniquement (pas d'effet à l'extérieur)
- 📌 **Type par défaut** si non spécifié

### 2️⃣ Paramètres OUT (Sortie)

**Définition** : Valeur retournée PAR la procédure.

```sql
DELIMITER $$
CREATE PROCEDURE get_customer_count(OUT p_total INT)
BEGIN
    -- La valeur d'entrée de p_total est ignorée
    SELECT COUNT(*) INTO p_total FROM customers;
END$$
DELIMITER ;

-- Utilisation
SET @total_customers = 0;
CALL get_customer_count(@total_customers);
SELECT @total_customers;  -- Affiche le résultat
```

**Caractéristiques** :
- ✅ Valeur retournée à l'appelant
- ⚠️ Valeur initiale ignorée
- ✅ Doit être une variable de session (`@variable`)
- 📌 Utile pour retourner des résultats calculés

### 3️⃣ Paramètres INOUT (Entrée/Sortie)

**Définition** : Valeur passée À et retournée PAR la procédure.

```sql
DELIMITER $$
CREATE PROCEDURE apply_vat(INOUT p_amount DECIMAL(10,2))
BEGIN
    -- Lecture de la valeur d'entrée
    -- Modification et retour
    SET p_amount = p_amount * 1.20;  -- TVA 20%
END$$
DELIMITER ;

-- Utilisation
SET @price = 100.00;
CALL apply_vat(@price);
SELECT @price;  -- Affiche 120.00
```

**Caractéristiques** :
- ✅ Lecture ET écriture
- ✅ Valeur modifiée retournée
- ⚠️ Plus complexe à comprendre/maintenir
- 📌 Utile pour transformations de valeurs

### 📋 Comparaison des Types de Paramètres

```sql
DELIMITER $$
CREATE PROCEDURE demo_parameters(
    IN p_input INT,           -- Lecture seule
    OUT p_output INT,         -- Écriture seule
    INOUT p_both INT          -- Lecture + Écriture
)
BEGIN
    -- Afficher les valeurs d'entrée
    SELECT
        p_input AS input_value,
        p_output AS output_value_initial,  -- NULL ou undefined
        p_both AS both_value_initial;

    -- Modifications
    -- SET p_input = 999;     -- Possible mais sans effet externe
    SET p_output = 200;        -- Définir la sortie
    SET p_both = p_both * 2;   -- Modifier la valeur entrée/sortie
END$$
DELIMITER ;

-- Test
SET @in = 10;
SET @out = 0;
SET @both = 5;

CALL demo_parameters(@in, @out, @both);

SELECT @in AS input,      -- 10 (inchangé)
       @out AS output,    -- 200 (défini)
       @both AS both;     -- 10 (5 * 2)
```

---

## Anatomie d'une Procédure Stockée

### Structure de Base

```sql
DELIMITER $$

CREATE [OR REPLACE] PROCEDURE nom_procedure(
    [IN | OUT | INOUT] param1 datatype,
    [IN | OUT | INOUT] param2 datatype
)
[SQL SECURITY {DEFINER | INVOKER}]
[COMMENT 'description']
BEGIN
    -- Déclarations de variables (en premier)
    DECLARE variable_name datatype [DEFAULT value];

    -- Déclarations de handlers
    DECLARE handler_type HANDLER FOR condition_value statement;

    -- Corps de la procédure (instructions SQL)
    -- ...

END$$

DELIMITER ;
```

### Exemple Annoté

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE process_order(
    IN p_customer_id INT,           -- ID du client
    IN p_product_id INT,            -- ID du produit
    IN p_quantity INT,              -- Quantité commandée
    OUT p_order_id INT,             -- ID de la commande créée
    OUT p_status VARCHAR(50)        -- Statut de l'opération
)
SQL SECURITY INVOKER                -- Droits de l'appelant
COMMENT 'Traite une nouvelle commande avec validation'
BEGIN
    -- 1. DÉCLARATIONS DE VARIABLES
    DECLARE v_stock INT DEFAULT 0;
    DECLARE v_price DECIMAL(10,2);
    DECLARE v_customer_exists BOOLEAN DEFAULT FALSE;

    -- 2. HANDLER D'ERREUR
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- En cas d'erreur, rollback automatique
        ROLLBACK;
        SET p_status = 'ERROR';
        SET p_order_id = NULL;
    END;

    -- 3. LOGIQUE MÉTIER

    -- Vérifier que le client existe
    SELECT COUNT(*) > 0 INTO v_customer_exists
    FROM customers
    WHERE id = p_customer_id;

    IF NOT v_customer_exists THEN
        SET p_status = 'CUSTOMER_NOT_FOUND';
        SET p_order_id = NULL;
    ELSE
        -- Démarrer la transaction
        START TRANSACTION;

        -- Vérifier le stock avec lock
        SELECT stock, price INTO v_stock, v_price
        FROM products
        WHERE id = p_product_id
        FOR UPDATE;  -- Lock pessimiste

        IF v_stock < p_quantity THEN
            -- Stock insuffisant
            ROLLBACK;
            SET p_status = 'INSUFFICIENT_STOCK';
            SET p_order_id = NULL;
        ELSE
            -- Créer la commande
            INSERT INTO orders (customer_id, total_amount, status)
            VALUES (p_customer_id, p_quantity * v_price, 'pending');

            SET p_order_id = LAST_INSERT_ID();

            -- Décrémenter le stock
            UPDATE products
            SET stock = stock - p_quantity
            WHERE id = p_product_id;

            -- Commit transaction
            COMMIT;
            SET p_status = 'SUCCESS';
        END IF;
    END IF;
END$$

DELIMITER ;

-- Utilisation
CALL process_order(1, 10, 5, @order_id, @status);
SELECT @order_id, @status;
```

---

## Gestion des Transactions

### Contrôle Transactionnel

Les procédures stockées permettent un contrôle fin des transactions :

```sql
DELIMITER $$

CREATE PROCEDURE transfer_funds(
    IN p_from_account INT,
    IN p_to_account INT,
    IN p_amount DECIMAL(10,2),
    OUT p_success BOOLEAN
)
BEGIN
    DECLARE v_balance DECIMAL(10,2);

    -- Handler pour rollback automatique
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
    END;

    -- Transaction ACID
    START TRANSACTION;

    -- 1. Vérifier le solde avec lock
    SELECT balance INTO v_balance
    FROM accounts
    WHERE id = p_from_account
    FOR UPDATE;

    IF v_balance < p_amount THEN
        ROLLBACK;
        SET p_success = FALSE;
    ELSE
        -- 2. Débiter le compte source
        UPDATE accounts
        SET balance = balance - p_amount
        WHERE id = p_from_account;

        -- 3. Créditer le compte destination
        UPDATE accounts
        SET balance = balance + p_amount
        WHERE id = p_to_account;

        -- 4. Logger la transaction
        INSERT INTO transactions (from_account, to_account, amount)
        VALUES (p_from_account, p_to_account, p_amount);

        COMMIT;
        SET p_success = TRUE;
    END IF;
END$$

DELIMITER ;
```

### Niveaux d'Isolation

```sql
DELIMITER $$

CREATE PROCEDURE batch_update_prices()
BEGIN
    -- Définir le niveau d'isolation pour cette transaction
    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

    START TRANSACTION;

    -- Opérations critiques
    UPDATE products
    SET price = price * 1.10
    WHERE category = 'electronics';

    COMMIT;
END$$

DELIMITER ;
```

---

## Variables Locales

### Déclaration et Utilisation

```sql
DELIMITER $$

CREATE PROCEDURE calculate_discount(
    IN p_amount DECIMAL(10,2),
    IN p_customer_type VARCHAR(20),
    OUT p_final_amount DECIMAL(10,2)
)
BEGIN
    -- Déclarations (TOUJOURS au début du BEGIN)
    DECLARE v_discount_rate DECIMAL(5,2) DEFAULT 0.00;
    DECLARE v_discount_amount DECIMAL(10,2);
    DECLARE v_is_vip BOOLEAN DEFAULT FALSE;

    -- Logique de calcul
    CASE p_customer_type
        WHEN 'VIP' THEN
            SET v_discount_rate = 0.20;  -- 20%
            SET v_is_vip = TRUE;
        WHEN 'GOLD' THEN
            SET v_discount_rate = 0.15;  -- 15%
        WHEN 'SILVER' THEN
            SET v_discount_rate = 0.10;  -- 10%
        ELSE
            SET v_discount_rate = 0.05;  -- 5%
    END CASE;

    -- Calcul du montant de réduction
    SET v_discount_amount = p_amount * v_discount_rate;

    -- Bonus VIP : réduction supplémentaire de 10€ si montant > 100€
    IF v_is_vip AND p_amount > 100 THEN
        SET v_discount_amount = v_discount_amount + 10.00;
    END IF;

    -- Montant final
    SET p_final_amount = p_amount - v_discount_amount;

    -- S'assurer que le montant n'est pas négatif
    IF p_final_amount < 0 THEN
        SET p_final_amount = 0;
    END IF;
END$$

DELIMITER ;

-- Test
CALL calculate_discount(150.00, 'VIP', @final);
SELECT @final;  -- 150 - (150*0.20) - 10 = 110
```

### Portée des Variables

```sql
DELIMITER $$

CREATE PROCEDURE demo_variable_scope()
BEGIN
    DECLARE v_outer INT DEFAULT 10;

    -- Bloc externe
    BEGIN
        DECLARE v_inner INT DEFAULT 20;
        SELECT v_outer, v_inner;  -- 10, 20
    END;

    -- v_inner n'existe plus ici
    -- SELECT v_inner;  ❌ Erreur
    SELECT v_outer;  -- ✅ 10
END$$

DELIMITER ;
```

---

## Exemples Pratiques

### 1. Procédure CRUD Simple

```sql
DELIMITER $$

-- CREATE
CREATE PROCEDURE create_customer(
    IN p_name VARCHAR(100),
    IN p_email VARCHAR(100),
    OUT p_customer_id INT,
    OUT p_message VARCHAR(255)
)
BEGIN
    DECLARE v_email_exists BOOLEAN;

    -- Vérifier si l'email existe déjà
    SELECT COUNT(*) > 0 INTO v_email_exists
    FROM customers
    WHERE email = p_email;

    IF v_email_exists THEN
        SET p_customer_id = NULL;
        SET p_message = 'Email already exists';
    ELSE
        INSERT INTO customers (name, email, created_at)
        VALUES (p_name, p_email, NOW());

        SET p_customer_id = LAST_INSERT_ID();
        SET p_message = 'Customer created successfully';
    END IF;
END$$

-- READ
CREATE PROCEDURE get_customer(IN p_customer_id INT)
BEGIN
    SELECT
        id,
        name,
        email,
        created_at,
        updated_at
    FROM customers
    WHERE id = p_customer_id;
END$$

-- UPDATE
CREATE PROCEDURE update_customer(
    IN p_customer_id INT,
    IN p_name VARCHAR(100),
    IN p_email VARCHAR(100),
    OUT p_affected_rows INT
)
BEGIN
    UPDATE customers
    SET
        name = p_name,
        email = p_email,
        updated_at = NOW()
    WHERE id = p_customer_id;

    SET p_affected_rows = ROW_COUNT();
END$$

-- DELETE (soft delete)
CREATE PROCEDURE delete_customer(
    IN p_customer_id INT,
    OUT p_success BOOLEAN
)
BEGIN
    UPDATE customers
    SET
        deleted_at = NOW(),
        is_active = FALSE
    WHERE id = p_customer_id;

    SET p_success = ROW_COUNT() > 0;
END$$

DELIMITER ;
```

### 2. Procédure avec Logique Métier Complexe

```sql
DELIMITER $$

CREATE PROCEDURE apply_loyalty_points(
    IN p_customer_id INT,
    IN p_order_amount DECIMAL(10,2),
    OUT p_points_earned INT,
    OUT p_new_tier VARCHAR(20)
)
BEGIN
    DECLARE v_current_points INT;
    DECLARE v_current_tier VARCHAR(20);
    DECLARE v_total_spending DECIMAL(10,2);

    -- Récupérer les données actuelles du client
    SELECT
        loyalty_points,
        loyalty_tier,
        total_spending
    INTO
        v_current_points,
        v_current_tier,
        v_total_spending
    FROM customers
    WHERE id = p_customer_id;

    -- Calculer les points gagnés (1 point par euro dépensé)
    SET p_points_earned = FLOOR(p_order_amount);

    -- Bonus selon le tier actuel
    CASE v_current_tier
        WHEN 'PLATINUM' THEN
            SET p_points_earned = p_points_earned * 3;  -- Triple points
        WHEN 'GOLD' THEN
            SET p_points_earned = p_points_earned * 2;  -- Double points
        WHEN 'SILVER' THEN
            SET p_points_earned = FLOOR(p_points_earned * 1.5);  -- 50% bonus
        ELSE
            -- BRONZE ou NULL : points normaux
            NULL;
    END CASE;

    -- Mettre à jour les points
    SET v_current_points = v_current_points + p_points_earned;
    SET v_total_spending = v_total_spending + p_order_amount;

    -- Déterminer le nouveau tier basé sur le total dépensé
    IF v_total_spending >= 10000 THEN
        SET p_new_tier = 'PLATINUM';
    ELSEIF v_total_spending >= 5000 THEN
        SET p_new_tier = 'GOLD';
    ELSEIF v_total_spending >= 1000 THEN
        SET p_new_tier = 'SILVER';
    ELSE
        SET p_new_tier = 'BRONZE';
    END IF;

    -- Sauvegarder dans la base
    UPDATE customers
    SET
        loyalty_points = v_current_points,
        loyalty_tier = p_new_tier,
        total_spending = v_total_spending,
        updated_at = NOW()
    WHERE id = p_customer_id;

    -- Logger l'événement
    INSERT INTO loyalty_events (customer_id, event_type, points, tier)
    VALUES (p_customer_id, 'POINTS_EARNED', p_points_earned, p_new_tier);
END$$

DELIMITER ;

-- Utilisation
CALL apply_loyalty_points(42, 250.00, @points, @tier);
SELECT @points AS points_earned, @tier AS new_tier;
```

### 3. Procédure avec Boucle et Conditions

```sql
DELIMITER $$

CREATE PROCEDURE generate_monthly_report(
    IN p_year INT,
    IN p_month INT
)
BEGIN
    DECLARE v_day INT DEFAULT 1;
    DECLARE v_days_in_month INT;
    DECLARE v_current_date DATE;
    DECLARE v_daily_total DECIMAL(10,2);

    -- Créer une table temporaire pour le rapport
    CREATE TEMPORARY TABLE IF NOT EXISTS temp_daily_sales (
        report_date DATE,
        total_sales DECIMAL(10,2),
        order_count INT
    ) ENGINE=MEMORY;

    -- Calculer le nombre de jours dans le mois
    SET v_days_in_month = DAY(LAST_DAY(CONCAT(p_year, '-', p_month, '-01')));

    -- Boucle sur chaque jour du mois
    WHILE v_day <= v_days_in_month DO
        SET v_current_date = CONCAT(p_year, '-', p_month, '-', v_day);

        -- Calculer le total pour ce jour
        SELECT
            COALESCE(SUM(total_amount), 0),
            COUNT(*)
        INTO
            v_daily_total,
            @order_count
        FROM orders
        WHERE DATE(created_at) = v_current_date;

        -- Insérer dans le rapport
        INSERT INTO temp_daily_sales (report_date, total_sales, order_count)
        VALUES (v_current_date, v_daily_total, @order_count);

        SET v_day = v_day + 1;
    END WHILE;

    -- Retourner le rapport
    SELECT
        report_date AS date,
        total_sales,
        order_count,
        ROUND(total_sales / order_count, 2) AS avg_order_value
    FROM temp_daily_sales
    ORDER BY report_date;

    -- Nettoyer
    DROP TEMPORARY TABLE IF EXISTS temp_daily_sales;
END$$

DELIMITER ;

-- Utilisation
CALL generate_monthly_report(2025, 12);
```

---

## Bonnes Pratiques

### 1. ✅ Nommage et Conventions

```sql
-- ✅ BON : Noms clairs et cohérents
CREATE PROCEDURE create_customer(...)
CREATE PROCEDURE update_customer_email(...)
CREATE PROCEDURE get_customer_orders(...)
CREATE PROCEDURE calculate_invoice_total(...)

-- ❌ MAUVAIS : Noms ambigus
CREATE PROCEDURE proc1(...)
CREATE PROCEDURE do_stuff(...)
CREATE PROCEDURE x(...)

-- Préfixes pour les variables
DECLARE v_total DECIMAL(10,2);      -- v_ = variable locale
DECLARE c_max_retry INT DEFAULT 3;  -- c_ = constante
-- p_customer_id                     -- p_ = paramètre (dans signature)
```

### 2. ✅ Documentation

```sql
DELIMITER $$

CREATE PROCEDURE process_refund(
    IN p_order_id INT,
    IN p_reason VARCHAR(255),
    OUT p_refund_amount DECIMAL(10,2)
)
COMMENT 'Traite un remboursement de commande avec validation et audit - v1.2 - 2025-12-12'
BEGIN
    -- ============================================
    -- Procédure : process_refund
    -- Description : Gère le remboursement complet d'une commande
    -- Auteur : Team Backend
    -- Version : 1.2
    -- Date : 2025-12-12
    --
    -- Paramètres :
    --   IN  p_order_id       : ID de la commande à rembourser
    --   IN  p_reason         : Motif du remboursement
    --   OUT p_refund_amount  : Montant remboursé
    --
    -- Retour :
    --   p_refund_amount > 0  : Succès
    --   p_refund_amount = 0  : Échec (voir audit_log)
    --
    -- Notes :
    --   - Vérifie que la commande n'est pas déjà remboursée
    --   - Crée automatiquement un avoir client
    --   - Log toutes les opérations dans audit_log
    -- ============================================

    -- Code ici
END$$

DELIMITER ;
```

### 3. ✅ Gestion d'Erreurs Robuste

```sql
DELIMITER $$

CREATE PROCEDURE safe_update_stock(
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_success BOOLEAN,
    OUT p_error_message VARCHAR(255)
)
BEGIN
    DECLARE v_current_stock INT;
    DECLARE v_rows_affected INT;

    -- Handler pour erreurs SQL génériques
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
        SET p_error_message = 'Database error occurred';

        -- Logger l'erreur
        INSERT IGNORE INTO error_log (procedure_name, error_message, created_at)
        VALUES ('safe_update_stock', p_error_message, NOW());
    END;

    -- Handler pour contraintes violées
    DECLARE EXIT HANDLER FOR SQLSTATE '23000'
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
        SET p_error_message = 'Constraint violation';
    END;

    START TRANSACTION;

    -- Validation métier
    IF p_quantity < 0 THEN
        SET p_success = FALSE;
        SET p_error_message = 'Quantity must be positive';
        ROLLBACK;
    ELSE
        UPDATE products
        SET stock = stock + p_quantity
        WHERE id = p_product_id;

        SET v_rows_affected = ROW_COUNT();

        IF v_rows_affected = 0 THEN
            SET p_success = FALSE;
            SET p_error_message = 'Product not found';
            ROLLBACK;
        ELSE
            SET p_success = TRUE;
            SET p_error_message = NULL;
            COMMIT;
        END IF;
    END IF;
END$$

DELIMITER ;
```

### 4. ✅ Performance et Optimisation

```sql
-- ❌ MAUVAIS : Requêtes multiples dans une boucle
DELIMITER $$
CREATE PROCEDURE update_prices_bad()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_id INT;
    DECLARE cur CURSOR FOR SELECT id FROM products;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;
    read_loop: LOOP
        FETCH cur INTO v_id;
        IF done THEN LEAVE read_loop; END IF;

        -- Une requête par produit ❌
        UPDATE products SET price = price * 1.1 WHERE id = v_id;
    END LOOP;
    CLOSE cur;
END$$
DELIMITER ;

-- ✅ BON : Opération ensembliste
DELIMITER $$
CREATE PROCEDURE update_prices_good()
BEGIN
    -- Une seule requête pour tous les produits ✅
    UPDATE products
    SET price = price * 1.1
    WHERE is_active = TRUE;

    -- Logger le nombre de produits modifiés
    INSERT INTO audit_log (operation, affected_rows)
    VALUES ('PRICE_UPDATE', ROW_COUNT());
END$$
DELIMITER ;
```

### 5. ✅ Sécurité

```sql
-- ❌ DANGEREUX : SQL dynamique non sécurisé
DELIMITER $$
CREATE PROCEDURE search_products_bad(IN p_keyword VARCHAR(100))
BEGIN
    SET @sql = CONCAT('SELECT * FROM products WHERE name LIKE "%', p_keyword, '%"');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    -- Vulnérable à l'injection SQL ! ❌
END$$
DELIMITER ;

-- ✅ SÉCURISÉ : Paramètres correctement utilisés
DELIMITER $$
CREATE PROCEDURE search_products_good(IN p_keyword VARCHAR(100))
BEGIN
    -- Méthode 1 : Requête directe (préférée)
    SELECT * FROM products
    WHERE name LIKE CONCAT('%', p_keyword, '%');

    -- Méthode 2 : Prepared statement avec paramètre
    -- PREPARE stmt FROM 'SELECT * FROM products WHERE name LIKE CONCAT("%", ?, "%")';
    -- EXECUTE stmt USING p_keyword;
    -- DEALLOCATE PREPARE stmt;
END$$
DELIMITER ;
```

### 6. ✅ Testabilité

```sql
DELIMITER $$

CREATE PROCEDURE calculate_commission(
    IN p_sales_amount DECIMAL(10,2),
    IN p_employee_level VARCHAR(20),
    OUT p_commission DECIMAL(10,2)
)
DETERMINISTIC  -- Facilite les tests
BEGIN
    -- Logique pure, sans effets de bord
    CASE p_employee_level
        WHEN 'SENIOR' THEN
            SET p_commission = p_sales_amount * 0.15;
        WHEN 'JUNIOR' THEN
            SET p_commission = p_sales_amount * 0.10;
        ELSE
            SET p_commission = p_sales_amount * 0.05;
    END CASE;

    -- Pas d'INSERT/UPDATE/DELETE
    -- Facile à tester avec différentes valeurs
END$$

DELIMITER ;

-- Tests unitaires simples
CALL calculate_commission(1000, 'SENIOR', @result);
SELECT @result = 150 AS test_senior_passed;

CALL calculate_commission(1000, 'JUNIOR', @result);
SELECT @result = 100 AS test_junior_passed;
```

---

## Sécurité SQL SECURITY

```sql
-- DEFINER : Exécution avec les droits du créateur
DELIMITER $$
CREATE DEFINER = 'admin'@'localhost'
PROCEDURE restricted_operation()
SQL SECURITY DEFINER
BEGIN
    -- S'exécute avec les droits de 'admin'
    -- Même si l'appelant n'a pas ces droits
    DELETE FROM sensitive_data WHERE obsolete = TRUE;
END$$
DELIMITER ;

-- INVOKER : Exécution avec les droits de l'appelant
DELIMITER $$
CREATE PROCEDURE user_operation()
SQL SECURITY INVOKER
BEGIN
    -- S'exécute avec les droits de l'utilisateur qui appelle
    -- Échoue si l'utilisateur n'a pas les droits nécessaires
    SELECT * FROM users WHERE id = USER();
END$$
DELIMITER ;
```

**Recommandations** :
- ✅ Utilisez `DEFINER` pour des opérations nécessitant des privilèges élevés
- ✅ Utilisez `INVOKER` pour respecter les droits de l'utilisateur
- ⚠️ Par défaut : `DEFINER = current_user`

---

## Gestion et Maintenance

### Lister les Procédures

```sql
-- Toutes les procédures de la base courante
SHOW PROCEDURE STATUS WHERE Db = DATABASE();

-- Afficher le code source
SHOW CREATE PROCEDURE nom_procedure;

-- Informations depuis INFORMATION_SCHEMA
SELECT
    ROUTINE_NAME,
    ROUTINE_TYPE,
    DEFINER,
    CREATED,
    LAST_ALTERED,
    SQL_MODE,
    SECURITY_TYPE,
    ROUTINE_COMMENT
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = 'mydb'
AND ROUTINE_TYPE = 'PROCEDURE';
```

### Modifier une Procédure

```sql
-- MariaDB 10.3+ : CREATE OR REPLACE
CREATE OR REPLACE PROCEDURE my_procedure()
BEGIN
    -- Nouvelle version
END;

-- Sinon : DROP puis CREATE
DROP PROCEDURE IF EXISTS my_procedure;
CREATE PROCEDURE my_procedure()
BEGIN
    -- Nouvelle version
END;
```

### Supprimer une Procédure

```sql
DROP PROCEDURE IF EXISTS nom_procedure;
```

### Privilèges

```sql
-- Accorder le droit de créer des procédures
GRANT CREATE ROUTINE ON mydb.* TO 'developer'@'%';

-- Accorder le droit d'exécuter une procédure
GRANT EXECUTE ON PROCEDURE mydb.process_order TO 'app_user'@'%';

-- Accorder le droit de modifier
GRANT ALTER ROUTINE ON mydb.* TO 'developer'@'%';

-- Révoquer
REVOKE EXECUTE ON PROCEDURE mydb.process_order FROM 'app_user'@'%';
```

---

## ⚠️ Erreurs Courantes et Solutions

### 1. Variable non déclarée

```sql
-- ❌ ERREUR
CREATE PROCEDURE bad_proc()
BEGIN
    SET total = 100;  -- Variable non déclarée
END;

-- ✅ SOLUTION
CREATE PROCEDURE good_proc()
BEGIN
    DECLARE total INT;
    SET total = 100;
END;
```

### 2. Ordre des déclarations incorrect

```sql
-- ❌ ERREUR : Handler avant variable
CREATE PROCEDURE bad_order()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    DECLARE v_total INT;  -- Erreur : déclarations après handler
END;

-- ✅ SOLUTION : Ordre correct
CREATE PROCEDURE good_order()
BEGIN
    -- 1. Variables
    DECLARE v_total INT;

    -- 2. Cursors
    -- DECLARE cur CURSOR FOR ...

    -- 3. Handlers
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    -- 4. Instructions
END;
```

### 3. Utilisation de OUT sans variable de session

```sql
-- ❌ ERREUR
CALL get_total(123);  -- Paramètre OUT manquant

-- ✅ SOLUTION
SET @result = 0;
CALL get_total(@result);
SELECT @result;
```

---

## ✅ Points clés à retenir

- 🎯 **Encapsulation** : Les procédures stockées encapsulent la logique métier côté serveur
- 📊 **Paramètres** : IN (entrée), OUT (sortie), INOUT (entrée/sortie) offrent une grande flexibilité
- 🔒 **Transactions** : Contrôle ACID complet avec START TRANSACTION, COMMIT, ROLLBACK
- ⚡ **Performance** : Réduction des allers-retours réseau, plans d'exécution en cache
- 🛡️ **Sécurité** : Privilèges granulaires (EXECUTE), SQL SECURITY DEFINER/INVOKER
- 🔧 **Maintenance** : CREATE OR REPLACE pour versions, documentation via COMMENT
- 📝 **Bonnes pratiques** : Nommage cohérent, gestion d'erreurs robuste, éviter les curseurs
- ⚠️ **Limitations** : Portabilité réduite, debugging complexe, charge CPU serveur

---

## 🔗 Ressources et références

- [📖 CREATE PROCEDURE - MariaDB Documentation](https://mariadb.com/kb/en/create-procedure/)
- [📖 Stored Routines - MariaDB Documentation](https://mariadb.com/kb/en/stored-routines/)
- [📖 Stored Procedure Best Practices](https://mariadb.com/kb/en/stored-procedure-best-practices/)
- [📖 CALL Statement - MariaDB Documentation](https://mariadb.com/kb/en/call/)
- [📝 SQL/PSM (Persistent Stored Modules)](https://mariadb.com/kb/en/sql-statements-structure/)

---

## ➡️ Section suivante

**8.1.1 Syntaxe CREATE PROCEDURE** : Détails de la syntaxe complète, options avancées (DETERMINISTIC, NO SQL, MODIFIES SQL DATA), et exemples exhaustifs de création de procédures stockées.

---


⏭️ [Syntaxe CREATE PROCEDURE](/08-programmation-cote-serveur/01.1-syntaxe-create-procedure.md)

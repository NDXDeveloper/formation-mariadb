🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.6 Gestion des Erreurs et Exceptions (DECLARE HANDLER)

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Procédures stockées (8.1), Transactions (chapitre 6), SQL avancé (chapitres 3-4)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le système de gestion d'erreurs dans MariaDB
- **Utiliser** les différents types de handlers (CONTINUE, EXIT, UNDO)
- **Déclarer** des handlers pour différentes conditions (SQLEXCEPTION, NOT FOUND, etc.)
- **Lever** des erreurs personnalisées avec SIGNAL et RESIGNAL
- **Gérer** les transactions en cas d'erreur avec ROLLBACK
- **Implémenter** des patterns robustes de gestion d'erreurs
- **Appliquer** les bonnes pratiques pour un code serveur fiable

---

## Introduction

La **gestion des erreurs** est essentielle pour créer des procédures stockées, fonctions et triggers robustes en production. MariaDB offre un système complet de gestion d'exceptions permettant d'intercepter, traiter et propager les erreurs de manière contrôlée.

### 🔍 Pourquoi Gérer les Erreurs ?

Sans gestion d'erreurs appropriée :
- ❌ Les transactions peuvent laisser les données dans un état incohérent
- ❌ Les erreurs se propagent de manière imprévisible
- ❌ Le debugging devient très difficile
- ❌ Pas de traçabilité des problèmes

Avec une gestion d'erreurs robuste :
- ✅ Rollback automatique en cas de problème
- ✅ Logging des erreurs pour diagnostic
- ✅ Comportement prévisible et contrôlé
- ✅ Maintenance facilitée

---

## Types de Handlers

MariaDB supporte trois types de handlers qui déterminent le comportement après qu'une condition soit rencontrée.

### 1️⃣ CONTINUE Handler

**Continue l'exécution** après avoir traité la condition.

```sql
DELIMITER $$

CREATE PROCEDURE continue_handler_example()
BEGIN
    DECLARE v_error_count INT DEFAULT 0;
    DECLARE v_id INT;

    -- Handler CONTINUE : Continue après l'erreur
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error_count = v_error_count + 1;
        INSERT INTO error_log (message, created_at)
        VALUES ('An error occurred', NOW());
    END;

    -- Première opération (peut échouer)
    INSERT INTO customers (id, name) VALUES (1, 'John');

    -- Deuxième opération (s'exécute même si la première a échoué)
    INSERT INTO customers (id, name) VALUES (2, 'Jane');

    -- Afficher le nombre d'erreurs
    SELECT CONCAT('Errors encountered: ', v_error_count) AS result;
END$$

DELIMITER ;
```

**Utilisation typique** :
- Traiter des erreurs non-critiques
- Logger les erreurs sans interrompre le processus
- Parcourir plusieurs éléments où certains peuvent échouer

### 2️⃣ EXIT Handler

**Quitte le bloc BEGIN/END** qui contient le handler.

```sql
DELIMITER $$

CREATE PROCEDURE exit_handler_example()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Code exécuté en cas d'erreur
        ROLLBACK;
        SELECT 'An error occurred, transaction rolled back' AS status;
        -- Quitte le bloc BEGIN/END principal
    END;

    START TRANSACTION;

    -- Si une erreur se produit ici, l'EXIT handler est déclenché
    INSERT INTO customers (id, name) VALUES (1, 'John');
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;

    -- Si tout va bien, on arrive ici
    COMMIT;
    SELECT 'Transaction completed successfully' AS status;
END$$

DELIMITER ;
```

**Utilisation typique** :
- Gérer les erreurs critiques
- Rollback de transactions
- Terminaison propre en cas de problème

### 3️⃣ UNDO Handler

**Rollback automatique** de la transaction et quitte le bloc.

⚠️ **Note** : Les handlers UNDO ne sont **pas supportés** dans MariaDB (ni MySQL). Ils sont définis dans la norme SQL mais non implémentés.

```sql
-- ❌ Non supporté dans MariaDB
DECLARE UNDO HANDLER FOR SQLEXCEPTION
BEGIN
    -- Ne fonctionne pas
END;
```

💡 **Alternative** : Utilisez EXIT HANDLER avec ROLLBACK explicite.

### 📊 Tableau Comparatif

| Type | Comportement | Usage Typique | Support MariaDB |
|------|--------------|---------------|-----------------|
| **CONTINUE** | Continue l'exécution | Erreurs non-critiques, logging | ✅ Oui |
| **EXIT** | Quitte le bloc BEGIN/END | Erreurs critiques, rollback | ✅ Oui |
| **UNDO** | Rollback automatique | - | ❌ Non |

---

## Conditions d'Erreur

Les handlers peuvent être déclarés pour différents types de conditions.

### Types de Conditions

#### 1. SQLEXCEPTION - Erreurs SQL Générales

Capture toutes les erreurs SQL (SQLSTATE commençant par autre chose que '00', '01', '02').

```sql
DELIMITER $$

CREATE PROCEDURE handle_sqlexception()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        INSERT INTO error_log (error_type, message)
        VALUES ('SQLEXCEPTION', 'A SQL error occurred');
    END;

    START TRANSACTION;

    -- Opérations qui peuvent échouer
    INSERT INTO orders (customer_id, total) VALUES (999, 100);  -- Customer peut ne pas exister

    COMMIT;
END$$

DELIMITER ;
```

#### 2. SQLWARNING - Avertissements

Capture les avertissements (SQLSTATE commençant par '01').

```sql
DELIMITER $$

CREATE PROCEDURE handle_sqlwarning()
BEGIN
    DECLARE v_warning_count INT DEFAULT 0;

    DECLARE CONTINUE HANDLER FOR SQLWARNING
    BEGIN
        SET v_warning_count = v_warning_count + 1;
    END;

    -- Opération qui génère un warning
    INSERT INTO products (name) VALUES ('Test');  -- Si une colonne NOT NULL n'a pas de valeur par défaut

    SELECT CONCAT('Warnings: ', v_warning_count) AS result;
END$$

DELIMITER ;
```

#### 3. NOT FOUND - Aucune Ligne Trouvée

Capture l'absence de résultat (SQLSTATE '02000').

```sql
DELIMITER $$

CREATE PROCEDURE handle_not_found()
BEGIN
    DECLARE v_name VARCHAR(100);
    DECLARE v_found BOOLEAN DEFAULT TRUE;

    DECLARE CONTINUE HANDLER FOR NOT FOUND
        SET v_found = FALSE;

    -- Tentative de récupération
    SELECT name INTO v_name
    FROM customers
    WHERE id = 999;  -- Peut ne pas exister

    IF v_found THEN
        SELECT CONCAT('Customer found: ', v_name) AS result;
    ELSE
        SELECT 'Customer not found' AS result;
    END IF;
END$$

DELIMITER ;
```

#### 4. SQLSTATE Spécifique

Capture un code SQLSTATE particulier.

```sql
DELIMITER $$

CREATE PROCEDURE handle_specific_sqlstate()
BEGIN
    -- Handler pour violation de clé primaire (SQLSTATE 23000)
    DECLARE CONTINUE HANDLER FOR SQLSTATE '23000'
    BEGIN
        INSERT INTO error_log (error_type, message)
        VALUES ('DUPLICATE_KEY', 'Duplicate key violation');
    END;

    -- Handler pour clé étrangère (SQLSTATE 23000 aussi, mais on peut différencier)
    DECLARE CONTINUE HANDLER FOR 1452  -- Errno pour FK violation
    BEGIN
        INSERT INTO error_log (error_type, message)
        VALUES ('FOREIGN_KEY', 'Foreign key constraint failed');
    END;

    -- Opérations
    INSERT INTO customers (id, name) VALUES (1, 'John');  -- Peut violer PK
END$$

DELIMITER ;
```

#### 5. Condition Nommée

Définir une condition pour la réutiliser.

```sql
DELIMITER $$

CREATE PROCEDURE named_condition_example()
BEGIN
    -- Définir une condition nommée
    DECLARE duplicate_key CONDITION FOR SQLSTATE '23000';
    DECLARE foreign_key_error CONDITION FOR 1452;

    -- Utiliser la condition nommée
    DECLARE CONTINUE HANDLER FOR duplicate_key
    BEGIN
        SELECT 'Duplicate key detected' AS error_message;
    END;

    DECLARE CONTINUE HANDLER FOR foreign_key_error
    BEGIN
        SELECT 'Foreign key constraint violated' AS error_message;
    END;

    -- Opérations
    INSERT INTO orders (id, customer_id) VALUES (1, 999);
END$$

DELIMITER ;
```

### 📋 SQLSTATE Courants

| SQLSTATE | Errno | Description |
|----------|-------|-------------|
| **23000** | 1062 | Duplicate key (PRIMARY/UNIQUE) |
| **23000** | 1452 | Foreign key constraint failed |
| **42S02** | 1146 | Table doesn't exist |
| **42S22** | 1054 | Unknown column |
| **22001** | 1406 | Data too long for column |
| **02000** | 1329 | No data (NOT FOUND) |
| **HY000** | Various | General error |

---

## SIGNAL - Lever une Erreur

`SIGNAL` permet de lever une erreur personnalisée, similaire à `throw` ou `raise` dans d'autres langages.

### Syntaxe de Base

```sql
DELIMITER $$

CREATE PROCEDURE signal_example(IN p_amount DECIMAL(10,2))
BEGIN
    -- Validation métier
    IF p_amount < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Amount cannot be negative';
    END IF;

    IF p_amount > 1000000 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Amount exceeds maximum limit',
            MYSQL_ERRNO = 1644;
    END IF;

    -- Traitement normal
    INSERT INTO transactions (amount) VALUES (p_amount);
END$$

DELIMITER ;

-- Test
CALL signal_example(-100);
-- Erreur : Amount cannot be negative
```

### SIGNAL avec Informations Détaillées

```sql
DELIMITER $$

CREATE PROCEDURE detailed_signal_example(IN p_user_id INT)
BEGIN
    DECLARE v_status VARCHAR(20);

    -- Vérifier le statut de l'utilisateur
    SELECT status INTO v_status
    FROM users
    WHERE id = p_user_id;

    IF v_status IS NULL THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'User not found',
            MYSQL_ERRNO = 1001,
            TABLE_NAME = 'users',
            COLUMN_NAME = 'id';
    END IF;

    IF v_status = 'suspended' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'User account is suspended',
            MYSQL_ERRNO = 1002;
    END IF;
END$$

DELIMITER ;
```

### SQLSTATE Personnalisés

```sql
DELIMITER $$

CREATE PROCEDURE custom_sqlstate_example()
BEGIN
    -- SQLSTATE commençant par '45' sont réservés pour les erreurs utilisateur

    -- Erreur de validation
    SIGNAL SQLSTATE '45001'
    SET MESSAGE_TEXT = 'Validation failed';

    -- Erreur métier
    SIGNAL SQLSTATE '45002'
    SET MESSAGE_TEXT = 'Business rule violation';

    -- Erreur d'autorisation
    SIGNAL SQLSTATE '45003'
    SET MESSAGE_TEXT = 'Insufficient permissions';
END$$

DELIMITER ;
```

---

## RESIGNAL - Propager une Erreur

`RESIGNAL` permet de propager une erreur capturée par un handler, éventuellement en modifiant le message.

### Propager Sans Modification

```sql
DELIMITER $$

CREATE PROCEDURE resignal_basic()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Logger l'erreur
        INSERT INTO error_log (message, created_at)
        VALUES ('Error occurred in resignal_basic', NOW());

        -- Propager l'erreur originale
        RESIGNAL;
    END;

    -- Opération qui peut échouer
    INSERT INTO orders (id, customer_id) VALUES (1, 999);
END$$

DELIMITER ;
```

### Propager avec Modification du Message

```sql
DELIMITER $$

CREATE PROCEDURE resignal_modified()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Logger l'erreur originale
        INSERT INTO error_log (message) VALUES ('Original error logged');

        -- Propager avec un message modifié
        RESIGNAL SET MESSAGE_TEXT = 'Failed to process order - see error log for details';
    END;

    INSERT INTO orders (customer_id, total) VALUES (NULL, 100);  -- Violation NOT NULL
END$$

DELIMITER ;
```

### Exemple Avancé : Enrichir l'Erreur

```sql
DELIMITER $$

CREATE PROCEDURE enrich_error_info(IN p_order_id INT)
BEGIN
    DECLARE v_original_message TEXT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Récupérer le message d'erreur original (pas directement accessible)
        -- On ajoute du contexte
        RESIGNAL SET
            MESSAGE_TEXT = CONCAT('Error processing order #', p_order_id,
                                  '. Original operation failed.'),
            MYSQL_ERRNO = 1644;
    END;

    -- Opération complexe
    UPDATE orders SET status = 'shipped' WHERE id = p_order_id;
    DELETE FROM cart WHERE order_id = p_order_id;
END$$

DELIMITER ;
```

---

## Gestion Transactionnelle des Erreurs

La combinaison de handlers et de transactions est essentielle pour maintenir l'intégrité des données.

### Pattern Classique : Transaction avec Rollback

```sql
DELIMITER $$

CREATE PROCEDURE transactional_operation(
    IN p_from_account INT,
    IN p_to_account INT,
    IN p_amount DECIMAL(10,2),
    OUT p_success BOOLEAN
)
BEGIN
    DECLARE v_balance DECIMAL(10,2);

    -- Handler pour rollback automatique en cas d'erreur
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;

        INSERT INTO error_log (operation, message, created_at)
        VALUES ('transfer', 'Transaction failed and rolled back', NOW());
    END;

    -- Initialiser
    SET p_success = FALSE;

    -- Démarrer la transaction
    START TRANSACTION;

    -- Vérifier le solde
    SELECT balance INTO v_balance
    FROM accounts
    WHERE id = p_from_account
    FOR UPDATE;  -- Lock pessimiste

    IF v_balance < p_amount THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Insufficient balance';
    END IF;

    -- Débiter le compte source
    UPDATE accounts
    SET balance = balance - p_amount
    WHERE id = p_from_account;

    -- Créditer le compte destination
    UPDATE accounts
    SET balance = balance + p_amount
    WHERE id = p_to_account;

    -- Logger la transaction
    INSERT INTO transaction_log (from_account, to_account, amount)
    VALUES (p_from_account, p_to_account, p_amount);

    -- Si tout s'est bien passé
    COMMIT;
    SET p_success = TRUE;
END$$

DELIMITER ;

-- Utilisation
CALL transactional_operation(1, 2, 500.00, @success);
SELECT @success;
```

### Savepoints pour Rollback Partiel

```sql
DELIMITER $$

CREATE PROCEDURE partial_rollback_example()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Rollback complet si erreur critique
        ROLLBACK;
        INSERT INTO error_log (message) VALUES ('Complete rollback');
    END;

    START TRANSACTION;

    -- Opération 1
    INSERT INTO orders (customer_id, total) VALUES (1, 100);

    -- Savepoint
    SAVEPOINT before_items;

    -- Opération 2 (peut échouer sans tout rollback)
    BEGIN
        DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
        BEGIN
            -- Rollback partiel uniquement
            ROLLBACK TO SAVEPOINT before_items;
            INSERT INTO error_log (message) VALUES ('Partial rollback to savepoint');
        END;

        INSERT INTO order_items (order_id, product_id, quantity)
        VALUES (LAST_INSERT_ID(), 999, 1);  -- Peut échouer
    END;

    -- Opération 3
    UPDATE customers SET last_order_date = NOW() WHERE id = 1;

    COMMIT;
END$$

DELIMITER ;
```

---

## Patterns de Gestion d'Erreurs

### 1. Pattern : Try-Catch-Finally

```sql
DELIMITER $$

CREATE PROCEDURE try_catch_finally_pattern()
BEGIN
    DECLARE v_cleanup_needed BOOLEAN DEFAULT FALSE;
    DECLARE v_error BOOLEAN DEFAULT FALSE;

    -- CATCH : Handler d'erreur
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error = TRUE;
        ROLLBACK;

        -- FINALLY : Nettoyage même en cas d'erreur
        IF v_cleanup_needed THEN
            DELETE FROM temp_processing_data;
        END IF;

        INSERT INTO error_log (message) VALUES ('Error in try_catch_finally');
    END;

    -- TRY : Bloc principal
    START TRANSACTION;

    -- Créer des données temporaires
    INSERT INTO temp_processing_data SELECT * FROM input_data;
    SET v_cleanup_needed = TRUE;

    -- Traitement principal
    CALL complex_processing();

    -- Si succès
    COMMIT;

    -- FINALLY : Nettoyage en cas de succès
    IF v_cleanup_needed THEN
        DELETE FROM temp_processing_data;
    END IF;
END$$

DELIMITER ;
```

### 2. Pattern : Error Accumulation

```sql
DELIMITER $$

CREATE PROCEDURE error_accumulation_pattern()
BEGIN
    DECLARE v_error_count INT DEFAULT 0;
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    -- Table pour accumuler les erreurs
    CREATE TEMPORARY TABLE IF NOT EXISTS temp_errors (
        record_id INT,
        error_message TEXT
    ) ENGINE=MEMORY;

    -- Handler pour continuer malgré les erreurs
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error_count = v_error_count + 1;
        INSERT INTO temp_errors (record_id, error_message)
        VALUES (v_id, 'Processing failed');
    END;

    DECLARE record_cursor CURSOR FOR SELECT id FROM records_to_process;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN record_cursor;

    process_loop: LOOP
        FETCH record_cursor INTO v_id;

        IF v_done THEN
            LEAVE process_loop;
        END IF;

        -- Traiter chaque enregistrement (peut échouer)
        CALL process_record(v_id);

    END LOOP process_loop;

    CLOSE record_cursor;

    -- Rapport final
    SELECT
        v_error_count AS total_errors,
        COUNT(*) AS successful_records
    FROM records_to_process
    WHERE id NOT IN (SELECT record_id FROM temp_errors);

    -- Afficher les erreurs
    IF v_error_count > 0 THEN
        SELECT * FROM temp_errors;
    END IF;

    DROP TEMPORARY TABLE IF EXISTS temp_errors;
END$$

DELIMITER ;
```

### 3. Pattern : Retry avec Backoff

```sql
DELIMITER $$

CREATE PROCEDURE retry_with_backoff(IN p_order_id INT)
BEGIN
    DECLARE v_retry_count INT DEFAULT 0;
    DECLARE v_max_retries INT DEFAULT 3;
    DECLARE v_success BOOLEAN DEFAULT FALSE;

    retry_loop: LOOP
        BEGIN
            DECLARE EXIT HANDLER FOR SQLEXCEPTION
            BEGIN
                SET v_retry_count = v_retry_count + 1;

                IF v_retry_count >= v_max_retries THEN
                    -- Abandonner après N tentatives
                    INSERT INTO error_log (message, details)
                    VALUES ('Max retries reached', CONCAT('Order: ', p_order_id));

                    SIGNAL SQLSTATE '45000'
                    SET MESSAGE_TEXT = 'Operation failed after retries';
                END IF;

                -- Attendre avant de réessayer (simulé)
                -- En production, utiliser un mécanisme externe
                INSERT INTO retry_log (attempt, order_id)
                VALUES (v_retry_count, p_order_id);
            END;

            -- Tentative d'opération
            START TRANSACTION;

            UPDATE orders
            SET status = 'processing'
            WHERE id = p_order_id;

            CALL complex_order_processing(p_order_id);

            COMMIT;

            -- Si on arrive ici, succès
            SET v_success = TRUE;
            LEAVE retry_loop;
        END;
    END LOOP retry_loop;

    IF v_success THEN
        SELECT 'Operation succeeded' AS result;
    END IF;
END$$

DELIMITER ;
```

### 4. Pattern : Validation Chain

```sql
DELIMITER $$

CREATE PROCEDURE validation_chain(
    IN p_customer_id INT,
    IN p_amount DECIMAL(10,2)
)
BEGIN
    DECLARE v_status VARCHAR(20);
    DECLARE v_balance DECIMAL(10,2);
    DECLARE v_credit_limit DECIMAL(10,2);

    -- Handler centralisé pour toutes les validations
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        -- Le message d'erreur spécifique a été set par SIGNAL
        RESIGNAL;
    END;

    START TRANSACTION;

    -- Validation 1 : Client existe
    SELECT status, balance, credit_limit
    INTO v_status, v_balance, v_credit_limit
    FROM customers
    WHERE id = p_customer_id;

    IF v_status IS NULL THEN
        SIGNAL SQLSTATE '45001'
        SET MESSAGE_TEXT = 'Customer not found';
    END IF;

    -- Validation 2 : Client actif
    IF v_status != 'active' THEN
        SIGNAL SQLSTATE '45002'
        SET MESSAGE_TEXT = 'Customer account is not active';
    END IF;

    -- Validation 3 : Montant positif
    IF p_amount <= 0 THEN
        SIGNAL SQLSTATE '45003'
        SET MESSAGE_TEXT = 'Amount must be positive';
    END IF;

    -- Validation 4 : Limite de crédit
    IF p_amount > v_credit_limit THEN
        SIGNAL SQLSTATE '45004'
        SET MESSAGE_TEXT = 'Amount exceeds credit limit';
    END IF;

    -- Validation 5 : Solde suffisant
    IF v_balance < p_amount THEN
        SIGNAL SQLSTATE '45005'
        SET MESSAGE_TEXT = 'Insufficient balance';
    END IF;

    -- Si toutes les validations passent
    INSERT INTO orders (customer_id, total, status)
    VALUES (p_customer_id, p_amount, 'pending');

    COMMIT;

    SELECT 'Order created successfully' AS result;
END$$

DELIMITER ;
```

---

## Bonnes Pratiques

### 1. ✅ Toujours Utiliser des Handlers en Production

```sql
-- ❌ MAUVAIS : Pas de gestion d'erreur
CREATE PROCEDURE bad_no_handler()
BEGIN
    START TRANSACTION;
    UPDATE accounts SET balance = balance - 1000;
    UPDATE accounts SET balance = balance + 1000;
    COMMIT;
    -- Si une erreur se produit, transaction partiellement commitée !
END;

-- ✅ BON : Handler avec rollback
CREATE PROCEDURE good_with_handler()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        INSERT INTO error_log (message) VALUES ('Transaction failed');
    END;

    START TRANSACTION;
    UPDATE accounts SET balance = balance - 1000;
    UPDATE accounts SET balance = balance + 1000;
    COMMIT;
END;
```

### 2. ✅ Logger les Erreurs pour Diagnostic

```sql
DELIMITER $$

-- Table de log d'erreurs
CREATE TABLE IF NOT EXISTS error_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    procedure_name VARCHAR(64),
    error_code INT,
    error_message TEXT,
    sqlstate VARCHAR(5),
    user VARCHAR(100),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE PROCEDURE logged_procedure()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Logger avec maximum d'informations
        INSERT INTO error_log (
            procedure_name,
            error_message,
            sqlstate,
            user
        ) VALUES (
            'logged_procedure',
            'An error occurred during processing',
            '45000',
            USER()
        );

        ROLLBACK;
    END;

    START TRANSACTION;
    -- Opérations
    COMMIT;
END$$

DELIMITER ;
```

### 3. ✅ Utiliser RESIGNAL pour Contexte

```sql
DELIMITER $$

CREATE PROCEDURE inner_procedure()
BEGIN
    -- Opération qui peut échouer
    INSERT INTO orders (customer_id) VALUES (NULL);
END$$

CREATE PROCEDURE outer_procedure(IN p_user_id INT)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Ajouter du contexte avant de propager
        INSERT INTO error_log (message, details)
        VALUES ('Error in outer_procedure', CONCAT('User: ', p_user_id));

        RESIGNAL SET MESSAGE_TEXT = CONCAT('Failed to process for user ', p_user_id);
    END;

    CALL inner_procedure();
END$$

DELIMITER ;
```

### 4. ✅ Hiérarchie de Handlers

```sql
DELIMITER $$

CREATE PROCEDURE handler_hierarchy()
BEGIN
    -- Handler général (moins spécifique)
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        INSERT INTO error_log (error_type) VALUES ('GENERAL_SQL_ERROR');
    END;

    -- Handler spécifique (plus spécifique, prioritaire)
    DECLARE CONTINUE HANDLER FOR SQLSTATE '23000'
    BEGIN
        INSERT INTO error_log (error_type) VALUES ('DUPLICATE_KEY');
    END;

    -- Handler encore plus spécifique
    DECLARE CONTINUE HANDLER FOR 1062  -- Errno pour duplicate
    BEGIN
        INSERT INTO error_log (error_type) VALUES ('DUPLICATE_PRIMARY_KEY');
    END;

    -- MariaDB choisira le handler le plus spécifique qui correspond
    INSERT INTO customers (id, name) VALUES (1, 'John');
END$$

DELIMITER ;
```

### 5. ✅ Messages d'Erreur Clairs et Actionnables

```sql
DELIMITER $$

CREATE PROCEDURE clear_error_messages(IN p_email VARCHAR(255))
BEGIN
    -- ❌ MAUVAIS : Message vague
    IF p_email NOT REGEXP '@' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Invalid input';
    END IF;

    -- ✅ BON : Message spécifique et actionnable
    IF p_email IS NULL THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Email is required and cannot be NULL';
    END IF;

    IF p_email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = CONCAT('Email format is invalid: "', p_email,
                                  '". Expected format: user@example.com');
    END IF;
END$$

DELIMITER ;
```

### 6. ✅ Nettoyer les Ressources

```sql
DELIMITER $$

CREATE PROCEDURE cleanup_resources()
BEGIN
    DECLARE v_cursor_opened BOOLEAN DEFAULT FALSE;
    DECLARE my_cursor CURSOR FOR SELECT id FROM temp_data;

    -- Handler pour garantir le nettoyage
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Fermer le curseur si ouvert
        IF v_cursor_opened THEN
            CLOSE my_cursor;
        END IF;

        -- Supprimer les données temporaires
        DELETE FROM temp_processing;

        ROLLBACK;
    END;

    START TRANSACTION;

    -- Créer des données temporaires
    INSERT INTO temp_processing SELECT * FROM source_data;

    -- Ouvrir le curseur
    OPEN my_cursor;
    SET v_cursor_opened = TRUE;

    -- Traitement
    -- ...

    -- Nettoyage normal
    CLOSE my_cursor;
    SET v_cursor_opened = FALSE;
    DELETE FROM temp_processing;

    COMMIT;
END$$

DELIMITER ;
```

### 7. ✅ Tester les Chemins d'Erreur

```sql
DELIMITER $$

CREATE PROCEDURE testable_procedure(
    IN p_customer_id INT,
    IN p_test_mode BOOLEAN
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;

        IF p_test_mode THEN
            SELECT 'ERROR_HANDLER_TRIGGERED' AS test_result;
        END IF;
    END;

    START TRANSACTION;

    -- En mode test, on peut forcer des erreurs
    IF p_test_mode AND p_customer_id = -1 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Test error';
    END IF;

    -- Traitement normal
    INSERT INTO orders (customer_id) VALUES (p_customer_id);

    COMMIT;

    IF p_test_mode THEN
        SELECT 'SUCCESS' AS test_result;
    END IF;
END$$

DELIMITER ;

-- Test du chemin de succès
CALL testable_procedure(1, TRUE);

-- Test du chemin d'erreur
CALL testable_procedure(-1, TRUE);
```

---

## Exemple Complet : Système de Commande Robuste

```sql
-- Tables nécessaires
CREATE TABLE IF NOT EXISTS customers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    email VARCHAR(100),
    status ENUM('active', 'suspended', 'closed') DEFAULT 'active',
    credit_limit DECIMAL(10,2) DEFAULT 1000.00
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    total_amount DECIMAL(10,2),
    status ENUM('pending', 'confirmed', 'failed') DEFAULT 'pending',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    price DECIMAL(10,2),
    stock INT DEFAULT 0
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS order_error_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    error_code VARCHAR(10),
    error_message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

DELIMITER $$

CREATE PROCEDURE create_order_robust(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_order_id INT,
    OUT p_status VARCHAR(20),
    OUT p_message TEXT
)
BEGIN
    DECLARE v_customer_status VARCHAR(20);
    DECLARE v_credit_limit DECIMAL(10,2);
    DECLARE v_product_price DECIMAL(10,2);
    DECLARE v_product_stock INT;
    DECLARE v_total_amount DECIMAL(10,2);

    -- ═══════════════════════════════════════════════════════
    -- Handler Central : Gestion de toutes les erreurs
    -- ═══════════════════════════════════════════════════════
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Rollback de la transaction
        ROLLBACK;

        -- Logger l'erreur
        INSERT INTO order_error_log (customer_id, error_code, error_message)
        VALUES (p_customer_id, 'SQL_ERROR', 'Database error occurred');

        -- Retourner l'état d'erreur
        SET p_order_id = NULL;
        SET p_status = 'ERROR';
        SET p_message = 'An unexpected error occurred. Please contact support.';
    END;

    -- Initialiser les variables de sortie
    SET p_order_id = NULL;
    SET p_status = 'PENDING';
    SET p_message = '';

    -- ═══════════════════════════════════════════════════════
    -- Démarrer la transaction
    -- ═══════════════════════════════════════════════════════
    START TRANSACTION;

    -- ═══════════════════════════════════════════════════════
    -- VALIDATION 1 : Client existe et est actif
    -- ═══════════════════════════════════════════════════════
    SELECT status, credit_limit
    INTO v_customer_status, v_credit_limit
    FROM customers
    WHERE id = p_customer_id;

    IF v_customer_status IS NULL THEN
        ROLLBACK;
        SET p_status = 'REJECTED';
        SET p_message = CONCAT('Customer ID ', p_customer_id, ' not found');

        INSERT INTO order_error_log (customer_id, error_code, error_message)
        VALUES (p_customer_id, 'CUSTOMER_NOT_FOUND', p_message);

        LEAVE;
    END IF;

    IF v_customer_status != 'active' THEN
        ROLLBACK;
        SET p_status = 'REJECTED';
        SET p_message = CONCAT('Customer account is ', v_customer_status);

        INSERT INTO order_error_log (customer_id, error_code, error_message)
        VALUES (p_customer_id, 'CUSTOMER_INACTIVE', p_message);

        LEAVE;
    END IF;

    -- ═══════════════════════════════════════════════════════
    -- VALIDATION 2 : Produit existe et en stock
    -- ═══════════════════════════════════════════════════════
    SELECT price, stock
    INTO v_product_price, v_product_stock
    FROM products
    WHERE id = p_product_id
    FOR UPDATE;  -- Lock pour éviter les races conditions

    IF v_product_price IS NULL THEN
        ROLLBACK;
        SET p_status = 'REJECTED';
        SET p_message = CONCAT('Product ID ', p_product_id, ' not found');

        INSERT INTO order_error_log (customer_id, error_code, error_message)
        VALUES (p_customer_id, 'PRODUCT_NOT_FOUND', p_message);

        LEAVE;
    END IF;

    IF v_product_stock < p_quantity THEN
        ROLLBACK;
        SET p_status = 'REJECTED';
        SET p_message = CONCAT('Insufficient stock. Available: ', v_product_stock,
                               ', Requested: ', p_quantity);

        INSERT INTO order_error_log (customer_id, error_code, error_message)
        VALUES (p_customer_id, 'INSUFFICIENT_STOCK', p_message);

        LEAVE;
    END IF;

    -- ═══════════════════════════════════════════════════════
    -- VALIDATION 3 : Limite de crédit
    -- ═══════════════════════════════════════════════════════
    SET v_total_amount = v_product_price * p_quantity;

    IF v_total_amount > v_credit_limit THEN
        ROLLBACK;
        SET p_status = 'REJECTED';
        SET p_message = CONCAT('Order total (', v_total_amount,
                               ') exceeds credit limit (', v_credit_limit, ')');

        INSERT INTO order_error_log (customer_id, error_code, error_message)
        VALUES (p_customer_id, 'CREDIT_LIMIT_EXCEEDED', p_message);

        LEAVE;
    END IF;

    -- ═══════════════════════════════════════════════════════
    -- Créer la commande
    -- ═══════════════════════════════════════════════════════
    INSERT INTO orders (customer_id, total_amount, status)
    VALUES (p_customer_id, v_total_amount, 'pending');

    SET p_order_id = LAST_INSERT_ID();

    -- Ajouter les items
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (p_order_id, p_product_id, p_quantity, v_product_price);

    -- Décrémenter le stock
    UPDATE products
    SET stock = stock - p_quantity
    WHERE id = p_product_id;

    -- ═══════════════════════════════════════════════════════
    -- Commit de la transaction
    -- ═══════════════════════════════════════════════════════
    COMMIT;

    SET p_status = 'SUCCESS';
    SET p_message = CONCAT('Order #', p_order_id, ' created successfully');

END$$

DELIMITER ;

-- ═══════════════════════════════════════════════════════
-- Tests
-- ═══════════════════════════════════════════════════════

-- Insérer des données de test
INSERT INTO customers (id, name, email, status, credit_limit)
VALUES (1, 'John Doe', 'john@example.com', 'active', 1000.00);

INSERT INTO products (id, name, price, stock)
VALUES (1, 'Laptop', 800.00, 10);

-- Test réussi
CALL create_order_robust(1, 1, 1, @order_id, @status, @message);
SELECT @order_id, @status, @message;

-- Test : Stock insuffisant
CALL create_order_robust(1, 1, 100, @order_id, @status, @message);
SELECT @order_id, @status, @message;

-- Vérifier les logs d'erreur
SELECT * FROM order_error_log;
```

---

## ✅ Points clés à retenir

- 🎯 **Handlers** : CONTINUE (continue), EXIT (quitte bloc), UNDO (non supporté)
- 🔍 **Conditions** : SQLEXCEPTION, SQLWARNING, NOT FOUND, SQLSTATE, condition nommée
- 🚨 **SIGNAL** : Lever une erreur personnalisée (SQLSTATE '45xxx')
- 🔄 **RESIGNAL** : Propager une erreur en l'enrichissant de contexte
- 💾 **Transactions** : Toujours combiner handlers avec START TRANSACTION/COMMIT/ROLLBACK
- 📝 **Logging** : Enregistrer toutes les erreurs pour diagnostic
- 🔧 **Patterns** : Try-Catch-Finally, Error Accumulation, Retry, Validation Chain
- ✅ **Bonnes pratiques** : Handlers systématiques, messages clairs, nettoyage ressources, tests

---

## 🔗 Ressources et références

- [📖 DECLARE HANDLER - MariaDB Documentation](https://mariadb.com/kb/en/declare-handler/)
- [📖 SIGNAL - MariaDB Documentation](https://mariadb.com/kb/en/signal/)
- [📖 RESIGNAL - MariaDB Documentation](https://mariadb.com/kb/en/resignal/)
- [📖 SQLSTATE Values - MariaDB Documentation](https://mariadb.com/kb/en/sqlstate/)
- [📖 Error Handling - MariaDB Documentation](https://mariadb.com/kb/en/programmatic-and-compound-statements-diagnostics-and-condition-handling/)

---

## ➡️ Sections suivantes

- **8.7 Variables et flow control** : IF, CASE, LOOP, WHILE, REPEAT, variables locales et de session
- **8.8 Bonnes pratiques** : Standards de production, tests, maintenance, documentation complète

---


⏭️ [Variables et flow control (IF, CASE, LOOP, WHILE, REPEAT)](/08-programmation-cote-serveur/07-variables-flow-control.md)

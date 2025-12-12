🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.8 Bonnes Pratiques de Programmation Serveur

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Toutes les sections précédentes du chapitre 8

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Appliquer** des standards de code cohérents et professionnels
- **Documenter** efficacement les routines serveur
- **Tester** les procédures stockées et fonctions
- **Optimiser** les performances du code serveur
- **Sécuriser** les routines contre les vulnérabilités
- **Maintenir** et versionner le code en production
- **Déployer** les changements de manière contrôlée
- **Suivre** les meilleures pratiques de l'industrie

---

## Introduction

Les **bonnes pratiques** de programmation serveur ne sont pas optionnelles en production. Elles garantissent la maintenabilité, la fiabilité et les performances de votre code SQL. Cette section synthétise les recommandations essentielles pour développer des routines serveur professionnelles.

### 🎯 Principes Fondamentaux

Les bonnes pratiques reposent sur quatre piliers :

1. **Lisibilité** : Code clair et compréhensible
2. **Fiabilité** : Gestion d'erreurs robuste
3. **Performance** : Optimisation systématique
4. **Sécurité** : Protection contre les vulnérabilités

---

## 1. Conventions de Nommage

### Standards Recommandés

```sql
-- ═══════════════════════════════════════════════════════════
-- PROCÉDURES : verbe_nom_contexte
-- ═══════════════════════════════════════════════════════════
CREATE PROCEDURE create_customer_account(...)
CREATE PROCEDURE update_order_status(...)
CREATE PROCEDURE get_customer_orders(...)
CREATE PROCEDURE delete_expired_sessions(...)
CREATE PROCEDURE calculate_monthly_revenue(...)

-- ═══════════════════════════════════════════════════════════
-- FONCTIONS : get_xxx ou calculate_xxx ou is_xxx
-- ═══════════════════════════════════════════════════════════
CREATE FUNCTION get_customer_name(id INT) RETURNS VARCHAR(100)
CREATE FUNCTION calculate_discount_amount(...) RETURNS DECIMAL(10,2)
CREATE FUNCTION is_valid_email(email VARCHAR) RETURNS BOOLEAN
CREATE FUNCTION format_phone_number(...) RETURNS VARCHAR(20)

-- ═══════════════════════════════════════════════════════════
-- TRIGGERS : table_timing_action
-- ═══════════════════════════════════════════════════════════
CREATE TRIGGER orders_before_insert
CREATE TRIGGER users_after_update
CREATE TRIGGER products_before_delete

-- ═══════════════════════════════════════════════════════════
-- EVENTS : action_fréquence
-- ═══════════════════════════════════════════════════════════
CREATE EVENT purge_logs_daily
CREATE EVENT update_statistics_hourly
CREATE EVENT backup_data_weekly

-- ═══════════════════════════════════════════════════════════
-- VARIABLES : préfixe selon le type
-- ═══════════════════════════════════════════════════════════
DECLARE v_customer_name VARCHAR(100);      -- v_ = variable locale
DECLARE c_max_attempts INT DEFAULT 3;     -- c_ = constante
-- p_customer_id                           -- p_ = paramètre (dans signature)
-- @session_var                            -- @ = variable de session
```

### ✅ Bonnes Pratiques de Nommage

```sql
DELIMITER $$

-- ✅ BON : Noms descriptifs et cohérents
CREATE PROCEDURE process_pending_orders(
    IN p_batch_size INT,
    OUT p_processed_count INT,
    OUT p_error_count INT
)
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_processing_date DATETIME DEFAULT NOW();
    DECLARE c_max_retries INT DEFAULT 3;

    -- Variables avec préfixes clairs
    DECLARE v_is_valid BOOLEAN;
    DECLARE v_total_amount DECIMAL(15,2);
END$$

-- ❌ MAUVAIS : Noms ambigus ou incohérents
CREATE PROCEDURE proc1(IN x INT, OUT y INT)
BEGIN
    DECLARE temp VARCHAR(100);
    DECLARE cnt INT;
    DECLARE flag BOOLEAN;
    -- Difficile à comprendre
END$$

DELIMITER ;
```

### Règles de Nommage

| Élément | Convention | Exemple |
|---------|-----------|---------|
| **Procédures** | `verbe_nom` | `create_order`, `update_stock` |
| **Fonctions** | `get_`, `calculate_`, `is_` | `get_total`, `is_valid` |
| **Triggers** | `table_timing_action` | `orders_after_insert` |
| **Events** | `action_frequency` | `purge_logs_daily` |
| **Variables locales** | `v_nom_descriptif` | `v_customer_id` |
| **Paramètres** | `p_nom_descriptif` | `p_amount` |
| **Constantes** | `c_NOM_MAJUSCULE` | `c_MAX_RETRY` |
| **Curseurs** | `nom_cursor` | `customer_cursor` |
| **Labels** | `nom_descriptif` | `process_loop` |

---

## 2. Documentation et Commentaires

### Header de Procédure Standard

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE process_customer_order(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_order_id INT,
    OUT p_status VARCHAR(20)
)
COMMENT 'Crée une commande client avec validation complète - v2.1 - 2025-12-12'
BEGIN
    -- ════════════════════════════════════════════════════════════════
    -- Procédure : process_customer_order
    -- ════════════════════════════════════════════════════════════════
    -- Description :
    --   Traite une nouvelle commande client en vérifiant :
    --   - Existence et statut du client
    --   - Disponibilité du produit
    --   - Stock suffisant
    --   - Limite de crédit
    --
    -- Paramètres :
    --   IN  p_customer_id : ID du client (doit exister et être actif)
    --   IN  p_product_id  : ID du produit à commander
    --   IN  p_quantity    : Quantité (doit être > 0)
    --   OUT p_order_id    : ID de la commande créée (NULL si erreur)
    --   OUT p_status      : Status ('SUCCESS', 'ERROR', 'REJECTED')
    --
    -- Retour :
    --   p_order_id : ID de la nouvelle commande ou NULL
    --   p_status   : Code de statut indiquant le résultat
    --
    -- Tables modifiées :
    --   - orders : INSERT
    --   - order_items : INSERT
    --   - products : UPDATE (stock)
    --
    -- Exceptions :
    --   - SQLSTATE 45001 : Client non trouvé
    --   - SQLSTATE 45002 : Stock insuffisant
    --   - SQLSTATE 45003 : Limite de crédit dépassée
    --
    -- Auteur : Team Backend
    -- Version : 2.1
    -- Date création : 2025-01-15
    -- Dernière modif : 2025-12-12
    --
    -- Historique :
    --   v2.1 (2025-12-12) : Ajout vérification limite crédit
    --   v2.0 (2025-06-10) : Refonte validation stock
    --   v1.0 (2025-01-15) : Version initiale
    --
    -- Dépendances :
    --   - Aucune procédure externe
    --
    -- Notes :
    --   - Transaction ACID complète avec rollback automatique
    --   - Optimisé pour charges moyennes (<1000 TPS)
    --   - Testé jusqu'à 100k commandes/jour
    -- ════════════════════════════════════════════════════════════════

    -- Déclarations
    DECLARE v_customer_status VARCHAR(20);
    DECLARE v_stock INT;

    -- Handler d'erreur
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_order_id = NULL;
        SET p_status = 'ERROR';
    END;

    -- Code de la procédure
    START TRANSACTION;

    -- ... logique ...

    COMMIT;
END$$

DELIMITER ;
```

### Commentaires Inline

```sql
DELIMITER $$

CREATE PROCEDURE well_commented_procedure()
BEGIN
    DECLARE v_total DECIMAL(15,2) DEFAULT 0.0;

    -- ════════════════════════════════════════════════════════════════
    -- SECTION 1 : Validation des données d'entrée
    -- ════════════════════════════════════════════════════════════════

    -- Vérifier que le montant est positif
    IF v_amount <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Amount must be positive';
    END IF;

    -- ════════════════════════════════════════════════════════════════
    -- SECTION 2 : Calcul du total avec remises progressives
    -- Règles métier :
    -- - 0-100€   : 5% de remise
    -- - 100-500€ : 10% de remise
    -- - >500€    : 15% de remise
    -- ════════════════════════════════════════════════════════════════

    SET v_total = CASE
        WHEN v_amount <= 100 THEN v_amount * 0.95
        WHEN v_amount <= 500 THEN v_amount * 0.90
        ELSE v_amount * 0.85
    END;

    -- ════════════════════════════════════════════════════════════════
    -- SECTION 3 : Enregistrement du résultat
    -- ════════════════════════════════════════════════════════════════

    INSERT INTO calculations (amount, result, created_at)
    VALUES (v_amount, v_total, NOW());
END$$

DELIMITER ;
```

### Types de Commentaires

```sql
-- Commentaire simple ligne : pour explications courtes

/*
 * Commentaire multi-lignes : pour explications longues
 * ou documentation de blocs complexes
 */

-- ════════════════════════════════════════════════════════════════
-- Séparateur visuel : pour marquer les sections importantes
-- ════════════════════════════════════════════════════════════════

-- TODO: Implémenter la validation email
-- FIXME: Bug avec les montants négatifs
-- HACK: Solution temporaire, à refactoriser
-- NOTE: Cette logique est spécifique au contexte français
-- WARNING: Ne pas modifier sans tester l'impact sur les commandes
```

---

## 3. Gestion des Erreurs

### Pattern Complet de Gestion d'Erreurs

```sql
DELIMITER $$

CREATE PROCEDURE error_handling_best_practice(
    IN p_customer_id INT,
    OUT p_success BOOLEAN,
    OUT p_error_code VARCHAR(10),
    OUT p_error_message TEXT
)
BEGIN
    -- Variables
    DECLARE v_customer_status VARCHAR(20);

    -- ════════════════════════════════════════════════════════════════
    -- Handler centralisé pour toutes les erreurs
    -- ════════════════════════════════════════════════════════════════
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Rollback automatique
        ROLLBACK;

        -- Logger l'erreur avec contexte
        INSERT INTO error_log (
            procedure_name,
            error_code,
            error_message,
            customer_id,
            created_at
        ) VALUES (
            'error_handling_best_practice',
            'SQL_ERROR',
            'Database error occurred',
            p_customer_id,
            NOW()
        );

        -- Définir les paramètres de sortie
        SET p_success = FALSE;
        SET p_error_code = 'SQL_ERROR';
        SET p_error_message = 'An unexpected error occurred. Please contact support.';
    END;

    -- Handler spécifique pour violations de contraintes
    DECLARE EXIT HANDLER FOR SQLSTATE '23000'
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
        SET p_error_code = 'CONSTRAINT';
        SET p_error_message = 'Data constraint violation';
    END;

    -- ════════════════════════════════════════════════════════════════
    -- Initialisation
    -- ════════════════════════════════════════════════════════════════
    SET p_success = FALSE;
    SET p_error_code = NULL;
    SET p_error_message = NULL;

    -- ════════════════════════════════════════════════════════════════
    -- Logique principale avec transaction
    -- ════════════════════════════════════════════════════════════════
    START TRANSACTION;

    -- Validation métier avec erreurs explicites
    SELECT status INTO v_customer_status
    FROM customers
    WHERE id = p_customer_id;

    IF v_customer_status IS NULL THEN
        SET p_error_code = 'NOT_FOUND';
        SET p_error_message = CONCAT('Customer ', p_customer_id, ' not found');
        ROLLBACK;
        LEAVE;
    END IF;

    IF v_customer_status != 'active' THEN
        SET p_error_code = 'INACTIVE';
        SET p_error_message = CONCAT('Customer account is ', v_customer_status);
        ROLLBACK;
        LEAVE;
    END IF;

    -- Traitement
    -- ...

    COMMIT;

    -- Succès
    SET p_success = TRUE;
    SET p_error_message = 'Operation completed successfully';
END$$

DELIMITER ;
```

### Checklist Gestion d'Erreurs

- ✅ Handler EXIT pour erreurs critiques avec ROLLBACK
- ✅ Handler CONTINUE pour erreurs non-critiques avec logging
- ✅ Messages d'erreur clairs et actionnables
- ✅ Codes d'erreur documentés
- ✅ Logging systématique dans une table error_log
- ✅ Paramètres OUT pour communiquer l'état
- ✅ RESIGNAL avec contexte enrichi
- ✅ Tests des chemins d'erreur

---

## 4. Performance et Optimisation

### Règles d'Or de Performance

```sql
-- ════════════════════════════════════════════════════════════════
-- RÈGLE 1 : Préférer SET-based à Row-by-row
-- ════════════════════════════════════════════════════════════════

-- ❌ LENT : Curseur (10 secondes pour 10k lignes)
DELIMITER $$
CREATE PROCEDURE slow_update()
BEGIN
    DECLARE v_id INT;
    DECLARE done BOOLEAN DEFAULT FALSE;
    DECLARE cur CURSOR FOR SELECT id FROM products;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;
    LOOP
        FETCH cur INTO v_id;
        IF done THEN LEAVE; END IF;
        UPDATE products SET price = price * 1.1 WHERE id = v_id;
    END LOOP;
    CLOSE cur;
END$$
DELIMITER ;

-- ✅ RAPIDE : Opération ensembliste (0.05 secondes)
DELIMITER $$
CREATE PROCEDURE fast_update()
BEGIN
    UPDATE products
    SET price = price * 1.1
    WHERE is_active = TRUE;

    SELECT CONCAT('Updated ', ROW_COUNT(), ' products') AS result;
END$$
DELIMITER ;

-- ════════════════════════════════════════════════════════════════
-- RÈGLE 2 : Limiter les SELECT dans les boucles
-- ════════════════════════════════════════════════════════════════

-- ❌ MAUVAIS : SELECT dans chaque itération
WHILE v_counter < 100 DO
    SELECT COUNT(*) INTO v_count FROM orders WHERE customer_id = v_counter;
    -- Très lent : 100 SELECT
    SET v_counter = v_counter + 1;
END WHILE;

-- ✅ BON : Un seul SELECT avec agrégation
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;

-- ════════════════════════════════════════════════════════════════
-- RÈGLE 3 : Utiliser des index appropriés
-- ════════════════════════════════════════════════════════════════

-- Avant de créer une procédure qui filtre sur created_at
CREATE INDEX idx_orders_created ON orders(created_at);

-- ════════════════════════════════════════════════════════════════
-- RÈGLE 4 : Limiter les données traitées
-- ════════════════════════════════════════════════════════════════

-- ❌ MAUVAIS : Traiter toute la table
CREATE PROCEDURE process_all_orders()
BEGIN
    DECLARE cur CURSOR FOR SELECT * FROM orders;  -- Potentiellement millions
    -- ...
END;

-- ✅ BON : Limiter avec WHERE et LIMIT
CREATE PROCEDURE process_recent_orders()
BEGIN
    DECLARE cur CURSOR FOR
        SELECT id, customer_id, total
        FROM orders
        WHERE created_at >= CURDATE() - INTERVAL 7 DAY
        AND status = 'pending'
        LIMIT 1000;
    -- ...
END;

-- ════════════════════════════════════════════════════════════════
-- RÈGLE 5 : Transactions par batch
-- ════════════════════════════════════════════════════════════════

DELIMITER $$
CREATE PROCEDURE batch_insert_optimized()
BEGIN
    DECLARE v_counter INT DEFAULT 0;

    START TRANSACTION;

    WHILE v_counter < 10000 DO
        INSERT INTO data VALUES (...);
        SET v_counter = v_counter + 1;

        -- Commit tous les 1000 inserts
        IF v_counter % 1000 = 0 THEN
            COMMIT;
            START TRANSACTION;
        END IF;
    END WHILE;

    COMMIT;
END$$
DELIMITER ;
```

### Checklist Performance

- ✅ Pas de curseurs (sauf cas exceptionnels justifiés)
- ✅ Index sur colonnes filtrées (WHERE, JOIN, ORDER BY)
- ✅ SELECT uniquement colonnes nécessaires
- ✅ LIMIT pour restreindre les résultats
- ✅ Transactions par batch (commit réguliers)
- ✅ Éviter les SELECT dans les boucles
- ✅ Utiliser EXPLAIN pour analyser les requêtes
- ✅ Benchmark avec données réelles

---

## 5. Sécurité

### Protection contre les Injections SQL

```sql
DELIMITER $$

-- ❌ VULNÉRABLE : SQL dynamique non sécurisé
CREATE PROCEDURE vulnerable_search(IN p_search_term VARCHAR(100))
BEGIN
    -- DANGER : Injection SQL possible !
    SET @query = CONCAT('SELECT * FROM products WHERE name = "', p_search_term, '"');
    PREPARE stmt FROM @query;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END$$

-- ✅ SÉCURISÉ : Paramètres liés
CREATE PROCEDURE secure_search(IN p_search_term VARCHAR(100))
BEGIN
    -- Méthode 1 : Requête directe (préférée)
    SELECT * FROM products
    WHERE name LIKE CONCAT('%', p_search_term, '%');

    -- Méthode 2 : Prepared statement avec paramètre
    PREPARE stmt FROM 'SELECT * FROM products WHERE name LIKE CONCAT("%", ?, "%")';
    EXECUTE stmt USING p_search_term;
    DEALLOCATE PREPARE stmt;
END$$

DELIMITER ;
```

### Validation des Entrées

```sql
DELIMITER $$

CREATE PROCEDURE secure_input_validation(
    IN p_email VARCHAR(255),
    IN p_amount DECIMAL(10,2),
    IN p_quantity INT
)
BEGIN
    -- ════════════════════════════════════════════════════════════════
    -- Validation complète des entrées
    -- ════════════════════════════════════════════════════════════════

    -- Validation 1 : Email format
    IF p_email IS NULL OR p_email = '' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Email is required';
    END IF;

    IF p_email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = CONCAT('Invalid email format: ', p_email);
    END IF;

    -- Validation 2 : Montant
    IF p_amount IS NULL THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Amount is required';
    END IF;

    IF p_amount < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Amount cannot be negative';
    END IF;

    IF p_amount > 1000000 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Amount exceeds maximum limit';
    END IF;

    -- Validation 3 : Quantité
    IF p_quantity IS NULL OR p_quantity <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Quantity must be positive';
    END IF;

    IF p_quantity > 1000 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Quantity exceeds maximum order size';
    END IF;

    -- Traitement sécurisé
    -- ...
END$$

DELIMITER ;
```

### Gestion des Privilèges

```sql
-- ════════════════════════════════════════════════════════════════
-- Principe du moindre privilège
-- ════════════════════════════════════════════════════════════════

-- Créer un rôle pour l'application
CREATE ROLE app_role;

-- Donner uniquement EXECUTE sur les procédures nécessaires
GRANT EXECUTE ON PROCEDURE mydb.create_order TO app_role;
GRANT EXECUTE ON PROCEDURE mydb.get_customer_info TO app_role;

-- NE PAS donner d'accès direct aux tables
-- REVOKE ALL ON mydb.* FROM app_role;

-- Assigner le rôle aux utilisateurs applicatifs
CREATE USER 'app_user'@'%' IDENTIFIED BY 'strong_password';
GRANT app_role TO 'app_user'@'%';
SET DEFAULT ROLE app_role FOR 'app_user'@'%';

-- ════════════════════════════════════════════════════════════════
-- SQL SECURITY dans les routines
-- ════════════════════════════════════════════════════════════════

-- DEFINER : Exécution avec droits du créateur (sensible)
CREATE DEFINER = 'admin'@'localhost'
PROCEDURE admin_operation()
SQL SECURITY DEFINER
BEGIN
    -- S'exécute avec les droits admin
    DELETE FROM sensitive_data WHERE obsolete = TRUE;
END;

-- INVOKER : Exécution avec droits de l'appelant (plus sûr)
CREATE PROCEDURE user_operation()
SQL SECURITY INVOKER
BEGIN
    -- S'exécute avec les droits de l'utilisateur
    SELECT * FROM his_own_data;
END;
```

### Checklist Sécurité

- ✅ Validation systématique des entrées
- ✅ Pas de SQL dynamique non sécurisé
- ✅ Principe du moindre privilège
- ✅ SQL SECURITY INVOKER par défaut
- ✅ Pas de données sensibles dans les commentaires
- ✅ Logging des opérations sensibles
- ✅ Utilisation de prepared statements si SQL dynamique nécessaire

---

## 6. Tests

### Stratégie de Tests

```sql
-- ════════════════════════════════════════════════════════════════
-- 1. Tests Unitaires : Tester chaque fonction individuellement
-- ════════════════════════════════════════════════════════════════

DELIMITER $$

-- Fonction à tester
CREATE FUNCTION calculate_vat(amount DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
BEGIN
    RETURN amount * 0.20;
END$$

-- Procédure de test
CREATE PROCEDURE test_calculate_vat()
BEGIN
    DECLARE v_result DECIMAL(10,2);
    DECLARE v_test_passed BOOLEAN DEFAULT TRUE;

    -- Test 1 : Montant normal
    SET v_result = calculate_vat(100.00);
    IF v_result != 20.00 THEN
        SELECT 'FAIL: Test 1 - Expected 20.00, got ' AS status, v_result;
        SET v_test_passed = FALSE;
    END IF;

    -- Test 2 : Montant zéro
    SET v_result = calculate_vat(0.00);
    IF v_result != 0.00 THEN
        SELECT 'FAIL: Test 2 - Expected 0.00, got ' AS status, v_result;
        SET v_test_passed = FALSE;
    END IF;

    -- Test 3 : Montant décimal
    SET v_result = calculate_vat(123.45);
    IF ABS(v_result - 24.69) > 0.01 THEN
        SELECT 'FAIL: Test 3 - Expected 24.69, got ' AS status, v_result;
        SET v_test_passed = FALSE;
    END IF;

    IF v_test_passed THEN
        SELECT 'PASS: All tests passed' AS status;
    END IF;
END$$

DELIMITER ;

-- Exécuter les tests
CALL test_calculate_vat();

-- ════════════════════════════════════════════════════════════════
-- 2. Tests d'Intégration : Tester avec données réelles
-- ════════════════════════════════════════════════════════════════

DELIMITER $$

CREATE PROCEDURE test_create_order_integration()
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_status VARCHAR(20);

    -- Setup : Créer des données de test
    START TRANSACTION;

    INSERT INTO test_customers (id, name, status, credit_limit)
    VALUES (9999, 'Test Customer', 'active', 1000.00);

    INSERT INTO test_products (id, name, price, stock)
    VALUES (9999, 'Test Product', 100.00, 50);

    -- Exécuter la procédure à tester
    CALL create_order(9999, 9999, 5, v_order_id, v_status);

    -- Vérifications
    IF v_status != 'SUCCESS' THEN
        SELECT CONCAT('FAIL: Expected SUCCESS, got ', v_status) AS test_result;
    ELSEIF v_order_id IS NULL THEN
        SELECT 'FAIL: Order ID is NULL' AS test_result;
    ELSE
        -- Vérifier que la commande existe
        IF NOT EXISTS(SELECT 1 FROM orders WHERE id = v_order_id) THEN
            SELECT 'FAIL: Order not created' AS test_result;
        ELSE
            SELECT 'PASS: Order created successfully' AS test_result;
        END IF;
    END IF;

    -- Cleanup : Rollback des données de test
    ROLLBACK;
END$$

DELIMITER ;

-- ════════════════════════════════════════════════════════════════
-- 3. Tests de Charge : Tester la performance
-- ════════════════════════════════════════════════════════════════

DELIMITER $$

CREATE PROCEDURE test_performance_benchmark()
BEGIN
    DECLARE v_start_time DATETIME;
    DECLARE v_end_time DATETIME;
    DECLARE v_duration DECIMAL(10,3);
    DECLARE v_iterations INT DEFAULT 0;
    DECLARE v_target_iterations INT DEFAULT 1000;

    SET v_start_time = NOW(3);

    WHILE v_iterations < v_target_iterations DO
        CALL process_order(v_iterations);
        SET v_iterations = v_iterations + 1;
    END WHILE;

    SET v_end_time = NOW(3);
    SET v_duration = TIMESTAMPDIFF(MICROSECOND, v_start_time, v_end_time) / 1000000;

    SELECT
        v_target_iterations AS iterations,
        v_duration AS duration_seconds,
        ROUND(v_target_iterations / v_duration, 2) AS ops_per_second,
        ROUND(v_duration / v_target_iterations * 1000, 3) AS ms_per_operation;
END$$

DELIMITER ;
```

### Checklist Tests

- ✅ Tests unitaires pour chaque fonction
- ✅ Tests d'intégration pour procédures complexes
- ✅ Tests des chemins d'erreur (pas seulement le happy path)
- ✅ Tests avec données limites (NULL, 0, valeurs extrêmes)
- ✅ Tests de performance avec volume réaliste
- ✅ Données de test isolées (pas de pollution de production)
- ✅ Cleanup automatique après tests
- ✅ Documentation des scénarios de test

---

## 7. Versioning et Déploiement

### Gestion des Versions

```sql
-- ════════════════════════════════════════════════════════════════
-- Intégrer le numéro de version dans COMMENT
-- ════════════════════════════════════════════════════════════════

DELIMITER $$

CREATE OR REPLACE PROCEDURE my_procedure()
COMMENT 'v2.3.1 - Added email notification - 2025-12-12 - Author: John Doe'
BEGIN
    -- Code
END$$

DELIMITER ;

-- ════════════════════════════════════════════════════════════════
-- Table de tracking des versions
-- ════════════════════════════════════════════════════════════════

CREATE TABLE schema_versions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    version VARCHAR(20) NOT NULL,
    description TEXT,
    script_name VARCHAR(255),
    applied_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    applied_by VARCHAR(100),
    execution_time_ms INT,
    UNIQUE KEY uk_version (version)
) ENGINE=InnoDB;

-- Script de migration exemple
DELIMITER $$

CREATE PROCEDURE apply_migration_v2_3_1()
BEGIN
    DECLARE v_start_time DATETIME(3);
    DECLARE v_execution_time INT;

    -- Vérifier si déjà appliqué
    IF EXISTS(SELECT 1 FROM schema_versions WHERE version = '2.3.1') THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Migration 2.3.1 already applied';
    END IF;

    SET v_start_time = NOW(3);

    START TRANSACTION;

    -- Appliquer les changements
    DROP PROCEDURE IF EXISTS old_procedure;

    CREATE PROCEDURE new_procedure_v2()
    COMMENT 'v2.3.1 - Refactored logic'
    BEGIN
        -- Nouvelle implémentation
    END;

    -- Enregistrer la migration
    SET v_execution_time = TIMESTAMPDIFF(MICROSECOND, v_start_time, NOW(3)) / 1000;

    INSERT INTO schema_versions (version, description, script_name, applied_by, execution_time_ms)
    VALUES ('2.3.1', 'Refactored order processing', 'migration_2_3_1.sql', USER(), v_execution_time);

    COMMIT;

    SELECT 'Migration 2.3.1 applied successfully' AS result;
END$$

DELIMITER ;
```

### Scripts de Déploiement

```sql
-- ════════════════════════════════════════════════════════════════
-- deploy.sql : Script de déploiement complet
-- ════════════════════════════════════════════════════════════════

-- Préambule
SELECT 'Starting deployment...' AS status;
SELECT NOW() AS deployment_time;
SELECT USER() AS deployed_by;

-- Vérifications pré-déploiement
DO (
    SELECT COUNT(*) INTO @active_connections
    FROM INFORMATION_SCHEMA.PROCESSLIST
    WHERE DB = DATABASE()
);

IF @active_connections > 10 THEN
    SELECT 'WARNING: High number of active connections' AS warning;
END IF;

-- Backup des anciennes versions
CREATE TABLE IF NOT EXISTS backup_procedures_20251212 AS
SELECT ROUTINE_NAME, ROUTINE_DEFINITION
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE()
AND ROUTINE_TYPE = 'PROCEDURE';

-- Déploiement des nouvelles versions
SOURCE procedures/create_order_v2.sql;
SOURCE procedures/process_payment_v2.sql;
SOURCE functions/calculate_discount_v2.sql;

-- Vérifications post-déploiement
SELECT ROUTINE_NAME, CREATED, LAST_ALTERED
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE()
AND LAST_ALTERED >= CURDATE()
ORDER BY LAST_ALTERED DESC;

-- Tests smoke (tests rapides de base)
CALL test_create_order_smoke();
CALL test_process_payment_smoke();

SELECT 'Deployment completed' AS status;
```

### Checklist Déploiement

- ✅ Versioning sémantique (MAJOR.MINOR.PATCH)
- ✅ Changelog documenté
- ✅ Scripts de migration idempotents
- ✅ Backup avant déploiement
- ✅ Tests smoke après déploiement
- ✅ Rollback plan préparé
- ✅ Monitoring post-déploiement
- ✅ Documentation mise à jour

---

## 8. Maintenance et Monitoring

### Logging et Audit

```sql
-- ════════════════════════════════════════════════════════════════
-- Table de monitoring des performances
-- ════════════════════════════════════════════════════════════════

CREATE TABLE procedure_performance_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    procedure_name VARCHAR(64),
    parameters_hash VARCHAR(64),  -- Hash des paramètres
    execution_time_ms DECIMAL(10,3),
    rows_affected INT,
    success BOOLEAN,
    created_at DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3),
    INDEX idx_proc_time (procedure_name, created_at),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;

DELIMITER $$

-- Wrapper pour logging automatique
CREATE PROCEDURE logged_procedure(IN p_param INT)
BEGIN
    DECLARE v_start_time DATETIME(3);
    DECLARE v_execution_time DECIMAL(10,3);
    DECLARE v_rows_affected INT;
    DECLARE v_success BOOLEAN DEFAULT TRUE;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_success = FALSE;
        ROLLBACK;
    END;

    -- Début du monitoring
    SET v_start_time = NOW(3);

    START TRANSACTION;

    -- Logique de la procédure
    -- ...

    COMMIT;

    -- Fin du monitoring
    SET v_execution_time = TIMESTAMPDIFF(MICROSECOND, v_start_time, NOW(3)) / 1000;
    SET v_rows_affected = ROW_COUNT();

    -- Logger les métriques
    INSERT INTO procedure_performance_log (
        procedure_name,
        parameters_hash,
        execution_time_ms,
        rows_affected,
        success
    ) VALUES (
        'logged_procedure',
        MD5(CONCAT(p_param)),
        v_execution_time,
        v_rows_affected,
        v_success
    );
END$$

DELIMITER ;

-- Requête d'analyse des performances
SELECT
    procedure_name,
    COUNT(*) AS executions,
    ROUND(AVG(execution_time_ms), 2) AS avg_time_ms,
    ROUND(MIN(execution_time_ms), 2) AS min_time_ms,
    ROUND(MAX(execution_time_ms), 2) AS max_time_ms,
    ROUND(STDDEV(execution_time_ms), 2) AS stddev_time_ms,
    SUM(CASE WHEN success = FALSE THEN 1 ELSE 0 END) AS error_count
FROM procedure_performance_log
WHERE created_at >= NOW() - INTERVAL 7 DAY
GROUP BY procedure_name
ORDER BY avg_time_ms DESC;
```

### Alertes Automatiques

```sql
DELIMITER $$

-- Event pour détecter les procédures lentes
CREATE EVENT monitor_slow_procedures
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    INSERT INTO system_alerts (alert_type, severity, message)
    SELECT
        'SLOW_PROCEDURE',
        'WARNING',
        CONCAT('Procedure ', procedure_name, ' averaged ',
               ROUND(avg_time, 2), 'ms in the last hour')
    FROM (
        SELECT
            procedure_name,
            AVG(execution_time_ms) AS avg_time
        FROM procedure_performance_log
        WHERE created_at >= NOW() - INTERVAL 1 HOUR
        GROUP BY procedure_name
        HAVING avg_time > 1000  -- Plus de 1 seconde
    ) AS slow_procs;
END$$

DELIMITER ;
```

---

## 9. Checklist Complète des Bonnes Pratiques

### 📝 Nommage et Structure

- [ ] Noms descriptifs et cohérents (verbe_nom pour procédures)
- [ ] Préfixes pour variables (v_, p_, c_, @)
- [ ] Labels pour toutes les boucles
- [ ] Pas de noms ambigus ou génériques (proc1, temp, x)

### 📚 Documentation

- [ ] Header complet avec description, paramètres, auteur, version
- [ ] Commentaires COMMENT dans CREATE
- [ ] Sections documentées (validation, traitement, cleanup)
- [ ] Historique des modifications
- [ ] TODO/FIXME pour travail futur

### 🛡️ Gestion d'Erreurs

- [ ] EXIT HANDLER pour erreurs critiques avec ROLLBACK
- [ ] Logging systématique des erreurs
- [ ] Messages d'erreur clairs et actionnables
- [ ] Paramètres OUT pour communiquer l'état
- [ ] Tests des chemins d'erreur

### ⚡ Performance

- [ ] Pas de curseurs (sauf justification documentée)
- [ ] Opérations ensemblistes (SET-based)
- [ ] Index sur colonnes filtrées
- [ ] LIMIT pour restreindre les résultats
- [ ] Transactions par batch
- [ ] Benchmark avec données réelles

### 🔒 Sécurité

- [ ] Validation de toutes les entrées
- [ ] Pas de SQL dynamique non sécurisé
- [ ] Principe du moindre privilège
- [ ] SQL SECURITY approprié (DEFINER vs INVOKER)
- [ ] Pas de données sensibles dans commentaires ou logs

### 🧪 Tests

- [ ] Tests unitaires pour fonctions
- [ ] Tests d'intégration pour procédures
- [ ] Tests des cas limites (NULL, 0, max)
- [ ] Tests de charge pour opérations critiques
- [ ] Cleanup automatique des données de test

### 📦 Versioning

- [ ] Numéro de version dans COMMENT
- [ ] Changelog documenté
- [ ] Scripts de migration idempotents
- [ ] Table schema_versions à jour

### 🚀 Déploiement

- [ ] Backup avant déploiement
- [ ] Script de déploiement automatisé
- [ ] Tests smoke post-déploiement
- [ ] Plan de rollback préparé
- [ ] Monitoring post-déploiement

---

## ✅ Points clés à retenir

- 🎯 **Standards** : Conventions de nommage cohérentes et documentation complète
- 📝 **Documentation** : Header détaillé, commentaires inline, changelog
- 🛡️ **Erreurs** : Handlers systématiques, logging, messages clairs
- ⚡ **Performance** : SET-based, index, limits, benchmarks
- 🔒 **Sécurité** : Validation entrées, pas de SQL injection, moindre privilège
- 🧪 **Tests** : Unitaires, intégration, charge, cas limites
- 📦 **Versioning** : Semantic versioning, migrations, schema_versions
- 🚀 **Déploiement** : Backup, automatisation, tests smoke, rollback
- 📊 **Monitoring** : Logging performances, alertes automatiques

---

## 🔗 Ressources et références

- [📖 MariaDB Best Practices](https://mariadb.com/kb/en/best-practices/)
- [📖 Stored Procedure Best Practices](https://mariadb.com/kb/en/stored-procedure-best-practices/)
- [📖 SQL Coding Standards](https://www.sqlstyle.guide/)
- [📖 Database CI/CD Best Practices](https://www.liquibase.org/get-started/best-practices)
- [📝 Clean Code Principles](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)

---

**🎉 Conclusion du Chapitre 8**

Vous avez maintenant une base solide en **programmation côté serveur** avec MariaDB. Les bonnes pratiques présentées ici ne sont pas optionnelles : elles sont essentielles pour créer des systèmes fiables, maintenables et performants en production.

**Principes à retenir** :
- Le code est lu plus souvent qu'il n'est écrit → **Lisibilité d'abord**
- Les erreurs vont arriver → **Gérez-les proprement**
- La performance compte → **Optimisez intelligemment**
- La sécurité est critique → **Validez tout**
- Le code évolue → **Documentez et versionnez**

---


⏭️ [Vues et Données Virtuelles](/09-vues-et-donnees-virtuelles/README.md)

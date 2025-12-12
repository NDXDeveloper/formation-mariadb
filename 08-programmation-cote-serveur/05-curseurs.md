🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.5 Curseurs (Cursors)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Procédures stockées (8.1), Fonctions (8.2), SQL avancé (chapitres 3-4)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le fonctionnement et les limites des curseurs dans MariaDB
- **Créer** et manipuler des curseurs pour traiter des ensembles de résultats ligne par ligne
- **Utiliser** les handlers pour gérer la fin de curseur (NOT FOUND)
- **Identifier** les cas où les curseurs sont réellement nécessaires
- **Privilégier** les opérations ensemblistes (SET-based) quand possible
- **Optimiser** les performances des procédures utilisant des curseurs
- **Éviter** les anti-patterns et erreurs courantes

---

## Introduction

Un **curseur** (cursor) est un mécanisme permettant de parcourir **ligne par ligne** le résultat d'une requête SELECT au sein d'une procédure stockée. Contrairement aux opérations ensemblistes qui traitent toutes les lignes en une seule opération, un curseur traite chaque ligne individuellement de manière séquentielle.

### 🔍 Qu'est-ce qu'un Curseur ?

```sql
DELIMITER $$

CREATE PROCEDURE simple_cursor_example()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE customer_name VARCHAR(100);

    -- 1. DÉCLARER le curseur
    DECLARE customer_cursor CURSOR FOR
        SELECT name FROM customers WHERE is_active = TRUE;

    -- 2. DÉCLARER le handler pour la fin
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    -- 3. OUVRIR le curseur
    OPEN customer_cursor;

    -- 4. BOUCLE de lecture
    read_loop: LOOP
        FETCH customer_cursor INTO customer_name;

        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Traiter chaque ligne
        SELECT CONCAT('Processing: ', customer_name);
    END LOOP;

    -- 5. FERMER le curseur
    CLOSE customer_cursor;
END$$

DELIMITER ;
```

### ⚠️ Avertissement Important

**Les curseurs sont généralement une MAUVAISE solution** pour la plupart des problèmes SQL. Ils sont :
- 🐌 **Lents** : Traitement ligne par ligne vs opération ensembliste
- 💾 **Gourmands en mémoire** : Maintiennent le résultat en mémoire
- 🔒 **Bloquants** : Peuvent garder des locks pendant longtemps
- 🐛 **Complexes** : Plus difficiles à débugger et maintenir

💡 **Règle d'or** : Si vous pouvez faire quelque chose avec UPDATE/INSERT/DELETE, ne utilisez PAS de curseur.

---

## Anatomie d'un Curseur

### Cycle de Vie Complet

```
┌─────────────────────────────────────────────────────┐
│  1. DECLARE CURSOR                                  │
│     Définir la requête SELECT                       │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│  2. DECLARE HANDLER                                 │
│     Gérer la fin du curseur (NOT FOUND)             │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│  3. OPEN CURSOR                                     │
│     Exécuter la requête SELECT                      │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│  4. FETCH (boucle)                                  │
│     Récupérer chaque ligne                          │
│     ├─ Traiter la ligne                             │
│     └─ Vérifier si fin (done = TRUE)                │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│  5. CLOSE CURSOR                                    │
│     Libérer les ressources                          │
└─────────────────────────────────────────────────────┘
```

### Syntaxe Détaillée

```sql
DELIMITER $$

CREATE PROCEDURE cursor_anatomy()
BEGIN
    -- ═══════════════════════════════════════════════
    -- ÉTAPE 1 : DÉCLARATIONS DE VARIABLES
    -- ═══════════════════════════════════════════════
    DECLARE v_id INT;
    DECLARE v_name VARCHAR(100);
    DECLARE v_amount DECIMAL(10,2);
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_counter INT DEFAULT 0;

    -- ═══════════════════════════════════════════════
    -- ÉTAPE 2 : DÉCLARATION DU CURSEUR
    -- ═══════════════════════════════════════════════
    -- Le curseur contient une requête SELECT
    DECLARE my_cursor CURSOR FOR
        SELECT id, name, amount
        FROM orders
        WHERE status = 'pending'
        ORDER BY created_at;

    -- ═══════════════════════════════════════════════
    -- ÉTAPE 3 : DÉCLARATION DU HANDLER
    -- ═══════════════════════════════════════════════
    -- Déclenché quand il n'y a plus de lignes à lire
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    -- ═══════════════════════════════════════════════
    -- ÉTAPE 4 : OUVERTURE DU CURSEUR
    -- ═══════════════════════════════════════════════
    -- Exécute la requête SELECT et stocke le résultat
    OPEN my_cursor;

    -- ═══════════════════════════════════════════════
    -- ÉTAPE 5 : BOUCLE DE LECTURE
    -- ═══════════════════════════════════════════════
    read_loop: LOOP
        -- Récupérer la ligne suivante
        FETCH my_cursor INTO v_id, v_name, v_amount;

        -- Vérifier si fin du curseur
        IF v_done THEN
            LEAVE read_loop;
        END IF;

        -- Traiter la ligne courante
        SET v_counter = v_counter + 1;

        -- Exemple de traitement
        INSERT INTO processing_log (order_id, processed_at)
        VALUES (v_id, NOW());

    END LOOP read_loop;

    -- ═══════════════════════════════════════════════
    -- ÉTAPE 6 : FERMETURE DU CURSEUR
    -- ═══════════════════════════════════════════════
    CLOSE my_cursor;

    -- Résultat
    SELECT CONCAT('Processed ', v_counter, ' orders') AS result;
END$$

DELIMITER ;
```

---

## Handlers pour Curseurs

### Types de Handlers

```sql
DELIMITER $$

CREATE PROCEDURE handler_examples()
BEGIN
    DECLARE v_name VARCHAR(100);
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    DECLARE cur CURSOR FOR SELECT name FROM customers;

    -- ═══════════════════════════════════════════════
    -- CONTINUE HANDLER : Continue l'exécution
    -- ═══════════════════════════════════════════════
    DECLARE CONTINUE HANDLER FOR NOT FOUND
        SET v_done = TRUE;

    -- Avec le CONTINUE handler, on peut vérifier v_done
    OPEN cur;
    read_loop: LOOP
        FETCH cur INTO v_name;
        IF v_done THEN
            LEAVE read_loop;
        END IF;
        -- Traiter v_name
    END LOOP;
    CLOSE cur;
END$$

-- Alternative : EXIT HANDLER
CREATE PROCEDURE exit_handler_example()
BEGIN
    DECLARE v_name VARCHAR(100);

    DECLARE cur CURSOR FOR SELECT name FROM customers;

    -- ═══════════════════════════════════════════════
    -- EXIT HANDLER : Quitte le bloc BEGIN/END
    -- ═══════════════════════════════════════════════
    DECLARE EXIT HANDLER FOR NOT FOUND
    BEGIN
        -- Code exécuté à la fin du curseur
        SELECT 'Cursor finished' AS status;
    END;

    OPEN cur;

    -- Pas besoin de vérifier la fin explicitement
    infinite_loop: LOOP
        FETCH cur INTO v_name;
        -- L'EXIT handler quittera automatiquement
        -- Traiter v_name
    END LOOP;

    CLOSE cur;  -- Ne sera jamais atteint avec EXIT HANDLER
END$$

DELIMITER ;
```

### Handler Avancé

```sql
DELIMITER $$

CREATE PROCEDURE advanced_handler()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_error_count INT DEFAULT 0;

    DECLARE cur CURSOR FOR SELECT id FROM orders;

    -- Handler pour NOT FOUND
    DECLARE CONTINUE HANDLER FOR NOT FOUND
        SET v_done = TRUE;

    -- Handler pour erreurs SQL
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error_count = v_error_count + 1;
        INSERT INTO error_log (message, created_at)
        VALUES ('Error processing order', NOW());
    END;

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_id;

        IF v_done THEN
            LEAVE read_loop;
        END IF;

        -- Cette opération peut échouer
        CALL risky_operation(v_id);

    END LOOP;

    CLOSE cur;

    SELECT CONCAT('Errors encountered: ', v_error_count) AS summary;
END$$

DELIMITER ;
```

---

## Exemples Pratiques

### 1. Curseur Simple : Traitement Séquentiel

```sql
DELIMITER $$

-- Appliquer une remise personnalisée par client
CREATE PROCEDURE apply_custom_discounts()
BEGIN
    DECLARE v_customer_id INT;
    DECLARE v_total_spent DECIMAL(15,2);
    DECLARE v_discount_rate DECIMAL(5,2);
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_processed INT DEFAULT 0;

    -- Curseur : clients actifs avec leur total dépensé
    DECLARE customer_cursor CURSOR FOR
        SELECT
            c.id,
            COALESCE(SUM(o.total_amount), 0) AS total_spent
        FROM customers c
        LEFT JOIN orders o ON c.id = o.customer_id
        WHERE c.is_active = TRUE
        GROUP BY c.id;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN customer_cursor;

    discount_loop: LOOP
        FETCH customer_cursor INTO v_customer_id, v_total_spent;

        IF v_done THEN
            LEAVE discount_loop;
        END IF;

        -- Calculer le taux de remise selon le total dépensé
        IF v_total_spent >= 10000 THEN
            SET v_discount_rate = 0.20;  -- 20% VIP
        ELSEIF v_total_spent >= 5000 THEN
            SET v_discount_rate = 0.15;  -- 15% Gold
        ELSEIF v_total_spent >= 1000 THEN
            SET v_discount_rate = 0.10;  -- 10% Silver
        ELSE
            SET v_discount_rate = 0.05;  -- 5% Bronze
        END IF;

        -- Appliquer la remise
        UPDATE customers
        SET
            discount_rate = v_discount_rate,
            updated_at = NOW()
        WHERE id = v_customer_id;

        SET v_processed = v_processed + 1;

    END LOOP discount_loop;

    CLOSE customer_cursor;

    SELECT CONCAT('Updated discount for ', v_processed, ' customers') AS result;
END$$

DELIMITER ;
```

### 2. Curseurs Imbriqués

```sql
DELIMITER $$

-- Générer un rapport détaillé par client et commande
CREATE PROCEDURE generate_customer_order_report()
BEGIN
    DECLARE v_customer_id INT;
    DECLARE v_customer_name VARCHAR(100);
    DECLARE v_order_id INT;
    DECLARE v_order_total DECIMAL(10,2);
    DECLARE v_done_customers BOOLEAN DEFAULT FALSE;
    DECLARE v_done_orders BOOLEAN DEFAULT FALSE;

    -- Curseur externe : Clients
    DECLARE customer_cursor CURSOR FOR
        SELECT id, name FROM customers WHERE is_active = TRUE;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done_customers = TRUE;

    -- Table temporaire pour le rapport
    CREATE TEMPORARY TABLE IF NOT EXISTS temp_report (
        customer_name VARCHAR(100),
        order_id INT,
        order_total DECIMAL(10,2)
    ) ENGINE=MEMORY;

    OPEN customer_cursor;

    customer_loop: LOOP
        FETCH customer_cursor INTO v_customer_id, v_customer_name;

        IF v_done_customers THEN
            LEAVE customer_loop;
        END IF;

        -- ═══════════════════════════════════════════════
        -- Curseur interne : Commandes du client
        -- ═══════════════════════════════════════════════
        BEGIN
            DECLARE order_cursor CURSOR FOR
                SELECT id, total_amount
                FROM orders
                WHERE customer_id = v_customer_id
                ORDER BY created_at DESC;

            DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done_orders = TRUE;

            SET v_done_orders = FALSE;
            OPEN order_cursor;

            order_loop: LOOP
                FETCH order_cursor INTO v_order_id, v_order_total;

                IF v_done_orders THEN
                    LEAVE order_loop;
                END IF;

                -- Insérer dans le rapport
                INSERT INTO temp_report (customer_name, order_id, order_total)
                VALUES (v_customer_name, v_order_id, v_order_total);

            END LOOP order_loop;

            CLOSE order_cursor;
        END;

    END LOOP customer_loop;

    CLOSE customer_cursor;

    -- Retourner le rapport
    SELECT * FROM temp_report ORDER BY customer_name, order_id;

    -- Nettoyer
    DROP TEMPORARY TABLE IF EXISTS temp_report;
END$$

DELIMITER ;
```

### 3. Curseur avec Traitement Conditionnel

```sql
DELIMITER $$

-- Traiter les commandes avec différentes logiques selon le statut
CREATE PROCEDURE process_orders_by_status()
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_status VARCHAR(20);
    DECLARE v_total_amount DECIMAL(10,2);
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_pending_count INT DEFAULT 0;
    DECLARE v_completed_count INT DEFAULT 0;
    DECLARE v_cancelled_count INT DEFAULT 0;

    DECLARE order_cursor CURSOR FOR
        SELECT id, status, total_amount
        FROM orders
        WHERE created_at >= CURDATE() - INTERVAL 7 DAY;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN order_cursor;

    process_loop: LOOP
        FETCH order_cursor INTO v_order_id, v_status, v_total_amount;

        IF v_done THEN
            LEAVE process_loop;
        END IF;

        -- Traitement selon le statut
        CASE v_status
            WHEN 'pending' THEN
                -- Envoyer un rappel
                INSERT INTO notifications (order_id, type, message)
                VALUES (v_order_id, 'REMINDER', 'Your order is pending payment');
                SET v_pending_count = v_pending_count + 1;

            WHEN 'completed' THEN
                -- Calculer les points de fidélité
                UPDATE customers c
                JOIN orders o ON c.id = o.customer_id
                SET c.loyalty_points = c.loyalty_points + FLOOR(v_total_amount)
                WHERE o.id = v_order_id;
                SET v_completed_count = v_completed_count + 1;

            WHEN 'cancelled' THEN
                -- Restaurer le stock
                UPDATE products p
                JOIN order_items oi ON p.id = oi.product_id
                SET p.stock = p.stock + oi.quantity
                WHERE oi.order_id = v_order_id;
                SET v_cancelled_count = v_cancelled_count + 1;

            ELSE
                -- Statut inconnu, logger
                INSERT INTO error_log (message, details)
                VALUES ('Unknown order status', CONCAT('Order ID: ', v_order_id, ', Status: ', v_status));
        END CASE;

    END LOOP process_loop;

    CLOSE order_cursor;

    -- Résumé
    SELECT
        v_pending_count AS pending_orders,
        v_completed_count AS completed_orders,
        v_cancelled_count AS cancelled_orders;
END$$

DELIMITER ;
```

---

## Curseurs vs Opérations Ensemblistes

### ❌ Mauvais : Utilisation de Curseur

```sql
DELIMITER $$

-- ❌ INEFFICACE : Mise à jour avec curseur
CREATE PROCEDURE bad_update_prices()
BEGIN
    DECLARE v_product_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    DECLARE product_cursor CURSOR FOR
        SELECT id FROM products WHERE category = 'electronics';

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN product_cursor;

    update_loop: LOOP
        FETCH product_cursor INTO v_product_id;

        IF v_done THEN
            LEAVE update_loop;
        END IF;

        -- ❌ Une requête par produit !
        UPDATE products
        SET price = price * 1.10
        WHERE id = v_product_id;

    END LOOP update_loop;

    CLOSE product_cursor;
END$$

DELIMITER ;
```

### ✅ Bon : Opération Ensembliste

```sql
DELIMITER $$

-- ✅ EFFICACE : Mise à jour ensembliste
CREATE PROCEDURE good_update_prices()
BEGIN
    -- Une seule requête pour tous les produits
    UPDATE products
    SET price = price * 1.10
    WHERE category = 'electronics';

    SELECT CONCAT('Updated ', ROW_COUNT(), ' products') AS result;
END$$

DELIMITER ;
```

### 📊 Comparaison de Performance

```sql
-- Test de performance : 10,000 produits

-- Curseur : ~8-12 secondes
CALL bad_update_prices();

-- Ensembliste : ~0.05 secondes
CALL good_update_prices();

-- ⚡ Opération ensembliste est 160-240x plus rapide !
```

---

## Quand Utiliser (ou ne pas utiliser) les Curseurs

### ✅ Cas Légitimes d'Utilisation

Les curseurs sont acceptables UNIQUEMENT quand :

#### 1. Logique Complexe Non-Ensembliste

```sql
DELIMITER $$

-- Algorithme complexe nécessitant un état entre les lignes
CREATE PROCEDURE calculate_running_balance()
BEGIN
    DECLARE v_transaction_id INT;
    DECLARE v_amount DECIMAL(10,2);
    DECLARE v_type VARCHAR(20);
    DECLARE v_running_balance DECIMAL(10,2) DEFAULT 0;
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    DECLARE transaction_cursor CURSOR FOR
        SELECT id, amount, type
        FROM transactions
        WHERE account_id = 123
        ORDER BY created_at;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN transaction_cursor;

    balance_loop: LOOP
        FETCH transaction_cursor INTO v_transaction_id, v_amount, v_type;

        IF v_done THEN
            LEAVE balance_loop;
        END IF;

        -- Calculer le solde cumulé
        IF v_type = 'credit' THEN
            SET v_running_balance = v_running_balance + v_amount;
        ELSE
            SET v_running_balance = v_running_balance - v_amount;
        END IF;

        -- Stocker le solde cumulé
        UPDATE transactions
        SET running_balance = v_running_balance
        WHERE id = v_transaction_id;

    END LOOP balance_loop;

    CLOSE transaction_cursor;
END$$

DELIMITER ;

-- Note : Même ceci peut souvent être fait avec des Window Functions !
-- SELECT
--     id,
--     amount,
--     SUM(CASE WHEN type = 'credit' THEN amount ELSE -amount END)
--         OVER (ORDER BY created_at) AS running_balance
-- FROM transactions
```

#### 2. Appel de Procédure par Ligne

```sql
DELIMITER $$

-- Besoin d'appeler une procédure pour chaque ligne
CREATE PROCEDURE process_each_customer()
BEGIN
    DECLARE v_customer_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    DECLARE customer_cursor CURSOR FOR
        SELECT id FROM customers WHERE needs_processing = TRUE;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN customer_cursor;

    process_loop: LOOP
        FETCH customer_cursor INTO v_customer_id;

        IF v_done THEN
            LEAVE process_loop;
        END IF;

        -- Appeler une procédure complexe par client
        CALL complex_customer_processing(v_customer_id);

    END LOOP process_loop;

    CLOSE customer_cursor;
END$$

DELIMITER ;
```

### ❌ Quand NE PAS Utiliser les Curseurs

N'utilisez JAMAIS de curseurs pour :

#### 1. Mises à Jour Simples

```sql
-- ❌ MAUVAIS
DECLARE cur CURSOR FOR SELECT id FROM products;
LOOP
    FETCH cur INTO v_id;
    UPDATE products SET price = price * 1.1 WHERE id = v_id;
END LOOP;

-- ✅ BON
UPDATE products SET price = price * 1.1;
```

#### 2. Insertions en Masse

```sql
-- ❌ MAUVAIS
DECLARE cur CURSOR FOR SELECT * FROM temp_data;
LOOP
    FETCH cur INTO v_col1, v_col2;
    INSERT INTO target_table VALUES (v_col1, v_col2);
END LOOP;

-- ✅ BON
INSERT INTO target_table SELECT * FROM temp_data;
```

#### 3. Agrégations

```sql
-- ❌ MAUVAIS
DECLARE v_total DECIMAL(10,2) DEFAULT 0;
DECLARE cur CURSOR FOR SELECT amount FROM orders;
LOOP
    FETCH cur INTO v_amount;
    SET v_total = v_total + v_amount;
END LOOP;

-- ✅ BON
SELECT SUM(amount) INTO v_total FROM orders;
```

#### 4. Calculs Conditionnels

```sql
-- ❌ MAUVAIS
DECLARE cur CURSOR FOR SELECT id, status FROM orders;
LOOP
    FETCH cur INTO v_id, v_status;
    IF v_status = 'pending' THEN
        UPDATE orders SET priority = 'high' WHERE id = v_id;
    END IF;
END LOOP;

-- ✅ BON
UPDATE orders
SET priority = 'high'
WHERE status = 'pending';
```

---

## Bonnes Pratiques

### 1. ✅ Toujours Fermer les Curseurs

```sql
DELIMITER $$

CREATE PROCEDURE proper_cursor_cleanup()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE my_cursor CURSOR FOR SELECT id FROM products;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    -- Handler d'erreur pour garantir le nettoyage
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Fermer le curseur même en cas d'erreur
        CLOSE my_cursor;
        ROLLBACK;
    END;

    OPEN my_cursor;

    read_loop: LOOP
        FETCH my_cursor INTO v_id;
        IF v_done THEN
            LEAVE read_loop;
        END IF;
        -- Traitement
    END LOOP;

    CLOSE my_cursor;  -- ✅ Toujours fermer
END$$

DELIMITER ;
```

### 2. ✅ Limiter la Taille du Résultat

```sql
DELIMITER $$

-- ❌ MAUVAIS : Curseur sur toute la table
CREATE PROCEDURE bad_unlimited_cursor()
BEGIN
    DECLARE cur CURSOR FOR
        SELECT * FROM huge_table;  -- Peut contenir des millions de lignes !
    -- ...
END$$

-- ✅ BON : Limiter avec WHERE et LIMIT
CREATE PROCEDURE good_limited_cursor()
BEGIN
    DECLARE cur CURSOR FOR
        SELECT id, name
        FROM huge_table
        WHERE created_at >= CURDATE() - INTERVAL 7 DAY
        LIMIT 1000;  -- Limite raisonnable
    -- ...
END$$

DELIMITER ;
```

### 3. ✅ Traiter par Batch

```sql
DELIMITER $$

-- Traiter de grandes quantités de données par batch
CREATE PROCEDURE process_by_batch()
BEGIN
    DECLARE v_offset INT DEFAULT 0;
    DECLARE v_batch_size INT DEFAULT 1000;
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    WHILE NOT v_done DO
        -- Curseur sur un batch
        BEGIN
            DECLARE v_id INT;
            DECLARE v_count INT DEFAULT 0;
            DECLARE batch_done BOOLEAN DEFAULT FALSE;

            DECLARE batch_cursor CURSOR FOR
                SELECT id FROM orders
                WHERE status = 'pending'
                ORDER BY id
                LIMIT v_batch_size OFFSET v_offset;

            DECLARE CONTINUE HANDLER FOR NOT FOUND SET batch_done = TRUE;

            OPEN batch_cursor;

            batch_loop: LOOP
                FETCH batch_cursor INTO v_id;

                IF batch_done THEN
                    LEAVE batch_loop;
                END IF;

                -- Traiter la ligne
                CALL process_order(v_id);
                SET v_count = v_count + 1;

            END LOOP batch_loop;

            CLOSE batch_cursor;

            -- Si moins d'un batch complet, on a fini
            IF v_count < v_batch_size THEN
                SET v_done = TRUE;
            ELSE
                SET v_offset = v_offset + v_batch_size;
            END IF;
        END;
    END WHILE;
END$$

DELIMITER ;
```

### 4. ✅ Logger les Progrès

```sql
DELIMITER $$

CREATE PROCEDURE cursor_with_logging()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_processed INT DEFAULT 0;
    DECLARE v_start_time DATETIME;

    DECLARE my_cursor CURSOR FOR SELECT id FROM large_table;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    SET v_start_time = NOW();

    OPEN my_cursor;

    process_loop: LOOP
        FETCH my_cursor INTO v_id;

        IF v_done THEN
            LEAVE process_loop;
        END IF;

        -- Traiter
        CALL process_record(v_id);
        SET v_processed = v_processed + 1;

        -- Logger tous les 1000 enregistrements
        IF v_processed % 1000 = 0 THEN
            INSERT INTO processing_log (records_processed, elapsed_seconds)
            VALUES (v_processed, TIMESTAMPDIFF(SECOND, v_start_time, NOW()));
        END IF;

    END LOOP process_loop;

    CLOSE my_cursor;

    -- Log final
    INSERT INTO processing_log (records_processed, elapsed_seconds, status)
    VALUES (v_processed, TIMESTAMPDIFF(SECOND, v_start_time, NOW()), 'COMPLETED');
END$$

DELIMITER ;
```

### 5. ✅ Documenter Pourquoi un Curseur est Nécessaire

```sql
DELIMITER $$

CREATE PROCEDURE justified_cursor_usage()
BEGIN
    -- ============================================
    -- NOTE : Ce curseur est nécessaire car :
    -- 1. Chaque ligne nécessite un appel à une API externe (via UDF)
    -- 2. La logique de validation dépend de l'état de la ligne précédente
    -- 3. Impossible à faire avec une opération ensembliste
    --
    -- Alternative évaluée : Window Functions
    -- Raison du rejet : Nécessite appel externe par ligne
    --
    -- Performance : ~5 secondes pour 1000 lignes
    -- Dernière optimisation : 2025-12-12
    -- ============================================

    -- Code du curseur
    -- ...
END$$

DELIMITER ;
```

---

## Optimisation des Performances

### Techniques d'Optimisation

```sql
DELIMITER $$

-- 1. ✅ Sélectionner uniquement les colonnes nécessaires
CREATE PROCEDURE optimized_cursor_columns()
BEGIN
    -- ❌ MAUVAIS
    DECLARE cur1 CURSOR FOR SELECT * FROM large_table;

    -- ✅ BON
    DECLARE cur2 CURSOR FOR SELECT id, name FROM large_table;
END$$

-- 2. ✅ Utiliser des index appropriés
CREATE PROCEDURE optimized_cursor_index()
BEGIN
    -- S'assurer qu'un index existe sur created_at
    -- CREATE INDEX idx_created ON orders(created_at);

    DECLARE cur CURSOR FOR
        SELECT id FROM orders
        WHERE created_at >= CURDATE() - INTERVAL 7 DAY
        ORDER BY created_at;  -- Peut utiliser l'index
END$$

-- 3. ✅ Traiter en transactions par batch
CREATE PROCEDURE optimized_cursor_transactions()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_counter INT DEFAULT 0;

    DECLARE cur CURSOR FOR SELECT id FROM orders;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN cur;

    START TRANSACTION;

    process_loop: LOOP
        FETCH cur INTO v_id;

        IF v_done THEN
            COMMIT;  -- Commit final
            LEAVE process_loop;
        END IF;

        -- Traiter
        UPDATE orders SET processed = TRUE WHERE id = v_id;
        SET v_counter = v_counter + 1;

        -- Commit tous les 100 enregistrements
        IF v_counter % 100 = 0 THEN
            COMMIT;
            START TRANSACTION;
        END IF;

    END LOOP process_loop;

    CLOSE cur;
END$$

DELIMITER ;
```

---

## Erreurs Courantes

### 1. ❌ Oublier le Handler NOT FOUND

```sql
-- ❌ ERREUR : Boucle infinie
CREATE PROCEDURE missing_handler()
BEGIN
    DECLARE v_id INT;
    DECLARE cur CURSOR FOR SELECT id FROM products;

    -- Manque : DECLARE CONTINUE HANDLER FOR NOT FOUND

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_id;
        -- Pas de condition de sortie !
        -- Boucle infinie !
    END LOOP;

    CLOSE cur;
END;
```

### 2. ❌ Fetch Après Vérification

```sql
-- ❌ ERREUR : Vérifier AVANT de traiter
CREATE PROCEDURE wrong_order()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE cur CURSOR FOR SELECT id FROM products;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN cur;

    read_loop: LOOP
        IF v_done THEN
            LEAVE read_loop;
        END IF;

        FETCH cur INTO v_id;  -- ❌ Fetch après vérification

        -- Traiter v_id (peut contenir NULL ou valeur invalide)
    END LOOP;

    CLOSE cur;
END;

-- ✅ CORRECT : Fetch PUIS vérifier
CREATE PROCEDURE correct_order()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE cur CURSOR FOR SELECT id FROM products;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_id;  -- ✅ Fetch en premier

        IF v_done THEN
            LEAVE read_loop;
        END IF;

        -- Traiter v_id (valeur garantie valide)
    END LOOP;

    CLOSE cur;
END;
```

### 3. ❌ Ne pas Fermer le Curseur

```sql
-- ❌ ERREUR : Fuite de ressources
CREATE PROCEDURE unclosed_cursor()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE cur CURSOR FOR SELECT id FROM products;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_id;
        IF v_done THEN
            LEAVE read_loop;
        END IF;
        -- Traiter
    END LOOP;

    -- Manque : CLOSE cur;
END;
```

---

## ✅ Points clés à retenir

- 🚫 **Éviter autant que possible** : Les curseurs sont lents et complexes
- 🎯 **Préférer SET-based** : UPDATE/INSERT/DELETE ensembliste est 100-1000x plus rapide
- 📋 **Cycle de vie** : DECLARE → OPEN → FETCH (loop) → CLOSE
- 🔄 **Handler obligatoire** : DECLARE CONTINUE HANDLER FOR NOT FOUND
- ✅ **Cas légitimes** : Logique complexe non-ensembliste, appels procédure par ligne
- 🔧 **Optimisation** : Limiter résultats, sélectionner colonnes nécessaires, traiter par batch
- 📝 **Documentation** : Toujours justifier l'utilisation d'un curseur
- 🐛 **Erreurs courantes** : Oublier le handler, mauvais ordre fetch/vérification, ne pas fermer

---

## 🔗 Ressources et références

- [📖 Cursors - MariaDB Documentation](https://mariadb.com/kb/en/cursors/)
- [📖 DECLARE CURSOR - MariaDB Documentation](https://mariadb.com/kb/en/declare-cursor/)
- [📖 FETCH - MariaDB Documentation](https://mariadb.com/kb/en/fetch/)
- [📖 OPEN - MariaDB Documentation](https://mariadb.com/kb/en/open/)
- [📖 CLOSE - MariaDB Documentation](https://mariadb.com/kb/en/close/)
- [📝 Performance: Row-by-Row vs Set-Based Processing](https://mariadb.com/kb/en/optimization-and-indexes/)

---

## ➡️ Section suivante

**8.6 Gestion des erreurs et exceptions** : DECLARE HANDLER, SIGNAL, RESIGNAL, gestion transactionnelle des erreurs, et stratégies de rollback.

---


⏭️ [Gestion des erreurs et exceptions (DECLARE HANDLER)](/08-programmation-cote-serveur/06-gestion-erreurs-exceptions.md)

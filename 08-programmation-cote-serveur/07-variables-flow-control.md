🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.7 Variables et Flow Control

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Procédures stockées (8.1), Fonctions (8.2), SQL de base (chapitres 2-3)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Déclarer** et utiliser différents types de variables (locales, session, globales)
- **Maîtriser** les structures conditionnelles (IF, CASE)
- **Utiliser** les boucles (LOOP, WHILE, REPEAT) efficacement
- **Comprendre** la portée et le cycle de vie des variables
- **Contrôler** le flux d'exécution avec LEAVE et ITERATE
- **Optimiser** les performances en choisissant les bonnes structures
- **Appliquer** les bonnes pratiques de programmation structurée

---

## Introduction

Les **variables** et les **structures de contrôle** sont les fondations de la programmation dans les procédures stockées, fonctions et triggers. Elles permettent de créer une logique conditionnelle complexe et des traitements itératifs.

### 🔍 Importance du Flow Control

Sans structures de contrôle :
- ❌ Code séquentiel uniquement
- ❌ Pas de logique conditionnelle
- ❌ Pas de répétitions
- ❌ Limité aux opérations SQL simples

Avec structures de contrôle :
- ✅ Logique métier complexe
- ✅ Décisions basées sur les données
- ✅ Traitements itératifs
- ✅ Code maintenable et lisible

---

## Variables

### Types de Variables

MariaDB supporte trois catégories de variables dans les procédures stockées.

#### 1️⃣ Variables Locales (DECLARE)

Déclarées dans un bloc BEGIN/END, visible uniquement dans ce bloc.

```sql
DELIMITER $$

CREATE PROCEDURE local_variables_example()
BEGIN
    -- Déclarations de variables locales
    DECLARE v_counter INT DEFAULT 0;
    DECLARE v_name VARCHAR(100);
    DECLARE v_total DECIMAL(10,2) DEFAULT 0.0;
    DECLARE v_is_active BOOLEAN DEFAULT TRUE;
    DECLARE v_created_date DATETIME DEFAULT NOW();

    -- Utilisation
    SET v_counter = 10;
    SET v_name = 'John Doe';

    -- SELECT INTO
    SELECT COUNT(*), SUM(amount)
    INTO v_counter, v_total
    FROM orders
    WHERE status = 'completed';

    SELECT v_counter AS total_orders, v_total AS revenue;
END$$

DELIMITER ;
```

**Caractéristiques** :
- ✅ Scope limité au bloc BEGIN/END
- ✅ Doivent être déclarées au début du bloc
- ✅ Initialisées avec DEFAULT ou NULL
- ✅ Disparaissent à la fin du bloc

#### 2️⃣ Variables de Session (@variable)

Variables utilisateur accessibles pendant toute la session.

```sql
-- Définir une variable de session
SET @customer_id = 123;
SET @discount_rate = 0.15;

-- Utiliser dans une requête
SELECT * FROM orders WHERE customer_id = @customer_id;

-- Utiliser dans une procédure
DELIMITER $$
CREATE PROCEDURE session_variables_example()
BEGIN
    -- Lire une variable de session
    IF @customer_id IS NOT NULL THEN
        SELECT * FROM customers WHERE id = @customer_id;
    END IF;

    -- Modifier une variable de session
    SET @last_order_id = LAST_INSERT_ID();
END$$
DELIMITER ;

-- Les variables de session persistent
CALL session_variables_example();
SELECT @last_order_id;  -- Toujours accessible
```

**Caractéristiques** :
- ✅ Scope : toute la session
- ✅ Pas besoin de DECLARE
- ✅ Persistent jusqu'à la déconnexion
- ⚠️ Non typées (conversion automatique)

#### 3️⃣ Variables Globales (@@variable)

Variables système configurant le comportement de MariaDB.

```sql
-- Lire une variable globale
SELECT @@max_connections;
SELECT @@version;
SELECT @@sql_mode;

-- Modifier une variable globale (avec privilèges SUPER)
SET GLOBAL max_connections = 200;

-- Modifier pour la session courante
SET SESSION sql_mode = 'STRICT_ALL_TABLES';
-- ou
SET @@session.sql_mode = 'STRICT_ALL_TABLES';

DELIMITER $$
CREATE PROCEDURE check_sql_mode()
BEGIN
    DECLARE v_sql_mode VARCHAR(1000);

    -- Lire la variable globale
    SET v_sql_mode = @@sql_mode;

    SELECT v_sql_mode AS current_sql_mode;
END$$
DELIMITER ;
```

**Caractéristiques** :
- ✅ Configuration du serveur
- ⚠️ Scope GLOBAL ou SESSION
- ⚠️ Modifications nécessitent des privilèges

### 📊 Tableau Comparatif

| Type | Syntaxe | Scope | Persistance | Usage |
|------|---------|-------|-------------|-------|
| **Locale** | `DECLARE v_name TYPE` | Bloc BEGIN/END | Fin du bloc | Variables temporaires |
| **Session** | `@variable` | Session | Déconnexion | Communication entre requêtes |
| **Globale** | `@@variable` | Server/Session | Permanent/Session | Configuration |

---

## Assignation de Valeurs

### Méthode 1 : SET

```sql
DELIMITER $$

CREATE PROCEDURE set_examples()
BEGIN
    DECLARE v_counter INT;
    DECLARE v_name VARCHAR(100);
    DECLARE v_total DECIMAL(10,2);

    -- Assignation simple
    SET v_counter = 0;

    -- Assignation avec expression
    SET v_counter = v_counter + 1;

    -- Assignation multiple
    SET v_name = 'John', v_total = 100.50;

    -- Assignation conditionnelle
    SET v_total = IF(v_counter > 0, v_counter * 10, 0);

    SELECT v_counter, v_name, v_total;
END$$

DELIMITER ;
```

### Méthode 2 : SELECT INTO

```sql
DELIMITER $$

CREATE PROCEDURE select_into_examples()
BEGIN
    DECLARE v_customer_name VARCHAR(100);
    DECLARE v_order_count INT;
    DECLARE v_total_spent DECIMAL(15,2);

    -- Une seule colonne
    SELECT name INTO v_customer_name
    FROM customers
    WHERE id = 1;

    -- Plusieurs colonnes
    SELECT
        COUNT(*),
        COALESCE(SUM(total_amount), 0)
    INTO
        v_order_count,
        v_total_spent
    FROM orders
    WHERE customer_id = 1;

    SELECT
        v_customer_name AS customer,
        v_order_count AS orders,
        v_total_spent AS spent;
END$$

DELIMITER ;
```

⚠️ **Attention** : SELECT INTO lève une erreur si :
- Aucune ligne n'est retournée (sauf avec handler NOT FOUND)
- Plus d'une ligne est retournée

```sql
DELIMITER $$

CREATE PROCEDURE safe_select_into()
BEGIN
    DECLARE v_name VARCHAR(100);
    DECLARE v_found BOOLEAN DEFAULT TRUE;

    -- Handler pour absence de résultat
    DECLARE CONTINUE HANDLER FOR NOT FOUND
        SET v_found = FALSE;

    -- SELECT INTO sécurisé
    SELECT name INTO v_name
    FROM customers
    WHERE id = 999;  -- Peut ne pas exister

    IF v_found THEN
        SELECT CONCAT('Found: ', v_name) AS result;
    ELSE
        SELECT 'Customer not found' AS result;
    END IF;
END$$

DELIMITER ;
```

---

## Portée des Variables

### Portée dans les Blocs Imbriqués

```sql
DELIMITER $$

CREATE PROCEDURE variable_scope_demo()
BEGIN
    DECLARE v_outer INT DEFAULT 10;

    SELECT v_outer AS outer_start;  -- 10

    -- Bloc interne 1
    BEGIN
        DECLARE v_inner INT DEFAULT 20;

        SELECT
            v_outer AS outer_value,   -- 10 (accessible)
            v_inner AS inner_value;   -- 20

        SET v_outer = 30;  -- Modifie la variable externe
    END;

    SELECT v_outer AS outer_modified;  -- 30
    -- SELECT v_inner;  ❌ Erreur : v_inner n'existe plus

    -- Bloc interne 2 (v_inner réutilisable)
    BEGIN
        DECLARE v_inner INT DEFAULT 40;
        SELECT v_inner AS inner_value_2;  -- 40
    END;
END$$

DELIMITER ;
```

### Masquage de Variables

```sql
DELIMITER $$

CREATE PROCEDURE variable_shadowing()
BEGIN
    DECLARE v_value INT DEFAULT 100;

    SELECT v_value AS outer_value;  -- 100

    BEGIN
        -- Variable locale masque la variable externe
        DECLARE v_value INT DEFAULT 200;

        SELECT v_value AS inner_value;  -- 200 (masque l'externe)
    END;

    SELECT v_value AS outer_value_again;  -- 100 (inchangé)
END$$

DELIMITER ;
```

💡 **Bonne pratique** : Évitez le masquage de variables pour la lisibilité.

---

## Structure IF

### Syntaxe de Base

```sql
IF condition THEN
    -- Instructions
ELSEIF autre_condition THEN
    -- Instructions
ELSE
    -- Instructions
END IF;
```

### Exemples

```sql
DELIMITER $$

-- IF simple
CREATE PROCEDURE if_simple(IN p_age INT)
BEGIN
    IF p_age >= 18 THEN
        SELECT 'Adult' AS category;
    ELSE
        SELECT 'Minor' AS category;
    END IF;
END$$

-- IF avec ELSEIF
CREATE PROCEDURE if_elseif(IN p_score INT)
BEGIN
    DECLARE v_grade CHAR(1);

    IF p_score >= 90 THEN
        SET v_grade = 'A';
    ELSEIF p_score >= 80 THEN
        SET v_grade = 'B';
    ELSEIF p_score >= 70 THEN
        SET v_grade = 'C';
    ELSEIF p_score >= 60 THEN
        SET v_grade = 'D';
    ELSE
        SET v_grade = 'F';
    END IF;

    SELECT v_grade AS grade;
END$$

-- IF avec conditions complexes
CREATE PROCEDURE if_complex(
    IN p_amount DECIMAL(10,2),
    IN p_customer_type VARCHAR(20)
)
BEGIN
    DECLARE v_discount DECIMAL(5,2);

    IF p_amount > 1000 AND p_customer_type = 'VIP' THEN
        SET v_discount = 0.20;
    ELSEIF p_amount > 500 OR p_customer_type = 'GOLD' THEN
        SET v_discount = 0.10;
    ELSE
        SET v_discount = 0.05;
    END IF;

    SELECT
        p_amount AS original_amount,
        v_discount AS discount_rate,
        p_amount * (1 - v_discount) AS final_amount;
END$$

DELIMITER ;
```

### IF vs Fonction IF()

```sql
-- Structure IF
IF condition THEN
    SET variable = value1;
ELSE
    SET variable = value2;
END IF;

-- Fonction IF() (plus concis pour assignations simples)
SET variable = IF(condition, value1, value2);

-- Exemple
DELIMITER $$
CREATE PROCEDURE if_comparison(IN p_value INT)
BEGIN
    DECLARE v_result1 VARCHAR(20);
    DECLARE v_result2 VARCHAR(20);

    -- Avec structure IF
    IF p_value > 0 THEN
        SET v_result1 = 'Positive';
    ELSE
        SET v_result1 = 'Not positive';
    END IF;

    -- Avec fonction IF()
    SET v_result2 = IF(p_value > 0, 'Positive', 'Not positive');

    SELECT v_result1, v_result2;
END$$
DELIMITER ;
```

💡 **Conseil** : Utilisez la fonction `IF()` pour les assignations simples, la structure `IF` pour la logique complexe.

---

## Structure CASE

### CASE Expression

```sql
CASE expression
    WHEN valeur1 THEN resultat1
    WHEN valeur2 THEN resultat2
    ELSE resultat_defaut
END CASE;
```

### CASE Condition

```sql
CASE
    WHEN condition1 THEN resultat1
    WHEN condition2 THEN resultat2
    ELSE resultat_defaut
END CASE;
```

### Exemples

```sql
DELIMITER $$

-- CASE avec expression (switch-like)
CREATE PROCEDURE case_expression(IN p_day_number INT)
BEGIN
    DECLARE v_day_name VARCHAR(20);

    SET v_day_name = CASE p_day_number
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
        WHEN 7 THEN 'Sunday'
        ELSE 'Invalid day'
    END;

    SELECT v_day_name AS day;
END$$

-- CASE avec conditions (if-elseif-like)
CREATE PROCEDURE case_condition(IN p_amount DECIMAL(10,2))
BEGIN
    DECLARE v_category VARCHAR(20);

    SET v_category = CASE
        WHEN p_amount < 0 THEN 'Invalid'
        WHEN p_amount = 0 THEN 'Zero'
        WHEN p_amount < 100 THEN 'Small'
        WHEN p_amount < 1000 THEN 'Medium'
        WHEN p_amount < 10000 THEN 'Large'
        ELSE 'Very Large'
    END;

    SELECT v_category AS category;
END$$

-- CASE dans les instructions (pas seulement assignations)
CREATE PROCEDURE case_in_statements(IN p_status VARCHAR(20))
BEGIN
    CASE p_status
        WHEN 'pending' THEN
            UPDATE orders SET priority = 'high' WHERE status = 'pending';
        WHEN 'processing' THEN
            CALL notify_warehouse();
        WHEN 'shipped' THEN
            INSERT INTO shipment_log (message) VALUES ('Shipment recorded');
        ELSE
            SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Unknown status';
    END CASE;

    SELECT 'Status processed' AS result;
END$$

DELIMITER ;
```

### CASE vs IF

```sql
DELIMITER $$

-- Avec IF (verbeux)
CREATE PROCEDURE with_if(IN p_value INT)
BEGIN
    DECLARE v_result VARCHAR(20);

    IF p_value = 1 THEN
        SET v_result = 'One';
    ELSEIF p_value = 2 THEN
        SET v_result = 'Two';
    ELSEIF p_value = 3 THEN
        SET v_result = 'Three';
    ELSE
        SET v_result = 'Other';
    END IF;

    SELECT v_result;
END$$

-- Avec CASE (plus lisible)
CREATE PROCEDURE with_case(IN p_value INT)
BEGIN
    DECLARE v_result VARCHAR(20);

    SET v_result = CASE p_value
        WHEN 1 THEN 'One'
        WHEN 2 THEN 'Two'
        WHEN 3 THEN 'Three'
        ELSE 'Other'
    END;

    SELECT v_result;
END$$

DELIMITER ;
```

💡 **Conseil** : Préférez `CASE` pour tester une valeur contre plusieurs constantes, `IF` pour des conditions booléennes complexes.

---

## Boucles

### LOOP - Boucle Infinie avec Sortie

```sql
[label:] LOOP
    -- Instructions
    IF condition THEN
        LEAVE label;  -- Sortir de la boucle
    END IF;
END LOOP [label];
```

#### Exemples LOOP

```sql
DELIMITER $$

-- Boucle simple avec compteur
CREATE PROCEDURE loop_basic()
BEGIN
    DECLARE v_counter INT DEFAULT 0;

    counter_loop: LOOP
        SET v_counter = v_counter + 1;

        -- Condition de sortie
        IF v_counter > 10 THEN
            LEAVE counter_loop;
        END IF;

        INSERT INTO log_table (message, counter)
        VALUES ('Loop iteration', v_counter);
    END LOOP counter_loop;

    SELECT CONCAT('Completed ', v_counter, ' iterations') AS result;
END$$

-- Boucle avec ITERATE (continue)
CREATE PROCEDURE loop_with_iterate()
BEGIN
    DECLARE v_i INT DEFAULT 0;

    number_loop: LOOP
        SET v_i = v_i + 1;

        IF v_i > 20 THEN
            LEAVE number_loop;
        END IF;

        -- Sauter les nombres pairs
        IF v_i % 2 = 0 THEN
            ITERATE number_loop;  -- Continue à l'itération suivante
        END IF;

        -- Traiter uniquement les nombres impairs
        INSERT INTO odd_numbers (value) VALUES (v_i);
    END LOOP number_loop;
END$$

DELIMITER ;
```

### WHILE - Boucle avec Condition d'Entrée

```sql
WHILE condition DO
    -- Instructions
END WHILE;
```

#### Exemples WHILE

```sql
DELIMITER $$

-- WHILE simple
CREATE PROCEDURE while_basic()
BEGIN
    DECLARE v_counter INT DEFAULT 1;

    WHILE v_counter <= 10 DO
        INSERT INTO numbers (value) VALUES (v_counter);
        SET v_counter = v_counter + 1;
    END WHILE;

    SELECT 'Numbers 1-10 inserted' AS result;
END$$

-- WHILE avec condition complexe
CREATE PROCEDURE while_complex()
BEGIN
    DECLARE v_balance DECIMAL(10,2) DEFAULT 1000.00;
    DECLARE v_iteration INT DEFAULT 0;
    DECLARE v_interest_rate DECIMAL(5,4) DEFAULT 0.05;

    -- Calculer combien d'années pour doubler avec intérêts composés
    WHILE v_balance < 2000.00 AND v_iteration < 100 DO
        SET v_balance = v_balance * (1 + v_interest_rate);
        SET v_iteration = v_iteration + 1;
    END WHILE;

    SELECT
        v_iteration AS years_to_double,
        ROUND(v_balance, 2) AS final_balance;
END$$

-- WHILE avec curseur (pattern courant)
CREATE PROCEDURE while_with_cursor()
BEGIN
    DECLARE v_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_processed INT DEFAULT 0;

    DECLARE my_cursor CURSOR FOR SELECT id FROM orders WHERE status = 'pending';
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN my_cursor;

    WHILE NOT v_done DO
        FETCH my_cursor INTO v_id;

        IF NOT v_done THEN
            CALL process_order(v_id);
            SET v_processed = v_processed + 1;
        END IF;
    END WHILE;

    CLOSE my_cursor;

    SELECT CONCAT('Processed ', v_processed, ' orders') AS result;
END$$

DELIMITER ;
```

### REPEAT - Boucle avec Condition de Sortie

```sql
REPEAT
    -- Instructions
UNTIL condition
END REPEAT;
```

#### Exemples REPEAT

```sql
DELIMITER $$

-- REPEAT simple (équivalent do-while)
CREATE PROCEDURE repeat_basic()
BEGIN
    DECLARE v_counter INT DEFAULT 0;

    REPEAT
        SET v_counter = v_counter + 1;
        INSERT INTO log_table (counter) VALUES (v_counter);
    UNTIL v_counter >= 10
    END REPEAT;

    SELECT 'Loop executed' AS result;
END$$

-- REPEAT avec logique métier
CREATE PROCEDURE repeat_business_logic()
BEGIN
    DECLARE v_stock INT;
    DECLARE v_reorder_count INT DEFAULT 0;

    REPEAT
        -- Vérifier le stock
        SELECT stock INTO v_stock FROM products WHERE id = 1;

        -- Réapprovisionner si nécessaire
        IF v_stock < 100 THEN
            UPDATE products SET stock = stock + 50 WHERE id = 1;
            SET v_reorder_count = v_reorder_count + 1;
        END IF;

    UNTIL v_stock >= 100
    END REPEAT;

    SELECT
        v_reorder_count AS reorders_made,
        v_stock AS final_stock;
END$$

DELIMITER ;
```

### 📊 Comparaison des Boucles

| Type | Condition | Exécution Minimale | Usage Typique |
|------|-----------|-------------------|---------------|
| **LOOP** | Sortie avec LEAVE | 0 (avec IF immédiat) | Boucles complexes, ITERATE |
| **WHILE** | Au début | 0 | Condition d'entrée connue |
| **REPEAT** | À la fin | 1 | Au moins une exécution nécessaire |

### Équivalences

```sql
-- Ces trois boucles sont équivalentes
DELIMITER $$

-- Avec LOOP
CREATE PROCEDURE loop_version()
BEGIN
    DECLARE i INT DEFAULT 0;
    my_loop: LOOP
        SET i = i + 1;
        IF i > 10 THEN LEAVE my_loop; END IF;
        -- Traitement
    END LOOP;
END$$

-- Avec WHILE
CREATE PROCEDURE while_version()
BEGIN
    DECLARE i INT DEFAULT 0;
    WHILE i < 10 DO
        SET i = i + 1;
        -- Traitement
    END WHILE;
END$$

-- Avec REPEAT
CREATE PROCEDURE repeat_version()
BEGIN
    DECLARE i INT DEFAULT 0;
    REPEAT
        SET i = i + 1;
        -- Traitement
    UNTIL i >= 10
    END REPEAT;
END$$

DELIMITER ;
```

---

## Contrôle de Flux : LEAVE et ITERATE

### LEAVE - Sortir d'un Bloc ou Boucle

```sql
DELIMITER $$

-- Sortir d'une boucle
CREATE PROCEDURE leave_loop_example()
BEGIN
    DECLARE v_i INT DEFAULT 0;

    search_loop: LOOP
        SET v_i = v_i + 1;

        -- Trouver un enregistrement spécifique
        IF EXISTS(SELECT 1 FROM orders WHERE id = v_i AND status = 'pending') THEN
            SELECT CONCAT('Found pending order: ', v_i) AS result;
            LEAVE search_loop;  -- Sortir immédiatement
        END IF;

        IF v_i > 1000 THEN
            SELECT 'No pending order found' AS result;
            LEAVE search_loop;
        END IF;
    END LOOP search_loop;
END$$

-- Sortir d'un bloc BEGIN/END
CREATE PROCEDURE leave_block_example(IN p_customer_id INT)
BEGIN
    DECLARE v_status VARCHAR(20);

    validation: BEGIN
        -- Vérifier que le client existe
        SELECT status INTO v_status FROM customers WHERE id = p_customer_id;

        IF v_status IS NULL THEN
            SELECT 'Customer not found' AS error;
            LEAVE validation;  -- Sortir du bloc validation
        END IF;

        IF v_status != 'active' THEN
            SELECT 'Customer not active' AS error;
            LEAVE validation;
        END IF;

        -- Traitement normal
        SELECT 'Processing customer' AS message;
    END validation;

    -- Code après le bloc (toujours exécuté)
    SELECT 'End of procedure' AS message;
END$$

DELIMITER ;
```

### ITERATE - Continuer à la Prochaine Itération

```sql
DELIMITER $$

-- ITERATE dans LOOP
CREATE PROCEDURE iterate_loop_example()
BEGIN
    DECLARE v_i INT DEFAULT 0;

    process_loop: LOOP
        SET v_i = v_i + 1;

        IF v_i > 20 THEN
            LEAVE process_loop;
        END IF;

        -- Sauter les multiples de 3
        IF v_i % 3 = 0 THEN
            ITERATE process_loop;  -- Continuer à l'itération suivante
        END IF;

        -- Traiter uniquement les nombres non-multiples de 3
        INSERT INTO processed_numbers (value) VALUES (v_i);
    END LOOP process_loop;
END$$

-- ITERATE dans WHILE
CREATE PROCEDURE iterate_while_example()
BEGIN
    DECLARE v_id INT;
    DECLARE v_status VARCHAR(20);
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    DECLARE order_cursor CURSOR FOR SELECT id, status FROM orders;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN order_cursor;

    order_loop: WHILE NOT v_done DO
        FETCH order_cursor INTO v_id, v_status;

        IF v_done THEN
            LEAVE order_loop;
        END IF;

        -- Sauter les commandes déjà traitées
        IF v_status = 'completed' THEN
            ITERATE order_loop;
        END IF;

        -- Traiter uniquement les commandes non-complétées
        CALL process_order(v_id);
    END WHILE order_loop;

    CLOSE order_cursor;
END$$

DELIMITER ;
```

---

## Exemples Pratiques Complets

### 1. Calculateur de Remise Progressive

```sql
DELIMITER $$

CREATE PROCEDURE calculate_tiered_discount(
    IN p_customer_id INT,
    IN p_base_amount DECIMAL(10,2),
    OUT p_final_amount DECIMAL(10,2),
    OUT p_discount_applied DECIMAL(5,2)
)
BEGIN
    DECLARE v_total_spent DECIMAL(15,2);
    DECLARE v_order_count INT;
    DECLARE v_is_vip BOOLEAN;
    DECLARE v_discount DECIMAL(5,2) DEFAULT 0.00;

    -- Récupérer les informations du client
    SELECT
        COALESCE(SUM(o.total_amount), 0),
        COUNT(o.id),
        c.is_vip
    INTO
        v_total_spent,
        v_order_count,
        v_is_vip
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    WHERE c.id = p_customer_id
    GROUP BY c.is_vip;

    -- Calculer la remise selon plusieurs critères

    -- Critère 1 : Total dépensé
    IF v_total_spent >= 10000 THEN
        SET v_discount = 0.20;
    ELSEIF v_total_spent >= 5000 THEN
        SET v_discount = 0.15;
    ELSEIF v_total_spent >= 1000 THEN
        SET v_discount = 0.10;
    END IF;

    -- Critère 2 : Nombre de commandes (bonus supplémentaire)
    IF v_order_count >= 50 THEN
        SET v_discount = v_discount + 0.05;
    ELSEIF v_order_count >= 20 THEN
        SET v_discount = v_discount + 0.03;
    END IF;

    -- Critère 3 : Statut VIP
    IF v_is_vip THEN
        SET v_discount = v_discount + 0.05;
    END IF;

    -- Plafonner à 30%
    IF v_discount > 0.30 THEN
        SET v_discount = 0.30;
    END IF;

    -- Calculer le montant final
    SET p_discount_applied = v_discount;
    SET p_final_amount = p_base_amount * (1 - v_discount);
END$$

DELIMITER ;

-- Utilisation
CALL calculate_tiered_discount(123, 500.00, @final, @discount);
SELECT @final AS final_amount, @discount * 100 AS discount_percent;
```

### 2. Générateur de Rapports avec Boucle

```sql
DELIMITER $$

CREATE PROCEDURE generate_monthly_report(IN p_year INT, IN p_month INT)
BEGIN
    DECLARE v_day INT DEFAULT 1;
    DECLARE v_days_in_month INT;
    DECLARE v_current_date DATE;
    DECLARE v_daily_total DECIMAL(15,2);
    DECLARE v_order_count INT;

    -- Table temporaire pour le rapport
    CREATE TEMPORARY TABLE IF NOT EXISTS temp_daily_report (
        report_date DATE,
        order_count INT,
        daily_total DECIMAL(15,2)
    ) ENGINE=MEMORY;

    -- Calculer le nombre de jours dans le mois
    SET v_days_in_month = DAY(LAST_DAY(CONCAT(p_year, '-', p_month, '-01')));

    -- Boucler sur chaque jour du mois
    WHILE v_day <= v_days_in_month DO
        SET v_current_date = CONCAT(p_year, '-', p_month, '-', v_day);

        -- Calculer les statistiques du jour
        SELECT
            COALESCE(SUM(total_amount), 0),
            COUNT(*)
        INTO
            v_daily_total,
            v_order_count
        FROM orders
        WHERE DATE(created_at) = v_current_date;

        -- Insérer dans le rapport
        INSERT INTO temp_daily_report (report_date, order_count, daily_total)
        VALUES (v_current_date, v_order_count, v_daily_total);

        SET v_day = v_day + 1;
    END WHILE;

    -- Retourner le rapport avec statistiques
    SELECT
        report_date,
        order_count,
        daily_total,
        SUM(daily_total) OVER (ORDER BY report_date) AS cumulative_total
    FROM temp_daily_report
    ORDER BY report_date;

    -- Statistiques globales
    SELECT
        COUNT(*) AS days_with_orders,
        SUM(order_count) AS total_orders,
        SUM(daily_total) AS monthly_total,
        AVG(daily_total) AS avg_daily_total
    FROM temp_daily_report
    WHERE order_count > 0;

    -- Nettoyer
    DROP TEMPORARY TABLE IF EXISTS temp_daily_report;
END$$

DELIMITER ;
```

### 3. Traitement Batch avec Gestion d'Erreurs

```sql
DELIMITER $$

CREATE PROCEDURE process_pending_orders_batch()
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_success_count INT DEFAULT 0;
    DECLARE v_error_count INT DEFAULT 0;
    DECLARE v_total_processed INT DEFAULT 0;

    -- Curseur pour les commandes en attente
    DECLARE order_cursor CURSOR FOR
        SELECT id FROM orders
        WHERE status = 'pending'
        ORDER BY created_at
        LIMIT 100;  -- Traiter par batch de 100

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN order_cursor;

    process_loop: LOOP
        FETCH order_cursor INTO v_order_id;

        IF v_done THEN
            LEAVE process_loop;
        END IF;

        SET v_total_processed = v_total_processed + 1;

        -- Traiter chaque commande avec gestion d'erreur
        BEGIN
            DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
            BEGIN
                SET v_error_count = v_error_count + 1;

                INSERT INTO error_log (entity_type, entity_id, error_message)
                VALUES ('order', v_order_id, 'Processing failed');

                -- Marquer la commande comme en erreur
                UPDATE orders
                SET status = 'error', updated_at = NOW()
                WHERE id = v_order_id;
            END;

            -- Tentative de traitement
            CALL process_single_order(v_order_id);

            -- Si succès (pas d'erreur), incrémenter
            SET v_success_count = v_success_count + 1;
        END;

    END LOOP process_loop;

    CLOSE order_cursor;

    -- Rapport final
    SELECT
        v_total_processed AS total_processed,
        v_success_count AS successful,
        v_error_count AS failed,
        ROUND((v_success_count / v_total_processed) * 100, 2) AS success_rate;
END$$

DELIMITER ;
```

---

## Bonnes Pratiques

### 1. ✅ Nommer les Labels de Boucle

```sql
-- ❌ MAUVAIS : Pas de label
LOOP
    IF condition THEN
        LEAVE;  -- Ambigu si boucles imbriquées
    END IF;
END LOOP;

-- ✅ BON : Label descriptif
order_processing_loop: LOOP
    IF condition THEN
        LEAVE order_processing_loop;
    END IF;
END LOOP order_processing_loop;
```

### 2. ✅ Initialiser les Variables

```sql
-- ❌ MAUVAIS : Variable non initialisée
DECLARE v_counter INT;
SET v_counter = v_counter + 1;  -- NULL + 1 = NULL

-- ✅ BON : Initialisation explicite
DECLARE v_counter INT DEFAULT 0;
SET v_counter = v_counter + 1;  -- 0 + 1 = 1
```

### 3. ✅ Limiter la Profondeur d'Imbrication

```sql
-- ❌ MAUVAIS : Trop d'imbrication
IF condition1 THEN
    IF condition2 THEN
        IF condition3 THEN
            IF condition4 THEN
                -- Code difficile à lire
            END IF;
        END IF;
    END IF;
END IF;

-- ✅ BON : Sortie anticipée
IF NOT condition1 THEN
    LEAVE;
END IF;

IF NOT condition2 THEN
    LEAVE;
END IF;

-- Code plus lisible
```

### 4. ✅ Préférer CASE à IF pour Multiples Valeurs

```sql
-- ❌ Verbeux avec IF
IF status = 'pending' THEN
    SET priority = 1;
ELSEIF status = 'processing' THEN
    SET priority = 2;
ELSEIF status = 'shipped' THEN
    SET priority = 3;
END IF;

-- ✅ Plus clair avec CASE
SET priority = CASE status
    WHEN 'pending' THEN 1
    WHEN 'processing' THEN 2
    WHEN 'shipped' THEN 3
    ELSE 0
END;
```

### 5. ✅ Éviter les Boucles Infinies

```sql
-- ❌ MAUVAIS : Risque de boucle infinie
WHILE TRUE DO
    -- Oubli de condition de sortie
    -- ...
END WHILE;

-- ✅ BON : Condition de sortie claire
DECLARE v_counter INT DEFAULT 0;
DECLARE v_max_iterations INT DEFAULT 1000;

WHILE v_counter < v_max_iterations DO
    -- Traitement
    SET v_counter = v_counter + 1;

    IF some_condition THEN
        LEAVE;
    END IF;
END WHILE;

-- Vérifier si terminé normalement
IF v_counter >= v_max_iterations THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Maximum iterations reached';
END IF;
```

### 6. ✅ Utiliser des Constantes

```sql
DELIMITER $$

CREATE PROCEDURE use_constants()
BEGIN
    -- Déclarer des constantes (convention : c_ prefix)
    DECLARE c_vip_discount DECIMAL(5,2) DEFAULT 0.20;
    DECLARE c_gold_discount DECIMAL(5,2) DEFAULT 0.15;
    DECLARE c_regular_discount DECIMAL(5,2) DEFAULT 0.05;
    DECLARE c_max_discount DECIMAL(5,2) DEFAULT 0.30;

    -- Utiliser les constantes
    DECLARE v_discount DECIMAL(5,2);

    SET v_discount = CASE customer_tier
        WHEN 'VIP' THEN c_vip_discount
        WHEN 'GOLD' THEN c_gold_discount
        ELSE c_regular_discount
    END;

    -- Plus facile à maintenir qu'avec des valeurs magiques
END$$

DELIMITER ;
```

### 7. ✅ Commenter le Code Complexe

```sql
DELIMITER $$

CREATE PROCEDURE well_documented_logic(IN p_amount DECIMAL(10,2))
BEGIN
    DECLARE v_discount DECIMAL(5,2) DEFAULT 0.00;

    -- ═════════════════════════════════════════════════════
    -- Calcul de remise progressive :
    -- - Base : 5% pour tous
    -- - +5% si montant > 100
    -- - +5% si montant > 500
    -- - +5% si montant > 1000
    -- Maximum : 20%
    -- ═════════════════════════════════════════════════════

    SET v_discount = 0.05;  -- Remise de base

    IF p_amount > 100 THEN
        SET v_discount = v_discount + 0.05;
    END IF;

    IF p_amount > 500 THEN
        SET v_discount = v_discount + 0.05;
    END IF;

    IF p_amount > 1000 THEN
        SET v_discount = v_discount + 0.05;
    END IF;

    -- Résultat final
    SELECT
        p_amount AS original_amount,
        v_discount * 100 AS discount_percent,
        p_amount * (1 - v_discount) AS final_amount;
END$$

DELIMITER ;
```

---

## ✅ Points clés à retenir

- 📊 **Variables** : Locales (DECLARE), Session (@var), Globales (@@var)
- 🎯 **Assignation** : SET ou SELECT INTO avec gestion NOT FOUND
- 🔀 **IF** : Conditions booléennes complexes, sortie anticipée
- 🔄 **CASE** : Comparaison valeur/constantes, plus lisible
- ♾️ **LOOP** : Boucle avec contrôle total (LEAVE, ITERATE)
- 🔁 **WHILE** : Condition testée au début (peut ne jamais s'exécuter)
- 🔂 **REPEAT** : Condition testée à la fin (au moins une exécution)
- 🚪 **LEAVE** : Sortir d'un bloc ou boucle
- ⏭️ **ITERATE** : Continuer à l'itération suivante
- ✅ **Bonnes pratiques** : Labels clairs, initialisation, limiter imbrication, constantes

---

## 🔗 Ressources et références

- [📖 Flow Control Statements - MariaDB Documentation](https://mariadb.com/kb/en/flow-control/)
- [📖 DECLARE Variable - MariaDB Documentation](https://mariadb.com/kb/en/declare-variable/)
- [📖 IF Statement - MariaDB Documentation](https://mariadb.com/kb/en/if-statement/)
- [📖 CASE Statement - MariaDB Documentation](https://mariadb.com/kb/en/case-statement/)
- [📖 LOOP Statement - MariaDB Documentation](https://mariadb.com/kb/en/loop/)
- [📖 WHILE Statement - MariaDB Documentation](https://mariadb.com/kb/en/while/)
- [📖 REPEAT Statement - MariaDB Documentation](https://mariadb.com/kb/en/repeat-loop/)

---

## ➡️ Section suivante

**8.8 Bonnes pratiques de programmation** : Standards de production, conventions de nommage, tests, documentation, maintenance et optimisation des routines serveur.

---


⏭️ [Bonnes pratiques de programmation](/08-programmation-cote-serveur/08-bonnes-pratiques.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8. Programmation Côté Serveur

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 8-10 heures
> **Prérequis** : Maîtrise SQL (chapitres 2-4), Transactions (chapitre 6)

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Créer et gérer** des procédures stockées avec paramètres IN/OUT/INOUT
- **Développer** des fonctions stockées réutilisables et performantes
- **Implémenter** des triggers pour automatiser la logique métier
- **Planifier** des tâches avec l'Event Scheduler
- **Manipuler** des curseurs pour traiter des ensembles de résultats
- **Gérer** les erreurs et exceptions avec des handlers appropriés
- **Appliquer** les bonnes pratiques de programmation serveur en production

---

## Introduction

La **programmation côté serveur** dans MariaDB permet d'encapsuler la logique métier directement au niveau de la base de données. Cette approche offre plusieurs avantages stratégiques :

### 🎯 Pourquoi programmer côté serveur ?

**Avantages** :
- ✅ **Performance** : Réduction des allers-retours réseau entre application et base
- ✅ **Sécurité** : Logique métier protégée, accès contrôlé via procédures
- ✅ **Réutilisabilité** : Code partagé entre plusieurs applications
- ✅ **Maintenance** : Centralisation de la logique métier complexe
- ✅ **Intégrité** : Contraintes métier appliquées automatiquement via triggers
- ✅ **Automatisation** : Tâches planifiées sans intervention externe

**Inconvénients à considérer** :
- ⚠️ **Portabilité** : Code spécifique au SGBD (difficile à migrer)
- ⚠️ **Debugging** : Outils de débogage moins riches qu'en applicatif
- ⚠️ **Scalabilité** : Charge CPU sur le serveur de base de données
- ⚠️ **Version control** : Complexité de gestion dans Git/SVN
- ⚠️ **Testing** : Tests unitaires plus difficiles à mettre en place

### 📊 Quand utiliser la programmation serveur ?

| Cas d'usage | Recommandation | Alternative |
|-------------|----------------|-------------|
| **Calculs complexes sur données** | ✅ Procédure stockée | Application si calcul léger |
| **Validation métier multi-tables** | ✅ Trigger | Application + transaction |
| **Transformation de données** | ✅ Fonction stockée | Vues ou requêtes SQL |
| **Tâches planifiées** | ✅ Event | Cron + script externe |
| **Audit automatique** | ✅ Trigger | Logging applicatif |
| **Business logic critique** | ⚠️ Selon contexte | Préférer application |

💡 **Conseil** : Privilégiez la logique applicative pour les règles métier complexes évolutives. Réservez la programmation serveur pour les opérations proches des données (intégrité, audit, transformations).

---

## Vue d'ensemble des composants

### 1️⃣ Procédures Stockées (Stored Procedures)

Blocs de code SQL réutilisables pouvant contenir de la logique complexe.

```sql
-- Syntaxe simplifiée
DELIMITER $$
CREATE PROCEDURE nom_procedure(IN param1 INT, OUT param2 VARCHAR(100))
BEGIN
    -- Instructions SQL
    -- Logique métier
    -- Assignation de valeurs OUT
END$$
DELIMITER ;
```

**Caractéristiques** :
- Peuvent modifier les données (INSERT, UPDATE, DELETE)
- Acceptent des paramètres IN, OUT, INOUT
- Ne retournent pas de valeur directe (utilisent OUT)
- Appelées avec `CALL nom_procedure(args)`

**Cas d'usage typiques** :
- Opérations CRUD complexes
- Traitement batch de données
- Workflows métier multi-étapes

---

### 2️⃣ Fonctions Stockées (Stored Functions)

Similaires aux procédures mais retournent une valeur unique.

```sql
-- Syntaxe simplifiée
DELIMITER $$
CREATE FUNCTION nom_fonction(param1 INT)
RETURNS VARCHAR(100)
DETERMINISTIC
BEGIN
    DECLARE resultat VARCHAR(100);
    -- Logique
    RETURN resultat;
END$$
DELIMITER ;
```

**Caractéristiques** :
- Retournent une valeur scalaire via `RETURN`
- Utilisables dans SELECT, WHERE, etc.
- Doivent être DETERMINISTIC ou NO SQL selon contexte
- Paramètres en lecture seule (IN implicite)

**Cas d'usage typiques** :
- Calculs métier réutilisables
- Formatage de données
- Validation de valeurs

---

### 3️⃣ Triggers (Déclencheurs)

Procédures automatiquement exécutées en réponse à des événements.

```sql
-- Syntaxe simplifiée
DELIMITER $$
CREATE TRIGGER nom_trigger
BEFORE INSERT ON table_name
FOR EACH ROW
BEGIN
    -- Accès à NEW.colonne (nouvelle valeur)
    -- Accès à OLD.colonne (ancienne valeur pour UPDATE/DELETE)
    SET NEW.colonne = valeur;
END$$
DELIMITER ;
```

**Caractéristiques** :
- Déclenchés par INSERT, UPDATE, DELETE
- Exécutés BEFORE ou AFTER l'événement
- Accès aux variables `OLD` et `NEW`
- Invisible pour l'application (exécution transparente)

**Cas d'usage typiques** :
- Audit automatique (traçabilité)
- Validation de données complexes
- Maintien de données dénormalisées
- Propagation de modifications

---

### 4️⃣ Events (Tâches Planifiées)

Équivalent d'un cron intégré à MariaDB.

```sql
-- Syntaxe simplifiée
CREATE EVENT nom_event
ON SCHEDULE EVERY 1 DAY
DO
BEGIN
    -- Instructions SQL à exécuter
END;
```

**Caractéristiques** :
- Planification flexible (ONE TIME, EVERY)
- Gérés par l'Event Scheduler
- Exécution asynchrone et automatique
- Idéal pour maintenance et traitements batch

**Cas d'usage typiques** :
- Purge de données anciennes
- Agrégation périodique
- Génération de rapports
- Maintenance automatique

---

### 5️⃣ Curseurs (Cursors)

Mécanisme pour parcourir les résultats d'une requête ligne par ligne.

```sql
-- Syntaxe simplifiée
DECLARE cursor_name CURSOR FOR SELECT ...;
OPEN cursor_name;
FETCH cursor_name INTO variables;
-- Traitement
CLOSE cursor_name;
```

**Caractéristiques** :
- Traitement séquentiel de résultats
- Utilisés dans les procédures stockées
- Impact performance (éviter si possible)
- Alternative : opérations ensemblistes

**Cas d'usage typiques** :
- Migration de données complexe
- Traitement ligne par ligne obligatoire
- Logique conditionnelle par enregistrement

💡 **Conseil** : Privilégiez les opérations ensemblistes (UPDATE avec JOIN) plutôt que les curseurs pour de meilleures performances.

---

### 6️⃣ Gestion des Erreurs (Error Handling)

Mécanisme pour capturer et traiter les erreurs.

```sql
-- Syntaxe simplifiée
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    -- Code en cas d'erreur
    ROLLBACK;
END;
```

**Caractéristiques** :
- Handlers pour SQLEXCEPTION, NOT FOUND, SQLWARNING
- Types : CONTINUE, EXIT, UNDO
- Gestion transactionnelle intégrée
- Logging et traçabilité des erreurs

---

## 🏗️ Architecture de la Programmation Serveur

```
┌─────────────────────────────────────────────────────┐
│              APPLICATION CLIENTE                    │
│         (PHP, Python, Java, Node.js...)             │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ CALL procedure()
                  │ SELECT fonction()
                  │
┌─────────────────▼───────────────────────────────────┐
│              MARIADB SERVER                         │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │     STORED PROGRAMS CACHE                   │    │
│  │  (Procédures, Fonctions compilées)          │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────┐  ┌───────────────────────┐     │
│  │   TRIGGERS      │  │   EVENT SCHEDULER     │     │
│  │  (Automatiques) │  │   (Tâches planifiées) │     │
│  └─────────────────┘  └───────────────────────┘     │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │          STORAGE ENGINE (InnoDB)              │  │
│  │              - Données                        │  │
│  │              - Transactions                   │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Syntaxe de Base et Conventions

### Délimiteurs (DELIMITER)

Les procédures, fonctions et triggers utilisent plusieurs instructions SQL. Il faut changer le délimiteur temporairement :

```sql
-- Changer le délimiteur pour permettre les ;
DELIMITER $$

CREATE PROCEDURE ma_procedure()
BEGIN
    SELECT 'Hello';  -- ; ne termine pas la procédure
    SELECT 'World';
END$$  -- $$ termine la procédure

DELIMITER ;  -- Retour au délimiteur par défaut
```

💡 **Conseil** : Utilisez `$$` ou `//` comme délimiteur alternatif. Évitez les caractères exotiques.

### Déclaration de Variables

```sql
DECLARE variable_name datatype [DEFAULT valeur];

-- Exemples
DECLARE total DECIMAL(10,2) DEFAULT 0.0;
DECLARE user_name VARCHAR(100);
DECLARE is_active BOOLEAN DEFAULT TRUE;
DECLARE created_date DATETIME DEFAULT NOW();
```

### Assignation de Valeurs

```sql
-- Méthode 1 : SET
SET variable_name = valeur;

-- Méthode 2 : SELECT INTO
SELECT colonne INTO variable_name FROM table WHERE condition;

-- Exemples
SET total = 1000;
SELECT COUNT(*) INTO nb_users FROM users WHERE active = 1;
```

### Structures de Contrôle

```sql
-- IF / ELSEIF / ELSE
IF condition THEN
    -- instructions
ELSEIF autre_condition THEN
    -- instructions
ELSE
    -- instructions
END IF;

-- CASE
CASE expression
    WHEN valeur1 THEN instruction1
    WHEN valeur2 THEN instruction2
    ELSE instruction_default
END CASE;

-- LOOP (boucle infinie avec EXIT)
label: LOOP
    -- instructions
    IF condition THEN
        LEAVE label;
    END IF;
END LOOP label;

-- WHILE
WHILE condition DO
    -- instructions
END WHILE;

-- REPEAT (équivalent do-while)
REPEAT
    -- instructions
UNTIL condition
END REPEAT;
```

---

## 📝 Exemple Complet : Système de Commandes

Voici un exemple intégré utilisant plusieurs composants :

### 1. Table de base

```sql
-- Tables pour l'exemple
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    price DECIMAL(10,2) NOT NULL
) ENGINE=InnoDB;

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    total_amount DECIMAL(10,2),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'confirmed', 'shipped', 'cancelled') DEFAULT 'pending'
) ENGINE=InnoDB;

CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
) ENGINE=InnoDB;

CREATE TABLE audit_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(50),
    operation VARCHAR(20),
    record_id INT,
    details TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

### 2. Fonction : Calculer le prix total

```sql
DELIMITER $$

CREATE FUNCTION calculate_order_total(p_order_id INT)
RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_total DECIMAL(10,2) DEFAULT 0.0;

    -- Calculer la somme des items
    SELECT COALESCE(SUM(quantity * unit_price), 0)
    INTO v_total
    FROM order_items
    WHERE order_id = p_order_id;

    RETURN v_total;
END$$

DELIMITER ;

-- Utilisation
-- SELECT calculate_order_total(123);
```

### 3. Procédure : Créer une commande

```sql
DELIMITER $$

CREATE PROCEDURE create_order(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_order_id INT,
    OUT p_message VARCHAR(255)
)
BEGIN
    DECLARE v_stock INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- En cas d'erreur, rollback et message
        ROLLBACK;
        SET p_message = 'Erreur lors de la création de la commande';
        SET p_order_id = NULL;
    END;

    -- Démarrer la transaction
    START TRANSACTION;

    -- Vérifier le stock disponible
    SELECT stock, price INTO v_stock, v_price
    FROM products
    WHERE id = p_product_id
    FOR UPDATE;  -- Lock pessimiste

    IF v_stock < p_quantity THEN
        SET p_message = CONCAT('Stock insuffisant. Disponible: ', v_stock);
        SET p_order_id = NULL;
        ROLLBACK;
    ELSE
        -- Créer la commande
        INSERT INTO orders (customer_id, status)
        VALUES (p_customer_id, 'pending');

        SET p_order_id = LAST_INSERT_ID();

        -- Ajouter l'item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (p_order_id, p_product_id, p_quantity, v_price);

        -- Mettre à jour le stock
        UPDATE products
        SET stock = stock - p_quantity
        WHERE id = p_product_id;

        -- Mettre à jour le total de la commande
        UPDATE orders
        SET total_amount = calculate_order_total(p_order_id)
        WHERE id = p_order_id;

        SET p_message = 'Commande créée avec succès';
        COMMIT;
    END IF;
END$$

DELIMITER ;

-- Utilisation
-- CALL create_order(1, 10, 5, @order_id, @msg);
-- SELECT @order_id, @msg;
```

### 4. Trigger : Audit automatique

```sql
DELIMITER $$

-- Trigger sur UPDATE des commandes
CREATE TRIGGER orders_after_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
    -- Logger le changement de statut
    IF OLD.status != NEW.status THEN
        INSERT INTO audit_log (table_name, operation, record_id, details)
        VALUES (
            'orders',
            'STATUS_CHANGE',
            NEW.id,
            CONCAT('Status changed from ', OLD.status, ' to ', NEW.status)
        );
    END IF;
END$$

-- Trigger BEFORE INSERT pour validation
CREATE TRIGGER order_items_before_insert
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    DECLARE v_stock INT;

    -- Vérifier que la quantité est positive
    IF NEW.quantity <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'La quantité doit être positive';
    END IF;

    -- Vérifier que le prix unitaire correspond au prix actuel
    SELECT price INTO v_stock
    FROM products
    WHERE id = NEW.product_id;

    -- S'assurer que le prix est à jour
    IF NEW.unit_price != v_stock THEN
        SET NEW.unit_price = v_stock;
    END IF;
END$$

DELIMITER ;
```

### 5. Event : Purge des anciennes commandes

```sql
DELIMITER $$

CREATE EVENT purge_old_cancelled_orders
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 02:00:00'
COMMENT 'Supprime les commandes annulées de plus de 90 jours'
DO
BEGIN
    -- Supprimer les items des anciennes commandes annulées
    DELETE oi FROM order_items oi
    INNER JOIN orders o ON oi.order_id = o.id
    WHERE o.status = 'cancelled'
    AND o.created_at < NOW() - INTERVAL 90 DAY;

    -- Supprimer les commandes
    DELETE FROM orders
    WHERE status = 'cancelled'
    AND created_at < NOW() - INTERVAL 90 DAY;

    -- Logger l'opération
    INSERT INTO audit_log (table_name, operation, details)
    VALUES ('orders', 'PURGE', CONCAT('Purged orders older than 90 days at ', NOW()));
END$$

DELIMITER ;

-- Activer l'Event Scheduler si nécessaire
SET GLOBAL event_scheduler = ON;
```

---

## ⚡ Performances et Considérations

### Impact sur les Performances

| Composant | Impact Performance | Optimisation |
|-----------|-------------------|--------------|
| **Procédures** | ⚠️ Moyen | Cache de plans d'exécution |
| **Fonctions** | ⚠️ Moyen à Élevé | DETERMINISTIC, indexes |
| **Triggers** | 🔴 Élevé | Logique minimale, éviter SELECT |
| **Events** | ✅ Faible | Exécution asynchrone |
| **Curseurs** | 🔴 Très Élevé | Éviter, préférer SET-based |

### Bonnes Pratiques de Performance

```sql
-- ❌ MAUVAIS : Curseur pour mise à jour
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
        UPDATE products SET price = price * 1.1 WHERE id = v_id;
    END LOOP;
    CLOSE cur;
END$$
DELIMITER ;

-- ✅ BON : Opération ensembliste
DELIMITER $$
CREATE PROCEDURE update_prices_good()
BEGIN
    UPDATE products
    SET price = price * 1.1;
END$$
DELIMITER ;
```

💡 **Règle d'or** : Une seule requête UPDATE/INSERT/DELETE est presque toujours plus rapide qu'un curseur.

---

## 🔐 Sécurité et Bonnes Pratiques

### 1. Gestion des Privilèges

```sql
-- Créer un rôle pour exécuter les procédures
CREATE ROLE app_executor;

-- Donner les privilèges d'exécution uniquement
GRANT EXECUTE ON PROCEDURE mydb.create_order TO app_executor;
GRANT EXECUTE ON FUNCTION mydb.calculate_order_total TO app_executor;

-- Ne pas donner d'accès direct aux tables
-- REVOKE ALL ON mydb.orders FROM app_executor;

-- Assigner le rôle aux utilisateurs applicatifs
GRANT app_executor TO 'app_user'@'localhost';
SET DEFAULT ROLE app_executor FOR 'app_user'@'localhost';
```

### 2. SQL Injection dans les Procédures

```sql
-- ❌ VULNÉRABLE : Construction dynamique dangereuse
DELIMITER $$
CREATE PROCEDURE search_user_bad(IN p_name VARCHAR(100))
BEGIN
    SET @query = CONCAT('SELECT * FROM users WHERE name = "', p_name, '"');
    PREPARE stmt FROM @query;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END$$
DELIMITER ;

-- ✅ SÉCURISÉ : Utiliser des paramètres
DELIMITER $$
CREATE PROCEDURE search_user_good(IN p_name VARCHAR(100))
BEGIN
    SELECT * FROM users WHERE name = p_name;
    -- Ou avec prepared statement sécurisé
    PREPARE stmt FROM 'SELECT * FROM users WHERE name = ?';
    EXECUTE stmt USING p_name;
    DEALLOCATE PREPARE stmt;
END$$
DELIMITER ;
```

### 3. Validation des Données

```sql
DELIMITER $$
CREATE PROCEDURE update_user_email(
    IN p_user_id INT,
    IN p_email VARCHAR(255)
)
BEGIN
    -- Validation de l'email
    IF p_email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Format email invalide';
    END IF;

    -- Mise à jour
    UPDATE users SET email = p_email WHERE id = p_user_id;
END$$
DELIMITER ;
```

---

## 📋 Conventions de Nommage

### Standards Recommandés

```sql
-- Procédures : verbe_nom (action claire)
CREATE PROCEDURE create_order()
CREATE PROCEDURE update_user_status()
CREATE PROCEDURE get_customer_orders()

-- Fonctions : get_xxx ou calculate_xxx
CREATE FUNCTION get_customer_name(id INT)
CREATE FUNCTION calculate_tax(amount DECIMAL)
CREATE FUNCTION is_valid_email(email VARCHAR)

-- Triggers : table_when_action
CREATE TRIGGER orders_before_insert
CREATE TRIGGER users_after_update
CREATE TRIGGER products_before_delete

-- Events : action_frequence
CREATE EVENT purge_old_logs_daily
CREATE EVENT update_statistics_hourly
CREATE EVENT backup_data_weekly

-- Variables : préfixe par type
DECLARE v_total DECIMAL(10,2);      -- v_ pour variable locale
DECLARE p_user_id INT;               -- p_ pour paramètre (déjà dans signature)
DECLARE c_max_retry INT DEFAULT 3;  -- c_ pour constante
```

---

## 🔍 Debugging et Maintenance

### Techniques de Debugging

```sql
-- 1. Logging dans une table
CREATE TABLE debug_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    procedure_name VARCHAR(100),
    message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

DELIMITER $$
CREATE PROCEDURE my_procedure_with_logging()
BEGIN
    INSERT INTO debug_log (procedure_name, message)
    VALUES ('my_procedure', 'Starting execution');

    -- Logique métier

    INSERT INTO debug_log (procedure_name, message)
    VALUES ('my_procedure', 'Completed successfully');
END$$
DELIMITER ;

-- 2. Utiliser SELECT pour afficher des valeurs
DELIMITER $$
CREATE PROCEDURE debug_example()
BEGIN
    DECLARE v_count INT;
    SELECT COUNT(*) INTO v_count FROM users;
    SELECT v_count AS debug_count;  -- Affiche dans les résultats
END$$
DELIMITER ;
```

### Gestion des Versions

```sql
-- Toujours utiliser CREATE OR REPLACE ou DROP IF EXISTS
DROP PROCEDURE IF EXISTS my_procedure;
DROP FUNCTION IF EXISTS my_function;

DELIMITER $$
CREATE PROCEDURE my_procedure_v2()
BEGIN
    -- Nouvelle version
END$$
DELIMITER ;

-- Documenter les changements en commentaire
DELIMITER $$
CREATE PROCEDURE process_orders()
COMMENT 'v2.1 - Added email notification - 2025-12-12'
BEGIN
    -- Code
END$$
DELIMITER ;
```

---

## 📚 Comparaison avec Autres SGBD

| Fonctionnalité | MariaDB | MySQL | PostgreSQL |
|----------------|---------|-------|------------|
| **Procédures** | ✅ Oui | ✅ Oui | ✅ Oui (PL/pgSQL) |
| **Fonctions** | ✅ Oui | ✅ Oui | ✅ Oui (multiples langages) |
| **Triggers** | ✅ BEFORE/AFTER | ✅ BEFORE/AFTER | ✅ BEFORE/AFTER/INSTEAD OF |
| **Events** | ✅ Oui | ✅ Oui | ❌ Non (utiliser pg_cron) |
| **Packages** | ❌ Non | ❌ Non | ✅ Oui (schemas) |
| **Exceptions nommées** | ❌ Non | ❌ Non | ✅ Oui |
| **Langage** | SQL/PSM | SQL/PSM | PL/pgSQL, PL/Python, etc. |

💡 **Note** : MariaDB est compatible avec MySQL pour la programmation serveur, mais offre des améliorations de performance et stabilité.

---

## ✅ Points clés à retenir

- 🎯 **Procédures vs Fonctions** : Les procédures modifient les données et utilisent OUT/INOUT ; les fonctions retournent une valeur et sont utilisables dans SELECT
- 🔄 **Triggers** : Automatisation puissante mais impact performance ; utiliser avec parcimonie et logique minimale
- ⏰ **Events** : Alternative viable aux cron jobs pour les tâches planifiées récurrentes
- 🚫 **Curseurs** : Éviter quand possible ; préférer les opérations ensemblistes (SET-based)
- 🛡️ **Gestion d'erreurs** : Toujours implémenter des handlers appropriés avec rollback transactionnel
- 🔐 **Sécurité** : Limiter les privilèges, valider les entrées, éviter le SQL dynamique non sécurisé
- 📊 **Performance** : La logique serveur consomme des ressources ; surveiller et optimiser régulièrement

---

## 🔗 Ressources et références

- [📖 Stored Routines - MariaDB Documentation](https://mariadb.com/kb/en/stored-routines/)
- [📖 Triggers - MariaDB Documentation](https://mariadb.com/kb/en/triggers/)
- [📖 Events - MariaDB Documentation](https://mariadb.com/kb/en/stored-programs-and-views-events/)
- [📖 Cursors - MariaDB Documentation](https://mariadb.com/kb/en/cursors/)
- [📖 Error Handling - MariaDB Documentation](https://mariadb.com/kb/en/declare-handler/)
- [📝 Best Practices for Stored Procedures](https://mariadb.com/kb/en/stored-procedure-best-practices/)

---

## ➡️ Sections suivantes

- **8.1 Procédures stockées** : Syntaxe détaillée, paramètres IN/OUT/INOUT, exemples complexes
- **8.2 Fonctions stockées** : Caractéristiques, DETERMINISTIC vs NOT DETERMINISTIC
- **8.3 Triggers** : BEFORE/AFTER, variables OLD/NEW, cas d'usage avancés
- **8.4 Events** : Event Scheduler, planification complexe, maintenance automatisée
- **8.5 Curseurs** : Parcours de résultats, handlers, alternatives performantes
- **8.6 Gestion des erreurs** : DECLARE HANDLER, SIGNAL, RESIGNAL, transactions
- **8.7 Variables et flow control** : IF, CASE, LOOP, WHILE, REPEAT
- **8.8 Bonnes pratiques** : Standards de production, tests, maintenance, documentation

---


⏭️ [Procédures stockées](/08-programmation-cote-serveur/01-procedures-stockees.md)

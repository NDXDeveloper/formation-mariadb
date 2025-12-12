🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3 Triggers (Déclencheurs)

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Procédures stockées (8.1), Transactions (chapitre 6), SQL avancé (chapitres 3-4)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et le fonctionnement des triggers dans MariaDB
- **Identifier** les différents types de triggers (BEFORE/AFTER, INSERT/UPDATE/DELETE)
- **Maîtriser** l'utilisation des variables OLD et NEW
- **Créer** des triggers pour l'audit, la validation et la synchronisation de données
- **Éviter** les problèmes de performance et les erreurs courantes
- **Implémenter** des patterns de triggers en production
- **Appliquer** les bonnes pratiques de développement et maintenance

---

## Introduction

Les **triggers** (déclencheurs) sont des routines automatiquement exécutées en réponse à des événements spécifiques sur une table. Ils constituent un mécanisme puissant pour maintenir l'intégrité des données, automatiser des tâches et implémenter une logique métier complexe de manière transparente pour les applications.

### 🔍 Qu'est-ce qu'un Trigger ?

Un trigger est un objet de base de données qui :

- **S'exécute automatiquement** lors d'un événement (INSERT, UPDATE, DELETE)
- **Agit au niveau de la ligne** (FOR EACH ROW)
- **S'exécute dans le contexte de la transaction** en cours
- **Est invisible pour l'application** cliente
- **Ne peut pas être appelé directement** (pas de CALL)

```sql
-- Exemple simple : Audit automatique
DELIMITER $$

CREATE TRIGGER log_user_changes
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, changed_at)
    VALUES ('users', NEW.id, 'UPDATE', NOW());
END$$

DELIMITER ;

-- Le trigger s'exécute automatiquement
UPDATE users SET email = 'new@example.com' WHERE id = 1;
-- → Le trigger log_user_changes s'est déclenché automatiquement
```

### 📊 Anatomie d'un Trigger

```
┌────────────────────────────────────────────────────┐
│                    APPLICATION                     │
└────────────────────┬───────────────────────────────┘
                     │
                     │ UPDATE users SET ...
                     ▼
┌────────────────────────────────────────────────────┐
│                  MARIADB SERVER                    │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │         BEFORE UPDATE TRIGGER                │  │
│  │  - Validation des données                    │  │
│  │  - Modification de NEW.colonne               │  │
│  │  - Vérifications métier                      │  │
│  └──────────────────┬───────────────────────────┘  │
│                     │                              │
│                     ▼                              │
│  ┌──────────────────────────────────────────────┐  │
│  │         EXÉCUTION UPDATE                     │  │
│  │  UPDATE de la ligne dans la table            │  │
│  └──────────────────┬───────────────────────────┘  │
│                     │                              │
│                     ▼                              │
│  ┌──────────────────────────────────────────────┐  │
│  │         AFTER UPDATE TRIGGER                 │  │
│  │  - Logging / Audit                           │  │
│  │  - Synchronisation                           │  │
│  │  - Notifications                             │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

---

## Types de Triggers

### 📋 Classification par Timing et Événement

MariaDB supporte **6 types de triggers** :

| Timing | INSERT | UPDATE | DELETE |
|--------|--------|--------|--------|
| **BEFORE** | BEFORE INSERT | BEFORE UPDATE | BEFORE DELETE |
| **AFTER** | AFTER INSERT | AFTER UPDATE | AFTER DELETE |

**BEFORE** : Exécuté **avant** la modification des données
- ✅ Peut modifier les valeurs à insérer/mettre à jour (NEW)
- ✅ Peut valider et rejeter l'opération (SIGNAL)
- ⚠️ Les données ne sont pas encore modifiées

**AFTER** : Exécuté **après** la modification des données
- ✅ Les données sont déjà modifiées
- ✅ Idéal pour logging, audit, synchronisation
- ❌ Ne peut pas modifier NEW (données déjà écrites)

### 🔄 Événements Déclencheurs

#### INSERT Triggers
```sql
-- Déclenché lors d'un INSERT
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');

-- Également déclenché par
INSERT INTO users SELECT * FROM temp_users;
LOAD DATA INFILE 'users.csv' INTO TABLE users;
REPLACE INTO users ...;  -- REPLACE = DELETE + INSERT
```

#### UPDATE Triggers
```sql
-- Déclenché lors d'un UPDATE
UPDATE users SET status = 'active' WHERE id = 1;

-- Également déclenché par
UPDATE users SET email = LOWER(email);  -- Mise à jour de masse
```

#### DELETE Triggers
```sql
-- Déclenché lors d'un DELETE
DELETE FROM users WHERE id = 1;

-- Également déclenché par
DELETE FROM users WHERE created_at < '2020-01-01';
TRUNCATE users;  -- ⚠️ ATTENTION : TRUNCATE ne déclenche PAS les triggers !
REPLACE INTO users ...;  -- REPLACE = DELETE + INSERT
```

---

## Variables OLD et NEW

Les triggers ont accès à des variables spéciales qui contiennent les valeurs des colonnes.

### 📊 Tableau de Disponibilité

| Type de Trigger | OLD (anciennes valeurs) | NEW (nouvelles valeurs) |
|-----------------|-------------------------|-------------------------|
| **BEFORE INSERT** | ❌ Non disponible | ✅ Lecture/Écriture |
| **AFTER INSERT** | ❌ Non disponible | ✅ Lecture seule |
| **BEFORE UPDATE** | ✅ Lecture seule | ✅ Lecture/Écriture |
| **AFTER UPDATE** | ✅ Lecture seule | ✅ Lecture seule |
| **BEFORE DELETE** | ✅ Lecture seule | ❌ Non disponible |
| **AFTER DELETE** | ✅ Lecture seule | ❌ Non disponible |

### 🔍 Utilisation de OLD et NEW

```sql
DELIMITER $$

-- INSERT : Seulement NEW
CREATE TRIGGER before_insert_user
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    -- OLD n'existe pas pour INSERT
    -- NEW.colonne est modifiable

    -- Normaliser l'email en minuscules
    SET NEW.email = LOWER(NEW.email);

    -- Définir une valeur par défaut si NULL
    IF NEW.created_at IS NULL THEN
        SET NEW.created_at = NOW();
    END IF;
END$$

-- UPDATE : OLD et NEW disponibles
CREATE TRIGGER before_update_user
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    -- OLD.colonne : valeur avant l'UPDATE (lecture seule)
    -- NEW.colonne : nouvelle valeur (modifiable)

    -- Empêcher la modification de l'email
    IF OLD.email != NEW.email THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Email cannot be changed';
    END IF;

    -- Mettre à jour automatiquement le timestamp
    SET NEW.updated_at = NOW();
END$$

-- DELETE : Seulement OLD
CREATE TRIGGER before_delete_user
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
    -- OLD.colonne : valeurs de la ligne à supprimer (lecture seule)
    -- NEW n'existe pas pour DELETE

    -- Empêcher la suppression des admins
    IF OLD.role = 'admin' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Cannot delete admin users';
    END IF;
END$$

DELIMITER ;
```

### 💡 Exemples Pratiques avec OLD et NEW

```sql
DELIMITER $$

-- Exemple 1 : Tracer les changements de prix
CREATE TRIGGER track_price_changes
AFTER UPDATE ON products
FOR EACH ROW
BEGIN
    -- Vérifier si le prix a changé
    IF OLD.price != NEW.price THEN
        INSERT INTO price_history (
            product_id,
            old_price,
            new_price,
            changed_by,
            changed_at
        ) VALUES (
            NEW.id,
            OLD.price,
            NEW.price,
            USER(),
            NOW()
        );
    END IF;
END$$

-- Exemple 2 : Calculer automatiquement une colonne dérivée
CREATE TRIGGER calculate_total
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    -- Calculer automatiquement le total
    SET NEW.total = NEW.quantity * NEW.unit_price;

    -- Appliquer une remise si quantité > 10
    IF NEW.quantity > 10 THEN
        SET NEW.total = NEW.total * 0.9;  -- 10% de réduction
    END IF;
END$$

-- Exemple 3 : Validation complexe
CREATE TRIGGER validate_order_item
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    DECLARE product_stock INT;

    -- Vérifier que la quantité est positive
    IF NEW.quantity <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Quantity must be positive';
    END IF;

    -- Vérifier le stock disponible
    SELECT stock INTO product_stock
    FROM products
    WHERE id = NEW.product_id;

    IF product_stock < NEW.quantity THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Insufficient stock';
    END IF;
END$$

DELIMITER ;
```

---

## Cas d'Usage Principaux

### 1️⃣ Audit et Logging

**Objectif** : Tracer toutes les modifications sur une table sensible.

```sql
-- Table d'audit
CREATE TABLE audit_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(50),
    record_id INT,
    action VARCHAR(10),
    old_values JSON,
    new_values JSON,
    changed_by VARCHAR(100),
    changed_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

DELIMITER $$

-- Trigger d'audit pour INSERT
CREATE TRIGGER audit_users_insert
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, new_values, changed_by)
    VALUES (
        'users',
        NEW.id,
        'INSERT',
        JSON_OBJECT(
            'name', NEW.name,
            'email', NEW.email,
            'status', NEW.status
        ),
        USER()
    );
END$$

-- Trigger d'audit pour UPDATE
CREATE TRIGGER audit_users_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (
        table_name,
        record_id,
        action,
        old_values,
        new_values,
        changed_by
    )
    VALUES (
        'users',
        NEW.id,
        'UPDATE',
        JSON_OBJECT(
            'name', OLD.name,
            'email', OLD.email,
            'status', OLD.status
        ),
        JSON_OBJECT(
            'name', NEW.name,
            'email', NEW.email,
            'status', NEW.status
        ),
        USER()
    );
END$$

-- Trigger d'audit pour DELETE
CREATE TRIGGER audit_users_delete
AFTER DELETE ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_values, changed_by)
    VALUES (
        'users',
        OLD.id,
        'DELETE',
        JSON_OBJECT(
            'name', OLD.name,
            'email', OLD.email,
            'status', OLD.status
        ),
        USER()
    );
END$$

DELIMITER ;
```

### 2️⃣ Validation de Données

**Objectif** : Appliquer des règles métier complexes avant l'insertion ou la modification.

```sql
DELIMITER $$

-- Valider un email
CREATE TRIGGER validate_user_email
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    -- Format email basique
    IF NEW.email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Invalid email format';
    END IF;

    -- Domaine interdit
    IF NEW.email LIKE '%@spam.com' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Email domain not allowed';
    END IF;
END$$

-- Valider les contraintes métier
CREATE TRIGGER validate_order
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    DECLARE customer_status VARCHAR(20);
    DECLARE customer_credit_limit DECIMAL(10,2);

    -- Récupérer le statut du client
    SELECT status, credit_limit
    INTO customer_status, customer_credit_limit
    FROM customers
    WHERE id = NEW.customer_id;

    -- Vérifier que le client est actif
    IF customer_status != 'active' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Customer account is not active';
    END IF;

    -- Vérifier la limite de crédit
    IF NEW.total_amount > customer_credit_limit THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Order exceeds customer credit limit';
    END IF;

    -- Définir le statut par défaut
    SET NEW.status = 'pending';
    SET NEW.created_at = NOW();
END$$

DELIMITER ;
```

### 3️⃣ Synchronisation et Dénormalisation

**Objectif** : Maintenir des données dénormalisées à jour automatiquement.

```sql
DELIMITER $$

-- Mettre à jour le nombre de commandes d'un client
CREATE TRIGGER update_customer_order_count_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    UPDATE customers
    SET
        order_count = order_count + 1,
        total_spent = total_spent + NEW.total_amount,
        last_order_date = NEW.created_at
    WHERE id = NEW.customer_id;
END$$

CREATE TRIGGER update_customer_order_count_delete
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
    UPDATE customers
    SET
        order_count = order_count - 1,
        total_spent = total_spent - OLD.total_amount
    WHERE id = OLD.customer_id;
END$$

-- Maintenir un compteur de stock
CREATE TRIGGER update_product_stock_on_order
AFTER INSERT ON order_items
FOR EACH ROW
BEGIN
    UPDATE products
    SET stock = stock - NEW.quantity
    WHERE id = NEW.product_id;
END$$

CREATE TRIGGER restore_product_stock_on_cancel
AFTER DELETE ON order_items
FOR EACH ROW
BEGIN
    UPDATE products
    SET stock = stock + OLD.quantity
    WHERE id = OLD.product_id;
END$$

DELIMITER ;
```

### 4️⃣ Timestamps Automatiques

**Objectif** : Gérer automatiquement les colonnes created_at et updated_at.

```sql
DELIMITER $$

CREATE TRIGGER set_created_at
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    IF NEW.created_at IS NULL THEN
        SET NEW.created_at = NOW();
    END IF;

    -- Initialiser updated_at également
    SET NEW.updated_at = NOW();
END$$

CREATE TRIGGER set_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    -- Toujours mettre à jour le timestamp
    SET NEW.updated_at = NOW();

    -- Empêcher la modification de created_at
    SET NEW.created_at = OLD.created_at;
END$$

DELIMITER ;
```

### 5️⃣ Soft Delete (Suppression Logique)

**Objectif** : Marquer les enregistrements comme supprimés au lieu de les supprimer physiquement.

```sql
-- Table avec soft delete
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    email VARCHAR(100),
    deleted_at DATETIME NULL,
    is_deleted BOOLEAN DEFAULT FALSE
) ENGINE=InnoDB;

DELIMITER $$

-- Intercepter DELETE et convertir en UPDATE
CREATE TRIGGER soft_delete_user
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
    -- Empêcher la vraie suppression
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Use UPDATE to set deleted_at instead of DELETE';
END$$

-- Alternative : Procédure pour soft delete
CREATE PROCEDURE soft_delete_user(IN p_user_id INT)
BEGIN
    UPDATE users
    SET
        deleted_at = NOW(),
        is_deleted = TRUE
    WHERE id = p_user_id
    AND is_deleted = FALSE;
END$$

DELIMITER ;

-- Utilisation
-- DELETE FROM users WHERE id = 1;  ❌ Erreur
CALL soft_delete_user(1);  -- ✅ Soft delete
```

### 6️⃣ Calculs Automatiques

**Objectif** : Calculer automatiquement des valeurs dérivées.

```sql
DELIMITER $$

-- Calculer le prix TTC automatiquement
CREATE TRIGGER calculate_price_ttc
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
    -- Prix TTC = Prix HT * 1.20 (TVA 20%)
    SET NEW.price_ttc = NEW.price_ht * 1.20;
END$$

CREATE TRIGGER update_price_ttc
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    -- Recalculer si le prix HT a changé
    IF OLD.price_ht != NEW.price_ht THEN
        SET NEW.price_ttc = NEW.price_ht * 1.20;
    END IF;
END$$

-- Calculer l'âge automatiquement
CREATE TRIGGER calculate_age
BEFORE INSERT ON persons
FOR EACH ROW
BEGIN
    SET NEW.age = TIMESTAMPDIFF(YEAR, NEW.birth_date, CURDATE());
END$$

CREATE TRIGGER update_age
BEFORE UPDATE ON persons
FOR EACH ROW
BEGIN
    IF OLD.birth_date != NEW.birth_date THEN
        SET NEW.age = TIMESTAMPDIFF(YEAR, NEW.birth_date, CURDATE());
    END IF;
END$$

DELIMITER ;
```

---

## Gestion des Erreurs dans les Triggers

### SIGNAL : Lever une Erreur

```sql
DELIMITER $$

CREATE TRIGGER validate_minimum_price
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
    -- Vérifier le prix minimum
    IF NEW.price < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Price cannot be negative';
    END IF;

    IF NEW.price < 1 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Price must be at least 1.00',
            MYSQL_ERRNO = 1644;
    END IF;
END$$

DELIMITER ;

-- Test
INSERT INTO products (name, price) VALUES ('Test', -10);
-- Erreur : Price cannot be negative
```

### Gestion d'Erreurs Complexe

```sql
DELIMITER $$

CREATE TRIGGER complex_validation
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    DECLARE error_message VARCHAR(255);
    DECLARE customer_exists BOOLEAN;

    -- Vérifier l'existence du client
    SELECT COUNT(*) > 0 INTO customer_exists
    FROM customers
    WHERE id = NEW.customer_id;

    IF NOT customer_exists THEN
        SET error_message = CONCAT('Customer ID ', NEW.customer_id, ' does not exist');
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = error_message;
    END IF;

    -- Valider le montant
    IF NEW.total_amount <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Total amount must be positive';
    END IF;

    -- Valider la date
    IF NEW.order_date > CURDATE() THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Order date cannot be in the future';
    END IF;
END$$

DELIMITER ;
```

---

## Bonnes Pratiques

### 1. ✅ Garder les Triggers Simples et Rapides

```sql
-- ❌ MAUVAIS : Logique complexe et lente
CREATE TRIGGER bad_complex_trigger
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- Multiples requêtes lourdes
    UPDATE customers SET order_count = (SELECT COUNT(*) FROM orders WHERE customer_id = NEW.customer_id);

    -- Calculs complexes
    UPDATE statistics SET daily_total = (SELECT SUM(total) FROM orders WHERE DATE(created_at) = CURDATE());

    -- Appels externes (très lent !)
    -- CALL send_email_notification(...);
END;

-- ✅ BON : Logique simple et ciblée
CREATE TRIGGER good_simple_trigger
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- Incrément simple
    UPDATE customers
    SET order_count = order_count + 1
    WHERE id = NEW.customer_id;
END;
```

💡 **Règle** : Un trigger doit s'exécuter en moins de 10ms. Pour des traitements longs, utilisez une table de queue + job asynchrone.

### 2. ✅ Éviter les Cascades de Triggers

```sql
-- ⚠️ ATTENTION : Trigger qui déclenche un autre trigger
CREATE TRIGGER trigger_a
AFTER UPDATE ON table_a
FOR EACH ROW
BEGIN
    UPDATE table_b SET value = NEW.value WHERE id = NEW.id;
END;

CREATE TRIGGER trigger_b
AFTER UPDATE ON table_b
FOR EACH ROW
BEGIN
    UPDATE table_c SET value = NEW.value WHERE id = NEW.id;
END;

-- Problème : UPDATE table_a → trigger_a → UPDATE table_b → trigger_b → UPDATE table_c
-- = Cascade difficile à debugger et maintenir
```

💡 **Solution** : Limiter à 1 niveau de profondeur maximum.

### 3. ✅ Documenter les Triggers

```sql
DELIMITER $$

CREATE TRIGGER audit_product_changes
AFTER UPDATE ON products
FOR EACH ROW
BEGIN
    -- ============================================
    -- Trigger: audit_product_changes
    -- Description: Log toutes les modifications de produits
    -- Auteur: Team Backend
    -- Date: 2025-12-12
    -- Version: 1.0
    --
    -- Déclenchement: AFTER UPDATE sur products
    -- Tables impactées: audit_log
    --
    -- Changements:
    --   v1.0 (2025-12-12): Version initiale
    -- ============================================

    INSERT INTO audit_log (table_name, record_id, action, changed_by)
    VALUES ('products', NEW.id, 'UPDATE', USER());
END$$

DELIMITER ;
```

### 4. ✅ Nommer les Triggers de Manière Cohérente

```sql
-- Convention de nommage recommandée :
-- [table]_[timing]_[action]_[description]

CREATE TRIGGER users_before_insert_validate;
CREATE TRIGGER users_after_update_audit;
CREATE TRIGGER orders_before_delete_check;
CREATE TRIGGER products_after_insert_sync_cache;
```

### 5. ✅ Gérer les Conditions Multiples

```sql
DELIMITER $$

-- ✅ BON : Vérifier les changements spécifiques
CREATE TRIGGER track_important_changes
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    -- Logger seulement si des champs importants ont changé
    IF OLD.email != NEW.email OR OLD.role != NEW.role THEN
        INSERT INTO audit_log (table_name, record_id, action, details)
        VALUES (
            'users',
            NEW.id,
            'CRITICAL_UPDATE',
            CONCAT('Email or role changed by ', USER())
        );
    END IF;
END$$

DELIMITER ;
```

### 6. ✅ Utiliser des Flags pour Désactiver Temporairement

```sql
DELIMITER $$

CREATE TRIGGER conditional_trigger
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    -- Variable de session pour désactiver le trigger
    IF @disable_triggers = 1 THEN
        -- Ne rien faire
        LEAVE;
    END IF;

    -- Logique normale du trigger
    SET NEW.updated_at = NOW();
END$$

DELIMITER ;

-- Désactiver temporairement
SET @disable_triggers = 1;
UPDATE products SET price = price * 1.1;  -- Trigger ignoré
SET @disable_triggers = 0;
```

### 7. ✅ Attention à la Performance

```sql
-- ❌ MAUVAIS : SELECT dans un trigger sur une grande table
CREATE TRIGGER bad_performance
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- Ceci s'exécute pour CHAQUE ligne !
    UPDATE customers
    SET order_count = (SELECT COUNT(*) FROM orders WHERE customer_id = NEW.customer_id)
    WHERE id = NEW.customer_id;
END;

-- ✅ BON : Incrément simple
CREATE TRIGGER good_performance
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    UPDATE customers
    SET order_count = order_count + 1
    WHERE id = NEW.customer_id;
END;
```

### 8. ✅ Éviter les Triggers pour les Imports de Masse

```sql
-- Pour les imports de masse, désactivez les triggers si possible

-- Méthode 1 : Utiliser LOAD DATA avec --disable-triggers (mysqlimport)
-- mysqlimport --disable-triggers ...

-- Méthode 2 : Créer une table temporaire sans triggers
CREATE TABLE temp_orders LIKE orders;
-- Pas de triggers sur temp_orders
LOAD DATA INFILE 'orders.csv' INTO TABLE temp_orders;
-- Ensuite insérer en une seule fois
INSERT INTO orders SELECT * FROM temp_orders;
DROP TABLE temp_orders;
```

---

## Limitations et Restrictions

### ❌ Ce qui n'est PAS Possible

```sql
-- 1. Pas de modification de la même table
-- ❌ ERREUR : Can't update table in trigger
CREATE TRIGGER impossible_self_update
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    UPDATE users SET login_count = 0 WHERE id = NEW.id;  -- Erreur !
END;

-- 2. Pas de TRUNCATE trigger
-- TRUNCATE ne déclenche PAS les triggers DELETE
TRUNCATE TABLE users;  -- Aucun trigger DELETE exécuté

-- 3. Pas de CALL vers procédures avec COMMIT/ROLLBACK
CREATE TRIGGER cannot_commit
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    CALL procedure_with_commit();  -- Peut causer des problèmes
END;

-- 4. Un seul trigger par timing/événement par table (limite historique levée)
-- MariaDB 10.2.3+ supporte plusieurs triggers
```

### ⚠️ Restrictions Importantes

1. **Triggers et Tables Temporaires** : Les triggers sur tables temporaires ne sont pas supportés.

2. **Triggers et Vues** : Les triggers sur vues ne sont pas supportés (utilisez INSTEAD OF triggers dans d'autres SGBD).

3. **Ordre d'Exécution** : Si plusieurs triggers existent pour le même timing/événement, l'ordre est déterminé par leur ordre de création.

```sql
-- MariaDB 10.2.3+ : Spécifier l'ordre
CREATE TRIGGER trigger1
BEFORE INSERT ON users
FOR EACH ROW
FOLLOWS trigger2  -- S'exécute après trigger2
BEGIN
    -- ...
END;
```

4. **Performance** : Les triggers s'exécutent **pour chaque ligne** (FOR EACH ROW), attention aux opérations de masse.

---

## Debugging et Maintenance

### Lister les Triggers

```sql
-- Tous les triggers de la base courante
SHOW TRIGGERS;

-- Triggers d'une table spécifique
SHOW TRIGGERS FROM mydb WHERE `Table` = 'users';

-- Détails complets
SHOW CREATE TRIGGER nom_trigger;

-- Via INFORMATION_SCHEMA
SELECT
    TRIGGER_NAME,
    EVENT_MANIPULATION,
    EVENT_OBJECT_TABLE,
    ACTION_TIMING,
    ACTION_STATEMENT,
    CREATED
FROM INFORMATION_SCHEMA.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb'
ORDER BY EVENT_OBJECT_TABLE, ACTION_TIMING, EVENT_MANIPULATION;
```

### Supprimer un Trigger

```sql
DROP TRIGGER IF EXISTS nom_trigger;
```

### Modifier un Trigger

```sql
-- Pas de ALTER TRIGGER en MariaDB
-- Il faut DROP puis CREATE

DROP TRIGGER IF EXISTS my_trigger;

CREATE TRIGGER my_trigger
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    -- Nouvelle version
END;
```

### Debugging avec Table de Log

```sql
-- Table de debug
CREATE TABLE trigger_debug_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    trigger_name VARCHAR(100),
    message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

DELIMITER $$

CREATE TRIGGER debug_example
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    -- Logger les valeurs pour debug
    INSERT INTO trigger_debug_log (trigger_name, message)
    VALUES (
        'debug_example',
        CONCAT('Customer ID: ', NEW.customer_id, ', Amount: ', NEW.total_amount)
    );

    -- Logique du trigger
    -- ...
END$$

DELIMITER ;
```

---

## 🔄 Triggers vs Alternatives

### Comparaison avec d'autres Approches

| Besoin | Trigger | Alternative | Recommandation |
|--------|---------|-------------|----------------|
| **Audit simple** | ✅ Excellent | Application | Trigger |
| **Validation simple** | ✅ Bon | CHECK constraint | CHECK ou Trigger |
| **Calculs complexes** | ⚠️ Possible | Application | Application |
| **Notifications** | ❌ Mauvais | Queue + Worker | Queue |
| **Synchronisation externe** | ❌ Mauvais | CDC (Debezium) | CDC |
| **Timestamps** | ✅ Excellent | DEFAULT CURRENT_TIMESTAMP | DEFAULT ou Trigger |
| **Logique métier complexe** | ⚠️ Déconseillé | Application | Application |

### Quand NE PAS Utiliser de Triggers

❌ **N'utilisez PAS de triggers pour :**

1. **Logique métier critique et changeante** → Application (plus flexible)
2. **Appels à des services externes** → Queue asynchrone
3. **Traitements longs** → Jobs batch
4. **Logique complexe multi-tables** → Procédures stockées + transaction
5. **Notifications utilisateur** → Application + websockets

✅ **Utilisez des triggers pour :**

1. **Audit et traçabilité** (logging automatique)
2. **Validation de données** (contraintes complexes)
3. **Synchronisation de données dénormalisées**
4. **Timestamps automatiques**
5. **Contraintes d'intégrité référentielle complexes**

---

## Exemple Complet : Système de Commandes

```sql
-- Tables
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    stock INT DEFAULT 0,
    reserved_stock INT DEFAULT 0,
    price DECIMAL(10,2)
) ENGINE=InnoDB;

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    status ENUM('pending', 'confirmed', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(10,2),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    total DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
) ENGINE=InnoDB;

CREATE TABLE order_history (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT,
    old_status VARCHAR(20),
    new_status VARCHAR(20),
    changed_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

DELIMITER $$

-- 1. Réserver le stock lors de l'ajout d'un item
CREATE TRIGGER reserve_stock_on_order_item
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    DECLARE available_stock INT;

    -- Vérifier le stock disponible
    SELECT stock - reserved_stock INTO available_stock
    FROM products
    WHERE id = NEW.product_id
    FOR UPDATE;

    IF available_stock < NEW.quantity THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Insufficient stock';
    END IF;

    -- Calculer le total
    SELECT price INTO NEW.unit_price
    FROM products
    WHERE id = NEW.product_id;

    SET NEW.total = NEW.quantity * NEW.unit_price;

    -- Réserver le stock
    UPDATE products
    SET reserved_stock = reserved_stock + NEW.quantity
    WHERE id = NEW.product_id;
END$$

-- 2. Libérer le stock si l'item est supprimé
CREATE TRIGGER release_stock_on_order_item_delete
AFTER DELETE ON order_items
FOR EACH ROW
BEGIN
    UPDATE products
    SET reserved_stock = reserved_stock - OLD.quantity
    WHERE id = OLD.product_id;
END$$

-- 3. Logger les changements de statut de commande
CREATE TRIGGER log_order_status_change
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
    IF OLD.status != NEW.status THEN
        INSERT INTO order_history (order_id, old_status, new_status)
        VALUES (NEW.id, OLD.status, NEW.status);

        -- Si confirmé, déduire du stock réel
        IF NEW.status = 'confirmed' THEN
            UPDATE products p
            JOIN order_items oi ON p.id = oi.product_id
            SET p.stock = p.stock - oi.quantity,
                p.reserved_stock = p.reserved_stock - oi.quantity
            WHERE oi.order_id = NEW.id;
        END IF;

        -- Si annulé, libérer le stock réservé
        IF NEW.status = 'cancelled' THEN
            UPDATE products p
            JOIN order_items oi ON p.id = oi.product_id
            SET p.reserved_stock = p.reserved_stock - oi.quantity
            WHERE oi.order_id = NEW.id;
        END IF;
    END IF;
END$$

DELIMITER ;
```

---

## ✅ Points clés à retenir

- 🎯 **Automatisation** : Les triggers s'exécutent automatiquement sans intervention de l'application
- 🔄 **Types** : 6 combinaisons (BEFORE/AFTER × INSERT/UPDATE/DELETE)
- 📊 **Variables** : OLD (anciennes valeurs) et NEW (nouvelles valeurs) selon le type
- 🛡️ **Validation** : BEFORE triggers pour valider, AFTER pour logger
- ⚡ **Performance** : Garder les triggers simples et rapides (<10ms)
- 🚫 **Limitations** : Pas de modification de la même table, attention aux cascades
- 📝 **Bonnes pratiques** : Documentation, nommage cohérent, logique simple
- ⚠️ **Cas d'usage** : Audit, validation, synchronisation, timestamps automatiques

---

## 🔗 Ressources et références

- [📖 CREATE TRIGGER - MariaDB Documentation](https://mariadb.com/kb/en/create-trigger/)
- [📖 Trigger Overview - MariaDB Documentation](https://mariadb.com/kb/en/trigger-overview/)
- [📖 Trigger Limitations - MariaDB Documentation](https://mariadb.com/kb/en/trigger-limitations/)
- [📖 DROP TRIGGER - MariaDB Documentation](https://mariadb.com/kb/en/drop-trigger/)
- [📝 Trigger Best Practices](https://mariadb.com/kb/en/trigger-best-practices/)

---

## ➡️ Section suivante

**8.3.1 BEFORE et AFTER** : Détails approfondis sur les différences entre triggers BEFORE et AFTER, cas d'usage spécifiques, et exemples avancés pour chaque timing.

---


⏭️ [BEFORE et AFTER](/08-programmation-cote-serveur/03.1-before-after.md)

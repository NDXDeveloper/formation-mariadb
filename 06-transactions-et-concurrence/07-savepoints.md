🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7 Savepoints : Points de Sauvegarde

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures

> **Prérequis** :
> - Section 6.1 (Propriétés ACID)
> - Section 6.2 (Gestion des transactions)
> - Compréhension des rollbacks

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le concept de savepoint et son utilité
- Maîtriser la syntaxe SAVEPOINT, ROLLBACK TO, RELEASE
- Implémenter des rollbacks partiels dans des transactions complexes
- Utiliser les savepoints imbriqués efficacement
- Appliquer les savepoints dans des cas d'usage production
- Identifier les limitations et les coûts des savepoints
- Implémenter des patterns de gestion d'erreur avec savepoints

---

## Introduction

Les **savepoints** (points de sauvegarde) sont des marqueurs placés à l'intérieur d'une transaction qui permettent d'effectuer des **rollbacks partiels** sans annuler toute la transaction. C'est comme créer des "checkpoints" dans un jeu vidéo : si vous échouez, vous revenez au dernier checkpoint au lieu de recommencer depuis le début.

### Le Problème Sans Savepoints

```sql
-- Scénario : Traitement multi-étapes
START TRANSACTION;

-- Étape 1 : Créer la commande (OK)
INSERT INTO commandes (client_id, total) VALUES (100, 500);

-- Étape 2 : Décrémenter stock produit A (OK)
UPDATE produits SET stock = stock - 1 WHERE id = 42;

-- Étape 3 : Décrémenter stock produit B (ERREUR : stock insuffisant)
UPDATE produits SET stock = stock - 1 WHERE id = 43;
-- 💥 Stock négatif !

-- Sans savepoints, seulement 2 options :
-- 1. ROLLBACK complet : Perd la commande ET le stock A ❌
-- 2. COMMIT avec erreur : Commande créée mais stock B incorrect ❌
```

### La Solution Avec Savepoints

```sql
START TRANSACTION;

-- Étape 1 : Créer la commande
INSERT INTO commandes (client_id, total) VALUES (100, 500);
SAVEPOINT apres_commande;  -- 📍 Point de sauvegarde

-- Étape 2 : Décrémenter stock produit A
UPDATE produits SET stock = stock - 1 WHERE id = 42;
SAVEPOINT apres_produit_A;  -- 📍 Point de sauvegarde

-- Étape 3 : Décrémenter stock produit B
UPDATE produits SET stock = stock - 1 WHERE id = 43;
-- 💥 Erreur détectée

-- Rollback seulement jusqu'au dernier savepoint
ROLLBACK TO SAVEPOINT apres_produit_A;
-- ✅ Garde : Commande + Stock A
-- ❌ Annule : Stock B

-- Marquer le produit B comme en rupture
UPDATE commande_items SET statut = 'rupture' WHERE produit_id = 43;

COMMIT;
-- ✅ Transaction partielle réussie
```

---

## 1. Syntaxe et Commandes de Base

### 1.1 SAVEPOINT : Créer un Point de Sauvegarde

```sql
-- Syntaxe
SAVEPOINT nom_savepoint;

-- Exemple
START TRANSACTION;

INSERT INTO logs (message) VALUES ('Début du traitement');
SAVEPOINT debut;  -- 📍 Marqueur créé

UPDATE produits SET prix = prix * 1.1;
SAVEPOINT apres_update;  -- 📍 Deuxième marqueur

DELETE FROM logs WHERE date < '2024-01-01';
SAVEPOINT apres_delete;  -- 📍 Troisième marqueur

COMMIT;
```

**Règles de nommage** :
- Identifier valide SQL (lettres, chiffres, underscore)
- Sensible à la casse
- Unique dans la transaction (réutiliser un nom écrase l'ancien)

### 1.2 ROLLBACK TO SAVEPOINT : Revenir en Arrière

```sql
-- Syntaxe
ROLLBACK TO SAVEPOINT nom_savepoint;
-- ou
ROLLBACK TO nom_savepoint;  -- TO est optionnel

-- Exemple
START TRANSACTION;

INSERT INTO users (id, name) VALUES (1, 'Alice');
SAVEPOINT user1;

INSERT INTO users (id, name) VALUES (2, 'Bob');
SAVEPOINT user2;

INSERT INTO users (id, name) VALUES (3, 'Charlie');

-- Erreur détectée sur Charlie, rollback jusqu'à user2
ROLLBACK TO SAVEPOINT user2;
-- ✅ Garde : Alice, Bob
-- ❌ Annule : Charlie

COMMIT;
-- Résultat final : Alice et Bob dans la table
```

**Comportement du ROLLBACK TO** :
1. **Annule** toutes les modifications après le savepoint
2. **Garde** le savepoint lui-même (peut être réutilisé)
3. **Supprime** tous les savepoints créés après celui-ci

```sql
START TRANSACTION;

SAVEPOINT sp1;
SAVEPOINT sp2;
SAVEPOINT sp3;

ROLLBACK TO sp2;
-- sp3 est supprimé
-- sp2 et sp1 existent toujours

ROLLBACK TO sp2;  -- ✅ OK, sp2 existe encore
ROLLBACK TO sp3;  -- ❌ ERROR: SAVEPOINT sp3 does not exist
```

### 1.3 RELEASE SAVEPOINT : Libérer un Savepoint

```sql
-- Syntaxe
RELEASE SAVEPOINT nom_savepoint;

-- Exemple
START TRANSACTION;

INSERT INTO orders (id) VALUES (1);
SAVEPOINT order1;

INSERT INTO order_items (order_id, product_id) VALUES (1, 42);

-- Si tout OK, libérer le savepoint (économise la mémoire)
RELEASE SAVEPOINT order1;

COMMIT;
```

**Pourquoi RELEASE ?**
- Chaque savepoint consomme de la mémoire
- RELEASE permet de nettoyer les savepoints intermédiaires
- Utile dans les transactions avec beaucoup de savepoints

**Différence RELEASE vs ROLLBACK TO** :

```sql
START TRANSACTION;

INSERT INTO t1 VALUES (1);
SAVEPOINT sp1;

INSERT INTO t1 VALUES (2);

-- Option 1 : ROLLBACK TO (garde le savepoint)
ROLLBACK TO sp1;
-- sp1 existe toujours, peut être réutilisé
ROLLBACK TO sp1;  -- ✅ OK

-- Option 2 : RELEASE (supprime le savepoint)
RELEASE SAVEPOINT sp1;
-- sp1 n'existe plus
ROLLBACK TO sp1;  -- ❌ ERROR: SAVEPOINT sp1 does not exist
```

---

## 2. Savepoints Imbriqués

### 2.1 Hiérarchie de Savepoints

Les savepoints peuvent être imbriqués à plusieurs niveaux :

```sql
START TRANSACTION;

-- Niveau 0 : Transaction principale
INSERT INTO logs (message) VALUES ('Niveau 0');
SAVEPOINT niveau1;

    -- Niveau 1
    INSERT INTO logs (message) VALUES ('Niveau 1');
    SAVEPOINT niveau2;

        -- Niveau 2
        INSERT INTO logs (message) VALUES ('Niveau 2');
        SAVEPOINT niveau3;

            -- Niveau 3
            INSERT INTO logs (message) VALUES ('Niveau 3');

-- Rollback au niveau 2
ROLLBACK TO niveau2;
-- ✅ Garde : Niveau 0, Niveau 1, Niveau 2
-- ❌ Annule : Niveau 3
-- 🗑️ Supprime : savepoint niveau3

-- Rollback au niveau 1
ROLLBACK TO niveau1;
-- ✅ Garde : Niveau 0, Niveau 1
-- ❌ Annule : Niveau 2
-- 🗑️ Supprime : savepoints niveau2 et niveau3

COMMIT;
```

### 2.2 Cas d'Usage : Traitement Récursif

```sql
-- Fonction récursive avec savepoints
DELIMITER //

CREATE FUNCTION process_hierarchy(
    p_node_id INT,
    p_level INT
) RETURNS BOOLEAN
BEGIN
    DECLARE v_child_id INT;
    DECLARE v_success BOOLEAN DEFAULT TRUE;
    DECLARE done BOOLEAN DEFAULT FALSE;

    DECLARE child_cursor CURSOR FOR
        SELECT child_id FROM tree WHERE parent_id = p_node_id;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    -- Créer un savepoint pour ce niveau
    SET @sp_name = CONCAT('level_', p_level, '_node_', p_node_id);
    SET @sql = CONCAT('SAVEPOINT ', @sp_name);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    -- Traiter le nœud actuel
    INSERT INTO processed_nodes (node_id, level) VALUES (p_node_id, p_level);

    -- Traiter les enfants
    OPEN child_cursor;

    child_loop: LOOP
        FETCH child_cursor INTO v_child_id;
        IF done THEN LEAVE child_loop; END IF;

        -- Appel récursif
        IF NOT process_hierarchy(v_child_id, p_level + 1) THEN
            -- Erreur dans un enfant, rollback ce niveau
            SET @sql = CONCAT('ROLLBACK TO ', @sp_name);
            PREPARE stmt FROM @sql;
            EXECUTE stmt;
            DEALLOCATE PREPARE stmt;

            SET v_success = FALSE;
            LEAVE child_loop;
        END IF;
    END LOOP;

    CLOSE child_cursor;

    RETURN v_success;
END//

DELIMITER ;

-- Utilisation
START TRANSACTION;
SELECT process_hierarchy(1, 0);  -- Process depuis la racine
COMMIT;
```

### 2.3 Visualisation de la Pile

```
État de la transaction avec savepoints imbriqués :

START TRANSACTION
    ↓
INSERT row 1
    ↓
SAVEPOINT sp1 ←──────────┐
    ↓                    │
INSERT row 2             │
    ↓                    │
SAVEPOINT sp2 ←──────┐   │
    ↓                │   │
INSERT row 3         │   │
    ↓                │   │
SAVEPOINT sp3 ←──┐   │   │
    ↓            │   │   │
INSERT row 4     │   │   │
    ↓            │   │   │
ERROR!           │   │   │
    ↓            │   │   │
ROLLBACK TO sp2 ─┘   │   │
    ↓ (sp3 supprimé) │   │
INSERT row 5         │   │
    ↓                │   │
ROLLBACK TO sp1 ─────┘   │
    ↓ (sp2 supprimé)     │
INSERT row 6             │
    ↓                    │
COMMIT                   │

Résultat final : row 1, row 6
```

---

## 3. Cas d'Usage Production

### 3.1 Traitement par Lots avec Reprise d'Erreur

```sql
DELIMITER //

CREATE PROCEDURE process_orders_batch(IN p_batch_size INT)
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_counter INT DEFAULT 0;
    DECLARE v_errors INT DEFAULT 0;
    DECLARE done BOOLEAN DEFAULT FALSE;

    DECLARE order_cursor CURSOR FOR
        SELECT id FROM orders WHERE status = 'pending'
        ORDER BY id
        LIMIT p_batch_size;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        -- En cas d'erreur, rollback jusqu'au dernier savepoint
        ROLLBACK TO SAVEPOINT before_order;
        SET v_errors = v_errors + 1;
    END;

    START TRANSACTION;

    OPEN order_cursor;

    order_loop: LOOP
        FETCH order_cursor INTO v_order_id;
        IF done THEN LEAVE order_loop; END IF;

        -- Savepoint avant chaque commande
        SAVEPOINT before_order;

        -- Traiter la commande (peut échouer)
        CALL process_single_order(v_order_id);

        -- Si succès, libérer le savepoint
        RELEASE SAVEPOINT before_order;

        SET v_counter = v_counter + 1;

        -- COMMIT partiel tous les 100 ordres
        IF MOD(v_counter, 100) = 0 THEN
            COMMIT;
            START TRANSACTION;
        END IF;
    END LOOP;

    CLOSE order_cursor;

    COMMIT;

    -- Log du résultat
    INSERT INTO batch_log (processed, errors, timestamp)
    VALUES (v_counter, v_errors, NOW());
END//

DELIMITER ;

-- Utilisation
CALL process_orders_batch(1000);
-- Traite 1000 commandes
-- Continue même si certaines échouent
-- Rollback seulement les commandes en erreur
```

### 3.2 Wizard Multi-Étapes

```sql
-- Scénario : E-commerce - Validation de commande en plusieurs étapes

DELIMITER //

CREATE PROCEDURE validate_and_create_order(
    IN p_client_id INT,
    IN p_items JSON,
    OUT p_order_id INT,
    OUT p_status VARCHAR(50)
)
BEGIN
    DECLARE v_item JSON;
    DECLARE v_product_id INT;
    DECLARE v_quantity INT;
    DECLARE v_stock INT;
    DECLARE v_idx INT DEFAULT 0;
    DECLARE v_items_count INT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_status = 'ERROR: Exception SQL';
        SET p_order_id = NULL;
    END;

    START TRANSACTION;

    -- Étape 1 : Vérifier le client
    IF NOT EXISTS (SELECT 1 FROM clients WHERE id = p_client_id AND statut = 'actif') THEN
        ROLLBACK;
        SET p_status = 'ERROR: Client invalide';
        SET p_order_id = NULL;
    ELSE
        SAVEPOINT client_valide;

        -- Étape 2 : Créer la commande
        INSERT INTO commandes (client_id, statut, created_at)
        VALUES (p_client_id, 'en_cours', NOW());

        SET p_order_id = LAST_INSERT_ID();
        SAVEPOINT commande_creee;

        -- Étape 3 : Traiter chaque article
        SET v_items_count = JSON_LENGTH(p_items);

        item_loop: WHILE v_idx < v_items_count DO
            SET v_item = JSON_EXTRACT(p_items, CONCAT('$[', v_idx, ']'));
            SET v_product_id = JSON_UNQUOTE(JSON_EXTRACT(v_item, '$.product_id'));
            SET v_quantity = JSON_UNQUOTE(JSON_EXTRACT(v_item, '$.quantity'));

            SAVEPOINT before_item;

            -- Vérifier le stock
            SELECT stock INTO v_stock
            FROM produits
            WHERE id = v_product_id
            FOR UPDATE;

            IF v_stock < v_quantity THEN
                -- Rollback cet article seulement
                ROLLBACK TO before_item;

                -- Enregistrer comme "en rupture"
                INSERT INTO commande_items
                (commande_id, produit_id, quantite, statut)
                VALUES (p_order_id, v_product_id, v_quantity, 'rupture');
            ELSE
                -- Décrémenter le stock
                UPDATE produits
                SET stock = stock - v_quantity
                WHERE id = v_product_id;

                -- Ajouter l'article
                INSERT INTO commande_items
                (commande_id, produit_id, quantite, statut)
                VALUES (p_order_id, v_product_id, v_quantity, 'valide');

                RELEASE SAVEPOINT before_item;
            END IF;

            SET v_idx = v_idx + 1;
        END WHILE item_loop;

        -- Étape 4 : Calculer le total
        UPDATE commandes
        SET total = (
            SELECT SUM(p.prix * ci.quantite)
            FROM commande_items ci
            JOIN produits p ON ci.produit_id = p.id
            WHERE ci.commande_id = p_order_id
              AND ci.statut = 'valide'
        )
        WHERE id = p_order_id;

        -- Étape 5 : Finaliser
        UPDATE commandes
        SET statut = 'confirmee'
        WHERE id = p_order_id;

        COMMIT;
        SET p_status = 'SUCCESS';
    END IF;
END//

DELIMITER ;

-- Utilisation
SET @order_id = NULL;
SET @status = NULL;

CALL validate_and_create_order(
    100,  -- client_id
    '[{"product_id": 42, "quantity": 2}, {"product_id": 43, "quantity": 1}]',
    @order_id,
    @status
);

SELECT @order_id, @status;
```

### 3.3 Import de Données avec Validation

```sql
DELIMITER //

CREATE PROCEDURE import_data_with_validation(
    IN p_file_path VARCHAR(255)
)
BEGIN
    DECLARE v_line TEXT;
    DECLARE v_line_number INT DEFAULT 0;
    DECLARE v_imported INT DEFAULT 0;
    DECLARE v_errors INT DEFAULT 0;
    DECLARE done BOOLEAN DEFAULT FALSE;

    -- Cursor pour lire les lignes (simulé)
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Rollback cette ligne seulement
        GET DIAGNOSTICS CONDITION 1
            @error_msg = MESSAGE_TEXT;

        ROLLBACK TO line_start;

        INSERT INTO import_errors (line_number, error_message)
        VALUES (v_line_number, @error_msg);

        SET v_errors = v_errors + 1;
    END;

    START TRANSACTION;

    -- Boucle de traitement
    import_loop: LOOP
        -- Lire la ligne suivante (simplifié)
        -- Dans la réalité, utiliser LOAD DATA ou un script externe

        IF done THEN LEAVE import_loop; END IF;

        SET v_line_number = v_line_number + 1;

        -- Savepoint avant chaque ligne
        SAVEPOINT line_start;

        -- Parser et insérer les données
        -- (Logique de parsing ici)

        -- Si succès
        RELEASE SAVEPOINT line_start;
        SET v_imported = v_imported + 1;

        -- COMMIT périodique
        IF MOD(v_line_number, 1000) = 0 THEN
            COMMIT;
            START TRANSACTION;
        END IF;
    END LOOP;

    COMMIT;

    -- Rapport
    SELECT
        v_line_number AS total_lines,
        v_imported AS imported,
        v_errors AS errors,
        ROUND((v_imported / v_line_number) * 100, 2) AS success_rate;
END//

DELIMITER ;
```

---

## 4. Limitations et Coûts

### 4.1 Coût en Mémoire

Chaque savepoint consomme de la mémoire :

```sql
-- Test de consommation
START TRANSACTION;

-- Créer 10000 savepoints
SET @i = 0;
WHILE @i < 10000 DO
    SET @sql = CONCAT('SAVEPOINT sp_', @i);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    SET @i = @i + 1;
END WHILE;

-- Consommation estimée : ~500KB - 1MB
-- ⚠️ Problème si trop de savepoints
```

**Recommandations** :
- Limiter à ~100-200 savepoints actifs par transaction
- Utiliser RELEASE SAVEPOINT pour nettoyer
- Éviter les savepoints dans les boucles infinies

### 4.2 Coût en Undo Log

Les savepoints génèrent des entrées undo log :

```sql
-- Scénario problématique
START TRANSACTION;

SAVEPOINT sp1;
UPDATE large_table SET col = col + 1;  -- 10M lignes
-- 💥 10M entrées undo créées

SAVEPOINT sp2;
UPDATE large_table SET col = col + 1;  -- 10M lignes
-- 💥 20M entrées undo au total

ROLLBACK TO sp1;
-- ✅ Rollback sp2 OK
-- ⚠️ Mais undo log très volumineux (plusieurs GB)
```

### 4.3 Pas de Savepoints Inter-Transactions

```sql
-- ❌ ERREUR : Savepoints perdus après COMMIT
START TRANSACTION;
SAVEPOINT sp1;
INSERT INTO t1 VALUES (1);
COMMIT;

-- Le savepoint n'existe plus
ROLLBACK TO sp1;  -- ERROR: SAVEPOINT sp1 does not exist

-- ✅ Les savepoints sont liés à la transaction
START TRANSACTION;
SAVEPOINT sp1;
INSERT INTO t1 VALUES (1);
ROLLBACK TO sp1;  -- OK, même transaction
COMMIT;
```

### 4.4 Limitations avec DDL

```sql
-- ⚠️ Les DDL causent un COMMIT implicite
START TRANSACTION;
INSERT INTO users (name) VALUES ('Alice');
SAVEPOINT after_alice;

CREATE TABLE temp (id INT);  -- 💥 COMMIT implicite !

-- Le savepoint est perdu
ROLLBACK TO after_alice;  -- ERROR: SAVEPOINT does not exist
```

### 4.5 Performance : Savepoints vs Sous-Transactions

```sql
-- Benchmark : 1000 opérations

-- Test 1 : Avec savepoints (1 transaction)
START TRANSACTION;
FOR i IN 1..1000 LOOP
    SAVEPOINT sp;
    INSERT INTO test VALUES (i);
    RELEASE SAVEPOINT sp;
END LOOP;
COMMIT;
-- Durée : ~500ms

-- Test 2 : Sous-transactions (1000 transactions)
FOR i IN 1..1000 LOOP
    START TRANSACTION;
    INSERT INTO test VALUES (i);
    COMMIT;
END LOOP;
-- Durée : ~5000ms (10x plus lent)

-- ✅ Savepoints sont plus rapides que des transactions multiples
-- Mais plus lents qu'une transaction simple sans savepoints
```

---

## 5. Patterns et Bonnes Pratiques

### 5.1 Pattern : Try-Catch avec Savepoints

```sql
DELIMITER //

CREATE PROCEDURE safe_update(
    IN p_id INT,
    IN p_new_value VARCHAR(100)
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Rollback au savepoint si erreur
        ROLLBACK TO SAVEPOINT before_update;

        -- Logger l'erreur
        GET DIAGNOSTICS CONDITION 1
            @error_code = RETURNED_SQLSTATE,
            @error_msg = MESSAGE_TEXT;

        INSERT INTO error_log (operation, error_code, error_message)
        VALUES ('safe_update', @error_code, @error_msg);

        -- Retourner FALSE
        SELECT FALSE AS success;
    END;

    -- Savepoint avant l'opération risquée
    SAVEPOINT before_update;

    -- Opération qui peut échouer
    UPDATE sensitive_table
    SET value = p_new_value
    WHERE id = p_id;

    -- Si succès, libérer le savepoint
    RELEASE SAVEPOINT before_update;

    SELECT TRUE AS success;
END//

DELIMITER ;
```

### 5.2 Pattern : Accumulation Progressive

```sql
DELIMITER //

CREATE PROCEDURE progressive_insert(
    IN p_data JSON
)
BEGIN
    DECLARE v_idx INT DEFAULT 0;
    DECLARE v_count INT;
    DECLARE v_item JSON;
    DECLARE v_inserted INT DEFAULT 0;

    SET v_count = JSON_LENGTH(p_data);

    START TRANSACTION;

    WHILE v_idx < v_count DO
        SAVEPOINT item_start;

        SET v_item = JSON_EXTRACT(p_data, CONCAT('$[', v_idx, ']'));

        BEGIN
            -- Bloc avec gestion d'erreur locale
            DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
            BEGIN
                -- Rollback cet item
                ROLLBACK TO item_start;
            END;

            -- Insérer l'item
            INSERT INTO items (data) VALUES (v_item);

            -- Si succès, compter et libérer
            RELEASE SAVEPOINT item_start;
            SET v_inserted = v_inserted + 1;
        END;

        SET v_idx = v_idx + 1;
    END WHILE;

    COMMIT;

    -- Retourner le nombre d'items insérés
    SELECT v_inserted AS inserted_count, v_count AS total_count;
END//

DELIMITER ;
```

### 5.3 Pattern : Opération Compensatoire

```sql
DELIMITER //

CREATE PROCEDURE transfer_with_compensation(
    IN p_from_account INT,
    IN p_to_account INT,
    IN p_amount DECIMAL(10,2)
)
BEGIN
    DECLARE v_balance DECIMAL(10,2);

    START TRANSACTION;

    -- Savepoint initial
    SAVEPOINT transfer_start;

    -- Étape 1 : Débiter le compte source
    UPDATE accounts
    SET balance = balance - p_amount
    WHERE id = p_from_account;

    -- Vérifier le solde
    SELECT balance INTO v_balance
    FROM accounts
    WHERE id = p_from_account;

    IF v_balance < 0 THEN
        -- Solde négatif : rollback
        ROLLBACK TO transfer_start;
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Solde insuffisant';
    END IF;

    SAVEPOINT after_debit;

    -- Étape 2 : Créditer le compte destination
    BEGIN
        DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
        BEGIN
            -- Si erreur, compenser le débit
            ROLLBACK TO after_debit;

            -- Re-créditer le compte source (compensation)
            UPDATE accounts
            SET balance = balance + p_amount
            WHERE id = p_from_account;

            -- Annuler toute la transaction
            ROLLBACK;

            SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Transfert échoué, compensation appliquée';
        END;

        UPDATE accounts
        SET balance = balance + p_amount
        WHERE id = p_to_account;
    END;

    -- Log de la transaction
    INSERT INTO transfer_log (from_account, to_account, amount, timestamp)
    VALUES (p_from_account, p_to_account, p_amount, NOW());

    COMMIT;
END//

DELIMITER ;
```

### 5.4 Best Practices : Checklist

```sql
-- ✅ À FAIRE

-- 1. Libérer les savepoints inutiles
SAVEPOINT sp1;
-- ... opérations ...
RELEASE SAVEPOINT sp1;  -- Économise mémoire

-- 2. Nommer clairement les savepoints
SAVEPOINT after_user_creation;  -- ✅ Clair
SAVEPOINT sp1;                  -- ❌ Peu clair

-- 3. Documenter la logique
SAVEPOINT before_critical_update;  -- Point de retour si échec
-- ... opération critique ...

-- 4. Limiter la profondeur d'imbrication
-- Maximum 3-5 niveaux pour la lisibilité

-- 5. COMMIT périodique dans les longs traitements
IF MOD(counter, 1000) = 0 THEN
    COMMIT;
    START TRANSACTION;
END IF;

-- ❌ À ÉVITER

-- 1. Trop de savepoints
FOR i IN 1..10000 LOOP
    SAVEPOINT sp;  -- ❌ 10000 savepoints !
END LOOP;

-- 2. Savepoints dans des transactions courtes
START TRANSACTION;
SAVEPOINT sp1;  -- ❌ Inutile pour 1 seule opération
INSERT INTO t1 VALUES (1);
COMMIT;

-- 3. Oublier de RELEASE
SAVEPOINT sp1;
-- ... beaucoup d'opérations ...
-- ❌ sp1 reste en mémoire inutilement

-- 4. Noms génériques
SAVEPOINT sp;  -- ❌ Quel est le contexte ?

-- 5. Savepoints avant DDL
SAVEPOINT sp1;
CREATE TABLE t;  -- ❌ COMMIT implicite, sp1 perdu
```

---

## 6. Comparaison avec les Alternatives

### 6.1 Savepoints vs Sous-Transactions

| Critère | Savepoints | Sous-Transactions |
|---------|-----------|-------------------|
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐ |
| **Simplicité** | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Rollback partiel** | ✅ Oui | ❌ Non (tout ou rien) |
| **Overhead** | Faible | Élevé (COMMIT/BEGIN) |
| **Coût undo log** | Moyen | Faible |
| **Cas d'usage** | Traitements complexes | Opérations simples |

### 6.2 Savepoints vs Try-Catch Applicatif

```python
# Approche 1 : Savepoints (dans la base)
def process_items_with_savepoints(items):
    conn.execute("START TRANSACTION")

    for item in items:
        conn.execute(f"SAVEPOINT item_{item.id}")
        try:
            conn.execute(f"INSERT INTO items VALUES ({item.id}, ...)")
            conn.execute(f"RELEASE SAVEPOINT item_{item.id}")
        except:
            conn.execute(f"ROLLBACK TO item_{item.id}")

    conn.execute("COMMIT")

# Approche 2 : Try-Catch applicatif
def process_items_with_subtransactions(items):
    for item in items:
        conn.execute("START TRANSACTION")
        try:
            conn.execute(f"INSERT INTO items VALUES ({item.id}, ...)")
            conn.execute("COMMIT")
        except:
            conn.execute("ROLLBACK")

# ✅ Savepoints : Plus rapide (1 transaction)
# ⚠️ Try-Catch : Plus simple mais plus lent (N transactions)
```

---

## ✅ Points clés à retenir

- **Savepoint** : Marqueur permettant un rollback partiel
- **ROLLBACK TO** : Annule jusqu'au savepoint, garde le savepoint
- **RELEASE** : Supprime le savepoint, économise mémoire
- **Imbrication** : Savepoints peuvent être hiérarchiques
- **Coût** : Mémoire + undo log (limiter à ~100-200 savepoints)
- **Limite** : Savepoints perdus après COMMIT/ROLLBACK complet
- **Use case** : Traitement multi-étapes avec reprise d'erreur
- **Performance** : Plus rapide que sous-transactions multiples
- **Best practice** : Nommer clairement, RELEASE quand inutile
- **Pattern** : Try-catch avec savepoint pour gestion d'erreur

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 SAVEPOINT](https://mariadb.com/kb/en/savepoint/)
- [📖 ROLLBACK TO SAVEPOINT](https://mariadb.com/kb/en/rollback/)
- [📖 RELEASE SAVEPOINT](https://mariadb.com/kb/en/release-savepoint/)

### SQL Standard
- ISO/IEC 9075-2 (SQL/Foundation) - Section sur les savepoints

---

## ➡️ Section suivante

**6.8 Transactions distribuées (XA)** : Coordination de transactions sur plusieurs bases de données avec le protocole 2PC.

---


⏭️ [Transactions distribuées (XA)](/06-transactions-et-concurrence/08-transactions-distribuees-xa.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Gestion des Transactions (START TRANSACTION, BEGIN, COMMIT, ROLLBACK)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures

> **Prérequis** :
> - Section 6.1 (Propriétés ACID)
> - Compréhension des niveaux d'isolation
> - Notions de programmation SQL

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser la syntaxe complète des commandes transactionnelles
- Utiliser les options avancées (CONSISTENT SNAPSHOT, READ ONLY)
- Implémenter des savepoints pour des rollbacks partiels
- Gérer correctement les erreurs dans les transactions
- Comprendre l'impact du mode autocommit
- Appliquer les bonnes pratiques transactionnelles en production
- Optimiser les performances transactionnelles

---

## Introduction

La **gestion des transactions** est l'art de contrôler précisément quand et comment les modifications de données sont validées ou annulées. Une maîtrise parfaite de ces commandes est essentielle pour construire des applications fiables et performantes.

Cette section couvre tous les aspects pratiques de la gestion transactionnelle : de la syntaxe de base aux options avancées, en passant par les pièges courants et les optimisations production.

---

## 1. START TRANSACTION et BEGIN

### 1.1 Syntaxe de Base

MariaDB propose deux syntaxes équivalentes pour démarrer une transaction :

```sql
-- Syntaxe SQL standard (recommandée)
START TRANSACTION;

-- Syntaxe alternative (compatibilité MySQL)
BEGIN;
-- ou
BEGIN WORK;

-- Les trois sont strictement équivalentes
```

💡 **Recommandation** : Utiliser `START TRANSACTION` pour la clarté et la conformité SQL standard.

### 1.2 Anatomie d'une Transaction Simple

```sql
-- Démarrage
START TRANSACTION;

-- Zone transactionnelle : toutes les modifications sont temporaires
INSERT INTO commandes (client_id, montant) VALUES (100, 250.00);
UPDATE stocks SET quantite = quantite - 1 WHERE produit_id = 42;
INSERT INTO logs (action, timestamp) VALUES ('commande_créée', NOW());

-- Validation : rend les modifications permanentes
COMMIT;

-- Après le COMMIT, la transaction est terminée
-- Une nouvelle transaction démarre automatiquement (si autocommit=1)
```

### 1.3 Options Avancées de START TRANSACTION

MariaDB offre plusieurs options pour contrôler le comportement de la transaction :

```sql
-- Syntaxe complète
START TRANSACTION
    [WITH CONSISTENT SNAPSHOT]
    [READ WRITE | READ ONLY];
```

#### 1.3.1 WITH CONSISTENT SNAPSHOT

**Crée un snapshot cohérent au moment du START TRANSACTION** :

```sql
-- Cas d'usage : Rapport cohérent sur des données qui changent rapidement
START TRANSACTION WITH CONSISTENT SNAPSHOT;

-- Cette transaction voit un "snapshot" figé de la base
SELECT COUNT(*) FROM commandes;  -- Compte à l'instant T0
-- Même si 1000 nouvelles commandes sont insérées par d'autres sessions...
SELECT COUNT(*) FROM commandes;  -- Même résultat qu'avant
-- Vue cohérente pendant toute la transaction

COMMIT;
```

**Différence avec START TRANSACTION normal** :

```sql
-- SANS WITH CONSISTENT SNAPSHOT
START TRANSACTION;
-- Le snapshot est créé à la PREMIÈRE lecture
SELECT COUNT(*) FROM commandes;  -- ← Snapshot créé ICI
-- Les lignes insérées AVANT cette lecture mais APRÈS le START ne sont pas vues

-- AVEC WITH CONSISTENT SNAPSHOT
START TRANSACTION WITH CONSISTENT SNAPSHOT;
-- Le snapshot est créé IMMÉDIATEMENT au START TRANSACTION
-- Garantit une vue 100% cohérente dès le début
```

**Cas d'usage en production** :

```sql
-- Backup logique cohérent avec mysqldump
-- mysqldump utilise --single-transaction qui fait :
START TRANSACTION WITH CONSISTENT SNAPSHOT;
-- Dump de toutes les tables avec vue cohérente
COMMIT;

-- Rapport financier multi-tables
START TRANSACTION WITH CONSISTENT SNAPSHOT;
SELECT
    (SELECT SUM(debit) FROM transactions) AS total_debits,
    (SELECT SUM(credit) FROM transactions) AS total_credits,
    (SELECT COUNT(*) FROM comptes WHERE solde < 0) AS comptes_negatifs;
-- Les trois requêtes voient le même instant T
COMMIT;
```

⚠️ **Important** : `WITH CONSISTENT SNAPSHOT` ne fonctionne qu'avec le moteur InnoDB et le niveau d'isolation REPEATABLE READ ou supérieur.

#### 1.3.2 READ WRITE vs READ ONLY

**Optimisation pour les transactions en lecture seule** :

```sql
-- Transaction en lecture seule (optimisation performance)
START TRANSACTION READ ONLY;

-- Seules les lectures sont autorisées
SELECT * FROM produits WHERE categorie = 'Électronique';
SELECT AVG(prix) FROM produits;

-- ❌ Tentative d'écriture : ERREUR
UPDATE produits SET prix = prix * 1.1 WHERE id = 1;
-- ERROR 1792 (25006): Cannot execute statement in a READ ONLY transaction

COMMIT;

-- Transaction en lecture/écriture (par défaut)
START TRANSACTION READ WRITE;
-- Lectures ET écritures autorisées
SELECT * FROM produits WHERE id = 1;
UPDATE produits SET prix = 99.99 WHERE id = 1;
COMMIT;
```

**Avantages de READ ONLY** :

1. **Performance** : InnoDB optimise les lectures sans gérer les verrous d'écriture
2. **Sécurité** : Garantit qu'aucune modification accidentelle ne sera faite
3. **Clarté** : Documente l'intention du code

**Exemple production : Rapports et analytics**

```sql
-- Service de reporting
START TRANSACTION READ ONLY WITH CONSISTENT SNAPSHOT;

-- Calculs complexes sans risque de modification
SELECT
    DATE(created_at) AS date,
    COUNT(*) AS nb_commandes,
    SUM(montant) AS ca_jour
FROM commandes
WHERE created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(created_at)
ORDER BY date;

-- Stockage du rapport dans une table
-- ❌ Impossible ici car READ ONLY
-- ✅ Solution : faire le calcul en READ ONLY, puis nouvelle transaction pour l'INSERT

COMMIT;

-- Nouvelle transaction pour l'écriture
START TRANSACTION READ WRITE;
INSERT INTO rapports_quotidiens (date, nb_commandes, ca)
VALUES (...);
COMMIT;
```

### 1.4 Combinaison des Options

```sql
-- Toutes les options ensemble
START TRANSACTION
    WITH CONSISTENT SNAPSHOT
    READ ONLY;

-- Cas d'usage typique : Export cohérent de données
-- mysqldump --single-transaction utilise exactement ceci
```

---

## 2. COMMIT : Validation des Transactions

### 2.1 Syntaxe et Comportement

```sql
-- Syntaxe standard
COMMIT;

-- Syntaxe alternative (même effet)
COMMIT WORK;
```

**Ce qui se passe lors d'un COMMIT** :

```
1. InnoDB écrit les modifications dans le redo log
2. Redo log est synchronisé sur disque (fsync)
3. Les verrous sont libérés
4. La transaction est marquée comme commitée
5. Le COMMIT retourne "OK" à l'application
6. Les modifications deviennent visibles aux autres transactions
```

### 2.2 COMMIT et Durabilité

Le comportement du COMMIT dépend de `innodb_flush_log_at_trx_commit` :

```sql
-- Vérifier la configuration actuelle
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';

-- Mode 1 : Durabilité maximale (DEFAULT)
-- COMMIT écrit ET synchronise le redo log immédiatement
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
START TRANSACTION;
INSERT INTO logs (message) VALUES ('important');
COMMIT;  -- fsync() appelé, données durables sur disque
-- ✅ Aucune perte possible, même si crash 1ms après

-- Mode 2 : Compromis
-- COMMIT écrit le redo log, mais fsync() toutes les 1s
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
START TRANSACTION;
INSERT INTO logs (message) VALUES ('moins critique');
COMMIT;  -- Écrit en mémoire OS, pas forcément sur disque
-- ⚠️ Perte possible si crash OS dans la seconde qui suit

-- Mode 0 : Performance maximale
-- Écriture et fsync() toutes les 1s, pas au COMMIT
SET GLOBAL innodb_flush_log_at_trx_commit = 0;
START TRANSACTION;
INSERT INTO logs (message) VALUES ('non critique');
COMMIT;  -- Très rapide, mais pas durable immédiatement
-- ⚠️ Perte possible de ~1 seconde de données
```

### 2.3 COMMIT Implicite

⚠️ **Attention** : Certaines commandes DDL provoquent un **COMMIT implicite** automatique :

```sql
START TRANSACTION;

INSERT INTO produits (nom, prix) VALUES ('Widget', 10.00);

-- ⚠️ Cette commande COMMIT automatiquement la transaction en cours !
CREATE TABLE temp_data (id INT);

-- La transaction précédente est déjà commitée
-- L'INSERT est devenu permanent, impossible de faire ROLLBACK

-- Autres commandes causant un COMMIT implicite :
-- - CREATE, ALTER, DROP (TABLE, DATABASE, INDEX, etc.)
-- - TRUNCATE TABLE
-- - LOCK TABLES, UNLOCK TABLES
-- - LOAD DATA
```

**Liste complète des commandes avec COMMIT implicite** :

```sql
-- DDL (Data Definition Language)
CREATE DATABASE, ALTER DATABASE, DROP DATABASE
CREATE TABLE, ALTER TABLE, DROP TABLE, RENAME TABLE
CREATE INDEX, DROP INDEX
CREATE VIEW, DROP VIEW
CREATE PROCEDURE, ALTER PROCEDURE, DROP PROCEDURE
CREATE FUNCTION, ALTER FUNCTION, DROP FUNCTION
CREATE TRIGGER, DROP TRIGGER
CREATE EVENT, ALTER EVENT, DROP EVENT

-- Autres commandes
TRUNCATE TABLE
LOAD DATA
LOCK TABLES
UNLOCK TABLES
SET autocommit = 1  -- Si auparavant = 0
```

💡 **Bonne pratique** : Ne jamais mélanger DDL et DML dans la même transaction logique.

```sql
-- ❌ MAUVAIS : Mélange DDL et DML
START TRANSACTION;
INSERT INTO users (name) VALUES ('Alice');
CREATE TEMPORARY TABLE temp (id INT);  -- COMMIT implicite !
INSERT INTO users (name) VALUES ('Bob');
ROLLBACK;  -- Ne rollback que 'Bob', pas 'Alice' !

-- ✅ BON : Séparer DDL et DML
CREATE TEMPORARY TABLE temp (id INT);

START TRANSACTION;
INSERT INTO users (name) VALUES ('Alice');
INSERT INTO users (name) VALUES ('Bob');
ROLLBACK;  -- Rollback des deux INSERTs
```

### 2.4 COMMIT AND CHAIN

**Commencer une nouvelle transaction immédiatement après le COMMIT** :

```sql
-- Syntaxe
COMMIT AND CHAIN;

-- Équivalent à :
COMMIT;
START TRANSACTION;

-- Cas d'usage : Traitement par lots
START TRANSACTION;

-- Traiter un lot de 1000 lignes
UPDATE commandes SET statut = 'traité' WHERE id BETWEEN 1 AND 1000;

COMMIT AND CHAIN;  -- Commit le lot 1, start transaction pour le lot 2

-- Traiter le lot suivant
UPDATE commandes SET statut = 'traité' WHERE id BETWEEN 1001 AND 2000;

COMMIT AND CHAIN;  -- Commit le lot 2, start transaction pour le lot 3

-- ...

COMMIT;  -- Dernier commit
```

**Avantages** :
- ✅ Pas d'interruption entre les transactions
- ✅ Code plus lisible (moins de START TRANSACTION)
- ✅ Évite les fenêtres sans transaction

---

## 3. ROLLBACK : Annulation des Transactions

### 3.1 Syntaxe et Comportement

```sql
-- Syntaxe standard
ROLLBACK;

-- Syntaxe alternative (même effet)
ROLLBACK WORK;
```

**Ce qui se passe lors d'un ROLLBACK** :

```
1. InnoDB lit l'undo log
2. Restaure toutes les modifications à leur état "avant"
3. Libère tous les verrous
4. La transaction est marquée comme annulée
5. Aucune modification n'est visible aux autres transactions
```

### 3.2 Exemple Complet

```sql
START TRANSACTION;

-- Modifications en cours
INSERT INTO commandes (client_id, montant) VALUES (100, 150.00);
UPDATE stocks SET quantite = quantite - 5 WHERE produit_id = 10;

-- Vérification métier
SELECT @stock := quantite FROM stocks WHERE produit_id = 10;

IF @stock < 0 THEN
    -- Stock négatif : annuler tout
    ROLLBACK;
    SELECT 'Commande annulée : stock insuffisant' AS message;
ELSE
    -- OK : valider
    COMMIT;
    SELECT 'Commande enregistrée' AS message;
END IF;
```

### 3.3 ROLLBACK Automatique

MariaDB effectue un **ROLLBACK automatique** dans certains cas :

```sql
START TRANSACTION;

INSERT INTO produits (id, nom, prix) VALUES (1, 'Widget', 10.00);

-- Violation de contrainte
INSERT INTO produits (id, nom, prix) VALUES (1, 'Gadget', 20.00);
-- ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'

-- ⚠️ ROLLBACK AUTOMATIQUE de toute la transaction
-- Les deux INSERTs sont annulés

-- Tenter un COMMIT maintenant :
COMMIT;
-- Aucun effet : la transaction a déjà été rollbackée
```

**Comportement par défaut** :

```sql
-- Vérifier le comportement sur erreur
SHOW VARIABLES LIKE 'innodb_rollback_on_timeout';
-- ON : rollback de toute la transaction sur timeout
-- OFF : rollback seulement de l'instruction qui a timeout

SHOW VARIABLES LIKE 'completion_type';
-- 0 : COMMIT termine la transaction (default)
-- 1 : COMMIT AND CHAIN automatique
-- 2 : COMMIT AND RELEASE (ferme la connexion)
```

### 3.4 ROLLBACK AND CHAIN

```sql
-- Rollback puis start d'une nouvelle transaction
ROLLBACK AND CHAIN;

-- Équivalent à :
ROLLBACK;
START TRANSACTION;

-- Cas d'usage : Retry après erreur
DECLARE retry_count INT DEFAULT 0;
DECLARE max_retries INT DEFAULT 3;

retry_loop: LOOP
    START TRANSACTION;

    -- Opération à risque
    UPDATE comptes SET solde = solde - 100 WHERE id = 1;

    IF @solde < 0 THEN
        ROLLBACK AND CHAIN;
        SET retry_count = retry_count + 1;

        IF retry_count >= max_retries THEN
            LEAVE retry_loop;
        END IF;
    ELSE
        COMMIT;
        LEAVE retry_loop;
    END IF;
END LOOP;
```

---

## 4. SAVEPOINT : Rollback Partiel

### 4.1 Concept et Syntaxe

Les **savepoints** permettent de créer des **points de sauvegarde** dans une transaction, permettant un rollback partiel sans annuler toute la transaction.

```sql
-- Créer un savepoint
SAVEPOINT nom_savepoint;

-- Rollback jusqu'au savepoint
ROLLBACK TO SAVEPOINT nom_savepoint;

-- Libérer un savepoint (optionnel, pour économiser la mémoire)
RELEASE SAVEPOINT nom_savepoint;
```

### 4.2 Exemple Pratique : Traitement par Étapes

```sql
START TRANSACTION;

-- Étape 1 : Créer la commande
INSERT INTO commandes (client_id, montant) VALUES (100, 500.00);
SET @commande_id = LAST_INSERT_ID();

SAVEPOINT apres_commande;

-- Étape 2 : Décrémenter le stock (peut échouer)
UPDATE stocks SET quantite = quantite - 10 WHERE produit_id = 42;

SELECT @stock := quantite FROM stocks WHERE produit_id = 42;

IF @stock < 0 THEN
    -- Rollback seulement l'étape 2
    ROLLBACK TO SAVEPOINT apres_commande;

    -- Marquer la commande comme "en attente de stock"
    UPDATE commandes SET statut = 'en_attente' WHERE id = @commande_id;

    COMMIT;  -- Commande créée, mais pas de décrémentation stock
ELSE
    -- Tout OK : finaliser la commande
    UPDATE commandes SET statut = 'confirmée' WHERE id = @commande_id;

    COMMIT;  -- Commande + décrémentation stock
END IF;
```

### 4.3 Savepoints Imbriqués

```sql
START TRANSACTION;

-- Niveau 1
INSERT INTO logs (message) VALUES ('Début du traitement');
SAVEPOINT niveau1;

-- Niveau 2
UPDATE produits SET stock = stock - 1 WHERE id = 1;
SAVEPOINT niveau2;

-- Niveau 3
UPDATE produits SET stock = stock - 1 WHERE id = 2;
SAVEPOINT niveau3;

-- Erreur au niveau 3
SELECT @stock := stock FROM produits WHERE id = 2;
IF @stock < 0 THEN
    -- Rollback seulement niveau 3
    ROLLBACK TO SAVEPOINT niveau2;
    -- Les niveaux 1 et 2 sont toujours actifs
END IF;

-- Continuer le traitement
UPDATE produits SET stock = stock - 1 WHERE id = 3;

COMMIT;
-- Résultat : logs + update produit 1 + update produit 3
-- Pas de update produit 2 (rollbacké)
```

### 4.4 Gestion des Savepoints en Production

```python
# Exemple Python : Traitement de commande avec savepoints
import pymysql

conn = pymysql.connect(...)
cursor = conn.cursor()

try:
    cursor.execute("START TRANSACTION")

    # Étape 1 : Créer la commande
    cursor.execute(
        "INSERT INTO commandes (client_id, montant) VALUES (%s, %s)",
        (client_id, montant_total)
    )
    commande_id = cursor.lastrowid

    cursor.execute("SAVEPOINT apres_commande")

    # Étape 2 : Traiter chaque produit
    for item in panier:
        try:
            cursor.execute("SAVEPOINT avant_produit")

            cursor.execute(
                "UPDATE stocks SET quantite = quantite - %s WHERE produit_id = %s",
                (item['quantite'], item['produit_id'])
            )

            cursor.execute(
                "SELECT quantite FROM stocks WHERE produit_id = %s",
                (item['produit_id'],)
            )
            stock = cursor.fetchone()[0]

            if stock < 0:
                # Rollback ce produit, mais garder la commande
                cursor.execute("ROLLBACK TO SAVEPOINT avant_produit")
                print(f"Produit {item['produit_id']} : stock insuffisant")
            else:
                # OK : insérer la ligne de commande
                cursor.execute(
                    "INSERT INTO commande_items (commande_id, produit_id, quantite) "
                    "VALUES (%s, %s, %s)",
                    (commande_id, item['produit_id'], item['quantite'])
                )

        except Exception as e:
            # Rollback ce produit
            cursor.execute("ROLLBACK TO SAVEPOINT avant_produit")
            print(f"Erreur produit {item['produit_id']}: {e}")

    # Commit final
    cursor.execute("COMMIT")
    print(f"Commande {commande_id} validée")

except Exception as e:
    cursor.execute("ROLLBACK")
    print(f"Erreur globale : {e}")
finally:
    cursor.close()
    conn.close()
```

### 4.5 Limites des Savepoints

⚠️ **Limitations importantes** :

1. **Pas de savepoints inter-transactions** :
```sql
START TRANSACTION;
SAVEPOINT sp1;
COMMIT;

-- ❌ Erreur : le savepoint n'existe plus après COMMIT
ROLLBACK TO SAVEPOINT sp1;
-- ERROR 1305 (42000): SAVEPOINT sp1 does not exist
```

2. **Consommation mémoire** :
```sql
-- Trop de savepoints peut consommer beaucoup de mémoire
START TRANSACTION;
-- Créer 10000 savepoints : ❌ Mauvaise pratique
FOR i IN 1..10000 LOOP
    SAVEPOINT CONCAT('sp_', i);
    -- ...
END LOOP;
-- ✅ Mieux : utiliser RELEASE SAVEPOINT après usage
```

3. **Performance** :
```sql
-- Chaque savepoint a un coût
-- Benchmark : 1000 savepoints vs 1 transaction
-- Savepoints : ~500ms
-- Transaction unique : ~50ms
-- ⚠️ Utiliser avec modération
```

---

## 5. Mode Autocommit

### 5.1 Comportement par Défaut

```sql
-- Vérifier l'état actuel
SELECT @@autocommit;  -- 1 = activé (défaut)

-- Avec autocommit=1, chaque instruction est une transaction automatique
UPDATE produits SET prix = 99.99 WHERE id = 1;
-- Équivalent à :
-- START TRANSACTION;
-- UPDATE produits SET prix = 99.99 WHERE id = 1;
-- COMMIT;

-- Désactiver autocommit
SET autocommit = 0;

-- Maintenant, les instructions ne sont pas commitées automatiquement
UPDATE produits SET prix = 89.99 WHERE id = 2;
-- Pas encore visible pour les autres sessions
-- Il faut un COMMIT explicite

COMMIT;  -- Maintenant visible
```

### 5.2 Autocommit et Transactions Explicites

```sql
-- Même avec autocommit=1, START TRANSACTION le désactive temporairement
SET autocommit = 1;

START TRANSACTION;
-- autocommit est ignoré dans cette transaction
UPDATE produits SET prix = 99.99 WHERE id = 1;
UPDATE produits SET stock = 50 WHERE id = 1;
-- Pas de commit automatique entre les deux UPDATE
COMMIT;
-- Fin de la transaction, autocommit redevient actif
```

### 5.3 Impact sur la Performance

```sql
-- ❌ MAUVAIS : Autocommit avec boucle (1000 commits)
SET autocommit = 1;
FOR i IN 1..1000 LOOP
    INSERT INTO logs (message) VALUES (CONCAT('Log ', i));
    -- COMMIT automatique à chaque itération
END LOOP;
-- Performance : ~5 secondes (avec innodb_flush_log_at_trx_commit=1)

-- ✅ BON : Transaction unique (1 commit)
SET autocommit = 0;
START TRANSACTION;
FOR i IN 1..1000 LOOP
    INSERT INTO logs (message) VALUES (CONCAT('Log ', i));
    -- Pas de commit
END LOOP;
COMMIT;  -- Un seul commit pour tout
-- Performance : ~0.5 secondes (10x plus rapide)
```

### 5.4 Autocommit et Connexions Longues

```sql
-- ⚠️ Problème avec autocommit=0 et pas de COMMIT
SET autocommit = 0;

-- Session reste ouverte longtemps
SELECT * FROM produits WHERE id = 1;
-- Transaction démarrée implicitement
-- Verrous potentiellement gardés
-- Undo log qui s'accumule

-- 💥 10 minutes plus tard, le DBA voit :
-- "Long running transaction: 10 minutes"
-- "Undo log size: 5GB"

-- ✅ Solution : COMMIT régulièrement ou SET autocommit = 1
COMMIT;  -- Libère les ressources
```

💡 **Bonne pratique production** :
- Applications web : `autocommit = 1` + transactions explicites pour les groupes d'opérations
- Scripts batch : `autocommit = 0` + COMMIT par lots

---

## 6. Gestion d'Erreur dans les Transactions

### 6.1 Détection d'Erreur avec DECLARE HANDLER

```sql
DELIMITER //

CREATE PROCEDURE traiter_commande(IN p_client_id INT, IN p_montant DECIMAL(10,2))
BEGIN
    -- Déclaration du handler d'erreur
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- En cas d'erreur, rollback et signaler
        ROLLBACK;
        SELECT 'Transaction annulée suite à une erreur' AS resultat;
    END;

    START TRANSACTION;

    -- Opérations risquées
    INSERT INTO commandes (client_id, montant) VALUES (p_client_id, p_montant);
    UPDATE comptes SET solde = solde - p_montant WHERE client_id = p_client_id;

    -- Vérification
    SELECT @solde := solde FROM comptes WHERE client_id = p_client_id;
    IF @solde < 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Solde insuffisant';
    END IF;

    COMMIT;
    SELECT 'Transaction validée' AS resultat;
END//

DELIMITER ;

-- Test
CALL traiter_commande(100, 500.00);
```

### 6.2 Gestion d'Erreur Fine

```sql
DELIMITER //

CREATE PROCEDURE traitement_complexe()
BEGIN
    -- Handlers pour différents types d'erreurs
    DECLARE duplicate_error CONDITION FOR 1062;  -- Duplicate entry
    DECLARE foreign_key_error CONDITION FOR 1452;  -- FK violation

    DECLARE CONTINUE HANDLER FOR duplicate_error
    BEGIN
        -- Log l'erreur, mais continue
        INSERT INTO error_log (message) VALUES ('Doublon détecté');
    END;

    DECLARE EXIT HANDLER FOR foreign_key_error
    BEGIN
        -- Erreur critique : rollback et sortie
        ROLLBACK;
        SELECT 'Erreur FK : données incohérentes' AS erreur;
    END;

    START TRANSACTION;

    -- Tentatives d'insertion
    INSERT INTO users (id, name) VALUES (1, 'Alice');
    INSERT INTO users (id, name) VALUES (1, 'Bob');  -- Doublon, log mais continue
    INSERT INTO commandes (user_id) VALUES (999);  -- FK error, rollback et sortie

    COMMIT;
END//

DELIMITER ;
```

### 6.3 Try-Catch Pattern (Simulation)

```sql
DELIMITER //

CREATE PROCEDURE safe_update(IN p_id INT, IN p_value DECIMAL(10,2))
BEGIN
    DECLARE error_occurred BOOLEAN DEFAULT FALSE;

    -- Handler qui capture toutes les erreurs
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET error_occurred = TRUE;
    END;

    START TRANSACTION;

    -- Bloc "try"
    UPDATE produits SET prix = p_value WHERE id = p_id;

    IF NOT error_occurred THEN
        -- Bloc "try" réussi
        COMMIT;
        SELECT 'Mise à jour réussie' AS resultat;
    ELSE
        -- Bloc "catch"
        ROLLBACK;
        SELECT 'Mise à jour échouée' AS resultat;
    END IF;
END//

DELIMITER ;
```

### 6.4 Gestion d'Erreur Applicative (Python)

```python
import pymysql
from contextlib import contextmanager

@contextmanager
def transaction(connection):
    """Context manager pour gérer les transactions automatiquement"""
    cursor = connection.cursor()
    try:
        cursor.execute("START TRANSACTION")
        yield cursor
        cursor.execute("COMMIT")
        print("Transaction validée")
    except Exception as e:
        cursor.execute("ROLLBACK")
        print(f"Transaction annulée : {e}")
        raise
    finally:
        cursor.close()

# Utilisation
conn = pymysql.connect(...)

try:
    with transaction(conn) as cursor:
        # Tout ce bloc est dans une transaction
        cursor.execute(
            "INSERT INTO commandes (client_id, montant) VALUES (%s, %s)",
            (100, 250.00)
        )
        cursor.execute(
            "UPDATE stocks SET quantite = quantite - 1 WHERE produit_id = 42"
        )
        # Si une erreur survient, rollback automatique
        # Si tout réussit, commit automatique

except pymysql.Error as e:
    print(f"Erreur base de données : {e}")
finally:
    conn.close()
```

---

## 7. Bonnes Pratiques en Production

### 7.1 Transactions Courtes

```sql
-- ❌ MAUVAIS : Transaction trop longue
START TRANSACTION;

SELECT * FROM produits WHERE id = 1 FOR UPDATE;  -- Verrou posé

-- Appel API externe (500ms)
-- Calculs complexes applicatifs (2s)
-- Envoi email (1s)

UPDATE produits SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- Verrou gardé pendant 3.5s

-- ✅ BON : Transaction minimale
-- 1. Faire les calculs HORS transaction
SELECT @prix := prix FROM produits WHERE id = 1;
-- Calculs applicatifs ici (sans verrou)
SET @nouveau_prix = @prix * 1.1;

-- 2. Transaction courte
START TRANSACTION;
UPDATE produits SET prix = @nouveau_prix WHERE id = 1;
COMMIT;  -- Verrou gardé <10ms
```

**Règle d'or** : Les transactions doivent durer **millisecondes, pas secondes**.

### 7.2 Ordre Cohérent des Accès

```sql
-- ❌ MAUVAIS : Ordre aléatoire = risque de deadlock
-- Transaction A
START TRANSACTION;
UPDATE comptes SET solde = solde - 100 WHERE id = 5;
UPDATE comptes SET solde = solde + 100 WHERE id = 3;
COMMIT;

-- Transaction B (en parallèle)
START TRANSACTION;
UPDATE comptes SET solde = solde - 50 WHERE id = 3;  -- 💥 Deadlock potentiel
UPDATE comptes SET solde = solde + 50 WHERE id = 5;
COMMIT;

-- ✅ BON : Ordre déterministe (par ID croissant)
-- Transaction A
START TRANSACTION;
UPDATE comptes SET solde = solde + 100 WHERE id = 3;  -- ID le plus petit d'abord
UPDATE comptes SET solde = solde - 100 WHERE id = 5;
COMMIT;

-- Transaction B
START TRANSACTION;
UPDATE comptes SET solde = solde - 50 WHERE id = 3;  -- Même ordre
UPDATE comptes SET solde = solde + 50 WHERE id = 5;
COMMIT;
-- ✅ Pas de deadlock : ordre cohérent
```

### 7.3 Transactions Implicites : À Éviter

```sql
-- ❌ MAUVAIS : Transaction implicite longue
SET autocommit = 0;

UPDATE produits SET stock = stock - 1 WHERE id = 1;
-- Développeur oublie le COMMIT
-- Transaction reste ouverte pendant des heures
-- Verrous gardés, undo log qui grandit

-- ✅ BON : Toujours explicite
SET autocommit = 1;  -- ou laisser par défaut

START TRANSACTION;  -- Début explicite
UPDATE produits SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- Fin explicite
-- Impossible d'oublier
```

### 7.4 Retry Logic pour Deadlocks

```python
def execute_with_retry(func, max_retries=3, backoff=0.1):
    """Retry automatique en cas de deadlock"""
    for attempt in range(max_retries):
        try:
            return func()
        except pymysql.err.OperationalError as e:
            if e.args[0] == 1213:  # Deadlock code
                if attempt < max_retries - 1:
                    time.sleep(backoff * (2 ** attempt))  # Backoff exponentiel
                    continue
                else:
                    raise  # Max retries atteint
            else:
                raise  # Autre erreur

# Utilisation
def transfert_argent(de_compte, vers_compte, montant):
    conn = pymysql.connect(...)
    try:
        with conn.cursor() as cursor:
            cursor.execute("START TRANSACTION")

            # Ordre cohérent pour éviter deadlock
            comptes = sorted([de_compte, vers_compte])
            for compte in comptes:
                cursor.execute(
                    "SELECT id FROM comptes WHERE id = %s FOR UPDATE",
                    (compte,)
                )

            cursor.execute(
                "UPDATE comptes SET solde = solde - %s WHERE id = %s",
                (montant, de_compte)
            )
            cursor.execute(
                "UPDATE comptes SET solde = solde + %s WHERE id = %s",
                (montant, vers_compte)
            )

            cursor.execute("COMMIT")
    except Exception:
        cursor.execute("ROLLBACK")
        raise
    finally:
        conn.close()

# Appel avec retry automatique
execute_with_retry(lambda: transfert_argent(1, 2, 100.00))
```

### 7.5 Logging et Monitoring

```sql
-- Créer une table de log transactionnel
CREATE TABLE transaction_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    session_id VARCHAR(64),
    transaction_type VARCHAR(50),
    start_time DATETIME(3),
    end_time DATETIME(3),
    duration_ms INT,
    status ENUM('COMMIT', 'ROLLBACK', 'ERROR'),
    error_message TEXT,
    INDEX idx_session (session_id),
    INDEX idx_status (status),
    INDEX idx_duration (duration_ms)
) ENGINE=InnoDB;

-- Procédure avec logging
DELIMITER //

CREATE PROCEDURE transfert_avec_log(
    IN p_de_compte INT,
    IN p_vers_compte INT,
    IN p_montant DECIMAL(10,2)
)
BEGIN
    DECLARE v_session_id VARCHAR(64);
    DECLARE v_start_time DATETIME(3);
    DECLARE v_log_id BIGINT;
    DECLARE v_error_msg TEXT DEFAULT NULL;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Log l'erreur
        GET DIAGNOSTICS CONDITION 1 v_error_msg = MESSAGE_TEXT;

        UPDATE transaction_log
        SET end_time = NOW(3),
            duration_ms = TIMESTAMPDIFF(MICROSECOND, start_time, NOW(3)) / 1000,
            status = 'ERROR',
            error_message = v_error_msg
        WHERE id = v_log_id;

        ROLLBACK;
    END;

    -- Initialisation
    SET v_session_id = CONNECTION_ID();
    SET v_start_time = NOW(3);

    -- Log début
    INSERT INTO transaction_log (session_id, transaction_type, start_time, status)
    VALUES (v_session_id, 'TRANSFERT', v_start_time, 'IN_PROGRESS');
    SET v_log_id = LAST_INSERT_ID();

    START TRANSACTION;

    -- Opérations
    UPDATE comptes SET solde = solde - p_montant WHERE id = p_de_compte;
    UPDATE comptes SET solde = solde + p_montant WHERE id = p_vers_compte;

    COMMIT;

    -- Log succès
    UPDATE transaction_log
    SET end_time = NOW(3),
        duration_ms = TIMESTAMPDIFF(MICROSECOND, start_time, NOW(3)) / 1000,
        status = 'COMMIT'
    WHERE id = v_log_id;
END//

DELIMITER ;

-- Analyse des transactions
-- Transactions les plus lentes
SELECT
    transaction_type,
    AVG(duration_ms) AS avg_ms,
    MAX(duration_ms) AS max_ms,
    COUNT(*) AS nb
FROM transaction_log
WHERE start_time >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY transaction_type
ORDER BY avg_ms DESC;

-- Taux d'erreur
SELECT
    status,
    COUNT(*) AS nb,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS pct
FROM transaction_log
WHERE start_time >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY status;
```

---

## 8. Optimisations Avancées

### 8.1 Batch Processing avec Transactions

```sql
-- ❌ MAUVAIS : Une transaction par ligne
SET autocommit = 1;
FOR i IN 1..10000 LOOP
    UPDATE produits SET prix = prix * 1.1 WHERE id = i;
    -- 10000 commits = très lent
END LOOP;

-- ✅ BON : Transactions par lots de 1000
SET autocommit = 0;
FOR i IN 1..10000 LOOP
    UPDATE produits SET prix = prix * 1.1 WHERE id = i;

    IF MOD(i, 1000) = 0 THEN
        COMMIT;  -- Commit tous les 1000
        START TRANSACTION;
    END IF;
END LOOP;
COMMIT;  -- Dernier lot
```

**Benchmark** :
- 10000 transactions : ~50 secondes
- 10 transactions de 1000 : ~5 secondes (10x plus rapide)
- 1 transaction de 10000 : ~3 secondes (mais risque de timeout)

💡 **Compromis optimal** : Lots de 500-1000 lignes

### 8.2 Préparation Hors Transaction

```sql
-- ❌ MAUVAIS : Calculs dans la transaction
START TRANSACTION;

-- Requête complexe pour calculer le nouveau prix
SELECT
    @nouveau_prix :=
    (SELECT AVG(prix) FROM produits WHERE categorie = 'X') * 1.1
FROM DUAL;

UPDATE produits SET prix = @nouveau_prix WHERE id = 1;

COMMIT;

-- ✅ BON : Calcul avant la transaction
-- 1. Calculs complexes (sans verrou)
SELECT
    @nouveau_prix :=
    (SELECT AVG(prix) FROM produits WHERE categorie = 'X') * 1.1
FROM DUAL;

-- 2. Transaction courte
START TRANSACTION;
UPDATE produits SET prix = @nouveau_prix WHERE id = 1;
COMMIT;
```

### 8.3 Utiliser FOR UPDATE Judicieusement

```sql
-- ❌ MAUVAIS : FOR UPDATE trop large
START TRANSACTION;
SELECT * FROM produits FOR UPDATE;  -- Verrouille toute la table !
-- Traitement...
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- ✅ BON : FOR UPDATE ciblé
START TRANSACTION;
SELECT * FROM produits WHERE id = 42 FOR UPDATE;  -- Verrouille 1 ligne
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- ✅ MEILLEUR : UPDATE direct (pas de SELECT)
START TRANSACTION;
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;
-- InnoDB pose automatiquement le verrou exclusif
```

---

## ✅ Points clés à retenir

- **START TRANSACTION** : Toujours préférer la forme explicite vs autocommit implicite
- **WITH CONSISTENT SNAPSHOT** : Essentiel pour les rapports cohérents et les backups
- **READ ONLY** : Optimisation performance pour les transactions de lecture seule
- **COMMIT** : Attention aux commits implicites (DDL, TRUNCATE, LOCK TABLES)
- **ROLLBACK** : Annulation complète, utiliser SAVEPOINT pour rollback partiel
- **SAVEPOINT** : Permet des rollbacks granulaires, mais consomme de la mémoire
- **Autocommit** : Garder à 1 en production web, utiliser transactions explicites
- **Transactions courtes** : Règle d'or = millisecondes, pas secondes
- **Ordre cohérent** : Accéder aux ressources dans le même ordre = éviter deadlocks
- **Retry logic** : Implémenter des retries automatiques pour les deadlocks

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 START TRANSACTION](https://mariadb.com/kb/en/start-transaction/)
- [📖 COMMIT](https://mariadb.com/kb/en/commit/)
- [📖 ROLLBACK](https://mariadb.com/kb/en/rollback/)
- [📖 SAVEPOINT](https://mariadb.com/kb/en/savepoint/)
- [📖 Autocommit](https://mariadb.com/kb/en/server-system-variables/#autocommit)
- [📖 Transaction Isolation Levels](https://mariadb.com/kb/en/set-transaction-isolation-level/)

### Bonnes pratiques
- [MySQL Transaction Best Practices](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [Percona: MySQL Transaction Management](https://www.percona.com/blog/)

---

## ➡️ Section suivante

**6.3 Niveaux d'isolation (READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE)** : Exploration détaillée de chaque niveau, phénomènes de concurrence et cas d'usage.

---


⏭️ [Niveaux d'isolation](/06-transactions-et-concurrence/03-niveaux-isolation.md)

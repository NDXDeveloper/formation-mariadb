🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 Verrous (LOCK TABLES, SELECT FOR UPDATE, SELECT FOR SHARE)

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures

> **Prérequis** :
> - Section 6.1 (Propriétés ACID)
> - Section 6.3 (Niveaux d'isolation)
> - Compréhension du MVCC

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre les différents types de verrous dans InnoDB
- Maîtriser LOCK TABLES et ses alternatives
- Utiliser SELECT FOR UPDATE et SELECT FOR SHARE efficacement
- Comprendre les gap locks et next-key locks
- Analyser les problèmes de verrouillage avec information_schema
- Optimiser les stratégies de verrouillage en production
- Éviter les pièges courants et les anti-patterns

---

## Introduction

Les **verrous** (locks) sont les mécanismes par lesquels MariaDB/InnoDB contrôle l'accès concurrent aux données. Ils sont essentiels pour garantir l'isolation et la cohérence, mais peuvent aussi être source de problèmes de performance et de blocages.

### Pourquoi les Verrous sont Nécessaires

Sans verrous, le chaos règne :

```sql
-- Sans verrous : PROBLÈME
-- Transaction A et B lisent simultanément stock = 10
-- Transaction A : UPDATE stock = 9 (vend 1 unité)
-- Transaction B : UPDATE stock = 9 (vend 1 unité)
-- 💥 Résultat : stock = 9 alors qu'on a vendu 2 unités !

-- Avec verrous : SOLUTION
-- Transaction A pose un verrou exclusif
-- Transaction B doit attendre
-- Stock final correct : 8
```

### Le Spectre des Verrous InnoDB

InnoDB offre une granularité de verrouillage très fine :

```
┌────────────────────────────────────────────────┐
│ Granularité des Verrous                        │
├────────────────────────────────────────────────┤
│ TABLE (toute la table)    ← Moins granulaire   │
│    ↓                          Plus de blocage  │
│ PAGE (16KB)                                    │
│    ↓                                           │
│ ROW (une ligne)           ← Plus granulaire    │
│                              Moins de blocage  │
└────────────────────────────────────────────────┘
```

InnoDB utilise principalement des **row-level locks** (verrous de ligne), ce qui permet une concurrence maximale.

---

## 1. Types de Verrous InnoDB

### 1.1 Verrous Partagés et Exclusifs

InnoDB implémente deux types de verrous fondamentaux :

#### Shared Lock (S) - Verrou Partagé

**Principe** : "Je lis, d'autres peuvent lire aussi, mais personne ne peut écrire"

```sql
-- Pose un shared lock
SELECT * FROM produits WHERE id = 1 FOR SHARE;
-- ou (ancienne syntaxe)
SELECT * FROM produits WHERE id = 1 LOCK IN SHARE MODE;

-- Plusieurs transactions peuvent poser des S-locks simultanément
-- Toutes peuvent lire
-- Aucune ne peut écrire
```

**Visualisation** :

```
Transaction A : SELECT ... FOR SHARE  [S-lock sur ligne 1]
Transaction B : SELECT ... FOR SHARE  [S-lock sur ligne 1] ✅ OK
Transaction C : UPDATE ligne 1        ⏳ BLOQUÉE (attend A et B)
```

#### Exclusive Lock (X) - Verrou Exclusif

**Principe** : "J'écris, personne d'autre ne peut ni lire ni écrire"

```sql
-- Pose un exclusive lock
SELECT * FROM produits WHERE id = 1 FOR UPDATE;

-- Ou implicitement lors d'un UPDATE/DELETE
UPDATE produits SET stock = stock - 1 WHERE id = 1;
-- Pose automatiquement un X-lock
```

**Visualisation** :

```
Transaction A : SELECT ... FOR UPDATE [X-lock sur ligne 1]
Transaction B : SELECT ... FOR SHARE  ⏳ BLOQUÉE
Transaction C : SELECT ... FOR UPDATE ⏳ BLOQUÉE
Transaction D : UPDATE ligne 1        ⏳ BLOQUÉE
```

### 1.2 Matrice de Compatibilité

| Verrou demandé ↓ \ Verrou existant → | Shared (S) | Exclusive (X) |
|---------------------------------------|------------|---------------|
| **Shared (S)**                        | ✅ Compatible | ⏳ Bloqué |
| **Exclusive (X)**                     | ⏳ Bloqué | ⏳ Bloqué |

**Règles simples** :
- ✅ Plusieurs S-locks peuvent coexister
- ⏳ Un X-lock bloque tout (S et X)
- ⏳ Un S-lock bloque les X-locks

### 1.3 Intention Locks (Verrous d'Intention)

Pour optimiser les verrous de table, InnoDB utilise des **intention locks** qui signalent l'intention de poser des verrous de ligne.

**Types** :
- **IS** (Intention Shared) : "Je veux poser des S-locks sur certaines lignes"
- **IX** (Intention Exclusive) : "Je veux poser des X-locks sur certaines lignes"

```sql
-- Quand vous faites :
SELECT * FROM produits WHERE id = 1 FOR UPDATE;

-- InnoDB fait en interne :
-- 1. Pose un IX-lock sur la TABLE produits
-- 2. Pose un X-lock sur la LIGNE id=1
```

**Pourquoi ?** Les intention locks permettent à InnoDB de vérifier rapidement si un `LOCK TABLES` serait compatible sans scanner toutes les lignes.

**Matrice complète** :

|          | IS | IX | S | X |
|----------|----|----|---|---|
| **IS**   | ✅ | ✅ | ✅ | ⏳ |
| **IX**   | ✅ | ✅ | ⏳ | ⏳ |
| **S**    | ✅ | ⏳ | ✅ | ⏳ |
| **X**    | ⏳ | ⏳ | ⏳ | ⏳ |

---

## 2. LOCK TABLES : L'Outil à Double Tranchant

### 2.1 Syntaxe et Comportement

```sql
-- Verrouiller une table en lecture
LOCK TABLES produits READ;
-- ✅ Toutes les sessions peuvent lire
-- ⏳ Personne ne peut écrire (incluant la session actuelle)

-- Verrouiller une table en écriture
LOCK TABLES produits WRITE;
-- ⏳ Personne d'autre ne peut lire ni écrire
-- ✅ Seule la session actuelle peut lire et écrire

-- Libérer tous les verrous de table
UNLOCK TABLES;
```

### 2.2 Exemple Concret

```sql
-- Session 1
LOCK TABLES produits WRITE;

-- Session 1 peut maintenant modifier
UPDATE produits SET prix = prix * 1.1;

-- Session 2 tente de lire
SELECT * FROM produits WHERE id = 1;
-- ⏳ BLOQUÉE jusqu'au UNLOCK TABLES de Session 1

-- Session 1 libère
UNLOCK TABLES;

-- Session 2 peut maintenant lire
```

### 2.3 Verrouillage Multiple

```sql
-- Verrouiller plusieurs tables simultanément
LOCK TABLES
    produits WRITE,
    categories READ,
    commandes WRITE;

-- ⚠️ Limitation importante :
-- Une fois les tables verrouillées, vous ne pouvez accéder
-- QU'aux tables verrouillées dans la liste

-- ❌ Ceci échouera :
SELECT * FROM clients;  -- ERROR : Table 'clients' was not locked

UNLOCK TABLES;
```

### 2.4 LOCK TABLES vs Transactions

```sql
-- ⚠️ PIÈGE : LOCK TABLES cause un COMMIT implicite !

START TRANSACTION;
INSERT INTO logs (message) VALUES ('début');

LOCK TABLES produits WRITE;
-- 💥 COMMIT implicite ici !
-- L'INSERT est maintenant permanent

-- Si erreur maintenant :
ROLLBACK;
-- Ne rollback PAS l'INSERT (déjà commité)

UNLOCK TABLES;
```

### 2.5 Pourquoi ÉVITER LOCK TABLES

**Problèmes** :

1. **Blocage massif** : Bloque toute la table, pas juste les lignes nécessaires
2. **Commit implicite** : Casse les transactions
3. **Complexité** : Doit lister toutes les tables utilisées
4. **Deadlock facile** : Si ordre différent entre sessions
5. **Anti-pattern InnoDB** : InnoDB est conçu pour les row-level locks

```sql
-- ❌ MAUVAIS : LOCK TABLES
LOCK TABLES produits WRITE;
UPDATE produits SET stock = stock - 1 WHERE id = 42;
UNLOCK TABLES;
-- Bloque tous les accès à la table entière !

-- ✅ BON : SELECT FOR UPDATE
START TRANSACTION;
SELECT * FROM produits WHERE id = 42 FOR UPDATE;
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;
-- Bloque seulement la ligne 42
```

### 2.6 Cas d'Usage Légitimes de LOCK TABLES

**Rares cas où LOCK TABLES est approprié** :

```sql
-- 1. Maintenance hors production
LOCK TABLES produits WRITE;
-- Reconstruction d'index, cleanup de données, etc.
UNLOCK TABLES;

-- 2. Import/Export massif
LOCK TABLES commandes WRITE;
LOAD DATA INFILE '/tmp/commandes.csv' INTO TABLE commandes;
UNLOCK TABLES;

-- 3. MyISAM (moteur sans transactions)
-- LOCK TABLES est le seul moyen de synchroniser
```

💡 **Règle générale** : Avec InnoDB, n'utilisez jamais `LOCK TABLES`. Utilisez les transactions avec `FOR UPDATE`/`FOR SHARE`.

---

## 3. SELECT FOR UPDATE : Verrou Exclusif

### 3.1 Syntaxe et Comportement

```sql
START TRANSACTION;

-- Pose un X-lock sur les lignes matchées
SELECT * FROM produits
WHERE id = 1
FOR UPDATE;

-- Personne d'autre ne peut :
-- - Lire avec FOR UPDATE
-- - Lire avec FOR SHARE
-- - Faire UPDATE
-- - Faire DELETE

-- Mais peuvent toujours :
-- - Lire normalement (SELECT sans FOR)

COMMIT;  -- Libère le verrou
```

### 3.2 Cas d'Usage : Éviter les Race Conditions

**Problème sans FOR UPDATE** :

```sql
-- ❌ MAUVAIS : Race condition
START TRANSACTION;

-- Lit le stock
SELECT @stock := stock FROM produits WHERE id = 42;
-- stock = 5

-- Entre-temps, une autre transaction modifie stock à 0

-- On décrémente
IF @stock > 0 THEN
    UPDATE produits SET stock = @stock - 1 WHERE id = 42;
    -- 💥 Stock devient -1 !
END IF;

COMMIT;
```

**Solution avec FOR UPDATE** :

```sql
-- ✅ BON : Verrou exclusif
START TRANSACTION;

-- Lit ET verrouille
SELECT @stock := stock FROM produits WHERE id = 42 FOR UPDATE;
-- 🔒 Personne ne peut modifier tant qu'on n'a pas commité

-- Décrémentation sécurisée
IF @stock > 0 THEN
    UPDATE produits SET stock = @stock - 1 WHERE id = 42;
    -- ✅ Stock correct garanti
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

### 3.3 FOR UPDATE NOWAIT

**Éviter les attentes infinies** :

```sql
-- Syntaxe standard : attend jusqu'à innodb_lock_wait_timeout
SELECT * FROM produits WHERE id = 1 FOR UPDATE;

-- NOWAIT : échoue immédiatement si verrou indisponible
SELECT * FROM produits WHERE id = 1 FOR UPDATE NOWAIT;
-- Si déjà verrouillé : ERROR 1205 (HY000): Lock wait timeout
```

🆕 **MariaDB 10.3+** : Support de `NOWAIT` et `SKIP LOCKED`

### 3.4 FOR UPDATE SKIP LOCKED

**Ignorer les lignes verrouillées** :

```sql
-- Scénario : Queue de tâches
CREATE TABLE tasks (
    id INT PRIMARY KEY AUTO_INCREMENT,
    status ENUM('pending', 'processing', 'done'),
    worker_id INT,
    data TEXT
);

-- Worker 1 prend une tâche
START TRANSACTION;
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Prend la première tâche non verrouillée

UPDATE tasks SET status = 'processing', worker_id = 1 WHERE id = ...;
COMMIT;

-- Worker 2 (simultané) prend une autre tâche
-- SKIP LOCKED garantit qu'il ne prend pas la même
```

**Cas d'usage** :
- ✅ File d'attente de jobs
- ✅ Distribution de tâches entre workers
- ✅ Traitement parallèle sans collision

### 3.5 FOR UPDATE OF (PostgreSQL-style, pas MariaDB)

⚠️ **Note** : `FOR UPDATE OF table_name` n'existe pas dans MariaDB/MySQL. Tous les FOR UPDATE verrouillent toutes les tables de la requête.

```sql
-- ❌ Pas supporté dans MariaDB
SELECT * FROM produits p
JOIN categories c ON p.categorie_id = c.id
FOR UPDATE OF p;  -- ERROR

-- ✅ MariaDB : verrouille produits ET categories
SELECT * FROM produits p
JOIN categories c ON p.categorie_id = c.id
FOR UPDATE;  -- Verrouille les deux tables
```

---

## 4. SELECT FOR SHARE : Verrou Partagé

### 4.1 Syntaxe et Comportement

```sql
START TRANSACTION;

-- Pose un S-lock
SELECT * FROM produits WHERE id = 1 FOR SHARE;
-- ou ancienne syntaxe :
SELECT * FROM produits WHERE id = 1 LOCK IN SHARE MODE;

-- Permet :
-- ✅ D'autres SELECT FOR SHARE
-- ✅ SELECT normaux

-- Bloque :
-- ⏳ SELECT FOR UPDATE
-- ⏳ UPDATE
-- ⏳ DELETE

COMMIT;
```

### 4.2 Cas d'Usage : Lecture Cohérente

```sql
-- Scénario : Calculer un rapport multi-tables
START TRANSACTION;

-- Lire et verrouiller les données pour cohérence
SELECT * FROM commandes WHERE date = CURDATE() FOR SHARE;
SELECT * FROM commande_items WHERE commande_id IN (...) FOR SHARE;

-- Calculs complexes ici
-- Garantit que personne ne modifie pendant les calculs

-- D'autres peuvent lire simultanément (FOR SHARE)
-- Mais personne ne peut modifier

COMMIT;
```

### 4.3 FOR SHARE vs SELECT Normal

```sql
-- SELECT normal : consistent read (MVCC)
SELECT * FROM produits WHERE id = 1;
-- ✅ Voit un snapshot
-- ✅ N'empêche aucune écriture

-- SELECT FOR SHARE : posed lock
SELECT * FROM produits WHERE id = 1 FOR SHARE;
-- 🔒 Pose un S-lock
-- ⏳ Empêche les écritures
```

**Quand utiliser FOR SHARE** :

```sql
-- ✅ Besoin de garantir qu'une donnée ne change pas
-- pendant qu'on fait d'autres opérations

START TRANSACTION;

-- Réserver un produit sans le modifier encore
SELECT @prix := prix FROM produits WHERE id = 42 FOR SHARE;

-- Vérifications métier complexes
-- ...

-- Maintenant on modifie
UPDATE produits SET stock = stock - 1 WHERE id = 42;

COMMIT;
```

### 4.4 FOR SHARE NOWAIT / SKIP LOCKED

```sql
-- NOWAIT : échoue si verrou indisponible
SELECT * FROM produits WHERE id = 1 FOR SHARE NOWAIT;

-- SKIP LOCKED : saute les lignes verrouillées
SELECT * FROM produits WHERE categorie = 'Livres'
FOR SHARE SKIP LOCKED;
```

---

## 5. Gap Locks et Next-Key Locks

### 5.1 Qu'est-ce qu'un Gap Lock ?

Un **gap lock** verrouille l'**espace entre** les valeurs d'index, pas les valeurs elles-mêmes.

**Exemple** :

```sql
-- Table avec IDs : 10, 20, 30, 40, 50

START TRANSACTION;

SELECT * FROM produits
WHERE id BETWEEN 25 AND 35
FOR UPDATE;

-- InnoDB pose des verrous :
-- 1. Record lock sur id=30 (seule ligne matchée)
-- 2. Gap lock sur (20, 30) - empêche INSERT id=25
-- 3. Gap lock sur (30, 40) - empêche INSERT id=35
```

**Visualisation** :

```
Index: ... 20 [gap] 30 [gap] 40 ...
               ↑🔒   ↑🔒   ↑🔒
          Gap lock  Record  Gap lock
                    lock
```

### 5.2 Pourquoi les Gap Locks ?

**Sans gap locks : Phantom read**

```sql
-- Transaction A
START TRANSACTION;
SELECT COUNT(*) FROM produits WHERE prix BETWEEN 10 AND 50;
-- Compte : 5 produits

-- Transaction B insère
INSERT INTO produits (nom, prix) VALUES ('Widget', 30);
COMMIT;

-- Transaction A recompte
SELECT COUNT(*) FROM produits WHERE prix BETWEEN 10 AND 50;
-- Compte : 6 produits ❌ Phantom !
```

**Avec gap locks** (REPEATABLE READ) :

```sql
-- Transaction A
START TRANSACTION;
SELECT COUNT(*) FROM produits WHERE prix BETWEEN 10 AND 50 FOR UPDATE;
-- 🔒 Gap locks posés sur les intervalles

-- Transaction B tente d'insérer
INSERT INTO produits (nom, prix) VALUES ('Widget', 30);
-- ⏳ BLOQUÉ par le gap lock
-- Attend que A commitée

-- Transaction A recompte
SELECT COUNT(*) FROM produits WHERE prix BETWEEN 10 AND 50;
-- Compte : 5 produits ✅ Cohérent
COMMIT;

-- Maintenant B peut insérer
```

### 5.3 Next-Key Locks

Un **next-key lock** = **record lock** + **gap lock**

```sql
-- InnoDB combine les deux types
SELECT * FROM produits WHERE id >= 20 FOR UPDATE;

-- Next-key locks :
-- - Record lock sur 20, 30, 40, 50, ...
-- - Gap lock sur (10, 20), (20, 30), (30, 40), ...
```

**Avantage** : Empêche les phantom reads (spécificité InnoDB REPEATABLE READ)

### 5.4 Coût des Gap Locks

**Problème** : Les gap locks peuvent bloquer plus que nécessaire

```sql
-- Table avec grandes gaps
-- IDs : 1, 100, 200, 300, 400, 500

START TRANSACTION;
SELECT * FROM produits WHERE id = 150 FOR UPDATE;
-- Pas de ligne matchée, mais...
-- Gap lock sur (100, 200) - LARGE plage !

-- Transaction B
INSERT INTO produits (id, nom) VALUES (120, 'Test');
-- ⏳ BLOQUÉ même si 120 ≠ 150
```

**Solution** : Utiliser des ID séquentiels (AUTO_INCREMENT) pour minimiser les gaps.

### 5.5 Désactiver les Gap Locks

```sql
-- En READ COMMITTED, pas de gap locks
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
SELECT * FROM produits WHERE id BETWEEN 25 AND 35 FOR UPDATE;
-- Record locks uniquement, pas de gap locks
-- ⚠️ Permet les phantom reads
COMMIT;
```

---

## 6. Patterns de Verrouillage

### 6.1 Pattern : Verrouiller dans un Ordre Cohérent

**Problème : Deadlock par ordre inversé**

```sql
-- Transaction A
START TRANSACTION;
UPDATE comptes SET solde = solde - 100 WHERE id = 5;  -- 🔒 Lock 5
UPDATE comptes SET solde = solde + 100 WHERE id = 3;  -- Attend lock 3

-- Transaction B (parallèle)
START TRANSACTION;
UPDATE comptes SET solde = solde - 50 WHERE id = 3;   -- 🔒 Lock 3
UPDATE comptes SET solde = solde + 50 WHERE id = 5;   -- Attend lock 5

-- 💥 DEADLOCK : Chacun attend l'autre
```

**Solution : Ordre déterministe**

```sql
-- ✅ BON : Toujours verrouiller par ID croissant
START TRANSACTION;

-- Trier les IDs avant de verrouiller
SET @min_id = LEAST(5, 3);  -- 3
SET @max_id = GREATEST(5, 3);  -- 5

-- Verrouiller dans l'ordre
SELECT * FROM comptes WHERE id = @min_id FOR UPDATE;  -- 3 d'abord
SELECT * FROM comptes WHERE id = @max_id FOR UPDATE;  -- 5 ensuite

-- Maintenant on peut modifier
UPDATE comptes SET solde = solde + 100 WHERE id = 3;
UPDATE comptes SET solde = solde - 100 WHERE id = 5;

COMMIT;
```

### 6.2 Pattern : Verrouillage Optimiste

**Approche** : Lire sans verrou, vérifier au moment de l'écriture

```sql
-- Table avec version
CREATE TABLE produits (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    version INT DEFAULT 0  -- Compteur de version
);

-- Lire sans verrou
SELECT @version := version, @prix := prix
FROM produits
WHERE id = 1;

-- Calculs applicatifs...

-- Écriture avec vérification de version
START TRANSACTION;

UPDATE produits
SET prix = 99.99, version = version + 1
WHERE id = 1 AND version = @version;

IF ROW_COUNT() = 0 THEN
    -- Quelqu'un a modifié entre-temps
    ROLLBACK;
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Conflit de version';
ELSE
    COMMIT;
END IF;
```

**Avantages** :
- ✅ Pas de verrous gardés longtemps
- ✅ Meilleure concurrence
- ✅ Détecte les conflits

**Inconvénients** :
- ⚠️ Possible retry nécessaire
- ⚠️ Plus complexe à implémenter

### 6.3 Pattern : Verrouillage Pessimiste

**Approche** : Verrouiller immédiatement, garantir le succès

```sql
-- ✅ Verrouillage pessimiste
START TRANSACTION;

-- Verrouiller dès la lecture
SELECT @stock := stock
FROM produits
WHERE id = 42
FOR UPDATE;

-- Personne ne peut modifier pendant nos calculs

IF @stock > 0 THEN
    UPDATE produits SET stock = @stock - 1 WHERE id = 42;
    INSERT INTO commandes (...) VALUES (...);
    COMMIT;  -- Succès garanti
ELSE
    ROLLBACK;
END IF;
```

**Avantages** :
- ✅ Pas de retry
- ✅ Logique simple
- ✅ Succès garanti si commit

**Inconvénients** :
- ⚠️ Verrous gardés plus longtemps
- ⚠️ Moins de concurrence
- ⚠️ Risque de deadlock

### 6.4 Pattern : Batch Locking

```sql
-- ❌ MAUVAIS : Verrouiller ligne par ligne
FOR i IN 1..1000 LOOP
    START TRANSACTION;
    SELECT * FROM produits WHERE id = i FOR UPDATE;
    UPDATE produits SET prix = prix * 1.1 WHERE id = i;
    COMMIT;
END LOOP;
-- 1000 transactions, lent

-- ✅ BON : Verrouiller par lots
START TRANSACTION;
FOR i IN 1..1000 LOOP
    IF MOD(i, 100) = 0 THEN
        COMMIT;
        START TRANSACTION;
    END IF;

    SELECT * FROM produits WHERE id = i FOR UPDATE;
    UPDATE produits SET prix = prix * 1.1 WHERE id = i;
END LOOP;
COMMIT;
-- 10 transactions, rapide
```

---

## 7. Monitoring et Diagnostic

### 7.1 Voir les Verrous Actifs

```sql
-- Vue des transactions en cours
SELECT
    trx_id,
    trx_state,
    trx_started,
    trx_requested_lock_id,
    trx_wait_started,
    trx_weight,  -- Nombre de lignes verrouillées/modifiées
    trx_mysql_thread_id,
    trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;

-- Vue des verrous posés
SELECT
    lock_id,
    lock_trx_id,
    lock_mode,  -- X, S, IX, IS
    lock_type,  -- RECORD, TABLE
    lock_table,
    lock_index,
    lock_data
FROM information_schema.INNODB_LOCKS;

-- Vue des attentes de verrous
SELECT
    requesting_trx_id,
    requested_lock_id,
    blocking_trx_id,
    blocking_lock_id
FROM information_schema.INNODB_LOCK_WAITS;
```

### 7.2 Identifier les Blocages

```sql
-- Qui bloque qui ?
SELECT
    CONCAT('Thread ', b.trx_mysql_thread_id, ' blocks thread ', r.trx_mysql_thread_id) AS blocking_info,
    r.trx_id AS waiting_trx,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx,
    b.trx_query AS blocking_query,
    TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_seconds
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
ORDER BY wait_seconds DESC;
```

### 7.3 Analyser les Verrous avec SHOW ENGINE INNODB STATUS

```sql
SHOW ENGINE INNODB STATUS\G

-- Chercher les sections importantes :
-- 1. LATEST DETECTED DEADLOCK
--    Détails du dernier deadlock

-- 2. TRANSACTIONS
--    Liste des transactions actives et leurs verrous

-- 3. LOCK WAIT
--    Informations sur les attentes de verrous
```

**Exemple de sortie** :

```
---TRANSACTION 12345, ACTIVE 5 sec
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 10, OS thread handle 140234, query id 100 localhost root updating
UPDATE produits SET stock = stock - 1 WHERE id = 42
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 58 page no 4 n bits 80 index idx_category
```

### 7.4 Tuer une Transaction Bloquante

```sql
-- Identifier le thread bloquant
SELECT
    b.trx_mysql_thread_id AS blocking_thread
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id;

-- Tuer le thread
KILL 12345;  -- Remplacer par le thread_id
```

⚠️ **Attention** : `KILL` termine la connexion, provoque un ROLLBACK de la transaction.

---

## 8. Configuration et Tuning

### 8.1 Variables Importantes

```sql
-- Timeout d'attente de verrou (en secondes)
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
-- Default : 50 secondes

SET GLOBAL innodb_lock_wait_timeout = 10;
-- Réduire pour fail-fast en production

-- Détection de deadlock
SHOW VARIABLES LIKE 'innodb_deadlock_detect';
-- Default : ON (recommandé)

-- Log tous les deadlocks dans error log
SET GLOBAL innodb_print_all_deadlocks = ON;
```

### 8.2 Configuration Production

```ini
[mysqld]
# Timeout verrou : fail-fast
innodb_lock_wait_timeout = 10

# Toujours détecter les deadlocks
innodb_deadlock_detect = ON

# Logger tous les deadlocks
innodb_print_all_deadlocks = 1

# Taille du buffer pour les verrous
# (augmenter si beaucoup de verrous simultanés)
innodb_buffer_pool_size = 8G

# Isolation level
transaction_isolation = REPEATABLE-READ
```

### 8.3 Monitoring Continu

```sql
-- Créer une vue de monitoring
CREATE OR REPLACE VIEW v_lock_monitoring AS
SELECT
    w.requesting_trx_id,
    w.blocking_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    r.trx_started AS waiting_started,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    b.trx_started AS blocking_started,
    TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_seconds
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id;

-- Requête d'alerte
SELECT * FROM v_lock_monitoring WHERE wait_seconds > 30;
```

---

## 9. Bonnes Pratiques

### 9.1 ✅ À FAIRE

```sql
-- 1. Utiliser FOR UPDATE pour les lectures critiques
START TRANSACTION;
SELECT * FROM produits WHERE id = 42 FOR UPDATE;
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- 2. Transactions courtes
-- Faire les calculs AVANT la transaction
SET @nouveau_prix = (SELECT AVG(prix) FROM produits) * 1.1;
START TRANSACTION;
UPDATE produits SET prix = @nouveau_prix WHERE id = 1;
COMMIT;

-- 3. Ordre cohérent des verrous
-- Toujours par ID croissant
START TRANSACTION;
SELECT * FROM comptes WHERE id IN (3, 5) ORDER BY id FOR UPDATE;
-- ...
COMMIT;

-- 4. Utiliser NOWAIT pour fail-fast
SELECT * FROM produits WHERE id = 1 FOR UPDATE NOWAIT;
```

### 9.2 ❌ À ÉVITER

```sql
-- 1. LOCK TABLES avec InnoDB
LOCK TABLES produits WRITE;  -- ❌ Bloque tout

-- 2. Transactions longues avec verrous
START TRANSACTION;
SELECT * FROM produits FOR UPDATE;  -- 🔒 Toute la table
-- Appel API (2 secondes)
-- Calculs (5 secondes)
UPDATE ...;
COMMIT;  -- ❌ Verrous gardés 7 secondes

-- 3. SELECT FOR UPDATE sans WHERE
SELECT * FROM produits FOR UPDATE;  -- ❌ Verrouille TOUT

-- 4. Ignorer les deadlocks
try {
    update();
} catch (DeadlockException e) {
    // ❌ Ne rien faire
}
-- ✅ Devrait retry
```

### 9.3 Checklist Pré-Production

- [ ] Transactions < 100ms en moyenne
- [ ] Pas de `LOCK TABLES` dans le code
- [ ] Ordre cohérent des verrous documenté
- [ ] Retry logic pour deadlocks implémenté
- [ ] Monitoring des lock waits en place
- [ ] `innodb_lock_wait_timeout` ajusté (10-30s)
- [ ] `innodb_print_all_deadlocks = ON`
- [ ] Tests de charge avec concurrence simulée

---

## 10. Cas d'Usage Production

### 10.1 E-commerce : Réservation de Stock

```sql
-- Pattern classique : Check-and-Set avec verrou
DELIMITER //

CREATE PROCEDURE reserver_produit(
    IN p_produit_id INT,
    IN p_quantite INT,
    IN p_client_id INT,
    OUT p_success BOOLEAN
)
BEGIN
    DECLARE v_stock INT;
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
    END;

    START TRANSACTION;

    -- Verrouiller et lire le stock
    SELECT stock INTO v_stock
    FROM produits
    WHERE id = p_produit_id
    FOR UPDATE;

    -- Vérifier disponibilité
    IF v_stock >= p_quantite THEN
        -- Décrémenter le stock
        UPDATE produits
        SET stock = stock - p_quantite
        WHERE id = p_produit_id;

        -- Créer la réservation
        INSERT INTO reservations (produit_id, client_id, quantite)
        VALUES (p_produit_id, p_client_id, p_quantite);

        COMMIT;
        SET p_success = TRUE;
    ELSE
        ROLLBACK;
        SET p_success = FALSE;
    END IF;
END//

DELIMITER ;

-- Utilisation
CALL reserver_produit(42, 2, 100, @success);
SELECT @success;  -- TRUE ou FALSE
```

### 10.2 Banking : Transfert Atomique

```sql
DELIMITER //

CREATE PROCEDURE transfert_bancaire(
    IN p_de_compte INT,
    IN p_vers_compte INT,
    IN p_montant DECIMAL(10,2),
    OUT p_resultat VARCHAR(100)
)
BEGIN
    DECLARE v_solde DECIMAL(10,2);

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_resultat = 'Erreur technique';
    END;

    START TRANSACTION;

    -- Verrouiller les comptes dans un ordre déterministe
    IF p_de_compte < p_vers_compte THEN
        SELECT solde INTO v_solde FROM comptes WHERE id = p_de_compte FOR UPDATE;
        SELECT 1 FROM comptes WHERE id = p_vers_compte FOR UPDATE;
    ELSE
        SELECT 1 FROM comptes WHERE id = p_vers_compte FOR UPDATE;
        SELECT solde INTO v_solde FROM comptes WHERE id = p_de_compte FOR UPDATE;
    END IF;

    -- Vérifier le solde
    IF v_solde < p_montant THEN
        ROLLBACK;
        SET p_resultat = 'Solde insuffisant';
    ELSE
        -- Effectuer le transfert
        UPDATE comptes SET solde = solde - p_montant WHERE id = p_de_compte;
        UPDATE comptes SET solde = solde + p_montant WHERE id = p_vers_compte;

        -- Audit log
        INSERT INTO transactions_log (de_compte, vers_compte, montant, timestamp)
        VALUES (p_de_compte, p_vers_compte, p_montant, NOW());

        COMMIT;
        SET p_resultat = 'Transfert réussi';
    END IF;
END//

DELIMITER ;
```

### 10.3 Queue de Jobs avec SKIP LOCKED

```sql
-- Table de tâches
CREATE TABLE job_queue (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    job_type VARCHAR(50),
    status ENUM('pending', 'processing', 'completed', 'failed'),
    worker_id INT,
    payload JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP NULL,
    completed_at TIMESTAMP NULL,
    INDEX idx_status (status)
);

-- Fonction pour récupérer un job (worker)
DELIMITER //

CREATE PROCEDURE get_next_job(
    IN p_worker_id INT,
    OUT p_job_id BIGINT
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_job_id = NULL;
    END;

    START TRANSACTION;

    -- Récupérer un job non verrouillé
    SELECT id INTO p_job_id
    FROM job_queue
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;

    -- Si job trouvé, le marquer comme en cours
    IF p_job_id IS NOT NULL THEN
        UPDATE job_queue
        SET status = 'processing',
            worker_id = p_worker_id,
            started_at = NOW()
        WHERE id = p_job_id;

        COMMIT;
    ELSE
        ROLLBACK;
    END IF;
END//

DELIMITER ;

-- Utilisation par les workers
CALL get_next_job(1, @job_id);  -- Worker 1
-- @job_id = 123

CALL get_next_job(2, @job_id);  -- Worker 2 (simultané)
-- @job_id = 124 (job différent grâce à SKIP LOCKED)
```

---

## ✅ Points clés à retenir

- **Row-level locks** : InnoDB verrouille au niveau ligne, pas table
- **Shared (S)** : Lecture partagée, bloque les écritures
- **Exclusive (X)** : Écriture exclusive, bloque tout
- **LOCK TABLES** : À éviter avec InnoDB, utiliser FOR UPDATE
- **FOR UPDATE** : Verrou exclusif, garantit cohérence
- **FOR SHARE** : Verrou partagé, permet lectures simultanées
- **Gap locks** : Empêchent phantom reads en REPEATABLE READ
- **Ordre cohérent** : Verrouiller toujours dans le même ordre (éviter deadlocks)
- **Transactions courtes** : Minimiser la durée des verrous
- **NOWAIT/SKIP LOCKED** : Options pour gérer l'indisponibilité des verrous

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 InnoDB Locking](https://mariadb.com/kb/en/innodb-lock-modes/)
- [📖 SELECT FOR UPDATE](https://mariadb.com/kb/en/select/#lock-in-share-modefor-update)
- [📖 LOCK TABLES](https://mariadb.com/kb/en/lock-tables/)
- [📖 Information Schema INNODB_LOCKS](https://mariadb.com/kb/en/information-schema-innodb_locks-table/)
- [📖 NOWAIT and SKIP LOCKED](https://mariadb.com/kb/en/wait-and-nowait/)

### Articles techniques
- [InnoDB Locking and Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html)
- [Understanding MySQL Gap Locks](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-gap-locks)

---

## ➡️ Section suivante

**6.5 Deadlocks : Détection et résolution** : Analyse approfondie des deadlocks, stratégies de prévention et résolution en production.

---


⏭️ [Deadlocks : Détection et résolution](/06-transactions-et-concurrence/05-deadlocks-resolution.md)

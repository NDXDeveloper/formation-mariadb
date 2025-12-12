🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Concept de Transaction et Propriétés ACID

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures

> **Prérequis** :
> - Compréhension des requêtes SQL (Chapitres 2-3)
> - Notions de moteurs de stockage (Chapitre 7, InnoDB)
> - Concepts de bases de données relationnelles

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Définir précisément ce qu'est une transaction dans MariaDB
- Expliquer en détail chacune des propriétés ACID
- Comprendre comment InnoDB implémente ACID
- Identifier les compromis entre ACID et performance
- Diagnostiquer les violations d'ACID en production
- Configurer MariaDB pour différents niveaux de garanties ACID

---

## Introduction

Les **transactions** sont le fondement de la fiabilité dans les systèmes de gestion de bases de données. Sans elles, il serait impossible de garantir la cohérence des données dans un environnement où des centaines d'utilisateurs modifient simultanément les mêmes informations, où des pannes système peuvent survenir à tout moment, et où des erreurs applicatives doivent être gérées proprement.

### Qu'est-ce qu'une transaction ?

Une **transaction** est une séquence d'opérations SQL traitée comme une **unité logique indivisible**. Soit toutes les opérations réussissent ensemble, soit aucune ne s'applique.

**Définition formelle** :
> Une transaction est un ensemble d'opérations qui fait passer la base de données d'un état cohérent à un autre état cohérent, avec les garanties ACID.

**Analogie du monde réel** :
Pensez à un retrait d'argent au distributeur :
1. Vérifier le solde
2. Débiter le compte
3. Distribuer les billets
4. Imprimer le reçu

Si le distributeur tombe en panne à l'étape 3, vous ne voudriez pas que votre compte soit débité sans avoir reçu l'argent. La transaction garantit que soit TOUT se passe (débit + distribution), soit RIEN ne se passe.

---

## 1. Syntaxe de Base des Transactions

Avant d'explorer ACID en profondeur, voyons comment gérer les transactions dans MariaDB.

### 1.1 Cycle de Vie d'une Transaction

```sql
-- Démarrer une transaction explicite
START TRANSACTION;
-- ou
BEGIN;

-- Opérations SQL...
INSERT INTO commandes (client_id, montant) VALUES (100, 250.00);
UPDATE stocks SET quantite = quantite - 1 WHERE produit_id = 42;

-- Valider la transaction (rendre les modifications permanentes)
COMMIT;

-- OU annuler toutes les modifications
ROLLBACK;
```

### 1.2 Mode Autocommit

Par défaut, MariaDB fonctionne en mode **autocommit** :

```sql
-- Vérifier le mode
SELECT @@autocommit;  -- 1 = activé (default)

-- Chaque instruction est une transaction automatique
UPDATE produits SET prix = 99.99 WHERE id = 1;
-- Équivalent à :
-- START TRANSACTION;
-- UPDATE produits SET prix = 99.99 WHERE id = 1;
-- COMMIT;

-- Désactiver l'autocommit
SET autocommit = 0;
-- Maintenant, il faut un COMMIT explicite

UPDATE produits SET prix = 99.99 WHERE id = 1;
-- Pas encore visible pour les autres connexions
COMMIT;  -- Maintenant visible
```

💡 **Conseil production** : Toujours utiliser des transactions explicites (`START TRANSACTION`) pour plus de clarté, même avec autocommit activé.

### 1.3 Transactions Implicites vs Explicites

**Transaction explicite** :
```sql
START TRANSACTION;
-- Zone transactionnelle claire
COMMIT;
```

**Transaction implicite** (autocommit ON) :
```sql
UPDATE ...;  -- Transaction automatique
```

**Pourquoi préférer l'explicite ?**
- ✅ Clarté du code
- ✅ Contrôle précis des limites transactionnelles
- ✅ Gestion d'erreur plus simple
- ✅ Performance (moins de commits)

---

## 2. Les Propriétés ACID en Détail

**ACID** est l'acronyme de quatre propriétés fondamentales que tout SGBD transactionnel doit garantir :
- **A**tomicity (Atomicité)
- **C**onsistency (Cohérence)
- **I**solation
- **D**urability (Durabilité)

Ces propriétés ont été formalisées par Jim Gray dans les années 1970 et restent le standard de l'industrie.

---

## 3. A - Atomicity (Atomicité)

### 3.1 Définition

**L'atomicité garantit qu'une transaction est une unité indivisible** : soit toutes les opérations de la transaction s'appliquent, soit aucune ne s'applique. Il n'y a pas d'état intermédiaire visible.

**Principe "Tout ou Rien"** :
- ✅ Toutes les opérations réussissent → COMMIT
- ❌ Une seule opération échoue → ROLLBACK automatique de tout

### 3.2 Exemple Fondamental : Transfert Bancaire

```sql
-- Sans atomicité : DANGEREUX ❌
UPDATE comptes SET solde = solde - 1000 WHERE numero = 'A12345';
-- 💥 Panne ici = 1000€ disparaissent !
UPDATE comptes SET solde = solde + 1000 WHERE numero = 'B67890';

-- Avec atomicité : SÉCURISÉ ✅
START TRANSACTION;

UPDATE comptes SET solde = solde - 1000 WHERE numero = 'A12345';
-- Vérification métier
SELECT @nouveau_solde := solde FROM comptes WHERE numero = 'A12345';

IF @nouveau_solde < 0 THEN
    ROLLBACK;  -- ❌ Découvert non autorisé : TOUT est annulé
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Solde insuffisant';
ELSE
    UPDATE comptes SET solde = solde + 1000 WHERE numero = 'B67890';
    COMMIT;  -- ✅ Les deux comptes sont mis à jour ensemble
END IF;
```

### 3.3 Mécanisme InnoDB : Undo Log

**Comment InnoDB implémente l'atomicité** :

InnoDB utilise un **undo log** (journal d'annulation) qui enregistre l'état "avant" de chaque modification.

```
Modification :
┌─────────────────────────────────────────┐
│ 1. Écrire ANCIEN état → Undo Log        │
│ 2. Modifier la ligne dans le buffer     │
│ 3. Si COMMIT : marquer undo obsolète    │
│ 4. Si ROLLBACK : restaurer depuis undo  │
└─────────────────────────────────────────┘
```

**Exemple concret** :

```sql
START TRANSACTION;

-- État initial : stock = 100
UPDATE produits SET stock = 95 WHERE id = 42;

-- Ce qui se passe en interne :
-- 1. InnoDB écrit dans undo log : "produit 42 : stock était 100"
-- 2. InnoDB modifie le buffer pool : stock = 95
-- 3. Modification pas encore sur disque

ROLLBACK;

-- 4. InnoDB lit l'undo log : "restaurer stock = 100"
-- 5. Buffer pool : stock = 100
-- ✅ État restauré comme si l'UPDATE n'avait jamais eu lieu
```

### 3.4 Atomicité en Cas de Crash

**Scénario : Le serveur crash pendant une transaction**

```sql
START TRANSACTION;
INSERT INTO logs (message) VALUES ('Début opération');
UPDATE stocks SET qte = qte - 10 WHERE id = 1;
-- 💥 CRASH SYSTÈME ICI
UPDATE commandes SET status = 'processed' WHERE id = 999;
COMMIT;
```

**Au redémarrage** :
1. InnoDB lit le redo log (journal des modifications)
2. Identifie les transactions non commitées
3. Utilise l'undo log pour annuler toutes leurs modifications
4. ✅ Résultat : Comme si la transaction n'avait jamais existé

💡 **En production** : L'atomicité protège contre :
- Les pannes matérielles
- Les crashes du serveur MariaDB
- Les erreurs applicatives (exceptions)
- Les violations de contraintes

### 3.5 Limites de l'Atomicité

**Ce que l'atomicité NE garantit PAS** :

```sql
START TRANSACTION;

-- Opération 1 : MariaDB
UPDATE comptes SET solde = solde - 100 WHERE id = 1;

-- Opération 2 : API externe (non transactionnelle)
-- Appel HTTP vers un service de paiement
-- ❌ Pas protégé par la transaction MariaDB !

COMMIT;  -- Commit uniquement le UPDATE, pas l'API
```

⚠️ **Attention** : L'atomicité est limitée à la base de données. Les opérations externes (API REST, files de messages, fichiers) ne sont pas couvertes par ACID.

**Solution** : Transactions distribuées (XA) ou patterns comme Saga (voir section 6.8).

---

## 4. C - Consistency (Cohérence)

### 4.1 Définition

**La cohérence garantit qu'une transaction fait passer la base d'un état cohérent à un autre état cohérent**, en respectant toutes les règles de l'intégrité des données :
- Contraintes de clés primaires (PRIMARY KEY)
- Contraintes de clés étrangères (FOREIGN KEY)
- Contraintes de vérification (CHECK)
- Contraintes d'unicité (UNIQUE)
- Triggers
- Règles métier

### 4.2 Cohérence Structurelle : Contraintes

```sql
-- Définition de contraintes
CREATE TABLE commandes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    montant DECIMAL(10,2) NOT NULL CHECK (montant > 0),
    date_commande DATE NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE RESTRICT
);

-- ✅ Transaction cohérente
START TRANSACTION;
INSERT INTO clients (id, nom, email) VALUES (100, 'Alice', 'alice@example.com');
INSERT INTO commandes (client_id, montant, date_commande)
VALUES (100, 250.00, CURDATE());
COMMIT;
-- État final : cohérent (client existe, montant > 0)

-- ❌ Transaction incohérente : échec automatique
START TRANSACTION;
INSERT INTO commandes (client_id, montant, date_commande)
VALUES (999, -50.00, CURDATE());
-- Erreur 1 : client_id 999 n'existe pas (violation FK)
-- Erreur 2 : montant négatif (violation CHECK)
-- MariaDB rejette l'INSERT et force un ROLLBACK
-- ✅ État final : inchangé = cohérent
```

### 4.3 Cohérence Métier : Triggers

Les triggers permettent de maintenir des règles métier complexes :

```sql
-- Règle : Le stock total doit toujours être >= stock réservé
CREATE TRIGGER verif_stock_avant_reservation
BEFORE INSERT ON reservations
FOR EACH ROW
BEGIN
    DECLARE stock_dispo INT;

    SELECT stock_total - stock_reserve INTO stock_dispo
    FROM produits
    WHERE id = NEW.produit_id;

    IF stock_dispo < NEW.quantite THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Stock insuffisant pour la réservation';
    END IF;
END;

-- Transaction utilisant le trigger
START TRANSACTION;

-- Ceci déclenche le trigger
INSERT INTO reservations (produit_id, quantite, client_id)
VALUES (42, 10, 100);

-- Si stock insuffisant : SIGNAL → ROLLBACK automatique
-- Si stock OK : UPDATE stock réservé puis COMMIT
UPDATE produits
SET stock_reserve = stock_reserve + 10
WHERE id = 42;

COMMIT;
-- ✅ État final : cohérent (stock_total >= stock_reserve)
```

### 4.4 Cohérence et Vues Matérialisées (Workaround MariaDB)

MariaDB n'a pas de vues matérialisées natives, mais on peut les simuler avec des tables + triggers pour maintenir la cohérence :

```sql
-- Table source
CREATE TABLE ventes (
    id INT PRIMARY KEY,
    produit_id INT,
    montant DECIMAL(10,2),
    date_vente DATE
);

-- "Vue matérialisée" : totaux par produit
CREATE TABLE ventes_totaux_produit (
    produit_id INT PRIMARY KEY,
    total_ventes DECIMAL(12,2),
    nombre_ventes INT
);

-- Trigger pour maintenir la cohérence
DELIMITER //
CREATE TRIGGER maj_totaux_apres_vente
AFTER INSERT ON ventes
FOR EACH ROW
BEGIN
    INSERT INTO ventes_totaux_produit (produit_id, total_ventes, nombre_ventes)
    VALUES (NEW.produit_id, NEW.montant, 1)
    ON DUPLICATE KEY UPDATE
        total_ventes = total_ventes + NEW.montant,
        nombre_ventes = nombre_ventes + 1;
END//
DELIMITER ;

-- Transaction garantissant la cohérence
START TRANSACTION;
INSERT INTO ventes (produit_id, montant, date_vente)
VALUES (10, 150.00, CURDATE());
-- Le trigger met automatiquement à jour ventes_totaux_produit
COMMIT;
-- ✅ Les deux tables sont cohérentes
```

### 4.5 Cohérence vs Intégrité Référentielle

**Différence importante** :

```sql
-- Cohérence structurelle (automatique via FK)
ALTER TABLE commandes
ADD CONSTRAINT fk_client
FOREIGN KEY (client_id) REFERENCES clients(id);

-- ✅ MariaDB empêche automatiquement :
DELETE FROM clients WHERE id = 100;
-- si des commandes référencent ce client

-- Cohérence métier (responsabilité applicative)
-- Règle : "Un client VIP ne peut pas avoir de commande < 100€"
-- ❌ Pas de contrainte SQL native pour cela
-- ✅ Solution : Trigger ou validation applicative

DELIMITER //
CREATE TRIGGER check_montant_vip
BEFORE INSERT ON commandes
FOR EACH ROW
BEGIN
    DECLARE type_client VARCHAR(20);
    SELECT type INTO type_client FROM clients WHERE id = NEW.client_id;

    IF type_client = 'VIP' AND NEW.montant < 100 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Montant minimum pour client VIP : 100€';
    END IF;
END//
DELIMITER ;
```

### 4.6 Cohérence en Production

**Stratégies pour garantir la cohérence** :

1. **Contraintes déclaratives** (préférable) :
```sql
-- ✅ Performant, automatique, documenté
CREATE TABLE produits (
    id INT PRIMARY KEY,
    prix DECIMAL(10,2) CHECK (prix > 0),
    stock INT CHECK (stock >= 0)
);
```

2. **Triggers** (pour logique complexe) :
```sql
-- ✅ Permet des règles métier avancées
-- ⚠️ Performance : exécuté à chaque opération
```

3. **Validation applicative** (complément) :
```python
# ✅ Feedback immédiat à l'utilisateur
# ⚠️ Ne remplace PAS les contraintes DB (risque de contournement)
if montant < 0:
    raise ValueError("Montant invalide")
```

💡 **Meilleure pratique** : Combinaison des trois :
- Contraintes DB = dernière ligne de défense
- Triggers = règles métier complexes
- Validation applicative = UX

---

## 5. I - Isolation

### 5.1 Définition

**L'isolation garantit que les transactions concurrentes s'exécutent comme si elles étaient seules**, sans interférence mutuelle. Chaque transaction a l'illusion d'avoir un accès exclusif à la base de données.

### 5.2 Pourquoi l'Isolation est Critique

**Problème sans isolation** :

```sql
-- Comptes bancaires : A = 1000€, B = 500€, Total = 1500€

-- Transaction 1 : Calcul du total
SELECT SUM(solde) FROM comptes;  -- Lit : 1500€
-- Utilise ce total pour un rapport financier

-- Transaction 2 (en parallèle) : Transfert
UPDATE comptes SET solde = 2000 WHERE id = 'A';  -- Dépôt sur A
-- ⏳ Pas encore committé

-- Transaction 1 (continue)
SELECT SUM(solde) FROM comptes;  -- Lit : 2500€ ou 1500€ ?
-- 💥 Résultat incohérent : le total a changé en pleine transaction !
```

### 5.3 Phénomènes d'Isolation

La norme SQL définit trois **phénomènes indésirables** que l'isolation doit prévenir :

#### 5.3.1 Dirty Read (Lecture Sale)

**Lire des données non commitées** d'une autre transaction :

```sql
-- Transaction A
START TRANSACTION;
UPDATE produits SET prix = 99.99 WHERE id = 1;
-- ⏳ Pas de COMMIT

-- Transaction B
SELECT prix FROM produits WHERE id = 1;
-- Lit : 99.99 (donnée "sale", non validée)

-- Transaction A
ROLLBACK;  -- Finalement on annule
-- 💥 Transaction B a lu une donnée qui n'existe plus !
```

**Impact** : Décisions basées sur des données qui seront peut-être annulées.

#### 5.3.2 Non-Repeatable Read (Lecture Non Répétable)

**Une même requête retourne des résultats différents** dans la même transaction :

```sql
-- Transaction A
START TRANSACTION;
SELECT prix FROM produits WHERE id = 1;  -- Lit : 50.00

-- Transaction B
UPDATE produits SET prix = 75.00 WHERE id = 1;
COMMIT;

-- Transaction A (même transaction)
SELECT prix FROM produits WHERE id = 1;  -- Lit : 75.00
-- 💥 La valeur a changé entre deux lectures !
COMMIT;
```

**Impact** : Calculs ou rapports incohérents.

#### 5.3.3 Phantom Read (Lecture Fantôme)

**De nouvelles lignes apparaissent** entre deux lectures :

```sql
-- Transaction A
START TRANSACTION;
SELECT COUNT(*) FROM commandes WHERE client_id = 5;  -- Compte : 10

-- Transaction B
INSERT INTO commandes (client_id, montant) VALUES (5, 100.00);
COMMIT;

-- Transaction A
SELECT COUNT(*) FROM commandes WHERE client_id = 5;  -- Compte : 11
-- 💥 Une "fantôme" ligne est apparue !
COMMIT;
```

**Impact** : Agrégations et statistiques erronées.

### 5.4 Niveaux d'Isolation : Résumé

Les quatre niveaux d'isolation SQL standard contrôlent quels phénomènes sont autorisés :

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read |
|--------|------------|---------------------|--------------|
| READ UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible |
| READ COMMITTED | ❌ Bloqué | ✅ Possible | ✅ Possible |
| REPEATABLE READ | ❌ Bloqué | ❌ Bloqué | ⚠️ InnoDB : Bloqué* |
| SERIALIZABLE | ❌ Bloqué | ❌ Bloqué | ❌ Bloqué |

\* InnoDB va au-delà du standard SQL : REPEATABLE READ empêche aussi les phantom reads grâce aux gap locks.

**Configuration** :

```sql
-- Niveau global (serveur)
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Niveau session (connexion actuelle)
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Niveau transaction (une seule transaction)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
-- ...
COMMIT;
```

### 5.5 Default InnoDB : REPEATABLE READ

**Pourquoi ce choix ?**

```sql
-- Test du niveau par défaut
SELECT @@transaction_isolation;
-- Résultat : 'REPEATABLE-READ'

-- Démonstration
START TRANSACTION;

SELECT stock FROM produits WHERE id = 1;  -- Lit : 100

-- Une autre session modifie
-- UPDATE produits SET stock = 50 WHERE id = 1; COMMIT;

SELECT stock FROM produits WHERE id = 1;  -- Lit : 100 (inchangé)
-- ✅ Lecture répétable : vue cohérente pendant toute la transaction

COMMIT;

-- Maintenant on voit le changement
SELECT stock FROM produits WHERE id = 1;  -- Lit : 50
```

**Avantages de REPEATABLE READ** :
- ✅ Lectures cohérentes (important pour les rapports)
- ✅ Performance acceptable (grâce au MVCC)
- ✅ Empêche les phantom reads (spécificité InnoDB)

💡 **Note** : PostgreSQL et Oracle utilisent READ COMMITTED par défaut. Le choix de REPEATABLE READ par InnoDB est plus conservateur (plus de cohérence).

### 5.6 Isolation et Performance : Le Compromis

```
Isolation élevée ←→ Performance élevée
     🔒              ⚡
SERIALIZABLE    READ UNCOMMITTED

   Plus de verrous     Moins de verrous
   Plus de blocages    Plus de concurrence
   Lectures cohérentes Lectures incohérentes
```

**Règle générale** :
- Applications critiques (finance) → Isolation élevée
- Applications web standards → REPEATABLE READ (default)
- Analytics/rapports temps réel → READ COMMITTED voire UNCOMMITTED

---

## 6. D - Durability (Durabilité)

### 6.1 Définition

**La durabilité garantit qu'une fois une transaction commitée, ses modifications sont permanentes**, même en cas de :
- Panne de courant
- Crash du serveur MariaDB
- Crash du système d'exploitation
- Défaillance matérielle (dans une certaine mesure)

**Principe** : Une fois que `COMMIT` retourne, les données sont **sur disque de manière durable**.

### 6.2 Mécanisme InnoDB : Redo Log

InnoDB utilise un **redo log** (journal de récupération) pour garantir la durabilité :

```
Transaction COMMIT :
┌────────────────────────────────────────────────┐
│ 1. Modifications en mémoire (buffer pool)      │
│ 2. Écrire les modifications → Redo Log         │
│ 3. Redo Log écrit sur disque (fsync)           │
│ 4. Retourner "COMMIT OK" à l'application       │
│ 5. Plus tard : Écrire buffer pool → data file  │
└────────────────────────────────────────────────┘
```

**Pourquoi deux étapes ?**

L'écriture dans le redo log est **séquentielle** (rapide), tandis que l'écriture dans les fichiers de données est **aléatoire** (lent). On écrit d'abord dans le redo log, puis on applique aux données en différé.

**En cas de crash** :

```
Au redémarrage :
1. InnoDB lit le redo log
2. Rejoue toutes les transactions commitées
3. ✅ Données restaurées même si le buffer n'a pas été écrit
```

### 6.3 Configuration de la Durabilité

**Variable clé : `innodb_flush_log_at_trx_commit`**

```ini
[mysqld]
# Valeur 0 : Performance maximale, durabilité minimale
innodb_flush_log_at_trx_commit = 0
# Redo log écrit toutes les ~1s, pas à chaque COMMIT
# ⚠️ Risque : perte des transactions des ~1 dernières secondes si crash

# Valeur 1 : Durabilité maximale (DEFAULT)
innodb_flush_log_at_trx_commit = 1
# Redo log écrit ET synchronisé (fsync) à CHAQUE COMMIT
# ✅ Garantie ACID complète
# ⚠️ Performance : plus lent (I/O disque à chaque commit)

# Valeur 2 : Compromis
innodb_flush_log_at_trx_commit = 2
# Redo log écrit à chaque COMMIT, mais fsync toutes les ~1s
# ⚠️ Risque : perte si crash du système d'exploitation (pas juste MariaDB)
```

**Comparaison de performance** :

| Mode | Commits/sec | Durabilité | Cas d'usage |
|------|-------------|------------|-------------|
| 0 | ~10,000 | ⚠️ Faible | Logs, data non critique |
| 1 | ~1,000 | ✅ Totale | Finance, e-commerce |
| 2 | ~5,000 | 🟡 Partielle | Applications web standards |

### 6.4 Double Write Buffer

InnoDB utilise un mécanisme supplémentaire pour protéger contre les **corruptions de pages partielles** :

```
Écriture d'une page :
┌─────────────────────────────────────────────┐
│ 1. Écrire page → Double Write Buffer (DWB)  │
│ 2. Flush DWB sur disque                     │
│ 3. Écrire page → emplacement final          │
└─────────────────────────────────────────────┘
```

**Pourquoi ?**

Une page InnoDB fait 16KB. Si le système crashe pendant l'écriture, on pourrait avoir une page "déchirée" (moitié ancienne, moitié nouvelle). Le DWB permet de détecter et récupérer ces corruptions.

```ini
[mysqld]
# Activer le double write buffer (DEFAULT)
innodb_doublewrite = ON

# Désactiver (gain de performance, risque de corruption)
# ⚠️ UNIQUEMENT si système de fichiers garantit l'atomicité (ex: ZFS, Btrfs)
innodb_doublewrite = OFF
```

### 6.5 Durabilité et SSD

Avec les SSD modernes :

```ini
[mysqld]
# Sur SSD, le double write est moins critique
# (écritures plus rapides et plus fiables)

# Performance optimale sur SSD :
innodb_flush_log_at_trx_commit = 1  # Garder la durabilité
innodb_io_capacity = 2000  # Augmenter (500-2000 pour SSD)
innodb_io_capacity_max = 4000

# Sur SSD NVMe ultra-rapide :
innodb_flush_method = O_DIRECT  # Bypass du cache système
```

### 6.6 Test de Durabilité

**Expérience** : Vérifier que les données survivent à un crash

```bash
# Terminal 1 : Créer une transaction
mariadb -e "
START TRANSACTION;
INSERT INTO test_durability (id, data) VALUES (1, 'test');
COMMIT;
"

# Terminal 2 : Crash simulé
sudo killall -9 mariadbd  # SIGKILL brutal

# Redémarrer MariaDB
sudo systemctl start mariadb

# Vérifier les données
mariadb -e "SELECT * FROM test_durability WHERE id = 1;"
# ✅ Doit retourner la ligne : la durabilité a fonctionné
```

### 6.7 Durabilité vs Performance : Configuration par Cas d'Usage

**Finance / E-commerce** :
```ini
innodb_flush_log_at_trx_commit = 1
innodb_doublewrite = ON
sync_binlog = 1  # Synchroniser aussi les binary logs
```

**Application web standard** :
```ini
innodb_flush_log_at_trx_commit = 2
innodb_doublewrite = ON
sync_binlog = 100  # Toutes les 100 transactions
```

**Analytics / Logs non critiques** :
```ini
innodb_flush_log_at_trx_commit = 0
innodb_doublewrite = OFF  # Si SSD
```

---

## 7. ACID : Vue d'Ensemble et Interactions

### 7.1 Les Quatre Propriétés Ensemble

ACID n'est pas quatre propriétés indépendantes, mais un **système cohérent** :

```
Atomicity ──┐
            ├─→ Transaction
Consistency ┤     fiable et
Isolation  ─┤     prévisible
Durability ─┘
```

**Exemple intégrant les 4 propriétés** :

```sql
-- Système de réservation de places de concert
START TRANSACTION;

-- Atomicité : Tout réussit ou tout échoue
UPDATE places SET statut = 'reservee', client_id = 100 WHERE id IN (1, 2, 3);
INSERT INTO reservations (client_id, places) VALUES (100, '1,2,3');
UPDATE clients SET credits = credits - 150 WHERE id = 100;

-- Cohérence : Vérification des contraintes
-- - places.client_id existe (FK)
-- - credits >= 0 (CHECK)
-- - statut IN ('libre', 'reservee') (CHECK)

-- Isolation : Empêche deux clients de réserver la même place
-- (REPEATABLE READ + FOR UPDATE implicite sur UPDATE)

-- Durabilité : Une fois COMMIT, impossible de perdre la réservation
COMMIT;
-- ✅ Les 4 propriétés garantissent la fiabilité
```

### 7.2 Compromis ACID vs Performance

**Le Triangle Impossible** :

```
      Performance
          /\
         /  \
        /    \
       /______\
   ACID        Simplicité
```

**En pratique** :

| Cas | Priorité | Compromis |
|-----|----------|-----------|
| Banque | ACID > Performance | Transactions + lentes, mais fiables |
| E-commerce | ACID ≈ Performance | READ COMMITTED, optimisations |
| Analytics | Performance > ACID | Cohérence éventuelle acceptable |
| Logs | Performance >> ACID | Perte de quelques secondes OK |

### 7.3 ACID et Architectures Modernes

**Microservices : Défi de l'ACID** :

```
┌──────────┐         ┌──────────┐
│ Service A│         │ Service B│
│ (MariaDB)│         │ (MariaDB)│
└──────────┘         └──────────┘
     🔒                   🔒
   Transaction        Transaction

❌ Pas d'ACID global entre les deux bases !
```

**Solutions** :
1. **Transaction distribuée (XA)** : ACID global, mais complexe
2. **Saga Pattern** : Cohérence éventuelle + compensation
3. **Event Sourcing** : Log immuable d'événements
4. **Base unique partagée** : Simple, mais couplage fort

💡 **Tendance moderne** : Accepter l'éventualité (eventual consistency) plutôt que forcer ACID partout.

---

## 8. Implications en Production

### 8.1 Diagnostic des Violations ACID

**Atomicité** : Transaction partiellement appliquée

```sql
-- Symptôme : Données incohérentes après erreur
-- Commande créée mais stock non décrémenté

-- Diagnostic
SELECT
    c.id AS commande_id,
    p.stock,
    ci.quantite
FROM commandes c
JOIN commande_items ci ON c.id = ci.commande_id
JOIN produits p ON ci.produit_id = p.id
WHERE c.statut = 'completed' AND p.stock + ci.quantite != (
    SELECT stock FROM produits_history WHERE produit_id = p.id LIMIT 1
);
-- Si résultats → violation d'atomicité
```

**Cohérence** : Contraintes violées

```sql
-- Vérifier l'intégrité référentielle
SELECT
    'commandes orphelines' AS probleme,
    COUNT(*) AS nb
FROM commandes c
LEFT JOIN clients cl ON c.client_id = cl.id
WHERE cl.id IS NULL

UNION ALL

SELECT
    'soldes négatifs' AS probleme,
    COUNT(*)
FROM comptes
WHERE solde < 0;
```

**Isolation** : Lectures incohérentes

```sql
-- Symptôme : Totaux qui ne correspondent pas
-- Debug : Activer le logging des transactions
SET GLOBAL general_log = ON;
SET GLOBAL log_output = 'TABLE';

-- Analyser les transactions concurrentes
SELECT
    thread_id,
    event_time,
    argument
FROM mysql.general_log
WHERE argument LIKE 'UPDATE commandes%'
ORDER BY event_time;
```

**Durabilité** : Données perdues après crash

```bash
# Vérifier les crash recovery dans error log
sudo grep -i "crash\|recovery\|redo" /var/log/mysql/error.log

# Vérifier la configuration
mariadb -e "SHOW VARIABLES LIKE 'innodb_flush%';"
```

### 8.2 Monitoring ACID en Production

**Métriques clés** :

```sql
-- 1. Transactions actives
SELECT COUNT(*) AS active_transactions
FROM information_schema.INNODB_TRX;

-- 2. Transactions longues (risque d'atomicité)
SELECT
    trx_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_query
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 5 MINUTE;

-- 3. Rollbacks (fréquence des échecs)
SHOW GLOBAL STATUS LIKE 'Com_rollback';
SHOW GLOBAL STATUS LIKE 'Com_commit';
-- Ratio rollback/commit > 10% = problème

-- 4. Lock waits (problème d'isolation)
SELECT COUNT(*) AS lock_waits
FROM information_schema.INNODB_LOCK_WAITS;
```

**Alertes recommandées** :
- 🚨 Transaction active > 10 minutes
- 🚨 Ratio rollback/commit > 20%
- 🚨 Plus de 50 lock waits simultanés
- 🚨 `innodb_flush_log_at_trx_commit != 1` en production critique

### 8.3 Configuration Optimale par Environnement

**Production Critique** :
```ini
[mysqld]
# Durabilité maximale
innodb_flush_log_at_trx_commit = 1
innodb_doublewrite = ON
sync_binlog = 1

# Isolation par défaut : sûre
transaction_isolation = REPEATABLE-READ

# Taille des logs pour absorber les pics
innodb_log_file_size = 512M
```

**Production Standard** :
```ini
[mysqld]
# Compromis performance/durabilité
innodb_flush_log_at_trx_commit = 2
innodb_doublewrite = ON
sync_binlog = 100

transaction_isolation = REPEATABLE-READ
innodb_log_file_size = 256M
```

**Développement** :
```ini
[mysqld]
# Performance maximale
innodb_flush_log_at_trx_commit = 0
innodb_doublewrite = OFF
sync_binlog = 0

transaction_isolation = READ-COMMITTED
```

---

## ✅ Points clés à retenir

- **Transaction** : Unité logique indivisible garantissant ACID
- **Atomicité** : Tout ou rien, implémentée via undo log
- **Cohérence** : Respect des contraintes et règles métier
- **Isolation** : Protection contre les accès concurrents (REPEATABLE READ par défaut)
- **Durabilité** : Persistance garantie après COMMIT via redo log
- **innodb_flush_log_at_trx_commit = 1** : Durabilité maximale (production critique)
- **MVCC** : Permet lectures sans blocage (détaillé en section 6.6)
- **Compromis** : ACID strict = moins de performance, mais fiabilité maximale

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 START TRANSACTION](https://mariadb.com/kb/en/start-transaction/)
- [📖 COMMIT](https://mariadb.com/kb/en/commit/)
- [📖 ROLLBACK](https://mariadb.com/kb/en/rollback/)
- [📖 InnoDB Transaction Model](https://mariadb.com/kb/en/innodb-transaction-model/)
- [📖 innodb_flush_log_at_trx_commit](https://mariadb.com/kb/en/innodb-system-variables/#innodb_flush_log_at_trx_commit)

### Articles académiques
- Jim Gray, "The Transaction Concept: Virtues and Limitations" (1981)
- [ACID Properties of Transactions](https://en.wikipedia.org/wiki/ACID)

### Livres recommandés
- "Designing Data-Intensive Applications" by Martin Kleppmann - Chapitre 7 (Transactions)
- "Database Internals" by Alex Petrov - Chapitre sur les transactions

---

## ➡️ Section suivante

**6.2 Gestion des transactions (START TRANSACTION, BEGIN, COMMIT, ROLLBACK)** : Syntaxe complète, options avancées, savepoints et gestion d'erreur.

---


⏭️ [Gestion des transactions (START TRANSACTION, BEGIN, COMMIT, ROLLBACK)](/06-transactions-et-concurrence/02-gestion-transactions.md)

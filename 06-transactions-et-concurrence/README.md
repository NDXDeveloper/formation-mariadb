🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6. Transactions et Concurrence

> **Niveau** : Avancé
> **Durée estimée** : 8-10 heures

> **Prérequis** :
> - Maîtrise du SQL (Chapitres 2-4)
> - Compréhension des moteurs de stockage (Chapitre 7, section InnoDB)
> - Notions de systèmes concurrents

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :
- Maîtriser les propriétés ACID et leur importance en production
- Choisir le niveau d'isolation approprié selon le cas d'usage
- Comprendre et exploiter le MVCC pour la concurrence
- Gérer efficacement les verrous et prévenir les deadlocks
- Implémenter des transactions distribuées avec XA
- Diagnostiquer et résoudre les problèmes de concurrence en production

---

## Introduction

Dans un environnement multi-utilisateurs, où des dizaines, centaines, voire milliers de connexions accèdent simultanément aux mêmes données, **la gestion de la concurrence devient critique**. Les transactions sont le mécanisme fondamental qui garantit l'intégrité des données face à ces accès simultanés, aux pannes système et aux erreurs applicatives.

### Pourquoi les transactions sont essentielles

Imaginez un système bancaire sans transactions :

```sql
-- ❌ SANS transaction : DANGEREUX
UPDATE comptes SET solde = solde - 100 WHERE id = 1;  -- Débit compte A
-- 💥 Panne système ici = l'argent disparaît !
UPDATE comptes SET solde = solde + 100 WHERE id = 2;  -- Crédit compte B
```

Avec une transaction :

```sql
-- ✅ AVEC transaction : SÉCURISÉ
START TRANSACTION;
UPDATE comptes SET solde = solde - 100 WHERE id = 1;
UPDATE comptes SET solde = solde + 100 WHERE id = 2;
COMMIT;  -- Les deux opérations réussissent ensemble ou échouent ensemble
```

### Ce que vous allez apprendre

Ce chapitre couvre l'ensemble des mécanismes de MariaDB pour gérer la concurrence en production :

1. **ACID** : Les quatre piliers de la fiabilité transactionnelle
2. **Niveaux d'isolation** : Le compromis entre cohérence et performance
3. **MVCC** : Comment InnoDB permet de lire sans bloquer
4. **Verrous** : Les mécanismes de contrôle d'accès
5. **Deadlocks** : Détection, prévention et résolution
6. **Transactions distribuées** : Coordination multi-bases avec XA

---

## 1. Les Propriétés ACID : Fondements de la Fiabilité

ACID est l'acronyme qui définit les quatre propriétés garantissant la fiabilité des transactions dans un SGBD.

### 1.1 Atomicity (Atomicité)

**Principe** : Une transaction est une unité indivisible. Toutes les opérations réussissent ensemble, ou aucune ne réussit.

**Analogie** : Comme un transfert bancaire où débit ET crédit doivent réussir, ou rien ne se passe.

```sql
-- Exemple : Transfert bancaire
START TRANSACTION;

UPDATE comptes SET solde = solde - 500 WHERE id = 'compte_A';
-- Vérification métier
SELECT @solde := solde FROM comptes WHERE id = 'compte_A';

IF @solde < 0 THEN
    ROLLBACK;  -- ❌ Annule TOUT : le solde est restauré
ELSE
    UPDATE comptes SET solde = solde + 500 WHERE id = 'compte_B';
    COMMIT;    -- ✅ Valide TOUT
END IF;
```

**Mécanisme MariaDB/InnoDB** :
- **Undo Log** : Enregistre l'état "avant" pour pouvoir annuler
- En cas de ROLLBACK ou crash, l'undo log restaure les données

💡 **En production** : L'atomicité protège contre les pannes. Si le serveur crash entre deux UPDATE, au redémarrage, InnoDB utilise l'undo log pour annuler les modifications partielles.

### 1.2 Consistency (Cohérence)

**Principe** : Une transaction fait passer la base d'un état cohérent à un autre état cohérent, en respectant toutes les contraintes (PK, FK, CHECK, triggers).

**Exemple avec contraintes** :

```sql
-- Schema avec contraintes
CREATE TABLE commandes (
    id INT PRIMARY KEY,
    client_id INT NOT NULL,
    montant DECIMAL(10,2) CHECK (montant > 0),
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

-- ✅ Transaction cohérente
START TRANSACTION;
INSERT INTO clients (id, nom) VALUES (100, 'Alice');
INSERT INTO commandes (id, client_id, montant) VALUES (1, 100, 250.00);
COMMIT;

-- ❌ Transaction incohérente : ROLLBACK automatique
START TRANSACTION;
INSERT INTO commandes (id, client_id, montant) VALUES (2, 999, -50);
-- Violation FK (client 999 n'existe pas) ET CHECK (montant négatif)
-- MariaDB rejette et force un ROLLBACK
```

**Mécanisme MariaDB** :
- Vérification des contraintes à chaque opération
- Exécution des triggers pour maintenir la cohérence métier
- Validation finale au COMMIT

⚠️ **Attention** : La cohérence applicative (logique métier complexe) reste sous votre responsabilité. Les contraintes SQL ne couvrent pas tous les cas.

### 1.3 Isolation

**Principe** : Les transactions s'exécutent comme si elles étaient seules, même en présence de centaines de transactions concurrentes.

**Le problème sans isolation** :

```sql
-- Session 1 : Calcul du total
SELECT SUM(montant) FROM commandes;  -- Résultat : 10,000 €

-- Session 2 : Pendant ce temps...
INSERT INTO commandes VALUES (..., 5000);  -- Nouvelle commande

-- Session 1 : Recalcul
SELECT SUM(montant) FROM commandes;  -- Résultat : 15,000 €
-- ❌ Lecture incohérente (non-repeatable read)
```

**Solution : Niveaux d'isolation** (détaillés dans la section suivante).

### 1.4 Durability (Durabilité)

**Principe** : Une fois qu'une transaction est COMMITée, les modifications sont **permanentes**, même en cas de panne système.

**Mécanisme MariaDB/InnoDB** :

```sql
START TRANSACTION;
UPDATE stocks SET quantite = quantite - 10 WHERE produit_id = 42;
COMMIT;
-- ✅ À ce stade, la modification est DURABLE
-- 💥 Même si le serveur crash 1 seconde après, la donnée est sauvegardée
```

**Comment InnoDB garantit la durabilité** :

1. **Redo Log** (journal de transactions) :
   - Écrit séquentiel sur disque AVANT le commit
   - Optimisé pour les performances (append-only)

2. **Double-Write Buffer** :
   - Protection contre les corruptions de pages partielles
   - Garantit l'intégrité physique des données

3. **Configuration** :

```ini
[mysqld]
# Durabilité maximale (default InnoDB)
innodb_flush_log_at_trx_commit = 1  # Flush à chaque COMMIT

# Performance vs durabilité
innodb_flush_log_at_trx_commit = 2  # Flush au système toutes les 1s
# ⚠️ Risque de perte : dernière seconde en cas de crash OS
```

💡 **Compromis production** :
- **Finance, e-commerce** : `= 1` (durabilité maximale)
- **Analytics, logs** : `= 2` (performance, perte acceptable)

---

## 2. Niveaux d'Isolation : Le Compromis Cohérence/Performance

Les niveaux d'isolation définissent **quel degré de cohérence** vous acceptez en échange de **quelle performance**. Plus l'isolation est forte, plus les lectures sont cohérentes, mais plus la concurrence est limitée.

### 2.1 Vue d'ensemble des quatre niveaux

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|--------|------------|---------------------|--------------|-------------|
| **READ UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible | ⭐⭐⭐⭐⭐ |
| **READ COMMITTED** | ❌ Bloqué | ✅ Possible | ✅ Possible | ⭐⭐⭐⭐ |
| **REPEATABLE READ** (default InnoDB) | ❌ Bloqué | ❌ Bloqué | ⚠️ Limité InnoDB | ⭐⭐⭐ |
| **SERIALIZABLE** | ❌ Bloqué | ❌ Bloqué | ❌ Bloqué | ⭐⭐ |

### 2.2 READ UNCOMMITTED : Lecture Sale (Dirty Read)

**Comportement** : Lit les données **non commitées** d'autres transactions.

```sql
-- Session 1
START TRANSACTION;
UPDATE produits SET prix = 99.99 WHERE id = 1;
-- ⏳ Transaction en cours, pas de COMMIT

-- Session 2 (READ UNCOMMITTED)
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT prix FROM produits WHERE id = 1;
-- Résultat : 99.99 ❌ Lecture d'une donnée non validée !

-- Session 1
ROLLBACK;  -- Finalement, on annule

-- Session 2 : La donnée lue était "sale" (dirty)
```

**Cas d'usage** :
- ✅ Rapports approximatifs en temps réel (dashboards)
- ✅ Statistiques non critiques
- ❌ **JAMAIS pour des données financières ou critiques**

⚠️ **Danger** : Lire des données qui seront peut-être annulées peut mener à des décisions incorrectes.

### 2.3 READ COMMITTED : Pas de Lecture Sale

**Comportement** : Lit uniquement les données commitées. Mais une même requête peut retourner des résultats différents si exécutée deux fois dans la même transaction.

```sql
-- Session 1
START TRANSACTION;
SELECT stock FROM produits WHERE id = 1;  -- Résultat : 100

-- Session 2
UPDATE produits SET stock = 50 WHERE id = 1;
COMMIT;

-- Session 1 (même transaction)
SELECT stock FROM produits WHERE id = 1;  -- Résultat : 50 ⚠️ Changé !
-- Non-repeatable read : la donnée a changé entre deux lectures
```

**Cas d'usage** :
- ✅ Applications web classiques avec transactions courtes
- ✅ Systèmes où la cohérence stricte n'est pas critique
- Default de **PostgreSQL** et **Oracle**

💡 **Compromis** : Bonne performance, cohérence acceptable pour la plupart des applications.

### 2.4 REPEATABLE READ : Le Default d'InnoDB

**Comportement** : Garantit que les lectures répétées retournent les mêmes données **pour les lignes existantes** au début de la transaction.

```sql
-- Configuration
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Session 1
START TRANSACTION;
SELECT stock FROM produits WHERE id = 1;  -- Résultat : 100

-- Session 2
UPDATE produits SET stock = 50 WHERE id = 1;
COMMIT;

-- Session 1 (même transaction)
SELECT stock FROM produits WHERE id = 1;  -- Résultat : 100 ✅ Inchangé
-- Lecture répétable : on voit toujours la même valeur
COMMIT;
```

**Phantom Reads : Cas spécial InnoDB**

En SQL standard, REPEATABLE READ autorise les "phantom reads" (nouvelles lignes apparaissant). **InnoDB les empêche grâce aux "gap locks"** :

```sql
-- Session 1
START TRANSACTION;
SELECT COUNT(*) FROM commandes WHERE client_id = 5;  -- Résultat : 10

-- Session 2 tente d'insérer
INSERT INTO commandes (client_id, ...) VALUES (5, ...);
-- ⏳ BLOQUÉ par le gap lock d'InnoDB

-- Session 1
SELECT COUNT(*) FROM commandes WHERE client_id = 5;  -- Toujours 10 ✅
COMMIT;

-- Session 2 peut maintenant insérer
```

**Cas d'usage** :
- ✅ **Default InnoDB** : Excellent compromis pour la plupart des applications
- ✅ Transactions nécessitant des lectures cohérentes (rapports, calculs)
- ✅ E-commerce, gestion de stocks

🆕 **MariaDB 11.8** : Optimisations du gap locking pour réduire les contentions.

### 2.5 SERIALIZABLE : Isolation Maximale

**Comportement** : Les transactions s'exécutent comme si elles étaient sérialisées (une après l'autre). Lecture = verrou partagé automatique.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Session 1
START TRANSACTION;
SELECT * FROM produits WHERE categorie = 'Électronique';
-- ✅ Verrou de lecture (shared lock) sur TOUTES les lignes

-- Session 2
UPDATE produits SET prix = prix * 1.1 WHERE categorie = 'Électronique';
-- ⏳ BLOQUÉ jusqu'au COMMIT de Session 1

-- Session 1
COMMIT;  -- Libère les verrous

-- Session 2 peut maintenant s'exécuter
```

**Cas d'usage** :
- ✅ Transactions critiques nécessitant une cohérence absolue
- ✅ Audits financiers, rapports réglementaires
- ❌ **Performance faible** : À éviter pour des applications à haute concurrence

⚠️ **Risque de deadlock élevé** : Les verrous de lecture automatiques augmentent les risques de deadlock.

### 2.6 Choisir le Bon Niveau d'Isolation

**Guide de décision** :

```
┌─────────────────────────────────────────────────┐
│ Ai-je besoin de cohérence absolue ?             │
│   OUI → SERIALIZABLE                            │
│   NON ↓                                         │
├─────────────────────────────────────────────────┤
│ Mes lectures doivent-elles être répétables ?    │
│   OUI → REPEATABLE READ (default InnoDB) ✅     │
│   NON ↓                                         │
├─────────────────────────────────────────────────┤
│ Puis-je tolérer des lectures non commitées ?    │
│   OUI → READ UNCOMMITTED (dashboards)           │
│   NON → READ COMMITTED                          │
└─────────────────────────────────────────────────┘
```

**Exemples réels** :

| Application | Niveau recommandé | Raison |
|-------------|-------------------|--------|
| E-commerce (panier) | REPEATABLE READ | Cohérence stock + performance |
| Dashboard temps réel | READ UNCOMMITTED | Performance maximale acceptable |
| Système bancaire | SERIALIZABLE | Cohérence absolue requise |
| Application web standard | READ COMMITTED | Compromis raisonnable |

---

## 3. MVCC : Multi-Version Concurrency Control

Le MVCC est **la technologie clé** qui permet à InnoDB d'offrir des performances élevées en environnement concurrent. Au lieu de bloquer les lecteurs, InnoDB maintient **plusieurs versions** des données.

### 3.1 Principe du MVCC

**Sans MVCC** (ancien MyISAM) :

```
Writer locks data → Readers BLOCKED → Performance ❌
```

**Avec MVCC** (InnoDB) :

```
Writer modifies → Creates new version
Readers see old version → NO BLOCKING → Performance ✅
```

### 3.2 Comment fonctionne le MVCC

Chaque ligne InnoDB possède des **métadonnées cachées** :

```
+------------------+------------------+----------------------+
| Données visibles | DB_TRX_ID (6B)   | DB_ROLL_PTR (7B)     |
| par l'utilisateur| Transaction ID   | Pointeur vers undo   |
+------------------+------------------+----------------------+
```

**Exemple concret** :

```sql
-- État initial
id | nom    | DB_TRX_ID
1  | Alice  | 100

-- Transaction 150 modifie
START TRANSACTION;  -- TRX_ID = 150
UPDATE users SET nom = 'Alicia' WHERE id = 1;

-- Version dans InnoDB :
id | nom     | DB_TRX_ID | DB_ROLL_PTR
1  | Alicia  | 150       | → [Undo: Alice, TRX=100]

-- Transaction 140 (commencée AVANT 150) lit :
SELECT nom FROM users WHERE id = 1;
-- Résultat : "Alice" ✅ Voit l'ancienne version via undo log
-- Transaction 140 < 150 → lit la version précédente

-- Transaction 160 (commencée APRÈS 150 commit) lit :
SELECT nom FROM users WHERE id = 1;
-- Résultat : "Alicia" ✅ Voit la nouvelle version
```

### 3.3 Consistent Read vs Locking Read

**Consistent Read (non-locking)** :

```sql
-- Lecture normale : MVCC, pas de verrou
SELECT * FROM produits WHERE id = 1;
-- ✅ N'empêche JAMAIS les écritures
```

**Locking Read (explicit)** :

```sql
-- FOR UPDATE : Verrou exclusif
SELECT * FROM produits WHERE id = 1 FOR UPDATE;
-- ⏳ Bloque les autres FOR UPDATE et UPDATE

-- FOR SHARE (LOCK IN SHARE MODE) : Verrou partagé
SELECT * FROM produits WHERE id = 1 FOR SHARE;
-- ✅ Permet d'autres FOR SHARE
-- ⏳ Bloque les FOR UPDATE et UPDATE
```

**Cas d'usage** :

```sql
-- E-commerce : Réserver un produit
START TRANSACTION;

-- Lecture avec verrou exclusif
SELECT stock INTO @stock
FROM produits
WHERE id = 42
FOR UPDATE;  -- 🔒 Verrou : personne d'autre ne peut modifier

IF @stock > 0 THEN
    UPDATE produits SET stock = stock - 1 WHERE id = 42;
    INSERT INTO commandes (...) VALUES (...);
    COMMIT;  -- ✅ Stock cohérent garanti
ELSE
    ROLLBACK;
END IF;
```

### 3.4 Read View et Snapshot Isolation

En **REPEATABLE READ**, InnoDB crée un "snapshot" au début de la transaction :

```sql
-- Transaction A
START TRANSACTION;  -- Snapshot créé à T0
SELECT SUM(solde) FROM comptes;  -- Lit snapshot T0

-- 10 minutes plus tard, beaucoup de transactions ont commit...

SELECT SUM(solde) FROM comptes;  -- ✅ Lit toujours snapshot T0
-- Vue cohérente figée dans le temps
COMMIT;
```

💡 **Avantage** : Rapports cohérents même sur de longues transactions, sans bloquer les écritures.

⚠️ **Attention** : Les snapshots consomment de l'espace dans l'undo log. Des transactions très longues peuvent :
- Augmenter la taille du tablespace système
- Dégrader les performances (undo log devient volumineux)

**Monitoring** :

```sql
-- Vérifier les transactions longues
SELECT *
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 10 MINUTE;
```

---

## 4. Gestion des Verrous (Locks)

Les verrous contrôlent l'accès concurrent aux données. InnoDB utilise un système de verrouillage sophistiqué pour maximiser la concurrence.

### 4.1 Types de Verrous

**1. Row-Level Locks (verrous de ligne)** :

```sql
-- Record Lock : Verrou sur UNE ligne spécifique
UPDATE users SET status = 'active' WHERE id = 1;
-- 🔒 Seule la ligne id=1 est verrouillée

-- Gap Lock : Verrou sur l'espace ENTRE les lignes (REPEATABLE READ)
SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;
-- 🔒 Empêche l'insertion de nouvelles lignes avec age 20-30

-- Next-Key Lock : Record Lock + Gap Lock
-- Combinaison des deux pour éviter les phantom reads
```

**2. Table-Level Locks** :

```sql
-- Verrou explicite sur toute la table
LOCK TABLES produits WRITE;
-- 🔒 Personne ne peut lire ni écrire
UPDATE produits SET prix = prix * 1.1;
UNLOCK TABLES;

-- Verrou de lecture
LOCK TABLES produits READ;
-- ✅ Autres peuvent lire
-- ⏳ Personne ne peut écrire
```

⚠️ **À éviter en production** : `LOCK TABLES` bloque toute concurrence. Préférer les transactions avec FOR UPDATE.

### 4.2 Modes de Verrous

**Shared Lock (S)** : Lecture partagée

```sql
SELECT * FROM produits WHERE id = 1 FOR SHARE;
-- ✅ Autres peuvent aussi lire (FOR SHARE)
-- ⏳ Personne ne peut écrire
```

**Exclusive Lock (X)** : Écriture exclusive

```sql
SELECT * FROM produits WHERE id = 1 FOR UPDATE;
-- ⏳ Personne ne peut lire (FOR SHARE/UPDATE) ni écrire
```

**Matrice de compatibilité** :

```
          | Shared (S) | Exclusive (X)
----------|------------|---------------
Shared    |     ✅     |      ⏳
Exclusive |     ⏳     |      ⏳
```

### 4.3 Verrous Implicites

InnoDB pose des verrous automatiquement :

```sql
-- UPDATE pose automatiquement un X-lock
UPDATE produits SET stock = stock - 1 WHERE id = 42;
-- Équivalent à :
-- SELECT ... FOR UPDATE
-- UPDATE ...

-- INSERT pose un X-lock sur la nouvelle ligne
INSERT INTO commandes VALUES (...);

-- DELETE pose un X-lock
DELETE FROM logs WHERE date < '2024-01-01';
```

### 4.4 Verrous d'Intention (Intent Locks)

Pour optimiser les verrous de table, InnoDB utilise des "intention locks" :

```sql
-- Transaction verrouille quelques lignes
SELECT * FROM produits WHERE categorie = 'Électronique' FOR UPDATE;
-- InnoDB pose :
-- - IX (Intention Exclusive) sur la TABLE
-- - X (Exclusive) sur les LIGNES matchées

-- Avantage : évite de scanner toutes les lignes pour savoir
-- si un LOCK TABLE serait compatible
```

**Types** :
- **IS** (Intention Shared) : Intention de poser S-locks sur des lignes
- **IX** (Intention Exclusive) : Intention de poser X-locks sur des lignes

### 4.5 Monitoring des Verrous

**Voir les verrous en cours** :

```sql
-- Transactions actives
SELECT * FROM information_schema.INNODB_TRX;

-- Verrous posés
SELECT * FROM information_schema.INNODB_LOCKS;

-- Attentes de verrous
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

**Identifier les blocages** :

```sql
-- Qui bloque qui ?
SELECT
    r.trx_id AS waiting_trx,
    r.trx_mysql_thread_id AS waiting_thread,
    b.trx_id AS blocking_trx,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id;
```

💡 **En production** : Configurer des alertes si des transactions attendent >10s.

---

## 5. Deadlocks : Détection et Résolution

Un **deadlock** (interblocage) survient quand deux transactions s'attendent mutuellement, créant un cycle impossible à résoudre.

### 5.1 Exemple Classique de Deadlock

```sql
-- Transaction A
START TRANSACTION;
UPDATE comptes SET solde = solde - 100 WHERE id = 1;  -- 🔒 Lock ligne 1
-- Attend 1 seconde...
UPDATE comptes SET solde = solde + 100 WHERE id = 2;  -- ⏳ Attend lock ligne 2

-- Transaction B (en parallèle)
START TRANSACTION;
UPDATE comptes SET solde = solde - 50 WHERE id = 2;   -- 🔒 Lock ligne 2
-- Attend 1 seconde...
UPDATE comptes SET solde = solde + 50 WHERE id = 1;   -- ⏳ Attend lock ligne 1

-- 💥 DEADLOCK !
-- A attend que B libère ligne 2
-- B attend que A libère ligne 1
-- → Cycle infini
```

**Détection par InnoDB** :

```
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

InnoDB détecte automatiquement le deadlock et **annule une des transactions** (rollback automatique de la plus petite).

### 5.2 Comment InnoDB Détecte les Deadlocks

**Algorithme de détection** :
1. InnoDB maintient un graphe d'attente (wait-for graph)
2. Périodiquement, il cherche des cycles dans ce graphe
3. Si cycle détecté → choisit une "victime" (plus petit coût de rollback)
4. Rollback automatique de la victime
5. Retourne une erreur 1213 à l'application

**Visualisation** :

```
A → attend B
B → attend A
= Cycle = Deadlock
```

### 5.3 Analyse des Deadlocks

**Voir le dernier deadlock** :

```sql
SHOW ENGINE INNODB STATUS\G

-- Chercher la section :
------------------------
LATEST DETECTED DEADLOCK
------------------------
2025-12-12 14:30:45
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 10, OS thread handle 140234, query id 100 localhost root updating
UPDATE comptes SET solde = solde + 100 WHERE id = 2

*** (1) WAITING FOR THIS LOCK:
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `banque`.`comptes`
trx id 12345 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
...
*** WE ROLL BACK TRANSACTION (1)
```

💡 **Conseil** : Activer le logging des deadlocks :

```ini
[mysqld]
innodb_print_all_deadlocks = 1  # Log TOUS les deadlocks dans error log
```

### 5.4 Prévention des Deadlocks

**1. Ordre cohérent des accès** :

```sql
-- ❌ MAUVAIS : Ordre aléatoire
-- Transaction A : UPDATE ligne 1 puis ligne 2
-- Transaction B : UPDATE ligne 2 puis ligne 1
-- → Risque de deadlock

-- ✅ BON : Ordre déterministe
-- TOUJOURS accéder aux lignes dans le même ordre (ex: par ID croissant)
START TRANSACTION;
UPDATE comptes SET solde = solde - 100 WHERE id = 1;  -- Toujours 1 d'abord
UPDATE comptes SET solde = solde + 100 WHERE id = 2;  -- Puis 2
COMMIT;
```

**2. Transactions courtes** :

```sql
-- ❌ MAUVAIS : Transaction longue
START TRANSACTION;
SELECT ... FOR UPDATE;
-- Appel API externe (500ms)
-- Calculs complexes (2s)
UPDATE ...;
COMMIT;  -- Verrous gardés pendant 2.5s

-- ✅ BON : Transaction minimale
-- Faire les calculs AVANT la transaction
SELECT ... INTO @data;  -- Lecture sans verrou
-- Calculs ici (en dehors de la transaction)
START TRANSACTION;
UPDATE ... WHERE ... FOR UPDATE;  -- Verrou + update rapide
COMMIT;  -- Verrous gardés <50ms
```

**3. Utiliser des niveaux d'isolation appropriés** :

```sql
-- Si possible, utiliser READ COMMITTED plutôt que SERIALIZABLE
-- Moins de verrous = moins de deadlocks
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**4. Détecter et retry automatiquement** :

```python
# Exemple Python avec retry automatique
def transfert_avec_retry(compte_a, compte_b, montant, max_retries=3):
    for attempt in range(max_retries):
        try:
            with connection.cursor() as cursor:
                cursor.execute("START TRANSACTION")
                cursor.execute(
                    "UPDATE comptes SET solde = solde - %s WHERE id = %s",
                    (montant, compte_a)
                )
                cursor.execute(
                    "UPDATE comptes SET solde = solde + %s WHERE id = %s",
                    (montant, compte_b)
                )
                cursor.execute("COMMIT")
                return True  # Succès
        except pymysql.err.OperationalError as e:
            if e.args[0] == 1213:  # Deadlock
                if attempt < max_retries - 1:
                    time.sleep(0.1 * (attempt + 1))  # Backoff exponentiel
                    continue
                else:
                    raise  # Échec après max_retries
            else:
                raise
```

### 5.5 Configuration Anti-Deadlock

```ini
[mysqld]
# Timeout d'attente de verrou (défaut : 50s)
innodb_lock_wait_timeout = 10  # Échouer après 10s au lieu de 50s

# Log tous les deadlocks
innodb_print_all_deadlocks = 1

# Taille du buffer pour détecter les deadlocks
innodb_deadlock_detect = ON  # Active la détection (default)
```

---

## 6. Transactions Distribuées (XA)

Les transactions **XA** (eXtended Architecture) permettent de coordonner des transactions sur **plusieurs bases de données** ou systèmes, garantissant l'atomicité globale.

### 6.1 Principe du Two-Phase Commit (2PC)

**Problème** : Comment garantir qu'une transaction réussit sur 2 bases ou échoue sur les 2 ?

**Solution : Protocole 2PC** :

```
Phase 1: PREPARE
  Coordinateur → Base A : "Peux-tu commiter ?"
  Coordinateur → Base B : "Peux-tu commiter ?"
  Bases répondent : "OUI" ou "NON"

Phase 2: COMMIT ou ROLLBACK
  Si TOUS ont dit OUI :
    Coordinateur → COMMIT sur toutes les bases
  Sinon :
    Coordinateur → ROLLBACK sur toutes les bases
```

### 6.2 Syntaxe XA dans MariaDB

**Exemple : Transaction distribuée entre 2 connexions** :

```sql
-- Connexion 1 : Base A
XA START 'trx123';
UPDATE comptes SET solde = solde - 100 WHERE id = 1;
XA END 'trx123';
XA PREPARE 'trx123';  -- Phase 1 : OK pour commiter

-- Connexion 2 : Base B (ou autre serveur MariaDB)
XA START 'trx123';
UPDATE comptes SET solde = solde + 100 WHERE id = 1;
XA END 'trx123';
XA PREPARE 'trx123';  -- Phase 1 : OK pour commiter

-- Coordinateur décide :
-- Si les 2 PREPARE ont réussi :
XA COMMIT 'trx123';  -- Sur connexion 1
XA COMMIT 'trx123';  -- Sur connexion 2
-- ✅ Transaction distribuée réussie

-- Si une PREPARE échoue :
XA ROLLBACK 'trx123';  -- Sur les deux connexions
```

### 6.3 États d'une Transaction XA

```
ACTIVE → END → PREPARED → COMMITTED
                      ↓
                   ROLLBACK
```

**Vérifier l'état** :

```sql
-- Lister les transactions XA en préparation
XA RECOVER;
-- Résultat : Liste des XIDs en état PREPARED

-- Relancer un commit différé
XA COMMIT 'xid_retrouve';
```

### 6.4 Cas d'Usage Réels

**1. Transaction multi-bases** :

```python
# Exemple : Facturation (DB MariaDB) + Inventaire (DB PostgreSQL)
import pymysql
import psycopg2

# Connexions
maria_conn = pymysql.connect(...)
pg_conn = psycopg2.connect(...)

xid = "facture_12345"

try:
    # Phase 1 : Opérations
    with maria_conn.cursor() as cur:
        cur.execute(f"XA START '{xid}'")
        cur.execute("INSERT INTO factures (...) VALUES (...)")
        cur.execute(f"XA END '{xid}'")
        cur.execute(f"XA PREPARE '{xid}'")

    with pg_conn.cursor() as cur:
        cur.execute("BEGIN")  # PostgreSQL n'a pas XA natif
        cur.execute("UPDATE stocks SET qty = qty - 1 WHERE ...")
        # Simuler PREPARE (commit différé)

    # Phase 2 : Commit global
    with maria_conn.cursor() as cur:
        cur.execute(f"XA COMMIT '{xid}'")
    pg_conn.commit()

except Exception as e:
    # Phase 2 : Rollback global
    maria_conn.execute(f"XA ROLLBACK '{xid}'")
    pg_conn.rollback()
    raise
```

**2. Intégration avec Message Queue** :

```
Transaction XA :
  1. Débite le compte (MariaDB)
  2. Envoie message "paiement effectué" (RabbitMQ)

→ Garantit que le message n'est envoyé QUE si le débit réussit
```

### 6.5 Limitations et Alternatives

**Limitations XA** :
- ⚠️ **Performance** : 2PC ajoute de la latence (2 round-trips réseau)
- ⚠️ **Blocage** : Si le coordinateur crash après PREPARE, ressources bloquées
- ⚠️ **Complexité** : Gestion d'erreur complexe

**Alternatives modernes** :

**1. Saga Pattern** (transactions compensatoires) :

```
Au lieu de :
  XA: Débit A + Crédit B (atomique)

Faire :
  1. Débit A (COMMIT)
  2. Crédit B (COMMIT)
  3. Si échec en 2 : Compensation → Re-crédite A
```

**2. Event Sourcing** :
- Stocker les événements (débits/crédits) plutôt que l'état
- Cohérence éventuelle

**3. Idempotence** :
- Transactions retriables sans effet de bord
- Plus simple que XA, plus résilient

💡 **Recommandation production** : Éviter XA si possible. Privilégier les patterns modernes (Saga, idempotence, eventual consistency).

---

## 7. Implications en Production

### 7.1 Choix d'Architecture

**Transaction locale (1 base)** :

```sql
-- ✅ Simple, performant, ACID natif
START TRANSACTION;
UPDATE compte SET solde = solde - 100 WHERE id = 1;
UPDATE compte SET solde = solde + 100 WHERE id = 2;
COMMIT;
```

**Microservices (bases séparées)** :

```
Options :
1. XA → Complexe, lent
2. Saga → Éventuellement cohérent, complexité métier
3. Event sourcing → Apprentissage élevé

Recommandation : Commencer simple (base partagée ou saga)
```

### 7.2 Monitoring en Production

**Métriques clés** :

```sql
-- Transactions actives
SELECT COUNT(*) FROM information_schema.INNODB_TRX;

-- Transactions longues (> 10 min)
SELECT
    trx_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_query
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 10 MINUTE;

-- Deadlocks (via status)
SHOW ENGINE INNODB STATUS\G
-- Chercher "Deadlock"

-- Attentes de verrous
SELECT COUNT(*) FROM information_schema.INNODB_LOCK_WAITS;
```

**Alertes** :
- 🚨 Transaction active > 5 minutes
- 🚨 Plus de 10 deadlocks/heure
- 🚨 Plus de 50 attentes de verrous simultanées

### 7.3 Tuning de la Concurrence

```ini
[mysqld]
# Pool de threads (plutôt que one-thread-per-connection)
thread_handling = pool-of-threads
thread_pool_size = 16  # Nombre de CPU cores

# Buffer pool : Plus grand = moins de contention sur I/O
innodb_buffer_pool_size = 8G  # 70-80% de la RAM

# Log buffer : Réduit la contention sur redo log
innodb_log_buffer_size = 32M

# Timeout d'attente de verrou
innodb_lock_wait_timeout = 10  # Fail-fast plutôt que bloquer 50s

# Détection de deadlock
innodb_deadlock_detect = ON
innodb_print_all_deadlocks = 1
```

---

## ✅ Points clés à retenir

- **ACID** : Les 4 piliers de la fiabilité (Atomicity, Consistency, Isolation, Durability)
- **REPEATABLE READ** : Niveau d'isolation par défaut d'InnoDB, excellent compromis
- **MVCC** : Permet des lectures sans blocage grâce aux versions multiples
- **Row-level locking** : InnoDB verrouille au niveau ligne, pas table
- **Deadlocks** : Inévitables en environnement concurrent, détectés automatiquement
- **XA** : Transactions distribuées possibles mais complexes, alternatives modernes préférables
- **FOR UPDATE** : Verrou explicite essentiel pour les opérations critiques (stocks, réservations)
- **Transactions courtes** : Minimiser la durée des verrous pour maximiser la concurrence

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Transactions](https://mariadb.com/kb/en/transactions/)
- [📖 InnoDB Transactions](https://mariadb.com/kb/en/innodb-transactions/)
- [📖 Transaction Isolation Levels](https://mariadb.com/kb/en/set-transaction-isolation-level/)
- [📖 XA Transactions](https://mariadb.com/kb/en/xa-transactions/)
- [📖 InnoDB Lock Modes](https://mariadb.com/kb/en/innodb-lock-modes/)

### Articles techniques
- [High Performance MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/) - Chapitres sur les transactions
- [Understanding InnoDB MVCC](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [Distributed Transactions: The Saga Pattern](https://microservices.io/patterns/data/saga.html)

### Outils
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit) - pt-deadlock-logger
- [MySQL Tuner](https://github.com/major/MySQLTuner-perl) - Analyse de la concurrence

---

## ➡️ Section suivante

**6.1 Concept de transaction et propriétés ACID** : Approfondissement détaillé des propriétés ACID avec exemples pratiques et cas limites.

---


⏭️ [Concept de transaction et propriétés ACID](/06-transactions-et-concurrence/01-concept-transaction-acid.md)

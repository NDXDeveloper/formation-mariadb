🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 Niveaux d'Isolation

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures

> **Prérequis** :
> - Section 6.1 (Propriétés ACID, en particulier Isolation)
> - Section 6.2 (Gestion des transactions)
> - Compréhension des accès concurrents

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre les quatre niveaux d'isolation SQL standard
- Identifier les phénomènes de concurrence (dirty read, non-repeatable read, phantom read)
- Choisir le niveau d'isolation approprié selon le cas d'usage
- Comprendre le rôle du MVCC dans l'isolation InnoDB
- Configurer et modifier les niveaux d'isolation
- Anticiper et résoudre les problèmes de concurrence
- Optimiser le compromis isolation/performance en production

---

## Introduction

L'**isolation** est la propriété ACID qui contrôle comment les transactions concurrentes interagissent entre elles. Dans un environnement où des centaines de transactions s'exécutent simultanément, l'isolation détermine :

- **Quelles données** une transaction peut voir
- **Quand** elle peut les voir
- **Comment** les modifications d'autres transactions affectent ses lectures

Sans isolation appropriée, les données peuvent devenir incohérentes, les calculs erronés et les décisions basées sur des informations incorrectes.

### Le Dilemme Fondamental

Il existe un **compromis inévitable** entre deux objectifs opposés :

```
┌─────────────────┐               ┌─────────────────┐
│  COHÉRENCE      │ ←─────×─────→ │  PERFORMANCE   │
│  (Isolation)    │               │  (Concurrence)  │
└─────────────────┘               └─────────────────┘
      ↑                                   ↑
  Lecture sûre                      Lecture rapide
  Mais blocages                     Mais incohérences
```

Les **niveaux d'isolation** sont les différents points d'équilibre possibles sur ce spectre, permettant d'adapter le comportement aux besoins spécifiques de chaque application.

---

## 1. Les Quatre Niveaux d'Isolation SQL

La norme SQL définit **quatre niveaux d'isolation standard**, du plus permissif (plus rapide, moins sûr) au plus strict (plus lent, plus sûr) :

### 1.1 Vue d'Ensemble Rapide

```sql
-- Du plus permissif au plus strict :

1. READ UNCOMMITTED   -- Lit les données non commitées
2. READ COMMITTED     -- Lit seulement les données commitées
3. REPEATABLE READ    -- Garantit des lectures répétables (default InnoDB)
4. SERIALIZABLE       -- Isolation maximale, transactions sérialisées
```

### 1.2 Tableau Comparatif des Phénomènes

| Niveau d'Isolation | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------------------|------------|---------------------|--------------|-------------|
| **READ UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible | ⭐⭐⭐⭐⭐ Maximale |
| **READ COMMITTED** | ❌ Bloqué | ✅ Possible | ✅ Possible | ⭐⭐⭐⭐ Élevée |
| **REPEATABLE READ** | ❌ Bloqué | ❌ Bloqué | ⚠️ InnoDB: Bloqué* | ⭐⭐⭐ Bonne |
| **SERIALIZABLE** | ❌ Bloqué | ❌ Bloqué | ❌ Bloqué | ⭐⭐ Faible |

\* **Spécificité InnoDB** : Contrairement au standard SQL qui autorise les phantom reads en REPEATABLE READ, InnoDB les empêche également grâce aux **gap locks**.

### 1.3 Adoption par les SGBD

Différents SGBD ont fait des choix différents pour leur niveau par défaut :

| SGBD | Niveau par Défaut | Raison |
|------|-------------------|--------|
| **MariaDB/MySQL InnoDB** | REPEATABLE READ | Cohérence élevée, support réplication |
| **PostgreSQL** | READ COMMITTED | Compromis performance/cohérence |
| **Oracle** | READ COMMITTED | Idem PostgreSQL |
| **SQL Server** | READ COMMITTED | Standard industrie |
| **SQLite** | SERIALIZABLE | Base embarquée, simplicité |

🆕 **MariaDB 11.8** : Optimisations du MVCC pour améliorer les performances en REPEATABLE READ, réduisant l'écart avec READ COMMITTED.

---

## 2. Les Trois Phénomènes de Concurrence

Pour comprendre les niveaux d'isolation, il faut d'abord comprendre les **problèmes qu'ils préviennent**. La norme SQL définit trois phénomènes indésirables :

### 2.1 Dirty Read (Lecture Sale)

**Définition** : Lire des données **non commitées** écrites par une autre transaction.

**Problème** : Si l'autre transaction fait un ROLLBACK, les données lues étaient invalides.

```sql
┌─────────────────────────────────────────────────────────────┐
│ Timeline : Dirty Read                                       │
├─────────────────────────────────────────────────────────────┤
│ Transaction A                  Transaction B                │
├─────────────────────────────────────────────────────────────┤
│ START TRANSACTION;                                          │
│                                                             │
│ UPDATE produits                                             │
│ SET prix = 99.99                                            │
│ WHERE id = 1;                                               │
│ -- Pas de COMMIT                │                           │
│                                 │ START TRANSACTION;        │
│                                 │                           │
│                                 │ SELECT prix               │
│                                 │ FROM produits             │
│                                 │ WHERE id = 1;             │
│                                 │ -- Lit : 99.99 ❌         │
│                                 │ -- (donnée sale)          │
│                                 │                           │
│ ROLLBACK;                       │                           │
│ -- Annule la modification       │                           │
│                                 │                           │
│                                 │ -- B a lu une donnée      │
│                                 │ -- qui n'existe plus      │
│                                 │ COMMIT;                   │
└─────────────────────────────────────────────────────────────┘
```

**Impact en production** :
- 💥 Décisions basées sur des données fantômes
- 💥 Rapports avec chiffres incorrects
- 💥 Calculs de prix erronés en e-commerce

**Exemple réel** :

```sql
-- Transaction A : Ajustement de prix temporaire
START TRANSACTION;
UPDATE produits SET prix = 0.01 WHERE id = 42;  -- Test temporaire
-- Développeur va vérifier quelque chose...

-- Transaction B : Calcul de statistiques
-- (avec READ UNCOMMITTED)
SELECT AVG(prix) FROM produits;
-- 💥 Inclut le prix de 0.01€, résultat faussé !

-- Transaction A
ROLLBACK;  -- Finalement on annule le test
```

### 2.2 Non-Repeatable Read (Lecture Non Répétable)

**Définition** : Relire les mêmes données dans la même transaction et obtenir un **résultat différent**.

**Problème** : La cohérence est rompue au sein d'une même transaction.

```sql
┌─────────────────────────────────────────────────────────────┐
│ Timeline : Non-Repeatable Read                              │
├─────────────────────────────────────────────────────────────┤
│ Transaction A                  Transaction B                │
├─────────────────────────────────────────────────────────────┤
│ START TRANSACTION;                                          │
│                                                             │
│ SELECT prix                                                 │
│ FROM produits                                               │
│ WHERE id = 1;                                               │
│ -- Lit : 50.00 €                │                           │
│                                 │                           │
│                                 │ START TRANSACTION;        │
│                                 │                           │
│                                 │ UPDATE produits           │
│                                 │ SET prix = 75.00          │
│                                 │ WHERE id = 1;             │
│                                 │                           │
│                                 │ COMMIT;                   │
│                                 │                           │
│ -- Quelques secondes plus tard...                           │
│                                                             │
│ SELECT prix                                                 │
│ FROM produits                                               │
│ WHERE id = 1;                                               │
│ -- Lit : 75.00 € ❌                                         │
│ -- Changé entre 2 lectures !    │                           │
│                                                             │
│ COMMIT;                                                     │
└─────────────────────────────────────────────────────────────┘
```

**Impact en production** :
- 💥 Rapports financiers incohérents
- 💥 Calculs de sommes qui ne correspondent pas
- 💥 Décisions basées sur des données qui ont changé

**Exemple réel : E-commerce**

```sql
-- Transaction A : Client consulte son panier
START TRANSACTION;

SELECT SUM(prix * quantite) AS total
FROM panier
WHERE client_id = 100;
-- Lit : 150.00 €

-- Transaction B : Admin applique une promotion
UPDATE produits SET prix = prix * 0.9 WHERE categorie = 'Électronique';
COMMIT;

-- Transaction A : Client valide la commande
SELECT SUM(prix * quantite) AS total
FROM panier
WHERE client_id = 100;
-- Lit : 135.00 € ❌ Le total a changé !

-- Le client voit un montant différent au moment du paiement
COMMIT;
```

### 2.3 Phantom Read (Lecture Fantôme)

**Définition** : De nouvelles lignes **apparaissent ou disparaissent** entre deux lectures identiques.

**Problème** : Les agrégations (COUNT, SUM, AVG) deviennent incohérentes.

```sql
┌──────────────────────────────────────────────────────────────┐
│ Timeline : Phantom Read                                      │
├──────────────────────────────────────────────────────────────┤
│ Transaction A                  Transaction B                 │
├──────────────────────────────────────────────────────────────┤
│ START TRANSACTION;                                           │
│                                                              │
│ SELECT COUNT(*)                                              │
│ FROM commandes                                               │
│ WHERE client_id = 5;                                         │
│ -- Compte : 10 commandes        │                            │
│                                 │                            │
│                                 │ START TRANSACTION;         │
│                                 │                            │
│                                 │ INSERT INTO commandes      │
│                                 │ (client_id, montant)       │
│                                 │ VALUES (5, 100.00);        │
│                                 │                            │
│                                 │ COMMIT;                    │
│                                 │                            │
│ SELECT COUNT(*)                                              │
│ FROM commandes                                               │
│ WHERE client_id = 5;                                         │
│ -- Compte : 11 commandes ❌                                  │
│ -- Une "fantôme" est apparue !  │                            │
│                                                              │
│ COMMIT;                                                      │
└──────────────────────────────────────────────────────────────┘
```

**Impact en production** :
- 💥 Statistiques qui changent en cours de calcul
- 💥 Audits financiers incohérents
- 💥 Pagination avec des résultats qui sautent

**Exemple réel : Dashboard analytique**

```sql
-- Transaction A : Génération de rapport mensuel
START TRANSACTION;

-- Calcul 1 : Nombre de commandes
SELECT @nb_commandes := COUNT(*)
FROM commandes
WHERE MONTH(created_at) = 12;
-- Résultat : 1000

-- Transaction B : Une nouvelle commande arrive
INSERT INTO commandes (created_at, montant) VALUES (NOW(), 50.00);
COMMIT;

-- Transaction A : Calcul 2 : Montant moyen
SELECT @montant_moyen := AVG(montant)
FROM commandes
WHERE MONTH(created_at) = 12;
-- 💥 La moyenne inclut maintenant 1001 commandes, pas 1000 !
-- Incohérence entre les deux calculs

COMMIT;
```

---

## 3. Configuration des Niveaux d'Isolation

### 3.1 Syntaxe Complète

MariaDB permet de définir le niveau d'isolation à **trois portées** différentes :

```sql
-- 1. GLOBAL : Toutes les futures sessions
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 2. SESSION : La connexion actuelle, toutes les futures transactions
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 3. TRANSACTION : Uniquement la prochaine transaction
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
-- ...
COMMIT;
-- Retour au niveau SESSION après le COMMIT
```

### 3.2 Vérifier le Niveau Actuel

```sql
-- Niveau global (serveur)
SELECT @@GLOBAL.transaction_isolation;
-- ou (ancienne syntaxe MySQL)
SELECT @@GLOBAL.tx_isolation;

-- Niveau session (connexion actuelle)
SELECT @@SESSION.transaction_isolation;
-- Raccourci :
SELECT @@transaction_isolation;

-- Exemple de résultat :
-- 'REPEATABLE-READ'
```

### 3.3 Configuration dans my.cnf

```ini
[mysqld]
# Niveau d'isolation par défaut
transaction-isolation = REPEATABLE-READ

# Alternatives :
# transaction-isolation = READ-UNCOMMITTED
# transaction-isolation = READ-COMMITTED
# transaction-isolation = SERIALIZABLE
```

### 3.4 Vérification dans les Logs

```sql
-- Activer le logging pour debug
SET GLOBAL general_log = ON;
SET GLOBAL log_output = 'TABLE';

-- Changer le niveau
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Vérifier dans les logs
SELECT * FROM mysql.general_log
WHERE argument LIKE '%ISOLATION LEVEL%'
ORDER BY event_time DESC
LIMIT 5;
```

---

## 4. MVCC : Le Secret des Performances d'InnoDB

### 4.1 Multi-Version Concurrency Control

Le **MVCC** (Multi-Version Concurrency Control) est la technologie fondamentale qui permet à InnoDB d'offrir de bonnes performances même avec REPEATABLE READ.

**Principe** : Au lieu de bloquer les lecteurs, InnoDB maintient **plusieurs versions** des données et donne à chaque transaction la version appropriée.

```
Ancien modèle (MyISAM) :
┌────────────────────────────────────┐
│ Writer locks data                  │
│        ↓                           │
│ Readers WAIT (blocked)             │
│        ↓                           │
│ Performance ❌                     │
└────────────────────────────────────┘

Modèle MVCC (InnoDB) :
┌────────────────────────────────────┐
│ Writer creates new version         │
│        ↓                           │
│ Readers see old version            │
│        ↓                           │
│ NO BLOCKING ✅                     │
│        ↓                           │
│ Performance ⭐⭐⭐⭐⭐            │
└────────────────────────────────────┘
```

### 4.2 Comment Fonctionne le MVCC

Chaque ligne dans InnoDB possède des **métadonnées cachées** :

```
Structure d'une ligne InnoDB :
┌──────────────────────────────────────────────────────────┐
│ Colonnes visibles │ DB_TRX_ID │ DB_ROLL_PTR │ DB_ROW_ID  │
│ (nom, prix, etc.) │  (6 bytes)│   (7 bytes) │ (6 bytes)  │
├──────────────────────────────────────────────────────────┤
│ Données utilisateur│ ID trans  │ Pointeur    │ ID ligne  │
│                    │ créatrice │ vers undo   │ interne   │
└──────────────────────────────────────────────────────────┘
```

**Champs cachés** :
- **DB_TRX_ID** : ID de la transaction qui a créé/modifié cette version
- **DB_ROLL_PTR** : Pointeur vers les anciennes versions (undo log)
- **DB_ROW_ID** : ID de ligne interne (si pas de PK)

### 4.3 Exemple Détaillé du MVCC

```sql
-- État initial : Transaction 100 crée une ligne
INSERT INTO users (id, name, age) VALUES (1, 'Alice', 25);
COMMIT;

-- État de la ligne :
┌────────────────────────────────────────┐
│ id=1 │ name='Alice' │ age=25           │
│ DB_TRX_ID = 100                        │
│ DB_ROLL_PTR = NULL                     │
└────────────────────────────────────────┘

-- Transaction 150 modifie
START TRANSACTION;  -- TRX_ID = 150
UPDATE users SET age = 26 WHERE id = 1;

-- Nouvelle version créée :
┌────────────────────────────────────────┐
│ id=1 │ name='Alice' │ age=26           │
│ DB_TRX_ID = 150                        │
│ DB_ROLL_PTR = → [Undo: age=25, TRX=100]│
└────────────────────────────────────────┘

-- Transaction 140 (démarrée AVANT 150) lit :
-- (REPEATABLE READ)
START TRANSACTION;  -- TRX_ID = 140
SELECT age FROM users WHERE id = 1;
-- Résultat : 25 ✅
-- InnoDB voit : TRX=140 < TRX=150
-- → Utilise la version depuis l'undo log

-- Transaction 160 (démarrée APRÈS 150 commitée) lit :
START TRANSACTION;  -- TRX_ID = 160
SELECT age FROM users WHERE id = 1;
-- Résultat : 26 ✅
-- InnoDB voit : TRX=160 > TRX=150 (commitée)
-- → Utilise la version actuelle
```

### 4.4 Read View (Snapshot Isolation)

En **REPEATABLE READ**, InnoDB crée un **read view** (snapshot) au début de la transaction :

```sql
START TRANSACTION;  -- ← Read view créé ICI
-- Snapshot : Liste des transactions actives à cet instant

-- Ce snapshot détermine quelles versions de données cette transaction voit
-- Règles :
-- 1. Voir les transactions commitées AVANT le snapshot
-- 2. Ne PAS voir les transactions actives au moment du snapshot
-- 3. Ne PAS voir les transactions démarrées APRÈS le snapshot

SELECT * FROM produits WHERE id = 1;
-- Voit toujours le snapshot de l'instant du START TRANSACTION

-- Même 10 minutes plus tard...
SELECT * FROM produits WHERE id = 1;
-- ✅ Toujours le même snapshot

COMMIT;
```

**Comparaison avec READ COMMITTED** :

```sql
-- En READ COMMITTED : Pas de snapshot fixe
START TRANSACTION;

SELECT price FROM produits WHERE id = 1;  -- Lit : 50€
-- Read view créé ici, pour CETTE requête

-- Autre transaction commit un changement
-- UPDATE produits SET price = 75 WHERE id = 1; COMMIT;

SELECT price FROM produits WHERE id = 1;  -- Lit : 75€ ❌
-- Nouveau read view créé, voit le changement

COMMIT;
```

### 4.5 Coût du MVCC

Le MVCC n'est pas gratuit :

```sql
-- Problème : Transactions très longues
START TRANSACTION;  -- T0

SELECT * FROM large_table;  -- 10 millions de lignes

-- 2 heures plus tard, toujours en transaction...
-- Pendant ce temps, des milliers de UPDATE ont eu lieu

SELECT * FROM large_table;
-- ✅ Voit toujours l'état à T0
-- ⚠️ Mais l'undo log est ÉNORME (toutes les versions depuis T0)
-- 💥 Performance dégradée, espace disque consommé

COMMIT;
```

**Monitoring des transactions longues** :

```sql
-- Identifier les transactions qui consomment l'undo log
SELECT
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_rows_locked,
    trx_rows_modified
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 10 MINUTE
ORDER BY duration_sec DESC;
```

---

## 5. Choisir le Bon Niveau d'Isolation

### 5.1 Arbre de Décision

```
┌─────────────────────────────────────────────────┐
│ Ai-je besoin de cohérence ABSOLUE ?             │
│   (audit, finance, réglementaire)               │
│   OUI → SERIALIZABLE                            │
│   NON ↓                                         │
├─────────────────────────────────────────────────┤
│ Mes lectures doivent-elles être RÉPÉTABLES ?    │
│   (rapports, calculs multi-étapes)              │
│   OUI → REPEATABLE READ (default) ✅            │
│   NON ↓                                         │
├─────────────────────────────────────────────────┤
│ Puis-je tolérer des lectures non-commitées ?    │
│   (dashboards temps réel, stats approximatives) │
│   OUI → READ UNCOMMITTED ⚠️                     │
│   NON → READ COMMITTED                          │
└─────────────────────────────────────────────────┘
```

### 5.2 Guide par Type d'Application

| Type Application | Niveau Recommandé | Justification |
|-----------------|-------------------|---------------|
| **E-commerce (panier)** | REPEATABLE READ | Cohérence stock + performance |
| **E-commerce (paiement)** | SERIALIZABLE | Aucune erreur acceptable |
| **CMS / Blog** | READ COMMITTED | Compromis raisonnable |
| **Dashboard analytics** | READ UNCOMMITTED | Performance > précision absolue |
| **ERP / Comptabilité** | REPEATABLE READ | Cohérence des rapports |
| **Système bancaire** | SERIALIZABLE | Réglementaire, audit |
| **Réservations** | REPEATABLE READ | Éviter double-booking |
| **Chat / Logs** | READ COMMITTED | Performance |
| **IoT / Metrics** | READ UNCOMMITTED | Volume élevé, approx OK |

### 5.3 Compromis Performance vs Cohérence

**Benchmark indicatif** (transactions/seconde) :

```
Environnement : 100 connexions concurrentes, mix lecture/écriture

READ UNCOMMITTED  : ~15,000 TPS ⭐⭐⭐⭐⭐
READ COMMITTED    : ~12,000 TPS ⭐⭐⭐⭐
REPEATABLE READ   :  ~8,000 TPS ⭐⭐⭐
SERIALIZABLE      :  ~2,000 TPS ⭐⭐

Note : Chiffres indicatifs, varient selon le workload
```

**Impact de la concurrence** :

```sql
-- READ UNCOMMITTED : Aucun blocage
-- 100 lecteurs + 100 écrivains = max concurrence

-- READ COMMITTED : Blocage minimal
-- Lecteurs ne bloquent jamais
-- Écrivains bloquent autres écrivains sur même ligne

-- REPEATABLE READ : Blocage modéré
-- Gap locks peuvent bloquer les INSERT
-- Verrous gardés plus longtemps

-- SERIALIZABLE : Blocage élevé
-- SELECT pose des verrous partagés
-- Forte contention
```

### 5.4 Stratégies Mixtes

Utiliser **différents niveaux selon l'opération** :

```sql
-- Opération critique : paiement
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
-- Traitement du paiement avec garanties maximales
COMMIT;

-- Opération standard : consultation produits
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM produits WHERE categorie = 'Livres';
COMMIT;

-- Opération analytics : dashboard
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT COUNT(*), AVG(montant) FROM commandes_today;
COMMIT;
```

**Exemple PHP : Adapter selon le contexte**

```php
class DatabaseManager {
    private $pdo;

    public function executeWithIsolation($callback, $level = 'REPEATABLE READ') {
        $this->pdo->exec("SET TRANSACTION ISOLATION LEVEL $level");
        $this->pdo->beginTransaction();

        try {
            $result = $callback($this->pdo);
            $this->pdo->commit();
            return $result;
        } catch (Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }

    public function processCriticalPayment($orderId, $amount) {
        return $this->executeWithIsolation(
            function($pdo) use ($orderId, $amount) {
                // Logique de paiement
            },
            'SERIALIZABLE'  // Cohérence maximale
        );
    }

    public function getDashboardStats() {
        return $this->executeWithIsolation(
            function($pdo) {
                // Requêtes analytics
            },
            'READ UNCOMMITTED'  // Performance maximale
        );
    }
}
```

---

## 6. Impact des Niveaux sur les Verrous

### 6.1 Verrous par Niveau d'Isolation

```sql
-- READ UNCOMMITTED : Aucun verrou de lecture
SELECT * FROM produits WHERE id = 1;
-- ✅ Aucun verrou, lecture "dirty"

-- READ COMMITTED : Verrous de lecture relâchés immédiatement
SELECT * FROM produits WHERE id = 1;
-- 🔒 Shared lock posé le temps de la lecture
-- ✅ Libéré immédiatement après

-- REPEATABLE READ : Verrous de lecture gardés
SELECT * FROM produits WHERE id = 1;
-- 🔒 Shared lock gardé jusqu'au COMMIT
-- + Gap locks pour empêcher phantom reads

-- SERIALIZABLE : Tous les SELECT = SELECT FOR SHARE
SELECT * FROM produits WHERE id = 1;
-- 🔒 Shared lock automatique
-- 🔒 Gap locks automatiques
-- Bloque tous les écrivains
```

### 6.2 Gap Locks et Phantom Reads

InnoDB utilise les **gap locks** en REPEATABLE READ pour empêcher les phantom reads :

```sql
-- REPEATABLE READ
START TRANSACTION;

-- Lit les produits entre 10€ et 50€
SELECT * FROM produits
WHERE prix BETWEEN 10 AND 50;

-- InnoDB pose :
-- 1. Record locks sur les lignes matchées
-- 2. Gap locks sur les "trous" (10-50)

-- Autre transaction tente d'insérer
INSERT INTO produits (nom, prix) VALUES ('Widget', 30);
-- ⏳ BLOQUÉ par le gap lock
-- Empêche le phantom read

COMMIT;  -- Libère tous les verrous
```

**Coût des gap locks** :

```sql
-- Scénario : Table avec gaps importants
-- ID: 1, 100, 200, 300, 400, 500

START TRANSACTION;
SELECT * FROM produits WHERE id BETWEEN 150 AND 250;
-- Gap lock sur [100, 300] (large plage !)

-- Autre transaction
INSERT INTO produits (id, nom) VALUES (120, 'Test');
-- ⏳ BLOQUÉ même si 120 n'est pas dans [150, 250]
```

💡 **Optimisation** : Utiliser des clés séquentielles (AUTO_INCREMENT) pour minimiser les gaps.

---

## 7. Cas d'Usage Concrets

### 7.1 E-commerce : Gestion de Stock

```sql
-- ❌ MAUVAIS : READ COMMITTED (race condition possible)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Session A
START TRANSACTION;
SELECT @stock := stock FROM produits WHERE id = 42;
-- Lit : stock = 1

-- Session B (parallèle)
START TRANSACTION;
SELECT @stock := stock FROM produits WHERE id = 42;
-- Lit : stock = 1 (même valeur)

-- Session A
IF @stock > 0 THEN
    UPDATE produits SET stock = stock - 1 WHERE id = 42;
    INSERT INTO commandes (...) VALUES (...);
END IF;
COMMIT;

-- Session B
IF @stock > 0 THEN
    UPDATE produits SET stock = stock - 1 WHERE id = 42;
    -- 💥 Stock devient -1 !
    INSERT INTO commandes (...) VALUES (...);
END IF;
COMMIT;

-- ✅ BON : REPEATABLE READ + FOR UPDATE
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

START TRANSACTION;
SELECT @stock := stock
FROM produits
WHERE id = 42
FOR UPDATE;  -- 🔒 Verrou exclusif

IF @stock > 0 THEN
    UPDATE produits SET stock = stock - 1 WHERE id = 42;
    INSERT INTO commandes (...) VALUES (...);
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

### 7.2 Finance : Transfert Bancaire

```sql
-- ✅ Utiliser SERIALIZABLE pour garanties maximales
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

START TRANSACTION;

-- Lire et verrouiller les comptes (ordre déterministe)
SELECT @solde_a := solde FROM comptes WHERE id = 1 FOR UPDATE;
SELECT @solde_b := solde FROM comptes WHERE id = 2 FOR UPDATE;

-- Vérifications
IF @solde_a < 1000 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Solde insuffisant';
END IF;

-- Transfert
UPDATE comptes SET solde = solde - 1000 WHERE id = 1;
UPDATE comptes SET solde = solde + 1000 WHERE id = 2;

-- Audit
INSERT INTO audit_log (type, details)
VALUES ('transfert', CONCAT('De ', 1, ' vers ', 2, ': 1000€'));

COMMIT;
-- ✅ Isolation maximale : aucune interférence possible
```

### 7.3 Analytics : Dashboard Temps Réel

```sql
-- ✅ READ UNCOMMITTED : Performance maximale, précision non critique
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

START TRANSACTION;

-- Statistiques approximatives
SELECT
    COUNT(*) AS commandes_jour,
    SUM(montant) AS ca_jour,
    AVG(montant) AS panier_moyen
FROM commandes
WHERE DATE(created_at) = CURDATE();

-- Données potentiellement "sales" mais :
-- - Très rapides (pas de verrous)
-- - Différence négligeable sur gros volumes
-- - Mises à jour toutes les 30s, précision absolue inutile

COMMIT;
```

---

## 8. Implications en Production

### 8.1 Monitoring des Niveaux d'Isolation

```sql
-- Vérifier les niveaux utilisés par les sessions
SELECT
    PROCESSLIST_ID AS session_id,
    PROCESSLIST_USER AS user,
    PROCESSLIST_HOST AS host,
    PROCESSLIST_DB AS db,
    PROCESSLIST_COMMAND AS command,
    PROCESSLIST_TIME AS duration_sec,
    @@SESSION.transaction_isolation AS isolation_level
FROM performance_schema.threads
WHERE PROCESSLIST_ID IS NOT NULL;

-- Statistiques sur les niveaux utilisés
SELECT
    isolation_level,
    COUNT(*) AS nb_sessions
FROM (
    SELECT @@SESSION.transaction_isolation AS isolation_level
    FROM performance_schema.threads
    WHERE PROCESSLIST_ID IS NOT NULL
) AS t
GROUP BY isolation_level;
```

### 8.2 Détection de Problèmes de Concurrence

```sql
-- Identifier les transactions qui causent des blocages
SELECT
    r.trx_id AS waiting_transaction,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_transaction,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    TIMESTAMPDIFF(SECOND, r.trx_started, NOW()) AS wait_duration_sec
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
ORDER BY wait_duration_sec DESC;
```

### 8.3 Alerting Recommandé

**Métriques à surveiller** :

```sql
-- 1. Transactions longues (> 10 minutes)
SELECT COUNT(*) AS long_transactions
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 10 MINUTE;
-- Alerte si > 5

-- 2. Lock waits élevés
SELECT COUNT(*) AS lock_waits
FROM information_schema.INNODB_LOCK_WAITS;
-- Alerte si > 50

-- 3. Undo log size (indicateur de transactions longues)
SHOW ENGINE INNODB STATUS\G
-- Chercher "History list length"
-- Alerte si > 10000
```

### 8.4 Configuration Production Recommandée

```ini
[mysqld]
# Niveau d'isolation par défaut
# Garder REPEATABLE READ pour la majorité des applications
transaction-isolation = REPEATABLE-READ

# Timeout pour lock wait (éviter blocages infinis)
innodb_lock_wait_timeout = 50  # secondes

# Détection de deadlock (toujours actif)
innodb_deadlock_detect = ON

# Log tous les deadlocks dans error log
innodb_print_all_deadlocks = 1

# Purge des anciennes versions (MVCC cleanup)
innodb_purge_threads = 4  # Augmenter si beaucoup de UPDATE/DELETE
innodb_purge_batch_size = 300
```

---

## ✅ Points clés à retenir

- **Quatre niveaux** : READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE
- **Default InnoDB** : REPEATABLE READ (bon compromis cohérence/performance)
- **Trois phénomènes** : Dirty read, non-repeatable read, phantom read
- **MVCC** : Technologie clé permettant lectures sans blocage
- **Gap locks** : Spécificité InnoDB, empêche phantom reads en REPEATABLE READ
- **Compromis** : Plus d'isolation = plus de cohérence mais moins de performance
- **Configuration flexible** : GLOBAL, SESSION ou TRANSACTION
- **Stratégie mixte** : Adapter le niveau selon l'opération (paiement vs analytics)
- **Monitoring** : Surveiller transactions longues et lock waits
- **FOR UPDATE** : Combiné avec REPEATABLE READ pour cohérence garantie

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 SET TRANSACTION ISOLATION LEVEL](https://mariadb.com/kb/en/set-transaction-isolation-level/)
- [📖 InnoDB Transaction Isolation](https://mariadb.com/kb/en/innodb-transaction-isolation-levels/)
- [📖 InnoDB Locking](https://mariadb.com/kb/en/innodb-lock-modes/)
- [📖 MVCC Implementation](https://mariadb.com/kb/en/innodb-storage-engine/)

### Articles techniques
- [Understanding InnoDB MVCC](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [Isolation Levels Explained](https://www.postgresql.org/docs/current/transaction-iso.html) (PostgreSQL, mais concepts universels)
- "Designing Data-Intensive Applications" by Martin Kleppmann - Chapitre 7

---

## ➡️ Sections suivantes

Les sections détaillées sur chaque niveau d'isolation :

- **6.3.1** READ UNCOMMITTED : Dirty reads possibles
- **6.3.2** READ COMMITTED : Lectures cohérentes
- **6.3.3** REPEATABLE READ : Default InnoDB
- **6.3.4** SERIALIZABLE : Isolation maximale

---


⏭️ [READ UNCOMMITTED : Dirty reads possibles](/06-transactions-et-concurrence/03.1-read-uncommitted.md)

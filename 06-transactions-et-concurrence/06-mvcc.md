🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 MVCC (Multi-Version Concurrency Control)

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures

> **Prérequis** :
> - Section 6.1 (Propriétés ACID)
> - Section 6.3 (Niveaux d'isolation)
> - Section 6.4 (Verrous)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le principe du MVCC et ses avantages
- Expliquer la structure interne des lignes InnoDB
- Comprendre le rôle de l'undo log dans le versioning
- Distinguer consistent read et locking read
- Analyser l'impact des transactions longues sur le MVCC
- Optimiser les performances liées au MVCC
- Monitorer et tuner le système de purge
- Gérer le history list length en production

---

## Introduction

Le **MVCC** (Multi-Version Concurrency Control) est la technologie fondamentale qui permet à InnoDB d'offrir des performances exceptionnelles dans les environnements concurrents. C'est le secret qui permet à des milliers de transactions de coexister sans se bloquer mutuellement.

### Le Problème Sans MVCC

Historiquement, les SGBD utilisaient un modèle simple mais limitant :

```
Ancien modèle (locks stricts) :
┌──────────────────────────────────────┐
│ Writer acquiert verrou exclusif      │
│        ↓                             │
│ Readers BLOQUÉS (doivent attendre)   │
│        ↓                             │
│ Throughput limité ❌                 │
│ Scalabilité faible ❌                │
└──────────────────────────────────────┘

Exemple : MyISAM
- Table-level locking
- Readers bloquent writers
- Writers bloquent readers
- Performance catastrophique avec concurrence
```

### La Révolution MVCC

```
Modèle MVCC (InnoDB) :
┌───────────────────────────────────────┐
│ Writer modifie → Crée nouvelle version│
│        ↓                              │
│ Readers voient ancienne version       │
│        ↓                              │
│ PAS DE BLOCAGE ✅                     │
│ Throughput élevé ✅                   │
│ Scalabilité excellente ✅             │
└───────────────────────────────────────┘

Principe clé :
Au lieu de bloquer, on maintient plusieurs
versions des données simultanément
```

### Avantages du MVCC

1. **Lectures sans blocage** : Les lecteurs ne bloquent jamais les écrivains
2. **Écritures sans blocage des lecteurs** : Les écrivains ne bloquent pas les lecteurs
3. **Isolation sans coût** : REPEATABLE READ sans impact majeur sur la performance
4. **Scalabilité** : Performance linéaire avec le nombre de cores

---

## 1. Principe Fondamental du MVCC

### 1.1 La Philosophie "Snapshot"

Chaque transaction voit un **snapshot cohérent** de la base de données à un instant T :

```sql
-- Transaction A démarre à T0
START TRANSACTION;  -- Snapshot créé à T0
SELECT * FROM produits;  -- Voit l'état à T0

-- Entre-temps, Transaction B modifie
-- (T1) UPDATE produits SET prix = 100 WHERE id = 1;
-- (T1) COMMIT;

-- Transaction A continue
SELECT * FROM produits;  -- Voit toujours l'état à T0
-- Le UPDATE de B n'est PAS visible

COMMIT;  -- Maintenant on peut voir les changements de B
```

### 1.2 Multi-Version = Plusieurs Vérités Simultanées

À un instant donné, **plusieurs versions** de la même ligne coexistent :

```
Ligne produit id=1 :

Version 1 (TRX_ID=100) : prix = 50€   ← Transaction ancienne voit ceci
Version 2 (TRX_ID=150) : prix = 75€   ← Transaction récente voit ceci
Version 3 (TRX_ID=200) : prix = 100€  ← Transaction future verra ceci

Toutes ces versions existent SIMULTANÉMENT
grâce à l'undo log
```

### 1.3 Exemple Concret

```sql
-- État initial : produit id=1, prix=50€, TRX_ID=100

-- Transaction A (TRX_ID=140) démarre
START TRANSACTION;
SELECT prix FROM produits WHERE id = 1;
-- Lit : 50€ (version TRX=100)

-- Transaction B (TRX_ID=150) modifie
START TRANSACTION;
UPDATE produits SET prix = 75 WHERE id = 1;
COMMIT;
-- Crée nouvelle version (TRX=150)

-- Transaction A continue (toujours actif depuis TRX=140)
SELECT prix FROM produits WHERE id = 1;
-- Lit : 50€ ✅ (toujours l'ancienne version)
-- Transaction A a démarré AVANT TRX=150, donc ne voit pas le changement

COMMIT;

-- Nouvelle transaction C (TRX_ID=160) démarre
START TRANSACTION;
SELECT prix FROM produits WHERE id = 1;
-- Lit : 75€ ✅ (nouvelle version)
COMMIT;
```

---

## 2. Structure Interne des Lignes InnoDB

### 2.1 Les Métadonnées Cachées

Chaque ligne InnoDB contient des **champs système cachés** utilisés par le MVCC :

```
Structure d'une ligne InnoDB :
┌────────────────────────────────────────────────────────────────────┐
│ Données utilisateur          │ Métadonnées système (cachées)       │
├────────────────────────────────────────────────────────────────────┤
│ id  │ nom   │ prix │ stock   │ DB_TRX_ID │ DB_ROLL_PTR │ DB_ROW_ID │
│ 42  │ Widget│ 99.99│ 100     │  (6 bytes)│  (7 bytes)  │ (6 bytes) │
└────────────────────────────────────────────────────────────────────┘
         ↑                              ↑           ↑            ↑
    Visible par                    Invisible pour l'utilisateur
    l'utilisateur                  Utilisé par InnoDB pour MVCC
```

### 2.2 DB_TRX_ID : L'ID de Transaction

**Rôle** : Identifie quelle transaction a créé ou modifié cette version de la ligne.

```sql
-- Transaction 100 crée une ligne
INSERT INTO produits (id, nom, prix) VALUES (1, 'Widget', 50);
COMMIT;

-- État de la ligne :
┌─────────────────────────────────────────┐
│ id=1 │ nom=Widget │ prix=50             │
│ DB_TRX_ID = 100                         │
│ DB_ROLL_PTR = NULL (première version)   │
└─────────────────────────────────────────┘

-- Transaction 150 modifie
UPDATE produits SET prix = 75 WHERE id = 1;
COMMIT;

-- Nouvelle version :
┌─────────────────────────────────────────┐
│ id=1 │ nom=Widget │ prix=75             │
│ DB_TRX_ID = 150                         │
│ DB_ROLL_PTR = → [undo: prix=50, TRX=100]│
└─────────────────────────────────────────┘
```

**Comment InnoDB utilise DB_TRX_ID** :

```python
# Pseudo-code : Algorithme de lecture
def read_row(row, my_trx_id):
    if row.DB_TRX_ID <= my_trx_id:
        # Version commitée avant ma transaction
        return row
    else:
        # Version trop récente, chercher dans l'undo log
        old_version = follow_rollback_pointer(row.DB_ROLL_PTR)
        return read_row(old_version, my_trx_id)
```

### 2.3 DB_ROLL_PTR : Le Pointeur de Rollback

**Rôle** : Pointeur vers l'ancienne version de la ligne dans l'**undo log**.

```
Chaîne de versions (undo chain) :

Version actuelle              Undo Log
┌──────────────────┐          ┌──────────────────┐
│ id=1, prix=100   │          │ id=1, prix=75    │
│ TRX_ID=200       │          │ TRX_ID=150       │
│ ROLL_PTR ────────┼────────→ │ ROLL_PTR ────────┼───→ ...
└──────────────────┘          └──────────────────┘
                                       ↓
                              ┌──────────────────┐
                              │ id=1, prix=50    │
                              │ TRX_ID=100       │
                              │ ROLL_PTR = NULL  │
                              └──────────────────┘
```

### 2.4 DB_ROW_ID : L'ID Interne de Ligne

**Rôle** : Identifiant unique de la ligne, utilisé uniquement si la table n'a **pas de PRIMARY KEY explicite**.

```sql
-- Table SANS primary key
CREATE TABLE logs (
    message TEXT,
    created_at TIMESTAMP
) ENGINE=InnoDB;

-- InnoDB ajoute automatiquement DB_ROW_ID
-- (Monotone croissant : 1, 2, 3, ...)

-- Table AVEC primary key
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB;

-- DB_ROW_ID n'est PAS utilisé (économie d'espace)
```

⚠️ **Important** : Si vous n'avez pas de PK, InnoDB utilise 6 bytes par ligne pour DB_ROW_ID. Toujours définir une PRIMARY KEY explicite !

### 2.5 Visualisation Complète

```sql
-- Exemple avec historique de modifications
-- T0: Création (TRX=100)
INSERT INTO produits (id, nom, prix, stock) VALUES (42, 'Widget', 50, 100);

-- T1: Modification 1 (TRX=150)
UPDATE produits SET prix = 75 WHERE id = 42;

-- T2: Modification 2 (TRX=200)
UPDATE produits SET stock = 95 WHERE id = 42;

-- État final :
┌─────────────────────────────────────────────────────────────────┐
│                   VERSION ACTUELLE (Buffer Pool)                │
├─────────────────────────────────────────────────────────────────┤
│ id=42 │ nom=Widget │ prix=75 │ stock=95                         │
│ DB_TRX_ID = 200                                                 │
│ DB_ROLL_PTR = → UNDO_LOG_ENTRY_2                                │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                      UNDO LOG (Tablespace)                      │
├─────────────────────────────────────────────────────────────────┤
│ ENTRY_2: id=42, stock=100 (avant UPDATE), TRX_ID=200            │
│ ROLL_PTR = → ENTRY_1                                            │
│                          ↓                                      │
│ ENTRY_1: id=42, prix=50 (avant UPDATE), TRX_ID=150              │
│ ROLL_PTR = → ENTRY_0                                            │
│                          ↓                                      │
│ ENTRY_0: NULL (première version)                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Read View : Le Snapshot Isolation

### 3.1 Qu'est-ce qu'un Read View ?

Un **read view** est un snapshot de l'état des transactions au moment où il est créé. Il détermine quelles versions de données une transaction peut voir.

**Composition d'un Read View** :

```python
class ReadView:
    low_limit_id      # Plus petit TRX_ID non commité
    up_limit_id       # Plus grand TRX_ID + 1
    trx_ids[]         # Liste des TRX_ID actives (non commitées)
    creator_trx_id    # TRX_ID de la transaction qui a créé le read view
```

### 3.2 Quand le Read View est-il Créé ?

Dépend du **niveau d'isolation** :

```sql
-- REPEATABLE READ (default InnoDB)
START TRANSACTION;
-- Read view créé ICI ↑
-- Figé pour toute la durée de la transaction

SELECT * FROM produits;  -- Utilise le read view créé au START
-- 10 minutes plus tard...
SELECT * FROM produits;  -- Toujours le MÊME read view
COMMIT;

-- READ COMMITTED
START TRANSACTION;
-- Pas de read view encore

SELECT * FROM produits;
-- Read view créé ICI ↑ pour CETTE requête seulement

-- Autre modification commitée entre-temps

SELECT * FROM produits;
-- NOUVEAU read view créé ↑ pour cette nouvelle requête
COMMIT;
```

### 3.3 Algorithme de Visibilité

**Comment InnoDB décide si une version est visible ?**

```python
def is_visible(row, read_view):
    row_trx_id = row.DB_TRX_ID

    # Cas 1 : Version créée par MA transaction
    if row_trx_id == read_view.creator_trx_id:
        return True  # Toujours visible

    # Cas 2 : Version très ancienne (avant toutes les transactions actives)
    if row_trx_id < read_view.low_limit_id:
        return True  # Commitée avant le snapshot

    # Cas 3 : Version très récente (après le snapshot)
    if row_trx_id >= read_view.up_limit_id:
        return False  # Commitée après le snapshot

    # Cas 4 : Entre low et up limit
    if row_trx_id in read_view.trx_ids:
        return False  # Transaction était active au moment du snapshot
    else:
        return True  # Transaction commitée entre low et up
```

### 3.4 Exemple Détaillé

```sql
-- État des transactions :
-- TRX 100 : COMMITTED
-- TRX 105 : COMMITTED
-- TRX 110 : ACTIVE (non commitée)
-- TRX 115 : ACTIVE (non commitée)

-- Transaction A (TRX 120) démarre
START TRANSACTION;

-- Read view créé :
-- low_limit_id = 110 (plus petit actif)
-- up_limit_id = 120 (mon ID)
-- trx_ids = [110, 115]

-- Ligne avec DB_TRX_ID = 105
-- 105 < 110 (low_limit) → Visible ✅

-- Ligne avec DB_TRX_ID = 110
-- 110 in [110, 115] → Transaction active → Pas visible ❌
-- → Suivre DB_ROLL_PTR pour trouver version plus ancienne

-- Ligne avec DB_TRX_ID = 125
-- 125 >= 120 (up_limit) → Trop récent → Pas visible ❌
-- → Suivre DB_ROLL_PTR
```

---

## 4. L'Undo Log : Système de Versioning

### 4.1 Structure et Rôle

L'**undo log** est le journal où InnoDB stocke les anciennes versions des lignes modifiées.

**Types d'undo logs** :

```sql
-- INSERT undo log
-- Stocke l'info pour DELETE la ligne si ROLLBACK
INSERT INTO produits VALUES (42, 'Widget', 50);
-- Undo : DELETE FROM produits WHERE id = 42

-- UPDATE undo log
-- Stocke l'ancienne valeur
UPDATE produits SET prix = 75 WHERE id = 42;
-- Undo : prix était 50, TRX_ID était 100

-- DELETE undo log (mark delete)
-- La ligne n'est pas vraiment supprimée, juste marquée
DELETE FROM produits WHERE id = 42;
-- Undo : Ligne existe encore, marquée pour purge
```

### 4.2 Undo Log Segments

```
InnoDB Tablespace :
┌────────────────────────────────────────┐
│ System Tablespace (ibdata1)            │
├────────────────────────────────────────┤
│ ┌────────────────────────────────────┐ │
│ │ Rollback Segment 0                 │ │
│ │  - Undo Log 1 (TRX 100)            │ │
│ │  - Undo Log 2 (TRX 101)            │ │
│ │  - Undo Log 3 (TRX 102)            │ │
│ └────────────────────────────────────┘ │
│ ┌────────────────────────────────────┐ │
│ │ Rollback Segment 1                 │ │
│ │  - Undo Log 4 (TRX 103)            │ │
│ │  ...                               │ │
│ └────────────────────────────────────┘ │
│ ...                                    │
│ (128 rollback segments par défaut)     │
└────────────────────────────────────────┘
```

**Configuration** :

```sql
-- Nombre de rollback segments
SHOW VARIABLES LIKE 'innodb_rollback_segments';
-- Default : 128

-- Undo tablespaces séparés (MariaDB 10.0+)
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
-- Recommandé : 2-4 (permet rotation et purge plus efficace)
```

### 4.3 Histoire d'une Modification

```sql
-- T0: Ligne originale (TRX 100)
INSERT INTO users (id, name, salary) VALUES (1, 'Alice', 50000);
COMMIT;

┌─────────────────────────────────────┐
│ Buffer Pool (version actuelle)      │
│ id=1, name=Alice, salary=50000      │
│ TRX_ID=100, ROLL_PTR=NULL           │
└─────────────────────────────────────┘

-- T1: Première modification (TRX 150)
UPDATE users SET salary = 55000 WHERE id = 1;

┌─────────────────────────────────────┐
│ Buffer Pool (nouvelle version)      │
│ id=1, name=Alice, salary=55000      │
│ TRX_ID=150, ROLL_PTR=→undo_1        │
└─────────────────────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ Undo Log Entry 1                    │
│ id=1, salary=50000 (old value)      │
│ TRX_ID=100, ROLL_PTR=NULL           │
└─────────────────────────────────────┘

COMMIT;

-- T2: Deuxième modification (TRX 200)
UPDATE users SET name = 'Alice Smith' WHERE id = 1;

┌─────────────────────────────────────┐
│ Buffer Pool (version finale)        │
│ id=1, name=Alice Smith, salary=55000│
│ TRX_ID=200, ROLL_PTR=→undo_2        │
└─────────────────────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ Undo Log Entry 2                    │
│ id=1, name=Alice (old value)        │
│ TRX_ID=150, ROLL_PTR=→undo_1        │
└─────────────────────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ Undo Log Entry 1                    │
│ id=1, salary=50000 (old value)      │
│ TRX_ID=100, ROLL_PTR=NULL           │
└─────────────────────────────────────┘

-- Transaction T (TRX 140) qui a démarré entre T0 et T1
-- lit : suit la chaîne jusqu'à trouver version <= 140
-- → Lit salary=50000, name=Alice
```

---

## 5. Consistent Read vs Locking Read

### 5.1 Consistent Read (Non-Locking)

**Principe** : Lecture via MVCC, sans poser de verrou.

```sql
-- Lecture normale : Utilise le MVCC
SELECT * FROM produits WHERE id = 42;

-- Ce qui se passe :
-- 1. InnoDB consulte le read view
-- 2. Vérifie si DB_TRX_ID est visible
-- 3. Si oui : retourne la ligne
-- 4. Si non : suit DB_ROLL_PTR dans l'undo log
-- 5. Pas de verrou posé ✅

-- Avantages :
-- - Aucun blocage
-- - Performance maximale
-- - Scalabilité parfaite

-- Inconvénients :
-- - Lit potentiellement une ancienne version
-- - Pas de garantie que la ligne ne changera pas
```

### 5.2 Locking Read

**Principe** : Lecture avec verrou explicite.

```sql
-- FOR UPDATE : Verrou exclusif
SELECT * FROM produits WHERE id = 42 FOR UPDATE;

-- FOR SHARE : Verrou partagé
SELECT * FROM produits WHERE id = 42 FOR SHARE;

-- Ce qui se passe :
-- 1. InnoDB lit la version ACTUELLE (pas via MVCC)
-- 2. Pose un verrou (X ou S) sur la ligne
-- 3. Les autres transactions doivent attendre
-- 4. Garantit que la ligne ne changera pas

-- Avantages :
-- - Garantie de cohérence
-- - Évite les race conditions

-- Inconvénients :
-- - Blocage possible
-- - Performance réduite
```

### 5.3 Exemple de Différence

```sql
-- Transaction A
START TRANSACTION;

-- Consistent read
SELECT @prix1 := prix FROM produits WHERE id = 42;
-- Lit : 50€ (via MVCC, pas de verrou)

-- Transaction B modifie (entre-temps)
UPDATE produits SET prix = 75 WHERE id = 42;
COMMIT;

-- Transaction A continue
SELECT @prix2 := prix FROM produits WHERE id = 42;
-- En REPEATABLE READ : Lit toujours 50€ ✅ (MVCC)
-- En READ COMMITTED : Lit 75€ ❌ (nouveau read view)

-- Problème : Si A doit modifier basé sur la lecture
UPDATE produits SET prix = @prix1 * 1.1 WHERE id = 42;
-- 💥 Race condition ! Prix basé sur ancienne valeur

COMMIT;

-- ✅ SOLUTION : Locking read
-- Transaction A refait
START TRANSACTION;

SELECT @prix := prix FROM produits WHERE id = 42 FOR UPDATE;
-- 🔒 Verrou posé, personne ne peut modifier

-- Calcul sécurisé
UPDATE produits SET prix = @prix * 1.1 WHERE id = 42;

COMMIT;
```

---

## 6. Purge : Nettoyage des Anciennes Versions

### 6.1 Pourquoi le Purge est Nécessaire

Sans purge, l'undo log **grandirait indéfiniment** :

```
Scénario sans purge :
T0: Version 1 (TRX 100)
T1: Version 2 (TRX 150) → Undo: Version 1
T2: Version 3 (TRX 200) → Undo: Version 2 → Undo: Version 1
T3: Version 4 (TRX 250) → Undo: Version 3 → ... → Version 1

💥 Après 1 million de modifications :
→ 1 million de versions dans l'undo log
→ Plusieurs GB d'espace consommé
→ Lectures de plus en plus lentes
```

**Solution : Purge Thread**

InnoDB a des threads dédiés qui nettoient les anciennes versions **qui ne sont plus nécessaires**.

### 6.2 Quand une Version Peut-elle Être Purgée ?

Une version undo peut être supprimée si :

1. **Plus aucune transaction active** ne pourrait avoir besoin de cette version
2. La version est plus ancienne que toutes les transactions actives

```sql
-- État des transactions :
-- TRX 100 : COMMITTED
-- TRX 150 : COMMITTED
-- TRX 200 : ACTIVE (démarrée)
-- TRX 250 : COMMITTED

-- Versions :
-- V1 (TRX 100)  ← Peut être purgée ? NON (TRX 200 peut en avoir besoin)
-- V2 (TRX 150)  ← Peut être purgée ? NON (TRX 200 peut en avoir besoin)
-- V3 (TRX 200)  ← Version actuelle
-- V4 (TRX 250)  ← Version actuelle

-- Si TRX 200 commit :
-- V1 et V2 peuvent maintenant être purgées ✅
```

### 6.3 History List Length

Le **history list length** mesure le nombre d'entrées undo en attente de purge.

```sql
-- Voir le history list length
SHOW ENGINE INNODB STATUS\G

-- Chercher la ligne :
-- History list length 12345
```

**Interprétation** :

```
History list length < 1000      : ✅ Excellent
History list length 1000-10000  : 🟡 Acceptable
History list length > 10000     : 🟠 Attention
History list length > 100000    : 🔴 PROBLÈME
```

**Causes d'un history list length élevé** :

1. **Transactions très longues** : Bloquent le purge
2. **Taux de modification élevé** : Le purge n'arrive pas à suivre
3. **Threads de purge insuffisants** : Pas assez de workers

### 6.4 Configuration du Purge

```sql
-- Nombre de threads de purge
SHOW VARIABLES LIKE 'innodb_purge_threads';
-- Default : 4
-- Recommandé : 4-8 sur serveurs modernes

SET GLOBAL innodb_purge_threads = 8;

-- Batch size du purge
SHOW VARIABLES LIKE 'innodb_purge_batch_size';
-- Default : 300
-- Augmenter si history list grandit

SET GLOBAL innodb_purge_batch_size = 500;

-- Nombre de pages à purger par batch
SHOW VARIABLES LIKE 'innodb_max_purge_lag';
-- Default : 0 (pas de limite)
-- Utiliser si on veut ralentir les écritures pour laisser le purge rattraper

SET GLOBAL innodb_max_purge_lag = 1000000;
```

**Configuration production recommandée** :

```ini
[mysqld]
# Threads de purge (4-8 sur serveur moderne)
innodb_purge_threads = 4

# Batch size (augmenter si workload élevé)
innodb_purge_batch_size = 300

# Max purge lag (0 = pas de limite)
innodb_max_purge_lag = 0
innodb_max_purge_lag_delay = 0
```

---

## 7. Impact des Transactions Longues

### 7.1 Le Problème

Les **transactions longues** empêchent le purge et font grossir l'undo log :

```sql
-- Transaction problématique
START TRANSACTION;  -- T0

SELECT * FROM large_table;  -- 10 millions de lignes

-- La transaction reste ouverte pendant 2 HEURES
-- Pendant ce temps :
-- - Des milliers d'UPDATE/DELETE ont lieu
-- - L'undo log accumule toutes ces versions
-- - Le purge ne peut PAS nettoyer (transaction T0 active)
-- - History list length explose : 1000 → 100000

-- 2 heures plus tard...
COMMIT;  -- Enfin !

-- Conséquences :
-- - Undo log : 5 GB consommés
-- - Lectures lentes (doivent parcourir l'undo chain)
-- - Écritures ralenties (undo log plein)
-- - Serveur sous pression
```

### 7.2 Monitoring des Transactions Longues

```sql
-- Identifier les transactions de plus de 10 minutes
SELECT
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_rows_modified,
    trx_rows_locked,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 10 MINUTE
ORDER BY trx_started;

-- Alerter si :
-- - duration_sec > 600 (10 minutes)
-- - trx_rows_modified = 0 (transaction en lecture, oubliée ?)
```

### 7.3 Impact sur les Performances

**Benchmark : Transaction longue vs courte**

```sql
-- Test 1 : Transaction courte (< 1 seconde)
START TRANSACTION;
SELECT COUNT(*) FROM produits;
COMMIT;
-- Throughput : 10000 TPS

-- Test 2 : Transaction longue (10 minutes)
-- En parallèle : 1 transaction reste ouverte 10 minutes
-- Sans rien faire (juste un SELECT initial)

-- Impact sur les autres transactions :
-- Throughput : 3000 TPS (70% de dégradation !)
-- History list length : 5000 → 150000
-- Latence reads : 5ms → 50ms (10x plus lent)
```

### 7.4 Solutions

```sql
-- ✅ Solution 1 : Éviter les transactions longues
-- Faire les calculs HORS transaction
SELECT @data := ... FROM large_table;  -- Sans transaction
-- Traitement des données ici

-- Transaction ultra-courte pour l'écriture
START TRANSACTION;
INSERT INTO results VALUES (@data);
COMMIT;

-- ✅ Solution 2 : COMMIT régulier pour les longs traitements
START TRANSACTION;
DECLARE done BOOLEAN DEFAULT FALSE;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

OPEN cursor_big_table;

read_loop: LOOP
    FETCH cursor_big_table INTO ...;
    IF done THEN LEAVE read_loop; END IF;

    -- Traiter la ligne

    -- COMMIT tous les 1000 enregistrements
    IF MOD(counter, 1000) = 0 THEN
        COMMIT;
        START TRANSACTION;
    END IF;
END LOOP;

COMMIT;

-- ✅ Solution 3 : Tuer les transactions zombies
-- Script de monitoring
SELECT CONCAT('KILL ', trx_mysql_thread_id, ';')
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 1 HOUR
  AND trx_rows_modified = 0;  -- Pas de modifications = zombie potentiel
```

---

## 8. Tuning et Optimisation du MVCC

### 8.1 Configuration Undo Tablespace

```sql
-- Voir la configuration actuelle
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
-- Recommandé : 2-4

SHOW VARIABLES LIKE 'innodb_undo_directory';
-- Recommandé : Disque SSD séparé si possible

SHOW VARIABLES LIKE 'innodb_undo_log_truncate';
-- ON pour permettre le truncate automatique

SHOW VARIABLES LIKE 'innodb_max_undo_log_size';
-- Taille max avant truncate (default : 1GB)
```

**Configuration optimale** :

```ini
[mysqld]
# Undo tablespaces séparés (rotation plus facile)
innodb_undo_tablespaces = 3

# Truncate automatique quand > 1GB
innodb_undo_log_truncate = ON
innodb_max_undo_log_size = 1073741824  # 1GB

# Répertoire dédié (optionnel, sur SSD)
innodb_undo_directory = /var/lib/mysql-undo
```

### 8.2 Buffer Pool et MVCC

```sql
-- Taille du buffer pool (70-80% de la RAM)
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Plus le buffer pool est grand :
-- + Plus de versions peuvent rester en mémoire
-- + Moins de I/O pour lire l'undo log depuis le disque
-- = Meilleure performance des consistent reads

-- Recommandation :
-- Serveur 16GB RAM → innodb_buffer_pool_size = 12GB
-- Serveur 64GB RAM → innodb_buffer_pool_size = 48GB
```

### 8.3 Monitoring du MVCC

```sql
-- Statistiques MVCC
SHOW ENGINE INNODB STATUS\G

-- Sections importantes :
-- 1. History list length
--    → Doit rester < 10000

-- 2. Purge done for trx's
--    → Nombre de transactions purgées

-- 3. Undo log entries
--    → Nombre d'entrées undo

-- Requête de monitoring
SELECT
    'History List Length' AS metric,
    SUBSTRING_INDEX(
        SUBSTRING_INDEX(variable_value, 'History list length ', -1),
        '\n', 1
    ) AS value
FROM information_schema.GLOBAL_STATUS
WHERE variable_name = 'Innodb_page_size'
LIMIT 1;
```

### 8.4 Alerting Recommandé

```python
# Script de monitoring (Python)
import pymysql
import re

def check_history_list_length():
    conn = pymysql.connect(...)
    cursor = conn.cursor()

    cursor.execute("SHOW ENGINE INNODB STATUS")
    status = cursor.fetchone()[2]  # Column 'Status'

    # Parser le history list length
    match = re.search(r'History list length (\d+)', status)
    if match:
        hll = int(match.group(1))

        # Alertes
        if hll > 100000:
            send_alert("CRITICAL: History list length = {}".format(hll))
        elif hll > 10000:
            send_alert("WARNING: History list length = {}".format(hll))

        return hll

    return None

# Exécuter toutes les 5 minutes
```

---

## 9. Cas d'Usage et Patterns

### 9.1 Long Report avec Cohérence

```sql
-- ❌ MAUVAIS : Transaction longue
START TRANSACTION;

SELECT * FROM sales WHERE year = 2024;  -- 10 millions de lignes
-- Traitement en mémoire (30 minutes)
-- Génération de graphiques
-- Calculs statistiques

COMMIT;

-- 💥 Problèmes :
-- - Transaction active 30 minutes
-- - Bloque le purge
-- - History list explose

-- ✅ BON : WITH CONSISTENT SNAPSHOT puis lecture par batch
START TRANSACTION WITH CONSISTENT SNAPSHOT;

-- Lire par lots
SELECT * FROM sales
WHERE year = 2024
LIMIT 100000 OFFSET 0;
-- Traiter ce lot
COMMIT;

-- Nouveau lot
START TRANSACTION WITH CONSISTENT SNAPSHOT;
SELECT * FROM sales
WHERE year = 2024
LIMIT 100000 OFFSET 100000;
COMMIT;

-- Etc.
```

### 9.2 High-Frequency Updates

```sql
-- Scénario : Compteur mis à jour très fréquemment
-- (ex: page views, likes, stats)

-- ❌ MAUVAIS : Update direct (hot spot + undo log énorme)
UPDATE counters SET views = views + 1 WHERE page_id = 'home';
-- 1000 updates/sec = 1000 versions dans l'undo log

-- ✅ BON : Buffer en mémoire puis flush périodique
-- Applicatif maintient le compteur en Redis/Memcached
-- Flush vers MariaDB toutes les 10 secondes

-- Pattern : Eventually consistent counter
```

### 9.3 Analyse de Données en Streaming

```sql
-- Pattern : Lecture continue sans bloquer le purge

DELIMITER //

CREATE PROCEDURE process_stream()
BEGIN
    DECLARE done BOOLEAN DEFAULT FALSE;
    DECLARE v_id INT;
    DECLARE cur CURSOR FOR
        SELECT id FROM events ORDER BY id;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;

    read_loop: LOOP
        -- Micro-transaction
        START TRANSACTION;

        FETCH cur INTO v_id;
        IF done THEN
            COMMIT;
            LEAVE read_loop;
        END IF;

        -- Traiter l'événement
        CALL process_event(v_id);

        -- COMMIT immédiat
        COMMIT;
        -- → Permet au purge de progresser
    END LOOP;

    CLOSE cur;
END//

DELIMITER ;
```

---

## ✅ Points clés à retenir

- **MVCC** : Permet lectures sans blocage via multi-versioning
- **DB_TRX_ID** : ID de transaction qui a créé/modifié la ligne
- **DB_ROLL_PTR** : Pointeur vers ancienne version (undo log)
- **Read View** : Snapshot déterminant quelles versions sont visibles
- **Undo Log** : Stocke les anciennes versions pour MVCC et rollback
- **Consistent Read** : Lecture via MVCC, sans verrou
- **Locking Read** : FOR UPDATE/SHARE, lit version actuelle avec verrou
- **Purge** : Nettoie les anciennes versions non nécessaires
- **History List Length** : Métrique critique (doit rester < 10000)
- **Transactions longues** : Ennemi #1 du MVCC (bloquent le purge)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 InnoDB Multi-Versioning](https://mariadb.com/kb/en/innodb-multi-versioning/)
- [📖 InnoDB Undo Log](https://mariadb.com/kb/en/innodb-undo-log/)
- [📖 InnoDB Purge](https://mariadb.com/kb/en/innodb-purge-configuration/)

### Articles techniques
- [Understanding InnoDB MVCC](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [InnoDB Internals: Undo Log](https://blog.jcole.us/innodb/)
- "Database Internals" by Alex Petrov - Chapitre sur MVCC

---

## ➡️ Section suivante

**6.7 Savepoints : Points de sauvegarde** : Rollback partiel et gestion granulaire des transactions.

---


⏭️ [Savepoints : Points de sauvegarde](/06-transactions-et-concurrence/07-savepoints.md)

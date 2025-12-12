🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2 InnoDB : Le moteur par défaut

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Section 7.0 (Architecture Pluggable Storage Engine), concepts de transactions ACID

> **Public cible** : DBA, Architectes de bases de données, Ingénieurs SRE

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre en profondeur l'architecture interne d'InnoDB
- Maîtriser la configuration du Buffer Pool pour optimiser les performances mémoire
- Configurer et monitorer les mécanismes Redo Log et Undo Log
- Optimiser InnoDB pour différents workloads (OLTP, mixte, haute concurrence)
- Diagnostiquer et résoudre les problèmes de performance liés à InnoDB
- Appliquer les bonnes pratiques de configuration en environnement de production

---

## Introduction

InnoDB est le **moteur de stockage transactionnel** par défaut de MariaDB depuis la version 10.2 (2017). Il représente le choix optimal pour **95% des applications** grâce à son équilibre entre performance, fiabilité et fonctionnalités avancées.

### Pourquoi InnoDB domine-t-il ?

Contrairement aux moteurs historiques (MyISAM, ISAM), InnoDB a été conçu dès l'origine pour les **charges de production critiques** avec des exigences strictes de :

- **Cohérence des données** : Transactions ACID complètes
- **Haute concurrence** : Row-level locking et MVCC
- **Récupération automatique** : Crash recovery sans intervention manuelle
- **Intégrité référentielle** : Support natif des Foreign Keys
- **Performance moderne** : Optimisations pour SSD, multi-core, grandes mémoires

### Historique et évolution

| Année | Version | Évolution majeure |
|-------|---------|-------------------|
| 2001 | InnoDB 1.0 | Première version commerciale (Innobase Oy) |
| 2005 | MySQL 5.0 | InnoDB devient le moteur par défaut |
| 2008 | Oracle | Rachat d'Innobase par Oracle |
| 2010 | MySQL 5.5 | InnoDB Plugin avec améliorations performances |
| 2016 | MySQL 5.7 | Buffer Pool dump/restore, online DDL |
| 2017 | MariaDB 10.2 | InnoDB par défaut dans MariaDB |
| 2023 | MariaDB 10.11 | Optimisations I/O et compression |
| 2025 | **MariaDB 11.8** 🆕 | **innodb_alter_copy_bulk**, cost optimizer SSD |

### Position dans l'écosystème MariaDB

```
MariaDB Server
    │
    ├── SQL Layer (parser, optimizer, executor)
    │       │
    │       └── Storage Engine API
    │               │
    │               ├── InnoDB ◄── 95% des tables en production
    │               ├── Aria (tables système)
    │               ├── ColumnStore (analytics)
    │               ├── Vector/HNSW (IA) 🆕
    │               └── Autres moteurs
    │
    └── InnoDB Internal Architecture
            ├── Buffer Pool (cache mémoire)
            ├── Redo Log (durabilité)
            ├── Undo Log (rollback + MVCC)
            ├── Change Buffer (optimisation insertions)
            ├── Adaptive Hash Index (optimisation auto)
            └── Double Write Buffer (protection corruption)
```

💡 **Principe directeur** : InnoDB privilégie la **durabilité et la cohérence** tout en offrant d'excellentes performances grâce à une gestion mémoire sophistiquée et des optimisations I/O avancées.

---

## Architecture globale d'InnoDB

### Vue d'ensemble des composants

InnoDB est structuré autour de plusieurs sous-systèmes interconnectés :

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATIONS / CLIENTS                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     SQL LAYER (MariaDB)                     │
│  • Query Parser    • Optimizer    • Query Executor          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   InnoDB STORAGE ENGINE                     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │           BUFFER POOL (In-Memory Cache)            │     │
│  │  • Data Pages  • Index Pages  • Adaptive Hash      │     │
│  └────────────────────────────────────────────────────┘     │
│                       ↑          ↓                          │
│  ┌──────────────┐    │           │    ┌──────────────┐      │
│  │  REDO LOG    │←───┘           └───→│  UNDO LOG    │      │
│  │  (WAL)       │                     │  (Rollback)  │      │
│  └──────────────┘                     └──────────────┘      │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Change       │    │ Double Write │    │ Adaptive     │   │
│  │ Buffer       │    │ Buffer       │    │ Hash Index   │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      DISK STORAGE                           │
│  • System Tablespace (ibdata1)                              │
│  • File-per-table (.ibd files)                              │
│  • Redo Log files (ib_logfile*)                             │
│  • Undo Tablespaces                                         │
└─────────────────────────────────────────────────────────────┘
```

### Flux d'une requête de modification

Comprendre le chemin complet d'une requête `UPDATE` aide à saisir les interactions entre composants :

```sql
-- Exemple de requête
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
```

**Étapes détaillées :**

1. **Parsing et optimisation** (SQL Layer)
   - Analyse syntaxique de la requête
   - Recherche du plan d'exécution optimal
   - Vérification des privilèges

2. **Lecture de la ligne** (InnoDB)
   ```
   a. Recherche dans Buffer Pool
      ├─ Page en cache ? → Lecture RAM (rapide)
      └─ Page absente ? → Lecture disque → Chargement dans Buffer Pool

   b. Placement d'un verrou X (exclusive) sur la ligne
      └─ Row-level locking : seule cette ligne est verrouillée
   ```

3. **Écriture dans Undo Log**
   ```
   Sauvegarde de l'ancienne valeur (balance = 1000)
   ├─ Permet ROLLBACK si nécessaire
   └─ Utilisé pour MVCC (autres transactions)
   ```

4. **Modification dans Buffer Pool**
   ```
   Page mémoire modifiée : balance = 900
   Page marquée comme "dirty" (modifiée)
   ```

5. **Écriture dans Redo Log**
   ```
   Enregistrement de : UPDATE accounts SET balance=900 WHERE id=42
   ├─ Write-Ahead Logging (WAL)
   └─ Garantit durabilité même si crash avant flush disque
   ```

6. **COMMIT**
   ```
   a. Flush Redo Log sur disque (fsync)
      └─ Transaction durable

   b. Libération du verrou

   c. Flush asynchrone de la page dirty vers disque
      └─ Peut être différé (pas bloquant)
   ```

💡 **Principe WAL (Write-Ahead Logging)** : Les modifications sont d'abord écrites dans le Redo Log (séquentiel, rapide) avant d'être appliquées aux pages de données (aléatoire, lent). Cela optimise les performances tout en garantissant la durabilité.

### Structures de données sur disque

#### System Tablespace vs File-per-table

```sql
-- Afficher la configuration actuelle
SHOW VARIABLES LIKE 'innodb_file_per_table';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_file_per_table   | ON    |
+-------------------------+-------+

-- Avec innodb_file_per_table = ON (recommandé)
/var/lib/mysql/
├── ibdata1                    # System tablespace (métadonnées, undo log)
├── ib_logfile0                # Redo log
├── ib_logfile1                # Redo log
├── mydb/
│   ├── users.ibd              # Table users (données + index)
│   ├── orders.ibd             # Table orders
│   └── products.ibd           # Table products
└── undo001, undo002           # Undo tablespaces (MariaDB 10.5+)
```

**Avantages file-per-table** :
- ✅ `DROP TABLE` libère immédiatement l'espace disque
- ✅ `OPTIMIZE TABLE` fonctionne efficacement
- ✅ Backup/restore sélectif par table
- ✅ Meilleure organisation et monitoring
- ✅ Possibilité de placer certaines tables sur des disques spécifiques

**Inconvénients** :
- ❌ Plus de fichiers ouverts (augmenter `open_files_limit`)
- ❌ Légère surcharge système avec beaucoup de tables

⚠️ **Configuration recommandée** :
```ini
[mysqld]
innodb_file_per_table = 1
```

#### Format de page et compression

InnoDB stocke les données en **pages de 16 KB** par défaut :

```sql
-- Vérifier la taille de page
SHOW VARIABLES LIKE 'innodb_page_size';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_page_size | 16384 |
+------------------+-------+

-- Différents formats de ligne
CREATE TABLE example_dynamic (
    id INT PRIMARY KEY,
    data VARCHAR(8000)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;  -- Défaut, recommandé

CREATE TABLE example_compressed (
    id INT PRIMARY KEY,
    data TEXT
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED
  KEY_BLOCK_SIZE=8;  -- Compression à 8 KB
```

**Formats de ligne disponibles** :

| Format | Description | Usage |
|--------|-------------|-------|
| **DYNAMIC** | Variable, données longues en overflow (recommandé) | **Par défaut depuis MariaDB 10.2** |
| COMPACT | Variable, plus ancien | Compatibilité legacy |
| REDUNDANT | Fixe, très ancien | Legacy MySQL 4.1 |
| COMPRESSED | Compression ZLIB page-level | Tables peu modifiées, gain espace |

💡 **Conseil** : Utiliser `ROW_FORMAT=DYNAMIC` pour les nouvelles tables. La compression est pertinente uniquement pour les tables volumineuses avec peu d'écritures (overhead CPU).

---

## Buffer Pool : Le cœur de la performance

Le **Buffer Pool** est la zone mémoire la plus critique d'InnoDB. Il met en cache :
- Pages de données
- Pages d'index
- Adaptive Hash Index
- Lock information
- Insert buffer

### Dimensionnement optimal

```ini
[mysqld]
# Règle générale : 70-80% de la RAM disponible
# Serveur avec 64 GB RAM → 48-50 GB pour Buffer Pool
innodb_buffer_pool_size = 48G

# Instances multiples pour réduire la contention
# Règle : 1 instance par 1 GB, maximum 64 instances
innodb_buffer_pool_instances = 48

# Chunk size (doit être diviseur de pool_size)
# Par défaut 128M, augmenter pour très gros pools
innodb_buffer_pool_chunk_size = 128M
```

**Formule de calcul** :
```
innodb_buffer_pool_size = innodb_buffer_pool_chunk_size
                          × innodb_buffer_pool_instances
                          × N (nombre de chunks par instance)
```

**Exemple** :
```
48 GB = 128 MB × 48 instances × 8 chunks
```

⚠️ **Contraintes** :
- `innodb_buffer_pool_size` doit être multiple de `chunk_size × instances`
- Si non multiple, MariaDB ajuste automatiquement (warning dans error log)

### Architecture interne du Buffer Pool

```
┌──────────────────── Buffer Pool Instance ───────────────────┐
│                                                             │
│  LRU List (Least Recently Used)                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  ┌────────────────┐  Young Sublist (37% par défaut)  │   │
│  │  │  Most recently │  • Pages accédées récemment      │   │
│  │  │  used pages    │  • Protected de l'éviction       │   │
│  │  └────────────────┘                                  │   │
│  │         ↓                                            │   │
│  │  Midpoint (séparation young/old)                     │   │
│  │         ↓                                            │   │
│  │  ┌────────────────┐  Old Sublist (63%)               │   │
│  │  │  Candidates    │  • Pages candidates à l'éviction │   │
│  │  │  for eviction  │  • Nouvelles pages chargées      │   │
│  │  └────────────────┘                                  │   │
│  │                                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Free List                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Pages libres disponibles pour chargement            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Flush List                                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Dirty pages (modifiées) en attente d'écriture       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Algorithme LRU amélioré

InnoDB utilise une variante sophistiquée du LRU pour éviter de polluer le cache avec des scans de grandes tables :

**Principe** :
1. Nouvelle page chargée → Insérée au **midpoint** (pas en tête)
2. Si accédée à nouveau dans un délai → Promue en **young sublist**
3. Si jamais réaccédée → Glisse vers old sublist → Éviction

```sql
-- Configuration du comportement LRU
[mysqld]
# Position du midpoint (% de la liste)
innodb_old_blocks_pct = 37  # 37% pour young, 63% pour old

# Délai avant promotion (millisecondes)
innodb_old_blocks_time = 1000  # 1 seconde
```

**Cas d'usage** :
- `innodb_old_blocks_time = 0` : Promotion immédiate (workload aléatoire)
- `innodb_old_blocks_time = 1000` : Protection contre scans (recommandé)

💡 **Conseil** : Pour des applications avec beaucoup de scans séquentiels (reporting, batch), augmenter `innodb_old_blocks_time` à 5000-10000 ms pour protéger le cache des requêtes OLTP.

### Monitoring du Buffer Pool

```sql
-- Vue globale des statistiques
SHOW ENGINE INNODB STATUS\G

-- Extrait pertinent :
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 51539607552
Dictionary memory allocated 1234567
Buffer pool size   3145728          -- Nombre de pages (pages × 16KB = taille totale)
Free buffers       1024             -- Pages libres
Database pages     3144000          -- Pages avec données
Old database pages 1160000          -- Pages dans old sublist
Modified db pages  50000            -- Dirty pages
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 1234567890, not young 987654321
10.00 youngs/s, 5.00 non-youngs/s
Pages read 9876543210, created 123456, written 7654321
50.00 reads/s, 0.10 creates/s, 30.00 writes/s
Buffer pool hit rate 999 / 1000      -- Hit ratio (objectif > 99%)
```

**Statistiques détaillées par instance** :

```sql
SELECT
    POOL_ID,
    POOL_SIZE,
    FREE_BUFFERS,
    DATABASE_PAGES,
    OLD_DATABASE_PAGES,
    MODIFIED_DATABASE_PAGES,
    PENDING_READS,
    PENDING_WRITES_LRU,
    PENDING_WRITES_FLUSH_LIST,
    PAGES_MADE_YOUNG,
    PAGES_NOT_MADE_YOUNG,
    HIT_RATE,
    YOUNG_MAKE_PER_THOUSAND_GETS AS young_rate
FROM information_schema.INNODB_BUFFER_POOL_STATS
ORDER BY POOL_ID;
```

**Calcul du hit ratio** :

```sql
-- Objectif : > 99% en production stable
SELECT
    ROUND(
        100 * (1 - (
            VARIABLE_VALUE /
            (SELECT VARIABLE_VALUE
             FROM information_schema.GLOBAL_STATUS
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
        )), 2
    ) AS buffer_pool_hit_ratio_pct
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads';
```

**Interprétation** :
- **> 99%** : Excellent, Buffer Pool bien dimensionné
- **95-99%** : Acceptable, mais peut bénéficier d'augmentation
- **< 95%** : Sous-dimensionné ou workload trop large pour la RAM

### Dump et restore du Buffer Pool

🔄 **Fonctionnalité** : Sauvegarder l'état du Buffer Pool au shutdown et le restaurer au démarrage pour éviter le "warm-up" lent.

```ini
[mysqld]
# Activer dump/restore automatique
innodb_buffer_pool_dump_at_shutdown = 1
innodb_buffer_pool_load_at_startup = 1

# Fichier de sauvegarde (emplacement datadir)
innodb_buffer_pool_filename = ib_buffer_pool

# Pourcentage de pages à sauvegarder (% du pool)
innodb_buffer_pool_dump_pct = 25  # 25% des pages les plus "chaudes"
```

**Opérations manuelles** :

```sql
-- Déclencher un dump manuel
SET GLOBAL innodb_buffer_pool_dump_now = ON;

-- Charger le dump
SET GLOBAL innodb_buffer_pool_load_now = ON;

-- Annuler un chargement en cours
SET GLOBAL innodb_buffer_pool_load_abort = ON;

-- Vérifier la progression
SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

💡 **Avantage** : Réduit drastiquement le temps de "warm-up" après restart. Un serveur avec 48 GB de Buffer Pool peut gagner 30-60 minutes de montée en performance.

---

## Redo Log : Garantie de durabilité (WAL)

Le **Redo Log** implémente le principe **Write-Ahead Logging** : toutes les modifications sont enregistrées dans le log **avant** d'être appliquées aux pages de données.

### Architecture du Redo Log

```
┌─────────────────────────────────────────────────────────┐
│                    Redo Log Buffer (RAM)                │
│  • Taille : innodb_log_buffer_size (16 MB par défaut)   │
│  • Écriture : Chaque modification de transaction        │
└─────────────────────────────────────────────────────────┘
                        ↓ Flush
┌─────────────────────────────────────────────────────────┐
│               Redo Log Files (disque)                   │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │ ib_logfile0  │  │ ib_logfile1  │  (circular buffer)  │
│  │   512 MB     │  │   512 MB     │                     │
│  └──────────────┘  └──────────────┘                     │
│        ↑                                                │
│        └── Log Sequence Number (LSN)                    │
│            Position d'écriture courante                 │
└─────────────────────────────────────────────────────────┘
```

### Configuration critique

```ini
[mysqld]
# Taille de chaque fichier redo log
# Plus grand = moins de checkpoints, mais recovery plus longue
# Recommandé : 512 MB - 2 GB (selon write throughput)
innodb_log_file_size = 1G

# Nombre de fichiers (généralement 2)
innodb_log_files_in_group = 2

# Taille totale = innodb_log_file_size × innodb_log_files_in_group
# Exemple : 1 GB × 2 = 2 GB de redo log

# Taille du buffer en RAM
innodb_log_buffer_size = 32M  # 16-64 MB selon write load
```

🆕 **MariaDB 11.8** : Les performances d'écriture dans le Redo Log ont été optimisées pour les SSD NVMe modernes.

### Politique de flush : innodb_flush_log_at_trx_commit

Ce paramètre contrôle le **compromis durabilité/performance** :

```ini
[mysqld]
# Valeurs possibles : 0, 1, 2
innodb_flush_log_at_trx_commit = 1  # Par défaut (sûr)
```

| Valeur | Comportement | Durabilité | Performance | Risque |
|--------|--------------|------------|-------------|--------|
| **0** | Flush vers disque toutes les 1 seconde | ⚠️ Faible | ⚡⚡⚡ Élevée | Perte jusqu'à 1s de transactions |
| **1** | Flush à chaque COMMIT (fsync) | ✅ Complète | ⚡ Moyenne | **Aucune perte** |
| **2** | Flush vers cache OS à chaque COMMIT | ⚠️ Moyenne | ⚡⚡ Bonne | Perte si crash OS (pas crash MariaDB) |

**Recommandations par cas d'usage** :

```ini
# Production critique (banque, e-commerce)
innodb_flush_log_at_trx_commit = 1

# Application tolérante aux pertes (analytics, logs)
innodb_flush_log_at_trx_commit = 2

# Développement/test
innodb_flush_log_at_trx_commit = 0
```

💡 **Conseil** : Pour la plupart des applications, utiliser **valeur 1** (défaut). La différence de performance avec la valeur 2 est minime sur SSD, et la sécurité est maximale.

### Checkpoints et cycle de vie du Redo Log

**Checkpoint** : Opération d'écriture des dirty pages vers disque pour libérer de l'espace dans le Redo Log.

```
Timeline :
───────────────────────────────────────────────────────────────►
         LSN 100      LSN 200      LSN 300      LSN 400
         │            │            │            │
         │            │            │            ▼
         │            │            │         Current Write
         │            │            ▼
         │            │         Checkpoint (dirty pages flushed)
         │            ▼
         │         Oldest LSN (début du log utilisable)
         ▼
      Transaction 1 committed
```

**Déclenchement d'un checkpoint** :
1. Redo Log proche de sa capacité (> 75%)
2. Timer périodique (toutes les N secondes)
3. Shutdown propre du serveur

```sql
-- Forcer un checkpoint (rarement nécessaire)
SET GLOBAL innodb_fast_shutdown = 0;
SHUTDOWN;
-- Au redémarrage, tous les logs sont flushés

-- Monitoring du checkpoint age
SHOW ENGINE INNODB STATUS\G
-- Section LOG :
Log sequence number 123456789012
Log flushed up to   123456789010
Pages flushed up to 123456780000
Last checkpoint at  123456770000
-- Checkpoint age = LSN current - LSN checkpoint
-- Si checkpoint age > 80% de log size → Saturation
```

⚠️ **Warning** : Si le checkpoint age atteint la capacité totale du Redo Log, **toutes les écritures sont bloquées** jusqu'à ce qu'un checkpoint libère de l'espace. Symptôme : application freeze brutalement.

**Dimensionnement du Redo Log** :

```
Règle empirique :
innodb_log_file_size × innodb_log_files_in_group
    ≥ 1 heure de write workload
```

Exemple : Si votre application génère 500 MB/heure d'écritures, configurez 1 GB de redo log total.

---

## Undo Log : Rollback et MVCC

Le **Undo Log** sert deux objectifs critiques :
1. **Rollback** de transactions non validées
2. **MVCC** (Multi-Version Concurrency Control) pour l'isolation

### Architecture des Undo Tablespaces

Depuis MariaDB 10.5, les Undo Logs sont stockés dans des **tablespaces dédiés** (plus dans ibdata1).

```sql
-- Configuration
[mysqld]
# Répertoire des undo tablespaces
innodb_undo_directory = /var/lib/mysql/undo

# Nombre de tablespaces undo (2-128)
innodb_undo_tablespaces = 2

# Taille max avant rotation (depuis 10.5)
innodb_max_undo_log_size = 1G

# Activation de la purge automatique
innodb_undo_log_truncate = ON
```

**Structure sur disque** :

```
/var/lib/mysql/undo/
├── undo001  # Tablespace undo actif
├── undo002  # Tablespace undo actif (rotation)
└── undo003  # Optionnel (rollback segments additionnels)
```

### MVCC en action

**Multi-Version Concurrency Control** permet aux lecteurs de voir une version cohérente des données sans bloquer les écrivains.

```sql
-- Timeline de deux transactions concurrentes

-- T0 : État initial
SELECT balance FROM accounts WHERE id = 1;
-- balance = 1000

-- T1 : Transaction 1 (longue lecture)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;
-- Lit 1000 (version originale)

-- T2 : Transaction 2 (modification)
UPDATE accounts SET balance = 1500 WHERE id = 1;
COMMIT;
-- Nouvelle version créée : balance = 1500
-- Ancienne version conservée dans Undo Log

-- T3 : Transaction 1 (reprend)
SELECT balance FROM accounts WHERE id = 1;
-- Lit toujours 1000 (grâce au Undo Log)
-- Isolation REPEATABLE READ

COMMIT;

-- T4 : Transaction 1 fermée
-- Ancienne version (1000) peut être purgée
```

**Mécanisme interne** :

```
Row structure :
┌───────────────────────────────────────────────────────────┐
│  Data:  id=1, balance=1500 (version actuelle)             │
│                                                           │
│  Metadata:                                                │
│  • TRX_ID : 987654 (transaction qui a modifié)            │
│  • ROLL_PTR : Pointer vers Undo Log (version précédente)  │
└───────────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │   Undo Log Record   │
              │  balance = 1000     │
              │  (ancienne version) │
              └─────────────────────┘
```

Lorsqu'une transaction en REPEATABLE READ lit la ligne :
1. InnoDB vérifie si `TRX_ID` de la ligne > `TRX_ID` du snapshot de la transaction
2. Si oui → Suit `ROLL_PTR` vers l'Undo Log pour lire l'ancienne version
3. Récursivité si nécessaire (plusieurs modifications successives)

### Purge automatique des anciennes versions

Le **purge thread** nettoie les anciennes versions devenues inutiles (aucune transaction active ne peut les lire).

```ini
[mysqld]
# Nombre de threads purge (1-32)
# Recommandé : 4-8 pour haute concurrence
innodb_purge_threads = 4

# Lag maximal avant ralentissement des écritures
# 0 = pas de limite (par défaut)
innodb_max_purge_lag = 0

# Délai avant de considérer qu'une transaction est "ancienne"
innodb_max_purge_lag_delay = 0  # ms
```

**Monitoring de la purge** :

```sql
SHOW ENGINE INNODB STATUS\G

-- Section TRANSACTIONS :
Trx id counter 987654
Purge done for trx's n:o < 987000 undo n:o < 0
History list length 123  -- Nombre de versions non purgées
-- Si History list length > 10000 → Purge thread surchargé
```

⚠️ **Problème courant** : Une transaction ouverte **très longtemps** (plusieurs heures) bloque la purge et cause :
- Croissance incontrôlée de l'Undo Log
- Saturation disque (ibdata1 ou undo tablespaces)
- Dégradation des performances (lecture des anciennes versions)

**Détection de transactions zombies** :

```sql
-- Transactions ouvertes depuis > 1 heure
SELECT
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 3600
ORDER BY duration_sec DESC;

-- Tuer une transaction problématique
KILL <trx_mysql_thread_id>;
```

---

## Configuration avancée InnoDB

### Gestion I/O et performances disque

```ini
[mysqld]
# Threads dédiés aux opérations I/O
innodb_read_io_threads = 8   # Lecture asynchrone (4-16)
innodb_write_io_threads = 8  # Écriture asynchrone (4-16)

# Capacité I/O du stockage (IOPS)
# HDD 7200 RPM : 100-200 IOPS
# SSD SATA : 2000-5000 IOPS
# SSD NVMe : 10000-50000 IOPS
innodb_io_capacity = 4000       # I/O background normal
innodb_io_capacity_max = 8000   # I/O background burst

# Méthode de flush des fichiers de données
# Linux recommandé : O_DIRECT (bypass cache OS)
innodb_flush_method = O_DIRECT
# Alternatives :
# - fsync : utilise le cache OS (peut causer double buffering)
# - O_DSYNC : sync après chaque write (très lent)
# - littlesync : hybride (MariaDB spécifique)
```

🆕 **MariaDB 11.8** : Le **cost-based optimizer** prend désormais mieux en compte les caractéristiques des SSD (latence, IOPS) pour ajuster les plans d'exécution.

**Recommandations par type de disque** :

| Type disque | io_capacity | io_capacity_max | flush_method |
|-------------|-------------|-----------------|--------------|
| HDD 7200 RPM | 200 | 400 | fsync |
| SSD SATA | 2000 | 4000 | **O_DIRECT** |
| SSD NVMe | 10000 | 20000 | **O_DIRECT** |
| Cloud (AWS EBS gp3) | 3000 | 6000 | O_DIRECT |

### Flush behavior et durabilité

```ini
[mysqld]
# Politique flush des pages modifiées (dirty pages)
innodb_flush_neighbors = 0  # 0 pour SSD (pas de seek), 1 pour HDD
# 0 : Flush uniquement la page demandée (optimal SSD)
# 1 : Flush les pages adjacentes (optimal HDD, réduit les seeks)

# Adaptive flushing (ajustement automatique selon workload)
innodb_adaptive_flushing = ON
innodb_adaptive_flushing_lwm = 10  # Low water mark (% redo log)
```

### Double Write Buffer

Le **Double Write Buffer** protège contre les **partial page writes** (corruption si crash pendant écriture d'une page 16 KB).

```
Workflow :
┌──────────────────────────────────────────────────────────┐
│ 1. Page modifiée dans Buffer Pool                        │
│    ↓                                                     │
│ 2. Écriture séquentielle vers Double Write Buffer        │
│    (2 MB zone dans ibdata1 ou fichier dédié)             │
│    ↓                                                     │
│ 3. fsync() du Double Write Buffer                        │
│    ↓                                                     │
│ 4. Écriture réelle vers .ibd file                        │
│    ↓                                                     │
│ 5. Si crash pendant étape 4 → Recovery utilise DWB       │
└──────────────────────────────────────────────────────────┘
```

```ini
[mysqld]
# Activer/désactiver (ON par défaut, recommandé)
innodb_doublewrite = ON

# Mode (depuis MariaDB 10.5)
innodb_doublewrite_file = ''  # Vide = dans ibdata1
# OU
innodb_doublewrite_file = 'xb_doublewrite'  # Fichier dédié
```

💡 **Conseil** : Ne désactiver `innodb_doublewrite` que si le système de fichiers garantit l'atomicité (ZFS, Btrfs avec checksums), ou pour des workloads très spécifiques (benchmarks, tables temporaires).

### Online DDL et ALTER TABLE

🆕 **MariaDB 11.8** : `innodb_alter_copy_bulk` améliore la vitesse de reconstruction d'index lors des `ALTER TABLE`.

```sql
-- Configuration
SET GLOBAL innodb_alter_copy_bulk = ON;

-- ALTER TABLE avec différents algorithmes
ALTER TABLE large_table
    ADD INDEX idx_email (email),
    ALGORITHM=INPLACE,     -- Pas de copie de table (optimal)
    LOCK=NONE;             -- Pas de lock lecture/écriture

-- Algorithmes disponibles :
-- INPLACE : Modification sur place (recommandé si supporté)
-- COPY : Copie complète de la table (ancien comportement)
-- INSTANT : Modification instantanée (métadonnées uniquement)

-- Modes de verrouillage :
-- NONE : Pas de lock (lectures et écritures autorisées)
-- SHARED : Lock partagé (lectures OK, écritures bloquées)
-- EXCLUSIVE : Lock exclusif (tout bloqué)

-- Vérifier les opérations supportées en INPLACE
SELECT * FROM information_schema.INNODB_SYS_TABLES
WHERE NAME = 'mydb/large_table';
```

**Opérations INSTANT (quasi-instantanées)** :
- `ADD COLUMN` (en fin de table, avec DEFAULT)
- `DROP COLUMN` (colonnes virtuelles)
- `ALTER COLUMN SET DEFAULT`
- `RENAME TABLE`

**Opérations INPLACE (sans copie de table)** :
- `ADD INDEX`, `DROP INDEX`
- `RENAME COLUMN`
- `ALTER COLUMN` (augmenter VARCHAR, nullable)
- `ADD FOREIGN KEY`

**Opérations nécessitant COPY** :
- Réduire la taille d'une colonne VARCHAR
- Changer le type de données (INT → BIGINT)
- Changer ROW_FORMAT
- Changer ENGINE

### Gestion de la concurrence

```ini
[mysqld]
# Nombre de threads pouvant entrer dans InnoDB simultanément
# 0 = illimité (recommandé, InnoDB gère bien)
innodb_thread_concurrency = 0

# Durée max d'attente pour entrer dans InnoDB
innodb_concurrency_tickets = 5000

# Spin loops avant de mettre un thread en sleep
innodb_sync_spin_loops = 30

# Timeout des lock wait (secondes)
innodb_lock_wait_timeout = 50
```

💡 **Conseil** : Laisser `innodb_thread_concurrency = 0` sauf si vous rencontrez des problèmes de saturation CPU avec plus de 100+ connexions concurrentes.

### Adaptive Hash Index (AHI)

L'**Adaptive Hash Index** est une structure en mémoire créée automatiquement par InnoDB pour accélérer les lookups fréquents.

```ini
[mysqld]
# Activer/désactiver (ON par défaut)
innodb_adaptive_hash_index = ON

# Partitions de l'AHI (réduire contention)
innodb_adaptive_hash_index_parts = 8  # 1-512
```

**Fonctionnement** :
- InnoDB observe les patterns d'accès aux index B-Tree
- Si certaines pages sont accédées très fréquemment de manière prévisible
- → Création automatique d'un hash index en RAM
- → Lookup O(1) au lieu de O(log n)

**Monitoring** :

```sql
SHOW ENGINE INNODB STATUS\G

-- Section INSERT BUFFER AND ADAPTIVE HASH INDEX :
Hash table size 276671, node heap has 42 buffer(s)
Hash table size 276671, node heap has 57 buffer(s)
2.50 hash searches/s, 15.00 non-hash searches/s
-- Ratio hash/non-hash indique l'efficacité de l'AHI
```

⚠️ **Quand désactiver AHI** :
- Workload très aléatoire (pas de patterns)
- Contention élevée sur l'AHI (visible dans `SHOW ENGINE INNODB STATUS`)
- Tables avec beaucoup d'écritures (AHI reconstruit fréquemment)

### Change Buffer (Insert Buffer)

Le **Change Buffer** agrège les modifications sur les **index secondaires non-uniques** pour réduire les I/O aléatoires.

```ini
[mysqld]
# Taille max du Change Buffer (% du Buffer Pool)
innodb_change_buffer_max_size = 25  # 0-50%

# Types d'opérations bufferisées
innodb_change_buffering = all
# Valeurs possibles :
# - none : Désactivé
# - inserts : INSERT uniquement
# - deletes : DELETE uniquement
# - changes : INSERT + DELETE
# - purges : Purge operations
# - all : Toutes les opérations (défaut)
```

**Fonctionnement** :

```
Sans Change Buffer :
INSERT → Index primaire (mémoire)
      → Index secondaire 1 (I/O aléatoire)
      → Index secondaire 2 (I/O aléatoire)
      → Index secondaire 3 (I/O aléatoire)
Total : 3 I/O aléatoires par INSERT

Avec Change Buffer :
INSERT → Index primaire (mémoire)
      → Change Buffer (mémoire, batch)
Background merge → Index secondaires (I/O séquentiel batché)
Total : Beaucoup moins d'I/O aléatoires
```

💡 **Conseil** : Le Change Buffer est particulièrement bénéfique pour les tables avec beaucoup d'index secondaires et des insertions en batch (ETL, imports).

---

## Monitoring et diagnostics avancés

### Métriques critiques à surveiller

```sql
-- 1. Buffer Pool hit ratio (objectif > 99%)
SELECT
    ROUND(
        100 * (1 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
        )), 2
    ) AS hit_ratio_pct;

-- 2. Nombre de dirty pages (attention si > 75% du pool)
SELECT
    POOL_SIZE,
    MODIFIED_DATABASE_PAGES,
    ROUND(100 * MODIFIED_DATABASE_PAGES / POOL_SIZE, 2) AS dirty_pct
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- 3. Checkpoint age (attention si proche de log size)
SHOW ENGINE INNODB STATUS\G
-- Analyser : Log sequence number vs Last checkpoint at

-- 4. Transactions longues (> 1 heure)
SELECT
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec
FROM information_schema.INNODB_TRX
WHERE trx_state = 'RUNNING'
  AND TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 3600;

-- 5. Lock waits (deadlocks potentiels)
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
SELECT * FROM information_schema.INNODB_LOCKS;

-- 6. Taille des tablespaces
SELECT
    TABLE_NAME,
    ENGINE,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb
FROM information_schema.TABLES
WHERE ENGINE = 'InnoDB'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 20;
```

### Variables système essentielles

```sql
-- Afficher la configuration InnoDB courante
SHOW VARIABLES LIKE 'innodb%';

-- Variables critiques à vérifier :
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'innodb_io_capacity%';
SHOW VARIABLES LIKE 'innodb_file_per_table';
```

---

## ✅ Points clés à retenir

1. **InnoDB est le moteur par défaut** depuis MariaDB 10.2 et le choix optimal pour 95% des applications OLTP.

2. **Buffer Pool** : Dimensionner à 70-80% de la RAM disponible. C'est le facteur de performance #1 pour InnoDB.

3. **Redo Log** : Implemente Write-Ahead Logging (WAL) pour garantir la durabilité. Dimensionner à ≥ 1h de write workload.

4. **innodb_flush_log_at_trx_commit = 1** : Valeur par défaut garantissant zéro perte de données (ACID complet).

5. **Undo Log** : Permet MVCC (isolation) et rollback. Attention aux transactions longues qui bloquent la purge.

6. **File-per-table** : Toujours activer (`innodb_file_per_table = 1`) pour une meilleure gestion de l'espace disque.

7. **O_DIRECT** : Méthode de flush recommandée sur SSD pour éviter le double buffering (cache OS + Buffer Pool).

8. 🆕 **MariaDB 11.8** : `innodb_alter_copy_bulk` accélère les reconstructions d'index. Cost optimizer optimisé pour SSD.

9. **Adaptive Hash Index** : Accélère automatiquement les lookups prévisibles. Peut être désactivé si contention.

10. **Monitoring** : Surveiller hit ratio (> 99%), dirty pages (< 75%), checkpoint age, et transactions longues.

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 InnoDB Storage Engine](https://mariadb.com/kb/en/innodb/)
- [📖 InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [📖 InnoDB Status Variables](https://mariadb.com/kb/en/innodb-status-variables/)
- [📖 InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [📖 InnoDB Redo Log](https://mariadb.com/kb/en/innodb-redo-log/)
- [📖 XtraDB/InnoDB File Format](https://mariadb.com/kb/en/xtradb-innodb-file-format/)

### Articles techniques approfondis

- [InnoDB Internal Architecture (Jeremy Cole)](https://blog.jcole.us/innodb/)
- [Understanding InnoDB MVCC (Percona)](https://www.percona.com/blog/understanding-mysql-innodb-mvcc/)
- [InnoDB Crash Recovery (MySQL Blog)](https://dev.mysql.com/blog-archive/innodb-crash-recovery/)
- [Tuning InnoDB Buffer Pool (MariaDB.org)](https://mariadb.org/tuning-innodb-buffer-pool/)

### Outils de monitoring et tuning

- **MySQLTuner** : [https://github.com/major/MySQLTuner-perl](https://github.com/major/MySQLTuner-perl)
- **Percona Monitoring and Management (PMM)** : [https://www.percona.com/software/database-tools/percona-monitoring-and-management](https://www.percona.com/software/database-tools/percona-monitoring-and-management)
- **pt-query-digest** (Percona Toolkit) : Analyse des slow queries
- **innotop** : Monitoring temps réel InnoDB

---

## ➡️ Section suivante

**[7.2.1 Caractéristiques : ACID, FK, Row-level locking](/07-moteurs-de-stockage/02.1-innodb-caracteristiques.md)** : Approfondissement des garanties ACID, gestion des Foreign Keys, et mécanismes de verrouillage granulaire au niveau ligne.

Puis nous continuerons avec :
- **7.2.2** : Buffer Pool et gestion mémoire (détails avancés)
- **7.2.3** : Redo Log et Undo Log (deep dive)
- **7.2.4** : Configuration avancée et tuning production

---

**📌 Mémo DBA** : Pour un serveur avec 64 GB RAM, configurer `innodb_buffer_pool_size = 48G`, `innodb_log_file_size = 2G`, et `innodb_flush_log_at_trx_commit = 1`. C'est 90% du tuning InnoDB.

⏭️ [Caractéristiques : ACID, FK, Row-level locking](/07-moteurs-de-stockage/02.1-innodb-caracteristiques.md)

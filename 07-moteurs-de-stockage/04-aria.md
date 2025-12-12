🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 Aria : Le successeur de MyISAM

> **Niveau** : Avancé
> **Durée estimée** : 1.5-2 heures
> **Prérequis** : Sections 7.1-7.3 (Architecture, InnoDB, MyISAM)

> **Public cible** : DBA, Architectes de bases de données, Développeurs MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture d'Aria et ses améliorations par rapport à MyISAM
- Maîtriser le mécanisme de crash recovery d'Aria
- Configurer et optimiser Aria pour différents cas d'usage
- Comprendre pourquoi et comment Aria est utilisé pour les tables système MariaDB
- Identifier les rares cas d'usage légitimes d'Aria
- Évaluer les compromis entre Aria, MyISAM et InnoDB
- Gérer la maintenance et la réparation des tables Aria

---

## Introduction

**Aria** (anciennement connu sous le nom de Maria) est un moteur de stockage développé par **Monty Widenius** (créateur original de MySQL) spécifiquement pour MariaDB. Il représente une évolution de MyISAM avec un objectif principal : ajouter la **crash-safety** tout en conservant la simplicité architecturale.

### Contexte et motivation

```
Évolution des moteurs MariaDB :
┌────────────────────────────────────────────────────────┐
│ 2000   MyISAM                                          │
│        • Rapide mais non transactionnel                │
│        • Corruption après crash                        │
│        ↓                                               │
│ 2007   Aria/Maria (début développement)                │
│        • "MyISAM with transactions" (objectif initial) │
│        • Crash recovery ajouté                         │
│        ↓                                               │
│ 2010   Aria 1.0 (MariaDB 5.1)                          │
│        • Crash-safe sans transactions complètes        │
│        • Utilisé pour tables système                   │
│        ↓                                               │
│ 2017   MariaDB 10.2                                    │
│        • Aria moteur par défaut pour tables système    │
│        • MyISAM déprécié pour nouveaux usages          │
│        ↓                                               │
│ 2025   MariaDB 11.8                                    │
│        • Aria stable et mature                         │
│        • Utilisé exclusivement pour tables système     │
└────────────────────────────────────────────────────────┘
```

### Positionnement d'Aria

```
Spectrum des moteurs de stockage :
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Simple/Rapide ◄──────────────────────► Robuste/ACID    │
│                                                         │
│  MyISAM          Aria          InnoDB                   │
│    │              │               │                     │
│    ├─ Pas de      ├─ Crash       ├─ Transactions        │
│    │  crash       │  recovery    │  ACID complètes      │
│    │  recovery    │              │                      │
│    │              ├─ Checksums   ├─ MVCC                │
│    ├─ Table       │              │                      │
│    │  lock        ├─ Page        ├─ Row-level           │
│    │              │  versioning  │  locking             │
│    │              │              │                      │
│    └─ Simple      └─ Intermédiaire └─ Complexe          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

💡 **Philosophie d'Aria** : "MyISAM avec crash-safety", pas un moteur transactionnel complet comme InnoDB.

---

## Architecture d'Aria

### Structure de fichiers

Comme MyISAM, Aria utilise une structure multi-fichiers, mais avec des composants additionnels :

```
/var/lib/mysql/mydb/
├── users.frm         # Format/définition de table (structure)
├── users.MAD         # Maria Data (données)
├── users.MAI         # Maria Index (index)
└── aria_log_control  # Fichier de contrôle du log
    aria_log.00000001 # Transaction log (WAL)
    aria_log.00000002
    ...
```

**Comparaison avec MyISAM** :

| Composant | MyISAM | Aria | Différence |
|-----------|--------|------|------------|
| Définition | `.frm` | `.frm` | Identique |
| Données | `.MYD` | `.MAD` | Format étendu avec checksums |
| Index | `.MYI` | `.MAI` | Format étendu avec versioning |
| Log | ❌ Aucun | ✅ `aria_log.*` | **Write-Ahead Log** pour recovery |
| Contrôle | ❌ Aucun | ✅ `aria_log_control` | État du log et LSN |

### Write-Ahead Log (WAL) d'Aria

Le **transaction log** d'Aria est le mécanisme clé qui permet la crash recovery :

```
Architecture du logging Aria :
┌─────────────────────────────────────────────────────────┐
│                  Application / SQL Layer                │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│                    Aria Handler                         │
│                                                         │
│  1. Modification en mémoire (Page Cache)                │
│     ↓                                                   │
│  2. Écriture dans aria_log.NNNNNNNN (WAL)               │
│     • Log Sequence Number (LSN)                         │
│     • Type d'opération (INSERT/UPDATE/DELETE)           │
│     • Anciennes et nouvelles valeurs                    │
│     ↓                                                   │
│  3. Flush log vers disque (fsync)                       │
│     ↓                                                   │
│  4. Modification pages données/index (asynchrone)       │
│                                                         │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│                  Fichiers disque                        │
│  • aria_log.NNNNNNNN (séquentiel, rapide)               │
│  • .MAD, .MAI (aléatoire, asynchrone)                   │
└─────────────────────────────────────────────────────────┘
```

**Principe Write-Ahead Logging** :
1. Toute modification est d'abord enregistrée dans le log
2. Le log est synchronisé sur disque (fsync)
3. Les pages de données sont modifiées en mémoire
4. Les pages modifiées sont écrites sur disque ultérieurement (flush asynchrone)

**En cas de crash** :
```
Recovery Process :
┌─────────────────────────────────────────────────────────┐
│ 1. MariaDB redémarre                                    │
│    ↓                                                    │
│ 2. Aria lit aria_log_control                            │
│    • Identifie le dernier LSN validé                    │
│    ↓                                                    │
│ 3. Rejoue le aria_log depuis ce LSN                     │
│    • Applique les modifications non flushées            │
│    • Reconstruit l'état cohérent                        │
│    ↓                                                    │
│ 4. Table disponible (recovery automatique)              │
└─────────────────────────────────────────────────────────┘
```

💡 **Différence avec MyISAM** : MyISAM n'a pas de log, donc crash = corruption potentielle. Aria rejoue le log = recovery automatique.

### Page Cache et gestion mémoire

```sql
-- Configuration du cache Aria
[mysqld]
aria_pagecache_buffer_size = 128M  # Cache pages données/index
aria_pagecache_division_limit = 100
aria_pagecache_age_threshold = 300
```

**Architecture du cache** :

```
┌────────────── Aria Page Cache ─────────────────┐
│                                                │
│  Hot Pages (fréquemment accédées)              │
│  ┌────────────────────────────────────────┐    │
│  │  Page 1  │  Page 5  │  Page 12  │ ...  │    │
│  └────────────────────────────────────────┘    │
│                                                │
│  Warm Pages                                    │
│  ┌────────────────────────────────────────┐    │
│  │  Page 3  │  Page 8  │  Page 15  │ ...  │    │
│  └────────────────────────────────────────┘    │
│                                                │
│  Cold Pages (candidates éviction)              │
│  ┌────────────────────────────────────────┐    │
│  │  Page 2  │  Page 7  │  Page 20  │ ...  │    │
│  └────────────────────────────────────────┘    │
└────────────────────────────────────────────────┘
```

**Comparaison avec InnoDB Buffer Pool** :

| Aspect | Aria Page Cache | InnoDB Buffer Pool |
|--------|-----------------|-------------------|
| Taille recommandée | 128-512 MB | 70-80% RAM totale |
| Gestion mémoire | LRU simple | LRU amélioré (young/old) |
| Granularité | Page (8 KB par défaut) | Page (16 KB par défaut) |
| Partitionnement | Non | Oui (instances multiples) |
| Complexité | Simple | Sophistiqué |

### Checksums et intégrité

Aria intègre des **checksums** sur toutes les pages pour détecter les corruptions :

```sql
-- Les checksums sont activés par défaut
[mysqld]
aria_page_checksum = ON  # Vérification intégrité pages

-- Lors de la lecture d'une page :
-- 1. Calcul du checksum de la page lue
-- 2. Comparaison avec checksum stocké
-- 3. Si différent → Erreur "page corrupted"
```

**Bénéfice** : Détection immédiate des corruptions silencieuses (bit flips, défaillances disque).

---

## Caractéristiques d'Aria

### ✅ Avantages par rapport à MyISAM

#### 1. Crash Recovery automatique

```sql
-- Scénario : Crash pendant une modification
CREATE TABLE test_aria (
    id INT PRIMARY KEY,
    data VARCHAR(100)
) ENGINE=Aria;

INSERT INTO test_aria VALUES (1, 'Data 1');
UPDATE test_aria SET data = 'Modified' WHERE id = 1;
-- CRASH SERVEUR ICI (kill -9, coupure électrique)

-- Redémarrage MariaDB
-- Aria :
--   ✅ Lit le aria_log
--   ✅ Rejoue les modifications
--   ✅ Table cohérente automatiquement
--   ✅ Pas de REPAIR nécessaire

-- MyISAM :
--   ❌ Table marquée "crashed"
--   ❌ Nécessite REPAIR TABLE
--   ❌ Risque de perte de données
```

**Logs au redémarrage** :

```
[Note] Aria engine: starting recovery
[Note] Aria: Recovering table: 'mydb/test_aria'
[Note] Aria: Log records applied: 42
[Note] Aria engine: recovery done
```

#### 2. Checksums de pages

```sql
-- Détection automatique des corruptions
-- Page lue depuis disque → Vérification checksum

-- Si corruption détectée :
-- [ERROR] Aria: Page 123 of table 'mydb/users' is corrupted
-- [ERROR] Calculated checksum: 0x12345678
-- [ERROR] Stored checksum: 0x87654321

-- Contrairement à MyISAM qui peut lire des données corrompues
-- sans le détecter
```

#### 3. Meilleure gestion du cache

```sql
-- Aria utilise un algorithme de cache plus sophistiqué
SHOW STATUS LIKE 'Aria_pagecache%';
-- +--------------------------------+----------+
-- | Variable_name                  | Value    |
-- +--------------------------------+----------+
-- | Aria_pagecache_blocks_not_flushed | 0     |
-- | Aria_pagecache_blocks_unused   | 8192     |
-- | Aria_pagecache_blocks_used     | 2048     |
-- | Aria_pagecache_read_requests   | 1000000  |
-- | Aria_pagecache_reads           | 5000     |
-- +--------------------------------+----------+

-- Hit ratio = (1 - reads / read_requests) = 99.5%
```

#### 4. Support des Row Format avancés

```sql
-- Aria supporte plusieurs formats de ligne
CREATE TABLE aria_dynamic (
    id INT,
    data TEXT
) ENGINE=Aria ROW_FORMAT=DYNAMIC;

CREATE TABLE aria_page (
    id INT,
    data VARCHAR(8000)
) ENGINE=Aria ROW_FORMAT=PAGE;  -- Format spécifique Aria
```

**Format PAGE** : Optimisé pour les grandes lignes avec meilleure gestion de la fragmentation.

### ❌ Limitations (similaires à MyISAM)

#### 1. Pas de transactions ACID complètes

```sql
-- Aria n'est PAS transactionnel
START TRANSACTION;
INSERT INTO aria_table VALUES (1, 'Data');
INSERT INTO aria_table VALUES (2, 'Data');
ROLLBACK;  -- ⚠️ N'A AUCUN EFFET

SELECT * FROM aria_table;
-- Les données sont persistées (auto-commit)
```

💡 **Important** : Le WAL d'Aria est utilisé uniquement pour le crash recovery, pas pour les transactions utilisateur.

#### 2. Table-level locking

```sql
-- Même limitation que MyISAM
-- Verrou d'écriture = table entière bloquée

-- Session 1
INSERT INTO aria_table VALUES (1, 'Data');
-- Verrou WRITE sur toute la table

-- Session 2
SELECT * FROM aria_table WHERE id = 99;
-- ⚠️ BLOQUÉE jusqu'à fin de l'INSERT
```

**Concurrence** : Faible pour les charges mixtes lecture/écriture.

#### 3. Pas de Foreign Keys

```sql
-- Foreign Keys ignorées (comme MyISAM)
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=Aria;

-- FK acceptée mais NON appliquée
DELETE FROM users WHERE id = 1;  -- Succès
-- orders.user_id=1 devient orphelin
```

#### 4. Pas de MVCC

Pas de Multi-Version Concurrency Control :
- Pas d'isolation entre lectures/écritures
- Pas de snapshots cohérents
- Readers bloquent writers et vice-versa

---

## Utilisation d'Aria dans MariaDB

### Tables système MariaDB

Depuis MariaDB 10.2, **toutes les tables système** utilisent Aria au lieu de MyISAM :

```sql
-- Vérifier les tables système
SELECT
    TABLE_NAME,
    ENGINE
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mysql'
ORDER BY TABLE_NAME;

-- +---------------------------+--------+
-- | TABLE_NAME                | ENGINE |
-- +---------------------------+--------+
-- | column_stats              | Aria   |
-- | columns_priv              | Aria   |
-- | db                        | Aria   |
-- | event                     | Aria   |
-- | func                      | Aria   |
-- | general_log               | CSV    |
-- | global_priv               | Aria   |
-- | gtid_slave_pos            | Aria   |
-- | help_category             | Aria   |
-- | help_keyword              | Aria   |
-- | help_relation             | Aria   |
-- | help_topic                | Aria   |
-- | index_stats               | Aria   |
-- | innodb_index_stats        | InnoDB |
-- | innodb_table_stats        | InnoDB |
-- | plugin                    | Aria   |
-- | proc                      | Aria   |
-- | procs_priv                | Aria   |
-- | proxies_priv              | Aria   |
-- | roles_mapping             | Aria   |
-- | servers                   | Aria   |
-- | slow_log                  | CSV    |
-- | table_stats               | Aria   |
-- | tables_priv               | Aria   |
-- | time_zone                 | Aria   |
-- | time_zone_leap_second     | Aria   |
-- | time_zone_name            | Aria   |
-- | time_zone_transition      | Aria   |
-- | time_zone_transition_type | Aria   |
-- | transaction_registry      | Aria   |
-- | user                      | Aria   |  -- Important !
-- +---------------------------+--------+
```

**Raison du choix d'Aria** :
- ✅ Crash-safe (critique pour tables système)
- ✅ Plus simple qu'InnoDB (moins d'overhead)
- ✅ Tables système modifiées rarement (table-lock acceptable)
- ✅ Pas besoin de transactions ACID complètes

### Tables temporaires internes

MariaDB utilise automatiquement Aria pour certaines tables temporaires internes :

```sql
-- Requête nécessitant une table temporaire
SELECT DISTINCT
    u.name,
    o.total
FROM users u
JOIN orders o ON u.id = o.user_id
ORDER BY total DESC;

-- MariaDB peut créer :
-- 1. Temporary table en mémoire (si petite)
-- 2. Temporary table Aria sur disque (si trop grande pour mémoire)

-- Configuration
[mysqld]
aria_used_for_temp_tables = ON  -- Par défaut
```

**Avantage** : Tables temporaires sur disque bénéficient du crash recovery d'Aria.

---

## Configuration d'Aria

### Variables système essentielles

```sql
-- Afficher la configuration Aria
SHOW VARIABLES LIKE 'aria%';

-- Variables critiques :
SHOW VARIABLES LIKE 'aria_pagecache_buffer_size';
-- +----------------------------+------------+
-- | Variable_name              | Value      |
-- +----------------------------+------------+
-- | aria_pagecache_buffer_size | 134217728  |  -- 128 MB par défaut
-- +----------------------------+------------+
```

### Configuration recommandée

```ini
[mysqld]
# ──────────────────────────────────────────────────
# ARIA CONFIGURATION
# ──────────────────────────────────────────────────

# Cache pour pages Aria (données + index)
# Recommandation : 128-512 MB (Aria utilisé peu en prod)
aria_pagecache_buffer_size = 256M

# Taille de page (doit correspondre au block size du FS)
aria_block_size = 8192  # 8 KB par défaut

# Taille des fichiers de log
aria_log_file_size = 1G

# Nombre de groupes de logs
aria_log_purge_type = immediate  # immediate, external, at_flush

# Interval de checkpoint (secondes)
aria_checkpoint_interval = 30

# Logs pour tables temporaires (ON recommandé)
aria_used_for_temp_tables = ON

# Checksums de page (ON recommandé)
aria_page_checksum = ON

# Statistiques
aria_stats_method = nulls_unequal  # nulls_equal, nulls_unequal, nulls_ignored

# Récupération
aria_recover_options = BACKUP,QUICK  # Options pour recovery automatique

# Synchronisation
aria_sync_log_dir = NEWFILE  # Quand synchroniser le répertoire de log
```

### Dimensionnement du cache

**Règles de dimensionnement** :

```sql
-- Calculer l'utilisation actuelle
SELECT
    SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 AS aria_total_mb
FROM information_schema.TABLES
WHERE ENGINE = 'Aria'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema');

-- Si Aria utilisé seulement pour tables système :
-- aria_pagecache_buffer_size = 128 MB (suffisant)

-- Si Aria utilisé pour tables applicatives :
-- aria_pagecache_buffer_size = 10-20% de la taille totale des tables Aria
```

**Monitoring du cache** :

```sql
-- Statistiques du cache
SHOW STATUS LIKE 'Aria_pagecache%';

-- Calcul du hit ratio
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
         WHERE VARIABLE_NAME = 'Aria_pagecache_reads') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
         WHERE VARIABLE_NAME = 'Aria_pagecache_read_requests')
    )) * 100 AS hit_ratio_pct;

-- Objectif : > 95%
```

### Gestion des logs

```sql
-- Vérifier l'état des logs
SHOW ENGINE ARIA LOGS;

-- Variables de configuration des logs
SHOW VARIABLES LIKE 'aria_log%';
-- +---------------------------+------------------+
-- | Variable_name             | Value            |
-- +---------------------------+------------------+
-- | aria_log_file_size        | 1073741824       |  -- 1 GB
-- | aria_log_purge_type       | immediate        |
-- | aria_log_dir_path         | (datadir)        |
-- +---------------------------+------------------+

-- Forcer un checkpoint (flush des modifications)
FLUSH TABLES;

-- Purger les anciens logs (si purge_type=external)
-- shell> aria_chk --zerofill-keep-lsn /path/to/aria_log_control
```

**Types de purge** :

| Type | Description | Usage |
|------|-------------|-------|
| `immediate` | Purge automatique après checkpoint | **Recommandé** (par défaut) |
| `external` | Purge manuelle via aria_chk | Contrôle fin, avancé |
| `at_flush` | Purge lors du flush | Compromis |

---

## Maintenance et diagnostics

### Vérification de l'intégrité

```sql
-- Check d'une table Aria
CHECK TABLE aria_table;

-- Check approfondi
CHECK TABLE aria_table EXTENDED;

-- Analyser et reconstruire les statistiques
ANALYZE TABLE aria_table;
```

**Outils en ligne de commande** :

```bash
# Vérifier une table (serveur arrêté)
aria_chk /var/lib/mysql/mydb/tablename.MAI

# Check avec statistiques
aria_chk --check --statistics /var/lib/mysql/mydb/tablename.MAI

# Informations détaillées
aria_chk --description /var/lib/mysql/mydb/tablename.MAI
```

### Réparation

```sql
-- Réparation d'une table (rarement nécessaire avec Aria)
REPAIR TABLE aria_table;

-- Réparation avec reconstruction
REPAIR TABLE aria_table EXTENDED;
```

```bash
# Réparation en ligne de commande
aria_chk --recover /var/lib/mysql/mydb/tablename

# Réparation sûre
aria_chk --safe-recover /var/lib/mysql/mydb/tablename
```

### Optimisation

```sql
-- Défragmenter et reconstruire
OPTIMIZE TABLE aria_table;

-- Résultat :
-- +-------------------+----------+----------+----------+
-- | Table             | Op       | Msg_type | Msg_text |
-- +-------------------+----------+----------+----------+
-- | mydb.aria_table   | optimize | status   | OK       |
-- +-------------------+----------+----------+----------+
```

### Monitoring

```sql
-- Statistiques globales Aria
SHOW STATUS LIKE 'Aria%';

-- Variables importantes à surveiller :
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'Aria_pagecache_read_requests',
    'Aria_pagecache_reads',
    'Aria_pagecache_write_requests',
    'Aria_pagecache_writes',
    'Aria_transaction_log_syncs'
);

-- Taille des tables Aria
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE ENGINE = 'Aria'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;
```

---

## Comparaison : Aria vs MyISAM vs InnoDB

### Tableau comparatif détaillé

| Caractéristique | MyISAM | Aria | InnoDB |
|-----------------|--------|------|--------|
| **Fiabilité** |
| Crash recovery | ❌ Non | ✅ Oui (WAL) | ✅ Oui (Redo/Undo Log) |
| Checksums | ❌ Non | ✅ Oui | ✅ Oui (optionnel) |
| Corruption après crash | ⚠️ Fréquente | ✅ Rare | ✅ Très rare |
| **Transactions** |
| ACID | ❌ Non | ❌ Non | ✅ Oui |
| MVCC | ❌ Non | ❌ Non | ✅ Oui |
| Savepoints | ❌ Non | ❌ Non | ✅ Oui |
| **Concurrence** |
| Locking | Table | Table | Row |
| Readers bloquent writers | ✅ Oui | ✅ Oui | ❌ Non (MVCC) |
| **Intégrité** |
| Foreign Keys | ❌ Non | ❌ Non | ✅ Oui |
| Triggers | ✅ Oui | ✅ Oui | ✅ Oui |
| **Performances** |
| Lecture seule | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Écriture simple | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Haute concurrence | ⭐ | ⭐ | ⭐⭐⭐⭐⭐ |
| **Stockage** |
| Compression | ✅ myisampack | ✅ aria_pack | ✅ ROW_FORMAT=COMPRESSED |
| Full-Text Search | ✅ Oui | ✅ Oui | ✅ Oui (depuis 10.0) |
| **Maintenance** |
| Complexité | Simple | Simple | Complexe |
| Overhead mémoire | Faible | Faible | Élevé |

### Benchmark comparatif

```sql
-- Test : INSERT séquentiel (10 000 lignes)
-- MyISAM : 1.2 secondes
-- Aria : 1.3 secondes (overhead log WAL)
-- InnoDB : 1.8 secondes (overhead transactions)

-- Test : SELECT avec WHERE (1000 requêtes)
-- MyISAM : 0.8 secondes
-- Aria : 0.8 secondes
-- InnoDB : 0.9 secondes

-- Test : Concurrence (100 threads, 50% read, 50% write)
-- MyISAM : 50 req/sec (table lock bottleneck)
-- Aria : 52 req/sec (similar)
-- InnoDB : 4800 req/sec (row-level locking)
```

💡 **Conclusion** : Aria ≈ MyISAM pour performance, mais avec crash-safety. InnoDB supérieur en concurrence.

---

## Cas d'usage d'Aria (limités)

### 1. Tables système (usage principal)

```sql
-- MariaDB utilise Aria pour toutes les tables système
-- Raison : Crash-safe + Performance acceptable + Simplicité

-- Exemple : Table mysql.user
SHOW CREATE TABLE mysql.user\G
-- Engine: Aria
-- Transactional: 0
-- Page_checksum: 1
```

**Justification** :
- Modifications rares (privilèges changent peu)
- Taille petite (quelques MB max)
- Criticité élevée (corruption inacceptable)
- Pas besoin de transactions ACID

### 2. Tables temporaires de grande taille

```sql
-- Tables temporaires qui dépassent la mémoire
CREATE TEMPORARY TABLE large_temp (
    id INT,
    data TEXT
) ENGINE=Aria;

-- INSERT de millions de lignes
INSERT INTO large_temp SELECT ...;

-- Avantages :
-- • Crash recovery (si serveur crash pendant calcul)
-- • Pas d'overhead transactionnel InnoDB
-- • Meilleur que MyISAM (crash-safe)
```

### 3. Tables de logs append-only (avec réserves)

```sql
-- Logs applicatifs (écriture uniquement, lecture rare)
CREATE TABLE app_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    level VARCHAR(10),
    message TEXT
) ENGINE=Aria;

-- Avantages :
-- • Pas besoin de transactions
-- • Crash-safe (contrairement à MyISAM)
-- • Performance acceptable

-- ⚠️ Mais InnoDB reste meilleur :
-- • Meilleure concurrence
-- • MVCC (pas de blocage lecteurs)
-- • Plus robuste
```

### 4. Tables de test et développement

```sql
-- Environnement de développement local
-- Aria acceptable car :
-- • Moins d'overhead qu'InnoDB
-- • Crash-safe (contrairement à MyISAM)
-- • Configuration simplifiée

-- ⚠️ Production : TOUJOURS InnoDB
```

---

## Migration vers/depuis Aria

### Conversion MyISAM → Aria

```sql
-- Identifier les tables MyISAM
SELECT
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND ENGINE = 'MyISAM';

-- Conversion simple
ALTER TABLE myisam_table ENGINE=Aria;

-- Conversion avec options
ALTER TABLE myisam_table
    ENGINE=Aria
    ROW_FORMAT=PAGE
    TRANSACTIONAL=0;  -- 0 pour non-transactionnel (normal)

-- Vérifier
SHOW CREATE TABLE myisam_table\G
```

**Considérations** :
- ✅ Migration simple (même structure de données)
- ✅ Gain immédiat : crash recovery
- ⚠️ Limitations identiques (pas de transactions, table-lock)
- 💡 **Recommandation** : Migrer vers **InnoDB** plutôt qu'Aria

### Conversion Aria → InnoDB (recommandé)

```sql
-- Pour tables applicatives : InnoDB supérieur
ALTER TABLE aria_table ENGINE=InnoDB;

-- Vérifier l'amélioration
-- Avant (Aria) :
-- • Table-level locking
-- • Pas de transactions
-- • Concurrence faible

-- Après (InnoDB) :
-- • Row-level locking
-- • Transactions ACID
-- • Haute concurrence
-- • MVCC
```

**Stratégie de migration** :

```sql
-- 1. Test sur copie
CREATE TABLE users_innodb LIKE users_aria;
ALTER TABLE users_innodb ENGINE=InnoDB;
INSERT INTO users_innodb SELECT * FROM users_aria;

-- 2. Benchmark
-- (utiliser sysbench ou mysqlslap)

-- 3. Migration production (zero-downtime avec gh-ost)
gh-ost \
    --host=localhost \
    --database=mydb \
    --table=users_aria \
    --alter="ENGINE=InnoDB" \
    --execute
```

### Conversion InnoDB → Aria (déconseillé)

```sql
-- ⚠️ Conversion régressive (perte de fonctionnalités)
-- À éviter sauf cas TRÈS spécifique

-- Si vraiment nécessaire :
ALTER TABLE innodb_table ENGINE=Aria;

-- Pertes :
-- ❌ Transactions ACID
-- ❌ Row-level locking
-- ❌ Foreign Keys (supprimées)
-- ❌ MVCC
```

💡 **Recommandation forte** : Ne JAMAIS faire cette conversion en production.

---

## Problèmes courants et solutions

### Problème 1 : Logs Aria qui grossissent

```sql
-- Symptôme : Disque plein, logs Aria très volumineux
-- shell> du -sh /var/lib/mysql/aria_log.*
-- 10G  aria_log.00000001
-- 10G  aria_log.00000002
-- ...

-- Cause : Purge automatique désactivée ou défaillante
SHOW VARIABLES LIKE 'aria_log_purge_type';

-- Solution 1 : Forcer un checkpoint
FLUSH TABLES;

-- Solution 2 : Changer la politique de purge
SET GLOBAL aria_log_purge_type = immediate;

-- Solution 3 : Réduire la taille des fichiers de log
-- my.cnf :
-- aria_log_file_size = 512M  -- Au lieu de 1G
```

### Problème 2 : Performance cache insuffisant

```sql
-- Symptôme : Hit ratio faible
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
         WHERE VARIABLE_NAME = 'Aria_pagecache_reads') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
         WHERE VARIABLE_NAME = 'Aria_pagecache_read_requests')
    )) * 100 AS hit_ratio;
-- 85% (< 95% = problème)

-- Solution : Augmenter le cache
SET GLOBAL aria_pagecache_buffer_size = 512 * 1024 * 1024;  -- 512 MB

-- Permanent dans my.cnf :
-- [mysqld]
-- aria_pagecache_buffer_size = 512M
```

### Problème 3 : Table corrompue malgré Aria

```sql
-- Symptôme (rare) :
-- ERROR: Table './mydb/aria_table' is marked as crashed

-- Causes possibles :
-- • Défaillance matérielle (disque)
-- • Bug logiciel (très rare)
-- • Arrêt brutal pendant recovery

-- Solution :
CHECK TABLE aria_table;
REPAIR TABLE aria_table;

-- Si échec :
-- shell> systemctl stop mariadb
-- shell> aria_chk --recover /var/lib/mysql/mydb/aria_table
-- shell> systemctl start mariadb
```

### Problème 4 : Migration MyISAM → Aria échoue

```sql
-- Erreur possible :
-- ERROR: Can't create table (errno: 140)

-- Causes :
-- • Espace disque insuffisant (logs Aria)
-- • Permissions fichiers
-- • Taille table trop grande

-- Solutions :
-- 1. Vérifier l'espace disque
-- shell> df -h

-- 2. Vérifier les permissions
-- shell> chown -R mysql:mysql /var/lib/mysql

-- 3. Migrer en plusieurs étapes (si très grosse table)
CREATE TABLE aria_table_new LIKE myisam_table;
ALTER TABLE aria_table_new ENGINE=Aria;
INSERT INTO aria_table_new SELECT * FROM myisam_table LIMIT 1000000;
-- Répéter par batch
```

---

## ✅ Points clés à retenir

1. **Aria = MyISAM + Crash Recovery** : Amélioration majeure mais toujours pas transactionnel.

2. **Usage principal : Tables système** : MariaDB utilise Aria pour toutes les tables système (mysql.user, etc.).

3. **Write-Ahead Log (WAL)** : Mécanisme clé permettant la crash recovery automatique.

4. **Table-level locking** : Même limitation que MyISAM, mauvaise concurrence.

5. **Pas de transactions ACID** : Ne remplace PAS InnoDB pour applications transactionnelles.

6. **Checksums intégrés** : Détection automatique des corruptions silencieuses.

7. **Configuration simple** : Overhead minimal, 128-512 MB de cache suffisant.

8. **Migration MyISAM → Aria** : Gain de crash-safety, mais mieux vaut migrer vers InnoDB.

9. **Aria ≠ Solution moderne** : Pour nouvelles applications, utiliser InnoDB directement.

10. **Maintenance facile** : Moins de corruptions que MyISAM, outils similaires (aria_chk).

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Aria Storage Engine](https://mariadb.com/kb/en/aria-storage-engine/)
- [📖 Aria System Variables](https://mariadb.com/kb/en/aria-system-variables/)
- [📖 Aria Status Variables](https://mariadb.com/kb/en/aria-server-status-variables/)
- [📖 aria_chk Tool](https://mariadb.com/kb/en/aria_chk/)
- [📖 Aria FAQ](https://mariadb.com/kb/en/aria-faq/)

### Articles et analyses

- [Monty Says: Aria Storage Engine](https://monty-says.blogspot.com/aria-storage-engine)
- [MariaDB Aria vs MyISAM: What's the Difference?](https://mariadb.org/aria-vs-myisam/)
- [Understanding Aria Transaction Log](https://mariadb.com/kb/en/aria-transaction-log/)

### Outils et scripts

- **aria_chk** : Outil de maintenance Aria (similaire à myisamchk)
- **aria_dump_log** : Analyse des logs Aria
- **aria_ftdump** : Full-Text index dump
- **aria_read_log** : Lecture des logs de transaction

---

## ➡️ Section suivante

**[7.5 ColumnStore : Analytique et OLAP](/07-moteurs-de-stockage/05-columnstore.md)** : Découverte du moteur columnar MariaDB optimisé pour l'analytique, le data warehousing et les requêtes d'agrégation massives.

Puis nous continuerons avec :
- **7.6** : Moteur S3 pour archivage cloud
- **7.7** : Vector/HNSW pour recherche vectorielle IA 🆕
- **7.8** : Comparaison et choix du moteur approprié

---

**📌 Mémo DBA** : "Aria est utile pour les tables système MariaDB, mais pour vos applications, utilisez InnoDB. Aria n'est pas une solution moderne pour la production."

**🎯 Règle d'or** : Si vous hésitez entre Aria et InnoDB → Choisissez InnoDB. Aria n'apporte un avantage que dans des cas très spécifiques (tables système, quelques cas edge).

⏭️ [ColumnStore : Analytique et OLAP](/07-moteurs-de-stockage/05-columnstore.md)

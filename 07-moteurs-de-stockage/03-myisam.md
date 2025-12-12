🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 MyISAM : Moteur legacy

> **Niveau** : Avancé
> **Durée estimée** : 1-2 heures
> **Prérequis** : Section 7.1 (Architecture Pluggable), Section 7.2 (InnoDB), concepts de systèmes de fichiers

> **Public cible** : DBA, Architectes, Ingénieurs migration legacy

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture et les caractéristiques de MyISAM
- Identifier les limitations critiques qui le rendent obsolète
- Analyser les risques liés à l'utilisation de MyISAM en production
- Évaluer les rares cas d'usage encore légitimes
- Planifier et exécuter une migration MyISAM → InnoDB
- Diagnostiquer et réparer les corruptions MyISAM
- Comprendre le contexte historique et l'évolution vers InnoDB

---

## Introduction

🔄 **Statut** : MyISAM est un moteur **legacy déprécié** qui ne devrait plus être utilisé pour les nouvelles applications. Il a été le moteur par défaut de MySQL jusqu'à la version 5.5 (2010), mais a été remplacé par InnoDB depuis.

### Pourquoi cette section existe-t-elle ?

Bien que MyISAM soit obsolète, il reste présent dans de nombreux systèmes legacy. Les DBA doivent :
- **Identifier** les tables MyISAM existantes
- **Comprendre** les risques associés
- **Planifier** les migrations vers InnoDB
- **Maintenir** temporairement les tables legacy (jusqu'à migration complète)

### Contexte historique

```
Timeline MySQL/MariaDB Storage Engines :
┌────────────────────────────────────────────────────────────┐
│ 1996   ISAM (Indexed Sequential Access Method)             │
│        ↓                                                   │
│ 2000   MyISAM (My ISAM, amélioré)                          │
│        • Moteur par défaut MySQL 3.23 → 5.5                │
│        • Très rapide en lecture seule                      │
│        • Pas de transactions                               │
│        ↓                                                   │
│ 2001   InnoDB (Innobase Oy)                                │
│        • Transactions ACID                                 │
│        • Foreign Keys                                      │
│        • Row-level locking                                 │
│        ↓                                                   │
│ 2008   Aria (MariaDB)                                      │
│        • "MyISAM with crash recovery"                      │
│        • Tables système MariaDB                            │
│        ↓                                                   │
│ 2010   MySQL 5.5 : InnoDB devient par défaut               │
│        ↓                                                   │
│ 2017   MariaDB 10.2 : InnoDB par défaut                    │
│        ↓                                                   │
│ 2025   MariaDB 11.8 : MyISAM toujours présent              │
│        mais fortement déconseillé                          │
└────────────────────────────────────────────────────────────┘
```

⚠️ **Avertissement** : Cette section documente MyISAM principalement à des fins de migration et de maintenance legacy. **N'utilisez pas MyISAM pour de nouvelles tables.**

---

## Architecture de MyISAM

### Structure de fichiers

Chaque table MyISAM est composée de **3 fichiers** :

```
/var/lib/mysql/mydb/
├── users.frm         # Format/définition de table (structure)
├── users.MYD         # MyISAM Data (données)
└── users.MYI         # MyISAM Index (index)
```

**Détails des fichiers** :

| Fichier | Extension | Contenu | Taille typique |
|---------|-----------|---------|----------------|
| Format | `.frm` | Structure table (colonnes, types) | Quelques KB |
| Data | `.MYD` | Lignes de données | Variable (taille des données) |
| Index | `.MYI` | Tous les index (B-Tree) | Variable (nombre/taille index) |

```sql
-- Créer une table MyISAM
CREATE TABLE legacy_data (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_name (name)
) ENGINE=MyISAM;

-- Vérifier les fichiers créés
-- shell> ls -lh /var/lib/mysql/mydb/legacy_data.*
-- -rw-rw---- 1 mysql mysql  8.5K  legacy_data.frm
-- -rw-rw---- 1 mysql mysql    0   legacy_data.MYD
-- -rw-rw---- 1 mysql mysql  1.0K  legacy_data.MYI
```

### Organisation des données

#### Format de stockage fixe vs dynamique

MyISAM supporte deux formats principaux :

**1. Format fixe (FIXED)** :
- Toutes les colonnes ont une longueur fixe
- Pas de VARCHAR, TEXT, BLOB
- Accès très rapide (calcul direct de l'offset)

```sql
CREATE TABLE fixed_format (
    id INT,              -- 4 bytes
    code CHAR(10),       -- 10 bytes
    value DECIMAL(10,2)  -- 5 bytes
) ENGINE=MyISAM ROW_FORMAT=FIXED;

-- Taille par ligne : 4 + 10 + 5 = 19 bytes
-- Ligne N à l'offset : N × 19 bytes
```

**2. Format dynamique (DYNAMIC)** :
- Colonnes de longueur variable (VARCHAR, TEXT, BLOB)
- Compact, mais accès plus lent
- Fragmentation possible

```sql
CREATE TABLE dynamic_format (
    id INT,
    name VARCHAR(255),   -- Variable
    data TEXT            -- Variable
) ENGINE=MyISAM ROW_FORMAT=DYNAMIC;

-- Taille variable par ligne
-- Fragmentation après UPDATE si nouvelle taille > ancienne
```

**3. Format compressé (COMPRESSED)** :
- Read-only après compression
- Gain d'espace maximal
- Nécessite l'outil `myisampack`

```bash
# Compresser une table MyISAM
myisampack /var/lib/mysql/mydb/archive_table
myisamchk -rq /var/lib/mysql/mydb/archive_table

# Table devient read-only mais très compacte
```

### Architecture des index

MyISAM utilise des **B-Tree index** pour tous les types d'index.

```
Structure d'un index MyISAM (.MYI file) :
┌─────────────────────────────────────────┐
│           Header (métadonnées)          │
│  • Nombre d'index                       │
│  • Statistiques                         │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│         Index 1 (PRIMARY KEY)           │
│                                         │
│         Root Node                       │
│        ┌────┬────┬────┐                 │
│        │ 10 │ 20 │ 30 │                 │
│        └─┬──┴──┬─┴──┬─┘                 │
│          ↓     ↓    ↓                   │
│       Branch Nodes                      │
│          ↓     ↓    ↓                   │
│       Leaf Nodes                        │
│    (pointeurs vers .MYD)                │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│         Index 2 (idx_name)              │
│         (même structure B-Tree)         │
└─────────────────────────────────────────┘
```

💡 **Différence clé avec InnoDB** :
- **MyISAM** : Index pointent vers offset physique dans .MYD
- **InnoDB** : Index secondaires pointent vers clé primaire (clustering)

---

## Caractéristiques de MyISAM

### ❌ Limitations majeures

#### 1. Pas de support transactionnel (ACID)

```sql
-- MyISAM : Pas de transactions
START TRANSACTION;
INSERT INTO myisam_table VALUES (1, 'Alice');
INSERT INTO myisam_table VALUES (2, 'Bob');
ROLLBACK;  -- ⚠️ N'A AUCUN EFFET !

SELECT * FROM myisam_table;
-- +----+-------+
-- | id | name  |
-- +----+-------+
-- |  1 | Alice |
-- |  2 | Bob   |
-- +----+-------+
-- Les données sont déjà commitées automatiquement
```

**Conséquence** : Impossible de garantir l'atomicité des opérations.

```sql
-- Exemple problématique : Transfert bancaire
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Exécuté
-- CRASH SERVEUR ICI
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Jamais exécuté
COMMIT;

-- Résultat : 100€ perdus ! (pas de rollback possible)
```

#### 2. Verrouillage au niveau table (Table-level locking)

```sql
-- Session 1
INSERT INTO myisam_table VALUES (1, 'Data');
-- → Verrou en ÉCRITURE sur TOUTE la table

-- Session 2 (en parallèle)
SELECT * FROM myisam_table WHERE id = 99;
-- → BLOQUÉE ! Doit attendre la fin de l'INSERT

-- Session 3
UPDATE myisam_table SET value = 'X' WHERE id = 50;
-- → BLOQUÉE également
```

**Types de verrous MyISAM** :

| Opération | Verrou | Bloque lectures ? | Bloque écritures ? |
|-----------|--------|-------------------|-------------------|
| `SELECT` | READ | ❌ Non | ✅ Oui |
| `INSERT` | WRITE | ✅ Oui | ✅ Oui |
| `UPDATE` | WRITE | ✅ Oui | ✅ Oui |
| `DELETE` | WRITE | ✅ Oui | ✅ Oui |

**Impact sur la concurrence** :

```sql
-- Benchmark concurrence (100 threads simultanés)
-- MyISAM :
--   Throughput : 50 req/sec
--   (goulot d'étranglement : table lock)

-- InnoDB (row-level locking) :
--   Throughput : 5000 req/sec
--   (100× plus rapide en haute concurrence)
```

💡 **Cas où table-lock est acceptable** : Charges de travail **lecture seule** avec écritures batch (ETL, data warehouse read-only).

#### 3. Pas de Foreign Keys

```sql
-- Tentative de créer une FK avec MyISAM
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=MyISAM;

-- ⚠️ Accepté SANS erreur, mais FK est IGNORÉE !
SHOW CREATE TABLE orders\G
-- Pas de FOREIGN KEY dans la définition

-- Conséquence : Intégrité référentielle NON garantie
DELETE FROM users WHERE id = 1;  -- Succès
-- orders.user_id=1 reste orphelin (données incohérentes)
```

#### 4. Pas de crash recovery automatique

```sql
-- Scénario : Crash serveur pendant un UPDATE
UPDATE myisam_table SET value = 'New' WHERE id BETWEEN 1 AND 1000000;
-- CRASH (coupure électrique, kill -9, etc.)

-- Redémarrage MariaDB
-- MyISAM :
--   ⚠️ Table potentiellement CORROMPUE
--   ⚠️ Index incohérents
--   ⚠️ Nécessite REPAIR TABLE manuel
```

**Logs d'erreur typiques** :

```
[ERROR] /var/lib/mysql/mydb/myisam_table.MYI: Table is marked as crashed
[ERROR] /var/lib/mysql/mydb/myisam_table: Table is marked as crashed and should be repaired
```

**Réparation nécessaire** :

```sql
-- Vérifier l'intégrité
CHECK TABLE myisam_table;
-- +-------------------------+-------+----------+----------------------------+
-- | Table                   | Op    | Msg_type | Msg_text                   |
-- +-------------------------+-------+----------+----------------------------+
-- | mydb.myisam_table       | check | warning  | Table is marked as crashed |
-- +-------------------------+-------+----------+----------------------------+

-- Réparer (peut perdre des données)
REPAIR TABLE myisam_table;
-- ou en ligne de commande :
-- myisamchk --recover /var/lib/mysql/mydb/myisam_table
```

⚠️ **Risques de la réparation** :
- Perte de lignes corrompues
- Reconstruction d'index (peut être long)
- Downtime de la table

#### 5. Fragmentation des données

```sql
-- Fragmentation due aux UPDATE de taille variable
CREATE TABLE logs (
    id INT PRIMARY KEY,
    message VARCHAR(1000)
) ENGINE=MyISAM;

INSERT INTO logs VALUES (1, 'Short');
INSERT INTO logs VALUES (2, 'Another short message');

-- UPDATE avec message plus long
UPDATE logs SET message = REPEAT('Long message ', 100) WHERE id = 1;
-- → MyISAM ne peut pas utiliser l'espace d'origine
-- → Écrit à la fin du fichier .MYD
-- → Ancien espace marqué comme "libre" mais fragmenté

-- Résultat après plusieurs UPDATE :
-- .MYD file :
--   [Row1_old (libre)] [Row2] [UNUSED] [Row1_new] [UNUSED] ...
--   → Gaspillage d'espace, performances dégradées
```

**Solution** : `OPTIMIZE TABLE` (reconstruction complète)

```sql
-- Avant optimisation
SELECT
    TABLE_NAME,
    DATA_LENGTH,
    DATA_FREE,  -- Espace fragmenté
    ROUND(DATA_FREE / DATA_LENGTH * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_NAME = 'logs';
-- DATA_FREE : 5 MB (50% fragmenté)

-- Optimiser (réorganisation complète)
OPTIMIZE TABLE logs;
-- Note: Pendant OPTIMIZE, table verrouillée (indisponible)

-- Après optimisation
-- DATA_FREE : 0 (plus de fragmentation)
```

### ✅ Quelques avantages (devenus obsolètes)

#### 1. Simplicité de l'implémentation

MyISAM est conceptuellement plus simple qu'InnoDB :
- Pas de gestion de transactions
- Pas de MVCC
- Pas de redo/undo logs
- Structure de fichiers directe

**Mais** : Cette simplicité ne compense plus les limitations critiques.

#### 2. Performances en lecture seule (sur charges spécifiques)

```sql
-- Benchmark : SELECT simple sur table sans écritures
-- Conditions : Table chargée en cache, pas de concurrence

-- MyISAM (table lock) : 10,000 req/sec
-- InnoDB (row lock) : 9,500 req/sec
-- Différence : ~5% (négligeable)
```

💡 **Réalité moderne** : Avec les optimisations InnoDB (Adaptive Hash Index, Buffer Pool), l'écart de performance en lecture pure est négligeable, et InnoDB reste supérieur dès qu'il y a de la concurrence.

#### 3. Compression avec myisampack

```bash
# Compression d'une table MyISAM
myisampack /var/lib/mysql/archive/old_data

# Résultat :
#   - Taille réduite de 50-80%
#   - Table devient read-only
#   - Decompression à la volée lors des lectures
```

**Alternative moderne** : InnoDB `ROW_FORMAT=COMPRESSED` offre une compression comparable sans perte de fonctionnalités.

#### 4. Full-Text Search (historique)

MyISAM a été le premier moteur à supporter Full-Text Search dans MySQL.

```sql
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT(title, content)
) ENGINE=MyISAM;

SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('MariaDB database' IN NATURAL LANGUAGE MODE);
```

🔄 **Mais** : Depuis MySQL 5.6 / MariaDB 10.0, **InnoDB supporte également Full-Text Search** avec les mêmes fonctionnalités.

```sql
-- Full-Text avec InnoDB (moderne)
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT(title, content)
) ENGINE=InnoDB;  -- Fonctionne parfaitement
```

---

## Cas d'usage encore légitimes (très rares)

### 1. Tables système internes (MariaDB utilise Aria)

MariaDB utilise **Aria** (et non MyISAM) pour les tables système :

```sql
SELECT
    TABLE_NAME,
    ENGINE
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mysql';
-- +------------------+--------+
-- | TABLE_NAME       | ENGINE |
-- +------------------+--------+
-- | user             | Aria   |
-- | db               | Aria   |
-- | tables_priv      | Aria   |
-- +------------------+--------+
```

💡 Aria = MyISAM amélioré avec crash recovery.

### 2. Tables temporaires de calcul (hors production)

```sql
-- Calcul intermédiaire dans un script ETL
CREATE TEMPORARY TABLE temp_aggregates (
    category VARCHAR(50),
    total DECIMAL(15,2)
) ENGINE=MyISAM;

-- Avantage : Pas besoin de redo/undo log (overhead minimal)
-- Inconvénient : Perte en cas de crash (acceptable pour temp)
```

**Mais** : InnoDB avec `innodb_flush_log_at_trx_commit=2` est presque aussi rapide.

### 3. Archivage read-only avec compression

```sql
-- Table d'archive ancienne (jamais modifiée)
CREATE TABLE archive_2020 (
    id INT,
    data TEXT
) ENGINE=MyISAM;

-- Compresser avec myisampack
-- shell> myisampack /var/lib/mysql/archive/archive_2020
-- shell> myisamchk -rq /var/lib/mysql/archive/archive_2020
```

**Alternative moderne supérieure** : Moteur **S3** (MariaDB 11.x) pour archivage sur cloud.

### 4. Import/export rapide (sans transactions)

```sql
-- Import massif de données (ETL, migration)
LOAD DATA INFILE '/tmp/huge_file.csv'
INTO TABLE staging_myisam;
-- Avantage : Pas d'overhead transactionnel

-- Puis conversion vers InnoDB
CREATE TABLE staging_innodb LIKE staging_myisam;
ALTER TABLE staging_innodb ENGINE=InnoDB;
INSERT INTO staging_innodb SELECT * FROM staging_myisam;
```

**Mais** : InnoDB avec `innodb_flush_log_at_trx_commit=0` pendant l'import est comparable.

---

## Diagnostiquer et réparer MyISAM

### Vérification de l'intégrité

```sql
-- Check simple
CHECK TABLE myisam_table;

-- Check approfondi (plus lent)
CHECK TABLE myisam_table EXTENDED;

-- Check très approfondi (très lent)
CHECK TABLE myisam_table MEDIUM;

-- Check avec métadonnées
CHECK TABLE myisam_table CHANGED;
```

**Outils en ligne de commande** :

```bash
# Vérifier une table (serveur arrêté)
myisamchk /var/lib/mysql/mydb/tablename.MYI

# Check rapide
myisamchk --check /var/lib/mysql/mydb/tablename.MYI

# Check complet
myisamchk --check --extend-check /var/lib/mysql/mydb/tablename.MYI

# Informations sur la table
myisamchk --description /var/lib/mysql/mydb/tablename.MYI
```

### Réparation des corruptions

```sql
-- Réparation standard
REPAIR TABLE myisam_table;

-- Réparation rapide (utilise le sort buffer)
REPAIR TABLE myisam_table QUICK;

-- Réparation étendue (reconstruction complète)
REPAIR TABLE myisam_table EXTENDED;

-- Réparation avec utilisation de clé unique
REPAIR TABLE myisam_table USE_FRM;
```

**Ligne de commande** :

```bash
# Réparation standard
myisamchk --recover /var/lib/mysql/mydb/tablename

# Réparation sûre (plus lente, meilleure garantie)
myisamchk --safe-recover /var/lib/mysql/mydb/tablename

# Réparation avec sort buffer (plus rapide)
myisamchk --recover --sort-buffer-size=256M /var/lib/mysql/mydb/tablename
```

### Optimisation et défragmentation

```sql
-- Optimiser (défragmenter + reconstruire index)
OPTIMIZE TABLE myisam_table;

-- Analyser (recalculer statistiques)
ANALYZE TABLE myisam_table;
```

```bash
# Analyse avec myisamchk
myisamchk --analyze /var/lib/mysql/mydb/tablename

# Sort d'index
myisamchk --sort-index /var/lib/mysql/mydb/tablename

# Sort d'index + données
myisamchk --sort-records=1 /var/lib/mysql/mydb/tablename
# (1 = premier index)
```

### Récupération après crash

**Procédure standard** :

```bash
# 1. Arrêter MariaDB
systemctl stop mariadb

# 2. Identifier les tables corrompues
myisamchk --check /var/lib/mysql/*/*.MYI | grep -i "crashed"

# 3. Réparer chaque table
myisamchk --recover /var/lib/mysql/mydb/table1.MYI
myisamchk --recover /var/lib/mysql/mydb/table2.MYI

# 4. Vérifier la réparation
myisamchk --check /var/lib/mysql/mydb/table1.MYI

# 5. Redémarrer MariaDB
systemctl start mariadb
```

**Script automatique** :

```bash
#!/bin/bash
# repair_all_myisam.sh

DATADIR="/var/lib/mysql"

echo "Stopping MariaDB..."
systemctl stop mariadb

echo "Repairing MyISAM tables..."
find $DATADIR -name "*.MYI" | while read table; do
    echo "Checking $table..."
    myisamchk --check "$table" 2>&1 | grep -i "crashed" && {
        echo "Repairing $table..."
        myisamchk --recover "$table"
    }
done

echo "Starting MariaDB..."
systemctl start mariadb
echo "Repair complete."
```

---

## Migration MyISAM → InnoDB

### Étape 1 : Audit des tables existantes

```sql
-- Identifier toutes les tables MyISAM
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
    TABLE_COLLATION
FROM information_schema.TABLES
WHERE ENGINE = 'MyISAM'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema')
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;
```

**Analyser les dépendances** :

```sql
-- Vérifier les Foreign Keys (doivent être recréées)
SELECT
    TABLE_NAME,
    CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'mydb'
  AND REFERENCED_TABLE_NAME IS NOT NULL;

-- Vérifier les Full-Text indexes
SELECT
    TABLE_NAME,
    INDEX_NAME,
    INDEX_TYPE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'mydb'
  AND INDEX_TYPE = 'FULLTEXT';
```

### Étape 2 : Tester sur un environnement de développement

```sql
-- Créer une copie de test
CREATE TABLE users_innodb LIKE users_myisam;
ALTER TABLE users_innodb ENGINE=InnoDB;

-- Copier les données
INSERT INTO users_innodb SELECT * FROM users_myisam;

-- Comparer les données
SELECT COUNT(*) FROM users_myisam;
SELECT COUNT(*) FROM users_innodb;

-- Vérifier les index
SHOW CREATE TABLE users_myisam\G
SHOW CREATE TABLE users_innodb\G
```

**Tester les performances** :

```sql
-- Benchmark lecture
SELECT BENCHMARK(10000, (SELECT * FROM users_myisam WHERE id = 42));
SELECT BENCHMARK(10000, (SELECT * FROM users_innodb WHERE id = 42));

-- Benchmark écriture
-- (créer script de test avec mysqlslap ou sysbench)
```

### Étape 3 : Planifier la migration

**Stratégies de migration** :

#### A. Migration directe (ALTER TABLE)

```sql
-- Simple mais BLOQUANT pour grandes tables
ALTER TABLE users ENGINE=InnoDB;
-- ⚠️ Verrou exclusif pendant toute la durée
-- ⚠️ Table indisponible (peut prendre des heures)
```

**Temps estimé** :
- 1 GB : ~2-5 minutes
- 10 GB : ~20-50 minutes
- 100 GB : ~3-8 heures
- 1 TB : ~1-3 jours

#### B. Migration avec création de nouvelle table (RENAME)

```sql
-- 1. Créer nouvelle table InnoDB
CREATE TABLE users_new (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    -- ... (même structure)
) ENGINE=InnoDB;

-- 2. Copier les données (en batch si grosse table)
INSERT INTO users_new SELECT * FROM users WHERE id <= 1000000;
INSERT INTO users_new SELECT * FROM users WHERE id > 1000000 AND id <= 2000000;
-- etc.

-- 3. Swap atomique (downtime minimal)
RENAME TABLE
    users TO users_old_myisam,
    users_new TO users;

-- 4. Vérifier et supprimer ancienne table
-- ... tests ...
DROP TABLE users_old_myisam;
```

#### C. Migration avec gh-ost (zero-downtime)

```bash
# Outil open-source GitHub pour migration online
gh-ost \
    --host=localhost \
    --user=admin \
    --password=secret \
    --database=mydb \
    --table=users \
    --alter="ENGINE=InnoDB" \
    --execute \
    --allow-on-master \
    --cut-over=default \
    --exact-rowcount \
    --concurrent-rowcount
```

**Avantages gh-ost** :
- ✅ Table reste accessible pendant migration
- ✅ Pause/resume possible
- ✅ Rollback si problème
- ✅ Impact minimal sur production

#### D. Migration avec pt-online-schema-change (Percona Toolkit)

```bash
pt-online-schema-change \
    --alter="ENGINE=InnoDB" \
    --execute \
    D=mydb,t=users \
    --host=localhost \
    --user=admin \
    --password=secret \
    --chunk-size=1000 \
    --max-load="Threads_running=100"
```

### Étape 4 : Checklist de migration

**Avant la migration** :

```sql
-- ✓ Backup complet
mysqldump --single-transaction --routines --triggers \
    mydb > backup_before_migration.sql

-- ✓ Vérifier l'espace disque
-- InnoDB utilise plus d'espace (redo/undo logs, overhead)
SELECT
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) * 1.5 / 1024 / 1024, 2) AS estimated_innodb_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND ENGINE = 'MyISAM';

-- ✓ Vérifier la configuration InnoDB
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_file_per_table';
```

**Pendant la migration** :

```sql
-- Monitorer la progression (pour ALTER TABLE direct)
SHOW PROCESSLIST;

-- Vérifier les locks
SELECT * FROM information_schema.INNODB_TRX;
SELECT * FROM information_schema.PROCESSLIST WHERE State LIKE '%Waiting%';
```

**Après la migration** :

```sql
-- ✓ Vérifier l'intégrité
CHECK TABLE users;

-- ✓ Vérifier le nombre de lignes
SELECT COUNT(*) FROM users;

-- ✓ Analyser la table (statistiques)
ANALYZE TABLE users;

-- ✓ Optimiser (si nécessaire)
OPTIMIZE TABLE users;

-- ✓ Vérifier les performances
EXPLAIN SELECT * FROM users WHERE id = 42;
EXPLAIN SELECT * FROM users WHERE name = 'Alice';

-- ✓ Tester l'application
-- (tests fonctionnels complets)
```

### Étape 5 : Gestion des cas particuliers

#### Full-Text Search

```sql
-- MyISAM Full-Text
CREATE TABLE articles_myisam (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT(title, content)
) ENGINE=MyISAM;

-- Migration vers InnoDB (supporte FTS depuis MariaDB 10.0)
CREATE TABLE articles_innodb (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT(title, content)
) ENGINE=InnoDB;

INSERT INTO articles_innodb SELECT * FROM articles_myisam;

-- Requêtes Full-Text identiques
SELECT * FROM articles_innodb
WHERE MATCH(title, content) AGAINST('database');
```

#### Tables avec AUTO_INCREMENT

```sql
-- Préserver la valeur AUTO_INCREMENT
-- 1. Noter la valeur actuelle
SELECT AUTO_INCREMENT
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users_myisam';
-- AUTO_INCREMENT : 123456

-- 2. Créer table InnoDB
CREATE TABLE users_innodb LIKE users_myisam;
ALTER TABLE users_innodb ENGINE=InnoDB;

-- 3. Définir AUTO_INCREMENT
ALTER TABLE users_innodb AUTO_INCREMENT=123456;

-- 4. Copier les données
INSERT INTO users_innodb SELECT * FROM users_myisam;
```

#### Tables partitionnées

```sql
-- MyISAM avec partitions
CREATE TABLE logs_myisam (
    id INT,
    log_date DATE,
    message TEXT
) ENGINE=MyISAM
PARTITION BY RANGE (YEAR(log_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- Migration : Convertir partition par partition
ALTER TABLE logs_myisam
    REORGANIZE PARTITION p2023 INTO (
        PARTITION p2023 VALUES LESS THAN (2024) ENGINE=InnoDB
    );

ALTER TABLE logs_myisam
    REORGANIZE PARTITION p2024 INTO (
        PARTITION p2024 VALUES LESS THAN (2025) ENGINE=InnoDB
    );
-- etc.
```

---

## Problèmes courants et solutions

### Problème 1 : Table marquée as crashed

```sql
-- Erreur :
-- Table './mydb/users' is marked as crashed and should be repaired

-- Solution :
REPAIR TABLE users;

-- Si échec, forcer avec USE_FRM :
REPAIR TABLE users USE_FRM;

-- Si toujours échec, ligne de commande :
-- systemctl stop mariadb
-- myisamchk --recover /var/lib/mysql/mydb/users
-- systemctl start mariadb
```

### Problème 2 : Table lock timeout

```sql
-- Session 1 (longue écriture)
INSERT INTO myisam_table SELECT * FROM huge_table;
-- Verrou WRITE sur myisam_table pendant plusieurs minutes

-- Session 2
SELECT * FROM myisam_table;
-- ERROR 1205 (HY000): Lock wait timeout exceeded

-- Solution : Augmenter timeout ou migrer vers InnoDB
SET GLOBAL lock_wait_timeout = 3600;  -- 1 heure
```

### Problème 3 : Espace disque insuffisant après migration

```sql
-- MyISAM : 10 GB
-- InnoDB : 15 GB (overhead redo/undo, padding)

-- Solution :
-- 1. Vérifier l'espace avant migration
-- 2. Nettoyer les anciennes tables après validation
-- 3. Utiliser ROW_FORMAT=COMPRESSED si nécessaire
CREATE TABLE users_innodb (
    -- ...
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

### Problème 4 : Performance dégradée après migration

```sql
-- Causes possibles :
-- 1. Buffer Pool sous-dimensionné
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
-- Recommandation : 70-80% RAM

-- 2. Statistiques obsolètes
ANALYZE TABLE users;

-- 3. Index manquants ou non optimaux
SHOW INDEXES FROM users;

-- 4. Configuration InnoDB non optimisée
-- Voir section 7.2 (Configuration InnoDB avancée)
```

---

## ✅ Points clés à retenir

1. **MyISAM est obsolète** : Ne jamais utiliser pour de nouvelles tables. Migration vers InnoDB est impérative.

2. **Pas de transactions ACID** : Impossible de garantir l'atomicité, cohérence, isolation, durabilité.

3. **Table-level locking** : Goulot d'étranglement majeur en haute concurrence (100× plus lent qu'InnoDB).

4. **Pas de Foreign Keys** : Intégrité référentielle non garantie par le moteur.

5. **Corruption fréquente après crash** : Nécessite réparation manuelle (REPAIR TABLE, myisamchk).

6. **Fragmentation** : Dégradation des performances après UPDATE de taille variable.

7. **Migration nécessaire** : Utiliser gh-ost ou pt-online-schema-change pour zero-downtime sur grandes tables.

8. **InnoDB supérieur dans tous les cas** : Performances, fiabilité, fonctionnalités (même Full-Text depuis 10.0).

9. **Aria remplace MyISAM** : Pour les rares cas où MyISAM serait envisagé, Aria offre crash recovery en plus.

10. **Audit régulier** : Identifier et éliminer progressivement toutes les tables MyISAM legacy.

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 MyISAM Storage Engine](https://mariadb.com/kb/en/myisam-storage-engine/)
- [📖 MyISAM System Variables](https://mariadb.com/kb/en/myisam-system-variables/)
- [📖 myisamchk Tool](https://mariadb.com/kb/en/myisamchk/)
- [📖 Converting Tables from MyISAM to InnoDB](https://mariadb.com/kb/en/converting-tables-from-myisam-to-innodb/)
- [📖 REPAIR TABLE](https://mariadb.com/kb/en/repair-table/)

### Outils de migration

- [gh-ost (GitHub)](https://github.com/github/gh-ost) - Online schema change
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit) - pt-online-schema-change
- [MySQL Utilities](https://dev.mysql.com/downloads/utilities/) - mysqldbcopy, mysqldiff

### Articles techniques

- [Why You Should NOT Use MyISAM](https://www.percona.com/blog/why-you-should-not-use-myisam/)
- [MyISAM to InnoDB Migration Guide](https://www.percona.com/blog/myisam-innodb-migration/)
- [MyISAM Table Corruption and Recovery](https://dba.stackexchange.com/questions/myisam-corruption)

### Comparaisons et benchmarks

- [MyISAM vs InnoDB Performance Comparison](https://blog.jcole.us/2010/09/09/innodb-performance-optimization-basics/)
- [InnoDB vs MyISAM: The End of an Era](https://planet.mysql.com/entry/?id=658933)

---

## ➡️ Section suivante

**[7.4 Aria : Le successeur de MyISAM](/07-moteurs-de-stockage/04-aria.md)** : Découverte d'Aria, le moteur MariaDB qui améliore MyISAM avec crash recovery, utilisé pour les tables système.

Puis nous continuerons avec :
- **7.5** : ColumnStore pour analytics OLAP
- **7.6** : S3 pour archivage cloud
- **7.7** : Vector/HNSW pour recherche vectorielle IA 🆕

---

**📌 Mémo DBA** : "Si vous voyez MyISAM en production en 2025, planifiez immédiatement une migration vers InnoDB. Il n'y a aucune bonne raison de continuer à utiliser MyISAM."

**🔴 Rappel critique** : MyISAM + crash serveur = corruption probable + perte de données potentielle. C'est un risque inacceptable en production moderne.

⏭️ [Aria : Le successeur de MyISAM](/07-moteurs-de-stockage/04-aria.md)

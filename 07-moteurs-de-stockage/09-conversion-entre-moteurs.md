🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.9 Conversion entre moteurs (ALTER TABLE ENGINE)

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 7.1-7.8 (tous les moteurs et critères de choix)

> **Public cible** : DBA, Ingénieurs DevOps, Architectes de bases de données

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser les différentes méthodes de conversion entre moteurs
- Choisir la stratégie optimale selon la taille et la criticité
- Utiliser ALTER TABLE ENGINE et ses alternatives
- Implémenter des migrations zero-downtime avec gh-ost ou pt-online-schema-change
- Gérer les cas particuliers (Foreign Keys, Full-Text, partitions)
- Vérifier l'intégrité après migration
- Planifier et exécuter des rollbacks
- Automatiser les conversions avec scripts et procédures

---

## Introduction

La **conversion entre moteurs** est une opération courante mais critique qui nécessite une planification minutieuse. Les enjeux sont importants :

- **Downtime** : De quelques secondes à plusieurs heures selon la méthode
- **Intégrité des données** : Risque de perte si mal exécutée
- **Performance** : Impact sur la production pendant la conversion
- **Espace disque** : Besoin temporaire de 2-3× la taille de la table

> "Une migration de moteur de stockage est une opération à haut risque. Une planification de 80% et une exécution de 20% garantissent le succès."

### Quand convertir un moteur ?

```
Raisons de conversion :
┌───────────────────────────────────────────────────────┐
│ 1. Migration legacy → moderne                         │
│    MyISAM → InnoDB (URGENT en 2025)                   │
│                                                       │
│ 2. Optimisation performance                           │
│    InnoDB → ColumnStore (analytics massif)            │
│                                                       │
│ 3. Réduction coûts                                    │
│    InnoDB → S3 (archivage données froides)            │
│                                                       │
│ 4. Nouveaux besoins fonctionnels                      │
│    InnoDB → Vector/HNSW (ajout recherche IA)          │
│                                                       │
│ 5. Correction erreur architecturale                   │
│    ColumnStore → InnoDB (mal dimensionné)             │
└───────────────────────────────────────────────────────┘
```

---

## Méthodes de conversion

### Vue d'ensemble des méthodes

| Méthode | Downtime | Complexité | Taille max | Rollback | Usage |
|---------|----------|------------|------------|----------|-------|
| **ALTER TABLE ENGINE** | ⚠️ Total | Faible | < 10 GB | Difficile | Tables petites, fenêtre maintenance |
| **CREATE + INSERT SELECT** | ⚠️ Total | Faible | < 50 GB | Facile | Tables moyennes, plus de contrôle |
| **CREATE + RENAME** | ⚡ Minimal | Moyenne | < 500 GB | Facile | Tables moyennes-grandes |
| **gh-ost** | ✅ Aucun | Moyenne | Illimité | Facile | Tables critiques, production |
| **pt-online-schema-change** | ✅ Aucun | Moyenne | Illimité | Facile | Alternative à gh-ost |
| **Partition swap** | ⚡ Minimal | Élevée | Illimité | Facile | Tables partitionnées |

### Méthode 1 : ALTER TABLE ENGINE (simple mais bloquant)

**Principe** : Reconstruction complète de la table en place.

```sql
-- Conversion simple
ALTER TABLE orders ENGINE=InnoDB;

-- MariaDB :
-- 1. Lock table en EXCLUSIVE (lecture/écriture bloquées)
-- 2. Crée table temporaire avec nouveau moteur
-- 3. Copie ligne par ligne
-- 4. Remplace table originale
-- 5. Déverrouille
```

**Avantages** :
- ✅ Simple (une seule commande)
- ✅ Pas de script complexe
- ✅ Gestion automatique des index

**Inconvénients** :
- ❌ Table bloquée pendant toute la durée
- ❌ Pas de progression visible
- ❌ Rollback difficile (CTRL+C = corruption possible)
- ❌ Espace disque : 2× la taille de la table

**Durée estimée** :

```
Exemples de durée ALTER TABLE ENGINE :
┌───────────────────────────────────────────────────────┐
│ Taille table  │ Hardware         │ Durée estimée      │
├───────────────┼──────────────────┼────────────────────┤
│ 100 MB        │ HDD              │ 10-30 sec          │
│ 1 GB          │ SSD SATA         │ 1-3 min            │
│ 10 GB         │ SSD NVMe         │ 5-15 min           │
│ 100 GB        │ SSD NVMe         │ 30 min - 2 heures  │
│ 1 TB          │ SSD NVMe RAID    │ 5-20 heures        │
└───────────────────────────────────────────────────────┘

Formule approximative :
Durée (min) = Taille (GB) × (30 / Throughput MB/s)
Exemple : 100 GB sur SSD 500 MB/s → 100 × (30/500) = 6 min
```

**Quand utiliser ALTER TABLE ENGINE** :
- ✅ Tables < 10 GB
- ✅ Fenêtre de maintenance disponible
- ✅ Environnement de développement/test
- ❌ Production sans fenêtre maintenance
- ❌ Tables > 50 GB

### Méthode 2 : CREATE + INSERT SELECT (contrôle total)

**Principe** : Créer nouvelle table, copier, puis remplacer.

```sql
-- Étape 1 : Créer nouvelle table avec moteur cible
CREATE TABLE orders_new LIKE orders;
ALTER TABLE orders_new ENGINE=InnoDB;

-- Étape 2 : Copier les données
INSERT INTO orders_new SELECT * FROM orders;

-- Étape 3 : Vérifier
SELECT COUNT(*) FROM orders;      -- 10000000
SELECT COUNT(*) FROM orders_new;  -- 10000000

-- Étape 4 : Swap atomique (downtime minimal)
RENAME TABLE
    orders TO orders_old,
    orders_new TO orders;

-- Étape 5 : Valider puis supprimer ancienne table
-- ... tests applicatifs ...
DROP TABLE orders_old;
```

**Avantages** :
- ✅ Contrôle total du processus
- ✅ Progression visible (SELECT COUNT(*))
- ✅ Rollback facile (garder ancienne table)
- ✅ RENAME atomique (downtime < 1 seconde)
- ✅ Possibilité de copie par batch

**Inconvénients** :
- ⚠️ Espace disque : 2× la taille
- ⚠️ Copie ne capture pas les modifications pendant l'INSERT
- ⚠️ Nécessite script pour gérer les dépendances

**Copie par batch** :

```sql
-- Pour très grosses tables : Copie par batch
SET @batch_size = 10000;
SET @offset = 0;
SET @total = (SELECT COUNT(*) FROM orders);

WHILE @offset < @total DO
    INSERT INTO orders_new
    SELECT * FROM orders
    LIMIT @offset, @batch_size;

    SET @offset = @offset + @batch_size;

    -- Afficher progression
    SELECT CONCAT(
        'Progress: ',
        ROUND((@offset / @total) * 100, 2),
        '%'
    ) AS status;

    -- Petit délai pour ne pas saturer
    DO SLEEP(0.1);
END WHILE;
```

### Méthode 3 : gh-ost (zero-downtime, recommandé production)

**gh-ost** (GitHub Online Schema Change) est l'outil state-of-the-art pour migrations zero-downtime.

**Architecture** :

```
┌──────────────────────────────────────────────────────────┐
│                  Table originale (orders)                │
│  • Reçoit les écritures de l'application                 │
└──────────────────────────────────────────────────────────┘
                         ↓
                    Binary Log
                         ↓
┌──────────────────────────────────────────────────────────┐
│                      gh-ost                              │
│  1. Crée table fantôme (_orders_gho)                     │
│  2. Copie données batch par batch                        │
│  3. Applique changements via binlog                      │
│  4. Swap atomique à la fin                               │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│              Table fantôme (_orders_gho)                 │
│  • Nouveau moteur (InnoDB)                               │
│  • Synchronisée en temps réel                            │
└──────────────────────────────────────────────────────────┘
```

**Installation** :

```bash
# Télécharger gh-ost
wget https://github.com/github/gh-ost/releases/download/v1.1.6/gh-ost-binary-linux-amd64-20231207144046.tar.gz
tar -xzf gh-ost-binary-linux-amd64-20231207144046.tar.gz
sudo mv gh-ost /usr/local/bin/
chmod +x /usr/local/bin/gh-ost

# Vérifier
gh-ost --version
```

**Utilisation** :

```bash
# Conversion MyISAM → InnoDB avec gh-ost
gh-ost \
  --host=localhost \
  --port=3306 \
  --user=root \
  --password=secret \
  --database=mydb \
  --table=orders \
  --alter="ENGINE=InnoDB" \
  --execute \
  --allow-on-master \
  --exact-rowcount \
  --concurrent-rowcount \
  --default-retries=120 \
  --cut-over=default \
  --chunk-size=1000 \
  --max-load="Threads_running=25,Threads_connected=100" \
  --critical-load="Threads_running=50,Threads_connected=200" \
  --nice-ratio=0.5

# Paramètres clés :
# --alter : Changement à effectuer
# --chunk-size : Taille des batch (1000 lignes)
# --max-load : Seuil de throttling (ralentit si dépassé)
# --critical-load : Seuil critique (pause si dépassé)
# --nice-ratio : Temps d'attente entre batch (0.5 = 50%)
```

**Commandes pendant l'exécution** :

```bash
# Surveiller la progression
# gh-ost crée un fichier de contrôle
echo "status" > /tmp/gh-ost.orders.sock

# Throttler (ralentir)
echo "throttle" > /tmp/gh-ost.orders.sock

# Reprendre
echo "no-throttle" > /tmp/gh-ost.orders.sock

# Annuler proprement
echo "panic" > /tmp/gh-ost.orders.sock
```

**Avantages** :
- ✅ Zero-downtime (< 1 seconde de lock final)
- ✅ Pausable et reprisable
- ✅ Throttling automatique
- ✅ Rollback propre (DROP _gho table)
- ✅ Progression visible en temps réel

**Inconvénients** :
- ⚠️ Nécessite binlog (row-based)
- ⚠️ Espace disque : 2× la taille
- ⚠️ Surcharge CPU/I/O pendant la migration
- ⚠️ Configuration plus complexe

### Méthode 4 : pt-online-schema-change (Percona Toolkit)

Alternative à gh-ost, très similaire.

**Installation** :

```bash
# Debian/Ubuntu
sudo apt-get install percona-toolkit

# RHEL/CentOS
sudo yum install percona-toolkit

# Vérifier
pt-online-schema-change --version
```

**Utilisation** :

```bash
pt-online-schema-change \
  --alter="ENGINE=InnoDB" \
  --execute \
  D=mydb,t=orders \
  --host=localhost \
  --user=root \
  --password=secret \
  --chunk-size=1000 \
  --chunk-size-limit=4.0 \
  --max-load="Threads_running:25" \
  --critical-load="Threads_running:50" \
  --progress=percentage,5 \
  --no-drop-old-table
```

**Différences gh-ost vs pt-online-schema-change** :

| Aspect | gh-ost | pt-osc |
|--------|--------|--------|
| Méthode | Binlog streaming | Triggers |
| Overhead | Moyen (binlog) | Élevé (3 triggers) |
| Maintenance | Actif (GitHub) | Mature (Percona) |
| Flexibilité | Plus moderne | Plus éprouvé |
| Use case | Préféré pour nouvelles migrations | Préféré si problème binlog |

**Recommandation** : gh-ost pour nouveaux projets, pt-osc si déjà en place.

---

## Conversions courantes : Procédures détaillées

### Conversion 1 : MyISAM → InnoDB (migration legacy URGENTE)

**Contexte** : MyISAM est déprécié et dangereux. Migration vers InnoDB impérative.

#### Étape 1 : Audit pré-migration

```sql
-- Identifier toutes les tables MyISAM
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS fragmentation_mb
FROM information_schema.TABLES
WHERE ENGINE = 'MyISAM'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema')
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;

-- Résultat exemple :
-- +-------------+----------------+--------+------------+---------+-----------------+
-- | TABLE_SCHEMA| TABLE_NAME     | ENGINE | TABLE_ROWS | size_mb | fragmentation_mb|
-- +-------------+----------------+--------+------------+---------+-----------------+
-- | mydb        | huge_table     | MyISAM | 50000000   | 12000   | 500             |
-- | mydb        | medium_table   | MyISAM | 1000000    | 500     | 20              |
-- | mydb        | small_table    | MyISAM | 10000      | 5       | 0               |
-- +-------------+----------------+--------+------------+---------+-----------------+
```

#### Étape 2 : Vérifier l'espace disque

```bash
# Espace disque nécessaire : 2× taille tables + 30% marge
# Exemple : 12.5 GB tables → 32.5 GB espace libre requis

df -h /var/lib/mysql
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       100G   60G   40G  60% /

# Si espace insuffisant → Nettoyer ou ajouter disque
```

#### Étape 3 : Backup complet

```bash
# Backup avant toute migration
mysqldump --single-transaction --routines --triggers \
    --master-data=2 --flush-logs \
    mydb > /backup/mydb_before_migration.sql

# Vérifier le backup
ls -lh /backup/mydb_before_migration.sql
# -rw-r--r-- 1 root root 15G Dec 12 10:00 mydb_before_migration.sql

# Tester la restauration sur serveur secondaire
mysql -h test-server mydb_test < /backup/mydb_before_migration.sql
```

#### Étape 4 : Migration par taille de table

**Tables < 1 GB : ALTER TABLE direct**

```sql
-- Fenêtre de maintenance acceptable
ALTER TABLE small_table ENGINE=InnoDB;
-- Durée : 5-30 secondes

-- Vérifier
SHOW CREATE TABLE small_table\G
-- ENGINE=InnoDB
```

**Tables 1-100 GB : CREATE + RENAME**

```sql
-- Créer nouvelle table InnoDB
CREATE TABLE medium_table_new LIKE medium_table;
ALTER TABLE medium_table_new ENGINE=InnoDB;

-- Copier données
INSERT INTO medium_table_new SELECT * FROM medium_table;
-- Durée : 5-30 minutes selon hardware

-- Vérifier intégrité
SELECT COUNT(*),
       SUM(CRC32(CONCAT_WS(',', col1, col2, col3))) AS checksum
FROM medium_table;
-- Comparer avec medium_table_new

-- Swap atomique (< 1 sec downtime)
RENAME TABLE
    medium_table TO medium_table_old_myisam,
    medium_table_new TO medium_table;

-- Tester application
-- ... tests ...

-- Supprimer ancienne table
DROP TABLE medium_table_old_myisam;
```

**Tables > 100 GB : gh-ost**

```bash
# Migration zero-downtime
gh-ost \
  --host=localhost \
  --user=root \
  --password=secret \
  --database=mydb \
  --table=huge_table \
  --alter="ENGINE=InnoDB" \
  --execute \
  --allow-on-master \
  --exact-rowcount \
  --concurrent-rowcount \
  --chunk-size=1000 \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50" \
  --nice-ratio=0.5 \
  --panic-flag-file=/tmp/gh-ost.panic \
  --postpone-cut-over-flag-file=/tmp/gh-ost.postpone

# Durée estimée : 2-10 heures selon taille et charge
# Progression visible en temps réel
```

#### Étape 5 : Cas particuliers MyISAM

**Full-Text indexes** :

```sql
-- MyISAM Full-Text → InnoDB Full-Text (supporté depuis MariaDB 10.0)
CREATE TABLE articles_innodb LIKE articles_myisam;
ALTER TABLE articles_innodb ENGINE=InnoDB;

-- Copier
INSERT INTO articles_innodb SELECT * FROM articles_myisam;

-- Vérifier Full-Text fonctionne
SELECT * FROM articles_innodb
WHERE MATCH(title, content) AGAINST('database');
-- Doit retourner résultats
```

**AUTO_INCREMENT** :

```sql
-- Préserver AUTO_INCREMENT value
SELECT AUTO_INCREMENT
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users_myisam';
-- 1234567

CREATE TABLE users_innodb LIKE users_myisam;
ALTER TABLE users_innodb ENGINE=InnoDB AUTO_INCREMENT=1234567;

INSERT INTO users_innodb SELECT * FROM users_myisam;
```

#### Étape 6 : Post-migration

```sql
-- Analyser les nouvelles tables InnoDB
ANALYZE TABLE small_table, medium_table, huge_table;

-- Vérifier les statistiques
SHOW TABLE STATUS LIKE 'huge_table'\G

-- Optimizer les tables (défragmentation)
OPTIMIZE TABLE huge_table;

-- Surveiller les performances
SHOW ENGINE INNODB STATUS\G
```

### Conversion 2 : InnoDB → ColumnStore (analytics)

**Contexte** : Table analytics avec requêtes d'agrégation lentes sur InnoDB.

#### Procédure

```sql
-- Table InnoDB existante (analytics)
-- sales_fact : 500 millions de lignes, 200 GB

-- Étape 1 : Créer table ColumnStore
CREATE TABLE sales_fact_cs (
    sale_date DATE,
    customer_id INT,
    product_id INT,
    region VARCHAR(50),
    revenue DECIMAL(10,2),
    quantity INT
) ENGINE=ColumnStore;

-- Étape 2 : Charger les données
-- Option A : INSERT SELECT (pour tables moyennes < 50 GB)
INSERT INTO sales_fact_cs SELECT * FROM sales_fact;

-- Option B : cpimport (pour tables massives > 50 GB)
-- 1. Exporter depuis InnoDB
SELECT * FROM sales_fact
INTO OUTFILE '/tmp/sales_fact.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';

-- 2. Importer dans ColumnStore
cpimport mydb sales_fact_cs /tmp/sales_fact.csv -s ',' -E '"'
-- Jusqu'à 10M lignes/sec

-- Étape 3 : Vérifier
SELECT COUNT(*) FROM sales_fact;     -- 500000000
SELECT COUNT(*) FROM sales_fact_cs;  -- 500000000

-- Étape 4 : Tester performance
-- Avant (InnoDB)
SELECT region, SUM(revenue) FROM sales_fact GROUP BY region;
-- 45 secondes

-- Après (ColumnStore)
SELECT region, SUM(revenue) FROM sales_fact_cs GROUP BY region;
-- 2 secondes (22× plus rapide !)

-- Étape 5 : Swap (si validation OK)
RENAME TABLE
    sales_fact TO sales_fact_innodb_old,
    sales_fact_cs TO sales_fact;

-- Étape 6 : Cleanup après validation
DROP TABLE sales_fact_innodb_old;
```

### Conversion 3 : InnoDB → S3 (archivage)

**Contexte** : Archives historiques rarement consultées.

#### Procédure

```sql
-- Table InnoDB : orders_2020 (50 millions lignes, 20 GB)

-- Étape 1 : Créer table Aria intermédiaire
CREATE TABLE orders_2020_aria ENGINE=Aria TRANSACTIONAL=0
SELECT * FROM orders_2020;

-- Étape 2 : Convertir en S3
ALTER TABLE orders_2020_aria ENGINE=S3;
-- Upload vers s3://my-bucket/mydb/orders_2020_aria/
-- Durée : 5-15 minutes selon connexion S3

-- Étape 3 : Vérifier accessibilité
SELECT COUNT(*) FROM orders_2020_aria;  -- 50000000
SELECT * FROM orders_2020_aria LIMIT 10;  -- Fonctionne

-- Étape 4 : Renommer
RENAME TABLE
    orders_2020 TO orders_2020_innodb_backup,
    orders_2020_aria TO orders_2020;

-- Étape 5 : Tester en production (quelques jours)

-- Étape 6 : Supprimer backup InnoDB
DROP TABLE orders_2020_innodb_backup;

-- Économie : 20 GB SSD → 4 GB S3 compressé
-- Coût : $2/mois → $0.10/mois (économie 95%)
```

### Conversion 4 : InnoDB → Vector/HNSW (ajout IA)

**Contexte** : Ajouter recherche sémantique sur table existante.

#### Procédure

```sql
-- Table InnoDB existante
CREATE TABLE documents (
    doc_id INT PRIMARY KEY,
    title VARCHAR(500),
    content TEXT
) ENGINE=InnoDB;

-- Étape 1 : Ajouter colonne VECTOR
ALTER TABLE documents
ADD COLUMN embedding VECTOR(1536);
-- Rapide (ajout colonne vide)

-- Étape 2 : Générer embeddings (via application)
-- Python/Node.js script pour appeler OpenAI API et UPDATE

-- Pseudo-code Python :
-- for doc in documents:
--     embedding = openai.embed(doc.content)
--     UPDATE documents SET embedding = embedding WHERE doc_id = doc.id

-- Étape 3 : Créer index HNSW
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW WITH (M=16, ef_construction=200, metric='cosine');
-- Durée : Dépend du nombre de vecteurs (1M vecteurs ~ 10-30 min)

-- Étape 4 : Tester recherche vectorielle
SELECT
    doc_id,
    title,
    VEC_DISTANCE_COSINE(embedding, '[0.1, -0.2, ...]') AS relevance
FROM documents
ORDER BY relevance ASC
LIMIT 10;
```

### Conversion 5 : ColumnStore → InnoDB (correction erreur)

**Contexte** : Table mal placée dans ColumnStore (trop petite, besoins OLTP).

#### Procédure

```sql
-- Table ColumnStore : products (100K lignes, 50 MB)
-- Problème : UPDATE fréquents (prix, stock) → ColumnStore inadapté

-- Étape 1 : Créer table InnoDB
CREATE TABLE products_innodb (
    product_id INT PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2),
    stock INT,
    INDEX idx_name (name)
) ENGINE=InnoDB;

-- Étape 2 : Copier depuis ColumnStore
INSERT INTO products_innodb SELECT * FROM products;

-- Étape 3 : Vérifier
SELECT COUNT(*) FROM products;         -- 100000
SELECT COUNT(*) FROM products_innodb;  -- 100000

-- Étape 4 : Swap
RENAME TABLE
    products TO products_columnstore_old,
    products_innodb TO products;

-- Étape 5 : Tester UPDATE
UPDATE products SET price = price * 1.05 WHERE product_id = 42;
-- Rapide avec InnoDB (0.1 ms vs 1 sec avec ColumnStore)

-- Étape 6 : Cleanup
DROP TABLE products_columnstore_old;
```

---

## Gestion des dépendances

### Foreign Keys

Les Foreign Keys doivent être gérées manuellement lors des conversions.

```sql
-- Scénario : orders (InnoDB) avec FK vers customers
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(200)
) ENGINE=InnoDB;

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
) ENGINE=InnoDB;

-- Conversion orders → ColumnStore (FK non supportée)

-- Étape 1 : Lister les FK
SELECT
    CONSTRAINT_NAME,
    TABLE_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'orders'
  AND REFERENCED_TABLE_NAME IS NOT NULL;

-- Étape 2 : Supprimer FK
ALTER TABLE orders DROP FOREIGN KEY orders_ibfk_1;

-- Étape 3 : Convertir
ALTER TABLE orders ENGINE=ColumnStore;

-- Étape 4 : Documenter perte de FK
-- ⚠️ Intégrité référentielle doit être gérée par l'application
```

**Alternative pour préserver intégrité** :

```sql
-- Garder tables relationnelles en InnoDB
-- Créer table analytics dénormalisée en ColumnStore

CREATE TABLE orders_analytics (
    order_id INT,
    order_date DATE,
    customer_name VARCHAR(200),  -- Dénormalisé
    customer_country VARCHAR(50), -- Dénormalisé
    amount DECIMAL(10,2)
) ENGINE=ColumnStore;

-- ETL quotidien
INSERT INTO orders_analytics
SELECT
    o.order_id,
    o.order_date,
    c.name,
    c.country,
    o.amount
FROM orders o  -- InnoDB (avec FK)
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date = CURDATE() - INTERVAL 1 DAY;
```

### Triggers

Les triggers restent attachés à la table lors de la conversion.

```sql
-- Table avec trigger
CREATE TABLE audit_log (
    log_id INT PRIMARY KEY AUTO_INCREMENT,
    action VARCHAR(50),
    timestamp DATETIME
) ENGINE=InnoDB;

CREATE TRIGGER orders_after_insert
AFTER INSERT ON orders
FOR EACH ROW
    INSERT INTO audit_log (action, timestamp)
    VALUES ('INSERT', NOW());

-- Conversion : Trigger reste fonctionnel
ALTER TABLE orders ENGINE=ColumnStore;

-- Vérifier triggers
SHOW TRIGGERS WHERE `Table` = 'orders'\G
```

### Vues

Les vues restent fonctionnelles mais peuvent devenir plus lentes.

```sql
-- Vue basée sur table InnoDB
CREATE VIEW active_orders AS
SELECT * FROM orders WHERE status = 'active';

-- Après conversion orders → ColumnStore
-- Vue fonctionne toujours mais :
-- • Plus lente si filtre non-optimal pour columnar
-- • Bénéficie des agrégations si la vue les utilise

-- Recommandation : Revoir définition vue après conversion
```

---

## Vérifications post-migration

### Checklist complète

```sql
-- ═══════════════════════════════════════════════════════
-- CHECKLIST POST-MIGRATION
-- ═══════════════════════════════════════════════════════

-- 1. Vérifier nombre de lignes
SELECT 'Old table', COUNT(*) FROM table_old;
SELECT 'New table', COUNT(*) FROM table_new;
-- Doivent être identiques

-- 2. Vérifier checksum (détection corruption)
SELECT
    SUM(CRC32(CONCAT_WS(',', col1, col2, col3))) AS checksum
FROM table_old;
-- Comparer avec table_new

-- 3. Vérifier structure
SHOW CREATE TABLE table_old\G
SHOW CREATE TABLE table_new\G
-- Comparer ENGINE, colonnes, index

-- 4. Vérifier index
SHOW INDEX FROM table_new;
-- Tous les index présents ?

-- 5. Vérifier AUTO_INCREMENT
SHOW TABLE STATUS LIKE 'table_new'\G
-- Auto_increment doit être cohérent

-- 6. Vérifier statistiques
ANALYZE TABLE table_new;
SHOW TABLE STATUS LIKE 'table_new'\G
-- Rows, Avg_row_length, Data_length

-- 7. Test requêtes critiques
-- Exécuter 10-20 requêtes SQL critiques de l'application
SELECT * FROM table_new WHERE id = 12345;
SELECT COUNT(*), AVG(amount) FROM table_new GROUP BY category;
-- etc.

-- 8. Test performance
-- Comparer temps d'exécution avant/après
EXPLAIN SELECT ... FROM table_new;

-- 9. Test écriture (si applicable)
INSERT INTO table_new VALUES (...);
UPDATE table_new SET ... WHERE ...;
DELETE FROM table_new WHERE ...;

-- 10. Vérifier logs erreurs
-- shell> tail -100 /var/log/mysql/error.log
-- Pas d'erreurs liées à la nouvelle table ?
```

### Script de vérification automatique

```bash
#!/bin/bash
# verify_migration.sh

DB="mydb"
TABLE_OLD="orders_old"
TABLE_NEW="orders"
USER="root"
PASS="secret"

echo "=== Migration Verification ==="
echo "Database: $DB"
echo "Old table: $TABLE_OLD"
echo "New table: $TABLE_NEW"
echo ""

# 1. Row count
echo "1. Checking row count..."
COUNT_OLD=$(mysql -u$USER -p$PASS -D$DB -sNe "SELECT COUNT(*) FROM $TABLE_OLD")
COUNT_NEW=$(mysql -u$USER -p$PASS -D$DB -sNe "SELECT COUNT(*) FROM $TABLE_NEW")

if [ "$COUNT_OLD" == "$COUNT_NEW" ]; then
    echo "✓ Row count match: $COUNT_NEW"
else
    echo "✗ Row count mismatch: $COUNT_OLD vs $COUNT_NEW"
    exit 1
fi

# 2. Checksum
echo "2. Checking checksum..."
CHECKSUM_OLD=$(mysql -u$USER -p$PASS -D$DB -sNe \
    "SELECT SUM(CRC32(CONCAT_WS(',', *))) FROM $TABLE_OLD")
CHECKSUM_NEW=$(mysql -u$USER -p$PASS -D$DB -sNe \
    "SELECT SUM(CRC32(CONCAT_WS(',', *))) FROM $TABLE_NEW")

if [ "$CHECKSUM_OLD" == "$CHECKSUM_NEW" ]; then
    echo "✓ Checksum match"
else
    echo "✗ Checksum mismatch"
    exit 1
fi

# 3. Engine verification
echo "3. Checking engine..."
ENGINE=$(mysql -u$USER -p$PASS -D$DB -sNe \
    "SELECT ENGINE FROM information_schema.TABLES
     WHERE TABLE_SCHEMA='$DB' AND TABLE_NAME='$TABLE_NEW'")
echo "✓ Engine: $ENGINE"

# 4. Index count
echo "4. Checking indexes..."
INDEX_COUNT=$(mysql -u$USER -p$PASS -D$DB -sNe \
    "SELECT COUNT(DISTINCT INDEX_NAME) FROM information_schema.STATISTICS
     WHERE TABLE_SCHEMA='$DB' AND TABLE_NAME='$TABLE_NEW'")
echo "✓ Index count: $INDEX_COUNT"

echo ""
echo "=== Verification Complete ==="
echo "All checks passed ✓"
```

---

## Rollback strategies

### Rollback méthode CREATE + RENAME

```sql
-- Si problème détecté après swap

-- Situation actuelle :
-- orders (nouvelle table, problème)
-- orders_old (ancienne table, sauvegardée)

-- Rollback (< 1 sec)
RENAME TABLE
    orders TO orders_problematic,
    orders_old TO orders;

-- Application revient sur ancienne table
-- Analyser le problème sur orders_problematic
-- Puis DROP orders_problematic une fois résolu
```

### Rollback gh-ost

```bash
# Pendant la migration
echo "panic" > /tmp/gh-ost.orders.sock
# gh-ost arrête et DROP la table fantôme
# Table originale non modifiée

# Après le cut-over (si problème immédiat)
# L'ancienne table existe encore : _orders_old
RENAME TABLE
    orders TO orders_new_problematic,
    _orders_old TO orders;
```

### Rollback via backup

```bash
# Dernier recours si corruption ou perte données

# 1. Arrêter MariaDB
systemctl stop mariadb

# 2. Supprimer table corrompue
rm /var/lib/mysql/mydb/orders.*

# 3. Restaurer depuis backup
mysql mydb < /backup/orders_backup.sql

# 4. Redémarrer
systemctl start mariadb

# 5. Vérifier
mysql -e "SELECT COUNT(*) FROM mydb.orders"
```

---

## Automatisation : Scripts et procédures

### Procédure stockée : Migration automatique

```sql
DELIMITER $$

CREATE PROCEDURE migrate_table_to_innodb(
    IN p_schema VARCHAR(64),
    IN p_table VARCHAR(64),
    IN p_method VARCHAR(20)  -- 'ALTER' ou 'CREATE_RENAME'
)
BEGIN
    DECLARE v_old_engine VARCHAR(64);
    DECLARE v_row_count BIGINT;
    DECLARE v_table_new VARCHAR(128);
    DECLARE v_table_old VARCHAR(128);

    -- Vérifier moteur actuel
    SELECT ENGINE, TABLE_ROWS
    INTO v_old_engine, v_row_count
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = p_schema
      AND TABLE_NAME = p_table;

    IF v_old_engine = 'InnoDB' THEN
        SELECT CONCAT('Table ', p_table, ' is already InnoDB') AS result;
        LEAVE;
    END IF;

    -- Logging
    INSERT INTO migration_log (schema_name, table_name, started_at, old_engine, row_count, method)
    VALUES (p_schema, p_table, NOW(), v_old_engine, v_row_count, p_method);

    IF p_method = 'ALTER' THEN
        -- Méthode ALTER TABLE
        SET @sql = CONCAT('ALTER TABLE ', p_schema, '.', p_table, ' ENGINE=InnoDB');
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;

    ELSEIF p_method = 'CREATE_RENAME' THEN
        -- Méthode CREATE + RENAME
        SET v_table_new = CONCAT(p_table, '_new');
        SET v_table_old = CONCAT(p_table, '_old');

        -- Créer nouvelle table
        SET @sql = CONCAT('CREATE TABLE ', p_schema, '.', v_table_new,
                          ' LIKE ', p_schema, '.', p_table);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;

        -- Changer moteur
        SET @sql = CONCAT('ALTER TABLE ', p_schema, '.', v_table_new, ' ENGINE=InnoDB');
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;

        -- Copier données
        SET @sql = CONCAT('INSERT INTO ', p_schema, '.', v_table_new,
                          ' SELECT * FROM ', p_schema, '.', p_table);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;

        -- Swap
        SET @sql = CONCAT('RENAME TABLE ',
                          p_schema, '.', p_table, ' TO ', p_schema, '.', v_table_old, ',',
                          p_schema, '.', v_table_new, ' TO ', p_schema, '.', p_table);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END IF;

    -- Log success
    UPDATE migration_log
    SET completed_at = NOW(), status = 'SUCCESS'
    WHERE schema_name = p_schema
      AND table_name = p_table
      AND completed_at IS NULL;

    SELECT CONCAT('Migration of ', p_table, ' to InnoDB completed') AS result;
END$$

DELIMITER ;

-- Table de log
CREATE TABLE migration_log (
    log_id INT PRIMARY KEY AUTO_INCREMENT,
    schema_name VARCHAR(64),
    table_name VARCHAR(64),
    started_at DATETIME,
    completed_at DATETIME,
    old_engine VARCHAR(64),
    row_count BIGINT,
    method VARCHAR(20),
    status VARCHAR(20),
    INDEX idx_table (schema_name, table_name)
) ENGINE=InnoDB;

-- Utilisation
CALL migrate_table_to_innodb('mydb', 'orders', 'CREATE_RENAME');
```

### Script Bash : Migration batch

```bash
#!/bin/bash
# migrate_all_myisam.sh
# Migre toutes les tables MyISAM vers InnoDB

DB_USER="root"
DB_PASS="secret"
DB_NAME="mydb"

# Lister tables MyISAM
TABLES=$(mysql -u$DB_USER -p$DB_PASS -D$DB_NAME -sNe \
    "SELECT TABLE_NAME FROM information_schema.TABLES
     WHERE TABLE_SCHEMA='$DB_NAME' AND ENGINE='MyISAM'")

echo "=== MyISAM → InnoDB Migration ==="
echo "Database: $DB_NAME"
echo "Tables to migrate: $(echo "$TABLES" | wc -l)"
echo ""

for TABLE in $TABLES; do
    echo "Migrating $TABLE..."

    # Obtenir taille
    SIZE=$(mysql -u$DB_USER -p$DB_PASS -D$DB_NAME -sNe \
        "SELECT ROUND((DATA_LENGTH+INDEX_LENGTH)/1024/1024,2)
         FROM information_schema.TABLES
         WHERE TABLE_SCHEMA='$DB_NAME' AND TABLE_NAME='$TABLE'")

    echo "  Size: ${SIZE} MB"

    # Choisir méthode selon taille
    if (( $(echo "$SIZE < 1000" | bc -l) )); then
        echo "  Method: ALTER TABLE (direct)"
        mysql -u$DB_USER -p$DB_PASS -D$DB_NAME -e \
            "ALTER TABLE $TABLE ENGINE=InnoDB"
    else
        echo "  Method: gh-ost (large table)"
        gh-ost \
            --host=localhost \
            --user=$DB_USER \
            --password=$DB_PASS \
            --database=$DB_NAME \
            --table=$TABLE \
            --alter="ENGINE=InnoDB" \
            --execute \
            --allow-on-master \
            --exact-rowcount
    fi

    echo "  ✓ Completed"
    echo ""
done

echo "=== Migration Complete ==="
```

---

## Cas particuliers et pièges à éviter

### Piège 1 : Espace disque insuffisant

```bash
# Problème : Conversion échoue à 90% par manque d'espace

# Prévention : Vérifier avant
TABLE_SIZE=$(mysql -uroot -psecret -sNe \
    "SELECT (DATA_LENGTH+INDEX_LENGTH)*2/1024/1024/1024
     FROM information_schema.TABLES
     WHERE TABLE_NAME='huge_table'")

DISK_FREE=$(df /var/lib/mysql | tail -1 | awk '{print $4/1024/1024}')

if (( $(echo "$TABLE_SIZE > $DISK_FREE" | bc -l) )); then
    echo "ERROR: Insufficient disk space"
    echo "Required: ${TABLE_SIZE} GB"
    echo "Available: ${DISK_FREE} GB"
    exit 1
fi
```

### Piège 2 : Conversion pendant haute charge

```bash
# Problème : Conversion sature le serveur, ralentit production

# Solution : Throttler avec gh-ost
gh-ost \
    --max-load="Threads_running=25,Threads_connected=100" \
    --critical-load="Threads_running=50" \
    --nice-ratio=0.8  # 80% attente entre batch
    ...
```

### Piège 3 : Oublier de vérifier Foreign Keys

```sql
-- Problème : Conversion casse les FK

-- Solution : Lister et documenter AVANT
SELECT
    CONSTRAINT_NAME,
    TABLE_NAME,
    REFERENCED_TABLE_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'mydb'
  AND REFERENCED_TABLE_NAME IS NOT NULL;

-- Recréer après si nécessaire
ALTER TABLE orders
ADD CONSTRAINT fk_customer
FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

### Piège 4 : Négliger les statistiques

```sql
-- Problème : Performances dégradées après conversion

-- Solution : ANALYZE TABLE obligatoire
ALTER TABLE orders ENGINE=InnoDB;
ANALYZE TABLE orders;  -- ← OBLIGATOIRE

-- Puis surveiller plans d'exécution
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

### Piège 5 : Conversion pendant backup

```bash
# Problème : Corruption ou backup incohérent

# Solution : Coordination backup/migration
# 1. Planifier migrations hors fenêtre backup
# 2. Ou suspendre backup pendant migration
# 3. Ou utiliser gh-ost (pas d'impact backup)
```

---

## ✅ Points clés à retenir

1. **Planification critique** : 80% planification, 20% exécution pour succès.

2. **Backup impératif** : Toujours backup complet avant conversion.

3. **Méthode selon taille** : ALTER (< 10 GB), CREATE+RENAME (< 100 GB), gh-ost (> 100 GB).

4. **gh-ost recommandé production** : Zero-downtime, pausable, throttling auto.

5. **Vérifications post-migration** : COUNT, checksum, structure, performance.

6. **Espace disque** : Prévoir 2-3× taille table (conversion + marge).

7. **Foreign Keys** : Gérées manuellement, documenter pertes.

8. **ANALYZE TABLE** : Obligatoire après conversion pour statistiques correctes.

9. **Rollback prévu** : Garder ancienne table jusqu'à validation complète.

10. **MyISAM → InnoDB URGENT** : Priorité absolue en 2025, MyISAM dangereux.

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 ALTER TABLE](https://mariadb.com/kb/en/alter-table/)
- [📖 Storage Engine Conversion](https://mariadb.com/kb/en/storage-engine-conversion/)
- [📖 Online DDL](https://mariadb.com/kb/en/innodb-online-ddl/)

### Outils de migration

- [gh-ost (GitHub)](https://github.com/github/gh-ost)
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)
- [mariabackup Documentation](https://mariadb.com/kb/en/mariabackup/)

### Guides pratiques

- [Migration Best Practices](https://mariadb.com/kb/en/migration-best-practices/)
- [MyISAM to InnoDB Migration Guide](https://mariadb.com/resources/blog/myisam-to-innodb/)
- [Zero-Downtime Migrations](https://mariadb.org/zero-downtime-migrations/)

---

## 🎓 Conclusion du chapitre 7

Vous avez maintenant une **maîtrise complète** des moteurs de stockage MariaDB :

1. ✅ **Architecture Pluggable** (7.1) - Compréhension système
2. ✅ **InnoDB** (7.2) - Moteur transactionnel maître
3. ✅ **MyISAM** (7.3) - Legacy à migrer
4. ✅ **Aria** (7.4) - Crash-safe simple
5. ✅ **ColumnStore** (7.5) - Analytics OLAP
6. ✅ **S3** (7.6) - Archivage économique
7. ✅ **Vector/HNSW** (7.7) - IA et recherche sémantique 🆕
8. ✅ **Comparaison et choix** (7.8) - Décisions architecturales
9. ✅ **Conversion entre moteurs** (7.9) - Migration pratique

**Compétences acquises** :
- Choisir le moteur optimal selon le cas d'usage
- Concevoir des architectures hybrides multi-moteurs
- Migrer entre moteurs avec stratégies zero-downtime
- Optimiser configurations moteurs
- Éviter les anti-patterns courants

**Prochaines étapes** : Appliquer ces connaissances sur vos projets réels, documenter vos choix architecturaux, et réévaluer régulièrement vos décisions selon l'évolution des besoins.

---

**📌 Mémo DBA Final** :
```
Migration = Backup + Planification + Vérification + Rollback prévu
Jamais sur production sans tests complets en préproduction
gh-ost = ami des grandes migrations
ALTER TABLE = petit tables uniquement
MyISAM → InnoDB = URGENT 2025
```

**🎯 Checklist universelle migration** :
1. ✅ Backup complet vérifié
2. ✅ Espace disque suffisant (2-3× table)
3. ✅ Fenêtre maintenance ou gh-ost
4. ✅ Plan rollback documenté
5. ✅ Vérifications post-migration (count, checksum, perf)
6. ✅ Validation applicative complète
7. ✅ ANALYZE TABLE exécuté
8. ✅ Monitoring 48h post-migration
9. ✅ Documentation mise à jour
10. ✅ Cleanup après validation (DROP old tables)

⏭️ [Moteurs spécialisés](/07-moteurs-de-stockage/10-moteurs-specialises.md)

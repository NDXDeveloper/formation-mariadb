🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.3 - Export/Import

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 60 minutes  
> **Prérequis** : Sections B.1 et B.2

---

## 📖 Introduction

Cette section couvre les **commandes et techniques d'export et import** de données dans MariaDB. Ces opérations sont essentielles pour les sauvegardes, migrations, transferts de données et synchronisation entre environnements.

### 🎯 Objectifs

À l'issue de cette section, vous serez capable de :
- Exécuter des scripts SQL avec SOURCE
- Enregistrer les sorties avec TEE
- Exporter des bases complètes avec mariadb-dump
- Importer des données CSV avec LOAD DATA
- Exporter vers fichiers avec SELECT INTO OUTFILE
- Réaliser des sauvegardes et restaurations complètes

---

## 📥 SOURCE - Exécuter un Fichier SQL

### Syntaxe de Base

```sql
-- Méthode 1 : Méta-commande SOURCE
SOURCE /path/to/script.sql

-- Méthode 2 : Opérateur \. (raccourci)
\. /path/to/script.sql
```

### Exemples Pratiques

#### Importer une Structure de Base

```sql
-- Se connecter et sélectionner la base
USE production_db;

-- Exécuter le script de création
SOURCE /home/user/schema.sql

-- Vérifier
SHOW TABLES;
```

**Fichier schema.sql** :
```sql
-- schema.sql
-- Création des tables pour production_db

CREATE TABLE IF NOT EXISTS customers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  total DECIMAL(10,2) NOT NULL,
  status ENUM('pending','processing','completed','cancelled') DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Créer les index
CREATE INDEX idx_customer_email ON customers(email);
CREATE INDEX idx_order_status ON orders(status);
CREATE INDEX idx_order_created ON orders(created_at);
```

#### Importer des Données

```sql
-- Importer les données après la structure
SOURCE /home/user/data.sql
```

**Fichier data.sql** :
```sql
-- data.sql
-- Insertion de données de test

START TRANSACTION;

INSERT INTO customers (name, email) VALUES
  ('Alice Martin', 'alice@example.com'),
  ('Bob Dupont', 'bob@example.com'),
  ('Claire Bernard', 'claire@example.com');

INSERT INTO orders (customer_id, total, status) VALUES
  (1, 150.00, 'completed'),
  (1, 89.50, 'pending'),
  (2, 245.75, 'processing'),
  (3, 67.20, 'completed');

COMMIT;
```

#### Exécuter depuis la Ligne de Commande

```bash
# Sans entrer dans le client
mariadb -u root -p production_db < schema.sql

# Avec verbose (affiche les commandes)
mariadb -u root -p -v production_db < schema.sql

# Avec gestion d'erreurs
mariadb -u root -p production_db < schema.sql 2> errors.log

# Chaîner plusieurs fichiers
mariadb -u root -p production_db < schema.sql
mariadb -u root -p production_db < data.sql

# Ou en une commande
cat schema.sql data.sql | mariadb -u root -p production_db
```

### Gestion des Erreurs

```sql
-- Script avec gestion d'erreurs
-- script_safe.sql

-- Arrêter en cas d'erreur (comportement par défaut)
-- Si une commande échoue, le script continue quand même
-- Pour arrêter sur erreur, utiliser --force=false en CLI

-- Créer base si n'existe pas
CREATE DATABASE IF NOT EXISTS production_db;
USE production_db;

-- Supprimer table si existe (pour réinitialisation)
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customers;

-- Créer tables
SOURCE /path/to/schema.sql;

-- Vérifier
SELECT 'Tables créées avec succès' AS status;
SHOW TABLES;
```

### Options Utiles avec SOURCE

```bash
# Mode force : continue malgré erreurs
mariadb -u root -p --force < script.sql

# Verbose : affiche les commandes
mariadb -u root -p -v < script.sql

# Mode batch : pas de formatage tableau
mariadb -u root -p -B < script.sql

# Silent : pas de sortie sauf erreurs
mariadb -u root -p -s < script.sql

# Redirection des erreurs
mariadb -u root -p < script.sql 2> errors.log

# Tout dans un log
mariadb -u root -p < script.sql &> full.log
```

---

## 📝 TEE - Enregistrer la Sortie

### Syntaxe de Base

```sql
-- Démarrer l'enregistrement
TEE /path/to/output.log

-- ou méta-commande équivalente
\T /path/to/output.log

-- Arrêter l'enregistrement
NOTEE;

-- ou méta-commande
\t
```

### Exemples Pratiques

#### Session de Diagnostic

```sql
-- Démarrer l'enregistrement
TEE /tmp/diagnostic_$(date +%Y%m%d_%H%M%S).log

-- Informations serveur
SELECT VERSION(), CURRENT_USER(), DATABASE();
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW STATUS LIKE 'Threads_connected';

-- Lister les bases volumineuses
SELECT 
  table_schema AS 'Database',
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.TABLES
GROUP BY table_schema
ORDER BY SUM(data_length + index_length) DESC
LIMIT 10;

-- Processus actifs
SHOW PROCESSLIST;

-- Arrêter l'enregistrement
NOTEE;
```

**Résultat** : Tout est enregistré dans le fichier log.

#### Audit de Requêtes

```sql
-- Session d'audit
TEE /var/log/mariadb/audit_session.log

-- Tracer l'activité
SELECT 'Starting audit session' AS status, NOW() AS timestamp;

-- Requêtes d'analyse
SELECT COUNT(*) FROM orders WHERE status = 'pending';
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;

-- Fin de session
SELECT 'Ending audit session' AS status, NOW() AS timestamp;
NOTEE;
```

#### Export de Résultats

```sql
-- Exporter résultats vers fichier
TEE /tmp/report.txt

-- Rapport
SELECT 
  'Daily Sales Report' AS title,
  DATE(created_at) AS date,
  COUNT(*) AS orders,
  SUM(total) AS revenue
FROM orders
WHERE created_at >= CURDATE() - INTERVAL 7 DAY
GROUP BY DATE(created_at)
ORDER BY date DESC;

NOTEE;
```

### TEE depuis la Ligne de Commande

```bash
# Option --tee à la connexion
mariadb -u root -p --tee=/tmp/session.log

# Tout sera enregistré automatiquement
# Équivalent à TEE dès le début de session
```

### Combiner TEE avec Autres Commandes

```bash
# Script avec enregistrement automatique
mariadb -u root -p --tee=/tmp/import.log production_db < import.sql

# Session interactive avec log
mariadb -u root -p --tee=/var/log/mariadb/interactive_$(date +%Y%m%d).log
```

---

## 💾 mariadb-dump - Export de Bases de Données

### Syntaxe de Base

```bash
# Export d'une base complète
mariadb-dump [options] database_name > backup.sql

# ou nom legacy
mysqldump [options] database_name > backup.sql
```

### Export d'une Base Complète

```bash
# Export simple
mariadb-dump -u root -p production_db > production_backup.sql

# Avec timestamp
mariadb-dump -u root -p production_db > production_$(date +%Y%m%d).sql

# Avec compression
mariadb-dump -u root -p production_db | gzip > production_backup.sql.gz
```

### Options Essentielles

#### Structure Seule (--no-data)

```bash
# Exporter uniquement la structure (DDL)
mariadb-dump -u root -p --no-data production_db > schema_only.sql

# Utile pour :
# - Documenter le schéma
# - Créer base de test vide
# - Comparer structures
```

#### Données Seules (--no-create-info)

```bash
# Exporter uniquement les données (DML)
mariadb-dump -u root -p --no-create-info production_db > data_only.sql

# Utile pour :
# - Importer données dans structure existante
# - Rafraîchir données de test
```

#### Transactions (--single-transaction)

```bash
# Export cohérent sans bloquer les tables
mariadb-dump -u root -p --single-transaction production_db > backup.sql

# ✅ IMPORTANT pour production :
# - Pas de lock tables (lecture continue possible)
# - Snapshot cohérent InnoDB
# - Recommandé pour bases actives
```

#### Routines et Triggers (--routines --triggers)

```bash
# Inclure procédures, fonctions, triggers
mariadb-dump -u root -p \
  --routines \
  --triggers \
  --events \
  production_db > full_backup.sql

# Par défaut :
# --triggers activé
# --routines désactivé
# --events désactivé
```

#### Format Étendu (--extended-insert)

```bash
# Inserts groupés (plus rapide à restaurer)
mariadb-dump -u root -p --extended-insert production_db > backup.sql

# Génère :
# INSERT INTO table VALUES (row1), (row2), (row3), ...;

# Au lieu de :
# INSERT INTO table VALUES (row1);
# INSERT INTO table VALUES (row2);
# INSERT INTO table VALUES (row3);
```

### Export de Plusieurs Bases

```bash
# Toutes les bases
mariadb-dump -u root -p --all-databases > all_databases.sql

# Bases spécifiques
mariadb-dump -u root -p --databases db1 db2 db3 > multiple_dbs.sql
```

### Export de Tables Spécifiques

```bash
# Une table
mariadb-dump -u root -p production_db customers > customers.sql

# Plusieurs tables
mariadb-dump -u root -p production_db customers orders > tables.sql

# Avec WHERE (filtrer données)
mariadb-dump -u root -p production_db customers \
  --where="created_at >= '2025-01-01'" > recent_customers.sql
```

### Options de Production Recommandées

```bash
# Backup production complet
mariadb-dump -u backup_user -p \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --flush-logs \
  --master-data=2 \
  --hex-blob \
  production_db | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Explication options :
# --single-transaction : Cohérence sans lock
# --routines : Inclure procédures/fonctions
# --triggers : Inclure triggers (défaut)
# --events : Inclure events planifiés
# --flush-logs : Flush binary logs
# --master-data=2 : Position binlog (commenté)
# --hex-blob : Format hex pour BLOB
# | gzip : Compression
```

### Restaurer un Export

```bash
# Restauration simple
mariadb -u root -p production_db < production_backup.sql

# Depuis archive compressée
gunzip < production_backup.sql.gz | mariadb -u root -p production_db

# ou
zcat production_backup.sql.gz | mariadb -u root -p production_db

# Créer base si nécessaire
mariadb -u root -p -e "CREATE DATABASE IF NOT EXISTS production_db"
mariadb -u root -p production_db < production_backup.sql

# Avec verbose
mariadb -u root -p -v production_db < production_backup.sql

# Monitoring progression (avec pv)
pv production_backup.sql | mariadb -u root -p production_db
```

### Backup Incrémental avec Binlogs

```bash
# 1. Backup complet avec position binlog
mariadb-dump -u root -p \
  --single-transaction \
  --master-data=2 \
  --flush-logs \
  production_db > full_backup.sql

# Le fichier contient :
# -- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000042', MASTER_LOG_POS=154;

# 2. Pour restauration point-in-time, rejouer binlogs après full backup
# Extraire position du backup :
head -30 full_backup.sql | grep "CHANGE MASTER"

# 3. Rejouer binlogs depuis cette position
mysqlbinlog --start-position=154 binlog.000042 binlog.000043 \
  | mariadb -u root -p production_db
```

---

## 📊 LOAD DATA - Import de Fichiers CSV

### Syntaxe de Base

```sql
LOAD DATA INFILE '/path/to/file.csv'
INTO TABLE table_name
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;  -- Ignorer header
```

### Exemple Complet : Import CSV

#### Fichier customers.csv
```csv
name,email,phone
Alice Martin,alice@example.com,0601020304
Bob Dupont,bob@example.com,0612345678
Claire Bernard,claire@example.com,0623456789
```

#### Import avec LOAD DATA

```sql
-- 1. Créer la table
CREATE TABLE customers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 2. Importer les données
LOAD DATA INFILE '/tmp/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(name, email, phone);  -- Mapper colonnes
```

### Options Détaillées

#### Délimiteurs

```sql
-- Fichier CSV standard (virgule)
FIELDS TERMINATED BY ','

-- Fichier TSV (tabulation)
FIELDS TERMINATED BY '\t'

-- Fichier pipe-delimited
FIELDS TERMINATED BY '|'

-- Texte encadré par guillemets
ENCLOSED BY '"'

-- Caractère d'échappement
ESCAPED BY '\\'

-- Fin de ligne (Linux/Mac)
LINES TERMINATED BY '\n'

-- Fin de ligne Windows
LINES TERMINATED BY '\r\n'
```

#### Ignorer Lignes

```sql
-- Ignorer header (1 ligne)
IGNORE 1 ROWS

-- Ignorer plusieurs lignes
IGNORE 5 ROWS

-- Limiter le nombre de lignes
LOAD DATA INFILE '/tmp/data.csv'
INTO TABLE test
IGNORE 1 ROWS
LIMIT 1000;  -- Charger seulement 1000 lignes
```

#### Mapper les Colonnes

```sql
-- Ordre différent dans CSV vs table
LOAD DATA INFILE '/tmp/data.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
IGNORE 1 ROWS
(email, name, phone);  -- Ordre dans CSV

-- Avec colonnes calculées
LOAD DATA INFILE '/tmp/data.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
IGNORE 1 ROWS
(name, email, @phone_raw)
SET phone = CONCAT('+33', SUBSTRING(@phone_raw, 2));
```

#### Gestion des Erreurs

```sql
-- Remplacer en cas de duplicate key
LOAD DATA INFILE '/tmp/data.csv'
REPLACE INTO TABLE customers
FIELDS TERMINATED BY ','
IGNORE 1 ROWS;

-- Ignorer les duplicates
LOAD DATA INFILE '/tmp/data.csv'
IGNORE INTO TABLE customers
FIELDS TERMINATED BY ','
IGNORE 1 ROWS;

-- Par défaut : erreur et arrêt sur duplicate
```

### LOAD DATA LOCAL INFILE

```sql
-- Charger depuis client (pas serveur)
LOAD DATA LOCAL INFILE '/local/path/data.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
IGNORE 1 ROWS;
```

⚠️ **Sécurité** :
- Nécessite `local_infile = ON` (client et serveur)
- Vérifier `secure_file_priv` pour LOAD DATA standard
- LOCAL INFILE peut être désactivé pour sécurité

```sql
-- Vérifier configuration
SHOW VARIABLES LIKE 'local_infile';
SHOW VARIABLES LIKE 'secure_file_priv';
```

### Import depuis la Ligne de Commande

```bash
# Via mysqlimport (wrapper de LOAD DATA)
mysqlimport -u root -p \
  --fields-terminated-by=',' \
  --fields-enclosed-by='"' \
  --lines-terminated-by='\n' \
  --ignore-lines=1 \
  production_db \
  /tmp/customers.csv

# Note : Le fichier doit s'appeler comme la table
# customers.csv → table customers
```

---

## 📤 SELECT INTO OUTFILE - Export vers Fichiers

### Syntaxe de Base

```sql
SELECT column1, column2, ...
INTO OUTFILE '/path/to/output.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM table_name
WHERE conditions;
```

### Exemple : Export CSV

```sql
-- Export simple
SELECT name, email, phone
INTO OUTFILE '/tmp/customers_export.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM customers;

-- Résultat : /tmp/customers_export.csv
"Alice Martin","alice@example.com","0601020304"
"Bob Dupont","bob@example.com","0612345678"
```

### Avec Header

```sql
-- Créer header manuellement (MariaDB n'a pas d'option native)
-- Méthode : UNION avec valeurs statiques

SELECT 'name', 'email', 'phone'
UNION ALL
SELECT name, email, phone
INTO OUTFILE '/tmp/customers_with_header.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM customers;

-- Résultat :
"name","email","phone"
"Alice Martin","alice@example.com","0601020304"
```

### Export Filtré

```sql
-- Export avec conditions
SELECT 
  id,
  name,
  email,
  DATE(created_at) AS registration_date
INTO OUTFILE '/tmp/recent_customers.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM customers
WHERE created_at >= '2025-01-01'
ORDER BY created_at DESC;
```

### Export Complexe avec JOIN

```sql
-- Export de rapport avec jointures
SELECT 
  c.name AS customer_name,
  c.email,
  COUNT(o.id) AS total_orders,
  COALESCE(SUM(o.total), 0) AS total_spent
INTO OUTFILE '/tmp/customer_summary.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email
ORDER BY total_spent DESC;
```

### Contraintes et Limitations

⚠️ **Important** :

1. **Le fichier ne doit pas exister**
```sql
-- Erreur si fichier existe
SELECT * INTO OUTFILE '/tmp/existing.csv' FROM customers;
-- ERROR 1086 (HY000): File '/tmp/existing.csv' already exists

-- Solution : supprimer d'abord
-- Via shell : rm /tmp/existing.csv
```

2. **Permissions et secure_file_priv**
```sql
-- Vérifier répertoire autorisé
SHOW VARIABLES LIKE 'secure_file_priv';
-- Résultat : /var/lib/mysql-files/ (ou NULL, ou '')

-- Utiliser répertoire autorisé
SELECT * 
INTO OUTFILE '/var/lib/mysql-files/export.csv'
FROM customers;
```

3. **Privilège FILE requis**
```sql
-- Vérifier privilèges
SHOW GRANTS;
-- Doit contenir : GRANT FILE ON *.* TO ...
```

---

## 🔄 Workflows Complets

### Workflow 1 : Migration Complète d'une Base

```bash
#!/bin/bash
# migration.sh - Migration production → staging

# Variables
SOURCE_HOST="prod-db.example.com"
SOURCE_USER="admin"
SOURCE_DB="production_db"
TARGET_HOST="staging-db.example.com"
TARGET_USER="admin"
TARGET_DB="staging_db"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

echo "=== Migration de $SOURCE_DB vers $TARGET_DB ==="
echo "Date : $DATE"

# 1. Export depuis production
echo "1. Export depuis production..."
mariadb-dump -h $SOURCE_HOST -u $SOURCE_USER -p \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  $SOURCE_DB | gzip > $BACKUP_DIR/${SOURCE_DB}_${DATE}.sql.gz

if [ $? -eq 0 ]; then
  echo "✓ Export réussi"
else
  echo "✗ Erreur lors de l'export"
  exit 1
fi

# 2. Créer base de destination
echo "2. Création base de destination..."
mariadb -h $TARGET_HOST -u $TARGET_USER -p <<EOF
DROP DATABASE IF EXISTS $TARGET_DB;
CREATE DATABASE $TARGET_DB CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
EOF

# 3. Import vers staging
echo "3. Import vers staging..."
gunzip < $BACKUP_DIR/${SOURCE_DB}_${DATE}.sql.gz | \
  mariadb -h $TARGET_HOST -u $TARGET_USER -p $TARGET_DB

if [ $? -eq 0 ]; then
  echo "✓ Import réussi"
else
  echo "✗ Erreur lors de l'import"
  exit 1
fi

# 4. Vérification
echo "4. Vérification..."
mariadb -h $TARGET_HOST -u $TARGET_USER -p $TARGET_DB <<EOF
SELECT 'Tables' AS type, COUNT(*) AS count FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = '$TARGET_DB'
UNION ALL
SELECT 'Rows in customers', COUNT(*) FROM customers
UNION ALL
SELECT 'Rows in orders', COUNT(*) FROM orders;
EOF

echo "=== Migration terminée ==="
```

### Workflow 2 : Backup Quotidien Automatisé

```bash
#!/bin/bash
# daily_backup.sh - Sauvegarde quotidienne

# Configuration
BACKUP_DIR="/backups/mariadb"
RETENTION_DAYS=7
DB_USER="backup_user"
DB_PASS="backup_password"
DATABASES="production_db app_db analytics_db"
DATE=$(date +%Y%m%d)
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Créer répertoire du jour
mkdir -p $BACKUP_DIR/$DATE

# Backup de chaque base
for DB in $DATABASES; do
  echo "Sauvegarde de $DB..."
  
  mariadb-dump -u $DB_USER -p$DB_PASS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --master-data=2 \
    --flush-logs \
    $DB | gzip > $BACKUP_DIR/$DATE/${DB}_${TIMESTAMP}.sql.gz
  
  if [ $? -eq 0 ]; then
    echo "✓ $DB sauvegardé"
  else
    echo "✗ Erreur $DB"
  fi
done

# Purge anciens backups
echo "Purge des backups > $RETENTION_DAYS jours..."
find $BACKUP_DIR -type f -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -type d -empty -delete

echo "Sauvegarde terminée"
```

**Cron pour exécution quotidienne** :
```bash
# Éditer crontab
crontab -e

# Ajouter ligne (tous les jours à 2h du matin)
0 2 * * * /usr/local/bin/daily_backup.sh >> /var/log/mariadb_backup.log 2>&1
```

### Workflow 3 : Refresh Base de Dev depuis Prod

```bash
#!/bin/bash
# refresh_dev.sh - Rafraîchir dev depuis production

PROD_DB="production_db"
DEV_DB="dev_db"
BACKUP_FILE="/tmp/prod_refresh_$(date +%Y%m%d).sql"

echo "=== Refresh Dev Database ==="

# 1. Backup production
echo "1. Export production (structure + données limitées)..."
mariadb-dump -u root -p \
  --single-transaction \
  --routines \
  --triggers \
  $PROD_DB \
  --where="created_at >= CURDATE() - INTERVAL 30 DAY" \
  > $BACKUP_FILE

# 2. Anonymiser données sensibles dans le dump
echo "2. Anonymisation des données..."
sed -i "s/[a-zA-Z0-9._%+-]\+@[a-zA-Z0-9.-]\+\.[a-zA-Z]\{2,\}/dev@example.com/g" $BACKUP_FILE

# 3. Recréer base dev
echo "3. Recréation base dev..."
mariadb -u root -p <<EOF
DROP DATABASE IF EXISTS $DEV_DB;
CREATE DATABASE $DEV_DB CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
EOF

# 4. Import
echo "4. Import vers dev..."
mariadb -u root -p $DEV_DB < $BACKUP_FILE

# 5. Nettoyage
rm $BACKUP_FILE

echo "✓ Dev database refreshed"
```

### Workflow 4 : Export Données pour Analytics

```bash
#!/bin/bash
# export_analytics.sh - Export données pour analyse

EXPORT_DIR="/exports/analytics"
DATE=$(date +%Y%m%d)

mkdir -p $EXPORT_DIR/$DATE

# Export rapports divers en CSV
mariadb -u analytics -p production_db <<EOF

-- Rapport ventes quotidiennes
SELECT 'date', 'orders', 'revenue', 'avg_order'
UNION ALL
SELECT 
  DATE(created_at),
  COUNT(*),
  SUM(total),
  AVG(total)
INTO OUTFILE '$EXPORT_DIR/$DATE/daily_sales.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM orders
WHERE created_at >= CURDATE() - INTERVAL 90 DAY
GROUP BY DATE(created_at)
ORDER BY DATE(created_at);

-- Rapport clients actifs
SELECT 'customer_id', 'customer_name', 'total_orders', 'total_spent'
UNION ALL
SELECT 
  c.id,
  c.name,
  COUNT(o.id),
  COALESCE(SUM(o.total), 0)
INTO OUTFILE '$EXPORT_DIR/$DATE/active_customers.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.created_at >= CURDATE() - INTERVAL 1 YEAR
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 0
ORDER BY total_spent DESC;

EOF

echo "✓ Analytics exports créés dans $EXPORT_DIR/$DATE"
```

---

## 💡 Bonnes Pratiques

### Sauvegardes

✅ **À FAIRE**
- Utiliser `--single-transaction` pour InnoDB (cohérence)
- Compresser les backups (`gzip`)
- Inclure timestamp dans nom fichier
- Tester régulièrement les restaurations
- Stocker backups hors serveur DB
- Automatiser avec cron
- Monitorer succès/échecs
- Garder multiple générations (7 jours, 4 semaines, 12 mois)

❌ **À ÉVITER**
- Backups sur même disque que données
- Pas de test de restauration
- Pas de monitoring
- Permissions trop larges sur fichiers backup
- Mots de passe en clair dans scripts

### Performance Import/Export

```bash
# Import rapide : désactiver temporairement contraintes
mariadb -u root -p production_db <<EOF
SET FOREIGN_KEY_CHECKS=0;
SET UNIQUE_CHECKS=0;
SET AUTOCOMMIT=0;

SOURCE /path/to/large_import.sql;

COMMIT;
SET FOREIGN_KEY_CHECKS=1;
SET UNIQUE_CHECKS=1;
SET AUTOCOMMIT=1;
EOF

# Ou pour mariadb-dump :
mariadb-dump -u root -p \
  --skip-extended-insert \  # Inserts séparés (plus lent mais reprise possible)
  --quick \                 # Ne buffer pas résultats
  --skip-lock-tables \      # Pas de lock (avec --single-transaction)
  production_db > backup.sql
```

### Sécurité

```bash
# Permissions fichiers backup
chmod 600 backup.sql
chown mysql:mysql backup.sql

# Chiffrer backups sensibles
mariadb-dump -u root -p production_db | \
  gzip | \
  openssl enc -aes-256-cbc -salt -out backup.sql.gz.enc

# Déchiffrer
openssl enc -d -aes-256-cbc -in backup.sql.gz.enc | \
  gunzip | \
  mariadb -u root -p production_db

# Fichier credentials sécurisé
# ~/.my.cnf
[client]
user=backup_user
password=secure_password

chmod 600 ~/.my.cnf

# Utiliser sans -p
mariadb-dump production_db > backup.sql
```

---

## ⚠️ Erreurs Courantes et Solutions

### Erreur : "File exists"

```sql
-- Problème
SELECT * INTO OUTFILE '/tmp/export.csv' FROM customers;
-- ERROR 1086 (HY000): File '/tmp/export.csv' already exists

-- Solution
\! rm /tmp/export.csv
-- Puis réessayer
```

### Erreur : "Access denied for FILE privilege"

```sql
-- Problème
SELECT * INTO OUTFILE '/tmp/export.csv' FROM customers;
-- ERROR 1045 (28000): Access denied; you need FILE privilege

-- Solution : Accorder privilège
GRANT FILE ON *.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
```

### Erreur : "The MariaDB server is running with --secure-file-priv"

```sql
-- Problème
SELECT * INTO OUTFILE '/tmp/export.csv' FROM customers;
-- ERROR 1290 (HY000): The MariaDB server is running with --secure-file-priv option

-- Solution : Vérifier répertoire autorisé
SHOW VARIABLES LIKE 'secure_file_priv';
-- Utiliser ce répertoire
SELECT * INTO OUTFILE '/var/lib/mysql-files/export.csv' FROM customers;
```

### Erreur : Import échoue sur Constraint

```bash
# Problème
mariadb -u root -p production_db < backup.sql
# ERROR 1452 (23000): Cannot add or update a child row: foreign key constraint fails

# Solution : Désactiver temporairement
mariadb -u root -p production_db <<EOF
SET FOREIGN_KEY_CHECKS=0;
SOURCE backup.sql;
SET FOREIGN_KEY_CHECKS=1;
EOF
```

### Erreur : "Packet too large"

```bash
# Problème avec gros INSERT
mariadb -u root -p production_db < large_import.sql
# ERROR 1153 (08S01): Got a packet bigger than 'max_allowed_packet' bytes

# Solution : Augmenter limite
mariadb -u root -p --max_allowed_packet=512M production_db < large_import.sql

# Ou modifier configuration serveur
# /etc/mysql/my.cnf
[mysqld]
max_allowed_packet=512M
```

---

## ✅ Points Clés à Retenir

### Commandes Essentielles
- ✅ `SOURCE script.sql` : Exécuter fichier SQL
- ✅ `TEE file.log` : Enregistrer sortie
- ✅ `mariadb-dump` : Export bases de données
- ✅ `LOAD DATA INFILE` : Import CSV
- ✅ `SELECT INTO OUTFILE` : Export vers CSV

### Options Importantes mariadb-dump
- ✅ `--single-transaction` : Cohérence InnoDB
- ✅ `--routines --triggers --events` : Objets programmation
- ✅ `--master-data=2` : Position binlog (PITR)
- ✅ Compression : `| gzip`

### Sécurité
- ✅ Permissions 600 sur fichiers backup
- ✅ Stockage hors serveur DB
- ✅ Chiffrement pour données sensibles
- ✅ Tests réguliers de restauration

### Performance
- ✅ `--extended-insert` pour imports rapides
- ✅ Désactiver checks temporairement si nécessaire
- ✅ Compression pour économiser espace

---

## 🔗 Ressources et Références

### Documentation Officielle
- [mariadb-dump](https://mariadb.com/kb/en/mariadb-dump/)
- [LOAD DATA INFILE](https://mariadb.com/kb/en/load-data-infile/)
- [SELECT INTO OUTFILE](https://mariadb.com/kb/en/select-into-outfile/)
- [SOURCE Command](https://mariadb.com/kb/en/mysql-commands/)
- [Backup and Restore](https://mariadb.com/kb/en/backup-and-restore-overview/)

### Outils Complémentaires
- **Mariabackup** : Backup physique à chaud
- **mydumper/myloader** : Dump parallèle
- **pt-table-checksum** : Vérification cohérence réplication
- **Percona Toolkit** : Suite d'outils administration

---

## ➡️ Retour

**[← B.2 Informations Système](./02-informations-systeme.md)**  
**[← B.1 Connexion et Navigation](./01-connexion-navigation.md)**  
**[← B. Commandes CLI - Introduction](./README.md)**

---

**MariaDB** : 11.8 LTS

⏭️ [Requêtes SQL de Référence](/annexes/requetes-sql-reference/README.md)

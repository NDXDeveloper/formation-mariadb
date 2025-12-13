🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7 Gestion de l'espace disque

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** :
> - Sections 11.1-11.6 (Configuration, logs, maintenance)
> - Chapitre 7 (Moteurs de stockage)
> - Compréhension des systèmes de fichiers
> - Expérience en administration système

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Surveiller** l'utilisation de l'espace disque de MariaDB
- **Identifier** les sources de consommation d'espace
- **Optimiser** l'utilisation de l'espace disque
- **Planifier** les besoins de stockage futurs
- **Gérer** les crises de saturation disque
- **Automatiser** la surveillance et l'alerting
- **Appliquer** les stratégies de compression et d'archivage
- **Exploiter** les nouveautés MariaDB 11.8 pour le contrôle de l'espace

---

## Introduction

La **gestion de l'espace disque** est une responsabilité critique pour tout DBA. Une saturation disque peut entraîner :

- 💥 **Arrêt complet** du serveur MariaDB
- 🔒 **Blocage des transactions** (impossibilité d'écrire)
- ❌ **Corruption de données** (écritures partielles)
- 📉 **Dégradation des performances** (disque plein = fragmentation)
- 🚨 **Perte de service** (indisponibilité applicative)

### Impact d'une saturation disque

```
Disque à 100% :
    → MariaDB ne peut plus écrire
    → Transactions bloquées
    → Applications en erreur
    → Binary logs non écrits (perte réplication)
    → Potentielle corruption de données
    → RTO élevé (Recovery Time Objective)
```

💡 **Règle d'or** : **Prévenir** plutôt que guérir. La surveillance proactive de l'espace disque n'est pas optionnelle.

---

## Anatomie de l'espace disque MariaDB

### Composants principaux

```
/var/lib/mysql/ (datadir)
├── ibdata1                     # InnoDB system tablespace
├── ib_logfile0                 # InnoDB redo log
├── ib_logfile1
├── mysql-bin.000001            # Binary logs
├── mysql-bin.000002
├── mysql-bin.index
├── ecommerce/                  # Base de données
│   ├── orders.ibd              # Table InnoDB (file-per-table)
│   ├── products.ibd
│   └── customers.ibd
├── #innodb_temp/               # Tablespace temporaire InnoDB
│   └── temp_1.ibt
└── mysql/                      # Système
    ├── user.MYD
    └── user.MYI

/var/log/mysql/
├── error.log                   # Error log
├── slow-query.log              # Slow query log
└── archived/                   # Logs archivés

/tmp/ ou /var/tmp/
└── ML*                         # Fichiers temporaires MariaDB
```

### Répartition typique de l'espace

| Composant | % Espace | Taille typique (1TB total) | Croissance |
|-----------|----------|----------------------------|------------|
| **Tables InnoDB (.ibd)** | 60-70% | 600-700 GB | Linéaire avec données |
| **Binary logs** | 10-20% | 100-200 GB | Constante (avec purge) |
| **InnoDB system tablespace** | 5-10% | 50-100 GB | Lente |
| **Logs (error, slow)** | 1-5% | 10-50 GB | Variable |
| **Fichiers temporaires** | 5-10% | 50-100 GB | Pics ponctuels |
| **Index** | Inclus dans tables | N/A | Avec tables |

---

## Surveillance de l'espace disque

### 1. Espace disque système

#### Commande de base

```bash
# Vérifier l'espace disque global
df -h /var/lib/mysql

# Sortie exemple
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       1.0T  750G  250G  75% /var/lib/mysql
```

#### Script de surveillance

```bash
#!/bin/bash
# /usr/local/bin/check-mysql-disk.sh

DATADIR="/var/lib/mysql"
LOGDIR="/var/log/mysql"
THRESHOLD=80  # Alerte si > 80%

check_disk() {
    local dir=$1
    local usage=$(df -h "$dir" | awk 'NR==2 {print $5}' | sed 's/%//')

    if [ "$usage" -ge "$THRESHOLD" ]; then
        echo "ALERTE: $dir à ${usage}% d'utilisation"
        return 1
    else
        echo "OK: $dir à ${usage}% d'utilisation"
        return 0
    fi
}

# Vérifier datadir
check_disk "$DATADIR"
DATA_STATUS=$?

# Vérifier logdir
check_disk "$LOGDIR"
LOG_STATUS=$?

# Alerter si nécessaire
if [ $DATA_STATUS -ne 0 ] || [ $LOG_STATUS -ne 0 ]; then
    echo "Espace disque critique détecté" | \
        mail -s "ALERTE MariaDB Disk Space" dba@example.com
fi
```

### 2. Espace par base de données

```sql
-- Taille de chaque base de données
SELECT
    TABLE_SCHEMA AS 'Base de données',
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Taille (MB)',
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS 'Taille (GB)'
FROM information_schema.TABLES
GROUP BY TABLE_SCHEMA
ORDER BY SUM(DATA_LENGTH + INDEX_LENGTH) DESC;
```

**Sortie exemple** :

```
+----------------+------------+-------------+
| Base de données| Taille (MB)| Taille (GB) |
+----------------+------------+-------------+
| ecommerce      | 524288.50  | 512.00      |
| analytics      | 262144.25  | 256.00      |
| logs           | 131072.13  | 128.00      |
| crm            | 65536.06   | 64.00       |
+----------------+------------+-------------+
```

### 3. Top tables par taille

```sql
-- Top 20 tables les plus volumineuses
SELECT
    TABLE_SCHEMA AS 'Base',
    TABLE_NAME AS 'Table',
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Taille (MB)',
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS 'Données (MB)',
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS 'Index (MB)',
    TABLE_ROWS AS 'Nb lignes',
    ROUND((DATA_LENGTH + INDEX_LENGTH) / TABLE_ROWS, 2) AS 'Octets/ligne'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND TABLE_ROWS > 0
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 20;
```

**Sortie exemple** :

```
+----------+-------------+-----------+-------------+-----------+-----------+--------------+
| Base     | Table       | Taille(MB)| Données(MB) | Index(MB) | Nb lignes | Octets/ligne |
+----------+-------------+-----------+-------------+-----------+-----------+--------------+
| ecommerce| order_items | 102400.50 | 81920.40    | 20480.10  | 500000000 | 214.75       |
| analytics| events      | 51200.25  | 40960.20    | 10240.05  | 250000000 | 214.75       |
| logs     | access_log  | 25600.13  | 20480.10    | 5120.03   | 125000000 | 214.75       |
+----------+-------------+-----------+-------------+-----------+-----------+--------------+
```

### 4. Croissance de l'espace

```sql
-- Créer une table de tracking (à exécuter quotidiennement)
CREATE TABLE IF NOT EXISTS admin.disk_usage_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    check_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    table_schema VARCHAR(64),
    table_name VARCHAR(64),
    data_length BIGINT,
    index_length BIGINT,
    data_free BIGINT,
    table_rows BIGINT,
    INDEX idx_date (check_date),
    INDEX idx_table (table_schema, table_name)
);

-- Capturer l'état actuel
INSERT INTO admin.disk_usage_history
    (table_schema, table_name, data_length, index_length, data_free, table_rows)
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    DATA_LENGTH,
    INDEX_LENGTH,
    DATA_FREE,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys');

-- Analyser la croissance sur 30 jours
SELECT
    table_schema,
    table_name,
    ROUND((MAX(data_length + index_length) - MIN(data_length + index_length)) / 1024 / 1024, 2) AS growth_mb,
    ROUND(((MAX(data_length + index_length) - MIN(data_length + index_length)) / MIN(data_length + index_length)) * 100, 2) AS growth_pct
FROM admin.disk_usage_history
WHERE check_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY table_schema, table_name
HAVING growth_mb > 100  -- Croissance > 100 MB
ORDER BY growth_mb DESC;
```

---

## Sources de consommation d'espace

### 1. Tables de données

#### Croissance normale

```sql
-- Tables avec croissance rapide (> 1 GB/mois)
-- (Nécessite disk_usage_history sur au moins 30 jours)
SELECT
    table_schema,
    table_name,
    ROUND((MAX(data_length + index_length) - MIN(data_length + index_length)) / 1024 / 1024 / 1024, 2) AS growth_gb_month
FROM admin.disk_usage_history
WHERE check_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY table_schema, table_name
HAVING growth_gb_month > 1
ORDER BY growth_gb_month DESC;
```

#### Fragmentation excessive

```sql
-- Tables fragmentées gaspillant de l'espace
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb,
    ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND DATA_FREE > 0
    AND (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) > 0.15  -- > 15%
ORDER BY free_mb DESC;
```

### 2. Binary logs

```sql
-- Taille totale des binary logs
SELECT
    CONCAT(ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2), ' GB') AS total_binlog_size,
    COUNT(*) AS nb_binlogs,
    CONCAT(ROUND(AVG(FILE_SIZE) / 1024 / 1024, 2), ' MB') AS avg_binlog_size
FROM information_schema.BINARY_LOGS;

-- Détail par fichier
SELECT
    LOG_NAME,
    ROUND(FILE_SIZE / 1024 / 1024, 2) AS size_mb,
    CREATED AS creation_date,
    TIMESTAMPDIFF(DAY, CREATED, NOW()) AS age_days
FROM information_schema.BINARY_LOGS
ORDER BY CREATED DESC;
```

### 3. InnoDB tablespace

```bash
# Taille du system tablespace
ls -lh /var/lib/mysql/ibdata1

# Taille des redo logs
ls -lh /var/lib/mysql/ib_logfile*

# Taille du tablespace temporaire
ls -lh /var/lib/mysql/#innodb_temp/temp_*.ibt
```

```sql
-- Vérifier la configuration InnoDB
SHOW VARIABLES LIKE 'innodb_data_file_path';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_temp_data_file_path';
```

### 4. Fichiers temporaires

```bash
# Fichiers temporaires MariaDB
ls -lh /tmp/ML* 2>/dev/null | wc -l
du -sh /tmp/ML* 2>/dev/null

# Tablespace temporaire InnoDB
du -sh /var/lib/mysql/#innodb_temp/
```

```sql
-- 🆕 MariaDB 11.8 : Surveillance espace temporaire
SHOW STATUS LIKE 'Created_tmp%';

-- Variables de contrôle (nouveauté 11.8)
SHOW VARIABLES LIKE '%tmp_space%';
-- max_tmp_space_usage (par session)
-- max_total_tmp_space_usage (global)
```

### 5. Logs (error, slow query, general)

```bash
# Taille des logs
du -sh /var/log/mysql/

# Détail par fichier
ls -lh /var/log/mysql/ | grep -v "^d"

# Logs non rotatés (danger)
find /var/log/mysql/ -name "*.log" -size +1G
```

---

## Stratégies d'optimisation de l'espace

### 1. Compression des tables InnoDB

MariaDB supporte plusieurs méthodes de compression :

#### Page Compression (recommandée)

```sql
-- Créer une table avec compression de page
CREATE TABLE compressed_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME,
    message TEXT,
    data JSON
) ENGINE=InnoDB
  PAGE_COMPRESSED=1
  PAGE_COMPRESSION_LEVEL=9;  -- 1-9, 9 = max compression

-- Convertir une table existante
ALTER TABLE logs
    PAGE_COMPRESSED=1,
    PAGE_COMPRESSION_LEVEL=6;
```

**Gains typiques** : 50-70% de réduction selon les données.

#### ROW_FORMAT=COMPRESSED (legacy)

```sql
-- Ancienne méthode (toujours supportée)
CREATE TABLE old_compressed (
    id INT PRIMARY KEY,
    data TEXT
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED
  KEY_BLOCK_SIZE=8;  -- 1, 2, 4, 8, 16 KB

-- Vérifier les tables compressées
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROW_FORMAT,
    CREATE_OPTIONS
FROM information_schema.TABLES
WHERE ROW_FORMAT = 'Compressed' OR CREATE_OPTIONS LIKE '%PAGE_COMPRESSED%';
```

### 2. Purge des données anciennes

#### Stratégie de rétention

```sql
-- Identifier les données candidates à la purge
SELECT
    'orders' AS table_name,
    COUNT(*) AS old_rows,
    ROUND(COUNT(*) * 214 / 1024 / 1024, 2) AS estimated_mb  -- 214 octets/ligne moyen
FROM orders
WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR);

-- Purge par batch (évite locks longs)
-- Batch de 10,000 lignes
DELETE FROM orders
WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR)
LIMIT 10000;

-- Répéter jusqu'à 0 ligne affectée
-- Ou automatiser avec script
```

#### Script de purge automatisé

```bash
#!/bin/bash
# /usr/local/bin/purge-old-data.sh

DB="ecommerce"
TABLE="access_logs"
RETENTION_DAYS=90
BATCH_SIZE=10000

while true; do
    # Purger un batch
    ROWS=$(mariadb -sN -e "
        DELETE FROM $DB.$TABLE
        WHERE created_at < DATE_SUB(NOW(), INTERVAL $RETENTION_DAYS DAY)
        LIMIT $BATCH_SIZE;
        SELECT ROW_COUNT();
    " | tail -1)

    echo "Purgé $ROWS lignes de $DB.$TABLE"

    # Arrêter si plus rien à purger
    if [ "$ROWS" -eq 0 ]; then
        break
    fi

    # Pause pour ne pas surcharger
    sleep 1
done

# OPTIMIZE après purge massive
mariadb -e "OPTIMIZE TABLE $DB.$TABLE"
```

### 3. Archivage vers stockage froid

#### Moteur S3 (MariaDB ColumnStore)

```sql
-- Archiver les anciennes données vers S3
-- (Nécessite MariaDB ColumnStore avec S3 plugin)

-- 1. Créer table archive sur S3
CREATE TABLE orders_archive (
    id BIGINT,
    customer_id INT,
    order_date DATE,
    total DECIMAL(10,2),
    -- ...
) ENGINE=S3
  S3_BUCKET='company-archive'
  S3_REGION='us-east-1'
  S3_ACCESS_KEY='...'
  S3_SECRET_KEY='...';

-- 2. Copier données anciennes
INSERT INTO orders_archive
SELECT * FROM orders
WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 2 YEAR);

-- 3. Purger de la table principale
DELETE FROM orders
WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 2 YEAR);

-- 4. OPTIMIZE
OPTIMIZE TABLE orders;
```

#### Archivage vers fichiers

```bash
#!/bin/bash
# Archive vers fichiers CSV compressés

YEAR=$(date -d "2 years ago" +%Y)
ARCHIVE_DIR="/backup/archives/$YEAR"

mkdir -p "$ARCHIVE_DIR"

# Export vers CSV
mariadb -e "
    SELECT * FROM ecommerce.orders
    WHERE YEAR(order_date) = $YEAR
    INTO OUTFILE '$ARCHIVE_DIR/orders_$YEAR.csv'
    FIELDS TERMINATED BY ','
    ENCLOSED BY '\"'
    LINES TERMINATED BY '\n';
"

# Compresser
gzip "$ARCHIVE_DIR/orders_$YEAR.csv"

# Purger après vérification
mariadb -e "
    DELETE FROM ecommerce.orders
    WHERE YEAR(order_date) = $YEAR;
"
```

### 4. Partitionnement de tables

Le **partitionnement** facilite la purge de données anciennes sans fragmentation.

```sql
-- Créer table partitionnée par mois
CREATE TABLE events_partitioned (
    id BIGINT AUTO_INCREMENT,
    event_date DATE NOT NULL,
    event_type VARCHAR(50),
    data JSON,
    PRIMARY KEY (id, event_date)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(event_date)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    -- ...
    PARTITION p202512 VALUES LESS THAN (TO_DAYS('2026-01-01')),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Purge ultra-rapide d'une partition complète
ALTER TABLE events_partitioned DROP PARTITION p202401;
-- Instantané, pas de fragmentation !
```

### 5. Gestion des binary logs

```sql
-- Vérifier la rétention actuelle
SHOW VARIABLES LIKE 'expire_logs_days';

-- Ajuster la rétention (7 jours)
SET GLOBAL expire_logs_days = 7;

-- Purge manuelle immédiate
PURGE BINARY LOGS BEFORE DATE(NOW() - INTERVAL 3 DAY);

-- Vérifier l'espace récupéré
SELECT
    CONCAT(ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2), ' GB') AS total_binlog_size
FROM information_schema.BINARY_LOGS;
```

### 6. Optimisation du tablespace système InnoDB

#### innodb_file_per_table

```sql
-- Vérifier la configuration
SHOW VARIABLES LIKE 'innodb_file_per_table';
-- ON depuis MariaDB 5.6 (défaut)

-- Avec file_per_table=ON :
-- - Chaque table a son propre fichier .ibd
-- - DROP TABLE libère immédiatement l'espace
-- - TRUNCATE TABLE libère immédiatement l'espace
```

#### Shrink du system tablespace (complexe)

⚠️ **Attention** : Le system tablespace (`ibdata1`) **ne rétrécit jamais** automatiquement.

**Seule solution** : Dump/Restore complet.

```bash
#!/bin/bash
# Réduire ibdata1 (nécessite downtime)

# 1. Dump complet
mysqldump --all-databases --routines --triggers > /backup/full-dump.sql

# 2. Arrêter MariaDB
systemctl stop mariadb

# 3. Sauvegarder ibdata1
mv /var/lib/mysql/ibdata1 /backup/ibdata1.old

# 4. Supprimer redo logs
rm /var/lib/mysql/ib_logfile*

# 5. Redémarrer MariaDB (nouveau ibdata1 créé)
systemctl start mariadb

# 6. Restore
mysql < /backup/full-dump.sql

# 7. Vérifier la taille
ls -lh /var/lib/mysql/ibdata1
```

---

## Gestion des crises de saturation disque

### Procédure d'urgence

#### Étape 1 : Identifier les gros consommateurs

```bash
# Top 10 fichiers les plus gros dans datadir
find /var/lib/mysql -type f -exec du -h {} + | sort -rh | head -20

# Espace par répertoire
du -sh /var/lib/mysql/*/
```

#### Étape 2 : Actions immédiates

```sql
-- 1. Purger les binary logs (libère immédiatement)
PURGE BINARY LOGS BEFORE DATE(NOW() - INTERVAL 1 DAY);

-- 2. Désactiver temporairement le general log (si activé)
SET GLOBAL general_log = OFF;

-- 3. Tronquer les tables de logs non critiques
TRUNCATE TABLE mysql.slow_log;
TRUNCATE TABLE mysql.general_log;
```

```bash
# 4. Rotation forcée des logs
mariadb-admin flush-logs

# 5. Purger les anciens logs
find /var/log/mysql -name "*.log.*" -mtime +7 -delete

# 6. Compresser les logs actuels
gzip /var/log/mysql/slow-query.log.1
```

#### Étape 3 : Libération d'espace temporaire

```bash
# Nettoyer les fichiers temporaires
rm -f /tmp/ML*
rm -f /tmp/sql*
rm -f /tmp/mysql*
```

```sql
-- Tronquer le tablespace temporaire (redémarrage nécessaire)
-- Ou attendre le prochain redémarrage
```

#### Étape 4 : Tables candidates à la purge

```sql
-- Identifier tables temporaires ou de cache
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_NAME LIKE '%cache%'
    OR TABLE_NAME LIKE '%temp%'
    OR TABLE_NAME LIKE '%tmp%'
ORDER BY size_mb DESC;

-- Tronquer si sûr
TRUNCATE TABLE application_cache;
```

---

## Surveillance et alerting automatisés

### Script de monitoring complet

```bash
#!/bin/bash
# /usr/local/bin/monitor-mysql-disk.sh

DATADIR="/var/lib/mysql"
LOG_FILE="/var/log/mysql-disk-monitor.log"
ALERT_EMAIL="dba@example.com"

# Seuils
WARN_THRESHOLD=75
CRIT_THRESHOLD=90

# Fonction de log
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Fonction d'alerte
alert() {
    local severity=$1
    local message=$2
    log "[$severity] $message"
    echo "$message" | mail -s "[$severity] MariaDB Disk Alert" "$ALERT_EMAIL"
}

# 1. Vérifier l'espace disque système
DISK_USAGE=$(df -h "$DATADIR" | awk 'NR==2 {print $5}' | sed 's/%//')
log "Espace disque: ${DISK_USAGE}%"

if [ "$DISK_USAGE" -ge "$CRIT_THRESHOLD" ]; then
    alert "CRITICAL" "Espace disque à ${DISK_USAGE}% (seuil critique: ${CRIT_THRESHOLD}%)"
elif [ "$DISK_USAGE" -ge "$WARN_THRESHOLD" ]; then
    alert "WARNING" "Espace disque à ${DISK_USAGE}% (seuil avertissement: ${WARN_THRESHOLD}%)"
fi

# 2. Taille des binary logs
BINLOG_SIZE=$(mariadb -sN -e "
    SELECT ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2)
    FROM information_schema.BINARY_LOGS
")
log "Binary logs: ${BINLOG_SIZE} GB"

if [ $(echo "$BINLOG_SIZE > 200" | bc) -eq 1 ]; then
    alert "WARNING" "Binary logs volumineux: ${BINLOG_SIZE} GB"
fi

# 3. Tables les plus volumineuses
log "Top 5 tables par taille:"
mariadb -sN -e "
    SELECT
        CONCAT(TABLE_SCHEMA, '.', TABLE_NAME) AS table_name,
        ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS size_gb
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
    LIMIT 5
" | while read table size; do
    log "  $table: ${size} GB"
done

# 4. Fragmentation excessive
FRAGMENTED=$(mariadb -sN -e "
    SELECT COUNT(*)
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
        AND DATA_FREE > 0
        AND (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) > 0.20
")

if [ "$FRAGMENTED" -gt 0 ]; then
    alert "INFO" "$FRAGMENTED tables avec fragmentation > 20%"
fi

log "Fin du monitoring"
```

### Configuration Nagios/Icinga

```bash
# /etc/nagios/nrpe.d/mariadb-disk.cfg

# Vérifier espace disque datadir
command[check_mysql_datadir]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /var/lib/mysql

# Vérifier espace disque logs
command[check_mysql_logs]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /var/log/mysql

# Vérifier taille binary logs
command[check_mysql_binlogs]=/usr/local/bin/check_binlog_size.sh -w 150 -c 200
```

### Prometheus Exporter

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mariadb'
    static_configs:
      - targets: ['localhost:9104']
```

**Métriques clés** :
- `mysql_global_status_innodb_data_written` : Octets écrits InnoDB
- `mysql_global_variables_innodb_buffer_pool_size` : Taille buffer pool
- `mysql_info_schema_table_size` : Taille des tables

---

## Planification de la capacité

### Projection de croissance

```sql
-- Croissance quotidienne moyenne (sur 30 jours)
SELECT
    table_schema,
    table_name,
    ROUND(
        (MAX(data_length + index_length) - MIN(data_length + index_length))
        / DATEDIFF(MAX(check_date), MIN(check_date))
        / 1024 / 1024, 2
    ) AS daily_growth_mb
FROM admin.disk_usage_history
WHERE check_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY table_schema, table_name
HAVING daily_growth_mb > 10  -- Croissance > 10 MB/jour
ORDER BY daily_growth_mb DESC;

-- Projection à 6 mois
-- Supposons croissance moyenne de 500 MB/jour
-- 500 MB * 180 jours = 90 GB
```

### Calcul des besoins futurs

```sql
-- Besoins en stockage projetés
SELECT
    table_schema,
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS current_gb,
    ROUND((
        SUM(data_length + index_length) *
        (1 + (180 * 0.01))  -- Croissance 1%/jour * 180 jours
    ) / 1024 / 1024 / 1024, 2) AS projected_6m_gb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
GROUP BY table_schema;
```

---

## Bonnes pratiques de gestion de l'espace

### ✅ À FAIRE

1. **Surveiller quotidiennement** : Alertes à 75% et 90%
2. **Purger régulièrement** : Binary logs, logs applicatifs, données anciennes
3. **Compresser** : Tables volumineuses (PAGE_COMPRESSED)
4. **Partitionner** : Tables avec rétention (DROP PARTITION)
5. **OPTIMIZE mensuellement** : Tables fragmentées > 15%
6. **Archiver** : Données froides vers S3 ou fichiers
7. **Planifier la capacité** : Projection à 6-12 mois
8. **Documenter** : Politique de rétention, procédures d'urgence
9. **Automatiser** : Scripts de purge, monitoring, alerting
10. **innodb_file_per_table = ON** : Libération espace immédiate

### ❌ À ÉVITER

1. **Négliger la surveillance** : Saturation soudaine
2. **Désactiver binary logs** pour libérer espace (perte réplication/PITR)
3. **Supprimer ibdata1** manuellement (corruption garantie)
4. **OPTIMIZE toutes les tables** simultanément (downtime)
5. **Conserver General Log** activé en production
6. **Pas de rétention définie** : Croissance infinie
7. **Fragmentation > 30%** ignorée
8. **Pas de plan d'urgence** documenté
9. **Projections ignorées** : Saturation prévisible mais non anticipée
10. **Sauvegardes sur même disque** que datadir

---

## Checklist de gestion d'espace

### Quotidienne (automatisée)

- [ ] Vérifier utilisation disque (datadir, logs)
- [ ] Surveiller taille binary logs
- [ ] Alerter si > 80% utilisation
- [ ] Vérifier fichiers temporaires orphelins

### Hebdomadaire

- [ ] Analyser croissance tables (top 20)
- [ ] Vérifier fragmentation > 15%
- [ ] Purger logs applicatifs anciens
- [ ] Capturer métriques de croissance

### Mensuelle

- [ ] OPTIMIZE tables fragmentées
- [ ] Purger données selon politique rétention
- [ ] Analyser tendances de croissance
- [ ] Réviser projections capacité

### Trimestrielle

- [ ] Audit complet utilisation espace
- [ ] Réviser politique de rétention
- [ ] Planifier besoins stockage futurs (6-12 mois)
- [ ] Test procédure urgence saturation

---

## ✅ Points clés à retenir

- **Surveillance proactive** : Alertes à 75% (warning) et 90% (critical)
- **Sources principales** : Tables (60-70%), Binary logs (10-20%), Temporaires (5-10%)
- **Fragmentation** : OPTIMIZE si > 15%, critique si > 30%
- **Compression** : PAGE_COMPRESSED=1 pour économiser 50-70%
- **Purge régulière** : Binary logs (expire_logs_days), données anciennes, logs
- **Partitionnement** : DROP PARTITION pour purge ultra-rapide
- **Archivage** : S3, fichiers compressés pour données froides
- **Urgence** : Purger binlogs, désactiver general log, nettoyer /tmp
- **Planification** : Projection croissance à 6-12 mois
- **innodb_file_per_table** : ON pour libération espace immédiate
- 🆕 **MariaDB 11.8** : max_tmp_space_usage, max_total_tmp_space_usage
- **Automatisation** : Scripts quotidiens de surveillance et purge

---

## 🔗 Ressources et références

- [📖 Documentation officielle - InnoDB Tablespaces](https://mariadb.com/kb/en/innodb-tablespaces/)
- [📖 Documentation officielle - InnoDB Compression](https://mariadb.com/kb/en/innodb-page-compression/)
- [📖 Documentation officielle - Table Partitioning](https://mariadb.com/kb/en/partitioning-overview/)
- [📖 Documentation officielle - S3 Storage Engine](https://mariadb.com/kb/en/s3-storage-engine/)
- [🔧 Percona Toolkit - pt-archiver](https://www.percona.com/doc/percona-toolkit/LATEST/pt-archiver.html)
- [📊 Prometheus MySQL Exporter](https://github.com/prometheus/mysqld_exporter)

---

## ➡️ Section suivante

**[11.8 Contrôle espace temporaire (max_tmp_space_usage, max_total_tmp_space_usage)](./08-controle-espace-temporaire.md)** 🆕 : Nouveauté MariaDB 11.8 pour limiter l'utilisation de l'espace temporaire et prévenir les saturations disque.

---

**💡 Conseil final** : L'espace disque est comme l'essence dans une voiture : quand vous arrivez à zéro, c'est trop tard. Surveillez, anticipez, planifiez. La saturation disque est 100% évitable avec une bonne discipline ! ⚠️💾

⏭️ [Contrôle espace temporaire (max_tmp_space_usage, max_total_tmp_space_usage)](/11-administration-configuration/08-controle-espace-temporaire.md)

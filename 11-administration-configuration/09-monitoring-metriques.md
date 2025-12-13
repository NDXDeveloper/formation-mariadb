🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.9 Monitoring et métriques importantes

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** :
> - Sections 11.1-11.8 (Configuration, logs, maintenance, espace disque)
> - Chapitre 7 (Moteurs de stockage)
> - Compréhension des performances système
> - Expérience en administration

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Identifier** les métriques critiques à surveiller en production
- **Utiliser** SHOW STATUS, SHOW VARIABLES et Performance Schema
- **Surveiller** les performances (Buffer Pool, requêtes, threads)
- **Détecter** les problèmes avant qu'ils n'impactent les utilisateurs
- **Configurer** des alertes proactives
- **Exploiter** les outils de monitoring (Prometheus, Grafana, PMM)
- **Analyser** les tendances et planifier la capacité
- **Monitorer** le Thread Pool et les nouveautés MariaDB 11.8

---

## Introduction

Le **monitoring** est la pratique de **surveiller continuellement** l'état et les performances de MariaDB pour :

- 🚨 **Détecter** les anomalies avant qu'elles ne deviennent critiques
- 📊 **Mesurer** les performances et identifier les goulots d'étranglement
- 📈 **Analyser** les tendances pour la planification de capacité
- ⚡ **Optimiser** les performances basées sur des métriques réelles
- 🔍 **Diagnostiquer** rapidement les incidents
- 📉 **Valider** l'impact des changements de configuration

### Monitoring proactif vs réactif

```
Monitoring RÉACTIF (mauvais):
    Problème → Utilisateurs se plaignent → Investigation → Résolution
    RTO élevé, Impact utilisateurs, Stress

Monitoring PROACTIF (bon):
    Métrique dégradée → Alerte automatique → Investigation → Résolution préventive
    RTO faible, Pas d'impact utilisateurs, Contrôle
```

💡 **Principe fondamental** : "You can't improve what you don't measure" - Peter Drucker. Le monitoring n'est pas optionnel en production.

---

## Métriques essentielles par catégorie

### 1. Santé générale du serveur

#### Uptime et disponibilité

```sql
-- Temps depuis le dernier démarrage
SHOW STATUS LIKE 'Uptime';
-- Uptime = 2592000 (30 jours)

-- Nombre de connexions échouées
SHOW STATUS LIKE 'Aborted_connects';
SHOW STATUS LIKE 'Aborted_clients';

-- Threads actifs
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';
SHOW STATUS LIKE 'Max_used_connections';
```

**Métriques clés** :

| Métrique | Description | Seuil alerte |
|----------|-------------|--------------|
| `Uptime` | Secondes depuis démarrage | - |
| `Aborted_connects` | Échecs de connexion | Augmentation soudaine |
| `Threads_connected` | Connexions actives | > 80% de max_connections |
| `Threads_running` | Requêtes en exécution | > 50 (serveur surchargé) |
| `Max_used_connections` | Pic connexions | > 80% de max_connections |

#### Version et configuration

```sql
-- Version MariaDB
SELECT VERSION();
-- 11.8.0-MariaDB

-- Variables critiques
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'query_cache_type';
```

### 2. Performance des requêtes

#### Slow queries

```sql
-- Requêtes lentes
SHOW STATUS LIKE 'Slow_queries';

-- Taux de requêtes lentes
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Slow_queries') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Questions') * 100
    AS slow_query_rate;
```

**Interprétation** :
- **< 0.05%** : Excellent
- **0.05-0.5%** : Acceptable
- **> 0.5%** : Investigation nécessaire

#### Requêtes par seconde (QPS)

```sql
-- Questions (requêtes totales)
SHOW STATUS LIKE 'Questions';

-- Calcul QPS (sur 1 minute)
-- Nécessite 2 mesures à 60 secondes d'intervalle
-- QPS = (Questions_t2 - Questions_t1) / 60
```

#### Débit de requêtes par type

```sql
-- Répartition des opérations
SHOW STATUS LIKE 'Com_select';
SHOW STATUS LIKE 'Com_insert';
SHOW STATUS LIKE 'Com_update';
SHOW STATUS LIKE 'Com_delete';

-- Ratio lecture/écriture
SELECT
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Com_select') /
        ((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Com_insert') +
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Com_update') +
         (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Com_delete')),
        2
    ) AS read_write_ratio;
```

### 3. InnoDB Buffer Pool (métrique #1)

Le **Buffer Pool** est la **métrique la plus critique** pour InnoDB.

```sql
-- Taille du Buffer Pool
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Statistiques d'utilisation
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_total';
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_free';
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_data';
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_dirty';

-- Taux de hit du Buffer Pool (CRITIQUE)
SHOW STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW STATUS LIKE 'Innodb_buffer_pool_reads';
```

**Calcul du hit rate** :

```sql
SELECT
    ROUND(
        (1 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
        )) * 100,
        2
    ) AS buffer_pool_hit_rate_pct;
```

**Objectifs** :
- **> 99%** : Excellent (recommandé)
- **95-99%** : Acceptable
- **< 95%** : Buffer Pool trop petit, augmenter innodb_buffer_pool_size

#### Pages dirty ratio

```sql
SELECT
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') * 100,
        2
    ) AS dirty_pages_pct;
```

**Interprétation** :
- **< 75%** : Normal
- **75-90%** : Haute charge écriture, surveiller
- **> 90%** : Flush trop lent, risque de stall

### 4. Connexions

```sql
-- Connexions actuelles vs max
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected') AS current,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Max_used_connections') AS peak,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'max_connections') AS max_allowed,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'max_connections') * 100,
        2
    ) AS utilization_pct;
```

**Alertes** :
- **> 80%** : Warning - Approche de la limite
- **> 90%** : Critical - Risque de refus de connexion

#### Connexions refusées

```sql
-- Erreurs de connexion
SHOW STATUS LIKE 'Connection_errors_max_connections';
SHOW STATUS LIKE 'Connection_errors_internal';
SHOW STATUS LIKE 'Connection_errors_accept';
```

### 5. Tables temporaires

```sql
-- Tables temporaires créées
SHOW STATUS LIKE 'Created_tmp_tables';
SHOW STATUS LIKE 'Created_tmp_disk_tables';

-- Ratio tables sur disque (doit être < 25%)
SELECT
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_tables') * 100,
        2
    ) AS tmp_disk_ratio_pct;
```

### 6. Verrous et concurrence

```sql
-- Attentes de verrous InnoDB
SHOW STATUS LIKE 'Innodb_row_lock_waits';
SHOW STATUS LIKE 'Innodb_row_lock_time';
SHOW STATUS LIKE 'Innodb_row_lock_time_avg';

-- Table locks
SHOW STATUS LIKE 'Table_locks_waited';
SHOW STATUS LIKE 'Table_locks_immediate';

-- Deadlocks
SHOW STATUS LIKE 'Innodb_deadlocks';
```

### 7. Réplication (si applicable)

```sql
-- État réplication sur slave
SHOW SLAVE STATUS\G

-- Métriques critiques :
-- - Slave_IO_Running: Yes
-- - Slave_SQL_Running: Yes
-- - Seconds_Behind_Master: < 10 (idéalement 0)
-- - Last_Error: (vide)
```

**Lag de réplication** :

```sql
-- Sur le slave
SELECT
    TIMESTAMPDIFF(SECOND,
        STR_TO_DATE(SUBSTRING_INDEX(SUBSTRING_INDEX(SHOW_SLAVE_STATUS, 'Read_Master_Log_Pos', -1), '\n', 1), '%Y%m%d%H%i%s'),
        NOW()
    ) AS replication_lag_seconds;
```

### 8. Espace disque et I/O

```sql
-- Écritures InnoDB
SHOW STATUS LIKE 'Innodb_data_written';
SHOW STATUS LIKE 'Innodb_data_read';

-- Opérations fsync
SHOW STATUS LIKE 'Innodb_data_fsyncs';

-- Logs
SHOW STATUS LIKE 'Innodb_os_log_written';

-- Taille des bases
SELECT
    TABLE_SCHEMA,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS size_gb
FROM information_schema.TABLES
GROUP BY TABLE_SCHEMA;
```

---

## Outils de monitoring natifs MariaDB

### SHOW STATUS

```sql
-- Toutes les statistiques
SHOW GLOBAL STATUS;

-- Filtrage
SHOW GLOBAL STATUS LIKE 'Innodb%';
SHOW GLOBAL STATUS LIKE '%connect%';

-- Format vertical (plus lisible)
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%'\G
```

### SHOW VARIABLES

```sql
-- Configuration actuelle
SHOW GLOBAL VARIABLES;

-- Filtrage
SHOW VARIABLES LIKE 'innodb%';
SHOW VARIABLES LIKE '%cache%';
```

### SHOW PROCESSLIST

```sql
-- Requêtes en cours d'exécution
SHOW FULL PROCESSLIST;

-- Identification requêtes lentes
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE,
    LEFT(INFO, 100) AS QUERY_PREVIEW
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
    AND TIME > 5  -- Plus de 5 secondes
ORDER BY TIME DESC;
```

### INFORMATION_SCHEMA

```sql
-- Tables volumineuses
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 20;

-- Fragmentation
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb,
    ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND DATA_FREE > 0
ORDER BY fragmentation_pct DESC
LIMIT 20;
```

### Performance Schema

**Activation** :

```ini
# my.cnf
[mysqld]
performance_schema = ON
```

**Exemples d'utilisation** :

```sql
-- Top requêtes par temps total
SELECT
    DIGEST_TEXT AS query,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000, 3) AS avg_seconds,
    ROUND(SUM_TIMER_WAIT / 1000000000, 3) AS total_seconds
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Requêtes avec full table scans
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_NO_INDEX_USED,
    SUM_NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
   OR SUM_NO_GOOD_INDEX_USED > 0
ORDER BY SUM_NO_INDEX_USED DESC
LIMIT 10;

-- Attentes I/O par fichier
SELECT
    FILE_NAME,
    COUNT_STAR AS events,
    ROUND(SUM_TIMER_WAIT / 1000000000, 3) AS total_wait_seconds
FROM performance_schema.file_summary_by_instance
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

## Thread Pool (monitoring spécifique)

### Configuration du Thread Pool

```sql
-- Vérifier si Thread Pool est actif
SHOW VARIABLES LIKE 'thread_handling';
-- thread_handling = pool-of-threads

-- Variables Thread Pool
SHOW VARIABLES LIKE 'thread_pool%';
```

**Variables importantes** :

| Variable | Description | Valeur recommandée |
|----------|-------------|-------------------|
| `thread_pool_size` | Nombre de groupes | = Nombre de CPU cores |
| `thread_pool_max_threads` | Threads max total | 1000-2000 |
| `thread_pool_idle_timeout` | Timeout thread inactif | 60 secondes |
| `thread_pool_stall_limit` | Détection stall (ms) | 500 |
| `thread_pool_oversubscribe` | Threads actifs/groupe | 3 |

### Métriques Thread Pool

```sql
-- Statistiques Thread Pool
SHOW STATUS LIKE 'Threadpool%';
```

**Métriques clés** :

```sql
-- Threads actifs
SHOW STATUS LIKE 'Threadpool_threads';

-- Threads idle
SHOW STATUS LIKE 'Threadpool_idle_threads';

-- Stalls détectés
SHOW STATUS LIKE 'Threadpool_stall_count';
```

**Analyse de santé** :

```sql
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threadpool_threads') AS total_threads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threadpool_idle_threads') AS idle_threads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected') AS connections,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'thread_pool_size') AS pool_groups;
```

**Alertes Thread Pool** :
- **Threadpool_stall_count** augmente rapidement → Requêtes lentes bloquent le pool
- **Threadpool_idle_threads = 0** → Pool saturé
- **Threadpool_threads proche de thread_pool_max_threads** → Augmenter la limite

---

## Dashboard de monitoring complet

### Script SQL de tableau de bord

```sql
-- ========================================
-- MARIADB MONITORING DASHBOARD
-- ========================================

-- 1. SANTÉ GÉNÉRALE
SELECT 'SANTÉ GÉNÉRALE' AS section;

SELECT
    'Version' AS metric,
    VERSION() AS value;

SELECT
    'Uptime' AS metric,
    CONCAT(
        FLOOR((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Uptime') / 86400), ' jours ',
        FLOOR(((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Uptime') % 86400) / 3600), ' heures'
    ) AS value;

-- 2. CONNEXIONS
SELECT 'CONNEXIONS' AS section;

SELECT
    'Connexions actuelles' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected') AS value,
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'max_connections') * 100,
            2
        ), '%'
    ) AS utilization;

SELECT
    'Threads running' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_running') AS value;

SELECT
    'Connexions refusées' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Connection_errors_max_connections') AS value;

-- 3. BUFFER POOL
SELECT 'BUFFER POOL INNODB' AS section;

SELECT
    'Buffer Pool Size' AS metric,
    CONCAT(
        ROUND((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'innodb_buffer_pool_size') / 1024 / 1024 / 1024, 2),
        ' GB'
    ) AS value;

SELECT
    'Buffer Pool Hit Rate' AS metric,
    CONCAT(
        ROUND(
            (1 - (
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
                (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
            )) * 100,
            2
        ), '%'
    ) AS value;

SELECT
    'Dirty Pages' AS metric,
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') * 100,
            2
        ), '%'
    ) AS value;

-- 4. REQUÊTES
SELECT 'REQUÊTES' AS section;

SELECT
    'Slow Queries' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Slow_queries') AS value;

SELECT
    'Tables Temporaires (Disque)' AS metric,
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_tables') * 100,
            2
        ), '%'
    ) AS value;

-- 5. VERROUS
SELECT 'VERROUS ET CONCURRENCE' AS section;

SELECT
    'Deadlocks' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_deadlocks') AS value;

SELECT
    'Row Lock Waits' AS metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_row_lock_waits') AS value;

-- 6. TOP TABLES
SELECT 'TOP 5 TABLES PAR TAILLE' AS section;

SELECT
    CONCAT(TABLE_SCHEMA, '.', TABLE_NAME) AS table_name,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 5;
```

### Script Bash de monitoring système

```bash
#!/bin/bash
# /usr/local/bin/mariadb-monitoring.sh

LOG_FILE="/var/log/mysql/monitoring-$(date +%Y%m%d).log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== MARIADB MONITORING ==="

# 1. Uptime
UPTIME=$(mariadb -sN -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Uptime'")
UPTIME_DAYS=$((UPTIME / 86400))
log "Uptime: $UPTIME_DAYS jours"

# 2. Connexions
CONN_CURRENT=$(mariadb -sN -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected'")
CONN_MAX=$(mariadb -sN -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'max_connections'")
CONN_PCT=$((CONN_CURRENT * 100 / CONN_MAX))
log "Connexions: $CONN_CURRENT / $CONN_MAX ($CONN_PCT%)"

if [ $CONN_PCT -gt 80 ]; then
    log "ALERTE: Connexions > 80%"
fi

# 3. Buffer Pool Hit Rate
HIT_RATE=$(mariadb -sN -e "
    SELECT ROUND(
        (1 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
        )) * 100, 2
    )
")
log "Buffer Pool Hit Rate: ${HIT_RATE}%"

if [ $(echo "$HIT_RATE < 95" | bc) -eq 1 ]; then
    log "ALERTE: Buffer Pool Hit Rate < 95%"
fi

# 4. Slow Queries
SLOW_QUERIES=$(mariadb -sN -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Slow_queries'")
log "Slow Queries: $SLOW_QUERIES"

# 5. Espace disque
DISK_USAGE=$(df -h /var/lib/mysql | awk 'NR==2 {print $5}' | sed 's/%//')
log "Espace disque datadir: ${DISK_USAGE}%"

if [ $DISK_USAGE -gt 80 ]; then
    log "ALERTE: Disque > 80%"
    echo "Disque MariaDB à ${DISK_USAGE}%" | mail -s "MariaDB Disk Alert" dba@example.com
fi

log "=== FIN MONITORING ==="
```

---

## Outils de monitoring externes

### Prometheus + mysqld_exporter

**Installation mysqld_exporter** :

```bash
# Télécharger mysqld_exporter
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.0/mysqld_exporter-0.15.0.linux-amd64.tar.gz
tar xvfz mysqld_exporter-0.15.0.linux-amd64.tar.gz
sudo mv mysqld_exporter-0.15.0.linux-amd64/mysqld_exporter /usr/local/bin/

# Créer utilisateur MariaDB pour exporter
mariadb -e "
    CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'password';
    GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
    FLUSH PRIVILEGES;
"

# Créer fichier config
cat > /etc/.mysqld_exporter.cnf <<EOF
[client]
user=exporter
password=password
host=localhost
EOF

chmod 600 /etc/.mysqld_exporter.cnf

# Créer service systemd
cat > /etc/systemd/system/mysqld_exporter.service <<EOF
[Unit]
Description=MySQL Exporter
After=network.target

[Service]
Type=simple
User=mysql
ExecStart=/usr/local/bin/mysqld_exporter --config.my-cnf=/etc/.mysqld_exporter.cnf
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Démarrer
sudo systemctl daemon-reload
sudo systemctl enable mysqld_exporter
sudo systemctl start mysqld_exporter
```

**Prometheus configuration** :

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mariadb'
    static_configs:
      - targets: ['localhost:9104']
```

**Métriques Prometheus principales** :
- `mysql_global_status_threads_connected`
- `mysql_global_status_innodb_buffer_pool_read_requests`
- `mysql_global_status_slow_queries`
- `mysql_global_variables_max_connections`

### Grafana Dashboards

**Dashboards recommandés** :
- **MySQL Overview** : ID 7362
- **MySQL InnoDB Metrics** : ID 7365
- **MySQL Performance Schema** : ID 7366

### PMM (Percona Monitoring and Management)

**Installation PMM Server** (Docker) :

```bash
docker pull percona/pmm-server:2
docker create --volume pmm-data:/srv \
    --name pmm-server \
    --restart always \
    --publish 443:443 \
    percona/pmm-server:2
docker start pmm-server
```

**Installation PMM Client** :

```bash
wget https://downloads.percona.com/downloads/pmm2/2.40.0/binary/debian/bullseye/x86_64/pmm2-client_2.40.0-1.bullseye_amd64.deb
sudo dpkg -i pmm2-client_2.40.0-1.bullseye_amd64.deb

# Configurer
sudo pmm-admin config --server-insecure-tls --server-url=https://admin:admin@localhost:443

# Ajouter MariaDB
sudo pmm-admin add mysql --username=pmm --password=password --query-source=perfschema
```

---

## Alerting et notifications

### Script d'alerte simple

```bash
#!/bin/bash
# /usr/local/bin/mariadb-alerts.sh

ALERT_EMAIL="dba@example.com"

# Fonction d'alerte
alert() {
    local severity=$1
    local message=$2
    echo "[$severity] $message" | mail -s "[$severity] MariaDB Alert" "$ALERT_EMAIL"
}

# 1. Vérifier connexions
CONN_PCT=$(mariadb -sN -e "
    SELECT ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Threads_connected') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME = 'max_connections') * 100, 2
    )
")

if [ $(echo "$CONN_PCT > 90" | bc) -eq 1 ]; then
    alert "CRITICAL" "Connexions à ${CONN_PCT}%"
elif [ $(echo "$CONN_PCT > 75" | bc) -eq 1 ]; then
    alert "WARNING" "Connexions à ${CONN_PCT}%"
fi

# 2. Vérifier Buffer Pool Hit Rate
HIT_RATE=$(mariadb -sN -e "
    SELECT ROUND(
        (1 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
        )) * 100, 2
    )
")

if [ $(echo "$HIT_RATE < 90" | bc) -eq 1 ]; then
    alert "WARNING" "Buffer Pool Hit Rate faible: ${HIT_RATE}%"
fi

# 3. Vérifier réplication (si slave)
if mariadb -e "SHOW SLAVE STATUS\G" >/dev/null 2>&1; then
    LAG=$(mariadb -sN -e "SHOW SLAVE STATUS\G" | grep "Seconds_Behind_Master" | awk '{print $2}')

    if [ "$LAG" != "0" ] && [ "$LAG" != "NULL" ]; then
        if [ "$LAG" -gt 60 ]; then
            alert "CRITICAL" "Réplication lag: ${LAG} secondes"
        elif [ "$LAG" -gt 10 ]; then
            alert "WARNING" "Réplication lag: ${LAG} secondes"
        fi
    fi
fi

# 4. Vérifier deadlocks
DEADLOCKS=$(mariadb -sN -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_deadlocks'")
# Comparer avec valeur précédente stockée dans un fichier
# Si augmentation significative, alerter
```

### Configuration cron

```bash
# /etc/cron.d/mariadb-monitoring

# Monitoring toutes les 5 minutes
*/5 * * * * root /usr/local/bin/mariadb-monitoring.sh

# Alertes toutes les minutes
* * * * * root /usr/local/bin/mariadb-alerts.sh
```

---

## Bonnes pratiques de monitoring

### ✅ À FAIRE

1. **Monitorer en continu** : Pas seulement lors des problèmes
2. **Alertes intelligentes** : Seuils adaptés à votre charge
3. **Graphiques temporels** : Analyser tendances, pas juste valeurs ponctuelles
4. **Baseline** : Établir des valeurs normales pour comparaison
5. **Monitoring proactif** : Détecter avant l'impact utilisateur
6. **Documenter** : Valeurs normales, seuils d'alerte
7. **Tester les alertes** : Vérifier qu'elles fonctionnent
8. **Tableaux de bord** : Vue d'ensemble rapide (Grafana)
9. **Rétention historique** : Garder 30-90 jours de métriques
10. **Automatiser** : Scripts, cron, systemd timers

### ❌ À ÉVITER

1. **Alertes trop fréquentes** : Fatigue d'alerte, ignorées
2. **Pas de contexte** : Métriques isolées difficiles à interpréter
3. **Monitoring ponctuel** : Snapshots vs tendances
4. **Ignorer les warnings** : Attendre le critical
5. **Pas de documentation** : Que signifie cette métrique ?
6. **Oublier les logs** : Métriques + logs = complet
7. **Monitoring sans action** : Surveiller sans réagir
8. **Surcharger Performance Schema** : Impact performance

---

## Checklist de monitoring

### Quotidien (automatisé)

- [ ] Uptime > 0 (serveur actif)
- [ ] Threads_connected < 80% max_connections
- [ ] Buffer Pool Hit Rate > 95%
- [ ] Slow_queries (augmentation normale ?)
- [ ] Espace disque < 80%
- [ ] Réplication lag < 10 secondes
- [ ] Error log (erreurs récentes ?)

### Hebdomadaire

- [ ] Analyser slow query log (pt-query-digest)
- [ ] Vérifier fragmentation tables
- [ ] Revue des connexions refusées
- [ ] Tendances croissance données
- [ ] Deadlocks (investigation si > 10/jour)
- [ ] Thread Pool stalls

### Mensuel

- [ ] Revue complète dashboards
- [ ] Planification capacité (projection 6 mois)
- [ ] Optimisation tables fragmentées
- [ ] Revue configuration vs charge réelle
- [ ] Audit sécurité (connexions, privilèges)

---

## ✅ Points clés à retenir

- **Métriques critiques** : Buffer Pool Hit Rate (> 99%), Connexions (< 80%), Slow Queries, Espace disque
- **SHOW STATUS** : Statistiques en temps réel
- **Performance Schema** : Analyse détaillée requêtes et attentes
- **Thread Pool** : Monitoring via Threadpool_* status variables
- **Buffer Pool** : Métrique #1 pour performances InnoDB
- **Alerting** : Proactif (75% warning, 90% critical)
- **Outils externes** : Prometheus + Grafana, PMM pour visualisation avancée
- **Tendances** : Plus importantes que valeurs ponctuelles
- **Baseline** : Établir valeurs normales pour comparaison
- **Automatisation** : Cron, systemd timers, alertes email
- **Documentation** : Seuils, procédures, valeurs normales
- **Réplication** : Lag < 10 secondes, IO/SQL threads actifs

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Server Status Variables](https://mariadb.com/kb/en/server-status-variables/)
- [📖 Documentation officielle - Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [📖 Documentation officielle - Thread Pool](https://mariadb.com/kb/en/thread-pool-in-mariadb/)
- [🔧 Prometheus MySQL Exporter](https://github.com/prometheus/mysqld_exporter)
- [📊 Grafana MySQL Dashboards](https://grafana.com/grafana/dashboards/)
- [📊 Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management)
- [🔧 Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)

---

## ➡️ Section suivante

**[11.10 Thread Pool et gestion de la concurrence](./10-thread-pool.md)** : Configuration avancée du Thread Pool, optimisation pour haute concurrence, comparaison avec le modèle one-thread-per-connection.

---

**💡 Conseil final** : "In God we trust, all others must bring data" - W. Edwards Deming. Le monitoring transforme l'intuition en certitude. Mesurez tout, analysez les tendances, alertez intelligemment. Votre production vous remerciera ! 📊🚀

⏭️ [Thread Pool et gestion de la concurrence](/11-administration-configuration/10-thread-pool.md)

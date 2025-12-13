🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.7 Monitoring et Troubleshooting

> **Niveau** : Avancé  
> **Durée estimée** : 2.5-3 heures  
> **Prérequis** : 
> - Sections 13.1 à 13.6 (Configuration complète de réplication)
> - Expérience en monitoring de systèmes distribués
> - Connaissance des métriques systèmes (CPU, I/O, réseau)
> - Familiarité avec les outils de monitoring (Prometheus, Grafana)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Identifier les métriques critiques de santé de la réplication
- Mettre en place un monitoring complet et proactif
- Diagnostiquer rapidement les problèmes de réplication
- Utiliser les outils de troubleshooting avancés
- Implémenter un système d'alerting efficace
- Résoudre les erreurs courantes de réplication
- Analyser et réduire le lag de réplication
- Établir des SLA de réplication réalistes

---

## Introduction

Le **monitoring de la réplication** est un aspect **critique** de toute infrastructure MariaDB en production. Une réplication qui semble fonctionner peut masquer des problèmes latents qui se manifesteront lors d'un failover ou sous forte charge.

### Pourquoi le monitoring est critique

**Scénarios catastrophes sans monitoring** :

```
Scénario 1 : Le lag silencieux
─────────────────────────────────────────
T0: Réplication fonctionne (lag: 0s)
T1: Grosse transaction (ALTER TABLE, 2h)
T2: Lag = 2 heures (non détecté)
T3: Primary crash !
T4: Promotion Replica avec 2h de données manquantes
    → Perte de données massive ❌

Avec monitoring:
T1: Alerte "Lag > 60s"
T2: Investigation et optimisation
T3: Lag réduit avant crash
    → Perte minimale ou nulle ✓
```

```
Scénario 2 : L'erreur ignorée
─────────────────────────────────────────
T0: Erreur SQL réplication (duplicate key)
    SQL Thread: Stopped
    IO Thread: Continue (accumule relay logs)
T1-T30: 30 jours sans monitoring
T31: Disk full (relay logs)
T32: Tout le serveur tombe
    → Incident majeur ❌

Avec monitoring:
T0: Alerte immédiate "SQL Thread stopped"
T1: Résolution en minutes
    → Incident évité ✓
```

### Principes de monitoring

**Règle 1 : Monitoring proactif > Réactif**

```
Mauvais : Surveiller seulement "Replica UP/DOWN"
         → Détection trop tardive

Bon : Surveiller tendances et seuils
      → Détection précoce et prévention
```

**Règle 2 : Monitoring multi-niveaux**

```
Niveau 1: Santé globale (UP/DOWN)
Niveau 2: Performance (lag, throughput)
Niveau 3: Ressources (CPU, I/O, réseau)
Niveau 4: Tendances (croissance lag, erreurs)
```

**Règle 3 : Alerting intelligent**

```
Mauvais : Alerte pour chaque micro-spike
         → Alert fatigue

Bon : Alerting basé sur seuils et durée
      → Signal/bruit optimisé
```

---

## Vue d'Ensemble des Métriques

### Classification des métriques

```
┌─────────────────────────────────────────────────────────────┐
│                  MÉTRIQUE DE RÉPLICATION                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CRITICITÉ 1 : Santé (Health)                         │   │
│  │ ✓ Must-monitor, alerting immédiat                    │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ - Slave_IO_Running (Yes/No)                          │   │
│  │ - Slave_SQL_Running (Yes/No)                         │   │
│  │ - Last_IO_Error                                      │   │
│  │ - Last_SQL_Error                                     │   │
│  │ - Seconds_Behind_Master                              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CRITICITÉ 2 : Performance                            │   │
│  │ ✓ Monitor, alerting sur seuils                       │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ - Replication lag (secondes/bytes)                   │   │
│  │ - Events/second throughput                           │   │
│  │ - Relay log space                                    │   │
│  │ - Binlog position delta                              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CRITICITÉ 3 : Ressources                             │   │
│  │ ✓ Monitoring tendances                               │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ - CPU utilization (IO/SQL threads)                   │   │
│  │ - Disk I/O (relay log, binlog)                       │   │
│  │ - Network bandwidth                                  │   │
│  │ - Memory (buffer pool, caches)                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CRITICITÉ 4 : Diagnostic                             │   │
│  │ ✓ Pour troubleshooting avancé                        │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ - GTID positions                                     │   │
│  │ - Semi-sync status                                   │   │
│  │ - Parallel threads efficiency                        │   │
│  │ - Slave skip counter                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Métriques essentielles

#### 1. État des threads

```sql
-- Métrique la plus critique
SELECT 
  Slave_IO_Running,    -- Doit être "Yes"
  Slave_SQL_Running    -- Doit être "Yes"
FROM information_schema.SLAVE_STATUS;

-- Interprétation:
-- Yes/Yes : ✓ OK
-- Yes/No  : ⚠️ Erreur SQL (urgent)
-- No/Yes  : ⚠️ Problème connexion Primary (urgent)
-- No/No   : 🚨 Réplication arrêtée (critique)
```

**Causes typiques** :

| État | Cause probable | Urgence |
|------|----------------|---------|
| IO=No, SQL=Yes | Problème réseau, Primary down | 🔴 Critique |
| IO=Yes, SQL=No | Erreur SQL (duplicate, missing row) | 🔴 Critique |
| IO=No, SQL=No | STOP SLAVE manuel ou config erreur | 🟠 Élevée |

#### 2. Lag de réplication

```sql
-- Lag en secondes
SELECT 
  Seconds_Behind_Master
FROM information_schema.SLAVE_STATUS;

-- Interprétation:
-- 0      : ✓ Parfaitement synchronisé
-- 1-10   : ✓ Acceptable (normal spike)
-- 10-60  : ⚠️ À surveiller
-- 60-300 : 🟠 Lag élevé (investiguer)
-- >300   : 🔴 Lag critique (action immédiate)
-- NULL   : 🚨 IO Thread non connecté
```

💡 **Important** : `Seconds_Behind_Master` peut être **trompeur** avec:
- Parallélisation (lag apparent vs réel)
- Longues transactions (pic temporaire)
- Pas d'écritures sur Primary (lag=0 mais non significatif)

**Mesure alternative : pt-heartbeat**

```bash
# Sur Primary : Insérer heartbeat
pt-heartbeat \
  --user=root \
  --password=xxx \
  --database=heartbeat \
  --table=heartbeat \
  --update \
  --interval=1

# Sur Replica : Mesurer lag réel
pt-heartbeat \
  --user=root \
  --password=xxx \
  --database=heartbeat \
  --table=heartbeat \
  --monitor \
  --frames=1m,5m,15m

# Output:
#    0s [  0.00s,  0.00s,  0.00s ]  ← Lag instantané, 1min avg, 5min avg, 15min avg
```

#### 3. Positions et progression

```sql
-- Positions de réplication
SELECT 
  Master_Log_File,         -- Fichier binlog Primary
  Read_Master_Log_Pos,     -- Position lue (IO Thread)
  Relay_Master_Log_File,   -- Fichier correspondant (SQL Thread)
  Exec_Master_Log_Pos      -- Position exécutée (SQL Thread)
FROM information_schema.SLAVE_STATUS;

-- Calcul du lag en bytes
SELECT 
  Read_Master_Log_Pos - Exec_Master_Log_Pos AS lag_bytes
FROM information_schema.SLAVE_STATUS;

-- Avec GTID
SELECT 
  Gtid_IO_Pos,    -- GTID reçu
  Gtid_Slave_Pos  -- GTID appliqué
FROM information_schema.SLAVE_STATUS;
```

#### 4. Erreurs

```sql
-- Dernières erreurs IO et SQL
SELECT 
  Last_IO_Errno,
  Last_IO_Error,
  Last_SQL_Errno,
  Last_SQL_Error
FROM information_schema.SLAVE_STATUS;

-- Erreurs courantes:
-- 1062: Duplicate entry (INSERT sur clé existante)
-- 1032: Can't find record (UPDATE/DELETE sur ligne absente)
-- 1236: Cannot replicate because ... purged binary log
-- 2003: Can't connect to MySQL server (réseau)
```

#### 5. Relay log

```sql
-- Espace disque utilisé par relay logs
SELECT 
  Relay_Log_Space / 1024 / 1024 AS relay_log_mb
FROM information_schema.SLAVE_STATUS;

-- Alerte si > 10GB (accumulation anormale)
```

---

## Outils de Monitoring

### 1. Commandes SQL natives

**SHOW SLAVE STATUS** (détails en 13.7.1)

```sql
-- Vue complète
SHOW SLAVE STATUS\G

-- Vue condensée
SELECT 
  CONCAT(Slave_IO_Running, '/', Slave_SQL_Running) AS 'IO/SQL',
  Seconds_Behind_Master AS Lag,
  Master_Log_File,
  Read_Master_Log_Pos
FROM information_schema.SLAVE_STATUS;
```

**SHOW PROCESSLIST**

```sql
-- Identifier threads de réplication
SHOW PROCESSLIST;

-- Output:
-- +-----+-------------+-------------------+------+---------+------+----------------------------------+
-- | Id  | User        | Host              | db   | Command | Time | State                            |
-- +-----+-------------+-------------------+------+---------+------+----------------------------------+
-- | 10  | system user |                   | NULL | Connect | 3456 | Waiting for master to send event |  ← IO Thread
-- | 11  | system user |                   | NULL | Connect | 3456 | Slave has read all relay log     |  ← SQL Thread
-- +-----+-------------+-------------------+------+---------+------+----------------------------------+
```

**SHOW RELAYLOG EVENTS**

```sql
-- Inspecter le relay log
SHOW RELAYLOG EVENTS IN 'relay-bin.000003' LIMIT 10;

-- Utile pour debug erreurs SQL
```

### 2. pt-heartbeat (Percona Toolkit)

**Installation** :

```bash
# Debian/Ubuntu
apt-get install percona-toolkit

# RHEL/CentOS
yum install percona-toolkit
```

**Configuration** :

```sql
-- Sur Primary : Créer table heartbeat
CREATE DATABASE IF NOT EXISTS heartbeat;

CREATE TABLE heartbeat.heartbeat (
  ts TIMESTAMP(6) NOT NULL,
  server_id INT UNSIGNED NOT NULL PRIMARY KEY,
  file VARCHAR(255),
  position BIGINT UNSIGNED,
  relay_master_log_file VARCHAR(255),
  exec_master_log_pos BIGINT UNSIGNED
);
```

**Utilisation** :

```bash
# Sur Primary : Daemon heartbeat
pt-heartbeat \
  --user=root \
  --password=SecureP@ss \
  --database=heartbeat \
  --table=heartbeat \
  --update \
  --interval=1 \
  --daemonize

# Sur Replica : Monitoring
pt-heartbeat \
  --user=root \
  --password=SecureP@ss \
  --database=heartbeat \
  --table=heartbeat \
  --monitor \
  --print-master-server-id

# Output continu:
#    0.50s [  0.48s,  0.52s,  0.55s ]
#    0.51s [  0.49s,  0.53s,  0.56s ]
```

**Avantages** :
- ✅ Mesure précise (microseconde)
- ✅ Moyennes glissantes (1m, 5m, 15m)
- ✅ Fonctionne même sans écritures applicatives

### 3. Orchestrator

**Orchestrator** : Gestionnaire de topologie avec monitoring intégré

**Installation** :

```bash
# Download binary
wget https://github.com/openark/orchestrator/releases/download/v3.2.6/orchestrator-3.2.6-1.x86_64.rpm
rpm -i orchestrator-3.2.6-1.x86_64.rpm

# Configuration
vim /etc/orchestrator.conf.json
```

**Features monitoring** :
- Découverte automatique de topologie
- Monitoring lag en temps réel
- Détection anomalies
- Failover automatique (optionnel)
- Dashboard web intégré

**Accès** :

```bash
# Web UI
http://orchestrator-host:3000

# CLI
orchestrator-client -c topology -i primary.example.com:3306
```

### 4. Prometheus + Grafana

**mysqld_exporter** :

```bash
# Installation
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.linux-amd64.tar.gz
tar xvf mysqld_exporter-0.14.0.linux-amd64.tar.gz
cd mysqld_exporter-0.14.0.linux-amd64

# Créer utilisateur monitoring
mysql -e "
  CREATE USER 'exporter'@'localhost' 
    IDENTIFIED BY 'ExporterP@ss' 
    WITH MAX_USER_CONNECTIONS 3;
  GRANT PROCESS, REPLICATION CLIENT, SELECT 
    ON *.* TO 'exporter'@'localhost';
"

# Configuration
cat > .my.cnf <<EOF
[client]
user=exporter
password=ExporterP@ss
EOF

# Lancer exporter
./mysqld_exporter --config.my-cnf=.my.cnf &
```

**Configuration Prometheus** :

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mariadb'
    static_configs:
      - targets: 
        - 'replica1:9104'
        - 'replica2:9104'
        labels:
          env: 'production'
```

**Métriques exportées** :

```
# Lag
mysql_slave_status_seconds_behind_master

# Threads
mysql_slave_status_slave_io_running (1=Yes, 0=No)
mysql_slave_status_slave_sql_running (1=Yes, 0=No)

# Positions
mysql_slave_status_read_master_log_pos
mysql_slave_status_exec_master_log_pos

# Relay log
mysql_slave_status_relay_log_space
```

**Dashboard Grafana** :

```json
{
  "panels": [
    {
      "title": "Replication Status",
      "targets": [
        {
          "expr": "mysql_slave_status_slave_io_running{job='mariadb'}",
          "legendFormat": "{{instance}} - IO"
        },
        {
          "expr": "mysql_slave_status_slave_sql_running{job='mariadb'}",
          "legendFormat": "{{instance}} - SQL"
        }
      ],
      "type": "graph"
    },
    {
      "title": "Replication Lag",
      "targets": [
        {
          "expr": "mysql_slave_status_seconds_behind_master{job='mariadb'}",
          "legendFormat": "{{instance}}"
        }
      ],
      "type": "graph",
      "alert": {
        "conditions": [
          {
            "query": "avg() OF query(A, 5m, now) IS ABOVE 60"
          }
        ]
      }
    }
  ]
}
```

### 5. Scripts personnalisés

**Script de santé global** :

```bash
#!/bin/bash
# replication_health.sh

MYSQL="mysql -N -e"

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo "=== MariaDB Replication Health Check ==="
echo "Time: $(date)"
echo ""

# 1. Vérifier threads
IO_RUNNING=$($MYSQL "SELECT Slave_IO_Running FROM information_schema.SLAVE_STATUS")
SQL_RUNNING=$($MYSQL "SELECT Slave_SQL_Running FROM information_schema.SLAVE_STATUS")

echo -n "IO Thread: "
if [ "$IO_RUNNING" = "Yes" ]; then
  echo -e "${GREEN}✓ Running${NC}"
else
  echo -e "${RED}✗ Stopped${NC}"
  IO_ERROR=$($MYSQL "SELECT Last_IO_Error FROM information_schema.SLAVE_STATUS")
  echo "  Error: $IO_ERROR"
fi

echo -n "SQL Thread: "
if [ "$SQL_RUNNING" = "Yes" ]; then
  echo -e "${GREEN}✓ Running${NC}"
else
  echo -e "${RED}✗ Stopped${NC}"
  SQL_ERROR=$($MYSQL "SELECT Last_SQL_Error FROM information_schema.SLAVE_STATUS")
  echo "  Error: $SQL_ERROR"
fi

# 2. Vérifier lag
LAG=$($MYSQL "SELECT IFNULL(Seconds_Behind_Master, -1) FROM information_schema.SLAVE_STATUS")

echo -n "Replication Lag: "
if [ "$LAG" = "-1" ]; then
  echo -e "${RED}NULL (IO not connected)${NC}"
elif [ "$LAG" -eq 0 ]; then
  echo -e "${GREEN}0 seconds (synchronized)${NC}"
elif [ "$LAG" -lt 10 ]; then
  echo -e "${GREEN}$LAG seconds (normal)${NC}"
elif [ "$LAG" -lt 60 ]; then
  echo -e "${YELLOW}$LAG seconds (elevated)${NC}"
else
  echo -e "${RED}$LAG seconds (critical!)${NC}"
fi

# 3. Vérifier positions
READ_POS=$($MYSQL "SELECT Read_Master_Log_Pos FROM information_schema.SLAVE_STATUS")
EXEC_POS=$($MYSQL "SELECT Exec_Master_Log_Pos FROM information_schema.SLAVE_STATUS")
LAG_BYTES=$((READ_POS - EXEC_POS))

echo "Position lag: $LAG_BYTES bytes"

# 4. Vérifier relay log space
RELAY_SPACE=$($MYSQL "SELECT Relay_Log_Space FROM information_schema.SLAVE_STATUS")
RELAY_MB=$((RELAY_SPACE / 1024 / 1024))

echo -n "Relay log space: ${RELAY_MB}MB "
if [ $RELAY_MB -gt 10240 ]; then  # > 10GB
  echo -e "${RED}(warning: high accumulation)${NC}"
else
  echo -e "${GREEN}(normal)${NC}"
fi

# 5. Vérifier GTID (si activé)
USING_GTID=$($MYSQL "SELECT Using_Gtid FROM information_schema.SLAVE_STATUS")
if [ -n "$USING_GTID" ] && [ "$USING_GTID" != "No" ]; then
  echo "Using GTID: $USING_GTID"
  GTID_POS=$($MYSQL "SELECT Gtid_Slave_Pos FROM information_schema.SLAVE_STATUS")
  echo "GTID Position: $GTID_POS"
fi

# 6. Exit code
if [ "$IO_RUNNING" != "Yes" ] || [ "$SQL_RUNNING" != "Yes" ]; then
  echo ""
  echo -e "${RED}CRITICAL: Replication not running${NC}"
  exit 2
elif [ "$LAG" -gt 60 ]; then
  echo ""
  echo -e "${YELLOW}WARNING: High replication lag${NC}"
  exit 1
else
  echo ""
  echo -e "${GREEN}OK: Replication healthy${NC}"
  exit 0
fi
```

**Utilisation avec cron** :

```bash
# Exécuter toutes les 5 minutes
*/5 * * * * /usr/local/bin/replication_health.sh >> /var/log/replication_health.log 2>&1
```

---

## Stratégies de Troubleshooting

### Méthodologie systématique

**1. Identifier le symptôme**

```
Questions à poser:
├─ Threads IO/SQL en erreur ?
├─ Lag élevé ?
├─ Erreur spécifique ?
├─ Problème récent ou récurrent ?
└─ Impact sur l'application ?
```

**2. Collecter les données**

```sql
-- Snapshot complet état réplication
SELECT 
  NOW() AS snapshot_time,
  Slave_IO_Running,
  Slave_SQL_Running,
  Seconds_Behind_Master,
  Last_IO_Errno,
  Last_IO_Error,
  Last_SQL_Errno,
  Last_SQL_Error,
  Master_Host,
  Master_Log_File,
  Read_Master_Log_Pos,
  Relay_Master_Log_File,
  Exec_Master_Log_Pos,
  Relay_Log_Space,
  Using_Gtid,
  Gtid_IO_Pos,
  Gtid_Slave_Pos
FROM information_schema.SLAVE_STATUS\G
```

**3. Analyser les logs**

```bash
# Error log MariaDB
tail -100 /var/log/mysql/error.log

# Filtrer erreurs réplication
grep -i "slave\|replica\|replication" /var/log/mysql/error.log | tail -50

# Analyser slow query log (cause de lag)
pt-query-digest /var/log/mysql/slow.log \
  --since '1h ago' \
  --order-by Query_time:sum \
  --limit 10
```

**4. Tester la connectivité**

```bash
# Réseau
ping primary.example.com
telnet primary.example.com 3306

# Authentification
mysql -h primary.example.com -u repl_user -p

# Firewall
iptables -L -n | grep 3306
```

**5. Vérifier les ressources**

```bash
# CPU
top -bn1 | head -20

# Mémoire
free -h

# Disque I/O
iostat -x 1 5

# Réseau
iftop -i eth0
```

### Arbre de décision troubleshooting

```
                    PROBLÈME RÉPLICATION
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
      IO Thread       SQL Thread        Lag élevé
       stopped         stopped          (threads OK)
            │               │               │
            ▼               ▼               ▼
    ┌──────────────┐ ┌─────────────┐ ┌──────────────┐
    │ Causes:      │ │ Causes:     │ │ Causes:      │
    │ - Réseau     │ │ - Duplicate │ │ - Grosse trx │
    │ - Primary DN │ │ - Missing   │ │ - Slow query │
    │ - Auth       │ │   row       │ │ - Disque I/O │
    │ - Binlog     │ │ - DDL error │ │ - CPU        │
    │   purgé      │ │             │ │ - Parallel   │
    └──────────────┘ └─────────────┘ └──────────────┘
            │               │               │
            ▼               ▼               ▼
       [Section          [Section        [Section
        13.7.2]          13.7.3]         13.7.4]
```

---

## Erreurs Courantes et Résolutions

### 1. Erreur 1062 : Duplicate entry

```sql
SHOW SLAVE STATUS\G
-- Last_SQL_Errno: 1062
-- Last_SQL_Error: Error 'Duplicate entry '123' for key 'PRIMARY'' on query
```

**Causes** :
- Écriture manuelle sur Replica
- Replay d'événements déjà appliqués
- Bug application (double INSERT)

**Diagnostic** :

```sql
-- Identifier la table et la valeur
-- Depuis Last_SQL_Error: table `mydb`.`users`, key=123

-- Vérifier sur Replica
SELECT * FROM mydb.users WHERE id = 123;

-- Comparer avec Primary
-- Sur Primary:
SELECT * FROM mydb.users WHERE id = 123;
```

**Solutions** :

**Option A : Skip événement (si données identiques)**

```sql
STOP SLAVE SQL_THREAD;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE SQL_THREAD;

-- Vérifier
SHOW SLAVE STATUS\G
-- Slave_SQL_Running: Yes
```

**Option B : Supprimer doublon (si corruption locale)**

```sql
STOP SLAVE SQL_THREAD;
DELETE FROM mydb.users WHERE id = 123;  -- Si données incorrectes
START SLAVE SQL_THREAD;
```

**Option C : Resynchroniser (si corruption massive)**

```bash
# Dump Primary
mysqldump -u root -p mydb > mydb_fix.sql

# Restaurer sur Replica
mariadb -u root -p mydb < mydb_fix.sql

# Reprendre réplication
mysql -e "START SLAVE"
```

### 2. Erreur 1032 : Can't find record

```sql
-- Last_SQL_Errno: 1032
-- Last_SQL_Error: Could not execute Update_rows event on table mydb.users; 
--                 Can't find record in 'users'
```

**Causes** :
- Ligne supprimée manuellement sur Replica
- Corruption de données
- Réplication partielle (filters)

**Diagnostic** :

```sql
-- Identifier la ligne manquante
-- Extraire du binlog Primary
mysqlbinlog mariadb-bin.000042 \
  --start-position=<Read_Master_Log_Pos> \
  --stop-position=<Read_Master_Log_Pos+1000>

-- Chercher UPDATE/DELETE concerné
```

**Solutions** :

**Option A : Skip (si ligne non critique)**

```sql
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE SQL_THREAD;
```

**Option B : Insérer ligne manquante**

```sql
STOP SLAVE SQL_THREAD;

-- Récupérer la ligne depuis Primary
-- Sur Primary:
SELECT * FROM mydb.users WHERE id = 456;

-- Sur Replica: Insérer
INSERT INTO mydb.users VALUES (...);  -- Copier données Primary

START SLAVE SQL_THREAD;
```

### 3. Erreur 1236 : Binary log purged

```sql
-- Last_IO_Errno: 1236
-- Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 
--                'Could not find first log file name in binary log index file'
```

**Causes** :
- Primary a purgé binlog nécessaire
- Replica arrêté trop longtemps
- Purge automatique agressive

**Solution unique : Resynchroniser**

```bash
# 1. Dump Primary
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  > full_dump.sql

# 2. Restaurer Replica
mariadb -u root -p < full_dump.sql

# 3. Reconfigurer réplication
# Extraire position du dump
grep "CHANGE MASTER" full_dump.sql

# Avec GTID (plus simple)
mysql -e "
  STOP SLAVE;
  CHANGE MASTER TO MASTER_USE_GTID=slave_pos;
  START SLAVE;
"
```

**Prévention** :

```ini
[mysqld]
# Conserver binlogs plus longtemps
expire_logs_days = 14  # Au lieu de 7

# Vérifier que Replicas sont à jour avant purge
# (Aucun paramètre auto, script manuel requis)
```

### 4. Erreur 2003/2013 : Connexion réseau

```sql
-- Last_IO_Errno: 2003
-- Last_IO_Error: error connecting to master 'repl_user@primary:3306' - 
--                retry-time: 60  retries: 1
```

**Causes** :
- Primary down
- Problème réseau
- Firewall
- DNS

**Diagnostic** :

```bash
# 1. Ping
ping primary.example.com

# 2. Telnet
telnet primary.example.com 3306
# Doit afficher bannière MariaDB

# 3. DNS
nslookup primary.example.com
dig primary.example.com

# 4. Firewall Replica
iptables -L -n | grep 3306

# 5. Firewall Primary (depuis Primary)
iptables -L -n | grep 3306

# 6. Test connexion MySQL
mysql -h primary.example.com -u repl_user -p
```

**Solutions** :

```bash
# Si Primary down: Promouvoir Replica (voir 13.8)

# Si firewall:
iptables -A OUTPUT -p tcp --dport 3306 -j ACCEPT  # Replica
iptables -A INPUT -p tcp --dport 3306 -j ACCEPT   # Primary

# Si DNS:
# Utiliser IP directement
mysql -e "
  STOP SLAVE;
  CHANGE MASTER TO MASTER_HOST='192.168.1.100';
  START SLAVE;
"
```

---

## Analyse du Lag

### Causes du lag

**1. Grosse transaction unique**

```sql
-- Sur Primary: ALTER TABLE de 1TB
ALTER TABLE huge_table ADD COLUMN new_col INT;
-- Durée: 2 heures

-- Sur Replica: Même ALTER
-- Lag = 2 heures (temporaire, normal)
```

**Solution** :
- ✅ Attendre (temporaire)
- ✅ Utiliser Online DDL (13.10 - Optimistic ALTER)
- ✅ Planifier maintenance

**2. Throughput élevé**

```sql
-- Primary: 10,000 TPS (transactions/sec)
-- Replica: SQL Thread peut traiter 5,000 TPS
-- Lag = Cumulatif progressif
```

**Solution** :

```sql
-- Paralléliser réplication
SET GLOBAL slave_parallel_threads = 8;
SET GLOBAL slave_parallel_mode = 'optimistic';
```

**3. Slow queries sur Replica**

```sql
-- Requête rapide sur Primary (index présent)
UPDATE users SET status='active' WHERE email='alice@example.com';
-- 10ms

-- Même requête lente sur Replica (index absent/corrompu)
-- 5000ms

-- Lag = Accumulation de slow queries
```

**Diagnostic** :

```sql
-- Activer slow query log sur Replica
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;

-- Analyser
pt-query-digest /var/log/mysql/slow.log
```

**Solution** :

```sql
-- Reconstruire indexes manquants/corrompus
ANALYZE TABLE users;
OPTIMIZE TABLE users;
```

**4. Ressources insuffisantes**

```
CPU saturé:
- SQL Thread attend CPU
- Lag augmente progressivement

I/O disque saturé:
- Écriture relay log lente
- Application événements lente

Réseau saturé:
- IO Thread lent à récupérer binlog
```

**Diagnostic** :

```bash
# CPU
top -bn1 | grep -E "mysqld|%CPU"

# I/O
iostat -x 1 5
# Regarder %util et await

# Réseau
iftop -i eth0
```

**Solution** :
- Upgrade hardware
- Optimiser configuration
- Réduire charge Replica (déplacer lectures)

---

## Système d'Alerting

### Seuils recommandés

```yaml
alerts:
  critical:
    - name: "Replication Stopped"
      condition: "IO_Running = No OR SQL_Running = No"
      action: "Page on-call DBA immediately"
      
    - name: "Replication Lag Critical"
      condition: "Seconds_Behind_Master > 300"  # 5 minutes
      duration: "2m"  # Persistant 2 minutes
      action: "Page on-call DBA"
      
    - name: "Binlog Purged"
      condition: "Last_IO_Errno = 1236"
      action: "Page on-call DBA + trigger resync"
      
  warning:
    - name: "Replication Lag Warning"
      condition: "Seconds_Behind_Master > 60"  # 1 minute
      duration: "5m"
      action: "Alert DevOps team"
      
    - name: "Relay Log Accumulation"
      condition: "Relay_Log_Space > 10GB"
      action: "Investigate SQL Thread performance"
      
    - name: "IO Thread Reconnecting"
      condition: "Slave_IO_State LIKE '%reconnecting%'"
      duration: "1m"
      action: "Check network/Primary health"
      
  info:
    - name: "Replication Lag Info"
      condition: "Seconds_Behind_Master > 10"
      duration: "2m"
      action: "Log for trending"
```

### Configuration Prometheus Alertmanager

```yaml
# alertmanager.yml
groups:
  - name: mariadb_replication
    interval: 30s
    rules:
      - alert: ReplicationStopped
        expr: |
          mysql_slave_status_slave_io_running == 0 
          OR 
          mysql_slave_status_slave_sql_running == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Replication stopped on {{ $labels.instance }}"
          description: "IO: {{ $value }}, check SHOW SLAVE STATUS"
          
      - alert: ReplicationLagCritical
        expr: mysql_slave_status_seconds_behind_master > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Replication lag > 5min on {{ $labels.instance }}"
          description: "Current lag: {{ $value }}s"
          
      - alert: ReplicationLagWarning
        expr: mysql_slave_status_seconds_behind_master > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag > 1min on {{ $labels.instance }}"
```

### Script d'alerte personnalisé

```bash
#!/bin/bash
# replication_alert.sh

WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
LAG_THRESHOLD_WARN=60
LAG_THRESHOLD_CRIT=300

LAG=$(mysql -N -e "SELECT IFNULL(Seconds_Behind_Master, -1) FROM information_schema.SLAVE_STATUS")
IO_RUNNING=$(mysql -N -e "SELECT Slave_IO_Running FROM information_schema.SLAVE_STATUS")
SQL_RUNNING=$(mysql -N -e "SELECT Slave_SQL_Running FROM information_schema.SLAVE_STATUS")

# Fonction envoi Slack
send_alert() {
  local severity=$1
  local message=$2
  
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"[$severity] $message\"}" \
    $WEBHOOK_URL
}

# Vérifier threads
if [ "$IO_RUNNING" != "Yes" ] || [ "$SQL_RUNNING" != "Yes" ]; then
  ERROR=$(mysql -N -e "SELECT CONCAT(Last_IO_Error, ' | ', Last_SQL_Error) FROM information_schema.SLAVE_STATUS")
  send_alert "CRITICAL" "Replication stopped on $(hostname): $ERROR"
  exit 2
fi

# Vérifier lag
if [ "$LAG" -ge "$LAG_THRESHOLD_CRIT" ]; then
  send_alert "CRITICAL" "Replication lag ${LAG}s on $(hostname) (threshold: ${LAG_THRESHOLD_CRIT}s)"
  exit 2
elif [ "$LAG" -ge "$LAG_THRESHOLD_WARN" ]; then
  send_alert "WARNING" "Replication lag ${LAG}s on $(hostname) (threshold: ${LAG_THRESHOLD_WARN}s)"
  exit 1
fi

exit 0
```

---

## Sous-sections du Chapitre 13.7

Ce chapitre se décompose en plusieurs sous-sections détaillées :

### 📖 13.7.1 SHOW SLAVE STATUS / SHOW REPLICA STATUS

Analyse ligne par ligne de la sortie de `SHOW SLAVE STATUS`, interprétation de chaque champ, différences SHOW SLAVE vs SHOW REPLICA, utilisation pour diagnostic.

### 📖 13.7.2 Seconds_Behind_Master et lag

Calcul du lag, limitations de `Seconds_Behind_Master`, mesures alternatives (pt-heartbeat), lag vs throughput, analyse des tendances.

### 📖 13.7.3 Erreurs courantes et résolution

Catalogue complet des erreurs de réplication (1062, 1032, 1236, 2003, etc.), diagnostic approfondi, procédures de résolution step-by-step, prévention.

---

## ✅ Points clés à retenir

1. **Monitoring proactif** : Anticiper les problèmes avant qu'ils ne deviennent critiques

2. **Métriques essentielles** : IO/SQL Running, Seconds_Behind_Master, Last_Error minimum absolu

3. **pt-heartbeat** : Mesure de lag plus fiable que Seconds_Behind_Master

4. **Alerting intelligent** : Seuils + durée pour éviter alert fatigue

5. **Troubleshooting systématique** : Identifier → Collecter → Analyser → Résoudre

6. **Erreurs courantes** : 1062 (duplicate), 1032 (missing row), 1236 (purged binlog)

7. **Lag causes** : Grosse transaction, throughput élevé, slow queries, ressources

8. **Outils multiples** : SQL natif, pt-heartbeat, Orchestrator, Prometheus/Grafana

9. **Documentation** : Runbooks pour chaque type d'erreur

10. **Tests réguliers** : Vérifier monitoring et alerting fonctionnent

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Replication Monitoring](https://mariadb.com/kb/en/replication-monitoring/)
- [📖 SHOW SLAVE STATUS](https://mariadb.com/kb/en/show-slave-status/)
- [📖 Replication Troubleshooting](https://mariadb.com/kb/en/replication-troubleshooting/)

### Outils

- **Percona Toolkit** : pt-heartbeat, pt-table-checksum
- **Orchestrator** : github.com/openark/orchestrator
- **mysqld_exporter** : github.com/prometheus/mysqld_exporter

### Articles techniques

- [🔗 Monitoring MySQL Replication (Percona)](https://www.percona.com/blog/monitoring-mysql-replication/)
- [🔗 Understanding Replication Lag](https://www.percona.com/blog/understanding-mysql-replication-lag/)

---

## ➡️ Sections suivantes

### 13.7.1 SHOW SLAVE STATUS / SHOW REPLICA STATUS

Analyse détaillée de chaque champ de `SHOW SLAVE STATUS`, interprétation, utilisation pour diagnostic, différences entre versions MariaDB.

### 13.7.2 Seconds_Behind_Master et lag

Fonctionnement du calcul de lag, limitations, mesures alternatives, analyse de tendances, corrélation avec performance applicative.

### 13.7.3 Erreurs courantes et résolution

Catalogue complet des erreurs (codes 1062, 1032, 1236, 2003, etc.), procédures de diagnostic et résolution, prévention.

Le monitoring et troubleshooting sont la clé d'une réplication stable et fiable en production !

---


⏭️ [SHOW SLAVE STATUS / SHOW REPLICA STATUS](/13-replication/07.1-show-slave-status.md)

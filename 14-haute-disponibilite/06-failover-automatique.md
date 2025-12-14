🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.6 Solutions de Failover Automatique

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Sections 14.1-14.3 (HA, Galera, Quorum), expérience opérationnelle production

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les différents types de failover et leurs implications
- **Concevoir** des stratégies de détection de pannes fiables
- **Implémenter** des solutions de failover automatique pour réplication et Galera
- **Configurer** des outils spécialisés (Orchestrator, MHA, mariadbmon)
- **Exploiter** les nouveautés MariaDB 11.8 : Transaction Replay et Connection Migration
- **Éviter** les scénarios catastrophiques (double-master, data loss)
- **Tester** et valider vos procédures de failover
- **Opérer** et monitorer le failover en production

---

## Introduction

Le **failover** est le processus de bascule automatique d'un système défaillant vers un système de secours. Dans le contexte MariaDB, il s'agit de promouvoir un replica en master (ou de router vers un autre nœud Galera) lorsqu'une défaillance est détectée.

**Dilemme fondamental** :
```
┌──────────────────────────────────────────────────┐
│  Failover Manuel                                 │
│  ✅ Contrôle total                               │
│  ✅ Évite faux positifs                          │
│  ✅ Décision humaine informée                    │
│  ❌ RTO élevé (15-60 minutes)                    │
│  ❌ Nécessite astreinte 24/7                     │
│  ❌ Erreurs humaines possibles                   │
└──────────────────────────────────────────────────┘
                      VS
┌──────────────────────────────────────────────────┐
│  Failover Automatique                            │
│  ✅ RTO minimal (30-90 secondes)                 │
│  ✅ Disponibilité 24/7 sans humain               │
│  ✅ Procédure reproductible                      │
│  ❌ Risque de faux positifs                      │
│  ❌ Décisions sans contexte                      │
│  ❌ Complexité opérationnelle                    │
└──────────────────────────────────────────────────┘
```

> ⚠️ **Principe de Prudence** : "Le failover automatique doit être configuré de manière conservatrice. Mieux vaut une alerte à 3h du matin qu'un split-brain silencieux."

### 🆕 Nouveautés MariaDB 11.8 pour le Failover

**1. Transaction Replay (Rejouabilité Automatique)**
```sql
-- Transactions automatiquement rejouées après failover
SET GLOBAL transaction_replay = ON;
SET GLOBAL transaction_replay_attempts = 3;
```
→ Réduction significative du RTO et transparence applicative

**2. Connection Migration (Migration de Sessions)**
```sql
-- Préservation des sessions après failover
SET GLOBAL connection_migration = ON;
SET GLOBAL connection_migration_preserve_session = ON;
```
→ Continuité des connexions sans reconnexion applicative

Ces deux fonctionnalités réduisent drastiquement l'impact utilisateur d'un failover.

---

## 1. Types de Failover

### 1.1 Failover vs Switchover

#### **Failover (Bascule d'Urgence)**

```
Définition : Bascule NON PLANIFIÉE suite à une défaillance

Timeline typique :
T+0s    : Master crash (panne hardware)
T+10s   : Détection automatique (monitoring)
T+15s   : Validation (éviter faux positif)
T+30s   : Promotion replica → nouveau master
T+45s   : Reconfiguration autres replicas
T+60s   : Applications redirigées (DNS/VIP)
T+90s   : Service complètement restauré

RTO = ~90 secondes
RPO = 0-30 secondes (dépend semi-sync)
```

**Caractéristiques** :
- ⚠️ **Urgent** : Décision rapide requise
- ⚠️ **Risque** : Perte potentielle de données (si async)
- ⚠️ **Stress** : Situation de crise
- ✅ **Automatisable** : Décision algorithmique possible

#### **Switchover (Bascule Planifiée)**

```
Définition : Bascule PLANIFIÉE pour maintenance

Timeline typique :
T-60min : Annonce maintenance planifiée
T-30min : Préparation (vérif réplication, backup)
T-15min : Mise en read-only master actuel
T-10min : Attente synchronisation complète replicas
T-5min  : Promotion nouveau master
T-2min  : Reconfiguration
T+0min  : Bascule applications (maintenance window)
T+5min  : Validation nouveau master
T+10min : Ancien master rejoint comme replica

RTO = planifié (ex: 5 minutes maintenance)
RPO = 0 (synchronisation garantie)
```

**Caractéristiques** :
- ✅ **Contrôlé** : Timing maîtrisé
- ✅ **Sûr** : Aucune perte de données
- ✅ **Testé** : Répétable pour validation procédures
- ✅ **Rollback** : Possible si problème détecté

💡 **Best Practice** : Effectuer des switchovers réguliers (mensuels) pour :
- Valider les procédures de failover
- Entraîner les équipes
- Détecter problèmes de configuration
- Maintenir les compétences

### 1.2 Failover par Architecture

#### **A. Failover Master-Slave Traditionnel**

```
État Initial :
┌────────────┐
│   Master   │  ← Applications (writes)
└─────┬──────┘
      │ Replication
      ▼
┌────────────┐
│  Replica 1 │  ← Applications (reads)
└────────────┘
      │
      ▼
┌────────────┐
│  Replica 2 │
└────────────┘

Après Failover :
┌────────────┐
│   Master   │  ✖ OFFLINE
└────────────┘

┌────────────┐
│  Replica 1 │  ← Promu NOUVEAU MASTER ✅
└─────┬──────┘
      │ Replication
      ▼
┌────────────┐
│  Replica 2 │  ← Reconfiguré vers nouveau master
└────────────┘
```

**Complexité** :
- Promotion d'un replica
- Reconfiguration des autres replicas
- Gestion du GTID ou positions binlog
- Redirection des applications

#### **B. Failover Galera Cluster**

```
État Initial :
┌─────┐  ┌─────┐  ┌─────┐
│Node1│──│Node2│──│Node3│
└─────┘  └─────┘  └─────┘
  All PRIMARY, All accepting writes

Après Défaillance Node1 :
┌─────┐
│Node1│  ✖ OFFLINE
└─────┘

┌─────┐  ┌─────┐
│Node2│──│Node3│
└─────┘  └─────┘
  Still PRIMARY (quorum 2/3)
  Automatic failover (routing via MaxScale)
```

**Simplicité** :
- ✅ Pas de promotion nécessaire
- ✅ Quorum automatique
- ✅ Routing géré par MaxScale
- ⚠️ Nécessite quorum maintenu

### 1.3 Métriques de Failover

```
┌─────────────────────────────────────────────────────┐
│  Métriques Clés                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  RTO (Recovery Time Objective)                      │
│  ├─ Détection : 5-15 secondes                       │
│  ├─ Validation : 5-10 secondes                      │
│  ├─ Promotion : 10-30 secondes                      │
│  ├─ Reconfiguration : 10-20 secondes                │
│  └─ Total : 30-90 secondes                          │
│                                                     │
│  RPO (Recovery Point Objective)                     │
│  ├─ Async Replication : 1-60 secondes               │
│  ├─ Semi-Sync : 0-5 secondes                        │
│  └─ Galera : 0 secondes (synchrone)                 │
│                                                     │
│  MTBF (Mean Time Between Failures)                  │
│  └─ Hardware moderne : 720 heures (30 jours)        │
│                                                     │
│  MTTR (Mean Time To Repair)                         │
│  ├─ Manuel : 15-60 minutes                          │
│  └─ Automatique : 1-3 minutes                       │
│                                                     │
│  Availability Calculation :                         │
│  = MTBF / (MTBF + MTTR)                             │
│  = 720h / (720h + 0.05h) = 99.993%                  │
└─────────────────────────────────────────────────────┘
```

---

## 2. Détection de Pannes

### 2.1 Méthodes de Détection

#### **A. Health Checks Actifs**

**Ping MySQL** :
```bash
#!/bin/bash
# check_mysql.sh

MYSQL_HOST="master.example.com"
MYSQL_USER="monitor"
MYSQL_PASS="SecurePassword"

# Tentative de connexion
mysqladmin ping \
  -h "$MYSQL_HOST" \
  -u "$MYSQL_USER" \
  -p"$MYSQL_PASS" \
  --connect_timeout=2 \
  2>/dev/null

if [ $? -eq 0 ]; then
    echo "OK: MySQL is alive"
    exit 0
else
    echo "CRITICAL: MySQL is down"
    exit 2
fi
```

**Query Test** :
```sql
-- Health check query (lecture + écriture)
CREATE TABLE IF NOT EXISTS health_check (
    id INT PRIMARY KEY,
    last_check TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Check read
SELECT id FROM health_check WHERE id = 1;

-- Check write
INSERT INTO health_check (id) VALUES (1)
ON DUPLICATE KEY UPDATE last_check = NOW();

-- Vérifier replication lag
SHOW SLAVE STATUS\G
# Seconds_Behind_Master < 10 → OK
```

**Advanced Check (Galera)** :
```sql
-- Vérifier état Galera complet
SELECT 
    @@wsrep_cluster_status = 'Primary' AS is_primary,
    @@wsrep_local_state = 4 AS is_synced,
    @@wsrep_ready = 'ON' AS is_ready,
    @@wsrep_connected = 'ON' AS is_connected,
    @@wsrep_cluster_size >= 3 AS has_quorum;

-- Résultat attendu :
-- +------------+-----------+----------+--------------+------------+
-- | is_primary | is_synced | is_ready | is_connected | has_quorum |
-- +------------+-----------+----------+--------------+------------+
-- |          1 |         1 |        1 |            1 |          1 |
-- +------------+-----------+----------+--------------+------------+
-- → Tous à 1 = serveur sain
```

#### **B. Monitoring Passif**

**Heartbeat Table** :
```sql
-- Table heartbeat (écrite périodiquement par master)
CREATE TABLE heartbeat (
    server_id INT PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    counter BIGINT
) ENGINE=InnoDB;

-- Script sur master (cron chaque seconde)
UPDATE heartbeat 
SET counter = counter + 1, timestamp = NOW() 
WHERE server_id = @@server_id;

-- Vérification sur replica
SELECT 
    server_id,
    timestamp,
    TIMESTAMPDIFF(SECOND, timestamp, NOW()) as lag_seconds
FROM heartbeat;

-- Si lag_seconds > 60 → Master potentiellement down
```

**Network Monitoring** :
```bash
# Ping ICMP
ping -c 3 -W 2 master.example.com

# TCP Port Check
nc -zv master.example.com 3306

# Application-Level
curl -f http://master.example.com:9104/metrics
# (Prometheus exporter)
```

#### **C. Consensus-Based Detection**

```
Multiple Monitoring Nodes :

┌─────────┐  ┌─────────┐  ┌─────────┐
│Monitor1 │  │Monitor2 │  │Monitor3 │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
           Check Master
                  ▼
            ┌──────────┐
            │  Master  │
            └──────────┘

Décision de failover SI :
- 2/3 monitors détectent panne (majorité)
- OU 3/3 monitors (unanimité)

→ Évite faux positifs dus à partition réseau
```

### 2.2 Critères de Validation

**Checklist avant Failover** :
```python
def should_failover(master, replicas):
    """
    Décision de failover basée sur critères multiples
    """
    checks = {
        'master_unreachable': False,
        'validated_by_multiple_monitors': False,
        'replica_available': False,
        'replica_synced': False,
        'no_recent_failover': False,
        'manual_override_disabled': False
    }
    
    # 1. Master vraiment down ?
    if not ping_mysql(master):
        checks['master_unreachable'] = True
    
    # 2. Validé par consensus ?
    down_votes = count_monitors_detecting_failure(master)
    if down_votes >= 2:  # Majorité sur 3
        checks['validated_by_multiple_monitors'] = True
    
    # 3. Replica disponible ?
    best_replica = select_best_replica(replicas)
    if best_replica:
        checks['replica_available'] = True
        
        # 4. Replica synchronisé ?
        lag = get_replication_lag(best_replica)
        if lag < 10:  # Moins de 10 secondes de retard
            checks['replica_synced'] = True
    
    # 5. Éviter flapping (pas de failover récent)
    last_failover_time = get_last_failover_time()
    if (now() - last_failover_time) > timedelta(minutes=10):
        checks['no_recent_failover'] = True
    
    # 6. Pas de verrou manuel
    if not is_failover_locked():
        checks['manual_override_disabled'] = True
    
    # Décision : TOUS les critères doivent être vrais
    return all(checks.values()), checks
```

### 2.3 Éviter les Faux Positifs

**Causes fréquentes de faux positifs** :
```
1. Partition réseau temporaire
   → Solution : Consensus multi-monitors

2. Charge CPU élevée (slow to respond)
   → Solution : Timeouts adaptatifs

3. Maintenance système (reboot)
   → Solution : Maintenance mode flag

4. Network jitter
   → Solution : Multiple failed checks consécutifs

5. DNS issues
   → Solution : Check par IP aussi

6. Monitoring tool failure
   → Solution : Multiple monitoring stacks
```

**Configuration conservatrice** :
```ini
# Orchestrator configuration
[Topology]
# Require 3 consecutive failures
FailureDetectionPeriodBlockMinutes = 10
InstancePollSeconds = 5
# 3 checks × 5s = 15s minimum avant considération

# Semi-sync timeout protection
SemiSyncMasterTimeout = 10s
# Préfère disponibilité sur cohérence après 10s

[Recovery]
# Prevent flapping
PreventCrossDataCenterMasterFailover = true
DelayMasterPromotionIfSQLThreadNotUpToDate = true
```

---

## 3. Solutions de Failover Automatique

### 3.1 MariaDB Monitor (MaxScale)

**Configuration mariadbmon** :
```ini
# /etc/maxscale.cnf
[MariaDB-Monitor]
type = monitor
module = mariadbmon
servers = server1, server2, server3
user = maxscale_monitor
password = SecurePassword

# ========================================
# FAILOVER CONFIGURATION
# ========================================

# Activer failover automatique
auto_failover = true

# Rejoindre automatiquement ancien master comme replica
auto_rejoin = true

# Promotion basée sur :
# - Replication lag minimal
# - GTID position
# - Priority weight (optionnel)

# ========================================
# SWITCHOVER CONFIGURATION
# ========================================

# Switchover planifié possible via API
switchover_timeout = 90s

# Vérifications avant switchover
enforce_read_only_slaves = true

# ========================================
# DETECTION
# ========================================

# Interval de vérification
monitor_interval = 2000ms

# Nombre de checks avant déclaration panne
failcount = 3
# 3 × 2s = 6 secondes minimum

# Timeout pour considérer serveur down
backend_connect_timeout = 3s
backend_read_timeout = 2s
backend_write_timeout = 2s

# ========================================
# CONSTRAINTS
# ========================================

# Prevent flapping
failover_timeout = 90s
# Pas de nouveau failover pendant 90s après un failover

# Disk space monitoring
switchover_on_low_disk_space = true
disk_space_threshold = 20%  # Switchover si < 20% espace

# ========================================
# EXTERNAL SCRIPTS
# ========================================

# Scripts pre/post failover
script = /usr/local/bin/failover-notify.sh
# Paramètres passés au script :
# $1 = event type (master_down, server_down, etc.)
# $2 = server name
# $3 = cluster info (JSON)
```

**Commandes manuelles** :
```bash
# Déclencher switchover manuel (maintenance)
maxctrl call command mariadbmon switchover \
  MariaDB-Monitor \
  server2 \
  server1

# Réinitialiser réplication
maxctrl call command mariadbmon reset-replication \
  MariaDB-Monitor \
  server1

# Rejoindre cluster
maxctrl call command mariadbmon rejoin \
  MariaDB-Monitor \
  server3
```

### 3.2 Orchestrator

**Installation** :
```bash
# Installation Orchestrator
wget https://github.com/openark/orchestrator/releases/download/v3.2.6/orchestrator-3.2.6-1.x86_64.rpm
rpm -ivh orchestrator-3.2.6-1.x86_64.rpm

# Configuration backend database
mysql -u root -p << EOF
CREATE DATABASE orchestrator;
GRANT ALL ON orchestrator.* TO 'orchestrator'@'localhost' 
  IDENTIFIED BY 'SecurePassword';
EOF
```

**Configuration** :
```json
// /etc/orchestrator.conf.json
{
  "Debug": false,
  "ListenAddress": ":3000",
  
  "MySQLTopologyUser": "orchestrator",
  "MySQLTopologyPassword": "SecurePassword",
  
  "MySQLOrchestratorHost": "localhost",
  "MySQLOrchestratorPort": 3306,
  "MySQLOrchestratorDatabase": "orchestrator",
  "MySQLOrchestratorUser": "orchestrator",
  "MySQLOrchestratorPassword": "SecurePassword",
  
  "DiscoverByShowSlaveHosts": true,
  "InstancePollSeconds": 5,
  "DiscoveryPollSeconds": 5,
  
  "RecoveryPeriodBlockSeconds": 600,
  "RecoveryIgnoreHostnameFilters": [],
  "RecoverMasterClusterFilters": ["*"],
  "RecoverIntermediateMasterClusterFilters": ["*"],
  
  "OnFailureDetectionProcesses": [
    "/usr/local/bin/orchestrator-notify.sh {failureType} {failureCluster} {failedHost}"
  ],
  
  "PreFailoverProcesses": [
    "/usr/local/bin/pre-failover-check.sh {failureCluster} {candidateHost}"
  ],
  
  "PostMasterFailoverProcesses": [
    "/usr/local/bin/post-failover.sh {successorHost} {failedHost} {failureCluster}"
  ],
  
  "DetachLostReplicasAfterMasterFailover": true,
  "ApplyMySQLPromotionAfterMasterFailover": true,
  "MasterFailoverLostInstancesDowntimeMinutes": 10,
  
  "HTTPAdvertise": "http://orchestrator.example.com:3000"
}
```

**Hooks de notification** :
```bash
#!/bin/bash
# /usr/local/bin/post-failover.sh

SUCCESSOR_HOST="$1"
FAILED_HOST="$2"
CLUSTER="$3"

# Notification Slack
curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
  -H 'Content-Type: application/json' \
  -d "{
    \"text\": \"🚨 FAILOVER EXECUTED\",
    \"attachments\": [{
      \"color\": \"danger\",
      \"fields\": [
        {\"title\": \"Cluster\", \"value\": \"$CLUSTER\", \"short\": true},
        {\"title\": \"Failed\", \"value\": \"$FAILED_HOST\", \"short\": true},
        {\"title\": \"New Master\", \"value\": \"$SUCCESSOR_HOST\", \"short\": true},
        {\"title\": \"Time\", \"value\": \"$(date)\", \"short\": true}
      ]
    }]
  }"

# Update DNS (exemple Route53)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"master.db.example.com\",
        \"Type\": \"A\",
        \"TTL\": 60,
        \"ResourceRecords\": [{\"Value\": \"$(dig +short $SUCCESSOR_HOST)\"}]
      }
    }]
  }"

# Créer ticket incident
curl -X POST https://your-ticketing-system/api/incidents \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -d "{
    \"title\": \"Database Failover - $CLUSTER\",
    \"description\": \"Automatic failover from $FAILED_HOST to $SUCCESSOR_HOST\",
    \"severity\": \"high\",
    \"assigned_to\": \"dba-team\"
  }"
```

### 3.3 MHA (Master High Availability)

**Configuration Manager** :
```ini
# /etc/mha/app1.cnf
[server default]
# Manager settings
manager_workdir=/var/log/mha/app1
manager_log=/var/log/mha/app1/manager.log
remote_workdir=/var/log/mha/app1

# SSH settings
ssh_user=mha
ssh_port=22

# MySQL settings
user=mha
password=SecurePassword
repl_user=repl_user
repl_password=ReplicationPassword

# Failover settings
ping_interval=3
secondary_check_script=/usr/local/bin/masterha_secondary_check \
  -s remote_host1 -s remote_host2

# Post-failover
master_ip_failover_script=/usr/local/bin/master_ip_failover
shutdown_script=""
report_script=/usr/local/bin/mha_report

# Servers
[server1]
hostname=db1.example.com
port=3306
candidate_master=1
check_repl_delay=0

[server2]
hostname=db2.example.com
port=3306
candidate_master=1
check_repl_delay=0

[server3]
hostname=db3.example.com
port=3306
no_master=1
```

**VIP Failover Script** :
```bash
#!/bin/bash
# /usr/local/bin/master_ip_failover

command=$1
orig_master_host=$2
new_master_host=$3
orig_master_port=$4
new_master_port=$5

VIP=10.0.1.100
NETMASK=24
INTERFACE=eth0

case "$command" in
    stop)
        # Remove VIP from old master
        ssh $orig_master_host "ip addr del $VIP/$NETMASK dev $INTERFACE"
        exit 0
        ;;
    start)
        # Add VIP to new master
        ssh $new_master_host "ip addr add $VIP/$NETMASK dev $INTERFACE"
        ssh $new_master_host "arping -c 3 -I $INTERFACE -s $VIP $VIP"
        exit 0
        ;;
    *)
        echo "Usage: $0 {stop|start}"
        exit 1
        ;;
esac
```

⚠️ **Note** : MHA est en maintenance limitée. Préférer mariadbmon ou Orchestrator pour nouveaux déploiements.

---

## 4. 🆕 Transaction Replay et Connection Migration (MariaDB 11.8)

### 4.1 Transaction Replay

**Problème résolu** :
```
Scenario AVANT MariaDB 11.8 :

Application :
  BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 123;
  UPDATE accounts SET balance = balance + 100 WHERE id = 456;
  COMMIT;  ← Master crash PENDANT le commit
  
Result :
  ❌ Error: Lost connection to MySQL server during query
  ❌ Application must handle retry manually
  ❌ Risk of duplicate transaction if retry logic buggy
```

**Solution MariaDB 11.8** :
```sql
-- Configuration globale
SET GLOBAL transaction_replay = ON;
SET GLOBAL transaction_replay_attempts = 3;
SET GLOBAL transaction_replay_timeout = 30;  -- secondes

-- Configuration par session (optionnel)
SET SESSION transaction_replay = ON;
```

**Fonctionnement** :
```
Application :
  BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 123;
  UPDATE accounts SET balance = balance + 100 WHERE id = 456;
  COMMIT;  ← Master crash
  
MariaDB 11.8 (automatiquement) :
  1. Détecte perte connexion pendant commit
  2. Vérifie si transaction commitée sur ancien master
     - Si OUI : Retourne success à l'application ✅
     - Si NON : Rejoue transaction sur nouveau master
  3. Retente jusqu'à 3 fois (transaction_replay_attempts)
  4. Retourne résultat à l'application
  
Application :
  ✅ Reçoit success (transparence totale)
  ✅ Pas de code retry nécessaire
```

**Limitations** :
```sql
-- Transaction Replay fonctionne pour :
✅ Transactions standards (BEGIN...COMMIT)
✅ Autocommit statements
✅ READ COMMITTED et REPEATABLE READ

-- Ne fonctionne PAS pour :
❌ LOCK TABLES
❌ GET_LOCK() / RELEASE_LOCK()
❌ Transactions avec side effects externes
❌ SERIALIZABLE isolation level
```

**Configuration production** :
```ini
# /etc/mysql/conf.d/transaction-replay.cnf
[mysqld]
# Activer transaction replay
transaction_replay = ON

# Nombre de tentatives
transaction_replay_attempts = 3

# Timeout pour replay
transaction_replay_timeout = 30

# Logging (debug)
log_error_verbosity = 3
# Logs détaillés dans error log :
# "Transaction replay: attempting replay of transaction XYZ"
# "Transaction replay: successfully replayed transaction XYZ"
# "Transaction replay: failed to replay transaction XYZ after 3 attempts"
```

**Monitoring** :
```sql
-- Métriques transaction replay
SHOW GLOBAL STATUS LIKE 'Transaction_replay%';

-- +----------------------------------+-------+
-- | Variable_name                    | Value |
-- +----------------------------------+-------+
-- | Transaction_replay_attempted     | 1234  |
-- | Transaction_replay_succeeded     | 1198  |
-- | Transaction_replay_failed        | 36    |
-- | Transaction_replay_avg_time_ms   | 45    |
-- +----------------------------------+-------+

-- Taux de succès
SELECT 
    (Transaction_replay_succeeded / Transaction_replay_attempted) * 100 
    AS success_rate_percent;
```

### 4.2 Connection Migration

**Problème résolu** :
```
Scenario AVANT MariaDB 11.8 :

Application établit connexion avec :
  - SET SESSION time_zone = 'America/New_York'
  - SET SESSION sql_mode = 'STRICT_TRANS_TABLES'
  - PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?'
  
Master failover →
  ❌ Connexion coupée
  ❌ Variables de session perdues
  ❌ Prepared statements perdus
  ❌ Application doit tout réinitialiser
```

**Solution MariaDB 11.8** :
```sql
-- Configuration globale
SET GLOBAL connection_migration = ON;
SET GLOBAL connection_migration_preserve_session = ON;
SET GLOBAL connection_migration_preserve_prepared = ON;
```

**Fonctionnement** :
```
Application :
  -- Session setup
  SET time_zone = 'America/New_York';
  SET sql_mode = 'STRICT_TRANS_TABLES';
  PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?';
  
Master failover (transparent) →
  
MariaDB 11.8 (automatiquement) :
  1. Détecte failover
  2. Établit nouvelle connexion vers nouveau master
  3. Réapplique variables de session :
     - SET time_zone = 'America/New_York'
     - SET sql_mode = 'STRICT_TRANS_TABLES'
  4. Recrée prepared statements :
     - PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?'
  5. Retourne contrôle à l'application
  
Application :
  ✅ Continue comme si rien ne s'était passé
  EXECUTE stmt USING @user_id;  -- Fonctionne !
```

**Configuration production** :
```ini
# /etc/mysql/conf.d/connection-migration.cnf
[mysqld]
# Activer connection migration
connection_migration = ON

# Préserver session variables
connection_migration_preserve_session = ON

# Préserver prepared statements
connection_migration_preserve_prepared = ON

# Timeout migration
connection_migration_timeout = 10

# Retry
connection_migration_max_retries = 3
```

**Variables préservées** :
```sql
-- Variables de session migrées automatiquement :
SET time_zone = '...';
SET sql_mode = '...';
SET autocommit = ...;
SET transaction_isolation = '...';
SET character_set_client = '...';
SET character_set_connection = '...';
SET character_set_results = '...';
SET collation_connection = '...';
-- + autres variables SET SESSION
```

**Limitations** :
```sql
-- NE SONT PAS migrés :
❌ Temporary tables (CREATE TEMPORARY TABLE)
❌ User-defined variables (@var = value)
❌ LOCK TABLES actifs
❌ GET_LOCK() actifs
❌ Transactions en cours (voir Transaction Replay)
```

### 4.3 Combinaison Transaction Replay + Connection Migration

**Scénario optimal** :
```sql
-- Configuration complète haute disponibilité
SET GLOBAL transaction_replay = ON;
SET GLOBAL transaction_replay_attempts = 3;
SET GLOBAL connection_migration = ON;
SET GLOBAL connection_migration_preserve_session = ON;

-- Application (Java exemple)
Connection conn = DriverManager.getConnection(
    "jdbc:mariadb://maxscale:3306/mydb",
    props
);

// Configuration session
conn.createStatement().execute("SET time_zone = 'UTC'");
PreparedStatement pstmt = conn.prepareStatement(
    "UPDATE accounts SET balance = balance + ? WHERE id = ?"
);

// Transaction
conn.setAutoCommit(false);
pstmt.setDouble(1, 100.0);
pstmt.setInt(2, 123);
pstmt.executeUpdate();
conn.commit();  ← Master failover ICI

// Résultat :
// ✅ Transaction Replay : Transaction rejouée sur nouveau master
// ✅ Connection Migration : Session et prepared statement préservés
// ✅ Application : Aucune erreur, tout transparent !
```

**Métriques combinées** :
```sql
-- Dashboard monitoring
SELECT 
    'Transaction Replay' AS feature,
    Transaction_replay_attempted AS attempted,
    Transaction_replay_succeeded AS succeeded,
    Transaction_replay_failed AS failed,
    CONCAT(
        ROUND((Transaction_replay_succeeded / Transaction_replay_attempted) * 100, 2),
        '%'
    ) AS success_rate
FROM information_schema.GLOBAL_STATUS
WHERE Variable_name LIKE 'Transaction_replay%'

UNION ALL

SELECT 
    'Connection Migration',
    Connection_migration_attempted,
    Connection_migration_succeeded,
    Connection_migration_failed,
    CONCAT(
        ROUND((Connection_migration_succeeded / Connection_migration_attempted) * 100, 2),
        '%'
    )
FROM information_schema.GLOBAL_STATUS
WHERE Variable_name LIKE 'Connection_migration%';
```

---

## 5. Stratégies de Récupération

### 5.1 Procédure Post-Failover

```bash
#!/bin/bash
# post-failover-recovery.sh

NEW_MASTER="$1"
OLD_MASTER="$2"

echo "=== Post-Failover Recovery ==="
echo "New Master: $NEW_MASTER"
echo "Old Master: $OLD_MASTER"

# 1. Valider nouveau master opérationnel
echo "[1/7] Validating new master..."
mysql -h $NEW_MASTER -e "SELECT 1" || exit 1

# 2. Vérifier réplication des replicas vers nouveau master
echo "[2/7] Checking replica replication..."
for replica in $(cat /etc/replicas.list); do
    mysql -h $replica -e "SHOW SLAVE STATUS\G" | grep -q "$NEW_MASTER"
    if [ $? -ne 0 ]; then
        echo "WARNING: $replica not replicating from $NEW_MASTER"
    fi
done

# 3. Monitorer lag
echo "[3/7] Monitoring replication lag..."
for i in {1..60}; do
    max_lag=$(mysql -h $NEW_MASTER -N -e "
        SELECT COALESCE(MAX(Seconds_Behind_Master), 0)
        FROM information_schema.SLAVE_HOSTS
        JOIN information_schema.PROCESSLIST 
        ON SLAVE_HOSTS.Host = PROCESSLIST.Host
    ")
    
    if [ "$max_lag" -lt 10 ]; then
        echo "✅ All replicas caught up (max lag: ${max_lag}s)"
        break
    fi
    
    echo "Waiting for replicas to catch up (max lag: ${max_lag}s)..."
    sleep 5
done

# 4. Backup immédiat du nouveau master
echo "[4/7] Creating post-failover backup..."
mariabackup --backup \
    --target-dir=/backup/post-failover-$(date +%s) \
    --user=backup \
    --password=SecurePassword \
    --host=$NEW_MASTER

# 5. Tenter récupération ancien master (si accessible)
echo "[5/7] Attempting to recover old master..."
if ping -c 1 $OLD_MASTER &>/dev/null; then
    echo "Old master is reachable, attempting to join as replica..."
    
    # Récupérer GTID position
    NEW_MASTER_GTID=$(mysql -h $NEW_MASTER -N -e "SELECT @@gtid_current_pos")
    
    # Reconfigurer ancien master comme replica
    ssh $OLD_MASTER "mysql -e \"
        STOP SLAVE;
        CHANGE MASTER TO 
            MASTER_HOST='$NEW_MASTER',
            MASTER_USER='repl_user',
            MASTER_PASSWORD='ReplicationPassword',
            MASTER_USE_GTID=current_pos;
        SET GLOBAL gtid_slave_pos='$NEW_MASTER_GTID';
        START SLAVE;
    \""
else
    echo "Old master not reachable, manual intervention required"
fi

# 6. Notification équipe
echo "[6/7] Sending notifications..."
curl -X POST https://hooks.slack.com/YOUR/WEBHOOK \
    -d "{
        \"text\": \"✅ Post-Failover Recovery Complete\",
        \"attachments\": [{
            \"fields\": [
                {\"title\": \"New Master\", \"value\": \"$NEW_MASTER\"},
                {\"title\": \"Old Master Status\", \"value\": \"$(ping -c 1 $OLD_MASTER &>/dev/null && echo 'Rejoined as replica' || echo 'Offline')\"},
                {\"title\": \"Time\", \"value\": \"$(date)\"}
            ]
        }]
    }"

# 7. Créer post-mortem ticket
echo "[7/7] Creating post-mortem ticket..."
curl -X POST https://your-ticketing-system/api/issues \
    -H 'Content-Type: application/json' \
    -d "{
        \"title\": \"Post-Mortem: Database Failover $(date +%Y-%m-%d)\",
        \"description\": \"Analyze root cause of failover from $OLD_MASTER to $NEW_MASTER\",
        \"labels\": [\"post-mortem\", \"database\", \"high-priority\"]
    }"

echo "=== Recovery Complete ==="
```

### 5.2 Rollback Strategy

**Quand rollback ?**
```
Situations nécessitant rollback :
1. Nouveau master instable (crashes répétés)
2. Corruption données détectée
3. Problème applicatif grave
4. Performance inacceptable
```

**Procédure rollback** :
```bash
#!/bin/bash
# rollback-failover.sh

ORIGINAL_MASTER="$1"  # Serveur qu'on veut remettre master
CURRENT_MASTER="$2"   # Master actuel (post-failover)

echo "=== Rollback Failover ==="
echo "Target: Make $ORIGINAL_MASTER master again"

# Pré-requis : Original master doit être opérationnel
mysql -h $ORIGINAL_MASTER -e "SELECT 1" || {
    echo "ERROR: Original master not accessible"
    exit 1
}

# 1. Vérifier que original master est à jour
ORIGINAL_GTID=$(mysql -h $ORIGINAL_MASTER -N -e "SELECT @@gtid_current_pos")
CURRENT_GTID=$(mysql -h $CURRENT_MASTER -N -e "SELECT @@gtid_current_pos")

echo "Original Master GTID: $ORIGINAL_GTID"
echo "Current Master GTID: $CURRENT_GTID"

# 2. Mettre current master en read-only
echo "Setting current master to read-only..."
mysql -h $CURRENT_MASTER -e "SET GLOBAL read_only = ON"

# 3. Attendre synchronisation complète
echo "Waiting for original master to catch up..."
while true; do
    ORIG_GTID=$(mysql -h $ORIGINAL_MASTER -N -e "SELECT @@gtid_current_pos")
    CURR_GTID=$(mysql -h $CURRENT_MASTER -N -e "SELECT @@gtid_current_pos")
    
    if [ "$ORIG_GTID" == "$CURR_GTID" ]; then
        echo "✅ Fully synchronized"
        break
    fi
    
    echo "Waiting... (${ORIG_GTID} vs ${CURR_GTID})"
    sleep 2
done

# 4. Arrêter réplication sur original master
mysql -h $ORIGINAL_MASTER -e "STOP SLAVE"

# 5. Promouvoir original master
mysql -h $ORIGINAL_MASTER -e "
    STOP SLAVE;
    RESET SLAVE ALL;
    SET GLOBAL read_only = OFF;
"

# 6. Reconfigurer current master comme replica
mysql -h $CURRENT_MASTER -e "
    STOP SLAVE;
    CHANGE MASTER TO 
        MASTER_HOST='$ORIGINAL_MASTER',
        MASTER_USER='repl_user',
        MASTER_PASSWORD='ReplicationPassword',
        MASTER_USE_GTID=current_pos;
    START SLAVE;
"

# 7. Rediriger applications
echo "Update your application configuration to point to $ORIGINAL_MASTER"
echo "Or update DNS/VIP"

echo "=== Rollback Complete ==="
```

### 5.3 Testing de Failover

**Chaos Engineering pour Databases** :
```bash
#!/bin/bash
# chaos-failover-test.sh

echo "=== Chaos Failover Test ==="
echo "⚠️  WARNING: This will simulate a master failure"
read -p "Continue? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Aborted"
    exit 0
fi

MASTER_HOST="db-master.example.com"
START_TIME=$(date +%s)

# 1. Baseline metrics
echo "[1/5] Capturing baseline metrics..."
BASELINE_QPS=$(mysql -h $MASTER_HOST -N -e "SHOW GLOBAL STATUS LIKE 'Queries'" | awk '{print $2}')

# 2. Simulate master crash
echo "[2/5] Simulating master crash (stopping MySQL)..."
ssh $MASTER_HOST "systemctl stop mariadb"

# 3. Wait for failover
echo "[3/5] Waiting for automatic failover..."
FAILOVER_DETECTED=false
for i in {1..60}; do
    # Check if new master elected
    NEW_MASTER=$(maxctrl list servers --tsv | grep Master | awk '{print $1}')
    
    if [ -n "$NEW_MASTER" ] && [ "$NEW_MASTER" != "$MASTER_HOST" ]; then
        FAILOVER_TIME=$(($(date +%s) - START_TIME))
        echo "✅ Failover completed to $NEW_MASTER in ${FAILOVER_TIME}s"
        FAILOVER_DETECTED=true
        break
    fi
    
    echo "Waiting for failover... (${i}s)"
    sleep 1
done

if [ "$FAILOVER_DETECTED" = false ]; then
    echo "❌ Failover did not complete within 60s"
    exit 1
fi

# 4. Validate application connectivity
echo "[4/5] Validating application connectivity..."
APP_ERROR_COUNT=0
for i in {1..10}; do
    mysql -h maxscale.example.com -e "SELECT 1" &>/dev/null || APP_ERROR_COUNT=$((APP_ERROR_COUNT + 1))
    sleep 1
done

if [ $APP_ERROR_COUNT -gt 2 ]; then
    echo "❌ High error rate: $APP_ERROR_COUNT/10"
else
    echo "✅ Application connectivity OK: $((10 - APP_ERROR_COUNT))/10 success"
fi

# 5. Cleanup - restart old master
echo "[5/5] Cleanup - restarting old master..."
ssh $MASTER_HOST "systemctl start mariadb"

# Report
cat << EOF

=== Failover Test Report ===
Failover Time: ${FAILOVER_TIME}s
New Master: $NEW_MASTER
Application Errors: $APP_ERROR_COUNT/10 queries
Status: $([ $APP_ERROR_COUNT -gt 2 ] && echo "❌ FAILED" || echo "✅ PASSED")

RTO Achieved: ${FAILOVER_TIME}s
Target RTO: 90s
Result: $([ $FAILOVER_TIME -le 90 ] && echo "✅ Within SLA" || echo "❌ Exceeds SLA")
EOF
```

---

## 6. Monitoring et Alerting

### 6.1 Métriques Critiques

```yaml
# prometheus_alerts.yml
groups:
  - name: mariadb_failover
    interval: 10s
    rules:
      # Master down detection
      - alert: MariaDBMasterDown
        expr: mysql_up{role="master"} == 0
        for: 15s
        labels:
          severity: critical
        annotations:
          summary: "MariaDB Master {{ $labels.instance }} is down"
          description: "Automatic failover should trigger within 90s"
      
      # Replication lag
      - alert: MariaDBReplicationLagHigh
        expr: mysql_slave_lag_seconds > 60
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High replication lag on {{ $labels.instance }}"
          description: "Lag: {{ $value }}s - May impact failover RPO"
      
      # No master detected
      - alert: MariaDBNoMasterDetected
        expr: count(mysql_up{role="master"} == 1) == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "No MariaDB master detected in cluster"
          description: "Failover may have failed or is in progress"
      
      # Multiple masters (split-brain)
      - alert: MariaDBMultipleMasters
        expr: count(mysql_up{role="master"} == 1) > 1
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: "Multiple MariaDB masters detected (split-brain)"
          description: "IMMEDIATE ACTION REQUIRED - {{ $value }} masters active"
      
      # Transaction Replay failures (11.8)
      - alert: TransactionReplayFailureHigh
        expr: rate(mysql_transaction_replay_failed[5m]) > 0.01
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High transaction replay failure rate"
          description: "Rate: {{ $value }} failures/sec on {{ $labels.instance }}"
```

### 6.2 Dashboard Grafana

```json
{
  "dashboard": {
    "title": "MariaDB Failover Monitoring",
    "panels": [
      {
        "title": "Cluster Status",
        "targets": [
          {
            "expr": "mysql_up",
            "legendFormat": "{{ instance }} - {{ role }}"
          }
        ]
      },
      {
        "title": "Replication Lag",
        "targets": [
          {
            "expr": "mysql_slave_lag_seconds",
            "legendFormat": "{{ instance }}"
          }
        ]
      },
      {
        "title": "Failover Events (Last 24h)",
        "targets": [
          {
            "expr": "changes(mysql_master_host[24h])",
            "legendFormat": "Failovers"
          }
        ]
      },
      {
        "title": "Transaction Replay (11.8)",
        "targets": [
          {
            "expr": "rate(mysql_transaction_replay_attempted[5m])",
            "legendFormat": "Attempted"
          },
          {
            "expr": "rate(mysql_transaction_replay_succeeded[5m])",
            "legendFormat": "Succeeded"
          },
          {
            "expr": "rate(mysql_transaction_replay_failed[5m])",
            "legendFormat": "Failed"
          }
        ]
      }
    ]
  }
}
```

---

## ✅ Points Clés à Retenir

- **Failover automatique** : RTO 30-90s vs 15-60min manuel
- **Détection fiable** : Consensus multi-monitors, éviter faux positifs
- **mariadbmon** (MaxScale) : Solution native recommandée pour MariaDB
- **Orchestrator** : Alternative puissante, GUI web, mature
- **🆕 Transaction Replay** (11.8) : Transparence applicative totale
- **🆕 Connection Migration** (11.8) : Préservation session et prepared statements
- **Testing régulier** : Chaos engineering mensuel recommandé
- **Post-failover** : Procédure de récupération documentée essentielle
- **Monitoring** : Alerting proactif sur lag, master status, replay failures
- **Rollback strategy** : Plan B testé en cas de failover problématique

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 MariaDB Monitor Documentation](https://mariadb.com/kb/en/mariadb-monitor/)
- [📖 Transaction Replay (11.8)](https://mariadb.com/kb/en/transaction-replay/)
- [📖 Connection Migration (11.8)](https://mariadb.com/kb/en/connection-migration/)

### Outils Open Source
- [Orchestrator](https://github.com/openark/orchestrator)
- [MHA](https://github.com/yoshinorim/mha4mysql-manager)
- [MaxScale](https://github.com/mariadb-corporation/MaxScale)

### Articles et Blogs
- **"Automated Failover Best Practices"** - Percona Blog
- **"Transaction Replay Deep Dive"** - MariaDB Engineering Blog
- **"Chaos Engineering for Databases"** - Netflix Tech Blog

---

## ➡️ Section Suivante

**[14.7 Virtual IP et keepalived](/14-haute-disponibilite/07-virtual-ip-keepalived.md)**

La section suivante détaillera l'utilisation de VIP (Virtual IP) avec keepalived pour fournir un point d'accès unique et gérer le basculement réseau lors d'un failover.

---

**Le failover automatique n'est pas une option en production moderne, c'est une nécessité. MariaDB 11.8 avec Transaction Replay et Connection Migration élève la barre de ce qui est possible.**

⏭️ [Virtual IP et keepalived](/14-haute-disponibilite/07-virtual-ip-keepalived.md)

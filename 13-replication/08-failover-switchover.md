🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.8 Failover et Switchover

> **Niveau** : Avancé  
> **Durée estimée** : 2.5-3 heures  
> **Prérequis** : 
> - Sections 13.1 à 13.7 (Réplication complète)
> - Compréhension des architectures haute disponibilité
> - Expérience en gestion d'incidents production
> - Connaissance des SLA et métriques RTO/RPO

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Différencier failover (non planifié) et switchover (planifié)
- Exécuter un failover manuel en cas de panne Primary
- Planifier et exécuter un switchover sans perte de données
- Utiliser GTID pour simplifier les opérations de basculement
- Automatiser le failover avec des outils spécialisés
- Gérer la reconfiguration des applications et clients
- Tester régulièrement les procédures de failover
- Définir et respecter des objectifs RTO/RPO

---

## Introduction

Le **failover** et le **switchover** sont deux opérations essentielles pour assurer la **continuité de service** d'une infrastructure MariaDB répliquée.

### Définitions

**Failover (basculement d'urgence)** :
```
Situation : Primary tombe en panne (crash, hardware failure)
Action : Promotion d'urgence d'un Replica en nouveau Primary
Nature : NON PLANIFIÉE, réactive
Objectif : Minimiser downtime (RTO)
Risque : Potentielle perte de données (RPO > 0)
```

**Switchover (basculement planifié)** :
```
Situation : Maintenance planifiée, upgrade, migration
Action : Promotion ordonnée d'un Replica en nouveau Primary
Nature : PLANIFIÉE, contrôlée
Objectif : Zero downtime, zero data loss
Risque : Minimal (procédure testée)
```

### Comparaison

| Aspect | Failover | Switchover |
|--------|----------|------------|
| **Déclencheur** | Panne inattendue | Opération planifiée |
| **Urgence** | 🔴 Critique | 🟢 Contrôlée |
| **Downtime** | Minimiser (1-5min) | Quasi-zero (< 30s) |
| **Perte données** | Possible (RPO > 0) | Zero (RPO = 0) |
| **Complexité** | Élevée (stress) | Moyenne (procédure) |
| **Testing** | Simulation difficile | Tests réguliers possibles |
| **Rollback** | Complexe | Possible |

### Architecture de référence

```
┌─────────────────────────────────────────────────────────────┐
│                TOPOLOGIE HAUTE DISPONIBILITÉ                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    ┌──────────────┐                         │
│                    │   PRIMARY    │                         │
│                    │  (Active)    │                         │
│                    │              │                         │
│                    │  GTID: 0-1-  │                         │
│                    │  1000        │                         │
│                    └──────┬───────┘                         │
│                           │                                 │
│              ┌────────────┼────────────┐                    │
│              │            │            │                    │
│              ▼            ▼            ▼                    │
│      ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│      │ Replica 1  │ │ Replica 2  │ │ Replica 3  │           │
│      │ (Standby)  │ │ (Reads)    │ │ (Backup)   │           │
│      │            │ │            │ │            │           │
│      │ GTID: 0-1- │ │ GTID: 0-1- │ │ GTID: 0-1- │           │
│      │ 1000       │ │ 999        │ │ 998        │           │
│      └────────────┘ └────────────┘ └────────────┘           │
│                                                             │
│      Failover Scenario:                                     │
│      Primary CRASH 💥                                       │
│      → Promote Replica 1 (most up-to-date)                  │
│      → Reconfigure Replica 2, 3 to new Primary              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Métriques Critiques

### RTO (Recovery Time Objective)

**Définition** : Temps maximum acceptable de downtime

```
RTO = Temps de détection + Temps de décision + Temps d'exécution

Exemple:
- Détection panne: 30 secondes (monitoring)
- Décision promotion: 30 secondes (manuel ou auto)
- Exécution failover: 2 minutes
- RTO total = 3 minutes
```

**Objectifs typiques** :

| Criticité | RTO Target | Stratégie |
|-----------|------------|-----------|
| Critique (e-commerce) | < 1 minute | Failover automatique |
| Production standard | < 5 minutes | Failover manuel supervisé |
| Non-critique | < 30 minutes | Failover manuel |

### RPO (Recovery Point Objective)

**Définition** : Quantité maximale acceptable de perte de données

```
RPO = Temps entre dernière transaction confirmée et crash

Avec réplication asynchrone:
RPO = Lag de réplication au moment du crash
    = Secondes_Behind_Master

Avec réplication semi-synchrone:
RPO = 0 (zero data loss)
```

**Calcul** :

```sql
-- Estimer RPO actuel
SELECT 
  IFNULL(Seconds_Behind_Master, 'N/A') AS current_lag_sec,
  CASE 
    WHEN Seconds_Behind_Master IS NULL 
      THEN 'INFINITE (not replicating)'
    WHEN Seconds_Behind_Master = 0 
      THEN 'ZERO (synchronized)'
    WHEN Seconds_Behind_Master < 5 
      THEN 'LOW (< 5s data loss risk)'
    WHEN Seconds_Behind_Master < 60 
      THEN 'MEDIUM (< 1min data loss risk)'
    ELSE 'HIGH (> 1min data loss risk)'
  END AS rpo_risk
FROM information_schema.SLAVE_STATUS;
```

**Objectifs typiques** :

| Criticité | RPO Target | Stratégie |
|-----------|------------|-----------|
| Financier | 0 (zero loss) | Semi-sync + validation |
| Critique | < 5 secondes | Semi-sync ou async faible lag |
| Standard | < 1 minute | Async avec monitoring |
| Non-critique | < 5 minutes | Async |

---

## Failover (Basculement d'Urgence)

### Scénario type

```
Timeline d'un failover:

T0: 14:35:00 - Primary fonctionne normalement
               Application écrit/lit sur Primary
               Replicas répliquent (lag: 2s)

T1: 14:35:15 - Primary CRASH 💥
               (Cause: hardware failure, kernel panic, etc.)
               
T2: 14:35:30 - Monitoring détecte Primary DOWN
               Alerte envoyée aux DBAs

T3: 14:35:45 - Décision: Promouvoir Replica1
               (Replica le plus à jour)

T4: 14:36:00 - Exécution failover
               - STOP SLAVE sur Replica1
               - Promotion Replica1 → New Primary
               - Reconfiguration Replica2, Replica3

T5: 14:37:00 - Applications basculées vers New Primary
               Service restauré ✓

Downtime total: 2 minutes (T1 → T5)
Data loss: ~2 secondes (lag au moment T1)
```

### Procédure manuelle de failover

**Étape 1 : Identifier le Replica à promouvoir**

```sql
-- Sur chaque Replica: Vérifier la position GTID
SELECT 
  @@hostname AS replica,
  @@gtid_slave_pos AS gtid_position
FROM dual;

-- Output:
-- +----------+---------------+
-- | replica  | gtid_position |
-- +----------+---------------+
-- | replica1 | 0-1-1000      |  ← Plus à jour
-- | replica2 | 0-1-999       |
-- | replica3 | 0-1-995       |
-- +----------+---------------+

-- Choisir replica1 (position la plus avancée)
```

**Sans GTID** : Comparer positions binlog

```sql
-- Sur chaque Replica
SELECT 
  @@hostname,
  Master_Log_File,
  Read_Master_Log_Pos
FROM information_schema.SLAVE_STATUS;

-- Choisir le Replica avec:
-- 1. Fichier binlog le plus récent
-- 2. Si même fichier, position la plus élevée
```

**Étape 2 : Promouvoir le Replica**

```sql
-- Sur Replica1 (futur nouveau Primary)

-- 1. Arrêter la réplication
STOP SLAVE;

-- 2. Vérifier qu'aucune réplication ne tourne
SHOW SLAVE STATUS\G
-- Doit être vide ou Slave_IO_Running: No

-- 3. Réinitialiser statut réplication
RESET SLAVE ALL;

-- 4. Désactiver read-only
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;

-- 5. (Optionnel) Flush logs
FLUSH LOGS;

-- 6. Replica1 est maintenant le nouveau Primary ✓
SELECT @@read_only;  -- Doit être 0
```

**Étape 3 : Reconfigurer les autres Replicas**

```sql
-- Sur Replica2 et Replica3

-- 1. Arrêter réplication de l'ancien Primary
STOP SLAVE;

-- 2. Pointer vers le nouveau Primary (Replica1)
CHANGE MASTER TO
  MASTER_HOST = 'replica1.example.com',  -- Nouveau Primary
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;  -- Avec GTID: automatique !

-- Sans GTID, calculer position:
-- (Complexe, nécessite analyse binlog du nouveau Primary)

-- 3. Redémarrer réplication
START SLAVE;

-- 4. Vérifier
SHOW SLAVE STATUS\G
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
```

**Étape 4 : Basculer les applications**

```bash
# Méthode 1: DNS (lent, TTL)
# Changer A record: primary.example.com → replica1_ip

# Méthode 2: Virtual IP (rapide)
# Déplacer VIP vers replica1
ip addr add 192.168.1.100/24 dev eth0  # Sur replica1
arping -c 3 -S 192.168.1.100 -i eth0  # Annonce ARP

# Méthode 3: HAProxy/ProxySQL (instantané)
# Reconfigurer backend
```

**Étape 5 : Validation**

```sql
-- Sur nouveau Primary
-- Tester écriture
CREATE DATABASE failover_test;
USE failover_test;
CREATE TABLE test (id INT, ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
INSERT INTO test (id) VALUES (1);

-- Sur Replicas: Vérifier réplication
SELECT * FROM failover_test.test;
-- Doit afficher la ligne
```

### Procédure avec GTID

**Avantage décisif** : Positions calculées automatiquement

```sql
-- Sur les Replicas (Replica2, Replica3)
-- Reconfiguration triviale avec GTID:

STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST = 'new-primary.example.com',
  MASTER_USE_GTID = slave_pos;  -- Magic ! ✨

START SLAVE;

-- MariaDB calcule automatiquement:
-- "Je suis à 0-1-999, je demande à partir de 0-1-1000"
-- Zero calcul de position manuel !
```

**Sans GTID** : Procédure complexe et error-prone

```bash
# 1. Sur nouveau Primary: Trouver position équivalente
# Analyser binlog, comparer événements, calculer offset
# → Complexe, long, risque d'erreur

# 2. Configuration manuelle
CHANGE MASTER TO
  MASTER_HOST = 'new-primary.example.com',
  MASTER_LOG_FILE = 'new-primary-bin.000001',  # Quel fichier ?
  MASTER_LOG_POS = 12345;  # Quelle position ? ⚠️

# Risque: Mauvaise position → Skip ou duplicate transactions
```

### Gestion du Primary en panne

```sql
-- Lorsque l'ancien Primary revient (après réparation)

-- Option A: Le transformer en Replica
-- Sur ancien Primary (maintenant Replica)
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;

CHANGE MASTER TO
  MASTER_HOST = 'new-primary.example.com',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;

START SLAVE;

-- Option B: Le réinitialiser complètement
-- (Si données corrompues)
# Dump depuis nouveau Primary
mysqldump ... > full_dump.sql
# Restaurer sur ancien Primary
mariadb < full_dump.sql
# Puis configurer comme Replica (Option A)
```

---

## Switchover (Basculement Planifié)

### Scénario type

```
Timeline d'un switchover:

T0: Planification
    - Objectif: Migrer vers nouveau serveur
    - Fenêtre de maintenance: Dimanche 2h-4h

T1: 01:55 - Préparation
    - Vérification Replicas synchronisés
    - Tests pré-migration

T2: 02:00 - Début switchover
    - Applications en mode read-only
    - Drain écritures

T3: 02:02 - Synchronisation finale
    - Attente lag = 0 sur tous Replicas

T4: 02:03 - Basculement
    - Promotion nouveau Primary
    - Reconfiguration

T5: 02:05 - Applications rebasculent
    - Mode read-write restauré
    - Validation

Downtime: ~5 minutes (lecture seule: 3 min, full down: 2 min)
Data loss: ZERO (planifié, synchronisation validée)
```

### Procédure de switchover

**Étape 1 : Pré-vérifications**

```sql
-- Sur le Primary actuel
-- Vérifier santé globale
SELECT 
  @@hostname AS primary,
  COUNT(*) AS active_connections
FROM information_schema.PROCESSLIST
WHERE command != 'Sleep';

-- Vérifier binlog actif
SELECT @@log_bin, @@binlog_format;

-- Sur chaque Replica
-- Vérifier réplication saine
SELECT 
  @@hostname AS replica,
  Slave_IO_Running,
  Slave_SQL_Running,
  Seconds_Behind_Master,
  Gtid_Slave_Pos
FROM information_schema.SLAVE_STATUS;

-- Tous doivent être:
-- IO/SQL: Yes/Yes
-- Lag: 0 ou très faible (< 5s)
```

**Étape 2 : Mettre applications en read-only**

```sql
-- Sur Primary actuel
-- Bloquer nouvelles écritures (sauf SUPER)
SET GLOBAL read_only = ON;

-- Attendre que transactions en cours se terminent
SELECT COUNT(*) FROM information_schema.PROCESSLIST 
WHERE command != 'Sleep' AND user != 'root';
-- Attendre que COUNT = 0

-- Vérifier plus aucune écriture
SHOW MASTER STATUS;
-- Noter la position finale
```

**Alternative application-level** :

```bash
# Mettre load balancer en mode maintenance
haproxy -sf $(cat /var/run/haproxy.pid)

# Ou via API application
curl -X POST https://app.example.com/api/maintenance/enable
```

**Étape 3 : Synchronisation finale**

```sql
-- Sur tous les Replicas
-- Attendre synchronisation complète
SELECT 
  @@hostname,
  Seconds_Behind_Master
FROM information_schema.SLAVE_STATUS;

-- Attendre que TOUS affichent: 0

-- Script d'attente automatique
DELIMITER //
CREATE PROCEDURE wait_for_sync(max_wait_sec INT)
BEGIN
  DECLARE lag INT DEFAULT 999;
  DECLARE elapsed INT DEFAULT 0;
  
  WHILE lag > 0 AND elapsed < max_wait_sec DO
    SELECT IFNULL(Seconds_Behind_Master, 0) INTO lag
    FROM information_schema.SLAVE_STATUS;
    
    IF lag > 0 THEN
      DO SLEEP(1);
      SET elapsed = elapsed + 1;
    END IF;
  END WHILE;
  
  IF lag > 0 THEN
    SIGNAL SQLSTATE '45000' 
      SET MESSAGE_TEXT = 'Timeout waiting for replication sync';
  END IF;
END//
DELIMITER ;

-- Utilisation
CALL wait_for_sync(300);  -- Attendre max 5 minutes
```

**Étape 4 : Vérification positions identiques**

```sql
-- Sur Primary actuel
SELECT 
  @@gtid_binlog_pos AS primary_gtid,
  @@gtid_current_pos AS primary_current;

-- Sur Replica cible (futur Primary)
SELECT 
  @@gtid_slave_pos AS replica_gtid,
  @@gtid_current_pos AS replica_current;

-- Les positions GTID doivent être identiques !
-- Si différentes: ATTENDRE ou INVESTIGUER
```

**Étape 5 : Promotion du nouveau Primary**

```sql
-- Sur Replica cible (nouveau Primary)

-- 1. Arrêter réplication
STOP SLAVE;

-- 2. Vérifier position finale
SELECT @@gtid_slave_pos;

-- 3. Réinitialiser statut esclave
RESET SLAVE ALL;

-- 4. Activer écritures
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;

-- 5. (Optionnel) Flush logs
FLUSH LOGS;

-- Nouveau Primary prêt ✓
```

**Étape 6 : Reconfiguration topologie**

```sql
-- Sur ancien Primary (devient Replica)
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;

CHANGE MASTER TO
  MASTER_HOST = 'new-primary.example.com',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;

START SLAVE;

-- Sur autres Replicas
STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST = 'new-primary.example.com',
  MASTER_USE_GTID = slave_pos;

START SLAVE;
```

**Étape 7 : Basculement applications**

```bash
# Mise à jour configuration application
# Option 1: Variable d'environnement
export DB_HOST=new-primary.example.com

# Option 2: Fichier config
sed -i 's/old-primary/new-primary/g' /etc/app/database.conf

# Option 3: DNS (avec TTL court)
# Changer A record primary.example.com

# Option 4: Virtual IP (instantané)
# Déplacer VIP vers nouveau Primary

# Redémarrer applications ou reload config
systemctl reload application
```

**Étape 8 : Validation post-switchover**

```sql
-- Sur nouveau Primary
-- Test écriture
INSERT INTO switchover_log (event, timestamp) 
VALUES ('Switchover completed', NOW());

-- Vérifier connexions
SELECT COUNT(*) FROM information_schema.PROCESSLIST;

-- Sur Replicas (y compris ancien Primary)
-- Vérifier réplication
SHOW SLAVE STATUS\G
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0

-- Vérifier donnée test répliquée
SELECT * FROM switchover_log 
WHERE event = 'Switchover completed';
```

**Étape 9 : Désactiver mode maintenance**

```bash
# Applications
curl -X POST https://app.example.com/api/maintenance/disable

# Load balancer
# Remettre en production

# Monitoring
# Vérifier métriques normales
```

---

## Automatisation du Failover

### Orchestrator

**Orchestrator** : Outil open-source de gestion de topologie et failover automatique

**Installation** :

```bash
# Télécharger
wget https://github.com/openark/orchestrator/releases/download/v3.2.6/orchestrator-3.2.6-1.x86_64.rpm

# Installer
rpm -i orchestrator-3.2.6-1.x86_64.rpm

# Créer base de données
mysql -e "CREATE DATABASE IF NOT EXISTS orchestrator"
mysql orchestrator < /usr/share/orchestrator/orchestrator-schema.sql

# Créer utilisateur
mysql -e "
  CREATE USER 'orchestrator'@'%' IDENTIFIED BY 'OrchP@ss';
  GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD 
    ON *.* TO 'orchestrator'@'%';
  GRANT ALL ON orchestrator.* TO 'orchestrator'@'%';
"
```

**Configuration** :

```json
// /etc/orchestrator.conf.json
{
  "Debug": false,
  "MySQLTopologyUser": "orchestrator",
  "MySQLTopologyPassword": "OrchP@ss",
  "MySQLOrchestratorHost": "localhost",
  "MySQLOrchestratorPort": 3306,
  "MySQLOrchestratorDatabase": "orchestrator",
  "MySQLOrchestratorUser": "orchestrator",
  "MySQLOrchestratorPassword": "OrchP@ss",
  
  "DiscoverByShowSlaveHosts": true,
  "InstancePollSeconds": 5,
  "ReadOnly": false,
  
  "RecoveryPeriodBlockSeconds": 300,
  "RecoverMasterClusterFilters": [
    ".*"
  ],
  "RecoverIntermediateMasterClusterFilters": [
    ".*"
  ],
  
  "OnFailureDetectionProcesses": [
    "echo 'Detected {failureType} on {failureCluster}' >> /tmp/orchestrator-failure.log"
  ],
  
  "PreFailoverProcesses": [
    "echo 'Will recover from {failureType} on {failureCluster}' >> /tmp/orchestrator-recovery.log"
  ],
  
  "PostFailoverProcesses": [
    "/usr/local/bin/notify-failover.sh {failureCluster} {newMaster}"
  ],
  
  "HTTPAdvertise": "http://orchestrator.example.com:3000"
}
```

**Démarrage** :

```bash
systemctl start orchestrator
systemctl enable orchestrator

# Web UI
http://orchestrator.example.com:3000
```

**Découverte de topologie** :

```bash
# CLI
orchestrator-client -c discover -i primary.example.com:3306

# Ou via Web UI: Cluster → Discover
```

**Failover automatique** :

```
Orchestrator détecte Primary down:
1. Polling échoue 3 fois consécutives
2. Vérifie que Replicas confirment (quorum)
3. Identifie meilleur Replica (GTID position)
4. Exécute PreFailoverProcesses
5. Promote Replica
6. Reconfigure topologie
7. Exécute PostFailoverProcesses
8. Notifie (Slack, PagerDuty, etc.)

Durée totale: 30-60 secondes
```

### MHA (Master High Availability)

**MHA** : Alternative pour failover automatique

```bash
# Installation (Perl-based)
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager

wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.noarch.rpm
rpm -i mha4mysql-manager-0.58-0.el7.noarch.rpm

# Configuration
cat > /etc/mha/app1.cnf <<EOF
[server default]
user=mha
password=MhaP@ss
ssh_user=root
repl_user=repl_user
repl_password=ReplP@ss

master_binlog_dir=/var/log/mysql
remote_workdir=/tmp

[server1]
hostname=primary.example.com
candidate_master=1

[server2]
hostname=replica1.example.com
candidate_master=1

[server3]
hostname=replica2.example.com
no_master=1
EOF

# Lancer MHA Manager
nohup masterha_manager --conf=/etc/mha/app1.cnf &
```

### MaxScale Auto-Failover

**MaxScale** avec module `mariadbmon` :

```ini
# /etc/maxscale.cnf
[mariadbmon]
type=monitor
module=mariadbmon
servers=primary,replica1,replica2
user=maxscale
password=MaxScaleP@ss

auto_failover=true
auto_rejoin=true

failcount=3
failover_timeout=90s

[Read-Write-Service]
type=service
router=readwritesplit
servers=primary,replica1,replica2
user=maxscale
password=MaxScaleP@ss

master_failure_mode=fail_on_write
```

```bash
# Démarrer MaxScale
systemctl start maxscale

# Monitorer
maxctrl list servers
```

---

## Tests de Failover

### Importance des tests

```
Test régulier de failover:
✅ Valide procédures
✅ Forme équipes
✅ Identifie gaps
✅ Mesure RTO/RPO réels
✅ Réduit stress incidents

Sans tests:
❌ Procédures obsolètes
❌ Équipes non préparées
❌ Découverte problèmes en prod
❌ RTO/RPO inconnus
❌ Panique lors incidents
```

### Planification des tests

**Fréquence recommandée** :

| Type | Fréquence | Durée |
|------|-----------|-------|
| Failover automatisé | Mensuel | 30 min |
| Switchover planifié | Trimestriel | 1 heure |
| Disaster recovery complet | Annuel | 4 heures |

**Environnements de test** :

```
Niveau 1: Lab isolé
- Topologie réplique production
- Zero risque
- Tests fréquents

Niveau 2: Staging
- Proche production
- Traffic synthétique
- Tests trimestriels

Niveau 3: Production (fenêtre maintenance)
- Réel
- Risque contrôlé
- Tests annuels
```

### Script de test automatisé

```bash
#!/bin/bash
# test_failover.sh

set -e

PRIMARY="primary.example.com"
REPLICA1="replica1.example.com"
REPLICA2="replica2.example.com"

echo "=== Failover Test ==="
echo "Date: $(date)"
echo ""

# Phase 1: État initial
echo "[1/6] Vérification état initial..."
mysql -h $PRIMARY -e "SELECT @@hostname, @@read_only"
mysql -h $REPLICA1 -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running"

# Phase 2: Insérer données test
echo "[2/6] Insertion données test..."
mysql -h $PRIMARY -e "
  CREATE DATABASE IF NOT EXISTS failover_test;
  USE failover_test;
  CREATE TABLE IF NOT EXISTS test_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event VARCHAR(255),
    ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  INSERT INTO test_log (event) VALUES ('Before failover');
"

# Phase 3: Simuler crash Primary
echo "[3/6] Simulation crash Primary..."
ssh root@$PRIMARY "systemctl stop mariadb"
sleep 5

# Phase 4: Promouvoir Replica1
echo "[4/6] Promotion Replica1..."
mysql -h $REPLICA1 -e "
  STOP SLAVE;
  RESET SLAVE ALL;
  SET GLOBAL read_only = OFF;
  SET GLOBAL super_read_only = OFF;
"

# Phase 5: Reconfigurer Replica2
echo "[5/6] Reconfiguration Replica2..."
mysql -h $REPLICA2 -e "
  STOP SLAVE;
  CHANGE MASTER TO
    MASTER_HOST = '$REPLICA1',
    MASTER_USE_GTID = slave_pos;
  START SLAVE;
"

# Phase 6: Validation
echo "[6/6] Validation..."
mysql -h $REPLICA1 -e "
  USE failover_test;
  INSERT INTO test_log (event) VALUES ('After failover - new primary');
"

sleep 2

mysql -h $REPLICA2 -e "
  SELECT COUNT(*) AS replicated_events 
  FROM failover_test.test_log 
  WHERE event LIKE '%failover%';
"

echo ""
echo "✅ Failover test completed successfully"
echo "New Primary: $REPLICA1"
echo "Replicas: $REPLICA2"

# Cleanup (optionnel)
echo ""
echo "Cleanup: Redémarrer ancien Primary comme Replica..."
ssh root@$PRIMARY "systemctl start mariadb"
sleep 5
mysql -h $PRIMARY -e "
  SET GLOBAL read_only = ON;
  CHANGE MASTER TO
    MASTER_HOST = '$REPLICA1',
    MASTER_USE_GTID = slave_pos;
  START SLAVE;
"

echo "✅ Cleanup completed"
```

### Checklist de test

```
Pré-test:
☐ Backup de toute la topologie
☐ Documentation procédures à jour
☐ Équipe complète disponible
☐ Monitoring actif
☐ Fenêtre de temps allouée

Pendant le test:
☐ Chronométrer chaque étape
☐ Logger toutes les commandes
☐ Capturer erreurs/warnings
☐ Mesurer RTO effectif
☐ Mesurer RPO effectif

Post-test:
☐ Vérifier intégrité données
☐ Vérifier performance
☐ Analyser métriques
☐ Documenter leçons apprises
☐ Mettre à jour runbooks
```

---

## Bonnes Pratiques

### 1. GTID obligatoire

```sql
-- Activer GTID sur TOUTE la topologie
[mysqld]
gtid_domain_id = 0
gtid_strict_mode = ON
log_slave_updates = ON  -- Si cascade

-- Simplifie drastiquement failover/switchover
```

### 2. Réplication semi-synchrone pour RPO=0

```sql
-- Sur Primary
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;

-- Sur Replicas
SET GLOBAL rpl_semi_sync_slave_enabled = ON;

-- Garantie: Zero data loss en failover
```

### 3. Identifier Replica candidat

```sql
-- Taguer les Replicas éligibles
CREATE TABLE replication_config (
  server_id INT PRIMARY KEY,
  hostname VARCHAR(255),
  role ENUM('primary', 'replica'),
  failover_candidate BOOLEAN,
  priority INT,  -- 1=highest
  notes TEXT
);

INSERT INTO replication_config VALUES
(1, 'primary.example.com', 'primary', FALSE, 0, 'Production primary'),
(2, 'replica1.example.com', 'replica', TRUE, 1, 'Hot standby - promote first'),
(3, 'replica2.example.com', 'replica', TRUE, 2, 'Fallback'),
(4, 'replica3.example.com', 'replica', FALSE, 0, 'Reporting only - never promote');
```

### 4. Automatiser reconfiguration applications

```bash
#!/bin/bash
# update_app_config.sh <new_primary_host>

NEW_PRIMARY=$1

# Mise à jour fichiers config
for SERVER in app1 app2 app3; do
  ssh $SERVER "
    sed -i 's/^DB_HOST=.*/DB_HOST=$NEW_PRIMARY/' /etc/app/database.env
    systemctl reload app
  "
done

# Mise à jour DNS (si utilisé)
# aws route53 change-resource-record-sets ...

# Notification
curl -X POST $SLACK_WEBHOOK -d "{\"text\":\"Database primary updated to $NEW_PRIMARY\"}"
```

### 5. Virtual IP pour transparence

```bash
# Keepalived configuration
# /etc/keepalived/keepalived.conf

vrrp_script check_mariadb {
  script "/usr/local/bin/check_mariadb.sh"
  interval 2
  weight -20
}

vrrp_instance VI_1 {
  interface eth0
  state MASTER  # Sur Primary
  virtual_router_id 51
  priority 100  # Plus haut = preferred
  
  virtual_ipaddress {
    192.168.1.100/24
  }
  
  track_script {
    check_mariadb
  }
  
  notify_master "/usr/local/bin/notify_master.sh"
}

# Script de check
cat > /usr/local/bin/check_mariadb.sh <<'EOF'
#!/bin/bash
mysql -e "SELECT 1" > /dev/null 2>&1
exit $?
EOF

chmod +x /usr/local/bin/check_mariadb.sh
```

### 6. Documenter runbooks

```markdown
# Runbook: Failover d'Urgence

## Détection
- Alerte: "Primary DOWN"
- Vérifier: `ping primary.example.com`
- Confirmer: Impossible de se connecter au Primary

## Décision
- Si < 5 minutes downtime attendu: ATTENDRE
- Si > 5 minutes ou hardware failure: FAILOVER

## Exécution
1. Identifier meilleur Replica
   ```sql
   SELECT @@gtid_slave_pos FROM replica1;
   ```

2. Promouvoir Replica
   ```sql
   STOP SLAVE; RESET SLAVE ALL;
   SET GLOBAL read_only = OFF;
   ```

3. Reconfigurer topologie
   [...]

## Validation
- Test écriture sur nouveau Primary
- Vérifier réplication autres Replicas
- Basculer applications

## Communication
- Notifier équipes
- Mettre à jour documentation
- Post-mortem dans les 48h
```

### 7. Monitoring post-failover

```sql
-- Créer table de suivi
CREATE TABLE failover_history (
  id INT AUTO_INCREMENT PRIMARY KEY,
  event_type ENUM('failover', 'switchover'),
  old_primary VARCHAR(255),
  new_primary VARCHAR(255),
  initiated_by VARCHAR(255),
  reason TEXT,
  rto_seconds INT,  -- Temps downtime effectif
  rpo_seconds INT,  -- Données perdues estimées
  success BOOLEAN,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Logger chaque événement
INSERT INTO failover_history 
  (event_type, old_primary, new_primary, initiated_by, reason, rto_seconds, rpo_seconds, success) 
VALUES 
  ('failover', 'primary.example.com', 'replica1.example.com', 'dba@example.com', 
   'Hardware failure', 120, 2, TRUE);
```

---

## ✅ Points clés à retenir

1. **Failover vs Switchover** : Non planifié vs planifié, urgence vs contrôle

2. **RTO/RPO** : Définir et mesurer objectifs de récupération

3. **GTID essentiel** : Simplifie drastiquement les basculements

4. **Semi-sync** : RPO=0 pour données critiques

5. **Automatisation** : Orchestrator, MHA, MaxScale pour failover rapide

6. **Tests réguliers** : Valider procédures, former équipes, mesurer métriques

7. **Meilleur Replica** : Identifier par GTID position (le plus à jour)

8. **Applications** : Prévoir reconfiguration automatisée

9. **Virtual IP** : Transparence basculement pour applications

10. **Documentation** : Runbooks détaillés et à jour, post-mortem systématique

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Replication Failover](https://mariadb.com/kb/en/replication-failover/)
- [📖 GTID and Failover](https://mariadb.com/kb/en/gtid/#using-gtid-for-failover)

### Outils

- **Orchestrator** : github.com/openark/orchestrator
- **MHA** : github.com/yoshinorim/mha4mysql-manager
- **MaxScale** : mariadb.com/products/skysql/mariadb-maxscale/

### Articles techniques

- [🔗 Failover Best Practices](https://mariadb.com/resources/blog/replication-failover-best-practices/)
- [🔗 Zero Downtime Migrations](https://www.percona.com/blog/zero-downtime-mysql-migrations/)

---

## ➡️ Section suivante

**13.9 Réplication semi-synchrone** : Configuration détaillée de la réplication semi-synchrone, modes AFTER_SYNC vs AFTER_COMMIT, impact performance, monitoring spécifique, et garanties de durabilité.

---


⏭️ [Réplication semi-synchrone](/13-replication/09-replication-semi-synchrone.md)

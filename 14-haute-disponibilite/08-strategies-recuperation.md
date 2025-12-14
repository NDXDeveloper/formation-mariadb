🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.8 Stratégies de Récupération après Incident

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Sections 14.1-14.7, expérience opérationnelle en gestion d'incidents

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Concevoir** une stratégie complète de disaster recovery (DR)
- **Définir** des objectifs RPO/RTO réalistes et mesurables
- **Documenter** des procédures de récupération testées et validées
- **Réagir** efficacement aux différents types d'incidents
- **Tester** régulièrement vos plans de récupération
- **Conduire** des post-mortems constructifs
- **Améliorer** continuellement votre résilience
- **Communiquer** efficacement durant les crises

---

## Introduction

Le **disaster recovery (DR)** n'est pas une question de "si" mais de "quand". Tout système, même parfaitement conçu, finira par rencontrer un incident. Ce qui distingue les organisations matures n'est pas l'absence d'incidents, mais leur capacité à récupérer rapidement et à apprendre de chaque échec.

**Statistiques industrie** :
```
93% des entreprises sans plan DR qui subissent une perte de données 
    majeure font faillite dans les 5 ans (Source: National Archives)

70% des petites entreprises ferment dans l'année suivant un désastre majeur
    (Source: FEMA)

Coût moyen downtime : 5,600$ par minute (Source: Gartner 2024)
    = 336,000$ par heure
    = 8M$ par jour
```

> ⚠️ **Réalité Brutale** : "Tout le monde a un plan jusqu'à ce qu'ils se prennent un coup dans la figure." - Mike Tyson (adaptable au DR)

**Principe fondamental** :
```
┌───────────────────────────────────────────────────┐
│  Un plan DR non testé = Pas de plan DR            │
│                                                   │
│  Tester 1× par an = Insuffisant                   │
│  Tester 1× par trimestre = Minimum                │
│  Tester 1× par mois = Recommandé                  │
│  Chaos engineering continu = Optimal              │
└───────────────────────────────────────────────────┘
```

---

## 1. Planification RPO/RTO

### 1.1 Définitions et Calculs

#### **RTO (Recovery Time Objective)**

```
RTO = Temps maximum acceptable de downtime

Calcul du coût RTO :
┌─────────────────────────────────────────────────┐
│  Exemple E-commerce                             │
│  Revenue moyen : 10,000€/heure                  │
│                                                 │
│  RTO 1 heure  → Perte max : 10,000€             │
│  RTO 4 heures → Perte max : 40,000€             │
│  RTO 1 jour   → Perte max : 240,000€            │
│                                                 │
│  + Coûts indirects :                            │
│    - Perte de confiance clients                 │
│    - Pénalités SLA                              │
│    - Coûts équipe d'urgence                     │
│    - Image de marque                            │
└─────────────────────────────────────────────────┘
```

**Composantes du RTO** :
```
RTO Total = Détection + Décision + Récupération + Validation

Exemple breakdown :
┌──────────────────────┬──────────┬─────────────┐
│ Phase                │ Durée    │ Optimisable │
├──────────────────────┼──────────┼─────────────┤
│ Détection incident   │ 5 min    │ ✅ Alerting │
│ Analyse initiale     │ 10 min   │ ✅ Runbooks │
│ Décision recovery    │ 5 min    │ ✅ Auto     │
│ Execution failover   │ 2 min    │ ✅ Auto     │
│ Validation service   │ 3 min    │ ⚠️ Manuel   │
│ Communication        │ 5 min    │ ⚠️ Manuel   │
├──────────────────────┼──────────┼─────────────┤
│ TOTAL                │ 30 min   │             │
└──────────────────────┴──────────┴─────────────┘

Objectif : RTO < 30 minutes
```

#### **RPO (Recovery Point Objective)**

```
RPO = Perte de données maximum acceptable

Calcul du coût RPO :
┌─────────────────────────────────────────────────┐
│  Exemple Banking                                │
│  Transactions moyennes : 1,000/minute           │
│  Valeur moyenne transaction : 500€              │
│                                                 │
│  RPO 1 minute  → Perte : 1,000 tx (500,000€)    │
│  RPO 5 minutes → Perte : 5,000 tx (2,500,000€)  │
│  RPO 1 heure   → Perte : 60,000 tx (30M€)       │
│                                                 │
│  + Coûts de reconstitution manuelle             │
│  + Implications légales/réglementaires          │
└─────────────────────────────────────────────────┘
```

**Technologies par RPO** :
```
┌──────────────┬───────────────────┬──────────────────────┐
│ RPO Objectif │ Solution          │ Coût Relatif         │
├──────────────┼───────────────────┼──────────────────────┤
│ 0 seconde    │ Galera Cluster    │ €€€€€ (très élevé)   │
│              │ Synchronous Rep   │                      │
├──────────────┼───────────────────┼──────────────────────┤
│ < 1 minute   │ Semi-Sync Rep     │ €€€€ (élevé)         │
│              │ DRBD              │                      │
├──────────────┼───────────────────┼──────────────────────┤
│ < 5 minutes  │ Async Rep         │ €€€ (moyen)          │
│              │ + Monitoring      │                      │
├──────────────┼───────────────────┼──────────────────────┤
│ < 1 heure    │ Async Rep         │ €€ (faible)          │
│              │ + Backup incr.    │                      │
├──────────────┼───────────────────┼──────────────────────┤
│ < 24 heures  │ Backup quotidien  │ € (très faible)      │
└──────────────┴───────────────────┴──────────────────────┘
```

### 1.2 Matrice RPO/RTO par Type d'Application

```
┌─────────────────────────────────────────────────────────┐
│  Type Application    │ RPO Typique  │ RTO Typique       │
├──────────────────────┼──────────────┼───────────────────┤
│  Banking (core)      │ 0 sec        │ < 1 min           │
│  Trading             │ 0 sec        │ < 30 sec          │
│  E-commerce          │ < 5 min      │ < 15 min          │
│  SaaS B2B            │ < 15 min     │ < 1 heure         │
│  CMS/Blog            │ < 1 heure    │ < 4 heures        │
│  Internal tools      │ < 4 heures   │ < 1 jour          │
│  Analytics/BI        │ < 24 heures  │ < 2 jours         │
└─────────────────────────────────────────────────────────┘
```

### 1.3 Framework de Décision RPO/RTO

```python
#!/usr/bin/env python3
# rpo_rto_calculator.py

class RPORTOCalculator:
    """Calculateur RPO/RTO basé sur impact business"""
    
    def __init__(self):
        self.revenue_per_hour = 0
        self.transactions_per_minute = 0
        self.avg_transaction_value = 0
        self.reputation_cost_multiplier = 1.5
        
    def calculate_rto_cost(self, rto_hours):
        """Calcule le coût d'un RTO donné"""
        direct_cost = self.revenue_per_hour * rto_hours
        indirect_cost = direct_cost * (self.reputation_cost_multiplier - 1)
        total_cost = direct_cost + indirect_cost
        
        return {
            'rto_hours': rto_hours,
            'direct_cost': direct_cost,
            'indirect_cost': indirect_cost,
            'total_cost': total_cost
        }
    
    def calculate_rpo_cost(self, rpo_minutes):
        """Calcule le coût d'un RPO donné"""
        lost_transactions = self.transactions_per_minute * rpo_minutes
        data_loss_cost = lost_transactions * self.avg_transaction_value
        reconstitution_cost = lost_transactions * 10  # 10€ par transaction à recréer
        
        return {
            'rpo_minutes': rpo_minutes,
            'lost_transactions': lost_transactions,
            'data_loss_cost': data_loss_cost,
            'reconstitution_cost': reconstitution_cost,
            'total_cost': data_loss_cost + reconstitution_cost
        }
    
    def recommend_solution(self, max_budget):
        """Recommande une solution basée sur le budget"""
        solutions = [
            {
                'name': 'Galera Cluster 5 nodes + MaxScale HA',
                'rpo': 0,
                'rto_minutes': 2,
                'annual_cost': 150000,
                'availability': 99.99
            },
            {
                'name': 'Semi-Sync Replication + Auto-failover',
                'rpo': 5,
                'rto_minutes': 5,
                'annual_cost': 80000,
                'availability': 99.95
            },
            {
                'name': 'Async Replication + Manual failover',
                'rpo': 60,
                'rto_minutes': 30,
                'annual_cost': 40000,
                'availability': 99.9
            },
            {
                'name': 'Daily backups + DR site',
                'rpo': 1440,  # 24 hours
                'rto_minutes': 240,  # 4 hours
                'annual_cost': 15000,
                'availability': 99.5
            }
        ]
        
        affordable_solutions = [s for s in solutions if s['annual_cost'] <= max_budget]
        
        if not affordable_solutions:
            return None
        
        # Trier par meilleur RTO
        return sorted(affordable_solutions, key=lambda x: x['rto_minutes'])[0]

# Exemple d'utilisation
if __name__ == "__main__":
    calc = RPORTOCalculator()
    
    # E-commerce moyen
    calc.revenue_per_hour = 10000
    calc.transactions_per_minute = 50
    calc.avg_transaction_value = 80
    
    # Calculer coûts pour différents RTO
    for rto in [0.5, 1, 4, 24]:
        cost = calc.calculate_rto_cost(rto)
        print(f"\nRTO {rto}h: Coût total = {cost['total_cost']:,.0f}€")
    
    # Recommandation
    budget = 100000
    solution = calc.recommend_solution(budget)
    print(f"\n\nBudget: {budget:,}€")
    print(f"Solution recommandée: {solution['name']}")
    print(f"RPO: {solution['rpo']} min, RTO: {solution['rto_minutes']} min")
    print(f"Disponibilité: {solution['availability']}%")
```

---

## 2. Types d'Incidents et Procédures

### 2.1 Classification des Incidents

```
┌────────────────────────────────────────────────────────┐
│  Severité │ Définition              │ Exemples         │
├───────────┼─────────────────────────┼──────────────────┤
│  P0       │ Service complètement    │ - Cluster down   │
│  CRITICAL │ down, perte financière  │ - Data loss      │
│           │ importante              │ - Security breach│
├───────────┼─────────────────────────┼──────────────────┤
│  P1       │ Service dégradé,        │ - 1 node down    │
│  HIGH     │ impact utilisateurs     │ - High latency   │
│           │ significatif            │ - Repl lag       │
├───────────┼─────────────────────────┼──────────────────┤
│  P2       │ Impact limité,          │ - Slow queries   │
│  MEDIUM   │ workaround possible     │ - Disk 80% full  │
├───────────┼─────────────────────────┼──────────────────┤
│  P3       │ Impact minime,          │ - Log warnings   │
│  LOW      │ pas d'urgence           │ - Monitoring gap │
└────────────────────────────────────────────────────────┘

Temps de réponse requis :
P0 : Immédiat (< 15 minutes)
P1 : < 1 heure
P2 : < 4 heures
P3 : < 1 jour ouvré
```

### 2.2 Scénario 1 : Panne Matérielle (Serveur Down)

#### **Phase 1 : Détection et Alerte**

```bash
#!/bin/bash
# /usr/local/bin/detect_server_down.sh

SERVER="db-master.example.com"
ALERT_CHANNEL="#database-alerts"

# Tentative ping
if ! ping -c 3 -W 2 $SERVER &>/dev/null; then
    # Serveur inaccessible
    
    # Double-check via monitoring secondaire
    if ! curl -f http://monitoring-backup/check/$SERVER &>/dev/null; then
        # Confirmé down
        
        # Alert P0
        send_pagerduty_alert "P0: Database server $SERVER down" "critical"
        send_slack_alert "$ALERT_CHANNEL" "🚨 P0 INCIDENT: $SERVER DOWN"
        
        # Lancer procédure failover automatique
        if [ "$AUTO_FAILOVER_ENABLED" = "true" ]; then
            /usr/local/bin/trigger_failover.sh $SERVER
        else
            echo "Manual failover required - awaiting ops intervention"
        fi
    fi
fi
```

#### **Phase 2 : Procédure de Récupération**

```markdown
# RUNBOOK: Panne Serveur Master

## Détection
✅ Alert PagerDuty reçue
✅ Vérifier dashboard monitoring

## Validation
[ ] Confirmer serveur réellement down (pas faux positif)
    - Ping depuis 2+ sources différentes
    - Check console hyperviseur (si VM)
    - Check IPMI/iLO (si bare metal)

## Décision Failover
[ ] Vérifier state replicas
    ```bash
    for replica in replica1 replica2 replica3; do
        mysql -h $replica -e "SHOW SLAVE STATUS\G"
    done
    ```

[ ] Sélectionner best candidate
    - Lag minimal
    - GTID position la plus avancée
    - Ressources disponibles (CPU, RAM, Disk)

## Exécution Failover (automatique ou manuel)

### Automatique (mariadbmon)
[ ] Vérifier logs MaxScale
    ```bash
    tail -f /var/log/maxscale/maxscale.log | grep -i failover
    ```

[ ] Attendre promotion automatique (~60s)

### Manuel
[ ] Promouvoir replica sélectionnée
    ```bash
    REPLICA="replica1.example.com"
    
    # Sur la replica à promouvoir
    mysql -h $REPLICA << EOF
    STOP SLAVE;
    RESET SLAVE ALL;
    SET GLOBAL read_only = OFF;
    EOF
    ```

[ ] Reconfigurer autres replicas
    ```bash
    for other in replica2 replica3; do
        mysql -h $other << EOF
        STOP SLAVE;
        CHANGE MASTER TO 
            MASTER_HOST='$REPLICA',
            MASTER_USER='repl_user',
            MASTER_PASSWORD='ReplicationPassword',
            MASTER_USE_GTID=current_pos;
        START SLAVE;
        EOF
    done
    ```

[ ] Basculer VIP (si keepalived non auto)
    ```bash
    # Sur nouveau master
    systemctl stop keepalived
    systemctl start keepalived
    # Vérifier VIP assignée
    ip addr show | grep 10.0.1.100
    ```

## Validation
[ ] Tester connexion via VIP
    ```bash
    mysql -h 10.0.1.100 -u app -p -e "SELECT @@hostname, @@read_only"
    # hostname = nouveau master
    # read_only = OFF
    ```

[ ] Vérifier réplication autres replicas
    ```bash
    for replica in replica2 replica3; do
        mysql -h $replica -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
    done
    # Doit être < 10 secondes
    ```

[ ] Test applicatif
    ```bash
    # Test write
    mysql -h 10.0.1.100 -u app -p << EOF
    CREATE TABLE IF NOT EXISTS test_failover (
        id INT AUTO_INCREMENT PRIMARY KEY,
        timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    INSERT INTO test_failover VALUES ();
    SELECT * FROM test_failover ORDER BY id DESC LIMIT 1;
    EOF
    ```

## Communication
[ ] Update incident ticket
[ ] Notify stakeholders (Slack, Email)
[ ] Update status page

## Post-Incident (après stabilisation)
[ ] Investiguer cause panne master original
[ ] Planifier reconstruction master original
[ ] Scheduler post-mortem meeting
[ ] Documenter timeline incident

## RTO Visé: < 5 minutes
## Statut: [ ] Résolu  [ ] En cours  [ ] Escaladé
```

### 2.3 Scénario 2 : Corruption de Données

#### **Détection**

```sql
-- Symptômes typiques
-- 1. Erreurs de cohérence InnoDB
ERROR 1030 (HY000): Got error 168 from storage engine

-- 2. Inconsistencies détectées
CHECK TABLE orders;
-- Table is marked as crashed

-- 3. Réplication cassée
SHOW SLAVE STATUS\G
# Last_SQL_Error: Error 'Duplicate entry' on query

-- 4. Queries retournent résultats invalides
SELECT COUNT(*) FROM accounts WHERE balance < 0;
-- 52 rows (impossible normalement !)
```

#### **Procédure de Récupération**

```bash
#!/bin/bash
# /usr/local/bin/recover_from_corruption.sh

set -e

CORRUPTED_SERVER="db-master.example.com"
BACKUP_LOCATION="/mnt/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== Data Corruption Recovery ==="
echo "Server: $CORRUPTED_SERVER"
echo "Timestamp: $TIMESTAMP"

# 1. STOP WRITES IMMEDIATELY
echo "[1/8] Stopping writes..."
mysql -h $CORRUPTED_SERVER -e "SET GLOBAL read_only = ON"

# 2. Isoler serveur corrompu
echo "[2/8] Isolating corrupted server..."
# Retirer du pool MaxScale
maxctrl unlink service Read-Write-Service $CORRUPTED_SERVER
# Ou arrêter keepalived
ssh $CORRUPTED_SERVER "systemctl stop keepalived"

# 3. Snapshot état actuel (forensics)
echo "[3/8] Creating forensic snapshot..."
mysqldump -h $CORRUPTED_SERVER --all-databases \
    --single-transaction \
    --routines --triggers \
    --events \
    > /tmp/corrupted_snapshot_${TIMESTAMP}.sql

# 4. Identifier dernier backup sain
echo "[4/8] Identifying last good backup..."
LAST_BACKUP=$(ls -t $BACKUP_LOCATION/*.xbstream | head -1)
echo "Using backup: $LAST_BACKUP"

# 5. Restauration sur serveur temporaire
echo "[5/8] Restoring to temporary server..."
TEMP_SERVER="db-temp-recovery.example.com"

ssh $TEMP_SERVER << 'EOF'
    systemctl stop mariadb
    rm -rf /var/lib/mysql/*
    
    # Restore mariabackup
    mariabackup --decompress --remove-original \
        --target-dir=/var/lib/mysql
    mariabackup --prepare --target-dir=/var/lib/mysql
    
    chown -R mysql:mysql /var/lib/mysql
    systemctl start mariadb
EOF

# 6. Replay binlogs depuis backup jusqu'au point de corruption
echo "[6/8] Replaying binary logs..."
BACKUP_BINLOG_POS=$(ssh $TEMP_SERVER "mysql -N -e \"SELECT @@GLOBAL.gtid_binlog_pos\"")
CORRUPTION_TIME="2025-12-13 14:32:00"  # Time corruption detected

mysqlbinlog --start-position=$BACKUP_BINLOG_POS \
            --stop-datetime="$CORRUPTION_TIME" \
            /var/lib/mysql-binlogs/binlog.* | \
    mysql -h $TEMP_SERVER

# 7. Validation données restaurées
echo "[7/8] Validating restored data..."
ssh $TEMP_SERVER << 'EOF'
    # Check tables
    mysqlcheck --all-databases --check
    
    # Verify critical tables
    mysql -e "SELECT COUNT(*) FROM accounts WHERE balance < 0"
    # Doit retourner 0
    
    # Verify consistency
    mysql -e "
        SELECT 
            (SELECT SUM(amount) FROM transactions WHERE type='debit') as total_debit,
            (SELECT SUM(amount) FROM transactions WHERE type='credit') as total_credit,
            (SELECT SUM(balance) FROM accounts) as total_balance
    "
    # total_credit - total_debit = total_balance (must match)
EOF

# 8. Promouvoir serveur restauré
echo "[8/8] Promoting restored server as new master..."
# Point de décision : restaurer sur serveur original ou promouvoir temp ?

# Option A: Restaurer sur serveur original
# - Arrêter serveur original
# - Rsync depuis temp vers original
# - Redémarrer original

# Option B: Promouvoir temp (plus rapide)
# - Reconfigurer replicas vers temp
# - Basculer VIP vers temp
# - Décommissionner original

echo "=== Recovery Complete ==="
echo "Next steps:"
echo "1. Investigate root cause of corruption"
echo "2. Review binlogs for unauthorized changes"
echo "3. Update monitoring to detect earlier"
echo "4. Schedule post-mortem"
```

### 2.4 Scénario 3 : Disaster Complet (Datacenter Down)

#### **Architecture Multi-DC Required**

```
Primary DC (Paris)          Disaster Recovery DC (Londres)
┌───────────────────┐       ┌───────────────────┐
│  Galera Node 1    │───────│  Galera Node 4    │
│  Galera Node 2    │       │  Galera Node 5    │
│  Galera Node 3    │       │                   │
│  MaxScale 1       │       │  MaxScale 2       │
└───────────────────┘       └───────────────────┘
         │                           │
         └──────── Garbd (Dublin) ───┘
                   (Arbitrator)
```

#### **Procédure DR Complète**

```bash
#!/bin/bash
# /usr/local/bin/dr_failover_complete.sh

# SCENARIO: Primary DC (Paris) complètement inaccessible

echo "=== DISASTER RECOVERY PROCEDURE ==="
echo "PRIMARY DC DOWN - Activating DR site"

DR_SITE="london"
DR_NODES=("db-london-1" "db-london-2")
DR_MAXSCALE="maxscale-london"

# 1. Validation DR site opérationnel
echo "[1/6] Validating DR site..."
for node in "${DR_NODES[@]}"; do
    if ! mysql -h $node -e "SELECT 1" &>/dev/null; then
        echo "ERROR: DR node $node not accessible"
        exit 1
    fi
done

# 2. Vérifier état Galera DR
echo "[2/6] Checking Galera cluster state..."
CLUSTER_SIZE=$(mysql -h ${DR_NODES[0]} -N -e \
    "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_cluster_size'")

if [ "$CLUSTER_SIZE" -lt 2 ]; then
    echo "WARNING: Only $CLUSTER_SIZE nodes in DR cluster"
fi

# 3. Bootstrap DR cluster si nécessaire
echo "[3/6] Ensuring DR cluster is PRIMARY..."
CLUSTER_STATUS=$(mysql -h ${DR_NODES[0]} -N -e \
    "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_cluster_status'")

if [ "$CLUSTER_STATUS" != "Primary" ]; then
    echo "Bootstrapping DR cluster..."
    # Sur le nœud le plus à jour
    ssh ${DR_NODES[0]} "
        mysql -e \"SET GLOBAL wsrep_provider_options='pc.bootstrap=true'\"
    "
fi

# 4. Activer écritures sur DR
echo "[4/6] Enabling writes on DR cluster..."
for node in "${DR_NODES[@]}"; do
    mysql -h $node -e "SET GLOBAL read_only = OFF"
done

# 5. Basculer DNS/VIP vers DR
echo "[5/6] Updating DNS to point to DR site..."

# Option A: DNS (propagation lente, 5-60 minutes)
aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890ABC \
    --change-batch "{
        \"Changes\": [{
            \"Action\": \"UPSERT\",
            \"ResourceRecordSet\": {
                \"Name\": \"db.example.com\",
                \"Type\": \"CNAME\",
                \"TTL\": 60,
                \"ResourceRecords\": [{
                    \"Value\": \"db-london.example.com\"
                }]
            }
        }]
    }"

# Option B: Global Load Balancer (propagation rapide, ~30s)
# CloudFlare, AWS Global Accelerator, Azure Traffic Manager

# 6. Validation end-to-end
echo "[6/6] Validating DR activation..."

# Test écriture
mysql -h db.example.com -u app -p << EOF
CREATE TABLE IF NOT EXISTS dr_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    activated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO dr_test VALUES ();
EOF

# Vérifier réplication intra-DR
for node in "${DR_NODES[@]}"; do
    COUNT=$(mysql -h $node -N -e "SELECT COUNT(*) FROM dr_test")
    echo "Node $node: $COUNT rows in dr_test"
done

echo "=== DR ACTIVATION COMPLETE ==="
echo ""
echo "CRITICAL NEXT STEPS:"
echo "1. Notify all stakeholders (exec, customers, teams)"
echo "2. Update status page"
echo "3. Monitor DR cluster closely (alerts, dashboards)"
echo "4. Investigate primary DC status"
echo "5. Plan return to primary DC when available"
echo ""
echo "RTO Achieved: Check incident timeline"
echo "Expected service restoration: IMMEDIATE"
```

---

## 3. Tests et Validation

### 3.1 Programme de Tests DR

```
┌────────────────────────────────────────────────────────┐
│  Fréquence │ Type de Test             │ Durée          │
├────────────┼──────────────────────────┼────────────────┤
│  Mensuel   │ Failover automatique     │ 1 heure        │
│            │ (master → replica)       │                │
├────────────┼──────────────────────────┼────────────────┤
│  Trimes.   │ Restauration backup      │ 4 heures       │
│            │ complète                 │                │
├────────────┼──────────────────────────┼────────────────┤
│  Trimes.   │ Chaos engineering        │ 2 heures       │
│            │ (random node kill)       │                │
├────────────┼──────────────────────────┼────────────────┤
│  Semes.    │ DR complet               │ 8 heures       │
│            │ (datacenter failover)    │                │
├────────────┼──────────────────────────┼────────────────┤
│  Annuel    │ DR Full Drill            │ 1-2 jours      │
│            │ (simulation complète)    │                │
└────────────────────────────────────────────────────────┘
```

### 3.2 Checklist Test Failover Mensuel

```markdown
# DR Test Checklist - Failover Master

**Date**: _____________
**Participants**: _____________
**Environnement**: [ ] Production  [ ] Staging

## Pré-Test
- [ ] Notification équipes (24h avance)
- [ ] Backup complet < 24h
- [ ] Validation monitoring opérationnel
- [ ] Communication maintenance window (si prod)
- [ ] Rollback plan documenté

## Phase 1: Baseline (T-15min)
- [ ] Capturer métriques baseline
    - QPS: _______
    - Latency P99: _______
    - Replication lag: _______
- [ ] Vérifier état cluster
    ```bash
    for server in master replica1 replica2; do
        echo "=== $server ==="
        mysql -h $server -e "SHOW MASTER STATUS\G"
        mysql -h $server -e "SHOW SLAVE STATUS\G"
    done
    ```
- [ ] Screenshot dashboards

## Phase 2: Exécution Failover (T+0)
- [ ] Timestamp début: _____________
- [ ] Méthode:
    - [ ] Arrêt propre master: `systemctl stop mariadb`
    - [ ] Crash simulation: `kill -9 $(pidof mariadbd)`
    - [ ] Network partition: `iptables -A INPUT -p tcp --dport 3306 -j DROP`
    
- [ ] Observer failover automatique
    - Temps détection: _______ secondes
    - Temps promotion: _______ secondes
    - Nouveau master: _____________

## Phase 3: Validation (T+5min)
- [ ] Vérifier nouveau master
    ```bash
    mysql -h NEW_MASTER -e "SHOW MASTER STATUS\G"
    mysql -h NEW_MASTER -e "SELECT @@read_only"  # Must be OFF
    ```

- [ ] Vérifier replicas suivent nouveau master
    ```bash
    mysql -h replica1 -e "SHOW SLAVE STATUS\G" | grep Master_Host
    # Doit afficher NEW_MASTER
    ```

- [ ] Test écriture applicative
    ```bash
    mysql -h VIP -u app -p << EOF
    INSERT INTO test_failover (test_id, timestamp) 
    VALUES (UUID(), NOW());
    EOF
    ```

- [ ] Vérifier métriques post-failover
    - QPS: _______ (vs baseline: ______)
    - Latency P99: _______ (vs baseline: ______)
    - Errors: _______

## Phase 4: Récupération Ancien Master (T+10min)
- [ ] Redémarrer ancien master
    ```bash
    systemctl start mariadb
    ```

- [ ] Vérifier rejoint comme replica
    ```bash
    mysql -h OLD_MASTER -e "SHOW SLAVE STATUS\G"
    ```

- [ ] Attendre synchronisation complète
    ```bash
    while true; do
        LAG=$(mysql -h OLD_MASTER -N -e \
          "SELECT Seconds_Behind_Master FROM information_schema.SLAVE_STATUS")
        [ "$LAG" -eq 0 ] && break
        echo "Lag: ${LAG}s..."
        sleep 2
    done
    ```

## Phase 5: Cleanup (T+20min)
- [ ] Vérifier logs erreurs
    ```bash
    tail -100 /var/log/mysql/error.log | grep -i error
    ```

- [ ] Supprimer données test
    ```bash
    mysql -e "DELETE FROM test_failover WHERE test_id IS NOT NULL"
    ```

- [ ] Restaurer état initial (optionnel)
    - [ ] Switchover vers master original

## Résultats
- RTO Mesuré: _______ (objectif: < 5 min)
- RPO Mesuré: _______ (objectif: 0)
- Succès: [ ] Oui  [ ] Non
- Issues rencontrées:
    1. _____________
    2. _____________

## Actions Post-Test
- [ ] Documenter lessons learned
- [ ] Créer tickets pour issues
- [ ] Update runbooks si nécessaire
- [ ] Partager rapport avec équipe

**Responsable Test**: _____________
**Signature**: _____________
```

### 3.3 Chaos Engineering

```python
#!/usr/bin/env python3
# chaos_mariadb.py
"""
Chaos Engineering pour MariaDB
Simule pannes aléatoires pour valider résilience
"""

import random
import time
import subprocess
from datetime import datetime

class MariaDBChaos:
    def __init__(self, nodes):
        self.nodes = nodes
        self.incidents = []
        
    def random_node_kill(self):
        """Tuer un nœud aléatoire"""
        node = random.choice(self.nodes)
        print(f"[{datetime.now()}] Killing node: {node}")
        
        subprocess.run([
            "ssh", node, "systemctl stop mariadb"
        ])
        
        self.incidents.append({
            'timestamp': datetime.now(),
            'type': 'node_kill',
            'target': node
        })
        
        # Attendre 2 minutes
        time.sleep(120)
        
        # Redémarrer
        print(f"[{datetime.now()}] Restarting node: {node}")
        subprocess.run([
            "ssh", node, "systemctl start mariadb"
        ])
    
    def network_partition(self, duration=60):
        """Simuler partition réseau"""
        node = random.choice(self.nodes)
        print(f"[{datetime.now()}] Network partition: {node}")
        
        # Bloquer port 3306 et 4567 (Galera)
        subprocess.run([
            "ssh", node, 
            "iptables -A INPUT -p tcp --dport 3306 -j DROP; "
            "iptables -A INPUT -p tcp --dport 4567 -j DROP"
        ])
        
        time.sleep(duration)
        
        # Restaurer
        print(f"[{datetime.now()}] Restoring network: {node}")
        subprocess.run([
            "ssh", node,
            "iptables -D INPUT -p tcp --dport 3306 -j DROP; "
            "iptables -D INPUT -p tcp --dport 4567 -j DROP"
        ])
    
    def disk_full_simulation(self):
        """Simuler disque plein"""
        node = random.choice(self.nodes)
        print(f"[{datetime.now()}] Disk full simulation: {node}")
        
        # Créer fichier de 10GB
        subprocess.run([
            "ssh", node,
            "dd if=/dev/zero of=/tmp/fillup bs=1G count=10"
        ])
        
        time.sleep(300)  # 5 minutes
        
        # Cleanup
        subprocess.run([
            "ssh", node,
            "rm -f /tmp/fillup"
        ])
    
    def run_chaos_day(self, duration_hours=4):
        """Journée chaos : incidents aléatoires pendant N heures"""
        end_time = time.time() + (duration_hours * 3600)
        
        while time.time() < end_time:
            # Choisir type d'incident aléatoire
            incident_type = random.choice([
                'node_kill',
                'network_partition',
                'disk_full'
            ])
            
            if incident_type == 'node_kill':
                self.random_node_kill()
            elif incident_type == 'network_partition':
                self.network_partition()
            elif incident_type == 'disk_full':
                self.disk_full_simulation()
            
            # Pause entre incidents (15-45 minutes)
            pause = random.randint(900, 2700)
            print(f"Next incident in {pause/60:.1f} minutes...")
            time.sleep(pause)
        
        print("\n=== Chaos Day Complete ===")
        print(f"Total incidents: {len(self.incidents)}")
        for incident in self.incidents:
            print(f"  {incident['timestamp']}: {incident['type']} on {incident['target']}")

if __name__ == "__main__":
    nodes = ["db1.example.com", "db2.example.com", "db3.example.com"]
    chaos = MariaDBChaos(nodes)
    
    # Lancer 4h de chaos
    chaos.run_chaos_day(duration_hours=4)
```

---

## 4. Documentation et Runbooks

### 4.1 Structure Runbook

```markdown
# Runbook Template

## Métadonnées
- **Titre**: [Titre court descriptif]
- **ID**: RUNBOOK-XXX
- **Catégorie**: [Incident / Maintenance / Routine]
- **Severité**: [P0 / P1 / P2 / P3]
- **Auteur**: [Nom]
- **Dernière mise à jour**: [Date]
- **Version**: [X.Y]

## Résumé Exécutif
[1-2 paragraphes: Quoi, Pourquoi, Quand utiliser ce runbook]

## Signes et Symptômes
- [ ] Symptôme 1
- [ ] Symptôme 2
- [ ] Symptôme 3

## Prérequis
- Accès: [SSH, MySQL, AWS Console, etc.]
- Permissions: [sudo, DBA role, etc.]
- Outils: [Liste]

## RTO/RPO Visé
- RTO: [X minutes]
- RPO: [Y minutes]

## Procédure Étape par Étape

### Phase 1: Validation
**Durée estimée**: [X min]

1. [Étape 1]
   ```bash
   # Commande
   ```
   **Résultat attendu**: [...]

2. [Étape 2]
   ```bash
   # Commande
   ```
   **Résultat attendu**: [...]

### Phase 2: Exécution
**Durée estimée**: [Y min]

[...]

### Phase 3: Validation
**Durée estimée**: [Z min]

[...]

## Points de Décision
**Si [condition]:**
- [ ] Option A: [...]
- [ ] Option B: [...]

## Rollback
**Si échec, procédure de rollback:**

1. [Étape rollback 1]
2. [Étape rollback 2]

## Validation Succès
- [ ] Critère 1
- [ ] Critère 2
- [ ] Critère 3

## Communication
**Qui notifier:**
- [ ] Équipe: [#channel]
- [ ] Management: [emails]
- [ ] Clients: [status page]

**Template message:**
```
[Incident/Maintenance] [Titre]
Status: [In Progress / Resolved]
Impact: [Description]
ETA: [Estimation]
```

## Post-Action
- [ ] Documenter timeline
- [ ] Créer ticket post-mortem
- [ ] Update KB
- [ ] Update ce runbook

## Références
- [Doc 1]
- [Doc 2]

## Changelog
| Version | Date | Auteur | Modifications |
|---------|------|--------|---------------|
| 1.0 | 2025-01-01 | John Doe | Version initiale |
```

### 4.2 Runbook Repository

```bash
# Structure repo runbooks
runbooks/
├── README.md
├── templates/
│   ├── incident_runbook.md
│   └── maintenance_runbook.md
├── incidents/
│   ├── RB-001-master-failover.md
│   ├── RB-002-data-corruption.md
│   ├── RB-003-disk-full.md
│   ├── RB-004-replication-lag.md
│   └── RB-005-split-brain.md
├── maintenance/
│   ├── RB-M01-backup-restore.md
│   ├── RB-M02-upgrade-mariadb.md
│   └── RB-M03-add-replica.md
└── automation/
    ├── failover.sh
    ├── backup.sh
    └── health_check.sh
```

---

## 5. Post-Mortem et Amélioration Continue

### 5.1 Template Post-Mortem

```markdown
# Post-Mortem: [Incident Title]

**Date Incident**: 2025-12-13
**Severité**: P0
**Durée Total Downtime**: 15 minutes
**Participants Post-Mortem**: [Noms]

## Résumé Exécutif (pour non-techniques)
[2-3 paragraphes résumant l'incident, impact, et résolution]

## Timeline Détaillée

| Time (UTC) | Event | Actor | System State |
|------------|-------|-------|--------------|
| 14:32:00 | Master DB crash (OOM) | System | Master DOWN |
| 14:32:15 | Alert PagerDuty | Monitoring | - |
| 14:32:30 | DBA acknowledged | John Doe | - |
| 14:33:00 | Validated failure | John Doe | - |
| 14:34:00 | Initiated failover | John Doe | Failover started |
| 14:35:30 | Replica promoted | mariadbmon | New master UP |
| 14:36:00 | VIP migrated | keepalived | Traffic restored |
| 14:37:00 | Validation complete | John Doe | RESOLVED |
| 14:47:00 | Old master rejoined | John Doe | Cluster healthy |

**Total RTO**: 5 minutes (détection + résolution)
**Total RPO**: 0 seconds (semi-sync)

## Impact

### Utilisateurs
- **Impactés**: ~5,000 utilisateurs actifs
- **Durée**: 5 minutes
- **Expérience**: 
  - Queries retournaient erreurs "Lost connection"
  - Certaines transactions ont dû être rejouées (Transaction Replay ON)

### Business
- **Revenue Loss**: ~500€ (10,000€/h × 5min/60)
- **Transactions échouées**: ~150 (30/min × 5min)
- **SLA Breach**: Non (SLA = 99.9% mensuel, cet incident = 99.988%)

## Root Cause Analysis

### Cause Immédiate
Master MariaDB OOM (Out Of Memory) kill par kernel

### Cause Racine
Configuration `innodb_buffer_pool_size` trop élevée (90% RAM) combinée avec :
- Spike trafic (+200% normal)
- Slow queries accumulant mémoire
- Absence limite `max_connections`

### Pourquoi cascade
1. **Pourquoi OOM?** 
   → Buffer pool 90% RAM + connection overhead + slow queries

2. **Pourquoi buffer pool 90%?** 
   → Configuration basée sur recommandation générique, pas adaptée à notre workload

3. **Pourquoi spike trafic non anticipé?** 
   → Pas de rate limiting applicatif

4. **Pourquoi slow queries?** 
   → Index manquant sur nouvelle feature (déployée J-2)

5. **Pourquoi index manquant détecté tard?** 
   → Pas de review performance queries en staging

## Ce Qui a Bien Fonctionné ✅
1. Alerting rapide (15 secondes)
2. Failover automatique fonctionnel (90 secondes)
3. Transaction Replay MariaDB 11.8 → Aucune perte transaction
4. Communication équipe efficace (Slack war room)
5. Runbook suivi correctement

## Ce Qui Doit Être Amélioré ❌
1. Configuration buffer pool non adaptée
2. Pas de rate limiting applicatif
3. Pas de review performance avant deploy
4. Monitoring RAM insuffisant (alerte à 95%, trop tard)
5. Documentation limite `max_connections` manquante

## Actions Correctives

| Action | Responsable | Deadline | Statut | Ticket |
|--------|-------------|----------|--------|--------|
| Réduire buffer pool à 70% RAM | DBA Team | 2025-12-15 | ✅ Done | OPS-1234 |
| Implémenter rate limiting app | Dev Team | 2025-12-20 | 🔄 In Progress | DEV-5678 |
| Ajouter index manquant | DBA Team | 2025-12-14 | ✅ Done | OPS-1235 |
| Alert RAM à 80% (early warning) | SRE Team | 2025-12-16 | ✅ Done | OPS-1236 |
| Documenter max_connections tuning | DBA Team | 2025-12-17 | 🔄 In Progress | DOC-789 |
| Review perf queries staging CI | Dev Team | 2025-12-31 | 📅 Scheduled | DEV-5679 |
| Chaos test OOM scenario | SRE Team | 2026-01-15 | 📅 Scheduled | OPS-1237 |

## Leçons Apprises

### Technique
1. **MariaDB 11.8 Transaction Replay = Game Changer**
   - Aucune transaction perdue malgré crash brutal
   - Transparence totale pour applications
   - Investment upgrade 11.8 validé

2. **Semi-Sync Replication Essentiel**
   - RPO = 0 grâce à semi-sync
   - Coût performance négligeable (<5%)

### Processus
1. **Runbooks Saved Us**
   - Procédure suivie sans hésitation
   - RTO respecté grâce à procédure claire

2. **Communication War Room Efficace**
   - Slack channel dédié
   - Updates fréquents
   - Pas de confusion

### Organisationnel
1. **Manque Review Performance Pre-Prod**
   - Besoin CI pipeline avec tests charge
   - Staging doit avoir données représentatives

## Métriques

```
RTO Objectif: 5 minutes
RTO Réalisé: 5 minutes ✅

RPO Objectif: 0
RPO Réalisé: 0 ✅

MTBF (avant incident): 45 jours
MTTR: 5 minutes
Availability (mensuel): 99.988%
```

## Conclusion
Incident résolu efficacement grâce à automatisation (failover) et procédures (runbooks). Transaction Replay MariaDB 11.8 a empêché toute perte de données. Actions correctives identifiées pour éviter récurrence.

**Prochain Post-Mortem Review**: 2025-12-20

**Archivage**: [Link to ticket/wiki]
```

### 5.2 Métriques de Qualité DR

```sql
-- KPIs Disaster Recovery à suivre

-- 1. MTBF (Mean Time Between Failures)
SELECT 
    DATE_FORMAT(incident_date, '%Y-%m') AS month,
    COUNT(*) AS incidents,
    ROUND(30 * 24 / COUNT(*), 1) AS mtbf_hours
FROM incidents
WHERE severity IN ('P0', 'P1')
GROUP BY DATE_FORMAT(incident_date, '%Y-%m')
ORDER BY month DESC;

-- 2. MTTR (Mean Time To Recover)
SELECT 
    DATE_FORMAT(incident_date, '%Y-%m') AS month,
    AVG(TIMESTAMPDIFF(MINUTE, detected_at, resolved_at)) AS avg_mttr_minutes,
    MIN(TIMESTAMPDIFF(MINUTE, detected_at, resolved_at)) AS best_mttr,
    MAX(TIMESTAMPDIFF(MINUTE, detected_at, resolved_at)) AS worst_mttr
FROM incidents
WHERE severity IN ('P0', 'P1')
GROUP BY DATE_FORMAT(incident_date, '%Y-%m');

-- 3. RTO Achievement Rate
SELECT 
    COUNT(*) AS total_incidents,
    SUM(CASE 
        WHEN TIMESTAMPDIFF(MINUTE, detected_at, resolved_at) <= rto_target 
        THEN 1 ELSE 0 
    END) AS within_rto,
    ROUND(100.0 * SUM(CASE 
        WHEN TIMESTAMPDIFF(MINUTE, detected_at, resolved_at) <= rto_target 
        THEN 1 ELSE 0 
    END) / COUNT(*), 2) AS rto_achievement_rate
FROM incidents
WHERE severity IN ('P0', 'P1');

-- 4. DR Test Success Rate
SELECT 
    DATE_FORMAT(test_date, '%Y-%m') AS month,
    COUNT(*) AS total_tests,
    SUM(CASE WHEN result = 'SUCCESS' THEN 1 ELSE 0 END) AS successful,
    ROUND(100.0 * SUM(CASE WHEN result = 'SUCCESS' THEN 1 ELSE 0 END) / COUNT(*), 2) AS success_rate
FROM dr_tests
GROUP BY DATE_FORMAT(test_date, '%Y-%m')
ORDER BY month DESC;
```

---

## ✅ Points Clés à Retenir

- **RPO/RTO** : Définir objectifs réalistes basés sur impact business
- **Runbooks** : Documentation vivante, testée régulièrement, pas un PDF poussiéreux
- **Tests DR** : Minimum trimestriel, idéalement mensuel avec chaos engineering
- **Post-Mortems** : Sans blâme, focus sur amélioration système
- **Automatisation** : Failover manuel = RTO élevé, automatisation = RTO < 5 min
- **MariaDB 11.8** : Transaction Replay révolutionne la gestion d'incidents
- **Communication** : War room, updates fréquents, transparence stakeholders
- **Amélioration continue** : Chaque incident = opportunité d'apprentissage
- **Mesure** : MTBF, MTTR, RTO achievement → KPIs essentiels
- **Culture** : Résilience est une responsabilité d'équipe, pas individuelle

---

## 🔗 Ressources et Références

### Frameworks et Standards
- [📖 ITIL Incident Management](https://www.axelos.com/certifications/itil-service-management)
- [📖 ISO 22301 Business Continuity](https://www.iso.org/standard/75106.html)
- [📖 NIST Disaster Recovery](https://www.nist.gov/publications/contingency-planning-guide-federal-information-systems)

### Outils
- [PagerDuty](https://www.pagerduty.com/) - Incident management
- [Statuspage](https://www.atlassian.com/software/statuspage) - Status communication
- [Chaos Toolkit](https://chaostoolkit.org/) - Chaos engineering
- [Gremlin](https://www.gremlin.com/) - Enterprise chaos engineering

### Livres Recommandés
- **"Site Reliability Engineering"** - Google (O'Reilly)
- **"The Phoenix Project"** - Gene Kim
- **"Incident Management for Operations"** - Rob Schnepp (O'Reilly)

### Articles et Blogs
- **"How We Do Post-Mortems at Google"** - Google SRE Blog
- **"Runbook Template"** - Atlassian
- **"Chaos Engineering Principles"** - Netflix Tech Blog

---

**La meilleure stratégie de récupération est celle qui est testée régulièrement, documentée clairement, et améliorée continuellement. Un incident bien géré renforce la confiance ; un incident mal géré détruit la réputation.**

⏭️ [Alternatives : ProxySQL et HAProxy](/14-haute-disponibilite/09-proxysql-haproxy.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.3 Split-Brain et Quorum

> **Niveau** : Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Section 14.2 (Galera Cluster), compréhension des systèmes distribués

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le phénomène de split-brain et ses conséquences catastrophiques
- **Configurer** le quorum Galera pour prévenir les split-brains
- **Détecter** rapidement les situations de partition réseau
- **Mettre en œuvre** des stratégies de fencing et de résolution
- **Récupérer** d'un split-brain avec perte de données minimale
- **Concevoir** des architectures résistantes aux partitions réseau
- **Opérer** un cluster multi-datacenter avec gestion du quorum avancée

---

## Introduction

Le **split-brain** (cerveau divisé) est l'un des problèmes les plus redoutés dans les systèmes distribués. Dans le contexte Galera Cluster, il survient lorsque plusieurs segments du cluster se croient légitimes et continuent à accepter des écritures de manière indépendante, créant des **divergences de données irréconciliables**.

Le **quorum** est le mécanisme fondamental qui prévient ce scénario catastrophique. Comprendre son fonctionnement et savoir le configurer correctement est **absolument critique** pour toute architecture Galera en production.

> ⚠️ **Warning Critical** : Un split-brain mal géré peut entraîner une perte de données définitive et nécessiter une reconstruction complète du cluster. Ce n'est pas une simple indisponibilité, c'est une corruption de données.

---

## 1. Le Phénomène de Split-Brain

### 1.1 Définition et Anatomie

**Split-brain** : État où un cluster distribué se scinde en deux (ou plus) partitions indépendantes, chacune se considérant comme le cluster légitime et continuant à accepter des transactions.

#### **Scénario Classique de Split-Brain**

```
État Initial : Cluster 3 nœuds sain
┌─────────────────────────────────────────┐
│         Application Servers             │
└────────┬───────────┬───────────┬────────┘
         │           │           │
    ┌────▼────┐ ┌───▼─────┐ ┌──▼──────┐
    │ Node 1  │─│ Node 2  │─│ Node 3  │
    │ (DC1)   │ │ (DC1)   │ │ (DC2)   │
    └─────────┘ └─────────┘ └─────────┘
         All synchronized ✅


T+0 : Partition réseau entre DC1 et DC2
┌──────────────────────────────────┐     ┌──────────────┐
│        PARTITION 1 (DC1)         │ ✖   │ PARTITION 2  │
│   ┌────────┐ ┌────────┐          │     │  ┌────────┐  │
│   │ Node 1 │─│ Node 2 │          │     │  │ Node 3 │  │
│   └────────┘ └────────┘          │     │  └────────┘  │
│   Size: 2 (Majority ✅)          │     │  Size: 1     │
│   Status: PRIMARY                │     │  Status: NON-PRIMARY
│   Accepts writes ✅              │     │  Read-only ✅│
└──────────────────────────────────┘     └──────────────┘

✅ GOOD : Quorum empêche le split-brain
→ Partition 1 continue seule (a le quorum 2/3)
→ Partition 2 se met automatiquement en read-only


T+0 : Scénario SANS quorum (2 nœuds initiaux)
┌────────────────────┐                 ┌────────────────────┐
│   PARTITION 1      │ ✖ ✖ ✖           │   PARTITION 2      │
│   ┌────────┐       │                 │   ┌────────┐       │
│   │ Node 1 │       │                 │   │ Node 2 │       │
│   └────────┘       │                 │   └────────┘       │
│   Size: 1          │                 │   Size: 1          │
│   Thinks: I'm alone│                 │   Thinks: I'm alone│
│   Accepts writes ❌│                 │   Accepts writes ❌│
└────────────────────┘                 └────────────────────┘

❌ DISASTER : Les deux partitions acceptent des writes divergents !

-- Partition 1
UPDATE accounts SET balance = 1000 WHERE id = 123;

-- Partition 2 (simultané)
UPDATE accounts SET balance = 500 WHERE id = 123;

→ Données irréconciliables sans intervention manuelle
```

### 1.2 Causes Fréquentes de Split-Brain

#### **1. Partition Réseau (Network Partition)**

**Causes techniques** :
```
- Défaillance switch/routeur
- Câble réseau débranché/coupé
- Configuration firewall erronée
- Saturation bande passante
- Problèmes routing (BGP flapping)
- Maintenance réseau mal planifiée
```

**Exemple réel** :
```bash
# Vérification logs système lors partition
tail -f /var/log/mysql/error.log

2025-12-13 14:32:15 [Warning] WSREP: (node1) gcs_core(EVS): 
  Unable to receive from tcp://10.0.2.10:4567
2025-12-13 14:32:20 [Warning] WSREP: gcs_core(EVS): 
  evs::proto(node1, OPERATIONAL, view_id(REG,node1,5)): 
  suspecting node: node3
2025-12-13 14:32:30 [Note] WSREP: declaring node3 at tcp://10.0.2.10:4567 stable
2025-12-13 14:32:30 [Note] WSREP: forgetting node3 (tcp://10.0.2.10:4567)
2025-12-13 14:32:31 [Note] WSREP: New cluster view: global state: 
  a1b2c3d4:123, view#6: PRIMARY (2)
```

#### **2. Asymmetric Network Failure**

**Partition asymétrique** : Cas particulièrement vicieux
```
Node1 peut communiquer avec Node2 : ✅
Node2 peut communiquer avec Node1 : ✅
Node1 peut communiquer avec Node3 : ❌
Node2 peut communiquer avec Node3 : ✅
Node3 peut communiquer avec Node1 : ❌
Node3 peut communiquer avec Node2 : ✅

Résultat :
- Node1 voit : {Node1, Node2} = 2 nœuds
- Node2 voit : {Node1, Node2, Node3} = 3 nœuds (all OK)
- Node3 voit : {Node2, Node3} = 2 nœuds

→ Situation instable, peut générer flapping
```

#### **3. Tuning Agressif de Timeouts**

```ini
# Configuration DANGEREUSE
[galera]
wsrep_provider_options = "
    evs.suspect_timeout = PT1S;    # Trop court !
    evs.inactive_timeout = PT3S;   # Trop court !
"

# Conséquence : False positives
# → Nœud temporairement lent marqué suspect
# → Éviction prématurée
# → Réintégration → Éviction → Cycle
```

💡 **Recommandation** : Timeouts conservateurs, surtout pour WAN.

#### **4. Défaillances en Cascade**

```
Initial state: 5 nœuds sains

T+0  : Node1 crash (hardware)
       → Cluster: 4 nœuds (quorum OK: 3/5)

T+30s: Node2 surchargé (prend la charge de Node1)
       → Ralentissement
       → Heartbeat tardifs

T+45s: Node2 suspecté par Node3, Node4, Node5
       → Éviction Node2
       → Cluster: 3 nœuds (quorum OK: 2/5)

T+60s: Node3 surchargé à son tour
       → Même scénario

→ Effet domino jusqu'à perte de quorum
```

### 1.3 Conséquences Catastrophiques

#### **Divergence de Données**

```sql
-- État avant split-brain
SELECT * FROM orders WHERE id = 999;
-- +-----+--------+--------+
-- | id  | status | amount |
-- +-----+--------+--------+
-- | 999 | PAID   | 100.00 |
-- +-----+--------+--------+

-- SPLIT-BRAIN COMMENCE

-- Partition 1 (Node1, Node2)
UPDATE orders SET status = 'SHIPPED', amount = 95.00 
WHERE id = 999;
-- Client applique un discount

-- Partition 2 (Node3) - simultané
UPDATE orders SET status = 'CANCELLED', amount = 0.00 
WHERE id = 999;
-- Annulation commande

-- APRÈS RÉSOLUTION
-- Node1: status = SHIPPED, amount = 95.00
-- Node2: status = SHIPPED, amount = 95.00
-- Node3: status = CANCELLED, amount = 0.00

→ Incohérence irréconciliable
→ Nécessite arbitrage manuel (quelle est la vraie version ?)
```

#### **Violations d'Intégrité**

```sql
-- Partition 1
INSERT INTO users (id, email) VALUES (123, 'user@example.com');

-- Partition 2 (simultané)
INSERT INTO users (id, email) VALUES (123, 'different@example.com');

-- Après résolution
→ ERREUR: Duplicate key violation
→ Impossible de fusionner automatiquement
```

#### **Impact Business**

```
Exemples réels de coûts split-brain :

E-commerce :
- Commandes facturées 2 fois
- Stock inventory incohérent
- Pertes estimées : 50k-500k€ / incident

Banking :
- Transactions dupliquées
- Soldes de comptes divergents
- Implications réglementaires + amendes

SaaS :
- Données clients corrompues
- Perte de confiance
- Churn client massif
```

---

## 2. Le Quorum : Mécanisme de Protection

### 2.1 Principe du Quorum

**Définition** : Le quorum est le nombre minimal de nœuds qui doivent être connectés pour qu'une partition soit considérée comme **PRIMARY** (apte à accepter des écritures).

**Formule** :
```
Quorum = floor(n_nodes / 2) + 1

Exemples :
3 nœuds : Quorum = floor(3/2) + 1 = 2
5 nœuds : Quorum = floor(5/2) + 1 = 3
7 nœuds : Quorum = floor(7/2) + 1 = 4
```

**Garantie** : Au maximum **une seule** partition peut avoir le quorum à un instant donné.

```
Cluster 5 nœuds, partition en 3-2 :

Partition A (3 nœuds) : 3 ≥ 3 (quorum) → PRIMARY ✅
Partition B (2 nœuds) : 2 < 3 (quorum) → NON-PRIMARY ❌

→ Impossible d'avoir 2 partitions PRIMARY simultanément
```

### 2.2 États de Partition

#### **PRIMARY (Opérationnel)**

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';
-- +----------------------+---------+
-- | Variable_name        | Value   |
-- +----------------------+---------+
-- | wsrep_cluster_status | Primary |
-- +----------------------+---------+

-- Caractéristiques :
-- ✅ Accepte les écritures (READ/WRITE)
-- ✅ Réplication active
-- ✅ Peut servir de donneur SST/IST
```

**Configuration visible** :
```sql
SHOW STATUS LIKE 'wsrep%';

wsrep_cluster_size          : 3      -- Taille partition
wsrep_cluster_status        : Primary
wsrep_local_state_comment   : Synced
wsrep_ready                 : ON
wsrep_connected             : ON
```

#### **NON-PRIMARY (Read-Only Forcé)**

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';
-- +----------------------+--------------+
-- | Variable_name        | Value        |
-- +----------------------+--------------+
-- | wsrep_cluster_status | Non-primary  |
-- +----------------------+--------------+

-- Caractéristiques :
-- ❌ Refuse les écritures
-- ❌ Pas de réplication
-- ✅ Lectures possibles (données potentiellement obsolètes)
```

**Comportement automatique** :
```sql
-- Tentative INSERT dans partition NON-PRIMARY
INSERT INTO users (name) VALUES ('Test');
-- ERROR 1047 (08S01): WSREP has not yet prepared node for application use

-- Mode effectif
SHOW VARIABLES LIKE 'read_only';
-- +---------------+-------+
-- | Variable_name | Value |
-- +---------------+-------+
-- | read_only     | ON    |
-- +---------------+-------+
```

### 2.3 Configuration du Quorum

#### **Configuration Standard (Automatique)**

```ini
# /etc/mysql/conf.d/galera.cnf
[galera]
wsrep_provider = /usr/lib/libgalera_smm.so

# Quorum automatique basé sur cluster_size
# PAS BESOIN de configurer explicitement

# Liste des nœuds du cluster
wsrep_cluster_address = "gcomm://node1,node2,node3"

# Le quorum est calculé automatiquement :
# - 3 nœuds → quorum = 2
# - La partition avec 2+ nœuds sera PRIMARY
```

#### **Bootstrap du Cluster (Premier Démarrage)**

```bash
# Sur le PREMIER nœud uniquement
# /etc/mysql/conf.d/galera.cnf

[galera]
wsrep_cluster_address = "gcomm://"  # Adresse vide = bootstrap

# OU via ligne de commande
galera_new_cluster

# OU systemd
systemctl start mariadb@bootstrap
```

⚠️ **CRITICAL** : Ne jamais bootstrapper plusieurs nœuds simultanément → split-brain garanti !

**Procédure de bootstrap correcte** :
```bash
# 1. Bootstrap Node1
node1$ systemctl start mariadb@bootstrap

node1$ mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# wsrep_cluster_size = 1

# 2. Démarrer Node2
node2$ systemctl start mariadb
# Rejoint automatiquement Node1

node1$ mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# wsrep_cluster_size = 2  (quorum atteint ✅)

# 3. Démarrer Node3
node3$ systemctl start mariadb

node1$ mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# wsrep_cluster_size = 3

# 4. Remettre Node1 en mode normal (redémarrage)
node1$ systemctl stop mariadb@bootstrap
node1$ systemctl start mariadb  # Rejoint le cluster normalement
```

#### **Quorum Forcé (wsrep_provider_options)**

```ini
# Configuration avancée (rarement nécessaire)
[galera]
wsrep_provider_options = "
    pc.ignore_sb = false;           # Default: Respect quorum strict
    pc.ignore_quorum = false;       # Default: Quorum obligatoire
    pc.bootstrap = false;           # Ne pas bootstrap automatiquement
"
```

⚠️ **DANGER** : `pc.ignore_sb = true` désactive la protection split-brain → **JAMAIS en production !**

### 2.4 Quorum dans Architectures Spéciales

#### **Cluster 2 Nœuds (Anti-Pattern)**

```
Problème : 2 nœuds, quorum = 2

Partition réseau :
- Node1 (size=1) : 1 < 2 → NON-PRIMARY ❌
- Node2 (size=1) : 1 < 2 → NON-PRIMARY ❌

→ Cluster COMPLÈTEMENT down (les deux en read-only)
```

**Solutions de contournement** :

**A. Utiliser Arbitrator (Garbd)** 
```bash
# Déployer garbd (pas de données, juste vote quorum)
apt-get install galera-arbitrator-4

# /etc/default/garbd
GALERA_NODES="node1:4567,node2:4567"
GALERA_GROUP="production_cluster"

systemctl start garbd

# Cluster effectif : 3 membres (2 DB + 1 arbitrator)
# Quorum = 2
# Perte Node1 → {Node2, Garbd} = 2 → quorum OK ✅
```

**B. PC Weight (Poids personnalisé)** ⚠️ Risqué
```ini
# Node1 (serveur principal)
wsrep_provider_options = "pc.weight=2"  # Compte double

# Node2 (standby)
wsrep_provider_options = "pc.weight=1"

# Quorum basé sur poids total, pas nombre de nœuds
# Partition Node1 seul : weight=2 ≥ 2 → PRIMARY
# Partition Node2 seul : weight=1 < 2 → NON-PRIMARY

# ⚠️ DANGER : En cas de panne Node1, Node2 ne peut pas prendre le relais !
```

💡 **Recommandation forte** : **Toujours 3+ nœuds** ou utiliser garbd. Jamais 2 nœuds en production.

#### **Multi-Datacenter avec Segments**

```ini
# Segmentation pour optimiser communication intra-DC
[galera]
wsrep_provider_options = "
    gmcast.segment = 0;  # DC1 (Paris)
"

# Node3, Node4 (Londres)
wsrep_provider_options = "
    gmcast.segment = 1;  # DC2 (Londres)
"

# Garbd (Dublin)
wsrep_provider_options = "
    gmcast.segment = 2;  # DC3 (Dublin - arbitre)
"

# Avantage : Optimise traffic réseau intra-segment
# Quorum global maintenu sur l'ensemble
```

**Scénario perte DC1** :
```
5 nœuds : DC1(2) + DC2(2) + Garbd(1)
Quorum = 3

Perte DC1 :
Partition DC2+Garbd : 2+1 = 3 ≥ 3 → PRIMARY ✅
```

---

## 3. Détection de Split-Brain

### 3.1 Monitoring Proactif

#### **Métriques Clés à Surveiller**

```sql
-- Script de monitoring (à exécuter périodiquement)
SELECT 
    @@hostname AS node_name,
    -- Statut cluster
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_cluster_status') AS cluster_status,
    
    -- Taille cluster
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_cluster_size') AS cluster_size,
    
    -- Taille configurée
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_cluster_conf_id') AS conf_id,
    
    -- État local
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_local_state_comment') AS local_state,
    
    -- Connecté ?
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_connected') AS connected,
    
    -- Ready ?
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='wsrep_ready') AS ready;

-- Résultat SAIN :
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
-- | node_name | cluster_status | cluster_size | conf_id | local_state | connected | ready |
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
-- | node1     | Primary        | 3            | 42      | Synced      | ON        | ON    |
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+

-- Résultat SPLIT-BRAIN :
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
-- | node_name | cluster_status | cluster_size | conf_id | local_state | connected | ready |
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
-- | node1     | Primary        | 2            | 43      | Synced      | ON        | ON    |
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+

-- Simultanément sur node3 :
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
-- | node_name | cluster_status | cluster_size | conf_id | local_state | connected | ready |
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
-- | node3     | Non-primary    | 1            | 43      | Synced      | ON        | OFF   |
-- +-----------+----------------+--------------+---------+-------------+-----------+-------+
```

**Alertes critiques** :
```bash
#!/bin/bash
# galera_monitor.sh

EXPECTED_SIZE=5  # Nombre total de nœuds

current_size=$(mysql -N -e "SHOW STATUS LIKE 'wsrep_cluster_size'" | awk '{print $2}')
cluster_status=$(mysql -N -e "SHOW STATUS LIKE 'wsrep_cluster_status'" | awk '{print $2}')

if [ "$cluster_status" != "Primary" ]; then
    echo "CRITICAL: Node is NON-PRIMARY !"
    # Envoyer alerte PagerDuty/OpsGenie
    exit 2
fi

if [ "$current_size" -lt "$((EXPECTED_SIZE / 2 + 1))" ]; then
    echo "WARNING: Cluster size ($current_size) below quorum threshold"
    exit 1
fi

if [ "$current_size" -lt "$EXPECTED_SIZE" ]; then
    echo "WARNING: Cluster degraded ($current_size/$EXPECTED_SIZE nodes)"
    exit 1
fi

echo "OK: Cluster healthy ($current_size nodes, Primary)"
exit 0
```

#### **Prometheus Exporters**

```yaml
# mysqld_exporter configuration
# /etc/mysqld_exporter/mysqld_exporter.cnf

[client]
user = exporter
password = secure_password

# Métriques Galera exposées
[mysqld_exporter]
collect.global_status = true
collect.info_schema.innodb_metrics = true
```

**Requêtes PromQL pour alerting** :
```promql
# Alerte : Nœud NON-PRIMARY
mysql_global_status_wsrep_cluster_status != 1

# Alerte : Cluster size réduit
mysql_global_status_wsrep_cluster_size < 5

# Alerte : Nœud déconnecté
mysql_global_status_wsrep_connected != 1

# Alerte : Nœud not ready
mysql_global_status_wsrep_ready != 1
```

### 3.2 Analyse des Logs

#### **Patterns de Logs Suspects**

```bash
# /var/log/mysql/error.log

# 1. Détection perte de connexion
2025-12-13 15:42:10 [Warning] WSREP: gcs_core(EVS): Unable to receive from tcp://10.0.2.12:4567
2025-12-13 15:42:15 [Warning] WSREP: gcs_core(EVS): suspecting node: node3 (tcp://10.0.2.12:4567)

# 2. Éviction d'un nœud
2025-12-13 15:42:30 [Note] WSREP: forgetting node3 (tcp://10.0.2.12:4567)
2025-12-13 15:42:31 [Note] WSREP: New cluster view: global state: uuid:123, view#44: Primary (2)

# 3. Perte de quorum (DANGER !)
2025-12-13 15:45:00 [Note] WSREP: New cluster view: global state: uuid:123, view#45: Non-Primary (1)
2025-12-13 15:45:00 [Note] WSREP: Current view of cluster as seen by this node:
view (view_id(NON_PRIM,node1,45) memb {
    node1,tcp://10.0.1.10:4567
} joined {
} left {
} partitioned {
    node2,tcp://10.0.1.11:4567
    node3,tcp://10.0.2.12:4567
})

# 4. Basculement en read_only
2025-12-13 15:45:01 [Note] WSREP: Server status change connected -> donor/desynced
2025-12-13 15:45:01 [Note] WSREP: wsrep_ready changed from ON to OFF
```

**Parser automatique** :
```bash
#!/bin/bash
# detect_split_brain.sh

LOG_FILE="/var/log/mysql/error.log"

# Recherche patterns critiques dans les 5 dernières minutes
SINCE=$(date --date='5 minutes ago' '+%Y-%m-%d %H:%M')

grep -A 5 "Non-Primary" $LOG_FILE | grep "$SINCE" && {
    echo "CRITICAL: Split-brain detected!"
    echo "Node is in NON-PRIMARY state"
    
    # Extraire taille partition
    cluster_size=$(mysql -N -e "SHOW STATUS LIKE 'wsrep_cluster_size'" | awk '{print $2}')
    echo "Current partition size: $cluster_size"
    
    # Alerter équipe
    send_alert "Split-brain on $(hostname)"
}
```

---

## 4. Résolution de Split-Brain

### 4.1 Scénarios de Résolution

#### **Scénario 1 : Split-Brain avec Partition Gagnante Claire**

```
Situation :
- Partition A (3 nœuds) : PRIMARY ✅
- Partition B (2 nœuds) : NON-PRIMARY ❌

Résolution automatique :
→ Pas d'intervention nécessaire
→ Partition B attend reconnexion
→ Après reconnexion, IST depuis Partition A
```

**Reconnexion réseau** :
```bash
# Sur les nœuds de Partition B
# Logs automatiques après rétablissement réseau

2025-12-13 16:00:00 [Note] WSREP: Received NON-PRIMARY view
2025-12-13 16:00:05 [Note] WSREP: Connection restored to node1 (tcp://10.0.1.10:4567)
2025-12-13 16:00:06 [Note] WSREP: Requesting IST from node1
2025-12-13 16:00:10 [Note] WSREP: IST received: 234 transactions
2025-12-13 16:00:11 [Note] WSREP: New cluster view: Primary (5)
2025-12-13 16:00:11 [Note] WSREP: Synchronized with group, ready for connections

# Vérification
mysql -e "SHOW STATUS LIKE 'wsrep%'"
wsrep_cluster_size     : 5
wsrep_cluster_status   : Primary
wsrep_local_state      : 4 (Synced)
```

✅ **Résultat** : Synchronisation automatique, aucune perte de données.

#### **Scénario 2 : Split-Brain 50-50 (Toutes Partitions NON-PRIMARY)**

```
Situation critique :
Cluster 4 nœuds split 2-2

- Partition A (2 nœuds) : 2 < 3 → NON-PRIMARY ❌
- Partition B (2 nœuds) : 2 < 3 → NON-PRIMARY ❌

→ Cluster COMPLÈTEMENT down (catastrophe !)
```

**Résolution manuelle requise** :

**Méthode 1 : Bootstrap une partition (si réseau restauré)**
```bash
# 1. Arrêter tous les nœuds proprement
node1$ systemctl stop mariadb
node2$ systemctl stop mariadb
node3$ systemctl stop mariadb
node4$ systemctl stop mariadb

# 2. Identifier le nœud avec seqno le plus récent
node1$ galera_recovery
# Recovered position: uuid:seqno = a1b2c3:1234

node2$ galera_recovery
# Recovered position: uuid:seqno = a1b2c3:1234

node3$ galera_recovery
# Recovered position: uuid:seqno = a1b2c3:1220  # Plus ancien

node4$ galera_recovery
# Recovered position: uuid:seqno = a1b2c3:1220

# 3. Bootstrap depuis le nœud le plus récent (node1 ou node2)
node1$ systemctl start mariadb@bootstrap

# 4. Rejoindre les autres nœuds
node2$ systemctl start mariadb
node3$ systemctl start mariadb
node4$ systemctl start mariadb

# 5. Vérifier cluster
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# wsrep_cluster_size = 4 ✅
```

**Méthode 2 : Forcer quorum (DANGER - dernier recours)**
```sql
-- SUR UN SEUL NŒUD de la partition à privilégier
SET GLOBAL wsrep_provider_options = 'pc.bootstrap=true';

-- Vérification
SHOW STATUS LIKE 'wsrep_cluster_status';
-- wsrep_cluster_status : Primary ✅

-- ⚠️ ATTENTION : Les autres nœuds doivent être arrêtés
-- avant d'utiliser cette commande !
```

#### **Scénario 3 : Split-Brain avec Écritures Divergentes (PIRE CAS)**

```
Situation catastrophique :
- 2 partitions ont accepté des écritures
- Données divergentes irréconciliables

Exemple :
Partition A :
  UPDATE users SET email='a@example.com' WHERE id=1;
  
Partition B :
  UPDATE users SET email='b@example.com' WHERE id=1;
```

**Résolution nécessite arbitrage humain** :

```bash
# 1. Identifier quelle partition contient les données "correctes"
# Critères de décision :
#   - Quelle partition a le plus de nœuds ?
#   - Quelle partition a les données les plus récentes ?
#   - Impact business : quelle version accepter ?

# 2. Backup exhaustif des DEUX partitions
# Partition A
node1$ mysqldump --all-databases > /backup/partition_a_$(date +%s).sql

# Partition B
node3$ mysqldump --all-databases > /backup/partition_b_$(date +%s).sql

# 3. Décision : Partition A = source de vérité

# 4. Détruire et reconstruire Partition B
node3$ systemctl stop mariadb
node3$ rm -rf /var/lib/mysql/*
node3$ systemctl start mariadb  # SST complet depuis Partition A

# 5. Analyse manuelle des différences pour récupération données
diff /backup/partition_a_*.sql /backup/partition_b_*.sql > divergences.txt

# 6. Réappliquer manuellement les transactions perdues si nécessaire
# (après validation business)
```

⚠️ **IMPORTANT** : Documenter exhaustivement l'incident et les décisions prises.

### 4.2 Outils de Récupération

#### **galera_recovery**

```bash
# Déterminer le dernier état connu du nœud
/usr/bin/galera_recovery

# Output :
# WSREP: Recovered position: uuid:seqno = a1b2c3d4-e5f6-7890-abcd-ef1234567890:1234

# Interprétation :
# uuid : Identifiant unique du cluster
# seqno : Numéro de séquence (transaction)
# → Ce nœud a appliqué jusqu'à la transaction #1234
```

**Usage dans bootstrap** :
```bash
# Comparer seqno de tous les nœuds
for node in node1 node2 node3; do
    echo "=== $node ==="
    ssh $node "/usr/bin/galera_recovery"
done

# Bootstrapper depuis le nœud avec seqno le plus élevé
```

#### **grastate.dat**

```bash
# Fichier d'état Galera
cat /var/lib/mysql/grastate.dat

# version: 2.1
# uuid:    a1b2c3d4-e5f6-7890-abcd-ef1234567890
# seqno:   1234
# safe_to_bootstrap: 0

# safe_to_bootstrap:
#   0 = Shutdown non gracieux (crash) → Vérifier avec d'autres nœuds
#   1 = Dernier nœud à se déconnecter proprement → Bootstrap OK
```

**Manipulation manuelle (cas extrême)** :
```bash
# SI ET SEULEMENT SI :
# - Cluster complètement down
# - Ce nœud a les données les plus récentes
# - Backup effectué

vim /var/lib/mysql/grastate.dat
# Modifier :
# safe_to_bootstrap: 1

# Puis bootstrap
systemctl start mariadb@bootstrap
```

⚠️ **DANGER** : Ne modifier grastate.dat qu'en dernier recours et avec backup !

---

## 5. Stratégies de Prévention

### 5.1 Architecture Résiliante

#### **Nombre de Nœuds Optimal**

```
Production recommandée :

Small cluster  : 3 nœuds  (tolère 1 panne)
Medium cluster : 5 nœuds  (tolère 2 pannes)
Large cluster  : 7 nœuds  (tolère 3 pannes)

Règle d'or : Toujours un nombre IMPAIR de nœuds
→ Évite les partitions 50-50
```

**Distribution multi-DC** :
```
Exemple 5 nœuds, 3 DC :

DC1 (Paris)   : 2 nœuds
DC2 (Londres) : 2 nœuds
DC3 (Dublin)  : 1 Garbd (arbitrator)

Perte DC1 → {DC2 + DC3} = 2+1 = 3 ≥ 3 ✅
Perte DC2 → {DC1 + DC3} = 2+1 = 3 ≥ 3 ✅
Perte DC3 → {DC1 + DC2} = 2+2 = 4 ≥ 3 ✅
```

#### **Redondance Réseau**

```bash
# Configuration multi-path (bonding)
# /etc/network/interfaces

auto bond0
iface bond0 inet static
    address 10.0.1.10
    netmask 255.255.255.0
    bond-slaves eth0 eth1
    bond-mode active-backup
    bond-miimon 100
    bond-primary eth0

# Galera utilise l'interface bonding
# → Tolérance panne d'une NIC
```

**VLAN dédié Galera** :
```
Réseau application : VLAN 10 (10.10.0.0/24)
Réseau Galera      : VLAN 20 (10.20.0.0/24)
Réseau management  : VLAN 30 (10.30.0.0/24)

Avantages :
- Isolation du trafic de réplication
- QoS configurable
- Debugging facilité
```

### 5.2 Configuration Conservatrice

```ini
# /etc/mysql/conf.d/galera.cnf
[galera]

# Timeouts généreux (surtout multi-DC)
wsrep_provider_options = "
    # Keepalive léger
    evs.keepalive_period = PT1S;
    
    # Délai avant suspicion (générique)
    evs.suspect_timeout = PT10S;
    
    # Timeout inactivité (multi-DC : 30s+)
    evs.inactive_timeout = PT30S;
    
    # Timeout installation vue
    evs.install_timeout = PT30S;
    
    # Send window (limiter burst)
    evs.send_window = 512;
    
    # User messages (heartbeat applicatif)
    evs.user_send_window = 256;
    
    # Consensus timeout
    evs.consensus_timeout = PT30S;
"

# Éviter évictions prématurées
# Multi-DC avec latence WAN : doubler/tripler ces valeurs
```

**Tuning réseau WAN** :
```ini
# Configuration pour latence 50-100ms inter-DC
wsrep_provider_options = "
    evs.suspect_timeout = PT30S;
    evs.inactive_timeout = PT60S;
    evs.install_timeout = PT60S;
    
    # Augmenter buffers
    evs.send_window = 1024;
    evs.user_send_window = 512;
    
    # Segment markers
    gmcast.segment = 0;  # Ajuster par DC
"
```

### 5.3 Monitoring et Alerting Agressif

```yaml
# prometheus_alerts.yml
groups:
  - name: galera_cluster
    interval: 10s
    rules:
      - alert: GaleraNodeNonPrimary
        expr: mysql_global_status_wsrep_cluster_status != 1
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Galera node {{ $labels.instance }} is NON-PRIMARY"
          description: "Split-brain possible, immediate investigation required"
      
      - alert: GaleraClusterSizeReduced
        expr: mysql_global_status_wsrep_cluster_size < 5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Galera cluster degraded on {{ $labels.instance }}"
          description: "Cluster size: {{ $value }}/5 - Risk of quorum loss"
      
      - alert: GaleraNodeDisconnected
        expr: mysql_global_status_wsrep_connected != 1
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Galera node {{ $labels.instance }} disconnected"
      
      - alert: GaleraNodeNotReady
        expr: mysql_global_status_wsrep_ready != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Galera node {{ $labels.instance }} not ready"
```

### 5.4 Runbooks et Procédures

**Checklist incident split-brain** :
```markdown
# RUNBOOK : Split-Brain Galera

## Détection
- [ ] Alerte Prometheus/Nagios reçue
- [ ] Vérifier wsrep_cluster_status sur tous les nœuds
- [ ] Identifier taille de chaque partition

## Investigation
- [ ] Analyser /var/log/mysql/error.log (patterns partition)
- [ ] Vérifier connectivité réseau entre nœuds (ping, nc)
- [ ] Consulter monitoring réseau (switch, firewall)

## Résolution (si partition réseau résolue)
- [ ] Attendre reconnexion automatique (5 min max)
- [ ] Vérifier IST sur nœuds NON-PRIMARY
- [ ] Valider cluster_size = attendu

## Résolution (si cluster complètement down)
- [ ] BACKUP immédiat de tous les nœuds
- [ ] galera_recovery sur tous les nœuds
- [ ] Identifier seqno max
- [ ] Bootstrap depuis nœud seqno max
- [ ] Rejoindre autres nœuds séquentiellement

## Résolution (écritures divergentes)
- [ ] BACKUP exhaustif TOUTES les partitions
- [ ] Arbitrage business (quelle version garder)
- [ ] Reconstruction partition perdante (SST complet)
- [ ] Analyse diff pour récupération manuelle

## Post-Mortem
- [ ] Documenter incident timeline
- [ ] Identifier root cause
- [ ] Plan d'action préventif
- [ ] Mise à jour runbook
```

---

## 6. Cas Avancés et Edge Cases

### 6.1 Garagekeeper (Arbitrator) - Configuration Avancée

```bash
# Installation garbd
apt-get install galera-arbitrator-4

# Configuration systemd
cat > /etc/systemd/system/garbd.service <<EOF
[Unit]
Description=Galera Arbitrator Daemon
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/bin/garbd \\
    --group production_cluster \\
    --address "gcomm://node1:4567,node2:4567,node3:4567,node4:4567" \\
    --options "gmcast.listen_addr=tcp://0.0.0.0:4567;gmcast.segment=2" \\
    --log /var/log/garbd.log \\
    --sst rsync

Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable garbd
systemctl start garbd
```

**Monitoring garbd** :
```bash
# Vérifier logs
tail -f /var/log/garbd.log

# Vérifier depuis nœuds DB
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# Doit inclure garbd (ex: 5 si 4 DB + 1 garbd)

mysql -e "SHOW STATUS LIKE 'wsrep_incoming_addresses'"
# Liste tous les membres incluant garbd
```

### 6.2 Weighted Quorum (PC Weight)

**Configuration poids personnalisé** :
```ini
# Use case : Cluster asymétrique

# Node1 (Datacenter principal, hardware puissant)
[galera]
wsrep_provider_options = "pc.weight=3"

# Node2 (Datacenter principal)
wsrep_provider_options = "pc.weight=3"

# Node3 (Datacenter DR, hardware moindre)
wsrep_provider_options = "pc.weight=1"

# Node4 (Datacenter DR)
wsrep_provider_options = "pc.weight=1"

# Quorum basé sur poids :
# Total weight = 3+3+1+1 = 8
# Quorum weight = 8/2 + 1 = 5

# Scénarios :
# Perte DC principal (Node1+2) : weight=1+1=2 < 5 → NON-PRIMARY ❌
# Perte DC DR (Node3+4)        : weight=3+3=6 ≥ 5 → PRIMARY ✅

# → DC principal peut fonctionner seul
```

⚠️ **ATTENTION** : Usage avancé, nécessite compréhension profonde des implications.

### 6.3 Last Committed (pc.recovery)

```ini
# Active automatiquement la récupération intelligente
[galera]
wsrep_provider_options = "pc.recovery=true"

# Comportement :
# - En cas de crash cluster complet
# - Nœuds tentent automatiquement de déterminer
#   lequel a le seqno le plus récent
# - Bootstrap automatique depuis ce nœud
# - SANS intervention manuelle

# ⚠️ Utiliser avec précaution :
# - Risque de bootstrap non intentionnel
# - Préférer procédure manuelle contrôlée en prod
```

---

## ✅ Points Clés à Retenir

- **Split-brain = divergence de données irréconciliable**, le pire scénario en système distribué
- **Quorum = seul mécanisme fiable** pour prévenir split-brain (≥ 50% + 1)
- **Toujours 3+ nœuds** (5 recommandé production), **jamais 2 nœuds** seuls
- **Arbitrator (garbd)** excellent pour 3ᵉ site ou cluster 2+1
- **Monitoring proactif** : alerting immédiat sur cluster_status, cluster_size
- **Logs critiques** : surveiller patterns "Non-Primary", "forgetting node"
- **Résolution automatique** si une partition a quorum clair (IST)
- **Résolution manuelle** si cluster complètement down (bootstrap + galera_recovery)
- **Arbitrage humain** si écritures divergentes (backup + reconstruction)
- **Prévention** : architecture multi-DC, réseau redondant, timeouts conservateurs
- **Runbooks essentiels** : procédures documentées et testées régulièrement

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 Galera Cluster Quorum](https://galeracluster.com/library/documentation/quorum.html)
- [📖 PC (Primary Component) Protocol](https://galeracluster.com/library/documentation/pc-protocol.html)
- [📖 Split-Brain Scenarios](https://galeracluster.com/library/kb/split-brain-and-quorum.html)

### Whitepapers et Articles
- **"Understanding Quorum in Distributed Systems"** - Academic Paper
- **"Split-Brain Resolution Strategies"** - Codership Blog
- **"Galera Arbitrator Deep Dive"** - MariaDB Knowledge Base

### Outils
- [garbd Documentation](https://galeracluster.com/library/documentation/arbitrator.html)
- [myq_gadgets - Galera Monitoring](https://github.com/jayjanssen/myq_gadgets)
- [Chaos Monkey for Galera](https://github.com/galera-chaos) - Testing tool

---

## ➡️ Section Suivante

**[14.4 MaxScale](/14-haute-disponibilite/04-maxscale.md)**

Maintenant que vous maîtrisez la gestion du quorum et la résolution de split-brain, la section suivante introduit **MaxScale**, le proxy intelligent qui :
- Détecte automatiquement les nœuds PRIMARY/NON-PRIMARY
- Route les connexions vers les nœuds sains
- Fournit le failover transparent pour les applications
- Offre des fonctionnalités avancées (Database Firewall, Query Routing)

MaxScale est la couche qui rend Galera réellement transparent pour les applications.

---

**Le split-brain n'est pas une fatalité. Une compréhension solide du quorum et des procédures de récupération bien rodées sont votre meilleure assurance.**

⏭️ [MaxScale](/14-haute-disponibilite/04-maxscale.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.2 MariaDB Galera Cluster

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Section 14.1, maîtrise de la réplication MariaDB, concepts de systèmes distribués

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture interne de Galera Cluster et le protocole wsrep
- **Différencier** Galera des solutions de réplication traditionnelles
- **Évaluer** les avantages et limitations pour vos use cases
- **Identifier** les prérequis réseau, matériel et logiciel pour un déploiement production
- **Planifier** une architecture Galera adaptée à vos contraintes (single-DC, multi-DC)
- **Anticiper** les challenges opérationnels et leurs solutions

---

## Introduction

**MariaDB Galera Cluster** représente une rupture paradigmatique dans le monde de la réplication MariaDB. Contrairement aux modèles master-slave traditionnels, Galera implémente une **réplication synchrone multi-master** basée sur la certification des transactions, offrant un **RPO de zéro** et une disponibilité quasi-continue.

Galera n'est pas simplement une couche de réplication ajoutée à MariaDB, c'est une **intégration profonde** au niveau du moteur de stockage via l'API wsrep (Write Set Replication). Cette architecture permet à tous les nœuds d'accepter simultanément des écritures tout en garantissant la cohérence des données à travers le cluster.

### 🔍 Pourquoi Galera ?

Dans un monde où l'indisponibilité coûte cher, Galera répond à des besoins critiques :

**Problèmes résolus** :
- ❌ **Perte de données** lors d'une panne master (réplication async)
- ❌ **Downtime prolongé** pendant un failover manuel
- ❌ **Complexité** de gestion des topologies master-slave
- ❌ **Sous-utilisation** des serveurs standby
- ❌ **Latence** de lecture sur les replicas (slave lag)

**Solutions Galera** :
- ✅ **RPO = 0** : Aucune perte de données garantie
- ✅ **RTO < 30 sec** : Failover automatique quasi-instantané
- ✅ **Simplicité** : Tous les nœuds sont égaux (peer-to-peer)
- ✅ **Utilisation 100%** : Tous les nœuds actifs simultanément
- ✅ **Cohérence** : Lectures toujours à jour (configurable)

> 💡 **Citation** : "Galera Cluster turns the database itself into a highly available service, not just a replicated data store." - Codership (créateurs de Galera)

---

## 1. Vue d'Ensemble de Galera Cluster

### 1.1 Architecture Fondamentale

```
┌────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                       │
│              ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│              │  App 1   │  │  App 2   │  │  App 3   │      │
│              └────┬─────┘  └────┬─────┘  └────┬─────┘      │
└───────────────────┼─────────────┼─────────────┼────────────┘
                    │             │             │
                    │    Load Balancing Layer   │
                    │    (MaxScale/HAProxy)     │
                    │             │             │
┌───────────────────┼─────────────┼─────────────┼────────────┐
│                   ▼             ▼             ▼            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────┐ │
│  │   GALERA NODE 1  │  │   GALERA NODE 2  │  │  NODE 3   │ │
│  │                  │  │                  │  │           │ │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │ ┌───────┐ │ │
│  │  │   MariaDB  │  │  │  │   MariaDB  │  │  │ │MariaDB│ │
│  │  │   Server   │◄─┼──┼─►│   Server   │◄─┼──┼►│Server │ │
│  │  └─────┬──────┘  │  │  └─────┬──────┘  │  │ └──┬────┘ │ │
│  │        │         │  │        │         │  │    │      │ │
│  │  ┌─────▼──────┐  │  │  ┌─────▼──────┐  │  │ ┌──▼───┐  │ │
│  │  │   wsrep    │  │  │  │   wsrep    │  │  │ │wsrep │  │ │
│  │  │  Provider  │  │  │  │  Provider  │  │  │ │Provdr│  │ │
│  │  │(Galera lib)│  │  │  │(Galera lib)│  │  │ │      │  │ │
│  │  └─────┬──────┘  │  │  └─────┬──────┘  │  │ └──┬───┘  │ │
│  │        │         │  │        │         │  │    │      │ │
│  │  ┌─────▼──────┐  │  │  ┌─────▼──────┐  │  │ ┌──▼───┐  │ │
│  │  │  InnoDB    │  │  │  │  InnoDB    │  │  │ │InnoDB│  │ │
│  │  │  Storage   │  │  │  │  Storage   │  │  │ │      │  │ │
│  │  └────────────┘  │  │  └────────────┘  │  │ └──────┘  │ │
│  └──────────────────┘  └──────────────────┘  └───────────┘ │
│                                                            │
│         Group Communication (gcomm://)                     │
│         ◄───────── Cluster Coordination ────────►          │
└────────────────────────────────────────────────────────────┘
```

**Composants Clés** :

1. **MariaDB Server** : Moteur de base standard
2. **wsrep API** : Interface de réplication (Write Set Replication)
3. **Galera Replication Plugin** : Bibliothèque libgalera_smm.so
4. **Group Communication** : Protocole de communication inter-nœuds (gcomm)
5. **Certification Layer** : Validation des transactions

### 1.2 Différences avec la Réplication Traditionnelle

| Aspect | Réplication Async | Réplication Semi-Sync | Galera Cluster |
|--------|-------------------|----------------------|----------------|
| **Topologie** | Master → Slave(s) | Master → Slave(s) | Peer-to-peer mesh |
| **Writes** | Master uniquement | Master uniquement | Tous les nœuds |
| **Synchronisation** | Asynchrone (lag) | Commit Master + 1 ACK | Synchrone globale |
| **Validation** | Après application | Avant commit master | Avant commit cluster |
| **Cohérence** | Éventuelle | Forte (1 slave min) | Stricte (tous nœuds) |
| **Perte données** | Possible | Très rare | Impossible |
| **Latence commit** | Faible | Moyenne | Moyenne-Haute |
| **Conflits** | Impossibles | Impossibles | Possibles (rares) |
| **Failover** | Manuel/Scripts | Manuel/Scripts | Automatique |
| **Complexité** | Faible | Faible | Moyenne-Haute |

### 1.3 Versions et Composants

**Galera Cluster disponible en deux variantes** :

#### **MariaDB Galera Cluster (Intégré)**
```bash
# Déjà intégré dans MariaDB depuis 10.1
apt-get install mariadb-server galera-4

# Vérification version
mysql -e "SHOW STATUS LIKE 'wsrep_provider_version'"
# +-----------------------+---------+
# | Variable_name         | Value   |
# +-----------------------+---------+
# | wsrep_provider_version| 4.16    |
# +-----------------------+---------+
```

**Versions Galera** :
- **Galera 3.x** : Legacy (MariaDB 10.1-10.3)
- **Galera 4.x** : Current (MariaDB 10.4+) 🆕
  - Streaming Replication (grandes transactions)
  - Parallel applying amélioré
  - Automatic IST optimisé

#### **Codership Galera (Standalone)**
Alternative maintenue par les créateurs originaux de Galera :
```bash
# Installation alternative
wget https://releases.galeracluster.com/galera-4/galera-4-26.4.16.tar.gz
```

💡 **Recommandation** : Utiliser MariaDB Galera Cluster intégré pour simplifier la maintenance.

---

## 2. Concepts Fondamentaux

### 2.1 Write Set Replication (wsrep)

Galera n'utilise pas les binary logs traditionnels pour la réplication. À la place, il implémente **wsrep** :

**Workflow d'une transaction** :

```
1. Client envoie: BEGIN; UPDATE ...; COMMIT;
                        │
                        ▼
2. Exécution locale (optimistic execution)
   ┌─────────────────────────────────────┐
   │  Transaction exécutée en local      │
   │  Modifications en mémoire InnoDB    │
   └─────────────────────────────────────┘
                        │
                        ▼
3. Pré-commit: Création du Write Set
   ┌─────────────────────────────────────┐
   │  Write Set = {                      │
   │    - Rows modifiées (PK)            │
   │    - Valeurs BEFORE/AFTER           │
   │    - Transaction metadata           │
   │    - Clés primaires touchées        │
   │  }                                  │
   └─────────────────────────────────────┘
                        │
                        ▼
4. Broadcast au cluster (TOTEM protocol)
   ┌─────────────────────────────────────┐
   │  Envoi Write Set à tous les nœuds   │
   │  via Group Communication            │
   └─────────────────────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ Node 1  │    │ Node 2  │    │ Node 3  │
   └─────────┘    └─────────┘    └─────────┘
         │              │              │
         └──────────────┼──────────────┘
                        ▼
5. Certification (sur tous les nœuds)
   ┌─────────────────────────────────────┐
   │  Vérification conflits:             │
   │  - Même PK modifiée simultanément?  │
   │  - Violations contraintes?          │
   │  - Deadlock potentiel?              │
   └─────────────────────────────────────┘
                        │
         ┌──────────────┴──────────────┐
         ▼                             ▼
    ✅ PASS                         ❌ FAIL
         │                             │
         ▼                             ▼
   ┌─────────┐                   ┌─────────┐
   │ COMMIT  │                   │ROLLBACK │
   │ (local) │                   │ (local) │
   └─────────┘                   └─────────┘
         │                             │
         ▼                             ▼
   ┌─────────────────┐           ┌─────────────────┐
   │ Apply sur       │           │ Client reçoit   │
   │ autres nœuds    │           │ ER_LOCK_DEADLOCK│
   │ (async après)   │           │                 │
   └─────────────────┘           └─────────────────┘
```

**Caractéristiques clés** :
- ✅ **Exécution optimiste** : Transaction d'abord exécutée localement
- ✅ **Certification distribuée** : Validation parallèle sur tous les nœuds
- ✅ **Ordre déterministe** : Total Order Broadcast garantit même ordre partout
- ✅ **Rollback automatique** : Si conflit détecté, rollback local immédiat

### 2.2 Virtual Synchrony et Group Communication

Galera utilise un modèle de **synchronie virtuelle** :

**Garanties** :
1. **Total Order** : Tous les nœuds reçoivent les write sets dans le même ordre
2. **Atomic Broadcast** : Un message est soit reçu par tous, soit par aucun
3. **View Synchrony** : Tous les nœuds ont la même vision du cluster

**Protocoles utilisés** :
```
┌─────────────────────────────────────────┐
│   Application Layer (MariaDB/wsrep)     │
├─────────────────────────────────────────┤
│   Certification Based Replication       │
├─────────────────────────────────────────┤
│   EVS (Extended Virtual Synchrony)      │
│   - Membership management               │
│   - Message ordering                    │
├─────────────────────────────────────────┤
│   gcomm (Group Communication)           │
│   - TOTEM: Message delivery             │
│   - UDP/TCP transport                   │
└─────────────────────────────────────────┘
```

### 2.3 État du Cluster (Cluster State)

Chaque nœud maintient un **état de cluster** :

**Variables wsrep critiques** :
```sql
-- Taille du cluster
SHOW STATUS LIKE 'wsrep_cluster_size';
-- Doit être égal au nombre total de nœuds (ex: 3, 5, 7)

-- Statut du nœud local
SHOW STATUS LIKE 'wsrep_local_state_comment';
-- Valeurs possibles:
--   'Joining'    : Nœud en cours de jonction (SST/IST)
--   'Donor'      : Nœud donneur pour SST/IST
--   'Joined'     : Synchronisé mais pas ready
--   'Synced'     : Prêt à recevoir des transactions ✅
--   'Desync'     : Temporairement désynchro (ex: backup)

-- Numéro de vue du cluster
SHOW STATUS LIKE 'wsrep_cluster_conf_id';
-- Incrémenté à chaque changement de membership

-- Statut de connexion
SHOW STATUS LIKE 'wsrep_connected';
-- ON = connecté au cluster
```

**Diagramme des États** :
```
    ┌──────────┐
    │  OPEN    │ ← Démarrage initial
    └────┬─────┘
         │
         ▼
    ┌──────────┐
    │ PRIMARY  │ ← Cluster a le quorum
    └────┬─────┘
         │
    ┌────┴─────┐
    │          │
    ▼          ▼
┌────────┐  ┌────────┐
│JOINER  │  │ DONOR  │
└────┬───┘  └────────┘
     │
     ▼
┌────────┐
│ JOINED │
└────┬───┘
     │
     ▼
┌────────┐
│ SYNCED │ ← État opérationnel ✅
└────┬───┘
     │
     ▼
┌────────┐
│DONOR/  │ ← Fournit SST/IST
│DESYNC  │
└────────┘
```

---

## 3. Avantages et Bénéfices

### 3.1 Avantages Techniques

#### **1. RPO = 0 (Zero Data Loss)**
```sql
-- Configuration pour garantir RPO zéro
[galera]
wsrep_sync_wait = 1  -- Attendre sync avant retourner résultat
```

**Garantie** : Si `COMMIT` réussit, données présentes sur tous les nœuds actifs.

**Exemple de reprise après panne** :
```
# Avant panne
Node1: Transaction T1 commitée ✅
Node2: Transaction T1 commitée ✅
Node3: Transaction T1 commitée ✅

# Panne brutale Node1 (crash disque)
Node1: ❌ OFFLINE

# Après failover
Node2: Transaction T1 présente ✅
Node3: Transaction T1 présente ✅
→ Aucune perte de données
```

#### **2. RTO Minimal (< 30 secondes)**

**Détection automatique de panne** :
```ini
# Configuration réseau optimisée
wsrep_provider_options = "
    evs.keepalive_period = PT1S;      # Heartbeat chaque seconde
    evs.suspect_timeout = PT5S;       # Suspect après 5s
    evs.inactive_timeout = PT15S;     # Éviction après 15s
    evs.install_timeout = PT15S;      # Timeout installation vue
"
```

**Timeline failover typique** :
```
T+0s   : Node1 crash (panne matérielle)
T+1s   : Dernière réponse heartbeat manquée
T+5s   : Node1 marqué "suspect" par Node2 et Node3
T+10s  : Node1 évincé du cluster
T+12s  : Nouvelle vue cluster installée (size=2)
T+15s  : Applications redirigées vers Node2/Node3
         (via MaxScale/HAProxy)
→ RTO = 15 secondes
```

#### **3. Multi-Master Actif**

Contrairement au master-slave, **tous les nœuds acceptent des writes** :

```sql
-- Application 1 écrit sur Node1
-- Transaction simultanée
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 123;
COMMIT; -- ✅ Succès

-- Application 2 écrit sur Node2 (même temps)
-- Transaction différente
BEGIN;
UPDATE accounts SET balance = balance + 50 WHERE user_id = 456;
COMMIT; -- ✅ Succès

-- Les deux transactions répliquées sur tous les nœuds
-- Ordre déterministe garanti par Total Order Broadcast
```

**Bénéfice** : Pas de SPOF au niveau write, scalabilité horizontale partielle.

#### **4. Cohérence Stricte**

```sql
-- Configuration cohérence maximale
SET GLOBAL wsrep_sync_wait = 7;  -- Bitmask complet
-- Bit 1 (1)  : READ
-- Bit 2 (2)  : UPDATE/DELETE
-- Bit 4 (4)  : INSERT/REPLACE

-- Avec wsrep_sync_wait=7, garantie:
SELECT * FROM users WHERE id = 123;
-- ✅ Retourne TOUJOURS la dernière version commitée cluster-wide
```

**Trade-off** : Latence accrue (+5-50ms selon réseau) vs cohérence parfaite.

### 3.2 Avantages Opérationnels

#### **1. Maintenance Sans Downtime**

**Rolling upgrade** :
```bash
# Upgrade Node1
systemctl stop mysql@node1
apt-get upgrade mariadb-server
systemctl start mysql@node1  # Rejoint automatiquement via IST

# Cluster reste opérationnel sur Node2+Node3 pendant l'upgrade
# Répéter pour Node2, puis Node3

→ Zero downtime upgrade
```

#### **2. Backup Sans Impact**

```bash
# Backup depuis nœud donneur sans bloquer le cluster
mysql -e "SET GLOBAL wsrep_desync = ON"  # Node3 se désynchro temporairement

# Backup physique
mariabackup --backup --target-dir=/backup/full

mysql -e "SET GLOBAL wsrep_desync = OFF"  # Resynchronisation automatique (IST)

# Cluster a continué à fonctionner sur Node1+Node2
```

#### **3. Élasticité Dynamique**

```bash
# Ajout d'un nouveau nœud à chaud
# Node4 démarre et rejoint automatiquement
systemctl start mysql@node4

# SST automatique depuis Node1 (donneur choisi par Galera)
# Après sync, Node4 devient actif
# Cluster passe de 3 à 4 nœuds sans interruption
```

---

## 4. Limitations et Contraintes

### 4.1 Limitations Techniques

#### **1. Scalabilité Writes Limitée**

**Problème** : Chaque write doit être répliqué et certifié sur **tous** les nœuds.

```
Writes/sec maximum ≈ (Single node writes) × 0.7-0.9

Exemple:
- 1 nœud : 5000 writes/sec
- 3 nœuds Galera : ~4000 writes/sec (80%)
- 5 nœuds Galera : ~3500 writes/sec (70%)

→ Anti-scaling pour writes au-delà de 5 nœuds
```

💡 **Mitigation** : Sharding applicatif, optimisation requêtes, caching.

#### **2. Contraintes Réseau Strictes**

**Exigences réseau** :
```
Latence:
- Single DC : < 1ms (idéal)
- Single DC : < 5ms (acceptable)
- Multi-DC : < 10ms (LAN extension)
- Multi-DC : 10-100ms (possiblewith tuning)
- > 100ms : Non recommandé

Bande passante:
- Minimum : 100 Mbps
- Recommandé : 1 Gbps
- Multi-DC : 100 Mbps dédié inter-DC

Perte de paquets:
- < 0.1% impératif
- Jitter < 10ms
```

**Test réseau** :
```bash
# Entre tous les nœuds
ping -c 100 node2.example.com | tail -1
# rtt min/avg/max/mdev = 0.234/0.456/1.234/0.123 ms
# → avg < 1ms ✅

iperf3 -c node2.example.com
# [ ID] Interval           Transfer     Bitrate
# [  5]   0.00-10.00  sec  1.09 GBytes   938 Mbits/sec
# → > 100 Mbps ✅
```

#### **3. Moteurs de Stockage Supportés**

```sql
-- Uniquement InnoDB et XtraDB supportés pour réplication
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB;  -- ✅ OK

CREATE TABLE logs (
    message TEXT
) ENGINE=MyISAM;  -- ❌ Ne sera PAS répliqué

SHOW WARNINGS;
-- Warning: wsrep_mode does not support MyISAM
```

⚠️ **Critical** : MyISAM, Memory, Archive non répliqués → risque de divergence.

#### **4. Transactions DDL (ALTER TABLE)**

**Limitations DDL** :
```sql
-- Les DDL sont répliqués mais bloquent le cluster
ALTER TABLE large_table ADD COLUMN email VARCHAR(255);

-- Impact:
-- 1. TOI (Total Order Isolation) par défaut
--    → Bloque TOUS les nœuds pendant l'exécution
--    → Si ALTER prend 5 minutes, cluster bloqué 5 min

-- 2. Alternative: RSU (Rolling Schema Upgrade)
SET SESSION wsrep_OSU_method='RSU';
ALTER TABLE large_table ADD COLUMN email VARCHAR(255);
-- → Exécuté uniquement localement
-- → Nécessite répéter sur chaque nœud manuellement
```

💡 **Best Practice** : Utiliser `pt-online-schema-change` ou `gh-ost`.

#### **5. Impossibilité de LOCK TABLES Global**

```sql
-- Ne fonctionne PAS comme attendu en Galera
LOCK TABLES users WRITE;
-- Verrouille uniquement le nœud local
-- Autres nœuds peuvent toujours modifier 'users'

-- Résultat: Possibilité de deadlock distribué
```

### 4.2 Contraintes Opérationnelles

#### **1. Quorum Strict (Minimum 3 Nœuds)**

```
2 nœuds: Dangereux - perte d'un nœud = perte quorum = cluster DOWN
3 nœuds: Minimum viable - tolère 1 panne
5 nœuds: Recommandé production - tolère 2 pannes
```

**Coût quorum** :
```
3 nœuds × (Serveur + Storage + Réseau) = Coût élevé
vs
1 Master + 1 Standby = Coût modéré
```

#### **2. Complexité de Troubleshooting**

**Scénarios complexes** :
- Split-brain multi-segments
- Deadlocks distribués
- Slow joiner (nœud lent retarde cluster)
- Flow control (backpressure automatique)

**Expertise requise** : DBA senior avec expérience systèmes distribués.

#### **3. Conflits d'Écriture**

```sql
-- Conflit de certification possible (rare mais existe)
-- Node1
BEGIN;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 99;
-- Local quantity: 5 → 4
COMMIT;

-- Node2 (simultané, même milliseconde)
BEGIN;
UPDATE inventory SET quantity = quantity - 2 WHERE product_id = 99;
-- Local quantity: 5 → 3
COMMIT;

-- Résultat certification:
-- Node1 COMMIT: ✅ Success (arrivé en premier)
-- Node2 COMMIT: ❌ Deadlock (conflit détecté, rollback)

-- Application Node2 reçoit:
-- ERROR 1213 (40001): Deadlock found when trying to get lock
```

**Fréquence** : < 0.01% des transactions dans un cluster bien conçu, mais nécessite gestion applicative (retry).

---

## 5. Prérequis et Préparation

### 5.1 Prérequis Matériels

**Configuration minimale par nœud** :
```
CPU:      4 cores (8+ recommandé)
RAM:      16 GB (32+ recommandé)
          - Buffer Pool: 70% RAM
          - wsrep threads: 1-2 GB
          
Disque:   SSD obligatoire pour performance
          - IOPS: > 5000 4K random reads
          - Latency: < 1ms
          
Réseau:   1 Gbps (10 Gbps recommandé)
          - Latence < 1ms intra-DC
          - Dédié VLAN pour Galera recommandé
```

**Calcul dimensionnement** :
```python
# Calcul RAM pour Galera
data_size_gb = 500  # Taille totale données

# InnoDB Buffer Pool (70-80% RAM)
buffer_pool_gb = data_size_gb * 0.8

# wsrep overhead
wsrep_slave_threads = 16  # Nombre de cores
wsrep_overhead_gb = 2

# RAM totale
total_ram_gb = buffer_pool_gb + wsrep_overhead_gb + 4 # (4GB OS)
# = 500 * 0.8 + 2 + 4 = 406 GB

→ Serveur avec 512 GB RAM recommandé
```

### 5.2 Prérequis Réseau

**Ports requis** :
```bash
# Port Galera Cluster (TCP)
3306    : MySQL/MariaDB
4567    : Galera Cluster (wsrep)
4568    : IST (Incremental State Transfer)
4444    : SST (State Snapshot Transfer)

# Configuration firewall
ufw allow from 10.0.1.0/24 to any port 3306 proto tcp
ufw allow from 10.0.1.0/24 to any port 4567 proto tcp
ufw allow from 10.0.1.0/24 to any port 4568 proto tcp
ufw allow from 10.0.1.0/24 to any port 4444 proto tcp
```

**Validation réseau** :
```bash
#!/bin/bash
# check_network.sh - Vérifier prérequis réseau

NODES=("node1:10.0.1.10" "node2:10.0.1.11" "node3:10.0.1.12")

for node in "${NODES[@]}"; do
    name="${node%%:*}"
    ip="${node##*:}"
    
    echo "Testing $name ($ip)..."
    
    # Latency
    latency=$(ping -c 10 $ip | tail -1 | awk -F'/' '{print $5}')
    echo "  Latency: ${latency}ms"
    
    if (( $(echo "$latency > 5" | bc -l) )); then
        echo "  ⚠️ WARNING: Latency > 5ms"
    fi
    
    # Port accessibility
    nc -zv $ip 4567 2>&1 | grep -q succeeded
    [ $? -eq 0 ] && echo "  ✅ Port 4567 accessible" || echo "  ❌ Port 4567 blocked"
    
done
```

### 5.3 Prérequis Logiciels

**Stack recommandée** :
```bash
# OS
Ubuntu 24.04 LTS / Debian 12 / RHEL 9

# MariaDB
MariaDB 11.8 LTS (ou 11.4 LTS minimum)

# Galera
galera-4 (version 4.16+)

# Outils
- mariabackup (pour SST)
- socat (pour streaming SST)
- pv (pipe viewer, monitoring transferts)
```

**Vérification compatibilité** :
```sql
-- Vérifier support Galera
SHOW VARIABLES LIKE 'wsrep_on';
-- wsrep_on = ON ✅

SHOW GLOBAL STATUS LIKE 'wsrep_provider_version';
-- wsrep_provider_version = 4.16 ✅

-- Vérifier InnoDB (requis)
SHOW VARIABLES LIKE 'default_storage_engine';
-- default_storage_engine = InnoDB ✅
```

---

## 6. Architecture de Déploiement

### 6.1 Single Datacenter (3 nœuds)

```
┌────────────────── DATACENTER ──────────────────┐
│                                                │
│  ┌──────────────┐  ┌──────────────┐            │
│  │  MaxScale 1  │  │  MaxScale 2  │            │
│  │   (Active)   │  │  (Standby)   │            │
│  └──────┬───────┘  └──────┬───────┘            │
│         │                 │                    │
│         └────────┬────────┘                    │
│                  │ Virtual IP                  │
│                  │                             │
│  ┌───────────────┼────────────────┐            │
│  │               │                │            │
│  │      ┌────────▼────────┐       │            │
│  ▼      ▼                 ▼       ▼            │
│ ┌─────┐ ┌─────┐         ┌─────┐ ┌─────┐        │
│ │Node1│─│Node2│─────────│Node3│ │     │        │
│ └─────┘ └─────┘         └─────┘ │Load │        │
│    │       │               │    │Balan│        │
│    └───────┼───────────────┘    │cer  │        │
│            │ Galera Cluster     └─────┘        │
│            │ (All-to-All Mesh)                 │
│                                                │
└────────────────────────────────────────────────┘
```

**Caractéristiques** :
- ✅ Haute disponibilité intra-DC
- ✅ RTO < 30 secondes
- ✅ RPO = 0
- ⚠️ Vulnérable à défaillance datacenter

### 6.2 Multi-Datacenter (5 nœuds + Arbitre)

```
┌──────────── DC1 (Paris) ─────────────┐
│  ┌─────┐  ┌─────┐                    │
│  │Node1│──│Node2│                    │
│  └──┬──┘  └──┬──┘                    │
│     │        │                       │
└─────┼────────┼───────────────────────┘
      │        │
      │   WAN  │ (10-50ms)
      │        │
┌─────┼────────┼──────── DC2 (London) ─┐
│     │        │                       │
│  ┌──▼──┐  ┌─▼───┐                    │
│  │Node3│──│Node4│                    │
│  └─────┘  └─────┘                    │
│                                      │
└──────────────────────────────────────┘
           │
           │ WAN
           │
┌──────────▼──────── DC3 (Dublin) ─────┐
│      ┌─────┐                         │
│      │Garbd│ (Arbitrator only)       │
│      └─────┘                         │
└──────────────────────────────────────┘
```

**Avantages** :
- ✅ Protection disaster recovery
- ✅ Compliance multi-région
- ✅ Quorum maintenu avec garbd
- ⚠️ Latence writes accrue (WAN)

---

## 7. Cas d'Usage Recommandés

### 7.1 Excellents Cas d'Usage

**1. Applications Transactionnelles Critiques**
- E-commerce (catalogue, commandes)
- Banking (transactions, comptes)
- SaaS B2B (données clients)

**2. Haute Disponibilité Stricte**
- Services 24/7 (télécommunications)
- Applications médicales
- Services publics critiques

**3. Lectures Intensives avec Writes Modérés**
- CMS (Content Management)
- Portails d'information
- Applications mobiles (backend)

### 7.2 Cas d'Usage à Éviter

**1. Writes Très Intensifs**
```
> 10 000 writes/sec sustained
→ Considérer sharding ou autre solution
```

**2. Grandes Transactions**
```
Transactions > 1 GB writesets
→ Risque de timeout certification
→ Utiliser Streaming Replication (Galera 4.x)
```

**3. Schémas avec Beaucoup de DDL**
```
Fréquents ALTER TABLE
→ Bloquerait le cluster (TOI)
→ Considérer architecture différente
```

**4. Applications Incompatibles Galera**
- Utilisation extensive MyISAM
- LOCK TABLES global requis
- Applications non idempotentes (pas de retry)

---

## ✅ Points Clés à Retenir

- **Galera = réplication synchrone multi-master** basée sur certification de write sets
- **RPO = 0** garanti : aucune perte de données si COMMIT réussit
- **RTO < 30 sec** avec détection automatique et failover transparent
- **Tous les nœuds actifs** : pas de gaspillage de ressources comme avec standby
- **wsrep API** : intégration profonde MariaDB, pas une simple couche réplication
- **Cohérence stricte** configurable via `wsrep_sync_wait`
- **Limitations réseau critiques** : latence < 10ms recommandée, < 100ms maximum
- **Minimum 3 nœuds** pour quorum (5 recommandé production)
- **Scalabilité reads excellente**, writes limitée par certification globale
- **InnoDB uniquement** pour tables répliquées

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 MariaDB Galera Cluster Documentation](https://mariadb.com/kb/en/galera-cluster/)
- [📖 Codership Galera Documentation](https://galeracluster.com/library/documentation/)
- [📖 wsrep API Specification](https://github.com/codership/wsrep-API)

### Whitepapers
- **"Understanding Galera Cluster"** - Codership AB
- **"Certification-Based Replication"** - Academic Paper
- **"Galera Performance Best Practices"** - MariaDB Corporation

### Outils
- [Galera Arbitrator (garbd)](https://galeracluster.com/library/documentation/arbitrator.html)
- [myq_gadgets](https://github.com/jayjanssen/myq_gadgets) - Monitoring Galera
- [Percona Monitoring Plugins](https://www.percona.com/software/database-tools/percona-monitoring-plugins)

---

## ➡️ Sections Suivantes

Cette introduction a posé les fondations conceptuelles de Galera Cluster. Les sections suivantes approfondiront :

- **14.2.1** : Architecture synchrone multi-master (détails protocole wsrep)
- **14.2.2** : Certification-based replication (algorithmes, conflits)
- **14.2.3** : Configuration et déploiement production
- **14.2.4** : State transfers (SST vs IST, optimisations)

Chaque sous-section construira sur ces concepts pour vous permettre de déployer et opérer un cluster Galera production-ready.

---

**Galera Cluster représente un saut qualitatif dans la haute disponibilité MariaDB. Maîtriser ses concepts est essentiel pour tout architecte ou DBA gérant des applications critiques.**

⏭️ [Architecture synchrone multi-master](/14-haute-disponibilite/02.1-architecture-synchrone.md)

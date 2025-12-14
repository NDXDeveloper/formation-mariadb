🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.1 Architectures Haute Disponibilité : Concepts

> **Niveau** : Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Maîtrise de la réplication, expérience en architecture distribuée

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes fondamentaux de la haute disponibilité et le théorème CAP
- **Évaluer** les différents patterns d'architecture HA et leurs trade-offs
- **Calculer** et interpréter les métriques de disponibilité (RTO, RPO, MTBF, MTTR)
- **Concevoir** une architecture adaptée aux contraintes métier et techniques
- **Anticiper** les points de défaillance (SPOF) et les mitiger
- **Établir** un framework de décision pour le choix d'architecture

---

## Introduction

La haute disponibilité n'est pas un produit ou une technologie unique, mais une **stratégie architecturale globale** visant à minimiser l'indisponibilité des services critiques. Dans le contexte MariaDB, cela implique de comprendre les fondamentaux théoriques avant de choisir et implémenter une solution technique.

> 💡 **Citation clé** : "La haute disponibilité est un voyage, pas une destination. C'est un équilibre constant entre coûts, complexité et besoins métier." - Werner Vogels, CTO Amazon

---

## 1. Fondamentaux de la Haute Disponibilité

### 1.1 Définitions Essentielles

#### **Disponibilité (Availability)**

La disponibilité mesure le pourcentage de temps où un système est opérationnel et accessible :

```
Disponibilité = (Temps d'uptime / Temps total) × 100
```

**Niveaux de disponibilité standard** :

| Niveau | Disponibilité | Downtime/an | Downtime/mois | Use Case |
|--------|---------------|-------------|---------------|----------|
| **Two Nines** | 99% | 3.65 jours | 7.2 heures | Dev/Test |
| **Three Nines** | 99.9% | 8.76 heures | 43.2 minutes | Applications standard |
| **Four Nines** | 99.99% | 52.56 minutes | 4.32 minutes | Applications critiques |
| **Five Nines** | 99.999% | 5.26 minutes | 25.9 secondes | Services financiers, santé |
| **Six Nines** | 99.9999% | 31.5 secondes | 2.59 secondes | Télécommunications |

#### **RTO (Recovery Time Objective)**

Le **temps maximal acceptable** entre la défaillance et la restauration complète du service :

```
RTO = Temps de détection + Temps de décision + Temps de récupération
```

**Exemples concrets** :
- **E-commerce** : RTO = 5 minutes (perte revenue estimée : 10k€/minute)
- **Banking** : RTO = 30 secondes (exigences réglementaires)
- **SaaS B2B** : RTO = 15 minutes (SLA contractuel 99.9%)

#### **RPO (Recovery Point Objective)**

La **perte de données maximale acceptable** mesurée en temps :

```
RPO = Dernière sauvegarde valide - Point de défaillance
```

**Exemples concrets** :
- **Transactions financières** : RPO = 0 (aucune perte acceptable)
- **Analytics** : RPO = 1 heure (données batch)
- **Logs applicatifs** : RPO = 24 heures (rechargeable)

#### **MTBF et MTTR**

**MTBF (Mean Time Between Failures)** : Temps moyen entre deux pannes  
**MTTR (Mean Time To Repair)** : Temps moyen de réparation

```
Disponibilité = MTBF / (MTBF + MTTR)
```

**Exemple de calcul** :
```
MTBF = 720 heures (30 jours)
MTTR = 0.5 heures (30 minutes)
Disponibilité = 720 / (720 + 0.5) = 99.93%
```

💡 **Insight** : Augmenter MTBF (fiabilité) est souvent plus efficace que réduire MTTR (rapidité).

---

### 1.2 Le Théorème CAP

Le théorème CAP (aussi appelé théorème de Brewer) stipule qu'un système distribué ne peut garantir simultanément que **deux** des trois propriétés suivantes :

#### **C - Consistency (Cohérence)**
Tous les nœuds voient les mêmes données au même moment. Toute lecture retourne la dernière écriture.

#### **A - Availability (Disponibilité)**
Chaque requête reçoit une réponse (succès ou échec), même si certains nœuds sont défaillants.

#### **P - Partition Tolerance (Tolérance au partitionnement)**
Le système continue de fonctionner malgré la perte de messages réseau entre nœuds.

```
        CAP Theorem
           /\
          /  \
         /    \
        /  CP  \
       /________\
      /\        /\
     /  \  CA  /  \
    / AP \____/ CP \
   /______\  /______\
  
  CA : Cohérence + Disponibilité (impossible en cas de partition)
  CP : Cohérence + Partition (sacrifice de la disponibilité)
  AP : Disponibilité + Partition (cohérence éventuelle)
```

#### **Positionnement des Technologies MariaDB**

| Technologie | Type CAP | Justification |
|-------------|----------|---------------|
| **Standalone** | CA | Cohérent et disponible jusqu'à panne serveur |
| **Réplication Async** | AP | Disponible, cohérence éventuelle |
| **Réplication Semi-sync** | CP | Privilégie cohérence, peut bloquer si replica down |
| **Galera Cluster** | CP | Cohérence stricte, indisponible si pas de quorum |
| **Multi-DC Galera** | AP/CP | Configurable selon wsrep_sync_wait |

💡 **En pratique** : Les partitions réseau étant inévitables dans les systèmes distribués, le choix se résume à **CP vs AP**.

#### **Implications pour MariaDB**

**Choix CP (Galera en mode strict)** :
```sql
-- Configuration pour cohérence maximale
SET GLOBAL wsrep_sync_wait = 7; -- Attendre synchronisation complète
SET GLOBAL wsrep_causal_reads = ON;
```
- ✅ Lectures toujours cohérentes
- ✅ Pas de divergence possible
- ❌ Indisponibilité si perte de quorum
- ❌ Latence accrue

**Choix AP (Réplication async)** :
```sql
-- Configuration pour disponibilité maximale
SET GLOBAL read_only = 0 ON replicas; -- Lecture sur replicas
-- Promotion automatique en cas de panne master
```
- ✅ Toujours disponible en lecture
- ✅ Faible latence
- ❌ Possible slave lag
- ❌ Lectures potentiellement obsolètes

---

## 2. Patterns d'Architectures Haute Disponibilité

### 2.1 Active-Passive (Primary-Standby)

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │
       ▼
┌──────────────┐     Réplication      ┌──────────────┐
│   PRIMARY    │ ─────────────────→   │   STANDBY    │
│  (Active)    │     Asynchrone       │  (Passive)   │
└──────────────┘                      └──────────────┘
     ↓ Heartbeat                            ↑
     ↓                                      │
     └─────── Failover (manuel/auto) ───────┘
```

#### **Caractéristiques**

**Avantages** :
- ✅ Simple à comprendre et implémenter
- ✅ Pas de risque de conflits d'écriture
- ✅ Compatible avec réplication async standard
- ✅ Coût modéré (2 serveurs minimum)

**Inconvénients** :
- ❌ Standby inutilisé (50% de ressources gaspillées)
- ❌ RPO > 0 (dépend du replication lag)
- ❌ RTO de minutes (détection + bascule)
- ❌ Nécessite scripts de failover

#### **Configuration de Production**

```ini
# my.cnf sur PRIMARY
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

# Réplication semi-synchrone pour réduire RPO
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_master_timeout = 10000 # 10 secondes

# my.cnf sur STANDBY
[mysqld]
server-id = 2
relay-log = /var/log/mysql/relay-bin.log
read_only = 1
rpl_semi_sync_slave_enabled = 1

# Activation réplication
CHANGE MASTER TO
  MASTER_HOST = 'primary.example.com',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'secure_password',
  MASTER_AUTO_POSITION = 1; -- Utilise GTID

START SLAVE;
```

#### **Failover Automatique avec Keepalived**

```bash
# /etc/keepalived/keepalived.conf
vrrp_script check_mysql {
    script "/usr/local/bin/check_mysql.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101  # 100 sur standby
    advert_int 1
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0
    }
    
    track_script {
        check_mysql
    }
    
    notify_master "/usr/local/bin/promote_to_master.sh"
}
```

```bash
#!/bin/bash
# check_mysql.sh
mysqladmin ping -h localhost -u monitor -p${MONITOR_PASSWORD} &>/dev/null
exit $?
```

#### **Métriques Typiques**

- **RTO** : 1-5 minutes (automatique), 5-30 minutes (manuel)
- **RPO** : 0-60 secondes (semi-sync), jusqu'à plusieurs minutes (async)
- **Disponibilité théorique** : 99.9% - 99.95%
- **Coût** : Faible à moyen

---

### 2.2 Active-Active (Multi-Master)

```
┌─────────────┐        ┌─────────────┐
│ Application │        │ Application │
└──────┬──────┘        └──────┬──────┘
       │                      │
       ▼                      ▼
┌──────────────┐ Réplication ┌──────────────┐
│   MASTER 1   │ ←─────────→ │   MASTER 2   │
│  (Active)    │  Synchrone  │  (Active)    │
└──────────────┘             └──────────────┘
       ↕                            ↕
   Réplication                 Réplication
       ↕                            ↕
┌──────────────┐             ┌──────────────┐
│   MASTER 3   │ ←─────────→ │   MASTER 4   │
│  (Active)    │             │  (Active)    │
└──────────────┘             └──────────────┘
```

#### **Implémentation avec Galera Cluster**

**Principe de Certification-Based Replication** :
1. Transaction exécutée localement
2. Diffusion (broadcast) aux autres nœuds pour certification
3. Validation si pas de conflit
4. Application ou rollback sur tous les nœuds

```sql
-- Configuration Galera optimale production
[galera]
wsrep_provider = /usr/lib/libgalera_smm.so
wsrep_cluster_address = "gcomm://node1,node2,node3,node4,node5"
wsrep_cluster_name = "production_cluster"

# Optimisations performance
wsrep_slave_threads = 16  # Nombre de CPU cores
wsrep_provider_options = "gcache.size=2G;gcs.fc_limit=128"

# Sécurité
wsrep_sst_method = mariabackup
wsrep_sst_auth = "sst_user:secure_password"

# Cohérence
wsrep_sync_wait = 1  # Attendre sync avant SELECT
wsrep_causal_reads = ON

# Optimisation réseau
wsrep_provider_options = "gmcast.segment=0;evs.keepalive_period=PT1S"
```

#### **Avantages Active-Active**

- ✅ **RPO = 0** : Synchronisation avant commit
- ✅ **RTO < 30 secondes** : Bascule automatique
- ✅ **Utilisation 100%** : Tous les nœuds actifs
- ✅ **Scalabilité reads** : Linéaire avec le nombre de nœuds
- ✅ **Pas de SPOF** : N'importe quel nœud peut être primary

#### **Inconvénients et Défis**

- ❌ **Scalabilité writes limitée** : Chaque write répliqué sur tous
- ❌ **Latence réseau critique** : Recommandé < 10ms inter-nœuds
- ❌ **Complexité opérationnelle** : Split-brain, quorum
- ❌ **Conflits possibles** : Rollback en cas de conflit de certification
- ❌ **Coût élevé** : Minimum 3 nœuds (5 recommandé)

#### **Gestion des Conflits**

```sql
-- Exemple de conflit de certification
-- Nœud 1                          Nœud 2
BEGIN;                              BEGIN;
UPDATE accounts                     UPDATE accounts
SET balance = balance - 100         SET balance = balance - 50
WHERE id = 123;                     WHERE id = 123;
-- Commit envoyé                   -- Commit envoyé
COMMIT;                             COMMIT;
-- ✅ Validé (arrivé en premier)   -- ❌ ROLLBACK (conflit détecté)
```

**Stratégies d'atténuation** :
```sql
-- 1. Sharding applicatif (sticky sessions)
-- Diriger user_id impair → node1, pair → node2

-- 2. Optimistic locking
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 123 AND version = 5;

-- 3. Utiliser AUTO_INCREMENT avec offset
SET GLOBAL auto_increment_offset = 1;  -- Node 1
SET GLOBAL auto_increment_increment = 5; -- Nb total de nœuds
```

#### **Métriques Typiques**

- **RTO** : 5-30 secondes (détection + routing)
- **RPO** : 0 (synchrone)
- **Disponibilité théorique** : 99.99% - 99.999%
- **Coût** : Élevé

---

### 2.3 Multi-Datacenter (Geo-Distribution)

```
┌─────────────────── DATACENTER 1 (Paris) ────────────────────┐
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ Galera 1 │──│ Galera 2 │──│ MaxScale │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
└──────────────────────┬──────────────────────────────────────┘
                       │ WAN (50-100ms)
                       │
┌──────────────────── DATACENTER 2 (Londres) ─────────────────┐
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ Galera 3 │──│ Galera 4 │──│ MaxScale │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
└──────────────────────┬──────────────────────────────────────┘
                       │ WAN (80-120ms)
                       │
┌──────────────────── DATACENTER 3 (Dublin - Arbitre) ────────┐
│  ┌──────────┐                                               │
│  │ Garbd    │  (Pas de données, uniquement quorum)          │
│  └──────────┘                                               │
└─────────────────────────────────────────────────────────────┘
```

#### **Configuration Multi-DC Galera**

```sql
-- Segmentation réseau pour optimisation
[galera]
wsrep_provider_options = "gmcast.segment=0"  # DC1 (Paris)
wsrep_provider_options = "gmcast.segment=1"  # DC2 (Londres)
wsrep_provider_options = "gmcast.segment=2"  # DC3 (Dublin)

-- Tous les nœuds dans la même grappe logique
wsrep_cluster_address = "gcomm://paris1,paris2,london1,london2,dublin-garbd"

-- Optimisation WAN
wsrep_provider_options = "
    evs.keepalive_period = PT3S;
    evs.suspect_timeout = PT30S;
    evs.inactive_timeout = PT60S;
    evs.install_timeout = PT60S
"
```

#### **Arbitre (Garagekeeper Daemon)**

```bash
# Installation garbd
apt-get install galera-arbitrator-4

# Configuration /etc/default/garbd
GALERA_GROUP="production_cluster"
GALERA_NODES="paris1:4567,paris2:4567,london1:4567,london2:4567"
LOG_FILE="/var/log/garbd.log"

# Démarrage
systemctl enable garbd
systemctl start garbd
```

💡 **Rôle du Garbd** : Participe au quorum sans stocker de données, idéal pour un 3ᵉ DC.

#### **Trade-offs Multi-DC**

**Avantages** :
- ✅ Protection contre défaillance datacenter
- ✅ Disaster Recovery natif
- ✅ Compliance réglementaire (données géo-distribuées)
- ✅ Latence locale pour utilisateurs géo-distribués

**Défis** :
- ❌ **Latence WAN** : Impact sur performance writes (50-150ms)
- ❌ **Bande passante** : Coûts réseau inter-DC
- ❌ **Split-brain complexe** : Nécessite quorum strict
- ❌ **Coût infrastructure** : Très élevé

#### **Stratégie de Writes Locaux**

```sql
-- MaxScale routing intelligent par géo-localisation
[router]
type = readwritesplit
master_accept_reads = false

# Préférence locale pour writes
router_options = master_reconnect=true,delayed_retry=true

[server-paris1]
type = server
address = 10.1.1.10
port = 3306
priority = 1  # Haute priorité si client en Europe

[server-london1]
type = server
address = 10.2.1.10
port = 3306
priority = 2  # Priorité moyenne
```

#### **Métriques Typiques**

- **RTO** : 10-60 secondes (selon détection)
- **RPO** : 0 (synchrone) ou configuré par segment
- **Disponibilité théorique** : 99.999% (Five Nines)
- **Coût** : Très élevé

---

## 3. Métriques et Calculs de Disponibilité

### 3.1 Calcul de Disponibilité Composite

Pour une architecture avec plusieurs composants :

```
Disponibilité_Totale = ∏ Disponibilité_i
                       i=1..n
```

**Exemple : Architecture 3-tier**
- Load Balancer : 99.99%
- MaxScale : 99.95%
- Galera Cluster (3 nœuds) : 99.999%

```
Disponibilité = 0.9999 × 0.9995 × 0.99999
              = 0.9994 (99.94%)
```

### 3.2 Amélioration par Redondance

Avec **n composants redondants** en parallèle :

```
Disponibilité = 1 - (1 - Disponibilité_individuelle)^n
```

**Exemple : 2 MaxScale en HA**
- MaxScale individuel : 99.9%
- 2 MaxScale en HA : 1 - (1 - 0.999)² = 1 - 0.000001 = 99.9999%

### 3.3 Calcul du RTO Réel

```
RTO_réel = T_détection + T_validation + T_bascule + T_vérification
```

**Exemple concret** :
- Détection panne (heartbeat) : 10 secondes
- Validation (éviter false positive) : 5 secondes
- Bascule (failover script) : 15 secondes
- Vérification santé : 5 secondes
- **RTO total** : 35 secondes

⚠️ **Attention** : Toujours ajouter 20-30% de marge pour les imprévus.

### 3.4 Formule de Coût de l'Indisponibilité

```
Coût = (Revenue_horaire / 3600) × Downtime_secondes × Facteur_impact
```

**Exemple e-commerce** :
- Revenue : 1M€/jour = 41 667€/heure
- Downtime : 300 secondes (5 minutes)
- Impact : 1.5 (perte client + image)

```
Coût = (41 667 / 3600) × 300 × 1.5 = 5 208€
```

💡 **ROI de la HA** : Si downtime réduit de 50h à 5h/an, économie = 2.08M€/an

---

## 4. Framework de Décision Architecturale

### 4.1 Arbre de Décision

```
┌─────────────────────────────────────┐
│ Budget disponible pour HA ?         │
└──────────┬──────────────────────────┘
           │
    ┌──────┴────────┐
    │               │
  Limité        Conséquent
    │               │
    ▼               ▼
┌─────────┐    ┌──────────────────┐
│ Active- │    │ RPO acceptable ? │
│ Passive │    └────────┬─────────┘
└─────────┘         ┌───┴────┐
                    │        │
                 RPO=0    RPO>0
                    │        │
                    ▼        ▼
              ┌─────────┐ ┌──────────┐
              │ Galera  │ │ Async    │
              │ Cluster │ │ Replica  │
              └─────────┘ └──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ Multi-DC requis ?   │
         └──────┬──────────────┘
            ┌───┴────┐
            │        │
           Oui      Non
            │        │
            ▼        ▼
      ┌──────────┐ ┌──────────┐
      │ Galera   │ │ Galera   │
      │ Multi-DC │ │ Single   │
      └──────────┘ └──────────┘
```

### 4.2 Matrice de Décision Détaillée

| Critère | Active-Passive | Galera Single-DC | Galera Multi-DC |
|---------|----------------|------------------|-----------------|
| **Coût initial** | € | €€ | €€€€ |
| **Coût opérationnel** | € | €€ | €€€€ |
| **RTO** | 1-5 min | 10-30 sec | 10-60 sec |
| **RPO** | Secondes | 0 | 0 |
| **Complexité** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalabilité reads** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalabilité writes** | ⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Latence writes** | Faible | Faible | Moyenne-Haute |
| **Protection DC** | ❌ | ❌ | ✅ |
| **Risque split-brain** | ❌ | ⚠️ Faible | ⚠️ Moyen |
| **Maintenance** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |

### 4.3 Scoring Model

Attribuez des points selon vos priorités :

```python
# Exemple de scoring
weights = {
    'rto': 10,          # Criticité maximale
    'rpo': 10,          # Criticité maximale
    'cost': 3,          # Moins critique
    'complexity': 5,    # Important
    'scalability': 7    # Très important
}

architectures = {
    'active_passive': {
        'rto': 5,       # 1-5 min
        'rpo': 7,       # Quelques secondes
        'cost': 9,      # Faible coût
        'complexity': 9,
        'scalability': 5
    },
    'galera_single': {
        'rto': 9,       # 10-30 sec
        'rpo': 10,      # Zéro
        'cost': 6,
        'complexity': 6,
        'scalability': 9
    },
    'galera_multi_dc': {
        'rto': 8,
        'rpo': 10,
        'cost': 3,
        'complexity': 3,
        'scalability': 10
    }
}

# Calcul des scores
for arch, scores in architectures.items():
    total = sum(scores[k] * weights[k] for k in scores)
    print(f"{arch}: {total}")
```

**Résultat exemple** :
- active_passive: **245**
- galera_single: **305** ← Meilleur score
- galera_multi_dc: **275**

### 4.4 Questions Clés à Poser

#### **Business**
- [ ] Quel est le coût réel d'une heure d'indisponibilité ?
- [ ] Quelles sont les obligations contractuelles (SLA) ?
- [ ] Y a-t-il des exigences réglementaires (RGPD, SOC2, etc.) ?
- [ ] Quelle est la croissance prévue (3 ans) ?

#### **Technique**
- [ ] Quelle est la latence réseau entre datacenters ?
- [ ] Quel est le ratio read/write de la charge ?
- [ ] Quels sont les pics de charge prévisibles ?
- [ ] Quelle expertise interne disponible ?

#### **Opérationnel**
- [ ] Existe-t-il une astreinte 24/7 ?
- [ ] Quelle est la maturité des processus (runbooks, DR drills) ?
- [ ] Quelle est la tolérance à la complexité ?
- [ ] Budget annuel infrastructure ?

---

## 5. Identification et Mitigation des SPOF

### 5.1 Points de Défaillance Uniques (SPOF)

**SPOF classiques dans une architecture MariaDB** :

```
Application Layer:
  ├─ SPOF: Load Balancer unique
  └─ Mitigation: Keepalived + Virtual IP

Proxy Layer:
  ├─ SPOF: MaxScale unique
  └─ Mitigation: Multiple MaxScale + HAProxy

Database Layer:
  ├─ SPOF: Single Master
  └─ Mitigation: Galera Cluster

Network Layer:
  ├─ SPOF: Switch unique
  └─ Mitigation: Redondance réseau (LACP)

Datacenter:
  ├─ SPOF: Single DC
  └─ Mitigation: Multi-DC deployment

Storage:
  ├─ SPOF: Single disk
  └─ Mitigation: RAID, réplication storage
```

### 5.2 Checklist SPOF

```bash
# Script d'analyse SPOF
#!/bin/bash

echo "=== SPOF Analysis ==="

# 1. Nombre de MaxScale
maxscale_count=$(systemctl list-units | grep -c maxscale)
[ $maxscale_count -lt 2 ] && echo "⚠️ SPOF: Only $maxscale_count MaxScale"

# 2. Nombre de nœuds Galera
galera_nodes=$(mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'" -sN | awk '{print $2}')
[ $galera_nodes -lt 3 ] && echo "⚠️ SPOF: Only $galera_nodes Galera nodes"

# 3. Quorum
quorum=$((galera_nodes / 2 + 1))
echo "ℹ️ Quorum required: $quorum / $galera_nodes"

# 4. Virtual IP configured?
ip addr show | grep -q "inet.*secondary" || echo "⚠️ SPOF: No Virtual IP"

# 5. Backup retention
backup_count=$(find /backup -name "*.sql.gz" -mtime -7 | wc -l)
[ $backup_count -eq 0 ] && echo "⚠️ SPOF: No recent backups"

echo "=== Analysis Complete ==="
```

---

## 6. Stratégies de Tests

### 6.1 Chaos Engineering pour HA

**Principe** : Introduire délibérément des pannes pour valider la résilience.

```bash
#!/bin/bash
# chaos_test.sh - Tests de résilience

function test_node_failure() {
    echo "Test: Node failure simulation"
    
    # Arrêt brutal d'un nœud
    ssh node2 "systemctl stop mariadb"
    
    # Vérification failover
    sleep 10
    
    # Test connexion application
    mysql -h vip.example.com -e "SELECT 1" &>/dev/null
    [ $? -eq 0 ] && echo "✅ Failover successful" || echo "❌ Failover failed"
    
    # Restauration
    ssh node2 "systemctl start mariadb"
}

function test_network_partition() {
    echo "Test: Network partition (split-brain)"
    
    # Isolation réseau node1
    ssh node1 "iptables -A INPUT -s 10.0.1.0/24 -j DROP"
    
    # Attendre détection
    sleep 30
    
    # Vérifier quorum
    cluster_size=$(mysql -h node2 -e "SHOW STATUS LIKE 'wsrep_cluster_size'" -sN | awk '{print $2}')
    echo "Cluster size after partition: $cluster_size"
    
    # Restauration
    ssh node1 "iptables -F"
}

function test_maxscale_failure() {
    echo "Test: MaxScale proxy failure"
    
    # Arrêt MaxScale primary
    systemctl stop maxscale
    
    # Vérification bascule VIP
    sleep 5
    ping -c 1 vip.maxscale.example.com &>/dev/null
    
    [ $? -eq 0 ] && echo "✅ VIP migrated" || echo "❌ VIP not migrated"
    
    # Restauration
    systemctl start maxscale
}

# Exécution séquentielle
test_node_failure
sleep 60
test_network_partition
sleep 60
test_maxscale_failure
```

### 6.2 Scénarios de Tests Recommandés

| Scénario | Objectif | Fréquence |
|----------|----------|-----------|
| **Node crash** | Valider détection et failover | Mensuel |
| **Split-brain** | Valider quorum et fencing | Trimestriel |
| **Full DC outage** | Valider DR multi-DC | Semestriel |
| **Backup restore** | Valider procédure restauration | Mensuel |
| **Load spike** | Valider scalabilité | Continu (prod) |
| **Upgrade simulation** | Valider rolling upgrade | Avant chaque release |

---

## ✅ Points Clés à Retenir

- **Le théorème CAP** impose de choisir entre cohérence (CP) et disponibilité (AP) en cas de partition réseau
- **RTO et RPO** sont les métriques business fondamentales pour dimensionner une architecture HA
- **Active-Passive** convient pour des budgets limités avec RTO de quelques minutes acceptable
- **Galera Cluster** offre un RPO de zéro et un RTO < 30 secondes, au prix d'une complexité accrue
- **Multi-DC** protège contre la défaillance datacenter mais introduit des défis de latence WAN
- **Identifier et mitiger les SPOF** est crucial pour atteindre une disponibilité > 99.99%
- **Tester régulièrement** les scénarios de défaillance est non négociable (Chaos Engineering)

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 MariaDB High Availability Guide](https://mariadb.com/kb/en/high-availability-performance-tuning-mariadb-replication/)
- [📖 CAP Theorem Explained](https://en.wikipedia.org/wiki/CAP_theorem)
- [📖 Google SRE Book - Availability](https://sre.google/sre-book/availability/)

### Articles de Référence
- **"Designing Data-Intensive Applications"** - Martin Kleppmann (Chapitre 5-9)
- **"The Calculus of Service Availability"** - Google SRE
- **"Chaos Engineering: Building Confidence in System Behavior"** - Netflix

### Outils d'Analyse
- [Availability Calculator](https://availability.sre.xyz/)
- [RTO/RPO Calculator](https://www.druva.com/resources/rto-rpo-calculator/)
- [Chaos Monkey](https://netflix.github.io/chaosmonkey/) - Netflix

---

## ➡️ Section Suivante

**[14.2 MariaDB Galera Cluster](/14-haute-disponibilite/02-galera-cluster.md)**

Maintenant que les fondations théoriques sont posées, nous plongeons dans l'implémentation concrète de Galera Cluster : architecture synchrone multi-master, certification-based replication, configuration de production, et opérations (SST, IST).

---

**Maîtriser ces concepts architecturaux est essentiel avant toute implémentation HA en production.**

⏭️ [MariaDB Galera Cluster](/14-haute-disponibilite/02-galera-cluster.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 6 : Réplication et Haute Disponibilité (DBA/DevOps)

> **Niveau** : Expert — DBA Senior, DevOps/SRE, Architectes Infrastructure  
> **Durée estimée** : 3-4 jours  
> **Prérequis** : Maîtrise administration MariaDB, expérience systèmes distribués, compréhension réseaux et tolérance aux pannes, notions de consensus distribué

---

## 🎯 L'infrastructure qui ne dort jamais

Dans le monde moderne des services 24/7, **l'indisponibilité n'est plus acceptable**. Un site e-commerce qui tombe perd $5,600 par minute en moyenne. Une application bancaire inaccessible génère un préjudice réputationnel majeur. Un système médical hors ligne peut mettre des vies en danger. La haute disponibilité n'est pas un luxe — c'est une **exigence business, contractuelle, et parfois légale**.

La différence entre 99.9% et 99.99% de disponibilité semble minime sur le papier, mais elle représente un écart de **8h43 contre 52 minutes d'indisponibilité annuelle**. Pour atteindre 99.999% ("five nines"), vous ne disposez que de **5 minutes 15 secondes par an** pour toutes les pannes combinées. Cette exigence transforme fondamentalement l'architecture : les composants simples (single point of failure) deviennent inadmissibles, et chaque décision technique doit intégrer la résilience.

La réplication et la haute disponibilité dans MariaDB répondent à trois objectifs stratégiques :

1. **Tolérance aux pannes** : Continuer à fonctionner malgré la défaillance de serveurs, disques, datacenters
2. **Scalabilité en lecture** : Distribuer la charge sur plusieurs nœuds pour performances accrues
3. **Disaster Recovery** : Récupération rapide après incident majeur (incendie, catastrophe naturelle, cyberattaque)

Cette partie vous enseigne comment concevoir, implémenter, et opérer des **architectures MariaDB résilientes** capables de maintenir le service même face à des défaillances multiples. Vous maîtriserez la réplication asynchrone et semi-synchrone pour scaling et redondance, Galera Cluster pour synchronisation multi-master, MaxScale pour load balancing et failover automatique, et les stratégies de récupération après incident.

Ces compétences sont **essentielles pour tout DBA ou DevOps responsable d'infrastructures critiques**. Vous apprendrez non seulement les technologies, mais surtout comment concevoir des systèmes qui continuent de fonctionner quand (et non "si") les composants tombent.

---

## 📚 Les deux modules de cette partie

### Module 13 : Réplication
**10 sections | Durée : ~2 jours**

Ce module couvre l'ensemble du spectre de la réplication MariaDB, des fondamentaux à l'exploitation en production :

#### 🔄 Fondamentaux de la réplication
- **Concepts** : Asynchrone vs semi-synchrone vs synchrone
- **Topologies** : Source-Replica, multi-tier, hub-and-spoke
- **Use cases** : Scaling en lecture, backup, disaster recovery, analytics

#### 🏗️ Réplication Master-Slave (Source-Replica)
- **Configuration Primary** : Binary logs, server-id, binlog_format
- **Configuration Replica** : Connexion au primary, threads SQL/IO
- **CHANGE MASTER TO** : Configuration de la réplication
- **Monitoring** : `SHOW SLAVE STATUS`, lag, threads health

#### 📍 Méthodes de réplication
- **Binlog coordinates** : Réplication basée sur positions (fichier + position)
- **GTID (Global Transaction Identifier)** : Réplication sans positions 🔄
  - Avantages : Failover simplifié, topologies flexibles
  - Configuration et migration vers GTID
  - Transactions sur plusieurs serveurs

#### 🌳 Topologies avancées
- **Multi-source replication** : Un replica reçoit de plusieurs primaries
  - Use case : Consolidation de données, fédération
  - Configuration et gestion des canaux
- **Réplication en cascade** : Primary → Intermediate → Replicas
  - Réduction de charge sur primary
  - Distribution géographique

#### 🔍 Monitoring et troubleshooting
- **Métriques critiques** : `Seconds_Behind_Master`, lag accumulation, thread status
- **Erreurs courantes** : Duplicate key, network issues, disk full
- **Outils de diagnostic** : `SHOW PROCESSLIST`, error logs, binary log inspection
- **Performance tuning** : Parallel replication, compression

#### 🚨 Failover et switchover
- **Switchover planifié** : Migration contrôlée vers nouveau primary
- **Failover d'urgence** : Promotion automatique en cas de panne
- **Réconciliation** : Resynchronisation après failover
- **Tools** : MaxScale, Orchestrator, MHA

#### ⚡ Réplication semi-synchrone
- **Garanties de durabilité** : Confirmation avant commit
- **Trade-offs** : Latence vs consistance
- **Configuration** : Plugins et timeouts
- **Use cases** : Systèmes financiers, données critiques

#### 🆕 Optimistic ALTER TABLE (11.8)
- **Problème** : `ALTER TABLE` bloque la réplication (lag majeur)
- **Solution** : Réplication continue pendant DDL
- **Impact** : Réduction lag de 90%+ sur grandes tables
- **Configuration** : Activation et précautions

💡 **Impact opérationnel** : La réplication correctement configurée permet de maintenir le service pendant maintenance, pannes matérielles, et croissance de charge. Elle est la base de toute stratégie HA.

---

### Module 14 : Haute Disponibilité
**10 sections | Durée : ~2 jours**

Ce module vous enseigne les architectures et outils pour éliminer les single points of failure :

#### 🏛️ Architectures HA : Fondamentaux
- **Principes** : Redondance, détection rapide, failover automatique
- **Topologies** : Active-Passive, Active-Active, N+1, N+M
- **Trade-offs** : Coût vs disponibilité, simplicité vs résilience
- **SLA calculations** : De l'infrastructure au service

#### 🔗 MariaDB Galera Cluster
- **Architecture synchrone multi-master** : Tous les nœuds acceptent écritures
- **Certification-based replication** : Cohérence garantie
- **Configuration et déploiement** : Bootstrap, joindre un cluster
- **State transfers** : SST (State Snapshot Transfer) vs IST (Incremental)
- **Performance** : Write scaling limitations, certification overhead

#### ⚖️ Split-brain et quorum
- **Le problème du split-brain** : Partitions réseau créent clusters indépendants
- **Mécanismes de quorum** : Majorité simple, weighted voting
- **Prevention** : Garbd (Galera Arbitrator), odd number of nodes
- **Recovery** : Réunification après partition

#### 🌐 MaxScale : Le couteau suisse de la HA
- **Architecture** : Proxy intelligent entre apps et bases 🔄
- **Load Balancing** : Distribution de charge automatique
- **Read/Write Split** : Routage intelligent selon type de requête
- **Query Routing** : Règles complexes de redirection
- **Database Firewall** : Protection contre SQL injection

#### 🆕 MaxScale 25.01 : Innovations majeures
- **Workload Capture** : Enregistrement du trafic production 🆕
  - Capture complète : requêtes, timing, transactions
  - Anonymisation pour conformité
  - Storage compressé
- **Workload Replay** : Rejeu de charge réelle 🆕
  - Tests de performance réalistes
  - Validation de migrations/upgrades
  - Capacity planning basé sur données réelles
- **Diff Router** : Comparaison de versions 🆕
  - Routage vers deux backends simultanément
  - Comparaison de résultats en temps réel
  - Validation d'upgrades MariaDB sans risque

#### 🤖 Failover automatique
- **Détection de pannes** : Health checks, timeouts, consensus
- **Promotion de replica** : Sélection du meilleur candidat
- **Reconfiguration automatique** : Réorientation du trafic
- **Notifications** : Alerting et escalation
- **Tools** : MaxScale auto-failover, Corosync/Pacemaker

#### 🌐 Virtual IP et keepalived
- **Floating VIP** : IP qui migre avec le primary actif
- **VRRP protocol** : Virtual Router Redundancy Protocol
- **Configuration** : keepalived setup et priority
- **Integration** : Avec réplication ou Galera

#### 🔄 Transaction Replay et Connection Migration (11.8) 🆕
- **Transaction Replay** : Re-exécution automatique après échec temporaire
- **Connection Migration** : Transfert transparent de sessions entre nœuds
- **Use cases** : Rolling upgrades sans downtime, maintenance transparente
- **Configuration** : Activation et limites

#### 🛡️ Stratégies de récupération après incident
- **Incident classification** : Mineur (1 node) vs Majeur (DC entier)
- **Runbooks** : Procédures documentées et testées
- **Escalation** : Qui appeler, quand, comment
- **Post-mortem** : Analyse et amélioration continue

#### 🔀 Alternatives : ProxySQL et HAProxy
- **ProxySQL** : Proxy avancé avec query caching et rewrite
- **HAProxy** : Load balancer générique pour MariaDB
- **Comparaison** : Forces et faiblesses vs MaxScale

💡 **Impact business** : Une architecture HA bien conçue peut atteindre 99.99% de disponibilité (52 min/an downtime) voire 99.999% avec multi-DC. C'est la différence entre un service fiable et un service de classe mondiale.

---

## 🆕 Innovations MariaDB 11.8 et MaxScale 25.01 pour la HA

### Réduction drastique du lag de réplication : Optimistic ALTER TABLE

#### Le problème historique

```sql
-- Scénario : Table de 500M lignes, index à ajouter
ALTER TABLE large_table ADD INDEX idx_email (email);
-- Temps : 2 heures

-- Pendant ces 2 heures :
-- ❌ Réplication complètement bloquée sur replicas
-- ❌ Lag accumulation : 2 heures de retard
-- ❌ Risque : Replicas inutilisables pour lectures
```

**Impact** : Sur une application avec 10,000 requêtes/sec, un lag de 2h signifie 72 millions de requêtes en retard.

#### La solution MariaDB 11.8 : Optimistic ALTER TABLE 🆕

```sql
-- MariaDB 11.8 : Réplication continue pendant ALTER
SET GLOBAL replicate_alter_as_create_select = ON;

ALTER TABLE large_table ADD INDEX idx_email (email);
-- Primary : 2 heures
-- Replicas : Réplication continue, lag < 10 secondes

-- Résultat :
-- ✅ Réplication parallèle au DDL
-- ✅ Lag réduit de 99.9% (2h → 10s)
-- ✅ Replicas restent utilisables
```

**Impact opérationnel** : Maintenance sans interruption de service, scaling sans downtime.

---

### Continuité de service : Transaction Replay et Connection Migration 🆕

#### Transaction Replay : Résilience automatique

```python
# Application : Transaction bancaire
try:
    connection.begin()
    connection.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    connection.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
    connection.commit()
except TransientError:
    # MariaDB 11.8 : Transaction Replay automatique
    # La transaction est re-exécutée sur un autre nœud
    # Transparent pour l'application
    pass
```

**Bénéfices** :
- ✅ Échecs temporaires résolus automatiquement
- ✅ Moins de code de retry dans applications
- ✅ Meilleure expérience utilisateur

#### Connection Migration : Maintenance transparente

```sql
-- Scénario : Maintenance d'un nœud Galera
-- Avant 11.8 : 1000+ connexions perdues, erreurs applicatives

-- Avec Connection Migration :
-- 1. MaxScale détecte node en maintenance
-- 2. Migrations progressives des connexions vers autres nœuds
-- 3. Transactions en cours relocalisées automatiquement
-- 4. Zéro erreur applicative

-- Résultat : Rolling upgrades sans downtime
```

**Impact** : Maintenance et upgrades sans fenêtre de downtime planifiée.

---

### MaxScale 25.01 : Testing et validation nouvelle génération

#### Workload Capture : Production comme jeu de données de test 🆕

```bash
# Capturer trafic production pendant 1 heure
maxctrl call command workload capture start \
  --duration=3600 \
  --output=/data/capture/prod-2025-12-15.wcap

# Analyse du workload capturé
maxctrl call command workload analyze /data/capture/prod-2025-12-15.wcap
# Output :
# - Queries per second: 15,432
# - Read/Write ratio: 75/25
# - Peak connections: 842
# - Top 10 queries
# - Transaction patterns
```

**Use cases** :
- 📊 Capacity planning basé sur trafic réel
- 🔬 Optimisation de requêtes avec données réelles
- 📈 Benchmark avant scaling infrastructure

#### Workload Replay : Validation à l'échelle production 🆕

```bash
# Rejouer workload capturé sur nouvel environnement
maxctrl call command workload replay start \
  --input=/data/capture/prod-2025-12-15.wcap \
  --target=mariadb-new-11.8 \
  --speed=1.0  # Temps réel

# Monitoring pendant replay
maxctrl call command workload replay status
# Output :
# - Progress: 35%
# - Queries executed: 5,432,890
# - Errors: 12
# - Average latency: 4.2ms (vs 4.5ms prod)
# - 95th percentile: 15ms (vs 18ms prod)
```

**Avantages révolutionnaires** :
- ✅ Tests de charge avec trafic réel, pas synthétique
- ✅ Validation d'upgrades MariaDB sans risque
- ✅ Identification de régressions de performance
- ✅ Capacity planning précis

#### Diff Router : Comparaison de versions en temps réel 🆕

```yaml
# Configuration MaxScale Diff Router
[Diff-Service]
type=service
router=diffsvc
servers=mariadb-11.4-prod, mariadb-11.8-test
user=maxscale
password=***

# Routage vers les deux versions simultanément
# Comparaison automatique des résultats
```

**Workflow de validation** :
```bash
# 1. Routage vers anciennes et nouvelles versions
# 2. MaxScale compare résultats en temps réel
# 3. Logging des différences détectées
maxctrl call command diff-router analyze

# Output :
# - Total queries: 1,250,000
# - Identical results: 1,249,987 (99.999%)
# - Differences detected: 13
#   → Query ID 4521: Row order difference (acceptable)
#   → Query ID 8934: Aggregation precision difference (investigate)
```

**Impact** : Migrations et upgrades MariaDB avec validation exhaustive et zéro surprise.

---

## ✅ Compétences acquises

À la fin de cette sixième partie, vous serez capable de :

### Conception d'architectures résilientes
- ✅ **Concevoir** des topologies de réplication adaptées aux besoins (read scaling, HA, DR)
- ✅ **Choisir** entre réplication asynchrone, semi-synchrone, et synchrone selon contraintes
- ✅ **Dimensionner** des clusters Galera pour charge de travail donnée
- ✅ **Implémenter** des architectures multi-datacenter pour disaster recovery
- ✅ **Calculer** la disponibilité théorique d'une architecture (SLA)
- ✅ **Identifier** et éliminer les single points of failure

### Opération en production
- ✅ **Configurer** la réplication MariaDB avec GTID
- ✅ **Déployer** et maintenir un cluster Galera en production
- ✅ **Implémenter** MaxScale pour load balancing et failover
- ✅ **Monitorer** la santé de la réplication (lag, threads, errors)
- ✅ **Diagnostiquer** et résoudre les problèmes de réplication
- ✅ **Effectuer** des failovers planifiés sans downtime

### Zéro downtime operations
- ✅ **Planifier** et exécuter des maintenances sans interruption
- ✅ **Upgrader** MariaDB sans arrêt de service (rolling upgrades)
- ✅ **Utiliser** Workload Capture/Replay pour tests réalistes
- ✅ **Valider** des migrations avec Diff Router
- ✅ **Appliquer** Transaction Replay pour résilience applicative

### Disaster recovery
- ✅ **Concevoir** des stratégies de récupération après incident
- ✅ **Définir** RTO et RPO adaptés aux besoins business
- ✅ **Documenter** des runbooks de failover testés
- ✅ **Effectuer** des drill exercises réguliers
- ✅ **Récupérer** d'un incident de split-brain
- ✅ **Coordonner** une réponse à incident majeur

### Expertise avancée
- ✅ **Optimiser** la performance de la réplication
- ✅ **Prévenir** les problèmes de quorum et split-brain
- ✅ **Configurer** Virtual IP avec keepalived
- ✅ **Intégrer** monitoring et alerting pour HA
- ✅ **Automatiser** le failover avec détection intelligente

---

## 🎓 Parcours recommandés

Cette partie est **absolument critique** pour DBA et DevOps gérant des systèmes en production.

| Parcours | Importance | Justification |
|----------|------------|---------------|
| 🔐 **Administrateur/DBA** | ⭐⭐⭐ ABSOLUMENT CRITIQUE | La HA est une responsabilité fondamentale du DBA. Sans maîtrise de la réplication et Galera, impossible d'opérer en production critique. |
| ⚙️ **DevOps/SRE** | ⭐⭐⭐ ABSOLUMENT CRITIQUE | DevOps responsables du uptime doivent maîtriser failover, monitoring, et récupération après incident. Non négociable pour atteindre SLA. |
| 🏗️ **Architecte Infrastructure** | ⭐⭐⭐ ESSENTIEL | Conception d'architectures résilientes nécessite compréhension profonde des capacités et limitations de chaque solution HA. |
| 🔧 **Développeur Senior** | ⭐⭐ IMPORTANT | Comprendre la HA aide à écrire du code résilient (retry logic, timeouts, idempotence). Particulièrement pour développeurs d'applications critiques. |

### Pourquoi cette partie est non négociable ?

#### Pour les DBA
Responsabilité contractuelle d'assurer la disponibilité :
- **SLA 99.9%** = Maximum 8h43 downtime/an
- **SLA 99.99%** = Maximum 52 min downtime/an  
- **SLA 99.999%** = Maximum 5 min 15s downtime/an

**Sans maîtrise de la HA, ces SLA sont impossibles à atteindre.**

#### Pour les DevOps/SRE
La réplication et HA sont au cœur du métier SRE :
- **Toil reduction** : Automatisation du failover
- **Error budgets** : Calcul basé sur disponibilité réelle
- **Incident response** : Procédures de récupération testées
- **Capacity planning** : Dimensionnement pour redondance

#### Pour les Architectes
Décisions structurelles impactant des années :
- **Multi-DC** : Latence vs consistance
- **Active-Active** : Complexité vs performance
- **Costs** : 2x infrastructure pour N+1, 3x pour N+M
- **Trade-offs** : CAP theorem dans le monde réel

---

## 🚨 Scénarios de défaillance et stratégies de récupération

### Taxonomie des pannes

#### Type 1 : Défaillance d'un nœud (Fréquence : Mensuelle)

**Scénario** : Un serveur replica tombe (problème disque, OOM, crash)

**Impact** :
- ⚠️ Réplication interrompue sur ce nœud
- ⚠️ Lectures distribuées vers nœuds restants
- ✅ Écritures non affectées (sur primary)

**Récupération** :
```bash
# 1. Identifier le problème
mariadb -e "SHOW SLAVE STATUS\G" | grep -E "(Slave_IO_Running|Slave_SQL_Running|Last_Error)"

# 2. Si erreur réparable
mariadb -e "STOP SLAVE; SET GLOBAL sql_slave_skip_counter = 1; START SLAVE;"

# 3. Si corruption, resync complet
mariabackup --backup --target-dir=/backup/resync
# Restaurer sur replica
mariabackup --copy-back --target-dir=/backup/resync
mariadb -e "CHANGE MASTER TO MASTER_LOG_FILE='binlog.xxx', MASTER_LOG_POS=yyy; START SLAVE;"
```

**RTO** : 10-30 minutes  
**RPO** : 0 (pas de perte de données)

---

#### Type 2 : Défaillance du primary (Fréquence : Trimestrielle)

**Scénario** : Le serveur primary tombe (hardware failure, data center issue)

**Impact** :
- 🔴 Écritures bloquées → Downtime applicatif
- ⚠️ Lectures fonctionnent sur replicas
- 🚨 Urgence : Promouvoir un replica en primary

**Récupération automatique avec MaxScale** :
```bash
# MaxScale détecte panne primary (health check échoue)
# → Auto-promotion du replica le plus à jour
# → Reconfiguration du routage
# → Notifications

# Timeline :
# T+0s : Primary tombe
# T+5s : MaxScale détecte (health check interval)
# T+10s : Promotion du meilleur replica
# T+15s : Routage reconfiguré
# T+20s : Applications fonctionnelles

# RTO : 20-30 secondes
```

**Récupération manuelle (sans MaxScale)** :
```bash
# 1. Identifier le replica le plus à jour
mariadb -h replica1 -e "SHOW SLAVE STATUS\G" | grep Exec_Master_Log_Pos
mariadb -h replica2 -e "SHOW SLAVE STATUS\G" | grep Exec_Master_Log_Pos

# 2. Promouvoir le meilleur replica
mariadb -h replica1 -e "STOP SLAVE; RESET SLAVE ALL;"

# 3. Repointer autres replicas
mariadb -h replica2 -e "STOP SLAVE; CHANGE MASTER TO MASTER_HOST='replica1'; START SLAVE;"

# 4. Reconfigurer applications vers nouveau primary
# (Idéalement via VIP pour transparence)

# RTO : 5-15 minutes (manuel)
```

**RPO** : 
- Réplication async : 0-30 secondes (lag typique)
- Réplication semi-sync : 0 (garanti)

---

#### Type 3 : Split-brain Galera (Fréquence : Annuelle)

**Scénario** : Partition réseau divise un cluster Galera 4 nœuds en deux groupes (2+2)

**Impact** :
- 🔴 Deux groupes indépendants pensent être le cluster primaire
- 🔴 Risque de divergence de données (write conflicts)
- 🔴 Requiert intervention manuelle

**Prévention** :
```ini
# galera.cnf : Toujours nombre impair de nœuds
wsrep_cluster_size = 3  # ou 5, 7, etc.

# Avec Garbd (Galera Arbitrator) si contrainte budgétaire
# Node 1, 2 : Serveurs complets
# Node 3 : Garbd (léger, vote uniquement)
garbd --group=my_cluster --address=gcomm://node1,node2
```

**Récupération** :
```bash
# 1. Identifier la partition avec le quorum
# Partition avec majorité continue (2 nœuds sur 3)
# Partition minoritaire se met en "non-primary" state

# 2. Sur partition minoritaire, forcer shutdown
systemctl stop mariadb

# 3. Après résolution réseau, rejoindre cluster
# Le nœud va faire un IST (Incremental State Transfer)
systemctl start mariadb

# Vérification
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
# Output : Primary (OK)
```

**RTO** : 10-30 minutes (selon durée partition réseau)  
**RPO** : 0 (pas de perte si résolution correcte)

---

#### Type 4 : Disaster complet d'un datacenter (Fréquence : Tous les 5-10 ans)

**Scénario** : Incendie, inondation, cyberattaque, coupure électrique prolongée → DC entier hors ligne

**Impact** :
- 🔴🔴🔴 Service entièrement down si pas de multi-DC
- 🔴🔴🔴 Perte de données si RPO > durée incident

**Architecture préventive : Multi-DC avec Galera** :
```plaintext
Datacenter A (Paris)       Datacenter B (Frankfurt)
├─ Galera Node 1          ├─ Galera Node 3
├─ Galera Node 2          ├─ Galera Node 4
└─ Garbd                  └─ Garbd

Latence inter-DC : 15-20ms (acceptable pour Galera)
Write Performance : Impacté par latence, mais disponibilité garantie
```

**Récupération** :
```bash
# Scenario : DC A en feu, DC B continue seul

# 1. DC B maintient le service automatiquement (quorum 2/4 + Garbd)
# 2. Applications re-routées vers DC B (DNS, load balancer)
# 3. Opération continue avec légère dégradation performance

# Après reconstruction DC A :
# 1. Provisionner nouveaux serveurs
# 2. Joindre cluster existant
# 3. SST automatique depuis DC B
# 4. Service complètement restauré

# RTO : 0 (transparent pour utilisateurs)
# RPO : 0 (synchronous replication)
```

**Coût** : 2x infrastructure minimum (Active-Active multi-DC)

---

#### Type 5 : Corruption de données silencieuse (Fréquence : Rare mais critique)

**Scénario** : Bug applicatif, ransomware, ou erreur humaine corrompt des données

**Impact** :
- 🔴 Données incorrectes répliquées sur tous les nœuds
- 🔴 Découverte souvent tardive (heures/jours après)
- 🔴 Nécessite restauration depuis backup

**Récupération PITR (Point-in-Time Recovery)** :
```bash
# Scenario : Découverte d'une suppression accidentelle à 14:35
# Last good backup : 02:00 ce matin

# 1. Identifier position binlog du backup
cat /backup/xtrabackup_binlog_info
# Output : binlog.000042  12847392

# 2. Restaurer backup sur serveur temporaire
mariabackup --copy-back --target-dir=/backup/full-02h00

# 3. Rejouer binlogs jusqu'à 14:34 (avant corruption)
mysqlbinlog --start-position=12847392 \
            --stop-datetime="2025-12-15 14:34:00" \
            /var/lib/mysql/binlog.0000* \
  | mariadb

# 4. Valider les données
mariadb -e "SELECT COUNT(*) FROM critical_table WHERE deleted_at IS NULL;"

# 5. Migrer données corrigées vers production
# (Via dump/restore ou réplication temporaire)

# RTO : 2-6 heures (selon taille base)
# RPO : <1 minute (granularité binlog)
```

---

### Matrice de récupération

| Type d'incident | Fréquence | RTO | RPO | Automatisable | Coût prévention |
|-----------------|-----------|-----|-----|---------------|-----------------|
| Nœud replica down | Mensuel | 10-30 min | 0 | ✅ Oui (monitoring) | Faible |
| Primary down | Trimestriel | 20s-15min | 0-30s | ✅ Oui (MaxScale) | Moyen |
| Split-brain Galera | Annuel | 10-30 min | 0 | ⚠️ Partiellement | Faible (Garbd) |
| DC disaster | 5-10 ans | 0-4h | 0-1h | ✅ Oui (Multi-DC) | Élevé (2x infra) |
| Data corruption | Rare | 2-6h | 1 min | ❌ Non (PITR manuel) | Moyen (backups) |

---

## 📊 SLA et objectifs RTO/RPO

### Comprendre les SLA (Service Level Agreements)

Un SLA définit contractuellement le niveau de service garanti. Pour une base de données, le critère principal est la **disponibilité**.

#### Calcul de disponibilité

```
Disponibilité (%) = (Temps total - Temps indisponible) / Temps total × 100
```

#### Tableau de référence des SLA

| SLA | Downtime annuel | Downtime mensuel | Downtime hebdomadaire | Classe de service |
|-----|-----------------|------------------|----------------------|-------------------|
| 90% | 36.5 jours | 72 heures | 16.8 heures | Inacceptable |
| 95% | 18.25 jours | 36 heures | 8.4 heures | Basic |
| 99% | 3.65 jours | 7.2 heures | 1.68 heures | Standard |
| 99.9% ("three nines") | **8h 43min** | 43.2 minutes | 10.1 minutes | High Availability |
| 99.95% | 4h 22min | 21.6 minutes | 5 minutes | Very High Availability |
| 99.99% ("four nines") | **52 minutes** | 4.3 minutes | 1 minute | Mission Critical |
| 99.999% ("five nines") | **5min 15s** | 26 secondes | 6 secondes | Ultra Critical |

#### Impact business selon SLA

```
E-commerce $10M/an de CA (24/7) :
- 99%    → 3.65 jours down  → $100,000 de perte potentielle
- 99.9%  → 8h43min down     → $3,650 de perte potentielle  
- 99.99% → 52 minutes down  → $360 de perte potentielle

Banking application :
- 99.99% minimum requis par régulation
- Pénalités contractuelles si dépassement
```

---

### RTO (Recovery Time Objective)

**Définition** : Temps maximum acceptable entre une panne et la restauration du service.

#### Exemples par cas d'usage

| Cas d'usage | RTO typique | Architecture requise |
|-------------|-------------|----------------------|
| Blog personnel | 24 heures | Single server + backup hebdomadaire |
| Site e-commerce PME | 4 heures | Réplication async + backup quotidien |
| Application SaaS B2B | 1 heure | Réplication semi-sync + MaxScale |
| Banking core system | 15 minutes | Galera cluster + Multi-DC |
| Trading platform | 30 secondes | Galera multi-DC + auto-failover |
| Payment processing | 0 (continuous) | Multi-DC active-active + edge caching |

#### Facteurs influençant RTO

1. **Détection de panne** : Monitoring fréquence (1s, 5s, 30s)
2. **Failover automatique** : Automatisé vs manuel
3. **Taille des données** : Impact sur restauration
4. **Complexité architecture** : Plus de composants = plus de points de défaillance
5. **Compétences équipe** : Runbooks documentés et testés

---

### RPO (Recovery Point Objective)

**Définition** : Quantité maximale acceptable de données perdues, mesurée en temps.

#### Exemples par cas d'usage

| Cas d'usage | RPO typique | Stratégie backup/réplication |
|-------------|-------------|------------------------------|
| Blog personnel | 24 heures | Backup hebdomadaire |
| Analytics database | 6 heures | Backup quotidien + binlogs |
| CMS/CRM | 1 heure | Backup quotidien + binlog streaming |
| E-commerce | 5 minutes | Réplication async continue |
| Financial transactions | 0 (zero data loss) | Réplication semi-sync ou Galera |
| Banking core | 0 (zero data loss) | Galera multi-DC synchronous |

#### Relation RTO/RPO et architecture

```plaintext
Architecture Simple (Single Server + Backup)
├─ RTO : 4-24 heures (restauration manuelle)
└─ RPO : 1-24 heures (fréquence backup)

Architecture Réplication Async
├─ RTO : 5-15 minutes (failover manuel)
└─ RPO : 0-60 secondes (lag réplication)

Architecture Réplication Semi-Sync + MaxScale
├─ RTO : 20-60 secondes (failover automatique)
└─ RPO : 0 secondes (garanti)

Architecture Galera Multi-DC
├─ RTO : 0-5 secondes (transparente)
└─ RPO : 0 secondes (synchronous)
```

---

### Coût de la haute disponibilité

**Principe fondamental** : Chaque "nine" supplémentaire coûte ~10x plus cher.

#### Matrice coût/disponibilité

| SLA cible | Architecture minimale | Coût relatif | Complexité |
|-----------|----------------------|--------------|------------|
| 99% | Single server + backup | 1x | Faible |
| 99.9% | Primary + 2 replicas + backup | 3x | Moyenne |
| 99.99% | Galera 3 nodes + MaxScale + monitoring | 5-7x | Élevée |
| 99.999% | Galera multi-DC + edge caching + automation | 10-15x | Très élevée |

#### Analyse coût/bénéfice

```
Exemple : E-commerce $10M CA/an

Option A : 99.9% SLA
- Infrastructure : $50k/an
- Downtime potentiel : 8h43min → $3,650 perte
- Total Cost of Ownership : $53,650

Option B : 99.99% SLA
- Infrastructure : $200k/an
- Downtime potentiel : 52min → $360 perte
- Total Cost of Ownership : $200,360

Différentiel : $146,710/an
Bénéfice : -$3,290 de pertes évitées

→ Pour ce cas, 99.9% est optimal sauf si :
  - Impact réputationnel significatif
  - Pénalités contractuelles
  - Exigence réglementaire
```

**Règle pragmatique** : Viser le SLA le plus bas acceptable business, pas le plus élevé techniquement possible.

---

## 💡 Principes d'architecture résiliente

### Les 7 piliers de la haute disponibilité

#### 1️⃣ Redondance à tous les niveaux

```plaintext
Application Layer     [LB] → [App1] [App2] [App3]
                       ↓
Database Proxy Layer  [MaxScale1] [MaxScale2]
                       ↓
Database Layer        [Primary] [Replica1] [Replica2]
                       ↓
Storage Layer         [SAN] avec RAID10 + snapshots
                       ↓
Network Layer         [Switch1] [Switch2] avec bonding
                       ↓
Power Layer           [UPS] + [Generator]
```

**Principe** : Aucun single point of failure à aucun niveau.

#### 2️⃣ Détection rapide et précise

```bash
# Monitoring multi-niveaux
- Health check TCP : 1 seconde
- Health check SQL : 5 secondes (SELECT 1)
- Health check applicatif : 10 secondes (query complexe)
- Monitoring métrique : 30 secondes (lag, connections, etc.)
```

**Principe** : Détecter avant que l'utilisateur ne soit impacté.

#### 3️⃣ Failover automatique testé

```yaml
# Automatisation complète
Detection → Decision → Action → Validation
   ↓           ↓         ↓          ↓
  5s         2s       10s       3s
  
Total : 20 secondes RTO
```

**Principe** : L'humain est trop lent pour maintenir des SLA élevés.

#### 4️⃣ Isolation des défaillances

```plaintext
Failure Domain 1     Failure Domain 2     Failure Domain 3
├─ Rack A           ├─ Rack B           ├─ Rack C
├─ Switch A         ├─ Switch B         ├─ Switch C
├─ Node 1           ├─ Node 2           ├─ Node 3
└─ Subnet 10.0.1.x  └─ Subnet 10.0.2.x  └─ Subnet 10.0.3.x
```

**Principe** : Une défaillance ne doit affecter qu'un domaine.

#### 5️⃣ Graceful degradation

```python
# Application : Dégradation gracieuse
if primary_db.available():
    return full_featured_query()
elif replica_db.available():
    return read_only_query()  # Mode dégradé
elif cache.available():
    return cached_result()    # Mode très dégradé
else:
    return error_page()       # Dernier recours
```

**Principe** : Service dégradé > pas de service.

#### 6️⃣ Chaos Engineering

```bash
# Tests de résilience réguliers
- Kill random node monthly
- Inject network latency weekly
- Simulate disk full quarterly
- Full DR drill annually
```

**Principe** : Casser volontairement pour valider la résilience.

#### 7️⃣ Documentation et runbooks

```markdown
# Runbook : Primary Database Failure
## Symptoms
- MaxScale logs : "Server primary marked as down"
- Application errors : Connection refused

## Impact
- Write operations blocked
- Read operations functioning

## Actions
1. Verify primary really down (ssh, ping, telnet)
2. Check replica lag : SHOW SLAVE STATUS
3. If lag < 10s : Promote best replica
4. If lag > 10s : Wait for catch-up or force promotion
5. Reconfigure applications/MaxScale
6. Notify team

## Rollback
- Keep old primary offline until sync
- Rejoin as replica when fixed

## Post-mortem
- Root cause analysis
- Update automation if needed
```

**Principe** : Procédures écrites, testées, et à jour.

---

## 🎯 Prérequis pour cette partie

Cette partie est **la plus exigeante** de la formation. Assurez-vous de maîtriser :

### Techniques
- ✅ Administration MariaDB avancée (Parties 3-5)
- ✅ Transactions, verrouillage, MVCC
- ✅ Binary logs et leur rôle
- ✅ Backup/restauration (Mariabackup, PITR)

### Systèmes distribués
- ✅ Concepts de consensus (Paxos, Raft basics)
- ✅ Théorème CAP (Consistency, Availability, Partition tolerance)
- ✅ Trade-offs latence vs cohérence
- ✅ Split-brain et quorum

### Réseaux
- ✅ TCP/IP, latence, bande passante
- ✅ Virtual IPs et routage
- ✅ Firewalls et segmentation réseau
- ✅ DNS et load balancers

### Opérationnel
- ✅ Monitoring et alerting
- ✅ Incident response
- ✅ Linux administration avancée
- ✅ Scripting (bash, Python)

Si certains concepts ne sont pas clairs, revoyez les parties précédentes ou documentez-vous sur les systèmes distribués avant de continuer.

---

## ⚠️ Avertissements critiques

### Les erreurs qui cassent la HA

1. **❌ Sous-estimer la latence réseau**
   - Galera multi-DC avec 100ms latency → performances catastrophiques
   - **Solution** : <20ms pour Galera, sinon réplication async

2. **❌ Négliger les tests de failover**
   - "Notre failover fonctionne" basé sur théorie, pas pratique
   - **Solution** : Drill mensuel, failover planifié trimestriel

3. **❌ Nombre pair de nœuds Galera**
   - 2 ou 4 nœuds → risque de split-brain 50/50
   - **Solution** : Toujours impair (3, 5, 7) ou Garbd

4. **❌ Ignorer le monitoring**
   - Découvrir le lag de réplication via utilisateurs
   - **Solution** : Alerting proactif sur toutes les métriques critiques

5. **❌ Oublier les dépendances externes**
   - Base HA mais DNS single point of failure
   - **Solution** : HA à tous les niveaux, pas seulement DB

6. **❌ Surcharger le primary avec lectures**
   - Primary surchargé → lent → replicas en lag → tout dégradé
   - **Solution** : Read/Write split strict avec MaxScale

7. **❌ Pas de plan de rollback**
   - Upgrade raté sans possibilité de revenir en arrière
   - **Solution** : Toujours un plan B documenté et testé

---

## 🚀 Prêt pour l'infrastructure critique ?

Cette partie vous prépare à gérer des systèmes où **chaque seconde d'indisponibilité compte**. Vous allez acquérir les compétences pour :

- ✅ Concevoir des architectures capables de résister à des défaillances multiples
- ✅ Atteindre des SLA de 99.99% ou plus
- ✅ Opérer des maintenances sans downtime
- ✅ Récupérer d'incidents en quelques secondes
- ✅ Valider des upgrades sans risque avec Workload Replay
- ✅ Dormir tranquille en sachant votre infrastructure résiliente

Les compétences de cette partie sont **différenciantes sur le marché**. Les DBA et DevOps maîtrisant Galera, MaxScale, et les stratégies de HA avancées sont très recherchés et bien rémunérés.

**Préparez-vous à construire des systèmes qui ne tombent jamais.** 🛡️

---

## ➡️ Prochaine étape

**Module 13 : Réplication** → Maîtrisez la réplication asynchrone et semi-synchrone, GTID, topologies avancées, et techniques de failover. La base de toute stratégie de haute disponibilité.

Bienvenue dans le monde de l'infrastructure qui ne dort jamais ! 🌐

---

**MariaDB** : Version 11.8 LTS  
**MaxScale** : Version 25.01

⏭️ [Réplication](/13-replication/README.md)

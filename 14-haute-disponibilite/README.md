🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14. Haute Disponibilité

> **Niveau** : Expert  
> **Durée estimée** : 12-15 heures  
> **Prérequis** : Chapitre 13 (Réplication), maîtrise de l'administration MariaDB, expérience en architecture distribuée

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Concevoir** des architectures haute disponibilité adaptées aux besoins métier (RTO/RPO)
- **Déployer et configurer** MariaDB Galera Cluster en production
- **Implémenter** MaxScale pour le load balancing, routing intelligent et sécurité
- **Maîtriser** les nouveautés MaxScale 25.01 (Workload Capture/Replay, Diff Router)
- **Gérer** les situations critiques : split-brain, quorum, failover automatique
- **Exploiter** les fonctionnalités 11.8 : Transaction Replay et Connection Migration
- **Mettre en place** des stratégies de récupération après incident robustes

---

## Introduction

La haute disponibilité (HA) est un enjeu critique pour les systèmes de bases de données modernes. Dans un contexte où l'indisponibilité d'une application peut coûter des milliers d'euros par minute, garantir une disponibilité de **99.99%** (moins de 53 minutes d'indisponibilité par an) ou **99.999%** (moins de 5 minutes par an) devient un impératif business.

MariaDB offre un écosystème complet de solutions HA, depuis la réplication asynchrone traditionnelle jusqu'aux clusters synchrones multi-master avec Galera, en passant par des proxies intelligents comme MaxScale qui permettent de maintenir la continuité de service même lors de défaillances matérielles ou logicielles.

### 🔍 Pourquoi la Haute Disponibilité ?

La haute disponibilité répond à plusieurs objectifs stratégiques :

1. **Continuité de service** : Maintenir l'application opérationnelle 24/7/365
2. **Protection contre les défaillances** : Matériel, réseau, datacenter, erreurs humaines
3. **Maintenance sans interruption** : Mises à jour, migrations, optimisations
4. **Performance distribuée** : Répartition de charge sur plusieurs nœuds
5. **Conformité réglementaire** : Respect des SLA contractuels ou légaux

### 📊 Concepts Fondamentaux

**RTO (Recovery Time Objective)** : Temps maximal acceptable d'interruption de service  
**RPO (Recovery Point Objective)** : Perte de données maximale acceptable (en temps)

| Architecture | RTO | RPO | Complexité | Coût |
|--------------|-----|-----|------------|------|
| **Standalone avec backup** | Heures | Minutes à heures | Faible | Faible |
| **Réplication async** | Minutes | Secondes à minutes | Moyenne | Moyen |
| **Réplication semi-sync** | Minutes | Quasi-zéro | Moyenne | Moyen |
| **Galera Cluster** | Secondes | Zéro | Élevée | Élevé |
| **Multi-datacenter** | Secondes | Zéro | Très élevée | Très élevé |

### 🏗️ Vue d'Ensemble du Chapitre

Ce chapitre est structuré pour vous accompagner de la théorie à la pratique opérationnelle :

#### **Partie 1 : Fondations Architecturales**
- **Section 14.1** : Principes et patterns d'architectures HA
- Compréhension des trade-offs : CAP theorem, consistency vs availability

#### **Partie 2 : MariaDB Galera Cluster**
- **Section 14.2** : Architecture synchrone multi-master
  - Certification-based replication
  - Configuration et déploiement production
  - State transfers (SST/IST)
- **Section 14.3** : Gestion du split-brain et quorum
  - Mécanismes de protection
  - Stratégies de résolution

#### **Partie 3 : MaxScale - Le Proxy Intelligent**
- **Section 14.4** : Fonctionnalités core
  - Load Balancing intelligent
  - Read/Write Split automatique
  - Query Routing avancé
  - Database Firewall pour la sécurité

- **Section 14.5** : 🆕 Nouveautés MaxScale 25.01
  - **Workload Capture** : Enregistrement du trafic production
  - **Workload Replay** : Tests de charge réalistes
  - **Diff Router** : Comparaison de versions en temps réel

#### **Partie 4 : Failover et Résilience**
- **Section 14.6** : Solutions de failover automatique
- **Section 14.7** : Virtual IP et keepalived
- **Section 14.8** : Stratégies de récupération après incident

#### **Partie 5 : Alternatives et Innovations**
- **Section 14.9** : ProxySQL et HAProxy
- **Section 14.10** : 🆕 Transaction Replay et Connection Migration (11.8)

---

## 🆕 Nouveautés MariaDB 11.8 pour la Haute Disponibilité

### **1. Transaction Replay (Rejouabilité Automatique)**

MariaDB 11.8 introduit la capacité de rejouer automatiquement les transactions en cas de défaillance d'un nœud :

```sql
-- Configuration de Transaction Replay
SET GLOBAL transaction_replay = ON;
SET GLOBAL transaction_replay_attempts = 3;
SET GLOBAL transaction_replay_timeout = 30; -- secondes
```

**Avantages** :
- ✅ Réduction du RTO : moins d'interventions manuelles
- ✅ Transparence applicative : l'application n'a pas besoin de gérer le retry
- ✅ Cohérence garantie : rejeu uniquement si la transaction n'a pas été committée

**Cas d'usage** :
```sql
-- Transaction automatiquement rejouée en cas de failover
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 123;
UPDATE accounts SET balance = balance + 100 WHERE id = 456;
COMMIT; -- Si échec ici, rejouée automatiquement sur nouveau primary
```

### **2. Connection Migration (Migration Transparente)**

Permet de migrer les connexions actives vers un autre nœud sans perte de session :

```sql
-- Configuration de Connection Migration
SET GLOBAL connection_migration = ON;
SET GLOBAL connection_migration_preserve_session = ON;
```

**Fonctionnalités** :
- Préservation des variables de session
- Maintien des prepared statements
- Continuité des transactions (couplé avec Transaction Replay)

**Impact sur l'architecture** :
```
Client → MaxScale → Primary (défaillance détectée)
                  ↓
              Migration automatique
                  ↓
Client → MaxScale → New Primary (session préservée)
```

### **3. Améliorations Galera 4.x**

- **Streaming Replication** : Réplication de très grandes transactions par fragments
- **Parallel Applying** : Amélioration du parallélisme sur les replicas
- **Automatic IST** : Déclenchement automatique d'IST au lieu de SST quand possible

---

## 🎯 Architecture de Référence : Production-Ready HA

Voici une architecture complète combinant les meilleures pratiques :

```
                          ┌─────────────────┐
                          │   Application   │
                          │    Servers      │
                          └────────┬────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
              ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
              │ MaxScale  │  │ MaxScale  │  │ MaxScale  │
              │  Primary  │  │  Standby  │  │  Standby  │
              │ (VIP)     │  │           │  │           │
              └─────┬─────┘  └────┬──────┘  └───┬───────┘
                    │             │             │
        ┌───────────┼─────────────┼─────────────┼───────────┐
        │           │             │             │           │
   ┌────▼────┐ ┌────▼────┐  ┌─────▼───┐  ┌──────▼──┐ ┌──────▼──┐
   │ Galera  │ │ Galera  │  │ Galera  │  │ Galera  │ │ Galera  │
   │ Node 1  │ │ Node 2  │  │ Node 3  │  │ Node 4  │ │ Node 5  │
   │  DC1    │ │  DC1    │  │  DC2    │  │  DC2    │ │  DC3    │
   │ Primary │ │ Primary │  │ Primary │  │ Primary │ │ Arbitr. │
   └─────────┘ └─────────┘  └─────────┘  └─────────┘ └─────────┘
```

**Caractéristiques** :
- **5 nœuds Galera** : Tolérance de 2 pannes simultanées
- **3 datacenters** : Protection contre défaillance datacenter
- **3 instances MaxScale** : Pas de SPOF au niveau proxy
- **VIP avec keepalived** : Basculement automatique MaxScale
- **Nœud arbitre (DC3)** : Garagekeeper pour quorum, sans données

**Disponibilité théorique** : 99.999% (Five Nines)

---

## 📋 Matrice de Décision : Quelle Solution HA ?

| Critère | Réplication Async | Galera Cluster | Multi-DC Galera |
|---------|-------------------|----------------|-----------------|
| **RTO** | 1-5 minutes | < 30 secondes | < 30 secondes |
| **RPO** | Secondes-minutes | Zéro | Zéro |
| **Writes simultanés** | ❌ Non | ✅ Oui | ✅ Oui |
| **Latence réseau** | Tolérant | < 10ms recommandé | < 100ms |
| **Complexité** | Faible | Moyenne | Élevée |
| **Conflits** | N/A | Possibles (rares) | Plus fréquents |
| **Scalabilité reads** | ✅ Excellente | ✅ Excellente | ✅ Excellente |
| **Scalabilité writes** | ❌ Limitée | ⚠️ Moyenne | ⚠️ Moyenne |
| **Coût infrastructure** | Faible | Moyen | Élevé |

---

## ⚠️ Points Critiques à Maîtriser

### **1. Split-Brain : Le Cauchemar des Architectures Distribuées**

Un split-brain survient quand deux parties d'un cluster croient être le cluster légitime :

```
Cluster Initial (3 nœuds)
    A ←→ B ←→ C
         ↓ Partition réseau
    A (seul)     B ←→ C
    ↓ Continue   ↓ Continue
  SPLIT-BRAIN !
```

**Conséquences** :
- Divergence des données
- Violations de contraintes
- Corruption potentielle
- Résolution manuelle complexe

**Solutions** :
- Quorum obligatoire (minimum 3 nœuds)
- Garagekeeper (arbitre)
- Fencing automatique
- Monitoring proactif

### **2. Quorum et Formation de Cluster**

Le quorum garantit qu'une majorité de nœuds est d'accord :

```
3 nœuds : Quorum = 2 (tolère 1 panne)
5 nœuds : Quorum = 3 (tolère 2 pannes)
7 nœuds : Quorum = 4 (tolère 3 pannes)
```

**Formule** : `Quorum = (Nodes / 2) + 1`

### **3. Failover : Automatique vs Manuel**

**Failover Automatique** :
- ✅ RTO minimal (< 1 minute)
- ✅ Disponibilité 24/7
- ⚠️ Risque de "flapping"
- ⚠️ Nécessite configuration précise

**Failover Manuel** :
- ✅ Contrôle total
- ✅ Évite les décisions précipitées
- ❌ RTO plus long
- ❌ Nécessite astreinte

**Recommandation** : Automatique avec supervision humaine et alerting robuste

---

## 🔧 Outils de l'Écosystème HA

| Outil | Fonction | Niveau |
|-------|----------|--------|
| **MaxScale** | Proxy HA, routing, firewall | Production |
| **ProxySQL** | Alternative proxy, query caching | Production |
| **HAProxy** | Load balancing TCP/HTTP | Production |
| **Keepalived** | Virtual IP, VRRP | Production |
| **Corosync/Pacemaker** | Cluster management | Avancé |
| **Orchestrator** | Topology management, failover | Avancé |
| **MHA (Master HA)** | Failover MySQL/MariaDB | Legacy |
| **ClusterControl** | Management GUI complet | Commercial |

---

## 📚 Structure des Sections Suivantes

Chaque section de ce chapitre est conçue pour être autonome mais s'intègre dans une progression logique :

1. **14.1** : Établir les fondations théoriques
2. **14.2-14.3** : Maîtriser Galera Cluster en profondeur
3. **14.4-14.5** : Exploiter MaxScale et ses nouveautés
4. **14.6-14.8** : Gérer les situations opérationnelles
5. **14.9-14.10** : Explorer les alternatives et innovations

---

## 💡 Conseils pour Aborder ce Chapitre

### **Pour les Architectes**
- Concentrez-vous sur les sections 14.1, 14.8, et la matrice de décision
- Évaluez les trade-offs CAP selon vos contraintes métier
- Considérez les coûts opérationnels, pas seulement techniques

### **Pour les DBA Senior**
- Pratiquez les sections 14.2-14.3 en environnement de test
- Maîtrisez les commandes de diagnostic Galera
- Préparez des runbooks pour chaque scénario de défaillance

### **Pour les DevOps/SRE**
- Automatisez les déploiements (sections 14.2, 14.4)
- Intégrez le monitoring (Prometheus, Grafana)
- Testez régulièrement vos procédures de failover

---

## ⚡ Quick Start : Lab HA en 30 Minutes

Pour expérimenter rapidement avec la HA :

```bash
# Déployer un cluster Galera 3 nœuds avec Docker Compose
git clone https://github.com/mariadb-corporation/mariadb-docker-galera
cd mariadb-docker-galera
docker-compose up -d

# Déployer MaxScale
docker run -d \
  --name maxscale \
  --network mariadb-galera_default \
  -v $(pwd)/maxscale.cnf:/etc/maxscale.cnf \
  mariadb/maxscale:25.01

# Tester le failover
docker stop galera-node1
# Observer la bascule automatique dans MaxScale
```

Cette configuration de lab vous permettra d'expérimenter les concepts avant de les déployer en production.

---

## ✅ Prérequis Techniques Avant de Continuer

Assurez-vous de maîtriser :

- [ ] Réplication MariaDB (Chapitre 13)
- [ ] Binary logs et GTID
- [ ] Transactions et niveaux d'isolation
- [ ] Configuration réseau (latence, bande passante)
- [ ] Linux : iptables, systemd, monitoring
- [ ] Concepts de quorum et consensus distribué

Si vous n'êtes pas à l'aise avec ces sujets, revoyez le Chapitre 13 avant de poursuivre.

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 MariaDB Galera Cluster](https://mariadb.com/kb/en/galera-cluster/)
- [📖 MaxScale 25.01 Documentation](https://mariadb.com/kb/en/maxscale/)
- [📖 High Availability Solutions](https://mariadb.com/kb/en/high-availability/)

### Articles Techniques
- **"Understanding Galera Cluster"** - MariaDB Corporation
- **"MaxScale Best Practices"** - MariaDB Enterprise Guide
- **"CAP Theorem and MariaDB"** - Architectural Whitepaper

### Outils Open Source
- [Galera Cluster GitHub](https://github.com/codership/galera)
- [MaxScale GitHub](https://github.com/mariadb-corporation/MaxScale)
- [Orchestrator](https://github.com/openark/orchestrator)

---

## ➡️ Section Suivante

**[14.1 Architectures haute disponibilité : Concepts](/14-haute-disponibilite/01-architectures-ha-concepts.md)**

Dans la section suivante, nous poserons les fondations théoriques essentielles : le théorème CAP, les patterns d'architecture (active-passive, active-active, multi-datacenter), et les métriques de disponibilité. Nous établirons également un framework de décision pour choisir l'architecture adaptée à vos besoins.

---

**Bonne exploration de la haute disponibilité avec MariaDB ! 🚀**

*Ce chapitre représente le sommet de l'expertise opérationnelle MariaDB. Prenez le temps de bien assimiler chaque concept et de les tester dans des environnements contrôlés avant toute mise en production.*

⏭️ [Architectures haute disponibilité : Concepts](/14-haute-disponibilite/01-architectures-ha-concepts.md)

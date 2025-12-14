🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.4 MaxScale

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Section 14.2 (Galera Cluster), Section 14.3 (Quorum), administration réseau

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et l'architecture de MaxScale dans un écosystème haute disponibilité
- **Installer et configurer** MaxScale pour environnements de production
- **Implémenter** les services essentiels : Load Balancing, Read/Write Split, Query Routing
- **Sécuriser** votre infrastructure avec le Database Firewall
- **Exploiter** les nouveautés MaxScale 25.01 : Workload Capture/Replay, Diff Router
- **Monitorer** et troubleshooter MaxScale en production
- **Comparer** MaxScale avec les alternatives (ProxySQL, HAProxy)

---

## Introduction

**MaxScale** est un proxy de base de données intelligent développé par MariaDB Corporation. Il agit comme une couche d'abstraction entre les applications et les serveurs MariaDB, offrant des fonctionnalités avancées de **routing**, **load balancing**, **haute disponibilité** et **sécurité**.

Dans une architecture Galera Cluster, MaxScale joue un rôle critique :
- Détecte automatiquement l'état des nœuds (PRIMARY/NON-PRIMARY)
- Route intelligemment les requêtes selon leur type (READ/WRITE)
- Fournit un failover transparent pour les applications
- Masque la complexité du cluster aux applications

> 💡 **Analogie** : Si Galera est le "cerveau distribué" de votre base de données, MaxScale est le "système nerveux" qui coordonne les communications et prend des décisions de routing intelligentes.

### 🆕 Nouveautés MaxScale 25.01 (Janvier 2025)

MaxScale 25.01, sorti en janvier 2025, apporte des fonctionnalités révolutionnaires pour le testing et la validation :

1. **Workload Capture** : Enregistrement du trafic production réel
2. **Workload Replay** : Rejeu fidèle pour tests de charge réalistes
3. **Diff Router** : Comparaison en temps réel entre versions MariaDB

Ces outils transforment la façon dont nous testons les upgrades, validons les performances et détectons les régressions.

---

## 1. Vue d'Ensemble et Positionnement

### 1.1 Qu'est-ce que MaxScale ?

**MaxScale** est un **proxy SQL intelligent** qui :

```
┌───────────────────────────────────────────────────┐
│              APPLICATION TIER                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  App 1   │  │  App 2   │  │  App 3   │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
└───────┼─────────────┼─────────────┼───────────────┘
        │             │             │
        │   Single Connection Point │
        └─────────────┼─────────────┘
                      │
        ┌─────────────▼─────────────┐
        │       MaxScale Proxy      │
        │                           │
        │  ┌─────────────────────┐  │
        │  │  Intelligent        │  │
        │  │  Query Routing      │  │
        │  └─────────────────────┘  │
        │                           │
        │  ┌─────────────────────┐  │
        │  │  Load Balancing     │  │
        │  └─────────────────────┘  │
        │                           │
        │  ┌─────────────────────┐  │
        │  │  Failover Detection │  │
        │  └─────────────────────┘  │
        │                           │
        │  ┌─────────────────────┐  │
        │  │  Database Firewall  │  │
        │  └─────────────────────┘  │
        └─────┬──────────┬──────┬───┘
              │          │      │
    ┌─────────▼──┐  ┌────▼────┐ └─────▼──────┐
    │ Galera     │  │ Galera  │   │ Galera   │
    │ Node 1     │  │ Node 2  │   │ Node 3   │
    │ (PRIMARY)  │  │(PRIMARY)│   │(PRIMARY) │
    └────────────┘  └─────────┘   └──────────┘
```

**Fonctionnalités principales** :

1. **Query Routing** : Dirige les requêtes vers les serveurs appropriés
2. **Load Balancing** : Répartit la charge sur plusieurs backends
3. **Read/Write Split** : Sépare automatiquement reads et writes
4. **High Availability** : Détection de pannes et failover automatique
5. **Connection Pooling** : Réutilisation de connexions backend
6. **Query Filtering** : Firewall, rewriting, logging
7. **Protocol Translation** : Support MySQL, PostgreSQL (expérimental)
8. **Monitoring** : Métriques et diagnostics avancés

### 1.2 Architecture Interne

```
┌──────────────────── MaxScale Process ───────────────────┐
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Listener (Port 3306)               │    │
│  │  - Accept client connections                    │    │
│  │  - SSL/TLS termination                          │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐    │
│  │           Protocol Module (MariaDB)             │    │
│  │  - Parse SQL statements                         │    │
│  │  - Extract query type (SELECT/INSERT/etc)       │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐    │
│  │              Router Module                      │    │
│  │  - readwritesplit / readconnroute /             │    │
│  │    schemarouter / binlogrouter                  │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐    │
│  │             Filter Chain                        │    │
│  │  - Database Firewall                            │    │
│  │  - Query Log All (QLA)                          │    │
│  │  - Regex Filter                                 │    │
│  │  - Cache Filter                                 │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐    │
│  │            Monitor Module                       │    │
│  │  - Galera Monitor                               │    │
│  │  - MariaDB Monitor                              │    │
│  │  - Health checks                                │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                 │
│  ┌────────────────────▼────────────────────────────┐    │
│  │          Backend Connection Pool                │    │
│  │  - Persistent connections to servers            │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                 │
└───────────────────────┼─────────────────────────────────┘
                        │
            ┌───────────┼───────────┐
            │           │           │
        ┌───▼───┐   ┌───▼───┐  ┌────▼──┐
        │Server1│   │Server2│  │Server3│
        └───────┘   └───────┘  └───────┘
```

**Composants clés** :

- **Listener** : Point d'entrée pour connexions clients
- **Protocol** : Analyse et compréhension du protocole SQL
- **Router** : Logique de routing (où envoyer la requête ?)
- **Filter** : Transformation/validation des requêtes
- **Monitor** : Surveillance santé des backends
- **Backend Pool** : Gestion connexions aux serveurs

### 1.3 Positionnement dans la Stack

```
┌────────────── Layer 7 (Application) ──────────────┐
│  Web Servers, Microservices, API Gateway          │
└──────────────────────┬────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────┐
│              Load Balancer (L4)                   │
│         (HAProxy, Nginx, AWS ALB)                 │
└──────────────────────┬────────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────▼────┐   ┌────▼────┐  ┌─────▼───┐
    │MaxScale │   │MaxScale │  │MaxScale │
    │Primary  │   │Standby  │  │Standby  │
    └────┬────┘   └───┬─────┘  └───┬─────┘
         │            │            │
         └────────────┼────────────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
    ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
    │ Galera  │  │ Galera  │  │ Galera  │
    │ Node 1  │  │ Node 2  │  │ Node 3  │
    └─────────┘  └─────────┘  └─────────┘
```

**Avantages de cette architecture** :

- ✅ **Single point of entry** : Applications connectent à MaxScale uniquement
- ✅ **Transparence** : Cluster invisible pour l'application
- ✅ **Failover automatique** : MaxScale détecte et route autour des pannes
- ✅ **Scalabilité** : Ajout/retrait nœuds sans changement applicatif
- ✅ **Sécurité** : Firewall centralisé, filtrage requêtes

---

## 2. Installation et Configuration de Base

### 2.1 Installation

#### **Ubuntu/Debian**

```bash
# Ajouter le repository MariaDB
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | \
  sudo bash -s -- --mariadb-maxscale

# Installer MaxScale 25.01
apt-get update
apt-get install maxscale

# Vérifier version
maxscale --version
# MaxScale 25.01.0

# Vérifier service
systemctl status maxscale
```

#### **RHEL/CentOS**

```bash
# Repository setup
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | \
  sudo bash -s -- --mariadb-maxscale

# Installation
yum install maxscale

# Démarrage
systemctl enable maxscale
systemctl start maxscale
```

#### **Docker (pour développement)**

```bash
# Image officielle
docker pull mariadb/maxscale:25.01

# Lancement avec configuration
docker run -d \
  --name maxscale \
  -p 3306:3306 \
  -p 8989:8989 \
  -v /path/to/maxscale.cnf:/etc/maxscale.cnf \
  mariadb/maxscale:25.01
```

### 2.2 Structure de Configuration

```bash
# Fichier de configuration principal
/etc/maxscale.cnf

# Structure recommandée (modulaire)
/etc/maxscale.cnf.d/
├── servers.cnf          # Définition backends
├── monitors.cnf         # Monitors Galera/MariaDB
├── services.cnf         # Services (routing)
├── listeners.cnf        # Listeners (ports)
└── filters.cnf          # Filtres optionnels

# Logs
/var/log/maxscale/maxscale.log

# Runtime state
/var/lib/maxscale/
```

### 2.3 Configuration Minimale Fonctionnelle

```ini
# /etc/maxscale.cnf - Configuration de base Galera

[maxscale]
# Threads de traitement (1 par core recommandé)
threads = auto

# Logging
log_info = true
log_warning = true
log_notice = true

# ============================================
# SERVERS
# ============================================
[server1]
type = server
address = 10.0.1.10
port = 3306
protocol = MariaDBBackend

[server2]
type = server
address = 10.0.1.11
port = 3306
protocol = MariaDBBackend

[server3]
type = server
address = 10.0.1.12
port = 3306
protocol = MariaDBBackend

# ============================================
# MONITOR (Galera)
# ============================================
[Galera-Monitor]
type = monitor
module = galeramon
servers = server1, server2, server3
user = maxscale_monitor
password = SecureMonitorPassword

# Health check interval
monitor_interval = 2000ms

# Désactiver nœuds non-synced
disable_master_failback = false
available_when_donor = false
use_priority = false

# ============================================
# SERVICE (Read/Write Split)
# ============================================
[Read-Write-Service]
type = service
router = readwritesplit
servers = server1, server2, server3
user = maxscale_user
password = SecureServicePassword

# Configuration Read/Write Split
master_failure_mode = fail_on_write
master_reconnection = true
transaction_replay = true
delayed_retry = true
delayed_retry_timeout = 60s

# ============================================
# LISTENER
# ============================================
[Read-Write-Listener]
type = listener
service = Read-Write-Service
protocol = MariaDBClient
port = 3306
address = 0.0.0.0
```

### 2.4 Création des Utilisateurs MaxScale

```sql
-- Sur l'un des nœuds Galera (sera répliqué sur tous)

-- 1. Utilisateur Monitor (surveillance cluster)
CREATE USER 'maxscale_monitor'@'%' 
  IDENTIFIED BY 'SecureMonitorPassword';

GRANT SELECT ON mysql.user TO 'maxscale_monitor'@'%';
GRANT SELECT ON mysql.db TO 'maxscale_monitor'@'%';
GRANT SELECT ON mysql.tables_priv TO 'maxscale_monitor'@'%';
GRANT SELECT ON mysql.columns_priv TO 'maxscale_monitor'@'%';
GRANT SELECT ON mysql.procs_priv TO 'maxscale_monitor'@'%';
GRANT SELECT ON mysql.proxies_priv TO 'maxscale_monitor'@'%';
GRANT SELECT ON mysql.roles_mapping TO 'maxscale_monitor'@'%';
GRANT SHOW DATABASES ON *.* TO 'maxscale_monitor'@'%';

-- Permissions Galera spécifiques
GRANT REPLICATION CLIENT ON *.* TO 'maxscale_monitor'@'%';

-- 2. Utilisateur Service (routing requêtes)
CREATE USER 'maxscale_user'@'%' 
  IDENTIFIED BY 'SecureServicePassword';

GRANT SELECT ON mysql.user TO 'maxscale_user'@'%';
GRANT SELECT ON mysql.db TO 'maxscale_user'@'%';
GRANT SELECT ON mysql.tables_priv TO 'maxscale_user'@'%';
GRANT SELECT ON mysql.roles_mapping TO 'maxscale_user'@'%';
GRANT SHOW DATABASES ON *.* TO 'maxscale_user'@'%';

-- 3. Utilisateur Admin MaxScale (GUI/REST API)
CREATE USER 'maxscale_admin'@'%' 
  IDENTIFIED BY 'SecureAdminPassword';

GRANT ALL PRIVILEGES ON *.* TO 'maxscale_admin'@'%';

FLUSH PRIVILEGES;
```

### 2.5 Vérification et Démarrage

```bash
# Vérifier syntaxe configuration
maxscale --config-check

# Démarrer MaxScale
systemctl start maxscale

# Vérifier logs
tail -f /var/log/maxscale/maxscale.log

# Devrait afficher :
# MariaDB MaxScale 25.01.0 started
# Galera-Monitor: All 3 servers are running
# Read-Write-Listener: Listening on [0.0.0.0]:3306

# Tester connexion via MaxScale
mysql -h maxscale.example.com -P 3306 -u myapp -p

# Vérifier routing
mysql -h maxscale.example.com -P 3306 -u myapp -p -e "SELECT @@hostname"
# Devrait retourner l'un des nœuds Galera
```

---

## 3. Modules et Composants Essentiels

### 3.1 Routers Disponibles

MaxScale propose plusieurs routers spécialisés :

#### **readwritesplit** (Recommandé pour Galera)

```ini
[RW-Service]
type = service
router = readwritesplit
servers = server1, server2, server3

# Caractéristiques :
# ✅ Route automatiquement SELECTs vers replicas
# ✅ Route writes (INSERT/UPDATE/DELETE) vers master
# ✅ Maintient cohérence transactionnelle
# ✅ Session sticky après premier write
```

**Cas d'usage** :
- Applications OLTP standard (reads >> writes)
- E-commerce, CMS, SaaS
- Charge mixte read/write

#### **readconnroute** (Round-Robin Simple)

```ini
[RoundRobin-Service]
type = service
router = readconnroute
router_options = master

# Caractéristiques :
# ✅ Simple round-robin sur tous les serveurs
# ✅ Bonne répartition de charge
# ❌ Pas de distinction read/write
```

**Cas d'usage** :
- Applications read-only (reporting, analytics)
- Charge homogène entre nœuds
- Besoin de load balancing pur

#### **schemarouter** (Routing par Base de Données)

```ini
[Schema-Service]
type = service
router = schemarouter
servers = server1, server2, server3

# Caractéristiques :
# ✅ Route par database (sharding applicatif)
# ✅ Permet isolation par tenant
# ✅ Scalabilité horizontale
```

**Cas d'usage** :
- Architecture multi-tenant (database par client)
- Sharding manuel
- Isolation de charges (analytics vs OLTP)

#### **🆕 differencerouter** (Comparaison en Temps Réel)

```ini
[Diff-Service]
type = service
router = differencerouter
servers = server1, server2
match_host = server1
target_host = server2

# Nouveauté MaxScale 25.01
# Envoie requêtes aux deux serveurs
# Compare résultats et timings
# Log différences
```

**Cas d'usage** (détaillé en section 14.5.3) :
- Validation upgrade MariaDB 11.4 → 11.8
- Détection régressions
- A/B testing requêtes

### 3.2 Monitors Disponibles

#### **galeramon** (Monitor Galera)

```ini
[Galera-Monitor]
type = monitor
module = galeramon
servers = server1, server2, server3
user = maxscale_monitor
password = SecurePassword

# Configuration spécifique Galera
monitor_interval = 2000ms
disable_master_failback = false
available_when_donor = true
use_priority = false

# Variables wsrep surveillées
# - wsrep_cluster_status (Primary/Non-Primary)
# - wsrep_local_state (4 = Synced)
# - wsrep_cluster_size
# - wsrep_ready
```

**Détection automatique** :
- Nœud NON-PRIMARY → Marqué indisponible
- Nœud en état DONOR → Disponible ou non selon config
- Nœud déconnecté → Retrait automatique du pool

#### **mariadbmon** (Monitor MariaDB Réplication)

```ini
[MariaDB-Monitor]
type = monitor
module = mariadbmon
servers = master, slave1, slave2
user = maxscale_monitor
password = SecurePassword

# Failover automatique
auto_failover = true
auto_rejoin = true
switchover_on_low_disk_space = true

# Timeouts
failcount = 3
failover_timeout = 90s
```

**Cas d'usage** :
- Réplication master-slave traditionnelle
- Failover automatique master
- Promotion automatique slave

### 3.3 Filters (Chaîne de Traitement)

Les filters permettent d'intercepter et modifier les requêtes :

```
Client Request
      │
      ▼
┌──────────────┐
│   Filter 1   │  Database Firewall
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Filter 2   │  Query Log All
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Filter 3   │  Regex Filter
└──────┬───────┘
       │
       ▼
    Router
      │
      ▼
   Backend
```

**Filters disponibles** :
- **Database Firewall** (dbfwfilter) : Blocage requêtes dangereuses
- **Query Log All** (qla) : Logging exhaustif requêtes
- **Regex Filter** (regexfilter) : Rewriting requêtes
- **Cache** (cache) : Mise en cache résultats SELECT
- **Tee Filter** (tee) : Duplication requêtes (testing)
- **Masking Filter** (masking) : Masquage données sensibles

---

## 4. Fonctionnalités Principales

### 4.1 Load Balancing

**Algorithmes disponibles** :

```ini
[LB-Service]
type = service
router = readconnroute

# ADAPTIVE_ROUTING (par défaut, recommandé)
router_options = running

# Critères :
# - Connections actives sur backend
# - Charge CPU serveur
# - Temps de réponse moyen

# Alternative : LEAST_CURRENT_OPERATIONS
router_options = least_current_operations
```

💡 Détails complets en section **14.4.1**

### 4.2 Read/Write Split

**Fonctionnement intelligent** :

```sql
-- Connection à MaxScale
mysql -h maxscale -u app -p

-- SELECT routé vers replica (server2 ou server3)
SELECT * FROM users WHERE id = 123;
-- Exécuté sur : server2

-- INSERT routé vers master
BEGIN;
INSERT INTO orders (user_id, amount) VALUES (123, 99.99);
-- Exécuté sur : server1 (master)

-- Tous les SELECTs suivants dans cette transaction
-- routés vers master pour cohérence
SELECT * FROM orders WHERE user_id = 123;
-- Exécuté sur : server1 (même serveur que INSERT)

COMMIT;

-- Après COMMIT, retour au routing normal
SELECT * FROM products WHERE price < 100;
-- Exécuté sur : server3 (replica)
```

**Configuration avancée** :

```ini
[RW-Service]
type = service
router = readwritesplit

# Sticky sessions après premier write
transaction_replay = true

# Retry automatique en cas de failover
master_reconnection = true
delayed_retry = true
delayed_retry_timeout = 60s

# Causal reads (cohérence après write)
causal_reads = global
causal_reads_timeout = 10s

# 🆕 MaxScale 25.01
# Détaillé en section 14.10
connection_keepalive = 300s
```

💡 Détails complets en section **14.4.2**

### 4.3 Query Routing Avancé

**Routing par regex** :

```ini
[Regex-Service]
type = service
router = readwritesplit
servers = server1, server2, server3

# Filtres de routing
filters = QueryRouter

[QueryRouter]
type = filter
module = regexfilter

# Forcer certaines requêtes vers master
match = ^SELECT.*FOR UPDATE
server = server1
```

**Routing par hint SQL** :

```sql
-- Forcer exécution sur master
-- maxscale route to master
SELECT * FROM users WHERE id = 123;

-- Forcer exécution sur serveur spécifique
-- maxscale route to server server2
SELECT * FROM analytics.big_table;
```

💡 Détails complets en section **14.4.3**

### 4.4 Database Firewall

**Protection multicouche** :

```ini
[Firewall-Filter]
type = filter
module = dbfwfilter
rules = /etc/maxscale.d/firewall_rules.txt

[RW-Service]
type = service
router = readwritesplit
servers = server1, server2, server3
filters = Firewall-Filter
```

```bash
# /etc/maxscale.d/firewall_rules.txt

# Bloquer DROP TABLE en production
rule block_drop_table deny regex '.*DROP\s+TABLE.*'

# Bloquer DELETE sans WHERE
rule block_unsafe_delete deny regex '.*DELETE\s+FROM\s+\w+\s*;'

# Limiter taille résultats
rule limit_rows deny columns > 1000

# Bloquer accès table sensible
rule protect_secrets deny wildcard '*SELECT*FROM*secrets*'

# Whitelist : Autoriser uniquement SELECT/INSERT/UPDATE
rule allowed_ops match regex '^(SELECT|INSERT|UPDATE).*'
users app_user@%
```

💡 Détails complets en section **14.4.4**

---

## 5. 🆕 Nouveautés MaxScale 25.01

### 5.1 Workload Capture (Enregistrement Trafic)

**Fonctionnalité révolutionnaire** : Enregistrer le trafic production réel pour tests.

```ini
[Capture-Service]
type = service
router = readwritesplit
servers = server1, server2, server3

# 🆕 Activation Workload Capture
filters = WorkloadCapture

[WorkloadCapture]
type = filter
module = workloadcapture

# Fichier d'enregistrement
output = /var/lib/maxscale/workload_capture_$(date).log

# Options
capture_reads = true
capture_writes = true
capture_transactions = true

# Limites
max_file_size = 10G
max_duration = 3600s  # 1 heure
```

**Démarrage/arrêt dynamique** :

```bash
# API REST MaxScale
curl -X POST http://localhost:8989/v1/maxscale/modules/workloadcapture/start \
  -u admin:mariadb

# Arrêt
curl -X POST http://localhost:8989/v1/maxscale/modules/workloadcapture/stop \
  -u admin:mariadb

# Fichier généré
/var/lib/maxscale/workload_capture_2025-12-13.log
# Contient :
# - Toutes les requêtes SQL
# - Timestamps précis
# - Paramètres de requêtes
# - Contexte de transaction
# - Métadonnées client
```

**Cas d'usage** :
- Capturer charge production pour testing upgrade
- Reproduire bugs intermittents
- Benchmarking avec workload réaliste
- Validation performance nouvelles versions

💡 Détails complets en section **14.5.1**

### 5.2 Workload Replay (Rejeu Trafic)

**Rejeu fidèle** du trafic capturé :

```ini
[Replay-Service]
type = service
router = workloadreplay

# 🆕 Configuration Replay
workload_file = /var/lib/maxscale/workload_capture_2025-12-13.log
target_servers = server1, server2, server3

# Options rejeu
replay_speed = 1.0     # 1.0 = vitesse réelle, 2.0 = 2x plus rapide
loop = false           # Rejouer une seule fois
start_offset = 0s      # Commencer au début
duration = 3600s       # Rejouer 1 heure de workload

# Métriques
collect_metrics = true
metrics_file = /var/lib/maxscale/replay_metrics.json
```

**Exécution replay** :

```bash
# Démarrer replay
maxscale-replay --config /etc/maxscale.cnf \
  --service Replay-Service \
  --start

# Monitoring en temps réel
maxscale-replay --stats

# Output:
# Queries replayed: 125,438 / 500,000 (25%)
# Current QPS: 1,247
# Avg latency: 12.3ms
# Errors: 3 (0.002%)
```

**Cas d'usage** :
- Load testing avec workload production réaliste
- Validation upgrade MariaDB 11.4 → 11.8
- Sizing infrastructure (combien de CPU/RAM nécessaires ?)
- Détection régressions performance

💡 Détails complets en section **14.5.2**

### 5.3 Diff Router (Comparaison Versions)

**A/B Testing automatisé** entre deux versions MariaDB :

```ini
[Diff-Service]
type = service
router = differencerouter

# 🆕 Configuration Diff
match_host = server1       # MariaDB 11.4 (baseline)
target_host = server2      # MariaDB 11.8 (test)

# Comparaison
compare_results = true
compare_timings = true
compare_errors = true

# Logging différences
diff_log = /var/log/maxscale/diff_router.log
log_level = detailed

# Seuils d'alerte
timing_threshold = 20%    # Alert si >20% différence latence
result_mismatch = error   # Alert si résultats différents
```

**Exemple de sortie** :

```bash
# /var/log/maxscale/diff_router.log

2025-12-13 10:15:23 [INFO] Query: SELECT * FROM users WHERE created_at > '2025-01-01'
  Match (11.4):  Rows=1523, Time=45ms
  Target (11.8): Rows=1523, Time=38ms
  Result: ✅ IDENTICAL, 🚀 FASTER (15.6%)

2025-12-13 10:15:24 [WARNING] Query: SELECT COUNT(*) FROM orders GROUP BY status
  Match (11.4):  Rows=5, Time=123ms
  Target (11.8): Rows=5, Time=89ms
  Result: ✅ IDENTICAL, 🚀 FASTER (27.6%)

2025-12-13 10:15:25 [ERROR] Query: SELECT DATE_FORMAT(timestamp, '%Y-%m-%d')...
  Match (11.4):  Rows=100, Time=56ms
  Target (11.8): Rows=100, Time=52ms
  Result: ❌ DIFFERENT VALUES
  Details: Row 42: '2025-02-31' vs '2025-03-03'
  → Changement comportement DATE_FORMAT pour dates invalides
```

**Cas d'usage** :
- Validation upgrade sans risque (détection régressions)
- Comparaison optimiseur query (11.4 vs 11.8)
- Détection breaking changes
- Documentation différences comportement

💡 Détails complets en section **14.5.3**

---

## 6. Haute Disponibilité de MaxScale

### 6.1 Architecture HA MaxScale

**Problème** : MaxScale lui-même peut devenir un SPOF.

**Solution** : Déployer plusieurs instances MaxScale avec VIP :

```
┌─────────────────────────────────────────┐
│           Applications                  │
└────────────────┬────────────────────────┘
                 │
                 │ Virtual IP (keepalived)
                 │ 10.0.0.100:3306
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼──────┐ ┌───▼──────┐ ┌───▼──────┐
│MaxScale1 │ │MaxScale2 │ │MaxScale3 │
│ MASTER   │ │ BACKUP   │ │ BACKUP   │
│Priority  │ │Priority  │ │Priority  │
│  100     │ │   90     │ │   80     │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  │
         ┌────────┼────────┐
         │        │        │
     ┌───▼──┐ ┌───▼──┐ ┌───▼──┐
     │Galera│ │Galera│ │Galera│
     │Node1 │ │Node2 │ │Node3 │
     └──────┘ └──────┘ └──────┘
```

### 6.2 Configuration Keepalived

```bash
# /etc/keepalived/keepalived.conf sur MaxScale1

vrrp_script check_maxscale {
    script "/usr/local/bin/check_maxscale.sh"
    interval 2
    weight -20
}

vrrp_instance MAXSCALE_VIP {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100        # 90 sur MaxScale2, 80 sur MaxScale3
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecurePassword
    }
    
    virtual_ipaddress {
        10.0.0.100/24 dev eth0
    }
    
    track_script {
        check_maxscale
    }
    
    notify_master "/usr/local/bin/maxscale_master.sh"
    notify_backup "/usr/local/bin/maxscale_backup.sh"
}
```

```bash
#!/bin/bash
# /usr/local/bin/check_maxscale.sh

# Vérifier que MaxScale répond
maxctrl show maxscale &>/dev/null
if [ $? -ne 0 ]; then
    exit 1
fi

# Vérifier qu'au moins un serveur est disponible
available=$(maxctrl list servers --tsv | grep -c "Running")
if [ "$available" -lt 1 ]; then
    exit 1
fi

exit 0
```

### 6.3 Session Persistence

**Problème** : Basculement VIP coupe les connexions actives.

**Solution 1 : Connection Pooling Applicatif**
```python
# Python avec pooling
import mysql.connector.pooling

pool = mysql.connector.pooling.MySQLConnectionPool(
    pool_name = "mypool",
    pool_size = 10,
    host = "10.0.0.100",  # VIP MaxScale
    database = "mydb",
    pool_reset_session = True
)

# Reconnexion automatique
connection = pool.get_connection()
```

**Solution 2 : 🆕 Connection Migration (MaxScale 25.01 + MariaDB 11.8)**
```ini
[RW-Service]
type = service
router = readwritesplit
servers = server1, server2, server3

# 🆕 Migration de connexions entre instances MaxScale
connection_migration = true
migration_timeout = 30s

# Préserve :
# - Variables de session
# - Prepared statements
# - Transaction state
```

---

## 7. Monitoring et Diagnostics

### 7.1 MaxCtrl (CLI Administration)

```bash
# Liste des serveurs et leur état
maxctrl list servers

# ┌─────────┬─────────────┬──────┬─────────────┬─────────────────┬──────────┐
# │ Server  │ Address     │ Port │ Connections │ State           │ GTID     │
# ├─────────┼─────────────┼──────┼─────────────┼─────────────────┼──────────┤
# │ server1 │ 10.0.1.10   │ 3306 │ 42          │ Master, Running │ 0-1-1234 │
# │ server2 │ 10.0.1.11   │ 3306 │ 38          │ Slave, Running  │ 0-1-1234 │
# │ server3 │ 10.0.1.12   │ 3306 │ 35          │ Slave, Running  │ 0-1-1234 │
# └─────────┴─────────────┴──────┴─────────────┴─────────────────┴──────────┘

# Liste des services
maxctrl list services

# État détaillé d'un serveur
maxctrl show server server1

# Sessions actives
maxctrl list sessions

# Statistiques service
maxctrl show service Read-Write-Service

# Reload configuration (sans restart)
maxctrl reload config
```

### 7.2 REST API

```bash
# MaxScale expose API REST sur port 8989

# Authentification
curl -u admin:mariadb http://localhost:8989/v1/maxscale

# Serveurs
curl -u admin:mariadb http://localhost:8989/v1/servers

# Services
curl -u admin:mariadb http://localhost:8989/v1/services

# Sessions
curl -u admin:mariadb http://localhost:8989/v1/sessions

# Métriques Prometheus
curl http://localhost:9195/metrics
```

### 7.3 Métriques Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'maxscale'
    static_configs:
      - targets: ['maxscale1:9195', 'maxscale2:9195', 'maxscale3:9195']
```

**Métriques clés** :
```promql
# Connexions actives
maxscale_connections

# Requêtes par seconde
rate(maxscale_queries_total[1m])

# Latence moyenne
maxscale_query_duration_seconds

# Erreurs
rate(maxscale_errors_total[1m])

# Serveurs disponibles
maxscale_servers_available
```

### 7.4 Logs et Debugging

```bash
# Configuration logging
# /etc/maxscale.cnf
[maxscale]
log_info = true
log_warning = true
log_notice = true
log_debug = false

# Rotation logs
logrotate /var/log/maxscale/maxscale.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    missingok
}

# Debugging requête spécifique
maxctrl call command mariadbmon debug server1

# Activer tracing
maxctrl set server server1 debug true
```

---

## 8. Comparaison avec Alternatives

### 8.1 MaxScale vs ProxySQL

| Critère | MaxScale | ProxySQL |
|---------|----------|----------|
| **License** | BSL (commercial après 4 ans) | GPL |
| **Galera Support** | ✅ Natif (galeramon) | ✅ Via scripts |
| **Read/Write Split** | ✅ Intelligent | ✅ Via rules |
| **Query Caching** | ✅ Via filter | ✅ Natif (performant) |
| **Firewall** | ✅ dbfwfilter | ⚠️ Via regex |
| **Workload Capture** | 🆕 ✅ Natif (25.01) | ❌ Non |
| **GUI** | ✅ MaxGUI | ✅ ProxySQL Admin |
| **Complexité** | Moyenne | Élevée (règles) |
| **Overhead** | ~5-10% | ~3-5% |
| **Support MariaDB** | ✅ Officiel | ⚠️ Communauté |

**Recommandation** :
- **MaxScale** : Ecosystème MariaDB/Galera, support officiel, fonctionnalités avancées
- **ProxySQL** : Budget limité, query caching essentiel, expertise ProxySQL

### 8.2 MaxScale vs HAProxy

| Critère | MaxScale | HAProxy |
|---------|----------|---------|
| **Layer** | Layer 7 (SQL-aware) | Layer 4 (TCP) |
| **SQL Parsing** | ✅ Oui | ❌ Non |
| **Read/Write Split** | ✅ Automatique | ❌ Impossible |
| **Health Checks** | ✅ SQL-level | ✅ TCP/HTTP |
| **Performance** | Excellent | Exceptionnel |
| **Overhead** | ~5-10% | ~1-2% |
| **Firewall SQL** | ✅ Oui | ❌ Non |

**Recommandation** :
- **MaxScale** : Read/Write split, firewall SQL, intelligence requêtes
- **HAProxy** : Simple load balancing, performance maximale, compatibilité universelle

### 8.3 Cas d'Usage Recommandés

**Utiliser MaxScale quand** :
- ✅ Architecture Galera Cluster
- ✅ Besoin Read/Write Split automatique
- ✅ Firewall SQL requis (sécurité, compliance)
- ✅ Workload Capture/Replay (testing, validation)
- ✅ Support officiel MariaDB souhaité

**Utiliser ProxySQL quand** :
- ✅ Budget limité (open source pur)
- ✅ Query caching critique pour performance
- ✅ Expertise ProxySQL dans l'équipe
- ✅ Besoin de flexibilité maximale (rules complexes)

**Utiliser HAProxy quand** :
- ✅ Simple TCP load balancing suffisant
- ✅ Performance absolue requise (trading, HFT)
- ✅ Stack polyglotte (PostgreSQL, Redis, etc.)
- ✅ Infrastructure déjà standardisée sur HAProxy

---

## ✅ Points Clés à Retenir

- **MaxScale = proxy SQL intelligent** pour haute disponibilité et routing avancé
- **Architecture modulaire** : Listener → Protocol → Router → Filter → Monitor → Backend
- **4 routers principaux** : readwritesplit, readconnroute, schemarouter, differencerouter
- **Galera Monitor** détecte automatiquement PRIMARY/NON-PRIMARY et ajuste routing
- **🆕 MaxScale 25.01** introduit Workload Capture/Replay et Diff Router
- **HA MaxScale** : Déployer 3+ instances avec keepalived (VIP)
- **Monitoring** : maxctrl CLI, REST API, métriques Prometheus
- **Read/Write Split automatique** avec cohérence transactionnelle garantie
- **Database Firewall** pour sécurité et compliance
- **Alternative ProxySQL** pour open source pur, **HAProxy** pour simplicité Layer 4

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 MaxScale Documentation](https://mariadb.com/kb/en/maxscale/)
- [📖 MaxScale 25.01 Release Notes](https://mariadb.com/kb/en/maxscale-25-01-release-notes/)
- [📖 MaxScale Configuration Guide](https://mariadb.com/kb/en/maxscale-configuration-guide/)

### Nouveautés 25.01
- [📖 Workload Capture Documentation](https://mariadb.com/kb/en/maxscale-workload-capture/)
- [📖 Workload Replay Documentation](https://mariadb.com/kb/en/maxscale-workload-replay/)
- [📖 Difference Router](https://mariadb.com/kb/en/maxscale-difference-router/)

### Outils
- [MaxGUI](https://mariadb.com/kb/en/maxgui/) - Interface graphique
- [maxctrl](https://mariadb.com/kb/en/maxctrl/) - CLI administration
- [MaxScale Docker Images](https://hub.docker.com/r/mariadb/maxscale)

### Comparaisons
- [MaxScale vs ProxySQL Benchmark](https://mariadb.com/resources/blog/maxscale-vs-proxysql/)
- [MariaDB Proxy Solutions Comparison](https://mariadb.com/kb/en/choosing-the-right-proxy/)

---

## ➡️ Sections Suivantes

Les 4 sections suivantes approfondiront chaque fonctionnalité principale :

- **14.4.1** : Load Balancing (algorithmes, stratégies, tuning)
- **14.4.2** : Read/Write Split (cohérence, transactions, edge cases)
- **14.4.3** : Query Routing (regex, hints, schemarouter)
- **14.4.4** : Database Firewall (règles, whitelisting, compliance)

Puis la section 14.5 détaillera les nouveautés 25.01 avec exemples pratiques complets.

---

**MaxScale transforme un cluster Galera en solution haute disponibilité enterprise-grade avec routing intelligent, failover automatique et sécurité renforcée.**

⏭️ [Load Balancing](/14-haute-disponibilite/04.1-load-balancing.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.5 Géo-Distribution

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Chapitres 13-14 (Réplication, Haute Disponibilité), Section 20.2 (Microservices), notions de réseau WAN

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les enjeux et contraintes de la distribution géographique des données
- Concevoir des architectures multi-régions avec MariaDB selon différents modèles (Active-Passive, Active-Active)
- Choisir la stratégie de réplication adaptée aux contraintes de latence et de cohérence
- Implémenter des solutions de failover géographique avec Galera et MaxScale
- Gérer les exigences de souveraineté des données (RGPD, data residency)
- Optimiser les performances et la résilience dans un contexte distribué globalement

---

## Introduction

La **géo-distribution** consiste à déployer une base de données sur plusieurs régions géographiques distantes. Cette architecture répond à plusieurs besoins : réduire la latence pour les utilisateurs globaux, assurer la continuité de service en cas de catastrophe régionale, et respecter les exigences de souveraineté des données.

Cependant, la géo-distribution introduit des défis majeurs liés à la physique même des réseaux : la latence entre continents (100-300ms) rend les transactions synchrones impraticables, et le théorème CAP impose des compromis entre cohérence et disponibilité.

MariaDB 11.8 LTS offre plusieurs mécanismes pour adresser ces défis : réplication asynchrone avec GTID, Galera Cluster pour la cohérence intra-région, et MaxScale pour le routage intelligent.

### Pourquoi la géo-distribution ?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MOTIVATIONS DE LA GÉO-DISTRIBUTION                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. RÉDUCTION DE LA LATENCE                                                 │
│  ═══════════════════════════                                                │
│                                                                             │
│  Sans géo-distribution :           Avec géo-distribution :                  │
│                                                                             │
│  Utilisateur Paris                 Utilisateur Paris                        │
│       │                                 │                                   │
│       │ 150ms (RTT)                     │ 5ms                               │
│       ▼                                 ▼                                   │
│  ┌─────────────┐                   ┌─────────────┐                          │
│  │ DB US-East  │                   │  DB Europe  │                          │
│  └─────────────┘                   └─────────────┘                          │
│                                                                             │
│  Latence perçue : 150ms+           Latence perçue : 5ms                     │
│                                                                             │
│  2. DISASTER RECOVERY (DR)                                                  │
│  ═════════════════════════                                                  │
│                                                                             │
│  ┌─────────────┐                   ┌─────────────┐                          │
│  │  Région A   │  ═══════════════► │  Région B   │                          │
│  │  (Primary)  │    Réplication    │    (DR)     │                          │
│  └─────────────┘                   └─────────────┘                          │
│        │                                 │                                  │
│        ▼                                 ▼                                  │
│  Si catastrophe régionale,         Prend le relais avec                     │
│  perte de la région A              RPO < 1 minute                           │
│                                                                             │
│  3. SOUVERAINETÉ DES DONNÉES                                                │
│  ════════════════════════════                                               │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Application Globale                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│        │                    │                    │                          │
│        ▼                    ▼                    ▼                          │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                    │
│  │  EU Data    │     │  US Data    │     │ APAC Data   │                    │
│  │  (RGPD)     │     │  (CCPA)     │     │  (PDPA)     │                    │
│  │  Frankfurt  │     │  Virginia   │     │  Singapore  │                    │
│  └─────────────┘     └─────────────┘     └─────────────┘                    │
│                                                                             │
│  Les données des citoyens européens restent en Europe                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Le défi de la latence réseau

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LATENCES RÉSEAU TYPIQUES                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Intra-datacenter        :   < 1ms     ✅ Galera synchrone OK               │
│  Intra-région (100km)    :   1-5ms     ✅ Galera synchrone OK               │
│  Inter-région même pays  :   10-30ms   ⚠️ Galera limite acceptable          │
│  Transatlantique         :   70-100ms  ❌ Galera déconseillé                │
│  Transpacifique          :   150-200ms ❌ Galera impossible                 │
│  Europe-Asie             :   200-300ms ❌ Async obligatoire                 │
│                                                                             │
│  Impact sur les transactions :                                              │
│  ════════════════════════════                                               │
│                                                                             │
│  Transaction avec 5 écritures + commit Galera :                             │
│  • Intra-DC : 5ms total                                                     │
│  • Transatlantique : 5 × 150ms = 750ms minimum !                            │
│                                                                             │
│  Règle pratique :                                                           │
│  • RTT < 10ms → Galera synchrone viable                                     │
│  • RTT > 30ms → Réplication asynchrone recommandée                          │
│  • RTT > 100ms → Réplication asynchrone obligatoire                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Modèles architecturaux

### 1. Active-Passive (Primary-DR)

Le modèle le plus simple : une région active, une région de secours.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE ACTIVE-PASSIVE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         Global Load Balancer                                │
│                         (DNS Failover)                                      │
│                                │                                            │
│                    100% trafic │                                            │
│                                ▼                                            │
│  ┌─────────────────────────────────────────┐                                │
│  │              RÉGION PRIMAIRE            │                                │
│  │              (EU-WEST-1)                │                                │
│  │                                         │                                │
│  │  ┌─────────────────────────────────┐    │                                │
│  │  │        Galera Cluster           │    │                                │
│  │  │  ┌─────┐  ┌─────┐  ┌─────┐      │    │                                │
│  │  │  │Node1│  │Node2│  │Node3│      │    │                                │
│  │  │  │ RW  │  │ RW  │  │ RW  │      │    │                                │
│  │  │  └──┬──┘  └──┬──┘  └──┬──┘      │    │                                │
│  │  │     └────────┼────────┘         │    │                                │
│  │  │              │ Sync             │    │                                │
│  │  └──────────────┼──────────────────┘    │                                │
│  │                 │                       │                                │
│  └─────────────────┼───────────────────────┘                                │
│                    │                                                        │
│                    │ Async Replication (GTID)                               │
│                    │ Lag: 1-60 seconds                                      │
│                    ▼                                                        │
│  ┌─────────────────────────────────────────┐                                │
│  │              RÉGION DR                  │                                │
│  │              (US-EAST-1)                │                                │
│  │                                         │                                │
│  │  ┌─────────────────────────────────┐    │                                │
│  │  │        Galera Cluster           │    │                                │
│  │  │  ┌─────┐  ┌─────┐  ┌─────┐      │    │                                │
│  │  │  │Node1│  │Node2│  │Node3│      │    │                                │
│  │  │  │ RO  │  │ RO  │  │ RO  │      │    │   ← Standby (read-only)        │
│  │  │  └─────┘  └─────┘  └─────┘      │    │                                │
│  │  └─────────────────────────────────┘    │                                │
│  │                                         │                                │
│  └─────────────────────────────────────────┘                                │
│                                                                             │
│  Caractéristiques :                                                         │
│  • RPO : 1-60 secondes (lag de réplication)                                 │
│  • RTO : 5-30 minutes (failover manuel ou semi-auto)                        │
│  • Coût : Infrastructure DR sous-utilisée                                   │
│  • Complexité : Faible                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. Active-Active (Multi-Primary)

Écritures possibles dans plusieurs régions, avec résolution de conflits.

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE ACTIVE-ACTIVE                             │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│                         Global Load Balancer                              │
│                      (Geo-routing par proximité)                          │
│                                │                                          │
│              ┌─────────────────┴─────────────────┐                        │
│              │                                   │                        │
│              ▼                                   ▼                        │
│  ┌───────────────────────┐         ┌───────────────────────┐              │
│  │     RÉGION EUROPE     │         │     RÉGION US         │              │
│  │     (EU-WEST-1)       │         │     (US-EAST-1)       │              │
│  │                       │         │                       │              │
│  │  ┌─────────────────┐  │         │  ┌─────────────────┐  │              │
│  │  │  Galera (3)     │  │◄───────►│  │  Galera (3)     │  │              │
│  │  │  Primary EU     │  │  Async  │  │  Primary US     │  │              │
│  │  │  RW local       │  │  Bidir  │  │  RW local       │  │              │
│  │  └─────────────────┘  │         │  └─────────────────┘  │              │
│  │         │             │         │         │             │              │
│  │         ▼             │         │         ▼             │              │
│  │  ┌─────────────────┐  │         │  ┌─────────────────┐  │              │
│  │  │   MaxScale      │  │         │  │   MaxScale      │  │              │
│  │  │   (Router)      │  │         │  │   (Router)      │  │              │
│  │  └─────────────────┘  │         │  └─────────────────┘  │              │
│  └───────────────────────┘         └───────────────────────┘              │
│                                                                           │
│  Défis :                                                                  │
│  • Conflits d'écriture sur les mêmes données                              │
│  • Eventual consistency entre régions                                     │
│  • Complexité de résolution de conflits                                   │
│                                                                           │
│  Solutions :                                                              │
│  • Partitionnement par région (users EU → DB EU)                          │
│  • Timestamps/versions pour résolution (last-write-wins)                  │
│  • Conflit accepté sur données non-critiques                              │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 3. Follow-the-Sun (Rotation de primaire)

Le primaire suit le fuseau horaire des heures de bureau.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE FOLLOW-THE-SUN                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Heure (UTC)    │  0-8        │  8-16       │  16-24      │                │
│  ════════════   │  ═══════    │  ═══════    │  ═══════    │                │
│  Primary        │  APAC       │  EUROPE     │  AMERICAS   │                │
│                 │  (Tokyo)    │  (Frankfurt)│  (Virginia) │                │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │     ┌───────────┐       ┌───────────┐       ┌───────────┐           │   │
│  │     │   APAC    │       │  EUROPE   │       │ AMERICAS  │           │   │
│  │     │  Tokyo    │◄─────►│ Frankfurt │◄─────►│ Virginia  │           │   │
│  │     │           │ Async │           │ Async │           │           │   │
│  │     └───────────┘       └───────────┘       └───────────┘           │   │
│  │          │                   │                   │                  │   │
│  │          │                   │                   │                  │   │
│  │   UTC 0-8: PRIMARY    UTC 8-16: PRIMARY   UTC 16-24: PRIMARY        │   │
│  │   Autres: REPLICA     Autres: REPLICA     Autres: REPLICA           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Avantages :                                                               │
│  • Maintenance pendant les heures creuses de chaque région                 │
│  • Charge de backup distribuée                                             │
│  • Équipes ops par région avec responsabilité claire                       │
│                                                                            │
│  Inconvénients :                                                           │
│  • Switchover planifié complexe                                            │
│  • Fenêtre de transition délicate                                          │
│  • Automatisation avancée requise                                          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4. Architecture Hub-and-Spoke

Un hub central avec des spokes régionaux pour les lectures.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE HUB-AND-SPOKE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              ┌───────────────┐                              │
│                              │     HUB       │                              │
│                              │  (US-EAST)    │                              │
│                              │               │                              │
│                              │  ┌─────────┐  │                              │
│                              │  │ Primary │  │                              │
│                              │  │ Galera  │  │                              │
│                              │  │  (RW)   │  │                              │
│                              │  └────┬────┘  │                              │
│                              │       │       │                              │
│                              └───────┼───────┘                              │
│                                      │                                      │
│              ┌───────────────────────┼───────────────────────┐              │
│              │                       │                       │              │
│              ▼                       ▼                       ▼              │
│  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐      │
│  │      SPOKE        │   │      SPOKE        │   │      SPOKE        │      │
│  │     EUROPE        │   │      APAC         │   │    SOUTH AM       │      │
│  │                   │   │                   │   │                   │      │
│  │  ┌─────────────┐  │   │  ┌─────────────┐  │   │  ┌─────────────┐  │      │
│  │  │  Replica    │  │   │  │  Replica    │  │   │  │  Replica    │  │      │
│  │  │   (RO)      │  │   │  │   (RO)      │  │   │  │   (RO)      │  │      │
│  │  └─────────────┘  │   │  └─────────────┘  │   │  └─────────────┘  │      │
│  │                   │   │                   │   │                   │      │
│  │  Lectures locales │   │  Lectures locales │   │  Lectures locales │      │
│  │  Écritures → Hub  │   │  Écritures → Hub  │   │  Écritures → Hub  │      │
│  └───────────────────┘   └───────────────────┘   └───────────────────┘      │
│                                                                             │
│  Use case idéal :                                                           │
│  • Application à forte dominance lectures (80%+ reads)                      │
│  • Latence lecture critique, latence écriture acceptable                    │
│  • Exemple : Catalogue produits, CMS, documentation                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Configuration de la réplication géo-distribuée

### Réplication asynchrone avec GTID

```ini
# ═══════════════════════════════════════════════════════════════════════════
# CONFIGURATION PRIMARY (EU-WEST)
# /etc/mysql/mariadb.conf.d/geo-primary.cnf
# ═══════════════════════════════════════════════════════════════════════════

[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL

# GTID pour failover simplifié
gtid_domain_id = 1
gtid_strict_mode = ON
log_slave_updates = ON

# Durabilité
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

# Rétention binlog (doit couvrir les pannes réseau prolongées)
expire_logs_days = 7
max_binlog_size = 1G

# Semi-sync pour réplication locale (intra-région)
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_master_timeout = 1000  # 1 seconde

# Compression pour WAN
slave_compressed_protocol = ON
```

```ini
# ═══════════════════════════════════════════════════════════════════════════
# CONFIGURATION REPLICA (US-EAST - DR)
# /etc/mysql/mariadb.conf.d/geo-replica.cnf
# ═══════════════════════════════════════════════════════════════════════════

[mysqld]
server_id = 100
gtid_domain_id = 2

# Mode lecture seule par défaut
read_only = ON
super_read_only = ON

# Recevoir les mises à jour pour cascade éventuelle
log_slave_updates = ON
log_bin = mysql-bin

# Tolérance aux problèmes réseau WAN
slave_net_timeout = 120
slave_reconnect_timeout = 60

# Parallélisme de réplication
slave_parallel_threads = 8
slave_parallel_mode = optimistic

# Compression
slave_compressed_protocol = ON

# Skip des erreurs temporaires (avec précaution)
# slave_skip_errors = 1062,1032
```

### Établir la réplication

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SUR LE REPLICA (US-EAST)
-- ═══════════════════════════════════════════════════════════════════════════

-- Configuration de la réplication avec GTID
CHANGE MASTER TO
    MASTER_HOST = 'primary-eu.example.com',
    MASTER_PORT = 3306,
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'secure_replication_password',
    MASTER_USE_GTID = slave_pos,
    MASTER_SSL = 1,
    MASTER_SSL_CA = '/etc/mysql/ssl/ca-cert.pem',
    MASTER_SSL_CERT = '/etc/mysql/ssl/client-cert.pem',
    MASTER_SSL_KEY = '/etc/mysql/ssl/client-key.pem',
    MASTER_SSL_VERIFY_SERVER_CERT = 1,
    MASTER_CONNECT_RETRY = 60,
    MASTER_RETRY_COUNT = 86400;  -- Retry pendant 24h

-- Démarrer la réplication
START SLAVE;

-- Vérifier le statut
SHOW SLAVE STATUS\G

-- Points clés à vérifier :
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: < 60 (acceptable pour DR)
-- Gtid_Slave_Pos: doit avancer
```

### Réplication bidirectionnelle (Active-Active)

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- CONFIGURATION ACTIVE-ACTIVE (ATTENTION : CONFLITS POSSIBLES)
-- ═══════════════════════════════════════════════════════════════════════════

-- Sur EU (server_id = 1, gtid_domain_id = 1)
CHANGE MASTER TO
    MASTER_HOST = 'primary-us.example.com',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'secure_password',
    MASTER_USE_GTID = slave_pos;

-- Sur US (server_id = 100, gtid_domain_id = 2)  
CHANGE MASTER TO
    MASTER_HOST = 'primary-eu.example.com',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'secure_password',
    MASTER_USE_GTID = slave_pos;

-- IMPORTANT : Chaque région a son propre gtid_domain_id
-- Cela permet d'identifier l'origine des transactions
-- et d'éviter les boucles de réplication
```

⚠️ **Attention** : La réplication Active-Active nécessite une gestion applicative des conflits. Voir la section "Gestion des conflits" ci-dessous.

---

## Routage avec MaxScale

### Configuration MaxScale pour géo-distribution

```ini
# /etc/maxscale/maxscale.cnf
# Configuration MaxScale pour routage géographique

[maxscale]
threads = auto
log_info = true

# ═══════════════════════════════════════════════════════════════════════════
# SERVEURS - Toutes les régions
# ═══════════════════════════════════════════════════════════════════════════

[eu-primary-1]
type = server
address = eu-node1.example.com
port = 3306
protocol = MariaDBBackend
ssl = required
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

[eu-primary-2]
type = server
address = eu-node2.example.com
port = 3306
protocol = MariaDBBackend
ssl = required
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

[eu-primary-3]
type = server
address = eu-node3.example.com
port = 3306
protocol = MariaDBBackend
ssl = required
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

[us-dr-1]
type = server
address = us-node1.example.com
port = 3306
protocol = MariaDBBackend
ssl = required
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

[us-dr-2]
type = server
address = us-node2.example.com
port = 3306
protocol = MariaDBBackend
ssl = required
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

[us-dr-3]
type = server
address = us-node3.example.com
port = 3306
protocol = MariaDBBackend
ssl = required
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

# ═══════════════════════════════════════════════════════════════════════════
# MONITORS
# ═══════════════════════════════════════════════════════════════════════════

[eu-monitor]
type = monitor
module = galeramon
servers = eu-primary-1, eu-primary-2, eu-primary-3
user = maxscale_monitor
password = secure_monitor_password
monitor_interval = 2000ms
use_priority = true
disable_master_failback = false

[us-monitor]
type = monitor
module = galeramon
servers = us-dr-1, us-dr-2, us-dr-3
user = maxscale_monitor
password = secure_monitor_password
monitor_interval = 2000ms

# ═══════════════════════════════════════════════════════════════════════════
# SERVICES ET ROUTAGE
# ═══════════════════════════════════════════════════════════════════════════

# Service principal EU (Read/Write)
[eu-rw-service]
type = service
router = readwritesplit
servers = eu-primary-1, eu-primary-2, eu-primary-3
user = maxscale_user
password = secure_service_password
master_accept_reads = true
max_slave_replication_lag = 10s
transaction_replay = true
transaction_replay_max_size = 10Mi

# Service lecture locale (inclut DR pour failover reads)
[global-read-service]
type = service
router = readconnroute
router_options = slave
servers = eu-primary-1, eu-primary-2, eu-primary-3, us-dr-1, us-dr-2, us-dr-3
user = maxscale_user
password = secure_service_password

# 🆕 MaxScale 25.01 : Diff Router pour comparaison
[diff-service]
type = service
router = diff
servers = eu-primary-1, us-dr-1
user = maxscale_user
password = secure_service_password

# ═══════════════════════════════════════════════════════════════════════════
# LISTENERS
# ═══════════════════════════════════════════════════════════════════════════

[eu-rw-listener]
type = listener
service = eu-rw-service
protocol = MariaDBClient
port = 3306
ssl = required
ssl_cert = /etc/maxscale/ssl/server.pem
ssl_key = /etc/maxscale/ssl/server.key
ssl_ca_cert = /etc/maxscale/ssl/ca.pem

[global-read-listener]
type = listener
service = global-read-service
protocol = MariaDBClient
port = 3307
ssl = required
ssl_cert = /etc/maxscale/ssl/server.pem
ssl_key = /etc/maxscale/ssl/server.key
```

### Failover automatique avec MaxScale

```bash
#!/bin/bash
# geo-failover.sh
# Script de failover géographique

PRIMARY_REGION="eu"
DR_REGION="us"
MAXSCALE_HOST="maxscale.example.com"
SLACK_WEBHOOK="https://hooks.slack.com/services/..."

# Fonction de notification
notify() {
    local message=$1
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"🔄 GEO-FAILOVER: $message\"}" \
        "$SLACK_WEBHOOK"
    echo "[$(date)] $message"
}

# Vérifier l'état du primary
check_primary() {
    maxctrl --hosts "$MAXSCALE_HOST" list servers | grep -E "eu-primary.*Master"
    return $?
}

# Promouvoir le DR en primary
promote_dr() {
    notify "Promoting $DR_REGION to primary..."
    
    # 1. Arrêter la réplication sur DR
    for node in us-dr-1 us-dr-2 us-dr-3; do
        mysql -h "$node" -e "STOP SLAVE; RESET SLAVE ALL;"
    done
    
    # 2. Désactiver read_only sur DR
    for node in us-dr-1 us-dr-2 us-dr-3; do
        mysql -h "$node" -e "SET GLOBAL read_only = OFF; SET GLOBAL super_read_only = OFF;"
    done
    
    # 3. Reconfigurer MaxScale
    maxctrl --hosts "$MAXSCALE_HOST" alter monitor eu-monitor auto_failover=false
    maxctrl --hosts "$MAXSCALE_HOST" alter service eu-rw-service servers="us-dr-1,us-dr-2,us-dr-3"
    
    # 4. Mettre à jour le DNS
    # aws route53 change-resource-record-sets ... (ou équivalent)
    
    notify "Failover completed. DR ($DR_REGION) is now primary."
}

# Logique principale
if ! check_primary; then
    notify "Primary region $PRIMARY_REGION is DOWN!"
    
    # Attendre confirmation (éviter les faux positifs)
    sleep 30
    
    if ! check_primary; then
        notify "Confirmed: Primary is still down. Initiating failover..."
        promote_dr
    else
        notify "Primary recovered. No failover needed."
    fi
else
    echo "Primary region is healthy."
fi
```

---

## Gestion des conflits (Active-Active)

### Stratégies de résolution

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- STRATÉGIES DE RÉSOLUTION DE CONFLITS
-- ═══════════════════════════════════════════════════════════════════════════

-- 1. PARTITIONNEMENT PAR RÉGION (Évite les conflits)
-- ══════════════════════════════════════════════════

-- Les utilisateurs sont assignés à une région "home"
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    home_region ENUM('EU', 'US', 'APAC') NOT NULL,
    -- ... autres colonnes
    
    -- Contrainte : seule la région home peut modifier
    INDEX idx_region (home_region)
) ENGINE=InnoDB;

-- Application : Routing des écritures vers la région home
-- Si user.home_region = 'EU' → écriture vers EU cluster
-- Lectures possibles depuis n'importe quelle région

-- 2. LAST-WRITE-WINS (Timestamp-based)
-- ═════════════════════════════════════

CREATE TABLE documents (
    id BIGINT PRIMARY KEY,
    content TEXT,
    version INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMP(6) NOT NULL,  -- Microsecond precision
    updated_by_region CHAR(2) NOT NULL,
    
    -- Version vector pour détection de conflit
    version_vector JSON,
    
    INDEX idx_updated (updated_at)
) ENGINE=InnoDB;

-- Trigger de résolution (simplifié - LWW)
DELIMITER //
CREATE TRIGGER trg_document_conflict
BEFORE UPDATE ON documents
FOR EACH ROW
BEGIN
    -- Si la version entrante est plus ancienne, ignorer
    IF NEW.updated_at < OLD.updated_at THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Conflict: incoming update is older';
    END IF;
    
    -- Incrémenter la version
    SET NEW.version = OLD.version + 1;
END //
DELIMITER ;

-- 3. APPLICATION-LEVEL CONFLICT RESOLUTION
-- ════════════════════════════════════════

-- Table de conflits pour résolution manuelle
CREATE TABLE replication_conflicts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(100) NOT NULL,
    row_id BIGINT NOT NULL,
    conflict_type ENUM('UPDATE', 'DELETE', 'INSERT') NOT NULL,
    local_data JSON,
    remote_data JSON,
    local_timestamp TIMESTAMP(6),
    remote_timestamp TIMESTAMP(6),
    local_region CHAR(2),
    remote_region CHAR(2),
    resolution_status ENUM('PENDING', 'AUTO_RESOLVED', 'MANUAL_RESOLVED') DEFAULT 'PENDING',
    resolved_at TIMESTAMP NULL,
    resolved_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_pending (resolution_status, created_at)
) ENGINE=InnoDB;
```

### CRDT (Conflict-free Replicated Data Types)

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- CRDT : COMPTEURS (Grow-only Counter)
-- ═══════════════════════════════════════════════════════════════════════════

-- Chaque région maintient son propre compteur
CREATE TABLE page_views_crdt (
    page_id BIGINT NOT NULL,
    region CHAR(2) NOT NULL,
    view_count BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (page_id, region)
) ENGINE=InnoDB;

-- Incrémenter (jamais de conflit car chaque région écrit sa ligne)
-- Sur EU :
UPDATE page_views_crdt SET view_count = view_count + 1 
WHERE page_id = 123 AND region = 'EU';

-- Sur US :
UPDATE page_views_crdt SET view_count = view_count + 1 
WHERE page_id = 123 AND region = 'US';

-- Lecture du total (fusion des compteurs)
SELECT page_id, SUM(view_count) AS total_views
FROM page_views_crdt
WHERE page_id = 123
GROUP BY page_id;

-- ═══════════════════════════════════════════════════════════════════════════
-- CRDT : SET (Add-only Set avec tombstones)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE user_tags_crdt (
    user_id BIGINT NOT NULL,
    tag VARCHAR(100) NOT NULL,
    added_at TIMESTAMP(6) NOT NULL,
    added_by_region CHAR(2) NOT NULL,
    deleted_at TIMESTAMP(6) NULL,
    deleted_by_region CHAR(2) NULL,
    
    PRIMARY KEY (user_id, tag, added_by_region)
) ENGINE=InnoDB;

-- Vue des tags actifs
CREATE VIEW v_user_tags AS
SELECT user_id, tag, MIN(added_at) AS first_added
FROM user_tags_crdt
WHERE deleted_at IS NULL
   OR deleted_at > added_at  -- Tombstone invalide si antérieur à l'ajout
GROUP BY user_id, tag;
```

---

## Souveraineté des données

### Architecture pour conformité RGPD

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE RGPD-COMPLIANT                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         Application Layer                                   │
│                              │                                              │
│              ┌───────────────┴───────────────┐                              │
│              │       Data Router             │                              │
│              │   (Route par nationalité)     │                              │
│              └───────────────┬───────────────┘                              │
│                              │                                              │
│         ┌────────────────────┼────────────────────┐                         │
│         │                    │                    │                         │
│         ▼                    ▼                    ▼                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │   EU DATA       │  │   US DATA       │  │  GLOBAL DATA    │              │
│  │   (Frankfurt)   │  │   (Virginia)    │  │  (Anonymisée)   │              │
│  │                 │  │                 │  │                 │              │
│  │  • Users EU     │  │  • Users US     │  │  • Analytics    │              │
│  │  • PII EU       │  │  • PII US       │  │  • Agrégats     │              │
│  │  • Orders EU    │  │  • Orders US    │  │  • Logs (anon)  │              │
│  │                 │  │                 │  │                 │              │
│  │  RGPD applies   │  │  CCPA applies   │  │  No PII         │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
│                                                                             │
│  Règles de routage :                                                        │
│  • user.country IN ('FR','DE','IT',...) → EU cluster                        │
│  • user.country IN ('US','CA') → US cluster                                 │
│  • Données anonymisées → Global cluster                                     │
│                                                                             │
│  Contraintes :                                                              │
│  • Pas de réplication PII cross-région                                      │
│  • Backup dans la même région                                               │
│  • Droit à l'effacement par région                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implémentation du routage par juridiction

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- TABLES DE ROUTAGE JURIDICTIONNEL
-- ═══════════════════════════════════════════════════════════════════════════

-- Mapping pays → région de données
CREATE TABLE data_jurisdiction (
    country_code CHAR(2) PRIMARY KEY,
    country_name VARCHAR(100) NOT NULL,
    data_region ENUM('EU', 'US', 'APAC', 'LATAM') NOT NULL,
    privacy_law VARCHAR(50),  -- GDPR, CCPA, LGPD, PDPA, etc.
    requires_local_storage BOOLEAN DEFAULT FALSE,
    notes TEXT
) ENGINE=InnoDB;

INSERT INTO data_jurisdiction VALUES
    ('FR', 'France', 'EU', 'GDPR', TRUE, NULL),
    ('DE', 'Germany', 'EU', 'GDPR', TRUE, NULL),
    ('IT', 'Italy', 'EU', 'GDPR', TRUE, NULL),
    ('ES', 'Spain', 'EU', 'GDPR', TRUE, NULL),
    ('US', 'United States', 'US', 'CCPA', FALSE, 'State laws vary'),
    ('CA', 'Canada', 'US', 'PIPEDA', FALSE, NULL),
    ('BR', 'Brazil', 'LATAM', 'LGPD', TRUE, NULL),
    ('JP', 'Japan', 'APAC', 'APPI', FALSE, NULL),
    ('SG', 'Singapore', 'APAC', 'PDPA', FALSE, NULL),
    ('AU', 'Australia', 'APAC', 'Privacy Act', FALSE, NULL);

-- Utilisateurs avec indication de la région de stockage
CREATE TABLE users_metadata (
    user_id BIGINT PRIMARY KEY,
    email_hash VARCHAR(64) NOT NULL,  -- Hash pour lookup cross-région
    country_code CHAR(2) NOT NULL,
    data_region ENUM('EU', 'US', 'APAC', 'LATAM') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_hash (email_hash),
    INDEX idx_region (data_region),
    
    FOREIGN KEY (country_code) REFERENCES data_jurisdiction(country_code)
) ENGINE=InnoDB;

-- Procédure de lookup cross-région
DELIMITER //

CREATE FUNCTION get_user_data_region(p_email VARCHAR(255))
RETURNS VARCHAR(10)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_region VARCHAR(10);
    
    SELECT data_region INTO v_region
    FROM users_metadata
    WHERE email_hash = SHA2(LOWER(p_email), 256);
    
    RETURN COALESCE(v_region, 'UNKNOWN');
END //

DELIMITER ;
```

---

## Monitoring et observabilité

### Métriques de réplication géo-distribuée

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- MONITORING DE LA RÉPLICATION GÉO-DISTRIBUÉE
-- ═══════════════════════════════════════════════════════════════════════════

-- Vue consolidée du statut de réplication
CREATE VIEW v_geo_replication_status AS
SELECT 
    @@server_id AS server_id,
    @@gtid_domain_id AS gtid_domain,
    @@gtid_current_pos AS current_gtid,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Slave_running') AS slave_running,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Seconds_Behind_Master') AS lag_seconds,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Slave_received_heartbeats') AS heartbeats,
    NOW() AS checked_at;

-- Historique du lag (pour graphiques)
CREATE TABLE replication_lag_history (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    source_region CHAR(10) NOT NULL,
    target_region CHAR(10) NOT NULL,
    lag_seconds INT,
    gtid_lag INT,
    network_latency_ms INT,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_regions_time (source_region, target_region, recorded_at)
) ENGINE=InnoDB;

-- Job de collecte (Event)
DELIMITER //

CREATE EVENT collect_replication_metrics
ON SCHEDULE EVERY 1 MINUTE
DO
BEGIN
    DECLARE v_lag INT;
    DECLARE v_gtid_lag INT;
    
    -- Récupérer le lag actuel
    SELECT Seconds_Behind_Master INTO v_lag
    FROM information_schema.SLAVE_STATUS
    LIMIT 1;
    
    -- Calculer le lag GTID (transactions en attente)
    -- Simplifié - en production utiliser gtid_subtract
    SET v_gtid_lag = 0;
    
    INSERT INTO replication_lag_history 
        (source_region, target_region, lag_seconds, gtid_lag)
    VALUES 
        ('EU', @@hostname, v_lag, v_gtid_lag);
END //

DELIMITER ;
```

### Dashboard Grafana pour géo-distribution

```yaml
# prometheus-geo-alerts.yaml
groups:
  - name: geo-replication
    rules:
      # Lag de réplication élevé
      - alert: GeoReplicationLagHigh
        expr: mysql_slave_status_seconds_behind_master > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Geo-replication lag > 5 minutes"
          description: "Replica {{ $labels.instance }} is {{ $value }}s behind master"

      # Lag critique (risque de perte de données en cas de failover)
      - alert: GeoReplicationLagCritical
        expr: mysql_slave_status_seconds_behind_master > 3600
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Geo-replication lag > 1 hour - DR at risk"
          
      # Réplication arrêtée
      - alert: GeoReplicationStopped
        expr: mysql_slave_status_slave_io_running == 0 OR mysql_slave_status_slave_sql_running == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Geo-replication stopped on {{ $labels.instance }}"
          
      # Latence réseau inter-région
      - alert: GeoNetworkLatencyHigh
        expr: probe_duration_seconds{job="blackbox-geo"} > 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Network latency between regions > 200ms"
          
      # Divergence GTID (split-brain potentiel)
      - alert: GeoGtidDivergence
        expr: |
          abs(
            mysql_global_status_gtid_executed{region="eu"} - 
            mysql_global_status_gtid_executed{region="us"}
          ) > 10000
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "GTID divergence detected between regions"
```

---

## Étude de cas : E-commerce global

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ÉTUDE DE CAS : E-COMMERCE GLOBAL                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Contexte :                                                                 │
│  • 5M utilisateurs répartis : 40% EU, 35% US, 25% APAC                      │
│  • 100K commandes/jour                                                      │
│  • SLA : 99.95% disponibilité, latence P99 < 200ms                          │
│  • Conformité : RGPD (EU), CCPA (US), PDPA (APAC)                           │
│                                                                             │
│  Architecture retenue :                                                     │
│                                                                             │
│                    ┌─────────────────────────────────────┐                  │
│                    │       Global Load Balancer          │                  │
│                    │     (Cloudflare / AWS Global)       │                  │
│                    └──────────────────┬──────────────────┘                  │
│                                       │                                     │
│           ┌───────────────────────────┼───────────────────────────┐         │
│           │                           │                           │         │
│           ▼                           ▼                           ▼         │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │   EU (Primary)  │        │   US (Primary)  │        │  APAC (Primary) │  │
│  │   Frankfurt     │        │   Virginia      │        │   Singapore     │  │
│  │                 │        │                 │        │                 │  │
│  │ ┌─────────────┐ │        │ ┌─────────────┐ │        │ ┌─────────────┐ │  │
│  │ │Galera (3)   │ │◄──────►│ │Galera (3)   │ │◄──────►│ │Galera (3)   │ │  │
│  │ │EU Users     │ │ Async  │ │US Users     │ │ Async  │ │APAC Users   │ │  │
│  │ │EU Orders    │ │ Bidir  │ │US Orders    │ │ Bidir  │ │APAC Orders  │ │  │
│  │ └─────────────┘ │        │ └─────────────┘ │        │ └─────────────┘ │  │
│  │                 │        │                 │        │                 │  │
│  │ ┌─────────────┐ │        │ ┌─────────────┐ │        │ ┌─────────────┐ │  │
│  │ │MaxScale     │ │        │ │MaxScale     │ │        │ │MaxScale     │ │  │
│  │ │(RW Split)   │ │        │ │(RW Split)   │ │        │ │(RW Split)   │ │  │
│  │ └─────────────┘ │        │ └─────────────┘ │        │ └─────────────┘ │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│           │                           │                           │         │
│           └───────────────────────────┼───────────────────────────┘         │
│                                       │                                     │
│                                       ▼                                     │
│                    ┌─────────────────────────────────────┐                  │
│                    │     Global Catalog (Read-only)      │                  │
│                    │     Products, Categories, Config    │                  │
│                    │     (Répliqué dans toutes régions)  │                  │
│                    └─────────────────────────────────────┘                  │
│                                                                             │
│  Décisions clés :                                                           │
│  ═══════════════                                                            │
│                                                                             │
│  1. Partitionnement par région home :                                       │
│     • Utilisateur créé en EU → données dans EU cluster                      │
│     • Évite les conflits d'écriture (pas d'Active-Active sur mêmes users)   │
│                                                                             │
│  2. Catalogue global répliqué :                                             │
│     • Products, Categories = données peu changées                           │
│     • Réplication async vers toutes régions                                 │
│     • Lectures locales, écritures vers master (US)                          │
│                                                                             │
│  3. Async bidirectionnelle pour référentiel commun :                        │
│     • Inventory agrégé visible globalement                                  │
│     • Résolution LWW acceptable (stock = approximation)                     │
│                                                                             │
│  Métriques atteintes :                                                      │
│  • Latence P99 : 85ms (EU), 90ms (US), 110ms (APAC)                         │
│  • Disponibilité : 99.97%                                                   │
│  • RPO cross-région : < 60 secondes                                         │
│  • RTO failover régional : < 5 minutes                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Points clés à retenir

- **La latence réseau est la contrainte principale** : Galera synchrone viable jusqu'à ~30ms RTT, asynchrone au-delà
- **Active-Passive** est le modèle le plus simple et recommandé pour débuter — DR avec RPO de quelques secondes
- **Active-Active** nécessite une gestion des conflits — privilégier le partitionnement par région pour les éviter
- **GTID est essentiel** pour la géo-distribution : simplifie le failover et le tracking cross-région
- **MaxScale** permet un routage intelligent et un failover automatique entre régions
- **La souveraineté des données** (RGPD) impose souvent un partitionnement géographique strict des PII
- **Le monitoring du lag** est critique — un lag élevé augmente le RPO en cas de failover
- **Les CRDT** sont une solution élégante pour les données qui doivent être modifiées dans plusieurs régions sans conflit

---

## 🔗 Ressources et références

- [📖 MariaDB Replication](https://mariadb.com/kb/en/replication/)
- [📖 MariaDB GTID](https://mariadb.com/kb/en/gtid/)
- [📖 Galera Cluster Geographic Distribution](https://galeracluster.com/library/documentation/wan-replication.html)
- [📖 MaxScale Configuration Guide](https://mariadb.com/kb/en/maxscale/)
- [📖 CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)
- [📖 GDPR Data Residency Requirements](https://gdpr.eu/data-processing/)

---


⏭️ [Hybrid cloud et multi-cloud](/20-cas-usage-architectures/06-hybrid-multi-cloud.md)

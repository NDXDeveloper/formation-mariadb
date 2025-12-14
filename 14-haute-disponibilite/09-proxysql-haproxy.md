🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.9 Alternatives : ProxySQL et HAProxy

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Section 14.4 (MaxScale), connaissances réseau (TCP/IP, load balancing)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les architectures ProxySQL et HAProxy
- **Comparer** objectivement MaxScale, ProxySQL et HAProxy
- **Configurer** ProxySQL pour MariaDB et Galera Cluster
- **Configurer** HAProxy pour load balancing MariaDB
- **Choisir** la solution optimale selon vos contraintes
- **Combiner** plusieurs proxies pour architectures hybrides
- **Optimiser** performance et résilience de chaque solution
- **Migrer** entre solutions si nécessaire

---

## Introduction

MaxScale est la solution native MariaDB pour le proxy SQL, mais ce n'est pas la seule option. **ProxySQL** et **HAProxy** sont deux alternatives matures et largement déployées en production, chacune avec ses forces et faiblesses.

**Contexte décisionnel** :
```
┌────────────────────────────────────────────────────┐
│  Critères de Choix                                 │
├────────────────────────────────────────────────────┤
│                                                    │
│  Budget                                            │
│  ├─ MaxScale : BSL (gratuit < 3 serveurs)          │
│  │             ou Commercial                       │
│  ├─ ProxySQL : GPL (gratuit)                       │
│  └─ HAProxy : GPL (gratuit)                        │
│                                                    │
│  Complexité                                        │
│  ├─ MaxScale : Moyenne (intégré MariaDB)           │
│  ├─ ProxySQL : Élevée (courbe apprentissage)       │
│  └─ HAProxy : Faible (simple config)               │
│                                                    │
│  Fonctionnalités                                   │
│  ├─ MaxScale : SQL-aware (L7), Galera natif        │
│  ├─ ProxySQL : SQL-aware (L7), Query caching       │
│  └─ HAProxy : TCP-only (L4), Pure LB               │
│                                                    │
│  Performance                                       │
│  ├─ MaxScale : Overhead 5-10%                      │
│  ├─ ProxySQL : Overhead 3-5%                       │
│  └─ HAProxy : Overhead 1-2%                        │
└────────────────────────────────────────────────────┘
```

> 💡 **Principe** : "Il n'y a pas de 'meilleure' solution absolue, seulement la meilleure solution pour VOTRE contexte."

---

## 1. ProxySQL : Le Proxy Haute Performance

### 1.1 Architecture et Concepts

**ProxySQL** est un proxy SQL Layer 7 (application-aware) conçu pour MySQL/MariaDB avec un focus sur la **performance** et le **query caching**.

```
Architecture ProxySQL :
┌─────────────────────────────────────────────────┐
│               Applications                      │
└────────────────┬────────────────────────────────┘
                 │
                 │ Port 6033 (MySQL Protocol)
                 │
        ┌────────▼────────┐
        │   ProxySQL      │
        │                 │
        │  ┌───────────┐  │
        │  │ Query     │  │ ← Query Rules Engine
        │  │ Processor │  │
        │  └─────┬─────┘  │
        │        │        │
        │  ┌─────▼─────┐  │
        │  │ Query     │  │ ← Query Cache (in-memory)
        │  │ Cache     │  │
        │  └─────┬─────┘  │
        │        │        │
        │  ┌─────▼─────┐  │
        │  │ Connection│  │ ← Connection Pool
        │  │ Pool      │  │
        │  └─────┬─────┘  │
        │        │        │
        │  ┌─────▼─────┐  │
        │  │ Health    │  │ ← Server Monitoring
        │  │ Check     │  │
        │  └───────────┘  │
        └─────────────────┘
                 │
     ┌───────────┼───────────┐
     │           │           │
┌────▼────┐ ┌────▼────┐ ┌────▼────┐
│ Master  │ │ Slave 1 │ │ Slave 2 │
└─────────┘ └─────────┘ └─────────┘
```

**Composants clés** :

1. **Query Rules** : Routage intelligent basé sur patterns SQL
2. **Query Cache** : Cache in-memory des résultats (clé = requête)
3. **Connection Pooling** : Réutilisation connexions backend
4. **MySQL Hostgroups** : Groupes logiques de serveurs (master, slaves)
5. **MySQL Users** : Mapping utilisateurs frontend ↔ backend

### 1.2 Installation et Configuration

#### **Installation Ubuntu/Debian**

```bash
# Ajouter repository ProxySQL
wget https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/repo_pub_key
apt-key add repo_pub_key

cat > /etc/apt/sources.list.d/proxysql.list << 'EOF'
deb https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/$(lsb_release -sc)/ ./
EOF

# Installation
apt-get update
apt-get install -y proxysql

# Démarrage
systemctl start proxysql
systemctl enable proxysql

# Port admin : 6032 (MySQL protocol)
# Port applicatif : 6033 (MySQL protocol)
```

#### **Configuration de Base**

```bash
# Connexion admin ProxySQL
mysql -h 127.0.0.1 -P 6032 -u admin -padmin
```

```sql
-- ============================================
-- CONFIGURATION PROXYSQL POUR MARIADB GALERA
-- ============================================

-- 1. Configurer les serveurs backend
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES
  (0, '10.0.1.10', 3306, 1000, 1000),  -- Node1 (writer)
  (0, '10.0.1.11', 3306, 1000, 1000),  -- Node2 (writer)
  (0, '10.0.1.12', 3306, 1000, 1000),  -- Node3 (writer)
  (1, '10.0.1.10', 3306, 1000, 1000),  -- Node1 (reader)
  (1, '10.0.1.11', 3306, 1000, 1000),  -- Node2 (reader)
  (1, '10.0.1.12', 3306, 1000, 1000);  -- Node3 (reader)

-- Hostgroup 0 = Writers (Galera : tous les nœuds acceptent writes)
-- Hostgroup 1 = Readers

-- Appliquer changements
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

-- 2. Configurer utilisateurs
-- Créer utilisateur monitor sur MariaDB d'abord :
-- CREATE USER 'monitor'@'%' IDENTIFIED BY 'MonitorPassword';
-- GRANT REPLICATION CLIENT ON *.* TO 'monitor'@'%';

INSERT INTO mysql_users (username, password, default_hostgroup, transaction_persistent)
VALUES
  ('app_user', 'AppPassword', 0, 1),
  ('app_readonly', 'ReadOnlyPassword', 1, 1);

-- transaction_persistent = 1 : toutes requêtes d'une transaction vers même backend
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;

-- 3. Configurer monitoring
UPDATE global_variables 
SET variable_value='monitor' 
WHERE variable_name='mysql-monitor_username';

UPDATE global_variables 
SET variable_value='MonitorPassword' 
WHERE variable_name='mysql-monitor_password';

UPDATE global_variables 
SET variable_value='2000' 
WHERE variable_name='mysql-monitor_connect_interval';
-- Check connect toutes les 2 secondes

UPDATE global_variables 
SET variable_value='10000' 
WHERE variable_name='mysql-monitor_ping_interval';
-- Ping toutes les 10 secondes

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

-- 4. Query Rules : Read/Write Split
-- Règle 1 : Toutes les écritures vers hostgroup 0 (writers)
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES 
  (1, 1, '^SELECT.*FOR UPDATE', 0, 1),
  (2, 1, '^SELECT.*LOCK IN SHARE MODE', 0, 1),
  (3, 1, '^INSERT', 0, 1),
  (4, 1, '^UPDATE', 0, 1),
  (5, 1, '^DELETE', 0, 1),
  (6, 1, '^REPLACE', 0, 1);

-- Règle 2 : Lectures simples vers hostgroup 1 (readers)
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES 
  (10, 1, '^SELECT', 1, 1);

-- Règle 3 : Pattern spécifique vers cache
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, cache_ttl, apply)
VALUES 
  (20, 1, '^SELECT.*FROM products WHERE category', 60000, 1);
-- Cache 60 secondes

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;

-- 5. Connection Pooling
UPDATE global_variables 
SET variable_value='4' 
WHERE variable_name='mysql-max_connections';
-- Max 4 connexions par utilisateur depuis ProxySQL vers backend

UPDATE global_variables 
SET variable_value='300' 
WHERE variable_name='mysql-free_connections_pct';
-- Maintenir 30% connexions libres

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

#### **Configuration Avancée : Galera-Specific**

```sql
-- Monitoring Galera spécifique
-- Script de health check Galera-aware

-- Créer scheduler pour check Galera status
INSERT INTO scheduler (id, active, interval_ms, filename, arg1)
VALUES 
  (1, 1, 5000, '/usr/share/proxysql/tools/proxysql_galera_checker.sh', 0);
-- Check toutes les 5 secondes

LOAD SCHEDULER TO RUNTIME;
SAVE SCHEDULER TO DISK;
```

```bash
#!/bin/bash
# /usr/share/proxysql/tools/proxysql_galera_checker.sh
# Health check Galera pour ProxySQL

HOSTGROUP_WRITER_ID=0
HOSTGROUP_READER_ID=1
PROXYSQL_USER="admin"
PROXYSQL_PASS="admin"

# Vérifier chaque serveur
for server in 10.0.1.10 10.0.1.11 10.0.1.12; do
    # Check Galera status
    CLUSTER_STATUS=$(mysql -h $server -u monitor -pMonitorPassword -N -e \
        "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME='wsrep_cluster_status'")
    
    LOCAL_STATE=$(mysql -h $server -u monitor -pMonitorPassword -N -e \
        "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME='wsrep_local_state'")
    
    # wsrep_local_state = 4 → Synced
    # wsrep_cluster_status = 'Primary' → In quorum
    
    if [ "$CLUSTER_STATUS" = "Primary" ] && [ "$LOCAL_STATE" = "4" ]; then
        # Serveur sain, activer dans ProxySQL
        mysql -h 127.0.0.1 -P 6032 -u $PROXYSQL_USER -p$PROXYSQL_PASS -e "
            UPDATE mysql_servers 
            SET status='ONLINE' 
            WHERE hostname='$server';
            LOAD MYSQL SERVERS TO RUNTIME;
        "
    else
        # Serveur problème, désactiver
        mysql -h 127.0.0.1 -P 6032 -u $PROXYSQL_USER -p$PROXYSQL_PASS -e "
            UPDATE mysql_servers 
            SET status='OFFLINE_SOFT' 
            WHERE hostname='$server';
            LOAD MYSQL SERVERS TO RUNTIME;
        "
    fi
done
```

### 1.3 Query Caching (Killer Feature ProxySQL)

```sql
-- Configuration query cache
UPDATE global_variables 
SET variable_value='true' 
WHERE variable_name='mysql-query_cache_size_MB';
-- Activer cache

UPDATE global_variables 
SET variable_value='256' 
WHERE variable_name='mysql-query_cache_size_MB';
-- 256 MB cache

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

-- Règles de cache spécifiques
-- Cache agressif pour données quasi-statiques
INSERT INTO mysql_query_rules (
    rule_id, 
    active, 
    match_pattern, 
    cache_ttl, 
    apply
)
VALUES 
  -- Produits : cache 5 minutes
  (100, 1, '^SELECT.*FROM products', 300000, 1),
  
  -- Catégories : cache 10 minutes
  (101, 1, '^SELECT.*FROM categories', 600000, 1),
  
  -- Configuration : cache 1 heure
  (102, 1, '^SELECT.*FROM config', 3600000, 1),
  
  -- Stats dashboard : cache 30 secondes
  (103, 1, '^SELECT COUNT.*FROM orders.*GROUP BY DATE', 30000, 1);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;

-- Visualiser efficacité cache
SELECT 
    cache_hit_ratio,
    cached_queries,
    hits,
    misses
FROM stats_mysql_query_cache
ORDER BY cache_hit_ratio DESC;
```

**Impact performance** :
```
Benchmark avec cache :
┌──────────────────────────────────────────────────┐
│  Query                 │ Sans Cache │ Avec Cache │
├────────────────────────┼────────────┼────────────┤
│  SELECT * FROM         │   45ms     │    0.3ms   │
│  products WHERE cat=1  │            │  (150x)    │
├────────────────────────┼────────────┼────────────┤
│  Dashboard COUNT(*)    │  230ms     │    1.2ms   │
│  GROUP BY date         │            │  (190x)    │
├────────────────────────┼────────────┼────────────┤
│  Categories lookup     │   12ms     │    0.2ms   │
│                        │            │   (60x)    │
└──────────────────────────────────────────────────┘

→ Cache hit ratio > 80% = Gains massifs
```

### 1.4 Monitoring ProxySQL

```sql
-- Statistiques connexions
SELECT * FROM stats_mysql_connection_pool;

-- Statistiques queries
SELECT * FROM stats_mysql_query_digest 
ORDER BY sum_time DESC 
LIMIT 10;

-- Statistiques commands
SELECT * FROM stats_mysql_commands_counters 
ORDER BY Total_cnt DESC;

-- Santé serveurs backend
SELECT 
    hostgroup_id,
    hostname,
    port,
    status,
    ConnUsed,
    ConnFree,
    ConnOK,
    ConnERR,
    Queries,
    Latency_us
FROM stats_mysql_connection_pool;
```

```bash
# Export métriques Prometheus
# Utiliser proxysql_exporter
docker run -d \
  --name proxysql_exporter \
  -p 42004:42004 \
  -e PROXYSQL_HOST=localhost \
  -e PROXYSQL_PORT=6032 \
  -e PROXYSQL_USER=stats \
  -e PROXYSQL_PASSWORD=stats \
  percona/proxysql_exporter:latest
```

---

## 2. HAProxy : Le Load Balancer Pure Performance

### 2.1 Architecture et Positionnement

**HAProxy** est un load balancer TCP/HTTP **Layer 4** (et Layer 7 HTTP). Pour MariaDB, il opère principalement en **Layer 4** (TCP pure), donc **sans intelligence SQL**.

```
Architecture HAProxy (Layer 4) :
┌─────────────────────────────────────────────────┐
│               Applications                      │
└────────────────┬────────────────────────────────┘
                 │
                 │ Port 3306 (MySQL Protocol)
                 │
        ┌────────▼────────┐
        │    HAProxy      │
        │                 │
        │  ┌───────────┐  │
        │  │ Frontend  │  │ ← Listen 0.0.0.0:3306
        │  │  (bind)   │  │
        │  └─────┬─────┘  │
        │        │        │
        │  ┌─────▼─────┐  │
        │  │  Backend  │  │ ← Server pool
        │  │  (pool)   │  │
        │  └─────┬─────┘  │
        │        │        │
        │  ┌─────▼─────┐  │
        │  │  Health   │  │ ← TCP check + MySQL check
        │  │  Checks   │  │
        │  └───────────┘  │
        └─────────────────┘
                 │
     ┌───────────┼───────────┐
     │           │           │
┌────▼────┐ ┌────▼────┐ ┌────▼────┐
│ Master  │ │ Slave 1 │ │ Slave 2 │
└─────────┘ └─────────┘ └─────────┘
```

**Différence fondamentale** :
```
┌────────────────────────────────────────────────────┐
│  ProxySQL/MaxScale (L7)    │  HAProxy (L4)         │
├────────────────────────────┼───────────────────────┤
│  Comprend SQL              │  Ne comprend pas SQL  │
│  Query routing intelligent │  Round-robin simple   │
│  Query cache possible      │  Pas de cache         │
│  Read/Write split          │  Requires 2 frontends │
│  Overhead : 3-10%          │  Overhead : 1-2%      │
│  Complexe à tuner          │  Simple à configurer  │
└────────────────────────────────────────────────────┘
```

### 2.2 Configuration HAProxy pour MariaDB

#### **Installation**

```bash
# Ubuntu/Debian
apt-get install -y haproxy

# Vérifier version
haproxy -v
# HAProxy version 2.8.5
```

#### **Configuration Master-Slave**

```bash
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Performance tuning
    maxconn 40000
    nbproc 4  # 4 processus (1 par CPU core)
    cpu-map auto:1/1-4 0-3

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 10s
    timeout client  28800s  # 8 heures (long-running queries)
    timeout server  28800s
    retries 3
    option redispatch
    maxconn 10000

# ================================================
# STATS UI (HTTP)
# ================================================
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 5s
    stats show-legends
    stats auth admin:AdminPassword

# ================================================
# FRONTEND : WRITES (Master)
# ================================================
frontend mysql_writes
    bind *:3306
    mode tcp
    default_backend mysql_master_backend

# Backend Master (writes)
backend mysql_master_backend
    mode tcp
    balance leastconn
    option tcp-check
    
    # Health check MySQL spécifique
    tcp-check connect
    tcp-check send-binary 00000001  # Packet length
    tcp-check send-binary 00        # Packet number
    tcp-check send-binary 0a        # Protocol version
    
    # Expect handshake response
    tcp-check expect binary 0a
    
    # Serveur master avec check read_only
    server master1 10.0.1.10:3306 check inter 2000 rise 2 fall 3 \
        on-marked-down shutdown-sessions

# ================================================
# FRONTEND : READS (Replicas)
# ================================================
frontend mysql_reads
    bind *:3307
    mode tcp
    default_backend mysql_replica_backend

# Backend Replicas (reads)
backend mysql_replica_backend
    mode tcp
    balance roundrobin
    option tcp-check
    
    tcp-check connect
    tcp-check send-binary 00000001
    tcp-check send-binary 00
    tcp-check send-binary 0a
    tcp-check expect binary 0a
    
    server replica1 10.0.1.11:3306 check inter 2000 rise 2 fall 3 weight 100
    server replica2 10.0.1.12:3306 check inter 2000 rise 2 fall 3 weight 100
    server replica3 10.0.1.13:3306 check inter 2000 rise 2 fall 3 weight 50 backup
    # replica3 = backup (utilisé uniquement si replica1/2 down)
```

#### **Configuration Galera Cluster**

```bash
# /etc/haproxy/haproxy.cfg - Galera specific

# ================================================
# FRONTEND : GALERA (All nodes accept writes)
# ================================================
frontend mysql_galera
    bind *:3306
    mode tcp
    default_backend galera_cluster_backend

backend galera_cluster_backend
    mode tcp
    balance leastconn
    option httpchk
    
    # Health check via Galera-specific script
    # Requiert xinetd + script custom sur chaque nœud
    server galera1 10.0.1.10:3306 check port 9200 inter 2000 rise 2 fall 3
    server galera2 10.0.1.11:3306 check port 9200 inter 2000 rise 2 fall 3
    server galera3 10.0.1.12:3306 check port 9200 inter 2000 rise 2 fall 3
```

**Script Health Check Galera** :
```bash
#!/bin/bash
# /usr/local/bin/clustercheck
# Health check script pour Galera via xinetd

MYSQL_USERNAME="clustercheck"
MYSQL_PASSWORD="CheckPassword"
MYSQL_HOST="localhost"
MYSQL_PORT="3306"

# Check Galera status
WSREP_STATUS=$(mysql -h $MYSQL_HOST -P $MYSQL_PORT \
    -u $MYSQL_USERNAME -p$MYSQL_PASSWORD \
    -N -e "SHOW STATUS LIKE 'wsrep_local_state';" | awk '{print $2}')

# wsrep_local_state = 4 → Synced
if [ "$WSREP_STATUS" = "4" ]; then
    # HTTP 200 OK
    echo -e "HTTP/1.1 200 OK\r\n"
    echo -e "Content-Type: text/plain\r\n"
    echo -e "\r\n"
    echo "Galera node is healthy"
else
    # HTTP 503 Service Unavailable
    echo -e "HTTP/1.1 503 Service Unavailable\r\n"
    echo -e "Content-Type: text/plain\r\n"
    echo -e "\r\n"
    echo "Galera node is NOT healthy"
fi
```

**Configuration xinetd** :
```bash
# /etc/xinetd.d/mysqlchk
service mysqlchk
{
    disable = no
    flags = REUSE
    socket_type = stream
    port = 9200
    wait = no
    user = nobody
    server = /usr/local/bin/clustercheck
    log_on_failure += USERID
    only_from = 0.0.0.0/0
    per_source = UNLIMITED
}

# Ajouter service
echo "mysqlchk 9200/tcp" >> /etc/services

# Redémarrer xinetd
systemctl restart xinetd
```

### 2.3 Optimisations Performance HAProxy

```bash
# Tuning avancé
global
    # Multi-threading (HAProxy 2.0+)
    nbthread 4
    
    # Tune buffers
    tune.bufsize 32768
    tune.maxrewrite 8192
    
    # TCP optimizations
    tune.rcvbuf.client 1048576
    tune.rcvbuf.server 1048576
    tune.sndbuf.client 1048576
    tune.sndbuf.server 1048576

defaults
    # TCP optimizations
    option tcp-smart-accept
    option tcp-smart-connect
    
    # Keep-alive
    option clitcpka
    option srvtcpka
```

### 2.4 Monitoring HAProxy

```bash
# Stats socket commands
echo "show stat" | socat stdio /run/haproxy/admin.sock

# Show servers status
echo "show servers state" | socat stdio /run/haproxy/admin.sock

# Disable server
echo "disable server mysql_replica_backend/replica1" | \
    socat stdio /run/haproxy/admin.sock

# Enable server
echo "enable server mysql_replica_backend/replica1" | \
    socat stdio /run/haproxy/admin.sock

# Prometheus exporter
docker run -d \
  --name haproxy_exporter \
  -p 9101:9101 \
  quay.io/prometheus/haproxy-exporter:latest \
  --haproxy.scrape-uri="http://admin:AdminPassword@haproxy:8404/stats;csv"
```

---

## 3. Comparaison Détaillée

### 3.1 Tableau Comparatif Fonctionnalités

| Fonctionnalité | MaxScale | ProxySQL | HAProxy |
|----------------|----------|----------|---------|
| **License** | BSL / Commercial | GPL | GPL |
| **Layer** | 7 (SQL-aware) | 7 (SQL-aware) | 4 (TCP) + 7 (HTTP) |
| **Read/Write Split** | ✅ Natif | ✅ Via rules | ⚠️ 2 frontends |
| **Query Routing** | ✅ Avancé | ✅ Regex-based | ❌ Non |
| **Query Cache** | ❌ Non | ✅ In-memory | ❌ Non |
| **Connection Pool** | ✅ Oui | ✅ Oui | ⚠️ Limité |
| **Load Balancing** | ✅ Adaptatif | ✅ Configurable | ✅ Excellent |
| **Galera Support** | ✅ Natif | ⚠️ Via scripts | ⚠️ Via xinetd |
| **Failover Auto** | ✅ mariadbmon | ⚠️ External | ❌ Non |
| **Query Firewall** | ✅ Oui | ✅ Via rules | ❌ Non |
| **SSL Termination** | ✅ Oui | ✅ Oui | ✅ Oui |
| **Multi-Master** | ✅ Oui | ✅ Oui | ✅ Oui |
| **GUI** | ✅ MaxGUI | ⚠️ Stats web | ✅ Stats web |
| **Config Hot Reload** | ✅ Oui | ✅ Oui | ✅ Oui |

### 3.2 Performance Benchmarks

```
Benchmark Setup :
- 3 nœuds MariaDB 11.8 (Galera)
- 16 cores, 64GB RAM chacun
- 10 Gbps network
- sysbench oltp_read_write

┌────────────────────────────────────────────────────────────┐
│  Solution  │ QPS    │ Latency P95 │ Overhead │ CPU Usage   │
├────────────┼────────┼─────────────┼──────────┼─────────────┤
│  Direct    │ 42,000 │    12.5ms   │    0%    │    45%      │
│  (no proxy)│        │             │          │             │
├────────────┼────────┼─────────────┼──────────┼─────────────┤
│  HAProxy   │ 41,200 │    13.1ms   │   ~2%    │    48%      │
│            │        │  (+0.6ms)   │          │  (+3%)      │
├────────────┼────────┼─────────────┼──────────┼─────────────┤
│  ProxySQL  │ 39,800 │    14.2ms   │   ~5%    │    52%      │
│            │        │  (+1.7ms)   │          │  (+7%)      │
│  (no cache)│        │             │          │             │
├────────────┼────────┼─────────────┼──────────┼─────────────┤
│  ProxySQL  │ 58,000 │     8.3ms   │  +38%    │    41%      │
│  (80% hit) │        │  (-4.2ms)   │ (gain!)  │  (-4%)      │
├────────────┼────────┼─────────────┼──────────┼─────────────┤
│  MaxScale  │ 38,500 │    15.8ms   │   ~8%    │    56%      │
│            │        │  (+3.3ms)   │          │ (+11%)      │
└────────────────────────────────────────────────────────────┘

Conclusion :
- HAProxy : Meilleure performance brute (overhead minimal)
- ProxySQL : Champion si cache hit ratio élevé
- MaxScale : Overhead plus élevé, compensé par fonctionnalités
```

### 3.3 Complexité de Configuration

```
┌─────────────────────────────────────────────────────────────┐
│  Critère              │ MaxScale │ ProxySQL  │ HAProxy      │
├───────────────────────┼──────────┼───────────┼──────────────┤
│  Lignes config        │   ~150   │   ~200    │    ~80       │
│  Courbe apprentissage │  Moyenne │  Élevée   │   Faible     │
│  Debug facilité       │  Moyen   │  Difficile│   Facile     │
│  Hot reload           │  Facile  │  Facile   │   Facile     │
│  Monitoring           │  Complet │  Détaillé │   Basique    │
│  Documentation        │  Bonne   │  Moyenne  │   Excellente │
└─────────────────────────────────────────────────────────────┘

Temps moyen pour configuration production :
- HAProxy   : 2-4 heures
- MaxScale  : 4-8 heures
- ProxySQL  : 8-16 heures (query rules complexes)
```

### 3.4 Coût Total de Possession (TCO)

```
┌────────────────────────────────────────────────────────────┐
│  Poste                │ MaxScale │ ProxySQL │ HAProxy      │
├───────────────────────┼──────────┼──────────┼──────────────┤
│  License (3 servers)  │  Gratuit │  Gratuit │   Gratuit    │
│  License (4+ servers) │  5k€/an  │  Gratuit │   Gratuit    │
│  Support commercial   │  10k€/an │  5k€/an  │   3k€/an     │
│  Training             │  2k€     │  3k€     │   1k€        │
│  Ops time (setup)     │  1 jour  │  2 jours │   0.5 jour   │
│  Ops time (maint.)    │  2h/mois │  4h/mois │   1h/mois    │
│  Hardware overhead    │  +10%    │  +5%     │   +2%        │
├───────────────────────┼──────────┼──────────┼──────────────┤
│  TCO annuel (estimé)  │  15-25k€ │  8-15k€  │   5-10k€     │
└────────────────────────────────────────────────────────────┘

Note : TCO MaxScale BSL gratuit si < 3 servers backend
```

### 3.5 Cas d'Usage Recommandés

```
┌────────────────────────────────────────────────────────────┐
│  Scénario                        │ Solution Recommandée    │
├──────────────────────────────────┼─────────────────────────┤
│  Écosystème 100% MariaDB         │ MaxScale                │
│  + Support commercial souhaité   │                         │
├──────────────────────────────────┼─────────────────────────┤
│  Workload lecture-intensive      │ ProxySQL                │
│  + Données quasi-statiques       │ (query cache)           │
│  + Budget limité                 │                         │
├──────────────────────────────────┼─────────────────────────┤
│  Performance maximale requise    │ HAProxy                 │
│  + Simple load balancing         │                         │
│  + Pas besoin SQL intelligence   │                         │
├──────────────────────────────────┼─────────────────────────┤
│  Galera Cluster production       │ MaxScale                │
│  + Failover automatique requis   │ (mariadbmon natif)      │
├──────────────────────────────────┼─────────────────────────┤
│  Mix MySQL + MariaDB             │ ProxySQL                │
│  + Query routing complexe        │ (regex rules)           │
├──────────────────────────────────┼─────────────────────────┤
│  Read/Write split simple         │ HAProxy                 │
│  + 2 frontends acceptables       │ (2 ports: 3306, 3307)   │
├──────────────────────────────────┼─────────────────────────┤
│  Database firewall requis        │ MaxScale ou ProxySQL    │
│  + Compliance/sécurité           │                         │
└────────────────────────────────────────────────────────────┘
```

---

## 4. Architectures Hybrides

### 4.1 HAProxy + MaxScale (Haute Résilience)

```
┌─────────────────────────────────────────────────┐
│                Applications                     │
└────────────────┬────────────────────────────────┘
                 │
                 │ Port 3306
                 │
        ┌────────▼────────┐
        │    HAProxy      │ ← Layer 4 LB
        │   (Active/Hot)  │
        └────────┬────────┘
                 │
      ┌──────────┼──────────┐
      │                     │
┌─────▼──────┐         ┌────▼───────┐
│ MaxScale 1 │         │ MaxScale 2 │ ← Layer 7 Intelligence
│ (Active)   │         │ (Standby)  │
└─────┬──────┘         └────┬───────┘
      │                     │
      └──────────┬──────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼─────┐ ┌────▼────┐ ┌─────▼───┐
│ Galera1 │ │ Galera2 │ │ Galera3 │
└─────────┘ └─────────┘ └─────────┘
```

**Avantages** :
- ✅ HAProxy : Load balancing ultra-performant entre MaxScale instances
- ✅ MaxScale : Intelligence SQL (read/write split, query routing)
- ✅ Résilience : Failover à deux niveaux

**Configuration HAProxy** :
```bash
frontend maxscale_lb
    bind *:3306
    mode tcp
    default_backend maxscale_pool

backend maxscale_pool
    mode tcp
    balance leastconn
    option tcp-check
    server maxscale1 10.0.2.10:3306 check inter 2000
    server maxscale2 10.0.2.11:3306 check inter 2000 backup
```

### 4.2 ProxySQL + HAProxy (Cache + Performance)

```
┌─────────────────────────────────────────────────┐
│                Applications                     │
└────────────────┬────────────────────────────────┘
                 │
        ┌────────▼────────┐
        │   ProxySQL      │ ← Query Cache + Routing
        │  (Query Cache)  │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │    HAProxy      │ ← Load Balancing
        └────────┬────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼─────┐ ┌────▼────┐ ┌─────▼───┐
│ Master  │ │ Slave 1 │ │ Slave 2 │
└─────────┘ └─────────┘ └─────────┘
```

**Cas d'usage** : Maximiser cache hits (ProxySQL) tout en garantissant load balancing optimal (HAProxy).

### 4.3 Multi-Region avec DNS Load Balancing

```
                  ┌─────────────┐
                  │ Global DNS  │
                  │ (Route53)   │
                  └──────┬──────┘
                         │
           ┌─────────────┼─────────────┐
           │                           │
    ┌──────▼──────┐             ┌──────▼──────┐
    │  Region EU  │             │ Region US   │
    │ (Paris)     │             │ (Virginia)  │
    └──────┬──────┘             └─────┬───────┘
           │                          │
    ┌──────▼──────┐             ┌─────▼───────┐
    │  MaxScale   │             │  ProxySQL   │
    └──────┬──────┘             └─────┬───────┘
           │                          │
    ┌──────▼──────┐             ┌─────▼───────┐
    │ Galera EU   │─────────────│ Galera US   │
    │ (3 nodes)   │  WAN Repl   │ (3 nodes)   │
    └─────────────┘             └─────────────┘
```

**Latency-based routing** :
- EU users → MaxScale EU → Galera EU
- US users → ProxySQL US → Galera US
- Cross-region replication for DR

---

## 5. Migration entre Solutions

### 5.1 Migration MaxScale → ProxySQL

```bash
#!/bin/bash
# migration_maxscale_to_proxysql.sh

echo "=== Migration MaxScale → ProxySQL ==="

# 1. Installer ProxySQL en parallèle
apt-get install -y proxysql

# 2. Configurer ProxySQL (mirrors MaxScale config)
# Extraire serveurs de MaxScale
SERVERS=$(maxctrl list servers --tsv | tail -n +2)

# Importer dans ProxySQL
while IFS=$'\t' read -r name address port state; do
    HOST=$(echo $address | cut -d: -f1)
    PORT=$(echo $address | cut -d: -f2)
    
    mysql -h 127.0.0.1 -P 6032 -u admin -padmin << EOF
    INSERT INTO mysql_servers (hostgroup_id, hostname, port) 
    VALUES (0, '$HOST', $PORT);
EOF
done <<< "$SERVERS"

mysql -h 127.0.0.1 -P 6032 -u admin -padmin << 'EOF'
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
EOF

# 3. Test ProxySQL
echo "Testing ProxySQL connectivity..."
mysql -h 127.0.0.1 -P 6033 -u app_user -p -e "SELECT 1"

# 4. Basculement progressif (canary)
# HAProxy route 10% traffic vers ProxySQL
cat >> /etc/haproxy/haproxy.cfg << 'EOF'
backend db_pool
    balance roundrobin
    server maxscale 10.0.2.10:3306 weight 90
    server proxysql 10.0.2.20:6033 weight 10
EOF

systemctl reload haproxy

# 5. Monitoring durant migration
watch -n 2 'mysql -h 127.0.0.1 -P 6032 -u admin -padmin \
    -e "SELECT * FROM stats_mysql_connection_pool"'

# 6. Augmenter graduellement ratio ProxySQL
# weight 90/10 → 80/20 → 50/50 → 0/100

echo "Migration en cours. Monitorer métriques."
```

### 5.2 Migration HAProxy → MaxScale

```bash
#!/bin/bash
# migration_haproxy_to_maxscale.sh

echo "=== Migration HAProxy → MaxScale ==="

# 1. Extraire configuration HAProxy
BACKENDS=$(grep "^[[:space:]]*server" /etc/haproxy/haproxy.cfg | \
    grep -v "backup" | awk '{print $2, $3}')

# 2. Générer configuration MaxScale
cat > /etc/maxscale.cnf.d/migrated.cnf << 'EOF'
[MariaDB-Monitor]
type = monitor
module = mariadbmon
servers = server1, server2, server3
user = maxscale_monitor
password = MonitorPassword
auto_failover = true
auto_rejoin = true

[Read-Write-Service]
type = service
router = readwritesplit
servers = server1, server2, server3
user = maxscale_user
password = UserPassword

[Read-Write-Listener]
type = listener
service = Read-Write-Service
protocol = MariaDBClient
port = 3306
EOF

# Ajouter serveurs dynamiquement
i=1
while IFS=' ' read -r name address; do
    HOST=$(echo $address | cut -d: -f1)
    PORT=$(echo $address | cut -d: -f2)
    
    cat >> /etc/maxscale.cnf.d/migrated.cnf << EOF

[server${i}]
type = server
address = $HOST
port = $PORT
protocol = MariaDBBackend
EOF
    ((i++))
done <<< "$BACKENDS"

# 3. Démarrer MaxScale
systemctl start maxscale

# 4. Validation
maxctrl list servers

# 5. Basculement DNS (blue-green)
# OLD: db.example.com → HAProxy IP
# NEW: db.example.com → MaxScale IP

echo "Update DNS to point to MaxScale"
echo "Monitor during TTL propagation (typically 60-300s)"
```

---

## ✅ Points Clés à Retenir

- **MaxScale** : Solution native MariaDB, BSL license, Layer 7 intelligent
- **ProxySQL** : Champion du query cache, GPL, courbe apprentissage élevée
- **HAProxy** : Performance maximale, Layer 4, configuration simple
- **Pas de winner absolu** : Choix dépend du contexte (budget, fonctionnalités, performance)
- **Query cache ProxySQL** : Game changer pour workloads read-heavy
- **HAProxy overhead** : Minimal (~2%), idéal si pas besoin SQL intelligence
- **MaxScale Galera** : Support natif optimal pour Galera Cluster
- **Architectures hybrides** : Combinaison possible pour meilleur des deux mondes
- **Migration progressive** : Canary deployment recommandé (éviter big bang)
- **Monitoring essentiel** : Comparer métriques avant/après migration

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 ProxySQL Documentation](https://proxysql.com/documentation/)
- [📖 HAProxy Documentation](https://www.haproxy.org/#docs)
- [📖 MaxScale Documentation](https://mariadb.com/kb/en/maxscale/)

### Guides de Configuration
- **"ProxySQL for MySQL"** - Percona Blog
- **"HAProxy for MySQL Load Balancing"** - DigitalOcean
- **"Comparing Database Proxies"** - High Scalability Blog

### Benchmarks
- [ProxySQL vs MaxScale Benchmark](https://www.percona.com/blog/)
- [HAProxy Performance Tuning Guide](https://www.haproxy.com/blog/)

### Outils
- [ProxySQL Admin Tool](https://github.com/sysown/proxysql-admin-tool)
- [HAProxy Exporter (Prometheus)](https://github.com/prometheus/haproxy_exporter)
- [MaxScale Docker Images](https://hub.docker.com/r/mariadb/maxscale)

---

## ➡️ Section Suivante

**[14.10 Transaction Replay et Connection Migration (11.8)](/14-haute-disponibilite/10-transaction-replay-connection-migration.md)**

La dernière section du chapitre détaillera en profondeur les deux innovations majeures de MariaDB 11.8 pour la haute disponibilité : Transaction Replay (rejouabilité automatique) et Connection Migration (préservation de session).

---

**Le choix d'un proxy n'est pas une question de religion technique, mais d'adéquation à vos besoins réels. ProxySQL brille avec son cache, HAProxy avec sa performance, MaxScale avec son intégration native. Choisissez selon votre contexte, pas selon la hype.**

⏭️ [Transaction Replay et Connection Migration](/14-haute-disponibilite/10-transaction-replay-connection-migration.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.6 Réplication en Cascade

> **Niveau** : Avancé  
> **Durée estimée** : 2-2.5 heures  
> **Prérequis** : 
> - Sections 13.1 à 13.5 (Réplication fondamentale, GTID, multi-source)
> - Compréhension des architectures hiérarchiques
> - Notions de latence réseau et bande passante
> - Expérience en troubleshooting de réplication

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre l'architecture de réplication en cascade et ses cas d'usage
- Configurer une topologie de réplication multi-niveaux
- Activer et gérer `log_slave_updates` correctement
- Analyser et gérer le lag cumulatif dans une cascade
- Identifier les points de défaillance et les stratégies de mitigation
- Monitorer efficacement une topologie en cascade
- Décider quand utiliser (ou éviter) la réplication en cascade

---

## Introduction

La **réplication en cascade** (ou **chained replication**) est une architecture où un serveur Replica agit **simultanément** comme :
- **Replica** d'un Primary (reçoit les données)
- **Primary** pour d'autres Replicas (transmet les données)

```
Primary → Relay1 → Relay2 → Replica
          (Replica + Primary)
```

### Le problème résolu

**Sans cascade** : Surcharge du Primary

```
                    Primary
                      │
      ┌───────┬───────┼───────┬───────┬───────┐
      │       │       │       │       │       │
      ▼       ▼       ▼       ▼       ▼       ▼
   Replica1 Replica2 Replica3 Replica4 Replica5 Replica6

Problème:
- Primary envoie binlog à 6 Replicas
- 6 × threads Binlog Dump
- 6 × consommation bande passante
- Scalabilité limitée
```

**Avec cascade** : Distribution de charge

```
                Primary
                  │
                  ▼
                Relay1
                  │
          ┌───────┼───────┐
          │       │       │
          ▼       ▼       ▼
       Replica1 Replica2 Replica3
       
                Relay2
                  │
          ┌───────┼───────┐
          │       │       │
          ▼       ▼       ▼
       Replica4 Replica5 Replica6

Avantages:
- Primary → 2 connexions seulement
- Relay1/Relay2 distribuent la charge
- Scalabilité améliorée
```

### Historique

**Timeline** :

```
2003 - MySQL 4.0
       │ Introduction concept cascade
       │ log-slave-updates disponible
       ▼
       
2014 - MariaDB 10.0
       │ GTID facilite cascade multi-niveaux
       │ Meilleure gestion lag cumulatif
       ▼
       
2017 - MariaDB 10.3
       │ Optimisations cascade avec GTID
       │ Monitoring amélioré
       ▼
       
2025 - MariaDB 11.8 LTS
       │ Cascade production-ready
       │ Parallel replication optimisée pour cascade
```

---

## Architecture de Réplication en Cascade

### Topologie simple (2 niveaux)

```
┌─────────────────────────────────────────────────────────────┐
│                    NIVEAU 1: PRIMARY                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────┐                   │
│  │  Primary                             │                   │
│  │  server-id: 1                        │                   │
│  │  log-bin: ON                         │                   │
│  │  gtid_domain_id: 0                   │                   │
│  │                                      │                   │
│  │  Binlog: mariadb-bin.000042          │                   │
│  └─────────────────┬────────────────────┘                   │
│                    │                                        │
└────────────────────┼────────────────────────────────────────┘
                     │
                     │ Replication
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    NIVEAU 2: RELAY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────┐                   │
│  │  Relay (Intermediate)                │                   │
│  │  server-id: 2                        │                   │
│  │  log-bin: ON ✓                       │  ← CRITIQUE !     │
│  │  log-slave-updates: ON ✓             │  ← CRITIQUE !     │
│  │  gtid_domain_id: 0                   │                   │
│  │                                      │                   │
│  │  ┌────────────────────────────────┐  │                   │
│  │  │ Rôle REPLICA (du Primary):     │  │                   │
│  │  │ - IO Thread                    │  │                   │
│  │  │ - SQL Thread                   │  │                   │
│  │  │ - Relay Log: relay-bin.*       │  │                   │
│  │  └────────────────────────────────┘  │                   │
│  │                                      │                   │
│  │  ┌────────────────────────────────┐  │                   │
│  │  │ Rôle PRIMARY (pour Replicas):  │  │                   │
│  │  │ - Binlog: mariadb-bin.000010   │  │                   │
│  │  │ - Binlog Dump Threads          │  │                   │
│  │  └────────┬───────────────────────┘  │                   │
│  └───────────┼──────────────────────────┘                   │
│              │                                              │
└──────────────┼──────────────────────────────────────────────┘
               │
               ├────────────┬─────────────┐
               │            │             │
               ▼            ▼             ▼
┌─────────────────────────────────────────────────────────────┐
│                  NIVEAU 3: REPLICAS FINAUX                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐             │
│  │ Replica1 │     │ Replica2 │     │ Replica3 │             │
│  │ id: 10   │     │ id: 11   │     │ id: 12   │             │
│  └──────────┘     └──────────┘     └──────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Topologie complexe (3+ niveaux)

```
                        Primary
                          │
                ┌─────────┴─────────┐
                │                   │
                ▼                   ▼
             Relay1               Relay2
          (Datacenter A)      (Datacenter B)
                │                   │
        ┌───────┼───────┐   ┌───────┼───────┐
        │       │       │   │       │       │
        ▼       ▼       ▼   ▼       ▼       ▼
     Relay1A Relay1B Relay1C Relay2A Relay2B Relay2C
        │       │       │   │       │       │
        ▼       ▼       ▼   ▼       ▼       ▼
    Replicas Replicas Replicas Replicas Replicas Replicas
```

**Niveaux typiques** :
- **Niveau 1** : Primary source
- **Niveau 2** : Relays régionaux (par datacenter, continent)
- **Niveau 3** : Relays locaux (par site, service)
- **Niveau 4** : Replicas finaux (application, backup)

⚠️ **Limitation** : Maximum **4-5 niveaux** recommandés (lag exponentiel au-delà)

---

## Configuration log_slave_updates

Le paramètre **`log_slave_updates`** est **CRITIQUE** pour la réplication en cascade.

### Fonctionnement

**Sans log_slave_updates (défaut)** :

```
Primary
  ↓ Binlog: INSERT INTO users...
Relay
  ├─ Reçoit via IO Thread
  ├─ Applique via SQL Thread
  └─ N'ÉCRIT PAS dans son propre binlog ❌

Replicas niveau 3 → Ne reçoivent RIEN ❌
```

**Avec log_slave_updates=ON** :

```
Primary
  ↓ Binlog: INSERT INTO users...
Relay
  ├─ Reçoit via IO Thread
  ├─ Applique via SQL Thread
  └─ ÉCRIT dans son propre binlog ✓

Replicas niveau 3 → Reçoivent les événements ✓
```

### Configuration

**Sur TOUS les serveurs intermédiaires (Relays)** :

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf

[mysqld]
# Configuration Relay
server-id = 2                # Unique !
log-bin = mariadb-bin        # Activer binlog
log-slave-updates = ON       # CRITIQUE pour cascade !

# Si GTID utilisé
gtid_domain_id = 0
gtid_strict_mode = ON

# Relay log
relay-log = relay-bin
relay_log_recovery = ON

# Read-only (optionnel, selon use case)
read_only = ON
super_read_only = ON
```

**Vérification** :

```sql
-- Sur le Relay
SELECT @@log_slave_updates;
-- +---------------------+
-- | @@log_slave_updates |
-- +---------------------+
-- |                   1 |  ← Doit être 1 (ON)
-- +---------------------+

-- Vérifier que le binlog est actif
SHOW MASTER STATUS;
-- +--------------------+----------+--------------+------------------+
-- | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +--------------------+----------+--------------+------------------+
-- | mariadb-bin.000010 | 5678     |              |                  |
-- +--------------------+----------+--------------+------------------+
```

### Impact sur les performances

**Overhead de log_slave_updates** :

```
Transaction sur Primary:
1. Écriture InnoDB
2. Écriture binlog Primary
3. Réplication → Relay
4. Relay: Écriture InnoDB
5. Relay: Écriture binlog (overhead log_slave_updates)
6. Réplication → Replicas niveau 3

Overhead:
- CPU: +5-10% (parsing + écriture binlog)
- I/O disque: +20-30% (binlog supplémentaire)
- Mémoire: Minime
```

**Benchmark** :

```
Sysbench OLTP Mixed (relay avec log_slave_updates)
Config: 16 vCPU, 64GB RAM, SSD NVMe

Sans log_slave_updates (Relay simple):
- TPS: 15,420
- Latence P95: 8.2ms
- Disk Write: 120 MB/s

Avec log_slave_updates (Relay cascade):
- TPS: 14,180 (-8%)
- Latence P95: 9.1ms (+0.9ms)
- Disk Write: 156 MB/s (+30%)

Conclusion: Overhead acceptable pour les avantages de la cascade
```

---

## Lag Cumulatif

### Le problème

Le **lag se cumule** à travers les niveaux de cascade :

```
Timeline d'une transaction:

T0: Primary exécute INSERT
T1: Primary écrit binlog (lag P = 0)
T2: Relay1 reçoit (lag R1 = 100ms)
T3: Relay1 applique (lag R1 = 100ms)
T4: Relay1 écrit binlog
T5: Relay2 reçoit (lag R2 = 100ms + 100ms = 200ms)
T6: Replica final reçoit (lag = 300ms+)

Lag total = Lag Niveau1 + Lag Niveau2 + Lag Niveau3 + ...
```

### Calcul du lag cumulatif

**Formule** :

```
Lag_total = Σ (Lag_niveau_i + Latence_réseau_i)
            i=1 to N

Où:
- N = Nombre de niveaux
- Lag_niveau_i = Secondes de retard application SQL thread
- Latence_réseau_i = Temps de transfert réseau
```

**Exemple concret** :

```
Topologie:
Primary → Relay1 (WAN, 50ms) → Relay2 (LAN, 2ms) → Replica

Scénario:
1. Primary → Relay1
   - Latence réseau: 50ms
   - Application SQL: 20ms
   - Total niveau 1: 70ms

2. Relay1 → Relay2
   - Latence réseau: 2ms
   - Application SQL: 15ms
   - Total niveau 2: 17ms

3. Relay2 → Replica
   - Latence réseau: 1ms
   - Application SQL: 10ms
   - Total niveau 3: 11ms

Lag total: 70 + 17 + 11 = 98ms (acceptable)

Mais si chaque niveau a 1s de lag:
Lag total: 1000 + 1000 + 1000 = 3000ms (problématique !)
```

### Visualisation du lag

```sql
-- Script pour tracer le lag à travers la cascade

-- Sur Primary
CREATE TABLE heartbeat (
  ts TIMESTAMP(6),
  server_id INT,
  PRIMARY KEY (server_id)
) ENGINE=InnoDB;

-- Job périodique sur Primary
INSERT INTO heartbeat VALUES (NOW(6), @@server_id)
ON DUPLICATE KEY UPDATE ts = NOW(6);

-- Sur chaque Relay et Replica
SELECT 
  @@hostname AS server,
  @@server_id AS server_id,
  TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000 AS lag_ms
FROM heartbeat
WHERE server_id = 1  -- Primary server_id
ORDER BY lag_ms DESC;
```

**Output cascade à 3 niveaux** :

```
+------------------+-----------+--------+
| server           | server_id | lag_ms |
+------------------+-----------+--------+
| replica-final    | 12        | 350.5  |  ← Lag cumulatif
| relay2           | 3         | 180.2  |  ← Niveau 2
| relay1           | 2         | 85.1   |  ← Niveau 1
| primary          | 1         | 0.0    |  ← Source
+------------------+-----------+--------+
```

### Stratégies de réduction du lag

**1. Parallélisation agressive sur Relays**

```sql
-- Sur chaque Relay
SET GLOBAL slave_parallel_threads = 8;  -- Plus élevé que Replica final
SET GLOBAL slave_parallel_mode = 'optimistic';
SET GLOBAL slave_domain_parallel_threads = 4;  -- Si multi-domain
```

**2. Hardware performant pour Relays**

```
Relays:
- SSD NVMe (I/O rapide pour binlog + relay log)
- CPU rapide (parsing binlog)
- RAM suffisante (buffer pool)
- Réseau 10Gbps+ si haute volumétrie

Replicas finaux:
- Peuvent être moins puissants
- Le Relay absorbe la charge
```

**3. Minimiser les niveaux**

```
Mauvais (5 niveaux):
Primary → R1 → R2 → R3 → R4 → Replica
Lag: 50 + 50 + 50 + 50 + 50 = 250ms

Meilleur (2 niveaux):
Primary → R1 → Replica
Lag: 50 + 50 = 100ms

Optimal (1 niveau, si possible):
Primary → Replica
Lag: 50ms
```

**4. Compression pour WAN**

```sql
-- Sur connexions longue distance
CHANGE MASTER TO
  MASTER_HOST = 'remote-relay.example.com',
  MASTER_USE_GTID = slave_pos,
  MASTER_COMPRESSION_ALGORITHMS = 'zstd';  -- Réduit latence réseau
```

**5. Monitoring continu**

```sql
-- Alerte si lag > seuil
SELECT 
  @@hostname,
  (SELECT TIMESTAMPDIFF(SECOND, ts, NOW()) 
   FROM heartbeat WHERE server_id = 1) AS lag_sec
HAVING lag_sec > 10;  -- Alerte si > 10 secondes
```

---

## Cas d'Usage

### 1. Distribution géographique

**Scénario** : Application multi-continents

```
Europe (Primary)
  │
  ├─────────────────────┬─────────────────────┐
  │                     │                     │
  ▼                     ▼                     ▼
US Relay            APAC Relay          LATAM Relay
(New York)          (Singapore)         (São Paulo)
  │                     │                     │
  ├──────┬──────┐       ├──────┬──────┐       └─── Replicas
  │      │      │       │      │      │
  ▼      ▼      ▼       ▼      ▼      ▼
Replicas locales   Replicas locales
(Boston, Miami,    (Tokyo, Sydney,
 Chicago)           Mumbai)
```

**Avantages** :
- ✅ Un seul flux transatlantique (Primary → Relay)
- ✅ Distribution locale rapide
- ✅ Économie de bande passante internationale
- ✅ Latence réduite pour lectures locales

**Configuration** :

```sql
-- US Relay (New York)
CHANGE MASTER TO
  MASTER_HOST = 'primary-eu.example.com',
  MASTER_USE_GTID = slave_pos,
  MASTER_COMPRESSION_ALGORITHMS = 'zstd';  -- WAN

-- Replicas US locales
CHANGE MASTER TO
  MASTER_HOST = 'relay-us-ny.example.com',  -- LAN
  MASTER_USE_GTID = slave_pos;
```

### 2. Scalabilité de lecture

**Scénario** : 50+ Replicas en lecture

```
Primary
  │
  ├───────┬───────┐
  │       │       │
  ▼       ▼       ▼
Relay1  Relay2  Relay3
  │       │       │
  ├─...   ├─...   ├─...
  ▼       ▼       ▼
20 Reps 15 Reps 15 Reps

Total: 50 Replicas
Primary connections: 3 (vs 50 sans cascade !)
```

**Avantages** :
- ✅ Charge Primary réduite (3 connexions vs 50)
- ✅ Scalabilité quasi-illimitée
- ✅ Ajout/retrait Replica transparent

### 3. Migration progressive

**Scénario** : Migration datacenter

```
Phase 1: Setup Relay dans nouveau DC
Old DC Primary → New DC Relay
                     (pas encore de Replicas)

Phase 2: Ajouter Replicas dans nouveau DC
Old DC Primary → New DC Relay → New DC Replicas

Phase 3: Basculer Primary
New DC Primary → Old DC Relay (réplication inverse)
              → New DC Replicas

Phase 4: Nettoyer
New DC Primary → New DC Replicas (direct)
Old DC décommissioné
```

### 4. Isolation environnements

**Scénario** : Production → Staging → Dev

```
Production Primary
  │
  ▼
Staging Relay (log_slave_updates)
  │
  ├─────────────┐
  │             │
  ▼             ▼
Dev1 Replica  Dev2 Replica

Avantages:
- Production protégée (1 seule connexion)
- Staging peut avoir données masquées
- Devs ont données récentes sans impact prod
```

---

## Configuration Complète Cascade

### Étape 1 : Préparer le Primary

```ini
# Primary
[mysqld]
server-id = 1
log-bin = mariadb-bin
binlog_format = ROW
gtid_domain_id = 0
max_binlog_size = 100M
sync_binlog = 1
```

```sql
-- Créer utilisateur réplication
CREATE USER 'repl_relay'@'%' 
  IDENTIFIED BY 'Secure_Relay_P@ss_2025!';
  
GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_relay'@'%';
```

### Étape 2 : Configurer le Relay

```ini
# Relay (Intermediate)
[mysqld]
server-id = 2                # Unique !
log-bin = mariadb-bin        # Binlog actif
log-slave-updates = ON       # CRITIQUE !

binlog_format = ROW
gtid_domain_id = 0

# Relay log config
relay-log = relay-bin
relay_log_recovery = ON

# Performance
slave_parallel_threads = 4
slave_parallel_mode = optimistic

# Read-only (optionnel)
read_only = ON
super_read_only = ON
```

**Initialisation données** :

```bash
# Dump depuis Primary
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --routines \
  --triggers \
  --events \
  > primary_dump.sql

# Restaurer sur Relay
mariadb -u root -p < primary_dump.sql
```

**Configuration réplication** :

```sql
-- Sur Relay
CHANGE MASTER TO
  MASTER_HOST = 'primary.example.com',
  MASTER_USER = 'repl_relay',
  MASTER_PASSWORD = 'Secure_Relay_P@ss_2025!',
  MASTER_USE_GTID = slave_pos;

START SLAVE;

-- Vérifier
SHOW SLAVE STATUS\G
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
```

**Créer utilisateur pour Replicas niveau 3** :

```sql
-- Sur Relay
CREATE USER 'repl_final'@'%' 
  IDENTIFIED BY 'Secure_Final_P@ss_2025!';
  
GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_final'@'%';
```

### Étape 3 : Configurer Replicas finaux

```ini
# Replica final
[mysqld]
server-id = 10               # Unique !
relay-log = relay-bin
relay_log_recovery = ON
read_only = ON
super_read_only = ON
```

```bash
# Dump depuis Relay (ou Primary)
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  > relay_dump.sql

mariadb -u root -p < relay_dump.sql
```

```sql
-- Sur Replica final
CHANGE MASTER TO
  MASTER_HOST = 'relay.example.com',  -- Pointe vers Relay !
  MASTER_USER = 'repl_final',
  MASTER_PASSWORD = 'Secure_Final_P@ss_2025!',
  MASTER_USE_GTID = slave_pos;

START SLAVE;

SHOW SLAVE STATUS\G
```

### Étape 4 : Validation

**Test de propagation** :

```sql
-- Sur Primary
CREATE DATABASE cascade_test;
USE cascade_test;
CREATE TABLE test (id INT PRIMARY KEY, data VARCHAR(100));
INSERT INTO test VALUES (1, 'Test at ' || NOW());

-- Sur Relay (attendre quelques secondes)
USE cascade_test;
SELECT * FROM test;
-- Doit afficher la ligne

-- Sur Replica final (attendre quelques secondes)
USE cascade_test;
SELECT * FROM test;
-- Doit afficher la ligne

-- Si OK : Cascade fonctionne ✓
```

---

## Monitoring Cascade

### Métriques essentielles

**1. Lag par niveau**

```sql
-- Sur Primary: Insérer heartbeat
CREATE EVENT heartbeat_event
ON SCHEDULE EVERY 1 SECOND
DO
  INSERT INTO heartbeat (ts, server_id) 
  VALUES (NOW(6), @@server_id)
  ON DUPLICATE KEY UPDATE ts = NOW(6);

-- Sur chaque serveur: Mesurer lag
SELECT 
  @@hostname AS server,
  @@server_id AS id,
  'Primary' AS level,
  0 AS lag_ms
FROM dual
WHERE @@server_id = 1

UNION ALL

SELECT 
  @@hostname,
  @@server_id,
  'Relay' AS level,
  TIMESTAMPDIFF(MICROSECOND, 
    (SELECT ts FROM heartbeat WHERE server_id = 1), 
    NOW(6)
  ) / 1000 AS lag_ms
FROM dual
WHERE @@server_id = 2

UNION ALL

SELECT 
  @@hostname,
  @@server_id,
  'Replica' AS level,
  TIMESTAMPDIFF(MICROSECOND, 
    (SELECT ts FROM heartbeat WHERE server_id = 1), 
    NOW(6)
  ) / 1000 AS lag_ms
FROM dual
WHERE @@server_id = 10;
```

**2. État réplication à chaque niveau**

```sql
-- Script consolidé (à exécuter sur monitoring server)
SELECT 
  h.hostname,
  h.server_id,
  h.level,
  s.slave_io_running,
  s.slave_sql_running,
  s.seconds_behind_master,
  s.gtid_io_pos
FROM 
  (SELECT 'relay' AS hostname, 2 AS server_id, 'Relay1' AS level
   UNION SELECT 'replica1', 10, 'Replica1'
   UNION SELECT 'replica2', 11, 'Replica2') h
LEFT JOIN information_schema.SLAVE_STATUS s 
  ON h.server_id = @@server_id;
```

**3. Throughput par niveau**

```sql
-- Événements répliqués par seconde
SELECT 
  connection_name,
  @prev_pos := @curr_pos AS prev_pos,
  @curr_pos := read_master_log_pos AS curr_pos,
  @curr_pos - @prev_pos AS events_per_sec
FROM information_schema.SLAVE_STATUS,
     (SELECT @prev_pos := 0, @curr_pos := 0) vars
WHERE connection_name = '';

-- Exécuter toutes les secondes pour calculer taux
```

### Dashboard Grafana

**Panel 1: Cascade Lag Overview**

```promql
# Prometheus query
mariadb_cascade_lag_seconds{level="relay1"}
mariadb_cascade_lag_seconds{level="relay2"}
mariadb_cascade_lag_seconds{level="replica_final"}

# Visualisation: Graph empilé (stacked area)
```

**Panel 2: Cascade Health**

```promql
# Status binaire (1=OK, 0=ERROR)
mariadb_cascade_replication_status{server="relay1"}
mariadb_cascade_replication_status{server="relay2"}
mariadb_cascade_replication_status{server="replica"}

# Visualisation: Heatmap ou gauge
```

**Panel 3: Cumulative Lag**

```promql
# Somme du lag à travers cascade
sum(mariadb_cascade_lag_seconds)

# Alert: > 10 seconds
```

### Script de monitoring automatisé

```bash
#!/bin/bash
# cascade_health_check.sh

SERVERS=(
  "primary.example.com:primary"
  "relay1.example.com:relay1"
  "relay2.example.com:relay2"
  "replica1.example.com:replica1"
)

echo "=== Cascade Replication Health ==="
echo "Timestamp: $(date)"
echo ""

TOTAL_LAG=0
ERROR_COUNT=0

for SERVER_INFO in "${SERVERS[@]}"; do
  IFS=':' read -r HOST ROLE <<< "$SERVER_INFO"
  
  echo "[$ROLE] $HOST"
  
  if [ "$ROLE" = "primary" ]; then
    # Primary: Vérifier binlog actif
    BINLOG=$(mysql -h $HOST -N -e "SELECT @@log_bin")
    if [ "$BINLOG" = "1" ]; then
      echo "  ✓ Binlog: Active"
    else
      echo "  ✗ Binlog: Inactive"
      ERROR_COUNT=$((ERROR_COUNT + 1))
    fi
  else
    # Replica/Relay: Vérifier réplication
    STATUS=$(mysql -h $HOST -N -e "
      SELECT CONCAT(
        CASE WHEN slave_io_running='Yes' AND slave_sql_running='Yes' 
        THEN 'RUNNING' ELSE 'STOPPED' END,
        '|',
        IFNULL(seconds_behind_master, 'NULL')
      )
      FROM information_schema.SLAVE_STATUS 
      LIMIT 1
    ")
    
    IFS='|' read -r STATE LAG <<< "$STATUS"
    
    if [ "$STATE" = "RUNNING" ]; then
      echo "  ✓ Status: $STATE"
      echo "  ⏱ Lag: ${LAG}s"
      TOTAL_LAG=$((TOTAL_LAG + LAG))
    else
      echo "  ✗ Status: $STATE"
      ERROR_COUNT=$((ERROR_COUNT + 1))
    fi
    
    # Vérifier log_slave_updates si Relay
    if [[ "$ROLE" == relay* ]]; then
      LSU=$(mysql -h $HOST -N -e "SELECT @@log_slave_updates")
      if [ "$LSU" = "1" ]; then
        echo "  ✓ log_slave_updates: ON"
      else
        echo "  ✗ log_slave_updates: OFF (CRITICAL!)"
        ERROR_COUNT=$((ERROR_COUNT + 1))
      fi
    fi
  fi
  
  echo ""
done

echo "=== Summary ==="
echo "Total cumulative lag: ${TOTAL_LAG}s"
echo "Errors: $ERROR_COUNT"

if [ $ERROR_COUNT -gt 0 ]; then
  echo "❌ HEALTH CHECK FAILED"
  exit 2
elif [ $TOTAL_LAG -gt 60 ]; then
  echo "⚠️  WARNING: High cumulative lag"
  exit 1
else
  echo "✅ HEALTH CHECK PASSED"
  exit 0
fi
```

---

## Troubleshooting Cascade

### Problème 1 : Relay ne transmet pas aux Replicas

```sql
-- Sur Replica final
SHOW SLAVE STATUS\G
-- Last_IO_Error: Got fatal error 1236 from master when reading data from binary log
```

**Cause** : `log_slave_updates` non activé sur Relay

**Diagnostic** :

```sql
-- Sur Relay
SELECT @@log_slave_updates;
-- +---------------------+
-- | @@log_slave_updates |
-- +---------------------+
-- |                   0 |  ← PROBLÈME ! Doit être 1
-- +---------------------+
```

**Solution** :

```sql
-- Sur Relay: Activer log_slave_updates (nécessite redémarrage)
-- Dans my.cnf:
[mysqld]
log_slave_updates = ON

-- Redémarrer
systemctl restart mariadb

-- Vérifier
SELECT @@log_slave_updates;
-- Doit être 1

-- Sur Replica final: Reconfigurer
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST = 'relay.example.com',
  MASTER_USE_GTID = slave_pos;
START SLAVE;
```

### Problème 2 : Lag exponentiel

```sql
-- Lag observé
Primary → Relay1: 100ms
Relay1 → Relay2: 500ms
Relay2 → Replica: 2000ms

Total: 2600ms (inacceptable)
```

**Causes** :
- Relays sous-dimensionnés
- Pas de parallélisation
- Trop de niveaux

**Solutions** :

```sql
-- 1. Activer parallélisation agressive
-- Sur Relay1
SET GLOBAL slave_parallel_threads = 8;
SET GLOBAL slave_parallel_mode = 'optimistic';

-- Sur Relay2
SET GLOBAL slave_parallel_threads = 8;
SET GLOBAL slave_parallel_mode = 'optimistic';

-- 2. Vérifier hardware
-- CPU surchargé ?
top
# Si load average > nombre de cores → Upgrade CPU

-- Disque saturé ?
iostat -x 1
# Si %util > 80% → Upgrade vers SSD NVMe

-- 3. Réduire niveaux (si possible)
-- Éliminer Relay2:
Primary → Relay1 → Replicas (direct)
```

### Problème 3 : Relay tombe, Replicas orphelins

```sql
-- Relay1 CRASH 💥

-- Sur Replica final
SHOW SLAVE STATUS\G
-- Last_IO_Error: Error connecting to master 'repl_final@relay1:3306'
```

**Solution immédiate : Basculer sur Primary directement**

```sql
-- Sur Replica final
STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST = 'primary.example.com',  -- Direct vers Primary
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;  -- Position recalculée auto avec GTID

START SLAVE;
```

**Solution long terme : Redondance Relays**

```
Primary
  │
  ├──────────┬──────────┐
  │          │          │
  ▼          ▼          ▼
Relay1A   Relay1B   Relay1C
  │          │          │
  └────┬─────┴─────┬────┘
       │           │
       ▼           ▼
    Replicas   Replicas

Si Relay1A tombe:
- Basculer vers Relay1B ou Relay1C
- Avec GTID: Automatique
```

### Problème 4 : GTID out of order

```sql
-- Sur Replica niveau 3
SHOW SLAVE STATUS\G
-- Last_SQL_Error: An attempt was made to binlog GTID 0-1-1000 which would create an out-of-order sequence number
```

**Cause** : Transactions non ordonnées à travers cascade

**Solution** :

```sql
-- Activer gtid_strict_mode sur TOUS les serveurs
-- Primary
SET GLOBAL gtid_strict_mode = ON;

-- Relay
SET GLOBAL gtid_strict_mode = ON;

-- Replicas
SET GLOBAL gtid_strict_mode = ON;

-- Redémarrer réplication
STOP SLAVE;
START SLAVE;
```

---

## Bonnes Pratiques

### 1. Limiter les niveaux

```
✅ Recommandé:
Primary → Relay → Replicas (2 niveaux)
Primary → Relay1 → Relay2 → Replicas (3 niveaux)

⚠️ À éviter:
Primary → R1 → R2 → R3 → R4 → Replicas (5+ niveaux)
Lag cumulatif excessif
```

### 2. Dimensionner les Relays

**Hardware Relay** :
- CPU : 2-4× plus puissant que Replica standard
- RAM : Buffer pool généreux (≥ 50% de RAM)
- Disque : SSD NVMe (binlog + relay log I/O intense)
- Réseau : 10Gbps+ si haute volumétrie

**Configuration** :

```ini
[mysqld]
# Relay optimisé
innodb_buffer_pool_size = 32G       # 50-70% RAM
innodb_log_file_size = 2G
innodb_flush_log_at_trx_commit = 2  # Moins strict (Relay peut être reconstruit)
sync_binlog = 0                     # Moins strict
slave_parallel_threads = 8
slave_parallel_mode = optimistic
```

### 3. Toujours utiliser GTID

```sql
-- GTID simplifie radicalement la cascade

-- Ajout Replica niveau 3: Trivial avec GTID
CHANGE MASTER TO
  MASTER_HOST = 'relay.example.com',
  MASTER_USE_GTID = slave_pos;  -- Position auto !

-- Sans GTID: Complexe
-- Trouver position exacte dans binlog du Relay
-- Risque d'erreur
```

### 4. Redondance Relays critiques

```
Pour Relays critiques (geo-distribution):

          Primary
             │
     ┌───────┴───────┐
     ▼               ▼
  Relay-A         Relay-B  (Redondance)
     │               │
     └───────┬───────┘
             │
          Replicas

Si Relay-A tombe:
- Replicas basculent vers Relay-B
- Avec Orchestrator: Automatique
```

### 5. Monitoring proactif

```sql
-- Alerte si lag > seuil acceptable
CREATE EVENT check_cascade_lag
ON SCHEDULE EVERY 1 MINUTE
DO
  BEGIN
    DECLARE total_lag INT;
    
    SELECT SUM(seconds_behind_master) INTO total_lag
    FROM information_schema.SLAVE_STATUS;
    
    IF total_lag > 30 THEN
      INSERT INTO alerts (severity, message, created_at)
      VALUES ('WARNING', 
              CONCAT('Cascade lag: ', total_lag, 's'), 
              NOW());
    END IF;
  END;
```

### 6. Documentation topologie

```sql
CREATE TABLE cascade_topology (
  server_id INT PRIMARY KEY,
  hostname VARCHAR(255),
  role ENUM('primary', 'relay', 'replica'),
  level INT,  -- Niveau dans cascade
  parent_server_id INT,
  gtid_domain INT,
  notes TEXT,
  FOREIGN KEY (parent_server_id) REFERENCES cascade_topology(server_id)
);

INSERT INTO cascade_topology VALUES
(1, 'primary.example.com', 'primary', 1, NULL, 0, 'Production primary'),
(2, 'relay1.example.com', 'relay', 2, 1, 0, 'US relay'),
(3, 'relay2.example.com', 'relay', 2, 1, 0, 'EU relay'),
(10, 'replica1.example.com', 'replica', 3, 2, 0, 'US read replica'),
(11, 'replica2.example.com', 'replica', 3, 3, 0, 'EU read replica');

-- Requête topologie
SELECT 
  CONCAT(REPEAT('  ', level-1), hostname) AS topology
FROM cascade_topology
ORDER BY level, server_id;
-- primary.example.com
--   relay1.example.com
--     replica1.example.com
--   relay2.example.com
--     replica2.example.com
```

---

## ✅ Points clés à retenir

1. **log_slave_updates** : Paramètre CRITIQUE pour cascade, doit être ON sur TOUS les Relays

2. **Lag cumulatif** : Se cumule à travers les niveaux, limiter à 2-3 niveaux maximum

3. **GTID essentiel** : Simplifie configuration et failover de manière dramatique

4. **Hardware Relays** : Doivent être plus puissants que Replicas standards

5. **Use cases** : Geo-distribution, scalabilité lecture, isolation environnements

6. **Monitoring** : Mesurer lag à chaque niveau avec table heartbeat

7. **Parallélisation** : Agressive sur Relays pour réduire lag

8. **Redondance** : Relays critiques doivent être redondants

9. **Limitation** : Maximum 4-5 niveaux avant lag inacceptable

10. **Documentation** : Topologie doit être documentée et à jour

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Replication Overview](https://mariadb.com/kb/en/replication-overview/)
- [📖 log_slave_updates](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/#log_slave_updates)
- [📖 Parallel Replication](https://mariadb.com/kb/en/parallel-replication/)

### Articles techniques

- [🔗 Chained Replication Best Practices](https://mariadb.com/resources/blog/)
- [🔗 Measuring Replication Lag](https://www.percona.com/blog/)

### Outils

- **pt-heartbeat** (Percona Toolkit) : Mesure précise lag cascade
- **Orchestrator** : Gestion topologies complexes avec cascade
- **MaxScale** : Routing intelligent multi-niveaux

---

## ➡️ Section suivante

**13.7 Monitoring et troubleshooting** : Commandes SHOW REPLICA STATUS détaillées, analyse du lag, diagnostic d'erreurs courantes, outils de monitoring (pt-heartbeat, Orchestrator), et métriques critiques de production.

---


⏭️ [Monitoring et troubleshooting](/13-replication/07-monitoring-troubleshooting.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.5 Réplication Multi-Source

> **Niveau** : Avancé  
> **Durée estimée** : 2-2.5 heures  
> **Prérequis** : 
> - Sections 13.1 à 13.4 (Concepts réplication, GTID)
> - Compréhension des topologies distribuées
> - Expérience en configuration de réplication simple
> - Notions de consolidation de données

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre l'architecture et les cas d'usage de la réplication multi-source
- Configurer un serveur pour répliquer depuis plusieurs Primary simultanément
- Gérer les canaux de réplication nommés
- Utiliser GTID avec multi-source pour simplifier la gestion
- Identifier et résoudre les conflits potentiels entre sources
- Monitorer efficacement une topologie multi-source
- Évaluer quand utiliser (ou non) la réplication multi-source

---

## Introduction

La **réplication multi-source** permet à un serveur MariaDB de répliquer depuis **plusieurs serveurs Primary** simultanément. Chaque connexion de réplication est gérée dans un **canal nommé** indépendant.

### Le problème résolu

**Sans multi-source** : Consolidation manuelle

```
Primary A (Sales)     Primary B (HR)      Primary C (Finance)
     │                     │                     │
     ▼                     ▼                     ▼
  App reads           App reads              App reads
  from A only         from B only            from C only

Problème: Impossible de croiser les données sans ETL complexe
```

**Avec multi-source** : Consolidation automatique

```
Primary A (Sales)     Primary B (HR)      Primary C (Finance)
     │                     │                     │
     └──────────┬──────────┴──────────┬──────────┘
                │                     │
                ▼                     ▼
           ┌──────────────────────────────┐
           │   Multi-Source Replica       │
           │                              │
           │  Channel 'sales'   → DB sales│
           │  Channel 'hr'      → DB hr   │
           │  Channel 'finance' → DB fin  │
           └──────────────────────────────┘
                        ▼
              Unified Reporting/Analytics
```

### Historique

**Timeline** :

```
2015 - MySQL 5.7
       │ Introduction Multi-Source Replication
       │ Limitation: Pas de GTID obligatoire
       ▼
       
2014 - MariaDB 10.0
       │ Support Multi-Source avec GTID MariaDB
       │ Implémentation native et flexible
       ▼
       
2017 - MariaDB 10.3
       │ Améliorations stabilité multi-source
       │ Meilleur monitoring
       ▼
       
2025 - MariaDB 11.8 LTS
       │ Multi-source mature et production-ready
       │ Intégration parfaite avec GTID domains
       │ Outils monitoring améliorés
```

💡 **MariaDB 11.8** : La réplication multi-source est maintenant considérée comme stable et recommandée pour les architectures de consolidation.

---

## Architecture Multi-Source

### Topologie de base

```
┌────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE MULTI-SOURCE               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │  Primary A   │         │  Primary B   │                 │
│  │ (Sales DB)   │         │  (HR DB)     │                 │
│  │              │         │              │                 │
│  │ server-id: 1 │         │ server-id: 2 │                 │
│  │ domain: 0    │         │ domain: 1    │                 │
│  └──────┬───────┘         └──────┬───────┘                 │
│         │                        │                         │
│         │ Channel 'sales'        │ Channel 'hr'            │
│         │                        │                         │
│         └────────────┬───────────┘                         │
│                      │                                     │
│                      ▼                                     │
│         ┌────────────────────────┐                         │
│         │  Multi-Source Replica  │                         │
│         │                        │                         │
│         │  server-id: 10         │                         │
│         │                        │                         │
│         │  ┌──────────────────┐  │                         │
│         │  │ Channel 'sales'  │  │                         │
│         │  │ ├─ IO Thread     │  │                         │
│         │  │ ├─ SQL Thread    │  │                         │
│         │  │ └─ Relay Log     │  │                         │
│         │  └──────────────────┘  │                         │
│         │                        │                         │
│         │  ┌──────────────────┐  │                         │
│         │  │ Channel 'hr'     │  │                         │
│         │  │ ├─ IO Thread     │  │                         │
│         │  │ ├─ SQL Thread    │  │                         │
│         │  │ └─ Relay Log     │  │                         │
│         │  └──────────────────┘  │                         │
│         │                        │                         │
│         │  Databases:            │                         │
│         │  ├─ sales (from A)     │                         │
│         │  └─ hr (from B)        │                         │
│         └────────────────────────┘                         │
│                      │                                     │
│                      ▼                                     │
│            Unified Analytics/Reporting                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Canaux de réplication

Chaque connexion Primary → Replica est un **canal nommé** :

```sql
-- Canal 'sales'
├─ Nom: 'sales'
├─ Primary: primary-sales.example.com
├─ Relay Log: relay-sales.*
├─ Position: GTID domain 0 ou (file, pos)
└─ Threads: IO + SQL dédiés

-- Canal 'hr'
├─ Nom: 'hr'
├─ Primary: primary-hr.example.com
├─ Relay Log: relay-hr.*
├─ Position: GTID domain 1 ou (file, pos)
└─ Threads: IO + SQL dédiés
```

**Isolation** :
- Chaque canal a ses propres threads I/O et SQL
- Chaque canal a son propre relay log
- Erreur sur un canal n'affecte pas les autres
- Performance indépendante par canal

---

## Configuration Multi-Source

### Prérequis

**Sur chaque Primary** :

```ini
# Primary A (Sales)
[mysqld]
server-id = 1
log-bin = mariadb-bin
binlog_format = ROW
gtid_domain_id = 0    # Domain unique

# Primary B (HR)
[mysqld]
server-id = 2
log-bin = mariadb-bin
binlog_format = ROW
gtid_domain_id = 1    # Domain différent !
```

**Sur le Multi-Source Replica** :

```ini
[mysqld]
server-id = 10           # Unique dans toute la topologie
relay-log = relay
log-slave-updates = ON   # Si le Replica doit aussi être source
read-only = ON           # Protection
```

### Configuration étape par étape

**Étape 1 : Créer les utilisateurs de réplication**

```sql
-- Sur Primary A (Sales)
CREATE USER 'repl_sales'@'%' 
  IDENTIFIED BY 'SecureP@ss_Sales_2025!';
  
GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_sales'@'%';

-- Sur Primary B (HR)
CREATE USER 'repl_hr'@'%' 
  IDENTIFIED BY 'SecureP@ss_HR_2025!';
  
GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_hr'@'%';
```

**Étape 2 : Initialiser les données (dump ou backup)**

```bash
# Dump depuis Primary A
mysqldump -u root -p \
  --databases sales \
  --single-transaction \
  --master-data=2 \
  --routines \
  --triggers \
  > sales_dump.sql

# Dump depuis Primary B
mysqldump -u root -p \
  --databases hr \
  --single-transaction \
  --master-data=2 \
  --routines \
  --triggers \
  > hr_dump.sql

# Sur le Replica : Restaurer
mariadb -u root -p < sales_dump.sql
mariadb -u root -p < hr_dump.sql
```

**Étape 3 : Configurer les canaux de réplication**

**Avec GTID (recommandé)** :

```sql
-- Sur le Multi-Source Replica

-- Canal 'sales'
CHANGE MASTER 'sales' TO
  MASTER_HOST = 'primary-sales.example.com',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_sales',
  MASTER_PASSWORD = 'SecureP@ss_Sales_2025!',
  MASTER_USE_GTID = slave_pos;

-- Canal 'hr'
CHANGE MASTER 'hr' TO
  MASTER_HOST = 'primary-hr.example.com',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_hr',
  MASTER_PASSWORD = 'SecureP@ss_HR_2025!',
  MASTER_USE_GTID = slave_pos;
```

**Avec positions binlog** :

```sql
-- Extraire positions des dumps
grep "CHANGE MASTER" sales_dump.sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000042', MASTER_LOG_POS=4567;

grep "CHANGE MASTER" hr_dump.sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000123', MASTER_LOG_POS=9876;

-- Canal 'sales'
CHANGE MASTER 'sales' TO
  MASTER_HOST = 'primary-sales.example.com',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_sales',
  MASTER_PASSWORD = 'SecureP@ss_Sales_2025!',
  MASTER_LOG_FILE = 'mariadb-bin.000042',
  MASTER_LOG_POS = 4567;

-- Canal 'hr'
CHANGE MASTER 'hr' TO
  MASTER_HOST = 'primary-hr.example.com',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_hr',
  MASTER_PASSWORD = 'SecureP@ss_HR_2025!',
  MASTER_LOG_FILE = 'mariadb-bin.000123',
  MASTER_LOG_POS = 9876;
```

**Étape 4 : Démarrer les canaux**

```sql
-- Démarrer tous les canaux
START ALL SLAVES;

-- Ou individuellement
START SLAVE 'sales';
START SLAVE 'hr';
```

**Étape 5 : Vérifier l'état**

```sql
-- Voir tous les canaux
SHOW ALL SLAVES STATUS\G

-- Ou un canal spécifique
SHOW SLAVE 'sales' STATUS\G
SHOW SLAVE 'hr' STATUS\G
```

### Configuration avancée avec SSL

```sql
-- Canal 'sales' avec SSL
CHANGE MASTER 'sales' TO
  MASTER_HOST = 'primary-sales.example.com',
  MASTER_USER = 'repl_sales',
  MASTER_PASSWORD = 'SecureP@ss_Sales_2025!',
  MASTER_USE_GTID = slave_pos,
  MASTER_SSL = 1,
  MASTER_SSL_CA = '/etc/mysql/ssl/ca-cert.pem',
  MASTER_SSL_CERT = '/etc/mysql/ssl/client-cert.pem',
  MASTER_SSL_KEY = '/etc/mysql/ssl/client-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1;
```

---

## GTID et Multi-Source

### Pourquoi GTID est crucial

**Avec positions binlog** : Gestion manuelle complexe

```
Primary A: mariadb-bin-A.000042:4567
Primary B: mariadb-bin-B.000123:9876

→ 2 jeux de positions à gérer
→ Failover très complexe
→ Erreurs fréquentes
```

**Avec GTID** : Gestion automatisée

```
Primary A: Domain 0
Primary B: Domain 1

Replica position:
- Domain 0: 0-1-1000
- Domain 1: 1-2-500

→ Gestion unifiée
→ Failover simplifié
→ Robustesse accrue
```

### Configuration GTID Domains

**Règle fondamentale** : **Un Domain ID unique par Primary source**

```sql
-- Primary A (Sales)
SET GLOBAL gtid_domain_id = 0;

-- Primary B (HR)
SET GLOBAL gtid_domain_id = 1;

-- Primary C (Finance)
SET GLOBAL gtid_domain_id = 2;

-- etc.
```

**Sur le Replica** :

```sql
-- Vérifier la position multi-domain
SELECT @@gtid_slave_pos;
-- +---------------------------+
-- | @@gtid_slave_pos          |
-- +---------------------------+
-- | 0-1-1000,1-2-500,2-3-750  |
-- +---------------------------+

-- Signifie:
-- Domain 0 (Sales): Appliqué jusqu'à 0-1-1000
-- Domain 1 (HR): Appliqué jusqu'à 1-2-500
-- Domain 2 (Finance): Appliqué jusqu'à 2-3-750
```

### Avantages GTID pour Multi-Source

**1. Ajout d'une nouvelle source simplifié**

```sql
-- Nouvelle source : Primary D (Inventory)
-- Aucune position manuelle à calculer !

CHANGE MASTER 'inventory' TO
  MASTER_HOST = 'primary-inventory.example.com',
  MASTER_USER = 'repl_inventory',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;  -- Auto-détection position

START SLAVE 'inventory';
-- ✅ Commence automatiquement depuis la bonne position
```

**2. Failover d'une source**

```sql
-- Primary A tombe, promotion de Replica A1 en Primary A'
-- Sur le Multi-Source Replica:

STOP SLAVE 'sales';

CHANGE MASTER 'sales' TO
  MASTER_HOST = 'replica-a1.example.com',  -- Nouveau Primary
  MASTER_USE_GTID = slave_pos;  -- Position recalculée auto !

START SLAVE 'sales';
-- ✅ Reprend exactement où il était
```

**3. Isolation des domains**

```sql
-- Problème sur le canal 'hr'
SHOW SLAVE 'hr' STATUS\G
-- Last_SQL_Error: Duplicate entry...

-- Arrêter seulement ce canal
STOP SLAVE 'hr';

-- Les autres canaux continuent !
SHOW SLAVE 'sales' STATUS\G
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- ✅ Pas d'impact
```

---

## Cas d'Usage

### 1. Consolidation pour Reporting/Analytics

**Scénario** : Application microservices avec bases séparées

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Service A    │  │ Service B    │  │ Service C    │
│ (Orders DB)  │  │ (Users DB)   │  │ (Inventory)  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       │                 │                 │
       └────────┬────────┴────────┬────────┘
                │                 │
                ▼                 ▼
       ┌──────────────────────────────┐
       │   Analytics Database         │
       │   (Multi-Source Replica)     │
       │                              │
       │   - orders (from Service A)  │
       │   - users (from Service B)   │
       │   - inventory (from C)       │
       │                              │
       │   JOIN possible !            │
       └──────────────────────────────┘
                    ▼
              BI Tools / Dashboards
```

**Avantages** :
- ✅ Requêtes cross-database possibles
- ✅ Zero impact sur les bases de production
- ✅ Scalabilité indépendante du reporting

**Configuration** :

```sql
CHANGE MASTER 'orders' TO ...;
CHANGE MASTER 'users' TO ...;
CHANGE MASTER 'inventory' TO ...;

-- Requête unifiée possible
SELECT 
  o.order_id,
  u.username,
  i.product_name
FROM orders.orders o
JOIN users.users u ON o.user_id = u.id
JOIN inventory.products i ON o.product_id = i.id
WHERE o.created_at >= CURDATE();
```

### 2. Migration de données

**Scénario** : Migration progressive de plusieurs anciennes bases vers une nouvelle

```
Old System A     Old System B     Old System C
(MySQL 5.7)      (MariaDB 10.6)   (PostgreSQL)
     │                │                │
     │                │                │ (via Foreign Data Wrapper)
     └────────────────┴────────────────┘
                      │
                      ▼
           ┌──────────────────┐
           │  New System      │
           │  (MariaDB 11.8)  │
           │                  │
           │  Consolidation   │
           │  + Transform     │
           └──────────────────┘
```

**Use case** :
- Migration legacy → modern
- Période de transition avec double run
- Validation progressive

### 3. Géo-distribution

**Scénario** : Consolidation de données régionales

```
EU Primary       US Primary       APAC Primary
(Europe data)    (US data)        (Asia data)
     │                │                │
     └────────────────┴────────────────┘
                      │
                      ▼
              Global Replica
           (Reporting mondial)
```

**Configuration** :

```sql
-- Données européennes
CHANGE MASTER 'eu' TO
  MASTER_HOST = 'primary-eu.example.com',
  MASTER_USE_GTID = slave_pos;

-- Données US
CHANGE MASTER 'us' TO
  MASTER_HOST = 'primary-us.example.com',
  MASTER_USE_GTID = slave_pos;

-- Données APAC
CHANGE MASTER 'apac' TO
  MASTER_HOST = 'primary-apac.example.com',
  MASTER_USE_GTID = slave_pos;
```

### 4. Backup centralisé

**Scénario** : Un serveur backup pour toutes les bases

```
Production A    Production B    Production C
     │               │               │
     └───────────────┴───────────────┘
                     │
                     ▼
              Backup Server
           (Multi-Source Replica)
                     │
                     ▼
         Mariabackup / Dumps
         (Backup consolidé)
```

**Avantages** :
- ✅ Un seul point de backup
- ✅ Backup sans impact sur production
- ✅ PITR unifié possible

---

## Gestion des Conflits

### Types de conflits potentiels

**1. Conflits de noms de base/table**

```
Problème:
Primary A a : database sales, table orders
Primary B a : database sales, table orders

→ CONFLIT sur Replica !
```

**Solution 1 : Replication filtering**

```sql
-- Canal 'sales-a'
CHANGE MASTER 'sales-a' TO
  MASTER_HOST = 'primary-a.example.com',
  MASTER_USE_GTID = slave_pos;

SET GLOBAL sales-a.replicate_do_db = 'sales_a';

-- Canal 'sales-b'
CHANGE MASTER 'sales-b' TO
  MASTER_HOST = 'primary-b.example.com',
  MASTER_USE_GTID = slave_pos;

SET GLOBAL sales-b.replicate_do_db = 'sales_b';
```

**Solution 2 : Rewrite rules**

```ini
[mysqld]
# Renommer les bases en réplication
replicate-rewrite-db = sales:sales_a  # Pour canal sales-a
replicate-rewrite-db = sales:sales_b  # Pour canal sales-b
```

**Solution 3 : Design initial (recommandé)**

```
Primary A : Bases sales_a, orders_a
Primary B : Bases sales_b, orders_b

→ Pas de conflit par design
```

**2. Conflits de clés primaires**

```
Scénario problématique:
Primary A insère : users (id=1, name='Alice')
Primary B insère : users (id=1, name='Bob')

→ ERREUR sur Replica !
-- Duplicate entry '1' for key 'PRIMARY'
```

**Solutions** :

**A. Auto-increment offset (prévention)**

```sql
-- Primary A
SET GLOBAL auto_increment_increment = 10;
SET GLOBAL auto_increment_offset = 1;
-- Génère: 1, 11, 21, 31, ...

-- Primary B
SET GLOBAL auto_increment_increment = 10;
SET GLOBAL auto_increment_offset = 2;
-- Génère: 2, 12, 22, 32, ...

-- Primary C
SET GLOBAL auto_increment_increment = 10;
SET GLOBAL auto_increment_offset = 3;
-- Génère: 3, 13, 23, 33, ...

→ Jamais de conflit
```

**B. UUID comme clé primaire**

```sql
-- Au lieu de INT AUTO_INCREMENT
CREATE TABLE users (
  id CHAR(36) PRIMARY KEY DEFAULT (UUID()),  -- UUID
  name VARCHAR(100)
);

-- Primary A génère: '550e8400-e29b-41d4-a716-446655440000'
-- Primary B génère: '7c9e6679-7425-40de-944b-e07fc1f90ae7'
→ Jamais de conflit
```

**C. Préfixe par source**

```sql
-- Primary A : Préfixe 'A-'
INSERT INTO users (id, name) VALUES ('A-001', 'Alice');

-- Primary B : Préfixe 'B-'
INSERT INTO users (id, name) VALUES ('B-001', 'Bob');

→ Jamais de conflit
```

**3. Conflits de données**

```
Primary A modifie : UPDATE users SET status='active' WHERE id=123;
Primary B modifie : UPDATE users SET status='deleted' WHERE id=123;

→ Dernière écriture gagne (last-write-wins)
→ Potentiel incohérence selon l'ordre d'application
```

**Solution** : 
- Design applicatif : éviter les écritures concurrentes
- Partitionnement des données par source
- Application-level conflict resolution

---

## Monitoring Multi-Source

### Commandes essentielles

**1. État de tous les canaux**

```sql
-- Vue globale
SHOW ALL SLAVES STATUS\G

-- Output:
*************************** 1. row ***************************
              Connection_name: sales
              Slave_IO_State: Waiting for master to send event
                 Master_Host: primary-sales.example.com
                 Master_User: repl_sales
                 Master_Port: 3306
               Connect_Retry: 60
             Master_Log_File: mariadb-bin.000042
         Read_Master_Log_Pos: 8934
              Relay_Log_File: relay-sales.000003
               Relay_Log_Pos: 12345
       Relay_Master_Log_File: mariadb-bin.000042
            Slave_IO_Running: Yes
           Slave_SQL_Running: Yes
         Seconds_Behind_Master: 0
                  Using_Gtid: Slave_Pos
                 Gtid_IO_Pos: 0-1-1000
*************************** 2. row ***************************
              Connection_name: hr
              Slave_IO_State: Waiting for master to send event
                 Master_Host: primary-hr.example.com
                 Master_User: repl_hr
                 Master_Port: 3306
               Connect_Retry: 60
             Master_Log_File: mariadb-bin.000123
         Read_Master_Log_Pos: 5678
              Relay_Log_File: relay-hr.000002
               Relay_Log_Pos: 9876
       Relay_Master_Log_File: mariadb-bin.000123
            Slave_IO_Running: Yes
           Slave_SQL_Running: Yes
         Seconds_Behind_Master: 2
                  Using_Gtid: Slave_Pos
                 Gtid_IO_Pos: 1-2-500
```

**2. État d'un canal spécifique**

```sql
SHOW SLAVE 'sales' STATUS\G
```

**3. Liste des canaux configurés**

```sql
SELECT 
  connection_name,
  master_host,
  master_port,
  master_user
FROM mysql.slave_connections;
-- +-----------------+------------------------+-------------+--------------+
-- | connection_name | master_host            | master_port | master_user  |
-- +-----------------+------------------------+-------------+--------------+
-- | sales           | primary-sales.ex...    | 3306        | repl_sales   |
-- | hr              | primary-hr.example.com | 3306        | repl_hr      |
-- +-----------------+------------------------+-------------+--------------+
```

### Requêtes de monitoring avancées

**1. Santé globale de tous les canaux**

```sql
SELECT 
  connection_name AS channel,
  master_host,
  CASE 
    WHEN slave_io_running = 'Yes' AND slave_sql_running = 'Yes' 
    THEN 'OK'
    ELSE 'ERROR'
  END AS status,
  seconds_behind_master AS lag_sec,
  last_io_error,
  last_sql_error
FROM information_schema.SLAVE_STATUS
ORDER BY connection_name;
```

**2. Lag agrégé**

```sql
SELECT 
  COUNT(*) AS total_channels,
  SUM(CASE WHEN slave_io_running = 'Yes' AND slave_sql_running = 'Yes' 
      THEN 1 ELSE 0 END) AS healthy_channels,
  MAX(seconds_behind_master) AS max_lag_sec,
  AVG(seconds_behind_master) AS avg_lag_sec
FROM information_schema.SLAVE_STATUS;
```

**3. GTID positions par domain**

```sql
SELECT 
  connection_name,
  gtid_io_pos
FROM information_schema.SLAVE_STATUS
ORDER BY connection_name;
-- +-----------------+---------------+
-- | connection_name | gtid_io_pos   |
-- +-----------------+---------------+
-- | sales           | 0-1-1000      |
-- | hr              | 1-2-500       |
-- | finance         | 2-3-750       |
-- +-----------------+---------------+
```

### Script de monitoring Bash

```bash
#!/bin/bash
# multi_source_health.sh

CHANNELS=$(mysql -N -e "
  SELECT connection_name 
  FROM mysql.slave_connections
")

echo "=== Multi-Source Replication Health Check ==="
echo "Time: $(date)"
echo ""

for CHANNEL in $CHANNELS; do
  echo "Channel: $CHANNEL"
  
  STATUS=$(mysql -N -e "
    SELECT 
      CASE 
        WHEN slave_io_running = 'Yes' AND slave_sql_running = 'Yes' 
        THEN 'RUNNING'
        ELSE 'STOPPED'
      END
    FROM information_schema.SLAVE_STATUS 
    WHERE connection_name = '$CHANNEL'
  ")
  
  LAG=$(mysql -N -e "
    SELECT IFNULL(seconds_behind_master, 'NULL')
    FROM information_schema.SLAVE_STATUS 
    WHERE connection_name = '$CHANNEL'
  ")
  
  echo "  Status: $STATUS"
  echo "  Lag: $LAG seconds"
  
  if [ "$STATUS" != "RUNNING" ]; then
    ERROR=$(mysql -N -e "
      SELECT CONCAT(
        'IO: ', IFNULL(last_io_error, 'None'), 
        ' | SQL: ', IFNULL(last_sql_error, 'None')
      )
      FROM information_schema.SLAVE_STATUS 
      WHERE connection_name = '$CHANNEL'
    ")
    echo "  ERROR: $ERROR"
  fi
  
  echo ""
done

# Exit code
ERROR_COUNT=$(mysql -N -e "
  SELECT COUNT(*) 
  FROM information_schema.SLAVE_STATUS 
  WHERE slave_io_running != 'Yes' OR slave_sql_running != 'Yes'
")

if [ "$ERROR_COUNT" -gt 0 ]; then
  echo "❌ $ERROR_COUNT channel(s) in error state"
  exit 2
else
  echo "✅ All channels healthy"
  exit 0
fi
```

### Intégration Prometheus

```yaml
# mysqld_exporter config for multi-source
queries:
  - name: mariadb_multisource_channel_status
    help: "Multi-source channel replication status (1=running, 0=stopped)"
    labels:
      - connection_name
      - master_host
    query: |
      SELECT 
        connection_name,
        master_host,
        CASE 
          WHEN slave_io_running = 'Yes' AND slave_sql_running = 'Yes' 
          THEN 1 
          ELSE 0 
        END AS status
      FROM information_schema.SLAVE_STATUS;
      
  - name: mariadb_multisource_lag_seconds
    help: "Multi-source replication lag in seconds"
    labels:
      - connection_name
    query: |
      SELECT 
        connection_name,
        IFNULL(seconds_behind_master, 0) AS lag_seconds
      FROM information_schema.SLAVE_STATUS;
```

---

## Opérations Courantes

### Arrêt et démarrage

```sql
-- Arrêter tous les canaux
STOP ALL SLAVES;

-- Démarrer tous les canaux
START ALL SLAVES;

-- Arrêter un canal spécifique
STOP SLAVE 'sales';

-- Démarrer un canal spécifique
START SLAVE 'sales';

-- Arrêter seulement IO thread d'un canal
STOP SLAVE 'sales' IO_THREAD;

-- Démarrer seulement SQL thread d'un canal
START SLAVE 'sales' SQL_THREAD;
```

### Reconfiguration d'un canal

```sql
-- Arrêter le canal
STOP SLAVE 'sales';

-- Modifier la configuration
CHANGE MASTER 'sales' TO
  MASTER_HOST = 'new-primary-sales.example.com',
  MASTER_USE_GTID = slave_pos;

-- Redémarrer
START SLAVE 'sales';
```

### Suppression d'un canal

```sql
-- Arrêter le canal
STOP SLAVE 'sales';

-- Supprimer complètement le canal
RESET SLAVE 'sales' ALL;

-- Vérifier
SHOW ALL SLAVES STATUS\G
-- Le canal 'sales' n'apparaît plus
```

### Ajout d'un nouveau canal

```sql
-- Dump depuis la nouvelle source
mysqldump -u root -p \
  --databases newdb \
  --single-transaction \
  --master-data=2 \
  > newdb_dump.sql

-- Restaurer sur Replica
mariadb -u root -p < newdb_dump.sql

-- Configurer le nouveau canal
CHANGE MASTER 'newdb' TO
  MASTER_HOST = 'primary-newdb.example.com',
  MASTER_USER = 'repl_newdb',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;

-- Démarrer
START SLAVE 'newdb';
```

---

## Troubleshooting Multi-Source

### Problème 1 : Un canal ne démarre pas

```sql
SHOW SLAVE 'sales' STATUS\G
-- Slave_IO_Running: No
-- Last_IO_Error: Error connecting to master 'repl_sales@primary-sales:3306'...
```

**Diagnostic** :

```bash
# Tester connectivité réseau
ping primary-sales.example.com
telnet primary-sales.example.com 3306

# Tester credentials
mysql -h primary-sales.example.com -u repl_sales -p

# Vérifier firewall
iptables -L -n | grep 3306
```

**Solution** :

```sql
-- Corriger la configuration
STOP SLAVE 'sales';

CHANGE MASTER 'sales' TO
  MASTER_HOST = 'correct-host.example.com',
  MASTER_USER = 'repl_sales',
  MASTER_PASSWORD = 'correct_password';

START SLAVE 'sales';
```

### Problème 2 : Lag élevé sur un canal spécifique

```sql
SHOW SLAVE 'hr' STATUS\G
-- Seconds_Behind_Master: 3600  (1 heure !)
```

**Diagnostic** :

```sql
-- Vérifier activité du SQL Thread
SHOW PROCESSLIST;
-- Si SQL Thread en 'Making temp file' → Grosse transaction

-- Vérifier le relay log
SELECT 
  relay_log_space / 1024 / 1024 AS relay_log_mb
FROM information_schema.SLAVE_STATUS
WHERE connection_name = 'hr';
-- Si très gros : Accumulation d'événements
```

**Solutions** :

```sql
-- 1. Paralléliser ce canal spécifique (si possible)
STOP SLAVE 'hr';

SET GLOBAL hr.slave_parallel_threads = 4;
SET GLOBAL hr.slave_parallel_mode = 'optimistic';

START SLAVE 'hr';

-- 2. Vérifier slow queries sur Replica
SELECT * FROM mysql.slow_log 
WHERE sql_text LIKE '%hr.%' 
ORDER BY query_time DESC 
LIMIT 10;
```

### Problème 3 : Conflit de clés entre canaux

```sql
SHOW SLAVE 'sales' STATUS\G
-- Last_SQL_Error: Error 'Duplicate entry '123' for key 'PRIMARY''...
-- Sur table: consolidated.orders
```

**Cause** : Deux canaux écrivent dans la même table avec mêmes clés.

**Solution immédiate** :

```sql
-- Identifier le conflit
SELECT * FROM consolidated.orders WHERE id = 123;

-- Décider quelle version garder
-- Option A : Garder version du canal 'sales', skip erreur
STOP SLAVE 'sales';
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE 'sales';

-- Option B : Supprimer doublon, puis continuer
DELETE FROM consolidated.orders WHERE id = 123 AND source = 'hr';
START SLAVE 'sales';
```

**Solution long terme** : Revoir l'architecture

```sql
-- Séparer les tables par source
CREATE TABLE orders_sales (...);  -- Depuis canal 'sales'
CREATE TABLE orders_hr (...);     -- Depuis canal 'hr'

-- Vue unifiée
CREATE VIEW orders AS
  SELECT *, 'sales' AS source FROM orders_sales
  UNION ALL
  SELECT *, 'hr' AS source FROM orders_hr;
```

### Problème 4 : Tous les canaux se bloquent

```sql
SHOW ALL SLAVES STATUS\G
-- Tous : Slave_SQL_Running: No
-- Last_SQL_Error: Lock wait timeout exceeded
```

**Cause** : Verrou global (FLUSH TABLES, DDL, etc.)

**Solution** :

```sql
-- Identifier le verrou
SHOW PROCESSLIST;
-- Chercher "Waiting for table metadata lock"

-- Tuer la session bloquante
KILL <thread_id>;

-- Redémarrer tous les canaux
START ALL SLAVES;
```

---

## Bonnes Pratiques

### 1. Design de la topologie

✅ **Un domain ID par source** :

```sql
-- Primary A
gtid_domain_id = 0

-- Primary B
gtid_domain_id = 1

-- etc.
```

✅ **Nommage cohérent des canaux** :

```
Convention: <source>-<type>
Exemples:
- 'sales-production'
- 'hr-production'
- 'analytics-staging'
```

✅ **Séparation des données** :

```
Éviter: sales.orders (depuis 2 sources)
Préférer: 
- sales_a.orders
- sales_b.orders
- Ou vue consolidée
```

### 2. Monitoring continu

✅ **Alertes Nagios/Icinga** :

```ini
# check_multisource_replication.sh
define service {
  service_description    Multi-Source Replication
  check_command          check_multisource_health
  check_interval         1  # minute
}
```

✅ **Dashboards Grafana** :

```
Panel 1: Status par canal (gauge)
Panel 2: Lag par canal (graph)
Panel 3: Throughput par canal (graph)
Panel 4: Erreurs par canal (logs)
```

### 3. Documentation

✅ **Documenter la topologie** :

```sql
CREATE TABLE replication_topology_multisource (
  channel_name VARCHAR(64) PRIMARY KEY,
  source_host VARCHAR(255),
  source_purpose VARCHAR(255),
  gtid_domain INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  notes TEXT
);

INSERT INTO replication_topology_multisource VALUES
('sales', 'primary-sales.example.com', 'Sales transactions', 0, NOW(), 
 'Critical: 24/7 monitoring required'),
('hr', 'primary-hr.example.com', 'HR data', 1, NOW(), 
 'Low priority: can tolerate lag'),
('analytics', 'primary-analytics.example.com', 'Analytics events', 2, NOW(), 
 'High volume: monitor lag closely');
```

### 4. Tests réguliers

✅ **Tester le failover de chaque source** :

```bash
#!/bin/bash
# test_failover_source.sh <channel>

CHANNEL=$1

echo "Testing failover for channel: $CHANNEL"

# Simuler panne Primary
echo "1. Stopping primary..."
ssh primary-$CHANNEL "systemctl stop mariadb"

# Vérifier erreur sur Replica
sleep 5
mysql -e "SHOW SLAVE '$CHANNEL' STATUS\G" | grep "Last_IO_Error"

# Promouvoir Replica de ce Primary
echo "2. Promoting replica..."
# ...

# Reconfigurer le canal
echo "3. Reconfiguring channel..."
mysql -e "
  STOP SLAVE '$CHANNEL';
  CHANGE MASTER '$CHANNEL' TO 
    MASTER_HOST = 'new-primary-$CHANNEL.example.com',
    MASTER_USE_GTID = slave_pos;
  START SLAVE '$CHANNEL';
"

# Vérifier
mysql -e "SHOW SLAVE '$CHANNEL' STATUS\G"
```

### 5. Performance

✅ **Parallélisation par canal** :

```sql
-- Canal à haute volumétrie
SET GLOBAL sales.slave_parallel_threads = 8;

-- Canal à faible volumétrie
SET GLOBAL hr.slave_parallel_threads = 2;
```

✅ **Compression si WAN** :

```sql
CHANGE MASTER 'remote-site' TO
  MASTER_HOST = 'remote.example.com',
  MASTER_USE_GTID = slave_pos,
  MASTER_COMPRESSION_ALGORITHMS = 'zstd';
```

---

## ✅ Points clés à retenir

1. **Multi-source** : Un Replica réplique depuis plusieurs Primary via des canaux nommés

2. **Canaux indépendants** : Chaque canal a ses threads I/O/SQL et relay logs propres

3. **GTID crucial** : Un domain ID unique par source, gestion automatisée simplifiée

4. **Use cases** : Consolidation reporting, migration, géo-distribution, backup centralisé

5. **Conflits** : Éviter par design (noms uniques, auto-increment offset, UUID)

6. **Monitoring** : `SHOW ALL SLAVES STATUS`, requêtes agrégées, alerting par canal

7. **Opérations** : `CHANGE MASTER 'canal' TO`, `START/STOP SLAVE 'canal'`

8. **Isolation** : Erreur sur un canal n'affecte pas les autres

9. **Performance** : Parallélisation configurable par canal

10. **Production-ready** : MariaDB 11.8 LTS, architecture mature et stable

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Multi-Source Replication](https://mariadb.com/kb/en/multi-source-replication/)
- [📖 CHANGE MASTER TO](https://mariadb.com/kb/en/change-master-to/)
- [📖 SHOW ALL SLAVES STATUS](https://mariadb.com/kb/en/show-all-slaves-status/)
- [📖 Replication Filters](https://mariadb.com/kb/en/replication-filters/)

### Articles techniques

- [🔗 Multi-Source Replication Use Cases](https://mariadb.com/resources/blog/multi-source-replication-use-cases/)
- [🔗 GTID with Multi-Source](https://mariadb.com/kb/en/gtid/#using-gtid-with-multi-source-replication)

### Outils

- **Orchestrator** : Supporte multi-source pour topologies complexes
- **pt-table-checksum** : Vérification cohérence multi-source

---

## ➡️ Section suivante

**13.6 Réplication en cascade** : Chaîner les serveurs de réplication pour distribuer la charge, architectures multi-niveaux, configuration `log_slave_updates`, et gestion du lag cumulatif.

---


⏭️ [Réplication en cascade](/13-replication/06-replication-cascade.md)

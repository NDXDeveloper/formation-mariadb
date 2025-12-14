🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.2 - Informations Système

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 45 minutes  
> **Prérequis** : Section B.1 - Connexion et Navigation

---

## 📖 Introduction

Cette section couvre les **commandes de monitoring et diagnostic** du serveur MariaDB. Ces commandes permettent d'obtenir des informations critiques sur l'état du serveur, les processus actifs, les variables de configuration et les statistiques de performance.

### 🎯 Objectifs

À l'issue de cette section, vous serez capable de :
- Obtenir le statut complet du serveur
- Surveiller les connexions et processus actifs
- Analyser les moteurs de stockage
- Consulter et modifier les variables système
- Examiner les statistiques de performance
- Diagnostiquer les problèmes courants

---

## 📊 STATUS - Statut du Serveur

### Commande de Base

```sql
-- Afficher le statut complet
STATUS;

-- ou méta-commande équivalente
\s
```

### Sortie Détaillée

```
--------------
mariadb  Ver 15.1 Distrib 11.8.0-MariaDB, for Linux (x86_64)

Connection id:          847
Current database:       production_db
Current user:           admin@localhost
SSL:                    Cipher in use is TLS_AES_256_GCM_SHA384
Current pager:          less
Using outfile:          ''
Using delimiter:        ;
Server:                 MariaDB
Server version:         11.8.0-MariaDB MariaDB Server
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8mb4
Conn.  characterset:    utf8mb4
UNIX socket:            /var/run/mysqld/mysqld.sock
Binary data as:         Hexadecimal
Uptime:                 15 days 7 hours 23 min 42 sec

Threads: 24  Questions: 8547234  Slow queries: 156  Opens: 1234
Flush tables: 8  Open tables: 456  Queries per second avg: 6.483
--------------
```

### Analyse des Informations Clés

| Information | Description | Utilité |
|-------------|-------------|---------|
| **Connection id** | ID unique de session | Identifier votre connexion dans PROCESSLIST |
| **Current database** | Base active | Vérifier contexte actuel |
| **Current user** | User@host | Vérifier privilèges |
| **SSL** | Chiffrement actif | Sécurité connexion |
| **Server version** | Version exacte MariaDB | Compatibilité features |
| **Connection type** | TCP/Socket/Pipe | Type de connexion |
| **Characterset** | Encodage utilisé | Éviter problèmes d'encodage |
| **Uptime** | Temps depuis démarrage | Stabilité serveur |
| **Threads** | Connexions actives | Charge serveur |
| **Questions** | Requêtes totales | Activité globale |
| **Slow queries** | Requêtes lentes | Performance indicator |
| **QPS** | Queries/seconde | Throughput |

### Interpréter le Statut

```sql
-- Vérifier si SSL est actif
\s
-- Chercher ligne : SSL: Cipher in use is ...
-- Si pas de SSL : SSL: Not in use

-- Vérifier l'uptime (redémarrage récent ?)
-- Uptime court peut indiquer crash ou redémarrage

-- Vérifier QPS (activité)
-- QPS élevé = serveur chargé
-- QPS faible = serveur peu utilisé ou problème

-- Vérifier slow queries
-- Nombre élevé = problèmes de performance
```

---

## 🔍 SHOW PROCESSLIST - Connexions Actives

### Commande de Base

```sql
-- Liste complète des processus
SHOW PROCESSLIST;

-- Liste complète (non tronquée)
SHOW FULL PROCESSLIST;
```

### Sortie Exemple

```
+-----+------+-----------+------+---------+------+--------------+------------------+----------+
| Id  | User | Host      | db   | Command | Time | State        | Info             | Progress |
+-----+------+-----------+------+---------+------+--------------+------------------+----------+
| 847 | admin| localhost | prod | Sleep   |   42 |              | NULL             |    0.000 |
| 848 | web  | 10.0.1.5  | prod | Query   |    2 | Sending data | SELECT * FROM... |   45.234 |
| 849 | batch| localhost | prod | Query   |  156 | Locked       | UPDATE orders... |    0.000 |
| 850 | ro   | 10.0.1.8  | prod | Query   |    0 | init         | SHOW PROCESSLIST |    0.000 |
+-----+------+-----------+------+---------+------+--------------+------------------+----------+
```

### Colonnes Expliquées

| Colonne | Description | Valeurs Courantes |
|---------|-------------|-------------------|
| **Id** | ID du processus | Numéro unique |
| **User** | Utilisateur | Nom user MySQL |
| **Host** | Client hostname:port | localhost, IP:port |
| **db** | Base de données active | Nom DB ou NULL |
| **Command** | Type de commande | Query, Sleep, Connect, etc. |
| **Time** | Durée en secondes | Temps depuis début commande |
| **State** | État actuel | Sending data, Locked, Sorting, etc. |
| **Info** | Requête SQL | Texte de la requête (tronqué) |
| **Progress** | Progression | Pourcentage (ALTER TABLE, etc.) |

### États Courants (State)

| State | Signification | Action |
|-------|---------------|--------|
| **Sending data** | Traite et envoie résultats | Normal, vérifier si dure longtemps |
| **Locked** | Attend verrou | Vérifier deadlocks/locks |
| **Sorting result** | Trie résultats | Peut nécessiter index |
| **Creating tmp table** | Table temporaire | Optimisation possible |
| **Copying to tmp table** | Copie vers tmp | ALTER TABLE, GROUP BY |
| **Sleep** | Connexion inactive | Normal pour pool connexions |
| **Killed** | Processus tué | En cours d'annulation |
| **Waiting for table** | Attend verrou table | Table lock contention |

### Filtrer les Processus

```sql
-- Via INFORMATION_SCHEMA (plus flexible)
SELECT 
  ID,
  USER,
  HOST,
  DB,
  COMMAND,
  TIME,
  STATE,
  LEFT(INFO, 50) AS QUERY
FROM information_schema.PROCESSLIST
ORDER BY TIME DESC;

-- Processus actifs uniquement (pas Sleep)
SELECT *
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC;

-- Requêtes longues (> 60 secondes)
SELECT 
  ID,
  USER,
  DB,
  TIME,
  STATE,
  INFO
FROM information_schema.PROCESSLIST
WHERE TIME > 60
  AND COMMAND != 'Sleep'
ORDER BY TIME DESC;

-- Par utilisateur
SELECT 
  USER,
  COUNT(*) AS connections,
  SUM(TIME) AS total_time
FROM information_schema.PROCESSLIST
GROUP BY USER
ORDER BY connections DESC;

-- Par database
SELECT 
  DB,
  COUNT(*) AS connections,
  AVG(TIME) AS avg_time
FROM information_schema.PROCESSLIST
WHERE DB IS NOT NULL
GROUP BY DB;
```

### Tuer un Processus

```sql
-- Identifier le processus problématique
SHOW PROCESSLIST;

-- Tuer un processus spécifique
KILL 848;

-- Tuer la requête sans déconnecter
KILL QUERY 848;

-- Vérifier que le processus est terminé
SHOW PROCESSLIST;
```

⚠️ **Attention** : 
- `KILL` termine la connexion complètement
- `KILL QUERY` arrête uniquement la requête en cours
- Nécessite privilège SUPER ou PROCESS

### Surveillance en Temps Réel

```bash
# Surveiller processus en continu (refresh toutes les 2s)
watch -n 2 "mariadb -u root -p -e 'SHOW PROCESSLIST'"

# Ou avec mysql client
mariadb -u root -p -e "SHOW PROCESSLIST\G" | less

# Script de monitoring
while true; do
  clear
  mariadb -u root -p -e "
    SELECT ID, USER, DB, TIME, STATE, LEFT(INFO,50) 
    FROM information_schema.PROCESSLIST 
    WHERE COMMAND != 'Sleep' 
    ORDER BY TIME DESC"
  sleep 5
done
```

---

## 🔧 SHOW ENGINE - Moteurs de Stockage

### Lister les Moteurs Disponibles

```sql
-- Tous les moteurs disponibles
SHOW ENGINES;
```

**Sortie exemple** :
```
+--------------------+---------+----------------------------------------------------------------+
| Engine             | Support | Comment                                                        |
+--------------------+---------+----------------------------------------------------------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     |
| CSV                | YES     | CSV storage engine                                             |
| MRG_MyISAM         | YES     | Collection of identical MyISAM tables                          |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      |
| Aria               | YES     | Crash-safe tables with MyISAM heritage                         |
| MyISAM             | YES     | Non-transactional engine with good performance and small size  |
| SEQUENCE           | YES     | Generated tables filled with sequential values                 |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             |
| ColumnStore        | YES     | ColumnStore storage engine                                     |
+--------------------+---------+----------------------------------------------------------------+
```

### Colonnes Expliquées

| Colonne | Description | Valeurs |
|---------|-------------|---------|
| **Engine** | Nom du moteur | InnoDB, MyISAM, Aria, etc. |
| **Support** | Niveau de support | DEFAULT, YES, NO, DISABLED |
| **Comment** | Description | Caractéristiques principales |

### Statut InnoDB Détaillé

```sql
-- Statut complet InnoDB
SHOW ENGINE INNODB STATUS\G
```

**Sections principales** :

#### 1. Header - Informations Générales
```
=====================================
2025-12-14 15:30:45 0x7f8b4c001700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 60 seconds
```

#### 2. Background Thread
```
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 245678 srv_active, 0 srv_shutdown, 125689 srv_idle
srv_master_thread log flush and writes: 371367
```

#### 3. Semaphores - Verrous et Attentes
```
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 12345
OS WAIT ARRAY INFO: signal count 98765
RW-shared spins 54321, rounds 76543, OS waits 2345
RW-excl spins 12345, rounds 23456, OS waits 1234
```

💡 **Indicateurs de problème** :
- Nombreux OS waits → contention élevée
- Rounds élevés → lock contention

#### 4. Transactions Actives
```
------------
TRANSACTIONS
------------
Trx id counter 2847563
Purge done for trx's n:o < 2847550 undo n:o < 0 state: running
History list length 156

---TRANSACTION 2847562, ACTIVE 42 sec
mysql tables in use 2, locked 2
45 lock struct(s), heap size 8400, 234 row lock(s)
MySQL thread id 848, OS thread handle 140243567294208, query id 8547234 localhost admin
Sending data
SELECT o.*, c.name FROM orders o JOIN customers c ON o.customer_id = c.id WHERE ...
```

💡 **À surveiller** :
- **History list length** : > 1000 = purge lent
- **ACTIVE transactions** : durée excessive
- **lock struct(s)** : nombreux verrous
- **row lock(s)** : locks sur beaucoup de lignes

#### 5. File I/O
```
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (write thread)
Pending normal aio reads: 0 [0, 0, 0, 0] , aio writes: 0 [0, 0, 0, 0] ,
 ibuf aio reads: 0, log i/o's: 0, sync i/o's: 0
Pending flushes (fsync) log: 0; buffer pool: 0
4523678 OS file reads, 8976543 OS file writes, 3456789 OS fsyncs
```

💡 **Métriques clés** :
- **Pending aio** : > 0 de façon persistante = I/O saturé
- **Pending fsyncs** : élevé = disque lent

#### 6. Buffer Pool et Memory
```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 8589934592
Dictionary memory allocated 456789
Buffer pool size   524288
Free buffers       12345
Database pages     509876
Old database pages 188234
Modified db pages  2345
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 8976543, not young 4567890
0.12 youngs/s, 0.08 non-youngs/s
Pages read 4523678, created 123456, written 2345678
0.45 reads/s, 0.12 creates/s, 0.23 writes/s
Buffer pool hit rate 998 / 1000
```

💡 **Indicateurs importants** :
- **Buffer pool hit rate** : doit être > 99% (ici 99.8%)
- **Modified db pages** : élevé = beaucoup d'écritures en attente
- **Free buffers** : faible = buffer pool plein (normal)

#### 7. Row Operations
```
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=12345, Main thread ID=140243567294208, state: sleeping
Number of rows inserted 8976543, updated 4567890, deleted 234567, read 98765432
0.45 inserts/s, 0.23 updates/s, 0.01 deletes/s, 4.56 reads/s
```

💡 **Métriques de throughput** :
- Inserts/updates/deletes/reads par seconde
- queries in queue > 0 = saturation

### Extraire Informations Spécifiques

```sql
-- Buffer pool hit rate
-- Extraire manuellement de SHOW ENGINE INNODB STATUS
-- Ou via variables :
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- Résultat :
+---------------------------------------+-------------+
| Variable_name                         | Value       |
+---------------------------------------+-------------+
| Innodb_buffer_pool_read_requests      | 98765432100 |
| Innodb_buffer_pool_reads              |     1234567 |
+---------------------------------------+-------------+

-- Calcul hit rate :
-- Hit rate = (read_requests - reads) / read_requests * 100
-- = (98765432100 - 1234567) / 98765432100 * 100 ≈ 99.999%
```

### Autres Moteurs

```sql
-- Statut Aria
SHOW ENGINE ARIA STATUS\G

-- Statut ColumnStore (si installé)
SHOW ENGINE COLUMNSTORE STATUS\G

-- Statut Performance Schema
SHOW ENGINE PERFORMANCE_SCHEMA STATUS\G
```

---

## ⚙️ SHOW VARIABLES - Configuration Serveur

### Lister Toutes les Variables

```sql
-- Toutes les variables système
SHOW VARIABLES;

-- Avec format vertical (plus lisible)
SHOW VARIABLES\G

-- Compter les variables
SELECT COUNT(*) FROM information_schema.GLOBAL_VARIABLES;
-- Résultat : ~600 variables
```

### Rechercher des Variables

```sql
-- Variables InnoDB
SHOW VARIABLES LIKE 'innodb%';

-- Variables Buffer Pool
SHOW VARIABLES LIKE 'innodb_buffer_pool%';

-- Variables charset
SHOW VARIABLES LIKE '%character%';
SHOW VARIABLES LIKE '%collation%';

-- Variables réplication
SHOW VARIABLES LIKE '%binlog%';
SHOW VARIABLES LIKE '%gtid%';
```

### Variables Critiques à Surveiller

#### Mémoire

```sql
-- Buffer Pool (mémoire InnoDB la plus importante)
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
-- Recommandation : 70-80% RAM sur serveur dédié

-- Buffer Pool instances
SHOW VARIABLES LIKE 'innodb_buffer_pool_instances';
-- Recommandation : 1 instance par GB (max 64)

-- Max connexions
SHOW VARIABLES LIKE 'max_connections';
-- Par défaut : 151, ajuster selon besoin

-- Table cache
SHOW VARIABLES LIKE 'table_open_cache';
-- Augmenter si nombreuses tables

-- Thread cache
SHOW VARIABLES LIKE 'thread_cache_size';
-- Réduire création/destruction threads
```

#### Performance

```sql
-- Query cache (déprécié)
SHOW VARIABLES LIKE 'query_cache%';
-- Devrait être OFF sur versions récentes

-- I/O capacity
SHOW VARIABLES LIKE 'innodb_io_capacity%';
-- Ajuster selon type disque (SSD vs HDD)

-- Flush method
SHOW VARIABLES LIKE 'innodb_flush_method';
-- Recommandation : O_DIRECT sur Linux

-- Log file size
SHOW VARIABLES LIKE 'innodb_log_file_size';
-- Augmenter pour workloads écriture intensive
```

#### Sécurité

```sql
-- SSL/TLS
SHOW VARIABLES LIKE '%ssl%';

-- Validation plugin
SHOW VARIABLES LIKE 'validate_password%';

-- Secure file privileges
SHOW VARIABLES LIKE 'secure_file_priv';
-- Limite LOAD DATA et SELECT INTO OUTFILE
```

#### Réplication

```sql
-- Binary log
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';

-- GTID
SHOW VARIABLES LIKE 'gtid%';

-- Server ID
SHOW VARIABLES LIKE 'server_id';
```

### Via INFORMATION_SCHEMA

```sql
-- Variables avec valeurs et contexte
SELECT 
  VARIABLE_NAME,
  VARIABLE_VALUE,
  DEFAULT_VALUE,
  VARIABLE_SCOPE
FROM information_schema.SYSTEM_VARIABLES
WHERE VARIABLE_NAME LIKE 'innodb_buffer_pool%'
ORDER BY VARIABLE_NAME;

-- Variables modifiables dynamiquement
SELECT VARIABLE_NAME
FROM information_schema.SYSTEM_VARIABLES
WHERE VARIABLE_SCOPE IN ('GLOBAL', 'SESSION')
  AND VARIABLE_NAME LIKE 'innodb%'
ORDER BY VARIABLE_NAME;
```

### Modifier les Variables

```sql
-- Variable SESSION (session courante uniquement)
SET SESSION sql_mode = 'STRICT_ALL_TABLES';
SET @@session.autocommit = 0;

-- Variable GLOBAL (toutes nouvelles connexions)
SET GLOBAL max_connections = 500;
SET @@global.slow_query_log = 1;

-- Vérifier modification
SHOW VARIABLES LIKE 'max_connections';
SHOW GLOBAL VARIABLES LIKE 'slow_query_log';
```

⚠️ **Important** :
- Modifications GLOBAL ne persistent pas après redémarrage
- Pour persistance : modifier `/etc/mysql/my.cnf`
- Certaines variables nécessitent redémarrage (statiques)

### Variables Statiques vs Dynamiques

```sql
-- Identifier variables dynamiques
SELECT 
  VARIABLE_NAME,
  VARIABLE_SCOPE
FROM information_schema.SYSTEM_VARIABLES
WHERE VARIABLE_NAME LIKE 'innodb%'
  AND VARIABLE_SCOPE IN ('GLOBAL', 'SESSION')
LIMIT 20;

-- Variables statiques (nécessitent redémarrage)
-- Exemples : innodb_buffer_pool_size (avant MariaDB 10.2.2),
--            innodb_log_file_size, datadir, etc.
```

---

## 📈 SHOW STATUS - Statistiques Serveur

### Lister Tous les Statuts

```sql
-- Tous les compteurs
SHOW STATUS;

-- Format vertical
SHOW STATUS\G

-- Compter
SELECT COUNT(*) FROM information_schema.GLOBAL_STATUS;
-- Résultat : ~400-500 statuts
```

### Statuts par Catégorie

#### Activité Générale

```sql
-- Requêtes totales
SHOW STATUS LIKE 'Questions';
SHOW STATUS LIKE 'Queries';

-- Connexions
SHOW STATUS LIKE 'Connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';
SHOW STATUS LIKE 'Max_used_connections';

-- Uptime
SHOW STATUS LIKE 'Uptime';

-- Calcul QPS (Questions Per Second)
-- QPS = Questions / Uptime
```

#### InnoDB

```sql
-- Buffer Pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Métriques importantes :
SHOW STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW STATUS LIKE 'Innodb_buffer_pool_reads';
-- Hit rate = (requests - reads) / requests

-- Données écrites/lues
SHOW STATUS LIKE 'Innodb_data_read';
SHOW STATUS LIKE 'Innodb_data_written';

-- Row operations
SHOW STATUS LIKE 'Innodb_rows%';
```

#### Locks et Verrous

```sql
-- Table locks
SHOW STATUS LIKE 'Table_locks%';

-- Row locks InnoDB
SHOW STATUS LIKE 'Innodb_row_lock%';
```

#### Slow Queries

```sql
-- Requêtes lentes
SHOW STATUS LIKE 'Slow_queries';

-- Comparer avec total
SHOW STATUS LIKE 'Questions';
-- % slow = (Slow_queries / Questions) * 100
```

#### Threads

```sql
-- État des threads
SHOW STATUS LIKE 'Threads%';

+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 8     |
| Threads_connected | 24    |
| Threads_created   | 156   |
| Threads_running   | 3     |
+-------------------+-------+

-- Threads_connected : connexions actives
-- Threads_running : threads exécutant requêtes
-- Threads_cached : threads en cache
-- Threads_created : total threads créés
```

### Analyser les Performances

```sql
-- Buffer pool hit rate
SELECT 
  VARIABLE_VALUE AS read_requests
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
UNION ALL
SELECT 
  VARIABLE_VALUE AS reads
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads';

-- Ratio slow queries
SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Slow_queries') /
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Questions') * 100 AS slow_query_percentage;

-- QPS actuel
SELECT 
  VARIABLE_VALUE / 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Uptime') AS qps
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Questions';
```

### Comparer Statuts (Delta)

```bash
# Script pour calculer différence sur intervalle
mariadb -u root -p -e "SHOW STATUS LIKE 'Questions'" > /tmp/before.txt
sleep 60
mariadb -u root -p -e "SHOW STATUS LIKE 'Questions'" > /tmp/after.txt
# Comparer manuellement
```

Ou en SQL :

```sql
-- Créer table temporaire pour snapshot
CREATE TEMPORARY TABLE status_snapshot (
  variable_name VARCHAR(64),
  value BIGINT,
  captured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Snapshot initial
INSERT INTO status_snapshot (variable_name, value)
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN ('Questions', 'Connections', 'Slow_queries');

-- Attendre 60 secondes...
-- DO SLEEP(60);

-- Comparer
SELECT 
  s.variable_name,
  s.value AS before_value,
  g.VARIABLE_VALUE AS after_value,
  (g.VARIABLE_VALUE - s.value) AS delta,
  (g.VARIABLE_VALUE - s.value) / 
    TIMESTAMPDIFF(SECOND, s.captured_at, NOW()) AS per_second
FROM status_snapshot s
JOIN information_schema.GLOBAL_STATUS g 
  ON s.variable_name = g.VARIABLE_NAME;
```

---

## 🎯 Scénarios de Diagnostic

### Scénario 1 : Serveur Lent

```sql
-- 1. Vérifier connexions actives
SHOW PROCESSLIST;

-- 2. Identifier requêtes longues
SELECT ID, USER, DB, TIME, STATE, INFO
FROM information_schema.PROCESSLIST
WHERE TIME > 30 AND COMMAND != 'Sleep'
ORDER BY TIME DESC;

-- 3. Vérifier slow queries
SHOW STATUS LIKE 'Slow_queries';

-- 4. Buffer pool hit rate
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- 5. InnoDB status
SHOW ENGINE INNODB STATUS\G
-- Chercher : lock waits, pending I/O, history list length
```

### Scénario 2 : Problème Mémoire

```sql
-- 1. Configuration mémoire
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'max_connections';

-- 2. Utilisation actuelle
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- 3. Vérifier si swap utilisé (via système)
\! free -h
\! top -bn1 | grep -i maria
```

### Scénario 3 : Lock Contention

```sql
-- 1. Transactions actives longues
SELECT *
FROM information_schema.PROCESSLIST
WHERE STATE LIKE '%lock%'
   OR STATE LIKE '%wait%'
ORDER BY TIME DESC;

-- 2. InnoDB lock waits
SHOW ENGINE INNODB STATUS\G
-- Section TRANSACTIONS

-- 3. Statuts locks
SHOW STATUS LIKE '%lock%';

-- 4. Identifier transaction bloquante
SELECT 
  r.trx_id waiting_trx_id,
  r.trx_mysql_thread_id waiting_thread,
  r.trx_query waiting_query,
  b.trx_id blocking_trx_id,
  b.trx_mysql_thread_id blocking_thread,
  b.trx_query blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

### Scénario 4 : Réplication Lag

```sql
-- Sur le replica :
SHOW SLAVE STATUS\G

-- Chercher :
-- Seconds_Behind_Master : lag en secondes
-- Slave_IO_Running : YES
-- Slave_SQL_Running : YES

-- Sur le master :
SHOW MASTER STATUS;

-- Position binlog et GTID
```

### Scénario 5 : Problèmes I/O

```sql
-- 1. Pending I/O
SHOW ENGINE INNODB STATUS\G
-- Section FILE I/O

-- 2. Configuration I/O
SHOW VARIABLES LIKE 'innodb_io_capacity%';
SHOW VARIABLES LIKE 'innodb_flush_method';

-- 3. Statistiques I/O
SHOW STATUS LIKE 'Innodb_data%';

-- 4. Système (via shell)
\! iostat -x 5 3
```

---

## 💡 Outils et Scripts Utiles

### Script de Monitoring Complet

```bash
#!/bin/bash
# monitoring.sh - Snapshot état serveur MariaDB

echo "=== MariaDB Health Check ==="
echo "Date: $(date)"
echo

mariadb -u root -p <<EOF
-- Status général
SELECT 'SERVER STATUS' as '';
SELECT VERSION() as version, 
       @@hostname as hostname,
       @@port as port;

-- Uptime et connexions
SELECT 'UPTIME & CONNECTIONS' as '';
SELECT 
  VARIABLE_VALUE as uptime_seconds,
  ROUND(VARIABLE_VALUE / 3600, 2) as uptime_hours
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Uptime';

SELECT 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Threads_connected') as current_connections,
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
   WHERE VARIABLE_NAME = 'max_connections') as max_connections;

-- QPS
SELECT 'QUERIES PER SECOND' as '';
SELECT 
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime'), 2
  ) as avg_qps;

-- Buffer pool hit rate
SELECT 'BUFFER POOL HIT RATE' as '';
SELECT 
  ROUND(
    ((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') -
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads')) /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') * 100, 2
  ) as hit_rate_percent;

-- Slow queries
SELECT 'SLOW QUERIES' as '';
SELECT 
  VARIABLE_VALUE as total_slow_queries,
  ROUND(VARIABLE_VALUE / 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') * 100, 4) as slow_query_percent
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Slow_queries';

-- Processus actifs
SELECT 'ACTIVE PROCESSES' as '';
SELECT COUNT(*) as active_queries
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep';

-- Top 5 requêtes longues
SELECT 'TOP LONG QUERIES' as '';
SELECT ID, USER, DB, TIME, STATE, LEFT(INFO, 80)
FROM information_schema.PROCESSLIST
WHERE TIME > 5 AND COMMAND != 'Sleep'
ORDER BY TIME DESC
LIMIT 5;
EOF
```

### Commande One-Liner

```bash
# Quick health check
mariadb -u root -p -e "
SELECT 'Connections' as metric, 
  CONCAT(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME='Threads_connected'), 
    '/', 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES 
     WHERE VARIABLE_NAME='max_connections')
  ) as value
UNION ALL
SELECT 'QPS', 
  ROUND((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME='Questions') / 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME='Uptime'), 2)
UNION ALL
SELECT 'Slow Queries', 
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME='Slow_queries');
"
```

---

## ✅ Points Clés à Retenir

### Surveillance Basique
- ✅ `\s` ou `STATUS` : Statut connexion rapide
- ✅ `SHOW PROCESSLIST` : Surveiller connexions actives
- ✅ `SHOW ENGINE INNODB STATUS` : Diagnostic InnoDB complet

### Variables Critiques
- ✅ `innodb_buffer_pool_size` : 70-80% RAM
- ✅ `max_connections` : Ajuster selon charge
- ✅ `innodb_io_capacity` : Adapter au stockage

### Métriques Importantes
- ✅ **Buffer pool hit rate** : > 99%
- ✅ **Slow query ratio** : < 1%
- ✅ **Threads connected** : < max_connections
- ✅ **QPS** : Indicateur de charge

### Diagnostic
- ✅ Requêtes > 30s : Vérifier et optimiser
- ✅ Lock waits : Analyser transactions
- ✅ History list length > 1000 : Problème purge
- ✅ Pending I/O : Vérifier disques

---

## 🔗 Ressources et Références

### Documentation Officielle
- [SHOW STATUS](https://mariadb.com/kb/en/show-status/)
- [SHOW VARIABLES](https://mariadb.com/kb/en/show-variables/)
- [SHOW ENGINE INNODB STATUS](https://mariadb.com/kb/en/show-engine-innodb-status/)
- [SHOW PROCESSLIST](https://mariadb.com/kb/en/show-processlist/)
- [INFORMATION_SCHEMA](https://mariadb.com/kb/en/information-schema-tables/)

### Outils Complémentaires
- **mytop** : Monitoring temps réel style `top`
- **innotop** : Monitoring InnoDB avancé
- **pt-query-digest** : Analyse slow query log
- **Prometheus + mysqld_exporter** : Monitoring production

---

## ➡️ Section Suivante

**[B.3 Export/Import →](./03-export-import.md)**  
Découvrez les commandes pour exporter et importer des données : SOURCE, TEE, mariadb-dump, LOAD DATA

---

**MariaDB** : 11.8 LTS

⏭️ [Export/Import (SOURCE, TEE, mariadb-dump)](/annexes/commandes-cli/03-export-import.md)

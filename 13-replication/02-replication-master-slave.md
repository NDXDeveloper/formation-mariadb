🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.2 Réplication Master-Slave (Source-Replica)

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Section 13.1 (Concepts de réplication asynchrone/semi-synchrone)
> - Maîtrise des binary logs (Section 11.5)
> - Compréhension du networking TCP/IP
> - Expérience en administration système Linux

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre l'architecture complète de la réplication Master-Slave
- Identifier les composants (threads, logs, positions) et leur interaction
- Planifier une topologie de réplication adaptée aux besoins métier
- Configurer un environnement de réplication sécurisé et performant
- Appliquer les bonnes pratiques de production
- Diagnostiquer les problèmes courants de réplication
- Migrer d'une terminologie Master-Slave vers Source-Replica

---

## Introduction

La **réplication Master-Slave** (désormais appelée **Source-Replica** dans la terminologie moderne) est le modèle de réplication le plus répandu dans MariaDB. Elle établit une relation unidirectionnelle où :

- Un serveur **Primary/Source** accepte les écritures et génère un flux de modifications
- Un ou plusieurs serveurs **Replica/Slave** reçoivent et appliquent ces modifications

Cette architecture est la pierre angulaire de nombreuses stratégies de haute disponibilité, scalabilité et disaster recovery.

### Évolution de la terminologie

**Historique** :
- Jusqu'à MariaDB 10.5 : Master/Slave
- MariaDB 10.5+ : Introduction de la terminologie Source/Replica
- MariaDB 11.8 : Les deux terminologies coexistent (rétro-compatibilité)

**Commandes équivalentes** :

| Ancienne (Master-Slave) | Nouvelle (Source-Replica) |
|-------------------------|---------------------------|
| `SHOW MASTER STATUS` | `SHOW MASTER STATUS` (inchangé) |
| `SHOW SLAVE STATUS` | `SHOW REPLICA STATUS` |
| `CHANGE MASTER TO` | `CHANGE REPLICATION SOURCE TO` |
| `START SLAVE` | `START REPLICA` |
| `STOP SLAVE` | `STOP REPLICA` |
| `RESET SLAVE` | `RESET REPLICA` |

💡 **Dans ce document**, nous utiliserons principalement la terminologie moderne **Primary/Replica** tout en mentionnant les équivalents legacy pour la compatibilité.

---

## Architecture de Réplication

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────┐
│                     SERVEUR PRIMARY (Source)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                      │
│  │ Application  │  Write  │   MariaDB    │                      │
│  │   Clients    │ ──────→ │    Server    │                      │
│  └──────────────┘         └──────┬───────┘                      │
│                                  │                              │
│                            ┌─────▼─────────┐                    │
│                            │  Binary Log   │                    │
│                            │               │                    │
│                            │ mariadb-bin.* │                    │
│                            └──────┬────────┘                    │
│                                   │                             │
│                            ┌──────▼────────┐                    │
│                            │ Binlog Dump   │                    │
│                            │    Thread     │                    │
│                            └──────┬────────┘                    │
└────────────────────────────────┬──┬─────────────────────────────┘
                                 │  │
                     Network     │  │     Network
                     (TCP/IP)    │  │     (TCP/IP)
                                 │  │
      ┌──────────────────────────┘  └────────────────────────────┐
      │                                                          │
      ▼                                                          ▼
┌─────────────────────────┐                         ┌─────────────────────────┐
│   REPLICA 1 (Slave)     │                         │   REPLICA 2 (Slave)     │
├─────────────────────────┤                         ├─────────────────────────┤
│                         │                         │                         │
│  ┌──────────────┐       │                         │  ┌──────────────┐       │
│  │  IO Thread   │       │                         │  │  IO Thread   │       │
│  │ (Replication)│       │                         │  │ (Replication)│       │
│  └──────┬───────┘       │                         │  └──────┬───────┘       │
│         │               │                         │         │               │
│  ┌──────▼───────┐       │                         │  ┌──────▼───────┐       │
│  │  Relay Log   │       │                         │  │  Relay Log   │       │
│  │              │       │                         │  │              │       │
│  │ relay-bin.*  │       │                         │  │ relay-bin.*  │       │
│  └──────┬───────┘       │                         │  └──────┬───────┘       │
│         │               │                         │         │               │
│  ┌──────▼───────┐       │                         │  ┌──────▼───────┐       │
│  │  SQL Thread  │       │                         │  │  SQL Thread  │       │
│  │  (Apply)     │       │                         │  │  (Apply)     │       │
│  └──────┬───────┘       │                         │  └──────┬───────┘       │
│         │               │                         │         │               │
│  ┌──────▼───────┐       │                         │  ┌──────▼───────┐       │
│  │   MariaDB    │       │                         │  │   MariaDB    │       │
│  │   Server     │       │                         │  │   Server     │       │
│  └──────┬───────┘       │                         │  └──────┬───────┘       │
│         │               │                         │         │               │
│  ┌──────▼───────┐       │                         │  ┌──────▼───────┐       │
│  │ Application  │ Read  │                         │  │ Application  │ Read  │
│  │   Clients    │◄──────│                         │  │   Clients    │◄──────│
│  └──────────────┘       │                         │  └──────────────┘       │
│                         │                         │                         │
└─────────────────────────┘                         └─────────────────────────┘
```

### Composants essentiels

#### 1. Binary Log (Primary)

Le **binary log** enregistre toutes les modifications de données dans un format binaire séquentiel.

**Caractéristiques** :
- Format : STATEMENT, ROW, ou MIXED
- Fichiers numérotés séquentiellement : `mariadb-bin.000001`, `mariadb-bin.000002`, etc.
- Index : `mariadb-bin.index` (liste des fichiers binlog actifs)
- Rotation : Automatique selon `max_binlog_size` ou `FLUSH LOGS`

**Structure d'un événement binlog** :

```
┌─────────────────────────────────────┐
│     Binary Log Event                │
├─────────────────────────────────────┤
│ Timestamp        : 1702456789       │
│ Event Type       : QUERY_EVENT      │
│ Server ID        : 1                │
│ Log Position     : 4567             │
│ Flags            : 0                │
│ ─────────────────────────────────── │
│ Thread ID        : 42               │
│ Execution Time   : 123 ms           │
│ Error Code       : 0                │
│ ─────────────────────────────────── │
│ Database         : mydb             │
│ Query/Row Data   : INSERT INTO ...  │
└─────────────────────────────────────┘
```

**Commandes de gestion** :

```sql
-- Voir les fichiers binlog
SHOW BINARY LOGS;
-- +--------------------+-----------+
-- | Log_name           | File_size |
-- +--------------------+-----------+
-- | mariadb-bin.000041 | 1073741824|
-- | mariadb-bin.000042 | 536870912 |
-- +--------------------+-----------+

-- Voir la position actuelle
SHOW MASTER STATUS;
-- +--------------------+----------+--------------+------------------+
-- | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +--------------------+----------+--------------+------------------+
-- | mariadb-bin.000042 | 4567     |              |                  |
-- +--------------------+----------+--------------+------------------+

-- Afficher le contenu (format lisible)
SHOW BINLOG EVENTS IN 'mariadb-bin.000042' FROM 4000 LIMIT 10;

-- Ou avec mysqlbinlog (CLI)
mysqlbinlog /var/log/mysql/mariadb-bin.000042 --start-position=4000
```

#### 2. Binlog Dump Thread (Primary)

Thread côté Primary qui envoie les événements binlog aux Replicas.

**Fonctionnement** :
- Un thread créé **par Replica** connecté
- Lit les événements depuis la position demandée
- Transmet via connexion TCP/IP
- Suit la rotation des fichiers binlog automatiquement

**Visualisation** :

```sql
-- Sur le Primary
SHOW PROCESSLIST;
-- +-----+-------------+-----------------+------+---------+-------+---------------------------------------+
-- | Id  | User        | Host            | db   | Command | Time  | State                                 |
-- +-----+-------------+-----------------+------+---------+-------+---------------------------------------+
-- | 123 | repl_user   | replica1:45678  | NULL | Binlog  | 12456 | Master has sent all binlog to slave   |
-- | 124 | repl_user   | replica2:45679  | NULL | Binlog  | 8901  | Master has sent all binlog to slave   |
-- +-----+-------------+-----------------+------+---------+-------+---------------------------------------+
```

💡 **Astuce** : La colonne `Time` indique depuis combien de secondes la réplication est active. Des valeurs élevées sont normales et saines.

#### 3. Relay Log (Replica)

Le **relay log** est une copie locale des événements binlog sur le Replica.

**Raison d'être** :
- Découplage I/O (lecture réseau) et SQL (application)
- Permet de relire les événements en cas d'erreur
- Résistance aux interruptions réseau temporaires

**Gestion automatique** :

```sql
-- Voir les relay logs
SHOW RELAYLOG EVENTS IN 'relay-bin.000003' LIMIT 10;

-- Purge automatique après application
SET GLOBAL relay_log_purge = ON;  -- Default

-- Conservation pour debug
SET GLOBAL relay_log_purge = OFF;
```

**Configuration** :

```ini
[mysqld]
# Nom des relay logs
relay-log = /var/log/mysql/relay-bin

# Index
relay-log-index = /var/log/mysql/relay-bin.index

# Taille max avant rotation
max_relay_log_size = 100M  # 0 = utilise max_binlog_size

# Récupération automatique après crash
relay_log_recovery = ON    # Recommandé
```

#### 4. IO Thread (Replica)

Thread responsable de la **récupération** des événements binlog depuis le Primary.

**Workflow** :

```
1. Connexion au Primary (TCP/IP)
2. Authentification (user replication)
3. Demande d'événements depuis position X
4. Réception continue des événements
5. Écriture dans relay log local
6. Mise à jour position dans master.info
```

**Monitoring** :

```sql
SHOW REPLICA STATUS\G
-- Ou
SHOW SLAVE STATUS\G

-- Champs importants :
-- Slave_IO_State       : État du thread IO
-- Slave_IO_Running     : Yes/No
-- Master_Log_File      : Fichier binlog lu actuellement
-- Read_Master_Log_Pos  : Position de lecture
-- Relay_Log_File       : Fichier relay log écrit actuellement
-- Relay_Log_Pos        : Position d'écriture relay log
```

**États possibles du IO Thread** :

| État | Signification |
|------|---------------|
| `Waiting for master to send event` | Idle, en attente de nouvelles données (normal) |
| `Connecting to master` | Tentative de connexion initiale |
| `Reconnecting after a failed master event read` | Tentative de reconnexion après erreur |
| `Waiting for master to send event` | Synchronisé, en attente |
| `Queueing master event to the relay log` | Écriture active dans relay log |

#### 5. SQL Thread (Replica)

Thread responsable de l'**application** des événements depuis le relay log.

**Workflow** :

```
1. Lecture d'un événement depuis relay log
2. Parsing de l'événement
3. Exécution dans le moteur de stockage
4. Mise à jour position dans relay-log.info
5. Passage à l'événement suivant
```

**Monitoring** :

```sql
SHOW REPLICA STATUS\G

-- Champs importants :
-- Slave_SQL_Running       : Yes/No
-- Relay_Master_Log_File   : Fichier binlog correspondant (Primary)
-- Exec_Master_Log_Pos     : Position exécutée (Primary)
-- Relay_Log_Space         : Espace disque relay logs
-- Seconds_Behind_Master   : Lag estimé en secondes
```

**États possibles du SQL Thread** :

| État | Signification |
|------|---------------|
| `Reading event from the relay log` | Lecture d'un événement |
| `Waiting for the next event in relay log` | Synchronisé, en attente (normal) |
| `Making temp file` | Création de table temporaire |
| `Slave has read all relay log` | Tous les événements appliqués |

### Flux de données complet

**Scénario : INSERT sur le Primary**

```
┌──────────────────────────────────────────────────────────────┐
│                           Timeline                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  T0: Client                                                  │
│      INSERT INTO users (name) VALUES ('Alice');              │
│      COMMIT;                                                 │
│      ↓                                                       │
│  T1: PRIMARY                                                 │
│      1. InnoDB écrit dans buffer pool + redo log             │
│      2. Écriture dans binary log (binlog_format=ROW)         │
│         Event: WRITE_ROWS_EVENT                              │
│         Position: mariadb-bin.000042:4567                    │
│      3. Confirmation COMMIT au client ✓                      │
│      ↓                                                       │
│  T2: Binlog Dump Thread (Primary)                            │
│      Détecte nouveau événement                               │
│      Transmet à tous les Replicas connectés                  │
│      ↓                                                       │
│  T3: IO Thread (Replica)                                     │
│      Reçoit WRITE_ROWS_EVENT via réseau                      │
│      Écrit dans relay-bin.000005:8912                        │
│      Mise à jour Read_Master_Log_Pos = 4567                  │
│      ↓                                                       │
│  T4: SQL Thread (Replica)                                    │
│      Lit depuis relay-bin.000005:8912                        │
│      Parse l'événement ROW                                   │
│      Exécute : INSERT dans InnoDB local                      │
│      Mise à jour Exec_Master_Log_Pos = 4567                  │
│      ↓                                                       │
│  T5: REPLICA                                                 │
│      Données visibles pour SELECT                            │
│      Lag = (T5 - T1) ≈ quelques ms à secondes                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Latence typique** :

- LAN (< 1ms network) : 5-50ms total
- WAN (10-50ms network) : 50-500ms total
- Geo-distributed (100-200ms network) : 500-2000ms total

---

## Topologies de Réplication

### 1. Single Primary, Single Replica

La topologie la plus simple :

```
┌──────────┐
│ Primary  │
│  (RW)    │
└─────┬────┘
      │
      ▼
┌──────────┐
│ Replica  │
│  (RO)    │
└──────────┘
```

**Use cases** :
- ✅ Développement/Testing
- ✅ Disaster Recovery basique
- ✅ Offloading de requêtes de lecture

**Limitations** :
- ❌ Pas de HA (Single Point of Failure)
- ❌ Scalabilité limitée

### 2. Single Primary, Multiple Replicas

Architecture **scalable en lecture** :

```
              ┌──────────┐
              │ Primary  │
              │  (RW)    │
              └────┬─────┘
                   │
      ┌────────────┼────────────┐
      │            │            │
      ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Replica1 │ │ Replica2 │ │ Replica3 │
│  (RO)    │ │  (RO)    │ │  (RO)    │
│ Web App  │ │ Reporting│ │ Backup   │
└──────────┘ └──────────┘ └──────────┘
```

**Use cases** :
- ✅ Haute charge de lecture
- ✅ Séparation des workloads (OLTP vs OLAP)
- ✅ Redondance géographique

**Bonnes pratiques** :
- Assigner des **rôles distincts** aux Replicas
- Utiliser un **load balancer** pour les lectures
- Monitorer le **lag individuel** de chaque Replica

### 3. Chained Replication (Cascade)

Réplication en **chaîne** pour réduire la charge sur le Primary :

```
┌──────────┐
│ Primary  │
└─────┬────┘
      │
      ▼
┌──────────┐  log_slave_updates=ON
│ Relay    │  ← Génère son propre binlog
└─────┬────┘
      │
      ├───────────┐
      │           │
      ▼           ▼
┌──────────┐ ┌──────────┐
│ Replica1 │ │ Replica2 │
└──────────┘ └──────────┘
```

**Configuration du Relay** :

```ini
[mysqld]
log-slave-updates = ON   # CRITIQUE !
```

**Use cases** :
- ✅ Nombreux Replicas (>10)
- ✅ Bande passante limitée sur Primary
- ✅ Distribution géographique (Relay par région)

**Inconvénients** :
- ❌ Lag accumulé (chaque niveau ajoute du délai)
- ❌ Point de défaillance supplémentaire (Relay)
- ❌ Complexité de troubleshooting

⚠️ **Limitation** : Maximum 3-4 niveaux recommandés (lag exponentiel)

### 4. Multi-Source Replication

Un Replica réplique depuis **plusieurs Primary** :

```
┌──────────┐       ┌──────────┐
│ Primary1 │       │ Primary2 │
│ (Sales)  │       │  (HR)    │
└─────┬────┘       └────┬─────┘
      │                 │
      └────────┬────────┘
               │
               ▼
         ┌───────────┐
         │ Replica   │
         │(Reporting)│
         └───────────┘
```

**Configuration** :

```sql
-- Canal 'sales'
CHANGE MASTER 'sales' TO
  MASTER_HOST='sales-primary.example.com',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='password',
  MASTER_USE_GTID=slave_pos;

START SLAVE 'sales';

-- Canal 'hr'
CHANGE MASTER 'hr' TO
  MASTER_HOST='hr-primary.example.com',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='password',
  MASTER_USE_GTID=slave_pos;

START SLAVE 'hr';

-- Vérifier les deux canaux
SHOW ALL SLAVES STATUS\G
```

**Use cases** :
- ✅ Consolidation de bases séparées
- ✅ Reporting centralisé
- ✅ Migration progressive

**Détails** : Voir section **13.5 Réplication multi-source**

---

## Planification de la Réplication

### Étapes de mise en œuvre

**1. Planification de l'architecture** :

```
Checklist :
├─ Identifier le Primary et les Replicas (hardware, datacenter)
├─ Déterminer le mode : async ou semi-sync
├─ Choisir binlog_format : ROW (recommandé) vs STATEMENT
├─ Planifier la topologie : simple, cascade, multi-source
├─ Dimensionner le réseau (bande passante, latence)
├─ Définir la stratégie de backup (depuis Replica ?)
└─ Documenter la topologie et les rôles
```

**2. Préparation de l'infrastructure** :

```bash
# Réseau
├─ Vérifier connectivité TCP/IP entre serveurs
├─ Ouvrir port 3306 dans firewalls
├─ Configurer DNS ou hosts file
└─ Tester latence réseau (ping, iperf)

# Stockage
├─ Provisionner espace disque suffisant (binlog + relay log)
├─ SSD recommandé pour InnoDB + logs
└─ Monitoring espace disque (alertes < 20% libre)

# Serveurs
├─ Mêmes versions MariaDB recommandées
├─ Configuration système similaire (ulimit, sysctl)
└─ NTP synchronisé (important pour timestamps)
```

**3. Configuration de base** :

Sur le **Primary** :
```ini
[mysqld]
server-id = 1                    # Unique !
log-bin = mariadb-bin
binlog_format = ROW              # Recommandé
max_binlog_size = 100M
expire_logs_days = 7
sync_binlog = 1                  # Durabilité
```

Sur chaque **Replica** :
```ini
[mysqld]
server-id = 2                    # Unique !
relay-log = relay-bin
relay-log-index = relay-bin.index
relay_log_recovery = ON
read_only = ON                   # Protection
super_read_only = ON             # Même pour SUPER users
```

**4. Sécurité** :

```sql
-- Sur le Primary : Créer utilisateur de réplication
CREATE USER 'repl_user'@'%' 
  IDENTIFIED BY 'Str0ng_P@ssw0rd_2025!';

GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_user'@'%';

-- Optionnel : Restreindre par IP
CREATE USER 'repl_user'@'192.168.1.%' 
  IDENTIFIED BY 'Str0ng_P@ssw0rd_2025!';

-- Optionnel : Utiliser SSL
GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_user'@'%' 
  REQUIRE SSL;
```

**5. Initialisation des données** :

Plusieurs méthodes selon la taille de la base :

**Méthode A : Dump logique (petites bases < 10GB)**

```bash
# Sur le Primary
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --flush-logs \
  --routines \
  --triggers \
  --events \
  > full_dump.sql

# Copier vers Replica
scp full_dump.sql replica:/tmp/

# Sur le Replica
mariadb -u root -p < /tmp/full_dump.sql
```

**Méthode B : Backup physique (grosses bases > 10GB)**

```bash
# Sur le Primary : Mariabackup
mariabackup --backup \
  --target-dir=/backup/full \
  --user=root \
  --password=xxx

# Copier vers Replica
rsync -avz /backup/full/ replica:/backup/full/

# Sur le Replica : Préparer et restaurer
mariabackup --prepare --target-dir=/backup/full
mariabackup --copy-back --target-dir=/backup/full
chown -R mysql:mysql /var/lib/mysql
```

**Méthode C : Clone Plugin (MariaDB 10.5+)**

```sql
-- Sur le Replica
INSTALL PLUGIN clone SONAME 'ha_clone';

-- Cloner depuis Primary
CLONE INSTANCE FROM 'repl_user'@'primary.example.com':3306
  IDENTIFIED BY 'password';
```

**6. Démarrage de la réplication** :

```sql
-- Sur le Replica : Configurer la source
CHANGE MASTER TO
  MASTER_HOST = 'primary.example.com',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'Str0ng_P@ssw0rd_2025!',
  MASTER_LOG_FILE = 'mariadb-bin.000042',  -- Depuis dump/backup
  MASTER_LOG_POS = 4567;                    -- Depuis dump/backup

-- Démarrer la réplication
START SLAVE;

-- Vérifier l'état
SHOW SLAVE STATUS\G
```

**7. Validation** :

```sql
-- Sur le Primary : Créer données test
CREATE DATABASE repl_test;
USE repl_test;
CREATE TABLE test (id INT PRIMARY KEY AUTO_INCREMENT, data VARCHAR(100));
INSERT INTO test (data) VALUES ('Test replication at ' || NOW());

-- Sur le Replica : Vérifier réplication
USE repl_test;
SELECT * FROM test;
-- Doit afficher les mêmes données

-- Vérifier absence de lag
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 0 (ou proche de 0)
```

---

## Considérations de Sécurité

### 1. Authentification forte

**Utiliser ed25519** (recommandé MariaDB 10.4+) :

```sql
-- Sur le Primary
INSTALL SONAME 'auth_ed25519';

CREATE USER 'repl_user'@'%' 
  IDENTIFIED VIA ed25519 
  USING PASSWORD('Str0ng_P@ssw0rd_2025!');

GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
```

**Rotation des mots de passe** :

```sql
-- Changer le mot de passe
ALTER USER 'repl_user'@'%' 
  IDENTIFIED BY 'New_P@ssw0rd_2025!';

-- Sur le Replica : Mettre à jour
STOP SLAVE;

CHANGE MASTER TO 
  MASTER_PASSWORD = 'New_P@ssw0rd_2025!';

START SLAVE;
```

### 2. Chiffrement SSL/TLS

**Configuration du Primary** :

```ini
[mysqld]
ssl-ca = /etc/mysql/ssl/ca-cert.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key = /etc/mysql/ssl/server-key.pem

# Forcer SSL pour la réplication
require_secure_transport = ON
```

**Configuration du Replica** :

```sql
CHANGE MASTER TO
  MASTER_HOST = 'primary.example.com',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'password',
  MASTER_SSL = 1,
  MASTER_SSL_CA = '/etc/mysql/ssl/ca-cert.pem',
  MASTER_SSL_CERT = '/etc/mysql/ssl/client-cert.pem',
  MASTER_SSL_KEY = '/etc/mysql/ssl/client-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1;  -- Vérifier le certificat
```

**Vérification** :

```sql
SHOW SLAVE STATUS\G
-- Master_SSL_Allowed: Yes
-- Master_SSL_CA_File: /etc/mysql/ssl/ca-cert.pem
```

### 3. Restriction réseau

**Par firewall (iptables)** :

```bash
# Autoriser uniquement les Replicas
iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.10 -j ACCEPT  # Replica1
iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.11 -j ACCEPT  # Replica2
iptables -A INPUT -p tcp --dport 3306 -j DROP                    # Bloquer le reste
```

**Par MariaDB (bind-address)** :

```ini
[mysqld]
# Écouter seulement sur l'interface privée
bind-address = 192.168.1.5
```

### 4. Audit de la réplication

**Activer le Server Audit Plugin** :

```sql
INSTALL SONAME 'server_audit';

SET GLOBAL server_audit_events = 'CONNECT,QUERY';
SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_incl_users = 'repl_user';
```

**Logs générés** :

```
20251213 14:32:15,replica1,repl_user,192.168.1.10,45678,0,CONNECT,,,0
20251213 14:32:16,replica1,repl_user,192.168.1.10,45678,0,QUERY,,'SHOW MASTER STATUS',0
```

---

## Bonnes Pratiques de Production

### 1. Naming conventions

```
Serveurs :
├─ db-primary-01.prod.example.com
├─ db-replica-01.prod.example.com  (reads)
├─ db-replica-02.prod.example.com  (reporting)
└─ db-replica-03.dr.example.com    (disaster recovery)

Binary logs :
├─ Préfixe descriptif : mariadb-bin (pas mysql-bin)
└─ Relay logs : relay-bin (pas relay-log)

Users :
├─ repl_user (production)
├─ repl_user_dev (développement)
└─ repl_user_dr (DR site)
```

### 2. Configuration read-only stricte

```sql
-- Sur TOUS les Replicas
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;  -- Bloque même les SUPER users

-- Vérifier
SELECT @@read_only, @@super_read_only;
-- +-------------+-------------------+
-- | @@read_only | @@super_read_only |
-- +-------------+-------------------+
-- |           1 |                 1 |
-- +-------------+-------------------+

-- Tester (doit échouer)
INSERT INTO test VALUES (1);
-- ERROR 1290 (HY000): The MariaDB server is running with 
-- the --read-only option so it cannot execute this statement
```

⚠️ **Exception** : Le thread de réplication SQL peut toujours écrire (flag spécial).

### 3. Monitoring continu

**Script de monitoring** :

```bash
#!/bin/bash
# check_replication.sh

SLAVE_STATUS=$(mysql -e "SHOW SLAVE STATUS\G")

IO_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_IO_Running" | awk '{print $2}')
SQL_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_SQL_Running" | awk '{print $2}')
LAG=$(echo "$SLAVE_STATUS" | grep "Seconds_Behind_Master" | awk '{print $2}')

if [ "$IO_RUNNING" != "Yes" ] || [ "$SQL_RUNNING" != "Yes" ]; then
  echo "CRITICAL: Replication stopped!"
  exit 2
fi

if [ "$LAG" = "NULL" ]; then
  echo "WARNING: Cannot determine lag (IO not running?)"
  exit 1
fi

if [ "$LAG" -gt 300 ]; then
  echo "WARNING: Replication lag is $LAG seconds"
  exit 1
fi

echo "OK: Replication running, lag: $LAG seconds"
exit 0
```

**Intégration Nagios/Icinga** :

```ini
# /etc/nagios/nrpe.cfg
command[check_replication]=/usr/local/bin/check_replication.sh
```

### 4. Documentation de topologie

```sql
-- Table de documentation (sur un serveur de monitoring)
CREATE TABLE replication_topology (
  hostname VARCHAR(255) PRIMARY KEY,
  role ENUM('primary', 'replica', 'relay') NOT NULL,
  datacenter VARCHAR(100),
  replication_mode ENUM('async', 'semi-sync'),
  replicates_from VARCHAR(255),
  purpose TEXT,
  contact_email VARCHAR(255),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_role (role),
  INDEX idx_datacenter (datacenter)
);

-- Exemple de données
INSERT INTO replication_topology VALUES
('db-primary-01.prod', 'primary', 'DC1-Paris', 'semi-sync', NULL, 
 'Production writes', 'dba@example.com', NOW()),
('db-replica-01.prod', 'replica', 'DC1-Paris', 'semi-sync', 'db-primary-01.prod',
 'Production reads + HA failover', 'dba@example.com', NOW()),
('db-replica-02.prod', 'replica', 'DC1-Paris', 'async', 'db-primary-01.prod',
 'Reporting and analytics', 'analytics@example.com', NOW()),
('db-replica-03.dr', 'replica', 'DC2-London', 'async', 'db-primary-01.prod',
 'Disaster Recovery', 'dba@example.com', NOW());
```

### 5. Automatisation avec Ansible

**Playbook de déploiement** :

```yaml
# deploy_replication.yml
---
- name: Configure MariaDB Replication
  hosts: replicas
  become: yes
  vars:
    master_host: "{{ groups['primary'][0] }}"
    repl_user: "repl_user"
    repl_password: "{{ vault_repl_password }}"
  
  tasks:
    - name: Configure server-id
      lineinfile:
        path: /etc/mysql/mariadb.conf.d/50-server.cnf
        regexp: '^server-id'
        line: "server-id = {{ ansible_host.split('.')[-1] }}"
      notify: restart mariadb
    
    - name: Enable relay log
      lineinfile:
        path: /etc/mysql/mariadb.conf.d/50-server.cnf
        line: "relay-log = relay-bin"
      notify: restart mariadb
    
    - name: Set read-only
      lineinfile:
        path: /etc/mysql/mariadb.conf.d/50-server.cnf
        line: "read-only = ON"
      notify: restart mariadb
    
    - name: Configure replication
      mysql_replication:
        mode: changeprimary
        master_host: "{{ master_host }}"
        master_user: "{{ repl_user }}"
        master_password: "{{ repl_password }}"
        master_auto_position: yes
      
    - name: Start replication
      mysql_replication:
        mode: startslave
  
  handlers:
    - name: restart mariadb
      service:
        name: mariadb
        state: restarted
```

---

## Troubleshooting Courant

### Problème 1 : Replica ne démarre pas

**Symptômes** :

```sql
SHOW SLAVE STATUS\G
-- Slave_IO_Running: No
-- Last_IO_Error: error connecting to master 'repl_user@primary:3306'
```

**Solutions** :

```bash
# 1. Vérifier connectivité réseau
ping primary.example.com
telnet primary.example.com 3306

# 2. Vérifier credentials
mysql -h primary.example.com -u repl_user -p
# Doit se connecter

# 3. Vérifier privilèges
# Sur le Primary
SHOW GRANTS FOR 'repl_user'@'%';
# Doit inclure REPLICATION SLAVE

# 4. Vérifier firewall
# Sur le Primary
iptables -L -n | grep 3306
```

### Problème 2 : Lag de réplication élevé

**Diagnostic** :

```sql
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 3600  (1 heure de lag !)

-- Identifier la cause
SHOW PROCESSLIST;
-- Si SQL Thread en 'Making temp file' → Table temporaire énorme
-- Si SQL Thread en 'Waiting for table metadata lock' → Lock contention
```

**Solutions** :

```sql
-- 1. Activer parallélisation (si pas déjà fait)
SET GLOBAL slave_parallel_threads = 4;
SET GLOBAL slave_parallel_mode = 'optimistic';

-- 2. Augmenter max_allowed_packet (si gros binlog events)
SET GLOBAL max_allowed_packet = 64M;

-- 3. Vérifier slow queries sur Replica
SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;
```

### Problème 3 : Réplication cassée (duplicate key)

**Symptômes** :

```sql
SHOW SLAVE STATUS\G
-- Slave_SQL_Running: No
-- Last_SQL_Error: Error 'Duplicate entry '123' for key 'PRIMARY'' on query
```

**Solutions** :

```sql
-- Option 1 : Ignorer cette erreur (avec prudence !)
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE SQL_THREAD;

-- Option 2 : Fixer manuellement
STOP SLAVE SQL_THREAD;
-- Identifier et supprimer la ligne en doublon sur Replica
DELETE FROM problematic_table WHERE id = 123;
START SLAVE SQL_THREAD;

-- Option 3 : Resynchroniser complètement
-- (Dump/restore depuis Primary)
```

⚠️ **Attention** : Ces erreurs indiquent souvent un problème plus profond (écriture manuelle sur Replica, etc.)

---

## Sous-sections du Chapitre 13.2

Ce chapitre se décompose en 3 sous-sections détaillées :

### 📖 13.2.1 Configuration du Primary (binlog)

- Paramètres binlog détaillés (`binlog_format`, `sync_binlog`, etc.)
- Optimisation des performances binlog
- Gestion de la rotation et purge
- Sécurité et encryption des binlogs

### 📖 13.2.2 Configuration du Replica

- Paramètres relay log et recovery
- Configuration `read_only` et `super_read_only`
- Optimisation parallélisation (slave_parallel_threads)
- Monitoring et métriques

### 📖 13.2.3 CHANGE MASTER TO / CHANGE REPLICATION SOURCE

- Syntaxe complète de la commande
- Paramètres SSL/TLS
- Utilisation de GTID
- Migration de position vers GTID
- Reconfiguration à chaud

---

## ✅ Points clés à retenir

1. **Architecture en 5 composants** : Binary Log, Binlog Dump Thread, Relay Log, IO Thread, SQL Thread

2. **Terminologie moderne** : Préférer Source/Replica à Master/Slave, mais les deux coexistent

3. **Topologies variées** : Simple, multiple replicas, cascade, multi-source selon besoins

4. **Sécurité critique** : Utilisateur dédié, SSL/TLS, read_only strict, audit

5. **Planification essentielle** : Architecture, réseau, stockage, initialisation, validation

6. **server-id unique** : Absolument critique, ne jamais dupliquer

7. **Monitoring 24/7** : IO_Running, SQL_Running, Seconds_Behind_Master minimum

8. **read_only + super_read_only** : Protection contre écritures accidentelles sur Replica

9. **Documentation vivante** : Maintenir un schéma de topologie à jour

10. **Automation** : Ansible/Terraform pour déploiements reproductibles

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Setting Up Replication](https://mariadb.com/kb/en/setting-up-replication/)
- [📖 Replication System Variables](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/)
- [📖 CHANGE MASTER TO](https://mariadb.com/kb/en/change-master-to/)
- [📖 SHOW SLAVE STATUS](https://mariadb.com/kb/en/show-slave-status/)

### Outils recommandés

- **Orchestrator** : Topologie management et failover automatique
- **pt-heartbeat** (Percona Toolkit) : Mesure précise du lag
- **Ansible MariaDB Role** : galaxy.ansible.com/geerlingguy/mysql
- **ProxySQL** : Load balancing intelligent read/write split

### Lectures complémentaires

- [🔗 Binary Log Internals](https://dev.mysql.com/doc/internals/en/binary-log.html)
- [🔗 Replication Best Practices 2025](https://mariadb.com/resources/blog/)

---

## ➡️ Sections suivantes

### 13.2.1 Configuration du Primary (binlog)

Configuration approfondie du serveur Primary : formats de binlog, sync_binlog, optimisations, rotation automatique, et sécurité des binary logs.

### 13.2.2 Configuration du Replica

Configuration avancée du Replica : relay logs, parallélisation, read-only, recovery automatique, et optimisations spécifiques.

### 13.2.3 CHANGE MASTER TO / CHANGE REPLICATION SOURCE

Commande complète de configuration de la réplication : syntaxe, paramètres SSL, GTID, et reconfiguration dynamique.

---


⏭️ [Configuration du Primary (binlog)](/13-replication/02.1-configuration-primary.md)

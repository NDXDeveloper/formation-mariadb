🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.3 Réplication basée sur les Positions (Binlog Coordinates)

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** : 
> - Section 13.1 (Concepts de réplication)
> - Section 13.2 (Configuration Master-Slave)
> - Compréhension des binary logs (Section 11.5)
> - Notions de systèmes de fichiers et séquencement

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le système de coordonnées binlog (fichier + position)
- Configurer une réplication basée sur les positions
- Calculer et ajuster les positions lors de failover
- Diagnostiquer les problèmes de positions incorrectes
- Évaluer les limites de cette approche vs GTID
- Migrer d'une réplication par position vers GTID
- Gérer les opérations de maintenance sans interruption

---

## Introduction

La **réplication basée sur les positions binlog** est la méthode traditionnelle de réplication dans MariaDB et MySQL. Elle repose sur un système de **coordonnées** identifiant précisément chaque événement dans les binary logs :

```
Coordonnée binlog = (Nom de fichier, Position)
                    └──────┬──────┘  └───┬───┘
                     mariadb-bin.000042   4567
```

Cette méthode, bien qu'ancienne, reste largement utilisée et parfaitement fonctionnelle pour de nombreux cas d'usage. Cependant, elle présente des **défis opérationnels** notamment lors des failovers et dans les topologies complexes.

### Contexte historique

**Timeline** :
- **MySQL 3.23 (2001)** : Introduction de la réplication basée sur positions
- **MySQL 5.6 (2013)** : Introduction de GTID comme alternative
- **MariaDB 10.0 (2014)** : Implémentation propre de GTID
- **MariaDB 11.8 (2025)** : Les deux méthodes coexistent, GTID recommandé

💡 **Position actuelle** : La réplication par positions reste supportée et stable, mais GTID est maintenant recommandé pour les nouveaux déploiements.

---

## Anatomie d'un Binary Log

### Structure des fichiers binlog

Le Primary génère une **séquence de fichiers binlog** :

```
/var/log/mysql/
├── mariadb-bin.000001  (ancien, peut être purgé)
├── mariadb-bin.000002
├── mariadb-bin.000003
│   ...
├── mariadb-bin.000041
├── mariadb-bin.000042  ← Fichier actif
└── mariadb-bin.index   ← Index de tous les fichiers
```

**Fichier binlog-index** :

```bash
cat /var/log/mysql/mariadb-bin.index
# /var/log/mysql/mariadb-bin.000038
# /var/log/mysql/mariadb-bin.000039
# /var/log/mysql/mariadb-bin.000040
# /var/log/mysql/mariadb-bin.000041
# /var/log/mysql/mariadb-bin.000042
```

### Structure interne d'un fichier binlog

Chaque fichier binlog contient une **séquence d'événements** :

```
┌─────────────────────────────────────────────────────────────┐
│ mariadb-bin.000042                                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Position 0-4    : Magic Number (0xfe 0x62 0x69 0x6e)        │
│                                                             │
│ Position 4      : FORMAT_DESCRIPTION_EVENT                  │
│                   (Header du fichier)                       │
│                                                             │
│ Position 123    : QUERY_EVENT                               │
│                   "CREATE DATABASE mydb"                    │
│                                                             │
│ Position 456    : GTID_EVENT                                │
│                   (Si GTID activé)                          │
│                                                             │
│ Position 789    : QUERY_EVENT                               │
│                   "BEGIN"                                   │
│                                                             │
│ Position 890    : TABLE_MAP_EVENT                           │
│                   (Mapping table pour ROW format)           │
│                                                             │
│ Position 1234   : WRITE_ROWS_EVENT                          │
│                   (INSERT dans users)                       │
│                                                             │
│ Position 4567   : XID_EVENT                                 │ ← Position actuelle
│                   "COMMIT"                                  │
│                                                             │
│ Position 4890   : ROTATE_EVENT                              │
│                   (Rotation vers mariadb-bin.000043)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Visualisation avec mysqlbinlog** :

```bash
mysqlbinlog /var/log/mysql/mariadb-bin.000042 | head -50

# Output:
# /*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
# /*!40019 SET @@session.max_insert_delayed_threads=0*/;
# /*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
# DELIMITER /*!*/;
# # at 4
# #251213 14:32:15 server id 1  end_log_pos 256   Start: binlog v 4, server v 11.8.0-MariaDB created 251213 14:32:15
# BINLOG '
# d3PnZw8BAAAAAPgAAAABAAQAMTEuOC4wLU1hcmlhREIAAAAAAAAAAAA=
# '/*!*/;
# # at 256
# #251213 14:32:45 server id 1  end_log_pos 345   Query   thread_id=42 exec_time=0 error_code=0
# SET TIMESTAMP=1702456365/*!*/;
# CREATE DATABASE mydb
# /*!*/;
```

### Détail d'un événement binlog

**Structure d'un événement** :

```
┌──────────────────────────────────────────────────────────┐
│ BINLOG EVENT                                             │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ COMMON HEADER (19 bytes)                                 │
│ ┌────────────────────────────────────────────────────┐   │
│ │ Timestamp        : 4 bytes  (Unix timestamp)       │   │
│ │ Type Code        : 1 byte   (QUERY_EVENT, etc.)    │   │
│ │ Server ID        : 4 bytes  (Source server)        │   │
│ │ Event Length     : 4 bytes  (Total event size)     │   │
│ │ Next Position    : 4 bytes  (Position of next evt) │   │
│ │ Flags            : 2 bytes  (Event flags)          │   │
│ └────────────────────────────────────────────────────┘   │
│                                                          │
│ POST HEADER (Variable)                                   │
│ ┌────────────────────────────────────────────────────┐   │
│ │ Event-specific metadata                            │   │
│ └────────────────────────────────────────────────────┘   │
│                                                          │
│ EVENT DATA (Variable)                                    │
│ ┌────────────────────────────────────────────────────┐   │
│ │ Actual event payload (SQL, row data, etc.)         │   │
│ └────────────────────────────────────────────────────┘   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Exemple concret** :

```sql
-- Sur le Primary
SHOW BINLOG EVENTS IN 'mariadb-bin.000042' FROM 4567 LIMIT 1\G

-- Output:
*************************** 1. row ***************************
   Log_name: mariadb-bin.000042
        Pos: 4567
 Event_type: Query
  Server_id: 1
End_log_pos: 4890          ← Position du prochain événement
       Info: use `mydb`; INSERT INTO users (name) VALUES ('Alice')
```

💡 **Clé de compréhension** : `End_log_pos` est la position où commence le **prochain** événement. C'est cette valeur qu'on utilise pour `MASTER_LOG_POS`.

---

## Configuration de la Réplication par Positions

### 1. Préparation du Primary

**Vérification des binary logs** :

```sql
-- Vérifier que le binlog est activé
SHOW VARIABLES LIKE 'log_bin';
-- +---------------+-------+
-- | Variable_name | Value |
-- +---------------+-------+
-- | log_bin       | ON    |
-- +---------------+-------+

-- Voir les fichiers binlog disponibles
SHOW BINARY LOGS;
-- +--------------------+-----------+-----------+
-- | Log_name           | File_size | Encrypted |
-- +--------------------+-----------+-----------+
-- | mariadb-bin.000040 | 1073741824| No        |
-- | mariadb-bin.000041 | 1073741824| No        |
-- | mariadb-bin.000042 | 536870912 | No        |
-- +--------------------+-----------+-----------+

-- Obtenir la position actuelle
SHOW MASTER STATUS;
-- +--------------------+----------+--------------+------------------+
-- | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +--------------------+----------+--------------+------------------+
-- | mariadb-bin.000042 | 4567     |              |                  |
-- +--------------------+----------+--------------+------------------+
```

**Créer l'utilisateur de réplication** :

```sql
CREATE USER 'repl_user'@'%' 
  IDENTIFIED BY 'SecureP@ssw0rd_2025!';

GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_user'@'%';

FLUSH PRIVILEGES;
```

### 2. Obtenir une position de départ cohérente

**Méthode A : Avec verrouillage (petites bases)** :

```sql
-- Sur le Primary
FLUSH TABLES WITH READ LOCK;  -- Verrouille toutes les tables

-- Obtenir la position exacte
SHOW MASTER STATUS;
-- +--------------------+----------+--------------+------------------+
-- | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +--------------------+----------+--------------+------------------+
-- | mariadb-bin.000042 | 4567     |              |                  |
-- +--------------------+----------+--------------+------------------+

-- ⚠️ NOTER ces valeurs : mariadb-bin.000042, position 4567

-- Dump la base
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  > full_dump.sql

-- Déverrouiller
UNLOCK TABLES;
```

**Méthode B : Avec --master-data (automatique)** :

```bash
# mysqldump insère automatiquement CHANGE MASTER TO dans le dump
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \    # =2 : commenté, =1 : actif
  --routines \
  --triggers \
  --events \
  > full_dump.sql

# Le dump contient :
# -- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000042', MASTER_LOG_POS=4567;
```

**Méthode C : Avec Mariabackup (physique)** :

```bash
# Backup physique avec position automatique
mariabackup --backup \
  --target-dir=/backup/full \
  --user=root \
  --password=xxx

# Mariabackup crée automatiquement xtrabackup_binlog_info
cat /backup/full/xtrabackup_binlog_info
# mariadb-bin.000042	4567
```

### 3. Configuration du Replica

**Restaurer les données** :

```bash
# Avec dump logique
mariadb -u root -p < full_dump.sql

# Avec backup physique
mariabackup --prepare --target-dir=/backup/full
mariabackup --copy-back --target-dir=/backup/full
chown -R mysql:mysql /var/lib/mysql
systemctl start mariadb
```

**Configurer la réplication** :

```sql
-- Sur le Replica
CHANGE MASTER TO
  MASTER_HOST = 'primary.example.com',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'SecureP@ssw0rd_2025!',
  MASTER_LOG_FILE = 'mariadb-bin.000042',  -- Depuis dump/backup
  MASTER_LOG_POS = 4567;                    -- Depuis dump/backup

-- Démarrer la réplication
START SLAVE;

-- Vérifier
SHOW SLAVE STATUS\G
```

**Validation** :

```sql
-- Champs critiques à vérifier
SHOW SLAVE STATUS\G

-- Slave_IO_Running: Yes           ✓
-- Slave_SQL_Running: Yes           ✓
-- Master_Log_File: mariadb-bin.000042
-- Read_Master_Log_Pos: 4567        ← Position de lecture actuelle
-- Relay_Master_Log_File: mariadb-bin.000042
-- Exec_Master_Log_Pos: 4567        ← Position d'exécution actuelle
-- Seconds_Behind_Master: 0         ✓
-- Last_IO_Error:                   (vide = OK)
-- Last_SQL_Error:                  (vide = OK)
```

---

## Gestion des Positions

### Suivi de la progression

**Sur le Primary** :

```sql
-- Position actuelle d'écriture
SHOW MASTER STATUS;
-- +--------------------+----------+
-- | File               | Position |
-- +--------------------+----------+
-- | mariadb-bin.000042 | 8934     |
-- +--------------------+----------+
```

**Sur le Replica** :

```sql
-- Positions de réplication
SELECT 
  Master_Log_File AS 'Primary File',
  Read_Master_Log_Pos AS 'IO Thread Read',
  Relay_Master_Log_File AS 'SQL Thread File',
  Exec_Master_Log_Pos AS 'SQL Thread Exec'
FROM information_schema.SLAVE_STATUS\G

-- *************************** 1. row ***************************
--      Primary File: mariadb-bin.000042
--   IO Thread Read: 8934
-- SQL Thread File: mariadb-bin.000042
-- SQL Thread Exec: 8934
```

**Différence entre Read et Exec** :

```
Primary Binlog:  mariadb-bin.000042
                 ═══════════════════════════════════════════
                 0                4567                8934
                 │                │                   │
                 │                │                   └─ Position actuelle Primary
                 │                │
                 │                └─ Exec_Master_Log_Pos (SQL Thread)
                 │                   ↑ Événements exécutés
                 │
                 └─ Read_Master_Log_Pos (IO Thread)
                    ↑ Événements reçus (dans relay log)

Lag = Read_Master_Log_Pos - Exec_Master_Log_Pos
```

💡 **Interprétation** :
- **Read = Exec** : Replica parfaitement synchronisé
- **Read > Exec** : Relay log contient des événements non encore appliqués (lag)
- **Read << Primary** : IO Thread en retard (problème réseau ou binlog trop rapide)

### Rotation des fichiers binlog

**Rotation automatique** :

```ini
[mysqld]
# Taille max avant rotation (100MB par défaut)
max_binlog_size = 100M
```

**Rotation manuelle** :

```sql
-- Forcer une nouvelle rotation
FLUSH LOGS;

-- Nouveau fichier créé
SHOW MASTER STATUS;
-- +--------------------+----------+
-- | File               | Position |
-- +--------------------+----------+
-- | mariadb-bin.000043 | 4        | ← Nouveau fichier
-- +--------------------+----------+
```

**Impact sur la réplication** :

```
Timeline de rotation :

T0: Primary écrit dans mariadb-bin.000042 position 99999900
T1: Primary atteint max_binlog_size
T2: Primary crée mariadb-bin.000043
T3: Primary écrit ROTATE_EVENT dans 000042
T4: Primary continue dans 000043 position 4

Replica:
T5: IO Thread lit ROTATE_EVENT
T6: IO Thread bascule automatiquement sur 000043
T7: Continue la lecture depuis position 4

→ Transparent pour le Replica !
```

### Purge des anciens binlogs

**Purge automatique** :

```ini
[mysqld]
# Conserver 7 jours de binlogs
expire_logs_days = 7

# Ou depuis MariaDB 10.6 (plus précis)
binlog_expire_logs_seconds = 604800  # 7 jours
```

**Purge manuelle** :

```sql
-- Purger tous les binlogs avant un certain fichier
PURGE BINARY LOGS TO 'mariadb-bin.000040';

-- Purger tous les binlogs avant une certaine date
PURGE BINARY LOGS BEFORE '2025-12-06 14:30:00';

-- Purger tous sauf les N plus récents (MariaDB 10.1+)
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;
```

⚠️ **ATTENTION CRITIQUE** :

```sql
-- NE JAMAIS purger un binlog encore utilisé par un Replica !

-- 1. Vérifier les positions des Replicas
SELECT 
  Host,
  Master_Log_File,
  Read_Master_Log_Pos
FROM information_schema.SLAVE_HOSTS;

-- 2. Identifier le fichier le plus ancien utilisé
SELECT MIN(Master_Log_File) AS oldest_binlog_in_use
FROM information_schema.SLAVE_HOSTS;
-- +------------------------+
-- | oldest_binlog_in_use   |
-- +------------------------+
-- | mariadb-bin.000038     |
-- +------------------------+

-- 3. Purger UNIQUEMENT avant ce fichier
PURGE BINARY LOGS TO 'mariadb-bin.000038';
```

**Script de purge sécurisé** :

```bash
#!/bin/bash
# safe_purge_binlogs.sh

# Obtenir le binlog le plus ancien utilisé
OLDEST=$(mysql -N -e "
  SELECT IFNULL(MIN(Master_Log_File), 
    (SELECT MIN(Log_name) FROM information_schema.BINARY_LOGS))
  FROM information_schema.SLAVE_HOSTS
")

if [ -n "$OLDEST" ]; then
  echo "Oldest binlog in use by replicas: $OLDEST"
  echo "Purging binlogs before $OLDEST..."
  mysql -e "PURGE BINARY LOGS TO '$OLDEST';"
else
  echo "No replicas connected, skipping purge for safety"
fi
```

---

## Avantages et Limitations

### Avantages

✅ **1. Simplicité conceptuelle**

Le système de coordonnées (fichier, position) est **intuitif** :

```
Position = Pointeur dans un fichier séquentiel
         = Facile à comprendre et débugger
```

✅ **2. Compatibilité universelle**

- Supporté par toutes les versions de MariaDB et MySQL
- Pas de migration nécessaire depuis d'anciennes versions
- Outils tiers (pt-table-checksum, etc.) 100% compatibles

✅ **3. Overhead minimal**

```sql
-- Aucun metadata supplémentaire dans le binlog
-- Pas de GTID_EVENT (économie d'espace)

-- Comparaison taille binlog :
-- Position-based : 1073741824 bytes
-- GTID-based     : 1085839360 bytes (+1.1%)
```

✅ **4. Debugging facilité**

```bash
# Inspecter exactement une position
mysqlbinlog mariadb-bin.000042 \
  --start-position=4567 \
  --stop-position=8934

# Très pratique pour forensics
```

### Limitations

❌ **1. Failover complexe**

**Scénario problématique** :

```
Topologie initiale :
┌──────────┐
│ Primary  │  mariadb-bin.000042 pos 8934
└────┬─────┘
     │
     ├─────────────┬─────────────┐
     │             │             │
     ▼             ▼             ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│Replica1 │   │Replica2 │   │Replica3 │
│pos 8934 │   │pos 8920 │   │pos 8890 │
└─────────┘   └─────────┘   └─────────┘

Primary CRASH ! 💥

Promotion de Replica1 en nouveau Primary :
┌──────────┐
│Replica1  │  ← Nouveau Primary
│(ex-Replica1)│  Binlog : replica1-bin.000001 pos 4
└────┬─────┘
     │
     ├─────────────┐
     │             │
     ▼             ▼
┌─────────┐   ┌─────────┐
│Replica2 │   │Replica3 │
└─────────┘   └─────────┘

PROBLÈME : Replica2 et Replica3 cherchent encore :
  mariadb-bin.000042 pos 8920/8890
  
  Mais le nouveau Primary a :
  replica1-bin.000001 pos 4
  
  → Positions INCOMPATIBLES !
```

**Solution manuelle (complexe)** :

```sql
-- 1. Sur Replica2 : Calculer la position équivalente
-- Très complexe, nécessite :
-- - Comparer les événements manquants
-- - Identifier l'équivalent dans replica1-bin.000001
-- - Espérer que les événements soient identiques

-- 2. Reconfigurer manuellement
CHANGE MASTER TO
  MASTER_HOST = 'replica1.example.com',  -- Nouveau Primary
  MASTER_LOG_FILE = 'replica1-bin.000001',  -- ???? Quelle position ?
  MASTER_LOG_POS = ???;  -- Impossible à calculer avec certitude
```

💡 **Avec GTID**, ce problème n'existe pas (calcul automatique de la position).

❌ **2. Topologies complexes difficiles**

**Multi-source** :

```sql
-- Réplication depuis 2 sources
CHANGE MASTER 'source1' TO
  MASTER_LOG_FILE = 'source1-bin.000042',
  MASTER_LOG_POS = 4567;

CHANGE MASTER 'source2' TO
  MASTER_LOG_FILE = 'source2-bin.000123',  -- Fichiers différents
  MASTER_LOG_POS = 9876;                    -- Positions différentes

-- Gestion manuelle de 2 jeux de coordonnées !
```

**Circular replication** :

```
Primary A ←→ Primary B
(Complexe à configurer et à maintenir sans GTID)
```

❌ **3. Points de défaillance**

**Corruption de binlog** :

```bash
# Si un fichier binlog est corrompu
ls -lh /var/log/mysql/mariadb-bin.000042
# -rw-r----- 1 mysql mysql 0 Dec 13 14:32 mariadb-bin.000042
# ↑ Fichier vide ou tronqué !

# Réplication bloquée
SHOW SLAVE STATUS\G
# Last_IO_Error: Got fatal error 1236 from master when reading data from binary log
```

**Avec positions** : Très difficile de reprendre
**Avec GTID** : Peut sauter les transactions corrompues et continuer

❌ **4. Maintenance délicate**

**Ajouter un nouveau Replica** :

```
Avec positions :
1. Faire un dump du Primary avec --master-data
2. Noter la position exacte (mariadb-bin.X, pos Y)
3. Restaurer sur le nouveau Replica
4. Configurer avec les positions notées
5. Espérer qu'aucune rotation n'a eu lieu entre-temps

Avec GTID :
1. Faire un dump du Primary
2. CHANGE MASTER TO MASTER_AUTO_POSITION=1
3. C'est tout ! (positions calculées automatiquement)
```

---

## Opérations Avancées

### 1. Skip d'événements

**Situation** : Erreur de réplication due à un événement problématique

```sql
SHOW SLAVE STATUS\G
-- Slave_SQL_Running: No
-- Last_SQL_Error: Error 'Duplicate entry '123' for key 'PRIMARY'' on query

-- Solution : Ignorer cet événement (avec PRUDENCE !)
STOP SLAVE SQL_THREAD;

SET GLOBAL sql_slave_skip_counter = 1;  -- Skip 1 événement

START SLAVE SQL_THREAD;

-- Vérifier
SHOW SLAVE STATUS\G
-- Slave_SQL_Running: Yes
-- Exec_Master_Log_Pos: 5678  (position avancée de 1 événement)
```

⚠️ **DANGER** : Ignorer des événements peut créer des **incohérences de données** !

**Alternative sécurisée** : Fixer la cause racine

```sql
-- Identifier l'événement problématique
SHOW BINLOG EVENTS IN 'mariadb-bin.000042' 
  FROM 4567 LIMIT 10;

-- Exemple : Duplicate entry
-- → Vérifier si la ligne existe déjà sur le Replica
SELECT * FROM users WHERE id = 123;

-- Si elle existe avec les mêmes données : safe to skip
-- Si elle existe avec données différentes : CORRUPTION !
```

### 2. Rejouer des événements

**Situation** : Rejouer une transaction spécifique

```bash
# Extraire un événement spécifique
mysqlbinlog mariadb-bin.000042 \
  --start-position=4567 \
  --stop-position=4890 \
  > transaction.sql

# Inspecter
cat transaction.sql
# SET TIMESTAMP=1702456789/*!*/;
# BEGIN
# /*!*/;
# use `mydb`/*!*/;
# INSERT INTO users (name) VALUES ('Alice')
# /*!*/;
# COMMIT
# /*!*/;

# Rejouer sur le Replica (ou autre serveur)
mariadb < transaction.sql
```

### 3. Point-in-Time Recovery (PITR)

**Scénario** : Restaurer jusqu'à un moment précis avant une erreur

```bash
# 1. Restaurer le dernier backup complet
mariadb < backup_2025-12-13_02-00.sql

# 2. Identifier la position à restaurer
# Exemple : Erreur à 14:35:00
# Trouver la position correspondante
mysqlbinlog mariadb-bin.000042 \
  --start-datetime="2025-12-13 02:00:00" \
  --stop-datetime="2025-12-13 14:34:59" \
  | grep "^# at"

# 3. Rejouer les binlogs jusqu'à cette position
mysqlbinlog mariadb-bin.000042 \
  --start-position=4 \
  --stop-position=8900 \  # Juste avant l'erreur
  | mariadb

# Base restaurée jusqu'à 14:34:59 !
```

### 4. Changement de position manuel

**Situation** : Besoin d'ajuster la position (après skip, etc.)

```sql
-- Arrêter la réplication
STOP SLAVE;

-- Changer la position
CHANGE MASTER TO
  MASTER_LOG_FILE = 'mariadb-bin.000042',
  MASTER_LOG_POS = 5678;  -- Nouvelle position

-- Redémarrer
START SLAVE;

-- Vérifier
SHOW SLAVE STATUS\G
-- Read_Master_Log_Pos: 5678  ✓
```

**Use cases** :
- Après avoir manuellement fixé une erreur
- Pour ignorer une plage d'événements
- Pour resynchroniser après corruption

---

## Monitoring Spécifique aux Positions

### Métriques essentielles

**1. Écart entre IO et SQL Thread** :

```sql
-- Query de monitoring
SELECT 
  Master_Log_File AS current_binlog,
  Read_Master_Log_Pos AS io_position,
  Exec_Master_Log_Pos AS sql_position,
  Read_Master_Log_Pos - Exec_Master_Log_Pos AS position_lag_bytes
FROM information_schema.SLAVE_STATUS;

-- +--------------------+-------------+--------------+-------------------+
-- | current_binlog     | io_position | sql_position | position_lag_bytes|
-- +--------------------+-------------+--------------+-------------------+
-- | mariadb-bin.000042 | 8934        | 7890         | 1044              |
-- +--------------------+-------------+--------------+-------------------+
```

**Interprétation** :
- `position_lag_bytes = 0` : Parfaitement synchronisé
- `position_lag_bytes > 0` : Relay log contient des événements non appliqués
- `position_lag_bytes > 10MB` : Lag significatif, investiguer

**2. Distance par rapport au Primary** :

```sql
-- Requête complexe nécessitant une connexion au Primary
-- À exécuter depuis un serveur de monitoring

-- 1. Obtenir position Primary
SET @primary_pos = (
  SELECT Position 
  FROM primary_server.information_schema.MASTER_STATUS 
  LIMIT 1
);

-- 2. Obtenir position Replica
SET @replica_pos = (
  SELECT Exec_Master_Log_Pos 
  FROM replica_server.information_schema.SLAVE_STATUS 
  LIMIT 1
);

-- 3. Calculer le lag (même fichier binlog uniquement)
SELECT 
  @primary_pos - @replica_pos AS lag_bytes,
  ROUND((@primary_pos - @replica_pos) / 1024 / 1024, 2) AS lag_mb;
```

⚠️ **Limitation** : Cette méthode ne fonctionne que si Primary et Replica sont sur le **même fichier binlog**.

**3. Fichiers binlog en retard** :

```sql
-- Sur le Replica
SELECT 
  Master_Log_File AS replica_reading,
  (SELECT File FROM primary.information_schema.MASTER_STATUS) AS primary_writing,
  CASE 
    WHEN Master_Log_File < (SELECT File FROM primary.information_schema.MASTER_STATUS)
    THEN 'LAGGING (multiple files behind)'
    WHEN Master_Log_File = (SELECT File FROM primary.information_schema.MASTER_STATUS)
    THEN 'CURRENT (same file)'
    ELSE 'AHEAD (unexpected!)'
  END AS status
FROM information_schema.SLAVE_STATUS;
```

### Dashboard Grafana

**Panel : Replication Position Lag**

```promql
# Prometheus query
(mariadb_master_position - mariadb_slave_position) / 1024 / 1024

# Affiche le lag en MB
# Alert : lag > 100MB
```

**Panel : Binlog File Distance**

```promql
# Nombre de fichiers binlog d'écart
mariadb_master_binlog_file - mariadb_slave_binlog_file

# Alert : distance > 2 fichiers
```

---

## Migration vers GTID

La réplication par positions étant limitée, la migration vers GTID est souvent souhaitable.

### Approche en douceur (zero downtime)

**Étape 1 : Activer GTID en mode hybride**

```ini
# Sur Primary ET Replicas
[mysqld]
# Activer GTID sans le rendre obligatoire
gtid_domain_id = 0
log_slave_updates = ON  # Requis pour GTID
```

```sql
-- Redémarrer les serveurs (un par un)
-- Le Primary commence à générer des GTID
-- Mais accepte encore les connexions par positions
```

**Étape 2 : Vérifier la génération de GTID**

```sql
-- Sur le Primary
SELECT @@gtid_current_pos;
-- +---------------------+
-- | @@gtid_current_pos  |
-- +---------------------+
-- | 0-1-1000            |
-- +---------------------+

-- Chaque transaction reçoit maintenant un GTID
```

**Étape 3 : Basculer les Replicas vers GTID (un par un)**

```sql
-- Sur Replica1
STOP SLAVE;

CHANGE MASTER TO
  MASTER_USE_GTID = slave_pos;  -- Utiliser GTID

START SLAVE;

-- Vérifier
SHOW SLAVE STATUS\G
-- Using_Gtid: Slave_Pos  ✓
-- Gtid_IO_Pos: 0-1-1000
```

**Étape 4 : Activer GTID strict (optionnel)**

```sql
-- Sur Primary et Replicas (après migration complète)
SET GLOBAL gtid_strict_mode = ON;

-- Refuse désormais les transactions sans GTID
```

### Rollback (si nécessaire)

```sql
-- Revenir aux positions
STOP SLAVE;

CHANGE MASTER TO
  MASTER_USE_GTID = no,
  MASTER_LOG_FILE = 'mariadb-bin.000042',
  MASTER_LOG_POS = 8934;

START SLAVE;
```

---

## Troubleshooting Spécifique

### Problème 1 : Position non trouvée

```sql
SHOW SLAVE STATUS\G
-- Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 
-- 'Could not find first log file name in binary log index file'
```

**Cause** : Le fichier binlog demandé a été purgé sur le Primary.

**Solution** :

```sql
-- 1. Vérifier les binlogs disponibles sur le Primary
-- Sur Primary :
SHOW BINARY LOGS;
-- Le fichier demandé n'existe plus

-- 2. Resynchroniser depuis la position actuelle
-- Sur Replica :
STOP SLAVE;

-- Obtenir position actuelle du Primary
-- Sur Primary :
SHOW MASTER STATUS;
-- +--------------------+----------+
-- | File               | Position |
-- +--------------------+----------+
-- | mariadb-bin.000045 | 4        |
-- +--------------------+----------+

-- Sur Replica : Reconfigurer
CHANGE MASTER TO
  MASTER_LOG_FILE = 'mariadb-bin.000045',
  MASTER_LOG_POS = 4;

START SLAVE;

-- ⚠️ ATTENTION : Perte des événements entre 000042 et 000045 !
-- → Envisager un dump/restore complet
```

### Problème 2 : Mauvaise position après restauration

```sql
-- Après restauration d'un dump
START SLAVE;

SHOW SLAVE STATUS\G
-- Last_SQL_Error: Could not execute Update_rows event on table mydb.users; 
-- Duplicate entry '123' for key 'PRIMARY'
```

**Cause** : Position de départ incorrecte (événements déjà appliqués).

**Solution** :

```bash
# Vérifier la position dans le dump
grep "CHANGE MASTER" full_dump.sql
# -- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000042', MASTER_LOG_POS=4567;

# Comparer avec la position actuelle du Primary
# Si Primary est à 000042:8934, et dump à 000042:4567
# → 4367 bytes d'événements manquent

# Appliquer manuellement
mysqlbinlog mariadb-bin.000042 \
  --start-position=4567 \
  --stop-position=8934 \
  | mariadb

# Puis configurer la position actuelle
CHANGE MASTER TO
  MASTER_LOG_FILE = 'mariadb-bin.000042',
  MASTER_LOG_POS = 8934;
```

### Problème 3 : Binlog corrompu

```sql
SHOW SLAVE STATUS\G
-- Last_IO_Error: Binlog has bad magic number; 
-- It's not a binary log file that can be used by this version of MariaDB
```

**Diagnostic** :

```bash
# Vérifier l'intégrité du fichier binlog
mysqlbinlog /var/log/mysql/mariadb-bin.000042
# ERROR: Error in Log_event::read_log_event(): 'read error', data_len: 12345, event_type: 2
```

**Solution** :

```sql
-- 1. Identifier l'événement corrompu
mysqlbinlog mariadb-bin.000042 2>&1 | grep -i error
-- ERROR at position 8900: ...

-- 2. Skipper jusqu'après la corruption
STOP SLAVE;

CHANGE MASTER TO
  MASTER_LOG_FILE = 'mariadb-bin.000042',
  MASTER_LOG_POS = 8934;  -- Juste après la corruption

START SLAVE;

-- 3. Vérifier les données perdues
-- Comparer les checksums de tables entre Primary et Replica
```

---

## ✅ Points clés à retenir

1. **Coordonnées binlog** : Système (Nom fichier, Position) identifiant chaque événement

2. **Simplicité vs Flexibilité** : Simple à comprendre mais limité pour topologies complexes

3. **Failover manuel** : Nécessite calcul de positions équivalentes (complexe et error-prone)

4. **Purge prudente** : Toujours vérifier que les Replicas n'utilisent pas un binlog avant de le purger

5. **Position = End_log_pos** : Utiliser la fin de l'événement précédent, pas le début

6. **PITR possible** : Point-in-Time Recovery en rejouant binlogs jusqu'à une position

7. **Monitoring positions** : Surveiller écart IO/SQL Thread et distance Primary/Replica

8. **Migration GTID** : Possible en mode hybride sans downtime

9. **Limitations topologies** : Multi-source, cascade, circular difficiles à gérer

10. **GTID recommandé** : Pour nouveaux déploiements, sauf contraintes legacy

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Binary Log](https://mariadb.com/kb/en/binary-log/)
- [📖 Binary Log Formats](https://mariadb.com/kb/en/binary-log-formats/)
- [📖 SHOW MASTER STATUS](https://mariadb.com/kb/en/show-master-status/)
- [📖 SHOW SLAVE STATUS](https://mariadb.com/kb/en/show-slave-status/)
- [📖 CHANGE MASTER TO](https://mariadb.com/kb/en/change-master-to/)

### Outils

- **mysqlbinlog** : Lecture et analyse des binary logs
- **pt-table-checksum** (Percona Toolkit) : Vérification cohérence Primary/Replica
- **pt-table-sync** : Synchronisation de tables divergentes

### Articles techniques

- [🔗 Binary Log Internals](https://dev.mysql.com/doc/internals/en/binary-log.html)
- [🔗 Understanding MySQL Binary Logs](https://www.percona.com/blog/understanding-mysql-binary-logs/)

---

## ➡️ Section suivante

**13.4 GTID (Global Transaction Identifier)** : L'évolution moderne de la réplication avec identifiants uniques de transactions, failover automatisé, topologies complexes simplifiées, et élimination des problèmes de positions.

La section suivante vous montrera pourquoi **GTID est l'avenir de la réplication MariaDB** !

---


⏭️ [GTID (Global Transaction Identifier)](/13-replication/04-gtid.md)

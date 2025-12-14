🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.1 - Requêtes d'Administration

> **Niveau** : Intermédiaire à Avancé  
> **Type** : Requêtes prêtes à l'emploi  
> **Usage** : Administration quotidienne, troubleshooting

---

## 📖 Introduction

Cette section fournit des **requêtes SQL essentielles** pour l'administration quotidienne de MariaDB. Toutes les requêtes sont commentées, testées et prêtes à être utilisées en production.

### 🎯 Catégories Couvertes

- 🔒 **Locks et Verrous** : Identifier et résoudre les blocages
- 🔄 **Processus Actifs** : Surveiller connexions et requêtes
- 📦 **Tailles Tables** : Analyser volumétries et croissance
- 👥 **Gestion Utilisateurs** : Auditer privilèges et connexions
- ⚙️ **Configuration** : Inspecter variables et statuts

---

## 🔒 LOCKS ET VERROUS

### 1.1 - Lister Toutes les Transactions Actives
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Transactions InnoDB actives
-- ============================================================
-- Description :
-- Liste toutes les transactions en cours avec détails
--
-- Cas d'usage :
-- - Identifier transactions longues
-- - Détecter potentiels blocages
-- - Monitoring santé transactionnelle
-- ============================================================

SELECT 
  trx_id AS transaction_id,
  trx_state AS state,
  trx_started AS started_at,
  TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_seconds,
  trx_requested_lock_id AS requested_lock,
  trx_wait_started AS wait_started,
  trx_mysql_thread_id AS thread_id,
  trx_query AS current_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;
```

**Résultat exemple** :
```
+----------------+---------+---------------------+------------------+----------------+--------------+-----------+------------------+
| transaction_id | state   | started_at          | duration_seconds | requested_lock | wait_started | thread_id | current_query    |
+----------------+---------+---------------------+------------------+----------------+--------------+-----------+------------------+
| 847562         | RUNNING | 2025-12-14 10:30:15 |               45 | NULL           | NULL         |       123 | UPDATE orders... |
+----------------+---------+---------------------+------------------+----------------+--------------+-----------+------------------+
```

**Interprétation** :
- `duration_seconds` élevé (> 60s) : Transaction potentiellement problématique
- `requested_lock` non-NULL : Transaction en attente de verrou
- `state = LOCK WAIT` : Transaction bloquée

---

### 1.2 - Identifier les Transactions Bloquées (Lock Waits)
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Transactions bloquées et bloquantes
-- ============================================================
-- Description :
-- Affiche les transactions qui attendent un verrou et celles
-- qui les bloquent
--
-- Cas d'usage :
-- - Résoudre deadlocks
-- - Identifier requêtes à optimiser
-- - Décider quelle transaction tuer
-- ============================================================

SELECT 
  w.requesting_trx_id AS waiting_transaction,
  w.requested_lock_id AS waiting_for_lock,
  w.blocking_trx_id AS blocking_transaction,
  w.blocking_lock_id AS holding_lock,
  -- Informations sur transaction bloquée
  rt.trx_mysql_thread_id AS waiting_thread_id,
  rt.trx_query AS waiting_query,
  TIMESTAMPDIFF(SECOND, rt.trx_started, NOW()) AS waiting_duration_sec,
  -- Informations sur transaction bloquante
  bt.trx_mysql_thread_id AS blocking_thread_id,
  bt.trx_query AS blocking_query,
  TIMESTAMPDIFF(SECOND, bt.trx_started, NOW()) AS blocking_duration_sec
FROM information_schema.INNODB_LOCK_WAITS w
INNER JOIN information_schema.INNODB_TRX rt 
  ON w.requesting_trx_id = rt.trx_id
INNER JOIN information_schema.INNODB_TRX bt 
  ON w.blocking_trx_id = bt.trx_id
ORDER BY waiting_duration_sec DESC;
```

**Actions possibles** :
```sql
-- Tuer la transaction bloquante (avec précaution !)
KILL <blocking_thread_id>;

-- Ou juste la requête
KILL QUERY <blocking_thread_id>;
```

---

### 1.3 - Lister Tous les Verrous Actifs
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Verrous InnoDB actifs
-- ============================================================
-- Description :
-- Liste tous les verrous actuellement détenus par des transactions
--
-- Notes :
-- - LOCK_MODE : X (exclusive), S (shared), IX, IS
-- - LOCK_TYPE : RECORD, TABLE
-- ============================================================

SELECT 
  lock_id,
  lock_trx_id AS transaction_id,
  lock_mode,
  lock_type,
  lock_table,
  lock_index,
  lock_space,
  lock_page,
  lock_rec,
  lock_data
FROM information_schema.INNODB_LOCKS
ORDER BY lock_table, lock_index;
```

**Résultat exemple** :
```
+--------------+----------------+-----------+-----------+-----------------+------------+------------+-----------+----------+-----------+
| lock_id      | transaction_id | lock_mode | lock_type | lock_table      | lock_index | lock_space | lock_page | lock_rec | lock_data |
+--------------+----------------+-----------+-----------+-----------------+------------+------------+-----------+----------+-----------+
| 847562:23:4:5| 847562         | X         | RECORD    | `db`.`orders`   | PRIMARY    | 23         | 4         | 5        | 12345     |
+--------------+----------------+-----------+-----------+-----------------+------------+------------+-----------+----------+-----------+
```

---

### 1.4 - Détecter les Deadlocks Récents
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Dernière information deadlock
-- ============================================================
-- Description :
-- Extrait les informations du dernier deadlock détecté
--
-- Note :
-- Utilise SHOW ENGINE INNODB STATUS - parser manuellement
-- ou utiliser Performance Schema (si activé)
-- ============================================================

-- Méthode 1 : Via SHOW ENGINE (nécessite parsing)
SHOW ENGINE INNODB STATUS\G

-- Chercher section :
-- ------------------------
-- LATEST DETECTED DEADLOCK
-- ------------------------

-- Méthode 2 : Via variable système
SELECT @@innodb_print_all_deadlocks;
-- Si = ON, deadlocks sont loggés dans error log

-- Méthode 3 : Compter les deadlocks
SELECT 
  VARIABLE_VALUE AS total_deadlocks
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Innodb_deadlocks';
```

**Prévention deadlocks** :
```sql
-- Bonnes pratiques à vérifier :
-- 1. Transactions courtes
-- 2. Ordre cohérent d'accès aux tables
-- 3. Index appropriés
-- 4. Niveau d'isolation adapté
SELECT @@tx_isolation;  -- Vérifier isolation level
```

---

### 1.5 - Locks au Niveau Table (Table Locks)
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Verrous de table
-- ============================================================
-- Description :
-- Affiche les verrous au niveau table (MyISAM, metadata locks)
--
-- Cas d'usage :
-- - Tables MyISAM bloquées
-- - ALTER TABLE en cours
-- - FLUSH TABLES bloqué
-- ============================================================

SELECT 
  thread_id,
  processlist_id,
  processlist_user AS user,
  processlist_host AS host,
  processlist_db AS db,
  processlist_command AS command,
  processlist_time AS time_sec,
  processlist_state AS state,
  processlist_info AS query
FROM performance_schema.threads
WHERE processlist_state LIKE '%lock%'
   OR processlist_state LIKE '%wait%'
ORDER BY processlist_time DESC;
```

**Statistiques table locks** :
```sql
-- Nombre de locks attendus vs obtenus immédiatement
SELECT 
  VARIABLE_NAME,
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
  'Table_locks_immediate',
  'Table_locks_waited'
);

-- Calcul ratio attente
SELECT 
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Table_locks_waited') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Table_locks_immediate') * 100, 4
  ) AS table_lock_wait_ratio_pct;
```

---

## 🔄 PROCESSUS ACTIFS

### 2.1 - Lister Tous les Processus (PROCESSLIST)
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Processus actifs détaillés
-- ============================================================
-- Description :
-- Vue complète de toutes les connexions actives
--
-- Cas d'usage :
-- - Surveillance quotidienne
-- - Identifier connexions inactives
-- - Détecter requêtes longues
-- ============================================================

SELECT 
  ID AS process_id,
  USER AS user,
  HOST AS host,
  DB AS database_name,
  COMMAND AS command,
  TIME AS duration_seconds,
  STATE AS state,
  INFO AS query_text,
  -- Calculer depuis quand la connexion existe
  TIME AS connection_age_sec
FROM information_schema.PROCESSLIST
ORDER BY TIME DESC, ID;
```

**Version condensée** :
```sql
-- Top 10 processus les plus anciens
SELECT 
  ID,
  USER,
  DB,
  TIME AS seconds,
  STATE,
  LEFT(INFO, 80) AS query_preview
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC
LIMIT 10;
```

---

### 2.2 - Requêtes Longues (> N Secondes)
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Requêtes dépassant un seuil de temps
-- ============================================================
-- Description :
-- Identifie requêtes qui s'exécutent depuis plus de N secondes
--
-- Paramètres :
-- - Modifier TIME > 60 selon besoin (ici 60 secondes)
-- ============================================================

SELECT 
  ID AS process_id,
  USER AS user,
  HOST AS host,
  DB AS database_name,
  TIME AS duration_seconds,
  ROUND(TIME / 60, 2) AS duration_minutes,
  STATE AS state,
  INFO AS query
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND TIME > 60  -- ← Seuil en secondes
ORDER BY TIME DESC;
```

**Avec catégorisation** :
```sql
-- Classifier par durée
SELECT 
  CASE 
    WHEN TIME < 10 THEN '< 10 sec'
    WHEN TIME < 60 THEN '10-60 sec'
    WHEN TIME < 300 THEN '1-5 min'
    WHEN TIME < 600 THEN '5-10 min'
    ELSE '> 10 min'
  END AS duration_category,
  COUNT(*) AS query_count,
  GROUP_CONCAT(ID) AS process_ids
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
GROUP BY duration_category
ORDER BY 
  FIELD(duration_category, '< 10 sec', '10-60 sec', '1-5 min', '5-10 min', '> 10 min');
```

---

### 2.3 - Connexions par Utilisateur
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Statistiques connexions par utilisateur
-- ============================================================
-- Description :
-- Compte le nombre de connexions actives par utilisateur
--
-- Cas d'usage :
-- - Détecter utilisateurs monopolisant connexions
-- - Vérifier pool de connexions applicatif
-- - Audit sécurité
-- ============================================================

SELECT 
  USER AS username,
  COUNT(*) AS total_connections,
  SUM(CASE WHEN COMMAND = 'Sleep' THEN 1 ELSE 0 END) AS idle_connections,
  SUM(CASE WHEN COMMAND != 'Sleep' THEN 1 ELSE 0 END) AS active_connections,
  MAX(TIME) AS longest_query_sec,
  AVG(TIME) AS avg_query_time_sec
FROM information_schema.PROCESSLIST
GROUP BY USER
ORDER BY total_connections DESC;
```

**Résultat exemple** :
```
+----------+-------------------+------------------+--------------------+-------------------+-------------------+
| username | total_connections | idle_connections | active_connections | longest_query_sec | avg_query_time_sec|
+----------+-------------------+------------------+--------------------+-------------------+-------------------+
| webapp   |                45 |               42 |                  3 |               234 |              15.6 |
| admin    |                 3 |                1 |                  2 |                12 |               8.3 |
+----------+-------------------+------------------+--------------------+-------------------+-------------------+
```

---

### 2.4 - Connexions par Base de Données
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Statistiques connexions par database
-- ============================================================

SELECT 
  DB AS database_name,
  COUNT(*) AS connection_count,
  COUNT(DISTINCT USER) AS distinct_users,
  SUM(CASE WHEN COMMAND != 'Sleep' THEN 1 ELSE 0 END) AS active_queries,
  MAX(TIME) AS longest_query_sec
FROM information_schema.PROCESSLIST
WHERE DB IS NOT NULL
GROUP BY DB
ORDER BY connection_count DESC;
```

---

### 2.5 - Connexions Inactives (Sleep)
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Connexions inactives depuis longtemps
-- ============================================================
-- Description :
-- Liste connexions en état Sleep depuis plus de N secondes
--
-- Cas d'usage :
-- - Identifier connexions zombies
-- - Optimiser timeout wait_timeout
-- - Nettoyer connexions inutiles
--
-- Note :
-- Vérifier wait_timeout et interactive_timeout
-- ============================================================

SELECT 
  ID AS process_id,
  USER AS user,
  HOST AS host,
  DB AS database_name,
  TIME AS idle_seconds,
  ROUND(TIME / 60, 2) AS idle_minutes,
  ROUND(TIME / 3600, 2) AS idle_hours
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
  AND TIME > 300  -- ← Idle > 5 minutes
ORDER BY TIME DESC;
```

**Tuer connexions inactives** :
```sql
-- Générer commandes KILL pour connexions idle > 1 heure
SELECT 
  CONCAT('KILL ', ID, ';') AS kill_command
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
  AND TIME > 3600
  AND USER != 'root';  -- Sécurité : ne pas tuer root

-- Copier/coller les commandes générées ou exécuter via script
```

---

### 2.6 - Connexions par Host
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Connexions par adresse IP/hostname
-- ============================================================
-- Description :
-- Identifie d'où viennent les connexions
--
-- Cas d'usage :
-- - Sécurité : détecter connexions suspectes
-- - Identifier serveurs applicatifs
-- - Vérifier load balancing
-- ============================================================

SELECT 
  SUBSTRING_INDEX(HOST, ':', 1) AS client_host,  -- Enlever port
  COUNT(*) AS connection_count,
  COUNT(DISTINCT USER) AS distinct_users,
  GROUP_CONCAT(DISTINCT USER) AS users_list
FROM information_schema.PROCESSLIST
GROUP BY client_host
ORDER BY connection_count DESC;
```

---

## 📦 TAILLES DES TABLES

### 3.1 - Taille de Toutes les Tables d'une Base
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Tailles tables d'une base de données
-- ============================================================
-- Description :
-- Liste toutes les tables avec taille données + index
--
-- Paramètres :
-- - Remplacer 'YOUR_DATABASE' par nom de la base
-- ============================================================

SELECT 
  TABLE_NAME AS table_name,
  TABLE_ROWS AS row_count_approx,
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_size_mb,
  ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_size_mb,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_size_mb,
  ROUND(INDEX_LENGTH / DATA_LENGTH * 100, 2) AS index_ratio_pct,
  ENGINE AS engine
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;
```

**Résultat exemple** :
```
+-------------+-------------------+-------------+--------------+---------------+-----------------+--------+
| table_name  | row_count_approx  | data_size_mb| index_size_mb| total_size_mb | index_ratio_pct | engine |
+-------------+-------------------+-------------+--------------+---------------+-----------------+--------+
| orders      |          1250000  |       234.12|       111.55 |        345.67 |           47.65 | InnoDB |
| customers   |           500000  |        89.23|        34.22 |        123.45 |           38.35 | InnoDB |
+-------------+-------------------+-------------+--------------+---------------+-----------------+--------+
```

**Interprétation** :
- `index_ratio_pct` > 50% : Beaucoup d'index, vérifier utilité
- `row_count_approx` : Approximatif pour InnoDB (exact après ANALYZE TABLE)

---

### 3.2 - Top N Tables les Plus Volumineuses
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Top 20 plus grosses tables (toutes bases)
-- ============================================================

SELECT 
  TABLE_SCHEMA AS database_name,
  TABLE_NAME AS table_name,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS size_gb,
  TABLE_ROWS AS row_count_approx,
  ENGINE AS engine
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 20;
```

---

### 3.3 - Taille Totale par Base de Données
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Taille de chaque base de données
-- ============================================================
-- Description :
-- Somme des tailles de toutes les tables par database
--
-- Cas d'usage :
-- - Capacity planning
-- - Audit espace disque
-- - Identifier bases à archiver
-- ============================================================

SELECT 
  TABLE_SCHEMA AS database_name,
  COUNT(*) AS table_count,
  ROUND(SUM(DATA_LENGTH) / 1024 / 1024, 2) AS data_size_mb,
  ROUND(SUM(INDEX_LENGTH) / 1024 / 1024, 2) AS index_size_mb,
  ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_size_mb,
  ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS total_size_gb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema')
GROUP BY TABLE_SCHEMA
ORDER BY SUM(DATA_LENGTH + INDEX_LENGTH) DESC;
```

**Résultat exemple** :
```
+----------------+-------------+-------------+--------------+---------------+--------------+
| database_name  | table_count | data_size_mb| index_size_mb| total_size_mb | total_size_gb|
+----------------+-------------+-------------+--------------+---------------+--------------+
| production_db  |          45 |     2345.67 |       567.89 |       2913.56 |         2.84 |
| analytics_db   |          23 |      567.12 |       123.45 |        690.57 |         0.67 |
+----------------+-------------+-------------+--------------+---------------+--------------+
```

---

### 3.4 - Croissance Tables (Nécessite Historique)
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Analyse croissance tables
-- ============================================================
-- Description :
-- Compare taille actuelle avec snapshot précédent
--
-- Prérequis :
-- - Créer table table_size_history
-- - Alimenter régulièrement (cron quotidien)
-- ============================================================

-- 1. Créer table historique (une fois)
CREATE TABLE IF NOT EXISTS monitoring.table_size_history (
  id INT AUTO_INCREMENT PRIMARY KEY,
  table_schema VARCHAR(64),
  table_name VARCHAR(64),
  table_rows BIGINT,
  data_length BIGINT,
  index_length BIGINT,
  captured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_schema_table (table_schema, table_name),
  INDEX idx_captured (captured_at)
) ENGINE=InnoDB;

-- 2. Snapshot quotidien (exécuter via cron)
INSERT INTO monitoring.table_size_history 
  (table_schema, table_name, table_rows, data_length, index_length)
SELECT 
  TABLE_SCHEMA,
  TABLE_NAME,
  TABLE_ROWS,
  DATA_LENGTH,
  INDEX_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema', 'mysql', 'sys');

-- 3. Requête croissance (30 derniers jours)
SELECT 
  current.table_schema AS db_name,
  current.table_name,
  old.table_rows AS rows_30d_ago,
  current.table_rows AS rows_now,
  (current.table_rows - old.table_rows) AS row_growth,
  ROUND((current.data_length - old.data_length) / 1024 / 1024, 2) AS data_growth_mb,
  ROUND(
    ((current.data_length + current.index_length) - 
     (old.data_length + old.index_length)) / 1024 / 1024, 2
  ) AS total_growth_mb,
  ROUND(
    ((current.table_rows - old.table_rows) / old.table_rows) * 100, 2
  ) AS growth_rate_pct
FROM information_schema.TABLES current
LEFT JOIN (
  SELECT 
    table_schema,
    table_name,
    table_rows,
    data_length,
    index_length
  FROM monitoring.table_size_history
  WHERE captured_at >= CURDATE() - INTERVAL 30 DAY
    AND captured_at < CURDATE() - INTERVAL 29 DAY
  ORDER BY captured_at DESC
  LIMIT 1000
) old ON current.TABLE_SCHEMA = old.table_schema 
     AND current.TABLE_NAME = old.table_name
WHERE current.TABLE_SCHEMA = 'production_db'
  AND old.table_rows IS NOT NULL
  AND (current.table_rows - old.table_rows) > 0
ORDER BY total_growth_mb DESC;
```

---

### 3.5 - Tables Fragmentées (À Optimiser)
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Identifier tables fragmentées
-- ============================================================
-- Description :
-- Calcule le taux de fragmentation des tables InnoDB
--
-- Formule :
-- Fragmentation = (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH)) * 100
--
-- Cas d'usage :
-- - Identifier tables à OPTIMIZE
-- - Maintenance préventive
--
-- Note :
-- DATA_FREE = espace inutilisé dans les pages allouées
-- ============================================================

SELECT 
  TABLE_SCHEMA AS db_name,
  TABLE_NAME AS table_name,
  ENGINE AS engine,
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_size_mb,
  ROUND(DATA_FREE / 1024 / 1024, 2) AS data_free_mb,
  ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema', 'mysql', 'sys')
  AND ENGINE = 'InnoDB'
  AND DATA_FREE > 0
  AND (DATA_LENGTH + INDEX_LENGTH) > 10485760  -- Tables > 10 MB
HAVING fragmentation_pct > 10  -- Fragmentation > 10%
ORDER BY fragmentation_pct DESC;
```

**Action recommandée** :
```sql
-- Optimiser table fragmentée
OPTIMIZE TABLE database_name.table_name;

-- Ou rebuild via ALTER TABLE (plus rapide, online)
ALTER TABLE database_name.table_name ENGINE=InnoDB;
```

---

### 3.6 - Ratio Index/Data par Table
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Analyser ratio index vs données
-- ============================================================
-- Description :
-- Identifie tables avec trop ou pas assez d'index
--
-- Recommandations :
-- - Ratio < 10% : Probablement sous-indexée
-- - Ratio 10-50% : Normal
-- - Ratio > 50% : Possiblement sur-indexée
-- ============================================================

SELECT 
  TABLE_SCHEMA AS db_name,
  TABLE_NAME AS table_name,
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
  ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
  ROUND((INDEX_LENGTH / DATA_LENGTH) * 100, 2) AS index_ratio_pct,
  CASE 
    WHEN (INDEX_LENGTH / DATA_LENGTH) * 100 < 10 THEN '⚠️ Sous-indexée'
    WHEN (INDEX_LENGTH / DATA_LENGTH) * 100 > 50 THEN '⚠️ Sur-indexée'
    ELSE '✓ Normal'
  END AS status
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND TABLE_TYPE = 'BASE TABLE'
  AND DATA_LENGTH > 0
ORDER BY index_ratio_pct DESC;
```

---

## 👥 GESTION UTILISATEURS

### 4.1 - Lister Tous les Utilisateurs
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Liste complète des utilisateurs MariaDB
-- ============================================================

SELECT 
  User AS username,
  Host AS host,
  CASE 
    WHEN password = '' THEN '❌ SANS MOT DE PASSE'
    ELSE '✓ Protégé'
  END AS password_status,
  plugin AS auth_plugin,
  max_connections AS max_conn,
  max_user_connections AS max_user_conn
FROM mysql.user
ORDER BY User, Host;
```

**Version avec privilèges** :
```sql
-- Utilisateurs avec niveau d'accès
SELECT 
  User,
  Host,
  CASE 
    WHEN Super_priv = 'Y' THEN 'SUPER'
    WHEN Grant_priv = 'Y' THEN 'GRANT'
    WHEN Select_priv = 'Y' AND Insert_priv = 'Y' 
         AND Update_priv = 'Y' AND Delete_priv = 'Y' THEN 'READ/WRITE'
    WHEN Select_priv = 'Y' THEN 'READ ONLY'
    ELSE 'LIMITED'
  END AS access_level,
  CONCAT(
    IF(Select_priv='Y', 'SELECT ', ''),
    IF(Insert_priv='Y', 'INSERT ', ''),
    IF(Update_priv='Y', 'UPDATE ', ''),
    IF(Delete_priv='Y', 'DELETE ', '')
  ) AS basic_privileges
FROM mysql.user
ORDER BY 
  FIELD(access_level, 'SUPER', 'GRANT', 'READ/WRITE', 'READ ONLY', 'LIMITED'),
  User;
```

---

### 4.2 - Utilisateurs Sans Mot de Passe
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- AUDIT SÉCURITÉ : Utilisateurs sans mot de passe
-- ============================================================
-- Description :
-- Identifie comptes vulnérables sans protection
--
-- Action :
-- Définir mot de passe ou supprimer compte
-- ============================================================

SELECT 
  User AS username,
  Host AS host,
  '⚠️ SÉCURITÉ CRITIQUE' AS risk_level
FROM mysql.user
WHERE password = '' 
   OR authentication_string = ''
ORDER BY User, Host;

-- Corriger :
-- ALTER USER 'user'@'host' IDENTIFIED BY 'strong_password';
```

---

### 4.3 - Privilèges d'un Utilisateur Spécifique
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Afficher tous les privilèges d'un utilisateur
-- ============================================================

-- Méthode 1 : Via SHOW GRANTS
SHOW GRANTS FOR 'username'@'hostname';

-- Méthode 2 : Via tables système (plus détaillé)
-- Privilèges globaux
SELECT 
  'GLOBAL' AS privilege_level,
  User,
  Host,
  CONCAT(
    IF(Select_priv='Y', 'SELECT, ', ''),
    IF(Insert_priv='Y', 'INSERT, ', ''),
    IF(Update_priv='Y', 'UPDATE, ', ''),
    IF(Delete_priv='Y', 'DELETE, ', ''),
    IF(Create_priv='Y', 'CREATE, ', ''),
    IF(Drop_priv='Y', 'DROP, ', ''),
    IF(Grant_priv='Y', 'GRANT, ', ''),
    IF(Super_priv='Y', 'SUPER, ', ''),
    IF(Process_priv='Y', 'PROCESS, ', '')
  ) AS privileges
FROM mysql.user
WHERE User = 'username' AND Host = 'hostname';

-- Privilèges par base
SELECT 
  'DATABASE' AS privilege_level,
  Db AS database_name,
  User,
  Host,
  CONCAT(
    IF(Select_priv='Y', 'SELECT, ', ''),
    IF(Insert_priv='Y', 'INSERT, ', ''),
    IF(Update_priv='Y', 'UPDATE, ', ''),
    IF(Delete_priv='Y', 'DELETE, ', '')
  ) AS privileges
FROM mysql.db
WHERE User = 'username' AND Host = 'hostname';
```

---

### 4.4 - Utilisateurs Connectés Actuellement
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Utilisateurs actuellement connectés
-- ============================================================

SELECT 
  USER AS username,
  COUNT(*) AS connection_count,
  GROUP_CONCAT(DISTINCT HOST) AS connecting_from,
  GROUP_CONCAT(DISTINCT DB) AS databases_used,
  MAX(TIME) AS longest_query_sec
FROM information_schema.PROCESSLIST
GROUP BY USER
ORDER BY connection_count DESC;
```

---

### 4.5 - Audit Privilèges Dangereux
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Audit utilisateurs avec privilèges sensibles
-- ============================================================
-- Description :
-- Identifie utilisateurs avec privilèges élevés
--
-- Privilèges critiques surveillés :
-- - SUPER : Contrôle total serveur
-- - GRANT : Peut donner privilèges
-- - FILE : Accès fichiers système
-- - RELOAD : Peut flush/reload
-- - SHUTDOWN : Peut arrêter serveur
-- ============================================================

SELECT 
  User AS username,
  Host AS host,
  CONCAT(
    IF(Super_priv='Y', '🔴 SUPER ', ''),
    IF(Grant_priv='Y', '🔴 GRANT ', ''),
    IF(File_priv='Y', '🟠 FILE ', ''),
    IF(Reload_priv='Y', '🟠 RELOAD ', ''),
    IF(Shutdown_priv='Y', '🔴 SHUTDOWN ', ''),
    IF(Process_priv='Y', '🟡 PROCESS ', ''),
    IF(Create_user_priv='Y', '🟠 CREATE USER ', '')
  ) AS dangerous_privileges,
  CASE 
    WHEN Super_priv='Y' OR Grant_priv='Y' OR Shutdown_priv='Y' THEN 'CRITIQUE'
    WHEN File_priv='Y' OR Reload_priv='Y' OR Create_user_priv='Y' THEN 'ÉLEVÉ'
    WHEN Process_priv='Y' THEN 'MODÉRÉ'
    ELSE 'NORMAL'
  END AS risk_level
FROM mysql.user
WHERE Super_priv='Y' 
   OR Grant_priv='Y'
   OR File_priv='Y'
   OR Reload_priv='Y'
   OR Shutdown_priv='Y'
   OR Process_priv='Y'
   OR Create_user_priv='Y'
ORDER BY 
  FIELD(risk_level, 'CRITIQUE', 'ÉLEVÉ', 'MODÉRÉ', 'NORMAL'),
  User;
```

---

## ⚙️ CONFIGURATION ET VARIABLES

### 5.1 - Variables InnoDB Critiques
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Configuration InnoDB essentielle
-- ============================================================

SELECT 
  VARIABLE_NAME AS variable,
  VARIABLE_VALUE AS value,
  CASE VARIABLE_NAME
    WHEN 'innodb_buffer_pool_size' THEN 
      CONCAT(ROUND(VARIABLE_VALUE/1024/1024/1024, 2), ' GB')
    WHEN 'innodb_log_file_size' THEN
      CONCAT(ROUND(VARIABLE_VALUE/1024/1024, 2), ' MB')
    ELSE VARIABLE_VALUE
  END AS formatted_value
FROM information_schema.GLOBAL_VARIABLES
WHERE VARIABLE_NAME IN (
  'innodb_buffer_pool_size',
  'innodb_buffer_pool_instances',
  'innodb_log_file_size',
  'innodb_log_buffer_size',
  'innodb_flush_log_at_trx_commit',
  'innodb_flush_method',
  'innodb_io_capacity',
  'innodb_io_capacity_max',
  'innodb_read_io_threads',
  'innodb_write_io_threads'
)
ORDER BY VARIABLE_NAME;
```

---

### 5.2 - Comparer Configuration Actuelle vs Recommandée
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Audit configuration vs best practices
-- ============================================================

SELECT 
  v.VARIABLE_NAME AS variable,
  v.VARIABLE_VALUE AS current_value,
  CASE v.VARIABLE_NAME
    -- Buffer Pool : 70-80% RAM sur serveur dédié
    WHEN 'innodb_buffer_pool_size' THEN 
      'Recommandé: 70-80% de la RAM totale'
    
    -- Instances : 1 par GB jusqu'à 64
    WHEN 'innodb_buffer_pool_instances' THEN
      CONCAT('Recommandé: ', 
        LEAST(64, FLOOR(v.VARIABLE_VALUE/1024/1024/1024)), 
        ' (1 par GB)')
    
    -- Flush method : O_DIRECT sur Linux
    WHEN 'innodb_flush_method' THEN
      'Recommandé: O_DIRECT (Linux)'
    
    -- I/O capacity : Adapté au type disque
    WHEN 'innodb_io_capacity' THEN
      'HDD: 200, SSD SATA: 2000-5000, NVMe: 10000+'
    
    -- Flush commit : 1 pour ACID strict
    WHEN 'innodb_flush_log_at_trx_commit' THEN
      '1 = ACID strict, 2 = plus rapide mais risque'
    
    ELSE 'Vérifier documentation'
  END AS recommendation,
  CASE 
    WHEN v.VARIABLE_NAME = 'innodb_flush_method' 
         AND v.VARIABLE_VALUE != 'O_DIRECT' THEN '⚠️'
    WHEN v.VARIABLE_NAME = 'innodb_buffer_pool_size'
         AND CAST(v.VARIABLE_VALUE AS UNSIGNED) < 1073741824 THEN '⚠️'  -- < 1GB
    ELSE '✓'
  END AS status
FROM information_schema.GLOBAL_VARIABLES v
WHERE v.VARIABLE_NAME IN (
  'innodb_buffer_pool_size',
  'innodb_buffer_pool_instances',
  'innodb_flush_method',
  'innodb_io_capacity',
  'innodb_flush_log_at_trx_commit'
);
```

---

### 5.3 - Variables Modifiées vs Défaut
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Identifier variables non-défaut
-- ============================================================
-- Description :
-- Compare valeurs actuelles avec valeurs par défaut
--
-- Note :
-- Requiert MariaDB 10.2.2+ pour DEFAULT_VALUE
-- ============================================================

SELECT 
  VARIABLE_NAME AS variable,
  VARIABLE_VALUE AS current_value,
  DEFAULT_VALUE AS default_value,
  CASE 
    WHEN VARIABLE_VALUE = DEFAULT_VALUE THEN '='
    ELSE '≠'
  END AS status
FROM information_schema.SYSTEM_VARIABLES
WHERE VARIABLE_SCOPE IN ('GLOBAL', 'SESSION')
  AND VARIABLE_VALUE != DEFAULT_VALUE
ORDER BY VARIABLE_NAME;
```

---

### 5.4 - Uptime et Statistiques Serveur
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Statistiques serveur essentielles
-- ============================================================

SELECT 
  'Uptime' AS metric,
  CONCAT(
    FLOOR(VARIABLE_VALUE / 86400), ' jours, ',
    FLOOR((VARIABLE_VALUE % 86400) / 3600), ' heures, ',
    FLOOR((VARIABLE_VALUE % 3600) / 60), ' minutes'
  ) AS value
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Uptime'

UNION ALL

SELECT 
  'QPS (Queries/sec)',
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime'), 2
  )

UNION ALL

SELECT 
  'Connexions totales',
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Connections'

UNION ALL

SELECT 
  'Connexions actives',
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Threads_connected'

UNION ALL

SELECT 
  'Max connexions utilisées',
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Max_used_connections';
```

---

## ✅ Checklist Utilisation

### Diagnostic Quotidien
```sql
-- 1. Vérifier processus actifs
-- 2. Identifier requêtes longues (> 60s)
-- 3. Vérifier locks/blocages
-- 4. Contrôler connexions par utilisateur
-- 5. Surveiller tailles tables principales
```

### Troubleshooting
```sql
-- 1. PROCESSLIST pour identifier problème
-- 2. INNODB_TRX pour transactions actives
-- 3. INNODB_LOCK_WAITS pour blocages
-- 4. Tuer processus problématique si nécessaire
-- 5. Analyser slow query log
```

### Audit Sécurité
```sql
-- 1. Utilisateurs sans mot de passe
-- 2. Privilèges dangereux (SUPER, GRANT, FILE)
-- 3. Connexions depuis hosts inattendus
-- 4. Utilisateurs inactifs à supprimer
```

---

## 🔗 Ressources

### Liens Vers Autres Sections
- **[C.2 Requêtes de Monitoring →](./02-requetes-monitoring.md)** : Buffer pool, slow queries, réplication
- **[C.3 Requêtes d'Analyse →](./03-requetes-analyse.md)** : Index usage, fragmentation, performance

### Documentation
- [INFORMATION_SCHEMA Tables](https://mariadb.com/kb/en/information-schema-tables/)
- [SHOW Statements](https://mariadb.com/kb/en/show/)
- [InnoDB System Tables](https://mariadb.com/kb/en/innodb-system-tables/)

---

**MariaDB** : 11.8 LTS

⏭️ [Requêtes de monitoring (buffer pool, slow queries, replication)](/annexes/requetes-sql-reference/02-requetes-monitoring.md)

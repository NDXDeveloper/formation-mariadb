🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.10 Transaction Replay et Connection Migration (MariaDB 11.8)

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Sections 14.1-14.9, compréhension transactions ACID, réplication

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le fonctionnement interne de Transaction Replay et Connection Migration
- **Configurer** ces fonctionnalités pour production
- **Identifier** les cas d'usage optimaux et les limitations
- **Optimiser** les paramètres selon votre workload
- **Monitorer** l'efficacité de ces mécanismes
- **Débugger** les problèmes de replay ou migration
- **Intégrer** avec votre infrastructure HA existante
- **Mesurer** l'impact sur la disponibilité et l'expérience utilisateur

---

## Introduction

MariaDB 11.8 LTS (sortie janvier 2025) introduit deux fonctionnalités révolutionnaires qui transforment radicalement l'expérience du failover :

**🆕 Transaction Replay** : Rejouabilité automatique des transactions après une perte de connexion
**🆕 Connection Migration** : Préservation transparente des sessions et de leur contexte

**Problème historique résolu** :
```
AVANT MariaDB 11.8 :
┌──────────────────────────────────────────────────┐
│  Application → BEGIN TRANSACTION                 │
│             → UPDATE accounts SET balance...     │
│             → INSERT INTO audit_log...           │
│             → COMMIT ← ✖ Master crash           │
│                                                  │
│  Result :                                        │
│  ❌ Error: Lost connection to MySQL server       │
│  ❌ Transaction état inconnu (commitée ou non?)  │
│  ❌ Application doit gérer retry manuellement    │
│  ❌ Risque de duplicate si retry mal codé        │
│  ❌ Utilisateur voit erreur                      │
└──────────────────────────────────────────────────┘

AVEC MariaDB 11.8 :
┌──────────────────────────────────────────────────┐
│  Application → BEGIN TRANSACTION                 │
│             → UPDATE accounts SET balance...     │
│             → INSERT INTO audit_log...           │
│             → COMMIT ← Master crash              │
│                                                  │
│  MariaDB (automatiquement) :                     │
│  1. Détecte perte connexion                      │
│  2. Vérifie si transaction commitée              │
│  3. Si non, rejoue sur nouveau master            │
│  4. Retourne succès à l'application              │
│                                                  │
│  Result :                                        │
│  ✅ Application reçoit success (transparent!)    │
│  ✅ Aucune erreur visible utilisateur            │
│  ✅ Aucun code retry nécessaire                  │
└──────────────────────────────────────────────────┘
```

> 💡 **Game Changer** : Ces fonctionnalités réduisent le RTO de plusieurs minutes à quelques secondes, et rendent le failover complètement transparent pour 95%+ des cas d'usage.

---

## 1. Transaction Replay : Rejouabilité Automatique

### 1.1 Fonctionnement Interne

#### **Architecture du Mécanisme**

```
Phase 1 : Exécution Transaction (avant crash)
┌────────────────────────────────────────────────┐
│  Application                                   │
│      │                                         │
│      │ BEGIN                                   │
│      ├─────────────────────────────►           │
│      │                             MariaDB     │
│      │ UPDATE accounts...                      │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │                             │ Transaction 
│      │                             │ Log Buffer  
│      │                             │ (in-memory) 
│      │ INSERT audit_log...                     │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │ COMMIT                      │           │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │                      ┌──────▼──────┐    │
│      │                      │ Replay Log  │    │
│      │                      │ (persistent)│    │
│      │                      │             │    │
│      │                      │ - BEGIN     │    │
│      │                      │ - UPDATE... │    │
│      │                      │ - INSERT... │    │
│      │                      │ - COMMIT    │    │
│      │                      └─────────────┘    │
│      │                             │           │
│      │           ✖ CRASH MASTER    │          │
└───────────────────────────────────────────────┘

Phase 2 : Détection et Validation
┌────────────────────────────────────────────────┐
│  Application (perd connexion)                  │
│      │                                         │
│      │ Connection Lost Error                   │
│      ◄─────────────────────────────            │
│                                                │
│  MariaDB Client Library (11.8+) :              │
│  1. Détecte perte connexion                    │
│  2. Vérifie si replay enabled                  │
│  3. Check si transaction en cours              │
│                                                │
│  Si oui → Tenter Replay                        │
└────────────────────────────────────────────────┘

Phase 3 : Replay sur Nouveau Master
┌────────────────────────────────────────────────┐
│  Client Library                                │
│      │                                         │
│      │ Connexion vers nouveau master           │
│      ├─────────────────────────────►           │
│      │                         Nouveau Master  │
│      │                                         │
│      │ Envoie Replay Log                       │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │                     ┌───────▼──────┐    │
│      │                     │ Vérifie GTID │    │
│      │                     │ - Déjà exec? │    │
│      │                     │ - Si oui: OK │    │
│      │                     │ - Si non:    │    │
│      │                     │   Rejoue!    │    │
│      │                     └───────┬──────┘    │
│      │                             │           │
│      │                     BEGIN               │
│      │                     UPDATE accounts...  │
│      │                     INSERT audit_log... │
│      │                     COMMIT              │
│      │                             │           │
│      │ SUCCESS                     │           │
│      ◄─────────────────────────────┘           │
│      │                                         │
│      │ Retourne SUCCESS à app                  │
│      ▼                                         │
│  Application (ne voit aucune erreur !)         │
└────────────────────────────────────────────────┘
```

#### **Variables de Configuration**

```sql
-- ============================================
-- TRANSACTION REPLAY CONFIGURATION
-- ============================================

-- Activer Transaction Replay (global)
SET GLOBAL transaction_replay = ON;

-- Nombre de tentatives de replay
SET GLOBAL transaction_replay_attempts = 3;
-- Par défaut : 3
-- Range : 1-100
-- Recommandation : 3-5 pour production

-- Timeout pour chaque tentative de replay (secondes)
SET GLOBAL transaction_replay_timeout = 30;
-- Par défaut : 30 secondes
-- Range : 1-3600
-- Recommandation : 30-60 pour production

-- Taille max du replay log (MB)
SET GLOBAL transaction_replay_max_size = 10;
-- Par défaut : 10 MB
-- Range : 1-100
-- Transactions > cette taille ne seront PAS rejouées

-- Niveau de verbosité logging
SET GLOBAL transaction_replay_log_level = 2;
-- 0 = Aucun log
-- 1 = Erreurs seulement
-- 2 = Succès et erreurs (recommandé)
-- 3 = Debug complet

-- Configuration par session (override global)
SET SESSION transaction_replay = ON;
SET SESSION transaction_replay_timeout = 60;
```

### 1.2 Configuration Production

#### **Configuration my.cnf**

```ini
# /etc/mysql/conf.d/transaction-replay.cnf

[mysqld]
# ============================================
# TRANSACTION REPLAY (MariaDB 11.8+)
# ============================================

# Activer replay globalement
transaction_replay = ON

# Tentatives de replay
transaction_replay_attempts = 3

# Timeout par tentative (secondes)
transaction_replay_timeout = 30

# Taille max transaction rejouable (MB)
transaction_replay_max_size = 10

# Logging
transaction_replay_log_level = 2
log_error_verbosity = 3

# ============================================
# OPTIMISATIONS LIÉES
# ============================================

# Semi-sync replication (recommandé avec replay)
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_slave_enabled = 1
rpl_semi_sync_master_timeout = 10000  # 10 secondes

# GTID (requis pour replay)
gtid_strict_mode = 1
log_bin = /var/log/mysql/mariadb-bin
log_bin_index = /var/log/mysql/mariadb-bin.index
binlog_format = ROW

# InnoDB (cohérence)
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

#### **Configuration Applicative (Drivers)**

```python
# Python (mariadb-connector 1.2.0+)
import mariadb

config = {
    'host': 'db.example.com',
    'port': 3306,
    'user': 'app_user',
    'password': 'AppPassword',
    'database': 'production_db',
    
    # Transaction Replay (côté client)
    'transaction_replay': True,
    'transaction_replay_attempts': 3,
    'transaction_replay_timeout': 30,
    
    # Auto-reconnect
    'auto_reconnect': True,
    'reconnect_max_attempts': 5
}

conn = mariadb.connect(**config)
cursor = conn.cursor()

# Transaction avec replay automatique
try:
    cursor.execute("BEGIN")
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = ?", (123,))
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = ?", (456,))
    cursor.execute("INSERT INTO audit_log (action, timestamp) VALUES (?, NOW())", ('transfer',))
    cursor.execute("COMMIT")
    
    # Si master crash durant COMMIT :
    # → Client library tente replay automatique
    # → Retourne success si replay réussi
    # → Lance exception seulement si replay échoue après N tentatives
    
except mariadb.Error as e:
    # Erreur uniquement si replay échoué après 3 tentatives
    print(f"Transaction failed after replay attempts: {e}")
    cursor.execute("ROLLBACK")
```

```java
// Java (MariaDB Connector/J 3.3.0+)
import org.mariadb.jdbc.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

String url = "jdbc:mariadb://db.example.com:3306/production_db" +
    "?transactionReplay=true" +
    "&transactionReplayAttempts=3" +
    "&transactionReplayTimeout=30" +
    "&autoReconnect=true";

Connection conn = DriverManager.getConnection(url, "app_user", "AppPassword");

try {
    conn.setAutoCommit(false);
    
    PreparedStatement pstmt1 = conn.prepareStatement(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?"
    );
    pstmt1.setDouble(1, 100.0);
    pstmt1.setInt(2, 123);
    pstmt1.executeUpdate();
    
    PreparedStatement pstmt2 = conn.prepareStatement(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?"
    );
    pstmt2.setDouble(1, 100.0);
    pstmt2.setInt(2, 456);
    pstmt2.executeUpdate();
    
    conn.commit();  // Replay automatique si crash ici
    
} catch (SQLException e) {
    // Exception levée seulement si replay échoué
    conn.rollback();
    throw e;
}
```

### 1.3 Cas d'Usage et Limitations

#### **Scénarios Supportés** ✅

```sql
-- 1. Transactions standards (READ COMMITTED)
BEGIN;
UPDATE orders SET status = 'shipped' WHERE id = 123;
INSERT INTO shipments (order_id, tracking) VALUES (123, 'TRACK001');
COMMIT;  -- ✅ Rejouable

-- 2. Transactions complexes multi-tables
BEGIN;
UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 456;
INSERT INTO sales (product_id, quantity, price) VALUES (456, 10, 99.99);
UPDATE products SET last_sold = NOW() WHERE id = 456;
COMMIT;  -- ✅ Rejouable

-- 3. Stored Procedures déterministes
CALL transfer_funds(123, 456, 100.00);  -- ✅ Rejouable si déterministe

-- 4. Prepared Statements
PREPARE stmt FROM 'UPDATE accounts SET balance = balance + ? WHERE id = ?';
EXECUTE stmt USING @amount, @account_id;  -- ✅ Rejouable

-- 5. Autocommit statements
UPDATE users SET last_login = NOW() WHERE id = 789;  -- ✅ Rejouable
```

#### **Scénarios NON Supportés** ❌

```sql
-- 1. LOCK TABLES explicites
LOCK TABLES accounts WRITE;
UPDATE accounts SET balance = 1000;
UNLOCK TABLES;  -- ❌ NON rejouable

-- 2. GET_LOCK() / RELEASE_LOCK()
SELECT GET_LOCK('my_lock', 10);
UPDATE shared_resource SET value = 123;
SELECT RELEASE_LOCK('my_lock');  -- ❌ NON rejouable

-- 3. Transactions avec side effects externes
BEGIN;
UPDATE queue SET processed = 1 WHERE id = 1;
-- Application envoie email (side effect externe)
COMMIT;  -- ❌ Risque de duplicate email si replay

-- 4. SERIALIZABLE isolation level
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT * FROM accounts WHERE balance > 1000;
UPDATE accounts SET balance = balance * 1.1;
COMMIT;  -- ❌ NON rejouable (SERIALIZABLE non supporté)

-- 5. Transactions > transaction_replay_max_size
BEGIN;
-- Insert 50 MB de données
INSERT INTO large_table SELECT ... (50 MB);
COMMIT;  -- ❌ NON rejouable (> 10 MB par défaut)

-- 6. XA Transactions (distributed transactions)
XA START 'xid1';
UPDATE accounts SET balance = balance - 100;
XA END 'xid1';
XA PREPARE 'xid1';
XA COMMIT 'xid1';  -- ❌ NON rejouable

-- 7. Fonctions non-déterministes dans transaction
BEGIN;
INSERT INTO logs (timestamp, random_id) 
VALUES (NOW(), RAND());  -- ❌ RAND() non-déterministe
COMMIT;  -- Résultat différent si rejouée
```

#### **Best Practices**

```sql
-- ✅ FAIRE : Transactions idempotentes
BEGIN;
INSERT INTO users (id, email, name) 
VALUES (123, 'user@example.com', 'John Doe')
ON DUPLICATE KEY UPDATE name = VALUES(name);
COMMIT;

-- ✅ FAIRE : Vérifications avant side effects
BEGIN;
UPDATE orders SET status = 'processed' WHERE id = 123;
SELECT status FROM orders WHERE id = 123;  -- Vérifier avant email
COMMIT;
-- Application envoie email SEULEMENT si status = 'processed'

-- ❌ ÉVITER : Side effects dans transactions
BEGIN;
UPDATE queue SET sent = 1 WHERE id = 1;
-- DO sys_exec('send_email.sh')  -- ❌ Side effect
COMMIT;

-- ✅ FAIRE : Side effects après COMMIT confirmé
BEGIN;
UPDATE queue SET sent = 1 WHERE id = 1;
COMMIT;
-- Maintenant safe d'envoyer email

-- ✅ FAIRE : Utiliser GTID pour détection duplicates
SELECT @@gtid_current_pos;  -- Noter GTID
BEGIN;
-- Transaction
COMMIT;
SELECT @@gtid_current_pos;  -- Vérifier GTID incrémenté
```

### 1.4 Monitoring et Métriques

```sql
-- ============================================
-- MONITORING TRANSACTION REPLAY
-- ============================================

-- Vue globale des statistiques
SHOW GLOBAL STATUS LIKE 'Transaction_replay%';

/*
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| Transaction_replay_attempted     | 1234  |
| Transaction_replay_succeeded     | 1198  |
| Transaction_replay_failed        | 36    |
| Transaction_replay_skipped       | 12    |
| Transaction_replay_avg_time_ms   | 45    |
| Transaction_replay_max_time_ms   | 230   |
| Transaction_replay_total_time_ms | 53910 |
+----------------------------------+-------+
*/

-- Taux de succès
SELECT 
    CAST(@@global.Transaction_replay_succeeded AS UNSIGNED) AS succeeded,
    CAST(@@global.Transaction_replay_attempted AS UNSIGNED) AS attempted,
    ROUND(
        100.0 * CAST(@@global.Transaction_replay_succeeded AS UNSIGNED) / 
        NULLIF(CAST(@@global.Transaction_replay_attempted AS UNSIGNED), 0),
        2
    ) AS success_rate_percent
\G

-- Temps moyen de replay
SELECT 
    CONCAT(
        CAST(@@global.Transaction_replay_avg_time_ms AS UNSIGNED),
        ' ms'
    ) AS avg_replay_time;

-- Détails des échecs (via error log)
/*!
Rechercher dans /var/log/mysql/error.log :

2025-12-15 10:32:15 [Warning] Transaction replay failed for transaction starting at GTID 0-1-12345
  Reason: Timeout after 30 seconds
  Attempts: 3/3
  Transaction size: 2.3 MB
  
2025-12-15 10:35:22 [Info] Transaction replay succeeded for transaction starting at GTID 0-1-12346
  Attempt: 2/3
  Replay time: 1.2 seconds
  Transaction size: 512 KB
*/
```

**Dashboard Grafana** :
```yaml
# prometheus_queries.yml
- name: transaction_replay_success_rate
  query: |
    rate(mysql_transaction_replay_succeeded[5m]) / 
    rate(mysql_transaction_replay_attempted[5m]) * 100

- name: transaction_replay_latency_p99
  query: |
    histogram_quantile(0.99, 
      rate(mysql_transaction_replay_time_bucket[5m]))

- name: transaction_replay_failures
  query: |
    increase(mysql_transaction_replay_failed[1h])
```

---

## 2. Connection Migration : Préservation de Session

### 2.1 Fonctionnement Interne

#### **Architecture du Mécanisme**

```
Phase 1 : Établissement Session Initial
┌────────────────────────────────────────────────┐
│  Application                                   │
│      │                                         │
│      │ CONNECT                                 │
│      ├─────────────────────────────►           │
│      │                         Master (10.0.1.10) 
│      │                                         │
│      │ SET time_zone = 'America/New_York'      │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │ SET sql_mode = 'STRICT_...' │           │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │ PREPARE stmt FROM '...'     │           │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │                     ┌───────▼──────┐    │
│      │                     │ Session State│    │
│      │                     │ Snapshot     │    │
│      │                     │              │    │
│      │                     │ - time_zone  │    │
│      │                     │ - sql_mode   │    │
│      │                     │ - charset    │    │
│      │                     │ - collation  │    │
│      │                     │ - autocommit │    │
│      │                     │ - isolation  │    │
│      │                     │ - prepared:  │    │
│      │                     │   stmt → SQL │    │
│      │                     └──────────────┘    │
└────────────────────────────────────────────────┘

Phase 2 : Failover et Migration
┌────────────────────────────────────────────────┐
│  Application                                   │
│      │                                         │
│      │ EXECUTE stmt                            │
│      ├─────────────────────────────►           │
│      │                    ✖ CRASH Master      │
│      │                                         │
│      │ Connection Lost                         │
│      ◄─────────────────────────────            │
│                                                │
│  Client Library (11.8+) :                      │
│  1. Détecte perte connexion                    │
│  2. Connection migration enabled ?             │
│  3. Récupère Session State Snapshot            │
│  4. Connexion vers nouveau master              │
│      │                                         │
│      │ CONNECT                                 │
│      ├─────────────────────────────►           │
│      │                    Nouveau Master       │
│      │                    (10.0.1.11)          │
│      │                                         │
│      │ Réapplique Session State :              │
│      │ SET time_zone = 'America/New_York'      │
│      ├─────────────────────────────►           │
│      │ SET sql_mode = 'STRICT_...'             │
│      ├─────────────────────────────►           │
│      │ PREPARE stmt FROM '...'                 │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │ Session migrée ✅           │          │
│      │                             │           │
│      │ Rejoue dernière commande    │           │
│      │ EXECUTE stmt                │           │
│      ├─────────────────────────────►           │
│      │                             │           │
│      │ SUCCESS                     │           │
│      ◄─────────────────────────────┘           │
│      │                                         │
│      ▼                                         │
│  Application (session préservée !)             │
└────────────────────────────────────────────────┘
```

#### **Variables de Configuration**

```sql
-- ============================================
-- CONNECTION MIGRATION CONFIGURATION
-- ============================================

-- Activer Connection Migration (global)
SET GLOBAL connection_migration = ON;

-- Préserver variables de session
SET GLOBAL connection_migration_preserve_session = ON;
-- Migre : time_zone, sql_mode, autocommit, isolation, charset, etc.

-- Préserver prepared statements
SET GLOBAL connection_migration_preserve_prepared = ON;
-- Migre : tous les PREPARE stmt actifs

-- Préserver user variables (@var)
SET GLOBAL connection_migration_preserve_user_vars = OFF;
-- Par défaut : OFF (performance)
-- Activer si application dépend de @variables

-- Timeout migration (secondes)
SET GLOBAL connection_migration_timeout = 10;
-- Temps max pour migration complète

-- Max tentatives de migration
SET GLOBAL connection_migration_max_retries = 3;

-- Logging
SET GLOBAL connection_migration_log_level = 2;
-- 0 = Aucun, 1 = Erreurs, 2 = Info, 3 = Debug
```

### 2.2 Configuration Production

```ini
# /etc/mysql/conf.d/connection-migration.cnf

[mysqld]
# ============================================
# CONNECTION MIGRATION (MariaDB 11.8+)
# ============================================

# Activer migration
connection_migration = ON

# Préservation état session
connection_migration_preserve_session = ON
connection_migration_preserve_prepared = ON
connection_migration_preserve_user_vars = OFF

# Timeouts et retries
connection_migration_timeout = 10
connection_migration_max_retries = 3

# Logging
connection_migration_log_level = 2

# ============================================
# COMPATIBILITÉ AVEC TRANSACTION REPLAY
# ============================================

# Les deux features se complètent :
# - Connection Migration : préserve contexte session
# - Transaction Replay : rejoue transaction interrompue

transaction_replay = ON
transaction_replay_attempts = 3
```

### 2.3 Variables de Session Migrées

```sql
-- ============================================
-- VARIABLES AUTOMATIQUEMENT MIGRÉES
-- ============================================

-- Variables de base
SELECT @@session.time_zone;              -- ✅ Migré
SELECT @@session.sql_mode;               -- ✅ Migré
SELECT @@session.autocommit;             -- ✅ Migré
SELECT @@session.character_set_client;   -- ✅ Migré
SELECT @@session.character_set_connection; -- ✅ Migré
SELECT @@session.character_set_results;  -- ✅ Migré
SELECT @@session.collation_connection;   -- ✅ Migré

-- Isolation et transaction
SELECT @@session.transaction_isolation;  -- ✅ Migré
SELECT @@session.transaction_read_only;  -- ✅ Migré

-- Autres variables importantes
SELECT @@session.foreign_key_checks;     -- ✅ Migré
SELECT @@session.unique_checks;          -- ✅ Migré
SELECT @@session.sql_notes;              -- ✅ Migré
SELECT @@session.sql_warnings;           -- ✅ Migré

-- ============================================
-- NON MIGRÉS
-- ============================================

-- Temporary tables
CREATE TEMPORARY TABLE temp_data (id INT);  -- ❌ Perdu

-- User-defined variables (si preserve_user_vars = OFF)
SET @my_var = 123;                       -- ❌ Perdu (par défaut)

-- LOCK TABLES
LOCK TABLES accounts WRITE;              -- ❌ Perdu (normal)

-- GET_LOCK
SELECT GET_LOCK('my_lock', 10);          -- ❌ Perdu (normal)

-- Transactions en cours (sauf si Transaction Replay actif)
BEGIN;
UPDATE accounts SET balance = 100;
-- Failover ici → Transaction perdue si pas Transaction Replay
```

### 2.4 Configuration Applicative

```python
# Python - Connection Migration
import mariadb

config = {
    'host': 'db.example.com',
    'port': 3306,
    'user': 'app_user',
    'password': 'AppPassword',
    'database': 'production_db',
    
    # Connection Migration
    'connection_migration': True,
    'connection_migration_timeout': 10,
    
    # Transaction Replay (complémentaire)
    'transaction_replay': True,
    
    # Auto-reconnect
    'auto_reconnect': True
}

conn = mariadb.connect(**config)

# Configuration session
conn.execute("SET time_zone = 'UTC'")
conn.execute("SET sql_mode = 'STRICT_TRANS_TABLES'")

# Prepared statement
stmt = conn.prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?")

# Utilisation normale
stmt.execute([100, 123])

# Si failover durant execute :
# 1. Connection Migration : Session migrée (time_zone, sql_mode, prepared stmt)
# 2. Transaction Replay : Transaction rejouée si nécessaire
# → Transparent pour application
```

```java
// Java - Connection Migration
import org.mariadb.jdbc.Connection;
import java.sql.PreparedStatement;

String url = "jdbc:mariadb://db.example.com:3306/production_db" +
    "?connectionMigration=true" +
    "&connectionMigrationTimeout=10" +
    "&transactionReplay=true" +
    "&autoReconnect=true";

Connection conn = DriverManager.getConnection(url, "app_user", "AppPassword");

// Configuration session
conn.createStatement().execute("SET time_zone = 'UTC'");
conn.createStatement().execute("SET sql_mode = 'STRICT_TRANS_TABLES'");

// Prepared statement
PreparedStatement pstmt = conn.prepareStatement(
    "UPDATE accounts SET balance = balance + ? WHERE id = ?"
);

// Failover durant execute → Migration automatique
pstmt.setDouble(1, 100.0);
pstmt.setInt(2, 123);
pstmt.executeUpdate();  // Transparent même si failover
```

---

## 3. Intégration avec Infrastructure HA

### 3.1 Transaction Replay + Connection Migration + MaxScale

```
Architecture Complète :
┌─────────────────────────────────────────────────┐
│                Applications                     │
│   (MariaDB Connector 11.8+)                     │
│   - Transaction Replay : ON                     │
│   - Connection Migration : ON                   │
└────────────────┬────────────────────────────────┘
                 │
                 │ Port 3306
                 │
        ┌────────▼────────┐
        │    MaxScale     │
        │   (25.01+)      │
        │                 │
        │  mariadbmon     │ ← Auto-failover
        │  - Détecte down │
        │  - Promote new  │
        │  - Reroute      │
        └────────┬────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼─────┐ ┌────▼────┐ ┌─────▼───┐
│ Node1   │ │ Node2   │ │ Node3   │
│ Master  │ │ Replica │ │ Replica │
│         │ │         │ │         │
│ 11.8    │ │ 11.8    │ │ 11.8    │
└─────────┘ └─────────┘ └─────────┘

Timeline Failover :
T+0s    : Master (Node1) crash
T+5s    : MaxScale mariadbmon détecte
T+10s   : MaxScale promote Node2
T+12s   : Applications perdent connexion Node1
T+13s   : Connection Migration → reconnect Node2
T+14s   : Transaction Replay → rejoue transaction
T+15s   : Applications reprennent (transparent!)

RTO Total : 15 secondes
Impact utilisateur : AUCUN (si Transaction Replay réussit)
```

### 3.2 Configuration Optimale Multi-Layer

```sql
-- ============================================
-- LAYER 1 : MariaDB 11.8 (Backend)
-- ============================================

-- my.cnf
[mysqld]
# Transaction Replay
transaction_replay = ON
transaction_replay_attempts = 3
transaction_replay_timeout = 30
transaction_replay_max_size = 10

# Connection Migration
connection_migration = ON
connection_migration_preserve_session = ON
connection_migration_preserve_prepared = ON
connection_migration_timeout = 10

# Semi-sync (RPO = 0)
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_slave_enabled = 1
rpl_semi_sync_master_timeout = 10000

# GTID (requis)
gtid_strict_mode = 1
log_bin = ON
binlog_format = ROW
```

```ini
# ============================================
# LAYER 2 : MaxScale 25.01 (Proxy)
# ============================================

# /etc/maxscale.cnf
[MariaDB-Monitor]
type = monitor
module = mariadbmon
servers = server1, server2, server3

# Failover automatique
auto_failover = true
auto_rejoin = true

# Détection rapide
failcount = 3
monitor_interval = 1000ms

# Post-failover hooks
script = /usr/local/bin/failover-notify.sh

[Read-Write-Service]
type = service
router = readwritesplit
servers = server1, server2, server3

# Optimisations pour Transaction Replay
# → Pas de retry côté MaxScale (laissé au client)
max_slave_connections = 100%
use_sql_variables_in = all
```

```python
# ============================================
# LAYER 3 : Application (Client)
# ============================================

# Python app config
db_config = {
    'host': 'maxscale.example.com',  # MaxScale VIP
    'port': 3306,
    
    # Transaction Replay (MariaDB 11.8 feature)
    'transaction_replay': True,
    'transaction_replay_attempts': 3,
    
    # Connection Migration (MariaDB 11.8 feature)
    'connection_migration': True,
    
    # Auto-reconnect (fallback)
    'auto_reconnect': True,
    'reconnect_max_attempts': 5,
    
    # Timeouts
    'connect_timeout': 5,
    'read_timeout': 30,
    'write_timeout': 30
}
```

### 3.3 Testing Complet

```bash
#!/bin/bash
# test_replay_migration.sh
# Test Transaction Replay + Connection Migration

set -e

echo "=== Test Transaction Replay + Connection Migration ==="

MASTER="db-master.example.com"
REPLICA="db-replica.example.com"
VIP="db-vip.example.com"

# 1. Créer table de test
mysql -h $VIP << 'EOF'
CREATE DATABASE IF NOT EXISTS test_replay;
USE test_replay;

CREATE TABLE IF NOT EXISTS test_transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(10,2),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    server VARCHAR(50)
);

TRUNCATE test_transactions;
EOF

# 2. Lancer workload en background
python3 << 'PYTHON' &
import mariadb
import time
import random

config = {
    'host': 'db-vip.example.com',
    'user': 'test_user',
    'password': 'TestPassword',
    'database': 'test_replay',
    'transaction_replay': True,
    'connection_migration': True,
    'auto_reconnect': True
}

conn = mariadb.connect(**config)
cursor = conn.cursor()

# Configuration session
cursor.execute("SET time_zone = 'UTC'")
cursor.execute("SET sql_mode = 'STRICT_TRANS_TABLES'")

# Prepared statement
cursor.execute("PREPARE insert_tx FROM 'INSERT INTO test_transactions (amount, server) VALUES (?, @@hostname)'")

success = 0
errors = 0

for i in range(1000):
    try:
        cursor.execute("BEGIN")
        cursor.execute(f"SET @amount = {random.uniform(1, 100):.2f}")
        cursor.execute("EXECUTE insert_tx USING @amount")
        cursor.execute("COMMIT")
        success += 1
        
        if i % 100 == 0:
            print(f"Progress: {i}/1000 - Success: {success}, Errors: {errors}")
        
        time.sleep(0.1)
        
    except Exception as e:
        errors += 1
        print(f"Error: {e}")

print(f"\nFinal: Success={success}, Errors={errors}")
PYTHON

WORKLOAD_PID=$!

# 3. Attendre démarrage workload
sleep 5

# 4. Crash master pendant workload
echo "Crashing master in 5 seconds..."
sleep 5

ssh $MASTER "systemctl stop mariadb"
echo "Master crashed at $(date)"

# 5. Observer failover
echo "Waiting for failover..."
sleep 10

# 6. Vérifier nouveau master
NEW_MASTER=$(maxctrl list servers --tsv | grep Master | awk '{print $1}')
echo "New master: $NEW_MASTER"

# 7. Attendre fin workload
wait $WORKLOAD_PID

# 8. Vérifier résultats
echo ""
echo "=== Results ==="
mysql -h $VIP -e "
USE test_replay;
SELECT 
    COUNT(*) AS total_transactions,
    COUNT(DISTINCT server) AS servers_used,
    SUM(amount) AS total_amount,
    MIN(timestamp) AS first_tx,
    MAX(timestamp) AS last_tx
FROM test_transactions;
"

echo ""
echo "=== Transactions per server ==="
mysql -h $VIP -e "
USE test_replay;
SELECT 
    server,
    COUNT(*) AS tx_count,
    SUM(amount) AS total_amount
FROM test_transactions
GROUP BY server;
"

# 9. Vérifier logs replay/migration
echo ""
echo "=== Transaction Replay Stats ==="
mysql -h $VIP -e "SHOW GLOBAL STATUS LIKE 'Transaction_replay%';"

echo ""
echo "=== Connection Migration Stats ==="
mysql -h $VIP -e "SHOW GLOBAL STATUS LIKE 'Connection_migration%';"

# 10. Cleanup
echo ""
echo "Restarting old master..."
ssh $MASTER "systemctl start mariadb"

echo "Test complete!"
```

---

## 4. Troubleshooting et Optimisation

### 4.1 Diagnostics Transaction Replay

```sql
-- Vérifier si replay actif
SHOW VARIABLES LIKE 'transaction_replay%';

-- Statistiques détaillées
SHOW GLOBAL STATUS LIKE 'Transaction_replay%';

-- Identifier transactions problématiques
-- (via error log)
/*!
Patterns à rechercher dans /var/log/mysql/error.log :

[Warning] Transaction replay failed: Transaction too large (15.2 MB > 10 MB limit)
→ Solution : Augmenter transaction_replay_max_size

[Warning] Transaction replay failed: Timeout after 30 seconds
→ Solution : Augmenter transaction_replay_timeout

[Warning] Transaction replay failed: Non-deterministic function used: RAND()
→ Solution : Éviter RAND(), UUID(), NOW() dans transactions critiques

[Error] Transaction replay failed: SERIALIZABLE isolation not supported
→ Solution : Utiliser READ COMMITTED ou REPEATABLE READ
*/

-- Requête problèmes fréquents
SELECT 
    'Too Large' AS issue_type,
    ROUND(@@global.transaction_replay_max_size, 2) AS current_limit_mb,
    'Increase transaction_replay_max_size' AS recommendation
UNION ALL
SELECT 
    'Timeout',
    @@global.transaction_replay_timeout,
    'Increase transaction_replay_timeout or optimize queries'
UNION ALL
SELECT 
    'Success Rate',
    ROUND(100.0 * @@global.Transaction_replay_succeeded / 
        NULLIF(@@global.Transaction_replay_attempted, 0), 2),
    CASE 
        WHEN @@global.Transaction_replay_succeeded / 
             NULLIF(@@global.Transaction_replay_attempted, 0) < 0.95 
        THEN 'Investigate failed replays'
        ELSE 'OK'
    END;
```

### 4.2 Optimisations Performance

```sql
-- ============================================
-- TUNING TRANSACTION REPLAY
-- ============================================

-- Augmenter limite taille si transactions légitimement grandes
SET GLOBAL transaction_replay_max_size = 50;  -- 50 MB

-- Augmenter timeout si queries lentes
SET GLOBAL transaction_replay_timeout = 60;   -- 60 secondes

-- Réduire tentatives si échec rapide préférable
SET GLOBAL transaction_replay_attempts = 2;   -- 2 au lieu de 3

-- ============================================
-- TUNING CONNECTION MIGRATION
-- ============================================

-- Désactiver user vars si non utilisées (perf)
SET GLOBAL connection_migration_preserve_user_vars = OFF;

-- Timeout court si failover rapide
SET GLOBAL connection_migration_timeout = 5;  -- 5 secondes

-- ============================================
-- MONITORING OVERHEAD
-- ============================================

-- Mesurer overhead replay
SELECT 
    @@global.Transaction_replay_total_time_ms / 
    NULLIF(@@global.Transaction_replay_attempted, 0) 
    AS avg_replay_overhead_ms;

-- Si overhead > 100ms en moyenne :
-- → Optimiser transactions (réduire taille)
-- → Augmenter resources (CPU, I/O)
```

### 4.3 Alertes Prometheus

```yaml
# prometheus_alerts.yml
groups:
  - name: mariadb_replay_migration
    rules:
      # Transaction Replay failure rate élevé
      - alert: TransactionReplayHighFailureRate
        expr: |
          (
            rate(mysql_transaction_replay_failed[5m]) /
            rate(mysql_transaction_replay_attempted[5m])
          ) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High transaction replay failure rate"
          description: "{{ $value | humanizePercentage }} of replays failing"

      # Connection Migration échecs
      - alert: ConnectionMigrationFailures
        expr: |
          increase(mysql_connection_migration_failed[15m]) > 10
        labels:
          severity: warning
        annotations:
          summary: "Connection migration failures detected"
          description: "{{ $value }} failures in last 15 minutes"

      # Replay latency élevée
      - alert: TransactionReplayHighLatency
        expr: |
          mysql_transaction_replay_avg_time_ms > 200
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "Transaction replay latency high"
          description: "Average replay time: {{ $value }}ms"
```

---

## ✅ Points Clés à Retenir

- **Transaction Replay** : Rejouabilité automatique des transactions après failover
- **Connection Migration** : Préservation session (variables, prepared statements)
- **RPO = 0, RTO < 30s** : Combinaison Transaction Replay + Connection Migration
- **Transparence applicative** : 95%+ des failovers invisibles pour utilisateurs
- **Limitations connues** : LOCK TABLES, SERIALIZABLE, fonctions non-déterministes
- **Configuration combinée** : Transaction Replay + Connection Migration + MaxScale = optimal
- **Monitoring essentiel** : Success rate, latency, failures à surveiller
- **Production-ready** : MariaDB 11.8 LTS + Drivers compatibles requis
- **Game changer** : Réduit drastiquement complexité gestion failover côté application
- **Best practice** : Transactions idempotentes, éviter side effects externes

---

## 🔗 Ressources et Références

### Documentation Officielle MariaDB 11.8
- [📖 Transaction Replay Documentation](https://mariadb.com/kb/en/transaction-replay/)
- [📖 Connection Migration Documentation](https://mariadb.com/kb/en/connection-migration/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-0-release-notes/)

### Drivers Compatibles
- [MariaDB Connector/C 3.4+](https://mariadb.com/kb/en/mariadb-connector-c/)
- [MariaDB Connector/J 3.3+](https://mariadb.com/kb/en/mariadb-connector-j/)
- [MariaDB Connector/Python 1.2+](https://mariadb.com/kb/en/mariadb-connector-python/)

### Articles et Blogs
- **"Transaction Replay Deep Dive"** - MariaDB Engineering Blog
- **"Connection Migration Internals"** - MariaDB Corporation
- **"Zero-Downtime Failover with MariaDB 11.8"** - DBA Tutorial

### Webinars
- **"What's New in MariaDB 11.8 LTS"** - MariaDB OpenWorks 2025
- **"High Availability Revolution"** - Percona Live 2025

---

## 🎓 Conclusion du Chapitre 14

Vous avez maintenant une compréhension exhaustive de la haute disponibilité avec MariaDB 11.8 LTS :

1. **Architectures HA** : Fondations théoriques (CAP, RTO/RPO)
2. **Galera Cluster** : Synchronous multi-master, certification-based replication
3. **Split-brain et Quorum** : Prévention, détection, résolution
4. **MaxScale** : Proxy intelligent, nouveautés 25.01 (Workload Capture/Replay/Diff)
5. **Failover Automatique** : mariadbmon, Orchestrator, stratégies
6. **Virtual IP** : keepalived, VRRP, cloud alternatives
7. **Disaster Recovery** : Tests, runbooks, post-mortems
8. **Alternatives** : ProxySQL, HAProxy, comparaisons objectives
9. **🆕 Transaction Replay & Connection Migration** : Révolution failover transparent

**Message final** : La haute disponibilité n'est plus un luxe mais une nécessité. MariaDB 11.8 avec Transaction Replay et Connection Migration élève la barre de ce qui est possible, rendant les failovers pratiquement invisibles pour les utilisateurs finaux. Combiné avec MaxScale 25.01 et une architecture bien conçue, vous pouvez atteindre des disponibilités de 99.99%+ (moins de 1 heure de downtime par an).

**Prochaine étape** : Mettre en pratique ces concepts dans votre infrastructure, tester régulièrement vos procédures de failover, et améliorer continuellement votre résilience.

---

**"La meilleure architecture HA est celle qui est testée en production, documentée minutieusement, et améliorée après chaque incident. MariaDB 11.8 vous donne les outils ; à vous de construire la résilience."**

⏭️ [Performance et Tuning](/15-performance-tuning/README.md)

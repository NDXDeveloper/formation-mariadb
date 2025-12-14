🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.2 Connection Pooling

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Compréhension des connexions à MariaDB (Section 17.1)
> - Notions de concurrence et threads
> - Compréhension des performances réseau
> - Expérience en développement d'applications web/API

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** pourquoi le connection pooling est indispensable en production
- **Identifier** les problèmes de performance liés à l'absence de pooling
- **Configurer** un pool de connexions adapté à votre charge applicative
- **Dimensionner** correctement la taille de votre pool
- **Implémenter** le pooling côté application dans différents langages
- **Utiliser** ProxySQL comme pooler centralisé pour les architectures distribuées
- **Monitorer** les métriques critiques d'un connection pool
- **Optimiser** les performances en ajustant les paramètres du pool

---

## Introduction

Le **connection pooling** est une technique fondamentale d'optimisation qui **réutilise les connexions** à la base de données au lieu d'en créer de nouvelles à chaque requête. Cette approche transforme radicalement les performances de vos applications en production.

### 🚀 Impact du connection pooling

**Sans pooling** :
```
Requête 1 : Connexion → Auth → Query → Disconnect (100ms overhead)
Requête 2 : Connexion → Auth → Query → Disconnect (100ms overhead)
Requête 3 : Connexion → Auth → Query → Disconnect (100ms overhead)
...
Total : N × 100ms de surcharge !
```

**Avec pooling** :
```
Initialisation : Créer 10 connexions (1 fois, 1s au total)
Requête 1 : Emprunter → Query → Rendre (1ms overhead)
Requête 2 : Emprunter → Query → Rendre (1ms overhead)
Requête 3 : Emprunter → Query → Rendre (1ms overhead)
...
Total : ~0ms de surcharge en régime établi !
```

💡 **Gain typique** : **50-100x plus rapide** que la création/fermeture répétée de connexions.

### 📊 Pourquoi le pooling est critique

| Scénario | Sans pooling | Avec pooling |
|----------|--------------|--------------|
| **Latence par requête** | +50-200ms | +0-2ms |
| **Throughput (req/s)** | 10-50 | 1000-5000 |
| **Charge serveur DB** | Très élevée (handshakes) | Minimale |
| **Scalabilité** | Limitée (connexions) | Excellente |
| **Résilience** | Faible (timeout longs) | Bonne (retry, circuit breaker) |

⚠️ **En production** : Ne **JAMAIS** ouvrir/fermer une connexion par requête, c'est un anti-pattern majeur.

---

## Fonctionnement d'un connection pool

### 🏊 Métaphore de la piscine

Imaginez un **pool de connexions** comme une **piscine de connexions prêtes à l'emploi** :

```
┌─────────────────────────────────────────────────┐
│          CONNECTION POOL (taille=10)            │
│                                                 │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐             │
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ │ C5 │  (IDLE)     │
│  └────┘ └────┘ └────┘ └────┘ └────┘             │
│                                                 │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐             │
│  │ C6 │ │ C7 │ │ C8 │ │ C9 │ │C10 │  (IDLE)     │
│  └────┘ └────┘ └────┘ └────┘ └────┘             │
└─────────────────────────────────────────────────┘
          ▲                            │
          │                            │
     getConnection()              release()
          │                            ▼
    ┌─────────────┐            ┌─────────────┐
    │ Thread 1    │            │ Thread 2    │
    │ (utilise C1)│            │ (rend C7)   │
    └─────────────┘            └─────────────┘
```

### 🔄 Cycle de vie d'une connexion poolée

1. **Initialisation** : Le pool crée `minSize` connexions au démarrage
2. **Emprunt** : L'application demande une connexion via `getConnection()`
   - Si disponible → retournée immédiatement
   - Si toutes occupées → attend (`acquireTimeout`) ou crée nouvelle (jusqu'à `maxSize`)
3. **Utilisation** : Exécution de requêtes SQL
4. **Retour** : L'application appelle `release()` ou `close()` (ne ferme pas vraiment, rend au pool)
5. **Recyclage** : La connexion est nettoyée (rollback auto, reset variables) et remise en IDLE
6. **Éviction** : Si inactif trop longtemps (`idleTimeout`), la connexion est fermée

💡 **Clé** : `close()` sur une connexion poolée ne la ferme **pas**, elle la **rend au pool** !

---

## Paramètres essentiels d'un pool

### ⚙️ Configuration universelle

Tous les pools (quel que soit le langage) partagent ces paramètres fondamentaux :

| Paramètre | Description | Valeur typique | Impact |
|-----------|-------------|----------------|--------|
| **minSize** | Nombre minimal de connexions maintenues | 5-10 | Latence au démarrage |
| **maxSize** | Nombre maximal de connexions autorisées | 20-100 | Limite de charge |
| **acquireTimeout** | Temps max d'attente pour obtenir une connexion | 10-30s | Expérience utilisateur |
| **idleTimeout** | Temps avant fermeture connexion inactive | 600s (10min) | Ressources serveur |
| **maxLifetime** | Durée de vie maximale d'une connexion | 1800s (30min) | Recyclage connexions |
| **connectionTimeout** | Timeout création nouvelle connexion | 10-30s | Résilience |
| **leakDetectionThreshold** | Détection de fuites (connexions non rendues) | 60s | Debug |

### 🎯 Dimensionnement du pool

**Formule de base** (Hikari, créateur de HikariCP) :

```
maxPoolSize = ((core_count × 2) + effective_spindle_count)
```

**Pour un serveur moderne** :
- 4 cœurs, SSD (spindle=1) → **9 connexions**
- 8 cœurs, SSD (spindle=1) → **17 connexions**
- 16 cœurs, SSD (spindle=1) → **33 connexions**

💡 **Principe** : Plus de connexions n'est **pas toujours mieux** ! Au-delà d'un certain seuil, la contention ralentit tout.

**Règles empiriques** :

| Type d'application | minSize | maxSize | Justification |
|-------------------|---------|---------|---------------|
| **API REST (stateless)** | 10 | 20-50 | Requêtes courtes, besoin rapide |
| **Web app (sessions)** | 20 | 50-100 | Transactions longues, utilisateurs simultanés |
| **Microservice** | 5 | 20 | Pool par service, scaling horizontal |
| **Batch/ETL** | 5 | 10 | Parallélisme contrôlé, requêtes longues |
| **Serverless (Lambda, etc.)** | 1 | 10 | Cold start, durée limitée |

⚠️ **Piège** : `maxSize` trop grand → contention MariaDB, mémoire gaspillée, deadlocks.

### 📐 Calcul selon la charge

**Méthode de Little** :

```
Connexions nécessaires = (Requêtes/seconde) × (Durée moyenne requête en secondes)
```

**Exemple** :
- 1000 req/s
- Durée moyenne : 50ms (0.05s)
- Connexions nécessaires = 1000 × 0.05 = **50 connexions**

Ajoutez 20-30% de marge → **60-65 connexions**

💡 **Conseil** : Commencez petit (20), mesurez, augmentez si nécessaire.

---

## Connection pooling côté application

### 📦 Approche intégrée

Le pool est **dans votre application**, chaque instance a son propre pool.

```
┌─────────────────────────┐
│   Application Instance  │
│                         │
│  ┌──────────────────┐   │      ┌────────────┐
│  │  Connection Pool │───┼──────┤  MariaDB   │
│  │  (20 connexions) │   │      │   Server   │
│  └──────────────────┘   │      └────────────┘
│                         │
└─────────────────────────┘

┌─────────────────────────┐
│   Application Instance  │
│                         │
│  ┌──────────────────┐   │      ┌────────────┐
│  │  Connection Pool │───┼──────┤  MariaDB   │
│  │  (20 connexions) │   │      │   Server   │
│  └──────────────────┘   │      └────────────┘
│                         │
└─────────────────────────┘

Total sur MariaDB : 2 × 20 = 40 connexions
```

#### ✅ Avantages

- ✅ **Simple** : Pas de composant externe à gérer
- ✅ **Rapide** : Latence minimale (pas de proxy)
- ✅ **Isolation** : Chaque app contrôle son pool
- ✅ **Natif** : Support dans toutes les bibliothèques modernes

#### ❌ Inconvénients

- ❌ **Multiplication** : N instances × M connexions = surcharge si mal dimensionné
- ❌ **Pas de mutualisation** : Gaspillage si certaines apps peu utilisées
- ❌ **Pas de routing** : Connexions figées (pas de failover avancé)
- ❌ **Monitoring dispersé** : Difficile d'avoir une vue globale

#### 🎯 Cas d'usage idéaux

- ✅ Monolithes
- ✅ Applications avec charge stable
- ✅ Peu d'instances (< 10)
- ✅ Stack technologique homogène

---

## Connection pooling via proxy (ProxySQL)

### 🚪 Approche centralisée

Un **proxy** (comme ProxySQL) gère le pool pour **toutes les applications**.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App 1   │  │  App 2   │  │  App 3   │
│ (10 conn)│  │ (15 conn)│  │ (5 conn) │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   │
            ┌──────▼───────────┐
            │    ProxySQL      │
            │                  │
            │ ┌─────────────┐  │
            │ │  Pool Global│  │
            │ │ (50 connexions)│
            │ └─────────────┘  │
            └────────┬─────────┘
                     │
              ┌──────▼──────┐
              │   MariaDB   │
              │   Server    │
              └─────────────┘

Total sur MariaDB : 50 connexions (au lieu de 30 si pooling app)
```

#### ✅ Avantages

- ✅ **Mutualisation** : Pool partagé entre toutes les apps
- ✅ **Routing avancé** : Read/Write split, Query routing, Failover
- ✅ **Monitoring centralisé** : Vue globale des connexions
- ✅ **Cache de requêtes** : Amélioration performances
- ✅ **Firewall SQL** : Sécurité (blocage requêtes dangereuses)
- ✅ **Multiplexing** : Une connexion app → plusieurs connexions backend

#### ❌ Inconvénients

- ❌ **Complexité** : Composant additionnel à gérer, déployer, monitorer
- ❌ **Latence** : +1-5ms (hop réseau supplémentaire)
- ❌ **SPOF** : Point de défaillance unique (nécessite redondance)
- ❌ **Coût** : Ressources serveur dédiées

#### 🎯 Cas d'usage idéaux

- ✅ **Microservices** : Nombreux services, scaling horizontal
- ✅ **Architecture distribuée** : Multi-datacenter, multi-région
- ✅ **Charge variable** : Peak hours, auto-scaling
- ✅ **Haute disponibilité** : Failover automatique, read/write split
- ✅ **Sécurité renforcée** : Firewall SQL, audit centralisé

### 🔧 ProxySQL : Configuration de base

**Installation** :
```bash
# Debian/Ubuntu
apt-get install proxysql

# RedHat/CentOS
yum install proxysql

# Docker
docker run -d -p 6032:6032 -p 6033:6033 proxysql/proxysql
```

**Configuration minimale** (`/etc/proxysql.cnf`) :
```ini
datadir="/var/lib/proxysql"

admin_variables=
{
    admin_credentials="admin:admin"
    mysql_ifaces="0.0.0.0:6032"
}

mysql_variables=
{
    threads=4
    max_connections=2048
    default_query_delay=0
    default_query_timeout=36000000
    have_compress=true
    poll_timeout=2000
    interfaces="0.0.0.0:6033"
    default_schema="information_schema"
    stacksize=1048576
    server_version="5.5.30"
    connect_timeout_server=3000
    monitor_username="monitor"
    monitor_password="monitor"
    monitor_history=600000
    monitor_connect_interval=60000
    monitor_ping_interval=10000
    monitor_read_only_interval=1500
    monitor_read_only_timeout=500
    ping_interval_server_msec=120000
    ping_timeout_server=500
    commands_stats=true
    sessions_sort=true
    connect_retries_on_failure=10
}

# Serveurs backend MariaDB
mysql_servers =
(
    {
        address="192.168.1.10"
        port=3306
        hostgroup=0  # Write hostgroup
        max_connections=200
        max_replication_lag=10
    },
    {
        address="192.168.1.11"
        port=3306
        hostgroup=1  # Read hostgroup
        max_connections=500
        max_replication_lag=10
    }
)

# Utilisateurs
mysql_users =
(
    {
        username = "app_user"
        password = "app_password"
        default_hostgroup = 0
        max_connections = 1000
        active = 1
    }
)

# Règles de routing (optionnel)
mysql_query_rules =
(
    {
        rule_id=1
        active=1
        match_digest="^SELECT.*FOR UPDATE"
        destination_hostgroup=0  # Write
        apply=1
    },
    {
        rule_id=2
        active=1
        match_digest="^SELECT"
        destination_hostgroup=1  # Read
        apply=1
    }
)
```

**Connexion via ProxySQL** :
```bash
# Applications se connectent au port 6033 (pas 3306)
mysql -h proxysql_host -P 6033 -u app_user -p
```

---

## Exemples par langage

### 🐘 **PHP - Pas de pooling natif**

⚠️ **Limitation PHP** : PHP est **stateless** (requête → processus/thread éphémère), pas de pool natif dans le runtime.

**Solutions** :
1. **Connexions persistantes** (`PDO::ATTR_PERSISTENT`)
2. **ProxySQL** (recommandé en production)
3. **PHP-FPM + pool manager externe**

**PDO avec connexions persistantes** :
```php
<?php
// ATTENTION : Pas un vrai pool, réutilisation limitée au même worker PHP-FPM
$options = [
    PDO::ATTR_PERSISTENT => true,  // Tente de réutiliser la connexion
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES => false,
];

$pdo = new PDO(
    'mysql:host=localhost;dbname=mydb;charset=utf8mb4',
    'user',
    'password',
    $options
);

// Utilisation normale
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([123]);
$user = $stmt->fetch();
```

💡 **Recommandation PHP** : Utiliser **ProxySQL** pour un vrai pooling en production.

---

### 🐍 **Python - mysql-connector avec pool natif**

**Configuration du pool** :
```python
import mysql.connector
from mysql.connector import pooling

# Configuration du pool
pool_config = {
    "pool_name": "myapp_pool",
    "pool_size": 10,              # Équivalent minSize et maxSize
    "pool_reset_session": True,   # Reset variables session après usage
    "host": "localhost",
    "user": "app_user",
    "password": "secret",
    "database": "mydb",
    "charset": "utf8mb4",
    "autocommit": False,
    "get_warnings": True,
}

# Création du pool (une fois au démarrage de l'app)
connection_pool = mysql.connector.pooling.MySQLConnectionPool(**pool_config)

# Utilisation dans une fonction
def get_user(user_id):
    # Obtenir une connexion du pool
    conn = connection_pool.get_connection()
    try:
        cursor = conn.cursor(dictionary=True)
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        user = cursor.fetchone()
        return user
    except mysql.connector.Error as err:
        print(f"Database error: {err}")
        raise
    finally:
        cursor.close()
        conn.close()  # Rend la connexion au pool (ne ferme pas vraiment)

# Test
user = get_user(123)
print(user)
```

**Avec context manager (plus propre)** :
```python
from contextlib import contextmanager

@contextmanager
def get_db_connection():
    """Context manager pour connexions poolées"""
    conn = connection_pool.get_connection()
    try:
        yield conn
    finally:
        conn.close()  # Retour au pool

# Utilisation
def get_users():
    with get_db_connection() as conn:
        cursor = conn.cursor(dictionary=True)
        cursor.execute("SELECT * FROM users LIMIT 10")
        return cursor.fetchall()
```

**SQLAlchemy avec pool (recommandé pour ORM)** :
```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Engine avec pool configuré
engine = create_engine(
    'mysql+mysqlconnector://user:password@localhost/mydb',
    poolclass=QueuePool,
    pool_size=10,              # Connexions maintenues
    max_overflow=20,           # Connexions additionnelles en pic
    pool_timeout=30,           # Timeout acquireTimeout
    pool_recycle=3600,         # Recyclage connexions (1h)
    pool_pre_ping=True,        # Test connexion avant utilisation
    echo_pool=True,            # Logging du pool (debug)
)

# Utilisation avec session
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(bind=engine)

def get_user(user_id):
    session = Session()
    try:
        user = session.query(User).filter_by(id=user_id).first()
        return user
    finally:
        session.close()  # Rend la connexion au pool
```

---

### ☕ **Java - HikariCP (le standard)**

**HikariCP** : Pool le plus performant de l'écosystème Java.

**Configuration avec propriétés** :
```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class DatabaseConfig {
    private static HikariDataSource dataSource;
    
    static {
        HikariConfig config = new HikariConfig();
        
        // Connexion
        config.setJdbcUrl("jdbc:mariadb://localhost:3306/mydb");
        config.setUsername("app_user");
        config.setPassword("secret");
        
        // Pool sizing
        config.setMinimumIdle(10);              // minSize
        config.setMaximumPoolSize(20);          // maxSize
        config.setConnectionTimeout(30000);     // 30s
        config.setIdleTimeout(600000);          // 10min
        config.setMaxLifetime(1800000);         // 30min
        
        // Performance
        config.setAutoCommit(true);
        config.setConnectionTestQuery("SELECT 1");
        config.setPoolName("MyAppPool");
        
        // Leak detection (debug)
        config.setLeakDetectionThreshold(60000); // 60s
        
        // Propriétés MariaDB
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        
        dataSource = new HikariDataSource(config);
    }
    
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
    
    public static void close() {
        if (dataSource != null) {
            dataSource.close();
        }
    }
}

// Utilisation
public class UserRepository {
    public User findById(int userId) throws SQLException {
        String sql = "SELECT * FROM users WHERE id = ?";
        
        try (Connection conn = DatabaseConfig.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setInt(1, userId);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return mapToUser(rs);
                }
            }
        }
        return null;
    }
}
```

**Spring Boot (configuration automatique)** :
```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/mydb
    username: app_user
    password: secret
    driver-class-name: org.mariadb.jdbc.Driver
    hikari:
      minimum-idle: 10
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      pool-name: MyAppHikariPool
      connection-test-query: SELECT 1
```

---

### 🟢 **Node.js - Pool natif mysql2 et mariadb**

**mysql2 avec pool** :
```javascript
const mysql = require('mysql2/promise');

// Création du pool
const pool = mysql.createPool({
    host: 'localhost',
    user: 'app_user',
    password: 'secret',
    database: 'mydb',
    waitForConnections: true,      // Attendre si toutes connexions occupées
    connectionLimit: 10,            // maxSize
    queueLimit: 0,                  // Pas de limite de queue (0 = illimité)
    enableKeepAlive: true,          // Keep-alive TCP
    keepAliveInitialDelay: 0,
    charset: 'utf8mb4',
    timezone: 'Z',                  // UTC
});

// Utilisation avec async/await
async function getUser(userId) {
    try {
        const [rows] = await pool.query(
            'SELECT * FROM users WHERE id = ?',
            [userId]
        );
        return rows[0];
    } catch (error) {
        console.error('Database error:', error);
        throw error;
    }
}

// Avec connexion explicite (plus de contrôle)
async function createUser(userData) {
    const connection = await pool.getConnection();
    try {
        await connection.beginTransaction();
        
        const [result] = await connection.query(
            'INSERT INTO users (name, email) VALUES (?, ?)',
            [userData.name, userData.email]
        );
        
        await connection.commit();
        return result.insertId;
    } catch (error) {
        await connection.rollback();
        throw error;
    } finally {
        connection.release();  // Retour au pool
    }
}

// Fermeture du pool (shutdown graceful)
process.on('SIGINT', async () => {
    await pool.end();
    process.exit(0);
});
```

**mariadb (officiel) avec pool** :
```javascript
const mariadb = require('mariadb');

const pool = mariadb.createPool({
    host: 'localhost',
    user: 'app_user',
    password: 'secret',
    database: 'mydb',
    connectionLimit: 10,
    acquireTimeout: 30000,         // 30s
    idleTimeout: 600000,           // 10min
    minimumIdle: 5,                // minSize
    charset: 'utf8mb4',
    trace: true,                   // Debug
});

async function getUsers() {
    let conn;
    try {
        conn = await pool.getConnection();
        const rows = await conn.query('SELECT * FROM users LIMIT 10');
        return rows;
    } catch (err) {
        throw err;
    } finally {
        if (conn) conn.release();
    }
}

// Monitoring du pool
setInterval(() => {
    console.log('Pool stats:', {
        activeConnections: pool.activeConnections(),
        totalConnections: pool.totalConnections(),
        idleConnections: pool.idleConnections(),
        taskQueueSize: pool.taskQueueSize()
    });
}, 60000);  // Toutes les minutes
```

---

### 🔵 **Go - Pool natif dans database/sql**

Go a un **pool intégré** dans `database/sql`, très performant.

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"
    
    _ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

func initDB() error {
    var err error
    
    dsn := "app_user:secret@tcp(localhost:3306)/mydb?charset=utf8mb4&parseTime=true&loc=UTC"
    
    db, err = sql.Open("mysql", dsn)
    if err != nil {
        return err
    }
    
    // Configuration du pool
    db.SetMaxOpenConns(20)               // maxSize
    db.SetMaxIdleConns(10)               // minSize
    db.SetConnMaxLifetime(30 * time.Minute)   // maxLifetime
    db.SetConnMaxIdleTime(10 * time.Minute)   // idleTimeout
    
    // Test de la connexion
    err = db.Ping()
    if err != nil {
        return err
    }
    
    fmt.Println("Database connection pool initialized")
    return nil
}

// Requête simple
func getUser(userID int) (*User, error) {
    var user User
    
    query := "SELECT id, name, email FROM users WHERE id = ?"
    err := db.QueryRow(query, userID).Scan(&user.ID, &user.Name, &user.Email)
    
    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}

// Transaction (connexion explicite)
func createUser(name, email string) (int64, error) {
    tx, err := db.Begin()
    if err != nil {
        return 0, err
    }
    defer tx.Rollback()  // Rollback si pas de commit
    
    result, err := tx.Exec(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        name, email,
    )
    if err != nil {
        return 0, err
    }
    
    if err := tx.Commit(); err != nil {
        return 0, err
    }
    
    return result.LastInsertId()
}

// Monitoring du pool
func monitorPool() {
    stats := db.Stats()
    log.Printf("Pool stats: "+
        "OpenConnections=%d, InUse=%d, Idle=%d, "+
        "WaitCount=%d, WaitDuration=%s",
        stats.OpenConnections,
        stats.InUse,
        stats.Idle,
        stats.WaitCount,
        stats.WaitDuration,
    )
}

func main() {
    if err := initDB(); err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // Monitoring périodique
    ticker := time.NewTicker(60 * time.Second)
    go func() {
        for range ticker.C {
            monitorPool()
        }
    }()
    
    // Utilisation
    user, err := getUser(123)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User: %+v\n", user)
}
```

---

### 🔷 **.NET - Pool natif dans MySqlConnector**

**.NET** a un pool **automatique et transparent** dans tous les connecteurs ADO.NET.

```csharp
using System;
using System.Data;
using MySqlConnector;

public class DatabaseConfig
{
    // Connection string avec pool configuré
    private static readonly string ConnectionString = 
        "Server=localhost;" +
        "Database=mydb;" +
        "User ID=app_user;" +
        "Password=secret;" +
        "Pooling=true;" +                    // Active le pool (défaut)
        "MinimumPoolSize=10;" +               // minSize
        "MaximumPoolSize=20;" +               // maxSize
        "ConnectionTimeout=30;" +             // 30s
        "ConnectionIdleTimeout=600;" +        // 10min
        "ConnectionLifeTime=1800;" +          // 30min
        "CharSet=utf8mb4;";
    
    // Requête simple
    public static async Task<User?> GetUserAsync(int userId)
    {
        using var connection = new MySqlConnection(ConnectionString);
        await connection.OpenAsync();
        
        using var command = new MySqlCommand(
            "SELECT id, name, email FROM users WHERE id = @userId",
            connection
        );
        command.Parameters.AddWithValue("@userId", userId);
        
        using var reader = await command.ExecuteReaderAsync();
        if (await reader.ReadAsync())
        {
            return new User
            {
                Id = reader.GetInt32("id"),
                Name = reader.GetString("name"),
                Email = reader.GetString("email")
            };
        }
        
        return null;
    }
    
    // Transaction
    public static async Task<long> CreateUserAsync(string name, string email)
    {
        using var connection = new MySqlConnection(ConnectionString);
        await connection.OpenAsync();
        
        using var transaction = await connection.BeginTransactionAsync();
        try
        {
            using var command = new MySqlCommand(
                "INSERT INTO users (name, email) VALUES (@name, @email)",
                connection,
                transaction
            );
            command.Parameters.AddWithValue("@name", name);
            command.Parameters.AddWithValue("@email", email);
            
            await command.ExecuteNonQueryAsync();
            var userId = command.LastInsertedId;
            
            await transaction.CommitAsync();
            return userId;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// Utilisation dans ASP.NET Core
public class UserController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(int id)
    {
        try
        {
            var user = await DatabaseConfig.GetUserAsync(id);
            if (user == null)
                return NotFound();
            
            return Ok(user);
        }
        catch (MySqlException ex)
        {
            _logger.LogError(ex, "Database error getting user {UserId}", id);
            return StatusCode(500);
        }
    }
}
```

**Monitoring du pool (.NET)** :
```csharp
// Statistiques non exposées directement par MySqlConnector
// Utiliser des compteurs de performance ou logs
public static void LogPoolStats()
{
    // Alternative : Utiliser Application Insights ou Prometheus
    Console.WriteLine("Pool statistics not directly available in MySqlConnector");
    Console.WriteLine("Use performance counters or external monitoring");
}
```

---

## Monitoring et métriques

### 📊 Métriques critiques à surveiller

| Métrique | Signification | Seuil alerte | Action |
|----------|---------------|--------------|--------|
| **Active connections** | Connexions en cours d'utilisation | > 80% maxSize | Augmenter maxSize |
| **Idle connections** | Connexions disponibles | < 20% maxSize | Réduire minSize |
| **Wait count** | Threads en attente de connexion | > 0 fréquent | Augmenter maxSize |
| **Wait duration** | Temps d'attente moyen | > 1s | Pool sous-dimensionné |
| **Connection errors** | Échecs de connexion | > 1% | Problème réseau/serveur |
| **Leaked connections** | Connexions non rendues | > 0 | Bug applicatif |
| **Connection creation rate** | Nouvelles connexions/s | > 10/s | Pool instable |

### 🔍 Requêtes monitoring côté MariaDB

**Connexions actives** :
```sql
-- Connexions par utilisateur
SELECT USER, COUNT(*) as connections
FROM information_schema.PROCESSLIST
GROUP BY USER
ORDER BY connections DESC;

-- Connexions par état
SELECT COMMAND, COUNT(*) as count
FROM information_schema.PROCESSLIST
GROUP BY COMMAND
ORDER BY count DESC;

-- Connexions longues (> 1min)
SELECT ID, USER, HOST, DB, COMMAND, TIME, STATE, INFO
FROM information_schema.PROCESSLIST
WHERE TIME > 60
ORDER BY TIME DESC;
```

**Statistiques globales** :
```sql
-- Total connexions depuis démarrage
SHOW GLOBAL STATUS LIKE 'Connections';

-- Connexions max simultanées
SHOW GLOBAL STATUS LIKE 'Max_used_connections';

-- Limite configurée
SHOW VARIABLES LIKE 'max_connections';

-- Connexions refusées (saturation)
SHOW GLOBAL STATUS LIKE 'Connection_errors_max_connections';

-- Thread cache (efficacité)
SHOW GLOBAL STATUS LIKE 'Threads_%';
```

---

## ⚠️ Pièges courants et solutions

### 🐛 Problème 1 : Pool sous-dimensionné

**Symptôme** :
```
ERROR: Timeout waiting for connection (30s)
WARN: High wait count in pool
```

**Cause** : `maxSize` trop petit pour la charge.

**Diagnostic** :
```python
# Monitoring du pool
stats = pool.get_stats()
print(f"Active: {stats['active']} / {stats['max_size']}")
print(f"Wait count: {stats['wait_count']}")

if stats['active'] / stats['max_size'] > 0.8:
    print("WARNING: Pool near capacity!")
```

**Solution** :
1. Augmenter `maxSize` progressivement (+20%)
2. Vérifier que MariaDB supporte (`max_connections`)
3. Optimiser durée des requêtes (diminuer le besoin)

### 🐛 Problème 2 : Pool sur-dimensionné

**Symptôme** :
```
WARN: 80% idle connections
WARN: High memory usage on MariaDB
```

**Cause** : `minSize` ou `maxSize` trop grands.

**Solution** :
```yaml
# Avant
min_size: 50
max_size: 100

# Après
min_size: 10
max_size: 30
```

💡 **Principe** : Commencer petit, augmenter si nécessaire.

### 🐛 Problème 3 : Connexions leaked (non rendues)

**Symptôme** :
```
ERROR: Maximum pool size reached and all connections in use
WARN: Leak detection threshold exceeded (60s)
```

**Cause** : Code ne rend pas la connexion au pool.

**Mauvais code** :
```python
def bad_function():
    conn = pool.get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT ...")
    # OUBLI : conn.close() jamais appelé !
    return cursor.fetchall()
```

**Solution** :
```python
def good_function():
    conn = pool.get_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT ...")
        return cursor.fetchall()
    finally:
        conn.close()  # TOUJOURS dans finally

# Ou mieux : context manager
def better_function():
    with pool.get_connection() as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT ...")
        return cursor.fetchall()
    # conn.close() automatique
```

**Detection automatique** :
```java
// HikariCP
config.setLeakDetectionThreshold(60000);  // 60s

// Log si connexion tenue > 60s sans être rendue
// WARN: Apparent connection leak detected
```

### 🐛 Problème 4 : Connexions mortes (stale)

**Symptôme** :
```
ERROR: MySQL server has gone away
ERROR: Lost connection to MySQL server during query
```

**Cause** : Connexion inactive trop longtemps, fermée par MariaDB.

**Solution** :
```python
pool = mysql.connector.pooling.MySQLConnectionPool(
    pool_reset_session=True,        # Reset après usage
    pool_recycle=3600,              # Recycler toutes les heures
)

# Ou : Test avant utilisation (pool_pre_ping)
from sqlalchemy import create_engine

engine = create_engine(
    'mysql://...',
    pool_pre_ping=True  # Ping avant chaque utilisation
)
```

---

## ✅ Points clés à retenir

- 🏊 **Connection pooling** : Indispensable en production, évite de créer/fermer les connexions
- 📏 **Dimensionnement** : Commencer petit (10-20), mesurer, ajuster selon la charge
- 🔄 **Cycle de vie** : Emprunter → Utiliser → Rendre (toujours dans finally/using/defer)
- 📦 **Côté application** : Simple, faible latence, idéal pour monolithes
- 🚪 **ProxySQL** : Mutualisation, routing avancé, idéal pour microservices
- 📊 **Monitoring** : Surveiller active/idle connections, wait count, leaked connections
- ⚙️ **Paramètres critiques** : minSize, maxSize, acquireTimeout, maxLifetime
- 💡 **Formule Hikari** : `maxSize = (cores × 2) + spindles` (point de départ)
- 🐛 **Pièges** : Pool sous/sur-dimensionné, connexions leaked, connexions stale
- 🆕 **MariaDB 11.8** : Excellent support des pools modernes, UTF8MB4 et TLS par défaut

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [HikariCP Best Practices](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- 📖 [ProxySQL Documentation](https://proxysql.com/documentation/)
- 📖 [MariaDB max_connections](https://mariadb.com/kb/en/server-system-variables/#max_connections)

### **Guides de configuration**
- 🔗 [SQLAlchemy Pooling](https://docs.sqlalchemy.org/en/14/core/pooling.html)
- 🔗 [Node.js mysql2 Pool](https://github.com/sidorares/node-mysql2#using-connection-pools)
- 🔗 [Go database/sql Tutorial](https://go.dev/doc/database/manage-connections)

### **Articles avancés**
- 📝 [The Anatomy of Connection Pooling](https://vladmihalcea.com/the-anatomy-of-connection-pooling/)
- 📝 [ProxySQL Query Routing](https://proxysql.com/documentation/query-routing/)

---

## ➡️ Sections suivantes

- **17.2.1** - Pool côté application : Implémentations détaillées par langage
- **17.2.2** - ProxySQL comme pooler : Configuration avancée, routing, monitoring

---

**MariaDB** : Version 11.8 LTS

⏭️ [Pool côté application](/17-integration-developpement/02.1-pool-application.md)

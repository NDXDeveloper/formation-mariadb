🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.9 Prepared statements et parameterized queries

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Maîtrise du SQL (Chapitres 2-4)
> - Compréhension des injections SQL (Section 17.8)
> - Connexions à MariaDB (Section 17.1)
> - Notions de performance et optimisation

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le fonctionnement interne des prepared statements
- **Différencier** les types de prepared statements (client-side vs server-side)
- **Implémenter** correctement dans tous les langages majeurs
- **Optimiser** les performances avec le statement caching
- **Identifier** quand utiliser ou non les prepared statements
- **Mesurer** l'impact sur les performances
- **Résoudre** les limitations et cas particuliers
- **Débugger** les problèmes liés aux prepared statements

---

## Introduction

Les **prepared statements** (requêtes préparées) sont le mécanisme fondamental pour :

- 🛡️ **Sécurité** : Protection contre les injections SQL
- ⚡ **Performance** : Réutilisation du plan d'exécution
- 📦 **Séparation** : Code SQL vs données
- 🎯 **Type safety** : Validation des types de données

### 📊 Comparaison rapide

| Aspect | Requêtes dynamiques | Prepared statements |
|--------|---------------------|---------------------|
| **Sécurité** | ❌ Vulnérable aux injections | ✅ Protection complète |
| **Performance (1x)** | ⚡ Rapide | 🐢 Overhead de préparation |
| **Performance (100x)** | 🐢 Parse à chaque fois | ⚡ Parse une fois |
| **Complexité** | 🟢 Simple | 🟡 Modérée |
| **Type checking** | ❌ Aucun | ✅ Validation automatique |

---

## Fonctionnement interne

### 🔄 Cycle de vie d'un prepared statement

```
1. PREPARE (une fois)
   ├─ Parse SQL
   ├─ Analyse syntaxique
   ├─ Optimisation
   └─ Plan d'exécution
         ↓
2. BIND (pour chaque exécution)
   └─ Lier les valeurs aux placeholders
         ↓
3. EXECUTE (pour chaque exécution)
   └─ Exécuter avec les valeurs liées
         ↓
4. CLOSE (optionnel)
   └─ Libérer les ressources
```

### 🎭 Requête dynamique vs Prepared statement

**Requête dynamique** :
```sql
-- Chaque exécution parse et optimise
SELECT * FROM users WHERE email = 'alice@example.com';  -- Parse + Execute
SELECT * FROM users WHERE email = 'bob@example.com';    -- Parse + Execute (redondant!)
SELECT * FROM users WHERE email = 'charlie@example.com'; -- Parse + Execute (redondant!)
```

**Prepared statement** :
```sql
-- Préparation (une fois)
PREPARE stmt FROM 'SELECT * FROM users WHERE email = ?';

-- Exécutions (plusieurs fois, rapide)
SET @email = 'alice@example.com';
EXECUTE stmt USING @email;

SET @email = 'bob@example.com';
EXECUTE stmt USING @email;

SET @email = 'charlie@example.com';
EXECUTE stmt USING @email;

-- Nettoyage
DEALLOCATE PREPARE stmt;
```

### 🖥️ Client-side vs Server-side

#### **Server-side prepared statements** (MariaDB)

```
Application                     MariaDB Server
     │                               │
     ├─── PREPARE ──────────────────>│
     │                               ├─ Parse SQL
     │                               ├─ Optimize
     │                               └─ Store (ID: 1)
     │<────── Statement ID 1 ────────┤
     │                               │
     ├─── EXECUTE ID:1 + values ────>│
     │                               ├─ Bind values
     │                               └─ Execute plan
     │<────── Result set ────────────┤
     │                               │
     ├─── EXECUTE ID:1 + values ────>│
     │                               └─ Execute plan (réutilisé)
     │<────── Result set ────────────┤
```

✅ **Avantages** :
- Plan d'exécution mis en cache côté serveur
- Moins de bande passante (SQL envoyé une fois)
- Performance optimale pour requêtes répétées

❌ **Inconvénients** :
- Consomme de la mémoire serveur
- Limite par connexion (max_prepared_stmt_count)

#### **Client-side prepared statements** (émulation)

```
Application                     MariaDB Server
     │                               │
     ├─ Émulation prepare localement │
     │  (parse, store)               │
     │                               │
     ├─── SELECT ... (valeurs) ─────>│
     │    (SQL complet envoyé)       ├─ Parse
     │                               ├─ Optimize
     │                               └─ Execute
     │<────── Result set ────────────┤
```

✅ **Avantages** :
- Pas de limite serveur
- Compatibilité maximale
- Protection contre injections SQL

❌ **Inconvénients** :
- Pas de cache côté serveur
- SQL complet envoyé à chaque fois

---

## Implémentation par langage

### 🐘 PHP (PDO)

**Configuration** :
```php
<?php
$dsn = "mysql:host=localhost;dbname=myapp;charset=utf8mb4";
$options = [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    
    // Server-side prepared statements (recommandé)
    PDO::ATTR_EMULATE_PREPARES => false,
    
    // Type safety
    PDO::ATTR_STRINGIFY_FETCHES => false,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
];

$pdo = new PDO($dsn, 'user', 'password', $options);
```

**Utilisation basique** :
```php
// Préparation
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');

// Exécution 1
$stmt->execute(['email' => 'alice@example.com']);
$user1 = $stmt->fetch();

// Exécution 2 (réutilisation du statement)
$stmt->execute(['email' => 'bob@example.com']);
$user2 = $stmt->fetch();
```

**Binding explicite** :
```php
$stmt = $pdo->prepare('INSERT INTO users (name, email, age) VALUES (:name, :email, :age)');

// Bind avec types
$stmt->bindValue(':name', 'Alice', PDO::PARAM_STR);
$stmt->bindValue(':email', 'alice@example.com', PDO::PARAM_STR);
$stmt->bindValue(':age', 25, PDO::PARAM_INT);

$stmt->execute();
```

**Types PDO** :
```php
PDO::PARAM_NULL   // NULL
PDO::PARAM_INT    // INTEGER
PDO::PARAM_STR    // VARCHAR, TEXT, etc.
PDO::PARAM_BOOL   // BOOLEAN
PDO::PARAM_LOB    // BLOB, large objects
```

**Réutilisation optimale** :
```php
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');

// Batch insert (statement préparé une fois)
$users = [
    ['Alice', 'alice@example.com'],
    ['Bob', 'bob@example.com'],
    ['Charlie', 'charlie@example.com']
];

foreach ($users as $user) {
    $stmt->execute($user);
}

echo $stmt->rowCount() . " users inserted\n";
```

### 🐍 Python (mysql-connector)

**Configuration** :
```python
import mysql.connector

config = {
    'host': 'localhost',
    'user': 'myapp_user',
    'password': 'myapp_password',
    'database': 'myapp',
    'charset': 'utf8mb4',
    'use_unicode': True,
    
    # Server-side prepared statements
    'use_pure': False,  # C extension (plus rapide)
    'autocommit': False
}

connection = mysql.connector.connect(**config)
cursor = connection.cursor(prepared=True)  # Prepared statements
```

**Utilisation basique** :
```python
# Préparation automatique avec cursor prepared
cursor = connection.cursor(prepared=True)

query = "SELECT * FROM users WHERE email = %s"

# Exécution 1
cursor.execute(query, ('alice@example.com',))
user1 = cursor.fetchone()

# Exécution 2 (statement réutilisé)
cursor.execute(query, ('bob@example.com',))
user2 = cursor.fetchone()
```

**Insert avec auto-commit** :
```python
cursor = connection.cursor(prepared=True)

query = "INSERT INTO users (name, email, age) VALUES (%s, %s, %s)"

cursor.execute(query, ('Alice', 'alice@example.com', 25))
connection.commit()

user_id = cursor.lastrowid
print(f"User created with ID: {user_id}")
```

**Batch insert performant** :
```python
cursor = connection.cursor(prepared=True)

query = "INSERT INTO users (name, email) VALUES (%s, %s)"

users = [
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com')
]

# executemany utilise le même prepared statement
cursor.executemany(query, users)
connection.commit()

print(f"{cursor.rowcount} users inserted")
```

**Context manager** :
```python
from contextlib import contextmanager

@contextmanager
def get_cursor(prepared=True):
    """Context manager pour cursor avec prepared statements"""
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor(prepared=prepared)
    try:
        yield cursor
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()

# Utilisation
with get_cursor() as cursor:
    cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s)", 
                   ('Alice', 'alice@example.com'))
    user_id = cursor.lastrowid
```

### ☕ Java (JDBC)

**Configuration** :
```java
import java.sql.*;

public class DatabaseManager {
    private static final String URL = "jdbc:mariadb://localhost:3306/myapp";
    private static final String USER = "myapp_user";
    private static final String PASSWORD = "myapp_password";
    
    public static Connection getConnection() throws SQLException {
        // Properties pour optimisation
        Properties props = new Properties();
        props.setProperty("user", USER);
        props.setProperty("password", PASSWORD);
        props.setProperty("useServerPrepStmts", "true");  // Server-side
        props.setProperty("cachePrepStmts", "true");      // Cache local
        props.setProperty("prepStmtCacheSize", "250");
        props.setProperty("prepStmtCacheSqlLimit", "2048");
        props.setProperty("useUnicode", "true");
        props.setProperty("characterEncoding", "UTF-8");
        
        return DriverManager.getConnection(URL, props);
    }
}
```

**Utilisation basique** :
```java
String query = "SELECT * FROM users WHERE email = ?";

try (Connection conn = DatabaseManager.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(query)) {
    
    // Exécution 1
    pstmt.setString(1, "alice@example.com");
    ResultSet rs1 = pstmt.executeQuery();
    
    if (rs1.next()) {
        String name = rs1.getString("name");
        System.out.println("User: " + name);
    }
    rs1.close();
    
    // Exécution 2 (statement réutilisé)
    pstmt.setString(1, "bob@example.com");
    ResultSet rs2 = pstmt.executeQuery();
    
    // ...
}
```

**Insert avec generated keys** :
```java
String query = "INSERT INTO users (name, email, age) VALUES (?, ?, ?)";

try (Connection conn = DatabaseManager.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(query, Statement.RETURN_GENERATED_KEYS)) {
    
    pstmt.setString(1, "Alice");
    pstmt.setString(2, "alice@example.com");
    pstmt.setInt(3, 25);
    
    int rowsAffected = pstmt.executeUpdate();
    
    // Récupérer l'ID auto-généré
    try (ResultSet generatedKeys = pstmt.getGeneratedKeys()) {
        if (generatedKeys.next()) {
            long userId = generatedKeys.getLong(1);
            System.out.println("User created with ID: " + userId);
        }
    }
}
```

**Batch insert performant** :
```java
String query = "INSERT INTO users (name, email) VALUES (?, ?)";

try (Connection conn = DatabaseManager.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(query)) {
    
    // Désactiver auto-commit pour performance
    conn.setAutoCommit(false);
    
    String[][] users = {
        {"Alice", "alice@example.com"},
        {"Bob", "bob@example.com"},
        {"Charlie", "charlie@example.com"}
    };
    
    for (String[] user : users) {
        pstmt.setString(1, user[0]);
        pstmt.setString(2, user[1]);
        pstmt.addBatch();
    }
    
    int[] results = pstmt.executeBatch();
    conn.commit();
    
    System.out.println(results.length + " users inserted");
    
} catch (SQLException e) {
    conn.rollback();
    throw e;
}
```

**Tous les types JDBC** :
```java
pstmt.setString(1, "value");
pstmt.setInt(2, 42);
pstmt.setLong(3, 123456789L);
pstmt.setDouble(4, 3.14);
pstmt.setBoolean(5, true);
pstmt.setDate(6, new java.sql.Date(System.currentTimeMillis()));
pstmt.setTimestamp(7, new Timestamp(System.currentTimeMillis()));
pstmt.setNull(8, Types.VARCHAR);
pstmt.setBytes(9, byteArray);
pstmt.setBigDecimal(10, new BigDecimal("99.99"));
```

### 🟢 Node.js (mysql2)

**Configuration** :
```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
    host: 'localhost',
    user: 'myapp_user',
    password: 'myapp_password',
    database: 'myapp',
    charset: 'utf8mb4',
    
    // Connection pool
    connectionLimit: 10,
    queueLimit: 0,
    
    // Prepared statements
    namedPlaceholders: true,  // Support :name et ?
    
    // Performance
    enableKeepAlive: true,
    keepAliveInitialDelay: 10000
});

module.exports = pool;
```

**Utilisation avec execute() (recommandé)** :
```javascript
const pool = require('./database');

async function findUserByEmail(email) {
    // execute() utilise automatiquement prepared statements
    const [rows] = await pool.execute(
        'SELECT * FROM users WHERE email = ?',
        [email]
    );
    
    return rows[0];
}

// Utilisation
const user1 = await findUserByEmail('alice@example.com');
const user2 = await findUserByEmail('bob@example.com');
```

**Named placeholders** :
```javascript
async function createUser(name, email, age) {
    const [result] = await pool.execute(
        'INSERT INTO users (name, email, age) VALUES (:name, :email, :age)',
        { name, email, age }
    );
    
    return result.insertId;
}

const userId = await createUser('Alice', 'alice@example.com', 25);
console.log(`User created with ID: ${userId}`);
```

**Réutilisation explicite** :
```javascript
async function batchInsertUsers(users) {
    const connection = await pool.getConnection();
    
    try {
        // Préparer une fois
        const statement = await connection.prepare(
            'INSERT INTO users (name, email) VALUES (?, ?)'
        );
        
        // Exécuter plusieurs fois
        for (const user of users) {
            await statement.execute([user.name, user.email]);
        }
        
        await statement.close();
        
    } finally {
        connection.release();
    }
}

await batchInsertUsers([
    { name: 'Alice', email: 'alice@example.com' },
    { name: 'Bob', email: 'bob@example.com' }
]);
```

**Transactions avec prepared statements** :
```javascript
async function transferMoney(fromUserId, toUserId, amount) {
    const connection = await pool.getConnection();
    
    try {
        await connection.beginTransaction();
        
        // Prepared statement pour débit
        await connection.execute(
            'UPDATE accounts SET balance = balance - ? WHERE user_id = ?',
            [amount, fromUserId]
        );
        
        // Prepared statement pour crédit
        await connection.execute(
            'UPDATE accounts SET balance = balance + ? WHERE user_id = ?',
            [amount, toUserId]
        );
        
        await connection.commit();
        
    } catch (error) {
        await connection.rollback();
        throw error;
    } finally {
        connection.release();
    }
}
```

### 🔵 Go (database/sql)

**Configuration** :
```go
package main

import (
    "database/sql"
    "time"
    
    _ "github.com/go-sql-driver/mysql"
)

func initDB() (*sql.DB, error) {
    dsn := "myapp_user:myapp_password@tcp(localhost:3306)/myapp?charset=utf8mb4&parseTime=true"
    
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    
    // Pool configuration
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    
    // Test connexion
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}
```

**Utilisation basique** :
```go
type User struct {
    ID    int64
    Name  string
    Email string
    Age   int
}

func findUserByEmail(db *sql.DB, email string) (*User, error) {
    // Prepare automatique avec placeholders ?
    query := "SELECT id, name, email, age FROM users WHERE email = ?"
    
    var user User
    err := db.QueryRow(query, email).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Age,
    )
    
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}
```

**Prepared statement explicite** :
```go
func batchInsertUsers(db *sql.DB, users []User) error {
    // Préparer une fois
    stmt, err := db.Prepare("INSERT INTO users (name, email, age) VALUES (?, ?, ?)")
    if err != nil {
        return err
    }
    defer stmt.Close()
    
    // Exécuter plusieurs fois
    for _, user := range users {
        result, err := stmt.Exec(user.Name, user.Email, user.Age)
        if err != nil {
            return err
        }
        
        id, _ := result.LastInsertId()
        fmt.Printf("User %s created with ID: %d\n", user.Name, id)
    }
    
    return nil
}
```

**Transaction avec prepared statements** :
```go
func createUserWithPosts(db *sql.DB, user User, posts []string) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()  // Rollback si commit pas appelé
    
    // Insert user
    result, err := tx.Exec(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        user.Name, user.Email,
    )
    if err != nil {
        return err
    }
    
    userId, _ := result.LastInsertId()
    
    // Insert posts (prepared statement dans transaction)
    stmt, err := tx.Prepare("INSERT INTO posts (user_id, title) VALUES (?, ?)")
    if err != nil {
        return err
    }
    defer stmt.Close()
    
    for _, title := range posts {
        _, err := stmt.Exec(userId, title)
        if err != nil {
            return err
        }
    }
    
    return tx.Commit()
}
```

### 🔷 .NET (MySqlConnector)

**Configuration** :
```csharp
using MySqlConnector;

public class DatabaseManager
{
    private static string ConnectionString = 
        "Server=localhost;Database=myapp;User=myapp_user;Password=myapp_password;CharSet=utf8mb4";
    
    public static MySqlConnection GetConnection()
    {
        var connection = new MySqlConnection(ConnectionString);
        connection.Open();
        return connection;
    }
}
```

**Utilisation basique** :
```csharp
public class User
{
    public long Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}

public async Task<User> FindUserByEmailAsync(string email)
{
    using var connection = DatabaseManager.GetConnection();
    
    string query = "SELECT id, name, email, age FROM users WHERE email = @email";
    
    using var command = new MySqlCommand(query, connection);
    command.Parameters.AddWithValue("@email", email);
    
    using var reader = await command.ExecuteReaderAsync();
    
    if (await reader.ReadAsync())
    {
        return new User
        {
            Id = reader.GetInt64("id"),
            Name = reader.GetString("name"),
            Email = reader.GetString("email"),
            Age = reader.GetInt32("age")
        };
    }
    
    return null;
}
```

**Insert avec ID généré** :
```csharp
public async Task<long> CreateUserAsync(User user)
{
    using var connection = DatabaseManager.GetConnection();
    
    string query = "INSERT INTO users (name, email, age) VALUES (@name, @email, @age)";
    
    using var command = new MySqlCommand(query, connection);
    command.Parameters.AddWithValue("@name", user.Name);
    command.Parameters.AddWithValue("@email", user.Email);
    command.Parameters.AddWithValue("@age", user.Age);
    
    await command.ExecuteNonQueryAsync();
    
    return command.LastInsertedId;
}
```

**Batch insert avec transaction** :
```csharp
public async Task BatchInsertUsersAsync(List<User> users)
{
    using var connection = DatabaseManager.GetConnection();
    using var transaction = await connection.BeginTransactionAsync();
    
    try
    {
        string query = "INSERT INTO users (name, email, age) VALUES (@name, @email, @age)";
        
        using var command = new MySqlCommand(query, connection, transaction);
        
        // Préparer les paramètres
        command.Parameters.Add("@name", MySqlDbType.VarChar);
        command.Parameters.Add("@email", MySqlDbType.VarChar);
        command.Parameters.Add("@age", MySqlDbType.Int32);
        
        // Exécuter pour chaque user
        foreach (var user in users)
        {
            command.Parameters["@name"].Value = user.Name;
            command.Parameters["@email"].Value = user.Email;
            command.Parameters["@age"].Value = user.Age;
            
            await command.ExecuteNonQueryAsync();
        }
        
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

**Types explicites (recommandé)** :
```csharp
command.Parameters.Add("@name", MySqlDbType.VarChar).Value = user.Name;
command.Parameters.Add("@email", MySqlDbType.VarChar).Value = user.Email;
command.Parameters.Add("@age", MySqlDbType.Int32).Value = user.Age;
command.Parameters.Add("@is_active", MySqlDbType.Bool).Value = true;
command.Parameters.Add("@balance", MySqlDbType.Decimal).Value = 99.99m;
command.Parameters.Add("@created_at", MySqlDbType.DateTime).Value = DateTime.Now;
command.Parameters.Add("@data", MySqlDbType.JSON).Value = jsonString;
```

---

## Performance et optimisation

### ⚡ Benchmarks

**Scénario** : 1000 SELECT sur table users

| Méthode | Temps (ms) | vs Baseline | Mémoire serveur |
|---------|-----------|-------------|-----------------|
| **Requêtes dynamiques** | 1250 | Baseline | Faible |
| **Prepared statements (client)** | 1200 | -4% | Faible |
| **Prepared statements (serveur)** | 850 | -32% | Moyenne |
| **Prepared + cache serveur** | 650 | -48% | Haute |

**Conclusion** : Les prepared statements côté serveur sont **30-50% plus rapides** pour requêtes répétées.

### 🎯 Statement caching

**Configuration MariaDB** :
```ini
# my.cnf
[mysqld]
# Nombre max de prepared statements par connexion
max_prepared_stmt_count = 16382  # Défaut

# Cache des prepared statements
performance_schema = ON
```

**Monitoring** :
```sql
-- Nombre de prepared statements actifs
SHOW STATUS LIKE 'Prepared_stmt_count';

-- Statistiques
SELECT * FROM performance_schema.prepared_statements_instances;
```

**Cache côté client (Java)** :
```java
// Dans URL de connexion
jdbc:mariadb://localhost:3306/myapp
    ?useServerPrepStmts=true
    &cachePrepStmts=true
    &prepStmtCacheSize=250        -- Nombre de statements à cacher
    &prepStmtCacheSqlLimit=2048   -- Taille max SQL à cacher
```

### 🔄 Réutilisation optimale

**❌ INEFFICACE** : Créer et détruire à chaque fois
```python
for email in emails:
    cursor = connection.cursor(prepared=True)
    cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
    user = cursor.fetchone()
    cursor.close()  # Détruit le prepared statement !
```

**✅ EFFICACE** : Réutiliser le même statement
```python
cursor = connection.cursor(prepared=True)

for email in emails:
    cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
    user = cursor.fetchone()

cursor.close()  # Détruit à la fin seulement
```

**Gain de performance** : **3-5x plus rapide** !

### 📊 Mesurer l'impact

**Python - Benchmark** :
```python
import time
import mysql.connector

config = {'host': 'localhost', 'user': 'root', 'database': 'test'}

def benchmark_dynamic():
    """Requêtes dynamiques"""
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor()
    
    start = time.time()
    for i in range(1000):
        cursor.execute(f"SELECT * FROM users WHERE id = {i}")
        cursor.fetchall()
    
    elapsed = time.time() - start
    cursor.close()
    conn.close()
    
    return elapsed

def benchmark_prepared():
    """Prepared statements"""
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor(prepared=True)
    
    start = time.time()
    for i in range(1000):
        cursor.execute("SELECT * FROM users WHERE id = %s", (i,))
        cursor.fetchall()
    
    elapsed = time.time() - start
    cursor.close()
    conn.close()
    
    return elapsed

print(f"Dynamic: {benchmark_dynamic():.2f}s")
print(f"Prepared: {benchmark_prepared():.2f}s")
```

---

## Cas d'usage avancés

### 🔀 Transactions complexes

**Scénario** : Transfert d'argent avec validation

```python
def transfer_money(from_user_id, to_user_id, amount):
    """Transfert d'argent sécurisé avec prepared statements"""
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor(prepared=True)
    
    try:
        conn.start_transaction()
        
        # 1. Vérifier solde suffisant (FOR UPDATE = lock)
        cursor.execute(
            "SELECT balance FROM accounts WHERE user_id = %s FOR UPDATE",
            (from_user_id,)
        )
        
        balance = cursor.fetchone()[0]
        if balance < amount:
            raise ValueError("Insufficient balance")
        
        # 2. Débiter
        cursor.execute(
            "UPDATE accounts SET balance = balance - %s WHERE user_id = %s",
            (amount, from_user_id)
        )
        
        # 3. Créditer
        cursor.execute(
            "UPDATE accounts SET balance = balance + %s WHERE user_id = %s",
            (amount, to_user_id)
        )
        
        # 4. Logger transaction
        cursor.execute(
            "INSERT INTO transactions (from_user, to_user, amount) VALUES (%s, %s, %s)",
            (from_user_id, to_user_id, amount)
        )
        
        conn.commit()
        
    except Exception as e:
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()
```

### 📦 Batch operations

**Insert massif performant** :

```java
public void bulkInsertUsers(List<User> users) throws SQLException {
    String query = "INSERT INTO users (name, email, age) VALUES (?, ?, ?)";
    
    try (Connection conn = getConnection();
         PreparedStatement pstmt = conn.prepareStatement(query)) {
        
        conn.setAutoCommit(false);
        
        int batchSize = 1000;
        int count = 0;
        
        for (User user : users) {
            pstmt.setString(1, user.getName());
            pstmt.setString(2, user.getEmail());
            pstmt.setInt(3, user.getAge());
            pstmt.addBatch();
            
            if (++count % batchSize == 0) {
                pstmt.executeBatch();  // Exécuter par lots de 1000
                pstmt.clearBatch();
            }
        }
        
        pstmt.executeBatch();  // Reste
        conn.commit();
    }
}
```

**Performance** : **10-50x plus rapide** que des INSERT individuels !

### 🔍 Dynamic search avec prepared statements

**Recherche avec filtres optionnels** :

```typescript
interface SearchFilters {
    name?: string;
    email?: string;
    minAge?: number;
    maxAge?: number;
}

async function searchUsers(filters: SearchFilters) {
    const conditions: string[] = [];
    const params: any[] = [];
    
    // Construire WHERE dynamiquement
    if (filters.name) {
        conditions.push('name LIKE ?');
        params.push(`%${filters.name}%`);
    }
    
    if (filters.email) {
        conditions.push('email = ?');
        params.push(filters.email);
    }
    
    if (filters.minAge !== undefined) {
        conditions.push('age >= ?');
        params.push(filters.minAge);
    }
    
    if (filters.maxAge !== undefined) {
        conditions.push('age <= ?');
        params.push(filters.maxAge);
    }
    
    // Construire requête
    let query = 'SELECT * FROM users';
    
    if (conditions.length > 0) {
        query += ' WHERE ' + conditions.join(' AND ');
    }
    
    // Exécuter avec prepared statement
    const [rows] = await pool.execute(query, params);
    return rows;
}

// Utilisation
const users = await searchUsers({
    name: 'Alice',
    minAge: 18,
    maxAge: 65
});
```

---

## Limitations et solutions

### ⚠️ Limitations

#### **1. Pas de placeholders pour identifiants**

**❌ NE FONCTIONNE PAS** :
```python
table_name = 'users'
cursor.execute("SELECT * FROM %s", (table_name,))
# Résultat : SELECT * FROM 'users' (erreur de syntaxe)
```

**✅ SOLUTION : Whitelist**
```python
ALLOWED_TABLES = ['users', 'posts', 'comments']

table_name = request.args.get('table')

if table_name not in ALLOWED_TABLES:
    raise ValueError("Invalid table name")

# Sûr car validé
query = f"SELECT * FROM {table_name}"
cursor.execute(query)
```

#### **2. Limite de prepared statements**

MariaDB limite le nombre de prepared statements par connexion.

**Vérifier la limite** :
```sql
SHOW VARIABLES LIKE 'max_prepared_stmt_count';
-- Défaut : 16382
```

**Solution** : Fermer les statements non utilisés
```python
cursor.execute(query, params)
cursor.close()  # Libère le prepared statement
```

#### **3. Performance pour requêtes uniques**

Pour une requête exécutée **une seule fois**, les prepared statements ont un **léger overhead**.

**Benchmark** :
- Requête dynamique (1x) : 10ms
- Prepared statement (1x) : 12ms (PREPARE + EXECUTE)

**Recommandation** : Utiliser quand même pour la **sécurité** !

### 🔧 Debugging

**Voir les prepared statements actifs** :
```sql
-- MariaDB
SELECT * FROM information_schema.PROCESSLIST 
WHERE COMMAND = 'Prepare';

-- Performance Schema
SELECT * FROM performance_schema.prepared_statements_instances;
```

**Logger les requêtes** :

**PHP** :
```php
$pdo->setAttribute(PDO::ATTR_STATEMENT_CLASS, ['PDOStatement']);

// Voir SQL généré
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => 'alice@example.com']);

var_dump($stmt->queryString);
var_dump($stmt->debugDumpParams());
```

**Python** :
```python
import logging

# Activer logging MySQL
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('mysql.connector')
logger.setLevel(logging.DEBUG)

# Voir les requêtes préparées
cursor.execute("SELECT * FROM users WHERE email = %s", ('alice@example.com',))
```

---

## Quand utiliser ou non ?

### ✅ Utiliser TOUJOURS pour

- 🔒 **Requêtes avec input utilisateur** (sécurité)
- 🔄 **Requêtes répétées** dans une boucle
- 📦 **Batch operations** (INSERT, UPDATE massifs)
- 💰 **Transactions critiques** (paiements, etc.)
- 🎯 **Type safety** importante

### 🤔 Considérer les alternatives pour

- 📊 **Requêtes analytiques complexes** (EXPLAIN, optimisation manuelle)
- 🔀 **DDL statements** (CREATE, ALTER - pas supportés)
- 🗂️ **Requêtes avec structure dynamique** (nombre de colonnes variable)

### ❌ Ne PAS utiliser pour

- 🛠️ **Commandes administratives** (SHOW, DESCRIBE)
- 🔧 **Utility statements** (SET, USE)

**Règle simple** : **Par défaut, toujours utiliser des prepared statements** sauf raison technique spécifique.

---

## Bonnes pratiques

### ✅ Checklist

- [ ] **Toujours** utiliser prepared statements pour requêtes avec input utilisateur
- [ ] **Réutiliser** les statements dans les boucles
- [ ] **Fermer** les cursors/statements après usage
- [ ] **Valider** les types de données avant binding
- [ ] **Utiliser** les types explicites (PDO::PARAM_INT, etc.)
- [ ] **Activer** server-side prepared statements quand possible
- [ ] **Configurer** le statement caching (Java)
- [ ] **Monitorer** le nombre de prepared statements actifs
- [ ] **Tester** les performances (benchmarks)
- [ ] **Logger** les erreurs de préparation

### 🎯 Patterns recommandés

**Repository pattern** :
```python
class UserRepository:
    def __init__(self, connection):
        self.connection = connection
        # Préparer les statements fréquents
        self._find_by_id_stmt = connection.cursor(prepared=True)
        self._find_by_email_stmt = connection.cursor(prepared=True)
    
    def find_by_id(self, user_id):
        self._find_by_id_stmt.execute(
            "SELECT * FROM users WHERE id = %s",
            (user_id,)
        )
        return self._find_by_id_stmt.fetchone()
    
    def find_by_email(self, email):
        self._find_by_email_stmt.execute(
            "SELECT * FROM users WHERE email = %s",
            (email,)
        )
        return self._find_by_email_stmt.fetchone()
    
    def close(self):
        self._find_by_id_stmt.close()
        self._find_by_email_stmt.close()
```

---

## ✅ Points clés à retenir

- 🛡️ **Sécurité** : Prepared statements protègent contre injections SQL (100%)
- ⚡ **Performance** : 30-50% plus rapide pour requêtes répétées
- 🔄 **Réutilisation** : Préparer une fois, exécuter plusieurs fois
- 📦 **Séparation** : Code SQL vs données complètement séparés
- 🎯 **Type safety** : Validation automatique des types
- 🖥️ **Server-side** : Préférer si possible (performance max)
- 🔄 **Statement caching** : Configurer pour performance optimale
- 📊 **Benchmarks** : Mesurer l'impact réel sur votre application
- ⚠️ **Limitations** : Pas pour identifiants, DDL, ou commandes admin
- ✅ **Par défaut** : Toujours utiliser sauf raison technique

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [MariaDB - Prepared Statements](https://mariadb.com/kb/en/prepared-statements/)
- 📖 [MySQL Connector/Python - Prepared Statements](https://dev.mysql.com/doc/connector-python/en/connector-python-api-mysqlcursor-prepare.html)
- 📖 [JDBC - PreparedStatement](https://docs.oracle.com/javase/8/docs/api/java/sql/PreparedStatement.html)
- 📖 [PHP PDO - Prepared Statements](https://www.php.net/manual/en/pdo.prepared-statements.php)

### **Guides et best practices**
- 📝 [OWASP - Query Parameterization](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html)
- 📝 [Use The Index, Luke - Bind Parameters](https://use-the-index-luke.com/sql/where-clause/bind-parameters)

### **Performance**
- 📊 [MySQL Performance Blog - Prepared Statements](https://www.percona.com/blog/)
- 📊 [MariaDB Performance - Statement Caching](https://mariadb.com/kb/en/performance-schema/)

---

## ➡️ Prochaines sections suggérées

Vous avez maintenant une compréhension complète des prepared statements ! Pour aller plus loin :

- **Chapitre 18** - Sécurité : Chiffrement, SSL/TLS, audit
- **Chapitre 19** - Performance : Optimisation de requêtes, indexation avancée
- **Chapitre 20** - Monitoring : Surveillance, métriques, alertes

---

**MariaDB** : Version 11.8 LTS

💡 **Rappel** : Les prepared statements sont la **meilleure pratique absolue** pour toutes les requêtes avec input utilisateur. Aucune excuse pour ne pas les utiliser !

⏭️ [Fonctionnalités Avancées](/18-fonctionnalites-avancees/README.md)

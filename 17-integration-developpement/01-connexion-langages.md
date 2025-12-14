🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.1 Connexion depuis différents langages

> **Niveau** : Intermédiaire  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Maîtrise d'au moins un langage de programmation
> - Compréhension des bases SQL (Chapitres 2-4)
> - Notions de programmation réseau et protocoles client-serveur

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture client-serveur de MariaDB et le protocole MySQL
- **Choisir** la bibliothèque appropriée pour chaque langage de programmation
- **Établir** des connexions robustes et sécurisées à MariaDB
- **Configurer** les paramètres de connexion optimaux (charset, timeout, SSL)
- **Gérer** correctement le cycle de vie des connexions (ouverture, utilisation, fermeture)
- **Identifier** les différences entre connecteurs et leurs cas d'usage

---

## Introduction

La connexion à MariaDB depuis vos applications est la **première étape** de toute intégration. Bien que le principe reste identique quel que soit le langage (établir une connexion TCP/IP vers le serveur MariaDB), chaque écosystème propose ses propres bibliothèques, conventions et bonnes pratiques.

### 🔌 Pourquoi plusieurs bibliothèques par langage ?

Vous remarquerez que la plupart des langages offrent **plusieurs options** pour se connecter à MariaDB :

| Langage | Options disponibles | Choix recommandé |
|---------|---------------------|------------------|
| **PHP** | mysqli (procédural/OO), PDO | PDO (abstraction) ou mysqli (performances) |
| **Python** | mysql-connector-python, PyMySQL, mysqlclient, aiomysql | mysql-connector-python (officiel) ou PyMySQL (pure Python) |
| **Java** | JDBC générique, MariaDB Connector/J | MariaDB Connector/J (optimisé) |
| **Node.js** | mysql, mysql2, mariadb (officiel) | mariadb (officiel) ou mysql2 (async/await) |
| **Go** | go-sql-driver/mysql | go-sql-driver/mysql (standard de facto) |
| **.NET** | MySqlConnector, MariaDB.Data, MySQL Connector/NET | MySqlConnector (async/await natif) |

💡 **Pourquoi cette diversité ?** Chaque bibliothèque répond à des besoins spécifiques :
- **Officielles** : Support garanti, compatibilité
- **Communautaires** : Innovations, fonctionnalités avancées
- **Performances** : Optimisations spécifiques (async, préparation côté serveur)
- **Simplicité** : API minimaliste vs complète

---

## Architecture client-serveur MariaDB

### 🏗️ Comment fonctionne une connexion ?

```
┌─────────────────┐         ┌──────────────────┐
│   Application   │         │   MariaDB Server │
│                 │         │                  │
│  ┌───────────┐  │         │  ┌────────────┐  │
│  │ Connecteur│  │◄────────┤  │ Listener   │  │
│  │ MariaDB   │  │  TCP/IP │  │ port 3306  │  │
│  └───────────┘  │  Socket │  └────────────┘  │
│        │        │         │        │         │
│  ┌─────▼─────┐  │         │  ┌─────▼──────┐  │
│  │  Requêtes │  │────────►│  │ SQL Engine │  │
│  │  SQL      │  │         │  │            │  │
│  └───────────┘  │         │  └─────┬──────┘  │
│        ▲        │         │        │         │
│  ┌─────┴─────┐  │         │  ┌─────▼──────┐  │
│  │ Résultats │  │◄────────│  │  Storage   │  │
│  └───────────┘  │         │  │  Engines   │  │
└─────────────────┘         └──────────────────┘
```

### 📡 Le protocole MySQL/MariaDB

Tous les connecteurs utilisent le **protocole MySQL** (MariaDB le maintient avec extensions) :

1. **Handshake** : Échange de capacités client/serveur
2. **Authentification** : Vérification des credentials (plugin-based)
3. **Requête** : Envoi de commandes SQL
4. **Résultat** : Réception des données (rowset)
5. **Terminaison** : Fermeture de connexion

🆕 **MariaDB 11.8** : Support amélioré pour :
- Authentification ed25519 et PARSEC
- TLS 1.3 par défaut
- Compression zstd
- Pipelining de requêtes

---

## Critères de choix d'un connecteur

### ✅ Checklist pour évaluer une bibliothèque

| Critère | Importance | Questions à se poser |
|---------|------------|---------------------|
| **Support officiel** | ⭐⭐⭐⭐ | Est-ce maintenu par MariaDB Foundation ou Oracle ? |
| **Communauté active** | ⭐⭐⭐⭐⭐ | Activité GitHub, issues résolues, releases régulières ? |
| **Performance** | ⭐⭐⭐⭐⭐ | Support async ? Prepared statements côté serveur ? |
| **Sécurité** | ⭐⭐⭐⭐⭐ | Parameterized queries ? SSL/TLS ? Validation entrées ? |
| **Fonctionnalités** | ⭐⭐⭐⭐ | Connection pooling ? Failover ? Compression ? |
| **Documentation** | ⭐⭐⭐⭐ | Exemples clairs ? API complète ? |
| **Compatibilité** | ⭐⭐⭐⭐ | Version MariaDB supportée ? Migration facile ? |
| **Licence** | ⭐⭐⭐ | Compatible avec votre projet ? |

### 🎯 Cas d'usage typiques

**API REST/Web moderne** :
- Privilégier les connecteurs **asynchrones** (async/await)
- Support du **connection pooling** natif
- Exemples : mysql2 (Node.js), MySqlConnector (.NET), aiomysql (Python)

**Application batch/ETL** :
- Optimisation des **performances brutes**
- Support du **bulk insert** et **LOAD DATA**
- Exemples : mysqlclient (Python), MariaDB Connector/J (Java)

**Microservices** :
- **Légèreté** (peu de dépendances)
- **Monitoring** intégré (métriques, tracing)
- Support **cloud-native** (variables d'env, health checks)
- Exemples : go-sql-driver/mysql (Go), mariadb connector (Node.js)

**Applications legacy** :
- **Compatibilité** avec code existant
- **Stabilité** (peu de breaking changes)
- Exemples : mysqli (PHP), mysql-connector-python (Python)

---

## Vue d'ensemble des connecteurs par langage

### 🐘 **PHP : mysqli et PDO**

PHP offre deux approches principales pour se connecter à MariaDB :

#### **mysqli (MySQL Improved)**
- ✅ **Avantages** : Performant, accès aux fonctionnalités MariaDB spécifiques, API procédurale ET orientée objet
- ❌ **Inconvénients** : Lié à MySQL/MariaDB uniquement, migration difficile vers un autre SGBD
- 🎯 **Cas d'usage** : Applications nécessitant performances maximales et fonctionnalités MySQL/MariaDB avancées

```php
// Exemple rapide (détails en 17.1.1)
$mysqli = new mysqli("localhost", "user", "password", "database");
if ($mysqli->connect_error) {
    die("Connexion échouée: " . $mysqli->connect_error);
}
```

#### **PDO (PHP Data Objects)**
- ✅ **Avantages** : Abstraction (support multi-SGBD), API unifiée, excellente sécurité (prepared statements)
- ❌ **Inconvénients** : Léger overhead de performance, moins de fonctionnalités spécifiques MySQL/MariaDB
- 🎯 **Cas d'usage** : Applications devant supporter plusieurs SGBD, accent sur la portabilité

```php
// Exemple rapide (détails en 17.1.1)
$pdo = new PDO("mysql:host=localhost;dbname=database;charset=utf8mb4", "user", "password");
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
```

💡 **Recommandation** : PDO pour nouveaux projets (abstraction), mysqli si performances critiques ou fonctionnalités MariaDB spécifiques.

---

### 🐍 **Python : mysql-connector, PyMySQL, SQLAlchemy**

L'écosystème Python est particulièrement riche :

#### **mysql-connector-python**
- ✅ **Officiel** Oracle/MySQL, **pure Python** (pas de dépendance C)
- ✅ Support complet du protocole, excellente compatibilité
- ❌ Performances moyennes (pure Python)
- 🎯 **Cas d'usage** : Applications nécessitant portabilité maximale, installation simplifiée

#### **PyMySQL**
- ✅ **Pure Python**, API compatible DB-API 2.0
- ✅ Excellent pour **testing** (facile à installer)
- ❌ Moins performant que mysqlclient
- 🎯 **Cas d'usage** : Développement, prototypage, environnements contraints

#### **mysqlclient**
- ✅ **Performances maximales** (binding C vers libmariadb)
- ✅ Compatible MySQL-Python (fork moderne)
- ❌ Nécessite compilation, dépendances système
- 🎯 **Cas d'usage** : Production haute performance, ETL, data processing

#### **aiomysql**
- ✅ **Asynchrone** (asyncio), idéal pour applications async
- ✅ Compatible PyMySQL (drop-in replacement async)
- ❌ Communauté plus petite
- 🎯 **Cas d'usage** : Applications async/await, APIs FastAPI/aiohttp

```python
# Exemple rapide (détails en 17.1.2)
import mysql.connector

conn = mysql.connector.connect(
    host="localhost",
    user="user",
    password="password",
    database="database",
    charset="utf8mb4"
)
```

💡 **Recommandation** : mysql-connector-python (standard), mysqlclient (performance), aiomysql (async).

---

### ☕ **Java : JDBC et MariaDB Connector/J**

Java utilise l'abstraction **JDBC** (Java Database Connectivity) :

#### **MariaDB Connector/J**
- ✅ **Officiel** MariaDB Foundation
- ✅ Optimisations spécifiques MariaDB (pipelining, bulk insert)
- ✅ Support Galera Cluster (failover automatique)
- ✅ Excellent support TLS/SSL
- 🎯 **Cas d'usage** : Applications Java/Spring Boot avec MariaDB

#### **MySQL Connector/J (Oracle)**
- ✅ **Officiel** Oracle, très mature
- ✅ Compatible MariaDB (avec limitations)
- ❌ Moins d'optimisations MariaDB spécifiques
- 🎯 **Cas d'usage** : Migration MySQL → MariaDB, compatibilité maximale

```java
// Exemple rapide (détails en 17.1.3)
import org.mariadb.jdbc.Driver;
import java.sql.Connection;
import java.sql.DriverManager;

String url = "jdbc:mariadb://localhost:3306/database?user=user&password=password";
Connection conn = DriverManager.getConnection(url);
```

💡 **Recommandation** : MariaDB Connector/J pour nouveaux projets MariaDB, MySQL Connector/J si migration.

---

### 🟢 **Node.js : mysql2 et mariadb**

Node.js excelle dans les opérations I/O asynchrones :

#### **mariadb (officiel)**
- ✅ **Officiel** MariaDB Foundation
- ✅ **Promises natives**, async/await
- ✅ Connection pooling intégré
- ✅ Pipelining de requêtes, batch insert
- 🎯 **Cas d'usage** : Applications Node.js modernes, APIs REST

#### **mysql2**
- ✅ **Communauté très active**, très populaire
- ✅ Prepared statements côté serveur
- ✅ Support streams, performances excellentes
- ✅ Compatible mysql (drop-in replacement)
- 🎯 **Cas d'usage** : Applications nécessitant performances maximales

#### **mysql (legacy)**
- ⚠️ **Déprécié**, callbacks (pas async/await natif)
- ❌ Moins de fonctionnalités modernes
- 🎯 **Cas d'usage** : Maintenance code legacy uniquement

```javascript
// Exemple rapide (détails en 17.1.4)
const mariadb = require('mariadb');

const pool = mariadb.createPool({
    host: 'localhost',
    user: 'user',
    password: 'password',
    database: 'database',
    connectionLimit: 5
});

async function query() {
    let conn;
    try {
        conn = await pool.getConnection();
        const rows = await conn.query("SELECT * FROM users");
        return rows;
    } finally {
        if (conn) conn.release();
    }
}
```

💡 **Recommandation** : mariadb (officiel, moderne) ou mysql2 (performances, communauté).

---

### 🔵 **Go : go-sql-driver/mysql**

Go utilise l'interface standard `database/sql` :

#### **go-sql-driver/mysql**
- ✅ **Standard de facto** de l'écosystème Go
- ✅ Excellentes performances, production-ready
- ✅ Support TLS, connection pooling natif
- ✅ Compatible `database/sql` (interface standard Go)
- 🎯 **Cas d'usage** : Tous types d'applications Go

```go
// Exemple rapide (détails en 17.1.5)
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
)

db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/database?charset=utf8mb4&parseTime=true")
if err != nil {
    panic(err)
}
defer db.Close()
```

💡 **Recommandation** : go-sql-driver/mysql (unique choix viable, excellent).

---

### 🔷 **.NET : MySqlConnector, MariaDB.Data, ADO.NET**

L'écosystème .NET offre plusieurs options :

#### **MySqlConnector** (Recommandé)
- ✅ **Performances excellentes**, async/await natif
- ✅ **Communauté très active**, open source
- ✅ Support .NET Core, .NET 5+, .NET Framework
- ✅ Implémentation moderne du protocole MySQL
- 🎯 **Cas d'usage** : Nouveaux projets .NET, applications modernes

#### **MariaDB.Data**
- ✅ **Officiel** MariaDB Foundation
- ✅ Compatible ADO.NET
- ⚠️ Moins mature que MySqlConnector
- 🎯 **Cas d'usage** : Fonctionnalités MariaDB spécifiques

#### **MySQL Connector/NET (Oracle)**
- ⚠️ **Performances moyennes** (pas d'async natif jusqu'à récemment)
- ✅ Officiel Oracle, très stable
- 🎯 **Cas d'usage** : Applications legacy, compatibilité Oracle

#### **ADO.NET**
- ✅ **Abstraction standard** .NET
- ✅ Interface unifiée pour tous SGBD
- 🎯 **Cas d'usage** : Code abstrait, multi-SGBD

```csharp
// Exemple rapide (détails en 17.1.6)
using MySqlConnector;

var connectionString = "Server=localhost;User ID=user;Password=password;Database=database";
using var connection = new MySqlConnection(connectionString);
await connection.OpenAsync();

using var command = new MySqlCommand("SELECT * FROM users", connection);
using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    Console.WriteLine(reader.GetString(0));
}
```

💡 **Recommandation** : MySqlConnector pour nouveaux projets .NET (performances, async).

---

## Paramètres de connexion essentiels

### 🔧 Configuration universelle

Quel que soit le langage, certains paramètres sont **critiques** :

#### **1. Charset/Encoding**

```
✅ TOUJOURS UTF8MB4 (support complet Unicode, emojis)
```

| Langage | Configuration |
|---------|---------------|
| PHP | `charset=utf8mb4` (PDO), `set_charset('utf8mb4')` (mysqli) |
| Python | `charset='utf8mb4'` |
| Java | `characterEncoding=UTF-8` |
| Node.js | `charset: 'utf8mb4'` |
| Go | `charset=utf8mb4` dans DSN |
| .NET | `CharSet=utf8mb4` |

⚠️ **Piège** : `utf8` (alias `utf8mb3`) = 3 bytes max, ne supporte PAS les emojis ! Toujours `utf8mb4`.

🆕 **MariaDB 11.8** : utf8mb4 est le charset par défaut, plus besoin de le spécifier explicitement.

#### **2. Timeouts**

```
Connection Timeout : 10-30s (établissement connexion)
Read Timeout       : 30-60s (exécution requête)
Write Timeout      : 30-60s (envoi données)
```

**Exemple PHP** :
```php
$pdo = new PDO("mysql:host=localhost;dbname=db", "user", "pass", [
    PDO::ATTR_TIMEOUT => 10,  // Connection timeout
]);
```

**Exemple Python** :
```python
conn = mysql.connector.connect(
    connect_timeout=10,  # Connexion
    read_timeout=30,     # Lecture résultats
    write_timeout=30     # Écriture données
)
```

#### **3. SSL/TLS**

🔒 **Toujours** activer SSL/TLS en production :

```
Mode minimal  : PREFERRED (tente SSL, fallback non-SSL)
Mode sécurisé : REQUIRED (SSL obligatoire)
Mode strict   : VERIFY_CA ou VERIFY_IDENTITY (validation certificat)
```

**Exemple Java** :
```java
String url = "jdbc:mariadb://localhost:3306/db"
    + "?useSSL=true"
    + "&requireSSL=true"
    + "&trustServerCertificate=false";
```

**Exemple .NET** :
```csharp
var connString = "Server=localhost;Database=db;"
    + "SslMode=Required;"
    + "SslCa=/path/to/ca.pem;";
```

🆕 **MariaDB 11.8** : TLS activé par défaut côté serveur, configuration simplifiée.

#### **4. Timezone**

```
✅ Configurer la timezone explicitement
```

**Option 1 : Connection string**
```
serverTimezone=UTC
```

**Option 2 : Après connexion**
```sql
SET time_zone = '+00:00';  -- UTC
```

💡 **Recommandation** : Stocker en UTC, convertir dans l'application.

---

## Gestion du cycle de vie des connexions

### ♻️ Pattern universel

```
1. OUVRIR    → Établir connexion (avec retry si échec)
2. UTILISER  → Exécuter requêtes (dans try/catch)
3. FERMER    → Libérer ressources (dans finally)
```

### ✅ Bonne pratique : Try-Finally pattern

**PHP** :
```php
$conn = null;
try {
    $conn = new PDO("mysql:host=localhost;dbname=db", "user", "pass");
    // Utilisation
} catch (PDOException $e) {
    error_log($e->getMessage());
    throw $e;
} finally {
    $conn = null;  // Ferme la connexion
}
```

**Python** :
```python
conn = None
try:
    conn = mysql.connector.connect(host="localhost", user="user")
    # Utilisation
except mysql.connector.Error as err:
    logging.error(f"Database error: {err}")
    raise
finally:
    if conn and conn.is_connected():
        conn.close()
```

**Java** :
```java
// Try-with-resources (Java 7+)
try (Connection conn = DriverManager.getConnection(url)) {
    // Utilisation - fermeture automatique
} catch (SQLException e) {
    logger.error("Database error", e);
    throw e;
}
```

**Node.js** :
```javascript
let conn;
try {
    conn = await pool.getConnection();
    // Utilisation
} catch (err) {
    console.error('Database error:', err);
    throw err;
} finally {
    if (conn) conn.release();  // Retour au pool
}
```

**Go** :
```go
db, err := sql.Open("mysql", dsn)
if err != nil {
    return err
}
defer db.Close()  // Fermeture automatique

// Utilisation
```

**.NET** :
```csharp
using (var connection = new MySqlConnection(connectionString))
{
    await connection.OpenAsync();
    // Utilisation - fermeture automatique (Dispose)
}
```

### ⚠️ Anti-patterns à éviter

❌ **Connexions non fermées** :
```python
# MAUVAIS : Fuite de connexions !
def get_user(id):
    conn = mysql.connector.connect(...)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (id,))
    return cursor.fetchone()
    # conn jamais fermée !
```

❌ **Ouverture/fermeture répétées** :
```php
// MAUVAIS : Inefficace !
foreach ($users as $user) {
    $conn = new PDO(...);  // Nouvelle connexion à chaque itération !
    // ...
    $conn = null;
}
```

✅ **Solution** : Utiliser le connection pooling (voir Section 17.2).

---

## Tests de connectivité

### 🔍 Vérifier qu'une connexion fonctionne

**PHP** :
```php
try {
    $pdo = new PDO("mysql:host=localhost;dbname=db", "user", "pass");
    $stmt = $pdo->query("SELECT 1");
    echo "Connexion OK\n";
} catch (PDOException $e) {
    echo "Erreur: " . $e->getMessage() . "\n";
}
```

**Python** :
```python
import mysql.connector

try:
    conn = mysql.connector.connect(
        host="localhost",
        user="user",
        password="password"
    )
    if conn.is_connected():
        cursor = conn.cursor()
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()
        print(f"Connecté à MariaDB version: {version[0]}")
except mysql.connector.Error as err:
    print(f"Erreur: {err}")
finally:
    if conn and conn.is_connected():
        cursor.close()
        conn.close()
```

**Java** :
```java
String url = "jdbc:mariadb://localhost:3306/db";
try (Connection conn = DriverManager.getConnection(url, "user", "pass")) {
    System.out.println("Connexion OK");
    DatabaseMetaData meta = conn.getMetaData();
    System.out.println("Driver: " + meta.getDriverName());
    System.out.println("Version: " + meta.getDriverVersion());
} catch (SQLException e) {
    System.err.println("Erreur: " + e.getMessage());
}
```

**Node.js** :
```javascript
const mariadb = require('mariadb');

async function testConnection() {
    let conn;
    try {
        conn = await mariadb.createConnection({
            host: 'localhost',
            user: 'user',
            password: 'password'
        });
        const result = await conn.query("SELECT VERSION()");
        console.log("Connecté à MariaDB version:", result[0]['VERSION()']);
    } catch (err) {
        console.error("Erreur:", err.message);
    } finally {
        if (conn) conn.end();
    }
}

testConnection();
```

**Go** :
```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    dsn := "user:password@tcp(localhost:3306)/db"
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // Test ping
    err = db.Ping()
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("Connexion OK")
    
    // Version
    var version string
    err = db.QueryRow("SELECT VERSION()").Scan(&version)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("MariaDB version: %s\n", version)
}
```

**.NET** :
```csharp
using MySqlConnector;

string connectionString = "Server=localhost;User ID=user;Password=password;Database=db";
try
{
    using var connection = new MySqlConnection(connectionString);
    await connection.OpenAsync();
    Console.WriteLine("Connexion OK");
    
    using var command = new MySqlCommand("SELECT VERSION()", connection);
    var version = await command.ExecuteScalarAsync();
    Console.WriteLine($"MariaDB version: {version}");
}
catch (MySqlException ex)
{
    Console.WriteLine($"Erreur: {ex.Message}");
}
```

---

## Comparaison des connecteurs

### 📊 Tableau récapitulatif

| Langage | Connecteur recommandé | Async | Pool | Préparation | Perf | Popularité |
|---------|----------------------|-------|------|-------------|------|------------|
| **PHP** | PDO | ❌ | Via pool séparé | ✅ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **PHP** | mysqli | ❌ | Via pool séparé | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Python** | mysql-connector | ❌ | ✅ | ✅ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Python** | mysqlclient | ❌ | Via pool séparé | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Python** | aiomysql | ✅ | ✅ | ✅ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Java** | MariaDB Connector/J | ❌ | ✅ (HikariCP) | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Node.js** | mariadb | ✅ | ✅ | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Node.js** | mysql2 | ✅ | ✅ | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Go** | go-sql-driver | ✅ | ✅ (natif) | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **.NET** | MySqlConnector | ✅ | ✅ (natif) | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

💡 **Légende** :
- **Async** : Support natif async/await
- **Pool** : Connection pooling intégré
- **Préparation** : Prepared statements côté serveur
- **Perf** : Performances brutes
- **Popularité** : Adoption communautaire

---

## 💡 Bonnes pratiques transversales

### ✅ À faire systématiquement

1. **Charset UTF8MB4** : Support complet Unicode
   ```
   charset=utf8mb4 (connection string)
   ```

2. **Timeouts appropriés** : Éviter les blocages infinis
   ```
   connect_timeout=10
   read_timeout=30
   ```

3. **SSL/TLS en production** : Chiffrer les communications
   ```
   useSSL=true
   sslMode=Required
   ```

4. **Fermeture des ressources** : Try-finally, using, defer
   ```
   try-finally (Python, PHP, Java)
   using (C#)
   defer (Go)
   ```

5. **Gestion d'erreurs** : Logger, ne pas exposer détails techniques
   ```
   catch/except avec logging structuré
   ```

6. **Configuration externalisée** : Variables d'environnement, secrets
   ```
   DB_HOST, DB_USER, DB_PASSWORD (env vars)
   ```

### ❌ À éviter absolument

1. ❌ **Credentials hardcodés** dans le code
2. ❌ **Charset latin1/utf8** (utiliser utf8mb4)
3. ❌ **Connexions non fermées** (fuites mémoire)
4. ❌ **Pas de gestion d'erreurs** (exceptions silencieuses)
5. ❌ **Concaténation de SQL** (risque injection)
6. ❌ **Connexions sans timeout** (blocages infinis)
7. ❌ **SSL désactivé en production** (données en clair)

---

## ⚠️ Pièges courants

### 🐛 Problème 1 : Charset incorrect

**Symptôme** : Caractères spéciaux affichés incorrectement (�, Ã©, etc.)

**Cause** :
```python
# MAUVAIS : charset par défaut (souvent latin1)
conn = mysql.connector.connect(host="localhost", user="user")
```

**Solution** :
```python
# BON : UTF8MB4 explicite
conn = mysql.connector.connect(
    host="localhost",
    user="user",
    charset="utf8mb4"
)
```

### 🐛 Problème 2 : Timezone incohérente

**Symptôme** : Dates décalées entre application et base

**Cause** :
```javascript
// Timezone serveur différente de l'application
const conn = await mariadb.createConnection({host: 'localhost'});
```

**Solution** :
```javascript
const conn = await mariadb.createConnection({
    host: 'localhost',
    timezone: 'UTC'  // Forcer UTC
});
```

### 🐛 Problème 3 : Connexions non libérées

**Symptôme** : "Too many connections" après un certain temps

**Cause** :
```php
function getUsers() {
    $pdo = new PDO(...);
    return $pdo->query("SELECT * FROM users")->fetchAll();
    // PDO jamais fermé explicitement
}
```

**Solution** :
```php
function getUsers() {
    $pdo = new PDO(...);
    try {
        return $pdo->query("SELECT * FROM users")->fetchAll();
    } finally {
        $pdo = null;  // Fermeture explicite
    }
}
```

Ou mieux : utiliser un **connection pool** (Section 17.2).

---

## 🔍 Debugging des connexions

### 📝 Logs de connexion côté serveur

Activer le general log temporairement :

```sql
-- Activer
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/general.log';

-- Désactiver (après debug)
SET GLOBAL general_log = 'OFF';
```

⚠️ **Attention** : Performance impact significatif, uniquement pour debug !

### 🔎 Vérifier les connexions actives

```sql
-- Liste des connexions
SHOW PROCESSLIST;

-- Plus détaillé
SELECT * FROM information_schema.PROCESSLIST;

-- Statistiques par utilisateur
SELECT USER, COUNT(*) as connections
FROM information_schema.PROCESSLIST
GROUP BY USER;
```

### 📊 Monitoring des erreurs de connexion

```sql
-- Tentatives de connexion refusées
SHOW GLOBAL STATUS LIKE 'Connection_errors%';

-- Connexions échouées (mauvais password)
SHOW GLOBAL STATUS LIKE 'Access_denied_errors';

-- Connexions réussies
SHOW GLOBAL STATUS LIKE 'Connections';
```

---

## ✅ Points clés à retenir

- 🔌 **Chaque langage** a plusieurs bibliothèques, choisir selon le contexte (perf, async, abstraction)
- 🎯 **Connecteurs recommandés** : PDO (PHP), mysql-connector (Python), MariaDB Connector/J (Java), mariadb/mysql2 (Node.js), go-sql-driver (Go), MySqlConnector (.NET)
- 🌐 **Charset UTF8MB4** : TOUJOURS pour support Unicode complet
- 🔒 **SSL/TLS** : Obligatoire en production
- ⏱️ **Timeouts** : Configurer pour éviter blocages
- ♻️ **Fermeture** : Try-finally, using, defer selon le langage
- 📝 **Configuration externalisée** : Variables d'environnement, jamais hardcodé
- 🆕 **MariaDB 11.8** : UTF8MB4 et TLS par défaut, configuration simplifiée

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [MariaDB Connectors](https://mariadb.com/kb/en/connectors/)
- 📖 [MySQL Protocol](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html)
- 📖 [Character Sets and Collations](https://mariadb.com/kb/en/character-sets/)

### **Connecteurs**
- 🔗 [MariaDB Connector/J (Java)](https://mariadb.com/kb/en/about-mariadb-connector-j/)
- 🔗 [MariaDB Connector/Python](https://mariadb.com/kb/en/mariadb-connector-python/)
- 🔗 [MariaDB Connector/Node.js](https://github.com/mariadb-corporation/mariadb-connector-nodejs)
- 🔗 [MySqlConnector (.NET)](https://mysqlconnector.net/)
- 🔗 [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)

### **Guides de sécurité**
- 🔒 [Secure Connections (SSL/TLS)](https://mariadb.com/kb/en/securing-connections-for-client-and-server/)
- 🔒 [Authentication Plugins](https://mariadb.com/kb/en/authentication-plugins/)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent l'implémentation pratique pour chaque langage :

- **17.1.1** - PHP : mysqli et PDO (exemples détaillés, bonnes pratiques)
- **17.1.2** - Python : mysql-connector, PyMySQL, SQLAlchemy
- **17.1.3** - Java : JDBC, MariaDB Connector/J, intégration Spring
- **17.1.4** - Node.js : mysql2, mariadb officiel, promises et async/await
- **17.1.5** - Go : go-sql-driver, gestion du contexte, concurrence
- **17.1.6** - .NET : MySqlConnector, MariaDB.Data, ADO.NET, async/await

---

**MariaDB** : Version 11.8 LTS

⏭️ [PHP : mysqli et PDO](/17-integration-developpement/01.1-php-mysqli-pdo.md)

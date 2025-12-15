🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.5 Compatibilité des applications

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Connaissance des connecteurs de bases de données, expérience avec au moins un framework applicatif, compréhension des différences MySQL/MariaDB (section 19.1)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Évaluer la compatibilité des connecteurs et drivers avec MariaDB 11.8
- Identifier les différences de comportement SQL impactant les applications
- Adapter la configuration des ORM pour MariaDB
- Détecter et résoudre les incompatibilités applicatives avant la production
- Mettre en place des tests de régression efficaces
- Gérer les cas particuliers des applications legacy

---

## Introduction

Une migration de base de données ne se limite pas au transfert des données et du schéma. L'application qui consomme ces données doit également fonctionner correctement avec le nouveau SGBD. Cette dimension applicative est souvent sous-estimée et représente pourtant une source majeure de problèmes post-migration.

La compatibilité applicative couvre plusieurs niveaux : les connecteurs et drivers, les frameworks et ORM, les requêtes SQL générées ou écrites manuellement, et le comportement attendu par l'application. Chaque niveau peut introduire des incompatibilités subtiles qui ne se manifestent parfois qu'en production, sous certaines conditions spécifiques.

Cette section vous guide dans l'analyse systématique de la compatibilité applicative et les stratégies pour garantir une transition transparente vers MariaDB.

---

## Niveaux de compatibilité applicative

### Architecture des couches de compatibilité

```
Compatibilité applicative - Couches
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────┐
│                    APPLICATION                          │
│              (Code métier, logique)                     │
├─────────────────────────────────────────────────────────┤
│                    ORM / Framework                      │
│         (Hibernate, SQLAlchemy, Prisma...)              │
├─────────────────────────────────────────────────────────┤
│                 SQL généré / manuel                     │
│           (Requêtes, procédures stockées)               │
├─────────────────────────────────────────────────────────┤
│               Connecteur / Driver                       │
│      (JDBC, PDO, mysql2, MariaDB Connector...)          │
├─────────────────────────────────────────────────────────┤
│                Protocole MySQL/MariaDB                  │
│              (Wire protocol compatible)                 │
├─────────────────────────────────────────────────────────┤
│                    MariaDB Server                       │
└─────────────────────────────────────────────────────────┘

Chaque couche peut introduire des incompatibilités.
La compatibilité doit être validée à TOUS les niveaux.
```

### Matrice de risque par couche

| Couche | Risque MySQL→MariaDB | Risque Oracle→MariaDB | Risque version upgrade |
|--------|----------------------|----------------------|------------------------|
| **Protocole** | 🟢 Très faible | N/A | 🟢 Très faible |
| **Connecteur** | 🟢 Faible | 🟢 Faible | 🟡 Modéré |
| **SQL basique** | 🟢 Faible | 🔴 Élevé | 🟡 Modéré |
| **SQL avancé** | 🟡 Modéré | 🔴 Très élevé | 🟡 Modéré |
| **ORM** | 🟢 Faible | 🟡 Modéré | 🟢 Faible |
| **Application** | 🟡 Variable | 🔴 Variable | 🟡 Variable |

---

## Connecteurs et drivers

### Compatibilité des connecteurs MySQL avec MariaDB

MariaDB utilise le protocole MySQL, ce qui permet aux connecteurs MySQL de fonctionner avec MariaDB. Cependant, des nuances existent.

```
Compatibilité des connecteurs
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Connecteur MySQL ──────────────────▶ MariaDB Server
                                         │
    ┌────────────────────────────────────┤
    │                                    │
    ▼                                    ▼
Fonctionne                        Fonctionnalités
(protocole                        MariaDB-spécifiques
 compatible)                      non accessibles

Connecteur MariaDB ────────────────▶ MariaDB Server
                                         │
    ┌────────────────────────────────────┤
    │                                    │
    ▼                                    ▼
Fonctionne                        Toutes les
pleinement                        fonctionnalités
                                  accessibles
```

### Tableau des connecteurs par langage

| Langage | Connecteur MySQL | Connecteur MariaDB | Recommandation |
|---------|------------------|-------------------|----------------|
| **Java** | MySQL Connector/J | MariaDB Connector/J | MariaDB Connector/J |
| **Python** | mysql-connector-python, PyMySQL | mariadb | mariadb ou PyMySQL |
| **PHP** | mysqli, PDO_MySQL | mysqli, PDO_MySQL | PDO_MySQL |
| **Node.js** | mysql, mysql2 | mariadb | mariadb ou mysql2 |
| **Go** | go-sql-driver/mysql | go-sql-driver/mysql | go-sql-driver/mysql |
| **.NET** | MySql.Data | MySqlConnector, MariaDB.Data | MySqlConnector |
| **Ruby** | mysql2 | mysql2 | mysql2 |
| **Rust** | mysql | mysql | mysql |

### Configuration des connecteurs

#### Java - MariaDB Connector/J

```java
// Configuration JDBC pour MariaDB 11.8
String url = "jdbc:mariadb://localhost:3306/mydb?" +
    "useSSL=true&" +                          // TLS activé (défaut 11.8)
    "serverTimezone=UTC&" +                   // Timezone explicite
    "characterEncoding=UTF-8&" +              // UTF-8 explicite
    "useServerPrepStmts=true&" +              // Prepared statements serveur
    "cachePrepStmts=true&" +                  // Cache des PS
    "prepStmtCacheSize=250&" +                // Taille du cache
    "prepStmtCacheSqlLimit=2048&" +           // Limite SQL
    "allowMultiQueries=false";                // Sécurité

// Création de la connexion
Connection conn = DriverManager.getConnection(url, "user", "password");

// Vérification de la version
DatabaseMetaData meta = conn.getMetaData();
System.out.println("Database: " + meta.getDatabaseProductName());
System.out.println("Version: " + meta.getDatabaseProductVersion());
```

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>org.mariadb.jdbc</groupId>
    <artifactId>mariadb-java-client</artifactId>
    <version>3.3.0</version> <!-- Version compatible 11.8 -->
</dependency>
```

#### Python - Connecteur mariadb

```python
# Installation : pip install mariadb

import mariadb
import sys

# Configuration de connexion pour MariaDB 11.8
config = {
    'host': 'localhost',
    'port': 3306,
    'user': 'app_user',
    'password': 'secure_password',
    'database': 'mydb',
    'ssl': True,                    # TLS activé (défaut 11.8)
    'ssl_verify_cert': True,        # Vérification certificat
    'autocommit': False,            # Transactions explicites
    'connect_timeout': 10,
    'read_timeout': 30,
    'write_timeout': 30,
}

try:
    conn = mariadb.connect(**config)
    cursor = conn.cursor()
    
    # Vérification de la version
    cursor.execute("SELECT VERSION()")
    version = cursor.fetchone()[0]
    print(f"Connected to MariaDB {version}")
    
    # Vérification du charset
    cursor.execute("SHOW VARIABLES LIKE 'character_set_server'")
    charset = cursor.fetchone()
    print(f"Server charset: {charset[1]}")
    
except mariadb.Error as e:
    print(f"Error connecting to MariaDB: {e}")
    sys.exit(1)
finally:
    if conn:
        conn.close()
```

#### Node.js - Connecteur mariadb

```javascript
// Installation : npm install mariadb

const mariadb = require('mariadb');

// Configuration du pool de connexions
const pool = mariadb.createPool({
    host: 'localhost',
    port: 3306,
    user: 'app_user',
    password: 'secure_password',
    database: 'mydb',
    connectionLimit: 10,
    ssl: {
        rejectUnauthorized: true    // Vérification TLS (défaut 11.8)
    },
    connectTimeout: 10000,
    acquireTimeout: 10000,
    // Options spécifiques MariaDB
    allowPublicKeyRetrieval: true,
    trace: process.env.NODE_ENV === 'development'
});

// Utilisation
async function queryDatabase() {
    let conn;
    try {
        conn = await pool.getConnection();
        
        // Vérification de la version
        const version = await conn.query("SELECT VERSION() as version");
        console.log(`Connected to MariaDB ${version[0].version}`);
        
        // Requête avec paramètres
        const rows = await conn.query(
            "SELECT * FROM users WHERE status = ? LIMIT ?",
            ['active', 10]
        );
        return rows;
        
    } catch (err) {
        console.error("Database error:", err);
        throw err;
    } finally {
        if (conn) conn.release();
    }
}
```

#### PHP - PDO

```php
<?php
// Configuration PDO pour MariaDB 11.8

$dsn = 'mysql:host=localhost;port=3306;dbname=mydb;charset=utf8mb4';
$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,  // Prepared statements natifs
    PDO::MYSQL_ATTR_SSL_CA       => '/path/to/ca-cert.pem',  // TLS
    PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT => true,
    PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci",
];

try {
    $pdo = new PDO($dsn, 'app_user', 'secure_password', $options);
    
    // Vérification de la version
    $stmt = $pdo->query("SELECT VERSION()");
    $version = $stmt->fetchColumn();
    echo "Connected to MariaDB $version\n";
    
    // Vérification des paramètres
    $stmt = $pdo->query("SHOW VARIABLES LIKE 'character_set%'");
    while ($row = $stmt->fetch()) {
        echo "{$row['Variable_name']}: {$row['Value']}\n";
    }
    
} catch (PDOException $e) {
    die("Connection failed: " . $e->getMessage());
}
```

### Problèmes courants des connecteurs

| Problème | Symptôme | Solution |
|----------|----------|----------|
| **TLS requis (11.8)** | `SSL connection error` | Configurer TLS ou `ssl=false` |
| **Auth plugin** | `Authentication plugin not supported` | Mettre à jour le connecteur |
| **Charset mismatch** | Caractères corrompus | Spécifier `charset=utf8mb4` |
| **Timeout** | Connexions perdues | Configurer keepalive, reconnect |
| **Pool exhaustion** | `Too many connections` | Ajuster pool size, timeouts |

---

## Compatibilité des ORM

### Hibernate (Java)

Hibernate détecte généralement MariaDB automatiquement, mais une configuration explicite est recommandée.

```java
// hibernate.cfg.xml ou application.properties
// Configuration Hibernate pour MariaDB 11.8

// Option 1 : Dialect automatique (Hibernate 6+)
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDBDialect

// Option 2 : Dialect spécifique version
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDB106Dialect

// Configuration complète
spring.datasource.url=jdbc:mariadb://localhost:3306/mydb?useSSL=true
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.jdbc.time_zone=UTC
```

```java
// Entité avec types MariaDB spécifiques
@Entity
@Table(name = "documents")
public class Document {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(columnDefinition = "JSON")  // Type JSON MariaDB
    private String metadata;
    
    @Column(columnDefinition = "LONGTEXT")
    private String content;
    
    // Pour MariaDB 11.8 Vector (si extension)
    // @Column(columnDefinition = "VECTOR(1536)")
    // private float[] embedding;
    
    @CreationTimestamp
    @Column(name = "created_at", columnDefinition = "DATETIME(6)")
    private LocalDateTime createdAt;
}
```

**Points d'attention Hibernate :**

| Aspect | MySQL | MariaDB | Action |
|--------|-------|---------|--------|
| **Dialect** | MySQLDialect | MariaDBDialect | Changer le dialect |
| **IDENTITY** | AUTO_INCREMENT | AUTO_INCREMENT | ✅ Compatible |
| **JSON** | Type natif | Type natif | ✅ Compatible |
| **TIMESTAMP** | Jusqu'à 2038 | Jusqu'à 2106 (11.8) | ✅ Amélioré |
| **Sequences** | Non | Oui | Disponible en plus |

### SQLAlchemy (Python)

```python
# Configuration SQLAlchemy pour MariaDB 11.8

from sqlalchemy import create_engine, Column, Integer, String, JSON, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime

# URL de connexion MariaDB
DATABASE_URL = (
    "mariadb+mariadbconnector://user:password@localhost:3306/mydb"
    "?charset=utf8mb4"
)

# Alternative avec PyMySQL
# DATABASE_URL = "mysql+pymysql://user:password@localhost:3306/mydb?charset=utf8mb4"

# Création du moteur
engine = create_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,  # Vérifie la connexion avant utilisation
    pool_recycle=3600,   # Recycle les connexions après 1h
    echo=False,          # Logging SQL (True pour debug)
    connect_args={
        'ssl': {'ssl_mode': 'REQUIRED'}  # TLS pour 11.8
    }
)

Base = declarative_base()

# Modèle avec types MariaDB
class Document(Base):
    __tablename__ = 'documents'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    title = Column(String(255), nullable=False)
    content = Column(String(65535))  # TEXT
    metadata = Column(JSON)  # JSON natif MariaDB
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Index
    __table_args__ = (
        Index('idx_title', 'title'),
        {'mysql_engine': 'InnoDB', 'mysql_charset': 'utf8mb4'}
    )

# Session factory
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Vérification de la connexion
def check_connection():
    with engine.connect() as conn:
        result = conn.execute("SELECT VERSION()").fetchone()
        print(f"Connected to: {result[0]}")
```

### Prisma (Node.js/TypeScript)

```prisma
// schema.prisma pour MariaDB 11.8

datasource db {
  provider = "mysql"  // Prisma utilise le provider MySQL pour MariaDB
  url      = env("DATABASE_URL")
  // relationMode = "prisma"  // Si foreign keys désactivées
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique @db.VarChar(255)
  name      String?  @db.VarChar(100)
  metadata  Json?    // Type JSON MariaDB
  createdAt DateTime @default(now()) @db.DateTime(6)
  updatedAt DateTime @updatedAt @db.DateTime(6)
  posts     Post[]
  
  @@index([email])
  @@map("users")
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String   @db.VarChar(255)
  content   String?  @db.LongText
  published Boolean  @default(false)
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now()) @db.DateTime(6)
  
  @@index([authorId])
  @@map("posts")
}
```

```typescript
// Utilisation Prisma avec MariaDB
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error'],
});

async function main() {
  // Vérification connexion
  const result = await prisma.$queryRaw`SELECT VERSION() as version`;
  console.log('MariaDB version:', result);
  
  // Création avec JSON
  const user = await prisma.user.create({
    data: {
      email: 'user@example.com',
      name: 'John Doe',
      metadata: {
        preferences: { theme: 'dark', language: 'fr' },
        tags: ['developer', 'admin']
      }
    }
  });
  
  // Requête JSON (MariaDB)
  const users = await prisma.$queryRaw`
    SELECT * FROM users 
    WHERE JSON_EXTRACT(metadata, '$.preferences.theme') = 'dark'
  `;
}
```

### Entity Framework Core (.NET)

```csharp
// Configuration Entity Framework Core pour MariaDB 11.8

// Installation : dotnet add package Pomelo.EntityFrameworkCore.MySql

using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Document> Documents { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        var connectionString = "Server=localhost;Port=3306;Database=mydb;" +
                              "User=app_user;Password=secure_password;" +
                              "SslMode=Required;";  // TLS pour 11.8
        
        options.UseMySql(
            connectionString,
            new MariaDbServerVersion(new Version(11, 8, 0)),  // Version explicite
            mySqlOptions => {
                mySqlOptions.EnableRetryOnFailure(
                    maxRetryCount: 3,
                    maxRetryDelay: TimeSpan.FromSeconds(10),
                    errorNumbersToAdd: null
                );
                mySqlOptions.CommandTimeout(60);
            }
        );
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>(entity =>
        {
            entity.ToTable("users");
            entity.Property(e => e.Metadata)
                  .HasColumnType("json");  // JSON MariaDB
            entity.Property(e => e.CreatedAt)
                  .HasColumnType("datetime(6)");
        });
    }
}

public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
    public string Name { get; set; }
    public string Metadata { get; set; }  // JSON stocké comme string
    public DateTime CreatedAt { get; set; }
}
```

---

## Différences SQL impactant les applications

### Fonctions et comportements divergents

Certaines fonctions SQL ont des comportements légèrement différents entre MySQL et MariaDB, ou entre versions MariaDB.

#### Fonctions de date et heure

```sql
-- Comportement identique
SELECT NOW(), CURDATE(), CURTIME();
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');

-- MariaDB 11.8 : TIMESTAMP étendu jusqu'à 2106
-- Les applications manipulant des dates > 2038 bénéficient de cette extension
CREATE TABLE future_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_date TIMESTAMP,  -- Supporte maintenant > 2038
    description VARCHAR(255)
);

INSERT INTO future_events (event_date, description)
VALUES ('2050-01-01 00:00:00', 'Future event');  -- OK en 11.8
```

#### Fonctions JSON

```sql
-- Compatible MySQL 8.0 et MariaDB
SELECT JSON_EXTRACT(data, '$.name') FROM users;
SELECT JSON_UNQUOTE(JSON_EXTRACT(data, '$.email')) FROM users;
SELECT data->>'$.email' FROM users;  -- Raccourci

-- MariaDB uniquement
SELECT JSON_DETAILED(data) FROM users;  -- Formatage lisible
SELECT JSON_LOOSE(data) FROM users;     -- Format compact

-- MySQL 8.0 uniquement (NON compatible MariaDB)
-- SELECT * FROM JSON_TABLE(...);  -- À réécrire
-- SELECT 'value' MEMBER OF(json_array);  -- Utiliser JSON_CONTAINS
```

**Réécriture des fonctions incompatibles :**

```sql
-- MySQL 8.0 : JSON_TABLE (non supporté MariaDB < 10.6)
-- Original MySQL :
SELECT jt.* 
FROM orders,
     JSON_TABLE(items, '$[*]' COLUMNS (
         item_id INT PATH '$.id',
         quantity INT PATH '$.qty'
     )) AS jt;

-- Alternative MariaDB :
SELECT 
    JSON_EXTRACT(items, CONCAT('$[', n.n, '].id')) AS item_id,
    JSON_EXTRACT(items, CONCAT('$[', n.n, '].qty')) AS quantity
FROM orders
CROSS JOIN (
    SELECT 0 AS n UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
) AS n
WHERE JSON_EXTRACT(items, CONCAT('$[', n.n, ']')) IS NOT NULL;

-- MySQL 8.0 : MEMBER OF
-- Original :
SELECT * FROM products WHERE 'electronics' MEMBER OF(categories);

-- MariaDB :
SELECT * FROM products WHERE JSON_CONTAINS(categories, '"electronics"');
```

#### Expressions régulières

```sql
-- MariaDB utilise PCRE (Perl Compatible Regular Expressions)
-- MySQL 8.0 utilise ICU

-- Syntaxe compatible
SELECT * FROM logs WHERE message REGEXP 'error|warning';

-- MariaDB : PCRE avancé
SELECT REGEXP_REPLACE(text, '\\s+', ' ') FROM documents;  -- PCRE
SELECT REGEXP_SUBSTR(email, '[^@]+') FROM users;

-- Différences subtiles de comportement possibles
-- Toujours tester les regex complexes après migration
```

#### Mode SQL et comportement strict

```sql
-- Vérifier le sql_mode actuel
SELECT @@sql_mode;

-- MariaDB 11.8 sql_mode par défaut
-- STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

-- Configuration recommandée pour compatibilité
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';

-- Pour applications legacy nécessitant un mode permissif
-- (À éviter, mais parfois nécessaire en migration)
SET SESSION sql_mode = '';
```

### Tableau des différences SQL courantes

| Fonctionnalité | MySQL 8.0 | MariaDB 11.8 | Action migration |
|----------------|-----------|--------------|------------------|
| `JSON_TABLE()` | ✅ | ✅ (10.6+) | Compatible |
| `MEMBER OF()` | ✅ | ❌ | Réécrire avec JSON_CONTAINS |
| `->>`  opérateur | ✅ | ✅ | Compatible |
| `REGEXP_REPLACE()` | ICU | PCRE | Tester les regex |
| `WITH RECURSIVE` | ✅ | ✅ | Compatible |
| `LATERAL` | ✅ | ✅ (10.3+) | Compatible |
| `EXPLAIN ANALYZE` | ✅ | ✅ | Compatible |
| `INVISIBLE INDEX` | ✅ | ✅ | Compatible |
| `SKIP LOCKED` | ✅ | ✅ (10.6+) | Compatible |
| Collations `0900` | ✅ | ❌ | Convertir |

---

## Gestion des collations et charsets

### Impact de utf8mb4 par défaut en 11.8 🆕

MariaDB 11.8 utilise `utf8mb4` comme charset par défaut. Cela impacte les applications.

```sql
-- Vérification du charset par défaut
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- MariaDB 11.8 valeurs par défaut
-- character_set_server = utf8mb4
-- collation_server = utf8mb4_uca1400_ai_ci  🆕

-- Impact sur les nouvelles tables
CREATE TABLE test_default (
    name VARCHAR(100)  -- Sera utf8mb4, jusqu'à 400 bytes
);

-- Forcer un charset spécifique si nécessaire
CREATE TABLE test_latin1 (
    code VARCHAR(10) CHARACTER SET latin1
);
```

**Vérification de la compatibilité applicative :**

```python
# Script Python de vérification charset
import mariadb

def check_charset_compatibility(conn):
    cursor = conn.cursor()
    
    # Vérifier les tables avec charset non-utf8mb4
    cursor.execute("""
        SELECT 
            table_schema,
            table_name,
            table_collation
        FROM information_schema.tables
        WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema')
          AND table_collation NOT LIKE 'utf8mb4%'
    """)
    
    non_utf8mb4_tables = cursor.fetchall()
    
    if non_utf8mb4_tables:
        print("⚠️ Tables avec charset non-utf8mb4:")
        for schema, table, collation in non_utf8mb4_tables:
            print(f"  - {schema}.{table}: {collation}")
    else:
        print("✅ Toutes les tables utilisent utf8mb4")
    
    # Vérifier les colonnes avec collation incompatible
    cursor.execute("""
        SELECT 
            table_schema,
            table_name,
            column_name,
            collation_name
        FROM information_schema.columns
        WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema')
          AND collation_name LIKE '%0900%'
    """)
    
    incompatible_collations = cursor.fetchall()
    
    if incompatible_collations:
        print("🔴 Colonnes avec collations incompatibles (0900):")
        for schema, table, column, collation in incompatible_collations:
            print(f"  - {schema}.{table}.{column}: {collation}")
    
    return len(non_utf8mb4_tables), len(incompatible_collations)
```

### Conversion des collations

```sql
-- Identifier les collations à convertir
SELECT DISTINCT collation_name
FROM information_schema.columns
WHERE collation_name IS NOT NULL
  AND table_schema = 'mydb';

-- Conversion d'une table
ALTER TABLE users 
CONVERT TO CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- Conversion d'une colonne spécifique
ALTER TABLE users
MODIFY COLUMN name VARCHAR(255) 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- Script de conversion massive
DELIMITER //
CREATE PROCEDURE convert_to_utf8mb4()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE tbl_name VARCHAR(255);
    DECLARE cur CURSOR FOR 
        SELECT table_name 
        FROM information_schema.tables 
        WHERE table_schema = DATABASE()
          AND table_type = 'BASE TABLE';
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN cur;
    
    read_loop: LOOP
        FETCH cur INTO tbl_name;
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        SET @sql = CONCAT('ALTER TABLE `', tbl_name, 
            '` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci');
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
        
        SELECT CONCAT('Converted: ', tbl_name) AS status;
    END LOOP;
    
    CLOSE cur;
END //
DELIMITER ;
```

---

## Tests de compatibilité applicative

### Framework de tests

```python
# framework_test_compatibility.py
# Framework de tests de compatibilité applicative

import unittest
import mariadb
from datetime import datetime, timedelta
import json

class MariaDBCompatibilityTests(unittest.TestCase):
    """Suite de tests de compatibilité pour migration vers MariaDB 11.8"""
    
    @classmethod
    def setUpClass(cls):
        cls.conn = mariadb.connect(
            host='localhost',
            port=3306,
            user='test_user',
            password='test_password',
            database='test_db'
        )
        cls.cursor = cls.conn.cursor()
    
    @classmethod
    def tearDownClass(cls):
        cls.cursor.close()
        cls.conn.close()
    
    def test_connection(self):
        """Test de connexion basique"""
        self.cursor.execute("SELECT 1")
        result = self.cursor.fetchone()
        self.assertEqual(result[0], 1)
    
    def test_version(self):
        """Vérification de la version MariaDB"""
        self.cursor.execute("SELECT VERSION()")
        version = self.cursor.fetchone()[0]
        self.assertIn('11.8', version, f"Expected MariaDB 11.8, got {version}")
    
    def test_charset_utf8mb4(self):
        """Vérification du charset par défaut"""
        self.cursor.execute("SHOW VARIABLES LIKE 'character_set_server'")
        charset = self.cursor.fetchone()[1]
        self.assertEqual(charset, 'utf8mb4')
    
    def test_json_operations(self):
        """Test des opérations JSON"""
        # Création table de test
        self.cursor.execute("""
            CREATE TEMPORARY TABLE test_json (
                id INT AUTO_INCREMENT PRIMARY KEY,
                data JSON
            )
        """)
        
        # Insertion JSON
        test_data = {'name': 'Test', 'values': [1, 2, 3], 'nested': {'key': 'value'}}
        self.cursor.execute(
            "INSERT INTO test_json (data) VALUES (?)",
            (json.dumps(test_data),)
        )
        
        # Extraction JSON
        self.cursor.execute("SELECT data->>'$.name' FROM test_json")
        result = self.cursor.fetchone()[0]
        self.assertEqual(result, 'Test')
        
        # JSON_EXTRACT
        self.cursor.execute("SELECT JSON_EXTRACT(data, '$.values[0]') FROM test_json")
        result = self.cursor.fetchone()[0]
        self.assertEqual(int(result), 1)
    
    def test_datetime_extended(self):
        """Test des TIMESTAMP étendus (>2038) - Nouveauté 11.8"""
        self.cursor.execute("""
            CREATE TEMPORARY TABLE test_dates (
                id INT AUTO_INCREMENT PRIMARY KEY,
                future_date DATETIME
            )
        """)
        
        # Date au-delà de 2038
        future_date = datetime(2050, 1, 1, 12, 0, 0)
        self.cursor.execute(
            "INSERT INTO test_dates (future_date) VALUES (?)",
            (future_date,)
        )
        
        self.cursor.execute("SELECT future_date FROM test_dates")
        result = self.cursor.fetchone()[0]
        self.assertEqual(result.year, 2050)
    
    def test_window_functions(self):
        """Test des fonctions de fenêtrage"""
        self.cursor.execute("""
            CREATE TEMPORARY TABLE test_window (
                id INT AUTO_INCREMENT PRIMARY KEY,
                category VARCHAR(50),
                value INT
            )
        """)
        
        # Insertion de données
        self.cursor.executemany(
            "INSERT INTO test_window (category, value) VALUES (?, ?)",
            [('A', 10), ('A', 20), ('B', 15), ('B', 25)]
        )
        
        # Test ROW_NUMBER
        self.cursor.execute("""
            SELECT category, value,
                   ROW_NUMBER() OVER (PARTITION BY category ORDER BY value) as rn
            FROM test_window
        """)
        results = self.cursor.fetchall()
        self.assertEqual(len(results), 4)
    
    def test_cte_recursive(self):
        """Test des CTE récursives"""
        self.cursor.execute("""
            WITH RECURSIVE numbers AS (
                SELECT 1 AS n
                UNION ALL
                SELECT n + 1 FROM numbers WHERE n < 5
            )
            SELECT * FROM numbers
        """)
        results = self.cursor.fetchall()
        self.assertEqual(len(results), 5)
        self.assertEqual([r[0] for r in results], [1, 2, 3, 4, 5])
    
    def test_prepared_statements(self):
        """Test des prepared statements"""
        self.cursor.execute("""
            CREATE TEMPORARY TABLE test_prep (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(100)
            )
        """)
        
        # Insertion avec prepared statement
        stmt = "INSERT INTO test_prep (name) VALUES (?)"
        self.cursor.execute(stmt, ('Test Name',))
        
        # Sélection avec prepared statement
        self.cursor.execute("SELECT name FROM test_prep WHERE id = ?", (1,))
        result = self.cursor.fetchone()[0]
        self.assertEqual(result, 'Test Name')
    
    def test_transactions(self):
        """Test du comportement transactionnel"""
        self.cursor.execute("""
            CREATE TEMPORARY TABLE test_tx (
                id INT AUTO_INCREMENT PRIMARY KEY,
                value INT
            ) ENGINE=InnoDB
        """)
        
        # Test ROLLBACK
        self.conn.begin()
        self.cursor.execute("INSERT INTO test_tx (value) VALUES (100)")
        self.conn.rollback()
        
        self.cursor.execute("SELECT COUNT(*) FROM test_tx")
        count = self.cursor.fetchone()[0]
        self.assertEqual(count, 0, "Rollback should have removed the row")
        
        # Test COMMIT
        self.conn.begin()
        self.cursor.execute("INSERT INTO test_tx (value) VALUES (200)")
        self.conn.commit()
        
        self.cursor.execute("SELECT COUNT(*) FROM test_tx")
        count = self.cursor.fetchone()[0]
        self.assertEqual(count, 1, "Commit should have persisted the row")


class QueryCompatibilityTests(unittest.TestCase):
    """Tests de compatibilité des requêtes applicatives"""
    
    @classmethod
    def setUpClass(cls):
        cls.conn = mariadb.connect(
            host='localhost',
            port=3306,
            user='test_user',
            password='test_password',
            database='test_db'
        )
        cls.cursor = cls.conn.cursor()
    
    def test_app_query_1(self):
        """Test requête applicative #1 - À personnaliser"""
        # Remplacer par vos requêtes applicatives réelles
        query = "SELECT 1 + 1 AS result"
        self.cursor.execute(query)
        result = self.cursor.fetchone()[0]
        self.assertEqual(result, 2)
    
    def test_app_stored_procedure(self):
        """Test procédure stockée applicative"""
        # À adapter avec vos procédures
        pass


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

### Script de validation des requêtes

```bash
#!/bin/bash
# validate_queries.sh
# Validation des requêtes applicatives sur MariaDB 11.8

MARIADB_HOST="localhost"
MARIADB_USER="test_user"
MARIADB_PASS="test_password"
MARIADB_DB="test_db"
QUERY_FILE="application_queries.sql"
RESULTS_DIR="./validation_results"

mkdir -p $RESULTS_DIR

echo "=== Validation des requêtes applicatives ==="
echo "Host: $MARIADB_HOST"
echo "Database: $MARIADB_DB"
echo ""

# Compteurs
total=0
success=0
failed=0

# Lecture et exécution des requêtes
while IFS= read -r query || [[ -n "$query" ]]; do
    # Ignorer les lignes vides et commentaires
    [[ -z "$query" || "$query" =~ ^[[:space:]]*-- ]] && continue
    
    total=$((total + 1))
    echo -n "[$total] Testing query... "
    
    # Exécuter la requête
    result=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS \
             $MARIADB_DB -e "$query" 2>&1)
    exit_code=$?
    
    if [ $exit_code -eq 0 ]; then
        echo "✅ OK"
        success=$((success + 1))
    else
        echo "❌ FAILED"
        echo "   Query: ${query:0:80}..."
        echo "   Error: $result"
        failed=$((failed + 1))
        
        # Logger l'erreur
        echo "Query: $query" >> $RESULTS_DIR/failed_queries.log
        echo "Error: $result" >> $RESULTS_DIR/failed_queries.log
        echo "---" >> $RESULTS_DIR/failed_queries.log
    fi
done < "$QUERY_FILE"

echo ""
echo "=== Résumé ==="
echo "Total: $total"
echo "Succès: $success"
echo "Échecs: $failed"
echo ""

if [ $failed -gt 0 ]; then
    echo "⚠️ Des requêtes ont échoué. Voir $RESULTS_DIR/failed_queries.log"
    exit 1
else
    echo "✅ Toutes les requêtes ont réussi"
    exit 0
fi
```

---

## Scénarios de problèmes courants

### Scénario 1 : Application PHP legacy

**Problème** : Application PHP utilisant l'extension `mysql_*` deprecated.

```php
// Code legacy (NON fonctionnel avec PHP 7+)
$conn = mysql_connect("localhost", "user", "password");
mysql_select_db("mydb", $conn);
$result = mysql_query("SELECT * FROM users");

// Migration vers PDO (recommandé)
$pdo = new PDO(
    'mysql:host=localhost;dbname=mydb;charset=utf8mb4',
    'user',
    'password',
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);
$stmt = $pdo->query("SELECT * FROM users");
$result = $stmt->fetchAll(PDO::FETCH_ASSOC);

// Migration vers mysqli (alternative)
$mysqli = new mysqli("localhost", "user", "password", "mydb");
$mysqli->set_charset("utf8mb4");
$result = $mysqli->query("SELECT * FROM users");
```

### Scénario 2 : ORM générant des requêtes incompatibles

**Problème** : Hibernate générant des collations MySQL 8.0.

```java
// Erreur typique
// org.hibernate.exception.SQLGrammarException: 
// Unknown collation: 'utf8mb4_0900_ai_ci'

// Solution : Configuration du dialect
// application.properties
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDB106Dialect

// Ou override du schema generation
spring.jpa.properties.hibernate.hbm2ddl.auto=none
// Puis utiliser Flyway/Liquibase avec scripts MariaDB compatibles
```

### Scénario 3 : Connexions TLS rejetées

**Problème** : MariaDB 11.8 exige TLS par défaut.

```python
# Erreur typique
# mariadb.OperationalError: SSL connection error

# Solution 1 : Configurer TLS côté client
config = {
    'host': 'localhost',
    'user': 'app_user',
    'password': 'password',
    'database': 'mydb',
    'ssl': True,
    'ssl_ca': '/path/to/ca-cert.pem'  # Certificat CA
}

# Solution 2 : Désactiver l'exigence TLS (non recommandé en production)
# Sur le serveur MariaDB :
# SET GLOBAL require_secure_transport = OFF;

# Solution 3 : Connexion avec ssl_mode
config = {
    'host': 'localhost',
    'user': 'app_user',
    'password': 'password',
    'database': 'mydb',
    'ssl': {'ssl_mode': 'PREFERRED'}  # Ou 'DISABLED' si vraiment nécessaire
}
```

---

## ✅ Points clés à retenir

- La compatibilité applicative doit être validée à **tous les niveaux** : connecteur, ORM, SQL, application
- Les **connecteurs MySQL** fonctionnent avec MariaDB, mais les **connecteurs MariaDB** offrent un meilleur support
- Configurez explicitement le **dialect MariaDB** dans vos ORM (Hibernate, SQLAlchemy, Prisma)
- MariaDB 11.8 utilise **utf8mb4 par défaut** : vérifiez l'espace disque et les index
- Le **TLS est activé par défaut** en 11.8 : configurez les certificats ou désactivez si nécessaire
- Certaines fonctions MySQL 8.0 (`MEMBER OF`, collations `0900`) nécessitent une **réécriture**
- Mettez en place des **tests de régression automatisés** avant et après migration
- Testez les **requêtes critiques** de l'application dans l'environnement cible

---

## 🔗 Ressources et références

- [📖 MariaDB Connector/J Documentation](https://mariadb.com/kb/en/mariadb-connector-j/)
- [📖 MariaDB Connector/Python](https://mariadb.com/kb/en/mariadb-connector-python/)
- [📖 MariaDB Connector/Node.js](https://mariadb.com/kb/en/mariadb-connector-nodejs/)
- [📖 Hibernate MariaDB Dialect](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [📖 SQLAlchemy MariaDB](https://docs.sqlalchemy.org/en/20/dialects/mysql.html)
- [📖 Prisma MySQL/MariaDB](https://www.prisma.io/docs/concepts/database-connectors/mysql)
- [📖 Pomelo Entity Framework Core](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql)

---

## ➡️ Section suivante

**[19.6 Tests de compatibilité](./06-tests-compatibilite.md)** : Nous approfondirons les méthodologies de tests de compatibilité : environnements de validation, automatisation des tests de non-régression, benchmarking comparatif, et stratégies de validation en conditions réelles.

⏭️ [Tests de compatibilité](/19-migration-compatibilite/06-tests-compatibilite.md)

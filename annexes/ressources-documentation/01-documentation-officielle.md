🔝 Retour au [Sommaire](/SOMMAIRE.md)

# H.1 Documentation Officielle 📖

> **Niveau** : Tous niveaux (Référence)  
> **Durée estimée** : 15-20 minutes  
> **Prérequis** : Aucun

## 🎯 Objectif de cette section

Fournir un **guide complet et structuré** de la documentation officielle MariaDB pour :
- Trouver rapidement l'information recherchée
- Comprendre l'organisation de la documentation
- Utiliser efficacement les outils de recherche
- Identifier les ressources appropriées selon le contexte

---

## 📚 Vue d'Ensemble de la Documentation

### Portail Principal

**URL** : [https://mariadb.com/kb/en/](https://mariadb.com/kb/en/)

La **MariaDB Knowledge Base (KB)** est la source officielle et autoritaire pour toute documentation technique MariaDB.

**Caractéristiques** :
- ✅ **Gratuite et open-source**
- ✅ **Constamment mise à jour** par la MariaDB Foundation et la communauté
- ✅ **Multilingue** (anglais principal, traductions partielles)
- ✅ **Versionné** (documentation spécifique par version)
- ✅ **Recherche puissante** avec suggestions
- ✅ **Contributive** (possibilité de proposer corrections)

---

## 🗺️ Organisation de la Knowledge Base

### Structure Hiérarchique

```
MariaDB Knowledge Base
│
├─ 📘 Getting Started
│   ├─ Installation guides
│   ├─ First steps
│   └─ Basic tutorials
│
├─ 📗 MariaDB Server
│   ├─ SQL Statements
│   ├─ Built-in Functions
│   ├─ Data Types
│   ├─ Engines & Features
│   └─ Server Administration
│
├─ 📙 High Availability
│   ├─ Replication
│   ├─ Galera Cluster
│   └─ MaxScale
│
├─ 📕 Clients & APIs
│   ├─ Connectors
│   ├─ Client programs
│   └─ APIs & Protocols
│
├─ 📔 Tools
│   ├─ mariadb-dump
│   ├─ Mariabackup
│   └─ Administration tools
│
└─ 📓 Development
    ├─ Contributing
    ├─ Plugin development
    └─ Source code
```

---

## 1️⃣ Getting Started - Débuter avec MariaDB

### URL : [https://mariadb.com/kb/en/getting-started-with-mariadb/](https://mariadb.com/kb/en/getting-started-with-mariadb/)

**Public cible** : 🟢 Débutants

#### Sous-sections principales

| Section | Description | Lien |
|---------|-------------|------|
| **Getting, Installing, and Upgrading** | Téléchargement, installation multi-OS | [KB Getting Installing](https://mariadb.com/kb/en/getting-installing-and-upgrading-mariadb/) |
| **A MariaDB Primer** | Introduction concepts fondamentaux | [KB Primer](https://mariadb.com/kb/en/a-mariadb-primer/) |
| **MariaDB Basics** | Tutoriels de base (CREATE, SELECT, etc.) | [KB Basics](https://mariadb.com/kb/en/mariadb-basics/) |
| **Training & Tutorials** | Guides d'apprentissage structurés | [KB Training](https://mariadb.com/kb/en/training-tutorials/) |

#### Guides d'installation par OS

| OS | Guide | Notes |
|----|-------|-------|
| **Linux (Debian/Ubuntu)** | [Installing MariaDB on Debian/Ubuntu](https://mariadb.com/kb/en/installing-mariadb-deb-files/) | apt/dpkg |
| **Linux (RHEL/CentOS)** | [Installing MariaDB on RHEL/CentOS](https://mariadb.com/kb/en/yum/) | yum/dnf |
| **Windows** | [Installing MariaDB on Windows](https://mariadb.com/kb/en/installing-mariadb-msi-packages-on-windows/) | MSI installer |
| **macOS** | [Installing MariaDB on macOS](https://mariadb.com/kb/en/installing-mariadb-on-macos-using-homebrew/) | Homebrew |
| **Docker** | [Installing MariaDB with Docker](https://mariadb.com/kb/en/installing-and-using-mariadb-via-docker/) | Container |
| **Source** | [Compiling from Source](https://mariadb.com/kb/en/generic-build-instructions/) | cmake, gcc |

💡 **Recommandation** : Utiliser les repositories officiels MariaDB plutôt que les packages de distribution pour avoir les dernières versions.

---

## 2️⃣ SQL Statements & Structure

### URL : [https://mariadb.com/kb/en/sql-statements-structure/](https://mariadb.com/kb/en/sql-statements-structure/)

**Public cible** : 🟡 Tous niveaux

Référence **exhaustive** de toutes les commandes SQL supportées par MariaDB.

#### Catégories principales

| Catégorie | Description | Exemples |
|-----------|-------------|----------|
| **Data Definition** | Structure des données | CREATE, ALTER, DROP, TRUNCATE |
| **Data Manipulation** | Manipulation des données | SELECT, INSERT, UPDATE, DELETE |
| **Transactions** | Gestion transactionnelle | START TRANSACTION, COMMIT, ROLLBACK |
| **Prepared Statements** | Requêtes préparées | PREPARE, EXECUTE, DEALLOCATE |
| **Compound Statements** | Programmation SQL | BEGIN...END, IF, LOOP, WHILE |
| **Account Management** | Gestion utilisateurs | CREATE USER, GRANT, REVOKE |
| **Table Maintenance** | Maintenance tables | ANALYZE, OPTIMIZE, REPAIR |
| **Utility Statements** | Commandes utilitaires | SHOW, DESCRIBE, EXPLAIN |

#### Exemples de pages de référence

**CREATE TABLE** : [https://mariadb.com/kb/en/create-table/](https://mariadb.com/kb/en/create-table/)
```sql
CREATE TABLE employees (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    hire_date DATE,
    salary DECIMAL(10,2),
    INDEX idx_hire_date (hire_date)
) ENGINE=InnoDB;
```

**SELECT Statement** : [https://mariadb.com/kb/en/select/](https://mariadb.com/kb/en/select/)
```sql
SELECT 
    e.name,
    e.email,
    d.department_name,
    e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.salary > 50000
ORDER BY e.salary DESC
LIMIT 10;
```

**Window Functions** : [https://mariadb.com/kb/en/window-functions/](https://mariadb.com/kb/en/window-functions/)
```sql
SELECT 
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as rank,
    AVG(salary) OVER () as avg_salary
FROM employees;
```

---

## 3️⃣ Built-in Functions

### URL : [https://mariadb.com/kb/en/built-in-functions/](https://mariadb.com/kb/en/built-in-functions/)

**Public cible** : 🟡 Tous niveaux

Catalogue complet de **toutes les fonctions** disponibles dans MariaDB.

#### Catégories de fonctions

| Catégorie | Nombre | Exemples | Lien |
|-----------|--------|----------|------|
| **String Functions** | 70+ | CONCAT, SUBSTRING, REGEXP_REPLACE | [KB String](https://mariadb.com/kb/en/string-functions/) |
| **Numeric Functions** | 50+ | ROUND, FLOOR, ABS, RAND | [KB Numeric](https://mariadb.com/kb/en/numeric-functions/) |
| **Date & Time** | 60+ | NOW, DATE_FORMAT, DATEDIFF | [KB Date](https://mariadb.com/kb/en/date-time-functions/) |
| **Aggregate Functions** | 20+ | SUM, AVG, COUNT, GROUP_CONCAT | [KB Aggregate](https://mariadb.com/kb/en/aggregate-functions/) |
| **Window Functions** | 15+ | ROW_NUMBER, RANK, LAG, LEAD | [KB Window](https://mariadb.com/kb/en/window-functions/) |
| **JSON Functions** | 30+ | JSON_EXTRACT, JSON_SET, JSON_ARRAY | [KB JSON](https://mariadb.com/kb/en/json-functions/) |
| **🆕 Vector Functions** | 10+ | VEC_DISTANCE_COSINE, VEC_FromText | [KB Vector](https://mariadb.com/kb/en/vector-functions/) |
| **Encryption** | 15+ | AES_ENCRYPT, SHA2, PASSWORD | [KB Encryption](https://mariadb.com/kb/en/encryption-functions/) |
| **Control Flow** | 8 | IF, CASE, COALESCE, NULLIF | [KB Control](https://mariadb.com/kb/en/control-flow-functions/) |

#### Nouveautés MariaDB 11.8

**Vector Functions** : [https://mariadb.com/kb/en/vector-functions/](https://mariadb.com/kb/en/vector-functions/)

```sql
-- Distance cosinus entre vecteurs
SELECT VEC_DISTANCE_COSINE(
    embedding, 
    '[0.1, 0.2, 0.3]'::VECTOR
) AS similarity
FROM documents;

-- Conversion texte → vecteur
SELECT VEC_FromText('[1.0, 2.0, 3.0]');

-- Normalisation vecteur
SELECT VEC_Normalize(embedding) FROM vectors;
```

---

## 4️⃣ Data Types

### URL : [https://mariadb.com/kb/en/data-types/](https://mariadb.com/kb/en/data-types/)

**Public cible** : 🟡 Tous niveaux

Référence complète de tous les **types de données** supportés.

#### Types principaux

| Catégorie | Types | Page KB |
|-----------|-------|---------|
| **Numeric** | INT, BIGINT, DECIMAL, FLOAT, DOUBLE | [KB Numeric Types](https://mariadb.com/kb/en/numeric-data-types/) |
| **String** | VARCHAR, CHAR, TEXT, BLOB, ENUM, SET | [KB String Types](https://mariadb.com/kb/en/string-data-types/) |
| **Date/Time** | DATE, DATETIME, TIMESTAMP, TIME, YEAR | [KB Date Types](https://mariadb.com/kb/en/date-and-time-data-types/) |
| **JSON** | JSON (alias LONGTEXT) | [KB JSON Type](https://mariadb.com/kb/en/json-data-type/) |
| **🆕 Vector** | VECTOR(dimensions) | [KB Vector Type](https://mariadb.com/kb/en/vector-data-type/) |
| **Spatial** | GEOMETRY, POINT, LINESTRING, POLYGON | [KB Spatial Types](https://mariadb.com/kb/en/geometry-types/) |
| **Binary** | BINARY, VARBINARY, BLOB | [KB Binary Types](https://mariadb.com/kb/en/string-data-types/) |

#### Nouveauté 11.8 : Type VECTOR

**Documentation** : [https://mariadb.com/kb/en/vector-data-type/](https://mariadb.com/kb/en/vector-data-type/)

```sql
-- Déclaration
CREATE TABLE embeddings (
    id INT PRIMARY KEY,
    description TEXT,
    vector_data VECTOR(1536)  -- 1536 dimensions
);

-- Support dimensions : 1 à 65,535
-- Format de stockage optimisé
-- Compatible InnoDB et Aria
```

---

## 5️⃣ Storage Engines

### URL : [https://mariadb.com/kb/en/mariadb-storage-engines/](https://mariadb.com/kb/en/mariadb-storage-engines/)

**Public cible** : 🟡 Intermédiaire+

Documentation détaillée de chaque **moteur de stockage**.

#### Moteurs principaux

| Moteur | Page KB | Cas d'usage |
|--------|---------|-------------|
| **InnoDB** | [KB InnoDB](https://mariadb.com/kb/en/innodb/) | OLTP, transactions, défaut |
| **Aria** | [KB Aria](https://mariadb.com/kb/en/aria/) | Crash-safe, remplacement MyISAM |
| **MyISAM** | [KB MyISAM](https://mariadb.com/kb/en/myisam/) | Legacy (non recommandé) |
| **ColumnStore** | [KB ColumnStore](https://mariadb.com/kb/en/columnstore/) | OLAP, analytics |
| **S3** | [KB S3](https://mariadb.com/kb/en/s3-storage-engine/) | Archivage cloud |
| **Spider** | [KB Spider](https://mariadb.com/kb/en/spider/) | Sharding distribué |
| **CONNECT** | [KB CONNECT](https://mariadb.com/kb/en/connect/) | Données externes |
| **Memory** | [KB Memory](https://mariadb.com/kb/en/memory-storage-engine/) | Tables en RAM |

#### InnoDB - Documentation approfondie

**Configuration** : [https://mariadb.com/kb/en/innodb-system-variables/](https://mariadb.com/kb/en/innodb-system-variables/)

```ini
# my.cnf - Configuration InnoDB optimisée
[mysqld]
# Buffer Pool (70-80% RAM disponible)
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8

# Redo logs
innodb_log_file_size = 1G
innodb_log_buffer_size = 16M

# I/O optimizations
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_flush_method = O_DIRECT
innodb_flush_neighbors = 0  # SSD

# Nouvelles optimisations 11.8
innodb_alter_copy_bulk = ON
```

**Monitoring** : [https://mariadb.com/kb/en/innodb-status-variables/](https://mariadb.com/kb/en/innodb-status-variables/)

```sql
-- État du buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Statistiques I/O
SHOW STATUS LIKE 'Innodb_data%';

-- État des transactions
SHOW ENGINE INNODB STATUS\G
```

---

## 6️⃣ Server Administration

### URL : [https://mariadb.com/kb/en/server-administration/](https://mariadb.com/kb/en/server-administration/)

**Public cible** : 🟠 DBA, Administrateurs

Documentation complète pour **administrer** un serveur MariaDB.

#### Sections clés

| Section | Description | Lien |
|---------|-------------|------|
| **System Variables** | Configuration serveur | [KB System Variables](https://mariadb.com/kb/en/server-system-variables/) |
| **Status Variables** | Métriques monitoring | [KB Status Variables](https://mariadb.com/kb/en/server-status-variables/) |
| **Server Monitoring** | Surveillance performances | [KB Monitoring](https://mariadb.com/kb/en/server-monitoring-logs/) |
| **Optimization** | Tuning et optimisation | [KB Optimization](https://mariadb.com/kb/en/optimization-and-tuning/) |
| **Security** | Sécurisation serveur | [KB Security](https://mariadb.com/kb/en/securing-mariadb/) |
| **Backup & Restore** | Sauvegardes | [KB Backup](https://mariadb.com/kb/en/backup-and-restore-overview/) |
| **User Management** | Gestion utilisateurs | [KB Users](https://mariadb.com/kb/en/account-management-sql-commands/) |

#### Configuration Files

**my.cnf Reference** : [https://mariadb.com/kb/en/configuring-mariadb-with-option-files/](https://mariadb.com/kb/en/configuring-mariadb-with-option-files/)

**Localisation standard** :
- Linux : `/etc/mysql/my.cnf`, `/etc/my.cnf`
- Windows : `C:\Program Files\MariaDB\data\my.ini`
- macOS : `/usr/local/etc/my.cnf`

**Structure** :
```ini
[client]
port = 3306
socket = /var/run/mysqld/mysqld.sock

[mysqld]
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock
pid-file = /var/run/mysqld/mysqld.pid

# Character set (11.8 default)
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# InnoDB
innodb_buffer_pool_size = 8G
innodb_log_file_size = 1G

# Binary logging
log-bin = mysql-bin
server-id = 1

[mysql]
default-character-set = utf8mb4
```

---

## 7️⃣ High Availability & Replication

### URL : [https://mariadb.com/kb/en/high-availability-performance-tuning-mariadb-replication/](https://mariadb.com/kb/en/high-availability-performance-tuning-mariadb-replication/)

**Public cible** : 🟠 DBA Avancé

#### Replication

**Documentation** : [https://mariadb.com/kb/en/replication/](https://mariadb.com/kb/en/replication/)

**Guides principaux** :

| Guide | Description | Lien |
|-------|-------------|------|
| **Replication Overview** | Introduction concepts | [KB Overview](https://mariadb.com/kb/en/replication-overview/) |
| **Setting Up Replication** | Configuration step-by-step | [KB Setup](https://mariadb.com/kb/en/setting-up-replication/) |
| **GTID Replication** | Global Transaction ID | [KB GTID](https://mariadb.com/kb/en/gtid/) |
| **Multi-Source Replication** | Plusieurs sources | [KB Multi-Source](https://mariadb.com/kb/en/multi-source-replication/) |
| **Replication Filters** | Filtrage sélectif | [KB Filters](https://mariadb.com/kb/en/replication-filters/) |

#### Galera Cluster

**Documentation** : [https://mariadb.com/kb/en/galera-cluster/](https://mariadb.com/kb/en/galera-cluster/)

**Guides essentiels** :

```
Galera Documentation
├─ Getting Started with Galera
├─ Galera Cluster System Variables
├─ State Snapshot Transfer (SST)
│   ├─ mariabackup SST
│   └─ rsync SST
├─ Incremental State Transfer (IST)
├─ Monitoring Galera Cluster
└─ Galera Cluster Best Practices
```

#### MaxScale

**Documentation** : [https://mariadb.com/kb/en/maxscale/](https://mariadb.com/kb/en/maxscale/)

**Version 25.01 (11.8)** : [https://mariadb.com/kb/en/mariadb-maxscale-2501/](https://mariadb.com/kb/en/mariadb-maxscale-2501/)

**Nouveautés 25.01** :
- Workload Capture
- Workload Replay
- Diff Router

---

## 8️⃣ Clients & Connectors

### URL : [https://mariadb.com/kb/en/connectors/](https://mariadb.com/kb/en/connectors/)

**Public cible** : 🟢 Développeurs

Documentation de tous les **connecteurs officiels** par langage.

#### Connecteurs par langage

| Langage | Connecteur | Documentation | Qualité |
|---------|------------|---------------|---------|
| **C/C++** | MariaDB Connector/C | [KB Connector/C](https://mariadb.com/kb/en/mariadb-connector-c/) | ⭐⭐⭐⭐⭐ |
| **Java** | MariaDB Connector/J | [KB Connector/J](https://mariadb.com/kb/en/about-mariadb-connector-j/) | ⭐⭐⭐⭐⭐ |
| **Python** | MariaDB Connector/Python | [KB Connector/Python](https://mariadb.com/kb/en/mariadb-connector-python/) | ⭐⭐⭐⭐ |
| **Node.js** | MariaDB Connector/Node.js | [KB Connector/Node](https://mariadb.com/kb/en/nodejs-connector/) | ⭐⭐⭐⭐ |
| **ODBC** | MariaDB Connector/ODBC | [KB Connector/ODBC](https://mariadb.com/kb/en/mariadb-connector-odbc/) | ⭐⭐⭐⭐ |
| **R** | MariaDB Connector/R | [KB Connector/R](https://mariadb.com/kb/en/mariadb-connector-r/) | ⭐⭐⭐ |

#### Exemple : MariaDB Connector/J (Java)

**Documentation complète** : [https://mariadb.com/kb/en/about-mariadb-connector-j/](https://mariadb.com/kb/en/about-mariadb-connector-j/)

```java
// Installation via Maven
<dependency>
    <groupId>org.mariadb.jdbc</groupId>
    <artifactId>mariadb-java-client</artifactId>
    <version>3.3.0</version>
</dependency>

// Connexion
String url = "jdbc:mariadb://localhost:3306/mydb";
Connection conn = DriverManager.getConnection(url, "user", "password");

// Requête
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE id = ?"
);
stmt.setInt(1, 42);
ResultSet rs = stmt.executeQuery();
```

---

## 9️⃣ Tools & Utilities

### URL : [https://mariadb.com/kb/en/mariadb-database-client-library-tools/](https://mariadb.com/kb/en/mariadb-database-client-library-tools/)

**Public cible** : 🟡 DBA, DevOps

#### Outils de backup

| Outil | Description | Documentation |
|-------|-------------|---------------|
| **mariadb-dump** | Backup logique (SQL) | [KB mariadb-dump](https://mariadb.com/kb/en/mariadb-dump/) |
| **Mariabackup** | Backup physique (hot backup) | [KB Mariabackup](https://mariadb.com/kb/en/mariabackup/) |
| **mysqldump** | Compatible MySQL (alias) | [KB mysqldump](https://mariadb.com/kb/en/mysqldump/) |

**Mariabackup** (recommandé pour production) :

```bash
# Full backup
mariabackup --backup \
  --target-dir=/backup/full-$(date +%Y%m%d) \
  --user=root \
  --password=xxx

# Prepare backup
mariabackup --prepare \
  --target-dir=/backup/full-20251214

# Restore
mariabackup --copy-back \
  --target-dir=/backup/full-20251214
```

#### Outils client

| Outil | Description | Documentation |
|-------|-------------|---------------|
| **mariadb** | Client CLI interactif | [KB mariadb client](https://mariadb.com/kb/en/mysql-client/) |
| **mariadb-admin** | Administration CLI | [KB mariadb-admin](https://mariadb.com/kb/en/mysqladmin/) |
| **mariadb-upgrade** | Upgrade après migration | [KB mariadb-upgrade](https://mariadb.com/kb/en/mariadb-upgrade/) |
| **mariadb-check** | Vérification tables | [KB mariadb-check](https://mariadb.com/kb/en/mysqlcheck/) |

---

## 🔟 Release Notes & Changelogs

### URL : [https://mariadb.com/kb/en/release-notes/](https://mariadb.com/kb/en/release-notes/)

**Public cible** : 🟡 Tous

**Releases majeures récentes** :

| Version | Date GA | Type | Release Notes |
|---------|---------|------|---------------|
| **11.8.0** | Juin 2025 | LTS | [11.8.0 Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/) |
| **11.4.0** | Mai 2024 | LTS | [11.4.0 Notes](https://mariadb.com/kb/en/mariadb-1140-release-notes/) |
| 11.7.0 | Déc 2024 | Rolling | [11.7.0 Notes](https://mariadb.com/kb/en/mariadb-1170-release-notes/) |
| **10.11.0** | Fév 2023 | LTS | [10.11.0 Notes](https://mariadb.com/kb/en/mariadb-10110-release-notes/) |

**Contenu des Release Notes** :
- ✅ Nouvelles fonctionnalités détaillées
- ✅ Correctifs de bugs
- ✅ Changements de comportement
- ✅ Deprecated features
- ✅ Notes de migration
- ✅ Known issues

**Changelog détaillé** : [https://mariadb.com/kb/en/changes-improvements-in-mariadb-118/](https://mariadb.com/kb/en/changes-improvements-in-mariadb-118/)

---

## 1️⃣1️⃣ API & Protocol Documentation

### MySQL Protocol

**Documentation** : [https://mariadb.com/kb/en/clientserver-protocol/](https://mariadb.com/kb/en/clientserver-protocol/)

**Sections** :
- Connection lifecycle
- Authentication methods
- Command packets
- Result set packets
- Binary protocol (prepared statements)

**Utile pour** :
- Développeurs de connecteurs
- Debugging réseau
- Analyse de performance

---

## 🔍 Utiliser Efficacement la Knowledge Base

### Fonctionnalités de Recherche

#### 1. Recherche par mot-clé

**URL** : [https://mariadb.com/kb/en/](https://mariadb.com/kb/en/)

**Astuces** :
- ✅ Utiliser termes **anglais** (meilleurs résultats)
- ✅ Essayer **synonymes** (e.g., "backup" et "dump")
- ✅ Utiliser **guillemets** pour phrases exactes : `"GTID replication"`
- ✅ Combiner termes : `innodb buffer pool optimization`

#### 2. Navigation par version

Chaque page KB indique **les versions supportées** :

```
Applicable Versions:
✅ MariaDB 11.8 (applies)
✅ MariaDB 11.4 (applies)
✅ MariaDB 10.11 (applies)
⚠️ MariaDB 10.6 (partial support)
❌ MariaDB 10.5 (not available)
```

#### 3. Filtrage par catégorie

**URL structure** : `https://mariadb.com/kb/en/[category]/`

Exemples :
- `/en/sql-statements/` - Toutes commandes SQL
- `/en/built-in-functions/` - Toutes fonctions
- `/en/replication/` - Réplication
- `/en/galera-cluster/` - Galera

#### 4. Table des matières

Chaque page longue a une **table des matières** en haut à droite :

```
On this page:
├─ Syntax
├─ Description
├─ Options
├─ Examples
├─ See Also
└─ External References
```

---

## 📊 Matrice Documentation par Besoin

| Besoin | Documentation Recommandée | Priorité |
|--------|--------------------------|----------|
| **Apprendre MariaDB** | Getting Started + MariaDB Basics | 🔥 |
| **Référence SQL** | SQL Statements + Built-in Functions | 🔥 |
| **Optimiser performance** | InnoDB System Variables + Optimization Guide | ⚡ |
| **Configurer HA** | Galera Cluster + MaxScale docs | ⚡ |
| **Implémenter backup** | Mariabackup + mariadb-dump guides | ⚡ |
| **Développer application** | Connectors + Client/Server Protocol | 📊 |
| **Migrer depuis MySQL** | Migration guides + Compatibility notes | 📊 |
| **Contribuer au code** | Development + Contributing guides | 📊 |

---

## 🌐 Documentation Multi-Langues

### Langues disponibles

| Langue | Couverture | Qualité | URL |
|--------|-----------|---------|-----|
| **Anglais** | 100% | ⭐⭐⭐⭐⭐ | `/en/` |
| **Japonais** | ~40% | ⭐⭐⭐ | `/ja/` |
| **Chinois** | ~30% | ⭐⭐⭐ | `/zh-cn/` |
| **Français** | ~20% | ⭐⭐ | Contributions communautaires |
| **Autres** | <10% | ⭐ | Partiel |

💡 **Recommandation** : Utiliser la version **anglaise** pour avoir la documentation la plus complète et à jour.

---

## 📖 Documentation Hors-Ligne

### Téléchargement

Plusieurs formats disponibles pour consultation hors-ligne :

**PDF** (par version) :
- Généré via `pandoc` ou outils communautaires
- Non officiel mais disponible via communauté

**Git Clone** :
```bash
# Cloner la KB (site statique)
git clone https://github.com/MariaDB/mariadb.org-tools

# Générer version locale
# (instructions dans le README)
```

**Docker** :
```bash
# Serveur KB local (communautaire)
docker run -d -p 8080:80 \
  community/mariadb-kb-mirror:latest
```

---

## 🤝 Contribuer à la Documentation

### Comment contribuer

**URL** : [https://mariadb.com/kb/en/contributing-to-the-mariadb-knowledge-base/](https://mariadb.com/kb/en/contributing-to-the-mariadb-knowledge-base/)

**Processus** :
1. Créer compte MariaDB.com
2. Cliquer "Edit" sur page KB
3. Proposer modifications (Markdown)
4. Soumission pour review
5. Validation par équipe MariaDB

**Types de contributions** :
- ✅ Corrections typos/erreurs
- ✅ Ajout d'exemples
- ✅ Clarification explications
- ✅ Traductions
- ✅ Nouvelles pages (avec approbation)

💡 **Récompenses** : Contributeurs réguliers reconnus officiellement par MariaDB Foundation.

---

## ✅ Points Clés à Retenir

- **MariaDB KB** est la source officielle et autoritaire
- **Recherche en anglais** pour meilleurs résultats
- **Versionné** : vérifier compatibilité avec votre version
- **Structure claire** : Getting Started → Advanced Topics
- **Release Notes** essentielles avant migration
- **Mariabackup** recommandé pour backups production
- **Connectors officiels** disponibles pour 6+ langages
- **Documentation contributive** : vous pouvez améliorer !
- **Navigation par catégorie** plus efficace que recherche parfois
- **Table des matières** dans chaque page pour navigation rapide

---

## 🔗 Liens Rapides Essentiels

### Top 10 Pages KB à Bookmarker

1. 📘 [Knowledge Base Home](https://mariadb.com/kb/en/)
2. 📗 [SQL Statements Reference](https://mariadb.com/kb/en/sql-statements-structure/)
3. 📙 [Built-in Functions](https://mariadb.com/kb/en/built-in-functions/)
4. 📕 [System Variables](https://mariadb.com/kb/en/server-system-variables/)
5. 🆕 [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
6. 🔄 [Replication Guide](https://mariadb.com/kb/en/replication/)
7. 🌐 [Galera Cluster](https://mariadb.com/kb/en/galera-cluster/)
8. 💾 [Mariabackup Guide](https://mariadb.com/kb/en/mariabackup/)
9. 🆕 [Vector Functions](https://mariadb.com/kb/en/vector-functions/)
10. 📊 [Optimization & Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)

---

## ➡️ Sections Suivantes

- **H.2** [Communautés et forums](./02-communautes-forums.md)
- **H.3** [Blogs techniques recommandés](./03-blogs-techniques.md)
- **H.4** [Conférences et événements](./04-conferences-evenements.md)


---

💡 **Conseil final** : Marquer en favoris les pages que vous consultez fréquemment et utiliser la recherche KB comme **premier réflexe** avant de chercher sur Google ou StackOverflow !

⏭️ [Communautés et forums](/annexes/ressources-documentation/02-communautes-forums.md)

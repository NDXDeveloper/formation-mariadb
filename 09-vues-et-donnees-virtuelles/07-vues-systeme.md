🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.7 Vues système

> **Niveau** : Intermédiaire
> **Durée estimée** : 2.5-3 heures
> **Prérequis** : Sections 9.1-9.6, compréhension des vues et métadonnées

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le rôle et l'utilité des vues système dans MariaDB
- Naviguer efficacement dans INFORMATION_SCHEMA pour consulter les métadonnées
- Utiliser PERFORMANCE_SCHEMA pour le monitoring et l'optimisation
- Exploiter les tables système mysql pour la gestion des utilisateurs et privilèges
- Écrire des requêtes d'administration et de diagnostic sur les vues système
- Identifier les cas d'usage appropriés pour chaque type de vue système
- Optimiser l'utilisation des vues système pour minimiser l'impact sur les performances
- Combiner plusieurs vues système pour des analyses complexes

---

## Introduction

### Qu'est-ce que les vues système ?

Les **vues système** (system views) sont des vues spéciales prédéfinies par MariaDB qui exposent les **métadonnées** du serveur : informations sur les bases de données, tables, colonnes, index, utilisateurs, privilèges, performances, processus en cours, etc.

Contrairement aux vues créées par les utilisateurs (sections précédentes), les vues système sont :
- **Automatiquement créées** par MariaDB
- **En lecture seule** (généralement)
- **Toujours disponibles** sur chaque serveur MariaDB
- **Standardisées** (en grande partie) selon les normes SQL

### Pourquoi utiliser les vues système ?

Les vues système sont essentielles pour :

1. **Administration** : Gérer les bases de données, tables, utilisateurs
2. **Monitoring** : Surveiller les performances, les connexions, les requêtes
3. **Diagnostic** : Identifier les problèmes, les requêtes lentes, les verrous
4. **Audit** : Vérifier les privilèges, les accès, les modifications
5. **Optimisation** : Analyser l'utilisation des index, les statistiques de tables
6. **Automatisation** : Créer des scripts d'administration basés sur les métadonnées

```sql
-- Exemple simple : Lister toutes les tables d'une base
SELECT TABLE_NAME, TABLE_ROWS, DATA_LENGTH
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY DATA_LENGTH DESC;

-- Au lieu de mémoriser chaque table, interroger les métadonnées !
```

### Les trois types de vues système

MariaDB expose trois ensembles principaux de vues système :

| Type | Description | Cas d'usage principal |
|------|-------------|----------------------|
| **INFORMATION_SCHEMA** | Métadonnées sur la structure de la base | Administration, audit de schéma |
| **PERFORMANCE_SCHEMA** | Métriques de performance en temps réel | Monitoring, optimisation |
| **mysql (system tables)** | Configuration serveur et privilèges | Gestion utilisateurs, sécurité |

---

## INFORMATION_SCHEMA : Vue d'ensemble

### Qu'est-ce qu'INFORMATION_SCHEMA ?

**INFORMATION_SCHEMA** est une base de données virtuelle qui contient des vues sur les **métadonnées structurelles** de toutes les bases de données du serveur MariaDB.

```sql
-- INFORMATION_SCHEMA est une "base de données" spéciale
SHOW DATABASES;
-- +--------------------+
-- | Database           |
-- +--------------------+
-- | information_schema |  ← Base virtuelle
-- | mysql              |
-- | performance_schema |
-- | ma_base            |
-- +--------------------+

-- Lister les vues disponibles dans INFORMATION_SCHEMA
SHOW TABLES FROM INFORMATION_SCHEMA;
-- Plus de 60 vues disponibles !
```

### Principales vues INFORMATION_SCHEMA

Les vues les plus utilisées :

#### 1. Structure des bases et tables

```sql
-- SCHEMATA : Liste des bases de données
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM INFORMATION_SCHEMA.SCHEMATA;

-- TABLES : Liste des tables et vues
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE, ENGINE, TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'ma_base';

-- COLUMNS : Liste des colonnes
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'ma_base'
  AND TABLE_NAME = 'clients';
```

#### 2. Index et contraintes

```sql
-- STATISTICS : Informations sur les index
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, CARDINALITY
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'ma_base'
  AND TABLE_NAME = 'produits';

-- KEY_COLUMN_USAGE : Clés primaires et étrangères
SELECT TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME, REFERENCED_TABLE_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'ma_base';

-- TABLE_CONSTRAINTS : Toutes les contraintes
SELECT TABLE_NAME, CONSTRAINT_NAME, CONSTRAINT_TYPE
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = 'ma_base';
```

#### 3. Privilèges et utilisateurs

```sql
-- USER_PRIVILEGES : Privilèges globaux
SELECT GRANTEE, PRIVILEGE_TYPE, IS_GRANTABLE
FROM INFORMATION_SCHEMA.USER_PRIVILEGES
WHERE GRANTEE LIKE '%app_user%';

-- TABLE_PRIVILEGES : Privilèges au niveau table
SELECT GRANTEE, TABLE_SCHEMA, TABLE_NAME, PRIVILEGE_TYPE
FROM INFORMATION_SCHEMA.TABLE_PRIVILEGES
WHERE TABLE_SCHEMA = 'ma_base';

-- COLUMN_PRIVILEGES : Privilèges au niveau colonne
SELECT GRANTEE, TABLE_NAME, COLUMN_NAME, PRIVILEGE_TYPE
FROM INFORMATION_SCHEMA.COLUMN_PRIVILEGES;
```

#### 4. Vues et procédures stockées

```sql
-- VIEWS : Liste des vues
SELECT TABLE_NAME, VIEW_DEFINITION, CHECK_OPTION, IS_UPDATABLE
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'ma_base';

-- ROUTINES : Procédures et fonctions
SELECT ROUTINE_NAME, ROUTINE_TYPE, DTD_IDENTIFIER
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = 'ma_base';

-- TRIGGERS : Liste des triggers
SELECT TRIGGER_NAME, EVENT_MANIPULATION, EVENT_OBJECT_TABLE
FROM INFORMATION_SCHEMA.TRIGGERS
WHERE TRIGGER_SCHEMA = 'ma_base';
```

### Exemple pratique : Audit de schéma

```sql
-- Requête complète : Analyse des tables avec statistiques
SELECT
    t.TABLE_NAME,
    t.ENGINE,
    t.TABLE_ROWS,
    ROUND(t.DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(t.INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND((t.DATA_LENGTH + t.INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
    t.AUTO_INCREMENT,
    t.CREATE_TIME,
    t.UPDATE_TIME,
    COUNT(DISTINCT s.INDEX_NAME) AS nb_index
FROM INFORMATION_SCHEMA.TABLES t
LEFT JOIN INFORMATION_SCHEMA.STATISTICS s
    ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
    AND t.TABLE_NAME = s.TABLE_NAME
WHERE t.TABLE_SCHEMA = 'production_db'
  AND t.TABLE_TYPE = 'BASE TABLE'
GROUP BY t.TABLE_NAME
ORDER BY total_mb DESC;

-- Résultat : Vue complète de l'espace disque par table avec nb d'index
```

💡 **Conseil** : INFORMATION_SCHEMA est idéal pour l'administration quotidienne, les audits de structure et la génération de scripts automatisés.

---

## PERFORMANCE_SCHEMA : Vue d'ensemble

### Qu'est-ce que PERFORMANCE_SCHEMA ?

**PERFORMANCE_SCHEMA** est un moteur de stockage spécial qui collecte des **métriques de performance** en temps réel sur le serveur MariaDB : requêtes exécutées, temps d'attente, utilisation des ressources, verrous, etc.

```sql
-- PERFORMANCE_SCHEMA est aussi une "base de données"
USE PERFORMANCE_SCHEMA;

-- Plus de 100 tables de métriques
SHOW TABLES;
-- events_waits_summary_by_instance
-- events_statements_summary_by_digest
-- table_io_waits_summary_by_table
-- ...
```

### Différence avec INFORMATION_SCHEMA

| Aspect | INFORMATION_SCHEMA | PERFORMANCE_SCHEMA |
|--------|-------------------|-------------------|
| **Type de données** | Métadonnées structurelles | Métriques de performance |
| **Contenu** | Tables, colonnes, index, utilisateurs | Requêtes, latence, verrous, I/O |
| **Mise à jour** | Lors des changements de structure | En temps réel (continu) |
| **Usage** | Administration, audit | Monitoring, optimisation |
| **Coût** | Négligeable | Léger impact performance |

### Activation de PERFORMANCE_SCHEMA

```sql
-- Vérifier si PERFORMANCE_SCHEMA est activé
SHOW VARIABLES LIKE 'performance_schema';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | performance_schema | ON    |  ← Doit être ON
-- +--------------------+-------+

-- Si OFF, activer dans my.cnf et redémarrer
-- [mysqld]
-- performance_schema = ON
```

### Principales tables PERFORMANCE_SCHEMA

#### 1. Résumé des requêtes (statements)

```sql
-- events_statements_summary_by_digest : Top requêtes
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_time_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) AS total_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Résultat : Les 10 requêtes qui consomment le plus de temps cumulé
```

#### 2. I/O sur les tables

```sql
-- table_io_waits_summary_by_table : I/O par table
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    SUM_TIMER_READ / 1000000000000 AS total_read_sec,
    SUM_TIMER_WRITE / 1000000000000 AS total_write_sec
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema')
ORDER BY (SUM_TIMER_READ + SUM_TIMER_WRITE) DESC
LIMIT 20;

-- Résultat : Tables avec le plus d'activité I/O
```

#### 3. Verrous (locks)

```sql
-- metadata_locks : Verrous actuels
SELECT
    OBJECT_TYPE,
    OBJECT_SCHEMA,
    OBJECT_NAME,
    LOCK_TYPE,
    LOCK_STATUS,
    OWNER_THREAD_ID
FROM performance_schema.metadata_locks
WHERE OBJECT_SCHEMA = 'production_db';

-- Résultat : Verrous actifs en ce moment
```

#### 4. Threads et connexions

```sql
-- threads : Connexions actives
SELECT
    THREAD_ID,
    NAME,
    TYPE,
    PROCESSLIST_ID,
    PROCESSLIST_USER,
    PROCESSLIST_HOST,
    PROCESSLIST_DB,
    PROCESSLIST_COMMAND,
    PROCESSLIST_STATE
FROM performance_schema.threads
WHERE TYPE = 'FOREGROUND';

-- Résultat : Liste des connexions utilisateurs actives
```

### Exemple pratique : Diagnostic de performance

```sql
-- Identifier les requêtes les plus lentes en moyenne
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec,
    ROUND(MAX_TIMER_WAIT / 1000000000000, 3) AS max_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) AS total_sec,
    FIRST_SEEN,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
  AND SCHEMA_NAME = 'production_db'
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 20;

-- Puis analyser ces requêtes avec EXPLAIN
```

💡 **Conseil** : PERFORMANCE_SCHEMA est indispensable pour le diagnostic de performance et l'optimisation continue.

⚠️ **Attention** : PERFORMANCE_SCHEMA a un **léger impact** sur les performances (généralement < 5%). Pour des environnements ultra-critiques, certains instruments peuvent être désactivés.

---

## Vues système mysql : Vue d'ensemble

### Qu'est-ce que la base mysql ?

La base de données **mysql** contient les **tables système** qui stockent la configuration du serveur, les utilisateurs, les privilèges, les plugins, etc. Contrairement à INFORMATION_SCHEMA et PERFORMANCE_SCHEMA (vues virtuelles), les tables de **mysql** sont de **vraies tables** persistantes sur disque.

```sql
-- Tables principales de la base mysql
USE mysql;

SHOW TABLES;
-- +---------------------------+
-- | Tables_in_mysql           |
-- +---------------------------+
-- | user                      |  ← Utilisateurs et privilèges globaux
-- | db                        |  ← Privilèges par base
-- | tables_priv               |  ← Privilèges par table
-- | columns_priv              |  ← Privilèges par colonne
-- | procs_priv                |  ← Privilèges sur procédures
-- | plugin                    |  ← Plugins installés
-- | servers                   |  ← Serveurs distants
-- | time_zone                 |  ← Fuseaux horaires
-- | ...                       |
-- +---------------------------+
```

### Principales tables mysql

#### 1. Gestion des utilisateurs

```sql
-- mysql.user : Utilisateurs et privilèges globaux
SELECT
    User,
    Host,
    Select_priv,
    Insert_priv,
    Update_priv,
    Delete_priv,
    Create_priv,
    Drop_priv,
    Reload_priv,
    Shutdown_priv,
    Super_priv,
    authentication_string
FROM mysql.user;

-- Vérifier les privilèges d'un utilisateur spécifique
SELECT * FROM mysql.user WHERE User = 'app_user'\G
```

#### 2. Privilèges par base/table

```sql
-- mysql.db : Privilèges au niveau base de données
SELECT User, Host, Db, Select_priv, Insert_priv, Update_priv
FROM mysql.db
WHERE Db = 'production_db';

-- mysql.tables_priv : Privilèges au niveau table
SELECT User, Host, Db, Table_name, Table_priv, Column_priv
FROM mysql.tables_priv
WHERE Db = 'production_db';

-- mysql.columns_priv : Privilèges au niveau colonne
SELECT User, Host, Db, Table_name, Column_name, Column_priv
FROM mysql.columns_priv;
```

#### 3. Plugins et configuration

```sql
-- mysql.plugin : Plugins installés
SELECT name, dl, load_option
FROM mysql.plugin;

-- mysql.servers : Serveurs distants (FederatedX, Spider)
SELECT Server_name, Host, Db, Username, Port, Wrapper
FROM mysql.servers;
```

### Différence avec INFORMATION_SCHEMA

```sql
-- INFORMATION_SCHEMA : Vue abstraite des privilèges
SELECT * FROM INFORMATION_SCHEMA.USER_PRIVILEGES
WHERE GRANTEE LIKE '%app_user%';

-- mysql.user : Table physique avec les privilèges
SELECT * FROM mysql.user WHERE User = 'app_user';

-- ✅ INFORMATION_SCHEMA est plus facile à interroger (standardisé)
-- ⚠️ mysql.* est plus bas niveau (format MariaDB spécifique)
```

💡 **Recommandation** : Privilégiez **INFORMATION_SCHEMA** pour la lecture des métadonnées (plus portable), et réservez **mysql.*** pour l'administration système avancée.

⚠️ **Avertissement** : Ne modifiez **jamais directement** les tables mysql.user, mysql.db, etc. Utilisez toujours GRANT/REVOKE qui mettent à jour ces tables de manière sécurisée.

```sql
-- ❌ NE JAMAIS FAIRE
UPDATE mysql.user SET Super_priv = 'Y' WHERE User = 'app_user';
-- Risque de corruption, privilèges incohérents

-- ✅ TOUJOURS UTILISER
GRANT SUPER ON *.* TO 'app_user'@'%';
-- MariaDB gère la cohérence automatiquement
```

---

## Cas d'usage combinés des vues système

Les vues système sont souvent utilisées **ensemble** pour des analyses complexes.

### Cas 1 : Audit complet des privilèges d'un utilisateur

```sql
-- Combiner INFORMATION_SCHEMA et mysql.user
SELECT
    'Global' AS niveau,
    PRIVILEGE_TYPE AS privilege,
    '*.*' AS objet
FROM INFORMATION_SCHEMA.USER_PRIVILEGES
WHERE GRANTEE = "'app_user'@'%'"

UNION ALL

SELECT
    'Database' AS niveau,
    PRIVILEGE_TYPE,
    CONCAT(TABLE_SCHEMA, '.*') AS objet
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
WHERE GRANTEE = "'app_user'@'%'"

UNION ALL

SELECT
    'Table' AS niveau,
    PRIVILEGE_TYPE,
    CONCAT(TABLE_SCHEMA, '.', TABLE_NAME) AS objet
FROM INFORMATION_SCHEMA.TABLE_PRIVILEGES
WHERE GRANTEE = "'app_user'@'%'"

ORDER BY niveau, objet, privilege;

-- Résultat : Liste complète des privilèges de l'utilisateur
```

### Cas 2 : Identifier les tables sans index

```sql
-- Combiner INFORMATION_SCHEMA.TABLES et STATISTICS
SELECT
    t.TABLE_NAME,
    t.TABLE_ROWS,
    ROUND((t.DATA_LENGTH + t.INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM INFORMATION_SCHEMA.TABLES t
LEFT JOIN INFORMATION_SCHEMA.STATISTICS s
    ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
    AND t.TABLE_NAME = s.TABLE_NAME
WHERE t.TABLE_SCHEMA = 'production_db'
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND s.INDEX_NAME IS NULL  -- Aucun index trouvé
ORDER BY t.TABLE_ROWS DESC;

-- Résultat : Tables sans aucun index (problème de performance potentiel)
```

### Cas 3 : Analyser les requêtes lentes sur des tables spécifiques

```sql
-- Combiner PERFORMANCE_SCHEMA et INFORMATION_SCHEMA
SELECT
    t.TABLE_NAME,
    t.TABLE_ROWS,
    tio.COUNT_READ,
    tio.COUNT_WRITE,
    ROUND(tio.SUM_TIMER_WAIT / 1000000000000, 3) AS total_io_sec
FROM INFORMATION_SCHEMA.TABLES t
INNER JOIN performance_schema.table_io_waits_summary_by_table tio
    ON t.TABLE_SCHEMA = tio.OBJECT_SCHEMA
    AND t.TABLE_NAME = tio.OBJECT_NAME
WHERE t.TABLE_SCHEMA = 'production_db'
  AND t.TABLE_TYPE = 'BASE TABLE'
ORDER BY tio.SUM_TIMER_WAIT DESC
LIMIT 20;

-- Résultat : Tables avec le plus d'activité I/O
```

### Cas 4 : Vérifier la cohérence des foreign keys

```sql
-- Trouver les FK qui pointent vers des tables inexistantes
SELECT
    kcu.TABLE_NAME,
    kcu.COLUMN_NAME,
    kcu.CONSTRAINT_NAME,
    kcu.REFERENCED_TABLE_NAME,
    kcu.REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
LEFT JOIN INFORMATION_SCHEMA.TABLES t
    ON kcu.REFERENCED_TABLE_SCHEMA = t.TABLE_SCHEMA
    AND kcu.REFERENCED_TABLE_NAME = t.TABLE_NAME
WHERE kcu.TABLE_SCHEMA = 'production_db'
  AND kcu.REFERENCED_TABLE_NAME IS NOT NULL
  AND t.TABLE_NAME IS NULL;  -- Table référencée n'existe pas !

-- Résultat : FK orphelines (erreur de configuration)
```

---

## Performance et bonnes pratiques

### Impact sur les performances

Les requêtes sur les vues système ont un **coût** :

```sql
-- ⚠️ Requête coûteuse : INFORMATION_SCHEMA scanne toutes les bases
SELECT * FROM INFORMATION_SCHEMA.TABLES;
-- Peut prendre plusieurs secondes si beaucoup de bases/tables

-- ✅ Requête optimisée : Filtrer par base
SELECT * FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'production_db';
-- Beaucoup plus rapide (filtre précoce)
```

### Bonnes pratiques

#### 1. Toujours filtrer par TABLE_SCHEMA

```sql
-- ❌ Lent
SELECT * FROM INFORMATION_SCHEMA.COLUMNS;

-- ✅ Rapide
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'ma_base';
```

#### 2. Utiliser des index quand possible

```sql
-- Les vues INFORMATION_SCHEMA ont des "index virtuels"
-- Filtrer par TABLE_SCHEMA, TABLE_NAME utilise ces index

-- ✅ Optimisé
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'ma_base'
  AND TABLE_NAME = 'clients';
```

#### 3. Éviter SELECT * sur PERFORMANCE_SCHEMA

```sql
-- ❌ Très lent (trop de données)
SELECT * FROM performance_schema.events_statements_history_long;

-- ✅ Sélectionner uniquement les colonnes nécessaires
SELECT DIGEST_TEXT, TIMER_WAIT, LOCK_TIME
FROM performance_schema.events_statements_history_long
WHERE DIGEST_TEXT LIKE '%SELECT%production_db%'
LIMIT 100;
```

#### 4. Désactiver les instruments inutilisés dans PERFORMANCE_SCHEMA

```sql
-- Désactiver un instrument spécifique
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO'
WHERE NAME LIKE 'wait/io/file%';

-- Vérifier les instruments actifs
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE ENABLED = 'YES';
```

#### 5. Mettre en cache les résultats

```sql
-- Au lieu de requêter INFORMATION_SCHEMA à chaque fois,
-- créer une table de cache

CREATE TABLE cache_table_metadata AS
SELECT
    TABLE_NAME,
    TABLE_ROWS,
    DATA_LENGTH,
    INDEX_LENGTH,
    CREATE_TIME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'production_db';

-- Rafraîchir quotidiennement avec un EVENT
CREATE EVENT evt_refresh_metadata
ON SCHEDULE EVERY 1 DAY
DO
    REPLACE INTO cache_table_metadata
    SELECT ... FROM INFORMATION_SCHEMA.TABLES ...;
```

---

## Requêtes d'administration courantes

### 1. Trouver les tables volumineuses

```sql
SELECT
    TABLE_NAME,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 20;
```

### 2. Lister les vues et leur définition

```sql
SELECT
    TABLE_NAME,
    IS_UPDATABLE,
    CHECK_OPTION,
    ALGORITHM
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY TABLE_NAME;
```

### 3. Identifier les colonnes sans index (potentiel problème)

```sql
SELECT
    c.TABLE_NAME,
    c.COLUMN_NAME,
    c.DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS c
LEFT JOIN INFORMATION_SCHEMA.STATISTICS s
    ON c.TABLE_SCHEMA = s.TABLE_SCHEMA
    AND c.TABLE_NAME = s.TABLE_NAME
    AND c.COLUMN_NAME = s.COLUMN_NAME
WHERE c.TABLE_SCHEMA = 'production_db'
  AND s.INDEX_NAME IS NULL
  AND c.COLUMN_NAME LIKE '%_id'  -- Colonnes qui devraient avoir un index
ORDER BY c.TABLE_NAME, c.COLUMN_NAME;
```

### 4. Vérifier la fragmentation des tables

```sql
SELECT
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS fragmentation_mb,
    ROUND(DATA_FREE / DATA_LENGTH * 100, 2) AS fragmentation_pct
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'production_db'
  AND DATA_FREE > 0
ORDER BY DATA_FREE DESC;

-- Si fragmentation > 20%, envisager OPTIMIZE TABLE
```

### 5. Lister les processus en cours

```sql
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE,
    LEFT(INFO, 100) AS QUERY_PREVIEW
FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND TIME > 5  -- Plus de 5 secondes
ORDER BY TIME DESC;

-- Identifier les requêtes qui durent trop longtemps
```

---

## Structure de cette section

Cette section 9.7 est organisée en trois sous-sections détaillées :

### 9.7.1 INFORMATION_SCHEMA

Exploration approfondie de toutes les vues importantes d'INFORMATION_SCHEMA :
- TABLES, COLUMNS, STATISTICS pour la structure
- VIEWS, ROUTINES, TRIGGERS pour les objets
- *_PRIVILEGES pour la sécurité
- Requêtes pratiques pour chaque vue

### 9.7.2 PERFORMANCE_SCHEMA

Utilisation avancée de PERFORMANCE_SCHEMA pour le monitoring :
- Configuration et activation des instruments
- Analyse des requêtes lentes
- Monitoring de l'I/O et des verrous
- Diagnostic de performance en production

### 9.7.3 mysql (system tables)

Tables système mysql pour l'administration :
- Gestion des utilisateurs et privilèges
- Configuration des plugins
- Tables de configuration avancées
- Maintenance des tables système

---

## ✅ Points clés à retenir

- Les **vues système** exposent les métadonnées du serveur MariaDB
- **INFORMATION_SCHEMA** : Métadonnées structurelles (tables, colonnes, index, privilèges)
- **PERFORMANCE_SCHEMA** : Métriques de performance en temps réel (requêtes, I/O, verrous)
- **mysql** : Tables système physiques (utilisateurs, privilèges, configuration)
- Toujours **filtrer par TABLE_SCHEMA** pour optimiser les requêtes sur INFORMATION_SCHEMA
- Les vues système sont **essentielles** pour l'administration, le monitoring et le diagnostic
- Combiner plusieurs vues système pour des analyses complexes
- INFORMATION_SCHEMA est **standardisé** (portable), mysql.* est **spécifique MariaDB**
- Ne **jamais modifier directement** les tables mysql.user/db (utiliser GRANT/REVOKE)
- PERFORMANCE_SCHEMA a un **léger impact** sur les performances (< 5%)
- Mettre en **cache** les résultats des requêtes fréquentes sur INFORMATION_SCHEMA

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 INFORMATION_SCHEMA](https://mariadb.com/kb/en/information-schema/) - Documentation complète
- [📖 PERFORMANCE_SCHEMA](https://mariadb.com/kb/en/performance-schema/) - Guide d'utilisation
- [📖 MySQL System Tables](https://mariadb.com/kb/en/the-mysql-database-tables/) - Tables système
- [📖 System Variables](https://mariadb.com/kb/en/server-system-variables/) - Variables de configuration

### Articles et guides
- **"INFORMATION_SCHEMA Best Practices"** - MariaDB Blog
- **"Performance Schema for DBAs"** - Percona Blog
- **"Monitoring MariaDB with PERFORMANCE_SCHEMA"** - Database Journal

### Outils complémentaires
- **pt-query-digest** - Analyse des requêtes (utilise PERFORMANCE_SCHEMA)
- **MySQLTuner** - Analyse de configuration (interroge INFORMATION_SCHEMA)
- **PMM (Percona Monitoring and Management)** - Monitoring complet

---

## ➡️ Sous-sections suivantes

- **[9.7.1 INFORMATION_SCHEMA](./07.1-information-schema.md)** : Exploration détaillée de toutes les vues INFORMATION_SCHEMA avec requêtes pratiques pour chaque cas d'usage
- **[9.7.2 PERFORMANCE_SCHEMA](./07.2-performance-schema.md)** : Utilisation avancée de PERFORMANCE_SCHEMA pour le monitoring et l'optimisation en production
- **[9.7.3 mysql (system tables)](./07.3-mysql-system-tables.md)** : Gestion des utilisateurs, privilèges et configuration via les tables système mysql

Chaque sous-section approfondira les concepts introduits ici avec des exemples pratiques et des cas d'usage réels.

---


⏭️ [INFORMATION_SCHEMA](/09-vues-et-donnees-virtuelles/07.1-information-schema.md)

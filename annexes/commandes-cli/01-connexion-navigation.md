🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.1 - Connexion et Navigation

> **Niveau** : Débutant à Intermédiaire  
> **Durée estimée** : 30 minutes  
> **Prérequis** : MariaDB installé, accès serveur

---

## 📖 Introduction

Cette section couvre les **commandes essentielles** pour se connecter à un serveur MariaDB et naviguer dans la structure des bases de données. Ces commandes constituent la base de toute session de travail en ligne de commande.

### 🎯 Objectifs

À l'issue de cette section, vous serez capable de :
- Vous connecter à un serveur MariaDB (local ou distant)
- Vérifier l'état de votre connexion
- Naviguer entre les bases de données
- Lister et explorer la structure (bases, tables, colonnes)
- Obtenir des informations détaillées sur les objets

---

## 🔌 Connexion au Serveur

### Syntaxe de Base

```bash
mariadb [options] [database]
```

### Connexion Locale Simple

```bash
# Connexion en tant que root (demande mot de passe)
mariadb -u root -p

# Connexion avec utilisateur spécifique
mariadb -u myuser -p

# Connexion directe à une base
mariadb -u myuser -p mydatabase
```

💡 **Bonne pratique** : Toujours utiliser `-p` seul (sans mot de passe) pour que le système demande le mot de passe de manière sécurisée.

### Connexion Distante

```bash
# Connexion à un serveur distant
mariadb -h db.example.com -u myuser -p

# Spécifier le port (si différent de 3306)
mariadb -h db.example.com -P 3307 -u myuser -p

# Connexion complète avec base de données
mariadb -h 192.168.1.100 -P 3306 -u myuser -p production_db
```

### Connexion avec SSL/TLS

```bash
# Connexion sécurisée SSL/TLS
mariadb -h db.example.com -u myuser -p --ssl

# Avec certificat CA spécifique
mariadb -h db.example.com -u myuser -p \
  --ssl-ca=/path/to/ca.pem \
  --ssl-cert=/path/to/client-cert.pem \
  --ssl-key=/path/to/client-key.pem

# Vérification stricte du certificat
mariadb -h db.example.com -u myuser -p \
  --ssl-verify-server-cert
```

🔒 **Sécurité** : Toujours utiliser SSL/TLS pour connexions sur réseau non sécurisé.

### Connexion via Socket Unix

```bash
# Connexion via socket (plus rapide en local)
mariadb -u root -p --socket=/var/run/mysqld/mysqld.sock

# Socket par défaut (généralement /var/run/mysqld/mysqld.sock)
mariadb -u root -p
```

### Options de Connexion Avancées

```bash
# Connexion avec timeout personnalisé
mariadb -u myuser -p --connect-timeout=10

# Désactiver auto-complétion (plus rapide sur grandes bases)
mariadb -u myuser -p -A

# Mode batch (sans formatage, pour scripts)
mariadb -u myuser -p -B

# Connexion et exécution d'une commande unique
mariadb -u myuser -p -e "SELECT VERSION();"

# Connexion avec charset spécifique
mariadb -u myuser -p --default-character-set=utf8mb4
```

---

## 📊 Vérification de la Connexion

### Commande \s (Status)

La méta-commande `\s` affiche le statut complet de la connexion :

```sql
\s
-- ou équivalent :
STATUS;
```

**Sortie exemple** :
```
--------------
mariadb  Ver 15.1 Distrib 11.8.0-MariaDB, for Linux (x86_64)

Connection id:          42
Current database:       production_db
Current user:           myuser@localhost
SSL:                    Cipher in use is TLS_AES_256_GCM_SHA384
Current pager:          less
Using outfile:          ''
Using delimiter:        ;
Server:                 MariaDB
Server version:         11.8.0-MariaDB MariaDB Server
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8mb4
Conn.  characterset:    utf8mb4
UNIX socket:            /var/run/mysqld/mysqld.sock
Uptime:                 5 days 3 hours 42 min 18 sec

Threads: 8  Questions: 152834  Slow queries: 12  Opens: 234
Flush tables: 3  Open tables: 64  Queries per second avg: 0.345
--------------
```

### Informations Clés du Status

| Information | Description |
|-------------|-------------|
| **Connection id** | Identifiant unique de votre session |
| **Current database** | Base de données active |
| **Current user** | Utilisateur connecté + hostname |
| **SSL** | Statut chiffrement (cipher utilisé) |
| **Server version** | Version exacte de MariaDB |
| **Connection** | Type de connexion (TCP, socket, pipe) |
| **Characterset** | Encodage serveur/client/connexion |
| **Uptime** | Temps depuis démarrage serveur |
| **Threads** | Nombre de connexions actives |

💡 **Usage** : Exécuter `\s` au début de chaque session pour vérifier la connexion et la base active.

### Version du Serveur

```sql
-- Méthode 1 : Fonction VERSION()
SELECT VERSION();
-- Résultat : 11.8.0-MariaDB

-- Méthode 2 : Variable système
SELECT @@version;
-- Résultat : 11.8.0-MariaDB

-- Méthode 3 : Via ligne de commande
mariadb --version
# Résultat : mariadb  Ver 15.1 Distrib 11.8.0-MariaDB
```

### Utilisateur Actuel

```sql
-- Utilisateur et hostname
SELECT USER();
-- Résultat : myuser@localhost

-- Utilisateur authentifié
SELECT CURRENT_USER();
-- Résultat : myuser@%

-- Toutes les infos
SELECT USER(), CURRENT_USER(), DATABASE(), VERSION();
```

---

## 🗂️ Navigation entre Bases de Données

### Lister les Bases de Données

```sql
-- Lister toutes les bases accessibles
SHOW DATABASES;

-- Exemple de sortie :
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| production_db      |
| test_db            |
+--------------------+
```

### Filtrer les Bases de Données

```sql
-- Bases commençant par 'prod'
SHOW DATABASES LIKE 'prod%';

-- Bases contenant 'test'
SHOW DATABASES LIKE '%test%';

-- Avec expression régulière (MariaDB 10.0.5+)
SHOW DATABASES WHERE `Database` REGEXP '^(prod|dev)';
```

### Méthode Alternative (INFORMATION_SCHEMA)

```sql
-- Lister via INFORMATION_SCHEMA
SELECT SCHEMA_NAME 
FROM information_schema.SCHEMATA
ORDER BY SCHEMA_NAME;

-- Avec taille des bases
SELECT 
  SCHEMA_NAME AS 'Database',
  ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.TABLES
GROUP BY SCHEMA_NAME
ORDER BY SUM(DATA_LENGTH + INDEX_LENGTH) DESC;

-- Exemple de sortie :
+--------------------+-----------+
| Database           | Size (MB) |
+--------------------+-----------+
| production_db      |   2543.45 |
| test_db            |    156.78 |
| mysql              |      5.23 |
+--------------------+-----------+
```

### Changer de Base de Données

#### Méthode 1 : USE (SQL Standard)

```sql
-- Passer à une base spécifique
USE production_db;
-- Résultat : Database changed

-- Vérifier la base active
SELECT DATABASE();
-- Résultat : production_db
```

#### Méthode 2 : \u (Méta-commande)

```sql
-- Raccourci pour USE
\u production_db
-- Résultat : Database changed

-- Équivalent à :
USE production_db;
```

💡 **Avantage de \u** : Plus rapide à taper, pas besoin de `;`

#### Méthode 3 : À la Connexion

```bash
# Spécifier la base lors de la connexion
mariadb -u myuser -p production_db

# Ou avec option -D
mariadb -u myuser -p -D production_db
```

### Base de Données Non Existante

```sql
-- Tentative d'utiliser base inexistante
USE nonexistent_db;
-- Erreur : ERROR 1049 (42000): Unknown database 'nonexistent_db'

-- Vérifier l'existence avant
SELECT COUNT(*) 
FROM information_schema.SCHEMATA 
WHERE SCHEMA_NAME = 'production_db';
-- Résultat : 1 (existe) ou 0 (n'existe pas)
```

---

## 📋 Explorer les Tables

### Lister les Tables

```sql
-- Lister toutes les tables de la base courante
SHOW TABLES;

-- Exemple de sortie :
+-------------------------+
| Tables_in_production_db |
+-------------------------+
| customers               |
| orders                  |
| order_items             |
| products                |
| users                   |
+-------------------------+
```

### Lister Tables d'une Base Spécifique

```sql
-- Sans changer de base
SHOW TABLES FROM production_db;

-- ou
SHOW TABLES IN production_db;

-- Équivalent avec INFORMATION_SCHEMA
SELECT TABLE_NAME 
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY TABLE_NAME;
```

### Filtrer les Tables

```sql
-- Tables commençant par 'order'
SHOW TABLES LIKE 'order%';
-- Résultat :
+-------------------------+
| Tables_in_production_db |
+-------------------------+
| orders                  |
| order_items             |
+-------------------------+

-- Tables contenant 'user'
SHOW TABLES LIKE '%user%';

-- Expression régulière
SHOW TABLES WHERE Tables_in_production_db REGEXP '^(customer|product)';
```

### Tables avec Détails

```sql
-- Afficher type de table (BASE TABLE, VIEW, SYSTEM VIEW)
SHOW FULL TABLES;

-- Exemple de sortie :
+-------------------------+------------+
| Tables_in_production_db | Table_type |
+-------------------------+------------+
| customers               | BASE TABLE |
| orders                  | BASE TABLE |
| order_summary           | VIEW       |
| products                | BASE TABLE |
+-------------------------+------------+

-- Seulement les tables (pas les vues)
SHOW FULL TABLES WHERE Table_type = 'BASE TABLE';

-- Seulement les vues
SHOW FULL TABLES WHERE Table_type = 'VIEW';
```

### Informations Détaillées via INFORMATION_SCHEMA

```sql
-- Statistiques complètes sur les tables
SELECT 
  TABLE_NAME AS 'Table',
  ENGINE AS 'Engine',
  TABLE_ROWS AS 'Rows (approx)',
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Size (MB)',
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS 'Data (MB)',
  ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS 'Index (MB)',
  TABLE_COLLATION AS 'Collation'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;

-- Exemple de sortie :
+-------------+--------+---------------+-----------+-----------+------------+--------------------+
| Table       | Engine | Rows (approx) | Size (MB) | Data (MB) | Index (MB) | Collation          |
+-------------+--------+---------------+-----------+-----------+------------+--------------------+
| orders      | InnoDB |      1250000  |    345.67 |    234.12 |     111.55 | utf8mb4_unicode_ci |
| customers   | InnoDB |       500000  |    123.45 |     89.23 |      34.22 | utf8mb4_unicode_ci |
| products    | InnoDB |        25000  |     45.23 |     32.11 |      13.12 | utf8mb4_unicode_ci |
+-------------+--------+---------------+-----------+-----------+------------+--------------------+
```

---

## 🔍 Explorer la Structure des Tables

### DESCRIBE / DESC

Affiche la structure d'une table :

```sql
-- Méthode 1 : DESCRIBE
DESCRIBE customers;

-- Méthode 2 : DESC (raccourci)
DESC customers;

-- Méthode 3 : SHOW COLUMNS
SHOW COLUMNS FROM customers;

-- Exemple de sortie :
+------------+--------------+------+-----+---------------------+----------------+
| Field      | Type         | Null | Key | Default             | Extra          |
+------------+--------------+------+-----+---------------------+----------------+
| id         | int(11)      | NO   | PRI | NULL                | auto_increment |
| name       | varchar(100) | NO   |     | NULL                |                |
| email      | varchar(255) | NO   | UNI | NULL                |                |
| created_at | timestamp    | NO   |     | current_timestamp() |                |
| status     | enum(...)    | NO   | MUL | active              |                |
+------------+--------------+------+-----+---------------------+----------------+
```

### Explication des Colonnes

| Colonne | Description |
|---------|-------------|
| **Field** | Nom de la colonne |
| **Type** | Type de données (INT, VARCHAR, etc.) |
| **Null** | YES si NULL autorisé, NO sinon |
| **Key** | PRI (Primary Key), UNI (Unique), MUL (Index) |
| **Default** | Valeur par défaut |
| **Extra** | Informations supplémentaires (auto_increment, etc.) |

### DESCRIBE avec Pattern

```sql
-- Colonnes commençant par 'created'
SHOW COLUMNS FROM customers LIKE 'created%';

-- Colonnes contenant 'date'
SHOW COLUMNS FROM customers LIKE '%date%';
```

### SHOW CREATE TABLE

Affiche la commande SQL complète de création de la table :

```sql
SHOW CREATE TABLE customers;

-- Sortie formatée (utiliser \G pour verticalité)
SHOW CREATE TABLE customers\G

-- Exemple de sortie :
*************************** 1. row ***************************
       Table: customers
Create Table: CREATE TABLE `customers` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  `email` varchar(255) NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT current_timestamp(),
  `updated_at` timestamp NULL DEFAULT NULL ON UPDATE current_timestamp(),
  `status` enum('active','inactive','suspended') NOT NULL DEFAULT 'active',
  PRIMARY KEY (`id`),
  UNIQUE KEY `email` (`email`),
  KEY `idx_status` (`status`),
  KEY `idx_created` (`created_at`)
) ENGINE=InnoDB AUTO_INCREMENT=50001 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

💡 **Utilité** : Excellent pour :
- Copier la structure d'une table
- Comprendre les contraintes et index
- Documenter le schéma

### SHOW CREATE TABLE pour Autres Objets

```sql
-- Structure d'une vue
SHOW CREATE VIEW order_summary\G

-- Structure d'une procédure stockée
SHOW CREATE PROCEDURE calculate_total\G

-- Structure d'une fonction
SHOW CREATE FUNCTION get_discount\G

-- Structure d'un trigger
SHOW CREATE TRIGGER update_stock\G
```

---

## 📇 Explorer les Index

### Lister les Index d'une Table

```sql
-- Méthode 1 : SHOW INDEX
SHOW INDEX FROM customers;

-- Méthode 2 : SHOW KEYS (équivalent)
SHOW KEYS FROM customers;

-- Exemple de sortie :
+-----------+------------+-------------+--------------+-------------+
| Table     | Non_unique | Key_name    | Seq_in_index | Column_name |
+-----------+------------+-------------+--------------+-------------+
| customers |          0 | PRIMARY     |            1 | id          |
| customers |          0 | email       |            1 | email       |
| customers |          1 | idx_status  |            1 | status      |
| customers |          1 | idx_created |            1 | created_at  |
+-----------+------------+-------------+--------------+-------------+
```

### Index avec Détails Complets

```sql
-- Format vertical pour lisibilité
SHOW INDEX FROM customers\G

-- Exemple de sortie :
*************************** 1. row ***************************
        Table: customers
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 50000
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
      Visible: YES
*************************** 2. row ***************************
        Table: customers
   Non_unique: 0
     Key_name: email
 Seq_in_index: 1
  Column_name: email
    Collation: A
  Cardinality: 49856
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
      Visible: YES
```

### Informations Clés des Index

| Colonne | Description |
|---------|-------------|
| **Non_unique** | 0 = Unique, 1 = Non unique |
| **Key_name** | Nom de l'index |
| **Seq_in_index** | Position colonne dans index composite |
| **Column_name** | Nom de la colonne indexée |
| **Cardinality** | Nombre estimé de valeurs uniques |
| **Index_type** | Type d'index (BTREE, HASH, FULLTEXT, SPATIAL) |
| **Visible** | YES si index visible, NO si invisible |

### Index via INFORMATION_SCHEMA

```sql
-- Statistiques détaillées des index
SELECT 
  INDEX_NAME,
  COLUMN_NAME,
  SEQ_IN_INDEX,
  NON_UNIQUE,
  INDEX_TYPE,
  CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'production_db'
  AND TABLE_NAME = 'customers'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

---

## 🎯 Scénarios Pratiques Courants

### Scénario 1 : Connexion et Exploration Initiale

```bash
# 1. Connexion au serveur
mariadb -u admin -p

# 2. Vérifier la connexion
\s

# 3. Lister les bases disponibles
SHOW DATABASES;

# 4. Sélectionner une base
USE production_db;

# 5. Voir les tables
SHOW TABLES;

# 6. Explorer une table spécifique
DESC customers;
```

### Scénario 2 : Inspection Rapide d'une Base

```sql
-- 1. Changer de base
USE production_db;

-- 2. Compter les tables
SELECT COUNT(*) FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE();
-- Résultat : 15

-- 3. Lister tables avec nombre de lignes
SELECT 
  TABLE_NAME,
  TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_ROWS DESC;

-- 4. Voir les plus grosses tables
SELECT 
  TABLE_NAME,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 10;
```

### Scénario 3 : Trouver une Table ou Colonne

```sql
-- Trouver tables contenant 'order'
SHOW TABLES LIKE '%order%';

-- Trouver toutes les tables avec colonne 'user_id'
SELECT DISTINCT TABLE_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'production_db'
  AND COLUMN_NAME = 'user_id';

-- Trouver toutes les colonnes 'email' (toutes bases)
SELECT 
  TABLE_SCHEMA,
  TABLE_NAME,
  COLUMN_NAME,
  DATA_TYPE
FROM information_schema.COLUMNS
WHERE COLUMN_NAME = 'email'
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

### Scénario 4 : Audit de Sécurité Rapide

```sql
-- 1. Vérifier l'utilisateur actuel
SELECT USER(), CURRENT_USER();

-- 2. Voir les privilèges
SHOW GRANTS;
-- ou
SHOW GRANTS FOR CURRENT_USER();

-- 3. Lister les bases accessibles
SHOW DATABASES;

-- 4. Vérifier SSL
\s
-- Regarder la ligne "SSL: ..."

-- 5. Voir les autres utilisateurs connectés
SHOW PROCESSLIST;
```

### Scénario 5 : Documentation d'une Base

```sql
-- 1. Générer liste des tables avec description
SELECT 
  TABLE_NAME AS 'Table',
  ENGINE AS 'Engine',
  TABLE_ROWS AS 'Rows',
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Size (MB)',
  TABLE_COMMENT AS 'Comment'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY TABLE_NAME;

-- 2. Pour chaque table importante, générer CREATE TABLE
SHOW CREATE TABLE customers\G
SHOW CREATE TABLE orders\G
SHOW CREATE TABLE products\G

-- 3. Lister les vues
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- 4. Lister les procédures stockées
SHOW PROCEDURE STATUS WHERE Db = 'production_db'\G

-- 5. Lister les fonctions
SHOW FUNCTION STATUS WHERE Db = 'production_db'\G
```

---

## 💡 Astuces et Bonnes Pratiques

### Raccourcis Utiles

```sql
-- Raccourci pour DATABASE() actuelle
SELECT DATABASE();

-- Tout sur une table en une commande
SELECT * FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = DATABASE() 
  AND TABLE_NAME = 'customers'\G

-- Affichage vertical automatique pour résultats larges
\G au lieu de ;

-- Exemple :
SHOW VARIABLES LIKE 'innodb%'\G
```

### Auto-complétion

```bash
# Active par défaut
# TAB pour compléter :
USE prod<TAB>          # Complète production_db
SELECT * FROM cust<TAB> # Complète customers

# Désactiver pour performance (grandes bases)
mariadb -A -u myuser -p

# Recharger complétion en session
rehash;
```

### Historique de Navigation

```sql
-- Le client CLI garde trace de vos commandes
-- Flèche haut/bas pour naviguer
-- Ctrl+R pour rechercher dans l'historique

-- Historique stocké dans :
-- ~/.mariadb_history
-- ou ~/.mysql_history
```

### Prompt Personnalisé

```sql
-- Afficher la base courante dans le prompt
prompt \u@\h [\d]>\_

-- Résultat :
admin@localhost [production_db]>

-- Options de prompt :
-- \u : user
-- \h : host
-- \d : database
-- \v : version
-- \D : date complète
```

### Commandes Système

```sql
-- Exécuter commande shell (Linux/macOS)
\! pwd
\! ls -l /var/lib/mysql

-- Effacer l'écran
\! clear
-- ou sous Windows :
\! cls

-- Éditer dans éditeur externe
\e
-- Ouvre $EDITOR (vim, nano, etc.)
```

---

## 🔍 Commandes SHOW Avancées

### Lister Tous les Objets

```sql
-- Toutes les procédures stockées
SHOW PROCEDURE STATUS;

-- Procédures d'une base spécifique
SHOW PROCEDURE STATUS WHERE Db = 'production_db';

-- Toutes les fonctions
SHOW FUNCTION STATUS;

-- Tous les triggers
SHOW TRIGGERS;

-- Triggers d'une table spécifique
SHOW TRIGGERS LIKE 'customers';

-- Tous les events (tâches planifiées)
SHOW EVENTS;
```

### Informations Charset et Collation

```sql
-- Charsets disponibles
SHOW CHARACTER SET;

-- Collations disponibles
SHOW COLLATION;

-- Collations pour utf8mb4
SHOW COLLATION LIKE 'utf8mb4%';

-- Charset et collation par défaut
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
```

### Informations Utilisateurs

```sql
-- Privilèges de l'utilisateur courant
SHOW GRANTS;

-- Privilèges d'un utilisateur spécifique
SHOW GRANTS FOR 'myuser'@'localhost';

-- Tous les utilisateurs (depuis mysql.user)
SELECT User, Host FROM mysql.user;
```

---

## ⚠️ Erreurs Courantes et Solutions

### Erreur : "No database selected"

```sql
-- Problème :
SELECT * FROM customers;
-- ERROR 1046 (3D000): No database selected

-- Solution :
USE production_db;
SELECT * FROM customers;

-- Ou qualifier complètement :
SELECT * FROM production_db.customers;
```

### Erreur : "Unknown database"

```sql
-- Problème :
USE nonexistent;
-- ERROR 1049 (42000): Unknown database 'nonexistent'

-- Solution : Vérifier les bases disponibles
SHOW DATABASES;
```

### Erreur : "Table doesn't exist"

```sql
-- Problème :
SELECT * FROM customer;
-- ERROR 1146 (42S02): Table 'production_db.customer' doesn't exist

-- Solution : Vérifier le nom exact
SHOW TABLES LIKE '%customer%';
-- Résultat : customers (avec 's')
```

### Erreur : "Access denied"

```sql
-- Problème :
USE admin_db;
-- ERROR 1044 (42000): Access denied for user 'myuser'@'localhost' to database 'admin_db'

-- Solution : Vérifier privilèges
SHOW GRANTS;

-- Demander accès à l'administrateur
```

---

## ✅ Points Clés à Retenir

### Connexion
- ✅ Toujours utiliser `-p` seul (sécurité)
- ✅ Utiliser SSL/TLS pour connexions distantes
- ✅ Vérifier connexion avec `\s` en début de session

### Navigation
- ✅ `USE database` ou `\u database` pour changer de base
- ✅ `SELECT DATABASE()` pour vérifier base active
- ✅ `SHOW DATABASES` pour lister bases accessibles

### Exploration
- ✅ `SHOW TABLES` pour lister tables
- ✅ `DESC table` pour structure rapide
- ✅ `SHOW CREATE TABLE` pour définition complète
- ✅ `SHOW INDEX FROM table` pour voir les index

### Productivité
- ✅ Utiliser `\G` pour affichage vertical
- ✅ Activer auto-complétion (TAB)
- ✅ Personnaliser le prompt
- ✅ Utiliser INFORMATION_SCHEMA pour requêtes complexes

---

## 🔗 Ressources et Références

### Documentation Officielle
- [SHOW Statements](https://mariadb.com/kb/en/show/)
- [USE Statement](https://mariadb.com/kb/en/use/)
- [DESCRIBE Statement](https://mariadb.com/kb/en/describe/)
- [INFORMATION_SCHEMA](https://mariadb.com/kb/en/information-schema/)

### Commandes Associées
- **Section B.2** : SHOW PROCESSLIST, SHOW VARIABLES, SHOW STATUS
- **Section B.3** : SOURCE, TEE, export/import

---

## ➡️ Section Suivante

**[B.2 Informations Système →](./02-informations-systeme.md)**  
Découvrez les commandes pour surveiller et diagnostiquer votre serveur MariaDB : STATUS, SHOW PROCESSLIST, SHOW ENGINE, SHOW VARIABLES

---

**MariaDB** : 11.8 LTS

⏭️ [Informations système (STATUS, SHOW PROCESSLIST, SHOW ENGINE)](/annexes/commandes-cli/02-informations-systeme.md)

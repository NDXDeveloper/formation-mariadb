🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Création et gestion des bases de données

> **Niveau** : Débutant
> **Durée estimée** : 1 heure
> **Prérequis** : Section 2.2 (Types de données MariaDB)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Créer une base de données avec CREATE DATABASE
- Configurer le charset et la collation appropriés
- Modifier les propriétés d'une base existante avec ALTER DATABASE
- Supprimer une base de données en toute sécurité
- Lister et naviguer entre les bases de données
- Appliquer les bonnes pratiques de nommage et d'organisation
- Comprendre les nouveautés MariaDB 11.8 (utf8mb4 par défaut)

---

## Introduction

Une **base de données** (ou schéma) est un conteneur logique qui regroupe des objets liés : tables, vues, procédures stockées, fonctions, etc. C'est le niveau le plus haut d'organisation dans MariaDB.

```
Serveur MariaDB
    ├── Base de données 1 (ex: shop_production)
    │   ├── Table: clients
    │   ├── Table: commandes
    │   ├── Table: produits
    │   └── Vue: ventes_mensuelles
    │
    ├── Base de données 2 (ex: shop_dev)
    │   └── [même structure pour développement]
    │
    └── Base de données système
        ├── mysql (gestion utilisateurs)
        ├── information_schema (métadonnées)
        └── performance_schema (monitoring)
```

💡 **Analogie** : Une base de données est comme un classeur où chaque table est une feuille contenant des données organisées.

---

## CREATE DATABASE - Créer une base de données

### Syntaxe de base

```sql
CREATE DATABASE nom_base;
```

La forme la plus simple crée une base avec les paramètres par défaut :

```sql
-- Création d'une base simple
CREATE DATABASE ma_boutique;

-- Vérification
SHOW DATABASES;

-- Résultat :
-- +--------------------+
-- | Database           |
-- +--------------------+
-- | information_schema |
-- | ma_boutique        |   ← Notre nouvelle base
-- | mysql              |
-- | performance_schema |
-- +--------------------+
```

### CREATE DATABASE IF NOT EXISTS

Pour éviter les erreurs si la base existe déjà :

```sql
-- ❌ Sans IF NOT EXISTS : Erreur si existe
CREATE DATABASE ma_boutique;
-- ERROR 1007: Can't create database 'ma_boutique'; database exists

-- ✅ Avec IF NOT EXISTS : Pas d'erreur
CREATE DATABASE IF NOT EXISTS ma_boutique;
-- Query OK, 1 warning (0.00 sec)

-- Vérifier les warnings
SHOW WARNINGS;
-- Note 1007: Can't create database 'ma_boutique'; database exists
```

💡 **Bonne pratique** : Toujours utiliser `IF NOT EXISTS` dans les scripts pour éviter les erreurs en cas de ré-exécution.

---

## Charset et Collation

### Qu'est-ce que le charset et la collation ?

- **Charset (jeu de caractères)** : Définit quels caractères peuvent être stockés
  - `latin1` : Caractères occidentaux (obsolète)
  - `utf8` : Unicode 3 octets max (obsolète, pas d'emoji)
  - `utf8mb4` : Unicode 4 octets max (recommandé) ✅

- **Collation** : Définit comment comparer et trier les caractères
  - `_ci` : Case Insensitive (A = a)
  - `_cs` : Case Sensitive (A ≠ a)
  - `_bin` : Binaire (comparaison byte par byte)
  - `_ai` : Accent Insensitive (é = e)
  - `_as` : Accent Sensitive (é ≠ e)

🆕 **MariaDB 11.8** : Le charset par défaut est **utf8mb4** avec la collation **utf8mb4_uca_1400_ai_ci** (Unicode Collation Algorithm 14.0.0).

### Création avec charset explicite

```sql
-- Création avec utf8mb4 (recommandé)
CREATE DATABASE ma_boutique
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Vérification de la configuration
SELECT
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'ma_boutique';

-- Résultat :
-- +-------------+----------------------------+--------------------+
-- | SCHEMA_NAME | DEFAULT_CHARACTER_SET_NAME | DEFAULT_COLLATION  |
-- +-------------+----------------------------+--------------------+
-- | ma_boutique | utf8mb4                    | utf8mb4_unicode_ci |
-- +-------------+----------------------------+--------------------+
```

### Collations courantes utf8mb4

```sql
-- Collation générale (recommandée pour la plupart des cas)
CREATE DATABASE app_general
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;              -- Case insensitive, accent insensitive

-- Collation UCA 14.0.0 (nouvelle dans MariaDB 11.8)
CREATE DATABASE app_modern
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca_1400_ai_ci;         -- 🆕 Meilleure conformité Unicode

-- Collation sensible à la casse (pour logins, passwords)
CREATE DATABASE app_secure
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_bin;                     -- Case sensitive, binary comparison

-- Collation pour performance
CREATE DATABASE app_performance
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_general_ci;             -- Plus rapide mais moins précis
```

### Comparaison des collations

```sql
-- Créer une base de test
CREATE DATABASE test_collation
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE test_collation;

-- Table avec collation insensible à la casse
CREATE TABLE demo_ci (
    texte VARCHAR(50) COLLATE utf8mb4_unicode_ci
);

INSERT INTO demo_ci VALUES ('Café'), ('café'), ('CAFÉ');

-- Recherche insensible : trouve les 3
SELECT * FROM demo_ci WHERE texte = 'cafe';
-- Résultat : 3 lignes (Café, café, CAFÉ)

-- Table avec collation binaire
CREATE TABLE demo_bin (
    texte VARCHAR(50) COLLATE utf8mb4_bin
);

INSERT INTO demo_bin VALUES ('Café'), ('café'), ('CAFÉ');

-- Recherche sensible : trouve seulement la correspondance exacte
SELECT * FROM demo_bin WHERE texte = 'café';
-- Résultat : 1 ligne (café)
```

💡 **Recommandations** :
- **Usage général** : `utf8mb4_unicode_ci` ou `utf8mb4_uca_1400_ai_ci` 🆕
- **Logins/Passwords** : `utf8mb4_bin`
- **Performance critique** : `utf8mb4_general_ci`
- **Jamais** : `latin1` (obsolète), `utf8` (pas d'emoji)

---

## USE - Sélectionner une base de données

Avant de travailler avec les tables, il faut sélectionner une base :

```sql
-- Méthode 1 : Commande USE
USE ma_boutique;
-- Database changed

-- Vérifier la base active
SELECT DATABASE();
-- +-------------+
-- | DATABASE()  |
-- +-------------+
-- | ma_boutique |
-- +-------------+

-- Méthode 2 : Préfixer le nom de table (pas besoin de USE)
SELECT * FROM ma_boutique.clients;

-- Méthode 3 : À la connexion
-- mysql -u root -p -D ma_boutique
```

---

## SHOW DATABASES - Lister les bases

### Commandes de base

```sql
-- Lister toutes les bases de données
SHOW DATABASES;

-- Filtrer avec LIKE
SHOW DATABASES LIKE 'shop%';
-- Montre : shop_production, shop_dev, shop_test

-- Filtrer avec WHERE
SHOW DATABASES WHERE `Database` NOT LIKE 'mysql%';

-- Voir les détails via INFORMATION_SCHEMA
SELECT
    SCHEMA_NAME AS base_de_donnees,
    DEFAULT_CHARACTER_SET_NAME AS charset,
    DEFAULT_COLLATION_NAME AS collation
FROM information_schema.SCHEMATA
ORDER BY SCHEMA_NAME;
```

### Taille des bases de données

```sql
-- Calculer la taille de chaque base
SELECT
    table_schema AS base_de_donnees,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS taille_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY taille_mb DESC;

-- Résultat exemple :
-- +--------------------+----------+
-- | base_de_donnees    | taille_mb|
-- +--------------------+----------+
-- | shop_production    | 1250.45  |
-- | shop_dev           | 156.78   |
-- | mysql              | 15.23    |
-- +--------------------+----------+
```

---

## ALTER DATABASE - Modifier une base

### Changer le charset et la collation

```sql
-- Modifier le charset par défaut de la base
ALTER DATABASE ma_boutique
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- ⚠️ Important : Cela change seulement le DEFAULT pour les nouvelles tables
-- Les tables existantes ne sont PAS modifiées automatiquement

-- Pour modifier les tables existantes aussi :
-- 1. Modifier la base
ALTER DATABASE ma_boutique CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 2. Modifier chaque table
ALTER TABLE ma_boutique.clients CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE ma_boutique.produits CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Options de lecture seule (MariaDB 10.6+)

```sql
-- Mettre une base en lecture seule
ALTER DATABASE ma_boutique READ ONLY = 1;

-- Essayer d'insérer : ERREUR
USE ma_boutique;
INSERT INTO clients (nom) VALUES ('Test');
-- ERROR 1036: Table 'clients' is read only

-- Retirer le mode lecture seule
ALTER DATABASE ma_boutique READ ONLY = 0;
```

💡 **Cas d'usage** : Mode lecture seule utile pour :
- Maintenance et migration
- Backup cohérent
- Environnement de reporting

---

## DROP DATABASE - Supprimer une base

### Suppression simple

```sql
-- Supprimer une base (ATTENTION : Irréversible !)
DROP DATABASE ma_boutique;

-- Avec IF EXISTS pour éviter les erreurs
DROP DATABASE IF EXISTS ma_boutique;
```

⚠️ **DANGER** : DROP DATABASE supprime **toutes les tables et données** de manière **irréversible** !

### Sécurité et bonnes pratiques

```sql
-- ❌ MAUVAIS : Supprimer directement en production
DROP DATABASE shop_production;  -- DÉSASTRE !

-- ✅ BON : Vérifications avant suppression

-- 1. Vérifier qu'on est sur le bon serveur
SELECT @@hostname, @@port;

-- 2. Lister le contenu de la base
SHOW TABLES FROM shop_old;

-- 3. Vérifier la taille
SELECT
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS taille_mb
FROM information_schema.tables
WHERE table_schema = 'shop_old';

-- 4. Faire un backup avant suppression
-- mysqldump -u root -p shop_old > backup_shop_old_$(date +%Y%m%d).sql

-- 5. Supprimer avec confirmation
DROP DATABASE IF EXISTS shop_old;

-- 6. Vérifier que c'est bien supprimé
SHOW DATABASES LIKE 'shop_old';
-- Empty set (0.00 sec)
```

### Alternative sécurisée : Renommer avant supprimer

```sql
-- Renommer la base (nécessite création nouvelle + migration)
-- MariaDB n'a pas de commande RENAME DATABASE directe

-- Méthode : Créer nouvelle base et copier
CREATE DATABASE shop_archived LIKE shop_old;

-- Copier les tables une par une
RENAME TABLE
    shop_old.clients TO shop_archived.clients,
    shop_old.commandes TO shop_archived.commandes,
    shop_old.produits TO shop_archived.produits;

-- Vérifier que shop_old est vide
SHOW TABLES FROM shop_old;

-- Supprimer shop_old (maintenant vide)
DROP DATABASE shop_old;
```

---

## Conventions de nommage

### Règles de base

```sql
-- ✅ BONS noms de bases
CREATE DATABASE shop_production;      -- Descriptif + environnement
CREATE DATABASE blog_2025;            -- Avec année
CREATE DATABASE crm_clients;          -- Application + domaine
CREATE DATABASE api_v2;               -- Version incluse

-- ❌ MAUVAIS noms
CREATE DATABASE `2025-shop`;          -- Commence par chiffre (backticks requis)
CREATE DATABASE `ma base`;            -- Espaces (backticks requis)
CREATE DATABASE test;                 -- Trop générique
CREATE DATABASE db1;                  -- Pas descriptif
CREATE DATABASE SHOP_PROD;            -- MAJUSCULES (éviter)
```

### Conventions recommandées

| Convention | Exemple | Usage |
|------------|---------|-------|
| **snake_case** | `shop_production` | ✅ Recommandé |
| **Environnement** | `app_dev`, `app_prod`, `app_test` | ✅ Bonne pratique |
| **Version** | `api_v1`, `api_v2` | ✅ Pour compatibilité |
| **Préfixe projet** | `proj_shop_prod`, `proj_blog_prod` | ✅ Multi-projets |
| **camelCase** | `shopProduction` | ⚠️ Éviter (sensible casse) |
| **Accents** | `base_données` | ❌ Éviter |
| **Espaces** | `ma base` | ❌ Interdit sans backticks |

```sql
-- Structure recommandée pour plusieurs environnements
CREATE DATABASE myapp_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE myapp_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE myapp_prod CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## Organisation multi-environnements

### Approche 1 : Bases séparées par environnement

```sql
-- Développement
CREATE DATABASE shop_dev
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Test / Staging
CREATE DATABASE shop_test
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Production
CREATE DATABASE shop_prod
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Avantages :
-- ✅ Isolation complète
-- ✅ Facile à sauvegarder séparément
-- ✅ Permissions différentes par environnement

-- Inconvénients :
-- ❌ Multiplication des bases
-- ❌ Migration de schéma à répéter
```

### Approche 2 : Serveurs séparés par environnement

```sql
-- Serveur DEV (localhost:3306)
CREATE DATABASE shop;

-- Serveur TEST (test-db.example.com:3306)
CREATE DATABASE shop;

-- Serveur PROD (prod-db.example.com:3306)
CREATE DATABASE shop;

-- Avantages :
-- ✅ Isolation matérielle
-- ✅ Même nom de base partout
-- ✅ Meilleures performances prod

-- Inconvénients :
-- ❌ Coût infrastructure
-- ❌ Gestion plus complexe
```

---

## Permissions sur les bases de données

### Créer un utilisateur avec accès à une base

```sql
-- Créer une base
CREATE DATABASE app_production
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Créer un utilisateur applicatif
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'S3cur3P@ssw0rd!';

-- Donner tous les droits sur la base
GRANT ALL PRIVILEGES ON app_production.* TO 'app_user'@'localhost';

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Vérifier les privilèges
SHOW GRANTS FOR 'app_user'@'localhost';
-- +------------------------------------------------------------------------+
-- | Grants for app_user@localhost                                          |
-- +------------------------------------------------------------------------+
-- | GRANT USAGE ON *.* TO 'app_user'@'localhost'                           |
-- | GRANT ALL PRIVILEGES ON `app_production`.* TO 'app_user'@'localhost'   |
-- +------------------------------------------------------------------------+
```

### Permissions granulaires

```sql
-- Utilisateur en lecture seule
CREATE USER 'app_readonly'@'%' IDENTIFIED BY 'R3adOnlyP@ss!';
GRANT SELECT ON app_production.* TO 'app_readonly'@'%';

-- Utilisateur avec droits spécifiques
CREATE USER 'app_limited'@'localhost' IDENTIFIED BY 'L1mitedP@ss!';
GRANT SELECT, INSERT, UPDATE ON app_production.* TO 'app_limited'@'localhost';

-- Administrateur de base (pas super admin)
CREATE USER 'db_admin'@'localhost' IDENTIFIED BY 'Adm1nP@ss!';
GRANT ALL PRIVILEGES ON app_production.* TO 'db_admin'@'localhost';
GRANT CREATE, DROP, ALTER ON app_production.* TO 'db_admin'@'localhost';

-- Révoquer des droits
REVOKE INSERT ON app_production.* FROM 'app_limited'@'localhost';
```

---

## Bases de données système

MariaDB utilise plusieurs bases système qu'il **ne faut jamais supprimer** :

### mysql - Gestion des utilisateurs et privilèges

```sql
-- Base principale de gestion
USE mysql;

SHOW TABLES;
-- Tables importantes :
-- - user : Utilisateurs et authentification
-- - db : Privilèges par base de données
-- - tables_priv : Privilèges par table
-- - columns_priv : Privilèges par colonne

-- Lister les utilisateurs
SELECT User, Host FROM mysql.user;

-- Lister les bases avec privilèges
SELECT * FROM mysql.db WHERE User = 'app_user'\G
```

### information_schema - Métadonnées

```sql
-- Catalogue de toutes les métadonnées
USE information_schema;

-- Tables utiles :
-- - SCHEMATA : Liste des bases
-- - TABLES : Liste des tables
-- - COLUMNS : Colonnes de toutes les tables
-- - STATISTICS : Index et statistiques

-- Exemple : Toutes les tables d'une base
SELECT
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS taille_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'ma_boutique'
ORDER BY taille_mb DESC;
```

### performance_schema - Monitoring

```sql
-- Monitoring des performances en temps réel
USE performance_schema;

-- Tables importantes :
-- - events_statements_summary_by_digest : Requêtes agrégées
-- - table_io_waits_summary_by_table : I/O par table
-- - memory_summary_global_by_event_name : Utilisation mémoire

-- Exemple : Top 10 requêtes les plus lentes
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 2) AS avg_seconds,
    ROUND(MAX_TIMER_WAIT / 1000000000000, 2) AS max_seconds
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

⚠️ **IMPORTANT** : Ne jamais exécuter `DROP DATABASE` sur :
- `mysql`
- `information_schema`
- `performance_schema`
- `sys` (MariaDB 10.5+)

---

## Script de création complet

Exemple de script pour initialiser une nouvelle application :

```sql
-- ============================================
-- Script de création base de données
-- Application : Boutique en ligne
-- Version : 1.0
-- Date : 2025-12-12
-- ============================================

-- 1. Créer la base de données
CREATE DATABASE IF NOT EXISTS shop_production
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca_1400_ai_ci  -- 🆕 MariaDB 11.8
    COMMENT 'Base de production de la boutique';

-- 2. Vérifier la création
SELECT
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'shop_production';

-- 3. Créer les utilisateurs

-- Utilisateur applicatif (lecture/écriture)
CREATE USER IF NOT EXISTS 'shop_app'@'%'
    IDENTIFIED BY 'Sh0pApp_P@ssw0rd!'
    REQUIRE SSL;  -- Forcer SSL

GRANT SELECT, INSERT, UPDATE, DELETE
    ON shop_production.*
    TO 'shop_app'@'%';

-- Utilisateur admin base (DDL)
CREATE USER IF NOT EXISTS 'shop_admin'@'localhost'
    IDENTIFIED BY 'Sh0pAdm1n_P@ss!';

GRANT ALL PRIVILEGES
    ON shop_production.*
    TO 'shop_admin'@'localhost';

-- Utilisateur lecture seule (reporting)
CREATE USER IF NOT EXISTS 'shop_readonly'@'%'
    IDENTIFIED BY 'Sh0pR3ad_P@ss!';

GRANT SELECT
    ON shop_production.*
    TO 'shop_readonly'@'%';

-- 4. Appliquer les privilèges
FLUSH PRIVILEGES;

-- 5. Vérifier les utilisateurs
SELECT User, Host FROM mysql.user
WHERE User LIKE 'shop_%';

-- 6. Sélectionner la base
USE shop_production;

-- 7. Afficher un message de confirmation
SELECT
    'Base de données créée avec succès' AS Status,
    DATABASE() AS Base_active,
    @@character_set_database AS Charset,
    @@collation_database AS Collation;

-- ============================================
-- Fin du script
-- ============================================
```

---

## Sauvegarde et restauration

### Sauvegarder une base

```bash
# Sauvegarde complète
mysqldump -u root -p ma_boutique > backup_ma_boutique.sql

# Sauvegarde avec structure seulement (pas de données)
mysqldump -u root -p --no-data ma_boutique > structure_ma_boutique.sql

# Sauvegarde avec données seulement (pas de structure)
mysqldump -u root -p --no-create-info ma_boutique > donnees_ma_boutique.sql

# Sauvegarde compressée
mysqldump -u root -p ma_boutique | gzip > backup_ma_boutique.sql.gz

# Sauvegarde de plusieurs bases
mysqldump -u root -p --databases ma_boutique blog_2025 > backup_multiple.sql

# Sauvegarde de toutes les bases
mysqldump -u root -p --all-databases > backup_all.sql
```

### Restaurer une base

```bash
# Restauration simple
mysql -u root -p ma_boutique < backup_ma_boutique.sql

# Restaurer depuis une archive compressée
gunzip < backup_ma_boutique.sql.gz | mysql -u root -p ma_boutique

# Créer la base avant restauration si nécessaire
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS ma_boutique"
mysql -u root -p ma_boutique < backup_ma_boutique.sql
```

```sql
-- Restauration depuis le client MariaDB
USE ma_boutique;
SOURCE /path/to/backup_ma_boutique.sql;

-- Ou avec la commande system (si activé)
\. /path/to/backup_ma_boutique.sql
```

---

## Pièges courants à éviter

### 1. Oublier le charset utf8mb4

```sql
-- ❌ MAUVAIS : Charset par défaut (peut être latin1 sur vieux serveurs)
CREATE DATABASE shop;

-- Essayer d'insérer des emoji : ERREUR ou corruption
INSERT INTO shop.messages VALUES ('Hello 😊');

-- ✅ BON : Toujours spécifier utf8mb4
CREATE DATABASE shop
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

### 2. Supprimer une base sans backup

```sql
-- ❌ DÉSASTRE : Supprimer sans backup
DROP DATABASE shop_production;  -- Toutes les données perdues !

-- ✅ BON : Toujours sauvegarder avant
-- 1. Backup
-- mysqldump -u root -p shop_production > backup_avant_suppression.sql

-- 2. Vérifier le backup
-- mysql -u root -p -e "SELECT COUNT(*) FROM shop_production.clients"

-- 3. Supprimer
DROP DATABASE IF EXISTS shop_production;
```

### 3. Mélanger les charsets

```sql
-- ❌ PROBLÈME : Base en utf8mb4, tables en latin1
CREATE DATABASE shop CHARACTER SET utf8mb4;

USE shop;

CREATE TABLE clients (
    nom VARCHAR(100)
) CHARACTER SET latin1;  -- Différent de la base !

-- Problèmes de caractères garantis !

-- ✅ SOLUTION : Charset cohérent partout
CREATE DATABASE shop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE TABLE clients (
    nom VARCHAR(100)
);  -- Hérite du charset de la base
```

### 4. Nom de base avec caractères spéciaux

```sql
-- ❌ MAUVAIS : Nécessite backticks partout
CREATE DATABASE `ma-base-2025`;
USE `ma-base-2025`;
SELECT * FROM `ma-base-2025`.clients;  -- Backticks obligatoires !

-- ✅ BON : Nom simple
CREATE DATABASE ma_base_2025;
USE ma_base_2025;
SELECT * FROM ma_base_2025.clients;  -- Pas de backticks nécessaires
```

### 5. Permissions trop larges

```sql
-- ❌ DANGEREUX : Accès complet à toutes les bases
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- L'utilisateur peut :
-- - Supprimer n'importe quelle base
-- - Créer des utilisateurs
-- - Accéder à mysql, information_schema, etc.

-- ✅ SÉCURISÉ : Limiter à une base spécifique
GRANT SELECT, INSERT, UPDATE, DELETE
    ON shop_production.*
    TO 'app_user'@'%';
```

---

## ✅ Points clés à retenir

- **CREATE DATABASE** : Créer une base avec charset et collation
- **ALTER DATABASE** : Modifier les propriétés d'une base
- **DROP DATABASE** : Supprimer une base (⚠️ irréversible)
- **USE** : Sélectionner la base active
- **SHOW DATABASES** : Lister toutes les bases
- 🆕 **utf8mb4** : Charset par défaut dans MariaDB 11.8
- 🆕 **uca_1400_ai_ci** : Nouvelle collation Unicode 14.0.0
- **Toujours** spécifier charset utf8mb4 explicitement
- **IF NOT EXISTS** : Évite les erreurs dans les scripts
- **Backup avant DROP** : Toujours sauvegarder avant suppression
- **Bases système** : Ne jamais supprimer mysql, information_schema, etc.
- **Permissions** : Limiter aux bases nécessaires uniquement
- **Nommage** : snake_case, descriptif, avec environnement

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CREATE DATABASE](https://mariadb.com/kb/en/create-database/)
- [📖 ALTER DATABASE](https://mariadb.com/kb/en/alter-database/)
- [📖 DROP DATABASE](https://mariadb.com/kb/en/drop-database/)
- [📖 Character Sets and Collations](https://mariadb.com/kb/en/character-sets/)
- [📖 utf8mb4 Character Set](https://mariadb.com/kb/en/unicode/)
- [📖 SHOW DATABASES](https://mariadb.com/kb/en/show-databases/)
- [📖 mysqldump](https://mariadb.com/kb/en/mysqldump/)

### Bonnes pratiques
- [Database Naming Conventions](https://www.vertabelo.com/blog/database-naming-conventions/)
- [Character Set Best Practices](https://mathiasbynens.be/notes/mysql-utf8mb4)

---

## ➡️ Section suivante

**[2.4 Création et modification de tables (CREATE, ALTER, DROP)](/02-bases-du-sql/04-creation-modification-tables.md)**

Maintenant que vous savez créer et gérer des bases de données, apprenez à créer des tables avec la syntaxe CREATE TABLE, à les modifier avec ALTER TABLE, et à comprendre les options de stockage et de configuration des tables.

---


⏭️ [Création et modification de tables (CREATE, ALTER, DROP)](/02-bases-du-sql/04-creation-modification-tables.md)

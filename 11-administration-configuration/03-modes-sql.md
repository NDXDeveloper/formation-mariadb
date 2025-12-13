🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3 Modes SQL (sql_mode)

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** :
> - Section 11.2 (Variables système et de session)
> - Connaissance SQL intermédiaire
> - Expérience avec les contraintes et validations

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et l'impact du `sql_mode` sur le comportement MariaDB
- **Distinguer** les modes stricts des modes permissifs
- **Configurer** le `sql_mode` optimal pour votre environnement
- **Anticiper** les différences de comportement entre modes
- **Résoudre** les problèmes de compatibilité lors de migrations
- **Émuler** d'autres SGBD (MySQL, Oracle, PostgreSQL) avec les modes appropriés
- **Appliquer** les bonnes pratiques en production

---

## Introduction

Le **sql_mode** est une variable système qui contrôle le **niveau de rigueur** et la **compatibilité** du moteur SQL de MariaDB. C'est l'une des variables les plus importantes car elle affecte :

- ✅ La **validation des données** insérées
- ✅ Le **comportement des requêtes** SQL
- ✅ La **gestion des erreurs** et warnings
- ✅ La **compatibilité** avec d'autres SGBD
- ✅ Les **conversions implicites** de types

### Pourquoi sql_mode est crucial

```
Mode Permissif
    ↓
Données invalides acceptées silencieusement
    ↓
Corruption de données à long terme
    ↓
Problèmes de qualité et intégrité

VS

Mode Strict
    ↓
Validation rigoureuse des données
    ↓
Rejet immédiat des données invalides
    ↓
Intégrité garantie
```

💡 **Principe fondamental** : En production, préférez toujours un mode **strict** pour garantir l'intégrité des données, même si cela nécessite plus de travail au niveau applicatif.

---

## Consultation du sql_mode actuel

### Vérifier le mode en cours

```sql
-- Méthode 1 : SELECT @@
SELECT @@sql_mode;

-- Méthode 2 : SHOW VARIABLES
SHOW VARIABLES LIKE 'sql_mode';

-- Méthode 3 : Scope explicite
SELECT @@global.sql_mode AS mode_global;
SELECT @@session.sql_mode AS mode_session;
```

**Sortie exemple MariaDB 11.8** :

```
STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

Le `sql_mode` est une **liste de modes** séparés par des virgules. Chaque mode active un comportement spécifique.

---

## Modes individuels principaux

### STRICT_TRANS_TABLES

**Comportement** : Mode strict pour les tables **transactionnelles** (InnoDB).

```sql
-- Sans STRICT_TRANS_TABLES
SET sql_mode = '';

CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(10)
);

-- Insertion dépassant la longueur (10 caractères)
INSERT INTO users VALUES (1, 'NomTropLongQuiDepasse');
-- ⚠️ ACCEPTÉ : Valeur tronquée silencieusement à 'NomTropLon'
-- WARNING: Data truncated for column 'name'

-- Avec STRICT_TRANS_TABLES
SET sql_mode = 'STRICT_TRANS_TABLES';

INSERT INTO users VALUES (2, 'NomTropLongQuiDepasse');
-- ❌ ERREUR: Data too long for column 'name' at row 1
```

**Recommandation** : **TOUJOURS activer** en production pour les tables InnoDB.

### STRICT_ALL_TABLES

**Comportement** : Mode strict pour **toutes** les tables (InnoDB + MyISAM/Aria).

```sql
SET sql_mode = 'STRICT_ALL_TABLES';

-- S'applique même aux tables MyISAM
CREATE TABLE legacy_table (
    code CHAR(5)
) ENGINE=MyISAM;

INSERT INTO legacy_table VALUES ('ABCDEFGH');
-- ❌ ERREUR: Data too long
```

**Différence avec STRICT_TRANS_TABLES** :
- `STRICT_TRANS_TABLES` : Strict uniquement pour InnoDB
- `STRICT_ALL_TABLES` : Strict pour tous les moteurs

💡 **Usage** : Préférez `STRICT_ALL_TABLES` si vous utilisez MyISAM/Aria en production.

### NO_ZERO_DATE

**Comportement** : Interdit les dates `'0000-00-00'`.

```sql
-- Sans NO_ZERO_DATE
SET sql_mode = '';

CREATE TABLE events (
    id INT,
    event_date DATE
);

INSERT INTO events VALUES (1, '0000-00-00');
-- ✅ ACCEPTÉ (mais invalide logiquement)

-- Avec NO_ZERO_DATE
SET sql_mode = 'NO_ZERO_DATE';

INSERT INTO events VALUES (2, '0000-00-00');
-- ❌ ERREUR: Invalid default value for 'event_date'
```

⚠️ **Note** : `'0000-00-00'` est une date **non-standard** et devrait être remplacée par `NULL`.

### NO_ZERO_IN_DATE

**Comportement** : Interdit les composants zéro dans les dates (`'2025-00-15'`, `'2025-12-00'`).

```sql
SET sql_mode = 'NO_ZERO_IN_DATE';

INSERT INTO events VALUES (3, '2025-00-15');
-- ❌ ERREUR: Invalid date value

INSERT INTO events VALUES (4, '2025-12-00');
-- ❌ ERREUR: Invalid date value
```

### ERROR_FOR_DIVISION_BY_ZERO

**Comportement** : Division par zéro génère une **erreur** au lieu de `NULL`.

```sql
-- Sans ERROR_FOR_DIVISION_BY_ZERO
SET sql_mode = '';

SELECT 10 / 0;
-- Résultat: NULL (avec warning)

-- Avec ERROR_FOR_DIVISION_BY_ZERO + mode strict
SET sql_mode = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO';

SELECT 10 / 0;
-- ❌ ERREUR: Division by 0
```

💡 **Bonne pratique** : Activer pour détecter les bugs logiques dans les calculs.

### NO_AUTO_VALUE_ON_ZERO

**Comportement** : Empêche l'insertion de valeur `0` dans une colonne `AUTO_INCREMENT` de générer une nouvelle valeur.

```sql
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50)
);

-- Sans NO_AUTO_VALUE_ON_ZERO
SET sql_mode = '';

INSERT INTO products VALUES (0, 'Produit A');
-- id généré automatiquement (ex: 1)

-- Avec NO_AUTO_VALUE_ON_ZERO
SET sql_mode = 'NO_AUTO_VALUE_ON_ZERO';

INSERT INTO products VALUES (0, 'Produit B');
-- id = 0 (valeur littérale respectée)
```

**Cas d'usage** : Import de données avec des IDs existants incluant `0`.

### NO_ENGINE_SUBSTITUTION

**Comportement** : Erreur si le moteur de stockage demandé n'est pas disponible.

```sql
-- Sans NO_ENGINE_SUBSTITUTION
SET sql_mode = '';

CREATE TABLE test_engine (id INT) ENGINE=MoteurInexistant;
-- ⚠️ WARNING: Table créée avec moteur par défaut (InnoDB)

-- Avec NO_ENGINE_SUBSTITUTION
SET sql_mode = 'NO_ENGINE_SUBSTITUTION';

CREATE TABLE test_engine (id INT) ENGINE=MoteurInexistant;
-- ❌ ERREUR: Unknown storage engine 'MoteurInexistant'
```

**Recommandation** : Activer pour éviter les surprises lors de restaurations.

### NO_AUTO_CREATE_USER

**Comportement** : Empêche `GRANT` de créer automatiquement un utilisateur sans mot de passe.

```sql
-- Sans NO_AUTO_CREATE_USER (dangereux)
SET sql_mode = '';

GRANT SELECT ON mydb.* TO 'newuser'@'localhost';
-- ⚠️ Utilisateur créé SANS mot de passe !

-- Avec NO_AUTO_CREATE_USER
SET sql_mode = 'NO_AUTO_CREATE_USER';

GRANT SELECT ON mydb.* TO 'newuser'@'localhost';
-- ❌ ERREUR: Can't find user 'newuser'@'localhost'
```

**Sécurité** : **Toujours activer** pour éviter la création de comptes non sécurisés.

### PIPES_AS_CONCAT

**Comportement** : `||` devient l'opérateur de concaténation (comme PostgreSQL) au lieu de `OR`.

```sql
-- Sans PIPES_AS_CONCAT (comportement SQL standard)
SET sql_mode = '';

SELECT 'Hello' || 'World';
-- Résultat: 0 (évalué comme OR logique)

SELECT CONCAT('Hello', 'World');
-- Résultat: 'HelloWorld'

-- Avec PIPES_AS_CONCAT (compatible PostgreSQL)
SET sql_mode = 'PIPES_AS_CONCAT';

SELECT 'Hello' || 'World';
-- Résultat: 'HelloWorld'
```

**Migration PostgreSQL** : Facilite la compatibilité des requêtes.

### ANSI_QUOTES

**Comportement** : Double quotes `"` deviennent des délimiteurs d'identifiants (au lieu de chaînes).

```sql
-- Sans ANSI_QUOTES
SET sql_mode = '';

SELECT "Hello";
-- Résultat: 'Hello' (interprété comme chaîne)

-- Avec ANSI_QUOTES (standard ANSI SQL)
SET sql_mode = 'ANSI_QUOTES';

SELECT "Hello";
-- ❌ ERREUR: Unknown column 'Hello'

SELECT "name" FROM users;
-- ✅ Correct: "name" est un identifiant (colonne)

SELECT 'Hello';
-- ✅ Correct: 'Hello' est une chaîne
```

**Standard SQL** : Les chaînes utilisent `'` (simple quote), les identifiants `"` (double quote).

### ONLY_FULL_GROUP_BY

**Comportement** : Impose que toutes les colonnes dans `SELECT` non-agrégées soient dans `GROUP BY`.

```sql
-- Sans ONLY_FULL_GROUP_BY
SET sql_mode = '';

SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;
-- ⚠️ ACCEPTÉ : 'name' n'est pas dans GROUP BY (résultat indéterminé)

-- Avec ONLY_FULL_GROUP_BY (standard SQL)
SET sql_mode = 'ONLY_FULL_GROUP_BY';

SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;
-- ❌ ERREUR: 'name' isn't in GROUP BY

-- Solution correcte
SELECT department, COUNT(*) AS total
FROM employees
GROUP BY department;
-- ✅ OK
```

**Standard SQL:2003** : Garantit la déterminisme des requêtes.

### NO_UNSIGNED_SUBTRACTION

**Comportement** : Autorise les résultats négatifs dans les soustractions d'UNSIGNED.

```sql
-- Sans NO_UNSIGNED_SUBTRACTION
SET sql_mode = '';

SELECT CAST(5 AS UNSIGNED) - CAST(10 AS UNSIGNED);
-- ❌ ERREUR: BIGINT UNSIGNED value is out of range

-- Avec NO_UNSIGNED_SUBTRACTION
SET sql_mode = 'NO_UNSIGNED_SUBTRACTION';

SELECT CAST(5 AS UNSIGNED) - CAST(10 AS UNSIGNED);
-- Résultat: -5 (autorisé)
```

---

## Modes composés

Les modes composés sont des **raccourcis** regroupant plusieurs modes individuels.

### TRADITIONAL

**Équivalent à** :
```
STRICT_TRANS_TABLES,
STRICT_ALL_TABLES,
NO_ZERO_IN_DATE,
NO_ZERO_DATE,
ERROR_FOR_DIVISION_BY_ZERO,
NO_AUTO_CREATE_USER,
NO_ENGINE_SUBSTITUTION
```

**Usage** : Mode **strict maximal** recommandé pour la production.

```sql
SET sql_mode = 'TRADITIONAL';

-- Équivalent à activer tous les modes stricts
```

**Avantages** :
- ✅ Validation stricte des données
- ✅ Détection précoce des erreurs
- ✅ Intégrité garantie

**Inconvénients** :
- ⚠️ Peut casser des applications legacy tolérantes aux erreurs
- ⚠️ Nécessite un code applicatif rigoureux

### ANSI

**Équivalent à** :
```
REAL_AS_FLOAT,
PIPES_AS_CONCAT,
ANSI_QUOTES,
IGNORE_SPACE
```

**Usage** : Compatibilité avec le **standard ANSI SQL**.

```sql
SET sql_mode = 'ANSI';

-- Comportement proche SQL standard
SELECT "column_name" || 'text';
```

### ORACLE

**Équivalent à** :
```
PIPES_AS_CONCAT,
ANSI_QUOTES,
IGNORE_SPACE,
NO_KEY_OPTIONS,
NO_TABLE_OPTIONS,
NO_FIELD_OPTIONS
```

**Usage** : **Émulation Oracle** pour faciliter les migrations.

```sql
SET sql_mode = 'ORACLE';

-- Comportement similaire à Oracle Database
SELECT name || ' ' || surname AS full_name
FROM employees;
```

**Limitations** : Émulation **partielle** uniquement. Oracle et MariaDB restent fondamentalement différents.

### MSSQL / DB2 / POSTGRESQL

Modes spécialisés pour la compatibilité avec d'autres SGBD.

```sql
-- Compatibilité Microsoft SQL Server
SET sql_mode = 'MSSQL';

-- Compatibilité IBM DB2
SET sql_mode = 'DB2';

-- Compatibilité PostgreSQL
SET sql_mode = 'POSTGRESQL';
```

⚠️ **Attention** : Ces modes ne garantissent **pas** une compatibilité totale. Ils facilitent la migration mais ne remplacent pas une adaptation approfondie.

---

## Configuration du sql_mode

### Au niveau global (serveur)

```ini
# Dans my.cnf
[mysqld]
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

```sql
-- Via SQL (temporaire jusqu'au redémarrage)
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';

-- Via SQL (persistant avec SET PERSIST)
SET PERSIST sql_mode = 'TRADITIONAL';
```

### Au niveau session (connexion)

```sql
-- Pour la connexion courante uniquement
SET SESSION sql_mode = 'TRADITIONAL';

-- Forme abrégée
SET sql_mode = 'TRADITIONAL';
```

### Ajouter ou retirer un mode

```sql
-- Ajouter ONLY_FULL_GROUP_BY au mode actuel
SET sql_mode = CONCAT(@@sql_mode, ',ONLY_FULL_GROUP_BY');

-- Retirer ONLY_FULL_GROUP_BY
SET sql_mode = REPLACE(@@sql_mode, ',ONLY_FULL_GROUP_BY', '');
SET sql_mode = REPLACE(@@sql_mode, 'ONLY_FULL_GROUP_BY,', '');
```

**Méthode plus propre avec sys_exec()** :

```sql
-- Fonction helper (à créer)
DELIMITER //
CREATE FUNCTION add_sql_mode(mode_to_add VARCHAR(255))
RETURNS VARCHAR(1024)
DETERMINISTIC
BEGIN
    DECLARE current_mode VARCHAR(1024);
    SET current_mode = @@sql_mode;

    IF FIND_IN_SET(mode_to_add, current_mode) = 0 THEN
        RETURN CONCAT(current_mode, ',', mode_to_add);
    ELSE
        RETURN current_mode;
    END IF;
END//
DELIMITER ;

-- Utilisation
SET sql_mode = add_sql_mode('ONLY_FULL_GROUP_BY');
```

---

## Impact sur le comportement SQL

### Insertion de données invalides

#### Mode permissif

```sql
SET sql_mode = '';

CREATE TABLE test_strict (
    age TINYINT,           -- -128 à 127
    name VARCHAR(5)
);

INSERT INTO test_strict VALUES (300, 'NomTropLong');
-- ⚠️ ACCEPTÉ avec warnings
-- age tronqué à 127
-- name tronqué à 'NomTr'

SELECT * FROM test_strict;
-- Résultat: 127 | NomTr (données corrompues)
```

#### Mode strict

```sql
SET sql_mode = 'STRICT_TRANS_TABLES';

INSERT INTO test_strict VALUES (300, 'NomTropLong');
-- ❌ ERREUR: Out of range value for column 'age'
-- Transaction annulée, aucune donnée insérée
```

### Division par zéro

```sql
-- Mode permissif
SET sql_mode = '';
SELECT 10 / 0;
-- Résultat: NULL (warning)

-- Mode strict
SET sql_mode = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO';
SELECT 10 / 0;
-- ❌ ERREUR: Division by 0
```

### Dates invalides

```sql
-- Mode permissif
SET sql_mode = '';
INSERT INTO events VALUES (1, '2025-02-30');  -- 30 février n'existe pas
-- ⚠️ ACCEPTÉ, converti en '0000-00-00'

-- Mode strict
SET sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE';
INSERT INTO events VALUES (1, '2025-02-30');
-- ❌ ERREUR: Invalid date
```

### GROUP BY non-standard

```sql
CREATE TABLE sales (
    region VARCHAR(50),
    salesperson VARCHAR(50),
    amount DECIMAL(10,2)
);

INSERT INTO sales VALUES
    ('Nord', 'Alice', 1000),
    ('Nord', 'Bob', 1500),
    ('Sud', 'Charlie', 2000);

-- Sans ONLY_FULL_GROUP_BY (non-standard, indéterministe)
SET sql_mode = '';

SELECT region, salesperson, SUM(amount)
FROM sales
GROUP BY region;
-- ⚠️ ACCEPTÉ
-- Résultat imprévisible pour 'salesperson' (Alice ou Bob ?)

-- Avec ONLY_FULL_GROUP_BY (standard SQL)
SET sql_mode = 'ONLY_FULL_GROUP_BY';

SELECT region, salesperson, SUM(amount)
FROM sales
GROUP BY region;
-- ❌ ERREUR: 'salesperson' isn't in GROUP BY

-- Solution correcte
SELECT region, SUM(amount) AS total
FROM sales
GROUP BY region;
-- ✅ OK
```

---

## Recommandations par environnement

### Production

```ini
# my.cnf - Production stricte
[mysqld]
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE
```

**Justification** :
- ✅ Intégrité des données garantie
- ✅ Détection précoce des bugs
- ✅ Conformité aux standards

### Développement

```ini
# my.cnf - Développement (encore plus strict)
[mysqld]
sql_mode = TRADITIONAL
```

**Justification** :
- ✅ Force les développeurs à écrire du code correct
- ✅ Détecte les problèmes avant la production
- ✅ Réduit la dette technique

### Migration depuis MySQL

```ini
# my.cnf - Compatibilité MySQL
[mysqld]
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

**Justification** :
- ✅ Compatible avec MySQL 5.7+
- ✅ Mode par défaut MySQL 8.0
- ✅ Facilite la migration

### Migration depuis Oracle

```sql
-- Session de migration
SET sql_mode = 'ORACLE';

-- Permet l'utilisation de || pour la concaténation
-- Double quotes pour les identifiants
-- Comportement proche Oracle
```

---

## Détection et résolution des problèmes

### Application legacy incompatible avec mode strict

**Symptôme** : Erreurs massives après activation de `STRICT_TRANS_TABLES`.

**Diagnostic** :

```sql
-- Tester en mode permissif
SET sql_mode = '';
-- Requête fonctionne

-- Tester en mode strict
SET sql_mode = 'STRICT_TRANS_TABLES';
-- Requête échoue
```

**Solutions** :

1. **Option 1 : Corriger le code applicatif** (recommandé)

```sql
-- Avant (code legacy)
INSERT INTO users (name) VALUES ('NomTropLongQuiDepasse');

-- Après (code corrigé)
INSERT INTO users (name) VALUES (SUBSTRING('NomTropLongQuiDepasse', 1, 10));
```

2. **Option 2 : Mode strict sélectif** (temporaire)

```sql
-- Désactiver strict uniquement pour certaines requêtes
SET sql_mode = '';
INSERT INTO users (name) VALUES ('NomTropLongQuiDepasse');
SET sql_mode = 'STRICT_TRANS_TABLES';
```

3. **Option 3 : Mode permissif global** (déconseillé)

```ini
# À éviter en production
[mysqld]
sql_mode = ''
```

### ONLY_FULL_GROUP_BY casse les requêtes

**Symptôme** : Erreur "isn't in GROUP BY" sur des requêtes legacy.

**Solution 1 : Corriger la requête** (recommandé)

```sql
-- Avant (incorrect)
SELECT category, name, COUNT(*)
FROM products
GROUP BY category;

-- Après (correct)
SELECT
    category,
    GROUP_CONCAT(name) AS names,  -- Agrégation
    COUNT(*) AS total
FROM products
GROUP BY category;
```

**Solution 2 : Désactiver temporairement**

```sql
SET sql_mode = REPLACE(@@sql_mode, 'ONLY_FULL_GROUP_BY', '');
```

### Incompatibilité avec imports

**Symptôme** : Import mysqldump échoue avec mode strict.

**Solution** :

```sql
-- Désactiver temporairement les checks
SET SESSION sql_mode = '';
SET SESSION foreign_key_checks = 0;
SET SESSION unique_checks = 0;

-- Import
SOURCE dump.sql;

-- Réactiver
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';
SET SESSION foreign_key_checks = 1;
SET SESSION unique_checks = 1;
```

---

## Vérification de compatibilité

### Script de test de compatibilité

```sql
-- Créer une table de test
CREATE TABLE compat_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tiny_col TINYINT,
    small_str VARCHAR(5),
    date_col DATE,
    calc_result DECIMAL(10,2)
);

-- Test 1 : Dépassement de capacité
INSERT INTO compat_test (tiny_col) VALUES (300);
-- Mode strict : ERREUR
-- Mode permissif : Tronqué à 127

-- Test 2 : Dépassement longueur
INSERT INTO compat_test (small_str) VALUES ('ChaineTropLongue');
-- Mode strict : ERREUR
-- Mode permissif : Tronqué à 'Chain'

-- Test 3 : Date invalide
INSERT INTO compat_test (date_col) VALUES ('2025-02-30');
-- Mode strict + NO_ZERO_DATE : ERREUR
-- Mode permissif : Converti en '0000-00-00'

-- Test 4 : Division par zéro
INSERT INTO compat_test (calc_result) VALUES (10 / 0);
-- Mode strict + ERROR_FOR_DIVISION_BY_ZERO : ERREUR
-- Mode permissif : NULL

-- Nettoyage
DROP TABLE compat_test;
```

---

## Modes par défaut MariaDB

### Évolution historique

| Version | sql_mode par défaut |
|---------|---------------------|
| MariaDB 10.2 | `NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |
| MariaDB 10.3 | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |
| MariaDB 10.4+ | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |
| **MariaDB 11.8** | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |

💡 **Bonne nouvelle** : MariaDB adopte un mode **strict par défaut** depuis la version 10.3, améliorant l'intégrité des données.

---

## Migration et compatibilité

### Migration MySQL → MariaDB

```sql
-- Vérifier le sql_mode MySQL
-- Sur MySQL
SHOW VARIABLES LIKE 'sql_mode';

-- Appliquer le même sur MariaDB (pour compatibilité)
SET GLOBAL sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
```

### Migration Oracle → MariaDB

```sql
-- Phase 1 : Mode Oracle pour import initial
SET sql_mode = 'ORACLE';

-- Importer les données...

-- Phase 2 : Tester progressivement avec mode MariaDB
SET sql_mode = 'TRADITIONAL';

-- Identifier et corriger les incompatibilités
```

### Migration PostgreSQL → MariaDB

```sql
-- Activer modes compatibles PostgreSQL
SET sql_mode = 'POSTGRESQL';

-- Implique :
-- - PIPES_AS_CONCAT (|| = concat)
-- - ANSI_QUOTES (" = identifiant)
```

---

## ✅ Points clés à retenir

- **sql_mode** contrôle la rigueur et la compatibilité SQL de MariaDB
- **Deux philosophies** : Mode strict (TRADITIONAL) vs mode permissif ('')
- **Production** : Toujours utiliser un mode strict pour l'intégrité des données
- **Modes essentiels** : `STRICT_TRANS_TABLES`, `ERROR_FOR_DIVISION_BY_ZERO`, `NO_AUTO_CREATE_USER`
- **ONLY_FULL_GROUP_BY** : Force le standard SQL pour les GROUP BY
- **Modes composés** : TRADITIONAL (strict max), ANSI, ORACLE, POSTGRESQL
- **Configuration** : Global (my.cnf), session (SET), persistant (SET PERSIST)
- **Compatibilité** : Modes facilitent migrations mais ne garantissent pas compatibilité totale
- **Tests** : Toujours tester en mode strict avant mise en production
- **Migration** : Activer progressivement les modes stricts
- **Défaut 11.8** : Mode déjà relativement strict par défaut
- **Documentation** : Commenter les choix de sql_mode dans my.cnf

---

## 🔗 Ressources et références

- [📖 Documentation officielle - SQL Mode](https://mariadb.com/kb/en/sql-mode/)
- [📖 Documentation officielle - sql_mode Full List](https://mariadb.com/kb/en/sql_mode-full-list/)
- [📖 MySQL vs MariaDB - SQL Mode Differences](https://mariadb.com/kb/en/mariadb-vs-mysql-compatibility/)
- [📖 Oracle Compatibility Mode](https://mariadb.com/kb/en/sql_modeoracle-from-mariadb-103/)
- [📖 ANSI SQL Compatibility](https://mariadb.com/kb/en/sql-mode-ansi/)

---

## ➡️ Section suivante

**[11.4 Gestion des logs](./04-gestion-logs.md)** : Configuration et exploitation des différents types de logs MariaDB (error, slow query, general, binary) pour le diagnostic, l'audit et la réplication.

---

**💡 Conseil final** : Le sql_mode n'est pas juste un détail technique — c'est une décision stratégique sur la **qualité de vos données**. Préférez toujours la rigueur à la permissivité. Vos futurs DBA vous remercieront ! 🛡️🎯

⏭️ [Gestion des logs](/11-administration-configuration/04-gestion-logs.md)

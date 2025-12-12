🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Introduction au langage SQL

> **Niveau** : Débutant
> **Durée estimée** : 1-1.5 heures
> **Prérequis** : Comprendre les concepts de base de données relationnelles (tables, lignes, colonnes)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre ce qu'est le SQL et son rôle dans les bases de données relationnelles
- Identifier les différentes catégories de commandes SQL (DDL, DML, DCL, TCL)
- Reconnaître la syntaxe de base d'une commande SQL
- Appliquer les conventions de nommage et bonnes pratiques
- Écrire vos premières commandes SQL simples

---

## Introduction

Le **SQL (Structured Query Language)** est le langage universel pour interagir avec les bases de données relationnelles. Créé dans les années 1970 chez IBM, il est devenu le standard incontournable pour manipuler des données structurées.

### Pourquoi SQL est-il si important ?

Le SQL est :
- **Standardisé** : Normalisé par l'ISO/ANSI, il fonctionne sur presque tous les SGBD
- **Déclaratif** : Vous décrivez *ce que* vous voulez, pas *comment* l'obtenir
- **Puissant** : Permet de manipuler des millions de lignes en quelques lignes de code
- **Universel** : Utilisé dans tous les domaines (finance, santé, e-commerce, IA...)

💡 **Bon à savoir** : Bien que standardisé, chaque SGBD (MariaDB, PostgreSQL, Oracle) apporte ses propres extensions. MariaDB reste très proche du standard SQL tout en offrant des fonctionnalités avancées.

---

## Histoire et standards SQL

### Chronologie

| Année | Version | Nouveautés principales |
|-------|---------|------------------------|
| 1986 | SQL-86 | Premier standard ANSI |
| 1992 | SQL-92 | Base du SQL moderne (JOIN, etc.) |
| 1999 | SQL:1999 | Déclencheurs, récursivité |
| 2003 | SQL:2003 | XML, window functions |
| 2008 | SQL:2008 | TRUNCATE, MERGE |
| 2011 | SQL:2011 | Données temporelles |
| 2016 | SQL:2016 | JSON, pattern matching |

**MariaDB 11.8** implémente la majorité des fonctionnalités SQL:2016 et ajoute ses propres extensions (VECTOR, JSON amélioré, etc.).

### Dialectes SQL

Bien que standardisé, le SQL existe en plusieurs "dialectes" :
- **MySQL/MariaDB** : Focus performance et web
- **PostgreSQL** : Conformité stricte aux standards
- **Oracle PL/SQL** : Orienté entreprise
- **SQL Server T-SQL** : Écosystème Microsoft

🆕 **MariaDB 11.8** : Améliore la compatibilité avec le standard SQL tout en conservant la compatibilité avec MySQL.

---

## Les catégories de commandes SQL

Le SQL se divise en plusieurs catégories selon le type d'opération effectuée.

### 1. DDL - Data Definition Language (Définition)

**Rôle** : Créer, modifier ou supprimer la structure des objets de la base de données.

| Commande | Description | Exemple |
|----------|-------------|---------|
| `CREATE` | Créer un objet (base, table, index) | `CREATE TABLE users (...)` |
| `ALTER` | Modifier la structure d'un objet | `ALTER TABLE users ADD email VARCHAR(255)` |
| `DROP` | Supprimer un objet | `DROP TABLE users` |
| `TRUNCATE` | Vider une table (plus rapide que DELETE) | `TRUNCATE TABLE logs` |
| `RENAME` | Renommer un objet | `RENAME TABLE old_name TO new_name` |

```sql
-- Exemple DDL : Création d'une table
CREATE TABLE produits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2),
    stock INT DEFAULT 0
);
```

💡 **Important** : Les commandes DDL sont généralement **auto-commit**, c'est-à-dire qu'elles ne peuvent pas être annulées avec `ROLLBACK`.

---

### 2. DML - Data Manipulation Language (Manipulation)

**Rôle** : Manipuler les données à l'intérieur des tables.

| Commande | Description | Exemple |
|----------|-------------|---------|
| `SELECT` | Consulter/lire des données | `SELECT * FROM users` |
| `INSERT` | Ajouter de nouvelles lignes | `INSERT INTO users VALUES (...)` |
| `UPDATE` | Modifier des lignes existantes | `UPDATE users SET email = ...` |
| `DELETE` | Supprimer des lignes | `DELETE FROM users WHERE id = 5` |

```sql
-- Exemple DML : Insertion de données
INSERT INTO produits (nom, prix, stock)
VALUES ('Ordinateur portable', 899.99, 15);

-- Exemple DML : Consultation
SELECT nom, prix
FROM produits
WHERE stock > 0;

-- Exemple DML : Mise à jour
UPDATE produits
SET prix = prix * 0.9
WHERE stock > 50;

-- Exemple DML : Suppression
DELETE FROM produits
WHERE stock = 0 AND prix < 10;
```

💡 **Astuce** : Les commandes DML peuvent être annulées avec `ROLLBACK` si elles sont dans une transaction.

---

### 3. DCL - Data Control Language (Contrôle)

**Rôle** : Gérer les permissions et la sécurité.

| Commande | Description | Exemple |
|----------|-------------|---------|
| `GRANT` | Donner des privilèges à un utilisateur | `GRANT SELECT ON db.* TO 'user'@'localhost'` |
| `REVOKE` | Retirer des privilèges | `REVOKE INSERT ON db.* FROM 'user'@'localhost'` |

```sql
-- Exemple DCL : Attribution de droits
-- Créer un utilisateur en lecture seule
CREATE USER 'lecteur'@'localhost' IDENTIFIED BY 'motdepasse';

-- Lui donner uniquement le droit de lecture
GRANT SELECT ON formation_mariadb.* TO 'lecteur'@'localhost';

-- Retirer un droit
REVOKE SELECT ON formation_mariadb.logs FROM 'lecteur'@'localhost';
```

⚠️ **Sécurité** : Les commandes DCL doivent être utilisées avec précaution. Un mauvais `GRANT` peut exposer vos données.

---

### 4. TCL - Transaction Control Language (Transactions)

**Rôle** : Gérer les transactions pour garantir l'intégrité des données (ACID).

| Commande | Description | Exemple |
|----------|-------------|---------|
| `START TRANSACTION` / `BEGIN` | Démarrer une transaction | `START TRANSACTION;` |
| `COMMIT` | Valider la transaction | `COMMIT;` |
| `ROLLBACK` | Annuler la transaction | `ROLLBACK;` |
| `SAVEPOINT` | Créer un point de sauvegarde | `SAVEPOINT sp1;` |

```sql
-- Exemple TCL : Transaction bancaire
START TRANSACTION;

-- Débiter le compte A
UPDATE comptes SET solde = solde - 100 WHERE id = 1;

-- Créditer le compte B
UPDATE comptes SET solde = solde + 100 WHERE id = 2;

-- Si tout est OK, valider
COMMIT;

-- Sinon, annuler
-- ROLLBACK;
```

💡 **ACID** : Les transactions garantissent les propriétés **A**tomicité, **C**ohérence, **I**solation, **D**urabilité (voir chapitre 6).

---

## Syntaxe de base du SQL

### Structure d'une commande SQL

Une commande SQL suit généralement cette structure :

```sql
VERBE [MODIFICATEUR] OBJET [CLAUSES] [CONDITIONS] [OPTIONS];
```

**Exemple décomposé :**

```sql
SELECT nom, prix                    -- VERBE + colonnes
FROM produits                       -- OBJET (table)
WHERE categorie = 'électronique'   -- CONDITION
ORDER BY prix DESC                  -- CLAUSE de tri
LIMIT 10;                          -- OPTION de limitation
```

### Règles syntaxiques importantes

1. **Point-virgule** : Termine une commande SQL (`;`)
2. **Insensible à la casse** : `SELECT` = `select` = `Select`
3. **Espaces blancs** : Ignorés (sauf dans les chaînes)
4. **Commentaires** :
   - Une ligne : `-- Commentaire`
   - Multiligne : `/* Commentaire */`
   - MySQL/MariaDB : `# Commentaire`

```sql
-- Commentaire sur une ligne

/*
   Commentaire
   sur plusieurs lignes
*/

SELECT
    id,           -- Identifiant unique
    nom,          -- Nom du produit
    prix          -- Prix en euros
FROM produits;    # Table des produits
```

---

## Conventions de nommage et bonnes pratiques

### Conventions pour les identifiants

Les noms d'objets (bases, tables, colonnes) doivent suivre ces règles :

✅ **Bonnes pratiques :**
- Utiliser des **noms descriptifs** : `clients` plutôt que `c`
- Préférer le **snake_case** : `date_creation` plutôt que `dateCreation`
- Utiliser le **singulier ou pluriel** de façon cohérente : `client` ou `clients`
- Éviter les **mots réservés SQL** : `user`, `order`, `table`
- Limiter à **64 caractères** maximum (MariaDB)

❌ **À éviter :**
- Caractères spéciaux : `@`, `#`, `$`, espaces
- Accents : `réservé`, `numéro`
- Débuter par un chiffre : `2024_ventes`

```sql
-- ✅ BONS exemples
CREATE TABLE clients (
    client_id INT PRIMARY KEY,
    nom_complet VARCHAR(100),
    date_inscription DATE
);

-- ❌ MAUVAIS exemples (mais syntaxiquement corrects)
CREATE TABLE `2024-données` (
    `N°` INT,
    `Prénom&Nom` VARCHAR(100),
    `date création` DATE
);
```

💡 **Astuce** : Si vous devez utiliser un mot réservé ou des caractères spéciaux, entourez-le de backticks : `` `order` ``, `` `first-name` ``

---

### Style de code SQL

Pour améliorer la lisibilité :

**Style recommandé :**

```sql
-- Mots-clés en MAJUSCULES
-- Identifiants en minuscules
-- Indentation claire

SELECT
    c.nom_client,
    c.email,
    COUNT(co.commande_id) AS nombre_commandes,
    SUM(co.montant_total) AS total_achats
FROM clients AS c
INNER JOIN commandes AS co
    ON c.client_id = co.client_id
WHERE co.date_commande >= '2024-01-01'
GROUP BY c.client_id
HAVING total_achats > 1000
ORDER BY total_achats DESC
LIMIT 10;
```

**Points clés :**
- Mots-clés SQL en **MAJUSCULES** (SELECT, FROM, WHERE)
- Noms de tables/colonnes en **minuscules**
- **Une clause par ligne** pour les requêtes complexes
- **Indentation** pour montrer la hiérarchie
- **Alias explicites** : `AS c`, `AS nombre_commandes`

---

## Types de requêtes SQL

### Requêtes de définition (DDL)

Créent la structure :

```sql
-- Créer une base de données
CREATE DATABASE ma_boutique
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Utiliser une base de données
USE ma_boutique;

-- Créer une table
CREATE TABLE categories (
    categorie_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50) UNIQUE NOT NULL,
    description TEXT
);
```

---

### Requêtes de manipulation (DML)

Travaillent avec les données :

```sql
-- INSERT : Ajouter des données
INSERT INTO categories (nom, description)
VALUES ('Électronique', 'Appareils électroniques et accessoires');

-- SELECT : Lire des données
SELECT * FROM categories;

-- UPDATE : Modifier des données
UPDATE categories
SET description = 'Tous les appareils électroniques'
WHERE nom = 'Électronique';

-- DELETE : Supprimer des données
DELETE FROM categories
WHERE categorie_id = 5;
```

---

### Requêtes de contrôle (DCL)

Gèrent les permissions :

```sql
-- Créer un utilisateur
CREATE USER 'dev_readonly'@'localhost'
IDENTIFIED BY 'P@ssw0rd!';

-- Donner des droits de lecture
GRANT SELECT ON ma_boutique.*
TO 'dev_readonly'@'localhost';

-- Appliquer les changements
FLUSH PRIVILEGES;
```

---

## MariaDB et le standard SQL

### Conformité aux standards

MariaDB 11.8 respecte la majorité du standard SQL:2016 :

✅ **Fonctionnalités SQL standard supportées :**
- Requêtes relationnelles complètes (JOIN, UNION, etc.)
- Sous-requêtes et requêtes corrélées
- Window Functions (ROW_NUMBER, RANK, etc.)
- Common Table Expressions (CTE) avec WITH
- Transactions ACID complètes
- Contraintes d'intégrité référentielle

🆕 **Extensions MariaDB 11.8 :**
- Type **VECTOR** pour l'IA/ML (non standard)
- Fonctions JSON étendues
- System-Versioned Tables (tables temporelles)
- Support **RETURNING** pour INSERT/UPDATE/DELETE
- Plugin-based authentication

```sql
-- Exemple d'extension MariaDB : RETURNING
-- Retourne les lignes insérées immédiatement
INSERT INTO produits (nom, prix)
VALUES ('Nouveau produit', 49.99)
RETURNING produit_id, nom, prix;

-- Résultat immédiat :
-- produit_id | nom              | prix
-- 101        | Nouveau produit  | 49.99
```

---

### Différences avec d'autres SGBD

| Fonctionnalité | MariaDB | PostgreSQL | MySQL | SQL Server |
|----------------|---------|------------|-------|------------|
| **AUTO_INCREMENT** | ✅ Oui | `SERIAL` | ✅ Oui | `IDENTITY` |
| **Backticks** | ✅ `` `table` `` | `"table"` | ✅ `` `table` `` | `[table]` |
| **LIMIT** | ✅ `LIMIT 10` | ✅ `LIMIT 10` | ✅ `LIMIT 10` | `TOP 10` |
| **Chaînes** | `'simple'` ou `"double"` | `'simple'` uniquement | `'simple'` ou `"double"` | `'simple'` uniquement |
| **Concat** | `CONCAT()` | ` | |` ou `CONCAT()` | `CONCAT()` | `+` |

💡 **Portabilité** : Si vous écrivez du SQL destiné à plusieurs SGBD, respectez le standard strict (guillemets simples, pas de backticks, etc.).

---

## Exemples pratiques : Premiers pas

### Exemple 1 : Créer une base et se connecter

```sql
-- Créer une nouvelle base de données
CREATE DATABASE IF NOT EXISTS librairie
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Afficher toutes les bases
SHOW DATABASES;

-- Se positionner sur la base
USE librairie;

-- Vérifier la base active
SELECT DATABASE();
```

**Résultat attendu :**
```
+------------+
| DATABASE() |
+------------+
| librairie  |
+------------+
```

---

### Exemple 2 : Créer une première table

```sql
-- Table simple pour stocker des livres
CREATE TABLE livres (
    -- Clé primaire auto-incrémentée
    livre_id INT PRIMARY KEY AUTO_INCREMENT,

    -- Titre du livre (obligatoire)
    titre VARCHAR(200) NOT NULL,

    -- ISBN unique
    isbn VARCHAR(13) UNIQUE,

    -- Prix avec 2 décimales
    prix DECIMAL(6,2) CHECK (prix >= 0),

    -- Nombre d'exemplaires en stock
    stock INT DEFAULT 0,

    -- Date d'ajout au catalogue
    date_ajout DATE DEFAULT (CURRENT_DATE)
);

-- Afficher la structure de la table
DESCRIBE livres;
-- ou : SHOW COLUMNS FROM livres;
```

**Résultat de DESCRIBE :**
```
+------------+--------------+------+-----+------------+----------------+
| Field      | Type         | Null | Key | Default    | Extra          |
+------------+--------------+------+-----+------------+----------------+
| livre_id   | int(11)      | NO   | PRI | NULL       | auto_increment |
| titre      | varchar(200) | NO   |     | NULL       |                |
| isbn       | varchar(13)  | YES  | UNI | NULL       |                |
| prix       | decimal(6,2) | YES  |     | NULL       |                |
| stock      | int(11)      | YES  |     | 0          |                |
| date_ajout | date         | YES  |     | curdate()  |                |
+------------+--------------+------+-----+------------+----------------+
```

---

### Exemple 3 : Insérer et consulter des données

```sql
-- Insérer un livre
INSERT INTO livres (titre, isbn, prix, stock)
VALUES ('Le Seigneur des Anneaux', '9782266154345', 29.90, 12);

-- Insérer plusieurs livres en une fois
INSERT INTO livres (titre, isbn, prix, stock) VALUES
    ('1984', '9782072730013', 8.40, 25),
    ('Le Petit Prince', '9782070408504', 5.90, 50),
    ('Harry Potter', '9782070584628', 22.90, 8);

-- Consulter tous les livres
SELECT * FROM livres;

-- Consulter uniquement certaines colonnes
SELECT titre, prix, stock
FROM livres;

-- Consulter avec un filtre
SELECT titre, prix
FROM livres
WHERE prix < 10;

-- Consulter avec tri
SELECT titre, stock
FROM livres
ORDER BY stock DESC;
```

**Résultat de la dernière requête :**
```
+--------------------+-------+
| titre              | stock |
+--------------------+-------+
| Le Petit Prince    | 50    |
| 1984               | 25    |
| Le Seigneur...     | 12    |
| Harry Potter       | 8     |
+--------------------+-------+
```

---

## Client MariaDB en ligne de commande

### Connexion et commandes de base

```bash
# Se connecter au serveur MariaDB
mariadb -u root -p

# Se connecter directement à une base
mariadb -u root -p librairie

# Exécuter une commande SQL depuis le shell
mariadb -u root -p -e "SELECT DATABASE();"

# Exécuter un script SQL
mariadb -u root -p < script.sql
```

### Commandes internes du client

Une fois connecté au client `mariadb`, ces commandes sont disponibles :

| Commande | Description | Exemple |
|----------|-------------|---------|
| `\h` ou `help` | Afficher l'aide | `\h` |
| `\s` ou `status` | État du serveur | `\s` |
| `\u` | Changer de base | `\u librairie` |
| `\q` ou `exit` | Quitter | `\q` |
| `\c` | Annuler la commande courante | `\c` |
| `\G` | Affichage vertical | `SELECT * FROM livres\G` |
| `source` | Exécuter un fichier SQL | `source backup.sql` |

```sql
-- Exemple d'affichage vertical avec \G
SELECT * FROM livres WHERE livre_id = 1\G

-- Résultat :
-- *************************** 1. row ***************************
--   livre_id: 1
--      titre: Le Seigneur des Anneaux
--       isbn: 9782266154345
--       prix: 29.90
--      stock: 12
-- date_ajout: 2025-12-12
```

---

## Gestion des erreurs SQL

### Erreurs courantes pour les débutants

```sql
-- ❌ ERREUR : Table inexistante
SELECT * FROM livreees;
-- ERROR 1146: Table 'librairie.livreees' doesn't exist

-- ❌ ERREUR : Colonne inexistante
SELECT author FROM livres;
-- ERROR 1054: Unknown column 'author' in 'field list'

-- ❌ ERREUR : Syntaxe invalide
SELECT * FORM livres;
-- ERROR 1064: You have an error in your SQL syntax

-- ❌ ERREUR : Division par zéro
SELECT prix / 0 FROM livres;
-- Résultat : NULL (pas d'erreur, mais attention !)

-- ❌ ERREUR : Violation de contrainte
INSERT INTO livres (livre_id, titre)
VALUES (1, 'Doublon');
-- ERROR 1062: Duplicate entry '1' for key 'PRIMARY'
```

💡 **Astuce** : Lisez toujours le numéro et le message d'erreur. Ils vous guident vers le problème.

---

## ✅ Points clés à retenir

- Le **SQL** est le langage universel pour les bases de données relationnelles
- **Quatre catégories** principales :
  - **DDL** : Structure (CREATE, ALTER, DROP)
  - **DML** : Données (SELECT, INSERT, UPDATE, DELETE)
  - **DCL** : Sécurité (GRANT, REVOKE)
  - **TCL** : Transactions (COMMIT, ROLLBACK)
- Le SQL est **déclaratif** : on décrit *quoi*, pas *comment*
- MariaDB respecte le **standard SQL** tout en offrant des extensions
- Conventions : **mots-clés en MAJUSCULES**, **identifiants en snake_case**
- Les commandes DDL sont **auto-commit** (pas de ROLLBACK possible)
- Toujours **terminer par un point-virgule** (`;`)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 SQL Statements Structure](https://mariadb.com/kb/en/sql-statements-structure/)
- [📖 SQL Language Structure](https://mariadb.com/kb/en/sql-language-structure/)
- [📖 Comment Syntax](https://mariadb.com/kb/en/comment-syntax/)
- [📖 Identifier Names](https://mariadb.com/kb/en/identifier-names/)

### Standards SQL
- [ISO/IEC 9075 (SQL Standard)](https://www.iso.org/standard/63555.html)
- [SQL Tutorial - W3Schools](https://www.w3schools.com/sql/)

### Outils
- [DB Fiddle](https://www.db-fiddle.com/) - Tester du SQL en ligne
- [SQLFormat](https://sqlformat.org/) - Formater du code SQL

---

## ➡️ Section suivante

**[2.2 Types de données MariaDB](/02-bases-du-sql/02-types-de-donnees.md)**

Découvrez en détail tous les types de données disponibles dans MariaDB 11.8 : numériques (INT, DECIMAL, FLOAT), texte (VARCHAR, TEXT), temporels (DATE, DATETIME, TIMESTAMP), binaires (BLOB) et types spécifiques MariaDB (JSON, UUID, VECTOR 🆕).

---


⏭️ [Types de données MariaDB](/02-bases-du-sql/02-types-de-donnees.md)

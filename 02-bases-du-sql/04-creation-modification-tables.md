🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Création et modification de tables (CREATE, ALTER, DROP)

> **Niveau** : Débutant
> **Durée estimée** : 1.5 heures
> **Prérequis** : Section 2.3 (Création et gestion des bases de données)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Créer des tables avec CREATE TABLE et définir leurs colonnes
- Choisir les options appropriées (ENGINE, CHARSET, AUTO_INCREMENT)
- Modifier la structure d'une table avec ALTER TABLE
- Supprimer des tables en toute sécurité avec DROP TABLE
- Utiliser SHOW CREATE TABLE et DESCRIBE pour inspecter les tables
- Appliquer les bonnes pratiques de conception de tables
- Éviter les pièges courants de modification de structure

---

## Introduction

Une **table** est la structure fondamentale qui stocke les données dans une base de données relationnelle. Elle est composée de :

- **Colonnes** (attributs) : Définissent le type de données stocké
- **Lignes** (enregistrements) : Contiennent les données réelles
- **Contraintes** : Garantissent l'intégrité des données

```
Table: clients
+------------+-------------+------------------+
| client_id  | nom         | email            |  ← Colonnes
+------------+-------------+------------------+
| 1          | Alice       | alice@email.com  |  ← Ligne 1
| 2          | Bob         | bob@email.com    |  ← Ligne 2
| 3          | Charlie     | charlie@email.com|  ← Ligne 3
+------------+-------------+------------------+
```

---

## CREATE TABLE - Créer une table

### Syntaxe de base

```sql
CREATE TABLE nom_table (
    colonne1 TYPE [CONTRAINTES],
    colonne2 TYPE [CONTRAINTES],
    ...
    [CONTRAINTES DE TABLE]
) [OPTIONS DE TABLE];
```

### Premier exemple simple

```sql
-- Créer une table clients basique
CREATE TABLE clients (
    client_id INT,
    nom VARCHAR(100),
    email VARCHAR(255),
    date_inscription DATE
);

-- Vérifier la création
SHOW TABLES;

-- Voir la structure
DESCRIBE clients;
-- ou : DESC clients;
-- ou : SHOW COLUMNS FROM clients;

-- Résultat :
-- +------------------+--------------+------+-----+---------+-------+
-- | Field            | Type         | Null | Key | Default | Extra |
-- +------------------+--------------+------+-----+---------+-------+
-- | client_id        | int(11)      | YES  |     | NULL    |       |
-- | nom              | varchar(100) | YES  |     | NULL    |       |
-- | email            | varchar(255) | YES  |     | NULL    |       |
-- | date_inscription | date         | YES  |     | NULL    |       |
-- +------------------+--------------+------+-----+---------+-------+
```

### CREATE TABLE IF NOT EXISTS

```sql
-- ✅ Bonne pratique : Éviter les erreurs si la table existe
CREATE TABLE IF NOT EXISTS clients (
    client_id INT,
    nom VARCHAR(100),
    email VARCHAR(255)
);

-- Si la table existe déjà : Pas d'erreur, juste un warning
-- Si la table n'existe pas : Elle est créée
```

---

## Définition des colonnes

### Syntaxe d'une colonne

```sql
nom_colonne TYPE [NOT NULL] [DEFAULT valeur] [AUTO_INCREMENT] [COMMENT 'description']
```

### Types de données courants

```sql
CREATE TABLE exemple_types (
    -- Entiers
    id INT PRIMARY KEY AUTO_INCREMENT,
    age TINYINT UNSIGNED,                       -- 0 à 255
    quantite SMALLINT,                          -- -32,768 à 32,767
    population BIGINT,                          -- Très grands nombres

    -- Décimaux
    prix DECIMAL(10,2),                         -- Précision exacte pour argent
    temperature FLOAT,                          -- Approximatif
    coordonnee DOUBLE,                          -- Approximatif haute précision

    -- Texte
    code_pays CHAR(2),                          -- Longueur fixe
    nom VARCHAR(100) NOT NULL,                  -- Longueur variable
    description TEXT,                           -- Texte long

    -- Temporels
    date_naissance DATE,                        -- Date seule
    heure_ouverture TIME,                       -- Heure seule
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                        ON UPDATE CURRENT_TIMESTAMP,

    -- Binaires
    avatar BLOB,                                -- Données binaires

    -- Spécifiques
    preferences JSON,                           -- Données JSON
    ip_address INET6,                           -- Adresse IP

    -- Énumérations
    statut ENUM('actif', 'inactif', 'suspendu') DEFAULT 'actif'
);
```

### Attributs de colonnes

#### NOT NULL - Valeur obligatoire

```sql
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,                  -- Obligatoire
    description TEXT,                           -- Optionnel (peut être NULL)
    prix DECIMAL(10,2) NOT NULL,                -- Obligatoire
    stock INT NOT NULL DEFAULT 0                -- Obligatoire avec valeur par défaut
);

-- ✅ Insertion valide
INSERT INTO produits (nom, prix) VALUES ('Livre', 19.99);

-- ❌ Erreur : nom manquant
INSERT INTO produits (prix) VALUES (19.99);
-- ERROR: Field 'nom' doesn't have a default value
```

#### DEFAULT - Valeur par défaut

```sql
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    statut VARCHAR(20) DEFAULT 'en_attente',    -- Valeur par défaut
    date_commande DATETIME DEFAULT CURRENT_TIMESTAMP,
    montant DECIMAL(10,2) DEFAULT 0.00,
    livraison_express BOOLEAN DEFAULT FALSE,
    commentaire TEXT DEFAULT NULL               -- NULL est le défaut implicite
);

-- Insertion sans spécifier les colonnes avec DEFAULT
INSERT INTO commandes (client_id, montant) VALUES (1001, 150.00);

-- Les valeurs par défaut sont appliquées :
-- statut = 'en_attente'
-- date_commande = date/heure actuelle
-- livraison_express = FALSE
-- commentaire = NULL
```

#### AUTO_INCREMENT - Incrémentation automatique

```sql
CREATE TABLE utilisateurs (
    utilisateur_id INT PRIMARY KEY AUTO_INCREMENT,  -- Clé primaire auto
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Insertion sans spécifier l'ID
INSERT INTO utilisateurs (nom, email) VALUES
    ('Alice', 'alice@example.com'),             -- utilisateur_id = 1
    ('Bob', 'bob@example.com'),                 -- utilisateur_id = 2
    ('Charlie', 'charlie@example.com');         -- utilisateur_id = 3

-- Récupérer le dernier ID inséré
SELECT LAST_INSERT_ID();
-- Résultat : 3

-- Modifier la valeur de départ d'AUTO_INCREMENT
ALTER TABLE utilisateurs AUTO_INCREMENT = 1000;

-- Prochaine insertion aura ID = 1000
INSERT INTO utilisateurs (nom, email) VALUES ('David', 'david@example.com');
```

#### COMMENT - Documentation

```sql
CREATE TABLE produits_documentes (
    produit_id INT PRIMARY KEY AUTO_INCREMENT
        COMMENT 'Identifiant unique du produit',

    nom VARCHAR(100) NOT NULL
        COMMENT 'Nom commercial du produit',

    prix_ht DECIMAL(10,2) NOT NULL
        COMMENT 'Prix hors taxes en euros',

    tva DECIMAL(4,2) DEFAULT 20.00
        COMMENT 'Taux de TVA en pourcentage'
) COMMENT = 'Table principale des produits vendus';

-- Voir les commentaires
SHOW CREATE TABLE produits_documentes\G
```

---

## Contraintes de table

### PRIMARY KEY - Clé primaire

```sql
-- Méthode 1 : Inline (sur la colonne)
CREATE TABLE clients_v1 (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100)
);

-- Méthode 2 : Contrainte de table (recommandé si composite)
CREATE TABLE clients_v2 (
    client_id INT AUTO_INCREMENT,
    nom VARCHAR(100),
    PRIMARY KEY (client_id)
);

-- Clé primaire composite (plusieurs colonnes)
CREATE TABLE inscriptions_cours (
    etudiant_id INT,
    cours_id INT,
    date_inscription DATE,
    PRIMARY KEY (etudiant_id, cours_id)        -- Clé composite
);
```

### UNIQUE - Valeur unique

```sql
CREATE TABLE utilisateurs_unique (
    utilisateur_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,         -- Email unique (inline)
    telephone VARCHAR(20),
    UNIQUE KEY idx_telephone (telephone)        -- Unique (contrainte de table)
);

-- ✅ Insertion valide
INSERT INTO utilisateurs_unique (email, telephone) VALUES
    ('alice@example.com', '0601020304');

-- ❌ Erreur : Email déjà existant
INSERT INTO utilisateurs_unique (email) VALUES
    ('alice@example.com');
-- ERROR: Duplicate entry 'alice@example.com' for key 'email'
```

### FOREIGN KEY - Clé étrangère

```sql
-- Table parent
CREATE TABLE categories (
    categorie_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50) UNIQUE NOT NULL
);

-- Table enfant avec clé étrangère
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    categorie_id INT NOT NULL,

    -- Clé étrangère vers categories
    FOREIGN KEY (categorie_id)
        REFERENCES categories(categorie_id)
        ON DELETE RESTRICT                      -- Empêche suppression si référencé
        ON UPDATE CASCADE                       -- Mise à jour en cascade
);

-- Avec nom de contrainte explicite
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    CONSTRAINT fk_commandes_client
        FOREIGN KEY (client_id)
        REFERENCES clients(client_id)
        ON DELETE CASCADE                       -- Suppression en cascade
        ON UPDATE CASCADE
);
```

#### Options ON DELETE et ON UPDATE

| Option | Comportement |
|--------|--------------|
| **RESTRICT** | Empêche la suppression/modification (défaut) |
| **CASCADE** | Supprime/modifie les lignes liées en cascade |
| **SET NULL** | Définit la clé étrangère à NULL |
| **NO ACTION** | Comme RESTRICT (vérifié à la fin de la transaction) |
| **SET DEFAULT** | Définit à la valeur par défaut (peu supporté) |

```sql
-- Exemple complet avec différentes options
CREATE TABLE auteurs (
    auteur_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE livres (
    livre_id INT PRIMARY KEY AUTO_INCREMENT,
    titre VARCHAR(200) NOT NULL,
    auteur_id INT,
    categorie_id INT,

    -- CASCADE : Si auteur supprimé, livre supprimé aussi
    FOREIGN KEY (auteur_id)
        REFERENCES auteurs(auteur_id)
        ON DELETE CASCADE,

    -- SET NULL : Si catégorie supprimée, categorie_id devient NULL
    FOREIGN KEY (categorie_id)
        REFERENCES categories(categorie_id)
        ON DELETE SET NULL
);
```

### CHECK - Contrainte de vérification

```sql
CREATE TABLE produits_check (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL,
    reduction DECIMAL(5,2),

    -- Contraintes CHECK
    CHECK (prix > 0),                           -- Prix positif
    CHECK (stock >= 0),                         -- Stock non négatif
    CHECK (reduction >= 0 AND reduction <= 100) -- Réduction entre 0 et 100%
);

-- ✅ Insertion valide
INSERT INTO produits_check (nom, prix, stock, reduction)
VALUES ('Livre', 19.99, 10, 15.00);

-- ❌ Erreur : Prix négatif
INSERT INTO produits_check (nom, prix, stock)
VALUES ('Test', -10.00, 5);
-- ERROR: Check constraint 'produits_check_chk_1' is violated
```

---

## Options de table

### ENGINE - Moteur de stockage

```sql
-- InnoDB : Moteur par défaut (recommandé)
CREATE TABLE clients_innodb (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100)
) ENGINE=InnoDB;

-- MyISAM : Ancien moteur (pas de FK, pas de transactions)
CREATE TABLE logs_myisam (
    log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message TEXT
) ENGINE=MyISAM;

-- Aria : Successeur de MyISAM
CREATE TABLE cache_aria (
    cache_key VARCHAR(255) PRIMARY KEY,
    cache_value TEXT
) ENGINE=Aria;

-- Memory : Table en RAM (perdue au redémarrage)
CREATE TABLE session_temp (
    session_id VARCHAR(64) PRIMARY KEY,
    data TEXT
) ENGINE=Memory;
```

💡 **Recommandation** : Utilisez **InnoDB** pour toutes les tables sauf cas spécifiques (analytique → ColumnStore, cache → Memory).

### CHARACTER SET et COLLATE

```sql
-- Spécifier charset et collation au niveau table
CREATE TABLE articles (
    article_id INT PRIMARY KEY AUTO_INCREMENT,
    titre VARCHAR(200) NOT NULL,
    contenu TEXT
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Charset différent par colonne (rare)
CREATE TABLE multilangue (
    texte_fr TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
    texte_en TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
    code CHAR(10) CHARACTER SET latin1
);
```

🆕 **MariaDB 11.8** : Si non spécifié, hérite de **utf8mb4** et **utf8mb4_uca_1400_ai_ci** de la base.

### AUTO_INCREMENT - Valeur initiale

```sql
-- Définir la valeur de départ
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL
) AUTO_INCREMENT = 10000;

-- La première insertion aura commande_id = 10000
INSERT INTO commandes (client_id) VALUES (1001);
SELECT * FROM commandes;
-- commande_id: 10000
```

### ROW_FORMAT - Format de ligne

```sql
-- Format de ligne pour InnoDB
CREATE TABLE optimized_table (
    id INT PRIMARY KEY,
    data TEXT
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED;

-- Options ROW_FORMAT :
-- - DYNAMIC : Par défaut, optimal (recommandé)
-- - COMPRESSED : Compression (économise l'espace)
-- - COMPACT : Ancien format compact
-- - REDUNDANT : Ancien format redondant
```

---

## SHOW CREATE TABLE - Voir la structure complète

```sql
-- Voir la commande CREATE TABLE complète
SHOW CREATE TABLE clients\G

-- Résultat (exemple) :
-- *************************** 1. row ***************************
--        Table: clients
-- Create Table: CREATE TABLE `clients` (
--   `client_id` int(11) NOT NULL AUTO_INCREMENT,
--   `nom` varchar(100) NOT NULL,
--   `email` varchar(255) NOT NULL,
--   `date_inscription` date DEFAULT NULL,
--   PRIMARY KEY (`client_id`),
--   UNIQUE KEY `email` (`email`)
-- ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

---

## ALTER TABLE - Modifier une table

### Ajouter une colonne

```sql
-- Ajouter une colonne à la fin
ALTER TABLE clients
ADD COLUMN telephone VARCHAR(20);

-- Ajouter une colonne avec contraintes
ALTER TABLE clients
ADD COLUMN ville VARCHAR(100) NOT NULL DEFAULT 'Paris';

-- Ajouter une colonne à une position spécifique
ALTER TABLE clients
ADD COLUMN prenom VARCHAR(100) AFTER nom;

-- Ajouter en première position
ALTER TABLE clients
ADD COLUMN code_client VARCHAR(20) FIRST;

-- Ajouter plusieurs colonnes en une fois
ALTER TABLE clients
ADD COLUMN adresse VARCHAR(255),
ADD COLUMN code_postal CHAR(5),
ADD COLUMN pays VARCHAR(50) DEFAULT 'France';
```

### Modifier une colonne

```sql
-- Modifier le type d'une colonne
ALTER TABLE clients
MODIFY COLUMN telephone VARCHAR(30);

-- Modifier avec toutes les options (MODIFY ne garde pas les attributs)
ALTER TABLE clients
MODIFY COLUMN email VARCHAR(320) NOT NULL UNIQUE;

-- Renommer ET modifier (CHANGE)
ALTER TABLE clients
CHANGE COLUMN telephone tel_mobile VARCHAR(30);

-- Modifier uniquement le nom (RENAME COLUMN - MariaDB 10.5+)
ALTER TABLE clients
RENAME COLUMN tel_mobile TO telephone;

-- Modifier la valeur par défaut
ALTER TABLE clients
ALTER COLUMN pays SET DEFAULT 'France';

-- Supprimer la valeur par défaut
ALTER TABLE clients
ALTER COLUMN pays DROP DEFAULT;
```

⚠️ **Attention** : `MODIFY COLUMN` ne conserve pas les attributs existants. Spécifiez TOUT (NOT NULL, DEFAULT, etc.).

### Supprimer une colonne

```sql
-- Supprimer une colonne
ALTER TABLE clients
DROP COLUMN adresse;

-- Supprimer plusieurs colonnes
ALTER TABLE clients
DROP COLUMN code_postal,
DROP COLUMN pays;

-- ⚠️ La suppression est définitive et immédiate !
```

### Ajouter et supprimer des contraintes

```sql
-- Ajouter une clé primaire
ALTER TABLE logs
ADD PRIMARY KEY (log_id);

-- Ajouter une clé étrangère
ALTER TABLE commandes
ADD CONSTRAINT fk_commandes_client
    FOREIGN KEY (client_id)
    REFERENCES clients(client_id)
    ON DELETE CASCADE;

-- Ajouter une contrainte UNIQUE
ALTER TABLE clients
ADD CONSTRAINT uq_email UNIQUE (email);

-- Ajouter une contrainte CHECK
ALTER TABLE produits
ADD CONSTRAINT chk_prix_positif CHECK (prix > 0);

-- Supprimer une clé étrangère
ALTER TABLE commandes
DROP FOREIGN KEY fk_commandes_client;

-- Supprimer une contrainte UNIQUE
ALTER TABLE clients
DROP INDEX uq_email;

-- Supprimer une contrainte CHECK
ALTER TABLE produits
DROP CONSTRAINT chk_prix_positif;

-- Supprimer la clé primaire
ALTER TABLE logs
DROP PRIMARY KEY;
```

### Modifier les options de table

```sql
-- Changer le moteur de stockage
ALTER TABLE clients ENGINE=InnoDB;

-- Changer le charset et la collation
ALTER TABLE clients
CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Modifier l'AUTO_INCREMENT
ALTER TABLE clients AUTO_INCREMENT = 5000;

-- Changer le ROW_FORMAT
ALTER TABLE clients ROW_FORMAT=COMPRESSED;

-- Ajouter un commentaire
ALTER TABLE clients COMMENT = 'Table principale des clients';
```

### Renommer une table

```sql
-- Méthode 1 : ALTER TABLE
ALTER TABLE old_name RENAME TO new_name;

-- Méthode 2 : RENAME TABLE (peut renommer plusieurs tables)
RENAME TABLE
    old_name1 TO new_name1,
    old_name2 TO new_name2;

-- Déplacer une table vers une autre base
RENAME TABLE
    database1.table1 TO database2.table1;
```

---

## DROP TABLE - Supprimer une table

### Suppression simple

```sql
-- Supprimer une table
DROP TABLE clients;

-- Avec IF EXISTS pour éviter les erreurs
DROP TABLE IF EXISTS clients;

-- Supprimer plusieurs tables
DROP TABLE IF EXISTS clients, commandes, produits;
```

### Suppression sécurisée

```sql
-- ⚠️ Vérifications avant suppression

-- 1. Vérifier le contenu
SELECT COUNT(*) FROM old_table;

-- 2. Vérifier les dépendances (clés étrangères)
SELECT
    TABLE_NAME,
    CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME = 'old_table';

-- 3. Backup avant suppression (si données importantes)
CREATE TABLE old_table_backup AS SELECT * FROM old_table;

-- 4. Supprimer
DROP TABLE IF EXISTS old_table;

-- 5. Vérifier
SHOW TABLES LIKE 'old_table';
-- Empty set
```

### TRUNCATE TABLE - Vider sans supprimer

```sql
-- Supprimer toutes les lignes (structure conservée)
TRUNCATE TABLE logs;

-- Différence avec DELETE :
DELETE FROM logs;           -- Lent, peut être ROLLBACK, incrémente pas réinitialisé
TRUNCATE TABLE logs;        -- Rapide, AUTO-COMMIT, réinitialise AUTO_INCREMENT
```

---

## Exemple complet : Système de blog

```sql
-- ============================================
-- Système de blog complet
-- ============================================

-- 1. Table des auteurs
CREATE TABLE IF NOT EXISTS auteurs (
    auteur_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    bio TEXT,
    date_inscription DATETIME DEFAULT CURRENT_TIMESTAMP,
    actif BOOLEAN DEFAULT TRUE,

    INDEX idx_email (email),
    INDEX idx_nom (nom, prenom)
) ENGINE=InnoDB
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci
  COMMENT = 'Table des auteurs du blog';

-- 2. Table des catégories
CREATE TABLE IF NOT EXISTS categories (
    categorie_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,           -- URL-friendly
    description TEXT,

    INDEX idx_slug (slug)
) ENGINE=InnoDB
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- 3. Table des articles
CREATE TABLE IF NOT EXISTS articles (
    article_id INT PRIMARY KEY AUTO_INCREMENT,
    titre VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    contenu MEDIUMTEXT NOT NULL,
    resume VARCHAR(500),
    auteur_id INT NOT NULL,
    categorie_id INT,
    statut ENUM('brouillon', 'publie', 'archive') DEFAULT 'brouillon',
    nombre_vues INT UNSIGNED DEFAULT 0,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    date_publication DATETIME,
    date_modification TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                               ON UPDATE CURRENT_TIMESTAMP,

    -- Clés étrangères
    FOREIGN KEY (auteur_id)
        REFERENCES auteurs(auteur_id)
        ON DELETE RESTRICT                      -- Ne pas supprimer auteur avec articles
        ON UPDATE CASCADE,

    FOREIGN KEY (categorie_id)
        REFERENCES categories(categorie_id)
        ON DELETE SET NULL                      -- Si catégorie supprimée, NULL
        ON UPDATE CASCADE,

    -- Index pour performance
    INDEX idx_auteur (auteur_id),
    INDEX idx_categorie (categorie_id),
    INDEX idx_slug (slug),
    INDEX idx_statut (statut),
    INDEX idx_date_publication (date_publication),
    FULLTEXT INDEX idx_fulltext (titre, contenu),

    -- Contraintes
    CHECK (nombre_vues >= 0)
) ENGINE=InnoDB
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci
  ROW_FORMAT=DYNAMIC;

-- 4. Table des commentaires
CREATE TABLE IF NOT EXISTS commentaires (
    commentaire_id INT PRIMARY KEY AUTO_INCREMENT,
    article_id INT NOT NULL,
    auteur_nom VARCHAR(100) NOT NULL,
    auteur_email VARCHAR(255) NOT NULL,
    contenu TEXT NOT NULL,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    approuve BOOLEAN DEFAULT FALSE,

    FOREIGN KEY (article_id)
        REFERENCES articles(article_id)
        ON DELETE CASCADE,                      -- Suppression en cascade

    INDEX idx_article (article_id),
    INDEX idx_approuve (approuve),
    INDEX idx_date (date_creation)
) ENGINE=InnoDB
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- 5. Table de liaison : Tags (many-to-many)
CREATE TABLE IF NOT EXISTS tags (
    tag_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,

    INDEX idx_slug (slug)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS articles_tags (
    article_id INT NOT NULL,
    tag_id INT NOT NULL,

    PRIMARY KEY (article_id, tag_id),

    FOREIGN KEY (article_id)
        REFERENCES articles(article_id)
        ON DELETE CASCADE,

    FOREIGN KEY (tag_id)
        REFERENCES tags(tag_id)
        ON DELETE CASCADE
) ENGINE=InnoDB;

-- Vérification de la structure
SHOW TABLES;
```

---

## Bonnes pratiques de conception

### 1. Nommage cohérent

```sql
-- ✅ BONS noms
CREATE TABLE clients (              -- Pluriel ou singulier cohérent
    client_id INT PRIMARY KEY,      -- préfixe_id pour clé primaire
    nom VARCHAR(100),
    prenom VARCHAR(100),
    date_inscription DATE           -- snake_case
);

-- ❌ MAUVAIS noms
CREATE TABLE Client (               -- Majuscule (sensible à la casse)
    id INT,                         -- Trop générique
    Name VARCHAR(100),              -- CamelCase
    `date-inscription` DATE         -- Tirets (backticks requis)
);
```

### 2. Toujours définir une clé primaire

```sql
-- ✅ BON : Clé primaire explicite
CREATE TABLE logs (
    log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message TEXT,
    date_log TIMESTAMP
);

-- ❌ MAUVAIS : Pas de clé primaire
CREATE TABLE logs_bad (
    message TEXT,
    date_log TIMESTAMP
);
-- Problèmes : Pas d'identifiant unique, réplication difficile, UPDATE/DELETE ambigus
```

### 3. Utiliser NOT NULL quand approprié

```sql
-- ✅ BON : NOT NULL sur champs essentiels
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,              -- Obligatoire
    prix DECIMAL(10,2) NOT NULL,            -- Obligatoire
    description TEXT                        -- Optionnel
);

-- ❌ MAUVAIS : Tout optionnel
CREATE TABLE produits_bad (
    produit_id INT PRIMARY KEY,
    nom VARCHAR(100),                       -- Peut être NULL !
    prix DECIMAL(10,2)                      -- Peut être NULL !
);
```

### 4. Indexer les clés étrangères

```sql
-- ✅ BON : Index sur FK
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    FOREIGN KEY (client_id) REFERENCES clients(client_id),
    INDEX idx_client (client_id)            -- Index sur FK
);

-- Sans index, les JOIN sur cette FK seront lents
```

### 5. Utiliser des types appropriés

```sql
-- ✅ BON : Types adaptés
CREATE TABLE exemple (
    age TINYINT UNSIGNED,                   -- 0-255, 1 octet
    prix DECIMAL(10,2),                     -- Précision exacte pour argent
    actif BOOLEAN                           -- Plus clair que TINYINT(1)
);

-- ❌ MAUVAIS : Types surdimensionnés
CREATE TABLE exemple_bad (
    age BIGINT,                             -- 8 octets pour 0-120 !
    prix FLOAT,                             -- Imprécis pour argent
    actif VARCHAR(10)                       -- 'true'/'false' en texte
);
```

---

## Pièges courants à éviter

### 1. MODIFY sans spécifier tous les attributs

```sql
-- Table initiale
CREATE TABLE users (
    email VARCHAR(255) NOT NULL UNIQUE DEFAULT 'no-email@example.com'
);

-- ❌ PROBLÈME : MODIFY perd les attributs
ALTER TABLE users MODIFY COLUMN email VARCHAR(320);
-- Résultat : email devient NULL, pas UNIQUE, pas de DEFAULT !

-- ✅ SOLUTION : Tout respécifier
ALTER TABLE users
MODIFY COLUMN email VARCHAR(320) NOT NULL UNIQUE DEFAULT 'no-email@example.com';
```

### 2. ALTER TABLE bloquant sur grande table

```sql
-- ❌ PROBLÈME : ALTER TABLE bloque toute la table
ALTER TABLE huge_table ADD COLUMN new_col INT;
-- Sur une table de 100M lignes : peut prendre 1 heure et bloquer !

-- ✅ SOLUTIONS :
-- 1. Utiliser pt-online-schema-change (Percona Toolkit)
-- 2. Planifier en heures creuses
-- 3. Utiliser ALTER TABLE ALGORITHM=INPLACE si possible
ALTER TABLE huge_table ADD COLUMN new_col INT, ALGORITHM=INPLACE;
```

### 3. Oublier les transactions pour modifications multiples

```sql
-- ❌ RISQUE : Une erreur au milieu laisse base incohérente
ALTER TABLE users ADD COLUMN age INT;
ALTER TABLE users ADD COLUMN ville VARCHAR(100);
-- Si la 2e commande échoue, age existe mais pas ville

-- ✅ MEILLEUR : Tout en une transaction
START TRANSACTION;
ALTER TABLE users ADD COLUMN age INT;
ALTER TABLE users ADD COLUMN ville VARCHAR(100);
COMMIT;
-- Tout ou rien
```

### 4. Supprimer une colonne avec données importantes

```sql
-- ❌ DANGER : Suppression immédiate et définitive
ALTER TABLE clients DROP COLUMN ancien_email;
-- Données perdues !

-- ✅ SÉCURISÉ : Backup avant suppression
-- 1. Sauvegarder les données
CREATE TABLE clients_backup AS SELECT * FROM clients;

-- 2. Ou exporter la colonne
SELECT client_id, ancien_email INTO OUTFILE '/tmp/emails_backup.csv' FROM clients;

-- 3. Supprimer
ALTER TABLE clients DROP COLUMN ancien_email;
```

### 5. Créer des tables sans ENGINE spécifié

```sql
-- ❌ RISQUE : Moteur par défaut peut varier
CREATE TABLE logs (
    log_id INT PRIMARY KEY,
    message TEXT
);
-- Quel moteur ? Dépend de la configuration serveur

-- ✅ EXPLICITE : Toujours spécifier
CREATE TABLE logs (
    log_id INT PRIMARY KEY,
    message TEXT
) ENGINE=InnoDB;
```

---

## ✅ Points clés à retenir

- **CREATE TABLE** : Définir structure avec colonnes, types et contraintes
- **ALTER TABLE** : Modifier structure existante (ADD, MODIFY, DROP, RENAME)
- **DROP TABLE** : Supprimer table (⚠️ irréversible)
- **IF NOT EXISTS** / **IF EXISTS** : Éviter erreurs dans scripts
- **PRIMARY KEY** : Toujours définir une clé primaire
- **FOREIGN KEY** : Maintenir intégrité référentielle avec options CASCADE
- **NOT NULL** : Rendre obligatoires les champs essentiels
- **AUTO_INCREMENT** : Génération automatique d'identifiants
- **ENGINE=InnoDB** : Moteur recommandé (FK, transactions, ACID)
- **Charset utf8mb4** : Support complet Unicode (emojis)
- **Index sur FK** : Améliore performances des JOIN
- **MODIFY perd attributs** : Tout respécifier lors de MODIFY COLUMN
- **Backup avant DROP** : Toujours sauvegarder avant suppression
- **SHOW CREATE TABLE** : Voir structure complète d'une table

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CREATE TABLE](https://mariadb.com/kb/en/create-table/)
- [📖 ALTER TABLE](https://mariadb.com/kb/en/alter-table/)
- [📖 DROP TABLE](https://mariadb.com/kb/en/drop-table/)
- [📖 SHOW CREATE TABLE](https://mariadb.com/kb/en/show-create-table/)
- [📖 Storage Engines](https://mariadb.com/kb/en/storage-engines/)
- [📖 Foreign Keys](https://mariadb.com/kb/en/foreign-keys/)

### Outils
- [Percona Toolkit pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/)
- [gh-ost (GitHub Online Schema Change)](https://github.com/github/gh-ost)

---

## ➡️ Section suivante

**[2.5 Contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT)](/02-bases-du-sql/05-contraintes.md)**

Approfondissez votre compréhension des contraintes d'intégrité : comment et quand les utiliser, les différentes options CASCADE/RESTRICT, et les stratégies pour garantir la qualité des données dans votre base.

---


⏭️ [Contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT)](/02-bases-du-sql/05-contraintes.md)

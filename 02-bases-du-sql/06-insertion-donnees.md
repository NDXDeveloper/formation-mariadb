🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.6 Insertion de données (INSERT, INSERT INTO SELECT, LOAD DATA)

> **Niveau** : Débutant
> **Durée estimée** : 1.5 heures
> **Prérequis** : Section 2.5 (Contraintes)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Insérer des données avec INSERT INTO de différentes manières
- Gérer les insertions multiples pour optimiser les performances
- Utiliser INSERT INTO SELECT pour copier des données entre tables
- Importer des fichiers CSV avec LOAD DATA INFILE
- Gérer les erreurs d'insertion (IGNORE, ON DUPLICATE KEY UPDATE)
- Comprendre et utiliser les valeurs par défaut et AUTO_INCREMENT
- Appliquer les bonnes pratiques d'insertion de données

---

## Introduction

L'**insertion de données** est l'opération qui permet d'ajouter de nouvelles lignes dans une table. C'est l'une des opérations les plus fréquentes dans une base de données.

### Les différentes méthodes d'insertion

| Méthode | Usage | Volumétrie |
|---------|-------|------------|
| **INSERT VALUES** | Insertion manuelle de quelques lignes | 1-100 lignes |
| **INSERT SELECT** | Copie de données entre tables | 100-100K lignes |
| **LOAD DATA** | Import massif depuis fichiers CSV | 100K+ lignes |

---

## INSERT INTO - Insertion simple

### Syntaxe de base

```sql
-- Syntaxe complète
INSERT INTO nom_table (colonne1, colonne2, colonne3)
VALUES (valeur1, valeur2, valeur3);

-- Exemple concret
CREATE TABLE clients (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    telephone VARCHAR(20),
    date_inscription DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insertion avec toutes les colonnes spécifiées
INSERT INTO clients (nom, email, telephone)
VALUES ('Alice Martin', 'alice@example.com', '0612345678');

-- Vérification
SELECT * FROM clients;
-- client_id: 1 (AUTO_INCREMENT)
-- nom: 'Alice Martin'
-- email: 'alice@example.com'
-- telephone: '0612345678'
-- date_inscription: 2025-12-12 14:30:00 (CURRENT_TIMESTAMP)
```

### INSERT sans spécifier les colonnes

```sql
-- ⚠️ Insertion dans toutes les colonnes (ordre table)
INSERT INTO clients
VALUES (NULL, 'Bob Dupont', 'bob@example.com', '0698765432', NOW());
-- NULL pour AUTO_INCREMENT
-- Toutes les valeurs dans l'ordre exact de la table

-- ✅ RECOMMANDÉ : Toujours spécifier les colonnes
INSERT INTO clients (nom, email, telephone)
VALUES ('Charlie Durand', 'charlie@example.com', '0623456789');
-- Plus lisible, moins d'erreurs, indépendant de l'ordre des colonnes
```

💡 **Bonne pratique** : Spécifiez toujours les colonnes explicitement pour éviter les erreurs si la structure de la table change.

### Valeurs par défaut et NULL

```sql
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2) NOT NULL,
    stock INT DEFAULT 0,                        -- Valeur par défaut
    actif BOOLEAN DEFAULT TRUE,
    description TEXT,                           -- Peut être NULL
    date_ajout DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insertion minimale (colonnes obligatoires uniquement)
INSERT INTO produits (nom, prix)
VALUES ('Livre SQL', 29.99);
-- stock: 0 (DEFAULT)
-- actif: TRUE (DEFAULT)
-- description: NULL (non spécifié)
-- date_ajout: 2025-12-12 14:30:00 (DEFAULT)

-- Insertion avec valeur NULL explicite
INSERT INTO produits (nom, prix, description)
VALUES ('Cahier', 5.99, NULL);

-- Insertion en surchargeant les DEFAULT
INSERT INTO produits (nom, prix, stock, actif)
VALUES ('Stylo', 1.99, 50, FALSE);
```

### AUTO_INCREMENT et LAST_INSERT_ID()

```sql
-- Insertion avec AUTO_INCREMENT
INSERT INTO clients (nom, email)
VALUES ('David Leblanc', 'david@example.com');

-- Récupérer le dernier ID inséré
SELECT LAST_INSERT_ID();
-- Retourne: 4 (par exemple)

-- Utilisation dans une application
INSERT INTO commandes (client_id, montant)
VALUES (LAST_INSERT_ID(), 150.00);
-- Utilise le client_id qui vient d'être généré

-- Dans une transaction complète
START TRANSACTION;

INSERT INTO clients (nom, email) VALUES ('Emma Petit', 'emma@example.com');
SET @nouveau_client_id = LAST_INSERT_ID();

INSERT INTO commandes (client_id, montant) VALUES (@nouveau_client_id, 99.99);
INSERT INTO commandes (client_id, montant) VALUES (@nouveau_client_id, 49.99);

COMMIT;
```

---

## INSERT multiple - Insertions groupées

### Insérer plusieurs lignes en une commande

```sql
-- ✅ EFFICACE : Insertion multiple en une seule commande
INSERT INTO produits (nom, prix, stock) VALUES
    ('Livre MySQL', 34.99, 15),
    ('Livre PostgreSQL', 39.99, 8),
    ('Livre MongoDB', 29.99, 20),
    ('Guide SQL', 24.99, 12),
    ('Manuel MariaDB', 44.99, 5);

-- 1 seule requête = 5 lignes insérées

-- ❌ INEFFICACE : 5 requêtes séparées
INSERT INTO produits (nom, prix, stock) VALUES ('Livre MySQL', 34.99, 15);
INSERT INTO produits (nom, prix, stock) VALUES ('Livre PostgreSQL', 39.99, 8);
INSERT INTO produits (nom, prix, stock) VALUES ('Livre MongoDB', 29.99, 20);
INSERT INTO produits (nom, prix, stock) VALUES ('Guide SQL', 24.99, 12);
INSERT INTO produits (nom, prix, stock) VALUES ('Manuel MariaDB', 44.99, 5);
-- 5 requêtes = plus lent, plus de trafic réseau
```

### Performance : INSERT groupé vs individuel

```sql
-- Exemple comparatif de performance

-- ❌ LENT : 1000 INSERT individuels
-- Temps estimé : 5-10 secondes
DELIMITER //
CREATE PROCEDURE insert_lent()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 1000 DO
        INSERT INTO logs (message) VALUES (CONCAT('Log #', i));
        SET i = i + 1;
    END WHILE;
END//
DELIMITER ;

-- ✅ RAPIDE : INSERT groupé par lots de 100
-- Temps estimé : 0.5-1 seconde
INSERT INTO logs (message) VALUES
    ('Log #1'), ('Log #2'), ('Log #3'), ... ('Log #100');
-- Répéter 10 fois
```

💡 **Recommandation** : Groupez les insertions par lots de 100-1000 lignes pour optimiser les performances.

---

## INSERT INTO SELECT - Copie de données

### Copier des données d'une table à une autre

```sql
-- Création de tables pour l'exemple
CREATE TABLE clients_actifs (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100),
    email VARCHAR(255),
    derniere_commande DATE
);

CREATE TABLE clients_archive (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100),
    email VARCHAR(255),
    derniere_commande DATE,
    date_archivage DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insérer des données de test
INSERT INTO clients_actifs (nom, email, derniere_commande) VALUES
    ('Alice Martin', 'alice@example.com', '2025-12-01'),
    ('Bob Dupont', 'bob@example.com', '2024-06-15'),
    ('Charlie Durand', 'charlie@example.com', '2025-11-20');

-- INSERT INTO SELECT : Copier clients inactifs vers archive
INSERT INTO clients_archive (client_id, nom, email, derniere_commande)
SELECT client_id, nom, email, derniere_commande
FROM clients_actifs
WHERE derniere_commande < '2025-01-01';
-- Copie Bob Dupont (dernière commande en 2024)

-- Vérification
SELECT * FROM clients_archive;
```

### INSERT INTO SELECT avec transformation

```sql
-- Copie avec calculs et transformations
CREATE TABLE statistiques_commandes (
    client_id INT,
    total_commandes INT,
    montant_total DECIMAL(10,2),
    premiere_commande DATE,
    derniere_commande DATE
);

-- Insertion de statistiques agrégées
INSERT INTO statistiques_commandes (
    client_id,
    total_commandes,
    montant_total,
    premiere_commande,
    derniere_commande
)
SELECT
    client_id,
    COUNT(*) AS total_commandes,
    SUM(montant) AS montant_total,
    MIN(date_commande) AS premiere_commande,
    MAX(date_commande) AS derniere_commande
FROM commandes
GROUP BY client_id;
```

### Copier entre bases de données

```sql
-- Copier d'une base à une autre
INSERT INTO base_production.clients (nom, email, telephone)
SELECT nom, email, telephone
FROM base_test.clients
WHERE date_inscription >= '2025-01-01';

-- Ou avec USE
USE base_production;
INSERT INTO clients (nom, email)
SELECT nom, email
FROM base_test.clients
WHERE actif = TRUE;
```

### Créer une table et insérer en une seule commande

```sql
-- CREATE TABLE ... SELECT (crée + insère)
CREATE TABLE clients_vip
SELECT
    client_id,
    nom,
    email,
    COUNT(c.commande_id) AS total_commandes,
    SUM(c.montant) AS total_depense
FROM clients cl
JOIN commandes c ON cl.client_id = c.client_id
GROUP BY cl.client_id, cl.nom, cl.email
HAVING total_depense > 1000;

-- ⚠️ Attention : Les index et contraintes ne sont pas copiés !
-- Il faut les recréer manuellement
ALTER TABLE clients_vip ADD PRIMARY KEY (client_id);
```

---

## LOAD DATA INFILE - Import de fichiers CSV

### Préparer un fichier CSV

```csv
# fichier: clients.csv
nom,email,telephone,ville
"Alice Martin","alice@example.com","0612345678","Paris"
"Bob Dupont","bob@example.com","0698765432","Lyon"
"Charlie Durand","charlie@example.com","0623456789","Marseille"
"David Leblanc","david@example.com","0634567890","Toulouse"
"Emma Petit","emma@example.com","0645678901","Nice"
```

### Syntaxe LOAD DATA INFILE

```sql
-- Créer la table de destination
CREATE TABLE clients_import (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    telephone VARCHAR(20),
    ville VARCHAR(100)
);

-- LOAD DATA INFILE - Import basique
LOAD DATA INFILE '/var/lib/mysql-files/clients.csv'
INTO TABLE clients_import
FIELDS TERMINATED BY ','                        -- Séparateur de colonnes
ENCLOSED BY '"'                                 -- Délimiteur de texte
LINES TERMINATED BY '\n'                        -- Séparateur de lignes
IGNORE 1 LINES                                  -- Ignorer l'en-tête
(nom, email, telephone, ville);                 -- Colonnes du CSV

-- Vérification
SELECT * FROM clients_import;
```

### Options LOAD DATA

```sql
-- Import avec gestion d'erreurs
LOAD DATA INFILE '/var/lib/mysql-files/produits.csv'
INTO TABLE produits
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'                      -- " seulement si nécessaire
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(@nom, @prix, @stock, @categorie)               -- Variables temporaires
SET
    nom = TRIM(@nom),                           -- Nettoyer espaces
    prix = CAST(@prix AS DECIMAL(10,2)),        -- Conversion explicite
    stock = IF(@stock = '', 0, @stock),         -- Valeur par défaut si vide
    categorie_id = (SELECT categorie_id FROM categories WHERE nom = @categorie);

-- REPLACE : Remplace les doublons au lieu d'erreur
LOAD DATA INFILE '/var/lib/mysql-files/clients.csv'
REPLACE INTO TABLE clients_import
FIELDS TERMINATED BY ','
IGNORE 1 LINES
(nom, email, telephone, ville);

-- IGNORE : Ignore les doublons
LOAD DATA INFILE '/var/lib/mysql-files/clients.csv'
IGNORE INTO TABLE clients_import
FIELDS TERMINATED BY ','
IGNORE 1 LINES
(nom, email, telephone, ville);
```

### LOAD DATA LOCAL INFILE

```sql
-- LOCAL : Fichier sur la machine cliente (pas le serveur)
LOAD DATA LOCAL INFILE '/home/user/data/clients.csv'
INTO TABLE clients_import
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(nom, email, telephone, ville);

-- ⚠️ Nécessite l'option local_infile activée
-- mysql --local-infile=1 -u user -p

-- Vérifier si LOCAL INFILE est activé
SHOW GLOBAL VARIABLES LIKE 'local_infile';
-- ON : Activé, OFF : Désactivé

-- Activer LOCAL INFILE (avec privilèges)
SET GLOBAL local_infile = 1;
```

### Formats de fichiers spéciaux

```sql
-- Fichier TSV (Tab-Separated Values)
LOAD DATA INFILE '/var/lib/mysql-files/data.tsv'
INTO TABLE ma_table
FIELDS TERMINATED BY '\t'                       -- Tabulation
LINES TERMINATED BY '\n'
IGNORE 1 LINES;

-- Fichier avec séparateur personnalisé (|)
LOAD DATA INFILE '/var/lib/mysql-files/data.txt'
INTO TABLE ma_table
FIELDS TERMINATED BY '|'
LINES TERMINATED BY '\n';

-- Fichier Windows (fin de ligne \r\n)
LOAD DATA INFILE '/var/lib/mysql-files/data_windows.csv'
INTO TABLE ma_table
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\r\n'                      -- Windows
IGNORE 1 LINES;
```

---

## Gestion des erreurs d'insertion

### INSERT IGNORE - Ignorer les erreurs

```sql
-- Sans IGNORE : Erreur si violation de contrainte
INSERT INTO clients (nom, email) VALUES
    ('Alice Martin', 'alice@example.com'),
    ('Bob Dupont', 'alice@example.com');        -- Email en doublon
-- ERROR: Duplicate entry 'alice@example.com' for key 'email'
-- Aucune ligne insérée (même pas Alice)

-- Avec IGNORE : Continue malgré les erreurs
INSERT IGNORE INTO clients (nom, email) VALUES
    ('Alice Martin', 'alice@example.com'),
    ('Bob Dupont', 'alice@example.com');        -- Doublon ignoré
-- 1 row affected (Alice insérée, Bob ignoré)

-- Vérifier les avertissements
SHOW WARNINGS;
-- Warning: Duplicate entry 'alice@example.com' for key 'email'
```

⚠️ **Attention** : `INSERT IGNORE` masque toutes les erreurs, y compris celles que vous ne voulez peut-être pas ignorer (conversion de type, etc.).

### ON DUPLICATE KEY UPDATE - Upsert

```sql
-- Insérer ou mettre à jour si doublon
CREATE TABLE statistiques_produit (
    produit_id INT PRIMARY KEY,
    vues INT DEFAULT 0,
    achats INT DEFAULT 0,
    derniere_vue DATETIME
);

-- Première insertion
INSERT INTO statistiques_produit (produit_id, vues, derniere_vue)
VALUES (1, 1, NOW())
ON DUPLICATE KEY UPDATE
    vues = vues + 1,                            -- Incrémenter si existe
    derniere_vue = NOW();

-- Exécuter plusieurs fois : vues s'incrémente
-- Execution 1 : produit_id=1, vues=1
-- Execution 2 : produit_id=1, vues=2
-- Execution 3 : produit_id=1, vues=3

-- Exemple avec VALUES()
INSERT INTO statistiques_produit (produit_id, vues, achats)
VALUES (2, 5, 1)
ON DUPLICATE KEY UPDATE
    vues = vues + VALUES(vues),                 -- Ajouter la nouvelle valeur
    achats = achats + VALUES(achats);

-- Si produit_id=2 existe déjà : vues += 5, achats += 1
-- Si produit_id=2 n'existe pas : INSERT normal
```

### Gestion des clés étrangères

```sql
-- Erreur si clé étrangère invalide
INSERT INTO commandes (client_id, montant)
VALUES (999, 150.00);
-- ERROR: Cannot add or update a child row: a foreign key constraint fails

-- Solution 1 : Vérifier l'existence avant insertion
INSERT INTO commandes (client_id, montant)
SELECT 999, 150.00
WHERE EXISTS (SELECT 1 FROM clients WHERE client_id = 999);
-- Insère uniquement si le client existe

-- Solution 2 : INSERT IGNORE (pas recommandé)
INSERT IGNORE INTO commandes (client_id, montant)
VALUES (999, 150.00);
-- Ignore l'erreur silencieusement

-- Solution 3 : Créer le parent si absent
INSERT INTO clients (client_id, nom, email)
VALUES (999, 'Nouveau Client', 'nouveau@example.com')
ON DUPLICATE KEY UPDATE client_id = client_id;  -- Ne fait rien si existe

INSERT INTO commandes (client_id, montant)
VALUES (999, 150.00);
```

---

## Types de données et insertion

### Insertion de chaînes de caractères

```sql
-- Échappement des guillemets
INSERT INTO articles (titre, contenu) VALUES
    ('L''art de la programmation', 'Un article sur l''apprentissage'),
    ("L'art de la programmation", "Un article sur l'apprentissage");
-- ' doublé pour échapper OU utiliser "

-- Caractères spéciaux
INSERT INTO textes (contenu) VALUES
    ('Ligne 1\nLigne 2'),                       -- \n = saut de ligne
    ('Tabulation:\tici'),                       -- \t = tabulation
    ('Backslash: \\'),                          -- \\ = \
    ('Citation: \"test\"');                     -- \" = "

-- Encodage UTF-8 (emojis)
INSERT INTO messages (texte) VALUES
    ('Bonjour 👋 Comment allez-vous? 😊'),
    ('Prix: 10€ - Disponible ✅');
```

### Insertion de nombres

```sql
-- Nombres entiers
INSERT INTO statistiques (vues, likes, partages) VALUES
    (1250, 89, 23),
    (0, 0, 0),                                  -- Zéros valides
    (-50, 10, 5);                               -- Négatifs si SIGNED

-- Nombres décimaux
INSERT INTO produits (prix, poids) VALUES
    (29.99, 1.5),
    (99.95, 2.750),                             -- Trailing zeros ignorés
    (10, 1);                                    -- Converti en 10.00, 1.000

-- Notation scientifique
INSERT INTO mesures (valeur) VALUES
    (1.5e3),                                    -- 1500
    (2.5e-2);                                   -- 0.025
```

### Insertion de dates et heures

```sql
-- Formats de dates acceptés
INSERT INTO evenements (nom, date_evenement) VALUES
    ('Réunion', '2025-12-25'),                  -- Format ISO (recommandé)
    ('Conference', '25/12/2025'),               -- Format FR (selon SQL_MODE)
    ('Atelier', '2025-12-25 14:30:00');         -- DATETIME

-- Fonctions temporelles
INSERT INTO logs (message, date_creation) VALUES
    ('Log système', NOW()),                     -- Date/heure actuelle
    ('Log boot', CURRENT_TIMESTAMP),            -- Alias de NOW()
    ('Log hier', DATE_SUB(NOW(), INTERVAL 1 DAY)),
    ('Log demain', DATE_ADD(NOW(), INTERVAL 1 DAY));

-- Insertion avec calculs
INSERT INTO taches (titre, echeance) VALUES
    ('Rapport mensuel', LAST_DAY(CURRENT_DATE())),  -- Dernier jour du mois
    ('Backup', CURRENT_DATE() + INTERVAL 7 DAY);    -- Dans 7 jours
```

### Insertion de données binaires

```sql
-- Hash / Binary
INSERT INTO utilisateurs (email, mot_de_passe_hash) VALUES
    ('user@example.com', UNHEX(SHA2('password123', 256)));

-- Insertion de BLOB (petits fichiers)
INSERT INTO fichiers (nom, contenu, type_mime) VALUES
    ('image.png', LOAD_FILE('/tmp/image.png'), 'image/png');
-- ⚠️ LOAD_FILE nécessite le privilège FILE

-- Conversion hex → binary
INSERT INTO tokens (token_value) VALUES
    (UNHEX('4a7d1ed414474e4033ac29ccb8653d9b'));

-- UUID
INSERT INTO sessions (session_id, utilisateur_id) VALUES
    (UUID_TO_BIN(UUID()), 123);                 -- UUID optimal (16 octets)
```

### Insertion de JSON

```sql
-- JSON simple
INSERT INTO configurations (nom, parametres) VALUES
    ('app_config', '{"theme": "dark", "language": "fr"}');

-- JSON avec fonctions
INSERT INTO configurations (nom, parametres) VALUES
    ('user_prefs', JSON_OBJECT(
        'notifications', TRUE,
        'newsletter', FALSE,
        'theme', 'light'
    ));

-- JSON array
INSERT INTO configurations (nom, parametres) VALUES
    ('tags', JSON_ARRAY('sql', 'database', 'mariadb'));

-- Validation JSON automatique
INSERT INTO configurations (nom, parametres) VALUES
    ('test', '{"invalid json}');                -- Erreur de syntaxe
-- ERROR: Invalid JSON text
```

---

## Exemple complet : Système de blog

```sql
-- ============================================
-- Insertion de données : Système de blog
-- ============================================

-- 1. Insertion des auteurs
INSERT INTO auteurs (nom, email, bio) VALUES
    ('Alice Martin', 'alice@blog.com', 'Développeuse senior spécialisée en bases de données'),
    ('Bob Dupont', 'bob@blog.com', 'Expert SQL et performance'),
    ('Charlie Durand', 'charlie@blog.com', 'Architecte logiciel et formateur');

-- Récupérer les IDs
SET @alice_id = (SELECT auteur_id FROM auteurs WHERE email = 'alice@blog.com');
SET @bob_id = (SELECT auteur_id FROM auteurs WHERE email = 'bob@blog.com');
SET @charlie_id = (SELECT auteur_id FROM auteurs WHERE email = 'charlie@blog.com');

-- 2. Insertion des catégories
INSERT INTO categories (nom, slug, description) VALUES
    ('Tutoriels', 'tutoriels', 'Guides pas à pas pour apprendre'),
    ('Astuces', 'astuces', 'Tips et tricks pour gagner du temps'),
    ('Actualités', 'actualites', 'Dernières nouvelles du monde tech');

-- 3. Insertion des articles
INSERT INTO articles (
    titre,
    slug,
    contenu,
    auteur_id,
    categorie_id,
    statut,
    vues
) VALUES
    -- Article 1
    (
        'Introduction à MariaDB 11.8',
        'introduction-mariadb-11-8',
        'MariaDB 11.8 LTS est la dernière version stable...',
        @alice_id,
        (SELECT categorie_id FROM categories WHERE slug = 'tutoriels'),
        'publie',
        1250
    ),
    -- Article 2
    (
        'Optimisation des requêtes SQL',
        'optimisation-requetes-sql',
        'Les index sont essentiels pour la performance...',
        @bob_id,
        (SELECT categorie_id FROM categories WHERE slug = 'astuces'),
        'publie',
        890
    ),
    -- Article 3 (brouillon)
    (
        'Nouveautés MariaDB 2025',
        'nouveautes-mariadb-2025',
        'En cours de rédaction...',
        @charlie_id,
        (SELECT categorie_id FROM categories WHERE slug = 'actualites'),
        'brouillon',
        0
    );

-- 4. Insertion des tags
INSERT INTO tags (nom, slug) VALUES
    ('SQL', 'sql'),
    ('MariaDB', 'mariadb'),
    ('Performance', 'performance'),
    ('Index', 'index'),
    ('Tutoriel', 'tutoriel');

-- 5. Association articles-tags (many-to-many)
INSERT INTO articles_tags (article_id, tag_id)
SELECT
    a.article_id,
    t.tag_id
FROM articles a
CROSS JOIN tags t
WHERE
    (a.slug = 'introduction-mariadb-11-8' AND t.slug IN ('sql', 'mariadb', 'tutoriel'))
    OR (a.slug = 'optimisation-requetes-sql' AND t.slug IN ('sql', 'performance', 'index'));

-- 6. Insertion de commentaires
INSERT INTO commentaires (article_id, auteur_nom, auteur_email, contenu, approuve) VALUES
    (
        (SELECT article_id FROM articles WHERE slug = 'introduction-mariadb-11-8'),
        'David Leblanc',
        'david@example.com',
        'Excellent article ! Très clair et bien structuré.',
        TRUE
    ),
    (
        (SELECT article_id FROM articles WHERE slug = 'introduction-mariadb-11-8'),
        'Emma Petit',
        'emma@example.com',
        'Merci pour ce tutoriel complet.',
        TRUE
    ),
    (
        (SELECT article_id FROM articles WHERE slug = 'optimisation-requetes-sql'),
        'François Bernard',
        'francois@example.com',
        'Super article sur la performance !',
        FALSE                                   -- En attente de modération
    );

-- 7. Insertion statistiques depuis données existantes
CREATE TABLE statistiques_auteurs (
    auteur_id INT PRIMARY KEY,
    total_articles INT,
    total_vues BIGINT,
    moyenne_vues DECIMAL(10,2),
    premier_article DATETIME,
    dernier_article DATETIME
);

INSERT INTO statistiques_auteurs
SELECT
    a.auteur_id,
    COUNT(*) AS total_articles,
    SUM(ar.vues) AS total_vues,
    AVG(ar.vues) AS moyenne_vues,
    MIN(ar.date_publication) AS premier_article,
    MAX(ar.date_publication) AS dernier_article
FROM auteurs a
LEFT JOIN articles ar ON a.auteur_id = ar.auteur_id
WHERE ar.statut = 'publie'
GROUP BY a.auteur_id;

-- Vérification des données insérées
SELECT
    aut.nom AS auteur,
    cat.nom AS categorie,
    art.titre,
    art.statut,
    art.vues,
    COUNT(c.commentaire_id) AS nb_commentaires
FROM articles art
JOIN auteurs aut ON art.auteur_id = aut.auteur_id
JOIN categories cat ON art.categorie_id = cat.categorie_id
LEFT JOIN commentaires c ON art.article_id = c.article_id AND c.approuve = TRUE
GROUP BY art.article_id, aut.nom, cat.nom, art.titre, art.statut, art.vues
ORDER BY art.date_publication DESC;
```

---

## Import de données volumineuses

### Préparer l'import

```sql
-- 1. Désactiver les contraintes temporairement (si besoin)
SET FOREIGN_KEY_CHECKS = 0;
SET UNIQUE_CHECKS = 0;

-- 2. Désactiver les index (MyISAM uniquement)
ALTER TABLE ma_table DISABLE KEYS;

-- 3. Augmenter la taille du buffer
SET SESSION bulk_insert_buffer_size = 256 * 1024 * 1024;  -- 256 MB
```

### Import optimisé

```sql
-- Import avec transaction (InnoDB)
START TRANSACTION;

LOAD DATA INFILE '/var/lib/mysql-files/big_data.csv'
INTO TABLE ma_table
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES;

COMMIT;

-- Réactiver les contraintes
SET FOREIGN_KEY_CHECKS = 1;
SET UNIQUE_CHECKS = 1;

-- Réactiver les index
ALTER TABLE ma_table ENABLE KEYS;

-- Analyser la table pour optimiser les index
ANALYZE TABLE ma_table;
```

### Surveillance de l'import

```sql
-- Vérifier la progression dans une autre session
SELECT
    TABLE_NAME,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
    AND TABLE_NAME = 'ma_table';

-- Vérifier les processus en cours
SHOW PROCESSLIST;
```

---

## Bonnes pratiques

### 1. Utiliser les transactions pour les insertions multiples

```sql
-- ✅ BON : Transaction pour cohérence
START TRANSACTION;

INSERT INTO clients (nom, email) VALUES ('Alice', 'alice@example.com');
SET @client_id = LAST_INSERT_ID();

INSERT INTO adresses (client_id, rue, ville) VALUES (@client_id, '123 Rue Test', 'Paris');
INSERT INTO preferences (client_id, newsletter) VALUES (@client_id, TRUE);

COMMIT;

-- Si erreur quelque part : ROLLBACK annule tout
```

### 2. Valider les données avant insertion

```sql
-- ✅ BON : Validation avec sous-requête
INSERT INTO commandes (client_id, montant)
SELECT 123, 150.00
WHERE EXISTS (SELECT 1 FROM clients WHERE client_id = 123)
    AND 150.00 > 0;

-- Insertion conditionnelle
INSERT INTO logs (niveau, message)
SELECT 'ERROR', 'Connexion échouée'
WHERE (SELECT COUNT(*) FROM connexions_echouees) > 5;
```

### 3. Gérer les insertions en lots

```sql
-- ✅ BON : Lots de 1000 pour grande volumétrie
-- Script ou application
DELIMITER //
CREATE PROCEDURE insert_batch(IN batch_size INT)
BEGIN
    DECLARE i INT DEFAULT 0;
    DECLARE total INT DEFAULT 100000;

    WHILE i < total DO
        INSERT INTO ma_table (colonne1, colonne2)
        SELECT
            CONCAT('Value', i + n),
            RAND() * 1000
        FROM (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL ... SELECT 999) numbers
        WHERE i + n < total;

        SET i = i + batch_size;

        -- Commit tous les 1000
        IF i % 1000 = 0 THEN
            COMMIT;
        END IF;
    END WHILE;
END//
DELIMITER ;
```

### 4. Documenter les sources de données

```sql
-- ✅ BON : Tracer l'origine des données
CREATE TABLE imports_log (
    import_id INT PRIMARY KEY AUTO_INCREMENT,
    nom_fichier VARCHAR(255),
    table_destination VARCHAR(100),
    lignes_importees INT,
    lignes_rejetees INT,
    date_import DATETIME DEFAULT CURRENT_TIMESTAMP,
    utilisateur VARCHAR(100)
);

-- Logger chaque import
INSERT INTO imports_log (nom_fichier, table_destination, lignes_importees, lignes_rejetees, utilisateur)
VALUES ('clients_2025-12.csv', 'clients', 1523, 12, USER());
```

### 5. Nettoyer les données lors de l'import

```sql
-- ✅ BON : Nettoyage automatique
LOAD DATA INFILE '/var/lib/mysql-files/clients.csv'
INTO TABLE clients
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(@nom, @email, @telephone)
SET
    nom = TRIM(UPPER(@nom)),                    -- Majuscules + trim
    email = LOWER(TRIM(@email)),                -- Minuscules
    telephone = REGEXP_REPLACE(@telephone, '[^0-9]', '');  -- Seulement chiffres
```

---

## Pièges courants à éviter

### ❌ Piège 1 : Oublier de spécifier les colonnes

```sql
-- ❌ MAUVAIS : Insertion sans colonnes
INSERT INTO clients VALUES ('Alice', 'alice@example.com');
-- Erreur si la table a plus de 2 colonnes ou ordre différent

-- ✅ BON : Toujours spécifier
INSERT INTO clients (nom, email) VALUES ('Alice', 'alice@example.com');
```

### ❌ Piège 2 : INSERT individuels en boucle

```sql
-- ❌ TRÈS LENT : 1000 requêtes
FOR i = 1 TO 1000 DO
    INSERT INTO logs (message) VALUES (CONCAT('Log ', i));
END FOR

-- ✅ RAPIDE : 1 seule requête groupée
INSERT INTO logs (message) VALUES
    ('Log 1'), ('Log 2'), ... ('Log 1000');
```

### ❌ Piège 3 : Pas de transaction pour opérations liées

```sql
-- ❌ DANGEREUX : Sans transaction
INSERT INTO commandes (client_id) VALUES (123);
SET @cmd_id = LAST_INSERT_ID();
INSERT INTO lignes_commande (commande_id, produit_id) VALUES (@cmd_id, 456);
-- Si la 2e échoue, commande vide créée !

-- ✅ SÛRE : Avec transaction
START TRANSACTION;
INSERT INTO commandes (client_id) VALUES (123);
SET @cmd_id = LAST_INSERT_ID();
INSERT INTO lignes_commande (commande_id, produit_id) VALUES (@cmd_id, 456);
COMMIT;
```

### ❌ Piège 4 : Utiliser INSERT IGNORE aveuglément

```sql
-- ❌ CACHE TOUTES LES ERREURS
INSERT IGNORE INTO produits (nom, prix) VALUES
    ('Livre', 'invalid_price'),                 -- Erreur de conversion cachée
    ('Stylo', -10);                             -- Prix négatif ignoré

-- ✅ MIEUX : Gérer spécifiquement les doublons
INSERT INTO produits (nom, prix) VALUES ('Livre', 29.99)
ON DUPLICATE KEY UPDATE prix = VALUES(prix);
```

### ❌ Piège 5 : Chemin incorrect pour LOAD DATA

```sql
-- ❌ ERREUR : Fichier introuvable
LOAD DATA INFILE '/home/user/data.csv'         -- Pas accessible au serveur
INTO TABLE ma_table;

-- ✅ SOLUTION 1 : Utiliser le répertoire secure_file_priv
SHOW VARIABLES LIKE 'secure_file_priv';
-- Copier le fichier dans ce répertoire

-- ✅ SOLUTION 2 : Utiliser LOAD DATA LOCAL
LOAD DATA LOCAL INFILE '/home/user/data.csv'   -- Fichier sur client
INTO TABLE ma_table;
```

---

## Commandes utiles

### Afficher les dernières insertions

```sql
-- Voir les derniers clients insérés
SELECT * FROM clients
ORDER BY client_id DESC
LIMIT 10;

-- Compter les insertions du jour
SELECT COUNT(*) AS insertions_aujourd_hui
FROM clients
WHERE DATE(date_inscription) = CURDATE();

-- Statistiques d'insertion par jour
SELECT
    DATE(date_inscription) AS jour,
    COUNT(*) AS nb_insertions
FROM clients
WHERE date_inscription >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY DATE(date_inscription)
ORDER BY jour DESC;
```

### Vérifier l'état de l'AUTO_INCREMENT

```sql
-- Voir la valeur actuelle de l'AUTO_INCREMENT
SHOW TABLE STATUS LIKE 'clients';
-- Colonne Auto_increment: 156 (prochain ID)

-- Changer la valeur de l'AUTO_INCREMENT
ALTER TABLE clients AUTO_INCREMENT = 1000;

-- Réinitialiser (utiliser avec précaution)
TRUNCATE TABLE clients;  -- Remet AUTO_INCREMENT à 1
```

### Performance des insertions

```sql
-- Activer le profiling
SET profiling = 1;

-- Effectuer des insertions
INSERT INTO test VALUES (1), (2), (3);

-- Voir les temps d'exécution
SHOW PROFILES;

-- Détails d'une requête
SHOW PROFILE FOR QUERY 1;
```

---

## ✅ Points clés à retenir

- **INSERT VALUES** : Insertion directe, spécifier toujours les colonnes
- **INSERT multiple** : Grouper 100-1000 lignes pour performance
- **INSERT SELECT** : Copie efficace entre tables avec transformation
- **LOAD DATA** : Import massif optimal pour fichiers CSV/TSV
- **IGNORE** : Ignore erreurs mais attention aux effets de bord
- **ON DUPLICATE KEY UPDATE** : Upsert (insert or update)
- **LAST_INSERT_ID()** : Récupérer l'ID AUTO_INCREMENT généré
- **Transactions** : Obligatoires pour opérations liées
- **Validation** : Nettoyer et valider données avant insertion
- **Lots** : Insérer par paquets pour grande volumétrie
- **LOCAL INFILE** : Import depuis machine cliente
- **secure_file_priv** : Répertoire autorisé pour LOAD DATA
- **Performance** : Désactiver index/contraintes pour gros imports

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 INSERT Statement](https://mariadb.com/kb/en/insert/)
- [📖 LOAD DATA INFILE](https://mariadb.com/kb/en/load-data-infile/)
- [📖 INSERT SELECT](https://mariadb.com/kb/en/insert-select/)
- [📖 INSERT IGNORE](https://mariadb.com/kb/en/insert-ignore/)
- [📖 ON DUPLICATE KEY UPDATE](https://mariadb.com/kb/en/insert-on-duplicate-key-update/)
- [📖 LAST_INSERT_ID()](https://mariadb.com/kb/en/last_insert_id/)

### Lectures complémentaires
- [Optimizing INSERT Statements](https://mariadb.com/kb/en/optimization-and-tuning-inserting-data/)
- [Bulk Data Loading](https://mariadb.com/kb/en/how-to-quickly-insert-data-into-mariadb/)

---

## ➡️ Section suivante

**2.7 Requêtes de sélection simples (SELECT, WHERE, ORDER BY, LIMIT)**

Maintenant que vous savez insérer des données, apprenez à les interroger : SELECT pour extraire les données, WHERE pour filtrer, ORDER BY pour trier, et LIMIT pour paginer les résultats.

---


⏭️ [Requêtes de sélection simples (SELECT, WHERE, ORDER BY, LIMIT)](/02-bases-du-sql/07-requetes-selection-simples.md)

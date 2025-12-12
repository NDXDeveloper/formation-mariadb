🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT)

> **Niveau** : Débutant
> **Durée estimée** : 1.5 heures
> **Prérequis** : Section 2.4 (Création et modification de tables)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le rôle et l'importance des contraintes d'intégrité
- Définir des clés primaires simples et composites
- Créer des relations entre tables avec les clés étrangères
- Utiliser les contraintes UNIQUE, NOT NULL et DEFAULT efficacement
- Choisir les bonnes options CASCADE, RESTRICT, SET NULL
- Diagnostiquer et résoudre les violations de contraintes
- Appliquer les bonnes pratiques de conception relationnelle

---

## Introduction

Les **contraintes d'intégrité** sont des règles qui garantissent la qualité et la cohérence des données dans une base de données. Elles empêchent l'insertion de données invalides ou incohérentes.

### Pourquoi les contraintes sont-elles essentielles ?

Sans contraintes :
```sql
-- ❌ Données incohérentes possibles
INSERT INTO commandes (commande_id, client_id) VALUES (1, 999);
-- client_id 999 n'existe peut-être pas !

INSERT INTO utilisateurs (email) VALUES ('alice@example.com');
INSERT INTO utilisateurs (email) VALUES ('alice@example.com');
-- Doublon d'email !

INSERT INTO produits (prix) VALUES (-50.00);
-- Prix négatif accepté !
```

Avec contraintes :
```sql
-- ✅ Base de données protégée
-- Clé étrangère : Garantit que le client existe
-- UNIQUE : Empêche les doublons d'email
-- CHECK : Valide que le prix est positif
```

💡 **Principe** : Les contraintes déplacent la validation des données de l'application vers la base de données, garantissant l'intégrité même si plusieurs applications accèdent aux données.

---

## NOT NULL - Valeurs obligatoires

### Qu'est-ce que NULL ?

**NULL** représente l'absence de valeur (différent de 0, '', ou FALSE).

```sql
-- Démonstration de NULL
CREATE TABLE demo_null (
    id INT PRIMARY KEY AUTO_INCREMENT,
    valeur_optionnelle VARCHAR(50),             -- Peut être NULL
    valeur_obligatoire VARCHAR(50) NOT NULL     -- Ne peut PAS être NULL
);

-- ✅ Insertions valides
INSERT INTO demo_null (valeur_obligatoire) VALUES ('Test');
-- valeur_optionnelle = NULL automatiquement

INSERT INTO demo_null (valeur_optionnelle, valeur_obligatoire)
VALUES (NULL, 'Test2');
-- NULL explicite pour valeur_optionnelle : OK

-- ❌ Erreur : valeur obligatoire manquante
INSERT INTO demo_null (valeur_optionnelle) VALUES ('Test');
-- ERROR: Field 'valeur_obligatoire' doesn't have a default value

-- ❌ Erreur : NULL explicite sur NOT NULL
INSERT INTO demo_null (valeur_obligatoire) VALUES (NULL);
-- ERROR: Column 'valeur_obligatoire' cannot be null
```

### Quand utiliser NOT NULL ?

```sql
CREATE TABLE utilisateurs (
    utilisateur_id INT PRIMARY KEY AUTO_INCREMENT,

    -- ✅ Champs essentiels : NOT NULL
    email VARCHAR(255) NOT NULL,                -- Email toujours requis
    mot_de_passe_hash VARCHAR(255) NOT NULL,    -- Password toujours requis
    date_inscription DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- ⚠️ Champs optionnels : NULL autorisé
    nom VARCHAR(100),                           -- Peut être renseigné plus tard
    prenom VARCHAR(100),
    telephone VARCHAR(20),
    date_naissance DATE,
    bio TEXT
);
```

### Logique à trois valeurs avec NULL

```sql
-- NULL dans les comparaisons
SELECT
    NULL = NULL AS test1,           -- NULL (pas TRUE !)
    NULL != NULL AS test2,          -- NULL
    NULL > 0 AS test3,              -- NULL
    NULL AND TRUE AS test4,         -- NULL
    NULL OR TRUE AS test5;          -- TRUE

-- Tests spéciaux pour NULL
SELECT
    NULL IS NULL AS test1,          -- TRUE (correct)
    NULL IS NOT NULL AS test2,      -- FALSE
    5 IS NULL AS test3,             -- FALSE
    5 IS NOT NULL AS test4;         -- TRUE

-- Impact sur WHERE
CREATE TABLE demo_where (val INT);
INSERT INTO demo_where VALUES (1), (NULL), (3);

SELECT * FROM demo_where WHERE val = NULL;      -- 0 lignes (incorrect)
SELECT * FROM demo_where WHERE val IS NULL;     -- 1 ligne (correct)
```

⚠️ **Attention** : `WHERE colonne = NULL` ne fonctionne jamais ! Utilisez `IS NULL`.

---

## DEFAULT - Valeurs par défaut

### Définir des valeurs par défaut

```sql
CREATE TABLE parametres_utilisateur (
    utilisateur_id INT PRIMARY KEY AUTO_INCREMENT,

    -- Valeurs par défaut simples
    theme VARCHAR(20) DEFAULT 'light',
    langue VARCHAR(5) DEFAULT 'fr',
    notifications_actives BOOLEAN DEFAULT TRUE,
    items_par_page INT DEFAULT 20,

    -- Valeurs par défaut de fonctions
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    derniere_connexion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                                 ON UPDATE CURRENT_TIMESTAMP,

    -- Expression par défaut (MariaDB 10.2.1+)
    code_verification VARCHAR(6) DEFAULT (LPAD(FLOOR(RAND() * 1000000), 6, '0'))
);

-- Insertion sans spécifier les colonnes avec DEFAULT
INSERT INTO parametres_utilisateur (utilisateur_id) VALUES (1);

SELECT * FROM parametres_utilisateur WHERE utilisateur_id = 1;
-- theme: 'light'
-- langue: 'fr'
-- notifications_actives: 1 (TRUE)
-- items_par_page: 20
-- date_creation: 2025-12-12 14:30:00
```

### DEFAULT avec NOT NULL

```sql
-- Combinaison DEFAULT + NOT NULL
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,               -- Obligatoire mais valeur par défaut
    actif BOOLEAN NOT NULL DEFAULT TRUE,
    date_ajout DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ✅ Insertion minimale (stock, actif, date_ajout utilisent DEFAULT)
INSERT INTO produits (nom, prix) VALUES ('Livre', 19.99);

-- ✅ Forcer une valeur différente du DEFAULT
INSERT INTO produits (nom, prix, stock, actif) VALUES ('Stylo', 2.50, 100, FALSE);
```

### Modifier les valeurs par défaut

```sql
-- Ajouter un DEFAULT
ALTER TABLE produits
ALTER COLUMN prix SET DEFAULT 9.99;

-- Supprimer un DEFAULT
ALTER TABLE produits
ALTER COLUMN prix DROP DEFAULT;

-- Changer un DEFAULT (il faut DROP puis SET)
ALTER TABLE produits
ALTER COLUMN stock DROP DEFAULT,
ALTER COLUMN stock SET DEFAULT 5;
```

---

## PRIMARY KEY - Clé primaire

### Rôle de la clé primaire

La **clé primaire** identifie de manière unique chaque ligne dans une table :
- Valeurs **uniques** (pas de doublons)
- Valeurs **non NULL** (toujours présentes)
- **Une seule** clé primaire par table
- Crée automatiquement un **index** pour performances

### Clé primaire simple

```sql
-- Méthode 1 : Inline sur la colonne
CREATE TABLE clients (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL
);

-- Méthode 2 : Contrainte de table (plus flexible)
CREATE TABLE clients (
    client_id INT AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    PRIMARY KEY (client_id)
);

-- Méthode 3 : Avec nom de contrainte (MariaDB 10.5+)
CREATE TABLE clients (
    client_id INT AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    CONSTRAINT pk_clients PRIMARY KEY (client_id)
);
```

### Clé primaire composite (multiple colonnes)

```sql
-- Exemple : Table de liaison many-to-many
CREATE TABLE inscriptions_cours (
    etudiant_id INT NOT NULL,
    cours_id INT NOT NULL,
    date_inscription DATE NOT NULL,
    note DECIMAL(4,2),

    -- Clé primaire sur deux colonnes
    PRIMARY KEY (etudiant_id, cours_id),

    FOREIGN KEY (etudiant_id) REFERENCES etudiants(etudiant_id),
    FOREIGN KEY (cours_id) REFERENCES cours(cours_id)
);

-- Un étudiant ne peut s'inscrire qu'une fois au même cours
-- ✅ Valide : Étudiant 1 inscrit au cours 101
INSERT INTO inscriptions_cours VALUES (1, 101, '2025-01-15', NULL);

-- ✅ Valide : Étudiant 1 inscrit au cours 102 (cours différent)
INSERT INTO inscriptions_cours VALUES (1, 102, '2025-01-16', NULL);

-- ❌ Erreur : Doublon (étudiant 1, cours 101 déjà existant)
INSERT INTO inscriptions_cours VALUES (1, 101, '2025-01-20', NULL);
-- ERROR: Duplicate entry '1-101' for key 'PRIMARY'
```

### Choix de clé primaire

```sql
-- Option 1 : Clé artificielle (surrogate key) - RECOMMANDÉ
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT, -- ID artificiel
    numero_commande VARCHAR(20) UNIQUE NOT NULL, -- Numéro business
    client_id INT NOT NULL,
    date_commande DATE NOT NULL
);

-- Option 2 : Clé naturelle (natural key)
CREATE TABLE pays (
    code_iso CHAR(2) PRIMARY KEY,               -- FR, US, DE
    nom VARCHAR(100) NOT NULL,
    population BIGINT
);

-- Option 3 : Clé composite
CREATE TABLE disponibilites_medecin (
    medecin_id INT NOT NULL,
    jour DATE NOT NULL,
    heure_debut TIME NOT NULL,
    PRIMARY KEY (medecin_id, jour, heure_debut)
);
```

💡 **Recommandation** : Privilégiez les **clés artificielles** (INT AUTO_INCREMENT) pour la flexibilité.

### Ajouter/Supprimer une clé primaire

```sql
-- Ajouter une clé primaire (table sans PK)
ALTER TABLE logs ADD PRIMARY KEY (log_id);

-- Supprimer la clé primaire
ALTER TABLE logs DROP PRIMARY KEY;

-- Changer la clé primaire (il faut d'abord supprimer)
ALTER TABLE logs DROP PRIMARY KEY;
ALTER TABLE logs ADD PRIMARY KEY (nouveau_id);
```

---

## UNIQUE - Contrainte d'unicité

### Garantir l'unicité des valeurs

```sql
CREATE TABLE utilisateurs (
    utilisateur_id INT PRIMARY KEY AUTO_INCREMENT,

    -- UNIQUE inline
    email VARCHAR(255) UNIQUE NOT NULL,

    -- UNIQUE en contrainte de table
    nom_utilisateur VARCHAR(50) NOT NULL,
    UNIQUE KEY idx_username (nom_utilisateur),

    -- UNIQUE avec nom de contrainte
    telephone VARCHAR(20),
    CONSTRAINT uq_telephone UNIQUE (telephone)
);

-- ✅ Insertions valides
INSERT INTO utilisateurs (email, nom_utilisateur) VALUES
    ('alice@example.com', 'alice123'),
    ('bob@example.com', 'bob456');

-- ❌ Erreur : Email déjà existant
INSERT INTO utilisateurs (email, nom_utilisateur) VALUES
    ('alice@example.com', 'alice_new');
-- ERROR: Duplicate entry 'alice@example.com' for key 'email'

-- ✅ NULL autorisé plusieurs fois avec UNIQUE (comportement spécial)
INSERT INTO utilisateurs (email, nom_utilisateur, telephone) VALUES
    ('charlie@example.com', 'charlie', NULL),
    ('david@example.com', 'david', NULL);
-- OK : NULL != NULL en termes d'unicité
```

### UNIQUE composite

```sql
-- Unicité sur plusieurs colonnes
CREATE TABLE reservations (
    reservation_id INT PRIMARY KEY AUTO_INCREMENT,
    salle_id INT NOT NULL,
    date_reservation DATE NOT NULL,
    heure_debut TIME NOT NULL,

    -- Une salle ne peut être réservée qu'une fois à une date/heure donnée
    UNIQUE KEY uq_reservation (salle_id, date_reservation, heure_debut)
);

-- ✅ Valide : Salle 1 le 2025-12-12 à 10h
INSERT INTO reservations (salle_id, date_reservation, heure_debut)
VALUES (1, '2025-12-12', '10:00:00');

-- ✅ Valide : Salle 1 le 2025-12-12 à 14h (heure différente)
INSERT INTO reservations (salle_id, date_reservation, heure_debut)
VALUES (1, '2025-12-12', '14:00:00');

-- ❌ Erreur : Doublon complet
INSERT INTO reservations (salle_id, date_reservation, heure_debut)
VALUES (1, '2025-12-12', '10:00:00');
-- ERROR: Duplicate entry '1-2025-12-12-10:00:00' for key 'uq_reservation'
```

### UNIQUE vs PRIMARY KEY

| Critère | PRIMARY KEY | UNIQUE |
|---------|-------------|--------|
| **Nombre par table** | 1 seule | Plusieurs possibles |
| **NULL autorisé** | ❌ Non | ✅ Oui (plusieurs NULL) |
| **Crée un index** | ✅ Oui (clustered) | ✅ Oui (non-clustered) |
| **Usage** | Identifiant principal | Colonnes alternatives uniques |

```sql
CREATE TABLE exemple_difference (
    -- PRIMARY KEY : Identifiant unique, NOT NULL, une seule
    id INT PRIMARY KEY AUTO_INCREMENT,

    -- UNIQUE : Email unique, peut être NULL, plusieurs possibles
    email VARCHAR(255) UNIQUE,

    -- UNIQUE : Numéro téléphone unique
    telephone VARCHAR(20) UNIQUE,

    -- UNIQUE : Code employé unique
    code_employe VARCHAR(10) UNIQUE
);
```

---

## FOREIGN KEY - Clé étrangère

### Intégrité référentielle

Une **clé étrangère** crée un lien entre deux tables et garantit que :
- La valeur dans la colonne FK existe dans la table référencée
- Les données restent cohérentes lors des modifications/suppressions

### Syntaxe de base

```sql
-- Table parent (référencée)
CREATE TABLE clients (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Table enfant (avec FK)
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    date_commande DATE NOT NULL,
    montant DECIMAL(10,2) NOT NULL,

    -- Clé étrangère
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
);

-- ✅ Insertion valide (client existe)
INSERT INTO clients (client_id, nom, email) VALUES (1, 'Alice', 'alice@example.com');
INSERT INTO commandes (client_id, date_commande, montant) VALUES (1, '2025-12-12', 150.00);

-- ❌ Erreur : Client inexistant
INSERT INTO commandes (client_id, date_commande, montant) VALUES (999, '2025-12-12', 150.00);
-- ERROR: Cannot add or update a child row: a foreign key constraint fails
```

### Nommage des contraintes FK

```sql
-- Avec nom de contrainte (recommandé pour faciliter DROP)
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    CONSTRAINT fk_commandes_client
        FOREIGN KEY (client_id)
        REFERENCES clients(client_id)
);

-- Supprimer la FK par son nom
ALTER TABLE commandes DROP FOREIGN KEY fk_commandes_client;
```

---

## Options ON DELETE et ON UPDATE

### ON DELETE - Actions lors de suppression

| Option | Comportement |
|--------|--------------|
| **RESTRICT** | Empêche la suppression si des lignes liées existent (défaut) |
| **CASCADE** | Supprime automatiquement les lignes liées |
| **SET NULL** | Met la FK à NULL dans les lignes liées |
| **NO ACTION** | Comme RESTRICT (vérifié à la fin de la transaction) |

```sql
-- Exemple complet avec différentes options
CREATE TABLE auteurs (
    auteur_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE livres (
    livre_id INT PRIMARY KEY AUTO_INCREMENT,
    titre VARCHAR(200) NOT NULL,
    auteur_id INT NOT NULL,
    categorie_id INT,

    -- RESTRICT : Ne peut pas supprimer un auteur avec des livres
    FOREIGN KEY (auteur_id)
        REFERENCES auteurs(auteur_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);

-- Insertion de données
INSERT INTO auteurs VALUES (1, 'Victor Hugo');
INSERT INTO livres (titre, auteur_id) VALUES ('Les Misérables', 1);

-- ❌ Tentative de suppression d'auteur avec livres
DELETE FROM auteurs WHERE auteur_id = 1;
-- ERROR: Cannot delete or update a parent row: a foreign key constraint fails

-- ✅ Solution : Supprimer d'abord les livres
DELETE FROM livres WHERE auteur_id = 1;
DELETE FROM auteurs WHERE auteur_id = 1;
-- OK
```

### ON DELETE CASCADE - Suppression en cascade

```sql
CREATE TABLE clients (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    -- CASCADE : Si client supprimé, ses commandes aussi
    FOREIGN KEY (client_id)
        REFERENCES clients(client_id)
        ON DELETE CASCADE
);

CREATE TABLE lignes_commande (
    ligne_id INT PRIMARY KEY AUTO_INCREMENT,
    commande_id INT NOT NULL,
    produit VARCHAR(100),
    quantite INT,

    -- CASCADE en chaîne
    FOREIGN KEY (commande_id)
        REFERENCES commandes(commande_id)
        ON DELETE CASCADE
);

-- Insertion de données
INSERT INTO clients VALUES (1, 'Alice');
INSERT INTO commandes (client_id) VALUES (1);  -- commande_id = 1
INSERT INTO lignes_commande (commande_id, produit, quantite) VALUES (1, 'Livre', 2);

-- Suppression en cascade
DELETE FROM clients WHERE client_id = 1;
-- → Supprime automatiquement :
--   - La commande (ON DELETE CASCADE)
--   - Les lignes de commande (ON DELETE CASCADE en chaîne)

SELECT * FROM commandes;        -- Vide
SELECT * FROM lignes_commande;  -- Vide
```

⚠️ **Attention** : CASCADE est puissant mais dangereux ! Une suppression peut en entraîner des centaines d'autres.

### ON DELETE SET NULL

```sql
CREATE TABLE categories (
    categorie_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50) NOT NULL
);

CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    categorie_id INT,                           -- Peut être NULL

    -- SET NULL : Si catégorie supprimée, categorie_id devient NULL
    FOREIGN KEY (categorie_id)
        REFERENCES categories(categorie_id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);

-- Insertion
INSERT INTO categories VALUES (1, 'Électronique');
INSERT INTO produits (nom, categorie_id) VALUES ('Smartphone', 1);

SELECT * FROM produits;
-- produit_id: 1, nom: 'Smartphone', categorie_id: 1

-- Suppression de la catégorie
DELETE FROM categories WHERE categorie_id = 1;

SELECT * FROM produits;
-- produit_id: 1, nom: 'Smartphone', categorie_id: NULL
-- Produit conservé mais sans catégorie
```

### ON UPDATE CASCADE

```sql
CREATE TABLE departements (
    dept_id INT PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE employes (
    employe_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    dept_id INT NOT NULL,

    FOREIGN KEY (dept_id)
        REFERENCES departements(dept_id)
        ON UPDATE CASCADE                       -- Mise à jour en cascade
        ON DELETE RESTRICT
);

-- Insertion
INSERT INTO departements VALUES (10, 'IT');
INSERT INTO employes (nom, dept_id) VALUES ('Alice', 10);

-- Changement d'ID de département
UPDATE departements SET dept_id = 20 WHERE dept_id = 10;

-- ✅ dept_id mis à jour automatiquement dans employes
SELECT * FROM employes;
-- employe_id: 1, nom: 'Alice', dept_id: 20 (mis à jour automatiquement)
```

---

## CHECK - Contraintes de vérification

### Valider les données avec CHECK

```sql
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL,
    reduction DECIMAL(5,2) DEFAULT 0,
    note_moyenne DECIMAL(3,2),

    -- Contraintes CHECK
    CHECK (prix > 0),                           -- Prix positif
    CHECK (stock >= 0),                         -- Stock non négatif
    CHECK (reduction >= 0 AND reduction <= 100), -- Réduction 0-100%
    CHECK (note_moyenne IS NULL OR (note_moyenne >= 0 AND note_moyenne <= 5))
);

-- ✅ Insertions valides
INSERT INTO produits (nom, prix, stock, reduction, note_moyenne) VALUES
    ('Livre', 19.99, 10, 15.00, 4.5),
    ('Stylo', 2.50, 100, 0, NULL);

-- ❌ Erreur : Prix négatif
INSERT INTO produits (nom, prix, stock) VALUES ('Test', -10.00, 5);
-- ERROR: Check constraint 'produits_chk_1' is violated

-- ❌ Erreur : Réduction > 100
INSERT INTO produits (nom, prix, stock, reduction) VALUES ('Test', 10.00, 5, 150);
-- ERROR: Check constraint 'produits_chk_3' is violated
```

### CHECK avec nom de contrainte

```sql
CREATE TABLE employes (
    employe_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    age INT NOT NULL,
    salaire DECIMAL(10,2) NOT NULL,
    email VARCHAR(255),

    -- CHECK avec noms explicites
    CONSTRAINT chk_age_valide
        CHECK (age >= 18 AND age <= 70),

    CONSTRAINT chk_salaire_positif
        CHECK (salaire > 0),

    CONSTRAINT chk_email_format
        CHECK (email LIKE '%_@__%.__%')         -- Format email basique
);

-- Supprimer une contrainte CHECK
ALTER TABLE employes DROP CONSTRAINT chk_email_format;

-- Ajouter une contrainte CHECK
ALTER TABLE employes
ADD CONSTRAINT chk_salaire_minimum CHECK (salaire >= 2000);
```

---

## Exemple complet : Système de e-commerce

```sql
-- ============================================
-- Système de e-commerce avec contraintes
-- ============================================

-- 1. Table clients
CREATE TABLE clients (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,         -- UNIQUE : Email unique
    telephone VARCHAR(20) UNIQUE,               -- UNIQUE : Téléphone unique
    date_inscription DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actif BOOLEAN NOT NULL DEFAULT TRUE,        -- NOT NULL + DEFAULT

    -- CHECK : Email valide
    CONSTRAINT chk_email_format CHECK (email LIKE '%_@__%.__%'),

    INDEX idx_email (email),
    INDEX idx_actif (actif)
) ENGINE=InnoDB;

-- 2. Table categories
CREATE TABLE categories (
    categorie_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50) UNIQUE NOT NULL,            -- UNIQUE : Nom unique
    slug VARCHAR(50) UNIQUE NOT NULL,           -- UNIQUE : Slug unique
    parent_id INT,                              -- Peut être NULL (catégorie racine)

    -- FK récursive (auto-référence)
    FOREIGN KEY (parent_id)
        REFERENCES categories(categorie_id)
        ON DELETE SET NULL
) ENGINE=InnoDB;

-- 3. Table produits
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(200) NOT NULL,                  -- NOT NULL : Obligatoire
    slug VARCHAR(200) UNIQUE NOT NULL,          -- UNIQUE + NOT NULL
    description TEXT,
    prix DECIMAL(10,2) NOT NULL,                -- NOT NULL : Prix obligatoire
    stock INT NOT NULL DEFAULT 0,               -- NOT NULL + DEFAULT
    categorie_id INT,
    actif BOOLEAN NOT NULL DEFAULT TRUE,
    date_ajout DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- FK vers categories
    FOREIGN KEY (categorie_id)
        REFERENCES categories(categorie_id)
        ON DELETE SET NULL,                     -- Si catégorie supprimée → NULL

    -- Contraintes CHECK
    CONSTRAINT chk_prix_positif CHECK (prix > 0),
    CONSTRAINT chk_stock_positif CHECK (stock >= 0),

    INDEX idx_categorie (categorie_id),
    INDEX idx_actif (actif),
    FULLTEXT INDEX idx_fulltext (nom, description)
) ENGINE=InnoDB;

-- 4. Table commandes
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,                     -- NOT NULL : Client obligatoire
    numero_commande VARCHAR(20) UNIQUE NOT NULL, -- UNIQUE : Numéro unique
    statut ENUM('en_attente', 'confirmee', 'expediee', 'livree', 'annulee')
        NOT NULL DEFAULT 'en_attente',
    montant_total DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    date_commande DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    date_expedition DATETIME,

    -- FK vers clients avec CASCADE
    FOREIGN KEY (client_id)
        REFERENCES clients(client_id)
        ON DELETE RESTRICT                      -- Ne pas supprimer client avec commandes
        ON UPDATE CASCADE,

    -- CHECK : Montant positif
    CONSTRAINT chk_montant_positif CHECK (montant_total >= 0),

    INDEX idx_client (client_id),
    INDEX idx_statut (statut),
    INDEX idx_date (date_commande)
) ENGINE=InnoDB;

-- 5. Table lignes_commande (détails)
CREATE TABLE lignes_commande (
    ligne_id INT PRIMARY KEY AUTO_INCREMENT,
    commande_id INT NOT NULL,                   -- NOT NULL : Commande obligatoire
    produit_id INT NOT NULL,                    -- NOT NULL : Produit obligatoire
    quantite INT NOT NULL,                      -- NOT NULL : Quantité obligatoire
    prix_unitaire DECIMAL(10,2) NOT NULL,       -- NOT NULL : Prix obligatoire

    -- FK vers commandes avec CASCADE
    FOREIGN KEY (commande_id)
        REFERENCES commandes(commande_id)
        ON DELETE CASCADE,                      -- Si commande supprimée, lignes aussi

    -- FK vers produits avec RESTRICT
    FOREIGN KEY (produit_id)
        REFERENCES produits(produit_id)
        ON DELETE RESTRICT,                     -- Ne pas supprimer produit commandé

    -- CHECK : Quantité et prix positifs
    CONSTRAINT chk_quantite_positive CHECK (quantite > 0),
    CONSTRAINT chk_prix_unitaire_positif CHECK (prix_unitaire > 0),

    INDEX idx_commande (commande_id),
    INDEX idx_produit (produit_id)
) ENGINE=InnoDB;

-- 6. Table avis_produits
CREATE TABLE avis_produits (
    avis_id INT PRIMARY KEY AUTO_INCREMENT,
    produit_id INT NOT NULL,
    client_id INT NOT NULL,
    note TINYINT NOT NULL,                      -- NOT NULL : Note obligatoire
    commentaire TEXT,
    date_avis DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- FK vers produits et clients
    FOREIGN KEY (produit_id)
        REFERENCES produits(produit_id)
        ON DELETE CASCADE,

    FOREIGN KEY (client_id)
        REFERENCES clients(client_id)
        ON DELETE CASCADE,

    -- UNIQUE : Un client ne peut donner qu'un avis par produit
    UNIQUE KEY uq_client_produit (client_id, produit_id),

    -- CHECK : Note entre 1 et 5
    CONSTRAINT chk_note_valide CHECK (note >= 1 AND note <= 5),

    INDEX idx_produit (produit_id),
    INDEX idx_client (client_id)
) ENGINE=InnoDB;

-- Vérification des contraintes
SELECT
    TABLE_NAME,
    CONSTRAINT_NAME,
    CONSTRAINT_TYPE
FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME, CONSTRAINT_TYPE;
```

---

## Diagnostiquer les violations de contraintes

### Identifier les violations de FK

```sql
-- Trouver les lignes orphelines (FK invalides)
-- Exemple : Commandes sans client
SELECT c.*
FROM commandes c
LEFT JOIN clients cl ON c.client_id = cl.client_id
WHERE cl.client_id IS NULL;

-- Nettoyer avant d'ajouter une FK
DELETE FROM commandes
WHERE client_id NOT IN (SELECT client_id FROM clients);

-- Ajouter la FK
ALTER TABLE commandes
ADD CONSTRAINT fk_commandes_client
    FOREIGN KEY (client_id) REFERENCES clients(client_id);
```

### Messages d'erreur courants

```sql
-- Erreur 1 : Duplicate entry (UNIQUE/PRIMARY KEY)
-- ERROR 1062: Duplicate entry 'alice@example.com' for key 'email'
-- Solution : Vérifier les doublons avant insertion

-- Erreur 2 : Cannot be null (NOT NULL)
-- ERROR 1048: Column 'nom' cannot be null
-- Solution : Fournir une valeur ou ajouter un DEFAULT

-- Erreur 3 : Foreign key constraint fails
-- ERROR 1452: Cannot add or update a child row: a foreign key constraint fails
-- Solution : Vérifier que la valeur existe dans la table parent

-- Erreur 4 : Cannot delete parent row
-- ERROR 1451: Cannot delete or update a parent row: a foreign key constraint fails
-- Solution : Supprimer d'abord les lignes enfants ou utiliser CASCADE

-- Erreur 5 : Check constraint violated
-- ERROR 4025: Check constraint 'chk_prix_positif' is violated
-- Solution : Ajuster la valeur pour respecter la contrainte
```

---

## Bonnes pratiques

### 1. Nommer explicitement les contraintes

```sql
-- ✅ BON : Noms explicites
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY,
    client_id INT NOT NULL,

    CONSTRAINT fk_commandes_client
        FOREIGN KEY (client_id)
        REFERENCES clients(client_id),

    CONSTRAINT chk_montant_positif
        CHECK (montant_total >= 0)
);

-- ❌ MOINS BON : Noms auto-générés
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY,
    client_id INT NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(client_id),
    CHECK (montant_total >= 0)
);
-- Contraintes nommées : commandes_ibfk_1, commandes_chk_1 (peu clair)
```

### 2. Indexer les clés étrangères

```sql
-- ✅ BON : Index sur FK
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY,
    client_id INT NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(client_id),
    INDEX idx_client (client_id)                -- Index pour JOIN rapides
);

-- Sans index, les JOIN sur client_id seront lents
```

### 3. Choisir les bonnes options CASCADE/RESTRICT

```sql
-- ✅ BON : Réfléchir aux implications métier

-- RESTRICT : Données critiques (ne jamais perdre)
CREATE TABLE factures (
    client_id INT,
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
        ON DELETE RESTRICT                      -- Ne pas supprimer client avec factures
);

-- CASCADE : Données dépendantes (suppression logique)
CREATE TABLE sessions_utilisateur (
    utilisateur_id INT,
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(utilisateur_id)
        ON DELETE CASCADE                       -- Supprimer sessions si utilisateur supprimé
);

-- SET NULL : Données optionnelles
CREATE TABLE produits (
    categorie_id INT,
    FOREIGN KEY (categorie_id) REFERENCES categories(categorie_id)
        ON DELETE SET NULL                      -- Produit sans catégorie OK
);
```

### 4. Documenter les contraintes

```sql
CREATE TABLE produits (
    produit_id INT PRIMARY KEY AUTO_INCREMENT,
    prix DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,

    -- Contraintes documentées
    CONSTRAINT chk_prix_positif
        CHECK (prix > 0)
        /* Le prix doit être strictement positif */,

    CONSTRAINT chk_stock_positif
        CHECK (stock >= 0)
        /* Le stock ne peut pas être négatif */
) COMMENT = 'Table des produits avec validation des prix et stocks';
```

### 5. Tester les contraintes

```sql
-- Script de test des contraintes
START TRANSACTION;

-- Test 1 : NOT NULL
INSERT INTO clients (nom, email) VALUES (NULL, 'test@example.com');
-- Devrait échouer

-- Test 2 : UNIQUE
INSERT INTO clients (nom, email) VALUES ('Test1', 'test@example.com');
INSERT INTO clients (nom, email) VALUES ('Test2', 'test@example.com');
-- La 2e devrait échouer

-- Test 3 : FOREIGN KEY
INSERT INTO commandes (client_id) VALUES (99999);
-- Devrait échouer si client 99999 n'existe pas

-- Test 4 : CHECK
INSERT INTO produits (nom, prix, stock) VALUES ('Test', -10, 5);
-- Devrait échouer

ROLLBACK;  -- Annuler tous les tests
```

---

## ✅ Points clés à retenir

- **NOT NULL** : Empêche les valeurs NULL, garantit présence de données
- **DEFAULT** : Fournit valeur par défaut si non spécifiée
- **PRIMARY KEY** : Identifiant unique, NOT NULL, une seule par table
- **UNIQUE** : Garantit unicité, plusieurs possibles, NULL autorisé
- **FOREIGN KEY** : Maintient intégrité référentielle entre tables
- **CASCADE** : Propage suppressions/modifications automatiquement
- **RESTRICT** : Empêche suppressions si données liées (défaut)
- **SET NULL** : Met FK à NULL lors de suppression parent
- **CHECK** : Valide données selon condition logique
- **Nommer contraintes** : Facilite gestion et debugging
- **Indexer FK** : Améliore performances des JOIN
- **Tester contraintes** : Valider comportement avant production
- **NULL IS NULL** : Utiliser IS NULL, pas = NULL
- **Documenter** : Expliquer la logique métier des contraintes

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Constraints](https://mariadb.com/kb/en/constraint/)
- [📖 FOREIGN KEY Constraints](https://mariadb.com/kb/en/foreign-keys/)
- [📖 Primary Keys](https://mariadb.com/kb/en/getting-started-with-indexes/#primary-key)
- [📖 UNIQUE Constraint](https://mariadb.com/kb/en/unique/)
- [📖 CHECK Constraint](https://mariadb.com/kb/en/check/)
- [📖 NULL Values](https://mariadb.com/kb/en/null-values/)

### Lectures complémentaires
- [Database Integrity Rules](https://www.vertabelo.com/blog/database-integrity/)
- [Referential Integrity Best Practices](https://use-the-index-luke.com/)

---

## ➡️ Section suivante

**[2.6 Insertion de données (INSERT, INSERT INTO SELECT, LOAD DATA)](/02-bases-du-sql/06-insertion-donnees.md)**

Maintenant que vous maîtrisez la structure des tables et leurs contraintes, apprenez à insérer des données efficacement : INSERT simple et multiple, INSERT INTO SELECT pour copier des données, et LOAD DATA pour importer des fichiers CSV en masse.

---


⏭️ [Insertion de données (INSERT, INSERT INTO SELECT, LOAD DATA)](/02-bases-du-sql/06-insertion-donnees.md)

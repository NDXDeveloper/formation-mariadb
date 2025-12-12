🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3 Vues updatable : Conditions et limitations

> **Niveau** : Intermédiaire
> **Durée estimée** : 1.5-2 heures
> **Prérequis** : Sections 9.1-9.2, compréhension des jointures et contraintes d'intégrité

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre ce qu'est une vue updatable et quand elle peut être utilisée
- Identifier les conditions strictes qui rendent une vue modifiable (INSERT, UPDATE, DELETE)
- Utiliser INFORMATION_SCHEMA pour vérifier si une vue est updatable
- Effectuer des opérations de modification de données à travers des vues
- Comprendre les limitations liées aux jointures, agrégations et sous-requêtes
- Anticiper les erreurs courantes lors de la modification de vues
- Choisir entre vue updatable et accès direct aux tables selon le contexte

---

## Introduction

### Qu'est-ce qu'une vue updatable ?

Une **vue updatable** (ou vue modifiable) est une vue qui permet non seulement de lire des données (SELECT), mais aussi de les **modifier** via les opérations INSERT, UPDATE et DELETE. Lorsqu'une modification est effectuée sur la vue, elle est automatiquement répercutée sur la ou les tables sous-jacentes.

```sql
-- Créer une vue simple
CREATE VIEW v_clients_actifs AS
SELECT
    id,
    nom,
    prenom,
    email,
    telephone
FROM clients
WHERE statut = 'ACTIF';

-- ✅ Cette vue est updatable : on peut modifier les données
UPDATE v_clients_actifs
SET telephone = '0123456789'
WHERE id = 100;

-- La modification est appliquée à la table 'clients' sous-jacente
```

### Pourquoi utiliser des vues updatable ?

Les vues updatable offrent plusieurs avantages :

1. **Abstraction de la complexité** : Simplifier l'accès aux données pour les applications
2. **Sécurité** : Restreindre les colonnes ou lignes modifiables
3. **Validation métier** : Combiner avec WITH CHECK OPTION pour garantir l'intégrité
4. **Compatibilité** : Maintenir une interface stable même si le schéma change

```sql
-- Exemple : Vue qui masque les colonnes sensibles
CREATE VIEW v_clients_publics AS
SELECT
    id,
    nom,
    email
    -- Masque : num_secu, salaire, date_naissance
FROM clients;

-- Les applications peuvent modifier uniquement les champs exposés
UPDATE v_clients_publics
SET email = 'nouveau@email.com'
WHERE id = 50;
-- ✅ Fonctionne : email est dans la vue

UPDATE v_clients_publics
SET salaire = 50000
WHERE id = 50;
-- ❌ Erreur : salaire n'est pas dans la vue
```

### Vue updatable vs non-updatable

| Type de vue | INSERT | UPDATE | DELETE | Exemple |
|-------------|--------|--------|--------|---------|
| **Updatable** | ✅ | ✅ | ✅ | SELECT col FROM table WHERE ... |
| **Non-updatable** | ❌ | ❌ | ❌ | SELECT COUNT(*) FROM table GROUP BY ... |

---

## Conditions pour qu'une vue soit updatable

MariaDB impose des **règles strictes** pour déterminer si une vue est updatable. Voici les conditions principales :

### Règles de base (MUST)

Pour qu'une vue soit updatable, elle **DOIT** :

1. ✅ **Référencer une seule table** dans le FROM (ou des jointures simples sous conditions strictes)
2. ✅ **Ne pas contenir d'agrégations** (COUNT, SUM, AVG, MIN, MAX)
3. ✅ **Ne pas contenir DISTINCT**
4. ✅ **Ne pas contenir GROUP BY ou HAVING**
5. ✅ **Ne pas contenir UNION, UNION ALL, ou INTERSECT**
6. ✅ **Ne pas contenir de sous-requêtes dans le SELECT**
7. ✅ **Ne pas référencer de colonnes calculées** (sauf pour UPDATE dans certains cas)
8. ✅ **Ne pas utiliser ALGORITHM = TEMPTABLE**

### Règle simplifiée

> Une vue est updatable si elle représente un **mapping 1:1** simple entre les lignes de la vue et les lignes de la table sous-jacente, sans transformation complexe.

---

## Vérifier si une vue est updatable

### Utiliser INFORMATION_SCHEMA.VIEWS

```sql
-- Vérifier l'updatabilité d'une vue
SELECT
    TABLE_NAME,
    IS_UPDATABLE,
    VIEW_DEFINITION
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'v_clients_actifs';

-- Résultat :
-- TABLE_NAME         | IS_UPDATABLE | VIEW_DEFINITION
-- v_clients_actifs   | YES          | select `id`,`nom`,...
```

**Valeurs possibles** :
- **YES** : La vue est updatable (INSERT, UPDATE, DELETE possibles)
- **NO** : La vue n'est pas updatable

### Lister toutes les vues updatable

```sql
-- Trouver toutes les vues updatable de la base
SELECT
    TABLE_NAME,
    DEFINER,
    SECURITY_TYPE
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND IS_UPDATABLE = 'YES'
ORDER BY TABLE_NAME;
```

---

## Vues updatable : Exemples qui fonctionnent

### Exemple 1 : Vue simple avec filtrage

```sql
-- Table source
CREATE TABLE employes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    departement VARCHAR(50),
    salaire DECIMAL(10,2),
    statut ENUM('ACTIF', 'INACTIF') DEFAULT 'ACTIF'
) ENGINE=InnoDB;

-- Vue updatable simple
CREATE VIEW v_employes_actifs AS
SELECT
    id,
    nom,
    prenom,
    email,
    departement,
    salaire
FROM employes
WHERE statut = 'ACTIF';

-- ✅ INSERT fonctionne
INSERT INTO v_employes_actifs (nom, prenom, email, departement, salaire)
VALUES ('Dupont', 'Marie', 'marie.dupont@example.com', 'IT', 45000);

-- ✅ UPDATE fonctionne
UPDATE v_employes_actifs
SET salaire = 48000
WHERE nom = 'Dupont' AND prenom = 'Marie';

-- ✅ DELETE fonctionne
DELETE FROM v_employes_actifs
WHERE id = 1;
```

**Vérification** :

```sql
SELECT IS_UPDATABLE
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_employes_actifs';
-- Résultat : YES
```

### Exemple 2 : Vue avec sélection de colonnes

```sql
-- Vue qui expose uniquement certaines colonnes
CREATE VIEW v_contacts_publics AS
SELECT
    id,
    nom,
    prenom,
    email
    -- Masque : telephone, adresse, date_naissance
FROM clients;

-- ✅ UPDATE sur colonnes exposées fonctionne
UPDATE v_contacts_publics
SET email = 'nouveau@email.com'
WHERE id = 10;

-- ❌ UPDATE sur colonne masquée échoue
UPDATE v_contacts_publics
SET telephone = '0123456789'
WHERE id = 10;
-- Error: Unknown column 'telephone' in 'field list'

-- ✅ INSERT fonctionne (colonnes omises prennent DEFAULT ou NULL)
INSERT INTO v_contacts_publics (nom, prenom, email)
VALUES ('Martin', 'Paul', 'paul.martin@example.com');
-- Les colonnes masquées (telephone, adresse) auront NULL ou DEFAULT
```

### Exemple 3 : Vue avec calcul simple (colonne générée)

```sql
-- Vue avec colonne calculée
CREATE VIEW v_produits_avec_tva AS
SELECT
    id,
    nom,
    prix_ht,
    (prix_ht * 1.20) AS prix_ttc  -- Colonne calculée
FROM produits;

-- ✅ UPDATE sur colonne réelle fonctionne
UPDATE v_produits_avec_tva
SET prix_ht = 100
WHERE id = 5;

-- ❌ UPDATE sur colonne calculée échoue
UPDATE v_produits_avec_tva
SET prix_ttc = 120
WHERE id = 5;
-- Error: Column 'prix_ttc' is not updatable

-- Vérification
SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_produits_avec_tva';
-- Résultat : YES (la vue est updatable pour les colonnes non calculées)
```

---

## Vues NON-updatable : Exemples qui ne fonctionnent PAS

### Exemple 1 : Vue avec agrégation (GROUP BY)

```sql
-- ❌ Vue NON-updatable : contient GROUP BY
CREATE VIEW v_stats_departements AS
SELECT
    departement,
    COUNT(*) AS nb_employes,
    AVG(salaire) AS salaire_moyen
FROM employes
GROUP BY departement;

-- Vérification
SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_stats_departements';
-- Résultat : NO

-- ❌ INSERT échoue
INSERT INTO v_stats_departements (departement, nb_employes, salaire_moyen)
VALUES ('Marketing', 10, 50000);
-- Error: The target table v_stats_departements of the INSERT is not insertable-into

-- ❌ UPDATE échoue
UPDATE v_stats_departements
SET salaire_moyen = 55000
WHERE departement = 'IT';
-- Error: The target table v_stats_departements of the UPDATE is not updatable
```

### Exemple 2 : Vue avec DISTINCT

```sql
-- ❌ Vue NON-updatable : contient DISTINCT
CREATE VIEW v_villes_uniques AS
SELECT DISTINCT ville
FROM clients;

SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_villes_uniques';
-- Résultat : NO

-- ❌ Toute modification échoue
UPDATE v_villes_uniques SET ville = 'Paris' WHERE ville = 'PARIS';
-- Error: The target table v_villes_uniques of the UPDATE is not updatable
```

### Exemple 3 : Vue avec UNION

```sql
-- ❌ Vue NON-updatable : contient UNION
CREATE VIEW v_contacts_tous AS
SELECT id, nom, email FROM clients
UNION
SELECT id, nom, email FROM prospects;

SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_contacts_tous';
-- Résultat : NO

-- ❌ Modification impossible
DELETE FROM v_contacts_tous WHERE id = 10;
-- Error: Can not delete from join view
```

### Exemple 4 : Vue avec sous-requête dans SELECT

```sql
-- ❌ Vue NON-updatable : sous-requête dans SELECT
CREATE VIEW v_clients_avec_nb_commandes AS
SELECT
    c.id,
    c.nom,
    c.email,
    (SELECT COUNT(*) FROM commandes WHERE client_id = c.id) AS nb_commandes
FROM clients c;

SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_clients_avec_nb_commandes';
-- Résultat : NO

-- ❌ Modification impossible
UPDATE v_clients_avec_nb_commandes SET email = 'test@test.com' WHERE id = 1;
-- Error: The target table v_clients_avec_nb_commandes of the UPDATE is not updatable
```

### Exemple 5 : Vue avec ALGORITHM = TEMPTABLE

```sql
-- ❌ Vue NON-updatable : TEMPTABLE forcé
CREATE ALGORITHM = TEMPTABLE VIEW v_produits_temptable AS
SELECT id, nom, prix FROM produits;

SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_produits_temptable';
-- Résultat : NO

-- ❌ Modification impossible
UPDATE v_produits_temptable SET prix = 100 WHERE id = 5;
-- Error: The target table v_produits_temptable of the UPDATE is not updatable
```

---

## Vues avec jointures : Cas particuliers

Les vues avec **jointures** ont des règles d'updatabilité plus complexes.

### Règle générale

> Une vue avec jointure peut être partiellement updatable : on peut modifier uniquement les colonnes de **la table de gauche** (table principale), et sous certaines conditions.

### Exemple : Vue avec LEFT JOIN (partiellement updatable)

```sql
-- Tables sources
CREATE TABLE clients (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255)
) ENGINE=InnoDB;

CREATE TABLE adresses (
    id INT PRIMARY KEY,
    client_id INT,
    rue VARCHAR(255),
    ville VARCHAR(100),
    FOREIGN KEY (client_id) REFERENCES clients(id)
) ENGINE=InnoDB;

-- Vue avec LEFT JOIN
CREATE VIEW v_clients_avec_adresses AS
SELECT
    c.id AS client_id,
    c.nom AS client_nom,
    c.email AS client_email,
    a.rue,
    a.ville
FROM clients c
LEFT JOIN adresses a ON c.id = a.client_id;

-- Vérification
SELECT IS_UPDATABLE FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_clients_avec_adresses';
-- Résultat : YES (partiellement)

-- ✅ UPDATE sur colonnes de la table principale (clients) fonctionne
UPDATE v_clients_avec_adresses
SET client_email = 'nouveau@email.com'
WHERE client_id = 10;

-- ❌ UPDATE sur colonnes de la table jointe (adresses) échoue
UPDATE v_clients_avec_adresses
SET ville = 'Paris'
WHERE client_id = 10;
-- Error: Can not modify more than one base table through a join view

-- ❌ INSERT échoue (ambiguïté : quelle table ?)
INSERT INTO v_clients_avec_adresses (client_nom, client_email, rue, ville)
VALUES ('Test', 'test@test.com', '10 rue Test', 'Lyon');
-- Error: Can not insert into join view 'v_clients_avec_adresses' without fields list
```

### Explication du comportement

Avec une vue contenant une jointure :

- **UPDATE** : Possible uniquement sur les colonnes de **la table de gauche** (table principale du FROM)
- **INSERT** : Généralement impossible (quelle table cibler ?)
- **DELETE** : Possible, supprime la ligne de la table principale

```sql
-- ✅ DELETE fonctionne (supprime le client)
DELETE FROM v_clients_avec_adresses WHERE client_id = 10;
-- Supprime la ligne dans 'clients', pas dans 'adresses'
```

💡 **Conseil** : Pour les vues avec jointures, préférez l'accès direct aux tables pour les modifications, et utilisez les vues uniquement en lecture.

---

## Opérations INSERT sur les vues updatable

### INSERT simple

```sql
-- Vue updatable
CREATE VIEW v_produits_disponibles AS
SELECT
    id,
    nom,
    prix,
    stock
FROM produits
WHERE stock > 0;

-- ✅ INSERT fonctionne
INSERT INTO v_produits_disponibles (nom, prix, stock)
VALUES ('Nouveau Produit', 29.99, 100);

-- Vérification : la ligne est ajoutée dans la table 'produits'
SELECT * FROM produits WHERE nom = 'Nouveau Produit';
```

### INSERT avec colonnes manquantes (DEFAULT ou NULL)

```sql
-- Table avec colonnes DEFAULT et NOT NULL
CREATE TABLE articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    titre VARCHAR(255) NOT NULL,
    contenu TEXT,
    auteur_id INT NOT NULL,
    date_publication TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    statut ENUM('BROUILLON', 'PUBLIE') DEFAULT 'BROUILLON'
) ENGINE=InnoDB;

-- Vue exposant seulement certaines colonnes
CREATE VIEW v_articles_simples AS
SELECT
    id,
    titre,
    contenu,
    auteur_id
FROM articles;

-- ✅ INSERT fonctionne (colonnes omises prennent DEFAULT)
INSERT INTO v_articles_simples (titre, contenu, auteur_id)
VALUES ('Mon article', 'Contenu...', 5);
-- date_publication = NOW(), statut = 'BROUILLON'

-- ❌ INSERT échoue si colonne NOT NULL omise sans DEFAULT
INSERT INTO v_articles_simples (titre, contenu)
VALUES ('Test', 'Contenu');
-- Error: Field 'auteur_id' doesn't have a default value
```

### Problème : INSERT avec clause WHERE de la vue

⚠️ **Piège important** : Un INSERT via une vue peut créer des lignes qui **ne satisfont pas** la clause WHERE de la vue !

```sql
-- Vue : employés du département IT
CREATE VIEW v_employes_it AS
SELECT id, nom, prenom, email, departement, salaire
FROM employes
WHERE departement = 'IT';

-- ❌ INSERT qui viole la clause WHERE
INSERT INTO v_employes_it (nom, prenom, email, departement, salaire)
VALUES ('Martin', 'Sophie', 'sophie@example.com', 'RH', 40000);
-- ✅ L'INSERT réussit !

-- Mais la ligne créée n'est PAS visible dans la vue
SELECT * FROM v_employes_it WHERE nom = 'Martin';
-- Résultat : 0 ligne (car departement = 'RH', pas 'IT')

-- La ligne existe dans la table sous-jacente
SELECT * FROM employes WHERE nom = 'Martin';
-- Résultat : Sophie Martin, RH, ...
```

**Explication** : L'INSERT se fait directement dans la table `employes` sans vérification de la clause WHERE de la vue. Pour éviter ce problème, utilisez **WITH CHECK OPTION** (voir section 9.4).

---

## Opérations UPDATE sur les vues updatable

### UPDATE simple

```sql
-- Vue updatable
CREATE VIEW v_clients_actifs AS
SELECT id, nom, email, telephone, ville
FROM clients
WHERE statut = 'ACTIF';

-- ✅ UPDATE fonctionne
UPDATE v_clients_actifs
SET telephone = '0612345678',
    ville = 'Paris'
WHERE id = 100;

-- Vérification : la table sous-jacente est modifiée
SELECT * FROM clients WHERE id = 100;
```

### UPDATE avec clause WHERE de la vue

Comme pour INSERT, un UPDATE peut faire "disparaître" une ligne de la vue :

```sql
-- Vue : produits en stock
CREATE VIEW v_produits_en_stock AS
SELECT id, nom, prix, stock
FROM produits
WHERE stock > 0;

-- ❌ UPDATE qui fait sortir la ligne de la vue
UPDATE v_produits_en_stock
SET stock = 0
WHERE id = 50;
-- ✅ L'UPDATE réussit

-- Mais la ligne n'est plus visible dans la vue
SELECT * FROM v_produits_en_stock WHERE id = 50;
-- Résultat : 0 ligne (car stock = 0, donc WHERE stock > 0 est false)

-- La ligne existe toujours dans la table
SELECT * FROM produits WHERE id = 50;
-- Résultat : produit avec stock = 0
```

**Solution** : Utilisez WITH CHECK OPTION pour empêcher ce comportement.

### UPDATE sur colonnes calculées (non possible)

```sql
-- Vue avec colonne calculée
CREATE VIEW v_produits_avec_remise AS
SELECT
    id,
    nom,
    prix,
    (prix * 0.9) AS prix_remise  -- Colonne calculée
FROM produits;

-- ✅ UPDATE sur colonne réelle
UPDATE v_produits_avec_remise
SET prix = 100
WHERE id = 10;

-- ❌ UPDATE sur colonne calculée échoue
UPDATE v_produits_avec_remise
SET prix_remise = 90
WHERE id = 10;
-- Error: Column 'prix_remise' is not updatable
```

---

## Opérations DELETE sur les vues updatable

### DELETE simple

```sql
-- Vue updatable
CREATE VIEW v_articles_anciens AS
SELECT id, titre, date_publication
FROM articles
WHERE date_publication < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);

-- ✅ DELETE fonctionne
DELETE FROM v_articles_anciens
WHERE id = 150;

-- La ligne est supprimée de la table 'articles'
SELECT * FROM articles WHERE id = 150;
-- Résultat : 0 ligne (supprimée)
```

### DELETE avec jointure

```sql
-- Vue avec jointure
CREATE VIEW v_commandes_clients AS
SELECT
    c.id AS commande_id,
    c.date_commande,
    cl.nom AS client_nom
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id;

-- ✅ DELETE fonctionne (supprime de la table principale : commandes)
DELETE FROM v_commandes_clients WHERE commande_id = 100;
-- Supprime la ligne dans 'commandes', pas dans 'clients'
```

---

## Limitations et restrictions importantes

### Récapitulatif des restrictions

| Élément dans la vue | INSERT | UPDATE | DELETE | Updatable |
|---------------------|--------|--------|--------|-----------|
| **Table unique** | ✅ | ✅ | ✅ | ✅ |
| **WHERE simple** | ✅ | ✅ | ✅ | ✅ |
| **Colonnes calculées** | ⚠️ Ignorées | ❌ Non modifiables | - | ⚠️ Partiellement |
| **JOIN** | ❌ | ⚠️ Table gauche seulement | ⚠️ Table gauche | ⚠️ Partiellement |
| **GROUP BY** | ❌ | ❌ | ❌ | ❌ |
| **DISTINCT** | ❌ | ❌ | ❌ | ❌ |
| **Agrégations** | ❌ | ❌ | ❌ | ❌ |
| **UNION** | ❌ | ❌ | ❌ | ❌ |
| **Sous-requêtes SELECT** | ❌ | ❌ | ❌ | ❌ |
| **ALGORITHM=TEMPTABLE** | ❌ | ❌ | ❌ | ❌ |

### Limitations spécifiques par opération

#### INSERT

```sql
-- ❌ Ne peut pas insérer dans les colonnes calculées
-- ❌ Colonnes NOT NULL sans DEFAULT doivent être fournies
-- ❌ Ne peut pas insérer si la vue contient une jointure
-- ⚠️ Peut créer des lignes invisibles dans la vue (sans WITH CHECK OPTION)
```

#### UPDATE

```sql
-- ❌ Ne peut pas modifier les colonnes calculées
-- ❌ Ne peut pas modifier plusieurs tables via une jointure
-- ⚠️ Peut faire sortir des lignes de la vue (sans WITH CHECK OPTION)
```

#### DELETE

```sql
-- ❌ Avec jointure, supprime uniquement de la table principale
-- ⚠️ Ne vérifie pas les contraintes de la vue après suppression
```

---

## Alternatives aux vues updatable

Si une vue n'est pas updatable, plusieurs alternatives existent :

### Alternative 1 : INSTEAD OF Triggers (non supporté dans MariaDB)

⚠️ MariaDB ne supporte **pas** les triggers INSTEAD OF sur les vues (contrairement à SQL Server ou PostgreSQL).

```sql
-- ❌ Non supporté dans MariaDB
-- CREATE TRIGGER trg_instead_of_insert ON v_vue
-- INSTEAD OF INSERT AS ...
```

### Alternative 2 : Procédures stockées

Encapsuler la logique de modification dans des procédures :

```sql
-- Vue non-updatable avec agrégation
CREATE VIEW v_stats_produits AS
SELECT
    categorie_id,
    COUNT(*) AS nb_produits,
    AVG(prix) AS prix_moyen
FROM produits
GROUP BY categorie_id;

-- ❌ UPDATE impossible directement
-- UPDATE v_stats_produits SET prix_moyen = 100 WHERE categorie_id = 5;

-- ✅ Solution : Procédure stockée pour modifier les données sources
DELIMITER //

CREATE PROCEDURE sp_update_prix_categorie(
    IN p_categorie_id INT,
    IN p_pourcentage_augmentation DECIMAL(5,2)
)
BEGIN
    UPDATE produits
    SET prix = prix * (1 + p_pourcentage_augmentation / 100)
    WHERE categorie_id = p_categorie_id;
END//

DELIMITER ;

-- Utilisation
CALL sp_update_prix_categorie(5, 10);  -- +10% sur catégorie 5
```

### Alternative 3 : Accès direct aux tables

Tout simplement, contourner la vue et modifier directement la table :

```sql
-- Au lieu de modifier via la vue non-updatable
-- UPDATE v_stats_produits SET ...;

-- Modifier directement la table source
UPDATE produits
SET prix = prix * 1.1
WHERE categorie_id = 5;
```

### Alternative 4 : Vues empilées (vue updatable sur vue)

Si une vue complexe n'est pas updatable, créer une vue intermédiaire updatable :

```sql
-- Vue de base : updatable
CREATE VIEW v_base AS
SELECT id, nom, prix, categorie_id
FROM produits
WHERE actif = TRUE;

-- Vue complexe sur la vue de base : non-updatable
CREATE VIEW v_stats AS
SELECT
    categorie_id,
    COUNT(*) AS nb_produits
FROM v_base
GROUP BY categorie_id;

-- ❌ v_stats n'est pas updatable (GROUP BY)
-- ✅ Mais v_base l'est
UPDATE v_base SET prix = 100 WHERE id = 50;
```

---

## Cas d'usage pratiques

### Cas 1 : Vue de sécurité pour application

```sql
-- Table complète avec données sensibles
CREATE TABLE utilisateurs (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(255),
    password_hash VARCHAR(255),  -- Sensible
    salt VARCHAR(255),            -- Sensible
    role ENUM('USER', 'ADMIN'),
    created_at TIMESTAMP
) ENGINE=InnoDB;

-- Vue pour l'application : masque les données sensibles
CREATE VIEW v_utilisateurs_app AS
SELECT
    id,
    username,
    email,
    role,
    created_at
FROM utilisateurs;

-- ✅ L'application peut modifier via la vue
UPDATE v_utilisateurs_app
SET email = 'nouveau@email.com'
WHERE id = 100;

-- ❌ L'application ne peut PAS accéder au password_hash
UPDATE v_utilisateurs_app
SET password_hash = 'xxx'
WHERE id = 100;
-- Error: Unknown column 'password_hash'
```

### Cas 2 : Vue de filtrage multi-tenant

```sql
-- Table avec données de tous les tenants
CREATE TABLE documents (
    id INT PRIMARY KEY,
    titre VARCHAR(255),
    contenu TEXT,
    tenant_id INT,
    auteur_id INT
) ENGINE=InnoDB;

-- Vue pour un tenant spécifique (via variable de session)
CREATE VIEW v_mes_documents AS
SELECT
    id,
    titre,
    contenu,
    auteur_id
FROM documents
WHERE tenant_id = @current_tenant_id;

-- Configuration du tenant en session
SET @current_tenant_id = 42;

-- ✅ Modifications isolées par tenant
UPDATE v_mes_documents
SET titre = 'Nouveau titre'
WHERE id = 100;
-- Ne modifie que les documents du tenant 42

-- ❌ Ne peut pas modifier les documents d'autres tenants
UPDATE v_mes_documents
SET titre = 'Test'
WHERE id = 999;  -- Document du tenant 50
-- Résultat : 0 row affected (document invisible dans la vue)
```

### Cas 3 : Vue simplifiée pour formulaire

```sql
-- Table normalisée complexe
CREATE TABLE employes (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    telephone VARCHAR(20),
    date_embauche DATE,
    departement_id INT,
    manager_id INT,
    salaire DECIMAL(10,2),
    bonus_pct DECIMAL(5,2),
    -- ... 20 autres colonnes
) ENGINE=InnoDB;

-- Vue simple pour formulaire de modification profil
CREATE VIEW v_profil_employe AS
SELECT
    id,
    nom,
    prenom,
    email,
    telephone
FROM employes;

-- ✅ Le formulaire peut modifier uniquement ces 5 champs
UPDATE v_profil_employe
SET email = 'nouveau@email.com',
    telephone = '0612345678'
WHERE id = CURRENT_USER_ID;
```

---

## ⚠️ Pièges courants à éviter

### 1. Oublier que INSERT/UPDATE peut créer des lignes invisibles

```sql
-- Vue : produits chers (prix > 100)
CREATE VIEW v_produits_chers AS
SELECT id, nom, prix
FROM produits
WHERE prix > 100;

-- ❌ PIÈGE : INSERT crée une ligne invisible dans la vue
INSERT INTO v_produits_chers (nom, prix)
VALUES ('Produit pas cher', 50);
-- ✅ L'INSERT réussit !

-- Mais la ligne n'est pas dans la vue
SELECT * FROM v_produits_chers WHERE nom = 'Produit pas cher';
-- Résultat : 0 ligne

-- Elle existe dans la table
SELECT * FROM produits WHERE nom = 'Produit pas cher';
-- Résultat : Produit pas cher, 50
```

**Solution** : Toujours utiliser **WITH CHECK OPTION** (voir section 9.4) pour éviter ce comportement.

### 2. Confondre vue updatable et vue qui contient toutes les colonnes

```sql
-- ❌ Cette vue contient toutes les colonnes mais n'est PAS updatable
CREATE VIEW v_tous_champs AS
SELECT
    categorie_id,
    COUNT(*) AS nb_produits,
    SUM(prix) AS total_prix
FROM produits
GROUP BY categorie_id;

-- Toutes les colonnes sont présentes, mais...
UPDATE v_tous_champs SET total_prix = 1000 WHERE categorie_id = 5;
-- ❌ Error: The target table v_tous_champs of the UPDATE is not updatable
-- Raison : GROUP BY rend la vue non-updatable
```

**Important** : Ce n'est pas le **nombre de colonnes** qui détermine si une vue est updatable, mais la **complexité de la requête**.

### 3. Supposer qu'une vue avec LEFT JOIN est complètement updatable

```sql
-- Vue avec LEFT JOIN
CREATE VIEW v_clients_adresses AS
SELECT
    c.id,
    c.nom,
    a.ville
FROM clients c
LEFT JOIN adresses a ON c.id = a.client_id;

-- ✅ UPDATE sur colonnes de 'clients' fonctionne
UPDATE v_clients_adresses SET nom = 'Dupont' WHERE id = 10;

-- ❌ UPDATE sur colonnes de 'adresses' échoue
UPDATE v_clients_adresses SET ville = 'Paris' WHERE id = 10;
-- Error: Can not modify more than one base table through a join view
```

### 4. Oublier de vérifier IS_UPDATABLE

Toujours vérifier avant de supposer qu'une vue est updatable :

```sql
-- ❌ Code fragile : suppose que la vue est updatable
UPDATE v_ma_vue SET col = 'valeur' WHERE id = 10;
-- Si la vue n'est pas updatable, erreur à l'exécution !

-- ✅ Vérifier d'abord
SELECT IS_UPDATABLE
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_ma_vue';

-- Si YES, procéder aux modifications
```

### 5. Utiliser des vues updatable pour des modifications complexes

```sql
-- ❌ Mauvaise pratique : logique métier complexe via une vue
CREATE VIEW v_prix_avec_tva AS
SELECT
    id,
    nom,
    prix_ht,
    (prix_ht * 1.20) AS prix_ttc
FROM produits;

-- Vouloir modifier le prix TTC
UPDATE v_prix_avec_tva SET prix_ttc = 120 WHERE id = 10;
-- ❌ Impossible : prix_ttc est calculé

-- ✅ Solution : Procédure stockée avec logique explicite
DELIMITER //

CREATE PROCEDURE sp_update_prix_ttc(
    IN p_id INT,
    IN p_prix_ttc DECIMAL(10,2)
)
BEGIN
    UPDATE produits
    SET prix_ht = p_prix_ttc / 1.20
    WHERE id = p_id;
END//

DELIMITER ;

CALL sp_update_prix_ttc(10, 120);
```

---

## ✅ Points clés à retenir

- Une **vue updatable** permet INSERT, UPDATE et DELETE, pas seulement SELECT
- Les conditions strictes : **une seule table, pas d'agrégation, pas de DISTINCT, pas de UNION**
- Utiliser **INFORMATION_SCHEMA.VIEWS** pour vérifier si IS_UPDATABLE = 'YES'
- Les vues avec **jointures** sont partiellement updatable (table principale seulement)
- Les **colonnes calculées** ne sont jamais modifiables
- Sans **WITH CHECK OPTION**, INSERT/UPDATE peut créer des lignes invisibles dans la vue
- **ALGORITHM = TEMPTABLE** rend automatiquement une vue non-updatable
- Pour les modifications complexes, préférer les **procédures stockées** aux vues
- Les vues updatable sont utiles pour la **sécurité** (masquage de colonnes) et l'**abstraction**
- Toujours tester l'updatabilité avant de déployer une vue en production

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 INSERT ... SELECT](https://mariadb.com/kb/en/insert-select/) - INSERT via vue
- [📖 UPDATE](https://mariadb.com/kb/en/update/) - UPDATE avec vues
- [📖 DELETE](https://mariadb.com/kb/en/delete/) - DELETE via vue
- [📖 Updatable Views](https://mariadb.com/kb/en/inserting-and-updating-with-views/) - Documentation complète
- [📖 INFORMATION_SCHEMA.VIEWS](https://mariadb.com/kb/en/information-schema-views-table/) - Métadonnées des vues

### Articles complémentaires
- **"Understanding Updatable Views in MySQL/MariaDB"** - Percona Blog
- **"View Updatability Rules and Best Practices"** - Database Journal

---

## ➡️ Section suivante

**[9.4 WITH CHECK OPTION](./04-with-check-option.md)** : Découvrez comment utiliser WITH CHECK OPTION pour garantir que les INSERT et UPDATE via une vue respectent toujours la clause WHERE de la vue, évitant ainsi les lignes invisibles et les incohérences de données.

---


⏭️ [WITH CHECK OPTION](/09-vues-et-donnees-virtuelles/04-with-check-option.md)

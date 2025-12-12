🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 WITH CHECK OPTION

> **Niveau** : Intermédiaire
> **Durée estimée** : 1.5 heures
> **Prérequis** : Section 9.3 (Vues updatable), compréhension des contraintes d'intégrité

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le rôle de WITH CHECK OPTION pour garantir l'intégrité des données via les vues
- Utiliser WITH CHECK OPTION pour empêcher les INSERT/UPDATE qui violent la clause WHERE de la vue
- Distinguer les modes CASCADED et LOCAL et choisir le mode approprié
- Identifier les cas d'usage où WITH CHECK OPTION est indispensable
- Anticiper les erreurs liées aux vérifications de contraintes
- Implémenter des vues sécurisées avec validation automatique
- Combiner WITH CHECK OPTION avec d'autres mécanismes de sécurité

---

## Introduction

### Le problème : Lignes invisibles dans les vues

Comme nous l'avons vu dans la section 9.3, un INSERT ou UPDATE effectué via une vue updatable peut créer ou modifier des lignes qui **ne respectent pas** la clause WHERE de la vue, les rendant invisibles :

```sql
-- Vue : produits en stock (stock > 0)
CREATE VIEW v_produits_en_stock AS
SELECT id, nom, prix, stock
FROM produits
WHERE stock > 0;

-- ❌ PROBLÈME : INSERT d'un produit sans stock
INSERT INTO v_produits_en_stock (nom, prix, stock)
VALUES ('Produit test', 50, 0);
-- ✅ L'INSERT réussit !

-- Mais la ligne n'est pas visible dans la vue
SELECT * FROM v_produits_en_stock WHERE nom = 'Produit test';
-- Résultat : 0 ligne (stock = 0, donc WHERE stock > 0 est false)

-- La ligne existe dans la table sous-jacente
SELECT * FROM produits WHERE nom = 'Produit test';
-- Résultat : Produit test, 50, 0
```

Ce comportement peut causer des **incohérences de données** et des bugs difficiles à détecter.

### La solution : WITH CHECK OPTION

La clause **WITH CHECK OPTION** force MariaDB à vérifier que tout INSERT ou UPDATE via la vue respecte la clause WHERE de la vue. Si la condition n'est pas respectée, l'opération échoue avec une erreur.

```sql
-- Vue avec WITH CHECK OPTION
CREATE VIEW v_produits_en_stock AS
SELECT id, nom, prix, stock
FROM produits
WHERE stock > 0
WITH CHECK OPTION;

-- ❌ INSERT qui viole la contrainte échoue maintenant
INSERT INTO v_produits_en_stock (nom, prix, stock)
VALUES ('Produit test', 50, 0);
-- Error: CHECK OPTION failed 'database.v_produits_en_stock'

-- ✅ INSERT qui respecte la contrainte fonctionne
INSERT INTO v_produits_en_stock (nom, prix, stock)
VALUES ('Produit valide', 50, 10);
-- Succès : stock = 10 > 0
```

### Avantages de WITH CHECK OPTION

- ✅ **Intégrité des données** : Garantit que les données respectent les règles métier
- ✅ **Validation automatique** : Pas besoin de vérifier manuellement dans l'application
- ✅ **Prévention des bugs** : Empêche les modifications incohérentes
- ✅ **Sécurité** : Renforce les règles d'accès aux données
- ✅ **Documentation** : La contrainte est explicite dans la définition de la vue

---

## Syntaxe de WITH CHECK OPTION

### Syntaxe complète

```sql
CREATE VIEW view_name AS
SELECT ...
FROM ...
WHERE ...
WITH [CASCADED | LOCAL] CHECK OPTION;
```

### Les deux modes

MariaDB propose deux modes pour WITH CHECK OPTION :

1. **CASCADED** (par défaut) : Vérifie les contraintes de cette vue ET de toutes les vues sous-jacentes
2. **LOCAL** : Vérifie uniquement la contrainte de cette vue (pas les vues sous-jacentes)

```sql
-- Équivalent (CASCADED par défaut)
CREATE VIEW v1 AS ... WITH CHECK OPTION;
CREATE VIEW v1 AS ... WITH CASCADED CHECK OPTION;

-- Mode LOCAL explicite
CREATE VIEW v2 AS ... WITH LOCAL CHECK OPTION;
```

---

## Mode CASCADED (par défaut)

### Comportement

Le mode **CASCADED** vérifie :
- La clause WHERE de la vue courante
- Les clauses WHERE de **toutes les vues parentes** (si la vue est basée sur d'autres vues)

### Exemple simple

```sql
-- Vue avec CHECK OPTION CASCADED
CREATE VIEW v_employes_actifs AS
SELECT id, nom, prenom, email, statut, salaire
FROM employes
WHERE statut = 'ACTIF'
WITH CASCADED CHECK OPTION;

-- ✅ INSERT valide : statut = 'ACTIF'
INSERT INTO v_employes_actifs (nom, prenom, email, statut, salaire)
VALUES ('Dupont', 'Marie', 'marie@example.com', 'ACTIF', 45000);

-- ❌ INSERT invalide : statut = 'INACTIF'
INSERT INTO v_employes_actifs (nom, prenom, email, statut, salaire)
VALUES ('Martin', 'Paul', 'paul@example.com', 'INACTIF', 40000);
-- Error: CHECK OPTION failed 'database.v_employes_actifs'

-- ❌ UPDATE qui viole la contrainte échoue
UPDATE v_employes_actifs
SET statut = 'INACTIF'
WHERE id = 1;
-- Error: CHECK OPTION failed 'database.v_employes_actifs'
```

### Vues imbriquées avec CASCADED

Le mode CASCADED propage la vérification aux vues parentes :

```sql
-- Vue de base : tous les employes
CREATE TABLE employes (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    departement VARCHAR(50),
    salaire DECIMAL(10,2),
    statut ENUM('ACTIF', 'INACTIF')
) ENGINE=InnoDB;

-- Vue niveau 1 : employés actifs (sans CHECK OPTION)
CREATE VIEW v_employes_actifs AS
SELECT id, nom, departement, salaire, statut
FROM employes
WHERE statut = 'ACTIF';

-- Vue niveau 2 : employés IT avec CHECK OPTION CASCADED
CREATE VIEW v_employes_it AS
SELECT id, nom, departement, salaire, statut
FROM v_employes_actifs
WHERE departement = 'IT'
WITH CASCADED CHECK OPTION;

-- ✅ INSERT valide : statut='ACTIF' ET departement='IT'
INSERT INTO v_employes_it (nom, departement, salaire, statut)
VALUES ('Dubois', 'IT', 50000, 'ACTIF');

-- ❌ INSERT invalide : departement != 'IT'
INSERT INTO v_employes_it (nom, departement, salaire, statut)
VALUES ('Lambert', 'RH', 45000, 'ACTIF');
-- Error: CHECK OPTION failed 'database.v_employes_it'

-- ❌ INSERT invalide : statut != 'ACTIF' (contrainte de la vue parente)
INSERT INTO v_employes_it (nom, departement, salaire, statut)
VALUES ('Moreau', 'IT', 48000, 'INACTIF');
-- Error: CHECK OPTION failed 'database.v_employes_it'
-- CASCADED vérifie AUSSI la contrainte de v_employes_actifs !
```

**Explication** : Avec CASCADED, même si `v_employes_actifs` n'a pas WITH CHECK OPTION, la contrainte `statut = 'ACTIF'` est quand même vérifiée car `v_employes_it` utilise CASCADED.

---

## Mode LOCAL

### Comportement

Le mode **LOCAL** vérifie uniquement :
- La clause WHERE de la vue courante
- **Ignore** les contraintes des vues parentes (sauf si elles ont aussi WITH CHECK OPTION)

### Exemple avec vues imbriquées

```sql
-- Vue niveau 1 : employés actifs (SANS CHECK OPTION)
CREATE OR REPLACE VIEW v_employes_actifs AS
SELECT id, nom, departement, salaire, statut
FROM employes
WHERE statut = 'ACTIF';

-- Vue niveau 2 : employés IT avec CHECK OPTION LOCAL
CREATE OR REPLACE VIEW v_employes_it AS
SELECT id, nom, departement, salaire, statut
FROM v_employes_actifs
WHERE departement = 'IT'
WITH LOCAL CHECK OPTION;

-- ✅ INSERT valide : departement='IT' (contrainte de v_employes_it)
INSERT INTO v_employes_it (nom, departement, salaire, statut)
VALUES ('Dubois', 'IT', 50000, 'ACTIF');

-- ❌ INSERT invalide : departement != 'IT'
INSERT INTO v_employes_it (nom, departement, salaire, statut)
VALUES ('Lambert', 'RH', 45000, 'ACTIF');
-- Error: CHECK OPTION failed 'database.v_employes_it'

-- ⚠️ INSERT "invalide" mais réussit avec LOCAL !
INSERT INTO v_employes_it (nom, departement, salaire, statut)
VALUES ('Moreau', 'IT', 48000, 'INACTIF');
-- ✅ Succès avec LOCAL (ne vérifie pas statut='ACTIF' de la vue parente)
-- ❌ Aurait échoué avec CASCADED

-- Vérification : la ligne n'est pas visible dans v_employes_actifs
SELECT * FROM v_employes_actifs WHERE nom = 'Moreau';
-- Résultat : 0 ligne (statut='INACTIF')

-- Mais elle n'est pas non plus visible dans v_employes_it !
SELECT * FROM v_employes_it WHERE nom = 'Moreau';
-- Résultat : 0 ligne (car v_employes_it est basé sur v_employes_actifs)
```

**Problème avec LOCAL** : L'INSERT réussit mais crée une ligne **invisible** dans les deux vues ! C'est exactement le problème que WITH CHECK OPTION devait résoudre.

### Quand utiliser LOCAL ?

Le mode LOCAL est rarement recommandé car il peut créer des incohérences. Utilisez-le uniquement si :
- Vous contrôlez totalement la hiérarchie des vues
- Vous voulez délibérément ne vérifier que la contrainte de la vue courante
- Les vues parentes ont déjà leur propre WITH CHECK OPTION

---

## Comparaison CASCADED vs LOCAL

### Tableau récapitulatif

| Aspect | CASCADED | LOCAL |
|--------|----------|-------|
| **Vérifie la vue courante** | ✅ Oui | ✅ Oui |
| **Vérifie les vues parentes** | ✅ Toujours | ⚠️ Seulement si elles ont CHECK OPTION |
| **Par défaut** | ✅ Oui | ❌ Non |
| **Sécurité** | ✅ Maximale | ⚠️ Partielle |
| **Risque lignes invisibles** | ❌ Aucun | ⚠️ Possible |
| **Cas d'usage** | Standard, recommandé | Cas spécifiques |

### Exemple comparatif détaillé

```sql
-- Configuration : 3 niveaux de vues
CREATE TABLE produits (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    stock INT,
    categorie VARCHAR(50)
) ENGINE=InnoDB;

-- Vue niveau 1 : produits en stock (SANS CHECK OPTION)
CREATE VIEW v_produits_stock AS
SELECT id, nom, prix, stock, categorie
FROM produits
WHERE stock > 0;

-- Vue niveau 2A : produits de la catégorie Électronique (CASCADED)
CREATE VIEW v_electronique_cascaded AS
SELECT id, nom, prix, stock, categorie
FROM v_produits_stock
WHERE categorie = 'Électronique'
WITH CASCADED CHECK OPTION;

-- Vue niveau 2B : produits de la catégorie Électronique (LOCAL)
CREATE VIEW v_electronique_local AS
SELECT id, nom, prix, stock, categorie
FROM v_produits_stock
WHERE categorie = 'Électronique'
WITH LOCAL CHECK OPTION;

-- Test 1 : INSERT avec stock=0
-- CASCADED : ❌ Échoue (vérifie stock > 0 de la vue parente)
INSERT INTO v_electronique_cascaded (nom, prix, stock, categorie)
VALUES ('Produit A', 100, 0, 'Électronique');
-- Error: CHECK OPTION failed

-- LOCAL : ✅ Réussit (ne vérifie pas stock > 0 de la vue parente)
INSERT INTO v_electronique_local (nom, prix, stock, categorie)
VALUES ('Produit B', 100, 0, 'Électronique');
-- Succès mais ligne invisible dans les vues !

-- Test 2 : INSERT avec mauvaise catégorie
-- CASCADED : ❌ Échoue
INSERT INTO v_electronique_cascaded (nom, prix, stock, categorie)
VALUES ('Produit C', 100, 10, 'Jouet');
-- Error: CHECK OPTION failed

-- LOCAL : ❌ Échoue aussi
INSERT INTO v_electronique_local (nom, prix, stock, categorie)
VALUES ('Produit D', 100, 10, 'Jouet');
-- Error: CHECK OPTION failed
-- (Vérifie sa propre contrainte : categorie = 'Électronique')
```

### Recommandation

💡 **Utilisez toujours CASCADED (par défaut)** sauf si vous avez une raison spécifique d'utiliser LOCAL. CASCADED offre une sécurité maximale en vérifiant toute la hiérarchie de vues.

---

## Cas d'usage pratiques

### Cas 1 : Validation de statut

**Besoin** : Empêcher la création de commandes avec un statut invalide.

```sql
-- Table commandes
CREATE TABLE commandes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT,
    date_commande DATE,
    montant DECIMAL(10,2),
    statut ENUM('EN_ATTENTE', 'PAYEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE')
) ENGINE=InnoDB;

-- Vue : commandes actives uniquement (pas annulées)
CREATE VIEW v_commandes_actives AS
SELECT id, client_id, date_commande, montant, statut
FROM commandes
WHERE statut != 'ANNULEE'
WITH CHECK OPTION;

-- ✅ INSERT avec statut valide
INSERT INTO v_commandes_actives (client_id, date_commande, montant, statut)
VALUES (100, CURDATE(), 250.00, 'EN_ATTENTE');

-- ❌ INSERT avec statut ANNULEE échoue
INSERT INTO v_commandes_actives (client_id, date_commande, montant, statut)
VALUES (101, CURDATE(), 150.00, 'ANNULEE');
-- Error: CHECK OPTION failed

-- ❌ UPDATE vers ANNULEE échoue
UPDATE v_commandes_actives SET statut = 'ANNULEE' WHERE id = 1;
-- Error: CHECK OPTION failed

-- Pour annuler une commande, il faut passer par la table directement
UPDATE commandes SET statut = 'ANNULEE' WHERE id = 1;
```

**Avantage** : L'application ne peut pas créer ou modifier des commandes annulées via cette vue, garantissant que seules les commandes actives sont manipulées.

### Cas 2 : Isolation multi-tenant

**Besoin** : Chaque tenant ne peut modifier que ses propres données.

```sql
-- Table documents multi-tenant
CREATE TABLE documents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    titre VARCHAR(255),
    contenu TEXT,
    tenant_id INT,
    auteur_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Vue pour le tenant 42
CREATE VIEW v_documents_tenant42 AS
SELECT id, titre, contenu, auteur_id, created_at
FROM documents
WHERE tenant_id = 42
WITH CHECK OPTION;

-- Grant uniquement sur la vue
GRANT SELECT, INSERT, UPDATE, DELETE ON v_documents_tenant42 TO 'app_tenant42'@'%';

-- ✅ INSERT valide (tenant_id implicitement 42)
INSERT INTO v_documents_tenant42 (titre, contenu, auteur_id)
VALUES ('Mon document', 'Contenu...', 100);
-- ⚠️ Mais tenant_id n'est pas dans la vue !
-- Solution ci-dessous...
```

**Problème** : La colonne `tenant_id` n'est pas dans la vue, donc comment garantir qu'elle vaut 42 ?

**Solution** : Utiliser une colonne générée ou un trigger :

```sql
-- Option 1 : Ajouter tenant_id avec valeur par défaut
ALTER TABLE documents
    MODIFY tenant_id INT NOT NULL DEFAULT 42;

-- Option 2 : Trigger BEFORE INSERT
DELIMITER //

CREATE TRIGGER trg_documents_tenant42_insert
BEFORE INSERT ON documents
FOR EACH ROW
BEGIN
    IF NEW.tenant_id IS NULL OR NEW.tenant_id != 42 THEN
        SET NEW.tenant_id = 42;
    END IF;
END//

DELIMITER ;

-- Maintenant la vue avec CHECK OPTION est complète
CREATE OR REPLACE VIEW v_documents_tenant42 AS
SELECT id, titre, contenu, tenant_id, auteur_id, created_at
FROM documents
WHERE tenant_id = 42
WITH CHECK OPTION;

-- ✅ INSERT valide
INSERT INTO v_documents_tenant42 (titre, contenu, tenant_id, auteur_id)
VALUES ('Document', 'Contenu...', 42, 100);

-- ❌ INSERT avec mauvais tenant_id échoue
INSERT INTO v_documents_tenant42 (titre, contenu, tenant_id, auteur_id)
VALUES ('Piratage', 'Contenu...', 99, 100);
-- Error: CHECK OPTION failed
```

### Cas 3 : Validation de plages de valeurs

**Besoin** : Garantir que les prix sont dans une plage acceptable.

```sql
-- Table produits
CREATE TABLE produits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    categorie VARCHAR(50)
) ENGINE=InnoDB;

-- Vue : produits avec prix raisonnable (entre 10 et 1000)
CREATE VIEW v_produits_prix_valides AS
SELECT id, nom, prix, categorie
FROM produits
WHERE prix BETWEEN 10 AND 1000
WITH CHECK OPTION;

-- ✅ INSERT avec prix valide
INSERT INTO v_produits_prix_valides (nom, prix, categorie)
VALUES ('Produit normal', 50.00, 'Général');

-- ❌ INSERT avec prix trop bas
INSERT INTO v_produits_prix_valides (nom, prix, categorie)
VALUES ('Gratuit', 0.00, 'Général');
-- Error: CHECK OPTION failed

-- ❌ INSERT avec prix trop élevé
INSERT INTO v_produits_prix_valides (nom, prix, categorie)
VALUES ('Très cher', 5000.00, 'Luxe');
-- Error: CHECK OPTION failed

-- ❌ UPDATE vers prix invalide échoue
UPDATE v_produits_prix_valides SET prix = 0.01 WHERE id = 1;
-- Error: CHECK OPTION failed
```

### Cas 4 : Règles métier complexes

**Besoin** : Valider plusieurs conditions simultanément.

```sql
-- Table employés
CREATE TABLE employes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    age INT,
    salaire DECIMAL(10,2),
    departement VARCHAR(50),
    statut ENUM('STAGIAIRE', 'CDI', 'CDD', 'FREELANCE')
) ENGINE=InnoDB;

-- Vue : employés CDI avec règles métier
CREATE VIEW v_employes_cdi_valides AS
SELECT id, nom, age, salaire, departement, statut
FROM employes
WHERE statut = 'CDI'
  AND age >= 18                    -- Majeur
  AND salaire >= 1800              -- SMIC (exemple)
  AND departement IS NOT NULL
WITH CHECK OPTION;

-- ✅ INSERT valide : toutes les conditions respectées
INSERT INTO v_employes_cdi_valides (nom, age, salaire, departement, statut)
VALUES ('Dupont', 25, 35000, 'IT', 'CDI');

-- ❌ INSERT invalide : âge < 18
INSERT INTO v_employes_cdi_valides (nom, age, salaire, departement, statut)
VALUES ('Jeune', 16, 2000, 'IT', 'CDI');
-- Error: CHECK OPTION failed

-- ❌ INSERT invalide : salaire trop bas
INSERT INTO v_employes_cdi_valides (nom, age, salaire, departement, statut)
VALUES ('Sous-payé', 25, 1500, 'IT', 'CDI');
-- Error: CHECK OPTION failed

-- ❌ INSERT invalide : statut != CDI
INSERT INTO v_employes_cdi_valides (nom, age, salaire, departement, statut)
VALUES ('Stagiaire', 22, 2000, 'IT', 'STAGIAIRE');
-- Error: CHECK OPTION failed
```

---

## Limitations et contraintes

### Colonnes non présentes dans la vue

⚠️ **Problème** : WITH CHECK OPTION ne peut vérifier que les colonnes présentes dans la vue.

```sql
-- Table avec colonne sensible
CREATE TABLE utilisateurs (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(255),
    role ENUM('USER', 'ADMIN'),
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- Vue sans la colonne role
CREATE VIEW v_utilisateurs_actifs AS
SELECT id, username, email, is_active
FROM utilisateurs
WHERE is_active = TRUE
WITH CHECK OPTION;

-- ✅ INSERT sans spécifier role (prend la valeur par défaut)
INSERT INTO v_utilisateurs_actifs (username, email, is_active)
VALUES ('user1', 'user1@example.com', TRUE);
-- role sera NULL ou DEFAULT

-- ⚠️ On ne peut pas contrôler role via cette vue !
-- Pour contrôler role, il faut l'inclure dans la vue
CREATE OR REPLACE VIEW v_utilisateurs_actifs AS
SELECT id, username, email, is_active, role
FROM utilisateurs
WHERE is_active = TRUE
  AND role = 'USER'  -- Contrainte supplémentaire
WITH CHECK OPTION;
```

### Vues avec jointures

WITH CHECK OPTION est limité avec les vues contenant des jointures :

```sql
-- Vue avec jointure
CREATE VIEW v_commandes_clients AS
SELECT
    c.id AS commande_id,
    c.date_commande,
    c.montant,
    cl.id AS client_id,
    cl.nom AS client_nom
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id
WHERE c.statut != 'ANNULEE'
WITH CHECK OPTION;

-- ✅ UPDATE sur table principale (commandes)
UPDATE v_commandes_clients
SET montant = 500
WHERE commande_id = 10;

-- ❌ UPDATE vers statut ANNULEE échoue (si statut dans la vue)
-- Mais si statut n'est pas dans la vue, WITH CHECK OPTION ne peut pas vérifier !
```

💡 **Conseil** : Pour les vues avec jointures, WITH CHECK OPTION vérifie principalement les contraintes de la table principale. Privilégiez les vues sur table unique pour WITH CHECK OPTION.

### Performance

WITH CHECK OPTION ajoute une **vérification supplémentaire** à chaque INSERT/UPDATE :

```sql
-- Sans CHECK OPTION
INSERT INTO v_produits ...;
-- 1. Insérer dans la table
-- Temps : T

-- Avec CHECK OPTION
INSERT INTO v_produits ...;
-- 1. Insérer dans la table
-- 2. Vérifier la contrainte WHERE de la vue
-- Temps : T + Δ (surcoût généralement négligeable)
```

**Impact** : Le surcoût est généralement **négligeable** (< 1%) car il s'agit simplement d'évaluer la clause WHERE. Cependant, pour des INSERT/UPDATE massifs (millions de lignes), cela peut s'accumuler.

💡 **Recommandation** : Le gain en intégrité des données compense largement le léger surcoût de performance.

---

## Vérifier si une vue a CHECK OPTION

### Utiliser INFORMATION_SCHEMA.VIEWS

```sql
-- Vérifier CHECK OPTION d'une vue
SELECT
    TABLE_NAME,
    CHECK_OPTION,
    IS_UPDATABLE
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'v_produits_en_stock';

-- Résultat :
-- TABLE_NAME            | CHECK_OPTION | IS_UPDATABLE
-- v_produits_en_stock   | CASCADED     | YES
```

**Valeurs possibles de CHECK_OPTION** :
- **NONE** : Pas de WITH CHECK OPTION
- **LOCAL** : WITH LOCAL CHECK OPTION
- **CASCADED** : WITH CASCADED CHECK OPTION (ou WITH CHECK OPTION sans précision)

### Lister toutes les vues avec CHECK OPTION

```sql
-- Trouver toutes les vues avec CHECK OPTION
SELECT
    TABLE_NAME,
    CHECK_OPTION,
    DEFINER
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND CHECK_OPTION != 'NONE'
ORDER BY TABLE_NAME;
```

### SHOW CREATE VIEW

```sql
SHOW CREATE VIEW v_produits_en_stock\G

-- Résultat :
-- *************************** 1. row ***************************
--                 View: v_produits_en_stock
--          Create View: CREATE ALGORITHM=UNDEFINED
--                       DEFINER=`root`@`localhost`
--                       SQL SECURITY DEFINER
--                       VIEW `v_produits_en_stock` AS
--                       select `produits`.`id` AS `id`,
--                              `produits`.`nom` AS `nom`,
--                              `produits`.`prix` AS `prix`,
--                              `produits`.`stock` AS `stock`
--                       from `produits`
--                       where `produits`.`stock` > 0
--                       WITH CASCADED CHECK OPTION
```

---

## Modifier une vue pour ajouter/retirer CHECK OPTION

### Ajouter CHECK OPTION à une vue existante

```sql
-- Vue sans CHECK OPTION
CREATE VIEW v_employes AS
SELECT id, nom, departement
FROM employes
WHERE departement = 'IT';

-- Ajouter CHECK OPTION avec ALTER VIEW
ALTER VIEW v_employes AS
SELECT id, nom, departement
FROM employes
WHERE departement = 'IT'
WITH CHECK OPTION;

-- Ou avec CREATE OR REPLACE VIEW
CREATE OR REPLACE VIEW v_employes AS
SELECT id, nom, departement
FROM employes
WHERE departement = 'IT'
WITH CHECK OPTION;
```

### Retirer CHECK OPTION

```sql
-- Vue avec CHECK OPTION
CREATE VIEW v_produits AS
SELECT id, nom, prix
FROM produits
WHERE prix > 0
WITH CHECK OPTION;

-- Retirer CHECK OPTION (recréer sans la clause)
CREATE OR REPLACE VIEW v_produits AS
SELECT id, nom, prix
FROM produits
WHERE prix > 0;
-- Pas de WITH CHECK OPTION = CHECK_OPTION = NONE
```

### Changer de CASCADED à LOCAL (ou vice-versa)

```sql
-- Vue avec CASCADED
CREATE VIEW v_data AS
SELECT * FROM table WHERE condition
WITH CASCADED CHECK OPTION;

-- Changer en LOCAL
CREATE OR REPLACE VIEW v_data AS
SELECT * FROM table WHERE condition
WITH LOCAL CHECK OPTION;
```

---

## Combiner WITH CHECK OPTION et autres contraintes

### WITH CHECK OPTION + Contraintes de table

Les deux mécanismes sont complémentaires :

```sql
-- Table avec contrainte CHECK (MariaDB 10.2.1+)
CREATE TABLE produits (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    stock INT,
    CONSTRAINT chk_prix_positif CHECK (prix > 0),
    CONSTRAINT chk_stock_positif CHECK (stock >= 0)
) ENGINE=InnoDB;

-- Vue avec WITH CHECK OPTION supplémentaire
CREATE VIEW v_produits_disponibles AS
SELECT id, nom, prix, stock
FROM produits
WHERE stock > 10  -- Contrainte supplémentaire
WITH CHECK OPTION;

-- Vérifications cumulées :
-- 1. Contrainte table : prix > 0 (toujours vérifiée)
-- 2. Contrainte table : stock >= 0 (toujours vérifiée)
-- 3. Contrainte vue : stock > 10 (vérifiée via CHECK OPTION)

-- ❌ Échoue : prix <= 0 (contrainte table)
INSERT INTO v_produits_disponibles (nom, prix, stock)
VALUES ('Test', -10, 15);
-- Error: Check constraint 'chk_prix_positif' is violated

-- ❌ Échoue : stock <= 10 (contrainte vue)
INSERT INTO v_produits_disponibles (nom, prix, stock)
VALUES ('Test', 50, 5);
-- Error: CHECK OPTION failed
```

### WITH CHECK OPTION + Triggers

```sql
-- Table
CREATE TABLE commandes (
    id INT PRIMARY KEY,
    client_id INT,
    montant DECIMAL(10,2),
    statut VARCHAR(20)
) ENGINE=InnoDB;

-- Trigger BEFORE INSERT
DELIMITER //

CREATE TRIGGER trg_commandes_before_insert
BEFORE INSERT ON commandes
FOR EACH ROW
BEGIN
    -- Validation métier dans le trigger
    IF NEW.montant <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Le montant doit être positif';
    END IF;
END//

DELIMITER ;

-- Vue avec WITH CHECK OPTION
CREATE VIEW v_commandes_en_cours AS
SELECT id, client_id, montant, statut
FROM commandes
WHERE statut = 'EN_COURS'
WITH CHECK OPTION;

-- Vérifications cumulées :
-- 1. Trigger : montant > 0
-- 2. WITH CHECK OPTION : statut = 'EN_COURS'

-- ❌ Échoue : montant <= 0 (trigger)
INSERT INTO v_commandes_en_cours (client_id, montant, statut)
VALUES (100, -50, 'EN_COURS');
-- Error: Le montant doit être positif

-- ❌ Échoue : statut != 'EN_COURS' (CHECK OPTION)
INSERT INTO v_commandes_en_cours (client_id, montant, statut)
VALUES (100, 150, 'LIVREE');
-- Error: CHECK OPTION failed
```

---

## ⚠️ Pièges courants à éviter

### 1. Oublier d'inclure les colonnes de contrainte dans la vue

```sql
-- ❌ Piège : colonne de contrainte non exposée
CREATE VIEW v_employes_it AS
SELECT id, nom, email  -- departement n'est PAS dans la vue
FROM employes
WHERE departement = 'IT'
WITH CHECK OPTION;

-- INSERT sans spécifier departement
INSERT INTO v_employes_it (nom, email)
VALUES ('Test', 'test@test.com');
-- ⚠️ Comportement dépend du DEFAULT de la colonne departement

-- Si departement a un DEFAULT, l'INSERT peut réussir
-- Mais la ligne peut être invisible si DEFAULT != 'IT'

-- ✅ Solution : Inclure departement dans la vue
CREATE OR REPLACE VIEW v_employes_it AS
SELECT id, nom, email, departement
FROM employes
WHERE departement = 'IT'
WITH CHECK OPTION;
```

### 2. Utiliser LOCAL au lieu de CASCADED par erreur

```sql
-- ❌ Erreur : utiliser LOCAL sans comprendre les implications
CREATE VIEW v_produits_stock AS
SELECT id, nom, stock
FROM produits
WHERE stock > 0;

CREATE VIEW v_produits_chers AS
SELECT id, nom, stock, prix
FROM v_produits_stock
WHERE prix > 100
WITH LOCAL CHECK OPTION;  -- ⚠️ LOCAL = dangereux

-- INSERT avec stock=0 réussit (LOCAL n'a pas vérifié la vue parente)
INSERT INTO v_produits_chers (nom, stock, prix)
VALUES ('Test', 0, 150);
-- Ligne créée mais invisible !

-- ✅ Solution : Toujours utiliser CASCADED (ou rien, c'est le défaut)
CREATE OR REPLACE VIEW v_produits_chers AS
SELECT id, nom, stock, prix
FROM v_produits_stock
WHERE prix > 100
WITH CHECK OPTION;  -- Implicitement CASCADED
```

### 3. Croire que CHECK OPTION empêche toutes les violations

```sql
-- ❌ Idée fausse : CHECK OPTION protège contre tout
CREATE VIEW v_produits AS
SELECT id, nom, prix
FROM produits
WHERE prix > 0
WITH CHECK OPTION;

-- ✅ CHECK OPTION empêche INSERT/UPDATE via la vue
INSERT INTO v_produits (nom, prix) VALUES ('Test', -10);
-- Error: CHECK OPTION failed

-- ⚠️ Mais on peut toujours modifier directement la table !
INSERT INTO produits (nom, prix) VALUES ('Test', -10);
-- ✅ Réussit (contourne la vue)

-- ✅ Solution : Ajouter une vraie contrainte CHECK sur la table
ALTER TABLE produits ADD CONSTRAINT chk_prix CHECK (prix > 0);
```

### 4. Confusion entre CHECK OPTION et contraintes CHECK

```sql
-- Ce sont deux mécanismes différents !

-- WITH CHECK OPTION : Vérifie la clause WHERE de la vue
CREATE VIEW v1 AS SELECT * FROM t WHERE col > 0 WITH CHECK OPTION;

-- Contrainte CHECK : Vérifie une condition sur la table
ALTER TABLE t ADD CONSTRAINT chk1 CHECK (col > 0);

-- WITH CHECK OPTION :
-- - Appliquée uniquement via la vue
-- - Peut être contournée en accédant directement à la table
-- - Vérifie des conditions de la vue, pas forcément de la table

-- Contrainte CHECK :
-- - Appliquée toujours, quel que soit le moyen d'accès
-- - Ne peut pas être contournée
-- - Garantit l'intégrité au niveau table
```

### 5. Négliger la performance sur les INSERT massifs

```sql
-- ⚠️ Pour des INSERT massifs via une vue avec CHECK OPTION
-- Le surcoût de validation peut s'accumuler

-- INSERT de 1 million de lignes via la vue
INSERT INTO v_produits_valides
SELECT nom, prix, stock FROM table_source;
-- Chaque ligne vérifie WHERE prix > 0

-- ✅ Si performance critique, envisager :
-- 1. Désactiver temporairement CHECK OPTION (recréer la vue)
-- 2. Insérer directement dans la table
-- 3. Valider en batch avant insertion
```

---

## ✅ Points clés à retenir

- **WITH CHECK OPTION** empêche les INSERT/UPDATE qui violent la clause WHERE de la vue
- **CASCADED** (défaut) : Vérifie la vue courante ET toutes les vues parentes
- **LOCAL** : Vérifie uniquement la vue courante (risque de lignes invisibles)
- Toujours préférer **CASCADED** sauf cas spécifique nécessitant LOCAL
- WITH CHECK OPTION ne protège que les modifications **via la vue**, pas les accès directs à la table
- Inclure toujours les **colonnes de contrainte** dans la définition de la vue
- Combiner WITH CHECK OPTION avec des **contraintes CHECK** sur les tables pour une sécurité maximale
- Vérifier CHECK_OPTION avec **INFORMATION_SCHEMA.VIEWS**
- Le surcoût de performance est généralement **négligeable** (< 1%)
- WITH CHECK OPTION est essentiel pour les vues de **sécurité** et **validation métier**

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CREATE VIEW - WITH CHECK OPTION](https://mariadb.com/kb/en/create-view/#with-check-option) - Documentation complète
- [📖 CHECK OPTION Clause](https://mariadb.com/kb/en/view-algorithms/#with-check-option) - Détails techniques
- [📖 INFORMATION_SCHEMA.VIEWS](https://mariadb.com/kb/en/information-schema-views-table/) - Colonne CHECK_OPTION

### Articles complémentaires
- **"Understanding WITH CHECK OPTION in MySQL Views"** - Percona Blog
- **"View Constraints and Data Integrity"** - Database Journal
- **"CASCADED vs LOCAL CHECK OPTION"** - StackOverflow discussions

---

## ➡️ Section suivante

**[9.5 Sécurité et vues : Masquage de données](./05-securite-et-vues.md)** : Découvrez comment utiliser les vues comme mécanisme de sécurité pour masquer des colonnes sensibles, filtrer les lignes par utilisateur ou tenant, et combiner vues, privilèges et WITH CHECK OPTION pour une architecture de sécurité robuste.

---


⏭️ [Sécurité et vues : Masquage de données](/09-vues-et-donnees-virtuelles/05-securite-et-vues.md)

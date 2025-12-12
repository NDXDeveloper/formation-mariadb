🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.8 Mise à jour et suppression de données (UPDATE, DELETE, TRUNCATE)

> **Niveau** : Débutant
> **Durée estimée** : 2 heures
> **Prérequis** : Section 2.7 (Requêtes de sélection simples)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Mettre à jour des données avec UPDATE de manière sécurisée
- Modifier plusieurs colonnes simultanément
- Utiliser UPDATE avec calculs et sous-requêtes
- Supprimer des lignes avec DELETE en évitant les erreurs
- Comprendre la différence entre DELETE et TRUNCATE
- Gérer les contraintes de clés étrangères lors des suppressions
- Utiliser les transactions pour sécuriser les modifications
- Appliquer les bonnes pratiques pour éviter la perte de données

---

## Introduction

Les commandes **UPDATE** et **DELETE** permettent de modifier et supprimer des données existantes. Ces opérations sont **irréversibles** sans sauvegarde ou transaction, il est donc crucial de les utiliser avec précaution.

⚠️ **AVERTISSEMENT** : Une erreur dans une clause WHERE peut mettre à jour ou supprimer toutes les lignes d'une table. Toujours vérifier avec SELECT avant d'exécuter UPDATE ou DELETE.

### Règle d'or : SELECT avant UPDATE/DELETE

```sql
-- 1. Vérifier d'abord avec SELECT
SELECT * FROM clients WHERE client_id = 5;

-- 2. Si le résultat est correct, remplacer SELECT par UPDATE/DELETE
UPDATE clients SET email = 'nouveau@example.com' WHERE client_id = 5;
```

---

## Données de démonstration

Réutilisons notre base de librairie et ajoutons des tables pour les exemples :

```sql
USE librairie;

-- Table clients
CREATE TABLE clients (
    client_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    telephone VARCHAR(20),
    ville VARCHAR(100),
    points_fidelite INT DEFAULT 0,
    date_inscription DATETIME DEFAULT CURRENT_TIMESTAMP,
    actif BOOLEAN DEFAULT TRUE
);

-- Table commandes
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    date_commande DATETIME DEFAULT CURRENT_TIMESTAMP,
    statut ENUM('en_attente', 'confirmee', 'expediee', 'livree', 'annulee') DEFAULT 'en_attente',
    montant_total DECIMAL(10,2) DEFAULT 0,
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
);

-- Insertion de données de test
INSERT INTO clients (nom, email, telephone, ville, points_fidelite) VALUES
    ('Alice Martin', 'alice@example.com', '0612345678', 'Paris', 150),
    ('Bob Dupont', 'bob@example.com', '0698765432', 'Lyon', 75),
    ('Charlie Durand', 'charlie@example.com', '0623456789', 'Marseille', 0),
    ('David Leblanc', 'david@example.com', '0634567890', 'Toulouse', 200),
    ('Emma Petit', 'emma@example.com', '0645678901', 'Nice', 50);

INSERT INTO commandes (client_id, statut, montant_total) VALUES
    (1, 'livree', 89.99),
    (1, 'en_attente', 45.50),
    (2, 'confirmee', 120.00),
    (3, 'livree', 34.99),
    (4, 'expediee', 199.99);
```

---

## UPDATE - Mise à jour de données

### Syntaxe de base

```sql
UPDATE nom_table
SET colonne1 = valeur1, colonne2 = valeur2, ...
WHERE condition;
```

⚠️ **CRITIQUE** : La clause WHERE est essentielle ! Sans elle, TOUTES les lignes sont modifiées.

### UPDATE simple - Une colonne, une ligne

```sql
-- 1. Vérifier d'abord la ligne à modifier
SELECT * FROM clients WHERE client_id = 1;

-- 2. Mettre à jour l'email du client 1
UPDATE clients
SET email = 'alice.martin@example.com'
WHERE client_id = 1;

-- Query OK, 1 row affected

-- 3. Vérifier la modification
SELECT * FROM clients WHERE client_id = 1;
-- email changé : 'alice.martin@example.com'
```

### UPDATE plusieurs colonnes

```sql
-- Modifier plusieurs colonnes en une seule commande
UPDATE clients
SET
    telephone = '0611111111',
    ville = 'Paris 15e',
    points_fidelite = 175
WHERE client_id = 1;

-- Query OK, 1 row affected (même si une seule colonne change réellement)

-- Vérification
SELECT client_id, telephone, ville, points_fidelite
FROM clients
WHERE client_id = 1;
```

### UPDATE plusieurs lignes

```sql
-- Mettre à jour tous les clients de Paris
-- 1. Vérifier combien de lignes seront affectées
SELECT COUNT(*) FROM clients WHERE ville = 'Paris';

-- 2. Voir les lignes qui seront modifiées
SELECT * FROM clients WHERE ville = 'Paris';

-- 3. Effectuer la mise à jour
UPDATE clients
SET points_fidelite = points_fidelite + 10
WHERE ville = 'Paris';

-- Query OK, 1 row affected (1 client parisien)
```

### UPDATE sans WHERE (DANGER !)

```sql
-- ❌❌❌ CATASTROPHIQUE : Met à jour TOUTES les lignes
UPDATE clients SET ville = 'Paris';
-- Query OK, 5 rows affected
-- Tous les clients sont maintenant à Paris !

-- Pour corriger cette erreur, il faudrait une sauvegarde ou une transaction
ROLLBACK;  -- Seulement si dans une transaction
```

💡 **Protection** : Utilisez l'option `sql_safe_updates` pour éviter les UPDATE/DELETE sans WHERE.

```sql
-- Activer le mode sécurisé
SET sql_safe_updates = 1;

-- Maintenant cette commande sera rejetée
UPDATE clients SET ville = 'Paris';
-- ERROR: You are using safe update mode and you tried to update a table
-- without a WHERE that uses a KEY column
```

---

## UPDATE avec calculs et expressions

### Calculs arithmétiques

```sql
-- Augmenter tous les prix de 10%
UPDATE livres
SET prix = prix * 1.10
WHERE genre = 'Fantasy';

-- Doubler les points de fidélité des bons clients
UPDATE clients
SET points_fidelite = points_fidelite * 2
WHERE points_fidelite >= 100;

-- Appliquer une réduction de 5€ sur les livres chers
UPDATE livres
SET prix = prix - 5
WHERE prix > 30;

-- Incrémenter un compteur
UPDATE livres
SET vues = vues + 1
WHERE livre_id = 5;
```

### Expressions conditionnelles (CASE)

```sql
-- Appliquer différentes remises selon le stock
UPDATE livres
SET prix = CASE
    WHEN stock > 50 THEN prix * 0.80  -- 20% de réduction si stock > 50
    WHEN stock > 20 THEN prix * 0.90  -- 10% de réduction si stock > 20
    ELSE prix * 0.95                  -- 5% de réduction sinon
END
WHERE genre = 'Roman';

-- Mettre à jour le statut selon les conditions
UPDATE clients
SET actif = CASE
    WHEN points_fidelite > 100 THEN TRUE
    WHEN date_inscription < DATE_SUB(NOW(), INTERVAL 2 YEAR) THEN FALSE
    ELSE actif
END;
```

### Fonctions dans UPDATE

```sql
-- Mettre les emails en minuscules
UPDATE clients
SET email = LOWER(email);

-- Nettoyer les espaces dans les noms
UPDATE clients
SET nom = TRIM(nom);

-- Formater les numéros de téléphone
UPDATE clients
SET telephone = CONCAT('+33', SUBSTRING(telephone, 2))
WHERE telephone LIKE '0%';

-- Mettre à jour avec la date actuelle
UPDATE commandes
SET date_livraison = NOW()
WHERE statut = 'livree' AND date_livraison IS NULL;
```

---

## UPDATE avec sous-requêtes

### UPDATE basé sur une autre table

```sql
-- Mettre à jour les points de fidélité selon le total des commandes
UPDATE clients c
SET points_fidelite = (
    SELECT COALESCE(SUM(montant_total), 0) * 10
    FROM commandes
    WHERE client_id = c.client_id
);

-- Marquer comme inactifs les clients sans commande
UPDATE clients
SET actif = FALSE
WHERE client_id NOT IN (
    SELECT DISTINCT client_id FROM commandes
);

-- Appliquer une promotion aux clients ayant dépensé plus de 200€
UPDATE clients
SET points_fidelite = points_fidelite + 50
WHERE client_id IN (
    SELECT client_id
    FROM commandes
    GROUP BY client_id
    HAVING SUM(montant_total) > 200
);
```

### UPDATE avec JOIN

```sql
-- Mettre à jour le stock des livres selon les commandes
UPDATE livres l
INNER JOIN lignes_commande lc ON l.livre_id = lc.livre_id
INNER JOIN commandes c ON lc.commande_id = c.commande_id
SET l.stock = l.stock - lc.quantite
WHERE c.commande_id = 123 AND c.statut = 'confirmee';

-- Copier une valeur d'une table liée
UPDATE clients c
INNER JOIN commandes cmd ON c.client_id = cmd.client_id
SET c.derniere_commande = cmd.date_commande
WHERE cmd.commande_id = (
    SELECT MAX(commande_id)
    FROM commandes
    WHERE client_id = c.client_id
);
```

---

## UPDATE avec LIMIT

```sql
-- Mettre à jour seulement les N premières lignes
UPDATE livres
SET stock = stock + 10
WHERE stock < 5
ORDER BY livre_id
LIMIT 5;

-- Utile pour traiter des lots progressivement
UPDATE commandes
SET statut = 'archivee'
WHERE statut = 'livree'
  AND date_commande < DATE_SUB(NOW(), INTERVAL 1 YEAR)
ORDER BY commande_id
LIMIT 1000;
```

---

## DELETE - Suppression de lignes

### Syntaxe de base

```sql
DELETE FROM nom_table
WHERE condition;
```

⚠️ **CRITIQUE** : Sans WHERE, TOUTES les lignes sont supprimées !

### DELETE simple

```sql
-- 1. Vérifier d'abord ce qui va être supprimé
SELECT * FROM clients WHERE client_id = 3;

-- 2. Supprimer la ligne
DELETE FROM clients WHERE client_id = 3;

-- Query OK, 1 row deleted

-- 3. Vérifier la suppression
SELECT * FROM clients WHERE client_id = 3;
-- Résultat : vide (0 lignes)
```

### DELETE plusieurs lignes

```sql
-- 1. Compter les lignes qui seront supprimées
SELECT COUNT(*) FROM commandes WHERE statut = 'annulee';

-- 2. Voir les lignes
SELECT * FROM commandes WHERE statut = 'annulee';

-- 3. Supprimer
DELETE FROM commandes WHERE statut = 'annulee';

-- Query OK, X rows deleted
```

### DELETE avec conditions multiples

```sql
-- Supprimer les clients inactifs sans commandes
DELETE FROM clients
WHERE actif = FALSE
  AND client_id NOT IN (SELECT client_id FROM commandes);

-- Supprimer les vieilles données
DELETE FROM logs
WHERE date_creation < DATE_SUB(NOW(), INTERVAL 6 MONTH);

-- Supprimer selon plusieurs critères
DELETE FROM livres
WHERE stock = 0
  AND annee_publication < 1950
  AND genre NOT IN ('Fantasy', 'Science-Fiction');
```

### DELETE sans WHERE (CATASTROPHE !)

```sql
-- ❌❌❌ SUPPRIME TOUTES LES LIGNES
DELETE FROM clients;
-- Query OK, 5 rows deleted
-- TOUS les clients supprimés !

-- Sans transaction ou sauvegarde, les données sont perdues définitivement
```

---

## DELETE avec JOIN

### DELETE depuis plusieurs tables liées

```sql
-- Supprimer un client ET toutes ses commandes
-- (si pas de CASCADE sur la FK)
DELETE c, cmd
FROM clients c
LEFT JOIN commandes cmd ON c.client_id = cmd.client_id
WHERE c.client_id = 5;

-- Supprimer les commandes des clients inactifs
DELETE cmd
FROM commandes cmd
INNER JOIN clients c ON cmd.client_id = c.client_id
WHERE c.actif = FALSE;
```

---

## DELETE vs TRUNCATE

### Différences fondamentales

| Critère | DELETE | TRUNCATE |
|---------|--------|----------|
| **Clause WHERE** | ✅ Oui | ❌ Non (tout supprimer) |
| **Transaction** | ✅ Peut ROLLBACK | ⚠️ Pas toujours (InnoDB: oui) |
| **Triggers** | ✅ Déclenche | ❌ Ne déclenche pas |
| **AUTO_INCREMENT** | ❌ Conserve | ✅ Réinitialise |
| **Performance** | ❌ Plus lent | ✅ Très rapide |
| **Clés étrangères** | ✅ Vérifie | ❌ Échoue si FK existent |
| **Retour** | Nombre de lignes | 0 (MySQL) |

### TRUNCATE - Vider une table

```sql
-- TRUNCATE : Supprime toutes les lignes, très rapide
TRUNCATE TABLE logs;

-- Équivalent à (mais beaucoup plus rapide) :
DELETE FROM logs;

-- TRUNCATE réinitialise AUTO_INCREMENT
CREATE TABLE test (id INT AUTO_INCREMENT PRIMARY KEY, valeur INT);
INSERT INTO test (valeur) VALUES (10), (20), (30);
SELECT * FROM test;  -- id: 1, 2, 3

DELETE FROM test;
INSERT INTO test (valeur) VALUES (40);
SELECT * FROM test;  -- id: 4 (AUTO_INCREMENT conservé)

-- Avec TRUNCATE
TRUNCATE TABLE test;
INSERT INTO test (valeur) VALUES (50);
SELECT * FROM test;  -- id: 1 (AUTO_INCREMENT réinitialisé)
```

### Quand utiliser TRUNCATE ?

```sql
-- ✅ BON : Vider une table temporaire
TRUNCATE TABLE temp_import;

-- ✅ BON : Réinitialiser une table de test
TRUNCATE TABLE test_data;

-- ✅ BON : Nettoyer des logs anciens (toute la table)
TRUNCATE TABLE logs_2024;

-- ❌ MAUVAIS : Si des clés étrangères pointent vers cette table
TRUNCATE TABLE clients;
-- ERROR: Cannot truncate a table referenced in a foreign key constraint

-- ❌ MAUVAIS : Si vous voulez supprimer seulement certaines lignes
-- (TRUNCATE n'a pas de WHERE)
```

### TRUNCATE avec clés étrangères

```sql
-- TRUNCATE échoue si des FK existent
TRUNCATE TABLE clients;
-- ERROR: Cannot truncate a table referenced in a foreign key constraint

-- Solution 1 : Supprimer les lignes liées d'abord
DELETE FROM commandes;
TRUNCATE TABLE clients;

-- Solution 2 : Désactiver temporairement les FK (DANGER)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE clients;
TRUNCATE TABLE commandes;
SET FOREIGN_KEY_CHECKS = 1;
-- ⚠️ À utiliser avec extrême prudence
```

---

## Gestion des clés étrangères

### DELETE avec ON DELETE CASCADE

```sql
-- Si la FK a ON DELETE CASCADE
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
        ON DELETE CASCADE
);

-- Supprimer le client supprime automatiquement ses commandes
DELETE FROM clients WHERE client_id = 1;
-- Les commandes du client 1 sont automatiquement supprimées
```

### DELETE avec ON DELETE RESTRICT

```sql
-- Si la FK a ON DELETE RESTRICT (défaut)
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
        ON DELETE RESTRICT
);

-- Tentative de suppression d'un client avec commandes
DELETE FROM clients WHERE client_id = 1;
-- ERROR: Cannot delete or update a parent row: a foreign key constraint fails

-- Solution : Supprimer les commandes d'abord
DELETE FROM commandes WHERE client_id = 1;
DELETE FROM clients WHERE client_id = 1;
-- OK
```

### DELETE avec ON DELETE SET NULL

```sql
-- Si la FK a ON DELETE SET NULL
CREATE TABLE commandes (
    commande_id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT,  -- Peut être NULL
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
        ON DELETE SET NULL
);

-- Supprimer le client met client_id à NULL dans commandes
DELETE FROM clients WHERE client_id = 1;
-- Les commandes du client 1 ont maintenant client_id = NULL
```

---

## Transactions et sécurité

### Utiliser les transactions pour UPDATE/DELETE

```sql
-- Démarrer une transaction
START TRANSACTION;

-- Effectuer des modifications
UPDATE clients SET ville = 'Paris' WHERE ville = 'Lyon';
DELETE FROM commandes WHERE statut = 'annulee';

-- Vérifier les changements
SELECT * FROM clients WHERE ville = 'Paris';
SELECT COUNT(*) FROM commandes WHERE statut = 'annulee';

-- Si tout est correct
COMMIT;

-- Si erreur ou doute
ROLLBACK;
```

### Exemple de transaction complète

```sql
START TRANSACTION;

-- 1. Vérifier le client existe
SELECT * FROM clients WHERE client_id = 5;

-- 2. Mettre à jour les points
UPDATE clients
SET points_fidelite = points_fidelite + 100
WHERE client_id = 5;

-- 3. Créer une commande
INSERT INTO commandes (client_id, montant_total)
VALUES (5, 75.99);

-- 4. Si tout est OK
COMMIT;

-- Si une erreur se produit
-- ROLLBACK;  -- Annule TOUTES les modifications
```

### Savepoints pour transactions complexes

```sql
START TRANSACTION;

-- Point de sauvegarde 1
UPDATE clients SET points_fidelite = points_fidelite + 10;
SAVEPOINT point1;

-- Point de sauvegarde 2
DELETE FROM commandes WHERE statut = 'annulee';
SAVEPOINT point2;

-- Oups, erreur dans la suppression
-- Revenir au point 2 seulement
ROLLBACK TO SAVEPOINT point2;

-- Les UPDATE sont conservés, mais pas les DELETE

COMMIT;
```

---

## Bonnes pratiques

### 1. Toujours SELECT avant UPDATE/DELETE

```sql
-- ✅ BONNE PRATIQUE
-- Étape 1 : SELECT pour vérifier
SELECT * FROM clients WHERE ville = 'Lyon';
-- Résultat : 3 clients à Lyon

-- Étape 2 : UPDATE avec la même condition
UPDATE clients SET points_fidelite = 0 WHERE ville = 'Lyon';
-- Query OK, 3 rows affected (comme prévu)

-- Étape 3 : Vérifier la modification
SELECT * FROM clients WHERE ville = 'Lyon';
```

### 2. Utiliser des transactions pour modifications critiques

```sql
-- ✅ BON : Transaction pour modifications liées
START TRANSACTION;

UPDATE clients SET actif = FALSE WHERE client_id = 10;
UPDATE commandes SET statut = 'annulee' WHERE client_id = 10;

-- Vérifier avant de committer
SELECT * FROM clients WHERE client_id = 10;
SELECT * FROM commandes WHERE client_id = 10;

COMMIT;
```

### 3. Utiliser LIMIT avec UPDATE/DELETE pour traiter par lots

```sql
-- ✅ BON : Traiter 1000 lignes à la fois
WHILE (SELECT COUNT(*) FROM logs WHERE date_log < '2024-01-01') > 0 DO
    DELETE FROM logs
    WHERE date_log < '2024-01-01'
    LIMIT 1000;

    -- Pause pour ne pas surcharger
    SELECT SLEEP(1);
END WHILE;
```

### 4. Sauvegarder avant modifications massives

```sql
-- ✅ BON : Sauvegarde avant UPDATE massif
-- Dans le terminal
mysqldump -u user -p librairie > backup_avant_update.sql

-- Ensuite dans MariaDB
UPDATE livres SET prix = prix * 1.15;

-- Si problème, restaurer
-- mysql -u user -p librairie < backup_avant_update.sql
```

### 5. Activer sql_safe_updates

```sql
-- ✅ BON : Mode sécurisé
SET sql_safe_updates = 1;

-- Maintenant ces commandes échouent
UPDATE clients SET ville = 'Paris';  -- ERROR
DELETE FROM clients;                 -- ERROR

-- Mais celles-ci fonctionnent (avec WHERE sur PK/index)
UPDATE clients SET ville = 'Paris' WHERE client_id = 5;  -- OK
DELETE FROM clients WHERE client_id = 5;                 -- OK
```

### 6. Documenter les modifications importantes

```sql
-- ✅ BON : Traçabilité
CREATE TABLE audit_modifications (
    audit_id INT PRIMARY KEY AUTO_INCREMENT,
    table_modifiee VARCHAR(100),
    action VARCHAR(50),
    nombre_lignes INT,
    utilisateur VARCHAR(100),
    date_modification DATETIME DEFAULT CURRENT_TIMESTAMP,
    description TEXT
);

-- Avant une grosse modification
START TRANSACTION;

SET @lignes_avant = (SELECT COUNT(*) FROM clients WHERE actif = FALSE);

UPDATE clients SET actif = TRUE WHERE points_fidelite > 100;

SET @lignes_apres = (SELECT COUNT(*) FROM clients WHERE actif = FALSE);

INSERT INTO audit_modifications
    (table_modifiee, action, nombre_lignes, utilisateur, description)
VALUES
    ('clients', 'UPDATE', @lignes_avant - @lignes_apres, USER(),
     'Réactivation des clients avec plus de 100 points');

COMMIT;
```

### 7. Utiliser WHERE avec colonnes indexées

```sql
-- ✅ RAPIDE : WHERE sur PRIMARY KEY
UPDATE clients SET ville = 'Paris' WHERE client_id = 5;

-- ✅ RAPIDE : WHERE sur colonne indexée
UPDATE clients SET points_fidelite = 0 WHERE email = 'alice@example.com';

-- ❌ LENT : WHERE sur colonne non indexée (scan complet)
UPDATE clients SET actif = TRUE WHERE telephone LIKE '%1234%';

-- Solution : Créer un index
CREATE INDEX idx_telephone ON clients(telephone);
```

---

## Pièges courants à éviter

### ❌ Piège 1 : Oublier WHERE

```sql
-- ❌ CATASTROPHE : Toutes les lignes modifiées
UPDATE clients SET ville = 'Paris';
-- Tous les clients sont maintenant à Paris !

-- ✅ CORRECT : WHERE spécifique
UPDATE clients SET ville = 'Paris' WHERE client_id = 5;
```

### ❌ Piège 2 : Tester avec = au lieu de IN pour sous-requête

```sql
-- ❌ ERREUR si la sous-requête retourne plusieurs lignes
UPDATE clients
SET actif = TRUE
WHERE client_id = (SELECT client_id FROM commandes);
-- ERROR: Subquery returns more than 1 row

-- ✅ CORRECT : Utiliser IN
UPDATE clients
SET actif = TRUE
WHERE client_id IN (SELECT client_id FROM commandes);
```

### ❌ Piège 3 : Ne pas vérifier les contraintes FK avant DELETE

```sql
-- ❌ Tentative de suppression sans vérifier les FK
DELETE FROM clients WHERE client_id = 1;
-- ERROR: Cannot delete or update a parent row: a foreign key constraint fails

-- ✅ CORRECT : Vérifier les dépendances d'abord
SELECT COUNT(*) FROM commandes WHERE client_id = 1;
-- Si des commandes existent, les gérer d'abord

-- Option 1 : Supprimer les commandes
DELETE FROM commandes WHERE client_id = 1;
DELETE FROM clients WHERE client_id = 1;

-- Option 2 : Archiver au lieu de supprimer
UPDATE clients SET actif = FALSE WHERE client_id = 1;
```

### ❌ Piège 4 : UPDATE de la même table dans une sous-requête

```sql
-- ❌ ERREUR : Utiliser la table dans UPDATE et sous-requête
UPDATE livres
SET prix = prix * 0.9
WHERE prix > (SELECT AVG(prix) FROM livres);
-- ERROR: You can't specify target table 'livres' for update in FROM clause

-- ✅ SOLUTION : Utiliser une sous-requête dérivée
UPDATE livres
SET prix = prix * 0.9
WHERE prix > (SELECT avg_prix FROM (SELECT AVG(prix) AS avg_prix FROM livres) AS temp);

-- ✅ ALTERNATIVE : Variable temporaire
SET @prix_moyen = (SELECT AVG(prix) FROM livres);
UPDATE livres SET prix = prix * 0.9 WHERE prix > @prix_moyen;
```

### ❌ Piège 5 : Pas de COMMIT après START TRANSACTION

```sql
-- ❌ MAUVAIS : Transaction non terminée
START TRANSACTION;
UPDATE clients SET points_fidelite = 0;
-- ... puis déconnexion ou crash
-- Les modifications sont perdues (ROLLBACK automatique)

-- ✅ BON : Toujours COMMIT ou ROLLBACK
START TRANSACTION;
UPDATE clients SET points_fidelite = 0;
-- Vérifier
SELECT * FROM clients;
-- Si OK
COMMIT;
```

### ❌ Piège 6 : DELETE au lieu de TRUNCATE pour vider une table

```sql
-- ❌ INEFFICACE : DELETE toutes les lignes (lent sur grosse table)
DELETE FROM logs_temporaires;
-- Prend plusieurs secondes/minutes

-- ✅ RAPIDE : TRUNCATE
TRUNCATE TABLE logs_temporaires;
-- Quasi instantané, même avec millions de lignes
```

### ❌ Piège 7 : UPDATE avec JOIN sans précaution

```sql
-- ❌ DANGEREUX : Mise à jour multiple non voulue
UPDATE clients c
INNER JOIN commandes cmd ON c.client_id = cmd.client_id
SET c.derniere_commande = cmd.date_commande;
-- Si un client a plusieurs commandes, UPDATE peut être imprévisible

-- ✅ CORRECT : S'assurer d'un seul match
UPDATE clients c
INNER JOIN (
    SELECT client_id, MAX(date_commande) AS derniere_date
    FROM commandes
    GROUP BY client_id
) last_cmd ON c.client_id = last_cmd.client_id
SET c.derniere_commande = last_cmd.derniere_date;
```

---

## Exemples pratiques complets

### Exemple 1 : Gestion du stock de livres

```sql
-- Scenario : Mise à jour du stock après une commande

START TRANSACTION;

-- 1. Vérifier le stock disponible
SELECT l.livre_id, l.titre, l.stock
FROM livres l
WHERE l.livre_id IN (3, 5, 7);

-- Résultat :
-- livre_id: 3, titre: "L'Étranger", stock: 20
-- livre_id: 5, titre: "Le Seigneur des Anneaux", stock: 25
-- livre_id: 7, titre: "Harry Potter...", stock: 40

-- 2. Créer la commande
INSERT INTO commandes (client_id, statut, montant_total)
VALUES (1, 'confirmee', 71.48);

SET @commande_id = LAST_INSERT_ID();

-- 3. Ajouter les lignes de commande
INSERT INTO lignes_commande (commande_id, livre_id, quantite, prix_unitaire)
VALUES
    (@commande_id, 3, 2, 12.50),
    (@commande_id, 5, 1, 34.99),
    (@commande_id, 7, 1, 22.99);

-- 4. Mettre à jour le stock
UPDATE livres l
INNER JOIN lignes_commande lc ON l.livre_id = lc.livre_id
SET l.stock = l.stock - lc.quantite
WHERE lc.commande_id = @commande_id;

-- Query OK, 3 rows affected

-- 5. Vérifier le nouveau stock
SELECT l.livre_id, l.titre, l.stock
FROM livres l
WHERE l.livre_id IN (3, 5, 7);

-- Résultat :
-- livre_id: 3, stock: 18 (20 - 2)
-- livre_id: 5, stock: 24 (25 - 1)
-- livre_id: 7, stock: 39 (40 - 1)

COMMIT;
```

### Exemple 2 : Nettoyage de données obsolètes

```sql
-- Scenario : Archiver et supprimer les vieilles commandes

START TRANSACTION;

-- 1. Créer table d'archive si elle n'existe pas
CREATE TABLE IF NOT EXISTS commandes_archive LIKE commandes;

-- 2. Copier les commandes de plus de 2 ans dans l'archive
INSERT INTO commandes_archive
SELECT *
FROM commandes
WHERE date_commande < DATE_SUB(NOW(), INTERVAL 2 YEAR);

-- X rows affected

-- 3. Vérifier l'insertion
SELECT COUNT(*) AS nb_archivees FROM commandes_archive;

-- 4. Supprimer de la table principale
DELETE FROM commandes
WHERE date_commande < DATE_SUB(NOW(), INTERVAL 2 YEAR);

-- X rows deleted (même nombre que step 2)

-- 5. Vérifier
SELECT
    (SELECT COUNT(*) FROM commandes) AS commandes_actives,
    (SELECT COUNT(*) FROM commandes_archive) AS commandes_archivees;

COMMIT;
```

### Exemple 3 : Mise à jour conditionnelle des statuts

```sql
-- Scenario : Mettre à jour automatiquement les statuts de commandes

START TRANSACTION;

-- 1. Commandes "en_attente" depuis plus de 7 jours → "annulee"
UPDATE commandes
SET statut = 'annulee'
WHERE statut = 'en_attente'
  AND date_commande < DATE_SUB(NOW(), INTERVAL 7 DAY);

-- 2. Commandes "confirmee" depuis plus de 3 jours → "expediee"
UPDATE commandes
SET statut = 'expediee',
    date_expedition = NOW()
WHERE statut = 'confirmee'
  AND date_commande < DATE_SUB(NOW(), INTERVAL 3 DAY);

-- 3. Commandes "expediee" depuis plus de 5 jours → "livree"
UPDATE commandes
SET statut = 'livree',
    date_livraison = NOW()
WHERE statut = 'expediee'
  AND date_expedition < DATE_SUB(NOW(), INTERVAL 5 DAY);

-- 4. Vérifier les changements
SELECT
    statut,
    COUNT(*) AS nombre
FROM commandes
GROUP BY statut;

COMMIT;
```

### Exemple 4 : Synchronisation de données

```sql
-- Scenario : Mettre à jour les totaux de commandes

START TRANSACTION;

-- 1. Créer une table temporaire avec les totaux calculés
CREATE TEMPORARY TABLE temp_totaux AS
SELECT
    commande_id,
    SUM(quantite * prix_unitaire) AS total_calcule
FROM lignes_commande
GROUP BY commande_id;

-- 2. Mettre à jour les commandes avec les vrais totaux
UPDATE commandes c
INNER JOIN temp_totaux t ON c.commande_id = t.commande_id
SET c.montant_total = t.total_calcule
WHERE ABS(c.montant_total - t.total_calcule) > 0.01;  -- Différence > 1 centime

-- X rows affected

-- 3. Rapport des corrections
SELECT
    c.commande_id,
    c.montant_total AS ancien_total,
    t.total_calcule AS nouveau_total,
    t.total_calcule - c.montant_total AS difference
FROM commandes c
INNER JOIN temp_totaux t ON c.commande_id = t.commande_id
WHERE ABS(c.montant_total - t.total_calcule) > 0.01;

DROP TEMPORARY TABLE temp_totaux;

COMMIT;
```

### Exemple 5 : Gestion des doublons

```sql
-- Scenario : Supprimer les doublons et garder le plus récent

START TRANSACTION;

-- 1. Identifier les doublons (même email)
SELECT email, COUNT(*) as nb
FROM clients
GROUP BY email
HAVING COUNT(*) > 1;

-- 2. Créer table temporaire avec les IDs à garder (plus récent)
CREATE TEMPORARY TABLE clients_a_garder AS
SELECT email, MAX(client_id) AS client_id_a_garder
FROM clients
GROUP BY email;

-- 3. Avant suppression : Fusionner les données (points de fidélité)
UPDATE clients c1
INNER JOIN (
    SELECT
        cg.client_id_a_garder,
        SUM(c2.points_fidelite) AS total_points
    FROM clients_a_garder cg
    INNER JOIN clients c2 ON cg.email = c2.email
    GROUP BY cg.client_id_a_garder
) fusion ON c1.client_id = fusion.client_id_a_garder
SET c1.points_fidelite = fusion.total_points;

-- 4. Transférer les commandes vers le client à garder
UPDATE commandes cmd
INNER JOIN clients c_old ON cmd.client_id = c_old.client_id
INNER JOIN clients_a_garder cg ON c_old.email = cg.email
SET cmd.client_id = cg.client_id_a_garder
WHERE cmd.client_id != cg.client_id_a_garder;

-- 5. Supprimer les doublons
DELETE c
FROM clients c
LEFT JOIN clients_a_garder cg
    ON c.client_id = cg.client_id_a_garder
WHERE cg.client_id_a_garder IS NULL
  AND c.email IN (SELECT email FROM clients_a_garder);

-- 6. Vérifier qu'il n'y a plus de doublons
SELECT email, COUNT(*) as nb
FROM clients
GROUP BY email
HAVING COUNT(*) > 1;
-- Résultat : vide

DROP TEMPORARY TABLE clients_a_garder;

COMMIT;
```

---

## Performance et optimisation

### Indexer les colonnes dans WHERE

```sql
-- Les UPDATE/DELETE utilisent WHERE → bénéficient des index
CREATE INDEX idx_clients_ville ON clients(ville);
CREATE INDEX idx_commandes_statut ON commandes(statut);
CREATE INDEX idx_commandes_date ON commandes(date_commande);

-- Maintenant rapide grâce à l'index
UPDATE clients SET points_fidelite = 0 WHERE ville = 'Paris';
DELETE FROM commandes WHERE statut = 'annulee';
```

### Batch processing pour grandes tables

```sql
-- Au lieu de tout supprimer d'un coup (peut bloquer longtemps)
DELIMITER //
CREATE PROCEDURE nettoyer_vieux_logs()
BEGIN
    DECLARE rows_deleted INT DEFAULT 1;

    WHILE rows_deleted > 0 DO
        DELETE FROM logs
        WHERE date_log < DATE_SUB(NOW(), INTERVAL 1 YEAR)
        LIMIT 10000;

        SET rows_deleted = ROW_COUNT();

        -- Pause pour ne pas surcharger
        DO SLEEP(1);
    END WHILE;
END //
DELIMITER ;

-- Exécuter
CALL nettoyer_vieux_logs();
```

### EXPLAIN pour analyser UPDATE/DELETE

```sql
-- Voir comment MariaDB exécute l'UPDATE
EXPLAIN UPDATE clients SET ville = 'Paris' WHERE client_id > 100;

-- Résultat montre :
-- - Type de recherche (range, ALL, etc.)
-- - Index utilisé
-- - Nombre de lignes à scanner

-- Si "rows" est très élevé, créer un index
CREATE INDEX idx_client_id ON clients(client_id);
```

---

## ✅ Points clés à retenir

- **UPDATE SET** : Modifier valeurs de colonnes
- **DELETE FROM** : Supprimer lignes spécifiques
- **TRUNCATE** : Vider table (rapide, réinitialise AUTO_INCREMENT)
- **WHERE OBLIGATOIRE** : Sans WHERE, toutes les lignes affectées
- **SELECT avant** : Toujours vérifier avec SELECT avant UPDATE/DELETE
- **Transactions** : START TRANSACTION / COMMIT / ROLLBACK pour sécurité
- **sql_safe_updates** : Protection contre UPDATE/DELETE sans WHERE
- **LIMIT** : Traiter par lots pour grandes tables
- **Clés étrangères** : Comprendre CASCADE, RESTRICT, SET NULL
- **TRUNCATE vs DELETE** : TRUNCATE plus rapide mais moins flexible
- **Index** : Accélère les UPDATE/DELETE avec WHERE
- **Sauvegarde** : Backup avant modifications massives
- **Audit** : Tracer les modifications importantes
- **ROW_COUNT()** : Vérifier nombre de lignes affectées

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 UPDATE Statement](https://mariadb.com/kb/en/update/)
- [📖 DELETE Statement](https://mariadb.com/kb/en/delete/)
- [📖 TRUNCATE TABLE](https://mariadb.com/kb/en/truncate-table/)
- [📖 Transactions](https://mariadb.com/kb/en/transactions/)
- [📖 Foreign Keys](https://mariadb.com/kb/en/foreign-keys/)
- [📖 SQL Safe Updates Mode](https://mariadb.com/kb/en/sql-mode/)

### Lectures complémentaires
- [Transaction Isolation Levels](https://mariadb.com/kb/en/transaction-isolation-levels/)
- [Optimizing DELETE](https://mariadb.com/kb/en/optimization-and-tuning-deleting-data/)

---

## ➡️ Prochaines sections

**Chapitre 3 : Requêtes SQL avancées**

Dans le prochain chapitre, vous apprendrez :
- Les jointures (INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL JOIN)
- Les sous-requêtes et requêtes imbriquées
- Les fonctions d'agrégation (GROUP BY, HAVING)
- Les opérateurs d'ensemble (UNION, INTERSECT, EXCEPT)
- Les fonctions de fenêtre (window functions)

---


⏭️ [Requêtes SQL Intermédiaires](/03-requetes-sql-intermediaires/README.md)

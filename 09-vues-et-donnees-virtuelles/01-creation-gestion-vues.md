🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 Création et gestion des vues (CREATE VIEW, ALTER VIEW)

> **Niveau** : Intermédiaire
> **Durée estimée** : 1.5-2 heures
> **Prérequis** : Chapitre 9 (Introduction aux vues), maîtrise du SQL SELECT, compréhension des privilèges utilisateurs

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Créer des vues avec la syntaxe complète CREATE VIEW et toutes ses options
- Comprendre et choisir l'algorithme approprié (MERGE, TEMPTABLE, UNDEFINED)
- Maîtriser les concepts de DEFINER et SQL SECURITY pour la sécurité des vues
- Modifier des vues existantes avec ALTER VIEW
- Gérer le cycle de vie complet des vues (création, modification, suppression)
- Consulter les métadonnées des vues avec SHOW CREATE VIEW et INFORMATION_SCHEMA
- Appliquer les bonnes pratiques de nommage et de documentation

---

## Introduction

La création et la gestion des vues constituent une compétence fondamentale pour structurer efficacement l'accès aux données dans MariaDB. Cette section détaille la syntaxe complète de CREATE VIEW avec toutes ses options, permettant de contrôler précisément le comportement, la sécurité et les performances des vues.

Contrairement à une simple requête SELECT, la création d'une vue nécessite de prendre en compte plusieurs aspects : l'algorithme de traitement, le contexte de sécurité d'exécution, la maintenabilité à long terme et l'impact sur les performances. Une vue bien conçue peut simplifier considérablement le code applicatif, tandis qu'une vue mal conçue peut devenir un goulot d'étranglement.

---

## Syntaxe CREATE VIEW : Vue d'ensemble

### Syntaxe complète

```sql
CREATE
    [OR REPLACE]
    [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    [DEFINER = user]
    [SQL SECURITY {DEFINER | INVOKER}]
    VIEW view_name [(column_list)]
    AS select_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION];
```

### Syntaxe minimale

La forme la plus simple d'une vue :

```sql
CREATE VIEW nom_vue AS
SELECT colonne1, colonne2
FROM table;
```

**Exemple simple** :

```sql
-- Vue basique : clients actifs
CREATE VIEW v_clients_actifs AS
SELECT
    id,
    nom,
    prenom,
    email,
    date_inscription
FROM clients
WHERE statut = 'ACTIF';

-- Utilisation
SELECT * FROM v_clients_actifs WHERE date_inscription > '2025-01-01';
```

💡 **Conseil** : Commencez toujours par des vues simples et ajoutez progressivement les options selon vos besoins spécifiques.

---

## Option CREATE OR REPLACE

### Syntaxe

```sql
CREATE OR REPLACE VIEW nom_vue AS
SELECT ...;
```

### Fonctionnement

L'option **OR REPLACE** permet de recréer une vue existante sans avoir à la supprimer d'abord. C'est l'équivalent de :

```sql
DROP VIEW IF EXISTS nom_vue;
CREATE VIEW nom_vue AS SELECT ...;
```

**Avantages** :
- Évite les erreurs si la vue existe déjà
- Simplifie les scripts de déploiement
- Préserve les privilèges accordés sur la vue

**Exemple pratique** :

```sql
-- Première création
CREATE VIEW v_statistiques AS
SELECT COUNT(*) AS total FROM commandes;

-- Modification avec OR REPLACE
CREATE OR REPLACE VIEW v_statistiques AS
SELECT
    COUNT(*) AS total_commandes,
    SUM(montant_ttc) AS ca_total,
    AVG(montant_ttc) AS panier_moyen
FROM commandes;

-- La vue est mise à jour sans erreur
```

⚠️ **Attention** : OR REPLACE change complètement la définition de la vue. Assurez-vous que les applications utilisant cette vue sont compatibles avec la nouvelle structure.

### Cas d'usage typiques

```sql
-- Script de déploiement idempotent
CREATE OR REPLACE VIEW v_dashboard_ventes AS
SELECT
    DATE(date_vente) AS jour,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_jour
FROM ventes
WHERE date_vente >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(date_vente);

-- Peut être exécuté plusieurs fois sans erreur
```

---

## Option ALGORITHM : Contrôle du traitement

L'option **ALGORITHM** détermine comment MariaDB traite la vue lors de son exécution. C'est un paramètre crucial pour les performances.

### Les trois algorithmes disponibles

```sql
CREATE ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE} VIEW ...
```

### ALGORITHM = MERGE (Par défaut recommandé)

Le serveur **fusionne** la requête de la vue avec la requête de l'utilisateur pour créer une seule requête optimisée.

**Exemple** :

```sql
-- Création de la vue avec MERGE
CREATE ALGORITHM = MERGE VIEW v_produits_disponibles AS
SELECT
    id,
    nom,
    prix,
    stock
FROM produits
WHERE stock > 0;

-- Requête utilisateur
SELECT nom, prix FROM v_produits_disponibles WHERE prix < 100;

-- Devient en interne (fusion) :
SELECT nom, prix
FROM produits
WHERE stock > 0 AND prix < 100;
-- ✅ Les index sur 'stock' et 'prix' peuvent être utilisés
```

**Avantages de MERGE** :
- ✅ Performance optimale
- ✅ Utilisation efficace des index
- ✅ L'optimiseur peut analyser la requête complète
- ✅ Pas de table temporaire créée

**Limitations de MERGE** :

La fusion n'est pas possible si la vue contient :
- Agrégations (COUNT, SUM, AVG, etc.)
- DISTINCT
- GROUP BY / HAVING
- LIMIT
- UNION
- Sous-requêtes dans le SELECT
- Fonctions non déterministes

```sql
-- Cette vue NE PEUT PAS utiliser MERGE (agrégation)
CREATE ALGORITHM = MERGE VIEW v_stats AS
SELECT departement_id, COUNT(*) AS nb_employes
FROM employes
GROUP BY departement_id;
-- MariaDB forcera TEMPTABLE automatiquement
```

### ALGORITHM = TEMPTABLE

Le serveur crée une **table temporaire** avec le résultat de la vue, puis exécute la requête utilisateur sur cette table temporaire.

**Exemple** :

```sql
-- Vue avec agrégation : TEMPTABLE obligatoire
CREATE ALGORITHM = TEMPTABLE VIEW v_ventes_par_mois AS
SELECT
    DATE_FORMAT(date_vente, '%Y-%m') AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant_ttc) AS ca_mensuel,
    AVG(montant_ttc) AS panier_moyen
FROM ventes
GROUP BY DATE_FORMAT(date_vente, '%Y-%m');

-- Requête utilisateur
SELECT * FROM v_ventes_par_mois WHERE mois >= '2025-01';

-- Processus interne :
-- 1. Créer une table temporaire avec toutes les lignes de la vue
-- 2. Scanner la table temporaire pour appliquer WHERE mois >= '2025-01'
-- ⚠️ Pas d'index disponible sur la table temporaire
```

**Avantages de TEMPTABLE** :
- ✅ Seule option pour les vues avec agrégations
- ✅ Résultats cohérents et prévisibles
- ✅ Évite des calculs répétés dans les sous-requêtes

**Inconvénients de TEMPTABLE** :
- ❌ Moins performant (création de table temporaire)
- ❌ Pas d'utilisation des index
- ❌ Consommation mémoire/disque temporaire
- ❌ La table temporaire doit être entièrement générée avant filtrage

💡 **Conseil** : Évitez TEMPTABLE quand possible. Si une vue utilise TEMPTABLE et est lente, envisagez de créer une vraie table mise à jour périodiquement.

### ALGORITHM = UNDEFINED (Par défaut)

MariaDB choisit automatiquement entre MERGE et TEMPTABLE selon la structure de la vue.

```sql
-- Sans préciser ALGORITHM (= UNDEFINED par défaut)
CREATE VIEW v_clients AS
SELECT id, nom, email FROM clients;
-- MariaDB utilisera MERGE (vue simple)

CREATE VIEW v_statistiques AS
SELECT type, COUNT(*) AS total FROM events GROUP BY type;
-- MariaDB utilisera TEMPTABLE (agrégation)
```

**Règle de choix automatique** :
- Si la vue est **compatible MERGE** → MERGE
- Sinon → TEMPTABLE

💡 **Bonne pratique** : Laissez UNDEFINED par défaut, sauf si vous avez une raison spécifique de forcer un algorithme.

### Comparaison des algorithmes

| Critère | MERGE | TEMPTABLE |
|---------|-------|-----------|
| **Performance** | ⚡ Excellente | 🐢 Plus lente |
| **Utilisation index** | ✅ Oui | ❌ Non |
| **Mémoire** | Négligeable | Variable (peut être importante) |
| **Agrégations** | ❌ Non supporté | ✅ Supporté |
| **DISTINCT** | ❌ Non supporté | ✅ Supporté |
| **LIMIT** | ❌ Non supporté | ✅ Supporté |
| **Cas d'usage** | Vues simples de filtrage | Vues avec calculs/agrégations |

### Vérifier l'algorithme utilisé

```sql
-- Créer une vue
CREATE VIEW v_test AS
SELECT departement_id, COUNT(*) AS total
FROM employes
GROUP BY departement_id;

-- Vérifier l'algorithme
SELECT TABLE_NAME, VIEW_DEFINITION, ALGORITHM
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'v_test';

-- Résultat : ALGORITHM = 'TEMPTABLE' (à cause du GROUP BY)
```

---

## Option DEFINER : Contexte d'exécution

L'option **DEFINER** spécifie sous quelle identité utilisateur la vue s'exécute. Cela détermine les privilèges disponibles lors de l'exécution de la vue.

### Syntaxe

```sql
CREATE
    [DEFINER = {'user'@'host' | CURRENT_USER}]
    VIEW view_name AS
    SELECT ...;
```

### Valeurs possibles

1. **user@host explicite** : La vue s'exécute avec les privilèges de cet utilisateur
2. **CURRENT_USER** : Utilise l'utilisateur qui crée la vue (par défaut)

### Comportement par défaut

Si DEFINER n'est pas spécifié, MariaDB utilise automatiquement l'utilisateur courant :

```sql
-- Créé par l'utilisateur 'admin'@'localhost'
CREATE VIEW v_data AS SELECT * FROM table_sensible;

-- Équivalent à :
CREATE DEFINER = 'admin'@'localhost' VIEW v_data AS
SELECT * FROM table_sensible;
```

### Exemple pratique avec sécurité

```sql
-- Scénario : Un utilisateur privilégié crée une vue
-- pour donner accès contrôlé aux données

-- 1. Connexion en tant que 'admin' avec tous les privilèges
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_salaires_publics AS
SELECT
    employe_id,
    nom,
    prenom,
    departement,
    -- Masquer le salaire exact, montrer seulement la tranche
    CASE
        WHEN salaire < 30000 THEN 'Tranche A'
        WHEN salaire < 50000 THEN 'Tranche B'
        WHEN salaire < 80000 THEN 'Tranche C'
        ELSE 'Tranche D'
    END AS tranche_salaire
FROM employes;

-- 2. Donner accès à un utilisateur sans privilèges
GRANT SELECT ON v_salaires_publics TO 'app_user'@'%';

-- 3. L'utilisateur 'app_user' peut interroger la vue
-- même s'il n'a pas accès direct à la table 'employes'
-- car la vue s'exécute avec les privilèges de 'admin'
```

⚠️ **Sécurité importante** : Le DEFINER doit avoir les privilèges nécessaires sur les tables sous-jacentes au moment de la création ET de l'exécution de la vue.

### Changer le DEFINER d'une vue existante

```sql
-- Méthode 1 : Recréer la vue avec OR REPLACE
CREATE OR REPLACE DEFINER = 'nouvel_user'@'localhost'
    VIEW v_data AS
    SELECT * FROM table;

-- Méthode 2 : Utiliser ALTER VIEW (voir section suivante)
```

### Vérifier le DEFINER

```sql
-- Consulter le DEFINER d'une vue
SELECT
    TABLE_NAME,
    DEFINER,
    SQL_SECURITY
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'ma_base'
  AND TABLE_NAME = 'v_salaires_publics';

-- Résultat :
-- TABLE_NAME           | DEFINER              | SECURITY_TYPE
-- v_salaires_publics   | admin@localhost      | DEFINER
```

---

## Option SQL SECURITY : Modèle de sécurité

L'option **SQL SECURITY** détermine sous quelle identité les privilèges sont vérifiés lors de l'exécution de la vue.

### Syntaxe

```sql
CREATE
    SQL SECURITY {DEFINER | INVOKER}
    VIEW view_name AS
    SELECT ...;
```

### SQL SECURITY DEFINER (Par défaut)

La vue s'exécute avec les **privilèges du DEFINER** (créateur de la vue).

```sql
-- Vue créée par 'admin' avec tous les privilèges
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_confidential AS
SELECT * FROM table_sensible;

-- Accorder l'accès à un utilisateur limité
GRANT SELECT ON v_confidential TO 'user_limite'@'%';

-- Quand 'user_limite' interroge la vue :
-- ✅ Les privilèges de 'admin' sont utilisés
-- ✅ L'utilisateur peut voir les données même s'il n'a pas accès direct à table_sensible
SELECT * FROM v_confidential;
```

**Avantages** :
- ✅ Contrôle d'accès granulaire
- ✅ Les utilisateurs n'ont pas besoin de privilèges directs sur les tables sous-jacentes
- ✅ Encapsulation de la sécurité dans la vue

**Inconvénients** :
- ⚠️ Le DEFINER doit exister et avoir les privilèges nécessaires
- ⚠️ Si le DEFINER est supprimé, la vue ne fonctionne plus

### SQL SECURITY INVOKER

La vue s'exécute avec les **privilèges de l'utilisateur qui l'invoque** (exécute la requête).

```sql
-- Vue créée avec INVOKER
CREATE SQL SECURITY INVOKER
    VIEW v_mes_donnees AS
SELECT * FROM utilisateurs;

-- Chaque utilisateur voit seulement ce qu'il a le droit de voir
GRANT SELECT ON v_mes_donnees TO 'user1'@'%';
GRANT SELECT ON v_mes_donnees TO 'user2'@'%';

-- 'user1' interroge la vue
-- ❌ Si 'user1' n'a pas SELECT sur 'utilisateurs', erreur !
-- ✅ Si 'user1' a SELECT sur 'utilisateurs', fonctionne

-- 'user2' interroge la même vue
-- Les privilèges de 'user2' sont vérifiés, pas ceux de 'user1'
```

**Avantages** :
- ✅ Plus flexible : chaque utilisateur utilise ses propres privilèges
- ✅ Pas de dépendance à un DEFINER spécifique
- ✅ Utile pour les vues génériques

**Inconvénients** :
- ⚠️ Tous les utilisateurs doivent avoir les privilèges sur les tables sous-jacentes
- ⚠️ Moins de contrôle centralisé de la sécurité

### Comparaison DEFINER vs INVOKER

| Aspect | SQL SECURITY DEFINER | SQL SECURITY INVOKER |
|--------|---------------------|---------------------|
| **Privilèges vérifiés** | Ceux du créateur (DEFINER) | Ceux de l'utilisateur exécutant |
| **Sécurité** | Centralisée dans la vue | Décentralisée, par utilisateur |
| **Complexité** | Simple pour les utilisateurs | Requiert des privilèges pour tous |
| **Cas d'usage** | Vues de sécurité, masquage | Vues génériques, applications |
| **Par défaut** | ✅ Oui | ❌ Non |

### Exemple combiné : Vue de sécurité multi-niveau

```sql
-- Vue niveau 1 : Données publiques (DEFINER)
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_produits_publics AS
SELECT
    id,
    nom,
    description,
    prix_public
FROM produits;

GRANT SELECT ON v_produits_publics TO 'public_user'@'%';

-- Vue niveau 2 : Données internes (INVOKER)
CREATE SQL SECURITY INVOKER
    VIEW v_produits_internes AS
SELECT
    id,
    nom,
    description,
    prix_public,
    prix_achat,  -- Visible seulement aux utilisateurs autorisés
    marge
FROM produits;

GRANT SELECT ON v_produits_internes TO 'manager'@'%';
GRANT SELECT ON produits TO 'manager'@'%';  -- Nécessaire avec INVOKER
```

💡 **Bonne pratique** : Utilisez **DEFINER** (par défaut) pour les vues de sécurité qui masquent des données sensibles, et **INVOKER** pour les vues génériques où chaque utilisateur doit avoir ses propres privilèges.

---

## Liste de colonnes personnalisée

Vous pouvez spécifier des noms de colonnes différents de ceux dans le SELECT :

```sql
-- Syntaxe avec liste de colonnes
CREATE VIEW view_name (col1, col2, col3) AS
SELECT ...;
```

**Exemple** :

```sql
-- Sans liste de colonnes : utilise les noms du SELECT
CREATE VIEW v_stats1 AS
SELECT
    COUNT(*) AS nombre_total,
    SUM(montant) AS somme_montants
FROM commandes;

-- Avec liste de colonnes : renomme les colonnes
CREATE VIEW v_stats2 (total, ca) AS
SELECT
    COUNT(*),
    SUM(montant)
FROM commandes;

-- Les deux vues ont des colonnes avec des noms différents
SELECT total, ca FROM v_stats2;  -- Utilise les noms de la liste
```

**Cas d'usage** :

1. **Éviter les noms générés automatiquement** :

```sql
-- Sans alias : noms illisibles
CREATE VIEW v_mauvaise AS
SELECT
    COUNT(*),  -- Nom : COUNT(*)
    SUM(montant * quantite)  -- Nom : SUM(montant * quantite)
FROM ventes;

-- Solution 1 : Alias dans le SELECT
CREATE VIEW v_bonne1 AS
SELECT
    COUNT(*) AS nb_ventes,
    SUM(montant * quantite) AS ca_total
FROM ventes;

-- Solution 2 : Liste de colonnes
CREATE VIEW v_bonne2 (nb_ventes, ca_total) AS
SELECT
    COUNT(*),
    SUM(montant * quantite)
FROM ventes;
```

2. **Standardiser les noms dans une application** :

```sql
-- Base de données legacy avec noms en français
CREATE VIEW v_customers (id, name, email, created_at) AS
SELECT
    id_client,
    nom_complet,
    adresse_email,
    date_creation
FROM clients;

-- L'application peut utiliser des noms anglais standard
```

⚠️ **Attention** : Le nombre de colonnes dans la liste DOIT correspondre exactement au nombre de colonnes dans le SELECT.

```sql
-- ❌ ERREUR : 2 colonnes dans la liste, 3 dans le SELECT
CREATE VIEW v_erreur (col1, col2) AS
SELECT a, b, c FROM table;
-- Error: In definition of view, derived table or common table expression, SELECT list and column names list have different column counts
```

---

## Modifier une vue existante : ALTER VIEW

### Syntaxe ALTER VIEW

```sql
ALTER
    [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    [DEFINER = user]
    [SQL SECURITY {DEFINER | INVOKER}]
    VIEW view_name [(column_list)]
    AS select_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION];
```

### Utilisation

ALTER VIEW modifie une vue existante **sans changer les privilèges** accordés sur cette vue.

```sql
-- Vue initiale
CREATE VIEW v_employes AS
SELECT id, nom, prenom, email
FROM employes;

GRANT SELECT ON v_employes TO 'app_user'@'%';

-- Modifier la vue pour ajouter une colonne
ALTER VIEW v_employes AS
SELECT id, nom, prenom, email, departement_id
FROM employes;

-- ✅ Les privilèges de 'app_user' sont préservés
```

### ALTER VIEW vs CREATE OR REPLACE VIEW

Les deux permettent de modifier une vue, avec des différences subtiles :

| Aspect | ALTER VIEW | CREATE OR REPLACE VIEW |
|--------|-----------|------------------------|
| **Si vue n'existe pas** | ❌ Erreur | ✅ Crée la vue |
| **Si vue existe** | ✅ Modifie | ✅ Modifie |
| **Privilèges** | ✅ Préservés | ✅ Préservés |
| **Cas d'usage** | Modification explicite | Scripts idempotents |

**Exemple comparatif** :

```sql
-- Scénario 1 : Modification explicite (ALTER VIEW)
ALTER VIEW v_clients AS
SELECT id, nom, email, telephone FROM clients;
-- ✅ Si v_clients existe : modification
-- ❌ Si v_clients n'existe pas : erreur

-- Scénario 2 : Script déploiement (CREATE OR REPLACE)
CREATE OR REPLACE VIEW v_clients AS
SELECT id, nom, email, telephone FROM clients;
-- ✅ Si v_clients existe : modification
-- ✅ Si v_clients n'existe pas : création
```

💡 **Bonne pratique** :
- Utilisez **ALTER VIEW** dans les migrations où vous voulez garantir que la vue existe déjà
- Utilisez **CREATE OR REPLACE VIEW** dans les scripts de déploiement idempotents

### Modifier uniquement certaines options

Avec ALTER VIEW, vous devez fournir la requête SELECT complète, même pour changer juste une option :

```sql
-- Vue initiale avec MERGE
CREATE ALGORITHM = MERGE VIEW v_data AS
SELECT * FROM table;

-- Changer l'algorithme en TEMPTABLE
ALTER ALGORITHM = TEMPTABLE VIEW v_data AS
SELECT * FROM table;
-- ⚠️ Vous devez répéter le SELECT, même s'il ne change pas
```

---

## Supprimer une vue : DROP VIEW

### Syntaxe

```sql
DROP VIEW [IF EXISTS] view_name [, view_name] ...;
```

### Exemples

```sql
-- Supprimer une vue unique
DROP VIEW v_clients_actifs;

-- Supprimer sans erreur si la vue n'existe pas
DROP VIEW IF EXISTS v_clients_actifs;

-- Supprimer plusieurs vues simultanément
DROP VIEW IF EXISTS v_vue1, v_vue2, v_vue3;
```

### Implications de la suppression

⚠️ **Attention : Vérifiez les dépendances avant suppression !**

```sql
-- 1. Vérifier si d'autres vues dépendent de celle-ci
SELECT
    TABLE_NAME AS vue_dependante
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND VIEW_DEFINITION LIKE '%nom_vue_a_supprimer%';

-- 2. Vérifier si des procédures/fonctions utilisent cette vue
SELECT
    ROUTINE_NAME,
    ROUTINE_TYPE
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE()
  AND ROUTINE_DEFINITION LIKE '%nom_vue_a_supprimer%';

-- 3. Supprimer la vue seulement si aucune dépendance
DROP VIEW IF EXISTS nom_vue_a_supprimer;
```

💡 **Conseil** : Documentez toujours les dépendances entre vos vues dans un fichier de documentation ou un système de gestion de schéma.

---

## Consulter les métadonnées des vues

### SHOW CREATE VIEW

Affiche la définition complète d'une vue :

```sql
SHOW CREATE VIEW nom_vue;
```

**Exemple** :

```sql
-- Créer une vue complexe
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY DEFINER
    ALGORITHM = MERGE
    VIEW v_commandes_recentes AS
SELECT
    id,
    client_id,
    date_commande,
    montant_ttc
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY);

-- Afficher la définition
SHOW CREATE VIEW v_commandes_recentes\G

-- Résultat :
-- *************************** 1. row ***************************
--                 View: v_commandes_recentes
--          Create View: CREATE ALGORITHM=MERGE
--                       DEFINER=`admin`@`localhost`
--                       SQL SECURITY DEFINER
--                       VIEW `v_commandes_recentes` AS
--                       SELECT `id`,`client_id`,`date_commande`,`montant_ttc`
--                       FROM `commandes`
--                       WHERE `date_commande` >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
-- character_set_client: utf8mb4
-- collation_connection: utf8mb4_unicode_ci
```

### INFORMATION_SCHEMA.VIEWS

Requête structurée pour obtenir des métadonnées :

```sql
SELECT
    TABLE_NAME AS nom_vue,
    VIEW_DEFINITION AS definition,
    CHECK_OPTION,
    IS_UPDATABLE,
    DEFINER,
    SECURITY_TYPE,
    ALGORITHM
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'ma_base'
ORDER BY TABLE_NAME;
```

**Exemple avec filtrage** :

```sql
-- Trouver toutes les vues updatable
SELECT
    TABLE_NAME,
    DEFINER
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND IS_UPDATABLE = 'YES';

-- Trouver les vues utilisant TEMPTABLE
SELECT
    TABLE_NAME,
    ALGORITHM,
    CHARACTER_OCTET_LENGTH(VIEW_DEFINITION) AS taille_definition
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND ALGORITHM = 'TEMPTABLE';
```

### SHOW FULL TABLES

Lister toutes les vues d'une base :

```sql
-- Afficher tables ET vues
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- Résultat :
-- +---------------------------+------------+
-- | Tables_in_ma_base         | Table_type |
-- +---------------------------+------------+
-- | v_clients_actifs          | VIEW       |
-- | v_commandes_recentes      | VIEW       |
-- | v_statistiques_mensuelles | VIEW       |
-- +---------------------------+------------+
```

---

## Bonnes pratiques de création de vues

### 1. Nommage cohérent

Adoptez une convention de nommage claire et respectez-la dans tout le projet :

```sql
-- Convention recommandée : préfixe v_ ou view_
CREATE VIEW v_clients_actifs AS ...
CREATE VIEW v_commandes_en_cours AS ...
CREATE VIEW v_dashboard_ventes AS ...

-- Alternative : suffixe _view
CREATE VIEW clients_actifs_view AS ...

-- Convention métier : préfixe selon le domaine
CREATE VIEW rpt_ventes_mensuelles AS ...  -- rpt = report
CREATE VIEW sec_utilisateurs_limites AS ... -- sec = security
```

💡 **Conseil** : Choisissez une convention et documentez-la dans votre guide de style SQL.

### 2. Documentation dans les commentaires

MariaDB ne supporte pas les commentaires sur les vues directement, mais vous pouvez documenter dans la définition :

```sql
CREATE VIEW v_analyse_ventes AS
-- Vue : Analyse des ventes par vendeur
-- Description : Calcule les KPIs de performance pour chaque vendeur
-- Mise à jour : Décembre 2025
-- Auteur : Équipe Analytics
-- Dépendances : tables ventes, vendeurs, produits
-- Note : Utilisée par le dashboard managers (app/dashboard/sales)
SELECT
    v.id AS vendeur_id,
    v.nom AS vendeur_nom,
    COUNT(DISTINCT c.id) AS nb_ventes,
    SUM(c.montant_ttc) AS ca_total,
    AVG(c.montant_ttc) AS panier_moyen,
    COUNT(DISTINCT c.client_id) AS nb_clients_uniques
FROM vendeurs v
LEFT JOIN commandes c ON v.id = c.vendeur_id
WHERE c.date_commande >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
GROUP BY v.id, v.nom;
```

### 3. Éviter SELECT * dans les vues

```sql
-- ❌ Mauvaise pratique : fragile aux changements de schéma
CREATE VIEW v_clients AS
SELECT * FROM clients;

-- ✅ Bonne pratique : spécifier les colonnes explicitement
CREATE VIEW v_clients AS
SELECT
    id,
    nom,
    prenom,
    email,
    telephone,
    adresse,
    ville,
    code_postal,
    date_inscription
FROM clients;

-- Avantages :
-- 1. La vue reste stable si des colonnes sont ajoutées à la table
-- 2. Clarté : on sait exactement ce qui est exposé
-- 3. Sécurité : pas d'exposition accidentelle de nouvelles colonnes sensibles
```

### 4. Préférer ALGORITHM = UNDEFINED par défaut

```sql
-- ❌ Ne forcez pas l'algorithme sans raison
CREATE ALGORITHM = TEMPTABLE VIEW v_data AS
SELECT id, nom FROM clients WHERE actif = 1;
-- Pourquoi TEMPTABLE ? MERGE serait plus performant !

-- ✅ Laissez MariaDB choisir automatiquement
CREATE VIEW v_data AS
SELECT id, nom FROM clients WHERE actif = 1;
-- MariaDB utilisera MERGE (optimal)
```

Forcez l'algorithme uniquement si :
- Vous avez mesuré un problème de performance
- Vous avez une raison technique spécifique

### 5. Utiliser DEFINER approprié pour la sécurité

```sql
-- Pour les vues de sécurité : utilisez DEFINER explicite
CREATE DEFINER = 'app_admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_donnees_securisees AS
SELECT id, nom, email
FROM utilisateurs
WHERE role = 'CLIENT';  -- Filtre intégré

GRANT SELECT ON v_donnees_securisees TO 'app_readonly'@'%';
-- 'app_readonly' peut accéder via la vue mais pas directement à la table
```

### 6. Versionner les vues avec le schéma

Incluez les vues dans votre gestion de version (Git) et vos migrations :

```sql
-- migrations/001_create_views.sql
CREATE OR REPLACE VIEW v_clients_actifs AS
SELECT id, nom, email
FROM clients
WHERE statut = 'ACTIF';

-- migrations/015_update_view_add_column.sql
CREATE OR REPLACE VIEW v_clients_actifs AS
SELECT id, nom, email, telephone  -- Ajout colonne
FROM clients
WHERE statut = 'ACTIF';
```

### 7. Éviter les vues trop complexes ou imbriquées

```sql
-- ❌ Vue sur vue sur vue : mauvaise pratique
CREATE VIEW v_base AS SELECT * FROM t1 JOIN t2 ON ...;
CREATE VIEW v_niveau1 AS SELECT * FROM v_base WHERE ...;
CREATE VIEW v_niveau2 AS SELECT * FROM v_niveau1 WHERE ...;
CREATE VIEW v_niveau3 AS SELECT * FROM v_niveau2 WHERE ...;

-- ✅ Préférer une vue unique bien structurée
CREATE VIEW v_comprehensive AS
SELECT ...
FROM t1
JOIN t2 ON ...
WHERE ...  -- Toutes les conditions en une seule vue
```

**Règle** : Maximum 2-3 niveaux d'imbrication de vues.

---

## ⚠️ Pièges courants à éviter

### 1. Modification de structure sans vérifier les dépendances

```sql
-- Vue existante
CREATE VIEW v_produits AS
SELECT id, nom, prix FROM produits;

-- ❌ Modifier la table sous-jacente sans vérifier
ALTER TABLE produits DROP COLUMN prix;

-- La vue devient invalide !
SELECT * FROM v_produits;
-- Error: Unknown column 'produits.prix' in 'field list'
```

**Solution** : Toujours vérifier et mettre à jour les vues après modification de table.

### 2. Oublier les privilèges avec SQL SECURITY INVOKER

```sql
-- Vue créée avec INVOKER
CREATE SQL SECURITY INVOKER VIEW v_data AS
SELECT * FROM table_sensible;

GRANT SELECT ON v_data TO 'user'@'%';

-- 'user' essaie d'utiliser la vue
SELECT * FROM v_data;
-- ❌ Error: SELECT command denied to user 'user'@'%' for table 'table_sensible'

-- Solution : Accorder les privilèges sur la table sous-jacente
GRANT SELECT ON table_sensible TO 'user'@'%';
```

### 3. Utiliser des fonctions non déterministes avec MERGE

```sql
-- ⚠️ Vue avec fonction non déterministe
CREATE ALGORITHM = MERGE VIEW v_timestamp AS
SELECT id, nom, NOW() AS heure_requete
FROM clients;

-- MariaDB peut forcer TEMPTABLE ou produire des résultats inattendus
-- car NOW() change à chaque appel
```

**Solution** : Acceptez UNDEFINED ou utilisez TEMPTABLE pour les vues avec fonctions non déterministes.

### 4. Croire qu'une vue améliore les performances

```sql
-- ❌ Idée fausse : "Je crée une vue pour améliorer les performances"
CREATE VIEW v_calcul_complexe AS
SELECT
    client_id,
    SUM(montant) AS total,
    COUNT(*) AS nb_commandes,
    AVG(montant) AS moyenne
FROM commandes
GROUP BY client_id;

-- ❌ Le calcul est refait à CHAQUE interrogation de la vue !
SELECT * FROM v_calcul_complexe WHERE client_id = 123;
-- Pas de cache, pas d'optimisation magique

-- ✅ Pour les performances : table de cache mise à jour périodiquement
CREATE TABLE cache_stats_clients AS
SELECT ... FROM ... GROUP BY ...;
-- Rafraîchir avec un EVENT ou un job externe
```

### 5. Noms de colonnes ambigus dans les jointures

```sql
-- ❌ Colonnes sans alias dans une vue avec jointure
CREATE VIEW v_commandes AS
SELECT
    id,  -- ⚠️ De quelle table : commandes.id ou clients.id ?
    nom,
    montant
FROM commandes c
JOIN clients cl ON c.client_id = cl.id;

-- ✅ Toujours qualifier ou renommer explicitement
CREATE VIEW v_commandes AS
SELECT
    c.id AS commande_id,
    cl.id AS client_id,
    cl.nom AS client_nom,
    c.montant AS montant_commande
FROM commandes c
JOIN clients cl ON c.client_id = cl.id;
```

---

## ✅ Points clés à retenir

- **CREATE VIEW** crée une table virtuelle basée sur une requête SELECT stockée
- **CREATE OR REPLACE VIEW** permet de modifier une vue sans la supprimer (idempotent)
- **ALGORITHM** : MERGE (performant) vs TEMPTABLE (agrégations) vs UNDEFINED (automatique)
- **DEFINER** spécifie l'utilisateur sous l'identité duquel la vue s'exécute
- **SQL SECURITY DEFINER** (défaut) : privilèges du créateur | **INVOKER** : privilèges de l'utilisateur
- **ALTER VIEW** modifie une vue existante en préservant les privilèges
- **DROP VIEW** supprime une vue (vérifier les dépendances avant !)
- Utilisez **INFORMATION_SCHEMA.VIEWS** pour consulter les métadonnées des vues
- **Bonnes pratiques** : nommage cohérent, documentation, éviter SELECT *, limiter l'imbrication
- **Les vues ne stockent pas de données** : elles exécutent la requête à chaque appel

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CREATE VIEW](https://mariadb.com/kb/en/create-view/) - Syntaxe complète
- [📖 ALTER VIEW](https://mariadb.com/kb/en/alter-view/) - Modification de vues
- [📖 DROP VIEW](https://mariadb.com/kb/en/drop-view/) - Suppression de vues
- [📖 SHOW CREATE VIEW](https://mariadb.com/kb/en/show-create-view/) - Afficher la définition
- [📖 INFORMATION_SCHEMA.VIEWS](https://mariadb.com/kb/en/information-schema-views-table/) - Métadonnées
- [📖 View Algorithms](https://mariadb.com/kb/en/view-algorithms/) - MERGE vs TEMPTABLE

### Articles complémentaires
- **"Understanding DEFINER and SQL SECURITY in MySQL Views"** - Percona Blog
- **"Best Practices for Managing Database Views"** - Database Journal

---

## ➡️ Section suivante

**[9.2 Vues matérialisées : Alternatives et workarounds](./02-vues-materialisees.md)** : MariaDB ne supporte pas nativement les vues matérialisées. Découvrez plusieurs techniques pour simuler ce comportement : tables de cache, triggers, events planifiés, et stratégies de rafraîchissement.

---


⏭️ [Vues matérialisées : Alternatives et workarounds](/09-vues-et-donnees-virtuelles/02-vues-materialisees.md)

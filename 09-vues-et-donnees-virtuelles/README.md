🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9. Vues et Données Virtuelles

> **Niveau** : Intermédiaire
> **Durée estimée** : 4-6 heures
> **Prérequis** : Maîtrise du SQL (chapitres 2-4), compréhension des jointures et sous-requêtes, notions de modélisation relationnelle

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- Créer et gérer des vues pour simplifier les requêtes complexes et structurer l'accès aux données
- Comprendre les différences entre vues, tables et les implications sur les performances
- Mettre en œuvre des alternatives aux vues matérialisées dans MariaDB
- Identifier quand une vue est updatable et utiliser WITH CHECK OPTION pour garantir l'intégrité
- Exploiter les vues système (INFORMATION_SCHEMA, PERFORMANCE_SCHEMA) pour l'administration
- Optimiser les performances des vues en comprenant les algorithmes MERGE et TEMPTABLE
- Utiliser les vues comme mécanisme de sécurité pour masquer et restreindre l'accès aux données

---

## Introduction

### Qu'est-ce qu'une vue ?

Une **vue** (view) est une requête SQL stockée dans la base de données qui apparaît comme une table virtuelle. Contrairement à une table physique, une vue ne stocke pas de données : elle génère dynamiquement son résultat à partir des tables sous-jacentes lors de chaque interrogation.

```sql
-- Exemple simple : vue des employés actifs
CREATE VIEW v_employes_actifs AS
SELECT
    id,
    nom,
    prenom,
    email,
    departement_id,
    salaire
FROM employes
WHERE statut = 'ACTIF';

-- Utilisation : comme une table normale
SELECT * FROM v_employes_actifs WHERE departement_id = 5;
```

### Pourquoi utiliser des vues ?

Les vues apportent plusieurs avantages significatifs dans la conception et l'exploitation d'une base de données :

#### 1. **Simplification des requêtes complexes**

Les vues permettent d'encapsuler des jointures complexes, des agrégations ou des calculs dans un objet réutilisable :

```sql
-- Vue complexe avec plusieurs jointures
CREATE VIEW v_commandes_details AS
SELECT
    c.id AS commande_id,
    c.date_commande,
    cl.nom AS client_nom,
    cl.email AS client_email,
    p.nom AS produit_nom,
    lc.quantite,
    lc.prix_unitaire,
    (lc.quantite * lc.prix_unitaire) AS montant_ligne,
    c.statut
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id
INNER JOIN lignes_commande lc ON c.id = lc.commande_id
INNER JOIN produits p ON lc.produit_id = p.id;

-- Au lieu de réécrire cette jointure partout :
SELECT * FROM v_commandes_details WHERE client_email = 'john@example.com';
```

#### 2. **Abstraction et indépendance des données**

Les vues créent une couche d'abstraction entre les applications et le schéma physique :

```sql
-- Si la structure de la table change, seule la vue doit être modifiée
CREATE VIEW v_utilisateurs AS
SELECT
    id,
    CONCAT(prenom, ' ', nom) AS nom_complet,  -- Calcul transparent
    email,
    date_creation
FROM utilisateurs;

-- Les applications utilisent v_utilisateurs sans connaître la structure réelle
```

#### 3. **Sécurité et contrôle d'accès**

Les vues permettent de restreindre l'accès à certaines colonnes ou lignes :

```sql
-- Vue qui masque les données sensibles
CREATE VIEW v_employes_public AS
SELECT
    id,
    nom,
    prenom,
    departement_id,
    poste
    -- salaire, num_secu, date_naissance sont exclus
FROM employes;

-- Donner accès uniquement à cette vue
GRANT SELECT ON database.v_employes_public TO 'app_user'@'%';
```

#### 4. **Cohérence des calculs métier**

Les vues garantissent que les calculs métier sont appliqués de manière cohérente :

```sql
CREATE VIEW v_ventes_mensuelles AS
SELECT
    DATE_FORMAT(date_vente, '%Y-%m') AS mois,
    SUM(montant_ht) AS ca_ht,
    SUM(montant_ht * 0.20) AS tva,
    SUM(montant_ht * 1.20) AS ca_ttc,
    COUNT(*) AS nb_ventes
FROM ventes
GROUP BY DATE_FORMAT(date_vente, '%Y-%m');

-- Le calcul de la TVA est toujours correct
```

### Vues vs Tables : Différences clés

| Aspect | Table | Vue |
|--------|-------|-----|
| **Stockage** | Données physiques sur disque | Définition SQL uniquement |
| **Performance** | Accès direct aux données | Exécution de la requête à chaque appel |
| **Mise à jour** | Toujours possible | Sous conditions (vues updatable) |
| **Espace disque** | Occupe de l'espace | Négligeable (sauf TEMPTABLE) |
| **Maintenance** | Index, statistiques | Dépend des tables sous-jacentes |
| **Utilisation** | SELECT, INSERT, UPDATE, DELETE | Principalement SELECT (UPDATE limité) |

### Types de vues dans MariaDB

MariaDB supporte plusieurs concepts autour des vues :

1. **Vues standards** : Définition SQL stockée, données calculées dynamiquement
2. **Vues updatable** : Vues permettant INSERT/UPDATE/DELETE sous certaines conditions
3. **Vues avec WITH CHECK OPTION** : Validation des modifications selon la définition de la vue
4. **Vues système** : INFORMATION_SCHEMA, PERFORMANCE_SCHEMA, mysql.* (métadonnées)

💡 **Important** : MariaDB ne supporte pas nativement les **vues matérialisées** (materialized views comme PostgreSQL), mais des alternatives existent.

### Architecture interne : Comment fonctionnent les vues ?

Lorsqu'une vue est interrogée, MariaDB peut utiliser deux algorithmes principaux :

#### MERGE (Fusion)

Le serveur fusionne la requête de la vue avec la requête utilisateur :

```sql
-- Vue définie avec ALGORITHM=MERGE
CREATE ALGORITHM=MERGE VIEW v_produits_actifs AS
SELECT id, nom, prix FROM produits WHERE actif = 1;

-- Requête utilisateur
SELECT nom, prix FROM v_produits_actifs WHERE prix > 100;

-- Devient en interne (requête fusionnée) :
SELECT nom, prix FROM produits WHERE actif = 1 AND prix > 100;
```

**Avantages** : Performance optimale, les index sont utilisés efficacement.

#### TEMPTABLE (Table temporaire)

Le serveur crée une table temporaire avec le résultat de la vue, puis interroge cette table :

```sql
-- Vue complexe avec agrégation (TEMPTABLE obligatoire)
CREATE VIEW v_stats_departements AS
SELECT
    departement_id,
    COUNT(*) AS nb_employes,
    AVG(salaire) AS salaire_moyen
FROM employes
GROUP BY departement_id;

-- La requête crée une table temporaire d'abord
SELECT * FROM v_stats_departements WHERE nb_employes > 10;
```

**Inconvénient** : Moins performant, pas d'utilisation des index sur la table temporaire.

### Cas d'usage courants des vues

#### 1. Reporting et tableaux de bord

```sql
CREATE VIEW v_dashboard_ventes AS
SELECT
    DATE(date_vente) AS jour,
    COUNT(*) AS nb_commandes,
    SUM(montant_ttc) AS ca_jour,
    AVG(montant_ttc) AS panier_moyen
FROM commandes
WHERE YEAR(date_vente) = YEAR(CURDATE())
GROUP BY DATE(date_vente);
```

#### 2. Vues dénormalisées pour les applications

```sql
-- Au lieu de faire des jointures dans l'application
CREATE VIEW v_articles_blog AS
SELECT
    a.id,
    a.titre,
    a.contenu,
    a.date_publication,
    u.nom AS auteur_nom,
    c.nom AS categorie_nom,
    COUNT(co.id) AS nb_commentaires
FROM articles a
LEFT JOIN utilisateurs u ON a.auteur_id = u.id
LEFT JOIN categories c ON a.categorie_id = c.id
LEFT JOIN commentaires co ON a.id = co.article_id
GROUP BY a.id, a.titre, a.contenu, a.date_publication, u.nom, c.nom;
```

#### 3. Vues de sécurité multi-tenant

```sql
-- Vue qui filtre automatiquement par tenant
CREATE VIEW v_tenant_data AS
SELECT
    id,
    nom,
    description,
    tenant_id
FROM data
WHERE tenant_id = (SELECT tenant_id FROM utilisateurs WHERE id = CURRENT_USER());
```

### Limitations des vues

⚠️ **Attention aux pièges courants** :

1. **Performance** : Les vues complexes peuvent être lentes, surtout avec TEMPTABLE
2. **Debugging** : Les erreurs dans les vues peuvent être difficiles à diagnostiquer
3. **Mises à jour limitées** : Toutes les vues ne sont pas updatable
4. **Pas de cache** : Les vues recalculent leurs données à chaque appel (contrairement aux vues matérialisées)
5. **Dépendances** : Modifier une table sous-jacente peut casser les vues

```sql
-- Exemple de vue non-updatable (agrégation)
CREATE VIEW v_stats AS
SELECT departement_id, COUNT(*) AS total
FROM employes
GROUP BY departement_id;

-- Cette commande échouera :
UPDATE v_stats SET total = 10 WHERE departement_id = 1;
-- Error: The target table v_stats of the UPDATE is not updatable
```

### Structure du chapitre

Ce chapitre est organisé en sections progressives :

1. **Création et gestion des vues** : Syntaxe CREATE VIEW, ALTER VIEW, DROP VIEW
2. **Vues matérialisées** : Alternatives et workarounds dans MariaDB
3. **Vues updatable** : Conditions pour pouvoir modifier les données via les vues
4. **WITH CHECK OPTION** : Garantir la cohérence des modifications
5. **Sécurité et vues** : Masquage de données et contrôle d'accès
6. **Performance des vues** : MERGE vs TEMPTABLE, optimisations
7. **Vues système** : Exploitation d'INFORMATION_SCHEMA et PERFORMANCE_SCHEMA

💡 **Conseil pédagogique** : Les vues sont un concept fondamental qui nécessite une compréhension solide du SQL. Prenez le temps d'expérimenter avec des vues simples avant de passer aux cas complexes.

---

## 🔍 Aperçu des sections

### 9.1 Création et gestion des vues

Syntaxe complète de CREATE VIEW avec toutes les options (ALGORITHM, DEFINER, SQL SECURITY), modification avec ALTER VIEW, et suppression avec DROP VIEW.

### 9.2 Vues matérialisées : Alternatives et workarounds

MariaDB ne dispose pas de vues matérialisées natives. Nous explorerons plusieurs techniques pour simuler ce comportement : tables de cache, triggers, events planifiés.

### 9.3 Vues updatable : Conditions et limitations

Découvrez les conditions strictes qui rendent une vue updatable (pas d'agrégation, pas de DISTINCT, jointures simples) et comment MariaDB détermine si une vue permet les INSERT/UPDATE/DELETE.

### 9.4 WITH CHECK OPTION

Cette clause puissante garantit que les modifications via une vue respectent toujours la clause WHERE de la vue, évitant ainsi les incohérences de données.

### 9.5 Sécurité et vues : Masquage de données

Les vues comme mécanisme de sécurité : restriction de colonnes, filtrage de lignes, isolation multi-tenant, et bonnes pratiques de gestion des privilèges.

### 9.6 Performance des vues : MERGE vs TEMPTABLE

Analyse approfondie des algorithmes de traitement des vues, impact sur les performances, comment forcer un algorithme, et stratégies d'optimisation.

### 9.7 Vues système

Exploitation des vues système pour l'administration et le monitoring : INFORMATION_SCHEMA (métadonnées), PERFORMANCE_SCHEMA (métriques de performance), mysql system tables.

---

## 💡 Conseils pratiques

### Bonnes pratiques de nommage

Adoptez une convention de nommage cohérente pour vos vues :

```sql
-- Préfixe v_ pour identifier rapidement les vues
CREATE VIEW v_clients_actifs AS ...
CREATE VIEW v_commandes_en_cours AS ...

-- Ou convention plus explicite
CREATE VIEW view_reporting_ventes_mensuelles AS ...
```

### Documentation des vues

Documentez toujours vos vues, surtout les vues complexes :

```sql
CREATE VIEW v_analyse_performance_vendeurs AS
-- Description : Vue utilisée pour le tableau de bord des managers
-- Calcule les métriques de performance des vendeurs sur les 12 derniers mois
-- Mise à jour : Décembre 2025
-- Auteur : Équipe Data
SELECT
    v.id AS vendeur_id,
    v.nom AS vendeur_nom,
    COUNT(c.id) AS nb_ventes,
    SUM(c.montant_ttc) AS ca_total,
    AVG(c.montant_ttc) AS panier_moyen
FROM vendeurs v
LEFT JOIN commandes c ON v.id = c.vendeur_id
WHERE c.date_commande >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
GROUP BY v.id, v.nom;
```

### Vérifier les dépendances avant suppression

Avant de supprimer ou modifier une vue, vérifiez qu'elle n'est pas utilisée par d'autres objets :

```sql
-- Rechercher les vues qui dépendent d'une table
SELECT
    TABLE_NAME AS vue_dependante,
    VIEW_DEFINITION
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'ma_base'
  AND VIEW_DEFINITION LIKE '%nom_table%';
```

### Tester les performances

Comparez toujours les performances d'une vue avec la requête SQL équivalente :

```sql
-- Mesurer le temps d'exécution
SET profiling = 1;

SELECT * FROM v_ma_vue WHERE condition;
SELECT * FROM table WHERE condition;  -- Équivalent sans vue

SHOW PROFILES;
SET profiling = 0;
```

---

## ⚠️ Pièges à éviter

### 1. Vues imbriquées excessives

Évitez de créer des vues basées sur d'autres vues de manière excessive :

```sql
-- Mauvaise pratique : trop d'imbrication
CREATE VIEW v_base AS SELECT * FROM table1;
CREATE VIEW v_niveau1 AS SELECT * FROM v_base WHERE ...;
CREATE VIEW v_niveau2 AS SELECT * FROM v_niveau1 WHERE ...;
CREATE VIEW v_niveau3 AS SELECT * FROM v_niveau2 WHERE ...;

-- Le serveur doit résoudre toutes les vues, impact sur les performances
```

**Limite** : MariaDB a une limite de 61 niveaux de vue imbriquées, mais en pratique 2-3 niveaux maximum est recommandé.

### 2. Vues avec SELECT *

N'utilisez pas SELECT * dans les vues de production :

```sql
-- Mauvaise pratique : fragile aux changements de schéma
CREATE VIEW v_mauvaise AS
SELECT * FROM employes;  -- Si une colonne est ajoutée, la vue change

-- Bonne pratique : spécifier les colonnes explicitement
CREATE VIEW v_bonne AS
SELECT id, nom, prenom, email, departement_id FROM employes;
```

### 3. Oublier les permissions sur les tables sous-jacentes

```sql
-- La vue est créée, mais l'utilisateur n'a pas accès aux tables
CREATE VIEW v_data AS SELECT * FROM table_sensible;
GRANT SELECT ON v_data TO 'user'@'%';

-- L'utilisateur ne pourra pas interroger la vue si :
-- - SQL SECURITY = INVOKER (par défaut)
-- - Il n'a pas SELECT sur table_sensible
```

### 4. Confondre vues et tables matérialisées

```sql
-- Une vue n'améliore PAS les performances en elle-même
CREATE VIEW v_calcul_lourd AS
SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ...;

-- Chaque SELECT sur cette vue réexécute le calcul lourd
-- Pour des calculs coûteux répétés, considérez une vraie table de cache
```

---

## 🎓 Prérequis techniques

Avant d'aborder ce chapitre, assurez-vous de maîtriser :

### SQL Avancé
- Jointures complexes (INNER, LEFT, RIGHT, CROSS)
- Sous-requêtes et requêtes imbriquées
- Fonctions d'agrégation et GROUP BY
- CTE (Common Table Expressions)

### Concepts de sécurité
- Système de privilèges MariaDB (GRANT/REVOKE)
- Utilisateurs et rôles
- SQL SECURITY (DEFINER vs INVOKER)

### Modélisation relationnelle
- Normalisation et dénormalisation
- Clés primaires et étrangères
- Relations entre tables

---

## 📊 Vocabulaire clé

Termes importants utilisés dans ce chapitre :

- **Vue (View)** : Table virtuelle définie par une requête SQL
- **Updatable View** : Vue permettant les opérations INSERT, UPDATE, DELETE
- **ALGORITHM** : Méthode de traitement de la vue (MERGE, TEMPTABLE, UNDEFINED)
- **DEFINER** : Utilisateur sous l'identité duquel la vue s'exécute
- **SQL SECURITY** : Définit qui exécute la vue (DEFINER ou INVOKER)
- **WITH CHECK OPTION** : Validation des modifications selon la définition de la vue
- **Base Table** : Table physique sous-jacente référencée par la vue
- **Materialized View** : Vue dont le résultat est stocké physiquement (non supporté nativement)

---

## ✅ Points clés à retenir

- **Les vues sont des requêtes SQL stockées**, pas des tables avec données physiques
- Elles simplifient les requêtes complexes et créent une **couche d'abstraction**
- Les vues peuvent être utilisées pour la **sécurité** (masquage de colonnes/lignes)
- MariaDB utilise deux algorithmes : **MERGE** (performant) et **TEMPTABLE** (plus lent)
- Toutes les vues ne sont **pas updatable** : dépend de la complexité de la requête
- **WITH CHECK OPTION** garantit la cohérence lors des modifications via une vue
- MariaDB **ne supporte pas les vues matérialisées** natives, mais des alternatives existent
- Les **vues système** (INFORMATION_SCHEMA, PERFORMANCE_SCHEMA) sont essentielles pour l'administration
- **Performance** : Les vues peuvent être lentes si mal conçues (attention à TEMPTABLE et vues imbriquées)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CREATE VIEW](https://mariadb.com/kb/en/create-view/) - Syntaxe complète et options
- [📖 ALTER VIEW](https://mariadb.com/kb/en/alter-view/) - Modification de vues existantes
- [📖 INFORMATION_SCHEMA](https://mariadb.com/kb/en/information-schema/) - Vue d'ensemble du schéma système
- [📖 PERFORMANCE_SCHEMA](https://mariadb.com/kb/en/performance-schema/) - Métriques et monitoring
- [📖 View Algorithms](https://mariadb.com/kb/en/view-algorithms/) - MERGE vs TEMPTABLE
- [📖 Updatable and Insertable Views](https://mariadb.com/kb/en/inserting-and-updating-with-views/) - Conditions et limitations

### Articles et tutoriels recommandés
- **"Understanding View Performance in MariaDB"** - MariaDB Blog
- **"Security Best Practices with Views"** - Percona Blog
- **"Alternatives to Materialized Views in MariaDB"** - Database Journal

### Outils complémentaires
- **HeidiSQL** : Visualisation et gestion graphique des vues
- **DBeaver** : Éditeur SQL avec support complet des vues
- **pt-visual-explain** (Percona Toolkit) : Analyse des plans d'exécution de vues

---

## ➡️ Section suivante

**[9.1 Création et gestion des vues](./01-creation-gestion-vues.md)** : Syntaxe complète de CREATE VIEW, ALTER VIEW, DROP VIEW avec toutes les options (ALGORITHM, DEFINER, SQL SECURITY), bonnes pratiques de création, et gestion du cycle de vie des vues.

Nous commencerons par maîtriser la création et la gestion basique des vues avant d'explorer les concepts avancés dans les sections suivantes.

---

**MariaDB** : Compatible 10.6+ | Optimisé pour 11.8 LTS

⏭️ [Création et gestion des vues (CREATE VIEW, ALTER VIEW)](/09-vues-et-donnees-virtuelles/01-creation-gestion-vues.md)

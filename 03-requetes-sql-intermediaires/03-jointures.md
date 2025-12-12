🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 Jointures

> **Niveau** : Intermédiaire
> **Durée estimée** : 6-8 heures (pour toute la section 3.3 avec sous-sections)
> **Prérequis** : Sections 3.1 et 3.2, compréhension des clés primaires et étrangères, maîtrise des agrégations et GROUP BY

## 🎯 Objectifs d'apprentissage

À l'issue de cette section complète sur les jointures, vous serez capable de :
- Comprendre le principe et l'utilité des jointures dans le modèle relationnel
- Identifier le type de jointure approprié selon le besoin métier
- Maîtriser la syntaxe des différents types de jointures (INNER, LEFT, RIGHT, CROSS, Self-Join)
- Joindre efficacement 2, 3, 4 tables ou plus dans une même requête
- Éviter les pièges courants (produit cartésien accidentel, performances dégradées)
- Combiner jointures et agrégations pour des analyses multi-tables complexes
- Optimiser les requêtes utilisant des jointures

---

## Introduction

Les **jointures** (ou *joins* en anglais) sont le cœur même du **modèle relationnel**. Elles permettent de combiner les données de plusieurs tables en une seule requête, créant ainsi des résultats riches et cohérents.

### Pourquoi les jointures existent-elles ?

Dans une base de données bien conçue, les données sont **normalisées** : au lieu de répéter les informations, on les organise dans des tables distinctes reliées par des **clés étrangères**.

**Exemple sans normalisation (❌ mauvais design)** :
```
Table commandes_denormalisee
+-------------+--------------+--------------+--------------+-----------------+
| id_commande | client_nom   | client_email | produit_nom  | produit_prix    |
+-------------+--------------+--------------+--------------+-----------------+
|        1001 | Alice Martin | alice@...    | Laptop       |         1299.99 |
|        1002 | Alice Martin | alice@...    | Souris       |           29.99 |
|        1003 | Bob Dupont   | bob@...      | Clavier      |           89.99 |
+-------------+--------------+--------------+--------------+-----------------+
```

**Problèmes** :
- ❌ **Redondance** : Les infos client sont répétées pour chaque commande
- ❌ **Incohérence** : Si Alice change d'email, il faut modifier plusieurs lignes
- ❌ **Anomalies** : Impossible d'avoir un client sans commande
- ❌ **Gaspillage d'espace** : Mêmes chaînes de caractères dupliquées

**Avec normalisation (✅ bon design)** :
```
Table clients                    Table commandes
+-----------+--------------+      +-------------+-----------+------------+
| id_client | nom          |      | id_commande | id_client | id_produit |
+-----------+--------------+      +-------------+-----------+------------+
|         1 | Alice Martin |      |        1001 |         1 |       2001 |
|         2 | Bob Dupont   |      |        1002 |         1 |       2002 |
+-----------+--------------+      |        1003 |         2 |       2003 |
                                  +-------------+-----------+------------+

Table produits
+------------+---------+---------+
| id_produit | nom     | prix    |
+------------+---------+---------+
|       2001 | Laptop  | 1299.99 |
|       2002 | Souris  |   29.99 |
|       2003 | Clavier |   89.99 |
+------------+---------+---------+
```

**Avantages** :
- ✅ **Pas de redondance** : Chaque info n'est stockée qu'une fois
- ✅ **Cohérence** : Modifier Alice → une seule ligne à mettre à jour
- ✅ **Flexibilité** : On peut avoir des clients sans commandes
- ✅ **Économie d'espace** : Les IDs (entiers) prennent moins de place que du texte

**Mais comment récupérer toutes les infos ensemble ?** → **Les jointures !**

### Le rôle des jointures

Les jointures permettent de **reconstruire une vue complète** en combinant les données de plusieurs tables normalisées :

```sql
-- Récupérer les infos complètes d'une commande
SELECT
    cmd.id_commande,
    cl.nom AS client_nom,
    cl.email AS client_email,
    p.nom AS produit_nom,
    p.prix AS produit_prix
FROM commandes cmd
JOIN clients cl ON cmd.id_client = cl.id_client
JOIN produits p ON cmd.id_produit = p.id_produit;
```

Résultat : nous retrouvons la même vue que la table dénormalisée, mais les données source restent proprement organisées !

---

## Le modèle relationnel : rappel des concepts

### Clés primaires (Primary Keys)

Une **clé primaire** (PK) identifie de manière **unique** chaque ligne d'une table.

```sql
CREATE TABLE clients (
    id_client INT PRIMARY KEY AUTO_INCREMENT,  -- Clé primaire
    nom VARCHAR(100),
    email VARCHAR(150) UNIQUE
);
```

**Propriétés** :
- ✅ **Unique** : Pas deux lignes avec la même valeur
- ✅ **Non NULL** : Ne peut jamais être NULL
- ✅ **Immuable** : Ne devrait pas changer dans le temps
- ✅ **Simple ou composite** : Un champ (id) ou plusieurs (rare)

### Clés étrangères (Foreign Keys)

Une **clé étrangère** (FK) est une colonne qui **référence** la clé primaire d'une autre table.

```sql
CREATE TABLE commandes (
    id_commande INT PRIMARY KEY AUTO_INCREMENT,
    id_client INT NOT NULL,
    date_commande DATETIME,
    montant_total DECIMAL(10, 2),

    -- Clé étrangère vers la table clients
    FOREIGN KEY (id_client) REFERENCES clients(id_client)
);
```

**Rôle** :
- 🔗 **Établit une relation** entre deux tables
- ✅ **Garantit l'intégrité référentielle** : on ne peut pas avoir `id_client = 999` si ce client n'existe pas
- 🚫 **Empêche les suppressions incohérentes** : impossible de supprimer un client ayant des commandes (selon la config)

### Types de relations

| Type de relation | Description | Exemple |
|------------------|-------------|---------|
| **Un-à-plusieurs (1:N)** | Une ligne dans A liée à plusieurs dans B | Un client → plusieurs commandes |
| **Plusieurs-à-plusieurs (N:M)** | Plusieurs lignes dans A liées à plusieurs dans B | Commandes ↔ Produits (via table de liaison) |
| **Un-à-un (1:1)** | Une ligne dans A liée à une seule dans B | Rare : Utilisateur ↔ Profil détaillé |

### Schéma de référence pour cette section

Nous utiliserons le schéma e-commerce défini dans le chapitre :

```
┌─────────────┐         ┌──────────────┐         ┌─────────────────┐
│  clients    │         │  commandes   │         │ details_commande│
├─────────────┤         ├──────────────┤         ├─────────────────┤
│ id_client PK│────1:N──│ id_commande  │────1:N──│ id_detail       │
│ nom         │         │ id_client FK │         │ id_commande FK  │
│ email       │         │ date_commande│         │ id_produit FK   │
│ ville       │         │ statut       │         │ quantite        │
│ pays        │         │ montant_total│         │ prix_unitaire   │
└─────────────┘         └──────────────┘         └─────────────────┘
                                                          │
                                                          │
                        ┌─────────────┐                   │
                        │  produits   │                   │
                        ├─────────────┤                   │
                        │ id_produit  │───────────────────┘
                        │ nom_produit │
                        │ categorie   │
                        │ prix_unit.  │
                        │ stock       │
                        └─────────────┘
```

**Relations** :
- Un client (1) peut avoir plusieurs commandes (N) → `clients.id_client ← commandes.id_client`
- Une commande (1) peut avoir plusieurs détails (N) → `commandes.id_commande ← details_commande.id_commande`
- Un produit (1) peut apparaître dans plusieurs détails (N) → `produits.id_produit ← details_commande.id_produit`

---

## Vue d'ensemble des types de jointures

SQL propose **cinq types principaux** de jointures, chacun avec un comportement spécifique.

### Résumé visuel des jointures

```
Deux ensembles A et B :

INNER JOIN (intersection)        LEFT JOIN (tout A + correspondances B)
┌───────┐                         ┌───────┐
│   A   │                         │   A   │
│ ┌─────┼─────┐                   │ ┌─────┼─────┐
│ │ ∩∩∩ │  B  │                   │ │ AAA │  B  │
│ └─────┼─────┘                   │ └─────┼─────┘
│       │                         │       │
└───────┘                         └───────┘
Résultat : ∩∩∩                    Résultat : AAA + ∩∩∩


RIGHT JOIN (correspondances A + tout B)    CROSS JOIN (produit cartésien)
      ┌───────┐                            ┌───────┐
      │   A   │                            │   A   │
┌─────┼─────┐ │                      ┌─────┼─────┐ │
│  B  │ BBB │ │                      │ XXXX│XXXXX│ │
└─────┼─────┘ │                      └─────┼─────┘ │
      │       │                            │       │
      └───────┘                            └───────┘
Résultat : ∩∩∩ + BBB                Résultat : Toutes combinaisons A×B
```

### Tableau comparatif des jointures

| Type | Syntaxe SQL | Lignes retournées | Cas d'usage typique |
|------|-------------|-------------------|---------------------|
| **INNER JOIN** | `A INNER JOIN B ON ...` | Seulement les correspondances | Combiner données existantes des deux côtés |
| **LEFT JOIN** | `A LEFT JOIN B ON ...` | Tout A + correspondances B (NULL si pas de match) | Trouver ce qui manque dans B, analyses complètes sur A |
| **RIGHT JOIN** | `A RIGHT JOIN B ON ...` | Correspondances A + tout B (NULL si pas de match) | Symétrique de LEFT (peu utilisé, on préfère LEFT) |
| **CROSS JOIN** | `A CROSS JOIN B` | Toutes les combinaisons (A × B) | Matrices, calendriers, combinatoires |
| **SELF JOIN** | `A JOIN A AS A2 ON ...` | Lignes de A liées à d'autres lignes de A | Hiérarchies, comparaisons intra-table |

### Les 5 types en détail (aperçu)

Nous allons explorer chaque type dans des sous-sections dédiées. Voici un aperçu :

#### 1. INNER JOIN (Section 3.3.1)
**Le plus courant** : ne retourne que les lignes où il y a correspondance des deux côtés.

```sql
-- Clients AVEC commandes
SELECT c.nom, cmd.id_commande
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client;
```

**Caractéristique** : Les clients sans commande n'apparaissent **pas** dans le résultat.

#### 2. LEFT JOIN (Section 3.3.2)
**Préserve toutes les lignes de gauche** : retourne tout de la table de gauche, avec NULL si pas de correspondance à droite.

```sql
-- TOUS les clients, avec ou sans commandes
SELECT c.nom, cmd.id_commande
FROM clients c
LEFT JOIN commandes cmd ON c.id_client = cmd.id_client;
```

**Caractéristique** : Les clients sans commande apparaissent avec `cmd.id_commande = NULL`.

#### 3. RIGHT JOIN (Section 3.3.2)
**Symétrique du LEFT JOIN** : préserve toutes les lignes de droite.

```sql
-- Toutes les commandes, même si le client n'existe plus (rare)
SELECT c.nom, cmd.id_commande
FROM clients c
RIGHT JOIN commandes cmd ON c.id_client = cmd.id_client;
```

💡 **Note** : RIGHT JOIN est rarement utilisé en pratique. On préfère inverser l'ordre des tables et utiliser LEFT JOIN, qui est plus intuitif.

#### 4. CROSS JOIN (Section 3.3.3)
**Produit cartésien** : combine chaque ligne de A avec chaque ligne de B.

```sql
-- Toutes les combinaisons possibles
SELECT c.nom, p.nom_produit
FROM clients c
CROSS JOIN produits p;
```

**Attention** : Si `clients` a 1000 lignes et `produits` 500, le résultat aura **500 000 lignes** !

**Cas d'usage** : Matrices de préférences, calendriers multi-ressources, tests combinatoires.

#### 5. SELF JOIN (Section 3.3.4)
**Joindre une table à elle-même** : utile pour les hiérarchies ou comparaisons.

```sql
-- Employés et leurs managers (même table)
SELECT
    emp.nom AS employe,
    mgr.nom AS manager
FROM employes emp
JOIN employes mgr ON emp.id_manager = mgr.id_employe;
```

**Cas d'usage** : Organigrammes, structures hiérarchiques, comparaisons de lignes au sein d'une même table.

---

## Syntaxe générale des jointures

### Syntaxe ANSI-92 (moderne, recommandée)

```sql
SELECT colonnes
FROM table1 alias1
[INNER | LEFT | RIGHT | CROSS] JOIN table2 alias2
    ON alias1.colonne = alias2.colonne
[WHERE conditions]
[GROUP BY colonnes]
[HAVING conditions_agregees]
[ORDER BY colonnes];
```

**Composants** :
- **FROM table1** : Table de départ (à gauche)
- **JOIN table2** : Table à joindre (à droite)
- **ON condition** : Critère de jointure (généralement égalité entre clés)
- **WHERE** : Filtrage après jointure
- **Alias** (alias1, alias2) : Raccourcis pour qualifier les colonnes

### Exemple simple

```sql
-- Joindre clients et commandes
SELECT
    c.nom,              -- Vient de clients (alias c)
    cmd.id_commande,    -- Vient de commandes (alias cmd)
    cmd.montant_total
FROM clients c          -- Table gauche avec alias c
INNER JOIN commandes cmd    -- Table droite avec alias cmd
    ON c.id_client = cmd.id_client  -- Condition de jointure
WHERE cmd.statut = 'livrée'    -- Filtrage après jointure
ORDER BY cmd.montant_total DESC;
```

### Ancienne syntaxe (à éviter)

Avant SQL-92, on utilisait la syntaxe à virgule :

```sql
-- ❌ ANCIENNE SYNTAXE (éviter)
SELECT c.nom, cmd.id_commande
FROM clients c, commandes cmd
WHERE c.id_client = cmd.id_client;
```

**Pourquoi éviter cette syntaxe ?**
- ❌ Moins lisible (jointure mélangée avec filtrage dans WHERE)
- ❌ Risque de produit cartésien si on oublie la condition de jointure
- ❌ Ne permet pas de faire des LEFT/RIGHT JOIN facilement
- ❌ Confusion entre conditions de jointure et de filtrage

✅ **Recommandation** : Utilisez **toujours** la syntaxe ANSI-92 avec `JOIN ... ON`.

---

## Les alias : indispensables pour les jointures

Quand on joint plusieurs tables, les **alias** deviennent essentiels pour :
1. **Éviter les ambiguïtés** sur les noms de colonnes
2. **Rendre le code plus lisible**
3. **Permettre les self-joins** (joindre une table à elle-même)

### Pourquoi les alias sont nécessaires

```sql
-- ❌ AMBIGUË : id_client existe dans les deux tables
SELECT id_client, nom
FROM clients
INNER JOIN commandes ON clients.id_client = commandes.id_client;
-- Erreur : "Column 'id_client' in field list is ambiguous"
```

```sql
-- ✅ CLAIR : On qualifie avec l'alias
SELECT c.id_client, c.nom, cmd.id_commande
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client;
```

### Conventions de nommage des alias

Il existe plusieurs styles, choisissez celui qui vous convient :

```sql
-- Style 1 : Première lettre ou début du nom
FROM clients c
JOIN commandes cmd

-- Style 2 : Abréviations cohérentes
FROM clients cli
JOIN commandes com

-- Style 3 : Noms descriptifs (pour requêtes complexes)
FROM clients client
JOIN commandes commande
```

💡 **Conseil** : Pour les requêtes simples, des alias courts (1-3 lettres) suffisent. Pour les requêtes complexes avec 5+ tables, utilisez des alias plus explicites.

---

## Condition de jointure : ON vs WHERE

### Clause ON : Condition de jointure

La clause **ON** définit **comment** les tables sont reliées :

```sql
FROM clients c
INNER JOIN commandes cmd
    ON c.id_client = cmd.id_client  -- Relie clients et commandes
```

**Rôle** : Spécifie la **correspondance** entre les lignes des deux tables.

### Clause WHERE : Filtrage après jointure

La clause **WHERE** filtre les **résultats** après que la jointure a été effectuée :

```sql
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client
WHERE cmd.montant_total > 100  -- Filtre les résultats
```

### ON avec conditions multiples

Vous pouvez avoir plusieurs conditions dans ON :

```sql
FROM commandes cmd
JOIN details_commande dc
    ON cmd.id_commande = dc.id_commande
    AND dc.quantite > 0  -- Condition supplémentaire dans la jointure
```

⚠️ **Différence subtile** : Pour INNER JOIN, mettre une condition dans ON ou WHERE donne souvent le même résultat. Mais pour LEFT/RIGHT JOIN, l'emplacement **change complètement** le comportement (nous verrons cela en section 3.3.2).

---

## Jointures multiples : Combiner 3 tables ou plus

Les requêtes réelles utilisent souvent plusieurs jointures pour croiser 3, 4, 5 tables ou plus.

### Syntaxe pour jointures multiples

```sql
SELECT colonnes
FROM table1
JOIN table2 ON condition1
JOIN table3 ON condition2
JOIN table4 ON condition3
-- etc.
```

Chaque `JOIN` s'ajoute séquentiellement.

### Exemple : 3 tables

**Question métier** : *Obtenir le détail complet des commandes avec nom client et nom produit*

```sql
-- Joindre clients, commandes et produits
SELECT
    c.nom AS client,
    cmd.id_commande,
    cmd.date_commande,
    p.nom_produit,
    dc.quantite,
    dc.prix_unitaire
FROM clients c
INNER JOIN commandes cmd
    ON c.id_client = cmd.id_client
INNER JOIN details_commande dc
    ON cmd.id_commande = dc.id_commande
INNER JOIN produits p
    ON dc.id_produit = p.id_produit
WHERE cmd.statut = 'livrée'
ORDER BY cmd.date_commande DESC;
```

**Flux de jointures** :
1. `clients` JOIN `commandes` → lignes avec infos client + commande
2. Résultat précédent JOIN `details_commande` → ajout des lignes de détail
3. Résultat précédent JOIN `produits` → ajout des infos produit
4. Filtrage avec WHERE
5. Tri avec ORDER BY

### Visualisation du processus

```
Étape 1 : clients (1247 lignes) JOIN commandes (12463 lignes)
    → Résultat : 12463 lignes (une par commande avec info client)

Étape 2 : Résultat (12463) JOIN details_commande (45892 lignes)
    → Résultat : 45892 lignes (une par produit dans chaque commande)

Étape 3 : Résultat (45892) JOIN produits (726 lignes)
    → Résultat : 45892 lignes (même nombre, mais avec nom produit)

Étape 4 : WHERE filtre → ~41000 lignes (statut = 'livrée')
```

💡 **Principe** : Chaque jointure **ajoute** des colonnes et peut **changer le nombre de lignes** selon le type de relation (1:N, N:M).

---

## Ordre des jointures et optimisation

### L'optimiseur de requêtes

MariaDB possède un **optimiseur** qui décide de l'ordre réel d'exécution des jointures, indépendamment de l'ordre dans votre requête SQL.

```sql
-- Vous écrivez ceci :
FROM clients c
JOIN commandes cmd ON ...
JOIN produits p ON ...

-- MariaDB peut décider d'exécuter :
FROM produits p
JOIN commandes cmd ON ...
JOIN clients c ON ...
```

**Pourquoi ?** Pour minimiser le nombre de lignes à traiter à chaque étape.

### Facteurs pris en compte par l'optimiseur

1. **Nombre de lignes** dans chaque table (cardinalité)
2. **Présence d'index** sur les colonnes de jointure
3. **Sélectivité des filtres** WHERE
4. **Statistiques** des tables (mises à jour avec ANALYZE TABLE)

### Vérifier l'ordre d'exécution avec EXPLAIN

```sql
EXPLAIN
SELECT c.nom, cmd.id_commande, p.nom_produit
FROM clients c
JOIN commandes cmd ON c.id_client = cmd.id_client
JOIN produits p ON cmd.id_produit = p.id_produit;
```

Le résultat EXPLAIN montre :
- **table** : L'ordre réel d'accès aux tables
- **type** : Type d'accès (ALL=scan complet, ref=utilise index, etc.)
- **key** : Index utilisé
- **rows** : Nombre estimé de lignes examinées

💡 **Conseil** : Prenez l'habitude d'analyser vos jointures complexes avec EXPLAIN, surtout en production.

---

## Performance des jointures : Bonnes pratiques

### ✅ Bonne pratique 1 : Index sur les colonnes de jointure

**Règle d'or** : Créez des index sur **toutes les colonnes** utilisées dans les conditions ON.

```sql
-- Index sur les clés étrangères
CREATE INDEX idx_commandes_client ON commandes(id_client);
CREATE INDEX idx_details_commande ON details_commande(id_commande);
CREATE INDEX idx_details_produit ON details_commande(id_produit);
```

**Impact** : Transforme un scan complet (lent) en recherche indexée (rapide).

Sans index :
```
EXPLAIN → type: ALL, rows: 12463 (scan complet)
```

Avec index :
```
EXPLAIN → type: ref, rows: 3 (accès direct via index)
```

### ✅ Bonne pratique 2 : Filtrer avant de joindre

Si possible, filtrez les données **avant** la jointure avec WHERE :

```sql
-- ✅ BON : Filtre AVANT de joindre details_commande
SELECT ...
FROM commandes cmd
JOIN details_commande dc ON cmd.id_commande = dc.id_commande
WHERE cmd.date_commande >= '2025-01-01'  -- Réduit cmd avant jointure
```

MariaDB optimise souvent automatiquement, mais une WHERE bien placée aide l'optimiseur.

### ✅ Bonne pratique 3 : Éviter les fonctions sur les colonnes de jointure

```sql
-- ❌ LENT : Fonction sur colonne de jointure (empêche l'utilisation d'index)
FROM clients c
JOIN commandes cmd ON LOWER(c.email) = LOWER(cmd.email_client)

-- ✅ RAPIDE : Jointure directe
FROM clients c
JOIN commandes cmd ON c.id_client = cmd.id_client
```

### ✅ Bonne pratique 4 : Limiter le nombre de colonnes sélectionnées

```sql
-- ❌ MAUVAIS : Sélectionne tout (transfert de données inutiles)
SELECT *
FROM clients c
JOIN commandes cmd ON c.id_client = cmd.id_client;

-- ✅ BON : Ne sélectionne que ce qui est nécessaire
SELECT c.nom, cmd.id_commande, cmd.montant_total
FROM clients c
JOIN commandes cmd ON c.id_client = cmd.id_client;
```

---

## Pièges courants à éviter

### Piège 1 : Le produit cartésien accidentel

**Erreur** : Oublier la condition ON, ou utiliser une mauvaise condition.

```sql
-- ❌ DANGER : Produit cartésien !
SELECT c.nom, cmd.id_commande
FROM clients c
CROSS JOIN commandes cmd;  -- Toutes les combinaisons !

-- Si 1000 clients et 10000 commandes → 10 000 000 lignes !
```

**Symptômes** :
- Requête très lente
- Nombre de lignes résultat = Lignes_A × Lignes_B
- MariaDB peut afficher un warning

**Solution** : Toujours vérifier la condition ON.

### Piège 2 : Ambiguïté sur les noms de colonnes

```sql
-- ❌ ERREUR si id existe dans les deux tables
SELECT id, nom
FROM clients
JOIN commandes ON clients.id = commandes.client_id;
-- Erreur : "Column 'id' in field list is ambiguous"
```

**Solution** : Toujours qualifier avec l'alias.

```sql
-- ✅ CORRECT
SELECT c.id_client, c.nom
FROM clients c
JOIN commandes cmd ON c.id_client = cmd.id_client;
```

### Piège 3 : Mauvaise compréhension de LEFT vs INNER JOIN

```sql
-- INNER JOIN : Uniquement clients avec commandes
SELECT c.nom, COUNT(cmd.id_commande)
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client
GROUP BY c.nom;
-- Les clients SANS commande n'apparaissent PAS

-- LEFT JOIN : TOUS les clients, avec ou sans commandes
SELECT c.nom, COUNT(cmd.id_commande)
FROM clients c
LEFT JOIN commandes cmd ON c.id_client = cmd.id_client
GROUP BY c.nom;
-- Les clients SANS commande apparaissent avec COUNT = 0
```

⚠️ **Attention** : Le choix entre INNER et LEFT change radicalement le résultat !

### Piège 4 : Performances dégradées sans index

Sans index sur les colonnes de jointure, les performances s'effondrent sur de grandes tables :

```
Avec index :    0.05 secondes  ✅
Sans index :    45.2 secondes  ❌ (900x plus lent !)
```

**Solution** : EXPLAIN + CREATE INDEX.

---

## Jointures et agrégations : Combinaison puissante

Les jointures deviennent encore plus utiles combinées avec GROUP BY et les agrégations.

### Exemple conceptuel

**Question métier** : *Quel est le CA par catégorie de produit ?*

```sql
-- Besoin de joindre 3 tables puis agréger
SELECT
    p.categorie,
    COUNT(DISTINCT cmd.id_commande) AS nb_commandes,
    SUM(dc.quantite) AS quantite_totale,
    SUM(dc.quantite * dc.prix_unitaire) AS ca_categorie
FROM produits p
JOIN details_commande dc ON p.id_produit = dc.id_produit
JOIN commandes cmd ON dc.id_commande = cmd.id_commande
WHERE cmd.statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY p.categorie
ORDER BY ca_categorie DESC;
```

**Flux** :
1. Jointure des 3 tables
2. Filtrage WHERE (statut)
3. Regroupement par catégorie
4. Calcul des agrégations pour chaque groupe

Nous explorerons cela en détail dans les sous-sections spécifiques.

---

## Plan de la section 3.3

Cette introduction vous a présenté les concepts généraux des jointures. Les sous-sections suivantes détailleront chaque type :

### 📑 Sous-sections à venir

| Section | Titre | Concepts couverts |
|---------|-------|-------------------|
| **3.3.1** | INNER JOIN | Intersection, syntaxe, exemples multi-tables, agrégations |
| **3.3.2** | LEFT/RIGHT JOIN | Jointures externes, NULL, détection de manquants, différences WHERE/ON |
| **3.3.3** | CROSS JOIN | Produit cartésien, matrices, cas d'usage, performance |
| **3.3.4** | Self-Join | Hiérarchies, comparaisons intra-table, organigrammes, exemples avancés |

Chaque sous-section contiendra :
- ✅ Explication détaillée du type de jointure
- ✅ Syntaxe et variantes
- ✅ 5-10 exemples progressifs commentés
- ✅ Cas d'usage réels en production
- ✅ Pièges spécifiques et solutions
- ✅ Optimisations et bonnes pratiques

---

## Préparation pour les exemples

### Données de test recommandées

Pour suivre les exemples des sous-sections, assurez-vous d'avoir le schéma e-commerce avec quelques données :

```sql
-- Base minimale pour tester les jointures
CREATE DATABASE IF NOT EXISTS test_jointures;
USE test_jointures;

-- Tables (schéma simplifié)
CREATE TABLE clients (
    id_client INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100),
    email VARCHAR(150),
    ville VARCHAR(100),
    pays VARCHAR(50)
);

CREATE TABLE commandes (
    id_commande INT PRIMARY KEY AUTO_INCREMENT,
    id_client INT,
    date_commande DATETIME DEFAULT CURRENT_TIMESTAMP,
    montant_total DECIMAL(10,2),
    statut ENUM('en_attente', 'confirmée', 'expédiée', 'livrée', 'annulée'),
    FOREIGN KEY (id_client) REFERENCES clients(id_client)
);

CREATE TABLE produits (
    id_produit INT PRIMARY KEY AUTO_INCREMENT,
    nom_produit VARCHAR(200),
    categorie VARCHAR(50),
    prix_unitaire DECIMAL(10,2)
);

CREATE TABLE details_commande (
    id_detail INT PRIMARY KEY AUTO_INCREMENT,
    id_commande INT,
    id_produit INT,
    quantite INT,
    prix_unitaire DECIMAL(10,2),
    FOREIGN KEY (id_commande) REFERENCES commandes(id_commande),
    FOREIGN KEY (id_produit) REFERENCES produits(id_produit)
);

-- Quelques données de test
INSERT INTO clients (nom, email, ville, pays) VALUES
('Alice Martin', 'alice@email.com', 'Paris', 'France'),
('Bob Dupont', 'bob@email.com', 'Lyon', 'France'),
('Charlie Durand', 'charlie@email.com', NULL, 'Belgique');

INSERT INTO produits (nom_produit, categorie, prix_unitaire) VALUES
('Laptop Pro', 'Électronique', 1299.99),
('Souris sans fil', 'Électronique', 29.99),
('Clavier mécanique', 'Électronique', 89.99);

INSERT INTO commandes (id_client, montant_total, statut) VALUES
(1, 1329.98, 'livrée'),
(1, 29.99, 'livrée'),
(2, 89.99, 'confirmée');
-- Client 3 (Charlie) n'a AUCUNE commande → Important pour tester LEFT JOIN

INSERT INTO details_commande (id_commande, id_produit, quantite, prix_unitaire) VALUES
(1, 1, 1, 1299.99),
(1, 2, 1, 29.99),
(2, 2, 1, 29.99),
(3, 3, 1, 89.99);
```

💡 **Point important** : Charlie (id_client = 3) n'a **aucune commande**. Cela nous permettra de démontrer la différence entre INNER JOIN (il n'apparaîtra pas) et LEFT JOIN (il apparaîtra avec NULL).

---

## ✅ Points clés à retenir

1. **Les jointures combinent des tables normalisées** – permettent de reconstruire des vues complètes à partir de données fragmentées

2. **Clés étrangères relient les tables** – créent les relations dans le modèle relationnel (1:N, N:M, 1:1)

3. **5 types principaux de jointures** – INNER (intersection), LEFT/RIGHT (externes), CROSS (cartésien), SELF (même table)

4. **Syntaxe ANSI-92 recommandée** – `FROM A JOIN B ON condition` plus claire que l'ancienne syntaxe à virgule

5. **ON définit la jointure, WHERE filtre** – distinction critique, surtout pour LEFT/RIGHT JOIN

6. **Alias obligatoires en pratique** – évitent les ambiguïtés et rendent le code lisible

7. **Index sur colonnes de jointure = performance** – différence entre millisecondes et secondes

8. **L'optimiseur réordonne les jointures** – MariaDB choisit le meilleur ordre d'exécution automatiquement

9. **Vérifier avec EXPLAIN** – toujours analyser les jointures complexes pour détecter les problèmes

10. **Produit cartésien = danger** – oublier ON peut créer des millions de lignes accidentellement

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 JOIN Syntax](https://mariadb.com/kb/en/join-syntax/) – Syntaxe complète et comportements
- [📖 EXPLAIN Output](https://mariadb.com/kb/en/explain/) – Comprendre les plans d'exécution
- [📖 Foreign Keys](https://mariadb.com/kb/en/foreign-keys/) – Intégrité référentielle

### Articles approfondis
- [Visual Representation of SQL Joins](https://blog.codinghorror.com/a-visual-explanation-of-sql-joins/) – Diagrammes de Venn explicatifs
- [SQL Joins Explained](https://modern-sql.com/feature/join) – Guide moderne et complet
- [Join Order Optimization](https://use-the-index-luke.com/sql/join) – Optimisation avancée

### Outils de visualisation
- [SQL Joins Visualizer](https://sql-joins.leopard.in.ua/) – Outil interactif pour comprendre les jointures
- [Explain Analyzer](https://mariadb.org/explain-analyzer/) – Analyser les plans EXPLAIN

---

## ➡️ Section suivante

**[3.3.1 INNER JOIN : Intersection](./03.1-inner-join.md)**

Dans la première sous-section, nous plongerons dans **INNER JOIN**, le type de jointure le plus utilisé. Vous apprendrez :
- La syntaxe complète et ses variantes
- Comment joindre 2, 3, 4 tables et plus
- La différence entre équi-join et non-équi-join
- Comment combiner INNER JOIN avec GROUP BY pour des analyses puissantes
- Les optimisations spécifiques à INNER JOIN
- 10+ exemples progressifs du simple au complexe

INNER JOIN est la base de toutes les requêtes multi-tables – maîtrisez-le et vous débloquerez 80% des besoins métier ! 🔗

---


⏭️ [INNER JOIN : Intersection](/03-requetes-sql-intermediaires/03.1-inner-join.md)

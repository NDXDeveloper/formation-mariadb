🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 3.1 à 3.4, compréhension des ensembles mathématiques

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre les opérateurs ensemblistes et leur rôle
- Maîtriser UNION et UNION ALL pour fusionner des résultats
- Utiliser INTERSECT pour trouver les éléments communs
- Appliquer EXCEPT pour calculer les différences
- Différencier les opérateurs ensemblistes des jointures
- Respecter les règles de compatibilité des colonnes
- Optimiser les performances des opérations ensemblistes
- Résoudre des problèmes de consolidation et comparaison de données

---

## Introduction

Les **opérateurs ensemblistes** permettent de **combiner les résultats de plusieurs requêtes SELECT** en un seul ensemble de résultats. Ils opèrent sur des ensembles de lignes, à la manière des opérations mathématiques sur les ensembles.

### Les trois opérateurs principaux

```
Ensemble A          Ensemble B

UNION : A ∪ B (tous les éléments)
┌─────────┐
│    A    │
│  ┌──────┼──────┐
│  │ AAA  │  B   │
│  └──────┼──────┘
│         │
└─────────┘
Résultat : A + B (sans doublons)

INTERSECT : A ∩ B (éléments communs)
      ┌─────────┐
      │    A    │
┌─────┼──────┐  │
│  B  │ ∩∩∩  │  │
└─────┼──────┘  │
      │         │
      └─────────┘
Résultat : Seulement les éléments présents dans A ET B

EXCEPT : A \ B (éléments de A non dans B)
┌─────────┐
│    A    │
│  ┌──────┼──────┐
│  │ AAA  │  B   │
│  └──────┼──────┘
│         │
└─────────┘
Résultat : Éléments de A qui ne sont PAS dans B
```

### Caractéristiques communes

| Caractéristique | Description |
|-----------------|-------------|
| **Orientation verticale** | Empile les lignes (vs jointures qui ajoutent des colonnes) |
| **Même structure** | Même nombre de colonnes, types compatibles |
| **Ordre des colonnes** | Important (1ère colonne de A avec 1ère de B, etc.) |
| **Noms de colonnes** | Proviennent de la première requête |
| **ORDER BY** | Uniquement à la fin, après tous les opérateurs |

### UNION vs Jointures : Différences fondamentales

| Critère | Opérateurs ensemblistes | Jointures |
|---------|-------------------------|-----------|
| **Direction** | Vertical (ajoute des lignes) | Horizontal (ajoute des colonnes) |
| **Tables** | Peuvent être différentes | Généralement liées par clés |
| **Objectif** | Consolider, fusionner | Croiser, combiner |
| **Résultat** | Union de résultats | Produit ou correspondances |
| **Exemple** | Clients FR + Clients BE | Clients + Commandes |

---

## UNION : Fusionner des résultats

**UNION** combine les résultats de deux ou plusieurs SELECT et **élimine les doublons**.

### Syntaxe

```sql
SELECT colonnes FROM table1
UNION
SELECT colonnes FROM table2
[UNION SELECT colonnes FROM table3]
[ORDER BY colonnes];
```

### Règles strictes

1. ✅ **Même nombre de colonnes** dans chaque SELECT
2. ✅ **Types de données compatibles** (même position)
3. ✅ **Ordre important** : colonne 1 de A avec colonne 1 de B
4. ✅ **Noms de colonnes** : Ceux du premier SELECT sont utilisés
5. ✅ **ORDER BY** : Uniquement à la fin (après tous les UNION)

### Exemple 1 : UNION simple

**Question métier** : *Liste consolidée des clients français et belges*

```sql
-- Clients de France
SELECT
    nom,
    email,
    ville,
    'France' AS pays_origine
FROM clients
WHERE pays = 'France'

UNION

-- Clients de Belgique
SELECT
    nom,
    email,
    ville,
    'Belgique' AS pays_origine
FROM clients
WHERE pays = 'Belgique'

ORDER BY nom;
```

**Résultat attendu** :
```
+----------------+-------------------+-------------+---------------+
| nom            | email             | ville       | pays_origine  |
+----------------+-------------------+-------------+---------------+
| Alice Martin   | alice@email.com   | Paris       | France        |
| Bob Dupont     | bob@email.com     | Lyon        | France        |
| Charlie Durand | charlie@email.com | Bruxelles   | Belgique      |
| Diana Lopez    | diana@email.com   | Liège       | Belgique      |
+----------------+-------------------+-------------+---------------+
```

**Explication** :
- Deux SELECT distincts avec **4 colonnes chacun**
- Types compatibles : VARCHAR, VARCHAR, VARCHAR, VARCHAR
- Résultat trié par `nom` (ORDER BY à la fin)
- Les doublons seraient éliminés (si un client apparaissait dans les deux)

### Exemple 2 : UNION avec différentes tables

**Question métier** : *Liste complète de tous les contacts (clients + fournisseurs)*

```sql
-- Contacts clients
SELECT
    'Client' AS type_contact,
    nom AS nom_complet,
    email,
    telephone
FROM clients

UNION

-- Contacts fournisseurs
SELECT
    'Fournisseur' AS type_contact,
    nom_societe AS nom_complet,
    email_contact AS email,
    telephone_contact AS telephone
FROM fournisseurs

ORDER BY type_contact, nom_complet;
```

**Résultat attendu** :
```
+----------------+----------------------+----------------------+----------------+
| type_contact   | nom_complet          | email                | telephone      |
+----------------+----------------------+----------------------+----------------+
| Client         | Alice Martin         | alice@email.com      | 0601020304     |
| Client         | Bob Dupont           | bob@email.com        | 0605060708     |
| Fournisseur    | TechSupply SA        | contact@tech.com     | 0142847593     |
| Fournisseur    | GlobalParts Inc      | info@global.com      | 0147382947     |
+----------------+----------------------+----------------------+----------------+
```

**Cas d'usage** :
- Annuaires consolidés
- Listes de diffusion
- Exports pour CRM
- Analyses transverses

### Exemple 3 : UNION multiple (3+ requêtes)

**Question métier** : *Tous les événements d'un client (inscriptions, commandes, contacts)*

```sql
-- Inscription
SELECT
    date_inscription AS date_evenement,
    'Inscription' AS type_evenement,
    CONCAT('Nouveau client : ', nom) AS description
FROM clients
WHERE id_client = 1847

UNION

-- Commandes
SELECT
    date_commande AS date_evenement,
    'Commande' AS type_evenement,
    CONCAT('Commande #', id_commande, ' - ', montant_total, '€') AS description
FROM commandes
WHERE id_client = 1847

UNION

-- Contacts support
SELECT
    date_contact AS date_evenement,
    'Support' AS type_evenement,
    CONCAT('Ticket #', id_ticket, ' : ', sujet) AS description
FROM tickets_support
WHERE id_client = 1847

ORDER BY date_evenement DESC;
```

**Résultat attendu** :
```
+---------------------+-----------------+------------------------------------+
| date_evenement      | type_evenement  | description                        |
+---------------------+-----------------+------------------------------------+
| 2025-12-10 14:23:00 | Commande        | Commande #1847 - 289.95€           |
| 2025-11-05 09:15:00 | Support         | Ticket #847 : Retard de livraison  |
| 2025-10-20 18:42:00 | Commande        | Commande #1723 - 149.99€           |
| 2024-02-15 10:23:00 | Inscription     | Nouveau client : Alice Martin      |
+---------------------+-----------------+------------------------------------+
```

**Application** :
- Timeline client (CRM)
- Historique unifié
- Analyses comportementales
- Rapports d'activité

---

## UNION ALL : Garder tous les doublons

**UNION ALL** fusionne les résultats mais **conserve tous les doublons**.

### Différence UNION vs UNION ALL

```sql
-- UNION : Élimine les doublons (plus lent)
SELECT pays FROM clients WHERE ville = 'Paris'
UNION
SELECT pays FROM clients WHERE ville = 'Lyon';
-- Si 100 clients à Paris et 80 à Lyon (tous France) → 1 ligne

-- UNION ALL : Garde les doublons (plus rapide)
SELECT pays FROM clients WHERE ville = 'Paris'
UNION ALL
SELECT pays FROM clients WHERE ville = 'Lyon';
-- → 180 lignes
```

### Exemple 4 : UNION ALL pour agrégations

**Question métier** : *CA total et nombre de transactions par source*

```sql
-- Consolidation ventes online + magasin
SELECT
    'Online' AS canal,
    COUNT(*) AS nb_transactions,
    SUM(montant) AS ca_total
FROM ventes_online
WHERE date_vente >= '2025-01-01'

UNION ALL

SELECT
    'Magasin' AS canal,
    COUNT(*) AS nb_transactions,
    SUM(montant) AS ca_total
FROM ventes_magasin
WHERE date_vente >= '2025-01-01'

UNION ALL

-- Total global
SELECT
    'TOTAL' AS canal,
    (SELECT COUNT(*) FROM ventes_online WHERE date_vente >= '2025-01-01') +
    (SELECT COUNT(*) FROM ventes_magasin WHERE date_vente >= '2025-01-01') AS nb_transactions,
    (SELECT SUM(montant) FROM ventes_online WHERE date_vente >= '2025-01-01') +
    (SELECT SUM(montant) FROM ventes_magasin WHERE date_vente >= '2025-01-01') AS ca_total;
```

**Résultat attendu** :
```
+----------+------------------+-----------+
| canal    | nb_transactions  | ca_total  |
+----------+------------------+-----------+
| Online   |            12847 | 847293.45 |
| Magasin  |             8472 | 493847.92 |
| TOTAL    |            21319 | 1341141.37|
+----------+------------------+-----------+
```

**Pourquoi UNION ALL ici ?**
- ✅ Pas de doublons possibles (canaux différents)
- ✅ Plus rapide (pas de tri pour éliminer doublons)
- ✅ Inclusion d'une ligne de total

### Exemple 5 : UNION ALL pour performance

**Question métier** : *Liste de tous les produits (actifs + archivés)*

```sql
-- ✅ UNION ALL : Rapide si pas de doublons possibles
SELECT id_produit, nom_produit, 'Actif' AS statut
FROM produits_actifs

UNION ALL

SELECT id_produit, nom_produit, 'Archivé' AS statut
FROM produits_archives;

-- Les deux tables sont disjointes → UNION ALL approprié
```

💡 **Règle de choix** :
- **UNION** : Quand vous voulez éliminer les doublons
- **UNION ALL** : Quand vous savez qu'il n'y a pas de doublons OU vous voulez les garder

---

## INTERSECT : Trouver les éléments communs

**INTERSECT** retourne uniquement les lignes présentes dans **les deux** ensembles de résultats.

🆕 **Disponible depuis MariaDB 10.3.0**

### Syntaxe

```sql
SELECT colonnes FROM table1
INTERSECT
SELECT colonnes FROM table2;
```

### Exemple 6 : INTERSECT simple

**Question métier** : *Clients ayant commandé en 2024 ET en 2025*

```sql
-- Clients de 2024
SELECT id_client, nom, email
FROM clients c
WHERE EXISTS (
    SELECT 1
    FROM commandes
    WHERE id_client = c.id_client
      AND YEAR(date_commande) = 2024
)

INTERSECT

-- Clients de 2025
SELECT id_client, nom, email
FROM clients c
WHERE EXISTS (
    SELECT 1
    FROM commandes
    WHERE id_client = c.id_client
      AND YEAR(date_commande) = 2025
);
```

**Résultat attendu** :
```
+------------+----------------+-------------------+
| id_client  | nom            | email             |
+------------+----------------+-------------------+
|       1847 | Alice Martin   | alice@email.com   |
|       2934 | Bob Dupont     | bob@email.com     |
|       5621 | Sophie Bernard | sophie@email.com  |
+------------+----------------+-------------------+
```

**Alternative sans INTERSECT** (pour MariaDB < 10.3) :

```sql
-- Avec INNER JOIN ou EXISTS
SELECT DISTINCT c.id_client, c.nom, c.email
FROM clients c
INNER JOIN commandes cmd1 ON c.id_client = cmd1.id_client
INNER JOIN commandes cmd2 ON c.id_client = cmd2.id_client
WHERE YEAR(cmd1.date_commande) = 2024
  AND YEAR(cmd2.date_commande) = 2025;
```

### Exemple 7 : INTERSECT pour comparaison de listes

**Question métier** : *Produits vendus à la fois en France et en Belgique*

```sql
-- Produits vendus en France
SELECT DISTINCT p.id_produit, p.nom_produit
FROM produits p
INNER JOIN details_commande dc ON p.id_produit = dc.id_produit
INNER JOIN commandes cmd ON dc.id_commande = cmd.id_commande
INNER JOIN clients c ON cmd.id_client = c.id_client
WHERE c.pays = 'France'

INTERSECT

-- Produits vendus en Belgique
SELECT DISTINCT p.id_produit, p.nom_produit
FROM produits p
INNER JOIN details_commande dc ON p.id_produit = dc.id_produit
INNER JOIN commandes cmd ON dc.id_commande = cmd.id_commande
INNER JOIN clients c ON cmd.id_client = c.id_client
WHERE c.pays = 'Belgique';
```

**Application** :
- Analyses de marchés communs
- Stratégies produits transverses
- Identification de best-sellers internationaux

---

## EXCEPT : Calculer les différences

**EXCEPT** (ou **MINUS** dans certains SGBD) retourne les lignes de la première requête qui ne sont **pas** dans la deuxième.

🆕 **Disponible depuis MariaDB 10.3.0**

### Syntaxe

```sql
SELECT colonnes FROM table1
EXCEPT
SELECT colonnes FROM table2;
```

⚠️ **Ordre important** : `A EXCEPT B` ≠ `B EXCEPT A`

### Exemple 8 : EXCEPT pour trouver les absents

**Question métier** : *Clients inscrits mais n'ayant jamais commandé*

```sql
-- Tous les clients
SELECT id_client, nom, email
FROM clients

EXCEPT

-- Clients ayant commandé
SELECT DISTINCT c.id_client, c.nom, c.email
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client;
```

**Résultat attendu** :
```
+------------+----------------+-------------------+
| id_client  | nom            | email             |
+------------+----------------+-------------------+
|        892 | Charlie Durand | charlie@email.com |
|       1245 | Diana Lopez    | diana@email.com   |
|       3847 | Éric Moreau    | eric@email.com    |
+------------+----------------+-------------------+
```

**Alternative avec LEFT JOIN** :

```sql
-- Équivalent avec LEFT JOIN + IS NULL
SELECT c.id_client, c.nom, c.email
FROM clients c
LEFT JOIN commandes cmd ON c.id_client = cmd.id_client
WHERE cmd.id_commande IS NULL;
```

💡 **EXCEPT vs LEFT JOIN + IS NULL** :
- **EXCEPT** : Plus clair, intention évidente
- **LEFT JOIN** : Plus universel (compatible toutes versions)

### Exemple 9 : EXCEPT pour audits et conformité

**Question métier** : *Produits présents dans le catalogue mais absents du stock*

```sql
-- Produits au catalogue
SELECT id_produit, nom_produit, reference
FROM catalogue_produits

EXCEPT

-- Produits en stock
SELECT id_produit, nom_produit, reference
FROM inventaire_stock
WHERE quantite > 0;
```

**Application** :
- Détection de ruptures
- Audits d'inventaire
- Conformité catalogue/stock
- Alertes de réapprovisionnement

### Exemple 10 : EXCEPT pour comparaisons de versions

**Question métier** : *Nouvelles fonctionnalités dans version 2.0 vs 1.0*

```sql
-- Fonctionnalités version 2.0
SELECT nom_fonctionnalite, description
FROM fonctionnalites
WHERE version = '2.0'

EXCEPT

-- Fonctionnalités version 1.0
SELECT nom_fonctionnalite, description
FROM fonctionnalites
WHERE version = '1.0';
```

**Application** :
- Changelog automatique
- Documentation de release
- Comparaisons de versions
- Migrations de systèmes

---

## Combinaison d'opérateurs

Vous pouvez combiner plusieurs opérateurs ensemblistes dans une même requête.

### Exemple 11 : UNION + EXCEPT

**Question métier** : *Clients FR ou BE, sauf ceux ayant commandé récemment*

```sql
(
    -- Clients FR
    SELECT id_client, nom, pays
    FROM clients
    WHERE pays = 'France'

    UNION

    -- Clients BE
    SELECT id_client, nom, pays
    FROM clients
    WHERE pays = 'Belgique'
)

EXCEPT

-- Clients avec commandes récentes
(
    SELECT DISTINCT c.id_client, c.nom, c.pays
    FROM clients c
    INNER JOIN commandes cmd ON c.id_client = cmd.id_client
    WHERE cmd.date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
);
```

**Application** : Campagnes de réactivation, ciblage marketing, win-back.

### Ordre de priorité et parenthèses

```sql
-- Parenthèses recommandées pour clarté
(SELECT ... UNION SELECT ...)
INTERSECT
(SELECT ... EXCEPT SELECT ...);
```

---

## Compatibilité des colonnes : Règles et conversions

### Règle 1 : Même nombre de colonnes

```sql
-- ❌ ERREUR : Nombre différent de colonnes
SELECT nom, email FROM clients
UNION
SELECT nom_produit FROM produits;
-- Error: "The used SELECT statements have a different number of columns"

-- ✅ CORRECT : Même nombre
SELECT nom, email FROM clients
UNION
SELECT nom_produit, '' AS email_vide FROM produits;
```

### Règle 2 : Types compatibles

```sql
-- ✅ Compatible : VARCHAR et VARCHAR
SELECT nom FROM clients
UNION
SELECT nom_produit FROM produits;

-- ✅ Compatible : INT et INT
SELECT id_client FROM clients
UNION
SELECT id_produit FROM produits;

-- ⚠️ Compatible avec conversion : INT et VARCHAR
SELECT id_client FROM clients         -- INT
UNION
SELECT reference FROM produits;       -- VARCHAR
-- MariaDB convertit automatiquement

-- ❌ Incompatible : DATE et VARCHAR (dans certains cas)
-- Mais peut fonctionner avec conversion implicite
```

### Règle 3 : Longueur des colonnes

```sql
-- La colonne résultante prend la longueur maximale
SELECT nom FROM clients           -- VARCHAR(100)
UNION
SELECT nom_produit FROM produits; -- VARCHAR(200)
-- Résultat : VARCHAR(200)
```

### Exemple 12 : Normalisation des types

**Question métier** : *Consolidation avec types différents*

```sql
-- Normalisation explicite des types
SELECT
    CAST(id_client AS CHAR(10)) AS identifiant,
    nom AS libelle,
    DATE_FORMAT(date_inscription, '%Y-%m-%d') AS date_creation
FROM clients

UNION ALL

SELECT
    CAST(id_produit AS CHAR(10)) AS identifiant,
    nom_produit AS libelle,
    DATE_FORMAT(date_ajout, '%Y-%m-%d') AS date_creation
FROM produits

ORDER BY date_creation DESC;
```

---

## Performance et optimisation

### UNION vs UNION ALL : Impact performance

```sql
-- ❌ PLUS LENT : UNION élimine doublons
SELECT nom FROM clients        -- 10 000 lignes
UNION
SELECT nom FROM prospects;     -- 5 000 lignes
-- Étapes : 1) Fusion (15 000), 2) Tri, 3) Élimination doublons, 4) Résultat (~14 500)

-- ✅ PLUS RAPIDE : UNION ALL garde tout
SELECT nom FROM clients
UNION ALL
SELECT nom FROM prospects;
-- Étapes : 1) Fusion (15 000), 2) Résultat (15 000)
```

**Différence** : UNION peut être **2x à 10x plus lent** selon le volume et le nombre de doublons.

### Index et opérateurs ensemblistes

```sql
-- ✅ Index utiles sur colonnes de tri
CREATE INDEX idx_clients_nom ON clients(nom);
CREATE INDEX idx_prospects_nom ON prospects(nom);

SELECT nom FROM clients
UNION
SELECT nom FROM prospects
ORDER BY nom;
-- Les index accélèrent le tri final
```

### EXPLAIN avec UNION

```sql
EXPLAIN
SELECT nom FROM clients
UNION
SELECT nom FROM prospects;
```

**Points à vérifier** :
- `UNION RESULT` : Opération UNION effectuée
- `Using temporary` : Table temporaire créée (UNION)
- `Using filesort` : Tri pour éliminer doublons

### Optimisation : Filtrer avant UNION

```sql
-- ❌ MOINS OPTIMAL : UNION puis filtrage
(SELECT * FROM clients UNION SELECT * FROM prospects)
WHERE ville = 'Paris';

-- ✅ PLUS OPTIMAL : Filtrer d'abord
SELECT * FROM clients WHERE ville = 'Paris'
UNION
SELECT * FROM prospects WHERE ville = 'Paris';
```

---

## Cas d'usage métier avancés

### Cas 1 : Migration de données

```sql
-- Vérifier la complétude d'une migration
-- Données manquantes dans nouvelle base
SELECT id, nom FROM ancienne_base.clients
EXCEPT
SELECT id, nom FROM nouvelle_base.clients;

-- Données en trop (ajoutées par erreur)
SELECT id, nom FROM nouvelle_base.clients
EXCEPT
SELECT id, nom FROM ancienne_base.clients;
```

### Cas 2 : Rapports consolidés multi-sources

```sql
-- Rapport financier consolidé
SELECT
    'Ventes' AS type_operation,
    date_operation,
    montant,
    'Crédit' AS sens
FROM ventes

UNION ALL

SELECT
    'Achats' AS type_operation,
    date_operation,
    -montant AS montant,
    'Débit' AS sens
FROM achats

UNION ALL

SELECT
    'Frais' AS type_operation,
    date_operation,
    -montant AS montant,
    'Débit' AS sens
FROM frais_generaux

ORDER BY date_operation;
```

### Cas 3 : Recherche multi-tables

```sql
-- Recherche globale dans plusieurs entités
SELECT
    'Client' AS type_resultat,
    id_client AS id,
    nom AS libelle,
    email AS contact
FROM clients
WHERE nom LIKE '%Martin%'

UNION

SELECT
    'Produit' AS type_resultat,
    id_produit AS id,
    nom_produit AS libelle,
    reference AS contact
FROM produits
WHERE nom_produit LIKE '%Martin%'

UNION

SELECT
    'Fournisseur' AS type_resultat,
    id_fournisseur AS id,
    nom_societe AS libelle,
    email AS contact
FROM fournisseurs
WHERE nom_societe LIKE '%Martin%'

ORDER BY libelle;
```

**Application** : Moteur de recherche global, navigation transverse, autocomplete.

---

## Limites et alternatives

### Limite 1 : ORDER BY individuel impossible

```sql
-- ❌ IMPOSSIBLE : ORDER BY dans chaque SELECT
(SELECT nom FROM clients ORDER BY nom LIMIT 10)
UNION
(SELECT nom FROM prospects ORDER BY nom LIMIT 10);
-- Le ORDER BY est ignoré

-- ✅ SOLUTION : Sous-requêtes
SELECT nom FROM (
    SELECT nom FROM clients ORDER BY nom LIMIT 10
) AS top_clients
UNION
SELECT nom FROM (
    SELECT nom FROM prospects ORDER BY nom LIMIT 10
) AS top_prospects;
```

### Limite 2 : LIMIT global uniquement

```sql
-- ORDER BY et LIMIT uniquement après tous les UNION
SELECT nom FROM clients
UNION
SELECT nom FROM prospects
ORDER BY nom
LIMIT 20;  -- LIMIT s'applique au résultat final
```

### Alternative : CTE pour plus de clarté

```sql
-- Avec CTE (Common Table Expression)
WITH
    clients_fr AS (
        SELECT id_client, nom FROM clients WHERE pays = 'France'
    ),
    clients_be AS (
        SELECT id_client, nom FROM clients WHERE pays = 'Belgique'
    )
SELECT * FROM clients_fr
UNION
SELECT * FROM clients_be
ORDER BY nom;
```

---

## ✅ Points clés à retenir

1. **UNION combine verticalement** – empile les lignes (vs jointures qui ajoutent colonnes)

2. **UNION élimine doublons, UNION ALL les garde** – UNION ALL plus rapide si pas de doublons

3. **Même structure obligatoire** – même nombre de colonnes, types compatibles, ordre important

4. **Noms de colonnes du premier SELECT** – alias de la première requête utilisés

5. **ORDER BY uniquement à la fin** – après tous les opérateurs ensemblistes

6. **INTERSECT = éléments communs** – présents dans tous les ensembles (MariaDB 10.3+)

7. **EXCEPT = différence** – éléments du premier ensemble absents du second (MariaDB 10.3+)

8. **EXCEPT dépend de l'ordre** – A EXCEPT B ≠ B EXCEPT A

9. **Performance : filtrer avant UNION** – réduire le volume avant de fusionner

10. **Alternative avec jointures possible** – mais opérateurs ensemblistes plus clairs pour certains cas

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 UNION](https://mariadb.com/kb/en/union/) – Documentation complète
- [📖 INTERSECT](https://mariadb.com/kb/en/intersect/) – Disponible depuis 10.3.0
- [📖 EXCEPT](https://mariadb.com/kb/en/except/) – Disponible depuis 10.3.0

### Articles approfondis
- [Set Operations](https://modern-sql.com/feature/union) – Guide complet des opérations ensemblistes
- [UNION Performance](https://use-the-index-luke.com/sql/union) – Optimisation

---

## 🎓 Conclusion du Chapitre 3 : Requêtes SQL Intermédiaires

Félicitations ! Vous avez complété le chapitre sur les **Requêtes SQL Intermédiaires**. Vous maîtrisez maintenant :

### ✅ Compétences acquises

1. **Agrégations (3.1)** – COUNT, SUM, AVG, MIN, MAX
2. **Regroupements (3.2)** – GROUP BY, HAVING, WHERE vs HAVING
3. **Jointures (3.3)** – INNER, LEFT, RIGHT, CROSS, Self-Join
4. **Sous-requêtes (3.4)** – Scalaires, listes, tables dérivées, corrélées
5. **Opérateurs ensemblistes (3.5)** – UNION, INTERSECT, EXCEPT

### 🎯 Vous êtes maintenant capable de :

- ✅ Analyser des données avec agrégations et groupements
- ✅ Croiser des informations de plusieurs tables
- ✅ Décomposer des problèmes complexes en étapes logiques
- ✅ Consolider des résultats de sources multiples
- ✅ Résoudre 80% des besoins métier en SQL

### 📈 Progression

```
Chapitre 1-2 : Fondamentaux ████████░░░░░░░░ 40%
Chapitre 3   : Intermédiaire ████████████░░░░ 70%
Chapitre 4   : Avancé        ░░░░░░░░░░░░░░░░  0%
```

---


⏭️ [Fonctions de chaînes de caractères](/03-requetes-sql-intermediaires/06-fonctions-chaines.md)

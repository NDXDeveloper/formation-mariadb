🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 Sous-requêtes et requêtes imbriquées

> **Niveau** : Intermédiaire
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 3.1 à 3.3, maîtrise des jointures et agrégations

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre les différents types de sous-requêtes et leur utilité
- Maîtriser les sous-requêtes scalaires, à une colonne et à plusieurs colonnes
- Utiliser les opérateurs IN, EXISTS, ANY, ALL avec des sous-requêtes
- Différencier sous-requêtes corrélées et non-corrélées
- Placer des sous-requêtes dans SELECT, FROM, WHERE et HAVING
- Optimiser les performances des sous-requêtes
- Choisir entre sous-requêtes et jointures selon le contexte
- Résoudre des problèmes complexes en plusieurs étapes logiques

---

## Introduction

Les **sous-requêtes** (ou *subqueries*) sont des requêtes SQL **imbriquées** à l'intérieur d'une autre requête. Elles permettent de résoudre des problèmes complexes en les décomposant en **étapes logiques** plus simples.

### Le principe

```sql
-- Requête principale
SELECT nom, salaire
FROM employes
WHERE salaire > (
    -- Sous-requête : calcule la moyenne
    SELECT AVG(salaire)
    FROM employes
);
```

**Analogie** : C'est comme résoudre un problème mathématique en plusieurs étapes :
1. D'abord, calculer la moyenne (sous-requête)
2. Ensuite, utiliser ce résultat pour filtrer (requête principale)

### Pourquoi utiliser des sous-requêtes ?

| Avantage | Description | Exemple |
|----------|-------------|---------|
| **Décomposition logique** | Diviser un problème complexe en étapes | "Clients ayant commandé plus que la moyenne" |
| **Lisibilité** | Intentions claires, code auto-documenté | Calculer un seuil avant de filtrer |
| **Réutilisation** | Même calcul utilisé plusieurs fois | Sous-requête dans FROM utilisée en JOIN |
| **Filtrage avancé** | Conditions basées sur agrégations complexes | WHERE avec EXISTS |

💡 **Quand préférer les sous-requêtes aux jointures ?**
- Quand la logique métier est plus claire en étapes
- Pour des calculs intermédiaires (moyennes, totaux)
- Avec EXISTS pour tester l'existence
- Pour éviter les doublons dus aux relations 1:N

---

## Types de sous-requêtes par résultat

### 1. Sous-requête scalaire (une seule valeur)

Retourne **une seule ligne, une seule colonne** → une valeur unique.

```sql
-- Produits plus chers que le prix moyen
SELECT nom_produit, prix_unitaire
FROM produits
WHERE prix_unitaire > (
    SELECT AVG(prix_unitaire)  -- Scalaire : 1 valeur
    FROM produits
);
```

**Utilisation** : Comparaisons avec =, >, <, >=, <=, !=

### 2. Sous-requête à une colonne (liste de valeurs)

Retourne **plusieurs lignes, une seule colonne** → une liste.

```sql
-- Clients ayant commandé
SELECT nom
FROM clients
WHERE id_client IN (
    SELECT id_client  -- Liste : plusieurs valeurs
    FROM commandes
);
```

**Utilisation** : Opérateurs IN, NOT IN, ANY, ALL

### 3. Sous-requête à plusieurs colonnes (table)

Retourne **plusieurs lignes et colonnes** → une table complète.

```sql
-- Sous-requête dans FROM (table dérivée)
SELECT categorie, ca_moyen
FROM (
    SELECT
        categorie,
        AVG(prix_unitaire) AS ca_moyen
    FROM produits
    GROUP BY categorie
) AS stats
WHERE ca_moyen > 100;
```

**Utilisation** : Clause FROM (tables dérivées), comparaisons multi-colonnes

---

## Sous-requêtes dans les différentes clauses

### Sous-requêtes dans SELECT

#### Exemple 1 : Colonnes calculées avec sous-requête

**Question métier** : *Pour chaque client, afficher son CA et le CA moyen de tous les clients*

```sql
-- Sous-requête scalaire dans SELECT
SELECT
    c.nom,
    (
        SELECT COALESCE(SUM(montant_total), 0)
        FROM commandes
        WHERE id_client = c.id_client
          AND statut IN ('confirmée', 'expédiée', 'livrée')
    ) AS ca_client,
    (
        SELECT AVG(ca)
        FROM (
            SELECT SUM(montant_total) AS ca
            FROM commandes
            WHERE statut IN ('confirmée', 'expédiée', 'livrée')
            GROUP BY id_client
        ) AS ca_par_client
    ) AS ca_moyen_global,
    c.ville
FROM clients c
ORDER BY ca_client DESC
LIMIT 10;
```

**Résultat attendu** :
```
+----------------+------------+------------------+-------------+
| nom            | ca_client  | ca_moyen_global  | ville       |
+----------------+------------+------------------+-------------+
| Alice Martin   |  14283.45  |         1847.23  | Paris       |
| Bob Dupont     |  12847.92  |         1847.23  | Lyon        |
| Sophie Bernard |  11293.18  |         1847.23  | Marseille   |
+----------------+------------+------------------+-------------+
```

**Explication** :
- Première sous-requête : CA du client (corrélée à `c.id_client`)
- Deuxième sous-requête : CA moyen de tous les clients (non corrélée)
- Chaque sous-requête s'exécute pour chaque ligne de `clients`

⚠️ **Performance** : Les sous-requêtes dans SELECT peuvent être lentes car exécutées pour **chaque ligne**. Préférez les jointures si possible.

#### Exemple 2 : Sous-requête pour calcul de pourcentage

```sql
-- Part de chaque catégorie dans le CA total
SELECT
    p.categorie,
    SUM(dc.quantite * dc.prix_unitaire) AS ca_categorie,
    ROUND(
        100.0 * SUM(dc.quantite * dc.prix_unitaire) / (
            SELECT SUM(quantite * prix_unitaire)
            FROM details_commande dc2
            INNER JOIN commandes cmd2 ON dc2.id_commande = cmd2.id_commande
            WHERE cmd2.statut IN ('confirmée', 'expédiée', 'livrée')
        ),
        2
    ) AS pourcentage_ca
FROM produits p
INNER JOIN details_commande dc ON p.id_produit = dc.id_produit
INNER JOIN commandes cmd ON dc.id_commande = cmd.id_commande
WHERE cmd.statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY p.categorie
ORDER BY ca_categorie DESC;
```

**Cas d'usage** : Dashboards, rapports de contribution, analyses de mix.

---

### Sous-requêtes dans WHERE

#### Exemple 3 : Filtrage avec sous-requête scalaire

**Question métier** : *Produits au-dessus du prix moyen*

```sql
-- Comparaison avec une valeur calculée
SELECT
    nom_produit,
    categorie,
    prix_unitaire,
    (
        SELECT ROUND(AVG(prix_unitaire), 2)
        FROM produits
    ) AS prix_moyen
FROM produits
WHERE prix_unitaire > (
    SELECT AVG(prix_unitaire)
    FROM produits
)
ORDER BY prix_unitaire DESC;
```

**Résultat attendu** :
```
+--------------------+---------------+---------------+-------------+
| nom_produit        | categorie     | prix_unitaire | prix_moyen  |
+--------------------+---------------+---------------+-------------+
| Laptop Pro         | Électronique  |       1299.99 |       45.84 |
| Tablette Premium   | Électronique  |        649.99 |       45.84 |
| Écran 27"          | Électronique  |        389.99 |       45.84 |
+--------------------+---------------+---------------+-------------+
```

**Explication** :
- La sous-requête calcule `AVG(prix_unitaire)` une seule fois
- Le résultat est utilisé pour filtrer dans WHERE
- Sous-requête **non corrélée** (indépendante de la requête principale)

#### Exemple 4 : Opérateur IN avec sous-requête

**Question métier** : *Clients ayant commandé au moins une fois*

```sql
-- Utilisation de IN
SELECT
    c.nom,
    c.email,
    c.ville
FROM clients c
WHERE c.id_client IN (
    SELECT DISTINCT id_client
    FROM commandes
    WHERE statut IN ('confirmée', 'expédiée', 'livrée')
)
ORDER BY c.nom;
```

**Résultat attendu** :
```
+----------------+-------------------+-------------+
| nom            | email             | ville       |
+----------------+-------------------+-------------+
| Alice Martin   | alice@email.com   | Paris       |
| Bob Dupont     | bob@email.com     | Lyon        |
| Sophie Bernard | sophie@email.com  | Marseille   |
+----------------+-------------------+-------------+
```

**Alternative avec EXISTS** (souvent plus performant) :

```sql
SELECT c.nom, c.email, c.ville
FROM clients c
WHERE EXISTS (
    SELECT 1
    FROM commandes cmd
    WHERE cmd.id_client = c.id_client
      AND cmd.statut IN ('confirmée', 'expédiée', 'livrée')
);
```

💡 **EXISTS vs IN** :
- **EXISTS** : S'arrête dès qu'une ligne correspond (plus rapide)
- **IN** : Doit construire la liste complète avant de comparer

#### Exemple 5 : Opérateur NOT IN pour trouver les exclus

**Question métier** : *Clients n'ayant jamais commandé*

```sql
-- Clients sans commande
SELECT
    c.id_client,
    c.nom,
    c.email,
    c.date_inscription
FROM clients c
WHERE c.id_client NOT IN (
    SELECT id_client
    FROM commandes
    WHERE id_client IS NOT NULL  -- Important pour NOT IN !
)
ORDER BY c.date_inscription;
```

⚠️ **PIÈGE avec NOT IN et NULL** :
```sql
-- ❌ ATTENTION : Si la sous-requête contient NULL, NOT IN retourne toujours FALSE
WHERE id_client NOT IN (1, 2, NULL)  -- Ne retourne rien !

-- ✅ SOLUTION : Filtrer les NULL ou utiliser NOT EXISTS
WHERE id_client NOT IN (
    SELECT id_client
    FROM commandes
    WHERE id_client IS NOT NULL
)

-- ✅ MEILLEUR : Utiliser NOT EXISTS (pas de problème avec NULL)
WHERE NOT EXISTS (
    SELECT 1
    FROM commandes
    WHERE id_client = c.id_client
)
```

---

### Sous-requêtes dans FROM (tables dérivées)

#### Exemple 6 : Table dérivée simple

**Question métier** : *Catégories avec CA moyen > 100€*

```sql
-- Sous-requête créant une table temporaire
SELECT
    categorie,
    ca_moyen,
    nb_produits
FROM (
    -- Table dérivée
    SELECT
        categorie,
        AVG(prix_unitaire) AS ca_moyen,
        COUNT(*) AS nb_produits
    FROM produits
    GROUP BY categorie
) AS stats_categories
WHERE ca_moyen > 100
ORDER BY ca_moyen DESC;
```

**Résultat attendu** :
```
+---------------+-----------+--------------+
| categorie     | ca_moyen  | nb_produits  |
+---------------+-----------+--------------+
| Électronique  |    345.67 |          247 |
| Maison        |    142.38 |           89 |
+---------------+-----------+--------------+
```

**Avantage** : Permet d'utiliser des alias d'agrégation dans WHERE (normalement impossible).

#### Exemple 7 : Jointure avec table dérivée

**Question métier** : *Clients dépensant plus que la moyenne de leur pays*

```sql
-- Table dérivée avec agrégation par pays
SELECT
    c.nom,
    c.pays,
    c.ca_client,
    stats_pays.ca_moyen_pays
FROM (
    -- CA par client
    SELECT
        id_client,
        SUM(montant_total) AS ca_client
    FROM commandes
    WHERE statut IN ('confirmée', 'expédiée', 'livrée')
    GROUP BY id_client
) AS ca_clients
INNER JOIN clients c ON ca_clients.id_client = c.id_client
INNER JOIN (
    -- CA moyen par pays
    SELECT
        cl.pays,
        AVG(ca_client) AS ca_moyen_pays
    FROM (
        SELECT id_client, SUM(montant_total) AS ca_client
        FROM commandes
        WHERE statut IN ('confirmée', 'expédiée', 'livrée')
        GROUP BY id_client
    ) AS ca
    INNER JOIN clients cl ON ca.id_client = cl.id_client
    GROUP BY cl.pays
) AS stats_pays ON c.pays = stats_pays.pays
WHERE ca_clients.ca_client > stats_pays.ca_moyen_pays
ORDER BY c.pays, ca_clients.ca_client DESC;
```

**Application** : Segmentation avancée, identification d'outliers, benchmarking par segment.

---

### Sous-requêtes dans HAVING

#### Exemple 8 : Filtrage d'agrégations avec sous-requête

**Question métier** : *Catégories vendant plus que la moyenne globale*

```sql
-- HAVING avec sous-requête
SELECT
    p.categorie,
    COUNT(DISTINCT cmd.id_commande) AS nb_commandes,
    SUM(dc.quantite * dc.prix_unitaire) AS ca_categorie
FROM produits p
INNER JOIN details_commande dc ON p.id_produit = dc.id_produit
INNER JOIN commandes cmd ON dc.id_commande = cmd.id_commande
WHERE cmd.statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY p.categorie
HAVING SUM(dc.quantite * dc.prix_unitaire) > (
    -- CA moyen par catégorie
    SELECT AVG(ca_cat)
    FROM (
        SELECT SUM(dc2.quantite * dc2.prix_unitaire) AS ca_cat
        FROM produits p2
        INNER JOIN details_commande dc2 ON p2.id_produit = dc2.id_produit
        INNER JOIN commandes cmd2 ON dc2.id_commande = cmd2.id_commande
        WHERE cmd2.statut IN ('confirmée', 'expédiée', 'livrée')
        GROUP BY p2.categorie
    ) AS categories_ca
)
ORDER BY ca_categorie DESC;
```

**Cas d'usage** : Identification de segments performants, analyses de contribution.

---

## Opérateurs spéciaux avec sous-requêtes

### EXISTS et NOT EXISTS

#### Exemple 9 : EXISTS pour tester l'existence

**Question métier** : *Clients ayant commandé dans les 30 derniers jours*

```sql
-- EXISTS : Test d'existence
SELECT
    c.id_client,
    c.nom,
    c.email
FROM clients c
WHERE EXISTS (
    SELECT 1  -- La valeur n'a pas d'importance, seule l'existence compte
    FROM commandes cmd
    WHERE cmd.id_client = c.id_client
      AND cmd.date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
      AND cmd.statut IN ('confirmée', 'expédiée', 'livrée')
)
ORDER BY c.nom;
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

**Avantages de EXISTS** :
- ✅ S'arrête dès la première correspondance (efficace)
- ✅ Pas de problème avec NULL
- ✅ Peut utiliser des index efficacement
- ✅ Lisible pour "vérifier si au moins un..."

#### Exemple 10 : NOT EXISTS pour exclusion

**Question métier** : *Produits jamais commandés*

```sql
-- NOT EXISTS : Absence
SELECT
    p.id_produit,
    p.nom_produit,
    p.categorie,
    p.stock
FROM produits p
WHERE NOT EXISTS (
    SELECT 1
    FROM details_commande dc
    WHERE dc.id_produit = p.id_produit
)
ORDER BY p.categorie, p.nom_produit;
```

**Application** : Dead stock, produits à promouvoir, révision d'assortiment.

---

### ANY et ALL

#### Exemple 11 : ANY (au moins un)

**Question métier** : *Produits plus chers qu'au moins un produit de catégorie "Livres"*

```sql
-- ANY : Au moins une valeur satisfait la condition
SELECT
    nom_produit,
    categorie,
    prix_unitaire
FROM produits
WHERE prix_unitaire > ANY (
    SELECT prix_unitaire
    FROM produits
    WHERE categorie = 'Livres'
)
AND categorie != 'Livres'
ORDER BY prix_unitaire;
```

**Équivalent** :
```sql
-- ANY équivaut à > MIN()
WHERE prix_unitaire > (
    SELECT MIN(prix_unitaire)
    FROM produits
    WHERE categorie = 'Livres'
)
```

#### Exemple 12 : ALL (toutes les valeurs)

**Question métier** : *Produits plus chers que TOUS les livres*

```sql
-- ALL : Toutes les valeurs doivent satisfaire la condition
SELECT
    nom_produit,
    categorie,
    prix_unitaire
FROM produits
WHERE prix_unitaire > ALL (
    SELECT prix_unitaire
    FROM produits
    WHERE categorie = 'Livres'
)
ORDER BY prix_unitaire;
```

**Équivalent** :
```sql
-- ALL équivaut à > MAX()
WHERE prix_unitaire > (
    SELECT MAX(prix_unitaire)
    FROM produits
    WHERE categorie = 'Livres'
)
```

**Résumé des opérateurs** :

| Opérateur | Signification | Équivalent |
|-----------|---------------|------------|
| `> ANY` | Plus grand qu'au moins un | `> MIN(...)` |
| `< ANY` | Plus petit qu'au moins un | `< MAX(...)` |
| `> ALL` | Plus grand que tous | `> MAX(...)` |
| `< ALL` | Plus petit que tous | `< MIN(...)` |
| `= ANY` | Égal à au moins un | `IN (...)` |
| `!= ALL` | Différent de tous | `NOT IN (...)` |

💡 **Recommandation** : Préférez MIN/MAX qui sont plus clairs et plus courants.

---

## Sous-requêtes corrélées vs non-corrélées

### Sous-requête non-corrélée

**Indépendante** de la requête externe, exécutée **une seule fois**.

```sql
-- Non-corrélée : exécutée 1 fois
SELECT nom_produit, prix_unitaire
FROM produits
WHERE prix_unitaire > (
    SELECT AVG(prix_unitaire)  -- Calculé 1 fois
    FROM produits
);
```

**Avantage** : Performance optimale (1 exécution).

### Sous-requête corrélée

**Dépend** de la requête externe, exécutée **pour chaque ligne**.

#### Exemple 13 : Sous-requête corrélée classique

**Question métier** : *Produits au-dessus du prix moyen de leur catégorie*

```sql
-- Corrélée : exécutée pour chaque produit
SELECT
    p1.nom_produit,
    p1.categorie,
    p1.prix_unitaire,
    (
        SELECT ROUND(AVG(p2.prix_unitaire), 2)
        FROM produits p2
        WHERE p2.categorie = p1.categorie  -- Corrélation avec p1
    ) AS prix_moyen_categorie
FROM produits p1
WHERE p1.prix_unitaire > (
    SELECT AVG(p2.prix_unitaire)
    FROM produits p2
    WHERE p2.categorie = p1.categorie  -- Corrélation
)
ORDER BY p1.categorie, p1.prix_unitaire DESC;
```

**Résultat attendu** :
```
+--------------------+---------------+---------------+-----------------------+
| nom_produit        | categorie     | prix_unitaire | prix_moyen_categorie  |
+--------------------+---------------+---------------+-----------------------+
| Laptop Pro         | Électronique  |       1299.99 |                345.67 |
| Tablette Premium   | Électronique  |        649.99 |                345.67 |
| Manteau hiver      | Vêtements     |        249.99 |                 54.23 |
+--------------------+---------------+---------------+-----------------------+
```

**⚠️ Performance** : Sous-requête exécutée N fois (une par ligne) → peut être lent.

**Alternative avec jointure** (plus performant) :

```sql
-- Jointure avec table dérivée (plus rapide)
SELECT
    p.nom_produit,
    p.categorie,
    p.prix_unitaire,
    stats.prix_moyen_categorie
FROM produits p
INNER JOIN (
    SELECT
        categorie,
        AVG(prix_unitaire) AS prix_moyen_categorie
    FROM produits
    GROUP BY categorie
) AS stats ON p.categorie = stats.categorie
WHERE p.prix_unitaire > stats.prix_moyen_categorie
ORDER BY p.categorie, p.prix_unitaire DESC;
```

---

## Cas d'usage avancés

### Exemple 14 : Top N par catégorie

**Question métier** : *Les 3 produits les plus chers de chaque catégorie*

```sql
-- Top 3 avec sous-requête corrélée
SELECT
    p1.categorie,
    p1.nom_produit,
    p1.prix_unitaire
FROM produits p1
WHERE (
    SELECT COUNT(*)
    FROM produits p2
    WHERE p2.categorie = p1.categorie
      AND p2.prix_unitaire > p1.prix_unitaire
) < 3  -- Moins de 3 produits plus chers
ORDER BY p1.categorie, p1.prix_unitaire DESC;
```

**Résultat attendu** :
```
+---------------+--------------------+---------------+
| categorie     | nom_produit        | prix_unitaire |
+---------------+--------------------+---------------+
| Électronique  | Laptop Pro         |       1299.99 |
| Électronique  | Tablette Premium   |        649.99 |
| Électronique  | Écran 27"          |        389.99 |
| Vêtements     | Manteau hiver      |        249.99 |
| Vêtements     | Costume            |        189.99 |
| Vêtements     | Robe soirée        |        159.99 |
+---------------+--------------------+---------------+
```

💡 **Note** : En MariaDB 10.2+, utilisez plutôt **Window Functions** (ROW_NUMBER) pour ce cas (section 4.2).

### Exemple 15 : Comparaison avec valeurs calculées

**Question métier** : *Clients avec un panier moyen supérieur à 2x la médiane*

```sql
-- Calcul complexe multi-étapes
WITH stats_globales AS (
    SELECT
        AVG(montant_total) AS panier_moyen_global,
        -- Approximation de la médiane
        (
            SELECT montant_total
            FROM commandes
            WHERE statut IN ('confirmée', 'expédiée', 'livrée')
            ORDER BY montant_total
            LIMIT 1 OFFSET (
                SELECT COUNT(*) / 2
                FROM commandes
                WHERE statut IN ('confirmée', 'expédiée', 'livrée')
            )
        ) AS mediane_globale
    FROM commandes
    WHERE statut IN ('confirmée', 'expédiée', 'livrée')
)
SELECT
    c.nom,
    AVG(cmd.montant_total) AS panier_moyen_client,
    (SELECT panier_moyen_global FROM stats_globales) AS ref_moyenne,
    (SELECT mediane_globale FROM stats_globales) AS ref_mediane
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client
WHERE cmd.statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY c.id_client, c.nom
HAVING AVG(cmd.montant_total) > 2 * (
    SELECT mediane_globale FROM stats_globales
)
ORDER BY panier_moyen_client DESC;
```

**Application** : Identification de VIP, segmentation avancée, personnalisation.

### Exemple 16 : Détection d'anomalies

**Question métier** : *Commandes avec des quantités anormalement élevées (>3 écarts-types)*

```sql
-- Détection statistique d'outliers
SELECT
    cmd.id_commande,
    c.nom AS client,
    dc.id_produit,
    p.nom_produit,
    dc.quantite,
    stats.quantite_moyenne,
    stats.ecart_type
FROM commandes cmd
INNER JOIN clients c ON cmd.id_client = c.id_client
INNER JOIN details_commande dc ON cmd.id_commande = dc.id_commande
INNER JOIN produits p ON dc.id_produit = p.id_produit
INNER JOIN (
    -- Statistiques par produit
    SELECT
        id_produit,
        AVG(quantite) AS quantite_moyenne,
        STDDEV(quantite) AS ecart_type
    FROM details_commande
    GROUP BY id_produit
    HAVING ecart_type > 0
) AS stats ON dc.id_produit = stats.id_produit
WHERE dc.quantite > stats.quantite_moyenne + 3 * stats.ecart_type
ORDER BY dc.quantite DESC;
```

**Application** : Détection de fraude, alertes automatiques, contrôle qualité.

---

## Optimisation des sous-requêtes

### Performance : Sous-requêtes vs Jointures

#### Comparaison

| Critère | Sous-requête | Jointure |
|---------|--------------|----------|
| **Lisibilité** | ✅ Logique en étapes claires | ⚠️ Peut être complexe |
| **Performance non-corrélée** | ✅ Bonne | ✅ Bonne (similaire) |
| **Performance corrélée** | ❌ Lente (N exécutions) | ✅ Rapide |
| **Doublons** | ✅ Pas de doublons automatiques | ⚠️ Peut créer doublons (1:N) |
| **Maintenabilité** | ✅ Intentions claires | ⚠️ Nécessite compréhension jointures |

#### Exemple : Conversion sous-requête → jointure

```sql
-- ❌ LENT : Sous-requête corrélée dans SELECT
SELECT
    c.nom,
    (
        SELECT COUNT(*)
        FROM commandes
        WHERE id_client = c.id_client
    ) AS nb_commandes,
    (
        SELECT SUM(montant_total)
        FROM commandes
        WHERE id_client = c.id_client
    ) AS ca_total
FROM clients c;

-- ✅ RAPIDE : Jointure avec agrégation
SELECT
    c.nom,
    COUNT(cmd.id_commande) AS nb_commandes,
    COALESCE(SUM(cmd.montant_total), 0) AS ca_total
FROM clients c
LEFT JOIN commandes cmd ON c.id_client = cmd.id_client
GROUP BY c.id_client, c.nom;
```

**Gain de performance** : 10x à 100x plus rapide selon le volume.

### Index pour sous-requêtes

```sql
-- Index sur colonnes utilisées dans sous-requêtes
CREATE INDEX idx_commandes_client ON commandes(id_client);
CREATE INDEX idx_commandes_statut ON commandes(statut);
CREATE INDEX idx_details_produit ON details_commande(id_produit);
```

### EXPLAIN pour analyser

```sql
EXPLAIN
SELECT nom
FROM clients
WHERE id_client IN (
    SELECT id_client
    FROM commandes
);
```

Vérifiez :
- `dependent subquery` → Corrélée (potentiellement lent)
- `subquery` → Non-corrélée (bon)
- MariaDB transforme parfois automatiquement en jointure

---

## Pièges courants et solutions

### Piège 1 : NOT IN avec NULL

```sql
-- ❌ DANGER : Si la sous-requête retourne NULL, résultat vide
SELECT nom
FROM clients
WHERE id_client NOT IN (
    SELECT id_client  -- Peut contenir NULL !
    FROM commandes
);

-- ✅ SOLUTION 1 : Filtrer NULL
WHERE id_client NOT IN (
    SELECT id_client
    FROM commandes
    WHERE id_client IS NOT NULL
)

-- ✅ SOLUTION 2 : Utiliser NOT EXISTS
WHERE NOT EXISTS (
    SELECT 1
    FROM commandes
    WHERE id_client = clients.id_client
)
```

### Piège 2 : Sous-requête retournant plusieurs lignes

```sql
-- ❌ ERREUR : Sous-requête scalaire retourne > 1 ligne
SELECT nom, prix_unitaire
FROM produits
WHERE prix_unitaire = (
    SELECT prix_unitaire  -- Peut retourner plusieurs valeurs !
    FROM produits
    WHERE categorie = 'Électronique'
);
-- Error: "Subquery returns more than 1 row"

-- ✅ SOLUTION : Utiliser IN ou agrégation
WHERE prix_unitaire IN (
    SELECT prix_unitaire
    FROM produits
    WHERE categorie = 'Électronique'
)
-- OU
WHERE prix_unitaire = (
    SELECT MAX(prix_unitaire)  -- Garantit 1 valeur
    FROM produits
    WHERE categorie = 'Électronique'
)
```

### Piège 3 : Sous-requête corrélée lente

```sql
-- ❌ LENT : Corrélée dans SELECT
SELECT
    p.nom_produit,
    (SELECT COUNT(*) FROM details_commande WHERE id_produit = p.id_produit)
FROM produits p;

-- ✅ RAPIDE : LEFT JOIN + GROUP BY
SELECT
    p.nom_produit,
    COUNT(dc.id_detail) AS nb_ventes
FROM produits p
LEFT JOIN details_commande dc ON p.id_produit = dc.id_produit
GROUP BY p.id_produit, p.nom_produit;
```

### Piège 4 : Confusion sur le scope des alias

```sql
-- ❌ ERREUR : Alias de la sous-requête non accessible
SELECT nom
FROM (
    SELECT nom_produit AS nom, prix_unitaire
    FROM produits
) AS sub
WHERE prix_unitaire > 100;  -- Erreur : prix_unitaire n'existe pas dans le scope

-- ✅ CORRECT : Utiliser l'alias de la sous-requête
SELECT nom
FROM (
    SELECT nom_produit AS nom, prix_unitaire AS prix
    FROM produits
) AS sub
WHERE sub.prix > 100;
```

---

## Bonnes pratiques

### ✅ Quand utiliser les sous-requêtes

1. **Logique métier en étapes** : Calcul intermédiaire puis utilisation
2. **EXISTS pour test d'existence** : Plus clair que LEFT JOIN + IS NULL
3. **Agrégations dans WHERE/HAVING** : Comparaison avec moyenne, max, etc.
4. **Tables dérivées complexes** : Agrégations sur agrégations
5. **Lisibilité avant tout** : Si plus clair qu'une jointure complexe

### ❌ Quand éviter les sous-requêtes

1. **Sous-requêtes corrélées dans SELECT** : Préférer jointure
2. **NOT IN avec colonnes nullable** : Utiliser NOT EXISTS
3. **Performance critique** : Tester avec EXPLAIN
4. **Doublons acceptables** : Jointure directe plus simple

### Checklist optimisation

```sql
-- ✅ Vérifier avec EXPLAIN
EXPLAIN SELECT ... ;

-- ✅ Comparer avec version jointure
-- Si sous-requête corrélée, tester l'alternative jointure

-- ✅ Vérifier les index
SHOW INDEX FROM table;

-- ✅ Tester sur données volumineuses
-- Performance peut différer entre 100 et 1M lignes
```

---

## ✅ Points clés à retenir

1. **Sous-requêtes = requêtes imbriquées** – décomposent problèmes complexes en étapes logiques

2. **3 types par résultat** – scalaire (1 valeur), liste (N valeurs), table (N×M valeurs)

3. **4 emplacements possibles** – SELECT, FROM, WHERE, HAVING selon le besoin

4. **EXISTS plus performant que IN** – s'arrête à la première correspondance

5. **NOT IN dangereux avec NULL** – préférer NOT EXISTS systématiquement

6. **Corrélées vs non-corrélées** – corrélées exécutées N fois (lentes), non-corrélées 1 fois (rapides)

7. **Sous-requêtes dans SELECT souvent lentes** – préférer jointures avec agrégation

8. **Tables dérivées permettent alias dans WHERE** – contournent limitation HAVING

9. **ANY/ALL équivalent à MIN/MAX** – préférer MIN/MAX plus clairs

10. **Toujours vérifier avec EXPLAIN** – identifier sous-requêtes dépendantes et lentes

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Subqueries](https://mariadb.com/kb/en/subqueries/) – Documentation complète
- [📖 Optimizing Subqueries](https://mariadb.com/kb/en/subquery-optimizations/) – Techniques d'optimisation
- [📖 EXISTS vs IN](https://mariadb.com/kb/en/exists-to-in-optimization/) – Comparaison performance

### Articles approfondis
- [SQL Subqueries](https://modern-sql.com/feature/subqueries) – Patterns et best practices
- [Subquery Performance](https://use-the-index-luke.com/sql/where-clause/subqueries) – Guide performance

---

## ➡️ Section suivante

**[3.5 Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)](./05-operateurs-ensemblistes.md)**

La prochaine section couvre les opérateurs qui **combinent des résultats de plusieurs requêtes** :
- **UNION** et **UNION ALL** : Fusionner des résultats
- **INTERSECT** : Trouver l'intersection
- **EXCEPT** : Trouver la différence
- Différences avec les jointures
- Cas d'usage : consolidation de sources multiples, analyses comparatives

Les opérateurs ensemblistes complètent votre arsenal SQL intermédiaire ! 🎯

---


⏭️ [Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)](/03-requetes-sql-intermediaires/05-operateurs-ensemblistes.md)

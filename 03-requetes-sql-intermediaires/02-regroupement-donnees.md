🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 Regroupement de données (GROUP BY, HAVING)

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures
> **Prérequis** : Section 3.1 (Fonctions d'agrégation), maîtrise de COUNT, SUM, AVG, MIN, MAX

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Utiliser GROUP BY pour segmenter vos analyses par catégories
- Comprendre la différence fondamentale entre WHERE et HAVING
- Filtrer des résultats agrégés avec la clause HAVING
- Grouper sur plusieurs colonnes simultanément
- Éviter les erreurs courantes liées à GROUP BY (colonnes non agrégées)
- Optimiser les requêtes de regroupement pour la performance
- Appliquer GROUP BY à des cas d'usage métier complexes

---

## Introduction

La clause **GROUP BY** est l'outil qui transforme les fonctions d'agrégation en véritables analyses segmentées. Alors que les agrégations simples (section 3.1) calculent des statistiques sur **l'ensemble des données**, GROUP BY permet de calculer ces mêmes statistiques **par catégorie, par période, par segment**.

### Le problème que résout GROUP BY

Imaginez que vous voulez répondre à ces questions :
- *"Quel est le chiffre d'affaires **par catégorie de produit** ?"*
- *"Combien de commandes **par client** ?"*
- *"Quel est le panier moyen **par pays** ?"*

Sans GROUP BY, vous ne pourriez obtenir qu'un seul total global. Avec GROUP BY, vous obtenez **une ligne de résultat par groupe**, chacune avec ses propres agrégations.

### Analogie conceptuelle

Pensez à GROUP BY comme à l'action de **trier des cartes en piles** :
1. Vous prenez toutes vos lignes de données
2. Vous les répartissez en **groupes** selon un critère (catégorie, pays, client...)
3. Pour chaque groupe, vous calculez des statistiques (COUNT, SUM, AVG...)
4. Vous obtenez **une ligne de résultat par groupe**

---

## GROUP BY : Les fondamentaux

### Syntaxe de base

```sql
SELECT
    colonne_groupement,
    fonction_agregation(colonne)
FROM table
WHERE conditions_filtrage       -- Optionnel : filtre AVANT regroupement
GROUP BY colonne_groupement
HAVING condition_sur_agregat    -- Optionnel : filtre APRÈS regroupement
ORDER BY colonne                -- Optionnel : tri des résultats
```

### Règle d'or du GROUP BY

⚠️ **Règle absolue** : Toute colonne dans le SELECT qui n'est **pas dans une fonction d'agrégation** doit être dans la clause GROUP BY.

```sql
-- ✅ CORRECT : ville est dans GROUP BY
SELECT ville, COUNT(*)
FROM clients
GROUP BY ville;

-- ❌ ERREUR : ville n'est pas agrégée ni dans GROUP BY
SELECT ville, COUNT(*)
FROM clients;
-- Erreur : "Expression #1 of SELECT list is not in GROUP BY clause"
```

💡 **Pourquoi cette règle ?** Pour chaque groupe, MariaDB calcule **une seule ligne** de résultat. Si une colonne n'est pas agrégée et pas dans GROUP BY, MariaDB ne sait pas quelle valeur choisir parmi toutes celles du groupe.

---

## Exemples progressifs avec GROUP BY

### Exemple 1 : Groupement simple sur une colonne

**Question métier** : *Combien de clients par pays ?*

```sql
-- Compter les clients par pays
SELECT
    pays,
    COUNT(*) AS nombre_clients
FROM clients
GROUP BY pays
ORDER BY nombre_clients DESC;
```

**Résultat attendu** :
```
+----------------+------------------+
| pays           | nombre_clients   |
+----------------+------------------+
| France         |              847 |
| Belgique       |              203 |
| Suisse         |              142 |
| Canada         |               55 |
+----------------+------------------+
```

**Explication** :
1. MariaDB **regroupe** toutes les lignes ayant le même `pays`
2. Pour chaque groupe (pays), il **compte** le nombre de lignes
3. Le résultat contient **une ligne par pays**
4. ORDER BY trie les résultats par nombre décroissant

**Cas d'usage** : Analyse de la répartition géographique, ciblage marketing par pays.

### Exemple 2 : Plusieurs agrégations par groupe

**Question métier** : *Pour chaque catégorie de produit, obtenir le nombre de produits, prix moyen, min et max*

```sql
-- Statistiques complètes par catégorie
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    ROUND(AVG(prix_unitaire), 2) AS prix_moyen,
    MIN(prix_unitaire) AS prix_min,
    MAX(prix_unitaire) AS prix_max,
    SUM(stock) AS stock_total
FROM produits
GROUP BY categorie
ORDER BY nb_produits DESC;
```

**Résultat attendu** :
```
+------------------+--------------+-------------+----------+----------+--------------+
| categorie        | nb_produits  | prix_moyen  | prix_min | prix_max | stock_total  |
+------------------+--------------+-------------+----------+----------+--------------+
| Électronique     |          247 |      345.67 |    12.99 |  1299.99 |         4821 |
| Vêtements        |          189 |       54.23 |     9.99 |   249.99 |         8934 |
| Alimentation     |          156 |       12.45 |     2.99 |    89.99 |        12847 |
| Livres           |          134 |       18.90 |     5.99 |    59.99 |         3241 |
+------------------+--------------+-------------+----------+----------+--------------+
```

**Explication** :
- Chaque catégorie devient **un groupe distinct**
- Pour chaque groupe, MariaDB calcule **6 valeurs** (5 agrégations + la catégorie elle-même)
- Les agrégations sont **indépendantes** les unes des autres
- Une seule requête fournit une **vue d'ensemble complète**

**Cas d'usage** : Rapport de gestion des stocks, analyse de gammes de produits, pilotage des achats.

### Exemple 3 : Groupement avec filtrage préalable (WHERE)

**Question métier** : *Quel est le CA par catégorie pour les produits en stock ?*

```sql
-- CA par catégorie (uniquement produits en stock)
SELECT
    categorie,
    COUNT(*) AS nb_produits_en_stock,
    ROUND(SUM(prix_unitaire * stock), 2) AS valeur_stock
FROM produits
WHERE stock > 0  -- Filtre AVANT le regroupement
GROUP BY categorie
ORDER BY valeur_stock DESC;
```

**Résultat attendu** :
```
+------------------+-----------------------+--------------+
| categorie        | nb_produits_en_stock  | valeur_stock |
+------------------+-----------------------+--------------+
| Électronique     |                   234 |   847293.45  |
| Alimentation     |                   148 |   234847.92  |
| Vêtements        |                   172 |   184920.37  |
| Livres           |                   121 |    52847.18  |
+------------------+-----------------------+--------------+
```

**Explication** :
- Le **WHERE** filtre les lignes **avant** le regroupement
- Seuls les produits avec `stock > 0` sont inclus
- Ensuite, ces lignes filtrées sont **regroupées** par catégorie
- Les agrégations ne portent que sur les lignes qui ont passé le filtre WHERE

💡 **Ordre d'exécution** : WHERE → GROUP BY → Agrégations → HAVING → ORDER BY

### Exemple 4 : Groupement sur plusieurs colonnes

**Question métier** : *Nombre de clients par pays ET par ville*

```sql
-- Répartition géographique fine
SELECT
    pays,
    ville,
    COUNT(*) AS nombre_clients
FROM clients
WHERE ville IS NOT NULL
GROUP BY pays, ville
ORDER BY pays, nombre_clients DESC;
```

**Résultat attendu** :
```
+----------+------------------+------------------+
| pays     | ville            | nombre_clients   |
+----------+------------------+------------------+
| Belgique | Bruxelles        |              142 |
| Belgique | Liège            |               34 |
| Belgique | Anvers           |               27 |
| France   | Paris            |              284 |
| France   | Lyon             |              156 |
| France   | Marseille        |              123 |
| France   | Toulouse         |               89 |
| Suisse   | Genève           |               84 |
| Suisse   | Zurich           |               58 |
+----------+------------------+------------------+
```

**Explication** :
- GROUP BY sur **deux colonnes** : `pays, ville`
- Chaque **combinaison unique** (pays, ville) forme un groupe
- Paris-France et Lyon-France sont des groupes **différents**
- Le tri ORDER BY trie d'abord par pays, puis par nombre de clients décroissant

**Cas d'usage** : Analyse de densité géographique, ciblage local, logistique de livraison.

### Exemple 5 : Groupement avec calculs dérivés

**Question métier** : *CA par client avec calcul du panier moyen*

```sql
-- Analyse comportement clients
SELECT
    id_client,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total,
    ROUND(AVG(montant_total), 2) AS panier_moyen,
    MIN(date_commande) AS premiere_commande,
    MAX(date_commande) AS derniere_commande,
    DATEDIFF(MAX(date_commande), MIN(date_commande)) AS duree_relation_jours
FROM commandes
WHERE statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY id_client
HAVING nb_commandes >= 3  -- On verra HAVING en détail plus bas
ORDER BY ca_total DESC
LIMIT 10;
```

**Résultat attendu** :
```
+------------+---------------+----------+--------------+---------------------+---------------------+----------------------+
| id_client  | nb_commandes  | ca_total | panier_moyen | premiere_commande   | derniere_commande   | duree_relation_jours |
+------------+---------------+----------+--------------+---------------------+---------------------+----------------------+
|       1847 |           47  | 14283.45 |       303.90 | 2024-02-15 10:23:00 | 2025-12-08 15:42:00 |                  662 |
|       2934 |           38  | 12847.92 |       338.10 | 2023-11-20 08:15:00 | 2025-11-30 19:23:00 |                  741 |
|       5621 |           29  | 11293.18 |       389.42 | 2024-05-10 14:38:00 | 2025-12-10 11:05:00 |                  579 |
+------------+---------------+----------+--------------+---------------------+---------------------+----------------------+
```

**Explication** :
- Chaque client devient un groupe
- On calcule **7 métriques** par client
- `duree_relation_jours` est calculé à partir de MIN et MAX dates
- HAVING filtre les clients avec au moins 3 commandes (nous y reviendrons)
- LIMIT 10 ne garde que le top 10 des meilleurs clients

**Cas d'usage** : Segmentation RFM, identification des VIP clients, programmes de fidélité.

---

## WHERE vs HAVING : La différence fondamentale

C'est l'un des points les plus importants à comprendre en SQL.

### Principe de base

| Clause | Moment d'application | Fonction | Peut utiliser des agrégations ? |
|--------|---------------------|----------|--------------------------------|
| **WHERE** | **AVANT** le regroupement | Filtre les **lignes individuelles** | ❌ NON |
| **HAVING** | **APRÈS** le regroupement | Filtre les **groupes (résultats agrégés)** | ✅ OUI |

### Ordre d'exécution SQL

```
1. FROM       : Sélection des tables
2. WHERE      : Filtrage des lignes individuelles
3. GROUP BY   : Regroupement
4. Agrégations: COUNT, SUM, AVG, etc.
5. HAVING     : Filtrage des groupes
6. SELECT     : Projection des colonnes
7. ORDER BY   : Tri des résultats
8. LIMIT      : Limitation du nombre de résultats
```

### Exemple 6 : Comparaison WHERE vs HAVING

**Scénario** : Analyser les commandes par client

```sql
-- ❌ ERREUR : Impossible d'utiliser COUNT dans WHERE
SELECT
    id_client,
    COUNT(*) AS nb_commandes
FROM commandes
WHERE COUNT(*) > 5  -- ERREUR : agrégation dans WHERE
GROUP BY id_client;
-- Erreur : "Invalid use of group function"
```

```sql
-- ✅ CORRECT : Utiliser HAVING pour filtrer sur agrégations
SELECT
    id_client,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
GROUP BY id_client
HAVING COUNT(*) > 5  -- Filtre APRÈS regroupement
ORDER BY nb_commandes DESC;
```

**Résultat attendu** :
```
+------------+---------------+-----------+
| id_client  | nb_commandes  | ca_total  |
+------------+---------------+-----------+
|       1847 |           47  | 14283.45  |
|       2934 |           38  | 12847.92  |
|       5621 |           29  | 11293.18  |
|       3847 |           23  |  8294.73  |
|       8472 |           19  |  7183.92  |
+------------+---------------+-----------+
```

**Explication** :
- WHERE filtre les **lignes** avant regroupement (impossible d'y mettre COUNT)
- HAVING filtre les **groupes** après agrégation (on peut utiliser COUNT)
- Seuls les clients avec plus de 5 commandes apparaissent dans le résultat

### Exemple 7 : Combiner WHERE et HAVING

**Question métier** : *Clients avec plus de 3 commandes livrées et CA > 1000 €*

```sql
-- Filtrage à deux niveaux
SELECT
    id_client,
    COUNT(*) AS nb_commandes_livrees,
    SUM(montant_total) AS ca_total,
    ROUND(AVG(montant_total), 2) AS panier_moyen
FROM commandes
WHERE statut = 'livrée'             -- Filtre AVANT : uniquement livrées
GROUP BY id_client
HAVING COUNT(*) > 3                 -- Filtre APRÈS : plus de 3 commandes
   AND SUM(montant_total) > 1000    -- ET CA > 1000
ORDER BY ca_total DESC;
```

**Résultat attendu** :
```
+------------+------------------------+-----------+--------------+
| id_client  | nb_commandes_livrees   | ca_total  | panier_moyen |
+------------+------------------------+-----------+--------------+
|       1847 |                     42 | 13847.35  |       329.70 |
|       2934 |                     35 | 12193.82  |       348.39 |
|       5621 |                     27 | 10847.28  |       401.75 |
+------------+------------------------+-----------+--------------+
```

**Explication détaillée du flux** :
1. **FROM commandes** : On part de toutes les commandes
2. **WHERE statut = 'livrée'** : On ne garde que les commandes livrées (filtrage lignes)
3. **GROUP BY id_client** : On regroupe par client
4. **Agrégations** : On calcule COUNT, SUM, AVG pour chaque client
5. **HAVING COUNT(*) > 3 AND SUM(...) > 1000** : On ne garde que les clients avec >3 commandes ET CA>1000
6. **ORDER BY ca_total DESC** : On trie par CA décroissant

💡 **Règle pratique** :
- Utilisez **WHERE** pour filtrer sur des conditions qui ne nécessitent pas d'agrégation
- Utilisez **HAVING** pour filtrer sur des résultats d'agrégation

### Exemple 8 : HAVING avec alias

⚠️ **Attention** : Selon la version de MariaDB, l'utilisation d'alias dans HAVING peut varier.

```sql
-- ✅ FONCTIONNE dans MariaDB 11.8
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    AVG(prix_unitaire) AS prix_moyen
FROM produits
GROUP BY categorie
HAVING nb_produits > 50           -- Utilisation de l'alias
   AND prix_moyen > 100;
```

```sql
-- ✅ TOUJOURS CORRECT (portable)
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    AVG(prix_unitaire) AS prix_moyen
FROM produits
GROUP BY categorie
HAVING COUNT(*) > 50              -- Répétition de l'expression
   AND AVG(prix_unitaire) > 100;
```

**Résultat attendu** :
```
+------------------+--------------+------------+
| categorie        | nb_produits  | prix_moyen |
+------------------+--------------+------------+
| Électronique     |          247 |     345.67 |
+------------------+--------------+------------+
```

💡 **Bonne pratique** : Pour une compatibilité maximale, répétez l'expression complète dans HAVING plutôt que d'utiliser l'alias.

---

## Cas d'usage avancés

### Exemple 9 : Analyse temporelle par mois

**Question métier** : *CA mensuel pour l'année 2025*

```sql
-- Chiffre d'affaires par mois
SELECT
    YEAR(date_commande) AS annee,
    MONTH(date_commande) AS mois,
    DATE_FORMAT(date_commande, '%Y-%m') AS periode,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_mensuel,
    ROUND(AVG(montant_total), 2) AS panier_moyen
FROM commandes
WHERE YEAR(date_commande) = 2025
  AND statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY YEAR(date_commande), MONTH(date_commande)
ORDER BY annee, mois;
```

**Résultat attendu** :
```
+--------+------+----------+---------------+-------------+--------------+
| annee  | mois | periode  | nb_commandes  | ca_mensuel  | panier_moyen |
+--------+------+----------+---------------+-------------+--------------+
|   2025 |    1 | 2025-01  |         1147  |  184729.45  |       161.02 |
|   2025 |    2 | 2025-02  |         1089  |  172847.92  |       158.73 |
|   2025 |    3 | 2025-03  |         1253  |  203948.18  |       162.78 |
|   2025 |    4 | 2025-04  |         1198  |  195847.37  |       163.49 |
|   2025 |    5 | 2025-05  |         1342  |  219374.82  |       163.48 |
+--------+------+----------+---------------+-------------+--------------+
```

**Explication** :
- GROUP BY sur `YEAR()` ET `MONTH()` pour créer des groupes mensuels
- `DATE_FORMAT()` crée un affichage lisible (YYYY-MM)
- Permet d'analyser les **tendances** et **saisonnalité**
- WHERE filtre d'abord sur l'année pour la performance

**Cas d'usage** : Rapports mensuels, prévisions, analyse de saisonnalité, pilotage commercial.

### Exemple 10 : TOP N par catégorie

**Question métier** : *Les 3 produits les plus chers par catégorie*

💡 **Note** : Cette requête utilise des concepts avancés (sous-requêtes) que nous verrons en section 3.4.

```sql
-- Top 3 produits par catégorie avec GROUP BY
SELECT
    categorie,
    nom_produit,
    prix_unitaire
FROM (
    SELECT
        categorie,
        nom_produit,
        prix_unitaire,
        ROW_NUMBER() OVER (PARTITION BY categorie ORDER BY prix_unitaire DESC) AS rang
    FROM produits
) AS produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

**Alternative sans window functions** (plus simple pour ce niveau) :

```sql
-- Produits au-dessus du prix moyen de leur catégorie
SELECT
    p.categorie,
    p.nom_produit,
    p.prix_unitaire,
    stats.prix_moyen_categorie
FROM produits p
JOIN (
    SELECT
        categorie,
        AVG(prix_unitaire) AS prix_moyen_categorie
    FROM produits
    GROUP BY categorie
) AS stats ON p.categorie = stats.categorie
WHERE p.prix_unitaire > stats.prix_moyen_categorie
ORDER BY p.categorie, p.prix_unitaire DESC;
```

**Cas d'usage** : Identification des produits premium, gestion de gammes.

### Exemple 11 : Détection d'anomalies

**Question métier** : *Clients avec un panier moyen anormalement élevé (> 3x la moyenne globale)*

```sql
-- Détection de comportements atypiques
SELECT
    c.id_client,
    c.nom,
    COUNT(cmd.id_commande) AS nb_commandes,
    ROUND(AVG(cmd.montant_total), 2) AS panier_moyen_client,
    (SELECT ROUND(AVG(montant_total), 2) FROM commandes) AS panier_moyen_global
FROM clients c
JOIN commandes cmd ON c.id_client = cmd.id_client
WHERE cmd.statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY c.id_client, c.nom
HAVING AVG(cmd.montant_total) > (
    SELECT AVG(montant_total) * 3
    FROM commandes
    WHERE statut IN ('confirmée', 'expédiée', 'livrée')
)
ORDER BY panier_moyen_client DESC;
```

**Résultat attendu** :
```
+------------+------------------+---------------+-----------------------+----------------------+
| id_client  | nom              | nb_commandes  | panier_moyen_client   | panier_moyen_global  |
+------------+------------------+---------------+-----------------------+----------------------+
|       5621 | Martin Dubois    |           29  |                389.42 |               148.23 |
|       8472 | Sophie Bernard   |           19  |                378.10 |               148.23 |
|       2934 | Pierre Laurent   |           38  |                338.10 |               148.23 |
+------------+------------------+---------------+-----------------------+----------------------+
```

**Cas d'usage** : Détection de fraude, identification de VIP, ciblage marketing personnalisé.

### Exemple 12 : Cohort analysis (analyse de cohortes)

**Question métier** : *Taux de rétention par mois d'inscription*

```sql
-- Cohortes par mois d'inscription
SELECT
    DATE_FORMAT(c.date_inscription, '%Y-%m') AS cohorte_mois,
    COUNT(DISTINCT c.id_client) AS clients_total,
    COUNT(DISTINCT CASE
        WHEN cmd.date_commande >= DATE_ADD(c.date_inscription, INTERVAL 30 DAY)
        THEN c.id_client
    END) AS clients_actifs_mois_1,
    ROUND(100.0 * COUNT(DISTINCT CASE
        WHEN cmd.date_commande >= DATE_ADD(c.date_inscription, INTERVAL 30 DAY)
        THEN c.id_client
    END) / COUNT(DISTINCT c.id_client), 2) AS taux_retention_pct
FROM clients c
LEFT JOIN commandes cmd ON c.id_client = cmd.id_client
WHERE c.date_inscription >= '2025-01-01'
GROUP BY DATE_FORMAT(c.date_inscription, '%Y-%m')
ORDER BY cohorte_mois;
```

**Résultat attendu** :
```
+---------------+----------------+-----------------------+---------------------+
| cohorte_mois  | clients_total  | clients_actifs_mois_1 | taux_retention_pct  |
+---------------+----------------+-----------------------+---------------------+
| 2025-01       |            142 |                    89 |               62.68 |
| 2025-02       |            158 |                    97 |               61.39 |
| 2025-03       |            173 |                   112 |               64.74 |
+---------------+----------------+-----------------------+---------------------+
```

**Cas d'usage** : Analyse de rétention, efficacité de l'onboarding, prévisions de churn.

---

## Groupement sur expressions et fonctions

Vous pouvez grouper sur des **expressions calculées**, pas seulement sur des colonnes brutes.

### Exemple 13 : Groupement sur une tranche de valeurs

**Question métier** : *Répartition des produits par tranche de prix*

```sql
-- Segmentation par tranche de prix
SELECT
    CASE
        WHEN prix_unitaire < 20 THEN '< 20€'
        WHEN prix_unitaire < 50 THEN '20-50€'
        WHEN prix_unitaire < 100 THEN '50-100€'
        WHEN prix_unitaire < 500 THEN '100-500€'
        ELSE '> 500€'
    END AS tranche_prix,
    COUNT(*) AS nb_produits,
    SUM(stock) AS stock_total
FROM produits
GROUP BY
    CASE
        WHEN prix_unitaire < 20 THEN '< 20€'
        WHEN prix_unitaire < 50 THEN '20-50€'
        WHEN prix_unitaire < 100 THEN '50-100€'
        WHEN prix_unitaire < 500 THEN '100-500€'
        ELSE '> 500€'
    END
ORDER BY MIN(prix_unitaire);
```

**Résultat attendu** :
```
+--------------+--------------+--------------+
| tranche_prix | nb_produits  | stock_total  |
+--------------+--------------+--------------+
| < 20€        |          247 |        15842 |
| 20-50€       |          189 |         8934 |
| 50-100€      |          134 |         4287 |
| 100-500€     |           89 |         2184 |
| > 500€       |           67 |          847 |
+--------------+--------------+--------------+
```

**Explication** :
- GROUP BY sur une **expression CASE** (pas une colonne physique)
- ⚠️ **Important** : Il faut **répéter l'expression** dans GROUP BY (pas d'alias possible ici)
- MariaDB évalue l'expression CASE pour chaque ligne, puis groupe

**Cas d'usage** : Segmentation tarifaire, analyse de gammes, pricing strategy.

### Exemple 14 : Groupement par jour de la semaine

**Question métier** : *Quel jour de la semaine génère le plus de commandes ?*

```sql
-- Analyse par jour de la semaine
SELECT
    DAYOFWEEK(date_commande) AS jour_numero,
    DAYNAME(date_commande) AS jour_nom,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_jour,
    ROUND(AVG(montant_total), 2) AS panier_moyen
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
  AND statut IN ('confirmée', 'expédiée', 'livrée')
GROUP BY DAYOFWEEK(date_commande), DAYNAME(date_commande)
ORDER BY jour_numero;
```

**Résultat attendu** :
```
+--------------+-----------+---------------+-----------+--------------+
| jour_numero  | jour_nom  | nb_commandes  | ca_jour   | panier_moyen |
+--------------+-----------+---------------+-----------+--------------+
|            1 | Sunday    |          847  | 124847.35 |       147.42 |
|            2 | Monday    |         1293  | 195283.91 |       151.02 |
|            3 | Tuesday   |         1384  | 208472.83 |       150.63 |
|            4 | Wednesday |         1402  | 213847.29 |       152.54 |
|            5 | Thursday  |         1347  | 204938.47 |       152.15 |
|            6 | Friday    |         1489  | 231947.82 |       155.78 |
|            7 | Saturday  |         1124  | 172847.91 |       153.78 |
+--------------+-----------+---------------+-----------+--------------+
```

**Cas d'usage** : Planification des opérations (staffing), campagnes marketing ciblées, prévisions de charge.

---

## Optimisation et performance

### Index et GROUP BY

Les **index** sont cruciaux pour la performance des requêtes GROUP BY.

#### ✅ Bonne pratique 1 : Index sur colonnes de regroupement

```sql
-- Sans index : Scan complet + tri temporaire (LENT)
SELECT categorie, COUNT(*)
FROM produits
GROUP BY categorie;

-- Avec index : Scan d'index ordonné (RAPIDE)
CREATE INDEX idx_produits_categorie ON produits(categorie);

SELECT categorie, COUNT(*)
FROM produits
GROUP BY categorie;
```

**Impact** :
- **Sans index** : MariaDB doit scanner toute la table puis créer une table temporaire pour grouper
- **Avec index** : MariaDB peut lire les données **déjà groupées** dans l'index

💡 **Vérifiez avec EXPLAIN** :

```sql
EXPLAIN SELECT categorie, COUNT(*)
FROM produits
GROUP BY categorie;
```

Recherchez dans le résultat :
- `Using temporary` → Mauvais signe (table temporaire)
- `Using filesort` → Mauvais signe (tri sur disque)
- `Using index` → Bon signe (index covering)

#### ✅ Bonne pratique 2 : Ordre des colonnes dans GROUP BY multi-colonnes

```sql
-- Index composite sur (pays, ville)
CREATE INDEX idx_clients_pays_ville ON clients(pays, ville);

-- ✅ BON : Respecte l'ordre de l'index
SELECT pays, ville, COUNT(*)
FROM clients
GROUP BY pays, ville;

-- ⚠️ MOINS OPTIMAL : Ordre inversé
SELECT pays, ville, COUNT(*)
FROM clients
GROUP BY ville, pays;
```

**Règle** : L'ordre dans GROUP BY devrait **correspondre à l'ordre** des colonnes dans l'index pour une efficacité maximale.

#### ✅ Bonne pratique 3 : Limiter les groupes avec WHERE

```sql
-- ❌ LENT : Groupe PUIS filtre
SELECT categorie, COUNT(*) as nb
FROM produits
GROUP BY categorie
HAVING nb > 100;

-- ✅ RAPIDE : Filtre avec WHERE quand possible
-- (Seulement si le filtre est sur une colonne non agrégée)
SELECT categorie, COUNT(*)
FROM produits
WHERE stock > 0  -- Filtre avant regroupement
GROUP BY categorie;
```

**Principe** : Réduisez le volume de données **avant** le regroupement avec WHERE.

### Gestion de la mémoire

Pour les très grands groupements :

```sql
-- Variables à surveiller
SHOW VARIABLES LIKE 'tmp_table_size';
SHOW VARIABLES LIKE 'max_heap_table_size';

-- Si beaucoup de "Created_tmp_disk_tables"
SHOW STATUS LIKE 'Created_tmp%';
```

Si MariaDB crée trop de tables temporaires sur disque :
1. Augmenter `tmp_table_size` et `max_heap_table_size`
2. Ajouter des index sur les colonnes de regroupement
3. Filtrer plus agressivement avec WHERE

---

## Erreurs courantes et solutions

### Erreur 1 : Colonne non agrégée manquante dans GROUP BY

```sql
-- ❌ ERREUR
SELECT nom, categorie, COUNT(*)
FROM produits
GROUP BY categorie;
-- Error: "Expression #1 of SELECT list is not in GROUP BY clause"
```

**Solution** :
```sql
-- ✅ Agrégation sur nom
SELECT MIN(nom) as exemple_nom, categorie, COUNT(*)
FROM produits
GROUP BY categorie;

-- ✅ Ou ajouter nom au GROUP BY (si logique métier)
SELECT nom, categorie, COUNT(*)
FROM produits
GROUP BY categorie, nom;
```

### Erreur 2 : Utilisation d'agrégation dans WHERE

```sql
-- ❌ ERREUR
SELECT categorie, COUNT(*) as nb
FROM produits
WHERE COUNT(*) > 10
GROUP BY categorie;
-- Error: "Invalid use of group function"
```

**Solution** :
```sql
-- ✅ Utiliser HAVING
SELECT categorie, COUNT(*) as nb
FROM produits
GROUP BY categorie
HAVING COUNT(*) > 10;
```

### Erreur 3 : GROUP BY sans agrégation

```sql
-- ⚠️ INUTILE : GROUP BY sans fonction d'agrégation
SELECT categorie
FROM produits
GROUP BY categorie;

-- ✅ MIEUX : Utilisez DISTINCT
SELECT DISTINCT categorie
FROM produits;
```

**Explication** : Si vous n'utilisez pas de fonction d'agrégation, `DISTINCT` est plus clair et souvent plus performant.

### Erreur 4 : Confusion entre COUNT(*) et COUNT(colonne)

```sql
-- Résultats différents !
SELECT
    categorie,
    COUNT(*) as total_lignes,        -- Toutes les lignes
    COUNT(description) as avec_desc  -- Seulement non NULL
FROM produits
GROUP BY categorie;
```

💡 **Rappel** : COUNT(*) compte toutes les lignes, COUNT(colonne) ignore les NULL.

---

## Cas d'usage métier avancés

### Cas 1 : Rapport de ventes multi-dimensionnel

```sql
-- Dashboard complet : ventes par catégorie et statut
SELECT
    p.categorie,
    c.statut,
    COUNT(DISTINCT c.id_commande) AS nb_commandes,
    COUNT(DISTINCT c.id_client) AS nb_clients_uniques,
    SUM(dc.quantite) AS quantite_totale,
    ROUND(SUM(dc.quantite * dc.prix_unitaire), 2) AS ca_total
FROM commandes c
JOIN details_commande dc ON c.id_commande = dc.id_commande
JOIN produits p ON dc.id_produit = p.id_produit
WHERE c.date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY p.categorie, c.statut
ORDER BY p.categorie, c.statut;
```

**Application** : Tableau de bord opérationnel avec vision multi-axes.

### Cas 2 : Analyse ABC (Pareto)

**Question métier** : *Quels produits représentent 80% du CA ?*

```sql
-- Calcul de contribution cumulée
SELECT
    id_produit,
    SUM(quantite * prix_unitaire) AS ca_produit,
    ROUND(100.0 * SUM(quantite * prix_unitaire) / (
        SELECT SUM(quantite * prix_unitaire)
        FROM details_commande
    ), 2) AS pct_ca
FROM details_commande
GROUP BY id_produit
ORDER BY ca_produit DESC;
```

**Application** : Gestion de stocks, focus commercial sur produits à forte valeur.

### Cas 3 : Taux de transformation par canal

```sql
-- Si on avait une colonne canal_acquisition
SELECT
    canal_acquisition,
    COUNT(DISTINCT id_client) AS clients_inscrits,
    COUNT(DISTINCT CASE WHEN nb_commandes > 0 THEN id_client END) AS clients_acheteurs,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN nb_commandes > 0 THEN id_client END) /
          COUNT(DISTINCT id_client), 2) AS taux_conversion_pct
FROM (
    SELECT
        cl.id_client,
        cl.canal_acquisition,
        COUNT(cmd.id_commande) AS nb_commandes
    FROM clients cl
    LEFT JOIN commandes cmd ON cl.id_client = cmd.id_client
    GROUP BY cl.id_client, cl.canal_acquisition
) AS stats_clients
GROUP BY canal_acquisition
ORDER BY taux_conversion_pct DESC;
```

**Application** : Optimisation des investissements marketing, ROI par canal.

---

## ✅ Points clés à retenir

1. **GROUP BY segmente les agrégations** – transforme les totaux globaux en totaux par catégorie/groupe

2. **Règle d'or** : Toute colonne dans SELECT qui n'est pas agrégée doit être dans GROUP BY

3. **WHERE filtre avant, HAVING filtre après** le regroupement – distinction absolue à maîtriser

4. **WHERE ne peut pas utiliser d'agrégations**, HAVING le peut – COUNT, SUM, AVG dans HAVING uniquement

5. **GROUP BY sur plusieurs colonnes** crée des groupes pour chaque combinaison unique

6. **GROUP BY peut utiliser des expressions** (CASE, fonctions de dates) mais il faut répéter l'expression

7. **Index sur colonnes de GROUP BY** améliore drastiquement les performances

8. **L'ordre dans GROUP BY multi-colonnes** devrait correspondre à l'ordre de l'index

9. **Filtrer avec WHERE avant GROUP BY** réduit le volume de données à grouper

10. **EXPLAIN est essentiel** pour vérifier que GROUP BY utilise les index (éviter "Using temporary")

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 GROUP BY Clause](https://mariadb.com/kb/en/select/#group-by) – Syntaxe et comportement détaillé
- [📖 HAVING Clause](https://mariadb.com/kb/en/select/#having) – Filtrage post-agrégation
- [📖 Aggregate Functions](https://mariadb.com/kb/en/aggregate-functions/) – Liste complète des fonctions

### Articles approfondis
- [SQL GROUP BY Explained](https://modern-sql.com/feature/group-by) – Guide avancé
- [Optimizing GROUP BY](https://use-the-index-luke.com/sql/group-by) – Techniques de performance

### Outils
- [EXPLAIN Analyzer](https://mariadb.org/explain-analyzer/) – Visualiser les plans d'exécution GROUP BY

---

## ➡️ Section suivante

**[3.3 Jointures (INNER, LEFT, RIGHT, CROSS, Self-Join)](./03-jointures.md)**

Maintenant que vous maîtrisez les agrégations et le regroupement sur une seule table, la section suivante vous apprendra à **croiser les données de plusieurs tables**. Les jointures sont le cœur du modèle relationnel et vous permettront de :
- Combiner clients et commandes pour des analyses complètes
- Croiser produits et ventes pour identifier les best-sellers
- Détecter les clients sans commande (LEFT JOIN)
- Créer toutes les combinaisons possibles (CROSS JOIN)
- Comparer des lignes au sein d'une même table (Self-Join)

Les jointures, combinées avec GROUP BY, débloquent la puissance complète de SQL ! 🔗

---


⏭️ [Jointures](/03-requetes-sql-intermediaires/03-jointures.md)

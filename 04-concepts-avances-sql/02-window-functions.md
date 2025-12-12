🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 Window Functions (Fonctions de Fenêtre)

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Maîtrise des jointures, agrégations (GROUP BY, HAVING), sous-requêtes

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le concept de fenêtre (window) et son fonctionnement
- Utiliser les clauses OVER, PARTITION BY et ORDER BY
- Appliquer les fonctions de fenêtre pour des analyses avancées
- Maîtriser les frames de fenêtre (ROWS, RANGE, GROUPS)
- Résoudre des problèmes analytiques complexes sans sous-requêtes
- Optimiser les requêtes en remplaçant les agrégations multiples

---

## Introduction

Les **Window Functions** (fonctions de fenêtre ou fonctions analytiques) sont l'une des fonctionnalités SQL les plus puissantes pour l'analyse de données. Introduites dans le standard SQL:2003 et supportées par MariaDB depuis la version 10.2, elles permettent d'effectuer des calculs sur un ensemble de lignes **sans regrouper les résultats**, contrairement à `GROUP BY`.

### Pourquoi utiliser les Window Functions ?

**Problème classique :**
```sql
-- Sans window functions : calculer le salaire moyen par département
-- tout en conservant les détails de chaque employé
SELECT
    e.nom,
    e.salaire,
    e.departement_id,
    (SELECT AVG(salaire)
     FROM employes e2
     WHERE e2.departement_id = e.departement_id) AS salaire_moyen_dept
FROM employes e;
```

**Avec window functions :**
```sql
-- Beaucoup plus lisible et performant !
SELECT
    nom,
    salaire,
    departement_id,
    AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen_dept
FROM employes;
```

💡 **Avantages clés :**
- Pas de sous-requêtes corrélées (meilleures performances)
- Code plus lisible et maintenable
- Possibilité de combiner plusieurs analyses dans une seule requête
- Accès aux lignes précédentes/suivantes (LAG/LEAD)
- Calculs de rangs, percentiles, moyennes mobiles

---

## Syntaxe générale des Window Functions

### Structure de base

```sql
fonction_fenetre([arguments]) OVER (
    [PARTITION BY expression [, ...]]
    [ORDER BY expression [ASC|DESC] [, ...]]
    [frame_clause]
)
```

**Composants :**
- **fonction_fenetre** : La fonction à appliquer (ROW_NUMBER, SUM, AVG, etc.)
- **OVER** : Mot-clé obligatoire définissant la fenêtre
- **PARTITION BY** : Divise les données en partitions (optionnel)
- **ORDER BY** : Définit l'ordre dans la partition (optionnel mais souvent nécessaire)
- **frame_clause** : Définit le sous-ensemble de lignes dans la partition (optionnel)

### Différence fondamentale avec GROUP BY

```sql
-- Avec GROUP BY : perte des détails individuels
SELECT
    departement_id,
    AVG(salaire) AS salaire_moyen
FROM employes
GROUP BY departement_id;
-- Résultat : 1 ligne par département

-- Avec Window Function : conservation des détails
SELECT
    nom,
    departement_id,
    salaire,
    AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen
FROM employes;
-- Résultat : 1 ligne par employé + la moyenne de son département
```

---

## PARTITION BY : Diviser les données en fenêtres

La clause **PARTITION BY** divise le jeu de résultats en partitions indépendantes. Chaque partition est traitée séparément par la fonction de fenêtre.

### Exemple 1 : Calculs par catégorie

```sql
-- Contexte : Base de données e-commerce
CREATE TABLE produits (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    categorie VARCHAR(50),
    prix DECIMAL(10,2),
    quantite_stock INT
);

-- Analyse : Prix moyen et total des stocks par catégorie
SELECT
    nom,
    categorie,
    prix,
    quantite_stock,
    -- Moyenne des prix dans la même catégorie
    AVG(prix) OVER (PARTITION BY categorie) AS prix_moyen_categorie,
    -- Nombre de produits dans la même catégorie
    COUNT(*) OVER (PARTITION BY categorie) AS nb_produits_categorie,
    -- Valeur totale du stock de la catégorie
    SUM(prix * quantite_stock) OVER (PARTITION BY categorie) AS valeur_stock_categorie
FROM produits
ORDER BY categorie, prix DESC;
```

**Résultat conceptuel :**
```
| nom         | categorie    | prix  | prix_moyen_categorie | nb_produits_categorie  |
|-------------|--------------|-------|----------------------|------------------------|
| iPhone 15   | Smartphones  | 999   | 649.67               | 3                      |
| Galaxy S24  | Smartphones  | 899   | 649.67               | 3                      |
| Redmi Note  | Smartphones  | 299   | 649.67               | 3                      |
| MacBook Pro | Ordinateurs  | 2499  | 1499.50              | 2                      |
| Dell XPS    | Ordinateurs  | 1499  | 1499.50              | 2                      |
```

💡 **Conseil :** Pensez à PARTITION BY comme un GROUP BY qui ne réduit pas le nombre de lignes.

### Exemple 2 : Partition multiple

```sql
-- Analyse par région ET catégorie
SELECT
    nom,
    region,
    categorie,
    prix,
    AVG(prix) OVER (PARTITION BY region, categorie) AS prix_moyen_region_cat,
    MAX(prix) OVER (PARTITION BY region) AS prix_max_region
FROM produits
ORDER BY region, categorie;
```

---

## ORDER BY : Définir l'ordre dans la fenêtre

La clause **ORDER BY** définit l'ordre des lignes **dans chaque partition**. Cet ordre est crucial pour :
- Les fonctions de rang (ROW_NUMBER, RANK, DENSE_RANK)
- Les fonctions de valeur (LAG, LEAD, FIRST_VALUE, LAST_VALUE)
- Les frames de fenêtre (calculs cumulatifs, moyennes mobiles)

### Exemple 3 : Classement et cumuls

```sql
-- Contexte : Ventes mensuelles
CREATE TABLE ventes (
    id INT PRIMARY KEY,
    vendeur VARCHAR(50),
    mois DATE,
    montant DECIMAL(10,2)
);

-- Analyse : Évolution des ventes par vendeur
SELECT
    vendeur,
    mois,
    montant,
    -- Numéro de ligne par vendeur (ordre chronologique)
    ROW_NUMBER() OVER (PARTITION BY vendeur ORDER BY mois) AS num_mois,
    -- Montant cumulé par vendeur
    SUM(montant) OVER (
        PARTITION BY vendeur
        ORDER BY mois
    ) AS montant_cumule,
    -- Moyenne mobile sur 3 mois
    AVG(montant) OVER (
        PARTITION BY vendeur
        ORDER BY mois
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moyenne_mobile_3mois
FROM ventes
ORDER BY vendeur, mois;
```

**Résultat conceptuel :**
```
| vendeur | mois       | montant | num_mois | montant_cumule | moyenne_mobile_3mois |
|---------|------------|---------|----------|----------------|----------------------|
| Alice   | 2024-01-01 | 10000   | 1        | 10000          | 10000.00             |
| Alice   | 2024-02-01 | 12000   | 2        | 22000          | 11000.00             |
| Alice   | 2024-03-01 | 11000   | 3        | 33000          | 11000.00             |
| Alice   | 2024-04-01 | 13000   | 4        | 46000          | 12000.00             |
```

⚠️ **Attention :** Sans ORDER BY, l'ordre des lignes dans la fenêtre est **non déterministe**, ce qui peut produire des résultats incohérents pour certaines fonctions.

---

## Types de fonctions de fenêtre

MariaDB supporte trois catégories de fonctions de fenêtre :

### 1. Fonctions d'agrégation (avec OVER)

Toutes les fonctions d'agrégation classiques peuvent être utilisées comme window functions :

```sql
-- Fonctions d'agrégation en mode fenêtre
SELECT
    departement_id,
    nom,
    salaire,
    -- Agrégations sur la fenêtre
    SUM(salaire) OVER (PARTITION BY departement_id) AS masse_salariale_dept,
    AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen_dept,
    MIN(salaire) OVER (PARTITION BY departement_id) AS salaire_min_dept,
    MAX(salaire) OVER (PARTITION BY departement_id) AS salaire_max_dept,
    COUNT(*) OVER (PARTITION BY departement_id) AS nb_employes_dept,
    -- Écart par rapport à la moyenne du département
    salaire - AVG(salaire) OVER (PARTITION BY departement_id) AS ecart_moyenne
FROM employes;
```

### 2. Fonctions de rang

Ces fonctions attribuent un rang à chaque ligne dans la partition :

- **ROW_NUMBER()** : Numéro séquentiel unique (1, 2, 3, 4...)
- **RANK()** : Rang avec saut en cas d'égalité (1, 2, 2, 4...)
- **DENSE_RANK()** : Rang sans saut en cas d'égalité (1, 2, 2, 3...)
- **NTILE(n)** : Découpe en n groupes équitables

```sql
-- Classement des employés par salaire dans leur département
SELECT
    departement_id,
    nom,
    salaire,
    ROW_NUMBER() OVER (PARTITION BY departement_id ORDER BY salaire DESC) AS row_num,
    RANK() OVER (PARTITION BY departement_id ORDER BY salaire DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY departement_id ORDER BY salaire DESC) AS dense_rank,
    NTILE(4) OVER (PARTITION BY departement_id ORDER BY salaire DESC) AS quartile
FROM employes;
```

**Résultat illustratif (département Marketing) :**
```
| nom    | salaire | row_num | rank | dense_rank | quartile |
|--------|---------|---------|------|------------|----------|
| Bob    | 75000   | 1       | 1    | 1          | 1        |
| Alice  | 70000   | 2       | 2    | 2          | 1        |
| Claire | 70000   | 3       | 2    | 2          | 2        |
| David  | 65000   | 4       | 4    | 3          | 2        |
| Eve    | 60000   | 5       | 5    | 4          | 3        |
| Frank  | 55000   | 6       | 6    | 5          | 3        |
| Grace  | 50000   | 7       | 7    | 6          | 4        |
| Hugo   | 45000   | 8       | 8    | 7          | 4        |
```

### 3. Fonctions de valeur

Ces fonctions accèdent aux valeurs d'autres lignes dans la fenêtre :

- **LAG(col, offset)** : Valeur de la ligne précédente
- **LEAD(col, offset)** : Valeur de la ligne suivante
- **FIRST_VALUE(col)** : Première valeur de la fenêtre
- **LAST_VALUE(col)** : Dernière valeur de la fenêtre
- **NTH_VALUE(col, n)** : n-ième valeur de la fenêtre

```sql
-- Analyse de l'évolution des prix d'un produit
SELECT
    produit_id,
    date_prix,
    prix,
    -- Prix précédent (lag = retard)
    LAG(prix, 1) OVER (PARTITION BY produit_id ORDER BY date_prix) AS prix_precedent,
    -- Prix suivant (lead = avance)
    LEAD(prix, 1) OVER (PARTITION BY produit_id ORDER BY date_prix) AS prix_suivant,
    -- Variation absolue
    prix - LAG(prix, 1) OVER (PARTITION BY produit_id ORDER BY date_prix) AS variation,
    -- Variation en pourcentage
    ROUND(
        (prix - LAG(prix, 1) OVER (PARTITION BY produit_id ORDER BY date_prix)) /
        LAG(prix, 1) OVER (PARTITION BY produit_id ORDER BY date_prix) * 100,
        2
    ) AS variation_pct,
    -- Premier prix historique
    FIRST_VALUE(prix) OVER (PARTITION BY produit_id ORDER BY date_prix) AS prix_initial
FROM historique_prix
ORDER BY produit_id, date_prix;
```

---

## Frames de fenêtre : ROWS, RANGE, GROUPS

Les **frames** (cadres) définissent un **sous-ensemble de lignes** dans la partition pour le calcul de la fonction. C'est essentiel pour les calculs comme les moyennes mobiles ou les cumuls.

### Syntaxe des frames

```sql
{ROWS | RANGE | GROUPS} BETWEEN frame_start AND frame_end
```

**Options pour frame_start et frame_end :**
- `UNBOUNDED PRECEDING` : Début de la partition
- `n PRECEDING` : n lignes avant la ligne courante
- `CURRENT ROW` : Ligne courante
- `n FOLLOWING` : n lignes après la ligne courante
- `UNBOUNDED FOLLOWING` : Fin de la partition

### ROWS : Frame basé sur les lignes physiques

```sql
-- Moyenne mobile sur 7 jours (7 lignes)
SELECT
    date_vente,
    montant,
    AVG(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moyenne_mobile_7j
FROM ventes_quotidiennes;

-- Total cumulé depuis le début
SELECT
    date_vente,
    montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS total_cumule
FROM ventes_quotidiennes;

-- Fenêtre centrée (moyenne de la ligne précédente, actuelle et suivante)
SELECT
    date_vente,
    montant,
    AVG(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moyenne_centree
FROM ventes_quotidiennes;
```

### RANGE : Frame basé sur les valeurs

RANGE considère toutes les lignes ayant la **même valeur** dans ORDER BY comme un groupe.

```sql
-- Somme de toutes les ventes du même jour et des jours précédents
SELECT
    date_vente,
    heure_vente,
    montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS total_jusqu_au_jour
FROM ventes;

-- Avec RANGE, toutes les ventes du même jour sont incluses
-- même si heure_vente diffère
```

⚠️ **Différence ROWS vs RANGE :**
```sql
-- Données exemple
-- date_vente  | montant
-- 2024-01-01  | 100
-- 2024-01-01  | 200  -- Même date !
-- 2024-01-02  | 150

-- ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
-- Pour la ligne montant=200 : considère uniquement 100 et 200 (2 lignes physiques)

-- RANGE BETWEEN 1 PRECEDING AND CURRENT ROW
-- Pour la ligne montant=200 : considère 100, 200, et 150 car RANGE inclut toutes
-- les lignes avec date_vente <= 2024-01-01 + 1 jour
```

### GROUPS : Frame basé sur les groupes de valeurs (MariaDB 10.2+)

GROUPS traite chaque ensemble de lignes ayant la même valeur ORDER BY comme un groupe unique.

```sql
-- Somme des 3 derniers groupes de dates distinctes
SELECT
    date_vente,
    montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS total_3_jours_distincts
FROM ventes;
```

---

## Cas d'usage pratiques

### 1. Top N par catégorie

```sql
-- Trouver les 3 produits les plus chers par catégorie
WITH produits_classes AS (
    SELECT
        categorie,
        nom,
        prix,
        ROW_NUMBER() OVER (PARTITION BY categorie ORDER BY prix DESC) AS rang
    FROM produits
)
SELECT categorie, nom, prix
FROM produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

### 2. Comparaison période vs période

```sql
-- Comparer les ventes mensuelles avec le mois précédent
SELECT
    DATE_FORMAT(date_vente, '%Y-%m') AS mois,
    SUM(montant) AS ventes_mois,
    LAG(SUM(montant), 1) OVER (ORDER BY DATE_FORMAT(date_vente, '%Y-%m')) AS ventes_mois_prec,
    SUM(montant) - LAG(SUM(montant), 1) OVER (ORDER BY DATE_FORMAT(date_vente, '%Y-%m')) AS variation,
    ROUND(
        (SUM(montant) - LAG(SUM(montant), 1) OVER (ORDER BY DATE_FORMAT(date_vente, '%Y-%m'))) /
        LAG(SUM(montant), 1) OVER (ORDER BY DATE_FORMAT(date_vente, '%Y-%m')) * 100,
        2
    ) AS variation_pct
FROM ventes
GROUP BY DATE_FORMAT(date_vente, '%Y-%m')
ORDER BY mois;
```

### 3. Percentiles et quartiles

```sql
-- Distribution des salaires par département
SELECT
    departement_id,
    nom,
    salaire,
    NTILE(4) OVER (PARTITION BY departement_id ORDER BY salaire) AS quartile,
    -- Position relative dans le département (0-100%)
    PERCENT_RANK() OVER (PARTITION BY departement_id ORDER BY salaire) * 100 AS percentile,
    -- Comparaison avec la médiane du département
    salaire - PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salaire)
              OVER (PARTITION BY departement_id) AS ecart_mediane
FROM employes;
```

### 4. Détection des anomalies

```sql
-- Identifier les variations anormales de stock
WITH variations AS (
    SELECT
        produit_id,
        date_inventaire,
        quantite,
        LAG(quantite, 1) OVER (PARTITION BY produit_id ORDER BY date_inventaire) AS quantite_precedente,
        quantite - LAG(quantite, 1) OVER (PARTITION BY produit_id ORDER BY date_inventaire) AS variation,
        AVG(quantite) OVER (
            PARTITION BY produit_id
            ORDER BY date_inventaire
            ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
        ) AS moyenne_7j_precedents,
        STDDEV(quantite) OVER (
            PARTITION BY produit_id
            ORDER BY date_inventaire
            ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
        ) AS ecart_type_7j
    FROM inventaire
)
SELECT
    produit_id,
    date_inventaire,
    quantite,
    variation,
    moyenne_7j_precedents,
    -- Anomalie si variation > 2 écarts-types
    CASE
        WHEN ABS(variation) > 2 * ecart_type_7j THEN '⚠️ ANOMALIE'
        ELSE 'Normal'
    END AS statut
FROM variations
WHERE quantite_precedente IS NOT NULL
ORDER BY produit_id, date_inventaire;
```

### 5. Analyse de cohorte

```sql
-- Rétention client par cohorte d'inscription
SELECT
    DATE_FORMAT(date_inscription, '%Y-%m') AS cohorte,
    TIMESTAMPDIFF(MONTH, date_inscription, date_achat) AS mois_depuis_inscription,
    COUNT(DISTINCT client_id) AS nb_clients_actifs,
    -- Taille de la cohorte (premier mois)
    FIRST_VALUE(COUNT(DISTINCT client_id)) OVER (
        PARTITION BY DATE_FORMAT(date_inscription, '%Y-%m')
        ORDER BY TIMESTAMPDIFF(MONTH, date_inscription, date_achat)
    ) AS taille_cohorte,
    -- Taux de rétention
    ROUND(
        COUNT(DISTINCT client_id) * 100.0 /
        FIRST_VALUE(COUNT(DISTINCT client_id)) OVER (
            PARTITION BY DATE_FORMAT(date_inscription, '%Y-%m')
            ORDER BY TIMESTAMPDIFF(MONTH, date_inscription, date_achat)
        ),
        2
    ) AS taux_retention_pct
FROM achats
GROUP BY
    DATE_FORMAT(date_inscription, '%Y-%m'),
    TIMESTAMPDIFF(MONTH, date_inscription, date_achat)
ORDER BY cohorte, mois_depuis_inscription;
```

---

## Optimisation et performances

### Nommage de fenêtres (WINDOW clause)

Pour éviter de répéter la même définition de fenêtre, utilisez la clause WINDOW :

```sql
-- Sans WINDOW : répétition de la définition
SELECT
    nom,
    salaire,
    AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen,
    MAX(salaire) OVER (PARTITION BY departement_id) AS salaire_max,
    MIN(salaire) OVER (PARTITION BY departement_id) AS salaire_min
FROM employes;

-- Avec WINDOW : définition réutilisable
SELECT
    nom,
    salaire,
    AVG(salaire) OVER w AS salaire_moyen,
    MAX(salaire) OVER w AS salaire_max,
    MIN(salaire) OVER w AS salaire_min
FROM employes
WINDOW w AS (PARTITION BY departement_id);
```

💡 **Avantage :** Code plus lisible, maintenance facilitée, et potentiellement meilleures performances.

### Conseils de performance

1. **Indexation appropriée**
   - Créez des index sur les colonnes utilisées dans PARTITION BY et ORDER BY
   ```sql
   CREATE INDEX idx_dept_salaire ON employes(departement_id, salaire);
   ```

2. **Limiter le volume de données**
   - Filtrez avec WHERE avant d'appliquer les window functions
   ```sql
   -- BON : Filtre avant la fenêtre
   SELECT
       nom,
       salaire,
       AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen
   FROM employes
   WHERE date_embauche >= '2023-01-01'  -- Réduit le dataset
   ORDER BY departement_id;
   ```

3. **Éviter les frames trop larges**
   - ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING peut être coûteux
   - Privilégiez des fenêtres de taille raisonnable

4. **Utiliser EXPLAIN pour analyser**
   ```sql
   EXPLAIN
   SELECT
       nom,
       AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen
   FROM employes;
   ```

---

## Comparaison : Window Functions vs Alternatives

### Avant les Window Functions

```sql
-- Méthode 1 : Sous-requêtes corrélées (lent)
SELECT
    e1.nom,
    e1.salaire,
    (SELECT AVG(e2.salaire)
     FROM employes e2
     WHERE e2.departement_id = e1.departement_id) AS salaire_moyen,
    (SELECT COUNT(*)
     FROM employes e3
     WHERE e3.departement_id = e1.departement_id
       AND e3.salaire >= e1.salaire) AS rang
FROM employes e1;

-- Méthode 2 : Jointure avec agrégation (complexe)
SELECT
    e.nom,
    e.salaire,
    d.salaire_moyen,
    d.nb_employes
FROM employes e
JOIN (
    SELECT
        departement_id,
        AVG(salaire) AS salaire_moyen,
        COUNT(*) AS nb_employes
    FROM employes
    GROUP BY departement_id
) d ON e.departement_id = d.departement_id;
```

### Avec Window Functions

```sql
-- Beaucoup plus simple et performant
SELECT
    nom,
    salaire,
    AVG(salaire) OVER (PARTITION BY departement_id) AS salaire_moyen,
    COUNT(*) OVER (PARTITION BY departement_id) AS nb_employes,
    RANK() OVER (PARTITION BY departement_id ORDER BY salaire DESC) AS rang
FROM employes;
```

📊 **Gains typiques :**
- Performance : 2-10x plus rapide selon le contexte
- Lisibilité : Code 50-70% plus court
- Maintenabilité : Une seule requête au lieu de plusieurs imbriquées

---

## Limitations et pièges courants

### 1. Window Functions dans WHERE et GROUP BY

❌ **ERREUR** : Les window functions ne peuvent pas être utilisées dans WHERE ou GROUP BY
```sql
-- INVALIDE
SELECT nom, salaire
FROM employes
WHERE salaire > AVG(salaire) OVER ();  -- ❌ ERREUR

-- INVALIDE
SELECT departement_id, AVG(salaire)
FROM employes
GROUP BY AVG(salaire) OVER (PARTITION BY departement_id);  -- ❌ ERREUR
```

✅ **SOLUTION** : Utiliser une CTE ou une sous-requête
```sql
-- VALIDE
WITH salaires_analyses AS (
    SELECT
        nom,
        salaire,
        AVG(salaire) OVER () AS salaire_moyen
    FROM employes
)
SELECT nom, salaire
FROM salaires_analyses
WHERE salaire > salaire_moyen;
```

### 2. Ordre par défaut des frames

⚠️ **Attention** : Par défaut, le frame est `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

```sql
-- Ces deux requêtes sont équivalentes
SELECT SUM(montant) OVER (ORDER BY date_vente)
FROM ventes;

SELECT SUM(montant) OVER (
    ORDER BY date_vente
    RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
FROM ventes;
```

Pour un calcul sur la ligne courante uniquement :
```sql
-- Spécifier explicitement ROWS CURRENT ROW
SELECT
    montant,
    SUM(montant) OVER (ORDER BY date_vente ROWS CURRENT ROW) AS montant
FROM ventes;
```

### 3. LAST_VALUE et le frame par défaut

```sql
-- ❌ PIÈGE : LAST_VALUE avec frame par défaut
SELECT
    nom,
    salaire,
    LAST_VALUE(salaire) OVER (
        PARTITION BY departement_id
        ORDER BY salaire
    ) AS salaire_max  -- ❌ Renvoie la valeur actuelle, pas la dernière !
FROM employes;

-- ✅ SOLUTION : Spécifier le frame correctement
SELECT
    nom,
    salaire,
    LAST_VALUE(salaire) OVER (
        PARTITION BY departement_id
        ORDER BY salaire
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS salaire_max
FROM employes;

-- OU utiliser FIRST_VALUE avec ORDER BY inversé
SELECT
    nom,
    salaire,
    FIRST_VALUE(salaire) OVER (
        PARTITION BY departement_id
        ORDER BY salaire DESC
    ) AS salaire_max
FROM employes;
```

### 4. NULL dans ORDER BY

```sql
-- Les NULL sont triés en premier par défaut (NULLS FIRST)
SELECT
    nom,
    date_depart,
    ROW_NUMBER() OVER (ORDER BY date_depart) AS rang
FROM employes;
-- Les employés sans date_depart (NULL) auront rang = 1, 2, 3...

-- Pour placer les NULL à la fin
SELECT
    nom,
    date_depart,
    ROW_NUMBER() OVER (ORDER BY date_depart NULLS LAST) AS rang
FROM employes;
```

---

## ✅ Points clés à retenir

- **Window Functions** permettent des calculs analytiques **sans perdre le détail des lignes**, contrairement à GROUP BY
- La clause **OVER()** est obligatoire et définit la fenêtre de calcul
- **PARTITION BY** divise les données en groupes indépendants (comme GROUP BY mais sans agrégation)
- **ORDER BY** définit l'ordre dans la partition (essentiel pour rangs et valeurs)
- Les **frames** (ROWS/RANGE/GROUPS) définissent le sous-ensemble de lignes pour le calcul
- Trois types de fonctions : **agrégation** (SUM, AVG...), **rang** (ROW_NUMBER, RANK...), **valeur** (LAG, LEAD...)
- Utiliser la clause **WINDOW** pour nommer et réutiliser des définitions de fenêtre
- Optimiser avec des **index** sur les colonnes de PARTITION BY et ORDER BY
- Les window functions **ne peuvent pas** être utilisées dans WHERE ou GROUP BY → utiliser des CTE
- Attention au **frame par défaut** qui peut donner des résultats inattendus avec LAST_VALUE

---

## 🔗 Ressources et références

- [📖 Documentation officielle MariaDB - Window Functions](https://mariadb.com/kb/en/window-functions/)
- [📖 Window Functions Overview](https://mariadb.com/kb/en/window-functions-overview/)
- [📖 Window Frames](https://mariadb.com/kb/en/window-frames/)
- [📖 MariaDB 10.2 Release Notes - Window Functions](https://mariadb.com/kb/en/mariadb-1020-release-notes/)

**Lectures complémentaires :**
- [Modern SQL - Window Functions](https://modern-sql.com/feature/over)
- [Use The Index, Luke - Window Functions](https://use-the-index-luke.com/sql/partial-results/window-functions)

---

## ➡️ Section suivante

**[4.2.1 Fonctions de rang (ROW_NUMBER, RANK, DENSE_RANK, NTILE)](/04-concepts-avances-sql/02.1-fonctions-rang.md)** : Approfondir les fonctions de classement et leurs cas d'usage spécifiques (Top N, pagination, quartiles, détection de doublons).

---

**Note** : Les Window Functions sont un outil puissant mais peuvent être complexes au début. N'hésitez pas à expérimenter avec des jeux de données simples pour bien comprendre le comportement des différentes clauses (PARTITION BY, ORDER BY, frames). La pratique régulière vous permettra de maîtriser rapidement ces concepts avancés ! 🚀

⏭️ [Fonctions de rang (ROW_NUMBER, RANK, DENSE_RANK, NTILE)](/04-concepts-avances-sql/02.1-fonctions-rang.md)

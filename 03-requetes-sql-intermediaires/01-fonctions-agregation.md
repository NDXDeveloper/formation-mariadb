🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Fonctions d'agrégation (COUNT, SUM, AVG, MIN, MAX)

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures
> **Prérequis** : Chapitre 2 (Bases du SQL), requêtes SELECT simples, compréhension des types de données

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Utiliser les cinq fonctions d'agrégation principales de SQL
- Comprendre comment MariaDB traite les valeurs NULL dans les agrégations
- Calculer des statistiques simples et complexes sur vos données
- Combiner plusieurs fonctions d'agrégation dans une même requête
- Identifier les cas d'usage appropriés pour chaque fonction
- Appliquer les bonnes pratiques pour des requêtes performantes et fiables

---

## Introduction

Les **fonctions d'agrégation** sont au cœur de l'analyse de données en SQL. Elles permettent de transformer un ensemble de lignes en une valeur unique synthétique : un total, une moyenne, un comptage, etc.

### Pourquoi les fonctions d'agrégation sont essentielles ?

En environnement de production, vous aurez rarement besoin de toutes les lignes d'une table. Les questions métier portent sur des **synthèses** :
- *"Quel est le chiffre d'affaires total du mois ?"* → **SUM**
- *"Combien de clients se sont inscrits cette semaine ?"* → **COUNT**
- *"Quel est le panier moyen des commandes ?"* → **AVG**
- *"Quel est le produit le moins cher ? Le plus cher ?"* → **MIN / MAX**

Ces fonctions transforment des milliers, voire des millions de lignes en quelques chiffres exploitables pour le pilotage métier.

### Les cinq fonctions fondamentales

MariaDB propose cinq fonctions d'agrégation de base :

| Fonction | Objectif | Exemple d'usage |
|----------|----------|-----------------|
| **COUNT()** | Compter le nombre de lignes ou de valeurs | Nombre de clients, de commandes |
| **SUM()** | Additionner des valeurs numériques | Chiffre d'affaires, total des quantités |
| **AVG()** | Calculer une moyenne | Panier moyen, note moyenne |
| **MIN()** | Trouver la valeur minimale | Prix le plus bas, date la plus ancienne |
| **MAX()** | Trouver la valeur maximale | Prix le plus élevé, date la plus récente |

💡 **Principe fondamental** : Les fonctions d'agrégation **réduisent plusieurs lignes à une seule valeur**. Sans clause GROUP BY (que nous verrons en section 3.2), elles s'appliquent à l'ensemble du résultat.

---

## COUNT() : Compter les lignes et les valeurs

La fonction **COUNT()** est probablement la plus utilisée. Elle compte le nombre de lignes ou de valeurs non NULL.

### Syntaxe et variantes

```sql
COUNT(*)              -- Compte TOUTES les lignes (même avec NULL)
COUNT(colonne)        -- Compte les valeurs NON NULL dans la colonne
COUNT(DISTINCT col)   -- Compte les valeurs UNIQUES non NULL
```

### Exemple 1 : Comptage simple

**Question métier** : *Combien de clients sont enregistrés dans la base ?*

```sql
-- Compter le nombre total de clients
SELECT COUNT(*) AS nombre_clients
FROM clients;
```

**Résultat attendu** :
```
+-----------------+
| nombre_clients  |
+-----------------+
|            1247 |
+-----------------+
```

**Explication** :
- `COUNT(*)` compte **toutes les lignes** de la table `clients`
- L'alias `AS nombre_clients` rend le résultat plus lisible
- Cette syntaxe est la plus rapide car elle ne vérifie pas les valeurs NULL

### Exemple 2 : COUNT avec colonne spécifique

**Question métier** : *Combien de clients ont renseigné leur email ?*

```sql
-- Compter uniquement les emails non NULL
SELECT
    COUNT(*) AS total_clients,
    COUNT(email) AS clients_avec_email
FROM clients;
```

**Résultat attendu** :
```
+----------------+----------------------+
| total_clients  | clients_avec_email   |
+----------------+----------------------+
|           1247 |                 1198 |
+----------------+----------------------+
```

**Explication** :
- `COUNT(*)` = 1247 lignes au total
- `COUNT(email)` = 1198 lignes avec un email non NULL
- **Différence** : 49 clients n'ont pas d'email renseigné
- `COUNT(colonne)` **ignore les valeurs NULL**

💡 **Conseil** : Utilisez `COUNT(*)` quand vous voulez compter toutes les lignes, et `COUNT(colonne)` quand vous voulez vérifier la présence de données dans une colonne spécifique.

### Exemple 3 : COUNT DISTINCT

**Question métier** : *Combien de villes différentes sont représentées dans notre base clients ?*

```sql
-- Compter les valeurs uniques
SELECT
    COUNT(ville) AS total_villes_renseignees,
    COUNT(DISTINCT ville) AS nombre_villes_uniques
FROM clients;
```

**Résultat attendu** :
```
+---------------------------+------------------------+
| total_villes_renseignees  | nombre_villes_uniques  |
+---------------------------+------------------------+
|                      1180 |                    142 |
+---------------------------+------------------------+
```

**Explication** :
- 1180 clients ont une ville renseignée (67 clients avec ville = NULL)
- Ces 1180 clients sont répartis dans 142 villes différentes
- `COUNT(DISTINCT ...)` élimine les doublons avant de compter

**Cas d'usage réel** : Calculer le taux de couverture géographique, identifier le nombre de segments distincts.

⚠️ **Attention** : `COUNT(DISTINCT ...)` peut être coûteux en performance sur de très grandes tables car MariaDB doit trier ou utiliser une table temporaire pour éliminer les doublons.

---

## SUM() : Additionner des valeurs numériques

La fonction **SUM()** calcule la somme de toutes les valeurs d'une colonne numérique.

### Syntaxe

```sql
SUM(colonne_numerique)
```

⚠️ **Important** : SUM() ne fonctionne qu'avec des colonnes **numériques** (INT, DECIMAL, FLOAT, etc.). Sur une colonne texte, elle retournera une erreur ou un résultat incohérent.

### Exemple 4 : Somme simple

**Question métier** : *Quel est le chiffre d'affaires total généré par toutes les commandes ?*

```sql
-- Calculer le chiffre d'affaires total
SELECT
    SUM(montant_total) AS chiffre_affaires_total
FROM commandes;
```

**Résultat attendu** :
```
+--------------------------+
| chiffre_affaires_total   |
+--------------------------+
|              1847293.45  |
+--------------------------+
```

**Explication** :
- SUM() additionne toutes les valeurs de la colonne `montant_total`
- Le résultat est un DECIMAL reflétant la somme exacte
- Les valeurs NULL sont **ignorées** (comme pour COUNT(colonne))

### Exemple 5 : SUM avec condition

**Question métier** : *Quel est le montant total des commandes livrées ?*

```sql
-- Somme filtrée par statut
SELECT
    SUM(montant_total) AS ca_commandes_livrees
FROM commandes
WHERE statut = 'livrée';
```

**Résultat attendu** :
```
+------------------------+
| ca_commandes_livrees   |
+------------------------+
|            1652847.20  |
+------------------------+
```

**Explication** :
- La clause `WHERE` filtre **avant** l'agrégation
- Seules les commandes avec `statut = 'livrée'` sont incluses dans la somme
- Cette approche est essentielle pour calculer des KPI par statut, période, catégorie, etc.

### Exemple 6 : Plusieurs SUM() dans une même requête

**Question métier** : *Comparer le CA des commandes livrées vs en attente*

```sql
-- Plusieurs agrégations conditionnelles
SELECT
    SUM(montant_total) AS ca_total,
    SUM(CASE WHEN statut = 'livrée' THEN montant_total ELSE 0 END) AS ca_livrees,
    SUM(CASE WHEN statut IN ('en_attente', 'confirmée') THEN montant_total ELSE 0 END) AS ca_en_cours
FROM commandes;
```

**Résultat attendu** :
```
+------------+-------------+-------------+
| ca_total   | ca_livrees  | ca_en_cours |
+------------+-------------+-------------+
| 1847293.45 | 1652847.20  |   152468.80 |
+------------+-------------+-------------+
```

**Explication** :
- Utilisation de **CASE WHEN** pour créer des sommes conditionnelles
- Chaque SUM() applique sa propre logique de filtrage
- Alternative plus lisible que plusieurs requêtes séparées
- Les expressions `CASE` seront détaillées en section 3.8

💡 **Cas d'usage réel** : Dashboards avec plusieurs métriques calculées en une seule requête (réduction du nombre d'appels à la base).

---

## AVG() : Calculer des moyennes

La fonction **AVG()** calcule la moyenne arithmétique des valeurs d'une colonne numérique.

### Syntaxe

```sql
AVG(colonne_numerique)
```

La formule appliquée est : **AVG = SUM(valeurs) / COUNT(valeurs non NULL)**

### Exemple 7 : Moyenne simple

**Question métier** : *Quel est le montant moyen d'une commande ?*

```sql
-- Calculer le panier moyen
SELECT
    AVG(montant_total) AS panier_moyen
FROM commandes;
```

**Résultat attendu** :
```
+---------------+
| panier_moyen  |
+---------------+
|       148.23  |
+---------------+
```

**Explication** :
- MariaDB calcule la somme de tous les montants, puis divise par le nombre de commandes (avec montant non NULL)
- Le résultat est un DECIMAL avec précision
- AVG() ignore les valeurs NULL dans le calcul

### Exemple 8 : AVG et gestion des NULL

**Comprendre l'impact des NULL** :

```sql
-- Données de test
CREATE TEMPORARY TABLE test_avg (valeur DECIMAL(10,2));
INSERT INTO test_avg VALUES (100), (200), (300), (NULL), (NULL);

-- Calcul de la moyenne
SELECT
    COUNT(*) AS total_lignes,
    COUNT(valeur) AS valeurs_non_null,
    SUM(valeur) AS somme,
    AVG(valeur) AS moyenne
FROM test_avg;
```

**Résultat** :
```
+--------------+------------------+--------+----------+
| total_lignes | valeurs_non_null | somme  | moyenne  |
+--------------+------------------+--------+----------+
|            5 |                3 | 600.00 |   200.00 |
+--------------+------------------+--------+----------+
```

**Explication critique** :
- Il y a **5 lignes** au total
- Seulement **3 valeurs non NULL**
- Somme = 100 + 200 + 300 = 600
- Moyenne = 600 / 3 = **200** (et non 600 / 5 = 120)

⚠️ **Piège courant** : Si vous voulez inclure les NULL comme des zéros dans le calcul, vous devez utiliser `COALESCE` ou `IFNULL` :

```sql
-- Traiter les NULL comme des zéros
SELECT
    AVG(COALESCE(valeur, 0)) AS moyenne_avec_zeros
FROM test_avg;
-- Résultat : 120.00 (600 / 5)
```

### Exemple 9 : Moyenne avec arrondi

**Question métier** : *Quel est le prix unitaire moyen des produits, arrondi à 2 décimales ?*

```sql
-- Moyenne arrondie pour affichage
SELECT
    AVG(prix_unitaire) AS prix_moyen_exact,
    ROUND(AVG(prix_unitaire), 2) AS prix_moyen_arrondi
FROM produits;
```

**Résultat attendu** :
```
+-------------------+---------------------+
| prix_moyen_exact  | prix_moyen_arrondi  |
+-------------------+---------------------+
|      45.8372641   |              45.84  |
+-------------------+---------------------+
```

💡 **Conseil** : Pour l'affichage utilisateur, arrondissez toujours les moyennes avec `ROUND()` pour éviter des décimales excessives.

---

## MIN() et MAX() : Trouver les valeurs extrêmes

Les fonctions **MIN()** et **MAX()** retournent respectivement la valeur minimale et maximale d'une colonne.

### Syntaxe

```sql
MIN(colonne)
MAX(colonne)
```

**Particularités** :
- Fonctionnent sur **tous les types** : numériques, dates, chaînes de caractères
- Pour les chaînes : ordre alphabétique (A < Z)
- Pour les dates : ordre chronologique
- Ignorent les valeurs NULL

### Exemple 10 : MIN/MAX sur valeurs numériques

**Question métier** : *Quel est le prix du produit le moins cher et le plus cher ?*

```sql
-- Trouver les prix extrêmes
SELECT
    MIN(prix_unitaire) AS prix_minimum,
    MAX(prix_unitaire) AS prix_maximum,
    MAX(prix_unitaire) - MIN(prix_unitaire) AS ecart_prix
FROM produits;
```

**Résultat attendu** :
```
+---------------+---------------+------------+
| prix_minimum  | prix_maximum  | ecart_prix |
+---------------+---------------+------------+
|          2.99 |       1299.99 |    1297.00 |
+---------------+---------------+------------+
```

**Explication** :
- MIN() trouve la valeur la plus petite : 2.99 €
- MAX() trouve la valeur la plus grande : 1299.99 €
- On peut calculer l'écart directement dans le SELECT

**Cas d'usage** : Analyse de la gamme de produits, détection de valeurs aberrantes.

### Exemple 11 : MIN/MAX sur dates

**Question métier** : *Quelle est la date de la première et dernière inscription client ?*

```sql
-- Trouver les dates extrêmes
SELECT
    MIN(date_inscription) AS premiere_inscription,
    MAX(date_inscription) AS derniere_inscription,
    DATEDIFF(MAX(date_inscription), MIN(date_inscription)) AS jours_activite
FROM clients;
```

**Résultat attendu** :
```
+----------------------+----------------------+-----------------+
| premiere_inscription | derniere_inscription | jours_activite  |
+----------------------+----------------------+-----------------+
| 2020-01-15           | 2025-12-10           |            2156 |
+----------------------+----------------------+-----------------+
```

**Explication** :
- MIN(date) retourne la date la plus ancienne
- MAX(date) retourne la date la plus récente
- DATEDIFF() calcule le nombre de jours entre les deux
- Utile pour calculer l'ancienneté, la période d'activité

### Exemple 12 : MIN/MAX sur chaînes de caractères

**Question métier** : *Quelle est la première et dernière ville par ordre alphabétique ?*

```sql
-- Ordre alphabétique
SELECT
    MIN(ville) AS premiere_ville_alpha,
    MAX(ville) AS derniere_ville_alpha
FROM clients
WHERE ville IS NOT NULL;
```

**Résultat attendu** :
```
+-----------------------+----------------------+
| premiere_ville_alpha  | derniere_ville_alpha |
+-----------------------+----------------------+
| Aix-en-Provence       | Villeurbanne         |
+-----------------------+----------------------+
```

**Explication** :
- Sur les chaînes, MIN/MAX utilisent l'**ordre alphabétique** (collation)
- 'A' < 'B' < ... < 'Z'
- Le WHERE IS NOT NULL évite que NULL soit considéré

💡 **Cas d'usage** : Rarement utilisé sur du texte, mais peut servir pour trouver les premiers/derniers éléments d'une liste triée.

---

## Combiner plusieurs fonctions d'agrégation

Les fonctions d'agrégation peuvent être combinées dans une même requête SELECT pour obtenir une vue d'ensemble complète.

### Exemple 13 : Vue statistique complète

**Question métier** : *Obtenir un résumé statistique des commandes*

```sql
-- Statistiques complètes sur les commandes
SELECT
    COUNT(*) AS nombre_commandes,
    COUNT(DISTINCT id_client) AS nombre_clients_actifs,
    SUM(montant_total) AS ca_total,
    AVG(montant_total) AS panier_moyen,
    MIN(montant_total) AS commande_min,
    MAX(montant_total) AS commande_max,
    MIN(date_commande) AS premiere_commande,
    MAX(date_commande) AS derniere_commande
FROM commandes
WHERE statut IN ('confirmée', 'expédiée', 'livrée');
```

**Résultat attendu** :
```
+------------------+------------------------+-----------+--------------+--------------+--------------+-------------------+-------------------+
| nombre_commandes | nombre_clients_actifs  | ca_total  | panier_moyen | commande_min | commande_max | premiere_commande | derniere_commande |
+------------------+------------------------+-----------+--------------+--------------+--------------+-------------------+-------------------+
|            12463 |                   3847 | 1847293.45|       148.23 |        12.50 |      2847.90 | 2024-01-01 08:15  | 2025-12-10 23:47  |
+------------------+------------------------+-----------+--------------+--------------+--------------+-------------------+-------------------+
```

**Explication** :
- **Une seule requête** fournit 8 métriques différentes
- Chaque fonction d'agrégation opère sur l'ensemble des lignes filtrées
- Cette approche est **bien plus efficace** que 8 requêtes séparées
- Idéal pour construire des dashboards ou rapports récapitulatifs

### Exemple 14 : Comparaison avec calculs dérivés

**Question métier** : *Analyser la dispersion des montants de commandes*

```sql
-- Statistiques avec calculs dérivés
SELECT
    COUNT(*) AS nb_commandes,
    MIN(montant_total) AS montant_min,
    AVG(montant_total) AS montant_moyen,
    MAX(montant_total) AS montant_max,
    MAX(montant_total) - MIN(montant_total) AS etendue,
    ROUND(AVG(montant_total), 2) AS panier_moyen_arrondi
FROM commandes;
```

**Résultat attendu** :
```
+---------------+-------------+---------------+-------------+---------+------------------------+
| nb_commandes  | montant_min | montant_moyen | montant_max | etendue | panier_moyen_arrondi   |
+---------------+-------------+---------------+-------------+---------+------------------------+
|         12463 |       12.50 |       148.234 |     2847.90 | 2835.40 |                 148.23 |
+---------------+-------------+---------------+-------------+---------+------------------------+
```

**Explication** :
- L'**étendue** (range) = MAX - MIN mesure la dispersion
- Vous pouvez faire des calculs **sur les résultats d'agrégation**
- Combine agrégations et expressions arithmétiques

💡 **Cas d'usage réel** : Rapports statistiques, détection d'anomalies (commandes anormalement élevées ou basses), calcul de KPI complexes.

---

## Comportement avec les valeurs NULL

La gestion des **NULL** est un aspect crucial des fonctions d'agrégation. Un comportement mal compris peut conduire à des résultats incorrects.

### Règles fondamentales

| Fonction | Comportement avec NULL |
|----------|------------------------|
| **COUNT(*)** | Compte **toutes** les lignes, même celles avec NULL |
| **COUNT(colonne)** | Ignore les NULL, compte seulement les non NULL |
| **SUM(colonne)** | Ignore les NULL, additionne les non NULL |
| **AVG(colonne)** | Ignore les NULL, moyenne sur les non NULL uniquement |
| **MIN(colonne)** | Ignore les NULL, trouve le min des non NULL |
| **MAX(colonne)** | Ignore les NULL, trouve le max des non NULL |

### Exemple 15 : Impact des NULL sur les agrégations

```sql
-- Créer une table de test avec NULL
CREATE TEMPORARY TABLE ventes_test (
    id INT PRIMARY KEY AUTO_INCREMENT,
    produit VARCHAR(50),
    quantite INT,
    montant DECIMAL(10,2)
);

INSERT INTO ventes_test (produit, quantite, montant) VALUES
('Produit A', 10, 100.00),
('Produit B', 20, 200.00),
('Produit C', NULL, 150.00),  -- quantité NULL
('Produit D', 15, NULL),      -- montant NULL
('Produit E', NULL, NULL);    -- tout NULL

-- Analyser l'impact des NULL
SELECT
    COUNT(*) AS total_lignes,
    COUNT(quantite) AS lignes_avec_quantite,
    COUNT(montant) AS lignes_avec_montant,
    SUM(quantite) AS somme_quantites,
    AVG(quantite) AS moyenne_quantite,
    SUM(montant) AS somme_montants,
    AVG(montant) AS moyenne_montant
FROM ventes_test;
```

**Résultat** :
```
+--------------+----------------------+--------------------+------------------+------------------+----------------+----------------+
| total_lignes | lignes_avec_quantite | lignes_avec_montant| somme_quantites  | moyenne_quantite | somme_montants | moyenne_montant|
+--------------+----------------------+--------------------+------------------+------------------+----------------+----------------+
|            5 |                    3 |                  3 |               45 |            15.00 |         450.00 |         150.00 |
+--------------+----------------------+--------------------+------------------+------------------+----------------+----------------+
```

**Analyse** :
- 5 lignes au total (`COUNT(*)`)
- Seulement 3 ont une quantité non NULL → `SUM(quantite)` = 10+20+15 = 45
- Seulement 3 ont un montant non NULL → `AVG(montant)` = (100+200+150)/3 = 150
- Les NULL sont **systématiquement ignorés** dans les calculs

### Exemple 16 : Différence COUNT(*) vs COUNT(colonne)

**Question métier** : *Identifier les commandes sans montant renseigné*

```sql
-- Comparer le nombre total et le nombre avec montant
SELECT
    COUNT(*) AS total_commandes,
    COUNT(montant_total) AS commandes_avec_montant,
    COUNT(*) - COUNT(montant_total) AS commandes_sans_montant
FROM commandes;
```

**Résultat attendu** :
```
+------------------+------------------------+-------------------------+
| total_commandes  | commandes_avec_montant | commandes_sans_montant  |
+------------------+------------------------+-------------------------+
|            12463 |                  12458 |                       5 |
+------------------+------------------------+-------------------------+
```

**Explication** :
- 5 commandes ont un `montant_total` = NULL
- `COUNT(*)` les compte, `COUNT(montant_total)` les ignore
- Cette technique permet de **détecter les données manquantes**

### Comment traiter les NULL comme des zéros ?

Si vous voulez que NULL soit traité comme 0 dans les calculs :

```sql
-- Remplacer NULL par 0 avec COALESCE
SELECT
    SUM(COALESCE(montant_total, 0)) AS ca_avec_null_a_zero,
    AVG(COALESCE(montant_total, 0)) AS panier_moyen_null_a_zero
FROM commandes;
```

Ou avec `IFNULL` (syntaxe plus courte) :

```sql
SELECT
    SUM(IFNULL(montant_total, 0)) AS ca_avec_null_a_zero
FROM commandes;
```

⚠️ **Attention** : Cela modifie la sémantique de vos calculs. Un NULL n'est pas un zéro ! Assurez-vous que c'est le comportement souhaité.

---

## Cas d'usage réels en production

### Cas d'usage 1 : Dashboard de ventes en temps réel

**Objectif** : Afficher les KPI principaux pour le management

```sql
-- KPI dashboard
SELECT
    COUNT(DISTINCT id_commande) AS nb_commandes_aujourd_hui,
    COUNT(DISTINCT id_client) AS nb_clients_actifs,
    SUM(montant_total) AS ca_du_jour,
    AVG(montant_total) AS panier_moyen,
    MAX(montant_total) AS plus_grosse_commande
FROM commandes
WHERE DATE(date_commande) = CURDATE()
  AND statut != 'annulée';
```

**Application** : Requête exécutée toutes les 5 minutes pour mettre à jour un tableau de bord.

### Cas d'usage 2 : Analyse de stock

**Objectif** : Identifier les produits en rupture ou surstockés

```sql
-- Analyse de stock
SELECT
    COUNT(*) AS nb_produits_total,
    COUNT(CASE WHEN stock = 0 THEN 1 END) AS nb_ruptures,
    COUNT(CASE WHEN stock < 10 THEN 1 END) AS nb_stock_faible,
    AVG(stock) AS stock_moyen,
    MIN(stock) AS stock_minimum,
    MAX(stock) AS stock_maximum,
    SUM(stock * prix_unitaire) AS valeur_stock_total
FROM produits;
```

**Application** : Rapport quotidien pour le service logistique et achats.

### Cas d'usage 3 : Calcul de taux de conversion

**Objectif** : Mesurer l'efficacité du tunnel de vente

```sql
-- Taux de conversion par statut
SELECT
    COUNT(*) AS total_commandes,
    SUM(CASE WHEN statut = 'livrée' THEN 1 ELSE 0 END) AS commandes_livrees,
    ROUND(
        100.0 * SUM(CASE WHEN statut = 'livrée' THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS taux_livraison_pct,
    SUM(CASE WHEN statut = 'annulée' THEN 1 ELSE 0 END) AS commandes_annulees,
    ROUND(
        100.0 * SUM(CASE WHEN statut = 'annulée' THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS taux_annulation_pct
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY);
```

**Résultat attendu** :
```
+------------------+--------------------+---------------------+---------------------+----------------------+
| total_commandes  | commandes_livrees  | taux_livraison_pct  | commandes_annulees  | taux_annulation_pct  |
+------------------+--------------------+---------------------+---------------------+----------------------+
|             1847 |               1723 |               93.29 |                  48 |                 2.60 |
+------------------+--------------------+---------------------+---------------------+----------------------+
```

**Application** : Monitoring qualité de service, identification de problèmes dans la chaîne logistique.

### Cas d'usage 4 : Segmentation RFM (Récence, Fréquence, Montant)

**Objectif** : Calculer les métriques RFM pour chaque client (simplifié ici sans GROUP BY)

```sql
-- Agrégations globales pour benchmarking
SELECT
    AVG(DATEDIFF(CURDATE(), date_derniere_commande)) AS recence_moyenne_jours,
    AVG(nombre_commandes) AS frequence_moyenne,
    AVG(ca_total_client) AS ca_moyen_par_client
FROM (
    -- Sous-requête calculant les métriques par client
    SELECT
        id_client,
        MAX(date_commande) AS date_derniere_commande,
        COUNT(*) AS nombre_commandes,
        SUM(montant_total) AS ca_total_client
    FROM commandes
    WHERE statut IN ('confirmée', 'expédiée', 'livrée')
    GROUP BY id_client
) AS metriques_clients;
```

**Application** : Base pour la segmentation marketing et les campagnes ciblées.

💡 **Note** : Cet exemple utilise une sous-requête avec GROUP BY (section 3.2), pour illustrer l'utilisation d'agrégations sur des résultats déjà agrégés.

---

## Optimisation et bonnes pratiques

### Performance des fonctions d'agrégation

#### ✅ Bonne pratique 1 : Filtrer avant d'agréger

```sql
-- ✅ BON : Filtre avec WHERE avant agrégation
SELECT COUNT(*), AVG(montant_total)
FROM commandes
WHERE date_commande >= '2025-01-01';

-- ❌ MOINS BON : Filtre après agrégation (nécessite HAVING et GROUP BY)
-- Plus complexe et potentiellement moins performant
```

**Pourquoi ?** WHERE filtre les lignes **avant** que les agrégations ne soient calculées, réduisant le volume de données à traiter.

#### ✅ Bonne pratique 2 : Utiliser des index

Les fonctions d'agrégation bénéficient énormément d'index appropriés :

```sql
-- Index sur les colonnes fréquemment agrégées ou filtrées
CREATE INDEX idx_commandes_date ON commandes(date_commande);
CREATE INDEX idx_commandes_statut ON commandes(statut);
```

**Impact** : Les agrégations filtrées par date ou statut seront **beaucoup plus rapides**.

#### ✅ Bonne pratique 3 : COUNT(*) vs COUNT(colonne)

```sql
-- ✅ Préférer COUNT(*) pour compter les lignes
SELECT COUNT(*) FROM commandes;

-- ⚠️ COUNT(id_commande) fonctionne mais est légèrement plus lent
SELECT COUNT(id_commande) FROM commandes;
```

**Pourquoi ?** `COUNT(*)` est optimisé par MariaDB et ne vérifie pas les valeurs NULL.

#### ✅ Bonne pratique 4 : Éviter les agrégations sur trop de lignes

Pour les très grandes tables (millions de lignes), considérez :
- **Pré-agrégation** : Stocker des résultats agrégés dans des tables de résumé
- **Partitionnement** : Diviser les grandes tables par date
- **Limiter la période** : Toujours filtrer sur une plage de dates raisonnable

```sql
-- ❌ LENT : Agrégation sur des millions de lignes
SELECT SUM(montant_total) FROM commandes;

-- ✅ RAPIDE : Agrégation sur une période limitée
SELECT SUM(montant_total)
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY);
```

### Lisibilité et maintenabilité

#### ✅ Utiliser des alias explicites

```sql
-- ✅ LISIBLE
SELECT
    COUNT(*) AS nombre_total_commandes,
    SUM(montant_total) AS chiffre_affaires_total,
    AVG(montant_total) AS panier_moyen
FROM commandes;

-- ❌ PEU LISIBLE
SELECT
    COUNT(*),
    SUM(montant_total),
    AVG(montant_total)
FROM commandes;
```

#### ✅ Documenter les requêtes complexes

```sql
-- Calcul du CA par statut pour le rapport mensuel
-- Exclut les commandes annulées
-- Dernière mise à jour : 2025-12-10
SELECT
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
WHERE statut != 'annulée'
  AND date_commande >= '2025-12-01';
```

### Gestion des NULL : Checklist

- ✅ **Vérifiez toujours** si vos colonnes peuvent contenir des NULL
- ✅ **Documentez** si NULL doit être traité comme 0 ou ignoré
- ✅ **Utilisez COUNT(*)** pour compter toutes les lignes
- ✅ **Utilisez COUNT(colonne)** pour détecter les valeurs manquantes
- ✅ **Testez** vos requêtes avec des données contenant des NULL

---

## Points clés à retenir

1. **COUNT(*)** compte toutes les lignes, **COUNT(colonne)** compte les valeurs non NULL – distinction fondamentale pour détecter les données manquantes

2. **SUM() et AVG() ignorent les NULL** – les NULL ne sont ni additionnés ni inclus dans le dénominateur de la moyenne

3. **MIN() et MAX() fonctionnent sur tous types** – numériques (valeur), dates (chronologie), texte (ordre alphabétique)

4. **Combiner plusieurs agrégations dans un seul SELECT** est efficace – une requête au lieu de plusieurs, réduction de la charge serveur

5. **WHERE filtre avant agrégation, HAVING après** (section 3.2) – toujours filtrer avec WHERE quand possible pour la performance

6. **COUNT(DISTINCT colonne)** compte les valeurs uniques – utile mais potentiellement coûteux sur de grandes tables

7. **NULL ≠ 0** – ne les confondez pas, utilisez COALESCE/IFNULL seulement si sémantiquement correct

8. **Utilisez ROUND() pour l'affichage** – évite des décimales excessives sur AVG()

9. **Les index améliorent drastiquement les performances** – particulièrement sur les colonnes dans WHERE et les agrégations

10. **Toujours valider avec EXPLAIN** – vérifiez que votre requête utilise les index appropriés

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Aggregate Functions](https://mariadb.com/kb/en/aggregate-functions/) – Documentation complète des fonctions d'agrégation
- [📖 COUNT Function](https://mariadb.com/kb/en/count/) – Détails sur COUNT et ses variantes
- [📖 NULL Values](https://mariadb.com/kb/en/null-values/) – Comportement de NULL en SQL

### Articles approfondis
- [Understanding SQL Aggregate Functions](https://modern-sql.com/feature/aggregate-functions) – Guide approfondi sur les agrégations
- [NULL handling in SQL](https://use-the-index-luke.com/sql/where-clause/null) – Gestion des NULL et impact sur les performances

### Outils pratiques
- [SQLFiddle](http://sqlfiddle.com/) – Tester vos requêtes d'agrégation en ligne
- [EXPLAIN Analyzer](https://mariadb.org/explain-analyzer/) – Analyser les plans d'exécution

---

## ➡️ Section suivante

**[3.2 Regroupement de données (GROUP BY, HAVING)](./02-regroupement-donnees.md)**

Maintenant que vous maîtrisez les fonctions d'agrégation, la section suivante vous apprendra à les **segmenter par catégories** avec GROUP BY. Vous découvrirez comment :
- Calculer des agrégations **par groupe** (CA par catégorie, commandes par client, etc.)
- Utiliser **HAVING** pour filtrer les résultats agrégés
- Comprendre la différence cruciale entre WHERE et HAVING
- Maîtriser les règles de composition des GROUP BY

GROUP BY est l'outil qui transforme les fonctions d'agrégation en véritables analyses métier segmentées ! 📊

---


⏭️ [Regroupement de données (GROUP BY, HAVING)](/03-requetes-sql-intermediaires/02-regroupement-donnees.md)

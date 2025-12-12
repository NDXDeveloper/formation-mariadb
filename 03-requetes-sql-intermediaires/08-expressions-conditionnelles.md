🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.8 Expressions conditionnelles (CASE, IF, IFNULL, COALESCE, NULLIF)

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 3.1 à 3.7, compréhension des opérateurs logiques

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser l'expression CASE (simple et recherchée)
- Utiliser IF() pour des conditions simples
- Gérer les valeurs NULL avec COALESCE et IFNULL
- Appliquer NULLIF pour des comparaisons spécifiques
- Créer des logiques conditionnelles complexes
- Transformer des données avec des règles métier
- Catégoriser et segmenter des données dynamiquement
- Optimiser les expressions conditionnelles
- Résoudre des problèmes métier avec logique conditionnelle

---

## Introduction

Les **expressions conditionnelles** permettent d'ajouter de la **logique et des conditions** directement dans vos requêtes SQL. Elles sont essentielles pour transformer, catégoriser et adapter les données selon des règles métier.

### Principe fondamental

```
SQL sans conditions          SQL avec conditions
   ⬇                              ⬇
Données brutes              Données transformées
Traitement après            Traitement dans la requête
Code application            Logique SQL
```

### Les 5 expressions principales

| Expression | Usage | Complexité | Cas d'usage |
|------------|-------|------------|-------------|
| **CASE** | Conditions multiples | ⭐⭐⭐ | Catégorisation, transformation |
| **IF()** | Condition simple | ⭐ | Choix binaire (oui/non) |
| **COALESCE()** | Première valeur non-NULL | ⭐⭐ | Valeurs par défaut, fallback |
| **IFNULL()** | NULL vers valeur | ⭐ | Remplacement NULL simple |
| **NULLIF()** | Retourne NULL si égal | ⭐⭐ | Éviter division par zéro |

### Cas d'usage courants

```
📊 CATÉGORISATION          🔄 TRANSFORMATION
   Segments clients           Statuts lisibles
   Niveaux de prix            Formats adaptés
   Priorités                  Normalisation

🎯 RÈGLES MÉTIER           🛡️ GESTION NULL
   Remises dynamiques         Valeurs par défaut
   Alertes conditionnelles    Fallback multiples
   Scoring                    Données manquantes

📈 REPORTING               🔍 FILTRAGE AVANCÉ
   KPI conditionnels          Sélection dynamique
   Formules métier            Exclusions
   Agrégations filtrées       Nettoyage données
```

---

## CASE : L'expression conditionnelle principale

**CASE** est la structure conditionnelle la plus puissante en SQL, équivalente au `if/else if/else` ou `switch/case` en programmation.

### Deux syntaxes : Simple vs Recherchée

#### CASE Simple : Comparaison d'une valeur

```sql
CASE expression
    WHEN valeur1 THEN resultat1
    WHEN valeur2 THEN resultat2
    ...
    ELSE resultat_defaut
END
```

#### CASE Recherchée : Conditions booléennes

```sql
CASE
    WHEN condition1 THEN resultat1
    WHEN condition2 THEN resultat2
    ...
    ELSE resultat_defaut
END
```

💡 **CASE Recherchée est plus flexible** - permet des conditions complexes avec opérateurs (<, >, AND, OR, etc.)

---

### Exemple 1 : CASE Simple - Statuts lisibles

**Question métier** : *Traduire les codes statut en libellés*

```sql
SELECT
    id_commande,
    statut AS code_statut,
    CASE statut
        WHEN 'P' THEN 'En attente de paiement'
        WHEN 'C' THEN 'Confirmée'
        WHEN 'E' THEN 'Expédiée'
        WHEN 'L' THEN 'Livrée'
        WHEN 'A' THEN 'Annulée'
        ELSE 'Statut inconnu'
    END AS statut_lisible
FROM commandes
LIMIT 5;
```

**Résultat attendu** :
```
+-------------+--------------+---------------------------+
| id_commande | code_statut  | statut_lisible            |
+-------------+--------------+---------------------------+
|        1847 | C            | Confirmée                 |
|        1848 | E            | Expédiée                  |
|        1849 | L            | Livrée                    |
|        1850 | P            | En attente de paiement    |
|        1851 | A            | Annulée                   |
+-------------+--------------+---------------------------+
```

**Points clés** :
- CASE Simple compare `statut` à chaque WHEN
- ELSE facultatif mais recommandé (valeur par défaut)
- Peut être utilisé dans SELECT, WHERE, ORDER BY, HAVING

---

### Exemple 2 : CASE Recherchée - Catégories de prix

**Question métier** : *Classer les produits par gamme de prix*

```sql
SELECT
    id_produit,
    nom_produit,
    prix_unitaire,
    CASE
        WHEN prix_unitaire < 50 THEN '💰 Économique'
        WHEN prix_unitaire >= 50 AND prix_unitaire < 200 THEN '⭐ Standard'
        WHEN prix_unitaire >= 200 AND prix_unitaire < 500 THEN '💎 Premium'
        WHEN prix_unitaire >= 500 THEN '👑 Luxe'
        ELSE 'Non classé'
    END AS gamme_prix
FROM produits
ORDER BY prix_unitaire
LIMIT 10;
```

**Résultat attendu** :
```
+------------+----------------------+---------------+----------------+
| id_produit | nom_produit          | prix_unitaire | gamme_prix     |
+------------+----------------------+---------------+----------------+
|        234 | Câble USB-C          |         12.99 | 💰 Économique  |
|        457 | Souris Logitech      |         34.99 | 💰 Économique  |
|        892 | Clavier Mécanique    |         89.99 | ⭐ Standard    |
|        1024| Écran 24"            |        189.99 | ⭐ Standard    |
|        1483| iPhone 15            |        999.00 | 👑 Luxe        |
+------------+----------------------+---------------+----------------+
```

**Avantages CASE Recherchée** :
- Conditions complexes avec AND, OR, NOT
- Comparaisons multiples (>, <, >=, <=, BETWEEN, LIKE, etc.)
- Plus flexible que CASE Simple

---

### Exemple 3 : CASE avec agrégations

**Question métier** : *Compter les commandes par statut*

```sql
SELECT
    COUNT(*) AS total_commandes,
    SUM(CASE WHEN statut = 'L' THEN 1 ELSE 0 END) AS nb_livrees,
    SUM(CASE WHEN statut = 'E' THEN 1 ELSE 0 END) AS nb_expedies,
    SUM(CASE WHEN statut = 'P' THEN 1 ELSE 0 END) AS nb_en_attente,
    SUM(CASE WHEN statut = 'A' THEN 1 ELSE 0 END) AS nb_annulees,
    ROUND(
        SUM(CASE WHEN statut = 'L' THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS taux_livraison
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 30 DAY);
```

**Résultat attendu** :
```
+------------------+-------------+--------------+----------------+-------------+-----------------+
| total_commandes  | nb_livrees  | nb_expedies  | nb_en_attente  | nb_annulees | taux_livraison  |
+------------------+-------------+--------------+----------------+-------------+-----------------+
|             2847 |        2104 |          389 |            234 |         120 |           73.88 |
+------------------+-------------+--------------+----------------+-------------+-----------------+
```

**Technique** : `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` = compteur conditionnel

---

### Exemple 4 : CASE dans WHERE (filtrage dynamique)

**Question métier** : *Sélectionner différemment selon une condition*

```sql
-- Filtrage dynamique selon le type de client
SET @type_recherche = 'VIP';

SELECT
    id_client,
    nom,
    total_achats,
    CASE
        WHEN total_achats >= 10000 THEN 'VIP'
        WHEN total_achats >= 5000 THEN 'Gold'
        WHEN total_achats >= 1000 THEN 'Silver'
        ELSE 'Bronze'
    END AS categorie
FROM clients
WHERE
    CASE
        WHEN @type_recherche = 'VIP' THEN total_achats >= 10000
        WHEN @type_recherche = 'Gold' THEN total_achats >= 5000 AND total_achats < 10000
        WHEN @type_recherche = 'Silver' THEN total_achats >= 1000 AND total_achats < 5000
        ELSE total_achats < 1000
    END;
```

**Application** : Rapports dynamiques, filtres paramétrables.

---

### Exemple 5 : CASE imbriqué (nested)

**Question métier** : *Scoring complexe avec plusieurs critères*

```sql
SELECT
    id_client,
    nom,
    total_achats,
    nb_commandes,
    anciennete_jours,
    CASE
        WHEN total_achats >= 10000 THEN
            CASE
                WHEN nb_commandes >= 50 THEN 'A+ (VIP Fidèle)'
                WHEN nb_commandes >= 20 THEN 'A (VIP Standard)'
                ELSE 'B+ (VIP Occasionnel)'
            END
        WHEN total_achats >= 5000 THEN
            CASE
                WHEN anciennete_jours >= 365 THEN 'B (Gold Ancien)'
                ELSE 'B- (Gold Récent)'
            END
        WHEN total_achats >= 1000 THEN 'C (Silver)'
        ELSE 'D (Bronze)'
    END AS score_client
FROM clients
ORDER BY total_achats DESC
LIMIT 10;
```

**Résultat attendu** :
```
+-----------+----------------+--------------+--------------+------------------+---------------------+
| id_client | nom            | total_achats | nb_commandes | anciennete_jours | score_client        |
+-----------+----------------+--------------+--------------+------------------+---------------------+
|      1847 | Alice Martin   |     34892.45 |           87 |             1247 | A+ (VIP Fidèle)     |
|      2934 | Bob Dupont     |     28473.92 |           34 |              892 | A (VIP Standard)    |
|      3621 | Charlie Durand |     11847.23 |           12 |              234 | B+ (VIP Occasionnel)|
|      4582 | Diana Lopez    |      7293.18 |           23 |              523 | B (Gold Ancien)     |
+-----------+----------------+--------------+--------------+------------------+---------------------+
```

⚠️ **Attention** : CASE imbriqués réduisent la lisibilité. Limiter à 2-3 niveaux maximum.

---

### Exemple 6 : CASE dans ORDER BY

**Question métier** : *Trier avec priorités personnalisées*

```sql
SELECT
    id_commande,
    statut,
    montant_total,
    date_commande
FROM commandes
ORDER BY
    -- Priorité 1 : Statut (ordre personnalisé)
    CASE statut
        WHEN 'P' THEN 1  -- Paiement en attente = urgent
        WHEN 'C' THEN 2  -- Confirmé
        WHEN 'E' THEN 3  -- Expédié
        WHEN 'L' THEN 4  -- Livré
        WHEN 'A' THEN 5  -- Annulé (moins prioritaire)
        ELSE 6
    END,
    -- Priorité 2 : Montant décroissant
    montant_total DESC
LIMIT 10;
```

**Application** : Ordres de traitement, files d'attente, dashboards.

---

## IF() : Condition simple

**IF()** est une fonction conditionnelle pour des **choix binaires** (vrai/faux, oui/non).

```sql
IF(condition, valeur_si_vrai, valeur_si_faux)
```

### Exemple 7 : IF() basique

**Question métier** : *Marquer les produits en rupture*

```sql
SELECT
    id_produit,
    nom_produit,
    stock,
    IF(stock > 0, '✅ En stock', '❌ Rupture') AS disponibilite,
    IF(stock <= 10 AND stock > 0, '⚠️ Stock faible', NULL) AS alerte
FROM produits
ORDER BY stock
LIMIT 10;
```

**Résultat attendu** :
```
+------------+----------------------+-------+---------------+---------------+
| id_produit | nom_produit          | stock | disponibilite | alerte        |
+------------+----------------------+-------+---------------+---------------+
|        234 | Câble HDMI           |     0 | ❌ Rupture    | NULL
|        457 | Adaptateur USB       |     3 | ✅ En stock   | ⚠️ Stock faible
|        892 | Clavier Bluetooth    |     8 | ✅ En stock   | ⚠️ Stock faible
|       1024 | Souris Gaming        |    47 | ✅ En stock   | NULL
+------------+----------------------+-------+---------------+---------------+
```

### Exemple 8 : IF() vs CASE - Comparaison

```sql
-- Avec IF() : Simple mais limité
SELECT
    nom_produit,
    prix_unitaire,
    IF(prix_unitaire > 100, 'Cher', 'Abordable') AS categorie_if
FROM produits;

-- Avec CASE : Plus flexible pour >2 options
SELECT
    nom_produit,
    prix_unitaire,
    CASE
        WHEN prix_unitaire < 50 THEN 'Économique'
        WHEN prix_unitaire < 200 THEN 'Standard'
        WHEN prix_unitaire < 500 THEN 'Premium'
        ELSE 'Luxe'
    END AS categorie_case
FROM produits;
```

**Quand utiliser IF() ?**
- ✅ Condition simple (vrai/faux)
- ✅ Code plus court et lisible
- ❌ Éviter pour plus de 2 résultats possibles

---

## COALESCE() : Première valeur non-NULL

**COALESCE()** retourne la **première valeur non-NULL** dans une liste. Essentiel pour gérer les valeurs manquantes.

```sql
COALESCE(valeur1, valeur2, ..., valeurN)
```

### Exemple 9 : Valeurs par défaut

**Question métier** : *Afficher téléphone avec fallback*

```sql
SELECT
    id_client,
    nom,
    telephone_mobile,
    telephone_fixe,
    telephone_professionnel,
    COALESCE(telephone_mobile, telephone_fixe, telephone_professionnel, 'Aucun') AS telephone_contact
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+----------------+------------------+----------------+--------------------------+-------------------+
| id_client | nom            | telephone_mobile | telephone_fixe | telephone_professionnel  | telephone_contact |
+-----------+----------------+------------------+----------------+--------------------------+-------------------+
|      1847 | Alice Martin   | 0601020304       | NULL           | NULL                     | 0601020304        |
|      2934 | Bob Dupont     | NULL             | 0142847593     | NULL                     | 0142847593        |
|      3621 | Charlie Durand | NULL             | NULL           | 0147382947               | 0147382947        |
|      4582 | Diana Lopez    | NULL             | NULL           | NULL                     | Aucun             |
+-----------+----------------+------------------+----------------+--------------------------+-------------------+
```

**Ordre important** : COALESCE teste de gauche à droite et s'arrête au premier non-NULL.

### Exemple 10 : COALESCE dans calculs

**Question métier** : *Éviter NULL dans sommes*

```sql
SELECT
    id_commande,
    montant_produits,
    frais_livraison,
    reduction,
    -- ❌ Si une valeur est NULL, total = NULL
    montant_produits + frais_livraison - reduction AS total_mauvais,
    -- ✅ Traiter NULL comme 0
    COALESCE(montant_produits, 0) +
    COALESCE(frais_livraison, 0) -
    COALESCE(reduction, 0) AS total_correct
FROM commandes
LIMIT 5;
```

**Résultat attendu** :
```
+-------------+------------------+-----------------+-----------+---------------+---------------+
| id_commande | montant_produits | frais_livraison | reduction | total_mauvais | total_correct |
+-------------+------------------+-----------------+-----------+---------------+---------------+
|        1847 |           289.95 |            5.99 |      NULL | NULL          |        295.94 |
|        1848 |           149.99 |            NULL |     10.00 | NULL          |        139.99 |
|        1849 |            89.50 |            5.99 |     15.00 |         80.49 |         80.49 |
+-------------+------------------+-----------------+-----------+---------------+---------------+
```

### Exemple 11 : COALESCE avec requêtes

**Question métier** : *Adresse de livraison ou facturation*

```sql
SELECT
    id_client,
    nom,
    COALESCE(adresse_livraison, adresse_facturation) AS adresse_utilisee,
    CASE
        WHEN adresse_livraison IS NOT NULL THEN 'Livraison'
        WHEN adresse_facturation IS NOT NULL THEN 'Facturation'
        ELSE 'Aucune'
    END AS type_adresse
FROM clients
WHERE COALESCE(adresse_livraison, adresse_facturation) IS NOT NULL
LIMIT 5;
```

---

## IFNULL() : Remplacement NULL simple

**IFNULL()** est une version simplifiée de COALESCE avec **seulement 2 arguments**.

```sql
IFNULL(expression, valeur_si_null)
```

### Exemple 12 : IFNULL vs COALESCE

```sql
SELECT
    id_produit,
    nom_produit,
    stock,
    -- IFNULL : 2 arguments
    IFNULL(stock, 0) AS stock_ifnull,
    -- COALESCE : N arguments (plus flexible)
    COALESCE(stock, stock_reserve, 0) AS stock_coalesce
FROM produits
LIMIT 5;
```

**Différences** :

| Aspect | IFNULL | COALESCE |
|--------|--------|----------|
| **Nombre arguments** | Exactement 2 | 2 ou plus |
| **Standard SQL** | ❌ Non (MariaDB/MySQL) | ✅ Oui |
| **Performance** | Légèrement plus rapide | Négligeable |
| **Lisibilité** | Simple pour 1 NULL | Clair pour multiple fallbacks |

💡 **Recommandation** : Utiliser **COALESCE** pour portabilité et flexibilité.

---

## NULLIF() : Retourner NULL conditionnellement

**NULLIF()** retourne NULL si deux valeurs sont égales, sinon la première valeur.

```sql
NULLIF(expression1, expression2)
-- Retourne NULL si expression1 = expression2
-- Sinon retourne expression1
```

### Exemple 13 : Éviter division par zéro

**Question métier** : *Calculer taux de conversion sans erreur*

```sql
SELECT
    id_campagne,
    nom_campagne,
    nb_visiteurs,
    nb_conversions,
    -- ❌ Division par zéro si nb_visiteurs = 0
    nb_conversions / nb_visiteurs AS taux_erreur,
    -- ✅ NULLIF transforme 0 en NULL, évite erreur
    nb_conversions / NULLIF(nb_visiteurs, 0) AS taux_correct,
    -- ✅ Avec COALESCE pour afficher 0 au lieu de NULL
    COALESCE(
        nb_conversions / NULLIF(nb_visiteurs, 0),
        0
    ) AS taux_avec_defaut
FROM campagnes_marketing
LIMIT 5;
```

**Résultat attendu** :
```
+-------------+-----------------+--------------+-----------------+-------------+--------------+------------------+
| id_campagne | nom_campagne    | nb_visiteurs | nb_conversions  | taux_erreur | taux_correct | taux_avec_defaut |
+-------------+-----------------+--------------+-----------------+-------------+--------------+------------------+
|           1 | Black Friday    |        12847 |             389 |      0.0303 |       0.0303 |           0.0303 |
|           2 | Soldes Été      |            0 |               0 | ERROR       | NULL         |           0.0000 |
|           3 | Noël            |         8493 |             234 |      0.0275 |       0.0275 |           0.0275 |
+-------------+-----------------+--------------+-----------------+-------------+--------------+------------------+
```

**Pattern courant** :
```sql
-- Éviter division par zéro
valeur / NULLIF(diviseur, 0)

-- Avec valeur par défaut
COALESCE(valeur / NULLIF(diviseur, 0), 0)
```

### Exemple 14 : NULLIF pour normalisation

**Question métier** : *Transformer chaînes vides en NULL*

```sql
UPDATE clients
SET
    telephone_mobile = NULLIF(TRIM(telephone_mobile), ''),
    telephone_fixe = NULLIF(TRIM(telephone_fixe), ''),
    adresse_complement = NULLIF(TRIM(adresse_complement), '')
WHERE telephone_mobile = ''
   OR telephone_fixe = ''
   OR adresse_complement = '';
```

**Application** : Nettoyage de données, imports, normalisation.

---

## Cas d'usage métier avancés

### Cas 1 : Segmentation RFM (Recency, Frequency, Monetary)

**Question métier** : *Segmenter clients selon comportement d'achat*

```sql
WITH client_rfm AS (
    SELECT
        c.id_client,
        c.nom,
        DATEDIFF(CURDATE(), MAX(cmd.date_commande)) AS recence_jours,
        COUNT(cmd.id_commande) AS frequence,
        SUM(cmd.montant_total) AS montant_total
    FROM clients c
    LEFT JOIN commandes cmd ON c.id_client = cmd.id_client
    GROUP BY c.id_client, c.nom
)
SELECT
    id_client,
    nom,
    recence_jours,
    frequence,
    montant_total,
    -- Score Récence (1-5)
    CASE
        WHEN recence_jours IS NULL THEN 0
        WHEN recence_jours <= 30 THEN 5
        WHEN recence_jours <= 90 THEN 4
        WHEN recence_jours <= 180 THEN 3
        WHEN recence_jours <= 365 THEN 2
        ELSE 1
    END AS score_recence,
    -- Score Fréquence (1-5)
    CASE
        WHEN frequence IS NULL OR frequence = 0 THEN 0
        WHEN frequence >= 50 THEN 5
        WHEN frequence >= 20 THEN 4
        WHEN frequence >= 10 THEN 3
        WHEN frequence >= 5 THEN 2
        ELSE 1
    END AS score_frequence,
    -- Score Montant (1-5)
    CASE
        WHEN montant_total IS NULL THEN 0
        WHEN montant_total >= 10000 THEN 5
        WHEN montant_total >= 5000 THEN 4
        WHEN montant_total >= 2000 THEN 3
        WHEN montant_total >= 500 THEN 2
        ELSE 1
    END AS score_montant,
    -- Segment final
    CASE
        WHEN recence_jours IS NULL THEN 'Jamais commandé'
        WHEN CASE WHEN recence_jours <= 30 THEN 5 WHEN recence_jours <= 90 THEN 4 ELSE 0 END +
             CASE WHEN frequence >= 20 THEN 5 WHEN frequence >= 10 THEN 4 ELSE 0 END +
             CASE WHEN montant_total >= 5000 THEN 5 WHEN montant_total >= 2000 THEN 4 ELSE 0 END >= 12
            THEN '🏆 Champions'
        WHEN recence_jours <= 90 AND montant_total >= 2000
            THEN '⭐ Fidèles'
        WHEN recence_jours <= 180 AND frequence >= 5
            THEN '🎯 Prometteurs'
        WHEN recence_jours > 365
            THEN '😴 Dormants'
        WHEN recence_jours > 180
            THEN '⚠️ À risque'
        ELSE '🌱 Nouveaux'
    END AS segment
FROM client_rfm
ORDER BY
    CASE segment
        WHEN '🏆 Champions' THEN 1
        WHEN '⭐ Fidèles' THEN 2
        WHEN '🎯 Prometteurs' THEN 3
        WHEN '🌱 Nouveaux' THEN 4
        WHEN '⚠️ À risque' THEN 5
        WHEN '😴 Dormants' THEN 6
        ELSE 7
    END,
    montant_total DESC
LIMIT 10;
```

**Résultat attendu** :
```
+-----------+----------------+---------------+-----------+--------------+---------------+-----------------+--------------+----------------+
| id_client | nom            | recence_jours | frequence | montant_total| score_recence | score_frequence | score_montant| segment        |
+-----------+----------------+---------------+-----------+--------------+---------------+-----------------+--------------+----------------+
|      1847 | Alice Martin   |            23 |        87 |     34892.45 |             5 |               5 |            5 | 🏆 Champions   |
|      2934 | Bob Dupont     |            45 |        54 |     28473.92 |             4 |               5 |            5 | 🏆 Champions   |
|      3621 | Charlie Durand |            67 |        34 |     11847.23 |             4 |               4 |            5 | ⭐ Fidèles     |
|      4582 | Diana Lopez    |           234 |        23 |      7293.18 |             2 |               4 |            4 | ⚠️ À risque    |
|      5193 | Éric Moreau    |           487 |        12 |      3482.91 |             1 |               3 |            3 | 😴 Dormants    |
+-----------+----------------+---------------+-----------+--------------+---------------+-----------------+--------------+----------------+
```

### Cas 2 : Remises dynamiques complexes

**Question métier** : *Calculer remise selon règles métier*

```sql
SELECT
    id_commande,
    id_client,
    categorie_client,
    montant_brut,
    -- Règles de remise empilées
    CASE
        -- VIP : 20% sur tout
        WHEN categorie_client = 'VIP' THEN montant_brut * 0.20
        -- Gold : 15% si > 500€
        WHEN categorie_client = 'Gold' AND montant_brut >= 500 THEN montant_brut * 0.15
        -- Gold : 10% sinon
        WHEN categorie_client = 'Gold' THEN montant_brut * 0.10
        -- Silver : 5% si > 200€
        WHEN categorie_client = 'Silver' AND montant_brut >= 200 THEN montant_brut * 0.05
        -- Première commande : 10€ de réduction
        WHEN est_premiere_commande = 1 THEN 10.00
        -- Black Friday : 15% sur tout
        WHEN DATE(date_commande) BETWEEN '2025-11-29' AND '2025-12-02' THEN montant_brut * 0.15
        -- Aucune remise
        ELSE 0
    END AS remise_calculee,
    -- Montant final
    montant_brut -
    CASE
        WHEN categorie_client = 'VIP' THEN montant_brut * 0.20
        WHEN categorie_client = 'Gold' AND montant_brut >= 500 THEN montant_brut * 0.15
        WHEN categorie_client = 'Gold' THEN montant_brut * 0.10
        WHEN categorie_client = 'Silver' AND montant_brut >= 200 THEN montant_brut * 0.05
        WHEN est_premiere_commande = 1 THEN 10.00
        WHEN DATE(date_commande) BETWEEN '2025-11-29' AND '2025-12-02' THEN montant_brut * 0.15
        ELSE 0
    END AS montant_final
FROM commandes
LIMIT 5;
```

### Cas 3 : Statut de livraison avec alertes

**Question métier** : *Statut enrichi avec délais et alertes*

```sql
SELECT
    id_commande,
    date_commande,
    date_livraison_prevue,
    date_livraison_reelle,
    statut,
    -- Jours depuis commande
    DATEDIFF(COALESCE(date_livraison_reelle, CURDATE()), date_commande) AS jours_depuis_commande,
    -- Statut enrichi
    CASE
        -- Livrée
        WHEN date_livraison_reelle IS NOT NULL THEN
            CASE
                WHEN date_livraison_reelle <= date_livraison_prevue THEN '✅ Livrée à temps'
                WHEN DATEDIFF(date_livraison_reelle, date_livraison_prevue) <= 2 THEN '⚠️ Livrée avec léger retard'
                ELSE '❌ Livrée en retard'
            END
        -- En cours
        WHEN statut IN ('C', 'E') THEN
            CASE
                WHEN CURDATE() > date_livraison_prevue THEN '🔴 En retard'
                WHEN DATEDIFF(date_livraison_prevue, CURDATE()) <= 1 THEN '🟡 Urgent'
                ELSE '🟢 Dans les temps'
            END
        -- Annulée
        WHEN statut = 'A' THEN '❌ Annulée'
        -- Autres
        ELSE 'En attente'
    END AS statut_enrichi,
    -- Priorité de traitement
    CASE
        WHEN statut IN ('C', 'E') AND CURDATE() > date_livraison_prevue THEN 1
        WHEN statut IN ('C', 'E') AND DATEDIFF(date_livraison_prevue, CURDATE()) <= 1 THEN 2
        WHEN statut IN ('C', 'E') THEN 3
        ELSE 4
    END AS priorite_traitement
FROM commandes
WHERE date_livraison_reelle IS NULL OR date_livraison_reelle >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
ORDER BY priorite_traitement, date_livraison_prevue;
```

### Cas 4 : Rapport de ventes avec comparaisons

**Question métier** : *Analyse avec évolution vs période précédente*

```sql
WITH ventes_mois AS (
    SELECT
        DATE_FORMAT(date_commande, '%Y-%m') AS mois,
        COUNT(*) AS nb_commandes,
        SUM(montant_total) AS ca_total
    FROM commandes
    WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
    GROUP BY mois
)
SELECT
    mois,
    nb_commandes,
    FORMAT(ca_total, 2) AS ca_total_format,
    LAG(ca_total) OVER (ORDER BY mois) AS ca_mois_precedent,
    -- Évolution en valeur
    ca_total - LAG(ca_total) OVER (ORDER BY mois) AS evolution_valeur,
    -- Évolution en %
    ROUND(
        (ca_total - LAG(ca_total) OVER (ORDER BY mois)) * 100.0 /
        NULLIF(LAG(ca_total) OVER (ORDER BY mois), 0),
        2
    ) AS evolution_pct,
    -- Interprétation
    CASE
        WHEN LAG(ca_total) OVER (ORDER BY mois) IS NULL THEN 'Première période'
        WHEN (ca_total - LAG(ca_total) OVER (ORDER BY mois)) * 100.0 /
             NULLIF(LAG(ca_total) OVER (ORDER BY mois), 0) >= 10 THEN '📈 Forte croissance'
        WHEN (ca_total - LAG(ca_total) OVER (ORDER BY mois)) * 100.0 /
             NULLIF(LAG(ca_total) OVER (ORDER BY mois), 0) > 0 THEN '↗️ Croissance'
        WHEN (ca_total - LAG(ca_total) OVER (ORDER BY mois)) * 100.0 /
             NULLIF(LAG(ca_total) OVER (ORDER BY mois), 0) >= -10 THEN '↘️ Baisse légère'
        ELSE '📉 Forte baisse'
    END AS tendance
FROM ventes_mois
ORDER BY mois DESC
LIMIT 6;
```

---

## Performance et optimisation

### Impact sur les index

⚠️ **Règle critique** : Expressions conditionnelles sur colonnes indexées **empêchent l'utilisation des index**.

```sql
-- ❌ LENT : Index non utilisé
SELECT * FROM produits
WHERE CASE
    WHEN categorie = 'Électronique' THEN prix_unitaire > 100
    ELSE prix_unitaire > 50
END;

-- ✅ RAPIDE : Réécrire sans CASE
SELECT * FROM produits
WHERE (categorie = 'Électronique' AND prix_unitaire > 100)
   OR (categorie != 'Électronique' AND prix_unitaire > 50);
```

### CASE dans SELECT vs WHERE

```sql
-- ✅ CASE dans SELECT : OK, pas d'impact performance
SELECT
    id_produit,
    CASE
        WHEN prix_unitaire < 100 THEN 'Abordable'
        ELSE 'Cher'
    END AS categorie
FROM produits
WHERE prix_unitaire > 0;  -- Index utilisé

-- ⚠️ CASE dans WHERE : Peut ralentir
SELECT * FROM produits
WHERE CASE
    WHEN categorie = 'Promo' THEN prix_unitaire < 50
    ELSE prix_unitaire < 100
END;
```

### Simplification de CASE complexes

```sql
-- ❌ CASE imbriqué complexe (lent, illisible)
SELECT
    CASE
        WHEN condition1 THEN
            CASE
                WHEN condition2 THEN
                    CASE
                        WHEN condition3 THEN 'Résultat1'
                        ELSE 'Résultat2'
                    END
                ELSE 'Résultat3'
            END
        ELSE 'Résultat4'
    END AS resultat
FROM table;

-- ✅ CASE séquentiel (plus rapide, plus clair)
SELECT
    CASE
        WHEN condition1 AND condition2 AND condition3 THEN 'Résultat1'
        WHEN condition1 AND condition2 THEN 'Résultat2'
        WHEN condition1 THEN 'Résultat3'
        ELSE 'Résultat4'
    END AS resultat
FROM table;
```

### Table de référence vs CASE

Pour des correspondances fixes, une table de référence est souvent plus performante et maintenable.

```sql
-- ❌ CASE répété partout
SELECT
    id_produit,
    CASE statut
        WHEN 'A' THEN 'Actif'
        WHEN 'I' THEN 'Inactif'
        WHEN 'O' THEN 'Obsolète'
        ELSE 'Inconnu'
    END AS statut_libelle
FROM produits;

-- ✅ Jointure avec table de référence
CREATE TABLE ref_statuts (
    code CHAR(1) PRIMARY KEY,
    libelle VARCHAR(50)
);

INSERT INTO ref_statuts VALUES
    ('A', 'Actif'),
    ('I', 'Inactif'),
    ('O', 'Obsolète');

SELECT
    p.id_produit,
    COALESCE(rs.libelle, 'Inconnu') AS statut_libelle
FROM produits p
LEFT JOIN ref_statuts rs ON p.statut = rs.code;
```

---

## Pièges courants et solutions

### Piège 1 : CASE sans ELSE

```sql
-- ❌ Si aucun WHEN ne correspond → NULL
SELECT
    nom_produit,
    CASE prix_unitaire
        WHEN 100 THEN 'Cent euros'
        WHEN 200 THEN 'Deux cents euros'
    END AS prix_texte  -- NULL si prix != 100 et != 200
FROM produits;

-- ✅ Toujours inclure ELSE
SELECT
    nom_produit,
    CASE prix_unitaire
        WHEN 100 THEN 'Cent euros'
        WHEN 200 THEN 'Deux cents euros'
        ELSE CONCAT(prix_unitaire, ' euros')
    END AS prix_texte
FROM produits;
```

### Piège 2 : Types de données incompatibles

```sql
-- ❌ Types différents dans les résultats
SELECT
    CASE statut
        WHEN 'A' THEN 1          -- INT
        WHEN 'I' THEN 'Inactif'  -- VARCHAR
        ELSE NULL
    END AS resultat;
-- Résultat converti en VARCHAR, 1 devient '1'

-- ✅ Homogénéité des types
SELECT
    CASE statut
        WHEN 'A' THEN 'Actif'
        WHEN 'I' THEN 'Inactif'
        ELSE 'Inconnu'
    END AS resultat;
```

### Piège 3 : COALESCE avec types incompatibles

```sql
-- ⚠️ COALESCE peut convertir types
SELECT COALESCE(NULL, 0, 'zéro');  -- Retourne '0' (VARCHAR)

-- ✅ Assurer homogénéité
SELECT COALESCE(valeur_numerique, 0);  -- Tous numériques
SELECT COALESCE(valeur_texte, 'N/A');  -- Tous VARCHAR
```

### Piège 4 : NULL dans conditions CASE

```sql
-- ❌ NULL n'est jamais égal à NULL
SELECT
    CASE valeur
        WHEN NULL THEN 'Est NULL'  -- Ne sera JAMAIS vrai
        ELSE 'Autre'
    END;

-- ✅ Utiliser IS NULL
SELECT
    CASE
        WHEN valeur IS NULL THEN 'Est NULL'
        ELSE 'Autre'
    END;
```

### Piège 5 : Division par zéro sans NULLIF

```sql
-- ❌ Erreur si diviseur = 0
SELECT montant / diviseur FROM table;

-- ✅ NULLIF pour éviter erreur
SELECT montant / NULLIF(diviseur, 0) FROM table;

-- ✅ Avec valeur par défaut
SELECT COALESCE(montant / NULLIF(diviseur, 0), 0) FROM table;
```

### Piège 6 : CASE dans agrégations

```sql
-- ⚠️ Attention à l'ordre
SELECT
    COUNT(CASE WHEN condition THEN 1 END) AS compte_conditionnel,
    -- Équivalent à :
    SUM(CASE WHEN condition THEN 1 ELSE 0 END) AS compte_sum
FROM table;

-- ❌ ERREUR fréquente
SELECT
    COUNT(CASE WHEN condition THEN 1 ELSE 0 END);
-- Compte 1 ET 0 → mauvais résultat

-- ✅ CORRECT
SELECT
    COUNT(CASE WHEN condition THEN 1 ELSE NULL END);
-- ou
SELECT
    SUM(CASE WHEN condition THEN 1 ELSE 0 END);
```

---

## Bonnes pratiques

### ✅ DO : Recommandations

1. **CASE Recherchée pour flexibilité** : Préférer aux CASE Simple pour conditions complexes

2. **ELSE systématique** : Toujours inclure pour éviter NULL inattendu

3. **COALESCE pour portabilité** : Standard SQL vs IFNULL (MySQL/MariaDB)

4. **NULLIF pour divisions** : Éviter erreurs division par zéro

5. **Types homogènes** : Tous les THEN/ELSE doivent retourner même type

6. **Simplifier avant CASE** : Filtrer avec WHERE quand possible

7. **Tables de référence** : Pour correspondances fixes réutilisées

8. **Documenter logique complexe** : Commentaires pour CASE imbriqués

### ❌ DON'T : À éviter

1. **CASE dans WHERE avec index** : Empêche utilisation index

2. **CASE imbriqués excessifs** : Limiter à 2-3 niveaux maximum

3. **IF() pour >2 options** : Utiliser CASE plus lisible

4. **Oublier NULL** : `WHEN NULL` ne fonctionne pas, utiliser `IS NULL`

5. **Types incohérents** : Mélanger INT et VARCHAR dans résultats

6. **CASE répétés** : Factoriser avec CTE ou colonnes calculées

7. **Dupliquer logique** : Créer fonction si réutilisée souvent

8. **Négliger performance** : Mesurer avec EXPLAIN sur gros volumes

---

## Tableau récapitulatif

| Expression | Syntaxe | Usage principal | Complexité |
|------------|---------|-----------------|------------|
| **CASE** | `CASE WHEN ... THEN ... END` | Conditions multiples | ⭐⭐⭐ |
| **IF()** | `IF(condition, vrai, faux)` | Choix binaire | ⭐ |
| **COALESCE()** | `COALESCE(v1, v2, ..., vN)` | Première non-NULL | ⭐⭐ |
| **IFNULL()** | `IFNULL(expr, defaut)` | NULL → valeur | ⭐ |
| **NULLIF()** | `NULLIF(expr1, expr2)` | NULL si égal | ⭐⭐ |

### Comparaisons

| Besoin | Solution recommandée | Alternative |
|--------|---------------------|-------------|
| 2 résultats possibles | IF() | CASE (plus verbeux) |
| 3+ résultats | CASE | IF imbriqués (illisible) |
| Valeur par défaut | COALESCE() | IFNULL() |
| Division sécurisée | NULLIF() | CASE |
| Vide → NULL | NULLIF(TRIM(x), '') | IF/CASE |
| Multiple fallbacks | COALESCE(a,b,c) | IF/CASE imbriqués |

---

## ✅ Points clés à retenir

1. **CASE = if/else SQL** : Conditions multiples avec WHEN/THEN/ELSE

2. **CASE Simple vs Recherchée** : Simple pour égalité, Recherchée pour conditions complexes

3. **ELSE recommandé** : Évite NULL inattendu si aucun WHEN ne correspond

4. **IF() pour choix binaire** : Plus simple que CASE pour vrai/faux

5. **COALESCE première non-NULL** : Fallback multiple, standard SQL

6. **IFNULL alternative simple** : Équivalent COALESCE avec 2 arguments

7. **NULLIF évite division par zéro** : Transforme valeur en NULL conditionnellement

8. **Types homogènes obligatoires** : Tous résultats THEN/ELSE même type

9. **Éviter CASE dans WHERE** : Empêche usage index sur gros volumes

10. **NULL ≠ NULL** : Utiliser IS NULL, pas `WHEN NULL`

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CASE Operator](https://mariadb.com/kb/en/case-operator/) – CASE détaillé
- [📖 IF Function](https://mariadb.com/kb/en/if-function/) – IF()
- [📖 COALESCE](https://mariadb.com/kb/en/coalesce/) – Gestion NULL
- [📖 Control Flow Functions](https://mariadb.com/kb/en/control-flow-functions/) – Vue d'ensemble

### Articles approfondis
- [CASE Performance](https://use-the-index-luke.com/sql/where-clause/obfuscation/smart-logic) – Impact index
- [NULL Handling](https://modern-sql.com/feature/is-distinct-from) – Comparaisons NULL

---

## 🎓 Conclusion du Chapitre 3 : Requêtes SQL Intermédiaires

**Félicitations !** Vous avez complété l'intégralité du **chapitre 3** sur les **Requêtes SQL Intermédiaires**. 🎉

### ✅ Compétences maîtrisées

**8 sections complétées** :

1. ✅ **Agrégations (3.1)** – COUNT, SUM, AVG, MIN, MAX
2. ✅ **Regroupements (3.2)** – GROUP BY, HAVING, filtrage avancé
3. ✅ **Jointures (3.3)** – INNER, LEFT, RIGHT, CROSS, Self-Join
4. ✅ **Sous-requêtes (3.4)** – Scalaires, listes, corrélées, EXISTS
5. ✅ **Opérateurs ensemblistes (3.5)** – UNION, INTERSECT, EXCEPT
6. ✅ **Fonctions chaînes (3.6)** – CONCAT, SUBSTRING, TRIM, REPLACE
7. ✅ **Fonctions dates (3.7)** – DATE_ADD, DATEDIFF, DATE_FORMAT
8. ✅ **Expressions conditionnelles (3.8)** – CASE, IF, COALESCE, NULLIF

### 🎯 Vous êtes maintenant capable de :

- ✅ Analyser et agréger des données complexes
- ✅ Croiser des informations de multiples tables
- ✅ Manipuler texte et dates avec précision
- ✅ Appliquer de la logique conditionnelle dans SQL
- ✅ Consolider des résultats de sources variées
- ✅ Résoudre **90% des besoins métier** en SQL
- ✅ Optimiser les performances de vos requêtes
- ✅ Éviter les pièges courants

### 📈 Progression de votre formation

```
Niveau Débutant    ████████████████░░░░ 100% ✅
Niveau Intermédiaire ████████████████░░░░ 100% ✅
Niveau Avancé      ░░░░░░░░░░░░░░░░░░░░   0% 🎯

Progression globale : ███████████░░░░░░░░░ 55%
```

### 🏆 Ce que vous savez faire maintenant

**Requêtes simples** → **Requêtes complexes**
- ✅ Sélections basiques → Analyses multi-tables
- ✅ Filtres simples → Conditions imbriquées
- ✅ Tri → Agrégations et pivots
- ✅ Affichage brut → Formatage sophistiqué

**Vous maîtrisez** :
- 40+ fonctions SQL essentielles
- 5 types de jointures
- 6 types d'opérateurs
- Gestion complète des NULL
- Logique conditionnelle avancée
- Optimisation de performances

---

## ➡️ Chapitre suivant

**Chapitre 4 : Requêtes SQL Avancées**

Prêt pour le **niveau Avancé** ? Le prochain chapitre vous emmène vers l'expertise :

### 📚 Au programme du Chapitre 4

- **Window Functions (4.1)** – ROW_NUMBER, RANK, LAG, LEAD, cumuls glissants
- **CTE et Récursivité (4.2)** – WITH, WITH RECURSIVE, hiérarchies illimitées
- **Transactions (4.3)** – ACID, COMMIT, ROLLBACK, isolation
- **Index avancés (4.4)** – Optimisation, EXPLAIN, types d'index
- **Vues matérialisées (4.5)** – Cache de résultats, performances
- **JSON (4.6)** – Données semi-structurées, path expressions
- **Full-Text Search (4.7)** – Recherche textuelle puissante
- **Expressions régulières (4.8)** – REGEXP avancé

### 🎯 Objectif du Chapitre 4

Devenir un **expert SQL** capable de :
- Résoudre des problèmes complexes élégamment
- Optimiser des requêtes sur millions de lignes
- Architecturer des bases de données performantes
- Maîtriser les fonctionnalités avancées de MariaDB

---

## 💡 Avant de continuer

### Consolidez vos acquis

1. **Relisez les sections** qui vous semblent complexes
2. **Testez sur vos propres données** pour ancrer les concepts
3. **Comparez les approches** (jointures vs sous-requêtes, CASE vs tables de référence)
4. **Mesurez les performances** avec EXPLAIN sur vos requêtes

### Ressources complémentaires

- 📖 [MariaDB Knowledge Base](https://mariadb.com/kb/en/) – Documentation officielle
- 🎓 [SQL Performance Explained](https://use-the-index-luke.com/) – Optimisation
- 💻 [SQL Fiddle](http://sqlfiddle.com/) – Tester rapidement du SQL
- 🔧 [EXPLAIN Analyzer](https://mariadb.com/kb/en/explain/) – Comprendre les plans d'exécution

---

**Bravo pour votre progression !** Vous avez acquis des compétences SQL solides qui vous serviront dans tous vos projets de données. Le niveau avancé vous attend pour aller encore plus loin. 🚀

**Prêt à devenir un expert ?** Rendez-vous au Chapitre 4 ! ⚡

---


⏭️ [Concepts Avancés SQL](/04-concepts-avances-sql/README.md)

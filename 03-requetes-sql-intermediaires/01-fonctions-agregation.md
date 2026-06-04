🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Fonctions d'agrégation (COUNT, SUM, AVG, MIN, MAX)

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.1  
> Version de référence : **MariaDB 12.3 LTS**

---

## Qu'est-ce qu'une fonction d'agrégation ?

Jusqu'ici, une requête `SELECT` renvoyait *une ligne de résultat par ligne lue*. Une **fonction d'agrégation** fonctionne différemment : elle parcourt un ensemble de lignes et le **réduit à une seule valeur**. C'est l'outil de base pour répondre à des questions de synthèse : « combien de commandes ? », « quel est le chiffre d'affaires total ? », « quel est le panier moyen ? », « quelle est la vente la plus élevée ? ».

Appliquée sans regroupement (sans `GROUP BY`, abordé en [section 3.2](02-regroupement-donnees.md)), une fonction d'agrégation condense **toute la table** en une unique ligne.

MariaDB propose cinq fonctions d'agrégation fondamentales :

| Fonction | Rôle | S'applique à |
|----------|------|--------------|
| `COUNT()` | Compte des lignes ou des valeurs | tout type |
| `SUM()` | Additionne des valeurs | numérique |
| `AVG()` | Calcule la moyenne | numérique |
| `MIN()` | Renvoie la plus petite valeur | numérique, texte, date/heure |
| `MAX()` | Renvoie la plus grande valeur | numérique, texte, date/heure |

---

## Jeu de données utilisé dans les exemples

Les exemples de cette section s'appuient sur une table `commande` volontairement réduite. La colonne `remise` contient des valeurs `NULL`, ce qui permettra d'illustrer un point essentiel.

```sql
CREATE TABLE commande (
    id        INT PRIMARY KEY,
    client    VARCHAR(50),
    region    VARCHAR(20),
    montant   DECIMAL(10,2),
    remise    DECIMAL(10,2),   -- peut être NULL
    date_cmd  DATE
);
```

| id | client | region | montant | remise | date_cmd |
|----|--------|--------|---------|--------|----------|
| 1 | Dupont | Nord | 120.00 | 10.00 | 2026-01-05 |
| 2 | Martin | Sud | 80.50 | *NULL* | 2026-01-07 |
| 3 | Dupont | Nord | 200.00 | 25.00 | 2026-02-10 |
| 4 | Leroy | Est | 80.50 | *NULL* | 2026-02-15 |
| 5 | Martin | Sud | 150.00 | 15.00 | 2026-03-01 |

---

## COUNT() — compter

`COUNT()` est la seule fonction d'agrégation qui ne s'intéresse pas à la *valeur* mais à la *présence* de lignes. Elle se décline en trois formes, qu'il est crucial de distinguer.

```sql
SELECT
    COUNT(*)               AS nb_lignes,
    COUNT(remise)          AS nb_remises,
    COUNT(DISTINCT region) AS nb_regions
FROM commande;
```

| nb_lignes | nb_remises | nb_regions |
|-----------|------------|------------|
| 5 | 3 | 3 |

Trois comportements à retenir :

- **`COUNT(*)`** compte **toutes les lignes**, y compris celles contenant des `NULL`. Ici, 5 lignes. C'est la forme à privilégier pour « combien de lignes ? ». La variante `COUNT(1)` lui est strictement équivalente.
- **`COUNT(colonne)`** compte uniquement les lignes où la colonne **n'est pas `NULL`**. La colonne `remise` n'est renseignée que pour 3 commandes : le résultat est donc 3, et non 5. C'est l'erreur classique : `COUNT(remise)` ne répond pas à « combien de commandes ? » mais à « combien de commandes ont une remise ? ».
- **`COUNT(DISTINCT colonne)`** compte le nombre de **valeurs distinctes non `NULL`**. Les régions présentes sont *Nord*, *Sud* et *Est* : le résultat est 3.

> ⚠️ **Performance — `COUNT(*)` sur InnoDB.** Contrairement à certaines idées reçues, InnoDB ne maintient pas un compteur de lignes exact (conséquence du modèle MVCC, voir [section 6.6](../06-transactions-et-concurrence/06-mvcc.md)). Un `COUNT(*)` sans filtre doit donc parcourir un index complet : sur une très grande table, l'opération n'est pas instantanée. Pour des estimations rapides, on peut consulter la colonne `TABLE_ROWS` de [`INFORMATION_SCHEMA.TABLES`](../09-vues-et-donnees-virtuelles/07.1-information-schema.md), en gardant à l'esprit qu'il s'agit d'une approximation.

---

## SUM() — additionner

`SUM()` additionne les valeurs d'une expression numérique. Comme toutes les fonctions ci-dessous (sauf `COUNT(*)`), elle **ignore les valeurs `NULL`**.

```sql
SELECT
    SUM(montant) AS chiffre_affaires,
    SUM(remise)  AS total_remises
FROM commande;
```

| chiffre_affaires | total_remises |
|------------------|---------------|
| 631.00 | 50.00 |

Le chiffre d'affaires (`120 + 80.50 + 200 + 80.50 + 150`) vaut 631.00. Le total des remises ne porte que sur les trois valeurs renseignées (`10 + 25 + 15 = 50.00`) : les deux `NULL` sont simplement écartés, ils ne sont **pas** traités comme des zéros.

On peut dédupliquer avec `DISTINCT` :

```sql
SELECT SUM(DISTINCT montant) AS somme_montants_distincts
FROM commande;
```

Le montant `80.50` apparaît deux fois (commandes 2 et 4) mais n'est compté qu'une seule fois : le résultat est `120 + 80.50 + 200 + 150 = 550.50`, à comparer aux 631.00 de `SUM(montant)`.

Côté type de retour, `SUM()` sur une expression exacte (`INT`, `DECIMAL`) renvoie un `DECIMAL`, ce qui préserve la précision ; sur une expression approchée (`FLOAT`, `DOUBLE`), elle renvoie un `DOUBLE`.

---

## AVG() — moyenne

`AVG()` calcule la moyenne arithmétique. Conceptuellement, elle équivaut à `SUM(expr) / COUNT(expr)` — et c'est précisément ce point qui mérite l'attention : **le dénominateur est le nombre de valeurs non `NULL`**, pas le nombre total de lignes.

```sql
SELECT
    AVG(montant) AS panier_moyen,
    AVG(remise)  AS remise_moyenne
FROM commande;
```

| panier_moyen | remise_moyenne |
|--------------|----------------|
| 126.200000 | 16.666667 |

Le panier moyen vaut `631.00 / 5 = 126.20`. En revanche, la remise moyenne vaut `50.00 / 3 ≈ 16.67`, car `AVG` divise par les **3** remises réellement renseignées, et non par les 5 commandes. Si l'on souhaite au contraire une moyenne « sur l'ensemble des commandes » (en considérant une absence de remise comme une remise nulle), il faut remplacer explicitement les `NULL` par `0`, par exemple avec `COALESCE` (voir [section 3.8](08-expressions-conditionnelles.md)) :

```sql
SELECT AVG(COALESCE(remise, 0)) AS remise_moyenne_globale
FROM commande;   -- 50.00 / 5 = 10.00
```

On notera que `AVG()` renvoie une valeur **décimale même sur une colonne entière**, et qu'elle ajoute des décimales supplémentaires pour préserver la précision du quotient (d'où l'affichage `126.200000`).

---

## MIN() et MAX() — extrêmes

`MIN()` et `MAX()` renvoient respectivement la plus petite et la plus grande valeur. Elles ne se limitent pas aux nombres : elles s'appliquent aussi aux **chaînes** et aux **dates**.

```sql
SELECT
    MIN(montant)  AS montant_min,
    MAX(montant)  AS montant_max,
    MIN(date_cmd) AS premiere_commande,
    MAX(date_cmd) AS derniere_commande,
    MIN(client)   AS client_min,
    MAX(client)   AS client_max
FROM commande;
```

| montant_min | montant_max | premiere_commande | derniere_commande | client_min | client_max |
|-------------|-------------|-------------------|-------------------|------------|------------|
| 80.50 | 200.00 | 2026-01-05 | 2026-03-01 | Dupont | Martin |

- Sur des **nombres**, l'ordre est numérique : 80.50 et 200.00.
- Sur des **dates**, l'ordre est chronologique : `MIN` donne la commande la plus ancienne, `MAX` la plus récente.
- Sur des **chaînes**, l'ordre dépend de la **collation** de la colonne (voir [section 11.11 sur utf8mb4/UCA](../11-administration-configuration/11-charset-utf8mb4-uca14.md)). Avec une collation classique, *Dupont* précède *Martin* : `MIN(client) = 'Dupont'`, `MAX(client) = 'Martin'`.

Comme `SUM` et `AVG`, ces deux fonctions **ignorent les `NULL`**. Combiner `MIN` et `MAX` dans une expression permet par exemple d'obtenir une amplitude :

```sql
SELECT MAX(montant) - MIN(montant) AS amplitude
FROM commande;   -- 200.00 - 80.50 = 119.50
```

---

## Le traitement des valeurs NULL : à bien comprendre

La gestion des `NULL` est le point le plus souvent source d'erreurs avec les agrégats. Deux règles résument tout :

1. **`COUNT(*)` compte toutes les lignes** ; toutes les autres fonctions (y compris `COUNT(colonne)`) **ignorent les `NULL`**.
2. Sur un **ensemble vide** (aucune ligne ne correspond), `COUNT` renvoie `0`, mais `SUM`, `AVG`, `MIN` et `MAX` renvoient **`NULL`** — et non `0`.

| Fonction | Ignore les `NULL` ? | Résultat sur un ensemble vide |
|----------|---------------------|-------------------------------|
| `COUNT(*)` | Non — compte toutes les lignes | `0` |
| `COUNT(expr)` | Oui — compte les valeurs non `NULL` | `0` |
| `SUM(expr)` | Oui | `NULL` |
| `AVG(expr)` | Oui | `NULL` |
| `MIN(expr)` | Oui | `NULL` |
| `MAX(expr)` | Oui | `NULL` |

Le cas de l'ensemble vide se vérifie facilement avec un filtre qui ne ramène aucune ligne :

```sql
SELECT
    COUNT(*)     AS nb,
    SUM(montant) AS somme,
    AVG(montant) AS moyenne,
    MIN(montant) AS mini,
    MAX(montant) AS maxi
FROM commande
WHERE region = 'Ouest';   -- aucune commande dans cette région
```

| nb | somme | moyenne | mini | maxi |
|----|-------|---------|------|------|
| 0 | *NULL* | *NULL* | *NULL* | *NULL* |

Ce comportement explique une pratique courante en production : on enveloppe souvent un `SUM` ou un `AVG` dans un `COALESCE` lorsqu'on a besoin d'un `0` plutôt que d'un `NULL` dans l'application appelante :

```sql
SELECT COALESCE(SUM(montant), 0) AS ca
FROM commande
WHERE region = 'Ouest';   -- renvoie 0.00 au lieu de NULL
```

---

## Combiner agrégats et filtrage

Les fonctions d'agrégation se combinent librement dans un même `SELECT`, et s'appliquent toujours **après** le filtrage `WHERE`. Concrètement, `WHERE` sélectionne d'abord les lignes, puis l'agrégation s'effectue sur ce sous-ensemble.

```sql
SELECT
    COUNT(*)     AS nb_commandes_nord,
    SUM(montant) AS ca_nord
FROM commande
WHERE region = 'Nord';
```

| nb_commandes_nord | ca_nord |
|-------------------|---------|
| 2 | 320.00 |

En revanche, **on ne peut pas utiliser une fonction d'agrégation dans la clause `WHERE`** : celle-ci est évaluée ligne par ligne, avant que l'agrégat n'existe.

```sql
-- ❌ Incorrect : un agrégat ne peut pas figurer dans WHERE
SELECT region
FROM commande
WHERE SUM(montant) > 100;
```

Pour filtrer sur le **résultat d'un agrégat**, il faut la clause `HAVING`, qui intervient après le regroupement. C'est précisément l'objet de la [section 3.2](02-regroupement-donnees.md).

---

## De l'agrégat global au regroupement

Tant qu'aucune colonne non agrégée n'apparaît dans le `SELECT`, l'agrégat porte sur la table entière et produit une seule ligne — c'est le cas de tous les exemples précédents. Dès que l'on souhaite obtenir, par exemple, *un chiffre d'affaires par région*, on entre dans le domaine du regroupement avec `GROUP BY`.

À ce sujet, une particularité de MariaDB mérite d'être signalée : contrairement à MySQL, MariaDB **n'active pas le mode `ONLY_FULL_GROUP_BY` par défaut**. Mélanger une colonne non agrégée et un agrégat sans `GROUP BY` y est donc toléré, mais renvoie une valeur arbitraire pour la colonne non agrégée — un comportement à éviter. Le sujet est traité en détail dans la [section 3.2](02-regroupement-donnees.md) et, du côté de la configuration, dans la [section 11.3 sur `sql_mode`](../11-administration-configuration/03-modes-sql.md).

---

## Pour aller plus loin

Les cinq fonctions présentées ici sont les plus courantes, mais MariaDB en propose d'autres :

- **`GROUP_CONCAT()`** concatène les valeurs d'un groupe en une seule chaîne ; très utile avec `GROUP BY`, elle est abordée en [section 3.2](02-regroupement-donnees.md).
- **`STDDEV()`, `VARIANCE()`** et leurs variantes fournissent des statistiques de dispersion.
- Lorsqu'on a besoin d'un calcul agrégé **tout en conservant le détail des lignes** (cumuls, moyennes mobiles, classements), ce ne sont plus les fonctions d'agrégation classiques qu'il faut employer, mais les **window functions**, traitées en [section 4.2](../04-concepts-avances-sql/02-window-functions.md).

---

## À retenir

- Une fonction d'agrégation réduit un ensemble de lignes à une valeur unique.
- `COUNT(*)` compte les lignes ; `COUNT(colonne)` ne compte que les valeurs non `NULL` ; `COUNT(DISTINCT colonne)` compte les valeurs distinctes.
- `SUM`, `AVG`, `MIN` et `MAX` **ignorent les `NULL`**. C'est particulièrement sensible pour `AVG`, qui divise par le nombre de valeurs renseignées.
- Sur un ensemble vide, `COUNT` renvoie `0` tandis que les autres renvoient `NULL` ; d'où l'usage fréquent de `COALESCE`.
- `MIN` et `MAX` fonctionnent aussi sur les chaînes (selon la collation) et les dates.
- Un agrégat ne peut pas figurer dans `WHERE` : pour filtrer sur son résultat, on utilisera `HAVING` ([section 3.2](02-regroupement-donnees.md)).

---

## Navigation

- ⬅️ Section précédente : [3. Requêtes SQL Intermédiaires (introduction)](README.md)
- ➡️ Section suivante : [3.2 — Regroupement de données (GROUP BY, HAVING)](02-regroupement-donnees.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Regroupement de données (GROUP BY, HAVING)](/03-requetes-sql-intermediaires/02-regroupement-donnees.md)

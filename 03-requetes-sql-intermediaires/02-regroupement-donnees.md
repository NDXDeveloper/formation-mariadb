🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 Regroupement de données (GROUP BY, HAVING)

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.2  
> Version de référence : **MariaDB 12.3 LTS**

---

## Du calcul global au calcul par groupe

En [section 3.1](01-fonctions-agregation.md), les fonctions d'agrégation portaient sur la **table entière** et renvoyaient une unique ligne : un seul chiffre d'affaires, un seul panier moyen. Mais on a rarement besoin d'un total unique : on veut le chiffre d'affaires **par région**, le nombre de commandes **par client**, la vente maximale **par mois**.

C'est le rôle de la clause **`GROUP BY`** : elle découpe les lignes en **groupes** partageant la même valeur sur une ou plusieurs colonnes, puis applique les fonctions d'agrégation **à l'intérieur de chaque groupe**. La requête ne renvoie alors plus une ligne par enregistrement, mais **une ligne par groupe**.

La clause **`HAVING`**, elle, permet de **filtrer ces groupes** une fois constitués — par exemple ne garder que les régions dont le chiffre d'affaires dépasse un seuil.

---

## Jeu de données utilisé dans les exemples

Cette section reprend la table `commande` introduite en [section 3.1](01-fonctions-agregation.md) :

| id | client | region | montant | remise | date_cmd |
|----|--------|--------|---------|--------|----------|
| 1 | Dupont | Nord | 120.00 | 10.00 | 2026-01-05 |
| 2 | Martin | Sud | 80.50 | *NULL* | 2026-01-07 |
| 3 | Dupont | Nord | 200.00 | 25.00 | 2026-02-10 |
| 4 | Leroy | Est | 80.50 | *NULL* | 2026-02-15 |
| 5 | Martin | Sud | 150.00 | 15.00 | 2026-03-01 |

---

## GROUP BY : constituer des groupes

### Regroupement sur une colonne

Reprenons le chiffre d'affaires, mais cette fois **par région** :

```sql
SELECT
    region,
    COUNT(*)     AS nb,
    SUM(montant) AS ca,
    AVG(montant) AS panier_moyen
FROM commande
GROUP BY region;
```

| region | nb | ca | panier_moyen |
|--------|----|----|--------------|
| Est | 1 | 80.50 | 80.500000 |
| Nord | 2 | 320.00 | 160.000000 |
| Sud | 2 | 230.50 | 115.250000 |

Chaque ligne du résultat correspond à une valeur distincte de `region`, et les agrégats sont recalculés pour les seules lignes de ce groupe : la région *Nord* regroupe les commandes 1 et 3, dont les montants (120.00 + 200.00) donnent un chiffre d'affaires de 320.00.

### La règle SELECT / GROUP BY

Une règle structurante gouverne ce type de requête : **toute colonne du `SELECT` qui n'est pas encapsulée dans une fonction d'agrégation doit figurer dans le `GROUP BY`**. C'est logique : si une colonne n'est ni agrégée ni groupée, MariaDB ne saurait quelle valeur retenir pour un groupe qui en contient plusieurs.

Ici, une particularité de MariaDB doit être signalée : contrairement à MySQL, MariaDB **n'active pas le mode `ONLY_FULL_GROUP_BY` par défaut**. La requête suivante est donc *tolérée*, mais elle est **trompeuse** :

```sql
-- ⚠️ Toléré par défaut sous MariaDB, mais à éviter
SELECT region, client, SUM(montant) AS ca
FROM commande
GROUP BY region;
```

La colonne `client` n'est ni agrégée ni présente dans le `GROUP BY` : MariaDB renvoie pour chaque groupe une valeur **arbitraire et non déterministe** de `client`. Pour obtenir un comportement strict (et une erreur explicite dans ce cas), on peut activer le mode correspondant — voir la [section 11.3 sur `sql_mode`](../11-administration-configuration/03-modes-sql.md) :

```sql
SET sql_mode = CONCAT(@@sql_mode, ',ONLY_FULL_GROUP_BY');
```

La bonne pratique est de **toujours** lister dans le `GROUP BY` chaque colonne non agrégée du `SELECT`, quel que soit le `sql_mode`.

### Regroupement sur plusieurs colonnes

On peut grouper sur plusieurs colonnes : MariaDB forme alors un groupe par **combinaison distincte** de valeurs.

```sql
SELECT
    region,
    client,
    COUNT(*)     AS nb,
    SUM(montant) AS ca
FROM commande
GROUP BY region, client;
```

| region | client | nb | ca |
|--------|--------|----|----|
| Est | Leroy | 1 | 80.50 |
| Nord | Dupont | 2 | 320.00 |
| Sud | Martin | 2 | 230.50 |

### Regroupement sur une expression

Le `GROUP BY` ne se limite pas aux colonnes : il accepte des **expressions**. C'est fréquent pour agréger par période, en s'appuyant sur les [fonctions de dates](07-fonctions-dates-heures.md) :

```sql
SELECT
    YEAR(date_cmd)  AS annee,
    MONTH(date_cmd) AS mois,
    COUNT(*)        AS nb,
    SUM(montant)    AS ca
FROM commande
GROUP BY YEAR(date_cmd), MONTH(date_cmd)
ORDER BY annee, mois;
```

| annee | mois | nb | ca |
|-------|------|----|----|
| 2026 | 1 | 2 | 200.50 |
| 2026 | 2 | 2 | 280.50 |
| 2026 | 3 | 1 | 150.00 |

### Le cas des valeurs NULL

Dans un `GROUP BY`, **toutes les valeurs `NULL` d'une colonne sont rassemblées dans un seul et même groupe**. En regroupant sur `remise`, les deux commandes sans remise (2 et 4) ne sont pas écartées : elles forment ensemble un groupe `NULL`.

```sql
SELECT remise, COUNT(*) AS nb
FROM commande
GROUP BY remise;
```

| remise | nb |
|--------|----|
| *NULL* | 2 |
| 10.00 | 1 |
| 15.00 | 1 |
| 25.00 | 1 |

C'est une différence notable avec le comportement habituel de `NULL` (où `NULL = NULL` est indéterminé) : pour le regroupement, les `NULL` sont considérés comme « égaux entre eux ».

### Ordre des résultats

Point important : **`GROUP BY` ne garantit pas l'ordre des lignes renvoyées**. Si vous avez besoin d'un résultat trié, ajoutez une clause `ORDER BY` explicite — ne vous reposez jamais sur un tri implicite. Un index adapté sur les colonnes de regroupement peut par ailleurs permettre à MariaDB d'éviter un tri ou une table temporaire (voir la [section 5.5.3](../05-index-et-performance/05.3-index-order-group.md)).

---

## GROUP_CONCAT : agréger des valeurs en une chaîne

Au-delà des fonctions vues en 3.1, le regroupement introduit une fonction d'agrégation particulièrement utile : **`GROUP_CONCAT()`**. Plutôt que de produire un nombre, elle **concatène en une seule chaîne** les valeurs d'un groupe.

```sql
SELECT
    region,
    GROUP_CONCAT(DISTINCT client ORDER BY client SEPARATOR ', ') AS clients
FROM commande
GROUP BY region;
```

| region | clients |
|--------|---------|
| Est | Leroy |
| Nord | Dupont |
| Sud | Martin |

`GROUP_CONCAT` accepte trois clauses internes utiles : `DISTINCT` (pour éliminer les doublons — ici, *Dupont* n'apparaît qu'une fois bien qu'il ait passé deux commandes dans le Nord), `ORDER BY` (pour ordonner les éléments concaténés) et `SEPARATOR` (par défaut une virgule sans espace).

> ⚠️ **Limite de longueur.** Le résultat est tronqué silencieusement au-delà de `group_concat_max_len` octets (1024 par défaut), avec un simple avertissement. Sur des groupes volumineux, pensez à augmenter cette variable de session :
> ```sql
> SET SESSION group_concat_max_len = 1000000;
> ```

---

## WITH ROLLUP : sous-totaux et total général

L'extension **`WITH ROLLUP`** ajoute aux résultats des lignes de **super-agrégats** : des sous-totaux par niveau de regroupement et un total général.

```sql
SELECT
    region,
    SUM(montant) AS ca
FROM commande
GROUP BY region WITH ROLLUP;
```

| region | ca |
|--------|----|
| Est | 80.50 |
| Nord | 320.00 |
| Sud | 230.50 |
| *NULL* | 631.00 |

La dernière ligne, où `region` vaut `NULL`, représente le **total général** (631.00 = somme de toutes les régions). Avec plusieurs colonnes de regroupement, `WITH ROLLUP` produirait des sous-totaux intermédiaires à chaque niveau, le `NULL` marquant à chaque fois la dimension « agrégée ».

Pour l'affichage, on remplace généralement ce `NULL` de super-agrégat par un libellé explicite avec [`COALESCE`](08-expressions-conditionnelles.md) :

```sql
SELECT
    COALESCE(region, 'TOTAL') AS region,
    SUM(montant)              AS ca
FROM commande
GROUP BY region WITH ROLLUP;
```

> ⚠️ **Limite de ce libellé.** `COALESCE(region, 'TOTAL')` suppose que la colonne groupée ne contient **pas de `NULL` réel** : sinon, un groupe à `NULL` authentique serait lui aussi affiché « TOTAL », indistinguible du super-agrégat. Le SQL standard et MySQL lèvent cette ambiguïté avec la fonction `GROUPING()`, **que MariaDB ne propose pas** ; sous MariaDB, on réserve donc ce libellé aux colonnes de regroupement non *nullables*.

---

## HAVING : filtrer les groupes

### Principe

`HAVING` filtre les **groupes** produits par `GROUP BY`, à la manière dont `WHERE` filtre les lignes. Sa particularité est de pouvoir s'appuyer sur le **résultat d'une fonction d'agrégation** — ce que `WHERE` ne peut pas faire (voir [section 3.1](01-fonctions-agregation.md)).

```sql
SELECT
    region,
    SUM(montant) AS ca
FROM commande
GROUP BY region
HAVING SUM(montant) > 200;
```

| region | ca |
|--------|----|
| Nord | 320.00 |
| Sud | 230.50 |

La région *Est* (80.50) est écartée car son chiffre d'affaires ne dépasse pas 200. Le filtrage porte ici sur une valeur — `SUM(montant)` — qui n'existe qu'**après** le regroupement.

### HAVING peut référencer un alias du SELECT

MariaDB autorise `HAVING` à réutiliser un **alias** défini dans le `SELECT`, ce qui allège l'écriture (c'est une extension par rapport au SQL standard, et une chose que `WHERE` ne permet pas) :

```sql
SELECT
    region,
    SUM(montant) AS ca
FROM commande
GROUP BY region
HAVING ca > 200;     -- réutilise l'alias « ca »
```

### WHERE et HAVING : ne pas les confondre

C'est le point le plus important de cette section. `WHERE` et `HAVING` filtrent à deux moments différents du traitement :

| | `WHERE` | `HAVING` |
|---|---------|----------|
| **Agit sur** | les lignes individuelles | les groupes |
| **S'exécute** | *avant* le regroupement | *après* le regroupement |
| **Peut utiliser un agrégat ?** | Non | Oui |
| **Peut utiliser un alias du SELECT ?** | Non | Oui (sous MariaDB) |
| **Bénéficie des index ?** | Oui (typiquement) | Non directement |

Les deux clauses se combinent dans une même requête, chacune jouant son rôle :

```sql
SELECT
    region,
    COUNT(*)     AS nb,
    SUM(montant) AS ca
FROM commande
WHERE montant >= 100     -- filtre les LIGNES avant regroupement
GROUP BY region
HAVING COUNT(*) >= 2;    -- filtre les GROUPES après regroupement
```

| region | nb | ca |
|--------|----|----|
| Nord | 2 | 320.00 |

Décortiquons : le `WHERE` ne conserve que les commandes dont le montant atteint 100 (commandes 1, 3 et 5). Après regroupement par région, le *Nord* compte 2 commandes et le *Sud* une seule. Le `HAVING COUNT(*) >= 2` ne retient alors que le *Nord*.

---

## L'ordre logique d'évaluation des clauses

Pour bien saisir pourquoi `WHERE` ne « voit » pas les agrégats alors que `HAVING` le peut, il faut connaître l'**ordre logique** dans lequel MariaDB traite une requête — qui diffère de l'ordre d'écriture :

1. **`FROM`** (et les jointures, voir [section 3.3](03-jointures.md)) — détermine les lignes source ;
2. **`WHERE`** — filtre ces lignes ;
3. **`GROUP BY`** — constitue les groupes ;
4. **`HAVING`** — filtre les groupes ;
5. **`SELECT`** — évalue les colonnes, les agrégats et les alias ;
6. **`ORDER BY`** — trie le résultat ;
7. **`LIMIT`** — restreint le nombre de lignes finales.

Cet ordre explique tout :

- `WHERE` s'exécute à l'étape 2, **avant** la formation des groupes (étape 3) : les agrégats n'existent pas encore, d'où l'impossibilité de les y utiliser.
- `HAVING` s'exécute à l'étape 4, **après** les groupes : les agrégats sont disponibles.
- Les alias du `SELECT` sont calculés à l'étape 5 ; si MariaDB tolère leur usage en `HAVING` (étape 4) et en `ORDER BY` (étape 6), c'est au titre d'une extension de confort, pas du modèle standard.

---

## Performance et bonnes pratiques

Quelques principes découlent directement de ce qui précède :

- **Filtrez le plus tôt possible.** Une condition qui ne porte pas sur un agrégat doit aller dans `WHERE`, jamais dans `HAVING` : `WHERE` réduit le volume de lignes *avant* le regroupement et peut exploiter un index, tandis qu'un `HAVING` équivalent ferait le tri après coup, plus coûteux.
- **Réservez `HAVING` aux conditions sur agrégats** (`SUM`, `COUNT`, etc.), qui ne peuvent pas s'exprimer autrement.
- **Indexez les colonnes de regroupement** lorsque c'est pertinent : un index adapté peut éviter à MariaDB de construire une table temporaire et de trier (voir [section 5.5.3](../05-index-et-performance/05.3-index-order-group.md)).

---

## À retenir

- `GROUP BY` découpe les lignes en groupes et applique les agrégats par groupe ; la requête renvoie une ligne par groupe.
- Toute colonne non agrégée du `SELECT` doit figurer dans le `GROUP BY` ; MariaDB le tolère par défaut (pas de `ONLY_FULL_GROUP_BY`), mais la valeur renvoyée est alors arbitraire — à éviter.
- Les valeurs `NULL` sont regroupées ensemble dans un unique groupe.
- `GROUP BY` ne garantit aucun ordre : ajoutez `ORDER BY` si besoin.
- `GROUP_CONCAT` concatène les valeurs d'un groupe (clauses `DISTINCT`, `ORDER BY`, `SEPARATOR` ; attention à `group_concat_max_len`).
- `WITH ROLLUP` ajoute sous-totaux et total général, repérables par un `NULL` dans les colonnes groupées.
- `WHERE` filtre les lignes *avant* regroupement (pas d'agrégat) ; `HAVING` filtre les groupes *après* (agrégats autorisés).
- L'ordre logique `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT` explique ces différences.

---

## Navigation

- ⬅️ Section précédente : [3.1 — Fonctions d'agrégation](01-fonctions-agregation.md)
- ➡️ Section suivante : [3.3 — Jointures](03-jointures.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Jointures](/03-requetes-sql-intermediaires/03-jointures.md)

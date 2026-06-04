🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6 Gestion des valeurs `NULL` : Logique ternaire

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Référence : **MariaDB 12.3 LTS**

`NULL` n'est pas une valeur comme les autres : il représente une **absence** ou une donnée **inconnue**, et non un zéro ou une chaîne vide. Cette nuance, en apparence anodine, gouverne une famille entière de comportements surprenants qui ressurgissent tout au long de ce chapitre — du piège de `NOT IN` (§ 4.5) aux résultats inattendus d'un filtre ou d'une moyenne. La comprendre une fois pour toutes évite la plupart de ces écueils. Tout découle d'un principe : SQL ne raisonne pas en logique binaire, mais en **logique ternaire**.

## 🎯 Objectif de la section

Comprendre ce que `NULL` signifie réellement, maîtriser la logique à trois valeurs et ses conséquences sur les filtres, les calculs et les agrégations, et connaître les fonctions et opérateurs dédiés à la gestion des `NULL`.

## `NULL` n'est pas zéro

Avant tout : `NULL` n'est ni `0`, ni `''`, ni `FALSE`. C'est la marque d'une **valeur absente ou indéterminée**. « Je ne connais pas l'âge de cette personne » (`NULL`) diffère de « cette personne a 0 an ». De cette distinction découlent tous les comportements qui suivent.

## La logique ternaire : `TRUE`, `FALSE`, `UNKNOWN`

En SQL, une condition ne s'évalue pas seulement à *vrai* ou *faux*, mais à l'une de **trois** valeurs : `TRUE`, `FALSE` ou `UNKNOWN`. Cette troisième valeur apparaît dès qu'un `NULL` entre dans une comparaison.

En effet, **toute comparaison impliquant `NULL` renvoie `UNKNOWN`**, jamais `TRUE` ni `FALSE` :

```sql
NULL = 5      -- UNKNOWN
NULL <> 5     -- UNKNOWN
NULL > 5      -- UNKNOWN
NULL = NULL   -- UNKNOWN  (et non TRUE !)
```

Le dernier cas est contre-intuitif mais logique : deux valeurs inconnues ne sont pas *connues* comme égales. La présence d'`UNKNOWN` modifie le comportement des opérateurs booléens. Voici leurs tables de vérité :

**`AND`**

| `AND`     | TRUE    | FALSE | UNKNOWN |
|-----------|---------|-------|---------|
| **TRUE**  | TRUE    | FALSE | UNKNOWN |
| **FALSE** | FALSE   | FALSE | FALSE   |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

**`OR`**

| `OR`      | TRUE | FALSE   | UNKNOWN |
|-----------|------|---------|---------|
| **TRUE**  | TRUE | TRUE    | TRUE    |
| **FALSE** | TRUE | FALSE   | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

**`NOT`** : `NOT TRUE` = FALSE, `NOT FALSE` = TRUE, et `NOT UNKNOWN` = **UNKNOWN**.

Deux règles pratiques s'en dégagent : `FALSE AND` quoi que ce soit vaut `FALSE`, et `TRUE OR` quoi que ce soit vaut `TRUE` (l'autre opérande, même inconnu, n'y change rien). Dans les autres cas, l'`UNKNOWN` se propage.

## Tester un `NULL` : `IS NULL` / `IS NOT NULL`

Puisque `x = NULL` renvoie toujours `UNKNOWN`, ce test ne sélectionne **jamais** rien. Pour détecter un `NULL`, on emploie l'opérateur dédié :

```sql
WHERE telephone IS NULL        -- lignes sans téléphone
WHERE telephone IS NOT NULL    -- lignes avec téléphone
```

MariaDB offre par ailleurs un opérateur d'**égalité tolérante au `NULL`**, `<=>`, qui renvoie `TRUE` même lorsque les deux côtés sont `NULL` :

```sql
NULL <=> NULL   -- TRUE (1)
NULL <=> 5      -- FALSE (0)
5    <=> 5      -- TRUE (1)
```

Il est précieux pour comparer deux colonnes potentiellement nulles en considérant « `NULL` égale `NULL` ».

## `NULL` dans le `WHERE` : seules les lignes `TRUE` passent

Point capital : la clause `WHERE` ne conserve que les lignes pour lesquelles la condition vaut **`TRUE`**. Une condition `UNKNOWN` (comme `FALSE`) **exclut** la ligne. C'est la source de deux pièges fréquents.

D'abord, une exclusion par `<>` écarte silencieusement les `NULL`. La requête suivante :

```sql
WHERE statut <> 'actif'
```

ne renvoie **pas** les lignes dont `statut` est `NULL`, car `NULL <> 'actif'` vaut `UNKNOWN`. Pour les inclure, il faut l'expliciter :

```sql
WHERE statut <> 'actif' OR statut IS NULL
```

Ensuite, le piège de **`NOT IN` avec un `NULL`** (déjà signalé au § 4.5). L'expression `x NOT IN (a, b, NULL)` équivaut à `x <> a AND x <> b AND x <> NULL` ; le dernier terme valant toujours `UNKNOWN`, la conjonction ne peut jamais être `TRUE` et **aucune ligne ne ressort**. À l'inverse, `IN` n'est pas affecté (un `OR` suffit à donner `TRUE`). La parade est d'utiliser `NOT EXISTS`.

## `NULL` dans les calculs

En arithmétique, **toute opération impliquant `NULL` donne `NULL`** — y compris la multiplication par zéro :

```sql
5 + NULL      -- NULL
NULL * 0      -- NULL  (et non 0 !)
```

Il en va de même pour la concaténation : `CONCAT('a', NULL, 'b')` renvoie `NULL`. La fonction `CONCAT_WS`, elle, **ignore** les `NULL` (mais renvoie `NULL` si c'est le séparateur qui est nul).

## `NULL` dans les agrégations

Les fonctions d'agrégation **ignorent les `NULL`**, à une exception près : `COUNT(*)`.

- `COUNT(*)` compte **toutes** les lignes ; `COUNT(colonne)` ne compte que les valeurs **non nulles** de cette colonne.
- `SUM`, `AVG`, `MIN`, `MAX` ignorent les `NULL`. En particulier, `AVG` divise la somme des valeurs présentes par leur **nombre** — les `NULL` ne comptent donc **pas** comme des zéros, ce qui change le résultat.
- Un `SUM` ne portant que sur des `NULL` renvoie `NULL`, et non `0`.

Cette asymétrie (les `NULL` disparaissent des agrégats mais pas du dénombrement total) est à garder à l'esprit lorsqu'on rapproche un `COUNT(*)` d'un `COUNT(colonne)`.

## `NULL` dans `GROUP BY`, `DISTINCT`, `ORDER BY` et `UNIQUE`

Dans ces contextes, le traitement des `NULL` suit des règles propres, parfois à rebours de la sémantique de comparaison :

- **`GROUP BY` et `DISTINCT`** considèrent tous les `NULL` comme **équivalents** : ils sont regroupés en une seule catégorie (alors même que `NULL = NULL` vaut `UNKNOWN`). C'est un cas particulier assumé.
- **`ORDER BY`** range les `NULL` ensemble ; par défaut, ils apparaissent en tête d'un tri ascendant (traités comme les plus petites valeurs).
- Une contrainte **`UNIQUE`** autorise **plusieurs `NULL`** dans la colonne, puisque deux `NULL` ne sont pas considérés comme « égaux » au sens de l'unicité.

## Les fonctions de gestion des `NULL`

MariaDB fournit plusieurs fonctions pour remplacer ou neutraliser les `NULL` :

- **`COALESCE(a, b, c, …)`** renvoie le **premier argument non nul**. Idéal pour une valeur de repli en cascade :

```sql
SELECT COALESCE(tel_mobile, tel_fixe, 'Aucun') AS contact FROM clients;
```

- **`IFNULL(a, b)`** est le raccourci à deux arguments : `b` si `a` est `NULL`, sinon `a` (par exemple `IFNULL(remise, 0)` pour traiter une remise absente comme nulle).
- **`NULLIF(a, b)`** renvoie `NULL` si `a = b`, sinon `a`. Très utile pour transformer une valeur sentinelle en `NULL`, ou pour **prévenir une division par zéro** de façon explicite et indépendante du mode SQL :

```sql
SELECT montant / NULLIF(quantite, 0) AS prix_unitaire FROM lignes;
```

- **`ISNULL(x)`** renvoie `1` si `x` est `NULL`, `0` sinon.

## Récapitulatif des pièges courants

À retenir pour ne plus s'y faire prendre :

- `col = NULL` ne fonctionne pas → employer `col IS NULL` ;
- `col <> 'valeur'` exclut les lignes `NULL` → ajouter `OR col IS NULL` ;
- `NOT IN (… , NULL)` ne renvoie rien → préférer `NOT EXISTS` ;
- `NULL` en arithmétique ou en `CONCAT` propage `NULL` ;
- les agrégats ignorent les `NULL`, et `AVG` ne les compte pas comme des zéros.

## Points clés à retenir

`NULL` signifie **inconnu/absent**, pas zéro. SQL évalue les conditions en **logique ternaire** (`TRUE`, `FALSE`, `UNKNOWN`), et toute comparaison avec `NULL` — y compris `NULL = NULL` — vaut `UNKNOWN`. On teste donc un `NULL` avec `IS NULL`/`IS NOT NULL` (ou l'opérateur tolérant `<=>`), jamais avec `=`. Comme le `WHERE` ne garde que les lignes `TRUE`, les comparaisons inversées (`<>`, `NOT IN`) écartent silencieusement les `NULL`. En calcul, `NULL` se propage ; en agrégation, il est ignoré (sauf par `COUNT(*)`) et n'est pas assimilé à zéro. Dans `GROUP BY`/`DISTINCT`/`ORDER BY`, les `NULL` sont regroupés, et une contrainte `UNIQUE` en tolère plusieurs. Enfin, `COALESCE`, `IFNULL`, `NULLIF` et `ISNULL` permettent de remplacer ou neutraliser les `NULL` proprement.

---

**Section précédente :** [4.5 — Requêtes complexes multi-tables](05-requetes-complexes-multi-tables.md)  
**Section suivante :** [4.7 — JSON dans MariaDB](07-json-mariadb.md)  

⏭️ [JSON dans MariaDB](/04-concepts-avances-sql/07-json-mariadb.md)

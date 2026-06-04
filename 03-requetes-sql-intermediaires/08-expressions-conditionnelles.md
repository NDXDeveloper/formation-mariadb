🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.8 Expressions conditionnelles (CASE, IF, IFNULL, COALESCE, NULLIF)

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.8  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Les **expressions conditionnelles** introduisent de la logique « si… alors… sinon… » directement dans une requête : classer une valeur dans une catégorie, produire un libellé selon un seuil, remplacer une valeur manquante par une valeur par défaut. Ce sont des **expressions** (et non des instructions) : elles renvoient une valeur et s'emploient partout où une valeur est attendue — dans le `SELECT`, le `WHERE`, l'`ORDER BY`, le `GROUP BY` ou à l'intérieur d'une fonction d'agrégation.

`CASE` est la construction de référence, dont dérivent toutes les autres ; `IF`, `IFNULL`, `COALESCE` et `NULLIF` en sont des raccourcis spécialisés, plusieurs dédiés à la gestion des valeurs `NULL`.

---

## Rappel du schéma

Les exemples reposent sur les tables de la [section 3.3](03-jointures.md) :

**`client`** — Dupont (1, Lille), Martin (2, Marseille), Leroy (3, Strasbourg), **Petit (4, Lyon, sans commande)**  
**`commande`** — cmd 1→client 1 (120.00), 2→client 2 (80.50), 3→client 1 (200.00), 4→client 3 (80.50), 5→client 2 (150.00)  

---

## CASE : la construction centrale

`CASE` existe sous deux formes.

### CASE recherché

La forme **recherchée** évalue une suite de **conditions booléennes** et renvoie le résultat de la première qui est vraie. C'est la plus générale.

```sql
SELECT
    id,
    montant,
    CASE
        WHEN montant < 100 THEN 'Petit'
        WHEN montant < 180 THEN 'Moyen'
        ELSE 'Gros'
    END AS categorie
FROM commande;
```

| id | montant | categorie |
|----|---------|-----------|
| 1 | 120.00 | Moyen |
| 2 | 80.50 | Petit |
| 3 | 200.00 | Gros |
| 4 | 80.50 | Petit |
| 5 | 150.00 | Moyen |

L'**ordre des `WHEN` compte** : ils sont évalués de haut en bas et le **premier vrai l'emporte**. C'est ce qui permet d'exprimer des tranches (`< 100`, puis `< 180`, puis le reste). La clause `ELSE` est facultative ; **en son absence, une ligne sans condition vraie renvoie `NULL`**.

### CASE simple

La forme **simple** compare une expression à une liste de valeurs (par **égalité**) :

```sql
SELECT
    nom,
    ville,
    CASE ville
        WHEN 'Lille'      THEN 'Nord'
        WHEN 'Marseille'  THEN 'Sud'
        WHEN 'Strasbourg' THEN 'Est'
        ELSE 'Autre'
    END AS zone
FROM client;
```

| nom | ville | zone |
|-----|-------|------|
| Dupont | Lille | Nord |
| Martin | Marseille | Sud |
| Leroy | Strasbourg | Est |
| Petit | Lyon | Autre |

Lyon ne correspondant à aucun `WHEN`, la branche `ELSE` s'applique.

> ⚠️ La forme simple utilise l'**égalité** (`=`) : elle ne peut donc **pas** capturer un `NULL` via `WHEN NULL` (car `valeur = NULL` est toujours indéterminé, voir [section 4.6](../04-concepts-avances-sql/06-gestion-valeurs-null.md)). Pour tester un `NULL`, utilisez la forme recherchée avec `WHEN ville IS NULL`.

### Agrégation conditionnelle

Combiner `CASE` et une fonction d'agrégation est l'un des motifs les plus puissants du SQL : il permet de **ventiler des compteurs ou des sommes par catégorie sur une seule ligne** — un pivot manuel.

```sql
SELECT
    COUNT(CASE WHEN montant <  100 THEN 1 END)            AS nb_petits,
    COUNT(CASE WHEN montant >= 100 THEN 1 END)            AS nb_autres,
    SUM(CASE WHEN montant >= 100 THEN montant ELSE 0 END) AS ca_gros
FROM commande;
```

| nb_petits | nb_autres | ca_gros |
|-----------|-----------|---------|
| 2 | 3 | 470.00 |

L'astuce repose sur le comportement de `COUNT` vu en [section 3.1](01-fonctions-agregation.md) : `CASE WHEN … THEN 1 END` (sans `ELSE`) renvoie `NULL` pour les lignes non concernées, et `COUNT` **ignore les `NULL`** — il ne compte donc que les lignes correspondant à la condition. Ce mécanisme est la base des **requêtes pivotées**, approfondies en [section 4.3](../04-concepts-avances-sql/03-requetes-pivotees.md).

---

## IF : le conditionnel à deux branches

`IF(condition, valeur_si_vrai, valeur_si_faux)` est un raccourci pour un choix binaire :

```sql
SELECT
    id,
    montant,
    IF(montant >= 126.20, 'Au-dessus moyenne', 'En-dessous') AS position
FROM commande;
```

Pour les commandes 3 (200) et 5 (150), `position` vaut `Au-dessus moyenne` ; pour les autres, `En-dessous`.

Deux précisions :

- `IF` est une **fonction propre à MariaDB/MySQL** (non standard). Son équivalent portable est `CASE WHEN condition THEN a ELSE b END`.
- Ne pas confondre cette **fonction `IF`** (une expression) avec l'**instruction `IF … THEN … END IF`** des procédures stockées, qui relève du contrôle de flux ([section 8.7](../08-programmation-cote-serveur/07-variables-flow-control.md)).

---

## Gérer les valeurs NULL

Trois fonctions sont spécialisées dans le traitement des `NULL` — un thème déjà rencontré dans les sections précédentes (notamment avec les jointures externes en [3.3.2](03.2-left-right-join.md)).

### IFNULL

`IFNULL(expr, remplacement)` renvoie `remplacement` si `expr` est `NULL`, sinon `expr`. C'est une substitution à **deux arguments**.

### COALESCE

`COALESCE(a, b, c, …)` renvoie le **premier argument non `NULL`**. Plus général qu'`IFNULL` (nombre d'arguments illimité) et **standard SQL**, c'est la fonction à privilégier. Elle est idéale pour une **chaîne de repli** :

```sql
SELECT COALESCE(NULL, NULL, 'défaut') AS valeur;   -- 'défaut'
```

Appliquée à une jointure externe, elle remplace les `NULL` issus des lignes non appariées (ici, Petit n'a aucune commande) :

```sql
SELECT
    c.nom,
    COALESCE(SUM(cmd.montant), 0) AS total,
    IFNULL(MAX(cmd.montant), 0)   AS plus_grosse
FROM client c
LEFT JOIN commande cmd ON c.id = cmd.client_id
GROUP BY c.id, c.nom;
```

| nom | total | plus_grosse |
|-----|-------|-------------|
| Dupont | 320.00 | 200.00 |
| Martin | 230.50 | 150.00 |
| Leroy | 80.50 | 80.50 |
| Petit | 0.00 | 0.00 |

`SUM` et `MAX` renvoyant `NULL` sur un ensemble vide (rappel de la [section 3.1](01-fonctions-agregation.md)), `COALESCE`/`IFNULL` ramènent Petit à `0`.

### NULLIF

`NULLIF(a, b)` est l'opération inverse : elle renvoie `NULL` **si `a = b`**, sinon `a`. Son usage le plus courant est de **neutraliser une division par zéro** :

```sql
SELECT 10 / NULLIF(0, 0) AS resultat;   -- NULL au lieu d'une division par zéro
```

Si le diviseur vaut `0`, `NULLIF(diviseur, 0)` renvoie `NULL`, et la division par `NULL` donne `NULL` — proprement et explicitement, sans l'avertissement que produirait sinon une division par zéro (comportement lié à `ERROR_FOR_DIVISION_BY_ZERO`, voir [section 11.3](../11-administration-configuration/03-modes-sql.md)). `NULLIF` sert aussi à convertir une valeur sentinelle en `NULL`, par exemple `NULLIF(commentaire, '')` pour traiter une chaîne vide comme une absence.

---

## Tout se ramène à CASE

`IF`, `IFNULL`, `COALESCE` et `NULLIF` sont des raccourcis que l'on peut toujours réécrire avec `CASE` :

| Expression | Équivalent `CASE` |
|------------|-------------------|
| `IF(c, a, b)` | `CASE WHEN c THEN a ELSE b END` |
| `IFNULL(a, b)` | `CASE WHEN a IS NULL THEN b ELSE a END` |
| `COALESCE(a, b)` | `CASE WHEN a IS NOT NULL THEN a ELSE b END` |
| `NULLIF(a, b)` | `CASE WHEN a = b THEN NULL ELSE a END` |

`IFNULL(a, b)` et `COALESCE(a, b)` sont d'ailleurs équivalents à deux arguments ; on retient donc surtout **`COALESCE`** (standard, extensible) et **`CASE`** (le plus général).

---

## Note sur les types de retour

Une expression `CASE` renvoie un **type unique**, déterminé par MariaDB à partir des branches `THEN`/`ELSE`. Si celles-ci mélangent des types hétérogènes (un nombre et une chaîne, par exemple), MariaDB applique une conversion vers un type commun. Veillez donc à la cohérence des valeurs renvoyées par les différentes branches pour éviter des conversions inattendues.

---

## À retenir

- `CASE` est la construction conditionnelle de référence : forme **recherchée** (conditions booléennes) ou **simple** (égalité) ; `ELSE` facultatif (sinon `NULL`) ; **premier `WHEN` vrai retenu**.
- La forme **simple** ne capture pas les `NULL` (`=`) ; utiliser la forme recherchée avec `IS NULL`.
- L'**agrégation conditionnelle** (`COUNT`/`SUM` autour d'un `CASE`) ventile des résultats par catégorie sur une ligne — base des pivots ([section 4.3](../04-concepts-avances-sql/03-requetes-pivotees.md)).
- `IF(c, a, b)` est un choix binaire propre à MariaDB ; ne pas le confondre avec l'instruction `IF` des procédures ([section 8.7](../08-programmation-cote-serveur/07-variables-flow-control.md)).
- Gestion des `NULL` : `IFNULL` (2 args), **`COALESCE`** (n args, standard, à privilégier), `NULLIF` (renvoie `NULL` si `a = b`, utile contre la division par zéro).
- Toutes ces fonctions se réécrivent en `CASE`.

---

## Fin du chapitre 3

Cette section clôt le chapitre **3 — Requêtes SQL Intermédiaires**. Vous maîtrisez désormais l'agrégation, le regroupement, les jointures, les sous-requêtes, les opérateurs ensemblistes et les fonctions de manipulation (chaînes, dates, conditions). Le [chapitre 4](../04-concepts-avances-sql/README.md) aborde les **concepts avancés** : requêtes récursives, window functions, CTE, JSON et expressions régulières.

---

## Navigation

- ⬅️ Section précédente : [3.7.1 — Fonctions de compatibilité Oracle](07.1-fonctions-oracle.md)
- ➡️ Chapitre suivant : [4. Concepts Avancés SQL](../04-concepts-avances-sql/README.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Concepts Avancés SQL](/04-concepts-avances-sql/README.md)

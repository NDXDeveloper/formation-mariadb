🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 Sous-requêtes et requêtes imbriquées

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.4  
> Version de référence : **MariaDB 12.3 LTS**

---

## Définition

Une **sous-requête** (ou *requête imbriquée*) est une instruction `SELECT` placée **à l'intérieur** d'une autre instruction SQL. La requête extérieure (la *requête englobante*) exploite le résultat de la requête intérieure pour produire le sien.

Les sous-requêtes permettent de décomposer un problème en étapes : « trouver d'abord la moyenne, puis les lignes au-dessus » ; « identifier d'abord les clients ayant commandé, puis les lister ». Beaucoup de questions exprimables par une sous-requête le sont aussi par une jointure ([section 3.3](03-jointures.md)) — nous reviendrons sur ce choix en fin de section.

Deux grilles de lecture structurent tout ce qui suit :

1. **Ce que la sous-requête renvoie** : une valeur unique, une ligne, ou un ensemble de lignes ?
2. **Sa dépendance à la requête englobante** : est-elle autonome (*non corrélée*) ou se réfère-t-elle aux lignes de la requête extérieure (*corrélée*) ?

---

## Rappel du schéma

Les exemples reposent sur les tables de la [section 3.3](03-jointures.md) :

**`client`** — Dupont (1), Martin (2), Leroy (3), **Petit (4, sans commande)**  
**`commande`** — cmd 1→client 1 (120.00), 2→client 2 (80.50), 3→client 1 (200.00), 4→client 3 (80.50), 5→client 2 (150.00)  

---

## Les trois familles de sous-requêtes

| Famille | Renvoie | S'emploie avec |
|---------|---------|----------------|
| **Scalaire** | une seule valeur (1 ligne, 1 colonne) | `=`, `<`, `>`, … ; dans `SELECT`, `WHERE`, `HAVING` |
| **De ligne** | une seule ligne (plusieurs colonnes) | comparaison de n-uplets `(a, b) = (…)` |
| **De table** | plusieurs lignes | `IN`, `EXISTS`, `ANY`, `ALL` ; dans `FROM` (table dérivée) |

Quant à leur **emplacement**, une sous-requête peut apparaître dans le `WHERE` (le cas le plus fréquent), dans la liste du `SELECT`, dans le `FROM` (table dérivée), ou dans le `HAVING`.

---

## Sous-requête scalaire

Une sous-requête **scalaire** renvoie une valeur unique et s'utilise partout où une valeur est attendue. Exemple : les commandes dont le montant dépasse la **moyenne générale**.

```sql
SELECT id, montant
FROM commande
WHERE montant > (SELECT AVG(montant) FROM commande);
```

| id | montant |
|----|---------|
| 3 | 200.00 |
| 5 | 150.00 |

La sous-requête `(SELECT AVG(montant) FROM commande)` renvoie `126.20` (la moyenne des cinq montants, voir [section 3.1](01-fonctions-agregation.md)). Elle est **autonome** : elle ne dépend en rien de la requête englobante et n'est évaluée **qu'une seule fois**. On parle de sous-requête *non corrélée*.

> ⚠️ Une sous-requête utilisée avec un opérateur de comparaison (`=`, `>`, …) **doit renvoyer au plus une valeur**. Si elle en renvoie plusieurs, MariaDB lève une erreur (« Subquery returns more than 1 row »).

---

## IN et NOT IN

L'opérateur **`IN`** teste l'appartenance à l'ensemble de valeurs renvoyé par la sous-requête. Exemple : les clients ayant passé **au moins une commande**.

```sql
SELECT nom
FROM client
WHERE id IN (SELECT client_id FROM commande);
```

| nom |
|-----|
| Dupont |
| Martin |
| Leroy |

La sous-requête renvoie les `client_id` présents dans `commande` (`{1, 2, 3}`) ; Petit (id 4) n'y figurant pas, il est exclu.

### Le piège de NOT IN avec les NULL

`NOT IN` semble être la négation naturelle de `IN` — *les clients sans aucune commande* :

```sql
SELECT nom
FROM client
WHERE id NOT IN (SELECT client_id FROM commande);   -- renvoie : Petit
```

Ici le résultat est correct (Petit), **mais cette requête est fragile**. Si la sous-requête renvoyait ne serait-ce qu'**une seule valeur `NULL`**, `NOT IN` ne renverrait **aucune ligne**. La raison tient à la logique ternaire (voir [section 4.6](../04-concepts-avances-sql/06-gestion-valeurs-null.md)) : `x NOT IN (1, 2, NULL)` équivaut à `x <> 1 AND x <> 2 AND x <> NULL`, et `x <> NULL` vaut toujours *inconnu* — l'expression ne peut donc jamais être vraie.

Dans notre schéma, `commande.client_id` est protégé par une clé étrangère et ne contient pas de `NULL`. Mais sur une colonne *nullable*, `NOT IN` est une source de bugs silencieux. Deux parades : filtrer les `NULL` dans la sous-requête (`WHERE client_id IS NOT NULL`), ou — bien préférable — employer **`NOT EXISTS`** (ci-dessous), qui n'a pas ce défaut.

---

## EXISTS et NOT EXISTS

**`EXISTS`** ne s'intéresse pas aux *valeurs* renvoyées par la sous-requête mais à leur **existence** : il vaut vrai dès que la sous-requête renvoie au moins une ligne. Il s'emploie presque toujours en **corrélation** avec la requête englobante. Exemple : les clients ayant au moins une commande **supérieure à 150**.

```sql
SELECT c.nom
FROM client c
WHERE EXISTS (
    SELECT 1
    FROM commande cmd
    WHERE cmd.client_id = c.id
      AND cmd.montant > 150
);
```

| nom |
|-----|
| Dupont |

La sous-requête référence `c.id` — une colonne de la requête **extérieure** : elle est donc **corrélée**, évaluée conceptuellement pour chaque client. Seule la commande 3 (200, client 1) dépasse 150 : seul Dupont satisfait l'`EXISTS`.

Comme seule l'existence compte, la liste du `SELECT` interne est indifférente : on écrit par convention `SELECT 1`. MariaDB s'arrête dès la première ligne trouvée.

### NOT EXISTS : l'anti-jointure robuste

`NOT EXISTS` est la manière sûre d'exprimer « sans aucune correspondance » — sans le piège des `NULL` de `NOT IN` :

```sql
SELECT c.nom
FROM client c
WHERE NOT EXISTS (
    SELECT 1 FROM commande cmd WHERE cmd.client_id = c.id
);   -- renvoie : Petit
```

C'est l'équivalent, en sous-requête, du motif `LEFT JOIN ... IS NULL` vu en [section 3.3.2](03.2-left-right-join.md). **Privilégiez `NOT EXISTS` à `NOT IN`** dès qu'une valeur `NULL` est possible.

---

## ANY, SOME et ALL

Ces opérateurs comparent une valeur à **chaque** élément renvoyé par la sous-requête :

- **`ANY`** (synonyme : `SOME`) : la condition est vraie si elle l'est pour **au moins un** élément ;
- **`ALL`** : la condition est vraie si elle l'est pour **tous** les éléments.

Ils se ramènent souvent à des formes plus simples :

| Forme | Équivaut à |
|-------|-----------|
| `= ANY (…)` | `IN (…)` |
| `<> ALL (…)` | `NOT IN (…)` |
| `> ANY (…)` | `> (SELECT MIN(…))` |
| `> ALL (…)` | `> (SELECT MAX(…))` |

Exemple avec `ALL` — les commandes supérieures à **toutes** celles de Martin (id 2) :

```sql
SELECT id, montant
FROM commande
WHERE montant > ALL (
    SELECT montant FROM commande WHERE client_id = 2
);
```

| id | montant |
|----|---------|
| 3 | 200.00 |

Les montants de Martin étant `80.50` et `150.00`, `> ALL` revient à `> 150` (le maximum) : seule la commande 3 (200) qualifie.

---

## Sous-requête de ligne

La famille **de ligne** compare un **n-uplet de colonnes** — un *constructeur de ligne* `(a, b)` — au résultat de la sous-requête. Associée à `IN`, elle exprime avec élégance un besoin fréquent : retrouver, pour chaque client, sa **commande la plus chère**.

```sql
SELECT id, client_id, montant
FROM commande
WHERE (client_id, montant) IN (
    SELECT client_id, MAX(montant)
    FROM commande
    GROUP BY client_id
);
```

| id | client_id | montant |
|----|-----------|---------|
| 3 | 1 | 200.00 |
| 5 | 2 | 150.00 |
| 4 | 3 | 80.50 |

La sous-requête produit une paire `(client_id, montant maximal)` par client ; la requête englobante ne conserve que les commandes dont le couple `(client_id, montant)` figure dans cet ensemble. Si plusieurs commandes d'un même client atteignaient ce maximum, **toutes** seraient renvoyées — un classement plus fin relève des **window functions** ([section 4.2](../04-concepts-avances-sql/02-window-functions.md)). L'ordre des lignes n'étant pas garanti, ajoutez un `ORDER BY` si l'affichage doit être trié.

---

## Sous-requêtes corrélées vs non corrélées

C'est la distinction la plus importante pour comprendre le **comportement et le coût** d'une sous-requête.

- Une **sous-requête non corrélée** ne fait aucune référence à la requête englobante. Elle peut s'exécuter seule et n'est évaluée **qu'une fois** (cas de la moyenne générale, du `IN` plus haut).
- Une **sous-requête corrélée** référence une ou plusieurs colonnes de la requête extérieure. Conceptuellement, elle est **réévaluée pour chaque ligne** de la requête englobante (cas de l'`EXISTS` plus haut).

Une sous-requête scalaire **corrélée** placée dans le `SELECT` permet de calculer une valeur par ligne. Exemple : le nombre de commandes de chaque client.

```sql
SELECT
    c.nom,
    (SELECT COUNT(*)
     FROM commande cmd
     WHERE cmd.client_id = c.id) AS nb_commandes
FROM client c;
```

| nom | nb_commandes |
|-----|--------------|
| Dupont | 2 |
| Martin | 2 |
| Leroy | 1 |
| Petit | 0 |

Petit obtient `0` (un `COUNT` sur un ensemble vide, voir [section 3.1](01-fonctions-agregation.md)) et figure bien au résultat — contrairement à ce qu'aurait donné un `INNER JOIN`. Ce résultat est équivalent à un `LEFT JOIN ... GROUP BY` ([section 3.3.2](03.2-left-right-join.md)), mais la forme corrélée, réévaluée pour chaque client, peut être plus coûteuse sur de gros volumes (voir plus bas).

---

## Tables dérivées : une sous-requête dans le FROM

Une sous-requête placée dans le `FROM` est une **table dérivée** (ou *vue en ligne*) : MariaDB la traite comme une table temporaire interrogeable. Elle est indispensable lorsqu'on doit **agréger un résultat déjà agrégé**, ce qu'une simple imbrication d'agrégats ne permet pas. Exemple : la moyenne des totaux dépensés *par client*.

```sql
SELECT AVG(total_client) AS moyenne_des_totaux
FROM (
    SELECT client_id, SUM(montant) AS total_client
    FROM commande
    GROUP BY client_id
) AS totaux;
```

| moyenne_des_totaux |
|--------------------|
| 210.333333 |

La table dérivée calcule d'abord le total par client (`320`, `230.50`, `80.50`), puis la requête englobante en fait la moyenne (`631 / 3 ≈ 210.33`).

> ⚠️ Une table dérivée **doit obligatoirement porter un alias** (ici `AS totaux`), même s'il n'est pas réutilisé. C'est une exigence de syntaxe.

Lorsqu'une même table dérivée doit être réutilisée ou que l'imbrication nuit à la lisibilité, les **expressions de table communes (CTE)** — `WITH ... AS (...)` — offrent une alternative plus claire, traitée en [section 4.4](../04-concepts-avances-sql/04-expressions-table-communes.md).

---

## Sous-requêtes ou jointures ? Note de performance

De nombreuses sous-requêtes (`IN`, `EXISTS` corrélés) sont logiquement équivalentes à des jointures. Le choix se fait sur la lisibilité et la performance :

- L'optimiseur de MariaDB applique des **transformations** aux sous-requêtes `IN`/`EXISTS` (semi-jointure, matérialisation) pour les exécuter efficacement, souvent comme des jointures.
- Pour de l'**agrégation par entité**, un `LEFT JOIN ... GROUP BY` est généralement plus performant qu'une sous-requête scalaire corrélée dans le `SELECT`, qui se réévalue ligne par ligne.
- En cas de doute, comparez les plans avec `EXPLAIN` ([section 5.7](../05-index-et-performance/07-analyse-plans-execution.md)) et reportez-vous aux techniques d'[optimisation des requêtes](../05-index-et-performance/08-optimisation-requetes.md).

La règle pragmatique : écrivez la forme la plus **lisible** pour exprimer votre intention, puis vérifiez le plan si la requête est critique.

---

## Autres emplacements

Une sous-requête peut aussi apparaître :

- dans le **`HAVING`**, pour comparer un agrégat de groupe à une valeur calculée (ex. `HAVING SUM(montant) > (SELECT AVG(...) ...)`) ;
- dans les instructions **`INSERT`, `UPDATE` et `DELETE`**, pour cibler des lignes à partir d'une autre requête. Le cas particulier d'un `UPDATE`/`DELETE` s'appuyant sur une CTE est abordé en [section 4.4.1](../04-concepts-avances-sql/04.1-update-delete-from-cte.md), et les requêtes complexes multi-tables en [section 4.5](../04-concepts-avances-sql/05-requetes-complexes-multi-tables.md).

---

## À retenir

- Une sous-requête est un `SELECT` imbriqué dont la requête englobante exploite le résultat.
- Selon ce qu'elle renvoie : **scalaire** (une valeur, avec `=`/`<`/`>`…), **de ligne** (constructeur `(a, b) IN …`), ou **de table** (avec `IN`, `EXISTS`, `ANY`, `ALL`, ou dans le `FROM`).
- Une sous-requête **non corrélée** s'évalue une fois ; une sous-requête **corrélée** (qui référence la requête englobante) se réévalue par ligne.
- `EXISTS`/`NOT EXISTS` testent l'existence ; `NOT EXISTS` est l'anti-jointure **robuste** à préférer à `NOT IN`, sensible aux `NULL`.
- `= ANY` ≡ `IN`, `<> ALL` ≡ `NOT IN`, `> ALL` ≡ `> MAX`, `> ANY` ≡ `> MIN`.
- Une **table dérivée** (sous-requête dans `FROM`, **alias obligatoire**) permet d'agréger un résultat déjà agrégé ; les CTE en sont une alternative lisible ([section 4.4](../04-concepts-avances-sql/04-expressions-table-communes.md)).
- Sous-requête ou jointure : choisir la forme lisible, vérifier le plan avec `EXPLAIN` si nécessaire.

---

## Navigation

- ⬅️ Section précédente : [3.3.5 — Syntaxe ( + ) pour jointures externes en mode Oracle](03.5-oracle-outer-join.md)
- ➡️ Section suivante : [3.5 — Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)](05-operateurs-ensemblistes.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)](/03-requetes-sql-intermediaires/05-operateurs-ensemblistes.md)

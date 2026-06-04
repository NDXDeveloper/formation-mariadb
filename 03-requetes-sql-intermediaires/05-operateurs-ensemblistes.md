🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.5  
> Version de référence : **MariaDB 12.3 LTS**

---

## Définition

Là où une jointure ([section 3.3](03-jointures.md)) combine des tables **horizontalement** — en accolant leurs colonnes —, les **opérateurs ensemblistes** combinent les résultats de plusieurs `SELECT` **verticalement**, en empilant leurs lignes. Ils répondent à des questions de type ensembliste : « l'union de ces deux listes », « ce qui est commun aux deux », « ce qui est dans l'une mais pas dans l'autre ».

MariaDB propose les trois opérateurs du standard SQL :

- **`UNION`** — la réunion des deux ensembles ;
- **`INTERSECT`** — l'intersection (les lignes présentes des deux côtés) ;
- **`EXCEPT`** — la différence (les lignes du premier ensemble absentes du second).

---

## Règles communes à tous les opérateurs

Quel que soit l'opérateur, les `SELECT` combinés doivent être **compatibles** :

- **même nombre de colonnes** dans chaque `SELECT` (un nombre différent provoque une erreur) ;
- des **types compatibles**, colonne par colonne, dans l'ordre (MariaDB applique au besoin une conversion implicite) ;
- les **noms de colonnes du résultat sont ceux du premier `SELECT`** : ceux des `SELECT` suivants sont ignorés.

C'est la position des colonnes, et non leur nom, qui détermine l'alignement : la première colonne du premier `SELECT` est appariée à la première du second, et ainsi de suite.

---

## Tables d'exemple

Pour illustrer ces opérations, prenons deux listes de clients selon leur canal d'acquisition — en ligne et en magasin — sachant que certains clients existent dans les deux :

**`client_web`**

| email | nom |
|-------|-----|
| alice@ex.fr | Alice |
| bruno@ex.fr | Bruno |
| chloe@ex.fr | Chloé |

**`client_magasin`**

| email | nom |
|-------|-----|
| bruno@ex.fr | Bruno |
| david@ex.fr | David |
| chloe@ex.fr | Chloé |

Bruno et Chloé figurent dans les deux canaux ; Alice est exclusivement web, David exclusivement magasin.

---

## UNION et UNION ALL

**`UNION`** réunit les deux résultats **en supprimant les doublons** :

```sql
SELECT email, nom FROM client_web
UNION
SELECT email, nom FROM client_magasin;
```

| email | nom |
|-------|-----|
| alice@ex.fr | Alice |
| bruno@ex.fr | Bruno |
| chloe@ex.fr | Chloé |
| david@ex.fr | David |

On obtient **4 lignes** : Bruno et Chloé, présents dans les deux tables, n'apparaissent qu'une fois.

**`UNION ALL`**, lui, **conserve tous les doublons** :

```sql
SELECT email, nom FROM client_web
UNION ALL
SELECT email, nom FROM client_magasin;
```

Le résultat compte cette fois **6 lignes** (les 3 + 3 lignes brutes), Bruno et Chloé y figurant deux fois.

> 💡 **Performance.** `UNION` doit effectuer un tri ou un hachage pour éliminer les doublons, opération coûteuse. Si vous **savez** qu'il n'y a pas de doublons, ou s'ils ne vous gênent pas, préférez `UNION ALL`, nettement plus rapide. N'utilisez `UNION` que lorsque la déduplication est réellement nécessaire.

---

## INTERSECT

**`INTERSECT`** ne conserve que les lignes **présentes dans les deux** ensembles — les clients connus des deux canaux :

```sql
SELECT email, nom FROM client_web
INTERSECT
SELECT email, nom FROM client_magasin;
```

| email | nom |
|-------|-----|
| bruno@ex.fr | Bruno |
| chloe@ex.fr | Chloé |

Comme `UNION`, `INTERSECT` **supprime les doublons** par défaut. Une variante `INTERSECT ALL` existe pour conserver la multiplicité (sémantique de multiensembles), mais le cas courant est la forme simple.

---

## EXCEPT

**`EXCEPT`** renvoie les lignes du **premier** ensemble qui ne figurent **pas** dans le second — ici, les clients exclusivement web :

```sql
SELECT email, nom FROM client_web
EXCEPT
SELECT email, nom FROM client_magasin;
```

| email | nom |
|-------|-----|
| alice@ex.fr | Alice |

Deux points importants :

- **`EXCEPT` n'est pas commutatif.** Inverser l'ordre des `SELECT` (`client_magasin EXCEPT client_web`) renverrait au contraire David, le client exclusivement magasin. L'ordre détermine quel ensemble on « retranche » de l'autre.
- Comme les autres, `EXCEPT` **déduplique** par défaut ; une variante `EXCEPT ALL` existe pour les multiensembles. Son équivalent se nomme **`MINUS`** sous Oracle — utile à connaître en migration.

---

## ORDER BY et LIMIT

Un `ORDER BY` (ou `LIMIT`) placé **à la fin** s'applique au **résultat combiné dans son ensemble**, et non au dernier `SELECT` :

```sql
SELECT email, nom FROM client_web
UNION
SELECT email, nom FROM client_magasin
ORDER BY nom;
```

Le tri porte ici sur l'ensemble réuni, ordonné par `nom` (Alice, Bruno, Chloé, David). Les colonnes du tri final se réfèrent aux **noms du premier `SELECT`**.

Pour au contraire trier ou limiter **un `SELECT` individuel** avant la combinaison, il faut l'entourer de **parenthèses** :

```sql
(SELECT email, nom FROM client_web     ORDER BY nom LIMIT 2)
UNION
(SELECT email, nom FROM client_magasin ORDER BY nom LIMIT 2);
```

---

## Priorité des opérateurs

Lorsqu'on enchaîne plusieurs opérateurs différents, l'ordre d'évaluation n'est pas anodin : **`INTERSECT` a une priorité plus élevée** que `UNION` et `EXCEPT` (conformément au standard SQL). Dans une requête mêlant `UNION` et `INTERSECT`, l'`INTERSECT` est donc évalué en premier.

Pour lever toute ambiguïté et rendre l'intention explicite, **utilisez des parenthèses** autour des sous-ensembles :

```sql
(SELECT ... )
UNION
(SELECT ... INTERSECT SELECT ...);
```

C'est une bonne pratique systématique dès que plus de deux `SELECT` ou plus d'un type d'opérateur sont en jeu.

---

## Déduplication et valeurs NULL

La suppression des doublons opérée par `UNION`, `INTERSECT` et `EXCEPT` suit la même logique que `GROUP BY` ([section 3.2](02-regroupement-donnees.md)) : deux lignes sont considérées comme identiques si **toutes** leurs colonnes coïncident, et **deux `NULL` sont traités comme égaux** pour ce rapprochement. C'est cohérent avec le comportement du regroupement, mais c'est une exception à la règle habituelle où `NULL = NULL` est indéterminé (voir [section 4.6](../04-concepts-avances-sql/06-gestion-valeurs-null.md)).

---

## Cas d'usage

Les opérateurs ensemblistes servent notamment à :

- **réunir des sources hétérogènes** de même structure (plusieurs tables d'archives, plusieurs canaux), comme dans nos exemples ;
- **comparer deux ensembles** (clients communs avec `INTERSECT`, écarts avec `EXCEPT`) — par exemple pour de la réconciliation de données ;
- **émuler un `FULL OUTER JOIN`**, absent de MariaDB, en réunissant par `UNION` un `LEFT JOIN` et un `RIGHT JOIN` (technique présentée en [section 3.3.2](03.2-left-right-join.md)) — c'est précisément l'`UNION` qui y dédoublonne les lignes communes aux deux jointures.

---

## À retenir

- Les opérateurs ensemblistes empilent des lignes **verticalement** ; les jointures combinent des colonnes horizontalement.
- Chaque `SELECT` doit avoir le **même nombre de colonnes** et des **types compatibles** ; les **noms du résultat viennent du premier `SELECT`**.
- `UNION` déduplique ; **`UNION ALL` conserve les doublons** et est plus rapide.
- `INTERSECT` = lignes présentes des deux côtés ; `EXCEPT` = lignes du premier ensemble absentes du second (**non commutatif**, `MINUS` sous Oracle). Les deux dédupliquent par défaut (variantes `… ALL`).
- `ORDER BY`/`LIMIT` final portent sur l'**ensemble combiné** ; pour agir sur un `SELECT` isolé, l'entourer de parenthèses.
- `INTERSECT` a une **priorité plus élevée** que `UNION`/`EXCEPT` : parenthésez pour lever l'ambiguïté.
- La déduplication traite **deux `NULL` comme égaux** (comme `GROUP BY`).

---

## Navigation

- ⬅️ Section précédente : [3.4 — Sous-requêtes et requêtes imbriquées](04-sous-requetes.md)
- ➡️ Section suivante : [3.6 — Fonctions de chaînes de caractères](06-fonctions-chaines.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Fonctions de chaînes de caractères](/03-requetes-sql-intermediaires/06-fonctions-chaines.md)

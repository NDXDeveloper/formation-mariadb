🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.7 — Analyse des plans d'exécution (EXPLAIN et ANALYZE)

> **Chapitre 5 — Index et Performance** · Section 5.7  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Toutes les sections précédentes y renvoyaient comme au juge de paix : `EXPLAIN`. C'est l'outil qui révèle **comment** MariaDB exécute réellement une requête — quel index il choisit (ou ignore), comment il joint les tables, s'il doit trier ou créer une table temporaire. Sans lui, l'optimisation relève de la devinette ; avec lui, chaque décision d'indexation se vérifie objectivement. Cette section explique comment lire un plan d'exécution et en tirer des conclusions concrètes.

---

## EXPLAIN : le plan prévu

Précéder une requête `SELECT`, `UPDATE` ou `DELETE` du mot-clé **`EXPLAIN`** affiche le **plan d'exécution que l'optimiseur compte suivre** — sans renvoyer le résultat, et (pour un `SELECT`) sans exécuter la requête : ce ne sont que des **estimations**.

```sql
EXPLAIN SELECT * FROM commandes WHERE client_id = 42;
```

La sortie tabulaire comporte une ligne par table accédée, avec les colonnes suivantes :

| Colonne | Signification |
|---------|---------------|
| `id` | Identifiant du `SELECT` (utile avec sous-requêtes et `UNION`) |
| `select_type` | Nature du `SELECT` (`SIMPLE`, `PRIMARY`, `SUBQUERY`, `DERIVED`, `UNION`…) |
| `table` | Table (ou alias) accédée |
| `type` | **Méthode d'accès** — la colonne la plus importante |
| `possible_keys` | Index que l'optimiseur **pourrait** utiliser |
| `key` | Index **réellement** choisi (`NULL` = aucun) |
| `key_len` | Nombre d'octets de l'index utilisés |
| `ref` | Ce qui est comparé à l'index (`const`, une colonne…) |
| `rows` | Nombre **estimé** de lignes examinées |
| `filtered` | Pourcentage **estimé** de lignes passant la condition — colonne affichée par **`EXPLAIN EXTENDED`**, pas par l'`EXPLAIN` simple (voir plus bas) |
| `Extra` | Annotations textuelles (tri, table temporaire, index couvrant…) |

### Une sortie concrète, lue d'un coup d'œil

Les descriptions ci-dessus prennent tout leur sens sur un exemple réel. Sur une table `commandes` de 50 000 lignes dotée d'un index `idx_client (client_id)`, la requête précédente produit un plan de ce genre — présenté ici en **vertical** (terminateur `\G`), plus lisible que la grille tabulaire :

```
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: commandes
         type: ref            -- accès indexé par égalité
possible_keys: idx_client
          key: idx_client     -- l'index est bien utilisé…
      key_len: 4              -- …sur 4 octets = la colonne INT client_id
          ref: const          -- comparée à une constante (42)
         rows: 10             -- ~10 lignes estimées (sur 50 000)
        Extra:                -- ni « Using filesort » ni « Using temporary »
```

Tout est au vert : accès `ref`, index choisi, peu de lignes lues, aucun tri superflu. À l'inverse, sans index sur `client_id` (ou si la condition n'est pas *sargable*, cf. [section 5.8](08-optimisation-requetes.md)), le même plan bascule en :

```
         type: ALL            -- balayage complet de la table
          key: NULL           -- aucun index utilisé
         rows: 50000          -- toute la table est examinée
        Extra: Using where
```

Ce contraste résume à lui seul la lecture d'un plan : on vise un `type` meilleur qu'`ALL`, une `key` non nulle, un `rows` faible, et un `Extra` exempt de `Using filesort` et `Using temporary`.

---

## Lire le plan : les colonnes qui comptent

### `type` : la méthode d'accès

C'est **la première colonne à examiner**. Elle indique comment MariaDB trouve les lignes, de la plus efficace à la plus coûteuse :

| `type` | Signification | Qualité |
|--------|---------------|:-------:|
| `const` / `system` | Une seule ligne, via PK/UNIQUE sur une constante | ⭐ Optimal |
| `eq_ref` | Une ligne par jointure, via index unique (PK/UNIQUE) | ⭐ Excellent |
| `ref` | Plusieurs lignes, via index non unique, par égalité | ✅ Bon |
| `range` | Balayage d'une **plage** d'index (`BETWEEN`, `>`, `IN`…) | ✅ Correct |
| `index` | Balayage de l'index **entier** | ⚠️ À surveiller |
| `ALL` | Balayage **complet de la table** | ❌ Souvent un problème |

Sur une grosse table, voir `ALL` (ou parfois `index`) est le signal d'alerte typique : aucun index adapté n'est utilisé — souvent à cause de l'un des pièges de la [section 5.5.1](05.1-index-colonnes-filtrees.md).

### `key`, `possible_keys` et `key_len`

`key` indique l'index effectivement retenu. S'il vaut **`NULL`** alors qu'un index pertinent figure dans `possible_keys` (voire n'y figure pas du tout), c'est qu'une condition empêche son usage. La colonne **`key_len`** révèle le nombre d'octets — donc, indirectement, le **nombre de colonnes** d'un index composite — réellement exploités : un `key_len` plus court qu'attendu trahit un préfixe rompu (cf. [section 5.6](06-index-composites.md)).

### `rows` et `filtered`

`rows` estime le nombre de lignes que MariaDB devra examiner : plus c'est faible relativement à la taille de la table, mieux c'est. `filtered` estime le pourcentage de ces lignes qui satisferont la condition.

> **Précision MariaDB.** À la différence de MySQL, la colonne **`filtered` n'apparaît pas** dans la sortie d'`EXPLAIN` simple. On l'obtient avec **`EXPLAIN EXTENDED`**, ou avec **`ANALYZE`** — qui affiche alors côte à côte l'estimation `filtered` et la valeur **réellement mesurée** `r_filtered` (voir plus bas).

### `Extra` : les annotations décisives

La colonne `Extra` condense des informations capitales :

| Mention `Extra` | Signification | Lecture |
|-----------------|---------------|---------|
| `Using index` | Index **couvrant** (*index-only scan*), pas d'accès à la table | ✅ Très bon (cf. 5.9) |
| `Using where` | Filtrage appliqué après lecture des lignes | Neutre, selon contexte |
| `Using index condition` | *Index Condition Pushdown* (ICP) | ✅ Bon (cf. 5.8.1) |
| `Using filesort` | Tri supplémentaire nécessaire | ⚠️ À éliminer (cf. 5.5.3) |
| `Using temporary` | Table temporaire (souvent `GROUP BY`/`DISTINCT`) | ⚠️ À éliminer (cf. 5.5.3) |
| `Using index for group-by` | *Loose index scan* | ✅ Très bon (cf. 5.5.3) |
| `Using join buffer` | Jointure **sans index** (BNL/BKA) | ⚠️ Index de jointure manquant (cf. 5.5.2) |

---

## ANALYZE : le plan réel

`EXPLAIN` ne donne que des **estimations**. Pour confronter ces prévisions à la réalité, MariaDB fournit la commande **`ANALYZE`**.

> **Une précision de nommage importante.** Ce que beaucoup appellent « `EXPLAIN ANALYZE` » (notamment en venant de MySQL) s'obtient en MariaDB avec l'instruction **`ANALYZE`**, disponible depuis MariaDB 10.1. MariaDB **ne reconnaît pas** la syntaxe `EXPLAIN ANALYZE` (elle provoque une erreur de syntaxe) : c'est donc `ANALYZE SELECT …` qu'il faut écrire.

À la différence d'`EXPLAIN`, `ANALYZE` **exécute réellement** la requête, puis renvoie la sortie d'`EXPLAIN` **enrichie de statistiques d'exécution mesurées** :

```sql
ANALYZE SELECT * FROM commandes WHERE client_id = 42;
```

Elle ajoute deux colonnes par rapport à `EXPLAIN` :

- **`r_rows`** : le nombre **réel** de lignes lues, à comparer à l'estimation `rows` ;
- **`r_filtered`** : le pourcentage **réel** de lignes ayant passé la condition, à comparer à `filtered`.

L'intérêt est de **mesurer l'écart entre l'estimation et la réalité**. Un fossé important entre `rows` et `r_rows` signale que les **statistiques de l'optimiseur sont dépassées** — un cas que `ANALYZE TABLE` corrige ([chapitre 11](../11-administration-configuration/06.2-analyze-table.md)). Un repère pratique utile : en cas de **balayage complet** assorti d'un **`r_filtered` inférieur à ~15 %**, c'est le signe qu'un **index approprié** serait bénéfique (on lit beaucoup de lignes pour n'en conserver qu'une faible part).

---

## Formats détaillés et variantes

Plusieurs déclinaisons complètent la panoplie :

- **`FORMAT=JSON`** — `EXPLAIN FORMAT=JSON …` et `ANALYZE FORMAT=JSON …` produisent une vue **bien plus détaillée** du plan. La version `ANALYZE` y ajoute des mesures temporelles comme **`r_total_time_ms`** et **`r_loops`**, précieuses pour localiser l'étape coûteuse d'une requête.
- **`EXPLAIN EXTENDED`** — fournit des informations additionnelles (et la requête réécrite par l'optimiseur, consultable ensuite via `SHOW WARNINGS`).
- **`EXPLAIN PARTITIONS`** — utile pour les tables **partitionnées**, afin de vérifier l'élagage de partitions (cf. [chapitre 15](../15-performance-tuning/09.4-partition-pruning.md)).
- **`SHOW EXPLAIN FOR <id>`** / **`EXPLAIN FOR CONNECTION <id>`** — examinent le plan d'une requête **en cours d'exécution** dans une autre connexion (l'`<id>` provient de `SHOW PROCESSLIST`). Indispensable lorsqu'une requête lente n'en finit pas et qu'on ne peut donc pas attendre la sortie d'`ANALYZE`.

Enfin, la sortie d'`EXPLAIN` peut être **journalisée dans le slow query log** ([section 15.7](../15-performance-tuning/07-analyse-requetes-lentes.md)), ce qui permet d'analyser après coup les plans des requêtes lentes capturées.

---

## Une démarche de diagnostic

En pratique, l'analyse d'un plan suit toujours la même grille de lecture, qui synthétise tout le chapitre :

1. **`type`** : voit-on `ALL` sur une grosse table ? → index manquant ou neutralisé ([5.5.1](05.1-index-colonnes-filtrees.md)).
2. **`key`** : l'index attendu est-il utilisé ? S'il est `NULL`, pourquoi ? (fonction sur la colonne, conversion de type, plage en amont…).
3. **`key_len`** : tout le préfixe du composite est-il exploité ? ([5.6](06-index-composites.md)).
4. **`Extra`** : éliminer `Using filesort` et `Using temporary` ([5.5.3](05.3-index-order-group.md)) ; se réjouir d'un `Using index` (couvrant, [5.9](09-index-covering.md)).
5. **`rows` vs `r_rows`** (via `ANALYZE`) : un grand écart appelle un `ANALYZE TABLE` pour rafraîchir les statistiques.

---

> ### 📝 À retenir  
>  
> - **`EXPLAIN`** affiche le plan **prévu** (estimations, sans exécuter le `SELECT`) pour `SELECT`/`UPDATE`/`DELETE`.  
> - La colonne **`type`** est la plus importante : viser `const`/`eq_ref`/`ref`/`range` ; se méfier de **`ALL`** (balayage complet) et de `index`.  
> - Surveiller **`key`** (index réellement choisi), **`key_len`** (combien de colonnes d'un composite, cf. 5.6) et **`Extra`** (éliminer `Using filesort` et `Using temporary` ; `Using index` = couvrant).  
> - En MariaDB, l'« EXPLAIN ANALYZE » s'écrit **`ANALYZE`** (depuis 10.1) : il **exécute** la requête et ajoute **`r_rows`** et **`r_filtered`** (réel vs estimé). Un balayage complet avec `r_filtered` < ~15 % appelle un index.  
> - Variantes utiles : **`FORMAT=JSON`** (détaillé, avec `r_total_time_ms`), **`EXPLAIN EXTENDED`**, **`EXPLAIN PARTITIONS`**, et **`SHOW EXPLAIN FOR`** pour une requête en cours.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.6 Index composites et ordre des colonnes](06-index-composites.md)
- ➡️ Section suivante : [5.8 Optimisation des requêtes](08-optimisation-requetes.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Optimisation des requêtes](/05-index-et-performance/08-optimisation-requetes.md)

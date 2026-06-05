🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6 · Performance des vues : `MERGE` vs `TEMPTABLE`

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

Une vue n'a pas de coût intrinsèque : ce n'est qu'une requête enregistrée. En revanche, **la manière dont MariaDB la combine** avec la requête qui l'interroge fait toute la différence de performance. Deux stratégies — deux **algorithmes** — sont possibles, et le choix entre elles peut transformer une lecture instantanée en un balayage complet de table. Cette section explique ces algorithmes, leur impact, et comment vérifier celui qui est réellement employé.

## Deux façons d'exécuter une vue

Lorsqu'une requête référence une vue, MariaDB doit réconcilier deux textes SQL : la définition de la vue et la requête appelante. Il dispose pour cela de deux algorithmes, déclarables via la clause `ALGORITHM` de `CREATE`/`ALTER VIEW` :

```sql
CREATE ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE} VIEW ...
```

- **`MERGE`** : la définition de la vue est **fusionnée** dans la requête appelante, qui s'exécute alors directement sur les tables de base.
- **`TEMPTABLE`** : le résultat de la vue est d'abord **matérialisé** dans une table temporaire, sur laquelle la requête appelante s'exécute ensuite.
- **`UNDEFINED`** (valeur **par défaut**) : MariaDB **choisit** entre les deux, en privilégiant `MERGE` chaque fois que c'est possible.

## L'algorithme `MERGE` : fusionner la vue dans la requête

Avec `MERGE`, la vue est **transparente** : sa requête est intégrée à celle de l'appelant, et le tout est optimisé comme une seule requête. Une interrogation telle que `SELECT * FROM v WHERE id = 5` devient, en substance, une requête équivalente directement posée sur la table sous-jacente, avec la condition `id = 5` intégrée.

Les bénéfices sont décisifs : **aucune table temporaire** n'est créée ; les **index** des tables de base restent exploitables ; les **conditions** de la requête appelante peuvent être *poussées* jusqu'aux tables sous-jacentes pour n'en lire que les lignes utiles. C'est aussi la condition pour qu'une vue soit **modifiable** (§9.3).

```sql
CREATE ALGORITHM = MERGE VIEW v_emp_merge AS
SELECT id, nom, prenom, salaire, dept_id
FROM employes;

SELECT * FROM v_emp_merge WHERE id = 5;
```

Ici, la clause `WHERE id = 5` atteint la table `employes` et exploite l'index de clé primaire : **une seule ligne** est lue.

## L'algorithme `TEMPTABLE` : matérialiser la vue

Avec `TEMPTABLE`, MariaDB exécute d'abord la requête de la vue, **stocke tout son résultat** dans une table temporaire, puis applique la requête appelante à cette table temporaire. Les conséquences sont l'inverse de `MERGE` : une table temporaire est **construite et peuplée à chaque interrogation** (avec allocation mémoire, voire débordement sur disque si le volume est important) ; les **index des tables de base ne servent plus** à la requête appelante, qui ne voit que la table temporaire ; et la vue n'est **pas modifiable**, la correspondance ligne à ligne étant rompue par la matérialisation.

Cet algorithme présente un avantage situationnel et désormais mineur : une fois la table temporaire construite, les **verrous** sur les tables de base peuvent être relâchés plus tôt. Avec le MVCC d'InnoDB, ce point pèse rarement dans la balance.

## `UNDEFINED` : laisser l'optimiseur décider

C'est le comportement par défaut, et le bon choix dans la quasi-totalité des cas : MariaDB retient `MERGE` lorsque la vue s'y prête, et bascule sur `TEMPTABLE` sinon. On ne déclare explicitement un algorithme que pour une raison précise.

## Quand `MERGE` est impossible

Certaines constructions empêchent la fusion : MariaDB **doit** alors matérialiser. Une vue ne peut pas utiliser `MERGE` si sa définition contient :

- une **fonction d'agrégation** (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`…) ;
- une clause **`GROUP BY`** ou **`HAVING`** ;
- le mot-clé **`DISTINCT`** ;
- une clause **`LIMIT`** ;
- un opérateur **`UNION`** / **`UNION ALL`** ;
- une **sous-requête dans la liste du `SELECT`** ;
- une définition ne portant que sur des **valeurs littérales** (aucune table sous-jacente).

Cette liste recoupe très largement celle des constructions qui rendent une vue **non modifiable** (§9.3) — ce n'est pas une coïncidence : une vue qui doit être matérialisée perd la correspondance ligne à ligne, et n'est donc plus modifiable. **Matérialisation et non-modifiabilité ont la même cause racine.**

Conséquence pratique : déclarer `ALGORITHM = MERGE` sur une vue qui contient l'une de ces constructions **ne force rien**. MariaDB émet un **avertissement** et retombe sur `UNDEFINED` (donc `TEMPTABLE` pour cette vue).

## Le piège du filtre tardif

C'est l'erreur de performance classique. Comparons deux vues identiques, l'une fusionnée, l'autre matérialisée :

```sql
CREATE ALGORITHM = MERGE     VIEW v_emp_merge AS
SELECT id, nom, prenom, salaire, dept_id FROM employes;

CREATE ALGORITHM = TEMPTABLE VIEW v_emp_temp  AS
SELECT id, nom, prenom, salaire, dept_id FROM employes;
```

Puis la **même** requête filtrée sur chacune :

```sql
SELECT * FROM v_emp_merge WHERE id = 5;   -- filtre poussé : index PK, 1 ligne lue
SELECT * FROM v_emp_temp  WHERE id = 5;   -- TOUTE la table matérialisée, PUIS filtrée
```

Sur `v_emp_merge`, le filtre atteint la table et n'en lit qu'une ligne. Sur `v_emp_temp`, MariaDB matérialise **l'intégralité** de `employes` dans une table temporaire — sans index utile —, puis n'y conserve que la ligne `id = 5`. Sur une grande table, le coût est sans commune mesure : on paie la construction du résultat complet avant même de filtrer.

`EXPLAIN` (§5.7) révèle clairement la différence. Sur la vue fusionnée, le plan référence directement `employes` avec un accès par clé primaire (type `const`/`eq_ref`). Sur la vue matérialisée, le plan fait apparaître une ligne **`DERIVED`** (la table temporaire) alimentée par un **balayage complet** (type `ALL`) de `employes` :

```sql
EXPLAIN SELECT * FROM v_emp_temp WHERE id = 5;
-- … une ligne de type PRIMARY interrogeant <derived2>
-- … une ligne de select_type DERIVED faisant un scan ALL de « employes »
```

`EXPLAIN FORMAT=JSON` confirme la matérialisation (un nœud **`materialized`** apparaît dans le plan), et la commande **`ANALYZE`** (§5.7) en donne le coût réel d'exécution.

> **Précision.** Le balayage `ALL` ci-dessus illustre le cas où la **poussée de condition** (section suivante) ne s'applique pas. Or cette optimisation est **active par défaut** : sur ce *même* exemple simple, elle injecte en réalité `id = 5` dans la table dérivée, et l'`EXPLAIN` y montre alors un accès `const`. Le coût plein du *filtre tardif* se paie donc surtout sur les vues que cette optimisation ne peut pas aider (agrégations, `GROUP BY`…).

## Atténuer le coût : le *condition pushdown*

MariaDB n'est pas démuni face aux vues qui *doivent* être matérialisées. L'optimisation de **poussée de condition dans les tables dérivées et les vues** (*condition pushdown for derived tables*) peut, dans bien des cas, **injecter la condition de la requête appelante au sein de la matérialisation**, de sorte que la table temporaire ne contienne que les lignes pertinentes. Sur l'exemple précédent, cette optimisation transfère `id = 5` à l'intérieur de la vue matérialisée, réduisant drastiquement sa taille.

Ce comportement est gouverné par `optimizer_switch` (commutateur `condition_pushdown_for_derived`) et peut être contrôlé finement par l'**Optimizer Hint** `DERIVED_CONDITION_PUSHDOWN` (§15.15.4). De même, pour les vues matérialisées impliquées dans des **jointures**, l'optimiseur sait ajouter une clé à la table temporaire (`derived_with_keys`) afin d'accélérer la jointure. Ces mécanismes atténuent le coût de `TEMPTABLE`, mais ne suppriment pas l'arbitrage de fond : à structure égale, `MERGE` reste préférable.

## Connaître l'algorithme réellement employé

Deux niveaux d'information à ne pas confondre. La colonne **`ALGORITHM`** d'`INFORMATION_SCHEMA.VIEWS` (et la clause restituée par `SHOW CREATE VIEW`) indiquent l'algorithme **déclaré** :

```sql
SELECT TABLE_NAME, ALGORITHM
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE();
```

Mais une valeur `UNDEFINED` signifie justement que **l'optimiseur tranche au cas par cas**, requête par requête. L'algorithme **effectivement appliqué** pour une interrogation donnée ne se lit donc pas dans cette colonne, mais dans le **plan d'exécution** : c'est la présence (ou l'absence) d'une étape `DERIVED` / de matérialisation dans `EXPLAIN` qui fait foi.

## Bonnes pratiques

- **Privilégier `MERGE` ou `UNDEFINED`** pour les vues simples et filtrables, en particulier celles qui servent de briques réutilisées et re-filtrées par d'autres requêtes.
- **Ne pas imposer `ALGORITHM = TEMPTABLE`** sans raison : cela désactive la modifiabilité et dégrade en général les performances.
- Pour les vues qui *doivent* être matérialisées (agrégations, `GROUP BY`…), **filtrer le plus tôt possible** — en intégrant la condition dans la vue, ou en s'appuyant sur la poussée de condition.
- Se méfier des **vues imbriquées** (vue sur vue, §9.1) : elles peuvent forcer la matérialisation et compliquer les plans.
- **Toujours vérifier avec `EXPLAIN`** plutôt que de présumer du comportement.

## En résumé

MariaDB exécute une vue selon deux algorithmes : **`MERGE`** fusionne la vue dans la requête appelante — pas de table temporaire, index exploitables, conditions poussées, vue modifiable — tandis que **`TEMPTABLE`** matérialise le résultat dans une table temporaire, au prix d'un surcoût et de la perte d'optimisations. Le mode par défaut **`UNDEFINED`** choisit `MERGE` quand il le peut ; agrégats, `GROUP BY`, `DISTINCT`, `LIMIT`, `UNION` et sous-requêtes dans le `SELECT` l'en empêchent. Le **piège du filtre tardif** — tout matérialiser avant de filtrer — est en partie corrigé par la **poussée de condition** (§15.15.4), mais à structure équivalente, `MERGE` demeure préférable. Seul `EXPLAIN` révèle l'algorithme réellement employé.

Cette section s'est largement appuyée sur `INFORMATION_SCHEMA.VIEWS` pour inspecter les vues. Or `INFORMATION_SCHEMA` est… lui-même un ensemble de vues. La dernière section du chapitre explore ces **vues système** — `INFORMATION_SCHEMA`, `PERFORMANCE_SCHEMA` et les tables système `mysql` : c'est l'objet du **§9.7 — Vues système**.

⏭️ [Vues système](/09-vues-et-donnees-virtuelles/07-vues-systeme.md)

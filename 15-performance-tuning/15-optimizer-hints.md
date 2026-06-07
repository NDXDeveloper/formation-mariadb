🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.15 Optimizer Hints : contrôle fin du plan d'exécution 🆕

L'optimiseur basé sur les coûts (§15.14) choisit le plan d'exécution le moins coûteux d'après ses estimations. La plupart du temps, ce choix est judicieux — mais pas toujours. Lorsque les estimations sont faussées (données déséquilibrées, statistiques périmées, requête intrinsèquement complexe), le plan retenu peut être sous-optimal. Les **optimizer hints** permettent alors d'influencer, voire de forcer, les décisions de l'optimiseur pour une instruction donnée, sans modifier aucun réglage global.

MariaDB a introduit un système de hints « nouvelle génération » dans la série 12.x, de façon progressive : la **12.0** a posé le socle (ordre de jointure, sous-requêtes et certains hints de méthode d'accès), la **12.1** l'a étendu (hints d'index et contrôle de la plupart des optimisations), les versions suivantes ajoutant les optimisations restantes — l'ensemble étant **disponible et consolidé dans la 12.3 LTS**. Aligné sur la syntaxe de MySQL, ce système constitue l'une des fonctionnalités phares de l'optimiseur de cette série. Cette section en présente les principes ; les sous-sections détaillent ensuite chaque famille.

## Pourquoi des hints ?

L'optimiseur décide à partir d'estimations de coût (§15.14), et lorsque ces estimations s'écartent de la réalité, le plan choisi en pâtit. Les causes sont variées : une distribution de données fortement déséquilibrée que les statistiques résument mal, des statistiques devenues obsolètes, une requête dont la combinatoire dépasse ce que l'optimiseur explore, ou un cas limite que le modèle de coût estime de travers. Un hint offre alors une **intervention ciblée** : il corrige le plan d'une requête précise sans toucher à la configuration globale, qui affecterait toutes les autres.

Cela dit, un hint reste un **recours de dernier rang**. Avant d'y recourir, on cherchera d'abord à traiter la cause : remettre les statistiques à jour (`ANALYZE TABLE`), ajouter ou corriger un index (§5.5), ou réécrire la requête. Un hint figé dans une requête peut en effet perdre sa pertinence à mesure que les données et le schéma évoluent.

## Le système de hints « nouvelle génération »

Avant la série 12.x, le contrôle fin de certaines optimisations — accès par clé groupée (BKA), jointure par blocs imbriqués (BNL), pushdown de condition d'index (ICP), lecture multi-intervalle (MRR) — ne pouvait s'exercer que via la variable `optimizer_switch`, et ce **globalement ou par session**, c'est-à-dire pour l'ensemble des requêtes et des tables. La granularité faisait défaut.

Le système nouvelle génération lève cette limite : il permet d'activer ou de désactiver une optimisation **pour une seule requête, une seule table, un seul index ou un seul bloc de requête**. C'est précisément ce que recouvre l'expression « contrôle fin du plan d'exécution ». Étant par ailleurs compatible avec la syntaxe de MySQL, il facilite la portabilité des requêtes entre les deux systèmes.

## La syntaxe : le commentaire /*+ ... */

Les hints s'écrivent dans un **commentaire spécial** de la forme `/*+ ... */`, placé immédiatement après le mot-clé de l'instruction — `SELECT` le plus souvent, mais aussi `INSERT`, `UPDATE` ou `DELETE`, ainsi que le `SELECT` d'un `INSERT ... SELECT`.

```sql
-- Le hint suit immédiatement le mot-clé SELECT
SELECT /*+ NO_BNL() BKA(t1) */ t1.*  
FROM t1 JOIN t2 ON t1.ref = t2.id;  
```

Un même commentaire peut contenir plusieurs hints, et une instruction peut comporter plusieurs commentaires de hints :

```sql
-- Plusieurs hints dans un même commentaire
SELECT /*+ INDEX(t1 idx_date) JOIN_PREFIX(t2, t1) */ ...  
FROM t1, t2 ...;  

-- Fonctionne également avec INSERT ... SELECT
INSERT INTO archive  
SELECT /*+ MAX_EXECUTION_TIME(5000) */ ... FROM commandes WHERE ...;  
```

Les noms de hints, les noms de blocs de requête et les noms de stratégies ne sont **pas sensibles à la casse** ; les références aux noms de tables et d'index suivent en revanche les règles habituelles de sensibilité à la casse des identifiants.

## La granularité : par requête, table, index ou bloc

Tout l'intérêt du système réside dans sa capacité à cibler. Un hint peut s'appliquer à la requête entière, mais aussi à des tables ou des index nommés, voire à un **bloc de requête** déterminé. Pour cela, les blocs de requête peuvent recevoir un nom explicite au moyen du hint `QB_NAME` (§15.15.1), qu'un autre hint pourra ensuite désigner par la notation `@nom_de_bloc`. Cette possibilité de nommer et de cibler les blocs est le préalable à un contrôle réellement fin, ce qui justifie qu'elle ouvre la série des sous-sections.

## Précédence et conflits

Les hints nouvelle génération sont conçus comme plus spécifiques que les réglages de variables serveur : ils **priment** donc en règle générale sur `optimizer_switch` et les paramètres équivalents. Une exception documentée existe néanmoins : le hint `MAX_EXECUTION_TIME` est ignoré, avec un avertissement, si la variable `@@max_statement_time` est définie — ce qui contredit le principe général (voir §15.15.5). Par ailleurs, l'optimiseur ignore tout hint qui constituerait une impossibilité technique : un hint demeure une suggestion qu'il respecte lorsqu'il le peut.

Lorsqu'un commentaire contient des hints **dupliqués ou contradictoires**, la règle est simple : le premier est retenu, et un avertissement signale le second. Un hint en double comme `MRR(idx1) MRR(idx1)`, ou un hint contradictoire comme `BKA(t1) NO_BKA(t1)`, déclenche ainsi l'avertissement `4219` :

```sql
SELECT /*+ BKA(t1) NO_BKA(t1) */ * FROM t1 ...;
-- Warning 4219 : Hint NO_BKA(`t1`) is ignored as conflicting/duplicated
```

## Les familles de hints

Le système se structure en plusieurs familles, chacune détaillée dans une sous-section. Le tableau suivant en donne la cartographie.

| Famille | Hints principaux | Sous-section |
|---------|------------------|--------------|
| Nommage de bloc de requête | `QB_NAME` (et noms implicites `select#1`…) | §15.15.1 |
| Niveau table et index | `NO_ICP`, `MRR`/`NO_MRR`, `BKA`/`NO_BKA`, `BNL`/`NO_BNL`, `[NO_]INDEX`, `JOIN_INDEX`, `GROUP_INDEX`, `ORDER_INDEX`, `NO_RANGE_OPTIMIZATION` | §15.15.2 |
| Ordre de jointure | `JOIN_FIXED_ORDER`, `JOIN_ORDER`, `JOIN_PREFIX`, `JOIN_SUFFIX` | §15.15.3 |
| Sous-requêtes | `SEMIJOIN`, `SUBQUERY`, `SPLIT_MATERIALIZED`, `MERGE`, `DERIVED_CONDITION_PUSHDOWN` | §15.15.4 |
| Limite de temps d'exécution | `MAX_EXECUTION_TIME` | §15.15.5 |

Ces familles couvrent ainsi les principaux leviers de l'optimiseur : le choix des algorithmes de jointure (BKA, BNL), les stratégies d'accès aux index (ICP, MRR, choix ou exclusion d'index), l'ordre de jointure des tables, le traitement des sous-requêtes et des tables dérivées, et enfin la limitation du temps d'exécution.

## Hints nouvelle génération et mécanismes hérités

MariaDB disposait déjà de moyens plus anciens pour influencer l'optimiseur. Les **index hints** classiques — `USE INDEX`, `FORCE INDEX`, `IGNORE INDEX` —, placés après le nom de la table, orientent le choix d'index ; le mot-clé `STRAIGHT_JOIN` impose quant à lui l'ordre de jointure. Ces mécanismes subsistent, mais le système nouvelle génération est plus complet, plus granulaire (jusqu'au bloc de requête) et compatible MySQL. Le hint `JOIN_FIXED_ORDER` (§15.15.3) en est l'illustration : il joue le rôle de l'ancien `STRAIGHT_JOIN`.

Le rapport avec le modèle de coût (§15.14) mérite enfin d'être souligné. On pourrait théoriquement infléchir un plan en ajustant les coûts, mais cet ajustement est global et risqué (§15.14). Pour corriger **une seule requête**, un hint est bien plus sûr qu'une modification des coûts ou de `optimizer_switch`, dont la portée déborde la requête visée.

## Bonnes pratiques et mises en garde

Les hints s'emploient avec discernement, de façon chirurgicale, sur une requête précise dont on aura d'abord diagnostiqué le plan défaillant à l'aide de `EXPLAIN` (§5.7) et de l'Optimizer Trace. On privilégiera toujours le traitement de la cause racine — statistiques, indexation, réécriture — avant de figer un hint.

Trois précautions complètent cette discipline. D'abord, un hint inscrit dans une requête peut se **périmer** : ce qui constitue le bon plan aujourd'hui, pour un volume et une distribution donnés, peut cesser de l'être demain ; il faut donc réexaminer périodiquement les hints en place. Ensuite, l'optimiseur peut légitimement **ignorer** un hint impossible à honorer : on en vérifiera toujours l'effet réel avec `EXPLAIN`. Enfin, pour une tendance **globale**, ce sont `optimizer_switch` ou l'ajustement des coûts (§15.14) qui conviennent ; le hint, lui, est l'outil de la requête isolée.

Les sous-sections qui suivent détaillent chaque famille, en commençant par le nommage des blocs de requête (§15.15.1) — le préalable à un ciblage précis des hints.

⏭️ [Nommage des blocs de requête (QB_NAME) et noms implicites](/15-performance-tuning/15.1-qb-name.md)

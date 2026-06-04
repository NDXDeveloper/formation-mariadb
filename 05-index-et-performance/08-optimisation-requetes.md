🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.8 — Optimisation des requêtes

> **Chapitre 5 — Index et Performance** · Section 5.8  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

À ce stade du chapitre, on sait **créer** des index ([5.4](04-creation-gestion-index.md)), choisir lesquels ([5.5](05-strategies-indexation.md)), les **ordonner** dans un composite ([5.6](06-index-composites.md)) et **lire** un plan d'exécution ([5.7](07-analyse-plans-execution.md)). Reste la dernière pièce : **écrire des requêtes que l'optimiseur saura optimiser**. Car un index, aussi bien conçu soit-il, n'apporte rien si la requête est formulée de façon à l'empêcher de l'utiliser, ou si le travail demandé est inutilement lourd.

Cette section pose les principes généraux de l'optimisation au **niveau de la requête**. Elle introduit le rôle de l'**optimiseur** et prépare la sous-section suivante, qui détaillera les mécanismes bas niveau que MariaDB applique pour accélérer l'accès aux index.

---

## L'optimiseur basé sur les coûts

Point fondamental, déjà entrevu : MariaDB **n'utilise pas un index par réflexe**. Son optimiseur est **basé sur les coûts** (*cost-based optimizer*) : pour une même requête, il évalue plusieurs plans d'exécution possibles, en **estime le coût** respectif, et retient le moins cher. Il peut donc **délibérément préférer un balayage complet** à un index, lorsque le premier lui paraît plus économique — typiquement sur une condition peu sélective ([5.2.1](02.1-btree.md)).

Ces estimations reposent sur des **statistiques** (cardinalités, distributions) qu'il faut tenir à jour : c'est le rôle d'`ANALYZE TABLE` ([chapitre 11](../11-administration-configuration/06.2-analyze-table.md)). Des statistiques périmées conduisent l'optimiseur à de mauvais choix — ce que la comparaison `rows` vs `r_rows` d'`ANALYZE` permet de détecter ([5.7](07-analyse-plans-execution.md)). Le modèle de coûts moderne de MariaDB tient par ailleurs compte des spécificités du matériel, notamment des **SSD** ([section 15.14](../15-performance-tuning/14-cost-based-optimizer-ssd.md)).

**Optimiser une requête, c'est donc largement aider l'optimiseur** à prendre la bonne décision : lui fournir des conditions exploitables, des statistiques justes, et le minimum de travail.

---

## Écrire des requêtes « index-friendly »

Une condition est dite **sargable** (de *Search ARGument ABLE*) lorsqu'elle peut s'appuyer sur un index. Les pièges détaillés en [section 5.5.1](05.1-index-colonnes-filtrees.md) se résument ici en un principe positif : **présenter la colonne indexée telle quelle**, d'un côté de la comparaison, sans la transformer.

```sql
-- ❌ Non sargable : la fonction masque la colonne à l'index
WHERE YEAR(date_commande) = 2026

-- ✅ Sargable : la colonne est comparée directement → l'index est utilisable
WHERE date_commande >= '2026-01-01' AND date_commande < '2027-01-01'
```

Le même réflexe vaut pour éviter les **conversions de type implicites** (concordance des types) et les **jokers en tête** (`LIKE '%x'`).

---

## Réduire le travail demandé

Au-delà des conditions, la **forme** de la requête détermine le volume de données manipulé :

- **Ne sélectionner que les colonnes utiles** — proscrire le `SELECT *` systématique. Cela réduit les données lues, transférées et mises en mémoire, et surtout cela **autorise les index couvrants** ([section 5.9](09-index-covering.md)) : si l'index contient toutes les colonnes demandées, MariaDB répond sans accéder à la table.

  ```sql
  -- ❌ ramène tout, empêche un index couvrant
  SELECT * FROM commandes WHERE client_id = 42;
  -- ✅ colonnes ciblées → potentiellement couvert par un index
  SELECT id, date_commande FROM commandes WHERE client_id = 42;
  ```

- **Limiter le résultat** avec `LIMIT`, en pagination — combiné à un index de tri, il permet de **s'arrêter tôt** ([section 5.5.3](05.3-index-order-group.md)).
- **Filtrer au plus tôt** : restreindre les lignes le plus en amont possible, avant les jointures et les agrégations coûteuses.

---

## Jointures : aider l'optimiseur

Pour une jointure, l'optimiseur choisit l'**ordre** des tables et la **méthode**. Le cas idéal est la jointure **par boucle indexée** : pour chaque ligne de la table pilote, retrouver les lignes correspondantes via un **index sur la colonne de jointure**. D'où l'importance d'indexer les **clés étrangères** et plus généralement toute colonne de jointure ([section 5.5.2](05.2-index-cles-etrangeres.md)).

Sans index sur la colonne jointe, MariaDB retombe sur des méthodes à base de tampon (jointures par bloc, par hachage) — visibles dans `EXPLAIN` via `Using join buffer` — généralement bien plus lentes. Deux leviers à retenir : **indexer les colonnes de jointure**, et faire en sorte que la **table pilote** (la première parcourue) soit la plus **petite** ou la plus **filtrée**.

---

## Sous-requêtes, tables dérivées et CTE

L'optimiseur sait **réécrire et optimiser** de nombreuses sous-requêtes (transformation en semi-jointure, matérialisation…). Toutes ne se valent cependant pas : une sous-requête **corrélée** (réévaluée pour chaque ligne de la requête englobante) peut être coûteuse, et une réécriture en **jointure** ou en **CTE** ([section 4.4](../04-concepts-avances-sql/04-expressions-table-communes.md)) améliore parfois nettement le plan. Le réflexe reste le même : comparer les plans avec `EXPLAIN`/`ANALYZE` avant de trancher.

---

## Quand l'optimiseur se trompe : les hints

L'optimiseur est généralement pertinent, mais il lui arrive de mal estimer un coût (statistiques trompeuses, requête atypique). On peut alors le **guider** par des *hints*. Les plus simples concernent le choix d'index :

```sql
SELECT * FROM commandes FORCE INDEX (idx_client) WHERE client_id = 42;
-- USE INDEX (suggère), FORCE INDEX (impose), IGNORE INDEX (exclut)
```

MariaDB 12.x dote par ailleurs le serveur d'un **moteur d'Optimizer Hints** plus complet, permettant un contrôle fin du plan d'exécution (ordre de jointure, méthodes, sous-requêtes…). C'est l'une des nouveautés majeures de la version, traitée en [section 15.15](../15-performance-tuning/15-optimizer-hints.md). Les *hints* restent toutefois un **dernier recours** : on corrige d'abord l'index ou les statistiques.

---

## Au-delà du plan : les optimisations du moteur

Une fois le plan choisi, MariaDB applique encore des **techniques internes** pour rendre l'accès aux index plus efficace : pousser les conditions au plus près de l'index (*Index Condition Pushdown*), pré-filtrer par identifiants de ligne (*Rowid Filtering*), ou parcourir l'index de façon clairsemée (*Loose Index Scan*). La série 12.x étend en outre ces optimisations aux **parcours inversés** (clés descendantes). Ces mécanismes — visibles dans `EXPLAIN` et déterminants pour les performances — font l'objet de la [section 5.8.1](08.1-optimisations-scans-inverses.md).

---

> ### 📝 À retenir  
>  
> - L'**optimiseur basé sur les coûts** choisit le plan le moins cher et peut **préférer un balayage** à un index ; il s'appuie sur des **statistiques** à tenir à jour (`ANALYZE TABLE`). Optimiser, c'est **l'aider** à bien décider.  
> - Écrire des conditions **sargables** : colonne présentée telle quelle, sans fonction, types concordants, pas de joker en tête.  
> - **Réduire le travail** : sélectionner les seules colonnes utiles (cela autorise les index **couvrants**, 5.9), utiliser `LIMIT`, filtrer au plus tôt.  
> - **Jointures** : indexer les colonnes de jointure (5.5.2) et piloter par la table la plus petite/filtrée ; `Using join buffer` signale un index manquant.  
> - Les **hints** (`USE`/`FORCE`/`IGNORE INDEX`, et le moteur d'Optimizer Hints de 15.15) ne sont qu'un **dernier recours**.  
> - MariaDB applique ensuite des **optimisations moteur** (ICP, Rowid Filtering, Loose Index Scan, étendues aux scans inversés en 12.x) détaillées en 5.8.1.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.7 Analyse des plans d'exécution](07-analyse-plans-execution.md)
- ➡️ Section suivante : [5.8.1 Optimisations sur scans inversés (Rowid Filtering, Index Condition Pushdown, Loose Index Scan sur clés DESC)](08.1-optimisations-scans-inverses.md) 🆕
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Optimisations sur scans inversés (Rowid Filtering, Index Condition Pushdown, Loose Index Scan sur clés DESC)](/05-index-et-performance/08.1-optimisations-scans-inverses.md)

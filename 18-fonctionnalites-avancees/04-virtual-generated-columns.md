🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.4 Colonnes virtuelles et générées

## Le principe : des colonnes calculées

Dans une table classique, chaque colonne contient une valeur que l'on insère et met à jour explicitement. Mais certaines données ne sont pas réellement *saisies* : elles se **déduisent** d'autres colonnes. Le montant total d'une ligne de commande est le produit de la quantité par le prix unitaire ; le nom complet d'une personne est la concaténation de son prénom et de son nom ; une ville peut être extraite d'un document JSON.

On pourrait stocker ces valeurs dérivées dans des colonnes ordinaires, mais il faudrait alors les maintenir à la main à chaque modification — au risque qu'elles divergent de la réalité. Les **colonnes générées** (*generated columns*) résolvent ce problème : leur valeur est définie par une **expression** que le serveur évalue automatiquement à partir des autres colonnes de la même ligne. La cohérence est garantie par construction, sans trigger ni code applicatif.

Le terme « colonne virtuelle » est souvent employé au sens large pour ces colonnes calculées ; il désigne plus précisément l'un de leurs deux modes de fonctionnement, présenté plus bas.

## Syntaxe de base

Une colonne générée se déclare avec le mot-clé `AS`, suivi de l'expression entre parenthèses. La forme standard `GENERATED ALWAYS AS` est acceptée, mais le `GENERATED ALWAYS` est facultatif :

```sql
CREATE TABLE commande (
  quantite   INT,
  prix_unite DECIMAL(10,2),
  total      DECIMAL(12,2) AS (quantite * prix_unite)
);
```

La colonne `total` n'est jamais renseignée directement : elle se calcule toute seule.

```sql
INSERT INTO commande (quantite, prix_unite) VALUES (3, 19.90);

SELECT total FROM commande;
-- 59.70
```

Tenter d'écrire une valeur dans une colonne générée provoque une erreur — c'est précisément ce qui en fait une donnée fiable :

```sql
INSERT INTO commande (quantite, prix_unite, total) VALUES (3, 19.90, 0);
-- ERREUR : impossible d'affecter une valeur à la colonne générée « total »
```

Une colonne générée peut aussi être ajoutée à une table existante avec `ALTER TABLE … ADD COLUMN … AS (…)`. On précise toujours son type, qui doit être cohérent avec celui produit par l'expression.

## Deux modes : `VIRTUAL` et `STORED`

Une colonne générée peut fonctionner de deux manières :

- en mode **`VIRTUAL`**, sa valeur est **recalculée à chaque lecture** et n'occupe aucune place sur le disque ;
- en mode **`STORED`** (également appelé `PERSISTENT` dans la syntaxe historique de MariaDB), sa valeur est **calculée à l'écriture** puis stockée comme une colonne ordinaire.

En l'absence de précision, le mode par défaut est `VIRTUAL`. Le choix entre les deux a des conséquences importantes sur l'espace occupé, les performances en lecture et en écriture, et les usages possibles. C'est l'objet de la sous-section suivante (§18.4.1).

## Ce que l'expression peut contenir

L'expression d'une colonne générée doit être **déterministe** : pour des mêmes valeurs d'entrée, elle doit toujours produire le même résultat. Cette exigence interdit notamment les fonctions non déterministes telles que `NOW()`, `CURRENT_TIMESTAMP`, `RAND()` ou `CONNECTION_ID()`.

L'expression peut référencer les **autres colonnes de la même ligne** — y compris une colonne générée déclarée précédemment —, des constantes et la plupart des fonctions intégrées déterministes. Elle ne peut en revanche pas faire appel à une sous-requête, à des colonnes d'une autre table, ni à des variables ou paramètres. Et, comme on l'a vu, on ne lui affecte jamais de valeur directement.

## À quoi servent les colonnes générées

Au-delà de la simple commodité d'une valeur dérivée toujours juste, les colonnes générées rendent plusieurs services :

- **Cohérence des données calculées** : totaux, sous-totaux, durées, âges calculés à partir d'une date de naissance, libellés composés (`CONCAT_WS(' ', prenom, nom)`)… autant de valeurs qui ne peuvent plus se désynchroniser.
- **Extraction et indexation de champs JSON** : une colonne générée peut extraire un attribut d'une colonne JSON (par exemple via l'opérateur `->>` vu en §4.7.3) pour ensuite l'indexer. C'est l'un des usages les plus courants, traité en détail en §4.10.
- **Normalisation pour la recherche** : exposer une forme transformée d'une donnée (e-mail en minuscules, valeur dénormalisée, empreinte calculée) afin d'accélérer ou de simplifier les recherches.
- **Indexation fonctionnelle** : indexer une colonne générée revient à indexer le *résultat d'un calcul*, ce qui ouvre des optimisations abordées en §18.4.2 — y compris, dans les versions récentes, pour les opérations de regroupement et de tri.

Ce dernier point illustre la complémentarité entre colonnes générées et index : la valeur calculée devient un objet indexable à part entière.

## Organisation de cette section

La suite approfondit le sujet en deux temps :

1. **[VIRTUAL vs STORED](04.1-virtual-vs-stored.md)** (§18.4.1) — comparer les deux modes (espace, performances, contraintes) et savoir lequel choisir.
2. **[Indexation de colonnes générées](04.2-indexation-colonnes-generees.md)** (§18.4.2) — indexer une colonne calculée, qu'elle soit virtuelle ou stockée, et exploiter ces index pour le filtrage, le regroupement et le tri.

⏭️ [VIRTUAL vs STORED](/18-fonctionnalites-avancees/04.1-virtual-vs-stored.md)

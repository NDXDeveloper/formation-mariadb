🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.7 Requêtes de sélection simples (SELECT, WHERE, ORDER BY, LIMIT)

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

`SELECT` est l'instruction la plus utilisée du SQL : c'est par elle que l'on **interroge** les données. Cette section en présente la forme simple, articulée autour de quatre clauses : `SELECT` choisit les colonnes, `WHERE` filtre les lignes, `ORDER BY` les trie et `LIMIT` en restreint le nombre. Les aspects plus avancés — agrégation, regroupement, jointures et sous-requêtes — sont traités au chapitre 3.

## `SELECT` : choisir les colonnes

La forme la plus simple liste les colonnes souhaitées, suivies de la table source :

```sql
SELECT nom, email FROM client;
```

L'étoile `*` sélectionne **toutes les colonnes**. Pratique pour explorer une table, elle est en revanche **déconseillée dans le code applicatif** : nommer explicitement les colonnes rend la requête plus claire, plus stable face aux évolutions du schéma et souvent plus performante.

```sql
SELECT * FROM client;   -- toutes les colonnes : utile pour explorer, à éviter en production
```

Les colonnes peuvent recevoir un **alias** (mot-clé `AS`, facultatif) et la sélection peut contenir des **expressions** :

```sql
SELECT nom, prix * quantite AS total
FROM ligne_commande;
```

Le mot-clé `DISTINCT` élimine les lignes en double du résultat :

```sql
SELECT DISTINCT pays FROM client;   -- chaque pays une seule fois
```

Enfin, `SELECT` peut s'employer **sans `FROM`** pour évaluer des constantes ou des fonctions :

```sql
SELECT NOW(), 'Bonjour', 1 + 1;
```

## `WHERE` : filtrer les lignes

La clause `WHERE` ne conserve que les lignes satisfaisant une **condition**. Les opérateurs de comparaison habituels sont disponibles (`=`, `<>` ou `!=`, `<`, `>`, `<=`, `>=`), combinables avec les opérateurs logiques `AND`, `OR` et `NOT` :

```sql
SELECT nom, pays FROM client
WHERE pays = 'FR' AND actif = TRUE;
```

Plusieurs opérateurs facilitent l'écriture des conditions courantes. `BETWEEN` teste l'appartenance à un intervalle (bornes incluses), `IN` l'appartenance à une liste, et `LIKE` la correspondance à un motif — où `%` remplace une suite quelconque de caractères (éventuellement vide) et `_` exactement un caractère :

```sql
SELECT nom, prix FROM produit WHERE prix BETWEEN 10 AND 50;
SELECT nom FROM client       WHERE pays IN ('FR', 'BE', 'CH');
SELECT nom FROM client       WHERE email LIKE '%@example.com';
```

Le cas des valeurs `NULL` est particulier : `NULL` ne peut **jamais** être comparé avec `=`. Pour tester l'absence de valeur, on utilise `IS NULL` ou `IS NOT NULL` (la logique des `NULL` est détaillée en §4.6) :

```sql
SELECT nom FROM client WHERE telephone IS NULL;   -- et non "= NULL"
```

Deux points de vigilance. D'abord, l'opérateur `AND` est **prioritaire** sur `OR` : en cas de doute, des **parenthèses** lèvent toute ambiguïté. Ensuite, la comparaison de chaînes dépend de la **collation** (§2.2.2) : avec la collation par défaut, insensible à la casse, `WHERE nom = 'dupont'` retrouvera aussi `'Dupont'`.

```sql
SELECT nom FROM client
WHERE pays = 'FR' AND (statut = 'actif' OR statut = 'premium');
```

## `ORDER BY` : trier les résultats

`ORDER BY` trie le résultat selon une ou plusieurs colonnes, en ordre croissant (`ASC`, par défaut) ou décroissant (`DESC`). En présence de plusieurs critères, le tri s'applique de gauche à droite :

```sql
SELECT nom, prix FROM produit
ORDER BY prix DESC;                  -- du plus cher au moins cher

SELECT nom, pays, prix FROM produit
ORDER BY pays ASC, prix DESC;        -- d'abord par pays, puis par prix décroissant
```

On peut trier sur un alias ou une expression. Concernant les `NULL`, MariaDB les considère comme inférieurs à toute autre valeur : ils apparaissent donc en **premier** en tri croissant et en **dernier** en tri décroissant. Pour forcer leur position, un idiome courant consiste à trier d'abord sur un test `IS NULL` :

```sql
SELECT nom, date_livraison FROM commande
ORDER BY date_livraison IS NULL, date_livraison;   -- place les NULL à la fin
```

## `LIMIT` : limiter le nombre de lignes

`LIMIT` restreint le nombre de lignes renvoyées — par exemple pour n'obtenir que le « top N » :

```sql
SELECT nom, prix FROM produit
ORDER BY prix DESC
LIMIT 5;                             -- les 5 produits les plus chers
```

`LIMIT` sert aussi à la **pagination**, à l'aide d'un décalage (*offset*). Attention à la forme à deux arguments, où l'**offset vient en premier** : `LIMIT 40, 20` saute 40 lignes puis en renvoie 20. La forme `LIMIT 20 OFFSET 40`, équivalente, est plus lisible :

```sql
SELECT nom FROM produit ORDER BY id LIMIT 40, 20;       -- LIMIT offset, nombre
SELECT nom FROM produit ORDER BY id LIMIT 20 OFFSET 40; -- forme équivalente
```

Point essentiel : sans `ORDER BY`, l'ordre des lignes **n'est pas garanti**, et un `LIMIT` renverrait alors un sous-ensemble arbitraire. On associe donc presque toujours `LIMIT` à un `ORDER BY` pour obtenir un résultat déterministe.

## L'ordre logique d'évaluation

Bien que l'on écrive les clauses dans l'ordre `SELECT … FROM … WHERE … ORDER BY … LIMIT`, MariaDB les évalue logiquement dans un autre ordre : d'abord `FROM`, puis `WHERE`, puis `SELECT` (où les alias et expressions sont calculés), puis `ORDER BY`, et enfin `LIMIT`. Cela explique une règle qui surprend souvent les débutants : on **peut** réutiliser un alias défini dans `SELECT` au sein d'`ORDER BY` (évalué après), mais **pas** dans `WHERE` (évalué avant). Dans `WHERE`, il faut répéter l'expression complète.

## En résumé

Une requête de sélection simple combine quatre clauses : `SELECT` choisit les colonnes (en privilégiant des noms explicites plutôt que `*`, avec alias et expressions au besoin, et `DISTINCT` pour dédupliquer) ; `WHERE` filtre les lignes (opérateurs de comparaison et logiques, `BETWEEN`, `IN`, `LIKE`, `IS NULL`, en pensant aux parenthèses et à la collation) ; `ORDER BY` trie le résultat (`ASC`/`DESC`, plusieurs critères) ; et `LIMIT` en restreint le nombre, idéalement couplé à un `ORDER BY` pour la pagination et le « top N ». Garder en tête l'ordre logique d'évaluation éclaire l'usage des alias et prépare les requêtes plus riches du chapitre 3.

---

← Section précédente : [2.6 Insertion de données](06-insertion-donnees.md) · [Sommaire du chapitre](README.md) · Section suivante : [2.8 Mise à jour et suppression de données](08-mise-a-jour-suppression.md) →

⏭️ [Mise à jour et suppression de données (UPDATE, DELETE, TRUNCATE)](/02-bases-du-sql/08-mise-a-jour-suppression.md)

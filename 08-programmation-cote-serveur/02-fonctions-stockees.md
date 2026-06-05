🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.2 Fonctions stockées

Une *fonction stockée* est une routine nommée qui calcule et **retourne une valeur unique**. Sa particularité, par rapport à la procédure, est de s'utiliser directement au sein d'une expression SQL — dans une liste `SELECT`, une clause `WHERE`, un `ORDER BY` — exactement comme une fonction native telle que `UPPER()` ou `ROUND()`. Là où une procédure *exécute une action*, une fonction *produit un résultat* que la requête appelante exploite aussitôt.

Comme les procédures, les fonctions s'écrivent dans le langage procédural de MariaDB et sont conservées dans la base ; elles partagent avec elles l'essentiel du corps (`BEGIN … END`, variables, contrôle de flux, gestion d'erreurs). Ce qui les distingue tient à leur vocation : renvoyer une valeur, et le faire d'une manière compatible avec son emploi dans des requêtes.

## Fonction ou procédure ?

La distinction esquissée en 8.1 mérite d'être précisée du point de vue des fonctions. Une fonction se reconnaît à plusieurs traits propres :

- elle déclare un **type de retour** (clause `RETURNS`) et renvoie sa valeur par l'instruction `RETURN` ;
- ses paramètres sont **uniquement en entrée** : les modes `OUT` et `INOUT` n'existent pas pour une fonction (tous les paramètres sont implicitement `IN`) ;
- elle **ne peut pas renvoyer de jeu de résultats**, ni plus largement exécuter d'instructions qui retourneraient des lignes au client ;
- elle **ne peut pas être récursive** (voir 8.1.3) ;
- elle s'appelle en l'**insérant dans une expression**, sans `CALL`.

À l'inverse, une procédure s'invoque par `CALL`, accepte les trois modes de paramètres et peut renvoyer un ou plusieurs jeux de résultats, mais ne s'utilise pas dans une expression. En pratique : on écrit une fonction lorsqu'on a besoin d'une *valeur* réutilisable dans des requêtes, une procédure lorsqu'on orchestre un *traitement*.

## Utiliser une fonction dans une requête

L'atout majeur d'une fonction est de s'intégrer partout où une valeur est attendue. Une même fonction métier peut ainsi enrichir une projection, filtrer des lignes et trier un résultat :

```sql
-- en supposant une fonction prix_ttc(montant_ht) définie par ailleurs
SELECT id, nom, prix_ttc(prix) AS prix_ttc
  FROM produits
 WHERE prix_ttc(prix) > 100
 ORDER BY prix_ttc(prix) DESC;
```

Cette commodité a une contrepartie : dans une requête, la fonction peut être évaluée **un grand nombre de fois** — potentiellement une fois par ligne examinée. Son coût se répercute donc directement sur celui de la requête, ce qui rend la performance et le caractère déterministe d'une fonction particulièrement importants (voir 8.2.2). La définition de `prix_ttc` elle-même est présentée en 8.2.1.

## Caractéristiques et restrictions

Plusieurs éléments encadrent l'écriture d'une fonction et seront détaillés dans les sous-sections. La fonction doit annoncer son type de retour (`RETURNS`) et garantir qu'un `RETURN` est atteint à l'exécution. Elle se voit aussi attribuer des **caractéristiques** — `DETERMINISTIC` ou `NOT DETERMINISTIC`, et un niveau d'accès aux données (`NO SQL`, `READS SQL DATA`, `MODIFIES SQL DATA`, `CONTAINS SQL`) — qui, contrairement aux procédures, jouent ici un rôle concret : l'optimiseur et la journalisation binaire s'en servent.

Ce dernier point mérite une vigilance particulière. Lorsque la journalisation binaire est active (réplication, sauvegarde incrémentale), la création d'une fonction est soumise à condition : la fonction doit être déclarée avec des caractéristiques sûres (par exemple `DETERMINISTIC`, `NO SQL` ou `READS SQL DATA`), faute de quoi un privilège adapté ou l'activation de `log_bin_trust_function_creators` devient nécessaire. Les raisons et les réglages associés sont expliqués en 8.2.2.

## Fonctions stockées, fonctions natives et UDF

Le terme « fonction » recouvre plusieurs réalités qu'il est utile de distinguer. Les **fonctions natives** (`UPPER`, `NOW`, `COALESCE`…) sont intégrées au serveur. Les **fonctions stockées**, objet de cette section, sont écrites en SQL et conservées dans la base. Les **UDF** (*User-Defined Functions*) sont, elles, des fonctions compilées en C/C++ et chargées sous forme de bibliothèque (`CREATE FUNCTION … SONAME`) ; elles relèvent d'un mécanisme distinct, plus rarement employé. MariaDB permet par ailleurs de définir des **fonctions d'agrégation** personnalisées (`CREATE AGGREGATE FUNCTION`), une variante des fonctions stockées destinée à être utilisée avec `GROUP BY`.

## Points communs avec les procédures

Pour le reste, fonctions et procédures fonctionnent de la même manière, et les notions vues en 8.1 s'appliquent telles quelles : elles appartiennent à une base, se consultent via `INFORMATION_SCHEMA.ROUTINES`, `SHOW FUNCTION STATUS` et `SHOW CREATE FUNCTION`, s'exécutent dans le contexte de privilèges fixé par `DEFINER` / `SQL SECURITY`, et s'écrivent aussi bien en syntaxe historique qu'en mode Oracle. Il est inutile d'y revenir en détail ici.

## Plan de la section

- **8.2.1 — Syntaxe `CREATE FUNCTION`** : la définition d'une fonction, de la clause `RETURNS` à l'instruction `RETURN`.
- **8.2.2 — Caractéristiques (`DETERMINISTIC`, `NO SQL`, etc.)** : le rôle de chaque caractéristique et son incidence sur l'optimiseur et la journalisation binaire.

---

Commençons par la définition d'une fonction : la syntaxe de **[`CREATE FUNCTION`](02.1-syntaxe-create-function.md)**.

⏭️ [Syntaxe CREATE FUNCTION](/08-programmation-cote-serveur/02.1-syntaxe-create-function.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 Requêtes pivotées et transformations

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Référence : **MariaDB 12.3 LTS**

**Pivoter** des données, c'est transformer une présentation « en hauteur » (*tall*, une ligne par observation) en une présentation « en largeur » (*wide*, une matrice). On veut par exemple, à partir d'une table où chaque ligne est une vente d'un trimestre dans une région, obtenir un tableau croisé : une ligne par région, une colonne par trimestre. C'est l'opération typique des rapports et tableaux de bord. Cette section présente comment la réaliser en MariaDB — qui, contrairement à certains SGBD, ne dispose pas d'opérateur dédié — ainsi que l'opération inverse et quelques transformations apparentées.

## 🎯 Objectif de la section

Savoir pivoter des données par agrégation conditionnelle, choisir la bonne fonction d'agrégation, gérer le cas des colonnes inconnues à l'avance (pivot dynamique), réaliser l'opération inverse (*unpivot*), et connaître les transformations connexes (`GROUP_CONCAT`, `WITH ROLLUP`).

## Le problème du pivot

Considérons une table `ventes` au format « en hauteur » :

| region | trimestre | montant |
|--------|-----------|--------:|
| Nord   | T1        | 100     |
| Nord   | T2        | 150     |
| Sud    | T1        | 200     |
| Sud    | T2        | 120     |

L'objectif est d'obtenir une matrice région × trimestre :

| region | T1  | T2  | T3  | T4  |
|--------|----:|----:|----:|----:|
| Nord   | 100 | 150 | …   | …   |
| Sud    | 200 | 120 | …   | …   |

Ce format de stockage « en hauteur » est très courant : c'est notamment celui du modèle **entité-attribut-valeur** (EAV), où chaque attribut occupe sa propre ligne. Reconnaître qu'un besoin de restitution est en réalité un problème de pivot est souvent la première difficulté.

## L'absence d'opérateur `PIVOT` natif

Certains SGBD (SQL Server, Oracle) offrent des clauses `PIVOT` et `UNPIVOT`. **Ce n'est pas le cas de MariaDB** (ni de MySQL ou PostgreSQL). Le pivot s'y réalise donc « à la main », principalement par **agrégation conditionnelle**. C'est moins concis, mais parfaitement efficace et portable.

## Le pivot par agrégation conditionnelle

La technique de référence consiste à regrouper sur la dimension qui formera les **lignes**, puis à produire une colonne par valeur cible à l'aide d'une expression conditionnelle placée à l'intérieur d'une fonction d'agrégation :

```sql
SELECT region,
       SUM(CASE WHEN trimestre = 'T1' THEN montant END) AS T1,
       SUM(CASE WHEN trimestre = 'T2' THEN montant END) AS T2,
       SUM(CASE WHEN trimestre = 'T3' THEN montant END) AS T3,
       SUM(CASE WHEN trimestre = 'T4' THEN montant END) AS T4
FROM ventes
GROUP BY region;
```

Le mécanisme est simple : `GROUP BY region` produit une ligne par région ; pour chaque colonne, l'expression `CASE WHEN trimestre = 'T1' THEN montant END` renvoie le montant lorsque la ligne concerne T1, et `NULL` sinon (le `ELSE` implicite). Le `SUM` ignorant les `NULL`, il ne totalise que les montants du trimestre visé.

MariaDB propose aussi la fonction `IF()`, plus courte, en remplacement du `CASE` à deux branches :

```sql
SUM(IF(trimestre = 'T1', montant, 0)) AS T1
```

> 💡 **`NULL` ou `0` pour les cellules vides ?** Avec `CASE … END` (sans `ELSE`), une cellule sans donnée vaut `NULL` (case blanche). Avec `IF(cond, montant, 0)`, elle vaut `0`. Choisissez selon l'effet voulu dans le rapport : une absence de vente affichée comme vide, ou comme zéro.

## Choisir la fonction d'agrégation

La fonction qui enveloppe le `CASE` dépend de la nature de la cellule :

- **`SUM`** lorsque la cellule agrège plusieurs valeurs additives (un total de ventes) ;
- **`COUNT`** pour compter les occurrences, par exemple `COUNT(CASE WHEN trimestre = 'T1' THEN 1 END)` ;
- **`MAX`** (ou `MIN`) lorsque chaque cellule ne contient **qu'une seule valeur** à reporter telle quelle. C'est le cas du pivot d'un modèle EAV, et `MAX`/`MIN` présentent l'avantage de fonctionner aussi sur des **chaînes de caractères**, ce que `SUM` ne permet pas.

Exemple de pivot EAV, où chaque entité possède au plus une valeur par attribut :

```sql
SELECT entite,
       MAX(CASE WHEN attribut = 'couleur' THEN valeur END) AS couleur,
       MAX(CASE WHEN attribut = 'taille'  THEN valeur END) AS taille
FROM attributs
GROUP BY entite;
```

Ici, `MAX` se contente de remonter l'unique valeur textuelle de chaque attribut.

## Pivot dynamique : colonnes inconnues à l'avance

La limite de l'agrégation conditionnelle est que **SQL exige de nommer les colonnes du résultat à l'écriture de la requête**. Si la liste des valeurs à pivoter change (un nouveau trimestre, de nouvelles catégories), la requête doit être réécrite.

La parade consiste à **générer dynamiquement** le texte de la requête, puis à l'exécuter via une *prepared statement*. On construit d'abord la liste des expressions de colonnes avec `GROUP_CONCAT`, à partir des valeurs distinctes présentes dans les données :

```sql
SELECT GROUP_CONCAT(DISTINCT
         CONCAT('SUM(CASE WHEN trimestre = ''', trimestre,
                ''' THEN montant END) AS `', trimestre, '`')
       ) INTO @colonnes
FROM ventes;

SET @sql = CONCAT('SELECT region, ', @colonnes,
                  ' FROM ventes GROUP BY region');

PREPARE requete FROM @sql;
EXECUTE requete;
DEALLOCATE PREPARE requete;
```

La première instruction assemble, pour chaque trimestre distinct, l'expression `SUM(CASE … )` correspondante ; la deuxième compose la requête complète ; les suivantes la préparent et l'exécutent. On note l'attention portée au *quoting* : guillemets simples doublés pour les littéraux, accents graves pour les identifiants de colonnes.

Ce schéma s'inscrit logiquement dans une **procédure stockée** (§ 8.1), réutilisable avec différents paramètres ; la documentation officielle de MariaDB en fournit d'ailleurs un exemple générique. Les *prepared statements* sont détaillées au § 17.9. Pour de nombreuses colonnes, pensez à relever au besoin la limite de longueur de `GROUP_CONCAT` (variable `group_concat_max_len`).

## L'opération inverse : « unpivot »

Transformer des **colonnes en lignes** (passer du format large au format en hauteur) n'a pas non plus d'opérateur dédié. On l'obtient en réunissant, par `UNION ALL`, une requête par colonne source :

```sql
SELECT region, 'T1' AS trimestre, T1 AS montant FROM ventes_pivot
UNION ALL
SELECT region, 'T2', T2 FROM ventes_pivot
UNION ALL
SELECT region, 'T3', T3 FROM ventes_pivot
UNION ALL
SELECT region, 'T4', T4 FROM ventes_pivot;
```

Chaque `SELECT` extrait une colonne et l'étiquette ; leur union reconstitue une ligne par couple (région, trimestre).

## Alternative : le moteur CONNECT

Pour qui souhaite éviter l'agrégation conditionnelle, le moteur de stockage **CONNECT** propose un type de table `PIVOT` : une table virtuelle qui réalise automatiquement le pivot d'une table source au moment de l'interrogation. C'est une solution au niveau moteur, utile pour des pivots récurrents. Le moteur CONNECT est présenté au § 7.10.4.

## Transformations connexes

Deux autres outils relèvent de la même logique de transformation des résultats.

**`GROUP_CONCAT`** agrège plusieurs lignes en une seule chaîne délimitée, par groupe — une forme de pivot « ligne vers texte » :

```sql
SELECT region,
       GROUP_CONCAT(produit ORDER BY produit SEPARATOR ', ') AS produits
FROM ventes
GROUP BY region;
```

On peut ordonner les éléments (`ORDER BY` interne) et choisir le séparateur (`SEPARATOR`).

**`WITH ROLLUP`**, extension de `GROUP BY`, ajoute des lignes de **super-agrégats** (sous-totaux et total général) où les colonnes de regroupement passent à `NULL` :

```sql
SELECT region, trimestre, SUM(montant)
FROM ventes
GROUP BY region, trimestre WITH ROLLUP;
```

Pratique pour adjoindre totaux et sous-totaux à un tableau, souvent en complément d'un pivot.

## Points clés à retenir

MariaDB n'offre **pas d'opérateur `PIVOT`/`UNPIVOT`** : le pivot se réalise par **agrégation conditionnelle** — un `GROUP BY` sur la dimension des lignes et une `SUM`/`COUNT`/`MAX(CASE WHEN … END)` par colonne cible. Le choix de l'agrégat dépend de la cellule : `SUM` pour des totaux, `COUNT` pour des dénombrements, `MAX`/`MIN` pour reporter une valeur unique (y compris textuelle). Lorsque les colonnes ne sont pas connues d'avance, on génère la requête dynamiquement via `GROUP_CONCAT` et une *prepared statement*, idéalement dans une procédure stockée. L'*unpivot* s'obtient par `UNION ALL` d'un `SELECT` par colonne. Enfin, le moteur CONNECT, `GROUP_CONCAT` et `WITH ROLLUP` complètent la palette des transformations.

---

**Section précédente :** [4.2.4 — Cas d'usage : Top N, moyenne mobile, cumuls](02.4-cas-usage-window.md)  
**Section suivante :** [4.4 — Expressions de table communes (CTE)](04-expressions-table-communes.md)  

⏭️ [Expressions de table communes (CTE)](/04-concepts-avances-sql/04-expressions-table-communes.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 Window Functions

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Disponibles depuis MariaDB 10.2 · Référence : **MariaDB 12.3 LTS**

Les **fonctions de fenêtrage** (*window functions*, ou fonctions analytiques) comptent parmi les outils SQL les plus puissants pour l'analyse de données. Elles répondent à une frustration récurrente : avec un simple `GROUP BY`, calculer une statistique de groupe **fait disparaître le détail des lignes**. Pour comparer chaque valeur à la moyenne de son groupe, produire un classement ou un cumul, il fallait alors recourir à des sous-requêtes corrélées ou à des auto-jointures, souvent lourdes et coûteuses.

Une fonction de fenêtrage calcule une valeur pour **chaque ligne**, à partir d'un ensemble de lignes qui lui sont liées — sa *fenêtre* — **sans regrouper ni supprimer** ces lignes. Détail et agrégat coexistent ainsi dans le même résultat.

## 🎯 Objectif de la section

Comprendre ce qui distingue une fonction de fenêtrage d'une agrégation classique, maîtriser l'anatomie de la clause `OVER`, identifier les grandes familles de fonctions analytiques et connaître les règles qui régissent leur emploi. Les sous-sections suivantes détailleront ensuite chaque famille.

## L'idée : calculer sans agréger

Comparons deux requêtes sur une table `employes`.

Avec une agrégation classique, on obtient **une ligne par département** ; le détail des employés est perdu :

```sql
SELECT departement, AVG(salaire) AS moyenne_dept
FROM employes
GROUP BY departement;
```

Avec une fonction de fenêtrage, **chaque employé est conservé**, et la moyenne de son département vient s'ajouter en colonne :

```sql
SELECT nom, departement, salaire,
       AVG(salaire) OVER (PARTITION BY departement) AS moyenne_dept
FROM employes;
```

Il devient immédiat de comparer chaque salaire à la moyenne de son service — `salaire - moyenne_dept` — sans aucune jointure ni sous-requête. C'est là tout l'intérêt du fenêtrage : associer, sur la même ligne, une donnée individuelle et une statistique calculée sur un groupe de lignes voisines.

## Anatomie de la clause `OVER`

Toute fonction de fenêtrage se reconnaît à la clause `OVER`, qui décrit la fenêtre sur laquelle le calcul s'effectue :

```sql
fonction([arguments]) OVER (
    [ PARTITION BY expression[, ...] ]
    [ ORDER BY    expression [ASC|DESC][, ...] ]
    [ clause_de_frame ]
)
```

La clause `OVER` comporte trois composants, tous facultatifs :

**`PARTITION BY`** découpe les lignes en *partitions* ; la fonction est évaluée indépendamment dans chacune. Le rôle est comparable à celui de `GROUP BY`, mais sans collapse des lignes. En son absence, l'ensemble du résultat forme une seule et même partition.

**`ORDER BY`** ordonne les lignes **à l'intérieur de chaque partition**. Cet ordre est indispensable pour toute fonction dont le résultat dépend d'une position : un classement, un cumul, ou l'accès à la ligne précédente ou suivante. Sans lui, parler de « ligne précédente » n'aurait pas de sens.

**La clause de frame** restreint, pour la ligne courante, le sous-ensemble de la partition réellement pris en compte (par exemple « les trois lignes précédentes »). Elle s'exprime via les unités `ROWS`, `RANGE` et `GROUPS`, étudiées en détail au § 4.2.3.

> ⚠️ **À savoir dès maintenant.** Lorsqu'un `ORDER BY` est présent sans clause de frame explicite, une frame **par défaut** s'applique : la fenêtre s'étend du début de la partition jusqu'à la ligne courante. C'est ce qui transforme un `SUM(...) OVER (ORDER BY ...)` en **cumul progressif**. Cette mécanique, source de surprises classiques (notamment avec `LAST_VALUE`), est expliquée en § 4.2.3.

## Les grandes familles de fonctions

On distingue trois familles de fonctions utilisables avec `OVER`.

Les **fonctions d'agrégation employées comme fonctions de fenêtrage** : `SUM`, `AVG`, `COUNT`, `MIN`, `MAX`… Ce sont les agrégats habituels, mais appliqués à une fenêtre plutôt qu'à un groupe. Ils servent aux moyennes par partition, aux cumuls et aux moyennes mobiles.

Les **fonctions de rang**, qui attribuent une position aux lignes ordonnées : `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, ainsi que `PERCENT_RANK` et `CUME_DIST`. Elles font l'objet du **§ 4.2.1**.

Les **fonctions de valeur (ou de navigation)**, qui accèdent à la valeur d'une autre ligne de la fenêtre : `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`. Elles sont traitées au **§ 4.2.2**.

## Où peut-on utiliser une fonction de fenêtrage ?

C'est un point essentiel et souvent mal compris. Une fonction de fenêtrage ne peut apparaître que dans la liste du `SELECT` et dans la clause `ORDER BY` finale. Elle est **interdite** dans `WHERE`, `GROUP BY` et `HAVING`.

La raison tient à l'**ordre logique d'évaluation** d'une requête : `FROM` → `WHERE` → `GROUP BY` → `HAVING` → **fonctions de fenêtrage** → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`. Les fonctions de fenêtrage sont calculées *après* le filtrage et le regroupement ; elles ne peuvent donc pas servir de critère dans ces étapes antérieures.

Pour **filtrer sur le résultat** d'une fonction de fenêtrage — par exemple ne garder que les trois meilleurs par groupe — on calcule la valeur dans une sous-requête (ou une CTE), puis on filtre dans la requête englobante :

```sql
SELECT *
FROM (
    SELECT nom, departement, salaire,
           ROW_NUMBER() OVER (PARTITION BY departement
                              ORDER BY salaire DESC) AS rang
    FROM employes
) AS classement
WHERE rang <= 3;
```

Ce motif « calcul de fenêtrage en sous-requête, filtrage à l'extérieur » est fondamental ; il est repris et approfondi dans les § 4.2.1 et § 4.2.4.

## Fenêtres nommées : la clause `WINDOW`

Lorsque plusieurs fonctions partagent la **même** définition de fenêtre, répéter la clause `OVER (...)` devient verbeux et source d'erreurs. La clause `WINDOW` permet de nommer une fenêtre une fois pour la réutiliser :

```sql
SELECT date_vente, region, montant,
       SUM(montant) OVER w AS cumul,
       COUNT(*)     OVER w AS nb_ventes
FROM ventes
WINDOW w AS (PARTITION BY region ORDER BY date_vente);
```

Ici, `cumul` et `nb_ventes` s'appuient sur la même fenêtre ordonnée par région : le code est plus concis et la cohérence garantie.

## 🗺️ Plan de la section

La suite décompose le sujet en quatre volets :

- **4.2.1 — Fonctions de rang** (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`) : numéroter et classer les lignes.
- **4.2.2 — Fonctions de valeur** (`LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`) : comparer une ligne à ses voisines.
- **4.2.3 — Frames de fenêtre** (`ROWS`, `RANGE`, `GROUPS`) : maîtriser précisément l'étendue du calcul.
- **4.2.4 — Cas d'usage** : Top N par groupe, moyenne mobile, cumuls et autres motifs analytiques courants.

## Points clés à retenir

Une fonction de fenêtrage calcule une valeur **par ligne** à partir d'une fenêtre de lignes liées, **sans agréger** : c'est sa différence majeure avec `GROUP BY`. La clause `OVER` en définit la portée via `PARTITION BY` (découpage), `ORDER BY` (ordre interne) et une éventuelle clause de frame (étendue). Trois familles cohabitent : agrégats fenêtrés, fonctions de rang et fonctions de valeur. Ces fonctions ne s'utilisent que dans `SELECT` et l'`ORDER BY` final ; pour filtrer sur leur résultat, on les enveloppe dans une sous-requête ou une CTE. Enfin, la clause `WINDOW` factorise les définitions de fenêtre répétées.

---

**Section précédente :** [4.1 — Requêtes récursives (`WITH RECURSIVE`)](01-requetes-recursives.md)  
**Section suivante :** [4.2.1 — Fonctions de rang (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`)](02.1-fonctions-rang.md)  

⏭️ [Fonctions de rang (ROW_NUMBER, RANK, DENSE_RANK, NTILE)](/04-concepts-avances-sql/02.1-fonctions-rang.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 Expressions de table communes (CTE)

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Disponibles depuis MariaDB 10.2.1 (forme récursive : 10.2.2) · Référence : **MariaDB 12.3 LTS**

Une **expression de table commune** — *Common Table Expression*, ou **CTE** — est un ensemble de résultats temporaire, nommé, qui n'existe que le temps d'une requête. Introduite par le mot-clé `WITH`, elle sert avant tout à **structurer et clarifier** des requêtes complexes en les décomposant en étapes lisibles, et à **réutiliser** un sous-résultat sans le réécrire.

Nous avons déjà rencontré la forme *récursive* de la CTE (`WITH RECURSIVE`, § 4.1), dédiée aux structures hiérarchiques. Cette section traite de la CTE **non récursive** et des principes communs à toutes les CTE.

## 🎯 Objectif de la section

Comprendre ce qu'est une CTE et sa syntaxe, savoir pourquoi et quand l'employer, enchaîner plusieurs CTE, et la situer par rapport aux sous-requêtes dérivées et aux vues.

## Qu'est-ce qu'une CTE ?

Une CTE nomme le résultat d'une sous-requête, que la requête principale peut ensuite utiliser comme s'il s'agissait d'une table. Prenons un cas simple : isoler les « gros clients », puis récupérer leur nom.

```sql
WITH gros_clients AS (
    SELECT client_id, SUM(montant) AS total
    FROM commandes
    GROUP BY client_id
    HAVING SUM(montant) > 10000
)
SELECT c.nom, g.total
FROM gros_clients AS g
JOIN clients AS c ON c.id = g.client_id;
```

La lecture se fait de haut en bas, dans l'ordre du raisonnement : « voici l'ensemble des gros clients ; à présent, joignons-le à la table des clients pour obtenir leurs noms ». La CTE `gros_clients` disparaît dès la requête terminée.

## Syntaxe

La forme générale est la suivante :

```sql
WITH nom_cte [(colonne1, colonne2, …)] AS (
    sous_requête
)
[, autre_cte AS ( … )]
requête_principale;
```

La **liste de colonnes** entre parenthèses est facultative : en son absence, les colonnes prennent les noms issus du `SELECT` de la sous-requête. La préciser explicitement clarifie l'intention :

```sql
WITH stats(region, total, nb) AS (
    SELECT region, SUM(montant), COUNT(*)
    FROM ventes
    GROUP BY region
)
SELECT * FROM stats;
```

Une même clause `WITH` peut définir **plusieurs CTE**, séparées par des virgules — ce qui nous mène à leur enchaînement.

## Enchaîner plusieurs CTE

Une CTE peut référencer une CTE définie **avant elle** dans la même clause `WITH`. On construit ainsi un raisonnement par paliers. Pour ne retenir, par exemple, que les régions dont le chiffre dépasse la moyenne régionale :

```sql
WITH totaux_region AS (
    SELECT region, SUM(montant) AS total
    FROM ventes
    GROUP BY region
),
reference AS (
    SELECT AVG(total) AS moyenne
    FROM totaux_region          -- référence la CTE précédente
)
SELECT t.region, t.total
FROM totaux_region AS t
CROSS JOIN reference AS r
WHERE t.total > r.moyenne;
```

Deux choses méritent attention. D'une part, `reference` s'appuie sur `totaux_region` : c'est l'enchaînement. D'autre part, `totaux_region` est **utilisée deux fois** — dans `reference` et dans la requête principale — sans être réécrite : c'est l'un des atouts majeurs des CTE.

## Pourquoi utiliser une CTE ?

Les bénéfices d'une CTE sont essentiellement de l'ordre de la **clarté** et de la **réutilisation** :

- **Lisibilité** : une requête complexe se décompose en étapes nommées, lues de haut en bas, au lieu d'un empilement de sous-requêtes imbriquées difficile à suivre.
- **Réutilisation au sein de la requête** : une CTE peut être référencée plusieurs fois, là où une sous-requête devrait être recopiée à chaque emploi.
- **Documentation implicite** : le nom de la CTE (`gros_clients`, `totaux_region`) exprime l'intention, ce qu'une sous-requête anonyme ne fait pas.
- **Récursion** : la forme `WITH RECURSIVE` (§ 4.1) ouvre le traitement des hiérarchies, impossible autrement en une seule requête.

## CTE, sous-requête dérivée ou vue ?

Sur le plan du résultat, une CTE non récursive équivaut souvent à une **sous-requête dérivée** (une sous-requête placée dans le `FROM`). La même requête « gros clients » s'écrirait :

```sql
SELECT c.nom, g.total
FROM (
    SELECT client_id, SUM(montant) AS total
    FROM commandes
    GROUP BY client_id
    HAVING SUM(montant) > 10000
) AS g
JOIN clients AS c ON c.id = g.client_id;
```

Le résultat est identique, mais la version CTE se lit plus linéairement, surtout lorsque l'imbrication s'approfondit. Trois constructions sont à distinguer :

| Construction          | Portée                         | Réutilisable          | Usage typique                                   |
|-----------------------|--------------------------------|-----------------------|-------------------------------------------------|
| Sous-requête dérivée  | locale, anonyme                | non (recopier)        | étape unique, simple                            |
| **CTE** (`WITH`)      | locale à la requête, nommée    | oui, dans la requête  | décomposer/clarifier une requête, réutiliser    |
| Vue (`CREATE VIEW`)   | persistante (objet de schéma)  | oui, entre requêtes   | logique réutilisée par plusieurs requêtes       |

En résumé : une sous-requête dérivée pour une étape ponctuelle, une **CTE** pour structurer une requête et y réutiliser un sous-résultat, une **vue** (chapitre 9) lorsque la logique doit servir à plusieurs requêtes dans le temps.

## Note sur l'exécution

Une CTE est un outil de **structuration** ; l'optimiseur la traite soit en la **fusionnant** dans la requête principale (comme une sous-requête dérivée), soit en la **matérialisant** dans un résultat temporaire calculé une seule fois. C'est ce second comportement qui rend la réutilisation avantageuse : une CTE référencée plusieurs fois évite de réévaluer la même sous-requête à chaque emploi. Sur de gros volumes, il reste utile de vérifier le plan d'exécution (`EXPLAIN`, chapitre 5) pour confirmer la stratégie retenue.

## Portée et règles

Une CTE n'est visible que dans la **portée de l'instruction** où elle est déclarée : elle naît avec le `WITH` et disparaît avec la requête. Au sein d'une même clause `WITH`, une CTE ne peut référencer que celles définies **avant elle** (la référence à elle-même étant réservée au cas récursif, § 4.1).

Si une clause `WITH` est employée en tête d'une **instruction de modification** (`UPDATE`, `DELETE`), la CTE peut alors être lue par cette instruction — une extension du standard SQL (alignée sur MySQL), traitée à la section suivante.

## Points clés à retenir

Une CTE est un ensemble de résultats **temporaire et nommé**, introduit par `WITH`, dont la portée se limite à la requête. Elle sert d'abord la **lisibilité** (décomposition en étapes nommées, lues de haut en bas) et la **réutilisation** (une CTE référencée plusieurs fois n'est écrite — et souvent évaluée — qu'une fois). Plusieurs CTE peuvent s'enchaîner, chacune référençant les précédentes. Face à une sous-requête dérivée, la CTE gagne en clarté et en réutilisabilité ; face à une vue, elle reste locale à une seule requête. La forme récursive est traitée au § 4.1.

---

**Section précédente :** [4.3 — Requêtes pivotées et transformations](03-requetes-pivotees.md)  
**Section suivante :** [4.4.1 — `UPDATE` / `DELETE` lisant une CTE](04.1-update-delete-from-cte.md)  

⏭️ [UPDATE / DELETE lisant une CTE](/04-concepts-avances-sql/04.1-update-delete-from-cte.md)

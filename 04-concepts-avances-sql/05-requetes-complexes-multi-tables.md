🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Requêtes complexes multi-tables

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Référence : **MariaDB 12.3 LTS**

Les requêtes réelles dépassent rarement une seule table. Un rapport, un tableau de bord ou une extraction métier croisent souvent clients, commandes, lignes, paiements, produits… en combinant jointures, sous-requêtes, CTE, fonctions de fenêtrage et opérateurs ensemblistes. Cette section ne présente pas un nouvel opérateur : elle montre comment **assembler** ces briques de façon cohérente et lisible, et surtout comment éviter les pièges qui n'apparaissent qu'à cette échelle — au premier rang desquels la **multiplication des lignes**.

## 🎯 Objectif de la section

Savoir composer une requête sur plusieurs tables en choisissant le bon outil à chaque position, structurer un raisonnement complexe par décomposition, et reconnaître les écueils classiques des jointures multiples.

## La boîte à outils

Les requêtes complexes combinent des éléments déjà étudiés, chacun ayant sa place :

- les **jointures** (§ 3.3) relient les tables entre elles ;
- les **sous-requêtes** (§ 3.4) imbriquent une requête dans une autre ;
- les **opérateurs ensemblistes** (§ 3.5, `UNION`/`INTERSECT`/`EXCEPT`) combinent des résultats ;
- les **CTE** (§ 4.4) décomposent et clarifient ;
- les **fonctions de fenêtrage** (§ 4.2) ajoutent classements et agrégats sans regrouper.

La difficulté n'est pas chaque brique prise isolément, mais leur orchestration.

## Les sous-requêtes selon leur position

Une sous-requête ne joue pas le même rôle selon l'endroit où elle se trouve.

Dans la **liste du `SELECT`**, une sous-requête *scalaire* (renvoyant une seule valeur) calcule une donnée par ligne. Souvent **corrélée**, elle référence la requête englobante :

```sql
SELECT c.nom,
       (SELECT COUNT(*) FROM commandes AS cmd WHERE cmd.client_id = c.id) AS nb_commandes
FROM clients AS c;
```

Dans le **`FROM`**, une sous-requête forme une *table dérivée* — que l'on préfère souvent exprimer en CTE pour la lisibilité (§ 4.4).

Dans le **`WHERE`**, une sous-requête sert de prédicat : avec `IN`, avec un opérateur de comparaison, ou avec `EXISTS`. C'est le terrain des semi-jointures et anti-jointures, détaillé ci-dessous.

Une sous-requête **corrélée** (qui dépend de la ligne courante de la requête englobante) est conceptuellement évaluée pour chaque ligne ; une sous-requête **non corrélée** est indépendante et peut n'être évaluée qu'une fois. La corrélation est expressive mais peut coûter cher : une jointure suivie d'un `GROUP BY` est fréquemment plus performante qu'une sous-requête scalaire corrélée pour le même besoin.

## Semi-jointures et anti-jointures (`EXISTS` / `NOT EXISTS`)

Deux besoins reviennent constamment : « les lignes qui ont au moins une correspondance » et « les lignes qui n'en ont aucune ». On les exprime élégamment avec `EXISTS` et `NOT EXISTS`.

```sql
-- Clients ayant au moins une commande (semi-jointure)
SELECT c.nom
FROM clients AS c
WHERE EXISTS (SELECT 1 FROM commandes AS cmd WHERE cmd.client_id = c.id);

-- Clients sans aucune commande (anti-jointure)
SELECT c.nom
FROM clients AS c
WHERE NOT EXISTS (SELECT 1 FROM commandes AS cmd WHERE cmd.client_id = c.id);
```

`EXISTS` ne vérifie que la **présence** d'au moins une ligne : le contenu du `SELECT` interne (`SELECT 1` ou `SELECT *`) est sans importance, et l'évaluation s'arrête à la première correspondance.

> ⚠️ **`NOT EXISTS` plutôt que `NOT IN`.** Une anti-jointure écrite avec `NOT IN` devient un piège si la sous-requête peut renvoyer un `NULL` : la condition entière bascule alors en « inconnu » et ne renvoie aucune ligne. `NOT EXISTS` n'a pas ce défaut. Ce comportement tient à la logique ternaire des `NULL`, traitée au § 4.6.

## Structurer une requête complexe : décomposer avec des CTE

Face à une requête qui empile jointures, agrégations et fenêtres, la meilleure méthode est la **décomposition en étapes nommées**, chaque CTE représentant un palier du raisonnement. On construit ainsi de manière incrémentale : d'abord les tables et jointures de base, puis l'agrégation, puis les calculs analytiques.

Exemple : pour chaque client, son chiffre total, son rang et son écart à la moyenne — en croisant `clients`, `commandes` et `lignes_commande` :

```sql
WITH totaux_client AS (
    SELECT c.id, c.nom,
           SUM(l.quantite * l.prix_unitaire) AS total
    FROM clients AS c
    JOIN commandes AS cmd ON cmd.client_id = c.id
    JOIN lignes_commande AS l ON l.commande_id = cmd.id
    GROUP BY c.id, c.nom
)
SELECT nom, total,
       RANK()      OVER (ORDER BY total DESC) AS rang,
       ROUND(total - AVG(total) OVER (), 2)   AS ecart_moyenne
FROM totaux_client
ORDER BY rang;
```

La CTE isole l'agrégation multi-tables ; la requête principale applique sur ce résultat propre un classement (`RANK`) et un écart à la moyenne générale (`AVG(...) OVER ()`). Chaque préoccupation reste à sa place.

## Le piège de la multiplication des lignes (*fan-out*)

C'est l'écueil majeur des jointures multiples. Lorsqu'une table est jointe à **plusieurs** tables filles (relations « un-à-plusieurs»), le produit des correspondances **multiplie les lignes**, ce qui fausse les agrégats. Considérons des commandes ayant chacune plusieurs lignes **et** plusieurs paiements :

```sql
-- ❌ Résultats faussés
SELECT cmd.id,
       SUM(l.montant) AS total_lignes,
       SUM(p.montant) AS total_paiements
FROM commandes AS cmd
JOIN lignes_commande AS l ON l.commande_id = cmd.id
JOIN paiements       AS p ON p.commande_id = cmd.id
GROUP BY cmd.id;
```

Si une commande a 3 lignes et 2 paiements, la jointure produit 3 × 2 = 6 lignes : `SUM(l.montant)` compte chaque ligne deux fois (autant que de paiements) et `SUM(p.montant)` compte chaque paiement trois fois. Les deux totaux sont gonflés.

La solution consiste à **pré-agréger** chaque table fille indépendamment, dans sa propre CTE, **avant** de joindre :

```sql
-- ✅ Pré-agrégation
WITH lignes AS (
    SELECT commande_id, SUM(montant) AS total_lignes
    FROM lignes_commande
    GROUP BY commande_id
),
paiements_agg AS (
    SELECT commande_id, SUM(montant) AS total_paiements
    FROM paiements
    GROUP BY commande_id
)
SELECT cmd.id, l.total_lignes, p.total_paiements
FROM commandes AS cmd
LEFT JOIN lignes        AS l ON l.commande_id = cmd.id
LEFT JOIN paiements_agg AS p ON p.commande_id = cmd.id;
```

Chaque enfant est ramené à une seule ligne par commande avant la jointure : plus de multiplication. Les `LEFT JOIN` préservent les commandes dépourvues de lignes ou de paiements. À défaut de pré-agrégation, `COUNT(DISTINCT …)` permet parfois de corriger un dénombrement, mais ne règle pas le gonflement d'une somme.

## Performance et lisibilité

Une requête multi-tables sollicite l'optimiseur : les colonnes de **jointure** et de **filtrage** doivent être indexées (chapitre 5), et il est utile de vérifier le plan avec `EXPLAIN` (§ 5.7) pour confirmer que les index sont employés et qu'aucune étape ne balaie inutilement une table. Filtrer tôt (`WHERE`) réduit le volume avant jointures et agrégations. Côté lisibilité, les CTE restent le meilleur levier : une requête décomposée en paliers nommés se relit, se teste et se maintient bien plus facilement qu'un bloc monolithique.

## Points clés à retenir

Une requête complexe **orchestre** des briques déjà connues : jointures (§ 3.3), sous-requêtes (§ 3.4), opérateurs ensemblistes (§ 3.5), CTE (§ 4.4) et fonctions de fenêtrage (§ 4.2). Le choix de l'outil dépend de la position : sous-requête scalaire (souvent corrélée) dans le `SELECT`, table dérivée ou CTE dans le `FROM`, prédicat (`IN`, comparaison, `EXISTS`) dans le `WHERE`. Pour « au moins une » ou « aucune » correspondance, `EXISTS` / `NOT EXISTS` s'imposent — `NOT EXISTS` étant préférable à `NOT IN` en présence de `NULL` (§ 4.6). La bonne pratique structurante est la **décomposition en CTE**, étape par étape. Enfin, gare à la **multiplication des lignes** lors de jointures vers plusieurs tables filles : on **pré-agrège** chaque enfant avant de joindre. Indexation et `EXPLAIN` (chapitre 5) restent les garants de la performance.

---

**Section précédente :** [4.4.1 — `UPDATE` / `DELETE` lisant une CTE](04.1-update-delete-from-cte.md)  
**Section suivante :** [4.6 — Gestion des valeurs `NULL` : Logique ternaire](06-gestion-valeurs-null.md)  

⏭️ [Gestion des valeurs NULL : Logique ternaire](/04-concepts-avances-sql/06-gestion-valeurs-null.md)

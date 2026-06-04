🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.9 — Index covering et index-only scans

> **Chapitre 5 — Index et Performance** · Section 5.9  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La [section 5.1](01-fonctionnement-index.md) a décrit le **retour à la table** (*bookmark lookup*) : pour récupérer une colonne absente d'un index secondaire, InnoDB doit effectuer une **seconde traversée**, celle de l'index clusterisé, afin de lire la ligne complète. Cette section présente la technique qui **élimine** purement et simplement ce retour à la table : l'**index couvrant** (*covering index*), qui permet à MariaDB de répondre à une requête **sans jamais lire la table** — c'est l'*index-only scan*. C'est l'une des optimisations d'indexation les plus puissantes, et le couronnement de tout ce chapitre.

---

## Rappel : le retour à la table

Reprenons le mécanisme. Un index secondaire ne contient que **ses colonnes** et la **clé primaire** ([section 5.1](01-fonctionnement-index.md)). Pour une requête comme :

```sql
-- Index idx_client (client_id) uniquement
SELECT id, montant FROM commandes WHERE client_id = 42;
```

InnoDB localise les lignes via `client_id` dans l'index secondaire — mais `montant` n'y figure pas. Il doit donc, **pour chaque ligne trouvée**, retourner dans l'index clusterisé chercher la valeur de `montant`. Sur de nombreuses lignes, ces allers-retours génèrent beaucoup d'**I/O aléatoires** et plombent la requête.

---

## L'index couvrant : la définition

Un index est dit **couvrant** pour une requête donnée lorsqu'il contient **toutes les colonnes dont cette requête a besoin** — celles du `WHERE`, du `JOIN`, de l'`ORDER BY`/`GROUP BY`, **et** celles du `SELECT`. Quand cette condition est remplie, MariaDB trouve **tout** dans l'index : il n'a **aucune raison d'accéder à la table**.

```sql
-- Index couvrant idx_client_montant (client_id, montant)
SELECT id, montant FROM commandes WHERE client_id = 42;
```

Ici, l'index suffit à tout fournir : `client_id` (le filtre), `montant` (la colonne demandée)… et `id`, qui est la **clé primaire** — donc déjà présente dans chaque feuille de l'index secondaire (voir plus bas). Résultat : **aucun retour à la table**.

---

## Pourquoi c'est rapide

Le gain de l'index-only scan tient à plusieurs facteurs cumulés :

- **Suppression du retour à la table** : on économise une traversée d'index par ligne — souvent l'essentiel du coût d'une requête sur index secondaire.
- **Moins de données lues** : l'index ne contient que quelques colonnes, là où la ligne complète peut être large (nombreuses colonnes, voire des `BLOB`).
- **Meilleure mise en cache** : plus compact, l'index tient en moins de pages, davantage susceptibles de résider dans le Buffer Pool.

Sur une requête fréquente, transformer un accès « index + retour à la table » en index-only scan peut diviser le temps de réponse par un facteur important.

---

## La spécificité InnoDB : la clé primaire est « gratuite »

Point déjà évoqué en [section 5.1](01-fonctionnement-index.md), mais décisif ici : dans InnoDB, **toute feuille d'index secondaire contient déjà la clé primaire**. Les colonnes de la clé primaire sont donc **implicitement présentes** dans n'importe quel index secondaire, sans qu'on ait à les y ajouter.

Conséquence pratique : une requête qui ne sélectionne que des **colonnes indexées + la clé primaire** est **automatiquement couverte**. C'est pourquoi, dans l'exemple précédent, sélectionner `id` (la PK) ne « coûte » rien à la couverture.

---

## Reconnaître un index couvrant dans EXPLAIN

`EXPLAIN` ([section 5.7](07-analyse-plans-execution.md)) signale un index-only scan par la mention **`Using index`** dans la colonne `Extra`. Mais attention à **trois indications voisines qu'il ne faut surtout pas confondre** :

| Indication | Où | Signification |
|------------|-----|---------------|
| **`Using index`** | colonne `Extra` | ✅ **Index couvrant** : réponse depuis l'index seul (*index-only scan*) |
| **`Using index condition`** | colonne `Extra` | *Index Condition Pushdown* — **≠ couvrant** (cf. [5.8.1](08.1-optimisations-scans-inverses.md)) |
| **`index`** | colonne **`type`** | **Balayage de l'index entier** — **≠ couvrant**, souvent à surveiller |

La confusion entre `Using index` (excellent) et `type: index` (un balayage complet de l'index) est particulièrement fréquente : ce sont des notions **distinctes** — la première (colonne `Extra`) est excellente, la seconde (colonne `type`) est souvent à surveiller. Elles peuvent d'ailleurs **coexister** : un balayage d'index complet qui se trouve être couvrant affiche **à la fois** `type: index` et `Using index` (lecture de tout l'index, mais sans accès à la table).

---

## Concevoir un index couvrant

Pour rendre un index couvrant, il suffit, en MariaDB, d'**ajouter les colonnes manquantes comme parties de la clé** de l'index — il n'existe pas de clause `INCLUDE` séparée comme dans certains autres SGBD ; toutes les colonnes d'un index MariaDB sont des colonnes de clé.

La méthode combine les règles de la [section 5.6](06-index-composites.md) :

1. placer les colonnes de **filtrage** (égalité), puis de **tri**, dans l'ordre dicté par la règle ESR ;
2. **ajouter en fin** d'index les colonnes seulement **sélectionnées** — elles n'ont pas à respecter les règles de préfixe (elles ne servent pas à filtrer), il suffit qu'elles soient **présentes**.

```sql
-- Requête : SELECT date_commande, montant
--           WHERE client_id = 42 ORDER BY date_commande
-- Index couvrant : (client_id, date_commande, montant)
--   client_id  → filtre (égalité)
--   date_commande → tri  (évite aussi le filesort, cf. 5.5.3)
--   montant    → seulement présent, pour couvrir le SELECT
CREATE INDEX idx_couvrant ON commandes (client_id, date_commande, montant);

SELECT date_commande, montant FROM commandes
WHERE client_id = 42 ORDER BY date_commande;
-- EXPLAIN Extra : « Using index »
```

> **Limite à connaître :** un **index préfixe** sur une chaîne (`nom(20)`, cf. [5.2.1](02.1-btree.md)) ne stocke que les premiers caractères : il **ne peut pas couvrir** la colonne complète. Pour couvrir une colonne textuelle, l'index doit en contenir l'intégralité.

---

## Le compromis : largeur contre coût

L'index couvrant a un prix : en y ajoutant des colonnes, on l'**élargit**. Un index plus large est **plus volumineux** (stockage, mémoire) et **plus coûteux à maintenir** en écriture — chaque modification des colonnes couvertes doit le mettre à jour. On retrouve l'arbitrage central de la [section 5.5](05-strategies-indexation.md) : il faut **couvrir des requêtes ciblées à forte valeur** (fréquentes, sur peu de colonnes), et non chercher à tout couvrir.

C'est aussi pourquoi le `SELECT *` est l'ennemi de la couverture : couvrir *toutes* les colonnes reviendrait à dupliquer la table dans l'index. Ne sélectionner que les **colonnes nécessaires** ([section 5.8](08-optimisation-requetes.md)) est la condition même pour qu'un index couvrant soit réaliste.

---

## Cas d'usage privilégiés

L'index couvrant brille particulièrement pour :

- les **requêtes très fréquentes** ne renvoyant qu'un petit nombre de colonnes (recherches, listings) ;
- les **comptages** et **tests d'existence** (`COUNT`, `EXISTS`) résolus depuis l'index seul ;
- les **lectures triées et paginées** (`WHERE … ORDER BY … LIMIT`), où l'on cumule l'évitement du *filesort* ([5.5.3](05.3-index-order-group.md)) et celui du retour à la table.

---

> ### 📝 À retenir  
>  
> - Un **index couvrant** contient **toutes** les colonnes d'une requête (`WHERE`, jointure, tri **et** `SELECT`) : MariaDB répond **depuis l'index seul**, sans accéder à la table — c'est l'*index-only scan*.  
> - Il **supprime le retour à la table** (*bookmark lookup*), lit moins de données et se met mieux en cache : un gain souvent majeur.  
> - Dans InnoDB, la **clé primaire est implicitement présente** dans tout index secondaire : une requête sur colonnes indexées + PK est automatiquement couverte.  
> - `EXPLAIN` le signale par **`Using index`** (colonne `Extra`) — à ne **pas** confondre avec `Using index condition` (ICP) ni avec `type: index` (balayage complet de l'index).  
> - Le concevoir : colonnes de filtrage/tri selon ESR ([5.6](06-index-composites.md)), puis colonnes seulement sélectionnées **ajoutées en fin**. Un **index préfixe ne couvre pas**.  
> - **Compromis** : un index couvrant est plus **large** (coût en écriture/stockage) ; on couvre des **requêtes ciblées**, et le `SELECT *` rend la couverture irréaliste.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.8.1 Optimisations sur scans inversés](08.1-optimisations-scans-inverses.md)
- ➡️ Section suivante : [5.10 Invisible indexes et Progressive indexes](10-invisible-progressive-indexes.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Invisible indexes et Progressive indexes](/05-index-et-performance/10-invisible-progressive-indexes.md)

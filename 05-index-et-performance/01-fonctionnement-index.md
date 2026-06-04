🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.1 — Fonctionnement des index : la structure B-Tree

> **Chapitre 5 — Index et Performance** · Section 5.1  
> Version de référence : **MariaDB 12.3 LTS**

---

## Le problème : le balayage complet de table

Considérons une requête anodine sur une table de clients :

```sql
SELECT * FROM clients WHERE email = 'alice@example.fr';
```

En l'absence d'index sur la colonne `email`, MariaDB n'a aucun moyen de savoir *où* se trouve la ligne recherchée. Il doit donc lire **chaque ligne de la table**, une par une, et comparer son adresse e-mail à la valeur cible. C'est un *full table scan* (balayage complet), et son coût est **proportionnel au nombre de lignes** : on parle d'une complexité en O(*n*). Sur une table de mille lignes, c'est instantané ; sur une table de cinquante millions de lignes, c'est une catastrophe de performance.

Un index résout précisément ce problème. Grâce à lui, la même recherche ne coûte plus O(*n*) mais **O(log *n*)** : quelques accès suffisent, quel que soit le volume de la table. C'est cette transformation — d'un coût linéaire à un coût logarithmique — qui fait des index le levier de performance le plus puissant d'une base relationnelle. Pour comprendre comment elle est possible, il faut examiner la structure de données qui se cache derrière la quasi-totalité des index de MariaDB : l'arbre **B-Tree**.

---

## Qu'est-ce qu'un index, concrètement ?

Un index est une **structure de données auxiliaire**, généralement distincte de la table elle-même, qui maintient une copie **ordonnée** d'une ou plusieurs colonnes, associée à un moyen de localiser rapidement la ligne complète correspondante.

Le mot-clé est *ordonnée*. C'est parce que les valeurs sont triées que l'on peut les rechercher sans tout parcourir — exactement comme on retrouve un mot dans un dictionnaire papier sans en lire toutes les pages. L'index est cet ordre matérialisé et maintenu en permanence par le moteur de stockage, à chaque écriture.

Reste à choisir *comment* organiser ces valeurs triées pour que la recherche soit efficace, y compris quand les données changent sans cesse. La réponse retenue par InnoDB — et par l'immense majorité des SGBD — est l'arbre B-Tree.

---

## L'arbre équilibré (B-Tree)

### « B » ne signifie pas « binaire »

Première précision essentielle : le « B » de B-Tree **ne veut pas dire « binaire »**. On l'interprète le plus souvent comme *balanced* (équilibré). La structure a été décrite par Rudolf Bayer et Edward McCreight en 1972, spécifiquement pour les systèmes qui stockent leurs données sur disque.

Contrairement à un arbre binaire, où chaque nœud n'a que deux enfants, un nœud de B-Tree peut en avoir **plusieurs centaines, voire plus d'un millier**. Cette particularité, on le verra, est ce qui rend l'arbre si efficace pour une base de données.

### Anatomie d'un B-Tree

Un B-Tree est composé de trois sortes de nœuds :

- **La racine** (*root*) : le point d'entrée unique de l'arbre. Toute recherche y commence.
- **Les nœuds de branchement** (*branch* ou *internal nodes*) : les nœuds intermédiaires. Ils ne contiennent pas les données utiles, mais des **clés de séparation** et des pointeurs vers les nœuds enfants. Leur rôle est d'aiguiller la recherche.
- **Les feuilles** (*leaf nodes*) : le niveau le plus bas de l'arbre. Ce sont elles qui contiennent les **entrées indexées** (et, selon la variante, les données ou des pointeurs vers les données).

Voici une représentation schématique d'un index sur des valeurs entières :

```
Niveau 0 — racine                    [ 50 | 100 ]
                                     /      |      \
                              < 50      50 – 99     ≥ 100
                               /           |           \
Niveau 1 — branches      [20 | 35]     [60 | 80]    [120 | 160]
                          / | \          / | \         /  |  \
Niveau 2 — feuilles (valeurs triées, chaînées entre elles) :

   [5, 10] <─> [20, 35] <─> [55, 60] <─> [70, 80] <─> [120, 160] ...
    \________________________ ordre croissant ________________________/
```

### La propriété fondamentale : l'équilibre

Ce qui caractérise un B-Tree, c'est que **toutes les feuilles se trouvent exactement à la même profondeur**. L'arbre est *équilibré*. Cette propriété garantit que le nombre d'étapes pour atteindre n'importe quelle valeur est **constant et prévisible** : que l'on cherche la plus petite ou la plus grande clé, on traverse toujours le même nombre de niveaux.

Pour préserver cet équilibre, l'arbre se **réorganise automatiquement** à chaque insertion ou suppression. C'est cette autogestion qui a un coût en écriture, sur lequel nous reviendrons.

### Pourquoi l'arbre est large et peu profond

Le grand nombre d'enfants par nœud (le *fan-out*) a une conséquence décisive : l'arbre est **très large mais très peu profond**. Or la profondeur (la *hauteur* de l'arbre) détermine le nombre d'accès nécessaires à une recherche.

À titre d'illustration, avec quelques centaines à un millier d'entrées par nœud :

| Hauteur de l'arbre | Nombre d'entrées indexables (ordre de grandeur) |
|--------------------|--------------------------------------------------|
| 2 niveaux | ~ centaines de milliers |
| 3 niveaux | ~ centaines de millions |
| 4 niveaux | ~ plusieurs milliards |

Autrement dit, **trois ou quatre accès suffisent** pour localiser une ligne dans une table de plusieurs centaines de millions d'enregistrements. C'est tout le secret du O(log *n*) : la base du logarithme est si grande que le résultat reste minuscule, même pour des volumes considérables.

---

## B-Tree ou B+Tree ? Une nuance qui compte

Dans le langage courant — et dans la documentation SQL — on parle d'« index B-Tree ». Mais ce qu'InnoDB implémente réellement est une variante optimisée : le **B+Tree**. La distinction mérite d'être comprise, car elle explique plusieurs comportements de MariaDB.

| Aspect | B-Tree classique | B+Tree (utilisé par InnoDB) |
|--------|------------------|------------------------------|
| Localisation des données | À **tous** les niveaux (racine, branches, feuilles) | **Uniquement dans les feuilles** |
| Rôle des nœuds internes | Stockent des données *et* aiguillent | Aiguillent uniquement (clés de routage) |
| Feuilles | Indépendantes | **Chaînées entre elles** (liste doublement chaînée) |

Cette organisation procure deux avantages majeurs pour une base de données :

1. **Des nœuds internes plus compacts.** Comme ils ne contiennent que des clés de routage, ils tiennent davantage d'entrées par page. Le *fan-out* augmente, l'arbre est encore plus plat, et les recherches encore plus rapides.

2. **Des parcours de plage (*range scans*) ultra-efficaces.** Pour répondre à une requête comme `WHERE date BETWEEN '2026-01-01' AND '2026-03-31'`, MariaDB descend **une seule fois** jusqu'à la première feuille concernée, puis **suit le chaînage horizontal** entre feuilles pour lire toutes les valeurs suivantes, sans jamais remonter dans l'arbre. C'est ce qui rend le B+Tree excellent pour `BETWEEN`, les comparaisons `>` / `<`, et les tris `ORDER BY`.

Dans la suite de cette formation, par commodité, on continuera de parler d'« index B-Tree » comme le fait MariaDB — en gardant à l'esprit qu'il s'agit techniquement d'un B+Tree.

---

## Le rôle des pages et des accès disque

La véritable raison d'être du B-Tree est de **minimiser les accès au disque**, qui sont des ordres de grandeur plus lents que les accès à la mémoire.

InnoDB n'organise pas ses données ligne par ligne, mais par **pages** d'une taille fixe de **16 Ko par défaut** (paramètre `innodb_page_size`). Chaque nœud de l'arbre — racine, branche ou feuille — correspond exactement à **une page**. Lire un nœud revient donc à charger une page, soit depuis le disque, soit, le plus souvent, depuis le **Buffer Pool** (le cache mémoire d'InnoDB).

Comme l'arbre est plat (hauteur 3 à 4), une recherche ne réclame que quelques lectures de page. Mieux : les niveaux supérieurs — la racine et les premières branches — sont consultés à *chaque* requête et résident donc pratiquement en permanence dans le Buffer Pool. En pratique, seules les feuilles génèrent éventuellement une lecture disque. C'est pourquoi une recherche par index est, du point de vue de l'utilisateur, quasi instantanée.

---

## Index clusterisé et index secondaire dans InnoDB

InnoDB applique le B+Tree d'une manière particulière, qu'il est indispensable de comprendre car elle conditionne la conception des schémas.

### L'index clusterisé EST la table

Dans InnoDB, **la table elle-même est un B+Tree**, ordonné selon la **clé primaire**. C'est l'*index clusterisé* (*clustered index*). Sa particularité : les **feuilles contiennent les lignes complètes**, et non de simples pointeurs. La clé primaire ne se contente donc pas d'identifier une ligne — elle détermine l'**organisation physique** des données sur le disque.

Que se passe-t-il en l'absence de clé primaire explicite ? InnoDB applique une cascade de règles :

1. il utilise le premier index `UNIQUE` dont toutes les colonnes sont `NOT NULL` ;
2. à défaut, il génère une **clé cachée** interne de 6 octets (un identifiant de ligne, ou *ROWID*).

Définir une clé primaire explicite reste donc toujours préférable.

### Les index secondaires pointent vers la clé primaire

Tout autre index (sur `email`, sur `nom`, etc.) est un *index secondaire* : un B+Tree distinct dont les feuilles ne contiennent **pas** la ligne complète, mais les **colonnes indexées suivies de la valeur de la clé primaire**.

La conséquence est capitale. Pour répondre à une requête qui réclame des colonnes absentes de l'index secondaire, InnoDB effectue **deux traversées** :

```
Index secondaire (idx_email)                Index clusterisé (= la table, PK = id)
┌────────────────────────────┐              ┌──────────────────────────────────────────┐
│ email (trié)  │ id (PK)    │              │ id (trié) │ ...toutes les colonnes...    │
├───────────────┼────────────┤              ├───────────┼──────────────────────────────┤
│ alice@ex.fr   │   42       │ ───┐         │   41      │ ...                          │
│ bob@ex.fr     │   17       │    └───────► │   42      │ nom, email, ville, solde, ...│
│ ...           │   ...      │              │   43      │ ...                          │
└───────────────┴────────────┘              └───────────┴──────────────────────────────┘
   1) on localise l'email ici                  2) puis on retrouve la ligne via l'id (PK)
```

Cette seconde traversée — le **retour à la table** (*bookmark lookup*) — a deux implications pratiques majeures :

- **Les index couvrants sont précieux.** Si un index secondaire contient déjà *toutes* les colonnes nécessaires à la requête, la seconde traversée devient inutile : MariaDB répond directement depuis l'index. C'est le principe de l'*index covering* et de l'*index-only scan*, traités en [section 5.9](09-index-covering.md).
- **Une clé primaire large pénalise tout le schéma.** Puisque chaque index secondaire **réplique la valeur de la clé primaire** dans ses feuilles, une PK volumineuse (par exemple une longue chaîne, ou un UUID stocké en texte) alourdit *tous* les index secondaires de la table. C'est un argument fort en faveur de clés primaires **compactes**.

---

## Le coût caché : insertions, suppressions et éclatements de pages

L'efficacité en lecture d'un index a une contrepartie : son **maintien en écriture**. Maintenir l'ordre et l'équilibre de l'arbre représente un travail à chaque modification.

- **Insertion.** La nouvelle entrée doit être placée dans la feuille appropriée pour préserver l'ordre. Si cette feuille est déjà pleine, elle subit un **éclatement de page** (*page split*) : la page se scinde en deux, ce qui peut se propager vers les nœuds supérieurs.
- **Suppression.** L'entrée est retirée ; les pages devenues trop vides peuvent être fusionnées.

L'impact des éclatements dépend fortement de la **nature des clés insérées** :

| Type d'insertion | Comportement | Conséquence |
|------------------|--------------|-------------|
| **Séquentielle** (ex. `AUTO_INCREMENT`) | Les valeurs s'ajoutent toujours « à la fin » | Peu d'éclatements, pages bien remplies, arbre compact |
| **Aléatoire** (ex. UUIDv4 en clé primaire) | Les insertions sont dispersées dans tout l'arbre | Nombreux éclatements, fragmentation, pages à moitié vides, arbre plus volumineux |

C'est l'une des raisons pour lesquelles on privilégie des **clés primaires croissantes** : un `AUTO_INCREMENT`, ou, lorsqu'un identifiant non séquentiel est requis, un UUID **ordonné dans le temps** (UUIDv7) plutôt qu'un UUIDv4 purement aléatoire.

La fragmentation qui s'accumule au fil des écritures peut être résorbée par une opération de maintenance comme `OPTIMIZE TABLE`, abordée au [chapitre 11](../11-administration-configuration/06-maintenance-tables.md).

---

## Ce que le B-Tree fait bien… et ce qu'il ne sait pas faire

Parce que ses entrées sont **triées**, un index B-Tree est performant pour toute opération qui tire parti de cet ordre, mais inopérant lorsque l'ordre ne sert à rien.

**Opérations bien servies par un index B-Tree :**

- **Égalité** : `WHERE col = ?`
- **Plages** : `WHERE col BETWEEN ? AND ?`, ainsi que `>`, `<`, `>=`, `<=`
- **Préfixe** : `WHERE col LIKE 'abc%'` (le début est connu)
- **Tri et regroupement** : `ORDER BY col` et `GROUP BY col` peuvent éviter une opération de tri explicite, puisque l'index fournit déjà les valeurs en ordre
- **Valeurs extrêmes** : recherche du minimum ou du maximum

**Situations où l'index B-Tree est inutile ou inutilisable :**

- **Joker en tête** : `WHERE col LIKE '%abc'` — l'ordre alphabétique n'aide en rien, puisqu'on ignore le début de la valeur.
- **Fonction ou expression appliquée à la colonne** : `WHERE YEAR(date_creation) = 2026` empêche l'usage d'un index sur `date_creation`. La solution consiste soit à réécrire la condition sous forme de plage (`date_creation >= '2026-01-01' AND date_creation < '2027-01-01'`), soit à indexer une **colonne générée** dédiée ([section 18.4](../18-fonctionnalites-avancees/04-virtual-generated-columns.md)).
- **Très faible sélectivité** : sur une colonne ne comportant que quelques valeurs distinctes (un booléen, par exemple), un index apporte peu : l'optimiseur jugera souvent le balayage complet plus économique.

Notons enfin que le B-Tree est le **type d'index par défaut** dans InnoDB ; on peut l'expliciter avec la clause `USING BTREE`. D'autres structures existent pour des besoins spécifiques — Hash, Full-Text, Spatial (R-Tree) — et seront présentées en [section 5.2](02-types-index.md), tandis que l'index vectoriel HNSW fait l'objet de la [section 5.3](03-index-vector-hnsw.md).

---

> ### 📝 À retenir  
>  
> - Un index transforme une recherche linéaire (**O(*n*)**) en recherche logarithmique (**O(log *n*)**) grâce à une structure de données **ordonnée**.  
> - Le **B-Tree** est un arbre **équilibré** (toutes les feuilles à la même profondeur) au **fort *fan-out***, donc **large et peu profond** : 3 à 4 accès suffisent pour des centaines de millions de lignes.  
> - InnoDB utilise en réalité un **B+Tree** : données dans les feuilles, feuilles **chaînées** entre elles, ce qui rend les **parcours de plage** très efficaces.  
> - Chaque nœud est une **page** (16 Ko par défaut) ; les niveaux hauts résident en mémoire (**Buffer Pool**), d'où une recherche quasi instantanée.  
> - Dans InnoDB, l'**index clusterisé est la table** (organisée par la clé primaire) ; les **index secondaires** stockent la **clé primaire**, d'où un éventuel **retour à la table** — à éviter via les index couvrants (5.9) et les **clés primaires compactes**.  
> - Les index ont un **coût en écriture** (éclatements de pages) : préférer des **clés primaires croissantes** pour limiter la fragmentation.

---

## 🧭 Navigation

- ⬅️ Section précédente : [Introduction du chapitre 5](README.md)
- ➡️ Section suivante : [5.2 Types d'index](02-types-index.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Types d'index](/05-index-et-performance/02-types-index.md)

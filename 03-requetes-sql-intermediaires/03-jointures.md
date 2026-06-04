🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 Jointures

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.3 (introduction)  
> Version de référence : **MariaDB 12.3 LTS**

---

## Pourquoi des jointures ?

Dans une base relationnelle bien conçue, les données sont **réparties entre plusieurs tables** plutôt que dupliquées. On ne stocke pas le nom complet d'un client dans chacune de ses commandes : on conserve les clients dans une table `client`, les commandes dans une table `commande`, et l'on relie les deux par une **clé étrangère**. C'est le principe de la *normalisation*, vu en [section 2.5 sur les contraintes](../02-bases-du-sql/05-contraintes.md), qui évite les redondances et les incohérences.

Mais cette répartition pose une question : comment, au moment de la lecture, **recombiner** ces données éparpillées pour obtenir « la liste des commandes avec le nom de leur client » ? C'est précisément le rôle des **jointures**. Une jointure associe les lignes de deux tables (ou plus) en s'appuyant sur une **condition de correspondance**, généralement l'égalité entre une clé étrangère et la clé primaire qu'elle référence.

---

## Le principe d'une jointure

Joindre deux tables, c'est mettre leurs lignes en correspondance selon une condition, puis produire des lignes combinées réunissant les colonnes des deux tables. La condition s'exprime dans une clause **`ON`** :

```
SELECT colonnes
FROM table_gauche
[TYPE] JOIN table_droite
    ON table_gauche.cle = table_droite.cle;
```

Tout l'enjeu — et ce qui distingue les différents *types* de jointures — tient à une seule question : **que faire des lignes qui n'ont pas de correspondance ?** Faut-il les écarter, ou les conserver en complétant les colonnes manquantes par des `NULL` ? Les sections 3.3.1 à 3.3.5 répondent à cette question pour chaque variante. La présente introduction se concentre sur les éléments **communs à toutes les jointures** : le schéma de travail, la syntaxe et le vocabulaire.

---

## Schéma d'exemple

Les jointures supposant au moins deux tables, nous faisons évoluer l'exemple des sections précédentes vers une forme **normalisée** : les clients sont désormais isolés dans leur propre table, et `commande` y fait référence par une clé étrangère `client_id`.

```sql
CREATE TABLE client (
    id               INT PRIMARY KEY,
    nom              VARCHAR(50),
    ville            VARCHAR(50),
    date_inscription DATE
);

CREATE TABLE commande (
    id        INT PRIMARY KEY,
    client_id INT,
    montant   DECIMAL(10,2),
    date_cmd  DATE,
    CONSTRAINT fk_commande_client
        FOREIGN KEY (client_id) REFERENCES client(id)
);
```

**Table `client`**

| id | nom | ville | date_inscription |
|----|-----|-------|------------------|
| 1 | Dupont | Lille | 2025-11-02 |
| 2 | Martin | Marseille | 2025-12-15 |
| 3 | Leroy | Strasbourg | 2026-01-20 |
| 4 | Petit | Lyon | 2026-02-05 |

**Table `commande`**

| id | client_id | montant | date_cmd |
|----|-----------|---------|----------|
| 1 | 1 | 120.00 | 2026-01-05 |
| 2 | 2 | 80.50 | 2026-01-07 |
| 3 | 1 | 200.00 | 2026-02-10 |
| 4 | 3 | 80.50 | 2026-02-15 |
| 5 | 2 | 150.00 | 2026-03-01 |

Deux caractéristiques de ce jeu de données serviront à illustrer le comportement des jointures dans les sous-sections : le client **Petit (id 4) n'a passé aucune commande**, et le client **Dupont (id 1) en a passé deux**. Ces situations — une ligne sans correspondance, une correspondance multiple — sont au cœur de la distinction entre les types de jointures.

---

## La syntaxe d'une jointure

### Syntaxe explicite (ANSI) contre syntaxe implicite

Historiquement, on « joignait » deux tables en les listant séparées par une virgule, la correspondance étant exprimée dans le `WHERE` :

```sql
-- Syntaxe implicite (héritée, à éviter)
SELECT c.nom, cmd.montant
FROM client c, commande cmd
WHERE c.id = cmd.client_id;
```

La syntaxe moderne, dite **explicite** ou **ANSI**, sépare clairement la *relation entre les tables* (clause `JOIN ... ON`) de la *sélection des lignes* (clause `WHERE`) :

```sql
-- Syntaxe explicite (recommandée)
SELECT c.nom, cmd.montant
FROM client c
JOIN commande cmd ON c.id = cmd.client_id;
```

**Privilégiez systématiquement la syntaxe explicite.** Elle est plus lisible, sépare les responsabilités, et surtout évite un piège classique de la syntaxe implicite : oublier la condition dans le `WHERE` produit silencieusement un **produit cartésien** (toutes les combinaisons possibles), source d'erreurs et de requêtes monstrueuses. De plus, la syntaxe explicite est la seule à permettre les jointures externes (`LEFT`/`RIGHT JOIN`).

### La clause ON

`ON` exprime la condition d'appariement. Le plus souvent, il s'agit d'une égalité entre clé étrangère et clé primaire (`c.id = cmd.client_id`), mais la condition peut être n'importe quelle expression booléenne, y compris combinée :

```sql
... JOIN commande cmd
    ON c.id = cmd.client_id
   AND cmd.montant > 100
```

> ℹ️ Placer une condition dans le `ON` ou dans le `WHERE` n'a pas toujours le même effet : pour une jointure *interne* le résultat est identique, mais pour une jointure *externe* (`LEFT`/`RIGHT`) la distinction devient cruciale. Ce point est détaillé en [section 3.3.2](03.2-left-right-join.md).

### La clause USING

Lorsque la colonne de jointure porte **le même nom** dans les deux tables, `USING` offre un raccourci, et la colonne commune n'apparaît alors qu'une fois dans le résultat :

```sql
-- Applicable si les deux tables ont une colonne « client_id »
SELECT nom, montant
FROM client
JOIN commande USING (client_id);
```

Dans notre schéma, les colonnes se nomment `id` et `client_id` : `USING` ne s'applique donc pas directement ici, et l'on conserve la forme `ON`.

### Alias de table

Préfixer chaque table d'un **alias** court (`client c`, `commande cmd`) allège l'écriture et clarifie l'origine de chaque colonne. Les alias sont une simple commodité dans la plupart des cas, mais ils deviennent **indispensables** pour joindre une table à elle-même (le *self-join* de la [section 3.3.4](03.4-self-join.md)).

### Qualifier les colonnes

Quand une même colonne existe dans les deux tables (par exemple un `id` de part et d'autre), il faut **qualifier** la référence par son alias (`c.id`, `cmd.id`) sous peine d'erreur d'ambiguïté. Même lorsque ce n'est pas strictement nécessaire, qualifier toutes les colonnes d'une requête à jointures est une bonne habitude de lisibilité.

---

## Les types de jointures

Les sous-sections suivantes détaillent chaque type. En voici la vue d'ensemble :

| Type de jointure | Lignes conservées | Section |
|------------------|-------------------|---------|
| `INNER JOIN` | uniquement les lignes appariées dans les **deux** tables | [3.3.1](03.1-inner-join.md) |
| `LEFT` / `RIGHT JOIN` | **toutes** les lignes d'un côté, complétées par des `NULL` côté non apparié | [3.3.2](03.2-left-right-join.md) |
| `CROSS JOIN` | **toutes** les combinaisons possibles (produit cartésien) | [3.3.3](03.3-cross-join.md) |
| *Self-join* | une table jointe **à elle-même** | [3.3.4](03.4-self-join.md) |
| Syntaxe `( + )` (mode Oracle) 🆕 | équivalent d'une jointure externe, pour compatibilité Oracle | [3.3.5](03.5-oracle-outer-join.md) |

On rencontre souvent une représentation des jointures sous forme de **diagrammes de Venn** (intersection pour `INNER`, cercle complet pour `LEFT`, etc.). Cette image aide à mémoriser quelles lignes sont conservées, mais elle reste imparfaite : elle ne rend pas compte des correspondances *multiples* (un client lié à plusieurs commandes démultiplie les lignes) ni du produit cartésien. À garder comme intuition, pas comme modèle exact.

---

## Quelques points à connaître

### FULL OUTER JOIN n'existe pas nativement

MariaDB **ne prend pas en charge `FULL OUTER JOIN`** (qui conserverait toutes les lignes des deux tables). On l'émule en combinant un `LEFT JOIN` et un `RIGHT JOIN` réunis par [`UNION`](05-operateurs-ensemblistes.md) — une technique présentée en [section 3.3.2](03.2-left-right-join.md).

### NATURAL JOIN : à éviter

`NATURAL JOIN` joint automatiquement les tables sur **toutes** leurs colonnes de même nom, sans clause `ON`. Pratique en apparence, il est déconseillé en production : la jointure devient implicite et peut se rompre ou changer de sens dès qu'on ajoute une colonne de nom commun aux deux tables. Préférez toujours un `ON` ou un `USING` explicite.

### Joindre plus de deux tables

Une requête peut enchaîner plusieurs jointures pour relier trois tables ou davantage, en empilant les clauses `JOIN ... ON`. Les requêtes complexes multi-tables font l'objet de la [section 4.5](../04-concepts-avances-sql/05-requetes-complexes-multi-tables.md).

### La syntaxe Oracle `( + )` 🆕

Pour faciliter les migrations depuis Oracle, MariaDB 12.x reconnaît, **en mode Oracle** (`SET sql_mode = 'ORACLE'`), l'ancienne notation `( + )` des jointures externes. Il s'agit d'un dispositif de **compatibilité** : dans du code MariaDB natif, on emploie `LEFT`/`RIGHT JOIN`. Le sujet est traité en [section 3.3.5](03.5-oracle-outer-join.md) et remis dans le contexte de la migration en [section 19.2.1](../19-migration-compatibilite/02.1-depuis-oracle.md).

---

## Performance : indexer les colonnes de jointure

L'efficacité d'une jointure dépend très largement de la présence d'**index sur les colonnes utilisées dans le `ON`**. Sans index, MariaDB doit, pour chaque ligne d'une table, parcourir l'autre intégralement — coût qui explose avec le volume.

Bonne nouvelle côté clés étrangères : InnoDB **crée automatiquement un index** sur la colonne portant une contrainte de clé étrangère si aucun n'existe déjà. La colonne `commande.client_id` de notre schéma est donc indexée du fait de sa contrainte `fk_commande_client`. Pour les jointures portant sur des colonnes *sans* clé étrangère, l'indexation reste à votre charge. Ce sujet est développé en [section 5.5.2 sur l'indexation des clés étrangères](../05-index-et-performance/05.2-index-cles-etrangeres.md).

---

## À retenir

- Une jointure recombine des données réparties entre plusieurs tables, en appariant leurs lignes via une condition (`ON`), typiquement clé étrangère = clé primaire.
- Ce qui distingue les types de jointures, c'est le sort réservé aux lignes **sans correspondance** : écartées (interne) ou conservées avec des `NULL` (externe).
- Préférez la **syntaxe explicite `JOIN ... ON`** à la syntaxe implicite par virgule, plus risquée.
- `USING` simplifie l'écriture quand les colonnes de jointure ont le même nom ; les **alias** sont indispensables pour le self-join ; qualifier les colonnes évite les ambiguïtés.
- MariaDB ne propose pas `FULL OUTER JOIN` (à émuler par `UNION`) et permet la notation Oracle `( + )` en mode Oracle (compatibilité).
- Indexez les colonnes de jointure ; InnoDB indexe automatiquement les colonnes de clé étrangère.

---

## Navigation

- ⬅️ Section précédente : [3.2 — Regroupement de données (GROUP BY, HAVING)](02-regroupement-donnees.md)
- ➡️ Section suivante : [3.3.1 — INNER JOIN : Intersection](03.1-inner-join.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [INNER JOIN : Intersection](/03-requetes-sql-intermediaires/03.1-inner-join.md)

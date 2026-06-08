🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.5 Invisible columns

## Qu'est-ce qu'une colonne invisible ?

Une **colonne invisible** (*invisible column*) est une colonne qui n'apparaît pas dans le résultat d'un `SELECT *` et qui est ignorée par un `INSERT` ne nommant pas explicitement les colonnes. Elle n'est pas pour autant inaccessible : il suffit de la **désigner par son nom** pour la lire, l'insérer ou la modifier.

Disponible depuis MariaDB 10.3, cette fonctionnalité répond à un besoin très concret : ajouter ou conserver une colonne dans une table sans perturber le code existant qui s'appuie sur `SELECT *` ou sur des `INSERT` positionnels.

## Déclaration et comportement

Une colonne devient invisible grâce à l'attribut `INVISIBLE` :

```sql
CREATE TABLE produit (
  id      INT PRIMARY KEY,
  libelle VARCHAR(100),
  note    TEXT INVISIBLE
);
```

Le comportement se résume à deux règles. D'une part, la colonne est **omise des opérations implicites** :

```sql
SELECT * FROM produit;                 -- renvoie id, libelle (pas note)
INSERT INTO produit VALUES (1, 'Clé'); -- alimente id et libelle ; note prend son défaut
```

D'autre part, elle reste **pleinement accessible dès qu'on la nomme** :

```sql
SELECT id, libelle, note FROM produit;                       -- note est lisible
INSERT INTO produit (id, libelle, note) VALUES (2, 'Vis', 'fragile');
UPDATE produit SET note = 'à vérifier' WHERE id = 1;
```

Une colonne invisible se comporte donc comme n'importe quelle colonne dans un `WHERE`, un `ORDER BY` ou une jointure : seule sa présence dans les formes *implicites* (`SELECT *`, `INSERT` sans liste) change.

## La règle du `NOT NULL` : un `DEFAULT` obligatoire

Puisqu'un `INSERT` positionnel ne fournit aucune valeur pour une colonne invisible, celle-ci doit pouvoir se passer d'une valeur explicite. Conséquence : **une colonne invisible déclarée `NOT NULL` doit obligatoirement définir une valeur `DEFAULT`**, faute de quoi les insertions implicites échoueraient.

```sql
-- refusé : NOT NULL invisible sans DEFAULT
ALTER TABLE produit ADD COLUMN code INT NOT NULL INVISIBLE;

-- correct
ALTER TABLE produit ADD COLUMN code INT NOT NULL DEFAULT 0 INVISIBLE;
```

## Rendre une colonne visible ou invisible

La visibilité se modifie à tout moment, sans réécrire les données, en **redéfinissant la colonne** avec `MODIFY` (ou `CHANGE`). MariaDB ne connaît que l'attribut `INVISIBLE` : on rend une colonne invisible en l'ajoutant, et visible en l'**omettant** (il n'existe pas de mot-clé `VISIBLE`, contrairement à MySQL) :

```sql
ALTER TABLE produit MODIFY note TEXT INVISIBLE;   -- rendre invisible
ALTER TABLE produit MODIFY note TEXT;             -- rendre visible (on retire INVISIBLE)
```

## Le cas d'usage majeur : enrichir une table sans casser l'existant

C'est la raison d'être principale des colonnes invisibles. Imaginons une table déjà exploitée par une application qui utilise `SELECT *` et des `INSERT` positionnels :

```sql
CREATE TABLE client (
  id  INT PRIMARY KEY,
  nom VARCHAR(100)
);
```

Ajouter une colonne *ordinaire* casserait ce code : `SELECT *` renverrait une colonne inattendue, et les `INSERT` positionnels se retrouveraient en décalage. En la déclarant **invisible**, on l'ajoute sans perturber l'existant :

```sql
ALTER TABLE client
  ADD COLUMN cree_le TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP INVISIBLE;

SELECT * FROM client;              -- toujours id, nom seulement
INSERT INTO client VALUES (1, 'Dupont');   -- fonctionne encore, cree_le prend son défaut
```

Le code historique continue de tourner à l'identique, pendant que le nouveau code exploite la colonne explicitement :

```sql
SELECT id, nom, cree_le FROM client;
```

C'est un outil précieux pour les **migrations progressives** et l'évolution de bases dont on ne maîtrise pas tous les consommateurs.

## Autres usages

Au-delà de ce scénario, les colonnes invisibles permettent de **masquer des colonnes internes** — clé de substitution technique, métadonnées d'audit, colonne d'aide dénormalisée — afin d'alléger les `SELECT *` du quotidien tout en gardant la donnée disponible. C'est d'ailleurs le mécanisme employé par défaut pour les **colonnes de période du versionnement système** (§18.2.1), invisibles afin de préserver la transparence des requêtes.

## Ce que les colonnes invisibles ne sont pas

Un point essentiel : **l'invisibilité n'est pas une mesure de sécurité**. Une colonne invisible reste lisible par quiconque connaît son nom et dispose des privilèges adéquats ; elle est seulement écartée des formes implicites, par commodité. Pour réellement protéger une donnée sensible, il faut recourir aux véritables mécanismes prévus à cet effet : vues de masquage (§9.5), privilèges au niveau colonne (§10) ou techniques de masquage de données.

À ne pas confondre, par ailleurs, avec les **index invisibles** (§5.10) : ces derniers masquent un *index* à l'optimiseur, ce qui est un sujet entièrement distinct.

## Inspecter et restrictions

`SHOW CREATE TABLE` affiche fidèlement l'attribut `INVISIBLE`, ce qui permet de repérer les colonnes cachées d'une table :

```sql
SHOW CREATE TABLE client\G
```

Une seule restriction structurante est à connaître : **une table doit conserver au moins une colonne visible**. On ne peut donc pas rendre toutes les colonnes invisibles.

## Points clés à retenir

- Une colonne **`INVISIBLE`** est exclue de `SELECT *` et des `INSERT` sans liste de colonnes, mais reste accessible dès qu'on la **nomme explicitement**.
- Une colonne invisible **`NOT NULL`** doit définir un **`DEFAULT`**.
- La visibilité se bascule en redéfinissant la colonne (`ALTER TABLE … MODIFY col type [INVISIBLE]`), sans réécriture ; MariaDB n'a pas de mot-clé `VISIBLE` (on retire `INVISIBLE` pour rendre visible).
- Usage phare : **ajouter une colonne sans casser** le code existant (`SELECT *`, `INSERT` positionnels) ; utile aussi pour masquer des colonnes internes.
- **Ce n'est pas une protection** : pour la confidentialité, utiliser vues, privilèges colonne ou masquage (§9.5, §10). À distinguer des index invisibles (§5.10).
- Une table doit toujours garder **au moins une colonne visible**.

⏭️ [Compression de tables](/18-fonctionnalites-avancees/06-compression-tables.md)

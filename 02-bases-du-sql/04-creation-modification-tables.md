🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Création et modification de tables (CREATE, ALTER, DROP)

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

La **table** est l'objet central d'une base relationnelle : c'est elle qui contient les données, organisées en lignes et en colonnes. Cette section présente les trois instructions qui en gèrent la structure — `CREATE TABLE` pour la créer, `ALTER TABLE` pour la faire évoluer et `DROP TABLE` pour la supprimer — ainsi que les commandes permettant d'en inspecter la définition. Les **contraintes** (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, etc.) se déclarent au sein de `CREATE TABLE` ; elles sont introduites ici dans les exemples mais détaillées en section 2.5.

## Créer une table

Une table se crée en énumérant ses colonnes, chacune avec son nom, son type et d'éventuels attributs :

```sql
CREATE TABLE client (
    id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nom       VARCHAR(100)  NOT NULL,
    email     VARCHAR(255)  NOT NULL UNIQUE,
    pays      CHAR(2)       NOT NULL DEFAULT 'FR',
    cree_le   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actif     BOOLEAN       NOT NULL DEFAULT TRUE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;
```

Chaque colonne combine un **type** (§2.2) et des **attributs** qui en précisent le comportement : `NOT NULL` interdit l'absence de valeur, `DEFAULT` fixe une valeur par défaut, `AUTO_INCREMENT` génère un entier croissant, `UNIQUE` impose l'unicité, et `COMMENT 'texte'` documente la colonne. La table est créée dans la base courante (`USE`) ou dans une base explicitement qualifiée (`CREATE TABLE boutique.client (...)`).

La **clé primaire** peut se déclarer en ligne, à la suite d'une colonne (comme `id` ci-dessus), ou au niveau de la table — forme obligatoire pour une clé **composite** portant sur plusieurs colonnes :

```sql
CREATE TABLE ligne_commande (
    commande_id  BIGINT UNSIGNED NOT NULL,
    produit_id   BIGINT UNSIGNED NOT NULL,
    quantite     INT UNSIGNED NOT NULL DEFAULT 1,
    PRIMARY KEY (commande_id, produit_id)    -- clé primaire composite
);
```

Comme pour les bases, `IF NOT EXISTS` évite une erreur si la table existe déjà, et la variante propre à MariaDB `CREATE OR REPLACE TABLE` supprime la table existante avant de la recréer — pratique mais **destructive** :

```sql
CREATE TABLE IF NOT EXISTS client ( /* ... */ );
CREATE OR REPLACE TABLE client ( /* ... */ );   -- ⚠ supprime la table existante d'abord
```

## Les options de table

Après la liste des colonnes, plusieurs **options de table** peuvent être précisées. La plus importante est le **moteur de stockage**, fixé par `ENGINE` : à défaut, c'est `InnoDB`, le moteur transactionnel par défaut (les moteurs sont comparés au chapitre 7). Viennent ensuite le jeu de caractères et la collation (`DEFAULT CHARSET`, `COLLATE`), hérités de la base si on ne les précise pas, un commentaire de table (`COMMENT`), ou encore la valeur de départ d'un compteur (`AUTO_INCREMENT=1000`).

## Autres formes de création

Trois variantes de `CREATE TABLE` sont utiles au quotidien. `CREATE TABLE ... LIKE` recopie la **structure** d'une table existante (colonnes, index) sans ses données ; `CREATE TABLE ... AS SELECT` (souvent abrégé CTAS) crée une table à partir du **résultat d'une requête** — colonnes *et* données — mais **sans en recopier les clés, index ni contraintes** (qu'il faut donc ajouter ensuite si besoin) ; enfin `CREATE TEMPORARY TABLE` crée une table **temporaire**, visible de la seule session courante et automatiquement supprimée à sa fermeture — idéale pour des résultats intermédiaires.

```sql
CREATE TABLE client_archive LIKE client;                          -- structure seule
CREATE TABLE client_fr AS SELECT * FROM client WHERE pays = 'FR'; -- colonnes + données (sans clés ni index)
CREATE TEMPORARY TABLE resultats_tmp AS SELECT ...;               -- table de session
```

## Inspecter une table

Plusieurs commandes renseignent sur les tables existantes et leur structure :

```sql
SHOW TABLES;                  -- liste des tables de la base courante
SHOW CREATE TABLE client;     -- instruction CREATE complète, avec options
DESCRIBE client;              -- colonnes, types, nullabilité, clés (alias : DESC)
SHOW COLUMNS FROM client;     -- équivalent de DESCRIBE
```

Pour des besoins plus avancés (scripts, outils), les mêmes informations sont disponibles dans les vues d'`INFORMATION_SCHEMA`, notamment `INFORMATION_SCHEMA.TABLES` et `INFORMATION_SCHEMA.COLUMNS` (section 9.7).

## Modifier une table

`ALTER TABLE` regroupe toutes les modifications de structure. Les plus courantes concernent les colonnes :

```sql
ALTER TABLE client ADD COLUMN telephone VARCHAR(20) AFTER email;  -- ajouter (position optionnelle)
ALTER TABLE client MODIFY COLUMN nom VARCHAR(150) NOT NULL;       -- changer type/attributs
ALTER TABLE client CHANGE COLUMN nom nom_complet VARCHAR(150);    -- renommer ET redéfinir
ALTER TABLE client RENAME COLUMN pays TO code_pays;               -- renommer seulement
ALTER TABLE client DROP COLUMN actif;                             -- supprimer
ALTER TABLE client ALTER COLUMN code_pays SET DEFAULT 'BE';       -- ajuster la valeur par défaut
```

Trois précisions utiles. Lors d'un ajout, `FIRST` ou `AFTER une_colonne` permet de positionner la nouvelle colonne ; sans cela, elle est ajoutée en fin de table. La différence entre `MODIFY` (qui conserve le nom) et `CHANGE` (qui le change aussi) mérite d'être retenue, `RENAME COLUMN` se chargeant du seul renommage. Enfin, plusieurs opérations peuvent être combinées dans un même `ALTER TABLE`, séparées par des virgules :

```sql
ALTER TABLE client
    ADD COLUMN ville VARCHAR(100),
    DROP COLUMN telephone,
    MODIFY COLUMN nom_complet VARCHAR(200) NOT NULL;
```

`ALTER TABLE` sert aussi à renommer la table, changer son moteur ou convertir son jeu de caractères :

```sql
ALTER TABLE client RENAME TO usager;                 -- (équivaut à RENAME TABLE client TO usager)
ALTER TABLE usager ENGINE=InnoDB;                    -- changer/reconstruire avec un moteur
ALTER TABLE usager CONVERT TO CHARACTER SET utf8mb4
                   COLLATE utf8mb4_uca1400_ai_ci;    -- convertir colonnes textuelles
```

**Attention au coût** : sur une grande table, un `ALTER TABLE` peut être long, car il nécessite parfois de **reconstruire entièrement la table**. MariaDB sait toutefois exécuter certaines modifications de façon quasi instantanée ou sans bloquer les écritures, grâce aux clauses `ALGORITHM` (`INSTANT`, `INPLACE`, `COPY`…) et `LOCK` (`NONE`, `SHARED`, `EXCLUSIVE`). L'ajout d'une colonne en fin de table, par exemple, peut souvent être instantané :

```sql
ALTER TABLE usager ADD COLUMN remarque TEXT, ALGORITHM=INSTANT;
```

Le changement de schéma en ligne (non bloquant) est approfondi en section 18.11, et son impact sur la réplication en section 13.10.

## Supprimer une table

`DROP TABLE` supprime une ou plusieurs tables **avec leurs données**. L'opération est **irréversible** (hors sauvegarde) ; `IF EXISTS` évite une erreur si la table est déjà absente :

```sql
DROP TABLE IF EXISTS client;
DROP TABLE commande, ligne_commande;   -- plusieurs tables en une instruction
```

À ne pas confondre avec `TRUNCATE TABLE`, qui **vide** une table tout en conservant sa structure (voir section 2.8).

## En résumé

`CREATE TABLE` déclare les colonnes (type + attributs) et, le plus souvent, une clé primaire, en fixant volontiers le moteur (`InnoDB` par défaut) et le jeu de caractères ; ses variantes `LIKE`, `AS SELECT` et `TEMPORARY` couvrent la copie de structure, la création à partir d'une requête et les tables de session. `SHOW TABLES`, `SHOW CREATE TABLE` et `DESCRIBE` permettent d'inspecter l'existant. `ALTER TABLE` fait évoluer la structure (colonnes, index, moteur, jeu de caractères), en gardant à l'esprit son coût potentiel et les clauses `ALGORITHM`/`LOCK` qui l'atténuent. Enfin, `DROP TABLE` supprime définitivement une table et ses données — à manier avec prudence. Les contraintes utilisées dans ces exemples font l'objet de la section suivante.

---

← Section précédente : [2.3 Création et gestion des bases de données](03-creation-gestion-bases.md) · [Sommaire du chapitre](README.md) · Section suivante : [2.5 Contraintes](05-contraintes.md) →

⏭️ [Contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT)](/02-bases-du-sql/05-contraintes.md)

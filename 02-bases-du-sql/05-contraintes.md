🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT)

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

Les **contraintes** sont des règles que l'on attache au schéma pour garantir la **validité et la cohérence des données**, indépendamment de l'application qui les manipule. Confier ces garde-fous à la base plutôt qu'au seul code applicatif est une bonne pratique : la règle s'applique alors uniformément, quelle que soit la voie d'accès aux données. Cette section présente les contraintes essentielles — `NOT NULL`, `DEFAULT`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY` — ainsi que la contrainte `CHECK`, déjà évoquée comme alternative portable à `ENUM`.

L'exemple suivant les rassemble ; les paragraphes qui suivent détaillent chacune d'elles.

```sql
CREATE TABLE commande (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    client_id  BIGINT UNSIGNED NOT NULL,
    reference  VARCHAR(20)   NOT NULL UNIQUE,
    montant    DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    statut     VARCHAR(20)   NOT NULL DEFAULT 'en_attente',
    cree_le    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_commande_client
        FOREIGN KEY (client_id) REFERENCES client(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT chk_montant_positif CHECK (montant >= 0)
) ENGINE=InnoDB;
```

## `NOT NULL`

La contrainte `NOT NULL` **interdit l'absence de valeur** dans une colonne. À l'inverse, une colonne déclarée `NULL` (le comportement par défaut) accepte l'absence de valeur. Tenter d'insérer une ligne sans fournir de valeur pour une colonne `NOT NULL` dépourvue de défaut provoque une erreur (en mode SQL strict). On marque `NOT NULL` toute colonne pour laquelle une valeur est **toujours requise** — un identifiant, un libellé obligatoire, etc. — ce qui clarifie le modèle et évite des `NULL` parasites.

## `DEFAULT`

`DEFAULT` fournit une **valeur par défaut** appliquée lorsqu'aucune valeur n'est précisée à l'insertion. Il peut s'agir d'une valeur littérale (`DEFAULT 'FR'`, `DEFAULT 0`) ou d'une **expression** (`DEFAULT (CURRENT_TIMESTAMP)`, par exemple). Combiné à `NOT NULL`, il permet d'insérer une ligne sans mentionner la colonne tout en garantissant qu'elle reçoit une valeur valide :

```sql
quantite  INT UNSIGNED NOT NULL DEFAULT 1
cree_le   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
```

## `UNIQUE`

La contrainte `UNIQUE` garantit que toutes les valeurs d'une colonne (ou d'une **combinaison** de colonnes) sont **distinctes**. Elle est mise en œuvre par un index unique, ce qui accélère aussi les recherches sur la colonne concernée.

```sql
email      VARCHAR(255) NOT NULL UNIQUE       -- unicité sur une colonne
UNIQUE (commande_id, produit_id)              -- unicité sur un couple de colonnes
```

Subtilité importante : par défaut, une contrainte `UNIQUE` **autorise plusieurs valeurs `NULL`**, car en logique SQL deux `NULL` ne sont jamais considérés comme égaux (§4.6). Si l'on veut interdire les doublons *et* l'absence de valeur, on combine `UNIQUE` avec `NOT NULL`.

## `PRIMARY KEY`

La **clé primaire** identifie de façon unique chaque ligne d'une table. Elle équivaut à la combinaison de `UNIQUE` **et** `NOT NULL` : ses colonnes sont implicitement non nulles, et il ne peut exister qu'**une seule** clé primaire par table. Elle peut être composite (porter sur plusieurs colonnes), auquel cas elle se déclare au niveau de la table.

Sous InnoDB, la clé primaire joue un rôle particulier : elle constitue l'index *clustered*, c'est-à-dire que les lignes sont **physiquement ordonnées** selon elle. Son choix a donc un impact direct sur les performances (chapitre 5). En pratique, on emploie très souvent une **clé technique** (*surrogate key*) auto-incrémentée — comme `id BIGINT UNSIGNED AUTO_INCREMENT` — plutôt qu'une donnée métier susceptible de changer. Comme principe général, toute table de données gagne à posséder une clé primaire.

## `FOREIGN KEY`

La **clé étrangère** assure l'**intégrité référentielle** entre deux tables : une valeur de la table « enfant » doit correspondre à une valeur existante dans une colonne **clé** (clé primaire ou unique) de la table « parent ». Elle empêche ainsi de référencer une ligne inexistante.

```sql
CONSTRAINT fk_commande_client
    FOREIGN KEY (client_id) REFERENCES client(id)
    ON DELETE RESTRICT ON UPDATE CASCADE
```

Les clés étrangères ne sont réellement appliquées que par les moteurs qui les prennent en charge, au premier rang desquels **InnoDB** ; un moteur comme MyISAM les ignore (chapitre 7). La colonne référencée doit être indexée (c'est une clé), et il est recommandé d'indexer aussi la colonne porteuse côté enfant — InnoDB crée d'ailleurs cet index automatiquement à défaut.

Les clauses `ON DELETE` et `ON UPDATE` définissent le comportement lorsqu'une ligne parent est supprimée ou modifiée. Les actions les plus utilisées sont : **`RESTRICT`** (ou `NO ACTION`), qui **interdit** l'opération tant que des enfants référencent la ligne — c'est le comportement par défaut ; **`CASCADE`**, qui **propage** la suppression ou la mise à jour aux lignes enfant ; et **`SET NULL`**, qui met la clé étrangère enfant à `NULL` (la colonne doit donc être nullable). L'action `SET DEFAULT`, prévue par le standard, n'est pas appliquée par InnoDB.

Une contrainte de clé étrangère peut être **nommée** (`CONSTRAINT nom FOREIGN KEY ...`), ce qui facilite les messages d'erreur et sa suppression ultérieure. La **portée d'unicité de ces noms** a évolué dans la série 12.x : ce point précis est traité en section 18.12.

## `CHECK`

La contrainte `CHECK` valide qu'une **condition** est vraie pour chaque ligne, à l'insertion comme à la mise à jour — et MariaDB l'applique effectivement.

```sql
CONSTRAINT chk_montant_positif CHECK (montant >= 0)
CHECK (statut IN ('en_attente','payee','expediee','annulee'))
CHECK (date_fin >= date_debut)
```

Elle constitue notamment l'**alternative portable et standard** à un `ENUM` (§2.2.2) : `CHECK (statut IN (...))` exprime la même contrainte de domaine sans recourir à une extension propre à MariaDB.

## Contraintes nommées, niveau colonne et niveau table

Une contrainte peut se déclarer **au niveau de la colonne** (en ligne, juste après sa définition) ou **au niveau de la table** (dans une clause distincte). Le niveau table est obligatoire pour les contraintes **composites** (clé primaire, unique ou étrangère portant sur plusieurs colonnes). Le mot-clé `CONSTRAINT nom` permet de **nommer** explicitement une contrainte ; à défaut, MariaDB lui attribue un nom. Nommer ses contraintes — surtout les clés étrangères et les `CHECK` — rend les messages d'erreur plus parlants et simplifie leur suppression.

## Ajouter et supprimer des contraintes

Les contraintes peuvent être ajoutées ou retirées après coup avec `ALTER TABLE` :

```sql
ALTER TABLE commande ADD UNIQUE (reference);
ALTER TABLE commande ADD CONSTRAINT chk_montant_positif CHECK (montant >= 0);
ALTER TABLE commande DROP FOREIGN KEY fk_commande_client;   -- par son nom
ALTER TABLE commande DROP CONSTRAINT chk_montant_positif;
ALTER TABLE commande MODIFY COLUMN statut VARCHAR(20) NULL; -- lever un NOT NULL
```

À noter que l'ajout d'une contrainte sur une table déjà peuplée peut échouer si les données existantes ne la respectent pas : MariaDB vérifie la conformité du contenu avant de l'appliquer.

## En résumé

Les contraintes déplacent la garantie d'intégrité du code applicatif vers le schéma lui-même. `NOT NULL` rend une valeur obligatoire et `DEFAULT` en fournit une à défaut ; `UNIQUE` impose des valeurs distinctes (en autorisant plusieurs `NULL`) ; `PRIMARY KEY`, qui combine unicité et non-nullité, identifie chaque ligne et structure physiquement la table sous InnoDB ; `FOREIGN KEY` garantit l'intégrité référentielle entre tables (avec ses actions `RESTRICT`, `CASCADE`, `SET NULL`), à condition d'utiliser un moteur compatible comme InnoDB ; et `CHECK` valide une condition arbitraire, offrant une alternative portable à `ENUM`. Nommer ses contraintes et bien choisir entre niveau colonne et niveau table rend le schéma plus lisible et plus facile à maintenir.

---

← Section précédente : [2.4 Création et modification de tables](04-creation-modification-tables.md) · [Sommaire du chapitre](README.md) · Section suivante : [2.6 Insertion de données](06-insertion-donnees.md) →

⏭️ [Insertion de données (INSERT, INSERT INTO SELECT, LOAD DATA)](/02-bases-du-sql/06-insertion-donnees.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 — Création et gestion des index

> **Chapitre 5 — Index et Performance** · Section 5.4  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Après avoir vu *quels* types d'index existent, place à la **mécanique** : comment les créer, les inspecter, les modifier et les supprimer. Cette section couvre la « boîte à outils » DDL des index. Elle se concentre sur le **comment faire** — le *quelles* colonnes indexer relève de la stratégie, traitée en [section 5.5](05-strategies-indexation.md).

Un point mérite d'être souligné d'emblée : sur une table volumineuse en production, créer ou supprimer un index n'est pas une opération anodine. La dernière partie y est consacrée.

---

## Créer un index

### Trois moments, trois syntaxes

Un index peut être créé de trois manières équivalentes :

```sql
-- ① À la création de la table
CREATE TABLE commandes (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  client_id     INT NOT NULL,
  reference     VARCHAR(30) NOT NULL,
  statut        VARCHAR(20),
  date_commande DATETIME,
  INDEX idx_client (client_id),
  UNIQUE INDEX uq_reference (reference)
) ENGINE = InnoDB;

-- ② Sur une table existante, avec CREATE INDEX
CREATE INDEX idx_statut ON commandes (statut);

-- ③ Sur une table existante, avec ALTER TABLE … ADD
ALTER TABLE commandes ADD INDEX idx_date (date_commande);
```

Les options 2 et 3 sont strictement interchangeables. La forme `ALTER TABLE` présente un avantage : elle permet de **regrouper plusieurs modifications** en une seule instruction, ce qui ne reconstruit la table qu'une fois (voir plus bas).

> **Note.** `INDEX` et `KEY` sont des **synonymes** en SQL MariaDB ; les deux mots-clés produisent le même résultat.

### Conventions de nommage

Donner un nom explicite à chaque index facilite grandement la maintenance. Une convention répandue préfixe le nom selon la nature de l'index (`idx_` pour un index standard, `uq_` pour un index unique, `ft_` pour un full-text, etc.) suivi des colonnes concernées : `idx_client_date`, `uq_email`. Si le nom est omis, MariaDB en génère un automatiquement à partir du nom de la première colonne — mais un nom maîtrisé reste préférable.

### Les variantes par type

Le type d'index se choisit par un mot-clé à la déclaration (voir [section 5.2](02-types-index.md)) :

| Type | Forme `… INDEX (col)` | Forme `CREATE … INDEX` |
|------|----------------------|------------------------|
| Standard (B-Tree) | `INDEX` / `KEY` | `CREATE INDEX` |
| Unique | `UNIQUE [INDEX]` | `CREATE UNIQUE INDEX` |
| Full-Text | `FULLTEXT INDEX` | `CREATE FULLTEXT INDEX` |
| Spatial | `SPATIAL INDEX` | `CREATE SPATIAL INDEX` |
| Vector | `VECTOR INDEX` | `CREATE VECTOR INDEX` |

### Options de définition

Plusieurs options affinent un index :

```sql
-- Préfixe : n'indexer que les 20 premiers caractères d'une chaîne (cf. 5.2.1)
CREATE INDEX idx_nom ON clients (nom(20));

-- Ordre descendant (cf. 5.2.1 et 5.6)
CREATE INDEX idx_date_desc ON commandes (date_commande DESC);

-- Index composite : l'ordre des colonnes est déterminant (cf. 5.6)
CREATE INDEX idx_client_date ON commandes (client_id, date_commande);

-- Structure explicite (USING BTREE / USING HASH ; cf. 5.2.1 et 5.2.2)
CREATE INDEX idx_code ON correspondance (code) USING HASH;
```

---

## Inspecter les index existants

Trois outils permettent de connaître les index d'une table.

**`SHOW INDEX`** liste tous les index avec leurs caractéristiques :

```sql
SHOW INDEX FROM commandes;
```

Parmi ses colonnes, les plus utiles sont `Key_name` (nom de l'index), `Seq_in_index` (position de la colonne dans un index composite), `Column_name`, `Non_unique` (0 si unique), `Cardinality` (estimation du nombre de valeurs distinctes — un indicateur de sélectivité, cf. 5.2.1) et `Ignored` (l'index est-il rendu invisible, cf. 5.10).

**`SHOW CREATE TABLE`** restitue la définition complète, index compris — pratique pour reproduire ou auditer un schéma :

```sql
SHOW CREATE TABLE commandes\G
```

**`INFORMATION_SCHEMA.STATISTICS`** offre un accès **programmatique**, idéal pour des audits ou des scripts :

```sql
SELECT INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX, CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'boutique' AND TABLE_NAME = 'commandes'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

---

## Modifier un index

Il n'existe **pas** d'instruction pour changer les colonnes d'un index existant : pour modifier sa composition, on le **supprime puis on le recrée**. En revanche, deux modifications légères sont possibles sans reconstruction :

```sql
-- Renommer un index
ALTER TABLE commandes RENAME INDEX idx_statut TO idx_etat;

-- Rendre un index invisible (« ignoré ») sans le supprimer (détails en 5.10)
ALTER TABLE commandes ALTER INDEX idx_date IGNORED;
ALTER TABLE commandes ALTER INDEX idx_date NOT IGNORED;
```

Rendre un index *ignoré* permet de tester l'effet de sa suppression sans perdre sa définition — un usage abordé en [section 5.10](10-invisible-progressive-indexes.md).

---

## Supprimer un index

```sql
-- Deux syntaxes équivalentes
DROP INDEX idx_statut ON commandes;
ALTER TABLE commandes DROP INDEX idx_date;

-- Cas particulier : la clé primaire
ALTER TABLE commandes DROP PRIMARY KEY;
```

Supprimer la clé primaire d'une table InnoDB n'est pas neutre : faute de clé primaire explicite, le moteur retombe sur ses règles de substitution (premier index `UNIQUE NOT NULL`, ou clé cachée), comme vu en [section 5.1](01-fonctionnement-index.md). On évite donc de laisser une table InnoDB sans clé primaire.

> **⚠️ Cas de la colonne `AUTO_INCREMENT`.** Si la clé primaire porte sur une colonne `AUTO_INCREMENT` (le cas le plus fréquent), MariaDB **refuse** de la supprimer directement — `ALTER TABLE commandes DROP PRIMARY KEY` renvoie alors l'erreur 1075 (« there can be only one auto column and it must be defined as a key »), car une colonne `AUTO_INCREMENT` doit toujours faire partie d'une clé. Il faut d'abord **retirer l'attribut `AUTO_INCREMENT`**, puis supprimer la clé :  
>
> ```sql
> ALTER TABLE commandes MODIFY id INT NOT NULL;  -- retire AUTO_INCREMENT
> ALTER TABLE commandes DROP PRIMARY KEY;        -- désormais possible
> ```

---

## ⚠️ Opérations en ligne : ne pas bloquer la production

Ajouter ou supprimer un index sur une **grosse table** peut prendre du temps et, selon la méthode employée, **verrouiller la table** pendant toute l'opération — bloquant les écritures, voire les lectures. Sur un système en production, c'est à proscrire sans précaution.

MariaDB permet de contrôler ce comportement par deux clauses, `ALGORITHM` et `LOCK` :

- **`ALGORITHM`** — comment l'opération est menée : `INSTANT` (immédiat, pour quelques opérations seulement), `INPLACE` (sans recopier la table), `NOCOPY`, `COPY` (recopie complète), ou `DEFAULT`.
- **`LOCK`** — le niveau de verrouillage toléré : `NONE` (lectures *et* écritures concurrentes autorisées), `SHARED` (lectures seules), `EXCLUSIVE` (table verrouillée), ou `DEFAULT`.

L'**ajout d'un index secondaire** sur InnoDB s'effectue typiquement « en place » et **sans bloquer** les écritures :

```sql
ALTER TABLE commandes
  ADD INDEX idx_client (client_id),
  ALGORITHM = INPLACE, LOCK = NONE;
```

L'intérêt d'expliciter `LOCK = NONE` est double : on **garantit** l'absence de blocage, et si l'opération demandée ne peut **pas** se faire sans verrou plus contraignant, MariaDB renvoie une **erreur** plutôt que de bloquer silencieusement la production. C'est un filet de sécurité précieux.

Lorsqu'on a plusieurs index à manipuler sur une même table, il est plus efficace de **tout regrouper** dans une seule instruction `ALTER TABLE`, qui n'effectuera qu'un seul passage :

```sql
ALTER TABLE commandes
  ADD INDEX idx_statut (statut),
  ADD INDEX idx_date (date_commande),
  DROP INDEX idx_obsolete,
  ALGORITHM = INPLACE, LOCK = NONE;
```

Pour les cas que l'*online DDL* natif ne couvre pas sans interruption, des outils externes comme **gh-ost** ou **pt-online-schema-change** ([section 16.8.3](../16-devops-automatisation/08.3-gh-ost-pt-osc.md)) effectuent la modification via une table fantôme. Le sujet général de la modification de schéma non bloquante est traité en [section 18.11](../18-fonctionnalites-avancees/11-online-schema-change.md), et l'efficacité de la **construction d'index** elle-même (paramètre `innodb_alter_copy_bulk`) en [section 15.6](../15-performance-tuning/06-innodb-alter-copy-bulk.md).

---

## Maintenance des index

Une fois créés, les index demandent un entretien minimal mais réel :

- **Mettre à jour les statistiques.** L'optimiseur décide d'utiliser ou non un index d'après des estimations de **cardinalité**. Sur une table dont le contenu a beaucoup évolué, `ANALYZE TABLE` recalcule ces statistiques et restaure des plans d'exécution pertinents.

  ```sql
  ANALYZE TABLE commandes;
  ```

- **Défragmenter / reconstruire.** Au fil des écritures, un index B-Tree se fragmente (cf. 5.1). `OPTIMIZE TABLE` reconstruit la table et ses index, récupérant l'espace et l'efficacité perdus.

  ```sql
  OPTIMIZE TABLE commandes;
  ```

Ces deux opérations, ainsi que leur planification, sont détaillées au [chapitre 11](../11-administration-configuration/06-maintenance-tables.md).

> **Privilèges.** La création et la suppression d'index requièrent le privilège `INDEX` (ou `ALTER`) sur la table, abordé au [chapitre 10](../10-securite-gestion-utilisateurs/03-systeme-privileges.md).

---

> ### 📝 À retenir  
>  
> - Un index se crée de trois façons équivalentes : **dans `CREATE TABLE`**, avec **`CREATE INDEX`**, ou avec **`ALTER TABLE … ADD INDEX`** (`INDEX` et `KEY` sont synonymes).  
> - Le **type** se choisit par mot-clé (`UNIQUE`, `FULLTEXT`, `SPATIAL`, `VECTOR`) ; les **options** incluent préfixe `(col(n))`, ordre `DESC` et `USING`.  
> - On inspecte les index avec **`SHOW INDEX`**, **`SHOW CREATE TABLE`** et **`INFORMATION_SCHEMA.STATISTICS`**.  
> - **Pas d'`ALTER` pour changer les colonnes** d'un index : on le supprime et on le recrée. On peut en revanche le **renommer** ou le rendre **ignoré** (5.10).  
> - Sur une grosse table en production, **toujours** maîtriser le blocage via **`ALGORITHM = INPLACE, LOCK = NONE`** ; regrouper les modifications dans un seul `ALTER TABLE`.  
> - Entretien : **`ANALYZE TABLE`** (statistiques) et **`OPTIMIZE TABLE`** (défragmentation), détaillés au chapitre 11.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.3 Index VECTOR (HNSW)](03-index-vector-hnsw.md)
- ➡️ Section suivante : [5.5 Stratégies d'indexation](05-strategies-indexation.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Stratégies d'indexation](/05-index-et-performance/05-strategies-indexation.md)

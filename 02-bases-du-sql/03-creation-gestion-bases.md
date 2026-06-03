🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Création et gestion des bases de données

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

Une **base de données** est le conteneur de plus haut niveau dans lequel on regroupe des tables, des vues, des procédures et autres objets. Sous MariaDB, les mots **`DATABASE`** et **`SCHEMA`** sont strictement synonymes : on peut écrire indifféremment `CREATE DATABASE` ou `CREATE SCHEMA`. Cette section présente les opérations de base sur ce conteneur — création, consultation, sélection, modification et suppression — ainsi que les règles de nommage à connaître.

## Créer une base de données

L'instruction `CREATE DATABASE` crée une nouvelle base :

```sql
CREATE DATABASE boutique;
```

Deux options reviennent souvent. `IF NOT EXISTS` évite une erreur si la base existe déjà (utile dans un script rejouable), tandis que la clause de jeu de caractères fixe l'encodage par défaut de la base :

```sql
CREATE DATABASE IF NOT EXISTS boutique
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_ai_ci;
```

MariaDB propose aussi une variante propre, `CREATE OR REPLACE DATABASE`, qui supprime la base existante avant de la recréer. Elle est pratique pour repartir d'une base vierge, mais **destructive** : tout le contenu de la base précédente est perdu.

```sql
CREATE OR REPLACE DATABASE boutique;   -- ⚠ supprime la base existante puis la recrée
```

## Jeu de caractères et collation par défaut

Le jeu de caractères et la collation déclarés à la création de la base deviennent les **valeurs par défaut héritées par ses tables** (sauf si une table ou une colonne en précise d'autres). Les fixer au niveau de la base est donc un bon réflexe pour garantir la cohérence de l'ensemble.

Depuis la 11.8, la valeur par défaut du serveur est `utf8mb4`, associée aux collations UCA 14.0.0 (comme `utf8mb4_uca1400_ai_ci`) : si l'on ne précise rien, c'est ce paramétrage qui s'applique. Les notions de jeu de caractères et de collation ont été détaillées en section 2.2.2, et leur configuration au niveau du serveur en section 11.11.

## Lister et inspecter les bases

`SHOW DATABASES` affiche la liste des bases visibles, que l'on peut filtrer avec `LIKE` :

```sql
SHOW DATABASES;
SHOW DATABASES LIKE 'bout%';
```

Parmi les bases listées figurent des **bases système** à ne pas modifier ni supprimer : `mysql` (tables système et privilèges), `information_schema` (métadonnées en lecture seule), `performance_schema` (métriques de performance) et `sys` (vues d'aide). Elles sont abordées en section 9.7.

Pour examiner la définition d'une base — notamment son jeu de caractères — on utilise `SHOW CREATE DATABASE` :

```sql
SHOW CREATE DATABASE boutique;
```

## Sélectionner une base

L'instruction `USE` fixe la **base courante**, c'est-à-dire celle à laquelle se rapportent les instructions suivantes lorsqu'on ne qualifie pas les noms de tables :

```sql
USE boutique;
SELECT * FROM client;            -- 'client' est cherchée dans 'boutique'
```

On peut aussi qualifier explicitement un nom par sa base, sans `USE` préalable, ce qui permet de travailler avec plusieurs bases dans une même session :

```sql
SELECT * FROM boutique.client;   -- nom pleinement qualifié
```

La fonction `DATABASE()` renvoie la base courante (ou `NULL` si aucune n'est sélectionnée) :

```sql
SELECT DATABASE();
```

## Modifier une base

`ALTER DATABASE` permet essentiellement de changer le jeu de caractères ou la collation par défaut :

```sql
ALTER DATABASE boutique CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

À noter : modifier la collation par défaut d'une base **n'affecte pas** les tables et colonnes déjà créées, qui conservent la leur ; seul le défaut appliqué aux futurs objets change.

MariaDB ne propose **pas** d'instruction `RENAME DATABASE` (jugée trop risquée, elle a été retirée). Pour renommer une base, on procède généralement par export/import (`mariadb-dump` puis rechargement sous le nouveau nom, voir chapitre 12), ou en recréant la base cible puis en y déplaçant les tables avec `RENAME TABLE`, qui sait transférer une table d'une base à l'autre :

```sql
RENAME TABLE ancienne_base.client TO nouvelle_base.client;
```

## Supprimer une base

`DROP DATABASE` supprime une base **et tout son contenu** — tables, données, vues, procédures. L'opération est **irréversible** : en dehors d'une sauvegarde, les données ne peuvent pas être récupérées. La clause `IF EXISTS` évite une erreur si la base est déjà absente.

```sql
DROP DATABASE IF EXISTS boutique;   -- ⚠ supprime définitivement la base et son contenu
```

Compte tenu de son caractère définitif, cette commande est à manier avec une attention particulière, surtout en production.

## Règles de nommage et conventions

Un nom de base obéit à quelques contraintes : il est limité à **64 caractères** et, comme chaque base correspond à un sous-répertoire du répertoire de données (`datadir`), sa sensibilité à la casse dépend du système de fichiers et de la variable `lower_case_table_names` (par défaut, les noms sont sensibles à la casse sous Linux). Un nom contenant des caractères spéciaux ou correspondant à un mot réservé doit être entouré d'**accents graves** :

```sql
CREATE DATABASE `mon-projet`;   -- le tiret impose les accents graves
```

Par convention, et pour éviter tout souci de portabilité ou de casse, on privilégie des noms **en minuscules**, courts et sans espace, séparés par des tirets bas (`gestion_stock`, `app_facturation`).

## Privilèges

Créer une base requiert le privilège `CREATE`, et la supprimer le privilège `DROP`. L'attribution des droits d'accès aux bases et aux tables relève du système de privilèges, traité au chapitre 10.

## En résumé

Une base de données (synonyme de *schema*) se crée avec `CREATE DATABASE`, idéalement en fixant dès le départ `CHARACTER SET utf8mb4` et une collation, valeurs dont héritent ensuite les tables. On la liste avec `SHOW DATABASES`, on l'inspecte avec `SHOW CREATE DATABASE`, on s'y positionne avec `USE` (ou via des noms qualifiés `base.table`), et on ajuste ses défauts avec `ALTER DATABASE`. La suppression par `DROP DATABASE` est définitive et doit être maniée avec prudence. Faute de `RENAME DATABASE`, un renommage passe par un export/import ou par `RENAME TABLE`. Enfin, des noms en minuscules, courts et sans caractères spéciaux épargnent bien des ennuis de casse et de portabilité.

---

← Section précédente : [2.2 Types de données MariaDB](02-types-de-donnees.md) · [Sommaire du chapitre](README.md) · Section suivante : [2.4 Création et modification de tables](04-creation-modification-tables.md) →

⏭️ [Création et modification de tables (CREATE, ALTER, DROP)](/02-bases-du-sql/04-creation-modification-tables.md)

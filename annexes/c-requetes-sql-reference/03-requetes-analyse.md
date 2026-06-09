🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.3 — Requêtes d'analyse

> 🔬 L'analyse de fond qui guide l'optimisation : statistiques de distribution et usage réel des index.

Cette section regroupe des requêtes d'investigation destinées à étayer les décisions d'indexation et de tuning : connaître la cardinalité et la distribution des données, et savoir quels index sont réellement sollicités. Toutes sont en lecture seule ; l'usage des index suppose le `PERFORMANCE_SCHEMA` activé, et les statistiques moteur-indépendantes nécessitent un `ANALYZE TABLE` préalable.

---

## Statistiques de tables et de colonnes

La table `information_schema.STATISTICS` décrit la structure des index et leur **cardinalité** (nombre estimé de valeurs distinctes), donnée centrale pour l'optimiseur.

```sql
-- Index d'une table : colonnes, ordre, unicité, cardinalité estimée
SELECT index_name, seq_in_index, column_name, non_unique, cardinality
FROM   information_schema.STATISTICS
WHERE  table_schema = 'boutique' AND table_name = 'commandes'
ORDER  BY index_name, seq_in_index;
```

Pour repérer des index peu discriminants, on liste les index non uniques à faible cardinalité (à comparer au nombre de lignes de la table) :

```sql
-- Index non uniques à faible cardinalité (sélectivité douteuse)
SELECT table_schema, table_name, index_name, cardinality
FROM   information_schema.STATISTICS
WHERE  seq_in_index = 1
  AND  non_unique = 1
  AND  table_schema NOT IN ('mysql','performance_schema','information_schema','sys')
ORDER  BY cardinality ASC
LIMIT  20;
```

La cardinalité de `STATISTICS` est une **estimation** : elle se rafraîchit avec `ANALYZE TABLE`. MariaDB propose en outre des **statistiques moteur-indépendantes (EITS)**, stockées dans `mysql.table_stats`, `mysql.column_stats` et `mysql.index_stats`. Elles offrent ce que les statistiques du moteur ne donnent pas — notamment la distribution des valeurs d'une colonne, même non indexée — mais ne sont collectées qu'à la demande :

```sql
-- (Re)collecte des statistiques moteur-indépendantes pour une table
ANALYZE TABLE boutique.commandes PERSISTENT FOR ALL;
```

```sql
-- Distribution des valeurs par colonne (statistiques EITS)
SELECT column_name, nulls_ratio, avg_length, avg_frequency
FROM   mysql.column_stats
WHERE  db_name = 'boutique' AND table_name = 'commandes'
ORDER  BY column_name;
```

Trois colonnes se lisent ainsi : `nulls_ratio` donne la fraction de valeurs `NULL` (de 0 à 1) ; `avg_length`, la longueur moyenne en octets ; et `avg_frequency`, le nombre moyen de lignes partageant une même valeur — proche de 1, la colonne est très sélective (bonne candidate à l'indexation) ; nettement supérieur, elle comporte beaucoup de doublons. Ces statistiques sont gouvernées par la variable `use_stat_tables`. *Voir Ch. 5 pour les stratégies d'indexation et Ch. 11.6 pour `ANALYZE TABLE`.*

## Usage des index

> ⚠️ **Prérequis et prudence.** Ces requêtes exigent `PERFORMANCE_SCHEMA` activé. Les compteurs ne reflètent l'activité que **depuis le dernier redémarrage** (ou la dernière remise à zéro) : l'observation doit couvrir tous les cycles de charge (batchs de nuit, fin de mois…) avant d'en conclure qu'un index est inutile.

La table `performance_schema.table_io_waits_summary_by_index_usage` compte les accès par index. Un index jamais lu (`COUNT_READ = 0`) ne sert qu'à alourdir les écritures : c'est un candidat à la suppression. La requête suivante les isole en excluant clés primaires et index `UNIQUE` :

```sql
SELECT t.OBJECT_SCHEMA AS db, t.OBJECT_NAME AS tbl, t.INDEX_NAME,
       t.COUNT_READ, t.COUNT_WRITE
FROM   performance_schema.table_io_waits_summary_by_index_usage t
JOIN   information_schema.STATISTICS s
  ON  s.TABLE_SCHEMA = t.OBJECT_SCHEMA
 AND  s.TABLE_NAME   = t.OBJECT_NAME
 AND  s.INDEX_NAME   = t.INDEX_NAME
 AND  s.SEQ_IN_INDEX = 1
WHERE  t.INDEX_NAME IS NOT NULL
  AND  t.INDEX_NAME <> 'PRIMARY'
  AND  s.NON_UNIQUE = 1
  AND  t.COUNT_READ = 0
  AND  t.OBJECT_SCHEMA NOT IN ('mysql','performance_schema','information_schema','sys')
ORDER  BY t.OBJECT_SCHEMA, t.OBJECT_NAME, t.INDEX_NAME;
```

À l'inverse, on identifie les index les plus sollicités :

```sql
SELECT OBJECT_SCHEMA AS db, OBJECT_NAME AS tbl, INDEX_NAME, COUNT_READ
FROM   performance_schema.table_io_waits_summary_by_index_usage
WHERE  INDEX_NAME IS NOT NULL
  AND  OBJECT_SCHEMA NOT IN ('mysql','performance_schema','information_schema','sys')
ORDER  BY COUNT_READ DESC
LIMIT  20;
```

Un `INDEX_NAME` à `NULL` signale des opérations effectuées **sans index** — souvent révélatrices de parcours complets de table (les `INSERT` y sont toutefois aussi comptés). Les tables les plus concernées méritent un examen avec `EXPLAIN` :

```sql
SELECT OBJECT_SCHEMA AS db, OBJECT_NAME AS tbl, COUNT_STAR AS operations_sans_index
FROM   performance_schema.table_io_waits_summary_by_index_usage
WHERE  INDEX_NAME IS NULL
  AND  OBJECT_SCHEMA NOT IN ('mysql','performance_schema','information_schema','sys')
ORDER  BY COUNT_STAR DESC
LIMIT  20;
```

Avant de supprimer un index présumé inutile, deux précautions : vérifier avec `EXPLAIN` qu'aucune requête importante ne s'appuie dessus, puis le rendre **ignoré** plutôt que de le supprimer d'emblée. Sous MariaDB, `ALTER TABLE commandes ALTER INDEX idx_xxx IGNORED;` fait que l'optimiseur l'ignore sans le détruire — l'opération est instantanée et réversible, ce qui en fait une étape de transition idéale. *Voir Ch. 5.10 — Invisible/Progressive indexes.*

## Pour aller plus loin

Les stratégies d'indexation et l'analyse des plans (`EXPLAIN`) sont développées au [Ch. 5 — Index et Performance](../../05-index-et-performance/README.md), la maintenance et `ANALYZE TABLE` au [Ch. 11.6](../../11-administration-configuration/README.md), et les index invisibles/progressifs au [Ch. 5.10](../../05-index-et-performance/README.md). Pour un instantané rapide en ligne de commande, voir l'[Annexe B.2](../b-commandes-cli/02-informations-systeme.md).

---

⬅️ [C.2 — Requêtes de monitoring](02-requetes-monitoring.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [Annexe D — Configuration de Référence par Cas d'Usage](../d-configuration-reference/README.md)

⏭️ [Configuration de Référence par Cas d'Usage](/annexes/d-configuration-reference/README.md)

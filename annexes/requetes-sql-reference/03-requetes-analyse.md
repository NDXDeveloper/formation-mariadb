🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.3 - Requêtes d'Analyse

> **Niveau** : Avancé à Expert  
> **Type** : Requêtes d'optimisation et analyse  
> **Usage** : Performance tuning, capacity planning, refactoring

---

## 📖 Introduction

Cette section fournit des **requêtes SQL d'analyse avancée** pour optimiser les performances, planifier la capacité et améliorer la structure des bases de données. Ces requêtes aident à identifier les opportunités d'optimisation et à prendre des décisions éclairées.

### 🎯 Catégories Couvertes

- 📑 **Index Usage** : Utilisation et efficacité des index
- 📊 **Statistiques** : Cardinalité et distribution
- 🗜️ **Fragmentation** : Tables à optimiser
- 🔍 **Performance Schema** : Métriques avancées
- 🏗️ **Audit Schéma** : Design et structure
- 💡 **Recommandations** : Optimisations suggérées

---

## 📑 UTILISATION DES INDEX

### 1.1 - Index Inutilisés (Candidates à Suppression)
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Identifier index jamais utilisés
-- ============================================================
-- Description :
-- Liste les index qui n'ont jamais servi depuis le démarrage
--
-- Prérequis :
-- - Performance Schema activé
-- - userstat activé (optionnel)
--
-- Actions :
-- Analyser puis supprimer index inutiles pour :
-- - Réduire taille stockage
-- - Accélérer INSERT/UPDATE/DELETE
-- - Simplifier maintenance
--
-- ⚠️ Attention :
-- Vérifier sur période suffisante (plusieurs semaines)
-- ============================================================

-- Méthode 1 : Via Performance Schema
SELECT 
  t.TABLE_SCHEMA AS database_name,
  t.TABLE_NAME AS table_name,
  t.INDEX_NAME AS index_name,
  t.SEQ_IN_INDEX AS column_position,
  t.COLUMN_NAME AS column_name,
  t.CARDINALITY AS cardinality,
  t.INDEX_TYPE AS index_type,
  ROUND(
    (stat.stat_value * @@innodb_page_size) / 1024 / 1024, 2
  ) AS index_size_mb,
  '⚠️ JAMAIS UTILISÉ' AS status
FROM information_schema.STATISTICS t
LEFT JOIN mysql.innodb_index_stats stat
  ON t.TABLE_SCHEMA = stat.database_name
  AND t.TABLE_NAME = stat.table_name
  AND t.INDEX_NAME = stat.index_name
  AND stat.stat_name = 'size'
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND t.INDEX_NAME != 'PRIMARY'
  -- Index absent de performance_schema = non utilisé
  AND NOT EXISTS (
    SELECT 1 
    FROM performance_schema.table_io_waits_summary_by_index_usage u
    WHERE u.OBJECT_SCHEMA = t.TABLE_SCHEMA
      AND u.OBJECT_NAME = t.TABLE_NAME
      AND u.INDEX_NAME = t.INDEX_NAME
      AND (u.COUNT_STAR > 0 OR u.COUNT_READ > 0)
  )
GROUP BY t.TABLE_SCHEMA, t.TABLE_NAME, t.INDEX_NAME
ORDER BY index_size_mb DESC;

-- Méthode 2 : Analyse manuelle (si pas de Performance Schema)
-- Liste tous les index avec statistiques
SELECT 
  TABLE_SCHEMA AS db_name,
  TABLE_NAME AS table_name,
  INDEX_NAME AS index_name,
  GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS columns,
  CARDINALITY AS cardinality,
  INDEX_TYPE AS type,
  CASE 
    WHEN NON_UNIQUE = 0 THEN 'UNIQUE'
    ELSE 'NON-UNIQUE'
  END AS uniqueness
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND INDEX_NAME != 'PRIMARY'
GROUP BY TABLE_SCHEMA, TABLE_NAME, INDEX_NAME, INDEX_TYPE, NON_UNIQUE, CARDINALITY
ORDER BY TABLE_NAME, INDEX_NAME;
```

**Commande pour supprimer** :
```sql
-- Vérifier impact avant suppression
ALTER TABLE database.table DROP INDEX index_name;
```

---

### 1.2 - Index Redondants ou Dupliqués
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Détecter index redondants
-- ============================================================
-- Description :
-- Un index est redondant si un autre index a les mêmes
-- colonnes de gauche (left-prefix)
--
-- Exemple redondance :
-- - INDEX idx_a (col_a)
-- - INDEX idx_ab (col_a, col_b)  ← idx_a est redondant
--
-- Cas d'usage :
-- Nettoyer index redondants = moins de maintenance overhead
-- ============================================================

SELECT 
  s1.TABLE_SCHEMA AS db_name,
  s1.TABLE_NAME AS table_name,
  s1.INDEX_NAME AS redundant_index,
  GROUP_CONCAT(s1.COLUMN_NAME ORDER BY s1.SEQ_IN_INDEX) AS redundant_columns,
  s2.INDEX_NAME AS covered_by_index,
  GROUP_CONCAT(s2.COLUMN_NAME ORDER BY s2.SEQ_IN_INDEX) AS covering_columns,
  CASE 
    WHEN s1.INDEX_NAME = s2.INDEX_NAME THEN 'DUPLICATE'
    ELSE 'REDUNDANT (left-prefix)'
  END AS redundancy_type
FROM information_schema.STATISTICS s1
JOIN information_schema.STATISTICS s2 
  ON s1.TABLE_SCHEMA = s2.TABLE_SCHEMA
  AND s1.TABLE_NAME = s2.TABLE_NAME
  AND s1.INDEX_NAME != s2.INDEX_NAME
  AND s1.SEQ_IN_INDEX <= s2.SEQ_IN_INDEX
  AND s1.COLUMN_NAME = s2.COLUMN_NAME
WHERE s1.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND s1.INDEX_NAME != 'PRIMARY'
  AND s2.INDEX_NAME != 'PRIMARY'
GROUP BY 
  s1.TABLE_SCHEMA, 
  s1.TABLE_NAME, 
  s1.INDEX_NAME,
  s2.INDEX_NAME
HAVING 
  -- S1 est left-prefix de S2
  FIND_IN_SET(
    GROUP_CONCAT(s1.COLUMN_NAME ORDER BY s1.SEQ_IN_INDEX SEPARATOR ','),
    SUBSTRING_INDEX(
      GROUP_CONCAT(s2.COLUMN_NAME ORDER BY s2.SEQ_IN_INDEX SEPARATOR ','),
      ',',
      COUNT(DISTINCT s1.COLUMN_NAME)
    )
  ) > 0
ORDER BY s1.TABLE_NAME, s1.INDEX_NAME;
```

---

### 1.3 - Statistiques d'Utilisation Index (Performance Schema)
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Fréquence d'utilisation des index
-- ============================================================
-- Prérequis :
-- - Performance Schema activé
-- - table_io_waits_summary_by_index_usage peuplée
-- ============================================================

SELECT 
  OBJECT_SCHEMA AS database_name,
  OBJECT_NAME AS table_name,
  INDEX_NAME AS index_name,
  COUNT_FETCH AS select_operations,
  COUNT_INSERT AS insert_operations,
  COUNT_UPDATE AS update_operations,
  COUNT_DELETE AS delete_operations,
  COUNT_STAR AS total_operations,
  ROUND(
    COUNT_FETCH / NULLIF(COUNT_STAR, 0) * 100, 2
  ) AS select_ratio_pct,
  CASE 
    WHEN COUNT_STAR = 0 THEN '❌ Jamais utilisé'
    WHEN COUNT_FETCH > COUNT_INSERT + COUNT_UPDATE + COUNT_DELETE 
      THEN '✓ Utilisé principalement en lecture'
    ELSE '⚠️ Plus de writes que reads'
  END AS usage_pattern
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
  AND INDEX_NAME IS NOT NULL
ORDER BY COUNT_STAR DESC
LIMIT 50;
```

---

### 1.4 - Cardinalité des Index
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Analyser cardinalité (sélectivité) des index
-- ============================================================
-- Description :
-- Cardinalité = nombre de valeurs distinctes
-- Haute cardinalité = index efficace
-- Basse cardinalité = index peu sélectif
--
-- Sélectivité = Cardinalité / Total Rows
-- > 0.9 : Très sélectif (excellent)
-- 0.5-0.9 : Bon
-- < 0.5 : Peu sélectif (revoir index)
-- ============================================================

SELECT 
  s.TABLE_SCHEMA AS db_name,
  s.TABLE_NAME AS table_name,
  s.INDEX_NAME AS index_name,
  s.COLUMN_NAME AS column_name,
  s.CARDINALITY AS distinct_values,
  t.TABLE_ROWS AS total_rows,
  ROUND(
    s.CARDINALITY / NULLIF(t.TABLE_ROWS, 0), 4
  ) AS selectivity,
  CASE 
    WHEN s.CARDINALITY IS NULL THEN '⚠️ Statistiques manquantes'
    WHEN s.CARDINALITY / NULLIF(t.TABLE_ROWS, 0) > 0.9 THEN '✓ Très sélectif'
    WHEN s.CARDINALITY / NULLIF(t.TABLE_ROWS, 0) > 0.5 THEN '✓ Bon'
    WHEN s.CARDINALITY / NULLIF(t.TABLE_ROWS, 0) > 0.1 THEN '⚠️ Moyen'
    ELSE '❌ Peu sélectif - Revoir index'
  END AS quality
FROM information_schema.STATISTICS s
JOIN information_schema.TABLES t 
  ON s.TABLE_SCHEMA = t.TABLE_SCHEMA 
  AND s.TABLE_NAME = t.TABLE_NAME
WHERE s.TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND s.SEQ_IN_INDEX = 1  -- Première colonne de l'index
  AND t.TABLE_ROWS > 100  -- Ignorer petites tables
ORDER BY selectivity ASC, total_rows DESC;
```

**Mise à jour statistiques** :
```sql
-- Recalculer cardinalité
ANALYZE TABLE database_name.table_name;
```

---

### 1.5 - Index Manquants Suggérés
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Suggérer index potentiels via Performance Schema
-- ============================================================
-- Description :
-- Analyse requêtes pour détecter scans de tables
-- qui bénéficieraient d'un index
-- ============================================================

-- Requêtes avec full table scans
SELECT 
  DIGEST_TEXT AS query_pattern,
  SCHEMA_NAME AS database_name,
  COUNT_STAR AS execution_count,
  ROUND(AVG_TIMER_WAIT / 1000000000, 3) AS avg_time_sec,
  ROUND(SUM_ROWS_EXAMINED / COUNT_STAR, 0) AS avg_rows_scanned,
  ROUND(SUM_ROWS_SENT / COUNT_STAR, 0) AS avg_rows_returned,
  ROUND(
    (SUM_ROWS_EXAMINED / COUNT_STAR) / 
    NULLIF((SUM_ROWS_SENT / COUNT_STAR), 0), 2
  ) AS scan_efficiency_ratio,
  '⚠️ Considérer index' AS suggestion
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
  AND SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND DIGEST_TEXT NOT LIKE '%information_schema%'
  -- Ratio rows scanned/returned élevé = mauvais
  AND (SUM_ROWS_EXAMINED / COUNT_STAR) > 100
  AND (SUM_ROWS_EXAMINED / COUNT_STAR) / 
      NULLIF((SUM_ROWS_SENT / COUNT_STAR), 0) > 10
ORDER BY 
  (SUM_ROWS_EXAMINED / COUNT_STAR) * COUNT_STAR DESC  -- Impact total
LIMIT 20;
```

**Analyser requête spécifique** :
```sql
-- Copier query_pattern et analyser
EXPLAIN <query>;

-- Chercher dans Extra :
-- - "Using where" sans index
-- - "Using filesort"
-- - "Using temporary"
```

---

## 📊 STATISTIQUES ET DISTRIBUTION

### 2.1 - Distribution des Données par Colonne
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Analyser distribution valeurs d'une colonne
-- ============================================================
-- Description :
-- Identifier valeurs les plus fréquentes (hot values)
-- Utile pour partitioning, sharding, optimisation
--
-- Paramètres :
-- - Remplacer YOUR_DATABASE, YOUR_TABLE, YOUR_COLUMN
-- ============================================================

-- Template pour analyse distribution
SELECT 
  column_name AS value,  -- ← Remplacer column_name
  COUNT(*) AS frequency,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM YOUR_TABLE), 2) AS percentage,
  CASE 
    WHEN COUNT(*) * 100.0 / (SELECT COUNT(*) FROM YOUR_TABLE) > 10 
      THEN '🔥 Hot value'
    ELSE 'Normal'
  END AS status
FROM YOUR_DATABASE.YOUR_TABLE  -- ← À REMPLACER
GROUP BY column_name
ORDER BY frequency DESC
LIMIT 50;

-- Exemple concret : Distribution statuts commandes
SELECT 
  status,
  COUNT(*) AS order_count,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM orders), 2) AS pct,
  CONCAT(REPEAT('█', FLOOR(COUNT(*) * 50 / (SELECT MAX(cnt) FROM (SELECT COUNT(*) cnt FROM orders GROUP BY status) x))), ' ') AS bar_chart
FROM orders
GROUP BY status
ORDER BY order_count DESC;
```

**Résultat exemple** :
```
+-----------+-------------+-------+--------------------------------------------------+
| status    | order_count | pct   | bar_chart                                        |
+-----------+-------------+-------+--------------------------------------------------+
| completed |      450000 | 45.00 | ██████████████████████████████████████████████   |
| pending   |      300000 | 30.00 | ████████████████████████████████                 |
| cancelled |      150000 | 15.00 | ████████████████                                 |
+-----------+-------------+-------+--------------------------------------------------+
```

---

### 2.2 - Détection Valeurs NULL
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Identifier colonnes avec beaucoup de NULL
-- ============================================================
-- Description :
-- Colonnes avec >50% NULL peuvent :
-- - Bénéficier sparse indexes
-- - Être candidates normalisation
-- - Indiquer design problem
-- ============================================================

SELECT 
  c.TABLE_SCHEMA AS db_name,
  c.TABLE_NAME AS table_name,
  c.COLUMN_NAME AS column_name,
  c.IS_NULLABLE AS nullable,
  c.DATA_TYPE AS data_type,
  t.TABLE_ROWS AS total_rows,
  -- Requête dynamique pour compter NULL (à adapter)
  '-- SELECT COUNT(*) FROM table WHERE column IS NULL' AS count_null_query,
  CASE 
    WHEN c.IS_NULLABLE = 'NO' THEN '✓ NOT NULL'
    ELSE '⚠️ Nullable - vérifier % NULL'
  END AS recommendation
FROM information_schema.COLUMNS c
JOIN information_schema.TABLES t 
  ON c.TABLE_SCHEMA = t.TABLE_SCHEMA 
  AND c.TABLE_NAME = t.TABLE_NAME
WHERE c.TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND c.IS_NULLABLE = 'YES'
  AND t.TABLE_ROWS > 1000
ORDER BY t.TABLE_ROWS DESC, c.TABLE_NAME, c.COLUMN_NAME;

-- Requête spécifique pour une table
-- (Générer dynamiquement pour chaque colonne)
SELECT 
  'total_rows' AS metric,
  COUNT(*) AS value
FROM orders
UNION ALL
SELECT 
  'customer_id NULL',
  COUNT(*)
FROM orders
WHERE customer_id IS NULL
UNION ALL
SELECT 
  'notes NULL',
  COUNT(*)
FROM orders
WHERE notes IS NULL;
```

---

### 2.3 - Analyse Types de Données
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Audit types de données et tailles
-- ============================================================
-- Description :
-- Identifier colonnes potentiellement mal typées
--
-- Optimisations possibles :
-- - VARCHAR(255) → VARCHAR(50) si max < 50
-- - INT → SMALLINT si valeurs < 32767
-- - DATETIME → DATE si pas besoin heure
-- ============================================================

SELECT 
  TABLE_SCHEMA AS db_name,
  TABLE_NAME AS table_name,
  COLUMN_NAME AS column_name,
  DATA_TYPE AS current_type,
  CASE 
    WHEN DATA_TYPE = 'varchar' THEN CONCAT('VARCHAR(', CHARACTER_MAXIMUM_LENGTH, ')')
    WHEN DATA_TYPE IN ('int', 'bigint', 'smallint', 'tinyint') THEN 
      CONCAT(UPPER(DATA_TYPE), 
        IF(COLUMN_TYPE LIKE '%unsigned%', ' UNSIGNED', ''))
    WHEN DATA_TYPE = 'decimal' THEN 
      CONCAT('DECIMAL(', NUMERIC_PRECISION, ',', NUMERIC_SCALE, ')')
    ELSE UPPER(DATA_TYPE)
  END AS full_type,
  IS_NULLABLE AS nullable,
  COLUMN_DEFAULT AS default_value,
  CASE 
    WHEN DATA_TYPE = 'varchar' AND CHARACTER_MAXIMUM_LENGTH = 255 
      THEN '⚠️ Vérifier si 255 nécessaire'
    WHEN DATA_TYPE = 'int' AND COLUMN_NAME LIKE '%_id'
      THEN '💡 Considérer INT UNSIGNED pour IDs'
    WHEN DATA_TYPE = 'datetime' AND COLUMN_NAME LIKE '%_date'
      THEN '💡 DATE suffit peut-être ?'
    WHEN DATA_TYPE = 'text' AND TABLE_ROWS < 10000
      THEN '⚠️ VARCHAR pourrait suffire'
    ELSE '✓ OK'
  END AS suggestion
FROM information_schema.COLUMNS c
JOIN information_schema.TABLES t 
  ON c.TABLE_SCHEMA = t.TABLE_SCHEMA 
  AND c.TABLE_NAME = t.TABLE_NAME
WHERE c.TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
ORDER BY c.TABLE_NAME, c.ORDINAL_POSITION;
```

---

## 🗜️ FRAGMENTATION

### 3.1 - Tables Fragmentées Détectées
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Identifier tables nécessitant OPTIMIZE TABLE
-- ============================================================
-- Description :
-- Fragmentation = espace inutilisé dans pages allouées
-- Causé par : DELETE, UPDATE changeant taille rows
--
-- Formule :
-- Fragmentation % = (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH)) * 100
--
-- Actions :
-- > 10% fragmentation : Planifier OPTIMIZE TABLE
-- > 25% : Optimiser rapidement
-- ============================================================

SELECT 
  TABLE_SCHEMA AS db_name,
  TABLE_NAME AS table_name,
  ENGINE AS engine,
  TABLE_ROWS AS row_count,
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
  ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
  ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb,
  ROUND(
    (DATA_FREE / NULLIF((DATA_LENGTH + INDEX_LENGTH + DATA_FREE), 0)) * 100, 2
  ) AS fragmentation_pct,
  ROUND(
    (DATA_LENGTH + INDEX_LENGTH + DATA_FREE) / 1024 / 1024, 2
  ) AS total_allocated_mb,
  CASE 
    WHEN (DATA_FREE / NULLIF((DATA_LENGTH + INDEX_LENGTH + DATA_FREE), 0)) * 100 > 25 
      THEN '🔴 CRITICAL - OPTIMIZE NOW'
    WHEN (DATA_FREE / NULLIF((DATA_LENGTH + INDEX_LENGTH + DATA_FREE), 0)) * 100 > 10 
      THEN '⚠️ Plan OPTIMIZE soon'
    ELSE '✓ OK'
  END AS recommendation,
  CONCAT('OPTIMIZE TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';') AS optimize_command
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND ENGINE IN ('InnoDB', 'MyISAM')
  AND DATA_FREE > 0
  AND (DATA_LENGTH + INDEX_LENGTH) > 10485760  -- > 10 MB
HAVING fragmentation_pct > 10
ORDER BY fragmentation_pct DESC, total_allocated_mb DESC;
```

**Exécuter optimisation** :
```sql
-- Sur table spécifique
OPTIMIZE TABLE database.table_name;

-- Alternative InnoDB (plus rapide, online)
ALTER TABLE database.table_name ENGINE=InnoDB;
```

---

### 3.2 - Estimation Gains OPTIMIZE TABLE
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Calculer espace récupérable via OPTIMIZE
-- ============================================================

SELECT 
  TABLE_SCHEMA AS db_name,
  COUNT(*) AS fragmented_table_count,
  ROUND(SUM(DATA_FREE) / 1024 / 1024, 2) AS total_free_mb,
  ROUND(SUM(DATA_FREE) / 1024 / 1024 / 1024, 2) AS total_free_gb,
  ROUND(
    SUM(DATA_FREE) / 
    SUM(DATA_LENGTH + INDEX_LENGTH + DATA_FREE) * 100, 2
  ) AS avg_fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND ENGINE = 'InnoDB'
  AND DATA_FREE > 0
GROUP BY TABLE_SCHEMA
HAVING total_free_mb > 100  -- Au moins 100 MB récupérables
ORDER BY total_free_mb DESC;
```

---

## 🔍 PERFORMANCE SCHEMA AVANCÉ

### 4.1 - Top Requêtes par Temps Total
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Requêtes consommant le plus de temps total
-- ============================================================
-- Description :
-- Temps total = temps moyen × nombre exécutions
-- Identifier requêtes à optimiser en priorité
-- ============================================================

SELECT 
  SCHEMA_NAME AS database_name,
  DIGEST_TEXT AS query_pattern,
  COUNT_STAR AS exec_count,
  ROUND(SUM_TIMER_WAIT / 1000000000, 2) AS total_time_sec,
  ROUND(AVG_TIMER_WAIT / 1000000000, 3) AS avg_time_sec,
  ROUND(MAX_TIMER_WAIT / 1000000000, 3) AS max_time_sec,
  ROUND(SUM_ROWS_EXAMINED / COUNT_STAR, 0) AS avg_rows_examined,
  ROUND(SUM_ROWS_SENT / COUNT_STAR, 0) AS avg_rows_sent,
  ROUND(SUM_ROWS_AFFECTED / COUNT_STAR, 0) AS avg_rows_affected,
  FIRST_SEEN AS first_execution,
  LAST_SEEN AS last_execution
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
  AND SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

---

### 4.2 - Requêtes avec Temp Tables ou Filesort
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Détecter requêtes utilisant tables temporaires
-- ============================================================
-- Description :
-- Temp tables sur disque = lent
-- Optimiser avec index appropriés
-- ============================================================

SELECT 
  SCHEMA_NAME AS db_name,
  DIGEST_TEXT AS query_pattern,
  COUNT_STAR AS exec_count,
  SUM_CREATED_TMP_TABLES AS tmp_tables_created,
  SUM_CREATED_TMP_DISK_TABLES AS tmp_tables_on_disk,
  ROUND(
    SUM_CREATED_TMP_DISK_TABLES / 
    NULLIF(SUM_CREATED_TMP_TABLES, 0) * 100, 2
  ) AS disk_tmp_ratio_pct,
  SUM_SORT_ROWS AS sorted_rows,
  SUM_SORT_SCAN AS filesort_scans,
  ROUND(AVG_TIMER_WAIT / 1000000000, 3) AS avg_time_sec,
  CASE 
    WHEN SUM_CREATED_TMP_DISK_TABLES > 0 THEN '⚠️ Temp tables sur disque'
    WHEN SUM_SORT_SCAN > 0 THEN '⚠️ Filesort détecté'
    ELSE '✓ OK'
  END AS issue
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
  AND SCHEMA_NAME NOT IN ('performance_schema', 'mysql')
  AND (SUM_CREATED_TMP_TABLES > 0 OR SUM_SORT_ROWS > 0)
ORDER BY SUM_CREATED_TMP_DISK_TABLES DESC, SUM_SORT_ROWS DESC
LIMIT 20;
```

---

### 4.3 - I/O par Table
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Activité I/O par table
-- ============================================================

SELECT 
  OBJECT_SCHEMA AS database_name,
  OBJECT_NAME AS table_name,
  COUNT_READ AS read_operations,
  COUNT_WRITE AS write_operations,
  COUNT_FETCH AS select_ops,
  COUNT_INSERT AS insert_ops,
  COUNT_UPDATE AS update_ops,
  COUNT_DELETE AS delete_ops,
  ROUND(SUM_TIMER_READ / 1000000000, 2) AS total_read_time_sec,
  ROUND(SUM_TIMER_WRITE / 1000000000, 2) AS total_write_time_sec,
  ROUND(
    (SUM_TIMER_READ + SUM_TIMER_WRITE) / 1000000000, 2
  ) AS total_io_time_sec,
  CASE 
    WHEN COUNT_READ > COUNT_WRITE * 10 THEN '📖 Read-heavy'
    WHEN COUNT_WRITE > COUNT_READ * 10 THEN '✍️ Write-heavy'
    ELSE '⚖️ Balanced'
  END AS access_pattern
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
  AND COUNT_STAR > 0
ORDER BY (SUM_TIMER_READ + SUM_TIMER_WRITE) DESC
LIMIT 30;
```

---

### 4.4 - Locks Waits par Table
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Tables avec plus d'attente de locks
-- ============================================================

SELECT 
  OBJECT_SCHEMA AS database_name,
  OBJECT_NAME AS table_name,
  COUNT_STAR AS lock_operations,
  SUM_TIMER_WAIT / 1000000000 AS total_wait_time_sec,
  ROUND(AVG_TIMER_WAIT / 1000000000, 4) AS avg_wait_time_sec,
  ROUND(MAX_TIMER_WAIT / 1000000000, 4) AS max_wait_time_sec,
  CASE 
    WHEN SUM_TIMER_WAIT / 1000000000 > 100 THEN '🔴 High contention'
    WHEN SUM_TIMER_WAIT / 1000000000 > 10 THEN '⚠️ Moderate contention'
    ELSE '✓ Low contention'
  END AS contention_level
FROM performance_schema.table_lock_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
  AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

---

## 🏗️ AUDIT SCHÉMA

### 5.1 - Tables Sans Primary Key
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Identifier tables sans clé primaire
-- ============================================================
-- Description :
-- Tables sans PK :
-- - Problèmes réplication (row-based)
-- - Performance UPDATE/DELETE dégradée
-- - Impossibilité utiliser certaines features
--
-- ⚠️ BEST PRACTICE : Toujours avoir une PK
-- ============================================================

SELECT 
  t.TABLE_SCHEMA AS database_name,
  t.TABLE_NAME AS table_name,
  t.ENGINE AS engine,
  t.TABLE_ROWS AS row_count,
  ROUND((t.DATA_LENGTH + t.INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
  '❌ PAS DE PRIMARY KEY' AS issue,
  'ALTER TABLE ... ADD PRIMARY KEY (...)' AS fix_suggestion
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND NOT EXISTS (
    SELECT 1 
    FROM information_schema.STATISTICS s
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
      AND s.TABLE_NAME = t.TABLE_NAME
      AND s.INDEX_NAME = 'PRIMARY'
  )
ORDER BY row_count DESC;
```

---

### 5.2 - Foreign Keys Sans Index
**Niveau** : 🟡 Intermédiaire

```sql
-- ============================================================
-- Foreign keys sans index sur colonne référençante
-- ============================================================
-- Description :
-- FK sans index = performance dégradée pour :
-- - DELETE/UPDATE sur table parent
-- - JOIN entre tables
--
-- MySQL/MariaDB recommande index sur FK columns
-- ============================================================

SELECT 
  kcu.TABLE_SCHEMA AS db_name,
  kcu.TABLE_NAME AS table_name,
  kcu.COLUMN_NAME AS fk_column,
  kcu.CONSTRAINT_NAME AS fk_name,
  kcu.REFERENCED_TABLE_NAME AS referenced_table,
  kcu.REFERENCED_COLUMN_NAME AS referenced_column,
  '⚠️ Pas d\'index sur FK' AS issue,
  CONCAT(
    'CREATE INDEX idx_fk_', kcu.COLUMN_NAME, 
    ' ON ', kcu.TABLE_SCHEMA, '.', kcu.TABLE_NAME,
    ' (', kcu.COLUMN_NAME, ');'
  ) AS suggested_index
FROM information_schema.KEY_COLUMN_USAGE kcu
WHERE kcu.REFERENCED_TABLE_NAME IS NOT NULL
  AND kcu.TABLE_SCHEMA NOT IN ('mysql', 'sys')
  -- Vérifier si index existe déjà
  AND NOT EXISTS (
    SELECT 1 
    FROM information_schema.STATISTICS s
    WHERE s.TABLE_SCHEMA = kcu.TABLE_SCHEMA
      AND s.TABLE_NAME = kcu.TABLE_NAME
      AND s.COLUMN_NAME = kcu.COLUMN_NAME
      AND s.SEQ_IN_INDEX = 1  -- Première colonne d'un index
  )
ORDER BY kcu.TABLE_SCHEMA, kcu.TABLE_NAME;
```

---

### 5.3 - Colonnes ENUM avec Nombreuses Valeurs
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- ENUM avec trop de valeurs (candidat à normalisation)
-- ============================================================
-- Description :
-- ENUM avec >10 valeurs :
-- - Difficile à maintenir
-- - ALTER TABLE coûteux pour ajouter valeur
-- - Considérer table de référence
-- ============================================================

SELECT 
  TABLE_SCHEMA AS db_name,
  TABLE_NAME AS table_name,
  COLUMN_NAME AS column_name,
  DATA_TYPE AS data_type,
  COLUMN_TYPE AS full_definition,
  -- Compter valeurs ENUM
  (LENGTH(COLUMN_TYPE) - LENGTH(REPLACE(COLUMN_TYPE, ',', '')) + 1) AS enum_value_count,
  CASE 
    WHEN (LENGTH(COLUMN_TYPE) - LENGTH(REPLACE(COLUMN_TYPE, ',', '')) + 1) > 10 
      THEN '⚠️ Considérer table référence'
    ELSE '✓ OK'
  END AS recommendation
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND DATA_TYPE = 'enum'
HAVING enum_value_count > 5
ORDER BY enum_value_count DESC;
```

---

### 5.4 - Contraintes et Relations
**Niveau** : 🟢 Basique

```sql
-- ============================================================
-- Vue d'ensemble contraintes par table
-- ============================================================

SELECT 
  t.TABLE_SCHEMA AS db_name,
  t.TABLE_NAME AS table_name,
  -- Primary Key
  (SELECT COUNT(*) 
   FROM information_schema.STATISTICS s 
   WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
     AND s.TABLE_NAME = t.TABLE_NAME 
     AND s.INDEX_NAME = 'PRIMARY') AS has_primary_key,
  -- Foreign Keys (sortantes)
  (SELECT COUNT(DISTINCT kcu.CONSTRAINT_NAME)
   FROM information_schema.KEY_COLUMN_USAGE kcu
   WHERE kcu.TABLE_SCHEMA = t.TABLE_SCHEMA
     AND kcu.TABLE_NAME = t.TABLE_NAME
     AND kcu.REFERENCED_TABLE_NAME IS NOT NULL) AS outgoing_fk_count,
  -- Foreign Keys (entrantes)
  (SELECT COUNT(DISTINCT kcu.CONSTRAINT_NAME)
   FROM information_schema.KEY_COLUMN_USAGE kcu
   WHERE kcu.REFERENCED_TABLE_SCHEMA = t.TABLE_SCHEMA
     AND kcu.REFERENCED_TABLE_NAME = t.TABLE_NAME) AS incoming_fk_count,
  -- Unique constraints
  (SELECT COUNT(DISTINCT s.INDEX_NAME)
   FROM information_schema.STATISTICS s
   WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
     AND s.TABLE_NAME = t.TABLE_NAME
     AND s.NON_UNIQUE = 0
     AND s.INDEX_NAME != 'PRIMARY') AS unique_constraint_count,
  -- Index count (non-unique)
  (SELECT COUNT(DISTINCT s.INDEX_NAME)
   FROM information_schema.STATISTICS s
   WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
     AND s.TABLE_NAME = t.TABLE_NAME
     AND s.NON_UNIQUE = 1) AS index_count
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND t.TABLE_TYPE = 'BASE TABLE'
ORDER BY t.TABLE_NAME;
```

---

## 💡 RECOMMANDATIONS D'OPTIMISATION

### 6.1 - Score Santé Table
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Calculer score santé global par table
-- ============================================================
-- Description :
-- Score basé sur plusieurs métriques :
-- - Présence Primary Key
-- - Fragmentation
-- - Ratio index/data
-- - Nombre index
-- ============================================================

SELECT 
  t.TABLE_SCHEMA AS db_name,
  t.TABLE_NAME AS table_name,
  t.TABLE_ROWS AS row_count,
  
  -- Critères (1 point chacun)
  CASE WHEN EXISTS (
    SELECT 1 FROM information_schema.STATISTICS s 
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
      AND s.TABLE_NAME = t.TABLE_NAME 
      AND s.INDEX_NAME = 'PRIMARY'
  ) THEN 1 ELSE 0 END AS has_pk,
  
  CASE WHEN (t.DATA_FREE / NULLIF((t.DATA_LENGTH + t.INDEX_LENGTH + t.DATA_FREE), 0)) < 0.1 
    THEN 1 ELSE 0 END AS low_fragmentation,
  
  CASE WHEN (t.INDEX_LENGTH / NULLIF(t.DATA_LENGTH, 0)) BETWEEN 0.1 AND 0.5 
    THEN 1 ELSE 0 END AS good_index_ratio,
  
  CASE WHEN (
    SELECT COUNT(DISTINCT INDEX_NAME) 
    FROM information_schema.STATISTICS s 
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
      AND s.TABLE_NAME = t.TABLE_NAME
  ) BETWEEN 1 AND 10 THEN 1 ELSE 0 END AS reasonable_index_count,
  
  CASE WHEN t.ENGINE = 'InnoDB' THEN 1 ELSE 0 END AS uses_innodb,
  
  -- Score total (max 5)
  (CASE WHEN EXISTS (
    SELECT 1 FROM information_schema.STATISTICS s 
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
      AND s.TABLE_NAME = t.TABLE_NAME 
      AND s.INDEX_NAME = 'PRIMARY'
  ) THEN 1 ELSE 0 END +
  CASE WHEN (t.DATA_FREE / NULLIF((t.DATA_LENGTH + t.INDEX_LENGTH + t.DATA_FREE), 0)) < 0.1 
    THEN 1 ELSE 0 END +
  CASE WHEN (t.INDEX_LENGTH / NULLIF(t.DATA_LENGTH, 0)) BETWEEN 0.1 AND 0.5 
    THEN 1 ELSE 0 END +
  CASE WHEN (
    SELECT COUNT(DISTINCT INDEX_NAME) 
    FROM information_schema.STATISTICS s 
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
      AND s.TABLE_NAME = t.TABLE_NAME
  ) BETWEEN 1 AND 10 THEN 1 ELSE 0 END +
  CASE WHEN t.ENGINE = 'InnoDB' THEN 1 ELSE 0 END) AS health_score,
  
  CASE 
    WHEN (CASE WHEN EXISTS (
      SELECT 1 FROM information_schema.STATISTICS s 
      WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
        AND s.TABLE_NAME = t.TABLE_NAME 
        AND s.INDEX_NAME = 'PRIMARY'
    ) THEN 1 ELSE 0 END +
    CASE WHEN (t.DATA_FREE / NULLIF((t.DATA_LENGTH + t.INDEX_LENGTH + t.DATA_FREE), 0)) < 0.1 
      THEN 1 ELSE 0 END +
    CASE WHEN (t.INDEX_LENGTH / NULLIF(t.DATA_LENGTH, 0)) BETWEEN 0.1 AND 0.5 
      THEN 1 ELSE 0 END +
    CASE WHEN (
      SELECT COUNT(DISTINCT INDEX_NAME) 
      FROM information_schema.STATISTICS s 
      WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
        AND s.TABLE_NAME = t.TABLE_NAME
    ) BETWEEN 1 AND 10 THEN 1 ELSE 0 END +
    CASE WHEN t.ENGINE = 'InnoDB' THEN 1 ELSE 0 END) >= 4 
    THEN '✓ Excellent'
    WHEN (CASE WHEN EXISTS (
      SELECT 1 FROM information_schema.STATISTICS s 
      WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
        AND s.TABLE_NAME = t.TABLE_NAME 
        AND s.INDEX_NAME = 'PRIMARY'
    ) THEN 1 ELSE 0 END +
    CASE WHEN (t.DATA_FREE / NULLIF((t.DATA_LENGTH + t.INDEX_LENGTH + t.DATA_FREE), 0)) < 0.1 
      THEN 1 ELSE 0 END +
    CASE WHEN (t.INDEX_LENGTH / NULLIF(t.DATA_LENGTH, 0)) BETWEEN 0.1 AND 0.5 
      THEN 1 ELSE 0 END +
    CASE WHEN (
      SELECT COUNT(DISTINCT INDEX_NAME) 
      FROM information_schema.STATISTICS s 
      WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA 
        AND s.TABLE_NAME = t.TABLE_NAME
    ) BETWEEN 1 AND 10 THEN 1 ELSE 0 END +
    CASE WHEN t.ENGINE = 'InnoDB' THEN 1 ELSE 0 END) >= 3 
    THEN '⚠️ Bon'
    ELSE '❌ Nécessite attention'
  END AS status
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← À REMPLACER
  AND t.TABLE_TYPE = 'BASE TABLE'
ORDER BY health_score ASC, row_count DESC;
```

---

### 6.2 - Rapport Optimisation Complet
**Niveau** : 🔴 Avancé

```sql
-- ============================================================
-- Générer rapport recommandations optimisation
-- ============================================================

SELECT 'OPTIMIZATIONS RECOMMANDÉES' AS category, '' AS detail, '' AS priority
UNION ALL

-- Tables sans PK
SELECT 
  '1. Tables sans Primary Key',
  CONCAT(COUNT(*), ' table(s)'),
  '🔴 CRITIQUE'
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND NOT EXISTS (
    SELECT 1 FROM information_schema.STATISTICS s
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
      AND s.TABLE_NAME = t.TABLE_NAME
      AND s.INDEX_NAME = 'PRIMARY'
  )

UNION ALL

-- Tables fragmentées
SELECT 
  '2. Tables fragmentées (>10%)',
  CONCAT(COUNT(*), ' table(s), ', ROUND(SUM(DATA_FREE)/1024/1024, 0), ' MB récupérables'),
  '⚠️ MOYEN'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND ENGINE = 'InnoDB'
  AND (DATA_FREE / NULLIF((DATA_LENGTH + INDEX_LENGTH + DATA_FREE), 0)) > 0.1

UNION ALL

-- FK sans index
SELECT 
  '3. Foreign Keys sans index',
  CONCAT(COUNT(*), ' FK(s)'),
  '⚠️ MOYEN'
FROM information_schema.KEY_COLUMN_USAGE kcu
WHERE kcu.REFERENCED_TABLE_NAME IS NOT NULL
  AND kcu.TABLE_SCHEMA NOT IN ('mysql', 'sys')
  AND NOT EXISTS (
    SELECT 1 FROM information_schema.STATISTICS s
    WHERE s.TABLE_SCHEMA = kcu.TABLE_SCHEMA
      AND s.TABLE_NAME = kcu.TABLE_NAME
      AND s.COLUMN_NAME = kcu.COLUMN_NAME
      AND s.SEQ_IN_INDEX = 1
  )

UNION ALL

-- Buffer pool hit rate
SELECT 
  '4. Buffer Pool Hit Rate',
  CONCAT(
    ROUND((1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100, 2), '%'
  ),
  CASE 
    WHEN (1 - 
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100 < 99 THEN '🔴 CRITIQUE'
    ELSE '✓ OK'
  END

ORDER BY priority DESC, category;
```

---

## ✅ Checklist Optimisation

### Analyse Périodique
```sql
☐ Vérifier index inutilisés (mensuel)
☐ Analyser fragmentation (hebdomadaire)
☐ Audit cardinalité index (mensuel)
☐ Review slow queries digest (quotidien)
☐ Vérifier croissance tables (hebdomadaire)
```

### Actions Correctives
```sql
☐ OPTIMIZE tables fragmentées >15%
☐ Supprimer index redondants/inutilisés
☐ ANALYZE TABLE après gros imports
☐ Ajouter index sur FK columns
☐ Corriger tables sans PK
```

### Monitoring Continu
```sql
☐ Buffer pool hit rate >99%
☐ Slow query ratio <1%
☐ Temp tables sur disque = 0
☐ Table scans minimaux
```

---

## 🔗 Ressources

### Liens Vers Autres Sections
- **[← C.1 Requêtes d'Administration](./01-requetes-administration.md)** : Locks, processlist, users
- **[← C.2 Requêtes de Monitoring](./02-requetes-monitoring.md)** : Buffer pool, slow queries, réplication

### Documentation
- [OPTIMIZE TABLE](https://mariadb.com/kb/en/optimize-table/)
- [ANALYZE TABLE](https://mariadb.com/kb/en/analyze-table/)
- [Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [Index Hints](https://mariadb.com/kb/en/index-hints/)

### Outils Complémentaires
- **pt-duplicate-key-checker** : Détection index redondants
- **pt-index-usage** : Analyse utilisation index
- **pt-online-schema-change** : ALTER TABLE sans downtime
- **MySQLTuner** : Script audit configuration

---

**MariaDB** : 11.8 LTS

⏭️ [Configuration de Référence par Cas d'Usage](/annexes/configuration-reference/README.md)

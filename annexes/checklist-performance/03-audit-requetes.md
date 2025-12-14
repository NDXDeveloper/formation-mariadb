🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.3 Audit de Requêtes

> **Type** : Checklist d'audit queries SQL  
> **Focus** : Slow queries, N+1, optimisations, anti-patterns  
> **Durée** : 2-4 heures  
> **Prérequis** : Slow query log activé, Performance Schema ON

---

## 🎯 Objectifs de cet audit

Vérifier que les **requêtes SQL** sont optimales pour :
- ✅ Toutes les queries < seuil latence (1s OLTP, 10s OLAP)
- ✅ Aucun anti-pattern (N+1, SELECT *, sub-queries inefficaces)
- ✅ Utilisation efficace des index et plans d'exécution
- ✅ Pas de temporary tables sur disque
- ✅ Code applicatif optimisé (ORM, batching)

**Résultat attendu :** Top 10-20 queries à optimiser avec impact chiffré et solutions concrètes.

---

## 📋 Checklist complète (18 points)

### Catégorie 1 : Identification Slow Queries

#### ✅ 1.1 Top Slow Queries par temps total

**🔍 Diagnostic**

```sql
-- Top 20 queries par temps total (Performance Schema)
SELECT 
    LEFT(DIGEST_TEXT, 100) AS query_preview,
    COUNT_STAR AS executions,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec,
    ROUND(MAX_TIMER_WAIT / 1000000000000, 3) AS max_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec,
    ROUND(SUM_LOCK_TIME / 1000000000000, 2) AS total_lock_sec,
    SUM_ROWS_EXAMINED AS rows_examined,
    SUM_ROWS_SENT AS rows_sent,
    ROUND(SUM_ROWS_EXAMINED / SUM_ROWS_SENT, 0) AS rows_examined_per_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME NOT IN ('mysql', 'performance_schema', 'information_schema')
  AND DIGEST_TEXT NOT LIKE '%performance_schema%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

**Analyse slow query log :**

```bash
# Avec pt-query-digest (Percona Toolkit)
pt-query-digest /var/log/mysql/slow-query.log \
  --limit 20 \
  --order-by Query_time:sum

# Top queries par temps total, fréquence, variance
```

**⚠️ Seuils critiques**

| Métrique | 🟢 OLTP | 🟡 OLTP | 🔴 OLTP | 🟢 OLAP |
|----------|---------|---------|---------|---------|
| **Latence P95** | < 50ms | 50-100ms | > 100ms | < 5s |
| **Latence moyenne** | < 10ms | 10-50ms | > 50ms | < 2s |
| **Rows examined/sent** | 1-10× | 10-100× | > 100× | Variable |

**🔧 Actions correctives**

1. **Prioriser par impact** :
   ```
   Impact = executions × avg_latency
   
   Query A : 1000 exec/sec × 100ms = 100s/sec
   Query B : 10 exec/sec × 5s = 50s/sec
   
   → Optimiser A en priorité (impact 2× plus grand)
   ```

2. **Catégoriser slow queries** :
   ```
   Catégorie 1 : Missing index → Créer index (E.2)
   Catégorie 2 : Query mal écrite → Refactoring SQL
   Catégorie 3 : Volumétrie → Partitionnement, archivage
   Catégorie 4 : Lock contention → Optimistic locking
   ```

3. **Template analyse** :
   ```sql
   -- Pour chaque slow query :
   
   -- 1. EXPLAIN
   EXPLAIN FORMAT=JSON SELECT ...;
   
   -- 2. Identifier goulot
   -- - Full table scan ?
   -- - Filesort ?
   -- - Temporary table ?
   
   -- 3. Actions
   -- - Ajouter index
   -- - Rewrite query
   -- - Partitionner table
   ```

**📊 Validation**

```sql
-- Comparer avant/après optimisation
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS exec_before,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec_before
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%target_query%';

-- Après optimisation, reset stats
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;

-- Attendre 24h, re-mesurer
-- avg_sec_after devrait être << avg_sec_before
```

---

#### ✅ 1.2 Queries avec full table scans

**🔍 Diagnostic**

```sql
-- Queries sans index (SUM_NO_INDEX_USED > 0)
SELECT 
    LEFT(DIGEST_TEXT, 120) AS query,
    COUNT_STAR AS executions,
    SUM_NO_INDEX_USED AS full_scans,
    SUM_NO_GOOD_INDEX_USED AS suboptimal_index,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
   OR SUM_NO_GOOD_INDEX_USED > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

**EXPLAIN queries suspectes :**

```sql
-- Exemple query problématique
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- ❌ Signaux alerte :
-- type: ALL (full table scan)
-- rows: 1000000 (examine 1M lignes)
-- Extra: Using where (filtre après scan)
```

**⚠️ Seuils critiques**

| Type scan | OLTP | OLAP | Action |
|-----------|------|------|--------|
| **Index scan** | ✅ OK | ✅ OK | - |
| **Range scan** | ✅ OK | ✅ OK | - |
| **Full table scan (< 1K rows)** | ✅ OK | ✅ OK | - |
| **Full table scan (> 10K rows)** | 🔴 Problème | 🟡 Acceptable | Index |
| **Full table scan (> 1M rows)** | 🔴 Critique | 🟡 Normal | Index/Partition |

**🔧 Actions correctives**

1. **Ajouter index manquant** :
   ```sql
   -- Query lente : WHERE status = 'pending'
   EXPLAIN SELECT * FROM orders WHERE status = 'pending';
   -- type: ALL ❌
   
   -- Créer index
   CREATE INDEX idx_orders_status ON orders(status);
   
   -- Vérifier
   EXPLAIN SELECT * FROM orders WHERE status = 'pending';
   -- type: ref ✅
   -- rows: 1000 (au lieu de 1M)
   ```

2. **Index composite pour WHERE multiples** :
   ```sql
   -- WHERE status = X AND created_at > Y
   CREATE INDEX idx_status_created ON orders(status, created_at);
   
   -- Index utilisé complètement
   ```

3. **Cas OLAP : Full scan acceptable** :
   ```sql
   -- Analytics : Agrégation table entière
   SELECT DATE(created_at), COUNT(*)
   FROM orders
   WHERE created_at >= '2024-01-01'
   GROUP BY DATE(created_at);
   
   -- Full scan normal si > 10% table scannée
   -- Pas d'optimisation nécessaire
   ```

**📊 Validation**

```sql
-- Avant/après : Rows examined
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- Avant index :
-- rows: 1000000

-- Après index :
-- rows: 1000 (1000× réduction)
```

---

#### ✅ 1.3 Queries avec filesort/temporary tables

**🔍 Diagnostic**

```sql
-- Queries créant temp tables ou filesort
SELECT 
    LEFT(DIGEST_TEXT, 100) AS query,
    COUNT_STAR AS executions,
    SUM_SORT_MERGE_PASSES AS filesort_on_disk,
    SUM_SORT_ROWS AS rows_sorted,
    SUM_CREATED_TMP_TABLES AS tmp_tables,
    SUM_CREATED_TMP_DISK_TABLES AS tmp_tables_on_disk,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_SORT_MERGE_PASSES > 0
   OR SUM_CREATED_TMP_DISK_TABLES > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

**EXPLAIN pour identifier :**

```sql
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- Extra: Using filesort ❌

EXPLAIN SELECT customer_id, COUNT(*)
FROM orders
GROUP BY customer_id;
-- Extra: Using temporary ❌
```

**⚠️ Seuils critiques**

| Opération | 🟢 Optimal | 🟡 Acceptable | 🔴 Critique |
|-----------|-----------|---------------|-------------|
| **Filesort** | Index utilisé | Mémoire (RAM) | Sur disque |
| **Temp table** | Pas de temp | Mémoire | Sur disque |
| **Sort merge passes** | 0 | < 10/sec | > 100/sec |

**🔧 Actions correctives**

1. **Éliminer filesort avec index** :
   ```sql
   -- Query : ORDER BY created_at DESC
   CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
   
   EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
   -- Extra: Backward index scan ✅ (pas de filesort)
   ```

2. **Temp table → Index covering** :
   ```sql
   -- Query lente avec temp table
   SELECT customer_id, status, COUNT(*)
   FROM orders
   WHERE created_at > '2024-01-01'
   GROUP BY customer_id, status;
   
   -- Index covering élimine temp table
   CREATE INDEX idx_created_customer_status 
   ON orders(created_at, customer_id, status);
   
   -- Extra: Using index ✅
   ```

3. **Augmenter buffers si impossible éviter** :
   ```sql
   -- Si filesort/temp inévitable, augmenter buffers
   SET SESSION sort_buffer_size = 16777216;  -- 16MB
   SET SESSION tmp_table_size = 134217728;   -- 128MB
   
   -- Ou global (my.cnf)
   -- sort_buffer_size = 16M
   -- tmp_table_size = 128M
   ```

**📊 Validation**

```sql
-- Vérifier élimination dans EXPLAIN
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- Avant :
-- Extra: Using filesort

-- Après index :
-- Extra: Backward index scan (pas de filesort ✅)
```

---

### Catégorie 2 : N+1 Queries Problem

#### ✅ 2.1 Détection pattern N+1

**🔍 Diagnostic**

Le problème N+1 est **invisible dans Performance Schema** (queries individuelles rapides). Requiert analyse code application.

**Symptômes :**
```python
# ❌ Code application avec N+1
users = db.query("SELECT * FROM users LIMIT 100")  # 1 query
for user in users:
    posts = db.query(f"SELECT * FROM posts WHERE user_id = {user.id}")  # N queries
    # 100 queries supplémentaires !

# Total : 101 queries (1 + 100)
```

**Détection via monitoring :**

```sql
-- Spike queries identiques répétées
-- Pattern suspect dans Performance Schema
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    COUNT_STAR / (SELECT MAX(variable_value) FROM information_schema.global_status WHERE variable_name = 'Uptime') AS queries_per_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%WHERE user_id%'
ORDER BY COUNT_STAR DESC;

-- Si COUNT_STAR très élevé pour query simple → suspect N+1
```

**Outils code :**
- **Django** : django-debug-toolbar (affiche queries)
- **Rails** : Bullet gem (détecte N+1)
- **Node.js** : Dataloader (batching)
- **Laravel** : Debugbar, Telescope

**⚠️ Seuils critiques**

| Scénario | Queries | Impact | Action |
|----------|---------|--------|--------|
| **Liste 10 items** | 11 queries (1+10) | 🟡 Acceptable | Optimiser si fréquent |
| **Liste 100 items** | 101 queries | 🔴 Problème | JOIN ou batching |
| **Liste 1000+ items** | 1001+ queries | 🔴 Critique | Refactoring urgent |

**🔧 Actions correctives**

1. **Solution 1 : JOIN** :
   ```sql
   -- ❌ N+1 (2 queries)
   SELECT * FROM users WHERE id = 1;
   SELECT * FROM posts WHERE user_id = 1;
   
   -- ✅ JOIN (1 query)
   SELECT 
       u.id, u.name,
       p.id AS post_id, p.title
   FROM users u
   LEFT JOIN posts p ON u.id = p.user_id
   WHERE u.id = 1;
   ```

2. **Solution 2 : Eager Loading (ORM)** :
   ```python
   # ❌ Lazy loading (N+1)
   users = User.objects.all()
   for user in users:
       posts = user.posts.all()  # Query par user
   
   # ✅ Eager loading
   users = User.objects.prefetch_related('posts').all()
   # 2 queries optimisées :
   # 1. SELECT * FROM users
   # 2. SELECT * FROM posts WHERE user_id IN (1,2,3,...)
   ```

3. **Solution 3 : IN clause batching** :
   ```python
   # ❌ N queries
   for user_id in user_ids:
       posts = db.query(f"SELECT * FROM posts WHERE user_id = {user_id}")
   
   # ✅ 1 query avec IN
   posts = db.query(f"SELECT * FROM posts WHERE user_id IN ({','.join(map(str, user_ids))})")
   # Group by user_id en application
   ```

**📊 Validation**

```python
# Avant optimisation
import time
start = time.time()
# Code N+1
duration_n_plus_1 = time.time() - start
# Exemple : 2.5 secondes (101 queries)

# Après optimisation (JOIN ou eager loading)
start = time.time()
# Code optimisé
duration_optimized = time.time() - start
# Exemple : 0.05 secondes (1-2 queries)

# Gain : 50× plus rapide
```

---

#### ✅ 2.2 Batching et bulk operations

**🔍 Diagnostic**

```python
# ❌ Anti-pattern : INSERT en boucle
for record in records:  # 1000 records
    db.execute("INSERT INTO logs (message) VALUES (?)", record)
    # 1000 INSERTs individuels = lent
```

**Mesurer impact :**

```sql
-- Compter INSERTs individuels vs bulk
SHOW GLOBAL STATUS LIKE 'Com_insert';
-- Si Com_insert très élevé : suspect batching manquant
```

**⚠️ Seuils critiques**

| Opération | Individuel | Batch | Gain |
|-----------|------------|-------|------|
| **100 INSERTs** | 1s | 50ms | 20× |
| **1000 INSERTs** | 10s | 200ms | 50× |
| **10000 INSERTs** | 100s | 2s | 50× |

**🔧 Actions correctives**

1. **Bulk INSERT** :
   ```sql
   -- ❌ 1000 queries
   INSERT INTO logs (message) VALUES ('log1');
   INSERT INTO logs (message) VALUES ('log2');
   -- ...
   
   -- ✅ 1 query
   INSERT INTO logs (message) VALUES
   ('log1'),
   ('log2'),
   ('log3'),
   -- ... 1000 rows
   ('log1000');
   
   -- Ou LOAD DATA INFILE (encore plus rapide)
   LOAD DATA LOCAL INFILE '/tmp/logs.csv'
   INTO TABLE logs
   FIELDS TERMINATED BY ','
   LINES TERMINATED BY '\n';
   ```

2. **Bulk UPDATE** :
   ```sql
   -- ❌ 100 UPDATEs individuels
   UPDATE products SET price = 10 WHERE id = 1;
   UPDATE products SET price = 20 WHERE id = 2;
   -- ...
   
   -- ✅ CASE statement (1 query)
   UPDATE products
   SET price = CASE id
       WHEN 1 THEN 10
       WHEN 2 THEN 20
       WHEN 3 THEN 30
       -- ...
       ELSE price
   END
   WHERE id IN (1, 2, 3, ...);
   ```

3. **Transaction batching** :
   ```sql
   -- ❌ Auto-commit chaque INSERT (overhead)
   INSERT INTO logs VALUES (...);  -- Commit
   INSERT INTO logs VALUES (...);  -- Commit
   
   -- ✅ Transaction unique
   START TRANSACTION;
   INSERT INTO logs VALUES (...);
   INSERT INTO logs VALUES (...);
   -- ... 1000 INSERTs
   COMMIT;  -- 1 seul commit
   ```

**📊 Validation**

```bash
# Benchmark INSERT
time python insert_individual.py
# 10.5 secondes

time python insert_bulk.py
# 0.2 secondes (50× plus rapide)
```

---

### Catégorie 3 : Query Anti-Patterns

#### ✅ 3.1 SELECT * (éviter)

**🔍 Diagnostic**

```sql
-- Rechercher SELECT * dans queries
SELECT 
    LEFT(DIGEST_TEXT, 150) AS query
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%SELECT %*%FROM%'
  AND DIGEST_TEXT NOT LIKE '%COUNT%'
LIMIT 20;
```

**Problèmes SELECT * :**

```sql
-- ❌ SELECT *
SELECT * FROM users WHERE id = 1;
-- Problèmes :
-- 1. Transfert données inutiles (bio TEXT 10KB, avatar BLOB 500KB)
-- 2. Pas de covering index possible
-- 3. Fragile (schema change casse code)
```

**⚠️ Seuils critiques**

| Table | SELECT * | SELECT colonnes | Gain |
|-------|----------|-----------------|------|
| **10 colonnes, 50KB row** | 50KB | 200 bytes | 250× |
| **Table avec BLOB** | 1MB+ | 100 bytes | 10000× |

**🔧 Actions correctives**

1. **Sélection explicite colonnes** :
   ```sql
   -- ❌ Mauvais
   SELECT * FROM users WHERE id = 1;
   -- Retourne : id, email, name, bio(10KB), avatar(500KB), ...
   
   -- ✅ Bon
   SELECT id, email, name FROM users WHERE id = 1;
   -- Retourne : 3 colonnes, < 1KB
   ```

2. **Covering index possible** :
   ```sql
   -- Avec SELECT *
   SELECT * FROM orders WHERE customer_id = 123;
   -- Index : idx_customer_id ne peut pas être covering
   -- Table lookup requis
   
   -- Avec sélection explicite
   SELECT id, total_amount FROM orders WHERE customer_id = 123;
   -- Index covering : idx_customer_id_amount (customer_id, id, total_amount)
   -- Pas de table lookup ✅
   ```

3. **ORM : Specify fields** :
   ```python
   # ❌ Django ORM : SELECT *
   users = User.objects.all()
   
   # ✅ Specify fields
   users = User.objects.values('id', 'email', 'name')
   # Ou
   users = User.objects.only('id', 'email', 'name')
   ```

**📊 Validation**

```sql
-- Mesurer taille résultat
SELECT 
    LENGTH(CONCAT_WS(',', *)) AS row_size_bytes
FROM users
WHERE id = 1;
-- Avec SELECT * : 512000 bytes (500KB)

SELECT 
    LENGTH(CONCAT_WS(',', id, email, name)) AS row_size_bytes
FROM users
WHERE id = 1;
-- Avec colonnes spécifiques : 150 bytes (3000× moins)
```

---

#### ✅ 3.2 Fonctions sur colonnes indexées (éviter)

**🔍 Diagnostic**

```sql
-- Queries avec fonctions sur colonnes WHERE
-- Rendent index inutilisable

-- ❌ Fonction sur colonne indexée
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- Index idx_created_at NON utilisé (fonction empêche)

EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- type: ALL (full scan malgré index)
```

**Anti-patterns courants :**

```sql
-- ❌ DATE/TIME functions
WHERE YEAR(created_at) = 2024
WHERE DATE(created_at) = '2024-01-01'
WHERE DATE_FORMAT(created_at, '%Y-%m') = '2024-01'

-- ❌ String functions
WHERE LOWER(email) = 'test@example.com'
WHERE SUBSTRING(name, 1, 3) = 'Bob'

-- ❌ Math functions
WHERE price * 1.2 > 100
WHERE ABS(balance) < 1000
```

**⚠️ Seuils critiques**

| Pattern | Index utilisé ? | Performance |
|---------|-----------------|-------------|
| `WHERE col = value` | ✅ Oui | ⚡ Rapide |
| `WHERE func(col) = value` | ❌ Non | 🐌 Lent (full scan) |
| `WHERE col LIKE 'prefix%'` | ✅ Oui | ⚡ Rapide |
| `WHERE col LIKE '%suffix'` | ❌ Non | 🐌 Lent |

**🔧 Actions correctives**

1. **Réécrire sans fonction** :
   ```sql
   -- ❌ Fonction sur colonne
   WHERE YEAR(created_at) = 2024
   
   -- ✅ Range (index utilisable)
   WHERE created_at >= '2024-01-01' 
     AND created_at < '2025-01-01'
   
   -- Index idx_created_at utilisé ✅
   ```

2. **Colonne générée pour fonction** :
   ```sql
   -- Si fonction fréquente, créer colonne générée
   ALTER TABLE orders 
   ADD COLUMN created_year INT GENERATED ALWAYS AS (YEAR(created_at)) STORED;
   
   CREATE INDEX idx_created_year ON orders(created_year);
   
   -- Query rapide
   SELECT * FROM orders WHERE created_year = 2024;
   -- Index utilisé ✅
   ```

3. **Normalisation pour LOWER()** :
   ```sql
   -- ❌ Case-insensitive search
   WHERE LOWER(email) = 'test@example.com'
   
   -- ✅ Stocker lowercase dès l'insertion
   CREATE TRIGGER before_insert_user
   BEFORE INSERT ON users
   FOR EACH ROW
   SET NEW.email = LOWER(NEW.email);
   
   -- Query simple
   WHERE email = 'test@example.com'
   -- Index utilisé ✅
   ```

**📊 Validation**

```sql
-- Comparer plans d'exécution
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- type: ALL, rows: 1000000 ❌

EXPLAIN SELECT * FROM orders 
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
-- type: range, rows: 50000, key: idx_created_at ✅
```

---

#### ✅ 3.3 OR conditions (optimisation)

**🔍 Diagnostic**

```sql
-- OR peut empêcher utilisation optimale index
SELECT * FROM users 
WHERE email = 'test@example.com' 
   OR username = 'testuser';

EXPLAIN ...;
-- type: index_merge (parfois) ou ALL (pire cas)
```

**⚠️ Seuils critiques**

| Pattern | Index usage | Performance |
|---------|-------------|-------------|
| `WHERE a = X OR b = Y` (index sur a, b) | Index merge | 🟡 Moyen |
| `WHERE a = X OR a = Y` | Index scan | ✅ Bon |
| `WHERE a = X OR unindexed = Y` | Full scan | 🔴 Mauvais |

**🔧 Actions correctives**

1. **UNION ALL pour OR** :
   ```sql
   -- ❌ OR avec index merge
   SELECT * FROM users 
   WHERE email = 'test@example.com' 
      OR username = 'testuser';
   
   -- ✅ UNION ALL (2 index scans séparés)
   SELECT * FROM users WHERE email = 'test@example.com'
   UNION ALL
   SELECT * FROM users WHERE username = 'testuser';
   
   -- Parfois 2-3× plus rapide
   ```

2. **IN pour OR sur même colonne** :
   ```sql
   -- ❌ Multiple OR
   WHERE status = 'pending' 
      OR status = 'processing' 
      OR status = 'shipped'
   
   -- ✅ IN clause
   WHERE status IN ('pending', 'processing', 'shipped')
   -- Index utilisé efficacement
   ```

3. **Index composite pour OR fréquent** :
   ```sql
   -- Si query fréquente : WHERE a = X OR b = Y
   CREATE INDEX idx_a_b ON table(a, b);
   
   -- Ou mieux : index séparés + index merge
   CREATE INDEX idx_a ON table(a);
   CREATE INDEX idx_b ON table(b);
   -- Optimizer utilise index_merge
   ```

**📊 Validation**

```sql
-- Comparer OR vs UNION ALL
EXPLAIN SELECT * FROM users WHERE email = 'a' OR username = 'b';
-- type: index_merge, rows: 200

EXPLAIN (
    SELECT * FROM users WHERE email = 'a'
    UNION ALL
    SELECT * FROM users WHERE username = 'b'
);
-- Deux index scans séparés, rows: 1 + 1
```

---

### Catégorie 4 : Sub-queries et JOINs

#### ✅ 4.1 Sub-queries corrélées (éviter)

**🔍 Diagnostic**

```sql
-- Sub-query corrélée (exécutée pour CHAQUE ligne)
SELECT 
    o.id,
    o.total,
    (SELECT name FROM customers c WHERE c.id = o.customer_id) AS customer_name
FROM orders o;

-- EXPLAIN : Dependent subquery ❌
```

**Performance impact :**

```
Table orders : 100K lignes
→ Sub-query exécutée 100K fois
→ Très lent (même avec index sur customers)

Vs JOIN :
→ 1 seule opération
→ 100-1000× plus rapide
```

**⚠️ Seuils critiques**

| Type sub-query | Performance | Recommandation |
|----------------|-------------|----------------|
| **Non-corrélée** | ✅ Bon | OK (exécutée 1×) |
| **Corrélée simple** | 🟡 Moyen | Remplacer par JOIN |
| **Corrélée complexe** | 🔴 Mauvais | Refactoring urgent |

**🔧 Actions correctives**

1. **Remplacer par JOIN** :
   ```sql
   -- ❌ Sub-query corrélée
   SELECT 
       o.id,
       (SELECT name FROM customers WHERE id = o.customer_id)
   FROM orders o;
   
   -- ✅ JOIN
   SELECT 
       o.id,
       c.name
   FROM orders o
   JOIN customers c ON o.customer_id = c.id;
   
   -- 100-1000× plus rapide
   ```

2. **Sub-query non-corrélée OK** :
   ```sql
   -- ✅ Non-corrélée (exécutée 1 seule fois)
   SELECT * FROM orders
   WHERE customer_id IN (
       SELECT id FROM customers WHERE country = 'France'
   );
   
   -- Équivalent JOIN, performance similaire
   SELECT o.* FROM orders o
   JOIN customers c ON o.customer_id = c.id
   WHERE c.country = 'France';
   ```

3. **Derived table vs sub-query** :
   ```sql
   -- Sub-query dans FROM (derived table)
   SELECT *
   FROM (
       SELECT customer_id, SUM(total) AS total_spent
       FROM orders
       GROUP BY customer_id
   ) AS customer_totals
   WHERE total_spent > 1000;
   
   -- Equivalent CTE (plus lisible)
   WITH customer_totals AS (
       SELECT customer_id, SUM(total) AS total_spent
       FROM orders
       GROUP BY customer_id
   )
   SELECT * FROM customer_totals WHERE total_spent > 1000;
   ```

**📊 Validation**

```sql
-- Benchmark sub-query vs JOIN
-- Sub-query corrélée
SELECT BENCHMARK(100, (
    SELECT (SELECT name FROM customers WHERE id = o.customer_id)
    FROM orders o LIMIT 1000
));
-- 5.2 secondes

-- JOIN
SELECT BENCHMARK(100, (
    SELECT c.name FROM orders o JOIN customers c ON o.customer_id = c.id LIMIT 1000
));
-- 0.3 secondes (17× plus rapide)
```

---

#### ✅ 4.2 JOIN efficace (ordre et type)

**🔍 Diagnostic**

```sql
-- Analyser ordre JOINs
EXPLAIN FORMAT=JSON
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE o.created_at > '2024-01-01';
```

**Ordre JOINs important :**

```
Optimizer choisit ordre JOIN optimal MAIS :
- Statistiques parfois obsolètes
- Queries complexes > 6 tables : ordre sub-optimal

Règle manuelle :
1. Table avec WHERE restrictif d'abord (réduit dataset)
2. Petite table avant grosse table
3. Index disponibles
```

**⚠️ Seuils critiques**

| Type JOIN | Use case | Performance |
|-----------|----------|-------------|
| **INNER JOIN** | Intersection | ⚡ Rapide |
| **LEFT JOIN (small → large)** | Extension | ✅ Bon |
| **LEFT JOIN (large → small)** | Extension | 🟡 Moyen |
| **CROSS JOIN** | Produit cartésien | 🔴 Éviter |

**🔧 Actions correctives**

1. **Straight_JOIN pour forcer ordre** :
   ```sql
   -- Forcer ordre JOIN si optimizer se trompe
   SELECT STRAIGHT_JOIN
       o.*,
       c.name
   FROM small_table o
   JOIN large_table c ON o.customer_id = c.id
   WHERE o.status = 'active';  -- Très sélectif
   
   -- Force small_table d'abord (réduit dataset)
   ```

2. **EXISTS vs IN pour semi-join** :
   ```sql
   -- IN (matérialise sub-query)
   SELECT * FROM customers
   WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);
   
   -- EXISTS (short-circuit possible)
   SELECT * FROM customers c
   WHERE EXISTS (
       SELECT 1 FROM orders o 
       WHERE o.customer_id = c.id AND o.total > 1000
   );
   
   -- EXISTS souvent plus rapide pour grosses tables
   ```

3. **Index pour JOIN conditions** :
   ```sql
   -- JOIN sur colonnes non indexées
   SELECT *
   FROM orders o
   JOIN shipments s ON o.order_number = s.order_number;
   
   -- Créer index sur colonnes JOIN
   CREATE INDEX idx_orders_order_number ON orders(order_number);
   CREATE INDEX idx_shipments_order_number ON shipments(order_number);
   
   -- JOIN 10-100× plus rapide
   ```

**📊 Validation**

```sql
-- Comparer ordres JOIN
EXPLAIN SELECT ...;
-- Regarder "table" column dans ordre d'exécution
-- Si mauvais ordre : utiliser STRAIGHT_JOIN
```

---

### Catégorie 5 : LIMIT et Pagination

#### ✅ 5.1 LIMIT avec OFFSET élevé (éviter)

**🔍 Diagnostic**

```sql
-- Pagination avec OFFSET élevé
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100000;

-- ❌ MySQL/MariaDB lit 100020 lignes puis skip 100000
-- Très lent pour grandes offsets
```

**Performance degradation :**

```
OFFSET 0     : 10ms
OFFSET 1000  : 50ms
OFFSET 10000 : 500ms
OFFSET 100000: 5000ms (5s) ❌

Relation linéaire : offset 10× plus grand = 10× plus lent
```

**⚠️ Seuils critiques**

| OFFSET | Performance | Action |
|--------|-------------|--------|
| **< 1000** | ✅ OK | Standard pagination |
| **1000-10000** | 🟡 Moyen | Cursor-based |
| **> 10000** | 🔴 Lent | Cursor obligatoire |

**🔧 Actions correctives**

1. **Cursor-based pagination (keyset)** :
   ```sql
   -- ❌ OFFSET pagination
   SELECT * FROM orders 
   ORDER BY created_at DESC 
   LIMIT 20 OFFSET 100000;
   -- Lit 100020 lignes
   
   -- ✅ Cursor-based
   -- Page 1
   SELECT * FROM orders 
   ORDER BY created_at DESC 
   LIMIT 20;
   -- Retourne : last_created_at = '2024-01-15 10:00:00'
   
   -- Page 2
   SELECT * FROM orders 
   WHERE created_at < '2024-01-15 10:00:00'
   ORDER BY created_at DESC 
   LIMIT 20;
   -- Lit seulement 20 lignes ✅
   ```

2. **Index pour cursor pagination** :
   ```sql
   -- Index sur colonne ORDER BY
   CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
   
   -- Cursor pagination ultra rapide (même OFFSET "élevé")
   ```

3. **Infinite scroll vs numbered pages** :
   ```
   ❌ Numbered pages : "Page 5000" (OFFSET 100000)
   ✅ Infinite scroll : "Load more" (cursor-based)
   
   UI adaptation pour meilleure performance
   ```

**📊 Validation**

```sql
-- Benchmark OFFSET vs cursor
-- OFFSET 100000
SELECT BENCHMARK(100, (
    SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000
));
-- 50 secondes

-- Cursor
SELECT BENCHMARK(100, (
    SELECT * FROM orders WHERE id < 123456 ORDER BY id DESC LIMIT 20
));
-- 0.5 secondes (100× plus rapide)
```

---

## 📊 Script d'audit automatisé

### audit-queries.sql

```sql
-- ============================================================================
-- SCRIPT D'AUDIT REQUÊTES MARIADB 11.8
-- ============================================================================

SELECT '=' AS '', '=' AS '';
SELECT 'AUDIT REQUÊTES SQL' AS '';
SELECT '=' AS '', '=' AS '';

-- 1. TOP 10 SLOW QUERIES
SELECT '\n1. TOP 10 QUERIES PAR TEMPS TOTAL' AS '';
SELECT 
    LEFT(DIGEST_TEXT, 100) AS query,
    COUNT_STAR AS exec,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec,
    ROUND(SUM_ROWS_EXAMINED / SUM_ROWS_SENT, 0) AS rows_ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME NOT IN ('mysql', 'performance_schema', 'information_schema')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- 2. QUERIES SANS INDEX
SELECT '\n2. QUERIES SANS INDEX (🔴 CRITIQUE)' AS '';
SELECT 
    LEFT(DIGEST_TEXT, 100) AS query,
    COUNT_STAR AS exec,
    SUM_NO_INDEX_USED AS no_index,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- 3. FILESORT ET TEMP TABLES
SELECT '\n3. FILESORT / TEMP TABLES' AS '';
SELECT 
    LEFT(DIGEST_TEXT, 100) AS query,
    SUM_SORT_MERGE_PASSES AS filesort_disk,
    SUM_CREATED_TMP_DISK_TABLES AS tmp_disk,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_SORT_MERGE_PASSES > 0 OR SUM_CREATED_TMP_DISK_TABLES > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- 4. QUERIES AVEC SELECT *
SELECT '\n4. QUERIES AVEC SELECT *' AS '';
SELECT 
    LEFT(DIGEST_TEXT, 100) AS query,
    COUNT_STAR AS exec
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%SELECT%*%FROM%'
  AND DIGEST_TEXT NOT LIKE '%COUNT%'
ORDER BY COUNT_STAR DESC
LIMIT 10;

SELECT '=' AS '', '=' AS '';
SELECT 'FIN AUDIT REQUÊTES' AS '';
```

---

## ✅ Points clés à retenir

- 🎯 **Top 10 queries = 80% impact** : Focus sur queries les plus utilisées
- 🔍 **EXPLAIN systématique** : Analyser AVANT déploiement production
- ❌ **N+1 = #1 problème** : JOIN ou eager loading obligatoire
- 📦 **Bulk operations** : 50× plus rapide que loops
- 🚫 **SELECT * interdit** : Sélection colonnes explicite
- ⚙️ **Fonctions sur colonnes** : Rendent index inutilisable
- 🔀 **Sub-queries corrélées** : Remplacer par JOIN
- 📄 **Cursor pagination** : OFFSET élevé = anti-pattern
- 📊 **Performance Schema** : Monitoring continu requis
- 🔧 **Refactoring > Hardware** : Optimiser code avant scaler

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Optimization and Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)
- [EXPLAIN](https://mariadb.com/kb/en/explain/)
- [Subquery Optimizations](https://mariadb.com/kb/en/subquery-optimizations/)

### Outils
- [pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)
- [Performance Schema](https://mariadb.com/kb/en/performance-schema/)

### Autres checklists
- [E.1 - Audit de configuration](./01-audit-configuration.md)
- [E.2 - Audit d'indexation](./02-audit-indexation.md)
- [E.4 - Audit de schéma](./04-audit-schema.md)

---

## ➡️ Section suivante

**[E.4 - Audit de schéma](./04-audit-schema.md)** : Design tables, normalisation, partitionnement

---

**MariaDB** : Version 11.8 LTS

⏭️ [Audit de schéma](/annexes/checklist-performance/04-audit-schema.md)

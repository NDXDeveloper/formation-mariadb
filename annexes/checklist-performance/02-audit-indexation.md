🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.2 Audit d'Indexation

> **Type** : Checklist d'audit indexes  
> **Focus** : Stratégie indexation, efficacité, redondances  
> **Durée** : 1-2 heures  
> **Prérequis** : Accès base de données, Performance Schema activé

---

## 🎯 Objectifs de cet audit

Vérifier que la **stratégie d'indexation** est optimale pour :
- ✅ Toutes les colonnes WHERE/JOIN ont des index appropriés
- ✅ Aucun index inutilisé ne gaspille espace et ralentit writes
- ✅ Pas de doublons/redondances dans les index
- ✅ Index composites ordonnés correctement
- ✅ Types d'index adaptés aux use cases (B-Tree, Full-Text, Spatial, Vector)

**Résultat attendu :** Plan d'action priorisé pour optimiser indexation (ajout, suppression, refonte).

---

## 📋 Checklist complète (15 points)

### Catégorie 1 : Missing Indexes (Index Manquants)

#### ✅ 1.1 Colonnes WHERE sans index

**🔍 Diagnostic**

```sql
-- Requêtes sans index depuis Performance Schema
SELECT 
    DIGEST_TEXT AS query,
    COUNT_STAR AS executions,
    SUM_NO_INDEX_USED AS no_index_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 3) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
  AND SCHEMA_NAME NOT IN ('mysql', 'performance_schema', 'information_schema')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Ou depuis slow query log
-- SELECT * FROM table WHERE non_indexed_column = 'value'
-- grep "WHERE" /var/log/mysql/slow-query.log
```

**Identifier colonnes candidates :**

```sql
-- Analyser patterns WHERE dans queries lentes
-- Exemple : Table users
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
-- type: ALL (full scan) ❌
-- rows: 1000000
-- Extra: Using where

-- Index manquant sur email !
```

**⚠️ Seuils critiques**

| Métrique | 🟢 Optimal | 🟡 Acceptable | 🔴 Critique |
|----------|-----------|---------------|-------------|
| **Queries sans index** | 0 | < 5% total | > 10% total |
| **Full table scans** | 0 (OLTP)<br>OK (OLAP) | < 10/sec | > 100/sec |
| **Rows examinées/retournées** | 1-10× | 10-100× | > 1000× |

**🔧 Actions correctives**

1. **Ajouter index sur colonnes WHERE fréquentes** :
   ```sql
   -- Exemple : email utilisé dans WHERE
   CREATE INDEX idx_users_email ON users(email);
   
   -- Vérifier impact
   EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
   -- type: ref ✅
   -- rows: 1
   -- key: idx_users_email
   ```

2. **Index composites pour WHERE multiples** :
   ```sql
   -- Query fréquente : WHERE status = 'active' AND created_at > '2024-01-01'
   CREATE INDEX idx_status_created ON orders(status, created_at);
   
   -- Ordre important : colonne égalité d'abord, puis range
   ```

3. **🆕 MariaDB 11.8 : Invisible indexes pour test** :
   ```sql
   -- Tester index sans impact production
   CREATE INDEX idx_test ON table(column) INVISIBLE;
   
   -- Activer temporairement pour session
   SET optimizer_switch = 'use_invisible_indexes=on';
   EXPLAIN SELECT ...;
   
   -- Si bénéfique, rendre visible
   ALTER TABLE table ALTER INDEX idx_test VISIBLE;
   ```

**📊 Validation**

```sql
-- Avant/après : Comparer plan d'exécution
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'test@example.com'\G

-- Mesurer impact Performance Schema
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec_before
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%users%email%';

-- Créer index, attendre 24h, re-mesurer
-- avg_sec_after devrait être << avg_sec_before
```

**💡 Notes**

- **OLTP** : Toute query WHERE sans index = BUG critique
- **OLAP** : Full scans parfois normaux (analytics massives)
- **Priorité** : Index colonnes fréquentes (80/20 rule)

---

#### ✅ 1.2 Colonnes JOIN sans index

**🔍 Diagnostic**

```sql
-- Identifier JOINs sans index (EXPLAIN queries)
-- Exemple problématique :
EXPLAIN 
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';

-- Si "Using where; Using join buffer" → Pas d'index sur FK
```

**Audit systématique Foreign Keys :**

```sql
-- Trouver FK sans index
SELECT 
    kcu.TABLE_SCHEMA AS db,
    kcu.TABLE_NAME AS table_name,
    kcu.COLUMN_NAME AS fk_column,
    kcu.REFERENCED_TABLE_NAME AS ref_table,
    kcu.REFERENCED_COLUMN_NAME AS ref_column,
    (SELECT COUNT(*) 
     FROM information_schema.statistics s
     WHERE s.TABLE_SCHEMA = kcu.TABLE_SCHEMA
       AND s.TABLE_NAME = kcu.TABLE_NAME
       AND s.COLUMN_NAME = kcu.COLUMN_NAME) AS index_exists
FROM information_schema.key_column_usage kcu
WHERE kcu.REFERENCED_TABLE_NAME IS NOT NULL
  AND kcu.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema')
HAVING index_exists = 0;
```

**⚠️ Seuils critiques**

| Situation | Impact | Priorité |
|-----------|--------|----------|
| **FK sans index** | JOIN lent (nested loop) | 🔴 Critique |
| **JOIN 1M+ lignes sans index** | Query minutes au lieu de secondes | 🔴 Critique |
| **JOIN tables petites (<1000 lignes)** | Impact faible | 🟡 Basse |

**🔧 Actions correctives**

1. **Index sur toutes les FK (règle d'or)** :
   ```sql
   -- orders.customer_id référence customers.id
   CREATE INDEX idx_orders_customer_id ON orders(customer_id);
   
   -- posts.user_id référence users.id
   CREATE INDEX idx_posts_user_id ON posts(user_id);
   ```

2. **Index composites pour JOIN + WHERE** :
   ```sql
   -- Query : JOIN + WHERE
   SELECT o.*, c.name
   FROM orders o
   JOIN customers c ON o.customer_id = c.id
   WHERE o.status = 'pending' AND o.created_at > '2024-01-01';
   
   -- Index composite couvre JOIN + WHERE
   CREATE INDEX idx_orders_customer_status_created 
   ON orders(customer_id, status, created_at);
   ```

3. **Vérifier cardinalité FK** :
   ```sql
   -- Si FK faible cardinalité, index moins utile
   SELECT 
       customer_id,
       COUNT(*) AS orders_per_customer
   FROM orders
   GROUP BY customer_id
   ORDER BY COUNT(*) DESC
   LIMIT 10;
   
   -- Si max orders/customer < 10 : index moins critique
   -- Si max > 1000 : index TRÈS critique
   ```

**📊 Validation**

```sql
-- EXPLAIN avant/après
EXPLAIN SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';

-- Avant :
-- type: ALL (full scan)
-- Extra: Using where; Using join buffer

-- Après index :
-- type: ref
-- key: idx_orders_customer_id
-- Extra: Using where
```

---

#### ✅ 1.3 Colonnes ORDER BY / GROUP BY sans index

**🔍 Diagnostic**

```sql
-- Queries avec filesort (tri sur disque)
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_SORT_MERGE_PASSES AS filesort_on_disk,
    SUM_SORT_ROWS AS rows_sorted,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_SORT_MERGE_PASSES > 0
ORDER BY SUM_SORT_MERGE_PASSES DESC
LIMIT 10;

-- EXPLAIN query avec ORDER BY
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- Extra: Using filesort ❌
```

**⚠️ Seuils critiques**

| Métrique | 🟢 Optimal | 🟡 Acceptable | 🔴 Critique |
|----------|-----------|---------------|-------------|
| **Filesort** | 0 (index utilisé) | Mémoire (RAM) | Sur disque |
| **Sort merge passes** | 0 | < 100/sec | > 1000/sec |
| **Rows sorted** | < 1000 | < 100K | > 1M |

**🔧 Actions correctives**

1. **Index sur colonnes ORDER BY** :
   ```sql
   -- Query fréquente : ORDER BY created_at DESC
   CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
   
   -- MariaDB optimise tri avec index descendant
   EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
   -- Extra: Backward index scan ✅
   ```

2. **Index composite WHERE + ORDER BY** :
   ```sql
   -- Query : WHERE + ORDER BY
   SELECT * FROM orders 
   WHERE status = 'pending' 
   ORDER BY created_at DESC 
   LIMIT 10;
   
   -- Index composite (WHERE d'abord, ORDER BY ensuite)
   CREATE INDEX idx_orders_status_created 
   ON orders(status, created_at DESC);
   ```

3. **GROUP BY avec index** :
   ```sql
   -- GROUP BY bénéficie aussi d'index
   SELECT customer_id, COUNT(*) 
   FROM orders 
   GROUP BY customer_id;
   
   -- Index sur customer_id évite filesort
   CREATE INDEX idx_orders_customer_id ON orders(customer_id);
   ```

**📊 Validation**

```sql
-- Vérifier élimination filesort
EXPLAIN SELECT * FROM orders 
WHERE status = 'pending' 
ORDER BY created_at DESC;

-- Avant :
-- Extra: Using where; Using filesort

-- Après index :
-- Extra: Using where; Backward index scan
-- (pas de filesort ✅)
```

---

### Catégorie 2 : Unused Indexes (Index Inutilisés)

#### ✅ 2.1 Index jamais utilisés

**🔍 Diagnostic**

```sql
-- Index inutilisés (Performance Schema requis)
SELECT 
    t.TABLE_SCHEMA AS db,
    t.TABLE_NAME AS table_name,
    t.INDEX_NAME AS index_name,
    t.INDEX_TYPE,
    t.SEQ_IN_INDEX AS column_position,
    t.COLUMN_NAME,
    ROUND((s.DATA_LENGTH + s.INDEX_LENGTH) / 1024 / 1024, 2) AS table_size_mb
FROM information_schema.statistics t
JOIN information_schema.tables s 
  ON t.TABLE_SCHEMA = s.TABLE_SCHEMA 
  AND t.TABLE_NAME = s.TABLE_NAME
LEFT JOIN performance_schema.table_io_waits_summary_by_index_usage p
  ON t.TABLE_SCHEMA = p.OBJECT_SCHEMA
  AND t.TABLE_NAME = p.OBJECT_NAME
  AND t.INDEX_NAME = p.INDEX_NAME
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema')
  AND t.INDEX_NAME != 'PRIMARY'
  AND (p.COUNT_STAR IS NULL OR p.COUNT_STAR = 0)
ORDER BY table_size_mb DESC, t.TABLE_NAME, t.INDEX_NAME;
```

**⚠️ Seuils critiques**

| Situation | Action | Priorité |
|-----------|--------|----------|
| **Index 0 utilisations > 1 mois** | Supprimer | 🔴 Haute |
| **Table > 100GB avec index inutilisé** | Supprimer urgent | 🔴 Critique |
| **Table < 1GB, index récent** | Surveiller | 🟡 Basse |

**🔧 Actions correctives**

1. **Supprimer index inutilisés** :
   ```sql
   -- Vérifier index candidat
   SELECT 
       COUNT_STAR,
       SUM_TIMER_WAIT
   FROM performance_schema.table_io_waits_summary_by_index_usage
   WHERE OBJECT_SCHEMA = 'mydb'
     AND OBJECT_NAME = 'orders'
     AND INDEX_NAME = 'idx_old_column';
   -- COUNT_STAR = 0 → jamais utilisé
   
   -- Supprimer (attention : peut impacter queries futures)
   DROP INDEX idx_old_column ON orders;
   ```

2. **Index redondants avec PRIMARY KEY** :
   ```sql
   -- ❌ Redondant
   CREATE TABLE users (
       id BIGINT PRIMARY KEY,
       email VARCHAR(255),
       INDEX idx_id (id)  -- ❌ Inutile, PRIMARY déjà index
   );
   
   -- Supprimer
   DROP INDEX idx_id ON users;
   ```

3. **Invisible d'abord, supprimer ensuite** :
   ```sql
   -- Rendre invisible pour tester impact
   ALTER TABLE orders ALTER INDEX idx_suspect INVISIBLE;
   
   -- Attendre 1 semaine, surveiller
   -- Si aucun problème → supprimer
   DROP INDEX idx_suspect ON orders;
   ```

**📊 Validation**

```sql
-- Espace récupéré
SELECT 
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb_before
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders';

-- Après DROP INDEX
-- size_mb_after < size_mb_before

-- Vérifier performance writes
SHOW GLOBAL STATUS LIKE 'Handler_write';
-- Devrait augmenter (moins d'index à maintenir)
```

---

#### ✅ 2.2 Index avec très faible sélectivité

**🔍 Diagnostic**

```sql
-- Cardinalité des colonnes indexées
SELECT 
    t.TABLE_SCHEMA AS db,
    t.TABLE_NAME,
    t.INDEX_NAME,
    s.CARDINALITY,
    (SELECT COUNT(*) FROM information_schema.tables tab
     WHERE tab.TABLE_SCHEMA = t.TABLE_SCHEMA 
       AND tab.TABLE_NAME = t.TABLE_NAME) AS total_rows,
    ROUND(100.0 * s.CARDINALITY / 
          (SELECT TABLE_ROWS FROM information_schema.tables tab
           WHERE tab.TABLE_SCHEMA = t.TABLE_SCHEMA 
             AND tab.TABLE_NAME = t.TABLE_NAME), 2) AS selectivity_pct
FROM information_schema.statistics t
JOIN information_schema.statistics s
  ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
  AND t.TABLE_NAME = s.TABLE_NAME
  AND t.INDEX_NAME = s.INDEX_NAME
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema')
  AND t.INDEX_NAME != 'PRIMARY'
  AND t.SEQ_IN_INDEX = 1  -- Première colonne index
HAVING selectivity_pct < 10  -- Moins de 10% sélectivité
ORDER BY selectivity_pct;
```

**Exemple problématique :**

```sql
-- Table avec 1M lignes
-- Colonne status : 3 valeurs possibles ('pending', 'completed', 'cancelled')
CREATE INDEX idx_status ON orders(status);
-- Cardinalité : 3
-- Sélectivité : 3 / 1000000 = 0.0003% ❌

-- Optimizer ignore souvent cet index (full scan plus rapide)
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- type: ALL (full scan malgré index !)
```

**⚠️ Seuils critiques**

| Sélectivité | Efficacité | Action |
|-------------|------------|--------|
| **> 30%** | 🟢 Excellente | Garder |
| **10-30%** | 🟡 Moyenne | Évaluer cas par cas |
| **< 10%** | 🔴 Faible | Considérer suppression |
| **< 1%** | 🔴 Très faible | Supprimer (sauf cas spéciaux) |

**🔧 Actions correctives**

1. **Supprimer index faible sélectivité** :
   ```sql
   -- Colonne booléenne (2 valeurs) → Sélectivité 0.0002%
   DROP INDEX idx_is_active ON users;
   
   -- Full scan souvent plus rapide que index scan
   ```

2. **Remplacer par index composite** :
   ```sql
   -- Au lieu de index seul sur status (faible)
   -- Combiner avec colonne haute sélectivité
   CREATE INDEX idx_status_created ON orders(status, created_at);
   
   -- Utile pour WHERE status = X ORDER BY created_at
   ```

3. **Cas spécial : Index partiel** (MariaDB ne supporte pas, workaround) :
   ```sql
   -- MySQL 8.0+ : CREATE INDEX idx ON table(col) WHERE col = 'rare_value'
   -- MariaDB : Utiliser WHERE dans requête
   
   -- Ou créer table partitionnée par valeur status
   ```

**📊 Validation**

```sql
-- Vérifier utilisation réelle index
SELECT 
    INDEX_NAME,
    COUNT_STAR AS uses,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'mydb'
  AND OBJECT_NAME = 'orders'
  AND INDEX_NAME = 'idx_status';

-- Si COUNT_STAR faible et total_sec négligeable → supprimer
```

---

### Catégorie 3 : Duplicate Indexes (Index Redondants)

#### ✅ 3.1 Index entièrement dupliqués

**🔍 Diagnostic**

```sql
-- Détecter index identiques (même colonnes, même ordre)
SELECT 
    t1.TABLE_SCHEMA,
    t1.TABLE_NAME,
    t1.INDEX_NAME AS index1,
    t2.INDEX_NAME AS index2,
    GROUP_CONCAT(t1.COLUMN_NAME ORDER BY t1.SEQ_IN_INDEX) AS columns
FROM information_schema.statistics t1
JOIN information_schema.statistics t2
  ON t1.TABLE_SCHEMA = t2.TABLE_SCHEMA
  AND t1.TABLE_NAME = t2.TABLE_NAME
  AND t1.INDEX_NAME < t2.INDEX_NAME
  AND t1.SEQ_IN_INDEX = t2.SEQ_IN_INDEX
  AND t1.COLUMN_NAME = t2.COLUMN_NAME
WHERE t1.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema')
GROUP BY t1.TABLE_SCHEMA, t1.TABLE_NAME, t1.INDEX_NAME, t2.INDEX_NAME
HAVING COUNT(*) = (
    SELECT COUNT(*) 
    FROM information_schema.statistics s
    WHERE s.TABLE_SCHEMA = t1.TABLE_SCHEMA
      AND s.TABLE_NAME = t1.TABLE_NAME
      AND s.INDEX_NAME = t1.INDEX_NAME
);
```

**Outil Percona :**

```bash
# pt-duplicate-key-checker (plus fiable)
pt-duplicate-key-checker --host=localhost --user=root --password=xxx

# Output exemple :
# mydb.orders
# Key idx_customer_id (customer_id) is a duplicate of idx_customer_id_2 (customer_id)
```

**⚠️ Seuils critiques**

| Type duplication | Impact | Action |
|------------------|--------|--------|
| **100% identique** | Gaspillage pur | 🔴 Supprimer immédiatement |
| **Préfixe dupliqué** | Redondance partielle | 🟡 Évaluer (voir 3.2) |

**🔧 Actions correctives**

1. **Supprimer doublons exacts** :
   ```sql
   -- ❌ Problème
   CREATE INDEX idx_email ON users(email);
   CREATE INDEX idx_email_unique ON users(email);  -- Doublon !
   
   -- Solution
   DROP INDEX idx_email_unique ON users;
   -- Garder idx_email ou créer UNIQUE INDEX
   ALTER TABLE users ADD UNIQUE INDEX idx_email_uniq (email);
   ```

2. **Vérifier contraintes vs index** :
   ```sql
   -- UNIQUE constraint crée automatiquement un index
   ALTER TABLE users ADD UNIQUE (email);
   -- → Index implicite créé
   
   -- ❌ Ne pas ajouter index séparé
   -- CREATE INDEX idx_email ON users(email);  -- Redondant !
   ```

3. **PRIMARY KEY = index automatique** :
   ```sql
   -- ❌ Redondant
   CREATE TABLE products (
       id BIGINT PRIMARY KEY,
       INDEX idx_id (id)  -- Inutile !
   );
   
   -- PRIMARY KEY crée déjà un index sur id
   ```

**📊 Validation**

```sql
-- Lister tous les index d'une table
SHOW INDEXES FROM users;

-- Vérifier pas de doublons dans résultat
-- Column_name identique dans plusieurs Index_name = suspect
```

---

#### ✅ 3.2 Index redondants (préfixe)

**🔍 Diagnostic**

```sql
-- Index où colonnes initiales sont identiques
-- Exemple :
-- idx_a (col1)
-- idx_ab (col1, col2)  ← idx_a est redondant

SELECT 
    t1.TABLE_SCHEMA,
    t1.TABLE_NAME,
    t1.INDEX_NAME AS redundant_index,
    t2.INDEX_NAME AS covers_it,
    GROUP_CONCAT(t1.COLUMN_NAME ORDER BY t1.SEQ_IN_INDEX) AS redundant_cols,
    GROUP_CONCAT(t2.COLUMN_NAME ORDER BY t2.SEQ_IN_INDEX) AS covering_cols
FROM information_schema.statistics t1
JOIN information_schema.statistics t2
  ON t1.TABLE_SCHEMA = t2.TABLE_SCHEMA
  AND t1.TABLE_NAME = t2.TABLE_NAME
  AND t1.INDEX_NAME != t2.INDEX_NAME
  AND t1.SEQ_IN_INDEX = t2.SEQ_IN_INDEX
  AND t1.COLUMN_NAME = t2.COLUMN_NAME
WHERE t1.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema')
GROUP BY t1.TABLE_SCHEMA, t1.TABLE_NAME, t1.INDEX_NAME, t2.INDEX_NAME;
```

**Exemple concret :**

```sql
-- ❌ Redondance
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_email_name ON users(email, name);

-- idx_email est redondant !
-- idx_email_name peut servir pour WHERE email = X
-- Leftmost prefix rule
```

**⚠️ Seuils critiques**

| Situation | Garder ? | Raison |
|-----------|----------|--------|
| **Index simple + composite même préfixe** | Composite | Simple redondant |
| **Composite + composite préfixe** | Plus long | Préfixe redondant |
| **UNIQUE vs non-UNIQUE** | Les deux | Contrainte ≠ performance |

**🔧 Actions correctives**

1. **Supprimer index simple si composite existe** :
   ```sql
   -- Avant
   CREATE INDEX idx_customer ON orders(customer_id);
   CREATE INDEX idx_customer_status ON orders(customer_id, status);
   
   -- Supprimer idx_customer (redondant)
   DROP INDEX idx_customer ON orders;
   
   -- idx_customer_status couvre WHERE customer_id = X
   -- grâce à leftmost prefix rule
   ```

2. **Exception : Taille index** :
   ```sql
   -- Si colonnes additionnelles volumineuses
   CREATE INDEX idx_email ON users(email);  -- 255 bytes
   CREATE INDEX idx_email_bio ON users(email, bio);  -- 255 + 5000 bytes
   
   -- Garder idx_email car plus compact (covering index faster)
   -- MAIS cas rare, généralement supprimer
   ```

3. **Ordre colonnes important** :
   ```sql
   -- ❌ Non redondants (ordre différent)
   CREATE INDEX idx_status_customer ON orders(status, customer_id);
   CREATE INDEX idx_customer_status ON orders(customer_id, status);
   
   -- Deux index différents, pas de redondance
   -- Chacun optimise queries différentes
   ```

**📊 Validation**

```sql
-- Tester queries avec index composite
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
-- key: idx_customer_status ✅ (utilise préfixe)

EXPLAIN SELECT * FROM orders 
WHERE customer_id = 123 AND status = 'pending';
-- key: idx_customer_status ✅ (utilise tout)

-- Index simple customer_id seul = redondant
```

---

### Catégorie 4 : Index Composites (Ordre des colonnes)

#### ✅ 4.1 Ordre optimal des colonnes

**🔍 Diagnostic**

```sql
-- Analyser requêtes fréquentes
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'mydb'
ORDER BY COUNT_STAR DESC
LIMIT 20;

-- Identifier patterns WHERE
-- Exemple : WHERE status = X AND created_at > Y ORDER BY created_at
```

**Règles d'ordre :**

```
Ordre optimal index composite :
1. Égalité (=) d'abord
2. Range (<, >, BETWEEN) ensuite
3. ORDER BY en dernier

Exemple :
WHERE status = 'active'       ← Égalité
  AND created_at > '2024-01-01'  ← Range
ORDER BY created_at DESC      ← Tri

Index optimal : (status, created_at DESC)
```

**⚠️ Seuils critiques**

| Pattern query | Index optimal | Index sub-optimal |
|---------------|---------------|-------------------|
| `WHERE a = X AND b = Y` | (a, b) ou (b, a) | N/A (équivalent) |
| `WHERE a = X AND b > Y` | (a, b) | (b, a) ❌ |
| `WHERE a > X AND b = Y` | (b, a) | (a, b) ❌ |
| `WHERE a = X ORDER BY b` | (a, b) | (b, a) ❌ |

**🔧 Actions correctives**

1. **Égalité avant range** :
   ```sql
   -- ❌ Mauvais ordre
   CREATE INDEX idx_bad ON orders(created_at, status);
   
   -- Query : WHERE status = 'active' AND created_at > '2024-01-01'
   -- Index utilisé partiellement (seulement created_at)
   
   -- ✅ Bon ordre
   DROP INDEX idx_bad ON orders;
   CREATE INDEX idx_good ON orders(status, created_at);
   -- Index utilisé complètement
   ```

2. **Cardinalité : Colonne sélective d'abord** :
   ```sql
   -- Colonne A : 1M valeurs distinctes (email)
   -- Colonne B : 3 valeurs distinctes (status)
   
   -- ✅ Optimal
   CREATE INDEX idx_email_status ON users(email, status);
   -- Filtrage efficace : 1M → 1 → 1/3
   
   -- ❌ Sub-optimal
   CREATE INDEX idx_status_email ON users(status, email);
   -- Filtrage moins efficace : 1M → 333K → 1
   ```

3. **ORDER BY + WHERE** :
   ```sql
   -- Query fréquente
   SELECT * FROM posts 
   WHERE user_id = 123 
   ORDER BY created_at DESC 
   LIMIT 10;
   
   -- Index optimal (WHERE d'abord, ORDER BY ensuite)
   CREATE INDEX idx_user_created ON posts(user_id, created_at DESC);
   
   -- EXPLAIN confirme : Using where; Backward index scan
   ```

**📊 Validation**

```sql
-- Test différents ordres
-- Ordre 1
CREATE INDEX idx_test1 ON orders(status, created_at);
EXPLAIN SELECT * FROM orders 
WHERE status = 'active' AND created_at > '2024-01-01';
-- rows: 1000

-- Ordre 2
DROP INDEX idx_test1 ON orders;
CREATE INDEX idx_test2 ON orders(created_at, status);
EXPLAIN SELECT * FROM orders 
WHERE status = 'active' AND created_at > '2024-01-01';
-- rows: 10000 (pire !)

-- Garder idx_test1
```

---

#### ✅ 4.2 Covering Indexes

**🔍 Diagnostic**

```sql
-- Queries nécessitant table lookup
EXPLAIN SELECT id, email, name FROM users WHERE email = 'test@example.com';
-- Extra: NULL (pas de "Using index" = table lookup)

-- Amélioration : Covering index
CREATE INDEX idx_email_id_name ON users(email, id, name);

EXPLAIN SELECT id, email, name FROM users WHERE email = 'test@example.com';
-- Extra: Using index ✅ (pas de table lookup)
```

**Avantages covering index :**
- Pas de lecture table (index suffit)
- 2-10× plus rapide (selon volumétrie)
- Moins d'I/O

**⚠️ Seuils critiques**

| Métrique | Avec covering | Sans covering |
|----------|---------------|---------------|
| **Logical reads** | 2-5 | 10-50 |
| **Latence** | 1ms | 5-10ms |
| **I/O** | Index only | Index + Table |

**🔧 Actions correctives**

1. **Ajouter colonnes SELECT à index** :
   ```sql
   -- Query fréquente
   SELECT id, status, total_amount 
   FROM orders 
   WHERE customer_id = 123;
   
   -- ❌ Index partiel
   CREATE INDEX idx_customer ON orders(customer_id);
   -- Table lookup requis pour status, total_amount
   
   -- ✅ Covering index
   DROP INDEX idx_customer ON orders;
   CREATE INDEX idx_customer_covering 
   ON orders(customer_id, id, status, total_amount);
   -- "Using index" dans EXPLAIN
   ```

2. **Balance taille vs performance** :
   ```sql
   -- ⚠️ Attention taille index
   -- Colonne BLOB/TEXT → Index énorme
   
   -- Mauvais : Ajouter description TEXT à index
   CREATE INDEX idx_bad ON products(category_id, description);
   -- Index gigantesque
   
   -- Bon : Seulement colonnes légères
   CREATE INDEX idx_good ON products(category_id, id, name, price);
   ```

3. **PRIMARY KEY inclusion automatique** :
   ```sql
   -- InnoDB inclut PRIMARY KEY automatiquement
   CREATE TABLE users (
       id BIGINT PRIMARY KEY,
       email VARCHAR(255),
       name VARCHAR(100)
   );
   
   CREATE INDEX idx_email ON users(email);
   -- Équivalent implicite : idx_email(email, id)
   -- InnoDB ajoute PK à tous les secondary indexes
   ```

**📊 Validation**

```sql
-- Vérifier "Using index" dans EXPLAIN
EXPLAIN SELECT id, email FROM users WHERE email LIKE 'test%';

-- Avant covering :
-- Extra: Using where

-- Après covering (email, id) :
-- Extra: Using where; Using index ✅
```

---

### Catégorie 5 : Types d'Index Spécialisés

#### ✅ 5.1 Full-Text Indexes

**🔍 Diagnostic**

```sql
-- Recherche textuelle sans Full-Text
SELECT * FROM articles WHERE content LIKE '%keyword%';
-- ❌ Full table scan, très lent

-- Vérifier si Full-Text existe
SHOW INDEXES FROM articles WHERE Index_type = 'FULLTEXT';
```

**⚠️ Seuils critiques**

| Use case | Type index | Performance |
|----------|------------|-------------|
| **Recherche LIKE '%keyword%'** | B-Tree ❌ | Full scan (lent) |
| **Recherche LIKE 'keyword%'** | B-Tree ✅ | Index scan (OK) |
| **Recherche mots multiples** | FULLTEXT ✅ | Optimisé |

**🔧 Actions correctives**

1. **Créer Full-Text index** :
   ```sql
   -- Table InnoDB ou MyISAM
   CREATE FULLTEXT INDEX idx_ft_content ON articles(content);
   
   -- Recherche avec MATCH...AGAINST
   SELECT * FROM articles 
   WHERE MATCH(content) AGAINST('keyword' IN NATURAL LANGUAGE MODE);
   
   -- 10-100× plus rapide que LIKE '%keyword%'
   ```

2. **Full-Text multi-colonnes** :
   ```sql
   -- Recherche titre + contenu
   CREATE FULLTEXT INDEX idx_ft_title_content 
   ON articles(title, content);
   
   SELECT * FROM articles 
   WHERE MATCH(title, content) AGAINST('MariaDB performance');
   ```

3. **Modes Full-Text** :
   ```sql
   -- Natural Language (défaut)
   MATCH(col) AGAINST('query' IN NATURAL LANGUAGE MODE)
   
   -- Boolean (operateurs +, -, *)
   MATCH(col) AGAINST('+MariaDB -MySQL' IN BOOLEAN MODE)
   
   -- Query Expansion (synonymes)
   MATCH(col) AGAINST('database' WITH QUERY EXPANSION)
   ```

**📊 Validation**

```sql
-- Benchmark LIKE vs FULLTEXT
-- LIKE
SELECT BENCHMARK(1000, (
    SELECT COUNT(*) FROM articles WHERE content LIKE '%performance%'
));
-- 15.2 secondes

-- FULLTEXT
SELECT BENCHMARK(1000, (
    SELECT COUNT(*) FROM articles 
    WHERE MATCH(content) AGAINST('performance')
));
-- 0.8 secondes (20× plus rapide)
```

---

#### ✅ 5.2 🆕 Vector Indexes (HNSW) - MariaDB 11.8

**🔍 Diagnostic**

```sql
-- Vérifier support Vector
SHOW VARIABLES LIKE '%vector%';

-- Table avec embeddings IA
CREATE TABLE documents (
    id BIGINT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536) NOT NULL  -- OpenAI embeddings
);

-- Recherche sans index
SELECT id, VEC_DISTANCE_COSINE(embedding, target_vector) AS distance
FROM documents
ORDER BY distance
LIMIT 10;
-- Full scan 1M lignes = très lent
```

**⚠️ Seuils critiques**

| Dataset | Sans index | Avec HNSW | Gain |
|---------|------------|-----------|------|
| **10K vecteurs** | 2s | 50ms | 40× |
| **100K vecteurs** | 20s | 80ms | 250× |
| **1M vecteurs** | 200s | 120ms | 1600× |

**🔧 Actions correctives**

1. **Créer index HNSW** :
   ```sql
   -- 🆕 MariaDB 11.8 : Index vectoriel
   CREATE VECTOR INDEX idx_embedding_hnsw 
   ON documents(embedding)
   ALGORITHM=HNSW  -- Hierarchical Navigable Small Worlds
   M=16            -- Connexions par nœud (trade-off vitesse/précision)
   EF_CONSTRUCTION=200;  -- Précision construction
   
   -- Recherche vectorielle rapide
   SELECT id, VEC_DISTANCE_COSINE(embedding, target_vector) AS similarity
   FROM documents
   ORDER BY similarity
   LIMIT 10;
   -- 100ms vs 200s sans index
   ```

2. **Paramètres HNSW** :
   ```sql
   -- M (connexions) :
   -- - M = 8-16 : Rapide, moins précis
   -- - M = 32-64 : Précis, plus lent
   
   -- EF_CONSTRUCTION :
   -- - 100-200 : Bon compromis
   -- - 400-800 : Meilleure précision, construction lente
   
   -- Exemple haute précision
   CREATE VECTOR INDEX idx_embedding_precise
   ON documents(embedding)
   ALGORITHM=HNSW M=32 EF_CONSTRUCTION=400;
   ```

3. **Use cases Vector** :
   ```sql
   -- Semantic search (RAG)
   -- Recommendation engines
   -- Anomaly detection
   -- Image similarity
   
   -- Exemple RAG
   SELECT id, content, VEC_DISTANCE_COSINE(embedding, query_embedding) AS score
   FROM documents
   ORDER BY score
   LIMIT 5;
   -- Top 5 documents similaires pour context LLM
   ```

**📊 Validation**

```sql
-- Vérifier utilisation index
EXPLAIN SELECT id FROM documents
ORDER BY VEC_DISTANCE_COSINE(embedding, target_vector)
LIMIT 10;
-- key: idx_embedding_hnsw ✅
```

---

## 📊 Tableau récapitulatif priorités

### Matrice Impact/Effort

| Action | Impact | Effort | Priorité |
|--------|--------|--------|----------|
| **Ajouter index FK** | ⭐⭐⭐⭐⭐ | ⭐ | 🔴 P0 |
| **Ajouter index WHERE fréquent** | ⭐⭐⭐⭐⭐ | ⭐ | 🔴 P0 |
| **Supprimer index inutilisé** | ⭐⭐⭐ | ⭐ | 🟡 P1 |
| **Corriger ordre composite** | ⭐⭐⭐⭐ | ⭐⭐ | 🟡 P1 |
| **Covering index** | ⭐⭐⭐ | ⭐⭐ | 🟡 P1 |
| **Supprimer doublons** | ⭐⭐ | ⭐ | 🟢 P2 |
| **Full-Text search** | ⭐⭐⭐⭐ | ⭐⭐⭐ | 🟡 P1 |
| **Vector index (si IA)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 🔴 P0 (RAG) |

---

## 🤖 Script d'audit automatisé

### audit-indexes.sql

```sql
-- ============================================================================
-- SCRIPT D'AUDIT INDEXATION MARIADB 11.8
-- ============================================================================

SELECT '=' AS '', '=' AS '', '=' AS '';
SELECT 'AUDIT INDEXATION' AS '';
SELECT CONCAT('Base: ', DATABASE()) AS '';
SELECT '=' AS '', '=' AS '', '=' AS '';

-- 1. INDEX INUTILISÉS
SELECT '\n1. INDEX JAMAIS UTILISÉS (Candidats suppression)' AS '';
SELECT 
    CONCAT(t.TABLE_SCHEMA, '.', t.TABLE_NAME) AS table_name,
    t.INDEX_NAME,
    t.COLUMN_NAME,
    ROUND((s.INDEX_LENGTH) / 1024 / 1024, 2) AS index_size_mb,
    '🔴 SUPPRIMER ?' AS action
FROM information_schema.statistics t
JOIN information_schema.tables s 
  ON t.TABLE_SCHEMA = s.TABLE_SCHEMA 
  AND t.TABLE_NAME = s.TABLE_NAME
LEFT JOIN performance_schema.table_io_waits_summary_by_index_usage p
  ON t.TABLE_SCHEMA = p.OBJECT_SCHEMA
  AND t.TABLE_NAME = p.OBJECT_NAME
  AND t.INDEX_NAME = p.INDEX_NAME
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.INDEX_NAME != 'PRIMARY'
  AND (p.COUNT_STAR IS NULL OR p.COUNT_STAR = 0)
ORDER BY index_size_mb DESC
LIMIT 10;

-- 2. FOREIGN KEYS SANS INDEX
SELECT '\n2. FOREIGN KEYS SANS INDEX (🔴 CRITIQUE)' AS '';
SELECT 
    kcu.TABLE_NAME,
    kcu.COLUMN_NAME AS fk_column,
    kcu.REFERENCED_TABLE_NAME AS ref_table,
    '🔴 AJOUTER INDEX' AS action
FROM information_schema.key_column_usage kcu
WHERE kcu.REFERENCED_TABLE_NAME IS NOT NULL
  AND kcu.TABLE_SCHEMA = DATABASE()
  AND NOT EXISTS (
      SELECT 1 FROM information_schema.statistics s
      WHERE s.TABLE_SCHEMA = kcu.TABLE_SCHEMA
        AND s.TABLE_NAME = kcu.TABLE_NAME
        AND s.COLUMN_NAME = kcu.COLUMN_NAME
        AND s.SEQ_IN_INDEX = 1
  )
LIMIT 10;

-- 3. INDEX FAIBLE SÉLECTIVITÉ
SELECT '\n3. INDEX FAIBLE SÉLECTIVITÉ (< 10%)' AS '';
SELECT 
    t.TABLE_NAME,
    t.INDEX_NAME,
    s.CARDINALITY,
    tab.TABLE_ROWS AS total_rows,
    ROUND(100.0 * s.CARDINALITY / tab.TABLE_ROWS, 2) AS selectivity_pct,
    CASE 
        WHEN (100.0 * s.CARDINALITY / tab.TABLE_ROWS) < 1 THEN '🔴 TRÈS FAIBLE'
        WHEN (100.0 * s.CARDINALITY / tab.TABLE_ROWS) < 10 THEN '🟡 FAIBLE'
        ELSE '🟢 OK'
    END AS status
FROM information_schema.statistics t
JOIN information_schema.statistics s
  ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
  AND t.TABLE_NAME = s.TABLE_NAME
  AND t.INDEX_NAME = s.INDEX_NAME
JOIN information_schema.tables tab
  ON t.TABLE_SCHEMA = tab.TABLE_SCHEMA
  AND t.TABLE_NAME = tab.TABLE_NAME
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.INDEX_NAME != 'PRIMARY'
  AND t.SEQ_IN_INDEX = 1
  AND tab.TABLE_ROWS > 1000
HAVING selectivity_pct < 10
ORDER BY selectivity_pct
LIMIT 10;

-- 4. STATISTIQUES GLOBALES
SELECT '\n4. STATISTIQUES GLOBALES' AS '';
SELECT 
    COUNT(DISTINCT CONCAT(TABLE_SCHEMA, '.', TABLE_NAME)) AS total_tables,
    COUNT(DISTINCT INDEX_NAME) AS total_indexes,
    ROUND(SUM(INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS total_index_size_gb
FROM information_schema.statistics s
JOIN information_schema.tables t
  ON s.TABLE_SCHEMA = t.TABLE_SCHEMA
  AND s.TABLE_NAME = t.TABLE_NAME
WHERE s.TABLE_SCHEMA = DATABASE();

SELECT '=' AS '', '=' AS '', '=' AS '';
SELECT 'FIN AUDIT INDEXATION' AS '';
```

---

## ✅ Points clés à retenir

- 🎯 **Index sur FK obligatoire** : Toute Foreign Key doit avoir index
- 🔍 **Missing index = #1 problème** : WHERE sans index = full scan
- 🗑️ **Supprimer inutilisés** : Index 0 usage = gaspillage espace + slow writes
- 📐 **Ordre composite critique** : Égalité d'abord, range ensuite
- 📦 **Covering index** : "Using index" = 2-10× plus rapide
- ❌ **Éviter doublons** : pt-duplicate-key-checker périodique
- 📊 **Sélectivité < 10%** : Considérer suppression (sauf cas spéciaux)
- 🆕 **MariaDB 11.8 Vector** : HNSW pour embeddings IA/RAG
- 🔤 **Full-Text** : MATCH...AGAINST 10-100× plus rapide que LIKE '%x%'

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Optimization and Indexes](https://mariadb.com/kb/en/optimization-and-indexes/)
- [🆕 Vector Indexes](https://mariadb.com/kb/en/vector/)
- [Full-Text Indexes](https://mariadb.com/kb/en/fulltext-index-overview/)

### Outils
- [pt-duplicate-key-checker](https://www.percona.com/doc/percona-toolkit/LATEST/pt-duplicate-key-checker.html)
- [pt-index-usage](https://www.percona.com/doc/percona-toolkit/LATEST/pt-index-usage.html)

### Autres checklists
- [E.1 - Audit de configuration](./01-audit-configuration.md)
- [E.3 - Audit de requêtes](./03-audit-requetes.md)
- [E.4 - Audit de schéma](./04-audit-schema.md)

---

## ➡️ Section suivante

**[E.3 - Audit de requêtes](./03-audit-requetes.md)** : Slow queries, N+1, optimisations SQL

---

**MariaDB** : Version 11.8 LTS

⏭️ [Audit de requêtes](/annexes/checklist-performance/03-audit-requetes.md)

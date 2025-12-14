🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.1 Méthodologie d'optimisation

> **Niveau** : Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Expérience en administration MariaDB en production
> - Compréhension de l'architecture InnoDB
> - Connaissances en monitoring système (CPU, RAM, I/O)
> - Maîtrise du SQL et de l'analyse de requêtes

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Appliquer une méthodologie systématique** d'optimisation basée sur les données
- **Identifier rapidement les bottlenecks** (CPU, mémoire, I/O, réseau, application)
- **Prioriser les optimisations** selon leur impact et leur coût
- **Mesurer et valider** l'efficacité des changements en production
- **Établir un processus continu** de monitoring et d'amélioration
- **Éviter les pièges classiques** de l'optimisation prématurée ou mal ciblée

---

## Introduction

> 💡 **Citation de Donald Knuth** : "Premature optimization is the root of all evil (or at least most of it) in programming."

L'optimisation des performances d'une base de données en production est une **discipline exigeante** qui nécessite rigueur, méthode et patience. Contrairement aux idées reçues, optimiser efficacement ne consiste pas à :

- ❌ Copier des configurations "best practices" trouvées sur Internet  
- ❌ Modifier tous les paramètres en espérant améliorer les performances  
- ❌ Acheter du matériel plus puissant sans analyser le problème  
- ❌ Se fier à son intuition plutôt qu'aux métriques

✅ **La bonne approche** repose sur :
- Une **analyse factuelle** basée sur des métriques précises
- Une **compréhension approfondie** du workload et de l'architecture
- Des **changements incrémentaux** et mesurables
- Une **validation continue** des résultats
- Une **documentation rigoureuse** des modifications

### Pourquoi une méthodologie est cruciale

En production, chaque modification comporte des risques :

```
┌──────────────────────────────────────────────────────┐
│              RISQUES DE L'OPTIMISATION               │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Sans méthodologie          Avec méthodologie        │
│  ─────────────────          ─────────────────        │
│                                                      │
│  • Régression              • Amélioration mesurée    │
│    de performance            et validée              │
│                                                      │
│  • Instabilité             • Changements contrôlés   │
│    système                   et réversibles          │
│                                                      │
│  • Impossibilité de        • Traçabilité complète    │
│    rollback                  des modifications       │
│                                                      │
│  • Perte de temps          • Focus sur les vrais     │
│    sur faux problèmes        bottlenecks             │
│                                                      │
│  • Coûts matériels         • ROI optimisé            │
│    inutiles                                          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Contexte MariaDB 11.8 LTS

Avec MariaDB 11.8, certaines pratiques d'optimisation évoluent :

🆕 **Changements méthodologiques importants** :

1. **Cost optimizer SSD-aware** : L'optimiseur prend désormais en compte le type de stockage
2. **Métriques enrichies** : Performance Schema offre plus de granularité
3. **Configuration auto-adaptative** : Certains paramètres s'ajustent mieux automatiquement
4. **Nouveaux outils de diagnostic** : sys schema enrichi avec nouvelles vues

⚠️ **Impact sur la méthodologie** :
- Moins de tuning manuel nécessaire pour les configurations SSD
- Plus d'importance donnée à l'analyse applicative (requêtes, schéma)
- Validation plus rapide grâce aux nouveaux outils de monitoring

---

## Le cycle d'optimisation

### Modèle itératif en 6 phases

L'optimisation suit un cycle continu **OBSERVE → ANALYZE → OPTIMIZE → VALIDATE → DOCUMENT → MONITOR** :

```
        ┌─────────────────────────────────────────┐
        │         1. OBSERVE                      │
        │  Collecter métriques et données         │
        │  Établir baseline                       │
        └──────────────┬──────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────────┐
        │         2. ANALYZE                      │
        │  Identifier bottlenecks                 │
        │  Analyser patterns                      │
        └──────────────┬──────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────────┐
        │         3. OPTIMIZE                     │
        │  Hypothèse d'optimisation               │
        │  Changement ciblé                       │
        └──────────────┬──────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────────┐
        │         4. VALIDATE                     │
        │  Mesurer impact                         │
        │  Comparer avec baseline                 │
        └──────────────┬──────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────────┐
        │         5. DOCUMENT                     │
        │  Enregistrer changements                │
        │  Tracer résultats                       │
        └──────────────┬──────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────────┐
        │         6. MONITOR                      │
        │  Surveillance continue                  │
        │  Détection anomalies                    │
        └──────────────┬──────────────────────────┘
                       │
                       └──────────┐
                                  │
                       ┌──────────┘
                       ▼
                  Retour à OBSERVE si nécessaire
```

### Phase 1 : OBSERVE - Collecte des métriques

**Objectif** : Établir une baseline objective des performances actuelles.

#### Métriques système essentielles

```bash
# CPU
top -b -n 1 | grep Cpu
mpstat 1 10  # Si disponible

# Mémoire
free -h
vmstat 1 10

# I/O
iostat -x 1 10
iotop -o  # Processus les plus I/O intensifs

# Réseau (si pertinent)
iftop
nethogs
```

#### Métriques MariaDB critiques

```sql
-- 1. Vue d'ensemble système
SHOW GLOBAL STATUS;
SHOW GLOBAL VARIABLES;

-- 2. Métriques de performance clés
SELECT 
    'Questions' as metric,
    VARIABLE_VALUE as value
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Questions'
UNION ALL
SELECT 'Queries', VARIABLE_VALUE 
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Queries'
UNION ALL
SELECT 'Slow_queries', VARIABLE_VALUE 
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Slow_queries'
UNION ALL
SELECT 'Connections', VARIABLE_VALUE 
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Connections'
UNION ALL
SELECT 'Threads_connected', VARIABLE_VALUE 
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Threads_connected'
UNION ALL
SELECT 'Threads_running', VARIABLE_VALUE 
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Threads_running';

-- 3. État du Buffer Pool InnoDB
SELECT 
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') * 100, 2
    ) as buffer_pool_hit_rate_pct,
    
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_free') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') * 100, 2
    ) as buffer_pool_free_pct;

-- 4. Statistiques I/O
SELECT 
    'Data read (MB)' as metric,
    ROUND(VARIABLE_VALUE / 1024 / 1024, 2) as value
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_data_read'
UNION ALL
SELECT 'Data written (MB)', 
    ROUND(VARIABLE_VALUE / 1024 / 1024, 2)
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_data_written'
UNION ALL
SELECT 'Log writes', VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_log_writes';

-- 5. Connexions et threads
SHOW PROCESSLIST;

-- Version enrichie avec temps d'exécution
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state,
    LEFT(info, 100) as query_preview
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC
LIMIT 20;
```

#### Script de collecte baseline complet

```sql
-- Créer une table pour stocker les baselines
CREATE TABLE IF NOT EXISTS performance_baselines (
    id INT AUTO_INCREMENT PRIMARY KEY,
    collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metric_name VARCHAR(100),
    metric_value VARCHAR(255),
    metric_category ENUM('system', 'innodb', 'queries', 'connections', 'io'),
    INDEX idx_collected (collected_at),
    INDEX idx_metric (metric_name)
) ENGINE=InnoDB;

-- Procédure pour collecter la baseline
DELIMITER //
CREATE OR REPLACE PROCEDURE collect_baseline()
BEGIN
    DECLARE v_uptime INT;
    
    -- Récupérer uptime pour calculer les taux
    SELECT VARIABLE_VALUE INTO v_uptime
    FROM information_schema.GLOBAL_STATUS 
    WHERE VARIABLE_NAME = 'Uptime';
    
    -- Insérer métriques système
    INSERT INTO performance_baselines (metric_name, metric_value, metric_category)
    SELECT 
        VARIABLE_NAME,
        VARIABLE_VALUE,
        'system'
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME IN (
        'Uptime', 'Questions', 'Queries', 'Slow_queries',
        'Connections', 'Max_used_connections',
        'Threads_connected', 'Threads_running',
        'Aborted_connects', 'Aborted_clients'
    );
    
    -- Insérer métriques InnoDB
    INSERT INTO performance_baselines (metric_name, metric_value, metric_category)
    SELECT 
        VARIABLE_NAME,
        VARIABLE_VALUE,
        'innodb'
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME LIKE 'Innodb_%'
    AND VARIABLE_NAME IN (
        'Innodb_buffer_pool_read_requests',
        'Innodb_buffer_pool_reads',
        'Innodb_buffer_pool_pages_total',
        'Innodb_buffer_pool_pages_free',
        'Innodb_buffer_pool_pages_data',
        'Innodb_buffer_pool_pages_dirty',
        'Innodb_data_read',
        'Innodb_data_written',
        'Innodb_rows_read',
        'Innodb_rows_inserted',
        'Innodb_rows_updated',
        'Innodb_rows_deleted',
        'Innodb_row_lock_waits',
        'Innodb_row_lock_time_avg'
    );
    
    -- Calculer et insérer métriques dérivées
    INSERT INTO performance_baselines (metric_name, metric_value, metric_category)
    VALUES 
        ('queries_per_second', 
         (SELECT ROUND(VARIABLE_VALUE / v_uptime, 2) 
          FROM information_schema.GLOBAL_STATUS 
          WHERE VARIABLE_NAME = 'Queries'),
         'queries'),
        ('slow_queries_pct',
         (SELECT ROUND(
             (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Slow_queries') /
             (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Queries') * 100, 4)
          ),
         'queries');
    
    SELECT CONCAT('Baseline collected at ', NOW()) as result;
END //
DELIMITER ;

-- Exécuter la collecte
CALL collect_baseline();

-- Consulter la baseline
SELECT 
    metric_category,
    metric_name,
    metric_value,
    collected_at
FROM performance_baselines
WHERE collected_at = (SELECT MAX(collected_at) FROM performance_baselines)
ORDER BY metric_category, metric_name;
```

💡 **Conseil** : Collectez plusieurs baselines à différents moments (heures de pointe, heures creuses, différents jours de la semaine) pour avoir une vision complète du comportement du système.

### Phase 2 : ANALYZE - Identification des bottlenecks

**Objectif** : Déterminer quel composant limite les performances.

#### Les 5 types de bottlenecks

```
┌────────────────────────────────────────────────────────┐
│                  TYPOLOGIE DES BOTTLENECKS             │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. CPU           • Requêtes complexes non optimisées  │
│                   • Fonctions non indexées             │
│                   • Tri/groupement massifs             │
│                                                        │
│  2. MÉMOIRE       • Buffer pool trop petit             │
│                   • Temp tables sur disque             │
│                   • Swap actif                         │
│                                                        │
│  3. I/O           • Buffer pool misses                 │
│                   • Full table scans                   │
│                   • Disques saturés                    │
│                                                        │
│  4. RÉSEAU        • Transfert données massif           │
│                   • SELECT * sur large dataset         │
│                   • Connexions distantes               │
│                                                        │
│  5. APPLICATION   • N+1 queries                        │
│                   • Connexions non poolées             │
│                   • Transactions longues               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

#### Arbre de décision pour identifier le bottleneck

```sql
-- 1. CPU est-il le bottleneck ?
-- Si Threads_running élevé ET CPU usage >80% ET iostat %iowait <20%
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_running') as threads_running,
    (SELECT COUNT(*) FROM information_schema.processlist 
     WHERE command != 'Sleep' AND time > 1) as active_queries_over_1s;
-- Si threads_running constamment >10 et active_queries élevé → bottleneck CPU

-- 2. Mémoire est-elle le bottleneck ?
-- Si buffer pool hit rate <95% OU Created_tmp_disk_tables élevé
SELECT 
    -- Buffer pool hit rate
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'), 0) * 100, 2
    ) as bp_hit_rate_pct,
    
    -- Ratio temp tables sur disque
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Created_tmp_tables'), 0) * 100, 2
    ) as tmp_disk_pct;
-- Si bp_hit_rate < 95% → manque de mémoire buffer pool
-- Si tmp_disk_pct > 25% → manque tmp_table_size/max_heap_table_size

-- 3. I/O est-il le bottleneck ?
-- Vérifier à la fois système (iostat) et MariaDB
SELECT 
    'innodb_data_reads' as metric,
    VARIABLE_VALUE as total,
    ROUND(VARIABLE_VALUE / 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Uptime'), 2) as per_second
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_data_reads'
UNION ALL
SELECT 'innodb_data_writes',
    VARIABLE_VALUE,
    ROUND(VARIABLE_VALUE / 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Uptime'), 2)
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_data_writes';
-- Comparer avec iostat : si %util >80% → bottleneck I/O

-- 4. Locks et concurrence ?
SELECT 
    'Row_lock_waits' as metric,
    VARIABLE_VALUE as value
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_row_lock_waits'
UNION ALL
SELECT 'Row_lock_time_avg',
    VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_row_lock_time_avg';
-- Si row_lock_waits élevé → problème de concurrence

-- 5. Vue consolidée du diagnostic
SELECT 
    -- Performance générale
    CONCAT(
        ROUND((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
               WHERE VARIABLE_NAME = 'Queries') / 
              (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
               WHERE VARIABLE_NAME = 'Uptime'), 2),
        ' queries/sec'
    ) as throughput,
    
    -- CPU (approximation via threads)
    CONCAT(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Threads_running'),
        ' threads running'
    ) as cpu_indicator,
    
    -- Mémoire
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'), 0), 0),
        ':1 hit rate'
    ) as memory_indicator,
    
    -- I/O
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_data_reads') /
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Uptime'), 2),
        ' reads/sec'
    ) as io_indicator,
    
    -- Locks
    CONCAT(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_row_lock_current_waits'),
        ' current lock waits'
    ) as lock_indicator;
```

#### Utilisation de Performance Schema pour diagnostic avancé

```sql
-- Activer Performance Schema si pas déjà fait
UPDATE performance_schema.setup_instruments 
SET enabled = 'YES', timed = 'YES' 
WHERE name LIKE '%wait%' OR name LIKE '%stage%';

UPDATE performance_schema.setup_consumers 
SET enabled = 'YES' 
WHERE name LIKE '%events%';

-- Top 10 des waits par durée totale
SELECT 
    event_name,
    COUNT(*) as count,
    ROUND(SUM(timer_wait) / 1000000000000, 2) as total_wait_sec,
    ROUND(AVG(timer_wait) / 1000000000000, 6) as avg_wait_sec,
    ROUND(MAX(timer_wait) / 1000000000000, 6) as max_wait_sec
FROM performance_schema.events_waits_history_long
WHERE timer_wait IS NOT NULL
GROUP BY event_name
ORDER BY total_wait_sec DESC
LIMIT 10;

-- Identifier les requêtes qui attendent sur I/O
SELECT 
    t.processlist_id,
    t.processlist_user,
    t.processlist_db,
    esh.event_name,
    esh.sql_text,
    ROUND(esh.timer_wait / 1000000000000, 2) as wait_sec
FROM performance_schema.threads t
JOIN performance_schema.events_statements_history esh 
    ON t.thread_id = esh.thread_id
WHERE esh.event_name LIKE '%wait/io%'
ORDER BY wait_sec DESC
LIMIT 20;
```

💡 **Conseil** : Ne vous fiez pas à une seule métrique. Croisez toujours plusieurs sources (système + MariaDB + application) pour confirmer votre diagnostic.

### Phase 3 : OPTIMIZE - Application des changements

**Objectif** : Appliquer un changement ciblé et mesurable.

#### Règles d'or de l'optimisation

1. **Un changement à la fois** : Permet d'isoler l'impact
2. **Changement réversible** : Toujours pouvoir faire un rollback
3. **Test en non-prod d'abord** : Si possible
4. **Documentation immédiate** : Noter QUOI, POURQUOI, QUAND
5. **Mesure avant/après** : Comparer avec la baseline

#### Priorisation des optimisations

Utilisez la **matrice impact/effort** :

```
        Impact élevé
             │
    ┌────────┼────────┐
    │   2    │   1    │  1. Quick wins
    │  Faire │  Faire │     (faire en priorité)
    │ ensuite│ d'abord│
    ├────────┼────────┤  2. Projets majeurs
E   │   4    │   3    │     (planifier)
f   │ Éviter │ Peut-  │
f   │        │ être   │  3. Optimisations mineures
o   │        │        │     (si temps disponible)
r   └────────┼────────┘
t             │         4. Time sinks
    Effort faible        (éviter)
```

**Exemples de classification** :

- **Quick wins (1)** :
  - Ajouter un index manquant sur colonne WHERE fréquente
  - Augmenter buffer pool si <50% RAM utilisée
  - Corriger une requête N+1 évidente

- **Projets majeurs (2)** :
  - Refonte du schéma pour dénormalisation
  - Migration vers partitionnement
  - Mise en place de read replicas

- **Optimisations mineures (3)** :
  - Fine-tuning de `innodb_io_capacity`
  - Ajustement de `tmp_table_size`
  - Optimisation d'une requête rare

- **Time sinks (4)** :
  - Micro-optimisation de requêtes déjà rapides
  - Tweaking de paramètres obscurs sans impact mesurable
  - Over-engineering du schéma

#### Exemple de processus d'optimisation documenté

```sql
-- AVANT : Documenter l'état actuel
-- Problème identifié : Requête lente sur table orders (full table scan)

-- 1. Baseline
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345 AND order_date > '2024-01-01';
/*
+------+-------------+--------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table  | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+--------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | orders | ALL  | NULL          | NULL | NULL    | NULL | 125000 | Using where |
+------+-------------+--------+------+---------------+------+---------+------+--------+-------------+
*/

-- Temps d'exécution baseline
SELECT BENCHMARK(100, (
    SELECT COUNT(*) FROM orders 
    WHERE customer_id = 12345 AND order_date > '2024-01-01'
));
-- Temps : 2.34 secondes pour 100 exécutions

-- 2. Hypothèse d'optimisation
-- Créer un index composite sur (customer_id, order_date)
-- Impact attendu : Réduction du scan de 125000 à ~50 lignes

-- 3. Application du changement (avec timing)
SET @start_time = NOW(6);
CREATE INDEX idx_customer_date ON orders(customer_id, order_date);
SET @end_time = NOW(6);

SELECT TIMESTAMPDIFF(MICROSECOND, @start_time, @end_time) / 1000000 as creation_time_sec;
-- Temps de création : 3.2 secondes

-- 4. Validation
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345 AND order_date > '2024-01-01';
/*
+------+-------------+--------+-------+-------------------+-------------------+---------+------+------+-------------+
| id   | select_type | table  | type  | possible_keys     | key               | key_len | ref  | rows | Extra       |
+------+-------------+--------+-------+-------------------+-------------------+---------+------+------+-------------+
|    1 | SIMPLE      | orders | range | idx_customer_date | idx_customer_date | 8       | NULL |   48 | Using where |
+------+-------------+--------+-------+-------------------+-------------------+---------+------+------+-------------+
*/

-- Nouveau benchmark
SELECT BENCHMARK(100, (
    SELECT COUNT(*) FROM orders 
    WHERE customer_id = 12345 AND order_date > '2024-01-01'
));
-- Temps : 0.18 secondes pour 100 exécutions

-- 5. Calcul du gain
SELECT 
    2.34 as baseline_sec,
    0.18 as optimized_sec,
    ROUND((2.34 - 0.18) / 2.34 * 100, 2) as improvement_pct,
    '92.31% faster' as result;
```

#### Template de changement documenté

```markdown
# Changement d'optimisation

**Date** : 2024-12-14  
**Auteur** : DBA Team  
**Environnement** : Production DB1  

## Problème identifié
- Requête lente : `SELECT * FROM orders WHERE customer_id = X`
- Temps moyen : 2.34s pour 100 exécutions
- Type : Full table scan (125000 rows)
- Impact : 500 requêtes/min → 1170s/min de temps CPU gaspillé

## Diagnostic
- Analyse EXPLAIN : Pas d'index sur customer_id
- Buffer pool hit rate : 98.5% (pas le problème)
- Top query du slow log : 35% du temps total

## Solution proposée
Créer index composite : `idx_customer_date (customer_id, order_date)`

**Risques** :
- Temps de création : ~3-5s (table lockée)
- Espace disque : ~500MB supplémentaires
- Overhead sur INSERT/UPDATE : minime (index covering)

**Rollback** :
```sql
DROP INDEX idx_customer_date ON orders;
```

## Implémentation
```sql
-- Fenêtre de maintenance : 2024-12-14 02:00 UTC
CREATE INDEX idx_customer_date ON orders(customer_id, order_date);
```

## Résultats
- Temps exécution : 0.18s (92% amélioration)
- Espace utilisé : 485MB
- Buffer pool usage : +2% (acceptable)
- Slow queries count : -35%

## Validation
- ✅ Performance améliored
- ✅ Pas de régression sur autres requêtes
- ✅ Monitoring 24h : Stable
```

### Phase 4 : VALIDATE - Mesure de l'impact

**Objectif** : Confirmer que l'optimisation a eu l'effet escompté.

#### Checklist de validation

```sql
-- 1. Comparer les métriques clés avant/après
CREATE TEMPORARY TABLE validation_comparison AS
SELECT 
    'before' as period,
    metric_name,
    metric_value
FROM performance_baselines
WHERE collected_at = '2024-12-14 01:00:00'  -- Avant changement
UNION ALL
SELECT 
    'after',
    metric_name,
    metric_value
FROM performance_baselines
WHERE collected_at = '2024-12-14 03:00:00';  -- Après changement

-- Analyser les différences
SELECT 
    b.metric_name,
    b.metric_value as before_value,
    a.metric_value as after_value,
    ROUND(
        (CAST(a.metric_value AS DECIMAL(20,2)) - CAST(b.metric_value AS DECIMAL(20,2))) / 
        NULLIF(CAST(b.metric_value AS DECIMAL(20,2)), 0) * 100, 
        2
    ) as change_pct
FROM 
    (SELECT * FROM validation_comparison WHERE period = 'before') b
JOIN 
    (SELECT * FROM validation_comparison WHERE period = 'after') a
    ON b.metric_name = a.metric_name
WHERE 
    ABS(
        (CAST(a.metric_value AS DECIMAL(20,2)) - CAST(b.metric_value AS DECIMAL(20,2))) / 
        NULLIF(CAST(b.metric_value AS DECIMAL(20,2)), 0)
    ) > 0.05  -- Changements >5%
ORDER BY ABS(change_pct) DESC;
```

#### Validation continue (24-48h post-changement)

```sql
-- Créer une vue de monitoring post-optimisation
CREATE OR REPLACE VIEW v_optimization_health AS
SELECT 
    -- Général
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Queries') / 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Uptime'), 2
    ) as queries_per_sec,
    
    -- Slow queries
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Slow_queries') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Queries'), 0) * 100, 4
    ) as slow_queries_pct,
    
    -- Buffer pool
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100, 2
    ) as dirty_pages_pct,
    
    -- Threads
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_running') as threads_running,
     
    -- Locks
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_row_lock_current_waits') as current_lock_waits,
    
    NOW() as checked_at;

-- Surveiller pendant 24-48h
-- Créer une alerte si les métriques dévient de >10% de la baseline
SELECT * FROM v_optimization_health;
```

⚠️ **Attention** : Certains effets ne sont visibles qu'après plusieurs heures (cache warming, stats optimizer, etc.)

### Phase 5 : DOCUMENT - Traçabilité

**Objectif** : Maintenir un historique complet des changements.

#### Structure de documentation recommandée

```sql
-- Table de tracking des optimisations
CREATE TABLE optimization_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    optimization_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    category ENUM('configuration', 'index', 'query', 'schema', 'hardware'),
    description TEXT,
    change_sql TEXT,
    rollback_sql TEXT,
    baseline_before TEXT,  -- JSON ou texte des métriques
    baseline_after TEXT,
    impact_description TEXT,
    author VARCHAR(100),
    status ENUM('planned', 'testing', 'deployed', 'rolled_back'),
    INDEX idx_date (optimization_date),
    INDEX idx_category (category)
) ENGINE=InnoDB;

-- Exemple d'insertion
INSERT INTO optimization_log (
    category,
    description,
    change_sql,
    rollback_sql,
    baseline_before,
    baseline_after,
    impact_description,
    author,
    status
) VALUES (
    'index',
    'Ajout index composite orders(customer_id, order_date)',
    'CREATE INDEX idx_customer_date ON orders(customer_id, order_date);',
    'DROP INDEX idx_customer_date ON orders;',
    '{"avg_query_time": "2.34s", "slow_queries": "500/min"}',
    '{"avg_query_time": "0.18s", "slow_queries": "35/min"}',
    'Amélioration de 92% du temps de requête. Réduction de 93% des slow queries sur ce pattern.',
    'dba_team',
    'deployed'
);
```

### Phase 6 : MONITOR - Surveillance continue

**Objectif** : Détecter les régressions et nouveaux problèmes.

#### Dashboard de monitoring essentiel

```sql
-- Vue de synthèse pour monitoring quotidien
CREATE OR REPLACE VIEW v_daily_performance_summary AS
SELECT 
    DATE(collected_at) as date,
    metric_category,
    metric_name,
    AVG(CAST(metric_value AS DECIMAL(20,2))) as avg_value,
    MIN(CAST(metric_value AS DECIMAL(20,2))) as min_value,
    MAX(CAST(metric_value AS DECIMAL(20,2))) as max_value,
    STDDEV(CAST(metric_value AS DECIMAL(20,2))) as stddev_value
FROM performance_baselines
WHERE collected_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(collected_at), metric_category, metric_name;

-- Alertes automatiques sur anomalies
DELIMITER //
CREATE OR REPLACE PROCEDURE check_performance_anomalies()
BEGIN
    DECLARE v_queries_per_sec DECIMAL(10,2);
    DECLARE v_avg_queries_per_sec DECIMAL(10,2);
    DECLARE v_threshold_high DECIMAL(10,2);
    DECLARE v_threshold_low DECIMAL(10,2);
    
    -- Calculer queries/sec actuel
    SELECT 
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Queries') / 
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Uptime'), 2
        )
    INTO v_queries_per_sec;
    
    -- Calculer moyenne des 7 derniers jours
    SELECT AVG(avg_value) INTO v_avg_queries_per_sec
    FROM v_daily_performance_summary
    WHERE metric_name = 'queries_per_second'
    AND date > DATE_SUB(CURDATE(), INTERVAL 7 DAY);
    
    -- Seuils : ±30% de la moyenne
    SET v_threshold_high = v_avg_queries_per_sec * 1.3;
    SET v_threshold_low = v_avg_queries_per_sec * 0.7;
    
    -- Alerter si hors seuils
    IF v_queries_per_sec > v_threshold_high THEN
        SELECT CONCAT('ALERT: Queries/sec élevé: ', v_queries_per_sec, 
                     ' (moyenne: ', v_avg_queries_per_sec, ')') as alert;
    ELSEIF v_queries_per_sec < v_threshold_low THEN
        SELECT CONCAT('ALERT: Queries/sec faible: ', v_queries_per_sec,
                     ' (moyenne: ', v_avg_queries_per_sec, ')') as alert;
    ELSE
        SELECT 'OK: Performance normale' as status;
    END IF;
END //
DELIMITER ;

-- Exécuter périodiquement (via cron ou event scheduler)
CREATE EVENT IF NOT EXISTS check_perf_hourly
ON SCHEDULE EVERY 1 HOUR
DO CALL check_performance_anomalies();
```

---

## Méthodologies complémentaires

### Approche Top-Down vs Bottom-Up

#### Top-Down : De l'application vers la base

**Quand l'utiliser** : Quand le problème est applicatif (requêtes, architecture).

```
Application
    │
    ├─ Analyse des logs applicatifs
    │  • Temps de réponse endpoints
    │  • Requêtes SQL générées par ORM
    │
    ├─ Profiling applicatif
    │  • Query count par page
    │  • N+1 queries
    │
    ├─ Analyse des requêtes SQL
    │  • Slow query log
    │  • EXPLAIN plans
    │
    └─ Configuration MariaDB
       • Buffer pool
       • Indexes
```

**Exemple** :
1. Endpoint `/orders` prend 5 secondes
2. Profiling montre 50 requêtes SQL pour charger une page
3. Identification du pattern N+1
4. Solution : Eager loading ou refonte requête

#### Bottom-Up : Du système vers l'application

**Quand l'utiliser** : Quand le problème est systémique (ressources, configuration).

```
Système (CPU, RAM, I/O)
    │
    ├─ Monitoring système (top, iostat, vmstat)
    │  • CPU saturé ?
    │  • RAM insuffisante ?
    │  • I/O wait élevé ?
    │
    ├─ Configuration OS
    │  • Filesystem (XFS vs ext4)
    │  • Scheduler I/O
    │  • Kernel parameters
    │
    ├─ Configuration MariaDB
    │  • Buffer pool sizing
    │  • Thread pool
    │  • I/O capacity
    │
    └─ Requêtes et indexes
       • Optimisation ciblée
```

**Exemple** :
1. `iostat` montre 100% disk util
2. MariaDB metrics : buffer pool hit rate à 85%
3. Diagnostic : buffer pool trop petit
4. Solution : Augmenter buffer pool de 8GB à 32GB

### Méthode des 5 Pourquoi

Technique d'analyse root cause :

**Exemple réel** :

```
Problème : Le serveur est lent

Pourquoi ? → Les requêtes prennent 10s
  Pourquoi ? → Elles font des full table scans
    Pourquoi ? → Il n'y a pas d'index
      Pourquoi ? → L'index a été supprimé par erreur
        Pourquoi ? → Pas de process de validation des DDL
          
→ Solution root cause : Mettre en place un process de review 
  des changements de schéma avec rollback automatique
```

Cette méthode évite de traiter uniquement les symptômes.

---

## Outils de diagnostic essentiels

### 1. mysqltuner

Script Perl qui analyse la configuration et suggère des optimisations.

```bash
# Installation
wget https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl
chmod +x mysqltuner.pl

# Exécution
./mysqltuner.pl --user root --pass your_password

# Output exemple :
# [!!] Maximum possible memory usage: 48.2G (240% of installed RAM)
# [!!] Slow queries: 5% (12K/240K)
# [!!] Temporary tables created on disk: 35% (23K on disk / 65K total)
# [OK] InnoDB buffer pool / data size: 16.0G/12.3G
# [!!] Query cache efficiency: 12.5% (200K cached / 1.6M selects)
```

💡 **Conseil** : mysqltuner est un excellent point de départ, mais ses recommandations doivent être validées dans votre contexte.

### 2. pt-query-digest (Percona Toolkit)

Analyse détaillée du slow query log (voir section 15.7).

```bash
# Analyse du slow log
pt-query-digest /var/log/mysql/slow.log > slow_report.txt

# Top 10 des requêtes par temps total
pt-query-digest --limit 10 --order-by Query_time:sum /var/log/mysql/slow.log
```

### 3. sys schema

Collection de vues pour diagnostic rapide (voir section 15.8).

```sql
-- Top 10 des requêtes les plus lentes
SELECT * FROM sys.statement_analysis 
ORDER BY avg_latency DESC 
LIMIT 10;

-- Index non utilisés
SELECT * FROM sys.schema_unused_indexes;

-- Tables les plus lues
SELECT * FROM sys.schema_table_statistics 
ORDER BY total_latency DESC 
LIMIT 10;
```

### 4. Performance Schema

Instrumentation native de MariaDB (voir section 15.8).

```sql
-- Top statements par temps d'exécution
SELECT 
    DIGEST_TEXT,
    COUNT_STAR as count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) as avg_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) as total_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY total_sec DESC
LIMIT 20;
```

---

## Pièges courants à éviter

### 1. Optimisation prématurée

❌ **Erreur** : Optimiser avant de mesurer
```sql
-- Mauvais : Créer des index "au cas où"
CREATE INDEX idx_maybe_useful ON users(last_login);
```

✅ **Correct** : Mesurer d'abord
```sql
-- Analyser le slow log avant de créer l'index
-- Vérifier que last_login est utilisé dans des WHERE fréquents
SELECT COUNT(*) FROM mysql.slow_log 
WHERE sql_text LIKE '%last_login%';
```

### 2. Changer trop de paramètres simultanément

❌ **Erreur** :
```ini
# Changement multiple sans baseline
innodb_buffer_pool_size = 32G  # Était 8G
innodb_io_capacity = 10000     # Était 200
innodb_flush_log_at_trx_commit = 2  # Était 1
max_connections = 500          # Était 151
```

✅ **Correct** : Un par un, avec mesure
```ini
# Jour 1 : Buffer pool uniquement
innodb_buffer_pool_size = 32G
# Mesurer pendant 24h

# Jour 3 : Si stable, ajuster I/O
innodb_io_capacity = 10000
# Mesurer pendant 24h
```

### 3. Ignorer les statistiques de l'optimizer

❌ **Erreur** : Forcer un plan d'exécution
```sql
SELECT * FROM orders FORCE INDEX (idx_date) WHERE customer_id = 123;
```

✅ **Correct** : Mettre à jour les statistiques
```sql
ANALYZE TABLE orders;
-- Laisser l'optimizer choisir
SELECT * FROM orders WHERE customer_id = 123;
```

### 4. Sur-indexation

❌ **Erreur** : Index sur toutes les colonnes
```sql
-- Table avec 15 colonnes, 12 index !
CREATE INDEX idx_col1 ON mytable(col1);
CREATE INDEX idx_col2 ON mytable(col2);
-- ...
CREATE INDEX idx_col12 ON mytable(col12);
```

✅ **Correct** : Index stratégiques basés sur l'usage réel
```sql
-- Analyser les requêtes réelles d'abord
-- Créer uniquement les index utilisés
CREATE INDEX idx_frequent_where ON mytable(col1, col2);  -- Composite
CREATE INDEX idx_foreign_key ON mytable(col5);  -- FK uniquement
```

### 5. Négliger le monitoring post-changement

❌ **Erreur** : Déployer et oublier
```sql
-- Changement déployé vendredi soir
ALTER TABLE large_table ADD INDEX idx_new (column);
-- Pas de surveillance pendant le weekend
```

✅ **Correct** : Monitoring actif 24-48h
```sql
-- Mise en place alertes
-- Surveillance metrics pendant 48h
-- Rollback automatique si régression détectée
```

---

## ✅ Points clés à retenir

- 🎯 **Mesurer avant d'optimiser** : Établir toujours une baseline objective
- 🔍 **Identifier le vrai bottleneck** : CPU, RAM, I/O, réseau ou application
- 📊 **Prioriser selon impact/effort** : Focus sur les "quick wins" d'abord
- 🔄 **Cycle itératif** : Observe → Analyze → Optimize → Validate → Document → Monitor
- 📝 **Documentation rigoureuse** : Tracer tous les changements et leurs impacts
- ⚠️ **Un changement à la fois** : Permet d'isoler les effets
- ✅ **Validation continue** : Surveiller pendant 24-48h minimum post-changement
- 🚫 **Éviter l'optimisation prématurée** : Ne pas optimiser sans preuve de problème
- 🆕 **11.8 facilite le diagnostic** : Nouveaux outils Performance Schema et sys
- 🔧 **Utiliser les bons outils** : mysqltuner, pt-query-digest, Performance Schema

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Optimization and Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)
- [📖 Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [📖 EXPLAIN](https://mariadb.com/kb/en/explain/)

### Outils

- [MySQLTuner](https://github.com/major/MySQLTuner-perl)
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)
- [sys Schema](https://github.com/mysql/mysql-sys)

### Lectures recommandées

- [High Performance MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/) - Baron Schwartz et al.
- [Percona Blog - Performance category](https://www.percona.com/blog/category/mysql/)

---

## ➡️ Section suivante

**[15.2 Configuration mémoire](/15-performance-tuning/02-configuration-memoire.md)** : Maintenant que vous maîtrisez la méthodologie, apprenons à dimensionner correctement le buffer pool InnoDB et les autres paramètres mémoire pour des performances optimales.

---

*La méthodologie est la fondation de toute optimisation réussie. Prenez le temps de bien l'assimiler avant de plonger dans les optimisations techniques spécifiques.*

⏭️ [Configuration mémoire](/15-performance-tuning/02-configuration-memoire.md)

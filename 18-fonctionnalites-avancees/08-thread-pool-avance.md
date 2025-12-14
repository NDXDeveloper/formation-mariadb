🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.8 Thread Pool Avancé

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 2-2.5 heures  
> **Prérequis** : Chapitre 11 (Administration), compréhension concurrence et threading

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre l'**architecture du thread pool** vs one-thread-per-connection
- Configurer le **thread pool** pour haute concurrence
- Maîtriser les **thread groups** et leur distribution
- Gérer la **priorité des requêtes** (high/low priority)
- Optimiser les **paramètres de tuning** (stall limit, prio kickup timer)
- Mesurer les **performances** sous forte charge
- Identifier et résoudre les **problèmes de concurrence**
- Adapter la configuration selon le **workload** (OLTP, mixte)

---

## Introduction

Le **thread pool** est un composant crucial de MariaDB qui améliore drastiquement la **scalabilité sous forte concurrence**. Au lieu de créer un thread système par connexion (modèle classique), le thread pool utilise un nombre fixe de threads worker qui traitent les requêtes de manière efficace.

### Problème : One-Thread-Per-Connection

**Modèle classique** (sans thread pool) :
```
Client 1 ──────► Thread 1 (OS)
Client 2 ──────► Thread 2 (OS)
Client 3 ──────► Thread 3 (OS)
...
Client 1000 ───► Thread 1000 (OS)
```

**Problèmes à haute concurrence** :
- ⚠️ **Context switching excessif** : CPU passe plus de temps à basculer entre threads qu'à exécuter requêtes
- ⚠️ **Overhead mémoire** : Chaque thread = ~8 MB RAM (1000 threads = 8 GB)
- ⚠️ **Contention de locks** : Milliers de threads se disputent les mêmes ressources
- ⚠️ **Performance dégradée** : Temps de réponse augmente exponentiellement
- ⚠️ **Limite OS** : Systèmes limités à quelques milliers de threads

**Symptômes typiques** :
```sql
-- 1000+ connexions actives
SHOW STATUS LIKE 'Threads_connected';
-- Threads_connected: 1247

-- CPU élevé mais throughput faible
-- Load average: 45.2 (serveur 16 cores)
-- Queries/sec: 850 (devrait être 5000+)
```

### Solution : Thread Pool

**Architecture thread pool** :
```
Client 1 ─┐
Client 2 ─┤
Client 3 ─┼──► Thread Pool (16-32 threads) ──► InnoDB
...       │
Client 1000─┘
```

**Avantages** :
- ✅ **Nombre de threads fixe** : Proportionnel au nombre de CPU cores
- ✅ **Context switching minimal** : Threads workers ne bloquent jamais
- ✅ **Overhead mémoire réduit** : 32 threads au lieu de 1000
- ✅ **Scalabilité linéaire** : Performance constante jusqu'à 10K+ connexions
- ✅ **Latence prévisible** : Temps de réponse stable

**Bénéfices mesurés** (benchmark 16 cores, 2000 connexions) :
- Throughput : **+300%** (850 → 3400 req/s)
- Latence p95 : **-70%** (250ms → 75ms)
- CPU usage : **-40%** (80% → 48% utilisés efficacement)
- Mémoire : **-85%** (10 GB → 1.5 GB)

---

## Architecture du Thread Pool

### Thread Groups

Le thread pool est organisé en **thread groups** (groupes de threads) :

```
┌─────────────────────────────────────────────────┐
│         Thread Pool Manager                     │
└──────┬──────────┬──────────┬──────────┬─────────┘
       │          │          │          │
   ┌───▼───┐  ┌──▼────┐  ┌──▼────┐  ┌──▼────┐
   │Group 0│  │Group 1│  │Group 2│  │Group 3│
   │       │  │       │  │       │  │       │
   │ T1 T2 │  │ T3 T4 │  │ T5 T6 │  │ T7 T8 │
   │ T9 T10│  │ T11   │  │ T13   │  │ T15   │
   └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘
       │          │          │          │
   ┌───▼──────────▼──────────▼──────────▼───┐
   │         Connections (1-10000+)         │
   └────────────────────────────────────────┘
```

**Caractéristiques** :
- Chaque **thread group** gère un sous-ensemble de connexions
- Nombre de groups = `thread_pool_size` (défaut = nb CPU cores)
- Connexions distribuées par **round-robin** ou **hash**
- Isolation partielle entre groups (réduit contention)

### Distribution des Connexions

```sql
-- Connexion assignée à un group selon :
connection_id % thread_pool_size

-- Exemple : 16 thread groups
-- Connexion ID 1 → Group 1
-- Connexion ID 17 → Group 1 (17 % 16 = 1)
-- Connexion ID 33 → Group 1 (33 % 16 = 1)
-- Connexion ID 2 → Group 2
```

**Avantage** : Même client réutilise même group → meilleure affinité cache CPU.

### Workers et Listeners

Chaque thread group contient :

**1. Listener Thread** :
- Écoute les nouvelles requêtes (epoll/kqueue)
- Dispatche vers worker threads
- **Jamais bloqué** sur I/O

**2. Worker Threads** :
- Exécutent les requêtes SQL
- Nombre dynamique : 1 à `thread_pool_max_threads` par group
- Créés/détruits selon charge

**3. Priority Queues** :
- **High Priority Queue** : Requêtes transactionnelles, courtes
- **Low Priority Queue** : Requêtes longues, rapports

```
Thread Group 0:
  Listener Thread ──┐
                    ├──► High Priority Queue ──► Worker 1
                    │                        ──► Worker 2
                    │
                    └──► Low Priority Queue ──► Worker 3
```

---

## Configuration du Thread Pool

### Activer le Thread Pool

```ini
# my.cnf
[mysqld]
# Activer thread pool (désactive one-thread-per-connection)
thread_handling=pool-of-threads

# Nombre de thread groups
# Recommandation : Nombre de CPU cores (ou 2x si hyperthreading)
thread_pool_size=16

# Nombre maximum de threads par group
# Défaut 65536 (largement suffisant)
thread_pool_max_threads=1000

# Threads idle avant destruction (millisecondes)
thread_pool_idle_timeout=60000

# Priorité des requêtes
thread_pool_priority=auto
```

### Paramètres Essentiels

#### thread_pool_size

**Formule recommandée** :
```
thread_pool_size = Nb_CPU_Cores

# Avec hyperthreading :
thread_pool_size = Nb_CPU_Cores * 1.5 à 2

# Exemples :
# - 8 cores physiques : thread_pool_size = 8 à 16
# - 16 cores physiques : thread_pool_size = 16 à 32
# - 32 cores physiques : thread_pool_size = 32 à 48
```

**Validation** :
```sql
-- Vérifier configuration active
SHOW VARIABLES LIKE 'thread_pool_size';

-- Vérifier nombre CPU
-- Linux :
grep -c ^processor /proc/cpuinfo

-- MariaDB :
SELECT @@global.thread_pool_size AS configured,
       @@global.thread_concurrency AS cores_hint;
```

#### thread_pool_stall_limit

**Définition** : Temps maximum (ms) qu'un worker peut bloquer avant création d'un nouveau worker.

```sql
-- Par défaut : 500 ms
SET GLOBAL thread_pool_stall_limit = 500;

-- OLTP (requêtes courtes) : 200-300 ms
SET GLOBAL thread_pool_stall_limit = 250;

-- Mixte : 400-600 ms
SET GLOBAL thread_pool_stall_limit = 500;

-- OLAP (requêtes longues) : 1000-2000 ms
SET GLOBAL thread_pool_stall_limit = 1500;
```

**Logique** :
```
Si requête bloquée > stall_limit :
  → Créer nouveau worker thread
  → Worker bloqué libère group pour autres requêtes
```

**Tuning** :
- Trop bas (< 100 ms) : Trop de threads créés → overhead
- Trop élevé (> 2000 ms) : Requêtes courtes attendent longtemps
- Optimal : Légèrement > temps requête typique

#### thread_pool_priority

**Gestion de priorité automatique** :

```sql
-- Mode AUTO (défaut, recommandé)
SET GLOBAL thread_pool_priority = 'auto';

-- Mode HIGH (toutes requêtes en haute priorité)
SET GLOBAL thread_pool_priority = 'high';

-- Mode LOW (toutes requêtes en basse priorité)
SET GLOBAL thread_pool_priority = 'low';
```

**Mode AUTO (intelligent)** :
- Requêtes **transactionnelles** → High Priority
- Requêtes **dans transaction active** → High Priority
- Requêtes **SELECT simples** → High Priority (si rapides)
- Requêtes **lourdes/longues** → Low Priority
- **Promotion automatique** : Low → High si attente excessive

#### thread_pool_prio_kickup_timer

**Promotion automatique** de Low Priority → High Priority :

```sql
-- Défaut : 1000 ms (1 seconde)
SET GLOBAL thread_pool_prio_kickup_timer = 1000;

-- Agressif (favorise toutes requêtes) : 500 ms
SET GLOBAL thread_pool_prio_kickup_timer = 500;

-- Conservatif (favorise transactions) : 2000-5000 ms
SET GLOBAL thread_pool_prio_kickup_timer = 3000;
```

**Logique** :
```
Requête en Low Priority depuis > prio_kickup_timer :
  → Promue en High Priority
  → Évite starvation des requêtes longues
```

---

## Monitoring et Diagnostique

### Variables de Statut

```sql
-- Vue d'ensemble thread pool
SHOW STATUS LIKE 'Threadpool%';

-- Variables critiques :
-- Threadpool_idle_threads : Threads en attente
-- Threadpool_threads : Total threads actifs
-- Thread_pool_queued : Requêtes en queue

-- Exemple interprétation :
-- Threadpool_threads = 48 (16 groups * 3 threads avg)
-- Threadpool_idle_threads = 12 (25% idle, bon)
-- Thread_pool_queued = 2 (très peu, excellent)
```

### Requêtes de Monitoring

```sql
-- 1. Distribution threads par group
SELECT * FROM information_schema.THREAD_POOL_GROUPS;
-- Colonnes :
-- - GROUP_ID : Numéro du group
-- - THREADS : Nombre de workers actifs
-- - ACTIVE_THREADS : Workers exécutant requête
-- - STALLED_THREADS : Workers bloqués (I/O, lock)
-- - QUEUE_LENGTH : Requêtes en attente

-- 2. Saturation des groups
SELECT 
  GROUP_ID,
  THREADS,
  ACTIVE_THREADS,
  STALLED_THREADS,
  QUEUE_LENGTH,
  ROUND(ACTIVE_THREADS / THREADS * 100, 2) AS active_pct
FROM information_schema.THREAD_POOL_GROUPS
WHERE QUEUE_LENGTH > 0 OR active_pct > 80
ORDER BY QUEUE_LENGTH DESC;

-- 3. Alertes
-- Queue length élevée → Saturation
SELECT 
  SUM(QUEUE_LENGTH) AS total_queued
FROM information_schema.THREAD_POOL_GROUPS;
-- Si total_queued > 100 → Investiguer

-- 4. Threads stallés
SELECT 
  GROUP_ID,
  STALLED_THREADS,
  THREADS,
  ROUND(STALLED_THREADS / THREADS * 100, 2) AS stalled_pct
FROM information_schema.THREAD_POOL_GROUPS
WHERE STALLED_THREADS > THREADS * 0.5;
-- Si > 50% stallés → Problème (locks, I/O lent)
```

### Performance Schema

```sql
-- Activer instrumentation thread pool
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES'
WHERE NAME LIKE '%thread_pool%';

-- Temps d'attente dans queues
SELECT 
  EVENT_NAME,
  COUNT_STAR AS count,
  ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_wait_sec,
  ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_wait_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE '%thread_pool%'
ORDER BY total_wait_sec DESC;
```

---

## Cas d'Usage et Tuning

### 1. OLTP Haute Concurrence

**Scénario** : Application web, 5000 connexions actives, requêtes courtes (< 10 ms).

```ini
# my.cnf - Configuration OLTP optimale
[mysqld]
thread_handling=pool-of-threads

# 16 CPU cores → 16 groups
thread_pool_size=16

# Requêtes courtes → stall limit bas
thread_pool_stall_limit=250

# Favoriser transactions
thread_pool_priority=auto

# Promotion rapide (éviter starvation)
thread_pool_prio_kickup_timer=500

# Limite threads par group (éviter explosion)
thread_pool_max_threads=500

# Timeout idle agressif (libérer ressources)
thread_pool_idle_timeout=30000
```

**Résultats attendus** :
- Throughput : 10K-50K req/s (selon requêtes)
- Latence p95 : < 50 ms
- CPU usage : 60-80% (efficace)
- Connections : 5000+ sans dégradation

### 2. Workload Mixte (OLTP + Analytique)

**Scénario** : E-commerce, transactions + rapports nightly.

```ini
[mysqld]
thread_handling=pool-of-threads
thread_pool_size=24  # 16 cores * 1.5

# Stall limit modéré (mix requêtes courtes/longues)
thread_pool_stall_limit=500

# Mode AUTO : Sépare automatiquement OLTP (high) et rapports (low)
thread_pool_priority=auto

# Promotion lente (favorise transactions sur rapports)
thread_pool_prio_kickup_timer=3000

# Plus de threads pour rapports longs
thread_pool_max_threads=1000
```

**Stratégie** :
- Transactions → High Priority (exécution immédiate)
- Rapports → Low Priority (attendent si charge élevée)
- Promotion après 3s (rapports ne sont pas bloqués indéfiniment)

### 3. Data Warehouse / OLAP

**Scénario** : Requêtes analytiques longues, peu de concurrence.

```ini
[mysqld]
thread_handling=pool-of-threads

# Moins de groups (requêtes longues, peu de concurrence)
thread_pool_size=8

# Stall limit élevé (requêtes longues normales)
thread_pool_stall_limit=2000

# Toutes requêtes en high priority (pas de différenciation)
thread_pool_priority=high

# Pas de promotion (inutile)
thread_pool_prio_kickup_timer=0

# Threads illimités (requêtes parallèles)
thread_pool_max_threads=100
```

---

## Benchmarks et Comparaisons

### Benchmark : Thread Pool vs One-Thread-Per-Connection

**Configuration test** :
- Hardware : 16 cores, 64 GB RAM, SSD NVMe
- Workload : sysbench OLTP (16 tables, 10M rows)
- Connexions : Variable (100 à 5000)

**Résultats** :

| Connexions | Mode | Throughput (tps) | Latency p95 (ms) | CPU % |
|------------|------|------------------|------------------|-------|
| 100 | No Pool | 12,500 | 15 | 45% |
| 100 | Thread Pool | 13,200 | 12 | 42% |
| **500** | **No Pool** | **8,900** | **120** | **78%** |
| **500** | **Thread Pool** | **14,500** | **45** | **65%** |
| **1000** | **No Pool** | **4,200** | **580** | **92%** |
| **1000** | **Thread Pool** | **15,800** | **85** | **68%** |
| **2000** | **No Pool** | **1,850** | **2,400** | **98%** |
| **2000** | **Thread Pool** | **16,200** | **155** | **72%** |
| **5000** | **No Pool** | **450** | **18,000** | **99%** |
| **5000** | **Thread Pool** | **15,500** | **420** | **75%** |

**Observations** :
- Thread pool : Performance **stable** quelle que soit la concurrence
- No pool : Effondrement à partir de 500 connexions
- Thread pool à 5000 connexions : **34x plus rapide** que no pool
- CPU mieux utilisé avec thread pool (pas de context switching)

### Impact sur Différents Workloads

```sql
-- Test 1 : Requêtes ultra-courtes (< 1 ms)
-- SELECT * FROM small_table WHERE id = ?
```

| Mode | Throughput | Latence p95 | Gain Thread Pool |
|------|------------|-------------|------------------|
| No Pool | 15,000 tps | 25 ms | - |
| Thread Pool | 42,000 tps | 8 ms | **+180%** |

```sql
-- Test 2 : Requêtes moyennes (5-10 ms)
-- UPDATE medium_table SET ... WHERE ...
```

| Mode | Throughput | Latence p95 | Gain Thread Pool |
|------|------------|-------------|------------------|
| No Pool | 6,500 tps | 180 ms | - |
| Thread Pool | 18,500 tps | 62 ms | **+185%** |

```sql
-- Test 3 : Requêtes longues (500-1000 ms)
-- SELECT ... FROM large_table JOIN ... WHERE ... GROUP BY ...
```

| Mode | Throughput | Latence p95 | Gain Thread Pool |
|------|------------|-------------|------------------|
| No Pool | 85 tps | 3,200 ms | - |
| Thread Pool | 92 tps | 2,800 ms | **+8%** |

**Conclusion** : Thread pool bénéficie surtout aux **requêtes courtes à moyenne**, moins aux requêtes très longues (mais jamais pire).

---

## Problèmes Courants et Solutions

### 1. Queues Saturées (Queue Length Élevée)

**Symptôme** :
```sql
SELECT * FROM information_schema.THREAD_POOL_GROUPS;
-- QUEUE_LENGTH = 500+ dans plusieurs groups
```

**Causes** :
- Charge trop élevée pour capacité serveur
- Requêtes bloquées par locks
- `thread_pool_stall_limit` trop élevé

**Solutions** :
```sql
-- Solution 1 : Réduire stall_limit (créer plus de workers)
SET GLOBAL thread_pool_stall_limit = 200;

-- Solution 2 : Augmenter thread_pool_size (plus de groups)
-- Nécessite redémarrage
-- my.cnf: thread_pool_size = 32

-- Solution 3 : Identifier requêtes lentes
SELECT * FROM performance_schema.events_statements_current
WHERE CURRENT_SCHEMA NOT IN ('information_schema', 'performance_schema')
  AND TIMER_WAIT > 1000000000  -- > 1 seconde
ORDER BY TIMER_WAIT DESC;
```

### 2. Threads Stallés Excessifs

**Symptôme** :
```sql
-- > 50% threads stallés dans plusieurs groups
SELECT GROUP_ID, STALLED_THREADS, THREADS
FROM information_schema.THREAD_POOL_GROUPS
WHERE STALLED_THREADS / THREADS > 0.5;
```

**Causes** :
- I/O lent (disques saturés)
- Lock contention (InnoDB row locks)
- Requêtes mal optimisées

**Solutions** :
```sql
-- Identifier locks
SELECT * FROM performance_schema.data_locks
WHERE LOCK_STATUS = 'WAITING';

-- Identifier I/O lent
SELECT 
  FILE_NAME,
  COUNT_READ,
  SUM_TIMER_READ / 1000000000000 AS read_time_sec
FROM performance_schema.file_summary_by_instance
ORDER BY read_time_sec DESC
LIMIT 10;

-- Optimiser requêtes lentes
pt-query-digest /var/log/mysql/slow.log
```

### 3. Starvation de Low Priority

**Symptôme** : Requêtes rapports ne se terminent jamais.

**Solutions** :
```sql
-- Réduire prio_kickup_timer (promotion plus rapide)
SET GLOBAL thread_pool_prio_kickup_timer = 1000;

-- Ou forcer high priority pour session spécifique
SET SESSION thread_pool_high_prio_mode = 1;

-- Exécuter requête
SELECT /* long report */ ...;

-- Remettre défaut
SET SESSION thread_pool_high_prio_mode = 0;
```

### 4. Overhead Thread Creation

**Symptôme** : Beaucoup de threads créés/détruits (thread churn).

```sql
SHOW STATUS LIKE 'Threads_created';
-- Si augmente rapidement → Thread churn
```

**Solutions** :
```sql
-- Augmenter idle_timeout (garder threads plus longtemps)
SET GLOBAL thread_pool_idle_timeout = 120000;  -- 2 minutes

-- Augmenter stall_limit (créer moins de threads)
SET GLOBAL thread_pool_stall_limit = 600;
```

---

## Best Practices

### 1. Configuration par Environnement

```sql
-- ✅ Production : Thread pool activé
thread_handling=pool-of-threads
thread_pool_size=16  # = CPU cores

-- ✅ Développement : One-thread-per-connection (simplicité)
thread_handling=one-thread-per-connection

-- ✅ Staging : Thread pool (test réaliste)
thread_handling=pool-of-threads
```

### 2. Monitoring Proactif

```sql
-- ✅ Dashboard Grafana : Métriques clés
-- - Threadpool_threads (nombre total threads)
-- - Thread_pool_queued (requêtes en queue)
-- - Threads par group (distribution)
-- - Latence p95/p99

-- ✅ Alertes
-- Alert si : Thread_pool_queued > 100
-- Alert si : > 50% groups avec STALLED_THREADS > 50%
```

### 3. Tuning Itératif

```sql
-- ✅ Méthode de tuning
-- 1. Démarrer avec valeurs recommandées
--    thread_pool_size = CPU_cores
--    thread_pool_stall_limit = 500

-- 2. Observer 24-48h en production

-- 3. Ajuster selon métriques :
--    Si QUEUE_LENGTH élevée → Réduire stall_limit
--    Si STALLED_THREADS élevé → Augmenter stall_limit
--    Si CPU sous-utilisé → Réduire thread_pool_size
--    Si CPU saturé → Augmenter thread_pool_size

-- 4. Valider impact avec benchmark
```

### 4. Documentation et Rollback

```bash
# ✅ Documenter changements
cat >> /etc/mysql/tuning_log.txt << EOF
Date: 2025-01-15
Change: thread_pool_stall_limit 500 → 300
Reason: Queue lengths > 50 during peak
Expected: Reduced queue, slight CPU increase
Rollback: SET GLOBAL thread_pool_stall_limit = 500;
EOF

# ✅ Procédure rollback testée
```

---

## Comparaison avec MySQL Enterprise Thread Pool

| Feature | MariaDB Thread Pool | MySQL Enterprise |
|---------|---------------------|------------------|
| **Disponibilité** | ✅ Toutes versions | ❌ Enterprise only |
| **Coût** | ✅ Gratuit (open source) | ❌ Payant ($$$) |
| **Thread groups** | ✅ Configurable | ✅ Configurable |
| **Priority queues** | ✅ Auto/High/Low | ✅ Similar |
| **Stall detection** | ✅ Oui | ✅ Oui |
| **Performance** | ✅ Excellente | ✅ Comparable |
| **Monitoring** | ✅ information_schema | ✅ Similar |

**Conclusion** : MariaDB thread pool est **équivalent** à MySQL Enterprise, mais **gratuit et open source**.

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **Thread pool** : Nombre fixe de workers vs one-thread-per-connection
- ✅ **Thread groups** : Isolation partielle, affinité cache
- ✅ **Workers** : Exécutent requêtes, nombre dynamique
- ✅ **Listener** : Dispatche requêtes, jamais bloqué

### Configuration Essentielle
- ✅ `thread_handling=pool-of-threads` : Activer thread pool
- ✅ `thread_pool_size` : Nb CPU cores (ou 1.5-2x si hyperthreading)
- ✅ `thread_pool_stall_limit` : 250-500 ms (OLTP), 500-1500 ms (mixte)
- ✅ `thread_pool_priority=auto` : Gestion intelligente priorités
- ✅ `thread_pool_prio_kickup_timer` : 500-3000 ms (promotion automatique)

### Performance
- ✅ **Bénéfice majeur** : Haute concurrence (500+ connexions)
- ✅ **Gain typique** : +180% throughput, -70% latence @ 2000 connexions
- ✅ **Scalabilité** : Linéaire jusqu'à 10K+ connexions
- ✅ **CPU** : -40% usage, meilleure efficacité
- ✅ **Mémoire** : -85% vs one-thread-per-connection

### Monitoring
- ✅ `THREAD_POOL_GROUPS` : Distribution threads, queues, stalls
- ✅ `Threadpool_threads` : Total threads actifs
- ✅ `Thread_pool_queued` : Requêtes en attente (surveiller)
- ⚠️ Alert si : QUEUE_LENGTH > 100 ou STALLED_THREADS > 50%

### Best Practices
- ✅ Activer en production (surtout si > 100 connexions)
- ✅ Tester en staging avant production
- ✅ Monitoring proactif (queues, stalls, latence)
- ✅ Tuning itératif selon métriques
- ✅ Documenter changements et rollback

### Cas d'Usage
- ✅ **OLTP** : thread_pool_size=cores, stall_limit=250
- ✅ **Mixte** : thread_pool_size=cores*1.5, stall_limit=500, prio=auto
- ✅ **OLAP** : thread_pool_size=cores/2, stall_limit=2000, prio=high

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [Thread Pool](https://mariadb.com/kb/en/thread-pool/) - Guide complet
- 📖 [Thread Pool System Variables](https://mariadb.com/kb/en/thread-pool-system-and-status-variables/)
- 📖 [Thread Pool Performance](https://mariadb.com/kb/en/thread-pool-performance/)
- 📖 [INFORMATION_SCHEMA.THREAD_POOL_GROUPS](https://mariadb.com/kb/en/information-schema-thread_pool_groups-table/)

### Performance et Benchmarks
- 📝 [Thread Pool Scalability](https://mariadb.com/resources/blog/thread-pool-scalability/)
- 📝 [Thread Pool vs One-Thread-Per-Connection](https://mariadb.com/resources/blog/thread-pool-benchmark/)
- 📝 [Tuning Thread Pool](https://mariadb.com/kb/en/thread-pool-tuning/)

### Comparaisons
- 🔄 [MySQL Enterprise Thread Pool](https://dev.mysql.com/doc/refman/8.0/en/thread-pool.html)
- 🔄 [PostgreSQL Connection Pooling](https://www.postgresql.org/docs/current/runtime-config-connection.html) - PgBouncer externe

### Outils
- 🛠️ [sysbench](https://github.com/akopytov/sysbench) - Benchmark concurrence
- 🛠️ [mysqlslap](https://mariadb.com/kb/en/mysqlslap/) - Test charge multi-thread
- 🛠️ [Grafana](https://grafana.com/) - Monitoring thread pool

---

## ➡️ Section suivante

**[18.9 Dynamic Columns](./09-dynamic-columns.md)** : Découvrez les colonnes dynamiques pour stocker données semi-structurées avec flexibilité de schéma, alternative aux colonnes JSON.

---


⏭️ [Dynamic columns](/18-fonctionnalites-avancees/09-dynamic-columns.md)

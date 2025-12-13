🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.10 Thread Pool et gestion de la concurrence

> **Niveau** : Avancé  
> **Durée estimée** : 2-2.5 heures  
> **Prérequis** :
> - Section 11.9 (Monitoring et métriques)
> - Sections 11.1-11.2 (Configuration, variables système)
> - Compréhension des systèmes multi-threadés
> - Connaissances en concurrence et parallélisme

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture et le fonctionnement du Thread Pool
- **Distinguer** Thread Pool vs One-Thread-Per-Connection
- **Configurer** le Thread Pool pour votre charge de travail
- **Optimiser** les performances avec le Thread Pool
- **Surveiller** l'utilisation et la santé du Thread Pool
- **Diagnostiquer** les problèmes de concurrence
- **Adapter** la configuration selon le type de charge (OLTP, OLAP, mixte)
- **Exploiter** le Thread Pool pour la haute concurrence

---

## Introduction

Le **Thread Pool** est une fonctionnalité avancée de MariaDB qui révolutionne la gestion de la **concurrence** en remplaçant le modèle classique "un thread par connexion" par un **pool de threads réutilisables**.

### Problématique de la concurrence

En production, MariaDB doit gérer **simultanément** :
- Des centaines ou milliers de connexions
- Des requêtes de durées variables (ms à minutes)
- Des pics de charge imprévisibles
- Des ressources limitées (CPU, RAM, contexte switching)

### Le problème du modèle one-thread-per-connection

```
Modèle classique (problématique):
    1 connexion = 1 thread dédié

    10,000 connexions = 10,000 threads
        ↓
    Overhead énorme:
        - Mémoire: ~1 MB par thread = 10 GB
        - Context switching: CPU gaspillé
        - Scheduler OS saturé
        - Performance dégradée
```

**Conséquences** :
- 💥 **Scalabilité limitée** : Max ~1000-2000 connexions
- 🐢 **Performance dégradée** : Context switching excessif
- 💾 **Consommation RAM** : 1-2 MB par thread
- 🔥 **CPU gaspillé** : Threads inactifs consomment des ressources

### Solution : Thread Pool

```
Modèle Thread Pool (optimal):
    Pool de N groupes de threads (N = CPU cores)
    Connexions assignées aux groupes
    Threads réutilisés entre requêtes

    10,000 connexions → 16 groupes → ~48-64 threads actifs
        ↓
    Avantages:
        ✅ Mémoire: Fixe (~100-200 MB)
        ✅ Context switching: Minimal
        ✅ Scheduler: Sous contrôle
        ✅ Performance: Optimale
```

💡 **Principe clé** : Séparer le nombre de **connexions** du nombre de **threads**, permettant une scalabilité massive.

---

## Architecture du Thread Pool

### Concept de groupes (thread groups)

Le Thread Pool organise les threads en **groupes** :

```
┌─────────────────────────────────────────────────────────┐
│                    THREAD POOL                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Groupe 0    │  │  Groupe 1    │  │  Groupe N    │   │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤   │
│  │ Thread 1     │  │ Thread 1     │  │ Thread 1     │   │
│  │ Thread 2     │  │ Thread 2     │  │ Thread 2     │   │
│  │ Thread 3     │  │ Thread 3     │  │ Thread 3     │   │
│  │ ...          │  │ ...          │  │ ...          │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│         ↑                 ↑                 ↑           │
│         │                 │                 │           │
│  ┌──────┴────┐      ┌─────┴─────┐     ┌─────┴─────┐     │
│  │Connexions │      │Connexions │     │Connexions │     │
│  │ 1-100     │      │ 101-200   │     │ N-10000   │     │
│  └───────────┘      └───────────┘     └───────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Caractéristiques** :
- **1 groupe** = 1 queue de requêtes + N threads workers
- **Nombre de groupes** = `thread_pool_size` (recommandé : nombre de CPU cores)
- **Connexions** assignées aux groupes par round-robin
- **Threads** traîtent les requêtes de leur groupe

### Workflow d'exécution

```
1. Connexion établie
   → Assignée au groupe G (round-robin)

2. Requête arrive
   → Ajoutée à la queue du groupe G

3. Thread disponible dans groupe G
   → Prend la requête de la queue
   → Exécute la requête
   → Retourne dans le pool

4. Si tous les threads occupés (stall détecté)
   → Création d'un nouveau thread (jusqu'à thread_pool_max_threads)
```

### Détection de stall

Un **stall** se produit quand tous les threads d'un groupe sont **bloqués** (requêtes longues, locks).

**Mécanisme** :
```
Timer = thread_pool_stall_limit (500 ms par défaut)

Si TOUTES les requêtes du groupe prennent > 500 ms
    → Stall détecté
    → Création d'un nouveau thread
    → Évite le blocage complet du groupe
```

---

## Configuration du Thread Pool

### Activation

```ini
# my.cnf
[mysqld]
# Activer le Thread Pool
thread_handling = pool-of-threads

# Alternative: one-thread-per-connection (défaut)
# thread_handling = one-thread-per-connection
```

```sql
-- Vérifier le mode actif
SHOW VARIABLES LIKE 'thread_handling';
-- pool-of-threads ou one-thread-per-connection
```

### Variables principales

#### thread_pool_size

**Définition** : Nombre de **groupes** de threads.

```sql
-- Valeur actuelle
SHOW VARIABLES LIKE 'thread_pool_size';

-- Recommandation : = Nombre de CPU cores
SET GLOBAL thread_pool_size = 16;  -- Pour serveur 16 cores
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Serveur 8 cores
thread_pool_size = 8

# Serveur 16 cores
thread_pool_size = 16

# Serveur 32 cores
thread_pool_size = 32
```

**Règles** :
- **Minimum** : 1
- **Maximum** : 128 (théorique), 64 (pratique)
- **Optimal** : Nombre de CPU cores (physiques, pas logiques avec HT)

#### thread_pool_max_threads

**Définition** : Nombre **maximal** de threads dans tout le pool.

```sql
SHOW VARIABLES LIKE 'thread_pool_max_threads';

-- Recommandation : 500-2000 selon charge
SET GLOBAL thread_pool_max_threads = 1000;
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Limite totale de threads
thread_pool_max_threads = 1000

# Charge très élevée
thread_pool_max_threads = 2000

# Charge modérée
thread_pool_max_threads = 500
```

**Calcul** :
```
thread_pool_max_threads =
    thread_pool_size * thread_pool_oversubscribe * facteur_sécurité

Exemple:
    16 groupes * 3 threads/groupe * 20 (facteur) = 960 ≈ 1000
```

#### thread_pool_idle_timeout

**Définition** : Temps avant qu'un thread **inactif** soit terminé.

```sql
SHOW VARIABLES LIKE 'thread_pool_idle_timeout';

SET GLOBAL thread_pool_idle_timeout = 60;  -- 60 secondes
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Timeout thread inactif (secondes)
thread_pool_idle_timeout = 60

# Charge stable : timeout court
thread_pool_idle_timeout = 30

# Charge variable : timeout long
thread_pool_idle_timeout = 120
```

**Impact** :
- **Court** (30s) : Libère rapidement les ressources
- **Long** (120s) : Réutilisation meilleure, moins de créations

#### thread_pool_stall_limit

**Définition** : Temps (ms) avant de détecter un **stall** et créer un nouveau thread.

```sql
SHOW VARIABLES LIKE 'thread_pool_stall_limit';

SET GLOBAL thread_pool_stall_limit = 500;  -- 500 ms
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Détection stall (millisecondes)
thread_pool_stall_limit = 500

# Requêtes courtes (OLTP)
thread_pool_stall_limit = 100

# Requêtes longues (analytique)
thread_pool_stall_limit = 2000
```

**Impact** :
- **Court** (100ms) : Réactivité élevée, plus de threads créés
- **Long** (2000ms) : Moins de threads, mais risque de latence

#### thread_pool_oversubscribe

**Définition** : Nombre de threads **actifs simultanément** par groupe.

```sql
SHOW VARIABLES LIKE 'thread_pool_oversubscribe';

SET GLOBAL thread_pool_oversubscribe = 3;
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Threads actifs par groupe
thread_pool_oversubscribe = 3

# Workload I/O-bound (disque/réseau)
thread_pool_oversubscribe = 10

# Workload CPU-bound (calculs)
thread_pool_oversubscribe = 1
```

**Règle** :
```
Threads actifs par core = thread_pool_oversubscribe

Total threads actifs ≈ thread_pool_size * thread_pool_oversubscribe
```

#### thread_pool_priority

**Définition** : Stratégie de **priorisation** des requêtes.

```sql
SHOW VARIABLES LIKE 'thread_pool_priority';

-- Valeurs possibles :
-- auto (défaut) : Basé sur transaction state
-- high : Priorité haute
-- low : Priorité basse
```

**Configuration** :

```ini
# my.cnf
[mysqld]
thread_pool_priority = auto

# Options :
# - auto : Intelligent (recommandé)
# - high : Priorité transactions en cours
# - low : FIFO strict
```

---

## Configuration optimale par type de charge

### OLTP (haute concurrence, requêtes courtes)

```ini
# my.cnf - OLTP
[mysqld]
thread_handling = pool-of-threads

# Nombre de groupes = CPU cores
thread_pool_size = 16

# Limite élevée pour pics de charge
thread_pool_max_threads = 1000

# Timeout court (libération rapide)
thread_pool_idle_timeout = 30

# Détection stall rapide
thread_pool_stall_limit = 100

# Faible oversubscribe (CPU-bound)
thread_pool_oversubscribe = 3

# Priorité intelligente
thread_pool_priority = auto
```

**Justification** :
- Requêtes < 100ms en moyenne
- Haute concurrence (1000+ connexions)
- CPU-bound (index, joins rapides)

### OLAP / Analytique (requêtes longues)

```ini
# my.cnf - OLAP
[mysqld]
thread_handling = pool-of-threads

# Moins de groupes (éviter saturation CPU)
thread_pool_size = 8

# Limite modérée
thread_pool_max_threads = 200

# Timeout long (requêtes longues)
thread_pool_idle_timeout = 120

# Détection stall tolérante
thread_pool_stall_limit = 2000

# Oversubscribe élevé (I/O-bound)
thread_pool_oversubscribe = 10

thread_pool_priority = auto
```

**Justification** :
- Requêtes de plusieurs secondes
- Faible concurrence (10-50 requêtes simultanées)
- I/O-bound (scans de table, agrégations massives)

### Mixte (OLTP + batch)

```ini
# my.cnf - Mixte
[mysqld]
thread_handling = pool-of-threads

thread_pool_size = 16
thread_pool_max_threads = 800
thread_pool_idle_timeout = 60
thread_pool_stall_limit = 500
thread_pool_oversubscribe = 5
thread_pool_priority = auto
```

**Justification** :
- Équilibre entre réactivité et tolérance
- Adapté à la majorité des cas

---

## Monitoring du Thread Pool

### Variables de statut

```sql
-- Statistiques Thread Pool
SHOW STATUS LIKE 'Threadpool%';
```

**Métriques clés** :

| Variable | Description | Valeur optimale |
|----------|-------------|-----------------|
| `Threadpool_idle_threads` | Threads inactifs | > 0 (réserve disponible) |
| `Threadpool_threads` | Total threads actifs | < thread_pool_max_threads |
| `Threadpool_stall_count` | Nombre de stalls détectés | Faible (< 100/heure) |

### Analyse détaillée

```sql
-- État complet Thread Pool
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Threadpool_threads') AS total_threads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Threadpool_idle_threads') AS idle_threads,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Threads_connected') AS connections,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES
     WHERE VARIABLE_NAME = 'thread_pool_size') AS pool_groups,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES
     WHERE VARIABLE_NAME = 'thread_pool_max_threads') AS max_threads;
```

**Interprétation** :

```
Exemple :
    total_threads = 64
    idle_threads = 12
    connections = 500
    pool_groups = 16
    max_threads = 1000

Analyse :
    - 64 threads actifs pour 500 connexions ✅ Excellent
    - 12 threads idle = réserve disponible ✅ Bon
    - Bien en dessous de max_threads (1000) ✅ OK
    - Ratio connexions/threads = 7.8 ✅ Scalable
```

### Détection de problèmes

#### Stalls excessifs

```sql
-- Surveiller les stalls
SHOW STATUS LIKE 'Threadpool_stall_count';

-- Si augmentation rapide (> 100/minute)
-- → Requêtes trop longues bloquent le pool
```

**Solutions** :
1. Augmenter `thread_pool_stall_limit` (2000-5000ms)
2. Optimiser les requêtes lentes
3. Augmenter `thread_pool_max_threads`

#### Saturation du pool

```sql
-- Vérifier si proche de la limite
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Threadpool_threads') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES
     WHERE VARIABLE_NAME = 'thread_pool_max_threads') * 100
    AS utilization_pct;

-- Si > 80% → Risque de saturation
```

**Solutions** :
1. Augmenter `thread_pool_max_threads`
2. Réduire `thread_pool_stall_limit` (création moins agressive)
3. Optimiser les requêtes pour libérer threads plus vite

#### Pas de threads idle

```sql
SHOW STATUS LIKE 'Threadpool_idle_threads';
-- Threadpool_idle_threads = 0

-- → Tous les threads occupés en permanence
-- → Risque de latence accrue
```

**Solutions** :
1. Augmenter `thread_pool_max_threads`
2. Vérifier les requêtes longues (SHOW PROCESSLIST)
3. Optimiser la charge de travail

---

## Comparaison Thread Pool vs One-Thread-Per-Connection

### Benchmarks

| Métrique | One-Thread-Per-Connection | Thread Pool | Amélioration |
|----------|---------------------------|-------------|--------------|
| **Connexions max** | ~2,000 | ~10,000+ | **5x** |
| **Mémoire (10k conn)** | ~10-20 GB | ~100-200 MB | **50-100x** |
| **Context switches** | Élevé (milliers/sec) | Faible (centaines/sec) | **10x** |
| **Latency (1k conn)** | 50-100 ms | 5-10 ms | **5-10x** |
| **Throughput (QPS)** | 5,000 | 8,000-12,000 | **1.6-2.4x** |

### Cas d'usage recommandé

| Scénario | Recommandation | Justification |
|----------|----------------|---------------|
| **< 100 connexions** | One-Thread-Per-Connection | Overhead Thread Pool inutile |
| **100-500 connexions** | Thread Pool (optionnel) | Gain modéré |
| **> 500 connexions** | **Thread Pool** ✅ | Gain significatif |
| **> 1000 connexions** | **Thread Pool** ✅✅ | Indispensable |
| **Requêtes < 10ms** | **Thread Pool** | Excellente réutilisation |
| **Requêtes > 10s** | One-Thread-Per-Connection | Stalls fréquents |
| **SaaS multi-tenant** | **Thread Pool** | Isolation par groupe |

---

## Tuning avancé

### Optimisation par profiling

**Étape 1 : Baseline sans Thread Pool**

```sql
-- Désactiver Thread Pool
SET GLOBAL thread_handling = 'one-thread-per-connection';
-- Redémarrage nécessaire

-- Mesurer :
-- - QPS (Questions/sec)
-- - Latency moyenne
-- - Threads_connected
-- - CPU utilization
```

**Étape 2 : Activer Thread Pool avec config de base**

```ini
# my.cnf
[mysqld]
thread_handling = pool-of-threads
thread_pool_size = 16  # = CPU cores
thread_pool_max_threads = 500
```

**Étape 3 : Tuning itératif**

```bash
# 1. Charge de test (sysbench, application réelle)
sysbench oltp_read_write --threads=500 run

# 2. Mesurer métriques
# - Threadpool_stall_count (doit être faible)
# - Threadpool_idle_threads (doit être > 0)
# - QPS, latency

# 3. Ajuster
# Si stalls élevés → thread_pool_stall_limit + 500ms
# Si idle_threads = 0 → thread_pool_max_threads + 100
# Si CPU < 80% et QPS faible → thread_pool_oversubscribe + 1

# 4. Répéter jusqu'à optimum
```

### Formules de dimensionnement

```
thread_pool_size = Nombre de CPU cores physiques

thread_pool_max_threads =
    Si workload CPU-bound:
        thread_pool_size * 10
    Si workload I/O-bound:
        thread_pool_size * 50
    Si workload mixte:
        thread_pool_size * 30

thread_pool_oversubscribe =
    Si workload CPU-bound:
        1-3
    Si workload I/O-bound:
        10-20
    Si workload mixte:
        5

thread_pool_stall_limit =
    Si latence < 10ms cible:
        100-200 ms
    Si latence < 100ms cible:
        500 ms
    Si latence < 1s cible:
        2000 ms
```

---

## Troubleshooting

### Problème : Latence élevée avec Thread Pool

**Symptômes** :
- Requêtes simples prennent > 100ms
- Variabilité importante (p50 = 10ms, p99 = 500ms)

**Diagnostic** :

```sql
-- 1. Vérifier les stalls
SHOW STATUS LIKE 'Threadpool_stall_count';
-- Si augmente rapidement → Problème

-- 2. Vérifier threads idle
SHOW STATUS LIKE 'Threadpool_idle_threads';
-- Si = 0 → Saturation

-- 3. Identifier requêtes longues
SELECT ID, USER, TIME, STATE, INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC
LIMIT 20;
```

**Solutions** :
1. Augmenter `thread_pool_stall_limit` (500 → 2000ms)
2. Augmenter `thread_pool_max_threads` (500 → 1000)
3. Optimiser les requêtes lentes
4. Augmenter `thread_pool_size` si CPU sous-utilisé

### Problème : Stalls constants

**Symptômes** :
- `Threadpool_stall_count` augmente de 100+ par minute

**Causes** :
- Requêtes longues (> thread_pool_stall_limit)
- Locks (row locks, table locks)
- I/O lent (disque saturé)

**Solutions** :

```sql
-- 1. Identifier les blocages
SELECT
    r.trx_id AS waiting_trx,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r ON w.requesting_trx_id = r.trx_id
JOIN information_schema.INNODB_TRX b ON w.blocking_trx_id = b.trx_id;

-- 2. Tuer les requêtes bloquantes si nécessaire
KILL <blocking_thread_id>;
```

### Problème : CPU faible mais QPS limité

**Symptômes** :
- CPU à 30-50%
- QPS plafonné à 5,000 alors que capacité > 10,000

**Cause** : Configuration trop conservative

**Solutions** :

```sql
-- Augmenter oversubscribe
SET GLOBAL thread_pool_oversubscribe = 10;

-- Augmenter nombre de groupes (si CPU > 16 cores)
-- Nécessite redémarrage
```

---

## Migration vers Thread Pool

### Procédure de migration sécurisée

**Phase 1 : Test sur environnement staging**

```ini
# my.cnf - Staging
[mysqld]
thread_handling = pool-of-threads
thread_pool_size = 16
thread_pool_max_threads = 500
thread_pool_stall_limit = 500
```

**Phase 2 : Monitoring intensif (1 semaine)**

```bash
# Script de surveillance quotidien
#!/bin/bash
echo "=== Thread Pool Stats ==="
mariadb -e "SHOW STATUS LIKE 'Threadpool%'"
echo ""
echo "=== Performance ==="
mariadb -e "SHOW STATUS LIKE 'Questions'"
mariadb -e "SHOW STATUS LIKE 'Slow_queries'"
```

**Phase 3 : Déploiement production avec rollback plan**

```bash
# 1. Backup configuration actuelle
cp /etc/mysql/my.cnf /etc/mysql/my.cnf.backup-$(date +%Y%m%d)

# 2. Modifier configuration
vim /etc/mysql/my.cnf
# thread_handling = pool-of-threads

# 3. Redémarrer en fenêtre de maintenance
systemctl restart mariadb

# 4. Surveiller pendant 1 heure
watch -n 10 'mariadb -e "SHOW STATUS LIKE \"Threadpool%\""'

# 5. Si problème : Rollback immédiat
# cp /etc/mysql/my.cnf.backup-YYYYMMDD /etc/mysql/my.cnf
# systemctl restart mariadb
```

---

## Bonnes pratiques

### ✅ À FAIRE

1. **thread_pool_size = CPU cores** (point de départ optimal)
2. **Activer pour > 500 connexions** simultanées
3. **Monitorer Threadpool_stall_count** quotidiennement
4. **Tester sur staging** avant production
5. **Dimensionner thread_pool_max_threads** avec marge (2x besoin actuel)
6. **Ajuster thread_pool_stall_limit** selon latence cible
7. **Documenter la configuration** et les raisons
8. **Profiler avant/après** migration
9. **Plan de rollback** systématique
10. **Alertes** sur saturation du pool

### ❌ À ÉVITER

1. **Thread Pool avec < 100 connexions** (overhead inutile)
2. **thread_pool_size trop élevé** (> 2x CPU cores)
3. **thread_pool_max_threads trop faible** (< 100)
4. **Migration sans test** sur staging
5. **Ignorer les stalls** persistants
6. **Configuration identique OLTP/OLAP**
7. **Pas de monitoring** après activation
8. **thread_pool_stall_limit trop court** (< 100ms)

---

## Checklist de configuration Thread Pool

### Avant activation

- [ ] Nombre de CPU cores identifié
- [ ] Type de charge défini (OLTP/OLAP/mixte)
- [ ] Baseline de performance établie (QPS, latency)
- [ ] Test sur environnement staging
- [ ] Plan de rollback documenté
- [ ] Monitoring en place

### Configuration initiale

- [ ] `thread_handling = pool-of-threads`
- [ ] `thread_pool_size = <CPU_cores>`
- [ ] `thread_pool_max_threads = <dimensionné>`
- [ ] `thread_pool_stall_limit = <adapté>`
- [ ] `thread_pool_oversubscribe = <selon workload>`
- [ ] `thread_pool_idle_timeout = 60`

### Post-activation (surveillance 1 semaine)

- [ ] QPS maintenu ou amélioré
- [ ] Latency réduite
- [ ] Threadpool_stall_count acceptable (< 100/heure)
- [ ] Threadpool_idle_threads > 0
- [ ] CPU utilization optimale (60-80%)
- [ ] Pas de régression applicative
- [ ] Documentation mise à jour

---

## ✅ Points clés à retenir

- **Thread Pool** : Pool réutilisable vs 1 thread par connexion
- **Scalabilité** : 10,000+ connexions avec RAM constante
- **Architecture** : N groupes (= CPU cores) avec queue de requêtes
- **Activation** : `thread_handling = pool-of-threads`
- **thread_pool_size** : = Nombre de CPU cores (optimal)
- **thread_pool_max_threads** : Limite totale de threads
- **Stall** : Détection automatique de blocages (thread_pool_stall_limit)
- **Monitoring** : Threadpool_stall_count, Threadpool_idle_threads
- **OLTP** : Oversubscribe faible (3), stall_limit court (100ms)
- **OLAP** : Oversubscribe élevé (10), stall_limit long (2000ms)
- **Bénéfices** : > 500 connexions, 5-10x moins de mémoire, 2-3x meilleur QPS
- **Test obligatoire** : Staging avant production

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Thread Pool in MariaDB](https://mariadb.com/kb/en/thread-pool-in-mariadb/)
- [📖 Documentation officielle - Thread Pool Variables](https://mariadb.com/kb/en/thread-pool-system-and-status-variables/)
- [📖 Tuning Thread Pool for Best Performance](https://mariadb.com/kb/en/thread-pool-tuning/)
- [📊 Thread Pool Benchmarks](https://mariadb.com/resources/blog/thread-pool-in-mariadb/)
- [🔧 Performance Schema - Thread Monitoring](https://mariadb.com/kb/en/performance-schema/)

---

## ➡️ Sections suivantes

- **11.11 Charset par défaut : utf8mb4 avec UCA 14.0.0** 🆕 : Nouveauté MariaDB 11.8, Unicode complet par défaut
- **11.12 Extension TIMESTAMP 2106** 🆕 : Résolution du problème Y2038

---

**💡 Conseil final** : Le Thread Pool est comme un chef d'orchestre : il coordonne brillamment des milliers de musiciens (connexions) avec seulement quelques baguettes (threads). Activez-le pour haute concurrence, configurez intelligemment, et profitez d'une scalabilité exceptionnelle ! 🎼🚀

⏭️ [Charset par défaut : utf8mb4 avec collations UCA 14.0.0 (depuis 11.8)](/11-administration-configuration/11-charset-utf8mb4-uca14.md)

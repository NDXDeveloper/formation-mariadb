🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.12 Benchmarking

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Sections 15.1-15.11 (Performance, Optimisation, Sharding)
> - Compréhension des métriques de performance
> - Expérience en tuning de bases de données
> - Connaissance de l'architecture système (CPU, RAM, I/O)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre les types** de benchmarks et leurs objectifs
- **Définir une méthodologie** rigoureuse de benchmarking
- **Identifier les métriques** critiques à mesurer
- **Choisir les outils** appropriés pour chaque scénario
- **Interpréter les résultats** correctement et objectivement
- **Éviter les pièges** courants du benchmarking
- **Comparer des configurations** de manière fiable
- **Valider les optimisations** avant production
- **Réaliser le capacity planning** basé sur des données réelles
- **Documenter et communiquer** les résultats efficacement

---

## Introduction

Le **benchmarking** est le processus de mesure systématique et reproductible des performances d'un système de base de données. C'est un outil **essentiel** pour :

```
┌────────────────────────────────────────────────────┐
│  POURQUOI BENCHMARKER ?                            │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. VALIDER LES OPTIMISATIONS                      │
│     • Avant/après modification                     │
│     • Gain réel vs gain espéré                     │
│     • ROI des investissements                      │
│                                                    │
│  2. COMPARER DES CONFIGURATIONS                    │
│     • Matériel : SSD vs NVMe                       │
│     • Configuration : buffer pool 50% vs 75%       │
│     • Architecture : standalone vs sharded         │
│                                                    │
│  3. CAPACITY PLANNING                              │
│     • Quelle charge maximale ?                     │
│     • Quand scaler ?                               │
│     • Dimensionnement futur                        │
│                                                    │
│  4. IDENTIFIER LES BOTTLENECKS                     │
│     • CPU bound vs I/O bound                       │
│     • Limites mémoire                              │
│     • Saturation réseau                            │
│                                                    │
│  5. ACCEPTATION PRÉ-PRODUCTION                     │
│     • Validation SLA                               │
│     • Tests de charge                              │
│     • Certification performance                    │
│                                                    │
│  6. RÉGRESSION TESTING                             │
│     • Nouvelle version MariaDB                     │
│     • Changement configuration                     │
│     • Mise à jour système                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Le principe fondamental

> **"If you can't measure it, you can't improve it."**  
> — Peter Drucker

```
┌────────────────────────────────────────────────────┐
│  CYCLE D'OPTIMISATION                              │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. MESURER (Baseline)                             │
│     ↓                                              │
│  2. ANALYSER (Identifier bottleneck)               │
│     ↓                                              │
│  3. OPTIMISER (Implémenter changement)             │
│     ↓                                              │
│  4. BENCHMARKER (Mesurer nouveau)                  │
│     ↓                                              │
│  5. COMPARER (Gain réel ?)                         │
│     ↓                                              │
│  6. VALIDER ou ROLLBACK                            │
│     ↓                                              │
│  7. DOCUMENTER                                     │
│     ↓                                              │
│  RÉPÉTER ───────────────┘                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Types de benchmarks

### 1. Benchmarks synthétiques

**Définition** : Tests standardisés avec charge artificielle.

```
┌────────────────────────────────────────────────────┐
│  BENCHMARKS SYNTHÉTIQUES                           │
├────────────────────────────────────────────────────┤
│                                                    │
│  Caractéristiques :                                │
│  • Workload prédéfini et répétable                 │
│  • Patterns simples (SELECT, INSERT, UPDATE)       │
│  • Comparaison facile entre systèmes               │
│  • Résultats standardisés                          │
│                                                    │
│  Outils :                                          │
│  • sysbench (MySQL benchmark standard)             │
│  • TPC-C, TPC-H (standards industrie)              │
│  • mysqlslap (outil natif)                         │
│                                                    │
│  Avantages :                                       │
│  ✅ Reproductible                                  │
│  ✅ Comparaison objective                          │
│  ✅ Facile à automatiser                           │
│  ✅ Baseline rapide                                │
│                                                    │
│  Inconvénients :                                   │
│  ❌ Pas représentatif du workload réel             │
│  ❌ Peut masquer problèmes spécifiques             │
│  ❌ Optimisations artificielles possibles          │
│                                                    │
│  Quand utiliser :                                  │
│  • Comparer matériel                               │
│  • Tester changement configuration                 │
│  • Baseline initiale                               │
│  • Tests de régression                             │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Exemple** :

```bash
# sysbench OLTP read-only
sysbench oltp_read_only \
  --mysql-host=localhost \
  --mysql-user=bench \
  --mysql-password=bench \
  --tables=10 \
  --table-size=1000000 \
  run

# Résultat type :
# Transactions: 125000 (2083.33 per sec)
# Queries: 1000000 (16666.67 per sec)
# Latency p95: 12.52ms
```

### 2. Benchmarks applicatifs (Real-world)

**Définition** : Tests basés sur le workload réel de production.

```
┌────────────────────────────────────────────────────┐
│  BENCHMARKS APPLICATIFS                            │
├────────────────────────────────────────────────────┤
│                                                    │
│  Caractéristiques :                                │
│  • Rejeu de traces production                      │
│  • Queries réelles de l'application                │
│  • Distribution de charge authentique              │
│  • Patterns complexes (JOINs, subqueries)          │
│                                                    │
│  Méthodes :                                        │
│  • Capture slow query log production               │
│  • Rejeu avec pt-query-digest                      │
│  • Tests de charge avec JMeter/Gatling             │
│  • Clone de traffic (tcpcopy)                      │
│                                                    │
│  Avantages :                                       │
│  ✅ Représentatif du workload réel                 │
│  ✅ Détecte problèmes spécifiques app              │
│  ✅ Validation pré-production fiable               │
│                                                    │
│  Inconvénients :                                   │
│  ❌ Complexe à mettre en place                     │
│  ❌ Difficile à reproduire exactement              │
│  ❌ Nécessite données production                   │
│  ❌ Chronophage                                    │
│                                                    │
│  Quand utiliser :                                  │
│  • Validation finale avant déploiement             │
│  • Capacity planning précis                        │
│  • Tests de migration                              │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Exemple** :

```bash
# Capturer queries production (1 heure)
# Analyser avec pt-query-digest
pt-query-digest /var/log/mysql/slow.log > queries_production.txt

# Extraire top 100 queries
# Créer script de rejeu
# Exécuter sur serveur test avec même données
```

### 3. Benchmarks de stress

**Définition** : Tests aux limites du système.

```
┌────────────────────────────────────────────────────┐
│  BENCHMARKS DE STRESS                              │
├────────────────────────────────────────────────────┤
│                                                    │
│  Objectifs :                                       │
│  • Trouver limites maximales                       │
│  • Point de rupture (breaking point)               │
│  • Comportement sous saturation                    │
│  • Temps de récupération                           │
│                                                    │
│  Scénarios :                                       │
│  • Augmentation progressive de charge              │
│  • Pics soudains de trafic                         │
│  • Charge soutenue longue durée                    │
│  • Ressources limitées (RAM, CPU, I/O)             │
│                                                    │
│  Métriques :                                       │
│  • Point de saturation (TPS max)                   │
│  • Dégradation latence                             │
│  • Taux d'erreurs                                  │
│  • Temps de récupération                           │
│                                                    │
│  Cas d'usage :                                     │
│  • Dimensionnement infrastructure                  │
│  • Tests de résilience                             │
│  • Validation SLA                                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Méthodologie de benchmarking

### Les 7 règles d'or

```
┌────────────────────────────────────────────────────┐
│  RÈGLES FONDAMENTALES DU BENCHMARKING              │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. ISOLER L'ENVIRONNEMENT                         │
│     • Serveur dédié au benchmark                   │
│     • Pas d'autres processus concurrents           │
│     • Réseau stable et prévisible                  │
│                                                    │
│  2. ENVIRONNEMENT IDENTIQUE                        │
│     • Même version MariaDB                         │
│     • Même configuration OS                        │
│     • Même jeu de données                          │
│     • Même matériel (si comparaison config)        │
│                                                    │
│  3. WARMUP AVANT MESURE                            │
│     • Buffer pool chaud                            │
│     • Caches OS remplis                            │
│     • État stable du système                       │
│     • Min 5-10 minutes de warmup                   │
│                                                    │
│  4. DURÉE SUFFISANTE                               │
│     • Minimum 30 minutes par test                  │
│     • Idéal : 1-2 heures                           │
│     • Capturer variations temporelles              │
│                                                    │
│  5. RÉPÉTABILITÉ                                   │
│     • Exécuter 3-5 fois minimum                    │
│     • Calculer moyenne et écart-type               │
│     • Écart-type < 5% du résultat                  │
│                                                    │
│  6. UNE VARIABLE À LA FOIS                         │
│     • Ne changer qu'un paramètre                   │
│     • Isoler l'impact du changement                │
│     • Éviter conclusions erronées                  │
│                                                    │
│  7. DOCUMENTER TOUT                                │
│     • Configuration complète                       │
│     • Commandes exécutées                          │
│     • Résultats bruts                              │
│     • Conditions du test                           │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Processus étape par étape

```
┌────────────────────────────────────────────────────┐
│  PROCESSUS BENCHMARK COMPLET                       │
├────────────────────────────────────────────────────┤
│                                                    │
│  PHASE 1 : PRÉPARATION (1-2 jours)                 │
│  ────────────────────────────                      │
│  □ Définir objectif du benchmark                   │
│  □ Choisir métriques à mesurer                     │
│  □ Sélectionner outil approprié                    │
│  □ Provisionner infrastructure                     │
│  □ Installer et configurer MariaDB                 │
│  □ Charger jeu de données représentatif            │
│  □ Documenter baseline (hardware, config)          │
│                                                    │
│  PHASE 2 : WARMUP (15-30 min)                      │
│  ─────────────────────────                         │
│  □ Démarrer MariaDB                                │
│  □ Exécuter workload léger                         │
│  □ Vérifier buffer pool hit rate > 99%             │
│  □ Vérifier stabilité métriques système            │
│                                                    │
│  PHASE 3 : BASELINE (1-2 heures)                   │
│  ────────────────────────                          │
│  □ Exécuter benchmark configuration actuelle       │
│  □ Capturer toutes métriques                       │
│  □ Répéter 3 fois                                  │
│  □ Calculer moyenne et écart-type                  │
│  □ Sauvegarder résultats bruts                     │
│                                                    │
│  PHASE 4 : MODIFICATION (30 min)                   │
│  ─────────────────────────                         │
│  □ Appliquer changement (un seul!)                 │
│  □ Redémarrer MariaDB si nécessaire                │
│  □ Documenter le changement                        │
│                                                    │
│  PHASE 5 : TEST (1-2 heures)                       │
│  ────────────────────                              │
│  □ Warmup à nouveau                                │
│  □ Exécuter même benchmark                         │
│  □ Capturer toutes métriques                       │
│  □ Répéter 3 fois                                  │
│  □ Calculer moyenne et écart-type                  │
│                                                    │
│  PHASE 6 : ANALYSE (1-2 heures)                    │
│  ───────────────────────                           │
│  □ Comparer baseline vs nouveau                    │
│  □ Calculer % amélioration                         │
│  □ Vérifier significativité statistique            │
│  □ Analyser métriques secondaires                  │
│  □ Identifier effets de bord                       │
│                                                    │
│  PHASE 7 : RAPPORT (1 heure)                       │
│  ─────────────────────                             │
│  □ Synthèse résultats                              │
│  □ Graphiques comparatifs                          │
│  □ Recommandations                                 │
│  □ Archivage documentation                         │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Métriques critiques

### Métriques de throughput

```
┌────────────────────────────────────────────────────┐
│  MÉTRIQUES DE DÉBIT (Throughput)                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  TPS (Transactions Per Second)                     │
│  ────────────────────────────────                  │
│  • Nombre de transactions complètes/sec            │
│  • Métrique principale pour OLTP                   │
│  • Plus élevé = meilleur                           │
│  • Exemple : 5,000 TPS → 10,000 TPS = +100%        │
│                                                    │
│  QPS (Queries Per Second)                          │
│  ───────────────────────────                       │
│  • Nombre de requêtes SQL/sec                      │
│  • Inclut SELECT, INSERT, UPDATE, DELETE           │
│  • Exemple : 50,000 QPS                            │
│                                                    │
│  Reads/sec et Writes/sec                           │
│  ─────────────────────────                         │
│  • Débit lecture et écriture séparés               │
│  • Identifier workload read-heavy vs write-heavy   │
│  • Exemple : 80,000 reads/sec, 5,000 writes/sec    │
│                                                    │
│  MB/sec (I/O throughput)                           │
│  ──────────────────────                            │
│  • Volume de données transferées                   │
│  • Identifier saturation I/O                       │
│  • Exemple : 500 MB/sec reads                      │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Métriques de latence

```
┌────────────────────────────────────────────────────┐
│  MÉTRIQUES DE LATENCE (Response Time)              │
├────────────────────────────────────────────────────┤
│                                                    │
│  Latence moyenne (Average)                         │
│  ────────────────────────                          │
│  • Temps moyen d'exécution requête                 │
│  • Indicateur général                              │
│  • ⚠️ Peut masquer outliers                        │
│                                                    │
│  Latence médiane (p50)                             │
│  ────────────────────                              │
│  • 50% des requêtes plus rapides                   │
│  • Expérience utilisateur "typique"                │
│  • Moins sensible aux outliers                     │
│                                                    │
│  Latence p95, p99, p999                            │
│  ─────────────────────────                         │
│  • 95%, 99%, 99.9% des requêtes                    │
│  • CRITIQUES pour SLA                              │
│  • Détecte problèmes ponctuels                     │
│  • Exemple :                                       │
│    p50 = 5ms (bon)                                 │
│    p95 = 50ms (acceptable)                         │
│    p99 = 500ms (problème!)                         │
│                                                    │
│  Latence max                                       │
│  ──────────                                        │
│  • Pire cas observé                                │
│  • Outlier extrême                                 │
│  • Moins utile car peut être anomalie              │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Visualisation** :

```
Distribution de latence (exemple) :

Percentile  |  Latency
─────────────────────────
p0   (min)  |  1ms     ████
p25         |  3ms     ██████████████
p50  (med)  |  5ms     ████████████████████
p75         |  8ms     ████████████████
p90         |  12ms    ██████████
p95         |  15ms    ██████
p99         |  50ms    ██
p99.9       |  200ms   █
p100 (max)  |  5000ms  (outlier)

Important : p95-p99 pour SLA !
```

### Métriques système

```
┌────────────────────────────────────────────────────┐
│  MÉTRIQUES SYSTÈME                                 │
├────────────────────────────────────────────────────┤
│                                                    │
│  CPU                                               │
│  ───                                               │
│  • Utilisation % (user, system, iowait)            │
│  • Load average (1min, 5min, 15min)                │
│  • Context switches                                │
│  • Cible : <80% utilisation, pas de saturation     │
│                                                    │
│  Mémoire                                           │
│  ───────                                           │
│  • RAM utilisée / totale                           │
│  • Buffer pool hit rate (>99% cible)               │
│  • Swap utilisation (0 idéal)                      │
│  • Page faults                                     │
│                                                    │
│  I/O Disque                                        │
│  ──────────                                        │
│  • IOPS (read, write)                              │
│  • MB/sec throughput                               │
│  • Latency (await, svctm)                          │
│  • %util (< 90% recommandé)                        │
│  • Queue depth                                     │
│                                                    │
│  Réseau                                            │
│  ──────                                            │
│  • Bytes in/out                                    │
│  • Packets in/out                                  │
│  • Errors, drops                                   │
│  • Saturation liens                                │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Métriques MariaDB internes

```sql
-- Métriques critiques MariaDB

-- Threads
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';
SHOW STATUS LIKE 'Max_used_connections';

-- Queries
SHOW STATUS LIKE 'Questions';  -- Total queries
SHOW STATUS LIKE 'Queries';    -- Total + COM_* commands
SHOW STATUS LIKE 'Slow_queries';

-- InnoDB Buffer Pool
SHOW STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW STATUS LIKE 'Innodb_buffer_pool_reads';  -- Disk reads
-- Hit rate = (requests - reads) / requests * 100

-- InnoDB I/O
SHOW STATUS LIKE 'Innodb_data_reads';
SHOW STATUS LIKE 'Innodb_data_writes';
SHOW STATUS LIKE 'Innodb_data_fsyncs';

-- Locks
SHOW STATUS LIKE 'Innodb_row_lock_waits';
SHOW STATUS LIKE 'Innodb_row_lock_time_avg';
```

---

## Vue d'ensemble des outils

### Classification des outils

```
┌────────────────────────────────────────────────────┐
│  OUTILS DE BENCHMARKING                            │
├────────────────────────────────────────────────────┤
│                                                    │
│  BENCHMARKS SYNTHÉTIQUES                           │
│  ─────────────────────────                         │
│  • sysbench ⭐                                     │
│    - Standard industrie                            │
│    - OLTP, I/O, CPU tests                          │
│    - Très configurable                             │
│                                                    │
│  • mysqlslap                                       │
│    - Intégré à MariaDB                             │
│    - Simple et rapide                              │
│    - Moins puissant que sysbench                   │
│                                                    │
│  • TPC-C, TPC-H                                    │
│    - Standards industrie                           │
│    - Complexes à mettre en place                   │
│    - Comparaisons officielles                      │
│                                                    │
│  BENCHMARKS APPLICATIFS                            │
│  ────────────────────────                          │
│  • pt-query-digest (Percona)                       │
│    - Analyse slow query log                        │
│    - Replay queries                                │
│                                                    │
│  • Apache JMeter                                   │
│    - Tests de charge HTTP                          │
│    - Scripts complexes                             │
│                                                    │
│  • Gatling                                         │
│    - Tests de charge modernes                      │
│    - DSL Scala                                     │
│                                                    │
│  MONITORING                                        │
│  ──────────                                        │
│  • PMM (Percona Monitoring)                        │
│    - Dashboards temps réel                         │
│    - Historique long terme                         │
│                                                    │
│  • Grafana + Prometheus                            │
│    - Visualisation métriques                       │
│    - Alerting                                      │
│                                                    │
│  PROFILING                                         │
│  ─────────                                         │
│  • perf (Linux)                                    │
│    - CPU profiling                                 │
│    - Flamegraphs                                   │
│                                                    │
│  • iotop, iostat                                   │
│    - I/O monitoring                                │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Interprétation des résultats

### Analyse statistique

```
┌────────────────────────────────────────────────────┐
│  ANALYSE STATISTIQUE DES RÉSULTATS                 │
├────────────────────────────────────────────────────┤
│                                                    │
│  Exemple : Test buffer pool 50% vs 75%             │
│                                                    │
│  Configuration A (50% RAM) - 3 runs :              │
│  Run 1 : 8,234 TPS                                 │
│  Run 2 : 8,156 TPS                                 │
│  Run 3 : 8,301 TPS                                 │
│  Moyenne : 8,230 TPS                               │
│  Écart-type : 72.5 TPS (0.88%)                     │
│                                                    │
│  Configuration B (75% RAM) - 3 runs :              │
│  Run 1 : 10,523 TPS                                │
│  Run 2 : 10,489 TPS                                │
│  Run 3 : 10,556 TPS                                │
│  Moyenne : 10,523 TPS                              │
│  Écart-type : 33.5 TPS (0.32%)                     │
│                                                    │
│  Amélioration :                                    │
│  (10,523 - 8,230) / 8,230 × 100 = +27.9%           │
│                                                    │
│  Significativité :                                 │
│  Différence : 2,293 TPS                            │
│  Écart-types combinés : √(72.5² + 33.5²) = 80      │
│  Ratio : 2,293 / 80 = 28.7                         │
│  → Très significatif (ratio >> 2)                  │
│                                                    │
│  Conclusion :                                      │
│  ✅ Amélioration réelle de ~28%                    │
│  ✅ Statistiquement significative                  │
│  ✅ Résultats répétables (faible écart-type)       │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Règle de significativité

```
Différence significative si :

1. Écart-type < 5% de la moyenne
   → Résultats stables

2. Amélioration > 2 × écart-type
   → Changement réel, pas aléatoire

3. Minimum 3 runs
   → Confiance statistique

Exemple :
Moyenne A : 1000 TPS ± 50 (5%)
Moyenne B : 1200 TPS ± 40 (3.3%)
Différence : 200 TPS
2 × écart-types : 2 × √(50² + 40²) = 128
200 > 128 → SIGNIFICATIF ✅
```

### Identification des bottlenecks

```
┌────────────────────────────────────────────────────┐
│  IDENTIFIER LE BOTTLENECK                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  CPU-bound :                                       │
│  ──────────                                        │
│  • CPU utilisation > 80%                           │
│  • iowait < 10%                                    │
│  • Disk %util < 50%                                │
│  → Solution : Plus de CPU cores ou optimiser queries
│                                                    │
│  I/O-bound :                                       │
│  ─────────                                         │
│  • iowait > 20%                                    │
│  • Disk %util > 80%                                │
│  • CPU idle > 40%                                  │
│  → Solution : SSD/NVMe, plus d'IOPS                │
│                                                    │
│  Memory-bound :                                    │
│  ────────────                                      │
│  • Buffer pool hit rate < 95%                      │
│  • Swap utilisation > 0                            │
│  • Page faults élevés                              │
│  → Solution : Plus de RAM                          │
│                                                    │
│  Lock contention :                                 │
│  ───────────────                                   │
│  • Innodb_row_lock_waits élevé                     │
│  • Threads_running << Threads_connected            │
│  • CPU faible mais TPS faible                      │
│  → Solution : Optimiser transactions, partitioning │
│                                                    │
│  Network-bound :                                   │
│  ─────────────                                     │
│  • Bytes_sent/received proche limite réseau        │
│  • Retransmissions TCP élevées                     │
│  → Solution : Optimiser queries, compression       │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Pièges courants à éviter

### Les 10 erreurs fréquentes

```
┌────────────────────────────────────────────────────┐
│  PIÈGES À ÉVITER                                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. ❌ Pas de warmup                               │
│     ✅ Toujours warmer 5-10 minutes                │
│                                                    │
│  2. ❌ Test trop court (<5 min)                    │
│     ✅ Minimum 30 minutes                          │
│                                                    │
│  3. ❌ Un seul run                                 │
│     ✅ 3-5 runs minimum                            │
│                                                    │
│  4. ❌ Environnement différent                     │
│     ✅ Exact même setup pour comparaison           │
│                                                    │
│  5. ❌ Multiples changements simultanés            │
│     ✅ Un paramètre à la fois                      │
│                                                    │
│  6. ❌ Jeu de données trop petit                   │
│     ✅ > 2x RAM pour éviter cache complet          │
│                                                    │
│  7. ❌ Ignorer métriques système                   │
│     ✅ Toujours monitorer CPU, I/O, RAM            │
│                                                    │
│  8. ❌ Comparer versions différentes               │
│     ✅ Même version MariaDB/OS pour comparaison    │
│                                                    │
│  9. ❌ Négliger variance                           │
│     ✅ Calculer écart-type, vérifier répétabilité  │
│                                                    │
│  10. ❌ Pas de documentation                       │
│      ✅ Tout documenter (config, commandes)        │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Exemple d'erreur classique

```
❌ MAUVAIS BENCHMARK :

# Test 1 : Configuration A
sysbench oltp_read_only ... run  # 1 minute
# Résultat : 5000 TPS

# Changer buffer pool size
# Redémarrer MariaDB

# Test 2 : Configuration B
sysbench oltp_read_only ... run  # 1 minute
# Résultat : 8000 TPS

# Conclusion : +60% de gain ! 🎉

PROBLÈMES :
• Pas de warmup après redémarrage (cache froid)
• Un seul run (pas de variance)
• Test trop court (1 min)
• Pas de métriques système vérifiées
→ Résultats NON fiables


✅ BON BENCHMARK :

# Test 1 : Configuration A
# Redémarrer MariaDB
sysbench ... prepare
sysbench ... warmup --time=600  # 10 min warmup
sysbench ... run --time=3600    # 1h run (1/3)
# Nettoyer et répéter 2 fois
# Moyenne de 3 runs : 5,234 TPS ± 123

# Test 2 : Configuration B  
# Redémarrer MariaDB
sysbench ... warmup --time=600
sysbench ... run --time=3600    # 1h run (1/3)
# Répéter 2 fois
# Moyenne de 3 runs : 5,456 TPS ± 98

# Amélioration : (5456 - 5234) / 5234 = +4.2%
# Écart-type combiné : 154
# Différence / écart : 222 / 154 = 1.44
→ Amélioration MARGINALE, peu significative
→ Autre facteur à optimiser prioritaire
```

---

## Documentation et rapports

### Template de rapport

```markdown
# Benchmark Report

## Objectif
Comparer performances MariaDB 11.7 vs 11.8 (cost optimizer SSD)

## Configuration
**Hardware :**
- CPU : Intel Xeon E5-2680 v4 (28 cores, 56 threads)
- RAM : 256 GB DDR4
- Stockage : NVMe Gen3 Samsung 980 Pro (2 TB)
- Réseau : 10 Gbps

**Software :**
- OS : Ubuntu 22.04 LTS
- MariaDB A : 11.7.1
- MariaDB B : 11.8.0
- Kernel : 5.15.0

**Dataset :**
- Tables : 10 tables
- Lignes par table : 10M
- Taille totale : 80 GB
- Distribution : TPC-C like

## Méthodologie
- Outil : sysbench 1.0.20
- Test : oltp_read_write
- Threads : 64
- Warmup : 10 minutes
- Durée test : 60 minutes
- Runs : 3 par configuration

## Résultats

### Configuration A (MariaDB 11.7.1)
Run 1 : 12,456 TPS
Run 2 : 12,389 TPS
Run 3 : 12,501 TPS
**Moyenne : 12,449 TPS ± 56 (0.45%)**

### Configuration B (MariaDB 11.8.0)
Run 1 : 13,234 TPS
Run 2 : 13,189 TPS
Run 3 : 13,267 TPS
**Moyenne : 13,230 TPS ± 39 (0.29%)**

### Amélioration
**+6.3%** (781 TPS)
Statistiquement significatif (> 2σ)

## Métriques secondaires

| Metric          | 11.7   | 11.8   | Diff   |
|-----------------|--------|--------|--------|
| Latency p95     | 8.2ms  | 7.6ms  | -7.3%  |
| Latency p99     | 15.3ms | 14.1ms | -7.8%  |
| CPU %           | 72%    | 68%    | -5.6%  |
| BP hit rate     | 99.2%  | 99.3%  | +0.1%  |

## Analyse
Le cost optimizer amélioré de 11.8 réduit le nombre de full scans
sur SSD, résultant en :
- Throughput supérieur (+6.3%)
- Latence réduite (~7% sur p95/p99)
- CPU légèrement moins sollicité

## Recommandation
✅ Migration vers 11.8 recommandée
Gain de performance significatif sans risque identifié

## Annexes
- Configuration complète : config_11.7.cnf, config_11.8.cnf
- Logs complets : benchmark_11.7.log, benchmark_11.8.log
- Graphiques : graphs/
```

---

## ✅ Points clés à retenir

- 📊 **Mesurer = améliorer** : Sans benchmark, optimisation aveugle
- 🎯 **3 types** : Synthétique, applicatif, stress
- 📏 **Méthodologie rigoureuse** : 7 règles d'or à respecter
- 🔢 **Métriques multiples** : TPS, latence (p95/p99), système
- 🔄 **Répétabilité** : 3-5 runs minimum, écart-type <5%
- ⏱️ **Warmup essentiel** : 5-10 min avant mesure
- 📈 **Analyse statistique** : Vérifier significativité
- 🎛️ **Un changement à la fois** : Isolation des variables
- 📝 **Documentation** : Tout documenter pour reproductibilité
- ⚠️ **Éviter pièges** : Test trop court, pas de warmup, un seul run

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 MariaDB Performance Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)
- [📖 InnoDB Performance](https://mariadb.com/kb/en/innodb-performance/)

### Outils

- [sysbench](https://github.com/akopytov/sysbench)
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)
- [MySQL Performance Blog](https://www.percona.com/blog/)

### Standards industrie

- [TPC Benchmarks](http://www.tpc.org/)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent chaque outil de benchmarking :

### **Section 15.12.1** : [sysbench](/15-performance-tuning/12.1-sysbench.md)
*Outil de référence pour benchmarks synthétiques. Configuration, tests OLTP/I/O, interprétation.*

### **Section 15.12.2** : [mysqlslap](/15-performance-tuning/12.2-mysqlslap.md)
*Outil natif MariaDB pour tests rapides. Génération de charge, comparaisons simples.*


---

*"Premature optimization is the root of all evil, but informed optimization based on solid benchmarks is the path to performance heaven."*

⏭️ [sysbench](/15-performance-tuning/12.1-sysbench.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15. Performance et Tuning

> **Niveau** : Expert  
> **Durée estimée** : 8-10 heures  
> **Prérequis** : 
> - Maîtrise complète de MariaDB (chapitres 1-14)
> - Expérience en administration de bases de données en production
> - Connaissances en systèmes d'exploitation (Linux) et stockage
> - Compréhension des architectures matérielles (CPU, RAM, SSD/NVMe)

---

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Appliquer une méthodologie rigoureuse** d'optimisation des performances MariaDB
- **Dimensionner et configurer** la mémoire InnoDB de manière optimale pour votre charge de travail
- **Optimiser les performances I/O** en tenant compte des caractéristiques des SSD modernes
- **Exploiter les nouveautés MariaDB 11.8** pour améliorer les performances (innodb_alter_copy_bulk, cost optimizer SSD)
- **Analyser et optimiser** les requêtes lentes avec des outils professionnels
- **Maîtriser Performance Schema et sys schema** pour le diagnostic avancé
- **Concevoir et gérer** des stratégies de partitionnement efficaces
- **Mettre en œuvre** des techniques avancées d'optimisation en environnement de production

---

## Introduction

L'optimisation des performances d'une base de données MariaDB en production est un **art autant qu'une science**. Elle nécessite une compréhension approfondie de l'architecture du système, des patterns d'accès aux données, et des caractéristiques du matériel sous-jacent.

### Pourquoi ce chapitre est crucial

Dans un environnement de production moderne, les enjeux de performance sont multiples :

- **Temps de réponse** : Les utilisateurs s'attendent à des latences sub-secondes
- **Throughput** : Capacité à traiter des milliers de transactions par seconde
- **Scalabilité** : Anticiper la croissance des données et du trafic
- **Coûts** : Optimiser l'utilisation des ressources matérielles
- **Disponibilité** : Éviter les dégradations qui impactent la production

### Évolutions avec MariaDB 11.8 LTS

MariaDB 11.8 LTS apporte des **améliorations significatives** pour les environnements modernes :

🆕 **Nouveautés majeures pour la performance** :
- **innodb_alter_copy_bulk** : Construction d'index jusqu'à 40% plus rapide
- **Cost-based optimizer amélioré** : Prise en compte native des caractéristiques SSD/NVMe
- **Optimisations I/O** : Meilleures performances sur stockage moderne
- **Partitionnement avancé** : Conversion partition ↔ table sans reconstruction complète
- **Performance Schema enrichi** : Nouveaux instruments de monitoring

### Philosophy de l'optimisation

> 💡 **Principe fondamental** : "Mesurer avant d'optimiser, valider après chaque changement"

L'optimisation repose sur un cycle itératif :

1. **Mesurer** : Établir une baseline de performance
2. **Analyser** : Identifier les goulots d'étranglement (bottlenecks)
3. **Optimiser** : Appliquer des changements ciblés
4. **Valider** : Mesurer l'impact réel
5. **Documenter** : Tracer les modifications et leurs effets

---

## 📊 Vue d'ensemble du chapitre

Ce chapitre couvre l'ensemble du spectre de l'optimisation MariaDB, du niveau système à l'optimisation applicative.

### Architecture de l'optimisation

```
┌────────────────────────────────────────────────────────┐
│              MÉTHODOLOGIE D'OPTIMISATION               │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   HARDWARE   │  │   SYSTÈME    │  │   MARIADB    │  │
│  │              │  │              │  │              │  │
│  │  CPU, RAM    │  │  Filesystem  │  │  my.cnf      │  │
│  │  Storage     │  │  Scheduler   │  │  Variables   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │         OPTIMISATION MÉMOIRE (InnoDB)            │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  • Buffer Pool (70-80% RAM)                      │  │
│  │  • Buffer Pool Instances                         │  │
│  │  • Log Buffer                                    │  │
│  │  • Adaptive Hash Index                           │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │         OPTIMISATION I/O (SSD/NVMe) 🆕           │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  • innodb_io_capacity (SSD-aware)                │  │
│  │  • innodb_flush_method (O_DIRECT)                │  │
│  │  • Cost optimizer SSD 11.8                       │  │
│  │  • innodb_alter_copy_bulk 11.8                   │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │         ANALYSE ET MONITORING                    │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  • Slow Query Log + pt-query-digest              │  │
│  │  • Performance Schema                            │  │
│  │  • sys Schema                                    │  │
│  │  • EXPLAIN ANALYZE                               │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │         PARTITIONNEMENT & SHARDING               │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  • RANGE, LIST, HASH partitioning                │  │
│  │  • Partition pruning                             │  │
│  │  • Conversion partition ↔ table 🆕               │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Sections du chapitre

#### 🔍 **Méthodologie et Fondamentaux** (15.1)
- Approche systématique de l'optimisation
- Identification des bottlenecks
- Outils de diagnostic
- Baselines et benchmarking

#### 💾 **Configuration Mémoire** (15.2-15.3)
- InnoDB Buffer Pool : dimensionnement expert
- Buffer Pool instances et préchargement
- Query Cache : pourquoi il est déprécié
- Adaptive Hash Index

#### 💿 **Configuration I/O et Disques** (15.4-15.6)
- Optimisations pour SSD/NVMe modernes
- innodb_io_capacity et flush_method
- 🆕 innodb_alter_copy_bulk (11.8)
- 🆕 Cost-based optimizer SSD-aware (11.8)

#### 📈 **Analyse des Requêtes** (15.7-15.8)
- Slow Query Log : configuration avancée
- pt-query-digest : analyse professionnelle
- Performance Schema : instrumentation complète
- sys Schema : vues de diagnostic

#### 🗂️ **Partitionnement** (15.9-15.10)
- Stratégies RANGE, LIST, HASH
- Partition pruning et optimisations
- 🆕 Gestion avancée : conversion partition ↔ table
- Maintenance et reorganisation

#### 🎯 **Techniques Avancées** (15.11-15.14)
- Sharding et distribution horizontale
- Benchmarking avec sysbench
- Adaptive Hash Index
- Optimisations spécifiques au workload

---

## 🎓 Contexte : L'évolution des performances avec MariaDB 11.8

### Changements de paradigme

Traditionnellement, les SGBD relationnels ont été optimisés pour les **disques rotatifs (HDD)**, avec des stratégies comme :
- Minimiser les seeks (lectures aléatoires coûteuses)
- Maximiser les lectures séquentielles
- Utiliser agressivement le cache mémoire

Avec l'avènement des **SSD et NVMe**, ces hypothèses changent radicalement :

| Critère | HDD | SSD SATA | NVMe |
|---------|-----|----------|------|
| **IOPS aléatoires** | ~100-200 | ~50,000-100,000 | ~500,000-1,000,000 |
| **Latence lecture** | 5-10ms | 0.1-0.2ms | 0.02-0.05ms |
| **Bande passante** | 100-200 MB/s | 500-600 MB/s | 3,000-7,000 MB/s |
| **Coût seek** | Très élevé | Négligeable | Quasi-nul |

🆕 **MariaDB 11.8 est la première version LTS** à intégrer nativement ces caractéristiques dans son optimiseur de requêtes.

### Impact sur les stratégies d'optimisation

Avec MariaDB 11.8 sur SSD/NVMe :

✅ **Nouvelles bonnes pratiques** :
- Augmenter `innodb_io_capacity` significativement (10,000-20,000 pour NVMe)
- Utiliser `innodb_flush_method = O_DIRECT` sans compromis
- Activer `innodb_alter_copy_bulk` pour les DDL
- Laisser le cost optimizer gérer les SSD automatiquement

⚠️ **Anciennes pratiques à réévaluer** :
- Le buffer pool n'a plus besoin d'être aussi gigantesque (70% RAM au lieu de 80-90%)
- Les index covering sont moins critiques (random access rapide)
- Le partitionnement peut être moins nécessaire pour certains cas

---

## 💡 Principes clés de l'optimisation

### 1. Hiérarchie des optimisations

L'impact des optimisations suit généralement cet ordre (du plus au moins impactant) :

```
┌─────────────────────────────────────────────────┐
│ 1. SCHÉMA ET DESIGN (10x-100x)                  │
│    • Normalisation appropriée                   │
│    • Choix des types de données                 │
│    • Index stratégiques                         │
├─────────────────────────────────────────────────┤
│ 2. REQUÊTES SQL (5x-50x)                        │
│    • Élimination des N+1                        │
│    • Utilisation correcte des index             │
│    • Réduction du dataset                       │
├─────────────────────────────────────────────────┤
│ 3. CONFIGURATION MARIADB (2x-10x)               │
│    • Buffer pool sizing                         │
│    • Configuration I/O                          │
│    • Thread pool                                │
├─────────────────────────────────────────────────┤
│ 4. SYSTÈME ET MATÉRIEL (1.5x-5x)                │
│    • RAM suffisante                             │
│    • SSD/NVMe                                   │
│    • Filesystem (XFS, ext4)                     │
└─────────────────────────────────────────────────┘
```

> 💡 **Règle d'or** : Une mauvaise requête ne sera jamais rapide, même sur le meilleur matériel.

### 2. Les "Big Wins" universels

Certaines optimisations bénéficient à **presque tous les workloads** :

✅ **Toujours faire** :
- Utiliser InnoDB (pas MyISAM)
- Activer le binary logging en format ROW
- Configurer correctement le buffer pool (70-80% RAM)
- Utiliser des index appropriés
- Éviter `SELECT *`
- Utiliser des prepared statements

⚠️ **Toujours éviter** :
- Query Cache (déprécié, contre-productif)
- MyISAM pour les données transactionnelles
- Trop de connexions simultanées sans thread pool
- Binary logs sur le même disque que les données
- Swap actif sur le serveur

### 3. Mesurer pour décider

```sql
-- Exemple : Établir une baseline avant optimisation
SELECT 
    COUNT(*) as total_queries,
    SUM(query_time) as total_time,
    AVG(query_time) as avg_time,
    MAX(query_time) as max_time,
    SUM(rows_examined) as total_rows_examined,
    SUM(rows_examined) / COUNT(*) as avg_rows_per_query
FROM mysql.slow_log
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

---

## ⚠️ Pièges courants à éviter

### Erreurs fréquentes en optimisation

1. **Over-provisioning du buffer pool** (>90% RAM)
   - Laisse trop peu pour l'OS cache
   - Risque de swap
   - 🎯 **Recommandation 11.8** : 70-80% max avec SSD

2. **Modifier trop de paramètres à la fois**
   - Impossible d'isoler l'impact
   - Risque de régression
   - 🎯 **Recommandation** : Un changement à la fois, toujours mesurer

3. **Ignorer le monitoring continu**
   - Problèmes détectés trop tard
   - Pas de données historiques
   - 🎯 **Recommandation** : Performance Schema + Prometheus/Grafana

4. **Copier aveuglément des configurations "best practices"**
   - Chaque workload est unique
   - Ce qui marche pour l'OLTP peut nuire à l'OLAP
   - 🎯 **Recommandation** : Tester et adapter à votre cas

5. **Négliger l'analyse des requêtes**
   - Se concentrer uniquement sur la config serveur
   - Ignorer les requêtes N+1 ou les full table scans
   - 🎯 **Recommandation** : pt-query-digest hebdomadaire minimum

---

## 🔗 Structure du chapitre

### Sections détaillées

| Section | Titre | Nouveautés 11.8 | Niveau |
|---------|-------|-----------------|--------|
| **15.1** | Méthodologie d'optimisation | - | Expert |
| **15.2** | Configuration mémoire | - | Expert |
| **15.3** | Query Cache déprécié | - | Intermédiaire |
| **15.4** | Configuration I/O et disques | ✅ Cost optimizer SSD | Expert |
| **15.5** | Optimisation moteur InnoDB | - | Expert |
| **15.6** | innodb_alter_copy_bulk | 🆕 Feature 11.8 | Expert |
| **15.7** | Analyse requêtes lentes | - | Expert |
| **15.8** | Performance Schema et sys | - | Expert |
| **15.9** | Partitionnement de tables | - | Expert |
| **15.10** | Gestion avancée partitions | 🆕 Conversion partition↔table | Expert |
| **15.11** | Sharding et distribution | - | Expert |
| **15.12** | Benchmarking | - | Expert |
| **15.13** | Adaptive Hash Index | - | Expert |
| **15.14** | Cost optimizer SSD | 🆕 Feature 11.8 | Expert |

---

## 🎯 Objectifs par section

### Configuration et Optimisation Système

**15.1-15.6** : Maîtriser la configuration serveur pour performances optimales
- Appliquer une méthodologie rigoureuse d'audit et d'optimisation
- Dimensionner le buffer pool selon le workload et le matériel
- Configurer les paramètres I/O pour SSD/NVMe
- Exploiter innodb_alter_copy_bulk pour les migrations/DDL rapides

### Analyse et Diagnostic

**15.7-15.8** : Identifier et résoudre les problèmes de performance
- Configurer et exploiter le slow query log
- Utiliser pt-query-digest pour identifier les requêtes problématiques
- Maîtriser Performance Schema pour le diagnostic temps réel
- Utiliser sys schema pour des analyses rapides

### Scalabilité et Architecture

**15.9-15.12** : Concevoir des solutions scalables
- Choisir et implémenter la bonne stratégie de partitionnement
- Gérer dynamiquement les partitions en production
- Comprendre quand et comment sharding horizontalement
- Benchmarker pour valider les choix d'architecture

### Optimisations Avancées

**15.13-15.14** : Techniques expertes et nouveautés 11.8
- Comprendre et optimiser l'Adaptive Hash Index
- Exploiter le cost optimizer SSD-aware
- Appliquer des techniques avancées spécifiques au workload

---

## 📊 Métriques clés de performance

### Indicateurs à surveiller

```sql
-- Vue globale des performances système
SELECT 
    -- Temps de requête
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Questions') / UPTIME as queries_per_sec,
    
    -- Buffer pool
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') * 100 as buffer_pool_hit_rate,
    
    -- I/O
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_data_read') / 1024 / 1024 / UPTIME as mb_read_per_sec,
    
    -- Connexions
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_connected') as active_connections,
    
    -- Uptime
    TIME_FORMAT(SEC_TO_TIME(UPTIME), '%H:%i:%s') as uptime
FROM (
    SELECT VARIABLE_VALUE as UPTIME 
    FROM information_schema.GLOBAL_STATUS 
    WHERE VARIABLE_NAME = 'Uptime'
) u;
```

### Seuils d'alerte recommandés

| Métrique | Bon | Attention | Critique |
|----------|-----|-----------|----------|
| **Buffer Pool Hit Rate** | >99% | 95-99% | <95% |
| **Queries per second** | Variable | >10,000 | >50,000 |
| **Threads connected** | <80% max | 80-95% max | >95% max |
| **Slow queries** | <1% | 1-5% | >5% |
| **InnoDB Row Lock Waits** | <100/sec | 100-1000/sec | >1000/sec |
| **Temp tables on disk** | <5% | 5-15% | >15% |

---

## 🆕 Ce qui change avec MariaDB 11.8 LTS

### Améliorations majeures pour la performance

#### 1. innodb_alter_copy_bulk (15.6)

**Contexte** : La reconstruction d'index lors des ALTER TABLE était un processus lent, surtout sur de grandes tables.

🆕 **Innovation 11.8** :
```sql
-- Activation
SET GLOBAL innodb_alter_copy_bulk = ON;

-- Impact sur un ALTER TABLE avec reconstruction
ALTER TABLE large_table ADD INDEX idx_name (column_name);
-- Gain : jusqu'à 40% plus rapide sur SSD
```

**Cas d'usage** :
- Migrations de schéma sur tables volumineuses
- Ajout d'index en production
- Rebuild après suppression massive

#### 2. Cost-based Optimizer SSD-aware (15.14)

**Contexte** : L'optimiseur utilisait des coûts calibrés pour HDD (seek coûteux).

🆕 **Innovation 11.8** :
- Détection automatique du type de stockage
- Ajustement des coûts I/O pour SSD/NVMe
- Préférence pour random access quand approprié

```sql
-- L'optimiseur choisit maintenant mieux entre index scan et table scan
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;
-- Sur SSD : préfère index même si selectivité moyenne
-- Sur HDD : préférait table scan pour éviter seeks
```

#### 3. Optimisations I/O modernes (15.4)

🆕 **Nouveaux paramètres recommandés pour SSD/NVMe** :

```ini
# Configuration 11.8 optimisée SSD
[mariadb]
# I/O capacity augmentée pour SSD
innodb_io_capacity = 10000              # vs 200 pour HDD
innodb_io_capacity_max = 20000          # vs 2000 pour HDD

# Flush method optimal pour SSD
innodb_flush_method = O_DIRECT          # Évite double buffering

# Nouveau : Construction index rapide
innodb_alter_copy_bulk = ON

# Log buffer adapté
innodb_log_buffer_size = 64M            # Pour workload write-heavy
```

#### 4. Partition Management amélioré (15.10)

🆕 **Conversions partition ↔ table sans rebuild complet** :

```sql
-- Avant 11.8 : Rebuild complet (lent)
-- Avec 11.8 : Opération métadata (rapide)

-- Convertir une partition en table indépendante
ALTER TABLE orders_partitioned 
    EXCHANGE PARTITION p_2024_q1 
    WITH TABLE orders_2024_q1;
-- Quasi-instantané, même sur tables de plusieurs GB
```

---

## 💾 Prérequis matériels recommandés

Pour tirer pleinement parti des optimisations de ce chapitre :

### Configuration minimale (développement/test)
- **CPU** : 4 cores
- **RAM** : 8 GB
- **Storage** : SSD SATA 256 GB
- **Réseau** : 1 Gbps

### Configuration recommandée (production OLTP)
- **CPU** : 16+ cores (Intel Xeon ou AMD EPYC)
- **RAM** : 64-128 GB
- **Storage** : NVMe 1-2 TB (RAID optionnel)
- **Réseau** : 10 Gbps

### Configuration haute performance (production intensive)
- **CPU** : 32+ cores avec AVX2/AVX512
- **RAM** : 256-512 GB
- **Storage** : NVMe en RAID 10 ou NVMe-oF
- **Réseau** : 25+ Gbps

---

## 🛠️ Outils nécessaires

Pour suivre ce chapitre, vous aurez besoin de :

### Outils d'analyse
- **pt-query-digest** (Percona Toolkit) - Analyse slow query log
- **sysbench** - Benchmarking
- **mysqltuner** - Audit de configuration

### Monitoring
- **Performance Schema** - Intégré à MariaDB
- **sys schema** - Intégré à MariaDB 11.8
- **Prometheus + mysqld_exporter** (optionnel mais recommandé)
- **Grafana** (optionnel mais recommandé)

### Système
- **iostat**, **vmstat**, **top**, **htop** - Monitoring système
- **perf** - Profiling CPU (Linux)
- **bpftrace** / **ebpf** - Tracing avancé (optionnel)

Installation Percona Toolkit :
```bash
# Debian/Ubuntu
sudo apt-get install percona-toolkit

# RHEL/CentOS
sudo yum install percona-toolkit

# Vérification
pt-query-digest --version
```

---

## 📖 Comment utiliser ce chapitre

### Approche recommandée

1. **Lecture initiale** : Parcourir l'ensemble du chapitre pour comprendre le scope
2. **Établir une baseline** : Mesurer les performances actuelles de votre système
3. **Approche itérative** : Appliquer les optimisations section par section
4. **Validation continue** : Mesurer l'impact après chaque changement
5. **Documentation** : Tenir un journal des modifications et résultats

### Ordre suggéré selon votre situation

**Nouveau déploiement** :
1. Méthodologie (15.1)
2. Configuration mémoire (15.2)
3. Configuration I/O (15.4-15.6)
4. Monitoring (15.7-15.8)

**Système existant avec problèmes de performance** :
1. Analyse requêtes lentes (15.7)
2. Performance Schema diagnostic (15.8)
3. Méthodologie (15.1)
4. Optimisations ciblées selon findings

**Préparation migration vers 11.8** :
1. Nouveautés 11.8 (15.6, 15.14)
2. Configuration I/O SSD (15.4)
3. Gestion partitions avancée (15.10)

---

## ✅ Points clés à retenir

Avant de plonger dans les sections détaillées, gardez en tête ces principes fondamentaux :

- 🎯 **Mesurer avant d'optimiser** : Sans baseline, impossible de valider l'impact
- 🔍 **80/20 rule** : 20% des requêtes causent 80% des problèmes
- 💾 **Le buffer pool est crucial** : 70-80% RAM sur SSD, c'est le sweet spot
- 💿 **SSD change tout** : Les stratégies HDD ne s'appliquent plus
- 🆕 **11.8 optimise pour SSD nativement** : Laissez le cost optimizer travailler
- 📊 **Performance Schema est votre ami** : Source de vérité pour le diagnostic
- 🔄 **Optimisation continue** : Les workloads évoluent, vos optimisations aussi
- ⚠️ **Ne pas tout changer en même temps** : Changements incrémentaux et mesurés

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [📖 Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [📖 sys Schema](https://mariadb.com/kb/en/sys-schema/)
- [📖 Partitioning](https://mariadb.com/kb/en/partitioning-overview/)
- [📖 Optimization and Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)

### Documentation nouveautés 11.8

- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/changes-improvements-in-mariadb-118/)
- [📖 innodb_alter_copy_bulk](https://mariadb.com/kb/en/innodb-system-variables/#innodb_alter_copy_bulk)
- [📖 Cost-based Optimizer Improvements](https://mariadb.com/kb/en/optimizer/)

### Outils et ressources communautaires

- [Percona Toolkit Documentation](https://docs.percona.com/percona-toolkit/)
- [MySQL Performance Blog](https://www.percona.com/blog/category/mysql/)
- [High Performance MySQL (livre)](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)

---

## ➡️ Sections suivantes

Ce chapitre se décompose en 14 sections détaillées :

### 🎯 **Section 15.1** : [Méthodologie d'optimisation](./01-methodologie-optimisation.md)
*La fondation de toute démarche d'optimisation professionnelle - approche systématique, identification des bottlenecks, et cycles d'amélioration continue.*

### 💾 **Section 15.2** : [Configuration mémoire](./02-configuration-memoire.md)
*Dimensionnement expert du buffer pool InnoDB, instances multiples, et stratégies de préchargement.*

### 🗑️ **Section 15.3** : [Query Cache déprécié](./03-query-cache-deprecie.md)
*Pourquoi le Query Cache est contre-productif et quelles alternatives utiliser.*

### 💿 **Section 15.4** : [Configuration I/O et disques](./04-configuration-io-disques.md)
*Optimisation pour SSD/NVMe modernes - innodb_io_capacity, flush_method, et nouveautés 11.8.*

### ⚙️ **Section 15.5** : [Optimisation du moteur InnoDB](./05-optimisation-innodb.md)
*Tuning avancé : log files, checkpointing, purge threads, et adaptive flushing.*

### 🆕 **Section 15.6** : [innodb_alter_copy_bulk](./06-innodb-alter-copy-bulk.md)
*Nouveauté 11.8 : Construction d'index 40% plus rapide pour les DDL.*

### 🐌 **Section 15.7** : [Analyse des requêtes lentes](./07-analyse-requetes-lentes.md)
*Slow query log, configuration, et analyse avec pt-query-digest.*

### 📊 **Section 15.8** : [Performance Schema et sys schema](./08-performance-schema-sys.md)
*Diagnostic temps réel avec Performance Schema et requêtes sys schema.*

### 🗂️ **Section 15.9** : [Partitionnement de tables](./09-partitionnement-tables.md)
*RANGE, LIST, HASH partitioning - stratégies et partition pruning.*

### 🔄 **Section 15.10** : [Gestion avancée des partitions](./10-gestion-avancee-partitions.md)
*Conversion partition ↔ table, maintenance, et reorganisation (nouveauté 11.8).*

### 🌐 **Section 15.11** : [Sharding et distribution horizontale](./11-sharding-distribution.md)
*Au-delà du partitionnement : stratégies de sharding et architecture distribuée.*

### 🏎️ **Section 15.12** : [Benchmarking](./12-benchmarking.md)
*sysbench et mysqlslap pour mesurer et comparer les performances.*

### 🔧 **Section 15.13** : [Adaptive Hash Index](./13-adaptive-hash-index.md)
*Comprendre et optimiser l'AHI pour les workloads répétitifs.*

### 🆕 **Section 15.14** : [Cost-based optimizer SSD](./14-cost-based-optimizer-ssd.md)
*Nouveauté 11.8 : Optimiseur conscient des performances SSD/NVMe.*

---

## 🎓 Prêt à optimiser ?

Ce chapitre représente le sommet de l'expertise MariaDB en termes de performance. Les techniques présentées sont utilisées par les plus grandes organisations pour gérer des milliards de transactions par jour.

> 💡 **Conseil final** : L'optimisation est un marathon, pas un sprint. Prenez le temps de comprendre votre workload, mesurez rigoureusement, et appliquez les changements de manière méthodique.

**Bonne optimisation ! 🚀**

---

*Passons maintenant à la première section : [15.1 Méthodologie d'optimisation](/15-performance-tuning/01-methodologie-optimisation.md)*

⏭️ [Méthodologie d'optimisation](/15-performance-tuning/01-methodologie-optimisation.md)

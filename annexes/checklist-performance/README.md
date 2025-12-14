🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E. Checklist de Performance

> **Type** : Guide pratique d'audit et optimisation  
> **Objectif** : Identifier et résoudre les problèmes de performance  
> **Public** : DBA, Administrateurs système, DevOps, Développeurs

---

## 🎯 Objectif de cette annexe

Cette annexe fournit des **checklists méthodiques** pour auditer et optimiser les performances de MariaDB 11.8 LTS. Chaque checklist est structurée avec :

- ✅ **Points de vérification** : Que contrôler ?
- 🔍 **Méthodes de diagnostic** : Comment mesurer ?
- ⚠️ **Seuils critiques** : Quand agir ?
- 🔧 **Actions correctives** : Comment corriger ?
- 📊 **Métriques de succès** : Comment valider ?

---

## 📋 Structure de l'annexe

Cette annexe est organisée en **4 checklists complémentaires** couvrant tous les aspects de la performance :

| Checklist | Focus | Niveau | Durée audit |
|-----------|-------|--------|-------------|
| **[E.1 - Configuration](01-audit-configuration.md)** | Paramètres serveur my.cnf | Système | 30-60 min |
| **[E.2 - Indexation](02-audit-indexation.md)** | Stratégie et efficacité indexes | Schéma | 1-2 heures |
| **[E.3 - Requêtes](03-audit-requetes.md)** | Optimisation SQL queries | Application | 2-4 heures |
| **[E.4 - Schéma](04-audit-schema.md)** | Design base de données | Architecture | 1-3 heures |

### Parcours d'audit recommandé

```
1. Configuration (E.1) ──► Base solide serveur
   │
   ├─► 2. Indexation (E.2) ──► Accès données optimaux
   │    │
   │    └─► 3. Requêtes (E.3) ──► Code applicatif efficient
   │         │
   │         └─► 4. Schéma (E.4) ──► Architecture scalable
   │
   └─► Itération continue (monitoring, ajustements)
```

**Ordre logique :**
1. **Configuration** : Fondation (matériel, RAM, I/O)
2. **Indexation** : Structure d'accès (B-Tree, indexes composites)
3. **Requêtes** : Utilisation des index (SELECT, JOIN, WHERE)
4. **Schéma** : Design long terme (normalisation, partitionnement)

---

## 🔍 Méthodologie d'audit de performance

### Approche systématique

#### 1. Mesurer (Baseline)

**Établir métriques de référence avant optimisation**

```sql
-- Snapshot performance actuelle
SELECT 
    NOW() AS audit_date,
    @@version AS mariadb_version,
    (SELECT COUNT(*) FROM information_schema.processlist) AS active_connections,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Queries') AS total_queries,
    (SELECT variable_value FROM information_schema.global_status 
     WHERE variable_name = 'Slow_queries') AS slow_queries,
    ROUND((SELECT variable_value FROM information_schema.global_status 
           WHERE variable_name = 'Slow_queries') * 100.0 / 
          (SELECT variable_value FROM information_schema.global_status 
           WHERE variable_name = 'Queries'), 2) AS slow_query_pct
\G
```

**Métriques baseline essentielles :**
- Queries/seconde (QPS)
- Latence P50, P95, P99
- Buffer pool hit rate
- Connexions actives
- IOPS et latence disque
- CPU et RAM utilisation

#### 2. Identifier (Bottlenecks)

**Localiser les goulots d'étranglement**

```
Méthode descendante (Top-Down) :
─────────────────────────────────
Système OS          → CPU 100% ? RAM swapping ? Disk saturé ?
    ↓
Serveur MariaDB     → Buffer pool ? Locks ? Threads ?
    ↓
Base de données     → Tables volumineuses ? Fragmentation ?
    ↓
Requêtes            → Slow queries ? Full scans ? N+1 queries ?
```

**Signaux d'alerte :**
- 🔴 Latence P95 > 100ms (OLTP) ou > 10s (OLAP)
- 🔴 Buffer pool hit rate < 95%
- 🔴 Slow queries > 5% du total
- 🔴 Lock waits > 100/sec
- 🔴 Disk IOPS saturés (queue depth > 50)

#### 3. Analyser (Root Cause)

**Diagnostic approfondi avec outils**

| Outil | Usage | Commande |
|-------|-------|----------|
| **EXPLAIN** | Plan d'exécution query | `EXPLAIN SELECT ...` |
| **EXPLAIN ANALYZE** | Temps réel par étape | `EXPLAIN ANALYZE SELECT ...` |
| **SHOW PROCESSLIST** | Queries actives | `SHOW FULL PROCESSLIST` |
| **Performance Schema** | Statistiques détaillées | `SELECT * FROM performance_schema...` |
| **Slow Query Log** | Historique queries lentes | `pt-query-digest slow.log` |
| **SHOW ENGINE INNODB STATUS** | État interne InnoDB | `SHOW ENGINE INNODB STATUS\G` |

#### 4. Optimiser (Actions)

**Appliquer corrections par priorité**

```
Priorité 1 : Quick wins (impact max, effort min)
├─ Index manquant sur WHERE clause fréquente
├─ Requête N+1 → JOIN
└─ Buffer pool sous-dimensionné (+RAM)

Priorité 2 : Optimisations moyennes
├─ Refactoring queries complexes
├─ Partitionnement table volumineuse
└─ Ajustement paramètres my.cnf

Priorité 3 : Refonte architecture
├─ Dénormalisation sélective
├─ Séparation read/write (replicas)
└─ Sharding / Distribution
```

#### 5. Valider (Impact)

**Mesurer amélioration post-optimisation**

```sql
-- Comparer avant/après
-- Baseline : 2024-12-01, P95 = 250ms
-- Post-optimization : 2024-12-15, P95 = ?

SELECT 
    'After Optimization' AS phase,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_latency_sec,
    ROUND(MAX_TIMER_WAIT / 1000000000000, 3) AS max_latency_sec,
    COUNT_STAR AS executions
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%critical_query%'
  AND LAST_SEEN >= '2024-12-15';

-- Objectif : Réduction latence 50%+ = succès
```

#### 6. Monitorer (Continu)

**Surveillance post-optimisation**

- 📊 Dashboard Grafana : Métriques temps réel
- 🔔 Alertes Prometheus : Dégradation détectée
- 📈 Tendances : Croissance charge, volumétrie
- 🔄 Révision trimestrielle : Ré-audit checklist

---

## 💡 Principes généraux d'optimisation

### Loi de Pareto (80/20)

```
80% des problèmes de performance proviennent de 20% du code

Focus :
✅ Top 10 queries lentes (80% temps total)
✅ Tables volumineuses (80% données)
✅ Indexes manquants critiques (80% impact)

Éviter :
❌ Micro-optimisations négligeables
❌ Optimisation prématurée
❌ Complexité excessive pour 1% gain
```

### Ordre de priorité des optimisations

**Impact décroissant :**

1. **⭐⭐⭐⭐⭐ Indexes** : Gain 10-1000×
   - Index manquant → Full scan évité
   - Exemple : Query 30s → 50ms

2. **⭐⭐⭐⭐ Requêtes SQL** : Gain 2-100×
   - Refactoring logique (N+1 → JOIN)
   - Exemple : 100 queries → 1 query

3. **⭐⭐⭐ Configuration serveur** : Gain 1.2-3×
   - Buffer pool, sort buffers
   - Exemple : Hit rate 85% → 99%

4. **⭐⭐ Schéma base données** : Gain 1.5-5×
   - Partitionnement, dénormalisation
   - Exemple : Scan 1TB → 100GB

5. **⭐ Matériel** : Gain 1.2-2×
   - Upgrade CPU, RAM, SSD
   - Exemple : SSD SATA → NVMe

**Règle d'or :**
```
Optimiser le code avant le matériel
Une mauvaise requête sur un serveur surpuissant = toujours lente
```

### Méthode scientifique

```
1. Hypothèse    : "Buffer pool trop petit cause slow queries"
                  ↓
2. Mesure       : Buffer pool hit rate = 88% (< 99% objectif)
                  ↓
3. Changement   : innodb_buffer_pool_size 10G → 20G
                  ↓
4. Validation   : Hit rate = 98.5%, slow queries -60%
                  ↓
5. Documentation: Log changement, métriques avant/après
```

**Éviter :**
- ❌ Changer 10 paramètres simultanément
- ❌ Pas de mesure avant/après
- ❌ Pas de rollback plan

---

## 📝 Comment utiliser les checklists

### Format des checklists

Chaque section (E.1 à E.4) suit cette structure :

```markdown
## [Point à vérifier]

### 🔍 Diagnostic
[Requêtes SQL / Commandes pour mesurer]

### ⚠️ Seuils critiques
- 🟢 Optimal : [valeur]
- 🟡 Acceptable : [valeur]
- 🔴 Critique : [valeur]

### 🔧 Actions correctives
1. [Action prioritaire]
2. [Action secondaire]

### 📊 Validation
[Comment confirmer amélioration]

### 💡 Notes
[Contexte, exceptions, cas particuliers]
```

### Workflow d'utilisation

**Étape 1 : Audit initial (1ère fois)**

```bash
# 1. Créer dossier audit
mkdir -p audit-$(date +%Y%m%d)
cd audit-$(date +%Y%m%d)

# 2. Exécuter checklists séquentiellement
# E.1 Configuration
mariadb -e "SOURCE ../checklist-e1-config.sql" > e1-config-results.txt

# E.2 Indexation
mariadb mydb -e "SOURCE ../checklist-e2-indexes.sql" > e2-indexes-results.txt

# E.3 Requêtes
pt-query-digest /var/log/mysql/slow.log > e3-slow-queries.txt

# E.4 Schéma
mariadb mydb -e "SOURCE ../checklist-e4-schema.sql" > e4-schema-results.txt

# 3. Analyser résultats
cat e*-results.txt | grep "🔴\|CRITICAL" > issues-critical.txt
```

**Étape 2 : Priorisation**

```
Créer matrice impact/effort :

Impact ↑
│  🔴 A │ 🟡 B │     A = Haute priorité (faire maintenant)
│ ─────┼───── │     B = Moyenne priorité (planifier)
│  🟢 C │ ⚪ D │     C = Basse priorité (backlog)
└──────────── Effort →     D = Ignorer (ROI négatif)

Exemples :
- Index manquant critique : A (impact max, effort min)
- Refonte schéma complet : B (impact moyen, effort max)
- Optimiser query rare : D (impact min, effort variable)
```

**Étape 3 : Plan d'action**

```markdown
# Plan optimisation MariaDB - 2024-12

## Semaine 1 (Quick wins)
- [ ] Ajouter index users(email) → Query login -80% latence
- [ ] Augmenter buffer_pool 10G→20G → Hit rate 88%→99%
- [ ] Refactor N+1 dashboard → 50 queries → 2 queries

## Semaine 2-4 (Optimisations moyennes)
- [ ] Partitionner table orders par date (500M lignes)
- [ ] Dénormaliser table analytics (éviter JOIN 5 tables)
- [ ] Configurer read replica pour rapports

## Mois 2-3 (Architecture)
- [ ] Migration InnoDB → ColumnStore pour historique
- [ ] Setup MaxScale read/write split
- [ ] Implémentation cache applicatif Redis
```

**Étape 4 : Ré-audit périodique**

```
Fréquence recommandée :
─────────────────────
Production OLTP    : Mensuel (E.1, E.2, E.3)
Data Warehouse     : Trimestriel (E.1, E.2, E.4)
Dev/Staging        : Avant chaque release (E.2, E.3)
Post-incident      : Immédiat (toutes checklists)
```

---

## 🛠️ Outils recommandés

### Outils intégrés MariaDB

| Outil | Usage | Commande |
|-------|-------|----------|
| **EXPLAIN** | Plan d'exécution | `EXPLAIN SELECT ...` |
| **SHOW STATUS** | Métriques globales | `SHOW GLOBAL STATUS LIKE 'Innodb_%'` |
| **SHOW VARIABLES** | Configuration actuelle | `SHOW VARIABLES LIKE 'innodb%'` |
| **Performance Schema** | Profiling détaillé | `SELECT * FROM performance_schema...` |
| **INFORMATION_SCHEMA** | Métadonnées | `SELECT * FROM information_schema.tables` |

### Outils externes essentiels

#### Percona Toolkit

```bash
# Installation
sudo apt-get install percona-toolkit

# pt-query-digest : Analyse slow query log
pt-query-digest /var/log/mysql/slow.log

# pt-online-schema-change : ALTER TABLE sans downtime
pt-online-schema-change --alter "ADD INDEX idx_email (email)" \
  --execute D=mydb,t=users

# pt-table-checksum : Vérifier cohérence réplication
pt-table-checksum --databases=mydb

# pt-duplicate-key-checker : Détecter index redondants
pt-duplicate-key-checker --databases=mydb
```

#### MySQLTuner

```bash
# Installation
wget https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl
chmod +x mysqltuner.pl

# Exécution
./mysqltuner.pl --user root --pass password

# Rapport complet : Recommandations config
# Focus : Buffer pool, max_connections, indexes
```

#### Monitoring (Prometheus + Grafana)

```yaml
# docker-compose.yml
version: '3.8'
services:
  mysqld-exporter:
    image: prom/mysqld-exporter
    environment:
      DATA_SOURCE_NAME: "exporter:password@(mariadb:3306)/"
    ports:
      - "9104:9104"
  
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

---

## 📊 Métriques clés par cas d'usage

### OLTP (E-commerce, Banking, SaaS)

| Métrique | Seuil optimal | Seuil critique |
|----------|---------------|----------------|
| **Latence P95** | < 10ms | > 100ms |
| **QPS** | 1000-10000+ | - |
| **Buffer pool hit rate** | > 99% | < 95% |
| **Connexions actives** | 50-500 | > 80% max |
| **Slow queries** | < 1% | > 5% |
| **Lock waits/sec** | < 10 | > 100 |

### OLAP (Data Warehouse, Analytics)

| Métrique | Seuil optimal | Seuil critique |
|----------|---------------|----------------|
| **Query completion time** | < 5 min | > 30 min |
| **Buffer pool hit rate** | > 95% | < 90% |
| **Temp tables on disk** | < 10% | > 25% |
| **Full table scans/sec** | Variable (OK) | - |
| **Sort merge passes** | 0 | > 1000/sec |

### Mixed Workload

| Métrique | Seuil optimal | Seuil critique |
|----------|---------------|----------------|
| **OLTP latency P95** | < 50ms | > 200ms |
| **OLAP completion** | < 10 min | > 60 min |
| **Buffer pool hit rate** | > 97% | < 93% |
| **Lock contention** | < 50/sec | > 200/sec |

---

## 🎯 Scénarios d'audit typiques

### Scénario 1 : Latence dégradée progressive

```
Symptôme : P95 latency 50ms → 150ms sur 2 semaines

Checklist à exécuter :
1. E.3 Requêtes → Nouvelles queries lentes ?
2. E.2 Indexation → Table growth sans index adapté ?
3. E.1 Configuration → Buffer pool saturé (data growth) ?
4. E.4 Schéma → Fragmentation table ?

Root cause fréquent : Croissance données + buffer pool fixe
Action : Augmenter buffer_pool_size ou archiver vieilles données
```

### Scénario 2 : Pics de latence sporadiques

```
Symptôme : P95 stable 10ms, mais pics 500ms toutes les heures

Checklist à exécuter :
1. E.3 Requêtes → Batch job / reporting périodique ?
2. E.1 Configuration → Checkpoint flush InnoDB ?
3. E.4 Schéma → Lock contention table spécifique ?

Root cause fréquent : Analytics queries non optimisées
Action : Séparer read replica OLAP ou optimiser queries
```

### Scénario 3 : Nouvelle feature lente

```
Symptôme : Nouveau module applicatif déployé, latence ×10

Checklist à exécuter :
1. E.3 Requêtes → N+1 queries ? Queries sans index ?
2. E.2 Indexation → Missing indexes nouvelles tables ?
3. E.4 Schéma → Dénormalisation nécessaire ?

Root cause fréquent : Code applicatif non optimisé
Action : Code review + EXPLAIN queries + ajouter indexes
```

---

## ⚠️ Pièges courants à éviter

### 1. Sur-indexation

```sql
-- ❌ Mauvais : Index sur chaque colonne
CREATE INDEX idx_col1 ON table1(col1);
CREATE INDEX idx_col2 ON table1(col2);
CREATE INDEX idx_col3 ON table1(col3);
-- ... 20 indexes

-- Problèmes :
-- - Writes lents (maintenir 20 indexes)
-- - Espace disque ×3
-- - Optimizer confus (trop de choix)

-- ✅ Bon : Index composites ciblés
CREATE INDEX idx_search ON table1(col1, col2);  -- Query fréquente
CREATE INDEX idx_filter ON table1(col3);         -- WHERE clause
-- 2-5 indexes bien pensés > 20 indexes aléatoires
```

### 2. Optimisation prématurée

```python
# ❌ Optimiser avant mesurer
def get_users():
    # Cache Redis complexe, query pré-compilée, etc.
    # Alors que query prend 5ms (déjà rapide)
    pass

# ✅ Mesurer d'abord
def get_users():
    # Query simple
    # Profiling : 5ms → Pas besoin optimisation
    # Focus sur vraie bottleneck (autre module 500ms)
    pass
```

### 3. Ignorer root cause

```
Symptôme : Serveur RAM 100% utilisée

❌ Action réflexe : Acheter plus de RAM
   → Coût : 2000€
   → Problème persiste 1 mois plus tard

✅ Analyse root cause :
   1. Checklist E.1 Configuration
   2. Découverte : max_connections = 5000 (!!)
   3. Action : Réduire à 500 (adapté charge réelle)
   → RAM usage 100% → 40%
   → Coût : 0€
```

### 4. Changer plusieurs paramètres simultanément

```bash
# ❌ Changements multiples
SET GLOBAL innodb_buffer_pool_size = 30G;
SET GLOBAL max_connections = 1000;
SET GLOBAL innodb_io_capacity = 5000;
SET GLOBAL sort_buffer_size = 64M;
# ... 10 changements

# Performance améliore de 30%
# Mais impossible savoir quel paramètre a eu impact !

# ✅ Méthode scientifique
1. Changer innodb_buffer_pool_size = 30G
2. Mesurer : +15% performance ✓
3. Changer max_connections = 1000  
4. Mesurer : +2% performance ✓
5. Etc.
```

---

## ✅ Points clés à retenir

- 🔍 **Mesurer avant optimiser** : Baseline metrics indispensable
- 📊 **Focus 80/20** : Top 10 queries = 80% impact
- 🎯 **Ordre priorité** : Indexes > Queries > Config > Schéma > Matériel
- 🔬 **Méthode scientifique** : 1 changement → Mesure → Validation
- 📝 **Documentation** : Tracer tous changements (avant/après)
- 🔄 **Ré-audit périodique** : Mensuel (OLTP), Trimestriel (OLAP)
- 🛠️ **Outils essentiels** : EXPLAIN, pt-query-digest, Performance Schema
- ⚠️ **Éviter pièges** : Sur-indexation, optimisation prématurée, changements multiples

---

## 🗂️ Structure de cette annexe

```
E. Checklist de Performance/
│
├── README.md (ce fichier)
│   └── Méthodologie, principes, outils
│
├── E.1 - Audit de configuration
│   └── Paramètres serveur, RAM, I/O, logs
│       • 20+ points vérification
│       • Seuils optimaux OLTP/OLAP/Mixed
│       • Actions correctives prioritaires
│
├── E.2 - Audit d'indexation
│   └── Stratégie indexes, efficacité, redondance
│       • Missing indexes
│       • Unused indexes
│       • Duplicate indexes
│       • Index composites
│
├── E.3 - Audit de requêtes
│   └── Slow queries, N+1, full scans
│       • Top queries lentes
│       • Queries sans index
│       • Optimisations JOIN
│       • Sub-queries vs CTE
│
└── E.4 - Audit de schéma
    └── Design tables, normalisation, partitionnement
        • Volumétrie tables
        • Fragmentation
        • Types de données
        • Architecture données
```

---

## 📖 Utilisation recommandée

### Pour un audit complet (initial)

```
Temps : 1 journée (8 heures)
───────────────────────────

09h-10h : E.1 Configuration (baseline serveur)
10h-12h : E.2 Indexation (analyse indexes)
13h-15h : E.3 Requêtes (slow query log)
15h-17h : E.4 Schéma (design review)
17h-18h : Synthèse + Plan d'action
```

### Pour un audit rapide (mensuel)

```
Temps : 2 heures
───────────────

30 min : Top 3 checks E.1 (buffer pool, connexions, I/O)
30 min : Top 10 slow queries E.3
30 min : Missing indexes E.2
30 min : Volumétrie tables E.4
```

### Pour un audit ciblé (post-déploiement)

```
Temps : 1 heure
───────────────

Focus : Nouvelles features déployées
1. E.3 : Slow queries nouvelles tables
2. E.2 : Indexes nouvelles colonnes
3. Validation : Latence P95 stable
```

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Optimization and Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)
- [Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [Server System Variables](https://mariadb.com/kb/en/server-system-variables/)

### Outils
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit)
- [MySQLTuner](https://github.com/major/MySQLTuner-perl)
- [pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)

### Autres annexes utiles
- [Annexe D - Configurations de référence](/annexes/configuration-reference/README.md)
- [Annexe C - Requêtes SQL de référence](/annexes/requetes-sql-reference/README.md)
- [Section 15 - Performance et Tuning](/15-performance-tuning/README.md)

---

## ➡️ Sections suivantes

Consultez les checklists détaillées :

- **[E.1 - Audit de configuration](./01-audit-configuration.md)** → Paramètres serveur, buffer pool, I/O
- **[E.2 - Audit d'indexation](./02-audit-indexation.md)** → Stratégie indexes, missing, unused
- **[E.3 - Audit de requêtes](./03-audit-requetes.md)** → Slow queries, N+1, optimisations
- **[E.4 - Audit de schéma](./04-audit-schema.md)** → Design tables, normalisation, partitionnement

---

**MariaDB** : Version 11.8 LTS

⏭️ [Audit de configuration](/annexes/checklist-performance/01-audit-configuration.md)

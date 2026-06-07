🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 7 : Performance et Tuning (DBA Avancé)

> **Niveau** : Expert — DBA Senior, Architectes Performance, DevOps/SRE confirmés  
> **Durée estimée** : 2-3 jours  
> **Prérequis** : Maîtrise approfondie de MariaDB (index, transactions, moteurs de stockage), administration système avancée, compréhension des I/O et de la gestion mémoire, expérience de troubleshooting en production

---

## 🎯 L'art de l'optimisation systématique

La performance n'est pas une fonctionnalité — c'est une **propriété émergente** résultant de milliers de décisions techniques : choix d'index, configuration mémoire, stratégies I/O, conception de schéma, patterns d'accès aux données. Une base de données mal optimisée peut nécessiter 10x ou 100x plus de ressources matérielles qu'une base équivalente correctement tunée. L'optimisation de performance est donc autant une **question de coûts** qu'une question d'expérience utilisateur.

Dans le monde réel de la production, les problèmes de performance se manifestent de manière insidieuse : une requête qui fonctionnait parfaitement avec 100,000 lignes devient inutilisable à 10 millions. Un paramètre de configuration optimal pour OLTP crée des goulots d'étranglement pour OLAP. Une charge applicative qui évoluait graduellement atteint soudainement un point de rupture où le système s'effondre. **L'optimisation de performance requiert une approche méthodique, data-driven, et reproductible** — jamais basée sur l'intuition ou les "best practices" génériques appliquées aveuglément.

Cette partie est **dense et technique**. Vous allez plonger dans les entrailles de MariaDB pour comprendre comment l'optimiseur de requêtes prend ses décisions, comment InnoDB gère la mémoire et les I/O, comment identifier précisément les goulots d'étranglement, et comment appliquer des optimisations mesurables et validées. L'objectif n'est pas de collecter des "trucs et astuces", mais de développer une **méthodologie rigoureuse d'analyse et d'optimisation** applicable à n'importe quelle situation de performance.

Les DBA et architectes qui maîtrisent l'optimisation de performance sont rares et très recherchés. Cette compétence est **critique en production** : la différence entre un système qui peine sous la charge et un système qui scale élégamment réside presque toujours dans la qualité du tuning.

---

## 📚 Module unique : Performance et Tuning

### Module 15 : Performance et Tuning
**16 sections | Durée : 2-3 jours intensifs**

Ce module unique mais particulièrement dense couvre l'intégralité du spectre de l'optimisation MariaDB :

#### 🔬 Méthodologie d'optimisation
- **Approche scientifique** : Hypothèse → Mesure → Optimisation → Validation
- **Anti-patterns** : Les pièges de l'optimisation prématurée
- **Prioritisation** : Loi de Pareto (80% gains avec 20% efforts)
- **Documentation** : Tracer les changements et leurs impacts

#### 🧠 Configuration mémoire
- **InnoDB Buffer Pool** : Le paramètre le plus critique 🔄
  - Dimensionnement optimal : 50-75% RAM disponible (serveur dédié)
  - Redimensionnement à chaud par incréments fins jusqu'à `innodb_buffer_pool_size_max` (depuis 11.8.2) — l'ancien découpage en *instances* (`innodb_buffer_pool_instances`) a été retiré en 10.6 (instance unique)
  - Métriques de hit ratio et pages dirty
- **Key buffer (MyISAM)** : Pour tables legacy
- **Thread buffers** : sort_buffer_size, join_buffer_size, read_buffer_size
- **Query cache deprecation** : Pourquoi ne plus l'utiliser

#### 💾 Configuration I/O et disques
- **innodb_io_capacity** : Matching hardware capabilities 🔄
  - HDD : 200 IOPS
  - SSD SATA : 5,000-10,000 IOPS
  - NVMe : 50,000+ IOPS
- **innodb_flush_method** : O_DIRECT vs fsync trade-offs
- **Optimisations SSD modernes** : TRIM, over-provisioning
- **Cost-based optimizer amélioré (11.0)** : Prise en compte SSD dans l'estimation des coûts (modèle réécrit en 11.0, toujours en vigueur en 12.3)

#### ⚡ Optimisation du moteur InnoDB
- **Redo log sizing** : innodb_log_file_size impact sur checkpoints
- **Undo log management** : Purge threads et tablespaces
- **Doublewrite buffer** : Sécurité vs performance
- **Adaptive Hash Index** : Quand activer/désactiver (désactivé par défaut depuis 10.5)
- **innodb_alter_copy_bulk** : Construction d'index ultra-rapide par chargement en masse (introduit dans les séries 10.11.9 / 11.4.3 / 11.5.2 / 11.6.1…, activé par défaut en 12.3)

#### 🐌 Analyse des requêtes lentes
- **Slow query log** : Configuration et activation
  - long_query_time threshold
  - log_queries_not_using_indexes
- **pt-query-digest (Percona Toolkit)** : Analyse statistique
  - Top queries by execution time
  - Top queries by count
  - Query patterns et fingerprints
- **Optimisation systématique** : De l'identification à la correction

#### 📊 Performance Schema et sys schema
- **Performance Schema** : Instrumentation complète 🔄
  - Désactivé par défaut en MariaDB ; surcoût **minime** (perceptible surtout à très fort débit), maîtrisé via les tables `setup_*`
  - Tables critiques : events_statements, table_io_waits
  - Memory instrumentation
- **sys schema** : Vues simplifiées pour diagnostic
  - statements_with_full_table_scans
  - schema_unused_indexes
  - host_summary, user_summary
- **Requêtes de diagnostic essentielles**

#### 🗂️ Partitionnement de tables
- **Stratégies** : RANGE, LIST, HASH, KEY
- **Partition pruning** : Élimination automatique de partitions
- **Maintenance** : REORGANIZE, EXCHANGE, TRUNCATE PARTITION
- **Gestion avancée** : conversion atomique partition↔table — `CONVERT PARTITION TO TABLE` (depuis 10.7) et `CONVERT TABLE TO PARTITION` (depuis 11.4)
- **Anti-patterns** : Quand le partitionnement dégrade les performances

#### 🌐 Sharding et distribution horizontale
- **Sharding applicatif** : Patterns et stratégies
- **Spider engine** : Sharding natif MariaDB
- **Considérations** : Transactions distribuées, jointures inter-shards

#### 📈 Benchmarking professionnel
- **sysbench** : Benchmark standard OLTP/OLAP 🔄
  - oltp_read_write, oltp_read_only
  - Interprétation des résultats
- **mysqlslap** : Load testing natif
- **Méthodologie** : Répétabilité et représentativité
- **Comparaisons avant/après** : Validation des optimisations

#### 🔧 Optimisations avancées InnoDB
- **Adaptive Hash Index** : Conditions d'activation
- **Buffer Pool optimizations** : Warmup, dump/load, redimensionnement à chaud
- **Compression** : ROW_FORMAT=COMPRESSED trade-offs
- ⚠️ **Change buffer** : mécanisme **retiré en 11.0** (MDEV-29694) — ne plus chercher à le régler

#### 🎯 Optimiseur de requêtes
- **Statistics maintenance** : ANALYZE TABLE
- 🆕 **Optimizer Hints nouvelle génération (12.x)** : commentaires `/*+ ... */` par requête — `QB_NAME`, hints de table/index, ordre de jointure (`JOIN_ORDER`/`JOIN_PREFIX`…), sous-requêtes (`SEMIJOIN`/`SUBQUERY`…), `MAX_EXECUTION_TIME` (§15.15)
- **Index hints hérités** : `FORCE INDEX`, `USE INDEX`, `STRAIGHT_JOIN`
- **Optimizer switches** : Contrôle fin du comportement
- **Cost model amélioré (11.0)** : estimations précises pour SSD (modèle réécrit en 11.0)

#### 🔢 Vector : optimisations 12.3
- 🆕 **Distance par extrapolation + calcul dans le moteur de stockage (§15.16)** : recherche HNSW accélérée (≈ +10-30 % de QPS à rappel égal), transparente pour l'application

#### 🔍 Diagnostic avancé
- **SHOW ENGINE INNODB STATUS** : État interne complet
- **Information Schema queries** : Métadonnées système
- **Process list analysis** : Détection de blocages
- **Lock monitoring** : SHOW ENGINE INNODB MUTEX

#### 📐 Patterns d'optimisation éprouvés
- **Covering indexes** : Éliminer l'accès table
- **Index merge optimization**
- **Subquery optimizations** : Materialization vs semi-join
- **JOIN optimizations** : Order et algorithmes

💡 **Impact business** : Les techniques de cette partie peuvent réduire les coûts d'infrastructure de 50-80% en permettant de supporter 5-10x plus de charge sur le même matériel. Pour un système avec $100k/an de coûts serveurs, cela représente $50-80k d'économies annuelles.

---

## 🆕 Innovations récentes pour la performance (séries 11.x et 12.x)

> Ces améliorations s'échelonnent sur plusieurs versions et sont **toutes présentes en 12.3 LTS**. Les deux nouveautés propres à la 12.3 — les **Optimizer Hints** consolidés (§15.15) et les **optimisations vectorielles** (§15.16) — sont détaillées dans le chapitre.

### innodb_alter_copy_bulk : construction d'index par chargement en masse

#### Le problème historique

```sql
-- Scénario : reconstruction de table (ALGORITHM=COPY), 500M lignes
ALTER TABLE large_table MODIFY COLUMN ... ;   -- modification imposant une copie

-- Ancien comportement (sans innodb_alter_copy_bulk) :
-- Méthode : insertion ligne à ligne dans le B-tree
-- I/O : random writes intensifs
-- Index : moins compacts, plus fragmentés
```

**Goulots d'étranglement** :
- Insertions dans B-tree nécessitent réorganisations continues
- Random I/O pattern (non-optimisé pour SSD)
- Buffer pool thrashing avec grandes tables
- Checkpointing intensif ralentit progressivement

#### La solution : innodb_alter_copy_bulk

Introduit dans les séries **10.11.9 / 11.1.6 / 11.2.5 / 11.4.3 / 11.5.2 / 11.6.1** (et **activé par défaut** en 12.3), `innodb_alter_copy_bulk` applique aux reconstructions par `ALGORITHM=COPY` le chargement en masse qu'InnoDB utilisait déjà pour remplir une table vide :

```sql
-- Activé par défaut en 12.3 ; on peut le désactiver pour diagnostic
SHOW VARIABLES LIKE 'innodb_alter_copy_bulk';   -- ON par défaut

-- Avec le chemin bulk :
-- Méthode : chargement en masse + tri externe, index construits pré-triés page par page
-- I/O : écritures séquentielles, journalisation undo/redo fortement réduite
-- Index : plus compacts et moins fragmentés
```

**Algorithme optimisé** :
1. **Phase 1** : Scan de la table, extraction des colonnes indexées
2. **Phase 2** : Tri externe sur disque (algorithme merge-sort)
3. **Phase 3** : Construction bottom-up du B-tree (séquentielle)
4. **Phase 4** : Intégration dans tablespace (minimal I/O)

**Gains mesurés** :
- ⚡ **5-10x plus rapide** selon taille table et hardware
- 💾 **70% moins d'I/O** (sequential vs random)
- 🧠 **50% moins d'utilisation Buffer Pool**
- 🔧 **Maintenance sans downtime** : Table reste accessible en lecture

**Configuration optimale** :
```ini
[mysqld]
innodb_alter_copy_bulk = ON  
innodb_sort_buffer_size = 64M  # Augmenter pour grandes tables  
tmpdir = /fast-ssd/tmp         # SSD pour fichiers temporaires  
```

---

### Cost-based optimizer : reconnaissance des SSD (depuis 11.0)

#### Évolution du modèle de coûts

Le modèle de coût a été **profondément réécrit en MariaDB 11.0** (modèle purement basé sur les coûts, calibré pour le SSD), et c'est ce modèle qui est en vigueur en 12.3 :

```sql
-- Ancien modèle (pré-11.0) :
-- Coûts « de base » hérités, peu réalistes, supposant un disque mécanique (HDD)
-- → tendance à surévaluer le coût des accès aléatoires (par index)

-- Nouveau modèle (11.0+) :
-- Coûts élémentaires calibrés en microsecondes, par défaut pour un SSD moderne
-- optimizer_disk_read_cost ≈ 10,24 µs ; optimizer_disk_read_ratio ≈ 0,02 (98 % de cache)
-- → accès aléatoires correctement valorisés comme peu coûteux (cf. §15.14 pour les valeurs exactes)
```

**Impact sur les plans d'exécution** :

```sql
-- Requête : Trouver 10 utilisateurs par ville
SELECT * FROM users WHERE city = 'Paris' LIMIT 10;

-- Modèle hérité (pré-11.0, orienté HDD) :
-- Plan : Full table scan (séquentiel privilégié)
-- Rows scanned : 10,000,000
-- Time : 2.5 seconds

-- Modèle SSD (11.0+) :
-- Plan : Index scan sur idx_city
-- Rows scanned : 45,000 (users à Paris)
-- Time : 0.08 seconds (30x plus rapide)
```

**Requêtes bénéficiaires** :
- ✅ Lookups avec faible sélectivité (trouver 0.1-5% des lignes)
- ✅ Requêtes avec ORDER BY et LIMIT (top-N queries)
- ✅ Jointures avec petites tables de référence
- ✅ Subqueries avec EXISTS/IN sur index

**Validation** :
```sql
-- Comparer les plans d'exécution
EXPLAIN FORMAT=JSON  
SELECT * FROM orders WHERE customer_id = 12345;  

-- Vérifier le coût estimé : en 11.0+ (donc 12.3), les coûts sont calibrés pour SSD.
-- Les coûts par moteur s'inspectent dans information_schema.optimizer_costs (§15.14)
```

---

### Gestion avancée des partitions : conversion atomique partition↔table

#### Le vrai apport : l'atomicité

Auparavant, convertir une partition en table autonome (ou l'inverse) imposait une recette en plusieurs étapes autour d'`EXCHANGE PARTITION` — **non atomique** : un crash en cours de route pouvait laisser la table dans un état incohérent. MariaDB fournit désormais deux commandes dédiées, chacune **atomique** (l'opération aboutit entièrement ou est intégralement annulée, même si le serveur s'arrête en plein milieu) :

```sql
-- Extraire une partition en table autonome — depuis MariaDB 10.7
ALTER TABLE ventes CONVERT PARTITION p2023 TO TABLE archives_2023;

-- Rattacher une table autonome comme nouvelle partition — depuis MariaDB 11.4
ALTER TABLE ventes
  CONVERT TABLE import_2026 TO PARTITION p2026 VALUES LESS THAN ('2027-01-01');
```

**Cas d'usage** :

1. **Partition → Table** : archivage ou consolidation
```sql
-- Détacher une période ancienne en table autonome (opération de métadonnées, rapide)
ALTER TABLE ventes CONVERT PARTITION p2023 TO TABLE archives_2023;
```

2. **Table → Partition** : intégration d'un lot préparé à part
```sql
-- La table import_2026 a été remplie et indexée hors production, puis rattachée
ALTER TABLE ventes
  CONVERT TABLE import_2026 TO PARTITION p2026 VALUES LESS THAN ('2027-01-01');
-- WITH VALIDATION par défaut (chaque ligne contrôlée) ; WITHOUT VALIDATION pour aller plus vite
```

3. **Réorganisation de partitions** : fusion/scission (recopie physique)
```sql
-- Fusionner d'anciennes partitions (REORGANIZE recopie les lignes concernées)
ALTER TABLE sales
  REORGANIZE PARTITION p2020,p2021,p2022
  INTO (PARTITION p_old VALUES LESS THAN (2023));
```

> ⚠️ **À savoir** : en MariaDB 12.3, les opérations **spécifiques aux partitions** (`ADD`/`DROP`/`REBUILD`/`REORGANIZE`/`CONVERT`… `PARTITION`) **ne prennent pas en charge la clause `ALGORITHM`/`LOCK`** — `ALTER ONLINE TABLE … REBUILD PARTITION` (ou `ALGORITHM=INPLACE`) échoue avec l'erreur **`1846`**. Ces opérations s'exécutent en `LOCK=DEFAULT` ; celles qui relèvent des métadonnées (`DROP`/`TRUNCATE`, `EXCHANGE`, `CONVERT`) bloquent peu, tandis que `REORGANIZE`/`REBUILD` recopient des données. Le mode en ligne reste réservé aux modifications **non spécifiques aux partitions** (ajout de colonne/index). Voir §15.10.

**Bénéfices** :
- ✅ Conversions **atomiques** et sûres face aux pannes (vs l'ancienne recette `EXCHANGE` non atomique)
- ✅ Archivage et chargement en masse simplifiés (préparer/valider une table à part, puis la rattacher)
- ✅ Intention exprimée directement, en une seule instruction

---

## ✅ Compétences acquises

À la fin de cette septième partie, vous serez capable de :

### Diagnostic de performance
- ✅ **Identifier** les goulots d'étranglement (CPU, I/O, mémoire, réseau)
- ✅ **Analyser** les requêtes lentes avec pt-query-digest
- ✅ **Interpréter** SHOW ENGINE INNODB STATUS
- ✅ **Utiliser** Performance Schema et sys schema efficacement
- ✅ **Diagnostiquer** les problèmes de verrous et deadlocks
- ✅ **Détecter** les index manquants ou sous-utilisés

### Optimisation systématique
- ✅ **Appliquer** une méthodologie rigoureuse (mesure → optimisation → validation)
- ✅ **Dimensionner** le Buffer Pool et autres paramètres mémoire
- ✅ **Configurer** les paramètres I/O pour SSD/NVMe
- ✅ **Optimiser** les requêtes avec EXPLAIN et index appropriés
- ✅ **Implémenter** des partitionnements stratégiques
- ✅ **Benchmark** avant/après chaque changement

### Tuning avancé InnoDB
- ✅ **Configurer** les logs (redo, undo) pour charge de travail
- ✅ **Optimiser** la gestion du Buffer Pool (warmup dump/load, redimensionnement à chaud)
- ✅ **Ajuster** les threads I/O et purge
- ✅ **Utiliser** innodb_alter_copy_bulk pour maintenance efficace
- ✅ **Comprendre** les trade-offs durabilité vs performance

### Expertise de production
- ✅ **Créer** des dashboards de monitoring pertinents
- ✅ **Définir** des SLI/SLO de performance
- ✅ **Automatiser** la détection de régressions
- ✅ **Documenter** les optimisations et leurs impacts
- ✅ **Planifier** la capacité (capacity planning)

### Outils professionnels
- ✅ **Maîtriser** Percona Toolkit (pt-query-digest, pt-online-schema-change)
- ✅ **Utiliser** sysbench pour benchmarks reproductibles
- ✅ **Exploiter** Performance Schema pour instrumentation
- ✅ **Analyser** avec sys schema pour diagnostics rapides
- ✅ **Monitorer** avec Prometheus/Grafana (métriques MariaDB)

---

## 🎓 Parcours recommandés

Cette partie est **essentielle** pour les rôles d'optimisation et d'architecture performance.

| Parcours | Importance | Justification |
|----------|------------|---------------|
| 🔐 **DBA Senior** | ⭐⭐⭐ CRITIQUE | L'optimisation de performance est une responsabilité fondamentale du DBA senior. Expertise attendue pour résoudre 90% des problèmes de production. |
| 🏗️ **Architecte Performance** | ⭐⭐⭐ CRITIQUE | Conception d'architectures performantes nécessite compréhension profonde des capacités et limites de MariaDB. Décisions structurantes. |
| ⚙️ **DevOps/SRE Senior** | ⭐⭐⭐ ESSENTIEL | Capacity planning, monitoring, et troubleshooting performance sont au cœur du métier SRE. Nécessaire pour atteindre SLO. |
| 🔧 **Tech Lead/Développeur Senior** | ⭐⭐ TRÈS UTILE | Comprendre les implications performance des choix de code (requêtes, index, patterns) améliore drastiquement la qualité applicative. |

### Pourquoi cette partie est stratégique ?

#### Pour les DBA Senior
La performance est **le critère #1** sur lequel les DBA sont évalués en production :
- 70% des escalations concernent la performance
- Capacité à résoudre rapidement = valeur ajoutée immédiate
- Expertise différenciante sur le marché du travail

#### Pour les Architectes
Décisions d'architecture ont impact long terme :
- Sharding vs replication vs Galera : implications performance
- Choice of storage engine : 10-100x différence selon workload
- Schema design : Refactoriser après est coûteux

#### Pour les DevOps/SRE
Performance = Coûts d'infrastructure :
- Optimisation 2x = Division des coûts serveurs par 2
- Capacity planning précis évite sur/sous-provisioning
- Alerting performance proactif prévient incidents

---

## 🔬 Méthodologie d'optimisation : Approche scientifique

### Les 4 phases de l'optimisation

#### Phase 1 : OBSERVER 🔍

**Principe** : Comprendre avant d'agir. Ne jamais optimiser à l'aveugle.

```sql
-- Identifier les requêtes problématiques
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    AVG_TIMER_WAIT/1000000000 AS avg_ms,
    SUM_TIMER_WAIT/1000000000 AS total_ms,
    SUM_ROWS_EXAMINED AS rows_examined
FROM performance_schema.events_statements_summary_by_digest  
WHERE SCHEMA_NAME = 'production_db'  
ORDER BY SUM_TIMER_WAIT DESC  
LIMIT 10;  

-- Top 10 requêtes par temps cumulé
-- → Priorisation basée sur impact réel
```

**Outils d'observation** :
- 📊 Performance Schema (instrumentation complète)
- 📈 sys schema (vues simplifiées)
- 🐌 Slow query log (requêtes au-dessus d'un seuil)
- 📉 Grafana dashboards (métriques temps réel)
- 🔧 pt-query-digest (analyse statistique)

**Métriques clés à observer** :
- Throughput : Queries/second
- Latency : p50, p95, p99 response time
- Saturation : Buffer pool hit ratio, I/O wait
- Errors : Connection failures, deadlocks

---

#### Phase 2 : MESURER 📏

**Principe** : Établir une baseline avant toute optimisation.

```bash
# Benchmark initial avec sysbench
sysbench oltp_read_write \
  --table-size=1000000 \
  --threads=32 \
  --time=300 \
  --mysql-db=testdb \
  run > baseline_results.txt

# Extraire métriques clés
cat baseline_results.txt | grep -E "(transactions:|queries:|latency:)"
# transactions: 45234 (150.78 per sec.)
# queries: 904680 (3015.60 per sec.)
# 95th percentile: 42.61ms
```

**Mesures essentielles** :
1. **Performance actuelle** : Baseline mesurable et répétable
2. **Charge de travail** : Read/write ratio, query patterns
3. **Ressources utilisées** : CPU, RAM, IOPS, network
4. **Goulots identifiés** : Où est le blocage réel ?

**Documentation** :
```markdown
## Baseline Performance - 2025-12-15
- Throughput : 150 TPS
- Latency p95 : 42.61ms
- Buffer Pool Hit Ratio : 87.5%
- CPU Utilization : 65%
- Disk IOPS : 8,500 (85% capacity)

**Goulot identifié** : I/O disque saturé  
**Hypothèse** : Buffer Pool sous-dimensionné  
```

---

#### Phase 3 : OPTIMISER ⚡

**Principe** : Changement unique et isolé. Tester une modification à la fois.

```ini
# Avant optimisation (baseline documentée)
[mysqld]
innodb_buffer_pool_size = 4G        # 87.5% hit ratio  
innodb_io_capacity = 200            # HDD default  

# Après optimisation (un seul levier à la fois — ici le Buffer Pool)
[mysqld]
innodb_buffer_pool_size = 12G       # Augmentation à ~75% RAM  
innodb_io_capacity = 10000          # (réglé séparément, à sa propre itération)  

# Note : depuis 10.6, le Buffer Pool est en instance unique
# (innodb_buffer_pool_instances a été retiré) ; rien à régler de ce côté.

# Restart MariaDB
systemctl restart mariadb
```

**Principes d'optimisation** :
1. ✅ **Une modification à la fois** : Identifier l'impact exact
2. ✅ **Changements réversibles** : Possibilité de rollback immédiat
3. ✅ **Documentation** : Noter chaque changement avec timestamp
4. ✅ **Version control** : Versionner my.cnf dans Git
5. ✅ **Testing en staging** : Valider avant production

**Optimisations courantes par impact** :

| Optimisation | Impact potentiel | Complexité | Risque |
|--------------|------------------|------------|--------|
| Ajouter index manquant | 100-1000x | Faible | Faible |
| Augmenter Buffer Pool | 2-5x | Faible | Moyen |
| Réécrire requête inefficace | 10-100x | Moyen | Faible |
| Partitionnement table | 5-20x | Élevé | Moyen |
| Changer moteur stockage | 2-100x | Élevé | Élevé |
| Sharding horizontal | 5-50x | Très élevé | Élevé |

**Stratégie** : Commencer par optimisations à fort impact, faible complexité, faible risque.

---

#### Phase 4 : VALIDER ✅

**Principe** : Mesurer l'impact réel. Valider que l'optimisation fonctionne.

```bash
# Re-benchmark après optimisation
sysbench oltp_read_write \
  --table-size=1000000 \
  --threads=32 \
  --time=300 \
  --mysql-db=testdb \
  run > optimized_results.txt

# Comparaison baseline vs optimized
echo "=== BEFORE ===" && cat baseline_results.txt | grep "transactions:"  
echo "=== AFTER  ===" && cat optimized_results.txt | grep "transactions:"  

# Output :
# === BEFORE ===
# transactions: 45234 (150.78 per sec.)
# === AFTER  ===
# transactions: 135702 (452.34 per sec.)
# 
# → Amélioration : 3x throughput (200% gain)
```

**Validation multi-dimensionnelle** :
```sql
-- 1. Performance : Latence améliorée ?
SELECT AVG_TIMER_WAIT/1000000000 AS avg_ms_after  
FROM performance_schema.events_statements_summary_by_digest  
WHERE DIGEST_TEXT LIKE '%critical_query%';  
-- avg_ms_after : 15.2ms (vs 42.6ms avant = 65% réduction)

-- 2. Ressources : Utilisation optimale ?
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
-- Read requests : 95.2% from buffer (vs 87.5% avant)

-- 3. Effets secondaires : Problèmes introduits ?
SHOW GLOBAL STATUS LIKE '%lock%';
-- Lock wait time : Stable (pas de régression)
```

**Documentation de la validation** :
```markdown
## Optimization Results - 2025-12-15

### Changes Applied
- Increased Buffer Pool : 4G → 12G
- Increased instances : 4 → 12
- Updated I/O capacity : 200 → 10000 IOPS

### Results
- Throughput : 150 TPS → 452 TPS (3x improvement) ✅
- Latency p95 : 42.61ms → 15.2ms (65% reduction) ✅
- Buffer Pool Hit : 87.5% → 95.2% (target achieved) ✅
- CPU Utilization : 65% → 58% (reduced) ✅
- Disk IOPS : 8,500 → 3,200 (62% reduction) ✅

### Conclusion
**SUCCESS** - All metrics improved, no regressions.
Commit to production.

### Rollback Plan
If issues arise: 
1. Revert my.cnf to baseline_2025-12-15.cnf
2. Restart MariaDB
3. Validate with baseline benchmark
```

---

### Anti-patterns d'optimisation

#### ❌ Anti-pattern 1 : Optimisation prématurée
```python
# Mauvais : Optimiser sans mesurer
# "Je vais ajouter 10 index sur toutes les colonnes au cas où"
# → Ralentit les INSERT/UPDATE/DELETE
# → Gaspillage d'espace disque
# → Complexité maintenance

# Bon : Mesurer d'abord, optimiser ensuite
# 1. Slow query log active
# 2. Identifier requêtes réellement lentes
# 3. Ajouter index ciblés
```

#### ❌ Anti-pattern 2 : Suivre aveuglément les "best practices"
```ini
# Mauvais : Copier-coller configuration internet
# "Quelqu'un sur Stack Overflow dit que ça marche"
innodb_buffer_pool_size = 256G  # Si vous n'avez que 64G RAM...

# Bon : Adapter à votre environnement
# 1. Mesurer RAM disponible
# 2. Analyser workload (OLTP vs OLAP)
# 3. Configurer selon besoins réels
```

#### ❌ Anti-pattern 3 : Optimiser sans baseline
```bash
# Mauvais : Appliquer changements sans mesure
$ vim my.cnf  # Modifier plusieurs paramètres
$ systemctl restart mariadb
$ # "Ça semble plus rapide ?"

# Bon : Baseline → Change → Validate
$ sysbench run > before.txt
$ vim my.cnf  # UN changement
$ systemctl restart mariadb
$ sysbench run > after.txt
$ diff before.txt after.txt  # Quantifier l'impact
```

---

## 📊 Exemples de gains de performance concrets

### Cas réel 1 : Index manquant sur table utilisateurs

**Contexte** : Application SaaS B2B, 5M utilisateurs, recherche par email

**Problème identifié** :
```sql
-- Requête problématique (500ms)
SELECT * FROM users WHERE email = 'john.doe@company.com';

-- EXPLAIN montre full table scan
EXPLAIN SELECT * FROM users WHERE email = 'john.doe@company.com'\G
*************************** 1. row ***************************
           type: ALL
    possible_keys: NULL
          key: NULL
      rows: 5000000
         Extra: Using where

-- Scanne 5M lignes pour trouver 1 utilisateur
```

**Optimisation** :
```sql
-- Ajout index sur colonne email
CREATE INDEX idx_email ON users(email);
-- Query OK, 5000000 rows affected (2 min 34 sec)
```

**Résultat** :
```sql
-- Nouvelle performance (2ms)
EXPLAIN SELECT * FROM users WHERE email = 'john.doe@company.com'\G
*************************** 1. row ***************************
           type: ref
    possible_keys: idx_email
          key: idx_email
      rows: 1
         Extra: Using index condition

-- Scanne 1 ligne seulement
```

**Gains** :
- ⚡ Latence : 500ms → 2ms (250x plus rapide)
- 💾 Rows examined : 5M → 1 (99.99998% réduction)
- 💰 CPU usage : -95% sur cette requête
- 👥 User satisfaction : Login instantané vs 0.5s lag

**ROI** : 2min34s d'investissement pour gains permanents sur requête exécutée 10,000 fois/jour.

---

### Cas réel 2 : Buffer Pool sous-dimensionné

**Contexte** : E-commerce, base 80GB, serveur 128GB RAM

**Problème identifié** :
```sql
-- Buffer Pool hit ratio faible
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
+---------------------------------------+-------------+
| Variable_name                         | Value       |
+---------------------------------------+-------------+
| Innodb_buffer_pool_read_requests      | 2847293847  |
| Innodb_buffer_pool_reads              | 347482934   | ← Disk reads
+---------------------------------------+-------------+

-- Hit ratio : (2847293847 - 347482934) / 2847293847 = 87.8%
-- → 12.2% des lectures vont sur disque (lent)
```

**Configuration initiale** :
```ini
[mysqld]
innodb_buffer_pool_size = 16G  # Seulement 12.5% RAM !
```

**Optimisation** :
```ini
[mysqld]
innodb_buffer_pool_size = 96G  # 75% des 128GB RAM
# (instance unique depuis 10.6 ; innodb_buffer_pool_instances retiré)
```

**Résultat après 24h** :
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
+---------------------------------------+-------------+
| Variable_name                         | Value       |
+---------------------------------------+-------------+
| Innodb_buffer_pool_read_requests      | 3124857234  |
| Innodb_buffer_pool_reads              | 15624287    | ← Drastiquement réduit
+---------------------------------------+-------------+

-- Nouveau hit ratio : 99.5%
-- → 0.5% seulement va sur disque
```

**Gains mesurés** :
- ⚡ Latency p95 : 85ms → 12ms (7x amélioration)
- 📈 Throughput : 450 TPS → 1,200 TPS (2.7x)
- 💾 Disk IOPS : 12,000 → 800 (93% réduction)
- 💰 Infrastructure : Peut servir 3x plus de clients sur même hardware

**Coût** : $0 (RAM déjà disponible, juste configuration)

---

### Cas réel 3 : Requête N+1 problème (ORM)

**Contexte** : API REST, listing de 100 commandes avec détails clients

**Problème identifié** :
```python
# Code ORM (Django/SQLAlchemy pattern)
orders = Order.objects.all()[:100]  # 1 requête  
for order in orders:  
    print(order.customer.name)      # 100 requêtes supplémentaires !

# Total : 101 requêtes SQL pour afficher 100 commandes
# Latency totale : 101 × 5ms = 505ms
```

**Slow query log** :
```
# Time: 2025-12-15T10:23:45.123456Z
# Query_time: 0.505  Lock_time: 0.001  Rows_sent: 100
SELECT * FROM orders LIMIT 100;  
SELECT * FROM customers WHERE id = 1;  
SELECT * FROM customers WHERE id = 2;  
... (98 requêtes similaires)
SELECT * FROM customers WHERE id = 100;
```

**Optimisation** :
```python
# Solution 1 : select_related (JOIN)
orders = Order.objects.select_related('customer').all()[:100]
# → 1 seule requête avec JOIN
# SELECT * FROM orders 
# LEFT JOIN customers ON orders.customer_id = customers.id 
# LIMIT 100;

# Solution 2 : prefetch_related (IN query)
orders = Order.objects.prefetch_related('customer').all()[:100]
# → 2 requêtes seulement
# SELECT * FROM orders LIMIT 100;
# SELECT * FROM customers WHERE id IN (1,2,3,...,100);
```

**Résultat** :
```
# Solution 1 (JOIN) :
# Queries : 101 → 1 (99% réduction)
# Latency : 505ms → 8ms (63x plus rapide)

# Solution 2 (IN) :
# Queries : 101 → 2 (98% réduction)
# Latency : 505ms → 12ms (42x plus rapide)
```

**Gains** :
- ⚡ API response time : 505ms → 8ms
- 📡 Network overhead : -99% (1 round-trip vs 101)
- 🔌 Connection pool : -99% utilization
- 💰 Database load : Peut servir 50x plus de requêtes API

**Leçon** : Toujours profiler les requêtes ORM en développement. L'ORM est pratique mais cache souvent des anti-patterns de performance.

---

### Cas réel 4 : Partitionnement pour archivage

**Contexte** : Système de logs applicatifs, 2TB de données, 4 ans d'historique

**Problème identifié** :
```sql
-- Requête courante : Logs des 7 derniers jours
SELECT * FROM application_logs  
WHERE log_date >= CURDATE() - INTERVAL 7 DAY  
ORDER BY log_date DESC  
LIMIT 1000;  

-- Sans partitionnement :
-- Temps : 15 secondes
-- Rows scanned : 400,000,000 (toute la table)
```

**Optimisation : Partitionnement par mois** :
```sql
-- Créer table partitionnée
CREATE TABLE application_logs_partitioned (
    id BIGINT AUTO_INCREMENT,
    log_date DATE NOT NULL,
    level VARCHAR(20),
    message TEXT,
    PRIMARY KEY (id, log_date),
    INDEX idx_date_level (log_date, level)
) 
PARTITION BY RANGE (TO_DAYS(log_date)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    -- ... partitions mensuelles
    PARTITION p202512 VALUES LESS THAN (TO_DAYS('2026-01-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Migrer données (peut prendre plusieurs heures)
INSERT INTO application_logs_partitioned  
SELECT * FROM application_logs;  
```

**Résultat** :
```sql
-- Même requête sur table partitionnée
SELECT * FROM application_logs_partitioned  
WHERE log_date >= CURDATE() - INTERVAL 7 DAY  
ORDER BY log_date DESC  
LIMIT 1000;  

-- EXPLAIN montre partition pruning
partitions: p202512  ← Une seule partition accédée

-- Performance :
-- Temps : 0.3 secondes (50x plus rapide)
-- Rows scanned : 8,000,000 (seulement partition courante)
```

**Bonus : Archivage simplifié** :
```sql
-- Archiver partitions anciennes vers moteur S3
ALTER TABLE application_logs_partitioned 
  EXCHANGE PARTITION p202401 
  WITH TABLE logs_archive_202401;

-- Déplacer table vers S3
ALTER TABLE logs_archive_202401 ENGINE=S3;

-- Coût stockage : -90% pour données froides
```

**Gains** :
- ⚡ Requêtes récentes : 15s → 0.3s (50x)
- 💾 Maintenance : DROP PARTITION (instantané vs DELETE lent)
- 💰 Coûts stockage : -70% avec archivage S3
- 🗄️ Scalabilité : Supporte croissance illimitée

---

## 🛠️ Outils professionnels essentiels

### Performance Schema : Instrumentation complète

```sql
-- Activer Performance Schema (my.cnf)
[mysqld]
performance_schema = ON  
performance-schema-instrument = 'statement/%=ON'  
performance-schema-consumer-statements-digest = ON  

-- Top 10 requêtes par temps d'exécution
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    AVG_TIMER_WAIT/1000000000000 AS avg_sec,
    MAX_TIMER_WAIT/1000000000000 AS max_sec,
    SUM_TIMER_WAIT/1000000000000 AS total_sec
FROM performance_schema.events_statements_summary_by_digest  
WHERE SCHEMA_NAME = 'mydb'  
ORDER BY SUM_TIMER_WAIT DESC  
LIMIT 10;  
```

**Use cases** :
- 🔍 Identifier requêtes lentes
- 📊 Analyser patterns d'accès
- 🔒 Détecter problèmes de verrous
- 💾 Monitorer I/O par table

---

### sys Schema : Diagnostic simplifié

```sql
-- Tables avec le plus d'I/O
SELECT * FROM sys.io_global_by_file_by_bytes  
LIMIT 10;  

-- Index inutilisés
SELECT * FROM sys.schema_unused_indexes;

-- Tables sans clé primaire (DANGER !)
SELECT * FROM sys.schema_tables_with_no_pk;

-- Requêtes avec full table scan
SELECT * FROM sys.statements_with_full_table_scans  
LIMIT 20;  
```

---

### Percona Toolkit : pt-query-digest

```bash
# Analyser slow query log
pt-query-digest /var/log/mysql/slow.log > report.txt

# Output : Rapport statistique complet
# - Top queries by execution time
# - Top queries by count
# - Query patterns (fingerprints)
# - Lock time analysis
# - Rows examined statistics

# Exemple de sortie :
# Query 1: 0.52 QPS, 0.89x concurrency, ID 0xABCD...
# This item is included in the report because it matches --limit.
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         15    1834
# Exec time     45    523s    50ms      2s   285ms   450ms   180ms   245ms
# Lock time      5      2s     0us    15ms     1ms     2ms     2ms   950us
# Rows sent      8  3.67M       1   5.00k   2.05k   4.96k   1.89k   1.86k
# Rows examine  78   156M      10   100k   87.2k    96k    45.2k    89k
```

---

### sysbench : Benchmarking standard

```bash
# Préparer données de test
sysbench oltp_read_write \
  --table-size=1000000 \
  --tables=10 \
  --mysql-db=testdb \
  prepare

# Benchmark OLTP
sysbench oltp_read_write \
  --table-size=1000000 \
  --tables=10 \
  --threads=64 \
  --time=300 \
  --report-interval=10 \
  --mysql-db=testdb \
  run

# Output :
# SQL statistics:
#     queries performed:
#         read:                            1806524
#         write:                           516435
#         other:                           258218
#         total:                           2581177
#     transactions:                        129108 (430.30 per sec.)
#     queries:                             2581177 (8606.00 per sec.)
#     Latency (ms):
#          min:                                 2.05
#          avg:                                17.23
#          max:                               185.42
#          95th percentile:                    33.72
```

---

## 🎯 Prérequis pour cette partie

Cette partie est **techniquement exigeante**. Assurez-vous de maîtriser :

### MariaDB
- ✅ Index (B-Tree, Hash, Full-Text, composition)
- ✅ Transactions et niveaux d'isolation
- ✅ EXPLAIN et plans d'exécution
- ✅ Moteurs de stockage (InnoDB internals)
- ✅ Binary logs et replication

### Système
- ✅ Linux I/O stack (filesystem, cache, scheduler)
- ✅ Memory management (buffers, cache, swap)
- ✅ CPU scheduling et threads
- ✅ Storage (HDD vs SSD vs NVMe)
- ✅ Network stack et latency

### Outils
- ✅ Shell scripting (bash)
- ✅ Monitoring (Prometheus, Grafana)
- ✅ Profiling (perf, strace, iotop)

---

## 🚀 Prêt pour l'expertise en performance ?

Cette partie vous transformera en **expert d'optimisation de performance**, capable de :

- ✅ Diagnostiquer n'importe quel problème de performance
- ✅ Améliorer les performances de 10x, 50x, voire 100x
- ✅ Réduire les coûts d'infrastructure de 50-80%
- ✅ Concevoir des systèmes performants dès le départ
- ✅ Automatiser la détection de régressions
- ✅ Être reconnu comme expert de référence

Les compétences de cette partie sont **hautement valorisées** sur le marché. Les experts en optimisation de bases de données peuvent prétendre à des salaires 30-50% supérieurs aux DBA standards.

**Préparez-vous à devenir le sauveur des systèmes en crise de performance.** ⚡

---

## ➡️ Prochaine étape

**Module 15 : Performance et Tuning** → Plongez dans la méthodologie d'optimisation, les configurations critiques, l'analyse de requêtes lentes, et les techniques avancées pour des performances optimales.

Bienvenue dans le monde de l'optimisation data-driven ! 📊

---

**MariaDB** : Version 12.3 LTS

⏭️ [Performance et Tuning](/15-performance-tuning/README.md)

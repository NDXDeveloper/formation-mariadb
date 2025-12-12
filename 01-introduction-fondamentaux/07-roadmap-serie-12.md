🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.7 Roadmap : série 12.x (12.0→12.2 rolling, 12.3 LTS prévu Q2 2026) 🆕

> **Niveau** : Débutant
> **Durée estimée** : 30 minutes
> **Prérequis** : Sections 1.5 et 1.6 (Politique de versions et cycle de support)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre la roadmap de MariaDB pour 2025-2027
- Connaître les versions prévues de la série 12.x
- Identifier les fonctionnalités attendues dans les futures versions
- Anticiper l'évolution de MariaDB à long terme
- Suivre activement les développements de MariaDB
- Planifier l'adoption des futures versions
- Comprendre les tendances et orientations stratégiques

---

## Introduction

Vous connaissez maintenant les versions **actuelles** de MariaDB (10.6, 10.11, 11.4, 11.8). Mais **qu'en est-il du futur** ? Quelles sont les prochaines versions ? Quelles nouvelles fonctionnalités sont prévues ? Comment MariaDB va-t-il évoluer ?

**Pourquoi s'intéresser à la roadmap ?**
- 🔮 **Anticiper** les évolutions technologiques
- 📅 **Planifier** vos migrations futures
- 💡 **Découvrir** les innovations à venir
- 🎯 **Influencer** le développement (communauté)
- 🚀 **Rester compétitif** avec votre stack technique
- 📚 **Se former** en avance sur les nouvelles features

Dans cette section, nous allons explorer la **roadmap de MariaDB**, en se concentrant sur la **série 12.x** et les perspectives à long terme jusqu'en 2027 et au-delà.

---

## La série 12.x : Vue d'ensemble

### 🎯 Positionnement de la série 12.x

La série **12.x** succède à la série **11.x** et représente une **évolution majeure** de MariaDB avec :
- 🚀 Nouvelles fonctionnalités SQL
- ⚡ Optimisations de performance
- 🔒 Améliorations de sécurité
- 🤖 Intégration IA/ML enrichie
- 📊 Capacités analytiques renforcées

**Philosophy série 12.x** : Innovation continue tout en maintenant la compatibilité.

### 📅 Timeline série 12.x (2024-2026)

```
┌────────────────────────────────────────────────────────────────┐
│                   Série 12.x Timeline                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  2024          2024          2025          2025          2026  │
│  Sep           Déc           Mars          Juin          Q2    │
│   │             │             │             │             │    │
│   ▼             ▼             ▼             ▼             ▼    │
│ ┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐      ┌──────┐   │
│ │12.0 │─────►│12.1 │─────►│12.2 │─────►│12.3 │─────►│12.3  │   │
│ │Roll.│ 3m   │Roll.│ 3m   │Roll.│ 3m   │Roll.│      │LTS   │   │
│ └─────┘      └─────┘      └─────┘      └─────┘      └──────┘   │
│   │            │            │            │             │       │
│   │            │            │            │             │       │
│ Beta       Features     Features     Features      Support     │
│ test      continues    continues    continues      3 ans       │
│                                                    (→2029)     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 🗓️ Versions et dates clés

| Version | Type | Date GA | Support jusqu'à | Statut |
|---------|------|---------|-----------------|--------|
| **12.0** | Rolling | Sep 2024 | Déc 2024 | ✅ Sortie |
| **12.1** | Rolling | Déc 2024 | Mars 2025 | ✅ Sortie |
| **12.2** | Rolling | Mars 2025 | Juin 2025 | 🔄 Actuelle |
| **12.3** | Rolling | Juin 2025 | Q2 2026 | 🔮 Prévue |
| **12.3** | LTS | Q2 2026 | Q2 2029 | 🔮 Prévue (3 ans) |

💡 **Note** : 12.3 existe d'abord en Rolling (Juin 2025), puis devient LTS (Q2 2026) après stabilisation.

---

## MariaDB 12.0 : Première Rolling (Septembre 2024)

### 🆕 Nouveautés principales 12.0

#### 1️⃣ **Optimisations InnoDB**

**Améliorations du moteur** :
- ⚡ Flush log optimisé (réduction latence)
- 💾 Buffer pool éviction algorithm amélioré
- 🔄 Redo log parallel write

**Impact** :
```
Workloads intensifs en écriture : +15-25% performance
Latence moyenne : -10-20%
```

#### 2️⃣ **Extensions JSON**

**Nouvelles fonctions JSON** :
```sql
-- JSON_SEARCH avec expressions régulières
SELECT JSON_SEARCH(data, 'all', 'pattern%') FROM documents;

-- JSON_MERGE_PRESERVE amélioré
SELECT JSON_MERGE_PRESERVE('{"a":1}', '{"a":2, "b":3}');
-- Résultat : {"a":[1,2], "b":3}

-- JSON_OVERLAPS pour comparaison
SELECT JSON_OVERLAPS('[1,2,3]', '[3,4,5]');
-- Résultat : 1 (true, car 3 est commun)
```

#### 3️⃣ **Améliorations Vector (IA/ML)**

**Évolution de MariaDB Vector** :
- 🤖 Nouvelles fonctions de distance
- 📊 Support de dimensions variables
- ⚡ Optimisations SIMD étendues (ARM, Power10)

```sql
-- Nouvelle fonction de distance Manhattan (L1)
CREATE TABLE embeddings_v2 (
    id INT PRIMARY KEY,
    vector VECTOR(1536),
    VECTOR INDEX idx_vec (vector)
        USING HNSW
        WITH (distance='manhattan')  -- ← Nouveau
) ENGINE=InnoDB;
```

#### 4️⃣ **SQL Standard 2023 partial support**

**Nouveautés SQL:2023** :
- 🆕 `IS JSON` predicate
- 🆕 `JSON_SERIALIZE` / `JSON_PARSE`
- 🆕 Multiset operations étendues

```sql
-- IS JSON predicate
SELECT column_name
FROM table_name
WHERE json_column IS JSON;

-- JSON_SERIALIZE avec options
SELECT JSON_SERIALIZE(json_data
    RETURNING VARCHAR(1000)
    WITH UNIQUE KEYS)
FROM documents;
```

---

## MariaDB 12.1 : Seconde Rolling (Décembre 2024)

### 🆕 Nouveautés principales 12.1

#### 1️⃣ **Performance ColumnStore**

**Analytique plus rapide** :
- 📊 Compression améliorée (nouvel algorithme)
- ⚡ Parallel query execution optimisé
- 💾 Cache de métadonnées plus efficace

**Gains observés** :
```
Requêtes agrégation massive : +30-40% plus rapides
Compression : Ratio amélioré de 10-15%
```

#### 2️⃣ **Replication enhancements**

**Réplication plus robuste** :
- 🔄 Parallel replication amélioré
- 🛡️ Crash recovery plus rapide
- 📊 Monitoring metrics enrichis

```sql
-- Nouvelles variables de monitoring
SHOW STATUS LIKE 'Rpl_semi_sync%';
-- + nouvelles métriques :
-- Rpl_semi_sync_replica_lag_time
-- Rpl_semi_sync_replica_queue_size
```

#### 3️⃣ **Sécurité : Audit enrichi**

**Traçabilité améliorée** :
```sql
-- Audit des changements de schéma
SET GLOBAL server_audit_events = 'CONNECT,QUERY,TABLE,QUERY_DDL';

-- Audit filtré par utilisateur
SET GLOBAL server_audit_incl_users = 'admin1,admin2,auditor';

-- Nouvelles options de rotation des logs
SET GLOBAL server_audit_file_rotate_size = 100000000; -- 100 MB
```

#### 4️⃣ **Temporal tables improvements**

**Tables temporelles enrichies** :
```sql
-- Nouvelles options pour System-Versioned Tables
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME (
    PARTITION p_current CURRENT,
    PARTITION p_history HISTORY
  )
  -- ← Partitionnement automatique amélioré
;
```

---

## MariaDB 12.2 : Troisième Rolling (Mars 2025)

### 🆕 Nouveautés principales 12.2

#### 1️⃣ **Window Functions extensions**

**Nouvelles fonctions fenêtre** :
```sql
-- NTH_VALUE avec options
SELECT
    product,
    sales,
    NTH_VALUE(sales, 2) FROM FIRST
        OVER (PARTITION BY category ORDER BY sales DESC) as second_best
FROM sales_data;

-- PERCENT_RANK et CUME_DIST améliorés
SELECT
    employee,
    salary,
    PERCENT_RANK() OVER (ORDER BY salary) as percentile
FROM employees;
```

#### 2️⃣ **S3 Storage Engine v2**

**Archivage cloud amélioré** :
- ☁️ Support multi-cloud (AWS, Azure, GCP, MinIO)
- 🔒 Encryption at rest native
- ⚡ Parallel reads/writes

```sql
-- Configuration S3 enrichie
CREATE TABLE archived_orders (
    order_id INT,
    order_date DATE,
    data TEXT
) ENGINE=S3
  S3_ENDPOINT='https://s3.amazonaws.com'
  S3_REGION='us-east-1'
  S3_BUCKET='my-archives'
  S3_ENCRYPTION='AES256'  -- ← Nouveau
  S3_THREADS=4;            -- ← Nouveau (parallel)
```

#### 3️⃣ **Unicode 15.1 support**

**Support Unicode étendu** :
- 🌍 Nouveaux caractères et emojis
- 📝 Collations UCA 15.1
- 🔤 Normalisation améliorée

```sql
-- Nouveaux emojis supportés
INSERT INTO messages (content)
VALUES ('Hello 🫠🫱🏽‍🫲🏻');  -- Unicode 15.0+ emojis

-- Collations UCA 15.1
CREATE TABLE texts (
    content TEXT
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_uca1510_ai_ci;  -- ← Nouvelle collation
```

#### 4️⃣ **Query Optimizer SSD-aware**

**Optimiseur conscient du hardware** :
```sql
-- L'optimiseur détecte automatiquement SSD vs HDD
-- et adapte son coût de calcul

-- Configuration manuelle possible
SET GLOBAL optimizer_disk_type = 'SSD';  -- ou 'HDD', 'NVME'
SET GLOBAL optimizer_random_read_cost = 0.5;  -- Coût SSD
```

**Impact** :
```
Plans d'exécution mieux adaptés au hardware
Index scans préférés sur SSD (plus rapides que sequential sur SSD)
```

---

## MariaDB 12.3 LTS : Stabilisation (Q2 2026) 🎯

### 🎯 Objectif 12.3 LTS

**12.3 sera la consolidation** de toutes les innovations 12.0, 12.1, 12.2 :
- ✅ Toutes les features des Rolling précédentes
- ✅ Bugs corrigés
- ✅ Performance validée
- ✅ Production-ready
- 🛡️ Support 3 ans (Q2 2026 → Q2 2029)

### 🆕 Fonctionnalités attendues (cumul 12.0-12.3)

**Résumé des innovations série 12.x** :

| Domaine | Améliorations |
|---------|---------------|
| **Performance InnoDB** | +15-25% écritures, redo log parallel |
| **JSON** | Nouvelles fonctions, SQL:2023 partial |
| **Vector IA** | Distance Manhattan, optimisations SIMD |
| **ColumnStore** | +30-40% agrégations, compression améliorée |
| **Réplication** | Parallel optimisé, monitoring enrichi |
| **Sécurité** | Audit DDL, filtres enrichis |
| **Temporal tables** | Partitionnement automatique |
| **Window functions** | NTH_VALUE, PERCENT_RANK étendus |
| **S3 Storage** | Multi-cloud, encryption, parallel I/O |
| **Unicode** | 15.1, nouveaux emojis, UCA 15.1 |
| **Optimizer** | SSD-aware, cost model amélioré |

### 📅 Quand migrer vers 12.3 LTS ?

**Timeline recommandée** :
```
Q2 2026 : 12.3 LTS sort
  │
  ├─ Q2-Q3 2026 : Stabilisation (attendre 12.3.2-12.3.3)
  │
  ├─ Q4 2026 : Tester en staging
  │
  └─ Q1 2027 : Migration production

Ou : Rester sur 11.8 LTS jusqu'à fin 2027, puis migrer vers 12.3 LTS
```

---

## Au-delà de 12.x : Série 13.x et futur (2027+)

### 🔮 Prévisions série 13.x

**Attendu à partir de 2027**, la série 13.x pourrait apporter :

#### 1️⃣ **SQL:2026 Support (futur standard)**

Le prochain standard SQL (2026) inclura probablement :
- 🆕 Graph queries (SQL/PGQ - Property Graph Queries)
- 🆕 Machine Learning intégré au SQL
- 🆕 Meilleure intégration JSON/relationnel

```sql
-- Hypothétique SQL:2026 Graph queries
SELECT person.name, COUNT(friend) as friend_count
FROM persons AS person
  MATCH (person)-[:FRIEND_OF]->(friend)
GROUP BY person.name;
```

#### 2️⃣ **Vector Search v3**

**IA/ML de nouvelle génération** :
- 🤖 Support embeddings de très haute dimension (8192+)
- 📊 Index multi-vecteurs
- ⚡ Recherche hybride dense+sparse
- 🔄 Rafraîchissement incrémental d'index

```sql
-- Hypothétique : Multi-vector index
CREATE TABLE documents (
    id INT PRIMARY KEY,
    text_embedding VECTOR(1536),
    image_embedding VECTOR(768),
    VECTOR INDEX idx_text (text_embedding) USING HNSW,
    VECTOR INDEX idx_image (image_embedding) USING HNSW
);
```

#### 3️⃣ **Distributed SQL natif**

**Sharding et distribution** :
- 🌍 Sharding automatique intégré
- 🔄 Cross-shard queries transparentes
- 📊 Load balancing automatique

#### 4️⃣ **Quantum-safe cryptography**

**Préparation post-quantique** :
- 🔒 Algorithmes résistants à l'informatique quantique
- 🔐 Migration progressive des algorithmes de chiffrement

### 📅 Roadmap long terme (2025-2028)

```
┌────────────────────────────────────────────────────────────┐
│              MariaDB Roadmap 2025-2028                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  2025           2026           2027           2028         │
│   │              │              │              │           │
│   ▼              ▼              ▼              ▼           │
│                                                            │
│ 11.8 LTS ──────────────────────────────────►               │
│ (→2028)                                                    │
│                                                            │
│ 12.0-12.2 ►12.3 LTS ─────────────────────────►             │
│ Rolling    (Q2 26)         (→2029)                         │
│                                                            │
│                   13.0-13.2 ►13.3 LTS? ──────►             │
│                   Rolling?   (Q4 27?)  (→2030?)            │
│                                                            │
│ Innovations continues :                                    │
│ • Vector v2-v3                                             │
│ • SQL:2023 → SQL:2026                                      │
│ • ColumnStore v3                                           │
│ • Distributed SQL                                          │
│ • Quantum-safe crypto                                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Thématiques stratégiques de développement

MariaDB Foundation a identifié **5 axes stratégiques** pour l'évolution future :

### 1️⃣ **IA et Machine Learning** 🤖

**Objectif** : Faire de MariaDB le SGBDR de référence pour l'IA.

**Développements** :
- ✅ **MariaDB Vector** (11.8) : Fondation
- 🔄 **Vector v2** (12.x) : Enrichissements
- 🔮 **Vector v3** (13.x?) : ML intégré au SQL

**Vision** :
```sql
-- Futur : ML directement dans SQL ?
SELECT
    product_id,
    PREDICT_SALES(historical_data, seasonality) as forecast
FROM sales_history
WHERE category = 'Electronics';
```

### 2️⃣ **Analytics et OLAP** 📊

**Objectif** : Rivaliser avec data warehouses spécialisés.

**Développements** :
- ✅ **ColumnStore** : Mature
- 🔄 **ColumnStore v2** (12.x) : Performance++
- 🔮 **ColumnStore v3** : Distributed

**Vision** : MariaDB comme solution unified OLTP + OLAP.

### 3️⃣ **Cloud-native et Kubernetes** ☁️

**Objectif** : Meilleure expérience cloud.

**Développements** :
- ✅ **mariadb-operator** : Production-ready
- 🔄 **Operator v2** : Plus de features
- 🔮 **Serverless MariaDB** : Auto-scaling

**Vision** :
```yaml
# Futur : MariaDB serverless ?
apiVersion: mariadb.com/v1
kind: ServerlessMariaDB
metadata:
  name: my-app-db
spec:
  autoscaling:
    minReplicas: 0  # Scale to zero !
    maxReplicas: 100
  storage: 100Gi
```

### 4️⃣ **Sécurité et Compliance** 🔒

**Objectif** : Standards enterprise et réglementaires.

**Développements** :
- ✅ **Encryption at rest** : Mature
- ✅ **TLS par défaut** (11.8)
- 🔄 **Audit enrichi** (12.x)
- 🔮 **Quantum-safe** (13.x?)

**Vision** : Zero-trust database architecture.

### 5️⃣ **Performance et Efficacité** ⚡

**Objectif** : Performance de classe mondiale.

**Développements** :
- ✅ **InnoDB optimizations** : Continues
- ✅ **Thread Pool** : Gratuit
- 🔄 **Query Optimizer SSD-aware** (12.x)
- 🔮 **NUMA-aware** : Meilleur sur gros serveurs

---

## Comment suivre l'évolution de MariaDB

### 📰 Sources officielles

#### 1️⃣ **Blog MariaDB**
```
URL : https://mariadb.com/blog/
Contenu : Annonces, articles techniques, roadmap updates
Fréquence : 2-4 articles/semaine
```

#### 2️⃣ **MariaDB Foundation Blog**
```
URL : https://mariadb.org/blog/
Contenu : Développement communautaire, governance, features
Fréquence : 1-2 articles/semaine
```

#### 3️⃣ **Release Notes**
```
URL : https://mariadb.com/kb/en/release-notes/
Contenu : Détails de chaque release
Mise à jour : À chaque release
```

#### 4️⃣ **Roadmap officielle**
```
URL : https://mariadb.org/roadmap/
Contenu : Versions prévues, features planifiées
Mise à jour : Trimestrielle
```

### 💬 Canaux communautaires

#### 1️⃣ **Zulip Chat**
```
URL : https://mariadb.zulipchat.com/
Utilité : Discussions en temps réel, questions techniques
Streams : #announce, #development, #general
```

#### 2️⃣ **Mailing Lists**
```
announce@mariadb.org  : Annonces releases
developers@mariadb.org : Discussions développement
```

#### 3️⃣ **JIRA (Bug Tracker)**
```
URL : https://jira.mariadb.org/
Utilité : Suivre le développement des features
Accès : Public (lecture), compte gratuit (contribution)
```

#### 4️⃣ **GitHub**
```
URL : https://github.com/MariaDB/server
Utilité : Code source, Pull Requests, Issues
Watch : Activer notifications pour releases
```

### 🎥 Événements

#### 1️⃣ **MariaDB Server Fest**
```
Quand : Annuel (généralement Q2 ou Q3)
Quoi : Conférence officielle, roadmap keynotes
Où : En ligne + présentiel (lieu varie)
```

#### 2️⃣ **MariaDB User Meetups**
```
Quand : Mensuels (locaux)
Quoi : Présentations techniques, networking
Où : Villes majeures (Paris, London, SF, etc.)
```

#### 3️⃣ **Webinars techniques**
```
Fréquence : 1-2 par mois
Sujets : Nouvelles features, best practices
Enregistrements : Disponibles sur YouTube
```

### 📊 Métriques et tendances

**Sites de référence** :
- **DB-Engines Ranking** : https://db-engines.com/en/ranking
- **Stack Overflow Trends** : Popularité des technologies
- **GitHub Stars** : Intérêt communauté

---

## Se préparer aux futures versions

### 🎯 Stratégie d'adoption progressive

#### Phase 1 : Veille technologique (continue)

```
Actions :
✅ S'abonner aux newsletters
✅ Lire release notes mensuellement
✅ Suivre blogs officiels
✅ Participer à 1-2 webinars/an

Temps : 1-2h/mois
```

#### Phase 2 : Tests préliminaires (3-6 mois avant GA)

```
Actions :
✅ Télécharger beta/RC
✅ Tester en environnement dev
✅ Identifier features intéressantes
✅ Tester compatibilité applicative

Temps : 4-8h sur 1 mois
```

#### Phase 3 : Validation (1-3 mois après GA)

```
Actions :
✅ Installer version stable en staging
✅ Tests fonctionnels complets
✅ Tests de charge
✅ Validation équipe

Temps : 16-40h sur 2 mois
```

#### Phase 4 : Migration (quand mature)

```
Actions :
✅ Planning détaillé
✅ Communication équipes
✅ Migration progressive
✅ Monitoring renforcé

Temps : Variable selon taille
```

### 🧪 Lab de test personnel

**Créer un environnement de test** :

```bash
# Docker : Tester rapidement nouvelles versions
docker run -d \
  --name mariadb-12-2-test \
  -e MARIADB_ROOT_PASSWORD=testpass \
  -p 3307:3306 \
  mariadb:12.2

# Se connecter
mariadb -h 127.0.0.1 -P 3307 -u root -p

# Tester nouvelles features
MariaDB> SELECT VERSION();
MariaDB> -- Tester JSON_OVERLAPS, etc.
```

**Automatiser les tests** :
```bash
#!/bin/bash
# Script de test automatique

versions=("11.8" "12.0" "12.1" "12.2")

for v in "${versions[@]}"; do
  echo "Testing MariaDB $v..."
  docker run --rm mariadb:$v mariadb --version
  # Ajouter tests SQL automatisés
done
```

### 📚 Formation continue

**Ressources d'apprentissage** :

1. **MariaDB University** (gratuit)
   - Cours en ligne
   - Certifications
   - Labs pratiques

2. **YouTube MariaDB**
   - Tutoriels
   - Webinars enregistrés
   - Conférences

3. **Blogs techniques communautaires**
   - Planet MariaDB
   - Percona Blog (compatible)
   - Dev.to (tag: mariadb)

4. **Livres**
   - "MariaDB Cookbook" (updated régulièrement)
   - "High Performance MySQL" (applicable MariaDB)

---

## Comparaison avec la concurrence

### 📊 MariaDB vs Concurrents : Roadmap

| Base de données | Fréquence releases | LTS Duration | Innovation |
|-----------------|-------------------|--------------|------------|
| **MariaDB** | LTS ~18m, Rolling ~3m | 3 ans | ⭐⭐⭐⭐⭐ |
| **PostgreSQL** | Annuelle (majeure) | ~5 ans | ⭐⭐⭐⭐⭐ |
| **MySQL** | 2-3 ans (majeure) | 5-8 ans | ⭐⭐⭐ |
| **SQL Server** | ~3-5 ans | 10 ans (Extended) | ⭐⭐⭐⭐ |
| **Oracle DB** | ~3 ans | 5-10 ans | ⭐⭐⭐⭐ |

**Forces de MariaDB** :
- ✅ **Innovation rapide** : Nouvelles features tous les 3-18 mois
- ✅ **Flexibilité** : LTS pour stabilité + Rolling pour innovation
- ✅ **Open source** : Roadmap transparente et communautaire
- ✅ **Rétrocompatibilité** : Migration MySQL facile

**Défis de MariaDB** :
- ⚠️ Support LTS plus court que PostgreSQL (3 vs 5 ans)
- ⚠️ Écosystème plus petit que MySQL/PostgreSQL
- ⚠️ Moins de documentation tierce

### 🎯 Positionnement stratégique

**MariaDB se positionne comme** :
1. Alternative open source premium à MySQL
2. SGBDR moderne avec focus IA/Analytics
3. Solution unifiée OLTP + OLAP
4. Cloud-native friendly

---

## Tendances futures du secteur

### 🔮 Ce qui arrive dans les bases de données (2025-2030)

#### 1️⃣ **IA intégré natif** 🤖

**Tendance** : Toutes les BDD intègreront des capacités IA.

- PostgreSQL : pgvector, extensions ML
- MongoDB : Atlas Vector Search
- **MariaDB** : MariaDB Vector (avance sur MySQL)

**Prédiction** : En 2027, recherche vectorielle = feature standard.

#### 2️⃣ **Serverless et auto-scaling** ☁️

**Tendance** : Bases de données qui scalent automatiquement.

- AWS Aurora Serverless v2
- Azure SQL Database Serverless
- PlanetScale (MySQL-compatible)

**MariaDB** : SkySQL propose déjà du serverless.

#### 3️⃣ **Multi-model convergence** 🔄

**Tendance** : Support de plusieurs modèles de données (relationnel, document, graph, vector) dans une seule BDD.

**MariaDB déjà** :
- ✅ Relationnel (InnoDB)
- ✅ Document (JSON)
- ✅ Colonnes (ColumnStore)
- ✅ Vecteurs (Vector)
- 🔮 Graph ? (peut-être série 13.x)

#### 4️⃣ **Edge computing** 📱

**Tendance** : Bases de données légères pour edge/IoT.

- SQLite : Dominance edge
- DuckDB : Analytics edge
- **MariaDB** : Peut jouer ce rôle (léger, portable)

#### 5️⃣ **Quantum-readiness** 🔬

**Tendance** : Préparation post-quantique.

Toutes les BDD devront adopter :
- Chiffrement résistant quantique
- Nouveaux algorithmes crypto
- Migration progressive

---

## ✅ Points clés à retenir

- 🗓️ **Série 12.x** : 12.0 (Sep 24) → 12.1 (Déc 24) → 12.2 (Mar 25) → 12.3 LTS (Q2 26)
- 🎯 **12.3 LTS prévu Q2 2026** avec support 3 ans (→ Q2 2029)
- 🚀 **Innovations 12.x** : InnoDB +25%, JSON étendu, Vector v2, ColumnStore +40%, S3 multi-cloud
- 🔮 **Série 13.x** attendue 2027+ avec SQL:2026, Vector v3, Distributed SQL
- 🤖 **5 axes stratégiques** : IA/ML, Analytics, Cloud-native, Sécurité, Performance
- 📰 **Sources officielles** : blog, roadmap, release notes, JIRA
- 💬 **Communauté** : Zulip, mailing lists, GitHub, meetups
- 🎯 **Adoption progressive** : Veille → Tests beta → Validation → Migration
- 📊 **Positionnement** : Innovation rapide (vs MySQL), stabilité (LTS), open source
- 🔮 **Tendances** : IA natif, serverless, multi-model, edge, quantum-safe
- 🛠️ **Tests Docker** : Facile de tester nouvelles versions localement
- 📚 **Formation continue** : MariaDB University, YouTube, blogs techniques
- ⏰ **Cycle rapide** : Nouvelle feature tous les 3 mois (Rolling) ou ~18 mois (LTS)

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [MariaDB Roadmap](https://mariadb.org/roadmap/)
- [Release Calendar](https://mariadb.com/kb/en/release-calendar/)
- [Development Plan](https://mariadb.org/development/)

### 📰 Blogs et actualités
- [MariaDB Blog](https://mariadb.com/blog/)
- [MariaDB Foundation Blog](https://mariadb.org/blog/)
- [Planet MariaDB](https://planet.mariadb.org/)

### 💬 Communauté
- [Zulip Chat](https://mariadb.zulipchat.com/)
- [Mailing Lists](https://lists.mariadb.org/)
- [JIRA](https://jira.mariadb.org/)
- [GitHub](https://github.com/MariaDB/server)

### 🎥 Événements
- [MariaDB Server Fest](https://mariadb.com/resources/events/)
- [Webinars](https://mariadb.com/resources/webinars/)
- [YouTube Channel](https://www.youtube.com/user/mariadbserver)

### 📊 Métriques et tendances
- [DB-Engines Ranking](https://db-engines.com/en/ranking/relational+dbms)
- [Stack Overflow Trends](https://insights.stackoverflow.com/trends)

---

## ➡️ Section suivante

**[1.8 - Installation et configuration initiale](./08-installation-configuration.md)**

Vous connaissez maintenant l'histoire, l'architecture, les versions et la roadmap de MariaDB. Il est temps de **passer à la pratique** ! Dans la section suivante, nous allons installer MariaDB sur votre machine et le configurer pour vos premiers pas. Vous apprendrez à installer MariaDB sur Linux, Windows et macOS, à le sécuriser, et à effectuer votre première connexion.

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.7*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Installation et configuration initiale](/01-introduction-fondamentaux/08-installation-configuration.md)

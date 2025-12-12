🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.10 Moteurs spécialisés

> **Niveau** : Avancé
> **Durée estimée** : 1-2 heures
> **Prérequis** : Sections 7.1-7.9 (moteurs principaux et conversions)

> **Public cible** : DBA, Architectes de bases de données, Ingénieurs systèmes

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Identifier les moteurs spécialisés disponibles dans MariaDB
- Comprendre les cas d'usage spécifiques de chaque moteur
- Évaluer quand utiliser un moteur spécialisé vs un moteur principal
- Connaître les limitations et contraintes de chaque moteur
- Concevoir des architectures hybrides intégrant moteurs spécialisés
- Éviter les erreurs d'utilisation courantes
- Choisir le moteur spécialisé approprié selon le besoin

---

## Introduction

Au-delà des **moteurs principaux** (InnoDB, ColumnStore, S3, Vector/HNSW), MariaDB propose une palette de **moteurs spécialisés** conçus pour des cas d'usage très spécifiques. Ces moteurs ne sont pas destinés à un usage général mais excellent dans leur domaine de prédilection.

### Philosophie des moteurs spécialisés

```
Principe de spécialisation :
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Moteurs principaux (90% des besoins)               │
│  ┌──────────────────────────────────────────────┐   │
│  │ InnoDB     : OLTP, transactions, général     │   │
│  │ ColumnStore: OLAP, analytics, data warehouse │   │
│  │ S3         : Archivage froid, économique     │   │
│  │ Vector     : IA, recherche sémantique        │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  Moteurs spécialisés (10% des besoins)              │
│  ┌──────────────────────────────────────────────┐   │
│  │ Memory     : Cache ultra-rapide en RAM       │   │
│  │ Archive    : Compression maximale, insertion │   │
│  │ Blackhole  : /dev/null SQL, réplication      │   │
│  │ CSV        : Export/import fichiers plats    │   │
│  │ CONNECT    : Fédération données externes     │   │
│  │ Spider     : Sharding horizontal             │   │
│  │ FederatedX : Proxy distant                   │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Quand utiliser un moteur spécialisé ?

**Critères de décision** :
1. **Besoin très spécifique** non couvert par moteurs principaux
2. **Compromis acceptable** (limitations vs bénéfices)
3. **Volume limité** ou temporaire
4. **Architecture hybride** avec moteurs principaux

> "Un moteur spécialisé est comme un outil de chirurgie : très efficace pour son usage prévu, dangereux si mal utilisé."

---

## Vue d'ensemble des moteurs spécialisés

### Tableau comparatif

| Moteur | Type | Statut | Cas d'usage principal | Performance | Contraintes majeures |
|--------|------|--------|----------------------|-------------|----------------------|
| **Memory (HEAP)** | RAM pure | ✅ Stable | Cache temporaire, sessions | ⭐⭐⭐⭐⭐ | Volatile (perte au redémarrage) |
| **Archive** | Compression | ✅ Stable | Logs historiques, audit | ⭐⭐ | INSERT only, pas d'index |
| **Blackhole** | /dev/null | ✅ Stable | Test réplication, benchmark | ⭐⭐⭐⭐⭐ | Pas de stockage réel |
| **CSV** | Fichier texte | ✅ Stable | Export/import, intégration | ⭐⭐ | Pas d'index, lent |
| **CONNECT** | Fédération | ✅ Production | Intégration sources hétérogènes | ⭐⭐⭐ | Dépend source externe |
| **Spider** | Sharding | ✅ Production | Partitionnement horizontal | ⭐⭐⭐⭐ | Complexité haute |
| **FederatedX** | Proxy | ⚠️ Expérimental | Accès base distante | ⭐⭐ | Latence réseau |

### Matrice usage vs contraintes

```
          Simplicité ◄──────────────────────► Complexité
                │
    Rapide      │  Memory
    ⭐⭐⭐⭐⭐  │    ●
                │         CSV
    Performance │          ●
    ⭐⭐⭐⭐    │              Archive
                │               ●
    ⭐⭐⭐      │                   CONNECT
                │                    ●
    ⭐⭐        │                        Spider
                │                         ●
    Lent        │                             FederatedX
    ⭐          │                              ●
                │
                └────────────────────────────────────────
```

---

## Les 7 moteurs spécialisés

### 1. Memory (HEAP) : Tables en RAM

**Principe** : Données stockées intégralement en mémoire vive (RAM).

```
Architecture Memory :
┌────────────────────────────────────────────────────────┐
│                   Application                          │
└────────────────────────────────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│              MariaDB Server (SQL Layer)                │
└────────────────────────────────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│                Memory Storage Engine                   │
│  ┌──────────────────────────────────────────────────┐  │
│  │              RAM Heap                            │  │
│  │  Table data + Index (Hash/BTree)                 │  │
│  │  • Latence : < 0.01 ms                           │  │
│  │  • Throughput : Millions ops/sec                 │  │
│  │  • Volatile : Perte si redémarrage               │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Cas d'usage** :
- ✅ Cache applicatif (sessions utilisateurs)
- ✅ Tables de lookup fréquemment accédées
- ✅ Calculs intermédiaires temporaires
- ✅ Compteurs temps réel
- ❌ Données persistantes

**Exemple** :
```sql
CREATE TABLE user_sessions (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id INT,
    last_activity TIMESTAMP,
    data TEXT
) ENGINE=Memory;
```

**Limitations** :
- ⚠️ Volatile : Données perdues au redémarrage
- ⚠️ Taille limitée par `max_heap_table_size`
- ⚠️ Pas de TEXT/BLOB (jusqu'à VARCHAR(64K))
- ⚠️ Table-level locking

### 2. Archive : Compression maximale

**Principe** : Compression zlib agressive, INSERT-only.

```
Architecture Archive :
┌────────────────────────────────────────────────────────┐
│                  Insertion flow                        │
│                                                        │
│  INSERT → Compression zlib (ratio 10-20×) → Disque     │
│                                                        │
│  Lecture : Disque → Décompression → Résultat           │
│  (lente, scan séquentiel uniquement)                   │
└────────────────────────────────────────────────────────┘
```

**Cas d'usage** :
- ✅ Logs historiques (accès rare)
- ✅ Audit trail (compliance)
- ✅ Événements append-only
- ✅ Archivage intermédiaire (avant S3)
- ❌ Requêtes fréquentes

**Exemple** :
```sql
CREATE TABLE audit_logs (
    log_id BIGINT AUTO_INCREMENT,
    timestamp DATETIME,
    user_id INT,
    action VARCHAR(100),
    details TEXT,
    PRIMARY KEY (log_id)
) ENGINE=Archive;
```

**Limitations** :
- ❌ Pas d'UPDATE ni DELETE (INSERT/SELECT uniquement)
- ❌ Pas d'index (scan séquentiel obligatoire)
- ❌ Lectures lentes (décompression)
- ✅ Compression ~10-20× (vs 2-5× InnoDB)

### 3. Blackhole : Le /dev/null SQL

**Principe** : Accepte les données mais ne les stocke pas.

```
Architecture Blackhole :
┌───────────────────────────────────────────────────────┐
│  INSERT INTO blackhole_table VALUES (...);            │
│                     ↓                                 │
│              [Rien n'est stocké]                      │
│                     ↓                                 │
│              Binlog enregistré (si activé)            │
│                     ↓                                 │
│         Réplication vers esclaves                     │
└───────────────────────────────────────────────────────┘
```

**Cas d'usage** :
- ✅ Réplication intermédiaire (hub)
- ✅ Tests de performance (bypass I/O)
- ✅ Filtrage données de réplication
- ✅ Benchmark SQL (sans stockage)
- ❌ Stockage réel de données

**Exemple** :
```sql
-- Test performance INSERT sans I/O
CREATE TABLE benchmark_test (
    id INT,
    data VARCHAR(1000)
) ENGINE=Blackhole;

-- Benchmark
INSERT INTO benchmark_test
SELECT ... FROM huge_table;  -- Rapide (pas d'I/O)
```

**Architecture réplication** :
```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Master    │  binlog │  Blackhole  │  binlog │   Slave 1   │
│  (InnoDB)   │────────►│   (Hub)     │────────►│  (InnoDB)   │
└─────────────┘         └─────────────┘     │   └─────────────┘
                             │              │
                             │              └───►┌─────────────┐
                             │                   │   Slave 2   │
                             │                   │  (InnoDB)   │
                             │                   └─────────────┘
                             └──────────────────►┌─────────────┐
                                                 │   Slave 3   │
                                                 │(ColumnStore)│
                                                 └─────────────┘
```

### 4. CSV : Fichiers texte standards

**Principe** : Tables = Fichiers CSV sur disque.

```
Structure CSV :
/var/lib/mysql/mydb/
├── export_data.frm      # Définition table
├── export_data.CSV      # Données (fichier texte)
└── export_data.CSM      # Métadonnées

Contenu export_data.CSV :
1,"John Doe","john@example.com","2025-01-15"
2,"Jane Smith","jane@example.com","2025-01-16"
3,"Bob Johnson","bob@example.com","2025-01-17"
```

**Cas d'usage** :
- ✅ Export données pour Excel/scripts
- ✅ Import données externes (CSV)
- ✅ Intégration avec outils non-SQL
- ✅ Partage données inter-systèmes
- ❌ Performance (très lent)

**Exemple** :
```sql
CREATE TABLE csv_export (
    id INT,
    name VARCHAR(100),
    email VARCHAR(200),
    created_date DATE
) ENGINE=CSV;

INSERT INTO csv_export VALUES
(1, 'John Doe', 'john@example.com', '2025-01-15');

-- Fichier CSV créé automatiquement
-- Éditable manuellement avec éditeur texte
```

**Limitations** :
- ❌ Pas d'index (scan complet)
- ❌ Pas de transactions
- ❌ Performance faible
- ⚠️ Corruption facile si édition manuelle

### 5. CONNECT : Fédération de données

**Principe** : Accès à sources de données externes hétérogènes.

```
Architecture CONNECT :
┌────────────────────────────────────────────────────────┐
│              MariaDB avec CONNECT                      │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Table virtuelle CONNECT                         │  │
│  │  (pas de stockage local)                         │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
         ↓              ↓              ↓              ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  MySQL/PG    │ │  CSV/JSON    │ │  MongoDB     │ │  REST API    │
│  (distant)   │ │  (fichiers)  │ │  (NoSQL)     │ │  (HTTP)      │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

**Cas d'usage** :
- ✅ Intégration multi-sources (data warehouse)
- ✅ Accès bases externes (PostgreSQL, MongoDB)
- ✅ Lecture fichiers (CSV, JSON, XML)
- ✅ Consommation API REST
- ❌ Source primaire de données

**Exemple** :
```sql
-- Accéder à un fichier CSV externe
CREATE TABLE external_csv (
    id INT,
    name VARCHAR(100),
    value DECIMAL(10,2)
) ENGINE=CONNECT
TABLE_TYPE=CSV
FILE_NAME='/data/external.csv'
SEP_CHAR=','
QUOTED=1;

-- Requête comme table normale
SELECT * FROM external_csv WHERE value > 100;
```

**Sources supportées** :
- MySQL/MariaDB distant
- PostgreSQL
- Oracle
- MongoDB
- Fichiers : CSV, JSON, XML, INI
- REST API (JSON/XML)
- ODBC

### 6. Spider : Sharding horizontal

**Principe** : Partitionnement de données sur plusieurs serveurs.

```
Architecture Spider :
┌────────────────────────────────────────────────────────┐
│              MariaDB Spider (Coordinateur)             │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Table Spider (vue logique unifiée)              │  │
│  │  SELECT * FROM users WHERE ...                   │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
         ↓                  ↓                  ↓
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│   Shard 1      │  │   Shard 2      │  │   Shard 3      │
│   (InnoDB)     │  │   (InnoDB)     │  │   (InnoDB)     │
│   Users 1-1M   │  │   Users 1M-2M  │  │   Users 2M-3M  │
└────────────────┘  └────────────────┘  └────────────────┘
```

**Cas d'usage** :
- ✅ Scalabilité horizontale (10+ serveurs)
- ✅ Distribution géographique
- ✅ Isolation données (multi-tenant)
- ✅ Haute disponibilité
- ❌ Applications simples

**Exemple** :
```sql
-- Table Spider (coordinateur)
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(100),
    email VARCHAR(200)
) ENGINE=Spider
COMMENT='wrapper "mysql", srv "shard1", table "users"'
PARTITION BY RANGE (user_id) (
    PARTITION p0 VALUES LESS THAN (1000000)
        COMMENT='srv "shard1"',
    PARTITION p1 VALUES LESS THAN (2000000)
        COMMENT='srv "shard2"',
    PARTITION p2 VALUES LESS THAN MAXVALUE
        COMMENT='srv "shard3"'
);

-- Requête routée automatiquement vers bon shard
SELECT * FROM users WHERE user_id = 1500000;
-- → Envoyé à shard2 uniquement
```

**Avantages** :
- ✅ Scalabilité linéaire
- ✅ Transparence application
- ✅ Haute disponibilité (réplication par shard)

**Limitations** :
- ⚠️ Complexité configuration
- ⚠️ JOINs cross-shard lents
- ⚠️ Transactions distribuées complexes

### 7. FederatedX : Proxy base distante

**Principe** : Table locale = Proxy vers table distante.

```
Architecture FederatedX :
┌────────────────────────────────────────────────────────┐
│              MariaDB Local                             │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Table FederatedX (proxy)                        │  │
│  │  SELECT * FROM remote_orders;                    │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
                         ↓ MySQL Protocol
┌────────────────────────────────────────────────────────┐
│              MariaDB/MySQL Distant                     │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Table réelle (InnoDB)                           │  │
│  │  orders                                          │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Cas d'usage** :
- ✅ Accès ponctuel base distante
- ✅ Consolidation reporting
- ✅ Migration progressive
- ⚠️ Production (latence réseau)
- ❌ Haute performance

**Exemple** :
```sql
-- Créer lien vers table distante
CREATE TABLE remote_orders (
    order_id INT,
    customer_id INT,
    amount DECIMAL(10,2)
) ENGINE=FederatedX
CONNECTION='mysql://user:password@remote-server:3306/db/orders';

-- Requête transparente
SELECT * FROM remote_orders WHERE customer_id = 42;
-- Exécutée sur serveur distant
```

**Limitations** :
- ❌ Latence réseau (10-100 ms minimum)
- ❌ Dépendance réseau (SPOF)
- ❌ Pas de cache local
- ⚠️ Statut expérimental

---

## Comparaison et choix

### Matrice de décision rapide

| Besoin | Moteur recommandé | Moteur alternatif |
|--------|-------------------|-------------------|
| **Cache ultra-rapide** | Memory | Redis externe |
| **Compression maximale** | Archive | S3 + Glacier |
| **Hub réplication** | Blackhole | Proxy externe |
| **Export/import CSV** | CSV | LOAD DATA / SELECT INTO |
| **Fédération données** | CONNECT | ETL (Talend, Airflow) |
| **Sharding horizontal** | Spider | ProxySQL + partitioning |
| **Proxy distant** | FederatedX | Application-level proxy |

### Quand NE PAS utiliser un moteur spécialisé

```sql
-- ❌ MAUVAIS : Memory pour données critiques
CREATE TABLE customer_orders ENGINE=Memory;
-- Problème : Perte au redémarrage = perte commandes

-- ✅ BON : Memory pour cache temporaire
CREATE TABLE session_cache ENGINE=Memory;
-- OK : Reconstructible depuis source

-- ❌ MAUVAIS : Archive pour données consultées souvent
CREATE TABLE active_users ENGINE=Archive;
-- Problème : Lectures lentes, pas d'UPDATE

-- ✅ BON : Archive pour logs historiques
CREATE TABLE logs_2020 ENGINE=Archive;
-- OK : Accès rare, compression importante

-- ❌ MAUVAIS : CSV pour table applicative
CREATE TABLE products ENGINE=CSV;
-- Problème : Pas d'index, lent, corruptible

-- ✅ BON : CSV pour export ponctuel
CREATE TABLE export_temp ENGINE=CSV;
-- OK : Usage temporaire, éditable manuellement
```

### Architecture hybride typique

```
Application e-commerce moderne :
┌────────────────────────────────────────────────────────┐
│              Couche Application                        │
└────────────────────────────────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│                  MariaDB Server                        │
│                                                        │
│  ┌────────────────────────────────────────────────── ┐ │
│  │  InnoDB (OLTP principal)                         │  │
│  │  • orders, customers, products                   │  │
│  │  → Transactions, haute concurrence               │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Memory (Cache)                                  │  │
│  │  • user_sessions, rate_limiting                  │  │
│  │  → Accès ultra-rapide, volatile OK               │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  ColumnStore (Analytics)                         │  │
│  │  • sales_fact, customer_analytics                │  │
│  │  → Reporting BI, agrégations                     │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Archive (Logs)                                  │  │
│  │  • audit_trail, system_logs                      │  │
│  │  → Compliance, compression                       │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  S3 (Archivage froid)                            │  │
│  │  • orders_2020, orders_2021                      │  │
│  │  → Coût minimal, accès rare                      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  CONNECT (Intégration)                           │  │
│  │  • legacy_system_data, external_api              │  │
│  │  → Fédération sources externes                   │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

---

## Limitations générales

### Contraintes communes

| Contrainte | Memory | Archive | Blackhole | CSV | CONNECT | Spider | FederatedX |
|------------|--------|---------|-----------|-----|---------|--------|------------|
| **Transactions ACID** | ❌ | ❌ | ❌ | ❌ | ⚠️ | ⚠️ | ⚠️ |
| **Foreign Keys** | ❌ | ❌ | ❌ | ❌ | ❌ | ⚠️ | ❌ |
| **Triggers** | ✅ | ✅ | ✅ | ✅ | ⚠️ | ✅ | ⚠️ |
| **Full-Text** | ❌ | ❌ | ❌ | ❌ | ❌ | ⚠️ | ❌ |
| **Crash Recovery** | ❌ | ✅ | ✅ | ⚠️ | N/A | N/A | N/A |
| **Backup chaud** | ⚠️ | ✅ | ✅ | ⚠️ | N/A | Complexe | N/A |

### Performance relative (ordre de grandeur)

```
SELECT point-lookup (1 row) :
Memory      : 0.01 ms   ⭐⭐⭐⭐⭐
InnoDB      : 0.1 ms    ⭐⭐⭐⭐
Spider      : 0.5 ms    ⭐⭐⭐
CONNECT     : 5 ms      ⭐⭐
FederatedX  : 10 ms     ⭐⭐
CSV         : 50 ms     ⭐
Archive     : 100 ms    ⭐

INSERT (1000 rows batch) :
InnoDB      : 10 ms     ⭐⭐⭐⭐⭐
Memory      : 5 ms      ⭐⭐⭐⭐⭐
Blackhole   : 1 ms      ⭐⭐⭐⭐⭐
Archive     : 8 ms      ⭐⭐⭐⭐
CSV         : 100 ms    ⭐⭐
```

---

## Best practices

### 1. Toujours avoir un plan B

```sql
-- ✅ BON : Memory avec fallback InnoDB
CREATE TABLE sessions_memory ENGINE=Memory ...;
CREATE TABLE sessions_persistent ENGINE=InnoDB ...;

-- Au redémarrage : Reconstruire Memory depuis InnoDB
INSERT INTO sessions_memory
SELECT * FROM sessions_persistent
WHERE last_activity > NOW() - INTERVAL 1 HOUR;
```

### 2. Documenter l'usage de moteurs spécialisés

```sql
-- ✅ BON : Commentaires explicatifs
CREATE TABLE rate_limit_cache (
    ip_address VARCHAR(45) PRIMARY KEY,
    request_count INT,
    window_start TIMESTAMP
) ENGINE=Memory
COMMENT='Cache temporaire pour rate limiting.
         Perte acceptable au redémarrage.
         Reconstructible depuis Elasticsearch.';
```

### 3. Surveillance spécifique

```sql
-- Surveiller Memory : Taille vs max_heap_table_size
SELECT
    TABLE_NAME,
    (DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 AS size_mb,
    @@max_heap_table_size / 1024 / 1024 AS max_mb
FROM information_schema.TABLES
WHERE ENGINE = 'MEMORY';

-- Alerte si > 80% de max_heap_table_size
```

### 4. Tests de failover

```bash
# Tester comportement après crash simulé
# Pour tables Memory : Vérifier reconstruction

# 1. Kill MariaDB
kill -9 $(pidof mysqld)

# 2. Redémarrer
systemctl start mariadb

# 3. Vérifier tables Memory
mysql -e "SELECT COUNT(*) FROM sessions_memory"
-- Doit être 0 (vide après redémarrage)

# 4. Vérifier script de reconstruction
./rebuild_memory_tables.sh

# 5. Vérifier résultat
mysql -e "SELECT COUNT(*) FROM sessions_memory"
-- Doit être > 0 (reconstruit)
```

---

## ✅ Points clés à retenir

1. **Spécialisation** : Moteurs spécialisés = cas d'usage très spécifiques, pas usage général.

2. **Memory** : Ultra-rapide mais volatile. Cache seulement, jamais données critiques.

3. **Archive** : Compression maximale, INSERT-only. Logs historiques, audit trail.

4. **Blackhole** : /dev/null SQL. Réplication hub, benchmarks, filtrage.

5. **CSV** : Export/import fichiers plats. Intégration externe, pas production.

6. **CONNECT** : Fédération données hétérogènes. Data warehouse, intégration multi-sources.

7. **Spider** : Sharding horizontal. Scalabilité > 10 serveurs, complexe.

8. **FederatedX** : Proxy distant. Accès ponctuel, latence élevée, expérimental.

9. **Architecture hybride** : Combiner moteurs principaux + spécialisés = optimal.

10. **Limitations** : Tous ont contraintes importantes. Bien évaluer compromis.

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 Storage Engines Overview](https://mariadb.com/kb/en/storage-engines/)
- [📖 Memory Storage Engine](https://mariadb.com/kb/en/memory-storage-engine/)
- [📖 Archive Storage Engine](https://mariadb.com/kb/en/archive/)
- [📖 CONNECT Storage Engine](https://mariadb.com/kb/en/connect/)
- [📖 Spider Storage Engine](https://mariadb.com/kb/en/spider/)

### Guides pratiques

- [When to Use Specialized Storage Engines](https://mariadb.com/resources/blog/specialized-engines/)
- [CONNECT Use Cases](https://mariadb.com/kb/en/connect-use-cases/)
- [Spider Sharding Guide](https://mariadb.com/kb/en/spider-overview/)

---

## ➡️ Sections détaillées suivantes

Chaque moteur spécialisé sera détaillé dans les sous-sections suivantes :

- **7.10.1** : Memory - Tables en RAM
- **7.10.2** : Archive - Compression maximale
- **7.10.3** : Blackhole - Le /dev/null SQL
- **7.10.4** : CSV - Fichiers texte standards
- **7.10.5** : CONNECT - Fédération de données
- **7.10.6** : Spider - Sharding horizontal
- **7.10.7** : FederatedX - Proxy base distante

---

**📌 Mémo DBA** : "Moteurs spécialisés = outils chirurgicaux. Memory pour cache volatile, Archive pour logs compressés, CONNECT pour fédération, Spider pour sharding. Jamais en remplacement d'InnoDB pour OLTP général."

**🎯 Règle de décision** :
1. Besoin couvert par InnoDB/ColumnStore/S3/Vector ? → Utiliser ces moteurs (90% des cas)
2. Besoin très spécifique non couvert ? → Évaluer moteur spécialisé
3. Compromis acceptable (limitations) ? → Utiliser moteur spécialisé
4. Architecture hybride possible ? → Combiner moteurs

**⚠️ Attention** : Ne jamais utiliser un moteur spécialisé par défaut ou par méconnaissance d'InnoDB. Toujours justifier le choix par un besoin spécifique réel.

⏭️ [Memory : Tables en RAM](/07-moteurs-de-stockage/10.1-memory.md)

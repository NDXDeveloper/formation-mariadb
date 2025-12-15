🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20. Cas d'Usage et Architectures

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 12-16 heures  
> **Prérequis** : Chapitres 1-19, connaissances en architecture logicielle, notions de systèmes distribués

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- Distinguer les workloads OLTP et OLAP et choisir l'architecture adaptée
- Concevoir des architectures microservices avec MariaDB (database-per-service, shared database)
- Implémenter des solutions de data warehousing avec ColumnStore
- Architecturer des systèmes multi-tenant sécurisés et performants
- Déployer MariaDB en géo-distribution et environnements hybrid cloud
- Intégrer MariaDB dans des architectures Event-Driven avec CDC, Kafka et Debezium
- 🆕 Exploiter MariaDB Vector pour les use cases IA : RAG, recherche sémantique, recommandations
- 🆕 Utiliser MariaDB MCP Server et les frameworks IA (LangChain, LlamaIndex)
- Analyser des études de cas réelles et justifier des décisions architecturales

---

## Introduction

Ce chapitre représente l'aboutissement de votre parcours de formation MariaDB 11.8 LTS. Après avoir maîtrisé les fondamentaux SQL, les mécanismes transactionnels, les moteurs de stockage, la réplication, la haute disponibilité et les pratiques DevOps, il est temps de synthétiser ces connaissances pour concevoir des architectures complètes répondant à des besoins métier réels.

L'architecture d'une base de données n'est jamais un choix isolé. Elle s'inscrit dans un écosystème applicatif, répond à des contraintes de performance, de disponibilité, de coût et d'évolutivité. MariaDB 11.8 LTS apporte des innovations majeures — notamment MariaDB Vector pour l'IA — qui ouvrent de nouveaux horizons architecturaux.

### Pourquoi ce chapitre est essentiel

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ÉVOLUTION DES ARCHITECTURES DATA                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Monolithique        Microservices        Event-Driven        IA/ML    │
│   (2000-2010)         (2010-2018)          (2018-2023)        (2023+)   │
│                                                                         │
│   ┌─────────┐        ┌───┐ ┌───┐ ┌───┐    ┌───┐              ┌───────┐  │
│   │   App   │        │ S │ │ S │ │ S │    │CDC│──►Kafka      │Vector │  │
│   │  + DB   │        │ 1 │ │ 2 │ │ 3 │    └───┘              │Search │  │
│   └────┬────┘        └─┬─┘ └─┬─┘ └─┬─┘      │                │+ RAG  │  │
│        │               │     │     │        ▼                └───────┘  │
│   ┌────▼────┐        ┌─▼─┐ ┌─▼─┐ ┌─▼─┐  ┌──────┐                        │
│   │Single DB│        │DB1│ │DB2│ │DB3│  │Stream│                        │
│   └─────────┘        └───┘ └───┘ └───┘  │Process│                       │
│                                          └──────┘                       │
│                                                                         │
│   MariaDB accompagne cette évolution avec :                             │
│   • InnoDB pour OLTP haute performance                                  │
│   • ColumnStore pour OLAP et analytics                                  │
│   • Galera pour la haute disponibilité                                  │
│   • 🆕 Vector pour l'IA et la recherche sémantique                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Structure du chapitre

Ce chapitre est organisé en quatre grandes thématiques :

| Thématique | Sections | Focus |
|------------|----------|-------|
| **Patterns fondamentaux** | 20.1-20.4 | OLTP/OLAP, microservices, multi-tenant |
| **Distribution et cloud** | 20.5-20.7 | Géo-distribution, hybrid cloud, scaling |
| **Event-Driven & streaming** | 20.8 | CDC, Kafka, Debezium |
| **IA et nouvelles frontières** | 20.9-20.11 | Vector, RAG, MCP, LangChain 🆕 |
| **Retours d'expérience** | 20.12 | Études de cas réelles |

---

## Vue d'ensemble des architectures MariaDB

### Le spectre des workloads

MariaDB 11.8 LTS couvre l'ensemble du spectre des besoins data modernes grâce à son architecture pluggable de moteurs de stockage :

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         SPECTRE DES WORKLOADS                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  OLTP                    HTAP                      OLAP                    │
│  (Transactionnel)        (Hybride)                 (Analytique)            │
│                                                                            │
│  ◄─────────────────────────────────────────────────────────────────────►   │
│                                                                            │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────┐      │
│  │   InnoDB     │    │ InnoDB + Replica │    │    ColumnStore       │      │
│  │              │    │   ColumnStore    │    │                      │      │
│  │ • Transactions    │                  │    │ • Agrégations        │      │
│  │ • Row-level  │    │ • Read replicas  │    │ • Colonnes           │      │
│  │ • ACID       │    │ • Analytics      │    │ • Compression        │      │
│  │ • < 10ms     │    │ • Real-time      │    │ • Pétaoctets         │      │
│  └──────────────┘    └──────────────────┘    └──────────────────────┘      │
│                                                                            │
│  + MariaDB Vector 🆕 : Recherche vectorielle transversale à tous les        │
│    workloads pour l'intégration IA/ML                                      │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Critères de choix architectural

Avant de plonger dans les patterns, voici les critères fondamentaux qui guideront vos décisions :

| Critère | Questions clés | Impact sur l'architecture |
|---------|----------------|---------------------------|
| **Consistance** | Transactions ACID critiques ? Eventual consistency acceptable ? | Choix InnoDB vs réplication asynchrone |
| **Disponibilité** | SLA requis (99.9%, 99.99%) ? RTO/RPO ? | Galera, MaxScale, multi-région |
| **Partition tolerance** | Géo-distribution ? Latence acceptable ? | Réplication asynchrone, sharding |
| **Latence** | P99 < 10ms ? < 100ms ? < 1s ? | Index, cache, architecture |
| **Throughput** | QPS attendus ? Ratio lecture/écriture ? | Read replicas, connection pooling |
| **Volume** | Go, To, Po ? Croissance ? | Partitionnement, sharding, archivage |
| **Coût** | Budget infrastructure ? Licences ? | Community vs Enterprise, cloud |

💡 **Conseil** : Le théorème CAP s'applique. Vous ne pouvez optimiser que deux des trois propriétés (Consistency, Availability, Partition tolerance) simultanément. MariaDB avec Galera privilégie CP, tandis que la réplication asynchrone permet AP.

---

## 20.1 OLTP vs OLAP : Deux mondes, deux architectures

### Caractéristiques distinctives

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        OLTP vs OLAP                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  OLTP (Online Transaction Processing)    OLAP (Online Analytical Processing) │
│  ═══════════════════════════════════     ════════════════════════════════    │
│                                                                              │
│  ┌─────────────────────────────────┐     ┌─────────────────────────────────┐ │
│  │  Opérations                     │     │  Opérations                     │ │
│  │  • INSERT, UPDATE, DELETE       │     │  • SELECT avec agrégations      │ │
│  │  • Transactions courtes         │     │  • Requêtes complexes           │ │
│  │  • Accès par clé primaire       │     │  • Full table scans             │ │
│  │  • Haute concurrence            │     │  • Faible concurrence           │ │
│  └─────────────────────────────────┘     └─────────────────────────────────┘ │
│                                                                              │
│  ┌─────────────────────────────────┐     ┌─────────────────────────────────┐ │
│  │  Données                        │     │  Données                        │ │
│  │  • État actuel                  │     │  • Historique                   │ │
│  │  • Normalisées (3NF+)           │     │  • Dénormalisées (star schema)  │ │
│  │  • Go à To                      │     │  • To à Po                      │ │
│  │  • Mises à jour fréquentes      │     │  • Append-only / ETL batch      │ │
│  └─────────────────────────────────┘     └─────────────────────────────────┘ │
│                                                                              │
│  ┌─────────────────────────────────┐     ┌─────────────────────────────────┐ │
│  │  Moteur MariaDB                 │     │  Moteur MariaDB                 │ │
│  │  • InnoDB (défaut)              │     │  • ColumnStore                  │ │
│  │  • Row-based storage            │     │  • Columnar storage             │ │
│  │  • B-Tree indexes               │     │  • Compression native           │ │
│  │  • Buffer Pool optimisé         │     │  • Vectorized execution         │ │
│  └─────────────────────────────────┘     └─────────────────────────────────┘ │
│                                                                              │
│  Métriques typiques :                    Métriques typiques :                │
│  • Latence : < 10ms P99                  • Latence : secondes à minutes      │
│  • QPS : 10K - 100K+                     • QPS : 1 - 100                     │
│  • Transactions/sec : 1K - 50K           • Requêtes concurrentes : < 50      │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Configuration OLTP optimisée

```ini
# /etc/mysql/mariadb.conf.d/oltp.cnf
[mysqld]
# === Moteur de stockage ===
default_storage_engine = InnoDB

# === Buffer Pool (70-80% RAM disponible) ===
innodb_buffer_pool_size = 24G           # Serveur 32GB RAM
innodb_buffer_pool_instances = 8        # 1 instance par Go jusqu'à 8
innodb_buffer_pool_chunk_size = 1G

# === Performance I/O ===
innodb_io_capacity = 2000               # SSD standard
innodb_io_capacity_max = 4000
innodb_flush_method = O_DIRECT
innodb_flush_log_at_trx_commit = 1      # Durabilité ACID complète

# === Concurrence ===
innodb_thread_concurrency = 0           # Auto-détection
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# === Logs ===
innodb_log_file_size = 2G
innodb_log_buffer_size = 64M

# === 🆕 MariaDB 11.8 : Optimisations SSD ===
# Le cost-based optimizer prend en compte les SSD
optimizer_disk_read_ratio = 0.0002      # SSD NVMe rapide
```

### Configuration OLAP avec ColumnStore

```ini
# /etc/mysql/mariadb.conf.d/olap.cnf
[mysqld]
default_storage_engine = Columnstore

# === ColumnStore spécifique ===
columnstore_use_import_for_batchinsert = ON
columnstore_string_scan_threshold = 10

# === Mémoire pour les agrégations ===
max_heap_table_size = 4G
tmp_table_size = 4G
join_buffer_size = 256M
sort_buffer_size = 256M

# === Parallélisme ===
columnstore_diskjoin_smallsidelimit = 1073741824
columnstore_diskjoin_largesidelimit = 2147483648
```

### Architecture HTAP (Hybrid)

Pour les besoins mixtes, MariaDB permet une architecture HTAP (Hybrid Transactional/Analytical Processing) :

```
┌────────────────────────────────────────────────────────────────────────┐
│                     ARCHITECTURE HTAP MARIADB                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Applications                                                          │
│       │                                                                │
│       ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         MaxScale                                │   │
│  │                    (Read/Write Split)                           │   │
│  └──────────────┬──────────────────────────────────┬───────────────┘   │
│                 │                                  │                   │
│        Écritures│                         Lectures │                   │
│                 ▼                                  ▼                   │
│  ┌──────────────────────┐          ┌──────────────────────────────┐    │
│  │    Primary (InnoDB)  │          │   Replica Analytics          │    │
│  │                      │          │   (ColumnStore)              │    │
│  │  • Transactions      │  ──────► │                              │    │
│  │  • CRUD operations   │  Repli-  │  • Dashboards                │    │
│  │  • Row-based         │  cation  │  • Rapports                  │    │
│  └──────────────────────┘          │  • BI / Analytics            │    │
│                                    └──────────────────────────────┘    │
│                                                                        │
│  💡 La réplication cross-engine (InnoDB → ColumnStore) est supportée   │
│     depuis MariaDB 10.5                                                │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 20.2 Architectures Microservices

Les architectures microservices posent des défis uniques pour la gestion des données. Deux patterns principaux émergent :

### Pattern 1 : Database per Service

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATABASE PER SERVICE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │   Service   │   │   Service   │   │   Service   │   │   Service   │  │
│  │   Orders    │   │   Users     │   │  Inventory  │   │  Payments   │  │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  │
│         │                 │                 │                 │         │
│         ▼                 ▼                 ▼                 ▼         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │  MariaDB    │   │  MariaDB    │   │  MariaDB    │   │  MariaDB    │  │
│  │  orders_db  │   │  users_db   │   │ inventory_db│   │ payments_db │  │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘  │
│                                                                         │
│  ✅ Avantages                          ❌ Inconvénients                  │
│  • Isolation totale                    • Pas de jointures cross-service │
│  • Scaling indépendant                 • Transactions distribuées       │
│  • Déploiement autonome                • Duplication de données         │
│  • Choix techno par service            • Complexité opérationnelle      │
│  • Failure isolation                   • Coût infrastructure            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Implémentation avec mariadb-operator (Kubernetes)

```yaml
# orders-mariadb.yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: MariaDB
metadata:
  name: orders-db
  namespace: orders-service
spec:
  rootPasswordSecretKeyRef:
    name: orders-db-root
    key: password
  
  image: mariadb:11.8
  
  storage:
    size: 50Gi
    storageClassName: fast-ssd
  
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  
  # Haute disponibilité avec Galera
  galera:
    enabled: true
    replicas: 3
  
  # Configuration spécifique au service
  myCnf: |
    [mysqld]
    max_connections = 200
    innodb_buffer_pool_size = 2G
    
  # Base de données dédiée
  databases:
    - name: orders
      characterSet: utf8mb4
      collate: utf8mb4_unicode_ci
  
  # Utilisateur applicatif
  users:
    - name: orders_app
      passwordSecretKeyRef:
        name: orders-db-app
        key: password
      grants:
        - database: orders
          privileges:
            - SELECT
            - INSERT
            - UPDATE
            - DELETE
```

#### Gestion des transactions distribuées

Sans transactions ACID cross-services, utilisez le pattern Saga :

```sql
-- Service Orders : Créer une commande en état "pending"
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    status ENUM('pending', 'confirmed', 'cancelled', 'completed') DEFAULT 'pending',
    total_amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    saga_state JSON,  -- État de la saga pour compensation
    INDEX idx_status (status),
    INDEX idx_user (user_id)
) ENGINE=InnoDB;

-- Outbox pattern pour événements fiables
CREATE TABLE order_outbox (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL,
    INDEX idx_unprocessed (processed_at, created_at)
) ENGINE=InnoDB;

-- Transaction locale avec outbox
START TRANSACTION;

INSERT INTO orders (user_id, total_amount, saga_state)
VALUES (123, 99.99, '{"step": "order_created"}');

INSERT INTO order_outbox (aggregate_type, aggregate_id, event_type, payload)
VALUES ('Order', LAST_INSERT_ID(), 'OrderCreated', 
        JSON_OBJECT('order_id', LAST_INSERT_ID(), 'user_id', 123, 'amount', 99.99));

COMMIT;
```

### Pattern 2 : Shared Database

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SHARED DATABASE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │   Service   │   │   Service   │   │   Service   │   │   Service   │  │
│  │   Orders    │   │   Users     │   │  Inventory  │   │  Payments   │  │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  │
│         │                 │                 │                 │         │
│         │                 │                 │                 │         │
│         └─────────────────┼─────────────────┼─────────────────┘         │
│                           │                 │                           │
│                           ▼                 ▼                           │
│                    ┌─────────────────────────────┐                      │
│                    │     MariaDB Cluster         │                      │
│                    │  ┌─────────────────────┐    │                      │
│                    │  │ Schema: orders      │    │                      │
│                    │  │ Schema: users       │    │                      │
│                    │  │ Schema: inventory   │    │                      │
│                    │  │ Schema: payments    │    │                      │
│                    │  └─────────────────────┘    │                      │
│                    └─────────────────────────────┘                      │
│                                                                         │
│  ✅ Avantages                          ❌ Inconvénients                 │
│  • Jointures possibles                 • Couplage fort                  │
│  • Transactions ACID                   • Scaling limité                 │
│  • Simplicité opérationnelle           • Point de contention            │
│  • Coût réduit                         • Déploiement couplé             │
│  • Cohérence des données               • Risque de dégradation globale  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Isolation par schémas et utilisateurs

```sql
-- Création des schémas isolés
CREATE DATABASE orders_schema CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE users_schema CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE inventory_schema CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Utilisateur dédié par service avec privilèges minimaux
CREATE USER 'orders_svc'@'10.0.%' IDENTIFIED BY 'secure_password_1';
GRANT SELECT, INSERT, UPDATE, DELETE ON orders_schema.* TO 'orders_svc'@'10.0.%';

CREATE USER 'users_svc'@'10.0.%' IDENTIFIED BY 'secure_password_2';
GRANT SELECT, INSERT, UPDATE, DELETE ON users_schema.* TO 'users_svc'@'10.0.%';

-- Accès en lecture cross-schema si nécessaire (avec prudence)
GRANT SELECT ON users_schema.users TO 'orders_svc'@'10.0.%';

-- 🆕 MariaDB 11.8 : Privilèges granulaires
GRANT SELECT (id, email, name) ON users_schema.users TO 'orders_svc'@'10.0.%';
```

### Décision : Database per Service vs Shared Database

| Critère | Database per Service | Shared Database |
|---------|---------------------|-----------------|
| Équipes | Autonomes, nombreuses | Petite équipe, coordination facile |
| Scaling | Besoins très différents par service | Besoins similaires |
| Transactions | Saga acceptable | ACID cross-tables requis |
| Données | Faible couplage | Fort couplage référentiel |
| Budget | Conséquent | Limité |
| Maturité DevOps | Élevée | En développement |

💡 **Conseil** : Commencez par Shared Database avec isolation par schémas, puis migrez vers Database per Service quand le besoin d'autonomie le justifie.

---

## 20.3 Data Warehousing avec ColumnStore

### Architecture ColumnStore

MariaDB ColumnStore est optimisé pour l'analytique à grande échelle :

```
┌────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE COLUMNSTORE                            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      User Module (UM)                           │   │
│  │  • Parsing SQL                                                  │   │
│  │  • Query planning                                               │   │
│  │  • Result aggregation                                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                         │
│          ┌───────────────────┼───────────────────┐                     │
│          ▼                   ▼                   ▼                     │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐             │
│  │ Performance   │   │ Performance   │   │ Performance   │             │
│  │ Module (PM1)  │   │ Module (PM2)  │   │ Module (PM3)  │             │
│  │               │   │               │   │               │             │
│  │ ┌───────────┐ │   │ ┌───────────┐ │   │ ┌───────────┐ │             │
│  │ │ Extent 1  │ │   │ │ Extent 2  │ │   │ │ Extent 3  │ │             │
│  │ │ Extent 4  │ │   │ │ Extent 5  │ │   │ │ Extent 6  │ │             │
│  │ │ ...       │ │   │ │ ...       │ │   │ │ ...       │ │             │
│  │ └───────────┘ │   │ └───────────┘ │   │ └───────────┘ │             │
│  └───────────────┘   └───────────────┘   └───────────────┘             │
│                                                                        │
│  Caractéristiques :                                                    │
│  • Stockage par colonne (vs par ligne)                                 │
│  • Compression native (5-10x réduction)                                │
│  • Exécution vectorisée                                                │
│  • Scaling horizontal sur les PM                                       │
│  • Pas de FK ni transactions ACID                                      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Modélisation Star Schema

```sql
-- Dimension : Temps
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week TINYINT,
    day_name VARCHAR(10),
    month TINYINT,
    month_name VARCHAR(10),
    quarter TINYINT,
    year SMALLINT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
) ENGINE=Columnstore;

-- Dimension : Produits
CREATE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(200),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    supplier VARCHAR(200),
    unit_cost DECIMAL(10,2),
    unit_price DECIMAL(10,2)
) ENGINE=Columnstore;

-- Dimension : Clients
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    email VARCHAR(200),
    segment VARCHAR(50),        -- B2B, B2C, Enterprise
    region VARCHAR(100),
    country VARCHAR(100),
    city VARCHAR(100),
    registration_date DATE
) ENGINE=Columnstore;

-- Table de faits : Ventes
CREATE TABLE fact_sales (
    sale_id BIGINT,
    date_key INT,
    product_key INT,
    customer_key INT,
    store_key INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    discount_percent DECIMAL(5,2),
    total_amount DECIMAL(12,2),
    cost_amount DECIMAL(12,2),
    profit_amount DECIMAL(12,2)
) ENGINE=Columnstore;

-- Requête analytique typique
SELECT 
    d.year,
    d.quarter,
    p.category,
    c.segment,
    SUM(f.quantity) AS total_quantity,
    SUM(f.total_amount) AS revenue,
    SUM(f.profit_amount) AS profit,
    SUM(f.profit_amount) / NULLIF(SUM(f.total_amount), 0) * 100 AS profit_margin
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_customer c ON f.customer_key = c.customer_key
WHERE d.year >= 2024
GROUP BY d.year, d.quarter, p.category, c.segment
WITH ROLLUP
ORDER BY d.year, d.quarter, revenue DESC;
```

### Pipeline ETL vers ColumnStore

```sql
-- Chargement bulk optimisé via cpimport
-- bash: cpimport -s ',' -E '"' analytics fact_sales /data/sales_2024.csv

-- Ou via SQL avec INSERT batch
SET columnstore_use_import_for_batchinsert = ON;

INSERT INTO fact_sales 
SELECT 
    s.id,
    dd.date_key,
    dp.product_key,
    dc.customer_key,
    ds.store_key,
    s.quantity,
    s.unit_price,
    s.discount_percent,
    s.quantity * s.unit_price * (1 - s.discount_percent/100),
    s.quantity * dp.unit_cost,
    s.quantity * (s.unit_price * (1 - s.discount_percent/100) - dp.unit_cost)
FROM staging_sales s
JOIN dim_date dd ON DATE(s.sale_date) = dd.full_date
JOIN dim_product dp ON s.product_id = dp.product_id
JOIN dim_customer dc ON s.customer_id = dc.customer_id
JOIN dim_store ds ON s.store_id = ds.store_id
WHERE s.processed = 0;
```

---

## 20.4 Architectures Multi-Tenant

Les applications SaaS nécessitent une stratégie d'isolation des données par tenant :

### Comparaison des approches

```
┌────────────────────────────────────────────────────────────────────────┐
│                   STRATÉGIES MULTI-TENANT                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. DATABASE PER TENANT                                                │
│  ════════════════════════                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                                   │
│  │tenant_a │ │tenant_b │ │tenant_c │  • Isolation maximale             │
│  │  _db    │ │  _db    │ │  _db    │  • Backup/restore par tenant      │
│  └─────────┘ └─────────┘ └─────────┘  • Scaling indépendant            │
│                                        • Coût élevé (> 100 tenants)    │
│                                                                        │
│  2. SCHEMA PER TENANT                                                  │
│  ═══════════════════════                                               │
│  ┌─────────────────────────────────┐                                   │
│  │ Database: saas_app              │  • Bonne isolation                │
│  │ ┌─────────┐ ┌─────────┐         │  • Migrations plus simples        │
│  │ │schema_a │ │schema_b │ ...     │  • Limite ~1000 tenants           │
│  │ └─────────┘ └─────────┘         │  • Coût modéré                    │
│  └─────────────────────────────────┘                                   │
│                                                                        │
│  3. SHARED SCHEMA (Discriminateur)                                     │
│  ════════════════════════════════════                                  │
│  ┌─────────────────────────────────┐                                   │
│  │ Table: users                    │  • Scaling illimité               │
│  │ ├── tenant_id (discriminateur)  │  • Complexité applicative         │
│  │ ├── id                          │  • Risque de fuite de données     │
│  │ ├── name                        │  • Maintenance simplifiée         │
│  │ └── ...                         │  • Coût minimal                   │
│  └─────────────────────────────────┘                                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Implémentation Shared Schema sécurisée

```sql
-- Structure avec tenant_id systématique
CREATE TABLE tenants (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    subdomain VARCHAR(50) UNIQUE NOT NULL,
    plan ENUM('free', 'starter', 'pro', 'enterprise') DEFAULT 'free',
    settings JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_subdomain (subdomain)
) ENGINE=InnoDB;

CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role ENUM('admin', 'user', 'viewer') DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_tenant_email (tenant_id, email),
    INDEX idx_tenant (tenant_id),
    FOREIGN KEY (tenant_id) REFERENCES tenants(id)
) ENGINE=InnoDB;

-- Toutes les tables métier incluent tenant_id
CREATE TABLE projects (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    name VARCHAR(200) NOT NULL,
    owner_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_tenant (tenant_id),
    INDEX idx_tenant_owner (tenant_id, owner_id),
    FOREIGN KEY (tenant_id) REFERENCES tenants(id),
    FOREIGN KEY (owner_id) REFERENCES users(id)
) ENGINE=InnoDB;
```

#### Row-Level Security avec Vues

```sql
-- Vue sécurisée par tenant (le tenant_id vient de la session)
CREATE VIEW v_users AS
SELECT id, email, role, created_at
FROM users
WHERE tenant_id = @current_tenant_id;

CREATE VIEW v_projects AS
SELECT id, name, owner_id, created_at
FROM projects
WHERE tenant_id = @current_tenant_id;

-- Procédure de connexion définissant le contexte
DELIMITER //
CREATE PROCEDURE set_tenant_context(IN p_tenant_id INT)
BEGIN
    SET @current_tenant_id = p_tenant_id;
END //
DELIMITER ;

-- Utilisation dans l'application
CALL set_tenant_context(42);
SELECT * FROM v_users;  -- Ne retourne que les users du tenant 42
```

⚠️ **Attention** : Cette approche nécessite une rigueur applicative. Chaque requête doit passer par les vues ou inclure le filtre tenant_id. Utilisez un ORM configuré pour ajouter automatiquement ce filtre.

---

## 20.5 Géo-Distribution

### Architecture multi-région

```
┌────────────────────────────────────────────────────────────────────────┐
│                 GÉODISTRIBUTION MARIADB                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────┐         ┌─────────────────────┐               │
│  │     RÉGION EU       │         │     RÉGION US       │               │
│  │                     │         │                     │               │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │               │
│  │  │   Primary     │  │◄───────►│  │   Primary     │  │               │
│  │  │  (Galera x3)  │  │  WAN    │  │  (Galera x3)  │  │               │
│  │  └───────────────┘  │  Async  │  └───────────────┘  │               │
│  │         │           │         │         │           │               │
│  │  ┌──────▼──────┐    │         │  ┌──────▼──────┐    │               │
│  │  │  MaxScale   │    │         │  │  MaxScale   │    │               │
│  │  └──────┬──────┘    │         │  └──────┬──────┘    │               │
│  │         │           │         │         │           │               │
│  │     Apps EU         │         │     Apps US         │               │
│  └─────────────────────┘         └─────────────────────┘               │
│                                                                        │
│  Modèles de réplication :                                              │
│  • Active-Passive : Écritures EU → Réplication async vers US           │
│  • Active-Active : Écritures locales + résolution de conflits          │
│  • Galera intra-région + Async inter-région (recommandé)               │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Configuration réplication cross-région

```ini
# Primary EU - my.cnf
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
gtid_domain_id = 1
gtid_strict_mode = ON

# Réplication asynchrone vers US (accepte la latence)
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

# Replica US - my.cnf  
[mysqld]
server_id = 100
gtid_domain_id = 2
log_slave_updates = ON
read_only = ON

# Tolérance à la latence WAN
slave_net_timeout = 120
```

```sql
-- Sur le replica US
CHANGE MASTER TO
    MASTER_HOST = 'eu-primary.example.com',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'secure_repl_pass',
    MASTER_USE_GTID = slave_pos,
    MASTER_SSL = 1,
    MASTER_SSL_CA = '/etc/mysql/ssl/ca.pem',
    MASTER_SSL_CERT = '/etc/mysql/ssl/client-cert.pem',
    MASTER_SSL_KEY = '/etc/mysql/ssl/client-key.pem';

START SLAVE;
```

---

## 20.6 Hybrid Cloud et Multi-Cloud

### Stratégies de déploiement

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURES HYBRID/MULTI-CLOUD                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  HYBRID CLOUD                          MULTI-CLOUD                      │
│  ════════════                          ═══════════                      │
│                                                                         │
│  ┌─────────────────┐                   ┌─────────────────┐              │
│  │   On-Premise    │                   │      AWS        │              │
│  │  (Production)   │                   │  (Production)   │              │
│  │  ┌───────────┐  │                   │  ┌───────────┐  │              │
│  │  │ MariaDB   │  │                   │  │ MariaDB   │  │              │
│  │  │ Primary   │◄─┼─────┐             │  │ Primary   │  │              │
│  │  └───────────┘  │     │             │  └─────┬─────┘  │              │
│  └─────────────────┘     │             └────────┼────────┘              │
│          │               │                      │                       │
│          │               │                      │ Réplication           │
│  ┌───────▼───────────┐   │             ┌────────▼────────┐              │
│  │      Cloud        │   │             │      GCP        │              │
│  │   (DR/Burst)      │   │             │   (DR/Read)     │              │
│  │  ┌───────────┐    │   │             │  ┌───────────┐  │              │
│  │  │ MariaDB   │◄───┼───┘             │  │ MariaDB   │  │              │
│  │  │ Replica   │    │                 │  │ Replica   │  │              │
│  │  └───────────┘    │                 │  └───────────┘  │              │
│  └───────────────────┘                 └─────────────────┘              │
│                                                                         │
│  Use cases :                           Use cases :                      │
│  • Données sensibles on-prem           • Éviter le vendor lock-in       │
│  • Burst vers cloud                    • Résilience géographique        │
│  • Migration progressive               • Best-of-breed services         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Déploiement Terraform multi-cloud

```hcl
# main.tf - Déploiement MariaDB multi-cloud
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
}

# Primary sur AWS
module "mariadb_primary_aws" {
  source = "./modules/mariadb-aws"
  
  instance_type    = "r6g.xlarge"
  storage_size     = 500
  storage_type     = "gp3"
  multi_az         = true
  
  vpc_id           = aws_vpc.main.id
  subnet_ids       = aws_subnet.private[*].id
  
  mariadb_version  = "11.8"
  
  tags = {
    Environment = "production"
    Role        = "primary"
  }
}

# Replica DR sur GCP
module "mariadb_replica_gcp" {
  source = "./modules/mariadb-gcp"
  
  machine_type     = "n2-highmem-4"
  disk_size        = 500
  disk_type        = "pd-ssd"
  
  network          = google_compute_network.main.id
  subnetwork       = google_compute_subnetwork.private.id
  
  mariadb_version  = "11.8"
  
  # Configuration réplication depuis AWS
  replication_source = {
    host     = module.mariadb_primary_aws.endpoint
    user     = "repl_user"
    password = var.replication_password
    ssl      = true
  }
  
  labels = {
    environment = "production"
    role        = "replica"
  }
}
```

---

## 20.7 Scaling : Vertical vs Horizontal

### Stratégies de scaling

| Approche | Quand l'utiliser | Limites | Avec MariaDB |
|----------|------------------|---------|--------------|
| **Vertical** | Buffer pool < RAM, CPU < 70% | Coût exponentiel, limite hardware | Augmenter RAM, CPU, SSD NVMe |
| **Read replicas** | Ratio lecture > 80% | Lag de réplication | MaxScale + replicas |
| **Sharding** | Volume > capacité single node | Complexité applicative | Spider, application-level |
| **Galera** | HA + write scaling modéré | Latence inter-nœuds | 3-5 nœuds optimaux |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STRATÉGIES DE SCALING                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SCALING VERTICAL              SCALING HORIZONTAL                       │
│  ═════════════════            ══════════════════                        │
│                                                                         │
│  ┌─────────────────┐          ┌─────┐ ┌─────┐ ┌─────┐                   │
│  │ ████████████    │          │     │ │     │ │     │                   │
│  │ ████████████    │          │ DB1 │ │ DB2 │ │ DB3 │                   │
│  │ ████████████    │          │     │ │     │ │     │                   │
│  │ ████████████    │          └─────┘ └─────┘ └─────┘                   │
│  │ Plus gros       │                                                    │
│  │ serveur         │          Répartition de charge                     │
│  └─────────────────┘                                                    │
│                                                                         │
│  ✅ Simple                    ✅ Scaling quasi-illimité                 │
│  ✅ Pas de changement app     ✅ Haute disponibilité                    │
│  ❌ Limite physique           ❌ Complexité opérationnelle              │
│  ❌ Coût non-linéaire         ❌ Transactions distribuées               │
│                                                                         │
│  APPROCHE RECOMMANDÉE : Vertical d'abord, puis horizontal               │
│  1. Optimiser les requêtes et index                                     │
│  2. Augmenter RAM (buffer pool)                                         │
│  3. SSD NVMe pour I/O                                                   │
│  4. Read replicas pour lectures                                         │
│  5. Sharding si vraiment nécessaire                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 20.8 Architectures Event-Driven

### Change Data Capture (CDC)

Le CDC permet de capturer les changements de données en temps réel pour alimenter d'autres systèmes :

```
┌────────────────────────────────────────────────────────────────────────┐
│                    CDC AVEC MARIADB ET DEBEZIUM                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                          MariaDB                                │   │
│  │                                                                 │   │
│  │  ┌─────────┐        ┌─────────────┐        ┌─────────┐          │   │
│  │  │ Tables  │───────►│  Binlog     │───────►│ GTID    │          │   │
│  │  │ InnoDB  │ INSERT │  (ROW fmt)  │        │ Position│          │   │
│  │  │         │ UPDATE │             │        │         │          │   │
│  │  │         │ DELETE │             │        │         │          │   │
│  │  └─────────┘        └──────┬──────┘        └─────────┘          │   │
│  └────────────────────────────┼────────────────────────────────────┘   │
│                               │                                        │
│                               ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Debezium Connector                           │  │
│  │                                                                  │  │
│  │  • Lit le binlog en temps réel                                   │  │
│  │  • Convertit en événements JSON/Avro                             │  │
│  │  • Gère les positions (offsets)                                  │  │
│  │  • Exactly-once semantics                                        │  │
│  └───────────────────────────┬──────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      Apache Kafka                                │  │
│  │                                                                  │  │
│  │  Topic: dbserver1.inventory.orders                               │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │ {"op":"c","after":{"id":1,"customer":"Alice","total":99}}  │  │  │
│  │  │ {"op":"u","before":{...},"after":{"id":1,"total":149}}     │  │  │
│  │  │ {"op":"d","before":{"id":2,...}}                           │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────┬──────────────────────────────────────┘  │
│                              │                                         │
│          ┌───────────────────┼───────────────────┐                     │
│          ▼                   ▼                   ▼                     │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐             │
│  │ Elasticsearch │   │  Data Lake    │   │   Cache       │             │
│  │ (Search)      │   │  (Analytics)  │   │   (Redis)     │             │
│  └───────────────┘   └───────────────┘   └───────────────┘             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Configuration Debezium pour MariaDB

```json
{
  "name": "mariadb-connector",
  "config": {
    "connector.class": "io.debezium.connector.mariadb.MariaDbConnector",
    "database.hostname": "mariadb.example.com",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "${secrets:mariadb/debezium:password}",
    "database.server.id": "184054",
    "topic.prefix": "dbserver1",
    "database.include.list": "inventory,orders",
    "table.include.list": "inventory.products,orders.orders,orders.order_items",
    
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.inventory",
    
    "include.schema.changes": "true",
    "gtid.source.includes": "1-1-1,2-2-1",
    
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$3-events",
    
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "true"
  }
}
```

### Préparation MariaDB pour CDC

```sql
-- Utilisateur dédié CDC avec privilèges minimaux
CREATE USER 'debezium'@'%' IDENTIFIED BY 'secure_cdc_password';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT 
    ON *.* TO 'debezium'@'%';
GRANT SELECT ON inventory.* TO 'debezium'@'%';
GRANT SELECT ON orders.* TO 'debezium'@'%';

-- Vérifier la configuration binlog
SHOW VARIABLES LIKE 'log_bin';           -- ON
SHOW VARIABLES LIKE 'binlog_format';     -- ROW
SHOW VARIABLES LIKE 'binlog_row_image';  -- FULL
SHOW VARIABLES LIKE 'gtid_mode';         -- ON (recommandé)
```

---

## 20.9 Use Cases IA : RAG et Recherche Vectorielle 🆕

MariaDB 11.8 LTS introduit **MariaDB Vector**, permettant d'intégrer nativement les workloads IA/ML :

### Architecture RAG (Retrieval-Augmented Generation)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE RAG AVEC MARIADB VECTOR                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐      │
│  │                        INDEXATION                              │      │
│  └────────────────────────────────────────────────────────────────┘      │
│                                                                          │
│  Documents     Chunking      Embedding         Stockage                  │
│  ┌──────┐     ┌──────┐      ┌──────────┐      ┌────────────────┐         │
│  │ PDF  │     │Chunk1│      │ OpenAI   │      │  MariaDB 11.8  │         │
│  │ HTML │────►│Chunk2│─────►│ Claude   │─────►│                │         │
│  │ TXT  │     │Chunk3│      │ Mistral  │      │  ┌──────────┐  │         │
│  │ ...  │     │ ...  │      │ Local LLM│      │  │ VECTOR   │  │         │
│  └──────┘     └──────┘      └──────────┘      │  │ Column   │  │         │
│                                               │  │ + HNSW   │  │         │
│                                               │  │ Index    │  │         │
│  ┌────────────────────────────────────────────┴──┴──────────┴──┘         │
│  │                        RECHERCHE                                      │
│  └────────────────────────────────────────────────────────────────┐      │
│                                                                   │      │
│  Question      Embedding     Recherche        Contexte      LLM   ▼      │
│  ┌───────┐     ┌──────┐      ┌────────────┐   ┌────────┐   ┌───────────┐ │
│  │"Quelle│     │Query │      │VEC_DISTANCE│   │Top-K   │   │  Réponse  │ │
│  │est la │────►│Vector│─────►│_COSINE     │──►│Chunks  │──►│  augmentée│ │
│  │proc-  │     │      │      │< 0.3       │   │        │   │  avec     │ │
│  │édure" │     │      │      │            │   │        │   │  sources  │ │
│  └───────┘     └──────┘      └────────────┘   └────────┘   └───────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Implémentation MariaDB Vector

```sql
-- 🆕 Création d'une table avec colonne vectorielle
CREATE TABLE documents (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    chunk_index INT DEFAULT 0,
    source_url VARCHAR(2000),
    -- Vecteur d'embedding (dimension 1536 pour OpenAI ada-002)
    embedding VECTOR(1536) NOT NULL,
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 🆕 Index HNSW pour recherche ANN (Approximate Nearest Neighbor)
    VECTOR INDEX idx_embedding (embedding) 
        WITH (M=16, ef_construction=200)
) ENGINE=InnoDB;

-- Insertion avec embedding
INSERT INTO documents (title, content, embedding, metadata)
VALUES (
    'Guide MariaDB 11.8',
    'MariaDB 11.8 LTS introduit la recherche vectorielle native...',
    VEC_FromText('[0.023, -0.045, 0.089, ...]'),  -- 1536 dimensions
    '{"source": "documentation", "version": "11.8"}'
);

-- 🆕 Recherche sémantique par similarité cosinus
SELECT 
    id,
    title,
    content,
    VEC_DISTANCE_COSINE(embedding, @query_vector) AS distance
FROM documents
WHERE VEC_DISTANCE_COSINE(embedding, @query_vector) < 0.3
ORDER BY distance
LIMIT 5;

-- Recherche hybride : vecteurs + filtres SQL classiques
SELECT 
    d.id,
    d.title,
    d.content,
    VEC_DISTANCE_COSINE(d.embedding, @query_vector) AS semantic_score
FROM documents d
WHERE 
    d.metadata->>'$.source' = 'documentation'
    AND d.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND VEC_DISTANCE_COSINE(d.embedding, @query_vector) < 0.4
ORDER BY semantic_score
LIMIT 10;
```

### Semantic Search

```sql
-- Table produits avec embeddings de descriptions
CREATE TABLE products_vector (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2),
    -- Embedding de la description
    description_embedding VECTOR(768) NOT NULL,  -- MiniLM-L6-v2
    
    VECTOR INDEX idx_desc_embed (description_embedding)
        WITH (M=24, ef_construction=256)
) ENGINE=InnoDB;

-- Recherche produits par description naturelle
SET @search_query = 'laptop léger pour développeur avec bonne autonomie';
-- L'embedding de @search_query est généré côté application

SELECT 
    product_id,
    name,
    description,
    price,
    VEC_DISTANCE_COSINE(description_embedding, @query_embedding) AS relevance
FROM products_vector
WHERE category IN ('Laptops', 'Computers')
ORDER BY relevance
LIMIT 20;
```

### Recommendation Engine

```sql
-- Table des interactions utilisateurs
CREATE TABLE user_interactions (
    user_id BIGINT,
    product_id BIGINT,
    interaction_type ENUM('view', 'click', 'purchase', 'rating'),
    rating TINYINT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, product_id, timestamp)
) ENGINE=InnoDB;

-- Profil utilisateur comme moyenne pondérée des embeddings
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    -- Embedding du profil utilisateur (agrégation des préférences)
    preference_vector VECTOR(768),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    VECTOR INDEX idx_user_pref (preference_vector)
) ENGINE=InnoDB;

-- Mise à jour du profil utilisateur (procédure simplifiée)
DELIMITER //
CREATE PROCEDURE update_user_profile(IN p_user_id BIGINT)
BEGIN
    -- Calcul de la moyenne pondérée des embeddings des produits interagis
    -- (En production, utiliser un job batch ou un service dédié)
    INSERT INTO user_profiles (user_id, preference_vector)
    SELECT 
        p_user_id,
        -- Moyenne des vecteurs pondérée par le type d'interaction
        AVG(p.description_embedding)
    FROM user_interactions ui
    JOIN products_vector p ON ui.product_id = p.product_id
    WHERE ui.user_id = p_user_id
      AND ui.timestamp > DATE_SUB(NOW(), INTERVAL 90 DAY)
    ON DUPLICATE KEY UPDATE 
        preference_vector = VALUES(preference_vector),
        last_updated = NOW();
END //
DELIMITER ;

-- Recommandations personnalisées
SELECT 
    p.product_id,
    p.name,
    p.price,
    VEC_DISTANCE_COSINE(p.description_embedding, up.preference_vector) AS affinity
FROM products_vector p
CROSS JOIN user_profiles up
WHERE up.user_id = 12345
  AND p.product_id NOT IN (
      SELECT product_id FROM user_interactions 
      WHERE user_id = 12345 AND interaction_type = 'purchase'
  )
ORDER BY affinity
LIMIT 10;
```

### Hybrid Search (SQL + Vecteurs)

La puissance de MariaDB Vector réside dans la combinaison des filtres SQL classiques avec la recherche sémantique :

```sql
-- Recherche hybride : pertinence sémantique + contraintes métier
WITH semantic_results AS (
    SELECT 
        p.product_id,
        p.name,
        p.price,
        p.category,
        p.stock_quantity,
        VEC_DISTANCE_COSINE(p.description_embedding, @query_embedding) AS semantic_distance
    FROM products_vector p
    WHERE VEC_DISTANCE_COSINE(p.description_embedding, @query_embedding) < 0.5
),
keyword_boost AS (
    SELECT 
        sr.*,
        CASE 
            WHEN sr.name LIKE '%gaming%' THEN 0.1
            WHEN sr.category = 'Gaming' THEN 0.05
            ELSE 0
        END AS keyword_bonus
    FROM semantic_results sr
)
SELECT 
    product_id,
    name,
    price,
    category,
    -- Score combiné : sémantique + bonus keyword + boost stock
    (semantic_distance - keyword_bonus - 
     CASE WHEN stock_quantity > 100 THEN 0.02 ELSE 0 END) AS final_score
FROM keyword_boost
WHERE stock_quantity > 0
  AND price BETWEEN 500 AND 2000
ORDER BY final_score
LIMIT 20;
```

---

## 20.10 MariaDB MCP Server 🆕

Le **Model Context Protocol (MCP)** est un standard émergent pour connecter les LLMs aux sources de données. MariaDB propose un serveur MCP officiel :

```
┌────────────────────────────────────────────────────────────────────────┐
│                    MARIADB MCP SERVER                                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         LLM / Agent IA                          │   │
│  │                    (Claude, GPT, Llama, etc.)                   │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
│                                 │ MCP Protocol                         │
│                                 │ (JSON-RPC over stdio/HTTP)           │
│                                 ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    MariaDB MCP Server                           │   │
│  │                                                                 │   │
│  │  Capabilities:                                                  │   │
│  │  • list_tables     - Lister les tables disponibles              │   │
│  │  • describe_table  - Schéma d'une table                         │   │
│  │  • query           - Exécuter des requêtes SELECT               │   │
│  │  • insert/update   - Modifications (si autorisé)                │   │
│  │  • vector_search   - Recherche sémantique 🆕                     │   │
│  │                                                                 │   │
│  │  Sécurité:                                                      │   │
│  │  • Read-only par défaut                                         │   │
│  │  • Allowlist de tables/colonnes                                 │   │
│  │  • Rate limiting                                                │   │
│  │  • Audit logging                                                │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
│                                 │                                      │
│                                 ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      MariaDB 11.8 LTS                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │   Tables    │  │   Vues      │  │    Vector Indexes       │  │   │
│  │  │   InnoDB    │  │             │  │        HNSW             │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Configuration MCP Server

```yaml
# mcp-server-config.yaml
server:
  name: "mariadb-mcp"
  version: "1.0.0"
  transport: "stdio"  # ou "http" pour accès distant

database:
  host: "localhost"
  port: 3306
  user: "mcp_readonly"
  password: "${MCP_DB_PASSWORD}"
  database: "production"
  ssl:
    enabled: true
    ca: "/etc/mysql/ssl/ca.pem"

security:
  read_only: true
  allowed_tables:
    - "products"
    - "categories"
    - "orders"
    - "documents"  # Pour vector search
  blocked_columns:
    - "*.password_hash"
    - "*.ssn"
    - "*.credit_card"
  max_rows_per_query: 1000
  rate_limit:
    requests_per_minute: 60

vector:
  enabled: true
  embedding_model: "openai/text-embedding-3-small"
  default_limit: 10

logging:
  level: "info"
  audit: true
  destination: "/var/log/mcp/mariadb-mcp.log"
```

### Utilisation avec Claude Desktop

```json
// claude_desktop_config.json
{
  "mcpServers": {
    "mariadb": {
      "command": "mariadb-mcp-server",
      "args": ["--config", "/path/to/mcp-server-config.yaml"],
      "env": {
        "MCP_DB_PASSWORD": "secure_password"
      }
    }
  }
}
```

---

## 20.11 Intégrations Frameworks IA 🆕

### LangChain avec MariaDB Vector

```python
# langchain_mariadb.py
from langchain_community.vectorstores import MariaDBVector
from langchain_openai import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI
from langchain.chains import RetrievalQA

# Configuration connexion MariaDB
connection_string = "mariadb+mariadbconnector://user:pass@localhost:3306/vectordb"

# Initialisation du vector store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = MariaDBVector(
    embedding_function=embeddings,
    connection_string=connection_string,
    table_name="documents",
    embedding_column="embedding",
    content_column="content",
    metadata_columns=["title", "source_url", "created_at"]
)

# Indexation de documents
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)

documents = [...]  # Vos documents
chunks = text_splitter.split_documents(documents)

# Insertion batch avec embeddings
vectorstore.add_documents(chunks)

# Chaîne RAG
llm = ChatOpenAI(model="gpt-4", temperature=0)
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 5}
)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=retriever,
    return_source_documents=True
)

# Requête
result = qa_chain.invoke({
    "query": "Quelles sont les nouvelles fonctionnalités de MariaDB 11.8?"
})
print(result["result"])
print("Sources:", [doc.metadata["title"] for doc in result["source_documents"]])
```

### LlamaIndex avec MariaDB

```python
# llamaindex_mariadb.py
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.vector_stores.mariadb import MariaDBVectorStore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.anthropic import Anthropic

# Configuration MariaDB Vector Store
vector_store = MariaDBVectorStore(
    host="localhost",
    port=3306,
    user="llama_user",
    password="secure_password",
    database="vectordb",
    table_name="llama_documents",
    embedding_dimension=1536,
    hnsw_m=16,
    hnsw_ef_construction=200
)

# Chargement et indexation
documents = SimpleDirectoryReader("./data").load_data()
embed_model = OpenAIEmbedding(model="text-embedding-3-small")

index = VectorStoreIndex.from_documents(
    documents,
    vector_store=vector_store,
    embed_model=embed_model
)

# Query engine avec Claude
llm = Anthropic(model="claude-sonnet-4-20250514")
query_engine = index.as_query_engine(llm=llm)

response = query_engine.query(
    "Explique l'architecture de MariaDB Galera Cluster"
)
print(response)
```

### Tableau comparatif des frameworks

| Framework | Points forts | Intégration MariaDB | Use case idéal |
|-----------|--------------|---------------------|----------------|
| **LangChain** | Écosystème riche, chaînes complexes | VectorStore natif | Applications RAG générales |
| **LlamaIndex** | Focus indexation, data connectors | VectorStore natif | Knowledge bases structurées |
| **Haystack** | NLP avancé, pipelines flexibles | Via SQLDocumentStore | Systèmes Q&A complexes |
| **Semantic Kernel** | Intégration Microsoft, .NET | Connecteur custom | Applications entreprise |

---

## 20.12 Études de Cas Réelles

### Cas 1 : E-commerce haute disponibilité (500K commandes/jour)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CAS D'USAGE : PLATEFORME E-COMMERCE                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Contexte :                                                             │
│  • 500K commandes/jour, pics à 2000 commandes/minute (Black Friday)     │
│  • 10M produits, 50M utilisateurs                                       │
│  • SLA 99.95%, latence P99 < 100ms                                      │
│  • Présence EU + US                                                     │
│                                                                         │
│  Architecture retenue :                                                 │
│                                                                         │
│                    ┌──────────────────────────────────────┐             │
│                    │           Global Load Balancer       │             │
│                    └──────────────────┬───────────────────┘             │
│                                       │                                 │
│          ┌────────────────────────────┼────────────────────────────┐    │
│          │                            │                            │    │
│          ▼                            ▼                            ▼    │
│  ┌───────────────┐           ┌───────────────┐           ┌───────────┐  │
│  │   EU Region   │           │   US Region   │           │    CDN    │  │
│  │               │           │               │           │           │  │
│  │ ┌───────────┐ │           │ ┌───────────┐ │           │  Static   │  │
│  │ │ MaxScale  │ │           │ │ MaxScale  │ │           │  Assets   │  │
│  │ └─────┬─────┘ │           │ └─────┬─────┘ │           └───────────┘  │
│  │       │       │           │       │       │                          │
│  │ ┌─────▼─────┐ │           │ ┌─────▼─────┐ │                          │
│  │ │  Galera   │ │◄─────────►│ │  Galera   │ │                          │
│  │ │  (3 nodes)│ │   Async   │ │  (3 nodes)│ │                          │
│  │ └───────────┘ │   Repl    │ └───────────┘ │                          │
│  └───────────────┘           └───────────────┘                          │
│                                                                         │
│  Décisions clés :                                                       │
│  ✅ Galera intra-région pour HA et consistance forte                    │
│  ✅ Réplication async inter-région (latence WAN incompatible Galera)    │
│  ✅ MaxScale pour read/write split et failover automatique              │
│  ✅ Écritures locales à chaque région (eventual consistency acceptée)   │
│  ✅ InnoDB avec buffer pool 70% RAM (256GB → 180GB buffer pool)         │
│                                                                         │
│  Métriques atteintes :                                                  │
│  • Latence P99 : 45ms (lectures), 85ms (écritures)                      │
│  • Disponibilité : 99.97%                                               │
│  • Failover automatique : < 30 secondes                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Cas 2 : Plateforme SaaS multi-tenant (10K tenants)

```
┌────────────────────────────────────────────────────────────────────────┐
│  CAS D'USAGE : PLATEFORME SAAS B2B                                     │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Contexte :                                                            │
│  • 10K tenants, de 10 à 100K users par tenant                          │
│  • Modèle freemium : 80% free, 15% pro, 5% enterprise                  │
│  • Isolation des données critique (compliance SOC2)                    │
│  • Budget infrastructure optimisé                                      │
│                                                                        │
│  Architecture retenue : Hybrid (Shared + Dedicated)                    │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    API Gateway / Router                         │   │
│  │               (Route selon tenant tier)                         │   │
│  └───────────────────────────┬─────────────────────────────────────┘   │
│                              │                                         │
│         ┌────────────────────┼────────────────────┐                    │
│         │                    │                    │                    │
│         ▼                    ▼                    ▼                    │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │
│  │   FREE      │      │    PRO      │      │ ENTERPRISE  │             │
│  │   Tier      │      │    Tier     │      │    Tier     │             │
│  │             │      │             │      │             │             │
│  │ Shared DB   │      │ Schema/     │      │ Dedicated   │             │
│  │ tenant_id   │      │ Tenant      │      │ DB Cluster  │             │
│  │ discriminator      │             │      │             │             │
│  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘             │
│         │                    │                    │                    │
│         ▼                    ▼                    ▼                    │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │
│  │ MariaDB     │      │ MariaDB     │      │ Galera      │             │
│  │ Single      │      │ Galera (3)  │      │ Dédié (3)   │             │
│  │ + Read Rep  │      │             │      │ + MaxScale  │             │
│  └─────────────┘      └─────────────┘      └─────────────┘             │
│                                                                        │
│  Décisions clés :                                                      │
│  ✅ Free : Shared schema avec Row-Level Security (coût minimal)        │
│  ✅ Pro : Schema par tenant (isolation + performances prévisibles)     │
│  ✅ Enterprise : Cluster dédié (isolation totale, SLA custom)          │
│  ✅ Migration transparente entre tiers                                 │
│  ✅ Noisy neighbor mitigation via resource governor                    │
│                                                                        │
│  ROI :                                                                 │
│  • Coût infra Free tier : $0.50/tenant/mois                            │
│  • Coût infra Pro tier : $5/tenant/mois                                │
│  • Coût infra Enterprise : $200-2000/tenant/mois (custom)              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Cas 3 : Plateforme IA avec RAG (Knowledge Base) 🆕

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CAS D'USAGE : ASSISTANT IA ENTREPRISE AVEC RAG                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Contexte :                                                             │
│  • 500K documents internes (PDF, Wiki, Confluence, Slack)               │
│  • 10K utilisateurs, 50K requêtes/jour                                  │
│  • Réponses contextuelles avec sources citées                           │
│  • Données confidentielles (pas de cloud LLM pour le stockage)          │
│                                                                         │
│  Architecture retenue :                                                 │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                      Pipeline d'Indexation                     │     │
│  │  Confluence ──┐                                                │     │
│  │  SharePoint ──┼──► Chunking ──► Embedding ──► MariaDB Vector   │     │
│  │  Slack      ──┤    (1000 tokens)  (API)       (HNSW Index)     │     │
│  │  PDFs       ──┘                                                │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                      Pipeline de Requête                       │     │
│  │                                                                │     │
│  │  ┌─────────┐    ┌───────────────────────────────────────────┐  │     │
│  │  │ User    │    │            MariaDB 11.8                   │  │     │
│  │  │ Question│───►│  ┌───────────────────────────────────┐    │  │     │
│  │  └─────────┘    │  │     Hybrid Search                 │    │  │     │
│  │                 │  │                                   │    │  │     │
│  │  ┌─────────┐    │  │  Vector Search (HNSW)             │    │  │     │
│  │  │ Embedding───►│  │  + Permission Filter (SQL)        │    │  │     │
│  │  │ API     │    │  │  + Recency Boost                  │    │  │     │
│  │  └─────────┘    │  │  + Source Type Weight             │    │  │     │
│  │                 │  └───────────────────────────────────┘    │  │     │
│  │                 └──────────────────┬────────────────────────┘  │     │
│  │                                    │                           │     │
│  │                                    ▼                           │     │
│  │                 ┌──────────────────────────────────────────┐   │     │
│  │                 │  LLM (Claude/GPT) avec contexte          │   │     │
│  │                 │  Top-10 chunks + conversation history    │───┼────►│
│  │                 │  → Réponse + Sources citées              │   │     │
│  │                 └──────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  Schéma MariaDB :                                                       │
│                                                                         │
│  CREATE TABLE knowledge_chunks (                                        │
│      id BIGINT PRIMARY KEY,                                             │
│      source_type ENUM('confluence','sharepoint','slack','pdf'),         │
│      source_id VARCHAR(500),                                            │
│      content TEXT,                                                      │
│      embedding VECTOR(1536),                                            │
│      permissions JSON,        -- {"groups": ["engineering", "all"]}     │
│      created_at TIMESTAMP,                                              │
│      updated_at TIMESTAMP,                                              │
│      VECTOR INDEX (embedding) WITH (M=16, ef_construction=200)          │
│  );                                                                     │
│                                                                         │
│  Décisions clés :                                                       │
│  ✅ MariaDB Vector : Données on-premise, pas de vendor lock-in          │
│  ✅ Hybrid search : Vecteurs + permissions SQL (sécurité)               │
│  ✅ Chunking 1000 tokens avec 200 overlap (contexte optimal)            │
│  ✅ HNSW M=16 : Balance recall/performance pour 500K docs               │
│  ✅ Embedding via API (flexibility), stockage local                     │
│                                                                         │
│  Métriques :                                                            │
│  • Latence recherche P95 : 85ms (10 résultats)                          │
│  • Recall@10 : 92%                                                      │
│  • Satisfaction utilisateurs : 4.2/5                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Points clés à retenir

- **OLTP vs OLAP** : InnoDB pour les transactions, ColumnStore pour l'analytique, architecture HTAP possible avec réplication cross-engine
- **Microservices** : Database-per-service pour l'autonomie, shared database pour la simplicité — le choix dépend de la maturité de l'équipe et des besoins de transactions
- **Multi-tenant** : Trois approches (database, schema, shared) avec des trade-offs coût/isolation/complexité différents
- **Géo-distribution** : Galera intra-région + réplication async inter-région est le pattern recommandé
- **Event-Driven** : CDC avec Debezium transforme MariaDB en source d'événements pour architectures réactives
- 🆕 **MariaDB Vector** : Recherche sémantique native, index HNSW performant, intégration naturelle avec les workloads SQL existants
- 🆕 **RAG et IA** : MariaDB 11.8 permet des architectures RAG complètes avec hybrid search (vecteurs + filtres SQL)
- 🆕 **Frameworks IA** : Intégrations LangChain, LlamaIndex, et MCP Server disponibles pour une adoption rapide

---

## 🔗 Ressources et références

- [📖 MariaDB Vector Documentation](https://mariadb.com/kb/en/vector/)
- [📖 MariaDB ColumnStore](https://mariadb.com/kb/en/mariadb-columnstore/)
- [📖 Debezium MariaDB Connector](https://debezium.io/documentation/reference/connectors/mariadb.html)
- [📖 LangChain MariaDB Integration](https://python.langchain.com/docs/integrations/vectorstores/)
- [📖 Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [📖 MariaDB Architecture Guides](https://mariadb.com/resources/blog/)

---

## ➡️ Sections suivantes

Ce chapitre se décline en 12 sections détaillées :

| Section | Titre | Focus |
|---------|-------|-------|
| 20.1 | OLTP vs OLAP | Configurations optimisées par workload |
| 20.2 | Architecture microservices | Patterns database per service et shared |
| 20.3 | Data warehousing ColumnStore | Star schema et ETL |
| 20.4 | Architecture multi-tenant | Stratégies d'isolation |
| 20.5 | Géo-distribution | Réplication multi-région |
| 20.6 | Hybrid/Multi-cloud | Déploiements Terraform |
| 20.7 | Scaling vertical vs horizontal | Stratégies de croissance |
| 20.8 | Architectures Event-Driven | CDC, Kafka, Debezium |
| 20.9 | Use cases IA/RAG 🆕 | Vector search, semantic search |
| 20.10 | MariaDB MCP Server 🆕 | Intégration LLM |
| 20.11 | Frameworks IA 🆕 | LangChain, LlamaIndex |
| 20.12 | Études de cas réelles | Architectures production |

---

**MariaDB** : 11.8 LTS

⏭️ [OLTP vs OLAP](/20-cas-usage-architectures/01-oltp-vs-olap.md)

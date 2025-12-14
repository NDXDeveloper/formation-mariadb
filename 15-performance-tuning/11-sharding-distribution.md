🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.11 Sharding et distribution horizontale

> **Niveau** : Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : 
> - Sections 15.1-15.10 (Performance, Partitionnement, Gestion avancée)
> - Expérience avec réplication MariaDB
> - Architecture distribuée et concepts de scaling
> - Compréhension des trade-offs CAP theorem

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre le sharding** et le distinguer du partitionnement
- **Identifier les besoins** justifiant une architecture sharded
- **Choisir une stratégie** de sharding appropriée
- **Implémenter le sharding** avec Spider Storage Engine
- **Configurer ProxySQL** pour le routing intelligent
- **Gérer les migrations** vers architecture sharded
- **Résoudre les défis** des jointures cross-shard
- **Monitorer et maintenir** un cluster sharded
- **Appliquer les best practices** de production
- **Planifier la croissance** et le rebalancing

---

## Introduction

Le **sharding** (ou **distribution horizontale**) est une technique qui distribue les données d'une table sur **plusieurs serveurs MariaDB indépendants**, chacun gérant un sous-ensemble des données.

### Partitionnement vs Sharding

```
┌────────────────────────────────────────────────────────────┐
│  PARTITIONNEMENT (Vertical Scaling)                        │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Un serveur unique avec partitions                         │
│  ┌──────────────────────────────────────────────┐          │
│  │  Server 1 (500 GB RAM, 100 cores)            │          │
│  │  ├─ orders p2020 (100 GB)                    │          │
│  │  ├─ orders p2021 (100 GB)                    │          │
│  │  ├─ orders p2022 (100 GB)                    │          │
│  │  └─ orders p2023 (100 GB)                    │          │
│  └──────────────────────────────────────────────┘          │
│                                                            │
│  Limites :                                                 │
│  • RAM : ~4 TB maximum pratique                            │
│  • CPU : ~200 cores maximum                                │
│  • I/O : Limité par un seul serveur                        │
│  • Coût : Serveur unique très cher                         │
│  • SPOF : Single Point of Failure                          │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  SHARDING (Horizontal Scaling)                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Plusieurs serveurs indépendants                           │
│                                                            │
│  ┌────────────────────┐  ┌────────────────────┐            │
│  │  Shard 1           │  │  Shard 2           │            │
│  │  (64 GB RAM)       │  │  (64 GB RAM)       │            │
│  │  orders 2020-2021  │  │  orders 2022-2023  │            │
│  │  (200 GB data)     │  │  (200 GB data)     │            │
│  └────────────────────┘  └────────────────────┘            │
│                                                            │
│  ┌────────────────────┐  ┌────────────────────┐            │
│  │  Shard 3           │  │  Shard 4           │            │
│  │  (64 GB RAM)       │  │  (64 GB RAM)       │            │
│  │  orders region EU  │  │  orders region US  │            │
│  │  (150 GB data)     │  │  (250 GB data)     │            │
│  └────────────────────┘  └────────────────────┘            │
│                                                            │
│  Avantages :                                               │
│  ✅ Scaling quasi-illimité (ajout serveurs)                │
│  ✅ Coût optimisé (serveurs standards)                     │
│  ✅ Parallélisation native                                 │
│  ✅ Isolation des pannes                                   │
│  ✅ Localisation géographique possible                     │
│                                                            │
│  Défis :                                                   │
│  ⚠️ Complexité architecture                                │
│  ⚠️ Jointures cross-shard difficiles                       │
│  ⚠️ Transactions distribuées complexes                     │
│  ⚠️ Rebalancing nécessaire                                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Quand utiliser le sharding ?

### Scénarios appropriés ✅

```
┌────────────────────────────────────────────────────┐
│  SIGNAUX QUE LE SHARDING EST NÉCESSAIRE            │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. LIMITES MATÉRIELLES ATTEINTES                  │
│     • Table > 10 TB impossible à gérer             │
│     • RAM requise > 2 TB (limite pratique)         │
│     • Single server trop coûteux                   │
│                                                    │
│  2. CROISSANCE EXPONENTIELLE                       │
│     • +100% données/an                             │
│     • Vertical scaling non viable long terme       │
│     • Budget contraint                             │
│                                                    │
│  3. WORKLOAD ISOLABLE                              │
│     • Données naturellement partitionnables        │
│     • Peu de requêtes cross-partition              │
│     • Exemples : Multi-tenant, géographie          │
│                                                    │
│  4. HAUTE DISPONIBILITÉ REQUISE                    │
│     • Tolérance aux pannes partielles              │
│     • 99.99%+ uptime SLA                           │
│     • Isolation des incidents                      │
│                                                    │
│  5. LOCALISATION GÉOGRAPHIQUE                      │
│     • Conformité réglementaire (GDPR)              │
│     • Latence optimisée par région                 │
│     • Données EU vs US vs APAC                     │
│                                                    │
│  6. WRITE THROUGHPUT INSUFFISANT                   │
│     • Single server saturé en writes               │
│     • Besoin > 100k writes/sec                     │
│     • Distribution pour parallélisme               │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Quand NE PAS sharder ❌

```
┌────────────────────────────────────────────────────┐
│  ALTERNATIVES AU SHARDING                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  Si problème = READ performance :                  │
│  → Read replicas + ProxySQL                        │
│  → Beaucoup plus simple que sharding               │
│                                                    │
│  Si table < 5 TB :                                 │
│  → Partitionnement + serveur plus gros             │
│  → Optimisation requêtes + index                   │
│                                                    │
│  Si peu de croissance :                            │
│  → Archivage des anciennes données                 │
│  → Pas besoin de sharding                          │
│                                                    │
│  Si beaucoup de jointures complexes :              │
│  → Sharding rendra queries impossibles             │
│  → Rester sur architecture centralisée             │
│                                                    │
│  Si équipe petite/inexpérimentée :                 │
│  → Complexité opérationnelle trop élevée           │
│  → Risque d'erreurs critiques                      │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Stratégies de sharding

### 1. Sharding par clé (Key-based / Hash)

**Principe** : Distribuer selon hash d'une clé.

```
┌─────────────────────────────────────────────────────┐
│  SHARDING PAR CLÉ (customer_id)                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Fonction : shard_id = HASH(customer_id) MOD 4      │
│                                                     │
│  customer_id = 12345 → HASH → 2 → Shard 2           │
│  customer_id = 67890 → HASH → 0 → Shard 0           │
│                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Shard 0 │  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │ │
│  │ 25% data│  │ 25% data│  │ 25% data│  │ 25% data│ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │
│                                                     │
│  Avantages :                                        │
│  ✅ Distribution uniforme                           │
│  ✅ Routing simple (calcul hash)                    │
│  ✅ Scaling prévisible                              │
│                                                     │
│  Inconvénients :                                    │
│  ❌ Rebalancing complexe (ajout shard)              │
│  ❌ Requêtes multi-client = multi-shard             │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Implémentation** :

```sql
-- Fonction de sharding
CREATE FUNCTION get_shard_id(p_customer_id BIGINT)
RETURNS INT
DETERMINISTIC
RETURN MOD(p_customer_id, 4);  -- 4 shards

-- Application détermine shard
SET @shard_id = get_shard_id(12345);  -- 1

-- Connect to shard 1 et exécuter requête
SELECT * FROM orders WHERE customer_id = 12345;
```

### 2. Sharding par plage (Range-based)

**Principe** : Distribuer selon plages de valeurs.

```
┌────────────────────────────────────────────────────┐
│  SHARDING PAR PLAGE (order_date)                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  Shard 0 : 2020-2021                               │
│  Shard 1 : 2022-2023                               │
│  Shard 2 : 2024+                                   │
│                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  Shard 0    │  │  Shard 1    │  │  Shard 2    │ │
│  │  2020-2021  │  │  2022-2023  │  │  2024+      │ │
│  │  (archive)  │  │  (archive)  │  │  (actif)    │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                    │
│  Avantages :                                       │
│  ✅ Requêtes par plage efficaces                   │
│  ✅ Archivage naturel (shard complet)              │
│  ✅ Ajout shard simple                             │
│                                                    │
│  Inconvénients :                                   │
│  ❌ Distribution potentiellement inégale           │
│  ❌ Hot shard (shard actif surchargé)              │
│                                                    │
└────────────────────────────────────────────────────┘
```

### 3. Sharding géographique

**Principe** : Distribuer selon localisation.

```
┌────────────────────────────────────────────────────────┐
│  SHARDING GÉOGRAPHIQUE                                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Shard EU    │  │  Shard US    │  │  Shard APAC  │  │
│  │  Frankfurt   │  │  Virginia    │  │  Singapore   │  │
│  │              │  │              │  │              │  │
│  │  FR, DE, IT  │  │  US, CA, BR  │  │  SG, JP, AU  │  │
│  │  ES, UK      │  │  MX          │  │  IN, KR      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                        │
│  Avantages :                                           │
│  ✅ Latence minimale (proximité users)                 │
│  ✅ Conformité GDPR (données EU en EU)                 │
│  ✅ Isolation des pannes géographiques                 │
│                                                        │
│  Cas d'usage :                                         │
│  • SaaS global                                         │
│  • E-commerce international                            │
│  • Réglementation stricte                              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 4. Sharding par tenant (Multi-tenancy)

**Principe** : Un ou plusieurs tenants par shard.

```
┌────────────────────────────────────────────────────┐
│  SHARDING PAR TENANT (SaaS)                        │
├────────────────────────────────────────────────────┤
│                                                    │
│  Shard 0 : Tenants 1-100 (petits clients)          │
│  Shard 1 : Tenant 101 (gros client exclusif)       │
│  Shard 2 : Tenants 102-200                         │
│  Shard 3 : Tenant 201 (gros client exclusif)       │
│                                                    │
│  Avantages :                                       │
│  ✅ Isolation complète des données                 │
│  ✅ Performance prédictible par tenant             │
│  ✅ Migration tenant facile                        │
│  ✅ Facturation précise                            │
│                                                    │
│  Use case :                                        │
│  • Plateforme SaaS B2B                             │
│  • Clients avec besoins différents                 │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Implémentation avec Spider Storage Engine

### Introduction à Spider

**Spider** est un storage engine MariaDB qui permet de créer des tables fédérées distribuées sur plusieurs serveurs.

```
┌────────────────────────────────────────────────────┐
│  ARCHITECTURE SPIDER                               │
├────────────────────────────────────────────────────┤
│                                                    │
│  Application                                       │
│       ↓                                            │
│  ┌──────────────────────────────────────┐          │
│  │  MariaDB Spider Node (Proxy)         │          │
│  │  ┌────────────────────────────────┐  │          │
│  │  │ Table "orders" (ENGINE=Spider) │  │          │
│  │  │ - Metadata seulement           │  │          │
│  │  │ - Routing logic                │  │          │
│  │  └────────────────────────────────┘  │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                  │
│        ┌────────┼────────┐                         │
│        ▼        ▼        ▼                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│  │ Shard 0 │ │ Shard 1 │ │ Shard 2 │               │
│  │ orders  │ │ orders  │ │ orders  │               │
│  │ (data)  │ │ (data)  │ │ (data)  │               │
│  └─────────┘ └─────────┘ └─────────┘               │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Configuration Spider

```sql
-- Sur Spider Node (proxy)

-- 1. Installer Spider engine
INSTALL SONAME 'ha_spider';

-- 2. Créer serveurs backend (shards)
CREATE SERVER shard0
FOREIGN DATA WRAPPER mysql
OPTIONS (
    HOST '192.168.1.10',
    DATABASE 'orders_db',
    USER 'spider_user',
    PASSWORD 'spider_pass',
    PORT 3306
);

CREATE SERVER shard1
FOREIGN DATA WRAPPER mysql
OPTIONS (
    HOST '192.168.1.11',
    DATABASE 'orders_db',
    USER 'spider_user',
    PASSWORD 'spider_pass',
    PORT 3306
);

CREATE SERVER shard2
FOREIGN DATA WRAPPER mysql
OPTIONS (
    HOST '192.168.1.12',
    DATABASE 'orders_db',
    USER 'spider_user',
    PASSWORD 'spider_pass',
    PORT 3306
);

-- 3. Créer table Spider avec sharding par hash
CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, customer_id)
) ENGINE=Spider
COMMENT='wrapper "mysql", table "orders"'
PARTITION BY HASH (customer_id) (
    PARTITION pt0 COMMENT='srv "shard0"',
    PARTITION pt1 COMMENT='srv "shard1"',
    PARTITION pt2 COMMENT='srv "shard2"'
);

-- 4. Créer tables backend sur chaque shard
-- Sur shard0, shard1, shard2 :
CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, customer_id)
) ENGINE=InnoDB;
```

### Requêtes via Spider

```sql
-- Application se connecte au Spider Node

-- INSERT : Spider route automatiquement vers bon shard
INSERT INTO orders (customer_id, order_date, amount)
VALUES (12345, '2024-01-15', 99.99);
-- Automatiquement vers shard HASH(12345) MOD 3 = shard 1

-- SELECT avec sharding key : Query un seul shard
SELECT * FROM orders WHERE customer_id = 12345;
-- Automatiquement routé vers shard 1

-- SELECT sans sharding key : Query tous shards
SELECT COUNT(*) FROM orders WHERE order_date = '2024-01-15';
-- Spider query shard0, shard1, shard2 en parallèle
-- Agrège résultats

-- JOIN : Spider tente d'optimiser
SELECT o.*, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id = 12345;
-- Si customers aussi sharded par customer_id : local join
-- Sinon : Cross-shard join (lent)
```

### Sharding par plage avec Spider

```sql
-- Sharding par année
CREATE TABLE orders_by_year (
    id BIGINT NOT NULL AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, order_date)
) ENGINE=Spider
COMMENT='wrapper "mysql", table "orders"'
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023) 
        COMMENT='srv "shard_2022"',
    PARTITION p2023 VALUES LESS THAN (2024) 
        COMMENT='srv "shard_2023"',
    PARTITION p2024 VALUES LESS THAN (2025) 
        COMMENT='srv "shard_2024"'
);
```

---

## ProxySQL pour routing intelligent

### Architecture ProxySQL

ProxySQL est un proxy SQL haute performance qui peut router intelligemment les requêtes.

```
┌────────────────────────────────────────────────────┐
│  ARCHITECTURE PROXYSQL SHARDING                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  Applications                                      │
│       ↓ (Port 6033)                                │
│  ┌──────────────────────────────────────┐          │
│  │         ProxySQL                     │          │
│  │  ┌────────────────────────────────┐  │          │
│  │  │ Query Rules :                  │  │          │
│  │  │ - Parse SQL                    │  │          │
│  │  │ - Extract shard key            │  │          │
│  │  │ - Route to shard               │  │          │
│  │  └────────────────────────────────┘  │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                  │
│        ┌────────┼────────┐                         │
│        ▼        ▼        ▼                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│  │ Shard 0 │ │ Shard 1 │ │ Shard 2 │               │
│  │ Host 0  │ │ Host 1  │ │ Host 2  │               │
│  └─────────┘ └─────────┘ └─────────┘               │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Configuration ProxySQL

```sql
-- Configuration ProxySQL admin interface

-- 1. Définir les shards comme backend servers
INSERT INTO mysql_servers (hostgroup_id, hostname, port)
VALUES
(10, '192.168.1.10', 3306),  -- Shard 0
(11, '192.168.1.11', 3306),  -- Shard 1
(12, '192.168.1.12', 3306);  -- Shard 2

-- 2. Définir utilisateurs
INSERT INTO mysql_users (username, password, default_hostgroup)
VALUES ('app_user', 'app_pass', 10);

-- 3. Créer query rules pour routing
-- Rule : Route queries avec customer_id vers shard approprié

-- Utiliser hint SQL pour indiquer shard
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES
(1, 1, '.*/*+ SHARD0 */.*', 10, 1),
(2, 1, '.*/*+ SHARD1 */.*', 11, 1),
(3, 1, '.*/*+ SHARD2 */.*', 12, 1);

-- 4. Load config
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL USERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;

-- 5. Persist
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL USERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;
```

### Application avec ProxySQL

```python
# Application Python avec routing manuel

import mysql.connector
import hashlib

def get_shard_id(customer_id, num_shards=3):
    """Calculer shard basé sur customer_id"""
    return int(hashlib.md5(str(customer_id).encode()).hexdigest(), 16) % num_shards

def execute_query(customer_id, query):
    """Exécuter requête avec hint shard"""
    shard_id = get_shard_id(customer_id)
    
    # Connexion ProxySQL
    conn = mysql.connector.connect(
        host='proxysql-host',
        port=6033,
        user='app_user',
        password='app_pass',
        database='orders_db'
    )
    
    cursor = conn.cursor()
    
    # Ajouter hint pour routing
    query_with_hint = f"/*+ SHARD{shard_id} */ {query}"
    
    cursor.execute(query_with_hint)
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return results

# Utilisation
customer_id = 12345
results = execute_query(
    customer_id,
    f"SELECT * FROM orders WHERE customer_id = {customer_id}"
)
```

---

## Gestion des jointures cross-shard

### Le problème

```sql
-- Requête problématique : JOIN entre tables sharded différemment
SELECT o.*, c.name, p.product_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN products p ON p.id = o.product_id
WHERE o.order_date >= '2024-01-01';

-- Si :
-- orders sharded par customer_id
-- customers sharded par customer_id (OK, colocated)
-- products PAS sharded (table de référence)
-- → JOIN cross-shard nécessaire
```

### Solutions

#### 1. Denormalisation

```sql
-- Dupliquer données pour éviter JOIN
CREATE TABLE orders_denormalized (
    id BIGINT,
    customer_id BIGINT,
    customer_name VARCHAR(100),      -- Dénormalisé
    customer_email VARCHAR(100),     -- Dénormalisé
    product_id INT,
    product_name VARCHAR(100),       -- Dénormalisé
    amount DECIMAL(10,2),
    order_date DATE
);

-- Maintenant : Single table query, pas de JOIN
SELECT * FROM orders_denormalized
WHERE customer_id = 12345;
```

#### 2. Tables de référence (broadcast)

```sql
-- Petites tables : Répliquer sur tous shards

-- Sur chaque shard : Copie complète de products
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
) ENGINE=InnoDB;

-- Sync automatique via réplication ou script
-- Maintenant JOIN local possible sur chaque shard
```

#### 3. Application-level JOIN

```python
# JOIN au niveau application

def get_order_with_details(customer_id):
    # 1. Query orders (shard customer_id)
    orders = query_shard(
        get_shard(customer_id),
        f"SELECT * FROM orders WHERE customer_id = {customer_id}"
    )
    
    # 2. Extract product_ids
    product_ids = [o['product_id'] for o in orders]
    
    # 3. Query products (table centralisée ou cache)
    products = query_products(product_ids)
    products_map = {p['id']: p for p in products}
    
    # 4. JOIN en Python
    for order in orders:
        order['product'] = products_map.get(order['product_id'])
    
    return orders
```

#### 4. Agrégation distribuée

```sql
-- Pour requêtes analytiques : Map-Reduce pattern

-- Map phase : Query chaque shard
-- Shard 0 : SELECT region, SUM(amount) FROM orders GROUP BY region
-- Shard 1 : SELECT region, SUM(amount) FROM orders GROUP BY region
-- Shard 2 : SELECT region, SUM(amount) FROM orders GROUP BY region

-- Reduce phase : Application agrège résultats
-- Total = SUM(résultats de tous shards)
```

---

## Migration vers architecture sharded

### Approche progressive

```
┌────────────────────────────────────────────────────┐
│  MIGRATION PROGRESSIVE (Zero Downtime)             │
├────────────────────────────────────────────────────┤
│                                                    │
│  Phase 1 : Préparation (1-2 semaines)              │
│  ────────────────────────────────                  │
│  • Provisionner serveurs shards                    │
│  • Installer MariaDB + réplication                 │
│  • Configurer Spider ou ProxySQL                   │
│  • Tests intensifs                                 │
│                                                    │
│  Phase 2 : Migration données (2-4 semaines)        │
│  ──────────────────────────────────                │
│  • Copie initiale par batch                        │
│  • Binlog replication pour sync                    │
│  • Vérification intégrité                          │
│                                                    │
│  Phase 3 : Dual-write (1 semaine)                  │
│  ───────────────────────                           │
│  • Application écrit sur old + new                 │
│  • Lectures encore sur old                         │
│  • Monitoring différences                          │
│                                                    │
│  Phase 4 : Basculement lectures (1 jour)           │
│  ─────────────────────────────────                 │
│  • Lectures progressivement sur shards             │
│  • Canary deployment                               │
│  • Rollback possible                               │
│                                                    │
│  Phase 5 : Cleanup (1 semaine)                     │
│  ────────────────────                              │
│  • Arrêter dual-write                              │
│  • Archiver ancien système                         │
│  • Documentation                                   │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Script de migration

```python
# Script de migration batch avec vérification

import mysql.connector
from datetime import datetime

class ShardMigrator:
    def __init__(self, source_db, shard_configs):
        self.source = source_db
        self.shards = shard_configs
        self.batch_size = 10000
        
    def get_shard_for_row(self, customer_id):
        """Déterminer shard destination"""
        return customer_id % len(self.shards)
    
    def migrate_batch(self, table, start_id, end_id):
        """Migrer un batch de données"""
        # 1. Lire depuis source
        source_conn = mysql.connector.connect(**self.source)
        cursor = source_conn.cursor(dictionary=True)
        
        query = f"""
            SELECT * FROM {table}
            WHERE id >= {start_id} AND id < {end_id}
        """
        cursor.execute(query)
        rows = cursor.fetchall()
        
        # 2. Grouper par shard
        shard_rows = {}
        for row in rows:
            shard_id = self.get_shard_for_row(row['customer_id'])
            if shard_id not in shard_rows:
                shard_rows[shard_id] = []
            shard_rows[shard_id].append(row)
        
        # 3. Insert dans chaque shard
        for shard_id, rows in shard_rows.items():
            shard_conn = mysql.connector.connect(**self.shards[shard_id])
            shard_cursor = shard_conn.cursor()
            
            # Bulk insert
            values = []
            for row in rows:
                values.append(tuple(row.values()))
            
            insert_query = f"""
                INSERT INTO {table} 
                VALUES ({','.join(['%s'] * len(row))})
                ON DUPLICATE KEY UPDATE id=id
            """
            shard_cursor.executemany(insert_query, values)
            shard_conn.commit()
            
            shard_cursor.close()
            shard_conn.close()
        
        cursor.close()
        source_conn.close()
        
        return len(rows)
    
    def verify_migration(self, table):
        """Vérifier intégrité après migration"""
        # Count source
        source_conn = mysql.connector.connect(**self.source)
        cursor = source_conn.cursor()
        cursor.execute(f"SELECT COUNT(*) FROM {table}")
        source_count = cursor.fetchone()[0]
        
        # Count shards
        total_shard_count = 0
        for shard_config in self.shards:
            shard_conn = mysql.connector.connect(**shard_config)
            shard_cursor = shard_conn.cursor()
            shard_cursor.execute(f"SELECT COUNT(*) FROM {table}")
            total_shard_count += shard_cursor.fetchone()[0]
        
        # Vérifier
        if source_count == total_shard_count:
            print(f"✓ Migration OK : {source_count} lignes")
            return True
        else:
            print(f"✗ Erreur : Source={source_count}, Shards={total_shard_count}")
            return False

# Utilisation
migrator = ShardMigrator(
    source_db={'host': 'old-server', 'database': 'orders_db'},
    shard_configs=[
        {'host': 'shard0', 'database': 'orders_db'},
        {'host': 'shard1', 'database': 'orders_db'},
        {'host': 'shard2', 'database': 'orders_db'}
    ]
)

# Migrer par batches
max_id = 10000000  # 10M lignes
for start in range(0, max_id, 10000):
    count = migrator.migrate_batch('orders', start, start + 10000)
    print(f"Migrated batch {start}-{start+10000}: {count} rows")

# Vérifier
migrator.verify_migration('orders')
```

---

## Monitoring et maintenance

### Métriques critiques

```sql
-- Dashboard sharding

CREATE TABLE shard_health (
    shard_id INT,
    hostname VARCHAR(100),
    checked_at TIMESTAMP,
    status ENUM('OK', 'WARNING', 'CRITICAL'),
    row_count BIGINT,
    data_size_gb DECIMAL(10,2),
    cpu_usage DECIMAL(5,2),
    memory_usage DECIMAL(5,2),
    active_connections INT,
    slow_queries_count INT
);

-- Script de monitoring (exécuté périodiquement)
DELIMITER //
CREATE OR REPLACE PROCEDURE monitor_shards()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_shard_id INT;
    DECLARE v_hostname VARCHAR(100);
    
    DECLARE shard_cursor CURSOR FOR
        SELECT id, hostname FROM shard_registry;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN shard_cursor;
    
    read_loop: LOOP
        FETCH shard_cursor INTO v_shard_id, v_hostname;
        
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        -- Connect to shard et collecter métriques
        -- (Via FEDERATED ou API externe)
        
        -- Exemple simplifié
        INSERT INTO shard_health 
        VALUES (v_shard_id, v_hostname, NOW(), 'OK', 
                12500000, 45.2, 35.5, 62.1, 50, 2);
        
    END LOOP;
    
    CLOSE shard_cursor;
    
END //
DELIMITER ;

-- Exécuter toutes les 5 minutes
CREATE EVENT monitor_shards_event
ON SCHEDULE EVERY 5 MINUTE
DO CALL monitor_shards();
```

### Alertes

```sql
-- Détecter déséquilibre entre shards
CREATE OR REPLACE VIEW v_shard_imbalance AS
SELECT 
    shard_id,
    row_count,
    AVG(row_count) OVER() as avg_rows,
    row_count - AVG(row_count) OVER() as diff_from_avg,
    ROUND((row_count - AVG(row_count) OVER()) * 100.0 / 
          NULLIF(AVG(row_count) OVER(), 0), 2) as imbalance_pct
FROM (
    SELECT shard_id, MAX(row_count) as row_count
    FROM shard_health
    WHERE checked_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
    GROUP BY shard_id
) latest;

-- Alerte si déséquilibre > 20%
SELECT * FROM v_shard_imbalance
WHERE ABS(imbalance_pct) > 20;
```

---

## Rebalancing

### Quand rebalancer ?

```
Indicateurs de besoin de rebalancing :
• Déséquilibre > 30% entre shards
• Un shard saturé (CPU > 80%, I/O > 90%)
• Croissance inégale des données
• Ajout de nouveaux shards
```

### Stratégie de rebalancing

```python
# Rebalancing progressif

class ShardRebalancer:
    def __init__(self, shards):
        self.shards = shards
        
    def analyze_imbalance(self):
        """Calculer déséquilibre actuel"""
        counts = {}
        for shard_id, config in enumerate(self.shards):
            conn = mysql.connector.connect(**config)
            cursor = conn.cursor()
            cursor.execute("SELECT COUNT(*) FROM orders")
            counts[shard_id] = cursor.fetchone()[0]
        
        avg = sum(counts.values()) / len(counts)
        imbalance = {
            sid: ((count - avg) / avg * 100)
            for sid, count in counts.items()
        }
        
        return imbalance
    
    def move_data(self, from_shard, to_shard, customer_ids):
        """Déplacer données entre shards"""
        # 1. Copier vers nouveau shard
        source = mysql.connector.connect(**self.shards[from_shard])
        dest = mysql.connector.connect(**self.shards[to_shard])
        
        source_cursor = source.cursor(dictionary=True)
        dest_cursor = dest.cursor()
        
        # Copy en batch
        for customer_id in customer_ids:
            source_cursor.execute(
                "SELECT * FROM orders WHERE customer_id = %s",
                (customer_id,)
            )
            rows = source_cursor.fetchall()
            
            if rows:
                # Insert dans destination
                for row in rows:
                    dest_cursor.execute(
                        "INSERT INTO orders VALUES (...)",
                        tuple(row.values())
                    )
                dest.commit()
                
                # Delete de source (après vérification)
                source_cursor.execute(
                    "DELETE FROM orders WHERE customer_id = %s",
                    (customer_id,)
                )
                source.commit()
    
    def rebalance(self, max_move_pct=10):
        """Rebalancer progressivement"""
        imbalance = self.analyze_imbalance()
        
        # Identifier shard surcharge et sous-charge
        overloaded = [(sid, pct) for sid, pct in imbalance.items() if pct > max_move_pct]
        underloaded = [(sid, pct) for sid, pct in imbalance.items() if pct < -max_move_pct]
        
        # Déplacer données
        for over_sid, over_pct in overloaded:
            for under_sid, under_pct in underloaded:
                # Calculer combien déplacer
                # Move customer_ids...
                pass
```

---

## Best Practices

### 1. Choix de la clé de sharding

```
✅ BONNES clés de sharding :
───────────────────────────

• customer_id / user_id (multi-tenant)
  → Isolement parfait
  → Queries locales au shard

• tenant_id (SaaS)
  → Un tenant = un shard possible
  → Migration tenant facile

• Géographie (country_code)
  → Conformité réglementaire
  → Latence optimisée

❌ MAUVAISES clés :
─────────────────

• Timestamp / date
  → Hot shard (toutes écritures sur shard actuel)

• Auto-increment ID
  → Distribution aléatoire mais queries difficiles

• Colonnes fréquemment modifiées
  → Reclassification difficile
```

### 2. Nombre de shards

```
Recommandations :
• Commencer petit : 2-4 shards
• Doubler si nécessaire : 4 → 8 → 16
• Éviter nombres premiers (rebalancing difficile)
• Maximum pratique : ~100 shards
• Au-delà : Considérer autre architecture
```

### 3. Haute disponibilité

```sql
-- Chaque shard avec réplication

Shard 0 :
├─ Primary (writes)
└─ Replicas (reads, failover)
   ├─ Replica 1
   └─ Replica 2

Shard 1 :
├─ Primary
└─ Replicas
   ├─ Replica 1
   └─ Replica 2
```

---

## ✅ Points clés à retenir

- 🔀 **Sharding = horizontal scaling** : Distribution données sur plusieurs serveurs
- 🎯 **Clé de sharding critique** : Détermine distribution et performance
- 🕷️ **Spider Engine** : Solution native MariaDB pour sharding
- 🔌 **ProxySQL** : Routing intelligent et load balancing
- ⚠️ **Jointures cross-shard = défi** : Denormalisation ou application-level
- 📊 **Monitoring essentiel** : Déséquilibre, performance par shard
- 🔄 **Rebalancing nécessaire** : Croissance inégale des shards
- 🚀 **Migration progressive** : Zero downtime avec dual-write
- 📍 **Sharding géographique** : Conformité GDPR, latence
- 🏢 **Multi-tenancy** : Pattern idéal pour SaaS

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 Spider Storage Engine](https://mariadb.com/kb/en/spider-storage-engine-overview/)
- [📖 Spider Partitioning](https://mariadb.com/kb/en/spider-storage-engine-partitioning/)

### ProxySQL

- [📖 ProxySQL Documentation](https://proxysql.com/documentation/)
- [📖 ProxySQL Query Rules](https://proxysql.com/documentation/query-rules/)

### Architectures distribuées

- [Sharding Best Practices](https://www.percona.com/blog/sharding-best-practices/)
- [Database Sharding Explained](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)

---

*Le sharding est une technique puissante mais complexe. À n'utiliser que lorsque véritablement nécessaire, après avoir épuisé les options de vertical scaling et d'optimisation.*

⏭️ [Benchmarking](/15-performance-tuning/12-benchmarking.md)

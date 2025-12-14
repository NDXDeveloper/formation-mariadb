🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.9 Partitionnement de tables

> **Niveau** : Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : 
> - Sections 15.1-15.8 (Méthodologie, Mémoire, I/O, InnoDB, Analyse)
> - Maîtrise des index et optimisation de requêtes
> - Compréhension de l'architecture InnoDB
> - Expérience avec grandes tables (10M+ lignes)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre le partitionnement** et son fonctionnement interne
- **Identifier les cas d'usage** appropriés pour le partitionnement
- **Choisir le type de partitionnement** adapté (RANGE, LIST, HASH, KEY)
- **Concevoir une stratégie** de partitionnement optimale
- **Exploiter le partition pruning** pour des gains de performance
- **Gérer les conversions** partition ↔ table ordinaire
- **Maintenir les partitions** en production (ajout, suppression, réorganisation)
- **Éviter les pièges** courants du partitionnement
- **Appliquer les best practices** pour grandes tables partitionnées
- **Monitorer les performances** des tables partitionnées

---

## Introduction

Le **partitionnement** est une technique qui divise une grande table en **sous-tables plus petites** (partitions) tout en la présentant comme une table unique à l'application.

### Qu'est-ce que le partitionnement ?

```
┌────────────────────────────────────────────────────┐
│  TABLE NON PARTITIONNÉE                            │
├────────────────────────────────────────────────────┤
│                                                    │
│  orders (500 millions lignes, 400 GB)              │
│  ┌──────────────────────────────────────────────┐  │
│  │ id | customer_id | order_date | amount | ... │  │
│  │  1 | 123         | 2020-01-15 | 99.99  | ... │  │
│  │  2 | 456         | 2020-03-22 | 149.00 | ... │  │
│  │ ...                                          │  │
│  │ 500,000,000 rows                             │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  Problèmes :                                       │
│  • Requêtes lentes (scan de 500M lignes)           │
│  • Index énormes (> 100 GB)                        │
│  • Maintenance difficile (OPTIMIZE = jours)        │
│  • Archivage complexe                              │
│                                                    │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  TABLE PARTITIONNÉE (RANGE par année)              │
├────────────────────────────────────────────────────┤
│                                                    │
│  orders (même 500M lignes, 400 GB)                 │
│  ├─ Partition p2020 (125M lignes, 100 GB)          │
│  ├─ Partition p2021 (125M lignes, 100 GB)          │
│  ├─ Partition p2022 (125M lignes, 100 GB)          │
│  └─ Partition p2023 (125M lignes, 100 GB)          │
│                                                    │
│  Avantages :                                       │
│  ✅ Requête sur 2023 : Scan 125M lignes (vs 500M)  │
│  ✅ Index plus petits par partition                │
│  ✅ Maintenance partition par partition            │
│  ✅ Archivage = DROP/TRUNCATE partition            │
│  ✅ Partition pruning automatique                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Principe fondamental

Le partitionnement divise physiquement les données mais maintient une **interface unique** :

```sql
-- Pour l'application : Table unique
SELECT * FROM orders WHERE order_date = '2023-05-15';

-- MariaDB en interne : Accès seulement partition p2023
-- Partition pruning automatique
-- Scan de 125M lignes au lieu de 500M

-- Transparent pour l'application !
```

---

## Quand utiliser le partitionnement ?

### Cas d'usage appropriés ✅

```
┌────────────────────────────────────────────────────┐
│  SCÉNARIOS OÙ LE PARTITIONNEMENT AIDE              │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. TRÈS GRANDES TABLES (100M+ lignes)             │
│     • Table logs : 1 milliard de lignes            │
│     • Table transactions : 500M lignes             │
│     • Table events : Croissance illimitée          │
│                                                    │
│  2. DONNÉES AVEC DIMENSION TEMPORELLE              │
│     • Requêtes souvent filtrées par date           │
│     • Archivage périodique nécessaire              │
│     • Purge de données anciennes                   │
│     Exemple : logs, commandes, événements          │
│                                                    │
│  3. REQUÊTES PRÉDICTIBLES                          │
│     • WHERE toujours sur clé de partition          │
│     • Patterns d'accès connus                      │
│     Exemple : WHERE order_date BETWEEN ... AND     │
│                                                    │
│  4. MAINTENANCE FACILITÉE                          │
│     • OPTIMIZE table trop long (jours)             │
│     • Besoin de maintenance par tranches           │
│     • Rebuild d'index progressif                   │
│                                                    │
│  5. DISTRIBUTION DE CHARGE                         │
│     • Plusieurs disques/tablespaces                │
│     • Distribution I/O                             │
│     • Parallélisation possible                     │
│                                                    │
│  6. ARCHIVAGE/PURGE EFFICACE                       │
│     • DROP partition vs DELETE millions lignes     │
│     • Archivage partition complète                 │
│     • Performances : 1s vs plusieurs heures        │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Cas où NE PAS partitionner ❌

```
┌────────────────────────────────────────────────────┐
│  SCÉNARIOS OÙ LE PARTITIONNEMENT N'AIDE PAS        │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. PETITES TABLES (<1M lignes)                    │
│     ❌ Overhead du partitionnement > bénéfice      │
│     ✅ Mieux : Bons index suffisent                │
│                                                    │
│  2. REQUÊTES SANS FILTRE SUR CLÉ PARTITION         │
│     ❌ SELECT * FROM orders WHERE customer_id = ?  │
│        → Scan toutes partitions !                  │
│     ✅ Pas de partition pruning                    │
│                                                    │
│  3. PATTERNS D'ACCÈS IMPRÉVISIBLES                 │
│     ❌ Requêtes aléatoires sur toute la table      │
│     ❌ Pas de dimension évidente (temps, région)   │
│                                                    │
│  4. JOINTURES COMPLEXES                            │
│     ❌ JOIN entre tables partitionnées             │
│     ❌ Performance peut se dégrader                │
│                                                    │
│  5. CLÉS ÉTRANGÈRES REQUISES                       │
│     ❌ InnoDB ne supporte pas FK sur partitions    │
│     ❌ Doit désactiver FOREIGN KEY                 │
│                                                    │
│  6. RECHERCHE DE "QUICK FIX"                       │
│     ❌ Partitionnement n'est pas une baguette      │
│        magique                                     │
│     ✅ Analyser d'abord requêtes et index          │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Matrice de décision

```
                TAILLE TABLE
                │
                │         > 100M lignes
                │         ┌─────────────────────────┐
                │         │                         │
                │         │  Dimension temporelle ? │
                │         │  OUI → PARTITIONNER ✅  │
                │         │  NON → Analyser plus    │
                │         │                         │
                │         └─────────────────────────┘
                │
   1M-100M      │  ┌──────────────────────────────┐
                │  │ Requêtes sur plage dates ?   │
                │  │ OUI → CONSIDÉRER             │
                │  │ NON → Probablement pas       │
                │  └──────────────────────────────┘
                │
   < 1M         │  ┌──────────────────────────────┐
                │  │ NE PAS PARTITIONNER ❌       │
                │  │ Index suffisent              │
                │  └──────────────────────────────┘
                │
                └─────────────────────────────────────→
```

---

## Les 4 types de partitionnement

### Vue d'ensemble comparative

```
┌──────────────────────────────────────────────────────────────┐
│  TYPE         PRINCIPE           CAS D'USAGE       EXEMPLE   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  RANGE        Plages de valeurs  • Données temporelles       │
│                                  • Séries numériques         │
│                                  • Archivage périodique      │
│               Exemple : Par année, par mois, par ID          │
│                                                              │
│  LIST         Valeurs discrètes  • Catégories finies         │
│                                  • Régions/pays              │
│                                  • Statuts                   │
│               Exemple : Par pays, par statut, par type       │
│                                                              │
│  HASH         Hash automatique   • Distribution uniforme     │
│                                  • Aucune dimension          │
│                                  • Load balancing            │
│               Exemple : Par customer_id hashé                │
│                                                              │
│  KEY          Hash MariaDB       • Similaire HASH            │
│                                  • Hash natif MariaDB        │
│                                  • Clé primaire composite    │
│               Exemple : Par clé primaire                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### RANGE Partitioning

**Le plus courant et le plus utile.**

```sql
-- Exemple : Table orders partitionnée par année
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    PRIMARY KEY (id, order_date)  -- order_date DOIT être dans PK
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Requête optimisée automatiquement
SELECT * FROM orders WHERE order_date BETWEEN '2023-01-01' AND '2023-12-31';
-- Accède seulement partition p2023 ✅
```

**Avantages** :
- Partition pruning très efficace
- Archivage facile (DROP PARTITION)
- Maintenance par partition
- Gestion de l'historique simplifiée

**Cas d'usage** :
- Tables avec date/timestamp
- Données historiques
- Logs, événements, transactions

### LIST Partitioning

**Pour valeurs discrètes et connues.**

```sql
-- Exemple : Table customers partitionnée par pays
CREATE TABLE customers (
    id INT AUTO_INCREMENT,
    name VARCHAR(100),
    country_code CHAR(2) NOT NULL,
    email VARCHAR(100),
    PRIMARY KEY (id, country_code)
) ENGINE=InnoDB
PARTITION BY LIST COLUMNS(country_code) (
    PARTITION p_europe VALUES IN ('FR', 'DE', 'IT', 'ES', 'UK'),
    PARTITION p_americas VALUES IN ('US', 'CA', 'BR', 'MX'),
    PARTITION p_asia VALUES IN ('CN', 'JP', 'IN', 'KR'),
    PARTITION p_other VALUES IN ('AU', 'ZA', 'RU')
);

-- Requête optimisée
SELECT * FROM customers WHERE country_code = 'FR';
-- Accède seulement partition p_europe ✅
```

**Avantages** :
- Distribution logique par catégorie
- Facilite analyse par segment
- Maintenance ciblée

**Cas d'usage** :
- Données géographiques (pays, régions)
- Catégories produits
- Statuts, types

### HASH Partitioning

**Distribution automatique uniforme.**

```sql
-- Exemple : Table users partitionnée par hash de customer_id
CREATE TABLE users (
    id INT AUTO_INCREMENT,
    customer_id INT NOT NULL,
    username VARCHAR(50),
    created_at TIMESTAMP,
    PRIMARY KEY (id, customer_id)
) ENGINE=InnoDB
PARTITION BY HASH(customer_id)
PARTITIONS 8;  -- 8 partitions

-- Distribution automatique
-- customer_id 123 → partition HASH(123) MOD 8 = partition 3
-- customer_id 456 → partition HASH(456) MOD 8 = partition 0
```

**Avantages** :
- Distribution uniforme automatique
- Pas besoin de définir ranges/listes
- Bon pour load balancing

**Inconvénients** :
- Pas d'archivage facile
- Partition pruning limité
- Difficile à maintenir

**Cas d'usage** :
- Aucune dimension évidente
- Distribution pure de charge
- Tables de cache

### KEY Partitioning

**Similaire à HASH mais utilise fonction hash native MariaDB.**

```sql
-- Exemple : Partition par clé primaire composite
CREATE TABLE sessions (
    session_id VARCHAR(64),
    user_id INT,
    data TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (session_id, user_id)
) ENGINE=InnoDB
PARTITION BY KEY(session_id, user_id)
PARTITIONS 16;

-- Hash calculé par MariaDB sur (session_id, user_id)
```

**Différence avec HASH** :
- Hash algorithm de MariaDB (plus rapide)
- Supporte colonnes non-integer
- Recommandé vs HASH dans la plupart des cas

---

## Partition Pruning : Le cœur de la performance

### Qu'est-ce que le partition pruning ?

```
┌────────────────────────────────────────────────────┐
│         PARTITION PRUNING (Élagage)                │
├────────────────────────────────────────────────────┤
│                                                    │
│  Optimization où MariaDB détermine quelles         │
│  partitions accéder, ignorant les autres.          │
│                                                    │
│  Sans pruning :                                    │
│  ──────────────                                    │
│  SELECT * FROM orders WHERE customer_id = 123;     │
│  → Scan TOUTES les partitions (p2020...p2024)      │
│  → Pas d'amélioration performance                  │
│                                                    │
│  Avec pruning :                                    │
│  ─────────────                                     │
│  SELECT * FROM orders                              │
│  WHERE order_date = '2023-05-15';                  │
│  → Scan SEULEMENT partition p2023                  │
│  → 5x plus rapide (si 5 partitions)                │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Vérifier le partition pruning

```sql
-- Utiliser EXPLAIN PARTITIONS (ou EXPLAIN)
EXPLAIN PARTITIONS
SELECT * FROM orders 
WHERE order_date BETWEEN '2023-01-01' AND '2023-12-31';

/*
+----+-------------+--------+------------+------+---------------+
| id | select_type | table  | partitions | type | key           |
+----+-------------+--------+------------+------+---------------+
|  1 | SIMPLE      | orders | p2023      | ALL  | NULL          |
+----+-------------+--------+------------+------+---------------+

Colonne "partitions" = p2023
→ Partition pruning ACTIF ✅
Seulement partition p2023 accédée
*/

-- Contre-exemple : Requête sans filtre sur clé de partition
EXPLAIN PARTITIONS
SELECT * FROM orders WHERE customer_id = 123;

/*
+----+-------------+--------+------------------------------+
| id | select_type | table  | partitions                   |
+----+-------------+--------+------------------------------+
|  1 | SIMPLE      | orders | p2020,p2021,p2022,p2023,p2024|
+----+-------------+--------+------------------------------+

Toutes les partitions accédées
→ Partition pruning INACTIF ❌
Pas de gain de performance
*/
```

### Conditions pour partition pruning efficace

```sql
-- ✅ BON : Pruning actif
SELECT * FROM orders 
WHERE order_date = '2023-05-15';
-- → Partition p2023 uniquement

SELECT * FROM orders 
WHERE order_date >= '2023-01-01' AND order_date < '2024-01-01';
-- → Partition p2023 uniquement

SELECT * FROM orders 
WHERE YEAR(order_date) = 2023;
-- → Partition p2023 uniquement (MariaDB optimise)

-- ❌ MAUVAIS : Pruning inactif
SELECT * FROM orders 
WHERE customer_id = 123;
-- → Toutes partitions (customer_id n'est pas clé partition)

SELECT * FROM orders 
WHERE MONTH(order_date) = 5;
-- → Toutes partitions (fonction complexe)

SELECT * FROM orders 
WHERE amount > 1000;
-- → Toutes partitions (amount n'est pas clé partition)
```

---

## Architecture et stockage

### Organisation physique

```
┌────────────────────────────────────────────────────┐
│  STOCKAGE PHYSIQUE DES PARTITIONS                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  Table : orders (partitionnée RANGE par année)     │
│                                                    │
│  Fichiers sur disque :                             │
│  ├── orders#P#p2020.ibd  (100 GB)                  │
│  ├── orders#P#p2021.ibd  (100 GB)                  │
│  ├── orders#P#p2022.ibd  (100 GB)                  │
│  ├── orders#P#p2023.ibd  (100 GB)                  │
│  └── orders#P#p2024.ibd  (100 GB)                  │
│                                                    │
│  Chaque partition = fichier .ibd séparé            │
│                                                    │
│  Avantages :                                       │
│  • I/O distribués sur plusieurs fichiers           │
│  • Possibilité de fichiers sur disques différents  │
│  • Maintenance fichier par fichier                 │
│  • Drop partition = suppression fichier (rapide)   │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Index sur tables partitionnées

```sql
-- IMPORTANT : Les index sont LOCAUX à chaque partition

CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, order_date),
    INDEX idx_customer (customer_id, order_date)  -- Index LOCAL
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Résultat :
-- Partition p2023 a son propre idx_customer
-- Partition p2024 a son propre idx_customer
-- Pas d'index global sur toute la table

-- Implication pour requêtes
SELECT * FROM orders WHERE customer_id = 123;
-- Doit scanner idx_customer de CHAQUE partition
-- Pas de partition pruning possible
```

**Règle d'or** : La clé de partition DOIT être incluse dans la PRIMARY KEY et tous les UNIQUE indexes.

```sql
-- ❌ INVALIDE
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- id seul dans PK
    order_date DATE NOT NULL
) PARTITION BY RANGE (YEAR(order_date)) (...);
-- Erreur : A PRIMARY KEY must include all columns in the table's partitioning function

-- ✅ VALIDE
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    order_date DATE NOT NULL,
    PRIMARY KEY (id, order_date)  -- order_date inclus
) PARTITION BY RANGE (YEAR(order_date)) (...);
```

---

## Gestion des partitions

### Ajout de partitions

```sql
-- Ajouter partition pour nouvelle année
ALTER TABLE orders 
ADD PARTITION (
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- Ajouter plusieurs partitions
ALTER TABLE orders 
ADD PARTITION (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p2027 VALUES LESS THAN (2028)
);
```

### Suppression de partitions

```sql
-- Supprimer partition (EFFACE les données!)
ALTER TABLE orders DROP PARTITION p2020;
-- Équivalent à : DELETE FROM orders WHERE YEAR(order_date) = 2020
-- Mais instantané (suppression fichier) vs heures de DELETE

-- Archiver avant suppression
CREATE TABLE orders_archive_2020 LIKE orders REMOVE PARTITIONING;
ALTER TABLE orders 
EXCHANGE PARTITION p2020 WITH TABLE orders_archive_2020;
-- Maintenant orders_archive_2020 contient données 2020
-- orders.p2020 est vide
DROP TABLE orders_archive_2020;  -- Si archivage pas nécessaire
```

### Réorganisation de partitions

```sql
-- Fusionner partitions
ALTER TABLE orders 
REORGANIZE PARTITION p2020, p2021 INTO (
    PARTITION p2020_2021 VALUES LESS THAN (2022)
);

-- Diviser partition
ALTER TABLE orders 
REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### TRUNCATE partition

```sql
-- Vider partition (plus rapide que DELETE)
ALTER TABLE orders TRUNCATE PARTITION p2023;
-- Équivalent à : DELETE FROM orders WHERE YEAR(order_date) = 2023
-- Mais instantané
```

### EXCHANGE partition (conversion partition ↔ table)

```sql
-- Convertir partition en table ordinaire
CREATE TABLE orders_2023 LIKE orders REMOVE PARTITIONING;

ALTER TABLE orders 
EXCHANGE PARTITION p2023 WITH TABLE orders_2023;

-- Maintenant :
-- orders_2023 = table ordinaire avec données 2023
-- orders.p2023 = partition vide

-- Use cases :
-- 1. Archivage long terme (compression, stockage froid)
-- 2. Analyse détaillée sur subset de données
-- 3. Migration progressive
```

---

## Conversions avancées

### Table ordinaire → Table partitionnée

```sql
-- Table existante non partitionnée
CREATE TABLE orders_old (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2)
) ENGINE=InnoDB;

-- Conversion en table partitionnée
-- Méthode 1 : ALTER TABLE (bloque table pendant conversion)
ALTER TABLE orders_old
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
-- ⚠️ Peut prendre des heures sur grande table
-- ⚠️ Table lockée pendant l'opération

-- Méthode 2 : Créer nouvelle table + copie + bascule (recommandé)
CREATE TABLE orders_new (
    id BIGINT AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, order_date)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Copie données (peut être fait par batch)
INSERT INTO orders_new SELECT * FROM orders_old;

-- Bascule
RENAME TABLE orders_old TO orders_backup, orders_new TO orders;

-- Cleanup
DROP TABLE orders_backup;
```

### Table partitionnée → Table ordinaire

```sql
-- Supprimer le partitionnement
ALTER TABLE orders REMOVE PARTITIONING;
-- Fusionne toutes partitions en table unique
-- ⚠️ Opération longue et bloquante
```

---

## Maintenance des tables partitionnées

### OPTIMIZE par partition

```sql
-- Optimiser une partition spécifique
ALTER TABLE orders OPTIMIZE PARTITION p2023;
-- Plus rapide que OPTIMIZE TABLE orders (toutes partitions)

-- Optimiser plusieurs partitions
ALTER TABLE orders OPTIMIZE PARTITION p2022, p2023;
```

### ANALYZE par partition

```sql
-- Mettre à jour statistiques partition
ALTER TABLE orders ANALYZE PARTITION p2023;

-- Toutes partitions
ANALYZE TABLE orders;
```

### CHECK et REPAIR

```sql
-- Vérifier intégrité partition
ALTER TABLE orders CHECK PARTITION p2023;

-- Réparer si nécessaire
ALTER TABLE orders REPAIR PARTITION p2023;
```

### Automatisation maintenance

```sql
-- Procédure mensuelle de maintenance
DELIMITER //
CREATE OR REPLACE PROCEDURE maintain_orders_partitions()
BEGIN
    DECLARE v_current_year INT;
    DECLARE v_partition_name VARCHAR(64);
    
    SET v_current_year = YEAR(NOW());
    SET v_partition_name = CONCAT('p', v_current_year);
    
    -- 1. Ajouter partition pour année prochaine si inexistante
    SET @sql = CONCAT('ALTER TABLE orders ADD PARTITION IF NOT EXISTS (',
                      'PARTITION p', v_current_year + 1, 
                      ' VALUES LESS THAN (', v_current_year + 2, '))');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 2. Supprimer partitions > 3 ans (archivage)
    SET @sql = CONCAT('ALTER TABLE orders DROP PARTITION IF EXISTS p',
                      v_current_year - 3);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 3. Optimiser partition année courante
    SET @sql = CONCAT('ALTER TABLE orders OPTIMIZE PARTITION ', v_partition_name);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //
DELIMITER ;

-- Planifier exécution mensuelle
CREATE EVENT IF NOT EXISTS monthly_partition_maintenance
ON SCHEDULE EVERY 1 MONTH
STARTS '2024-01-01 02:00:00'
DO CALL maintain_orders_partitions();
```

---

## Monitoring des partitions

### Informations sur partitions

```sql
-- Lister toutes les partitions d'une table
SELECT 
    PARTITION_NAME,
    PARTITION_METHOD,
    PARTITION_EXPRESSION,
    PARTITION_DESCRIPTION,
    TABLE_ROWS,
    AVG_ROW_LENGTH,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) as data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) as index_mb,
    PARTITION_COMMENT
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
AND TABLE_NAME = 'orders'
ORDER BY PARTITION_ORDINAL_POSITION;

/*
Résultat exemple :
+----------------+--------+-----------+---------+----------+---------+
| PARTITION_NAME | METHOD | TABLE_ROWS| data_mb | index_mb |         |
+----------------+--------+-----------+---------+----------+---------+
| p2020          | RANGE  | 125000000 | 98234   | 12456    |         |
| p2021          | RANGE  | 125000000 | 98156   | 12389    |         |
| p2022          | RANGE  | 125000000 | 98445   | 12512    |         |
| p2023          | RANGE  | 125000000 | 98678   | 12589    |         |
+----------------+--------+-----------+---------+----------+---------+
*/
```

### Distribution des données

```sql
-- Vérifier distribution équilibrée
SELECT 
    PARTITION_NAME,
    TABLE_ROWS,
    ROUND(TABLE_ROWS * 100.0 / SUM(TABLE_ROWS) OVER(), 2) as pct_of_total,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) as data_mb
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
AND TABLE_NAME = 'orders'
AND PARTITION_NAME IS NOT NULL
ORDER BY PARTITION_ORDINAL_POSITION;

-- Alerte si distribution déséquilibrée (pour HASH partitions)
```

### Performance par partition

```sql
-- Via Performance Schema
SELECT 
    OBJECT_NAME,
    INDEX_NAME,
    COUNT_FETCH,
    COUNT_INSERT,
    COUNT_UPDATE,
    COUNT_DELETE,
    SUM_TIMER_FETCH / 1000000000000 as fetch_time_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'mydb'
AND OBJECT_NAME LIKE 'orders#P#%'  -- Partitions
ORDER BY fetch_time_sec DESC
LIMIT 20;
```

---

## Best Practices

### 1. Choix de la clé de partition

```
✅ BONNES clés de partition :
───────────────────────────

• Date/Timestamp : 90% des cas
  → Requêtes naturellement filtrées par date
  → Archivage évident

• Géographie (pays, région) : Si requêtes par région
  → Données localisées
  → Conformité réglementaire (RGPD)

• Catégorie stable : Si nombre limité
  → Type de produit
  → Statut (actif/archivé)

❌ MAUVAISES clés de partition :
─────────────────────────────

• Colonne rarement utilisée en WHERE
• ID auto-increment (sauf HASH/KEY pour load balancing)
• Colonnes avec trop de valeurs distinctes (LIST)
• Colonnes fréquemment modifiées
```

### 2. Nombre de partitions

```sql
-- Trop peu de partitions : Bénéfice limité
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p_old VALUES LESS THAN (2020),
    PARTITION p_recent VALUES LESS THAN MAXVALUE
);
-- Seulement 2 partitions → Peu d'amélioration

-- Trop de partitions : Overhead
PARTITION BY RANGE (DAY(order_date)) (
    PARTITION p_day1 VALUES LESS THAN (2),
    PARTITION p_day2 VALUES LESS THAN (3),
    ...
    PARTITION p_day31 VALUES LESS THAN (32)
);
-- 31 partitions → Overhead metadata, gestion complexe

-- Recommandation : 10-50 partitions
-- Sweet spot : 12-24 partitions (mois ou trimestres)
```

### 3. Stratégie d'archivage

```sql
-- Pattern recommandé : Partition roulante

-- Automatiser :
-- 1. Ajout partition future (1 mois avant)
-- 2. Archivage partition > N ans
-- 3. Suppression partition archivée

-- Exemple : Conserver 3 ans en ligne
-- Archiver > 3 ans dans table archive
-- Supprimer > 5 ans

CREATE EVENT monthly_partition_rollover
ON SCHEDULE EVERY 1 MONTH
DO
BEGIN
    -- Ajouter partition année prochaine
    CALL add_future_partition();
    
    -- Archiver partitions > 3 ans
    CALL archive_old_partitions(3);
    
    -- Supprimer partitions archivées > 5 ans
    CALL cleanup_archived_partitions(5);
END;
```

### 4. Testing avant production

```sql
-- TOUJOURS tester sur dataset réaliste

-- 1. Créer table test avec partitionnement
CREATE TABLE orders_test LIKE orders;
-- Ajouter partitionnement
ALTER TABLE orders_test PARTITION BY ...;

-- 2. Copier échantillon représentatif
INSERT INTO orders_test 
SELECT * FROM orders 
WHERE order_date >= DATE_SUB(NOW(), INTERVAL 2 YEAR);

-- 3. Benchmarker requêtes typiques
-- Sans partitionnement
SELECT COUNT(*) FROM orders WHERE order_date = '2023-05-15';

-- Avec partitionnement
SELECT COUNT(*) FROM orders_test WHERE order_date = '2023-05-15';

-- Comparer performance
```

### 5. Documentation

```sql
-- Documenter stratégie de partitionnement

-- Table de métadonnées
CREATE TABLE partition_metadata (
    table_name VARCHAR(64),
    partition_key VARCHAR(64),
    partition_type ENUM('RANGE', 'LIST', 'HASH', 'KEY'),
    retention_policy VARCHAR(255),
    maintenance_schedule VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT
);

INSERT INTO partition_metadata VALUES
('orders', 'order_date', 'RANGE', 
 'Keep 3 years online, archive > 3 years, delete > 5 years',
 'Monthly: add future partition, archive old, optimize current',
 NOW(),
 'Partitioned by year for efficient archival and query performance');
```

---

## ✅ Points clés à retenir

- 📊 **Partitionnement ≠ magie** : Utile pour très grandes tables (100M+ lignes) avec dimension temporelle
- 🎯 **Partition pruning = clé** : Requêtes DOIVENT filtrer sur clé partition
- 📅 **RANGE = le plus courant** : 90% des cas d'usage (données temporelles)
- 🔑 **Clé partition dans PK** : Obligatoire pour PRIMARY KEY et UNIQUE indexes
- 📁 **1 partition = 1 fichier** : Distribution I/O, maintenance ciblée
- 🗑️ **Archivage efficient** : DROP PARTITION vs DELETE (1s vs heures)
- ⚠️ **Foreign keys incompatibles** : InnoDB ne supporte pas FK sur tables partitionnées
- 📈 **10-50 partitions** : Sweet spot pour overhead vs bénéfice
- 🔄 **EXCHANGE partition** : Conversion partition ↔ table pour archivage
- 📝 **Automatiser maintenance** : Events pour ajout/suppression/optimisation

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Partitioning Overview](https://mariadb.com/kb/en/partitioning-overview/)
- [📖 Partition Types](https://mariadb.com/kb/en/partition-types/)
- [📖 Partitioning Limitations](https://mariadb.com/kb/en/partitioning-limitations/)

### Guides avancés

- [Percona - Partitioning Best Practices](https://www.percona.com/blog/partitioning-best-practices-mysql/)
- [MySQL Partitioning (compatible)](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent chaque type de partitionnement :

### **Section 15.9.1** : [RANGE Partitioning](/15-performance-tuning/09.1-range-partitioning.md)
*Partitionnement par plages de valeurs. Configuration détaillée, cas d'usage, optimisations temporelles.*

### **Section 15.9.2** : [LIST Partitioning](/15-performance-tuning/09.2-list-partitioning.md)
*Partitionnement par listes de valeurs discrètes. Géographie, catégories, statuts.*

### **Section 15.9.3** : [HASH et KEY Partitioning](/15-performance-tuning/09.3-hash-key-partitioning.md)
*Partitionnement par hash pour distribution uniforme. Load balancing et parallélisation.*

### **Section 15.9.4** : [Gestion avancée des partitions](/15-performance-tuning/09.4-gestion-avancee-partitions.md)
*Maintenance automatisée, monitoring, migrations, troubleshooting en production.*

---

*Le partitionnement est un outil puissant mais qui doit être utilisé judicieusement. La règle : "Partitionner parce que nécessaire, pas parce que possible."*

⏭️ [RANGE partitioning](/15-performance-tuning/09.1-range-partitioning.md)

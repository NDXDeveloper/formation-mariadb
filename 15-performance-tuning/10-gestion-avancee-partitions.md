🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.10 Gestion avancée des partitions (conversion partition↔table)

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Section 15.9 (Partitionnement de tables - vue d'ensemble)
> - Expérience en gestion de grandes tables (100M+ lignes)
> - Compréhension des locks InnoDB
> - Pratique des migrations en production

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Maîtriser EXCHANGE PARTITION** pour conversions partition ↔ table
- **Implémenter des stratégies d'archivage** efficaces et sans downtime
- **Gérer les migrations** de tables partitionnées en production
- **Automatiser la maintenance** des partitions (ajout, suppression, rotation)
- **Réorganiser les partitions** à chaud sans impact utilisateur
- **Diagnostiquer et corriger** les problèmes de partitionnement
- **Optimiser les performances** des opérations de gestion
- **Appliquer les patterns** de gestion avancés en production
- **Gérer les erreurs** et rollback de manière sécurisée

---

## Introduction

La **gestion avancée des partitions** va au-delà de la simple création de tables partitionnées. Elle englobe les opérations critiques en production :

- 🔄 **Conversions** partition ↔ table pour archivage et analyse
- 🗑️ **Archivage et purge** de données historiques
- 🔧 **Maintenance** automatisée sans impact
- 📊 **Réorganisation** pour optimisation
- 🚨 **Gestion d'erreurs** et recovery

### Pourquoi la gestion avancée est critique

```
┌────────────────────────────────────────────────────┐
│  SCÉNARIO PRODUCTION TYPIQUE                       │
├────────────────────────────────────────────────────┤
│                                                    │
│  Table : orders (partitionnée par année)           │
│  Croissance : 100M lignes/an                       │
│  Rétention : 3 ans en ligne                        │
│                                                    │
│  Besoins :                                         │
│  • Ajouter partition 2025 (avant fin 2024)         │
│  • Archiver partition 2021 (> 3 ans)               │
│  • Analyser données 2020 séparément                │
│  • Optimiser partition 2024 (active)               │
│  • Gérer croissance sans downtime                  │
│                                                    │
│  Sans gestion avancée :                            │
│  ❌ Opérations manuelles risquées                  │
│  ❌ Downtime pour maintenance                      │
│  ❌ DELETE lent pour purge (heures)                │
│  ❌ Pas d'archivage structuré                      │
│                                                    │
│  Avec gestion avancée :                            │
│  ✅ Automatisation complète                        │
│  ✅ Zero downtime                                  │
│  ✅ Archivage instantané (EXCHANGE)                │
│  ✅ Rollback possible                              │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## EXCHANGE PARTITION : La commande clé

### Concept et fonctionnement

**EXCHANGE PARTITION** permet d'échanger instantanément une partition avec une table ordinaire.

```
┌────────────────────────────────────────────────────┐
│         EXCHANGE PARTITION : MÉCANISME             │
├────────────────────────────────────────────────────┤
│                                                    │
│  AVANT :                                           │
│  ───────                                           │
│  Table partitionnée : orders                       │
│    ├─ p2023 (100M lignes) ← Partition à archiver   │
│    └─ p2024 (50M lignes)                           │
│                                                    │
│  Table vide : orders_archive_2023 (0 lignes)       │
│                                                    │
│  COMMANDE :                                        │
│  ─────────                                         │
│  ALTER TABLE orders                                │
│  EXCHANGE PARTITION p2023                          │
│  WITH TABLE orders_archive_2023;                   │
│                                                    │
│  APRÈS :                                           │
│  ──────                                            │
│  Table partitionnée : orders                       │
│    ├─ p2023 (0 lignes) ← Partition vidée           │
│    └─ p2024 (50M lignes)                           │
│                                                    │
│  Table archivée : orders_archive_2023 (100M)       │
│                                                    │
│  Durée : ~1 seconde (swap de métadonnées)          │
│  Impact : Minimal (metadata lock)                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Syntaxe et prérequis

```sql
-- Syntaxe de base
ALTER TABLE table_partitionnee
EXCHANGE PARTITION nom_partition
WITH TABLE table_ordinaire;

-- Prérequis CRITIQUES :
-- 1. Table ordinaire DOIT avoir même structure que partition
-- 2. Table ordinaire DOIT être vide (avant échange)
-- 3. Table ordinaire ne doit PAS être partitionnée
-- 4. Même ENGINE (InnoDB)
-- 5. Même ROW_FORMAT
-- 6. Pas de FOREIGN KEY sur table ordinaire
```

### Exemple complet : Archivage partition

```sql
-- Contexte : Archiver partition p2020 de la table orders

-- Étape 1 : Créer table archive avec structure identique
CREATE TABLE orders_archive_2020 LIKE orders;

-- Étape 2 : Supprimer le partitionnement de la table archive
ALTER TABLE orders_archive_2020 REMOVE PARTITIONING;

-- Étape 3 : Vérifier que structures correspondent
SHOW CREATE TABLE orders\G
SHOW CREATE TABLE orders_archive_2020\G
-- Doivent être identiques (sauf partitionnement)

-- Étape 4 : EXCHANGE partition avec table archive
ALTER TABLE orders 
EXCHANGE PARTITION p2020 
WITH TABLE orders_archive_2020;
-- Durée : ~1 seconde
-- orders.p2020 est maintenant vide
-- orders_archive_2020 contient toutes données 2020

-- Étape 5 : Vérifier le résultat
SELECT COUNT(*) FROM orders PARTITION (p2020);
-- 0

SELECT COUNT(*) FROM orders_archive_2020;
-- 125000000 (exemple)

-- Étape 6 : Optionnel - Compresser table archive
ALTER TABLE orders_archive_2020 
ROW_FORMAT=COMPRESSED 
KEY_BLOCK_SIZE=8;

-- Étape 7 : Optionnel - Déplacer vers stockage froid
-- Exporter puis supprimer de production
```

### Exemple inverse : Restaurer partition depuis archive

```sql
-- Contexte : Besoin temporaire de données 2020 pour analyse

-- Étape 1 : Vérifier que partition cible est vide
SELECT COUNT(*) FROM orders PARTITION (p2020);
-- Doit être 0

-- Étape 2 : Créer table temporaire pour swap
CREATE TABLE orders_temp_2020 LIKE orders;
ALTER TABLE orders_temp_2020 REMOVE PARTITIONING;

-- Étape 3 : Copier données archive vers temp
INSERT INTO orders_temp_2020 SELECT * FROM orders_archive_2020;

-- Étape 4 : EXCHANGE temp avec partition
ALTER TABLE orders 
EXCHANGE PARTITION p2020 
WITH TABLE orders_temp_2020;

-- Maintenant :
-- orders.p2020 contient données 2020
-- orders_temp_2020 est vide

-- Étape 5 : Cleanup
DROP TABLE orders_temp_2020;

-- Données 2020 restaurées dans partition !
```

---

## Stratégies d'archivage avancées

### Pattern 1 : Archivage mensuel automatisé

```sql
-- Procédure complète d'archivage mensuel

DELIMITER //
CREATE OR REPLACE PROCEDURE archive_old_partitions(
    IN p_table_name VARCHAR(64),
    IN p_retention_months INT
)
BEGIN
    DECLARE v_current_date DATE;
    DECLARE v_archive_threshold DATE;
    DECLARE v_partition_name VARCHAR(64);
    DECLARE v_archive_table_name VARCHAR(64);
    DECLARE v_year INT;
    DECLARE v_month INT;
    DECLARE v_partition_exists INT;
    DECLARE done INT DEFAULT FALSE;
    
    -- Curseur pour partitions à archiver
    DECLARE partition_cursor CURSOR FOR
        SELECT PARTITION_NAME
        FROM information_schema.PARTITIONS
        WHERE TABLE_SCHEMA = DATABASE()
        AND TABLE_NAME = p_table_name
        AND PARTITION_NAME IS NOT NULL
        AND PARTITION_NAME != 'p_future'
        ORDER BY PARTITION_ORDINAL_POSITION;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    SET v_current_date = CURDATE();
    SET v_archive_threshold = DATE_SUB(v_current_date, 
                                        INTERVAL p_retention_months MONTH);
    
    OPEN partition_cursor;
    
    read_loop: LOOP
        FETCH partition_cursor INTO v_partition_name;
        
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        -- Extraire année et mois du nom partition (format: p2023_01)
        SET v_year = CAST(SUBSTRING(v_partition_name, 2, 4) AS UNSIGNED);
        SET v_month = CAST(SUBSTRING(v_partition_name, 7, 2) AS UNSIGNED);
        
        -- Vérifier si partition doit être archivée
        IF MAKEDATE(v_year, 1) + INTERVAL v_month - 1 MONTH < v_archive_threshold THEN
            
            SET v_archive_table_name = CONCAT(p_table_name, '_archive_', 
                                              v_year, '_', 
                                              LPAD(v_month, 2, '0'));
            
            -- 1. Créer table archive
            SET @sql = CONCAT('CREATE TABLE IF NOT EXISTS ', v_archive_table_name,
                            ' LIKE ', p_table_name);
            PREPARE stmt FROM @sql;
            EXECUTE stmt;
            DEALLOCATE PREPARE stmt;
            
            -- 2. Supprimer partitionnement
            SET @sql = CONCAT('ALTER TABLE ', v_archive_table_name,
                            ' REMOVE PARTITIONING');
            PREPARE stmt FROM @sql;
            EXECUTE stmt;
            DEALLOCATE PREPARE stmt;
            
            -- 3. EXCHANGE partition
            SET @sql = CONCAT('ALTER TABLE ', p_table_name,
                            ' EXCHANGE PARTITION ', v_partition_name,
                            ' WITH TABLE ', v_archive_table_name);
            PREPARE stmt FROM @sql;
            EXECUTE stmt;
            DEALLOCATE PREPARE stmt;
            
            -- 4. Compresser table archive
            SET @sql = CONCAT('ALTER TABLE ', v_archive_table_name,
                            ' ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8');
            PREPARE stmt FROM @sql;
            EXECUTE stmt;
            DEALLOCATE PREPARE stmt;
            
            -- Log
            INSERT INTO partition_archive_log 
            VALUES (p_table_name, v_partition_name, v_archive_table_name,
                    NOW(), 'SUCCESS');
            
        END IF;
        
    END LOOP;
    
    CLOSE partition_cursor;
    
END //
DELIMITER ;

-- Table de log
CREATE TABLE IF NOT EXISTS partition_archive_log (
    table_name VARCHAR(64),
    partition_name VARCHAR(64),
    archive_table_name VARCHAR(64),
    archived_at TIMESTAMP,
    status VARCHAR(20),
    INDEX idx_archived_at (archived_at)
);

-- Utilisation
CALL archive_old_partitions('orders', 36);  -- Archiver > 3 ans
```

### Pattern 2 : Archivage avec export vers stockage externe

```sql
-- Archiver et exporter vers fichiers pour stockage froid

DELIMITER //
CREATE OR REPLACE PROCEDURE archive_and_export_partition(
    IN p_table_name VARCHAR(64),
    IN p_partition_name VARCHAR(64),
    IN p_export_path VARCHAR(255)
)
BEGIN
    DECLARE v_archive_table VARCHAR(64);
    DECLARE v_export_file VARCHAR(512);
    DECLARE v_row_count BIGINT;
    
    SET v_archive_table = CONCAT(p_table_name, '_archive_', p_partition_name);
    SET v_export_file = CONCAT(p_export_path, '/', v_archive_table, '.csv');
    
    -- 1. Créer table archive
    SET @sql = CONCAT('CREATE TABLE ', v_archive_table, ' LIKE ', p_table_name);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SET @sql = CONCAT('ALTER TABLE ', v_archive_table, ' REMOVE PARTITIONING');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 2. EXCHANGE partition
    SET @sql = CONCAT('ALTER TABLE ', p_table_name,
                      ' EXCHANGE PARTITION ', p_partition_name,
                      ' WITH TABLE ', v_archive_table);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 3. Compter lignes
    SET @sql = CONCAT('SELECT COUNT(*) INTO @row_count FROM ', v_archive_table);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    SET v_row_count = @row_count;
    
    -- 4. Exporter vers CSV
    SET @sql = CONCAT('SELECT * FROM ', v_archive_table,
                      ' INTO OUTFILE ''', v_export_file, '''',
                      ' FIELDS TERMINATED BY '','' ',
                      ' ENCLOSED BY ''"'' ',
                      ' LINES TERMINATED BY ''\\n''');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 5. Compresser le fichier (via système)
    SET @cmd = CONCAT('gzip ', v_export_file);
    -- Exécuter via UDF ou external script
    
    -- 6. Log
    INSERT INTO partition_export_log 
    VALUES (p_table_name, p_partition_name, v_export_file,
            v_row_count, NOW(), 'SUCCESS');
    
    -- 7. Optionnel : Supprimer table archive si export OK
    -- SET @sql = CONCAT('DROP TABLE ', v_archive_table);
    -- PREPARE stmt FROM @sql;
    -- EXECUTE stmt;
    -- DEALLOCATE PREPARE stmt;
    
END //
DELIMITER ;

-- Table de log
CREATE TABLE IF NOT EXISTS partition_export_log (
    table_name VARCHAR(64),
    partition_name VARCHAR(64),
    export_file VARCHAR(512),
    row_count BIGINT,
    exported_at TIMESTAMP,
    status VARCHAR(20)
);
```

### Pattern 3 : Archivage avec vérification d'intégrité

```sql
-- Archivage sécurisé avec checksums

DELIMITER //
CREATE OR REPLACE PROCEDURE safe_archive_partition(
    IN p_table_name VARCHAR(64),
    IN p_partition_name VARCHAR(64)
)
BEGIN
    DECLARE v_archive_table VARCHAR(64);
    DECLARE v_partition_count BIGINT;
    DECLARE v_archive_count BIGINT;
    DECLARE v_partition_checksum VARCHAR(64);
    DECLARE v_archive_checksum VARCHAR(64);
    DECLARE v_success BOOLEAN DEFAULT FALSE;
    
    SET v_archive_table = CONCAT(p_table_name, '_archive_', p_partition_name);
    
    -- Transaction pour atomicité
    START TRANSACTION;
    
    -- 1. Compter lignes partition AVANT
    SET @sql = CONCAT('SELECT COUNT(*) INTO @partition_count ',
                      'FROM ', p_table_name, ' PARTITION (', p_partition_name, ')');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    SET v_partition_count = @partition_count;
    
    -- 2. Calculer checksum partition AVANT
    SET @sql = CONCAT('SELECT MD5(GROUP_CONCAT(id ORDER BY id)) INTO @partition_checksum ',
                      'FROM ', p_table_name, ' PARTITION (', p_partition_name, ')');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    SET v_partition_checksum = @partition_checksum;
    
    -- 3. Créer table archive
    SET @sql = CONCAT('CREATE TABLE ', v_archive_table, ' LIKE ', p_table_name);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SET @sql = CONCAT('ALTER TABLE ', v_archive_table, ' REMOVE PARTITIONING');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 4. EXCHANGE partition
    SET @sql = CONCAT('ALTER TABLE ', p_table_name,
                      ' EXCHANGE PARTITION ', p_partition_name,
                      ' WITH TABLE ', v_archive_table);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 5. Vérifier après EXCHANGE
    -- Compter lignes archive
    SET @sql = CONCAT('SELECT COUNT(*) INTO @archive_count FROM ', v_archive_table);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    SET v_archive_count = @archive_count;
    
    -- Calculer checksum archive
    SET @sql = CONCAT('SELECT MD5(GROUP_CONCAT(id ORDER BY id)) INTO @archive_checksum ',
                      'FROM ', v_archive_table);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    SET v_archive_checksum = @archive_checksum;
    
    -- 6. Vérifier intégrité
    IF v_partition_count = v_archive_count 
       AND v_partition_checksum = v_archive_checksum THEN
        
        -- Vérifier que partition est maintenant vide
        SET @sql = CONCAT('SELECT COUNT(*) INTO @new_count ',
                          'FROM ', p_table_name, ' PARTITION (', p_partition_name, ')');
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
        
        IF @new_count = 0 THEN
            SET v_success = TRUE;
            COMMIT;
            
            -- Log succès
            INSERT INTO partition_archive_log 
            VALUES (p_table_name, p_partition_name, v_archive_table,
                    NOW(), 'SUCCESS', v_archive_count);
        ELSE
            ROLLBACK;
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Partition not empty after exchange';
        END IF;
        
    ELSE
        -- Intégrité échouée : ROLLBACK
        ROLLBACK;
        
        -- Log échec
        INSERT INTO partition_archive_log 
        VALUES (p_table_name, p_partition_name, v_archive_table,
                NOW(), 'FAILED', 0);
        
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Integrity check failed';
    END IF;
    
END //
DELIMITER ;
```

---

## Migration de tables non partitionnées

### Stratégie 1 : Migration offline (simple mais avec downtime)

```sql
-- Contexte : Convertir table orders (500M lignes) en table partitionnée

-- Étape 1 : Modifier clé primaire pour inclure clé partition
-- ⚠️ Opération longue et bloquante

ALTER TABLE orders 
DROP PRIMARY KEY,
ADD PRIMARY KEY (id, order_date);
-- Peut prendre plusieurs heures sur grande table

-- Étape 2 : Ajouter partitionnement
ALTER TABLE orders
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
-- Rebuild complet de la table
-- Durée : Plusieurs heures pour 500M lignes
```

### Stratégie 2 : Migration online (zero downtime)

```sql
-- Migration progressive avec table shadow

-- Étape 1 : Créer table partitionnée shadow
CREATE TABLE orders_partitioned (
    id BIGINT AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id, order_date),
    INDEX idx_customer (customer_id, order_date),
    INDEX idx_status (status, order_date)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Étape 2 : Copier données par batch (permet application running)
-- Script externe ou procédure stockée

DELIMITER //
CREATE OR REPLACE PROCEDURE copy_to_partitioned(
    IN p_batch_size INT,
    IN p_sleep_seconds INT
)
BEGIN
    DECLARE v_min_id BIGINT DEFAULT 0;
    DECLARE v_max_id BIGINT;
    DECLARE v_batch_count INT DEFAULT 0;
    
    -- Trouver max ID
    SELECT MAX(id) INTO v_max_id FROM orders;
    
    WHILE v_min_id < v_max_id DO
        
        -- Copier batch
        INSERT INTO orders_partitioned
        SELECT * FROM orders
        WHERE id > v_min_id AND id <= v_min_id + p_batch_size;
        
        SET v_min_id = v_min_id + p_batch_size;
        SET v_batch_count = v_batch_count + 1;
        
        -- Commit implicite après INSERT
        
        -- Log progression
        IF v_batch_count % 100 = 0 THEN
            INSERT INTO migration_progress 
            VALUES ('orders', v_min_id, v_max_id, NOW());
        END IF;
        
        -- Pause pour ne pas saturer
        DO SLEEP(p_sleep_seconds);
        
    END WHILE;
    
END //
DELIMITER ;

-- Exécuter copie
CALL copy_to_partitioned(10000, 1);  -- 10k lignes par batch, 1s pause

-- Étape 3 : Configurer réplication pour nouvelles insertions
-- Via triggers ou application dual-write

CREATE TRIGGER orders_after_insert
AFTER INSERT ON orders
FOR EACH ROW
INSERT INTO orders_partitioned VALUES (NEW.*);

CREATE TRIGGER orders_after_update
AFTER UPDATE ON orders
FOR EACH ROW
REPLACE INTO orders_partitioned VALUES (NEW.*);

CREATE TRIGGER orders_after_delete
AFTER DELETE ON orders
FOR EACH ROW
DELETE FROM orders_partitioned WHERE id = OLD.id;

-- Étape 4 : Vérifier synchronisation
SELECT 
    (SELECT COUNT(*) FROM orders) as original_count,
    (SELECT COUNT(*) FROM orders_partitioned) as partitioned_count;

-- Étape 5 : Bascule (fenêtre de maintenance courte)
START TRANSACTION;

-- Désactiver triggers
DROP TRIGGER orders_after_insert;
DROP TRIGGER orders_after_update;
DROP TRIGGER orders_after_delete;

-- Copier delta final
INSERT INTO orders_partitioned
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM orders_partitioned p WHERE p.id = o.id
);

-- Renommer tables
RENAME TABLE 
    orders TO orders_old,
    orders_partitioned TO orders;

COMMIT;

-- Étape 6 : Vérifier et cleanup
-- Attendre 24-48h pour s'assurer que tout fonctionne
-- DROP TABLE orders_old;
```

### Stratégie 3 : Utiliser pt-online-schema-change

```bash
# Percona Toolkit pour migration online automatique

pt-online-schema-change \
  --alter "PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
  )" \
  --execute \
  --chunk-size=5000 \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50" \
  --progress=time,30 \
  D=mydb,t=orders

# Avantages :
# - Zero downtime
# - Throttling automatique
# - Gestion erreurs
# - Rollback possible

# Inconvénient :
# - Temps de migration long (peut prendre jours pour très grande table)
```

---

## Réorganisation de partitions

### Fusionner partitions

```sql
-- Scénario : Fusionner partitions mensuelles en partitions annuelles

-- Avant : Partitions mensuelles
SHOW CREATE TABLE orders\G
/*
PARTITION BY RANGE (YEAR(order_date)*100 + MONTH(order_date)) (
    PARTITION p2023_01 VALUES LESS THAN (202302),
    PARTITION p2023_02 VALUES LESS THAN (202303),
    ...
    PARTITION p2023_12 VALUES LESS THAN (202401)
)
*/

-- Fusionner 12 partitions mensuelles 2023 en 1 partition annuelle
ALTER TABLE orders
REORGANIZE PARTITION 
    p2023_01, p2023_02, p2023_03, p2023_04,
    p2023_05, p2023_06, p2023_07, p2023_08,
    p2023_09, p2023_10, p2023_11, p2023_12
INTO (
    PARTITION p2023 VALUES LESS THAN (2024)
);

-- ⚠️ Opération bloquante
-- ⚠️ Durée proportionnelle au volume de données
-- Conseil : Exécuter pendant fenêtre de maintenance
```

### Diviser partitions

```sql
-- Scénario : Diviser partition annuelle en partitions trimestrielles

ALTER TABLE orders
REORGANIZE PARTITION p2024 INTO (
    PARTITION p2024_Q1 VALUES LESS THAN (202404),
    PARTITION p2024_Q2 VALUES LESS THAN (202407),
    PARTITION p2024_Q3 VALUES LESS THAN (202410),
    PARTITION p2024_Q4 VALUES LESS THAN (202501)
);

-- Use case : Granularité plus fine pour maintenance ciblée
```

### Réorganisation sans downtime

```sql
-- Pattern : Utiliser EXCHANGE pour réorganisation sans lock global

-- Étape 1 : Créer table temporaire partitionnée avec nouveau schéma
CREATE TABLE orders_reorg LIKE orders;

ALTER TABLE orders_reorg
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022_2023 VALUES LESS THAN (2024),  -- Fusion 2022+2023
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Étape 2 : Pour chaque partition, EXCHANGE vers temp
-- Partition p2022
CREATE TABLE temp_p2022 LIKE orders;
ALTER TABLE temp_p2022 REMOVE PARTITIONING;

ALTER TABLE orders 
EXCHANGE PARTITION p2022 WITH TABLE temp_p2022;

INSERT INTO orders_reorg PARTITION (p2022_2023)
SELECT * FROM temp_p2022;

DROP TABLE temp_p2022;

-- Répéter pour p2023...

-- Étape 3 : Bascule finale
RENAME TABLE orders TO orders_old, orders_reorg TO orders;

-- Étape 4 : Cleanup
DROP TABLE orders_old;
```

---

## Gestion d'erreurs et rollback

### Pattern de rollback sécurisé

```sql
-- Toujours avoir un plan de rollback

DELIMITER //
CREATE OR REPLACE PROCEDURE safe_partition_operation(
    IN p_operation VARCHAR(10),  -- 'ARCHIVE', 'REORGANIZE', etc.
    IN p_table_name VARCHAR(64),
    IN p_partition_name VARCHAR(64)
)
BEGIN
    DECLARE v_backup_table VARCHAR(64);
    DECLARE v_error_occurred BOOLEAN DEFAULT FALSE;
    
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error_occurred = TRUE;
    END;
    
    SET v_backup_table = CONCAT(p_table_name, '_backup_', 
                                DATE_FORMAT(NOW(), '%Y%m%d_%H%i%s'));
    
    -- 1. Créer backup de partition avant opération
    SET @sql = CONCAT('CREATE TABLE ', v_backup_table, ' LIKE ', p_table_name);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SET @sql = CONCAT('ALTER TABLE ', v_backup_table, ' REMOVE PARTITIONING');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- Copier données partition
    SET @sql = CONCAT('INSERT INTO ', v_backup_table,
                      ' SELECT * FROM ', p_table_name,
                      ' PARTITION (', p_partition_name, ')');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    IF v_error_occurred THEN
        -- Log erreur
        INSERT INTO operation_log VALUES 
            (p_table_name, p_operation, 'BACKUP_FAILED', NOW());
        
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Backup failed, operation aborted';
    END IF;
    
    -- 2. Exécuter opération réelle
    -- (code spécifique selon p_operation)
    
    -- 3. Vérifier résultat
    IF v_error_occurred THEN
        -- Rollback : Restaurer depuis backup
        SET @sql = CONCAT('ALTER TABLE ', p_table_name,
                          ' EXCHANGE PARTITION ', p_partition_name,
                          ' WITH TABLE ', v_backup_table);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
        
        INSERT INTO operation_log VALUES 
            (p_table_name, p_operation, 'ROLLBACK_SUCCESS', NOW());
    ELSE
        -- Succès : Garder backup 24h puis cleanup
        INSERT INTO operation_log VALUES 
            (p_table_name, p_operation, 'SUCCESS', NOW());
        
        -- Planifier suppression backup
        SET @sql = CONCAT('DROP TABLE ', v_backup_table);
        -- Exécuter après 24h via EVENT
    END IF;
    
END //
DELIMITER ;
```

### Vérification d'intégrité post-opération

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE verify_partition_integrity(
    IN p_table_name VARCHAR(64)
)
BEGIN
    DECLARE v_total_rows_partitions BIGINT;
    DECLARE v_total_rows_count BIGINT;
    DECLARE v_integrity_ok BOOLEAN;
    
    -- Compter via partitions
    SELECT SUM(TABLE_ROWS) INTO v_total_rows_partitions
    FROM information_schema.PARTITIONS
    WHERE TABLE_SCHEMA = DATABASE()
    AND TABLE_NAME = p_table_name
    AND PARTITION_NAME IS NOT NULL;
    
    -- Compter direct
    SET @sql = CONCAT('SELECT COUNT(*) INTO @total_rows FROM ', p_table_name);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    SET v_total_rows_count = @total_rows;
    
    -- Vérifier correspondance (avec marge pour approximation TABLE_ROWS)
    SET v_integrity_ok = (ABS(v_total_rows_partitions - v_total_rows_count) 
                          < v_total_rows_count * 0.05);  -- Marge 5%
    
    IF NOT v_integrity_ok THEN
        INSERT INTO integrity_check_log 
        VALUES (p_table_name, v_total_rows_partitions, 
                v_total_rows_count, 'MISMATCH', NOW());
        
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Partition integrity check failed';
    ELSE
        INSERT INTO integrity_check_log 
        VALUES (p_table_name, v_total_rows_partitions, 
                v_total_rows_count, 'OK', NOW());
    END IF;
    
END //
DELIMITER ;

CREATE TABLE IF NOT EXISTS integrity_check_log (
    table_name VARCHAR(64),
    rows_from_partitions BIGINT,
    rows_from_count BIGINT,
    status VARCHAR(20),
    checked_at TIMESTAMP
);
```

---

## Automatisation complète

### Système de gestion automatisé des partitions

```sql
-- Configuration de la gestion automatisée

CREATE TABLE IF NOT EXISTS partition_config (
    table_name VARCHAR(64) PRIMARY KEY,
    partition_type ENUM('RANGE_YEAR', 'RANGE_MONTH', 'RANGE_QUARTER'),
    partition_column VARCHAR(64),
    retention_periods INT COMMENT 'Number of periods to keep online',
    archive_enabled BOOLEAN DEFAULT TRUE,
    auto_add_future BOOLEAN DEFAULT TRUE,
    auto_optimize BOOLEAN DEFAULT TRUE,
    last_maintenance TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Exemple de configuration
INSERT INTO partition_config VALUES
('orders', 'RANGE_YEAR', 'order_date', 3, TRUE, TRUE, TRUE, NULL, NOW()),
('logs', 'RANGE_MONTH', 'log_date', 6, TRUE, TRUE, FALSE, NULL, NOW());

-- Procédure maître de maintenance
DELIMITER //
CREATE OR REPLACE PROCEDURE auto_maintain_all_partitions()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_table_name VARCHAR(64);
    DECLARE v_partition_type VARCHAR(20);
    DECLARE v_retention INT;
    DECLARE v_archive_enabled BOOLEAN;
    DECLARE v_auto_add BOOLEAN;
    DECLARE v_auto_optimize BOOLEAN;
    
    DECLARE config_cursor CURSOR FOR
        SELECT table_name, partition_type, retention_periods,
               archive_enabled, auto_add_future, auto_optimize
        FROM partition_config;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN config_cursor;
    
    read_loop: LOOP
        FETCH config_cursor INTO v_table_name, v_partition_type,
                                 v_retention, v_archive_enabled,
                                 v_auto_add, v_auto_optimize;
        
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        -- 1. Ajouter partitions futures si nécessaire
        IF v_auto_add THEN
            CALL add_future_partitions(v_table_name, v_partition_type);
        END IF;
        
        -- 2. Archiver partitions anciennes
        IF v_archive_enabled THEN
            CALL archive_old_partitions_auto(v_table_name, v_retention);
        END IF;
        
        -- 3. Optimiser partitions actives
        IF v_auto_optimize THEN
            CALL optimize_active_partitions(v_table_name);
        END IF;
        
        -- 4. Mettre à jour last_maintenance
        UPDATE partition_config 
        SET last_maintenance = NOW()
        WHERE table_name = v_table_name;
        
    END LOOP;
    
    CLOSE config_cursor;
    
END //
DELIMITER ;

-- Planifier exécution quotidienne
CREATE EVENT IF NOT EXISTS daily_partition_maintenance
ON SCHEDULE EVERY 1 DAY
STARTS TIMESTAMP(CURRENT_DATE) + INTERVAL 2 HOUR  -- 2h du matin
DO CALL auto_maintain_all_partitions();
```

### Dashboard de monitoring

```sql
-- Vue : État des partitions
CREATE OR REPLACE VIEW v_partition_status AS
SELECT 
    p.TABLE_NAME,
    p.PARTITION_NAME,
    p.PARTITION_METHOD,
    p.PARTITION_EXPRESSION,
    p.TABLE_ROWS,
    ROUND(p.DATA_LENGTH / 1024 / 1024, 2) as data_mb,
    ROUND(p.INDEX_LENGTH / 1024 / 1024, 2) as index_mb,
    ROUND((p.DATA_LENGTH + p.INDEX_LENGTH) / 1024 / 1024, 2) as total_mb,
    c.retention_periods,
    c.last_maintenance,
    CASE 
        WHEN c.last_maintenance < DATE_SUB(NOW(), INTERVAL 7 DAY)
        THEN 'MAINTENANCE_OVERDUE'
        WHEN p.TABLE_ROWS > 200000000  -- 200M lignes
        THEN 'LARGE_PARTITION'
        ELSE 'OK'
    END as status
FROM information_schema.PARTITIONS p
LEFT JOIN partition_config c ON c.table_name = p.TABLE_NAME
WHERE p.TABLE_SCHEMA = DATABASE()
AND p.PARTITION_NAME IS NOT NULL
ORDER BY p.TABLE_NAME, p.PARTITION_ORDINAL_POSITION;

-- Utiliser
SELECT * FROM v_partition_status WHERE status != 'OK';
```

---

## Troubleshooting

### Problème 1 : EXCHANGE échoue avec erreur structure

```sql
-- Erreur : "Tables have different definitions"

-- Diagnostic
SHOW CREATE TABLE orders\G
SHOW CREATE TABLE orders_archive_2020\G

-- Comparer champ par champ
SELECT 
    COLUMN_NAME,
    COLUMN_TYPE,
    IS_NULLABLE,
    COLUMN_DEFAULT,
    EXTRA
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;

SELECT 
    COLUMN_NAME,
    COLUMN_TYPE,
    IS_NULLABLE,
    COLUMN_DEFAULT,
    EXTRA
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
AND TABLE_NAME = 'orders_archive_2020'
ORDER BY ORDINAL_POSITION;

-- Solution : Recréer table archive exactement identique
DROP TABLE orders_archive_2020;
CREATE TABLE orders_archive_2020 LIKE orders;
ALTER TABLE orders_archive_2020 REMOVE PARTITIONING;
```

### Problème 2 : Partition trop grande pour opération

```sql
-- Erreur : "Lock wait timeout" ou opération trop longue

-- Solution : Diviser partition d'abord

-- Au lieu de :
ALTER TABLE orders DROP PARTITION p2020;  -- Bloque longtemps

-- Faire :
-- 1. Vider partition progressivement
DELETE FROM orders PARTITION (p2020)
WHERE id < 1000000
LIMIT 100000;
-- Répéter jusqu'à vide

-- 2. Puis drop partition vide
ALTER TABLE orders DROP PARTITION p2020;  -- Instantané
```

### Problème 3 : Données dans mauvaise partition

```sql
-- Diagnostic : Vérifier distribution
SELECT 
    PARTITION_NAME,
    COUNT(*) as row_count,
    MIN(YEAR(order_date)) as min_year,
    MAX(YEAR(order_date)) as max_year
FROM orders PARTITION (p2023)
GROUP BY PARTITION_NAME;

-- Si données 2024 dans partition p2023 :
-- Solution : REORGANIZE ou DELETE + REINSERT

-- Option 1 : REORGANIZE (automatique mais lent)
ALTER TABLE orders REORGANIZE PARTITION p2023, p2024 INTO (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Option 2 : Correction manuelle
-- Créer table temporaire
CREATE TABLE temp_misplaced AS
SELECT * FROM orders PARTITION (p2023)
WHERE YEAR(order_date) != 2023;

DELETE FROM orders PARTITION (p2023)
WHERE YEAR(order_date) != 2023;

INSERT INTO orders SELECT * FROM temp_misplaced;

DROP TABLE temp_misplaced;
```

---

## ✅ Points clés à retenir

- 🔄 **EXCHANGE PARTITION = outil clé** : Swap instantané partition ↔ table (1s vs heures)
- 📦 **Structure identique requise** : Table archive DOIT être exacte copie sans partitionnement
- 🗄️ **Archivage efficient** : EXCHANGE + compression = optimal pour stockage long terme
- 🔒 **Vérification intégrité** : Toujours vérifier count + checksums avant/après
- 🚀 **Migration online** : Table shadow + triggers pour zero downtime
- 🔧 **Automatisation critique** : Events + procédures pour maintenance proactive
- 📊 **Monitoring continu** : Dashboard pour détecter partitions problématiques
- ⚠️ **Plan rollback** : Toujours backup avant opération critique
- 🎯 **REORGANIZE = coûteux** : Éviter si possible, préférer EXCHANGE
- 📝 **Documentation** : Logger toutes opérations pour audit et troubleshooting

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 ALTER TABLE ... EXCHANGE PARTITION](https://mariadb.com/kb/en/alter-table/#exchange-partition)
- [📖 Partition Management](https://mariadb.com/kb/en/partition-management/)
- [📖 Partitioning Limitations](https://mariadb.com/kb/en/partitioning-limitations/)

### Outils complémentaires

- [Percona Toolkit - pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [Percona Toolkit - pt-archiver](https://docs.percona.com/percona-toolkit/pt-archiver.html)

### Lectures avancées

- [MySQL Partition Management Best Practices](https://www.percona.com/blog/partition-management-best-practices/)

---

## ➡️ Prochaines étapes

Avec la maîtrise de la gestion avancée des partitions, vous pouvez maintenant :

1. **Gérer des tables de plusieurs milliards de lignes** avec archivage automatisé
2. **Migrer vers partitionnement** sans downtime applicatif
3. **Optimiser les coûts** de stockage (compression des archives)
4. **Automatiser la maintenance** complète du cycle de vie des données
5. **Troubleshooter** efficacement les problèmes de partitionnement

La gestion avancée des partitions transforme le partitionnement d'une simple optimisation en un **système complet de lifecycle management** des données.

---

*EXCHANGE PARTITION est l'opération la plus puissante du partitionnement MariaDB. Sa maîtrise est essentielle pour gérer efficacement de très grandes tables en production.*

⏭️ [Sharding et distribution horizontale](/15-performance-tuning/11-sharding-distribution.md)

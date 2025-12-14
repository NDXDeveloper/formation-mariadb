🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.1 Sequences (CREATE SEQUENCE)

> **Niveau** : Avancé  
> **Durée estimée** : 1-1.5 heures  
> **Prérequis** : Chapitre 2 (Bases SQL), compréhension AUTO_INCREMENT

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Créer et configurer des **sequences** avec tous leurs paramètres
- Comprendre les **différences fondamentales** entre SEQUENCE et AUTO_INCREMENT
- Utiliser les sequences pour générer des **identifiants multi-tables**
- Implémenter des **schémas de numérotation métier** (factures, commandes, tickets)
- Maîtriser les fonctions **NEXT VALUE, LASTVAL, SETVAL**
- Optimiser les performances avec le paramètre **CACHE**
- Gérer les sequences dans un contexte de **haute concurrence**

---

## Introduction

Les **sequences** sont des objets de base de données qui génèrent des **nombres séquentiels uniques** de manière indépendante des tables. Contrairement à AUTO_INCREMENT qui est lié à une colonne de table spécifique, une sequence est un objet autonome réutilisable.

### Pourquoi utiliser des Sequences ?

**Problématiques résolues** :
1. **Numérotation multi-tables** : Partager un compteur entre plusieurs tables
2. **Contrôle granulaire** : Min/Max, Step, Cycle, Cache
3. **Schémas de numérotation métier** : Format année-numéro, préfixes, etc.
4. **Performance** : Allocation par batch (CACHE)
5. **Indépendance** : Pas de lock de table lors de génération

**Cas d'usage typiques** :
- 📄 Numérotation de documents (factures, commandes, tickets)
- 🔢 Identifiants partagés entre tables d'un même domaine
- 📊 Génération de clés techniques dans architectures distribuées
- 🔄 Systèmes avec rotations annuelles/mensuelles
- 🎫 Systèmes de ticketing avec garantie de séquence

💡 **Philosophie** : Une sequence est à la génération de nombres ce qu'une vue est à une requête - un objet réutilisable et configurable.

---

## Syntaxe CREATE SEQUENCE

### Syntaxe Complète

```sql
CREATE [OR REPLACE] [TEMPORARY] SEQUENCE [IF NOT EXISTS] sequence_name
    [START WITH start_value]
    [INCREMENT BY increment_value]
    [MINVALUE min_value | NO MINVALUE]
    [MAXVALUE max_value | NO MAXVALUE]
    [CACHE cache_size | NOCACHE]
    [CYCLE | NOCYCLE];
```

### Paramètres Détaillés

| Paramètre | Description | Valeur par défaut | Contraintes |
|-----------|-------------|-------------------|-------------|
| **START WITH** | Valeur de départ | 1 (ou MINVALUE si INCREMENT < 0) | Entre MINVALUE et MAXVALUE |
| **INCREMENT BY** | Pas d'incrémentation | 1 | Positif ou négatif, ≠ 0 |
| **MINVALUE** | Valeur minimale | 1 (ou -9223372036854775807 si négatif) | < MAXVALUE |
| **MAXVALUE** | Valeur maximale | 9223372036854775807 | > MINVALUE |
| **CACHE** | Taille du cache | 1000 | 0 (NOCACHE) ou entier positif |
| **CYCLE** | Retour au début après MAX | NOCYCLE | CYCLE ou NOCYCLE |

### Exemples de Création

```sql
-- 1. Sequence simple (comportement par défaut)
CREATE SEQUENCE simple_seq;
-- Démarre à 1, incrémente de 1, pas de limite supérieure

-- 2. Sequence pour numérotation de factures (commence à 1000)
CREATE SEQUENCE invoice_seq
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1000
    MAXVALUE 999999
    CACHE 50;

-- 3. Sequence décrémentale
CREATE SEQUENCE countdown_seq
    START WITH 100
    INCREMENT BY -1
    MINVALUE 1
    MAXVALUE 100
    NOCACHE;

-- 4. Sequence cyclique (rotative)
CREATE SEQUENCE rotation_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 100
    CYCLE
    CACHE 20;

-- 5. Sequence par année (réinitialisée manuellement chaque année)
CREATE SEQUENCE order_2025_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9999999;

-- 6. Sequence temporaire (session uniquement)
CREATE TEMPORARY SEQUENCE temp_batch_seq
    START WITH 1
    INCREMENT BY 1;
```

⚠️ **Attention** : 
- `MAXVALUE` n'est PAS une limite du nombre de valeurs, mais la valeur maximale autorisée
- Une fois MAXVALUE atteinte sans CYCLE, la sequence lève une erreur
- `CACHE` améliore les performances mais peut créer des gaps en cas de crash

---

## Utilisation des Sequences

### Fonctions Principales

#### NEXT VALUE FOR - Obtenir la Prochaine Valeur

```sql
-- Syntaxe
SELECT NEXT VALUE FOR sequence_name;

-- Utilisation dans INSERT
INSERT INTO invoices (invoice_number, customer_id, amount)
VALUES (NEXT VALUE FOR invoice_seq, 123, 1500.00);

-- Utilisation dans UPDATE
UPDATE orders 
SET tracking_number = NEXT VALUE FOR tracking_seq
WHERE order_id = 456;

-- Assigner à une variable
SET @next_id = NEXT VALUE FOR simple_seq;

-- Dans un SELECT (génère une valeur pour chaque ligne)
SELECT 
    NEXT VALUE FOR batch_seq AS batch_id,
    product_name,
    quantity
FROM staging_products;
```

**Comportement** :
- ✅ Thread-safe (pas de risque de doublon)
- ✅ Incrémente la sequence à chaque appel
- ✅ Utilise le cache si configuré
- ⚠️ Non transactionnel (pas de ROLLBACK possible)

#### LASTVAL - Dernière Valeur Générée

```sql
-- Obtenir la dernière valeur générée (dans la session courante)
SELECT LASTVAL(invoice_seq);

-- Exemple d'utilisation après INSERT
INSERT INTO invoices (invoice_number, customer_id)
VALUES (NEXT VALUE FOR invoice_seq, 123);

-- Récupérer le numéro généré
SET @invoice_num = LASTVAL(invoice_seq);

INSERT INTO invoice_items (invoice_number, product_id, quantity)
VALUES (@invoice_num, 789, 5);
```

**Important** :
- LASTVAL retourne la **dernière valeur générée dans la session**
- Si aucune valeur n'a été générée, retourne NULL
- Similaire à LAST_INSERT_ID() pour AUTO_INCREMENT

#### SETVAL - Modifier la Valeur Courante

```sql
-- Définir la prochaine valeur
SELECT SETVAL(sequence_name, new_value);

-- Exemple : Réinitialiser une sequence annuelle
SELECT SETVAL(order_2025_seq, 1);

-- Définir avec is_called (false = la valeur est incluse dans la prochaine génération)
SELECT SETVAL(invoice_seq, 5000, false);
-- Prochaine valeur sera 5000

SELECT SETVAL(invoice_seq, 5000, true);
-- Prochaine valeur sera 5001 (5000 a déjà été "consommée")

-- Exemple : Synchroniser après import de données
-- Si dernière facture importée = 4523
SELECT SETVAL(invoice_seq, 4523, true);
-- Prochaine facture sera 4524
```

⚠️ **Attention** : 
- SETVAL ne vérifie pas les conflits avec les données existantes
- Responsabilité de l'administrateur d'assurer la cohérence

---

## Gestion et Administration

### Inspection des Sequences

```sql
-- Lister toutes les sequences de la base
SELECT * FROM information_schema.SEQUENCES;

-- Détails d'une sequence spécifique
SHOW CREATE SEQUENCE invoice_seq;

-- Résultat typique :
-- CREATE SEQUENCE `invoice_seq` 
--   START WITH 1000 
--   INCREMENT BY 1 
--   MINVALUE 1000 
--   MAXVALUE 999999 
--   CACHE 50 
--   NOCYCLE;

-- Valeur courante (sans la consommer)
SELECT * FROM invoice_seq;
-- Retourne : next_not_cached_value, minimum_value, maximum_value, 
--            start_value, increment, cache_size, cycle_option
```

### Modification de Sequences

```sql
-- ALTER SEQUENCE pour modifier les paramètres
ALTER SEQUENCE invoice_seq
    MAXVALUE 9999999
    CACHE 100;

-- Redémarrer une sequence (nouveau cycle)
ALTER SEQUENCE order_2025_seq RESTART;

-- Redémarrer avec nouvelle valeur de départ
ALTER SEQUENCE order_2025_seq RESTART WITH 1;

-- Changer l'incrément
ALTER SEQUENCE rotation_seq INCREMENT BY 5;

-- Activer le cycle
ALTER SEQUENCE ticket_seq CYCLE;
```

### Suppression de Sequences

```sql
-- Supprimer une sequence
DROP SEQUENCE IF EXISTS old_seq;

-- Supprimer et recréer
CREATE OR REPLACE SEQUENCE invoice_seq
    START WITH 1000
    INCREMENT BY 1;
```

---

## Sequences vs AUTO_INCREMENT

### Comparaison Technique

| Aspect | SEQUENCE | AUTO_INCREMENT |
|--------|----------|----------------|
| **Scope** | Indépendant des tables | Lié à une colonne |
| **Partage** | ✅ Multi-tables | ❌ Une seule table |
| **Contrôle** | ✅ Total (min/max/cycle/cache) | ⚠️ Limité |
| **Performance** | ✅ CACHE optimisé | ⚠️ Lock de table possible |
| **Gaps** | ✅ Acceptables | ⚠️ Créés par ROLLBACK |
| **Réinitialisation** | ✅ SETVAL, ALTER RESTART | ⚠️ ALTER TABLE AUTO_INCREMENT |
| **Standards SQL** | ✅ SQL:2003 | ❌ MySQL-specific |
| **Complexité** | ⚠️ Plus complexe | ✅ Simple |

### Quand Utiliser Quoi ?

**Utilisez AUTO_INCREMENT si** :
- ✅ Clé primaire simple d'une seule table
- ✅ Pas de besoin de numérotation spécifique
- ✅ Simplicité prioritaire
- ✅ Équipe peu familière avec les sequences

**Utilisez SEQUENCE si** :
- ✅ Partage d'identifiants entre tables
- ✅ Schéma de numérotation métier complexe
- ✅ Besoin de contrôle (min/max/cycle)
- ✅ Performance critique (CACHE élevé)
- ✅ Architectures distribuées
- ✅ Compatibilité PostgreSQL/Oracle

### Migration AUTO_INCREMENT → SEQUENCE

```sql
-- Situation initiale : Table avec AUTO_INCREMENT
CREATE TABLE orders_old (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    order_date DATE
);

-- Migration vers SEQUENCE
-- Étape 1 : Créer la sequence
CREATE SEQUENCE order_seq
    START WITH 1
    INCREMENT BY 1
    CACHE 100;

-- Étape 2 : Synchroniser avec les données existantes
SELECT SETVAL(order_seq, (SELECT MAX(order_id) FROM orders_old), true);

-- Étape 3 : Nouvelle table sans AUTO_INCREMENT
CREATE TABLE orders_new (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE
);

-- Étape 4 : Utilisation dans l'application
INSERT INTO orders_new (order_id, customer_id, order_date)
VALUES (NEXT VALUE FOR order_seq, 123, CURDATE());

-- Alternative : Trigger pour transparence
DELIMITER $$
CREATE TRIGGER orders_before_insert
BEFORE INSERT ON orders_new
FOR EACH ROW
BEGIN
    IF NEW.order_id IS NULL OR NEW.order_id = 0 THEN
        SET NEW.order_id = NEXT VALUE FOR order_seq;
    END IF;
END$$
DELIMITER ;

-- Désormais, INSERT sans order_id fonctionne
INSERT INTO orders_new (customer_id, order_date)
VALUES (123, CURDATE());
```

---

## Cas d'Usage Avancés

### 1. Numérotation de Factures avec Format Année

**Besoin** : Factures numérotées `2025-0001`, `2025-0002`, etc. avec reset annuel.

```sql
-- Sequence pour 2025
CREATE SEQUENCE invoice_2025_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9999
    CACHE 50;

-- Table des factures
CREATE TABLE invoices (
    invoice_id INT PRIMARY KEY AUTO_INCREMENT,
    invoice_number VARCHAR(20) UNIQUE NOT NULL,
    customer_id INT,
    amount DECIMAL(10,2),
    invoice_date DATE
);

-- Procédure pour générer le numéro de facture
DELIMITER $$
CREATE FUNCTION generate_invoice_number(year INT)
RETURNS VARCHAR(20)
DETERMINISTIC
BEGIN
    DECLARE seq_name VARCHAR(50);
    DECLARE seq_value INT;
    
    SET seq_name = CONCAT('invoice_', year, '_seq');
    
    -- Générer la prochaine valeur
    SET @sql = CONCAT('SELECT NEXT VALUE FOR ', seq_name, ' INTO @seq_value');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- Format : YYYY-NNNN
    RETURN CONCAT(year, '-', LPAD(@seq_value, 4, '0'));
END$$
DELIMITER ;

-- Utilisation
INSERT INTO invoices (invoice_number, customer_id, amount, invoice_date)
VALUES (
    generate_invoice_number(YEAR(CURDATE())),
    123,
    1500.00,
    CURDATE()
);

-- Résultat : invoice_number = '2025-0001'

-- En début d'année suivante, créer nouvelle sequence
CREATE SEQUENCE invoice_2026_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9999
    CACHE 50;
```

### 2. Identifiants Partagés Multi-Tables

**Besoin** : Tables `customers` et `suppliers` partagent le même pool d'identifiants pour garantir unicité globale.

```sql
-- Sequence partagée pour entités business
CREATE SEQUENCE business_entity_seq
    START WITH 1000
    INCREMENT BY 1
    CACHE 100;

-- Table customers
CREATE TABLE customers (
    entity_id INT PRIMARY KEY,
    entity_type VARCHAR(10) DEFAULT 'CUSTOMER',
    customer_name VARCHAR(100),
    -- autres champs
);

-- Table suppliers
CREATE TABLE suppliers (
    entity_id INT PRIMARY KEY,
    entity_type VARCHAR(10) DEFAULT 'SUPPLIER',
    supplier_name VARCHAR(100),
    -- autres champs
);

-- Insertion dans customers
INSERT INTO customers (entity_id, customer_name)
VALUES (NEXT VALUE FOR business_entity_seq, 'Acme Corp');
-- entity_id = 1000

-- Insertion dans suppliers
INSERT INTO suppliers (entity_id, supplier_name)
VALUES (NEXT VALUE FOR business_entity_seq, 'Global Supplies');
-- entity_id = 1001

-- Vue unifiée
CREATE VIEW business_entities AS
    SELECT entity_id, entity_type, customer_name AS name FROM customers
    UNION ALL
    SELECT entity_id, entity_type, supplier_name AS name FROM suppliers;

-- Recherche : entity_id unique dans tout le système
SELECT * FROM business_entities WHERE entity_id = 1000;
```

**Avantages** :
- ✅ Unicité garantie entre tables
- ✅ Simplification des jointures et références
- ✅ Facilite les migrations (customer → supplier)

### 3. Système de Ticketing avec Priorités

**Besoin** : Tickets avec numérotation par priorité (P1: 1xxx, P2: 2xxx, P3: 3xxx).

```sql
-- Sequences par priorité
CREATE SEQUENCE ticket_p1_seq START WITH 1000 MAXVALUE 1999 CACHE 10;
CREATE SEQUENCE ticket_p2_seq START WITH 2000 MAXVALUE 2999 CACHE 20;
CREATE SEQUENCE ticket_p3_seq START WITH 3000 MAXVALUE 3999 CACHE 50;

-- Table tickets
CREATE TABLE tickets (
    ticket_id INT PRIMARY KEY AUTO_INCREMENT,
    ticket_number INT UNIQUE NOT NULL,
    priority ENUM('P1','P2','P3'),
    title VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Procédure pour créer un ticket
DELIMITER $$
CREATE PROCEDURE create_ticket(
    IN p_priority ENUM('P1','P2','P3'),
    IN p_title VARCHAR(255),
    OUT p_ticket_number INT
)
BEGIN
    CASE p_priority
        WHEN 'P1' THEN
            SET p_ticket_number = NEXT VALUE FOR ticket_p1_seq;
        WHEN 'P2' THEN
            SET p_ticket_number = NEXT VALUE FOR ticket_p2_seq;
        WHEN 'P3' THEN
            SET p_ticket_number = NEXT VALUE FOR ticket_p3_seq;
    END CASE;
    
    INSERT INTO tickets (ticket_number, priority, title)
    VALUES (p_ticket_number, p_priority, p_title);
END$$
DELIMITER ;

-- Utilisation
CALL create_ticket('P1', 'Production database down', @ticket_num);
SELECT @ticket_num;  -- 1000

CALL create_ticket('P3', 'Update documentation', @ticket_num);
SELECT @ticket_num;  -- 3000
```

### 4. Batch Processing avec Sequences

**Besoin** : Assigner un batch_id unique à chaque lot de traitement.

```sql
-- Sequence pour batch IDs
CREATE SEQUENCE batch_seq
    START WITH 1
    INCREMENT BY 1
    CACHE 1;  -- NOCACHE pour garantie stricte

-- Table de logs de batch
CREATE TABLE batch_logs (
    batch_id INT PRIMARY KEY,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    records_processed INT,
    status ENUM('RUNNING','SUCCESS','FAILED')
);

-- Procédure de batch processing
DELIMITER $$
CREATE PROCEDURE process_daily_batch()
BEGIN
    DECLARE batch_id INT;
    DECLARE records_count INT;
    
    -- Générer batch_id unique
    SET batch_id = NEXT VALUE FOR batch_seq;
    
    -- Logger le début
    INSERT INTO batch_logs (batch_id, start_time, status)
    VALUES (batch_id, NOW(), 'RUNNING');
    
    -- Traitement (exemple)
    INSERT INTO processed_data (batch_id, data_value)
    SELECT batch_id, raw_value
    FROM staging_data
    WHERE processed = 0;
    
    SET records_count = ROW_COUNT();
    
    -- Marquer les données comme traitées
    UPDATE staging_data 
    SET processed = 1, batch_id = batch_id
    WHERE processed = 0;
    
    -- Logger la fin
    UPDATE batch_logs
    SET end_time = NOW(),
        records_processed = records_count,
        status = 'SUCCESS'
    WHERE batch_id = batch_id;
END$$
DELIMITER ;

-- Exécution
CALL process_daily_batch();

-- Traçabilité complète
SELECT batch_id, 
       start_time, 
       end_time,
       records_processed,
       TIMESTAMPDIFF(SECOND, start_time, end_time) AS duration_sec
FROM batch_logs
ORDER BY batch_id DESC
LIMIT 10;
```

### 5. Distributed ID Generation (Architectures Distribuées)

**Besoin** : Générer des IDs uniques dans un système multi-datacenter.

```sql
-- Chaque datacenter a sa propre sequence avec range réservé
-- DC1 : 1-1000000, DC2 : 1000001-2000000, DC3 : 2000001-3000000

-- Datacenter 1
CREATE SEQUENCE dc1_id_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 1000000
    CACHE 1000;

-- Datacenter 2
CREATE SEQUENCE dc2_id_seq
    START WITH 1000001
    INCREMENT BY 1
    MINVALUE 1000001
    MAXVALUE 2000000
    CACHE 1000;

-- Datacenter 3
CREATE SEQUENCE dc3_id_seq
    START WITH 2000001
    INCREMENT BY 1
    MINVALUE 2000001
    MAXVALUE 3000000
    CACHE 1000;

-- Fonction pour générer ID selon DC
DELIMITER $$
CREATE FUNCTION generate_distributed_id(datacenter_id INT)
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE id INT;
    
    CASE datacenter_id
        WHEN 1 THEN SET id = NEXT VALUE FOR dc1_id_seq;
        WHEN 2 THEN SET id = NEXT VALUE FOR dc2_id_seq;
        WHEN 3 THEN SET id = NEXT VALUE FOR dc3_id_seq;
        ELSE SIGNAL SQLSTATE '45000' 
             SET MESSAGE_TEXT = 'Invalid datacenter_id';
    END CASE;
    
    RETURN id;
END$$
DELIMITER ;

-- Utilisation dans application
SET @current_dc = 1;  -- Déterminé par configuration application

INSERT INTO global_orders (order_id, customer_id)
VALUES (generate_distributed_id(@current_dc), 123);
```

**Alternative moderne : Snowflake-like IDs**
```sql
-- Format : timestamp (41 bits) + datacenter (5 bits) + sequence (18 bits)
-- Nécessite fonction custom ou UDF
```

---

## Performance et Optimisation

### Impact du Paramètre CACHE

Le paramètre **CACHE** est crucial pour les performances en haute concurrence.

```sql
-- Sans cache (NOCACHE ou CACHE 1)
CREATE SEQUENCE slow_seq NOCACHE;
-- Chaque NEXT VALUE génère une écriture disque (lent)
-- Garantie : aucun gap en cas de crash

-- Avec cache modéré
CREATE SEQUENCE medium_seq CACHE 100;
-- 1 écriture disque toutes les 100 valeurs
-- Compromis performance/gaps

-- Avec cache élevé
CREATE SEQUENCE fast_seq CACHE 10000;
-- Optimal pour très haute fréquence
-- Peut créer des gaps de 10000 en cas de crash
```

**Benchmark typique (insertions/sec)** :

| Configuration | Insertions/sec | Gaps possibles |
|---------------|----------------|----------------|
| NOCACHE | ~5,000 | 0 |
| CACHE 10 | ~15,000 | 10 |
| CACHE 100 | ~50,000 | 100 |
| CACHE 1000 | ~80,000 | 1000 |
| CACHE 10000 | ~100,000 | 10000 |

💡 **Best practice** : 
- CACHE 1-10 : Logs d'audit critiques (gaps inacceptables)
- CACHE 100-500 : Usage général (bon compromis)
- CACHE 1000-10000 : Très haute fréquence (gaps acceptables)

### Contention et Concurrence

```sql
-- Test de concurrence : 10 threads insèrent simultanément
-- Thread 1
INSERT INTO orders (order_id, ...) VALUES (NEXT VALUE FOR order_seq, ...);

-- Thread 2
INSERT INTO orders (order_id, ...) VALUES (NEXT VALUE FOR order_seq, ...);

-- Thread 10
INSERT INTO orders (order_id, ...) VALUES (NEXT VALUE FOR order_seq, ...);

-- MariaDB garantit :
-- - Aucun doublon
-- - Thread-safe
-- - Performance linéaire avec CACHE approprié
```

**Monitoring de performance** :
```sql
-- Statistiques sequences (MariaDB 11.x+)
SELECT TABLE_NAME, 
       NEXT_NOT_CACHED_VALUE,
       CACHE_SIZE
FROM information_schema.SEQUENCES
WHERE TABLE_SCHEMA = DATABASE();

-- Identifier sequences à faible cache
SELECT TABLE_NAME
FROM information_schema.SEQUENCES
WHERE CACHE_SIZE < 100
  AND TABLE_SCHEMA = DATABASE();
```

---

## Best Practices et Recommandations

### ✅ Bonnes Pratiques

1. **Nommage explicite**
```sql
-- ✅ Bon : Nom descriptif avec suffixe _seq
CREATE SEQUENCE invoice_number_seq;
CREATE SEQUENCE customer_id_seq;

-- ❌ Mauvais : Nom ambigu
CREATE SEQUENCE s1;
CREATE SEQUENCE seq;
```

2. **CACHE adapté au volume**
```sql
-- Faible fréquence (< 100/sec)
CREATE SEQUENCE occasional_seq CACHE 10;

-- Moyenne fréquence (100-1000/sec)
CREATE SEQUENCE standard_seq CACHE 100;

-- Haute fréquence (> 1000/sec)
CREATE SEQUENCE high_freq_seq CACHE 1000;
```

3. **MINVALUE/MAXVALUE pour limites métier**
```sql
-- Limiter range pour éviter débordement
CREATE SEQUENCE yearly_invoice_seq
    START WITH 1
    MAXVALUE 999999  -- Max 999,999 factures/an
    NOCYCLE;         -- Erreur si dépassement (volontaire)
```

4. **Documentation dans commentaires**
```sql
-- MariaDB ne supporte pas COMMENT sur SEQUENCE
-- Documenter dans une table dédiée
CREATE TABLE sequence_documentation (
    sequence_name VARCHAR(64) PRIMARY KEY,
    description TEXT,
    owner VARCHAR(64),
    reset_policy VARCHAR(100)
);

INSERT INTO sequence_documentation VALUES
('invoice_2025_seq', 'Numérotation factures 2025', 'Finance', 'Annuel');
```

5. **Monitoring et alertes**
```sql
-- Créer une vue pour monitoring
CREATE VIEW sequence_health AS
SELECT 
    TABLE_NAME AS sequence_name,
    NEXT_NOT_CACHED_VALUE AS current_value,
    MINIMUM_VALUE,
    MAXIMUM_VALUE,
    ROUND((NEXT_NOT_CACHED_VALUE - MINIMUM_VALUE) / 
          (MAXIMUM_VALUE - MINIMUM_VALUE) * 100, 2) AS usage_percent
FROM information_schema.SEQUENCES
WHERE TABLE_SCHEMA = DATABASE();

-- Identifier sequences proches de la limite
SELECT * FROM sequence_health WHERE usage_percent > 90;
```

### ⚠️ Pièges à Éviter

1. **Oublier NOCYCLE avec limites strictes**
```sql
-- ❌ Mauvais : CYCLE sans le vouloir
CREATE SEQUENCE limited_seq
    MAXVALUE 1000;
    -- CYCLE par défaut si on atteint 1000, repart à MINVALUE

-- ✅ Bon : NOCYCLE explicite
CREATE SEQUENCE limited_seq
    MAXVALUE 1000
    NOCYCLE;  -- Erreur explicite si limite atteinte
```

2. **CACHE trop élevé pour séquences critiques**
```sql
-- ❌ Mauvais : Gaps importants possibles
CREATE SEQUENCE audit_log_seq CACHE 10000;
-- Crash serveur = perte potentielle de 10000 numéros

-- ✅ Bon : CACHE minimal pour audit
CREATE SEQUENCE audit_log_seq CACHE 1;
```

3. **Ne pas synchroniser après import de données**
```sql
-- ❌ Oubli de synchronisation
-- Import de factures existantes avec numéros jusqu'à 5000
-- Sequence toujours à 1
-- → Conflit de clés

-- ✅ Synchronisation post-import
SELECT SETVAL(invoice_seq, 
              (SELECT MAX(invoice_number) FROM invoices), 
              true);
```

4. **Utiliser NEXT VALUE dans WHERE/JOIN**
```sql
-- ❌ Erreur : NEXT VALUE consomme la sequence à chaque évaluation
SELECT * FROM orders 
WHERE order_id = NEXT VALUE FOR order_seq;
-- Comportement imprévisible

-- ✅ Assigner d'abord à variable
SET @target_id = NEXT VALUE FOR order_seq;
SELECT * FROM orders WHERE order_id = @target_id;
```

---

## Gestion en Production

### Backup et Restauration

```sql
-- Sauvegarder la configuration des sequences
SELECT 
    CONCAT(
        'CREATE SEQUENCE ', TABLE_NAME,
        ' START WITH ', START_VALUE,
        ' INCREMENT BY ', INCREMENT,
        ' MINVALUE ', MINIMUM_VALUE,
        ' MAXVALUE ', MAXIMUM_VALUE,
        ' CACHE ', CACHE_SIZE,
        IF(CYCLE_OPTION='YES', ' CYCLE', ' NOCYCLE'),
        ';'
    ) AS create_statement
FROM information_schema.SEQUENCES
WHERE TABLE_SCHEMA = 'your_database';

-- Sauvegarder les valeurs courantes
SELECT 
    TABLE_NAME,
    NEXT_NOT_CACHED_VALUE
FROM information_schema.SEQUENCES
WHERE TABLE_SCHEMA = 'your_database';

-- Restauration : Recréer sequences puis SETVAL
SELECT SETVAL(invoice_seq, 12345, true);
```

### Migration et Déploiement

```sql
-- Script de migration (versioning de schéma)
-- migrations/V001__create_sequences.sql

-- Créer sequences avec IF NOT EXISTS
CREATE SEQUENCE IF NOT EXISTS invoice_seq
    START WITH 1000
    INCREMENT BY 1
    CACHE 100;

CREATE SEQUENCE IF NOT EXISTS order_seq
    START WITH 1
    INCREMENT BY 1
    CACHE 500;

-- Rollback script
-- migrations/V001__rollback.sql
DROP SEQUENCE IF EXISTS invoice_seq;
DROP SEQUENCE IF EXISTS order_seq;
```

### Monitoring en Production

```sql
-- Requête pour tableau de bord
SELECT 
    s.TABLE_NAME AS sequence_name,
    s.NEXT_NOT_CACHED_VALUE AS current_value,
    s.MAXIMUM_VALUE AS max_value,
    ROUND((s.NEXT_NOT_CACHED_VALUE / s.MAXIMUM_VALUE) * 100, 2) AS fill_percent,
    s.CACHE_SIZE,
    CASE 
        WHEN s.NEXT_NOT_CACHED_VALUE / s.MAXIMUM_VALUE > 0.9 THEN 'CRITICAL'
        WHEN s.NEXT_NOT_CACHED_VALUE / s.MAXIMUM_VALUE > 0.75 THEN 'WARNING'
        ELSE 'OK'
    END AS status
FROM information_schema.SEQUENCES s
WHERE s.TABLE_SCHEMA = DATABASE()
ORDER BY fill_percent DESC;
```

---

## ✅ Points clés à retenir

### Fondamentaux
- ✅ Les **sequences** sont des objets indépendants générant des nombres séquentiels
- ✅ Offrent plus de **flexibilité et contrôle** qu'AUTO_INCREMENT
- ✅ **Thread-safe** et optimisés pour la concurrence
- ✅ Standard **SQL:2003** (portable PostgreSQL, Oracle)

### Fonctions Essentielles
- ✅ **NEXT VALUE FOR** : Obtenir la prochaine valeur (incrémente)
- ✅ **LASTVAL()** : Dernière valeur générée dans la session
- ✅ **SETVAL()** : Modifier la valeur courante (admin)

### Performance
- ✅ **CACHE** est crucial : 100-1000 pour haute fréquence
- ✅ CACHE élevé = performance ↑ mais gaps possibles en cas de crash
- ✅ NOCACHE = garantie sans gaps mais performance ↓

### Cas d'Usage
- ✅ **Numérotation multi-tables** (entités business partagées)
- ✅ **Schémas métier** (factures, tickets, commandes avec format spécifique)
- ✅ **Batch processing** (traçabilité des lots)
- ✅ **Systèmes distribués** (ranges réservés par datacenter)

### Best Practices
- ✅ Nommage explicite avec suffixe `_seq`
- ✅ MINVALUE/MAXVALUE selon contraintes métier
- ✅ NOCYCLE pour éviter reset silencieux
- ✅ Documenter la politique de reset (annuel, etc.)
- ✅ Monitoring du fill_percent (alertes > 90%)
- ✅ Synchronisation post-import avec SETVAL

### Pièges à Éviter
- ❌ Ne pas oublier SETVAL après import de données
- ❌ CACHE trop élevé pour séquences critiques/audit
- ❌ CYCLE par défaut non voulu (toujours expliciter)
- ❌ Utiliser NEXT VALUE dans WHERE/JOIN

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [CREATE SEQUENCE](https://mariadb.com/kb/en/create-sequence/) - Syntaxe complète
- 📖 [NEXT VALUE FOR](https://mariadb.com/kb/en/next-value-for/) - Fonction de génération
- 📖 [SETVAL](https://mariadb.com/kb/en/setval/) - Modifier une sequence
- 📖 [ALTER SEQUENCE](https://mariadb.com/kb/en/alter-sequence/) - Modification
- 📖 [INFORMATION_SCHEMA.SEQUENCES](https://mariadb.com/kb/en/information-schema-sequences-table/) - Métadonnées

### Articles et Blogs
- 📝 [MariaDB Sequences vs AUTO_INCREMENT](https://mariadb.com/kb/en/sequences-vs-auto_increment/) - Comparaison approfondie
- 📝 [Sequence Performance Best Practices](https://mariadb.org/sequences-performance/)
- 📝 [Using Sequences in Production](https://mariadb.com/resources/blog/sequences-production/)

### Comparaison avec Autres SGBD
- 🔄 [PostgreSQL Sequences](https://www.postgresql.org/docs/current/sql-createsequence.html) - Syntaxe quasi-identique
- 🔄 [Oracle Sequences](https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/CREATE-SEQUENCE.html) - Inspiration originale
- 🔄 [SQL Server IDENTITY vs SEQUENCE](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-sequence-transact-sql)

---

## ➡️ Section suivante

**[18.2 System-Versioned Tables (Tables Temporelles)](./02-system-versioned-tables.md)** : Découvrez comment MariaDB conserve automatiquement l'historique complet des modifications de vos tables pour audit, conformité et analyse forensique.

---


⏭️ [Tables temporelles (System-Versioned Tables)](/18-fonctionnalites-avancees/02-system-versioned-tables.md)

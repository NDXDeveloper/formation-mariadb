🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.2 Tables Temporelles (System-Versioned Tables)

> **Niveau** : Avancé  
> **Durée estimée** : 2-2.5 heures  
> **Prérequis** : Chapitres 2-6, compréhension des transactions et MVCC

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le concept de **versioning système** et son utilité
- Créer et configurer des **tables temporelles** (System-Versioned)
- Maîtriser les **requêtes temporelles** (FOR SYSTEM_TIME AS OF, BETWEEN, ALL)
- Implémenter des **systèmes d'audit** automatiques conformes aux réglementations
- Analyser l'**historique complet** des modifications pour forensique
- Gérer la **table historique** et son partitionnement
- Comprendre l'impact de l'**extension TIMESTAMP 2106** (nouveauté 11.8) 🆕
- Optimiser les performances et le stockage des tables temporelles

---

## Introduction

Les **System-Versioned Tables** (tables temporelles) sont une fonctionnalité puissante qui permet à MariaDB de **conserver automatiquement l'historique complet de toutes les modifications** apportées aux données. Contrairement aux techniques d'audit traditionnelles qui nécessitent des triggers complexes, les tables temporelles offrent un mécanisme transparent et performant.

### Qu'est-ce qu'une Table Temporelle ?

Une table temporelle est une table qui :
1. ✅ **Enregistre automatiquement** chaque version d'une ligne modifiée
2. ✅ **Conserve les timestamps** de début et fin de validité pour chaque version
3. ✅ **Permet de requêter** les données telles qu'elles étaient à n'importe quel moment
4. ✅ **Sépare les données actuelles** (table courante) des données historiques (table d'historique)

**Métaphore** : Une table temporelle est comme une **machine à remonter le temps** pour vos données - vous pouvez voir votre base de données telle qu'elle était hier, la semaine dernière, ou il y a 6 mois.

### Pourquoi Utiliser les Tables Temporelles ?

**Cas d'usage principaux** :

1. **📋 Audit et Conformité Réglementaire**
   - RGPD (Article 5 : exactitude et conservation)
   - SOX (Sarbanes-Oxley) : traçabilité des données financières
   - HIPAA : historique des dossiers médicaux
   - Exigences sectorielles (banque, assurance, santé)

2. **🔍 Analyse Forensique**
   - Enquêtes sur incidents de sécurité
   - Détection de fraudes et anomalies
   - Analyse rétrospective de décisions business
   - Reconstruction d'états passés

3. **⏮️ Point-in-Time Recovery (Application Level)**
   - Restaurer des données à un instant précis
   - Annuler des modifications erronées
   - Comparer états avant/après un changement

4. **📊 Analyse Temporelle et BI**
   - Évolution de métriques dans le temps
   - Analyse de tendances historiques
   - Rapports "as-was" vs "as-is"

5. **🔄 Historisation Métier**
   - Suivi des modifications de prix
   - Évolution des contrats et conditions
   - Historique des affectations (employés, projets)

💡 **Avantage clé** : Tout ceci **sans une seule ligne de code applicatif** - MariaDB gère tout automatiquement.

---

## Concepts Fondamentaux

### Architecture System-Versioned

Une table temporelle est composée de **deux tables physiques** :

```
┌─────────────────────────────────────┐
│   TABLE COURANTE (employees)        │
│   Données actuellement valides      │
│   ┌────┬────────┬────────┬─────┐    │
│   │ id │  name  │ salary │ ... │    │
│   ├────┼────────┼────────┼─────┤    │
│   │ 1  │ Alice  │ 55000  │ ... │    │
│   │ 2  │ Bob    │ 60000  │ ... │    │
│   └────┴────────┴────────┴─────┘    │
│                                     │
│   Colonnes cachées (automatiques):  │
│   - row_start: TIMESTAMP(6)         │
│   - row_end:   TIMESTAMP(6)         │
└─────────────────────────────────────┘
              │
              │ Lors d'UPDATE/DELETE
              ↓
┌───────────────────────────────────────────────────────┐
│ TABLE HISTORIQUE (employees_history)                  │
│   Versions précédentes des lignes                     │
│   ┌────┬────────┬────────┬────────────┬───────────┐   │
│   │ id │  name  │ salary │ row_start  │ row_end   │   │
│   ├────┼────────┼────────┼────────────┼───────────┤   │
│   │ 1  │ Alice  │ 50000  │ 2024-01-01 │ 2024-06-01│   │
│   │ 1  │ Alice  │ 52000  │ 2024-06-01 │ 2025-01-01│   │
│   │ 2  │ Bob    │ 55000  │ 2024-01-01 │ 2024-11-01│   │
│   └────┴────────┴────────┴────────────┴───────────┘   │
└───────────────────────────────────────────────────────┘
```

### Colonnes de Versioning (row_start et row_end)

MariaDB ajoute automatiquement deux colonnes **TIMESTAMP(6)** (précision microseconde) :

- **row_start** : Date/heure de début de validité de la version
- **row_end** : Date/heure de fin de validité de la version

**Convention** :
- Ligne **active** : `row_end = 2038-01-19 03:14:07.999999` (valeur spéciale maximale)
- Ligne **historique** : `row_end = timestamp réel de modification`

🆕 **MariaDB 11.8 : Extension TIMESTAMP 2106** :
Avant 11.8, TIMESTAMP était limité à 2038 (problème Y2038). Depuis 11.8, l'extension permet des dates jusqu'à **2106**, résolvant ce problème pour les tables temporelles.

```sql
-- MariaDB 11.8+
-- row_end pour ligne active = 2106-02-07 06:28:15.999999
-- (au lieu de 2038-01-19 03:14:07.999999)
```

### Cycle de Vie d'une Ligne

```sql
-- INSERT d'une nouvelle ligne
INSERT INTO employees (id, name, salary) VALUES (1, 'Alice', 50000);

-- Table courante :
-- id=1, name='Alice', salary=50000
-- row_start=2024-01-01 10:00:00.123456
-- row_end=2106-02-07 06:28:15.999999 (ligne active)

-- Table historique : (vide)

-- ─────────────────────────────────────

-- UPDATE de la ligne
UPDATE employees SET salary = 55000 WHERE id = 1;

-- Table courante :
-- id=1, name='Alice', salary=55000
-- row_start=2024-06-15 14:30:00.654321 (nouveau)
-- row_end=2106-02-07 06:28:15.999999 (active)

-- Table historique : (ancienne version déplacée)
-- id=1, name='Alice', salary=50000
-- row_start=2024-01-01 10:00:00.123456
-- row_end=2024-06-15 14:30:00.654321

-- ─────────────────────────────────────

-- DELETE de la ligne
DELETE FROM employees WHERE id = 1;

-- Table courante : (ligne supprimée)

-- Table historique : (toutes les versions conservées)
-- Version 1: salary=50000, row_start=2024-01-01, row_end=2024-06-15
-- Version 2: salary=55000, row_start=2024-06-15, row_end=2024-12-10
```

💡 **Point clé** : Les données ne sont **jamais perdues** - même après DELETE, toutes les versions restent dans la table historique.

---

## Standard SQL:2011 et Compatibilité

Les tables temporelles MariaDB implémentent le standard **SQL:2011 Temporal**.

**Conformité au standard** :
- ✅ Clause `WITH SYSTEM VERSIONING`
- ✅ Colonnes `row_start` et `row_end` automatiques
- ✅ Requêtes `FOR SYSTEM_TIME`
- ✅ Séparation table courante / historique

**Extensions MariaDB** :
- 🔧 Flexibilité sur le nom de la table historique
- 🔧 Support des partitions sur table historique
- 🔧 Transactions distribuées (XA)

**Compatibilité avec autres SGBD** :
- **SQL Server** : Temporal Tables (syntaxe quasi-identique)
- **Oracle** : Flashback Query (mécanisme différent)
- **PostgreSQL** : Nécessite extensions (temporal_tables)

---

## Types de Tables Temporelles

MariaDB supporte deux types de versioning :

### 1. System Versioning (Timestamps Système)

**Colonnes gérées par le SGBD** :
```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  price DECIMAL(10,2)
) WITH SYSTEM VERSIONING;

-- MariaDB ajoute automatiquement :
-- row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START
-- row_end TIMESTAMP(6) GENERATED ALWAYS AS ROW END
-- PERIOD FOR SYSTEM_TIME (row_start, row_end)
```

**Avantages** :
- ✅ Transparent : aucune modification applicative
- ✅ Précis : microseconde
- ✅ Fiable : impossible de manipuler les timestamps

**Limitation** :
- ⚠️ Timestamps basés sur horloge serveur (attention fuseaux horaires)

### 2. Application-Time Period (Timestamps Applicatifs)

**Colonnes gérées par l'application** :
```sql
CREATE TABLE contracts (
  contract_id INT PRIMARY KEY,
  valid_from DATE,
  valid_until DATE,
  -- ... autres colonnes
  PERIOD FOR contract_validity (valid_from, valid_until)
);
```

**Usage** : Périodes métier (voir section 18.3 Application Time Period Tables)

💡 **Ce chapitre se concentre sur System Versioning** (le plus courant pour l'audit).

---

## Requêtes Temporelles : FOR SYSTEM_TIME

La clause **FOR SYSTEM_TIME** permet d'interroger les données à différents moments.

### Syntaxe Générale

```sql
SELECT columns
FROM table_name
FOR SYSTEM_TIME { AS OF | BETWEEN | FROM ... TO | ALL }
WHERE conditions;
```

### FOR SYSTEM_TIME AS OF - État à un Instant T

Récupère les lignes **valides à un moment précis**.

```sql
-- Quel était le salaire d'Alice le 1er mars 2024 ?
SELECT * FROM employees
FOR SYSTEM_TIME AS OF TIMESTAMP '2024-03-01 12:00:00'
WHERE name = 'Alice';

-- Avec datetime
SELECT * FROM employees
FOR SYSTEM_TIME AS OF '2024-03-01 12:00:00';

-- Avec variable
SET @target_date = '2024-03-01';
SELECT * FROM employees
FOR SYSTEM_TIME AS OF @target_date;

-- Relativement (il y a 7 jours)
SELECT * FROM employees
FOR SYSTEM_TIME AS OF (NOW() - INTERVAL 7 DAY);
```

**Logique** :
```sql
-- Équivalent conceptuel (simplifié)
SELECT * FROM (
  SELECT * FROM employees           -- Table courante
  UNION ALL
  SELECT * FROM employees_history   -- Table historique
) AS all_versions
WHERE row_start <= @target_timestamp
  AND row_end > @target_timestamp;
```

### FOR SYSTEM_TIME BETWEEN - Période avec Bornes

Récupère les lignes **valides à n'importe quel moment** dans la période.

```sql
-- Toutes les versions d'Alice entre janvier et juin 2024
SELECT * FROM employees
FOR SYSTEM_TIME BETWEEN 
  TIMESTAMP '2024-01-01' AND TIMESTAMP '2024-06-30'
WHERE name = 'Alice';

-- Lignes qui ont existé durant le Q2 2024
SELECT DISTINCT employee_id, name
FROM employees
FOR SYSTEM_TIME BETWEEN '2024-04-01' AND '2024-06-30';
```

**Logique** :
```sql
-- Inclut les lignes où :
-- row_start < end_timestamp AND row_end > start_timestamp
```

### FOR SYSTEM_TIME FROM ... TO - Période Asymétrique

Similaire à BETWEEN, mais **borne de fin exclusive**.

```sql
-- De début 2024 jusqu'à (mais excluant) début 2025
SELECT * FROM employees
FOR SYSTEM_TIME FROM 
  TIMESTAMP '2024-01-01' TO TIMESTAMP '2025-01-01';

-- Équivalent à :
-- row_start < '2025-01-01' AND row_end > '2024-01-01'
```

💡 **BETWEEN vs FROM...TO** :
- `BETWEEN '2024-01-01' AND '2024-12-31'` : inclut jusqu'à la fin du 31 décembre
- `FROM '2024-01-01' TO '2025-01-01'` : exclut le 1er janvier 2025

### FOR SYSTEM_TIME ALL - Historique Complet

Récupère **toutes les versions** de toutes les lignes (courante + historique).

```sql
-- Historique complet d'Alice avec timestamps
SELECT 
  employee_id,
  name,
  salary,
  department,
  row_start,
  row_end
FROM employees
FOR SYSTEM_TIME ALL
WHERE name = 'Alice'
ORDER BY row_start;

-- Résultat :
-- employee_id | name  | salary | department | row_start           | row_end
-- 1           | Alice | 50000  | IT         | 2024-01-01 10:00:00 | 2024-06-15 14:30:00
-- 1           | Alice | 55000  | IT         | 2024-06-15 14:30:00 | 2024-11-20 09:15:00
-- 1           | Alice | 60000  | Engineering| 2024-11-20 09:15:00 | 2106-02-07 06:28:15
```

**Cas d'usage** :
- Audit complet d'une entité
- Analyse d'évolution dans le temps
- Détection d'anomalies
- Rapports de conformité

---

## Cas d'Usage Détaillés

### 1. Audit de Conformité RGPD

**Exigence RGPD Article 5** : Traçabilité des modifications de données personnelles.

```sql
-- Table d'utilisateurs avec données personnelles
CREATE TABLE users (
  user_id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  phone VARCHAR(20),
  consent_marketing BOOLEAN,
  consent_date TIMESTAMP
) WITH SYSTEM VERSIONING;

-- Scénario : Utilisateur exerce son droit à l'effacement
-- Avant suppression, extraire historique complet
SELECT 
  user_id,
  email,
  first_name,
  last_name,
  consent_marketing,
  row_start AS valid_from,
  row_end AS valid_until
FROM users
FOR SYSTEM_TIME ALL
WHERE user_id = 12345
ORDER BY row_start;

-- Générer rapport d'audit pour autorités
SELECT 
  'Data deletion request executed' AS action,
  user_id,
  email,
  NOW() AS deletion_timestamp,
  COUNT(*) AS versions_deleted
FROM users
FOR SYSTEM_TIME ALL
WHERE user_id = 12345;

-- Puis supprimer
DELETE FROM users WHERE user_id = 12345;

-- L'historique reste disponible dans users_history pour audit
```

### 2. Analyse Forensique de Fraude

**Scénario** : Détection d'une modification suspecte de données bancaires.

```sql
-- Table de comptes bancaires
CREATE TABLE bank_accounts (
  account_id INT PRIMARY KEY,
  account_number VARCHAR(20),
  holder_name VARCHAR(100),
  balance DECIMAL(15,2),
  status ENUM('ACTIVE','FROZEN','CLOSED')
) WITH SYSTEM VERSIONING;

-- Alerte : Transfert suspect détecté le 15/12/2024
-- Investigation : Qui a modifié le compte dans les 48h précédentes ?

SELECT 
  account_id,
  holder_name,
  balance,
  status,
  row_start AS modification_time,
  row_end AS next_modification,
  TIMESTAMPDIFF(SECOND, row_start, row_end) AS duration_seconds
FROM bank_accounts
FOR SYSTEM_TIME BETWEEN 
  TIMESTAMP '2024-12-13 00:00:00' AND TIMESTAMP '2024-12-15 23:59:59'
WHERE account_id = 789456
ORDER BY row_start;

-- Identifier modifications rapides suspectes (<5 minutes entre changements)
WITH account_changes AS (
  SELECT 
    account_id,
    balance,
    row_start,
    LAG(balance) OVER (ORDER BY row_start) AS previous_balance,
    LAG(row_start) OVER (ORDER BY row_start) AS previous_time
  FROM bank_accounts
  FOR SYSTEM_TIME ALL
  WHERE account_id = 789456
)
SELECT *,
  balance - previous_balance AS balance_change,
  TIMESTAMPDIFF(MINUTE, previous_time, row_start) AS minutes_since_last_change
FROM account_changes
WHERE TIMESTAMPDIFF(MINUTE, previous_time, row_start) < 5;
```

### 3. Restauration Point-in-Time Applicative

**Scénario** : Batch nocturne a corrompu des données, restaurer état 22h hier.

```sql
-- Table de produits avec prix
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  product_name VARCHAR(100),
  price DECIMAL(10,2),
  stock_quantity INT
) WITH SYSTEM VERSIONING;

-- Batch de mise à jour nocturne (02h) a mal fonctionné
-- Restaurer les prix à 22h la veille

-- Étape 1 : Identifier les produits affectés
CREATE TEMPORARY TABLE corrupted_products AS
SELECT DISTINCT product_id
FROM products
FOR SYSTEM_TIME BETWEEN 
  TIMESTAMP '2024-12-14 22:00:00' AND NOW();

-- Étape 2 : Récupérer état correct (22h)
CREATE TEMPORARY TABLE correct_state AS
SELECT product_id, product_name, price, stock_quantity
FROM products
FOR SYSTEM_TIME AS OF TIMESTAMP '2024-12-14 22:00:00'
WHERE product_id IN (SELECT product_id FROM corrupted_products);

-- Étape 3 : Restaurer (UPDATE ou REPLACE)
REPLACE INTO products (product_id, product_name, price, stock_quantity)
SELECT * FROM correct_state;

-- Vérification
SELECT 
  p_now.product_id,
  p_now.price AS current_price,
  p_22h.price AS price_at_22h,
  p_now.price - p_22h.price AS price_difference
FROM products p_now
INNER JOIN correct_state p_22h ON p_now.product_id = p_22h.product_id
WHERE p_now.price != p_22h.price;
```

### 4. Analyse Business : Évolution des Prix

**Scénario** : Analyser impact des changements de prix sur les ventes.

```sql
CREATE TABLE product_prices (
  product_id INT PRIMARY KEY,
  product_name VARCHAR(100),
  price DECIMAL(10,2),
  category VARCHAR(50)
) WITH SYSTEM VERSIONING;

-- Requête : Historique complet des prix d'un produit
SELECT 
  product_id,
  product_name,
  price,
  row_start AS price_from,
  row_end AS price_until,
  DATEDIFF(
    IF(row_end = '2106-02-07 06:28:15.999999', NOW(), row_end),
    row_start
  ) AS days_at_this_price
FROM product_prices
FOR SYSTEM_TIME ALL
WHERE product_id = 42
ORDER BY row_start;

-- Analyse : Nombre de changements de prix par produit (derniers 6 mois)
SELECT 
  product_id,
  product_name,
  COUNT(*) - 1 AS price_changes,
  MIN(price) AS min_price,
  MAX(price) AS max_price,
  STDDEV(price) AS price_volatility
FROM product_prices
FOR SYSTEM_TIME BETWEEN 
  (NOW() - INTERVAL 6 MONTH) AND NOW()
GROUP BY product_id, product_name
HAVING price_changes > 0
ORDER BY price_changes DESC;

-- Corréler avec ventes (nécessite table sales)
SELECT 
  pp.product_id,
  pp.product_name,
  pp.price,
  pp.row_start,
  pp.row_end,
  COUNT(s.sale_id) AS sales_count,
  SUM(s.quantity * pp.price) AS revenue
FROM product_prices pp
FOR SYSTEM_TIME ALL
LEFT JOIN sales s ON s.product_id = pp.product_id
  AND s.sale_date >= pp.row_start
  AND s.sale_date < pp.row_end
WHERE pp.product_id = 42
GROUP BY pp.product_id, pp.price, pp.row_start
ORDER BY pp.row_start;
```

### 5. Comparaison Avant/Après un Changement

**Scénario** : Migration système le 01/12, comparer données avant/après.

```sql
-- État avant migration (30/11 à 23h59)
CREATE TEMPORARY TABLE state_before AS
SELECT * FROM critical_table
FOR SYSTEM_TIME AS OF '2024-11-30 23:59:59';

-- État après migration (02/12 à 01h00)
CREATE TEMPORARY TABLE state_after AS
SELECT * FROM critical_table
FOR SYSTEM_TIME AS OF '2024-12-02 01:00:00';

-- Différences
SELECT 
  COALESCE(b.record_id, a.record_id) AS record_id,
  CASE
    WHEN b.record_id IS NULL THEN 'ADDED'
    WHEN a.record_id IS NULL THEN 'REMOVED'
    ELSE 'MODIFIED'
  END AS change_type,
  b.important_field AS before_value,
  a.important_field AS after_value
FROM state_before b
FULL OUTER JOIN state_after a ON b.record_id = a.record_id
WHERE b.important_field != a.important_field
   OR b.record_id IS NULL
   OR a.record_id IS NULL;
```

---

## Gestion de la Table Historique

### Nom de la Table Historique

Par défaut, MariaDB nomme la table historique `{table_name}_history` :

```sql
CREATE TABLE employees (...) WITH SYSTEM VERSIONING;
-- Crée automatiquement : employees_history
```

**Personnaliser le nom** :
```sql
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  name VARCHAR(100)
) WITH SYSTEM VERSIONING 
  HISTORY TABLE = employee_audit_log;
```

### Structure de la Table Historique

```sql
-- Inspecter la table historique
SHOW CREATE TABLE employees_history;

-- Résultat :
CREATE TABLE `employees_history` (
  `employee_id` int(11) NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `row_start` timestamp(6) GENERATED ALWAYS AS ROW START,
  `row_end` timestamp(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (`row_start`, `row_end`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Structure identique à la table courante + row_start/row_end explicites
```

### Requêtes Directes sur Table Historique

```sql
-- Accès direct (rarement nécessaire)
SELECT * FROM employees_history
WHERE employee_id = 1
ORDER BY row_start;

-- Préférer FOR SYSTEM_TIME ALL (inclut table courante + historique)
SELECT * FROM employees
FOR SYSTEM_TIME ALL
WHERE employee_id = 1
ORDER BY row_start;
```

💡 **Best practice** : Toujours utiliser `FOR SYSTEM_TIME` plutôt que requêter directement la table historique.

### Suppression de la Table Historique

```sql
-- Impossible tant que le versioning est actif
DROP TABLE employees_history;
-- ERROR: "Versioned table employees_history in use"

-- Solution : Désactiver versioning d'abord
ALTER TABLE employees DROP SYSTEM VERSIONING;
DROP TABLE employees_history;

-- Ou supprimer les deux ensemble
DROP TABLE employees;
-- Supprime employees ET employees_history automatiquement
```

---

## Performance et Optimisation

### Impact sur les Performances

**Overhead d'écriture** :

| Opération | Sans Versioning | Avec Versioning | Overhead |
|-----------|-----------------|-----------------|----------|
| INSERT | Baseline | Baseline + 2 colonnes | ~5% |
| UPDATE | Baseline | INSERT dans historique | ~15-25% |
| DELETE | Baseline | INSERT dans historique | ~15-25% |
| SELECT (sans FOR SYSTEM_TIME) | Baseline | Baseline | 0% |

💡 **Point clé** : Les SELECT normaux ne sont **pas affectés** - seules les écritures ont un overhead.

### Optimisations

#### 1. Index sur row_start/row_end

```sql
-- Table historique automatiquement indexée sur row_end
-- Ajouter index composite pour requêtes fréquentes
ALTER TABLE employees_history
  ADD INDEX idx_id_time (employee_id, row_start, row_end);

-- Optimise les requêtes :
SELECT * FROM employees
FOR SYSTEM_TIME AS OF '2024-06-01'
WHERE employee_id = 123;
```

#### 2. Partitionnement de la Table Historique

**Stratégie recommandée** : Partitionner par `row_end` (date de fin de validité).

```sql
-- Créer table historique partitionnée
CREATE TABLE employees_history (
  employee_id INT,
  name VARCHAR(100),
  salary DECIMAL(10,2),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
)
PARTITION BY RANGE (YEAR(row_end)) (
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION p_current VALUES LESS THAN MAXVALUE
);

-- Créer table courante avec référence
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  name VARCHAR(100),
  salary DECIMAL(10,2)
) WITH SYSTEM VERSIONING
  HISTORY TABLE = employees_history;
```

**Avantages du partitionnement** :
- ✅ Requêtes temporelles plus rapides (partition pruning)
- ✅ Archivage/suppression facile de partitions anciennes
- ✅ Maintenance simplifiée (OPTIMIZE PARTITION)

#### 3. Archivage de Données Anciennes

```sql
-- Stratégie : Déplacer données >2 ans vers table d'archive

-- Créer table d'archive
CREATE TABLE employees_archive LIKE employees_history;

-- Copier anciennes données
INSERT INTO employees_archive
SELECT * FROM employees_history
WHERE row_end < (NOW() - INTERVAL 2 YEAR);

-- Supprimer de l'historique actif
DELETE FROM employees_history
WHERE row_end < (NOW() - INTERVAL 2 YEAR);

-- OPTIMIZE pour récupérer espace
OPTIMIZE TABLE employees_history;
```

**Alternative avec partitions** :
```sql
-- Supprimer partition entière (instantané)
ALTER TABLE employees_history DROP PARTITION p2022;
```

#### 4. Compression de la Table Historique

```sql
-- Table historique rarement modifiée = excellente candidate compression
ALTER TABLE employees_history 
  ROW_FORMAT=COMPRESSED 
  KEY_BLOCK_SIZE=8;

-- Ou compression transparente (punch hole)
ALTER TABLE employees_history 
  PAGE_COMPRESSED=1 
  PAGE_COMPRESSION_LEVEL=6;

-- Économie typique : 50-70% d'espace
```

---

## Limitations et Contraintes

### Ce qui N'est PAS Supporté

❌ **Colonnes AUTO_INCREMENT dans table historique**
```sql
-- Ne fonctionne pas :
CREATE TABLE orders (
  order_id INT AUTO_INCREMENT PRIMARY KEY,
  amount DECIMAL(10,2)
) WITH SYSTEM VERSIONING;
-- La table historique ne peut avoir AUTO_INCREMENT

-- Solution : Utiliser SEQUENCE ou timestamp pour clé primaire
```

❌ **Triggers sur table historique**
```sql
-- Impossible de créer triggers sur employees_history
CREATE TRIGGER audit_trigger
AFTER INSERT ON employees_history
FOR EACH ROW ...;
-- ERROR: Triggers not allowed on system-versioned tables
```

❌ **Modification manuelle de row_start/row_end**
```sql
-- Ces colonnes sont GENERATED ALWAYS - lecture seule
UPDATE employees SET row_start = NOW();
-- ERROR: Cannot update generated column
```

### Contraintes Techniques

⚠️ **Taille de la table historique**
- Croissance linéaire avec nombre de modifications
- Planning de capacité requis : estimer volume mensuel/annuel

⚠️ **Transactions longues**
- Les versions créées durant une transaction restent dans la table courante jusqu'au COMMIT
- Impact mémoire sur transactions très longues

⚠️ **Réplication**
- Les données historiques sont répliquées normalement
- Attention à la charge réseau si volume important

⚠️ **Dump et Restore**
```sql
-- mysqldump/mariadb-dump inclut les deux tables
-- Lors de la restauration, bien restaurer dans l'ordre :
-- 1. Table historique d'abord
-- 2. Table courante ensuite (avec référence)
```

---

## Sécurité et Droits d'Accès

### Isolation des Privilèges

```sql
-- Utilisateur applicatif : accès table courante uniquement
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.employees TO 'app_user'@'%';

-- Pas d'accès direct à la table historique
-- (accès via FOR SYSTEM_TIME autorisé car indirecte)

-- Utilisateur audit : lecture seule sur historique
GRANT SELECT ON mydb.employees_history TO 'auditor'@'%';

-- DBA : gestion complète
GRANT ALL ON mydb.* TO 'dba_user'@'localhost';
```

### Protection de l'Historique

```sql
-- Empêcher suppression accidentelle de l'historique
-- Option 1 : Privilèges stricts (pas de DROP sur table historique)

-- Option 2 : Backup régulier de la table historique
mysqldump --single-transaction mydb employees_history > history_backup.sql

-- Option 3 : Réplication dédiée pour audit
-- Serveur séparé en lecture seule pour équipe audit
```

---

## Migration vers System-Versioned

### Ajouter Versioning à Table Existante

```sql
-- Table existante sans versioning
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  price DECIMAL(10,2)
);

-- Données existantes
INSERT INTO products VALUES (1, 'Widget', 19.99);
INSERT INTO products VALUES (2, 'Gadget', 29.99);

-- Activer versioning
ALTER TABLE products ADD SYSTEM VERSIONING;

-- Résultat :
-- 1. Colonnes row_start/row_end ajoutées à products
-- 2. Table products_history créée (vide initialement)
-- 3. Données existantes reçoivent row_start = NOW()

-- Vérification
SELECT *, row_start, row_end FROM products;
-- product_id | name   | price | row_start           | row_end
-- 1          | Widget | 19.99 | 2024-12-14 10:00:00 | 2106-02-07 06:28:15

-- Désormais, modifications trackées automatiquement
UPDATE products SET price = 24.99 WHERE product_id = 1;

-- Historique disponible
SELECT * FROM products FOR SYSTEM_TIME ALL WHERE product_id = 1;
-- Version 1: price=19.99, row_end=2024-12-14 10:05:00
-- Version 2: price=24.99, row_end=2106-02-07 06:28:15 (actuelle)
```

### Désactiver Versioning

```sql
-- Désactiver temporairement (conserve données historiques)
ALTER TABLE products DROP SYSTEM VERSIONING;

-- Table products : colonnes row_start/row_end supprimées
-- Table products_history : conservée mais déconnectée

-- Requêtes temporelles ne fonctionnent plus
SELECT * FROM products FOR SYSTEM_TIME AS OF '2024-12-14';
-- ERROR: products is not system-versioned

-- Réactiver
ALTER TABLE products ADD SYSTEM VERSIONING;
-- Reconnecte à products_history existante
```

---

## Monitoring et Administration

### Requêtes de Monitoring

```sql
-- 1. Lister toutes les tables versionnées
SELECT 
  TABLE_SCHEMA,
  TABLE_NAME,
  ENGINE
FROM information_schema.TABLES
WHERE TABLE_TYPE = 'SYSTEM VERSIONED';

-- 2. Taille des tables courantes vs historiques
SELECT 
  t.TABLE_NAME,
  t.TABLE_ROWS AS current_rows,
  ROUND(t.DATA_LENGTH / 1024 / 1024, 2) AS current_size_mb,
  h.TABLE_ROWS AS history_rows,
  ROUND(h.DATA_LENGTH / 1024 / 1024, 2) AS history_size_mb,
  ROUND(h.DATA_LENGTH / t.DATA_LENGTH, 2) AS history_ratio
FROM information_schema.TABLES t
LEFT JOIN information_schema.TABLES h 
  ON h.TABLE_NAME = CONCAT(t.TABLE_NAME, '_history')
  AND h.TABLE_SCHEMA = t.TABLE_SCHEMA
WHERE t.TABLE_TYPE = 'SYSTEM VERSIONED'
  AND t.TABLE_SCHEMA = 'mydb';

-- 3. Activité récente (modifications dernières 24h)
SELECT 
  TABLE_NAME,
  COUNT(*) AS versions_created
FROM employees_history
WHERE row_end > (NOW() - INTERVAL 24 HOUR)
GROUP BY TABLE_NAME;

-- 4. Top 10 lignes les plus modifiées
SELECT 
  employee_id,
  name,
  COUNT(*) AS modification_count,
  MIN(row_start) AS first_seen,
  MAX(row_start) AS last_modified
FROM employees
FOR SYSTEM_TIME ALL
GROUP BY employee_id, name
ORDER BY modification_count DESC
LIMIT 10;
```

### Maintenance Régulière

```sql
-- Script de maintenance mensuel
DELIMITER $$
CREATE PROCEDURE maintain_versioned_tables()
BEGIN
  DECLARE done INT DEFAULT FALSE;
  DECLARE tbl VARCHAR(64);
  DECLARE cur CURSOR FOR 
    SELECT TABLE_NAME 
    FROM information_schema.TABLES
    WHERE TABLE_TYPE = 'SYSTEM VERSIONED'
      AND TABLE_SCHEMA = DATABASE();
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
  
  OPEN cur;
  
  read_loop: LOOP
    FETCH cur INTO tbl;
    IF done THEN
      LEAVE read_loop;
    END IF;
    
    -- Analyser table historique
    SET @sql = CONCAT('ANALYZE TABLE ', tbl, '_history');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- Supprimer données >3 ans si applicable
    SET @sql = CONCAT(
      'DELETE FROM ', tbl, '_history ',
      'WHERE row_end < (NOW() - INTERVAL 3 YEAR)'
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
  END LOOP;
  
  CLOSE cur;
END$$
DELIMITER ;

-- Planifier avec Event Scheduler
CREATE EVENT monthly_versioned_maintenance
ON SCHEDULE EVERY 1 MONTH
DO CALL maintain_versioned_tables();
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **System-Versioned Tables** : Historisation automatique de toutes les modifications
- ✅ **Architecture** : Table courante (données actives) + Table historique (versions passées)
- ✅ **Colonnes automatiques** : `row_start` et `row_end` (TIMESTAMP(6), précision microseconde)
- 🆕 **MariaDB 11.8** : Extension TIMESTAMP 2106 (résout problème Y2038)

### Requêtes Temporelles
- ✅ **FOR SYSTEM_TIME AS OF** : État à un instant T
- ✅ **FOR SYSTEM_TIME BETWEEN** : Période avec bornes inclusives
- ✅ **FOR SYSTEM_TIME FROM...TO** : Période avec borne de fin exclusive
- ✅ **FOR SYSTEM_TIME ALL** : Historique complet (toutes versions)

### Cas d'Usage
- ✅ **Audit et conformité** : RGPD, SOX, HIPAA - traçabilité automatique
- ✅ **Forensique** : Investigation incidents, détection fraudes
- ✅ **Point-in-Time Recovery** : Restauration applicative à instant T
- ✅ **Analyse temporelle** : Évolution métriques, tendances, rapports as-was

### Performance
- ✅ **Overhead écritures** : 15-25% sur UPDATE/DELETE (INSERT ~5%)
- ✅ **SELECT normaux** : Aucun impact (0% overhead)
- ✅ **Optimisations** : Index sur row_start/row_end, partitionnement, compression
- ✅ **Archivage** : Déplacer données anciennes hors table historique active

### Best Practices
- ✅ Partitionner la table historique par `row_end` (YEAR ou QUARTER)
- ✅ Compresser la table historique (ROW_FORMAT=COMPRESSED ou PAGE_COMPRESSED)
- ✅ Index composite : (clé_métier, row_start, row_end)
- ✅ Politique de rétention : archiver/supprimer données >2-3 ans selon besoins
- ✅ Monitoring : taille historique, ratio current/history, activité

### Limitations
- ❌ Pas de triggers sur table historique
- ❌ Pas d'AUTO_INCREMENT dans table historique
- ❌ row_start/row_end en lecture seule (GENERATED ALWAYS)
- ⚠️ Planning capacité requis (historique croît indéfiniment sans archivage)

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [System-Versioned Tables](https://mariadb.com/kb/en/system-versioned-tables/) - Guide complet
- 📖 [Temporal Tables Overview](https://mariadb.com/kb/en/temporal-tables/) - Vue d'ensemble
- 📖 [FOR SYSTEM_TIME](https://mariadb.com/kb/en/select-for-system-time/) - Syntaxe des requêtes
- 📖 [Partition Management](https://mariadb.com/kb/en/partitioning-types-overview/) - Partitionnement historique
- 🆕 [TIMESTAMP Extension 2106](https://mariadb.com/kb/en/timestamp/) - Nouveauté 11.8

### Standards SQL
- 📚 [ISO SQL:2011 Temporal](https://en.wikipedia.org/wiki/SQL:2011#Temporal_support) - Standard international
- 📚 [Temporal Data Management](https://www2.cs.arizona.edu/~rts/tdbbook.pdf) - Livre de référence

### Cas d'Usage et Best Practices
- 📝 [GDPR Compliance with Temporal Tables](https://mariadb.com/resources/blog/gdpr-compliance-temporal-tables/)
- 📝 [Audit Trail Best Practices](https://mariadb.com/kb/en/audit-trail-best-practices/)
- 📝 [Partition Pruning for History Tables](https://mariadb.com/resources/blog/partition-pruning/)

### Comparaison avec Autres SGBD
- 🔄 [SQL Server Temporal Tables](https://docs.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables) - Syntaxe similaire
- 🔄 [Oracle Flashback Query](https://docs.oracle.com/en/database/oracle/oracle-database/21/admin/managing-flashback-database.html)

---

## ➡️ Sous-sections suivantes

### **18.2.1 Création et Configuration**
Syntaxe détaillée CREATE TABLE WITH SYSTEM VERSIONING, options de configuration, personnalisation de la table historique.

### **18.2.2 Requêtes Temporelles (AS OF, BETWEEN, FROM...TO)**
Exemples approfondis de chaque type de requête temporelle avec cas d'usage concrets.

### **18.2.3 Partitionnement des Données Historiques**
Stratégies de partitionnement RANGE/LIST par row_end, automatisation avec Events, archivage.

---


⏭️ [Création et configuration](/18-fonctionnalites-avancees/02.1-creation-configuration.md)

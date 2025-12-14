🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.3 Impact sur Migration et Compatibilité 🔄

> **Niveau** : Tous niveaux (Veille technologique)  
> **Durée estimée** : 30-35 minutes  
> **Prérequis** : Connaissance de base de MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Évaluer la complexité de migration vers MariaDB 11.8 depuis votre version actuelle
- Identifier les points de rupture et changements de comportement
- Planifier une stratégie de migration adaptée à votre contexte
- Anticiper les risques et définir des plans de contingence
- Utiliser les outils et méthodes appropriés pour une migration réussie
- Valider la compatibilité de vos applications

---

## Introduction

MariaDB 11.8 LTS introduit des **innovations majeures** tout en maintenant un **niveau élevé de rétrocompatibilité**. Cependant, certains changements de comportement par défaut et nouvelles fonctionnalités nécessitent une **planification minutieuse** de la migration.

Cette section vous guide à travers les **impacts techniques**, les **stratégies de migration**, et les **bonnes pratiques** pour une transition réussie vers MariaDB 11.8 LTS.

---

## 📊 Matrice de Compatibilité Globale

### Vue d'ensemble par version source

| Version Source | Compatibilité | Complexité | Durée Estimée | Risque | Recommandation |
|----------------|---------------|------------|---------------|--------|----------------|
| **MariaDB 11.7** | 🟢 Excellente | Très faible | 1-2 jours | Minimal | ✅ Migration immédiate |
| **MariaDB 11.6** | 🟢 Excellente | Très faible | 1-2 jours | Minimal | ✅ Migration immédiate |
| **MariaDB 11.5** | 🟢 Excellente | Faible | 2-3 jours | Minimal | ✅ Migration immédiate |
| **MariaDB 11.4 LTS** | 🟢 Excellente | Faible | 3-5 jours | Faible | ✅ Migration recommandée |
| **MariaDB 11.0-11.3** | 🟢 Très bonne | Faible | 1 semaine | Faible | ✅ Migration recommandée |
| **MariaDB 10.11 LTS** | 🟡 Bonne | Moyenne | 2-3 semaines | Modéré | 🟡 Planifier Q1-Q2 2026 |
| **MariaDB 10.6 LTS** | 🟡 Bonne | Moyenne | 3-4 semaines | Modéré | 🟡 Planifier 2026 |
| **MariaDB 10.5** | 🟠 Acceptable | Élevée | 1-2 mois | Significatif | ⚠️ Tests approfondis |
| **MariaDB 10.4 et ant.** | 🟠 Acceptable | Élevée | 2-3 mois | Significatif | ⚠️ Migration par phases |
| **MySQL 8.0** | 🟡 Bonne | Moyenne | 2-4 semaines | Modéré | 🟡 Attention aux divergences |
| **MySQL 5.7** | 🔴 Difficile | Très élevée | 2-6 mois | Élevé | 🔴 POC obligatoire |
| **Autres SGBD** | 🔴 Complexe | Très élevée | 3-12 mois | Très élevé | 🔴 Projet dédié |

### Légende des couleurs

| Indicateur | Signification |
|------------|---------------|
| 🟢 Excellente/Très bonne | Rétrocompatibilité quasi totale, migration simple |
| 🟡 Bonne/Acceptable | Quelques ajustements nécessaires, tests requis |
| 🟠 Acceptable | Changements significatifs, validation approfondie |
| 🔴 Difficile/Complexe | Refonte possible, projet dédié recommandé |

---

## 🔄 Changements de Comportement par Défaut

### 1. Charset : utf8mb4 devient le défaut 🌐

**Impact** : 🔴 Majeur

```sql
-- AVANT MariaDB 11.8 (10.x, 11.0-11.7)
CREATE DATABASE myapp;
-- Charset: latin1 (défaut historique)

-- APRÈS MariaDB 11.8
CREATE DATABASE myapp;
-- Charset: utf8mb4 (nouveau défaut)
-- Collation: utf8mb4_unicode_ci
```

#### Implications techniques

| Aspect | Avant (latin1) | Après (utf8mb4) | Impact |
|--------|----------------|-----------------|--------|
| **Stockage ASCII** | 1 byte/char | 1 byte/char | Aucun |
| **Stockage Latin étendu** | 1 byte/char | 2 bytes/char | +100% |
| **Support emoji** | ❌ | ✅ | Nouvelle capacité |
| **Support multilingue** | Limité | Complet | Amélioration |
| **Taille index** | Plus petit | Plus grand | +33% en moyenne |
| **Performance** | Légèrement plus rapide | Légèrement plus lent | -5 à -10% |

#### Stratégies de migration

**Option 1 : Adopter utf8mb4 (recommandé)**

```sql
-- Nouvelles bases en utf8mb4 (automatique)
CREATE DATABASE new_app;

-- Migration bases existantes
ALTER DATABASE old_app 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- Migration tables
ALTER TABLE users 
CONVERT TO CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- ⚠️ Attention : Peut prendre du temps sur grandes tables
-- Utiliser pt-online-schema-change pour production
```

**Option 2 : Conserver latin1 (legacy)**

```sql
-- Forcer latin1 explicitement
CREATE DATABASE legacy_app 
CHARACTER SET latin1 
COLLATE latin1_swedish_ci;

-- Configuration globale (my.cnf)
[mysqld]
character-set-server = latin1
collation-server = latin1_swedish_ci
```

#### Tests de validation

```sql
-- Vérifier charset actuel
SELECT 
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema');

-- Vérifier tables
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_COLLATION
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'myapp';

-- Test comparaisons sensibles
SELECT * FROM users WHERE name = 'José';  -- Doit trouver "Jose", "JOSÉ", etc.
SELECT * FROM users WHERE email = 'user@example.com';  -- Case-sensitive recommandé
```

💡 **Recommandation** : Adopter utf8mb4 sauf si application ASCII pure et contraintes de performance critiques.

---

### 2. Collations : UCA 14.0.0 🔤

**Impact** : 🟡 Modéré

```sql
-- Nouvelles collations UCA 14.0.0
utf8mb4_0900_ai_ci    -- Accent/case insensitive (AI/CI)
utf8mb4_0900_as_ci    -- Accent sensitive, case insensitive
utf8mb4_0900_as_cs    -- Accent/case sensitive
utf8mb4_unicode_ci    -- Collation standard (UCA 9.0 compatible)
```

#### Changements d'ordre de tri

Certaines langues ont un ordre de tri modifié :

```sql
-- Test ordre de tri
CREATE TABLE test_collation (
    id INT PRIMARY KEY AUTO_INCREMENT,
    word VARCHAR(50) COLLATE utf8mb4_0900_ai_ci
);

INSERT INTO test_collation (word) VALUES
('apple'), ('Äpfel'), ('zebra'), ('çava'), ('cafe'), ('café');

-- Ancien comportement (UCA 9.0)
-- apple, Äpfel, cafe, café, çava, zebra

-- Nouveau comportement (UCA 14.0.0)
-- Äpfel, apple, cafe, café, çava, zebra
-- (meilleur respect des règles linguistiques)

SELECT word FROM test_collation ORDER BY word;
```

#### Actions requises

```sql
-- Auditer les ORDER BY sensibles
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    COLUMN_NAME,
    COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND DATA_TYPE IN ('varchar', 'char', 'text')
  AND COLLATION_NAME LIKE '%unicode%';

-- Tests de non-régression sur :
-- - Tri alphabétique affiché à l'utilisateur
-- - Recherches de plages (BETWEEN 'A' AND 'Z')
-- - Comparaisons de chaînes dans la logique métier
```

⚠️ **Attention** : Tester particulièrement si application multilingue (français, espagnol, allemand, etc.).

---

### 3. TIMESTAMP : Extension 2038→2106 ⏰

**Impact** : 🟡 Modéré à Élevé (selon usage de System-Versioned Tables)

```sql
-- Ancien format (jusqu'à 11.7)
-- TIMESTAMP : 32-bit signed, limite 2038-01-19 03:14:07

-- Nouveau format (11.8+)
-- TIMESTAMP : Extended format, limite 2106-02-07 06:28:15
```

#### Impact sur les tables existantes

| Type de table | Impact | Action requise |
|---------------|--------|----------------|
| **Tables normales** | ✅ Aucun | Aucune, transparent |
| **System-Versioned Tables** | 🔴 Majeur | Migration manuelle requise |
| **Application Time Periods** | 🟢 Minimal | Tests recommandés |

#### Migration System-Versioned Tables

```sql
-- AVANT migration : Sauvegarder données historiques
CREATE TABLE users_history_backup AS
SELECT * FROM users FOR SYSTEM_TIME ALL;

-- Désactiver versioning
ALTER TABLE users DROP SYSTEM VERSIONING;

-- Recréer avec nouveau format
DROP TABLE users;
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50),
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) WITH SYSTEM VERSIONING;

-- Réimporter données
INSERT INTO users (id, username, email, created_at, updated_at)
SELECT id, username, email, created_at, updated_at
FROM users_backup;

-- ⚠️ Historique perdu, voir section 19.9 pour migration complète
```

💡 **Bonne pratique** : Planifier la migration des System-Versioned Tables en dehors des heures de pointe.

---

### 4. TLS : Activé par défaut 🔒

**Impact** : 🟡 Modéré

```sql
-- AVANT 11.8
-- TLS désactivé par défaut, activation manuelle requise

-- APRÈS 11.8
-- TLS activé automatiquement si certificats présents
-- Localisation par défaut :
-- /etc/mysql/ssl/server-cert.pem
-- /etc/mysql/ssl/server-key.pem
-- /etc/mysql/ssl/ca-cert.pem
```

#### Impact sur les clients

| Type de client | Impact | Action |
|----------------|--------|--------|
| **Clients TLS-ready** | ✅ Aucun | Connexion TLS automatique |
| **Clients legacy** | 🟠 Connexion refusée | Mise à jour ou config |
| **Scripts automatisés** | 🟡 Vérifier --ssl-mode | Ajuster paramètres |

#### Configuration de compatibilité

```ini
# my.cnf - Forcer TLS pour tous (recommandé production)
[mysqld]
require_secure_transport = ON

# Ou permettre connexions non-TLS (dev uniquement)
[mysqld]
require_secure_transport = OFF
```

```bash
# Connexion client avec TLS explicite
mysql -h server.example.com -u user -p \
  --ssl-mode=REQUIRED \
  --ssl-ca=/path/to/ca-cert.pem

# Connexion client sans TLS (si autorisé serveur)
mysql -h localhost -u user -p --ssl-mode=DISABLED
```

#### Validation

```sql
-- Vérifier statut TLS de la connexion
SHOW STATUS LIKE 'Ssl_cipher';
-- Non vide = TLS actif

-- Lister utilisateurs avec/sans TLS
SELECT 
    user,
    host,
    ssl_type,
    ssl_cipher
FROM mysql.user;
```

---

### 5. Autres changements mineurs

#### innodb_alter_copy_bulk activable

```sql
-- Nouveau paramètre (11.8)
SET GLOBAL innodb_alter_copy_bulk = ON;  -- Défaut: OFF

-- Impact : ALTER TABLE 2-3x plus rapide
-- Aucune action requise, activation optionnelle
```

#### Privilèges granulaires étendus

```sql
-- Nouveaux privilèges disponibles
GRANT SELECT (column1, column2) ON mydb.users TO 'app'@'%';
GRANT INSERT, UPDATE ON mydb.orders TO 'app'@'%';
-- Plus fin que les versions précédentes
```

---

## 🛠️ Stratégies de Migration

### Stratégie 1 : In-Place Upgrade (Simple) 🔧

**Recommandé pour** : MariaDB 11.4+ → 11.8

```bash
# 1. Backup complet
mariabackup --backup --target-dir=/backup/full-backup

# 2. Arrêt serveur
systemctl stop mariadb

# 3. Mise à jour packages
# Debian/Ubuntu
apt update
apt install mariadb-server-11.8

# RHEL/CentOS
yum update mariadb-server

# 4. Exécution upgrade
mariadb-upgrade -u root -p

# 5. Redémarrage
systemctl start mariadb

# 6. Vérification version
mysql -V
# mariadb  Ver 15.1 Distrib 11.8.0-MariaDB
```

**Durée** : 30 minutes à 2 heures (selon taille base)

**Risques** : 🟢 Faibles
- Downtime pendant upgrade (15-60 min)
- Rollback possible via backup

---

### Stratégie 2 : Logical Migration (Dump/Restore) 📦

**Recommandé pour** : MariaDB 10.x → 11.8, MySQL → MariaDB

```bash
# 1. Dump depuis source
mariadb-dump -u root -p \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  > full-dump-$(date +%Y%m%d).sql

# 2. Installation MariaDB 11.8 (serveur neuf)
# ... (voir installation)

# 3. Restauration
mysql -u root -p < full-dump-20251214.sql

# 4. Upgrade système
mariadb-upgrade -u root -p

# 5. Validation
mysql -u root -p -e "SELECT VERSION();"
```

**Avantages** :
- ✅ Nettoyage automatique (fragmentation éliminée)
- ✅ Conversion charset/collation possible
- ✅ Serveur source reste opérationnel
- ✅ Rollback facile (garder source)

**Inconvénients** :
- ⏱️ Durée proportionnelle à la taille des données
- 💾 Espace disque doublé temporairement
- ⚠️ Downtime pendant switch

---

### Stratégie 3 : Réplication avec Basculement (Zero-Downtime) 🔄

**Recommandé pour** : Production critique, MariaDB 10.11+ → 11.8

```
┌─────────────────────────────────────────────────────┐
│       Migration Zero-Downtime via Réplication       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Phase 1: Setup réplication                         │
│  ┌──────────────┐                                   │
│  │ Primary 10.11│──┐                                │
│  │ (Production) │  │ Binlog                         │
│  └──────────────┘  │                                │
│                    │                                │
│                    ▼                                │
│               ┌──────────────┐                      │
│               │ Replica 11.8 │                      │
│               │ (En synchro) │                      │
│               └──────────────┘                      │
│                                                     │
│  Phase 2: Tests validation (Replica)                │
│  - Performance                                      │
│  - Compatibilité application                        │
│  - Tests de charge                                  │
│                                                     │
│  Phase 3: Basculement (Switchover)                  │
│  ┌──────────────┐    Stop writes                    │
│  │ Primary 10.11│────────────────X                  │
│  └──────────────┘                                   │
│                                                     │
│  ┌──────────────┐    Promote                        │
│  │ Primary 11.8 │◄───────────────✓                  │
│  │ (Production) │                                   │
│  └──────────────┘                                   │
│                                                     │
│  Phase 4: Validation post-switch                    │
│  - Monitoring                                       │
│  - Tests fonctionnels                               │
│  - Rollback possible si problème                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### Procédure détaillée

```bash
# 1. Setup Replica 11.8
# Sur Primary 10.11
mysql -u root -p
CREATE USER 'repl'@'%' IDENTIFIED BY 'secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

# Obtenir position binlog
SHOW MASTER STATUS;
# File: mysql-bin.000042
# Position: 154789

# 2. Sur Replica 11.8
mysql -u root -p
CHANGE MASTER TO
  MASTER_HOST='primary-server.example.com',
  MASTER_USER='repl',
  MASTER_PASSWORD='secure_password',
  MASTER_LOG_FILE='mysql-bin.000042',
  MASTER_LOG_POS=154789;

START SLAVE;
SHOW SLAVE STATUS\G
# Seconds_Behind_Master: 0 (en synchro)

# 3. Tests sur Replica (Read-Only)
# - Tests applicatifs
# - Benchmarks performance
# - Validation charset/collation

# 4. Basculement (fenêtre de maintenance)
# Sur Primary 10.11
SET GLOBAL read_only = ON;
FLUSH TABLES WITH READ LOCK;

# Attendre sync complète Replica
# Sur Replica 11.8
SHOW SLAVE STATUS\G
# Seconds_Behind_Master: 0

# Promouvoir Replica
STOP SLAVE;
RESET SLAVE ALL;
SET GLOBAL read_only = OFF;

# 5. Rediriger application vers nouveau Primary 11.8

# 6. Rollback si problème
# Réactiver Primary 10.11
SET GLOBAL read_only = OFF;
# Rediriger application
```

**Durée** : 10-30 minutes de downtime (phase basculement uniquement)

---

### Stratégie 4 : Blue-Green Deployment 🟦🟩

**Recommandé pour** : Architectures cloud-native, Kubernetes

```yaml
# Kubernetes: Déploiement Blue-Green
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-green  # Nouvelle version 11.8
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: mariadb
        image: mariadb:11.8
        # ... config ...

---
# Service pointant initialement vers Blue (10.11)
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
spec:
  selector:
    version: blue  # mariadb-blue (10.11)
  ports:
  - port: 3306

# Phase de test Green
# ... tests, validation ...

# Switch vers Green (11.8)
# Mise à jour selector:
#   version: green
```

---

## ✅ Checklist de Migration

### Phase 1 : Préparation (1-2 semaines avant)

- [ ] **Backup complet** (full + binlog)
- [ ] **Audit version actuelle**
  ```sql
  SELECT VERSION();
  SHOW VARIABLES LIKE 'version%';
  ```
- [ ] **Inventaire bases et tables**
  ```sql
  SELECT 
      TABLE_SCHEMA,
      COUNT(*) as table_count,
      SUM(DATA_LENGTH + INDEX_LENGTH)/1024/1024 as size_mb
  FROM INFORMATION_SCHEMA.TABLES
  GROUP BY TABLE_SCHEMA;
  ```
- [ ] **Identification features 11.8 utiles**
- [ ] **Revue documentation release notes**
- [ ] **Setup environnement de test**
- [ ] **Plan de rollback défini**
- [ ] **Communication équipes**

### Phase 2 : Tests (1-2 semaines)

- [ ] **Installation 11.8 sur environnement test**
- [ ] **Restauration dump production**
- [ ] **Tests compatibilité charset/collation**
  ```sql
  -- Vérifier ORDER BY
  SELECT * FROM users ORDER BY lastname, firstname;
  
  -- Vérifier comparaisons
  SELECT * FROM products WHERE name = 'café';
  ```
- [ ] **Tests applicatifs automatisés**
- [ ] **Tests manuels parcours critiques**
- [ ] **Benchmarks performance**
  ```bash
  sysbench oltp_read_write \
    --mysql-host=localhost \
    --mysql-user=root \
    --mysql-password=test \
    --mysql-db=testdb \
    --tables=10 \
    --table-size=100000 \
    run
  ```
- [ ] **Validation TLS**
- [ ] **Tests procédures/fonctions/triggers**
- [ ] **Validation réplication (si applicable)**

### Phase 3 : Migration Production (jour J)

- [ ] **Fenêtre de maintenance communiquée**
- [ ] **Backup de dernière minute**
  ```bash
  mariabackup --backup --target-dir=/backup/pre-upgrade-$(date +%Y%m%d-%H%M)
  ```
- [ ] **Mode maintenance application**
- [ ] **Arrêt serveur**
  ```bash
  systemctl stop mariadb
  ```
- [ ] **Upgrade packages**
- [ ] **Exécution mariadb-upgrade**
  ```bash
  mariadb-upgrade -u root -p --verbose
  ```
- [ ] **Démarrage serveur**
- [ ] **Vérification logs erreurs**
  ```bash
  tail -f /var/log/mysql/error.log
  ```
- [ ] **Tests smoke (connexion, requêtes simples)**
- [ ] **Sortie mode maintenance**
- [ ] **Monitoring intensif (2-4h)**

### Phase 4 : Post-Migration (1 semaine)

- [ ] **Monitoring performance**
- [ ] **Analyse slow query log**
  ```bash
  pt-query-digest /var/log/mysql/slow.log
  ```
- [ ] **Validation métriques business**
- [ ] **Optimisation si nécessaire**
  ```sql
  ANALYZE TABLE users;
  OPTIMIZE TABLE large_table;
  ```
- [ ] **Documentation mises à jour**
- [ ] **Retour d'expérience équipe**
- [ ] **Planification features 11.8** (Vector, etc.)
- [ ] **Archivage backups pré-migration** (conserver 30j min)

---

## 🚨 Gestion des Risques

### Identification des risques

| Risque | Probabilité | Impact | Mitigation |
|--------|-------------|--------|------------|
| **Downtime prolongé** | 🟡 Moyenne | 🔴 Élevé | Réplication, Blue-Green |
| **Incompatibilité app** | 🟢 Faible | 🟠 Moyen | Tests approfondis, POC |
| **Perte de performance** | 🟢 Faible | 🟡 Moyen | Benchmarks, tuning |
| **Corruption données** | 🟢 Très faible | 🔴 Critique | Backups multiples, validation |
| **Rollback impossible** | 🟡 Moyenne | 🔴 Élevé | Plan rollback testé |
| **Charset/collation** | 🟡 Moyenne | 🟠 Moyen | Tests comparaisons, ORDER BY |
| **TLS breaking clients** | 🟡 Moyenne | 🟡 Moyen | Inventaire clients, tests |

### Plan de rollback

```bash
# Scénario 1 : Problème détecté dans l'heure suivant migration

# 1. Arrêt MariaDB 11.8
systemctl stop mariadb

# 2. Restauration backup
# Si backup Mariabackup
mariabackup --copy-back --target-dir=/backup/pre-upgrade-20251214

# Si backup logique
mysql -u root -p < pre-upgrade-20251214.sql

# 3. Downgrade packages (si in-place upgrade)
apt install mariadb-server=10.11.x  # Debian/Ubuntu
yum downgrade mariadb-server-10.11  # RHEL/CentOS

# 4. Redémarrage
systemctl start mariadb

# 5. Validation
mysql -e "SELECT VERSION();"

# Durée rollback : 30-90 minutes
```

```bash
# Scénario 2 : Problème détecté après plusieurs heures/jours

# Si réplication active
# 1. Basculer vers Replica restée en 10.11
# 2. Analyser root cause
# 3. Planifier nouvelle tentative migration

# Si pas de réplication
# ⚠️ Rollback complexe, données perdues potentiellement
# → Importance de conserver Primary 10.11 plusieurs jours
```

---

## 🧪 Tests de Compatibilité

### Suite de tests recommandée

```sql
-- 1. Test charset/collation
CREATE DATABASE test_charset 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

USE test_charset;

CREATE TABLE test_collation (
    id INT PRIMARY KEY AUTO_INCREMENT,
    word VARCHAR(50)
);

INSERT INTO test_collation (word) VALUES
('élève'), ('élève'), ('ÉLÈVE'), ('eleve'),
('café'), ('Café'), ('CAFÉ'), ('cafe');

-- Vérifier unicité et tri
SELECT DISTINCT word FROM test_collation ORDER BY word;

-- 2. Test TIMESTAMP
CREATE TABLE test_timestamp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    event_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_timestamp (event_date) VALUES
('2025-01-01 12:00:00'),
('2038-01-19 03:14:07'),  -- Limite ancienne
('2106-02-07 06:28:15');  -- Nouvelle limite

SELECT * FROM test_timestamp ORDER BY event_date;

-- 3. Test TLS
SHOW STATUS LIKE 'Ssl%';
-- Ssl_cipher doit être non vide

-- 4. Test procédures stockées
DELIMITER //
CREATE PROCEDURE test_proc()
BEGIN
    SELECT 'Procédure OK' AS status;
END //
DELIMITER ;

CALL test_proc();
DROP PROCEDURE test_proc;

-- 5. Test triggers
CREATE TABLE test_trigger (
    id INT PRIMARY KEY AUTO_INCREMENT,
    value VARCHAR(50),
    modified_at TIMESTAMP
);

CREATE TRIGGER test_trigger_update
BEFORE UPDATE ON test_trigger
FOR EACH ROW
SET NEW.modified_at = CURRENT_TIMESTAMP;

INSERT INTO test_trigger (value) VALUES ('initial');
UPDATE test_trigger SET value = 'updated' WHERE id = 1;
SELECT * FROM test_trigger;  -- modified_at doit être mis à jour

-- 6. Test performances index
CREATE TABLE test_perf (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data VARCHAR(255),
    INDEX idx_data (data)
);

-- Insérer 100k lignes
INSERT INTO test_perf (data)
SELECT MD5(RAND())
FROM information_schema.columns c1, information_schema.columns c2
LIMIT 100000;

-- Analyser plan d'exécution
EXPLAIN SELECT * FROM test_perf WHERE data LIKE 'a%';

-- Cleanup
DROP DATABASE test_charset;
```

---

## 📈 Retours d'Expérience

### Cas 1 : E-commerce (100k commandes/jour)

**Contexte** : MariaDB 10.11 LTS → 11.8

**Stratégie** : Réplication + Basculement

**Timeline** :
- Semaine 1-2 : Setup replica 11.8, tests
- Semaine 3 : Benchmarks, validation perf
- Semaine 4 : Basculement (dimanche 3h-4h)

**Résultats** :
- ✅ Downtime : 12 minutes (au lieu de 45 min prévues)
- ✅ Performance : +8% (optimizer SSD-aware)
- ⚠️ 3 requêtes à ajuster (ORDER BY collation)
- 💰 Économies : -15% coûts infra (utf8mb4 plus efficace que pensé)

---

### Cas 2 : SaaS B2B (Multi-tenant)

**Contexte** : MariaDB 11.4 LTS → 11.8

**Stratégie** : In-place upgrade par étapes

**Timeline** :
- Jour 1 : Tenants dev/staging (100 bases)
- Jour 7 : Tenants beta (500 bases)
- Jour 14 : Tous production (5000 bases)

**Résultats** :
- ✅ Aucun incident majeur
- ✅ Adoption MariaDB Vector pour search (+30% satisfaction)
- 📊 Downtime moyen : 8 min/tenant
- 🔧 2 bugs mineurs identifiés et corrigés

---

### Cas 3 : Migration MySQL 8.0 → MariaDB 11.8

**Contexte** : Startup, volonté réduire coûts licensing

**Stratégie** : Logical migration + Blue-Green

**Timeline** :
- Mois 1 : POC, identification incompatibilités
- Mois 2 : Migration staging, tests
- Mois 3 : Migration production

**Résultats** :
- ✅ Coûts : -40% (vs MySQL Enterprise)
- ⚠️ 12 requêtes à réécrire (divergences MySQL/MariaDB)
- ✅ Performance : équivalente
- 🎁 Bonus : MariaDB Vector pour nouvelle feature IA

**Incompatibilités rencontrées** :
```sql
-- MySQL 8.0 feature non disponible MariaDB
-- LATERAL JOIN → Réécriture avec CTE

-- MySQL
SELECT *
FROM orders o,
LATERAL (
    SELECT * FROM order_items WHERE order_id = o.id LIMIT 3
) oi;

-- MariaDB 11.8 (alternative)
WITH ranked_items AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY id) as rn
    FROM order_items
)
SELECT o.*, ri.*
FROM orders o
JOIN ranked_items ri ON o.id = ri.order_id AND ri.rn <= 3;
```

---

## 💡 Recommandations Finales

### Par profil

#### Pour les Développeurs

1. **Tester applications en local** avec MariaDB 11.8 avant migration prod
2. **Valider charset utf8mb4** sur toutes les comparaisons de chaînes
3. **Profiter de MariaDB Vector** pour nouvelles features IA
4. **Utiliser Online DDL** pour modifications schéma sans downtime

#### Pour les DBA

1. **Planifier migration en heures creuses**
2. **Conserver ancienne version** 7-30 jours pour rollback
3. **Monitoring intensif** post-migration (3-7 jours)
4. **Documenter incidents** et partager retours d'expérience
5. **Activer innodb_alter_copy_bulk** pour maintenances futures

#### Pour les DevOps/SRE

1. **Automatiser tests compatibilité** dans CI/CD
2. **Implémenter Blue-Green** pour zero-downtime
3. **Monitoring Prometheus/Grafana** avec métriques 11.8
4. **Infrastructure as Code** pour reproductibilité
5. **Disaster Recovery Plan** testé régulièrement

---

## ✅ Points Clés à Retenir

- **Compatibilité globale** : 🟢 Excellente depuis MariaDB 11.4+, 🟡 Bonne depuis 10.11+
- **Changement majeur** : utf8mb4 devient charset par défaut (impact stockage +33%)
- **4 stratégies migration** : In-place, Logical, Réplication, Blue-Green
- **Tests essentiels** : Charset/collation, TIMESTAMP, TLS, performances
- **Plan de rollback** : Obligatoire, testé avant migration
- **Timeline type** : 1-2 semaines préparation + tests, 30-90 min migration
- **Risques maîtrisables** : Backups multiples, tests approfondis, rollback préparé
- **ROI positif** : Nouvelles fonctionnalités (Vector) compensent effort migration
- **Support LTS 3 ans** : Migration vers 11.8 sécurise jusqu'en 2028
- **Documentation complète** : Section 19 pour procédures détaillées

---

## 🔗 Ressources Complémentaires

### Documentation officielle

- 📖 [MariaDB 11.8 Upgrade Guide](https://mariadb.com/kb/en/upgrading-to-mariadb-118/)
- 📖 [mariadb-upgrade Documentation](https://mariadb.com/kb/en/mariadb-upgrade/)
- 📖 [Upgrading Between Major Versions](https://mariadb.com/kb/en/upgrading-between-major-mariadb-versions/)

### Sections détaillées de la formation

- **Section 19** - Migration et Compatibilité (guide complet)
- **Section 19.9** - Migration System-Versioned Tables
- **Section 11.11** - Charset utf8mb4 et UCA 14.0.0
- **Section 11.12** - Extension TIMESTAMP 2106
- **Section 10.7.3** - TLS par défaut

### Outils

- 🔧 [mariadb-upgrade](https://mariadb.com/kb/en/mariadb-upgrade/)
- 🔧 [Mariabackup](https://mariadb.com/kb/en/mariabackup/)
- 🔧 [pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html)
- 🔧 [sysbench](https://github.com/akopytov/sysbench)

---

## ➡️ Section Suivante

**F.4** [Recommandations d'adoption](./04-recommandations-adoption.md) - Décisions stratégiques et planification

---

**MariaDB** : Version 11.8 LTS (Juin 2025)

⏭️ [Recommandations d'adoption](/annexes/nouveautes-11-8/04-recommandations-adoption.md)

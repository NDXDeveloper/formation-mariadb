🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.3 Système de privilèges

> **Niveau** : Avancé
> **Durée estimée** : 2 heures
> **Prérequis** : Sections 10.1 (Modèle de sécurité) et 10.2 (Gestion des utilisateurs)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture du système de privilèges MariaDB
- **Identifier** les différents niveaux de privilèges (global, base, table, colonne)
- **Maîtriser** la hiérarchie et l'ordre d'évaluation des privilèges
- **Distinguer** les types de privilèges (DML, DDL, administration)
- **Appliquer** le principe du moindre privilège dans des contextes complexes
- **Utiliser** les nouveaux privilèges granulaires de MariaDB 11.8 🆕
- **Diagnostiquer** les problèmes de permissions
- **Auditer** les privilèges existants dans un système en production

---

## Introduction

Le système de privilèges de MariaDB est l'un des plus granulaires et flexibles parmi les SGBD relationnels. Il permet de contrôler précisément **qui peut faire quoi, où et comment** à travers une architecture multi-niveaux sophistiquée.

### Philosophie du système de privilèges

Le système de privilèges MariaDB repose sur trois principes fondamentaux :

```
┌─────────────────────────────────────────────────────────────┐
│  1. PRINCIPE DU MOINDRE PRIVILÈGE                           │
│     → Accorder uniquement les privilèges strictement        │
│       nécessaires pour accomplir une tâche                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  2. GRANULARITÉ MAXIMALE                                    │
│     → Du niveau global (tous les serveurs) au niveau        │
│       colonne individuelle                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  3. ÉVALUATION HIÉRARCHIQUE                                 │
│     → Les privilèges sont évalués du plus général au plus   │
│       spécifique, s'arrêtant au premier match               │
└─────────────────────────────────────────────────────────────┘
```

### Pourquoi un système de privilèges complexe ?

Un système de privilèges granulaire est essentiel pour :

1. **Sécurité** : Limiter les dégâts en cas de compromission
2. **Conformité** : Respecter les réglementations (RGPD, SOC2, PCI-DSS)
3. **Séparation des rôles** : Développeurs ≠ DBA ≠ Auditeurs ≠ Applications
4. **Auditabilité** : Tracer précisément qui a accès à quoi
5. **Multi-tenancy** : Isoler les données entre clients/départements

**Exemple concret** :

```
Application e-commerce
├── Frontend (lecture seule produits/commandes)
│   └── Privilèges: SELECT sur tables produits, commandes
├── Backend API (CRUD complet)
│   └── Privilèges: SELECT, INSERT, UPDATE sur tables métier
├── Service de paiement (accès limité)
│   └── Privilèges: UPDATE(status) sur table commandes uniquement
├── Analytics (lecture seule, toutes tables)
│   └── Privilèges: SELECT global sur base analytics
└── DBA (administration complète)
    └── Privilèges: ALL PRIVILEGES sur tous les serveurs
```

---

## Architecture du système de privilèges

### Hiérarchie des niveaux de privilèges

MariaDB évalue les privilèges selon **5 niveaux hiérarchiques**, du plus général au plus spécifique :

```
NIVEAU 1 : PRIVILÈGES GLOBAUX (mysql.user / mysql.global_priv)
           Portée: Tout le serveur MariaDB (*.*)
           │
           ├─→ Exemple: GRANT SELECT ON *.* TO user;
           │   (Lecture sur toutes les bases de données)
           │
           └─→ Stockage: mysql.global_priv (JSON)
                         mysql.user (vue de compatibilité)

NIVEAU 2 : PRIVILÈGES BASE DE DONNÉES (mysql.db)
           Portée: Une base de données spécifique (db.*)
           │
           ├─→ Exemple: GRANT INSERT ON production.* TO user;
           │   (Insertion dans toutes les tables de 'production')
           │
           └─→ Stockage: mysql.db

NIVEAU 3 : PRIVILÈGES TABLE (mysql.tables_priv)
           Portée: Une table spécifique (db.table)
           │
           ├─→ Exemple: GRANT UPDATE ON production.orders TO user;
           │   (Modification de la table 'orders' uniquement)
           │
           └─→ Stockage: mysql.tables_priv

NIVEAU 4 : PRIVILÈGES COLONNE (mysql.columns_priv)
           Portée: Colonnes spécifiques d'une table (db.table.column)
           │
           ├─→ Exemple: GRANT SELECT(email) ON users TO user;
           │   (Lecture de la colonne 'email' uniquement)
           │
           └─→ Stockage: mysql.columns_priv

NIVEAU 5 : PRIVILÈGES ROUTINES (mysql.procs_priv)
           Portée: Procédures stockées et fonctions
           │
           ├─→ Exemple: GRANT EXECUTE ON PROCEDURE calc_total TO user;
           │   (Exécution d'une procédure spécifique)
           │
           └─→ Stockage: mysql.procs_priv
```

### Ordre d'évaluation des privilèges

**Règle fondamentale** : MariaDB évalue les privilèges dans l'ordre hiérarchique et **s'arrête au premier niveau qui accorde le privilège**.

```
Requête: SELECT * FROM production.orders;
         │
         ├─→ ÉTAPE 1: Vérifier mysql.global_priv
         │            L'utilisateur a-t-il SELECT global (*.*) ?
         │            ├─→ OUI : Requête autorisée ✓
         │            └─→ NON : Passer à l'étape 2
         │
         ├─→ ÉTAPE 2: Vérifier mysql.db
         │            L'utilisateur a-t-il SELECT sur production.* ?
         │            ├─→ OUI : Requête autorisée ✓
         │            └─→ NON : Passer à l'étape 3
         │
         ├─→ ÉTAPE 3: Vérifier mysql.tables_priv
         │            L'utilisateur a-t-il SELECT sur production.orders ?
         │            ├─→ OUI : Requête autorisée ✓
         │            └─→ NON : Passer à l'étape 4
         │
         ├─→ ÉTAPE 4: Vérifier mysql.columns_priv
         │            L'utilisateur a-t-il SELECT sur certaines colonnes ?
         │            ├─→ OUI : Requête partielle (colonnes autorisées) ⚠️
         │            └─→ NON : Passer à l'étape 5
         │
         └─→ ÉTAPE 5: Aucun privilège trouvé
                      Requête refusée ✗ (ERROR 1142)
```

💡 **Optimisation** : Les privilèges plus généraux (globaux, DB) sont plus performants car l'évaluation s'arrête plus tôt.

### Exemple pratique d'évaluation

```sql
-- Configuration utilisateur
CREATE USER 'analyst'@'%' IDENTIFIED BY 'password';

-- Privilèges configurés
GRANT SELECT ON *.* TO 'analyst'@'%';                    -- Niveau 1: Global
GRANT INSERT, UPDATE ON production.* TO 'analyst'@'%';   -- Niveau 2: DB
GRANT DELETE ON production.orders TO 'analyst'@'%';      -- Niveau 3: Table

-- Test des requêtes
-- Requête 1:
SELECT * FROM production.orders;
-- Évaluation: Trouve SELECT global (Niveau 1) → Autorisé ✓

-- Requête 2:
INSERT INTO production.orders VALUES (...);
-- Évaluation: Pas d'INSERT global → Trouve INSERT sur production.* (Niveau 2) → Autorisé ✓

-- Requête 3:
DELETE FROM production.orders WHERE id = 1;
-- Évaluation: Pas de DELETE global, ni sur production.* → Trouve DELETE sur production.orders (Niveau 3) → Autorisé ✓

-- Requête 4:
DELETE FROM production.customers WHERE id = 1;
-- Évaluation: Pas de DELETE global, ni sur production.*, ni sur production.customers → Refusé ✗
-- ERROR 1142 (42000): DELETE command denied to user 'analyst'@'%' for table 'customers'
```

---

## Types de privilèges

MariaDB propose **plus de 40 privilèges différents**, regroupés en plusieurs catégories.

### 1. Privilèges de données (DML - Data Manipulation Language)

Contrôlent l'accès aux données contenues dans les tables.

| Privilège | Portée | Description | Exemple |
|-----------|--------|-------------|---------|
| **SELECT** | Global, DB, Table, Colonne | Lire les données | `SELECT * FROM users` |
| **INSERT** | Global, DB, Table, Colonne | Insérer des données | `INSERT INTO users VALUES (...)` |
| **UPDATE** | Global, DB, Table, Colonne | Modifier des données | `UPDATE users SET name = 'John'` |
| **DELETE** | Global, DB, Table | Supprimer des données | `DELETE FROM users WHERE ...` |

**Cas d'usage typique** :

```sql
-- Application web CRUD classique
GRANT SELECT, INSERT, UPDATE, DELETE ON webapp.* TO 'webapp_user'@'app_server';

-- Service en lecture seule (analytics, reporting)
GRANT SELECT ON warehouse.* TO 'analytics_user'@'%';

-- Service d'insertion de logs (write-only)
GRANT INSERT ON logs.* TO 'logger_service'@'log_collector';
```

### 2. Privilèges de structure (DDL - Data Definition Language)

Contrôlent les modifications de la structure de la base de données.

| Privilège | Portée | Description | Exemple |
|-----------|--------|-------------|---------|
| **CREATE** | Global, DB | Créer bases/tables | `CREATE TABLE users (...)` |
| **ALTER** | Global, DB, Table | Modifier structure | `ALTER TABLE users ADD COLUMN ...` |
| **DROP** | Global, DB | Supprimer objets | `DROP TABLE users` |
| **INDEX** | Global, DB, Table | Créer/supprimer index | `CREATE INDEX idx_name ON users(name)` |
| **CREATE TEMPORARY TABLES** | Global, DB | Créer tables temporaires | `CREATE TEMPORARY TABLE tmp AS ...` |
| **CREATE VIEW** | Global, DB | Créer vues | `CREATE VIEW v_users AS SELECT ...` |
| **SHOW VIEW** | Global, DB, Table | Voir définition vues | `SHOW CREATE VIEW v_users` |
| **CREATE ROUTINE** | Global, DB | Créer procédures/fonctions | `CREATE PROCEDURE proc() ...` |
| **ALTER ROUTINE** | Global, DB, Routine | Modifier routines | `ALTER PROCEDURE proc ...` |
| **EXECUTE** | Global, DB, Routine | Exécuter routines | `CALL proc()` |
| **EVENT** | Global, DB | Créer/modifier événements | `CREATE EVENT evt ...` |
| **TRIGGER** | Global, DB, Table | Créer/supprimer triggers | `CREATE TRIGGER trg ...` |

**Cas d'usage typique** :

```sql
-- DBA avec privilèges DDL complets
GRANT CREATE, ALTER, DROP, INDEX ON production.* TO 'dba'@'localhost';

-- Développeur avec DDL limité (pas de DROP en production)
GRANT CREATE, ALTER, INDEX ON dev_database.* TO 'developer'@'%';
-- Pas de DROP pour éviter suppressions accidentelles

-- Service applicatif (zéro DDL)
GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO 'app_user'@'%';
-- Applications ne doivent JAMAIS modifier la structure
```

⚠️ **Attention production** : Les privilèges DDL (`ALTER`, `DROP`) doivent être **strictement limités** en production pour éviter :
- Suppressions accidentelles de tables
- Modifications de schéma non contrôlées
- Indisponibilités dues à des ALTER TABLE bloquants

### 3. Privilèges d'administration

Privilèges pour les opérations d'administration du serveur.

| Privilège | Description | Cas d'usage |
|-----------|-------------|-------------|
| **RELOAD** | Recharger configs, caches | `FLUSH PRIVILEGES;`, `FLUSH TABLES;` |
| **SHUTDOWN** | Arrêter le serveur | Maintenance planifiée |
| **PROCESS** | Voir tous les processus | `SHOW PROCESSLIST;` monitoring |
| **FILE** | Lire/écrire fichiers serveur | `LOAD DATA INFILE`, `SELECT INTO OUTFILE` |
| **SUPER** | Opérations admin multiples | **⚠️ Très puissant** (voir section granularité) |
| **REPLICATION SLAVE** | Configurer réplication | Setup replica |
| **REPLICATION CLIENT** | Voir état réplication | `SHOW SLAVE STATUS;` monitoring |
| **SHOW DATABASES** | Lister toutes les bases | `SHOW DATABASES;` |
| **LOCK TABLES** | Verrouiller tables | Backups cohérents |
| **REFERENCES** | Créer clés étrangères | Intégrité référentielle |
| **CREATE USER** | Gérer utilisateurs | Administration sécurité |
| **GRANT OPTION** | Transmettre privilèges | Délégation |

**Cas d'usage typique** :

```sql
-- DBA complet
GRANT ALL PRIVILEGES ON *.* TO 'dba_admin'@'localhost' WITH GRANT OPTION;

-- DBA monitoring (lecture seule admin)
GRANT PROCESS, REPLICATION CLIENT, SHOW DATABASES ON *.* TO 'monitor'@'monitoring_server';

-- Agent de backup
GRANT SELECT, LOCK TABLES, RELOAD, REPLICATION CLIENT ON *.* TO 'backup_agent'@'localhost';
-- RELOAD pour FLUSH TABLES WITH READ LOCK
-- REPLICATION CLIENT pour binlog position

-- Service d'import de données
GRANT FILE, INSERT ON warehouse.* TO 'etl_service'@'etl_server';
```

### 4. 🆕 Privilèges granulaires (MariaDB 11.8)

MariaDB 11.8 introduit un découpage de `SUPER` en privilèges plus fins pour appliquer le principe du moindre privilège.

| Privilège 11.8 | Description | Remplace | Cas d'usage |
|----------------|-------------|----------|-------------|
| **BINLOG_ADMIN** | Administration binlogs | Partie de SUPER | DBA gérant binlogs |
| **BINLOG_REPLAY** | Rejouer binlogs (PITR) | Partie de SUPER | Point-in-time recovery |
| **BINLOG_MONITOR** | Lire binlogs | Partie de SUPER | Outils CDC (Debezium) |
| **CONNECTION_ADMIN** | Gérer connexions (KILL, etc.) | Partie de SUPER | Support N2, gestion incidents |
| **READ_ONLY_ADMIN** | Modifier en mode read-only | Partie de SUPER | Maintenance sur réplica |
| **REPLICATION_SLAVE_ADMIN** | Gérer réplication | Partie de SUPER | Setup/troubleshoot réplication |
| **SET_USER_ID** | Changer DEFINER | Partie de SUPER | Migration de routines |
| **SHOW_ROUTINE** 🆕 | Voir routines sans EXECUTE | Nouveau | Auditeurs, documentation |
| **SLAVE_MONITOR** | Monitoring réplication détaillé | Nouveau | Outils de monitoring avancés |

**Migration de SUPER vers privilèges granulaires** :

```sql
-- ❌ Ancien modèle (pré-11.8): SUPER trop permissif
GRANT SUPER ON *.* TO 'ops_user'@'localhost';
-- Problème: ops_user peut tout faire (dangereux)

-- ✅ Nouveau modèle (11.8+): Privilèges précis
-- Cas 1: DBA backup/restore
GRANT BINLOG_REPLAY ON *.* TO 'backup_dba'@'localhost';
-- Peut rejouer binlogs pour PITR, mais rien d'autre

-- Cas 2: Support technique
GRANT CONNECTION_ADMIN ON *.* TO 'support_user'@'%';
-- Peut tuer des connexions bloquantes, mais pas modifier binlogs

-- Cas 3: Outil de monitoring
GRANT BINLOG_MONITOR, SLAVE_MONITOR ON *.* TO 'monitoring_tool'@'monitoring_server';
-- Peut lire les binlogs et l'état de réplication, sans modifier

-- Cas 4: Auditeur de sécurité
GRANT SHOW_ROUTINE ON *.* TO 'security_auditor'@'%';
-- Peut voir les procédures stockées pour audit, sans les exécuter
```

**Vérification des nouveaux privilèges** :

```sql
-- Lister tous les privilèges disponibles (11.8+)
SHOW PRIVILEGES;
-- Rechercher: BINLOG_ADMIN, BINLOG_REPLAY, CONNECTION_ADMIN, SHOW_ROUTINE, etc.

-- Voir quels utilisateurs ont les nouveaux privilèges
SELECT User, Host,
  JSON_EXTRACT(Priv, '$.access') AS privileges_bitmask
FROM mysql.global_priv
WHERE JSON_EXTRACT(Priv, '$.access') IS NOT NULL;
```

### 5. Privilèges spéciaux

| Privilège | Description | Usage |
|-----------|-------------|-------|
| **ALL PRIVILEGES** | Tous les privilèges (sauf GRANT OPTION) | Raccourci DBA |
| **USAGE** | Connexion sans privilèges | Utilisateur sans droits (par défaut) |
| **PROXY** | Se connecter en tant qu'autre utilisateur | Délégation d'identité |

**Exemple USAGE** :

```sql
-- Créer un utilisateur qui peut se connecter mais rien faire
CREATE USER 'locked_user'@'%' IDENTIFIED BY 'password';
GRANT USAGE ON *.* TO 'locked_user'@'%';

-- L'utilisateur peut se connecter
mariadb -u locked_user -p
-- Mais ne peut rien faire
MariaDB> SELECT * FROM test.table;
-- ERROR 1142 (42000): SELECT command denied

-- USAGE est le privilège par défaut minimal
```

---

## Portées des privilèges

Les privilèges peuvent être accordés à différentes portées (scope).

### Niveau global (*.\*)

Privilèges sur **tout le serveur MariaDB**.

```sql
-- Format
GRANT privilege_list ON *.* TO 'user'@'host';

-- Exemples
GRANT SELECT ON *.* TO 'readonly_admin'@'localhost';
-- Peut lire toutes les bases de données

GRANT PROCESS ON *.* TO 'monitoring'@'%';
-- Peut voir tous les processus serveur

GRANT REPLICATION CLIENT ON *.* TO 'replication_monitor'@'%';
-- Peut surveiller l'état de réplication
```

**Stockage** : `mysql.global_priv` (table JSON moderne) ou `mysql.user` (vue compatible)

**Cas d'usage** :
- Administrateurs DBA
- Comptes de monitoring global
- Services de backup complet
- Outils de réplication

⚠️ **Attention** : Les privilèges globaux sont **très puissants**. À utiliser avec parcimonie.

### Niveau base de données (database.\*)

Privilèges sur **toutes les tables d'une base de données**.

```sql
-- Format
GRANT privilege_list ON database_name.* TO 'user'@'host';

-- Exemples
GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO 'app_user'@'app_server';
-- CRUD complet sur la base 'production'

GRANT CREATE, ALTER, DROP ON dev_db.* TO 'developer'@'%';
-- DDL complet sur la base de développement

GRANT SELECT ON analytics.* TO 'data_scientist'@'%';
-- Lecture seule sur la base analytics
```

**Stockage** : `mysql.db`

**Wildcards supportés** :

```sql
-- Toutes les bases commençant par 'test_'
GRANT ALL ON `test_%`.* TO 'dev_user'@'%';

-- Toutes les bases d'un tenant
GRANT SELECT, INSERT, UPDATE ON `tenant_123_%`.* TO 'tenant_123_app'@'%';
```

💡 **Note** : Les backticks (\`) sont obligatoires pour les patterns avec wildcards.

**Cas d'usage** :
- Applications avec accès à une base dédiée
- Multi-tenancy (une base par client)
- Développeurs avec sandbox

### Niveau table (database.table)

Privilèges sur **une table spécifique**.

```sql
-- Format
GRANT privilege_list ON database_name.table_name TO 'user'@'host';

-- Exemples
GRANT SELECT ON production.orders TO 'analyst'@'%';
-- Lecture uniquement de la table orders

GRANT UPDATE(status) ON production.orders TO 'shipping_service'@'%';
-- Modification de la colonne 'status' uniquement

GRANT DELETE ON production.logs TO 'cleanup_job'@'localhost';
-- Suppression de logs anciens
```

**Stockage** : `mysql.tables_priv`

**Cas d'usage** :
- Services avec accès limité à certaines tables
- Séparation des responsabilités (finance vs RH)
- Masquage de tables sensibles

### Niveau colonne (database.table.column)

Privilèges sur **des colonnes spécifiques**.

```sql
-- Format
GRANT privilege_list (column_list) ON database_name.table_name TO 'user'@'host';

-- Exemples
GRANT SELECT (id, name, email) ON production.users TO 'customer_service'@'%';
-- Lecture de 3 colonnes seulement (pas de SSN, salary, etc.)

GRANT UPDATE (last_login, login_count) ON production.users TO 'auth_service'@'%';
-- Modification de 2 colonnes de tracking uniquement

-- Combiner plusieurs privilèges
GRANT SELECT (id, name, email), UPDATE (email) ON production.users TO 'profile_service'@'%';
-- Lecture de 3 colonnes, modification de 'email' uniquement
```

**Stockage** : `mysql.columns_priv`

**Cas d'usage** :
- Protection de données sensibles (PII, PHI)
- Conformité RGPD (masquage de données personnelles)
- Séparation lecture/écriture par colonne

⚠️ **Impact performance** : Les privilèges par colonne sont **coûteux** en performance. Préférer les **vues** quand possible :

```sql
-- ❌ Moins performant: Privilèges par colonne
GRANT SELECT (id, name, email) ON users TO 'app_user';

-- ✅ Plus performant: Vue dédiée
CREATE VIEW users_safe AS SELECT id, name, email FROM users;
GRANT SELECT ON users_safe TO 'app_user';
```

### Niveau routine (procédure/fonction)

Privilèges sur **des procédures stockées ou fonctions spécifiques**.

```sql
-- Format
GRANT EXECUTE ON {PROCEDURE | FUNCTION} database_name.routine_name TO 'user'@'host';

-- Exemples
GRANT EXECUTE ON PROCEDURE production.calculate_shipping TO 'checkout_service'@'%';

GRANT EXECUTE ON FUNCTION analytics.revenue_by_month TO 'reporting_user'@'%';

-- Tous les routines d'une base
GRANT EXECUTE ON production.* TO 'app_user'@'%';
```

**Stockage** : `mysql.procs_priv`

**Cas d'usage** :
- Encapsulation de la logique métier
- Sécurité par obscurité (masquer les requêtes SQL)
- Contrôle d'accès fin sur opérations complexes

---

## Hiérarchie et précédence

### Règles de précédence

Lorsqu'un utilisateur a des privilèges à plusieurs niveaux, MariaDB applique ces règles :

1. **Privilège global = privilège partout**
   - Si `SELECT` global → Peut lire toutes les bases/tables

2. **Privilège DB s'applique à toutes les tables de cette DB**
   - Si `UPDATE` sur `production.*` → Peut modifier toutes les tables de production

3. **Privilège table remplace privilège DB pour cette table**
   - Non applicable (additive, pas de remplacement)

4. **Privilège colonne est additif avec privilège table**
   - Si `SELECT` sur table + `UPDATE(col)` → Peut lire toute la table, modifier une colonne

### Comportement additif (non restrictif)

**Important** : Les privilèges sont **additifs**, pas restrictifs. Impossible de "retirer" un privilège avec un niveau plus spécifique.

```sql
-- Configuration
GRANT SELECT ON production.* TO 'user'@'%';           -- Niveau DB
GRANT SELECT ON production.sensitive_table TO 'user'@'%';  -- Niveau table

-- ❌ IMPOSSIBLE: Retirer SELECT sur sensitive_table
-- Il faudrait REVOKE au niveau DB

-- L'utilisateur peut lire toutes les tables de production (privilège DB)
-- Le privilège table est redondant ici
```

**Cas problématique** :

```sql
-- Intention: Donner accès à tout sauf une table
GRANT SELECT ON production.* TO 'analyst'@'%';  -- Accès global à production

-- ❌ CECI NE FONCTIONNE PAS:
REVOKE SELECT ON production.sensitive_table FROM 'analyst'@'%';
-- ERROR: Impossible de révoquer un privilège table quand privilège DB existe

-- ✅ Solution: Privilèges par table explicites
REVOKE SELECT ON production.* FROM 'analyst'@'%';
GRANT SELECT ON production.table1 TO 'analyst'@'%';
GRANT SELECT ON production.table2 TO 'analyst'@'%';
-- ... (exclure sensitive_table)

-- ✅ Alternative: Vue sans la table sensible
CREATE VIEW production.analyst_view AS
  SELECT * FROM table1
  UNION ALL
  SELECT * FROM table2;
  -- ... (exclure sensitive_table)

GRANT SELECT ON production.analyst_view TO 'analyst'@'%';
```

### Exemple de hiérarchie complexe

```sql
-- Scénario: Utilisateur multi-niveaux
CREATE USER 'complex_user'@'%' IDENTIFIED BY 'password';

-- Niveau 1: Global
GRANT PROCESS ON *.* TO 'complex_user'@'%';
-- Peut voir tous les processus serveur

-- Niveau 2: Base de données
GRANT SELECT, INSERT ON production.* TO 'complex_user'@'%';
-- Peut lire et insérer dans toutes les tables de production

-- Niveau 3: Table
GRANT UPDATE ON production.orders TO 'complex_user'@'%';
-- Peut modifier la table orders

-- Niveau 4: Colonne
GRANT DELETE ON production.logs TO 'complex_user'@'%';
-- Peut supprimer des logs

-- Résultat final:
-- ✓ PROCESS: Global (monitoring)
-- ✓ SELECT: production.* (lecture partout)
-- ✓ INSERT: production.* (insertion partout)
-- ✓ UPDATE: production.orders uniquement
-- ✓ DELETE: production.logs uniquement

-- Test
SHOW GRANTS FOR 'complex_user'@'%';
/*
GRANT PROCESS ON *.* TO 'complex_user'@'%'
GRANT SELECT, INSERT ON `production`.* TO 'complex_user'@'%'
GRANT UPDATE ON `production`.`orders` TO 'complex_user'@'%'
GRANT DELETE ON `production`.`logs` TO 'complex_user'@'%'
*/
```

---

## Principe du moindre privilège (Least Privilege)

### Définition et importance

Le **principe du moindre privilège** stipule que chaque utilisateur/service doit avoir **uniquement** les privilèges nécessaires pour accomplir sa fonction, et **rien de plus**.

**Pourquoi c'est crucial** :

1. **Limiter le blast radius** : Compromission d'un compte = dégâts limités
2. **Prévenir les erreurs humaines** : Impossible de DROP TABLE accidentel
3. **Conformité réglementaire** : RGPD, SOC2, PCI-DSS exigent
4. **Audit et traçabilité** : Qui peut faire quoi est clair
5. **Défense en profondeur** : Multiples couches de sécurité

### Anti-patterns courants

```sql
-- ❌ ANTI-PATTERN 1: ALL PRIVILEGES pour une application
GRANT ALL PRIVILEGES ON *.* TO 'webapp'@'%';
-- Problème: L'application peut DROP DATABASE, SHUTDOWN SERVER, etc.

-- ❌ ANTI-PATTERN 2: SUPER pour un service
GRANT SUPER ON *.* TO 'api_service'@'%';
-- Problème: Le service peut court-circuiter max_connections, modifier binlogs, etc.

-- ❌ ANTI-PATTERN 3: Privilèges globaux pour accès local
GRANT SELECT ON *.* TO 'analyst'@'%';
-- Problème: Accès à TOUTES les bases, même celles qu'il ne devrait pas voir

-- ❌ ANTI-PATTERN 4: Même utilisateur dev/staging/prod
CREATE USER 'app_user'@'%' IDENTIFIED BY 'same_password_everywhere';
GRANT ALL ON *.* TO 'app_user'@'%';
-- Problème: Compromission en dev = compromission en production
```

### Bonnes pratiques du moindre privilège

```sql
-- ✅ BONNE PRATIQUE 1: Privilèges minimaux par service
-- Application web CRUD
CREATE USER 'webapp_user'@'webapp_server_ip'
  IDENTIFIED VIA ed25519 USING PASSWORD('strong_pass');
GRANT SELECT, INSERT, UPDATE ON production.products TO 'webapp_user'@'webapp_server_ip';
GRANT SELECT, INSERT, UPDATE ON production.orders TO 'webapp_user'@'webapp_server_ip';
GRANT SELECT ON production.customers TO 'webapp_user'@'webapp_server_ip';
-- Pas de DELETE, pas de DROP, pas de DDL

-- ✅ BONNE PRATIQUE 2: Séparation lecture/écriture
-- Service analytics (lecture seule)
CREATE USER 'analytics_readonly'@'analytics_server'
  IDENTIFIED VIA ed25519 USING PASSWORD('strong_pass');
GRANT SELECT ON warehouse.* TO 'analytics_readonly'@'analytics_server';

-- ✅ BONNE PRATIQUE 3: Privilèges par environnement
-- Développement: Privilèges larges
GRANT ALL ON dev_db.* TO 'dev_alice'@'dev_network';

-- Production: Privilèges minimaux + audit
GRANT SELECT, INSERT, UPDATE ON prod_db.* TO 'prod_app'@'prod_server_ip'
  REQUIRE SSL;
-- Audit: Surveiller toute modification de privilèges en production

-- ✅ BONNE PRATIQUE 4: Privilèges par tâche (DBA)
-- DBA backup (uniquement ce qui est nécessaire)
GRANT SELECT, LOCK TABLES, RELOAD, REPLICATION CLIENT ON *.*
  TO 'backup_dba'@'localhost';

-- DBA monitoring (lecture seule admin)
GRANT PROCESS, REPLICATION CLIENT, SHOW DATABASES ON *.*
  TO 'monitor_dba'@'monitoring_server';

-- DBA production (tout sauf GRANT OPTION)
GRANT ALL PRIVILEGES ON *.* TO 'dba_prod'@'localhost';
-- Pas de WITH GRANT OPTION: Ne peut pas créer d'autres admins
```

### Matrice de privilèges recommandée par rôle

| Rôle | Privilèges typiques | Justification |
|------|---------------------|---------------|
| **Application web** | SELECT, INSERT, UPDATE sur tables métier | CRUD sans suppression ni DDL |
| **API publique** | SELECT, INSERT sur tables publiques + rate limiting | Exposition publique = risque élevé |
| **Service de paiement** | UPDATE(status) sur table orders | Modification limitée à une colonne |
| **Analytics/BI** | SELECT sur warehouse.* | Lecture seule pour reporting |
| **ETL/Data pipeline** | INSERT, UPDATE sur staging.* + FILE | Import/export de données |
| **Backup agent** | SELECT, LOCK TABLES, RELOAD, REPLICATION CLIENT | Backup cohérent |
| **Monitoring** | PROCESS, REPLICATION CLIENT, SHOW DATABASES | Visibilité sans modification |
| **Support N1** | PROCESS, CONNECTION_ADMIN 🆕 | Voir/tuer connexions bloquantes |
| **DBA junior** | DDL sur bases non-prod uniquement | Formation sans risque production |
| **DBA senior** | ALL sur *.* (sans GRANT OPTION) | Administration complète contrôlée |
| **Auditeur sécurité** | SHOW_ROUTINE 🆕, SELECT sur mysql.* | Audit sans modification |

---

## Vérification et audit des privilèges

### Commandes essentielles

```sql
-- 1. Voir les privilèges d'un utilisateur spécifique
SHOW GRANTS FOR 'user'@'host';

-- 2. Voir les privilèges de l'utilisateur actuel
SHOW GRANTS;
-- ou
SHOW GRANTS FOR CURRENT_USER();

-- 3. Lister tous les utilisateurs avec privilèges globaux
SELECT User, Host,
  Select_priv, Insert_priv, Update_priv, Delete_priv,
  Create_priv, Drop_priv, Reload_priv, Shutdown_priv,
  Process_priv, File_priv, Grant_priv, References_priv,
  Index_priv, Alter_priv, Show_db_priv, Super_priv,
  Create_tmp_table_priv, Lock_tables_priv, Execute_priv,
  Repl_slave_priv, Repl_client_priv, Create_view_priv,
  Show_view_priv, Create_routine_priv, Alter_routine_priv,
  Create_user_priv, Event_priv, Trigger_priv
FROM mysql.user
WHERE User != '';

-- 4. Utilisateurs avec privilèges sur une base spécifique
SELECT User, Host, Db,
  Select_priv, Insert_priv, Update_priv, Delete_priv,
  Create_priv, Drop_priv, Grant_priv
FROM mysql.db
WHERE Db = 'production';

-- 5. Privilèges par table
SELECT User, Host, Db, Table_name, Table_priv
FROM mysql.tables_priv
ORDER BY Db, Table_name;

-- 6. Privilèges par colonne
SELECT User, Host, Db, Table_name, Column_name, Column_priv
FROM mysql.columns_priv
WHERE Db = 'production';
```

### Scripts d'audit avancés

```sql
-- Script 1: Détecter les privilèges excessifs
-- Utilisateurs avec ALL PRIVILEGES global
SELECT User, Host, 'ALL PRIVILEGES ON *.*' AS issue
FROM mysql.user
WHERE Select_priv='Y' AND Insert_priv='Y' AND Update_priv='Y'
  AND Delete_priv='Y' AND Create_priv='Y' AND Drop_priv='Y'
  AND Reload_priv='Y' AND Shutdown_priv='Y' AND Process_priv='Y'
  AND File_priv='Y' AND Grant_priv='Y' AND References_priv='Y'
  AND Index_priv='Y' AND Alter_priv='Y';

-- Script 2: Utilisateurs avec GRANT OPTION
SELECT User, Host, 'Can grant privileges to others' AS risk
FROM mysql.user
WHERE Grant_priv = 'Y';

-- Script 3: Utilisateurs avec accès depuis '%' (anywhere)
SELECT User, Host, 'Accessible from anywhere' AS risk
FROM mysql.user
WHERE Host = '%';

-- Script 4: Utilisateurs avec FILE privilege (lecture/écriture fichiers)
SELECT User, Host, 'Can read/write files on server' AS risk
FROM mysql.user
WHERE File_priv = 'Y';

-- Script 5: Comptes sans mot de passe
SELECT User, Host, 'No password set' AS critical_risk
FROM mysql.user
WHERE authentication_string = '' OR authentication_string IS NULL;
```

---

## Diagnostic des problèmes de privilèges

### Erreurs courantes et solutions

**Erreur 1045: Access denied**

```sql
-- Symptôme
ERROR 1045 (28000): Access denied for user 'user'@'host' (using password: YES)

-- Diagnostic
-- 1. Vérifier que l'utilisateur existe
SELECT User, Host FROM mysql.user WHERE User = 'user';

-- 2. Vérifier la correspondance d'hôte
SELECT User, Host FROM mysql.user WHERE User = 'user' AND Host LIKE '10.%';

-- 3. Tester l'authentification
-- Si connexion réussit mais erreur 1045 persiste = problème de privilèges
```

**Erreur 1142: Command denied**

```sql
-- Symptôme
ERROR 1142 (42000): SELECT command denied to user 'user'@'host' for table 'table'

-- Diagnostic
SHOW GRANTS FOR 'user'@'host';
-- Vérifier si SELECT est accordé au niveau approprié

-- Solution
GRANT SELECT ON database.table TO 'user'@'host';
```

**Erreur 1044: Access denied for database**

```sql
-- Symptôme
ERROR 1044 (42000): Access denied for user 'user'@'host' to database 'db'

-- Diagnostic
SELECT User, Host, Db, Select_priv FROM mysql.db WHERE User = 'user' AND Db = 'db';

-- Solution
GRANT SELECT ON db.* TO 'user'@'host';
```

---

## ✅ Points clés à retenir

- **Le système de privilèges MariaDB est hiérarchique** : 5 niveaux du global au colonne
- **Les privilèges sont évalués du plus général au plus spécifique**, s'arrêtant au premier match
- **Les privilèges sont additifs**, pas restrictifs (impossible de "retirer" avec un niveau plus spécifique)
- **Plus de 40 privilèges** différents : DML (données), DDL (structure), Admin (serveur)
- **🆕 MariaDB 11.8 découpe SUPER** en privilèges granulaires (BINLOG_ADMIN, CONNECTION_ADMIN, SHOW_ROUTINE, etc.)
- **Le principe du moindre privilège est fondamental** : donner uniquement ce qui est nécessaire
- **Les privilèges par colonne sont coûteux** en performance : préférer les vues
- **Auditer régulièrement** les privilèges pour détecter les excès et anomalies
- **Privilèges globaux = très puissants** : à réserver aux DBA
- **Séparer les environnements** : dev ≠ staging ≠ production (utilisateurs différents)

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 GRANT Statement](https://mariadb.com/kb/en/grant/)
- [📖 Privilege System](https://mariadb.com/kb/en/grant/#privilege-levels)
- [📖 🆕 Granular Privileges (11.8)](https://mariadb.com/kb/en/grant/#new-privileges-in-mariadb-118)
- [📖 SHOW GRANTS](https://mariadb.com/kb/en/show-grants/)
- [📖 Privilege Tables](https://mariadb.com/kb/en/grant-tables/)

### Guides de sécurité

- [CIS MariaDB Benchmark](https://www.cisecurity.org/)
- [OWASP Database Security](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)

---

## ➡️ Sections suivantes

Les sous-sections détailleront l'utilisation pratique du système de privilèges :

- **10.3.1** : GRANT - Attribution de privilèges (syntaxe complète, exemples)
- **10.3.2** : REVOKE - Révocation de privilèges (retrait, cascade)
- **10.3.3** : Niveaux de privilèges (global, DB, table, colonne) avec cas pratiques

**La section suivante (10.3.1)** entrera dans le détail de la commande **GRANT** avec toutes ses variantes et options.

---


⏭️ [GRANT : Attribution de privilèges](/10-securite-gestion-utilisateurs/03.1-grant.md)

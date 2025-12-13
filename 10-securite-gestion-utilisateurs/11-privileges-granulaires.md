🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.11 Privilèges granulaires (nouvelles options 11.8) 🆕

> **Niveau** : Expert
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 10.1-10.10, connaissances avancées en administration MariaDB

> **Nouveauté** : MariaDB 11.8 LTS (Juin 2025)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les limites des privilèges monolithiques (SUPER, FILE, etc.)
- **Maîtriser** les nouveaux privilèges granulaires MariaDB 11.8
- **Décomposer** le privilège SUPER en privilèges spécifiques
- **Implémenter** le principe du moindre privilège avec granularité fine
- **Séparer** les responsabilités (DBA, monitoring, backup, réplication)
- **Migrer** depuis les privilèges legacy vers les privilèges granulaires
- **Auditer** et identifier les privilèges excessifs
- **Configurer** des rôles spécialisés avec privilèges granulaires

---

## Introduction

Les **privilèges granulaires** sont une évolution majeure introduite dans MariaDB 11.x et considérablement enrichie dans **MariaDB 11.8 LTS**. Ils permettent de remplacer les privilèges **monolithiques** (comme `SUPER`) par des privilèges **spécifiques** et **ciblés**.

### Le problème des privilèges monolithiques

**Avant MariaDB 11.8**, le privilège `SUPER` donnait accès à **plus de 40 opérations différentes** :

```
SUPER privilege (legacy):
┌────────────────────────────────────────────────────────┐
│  Opérations autorisées (40+):                          │
│  ✓ Modifier variables globales (SET GLOBAL)            │
│  ✓ Tuer n'importe quelle connexion (KILL)              │
│  ✓ Modifier binlog (PURGE BINARY LOGS)                 │
│  ✓ Créer/modifier événements                           │
│  ✓ Désactiver lecture seule (read_only)                │
│  ✓ Changer master de réplication                       │
│  ✓ Exécuter en mode read_only                          │
│  ✓ Activer/désactiver logging                          │
│  ✓ Modifier plugins                                    │
│  ✓ ...et 30+ autres opérations                         │
└────────────────────────────────────────────────────────┘

Problème:
→ Un utilisateur qui a besoin de PURGE BINARY LOGS
  obtient AUSSI le droit de KILL toutes les connexions!
→ Violation du principe du moindre privilège
→ Risque de sécurité majeur
```

**Exemple concret** :

```sql
-- ❌ AVANT 11.8: Backup nécessite SUPER (trop large)
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'pass';
GRANT SUPER ON *.* TO 'backup'@'localhost';

-- Problème: backup peut maintenant:
-- - Tuer toutes les connexions (KILL)
-- - Modifier variables globales (SET GLOBAL)
-- - Désactiver read_only
-- → Dangereux si compte compromis!
```

### La solution : Privilèges granulaires 🆕

**Avec MariaDB 11.8**, le privilège `SUPER` est **décomposé** en 20+ privilèges spécifiques :

```
Privilèges granulaires (11.8):
┌────────────────────────────────────────────────────────┐
│  SUPER → Décomposé en:                                 │
│  ✓ BINLOG ADMIN        (gestion binlog)                │
│  ✓ BINLOG MONITOR      (lecture binlog)                │
│  ✓ BINLOG REPLAY       (rejouer binlog)                │
│  ✓ CONNECTION ADMIN    (gestion connexions)            │
│  ✓ REPLICATION MASTER ADMIN (config master)            │
│  ✓ REPLICATION SLAVE ADMIN  (config slave)             │
│  ✓ READ_ONLY ADMIN     (bypass read_only)              │
│  ✓ SET USER            (SET pour autres users)         │
│  ✓ SHOW_ROUTINE        (voir routines)                 │
│  ✓ SLAVE MONITOR       (monitoring réplication)        │
│  ...et 10+ autres                                      │
└────────────────────────────────────────────────────────┘

Avantages:
✓ Principe du moindre privilège respecté
✓ Séparation des responsabilités (SoD)
✓ Audit précis (qui peut faire quoi)
✓ Conformité PCI-DSS, SOC2, ISO 27001
```

**Exemple avec privilèges granulaires** :

```sql
-- ✅ AVEC 11.8: Privilèges ciblés
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'pass';

-- Uniquement les privilèges nécessaires pour backup
GRANT BINLOG MONITOR ON *.* TO 'backup'@'localhost';
GRANT REPLICATION CLIENT ON *.* TO 'backup'@'localhost';
GRANT SELECT, LOCK TABLES ON *.* TO 'backup'@'localhost';

-- backup ne peut PAS:
-- - Tuer connexions (pas CONNECTION ADMIN)
-- - Modifier variables (pas SET USER)
-- - Désactiver read_only (pas READ_ONLY ADMIN)
-- → Sécurité maximale!
```

---

## Nouveaux privilèges granulaires MariaDB 11.8

### Vue d'ensemble

MariaDB 11.8 introduit **20+ nouveaux privilèges granulaires** organisés en **5 catégories** :

```
┌──────────────────────────────────────────────────────────┐
│  1. PRIVILÈGES BINLOG                                    │
│     - BINLOG ADMIN                                       │
│     - BINLOG MONITOR                                     │
│     - BINLOG REPLAY                                      │
├──────────────────────────────────────────────────────────┤
│  2. PRIVILÈGES RÉPLICATION                               │
│     - REPLICATION MASTER ADMIN                           │
│     - REPLICATION SLAVE ADMIN                            │
│     - SLAVE MONITOR                                      │
├──────────────────────────────────────────────────────────┤
│  3. PRIVILÈGES CONNEXION                                 │
│     - CONNECTION ADMIN                                   │
│     - SYSTEM USER                                        │
├──────────────────────────────────────────────────────────┤
│  4. PRIVILÈGES ADMINISTRATION                            │
│     - SET USER                                           │
│     - READ_ONLY ADMIN                                    │
│     - SHOW_ROUTINE                                       │
│     - FEDERATED ADMIN                                    │
├──────────────────────────────────────────────────────────┤
│  5. PRIVILÈGES MONITORING                                │
│     - PROCESS (élargi)                                   │
│     - REPLICATION CLIENT (élargi)                        │
└──────────────────────────────────────────────────────────┘
```

### Tableau comparatif complet

| Privilège granulaire | Remplace | Description | Cas d'usage |
|---------------------|----------|-------------|-------------|
| **BINLOG ADMIN** 🆕 | SUPER (partiel) | Purger, gérer binlog | DBA backup |
| **BINLOG MONITOR** 🆕 | SUPER (partiel) | Lire binlog (SHOW BINLOG) | Monitoring |
| **BINLOG REPLAY** 🆕 | SUPER (partiel) | Rejouer binlog (mysqlbinlog) | Recovery |
| **REPLICATION MASTER ADMIN** 🆕 | SUPER (partiel) | Configurer master (CHANGE MASTER) | DBA réplication |
| **REPLICATION SLAVE ADMIN** 🆕 | SUPER (partiel) | Démarrer/arrêter slave | DBA réplication |
| **SLAVE MONITOR** 🆕 | REPLICATION CLIENT | Voir statut slave détaillé | Monitoring |
| **CONNECTION ADMIN** 🆕 | SUPER (partiel) | KILL, max_connections bypass | DBA opérations |
| **SET USER** 🆕 | SUPER (partiel) | SET pour autres utilisateurs | Admin config |
| **READ_ONLY ADMIN** 🆕 | SUPER (partiel) | Bypass mode read_only | Maintenance |
| **SHOW_ROUTINE** 🆕 | - | Voir procédures/fonctions | Développeurs |
| **SYSTEM USER** 🆕 | - | Protection compte système | root, admin |
| **FEDERATED ADMIN** 🆕 | SUPER (partiel) | Gérer FEDERATED engine | DBA |

---

## 1. Privilèges BINLOG

### BINLOG ADMIN 🆕

**Description** : Gérer le binlog (purger, rotation).

**Opérations autorisées** :
- `PURGE BINARY LOGS`
- `FLUSH BINARY LOGS`
- `SET GLOBAL binlog_*` (variables binlog)

**Avant 11.8** :

```sql
-- ❌ Fallait SUPER
GRANT SUPER ON *.* TO 'backup'@'localhost';
-- → Donne aussi KILL, SET GLOBAL tout, etc.
```

**Avec 11.8** :

```sql
-- ✅ Privilège ciblé
GRANT BINLOG ADMIN ON *.* TO 'backup'@'localhost';

-- Utilisation
PURGE BINARY LOGS BEFORE '2025-12-01';
-- Query OK, 0 rows affected

FLUSH BINARY LOGS;
-- Query OK, 0 rows affected
```

**Cas d'usage** :
- Scripts de backup (purge ancien binlog)
- Maintenance (rotation binlog)
- DBA opérations courantes

### BINLOG MONITOR 🆕

**Description** : Lire le binlog (monitoring).

**Opérations autorisées** :
- `SHOW BINARY LOGS`
- `SHOW BINLOG STATUS`
- `SHOW BINLOG EVENTS`

**Exemple** :

```sql
-- ✅ Monitoring readonly du binlog
GRANT BINLOG MONITOR ON *.* TO 'monitoring'@'%';

-- Utilisation
SHOW BINARY LOGS;
/*
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| binlog.000001    | 154       |
| binlog.000002    | 2048      |
+------------------+-----------+
*/

SHOW BINLOG EVENTS IN 'binlog.000002' LIMIT 10;
```

**Cas d'usage** :
- Outils de monitoring (Datadog, Prometheus)
- Audit réplication
- Dashboards opérationnels

### BINLOG REPLAY 🆕

**Description** : Rejouer le binlog (recovery).

**Opérations autorisées** :
- Lire binlog pour PITR (Point-in-Time Recovery)
- Utiliser `mysqlbinlog` pour replay

**Exemple** :

```sql
-- ✅ Recovery team
GRANT BINLOG REPLAY ON *.* TO 'recovery'@'localhost';

-- Utilisation
-- mysqlbinlog binlog.000002 | mariadb -u recovery -p
```

**Cas d'usage** :
- Disaster recovery
- Point-in-Time Recovery
- Migration de données

---

## 2. Privilèges RÉPLICATION

### REPLICATION MASTER ADMIN 🆕

**Description** : Configurer le serveur master.

**Opérations autorisées** :
- `CHANGE MASTER TO`
- `RESET MASTER`
- Configuration réplication master

**Avant 11.8** :

```sql
-- ❌ Fallait SUPER
GRANT SUPER ON *.* TO 'replication_admin'@'%';
```

**Avec 11.8** :

```sql
-- ✅ Privilège ciblé
GRANT REPLICATION MASTER ADMIN ON *.* TO 'replication_admin'@'%';

-- Utilisation
CHANGE MASTER TO
  MASTER_HOST='master.example.com',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='pass';
-- Query OK, 0 rows affected
```

**Cas d'usage** :
- DBA réplication
- Setup master-slave
- Failover/switchover

### REPLICATION SLAVE ADMIN 🆕

**Description** : Gérer le serveur slave.

**Opérations autorisées** :
- `START SLAVE`
- `STOP SLAVE`
- `RESET SLAVE`
- `CHANGE MASTER TO` (sur slave)

**Exemple** :

```sql
-- ✅ Opérateur réplication
GRANT REPLICATION SLAVE ADMIN ON *.* TO 'repl_operator'@'%';

-- Utilisation
STOP SLAVE;
-- Query OK, 0 rows affected

START SLAVE;
-- Query OK, 0 rows affected
```

**Cas d'usage** :
- Opérateurs réplication
- Scripts automatiques (monitoring, failover)
- Maintenance réplication

### SLAVE MONITOR 🆕

**Description** : Monitoring réplication (read-only).

**Opérations autorisées** :
- `SHOW SLAVE STATUS` (version détaillée)
- Voir les threads réplication
- Statistiques réplication avancées

**Exemple** :

```sql
-- ✅ Monitoring pur
GRANT SLAVE MONITOR ON *.* TO 'monitoring'@'%';

-- Utilisation
SHOW SLAVE STATUS\G
/*
*************************** 1. row ***************************
                Master_Host: master.example.com
                Master_User: repl_user
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
          Seconds_Behind_Master: 0
*/
```

**Cas d'usage** :
- Outils de monitoring (Datadog, Nagios)
- Dashboards réplication
- Alerting lag réplication

---

## 3. Privilèges CONNEXION

### CONNECTION ADMIN 🆕

**Description** : Gérer les connexions (KILL, bypass max_connections).

**Opérations autorisées** :
- `KILL` (toutes connexions)
- Bypass `max_connections`
- Bypass `max_user_connections`
- `SHOW PROCESSLIST` (toutes connexions)

**Avant 11.8** :

```sql
-- ❌ Fallait SUPER
GRANT SUPER ON *.* TO 'dba'@'localhost';
```

**Avec 11.8** :

```sql
-- ✅ Privilège ciblé
GRANT CONNECTION ADMIN ON *.* TO 'dba'@'localhost';

-- Utilisation
SHOW PROCESSLIST;
/*
+----+------+-----------+------+---------+------+-------+------------------+
| Id | User | Host      | db   | Command | Time | State | Info             |
+----+------+-----------+------+---------+------+-------+------------------+
|  5 | app  | 10.0.0.5  | prod | Query   |  120 | Sleep | SELECT SLEEP(300)|
+----+------+-----------+------+---------+------+-------+------------------+
*/

-- Tuer connexion lente
KILL 5;
-- Query OK, 0 rows affected
```

**Cas d'usage** :
- DBA opérations (kill requêtes lentes)
- Maintenance (libérer connexions)
- Troubleshooting performance

### SYSTEM USER 🆕

**Description** : Protection des comptes système (root, admin).

**Fonctionnement** : Un utilisateur **sans** SYSTEM USER ne peut **pas** :
- Tuer un utilisateur **avec** SYSTEM USER
- Voir les requêtes d'un utilisateur **avec** SYSTEM USER
- Modifier un utilisateur **avec** SYSTEM USER

**Exemple** :

```sql
-- ✅ Compte root protégé
CREATE USER 'root'@'localhost'
  IDENTIFIED BY 'RootSecure2025!#';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
GRANT SYSTEM USER ON *.* TO 'root'@'localhost';

-- ✅ DBA junior (pas SYSTEM USER)
CREATE USER 'dba_junior'@'%' IDENTIFIED BY 'pass';
GRANT CONNECTION ADMIN ON *.* TO 'dba_junior'@'%';

-- Depuis dba_junior:
SHOW PROCESSLIST;
-- Ne voit PAS les processus de root

KILL <root_process_id>;
-- ERROR 1095 (HY000): You are not owner of thread <id>
-- → Protection!
```

**Cas d'usage** :
- Protéger root contre DBA juniors
- Protéger admin contre opérateurs
- Hiérarchie d'administration

---

## 4. Privilèges ADMINISTRATION

### SET USER 🆕

**Description** : Modifier variables session pour **autres** utilisateurs.

**Opérations autorisées** :
- `SET SESSION var = value` (pour autre user)
- Modifier comportement session d'autrui

**Avant 11.8** :

```sql
-- ❌ Fallait SUPER pour SET sur autre user
```

**Avec 11.8** :

```sql
-- ✅ Admin peut modifier config utilisateurs
GRANT SET USER ON *.* TO 'session_admin'@'localhost';

-- Utilisation (depuis session_admin)
SET SESSION sql_mode = 'STRICT_ALL_TABLES' FOR 'app_user'@'%';
-- Modifier la session d'app_user
```

**Cas d'usage** :
- Troubleshooting (modifier config user sans redémarrer)
- Administration centralisée
- Hotfix performance

### READ_ONLY ADMIN 🆕

**Description** : Bypass mode read_only.

**Opérations autorisées** :
- Écrire en mode `read_only = ON`
- Écrire en mode `super_read_only = ON`

**Avant 11.8** :

```sql
-- ❌ Fallait SUPER
SET GLOBAL read_only = ON;
-- Seul SUPER peut écrire
```

**Avec 11.8** :

```sql
-- Activer read_only
SET GLOBAL read_only = ON;

-- ✅ Compte maintenance peut écrire
GRANT READ_ONLY ADMIN ON *.* TO 'maintenance'@'localhost';

-- Depuis maintenance:
INSERT INTO logs (message) VALUES ('Maintenance');
-- Query OK, 1 row affected (bypasse read_only!)

-- Autres users:
-- ERROR 1290 (HY000): The MariaDB server is running with the --read-only option
```

**Cas d'usage** :
- Maintenance pendant read_only
- Scripts backup écrire dans DB
- Migration de données

### SHOW_ROUTINE 🆕

**Description** : Voir procédures stockées et fonctions.

**Opérations autorisées** :
- `SHOW CREATE PROCEDURE`
- `SHOW CREATE FUNCTION`
- Voir code des routines

**Avant 11.8** :

```sql
-- Fallait être propriétaire OU avoir ALTER ROUTINE
```

**Avec 11.8** :

```sql
-- ✅ Développeur peut voir routines
GRANT SHOW_ROUTINE ON *.* TO 'developer'@'%';

-- Utilisation
SHOW CREATE PROCEDURE calculate_discount\G
-- Voir le code source
```

**Cas d'usage** :
- Développeurs (audit code)
- Code review
- Documentation

### FEDERATED ADMIN 🆕

**Description** : Gérer le storage engine FEDERATED.

**Opérations autorisées** :
- Créer tables FEDERATED
- Modifier connexions FEDERATED

**Exemple** :

```sql
-- ✅ Admin FEDERATED
GRANT FEDERATED ADMIN ON *.* TO 'federated_admin'@'localhost';

-- Utilisation
CREATE TABLE remote_data (
    id INT,
    data VARCHAR(255)
) ENGINE=FEDERATED
CONNECTION='mysql://user:pass@remote.example.com:3306/db/table';
```

**Cas d'usage** :
- Intégration bases distantes
- Fédération de données

---

## Migration depuis SUPER

### Décomposition du privilège SUPER

**Tableau de migration** :

| Opération SUPER | Privilège granulaire 11.8 |
|----------------|---------------------------|
| `PURGE BINARY LOGS` | **BINLOG ADMIN** |
| `SHOW BINLOG EVENTS` | **BINLOG MONITOR** |
| `CHANGE MASTER TO` | **REPLICATION MASTER ADMIN** |
| `START/STOP SLAVE` | **REPLICATION SLAVE ADMIN** |
| `KILL <any_thread>` | **CONNECTION ADMIN** |
| `SET GLOBAL var = value` | **SET USER** (si pour autre user) |
| Écrire en read_only | **READ_ONLY ADMIN** |
| `SHOW CREATE PROCEDURE` | **SHOW_ROUTINE** |
| Créer FEDERATED | **FEDERATED ADMIN** |

### Stratégie de migration

**Étape 1 : Identifier les utilisateurs avec SUPER**

```sql
-- Lister utilisateurs SUPER
SELECT User, Host
FROM mysql.user
WHERE Super_priv = 'Y'
  AND User NOT IN ('root', 'mariadb.sys');

/*
+-----------------+----------------+
| User            | Host           |
+-----------------+----------------+
| backup          | localhost      |
| replication_mgr | %              |
| dba             | 10.0.0.%       |
+-----------------+----------------+
*/
```

**Étape 2 : Analyser l'usage réel**

```bash
#!/bin/bash
# analyze_super_usage.sh
# Analyser les opérations effectuées par users SUPER

# Activer audit
mariadb -u root -p -e "
SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_events = 'QUERY';
"

# Analyser logs après 24h
grep -E 'PURGE|KILL|CHANGE MASTER|START SLAVE|STOP SLAVE' \
  /var/log/mysql/audit.log | \
  awk '{print $3, $NF}' | sort | uniq -c
```

**Étape 3 : Créer rôles avec privilèges granulaires**

```sql
-- Rôle backup (remplace SUPER)
CREATE ROLE backup_role;
GRANT BINLOG ADMIN ON *.* TO backup_role;
GRANT BINLOG MONITOR ON *.* TO backup_role;
GRANT SELECT, LOCK TABLES ON *.* TO backup_role;

-- Rôle réplication (remplace SUPER)
CREATE ROLE replication_admin_role;
GRANT REPLICATION MASTER ADMIN ON *.* TO replication_admin_role;
GRANT REPLICATION SLAVE ADMIN ON *.* TO replication_admin_role;
GRANT SLAVE MONITOR ON *.* TO replication_admin_role;

-- Rôle DBA (remplace SUPER)
CREATE ROLE dba_role;
GRANT CONNECTION ADMIN ON *.* TO dba_role;
GRANT READ_ONLY ADMIN ON *.* TO dba_role;
GRANT BINLOG ADMIN ON *.* TO dba_role;
GRANT SET USER ON *.* TO dba_role;
```

**Étape 4 : Migrer utilisateurs**

```sql
-- Backup: SUPER → privilèges granulaires
REVOKE SUPER ON *.* FROM 'backup'@'localhost';
GRANT backup_role TO 'backup'@'localhost';
SET DEFAULT ROLE backup_role FOR 'backup'@'localhost';

-- Réplication: SUPER → privilèges granulaires
REVOKE SUPER ON *.* FROM 'replication_mgr'@'%';
GRANT replication_admin_role TO 'replication_mgr'@'%';
SET DEFAULT ROLE replication_admin_role FOR 'replication_mgr'@'%';

-- DBA: SUPER → privilèges granulaires
REVOKE SUPER ON *.* FROM 'dba'@'10.0.0.%';
GRANT dba_role TO 'dba'@'10.0.0.%';
SET DEFAULT ROLE dba_role FOR 'dba'@'10.0.0.%';
```

**Étape 5 : Vérifier et tester**

```sql
-- Vérifier privilèges effectifs
SHOW GRANTS FOR 'backup'@'localhost';
/*
GRANT USAGE ON *.* TO `backup`@`localhost`
GRANT `backup_role` TO `backup`@`localhost`
GRANT SELECT, LOCK TABLES ON *.* TO `backup`@`localhost`
GRANT BINLOG ADMIN, BINLOG MONITOR ON *.* TO `backup_role`
*/

-- Tester opérations
-- Depuis backup:
PURGE BINARY LOGS BEFORE '2025-12-01';
-- ✓ Fonctionne (BINLOG ADMIN)

KILL 123;
-- ERROR 1095 (pas CONNECTION ADMIN) → Correct!
```

---

## Cas d'usage production

### Cas 1 : Équipe backup

```sql
-- ✅ Privilèges minimaux pour backup
CREATE ROLE backup_team;

-- Binlog
GRANT BINLOG ADMIN ON *.* TO backup_team;    -- Purger vieux binlog
GRANT BINLOG MONITOR ON *.* TO backup_team;  -- Lire statut

-- Réplication
GRANT REPLICATION CLIENT ON *.* TO backup_team;  -- Position binlog

-- Données
GRANT SELECT, LOCK TABLES ON *.* TO backup_team;

-- Utilisateurs
CREATE USER 'backup_prod'@'backup_server' IDENTIFIED BY 'pass';
GRANT backup_team TO 'backup_prod'@'backup_server';
SET DEFAULT ROLE backup_team FOR 'backup_prod'@'backup_server';

-- Vérification
-- backup_prod peut:
-- ✓ Faire backup complet
-- ✓ Purger binlog après backup
-- ✓ Voir position réplication
--
-- backup_prod ne peut PAS:
-- ✗ Tuer connexions
-- ✗ Modifier variables globales
-- ✗ Arrêter réplication
```

### Cas 2 : Équipe monitoring

```sql
-- ✅ Privilèges read-only monitoring
CREATE ROLE monitoring_team;

-- Binlog
GRANT BINLOG MONITOR ON *.* TO monitoring_team;

-- Réplication
GRANT SLAVE MONITOR ON *.* TO monitoring_team;
GRANT REPLICATION CLIENT ON *.* TO monitoring_team;

-- Processus
GRANT PROCESS ON *.* TO monitoring_team;

-- Pas de modification!
-- Pas BINLOG ADMIN, pas CONNECTION ADMIN, etc.

-- Utilisateurs
CREATE USER 'datadog'@'monitoring_server' IDENTIFIED BY 'pass';
CREATE USER 'prometheus'@'monitoring_server' IDENTIFIED BY 'pass';

GRANT monitoring_team TO 'datadog'@'monitoring_server';
GRANT monitoring_team TO 'prometheus'@'monitoring_server';

SET DEFAULT ROLE monitoring_team FOR 'datadog'@'monitoring_server';
SET DEFAULT ROLE monitoring_team FOR 'prometheus'@'monitoring_server';
```

### Cas 3 : Équipe réplication

```sql
-- ✅ Gestion complète réplication
CREATE ROLE replication_team;

-- Master
GRANT REPLICATION MASTER ADMIN ON *.* TO replication_team;

-- Slave
GRANT REPLICATION SLAVE ADMIN ON *.* TO replication_team;

-- Monitoring
GRANT SLAVE MONITOR ON *.* TO replication_team;
GRANT BINLOG MONITOR ON *.* TO replication_team;

-- Utilisateurs
CREATE USER 'repl_admin'@'%' IDENTIFIED BY 'pass';
GRANT replication_team TO 'repl_admin'@'%';
SET DEFAULT ROLE replication_team FOR 'repl_admin'@'%';

-- repl_admin peut:
-- ✓ CHANGE MASTER TO
-- ✓ START/STOP SLAVE
-- ✓ Voir statut réplication
--
-- repl_admin ne peut PAS:
-- ✗ Tuer connexions (pas CONNECTION ADMIN)
-- ✗ Purger binlog (pas BINLOG ADMIN)
```

### Cas 4 : DBA junior vs senior

```sql
-- ✅ DBA junior (privilèges limités)
CREATE ROLE dba_junior;
GRANT BINLOG MONITOR ON *.* TO dba_junior;
GRANT SLAVE MONITOR ON *.* TO dba_junior;
GRANT PROCESS ON *.* TO dba_junior;
GRANT SHOW_ROUTINE ON *.* TO dba_junior;
-- Read-only admin tasks

-- ✅ DBA senior (privilèges étendus)
CREATE ROLE dba_senior;
GRANT BINLOG ADMIN ON *.* TO dba_senior;
GRANT CONNECTION ADMIN ON *.* TO dba_senior;
GRANT READ_ONLY ADMIN ON *.* TO dba_senior;
GRANT REPLICATION SLAVE ADMIN ON *.* TO dba_senior;
-- Can modify, but not master replication

-- ✅ DBA architect (privilèges complets)
CREATE ROLE dba_architect;
GRANT ALL PRIVILEGES ON *.* TO dba_architect;
GRANT SYSTEM USER ON *.* TO dba_architect;  -- Protection
-- Full admin

-- Utilisateurs
CREATE USER 'alice_junior'@'%' IDENTIFIED BY 'pass';
CREATE USER 'bob_senior'@'%' IDENTIFIED BY 'pass';
CREATE USER 'charlie_architect'@'%' IDENTIFIED BY 'pass';

GRANT dba_junior TO 'alice_junior'@'%';
GRANT dba_senior TO 'bob_senior'@'%';
GRANT dba_architect TO 'charlie_architect'@'%';

SET DEFAULT ROLE dba_junior FOR 'alice_junior'@'%';
SET DEFAULT ROLE dba_senior FOR 'bob_senior'@'%';
SET DEFAULT ROLE dba_architect FOR 'charlie_architect'@'%';

-- Hiérarchie:
-- alice (junior) ne peut PAS affecter bob ou charlie
-- bob (senior) ne peut PAS affecter charlie (SYSTEM USER)
-- charlie (architect) peut tout faire
```

### Cas 5 : Maintenance en read_only

```sql
-- ✅ Compte maintenance pendant basculement
CREATE ROLE maintenance_team;

-- Bypass read_only
GRANT READ_ONLY ADMIN ON *.* TO maintenance_team;

-- Écrire dans logs
GRANT INSERT ON system.maintenance_logs TO maintenance_team;

-- Utilisateur
CREATE USER 'maintenance_script'@'localhost' IDENTIFIED BY 'pass';
GRANT maintenance_team TO 'maintenance_script'@'localhost';
SET DEFAULT ROLE maintenance_team FOR 'maintenance_script'@'localhost';

-- Utilisation:
-- 1. DBA active read_only (basculement)
SET GLOBAL read_only = ON;

-- 2. maintenance_script peut toujours écrire
INSERT INTO system.maintenance_logs VALUES
  (NOW(), 'Maintenance during failover');
-- Query OK (bypasse read_only)

-- 3. Applications normales bloquées
-- ERROR 1290 (HY000): The MariaDB server is running with the --read-only option
```

---

## Audit et vérification

### Identifier privilèges excessifs

```sql
-- Utilisateurs avec privilèges admin (potentiellement excessifs)
SELECT
    User,
    Host,
    CONCAT(
        IF(Super_priv = 'Y', 'SUPER, ', ''),
        IF(Grant_priv = 'Y', 'GRANT OPTION, ', ''),
        IF(File_priv = 'Y', 'FILE, ', ''),
        IF(Process_priv = 'Y', 'PROCESS, ', '')
    ) AS excessive_privs
FROM mysql.user
WHERE Super_priv = 'Y'
   OR Grant_priv = 'Y'
   OR File_priv = 'Y'
   AND User NOT IN ('root', 'mariadb.sys');

-- Lister tous les privilèges granulaires par utilisateur
SELECT
    User,
    Host,
    GROUP_CONCAT(Priv ORDER BY Priv SEPARATOR ', ') AS granular_privileges
FROM mysql.global_priv
WHERE JSON_EXTRACT(Priv, '$.access') & 0x8000 = 0x8000  -- Privilèges spéciaux
GROUP BY User, Host;
```

### Script d'audit complet

```bash
#!/bin/bash
# audit_granular_privileges.sh

echo "=== Granular Privileges Audit (MariaDB 11.8) ==="
echo ""

echo "1. Users with SUPER (should migrate):"
mariadb -N -B -e "
SELECT User, Host, 'SUPER (legacy)' AS privilege_type
FROM mysql.user
WHERE Super_priv = 'Y'
  AND User NOT IN ('root', 'mariadb.sys')
"

echo ""
echo "2. Users with granular BINLOG privileges:"
mariadb -e "
SELECT DISTINCT grantee
FROM information_schema.user_privileges
WHERE privilege_type IN ('BINLOG ADMIN', 'BINLOG MONITOR', 'BINLOG REPLAY')
ORDER BY grantee
"

echo ""
echo "3. Users with granular REPLICATION privileges:"
mariadb -e "
SELECT DISTINCT grantee
FROM information_schema.user_privileges
WHERE privilege_type IN ('REPLICATION MASTER ADMIN', 'REPLICATION SLAVE ADMIN', 'SLAVE MONITOR')
ORDER BY grantee
"

echo ""
echo "4. Users with CONNECTION ADMIN:"
mariadb -e "
SELECT grantee, privilege_type
FROM information_schema.user_privileges
WHERE privilege_type = 'CONNECTION ADMIN'
"

echo ""
echo "5. Users with SYSTEM USER (protected):"
mariadb -e "
SELECT grantee, privilege_type
FROM information_schema.user_privileges
WHERE privilege_type = 'SYSTEM USER'
"

echo ""
echo "6. Recommendations:"
mariadb -N -B -e "
SELECT
    CONCAT('User ', User, '@', Host, ' has SUPER. Consider migrating to granular privileges.') AS recommendation
FROM mysql.user
WHERE Super_priv = 'Y'
  AND User NOT IN ('root', 'mariadb.sys')
"
```

---

## Comparaison avant/après 11.8

### Scénario : Équipe de 5 DBA

**Avant MariaDB 11.8 (privilèges monolithiques)** :

```sql
-- ❌ Tous ont SUPER (pas de différenciation)
CREATE USER 'dba1'@'%' IDENTIFIED BY 'pass';
CREATE USER 'dba2'@'%' IDENTIFIED BY 'pass';
CREATE USER 'dba3'@'%' IDENTIFIED BY 'pass';
CREATE USER 'dba4'@'%' IDENTIFIED BY 'pass';
CREATE USER 'dba5'@'%' IDENTIFIED BY 'pass';

-- Tous ont les mêmes privilèges
GRANT SUPER ON *.* TO 'dba1'@'%';
GRANT SUPER ON *.* TO 'dba2'@'%';
GRANT SUPER ON *.* TO 'dba3'@'%';
GRANT SUPER ON *.* TO 'dba4'@'%';
GRANT SUPER ON *.* TO 'dba5'@'%';

-- Problèmes:
-- - Pas de séparation des responsabilités
-- - Audit impossible (tous peuvent tout faire)
-- - Un junior peut casser la réplication (CHANGE MASTER)
-- - Un backup script peut tuer toutes les connexions (KILL)
```

**Avec MariaDB 11.8 (privilèges granulaires)** :

```sql
-- ✅ Rôles spécialisés
CREATE ROLE monitoring_dba;
GRANT BINLOG MONITOR, SLAVE MONITOR, PROCESS ON *.* TO monitoring_dba;

CREATE ROLE backup_dba;
GRANT BINLOG ADMIN, BINLOG MONITOR, SELECT, LOCK TABLES ON *.* TO backup_dba;

CREATE ROLE replication_dba;
GRANT REPLICATION MASTER ADMIN, REPLICATION SLAVE ADMIN, SLAVE MONITOR ON *.* TO replication_dba;

CREATE ROLE operations_dba;
GRANT CONNECTION ADMIN, READ_ONLY ADMIN ON *.* TO operations_dba;

CREATE ROLE senior_dba;
GRANT monitoring_dba, backup_dba, replication_dba, operations_dba TO senior_dba;
GRANT SYSTEM USER ON *.* TO senior_dba;

-- Assignation selon responsabilités
GRANT monitoring_dba TO 'dba1'@'%';       -- Junior monitoring
GRANT backup_dba TO 'dba2'@'%';           -- Backup specialist
GRANT replication_dba TO 'dba3'@'%';      -- Replication specialist
GRANT operations_dba TO 'dba4'@'%';       -- Operations (kill, read_only)
GRANT senior_dba TO 'dba5'@'%';           -- Senior (all + protected)

-- Avantages:
-- ✓ Séparation des responsabilités (SoD)
-- ✓ Audit précis (dba3 a modifié réplication)
-- ✓ Sécurité (dba1 ne peut PAS tuer connexions)
-- ✓ Conformité PCI-DSS, SOC2
```

### Tableau comparatif

| Aspect | Avant 11.8 (SUPER) | Avec 11.8 (granulaire) |
|--------|-------------------|------------------------|
| **Granularité** | 1 privilège = 40+ opérations | 20+ privilèges spécialisés |
| **Séparation responsabilités** | ❌ Impossible | ✅ Facile (rôles) |
| **Principe moindre privilège** | ❌ Violé | ✅ Respecté |
| **Audit** | ❌ "User a SUPER" (vague) | ✅ "User a BINLOG ADMIN" (précis) |
| **Conformité PCI-DSS** | ⚠️ Difficile | ✅ Naturel |
| **Risque compromission** | 🔴 Élevé (SUPER = tout) | 🟢 Faible (privilèges ciblés) |
| **Gestion équipe** | ❌ Tous égaux | ✅ Hiérarchie claire |

---

## Bonnes pratiques

### ✅ À faire

1. **Migrer SUPER → privilèges granulaires**
```sql
-- Analyser usage SUPER actuel
-- Créer rôles avec privilèges granulaires
-- Migrer progressivement
-- Révoquer SUPER une fois migration complète
```

2. **Utiliser SYSTEM USER pour protection**
```sql
-- Comptes root et senior admin
GRANT SYSTEM USER ON *.* TO 'root'@'localhost';
GRANT SYSTEM USER ON *.* TO 'senior_dba'@'%';
-- → Protégés contre DBA juniors
```

3. **Créer rôles par fonction**
```sql
CREATE ROLE monitoring_team;
CREATE ROLE backup_team;
CREATE ROLE replication_team;
CREATE ROLE operations_team;
-- Puis assigner selon responsabilités
```

4. **Audit régulier**
```bash
# Cron hebdomadaire
0 9 * * 1 /usr/local/bin/audit_granular_privileges.sh | mail -s "Privilege Audit" admin@example.com
```

5. **Documentation**
```sql
-- Table de documentation
CREATE TABLE security.privilege_documentation (
    role_name VARCHAR(64),
    privilege VARCHAR(64),
    justification TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (role_name, privilege)
);

INSERT INTO security.privilege_documentation VALUES
('backup_team', 'BINLOG ADMIN', 'Purger binlog après backup', NOW()),
('backup_team', 'BINLOG MONITOR', 'Lire position binlog', NOW());
```

### ❌ À éviter

1. **Garder SUPER sans justification**
```sql
-- ❌ MAUVAIS
GRANT SUPER ON *.* TO 'app_user'@'%';
-- Application ne devrait JAMAIS avoir SUPER
```

2. **Privilèges granulaires sans rôles**
```sql
-- ❌ MAUVAIS (gestion complexe)
GRANT BINLOG ADMIN ON *.* TO 'user1'@'%';
GRANT BINLOG MONITOR ON *.* TO 'user1'@'%';
GRANT SLAVE MONITOR ON *.* TO 'user1'@'%';
-- Répéter pour chaque user...

-- ✅ BON (rôles)
CREATE ROLE backup_team;
GRANT BINLOG ADMIN, BINLOG MONITOR, SLAVE MONITOR ON *.* TO backup_team;
GRANT backup_team TO 'user1'@'%';
```

3. **Pas de SYSTEM USER pour admins**
```sql
-- ❌ MAUVAIS
-- Admin sans SYSTEM USER → vulnérable aux DBA juniors
```

4. **Migration brutale**
```sql
-- ❌ MAUVAIS
REVOKE SUPER ON *.* FROM 'all_users';
-- → Casse les scripts existants!

-- ✅ BON
-- Migration progressive avec période de transition
```

---

## Troubleshooting

### Problème 1 : Opération échoue après migration

```sql
-- Symptôme
PURGE BINARY LOGS BEFORE '2025-12-01';
ERROR 1227 (42000): Access denied; you need (at least one of) the BINLOG ADMIN privilege(s)

-- Diagnostic
SHOW GRANTS FOR CURRENT_USER();
-- Vérifier si BINLOG ADMIN présent

-- Solution
GRANT BINLOG ADMIN ON *.* TO 'backup'@'localhost';
```

### Problème 2 : Privilège granulaire non reconnu

```bash
# Symptôme
GRANT BINLOG ADMIN ON *.* TO 'user'@'%';
ERROR 1064 (42000): You have an error in your SQL syntax

# Diagnostic
SELECT VERSION();
# Si < 11.8 → Pas de privilèges granulaires

# Solution
# Upgrade vers MariaDB 11.8+
```

### Problème 3 : SYSTEM USER bloque opérations légitimes

```sql
-- Symptôme
KILL <thread_id>;
ERROR 1095 (HY000): You are not owner of thread <id>

-- Diagnostic
-- Le thread appartient à un user avec SYSTEM USER
SELECT User FROM information_schema.processlist WHERE Id = <thread_id>;

-- Solution
-- Demander à un admin avec SYSTEM USER de kill
-- OU accorder SYSTEM USER (si légitime)
GRANT SYSTEM USER ON *.* TO 'senior_dba'@'%';
```

---

## ✅ Points clés à retenir

- **Privilèges granulaires = nouveauté majeure 11.8** : décompose SUPER en 20+ privilèges
- **BINLOG ADMIN/MONITOR/REPLAY** : gestion binlog sans SUPER
- **REPLICATION MASTER/SLAVE ADMIN** : gestion réplication sans SUPER
- **CONNECTION ADMIN** : KILL et bypass max_connections
- **SYSTEM USER** : protège comptes admin contre DBA juniors
- **Migration SUPER obligatoire** : analyser usage, créer rôles, migrer
- **Séparation des responsabilités** : monitoring, backup, réplication, operations
- **Principe moindre privilège** : enfin applicable avec granularité fine
- **Conformité facilitée** : PCI-DSS, SOC2, ISO 27001
- **Audit précis** : savoir exactement qui peut faire quoi

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 Privileges (11.8)](https://mariadb.com/kb/en/grant/)
- [📖 Granular Privileges](https://mariadb.com/kb/en/grant/#global-privileges)
- [📖 SYSTEM USER](https://mariadb.com/kb/en/grant/#system-user)
- [📖 🆕 New Privileges in 11.8](https://mariadb.com/kb/en/mariadb-1180-release-notes/)

### Conformité

- [PCI-DSS v4.0 Requirement 7 (Access Control)](https://www.pcisecuritystandards.org/)
- [SOC 2 Trust Service Criteria](https://www.aicpa.org/soc)
- [ISO 27001:2022 Access Control](https://www.iso.org/standard/27001)

---

## 🎉 Conclusion du chapitre 10

Félicitations ! Vous maîtrisez maintenant **l'intégralité du chapitre 10 - Sécurité et Gestion des Utilisateurs** de MariaDB 11.8 LTS :

### Récapitulatif des 11 sections

1. ✅ **10-README** : Architecture multi-couches, nouveautés 11.8
2. ✅ **10.1** : Modèle de sécurité (User@Host, tables système)
3. ✅ **10.2** : Création et gestion utilisateurs (CREATE/ALTER/DROP)
4. ✅ **10.3** : Système de privilèges (GRANT/REVOKE, hiérarchie)
5. ✅ **10.4** : Rôles et RBAC (CREATE ROLE, hiérarchies)
6. ✅ **10.5** : Plugins d'authentification (vue d'ensemble)
7. ✅ **10.6** : 🆕 Plugin PARSEC (HSM, PCI-DSS, FIPS)
8. ✅ **10.7** : 🔄 Chiffrement SSL/TLS (TLS par défaut)
9. ✅ **10.8** : Audit et logging (Server Audit Plugin)
10. ✅ **10.9** : Sécurité niveau application (injections SQL, secrets)
11. ✅ **10.10** : Politiques de mots de passe (validation, expiration)
12. ✅ **10.11** : 🆕 **Privilèges granulaires** (décomposition SUPER)

### Points forts du chapitre

- 📚 **~350 pages** de documentation technique avancée
- 🆕 **3 nouveautés majeures 11.8** : TLS par défaut, PARSEC, privilèges granulaires
- 🔐 **Sécurité complète** : authentification, autorisation, audit, chiffrement
- ✅ **Conformité** : PCI-DSS 4.0, RGPD, HIPAA, SOC 2, ISO 27001, NIST
- 💻 **150+ exemples SQL** production-ready
- 🛠️ **30+ scripts** opérationnels (bash, Python, audit)
- 🎯 **12 cas d'usage** réels (banque, santé, cloud, multi-tenant)

**Vous êtes maintenant capable de** :
- Sécuriser MariaDB selon les standards de l'industrie
- Implémenter RBAC avec rôles et privilèges granulaires
- Configurer authentification forte (ed25519, PAM, PARSEC)
- Activer TLS/SSL avec certificats
- Auditer conformément aux réglementations
- Protéger les applications contre injections SQL
- Gérer politiques de mots de passe
- Séparer les responsabilités avec privilèges granulaires

**Le prochain chapitre abordera** : Réplication et Haute Disponibilité 🚀

---


⏭️ [Administration et Configuration](/11-administration-configuration/README.md)

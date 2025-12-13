🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4 Rôles (CREATE ROLE, SET ROLE, DEFAULT ROLE)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 10.1-10.3 (Modèle de sécurité, Utilisateurs, Privilèges)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le concept de RBAC (Role-Based Access Control) dans MariaDB
- **Créer et gérer** des rôles avec CREATE ROLE, DROP ROLE
- **Attribuer** des privilèges à des rôles et des rôles à des utilisateurs
- **Activer** des rôles avec SET ROLE et configurer des rôles par défaut
- **Implémenter** une hiérarchie de rôles (rôles imbriqués)
- **Migrer** d'une gestion par utilisateurs vers une gestion par rôles
- **Auditer** et documenter les rôles en production
- **Appliquer** les meilleures pratiques RBAC dans des architectures complexes

---

## Introduction

Les **rôles** (roles) sont une fonctionnalité introduite dans MariaDB 10.0.5 qui révolutionne la gestion des privilèges. Au lieu d'accorder des privilèges individuellement à chaque utilisateur, vous créez des **rôles nommés** qui regroupent des ensembles de privilèges, puis attribuez ces rôles aux utilisateurs.

### Problème sans les rôles

**Scénario** : Vous avez 50 développeurs qui doivent tous avoir les mêmes privilèges sur la base de développement.

**Sans rôles** (approche traditionnelle) :

```sql
-- Répéter 50 fois (une fois par développeur) 😱
GRANT SELECT, INSERT, UPDATE, DELETE ON dev_db.* TO 'dev1'@'%';
GRANT CREATE, ALTER, DROP, INDEX ON dev_db.* TO 'dev1'@'%';
GRANT CREATE TEMPORARY TABLES ON dev_db.* TO 'dev1'@'%';

GRANT SELECT, INSERT, UPDATE, DELETE ON dev_db.* TO 'dev2'@'%';
GRANT CREATE, ALTER, DROP, INDEX ON dev_db.* TO 'dev2'@'%';
GRANT CREATE TEMPORARY TABLES ON dev_db.* TO 'dev2'@'%';

-- ... répéter 48 fois de plus

-- Si vous devez modifier les privilèges :
-- → Répéter GRANT/REVOKE 50 fois
-- → Risque d'erreur élevé
-- → Impossible à maintenir
```

**Avec rôles** (approche moderne) :

```sql
-- 1. Créer un rôle une seule fois
CREATE ROLE developer;

-- 2. Attribuer les privilèges au rôle (une fois)
GRANT SELECT, INSERT, UPDATE, DELETE ON dev_db.* TO developer;
GRANT CREATE, ALTER, DROP, INDEX ON dev_db.* TO developer;
GRANT CREATE TEMPORARY TABLES ON dev_db.* TO developer;

-- 3. Attribuer le rôle aux 50 développeurs
GRANT developer TO 'dev1'@'%';
GRANT developer TO 'dev2'@'%';
-- ... 48 fois (une ligne simple par développeur)

-- Modification des privilèges :
-- → Une seule commande, appliquée à tous
GRANT EXECUTE ON dev_db.* TO developer;
-- Tous les développeurs ont instantanément le nouveau privilège ✓
```

**Gain** :
- ✅ Maintenance simplifiée (1 commande au lieu de 50)
- ✅ Cohérence garantie (tous les développeurs ont les mêmes droits)
- ✅ Lisibilité améliorée (rôles nommés = intention claire)
- ✅ Audit facilité (voir les rôles au lieu de milliers de lignes GRANT)

---

## Concept de RBAC (Role-Based Access Control)

### Qu'est-ce que RBAC ?

**RBAC** (Role-Based Access Control) est un modèle de contrôle d'accès où les permissions sont associées à des **rôles** plutôt qu'à des **utilisateurs individuels**.

```
Modèle traditionnel (ACL):
Utilisateur → Privilèges directs

Modèle RBAC:
Utilisateur → Rôle(s) → Privilèges
```

**Architecture RBAC dans MariaDB** :

```
┌─────────────────────────────────────────────────────────────┐
│  UTILISATEURS (Personnes/Services)                          │
│  - alice@'%'                                                │
│  - bob@'localhost'                                          │
│  - api_service@'10.0.0.%'                                   │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ GRANT role TO user
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  RÔLES (Fonctions métier)                                   │
│  - developer                                                │
│  - analyst                                                  │
│  - dba_readonly                                             │
│  - app_backend                                              │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ GRANT privilege TO role
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  PRIVILÈGES (Permissions)                                   │
│  - SELECT, INSERT, UPDATE, DELETE                           │
│  - CREATE, ALTER, DROP                                      │
│  - EXECUTE, TRIGGER, EVENT                                  │
└─────────────────────────────────────────────────────────────┘
```

### Avantages de RBAC

| Aspect | Sans RBAC | Avec RBAC |
|--------|-----------|-----------|
| **Maintenance** | 1 commande × N utilisateurs | 1 commande pour tous |
| **Cohérence** | Risque de divergence | Garantie par construction |
| **Onboarding** | Copier-coller de privilèges | 1 GRANT (attribution rôle) |
| **Offboarding** | REVOKE multiples | REVOKE 1 rôle |
| **Audit** | Milliers de lignes | Quelques rôles à vérifier |
| **Documentation** | Difficile | Noms de rôles auto-documentants |
| **Conformité** | Complexe | Simplifié (rôles = fonctions) |

---

## CREATE ROLE - Création de rôles

### Syntaxe

```sql
CREATE [OR REPLACE] ROLE [IF NOT EXISTS] role_name [, role_name] ...
  [WITH ADMIN {CURRENT_USER | CURRENT_ROLE | user_or_role}];
```

### Création basique

```sql
-- Créer un rôle simple
CREATE ROLE developer;

-- Créer plusieurs rôles en une commande
CREATE ROLE developer, analyst, dba_readonly;

-- Créer un rôle uniquement s'il n'existe pas
CREATE ROLE IF NOT EXISTS app_backend;

-- Remplacer un rôle existant
CREATE OR REPLACE ROLE legacy_role;
```

**Vérification** :

```sql
-- Voir tous les rôles
SELECT User AS role_name, Host, is_role
FROM mysql.user
WHERE is_role = 'Y';

/*
+----------------+------+---------+
| role_name      | Host | is_role |
+----------------+------+---------+
| developer      |      | Y       |
| analyst        |      | Y       |
| dba_readonly   |      | Y       |
| app_backend    |      | Y       |
+----------------+------+---------+
*/
```

💡 **Note** : Les rôles sont stockés dans `mysql.user` avec `is_role = 'Y'` et `Host = ''`.

### WITH ADMIN - Gestion déléguée

L'option `WITH ADMIN` permet de déléguer la gestion d'un rôle.

```sql
-- Le rôle 'team_lead' peut gérer le rôle 'developer'
CREATE ROLE developer WITH ADMIN 'team_lead'@'localhost';

-- Maintenant 'team_lead' peut :
-- 1. Attribuer/révoquer le rôle developer à d'autres
-- 2. Modifier les privilèges du rôle developer
```

**Exemple concret** :

```sql
-- Scénario: Hiérarchie d'équipe
CREATE ROLE developer;
CREATE ROLE senior_developer WITH ADMIN 'tech_lead'@'%';

-- Le tech_lead peut gérer senior_developer
-- Connexion en tant que tech_lead:
GRANT senior_developer TO 'alice'@'%';
GRANT senior_developer TO 'bob'@'%';
```

---

## GRANT avec les rôles

### Accorder des privilèges à un rôle

```sql
-- Créer un rôle
CREATE ROLE app_backend;

-- Accorder des privilèges au rôle
GRANT SELECT, INSERT, UPDATE ON production.orders TO app_backend;
GRANT SELECT ON production.products TO app_backend;
GRANT EXECUTE ON PROCEDURE production.calculate_shipping TO app_backend;

-- Vérifier les privilèges du rôle
SHOW GRANTS FOR app_backend;
/*
GRANT USAGE ON *.* TO `app_backend`
GRANT SELECT, INSERT, UPDATE ON `production`.`orders` TO `app_backend`
GRANT SELECT ON `production`.`products` TO `app_backend`
GRANT EXECUTE ON PROCEDURE `production`.`calculate_shipping` TO `app_backend`
*/
```

### Attribuer un rôle à un utilisateur

```sql
-- Créer des utilisateurs
CREATE USER 'api_service_1'@'10.0.0.10' IDENTIFIED BY 'password';
CREATE USER 'api_service_2'@'10.0.0.11' IDENTIFIED BY 'password';

-- Attribuer le rôle app_backend aux utilisateurs
GRANT app_backend TO 'api_service_1'@'10.0.0.10';
GRANT app_backend TO 'api_service_2'@'10.0.0.11';

-- Vérifier les rôles d'un utilisateur
SHOW GRANTS FOR 'api_service_1'@'10.0.0.10';
/*
GRANT USAGE ON *.* TO `api_service_1`@`10.0.0.10` IDENTIFIED BY PASSWORD '...'
GRANT `app_backend` TO `api_service_1`@`10.0.0.10`
*/
```

### Attribuer plusieurs rôles à un utilisateur

Un utilisateur peut avoir **plusieurs rôles** simultanément.

```sql
-- Créer des rôles spécialisés
CREATE ROLE read_orders;
CREATE ROLE write_logs;
CREATE ROLE execute_reports;

GRANT SELECT ON production.orders TO read_orders;
GRANT INSERT ON logs.* TO write_logs;
GRANT EXECUTE ON analytics.* TO execute_reports;

-- Utilisateur avec 3 rôles
CREATE USER 'multi_role_user'@'%' IDENTIFIED BY 'password';
GRANT read_orders, write_logs, execute_reports TO 'multi_role_user'@'%';

-- Vérification
SHOW GRANTS FOR 'multi_role_user'@'%';
/*
GRANT USAGE ON *.* TO `multi_role_user`@`%` ...
GRANT `read_orders` TO `multi_role_user`@`%`
GRANT `write_logs` TO `multi_role_user`@`%`
GRANT `execute_reports` TO `multi_role_user`@`%`
*/
```

---

## SET ROLE - Activation de rôles

Par défaut, lorsqu'un utilisateur se connecte, ses rôles **ne sont pas automatiquement activés** (sauf si configurés comme rôles par défaut).

### Activation manuelle

```sql
-- Connexion en tant que multi_role_user
mariadb -u multi_role_user -p

-- Les rôles ne sont pas actifs par défaut
SELECT CURRENT_ROLE();
/*
+----------------+
| CURRENT_ROLE() |
+----------------+
| NULL           |
+----------------+
*/

-- Activer tous les rôles attribués
SET ROLE ALL;

SELECT CURRENT_ROLE();
/*
+-----------------------------------------------+
| CURRENT_ROLE()                                |
+-----------------------------------------------+
| `read_orders`,`write_logs`,`execute_reports`  |
+-----------------------------------------------+
*/

-- Maintenant l'utilisateur a tous les privilèges des 3 rôles
```

### Activation sélective

```sql
-- Activer un seul rôle
SET ROLE read_orders;

SELECT CURRENT_ROLE();
/*
+--------------+
| CURRENT_ROLE()|
+--------------+
| `read_orders`|
+--------------+
*/

-- Activer plusieurs rôles spécifiques
SET ROLE read_orders, write_logs;

-- Désactiver tous les rôles
SET ROLE NONE;

SELECT CURRENT_ROLE();
/*
+----------------+
| CURRENT_ROLE() |
+----------------+
| NULL           |
+----------------+
*/
```

### Cas d'usage de l'activation sélective

**Scénario** : Auditeur avec accès normal + accès admin temporaire

```sql
-- Créer rôles
CREATE ROLE auditor_readonly;
CREATE ROLE auditor_admin;

GRANT SELECT ON audit_db.* TO auditor_readonly;
GRANT ALL ON audit_db.* TO auditor_admin;

-- Utilisateur auditeur
CREATE USER 'auditor_alice'@'%' IDENTIFIED BY 'password';
GRANT auditor_readonly, auditor_admin TO 'auditor_alice'@'%';

-- Par défaut: readonly uniquement
SET DEFAULT ROLE auditor_readonly FOR 'auditor_alice'@'%';

-- Utilisation:
-- Connexion normale → auditor_readonly actif (lecture seule)
-- Besoin d'admin temporaire → SET ROLE auditor_admin;
-- Retour à readonly → SET ROLE auditor_readonly;
```

---

## DEFAULT ROLE - Rôles par défaut

Les rôles par défaut sont **automatiquement activés** à la connexion.

### Syntaxe

```sql
SET DEFAULT ROLE {role | ALL | NONE} FOR user;
```

### Configuration de rôle par défaut

```sql
-- Créer utilisateur et rôle
CREATE USER 'developer_bob'@'%' IDENTIFIED BY 'password';
CREATE ROLE developer;
GRANT SELECT, INSERT, UPDATE ON dev_db.* TO developer;
GRANT developer TO 'developer_bob'@'%';

-- Configurer developer comme rôle par défaut
SET DEFAULT ROLE developer FOR 'developer_bob'@'%';

-- Vérification
SELECT User, Host, default_role
FROM mysql.user
WHERE User = 'developer_bob';
/*
+---------------+------+--------------+
| User          | Host | default_role |
+---------------+------+--------------+
| developer_bob | %    | developer    |
+---------------+------+--------------+
*/

-- Test de connexion
mariadb -u developer_bob -p
MariaDB> SELECT CURRENT_ROLE();
/*
+---------------+
| CURRENT_ROLE()|
+---------------+
| `developer`   |
+---------------+
*/
-- ✓ Rôle automatiquement activé à la connexion
```

### Tous les rôles par défaut

```sql
-- Utilisateur avec plusieurs rôles
CREATE USER 'power_user'@'%' IDENTIFIED BY 'password';
GRANT read_orders, write_logs, execute_reports TO 'power_user'@'%';

-- Activer tous les rôles par défaut
SET DEFAULT ROLE ALL FOR 'power_user'@'%';

-- À la connexion, tous les rôles sont actifs
SELECT CURRENT_ROLE();
/*
+-----------------------------------------------+
| CURRENT_ROLE()                                |
+-----------------------------------------------+
| `read_orders`,`write_logs`,`execute_reports`  |
+-----------------------------------------------+
*/
```

### Aucun rôle par défaut

```sql
-- Désactiver les rôles par défaut (activation manuelle requise)
SET DEFAULT ROLE NONE FOR 'developer_bob'@'%';

-- À la connexion:
SELECT CURRENT_ROLE();
-- NULL

-- L'utilisateur doit explicitement activer le rôle
SET ROLE developer;
```

---

## Hiérarchie de rôles (Rôles imbriqués)

MariaDB supporte les **rôles imbriqués** : un rôle peut contenir d'autres rôles.

### Création d'une hiérarchie

```sql
-- Niveau 1: Rôles de base
CREATE ROLE read_data;
CREATE ROLE write_data;
CREATE ROLE manage_schema;

GRANT SELECT ON production.* TO read_data;
GRANT INSERT, UPDATE, DELETE ON production.* TO write_data;
GRANT CREATE, ALTER, DROP ON production.* TO manage_schema;

-- Niveau 2: Rôles composés
CREATE ROLE developer;
CREATE ROLE senior_developer;
CREATE ROLE dba;

-- developer = read + write
GRANT read_data TO developer;
GRANT write_data TO developer;

-- senior_developer = developer + manage_schema
GRANT developer TO senior_developer;
GRANT manage_schema TO senior_developer;

-- dba = senior_developer + privilèges admin
GRANT senior_developer TO dba;
GRANT RELOAD, PROCESS ON *.* TO dba;
```

**Visualisation de la hiérarchie** :

```
dba
 ├─→ senior_developer
 │    ├─→ developer
 │    │    ├─→ read_data (SELECT)
 │    │    └─→ write_data (INSERT, UPDATE, DELETE)
 │    └─→ manage_schema (CREATE, ALTER, DROP)
 └─→ RELOAD, PROCESS (privilèges admin)

Privilèges effectifs de 'dba':
  - SELECT, INSERT, UPDATE, DELETE (via read_data, write_data)
  - CREATE, ALTER, DROP (via manage_schema)
  - RELOAD, PROCESS (directs)
```

### Attribution de rôles hiérarchiques

```sql
-- Développeur junior
CREATE USER 'junior_alice'@'%' IDENTIFIED BY 'password';
GRANT developer TO 'junior_alice'@'%';
SET DEFAULT ROLE developer FOR 'junior_alice'@'%';
-- Privilèges: SELECT, INSERT, UPDATE, DELETE

-- Développeur senior
CREATE USER 'senior_bob'@'%' IDENTIFIED BY 'password';
GRANT senior_developer TO 'senior_bob'@'%';
SET DEFAULT ROLE senior_developer FOR 'senior_bob'@'%';
-- Privilèges: SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, DROP

-- DBA
CREATE USER 'dba_charlie'@'localhost' IDENTIFIED BY 'password';
GRANT dba TO 'dba_charlie'@'localhost';
SET DEFAULT ROLE dba FOR 'dba_charlie'@'localhost';
-- Privilèges: Tous ceux de senior_developer + RELOAD, PROCESS
```

### Vérification des privilèges hérités

```sql
-- Voir tous les privilèges effectifs (y compris hérités)
SHOW GRANTS FOR 'dba_charlie'@'localhost';
/*
GRANT RELOAD, PROCESS ON *.* TO `dba_charlie`@`localhost` ...
GRANT `dba` TO `dba_charlie`@`localhost`
GRANT SELECT ON `production`.* TO `read_data`        -- via dba→senior→developer→read
GRANT INSERT, UPDATE, DELETE ON `production`.* TO `write_data`  -- via dba→senior→developer→write
GRANT CREATE, ALTER, DROP ON `production`.* TO `manage_schema`  -- via dba→senior→manage
*/
```

---

## DROP ROLE - Suppression de rôles

### Syntaxe

```sql
DROP ROLE [IF EXISTS] role_name [, role_name] ...;
```

### Suppression simple

```sql
-- Supprimer un rôle
DROP ROLE developer;

-- Supprimer plusieurs rôles
DROP ROLE read_data, write_data, manage_schema;

-- Supprimer uniquement si existe (pas d'erreur)
DROP ROLE IF EXISTS legacy_role;
```

⚠️ **Attention** : Supprimer un rôle **révoque automatiquement** ce rôle de tous les utilisateurs qui l'avaient.

### Vérification avant suppression

```sql
-- Voir quels utilisateurs ont ce rôle
SELECT GRANTEE
FROM information_schema.APPLICABLE_ROLES
WHERE ROLE_NAME = 'developer';
/*
+-------------------------+
| GRANTEE                 |
+-------------------------+
| 'developer_bob'@'%'     |
| 'junior_alice'@'%'      |
| 'senior_developer'      |
+-------------------------+
*/

-- Voir quels privilèges le rôle possède
SHOW GRANTS FOR developer;

-- Voir si le rôle est utilisé dans d'autres rôles (hiérarchie)
SELECT GRANTEE, IS_GRANTABLE
FROM information_schema.APPLICABLE_ROLES
WHERE ROLE_NAME = 'developer';
```

### Suppression avec migration

```bash
#!/bin/bash
# Script de suppression propre d'un rôle

ROLE_TO_DELETE="legacy_role"
REPLACEMENT_ROLE="modern_role"

# 1. Identifier les utilisateurs affectés
mariadb -u root -p <<EOF
SELECT CONCAT('GRANT ${REPLACEMENT_ROLE} TO ', GRANTEE, ';') AS migration_command
FROM information_schema.APPLICABLE_ROLES
WHERE ROLE_NAME = '${ROLE_TO_DELETE}'
  AND GRANTEE NOT LIKE '%@%'  -- Exclure les utilisateurs (gardez seulement les rôles)
EOF

# 2. Attribuer le nouveau rôle
# (Exécuter les commandes générées ci-dessus)

# 3. Révoquer l'ancien rôle
mariadb -u root -p -e "DROP ROLE ${ROLE_TO_DELETE};"

echo "Rôle ${ROLE_TO_DELETE} supprimé et remplacé par ${REPLACEMENT_ROLE}"
```

---

## REVOKE avec les rôles

### Révoquer un rôle d'un utilisateur

```sql
-- Retirer un rôle d'un utilisateur
REVOKE developer FROM 'developer_bob'@'%';

-- Retirer plusieurs rôles
REVOKE read_orders, write_logs FROM 'multi_role_user'@'%';

-- Vérification
SHOW GRANTS FOR 'developer_bob'@'%';
-- Le rôle developer n'apparaît plus
```

### Révoquer un privilège d'un rôle

```sql
-- Retirer un privilège du rôle
REVOKE DELETE ON production.* FROM developer;

-- Tous les utilisateurs ayant le rôle developer perdent le privilège DELETE
-- (appliqué instantanément)
```

### Révoquer un rôle d'un autre rôle (hiérarchie)

```sql
-- Retirer manage_schema du rôle senior_developer
REVOKE manage_schema FROM senior_developer;

-- senior_developer ne peut plus créer/modifier/supprimer des tables
-- Mais conserve les privilèges via developer (read_data, write_data)
```

---

## Cas d'usage production

### 1. Application web multi-tiers

```sql
-- Architecture: Frontend → API → Database

-- Rôle 1: Frontend (lecture seule publique)
CREATE ROLE frontend_readonly;
GRANT SELECT ON production.products TO frontend_readonly;
GRANT SELECT ON production.categories TO frontend_readonly;

-- Rôle 2: API Backend (CRUD métier)
CREATE ROLE api_backend;
GRANT SELECT, INSERT, UPDATE ON production.orders TO api_backend;
GRANT SELECT, INSERT, UPDATE ON production.customers TO api_backend;
GRANT SELECT ON production.products TO api_backend;

-- Rôle 3: Service de paiement (modification limitée)
CREATE ROLE payment_service;
GRANT SELECT ON production.orders TO payment_service;
GRANT UPDATE(status, payment_date) ON production.orders TO payment_service;
GRANT INSERT ON production.payment_logs TO payment_service;

-- Rôle 4: Analytics (lecture seule complète)
CREATE ROLE analytics_readonly;
GRANT SELECT ON production.* TO analytics_readonly;

-- Attribution aux utilisateurs
CREATE USER 'frontend_app'@'frontend_servers' IDENTIFIED BY 'password';
GRANT frontend_readonly TO 'frontend_app'@'frontend_servers';
SET DEFAULT ROLE frontend_readonly FOR 'frontend_app'@'frontend_servers';

CREATE USER 'api_app'@'api_servers' IDENTIFIED BY 'password';
GRANT api_backend TO 'api_app'@'api_servers';
SET DEFAULT ROLE api_backend FOR 'api_app'@'api_servers';

CREATE USER 'payment_processor'@'payment_server' IDENTIFIED BY 'password';
GRANT payment_service TO 'payment_processor'@'payment_server';
SET DEFAULT ROLE payment_service FOR 'payment_processor'@'payment_server';
```

### 2. Multi-tenancy avec rôles par tenant

```sql
-- Template de rôles pour chaque tenant
CREATE ROLE tenant_admin_template;
CREATE ROLE tenant_user_template;

GRANT ALL ON tenant_db_template.* TO tenant_admin_template;
GRANT SELECT, INSERT, UPDATE ON tenant_db_template.* TO tenant_user_template;

-- Tenant 1
CREATE ROLE tenant_123_admin;
CREATE ROLE tenant_123_user;

GRANT ALL ON tenant_123_db.* TO tenant_123_admin;
GRANT SELECT, INSERT, UPDATE ON tenant_123_db.* TO tenant_123_user;

-- Utilisateurs tenant 123
CREATE USER 'admin_tenant123'@'%' IDENTIFIED BY 'password';
GRANT tenant_123_admin TO 'admin_tenant123'@'%';
SET DEFAULT ROLE tenant_123_admin FOR 'admin_tenant123'@'%';

CREATE USER 'user1_tenant123'@'%' IDENTIFIED BY 'password';
GRANT tenant_123_user TO 'user1_tenant123'@'%';
SET DEFAULT ROLE tenant_123_user FOR 'user1_tenant123'@'%';

-- Tenant 2 (même structure)
CREATE ROLE tenant_456_admin;
CREATE ROLE tenant_456_user;
-- ... etc
```

### 3. Équipe DevOps avec séparation des environnements

```sql
-- Environnement DEV
CREATE ROLE dev_full_access;
GRANT ALL ON dev_database.* TO dev_full_access;

-- Environnement STAGING
CREATE ROLE staging_limited;
GRANT SELECT, INSERT, UPDATE, DELETE ON staging_database.* TO staging_limited;
GRANT CREATE TEMPORARY TABLES ON staging_database.* TO staging_limited;
-- Pas de DROP, ALTER en staging

-- Environnement PRODUCTION (lecture seule pour devs)
CREATE ROLE prod_readonly;
GRANT SELECT ON production.* TO prod_readonly;

-- Développeur standard
CREATE USER 'dev_alice'@'%' IDENTIFIED BY 'password';
GRANT dev_full_access, staging_limited, prod_readonly TO 'dev_alice'@'%';
SET DEFAULT ROLE dev_full_access FOR 'dev_alice'@'%';

-- Alice peut changer de rôle selon son besoin:
-- DEV: SET ROLE dev_full_access;
-- STAGING: SET ROLE staging_limited;
-- PROD (debug): SET ROLE prod_readonly;
```

### 4. Conformité RGPD avec rôles d'accès aux données personnelles

```sql
-- Rôle 1: Accès données anonymisées (analytics)
CREATE ROLE data_analyst;
GRANT SELECT ON analytics.aggregated_data TO data_analyst;
GRANT SELECT ON analytics.anonymized_users TO data_analyst;

-- Rôle 2: Accès données personnelles (support client)
CREATE ROLE customer_support;
GRANT SELECT ON production.customers TO customer_support;
GRANT UPDATE(phone, email, address) ON production.customers TO customer_support;

-- Rôle 3: Accès données sensibles (DPO - Data Protection Officer)
CREATE ROLE data_protection_officer;
GRANT SELECT, UPDATE, DELETE ON production.customers TO data_protection_officer;
GRANT SELECT ON production.audit_log TO data_protection_officer;

-- Rôle 4: Export RGPD (droit d'accès)
CREATE ROLE gdpr_export;
GRANT SELECT ON production.customers TO gdpr_export;
GRANT SELECT ON production.orders TO gdpr_export;
GRANT EXECUTE ON PROCEDURE production.export_customer_data TO gdpr_export;

-- Attribution avec audit
CREATE USER 'dpo_alice'@'%' IDENTIFIED BY 'password';
GRANT data_protection_officer TO 'dpo_alice'@'%';
SET DEFAULT ROLE data_protection_officer FOR 'dpo_alice'@'%';

-- Audit: Toutes les actions du DPO sont loguées
-- (via server_audit_plugin configuré pour ce rôle)
```

### 5. Hiérarchie DBA (Junior → Senior → Principal)

```sql
-- Niveau 1: DBA Monitoring (lecture seule admin)
CREATE ROLE dba_monitoring;
GRANT PROCESS, REPLICATION CLIENT, SHOW DATABASES ON *.* TO dba_monitoring;
GRANT SELECT ON performance_schema.* TO dba_monitoring;
GRANT SELECT ON information_schema.* TO dba_monitoring;

-- Niveau 2: DBA Junior (monitoring + backup + dev DDL)
CREATE ROLE dba_junior;
GRANT dba_monitoring TO dba_junior;
GRANT SELECT, LOCK TABLES, RELOAD ON *.* TO dba_junior;
GRANT ALL ON dev_database.* TO dba_junior;
GRANT ALL ON staging_database.* TO dba_junior;

-- Niveau 3: DBA Senior (junior + prod read + schema changes non-prod)
CREATE ROLE dba_senior;
GRANT dba_junior TO dba_senior;
GRANT SELECT ON production.* TO dba_senior;
GRANT CREATE, ALTER, DROP ON staging_database.* TO dba_senior;

-- Niveau 4: DBA Principal (senior + all prod)
CREATE ROLE dba_principal;
GRANT dba_senior TO dba_principal;
GRANT ALL ON production.* TO dba_principal;

-- Niveau 5: DBA Architect (principal + global admin)
CREATE ROLE dba_architect;
GRANT dba_principal TO dba_architect;
GRANT ALL ON *.* TO dba_architect WITH GRANT OPTION;

-- Attribution
CREATE USER 'dba_intern'@'%' IDENTIFIED BY 'password';
GRANT dba_monitoring TO 'dba_intern'@'%';
SET DEFAULT ROLE dba_monitoring FOR 'dba_intern'@'%';

CREATE USER 'dba_alice'@'localhost' IDENTIFIED BY 'password';
GRANT dba_senior TO 'dba_alice'@'localhost';
SET DEFAULT ROLE dba_senior FOR 'dba_alice'@'localhost';

CREATE USER 'dba_chief'@'localhost' IDENTIFIED BY 'password';
GRANT dba_architect TO 'dba_chief'@'localhost';
SET DEFAULT ROLE dba_architect FOR 'dba_chief'@'localhost';
```

---

## Migration vers les rôles

### Audit de l'existant

```sql
-- 1. Lister tous les utilisateurs et leurs privilèges
SELECT User, Host,
  COUNT(*) AS privilege_count
FROM (
  SELECT GRANTEE, PRIVILEGE_TYPE
  FROM information_schema.USER_PRIVILEGES
  UNION ALL
  SELECT GRANTEE, PRIVILEGE_TYPE
  FROM information_schema.SCHEMA_PRIVILEGES
  UNION ALL
  SELECT GRANTEE, PRIVILEGE_TYPE
  FROM information_schema.TABLE_PRIVILEGES
) AS all_privs
JOIN mysql.user ON CONCAT("'", User, "'@'", Host, "'") = GRANTEE
WHERE User NOT IN ('root', 'mariadb.sys')
GROUP BY User, Host
ORDER BY privilege_count DESC;

-- 2. Identifier les patterns de privilèges similaires
-- (Candidats pour regroupement en rôles)
SELECT GROUP_CONCAT(DISTINCT User) AS similar_users,
  GROUP_CONCAT(DISTINCT PRIVILEGE_TYPE) AS privileges,
  COUNT(DISTINCT User) AS user_count
FROM information_schema.SCHEMA_PRIVILEGES
WHERE TABLE_SCHEMA = 'production'
GROUP BY PRIVILEGE_TYPE
HAVING user_count > 1
ORDER BY user_count DESC;
```

### Stratégie de migration

**Étape 1 : Créer les rôles** (sans impacter l'existant)

```sql
-- Identifier les patterns et créer les rôles correspondants
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;

GRANT SELECT ON production.* TO app_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO app_readwrite;
GRANT ALL ON production.* TO app_admin;
```

**Étape 2 : Attribuer les rôles en parallèle** (doublon temporaire)

```sql
-- Les utilisateurs ont maintenant DEUX sources de privilèges:
-- 1. Privilèges directs (anciens)
-- 2. Rôles (nouveaux)

GRANT app_readwrite TO 'user1'@'%';
GRANT app_readwrite TO 'user2'@'%';
GRANT app_readwrite TO 'user3'@'%';

SET DEFAULT ROLE app_readwrite FOR 'user1'@'%';
SET DEFAULT ROLE app_readwrite FOR 'user2'@'%';
SET DEFAULT ROLE app_readwrite FOR 'user3'@'%';

-- Période de test: vérifier que tout fonctionne
```

**Étape 3 : Révoquer les privilèges directs**

```sql
-- Après validation, supprimer les privilèges directs
REVOKE SELECT, INSERT, UPDATE, DELETE ON production.* FROM 'user1'@'%';
REVOKE SELECT, INSERT, UPDATE, DELETE ON production.* FROM 'user2'@'%';
REVOKE SELECT, INSERT, UPDATE, DELETE ON production.* FROM 'user3'@'%';

-- Maintenant les utilisateurs ont uniquement les privilèges via rôles
```

**Script de migration automatisé** :

```bash
#!/bin/bash
# migrate_to_roles.sh

DB_USER="root"
DB_PASS="password"

# Étape 1: Extraire les privilèges actuels
mariadb -u $DB_USER -p$DB_PASS -N -B <<EOF > /tmp/current_privs.sql
SELECT CONCAT(
  'REVOKE ', PRIVILEGE_TYPE,
  ' ON ', TABLE_SCHEMA, '.', IFNULL(TABLE_NAME, '*'),
  ' FROM ', GRANTEE, ';'
) AS revoke_stmt
FROM information_schema.SCHEMA_PRIVILEGES
WHERE GRANTEE LIKE '%app%'
  AND TABLE_SCHEMA = 'production';
EOF

# Étape 2: Créer les rôles (déjà fait manuellement)

# Étape 3: Attribuer les rôles
mariadb -u $DB_USER -p$DB_PASS <<EOF
GRANT app_readwrite TO 'user1'@'%';
GRANT app_readwrite TO 'user2'@'%';
GRANT app_readwrite TO 'user3'@'%';

SET DEFAULT ROLE app_readwrite FOR 'user1'@'%';
SET DEFAULT ROLE app_readwrite FOR 'user2'@'%';
SET DEFAULT ROLE app_readwrite FOR 'user3'@'%';
EOF

# Étape 4: Période de test (attendre validation)
read -p "Tester l'application. Continuer avec révocation? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
  # Étape 5: Révoquer les privilèges directs
  mariadb -u $DB_USER -p$DB_PASS < /tmp/current_privs.sql
  echo "Migration terminée. Les utilisateurs utilisent maintenant les rôles."
fi
```

---

## Bonnes pratiques RBAC

### 1. Nommage cohérent des rôles

```sql
-- ✅ BON: Noms descriptifs et cohérents
CREATE ROLE app_frontend_readonly;
CREATE ROLE app_backend_readwrite;
CREATE ROLE dba_monitoring_readonly;
CREATE ROLE analyst_warehouse_readonly;

-- ❌ MAUVAIS: Noms ambigus
CREATE ROLE role1;
CREATE ROLE rw;
CREATE ROLE alice_role;
```

**Convention recommandée** :

```
Format: {contexte}_{fonction}_{niveau_accès}

Exemples:
- app_api_readonly
- app_api_readwrite
- dba_backup_limited
- dba_admin_full
- analyst_data_readonly
```

### 2. Granularité appropriée

```sql
-- ✅ BON: Rôles granulaires par fonction
CREATE ROLE read_orders;
CREATE ROLE write_orders;
CREATE ROLE manage_orders;

-- ❌ MAUVAIS: Rôle monolithique
CREATE ROLE all_orders;
GRANT ALL ON orders TO all_orders;  -- Trop large
```

**Règle** : 1 rôle = 1 fonction métier claire

### 3. Hiérarchie logique

```sql
-- ✅ BON: Hiérarchie métier claire
read_only
  └─→ utilisé par: analyst, support_viewer

read_write
  ├─→ read_only (héritage)
  └─→ utilisé par: app_backend, data_engineer

admin
  ├─→ read_write (héritage)
  └─→ utilisé par: team_lead, dba_junior

-- ❌ MAUVAIS: Hiérarchie circulaire ou illogique
GRANT role_a TO role_b;
GRANT role_b TO role_a;  -- ERREUR: Circulaire
```

### 4. Séparation dev/staging/prod

```sql
-- ✅ BON: Rôles distincts par environnement
CREATE ROLE dev_developer;
CREATE ROLE staging_tester;
CREATE ROLE prod_readonly;

GRANT ALL ON dev_db.* TO dev_developer;
GRANT SELECT, INSERT, UPDATE, DELETE ON staging_db.* TO staging_tester;
GRANT SELECT ON prod_db.* TO prod_readonly;

-- Un utilisateur peut avoir les 3 rôles mais activera selon besoin
CREATE USER 'alice'@'%' IDENTIFIED BY 'password';
GRANT dev_developer, staging_tester, prod_readonly TO 'alice'@'%';
SET DEFAULT ROLE dev_developer FOR 'alice'@'%';

-- Alice bascule selon l'environnement:
-- SET ROLE dev_developer;    -- Pour travailler en dev
-- SET ROLE staging_tester;   -- Pour tester en staging
-- SET ROLE prod_readonly;    -- Pour investiguer en prod
```

### 5. Documentation des rôles

```sql
-- Créer une table de documentation
CREATE TABLE role_documentation (
  role_name VARCHAR(128) PRIMARY KEY,
  description TEXT,
  owner VARCHAR(128),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_reviewed TIMESTAMP,
  business_justification TEXT
);

-- Documenter chaque rôle
INSERT INTO role_documentation (role_name, description, owner, business_justification)
VALUES
  ('app_backend', 'CRUD operations for backend API', 'backend-team@example.com',
   'Required for normal application operations'),
  ('dba_monitoring', 'Read-only monitoring access for DBAs', 'dba-team@example.com',
   'Required for 24/7 monitoring and alerting'),
  ('analyst_warehouse', 'Read access to data warehouse', 'analytics-team@example.com',
   'Required for business intelligence reporting');

-- Revue trimestrielle
SELECT role_name, description, owner,
  DATEDIFF(NOW(), last_reviewed) AS days_since_review
FROM role_documentation
WHERE last_reviewed IS NULL
   OR DATEDIFF(NOW(), last_reviewed) > 90
ORDER BY days_since_review DESC;
```

### 6. Principe du moindre privilège avec rôles

```sql
-- ✅ BON: Privilèges minimaux par rôle
CREATE ROLE payment_processor;
GRANT SELECT ON orders TO payment_processor;
GRANT UPDATE(status, payment_date, payment_ref) ON orders TO payment_processor;
GRANT INSERT ON payment_logs TO payment_processor;
-- Seulement ce qui est nécessaire pour traiter les paiements

-- ❌ MAUVAIS: Privilèges excessifs
CREATE ROLE payment_processor;
GRANT ALL ON *.* TO payment_processor;  -- Bien trop large!
```

### 7. Audit régulier des rôles

```bash
#!/bin/bash
# audit_roles.sh - Audit mensuel des rôles

mariadb -u root -p <<'EOF'
-- 1. Rôles sans utilisateurs (potentiellement obsolètes)
SELECT r.User AS unused_role
FROM mysql.user r
WHERE r.is_role = 'Y'
  AND r.User NOT IN (
    SELECT ROLE_NAME
    FROM information_schema.APPLICABLE_ROLES
  );

-- 2. Utilisateurs avec trop de rôles (> 5)
SELECT GRANTEE, COUNT(*) AS role_count
FROM information_schema.APPLICABLE_ROLES
WHERE IS_ROLE = 'NO'
GROUP BY GRANTEE
HAVING role_count > 5
ORDER BY role_count DESC;

-- 3. Rôles avec ALL PRIVILEGES (vérifier légitimité)
SELECT role.User AS role_name,
  'Has ALL PRIVILEGES' AS warning
FROM mysql.user role
WHERE role.is_role = 'Y'
  AND role.Select_priv = 'Y'
  AND role.Insert_priv = 'Y'
  AND role.Update_priv = 'Y'
  AND role.Delete_priv = 'Y'
  AND role.Create_priv = 'Y'
  AND role.Drop_priv = 'Y';

-- 4. Rôles inutilisés depuis 90 jours
-- (Nécessite audit logging activé)
SELECT r.User AS potentially_unused_role
FROM mysql.user r
WHERE r.is_role = 'Y'
  AND r.User NOT IN (
    SELECT DISTINCT role_name
    FROM audit_role_usage  -- Table custom de tracking
    WHERE last_used > DATE_SUB(NOW(), INTERVAL 90 DAY)
  );
EOF
```

---

## Vérification et troubleshooting

### Commandes de diagnostic

```sql
-- 1. Lister tous les rôles
SELECT User AS role_name,
  JSON_EXTRACT(Priv, '$.is_role') AS is_role
FROM mysql.global_priv
WHERE JSON_EXTRACT(Priv, '$.is_role') = 'true';

-- 2. Voir les rôles d'un utilisateur
SELECT ROLE_NAME, IS_DEFAULT
FROM information_schema.APPLICABLE_ROLES
WHERE GRANTEE = "'alice'@'%'";

-- 3. Voir les utilisateurs d'un rôle
SELECT GRANTEE, IS_GRANTABLE
FROM information_schema.APPLICABLE_ROLES
WHERE ROLE_NAME = 'developer';

-- 4. Voir tous les privilèges effectifs (utilisateur + rôles)
-- Connexion en tant qu'utilisateur
SET ROLE ALL;
SHOW GRANTS;

-- 5. Voir la hiérarchie d'un rôle
WITH RECURSIVE role_hierarchy AS (
  SELECT ROLE_NAME, GRANTEE, 0 AS level
  FROM information_schema.APPLICABLE_ROLES
  WHERE ROLE_NAME = 'dba_architect'

  UNION ALL

  SELECT ar.ROLE_NAME, ar.GRANTEE, rh.level + 1
  FROM information_schema.APPLICABLE_ROLES ar
  JOIN role_hierarchy rh ON ar.GRANTEE = rh.ROLE_NAME
  WHERE rh.level < 10  -- Limite pour éviter boucles infinies
)
SELECT CONCAT(REPEAT('  ', level), ROLE_NAME) AS hierarchy
FROM role_hierarchy
ORDER BY level;
```

### Problèmes courants

**Problème 1 : Rôle non actif**

```sql
-- Symptôme
SELECT CURRENT_ROLE();
-- NULL (alors que l'utilisateur a des rôles)

-- Solution
SET ROLE ALL;
-- ou
SET ROLE role_name;

-- Permanent: Configurer rôle par défaut
SET DEFAULT ROLE ALL FOR 'user'@'host';
```

**Problème 2 : Privilèges attendus manquants**

```sql
-- Diagnostic
SHOW GRANTS FOR CURRENT_USER();
-- Voir si le rôle est bien attribué

SELECT CURRENT_ROLE();
-- Vérifier si le rôle est actif

SHOW GRANTS FOR role_name;
-- Vérifier les privilèges du rôle

-- Solution: Activer le rôle
SET ROLE role_name;
```

**Problème 3 : Hiérarchie circulaire**

```sql
-- Symptôme
GRANT role_a TO role_b;
GRANT role_b TO role_a;
-- ERROR 1467 (HY000): Recursive role grants are not allowed

-- Solution: Restructurer la hiérarchie
REVOKE role_a FROM role_b;
-- ou
REVOKE role_b FROM role_a;
```

---

## ✅ Points clés à retenir

- **Les rôles simplifient radicalement la gestion des privilèges** : 1 modification au lieu de N
- **RBAC sépare les fonctions métier des utilisateurs** : rôles = fonctions, utilisateurs = personnes
- **Un utilisateur peut avoir plusieurs rôles** qui s'activent avec SET ROLE
- **Les rôles par défaut s'activent automatiquement** à la connexion (SET DEFAULT ROLE)
- **Les rôles supportent la hiérarchie** : un rôle peut contenir d'autres rôles
- **Les rôles sont stockés dans mysql.user** avec is_role = 'Y' et Host = ''
- **Supprimer un rôle révoque ce rôle de tous les utilisateurs** automatiquement
- **SHOW GRANTS FOR role** affiche les privilèges du rôle, pas les utilisateurs qui l'ont
- **APPLICABLE_ROLES (information_schema)** montre les relations utilisateurs↔rôles
- **Les bonnes pratiques incluent** : nommage cohérent, granularité appropriée, documentation, audit régulier

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 Roles Overview](https://mariadb.com/kb/en/roles/)
- [📖 CREATE ROLE](https://mariadb.com/kb/en/create-role/)
- [📖 GRANT (with roles)](https://mariadb.com/kb/en/grant/#roles)
- [📖 SET ROLE](https://mariadb.com/kb/en/set-role/)
- [📖 SET DEFAULT ROLE](https://mariadb.com/kb/en/set-default-role/)
- [📖 information_schema.APPLICABLE_ROLES](https://mariadb.com/kb/en/information-schema-applicable_roles-table/)

### Standards RBAC

- [NIST RBAC Model](https://csrc.nist.gov/projects/role-based-access-control)
- [ANSI INCITS 359-2004](https://webstore.ansi.org/) - Standard RBAC

---

## ➡️ Section suivante

**10.5 : Plugins d'authentification (mysql_native_password, ed25519, PAM, LDAP, GSSAPI)** - Vous apprendrez à configurer différents mécanismes d'authentification pour s'adapter aux besoins de sécurité de votre organisation.

---


⏭️ [Authentification : Plugins](/10-securite-gestion-utilisateurs/05-authentification-plugins.md)

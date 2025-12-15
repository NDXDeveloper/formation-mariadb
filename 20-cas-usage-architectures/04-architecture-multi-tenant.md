🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.4 Architecture Multi-Tenant

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Chapitre 10 (Sécurité), Section 20.2 (Microservices), notions d'architecture SaaS

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les enjeux spécifiques des architectures multi-tenant
- Distinguer les trois patterns principaux (Database, Schema, Shared) avec leurs trade-offs
- Choisir la stratégie d'isolation adaptée selon les contraintes métier et techniques
- Implémenter une isolation sécurisée des données avec MariaDB 11.8
- Concevoir des mécanismes de quotas et de limitation des ressources par tenant
- Planifier les migrations entre stratégies à mesure que l'application évolue

---

## Introduction

Les applications **SaaS (Software as a Service)** servent plusieurs clients (tenants) depuis une infrastructure partagée. Cette architecture multi-tenant pose un défi fondamental : **comment isoler les données de chaque client tout en mutualisant les ressources pour optimiser les coûts ?**

MariaDB 11.8 LTS offre plusieurs mécanismes pour implémenter le multi-tenant : bases de données dédiées, schémas isolés, ou tables partagées avec discriminateur. Chaque approche présente des compromis entre isolation, coût, complexité et performance.

### Le défi du multi-tenant

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     LE DÉFI DU MULTI-TENANT                                │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Application SaaS                                                          │
│  ═══════════════                                                           │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Application Layer                           │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐         │   │
│  │  │ Tenant A  │  │ Tenant B  │  │ Tenant C  │  │ Tenant D  │         │   │
│  │  │ (Startup) │  │ (PME)     │  │ (Enterprise) │ (Free)    │         │   │
│  │  │ 10 users  │  │ 500 users │  │ 10K users │  │ 5 users   │         │   │
│  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                       │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          Base de données                            │   │
│  │                                                                     │   │
│  │  QUESTIONS CRITIQUES :                                              │   │
│  │  • Comment isoler les données de chaque tenant ?                    │   │
│  │  • Comment éviter qu'un tenant accède aux données d'un autre ?      │   │
│  │  • Comment gérer des tenants avec des volumes très différents ?     │   │
│  │  • Comment facturer / limiter les ressources par tenant ?           │   │
│  │  • Comment migrer un tenant vers une instance dédiée si besoin ?    │   │
│  │  • Comment assurer la compliance (RGPD, SOC2, HIPAA) ?              │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Exigences typiques :                                                      │
│  ════════════════════                                                      │
│  • Isolation des données : Un tenant ne doit JAMAIS voir les données       │
│    d'un autre tenant (sécurité, compliance, confiance)                     │
│  • Performance prévisible : Un "noisy neighbor" ne doit pas impacter       │
│    les autres tenants                                                      │
│  • Scalabilité : De 10 à 100K tenants sans refonte architecturale          │
│  • Coût optimisé : Infrastructure mutualisée quand possible                │
│  • Flexibilité : Options d'isolation différentes selon le tier             │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Pourquoi le multi-tenant est complexe

| Dimension | Défi | Impact |
|-----------|------|--------|
| **Sécurité** | Isolation stricte des données | Une faille expose tous les tenants |
| **Performance** | Noisy neighbor effect | Un tenant peut dégrader les autres |
| **Scalabilité** | Croissance hétérogène | Certains tenants explosent, d'autres stagnent |
| **Opérations** | Maintenance mutualisée | Downtime affecte tous les tenants |
| **Compliance** | Réglementations variées | RGPD, HIPAA, SOC2 selon les tenants |
| **Coût** | Économies d'échelle | Infrastructure partagée = coût réduit |
| **Évolutivité** | Migrations de tier | Upgrade Free → Pro → Enterprise |

---

## Les trois patterns d'isolation

### Vue d'ensemble

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    PATTERNS D'ISOLATION MULTI-TENANT                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. DATABASE PER TENANT (Isolation maximale)                        │   │
│  │  ═══════════════════════════════════════════                        │   │
│  │                                                                     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │   │
│  │  │Tenant A │  │Tenant B │  │Tenant C │  │Tenant D │                 │   │
│  │  │   DB    │  │   DB    │  │   DB    │  │   DB    │                 │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘                 │   │
│  │                                                                     │   │
│  │  ✅ Isolation totale    ❌ Coût élevé (connexions, maintenance)     │   │
│  │  ✅ Performance isolée  ❌ Limité à ~100-1000 tenants               │   │
│  │  ✅ Compliance facile   ❌ Migrations cross-tenant impossibles      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  2. SCHEMA PER TENANT (Isolation logique)                           │   │
│  │  ═════════════════════════════════════════                          │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │                    Database: saas_app                       │    │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │    │   │
│  │  │  │Schema A │  │Schema B │  │Schema C │  │Schema D │         │    │   │
│  │  │  │ users   │  │ users   │  │ users   │  │ users   │         │    │   │
│  │  │  │ orders  │  │ orders  │  │ orders  │  │ orders  │         │    │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │  ✅ Bonne isolation      ❌ Limité à ~1000-5000 schémas             │   │
│  │  ✅ Migrations simples   ❌ Maintenance des schémas (DDL sync)      │   │
│  │  ✅ Coût modéré          ⚠️ Une connexion voit tous les schémas     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  3. SHARED SCHEMA (Tables partagées avec discriminateur)            │   │
│  │  ═══════════════════════════════════════════════════════            │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  Table: users                                               │    │   │
│  │  │  ┌──────────┬────────┬────────────────────────────────────┐ │    │   │
│  │  │  │tenant_id │   id   │  name  │  email  │  ...            │ │    │   │
│  │  │  ├──────────┼────────┼────────┼─────────┼─────────────────┤ │    │   │
│  │  │  │    A     │   1    │  Alice │  a@...  │                 │ │    │   │
│  │  │  │    A     │   2    │  Bob   │  b@...  │                 │ │    │   │
│  │  │  │    B     │   1    │  Carol │  c@...  │                 │ │    │   │
│  │  │  │    C     │   1    │  Dave  │  d@...  │                 │ │    │   │
│  │  │  └──────────┴────────┴────────┴─────────┴─────────────────┘ │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │  ✅ Scalabilité illimitée  ❌ Risque de fuite de données            │   │
│  │  ✅ Coût minimal           ❌ Complexité applicative                │   │
│  │  ✅ Maintenance simple     ❌ Performance dégradée sans index       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Comparaison détaillée

| Critère | Database per Tenant | Schema per Tenant | Shared Schema |
|---------|--------------------|--------------------|---------------|
| **Isolation données** | ✅✅✅ Totale | ✅✅ Logique | ✅ Applicative |
| **Isolation performance** | ✅✅✅ Totale | ✅✅ Partielle | ❌ Aucune |
| **Scalabilité (tenants)** | ~100-1000 | ~1000-5000 | Illimitée |
| **Coût infrastructure** | $$$ | $$ | $ |
| **Coût opérationnel** | $$$ | $$ | $ |
| **Complexité DDL** | Simple (par DB) | Modérée (sync) | Simple (unique) |
| **Compliance** | ✅ Facile | ✅ Possible | ⚠️ Attention |
| **Backup/Restore tenant** | ✅ Natif | ✅ Possible | ⚠️ Complexe |
| **Migration tenant** | ✅✅ Facile | ✅ Modérée | ⚠️ Difficile |
| **Cross-tenant queries** | ❌ Impossible | ⚠️ Possible | ✅ Natif |
| **Connexions DB** | N × pool | 1 pool | 1 pool |

---

## Critères de choix

### Arbre de décision

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARBRE DE DÉCISION MULTI-TENANT                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Combien de tenants prévoyez-vous ?                                         │
│  │                                                                          │
│  ├── < 100 tenants                                                          │
│  │   │                                                                      │
│  │   ├── Compliance stricte (HIPAA, données sensibles) ?                    │
│  │   │   ├── OUI → DATABASE PER TENANT                                      │
│  │   │   └── NON → SCHEMA PER TENANT (bon équilibre)                        │
│  │   │                                                                      │
│  │   └── Tenants avec besoins très hétérogènes ?                            │
│  │       ├── OUI → DATABASE PER TENANT (+ customisation)                    │
│  │       └── NON → SCHEMA PER TENANT                                        │
│  │                                                                          │
│  ├── 100 - 5000 tenants                                                     │
│  │   │                                                                      │
│  │   ├── Budget infrastructure conséquent ?                                 │
│  │   │   ├── OUI → SCHEMA PER TENANT                                        │
│  │   │   └── NON → SHARED SCHEMA + bonnes pratiques sécurité                │
│  │   │                                                                      │
│  │   └── Tenants Enterprise (isolation garantie contractuelle) ?            │
│  │       └── OUI → HYBRID : Enterprise = DB dédiée, autres = Shared         │
│  │                                                                          │
│  └── > 5000 tenants                                                         │
│      │                                                                      │
│      └── SHARED SCHEMA (obligatoire pour le scaling)                        │
│          │                                                                  │
│          ├── Implémenter Row-Level Security rigoureuse                      │
│          ├── Index sur (tenant_id, ...) systématique                        │
│          ├── Vues sécurisées par tenant                                     │
│          └── Option : tenants premium sur DB dédiée                         │
│                                                                             │
│  Facteurs additionnels à considérer :                                       │
│  ════════════════════════════════════                                       │
│  • Réglementations : RGPD (droit à l'effacement), HIPAA, SOC2               │
│  • Contrats : SLA d'isolation garanti à certains clients ?                  │
│  • Équipe : Maturité DevOps pour gérer N bases vs 1 base ?                  │
│  • Migration : Plan d'évolution du modèle dans le temps ?                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Pattern Hybrid (recommandé pour SaaS matures)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE HYBRID MULTI-TENANT                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         Application SaaS                                    │
│                              │                                              │
│              ┌───────────────┴───────────────┐                              │
│              │         Tenant Router         │                              │
│              │   (Détermine la stratégie)    │                              │
│              └───────────────┬───────────────┘                              │
│                              │                                              │
│         ┌────────────────────┼────────────────────┐                         │
│         │                    │                    │                         │
│         ▼                    ▼                    ▼                         │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                  │
│  │ ENTERPRISE  │      │    PRO      │      │    FREE     │                  │
│  │   TIER      │      │   TIER      │      │    TIER     │                  │
│  ├─────────────┤      ├─────────────┤      ├─────────────┤                  │
│  │ Database    │      │ Schema      │      │ Shared      │                  │
│  │ per Tenant  │      │ per Tenant  │      │ Schema      │                  │
│  │             │      │             │      │             │                  │
│  │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │                  │
│  │ │Tenant E1│ │      │ │Schema P1│ │      │ │tenant_id│ │                  │
│  │ └─────────┘ │      │ │Schema P2│ │      │ │   ...   │ │                  │
│  │ ┌─────────┐ │      │ │Schema P3│ │      │ │ (1000+  │ │                  │
│  │ │Tenant E2│ │      │ │   ...   │ │      │ │ tenants)│ │                  │
│  │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │                  │
│  └─────────────┘      └─────────────┘      └─────────────┘                  │
│                                                                             │
│  Caractéristiques par tier :                                                │
│  ═══════════════════════════                                                │
│                                                                             │
│  ENTERPRISE ($500+/mois)     PRO ($50/mois)        FREE ($0)                │
│  • DB dédiée                 • Schema dédié        • Shared schema          │
│  • SLA 99.99%                • SLA 99.9%           • Best effort            │
│  • Backup on-demand          • Backup quotidien    • Backup hebdo           │
│  • Custom config             • Config standard     • Config standard        │
│  • Support dédié             • Support ticket      • Community              │
│  • Compliance certifiée      • Compliance shared   • Pas de garantie        │
│                                                                             │
│  Migration entre tiers : Automatisée via scripts dédiés                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Gestion du routage et du contexte

### Tenant Router Pattern

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- TABLE DE CONFIGURATION DES TENANTS
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE tenant_registry (
    id INT PRIMARY KEY AUTO_INCREMENT,
    tenant_code VARCHAR(50) UNIQUE NOT NULL,
    tenant_name VARCHAR(200) NOT NULL,
    
    -- Stratégie d'isolation
    isolation_strategy ENUM('database', 'schema', 'shared') NOT NULL,
    
    -- Connexion (pour database/schema per tenant)
    db_host VARCHAR(255),
    db_port INT DEFAULT 3306,
    db_name VARCHAR(100),
    db_user VARCHAR(100),
    -- Note: credentials dans un secret manager, pas en base
    
    -- Configuration
    tier ENUM('free', 'starter', 'pro', 'enterprise') DEFAULT 'free',
    status ENUM('active', 'suspended', 'trial', 'deleted') DEFAULT 'trial',
    
    -- Limites (quotas)
    max_users INT,
    max_storage_gb INT,
    max_api_calls_per_day INT,
    
    -- Métadonnées
    settings JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    trial_ends_at TIMESTAMP,
    
    INDEX idx_code (tenant_code),
    INDEX idx_status (status),
    INDEX idx_tier (tier)
) ENGINE=InnoDB;

-- Exemples de tenants
INSERT INTO tenant_registry 
    (tenant_code, tenant_name, isolation_strategy, tier, db_name, max_users, max_storage_gb)
VALUES
    ('acme-corp', 'ACME Corporation', 'database', 'enterprise', 'tenant_acme', NULL, NULL),
    ('startup-xyz', 'Startup XYZ', 'schema', 'pro', 'saas_pro', 500, 100),
    ('small-biz', 'Small Business Inc', 'shared', 'starter', 'saas_shared', 50, 10),
    ('free-user-1', 'Free User 1', 'shared', 'free', 'saas_shared', 5, 1);
```

### Implémentation du contexte tenant

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- GESTION DU CONTEXTE TENANT (SHARED SCHEMA)
-- ═══════════════════════════════════════════════════════════════════════════

-- Variable de session pour le tenant courant
-- Doit être définie au début de chaque requête/transaction

DELIMITER //

-- Procédure pour définir le contexte tenant
CREATE PROCEDURE set_tenant_context(IN p_tenant_id INT)
BEGIN
    -- Vérifier que le tenant existe et est actif
    DECLARE v_status VARCHAR(20);
    
    SELECT status INTO v_status 
    FROM tenants 
    WHERE id = p_tenant_id;
    
    IF v_status IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant not found';
    ELSEIF v_status != 'active' THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant is not active';
    END IF;
    
    -- Définir la variable de session
    SET @current_tenant_id = p_tenant_id;
    
    -- Log de l'accès (optionnel, pour audit)
    INSERT INTO tenant_access_log (tenant_id, access_time, connection_id)
    VALUES (p_tenant_id, NOW(), CONNECTION_ID());
END //

-- Fonction pour récupérer le tenant courant
CREATE FUNCTION get_current_tenant_id()
RETURNS INT
DETERMINISTIC
READS SQL DATA
BEGIN
    IF @current_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant context not set';
    END IF;
    RETURN @current_tenant_id;
END //

-- Procédure de vérification (appelée dans les triggers/vues)
CREATE PROCEDURE verify_tenant_access(IN p_tenant_id INT)
BEGIN
    IF @current_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant context not set';
    ELSEIF @current_tenant_id != p_tenant_id THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Access denied: tenant mismatch';
    END IF;
END //

DELIMITER ;
```

---

## Sécurité et isolation

### Row-Level Security avec vues

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- ROW-LEVEL SECURITY AVEC VUES SÉCURISÉES
-- ═══════════════════════════════════════════════════════════════════════════

-- Table de base (jamais accédée directement par l'application)
CREATE TABLE _users_data (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(200) NOT NULL,
    role VARCHAR(50) DEFAULT 'user',
    status ENUM('active', 'suspended', 'deleted') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Index composite avec tenant_id en premier
    UNIQUE KEY uk_tenant_email (tenant_id, email),
    INDEX idx_tenant_status (tenant_id, status),
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id)
) ENGINE=InnoDB;

-- Vue sécurisée (seule interface pour l'application)
CREATE OR REPLACE 
    SQL SECURITY DEFINER
    VIEW users AS
SELECT 
    id,
    tenant_id,
    email,
    password_hash,
    name,
    role,
    status,
    created_at,
    updated_at
FROM _users_data
WHERE tenant_id = get_current_tenant_id();

-- Vue sans données sensibles (pour API publique)
CREATE OR REPLACE 
    SQL SECURITY DEFINER
    VIEW users_public AS
SELECT 
    id,
    name,
    role,
    status,
    created_at
FROM _users_data
WHERE tenant_id = get_current_tenant_id()
  AND status = 'active';

-- ═══════════════════════════════════════════════════════════════════════════
-- TRIGGERS POUR GARANTIR L'ISOLATION
-- ═══════════════════════════════════════════════════════════════════════════

DELIMITER //

-- Trigger INSERT : Force le tenant_id correct
CREATE TRIGGER trg_users_insert_tenant
BEFORE INSERT ON _users_data
FOR EACH ROW
BEGIN
    IF @current_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant context required for INSERT';
    END IF;
    
    -- Force le tenant_id à la valeur du contexte
    SET NEW.tenant_id = @current_tenant_id;
END //

-- Trigger UPDATE : Empêche la modification du tenant_id
CREATE TRIGGER trg_users_update_tenant
BEFORE UPDATE ON _users_data
FOR EACH ROW
BEGIN
    IF @current_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant context required for UPDATE';
    END IF;
    
    IF OLD.tenant_id != @current_tenant_id THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Access denied: cannot update other tenant data';
    END IF;
    
    -- Empêche le changement de tenant_id
    IF NEW.tenant_id != OLD.tenant_id THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Cannot change tenant_id';
    END IF;
END //

-- Trigger DELETE : Vérifie le tenant
CREATE TRIGGER trg_users_delete_tenant
BEFORE DELETE ON _users_data
FOR EACH ROW
BEGIN
    IF @current_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant context required for DELETE';
    END IF;
    
    IF OLD.tenant_id != @current_tenant_id THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Access denied: cannot delete other tenant data';
    END IF;
END //

DELIMITER ;
```

### Utilisateurs et privilèges MariaDB

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- GESTION DES UTILISATEURS MARIADB POUR MULTI-TENANT
-- ═══════════════════════════════════════════════════════════════════════════

-- Utilisateur applicatif (accès via vues uniquement)
CREATE USER 'app_user'@'%' IDENTIFIED BY 'secure_app_password';

-- Accès aux vues sécurisées uniquement (pas aux tables _*_data)
GRANT SELECT, INSERT, UPDATE, DELETE ON saas_app.users TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON saas_app.projects TO 'app_user'@'%';
GRANT SELECT ON saas_app.users_public TO 'app_user'@'%';

-- Accès aux procédures de contexte
GRANT EXECUTE ON PROCEDURE saas_app.set_tenant_context TO 'app_user'@'%';
GRANT EXECUTE ON FUNCTION saas_app.get_current_tenant_id TO 'app_user'@'%';

-- PAS d'accès aux tables de base
-- DENY implicite sur _users_data, _projects_data, etc.

-- 🆕 MariaDB 11.8 : Privilèges granulaires par colonne
-- Utilisateur analytics (lecture limitée)
CREATE USER 'analytics_user'@'%' IDENTIFIED BY 'secure_analytics_password';
GRANT SELECT (id, tenant_id, role, status, created_at) 
    ON saas_app._users_data TO 'analytics_user'@'%';
-- Note: Pas d'accès aux colonnes sensibles (email, password_hash)

-- Utilisateur admin multi-tenant (support interne)
CREATE USER 'support_admin'@'10.0.%' IDENTIFIED BY 'secure_support_password';
GRANT SELECT ON saas_app.* TO 'support_admin'@'10.0.%';
-- Accès lecture complète mais pas de modification
```

---

## Gestion des quotas et ressources

### Système de quotas

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SYSTÈME DE QUOTAS PAR TENANT
-- ═══════════════════════════════════════════════════════════════════════════

-- Table des quotas et limites
CREATE TABLE tenant_quotas (
    tenant_id INT PRIMARY KEY,
    
    -- Limites
    max_users INT NOT NULL DEFAULT 5,
    max_projects INT NOT NULL DEFAULT 3,
    max_storage_mb BIGINT NOT NULL DEFAULT 1024,  -- 1 GB
    max_api_calls_day INT NOT NULL DEFAULT 1000,
    max_file_size_mb INT NOT NULL DEFAULT 10,
    
    -- Usage actuel (mis à jour par triggers/jobs)
    current_users INT NOT NULL DEFAULT 0,
    current_projects INT NOT NULL DEFAULT 0,
    current_storage_mb BIGINT NOT NULL DEFAULT 0,
    current_api_calls_today INT NOT NULL DEFAULT 0,
    
    -- Timestamps
    api_calls_reset_at DATE NOT NULL DEFAULT (CURRENT_DATE),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id)
) ENGINE=InnoDB;

-- Quotas par tier
CREATE TABLE tier_quotas (
    tier VARCHAR(20) PRIMARY KEY,
    max_users INT NOT NULL,
    max_projects INT NOT NULL,
    max_storage_mb BIGINT NOT NULL,
    max_api_calls_day INT NOT NULL,
    max_file_size_mb INT NOT NULL
) ENGINE=InnoDB;

INSERT INTO tier_quotas VALUES
    ('free', 5, 3, 1024, 1000, 10),
    ('starter', 25, 10, 10240, 10000, 50),
    ('pro', 100, 50, 102400, 100000, 200),
    ('enterprise', NULL, NULL, NULL, NULL, NULL);  -- NULL = illimité

-- ═══════════════════════════════════════════════════════════════════════════
-- VÉRIFICATION DES QUOTAS
-- ═══════════════════════════════════════════════════════════════════════════

DELIMITER //

-- Fonction de vérification générique
CREATE FUNCTION check_quota(
    p_tenant_id INT,
    p_resource_type VARCHAR(50)
) RETURNS BOOLEAN
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_current INT;
    DECLARE v_max INT;
    
    SELECT 
        CASE p_resource_type
            WHEN 'users' THEN current_users
            WHEN 'projects' THEN current_projects
            WHEN 'api_calls' THEN current_api_calls_today
        END,
        CASE p_resource_type
            WHEN 'users' THEN max_users
            WHEN 'projects' THEN max_projects
            WHEN 'api_calls' THEN max_api_calls_day
        END
    INTO v_current, v_max
    FROM tenant_quotas
    WHERE tenant_id = p_tenant_id;
    
    -- NULL = illimité (Enterprise)
    IF v_max IS NULL THEN
        RETURN TRUE;
    END IF;
    
    RETURN v_current < v_max;
END //

-- Trigger pour vérifier le quota users avant INSERT
CREATE TRIGGER trg_check_user_quota
BEFORE INSERT ON _users_data
FOR EACH ROW
BEGIN
    IF NOT check_quota(NEW.tenant_id, 'users') THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'User quota exceeded for this tenant';
    END IF;
END //

-- Trigger pour mettre à jour le compteur après INSERT
CREATE TRIGGER trg_update_user_quota_insert
AFTER INSERT ON _users_data
FOR EACH ROW
BEGIN
    UPDATE tenant_quotas 
    SET current_users = current_users + 1,
        updated_at = NOW()
    WHERE tenant_id = NEW.tenant_id;
END //

-- Trigger pour mettre à jour le compteur après DELETE
CREATE TRIGGER trg_update_user_quota_delete
AFTER DELETE ON _users_data
FOR EACH ROW
BEGIN
    UPDATE tenant_quotas 
    SET current_users = current_users - 1,
        updated_at = NOW()
    WHERE tenant_id = OLD.tenant_id;
END //

DELIMITER ;

-- Job de reset des compteurs API quotidiens (Event)
CREATE EVENT reset_daily_api_quotas
ON SCHEDULE EVERY 1 DAY
STARTS CURRENT_DATE + INTERVAL 1 DAY
DO
    UPDATE tenant_quotas 
    SET current_api_calls_today = 0,
        api_calls_reset_at = CURRENT_DATE
    WHERE api_calls_reset_at < CURRENT_DATE;
```

### Monitoring des ressources par tenant

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- MONITORING MULTI-TENANT
-- ═══════════════════════════════════════════════════════════════════════════

-- Vue de synthèse des quotas
CREATE VIEW v_tenant_quota_status AS
SELECT 
    t.id AS tenant_id,
    t.name AS tenant_name,
    t.tier,
    tq.current_users,
    tq.max_users,
    ROUND(tq.current_users * 100.0 / NULLIF(tq.max_users, 0), 1) AS users_pct,
    tq.current_projects,
    tq.max_projects,
    ROUND(tq.current_projects * 100.0 / NULLIF(tq.max_projects, 0), 1) AS projects_pct,
    tq.current_storage_mb,
    tq.max_storage_mb,
    ROUND(tq.current_storage_mb * 100.0 / NULLIF(tq.max_storage_mb, 0), 1) AS storage_pct,
    tq.current_api_calls_today,
    tq.max_api_calls_day,
    ROUND(tq.current_api_calls_today * 100.0 / NULLIF(tq.max_api_calls_day, 0), 1) AS api_pct,
    CASE 
        WHEN tq.current_users >= tq.max_users 
          OR tq.current_projects >= tq.max_projects
          OR tq.current_storage_mb >= tq.max_storage_mb
          OR tq.current_api_calls_today >= tq.max_api_calls_day
        THEN 'QUOTA_EXCEEDED'
        WHEN tq.current_users >= tq.max_users * 0.9
          OR tq.current_projects >= tq.max_projects * 0.9
          OR tq.current_storage_mb >= tq.max_storage_mb * 0.9
          OR tq.current_api_calls_today >= tq.max_api_calls_day * 0.9
        THEN 'QUOTA_WARNING'
        ELSE 'OK'
    END AS quota_status
FROM tenants t
JOIN tenant_quotas tq ON t.id = tq.tenant_id
WHERE t.status = 'active';

-- Requête pour identifier les tenants proches des limites
SELECT * FROM v_tenant_quota_status
WHERE quota_status IN ('QUOTA_EXCEEDED', 'QUOTA_WARNING')
ORDER BY 
    CASE quota_status WHEN 'QUOTA_EXCEEDED' THEN 1 ELSE 2 END,
    GREATEST(
        COALESCE(users_pct, 0),
        COALESCE(projects_pct, 0),
        COALESCE(storage_pct, 0),
        COALESCE(api_pct, 0)
    ) DESC;

-- Statistiques d'utilisation par tenant
CREATE VIEW v_tenant_usage_stats AS
SELECT 
    t.id AS tenant_id,
    t.name,
    t.tier,
    t.created_at AS tenant_created,
    DATEDIFF(NOW(), t.created_at) AS tenant_age_days,
    
    -- Comptages
    (SELECT COUNT(*) FROM _users_data WHERE tenant_id = t.id) AS user_count,
    (SELECT COUNT(*) FROM _projects_data WHERE tenant_id = t.id) AS project_count,
    
    -- Activité récente
    (SELECT MAX(updated_at) FROM _users_data WHERE tenant_id = t.id) AS last_user_activity,
    (SELECT COUNT(*) FROM _users_data 
     WHERE tenant_id = t.id AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)) AS new_users_30d,
    
    -- Classification
    CASE 
        WHEN (SELECT MAX(updated_at) FROM _users_data WHERE tenant_id = t.id) 
             < DATE_SUB(NOW(), INTERVAL 90 DAY) THEN 'Dormant'
        WHEN (SELECT MAX(updated_at) FROM _users_data WHERE tenant_id = t.id) 
             < DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'At Risk'
        ELSE 'Active'
    END AS engagement_status
FROM tenants t
WHERE t.status = 'active';
```

---

## Performance et optimisation

### Indexation pour multi-tenant

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- STRATÉGIES D'INDEXATION MULTI-TENANT
-- ═══════════════════════════════════════════════════════════════════════════

-- RÈGLE D'OR : tenant_id TOUJOURS en première position des index composites

-- ✅ BON : tenant_id en premier
CREATE INDEX idx_users_tenant_email ON _users_data (tenant_id, email);
CREATE INDEX idx_users_tenant_status ON _users_data (tenant_id, status, created_at);
CREATE INDEX idx_projects_tenant_name ON _projects_data (tenant_id, name);

-- ❌ MAUVAIS : tenant_id pas en premier (ne sera pas utilisé pour filtrer)
-- CREATE INDEX idx_users_email_tenant ON _users_data (email, tenant_id);

-- Index covering pour les requêtes fréquentes
CREATE INDEX idx_users_tenant_covering ON _users_data 
    (tenant_id, status, id, name, email, created_at);

-- Vérifier l'utilisation des index
EXPLAIN SELECT id, name, email 
FROM _users_data 
WHERE tenant_id = 123 AND status = 'active'
ORDER BY created_at DESC
LIMIT 20;
-- Doit montrer "Using index" si l'index covering est utilisé
```

### Partitionnement par tenant

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- PARTITIONNEMENT POUR TRÈS GROS TENANTS (option avancée)
-- ═══════════════════════════════════════════════════════════════════════════

-- Partitionnement par liste de tenant_id (pour isoler les gros tenants)
CREATE TABLE _events_data (
    id BIGINT AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id, tenant_id),
    INDEX idx_tenant_type_date (tenant_id, event_type, created_at)
) ENGINE=InnoDB
PARTITION BY LIST (tenant_id) (
    -- Gros tenants isolés dans leur partition
    PARTITION p_tenant_1 VALUES IN (1),
    PARTITION p_tenant_2 VALUES IN (2),
    PARTITION p_tenant_5 VALUES IN (5),
    -- Autres tenants groupés par range
    PARTITION p_small_1 VALUES IN (3, 4, 6, 7, 8, 9, 10),
    PARTITION p_small_2 VALUES IN (11, 12, 13, 14, 15, 16, 17, 18, 19, 20),
    -- Partition par défaut pour les nouveaux
    PARTITION p_default VALUES IN (DEFAULT)
);

-- Alternative : Partitionnement par hash pour distribution uniforme
CREATE TABLE _logs_data (
    id BIGINT AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    log_level VARCHAR(10),
    message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id, tenant_id),
    INDEX idx_tenant_date (tenant_id, created_at)
) ENGINE=InnoDB
PARTITION BY HASH (tenant_id)
PARTITIONS 16;
```

---

## Opérations et maintenance

### Backup par tenant

```bash
#!/bin/bash
# backup-tenant.sh
# Backup des données d'un tenant spécifique (shared schema)

TENANT_ID=$1
BACKUP_DIR="/backup/tenants/${TENANT_ID}/$(date +%Y%m%d_%H%M%S)"
DB_HOST="localhost"
DB_NAME="saas_shared"
DB_USER="backup_user"

if [ -z "$TENANT_ID" ]; then
    echo "Usage: $0 <tenant_id>"
    exit 1
fi

mkdir -p "$BACKUP_DIR"

# Liste des tables avec tenant_id
TABLES=$(mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" -N -e "
    SELECT TABLE_NAME 
    FROM information_schema.COLUMNS 
    WHERE TABLE_SCHEMA = '$DB_NAME' 
      AND COLUMN_NAME = 'tenant_id'
      AND TABLE_NAME LIKE '\_%\_data'
")

# Export de chaque table filtrée par tenant
for TABLE in $TABLES; do
    echo "Exporting $TABLE for tenant $TENANT_ID..."
    
    mysqldump \
        -h "$DB_HOST" \
        -u "$DB_USER" \
        -p"$DB_PASSWORD" \
        --single-transaction \
        --where="tenant_id = $TENANT_ID" \
        "$DB_NAME" "$TABLE" \
        | gzip > "$BACKUP_DIR/${TABLE}.sql.gz"
done

# Export des métadonnées du tenant
mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" -N -e "
    SELECT * FROM tenants WHERE id = $TENANT_ID
" > "$BACKUP_DIR/tenant_metadata.txt"

echo "Backup completed: $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

### Migration de tenant entre stratégies

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- MIGRATION : Shared Schema → Schema per Tenant (upgrade vers Pro)
-- ═══════════════════════════════════════════════════════════════════════════

DELIMITER //

CREATE PROCEDURE migrate_tenant_to_dedicated_schema(
    IN p_tenant_id INT,
    IN p_new_schema_name VARCHAR(100)
)
BEGIN
    DECLARE v_tenant_code VARCHAR(50);
    
    -- Récupérer les infos du tenant
    SELECT tenant_code INTO v_tenant_code
    FROM tenant_registry WHERE id = p_tenant_id;
    
    -- 1. Créer le nouveau schéma
    SET @sql = CONCAT('CREATE DATABASE IF NOT EXISTS ', p_new_schema_name, 
                      ' CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 2. Créer les tables dans le nouveau schéma
    SET @sql = CONCAT('CREATE TABLE ', p_new_schema_name, '.users LIKE saas_shared._users_data');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- Répéter pour chaque table...
    
    -- 3. Copier les données
    SET @sql = CONCAT(
        'INSERT INTO ', p_new_schema_name, '.users ',
        'SELECT * FROM saas_shared._users_data WHERE tenant_id = ', p_tenant_id
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 4. Mettre à jour le registre des tenants
    UPDATE tenant_registry
    SET isolation_strategy = 'schema',
        db_name = p_new_schema_name,
        updated_at = NOW()
    WHERE id = p_tenant_id;
    
    -- 5. Supprimer les données de l'ancien emplacement (après validation)
    -- DELETE FROM saas_shared._users_data WHERE tenant_id = p_tenant_id;
    -- (À faire manuellement après validation)
    
    SELECT CONCAT('Migration completed for tenant ', v_tenant_code, 
                  ' to schema ', p_new_schema_name) AS result;
END //

DELIMITER ;

-- Exécution
CALL migrate_tenant_to_dedicated_schema(123, 'tenant_acme_pro');
```

---

## ✅ Points clés à retenir

- **Database per Tenant** : Isolation maximale, idéal pour compliance stricte et < 1000 tenants, coût élevé
- **Schema per Tenant** : Bon équilibre isolation/coût, adapté à 1000-5000 tenants, nécessite synchronisation DDL
- **Shared Schema** : Scalabilité illimitée, coût minimal, nécessite Row-Level Security rigoureuse
- **Le pattern Hybrid** est souvent optimal : Enterprise en DB dédiée, Pro en schema, Free en shared
- **tenant_id doit TOUJOURS être en première position** dans les index composites
- **Les vues sécurisées avec SQL SECURITY DEFINER** sont la meilleure protection contre les fuites de données
- **Les triggers garantissent l'isolation** même en cas de bug applicatif
- **Le système de quotas** prévient les abus et permet la facturation
- 🆕 **MariaDB 11.8 privilèges granulaires** permettent de limiter l'accès par colonne pour les tenants

---

## 🔗 Ressources et références

- [📖 Multi-Tenant Data Architecture - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data)
- [📖 SaaS Tenancy Models](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/tenant-isolation.html)
- [📖 MariaDB Row-Level Security](https://mariadb.com/kb/en/sql-security/)
- [📖 MariaDB Partitioning](https://mariadb.com/kb/en/partitioning-overview/)
- [📖 GDPR and Multi-Tenancy](https://gdpr.eu/data-processing/)

---

## ➡️ Sections suivantes

Cette section se décline en trois sous-sections détaillées :

| Section | Titre | Focus |
|---------|-------|-------|
| 20.4.1 | [Database per Tenant](./04.1-database-per-tenant.md) | Implémentation complète, automation, Kubernetes |
| 20.4.2 | [Schema per Tenant](./04.2-schema-per-tenant.md) | Gestion DDL, migrations, performances |
| 20.4.3 | [Shared Schema avec discriminateur](./04.3-shared-schema.md) | Row-Level Security avancée, ORM integration |

⏭️ [Database per tenant](/20-cas-usage-architectures/04.1-database-per-tenant.md)

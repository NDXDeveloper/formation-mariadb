🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.12 Études de Cas Réels

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Chapitres 13-14, Sections 20.1 à 20.11

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Analyser des architectures MariaDB déployées en production
- Comprendre les décisions de conception dans différents contextes métier
- Appliquer les patterns étudiés à vos propres projets
- Anticiper les défis et les solutions dans des scénarios réels
- Évaluer les compromis architecture/performance/coût

---

## Introduction

Cette section présente des études de cas détaillées d'architectures MariaDB déployées en production. Chaque cas illustre l'application pratique des concepts étudiés dans ce chapitre, avec les décisions de conception, les défis rencontrés et les résultats obtenus.

---

## Étude de cas 1 : Plateforme E-commerce Multi-tenant

### Contexte

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ÉTUDE DE CAS : E-COMMERCE MULTI-TENANT                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Entreprise : ShopCloud (SaaS e-commerce B2B)                               │
│  Secteur : Commerce électronique                                            │
│  Taille : 2,500 marchands, 15M produits, 500K commandes/jour                │
│                                                                             │
│  Exigences :                                                                │
│  • Multi-tenant avec isolation des données                                  │
│  • Performance < 100ms P95 pour les pages produits                          │
│  • Disponibilité 99.95%                                                     │
│  • Conformité RGPD (données EU)                                             │
│  • Scalabilité : supporter 10x la croissance sur 3 ans                      │
│  • Analytics temps réel pour chaque marchand                                │
│                                                                             │
│  Budget : Infrastructure ~$50K/mois                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architecture retenue

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE SHOPCLOUD                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                              CDN (Cloudflare)                              │
│                                    │                                       │
│                                    ▼                                       │
│                         ┌─────────────────────┐                            │
│                         │   Load Balancer     │                            │
│                         │   (HAProxy)         │                            │
│                         └──────────┬──────────┘                            │
│                                    │                                       │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         │                          │                          │            │
│         ▼                          ▼                          ▼            │
│  ┌─────────────┐            ┌─────────────┐            ┌─────────────┐     │
│  │   API GW    │            │   API GW    │            │   API GW    │     │
│  │   Zone A    │            │   Zone B    │            │   Zone C    │     │
│  └──────┬──────┘            └──────┬──────┘            └──────┬──────┘     │
│         │                          │                          │            │
│         └──────────────────────────┼──────────────────────────┘            │
│                                    │                                       │
│                                    ▼                                       │
│                         ┌─────────────────────┐                            │
│                         │      MaxScale       │                            │
│                         │   (Router + Cache)  │                            │
│                         └──────────┬──────────┘                            │
│                                    │                                       │
│              ┌─────────────────────┼─────────────────────┐                 │
│              │                     │                     │                 │
│              ▼                     ▼                     ▼                 │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐         │
│  │  Shard 1 (A-H)    │ │  Shard 2 (I-P)    │ │  Shard 3 (Q-Z)    │         │
│  │                   │ │                   │ │                   │         │
│  │  ┌─────────────┐  │ │  ┌─────────────┐  │ │  ┌─────────────┐  │         │
│  │  │   Galera    │  │ │  │   Galera    │  │ │  │   Galera    │  │         │
│  │  │   Cluster   │  │ │  │   Cluster   │  │ │  │   Cluster   │  │         │
│  │  │   (3 nodes) │  │ │  │   (3 nodes) │  │ │  │   (3 nodes) │  │         │
│  │  └─────────────┘  │ │  └─────────────┘  │ │  └─────────────┘  │         │
│  │                   │ │                   │ │                   │         │
│  │  ~850 tenants     │ │  ~850 tenants     │ │  ~800 tenants     │         │
│  │  ~5M products     │ │  ~5M products     │ │  ~5M products     │         │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘         │
│                                    │                                       │
│                                    │ CDC (Debezium)                        │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Apache Kafka                                │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│              ┌───────────────────┼───────────────────┐                     │
│              ▼                   ▼                   ▼                     │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐         │
│  │   ColumnStore     │ │  Elasticsearch    │ │   Redis           │         │
│  │   (Analytics DW)  │ │  (Product Search) │ │   (Cache)         │         │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘         │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Décisions de conception

| Décision | Justification | Alternative rejetée |
|----------|---------------|---------------------|
| **Sharding par tenant_id (hash)** | Distribution uniforme, pas de hotspots | Range-based (risque de déséquilibre) |
| **Galera par shard** | HA sans failover manuel, lectures locales | Async replication (RPO non acceptable) |
| **Schema partagé + tenant_id** | Efficacité opérationnelle, migrations simples | Schema per tenant (explosion des tables) |
| **Row-Level Security** | Isolation forte sans overhead | Application-level only (risque de bugs) |
| **CDC vers ColumnStore** | Analytics sans impact OLTP | ETL batch (délai inacceptable) |
| **MaxScale Query Cache** | Réduction 40% charge DB | Redis cache (complexité invalidation) |

### Implémentation du multi-tenant

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SCHÉMA MULTI-TENANT AVEC ROW-LEVEL SECURITY
-- ═══════════════════════════════════════════════════════════════════════════

-- Table des tenants (marchands)
CREATE TABLE tenants (
    tenant_id INT PRIMARY KEY AUTO_INCREMENT,
    tenant_code VARCHAR(50) UNIQUE NOT NULL,  -- Pour sharding
    company_name VARCHAR(255) NOT NULL,
    plan ENUM('starter', 'pro', 'enterprise') DEFAULT 'starter',
    settings JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_code (tenant_code)
) ENGINE=InnoDB;

-- Table produits avec tenant isolation
CREATE TABLE products (
    product_id BIGINT AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    sku VARCHAR(100) NOT NULL,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    price DECIMAL(12,2) NOT NULL,
    stock_quantity INT DEFAULT 0,
    category_id INT,
    attributes JSON,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (product_id, tenant_id),  -- Tenant dans la PK pour le sharding
    UNIQUE KEY uk_tenant_sku (tenant_id, sku),
    INDEX idx_tenant_category (tenant_id, category_id, is_active),
    INDEX idx_tenant_active (tenant_id, is_active, updated_at),
    FULLTEXT INDEX ft_search (name, description)
) ENGINE=InnoDB;

-- Vue sécurisée avec isolation automatique
CREATE SQL SECURITY DEFINER VIEW v_products AS
SELECT * FROM products
WHERE tenant_id = GET_CURRENT_TENANT_ID();

-- Fonction pour récupérer le tenant courant (stocké en session)
DELIMITER //
CREATE FUNCTION GET_CURRENT_TENANT_ID()
RETURNS INT
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_tenant_id INT;
    SELECT @current_tenant_id INTO v_tenant_id;
    IF v_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Tenant context not set';
    END IF;
    RETURN v_tenant_id;
END //
DELIMITER ;

-- Procédure pour définir le contexte tenant
DELIMITER //
CREATE PROCEDURE set_tenant_context(IN p_tenant_code VARCHAR(50))
BEGIN
    DECLARE v_tenant_id INT;
    
    SELECT tenant_id INTO v_tenant_id
    FROM tenants WHERE tenant_code = p_tenant_code;
    
    IF v_tenant_id IS NULL THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid tenant';
    END IF;
    
    SET @current_tenant_id = v_tenant_id;
END //
DELIMITER ;

-- ═══════════════════════════════════════════════════════════════════════════
-- CONFIGURATION MAXSCALE POUR ROUTING PAR TENANT
-- ═══════════════════════════════════════════════════════════════════════════

-- Dans MaxScale, routing basé sur le hash du tenant_code
-- Extrait du fichier maxscale.cnf:
/*
[ShardRouter]
type=service
router=schemarouter
servers=shard1,shard2,shard3
user=maxscale_user

[TenantFilter]
type=filter
module=regexfilter
match=tenant_id\s*=\s*(\d+)
replace=/* shard:$1 */

# Mapping tenant -> shard
# tenant_id % 3 = 0 -> shard1
# tenant_id % 3 = 1 -> shard2
# tenant_id % 3 = 2 -> shard3
*/
```

### Résultats

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RÉSULTATS APRÈS 18 MOIS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Métriques de performance :                                                 │
│  ═════════════════════════                                                  │
│  • Latence P50 : 23ms (objectif : <50ms) ✅                                 │
│  • Latence P95 : 67ms (objectif : <100ms) ✅                                │
│  • Latence P99 : 142ms                                                      │
│  • Throughput : 12,000 requêtes/seconde soutenu                             │
│  • Disponibilité : 99.97% (objectif : 99.95%) ✅                            │
│                                                                             │
│  Scalabilité :                                                              │
│  ═════════════                                                              │
│  • Tenants : 2,500 → 4,200 (+68%)                                           │
│  • Produits : 15M → 28M (+87%)                                              │
│  • Commandes/jour : 500K → 1.2M (+140%)                                     │
│  • Sans ajout de shard (headroom restant : ~40%)                            │
│                                                                             │
│  Coûts :                                                                    │
│  ═══════                                                                    │
│  • Infrastructure : $47K/mois (sous budget)                                 │
│  • Coût par tenant : $11.20/mois (-35% vs ancienne architecture)            │
│  • ROI de la migration : 8 mois                                             │
│                                                                             │
│  Leçons apprises :                                                          │
│  ═════════════════                                                          │
│  ✅ Le sharding par hash évite les hotspots mais complique les queries      │
│     cross-tenant (analytics résolues via ColumnStore)                       │
│  ✅ MaxScale Query Cache très efficace pour le catalogue produits           │
│     (hit ratio 72%)                                                         │
│  ⚠️  La migration de schéma requiert coordination sur 9 nodes Galera        │
│     → Adopté pt-online-schema-change                                        │
│  ⚠️  CDC lag occasionnel sous forte charge → augmenté les workers           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Étude de cas 2 : Fintech - Système de paiement temps réel

### Contexte

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ÉTUDE DE CAS : FINTECH PAIEMENT                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Entreprise : PayFlow (PSP - Payment Service Provider)                      │
│  Secteur : Services financiers                                              │
│  Volume : 2M transactions/jour, pics à 800 TPS                              │
│                                                                             │
│  Exigences réglementaires :                                                 │
│  • PCI-DSS Level 1                                                          │
│  • Conformité PSD2 (Europe)                                                 │
│  • RGPD                                                                     │
│  • Audit trail complet (7 ans de rétention)                                 │
│                                                                             │
│  Exigences techniques :                                                     │
│  • Latence < 200ms pour 99% des transactions                                │
│  • Zéro perte de données (RPO = 0)                                          │
│  • RTO < 30 secondes                                                        │
│  • Disponibilité 99.999% (5 minutes downtime/an max)                        │
│  • Multi-région (EU + backup US)                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architecture retenue

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE PAYFLOW                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  RÉGION EU (PRIMARY)                          RÉGION US (DR)               │
│  ══════════════════                           ══════════════               │
│                                                                            │
│  ┌─────────────────────────────┐    Async    ┌─────────────────────────┐   │
│  │     Global Load Balancer    │◄───────────►│   Global Load Balancer  │   │
│  └──────────────┬──────────────┘    Repl     └───────────┬─────────────┘   │
│                 │                                        │                 │
│  ┌──────────────┼──────────────┐             ┌───────────┼──────────────┐  │
│  │              │              │             │           │              │  │
│  │  ┌───────────▼───────────┐  │             │  ┌────────▼───────────┐  │  │
│  │  │      MaxScale HA      │  │             │  │    MaxScale HA     │  │  │
│  │  │   (Active-Standby)    │  │             │  │   (Standby)        │  │  │
│  │  └───────────┬───────────┘  │             │  └────────┬───────────┘  │  │
│  │              │              │             │           │              │  │
│  │  ┌───────────▼───────────┐  │             │  ┌────────▼───────────┐  │  │
│  │  │    Galera Cluster     │  │             │  │   Galera Cluster   │  │  │
│  │  │      (5 nodes)        │──┼─────────────┼──│     (3 nodes)      │  │  │
│  │  │                       │  │   Async     │  │                    │  │  │
│  │  │  ┌─────┐ ┌─────┐      │  │   GTID      │  │  ┌─────┐ ┌─────┐   │  │  │
│  │  │  │ N1  │ │ N2  │      │  │   Repl      │  │  │ N1  │ │ N2  │   │  │  │
│  │  │  │ DC1 │ │ DC1 │      │  │             │  │  │ DC3 │ │ DC3 │   │  │  │
│  │  │  └─────┘ └─────┘      │  │             │  │  └─────┘ └─────┘   │  │  │
│  │  │  ┌─────┐ ┌─────┐      │  │             │  │  ┌─────┐           │  │  │
│  │  │  │ N3  │ │ N4  │      │  │             │  │  │ N3  │           │  │  │
│  │  │  │ DC2 │ │ DC2 │      │  │             │  │  │ DC3 │           │  │  │
│  │  │  └─────┘ └─────┘      │  │             │  │  └─────┘           │  │  │
│  │  │  ┌─────┐              │  │             │  │                    │  │  │
│  │  │  │ N5  │  Arbitrator  │  │             │  │                    │  │  │
│  │  │  │ DC2 │              │  │             │  │                    │  │  │
│  │  │  └─────┘              │  │             │  │                    │  │  │
│  │  └───────────────────────┘  │             │  └────────────────────┘  │  │
│  │                             │             │                          │  │
│  │  Encryption at rest:        │             │                          │  │
│  │  • AWS KMS + CloudHSM       │             │                          │  │
│  │  • TDE (Transparent Data    │             │                          │  │
│  │    Encryption)              │             │                          │  │
│  │                             │             │                          │  │
│  └─────────────────────────────┘             └──────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Event Bus (Kafka)                           │   │
│  │                                                                     │   │
│  │  Topics:                                                            │   │
│  │  • transactions.initiated (Outbox pattern)                          │   │
│  │  • transactions.completed                                           │   │
│  │  • transactions.failed                                              │   │
│  │  • audit.events (immutable log)                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                │                                                           │
│                ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Audit & Compliance                               │   │
│  │                                                                     │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │   │
│  │  │   ColumnStore   │  │   S3 Glacier    │  │   Elasticsearch │      │   │
│  │  │   (Analytics)   │  │   (Archive 7y)  │  │   (Search)      │      │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Décisions de conception critiques

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DÉCISIONS DE CONCEPTION                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. GALERA 5 NODES (pas 3)                                                  │
│  ═════════════════════════                                                  │
│  Justification:                                                             │
│  • Tolérance à 2 pannes simultanées (quorum = 3)                            │
│  • Split-brain impossible avec 5 nodes                                      │
│  • Maintenance rolling sans impact disponibilité                            │
│                                                                             │
│  Coût additionnel: +$8K/mois                                                │
│  Bénéfice: 99.999% vs 99.99% disponibilité                                  │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  2. RÉPLICATION SYNCHRONE INTRA-RÉGION, ASYNC INTER-RÉGION                  │
│  ════════════════════════════════════════════════════════════               │
│  Justification:                                                             │
│  • Synchrone EU: RPO=0 pour les transactions (PCI-DSS)                      │
│  • Async US: Latence EU-US (~80ms) incompatible avec synchrone              │
│  • RPO inter-région: ~2 secondes (acceptable pour DR)                       │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  3. OUTBOX PATTERN POUR LES ÉVÉNEMENTS                                      │
│  ═════════════════════════════════════                                      │
│  Justification:                                                             │
│  • Cohérence transaction + événement garantie                               │
│  • Pas de dual-write (risque d'incohérence)                                 │
│  • Idempotence native (replay possible)                                     │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  4. ENCRYPTION EVERYWHERE                                                   │
│  ════════════════════════                                                   │
│  Implémentation:                                                            │
│  • TLS 1.3 in-transit (obligatoire)                                         │
│  • TDE at-rest (InnoDB tablespace encryption)                               │
│  • Column-level encryption pour PAN/CVV (HSM)                               │
│  • Key rotation automatique (90 jours)                                      │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  5. IMMUTABLE AUDIT LOG                                                     │
│  ════════════════════════                                                   │
│  Implémentation:                                                            │
│  • Table audit_log avec triggers AFTER INSERT/UPDATE/DELETE                 │
│  • Pas de UPDATE/DELETE autorisé sur audit_log                              │
│  • Hash chaîné pour intégrité (blockchain-like)                             │
│  • Archivage vers S3 Glacier après 1 an                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Schéma critique

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SCHÉMA TRANSACTIONNEL PAYFLOW
-- ═══════════════════════════════════════════════════════════════════════════

-- Table des transactions (cœur du système)
CREATE TABLE transactions (
    transaction_id BINARY(16) PRIMARY KEY,  -- UUID v7 (time-ordered)
    
    -- Identification
    merchant_id INT NOT NULL,
    external_reference VARCHAR(100),
    
    -- Montant
    amount DECIMAL(15,2) NOT NULL,
    currency CHAR(3) NOT NULL,
    
    -- Type et statut
    transaction_type ENUM('payment', 'refund', 'chargeback', 'payout') NOT NULL,
    status ENUM('initiated', 'pending', 'authorized', 'captured', 
                'settled', 'failed', 'cancelled', 'refunded') NOT NULL,
    
    -- Données sensibles (encrypted at column level)
    card_token VARBINARY(512),  -- Tokenized, not actual PAN
    
    -- Métadonnées
    metadata JSON,
    
    -- Timestamps précis
    initiated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    completed_at TIMESTAMP(6),
    
    -- Versioning optimiste
    version INT DEFAULT 1,
    
    -- Index
    INDEX idx_merchant_date (merchant_id, initiated_at),
    INDEX idx_status_date (status, initiated_at),
    INDEX idx_external_ref (merchant_id, external_reference)
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED
  ENCRYPTION='Y';

-- Table Outbox pour les événements (pattern Outbox)
CREATE TABLE transaction_outbox (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    aggregate_id BINARY(16) NOT NULL,  -- transaction_id
    event_type VARCHAR(100) NOT NULL,
    payload JSON NOT NULL,
    
    created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
    processed_at TIMESTAMP(6),
    
    INDEX idx_unprocessed (processed_at, created_at)
) ENGINE=InnoDB;

-- Audit log immuable
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    -- Données de l'événement
    table_name VARCHAR(100) NOT NULL,
    record_id VARCHAR(100) NOT NULL,
    action ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    old_values JSON,
    new_values JSON,
    
    -- Contexte
    user_id VARCHAR(100),
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    
    -- Timestamp et intégrité
    created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
    previous_hash CHAR(64),
    current_hash CHAR(64) AS (
        SHA2(CONCAT(
            COALESCE(previous_hash, 'GENESIS'),
            table_name,
            record_id,
            action,
            COALESCE(old_values, ''),
            COALESCE(new_values, ''),
            created_at
        ), 256)
    ) STORED,
    
    INDEX idx_table_record (table_name, record_id, created_at),
    INDEX idx_created (created_at)
) ENGINE=InnoDB
  ENCRYPTION='Y';

-- Trigger pour audit automatique
DELIMITER //

CREATE TRIGGER trg_transactions_audit_insert
AFTER INSERT ON transactions
FOR EACH ROW
BEGIN
    DECLARE v_prev_hash CHAR(64);
    
    SELECT current_hash INTO v_prev_hash 
    FROM audit_log ORDER BY id DESC LIMIT 1;
    
    INSERT INTO audit_log (table_name, record_id, action, new_values, 
                          user_id, previous_hash)
    VALUES (
        'transactions',
        HEX(NEW.transaction_id),
        'INSERT',
        JSON_OBJECT(
            'merchant_id', NEW.merchant_id,
            'amount', NEW.amount,
            'currency', NEW.currency,
            'status', NEW.status
        ),
        @current_user_id,
        v_prev_hash
    );
END //

CREATE TRIGGER trg_transactions_audit_update
AFTER UPDATE ON transactions
FOR EACH ROW
BEGIN
    DECLARE v_prev_hash CHAR(64);
    
    SELECT current_hash INTO v_prev_hash 
    FROM audit_log ORDER BY id DESC LIMIT 1;
    
    INSERT INTO audit_log (table_name, record_id, action, old_values, 
                          new_values, user_id, previous_hash)
    VALUES (
        'transactions',
        HEX(NEW.transaction_id),
        'UPDATE',
        JSON_OBJECT('status', OLD.status, 'version', OLD.version),
        JSON_OBJECT('status', NEW.status, 'version', NEW.version),
        @current_user_id,
        v_prev_hash
    );
END //

DELIMITER ;

-- Procédure de traitement transactionnel
DELIMITER //

CREATE PROCEDURE process_payment(
    IN p_transaction_id BINARY(16),
    IN p_merchant_id INT,
    IN p_amount DECIMAL(15,2),
    IN p_currency CHAR(3),
    IN p_card_token VARBINARY(512)
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- 1. Créer la transaction
    INSERT INTO transactions 
        (transaction_id, merchant_id, amount, currency, 
         transaction_type, status, card_token)
    VALUES 
        (p_transaction_id, p_merchant_id, p_amount, p_currency,
         'payment', 'initiated', p_card_token);
    
    -- 2. Créer l'événement outbox (sera capturé par CDC)
    INSERT INTO transaction_outbox (aggregate_id, event_type, payload)
    VALUES (
        p_transaction_id,
        'PaymentInitiated',
        JSON_OBJECT(
            'transaction_id', HEX(p_transaction_id),
            'merchant_id', p_merchant_id,
            'amount', p_amount,
            'currency', p_currency,
            'timestamp', UNIX_TIMESTAMP(NOW(6)) * 1000
        )
    );
    
    COMMIT;
END //

DELIMITER ;
```

### Résultats

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RÉSULTATS APRÈS 24 MOIS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Conformité :                                                               │
│  ════════════                                                               │
│  • Certification PCI-DSS Level 1 obtenue ✅                                 │
│  • Audit RGPD passé sans observation majeure ✅                             │
│  • Agrément PSD2 maintenu ✅                                                │
│                                                                             │
│  Performance :                                                              │
│  ═════════════                                                              │
│  • Latence P50 : 45ms                                                       │
│  • Latence P99 : 167ms (objectif : <200ms) ✅                               │
│  • Throughput max : 1,200 TPS (objectif : 800 TPS) ✅                       │
│                                                                             │
│  Disponibilité :                                                            │
│  ═══════════════                                                            │
│  • Uptime : 99.9985% (7.9 minutes downtime sur 24 mois)                     │
│  • Objectif 99.999% quasi-atteint                                           │
│  • 0 perte de données (RPO = 0) ✅                                          │
│                                                                             │
│  Incidents majeurs :                                                        │
│  ═══════════════════                                                        │
│  • 1 failover inter-datacenter (3.2 minutes) - panne réseau DC1             │
│  • 1 failover US activé en test DR (4.7 minutes)                            │
│  • 0 incident de sécurité                                                   │
│                                                                             │
│  Volume traité :                                                            │
│  ═══════════════                                                            │
│  • 1.5 milliards de transactions sur 24 mois                                │
│  • €47 milliards de volume de paiement                                      │
│  • Croissance : +85% YoY                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Étude de cas 3 : Média - Plateforme de contenu avec IA

### Contexte

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ÉTUDE DE CAS : MÉDIA & IA                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Entreprise : NewsAI (Agrégateur de news avec IA)                           │
│  Secteur : Média digital                                                    │
│  Volume : 500K articles, 2M utilisateurs, 50M requêtes/jour                 │
│                                                                             │
│  Fonctionnalités IA :                                                       │
│  • Recherche sémantique dans les articles                                   │
│  • Recommandations personnalisées                                           │
│  • Résumé automatique (RAG)                                                 │
│  • Chatbot Q&A sur l'actualité                                              │
│  • Détection de fake news                                                   │
│                                                                             │
│  Exigences :                                                                │
│  • Latence recherche < 500ms                                                │
│  • Recommandations temps réel                                               │
│  • Indexation des nouveaux articles < 5 minutes                             │
│  • Coût embedding maîtrisé                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architecture retenue

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE NEWSAI                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                            ┌─────────────────┐                             │
│                            │   Users / Apps  │                             │
│                            └────────┬────────┘                             │
│                                     │                                      │
│                            ┌────────▼────────┐                             │
│                            │   API Gateway   │                             │
│                            │   (FastAPI)     │                             │
│                            └────────┬────────┘                             │
│                                     │                                      │
│         ┌───────────────────────────┼───────────────────────────┐          │
│         │                           │                           │          │
│         ▼                           ▼                           ▼          │
│  ┌─────────────┐            ┌─────────────┐            ┌─────────────┐     │
│  │   Search    │            │   RAG /     │            │   Reco      │     │
│  │   Service   │            │   Chatbot   │            │   Engine    │     │
│  └──────┬──────┘            └──────┬──────┘            └──────┬──────┘     │
│         │                          │                          │            │
│         │                          │                          │            │
│         └──────────────────────────┼──────────────────────────┘            │
│                                    │                                       │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         LangChain / LlamaIndex                      │   │
│  │                                                                     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│  │  │  Embedding  │  │    LLM      │  │  Retriever  │                  │   │
│  │  │  (Cached)   │  │  (Claude)   │  │  (Hybrid)   │                  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                  │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│              ┌───────────────────┼───────────────────┐                     │
│              │                   │                   │                     │
│              ▼                   ▼                   ▼                     │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐         │
│  │     MariaDB       │ │      Redis        │ │    Embedding      │         │
│  │   Vector Store    │ │   (Cache + MQ)    │ │    Service        │         │
│  │                   │ │                   │ │                   │         │
│  │  ┌─────────────┐  │ │  - Query cache    │ │  Local model ou   │         │ 
│  │  │  articles   │  │ │  - Embedding cache│ │  OpenAI API       │         │
│  │  │  500K rows  │  │ │  - Session store  │ │                   │         │
│  │  └─────────────┘  │ │  - Rate limiting  │ │                   │         │
│  │  ┌─────────────┐  │ │                   │ │                   │         │
│  │  │  embeddings │  │ └───────────────────┘ └───────────────────┘         │
│  │  │  500K × 1536│  │                                                     │
│  │  └─────────────┘  │                                                     │
│  │  ┌─────────────┐  │                                                     │
│  │  │user_profiles│  │                                                     │
│  │  │  2M rows    │  │                                                     │
│  │  └─────────────┘  │                                                     │
│  │  ┌─────────────┐  │                                                     │
│  │  │interactions │  │                                                     │
│  │  │  100M rows  │  │                                                     │
│  │  └─────────────┘  │                                                     │
│  │                   │                                                     │
│  │  FULLTEXT +       │                                                     │
│  │  Vector Search    │                                                     │
│  └───────────────────┘                                                     │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Ingestion Pipeline                               │   │
│  │                                                                     │   │
│  │  RSS/API ──► Parser ──► Embedding ──► MariaDB ──► Index Update      │   │
│  │             │          (batch)       (INSERT)                       │   │
│  │             │                                                       │   │
│  │             └──► Fake News Detection ──► Flag                       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Implémentation du système de recommandation

```python
#!/usr/bin/env python3
"""
newsai_recommendation.py
Système de recommandation hybride pour NewsAI
"""

import numpy as np
import mariadb
from typing import List, Dict, Tuple, Optional
from dataclasses import dataclass
import json
import redis
from datetime import datetime, timedelta
import logging

logger = logging.getLogger(__name__)


@dataclass
class Article:
    id: int
    title: str
    content: str
    category: str
    published_at: datetime
    embedding: Optional[np.ndarray] = None


@dataclass
class Recommendation:
    article: Article
    score: float
    reason: str  # 'similar_content', 'trending', 'collaborative', 'personalized'


class NewsAIRecommendationEngine:
    """
    Moteur de recommandation hybride combinant :
    - Content-based (similarité vectorielle)
    - Collaborative filtering (comportement utilisateurs similaires)
    - Trending (popularité récente)
    - Personalization (profil utilisateur)
    """
    
    def __init__(self, db_config: Dict, redis_config: Dict):
        self.db_pool = mariadb.ConnectionPool(
            pool_name="reco_pool",
            pool_size=10,
            **db_config
        )
        self.redis = redis.Redis(**redis_config)
        
        # Poids pour le score hybride
        self.weights = {
            'content': 0.35,
            'collaborative': 0.25,
            'trending': 0.20,
            'personalized': 0.20
        }
    
    def get_recommendations(
        self,
        user_id: int,
        k: int = 20,
        exclude_read: bool = True,
        category_filter: Optional[str] = None
    ) -> List[Recommendation]:
        """
        Génère des recommandations personnalisées pour un utilisateur.
        """
        
        # 1. Récupérer le profil utilisateur
        user_profile = self._get_user_profile(user_id)
        
        # 2. Articles déjà lus (à exclure)
        read_articles = self._get_read_articles(user_id) if exclude_read else set()
        
        # 3. Candidats de chaque source
        content_candidates = self._get_content_based_candidates(
            user_profile, k * 2, read_articles, category_filter
        )
        
        collaborative_candidates = self._get_collaborative_candidates(
            user_id, k * 2, read_articles
        )
        
        trending_candidates = self._get_trending_candidates(
            k * 2, read_articles, category_filter
        )
        
        # 4. Fusion et scoring
        all_candidates = {}
        
        for article, score in content_candidates:
            if article.id not in all_candidates:
                all_candidates[article.id] = {
                    'article': article, 
                    'scores': {'content': 0, 'collaborative': 0, 'trending': 0, 'personalized': 0}
                }
            all_candidates[article.id]['scores']['content'] = score
        
        for article, score in collaborative_candidates:
            if article.id not in all_candidates:
                all_candidates[article.id] = {
                    'article': article,
                    'scores': {'content': 0, 'collaborative': 0, 'trending': 0, 'personalized': 0}
                }
            all_candidates[article.id]['scores']['collaborative'] = score
        
        for article, score in trending_candidates:
            if article.id not in all_candidates:
                all_candidates[article.id] = {
                    'article': article,
                    'scores': {'content': 0, 'collaborative': 0, 'trending': 0, 'personalized': 0}
                }
            all_candidates[article.id]['scores']['trending'] = score
        
        # 5. Calcul du score personnalisé
        for article_id, data in all_candidates.items():
            data['scores']['personalized'] = self._compute_personalization_score(
                data['article'], user_profile
            )
        
        # 6. Score final pondéré
        recommendations = []
        for article_id, data in all_candidates.items():
            final_score = sum(
                self.weights[source] * score 
                for source, score in data['scores'].items()
            )
            
            # Déterminer la raison principale
            main_reason = max(data['scores'].items(), key=lambda x: x[1])[0]
            
            recommendations.append(Recommendation(
                article=data['article'],
                score=final_score,
                reason=main_reason
            ))
        
        # 7. Tri et diversification
        recommendations.sort(key=lambda x: x.score, reverse=True)
        recommendations = self._diversify(recommendations, k)
        
        return recommendations
    
    def _get_user_profile(self, user_id: int) -> Dict:
        """Récupère ou calcule le profil utilisateur."""
        
        # Check cache
        cache_key = f"user_profile:{user_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        conn = self.db_pool.get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            # Embedding moyen des articles lus
            cursor.execute("""
                SELECT e.embedding
                FROM user_interactions ui
                JOIN embeddings e ON ui.article_id = e.article_id
                WHERE ui.user_id = %s
                  AND ui.interaction_type IN ('read', 'like', 'share')
                  AND ui.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
                ORDER BY 
                    CASE ui.interaction_type 
                        WHEN 'share' THEN 3
                        WHEN 'like' THEN 2
                        ELSE 1 
                    END DESC
                LIMIT 50
            """, (user_id,))
            
            embeddings = []
            weights = []
            for i, row in enumerate(cursor.fetchall()):
                emb = np.frombuffer(row['embedding'], dtype=np.float32)
                embeddings.append(emb)
                weights.append(1.0 / (i + 1))  # Décroissance temporelle
            
            if embeddings:
                weighted_avg = np.average(embeddings, axis=0, weights=weights)
                preference_embedding = weighted_avg / np.linalg.norm(weighted_avg)
            else:
                preference_embedding = None
            
            # Catégories préférées
            cursor.execute("""
                SELECT a.category, COUNT(*) as cnt
                FROM user_interactions ui
                JOIN articles a ON ui.article_id = a.id
                WHERE ui.user_id = %s
                  AND ui.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
                GROUP BY a.category
                ORDER BY cnt DESC
            """, (user_id,))
            
            category_prefs = {row['category']: row['cnt'] for row in cursor.fetchall()}
            
            profile = {
                'embedding': preference_embedding.tolist() if preference_embedding is not None else None,
                'categories': category_prefs,
                'computed_at': datetime.now().isoformat()
            }
            
            # Cache for 1 hour
            self.redis.setex(cache_key, 3600, json.dumps(profile))
            
            return profile
            
        finally:
            cursor.close()
            conn.close()
    
    def _get_content_based_candidates(
        self,
        user_profile: Dict,
        k: int,
        exclude: set,
        category: Optional[str]
    ) -> List[Tuple[Article, float]]:
        """Candidats basés sur la similarité de contenu."""
        
        if not user_profile.get('embedding'):
            return []
        
        query_embedding = np.array(user_profile['embedding'], dtype=np.float32)
        query_norm = query_embedding / np.linalg.norm(query_embedding)
        
        conn = self.db_pool.get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            where_clause = "a.published_at > DATE_SUB(NOW(), INTERVAL 7 DAY)"
            params = []
            
            if category:
                where_clause += " AND a.category = %s"
                params.append(category)
            
            cursor.execute(f"""
                SELECT a.id, a.title, a.content, a.category, a.published_at, e.embedding
                FROM articles a
                JOIN embeddings e ON a.id = e.article_id
                WHERE {where_clause}
            """, params)
            
            candidates = []
            for row in cursor.fetchall():
                if row['id'] in exclude:
                    continue
                
                doc_embedding = np.frombuffer(row['embedding'], dtype=np.float32)
                doc_norm = doc_embedding / np.linalg.norm(doc_embedding)
                similarity = float(np.dot(query_norm, doc_norm))
                
                article = Article(
                    id=row['id'],
                    title=row['title'],
                    content=row['content'][:500],
                    category=row['category'],
                    published_at=row['published_at']
                )
                candidates.append((article, similarity))
            
            candidates.sort(key=lambda x: x[1], reverse=True)
            return candidates[:k]
            
        finally:
            cursor.close()
            conn.close()
    
    def _get_collaborative_candidates(
        self,
        user_id: int,
        k: int,
        exclude: set
    ) -> List[Tuple[Article, float]]:
        """Candidats basés sur le comportement d'utilisateurs similaires."""
        
        conn = self.db_pool.get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            # Trouver des utilisateurs similaires
            cursor.execute("""
                WITH my_articles AS (
                    SELECT article_id FROM user_interactions
                    WHERE user_id = %s AND interaction_type = 'read'
                ),
                similar_users AS (
                    SELECT ui.user_id, COUNT(*) as common_reads
                    FROM user_interactions ui
                    JOIN my_articles ma ON ui.article_id = ma.article_id
                    WHERE ui.user_id != %s
                    GROUP BY ui.user_id
                    HAVING common_reads >= 5
                    ORDER BY common_reads DESC
                    LIMIT 50
                )
                SELECT a.id, a.title, a.content, a.category, a.published_at,
                       COUNT(DISTINCT ui.user_id) as recommenders
                FROM similar_users su
                JOIN user_interactions ui ON su.user_id = ui.user_id
                JOIN articles a ON ui.article_id = a.id
                WHERE ui.interaction_type IN ('read', 'like')
                  AND a.published_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
                  AND a.id NOT IN (
                      SELECT article_id FROM user_interactions WHERE user_id = %s
                  )
                GROUP BY a.id
                ORDER BY recommenders DESC
                LIMIT %s
            """, (user_id, user_id, user_id, k))
            
            candidates = []
            for row in cursor.fetchall():
                if row['id'] in exclude:
                    continue
                
                article = Article(
                    id=row['id'],
                    title=row['title'],
                    content=row['content'][:500] if row['content'] else '',
                    category=row['category'],
                    published_at=row['published_at']
                )
                # Normaliser le score (0-1)
                score = min(row['recommenders'] / 10.0, 1.0)
                candidates.append((article, score))
            
            return candidates
            
        finally:
            cursor.close()
            conn.close()
    
    def _get_trending_candidates(
        self,
        k: int,
        exclude: set,
        category: Optional[str]
    ) -> List[Tuple[Article, float]]:
        """Candidats basés sur la popularité récente."""
        
        # Check cache
        cache_key = f"trending:{category or 'all'}"
        cached = self.redis.get(cache_key)
        
        if cached:
            trending = json.loads(cached)
        else:
            conn = self.db_pool.get_connection()
            cursor = conn.cursor(dictionary=True)
            
            try:
                where_clause = "a.published_at > DATE_SUB(NOW(), INTERVAL 24 HOUR)"
                params = []
                
                if category:
                    where_clause += " AND a.category = %s"
                    params.append(category)
                
                cursor.execute(f"""
                    SELECT a.id, a.title, a.content, a.category, a.published_at,
                           COUNT(*) as views,
                           SUM(CASE WHEN ui.interaction_type = 'share' THEN 5
                                    WHEN ui.interaction_type = 'like' THEN 2
                                    ELSE 1 END) as engagement_score
                    FROM articles a
                    LEFT JOIN user_interactions ui ON a.id = ui.article_id
                        AND ui.created_at > DATE_SUB(NOW(), INTERVAL 6 HOUR)
                    WHERE {where_clause}
                    GROUP BY a.id
                    ORDER BY engagement_score DESC
                    LIMIT 100
                """, params)
                
                trending = [dict(row) for row in cursor.fetchall()]
                
                # Cache for 5 minutes
                self.redis.setex(cache_key, 300, json.dumps(trending, default=str))
                
            finally:
                cursor.close()
                conn.close()
        
        # Normaliser et retourner
        candidates = []
        max_score = max((t['engagement_score'] or 0) for t in trending) if trending else 1
        
        for t in trending[:k]:
            if t['id'] in exclude:
                continue
            
            article = Article(
                id=t['id'],
                title=t['title'],
                content=t['content'][:500] if t['content'] else '',
                category=t['category'],
                published_at=datetime.fromisoformat(t['published_at']) if isinstance(t['published_at'], str) else t['published_at']
            )
            score = (t['engagement_score'] or 0) / max_score if max_score > 0 else 0
            candidates.append((article, score))
        
        return candidates
    
    def _compute_personalization_score(
        self,
        article: Article,
        user_profile: Dict
    ) -> float:
        """Score de personnalisation basé sur les préférences."""
        
        score = 0.0
        
        # Bonus catégorie préférée
        categories = user_profile.get('categories', {})
        if categories:
            total = sum(categories.values())
            if article.category in categories:
                score += categories[article.category] / total
        
        # Fraîcheur (articles récents préférés)
        hours_old = (datetime.now() - article.published_at).total_seconds() / 3600
        freshness = max(0, 1 - (hours_old / 168))  # Décroissance sur 7 jours
        score += 0.3 * freshness
        
        return min(score, 1.0)
    
    def _diversify(
        self,
        recommendations: List[Recommendation],
        k: int
    ) -> List[Recommendation]:
        """Diversifie les recommandations par catégorie."""
        
        result = []
        category_counts = {}
        max_per_category = max(2, k // 4)
        
        for rec in recommendations:
            cat = rec.article.category
            if category_counts.get(cat, 0) < max_per_category:
                result.append(rec)
                category_counts[cat] = category_counts.get(cat, 0) + 1
            
            if len(result) >= k:
                break
        
        return result
    
    def _get_read_articles(self, user_id: int) -> set:
        """Récupère les IDs des articles déjà lus."""
        
        conn = self.db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute("""
                SELECT article_id FROM user_interactions
                WHERE user_id = %s AND interaction_type = 'read'
                  AND created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
            """, (user_id,))
            
            return {row[0] for row in cursor.fetchall()}
            
        finally:
            cursor.close()
            conn.close()
```

### Résultats

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RÉSULTATS APRÈS 12 MOIS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Métriques d'engagement :                                                   │
│  ════════════════════════                                                   │
│  • CTR recommandations : 12.3% (+145% vs baseline aléatoire)                │
│  • Temps de session moyen : 8.2 min (+67%)                                  │
│  • Articles lus par session : 4.7 (+89%)                                    │
│  • Taux de rebond : 34% (-41%)                                              │
│                                                                             │
│  Performance technique :                                                    │
│  ══════════════════════                                                     │
│  • Latence recherche sémantique P50 : 89ms                                  │
│  • Latence recherche sémantique P95 : 234ms                                 │
│  • Latence recommandations P50 : 156ms                                      │
│  • Indexation nouvel article : 2.3 min (objectif : <5 min) ✅               │
│                                                                             │
│  Coûts IA :                                                                 │
│  ══════════                                                                 │
│  • Embeddings (batch) : $1,200/mois                                         │
│  • Claude API (RAG) : $3,800/mois                                           │
│  • Infrastructure : $8,500/mois                                             │
│  • Total : $13,500/mois (budget : $15K) ✅                                  │
│                                                                             │
│  Optimisations clés :                                                       │
│  ═══════════════════                                                        │
│  • Cache Redis embeddings : réduction 70% appels API                        │
│  • Batch embedding (100 articles/appel) : réduction 85% coûts               │
│  • Hybrid search (FT + Vector) : +23% pertinence vs vector seul             │
│  • User profile caching : réduction 60% queries DB                          │
│                                                                             │
│  Qualité IA :                                                               │
│  ═══════════                                                                │
│  • Précision fake news detection : 94.2%                                    │
│  • Satisfaction utilisateurs chatbot : 4.1/5                                │
│  • Pertinence résumés (évaluation humaine) : 4.3/5                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Étude de cas 4 : IoT - Télémétrie industrielle

### Contexte

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ÉTUDE DE CAS : IOT INDUSTRIEL                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Entreprise : SmartFactory (Industrie 4.0)                                  │
│  Secteur : Manufacturing / IoT                                              │
│  Volume : 10,000 capteurs, 500M points/jour, 5 ans rétention                │
│                                                                             │
│  Types de données :                                                         │
│  • Température, pression, vibration (haute fréquence)                       │
│  • États machines (on/off, alarmes)                                         │
│  • Production (compteurs, qualité)                                          │
│  • Énergie (consommation, pics)                                             │
│                                                                             │
│  Exigences :                                                                │
│  • Ingestion : 50K points/seconde soutenu                                   │
│  • Requêtes agrégées < 2 secondes                                           │
│  • Alertes temps réel < 5 secondes                                          │
│  • Rétention : 5 ans avec downsampling                                      │
│  • Coût stockage optimisé                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architecture retenue

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE SMARTFACTORY                               │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         EDGE LAYER                                  │   │
│  │                                                                     │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐         ┌─────────────────┐    │   │
│  │  │Sensor 1 │ │Sensor 2 │ │   ...   │  ──────►│  Edge Gateway   │    │   │
│  │  └─────────┘ └─────────┘ └─────────┘         │  (Aggregation)  │    │   │
│  │                                              └────────┬────────┘    │   │
│  └───────────────────────────────────────────────────────┼─────────────┘   │
│                                                          │                 │
│                                                   MQTT / Kafka             │
│                                                          │                 │
│  ┌───────────────────────────────────────────────────────┼─────────────┐   │
│  │                         INGESTION LAYER               │             │   │
│  │                                                       ▼             │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │                    Apache Kafka                             │    │   │
│  │  │                                                             │    │   │
│  │  │  Topics:                                                    │    │   │
│  │  │  • telemetry.temperature (50K msg/s)                        │    │   │
│  │  │  • telemetry.vibration (100K msg/s)                         │    │   │
│  │  │  • events.alarms                                            │    │   │
│  │  │  • events.production                                        │    │   │
│  │  └─────────────────────────────────┬───────────────────────────┘    │   │
│  │                                    │                                │   │
│  │              ┌─────────────────────┼─────────────────────┐          │   │
│  │              │                     │                     │          │   │
│  │              ▼                     ▼                     ▼          │   │
│  │  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐  │   │
│  │  │  Stream Processor │ │  Stream Processor │ │   Alert Engine    │  │   │
│  │  │  (Aggregation)    │ │  (Anomaly Det.)   │ │   (Flink)         │  │   │
│  │  └─────────┬─────────┘ └─────────┬─────────┘ └─────────┬─────────┘  │   │
│  │            │                     │                     │            │   │
│  └────────────┼─────────────────────┼─────────────────────┼────────────┘   │
│               │                     │                     │                │
│               ▼                     ▼                     ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         STORAGE LAYER                               │   │
│  │                                                                     │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │   │
│  │  │   HOT TIER      │  │   WARM TIER     │  │   COLD TIER     │      │   │
│  │  │   (7 jours)     │  │   (90 jours)    │  │   (5 ans)       │      │   │
│  │  │                 │  │                 │  │                 │      │   │
│  │  │  MariaDB        │  │  ColumnStore    │  │  S3 + Parquet   │      │   │
│  │  │  InnoDB         │  │  (aggregated)   │  │  (archived)     │      │   │
│  │  │                 │  │                 │  │                 │      │   │
│  │  │  Raw data       │  │  1-min rollups  │  │  1-hour rollups │      │   │
│  │  │  ~50GB          │  │  ~200GB         │  │  ~2TB           │      │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │   │
│  │          │                    │                    │                │   │
│  │          │    Downsampling    │     Archiving      │                │   │
│  │          └───────────────────►└───────────────────►│                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         QUERY LAYER                                 │   │
│  │                                                                     │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │   │
│  │  │   Real-time     │  │   Historical    │  │   ML/Analytics  │      │   │
│  │  │   Dashboard     │  │   Analysis      │  │   (Python)      │      │   │
│  │  │   (Grafana)     │  │   (Superset)    │  │                 │      │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Schéma de stockage tiered

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- HOT TIER : Données brutes (InnoDB, 7 jours)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE telemetry_raw (
    ts TIMESTAMP(3) NOT NULL,  -- Millisecond precision
    sensor_id INT NOT NULL,
    metric_type ENUM('temperature', 'pressure', 'vibration', 'flow') NOT NULL,
    value DOUBLE NOT NULL,
    quality TINYINT DEFAULT 100,  -- 0-100
    
    PRIMARY KEY (ts, sensor_id, metric_type),
    INDEX idx_sensor_time (sensor_id, ts)
) ENGINE=InnoDB
  PARTITION BY RANGE (UNIX_TIMESTAMP(ts)) (
    PARTITION p_current VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 1 DAY))),
    PARTITION p_day1 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 2 DAY))),
    PARTITION p_day2 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 3 DAY))),
    PARTITION p_day3 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 4 DAY))),
    PARTITION p_day4 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 5 DAY))),
    PARTITION p_day5 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 6 DAY))),
    PARTITION p_day6 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 7 DAY))),
    PARTITION p_future VALUES LESS THAN MAXVALUE
  );

-- ═══════════════════════════════════════════════════════════════════════════
-- WARM TIER : Agrégations 1 minute (ColumnStore, 90 jours)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE telemetry_1min (
    ts_bucket TIMESTAMP NOT NULL,  -- Arrondi à la minute
    sensor_id INT NOT NULL,
    metric_type VARCHAR(20) NOT NULL,
    
    -- Agrégations
    value_min DOUBLE,
    value_max DOUBLE,
    value_avg DOUBLE,
    value_sum DOUBLE,
    value_count INT,
    value_stddev DOUBLE,
    
    -- Percentiles pré-calculés
    value_p50 DOUBLE,
    value_p95 DOUBLE,
    value_p99 DOUBLE,
    
    -- Qualité
    quality_avg DOUBLE,
    
    PRIMARY KEY (ts_bucket, sensor_id, metric_type)
) ENGINE=Columnstore;

-- ═══════════════════════════════════════════════════════════════════════════
-- PROCÉDURE DE DOWNSAMPLING
-- ═══════════════════════════════════════════════════════════════════════════

DELIMITER //

CREATE PROCEDURE downsample_to_1min(IN p_start_ts TIMESTAMP, IN p_end_ts TIMESTAMP)
BEGIN
    INSERT INTO telemetry_1min
    SELECT 
        DATE_FORMAT(ts, '%Y-%m-%d %H:%i:00') AS ts_bucket,
        sensor_id,
        metric_type,
        MIN(value) AS value_min,
        MAX(value) AS value_max,
        AVG(value) AS value_avg,
        SUM(value) AS value_sum,
        COUNT(*) AS value_count,
        STDDEV(value) AS value_stddev,
        -- Approximation percentiles via bucketing
        NULL AS value_p50,  -- Calculé séparément si nécessaire
        NULL AS value_p95,
        NULL AS value_p99,
        AVG(quality) AS quality_avg
    FROM telemetry_raw
    WHERE ts BETWEEN p_start_ts AND p_end_ts
    GROUP BY ts_bucket, sensor_id, metric_type
    ON DUPLICATE KEY UPDATE
        value_min = LEAST(value_min, VALUES(value_min)),
        value_max = GREATEST(value_max, VALUES(value_max)),
        value_avg = (value_avg * value_count + VALUES(value_avg) * VALUES(value_count)) 
                    / (value_count + VALUES(value_count)),
        value_sum = value_sum + VALUES(value_sum),
        value_count = value_count + VALUES(value_count);
END //

-- Job de downsampling (exécuté toutes les minutes)
CREATE EVENT evt_downsample_1min
ON SCHEDULE EVERY 1 MINUTE
DO
BEGIN
    CALL downsample_to_1min(
        DATE_SUB(NOW(), INTERVAL 2 MINUTE),
        DATE_SUB(NOW(), INTERVAL 1 MINUTE)
    );
END //

-- Job de purge (exécuté quotidiennement)
CREATE EVENT evt_purge_hot_tier
ON SCHEDULE EVERY 1 DAY
STARTS CURRENT_DATE + INTERVAL 1 DAY + INTERVAL 3 HOUR  -- 3h du matin
DO
BEGIN
    -- Supprimer les données > 7 jours du hot tier
    ALTER TABLE telemetry_raw 
    DROP PARTITION p_day6,
    REORGANIZE PARTITION p_future INTO (
        PARTITION p_day6 VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(CURDATE(), INTERVAL 7 DAY))),
        PARTITION p_future VALUES LESS THAN MAXVALUE
    );
END //

DELIMITER ;

-- ═══════════════════════════════════════════════════════════════════════════
-- REQUÊTES OPTIMISÉES
-- ═══════════════════════════════════════════════════════════════════════════

-- Dashboard temps réel (hot tier)
SELECT 
    DATE_FORMAT(ts, '%Y-%m-%d %H:%i') AS minute,
    AVG(value) AS avg_temp,
    MAX(value) AS max_temp
FROM telemetry_raw
WHERE sensor_id = 1234
  AND metric_type = 'temperature'
  AND ts > DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY minute
ORDER BY minute;

-- Analyse historique (warm tier - ColumnStore)
SELECT 
    DATE(ts_bucket) AS day,
    sensor_id,
    AVG(value_avg) AS daily_avg,
    MAX(value_max) AS daily_max,
    SUM(value_count) AS total_readings
FROM telemetry_1min
WHERE metric_type = 'temperature'
  AND ts_bucket BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY day, sensor_id
ORDER BY day, sensor_id;

-- Détection d'anomalies (utilise les agrégations)
SELECT 
    sensor_id,
    ts_bucket,
    value_avg,
    value_stddev,
    CASE 
        WHEN value_max > (value_avg + 3 * value_stddev) THEN 'HIGH_SPIKE'
        WHEN value_min < (value_avg - 3 * value_stddev) THEN 'LOW_SPIKE'
        ELSE 'NORMAL'
    END AS anomaly_status
FROM telemetry_1min
WHERE ts_bucket > DATE_SUB(NOW(), INTERVAL 24 HOUR)
  AND value_stddev > 0
HAVING anomaly_status != 'NORMAL';
```

### Résultats

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RÉSULTATS APRÈS 18 MOIS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Performance d'ingestion :                                                  │
│  ══════════════════════                                                     │
│  • Throughput soutenu : 65K points/seconde (objectif : 50K) ✅              │
│  • Pic géré : 120K points/seconde (burst 2 min)                             │
│  • Latence ingestion P99 : 45ms                                             │
│                                                                             │
│  Performance des requêtes :                                                 │
│  ══════════════════════════                                                 │
│  • Dashboard temps réel (1h) : 120ms P95                                    │
│  • Analyse journalière (30 jours) : 0.8s P95                                │
│  • Analyse mensuelle (1 an) : 3.2s P95                                      │
│  • Requêtes cross-sensor (100 capteurs) : 1.4s P95                          │
│                                                                             │
│  Stockage :                                                                 │
│  ══════════                                                                 │
│  • Hot tier (InnoDB) : 48GB (7 jours)                                       │
│  • Warm tier (ColumnStore) : 180GB (90 jours) - compression 12:1            │
│  • Cold tier (S3 Parquet) : 1.8TB (5 ans) - compression 25:1                │
│  • Coût stockage total : $420/mois                                          │
│                                                                             │
│  Alertes :                                                                  │
│  ═════════                                                                  │
│  • Latence détection anomalie : 2.3s (objectif : <5s) ✅                    │
│  • Taux de faux positifs : 0.3%                                             │
│  • Alertes critiques manquées : 0                                           │
│                                                                             │
│  ROI Maintenance prédictive :                                               │
│  ════════════════════════════                                               │
│  • Réduction pannes non planifiées : -67%                                   │
│  • Économies maintenance : $1.2M/an                                         │
│  • ROI du projet : 340% sur 18 mois                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Synthèse et recommandations

### Matrice de décision par use case

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MATRICE DE DÉCISION ARCHITECTURALE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Use Case              │ Architecture recommandée                           │
│  ══════════════════════│═══════════════════════════════════════════════════ │
│  SaaS Multi-tenant     │ Sharding + Galera + Schema partagé + RLS           │
│  E-commerce standard   │ Read replicas + MaxScale + CDC pour analytics      │
│  Fintech/Paiement      │ Galera 5+ nodes + Outbox + Audit immutable         │
│  Media/Contenu IA      │ Hybrid search + Vector store + Cache agressif      │
│  IoT/Télémétrie        │ Tiered storage + ColumnStore + Partitioning        │
│  Analytics/BI          │ ColumnStore dédié + CDC depuis OLTP                │
│  Microservices         │ Database per service + Event-driven (Kafka)        │
│  Géo-distribué         │ Multi-région async + Galera intra-région           │
│                                                                             │
│  Patterns transverses :                                                     │
│  ══════════════════════                                                     │
│  • Toujours : MaxScale pour routing + failover                              │
│  • Souvent : CDC (Debezium) pour découplage                                 │
│  • Si IA : LangChain/LlamaIndex + embeddings dans MariaDB                   │
│  • Si scale-out : Galera pour HA, sharding pour capacité                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Checklist avant mise en production

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CHECKLIST PRÉ-PRODUCTION                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ☐ HAUTE DISPONIBILITÉ                                                      │
│    ☐ Galera ou réplication configurée et testée                             │
│    ☐ MaxScale ou ProxySQL pour failover automatique                         │
│    ☐ Test de failover effectué et documenté                                 │
│    ☐ RTO/RPO validés et conformes aux exigences                             │
│                                                                             │
│  ☐ PERFORMANCE                                                              │
│    ☐ Benchmarks réalisés avec données réalistes                             │
│    ☐ Index optimisés (EXPLAIN vérifié)                                      │
│    ☐ Configuration MariaDB ajustée (buffer pool, etc.)                      │
│    ☐ Connection pooling configuré                                           │
│                                                                             │
│  ☐ SÉCURITÉ                                                                 │
│    ☐ TLS activé pour toutes les connexions                                  │
│    ☐ Utilisateurs avec privilèges minimaux                                  │
│    ☐ Encryption at rest si données sensibles                                │
│    ☐ Audit logging activé                                                   │
│                                                                             │
│  ☐ OBSERVABILITÉ                                                            │
│    ☐ Métriques Prometheus/Grafana configurées                               │
│    ☐ Alertes définies (lag, connexions, erreurs)                            │
│    ☐ Logs centralisés (ELK ou équivalent)                                   │
│    ☐ Slow query log activé et surveillé                                     │
│                                                                             │
│  ☐ BACKUP & RECOVERY                                                        │
│    ☐ Backups automatisés (mariabackup)                                      │
│    ☐ Test de restauration effectué                                          │
│    ☐ PITR configuré si nécessaire                                           │
│    ☐ Rétention conforme aux exigences légales                               │
│                                                                             │
│  ☐ SCALABILITÉ                                                              │
│    ☐ Plan de scaling documenté (vertical puis horizontal)                   │
│    ☐ Sharding strategy définie si > 1TB                                     │
│    ☐ Read replicas prêtes si read-heavy                                     │
│    ☐ Headroom suffisant (charge < 70%)                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Points clés à retenir

- **Le choix d'architecture dépend du use case** — il n'y a pas de solution universelle
- **Galera Cluster** reste le standard pour la HA avec RPO=0 en intra-région
- **Le sharding** n'est nécessaire que pour les très gros volumes (> 1TB ou > 50K QPS)
- **ColumnStore** est idéal pour l'analytics sans impacter l'OLTP
- **Le pattern Outbox + CDC** garantit la cohérence dans les architectures event-driven
- **L'IA avec MariaDB** est viable pour < 10M vecteurs avec hybrid search
- **Le tiered storage** optimise les coûts pour les données temporelles
- **MaxScale** simplifie considérablement le routing et le failover

---

## 🔗 Ressources et références

- [📖 MariaDB Architecture Guides](https://mariadb.com/kb/en/mariadb-architecture/)
- [📖 Galera Cluster Best Practices](https://galeracluster.com/library/documentation/)
- [📖 ColumnStore User Guide](https://mariadb.com/kb/en/mariadb-columnstore/)
- [📖 MaxScale Documentation](https://mariadb.com/kb/en/maxscale/)
- [📖 Debezium MariaDB Connector](https://debezium.io/documentation/reference/stable/connectors/mariadb.html)

---

## ➡️ Conclusion du chapitre

Ce chapitre a couvert les principaux cas d'usage et architectures pour MariaDB 11.8 LTS :

| Section | Sujet | Points clés |
|---------|-------|-------------|
| 20.1 | OLTP vs OLAP | InnoDB pour transactions, ColumnStore pour analytics |
| 20.2 | Microservices | Database per service, Saga pattern, API Composition |
| 20.3 | Data Warehousing | ColumnStore, ETL/ELT, modélisation dimensionnelle |
| 20.4 | Multi-tenant | Schema shared, Row-Level Security, isolation |
| 20.5 | Géo-distribution | Multi-région, latence, consistance |
| 20.6 | Hybrid/Multi-cloud | Portabilité, vendor lock-in, exit strategy |
| 20.7 | Scaling | Vertical vs horizontal, Galera, sharding |
| 20.8 | Event-driven | CDC, Debezium, Kafka, Outbox pattern |
| 20.9 | IA/RAG | Vector search, embeddings, hybrid search |
| 20.10 | MCP Server | Intégration Claude, agents SQL |
| 20.11 | Frameworks IA | LangChain, LlamaIndex, Haystack |
| 20.12 | Études de cas | Applications réelles, décisions, résultats |


⏭️ [Glossaire des Termes Techniques](/annexes/glossaire/README.md)

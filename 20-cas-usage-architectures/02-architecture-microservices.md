🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.2 Architecture Microservices

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Chapitres 6 (Transactions), 13-14 (Réplication, HA), notions d'architecture distribuée

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les défis spécifiques de la gestion des données dans les architectures microservices
- Distinguer les patterns Database-per-Service et Shared Database avec leurs trade-offs
- Choisir le pattern adapté selon le contexte organisationnel et technique
- Concevoir des stratégies de communication inter-services pour la cohérence des données
- Implémenter des patterns de résilience (Saga, Outbox, CQRS) avec MariaDB
- Planifier une migration progressive d'un monolithe vers les microservices

---

## Introduction

L'architecture microservices a révolutionné la façon dont nous concevons et déployons les applications. Cependant, si le découpage applicatif en services autonomes apporte agilité et scalabilité, il introduit des défis majeurs pour la gestion des données.

La question centrale est : **comment gérer les données dans un monde où l'application n'est plus un monolithe avec une base unique, mais un ensemble de services distribués ?**

MariaDB 11.8 LTS offre des fonctionnalités qui facilitent les deux approches principales : bases dédiées par service avec réplication et haute disponibilité, ou base partagée avec isolation fine grâce aux schémas et aux privilèges granulaires.

### Le défi fondamental

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              LE DÉFI DES DONNÉES DANS LES MICROSERVICES                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MONOLITHE (Avant)                    MICROSERVICES (Après)                 │
│  ══════════════════                   ═════════════════════                 │
│                                                                             │
│  ┌─────────────────────┐              ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │
│  │                     │              │ Svc │ │ Svc │ │ Svc │ │ Svc │       │
│  │    Application      │              │  A  │ │  B  │ │  C  │ │  D  │       │
│  │    Monolithique     │              └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘       │
│  │                     │                 │       │       │       │          │
│  └──────────┬──────────┘                 ?       ?       ?       ?          │
│             │                            │       │       │       │          │
│             │                         Comment gérer les données ?           │
│             ▼                                                               │
│  ┌─────────────────────┐                                                    │
│  │   Base de Données   │              Options :                             │
│  │      Unique         │              • Une DB par service ?                │
│  │                     │              • Une DB partagée ?                   │
│  │  • Transactions     │              • Hybrid ?                            │
│  │    ACID simples     │                                                    │
│  │  • Jointures        │              Nouveaux défis :                      │
│  │    cross-tables     │              • Transactions distribuées            │
│  │  • Cohérence        │              • Cohérence éventuelle                │
│  │    immédiate        │              • Duplication de données              │
│  │                     │              • Jointures cross-service             │
│  └─────────────────────┘              • Queries distribuées                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Pourquoi la gestion des données est-elle si complexe ?

Dans un monolithe, les transactions ACID garantissent la cohérence : une commande crée simultanément l'enregistrement order, décrémente le stock, et débite le compte client — tout cela dans une seule transaction.

Dans les microservices, ces opérations sont réparties entre services distincts (Orders, Inventory, Payments), potentiellement avec leurs propres bases de données. Les garanties ACID traditionnelles ne s'appliquent plus directement.

| Propriété | Monolithe | Microservices |
|-----------|-----------|---------------|
| **Atomicité** | Transaction unique | Saga pattern requis |
| **Cohérence** | Immédiate | Éventuelle (eventual) |
| **Isolation** | Niveaux SQL standard | Compensation sur erreur |
| **Durabilité** | Commit unique | Commits distribués |

---

## Les deux patterns fondamentaux

### Vue d'ensemble

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    PATTERNS DE DONNÉES MICROSERVICES                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────┐ ┌─────────────────────────────────┐   │
│  │    DATABASE PER SERVICE         │ │      SHARED DATABASE            │   │
│  │    (Base dédiée)                │ │      (Base partagée)            │   │
│  ├─────────────────────────────────┤ ├─────────────────────────────────┤   │
│  │                                 │ │                                 │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐        │ │  ┌─────┐ ┌─────┐ ┌─────┐        │   │
│  │  │Svc A│ │Svc B│ │Svc C│        │ │  │Svc A│ │Svc B│ │Svc C│        │   │
│  │  └──┬──┘ └──┬──┘ └──┬──┘        │ │  └──┬──┘ └──┬──┘ └──┬──┘        │   │
│  │     │       │       │           │ │     │       │       │           │   │
│  │     ▼       ▼       ▼           │ │     └───────┼───────┘           │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐        │ │             ▼                   │   │
│  │  │DB A │ │DB B │ │DB C │        │ │      ┌───────────┐              │   │
│  │  └─────┘ └─────┘ └─────┘        │ │      │  Shared   │              │   │
│  │                                 │ │      │    DB     │              │   │
│  │  Isolation : TOTALE             │ │      └───────────┘              │   │
│  │  Couplage  : FAIBLE             │ │                                 │   │
│  │  Complexité: ÉLEVÉE             │ │  Isolation : PAR SCHÉMA         │   │
│  │                                 │ │  Couplage  : MODÉRÉ             │   │
│  └─────────────────────────────────┘ │  Complexité: FAIBLE             │   │
│                                      └─────────────────────────────────┘   │
│                                                                            │
│                              HYBRID                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Combinaison : Services critiques avec DB dédiée,                   │   │
│  │               Services liés partageant une DB                       │   │
│  │                                                                     │   │
│  │  ┌─────┐ ┌─────┐       ┌─────┐ ┌─────┐                              │   │
│  │  │Svc A│ │Svc B│       │Svc C│ │Svc D│                              │   │
│  │  └──┬──┘ └──┬──┘       └──┬──┘ └──┬──┘                              │   │
│  │     │       │             └───┬───┘                                 │   │
│  │     ▼       ▼                 ▼                                     │   │
│  │  ┌─────┐ ┌─────┐         ┌───────┐                                  │   │
│  │  │DB A │ │DB B │         │Shared │                                  │   │
│  │  │     │ │     │         │DB C+D │                                  │   │
│  │  └─────┘ └─────┘         └───────┘                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Comparaison détaillée

| Critère | Database per Service | Shared Database |
|---------|---------------------|-----------------|
| **Autonomie des équipes** | ✅ Totale | ⚠️ Coordination requise |
| **Déploiement indépendant** | ✅ Oui | ⚠️ Schéma partagé = coordination |
| **Scalabilité** | ✅ Par service | ⚠️ Globale uniquement |
| **Transactions ACID** | ❌ Cross-service impossibles | ✅ Natives |
| **Jointures cross-données** | ❌ API calls requis | ✅ SQL standard |
| **Cohérence des données** | ⚠️ Éventuelle | ✅ Immédiate |
| **Complexité opérationnelle** | ❌ Élevée (N bases) | ✅ Faible (1 base) |
| **Coût infrastructure** | ❌ Plus élevé | ✅ Mutualisé |
| **Isolation des pannes** | ✅ Par service | ❌ Point unique |
| **Choix technologique** | ✅ Libre par service | ❌ Unique |
| **Migration de données** | ✅ Par service | ⚠️ Globale |
| **Sécurité/Compliance** | ✅ Isolation forte | ⚠️ Risque de fuite |

---

## Quand utiliser quel pattern ?

### Database per Service : les bons candidats

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DATABASE PER SERVICE - QUAND L'UTILISER                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✅ RECOMMANDÉ QUAND :                                                      │
│                                                                             │
│  Organisation                                                               │
│  ├── Équipes autonomes et nombreuses (> 5 équipes)                          │
│  ├── Cycles de release indépendants requis                                  │
│  ├── Responsabilité end-to-end par équipe (you build it, you run it)        │
│  └── Culture DevOps mature                                                  │
│                                                                             │
│  Technique                                                                  │
│  ├── Services avec besoins de scaling très différents                       │
│  │   (ex: Service A: 100 QPS, Service B: 100K QPS)                          │
│  ├── Données avec patterns d'accès distincts                                │
│  │   (ex: OLTP pour Orders, Time-series pour Logs)                          │
│  ├── Exigences de latence spécifiques par service                           │
│  └── Besoin de technologies différentes                                     │
│      (ex: MariaDB pour transactions, Redis pour cache, Elastic pour search) │
│                                                                             │
│  Données                                                                    │
│  ├── Bounded contexts bien définis (DDD)                                    │
│  ├── Peu de relations cross-services                                        │
│  ├── Eventual consistency acceptable pour le métier                         │
│  └── Volume/complexité justifiant l'isolation                               │
│                                                                             │
│  Compliance                                                                 │
│  ├── Données sensibles à isoler (PCI-DSS, RGPD)                             │
│  ├── Audit trails séparés requis                                            │
│  └── Certifications par domaine                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Shared Database : les bons candidats

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   SHARED DATABASE - QUAND L'UTILISER                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✅ RECOMMANDÉ QUAND :                                                      │
│                                                                             │
│  Organisation                                                               │
│  ├── Petite équipe (< 20 développeurs)                                      │
│  ├── Coordination facile entre équipes                                      │
│  ├── Un DBA/équipe data centralisée                                         │
│  └── Budget infrastructure limité                                           │
│                                                                             │
│  Technique                                                                  │
│  ├── Transactions ACID cross-services critiques                             │
│  │   (ex: Paiement + Stock + Commande atomiques)                            │
│  ├── Rapports et analytics nécessitant des jointures                        │
│  ├── Scaling uniforme suffisant                                             │
│  └── Stack technique homogène                                               │
│                                                                             │
│  Données                                                                    │
│  ├── Fort couplage référentiel entre domaines                               │
│  │   (ex: Tous les services utilisent la table users)                       │
│  ├── Cohérence immédiate requise par le métier                              │
│  ├── Modèle de données mature et stable                                     │
│  └── Volume gérable par une instance (< 1 TB)                               │
│                                                                             │
│  Transition                                                                 │
│  ├── Migration progressive depuis un monolithe                              │
│  ├── Équipe en apprentissage des microservices                              │
│  └── Phase de découverte des bounded contexts                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Matrice de décision

```sql
-- Pseudo-code de décision (pour illustration)
/*
IF (equipes_autonomes > 5 
    AND bounded_contexts_clairs 
    AND eventual_consistency_ok 
    AND budget_infra_suffisant)
THEN 
    pattern = 'DATABASE_PER_SERVICE'
    
ELSEIF (petite_equipe 
        AND transactions_acid_critiques 
        AND fort_couplage_referentiel)
THEN 
    pattern = 'SHARED_DATABASE'
    
ELSE
    pattern = 'HYBRID'
    -- Services critiques/isolés : DB dédiée
    -- Services liés/transactionnels : DB partagée
END IF
*/
```

---

## Communication inter-services et cohérence

Quel que soit le pattern choisi, la communication entre services pour maintenir la cohérence des données est cruciale.

### Patterns de communication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PATTERNS DE COMMUNICATION INTER-SERVICES                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. SYNCHRONE (Request/Response)                                            │
│  ════════════════════════════════                                           │
│                                                                             │
│  ┌─────────┐     HTTP/gRPC      ┌─────────┐                                 │
│  │Service A│ ──────────────────►│Service B│                                 │
│  │         │◄────────────────── │         │                                 │
│  └─────────┘     Response       └─────────┘                                 │
│                                                                             │
│  ✅ Simple, réponse immédiate                                               │
│  ❌ Couplage temporel, cascade de pannes                                    │
│  Usage: Lectures, validations                                               │
│                                                                             │
│  2. ASYNCHRONE (Event-Driven)                                               │
│  ═════════════════════════════                                              │
│                                                                             │
│  ┌─────────┐     Event      ┌─────────┐     Event      ┌─────────┐          │
│  │Service A│ ─────────────► │  Queue  │ ─────────────► │Service B│          │
│  └─────────┘   (Publish)    │ (Kafka) │   (Subscribe)  └─────────┘          │
│                             └─────────┘                                     │
│                                                                             │
│  ✅ Découplage, résilience, scalabilité                                     │
│  ❌ Complexité, eventual consistency                                        │
│  Usage: Mises à jour, notifications, intégrations                           │
│                                                                             │
│  3. SAGA PATTERN (Transactions distribuées)                                 │
│  ═══════════════════════════════════════════                                │
│                                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                                  │
│  │ Order   │───►│Inventory│───►│ Payment │                                  │
│  │ Create  │    │ Reserve │    │ Charge  │                                  │
│  └────┬────┘    └────┬────┘    └────┬────┘                                  │
│       │              │              │                                       │
│       │    Si échec: Compensation   │                                       │
│       │◄─────────────┴──────────────┘                                       │
│       │         Rollback                                                    │
│                                                                             │
│  ✅ Transactions cross-services                                             │
│  ❌ Complexité de compensation                                              │
│  Usage: Workflows métier critiques                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Le pattern Saga avec MariaDB

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- IMPLÉMENTATION SAGA : Création de commande
-- ═══════════════════════════════════════════════════════════════════════════

-- Service Orders : Table pour suivre l'état de la saga
CREATE TABLE order_sagas (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT UNIQUE NOT NULL,
    status ENUM(
        'STARTED',
        'INVENTORY_RESERVED',
        'PAYMENT_CHARGED',
        'COMPLETED',
        'COMPENSATING',
        'COMPENSATED',
        'FAILED'
    ) DEFAULT 'STARTED',
    current_step INT DEFAULT 1,
    payload JSON NOT NULL,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    completed_at TIMESTAMP NULL,
    
    INDEX idx_status (status),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;

-- Outbox pattern pour émission fiable d'événements
CREATE TABLE order_outbox (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    aggregate_type VARCHAR(50) NOT NULL,         -- 'Order'
    aggregate_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,            -- 'OrderCreated', 'OrderCancelled'
    payload JSON NOT NULL,
    status ENUM('PENDING', 'PUBLISHED', 'FAILED') DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP NULL,
    retry_count INT DEFAULT 0,
    
    INDEX idx_pending (status, created_at)
) ENGINE=InnoDB;

-- Procédure : Démarrer une saga de commande
DELIMITER //

CREATE PROCEDURE start_order_saga(
    IN p_customer_id BIGINT,
    IN p_items JSON,  -- [{"product_id": 1, "quantity": 2, "price": 99.99}, ...]
    OUT p_order_id BIGINT,
    OUT p_saga_id BIGINT
)
BEGIN
    DECLARE v_total DECIMAL(12,2);
    
    -- Calculer le total
    SELECT SUM(
        JSON_UNQUOTE(JSON_EXTRACT(item.value, '$.quantity')) * 
        JSON_UNQUOTE(JSON_EXTRACT(item.value, '$.price'))
    ) INTO v_total
    FROM JSON_TABLE(p_items, '$[*]' COLUMNS (value JSON PATH '$')) AS item;
    
    START TRANSACTION;
    
    -- Créer la commande en état pending
    INSERT INTO orders (customer_id, status, total_amount, items_json)
    VALUES (p_customer_id, 'pending', v_total, p_items);
    
    SET p_order_id = LAST_INSERT_ID();
    
    -- Créer la saga
    INSERT INTO order_sagas (order_id, status, payload)
    VALUES (p_order_id, 'STARTED', JSON_OBJECT(
        'customer_id', p_customer_id,
        'items', p_items,
        'total', v_total
    ));
    
    SET p_saga_id = LAST_INSERT_ID();
    
    -- Émettre l'événement via outbox
    INSERT INTO order_outbox (aggregate_type, aggregate_id, event_type, payload)
    VALUES ('Order', p_order_id, 'OrderSagaStarted', JSON_OBJECT(
        'saga_id', p_saga_id,
        'order_id', p_order_id,
        'customer_id', p_customer_id,
        'items', p_items,
        'total', v_total,
        'next_step', 'RESERVE_INVENTORY'
    ));
    
    COMMIT;
END //

-- Procédure : Gérer la réponse d'un step de la saga
CREATE PROCEDURE handle_saga_step_result(
    IN p_saga_id BIGINT,
    IN p_step_name VARCHAR(50),
    IN p_success BOOLEAN,
    IN p_result JSON
)
BEGIN
    DECLARE v_order_id BIGINT;
    DECLARE v_current_status VARCHAR(50);
    DECLARE v_next_event VARCHAR(100);
    
    -- Récupérer l'état actuel
    SELECT order_id, status INTO v_order_id, v_current_status
    FROM order_sagas WHERE id = p_saga_id FOR UPDATE;
    
    START TRANSACTION;
    
    IF p_success THEN
        -- Avancer dans la saga
        CASE p_step_name
            WHEN 'RESERVE_INVENTORY' THEN
                UPDATE order_sagas 
                SET status = 'INVENTORY_RESERVED', current_step = 2
                WHERE id = p_saga_id;
                SET v_next_event = 'ChargePayment';
                
            WHEN 'CHARGE_PAYMENT' THEN
                UPDATE order_sagas 
                SET status = 'PAYMENT_CHARGED', current_step = 3
                WHERE id = p_saga_id;
                SET v_next_event = 'ConfirmOrder';
                
            WHEN 'CONFIRM_ORDER' THEN
                UPDATE order_sagas 
                SET status = 'COMPLETED', completed_at = NOW()
                WHERE id = p_saga_id;
                
                UPDATE orders SET status = 'confirmed' WHERE id = v_order_id;
                SET v_next_event = 'OrderCompleted';
        END CASE;
        
        -- Émettre l'événement suivant
        IF v_next_event IS NOT NULL THEN
            INSERT INTO order_outbox (aggregate_type, aggregate_id, event_type, payload)
            VALUES ('Order', v_order_id, v_next_event, p_result);
        END IF;
    ELSE
        -- Démarrer la compensation
        UPDATE order_sagas 
        SET status = 'COMPENSATING', error_message = JSON_UNQUOTE(JSON_EXTRACT(p_result, '$.error'))
        WHERE id = p_saga_id;
        
        -- Émettre les événements de compensation
        INSERT INTO order_outbox (aggregate_type, aggregate_id, event_type, payload)
        VALUES ('Order', v_order_id, 'CompensateSaga', JSON_OBJECT(
            'saga_id', p_saga_id,
            'failed_step', p_step_name,
            'compensate_from', v_current_status
        ));
    END IF;
    
    COMMIT;
END //

DELIMITER ;
```

### Le pattern CQRS avec MariaDB

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CQRS (Command Query Responsibility Segregation)          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                            Application                                      │
│                                │                                            │
│              ┌─────────────────┴─────────────────┐                          │
│              │                                   │                          │
│          Commands                            Queries                        │
│     (Create, Update, Delete)              (Read optimisé)                   │
│              │                                   │                          │
│              ▼                                   ▼                          │
│  ┌───────────────────────┐         ┌───────────────────────┐                │
│  │    Write Model        │         │    Read Model         │                │
│  │    (InnoDB)           │         │    (InnoDB/Views)     │                │
│  │                       │         │                       │                │
│  │  • Normalisé (3NF)    │ ──────► │  • Dénormalisé        │                │
│  │  • Transactions       │  Sync   │  • Pré-jointures      │                │
│  │  • Intégrité FK       │  Event  │  • Index covering     │                │
│  │  • Validation         │         │  • Matérialisé        │                │
│  └───────────────────────┘         └───────────────────────┘                │
│                                                                             │
│  Implémentation MariaDB :                                                   │
│  • Write: Tables InnoDB normalisées                                         │
│  • Read: Vues matérialisées via tables + triggers, ou réplication           │
│  • Sync: Triggers, Events, ou CDC (Debezium)                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- IMPLÉMENTATION CQRS SIMPLIFIÉE
-- ═══════════════════════════════════════════════════════════════════════════

-- WRITE MODEL : Tables normalisées
CREATE TABLE orders_write (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB;

CREATE TABLE order_items_write (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders_write(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
) ENGINE=InnoDB;

-- READ MODEL : Table dénormalisée pour lectures rapides
CREATE TABLE orders_read (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    customer_name VARCHAR(200),
    customer_email VARCHAR(255),
    status VARCHAR(20) NOT NULL,
    item_count INT,
    total_amount DECIMAL(12,2),
    items_summary JSON,  -- Résumé des articles
    created_at TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_customer (customer_id),
    INDEX idx_status_date (status, created_at DESC),
    INDEX idx_date (created_at DESC)
) ENGINE=InnoDB;

-- SYNCHRONISATION : Trigger pour maintenir le read model
DELIMITER //

CREATE TRIGGER trg_order_insert_sync
AFTER INSERT ON orders_write
FOR EACH ROW
BEGIN
    DECLARE v_customer_name VARCHAR(200);
    DECLARE v_customer_email VARCHAR(255);
    
    SELECT CONCAT(first_name, ' ', last_name), email
    INTO v_customer_name, v_customer_email
    FROM customers WHERE id = NEW.customer_id;
    
    INSERT INTO orders_read (
        order_id, customer_id, customer_name, customer_email,
        status, item_count, total_amount, created_at
    ) VALUES (
        NEW.id, NEW.customer_id, v_customer_name, v_customer_email,
        NEW.status, 0, 0, NEW.created_at
    );
END //

CREATE TRIGGER trg_order_item_insert_sync
AFTER INSERT ON order_items_write
FOR EACH ROW
BEGIN
    UPDATE orders_read
    SET 
        item_count = item_count + NEW.quantity,
        total_amount = total_amount + (NEW.quantity * NEW.unit_price),
        items_summary = JSON_ARRAY_APPEND(
            COALESCE(items_summary, JSON_ARRAY()),
            '$',
            JSON_OBJECT(
                'product_id', NEW.product_id,
                'quantity', NEW.quantity,
                'price', NEW.unit_price
            )
        )
    WHERE order_id = NEW.order_id;
END //

DELIMITER ;

-- Requête READ optimisée (pas de jointure)
SELECT order_id, customer_name, status, item_count, total_amount, created_at
FROM orders_read
WHERE customer_id = 12345
ORDER BY created_at DESC
LIMIT 20;
```

---

## Gestion des données partagées (Reference Data)

Un défi récurrent : certaines données sont utilisées par tous les services (utilisateurs, configurations, référentiels).

### Stratégies pour les données de référence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              STRATÉGIES POUR LES DONNÉES DE RÉFÉRENCE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. SERVICE DE RÉFÉRENCE DÉDIÉ                                              │
│  ═════════════════════════════                                              │
│                                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                                        │
│  │Orders   │ │Inventory│ │Shipping │                                        │
│  └────┬────┘ └────┬────┘ └────┬────┘                                        │
│       │           │           │                                             │
│       └───────────┼───────────┘                                             │
│                   │ API calls                                               │
│                   ▼                                                         │
│           ┌─────────────┐                                                   │
│           │ User Service│  ← Single source of truth                         │
│           └──────┬──────┘                                                   │
│                  │                                                          │
│               ┌──▼──┐                                                       │
│               │Users│                                                       │
│               │ DB  │                                                       │
│               └─────┘                                                       │
│                                                                             │
│  ✅ Source unique de vérité                                                 │
│  ❌ Point de contention, latence                                            │
│                                                                             │
│  2. RÉPLICATION DES DONNÉES (Cache local)                                   │
│  ════════════════════════════════════════                                   │
│                                                                             │
│  ┌─────────┐          ┌─────────┐          ┌─────────┐                      │
│  │Orders   │          │Inventory│          │Shipping │                      │
│  │┌───────┐│          │┌───────┐│          │┌───────┐│                      │
│  ││Users  ││◄─────────││Users  ││◄─────────││Users  ││                      │
│  ││(copy) ││  Events  ││(copy) ││  Events  ││(copy) ││                      │
│  │└───────┘│          │└───────┘│          │└───────┘│                      │
│  └─────────┘          └─────────┘          └─────────┘                      │
│       ▲                    ▲                    ▲                           │
│       │                    │                    │                           │
│       └────────────────────┼────────────────────┘                           │
│                            │                                                │
│                    ┌───────┴───────┐                                        │
│                    │ User Service  │  → Publie les changements              │
│                    └───────────────┘                                        │
│                                                                             │
│  ✅ Performance (lecture locale)                                            │
│  ❌ Eventual consistency, duplication                                       │
│                                                                             │
│  3. SHARED DATABASE POUR RÉFÉRENTIELS                                       │
│  ══════════════════════════════════════                                     │
│                                                                             │
│  Services métier : Database per Service                                     │
│  Référentiels : Shared Database en lecture seule                            │
│                                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                                        │
│  │Orders   │ │Inventory│ │Shipping │                                        │
│  │  (RW)   │ │  (RW)   │ │  (RW)   │                                        │
│  └────┬────┘ └────┬────┘ └────┬────┘                                        │
│       │           │           │                                             │
│  ┌────▼────┐ ┌────▼────┐ ┌────▼────┐                                        │
│  │Orders DB│ │Inv DB   │ │Ship DB  │                                        │
│  └─────────┘ └─────────┘ └─────────┘                                        │
│       │           │           │                                             │
│       └───────────┼───────────┘ (READ ONLY)                                 │
│                   │                                                         │
│           ┌───────▼───────┐                                                 │
│           │  Référentiels │  Users, Products, Config                        │
│           │   (Shared)    │                                                 │
│           └───────────────┘                                                 │
│                                                                             │
│  ✅ Simple, cohérent                                                        │
│  ❌ Point unique de défaillance                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implémentation avec MariaDB

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- RÉFÉRENTIELS PARTAGÉS AVEC ACCÈS CONTRÔLÉ
-- ═══════════════════════════════════════════════════════════════════════════

-- Base de données référentiels (gérée par un service dédié)
CREATE DATABASE reference_data;
USE reference_data;

CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    status ENUM('active', 'suspended', 'deleted') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    category_id INT,
    price DECIMAL(10,2),
    status ENUM('active', 'discontinued') DEFAULT 'active'
) ENGINE=InnoDB;

-- Utilisateurs en lecture seule pour les autres services
CREATE USER 'orders_svc_ro'@'%' IDENTIFIED BY 'secure_password_1';
GRANT SELECT ON reference_data.users TO 'orders_svc_ro'@'%';
GRANT SELECT ON reference_data.products TO 'orders_svc_ro'@'%';

CREATE USER 'inventory_svc_ro'@'%' IDENTIFIED BY 'secure_password_2';
GRANT SELECT ON reference_data.products TO 'inventory_svc_ro'@'%';

-- 🆕 MariaDB 11.8 : Privilèges granulaires par colonne
-- Le service Orders n'a pas besoin de voir les données sensibles
CREATE USER 'orders_svc_limited'@'%' IDENTIFIED BY 'secure_password_3';
GRANT SELECT (id, email, name, status) ON reference_data.users TO 'orders_svc_limited'@'%';

-- ═══════════════════════════════════════════════════════════════════════════
-- CACHE LOCAL VIA RÉPLICATION ÉVÉNEMENTIELLE
-- ═══════════════════════════════════════════════════════════════════════════

-- Service Orders : Table cache locale
USE orders_db;

CREATE TABLE users_cache (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(200) NOT NULL,
    status VARCHAR(20) NOT NULL,
    synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_email (email)
) ENGINE=InnoDB;

-- Procédure de synchronisation (appelée via consumer Kafka/RabbitMQ)
DELIMITER //

CREATE PROCEDURE sync_user_cache(
    IN p_user_id BIGINT,
    IN p_email VARCHAR(255),
    IN p_name VARCHAR(200),
    IN p_status VARCHAR(20),
    IN p_event_type VARCHAR(50)
)
BEGIN
    IF p_event_type = 'UserDeleted' THEN
        DELETE FROM users_cache WHERE user_id = p_user_id;
    ELSE
        INSERT INTO users_cache (user_id, email, name, status)
        VALUES (p_user_id, p_email, p_name, p_status)
        ON DUPLICATE KEY UPDATE
            email = VALUES(email),
            name = VALUES(name),
            status = VALUES(status),
            synced_at = NOW();
    END IF;
END //

DELIMITER ;
```

---

## Migration d'un monolithe vers les microservices

### Stratégie de migration progressive

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              MIGRATION PROGRESSIVE : STRANGLER FIG PATTERN                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE 1 : Monolithe initial                                                │
│  ════════════════════════════                                               │
│                                                                             │
│  ┌─────────────────────────────────┐                                        │
│  │         Monolithe               │                                        │
│  │  ┌───────┬───────┬─────────┐    │                                        │
│  │  │Orders │Users  │Inventory│    │                                        │
│  │  └───────┴───────┴─────────┘    │                                        │
│  └──────────────┬──────────────────┘                                        │
│                 │                                                           │
│          ┌──────▼──────┐                                                    │
│          │  Single DB  │                                                    │
│          └─────────────┘                                                    │
│                                                                             │
│  PHASE 2 : Extraction du premier service                                    │
│  ════════════════════════════════════════                                   │
│                                                                             │
│  ┌─────────────────────────────────┐      ┌─────────────┐                   │
│  │         Monolithe               │      │   Users     │                   │
│  │  ┌───────┬───────────┐          │      │   Service   │                   │
│  │  │Orders │Inventory  │          │ ────►│             │                   │
│  │  └───────┴───────────┘          │      └──────┬──────┘                   │
│  └──────────────┬──────────────────┘             │                          │
│                 │                                │                          │
│          ┌──────▼──────┐                  ┌──────▼──────┐                   │
│          │  Main DB    │                  │  Users DB   │                   │
│          │ (- users)   │                  │             │                   │
│          └─────────────┘                  └─────────────┘                   │
│                                                                             │
│  PHASE 3 : Services autonomes                                               │
│  ═════════════════════════════                                              │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Orders    │  │   Users     │  │  Inventory  │  │  Shipping   │         │
│  │   Service   │  │   Service   │  │   Service   │  │   Service   │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                │                │
│   ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐          │
│   │ Orders DB │    │ Users DB  │    │Inventory  │    │Shipping DB│          │
│   │           │    │           │    │    DB     │    │           │          │
│   └───────────┘    └───────────┘    └───────────┘    └───────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Checklist de migration

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- CHECKLIST MIGRATION : EXTRACTION D'UN SERVICE
-- ═══════════════════════════════════════════════════════════════════════════

/*
AVANT L'EXTRACTION :
□ Identifier les tables du bounded context
□ Mapper les dépendances entrantes (qui lit ces tables ?)
□ Mapper les dépendances sortantes (quelles autres tables sont lues ?)
□ Identifier les transactions cross-boundary
□ Définir les APIs du nouveau service
□ Planifier la stratégie de synchronisation

PENDANT L'EXTRACTION :
□ Créer la nouvelle base de données
□ Migrer les données
□ Mettre en place la synchronisation bidirectionnelle (transition)
□ Déployer le nouveau service en shadow mode
□ Basculer le trafic progressivement
□ Surveiller les métriques

APRÈS L'EXTRACTION :
□ Supprimer les tables de l'ancienne base
□ Supprimer la synchronisation de transition
□ Documenter les APIs
□ Mettre à jour les procédures opérationnelles
*/

-- Exemple : Identification des dépendances pour extraction du service Users

-- Tables à extraire
SELECT 'users' AS table_name, 'core' AS category
UNION SELECT 'user_addresses', 'related'
UNION SELECT 'user_preferences', 'related'
UNION SELECT 'user_sessions', 'related';

-- Dépendances entrantes (qui référence users ?)
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME = 'users'
  AND TABLE_SCHEMA = 'monolith_db';
  
-- Résultat attendu :
-- orders.customer_id → users.id
-- reviews.user_id → users.id
-- ...

-- Transactions cross-boundary à refactorer
-- (Rechercher dans le code les transactions touchant users + autres tables)
```

---

## Considérations opérationnelles

### Monitoring distribué

```yaml
# prometheus-microservices.yaml
# Configuration pour monitorer plusieurs instances MariaDB

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mariadb-orders'
    static_configs:
      - targets: ['orders-db:9104']
    relabel_configs:
      - source_labels: [__address__]
        target_label: service
        replacement: 'orders'

  - job_name: 'mariadb-users'
    static_configs:
      - targets: ['users-db:9104']
    relabel_configs:
      - source_labels: [__address__]
        target_label: service
        replacement: 'users'

  - job_name: 'mariadb-inventory'
    static_configs:
      - targets: ['inventory-db:9104']
    relabel_configs:
      - source_labels: [__address__]
        target_label: service
        replacement: 'inventory'

# Alertes cross-services
rule_files:
  - 'microservices-alerts.yml'
```

```yaml
# microservices-alerts.yml
groups:
  - name: microservices-data
    rules:
      # Alerte si un service a un lag de réplication
      - alert: ServiceReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag on {{ $labels.service }}"
          
      # Alerte si saga en échec depuis trop longtemps
      - alert: StuckSaga
        expr: |
          increase(saga_status_total{status="COMPENSATING"}[1h]) > 0
          and
          saga_status_total{status="COMPENSATING"} > 0
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "Saga stuck in COMPENSATING state"
```

### Backup et restore multi-bases

```bash
#!/bin/bash
# backup-microservices.sh
# Backup coordonné de toutes les bases de données microservices

BACKUP_DIR="/backup/$(date +%Y%m%d_%H%M%S)"
SERVICES=("orders" "users" "inventory" "shipping")

mkdir -p "$BACKUP_DIR"

# Backup en parallèle
for service in "${SERVICES[@]}"; do
    (
        echo "Backing up $service..."
        mariadb-dump \
            --host="${service}-db" \
            --user=backup_user \
            --password="$BACKUP_PASSWORD" \
            --single-transaction \
            --routines \
            --triggers \
            --events \
            "${service}_db" | gzip > "$BACKUP_DIR/${service}.sql.gz"
        echo "$service backup completed"
    ) &
done

# Attendre tous les backups
wait

# Vérification
for service in "${SERVICES[@]}"; do
    if [ ! -s "$BACKUP_DIR/${service}.sql.gz" ]; then
        echo "ERROR: Backup failed for $service"
        exit 1
    fi
done

echo "All backups completed successfully"
```

---

## ✅ Points clés à retenir

- **Database per Service** offre une autonomie maximale mais introduit la complexité des transactions distribuées et de la cohérence éventuelle
- **Shared Database** simplifie les transactions et jointures mais crée du couplage et limite l'autonomie des équipes
- **L'approche Hybrid** est souvent le meilleur compromis : services critiques isolés, services liés partageant une base
- **Le pattern Saga** est essentiel pour les transactions cross-services — MariaDB excelle pour implémenter les états de saga et l'outbox pattern
- **Les données de référence** (users, products, config) nécessitent une stratégie dédiée : service dédié, réplication événementielle, ou base partagée en lecture
- **La migration depuis un monolithe** doit être progressive (Strangler Fig) — ne pas tout casser d'un coup
- **Le monitoring distribué** est crucial — chaque base doit être surveillée avec des alertes cross-services
- 🆕 **MariaDB 11.8** avec ses privilèges granulaires facilite l'isolation dans les architectures shared database

---

## 🔗 Ressources et références

- [📖 Microservices Patterns - Chris Richardson](https://microservices.io/patterns/)
- [📖 Building Microservices - Sam Newman](https://samnewman.io/books/building_microservices_2nd_edition/)
- [📖 Domain-Driven Design - Eric Evans](https://www.domainlanguage.com/ddd/)
- [📖 Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [📖 CQRS Pattern](https://microservices.io/patterns/data/cqrs.html)
- [📖 MariaDB Replication](https://mariadb.com/kb/en/replication/)

---

## ➡️ Sections suivantes

Cette section se décline en deux sous-sections détaillées :

| Section | Titre | Focus |
|---------|-------|-------|
| 20.2.1 | [Database per Service](./02.1-database-per-service.md) | Implémentation complète avec Kubernetes, patterns de résilience |
| 20.2.2 | [Shared Database](./02.2-shared-database.md) | Isolation par schémas, gouvernance, performance |

⏭️ [Database per service](/20-cas-usage-architectures/02.1-database-per-service.md)

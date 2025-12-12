🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4 Events (Tâches Planifiées)

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Procédures stockées (8.1), SQL avancé (chapitres 3-4), Administration (chapitre 11)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et le fonctionnement de l'Event Scheduler dans MariaDB
- **Créer** des events avec différents types de planification (ONE TIME, EVERY)
- **Planifier** des tâches récurrentes avec des intervalles complexes
- **Gérer** le cycle de vie des events (activer, désactiver, supprimer)
- **Monitorer** l'exécution des events et diagnostiquer les problèmes
- **Implémenter** des tâches de maintenance automatisées
- **Appliquer** les bonnes pratiques pour des events fiables en production

---

## Introduction

Les **events** (ou événements planifiés) sont l'équivalent d'un **cron intégré** directement dans MariaDB. Ils permettent d'exécuter automatiquement des instructions SQL à des moments précis ou à intervalles réguliers, sans intervention externe.

### 🔍 Qu'est-ce qu'un Event ?

Un event est un objet de base de données qui :

- **S'exécute automatiquement** selon un planning défini
- **Contient du code SQL** (instructions, appels de procédures)
- **Est géré par l'Event Scheduler** (processus serveur dédié)
- **Persiste dans la base** (survit aux redémarrages du serveur)
- **Peut être unique ou récurrent**

```sql
-- Exemple simple : Purge quotidienne des logs
DELIMITER $$

CREATE EVENT purge_old_logs
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 02:00:00'
DO
BEGIN
    DELETE FROM application_logs
    WHERE created_at < NOW() - INTERVAL 90 DAY;
END$$

DELIMITER ;
```

### 📊 Events vs Alternatives

| Solution | Avantages | Inconvénients | Usage Recommandé |
|----------|-----------|---------------|------------------|
| **Events MariaDB** | ✅ Intégré à la DB<br>✅ Transactionnel<br>✅ Accès direct aux données | ❌ Pas d'alerting natif<br>❌ Limité au SQL<br>❌ Dépendant du serveur DB | Maintenance DB, Purges, Agrégations |
| **Cron/Systemd** | ✅ Standard Unix<br>✅ Flexible<br>✅ Monitoring externe | ❌ Nécessite accès serveur<br>❌ Credentials externes<br>❌ Plus complexe | Scripts complexes, Backups, Orchestration |
| **Application Scheduler** | ✅ Logique métier<br>✅ Intégré au code<br>✅ Déployable | ❌ Doit être toujours actif<br>❌ Scaling complexe | Jobs métier, Notifications, Workflows |
| **Kubernetes CronJob** | ✅ Cloud-native<br>✅ Scalable<br>✅ Observabilité | ❌ Infrastructure K8s<br>❌ Complexité setup | Environnements containerisés |

**Règle générale** :
- ✅ **Events** : Tâches SQL simples, proches des données
- ✅ **Cron** : Scripts système, backups, orchestration
- ✅ **Application** : Logique métier complexe
- ✅ **K8s CronJob** : Environnements cloud-native

---

## Architecture de l'Event Scheduler

### 📐 Fonctionnement Interne

```
┌────────────────────────────────────────────────────┐
│              MARIADB SERVER                        │
│                                                    │
│  ┌─────────────────────────────────────────────┐   │
│  │         EVENT SCHEDULER THREAD              │   │
│  │  - Vérifie les events à exécuter            │   │
│  │  - Lance les threads d'exécution            │   │
│  │  - Gère la planification                    │   │
│  └────────────────┬────────────────────────────┘   │
│                   │                                │
│                   │ Déclenche                      │
│                   ▼                                │
│  ┌─────────────────────────────────────────────┐   │
│  │      EVENTS EN ATTENTE                      │   │
│  │  event_1: EVERY 1 HOUR                      │   │
│  │  event_2: EVERY 1 DAY AT 02:00              │   │
│  │  event_3: ONE TIME AT '2025-12-31'          │   │
│  └────────────────┬────────────────────────────┘   │
│                   │                                │
│                   │ Exécution                      │
│                   ▼                                │
│  ┌─────────────────────────────────────────────┐   │
│  │      WORKER THREADS                         │   │
│  │  - Exécutent le code SQL de l'event         │   │
│  │  - Une transaction par event                │   │
│  │  - Logging dans INFORMATION_SCHEMA          │   │
│  └─────────────────────────────────────────────┘   │
│                                                    │
└────────────────────────────────────────────────────┘
```

### ⚙️ Configuration de l'Event Scheduler

L'Event Scheduler doit être **activé** pour que les events s'exécutent.

```sql
-- Vérifier l'état de l'Event Scheduler
SHOW VARIABLES LIKE 'event_scheduler';
-- +------------------+-------+
-- | Variable_name    | Value |
-- +------------------+-------+
-- | event_scheduler  | OFF   |
-- +------------------+-------+

-- Activer l'Event Scheduler (session courante)
SET GLOBAL event_scheduler = ON;

-- Ou via my.cnf pour activation permanente
-- [mysqld]
-- event_scheduler = ON

-- Valeurs possibles :
-- - ON (ou 1) : Activé
-- - OFF (ou 0) : Désactivé
-- - DISABLED : Désactivé et ne peut pas être activé dynamiquement
```

💡 **Important** : Si `event_scheduler = DISABLED` dans my.cnf, il faut redémarrer le serveur après modification.

### 📊 Monitoring de l'Event Scheduler

```sql
-- Vérifier le processus Event Scheduler
SHOW PROCESSLIST;
-- Rechercher : "event_scheduler" dans la colonne User

-- Voir les threads d'exécution d'events
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE USER = 'event_scheduler' OR COMMAND = 'Connect';

-- Statistiques des events
SELECT
    EVENT_SCHEMA,
    EVENT_NAME,
    STATUS,
    LAST_EXECUTED,
    STARTS,
    ENDS
FROM INFORMATION_SCHEMA.EVENTS
WHERE EVENT_SCHEMA = 'mydb';
```

---

## Types de Planification

### 1️⃣ ONE TIME - Exécution Unique

Événement qui s'exécute **une seule fois** à une date/heure précise.

```sql
DELIMITER $$

-- Event unique dans le futur
CREATE EVENT one_time_cleanup
ON SCHEDULE AT '2025-12-31 23:59:00'
DO
BEGIN
    -- Nettoyer les données temporaires de fin d'année
    DELETE FROM temp_data WHERE year = 2025;

    INSERT INTO maintenance_log (action, executed_at)
    VALUES ('Year-end cleanup completed', NOW());
END$$

-- Event unique avec intervalle relatif
CREATE EVENT maintenance_in_2_hours
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 2 HOUR
DO
BEGIN
    CALL perform_maintenance();
END$$

DELIMITER ;
```

**Caractéristiques** :
- ✅ Exécuté une seule fois puis automatiquement supprimé (sauf si ON COMPLETION PRESERVE)
- ✅ Utile pour tâches ponctuelles ou reports planifiés
- ⚠️ L'event est supprimé après exécution par défaut

### 2️⃣ EVERY - Exécution Récurrente

Événement qui s'exécute **périodiquement** à intervalles réguliers.

```sql
DELIMITER $$

-- Toutes les heures
CREATE EVENT hourly_aggregation
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    CALL aggregate_hourly_stats();
END$$

-- Tous les jours à 2h du matin
CREATE EVENT daily_backup
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 02:00:00'
DO
BEGIN
    CALL backup_critical_tables();
END$$

-- Toutes les 5 minutes
CREATE EVENT frequent_check
ON SCHEDULE EVERY 5 MINUTE
DO
BEGIN
    UPDATE system_status SET last_check = NOW();
END$$

-- Toutes les semaines (lundi à minuit)
CREATE EVENT weekly_report
ON SCHEDULE EVERY 1 WEEK
STARTS '2025-01-06 00:00:00'  -- Premier lundi
DO
BEGIN
    CALL generate_weekly_report();
END$$

DELIMITER ;
```

**Intervalles supportés** :
- YEAR, QUARTER, MONTH, WEEK
- DAY, HOUR, MINUTE, SECOND
- Combinaisons : `EVERY '1:30' HOUR_MINUTE` (toutes les 90 minutes)

### 3️⃣ Planification Avancée avec STARTS et ENDS

Contrôle précis du début et de la fin d'un event récurrent.

```sql
DELIMITER $$

-- Event actif seulement pendant les heures de bureau
CREATE EVENT business_hours_monitor
ON SCHEDULE EVERY 15 MINUTE
STARTS '2025-01-01 08:00:00'
ENDS '2025-12-31 18:00:00'
DO
BEGIN
    -- Monitoring uniquement 8h-18h
    INSERT INTO activity_log (timestamp, active_users)
    SELECT NOW(), COUNT(*) FROM active_sessions;
END$$

-- Event saisonnier (soldes d'été)
CREATE EVENT summer_sales_update
ON SCHEDULE EVERY 1 HOUR
STARTS '2025-06-01 00:00:00'
ENDS '2025-08-31 23:59:59'
DO
BEGIN
    UPDATE products
    SET price = price * 0.8
    WHERE category = 'summer_collection'
    AND sale_price IS NULL;
END$$

-- Event qui commence dans 1 heure et se termine dans 24h
CREATE EVENT limited_time_event
ON SCHEDULE EVERY 10 MINUTE
STARTS CURRENT_TIMESTAMP + INTERVAL 1 HOUR
ENDS CURRENT_TIMESTAMP + INTERVAL 1 DAY
DO
BEGIN
    CALL process_special_offers();
END$$

DELIMITER ;
```

**Comportement après ENDS** :
- Par défaut : L'event est **supprimé** automatiquement
- Avec `ON COMPLETION PRESERVE` : L'event est **conservé** mais désactivé

---

## Cas d'Usage Pratiques

### 1️⃣ Purge Automatique de Données

**Objectif** : Nettoyer automatiquement les anciennes données pour économiser l'espace disque.

```sql
DELIMITER $$

-- Purger les logs de plus de 90 jours (quotidien à 2h)
CREATE EVENT purge_application_logs
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 02:00:00'
COMMENT 'Supprime les logs de plus de 90 jours'
DO
BEGIN
    DECLARE deleted_rows INT DEFAULT 0;

    -- Suppression par batch pour éviter les locks longs
    DELETE FROM application_logs
    WHERE created_at < NOW() - INTERVAL 90 DAY
    LIMIT 10000;

    SET deleted_rows = ROW_COUNT();

    -- Logger l'opération
    INSERT INTO maintenance_log (event_name, rows_affected, executed_at)
    VALUES ('purge_application_logs', deleted_rows, NOW());

    -- Si des lignes ont été supprimées, optimiser la table
    IF deleted_rows > 0 THEN
        OPTIMIZE TABLE application_logs;
    END IF;
END$$

-- Purger les sessions expirées (toutes les heures)
CREATE EVENT purge_expired_sessions
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    DELETE FROM user_sessions
    WHERE expires_at < NOW();
END$$

-- Archiver puis supprimer les anciennes commandes
CREATE EVENT archive_old_orders
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 03:00:00'
DO
BEGIN
    -- Archiver dans une table dédiée
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE created_at < NOW() - INTERVAL 2 YEAR
    AND status = 'completed';

    -- Supprimer les commandes archivées
    DELETE FROM orders
    WHERE created_at < NOW() - INTERVAL 2 YEAR
    AND status = 'completed'
    LIMIT 5000;
END$$

DELIMITER ;
```

### 2️⃣ Agrégations et Statistiques Périodiques

**Objectif** : Précalculer des statistiques pour améliorer les performances des requêtes.

```sql
-- Table pour stocker les statistiques
CREATE TABLE daily_statistics (
    stat_date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(15,2),
    new_customers INT,
    active_users INT,
    computed_at DATETIME
) ENGINE=InnoDB;

DELIMITER $$

-- Calculer les statistiques quotidiennes
CREATE EVENT compute_daily_statistics
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 01:00:00'
COMMENT 'Calcule les statistiques de la journée précédente'
DO
BEGIN
    DECLARE yesterday DATE;
    SET yesterday = CURDATE() - INTERVAL 1 DAY;

    -- Insérer ou mettre à jour les stats
    INSERT INTO daily_statistics (
        stat_date,
        total_orders,
        total_revenue,
        new_customers,
        active_users,
        computed_at
    )
    SELECT
        yesterday,
        COUNT(DISTINCT o.id),
        COALESCE(SUM(o.total_amount), 0),
        COUNT(DISTINCT CASE WHEN c.created_at >= yesterday THEN c.id END),
        COUNT(DISTINCT u.id),
        NOW()
    FROM orders o
    LEFT JOIN customers c ON o.customer_id = c.id
    LEFT JOIN user_sessions u ON DATE(u.last_activity) = yesterday
    WHERE DATE(o.created_at) = yesterday
    ON DUPLICATE KEY UPDATE
        total_orders = VALUES(total_orders),
        total_revenue = VALUES(total_revenue),
        new_customers = VALUES(new_customers),
        active_users = VALUES(active_users),
        computed_at = NOW();
END$$

-- Agrégations horaires pour dashboard temps réel
CREATE EVENT compute_hourly_metrics
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    INSERT INTO hourly_metrics (hour_timestamp, metric_name, metric_value)
    SELECT
        DATE_FORMAT(NOW() - INTERVAL 1 HOUR, '%Y-%m-%d %H:00:00'),
        'orders_last_hour',
        COUNT(*)
    FROM orders
    WHERE created_at >= NOW() - INTERVAL 1 HOUR
    AND created_at < NOW();

    -- Ajouter d'autres métriques
    INSERT INTO hourly_metrics (hour_timestamp, metric_name, metric_value)
    SELECT
        DATE_FORMAT(NOW() - INTERVAL 1 HOUR, '%Y-%m-%d %H:00:00'),
        'revenue_last_hour',
        COALESCE(SUM(total_amount), 0)
    FROM orders
    WHERE created_at >= NOW() - INTERVAL 1 HOUR
    AND created_at < NOW();
END$$

DELIMITER ;
```

### 3️⃣ Maintenance Automatique

**Objectif** : Optimiser les tables et mettre à jour les statistiques régulièrement.

```sql
DELIMITER $$

-- Optimiser les tables fragmentées (hebdomadaire)
CREATE EVENT weekly_table_optimization
ON SCHEDULE EVERY 1 WEEK
STARTS '2025-01-05 03:00:00'  -- Dimanche à 3h
COMMENT 'Optimise les tables les plus fragmentées'
DO
BEGIN
    -- Optimiser les tables avec plus de 10% de fragmentation
    OPTIMIZE TABLE orders;
    OPTIMIZE TABLE order_items;
    OPTIMIZE TABLE products;
    OPTIMIZE TABLE customers;

    -- Logger l'opération
    INSERT INTO maintenance_log (operation, executed_at)
    VALUES ('Weekly table optimization', NOW());
END$$

-- Analyser les tables pour les statistiques d'index (quotidien)
CREATE EVENT daily_analyze_tables
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 04:00:00'
DO
BEGIN
    ANALYZE TABLE orders, products, customers, order_items;

    INSERT INTO maintenance_log (operation, executed_at)
    VALUES ('Daily table analysis', NOW());
END$$

-- Vérifier l'intégrité des tables (mensuel)
CREATE EVENT monthly_table_check
ON SCHEDULE EVERY 1 MONTH
STARTS '2025-01-01 02:00:00'
DO
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE table_name VARCHAR(64);
    DECLARE check_result VARCHAR(20);

    DECLARE table_cursor CURSOR FOR
        SELECT TABLE_NAME
        FROM INFORMATION_SCHEMA.TABLES
        WHERE TABLE_SCHEMA = DATABASE()
        AND ENGINE = 'InnoDB';

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN table_cursor;

    check_loop: LOOP
        FETCH table_cursor INTO table_name;
        IF done THEN
            LEAVE check_loop;
        END IF;

        -- CHECK TABLE ne peut pas être dans une procédure stockée
        -- On logue juste qu'il faut le faire
        INSERT INTO maintenance_log (operation, details, executed_at)
        VALUES ('Table check needed', table_name, NOW());
    END LOOP;

    CLOSE table_cursor;
END$$

DELIMITER ;
```

### 4️⃣ Synchronisation et Cache

**Objectif** : Synchroniser des données entre tables ou rafraîchir des caches.

```sql
DELIMITER $$

-- Mettre à jour un cache matérialisé de produits populaires
CREATE EVENT refresh_popular_products_cache
ON SCHEDULE EVERY 15 MINUTE
DO
BEGIN
    -- Vider le cache
    TRUNCATE TABLE popular_products_cache;

    -- Recalculer les produits populaires
    INSERT INTO popular_products_cache (product_id, order_count, last_updated)
    SELECT
        product_id,
        COUNT(*) AS order_count,
        NOW()
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    WHERE o.created_at >= NOW() - INTERVAL 7 DAY
    GROUP BY product_id
    ORDER BY order_count DESC
    LIMIT 100;
END$$

-- Synchroniser les compteurs dénormalisés
CREATE EVENT sync_denormalized_counters
ON SCHEDULE EVERY 5 MINUTE
DO
BEGIN
    -- Mettre à jour le nombre de commandes par client
    UPDATE customers c
    LEFT JOIN (
        SELECT customer_id, COUNT(*) AS cnt
        FROM orders
        GROUP BY customer_id
    ) o ON c.id = o.customer_id
    SET c.order_count = COALESCE(o.cnt, 0);

    -- Mettre à jour le stock virtuel des produits
    UPDATE products p
    LEFT JOIN (
        SELECT product_id, SUM(quantity) AS reserved
        FROM order_items oi
        JOIN orders o ON oi.order_id = o.id
        WHERE o.status = 'pending'
        GROUP BY product_id
    ) r ON p.id = r.product_id
    SET p.reserved_stock = COALESCE(r.reserved, 0);
END$$

DELIMITER ;
```

### 5️⃣ Notifications et Alertes

**Objectif** : Détecter des conditions anormales et créer des alertes.

```sql
-- Table pour stocker les alertes
CREATE TABLE system_alerts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    alert_type VARCHAR(50),
    severity ENUM('info', 'warning', 'critical'),
    message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    acknowledged BOOLEAN DEFAULT FALSE
) ENGINE=InnoDB;

DELIMITER $$

-- Surveiller le stock faible
CREATE EVENT check_low_stock
ON SCHEDULE EVERY 30 MINUTE
DO
BEGIN
    INSERT INTO system_alerts (alert_type, severity, message)
    SELECT
        'LOW_STOCK',
        CASE
            WHEN stock = 0 THEN 'critical'
            WHEN stock < 10 THEN 'warning'
            ELSE 'info'
        END,
        CONCAT('Product "', name, '" has low stock: ', stock, ' units')
    FROM products
    WHERE stock < 10
    AND is_active = TRUE
    AND NOT EXISTS (
        SELECT 1 FROM system_alerts
        WHERE alert_type = 'LOW_STOCK'
        AND message LIKE CONCAT('%', products.name, '%')
        AND acknowledged = FALSE
        AND created_at > NOW() - INTERVAL 1 HOUR
    );
END$$

-- Détecter les commandes bloquées
CREATE EVENT detect_stuck_orders
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    INSERT INTO system_alerts (alert_type, severity, message)
    SELECT
        'STUCK_ORDER',
        'warning',
        CONCAT('Order #', id, ' stuck in status "', status, '" for more than 24 hours')
    FROM orders
    WHERE status IN ('pending', 'processing')
    AND updated_at < NOW() - INTERVAL 24 HOUR;
END$$

-- Surveiller l'espace disque des tables
CREATE EVENT check_table_sizes
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 08:00:00'
DO
BEGIN
    DECLARE max_size_gb DECIMAL(10,2);

    SELECT MAX(ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2))
    INTO max_size_gb
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = DATABASE();

    -- Alerte si une table dépasse 50GB
    IF max_size_gb > 50 THEN
        INSERT INTO system_alerts (alert_type, severity, message)
        VALUES (
            'LARGE_TABLE',
            'warning',
            CONCAT('A table has exceeded 50GB: ', max_size_gb, 'GB')
        );
    END IF;
END$$

DELIMITER ;
```

### 6️⃣ Tâches Métier Récurrentes

**Objectif** : Automatiser des processus métier réguliers.

```sql
DELIMITER $$

-- Passer les commandes en retard en statut "late"
CREATE EVENT mark_late_orders
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    UPDATE orders
    SET
        status = 'late',
        updated_at = NOW()
    WHERE status = 'confirmed'
    AND expected_delivery_date < CURDATE()
    AND status != 'late';

    -- Notifier les clients (insérer dans une table de notifications)
    INSERT INTO pending_notifications (type, order_id, customer_id)
    SELECT 'ORDER_LATE', id, customer_id
    FROM orders
    WHERE status = 'late'
    AND updated_at >= NOW() - INTERVAL 1 HOUR;
END$$

-- Renouveler automatiquement les abonnements
CREATE EVENT renew_subscriptions
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 06:00:00'
DO
BEGIN
    -- Identifier les abonnements à renouveler
    UPDATE subscriptions
    SET
        status = 'active',
        renewed_at = NOW(),
        expires_at = expires_at + INTERVAL 1 MONTH
    WHERE status = 'active'
    AND expires_at <= CURDATE()
    AND auto_renew = TRUE;

    -- Logger les renouvellements
    INSERT INTO subscription_log (subscription_id, action, executed_at)
    SELECT id, 'AUTO_RENEWED', NOW()
    FROM subscriptions
    WHERE renewed_at >= NOW() - INTERVAL 1 MINUTE;
END$$

-- Calculer et attribuer les points de fidélité
CREATE EVENT award_loyalty_points
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 23:00:00'
DO
BEGIN
    -- Attribuer des points pour les commandes du jour
    UPDATE customers c
    JOIN (
        SELECT
            customer_id,
            SUM(total_amount) AS daily_spent
        FROM orders
        WHERE DATE(created_at) = CURDATE()
        AND status = 'completed'
        GROUP BY customer_id
    ) o ON c.id = o.customer_id
    SET
        c.loyalty_points = c.loyalty_points + FLOOR(o.daily_spent),
        c.updated_at = NOW();
END$$

DELIMITER ;
```

---

## Options des Events

### ON COMPLETION

Contrôle le comportement après l'exécution (pour ONE TIME) ou après ENDS (pour EVERY).

```sql
-- Event supprimé automatiquement après exécution (défaut)
CREATE EVENT temp_event
ON SCHEDULE AT '2025-12-31 23:59:00'
ON COMPLETION NOT PRESERVE
DO
    DELETE FROM temp_data WHERE year = 2025;

-- Event conservé après exécution
CREATE EVENT kept_event
ON SCHEDULE AT '2025-12-31 23:59:00'
ON COMPLETION PRESERVE
DO
    CALL generate_year_end_report();
```

### ENABLE / DISABLE

Contrôle l'activation de l'event.

```sql
-- Event actif (défaut)
CREATE EVENT active_event
ON SCHEDULE EVERY 1 HOUR
ENABLE
DO
    CALL process_data();

-- Event créé mais désactivé
CREATE EVENT inactive_event
ON SCHEDULE EVERY 1 HOUR
DISABLE
DO
    CALL experimental_process();

-- Désactiver un event existant
ALTER EVENT active_event DISABLE;

-- Réactiver un event
ALTER EVENT inactive_event ENABLE;

-- Désactiver temporairement sur un slave (réplication)
CREATE EVENT conditional_event
ON SCHEDULE EVERY 1 HOUR
DISABLE ON SLAVE
DO
    CALL master_only_task();
```

### COMMENT

Documentation de l'event.

```sql
CREATE EVENT documented_event
ON SCHEDULE EVERY 1 DAY
COMMENT 'Génère le rapport quotidien des ventes - v2.1 - 2025-12-12'
DO
BEGIN
    CALL generate_daily_sales_report();
END;
```

---

## Gestion et Maintenance des Events

### Lister les Events

```sql
-- Tous les events de la base courante
SHOW EVENTS;

-- Events d'une base spécifique
SHOW EVENTS FROM mydb;

-- Détails complets d'un event
SHOW CREATE EVENT nom_event;

-- Via INFORMATION_SCHEMA (plus de détails)
SELECT
    EVENT_SCHEMA,
    EVENT_NAME,
    DEFINER,
    EVENT_TYPE,
    EXECUTE_AT,
    INTERVAL_VALUE,
    INTERVAL_FIELD,
    STARTS,
    ENDS,
    STATUS,
    ON_COMPLETION,
    LAST_EXECUTED,
    EVENT_COMMENT
FROM INFORMATION_SCHEMA.EVENTS
WHERE EVENT_SCHEMA = DATABASE()
ORDER BY EVENT_NAME;
```

### Modifier un Event

```sql
-- Changer la planification
ALTER EVENT purge_logs
ON SCHEDULE EVERY 2 DAY;

-- Changer le code SQL
ALTER EVENT purge_logs
DO
BEGIN
    DELETE FROM logs WHERE created_at < NOW() - INTERVAL 60 DAY;
END;

-- Renommer un event
ALTER EVENT old_name
RENAME TO new_name;

-- Désactiver temporairement
ALTER EVENT purge_logs DISABLE;

-- Réactiver
ALTER EVENT purge_logs ENABLE;
```

### Supprimer un Event

```sql
DROP EVENT IF EXISTS nom_event;
```

### Privilèges

```sql
-- Créer des events
GRANT EVENT ON mydb.* TO 'developer'@'%';

-- Voir les events
GRANT SELECT ON mysql.event TO 'developer'@'%';

-- Supprimer tous les privilèges EVENT
REVOKE EVENT ON mydb.* FROM 'developer'@'%';
```

---

## Bonnes Pratiques

### 1. ✅ Toujours Activer l'Event Scheduler

```sql
-- Dans my.cnf
[mysqld]
event_scheduler = ON

-- Vérifier au démarrage
SHOW VARIABLES LIKE 'event_scheduler';
```

💡 **Important** : Sans Event Scheduler actif, aucun event ne s'exécutera.

### 2. ✅ Logger les Exécutions

```sql
-- Table de log
CREATE TABLE event_execution_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    event_name VARCHAR(64),
    execution_time DATETIME,
    duration_seconds DECIMAL(10,3),
    rows_affected INT,
    status ENUM('success', 'error'),
    error_message TEXT
) ENGINE=InnoDB;

DELIMITER $$

CREATE EVENT logged_maintenance
ON SCHEDULE EVERY 1 DAY
DO
BEGIN
    DECLARE start_time DATETIME;
    DECLARE affected_rows INT;

    SET start_time = NOW();

    -- Opération de maintenance
    DELETE FROM old_data WHERE created_at < NOW() - INTERVAL 1 YEAR;
    SET affected_rows = ROW_COUNT();

    -- Logger le résultat
    INSERT INTO event_execution_log (
        event_name,
        execution_time,
        duration_seconds,
        rows_affected,
        status
    ) VALUES (
        'logged_maintenance',
        start_time,
        TIMESTAMPDIFF(SECOND, start_time, NOW()),
        affected_rows,
        'success'
    );
END$$

DELIMITER ;
```

### 3. ✅ Gérer les Erreurs

```sql
DELIMITER $$

CREATE EVENT error_handled_event
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Logger l'erreur
        INSERT INTO event_execution_log (
            event_name,
            execution_time,
            status,
            error_message
        ) VALUES (
            'error_handled_event',
            NOW(),
            'error',
            'An error occurred during execution'
        );
    END;

    -- Code de l'event
    CALL risky_operation();
END$$

DELIMITER ;
```

### 4. ✅ Limiter les Opérations Lourdes

```sql
DELIMITER $$

-- ❌ MAUVAIS : Suppression massive sans limite
CREATE EVENT bad_delete_all
ON SCHEDULE EVERY 1 DAY
DO
BEGIN
    DELETE FROM logs WHERE created_at < NOW() - INTERVAL 1 YEAR;
    -- Peut bloquer la table pendant longtemps !
END$$

-- ✅ BON : Suppression par batch
CREATE EVENT good_delete_batched
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    DELETE FROM logs
    WHERE created_at < NOW() - INTERVAL 1 YEAR
    LIMIT 10000;
    -- Limite de 10k lignes par heure
END$$

DELIMITER ;
```

### 5. ✅ Éviter les Chevauchements

```sql
-- Table de verrou
CREATE TABLE event_locks (
    event_name VARCHAR(64) PRIMARY KEY,
    locked_at DATETIME,
    locked_by VARCHAR(100)
) ENGINE=InnoDB;

DELIMITER $$

CREATE EVENT non_overlapping_event
ON SCHEDULE EVERY 5 MINUTE
DO
BEGIN
    DECLARE lock_acquired BOOLEAN DEFAULT FALSE;

    -- Tenter d'acquérir le verrou
    INSERT IGNORE INTO event_locks (event_name, locked_at, locked_by)
    VALUES ('non_overlapping_event', NOW(), CONNECTION_ID());

    SET lock_acquired = ROW_COUNT() > 0;

    IF lock_acquired THEN
        -- Exécuter la tâche
        CALL long_running_task();

        -- Libérer le verrou
        DELETE FROM event_locks WHERE event_name = 'non_overlapping_event';
    END IF;
END$$

DELIMITER ;
```

### 6. ✅ Utiliser des Noms Descriptifs

```sql
-- ✅ BON : Noms clairs
CREATE EVENT purge_application_logs_daily;
CREATE EVENT compute_hourly_metrics;
CREATE EVENT send_weekly_reports_monday;

-- ❌ MAUVAIS : Noms ambigus
CREATE EVENT event1;
CREATE EVENT cleanup;
CREATE EVENT process;
```

### 7. ✅ Documenter avec COMMENT

```sql
CREATE EVENT well_documented_event
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 02:00:00'
COMMENT 'Purge application logs older than 90 days - Version 2.0 - Last modified: 2025-12-12 - Owner: Team Backend'
DO
BEGIN
    DELETE FROM application_logs
    WHERE created_at < NOW() - INTERVAL 90 DAY
    LIMIT 10000;
END;
```

### 8. ✅ Tester en Développement d'Abord

```sql
-- Créer une version test avec fréquence élevée
CREATE EVENT test_purge_logs
ON SCHEDULE EVERY 1 MINUTE
STARTS NOW()
ENDS NOW() + INTERVAL 1 HOUR
ON COMPLETION NOT PRESERVE
DO
BEGIN
    -- Tester avec LIMIT réduit
    DELETE FROM test_logs
    WHERE created_at < NOW() - INTERVAL 1 DAY
    LIMIT 100;

    INSERT INTO test_log (event_name, rows_deleted)
    VALUES ('test_purge_logs', ROW_COUNT());
END;
```

---

## Monitoring et Diagnostics

### Surveiller les Exécutions

```sql
-- Vérifier la dernière exécution des events
SELECT
    EVENT_NAME,
    LAST_EXECUTED,
    TIMESTAMPDIFF(MINUTE, LAST_EXECUTED, NOW()) AS minutes_since_last_exec,
    INTERVAL_VALUE,
    INTERVAL_FIELD,
    STATUS
FROM INFORMATION_SCHEMA.EVENTS
WHERE EVENT_SCHEMA = DATABASE()
ORDER BY LAST_EXECUTED DESC;

-- Détecter les events qui ne s'exécutent pas
SELECT
    EVENT_NAME,
    LAST_EXECUTED,
    TIMESTAMPDIFF(HOUR, LAST_EXECUTED, NOW()) AS hours_since_exec
FROM INFORMATION_SCHEMA.EVENTS
WHERE EVENT_SCHEMA = DATABASE()
AND LAST_EXECUTED IS NOT NULL
AND TIMESTAMPDIFF(HOUR, LAST_EXECUTED, NOW()) >
    CASE INTERVAL_FIELD
        WHEN 'HOUR' THEN INTERVAL_VALUE * 2
        WHEN 'DAY' THEN INTERVAL_VALUE * 24 * 2
        ELSE 24
    END;
```

### Analyser les Performances

```sql
-- Si vous avez une table de log
SELECT
    event_name,
    COUNT(*) AS executions,
    AVG(duration_seconds) AS avg_duration,
    MAX(duration_seconds) AS max_duration,
    SUM(CASE WHEN status = 'error' THEN 1 ELSE 0 END) AS error_count
FROM event_execution_log
WHERE execution_time >= NOW() - INTERVAL 7 DAY
GROUP BY event_name
ORDER BY avg_duration DESC;
```

---

## Limitations et Considérations

### ⚠️ Limitations Importantes

1. **Précision de l'exécution** : Les events ne sont pas garantis à la milliseconde près. Il peut y avoir un délai de quelques secondes.

2. **Événements simultanés** : Si un event prend plus de temps que son intervalle, plusieurs instances peuvent s'exécuter en parallèle.

3. **Redémarrage du serveur** : Les events ONE TIME qui devaient s'exécuter pendant l'arrêt ne seront PAS exécutés au redémarrage.

4. **Pas d'historique natif** : MariaDB ne garde pas d'historique des exécutions. Vous devez implémenter votre propre logging.

5. **Complexité limitée** : Pour des workflows complexes avec dépendances, préférez un orchestrateur externe.

### 💡 Quand NE PAS Utiliser des Events

❌ **N'utilisez PAS d'events pour :**

- **Tâches critiques temps réel** : Utiliser une application avec monitoring
- **Workflows complexes** : Utiliser Airflow, Temporal, ou similaire
- **Tâches nécessitant retry** : Events ne retentent pas automatiquement
- **Notifications urgentes** : Latence imprévisible, utiliser messaging queue
- **Backups critiques** : Utiliser des outils dédiés (Mariabackup + cron)

✅ **Utilisez des events pour :**

- Maintenance DB (purges, OPTIMIZE, ANALYZE)
- Agrégations statistiques
- Synchronisation de données dénormalisées
- Tâches récurrentes simples proches des données

---

## ✅ Points clés à retenir

- 🎯 **Intégration** : Events = cron intégré à MariaDB, idéal pour tâches SQL récurrentes
- ⏰ **Planification** : ONE TIME (unique) ou EVERY (récurrent) avec STARTS/ENDS
- 🔧 **Event Scheduler** : Doit être activé (`event_scheduler = ON`) pour fonctionner
- 📊 **Cas d'usage** : Purges, agrégations, maintenance, synchronisation
- 🛡️ **Bonnes pratiques** : Logger les exécutions, gérer les erreurs, limiter les opérations lourdes
- ⚠️ **Limitations** : Pas de garantie temps réel, pas d'historique natif, pas de retry automatique
- 🔍 **Monitoring** : Surveiller via INFORMATION_SCHEMA.EVENTS et logs personnalisés
- 📝 **Documentation** : Utiliser COMMENT et nommer clairement les events

---

## 🔗 Ressources et références

- [📖 CREATE EVENT - MariaDB Documentation](https://mariadb.com/kb/en/create-event/)
- [📖 ALTER EVENT - MariaDB Documentation](https://mariadb.com/kb/en/alter-event/)
- [📖 Event Scheduler - MariaDB Documentation](https://mariadb.com/kb/en/events/)
- [📖 INFORMATION_SCHEMA.EVENTS](https://mariadb.com/kb/en/information-schema-events-table/)
- [📖 Event Scheduler System Variables](https://mariadb.com/kb/en/server-system-variables/#event_scheduler)

---

## ➡️ Section suivante

**8.4.1 CREATE EVENT et planification** : Syntaxe détaillée, toutes les options de planification (intervalles complexes, expressions temporelles), et exemples exhaustifs de création d'events.

---


⏭️ [CREATE EVENT et planification](/08-programmation-cote-serveur/04.1-create-event.md)

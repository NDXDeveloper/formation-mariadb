🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 Vues matérialisées : Alternatives et workarounds

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-2.5 heures
> **Prérequis** : Section 9.1 (Création de vues), Chapitre 8 (Procédures et événements), compréhension des triggers

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre ce qu'est une vue matérialisée et pourquoi MariaDB n'en propose pas nativement
- Implémenter des tables de cache manuelles comme alternative aux vues matérialisées
- Automatiser le rafraîchissement des données avec des triggers pour une mise à jour en temps réel
- Utiliser les événements planifiés (EVENTs) pour des rafraîchissements périodiques
- Choisir la stratégie de rafraîchissement adaptée (full, incrémental, on-demand)
- Comparer les avantages et inconvénients de chaque approche
- Optimiser les performances des tables de cache avec des index appropriés
- Gérer le cycle de vie complet d'une pseudo-vue matérialisée en production

---

## Introduction

### Qu'est-ce qu'une vue matérialisée ?

Une **vue matérialisée** (materialized view) est une vue dont le résultat est **physiquement stocké sur disque**, contrairement à une vue standard qui recalcule ses données à chaque interrogation.

```sql
-- Vue standard (recalcule à chaque SELECT)
CREATE VIEW v_statistiques AS
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    AVG(prix) AS prix_moyen
FROM produits
GROUP BY categorie;
-- ⚠️ Le GROUP BY s'exécute à chaque interrogation

-- Vue matérialisée (concept, non supporté dans MariaDB)
CREATE MATERIALIZED VIEW mv_statistiques AS
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    AVG(prix) AS prix_moyen
FROM produits
GROUP BY categorie;
-- ✅ Les résultats sont stockés physiquement
-- ✅ Les requêtes sont instantanées (simple SELECT sur la table stockée)
```

**Avantages des vues matérialisées** :
- ⚡ **Performance** : Pas de recalcul à chaque requête
- 📊 **Agrégations précalculées** : Idéal pour les tableaux de bord et le reporting
- 🔄 **Snapshots de données** : États figés dans le temps
- 💾 **Indexation possible** : Comme une vraie table

**Inconvénients** :
- 💽 **Espace disque** : Stockage physique des résultats
- 🔄 **Fraîcheur des données** : Nécessite un rafraîchissement (refresh)
- ⚙️ **Complexité** : Gestion du cycle de mise à jour

### Pourquoi MariaDB ne supporte pas les vues matérialisées ?

MariaDB (et MySQL) ne proposent **pas nativement** de vues matérialisées, contrairement à PostgreSQL, Oracle ou SQL Server. Les raisons principales :

1. **Philosophie de conception** : Simplicité et compatibilité ascendante
2. **Alternatives existantes** : Tables normales + rafraîchissement périodique
3. **Flexibilité** : Les développeurs peuvent implémenter la logique qui correspond à leurs besoins

💡 **Note** : Cette limitation n'est pas un problème insurmontable. Les alternatives présentées dans cette section sont largement utilisées en production et offrent souvent plus de contrôle.

### Comparaison : Vue standard vs Vue matérialisée (concept)

| Aspect | Vue Standard | Vue Matérialisée (concept) |
|--------|--------------|---------------------------|
| **Stockage** | Définition SQL uniquement | Résultats stockés physiquement |
| **Performance SELECT** | 🐢 Recalcul à chaque fois | ⚡ Lecture directe |
| **Fraîcheur** | ✅ Toujours à jour | ⚠️ Dépend du refresh |
| **Espace disque** | Négligeable | Significatif |
| **Index** | Sur tables source | Sur vue matérialisée |
| **Mise à jour** | Automatique | Nécessite REFRESH |
| **Support MariaDB** | ✅ Natif | ❌ Non natif |

---

## Approche 1 : Tables de cache manuelles

La méthode la plus simple consiste à créer une **table normale** qui stocke le résultat de la requête complexe, puis à la rafraîchir manuellement ou via un script.

### Principe de fonctionnement

```sql
-- 1. Créer une table pour stocker les résultats
CREATE TABLE cache_statistiques_ventes (
    mois DATE,
    nb_commandes INT,
    ca_total DECIMAL(15,2),
    panier_moyen DECIMAL(10,2),
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (mois)
) ENGINE=InnoDB;

-- 2. Remplir la table avec les données
INSERT INTO cache_statistiques_ventes (mois, nb_commandes, ca_total, panier_moyen)
SELECT
    DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_ttc) AS ca_total,
    AVG(montant_ttc) AS panier_moyen
FROM commandes
GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01');

-- 3. Interroger la table de cache (ultra-rapide)
SELECT * FROM cache_statistiques_ventes
WHERE mois >= '2025-01-01'
ORDER BY mois DESC;
```

### Rafraîchissement complet (full refresh)

Remplacer toutes les données du cache :

```sql
-- Méthode 1 : TRUNCATE + INSERT
TRUNCATE TABLE cache_statistiques_ventes;

INSERT INTO cache_statistiques_ventes (mois, nb_commandes, ca_total, panier_moyen)
SELECT
    DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_ttc) AS ca_total,
    AVG(montant_ttc) AS panier_moyen
FROM commandes
GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01');

-- Méthode 2 : CREATE TABLE ... AS (recréation complète)
DROP TABLE IF EXISTS cache_statistiques_ventes;

CREATE TABLE cache_statistiques_ventes AS
SELECT
    DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_ttc) AS ca_total,
    AVG(montant_ttc) AS panier_moyen,
    NOW() AS derniere_maj
FROM commandes
GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01');

-- Ajouter les index après création
ALTER TABLE cache_statistiques_ventes
    ADD PRIMARY KEY (mois),
    ADD INDEX idx_ca_total (ca_total);
```

### Rafraîchissement incrémental (incremental refresh)

Mettre à jour uniquement les lignes modifiées (plus efficace pour les grandes tables) :

```sql
-- Utiliser INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO cache_statistiques_ventes (mois, nb_commandes, ca_total, panier_moyen)
SELECT
    DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_ttc) AS ca_total,
    AVG(montant_ttc) AS panier_moyen
FROM commandes
WHERE date_commande >= DATE_FORMAT(NOW(), '%Y-%m-01')  -- Seulement le mois courant
GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01')
ON DUPLICATE KEY UPDATE
    nb_commandes = VALUES(nb_commandes),
    ca_total = VALUES(ca_total),
    panier_moyen = VALUES(panier_moyen),
    derniere_maj = NOW();
```

### Encapsuler dans une procédure stockée

Pour faciliter le rafraîchissement, créez une procédure :

```sql
DELIMITER //

CREATE OR REPLACE PROCEDURE sp_refresh_cache_statistiques()
BEGIN
    -- Rafraîchissement complet
    TRUNCATE TABLE cache_statistiques_ventes;

    INSERT INTO cache_statistiques_ventes (mois, nb_commandes, ca_total, panier_moyen)
    SELECT
        DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
        COUNT(*) AS nb_commandes,
        SUM(montant_ttc) AS ca_total,
        AVG(montant_ttc) AS panier_moyen
    FROM commandes
    GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01');

    -- Log du rafraîchissement
    INSERT INTO log_refresh (table_name, refresh_date, nb_rows)
    VALUES ('cache_statistiques_ventes', NOW(), ROW_COUNT());
END//

DELIMITER ;

-- Utilisation
CALL sp_refresh_cache_statistiques();
```

**Avantages de cette approche** :
- ✅ Simple à comprendre et à implémenter
- ✅ Contrôle total sur le moment du rafraîchissement
- ✅ Pas de surcharge en temps réel
- ✅ Facilement intégrable dans les scripts ETL

**Inconvénients** :
- ❌ Nécessite une exécution manuelle ou via un scheduler externe
- ❌ Les données peuvent être obsolètes entre les rafraîchissements
- ❌ Pas de mise à jour en temps réel

💡 **Cas d'usage** : Idéal pour les rapports quotidiens, hebdomadaires ou mensuels où la fraîcheur à la seconde près n'est pas critique.

---

## Approche 2 : Rafraîchissement automatique avec triggers

Les **triggers** permettent de maintenir le cache automatiquement à jour lors de chaque modification des tables sources.

### Principe de fonctionnement

Chaque fois qu'une ligne est insérée, modifiée ou supprimée dans la table source, un trigger met à jour la table de cache correspondante.

### Exemple : Cache de compteurs avec triggers

```sql
-- 1. Table source
CREATE TABLE ventes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produit_id INT,
    date_vente DATE,
    quantite INT,
    montant DECIMAL(10,2)
) ENGINE=InnoDB;

-- 2. Table de cache
CREATE TABLE cache_stats_produits (
    produit_id INT PRIMARY KEY,
    total_ventes INT DEFAULT 0,
    total_quantite INT DEFAULT 0,
    total_montant DECIMAL(15,2) DEFAULT 0,
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 3. Trigger AFTER INSERT
DELIMITER //

CREATE TRIGGER trg_ventes_after_insert
AFTER INSERT ON ventes
FOR EACH ROW
BEGIN
    INSERT INTO cache_stats_produits (produit_id, total_ventes, total_quantite, total_montant)
    VALUES (NEW.produit_id, 1, NEW.quantite, NEW.montant)
    ON DUPLICATE KEY UPDATE
        total_ventes = total_ventes + 1,
        total_quantite = total_quantite + NEW.quantite,
        total_montant = total_montant + NEW.montant,
        derniere_maj = NOW();
END//

-- 4. Trigger AFTER UPDATE
CREATE TRIGGER trg_ventes_after_update
AFTER UPDATE ON ventes
FOR EACH ROW
BEGIN
    -- Soustraire les anciennes valeurs
    UPDATE cache_stats_produits
    SET
        total_quantite = total_quantite - OLD.quantite,
        total_montant = total_montant - OLD.montant,
        derniere_maj = NOW()
    WHERE produit_id = OLD.produit_id;

    -- Ajouter les nouvelles valeurs
    UPDATE cache_stats_produits
    SET
        total_quantite = total_quantite + NEW.quantite,
        total_montant = total_montant + NEW.montant,
        derniere_maj = NOW()
    WHERE produit_id = NEW.produit_id;

    -- Si le produit a changé, ajuster les compteurs
    IF OLD.produit_id != NEW.produit_id THEN
        UPDATE cache_stats_produits
        SET total_ventes = total_ventes - 1
        WHERE produit_id = OLD.produit_id;

        INSERT INTO cache_stats_produits (produit_id, total_ventes, total_quantite, total_montant)
        VALUES (NEW.produit_id, 1, NEW.quantite, NEW.montant)
        ON DUPLICATE KEY UPDATE
            total_ventes = total_ventes + 1;
    END IF;
END//

-- 5. Trigger AFTER DELETE
CREATE TRIGGER trg_ventes_after_delete
AFTER DELETE ON ventes
FOR EACH ROW
BEGIN
    UPDATE cache_stats_produits
    SET
        total_ventes = total_ventes - 1,
        total_quantite = total_quantite - OLD.quantite,
        total_montant = total_montant - OLD.montant,
        derniere_maj = NOW()
    WHERE produit_id = OLD.produit_id;

    -- Supprimer l'entrée si plus aucune vente
    DELETE FROM cache_stats_produits
    WHERE produit_id = OLD.produit_id
      AND total_ventes = 0;
END//

DELIMITER ;
```

### Initialisation du cache

Après avoir créé les triggers, remplir le cache avec les données existantes :

```sql
-- Remplir le cache avec les données historiques
INSERT INTO cache_stats_produits (produit_id, total_ventes, total_quantite, total_montant)
SELECT
    produit_id,
    COUNT(*) AS total_ventes,
    SUM(quantite) AS total_quantite,
    SUM(montant) AS total_montant
FROM ventes
GROUP BY produit_id
ON DUPLICATE KEY UPDATE
    total_ventes = VALUES(total_ventes),
    total_quantite = VALUES(total_quantite),
    total_montant = VALUES(total_montant);
```

### Test du système

```sql
-- Insertion : le cache se met à jour automatiquement
INSERT INTO ventes (produit_id, date_vente, quantite, montant)
VALUES (101, '2025-12-01', 5, 150.00);

-- Vérifier le cache
SELECT * FROM cache_stats_produits WHERE produit_id = 101;
-- Résultat : total_ventes=1, total_quantite=5, total_montant=150.00

-- Nouvelle insertion
INSERT INTO ventes (produit_id, date_vente, quantite, montant)
VALUES (101, '2025-12-02', 3, 90.00);

-- Vérifier le cache mis à jour
SELECT * FROM cache_stats_produits WHERE produit_id = 101;
-- Résultat : total_ventes=2, total_quantite=8, total_montant=240.00
```

**Avantages des triggers** :
- ✅ Mise à jour en temps réel
- ✅ Transparence : pas besoin d'intervention manuelle
- ✅ Données toujours à jour
- ✅ Cohérence garantie

**Inconvénients** :
- ❌ Surcharge sur les opérations d'écriture (INSERT/UPDATE/DELETE)
- ❌ Complexité accrue pour les calculs complexes
- ❌ Difficile à déboguer en cas de problème
- ❌ Verrouillage possible si le trigger prend du temps

⚠️ **Attention** : Les triggers ralentissent les opérations d'écriture. Cette approche n'est recommandée que pour des tables avec des écritures modérées et des lectures fréquentes.

💡 **Cas d'usage** : Idéal pour les compteurs, les totaux, les statistiques simples où la fraîcheur en temps réel est critique.

---

## Approche 3 : Rafraîchissement périodique avec EVENTs

Les **événements planifiés** (EVENTs) permettent d'automatiser le rafraîchissement à intervalles réguliers (comme un cron intégré à MariaDB).

### Activation de l'Event Scheduler

```sql
-- Vérifier si l'event scheduler est activé
SHOW VARIABLES LIKE 'event_scheduler';

-- Activer l'event scheduler (redémarrage non requis)
SET GLOBAL event_scheduler = ON;

-- Persister la configuration dans my.cnf
-- [mysqld]
-- event_scheduler = ON
```

### Créer un événement de rafraîchissement

#### Exemple 1 : Rafraîchissement toutes les heures

```sql
DELIMITER //

CREATE EVENT IF NOT EXISTS evt_refresh_cache_ventes
ON SCHEDULE EVERY 1 HOUR
STARTS CURRENT_TIMESTAMP
DO
BEGIN
    -- Rafraîchir la table de cache
    TRUNCATE TABLE cache_statistiques_ventes;

    INSERT INTO cache_statistiques_ventes (mois, nb_commandes, ca_total, panier_moyen)
    SELECT
        DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
        COUNT(*) AS nb_commandes,
        SUM(montant_ttc) AS ca_total,
        AVG(montant_ttc) AS panier_moyen
    FROM commandes
    GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01');

    -- Logger le rafraîchissement
    INSERT INTO log_events (event_name, execution_time, rows_affected)
    VALUES ('evt_refresh_cache_ventes', NOW(), ROW_COUNT());
END//

DELIMITER ;
```

#### Exemple 2 : Rafraîchissement quotidien à heure fixe

```sql
DELIMITER //

CREATE EVENT IF NOT EXISTS evt_refresh_daily_stats
ON SCHEDULE EVERY 1 DAY
STARTS TIMESTAMP(CURRENT_DATE + INTERVAL 1 DAY, '03:00:00')  -- 3h du matin
DO
BEGIN
    CALL sp_refresh_cache_statistiques();
END//

DELIMITER ;
```

#### Exemple 3 : Rafraîchissement incrémental toutes les 15 minutes

```sql
DELIMITER //

CREATE EVENT IF NOT EXISTS evt_refresh_incremental_ventes
ON SCHEDULE EVERY 15 MINUTE
STARTS CURRENT_TIMESTAMP
DO
BEGIN
    -- Mettre à jour uniquement les données récentes
    INSERT INTO cache_statistiques_ventes (mois, nb_commandes, ca_total, panier_moyen)
    SELECT
        DATE_FORMAT(date_commande, '%Y-%m-01') AS mois,
        COUNT(*) AS nb_commandes,
        SUM(montant_ttc) AS ca_total,
        AVG(montant_ttc) AS panier_moyen
    FROM commandes
    WHERE date_modification >= DATE_SUB(NOW(), INTERVAL 20 MINUTE)  -- Marge de sécurité
    GROUP BY DATE_FORMAT(date_commande, '%Y-%m-01')
    ON DUPLICATE KEY UPDATE
        nb_commandes = VALUES(nb_commandes),
        ca_total = VALUES(ca_total),
        panier_moyen = VALUES(panier_moyen),
        derniere_maj = NOW();
END//

DELIMITER ;
```

### Gestion des événements

```sql
-- Lister tous les événements
SHOW EVENTS;

-- Voir la définition d'un événement
SHOW CREATE EVENT evt_refresh_cache_ventes\G

-- Désactiver temporairement un événement
ALTER EVENT evt_refresh_cache_ventes DISABLE;

-- Réactiver un événement
ALTER EVENT evt_refresh_cache_ventes ENABLE;

-- Modifier la fréquence d'un événement
ALTER EVENT evt_refresh_cache_ventes
ON SCHEDULE EVERY 30 MINUTE;

-- Supprimer un événement
DROP EVENT IF EXISTS evt_refresh_cache_ventes;
```

### Surveillance des événements

```sql
-- Vérifier le statut des événements
SELECT
    EVENT_NAME,
    EVENT_SCHEMA,
    STATUS,
    ON_SCHEDULE,
    INTERVAL_VALUE,
    INTERVAL_FIELD,
    LAST_EXECUTED,
    STARTS,
    ENDS
FROM INFORMATION_SCHEMA.EVENTS
WHERE EVENT_SCHEMA = DATABASE();

-- Créer une table de log pour tracer les exécutions
CREATE TABLE log_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_name VARCHAR(100),
    execution_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    rows_affected INT,
    execution_duration_ms INT,
    status ENUM('SUCCESS', 'ERROR') DEFAULT 'SUCCESS',
    error_message TEXT
) ENGINE=InnoDB;
```

**Avantages des EVENTs** :
- ✅ Automatisation complète sans dépendance externe
- ✅ Intégré à MariaDB (pas besoin de cron ou scheduler externe)
- ✅ Flexibilité des planifications (minutes, heures, jours)
- ✅ Facile à surveiller avec INFORMATION_SCHEMA.EVENTS

**Inconvénients** :
- ❌ Nécessite event_scheduler activé
- ❌ Moins flexible qu'un scheduler externe pour des planifications complexes
- ❌ Peut surcharger le serveur si trop d'événements simultanés

💡 **Cas d'usage** : Idéal pour les rapports réguliers (quotidiens, horaires) où un décalage de quelques minutes est acceptable.

---

## Comparaison des approches

### Tableau récapitulatif

| Critère | Tables manuelles | Triggers | EVENTs |
|---------|-----------------|---------|--------|
| **Fraîcheur données** | ⚠️ Obsolètes entre refresh | ✅ Temps réel | ⚠️ Décalage selon planification |
| **Performance écriture** | ✅ Aucun impact | ❌ Impact significatif | ✅ Aucun impact |
| **Performance lecture** | ✅ Excellente | ✅ Excellente | ✅ Excellente |
| **Complexité** | ✅ Simple | ⚠️ Moyenne à élevée | ⚠️ Moyenne |
| **Automatisation** | ❌ Manuelle | ✅ Complète | ✅ Complète |
| **Debugging** | ✅ Facile | ❌ Difficile | ⚠️ Moyenne |
| **Flexibilité** | ✅ Totale | ❌ Limitée | ✅ Élevée |
| **Cas d'usage** | Rapports batch | Compteurs temps réel | Rapports périodiques |

### Critères de choix

**Choisissez les tables manuelles si** :
- Vous avez déjà un système d'ETL ou de batch processing
- Les données peuvent être obsolètes de plusieurs heures/jours
- Vous voulez un contrôle total sur le moment du rafraîchissement
- Vous avez un faible volume d'écritures mais un fort volume de lectures

**Choisissez les triggers si** :
- Vous avez besoin de données en temps réel
- Le volume d'écritures est modéré (< 1000 transactions/seconde)
- Les calculs sont simples (compteurs, sommes)
- La cohérence immédiate est critique

**Choisissez les EVENTs si** :
- Vous voulez une automatisation sans dépendance externe
- Un décalage de quelques minutes/heures est acceptable
- Vous avez besoin de rafraîchissements réguliers (horaires, quotidiens)
- Vous voulez centraliser la logique dans la base de données

---

## Approche hybride : Combiner les méthodes

Dans la pratique, une approche hybride est souvent la meilleure solution.

### Exemple : Cache incrémental avec EVENT + trigger pour urgences

```sql
-- 1. Table de cache
CREATE TABLE cache_dashboard (
    metric_key VARCHAR(50) PRIMARY KEY,
    metric_value DECIMAL(15,2),
    last_update TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    update_source ENUM('EVENT', 'TRIGGER', 'MANUAL') DEFAULT 'EVENT'
) ENGINE=InnoDB;

-- 2. EVENT pour rafraîchissement complet quotidien (3h du matin)
DELIMITER //

CREATE EVENT evt_refresh_full_dashboard
ON SCHEDULE EVERY 1 DAY
STARTS TIMESTAMP(CURRENT_DATE + INTERVAL 1 DAY, '03:00:00')
DO
BEGIN
    -- Rafraîchir tous les KPIs
    INSERT INTO cache_dashboard (metric_key, metric_value, update_source)
    VALUES
        ('ca_jour', (SELECT SUM(montant) FROM ventes WHERE DATE(date_vente) = CURDATE()), 'EVENT'),
        ('ca_mois', (SELECT SUM(montant) FROM ventes WHERE YEAR(date_vente) = YEAR(CURDATE()) AND MONTH(date_vente) = MONTH(CURDATE())), 'EVENT'),
        ('nb_clients', (SELECT COUNT(*) FROM clients WHERE statut = 'ACTIF'), 'EVENT')
    ON DUPLICATE KEY UPDATE
        metric_value = VALUES(metric_value),
        last_update = NOW(),
        update_source = VALUES(update_source);
END//

DELIMITER ;

-- 3. Trigger pour mise à jour immédiate du CA du jour
DELIMITER //

CREATE TRIGGER trg_ventes_update_ca_jour
AFTER INSERT ON ventes
FOR EACH ROW
BEGIN
    IF DATE(NEW.date_vente) = CURDATE() THEN
        INSERT INTO cache_dashboard (metric_key, metric_value, update_source)
        VALUES ('ca_jour', NEW.montant, 'TRIGGER')
        ON DUPLICATE KEY UPDATE
            metric_value = metric_value + NEW.montant,
            last_update = NOW(),
            update_source = 'TRIGGER';
    END IF;
END//

DELIMITER ;

-- 4. Procédure pour rafraîchissement manuel si besoin
DELIMITER //

CREATE PROCEDURE sp_refresh_dashboard_manual()
BEGIN
    INSERT INTO cache_dashboard (metric_key, metric_value, update_source)
    VALUES
        ('ca_jour', (SELECT IFNULL(SUM(montant), 0) FROM ventes WHERE DATE(date_vente) = CURDATE()), 'MANUAL'),
        ('ca_mois', (SELECT IFNULL(SUM(montant), 0) FROM ventes WHERE YEAR(date_vente) = YEAR(CURDATE()) AND MONTH(date_vente) = MONTH(CURDATE())), 'MANUAL')
    ON DUPLICATE KEY UPDATE
        metric_value = VALUES(metric_value),
        last_update = NOW(),
        update_source = VALUES(update_source);
END//

DELIMITER ;
```

Cette approche hybride offre :
- ✅ Rafraîchissement complet quotidien (EVENT)
- ✅ Mise à jour temps réel pour les métriques critiques (TRIGGER)
- ✅ Possibilité de rafraîchissement manuel (PROCEDURE)

---

## Optimisation et bonnes pratiques

### 1. Indexer correctement les tables de cache

```sql
-- Les tables de cache doivent avoir des index appropriés
CREATE TABLE cache_produits_stats (
    produit_id INT PRIMARY KEY,
    categorie_id INT,
    nb_ventes INT,
    ca_total DECIMAL(15,2),
    derniere_vente DATE,
    INDEX idx_categorie (categorie_id),
    INDEX idx_ca_total (ca_total DESC),
    INDEX idx_derniere_vente (derniere_vente DESC)
) ENGINE=InnoDB;

-- Les requêtes seront alors ultra-rapides
SELECT * FROM cache_produits_stats
WHERE categorie_id = 5
ORDER BY ca_total DESC
LIMIT 10;
```

### 2. Ajouter une colonne de timestamp

Toujours inclure une colonne indiquant la dernière mise à jour :

```sql
CREATE TABLE cache_table (
    -- ... colonnes de données
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_derniere_maj (derniere_maj)
) ENGINE=InnoDB;

-- Permet de vérifier la fraîcheur des données
SELECT MAX(derniere_maj) AS derniere_mise_a_jour
FROM cache_table;
```

### 3. Prévoir une table de métadonnées

```sql
CREATE TABLE cache_metadata (
    cache_name VARCHAR(100) PRIMARY KEY,
    source_tables VARCHAR(500),
    refresh_frequency VARCHAR(50),
    last_refresh TIMESTAMP,
    next_refresh TIMESTAMP,
    refresh_duration_seconds INT,
    rows_count INT,
    refresh_method ENUM('EVENT', 'TRIGGER', 'MANUAL', 'EXTERNAL'),
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- Enregistrer les métadonnées
INSERT INTO cache_metadata VALUES
('cache_statistiques_ventes', 'commandes, clients', 'EVERY 1 HOUR', NOW(), DATE_ADD(NOW(), INTERVAL 1 HOUR), 45, 12000, 'EVENT', TRUE);
```

### 4. Gérer les transactions pour la cohérence

Pour les rafraîchissements complexes, utilisez des transactions :

```sql
DELIMITER //

CREATE PROCEDURE sp_refresh_cache_complex()
BEGIN
    DECLARE exit handler FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        INSERT INTO log_errors (procedure_name, error_time, error_message)
        VALUES ('sp_refresh_cache_complex', NOW(), 'Erreur lors du rafraîchissement');
    END;

    START TRANSACTION;

    -- Étape 1
    TRUNCATE TABLE cache_temp;
    INSERT INTO cache_temp SELECT ... FROM ...;

    -- Étape 2
    DELETE FROM cache_final;
    INSERT INTO cache_final SELECT * FROM cache_temp;

    -- Étape 3 : Mise à jour métadonnées
    UPDATE cache_metadata
    SET last_refresh = NOW(),
        rows_count = (SELECT COUNT(*) FROM cache_final)
    WHERE cache_name = 'cache_final';

    COMMIT;
END//

DELIMITER ;
```

### 5. Utiliser des tables partitionnées pour les caches volumineux

```sql
-- Partitionnement par mois pour un cache de ventes
CREATE TABLE cache_ventes_mensuelles (
    mois DATE,
    produit_id INT,
    nb_ventes INT,
    ca_total DECIMAL(15,2),
    PRIMARY KEY (mois, produit_id)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(mois) * 100 + MONTH(mois)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    PARTITION p202403 VALUES LESS THAN (202404),
    -- ...
    PARTITION p202412 VALUES LESS THAN (202501),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Rafraîchir uniquement une partition
ALTER TABLE cache_ventes_mensuelles TRUNCATE PARTITION p202412;
INSERT INTO cache_ventes_mensuelles
SELECT ... WHERE YEAR(mois) = 2024 AND MONTH(mois) = 12;
```

### 6. Monitorer les performances de rafraîchissement

```sql
-- Créer une table de monitoring
CREATE TABLE cache_refresh_monitoring (
    id INT AUTO_INCREMENT PRIMARY KEY,
    cache_name VARCHAR(100),
    refresh_start TIMESTAMP,
    refresh_end TIMESTAMP,
    duration_seconds INT GENERATED ALWAYS AS (TIMESTAMPDIFF(SECOND, refresh_start, refresh_end)) STORED,
    rows_before INT,
    rows_after INT,
    refresh_method ENUM('FULL', 'INCREMENTAL'),
    status ENUM('SUCCESS', 'ERROR', 'PARTIAL'),
    INDEX idx_cache_date (cache_name, refresh_start)
) ENGINE=InnoDB;

-- Utiliser dans les procédures
DELIMITER //

CREATE PROCEDURE sp_refresh_monitored()
BEGIN
    DECLARE v_start TIMESTAMP;
    DECLARE v_rows_before INT;
    DECLARE v_rows_after INT;

    SET v_start = NOW();
    SELECT COUNT(*) INTO v_rows_before FROM cache_table;

    -- Rafraîchissement
    TRUNCATE TABLE cache_table;
    INSERT INTO cache_table SELECT ... FROM ...;

    SELECT COUNT(*) INTO v_rows_after FROM cache_table;

    -- Log
    INSERT INTO cache_refresh_monitoring
        (cache_name, refresh_start, refresh_end, rows_before, rows_after, refresh_method, status)
    VALUES
        ('cache_table', v_start, NOW(), v_rows_before, v_rows_after, 'FULL', 'SUCCESS');
END//

DELIMITER ;
```

---

## Cas d'usage pratiques en production

### Cas 1 : Dashboard temps réel de ventes

**Besoin** : Afficher le CA du jour en temps réel avec des performances optimales.

**Solution** : Trigger + cache

```sql
-- Table de cache ultra-simple
CREATE TABLE cache_ca_jour (
    date_jour DATE PRIMARY KEY,
    ca_total DECIMAL(15,2) DEFAULT 0,
    nb_ventes INT DEFAULT 0,
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Trigger sur insertion
DELIMITER //

CREATE TRIGGER trg_ventes_ca_jour_insert
AFTER INSERT ON ventes
FOR EACH ROW
BEGIN
    INSERT INTO cache_ca_jour (date_jour, ca_total, nb_ventes)
    VALUES (DATE(NEW.date_vente), NEW.montant, 1)
    ON DUPLICATE KEY UPDATE
        ca_total = ca_total + NEW.montant,
        nb_ventes = nb_ventes + 1,
        derniere_maj = NOW();
END//

DELIMITER ;

-- Requête ultra-rapide
SELECT ca_total, nb_ventes, derniere_maj
FROM cache_ca_jour
WHERE date_jour = CURDATE();
```

### Cas 2 : Rapport mensuel de top produits

**Besoin** : Générer un rapport des 100 meilleurs produits par mois, consulté plusieurs fois par jour.

**Solution** : EVENT quotidien + table de cache

```sql
-- Table de cache
CREATE TABLE cache_top_produits_mois (
    mois DATE,
    produit_id INT,
    produit_nom VARCHAR(255),
    nb_ventes INT,
    ca_total DECIMAL(15,2),
    rang INT,
    PRIMARY KEY (mois, produit_id),
    INDEX idx_mois_rang (mois, rang)
) ENGINE=InnoDB;

-- Procédure de rafraîchissement
DELIMITER //

CREATE PROCEDURE sp_refresh_top_produits()
BEGIN
    DECLARE v_mois DATE;
    SET v_mois = DATE_FORMAT(CURDATE(), '%Y-%m-01');

    -- Supprimer uniquement le mois courant
    DELETE FROM cache_top_produits_mois WHERE mois = v_mois;

    -- Recalculer
    INSERT INTO cache_top_produits_mois (mois, produit_id, produit_nom, nb_ventes, ca_total, rang)
    SELECT
        v_mois,
        p.id,
        p.nom,
        COUNT(*) AS nb_ventes,
        SUM(v.montant) AS ca_total,
        RANK() OVER (ORDER BY SUM(v.montant) DESC) AS rang
    FROM ventes v
    INNER JOIN produits p ON v.produit_id = p.id
    WHERE DATE_FORMAT(v.date_vente, '%Y-%m-01') = v_mois
    GROUP BY p.id, p.nom
    ORDER BY ca_total DESC
    LIMIT 100;
END//

DELIMITER ;

-- EVENT quotidien à 2h du matin
CREATE EVENT evt_refresh_top_produits
ON SCHEDULE EVERY 1 DAY
STARTS TIMESTAMP(CURRENT_DATE + INTERVAL 1 DAY, '02:00:00')
DO
    CALL sp_refresh_top_produits();
```

### Cas 3 : Statistiques par client pour application web

**Besoin** : Afficher les statistiques de chaque client (nb commandes, CA total, dernière commande) sur leur profil.

**Solution** : Table de cache + EVENT incrémental toutes les 15 minutes

```sql
-- Table de cache
CREATE TABLE cache_stats_clients (
    client_id INT PRIMARY KEY,
    nb_commandes INT DEFAULT 0,
    ca_total DECIMAL(15,2) DEFAULT 0,
    derniere_commande TIMESTAMP NULL,
    panier_moyen DECIMAL(10,2) AS (ca_total / NULLIF(nb_commandes, 0)) STORED,
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_ca_total (ca_total DESC),
    INDEX idx_derniere_commande (derniere_commande DESC)
) ENGINE=InnoDB;

-- Procédure de rafraîchissement incrémental
DELIMITER //

CREATE PROCEDURE sp_refresh_stats_clients_incremental()
BEGIN
    -- Mettre à jour uniquement les clients avec nouvelles commandes (derniers 20 min)
    INSERT INTO cache_stats_clients (client_id, nb_commandes, ca_total, derniere_commande)
    SELECT
        client_id,
        COUNT(*) AS nb_commandes,
        SUM(montant_ttc) AS ca_total,
        MAX(date_commande) AS derniere_commande
    FROM commandes
    WHERE client_id IN (
        SELECT DISTINCT client_id
        FROM commandes
        WHERE date_modification >= DATE_SUB(NOW(), INTERVAL 20 MINUTE)
    )
    GROUP BY client_id
    ON DUPLICATE KEY UPDATE
        nb_commandes = VALUES(nb_commandes),
        ca_total = VALUES(ca_total),
        derniere_commande = VALUES(derniere_commande),
        derniere_maj = NOW();
END//

DELIMITER ;

-- EVENT toutes les 15 minutes
CREATE EVENT evt_refresh_stats_clients
ON SCHEDULE EVERY 15 MINUTE
DO
    CALL sp_refresh_stats_clients_incremental();

-- Rafraîchissement complet une fois par jour pour sécurité
CREATE EVENT evt_refresh_stats_clients_full
ON SCHEDULE EVERY 1 DAY
STARTS TIMESTAMP(CURRENT_DATE + INTERVAL 1 DAY, '04:00:00')
DO
BEGIN
    TRUNCATE TABLE cache_stats_clients;

    INSERT INTO cache_stats_clients (client_id, nb_commandes, ca_total, derniere_commande)
    SELECT
        client_id,
        COUNT(*) AS nb_commandes,
        SUM(montant_ttc) AS ca_total,
        MAX(date_commande) AS derniere_commande
    FROM commandes
    GROUP BY client_id;
END;
```

---

## ⚠️ Pièges à éviter

### 1. Ne pas gérer la cohérence pendant le rafraîchissement

```sql
-- ❌ Problème : données incohérentes pendant le rafraîchissement
DELETE FROM cache_table;
-- ⚠️ Entre DELETE et INSERT, la table est vide !
INSERT INTO cache_table SELECT ... FROM ...;

-- ✅ Solution 1 : Utiliser une table temporaire
CREATE TEMPORARY TABLE cache_temp AS
SELECT ... FROM ...;

DELETE FROM cache_table;
INSERT INTO cache_table SELECT * FROM cache_temp;
DROP TEMPORARY TABLE cache_temp;

-- ✅ Solution 2 : TRUNCATE (plus rapide mais bloque la table)
TRUNCATE TABLE cache_table;
INSERT INTO cache_table SELECT ... FROM ...;

-- ✅ Solution 3 : Table double (swap)
CREATE TABLE cache_table_new AS SELECT ... FROM ...;
RENAME TABLE cache_table TO cache_table_old, cache_table_new TO cache_table;
DROP TABLE cache_table_old;
```

### 2. Oublier de gérer les erreurs dans les triggers

```sql
-- ❌ Trigger sans gestion d'erreur
CREATE TRIGGER trg_update_cache
AFTER INSERT ON table_source
FOR EACH ROW
BEGIN
    UPDATE cache_table SET ...;  -- Si erreur, la transaction entière échoue !
END;

-- ✅ Trigger avec gestion d'erreur
DELIMITER //

CREATE TRIGGER trg_update_cache_safe
AFTER INSERT ON table_source
FOR EACH ROW
BEGIN
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Logger l'erreur au lieu de bloquer
        INSERT IGNORE INTO log_trigger_errors (trigger_name, error_time, error_data)
        VALUES ('trg_update_cache_safe', NOW(), CONCAT('Error for ID: ', NEW.id));
    END;

    UPDATE cache_table SET ... WHERE ...;
END//

DELIMITER ;
```

### 3. Surcharger la base avec trop de triggers

```sql
-- ❌ Mauvaise pratique : 10 triggers sur la même table
CREATE TRIGGER trg_1 AFTER INSERT ON ventes ...
CREATE TRIGGER trg_2 AFTER INSERT ON ventes ...
CREATE TRIGGER trg_3 AFTER INSERT ON ventes ...
-- Chaque INSERT exécute 10 triggers !

-- ✅ Solution : Regrouper la logique dans un seul trigger
CREATE TRIGGER trg_ventes_all
AFTER INSERT ON ventes
FOR EACH ROW
BEGIN
    -- Mise à jour cache 1
    UPDATE cache1 SET ...;

    -- Mise à jour cache 2
    UPDATE cache2 SET ...;

    -- Mise à jour cache 3
    UPDATE cache3 SET ...;
END;
```

### 4. Ne pas surveiller les EVENT qui échouent silencieusement

```sql
-- ❌ EVENT sans monitoring
CREATE EVENT evt_refresh
ON SCHEDULE EVERY 1 HOUR
DO
    INSERT INTO cache SELECT ... FROM ...;
-- Si erreur, l'EVENT échoue silencieusement !

-- ✅ EVENT avec monitoring et notification
DELIMITER //

CREATE EVENT evt_refresh_monitored
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        INSERT INTO log_events (event_name, status, error_time)
        VALUES ('evt_refresh_monitored', 'ERROR', NOW());

        -- Optionnel : notifier via table consultée par un service externe
        INSERT INTO alert_queue (alert_type, message, created_at)
        VALUES ('EVENT_FAILED', 'evt_refresh_monitored failed', NOW());
    END;

    INSERT INTO cache SELECT ... FROM ...;

    INSERT INTO log_events (event_name, status, execution_time, rows_affected)
    VALUES ('evt_refresh_monitored', 'SUCCESS', NOW(), ROW_COUNT());
END//

DELIMITER ;
```

### 5. Ne pas prévoir de stratégie de rollback

```sql
-- ✅ Toujours garder une sauvegarde avant rafraîchissement complet
CREATE PROCEDURE sp_refresh_cache_safe()
BEGIN
    -- 1. Créer une sauvegarde
    DROP TABLE IF EXISTS cache_table_backup;
    CREATE TABLE cache_table_backup AS SELECT * FROM cache_table;

    -- 2. Rafraîchir
    TRUNCATE TABLE cache_table;
    INSERT INTO cache_table SELECT ... FROM ...;

    -- 3. Vérifier l'intégrité
    IF (SELECT COUNT(*) FROM cache_table) = 0 THEN
        -- Rollback si problème
        INSERT INTO cache_table SELECT * FROM cache_table_backup;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Cache refresh failed: no data';
    ELSE
        -- Succès : supprimer la sauvegarde
        DROP TABLE cache_table_backup;
    END IF;
END;
```

---

## ✅ Points clés à retenir

- **MariaDB ne supporte pas nativement les vues matérialisées** contrairement à PostgreSQL ou Oracle
- Les **tables de cache** sont la solution la plus simple et flexible pour simuler les vues matérialisées
- Les **triggers** permettent une mise à jour en temps réel mais ajoutent de la surcharge aux écritures
- Les **EVENTs** offrent une automatisation complète pour des rafraîchissements périodiques
- Le choix de l'approche dépend du **compromis fraîcheur vs performance** requis
- Une **approche hybride** (EVENT + trigger) est souvent optimale en production
- Toujours **indexer** correctement les tables de cache pour garantir les performances en lecture
- Inclure une **colonne timestamp** (derniere_maj) pour tracer les rafraîchissements
- Prévoir une **table de métadonnées** pour documenter les caches et leur maintenance
- **Monitorer** les performances de rafraîchissement et les erreurs
- Gérer la **cohérence** pendant les rafraîchissements (transactions, tables temporaires)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 CREATE EVENT](https://mariadb.com/kb/en/create-event/) - Événements planifiés
- [📖 Event Scheduler](https://mariadb.com/kb/en/event-scheduler/) - Configuration du scheduler
- [📖 Triggers](https://mariadb.com/kb/en/triggers/) - Documentation complète sur les triggers
- [📖 CREATE TRIGGER](https://mariadb.com/kb/en/create-trigger/) - Syntaxe des triggers

### Articles complémentaires
- **"Implementing Materialized Views in MySQL/MariaDB"** - Percona Blog
- **"Event Scheduler vs Cron: Which to Choose?"** - Database Journal
- **"Trigger Performance Best Practices"** - MariaDB Blog

### Comparaison avec d'autres SGBD
- **PostgreSQL Materialized Views** - Pour comprendre ce que MariaDB ne fait pas
- **Oracle Materialized Views** - Inspiration pour les implémentations

---

## ➡️ Section suivante

**[9.3 Vues updatable : Conditions et limitations](./03-vues-updatable.md)** : Découvrez quelles vues permettent les opérations INSERT, UPDATE et DELETE, les conditions strictes imposées par MariaDB, comment vérifier si une vue est updatable, et les limitations à connaître.

---


⏭️ [Vues updatable : Conditions et limitations](/09-vues-et-donnees-virtuelles/03-vues-updatable.md)

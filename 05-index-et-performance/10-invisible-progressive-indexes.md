🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.10 Invisible indexes et Progressive indexes

> **Niveau** : Intermédiaire
> **Durée estimée** : 1.5 heures
> **Prérequis** : Section 5.1 à 5.9 (Index et optimisation)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Utiliser les invisible indexes pour tester l'impact d'un index sans le supprimer
- Comprendre la construction progressive d'index (online DDL)
- Minimiser l'impact des opérations DDL sur les applications en production
- Gérer les index en production sans interruption de service
- Appliquer les bonnes pratiques de création/suppression d'index
- Mesurer l'impact réel d'un index avant décision finale
- Utiliser les algorithmes INPLACE et COPY pour les modifications de schéma

---

## Introduction

La gestion des index en production est délicate : créer un index peut bloquer la table pendant plusieurs minutes, supprimer un index peut dégrader les performances. Les **invisible indexes** et la **construction progressive** sont deux techniques essentielles pour gérer les index en production **sans risque** et **sans interruption**.

💡 **Problématique** : Comment tester si un index est vraiment nécessaire sans risquer de dégrader les performances en le supprimant ? Comment créer un index volumineux sans bloquer les écritures ?

⚠️ **Enjeu production** : Sur une table de 100M de lignes, créer un index peut prendre 30 minutes à 2 heures. Pendant ce temps, selon l'algorithme utilisé, les écritures peuvent être bloquées ou ralenties.

---

## Invisible Indexes : Tester sans supprimer

### Concept des index invisibles

Un **invisible index** (disponible depuis MariaDB 10.0) est un index qui :
- ✅ **Existe physiquement** en mémoire et sur disque
- ✅ **Est maintenu à jour** lors des INSERT/UPDATE/DELETE
- ❌ **N'est PAS utilisé par l'optimiseur** pour les requêtes
- ✅ **Peut être rendu visible instantanément**

```sql
-- Créer un index invisible dès le départ
CREATE INDEX idx_users_country ON users(country) INVISIBLE;

-- Ou rendre un index existant invisible
ALTER TABLE users ALTER INDEX idx_users_country INVISIBLE;

-- Rendre à nouveau visible
ALTER TABLE users ALTER INDEX idx_users_country VISIBLE;
```

**Cas d'usage** : Tester l'impact de la suppression d'un index sans le supprimer réellement.

### Workflow de test avec invisible indexes

#### Étape 1 : Identifier un index potentiellement inutile

```sql
-- Analyser l'utilisation des index avec Performance Schema
SELECT
    object_schema,
    object_name,
    index_name,
    count_star as usage_count,
    sum_timer_wait / 1000000000 as total_time_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'mydb'
  AND object_name = 'products'
  AND index_name != 'PRIMARY'
ORDER BY count_star ASC;

-- Résultat exemple :
-- idx_products_legacy : 0 utilisations depuis 30 jours
-- idx_products_brand : 150 utilisations
-- idx_products_category : 1,500,000 utilisations
```

💡 **Index candidat à la suppression** : `idx_products_legacy` avec 0 utilisations.

#### Étape 2 : Rendre l'index invisible

```sql
-- Marquer comme invisible
ALTER TABLE products ALTER INDEX idx_products_legacy INVISIBLE;

-- Vérifier le statut
SELECT
    TABLE_NAME,
    INDEX_NAME,
    IS_VISIBLE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'products'
  AND INDEX_NAME = 'idx_products_legacy';

-- Résultat :
-- products | idx_products_legacy | NO
```

**Effet immédiat** : L'optimiseur ne considère plus cet index pour aucune requête.

#### Étape 3 : Surveiller les performances (1-2 semaines)

```sql
-- Surveiller les requêtes lentes qui pourraient utiliser cet index
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    AVG_TIMER_WAIT / 1000000000 as avg_time_sec,
    MAX_TIMER_WAIT / 1000000000 as max_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'mydb'
  AND AVG_TIMER_WAIT / 1000000000 > 1  -- Requêtes > 1 seconde
ORDER BY COUNT_STAR DESC
LIMIT 20;

-- Comparer avec les métriques avant invisible
```

**Questions à se poser** :
- Les temps de requêtes ont-ils augmenté ?
- Y a-t-il de nouvelles requêtes lentes ?
- Les utilisateurs signalent-ils des ralentissements ?

#### Étape 4 : Décision finale

**Scénario A : Aucun impact négatif détecté** (index vraiment inutile)
```sql
-- Supprimer définitivement l'index
DROP INDEX idx_products_legacy ON products;

-- Libère l'espace disque et réduit le coût de maintenance
```

**Scénario B : Impact négatif détecté** (index nécessaire)
```sql
-- Rétablir immédiatement la visibilité
ALTER TABLE products ALTER INDEX idx_products_legacy VISIBLE;

-- Analyse pourquoi l'index est utilisé malgré les statistiques
EXPLAIN SELECT ... WHERE ...;
```

### Avantages des invisible indexes

| Avantage | Description |
|----------|-------------|
| **Réversibilité instantanée** | ALTER INDEX VISIBLE = <1ms, vs recréer index = heures |
| **Pas de perte de données** | Index maintenu même invisible |
| **Test sécurisé** | Pas de risque de dégradation permanente |
| **Maintenance continue** | Index à jour si besoin de réactiver |

### Limitations des invisible indexes

```sql
-- ❌ Les index PRIMARY KEY et UNIQUE ne peuvent PAS être invisibles
ALTER TABLE users ALTER INDEX PRIMARY INVISIBLE;
-- ERROR: Primary key cannot be invisible

-- ❌ Les contraintes FOREIGN KEY nécessitent index visible
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    INDEX idx_customer (customer_id) INVISIBLE
);
-- Fonctionne à la création mais peut causer des problèmes
```

**Recommandation** : N'utiliser INVISIBLE que pour les index secondaires non-contraintes.

---

## Progressive Indexes : Construction en ligne

### Online DDL dans MariaDB

MariaDB supporte plusieurs algorithmes pour modifier les tables sans (ou avec minimal) verrouillage :

```sql
-- Syntaxe générale
ALTER TABLE table_name
    ADD INDEX index_name (columns),
    ALGORITHM = {DEFAULT | INPLACE | COPY},
    LOCK = {DEFAULT | NONE | SHARED | EXCLUSIVE};
```

**Algorithmes disponibles** :

| Algorithme | Description | Verrouillage | Performance |
|------------|-------------|--------------|-------------|
| **COPY** | Copie toute la table | Exclusive | Lent, bloquant |
| **INPLACE** | Modifie sur place | Minimal | Rapide, non-bloquant |
| **DEFAULT** | MariaDB choisit | Variable | Optimal |

### ALGORITHM=INPLACE : Construction progressive

```sql
-- Créer un index sans bloquer les écritures
ALTER TABLE products
ADD INDEX idx_products_category (category_id),
ALGORITHM=INPLACE,
LOCK=NONE;

-- MariaDB construit l'index en arrière-plan
-- Les INSERT/UPDATE/DELETE continuent de fonctionner
```

**Processus interne** :

1. **Phase 1 : Préparation** (< 1s)
   - Verrouillage metadata très bref
   - Crée structure d'index vide

2. **Phase 2 : Construction** (minutes à heures)
   - Lit la table et construit l'index
   - Les écritures continuent normalement
   - Les nouvelles modifications sont loggées

3. **Phase 3 : Application du log** (secondes)
   - Applique les modifications accumulées pendant construction
   - Verrouillage très bref

4. **Phase 4 : Finalisation** (< 1s)
   - Index activé
   - Verrouillage metadata très bref

**Durée totale** : Selon taille table
- 1M lignes : 30 secondes - 2 minutes
- 10M lignes : 5-15 minutes
- 100M lignes : 30 minutes - 2 heures

💡 **Impact sur les écritures** : Ralentissement de 5-15% pendant la construction (acceptable en production).

### Options de verrouillage

```sql
-- LOCK=NONE : Aucun verrou (optimal)
ALTER TABLE products
ADD INDEX idx_category (category_id),
ALGORITHM=INPLACE,
LOCK=NONE;
-- Lecture ET écriture continuent

-- LOCK=SHARED : Lectures autorisées, écritures bloquées
ALTER TABLE products
ADD INDEX idx_category (category_id),
ALGORITHM=INPLACE,
LOCK=SHARED;
-- SELECT OK, INSERT/UPDATE/DELETE bloqués

-- LOCK=EXCLUSIVE : Tout bloqué
ALTER TABLE products
ADD INDEX idx_category (category_id),
ALGORITHM=COPY,
LOCK=EXCLUSIVE;
-- Table complètement verrouillée
```

**Décision** :
- Production avec trafic : `LOCK=NONE` obligatoire
- Maintenance window : `LOCK=SHARED` acceptable
- Urgence hors heures : `LOCK=EXCLUSIVE` si nécessaire

### Opérations supportées par INPLACE

```sql
-- ✅ Supporté avec ALGORITHM=INPLACE, LOCK=NONE

-- Ajouter index
ALTER TABLE products ADD INDEX idx_name (name);

-- Supprimer index (sauf PRIMARY KEY)
ALTER TABLE products DROP INDEX idx_old;

-- Renommer index
ALTER TABLE products RENAME INDEX idx_old TO idx_new;

-- Ajouter colonne à la fin
ALTER TABLE products ADD COLUMN new_column INT;

-- Changer AUTO_INCREMENT
ALTER TABLE products AUTO_INCREMENT = 10000;

-- ⚠️ Supporté avec ALGORITHM=INPLACE, mais LOCK=SHARED

-- Ajouter clé étrangère
ALTER TABLE orders
ADD CONSTRAINT fk_customer
FOREIGN KEY (customer_id) REFERENCES customers(customer_id);

-- ❌ NON supporté avec INPLACE (nécessite COPY)

-- Modifier type de colonne (augmenter taille)
ALTER TABLE products MODIFY COLUMN name VARCHAR(500);

-- Ajouter colonne au milieu
ALTER TABLE products ADD COLUMN new_col INT AFTER existing_col;

-- Changer charset/collation de table
ALTER TABLE products CONVERT TO CHARACTER SET utf8mb4;
```

💡 **Vérifier avant exécution** :

```sql
-- Tester la compatibilité INPLACE
ALTER TABLE products
ADD INDEX idx_test (column),
ALGORITHM=INPLACE,
LOCK=NONE;

-- Si erreur "ALGORITHM=INPLACE not supported"
-- → nécessite COPY (bloquant)
```

---

## Stratégies de création d'index en production

### Stratégie 1 : Heures creuses

```sql
-- Planifier pendant nuit ou weekend
-- Fenêtre de maintenance : 2h00-6h00

-- 1. Annoncer la maintenance
-- 2. Réduire le trafic (mise en maintenance de certaines fonctionnalités)

-- 3. Créer l'index avec INPLACE
ALTER TABLE large_table
ADD INDEX idx_new (columns),
ALGORITHM=INPLACE,
LOCK=NONE;

-- 4. Surveiller la progression
SHOW PROCESSLIST;
-- Rechercher "copy to tmp table" ou "alter table"

-- 5. Valider après création
SHOW INDEX FROM large_table;
EXPLAIN SELECT ... -- vérifier utilisation
```

### Stratégie 2 : Réplica puis promotion

Pour tables **très volumineuses** (>1B lignes), créer l'index sur un réplica.

```sql
-- Architecture : Primary → Replica

-- Étape 1 : Sur le REPLICA (pas de trafic utilisateur)
-- Arrêter la réplication temporairement
STOP SLAVE;

-- Créer l'index
ALTER TABLE huge_table
ADD INDEX idx_new (columns);
-- Peut prendre plusieurs heures, pas de problème

-- Redémarrer réplication
START SLAVE;

-- Étape 2 : Failover vers le Replica
-- Le replica devient Primary (avec index déjà créé)

-- Étape 3 : Sur l'ancien Primary (devenu Replica)
-- Créer l'index à son tour
ALTER TABLE huge_table
ADD INDEX idx_new (columns);
```

**Avantage** : Aucun impact sur production pendant création initiale.

### Stratégie 3 : Throttling avec pt-online-schema-change

Pour MariaDB < 10.0 ou DDL complexes, utiliser **pt-online-schema-change** (Percona Toolkit).

```bash
# Installation
sudo apt-get install percona-toolkit

# Créer index avec throttling
pt-online-schema-change \
  --alter "ADD INDEX idx_products_category (category_id)" \
  --execute \
  --max-load="Threads_running=50" \
  --critical-load="Threads_running=100" \
  --chunk-size=1000 \
  --chunk-time=0.5 \
  D=mydb,t=products

# Options :
# --max-load : pause si charge trop élevée
# --critical-load : arrêt si charge critique
# --chunk-size : lignes par batch
# --chunk-time : secondes par batch
```

**Processus** :
1. Crée table temporaire avec nouvel index
2. Copie données par petits lots (chunks)
3. Installe triggers pour synchroniser modifications
4. Swap atomique des tables à la fin

**Avantages** :
- ✅ Contrôle fin de la charge
- ✅ Peut être annulé en cours
- ✅ Minimal impact sur production

**Inconvénients** :
- ❌ Plus lent que ALTER TABLE natif
- ❌ Nécessite espace disque × 2

---

## Gestion des index : bonnes pratiques

### Audit régulier des index

```sql
-- Script d'audit mensuel

-- 1. Index inutilisés (candidats à suppression)
SELECT
    CONCAT(TABLE_SCHEMA, '.', TABLE_NAME) as table_name,
    INDEX_NAME,
    ROUND(STAT_VALUE * @@innodb_page_size / 1024 / 1024, 2) as size_mb
FROM mysql.innodb_index_stats
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND INDEX_NAME != 'PRIMARY'
  AND INDEX_NAME NOT IN (
      SELECT DISTINCT index_name
      FROM performance_schema.table_io_waits_summary_by_index_usage
      WHERE count_star > 0
  )
ORDER BY size_mb DESC;

-- 2. Index redondants
SELECT
    t1.TABLE_SCHEMA,
    t1.TABLE_NAME,
    t1.INDEX_NAME as redundant_index,
    GROUP_CONCAT(t1.COLUMN_NAME ORDER BY t1.SEQ_IN_INDEX) as redundant_columns,
    t2.INDEX_NAME as covering_index,
    GROUP_CONCAT(t2.COLUMN_NAME ORDER BY t2.SEQ_IN_INDEX) as covering_columns
FROM INFORMATION_SCHEMA.STATISTICS t1
JOIN INFORMATION_SCHEMA.STATISTICS t2
    ON t1.TABLE_SCHEMA = t2.TABLE_SCHEMA
    AND t1.TABLE_NAME = t2.TABLE_NAME
    AND t1.INDEX_NAME != t2.INDEX_NAME
    AND t1.SEQ_IN_INDEX = 1
    AND t2.SEQ_IN_INDEX = 1
    AND t1.COLUMN_NAME = t2.COLUMN_NAME
WHERE t1.TABLE_SCHEMA = 'mydb'
GROUP BY t1.TABLE_SCHEMA, t1.TABLE_NAME, t1.INDEX_NAME, t2.INDEX_NAME
HAVING COUNT(*) > 0;

-- 3. Index volumineux (coût de maintenance élevé)
SELECT
    TABLE_NAME,
    INDEX_NAME,
    ROUND(SUM(STAT_VALUE * @@innodb_page_size) / 1024 / 1024, 2) as size_mb
FROM mysql.innodb_index_stats
WHERE TABLE_SCHEMA = 'mydb'
GROUP BY TABLE_NAME, INDEX_NAME
HAVING size_mb > 1000  -- Plus de 1GB
ORDER BY size_mb DESC;
```

### Workflow de suppression sécurisée

```sql
-- Processus en 4 étapes

-- Étape 1 : Identifier candidat (audit)
-- Index idx_products_old identifié comme inutilisé

-- Étape 2 : Rendre invisible (test 2 semaines)
ALTER TABLE products ALTER INDEX idx_products_old INVISIBLE;
-- Noter la date : 2025-01-15

-- Étape 3 : Surveillance
-- - Slow query log
-- - Performance Schema
-- - Alertes monitoring
-- - Feedback utilisateurs

-- Étape 4a : Si OK → Supprimer (après 2-4 semaines)
DROP INDEX idx_products_old ON products;

-- Étape 4b : Si problème → Restaurer immédiatement
ALTER TABLE products ALTER INDEX idx_products_old VISIBLE;
```

### Documentation des index

```sql
-- Créer table de documentation
CREATE TABLE index_documentation (
    id INT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(64),
    index_name VARCHAR(64),
    purpose TEXT,
    created_date DATE,
    created_by VARCHAR(100),
    usage_queries TEXT,
    review_date DATE,
    status ENUM('active', 'invisible', 'deprecated'),
    notes TEXT,
    UNIQUE KEY (table_name, index_name)
);

-- Documenter chaque index
INSERT INTO index_documentation VALUES
(NULL, 'products', 'idx_products_category',
 'Optimise les recherches par catégorie (page catalogue)',
 '2024-06-15', 'dba_team',
 'SELECT * FROM products WHERE category_id = ?',
 '2025-06-15', 'active',
 'Index covering incluant price, name pour éviter accès table');
```

---

## Monitoring de la construction d'index

### Suivre la progression

```sql
-- Voir les ALTER TABLE en cours
SHOW PROCESSLIST;

-- Résultat :
-- Id | Command | State                  | Info
-- 42 | Query   | copy to tmp table      | ALTER TABLE products ADD INDEX...
--
-- État "copy to tmp table" = construction en cours

-- Détails supplémentaires (MariaDB 10.3+)
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE,
    INFO
FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE STATE LIKE '%alter%' OR STATE LIKE '%copy%';
```

### Estimer le temps restant

```sql
-- Pour tables InnoDB : estimer via taille
SELECT
    TABLE_NAME,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) as data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) as index_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'products';

-- Estimation empirique :
-- - 1 Go de données ≈ 2-5 minutes (SSD)
-- - 1 Go de données ≈ 10-20 minutes (HDD)
-- - Dépend de : CPU, I/O, charge serveur
```

### Annuler une construction d'index

```sql
-- Si nécessaire (urgence), tuer le processus
SHOW PROCESSLIST;
-- Noter l'ID du processus ALTER TABLE

KILL 42;  -- Remplacer 42 par l'ID réel

-- ⚠️ Attention :
-- - InnoDB rollback automatiquement
-- - Peut prendre du temps (proportionnel au travail fait)
-- - Index partiellement créé est supprimé
```

---

## Cas d'usage avancés

### Cas 1 : Migration progressive de plusieurs index

```sql
-- Scénario : refonte complète de l'indexation d'une table

-- État initial : 5 index sous-optimaux
-- Objectif : 3 index covering optimaux

-- Phase 1 : Créer nouveaux index (INVISIBLE)
ALTER TABLE orders
ADD INDEX idx_orders_new1 (status, customer_id, order_date) INVISIBLE,
ADD INDEX idx_orders_new2 (customer_id, order_date DESC, total_amount) INVISIBLE,
ADD INDEX idx_orders_new3 (order_date, status, total_amount) INVISIBLE,
ALGORITHM=INPLACE,
LOCK=NONE;

-- Phase 2 : Tests sur copie de production ou staging
-- Rendre visible sur environnement de test
ALTER TABLE orders ALTER INDEX idx_orders_new1 VISIBLE;
ALTER TABLE orders ALTER INDEX idx_orders_new2 VISIBLE;
ALTER TABLE orders ALTER INDEX idx_orders_new3 VISIBLE;

-- Exécuter suite de tests, valider performances

-- Phase 3 : Activation progressive en production
-- Jour 1 : Activer index 1, surveiller
ALTER TABLE orders ALTER INDEX idx_orders_new1 VISIBLE;

-- Jour 3 : Si OK, activer index 2
ALTER TABLE orders ALTER INDEX idx_orders_new2 VISIBLE;

-- Jour 7 : Si OK, activer index 3
ALTER TABLE orders ALTER INDEX idx_orders_new3 VISIBLE;

-- Phase 4 : Désactivation anciens index
ALTER TABLE orders ALTER INDEX idx_orders_old1 INVISIBLE;
-- Attendre 2 semaines, surveiller

-- Phase 5 : Suppression définitive
DROP INDEX idx_orders_old1 ON orders;
DROP INDEX idx_orders_old2 ON orders;
-- etc.
```

### Cas 2 : A/B testing d'index

```sql
-- Scénario : deux stratégies d'indexation possibles

-- Option A : Index composite large (covering)
CREATE INDEX idx_strategy_a
ON products(category_id, brand_id, price, name, stock);

-- Option B : Plusieurs index ciblés
CREATE INDEX idx_strategy_b1 ON products(category_id, price);
CREATE INDEX idx_strategy_b2 ON products(brand_id, stock);

-- Test A/B sur deux serveurs identiques pendant 1 semaine
-- Comparer :
-- - Temps de réponse moyen
-- - P95, P99 des requêtes
-- - CPU usage
-- - I/O disk
-- - Taille des index

-- Choisir la meilleure stratégie et déployer partout
```

### Cas 3 : Rollback rapide en cas d'incident

```sql
-- Déploiement d'un nouvel index
ALTER TABLE large_table
ADD INDEX idx_new (columns),
ALGORITHM=INPLACE;

-- Après déploiement : incident de performance détecté !
-- Hypothèse : le nouvel index cause le problème

-- ✅ Rollback immédiat (< 1 seconde)
ALTER TABLE large_table ALTER INDEX idx_new INVISIBLE;

-- Vérifier si les performances sont restaurées
-- Si oui : l'index était la cause

-- Si non : autre cause, réactiver l'index
ALTER TABLE large_table ALTER INDEX idx_new VISIBLE;

-- Analyser le vrai problème
```

---

## Comparaison des approches

### Tableau comparatif

| Critère | DROP INDEX | INVISIBLE INDEX | INPLACE ADD | COPY ADD |
|---------|------------|-----------------|-------------|----------|
| **Réversibilité** | ❌ Lente (recréation) | ✅ Instantanée | ❌ Lente | ❌ Lente |
| **Temps d'exécution** | Rapide (secondes) | Instantané | Moyen (minutes) | Lent (heures) |
| **Verrouillage** | Minimal | Aucun | Minimal | Exclusif |
| **Impact production** | Moyen (perte perf) | Aucun | Faible | ❌ Critique |
| **Espace disque** | Libéré | Conservé | +Index | +Table+Index |
| **Cas d'usage** | Suppression certaine | Test réversible | Production moderne | Migration |

### Arbre de décision

```
Besoin de créer/supprimer un index ?
│
├─ Supprimer un index ?
│  │
│  ├─ Certain de son inutilité ?
│  │  └─ Oui → DROP INDEX directement
│  │
│  └─ Doute sur l'impact ?
│     └─ ALTER INDEX INVISIBLE
│        Surveiller 2-4 semaines
│        └─ Si OK → DROP INDEX
│
└─ Créer un index ?
   │
   ├─ Table < 10M lignes ?
   │  └─ ALTER TABLE ADD INDEX (rapide)
   │
   ├─ Table 10M-100M lignes ?
   │  └─ ALGORITHM=INPLACE, LOCK=NONE
   │     (pendant heures creuses)
   │
   ├─ Table > 100M lignes ?
   │  ├─ Fenêtre maintenance disponible ?
   │  │  └─ ALGORITHM=INPLACE, LOCK=NONE
   │  │
   │  └─ Aucune interruption tolérée ?
   │     └─ Créer sur replica puis failover
   │
   └─ DDL complexe (COPY obligatoire) ?
      └─ pt-online-schema-change avec throttling
```

---

## Checklist de gestion des index

### ✅ Avant création

- [ ] **Vérifier nécessité** : EXPLAIN montre que l'index sera utilisé
- [ ] **Estimer taille** : Calculer taille attendue de l'index
- [ ] **Choisir algorithme** : INPLACE si supporté, COPY sinon
- [ ] **Planifier fenêtre** : Heures creuses si table volumineuse
- [ ] **Préparer rollback** : Plan B en cas de problème
- [ ] **Notifier équipe** : Communication si impact attendu

### ✅ Pendant création

- [ ] **Monitorer progression** : SHOW PROCESSLIST
- [ ] **Surveiller charge** : CPU, I/O, mémoire
- [ ] **Vérifier logs** : Erreurs éventuelles
- [ ] **Tester queries** : Valider utilisation si possible

### ✅ Après création

- [ ] **Vérifier présence** : SHOW INDEX FROM table
- [ ] **Valider utilisation** : EXPLAIN sur requêtes cibles
- [ ] **Mesurer impact** : Avant/après sur métriques clés
- [ ] **Documenter** : Table index_documentation
- [ ] **Analyser stats** : ANALYZE TABLE si nécessaire

### ✅ Avant suppression

- [ ] **Rendre invisible** : ALTER INDEX INVISIBLE
- [ ] **Surveiller 2-4 semaines** : Métriques de performance
- [ ] **Vérifier slow queries** : Pas de nouvelle requête lente
- [ ] **Valider avec équipes** : Pas de plaintes utilisateurs
- [ ] **Préparer restauration** : ALTER INDEX VISIBLE si besoin

---

## ✅ Points clés à retenir

- **Invisible indexes** = test réversible de suppression (ALTER INDEX INVISIBLE)
- **Réactivation instantanée** : < 1ms vs recréation de plusieurs heures
- **ALGORITHM=INPLACE** = construction progressive sans verrouillage
- **LOCK=NONE** = écritures continuent pendant création (ralentissement 5-15%)
- **Workflow sécurisé** : invisible → surveiller → décider → supprimer
- **Tables volumineuses** : créer sur replica puis promouvoir
- **Audit régulier** : identifier index inutilisés et redondants
- **Documentation** : tracer l'historique et la raison de chaque index
- **Mesurer toujours** : EXPLAIN ANALYZE avant/après

---

## 🔗 Ressources et références

- [📖 MariaDB Invisible Columns and Indexes](https://mariadb.com/kb/en/invisible-columns/)
- [📖 ALTER TABLE Online Operations](https://mariadb.com/kb/en/alter-table/#online-operations)
- [📖 InnoDB Online DDL](https://mariadb.com/kb/en/innodb-online-ddl-overview/)
- [📖 ALGORITHM and LOCK Options](https://mariadb.com/kb/en/alter-table/#algorithm)
- [🛠️ pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html)
- [📊 Performance Schema Index Statistics](https://mariadb.com/kb/en/performance-schema-table_io_waits_summary_by_index_usage-table/)

---

## ➡️ Fin du chapitre 5

Vous avez maintenant une compréhension complète des **index et de l'optimisation des performances** dans MariaDB, depuis les fondamentaux du B-Tree jusqu'aux techniques avancées de gestion en production.

**Prochaine étape** : Chapitre 6 - Transactions et Concurrence, où vous apprendrez à gérer l'intégrité des données et les accès concurrents avec les niveaux d'isolation, MVCC et la gestion des deadlocks.

⏭️ [Transactions et Concurrence](/06-transactions-et-concurrence/README.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.8 Contrôle espace temporaire (max_tmp_space_usage, max_total_tmp_space_usage) 🆕

> **Niveau** : Avancé  
> **Durée estimée** : 1-1.5 heures  
> **Prérequis** :
> - Section 11.7 (Gestion de l'espace disque)
> - Section 11.2 (Variables système)
> - Compréhension des tables temporaires
> - Connaissance de l'optimiseur de requêtes

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le problème des tables temporaires incontrôlées
- **Configurer** les nouvelles limites d'espace temporaire de MariaDB 11.8
- **Distinguer** max_tmp_space_usage et max_total_tmp_space_usage
- **Surveiller** l'utilisation de l'espace temporaire
- **Prévenir** les saturations disque dues aux tables temporaires
- **Diagnostiquer** les requêtes consommant trop d'espace temporaire
- **Optimiser** les requêtes pour réduire l'usage temporaire

---

## Introduction

### 🆕 Nouveauté MariaDB 11.8

MariaDB 11.8 LTS introduit deux **variables critiques** pour contrôler l'utilisation de l'espace temporaire :

- **`max_tmp_space_usage`** : Limite par **session** (connexion)
- **`max_total_tmp_space_usage`** : Limite **globale** (tout le serveur)

Ces variables résolvent un problème majeur rencontré en production : les **tables temporaires incontrôlées** qui saturent le disque.

### Le problème avant MariaDB 11.8

```
Scénario typique:
1. Requête complexe avec GROUP BY / ORDER BY
2. Création d'une table temporaire de 50 GB
3. Saturation de /tmp (ou tmpdir)
4. MariaDB s'arrête brutalement
5. Corruption potentielle de données
6. Indisponibilité complète du service

Résultat: Incident majeur, RTO élevé
```

**Conséquences réelles** :
- 💥 **Crash serveur** : Disque /tmp plein = arrêt MariaDB
- 🔒 **Blocage général** : Toutes les sessions en attente
- ❌ **Corruption** : Écritures partielles
- 📉 **Performance** : Swap intense si RAM saturée
- 🚨 **Incident production** : Service down

💡 **Solution MariaDB 11.8** : Limiter **proactivement** l'espace temporaire utilisable.

---

## Architecture de l'espace temporaire

### Tables temporaires MariaDB

MariaDB crée des **tables temporaires** (invisibles pour l'utilisateur) dans plusieurs cas :

#### 1. Tables temporaires implicites

Créées automatiquement par l'**optimiseur** :

```sql
-- GROUP BY sur colonnes non-indexées
SELECT category, COUNT(*), AVG(price)
FROM products
GROUP BY category;
-- → Table temporaire pour regroupement

-- ORDER BY différent de l'index
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY total DESC;
-- → Table temporaire pour tri

-- DISTINCT sur large dataset
SELECT DISTINCT customer_id
FROM orders;
-- → Table temporaire pour élimination doublons

-- UNION
SELECT name FROM customers
UNION
SELECT name FROM suppliers;
-- → Table temporaire pour fusion

-- Sous-requêtes complexes
SELECT o.* FROM orders o
WHERE o.customer_id IN (
    SELECT id FROM customers WHERE country = 'FR'
);
-- → Table temporaire possible
```

#### 2. Tables temporaires explicites

Créées par l'utilisateur :

```sql
-- CREATE TEMPORARY TABLE
CREATE TEMPORARY TABLE temp_results (
    id INT,
    value DECIMAL(10,2)
);

INSERT INTO temp_results
SELECT product_id, SUM(quantity * price)
FROM order_items
GROUP BY product_id;
```

### Emplacements de stockage

```
┌─────────────────────────────────────────┐
│   Tables temporaires PETITES            │
│   (< tmp_table_size)                    │
│   ↓                                     │
│   Stockage: MEMORY (RAM)                │
└─────────────────────────────────────────┘
            ↓ (si dépassement)
┌─────────────────────────────────────────┐
│   Tables temporaires MOYENNES/GRANDES   │
│   (> tmp_table_size)                    │
│   ↓                                     │
│   Stockage: Disque                      │
│   - /tmp (tmpdir)                       │
│   - #innodb_temp (InnoDB)               │
└─────────────────────────────────────────┘
```

**Variables associées** :

```sql
-- Taille max en RAM
SHOW VARIABLES LIKE 'tmp_table_size';        -- 16 MB (défaut)
SHOW VARIABLES LIKE 'max_heap_table_size';   -- 16 MB (défaut)

-- Répertoire temporaire
SHOW VARIABLES LIKE 'tmpdir';                -- /tmp
SHOW VARIABLES LIKE 'innodb_temp_data_file_path';
```

---

## Les nouvelles variables de contrôle

### max_tmp_space_usage (limite par session)

**Définition** : Espace temporaire maximal qu'une **seule connexion** peut utiliser.

```sql
-- Vérifier la valeur actuelle
SHOW VARIABLES LIKE 'max_tmp_space_usage';

-- Valeur par défaut MariaDB 11.8
-- max_tmp_space_usage = 10737418240 (10 GB)
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Limite par session : 5 GB
max_tmp_space_usage = 5368709120

# Ou en notation simplifiée
max_tmp_space_usage = 5G
```

```sql
-- Modification dynamique (session)
SET SESSION max_tmp_space_usage = 2147483648;  -- 2 GB pour cette session

-- Modification globale (nouvelles sessions)
SET GLOBAL max_tmp_space_usage = 5368709120;  -- 5 GB
```

**Comportement** :

```sql
-- Requête consommant 6 GB d'espace temporaire
-- Avec max_tmp_space_usage = 5G
SELECT category, subcategory, COUNT(*), AVG(price)
FROM products
GROUP BY category, subcategory
ORDER BY COUNT(*) DESC;

-- ❌ ERREUR après 5 GB :
-- ERROR 1114 (HY000): Table is full
-- Ou
-- ERROR 3 (HY000): Temporary space limit exceeded
```

### max_total_tmp_space_usage (limite globale)

**Définition** : Espace temporaire maximal pour **TOUTES** les connexions combinées.

```sql
-- Vérifier la valeur actuelle
SHOW VARIABLES LIKE 'max_total_tmp_space_usage';

-- Valeur par défaut MariaDB 11.8
-- max_total_tmp_space_usage = 107374182400 (100 GB)
```

**Configuration** :

```ini
# my.cnf
[mysqld]
# Limite globale : 50 GB
max_total_tmp_space_usage = 53687091200

# Ou
max_total_tmp_space_usage = 50G
```

```sql
-- Modification dynamique
SET GLOBAL max_total_tmp_space_usage = 53687091200;  -- 50 GB
```

**Comportement** :

```
Serveur avec max_total_tmp_space_usage = 50G

Session A : Utilise 20 GB
Session B : Utilise 25 GB
Session C : Tente d'utiliser 10 GB

Total = 20 + 25 + 10 = 55 GB > 50 GB
→ Session C reçoit une erreur
```

### Interaction entre les deux limites

```
┌──────────────────────────────────────────┐
│  max_total_tmp_space_usage (global)      │
│  100 GB                                  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Session 1                          │  │
│  │ max_tmp_space_usage: 10 GB         │  │
│  │ Utilisé: 8 GB                      │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Session 2                          │  │
│  │ max_tmp_space_usage: 10 GB         │  │
│  │ Utilisé: 5 GB                      │  │
│  └────────────────────────────────────┘  │
│                                          │
│  Total utilisé: 13 GB / 100 GB           │
└──────────────────────────────────────────┘
```

**Règles** :
1. Une session ne peut **jamais** dépasser `max_tmp_space_usage`
2. Le total de toutes les sessions ne peut **jamais** dépasser `max_total_tmp_space_usage`
3. La limite **la plus restrictive** s'applique

---

## Configuration recommandée par environnement

### Production (serveur dédié, 128 GB RAM)

```ini
# my.cnf - Production
[mysqld]
# Tables temporaires en RAM jusqu'à 256 MB
tmp_table_size = 268435456          # 256 MB
max_heap_table_size = 268435456     # 256 MB

# Limite par session : 10 GB
# Raisonnement : Requêtes normales < 1 GB, max toléré = 10 GB
max_tmp_space_usage = 10737418240   # 10 GB

# Limite globale : 80 GB
# Raisonnement :
#   - Serveur 128 GB RAM
#   - Max 100 connexions simultanées
#   - Moyenne 800 MB/connexion = 80 GB
#   - Marge de sécurité : 20 GB restants pour OS/buffers
max_total_tmp_space_usage = 85899345920  # 80 GB
```

### Développement / Test

```ini
# my.cnf - Développement
[mysqld]
tmp_table_size = 67108864           # 64 MB
max_heap_table_size = 67108864      # 64 MB

# Limites plus strictes pour détecter problèmes
max_tmp_space_usage = 2147483648    # 2 GB
max_total_tmp_space_usage = 10737418240  # 10 GB
```

### Application analytique (OLAP)

```ini
# my.cnf - Analytique
[mysqld]
# Tables temporaires larges autorisées
tmp_table_size = 1073741824         # 1 GB
max_heap_table_size = 1073741824    # 1 GB

# Limites élevées pour agrégations massives
max_tmp_space_usage = 53687091200   # 50 GB
max_total_tmp_space_usage = 214748364800  # 200 GB
```

### Multi-tenant (SaaS)

```ini
# my.cnf - Multi-tenant
[mysqld]
tmp_table_size = 134217728          # 128 MB
max_heap_table_size = 134217728     # 128 MB

# Limites strictes par tenant
max_tmp_space_usage = 5368709120    # 5 GB
max_total_tmp_space_usage = 53687091200  # 50 GB
```

---

## Surveillance de l'espace temporaire

### Variables de statut

```sql
-- Nombre de tables temporaires créées
SHOW STATUS LIKE 'Created_tmp%';
```

**Sortie** :

```
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 1250  |  -- Sur disque (attention!)
| Created_tmp_tables      | 12500 |  -- Total (RAM + disque)
+-------------------------+-------+
```

**Interprétation** :

```sql
-- Ratio tables temporaires sur disque
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_tables') * 100
    AS pct_disk;

-- Si > 25% : Augmenter tmp_table_size ou optimiser requêtes
```

### Performance Schema (MariaDB 10.5+)

```sql
-- Activer instrumentation
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'stage/sql/creating%tmp%';

-- Surveiller utilisation espace temporaire par thread
SELECT
    t.THREAD_ID,
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_DB,
    SUM(ewt.COUNT_STAR) AS tmp_tables_created
FROM performance_schema.threads t
JOIN performance_schema.events_waits_summary_by_thread_by_event_name ewt
    ON t.THREAD_ID = ewt.THREAD_ID
WHERE ewt.EVENT_NAME LIKE '%tmp%'
GROUP BY t.THREAD_ID
ORDER BY tmp_tables_created DESC
LIMIT 10;
```

### Script de monitoring

```bash
#!/bin/bash
# /usr/local/bin/monitor-tmp-space.sh

LOG_FILE="/var/log/mysql/tmp-space-monitor.log"

# Fonction de log
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# 1. Vérifier les limites configurées
MAX_SESSION=$(mariadb -sN -e "SELECT @@max_tmp_space_usage")
MAX_TOTAL=$(mariadb -sN -e "SELECT @@max_total_tmp_space_usage")

log "Limites configurées:"
log "  Par session: $(numfmt --to=iec $MAX_SESSION)"
log "  Globale: $(numfmt --to=iec $MAX_TOTAL)"

# 2. Ratio tables temporaires sur disque
RATIO=$(mariadb -sN -e "
    SELECT ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Created_tmp_tables') * 100,
        2
    )
")

log "Ratio tables temporaires sur disque: ${RATIO}%"

if [ $(echo "$RATIO > 25" | bc) -eq 1 ]; then
    log "ALERTE: Ratio élevé de tables sur disque (> 25%)"
    echo "Ratio tables tmp sur disque: ${RATIO}%" | \
        mail -s "MariaDB Temporary Tables Alert" dba@example.com
fi

# 3. Taille du répertoire temporaire
TMP_SIZE=$(du -sb /tmp | awk '{print $1}')
log "Taille /tmp: $(numfmt --to=iec $TMP_SIZE)"

# 4. Taille tablespace temporaire InnoDB
INNODB_TEMP=$(du -sb /var/lib/mysql/#innodb_temp 2>/dev/null | awk '{print $1}')
if [ -n "$INNODB_TEMP" ]; then
    log "Taille tablespace temporaire InnoDB: $(numfmt --to=iec $INNODB_TEMP)"
fi
```

---

## Diagnostic des requêtes problématiques

### Identifier les requêtes créant des tables temporaires

```sql
-- Activer le slow query log avec détails
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0;  -- Toutes les requêtes
SET GLOBAL log_slow_verbosity = 'query_plan,explain';

-- Les requêtes loggées contiendront :
-- Tmp_tables: 1
-- Tmp_disk_tables: 1
```

### Analyse avec EXPLAIN

```sql
-- Requête problématique
EXPLAIN
SELECT category, subcategory, brand, COUNT(*), AVG(price)
FROM products
GROUP BY category, subcategory, brand;
```

**Sortie** :

```
+----+-------------+----------+------+---------------+------+---------+------+--------+----------+----------------------------------------------+
| id | select_type | table    | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                        |
+----+-------------+----------+------+---------------+------+---------+------+--------+----------+----------------------------------------------+
|  1 | SIMPLE      | products | ALL  | NULL          | NULL | NULL    | NULL | 500000 |   100.00 | Using temporary; Using filesort              |
+----+-------------+----------+------+---------------+------+---------+------+--------+----------+----------------------------------------------+
```

**Indicateurs** :
- ⚠️ **Using temporary** : Table temporaire créée
- ⚠️ **Using filesort** : Tri sur disque (souvent combiné avec temporary)

### Performance Schema : Requêtes actives

```sql
-- Requêtes en cours utilisant des tables temporaires
SELECT
    ess.SQL_TEXT,
    ess.CREATED_TMP_DISK_TABLES,
    ess.CREATED_TMP_TABLES,
    ess.ROWS_EXAMINED,
    ess.ROWS_SENT,
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER
FROM performance_schema.events_statements_current ess
JOIN performance_schema.threads t ON ess.THREAD_ID = t.THREAD_ID
WHERE ess.CREATED_TMP_TABLES > 0
ORDER BY ess.CREATED_TMP_DISK_TABLES DESC;
```

---

## Optimisation pour réduire l'usage temporaire

### 1. Ajouter des index appropriés

```sql
-- Problème : GROUP BY sans index
SELECT category, COUNT(*)
FROM products
GROUP BY category;
-- → Table temporaire

-- Solution : Index sur category
CREATE INDEX idx_category ON products(category);
-- → Plus de table temporaire !
```

### 2. Limiter les résultats

```sql
-- Problème : Tri de millions de lignes
SELECT * FROM orders
ORDER BY created_at DESC;
-- → Table temporaire énorme

-- Solution : LIMIT
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 100;
-- → Table temporaire réduite à 100 lignes
```

### 3. Diviser les requêtes complexes

```sql
-- Problème : Requête complexe avec UNION et sous-requêtes
SELECT ...
FROM (
    SELECT ... FROM t1
    UNION
    SELECT ... FROM t2
)
WHERE ...
GROUP BY ...
ORDER BY ...;
-- → Multiples tables temporaires

-- Solution : Décomposer
CREATE TEMPORARY TABLE step1 AS
SELECT ... FROM t1;

CREATE TEMPORARY TABLE step2 AS
SELECT ... FROM t2;

SELECT ... FROM step1
UNION ALL
SELECT ... FROM step2;
```

### 4. Utiliser des vues matérialisées (workaround)

```sql
-- Créer une table pré-agrégée
CREATE TABLE products_summary (
    category VARCHAR(50),
    subcategory VARCHAR(50),
    count INT,
    avg_price DECIMAL(10,2),
    PRIMARY KEY (category, subcategory)
) ENGINE=InnoDB;

-- Peupler périodiquement (cron)
TRUNCATE products_summary;
INSERT INTO products_summary
SELECT category, subcategory, COUNT(*), AVG(price)
FROM products
GROUP BY category, subcategory;

-- Requêtes utilisent la summary (pas de table temporaire)
SELECT * FROM products_summary
WHERE category = 'Electronics';
```

### 5. Augmenter tmp_table_size (si RAM disponible)

```sql
-- Si beaucoup de tables temporaires passent sur disque
-- Et si RAM disponible

-- Avant (défaut)
SET GLOBAL tmp_table_size = 16777216;        -- 16 MB
SET GLOBAL max_heap_table_size = 16777216;   -- 16 MB

-- Après (si 64 GB RAM disponibles)
SET GLOBAL tmp_table_size = 268435456;       -- 256 MB
SET GLOBAL max_heap_table_size = 268435456;  -- 256 MB
```

---

## Gestion des erreurs

### Erreur : Limite de session dépassée

```sql
-- Requête dépassant max_tmp_space_usage
SELECT ...;

-- Erreur
ERROR 3 (HY000): Temporary space limit exceeded
```

**Solutions** :

```sql
-- 1. Augmenter temporairement pour cette session
SET SESSION max_tmp_space_usage = 21474836480;  -- 20 GB

-- 2. Optimiser la requête (préféré)
-- Ajouter index, LIMIT, diviser en sous-requêtes

-- 3. Utiliser table permanente
CREATE TABLE temp_results AS
SELECT ... WITH ROLLUP;
```

### Erreur : Limite globale dépassée

```sql
-- Toutes les sessions combinées dépassent max_total_tmp_space_usage
ERROR 3 (HY000): Server temporary space limit exceeded
```

**Solutions** :

```sql
-- 1. Augmenter la limite globale (temporaire)
SET GLOBAL max_total_tmp_space_usage = 214748364800;  -- 200 GB

-- 2. Identifier et tuer les sessions consommatrices
SELECT
    ID, USER, DB, TIME, STATE, INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC;

KILL <ID>;

-- 3. Optimiser les requêtes problématiques
```

---

## Cas d'usage et exemples

### Cas 1 : Protection contre requête analytique runaway

**Scénario** : Analyste exécute une requête mal optimisée.

```sql
-- Configuration
SET GLOBAL max_tmp_space_usage = 5368709120;  -- 5 GB

-- Requête problématique
SELECT
    customer_id,
    product_id,
    COUNT(*) AS purchases,
    AVG(amount) AS avg_amount
FROM orders
GROUP BY customer_id, product_id
ORDER BY purchases DESC;
-- Tente de créer une table temporaire de 20 GB

-- Résultat : ERREUR après 5 GB
-- → Serveur protégé
-- → Analyste informé du problème
-- → Optimisation nécessaire
```

### Cas 2 : Multi-tenant avec isolation

**Scénario** : SaaS avec plusieurs clients, isolation nécessaire.

```sql
-- Configuration globale stricte
SET GLOBAL max_tmp_space_usage = 2147483648;  -- 2 GB par tenant

-- Tenant A : Requête normale (500 MB temp)
-- → OK

-- Tenant B : Requête abusive (10 GB temp)
-- → ERREUR
-- → Tenant A non impacté !
```

### Cas 3 : Import massif avec tables temporaires

**Scénario** : Import ETL avec transformations complexes.

```sql
-- Désactiver temporairement les limites pour cette session
SET SESSION max_tmp_space_usage = 0;  -- Illimité
-- Ou valeur très élevée
SET SESSION max_tmp_space_usage = 107374182400;  -- 100 GB

-- Import avec transformations
CREATE TEMPORARY TABLE staging AS
SELECT
    customer_id,
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(total) AS revenue
FROM imported_orders
GROUP BY customer_id, YEAR(order_date), MONTH(order_date);

-- Insérer dans table finale
INSERT INTO customer_revenue
SELECT * FROM staging;

-- Réactiver les limites
SET SESSION max_tmp_space_usage = 5368709120;  -- 5 GB
```

---

## Bonnes pratiques

### ✅ À FAIRE

1. **Définir des limites** appropriées dès l'installation
2. **Monitorer** régulièrement le ratio tmp disk tables
3. **Augmenter tmp_table_size** si RAM disponible (256 MB - 1 GB)
4. **Optimiser les requêtes** créant des tables temporaires
5. **Alerter** si ratio > 25% de tables sur disque
6. **Ajuster par environnement** : prod strict, dev plus souple
7. **Documenter** les exceptions (sessions nécessitant limites élevées)
8. **Tester** les limites sur staging avant production

### ❌ À ÉVITER

1. **Désactiver complètement** (max_tmp_space_usage = 0) en production
2. **Limites trop élevées** = protection inefficace
3. **Limites trop basses** = blocage opérationnel
4. **Ignorer** les erreurs "Temporary space limit exceeded"
5. **Pas de monitoring** de l'usage temporaire
6. **Résoudre par augmentation** systématique au lieu d'optimiser

---

## Comparaison avec d'autres SGBD

| SGBD | Mécanisme équivalent |
|------|----------------------|
| **MariaDB 11.8** | max_tmp_space_usage, max_total_tmp_space_usage |
| **MySQL** | Pas d'équivalent natif (limité par tmpdir) |
| **PostgreSQL** | temp_file_limit (par session) |
| **Oracle** | TEMP tablespace quota par utilisateur |
| **SQL Server** | tempdb autogrow avec limite max size |

MariaDB 11.8 est **pionnier** avec un double contrôle (session + global).

---

## ✅ Points clés à retenir

- 🆕 **Nouveauté 11.8** : max_tmp_space_usage et max_total_tmp_space_usage
- **Double protection** : Par session (10 GB défaut) + Globale (100 GB défaut)
- **Prévention** : Saturation disque /tmp impossible avec limites appropriées
- **Configuration** : Adapter selon RAM et charge de travail
- **Monitoring** : Surveiller Created_tmp_disk_tables / Created_tmp_tables
- **Optimisation** : Index, LIMIT, décomposition requêtes
- **Ratio acceptable** : < 25% de tables temporaires sur disque
- **tmp_table_size** : 256 MB - 1 GB si RAM disponible
- **Production** : Limites strictes (5-10 GB session, 50-80 GB global)
- **Analytique** : Limites élevées (50 GB session, 200 GB global)
- **Erreurs** : "Temporary space limit exceeded" = optimiser requête
- **Multi-tenant** : Isolation garantie par limite session

---

## 🔗 Ressources et références

- [📖 Documentation officielle - System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
- [📖 Documentation - Internal Temporary Tables](https://mariadb.com/kb/en/internal-temporary-tables/)
- [📖 Documentation - EXPLAIN](https://mariadb.com/kb/en/explain/)
- [📖 Performance Schema](https://mariadb.com/kb/en/performance-schema/)

---

## ➡️ Section suivante

**[11.9 Monitoring et métriques importantes](./09-monitoring-metriques.md)** : Surveillance proactive de MariaDB, métriques critiques, outils de monitoring (Prometheus, Grafana, PMM).

---

**💡 Conseil final** : Les limites d'espace temporaire sont comme des airbags : vous espérez ne jamais en avoir besoin, mais quand un incident arrive, vous êtes **très heureux** de les avoir activées ! Configurez-les dès l'installation. 🛡️💾

⏭️ [Monitoring et métriques importantes](/11-administration-configuration/09-monitoring-metriques.md)

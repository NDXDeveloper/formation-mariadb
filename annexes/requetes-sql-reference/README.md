🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe C - Requêtes SQL de Référence

> **Niveau** : Intermédiaire à Expert  
> **Durée estimée** : Consultation à la demande  
> **Type** : Bibliothèque de requêtes prêtes à l'emploi

---

## 📖 Introduction

Cette annexe constitue une **bibliothèque de requêtes SQL** essentielles pour l'administration, le monitoring et l'analyse de MariaDB. Chaque requête est documentée, commentée et prête à être utilisée en production.

### 🎯 Objectifs de cette annexe

- **Accélérer** les tâches d'administration quotidiennes
- **Standardiser** les requêtes de monitoring
- **Faciliter** le diagnostic et le troubleshooting
- **Documenter** les bonnes pratiques SQL
- **Fournir** des templates réutilisables

---

## 🎯 Public Cible

Cette annexe s'adresse à :

| Profil | Usage Principal |
|--------|-----------------|
| **DBA** | Administration, monitoring, maintenance |
| **DevOps/SRE** | Monitoring automatisé, alerting, métriques |
| **Développeurs** | Analyse de performance, debugging |
| **Architectes** | Audit de schéma, analyse capacité |
| **Data Engineers** | Analyse volumétries, optimisation ETL |

---

## 📋 Organisation de l'Annexe

Cette annexe est organisée en **trois sections thématiques** :

### C.1 - Requêtes d'Administration
Requêtes pour la gestion quotidienne :
- **Locks et verrous** : Identifier blocages et deadlocks
- **Processus actifs** : Surveiller connexions et requêtes
- **Tailles tables** : Analyser volumétries et croissance
- **Gestion utilisateurs** : Auditer privilèges et connexions
- **Configuration** : Inspecter variables et statuts

**Cas d'usage** : Opérations quotidiennes, troubleshooting

### C.2 - Requêtes de Monitoring
Requêtes pour surveillance et métriques :
- **Buffer Pool** : Analyser cache et hit rate
- **Slow Queries** : Identifier requêtes lentes
- **Réplication** : Surveiller lag et statut
- **Connexions** : Monitorer charge et concurrence
- **I/O et performance** : Métriques disque et mémoire

**Cas d'usage** : Dashboards, alerting, SLA monitoring

### C.3 - Requêtes d'Analyse
Requêtes pour optimisation et planification :
- **Statistiques index** : Analyser utilisation et efficacité
- **Fragmentation** : Détecter tables à optimiser
- **Distribution données** : Analyser cardinalité
- **Performance Schema** : Exploiter métriques avancées
- **Audit schéma** : Analyser structure et design

**Cas d'usage** : Optimisation, capacity planning, refactoring

---

## 🔍 Types de Requêtes

### Par Niveau de Complexité

| Niveau | Description | Exemple |
|--------|-------------|---------|
| 🟢 **Basique** | Requêtes simples, résultats directs | Taille d'une table |
| 🟡 **Intermédiaire** | Jointures, agrégations | Top 10 tables volumineuses |
| 🔴 **Avancé** | Requêtes complexes, window functions | Analyse tendances croissance |
| 🟣 **Expert** | Performance Schema, requêtes optimisées | Profiling complet requêtes |

### Par Source de Données

Les requêtes exploitent principalement :

1. **INFORMATION_SCHEMA**
   - Tables métadonnées
   - Colonnes, index, contraintes
   - Privilèges utilisateurs
   - Vue d'ensemble du schéma

2. **PERFORMANCE_SCHEMA**
   - Métriques de performance détaillées
   - Statistiques requêtes
   - Événements I/O
   - Locks et attentes

3. **mysql (système)**
   - Tables grant (user, db, tables_priv)
   - Configuration serveur
   - Logs et historique

4. **SHOW Statements**
   - SHOW STATUS
   - SHOW VARIABLES
   - SHOW PROCESSLIST
   - SHOW ENGINE INNODB STATUS

---

## 💡 Comment Utiliser Cette Annexe

### Consultation Rapide

1. **Identifier votre besoin** : Administration, monitoring ou analyse ?
2. **Naviguer vers la section** appropriée (C.1, C.2 ou C.3)
3. **Copier la requête** adaptée à votre situation
4. **Personnaliser** les paramètres si nécessaire
5. **Exécuter** dans votre environnement

### Adaptation des Requêtes

Toutes les requêtes sont des **templates** :

```sql
-- Exemple de template
SELECT 
  TABLE_NAME,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'YOUR_DATABASE'  -- ← Personnaliser ici
ORDER BY size_mb DESC
LIMIT 10;  -- ← Ajuster la limite
```

### Intégration dans Scripts

Les requêtes peuvent être intégrées dans :

```bash
# Script bash
RESULT=$(mariadb -u user -p -N -B -e "
  SELECT COUNT(*) 
  FROM information_schema.PROCESSLIST 
  WHERE TIME > 60
")

if [ $RESULT -gt 10 ]; then
  echo "ALERTE: $RESULT requêtes longues détectées"
fi
```

```python
# Script Python
import mysql.connector

query = """
SELECT TABLE_NAME, TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = %s
ORDER BY TABLE_ROWS DESC
"""

cursor.execute(query, ('production_db',))
```

---

## 📊 Conventions de Présentation

### Format des Requêtes

Chaque requête est présentée avec :

1. **Titre descriptif**
2. **Niveau de complexité** (🟢🟡🔴🟣)
3. **Description** du besoin
4. **Requête SQL** commentée
5. **Exemple de résultat**
6. **Notes d'utilisation** et contexte

### Structure Standard

```sql
-- ============================================================
-- [Titre de la requête]
-- Niveau : [🟢/🟡/🔴/🟣]
-- ============================================================
-- Description :
-- [Explication du besoin et de l'usage]
--
-- Notes :
-- - [Point important 1]
-- - [Point important 2]
-- ============================================================

SELECT 
  column1,              -- Description colonne 1
  column2,              -- Description colonne 2
  ROUND(calculation, 2) AS metric  -- Métrique calculée
FROM information_schema.TABLE_NAME
WHERE condition = 'value'
ORDER BY metric DESC
LIMIT 10;

-- Résultat exemple :
-- +----------+----------+--------+
-- | column1  | column2  | metric |
-- +----------+----------+--------+
-- | value1   | value2   | 123.45 |
-- +----------+----------+--------+
```

### Commentaires dans les Requêtes

```sql
-- Types de commentaires utilisés :

SELECT 
  column1,          -- Commentaire inline (description colonne)
  /*
   * Commentaire bloc
   * Pour explications longues
   */
  column2
FROM table
WHERE 1=1           -- Condition toujours vraie pour faciliter ajouts
  AND active = 1;   -- Condition réelle

-- Section séparateur
-- ============================================================
```

---

## 🎨 Catégories de Requêtes

### Administration Quotidienne

```sql
-- Exemples de catégories :
✓ Gestion des utilisateurs et privilèges
✓ Surveillance de l'espace disque
✓ Analyse des connexions actives
✓ Vérification de l'intégrité des données
✓ Gestion des verrous et deadlocks
```

### Monitoring et Alerting

```sql
-- Métriques surveillées :
✓ Buffer pool hit rate
✓ Slow query ratio
✓ Réplication lag
✓ Connexions actives vs max
✓ Taux d'erreurs
✓ Utilisation CPU/RAM/Disk I/O
```

### Optimisation et Performance

```sql
-- Analyses de performance :
✓ Index non utilisés
✓ Tables fragmentées
✓ Requêtes lentes récurrentes
✓ Cardinalité des index
✓ Distribution des données
```

---

## 🛠️ Outils et Contextes d'Utilisation

### En CLI

```bash
# Exécution directe
mariadb -u admin -p -e "SELECT COUNT(*) FROM information_schema.PROCESSLIST"

# Avec formatage
mariadb -u admin -p -t -e "SHOW FULL PROCESSLIST"

# Export vers fichier
mariadb -u admin -p < monitoring_queries.sql > report.txt
```

### Dans des Scripts

```bash
#!/bin/bash
# check_slow_queries.sh

SLOW_COUNT=$(mariadb -u monitor -p -N -B -e "
  SELECT VARIABLE_VALUE 
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Slow_queries'
")

echo "Slow queries: $SLOW_COUNT"
```

### Avec Monitoring Tools

```yaml
# Prometheus mysqld_exporter custom queries
# /etc/mysqld_exporter/queries.yml
custom_queries:
  - name: table_sizes
    query: |
      SELECT 
        TABLE_SCHEMA,
        TABLE_NAME,
        ROUND(DATA_LENGTH/1024/1024,2) AS data_mb
      FROM information_schema.TABLES
      WHERE TABLE_SCHEMA NOT IN ('mysql','information_schema')
    labels:
      - table_schema
      - table_name
    values:
      - data_mb
```

### Dans Applications

```python
# Monitoring dashboard
def get_slow_queries_ratio():
    query = """
    SELECT 
      CAST(slow.variable_value AS DECIMAL) / 
      CAST(total.variable_value AS DECIMAL) * 100 AS slow_ratio
    FROM 
      (SELECT variable_value FROM information_schema.global_status 
       WHERE variable_name = 'Slow_queries') slow,
      (SELECT variable_value FROM information_schema.global_status 
       WHERE variable_name = 'Questions') total
    """
    return execute_query(query)
```

---

## 📈 Métriques et KPI Importants

### Métriques de Performance

| Métrique | Requête Source | Seuil Recommandé |
|----------|----------------|------------------|
| **Buffer Pool Hit Rate** | INFORMATION_SCHEMA.GLOBAL_STATUS | > 99% |
| **Slow Query Ratio** | Slow_queries / Questions | < 1% |
| **Threads Connected** | SHOW STATUS | < 80% max_connections |
| **Table Cache Hit** | Open_tables / Opened_tables | > 95% |
| **InnoDB Row Lock Waits** | Innodb_row_lock_waits | Tendance stable |

### Métriques de Capacité

| Métrique | Description | Action si Dépassé |
|----------|-------------|-------------------|
| **Disk Space Used** | Taille totale bases | Planifier extension |
| **Largest Tables** | Top 10 volumineuses | Archivage/partitioning |
| **Growth Rate** | Croissance mensuelle | Capacity planning |
| **Index Size Ratio** | Index / Data | Optimiser si > 50% |

### Métriques de Santé

| Métrique | Description | Criticité |
|----------|-------------|-----------|
| **Replication Lag** | Seconds_Behind_Master | 🔴 Critique si > 60s |
| **Deadlocks** | Innodb_deadlocks | 🟡 Surveiller tendance |
| **Connection Errors** | Aborted_connects | 🟡 Investiguer si élevé |
| **Table Locks Waited** | Table_locks_waited | 🟡 Optimiser requêtes |

---

## 🔐 Sécurité et Privilèges

### Privilèges Requis par Type de Requête

| Catégorie | Privilèges Minimaux | Justification |
|-----------|---------------------|---------------|
| **INFORMATION_SCHEMA** | SELECT sur system schemas | Métadonnées publiques |
| **PERFORMANCE_SCHEMA** | SELECT + PERFORMANCE_SCHEMA | Métriques détaillées |
| **SHOW STATUS** | PROCESS ou SUPER | Statuts serveur |
| **SHOW PROCESSLIST** | PROCESS | Voir autres connexions |
| **KILL** | SUPER ou CONNECTION ADMIN | Terminer processus |

### Utilisateur de Monitoring Recommandé

```sql
-- Créer utilisateur monitoring dédié
CREATE USER 'monitor'@'localhost' 
  IDENTIFIED BY 'secure_password';

-- Privilèges minimaux
GRANT SELECT ON *.* TO 'monitor'@'localhost';
GRANT PROCESS ON *.* TO 'monitor'@'localhost';
GRANT REPLICATION CLIENT ON *.* TO 'monitor'@'localhost';

-- Pour Performance Schema
GRANT SELECT ON performance_schema.* TO 'monitor'@'localhost';

FLUSH PRIVILEGES;
```

---

## ⚡ Optimisation des Requêtes de Monitoring

### Bonnes Pratiques

✅ **À FAIRE**
- Utiliser `LIMIT` pour requêtes volumineuses
- Indexer tables custom de monitoring si créées
- Exécuter requêtes lourdes hors heures de pointe
- Cacher résultats côté application si fréquence élevée
- Utiliser vues pour requêtes complexes répétitives

❌ **À ÉVITER**
- Requêtes INFORMATION_SCHEMA trop fréquentes (< 1 min)
- SELECT * sur INFORMATION_SCHEMA sans WHERE
- Jointures complexes sur PERFORMANCE_SCHEMA en production
- Requêtes bloquantes sur tables applicatives pour monitoring

### Exemple d'Optimisation

```sql
-- ❌ LENT : Scanne toutes les tables
SELECT TABLE_NAME, TABLE_ROWS
FROM information_schema.TABLES
ORDER BY TABLE_ROWS DESC;

-- ✅ RAPIDE : Filtre sur schema spécifique
SELECT TABLE_NAME, TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY TABLE_ROWS DESC
LIMIT 10;
```

---

## 📚 Sources de Données Détaillées

### INFORMATION_SCHEMA

Tables les plus utilisées :

```sql
-- Métadonnées tables
INFORMATION_SCHEMA.TABLES
INFORMATION_SCHEMA.COLUMNS
INFORMATION_SCHEMA.STATISTICS (index)

-- Utilisateurs et privilèges
INFORMATION_SCHEMA.USER_PRIVILEGES
INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
INFORMATION_SCHEMA.TABLE_PRIVILEGES

-- Activité
INFORMATION_SCHEMA.PROCESSLIST
INFORMATION_SCHEMA.INNODB_TRX (transactions)
INFORMATION_SCHEMA.INNODB_LOCKS

-- Statuts
INFORMATION_SCHEMA.GLOBAL_STATUS
INFORMATION_SCHEMA.GLOBAL_VARIABLES
INFORMATION_SCHEMA.SESSION_STATUS
```

### PERFORMANCE_SCHEMA

Tables clés (MariaDB 10.5+) :

```sql
-- Digest de requêtes
performance_schema.events_statements_summary_by_digest

-- I/O
performance_schema.file_summary_by_instance
performance_schema.table_io_waits_summary_by_table

-- Locks
performance_schema.metadata_locks
performance_schema.table_lock_waits_summary_by_table

-- Connexions
performance_schema.accounts
performance_schema.users
```

---

## 🎯 Cas d'Usage par Profil

### Pour DBA

```sql
-- Tâches quotidiennes :
✓ Vérifier santé serveur
✓ Analyser slow queries
✓ Surveiller réplication
✓ Gérer espace disque
✓ Auditer privilèges utilisateurs
```

### Pour DevOps/SRE

```sql
-- Automatisation et alerting :
✓ Métriques pour Prometheus/Grafana
✓ Health checks automatisés
✓ Seuils d'alerte (lag, connexions, etc.)
✓ Rapports quotidiens/hebdomadaires
✓ Capacity planning
```

### Pour Développeurs

```sql
-- Optimisation applicative :
✓ Identifier requêtes lentes
✓ Analyser utilisation des index
✓ Vérifier distribution des données
✓ Débugger locks et deadlocks
✓ Profiler performance requêtes
```

---

## 📖 Conventions de Nommage

### Variables et Alias

```sql
-- Conventions utilisées dans les requêtes :

-- Alias courts et explicites
SELECT 
  t.TABLE_NAME AS table_name,      -- Lowercase snake_case
  t.TABLE_ROWS AS row_count,        -- Descriptif
  ROUND(DATA_LENGTH/1024/1024, 2) AS data_mb  -- Unité dans nom
FROM information_schema.TABLES t;

-- Préfixes pour clarté
db_name      -- Database name
tbl_name     -- Table name
idx_name     -- Index name
size_mb      -- Taille en MB
size_gb      -- Taille en GB
pct_used     -- Pourcentage
```

### Paramètres à Personnaliser

```sql
-- Marqués clairement dans les requêtes :

WHERE TABLE_SCHEMA = 'YOUR_DATABASE'     -- ← À REMPLACER
  AND TABLE_NAME LIKE 'prefix_%'         -- ← OPTIONNEL
  AND DATA_LENGTH > 1024*1024*100        -- ← 100 MB, ajuster
LIMIT 10;                                -- ← Nombre de résultats
```

---

## 🔗 Ressources Complémentaires

### Documentation MariaDB
- [INFORMATION_SCHEMA](https://mariadb.com/kb/en/information-schema/)
- [PERFORMANCE_SCHEMA](https://mariadb.com/kb/en/performance-schema/)
- [SHOW Statements](https://mariadb.com/kb/en/show/)
- [System Variables](https://mariadb.com/kb/en/server-system-variables/)

### Outils Externes
- **pt-query-digest** : Analyse slow query log
- **Percona Monitoring Plugins** : Templates Nagios/Zabbix
- **mysqld_exporter** : Métriques Prometheus
- **PMM (Percona Monitoring)** : Solution monitoring complète

### Formations Associées
- **Partie 5** : Sécurité et Administration
- **Partie 7** : Performance et Tuning
- **Partie 8** : DevOps et Automatisation
- **Section B.2** : Commandes CLI - Informations Système

---

## ✅ Checklist d'Utilisation

### Avant de Commencer
- [ ] Identifier le besoin (admin, monitoring, analyse)
- [ ] Vérifier les privilèges utilisateur
- [ ] Choisir la section appropriée (C.1, C.2, C.3)
- [ ] Comprendre l'impact de la requête (charge serveur)

### Lors de l'Exécution
- [ ] Personnaliser les paramètres (database, limits)
- [ ] Tester sur environnement non-prod si possible
- [ ] Vérifier le temps d'exécution
- [ ] Interpréter les résultats correctement

### Après Exécution
- [ ] Documenter les résultats si pertinent
- [ ] Automatiser si requête récurrente
- [ ] Adapter les seuils d'alerte si nécessaire
- [ ] Partager avec l'équipe si trouvaille importante

---

## 🎓 Comment Contribuer

Cette annexe est un **document vivant** qui peut être enrichi :

### Suggestions Bienvenues
- 📝 Nouvelles requêtes utiles
- 🔧 Optimisations de requêtes existantes
- 💡 Cas d'usage supplémentaires
- 🐛 Corrections d'erreurs
- 📚 Liens vers ressources pertinentes

### Format de Contribution
```sql
-- ============================================================
-- [Titre de la nouvelle requête]
-- Niveau : [🟢/🟡/🔴/🟣]
-- Auteur : [Nom]
-- Date : [YYYY-MM-DD]
-- ============================================================
-- Description :
-- [Explication détaillée]
--
-- Cas d'usage :
-- [Quand utiliser cette requête]
--
-- Notes :
-- [Points importants]
-- ============================================================

[Requête SQL]
```

---

## 🔗 Sections de l'Annexe

### → [C.1 Requêtes d'Administration](./01-requetes-administration.md)
Requêtes pour gestion quotidienne : locks, processlist, table sizes, utilisateurs

### → [C.2 Requêtes de Monitoring](./02-requetes-monitoring.md)
Requêtes pour surveillance : buffer pool, slow queries, réplication, métriques

### → [C.3 Requêtes d'Analyse](./03-requetes-analyse.md)
Requêtes pour optimisation : statistiques index, fragmentation, performance schema

---

## ⚡ Quick Start

### Top 5 Requêtes les Plus Utilisées

```sql
-- 1️⃣ Tailles des tables
SELECT TABLE_NAME, ROUND((DATA_LENGTH+INDEX_LENGTH)/1024/1024,2) AS size_mb
FROM information_schema.TABLES WHERE TABLE_SCHEMA='YOUR_DB' ORDER BY size_mb DESC LIMIT 10;

-- 2️⃣ Processus actifs
SELECT ID, USER, DB, TIME, STATE, LEFT(INFO,50) FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep' ORDER BY TIME DESC;

-- 3️⃣ Buffer pool hit rate
SELECT ROUND(
  (1 - (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') /
       (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests')
  ) * 100, 2) AS hit_rate_pct;

-- 4️⃣ Slow queries ratio
SELECT ROUND(
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Slow_queries') /
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Questions') * 100, 4
) AS slow_pct;

-- 5️⃣ Réplication lag
SHOW SLAVE STATUS\G
-- Regarder : Seconds_Behind_Master
```

---

**MariaDB** : 11.8 LTS

---

## ➡️ Sections Suivantes

**[C.1 Requêtes d'Administration →](./01-requetes-administration.md)**  
Découvrez les requêtes essentielles pour l'administration quotidienne

**[C.2 Requêtes de Monitoring →](./02-requetes-monitoring.md)**  
Explorez les requêtes de surveillance et métriques

**[C.3 Requêtes d'Analyse →](./03-requetes-analyse.md)**  
Apprenez à analyser et optimiser vos bases de données

---

*💡 Astuce : Créez un fichier `my_queries.sql` avec vos requêtes favorites personnalisées pour votre environnement !*

⏭️ [Requêtes d'administration (locks, processlist, table sizes)](/annexes/requetes-sql-reference/01-requetes-administration.md)

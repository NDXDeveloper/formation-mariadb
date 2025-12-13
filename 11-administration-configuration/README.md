🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11. Administration et Configuration

> **Niveau** : Avancé  
> **Durée estimée** : 8-10 heures  
> **Prérequis** :
> - Chapitres 1-2 (Fondamentaux et SQL)
> - Chapitre 7 (Moteurs de stockage)
> - Connaissances système Linux/Unix
> - Expérience avec la ligne de commande

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Configurer** MariaDB de manière optimale via les fichiers de configuration
- **Gérer** les variables système et de session pour adapter le comportement du serveur
- **Maîtriser** les modes SQL (sql_mode) et leur impact sur la compatibilité
- **Administrer** les différents types de logs (error, slow query, general, binary)
- **Effectuer** la maintenance régulière des tables (OPTIMIZE, ANALYZE, CHECK, REPAIR)
- **Surveiller** les métriques critiques et configurer le Thread Pool
- **Exploiter** les nouveautés MariaDB 11.8 : contrôle de l'espace temporaire, utf8mb4 par défaut, extension TIMESTAMP

---

## Introduction

L'administration et la configuration de MariaDB sont des compétences essentielles pour tout DBA ou administrateur système souhaitant maintenir un serveur de base de données performant, fiable et sécurisé en production.

### Pourquoi ce chapitre est crucial

La différence entre un serveur MariaDB "de base" et un serveur **optimisé pour la production** réside principalement dans :

1. **La configuration** : Les paramètres par défaut sont conçus pour être conservateurs et compatibles, mais rarement optimaux pour votre charge de travail spécifique
2. **Le monitoring** : Savoir quoi surveiller et comment réagir aux anomalies
3. **La maintenance** : Des tables bien entretenues garantissent des performances constantes
4. **Les logs** : Une gestion efficace des logs permet le diagnostic rapide et la traçabilité

### Architecture de configuration MariaDB

MariaDB utilise une architecture de configuration hiérarchique :

```
┌─────────────────────────────────────┐
│   Fichiers de configuration         │
│   (my.cnf / my.ini)                 │
│   ├── /etc/my.cnf                   │
│   ├── /etc/mysql/my.cnf             │
│   └── ~/.my.cnf                     │
└─────────────────────────────────────┘
            ↓
┌──────────────────────────────────────┐
│   Variables système                  │
│   ├── Globales (affectent le serveur)│
│   └── Session (par connexion)        │
└──────────────────────────────────────┘
            ↓
┌──────────────────────────────────────┐
│   Comportement du serveur            │
│   ├── Modes SQL                      │
│   ├── Performance                    │
│   └── Sécurité                       │
└──────────────────────────────────────┘
```

### 🆕 Nouveautés MariaDB 11.8 dans ce chapitre

Ce chapitre couvre plusieurs innovations majeures introduites dans MariaDB 11.8 LTS :

| Nouveauté | Impact | Section |
|-----------|--------|---------|
| **Contrôle espace temporaire** | Prévention saturation disque | 11.8 |
| **utf8mb4 par défaut** | Unicode complet natif | 11.11 |
| **UCA 14.0.0** | Nouvelles collations | 11.11 |
| **Extension TIMESTAMP 2106** | Résolution problème Y2038 | 11.12 |
| **TLS par défaut** | Sécurité renforcée | (Chapitre 10) |

---

## Vue d'ensemble des sections

### 11.1 Fichiers de configuration

La gestion des fichiers `my.cnf` (Linux/Unix) et `my.ini` (Windows) constitue la base de l'administration MariaDB. Vous apprendrez :

- La structure hiérarchique des fichiers de configuration
- L'ordre de lecture et la priorité des fichiers
- L'organisation en sections `[client]`, `[mysqld]`, `[mariadb]`
- Les bonnes pratiques pour une configuration modulaire

```ini
# Exemple de structure my.cnf
[client-server]
socket = /var/run/mysqld/mysqld.sock

[mysqld]
# Configuration serveur
datadir = /var/lib/mysql
port = 3306
innodb_buffer_pool_size = 8G

# Nouveauté 11.8 : utf8mb4 par défaut
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

!includedir /etc/mysql/conf.d/
```

### 11.2 Variables système et de session

MariaDB expose des centaines de variables contrôlant tous les aspects du serveur. Cette section couvre :

- La différence entre variables **globales** et **session**
- `SHOW VARIABLES` : inspection de la configuration
- `SET GLOBAL` vs `SET SESSION` : modifications à chaud
- Variables **dynamiques** (modifiables sans redémarrage) vs **statiques**

```sql
-- Afficher toutes les variables
SHOW VARIABLES;

-- Afficher une variable spécifique
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Modifier une variable globale (à chaud)
SET GLOBAL max_connections = 500;

-- Modifier une variable de session
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';
```

💡 **Conseil** : Toute modification via `SET GLOBAL` est **temporaire** et sera perdue au redémarrage. Pour la rendre permanente, ajoutez-la dans `my.cnf`.

### 11.3 Modes SQL (sql_mode)

Le `sql_mode` définit le niveau de rigueur et de compatibilité SQL. C'est crucial pour :

- La **compatibilité** avec d'autres SGBD (MySQL, Oracle, SQL Server)
- Le comportement en cas d'**erreurs** (strict vs permissif)
- La gestion des **valeurs NULL** et des **divisions par zéro**

Modes importants :
- `STRICT_TRANS_TABLES` : Rejette les données invalides
- `NO_ZERO_DATE` : Interdit les dates '0000-00-00'
- `ANSI` : Compatibilité ANSI SQL
- `ORACLE` : Émulation Oracle
- `TRADITIONAL` : Mode strict (équivalent MySQL)

### 11.4 Gestion des logs

Les logs sont essentiels pour le **diagnostic**, l'**audit** et la **réplication**. Quatre types principaux :

| Type de log | Usage principal | Performance | Taille |
|-------------|-----------------|-------------|--------|
| **Error log** | Erreurs et avertissements | Négligeable | Faible |
| **Slow query log** | Requêtes lentes | Faible impact | Moyen |
| **General log** | Toutes les requêtes | ⚠️ Impact élevé | Élevé |
| **Binary log** | Réplication, PITR | Impact modéré | Très élevé |

```sql
-- Activer le slow query log
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 2; -- Requêtes > 2 secondes

-- Vérifier la configuration binlog
SHOW VARIABLES LIKE 'log_bin%';
SHOW BINARY LOGS;
```

⚠️ **Attention** : Le **general log** ne doit **jamais** être activé en production (sauf debug ponctuel) car il enregistre TOUTES les requêtes et dégrade significativement les performances.

### 11.5 Binary logs et réplication

Les binary logs (binlogs) sont critiques pour :

- La **réplication** (master-slave, Galera)
- Le **Point-in-Time Recovery** (PITR)
- L'**audit** des modifications

Trois formats disponibles :

1. **STATEMENT** : Enregistre les requêtes SQL (compact mais non-déterministe)
2. **ROW** : Enregistre les changements de lignes (déterministe, taille importante)
3. **MIXED** : Hybride intelligent (recommandé)

```sql
-- Vérifier le format binlog
SHOW VARIABLES LIKE 'binlog_format';

-- Lister les binary logs
SHOW BINARY LOGS;

-- Purger les anciens logs (attention !)
PURGE BINARY LOGS BEFORE '2025-12-01 00:00:00';
```

### 11.6 Maintenance des tables

La maintenance régulière garantit des **performances optimales** et la **santé des données**. Quatre opérations essentielles :

#### OPTIMIZE TABLE
Réorganise les données et récupère l'espace fragmenté.

```sql
-- Optimiser une table InnoDB
OPTIMIZE TABLE ma_table;

-- Optimiser toutes les tables d'une base
SELECT CONCAT('OPTIMIZE TABLE ', table_schema, '.', table_name, ';')
FROM information_schema.tables
WHERE table_schema = 'ma_base';
```

💡 **Conseil** : Pour InnoDB, `OPTIMIZE TABLE` réécrit complètement la table. Planifiez cette opération pendant les fenêtres de maintenance.

#### ANALYZE TABLE
Met à jour les statistiques de distribution des données pour l'optimiseur de requêtes.

```sql
-- Analyser une table
ANALYZE TABLE orders;

-- Analyser en mode persistant (statistiques sauvegardées)
SET GLOBAL innodb_stats_persistent = ON;
ANALYZE TABLE orders PERSISTENT FOR ALL;
```

#### CHECK TABLE
Vérifie l'intégrité physique et logique d'une table.

```sql
-- Vérification rapide
CHECK TABLE users FAST;

-- Vérification complète
CHECK TABLE users EXTENDED;
```

#### REPAIR TABLE
Répare une table corrompue (principalement MyISAM/Aria).

```sql
-- Réparer une table MyISAM
REPAIR TABLE old_myisam_table;
```

⚠️ **Attention** : `REPAIR TABLE` ne fonctionne **pas** sur les tables InnoDB. En cas de corruption InnoDB, utilisez `innodb_force_recovery`.

### 11.7 Gestion de l'espace disque

La surveillance de l'espace disque est critique pour éviter les interruptions de service. Points clés :

- Monitoring de `/var/lib/mysql` (datadir)
- Monitoring des logs (`/var/log/mysql`)
- Surveillance des tables temporaires (`/tmp`)
- Gestion de l'`innodb_undo_tablespace`

```sql
-- Taille totale des bases de données
SELECT
    table_schema AS 'Base de données',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Taille (MB)'
FROM information_schema.tables
GROUP BY table_schema
ORDER BY SUM(data_length + index_length) DESC;

-- Top 10 des plus grosses tables
SELECT
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Taille (MB)'
FROM information_schema.tables
WHERE table_schema = 'ma_base'
ORDER BY (data_length + index_length) DESC
LIMIT 10;
```

### 11.8 🆕 Contrôle de l'espace temporaire

**Nouveauté MariaDB 11.8** : Deux nouvelles variables pour prévenir la saturation du disque temporaire.

```sql
-- Limite par session (connexion)
SET GLOBAL max_tmp_space_usage = 10737418240; -- 10 GB

-- Limite globale (tout le serveur)
SET GLOBAL max_total_tmp_space_usage = 107374182400; -- 100 GB
```

**Cas d'usage** :
- Protéger contre les requêtes générant d'énormes tables temporaires
- Éviter les `Out of disk space` sur `/tmp`
- Garantir la stabilité en environnement multi-tenant

```sql
-- Surveiller l'utilisation actuelle
SELECT
    THREAD_ID,
    USER,
    TMP_TABLES_SIZE
FROM performance_schema.threads
WHERE TMP_TABLES_SIZE > 0
ORDER BY TMP_TABLES_SIZE DESC;
```

💡 **Conseil** : Combinez avec `tmp_table_size` et `max_heap_table_size` pour une gestion complète de la mémoire temporaire.

### 11.9 Monitoring et métriques importantes

Un DBA efficace surveille en permanence ces métriques clés :

#### Métriques de performance

```sql
-- Taux de cache du buffer pool InnoDB
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- Calcul du hit rate
SELECT
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS buffer_pool_hit_rate
FROM
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_reads
     FROM information_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') AS reads,
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
     FROM information_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS requests;
```

#### Métriques de connexions

```sql
-- Connexions actives
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';

-- Vérifier les rejets de connexions
SHOW STATUS LIKE 'Connection_errors%';
```

#### Métriques de verrous et concurrence

```sql
-- Détection de deadlocks
SHOW STATUS LIKE 'Innodb_deadlocks';

-- Temps d'attente des verrous
SHOW STATUS LIKE 'Innodb_row_lock_time';
```

### 11.10 Thread Pool

Le **Thread Pool** améliore la gestion de la concurrence en réutilisant les threads au lieu d'en créer un par connexion.

**Avantages** :
- Réduit l'overhead de création/destruction de threads
- Meilleure performance avec **beaucoup de connexions courtes**
- Limite le nombre de threads actifs

```ini
# Configuration dans my.cnf
[mariadb]
thread_handling = pool-of-threads
thread_pool_size = 16  # Généralement = nombre de CPU cores
thread_pool_max_threads = 1000
thread_pool_idle_timeout = 60
thread_pool_stall_limit = 500  # ms avant création nouveau thread
```

```sql
-- Vérifier le Thread Pool
SHOW VARIABLES LIKE 'thread_pool%';

-- Statistiques du Thread Pool
SHOW STATUS LIKE 'Threadpool%';
```

💡 **Conseil** : Le Thread Pool est particulièrement efficace pour les charges de travail **OLTP** avec de nombreuses connexions simultanées de courte durée.

### 11.11 🆕 Charset par défaut : utf8mb4 avec UCA 14.0.0

**Nouveauté MariaDB 11.8** : `utf8mb4` est maintenant le charset **par défaut**, avec les collations **UCA 14.0.0** (Unicode Collation Algorithm).

#### Changements majeurs

```sql
-- Avant MariaDB 11.8
DEFAULT CHARACTER SET = latin1
DEFAULT COLLATION = latin1_swedish_ci

-- Depuis MariaDB 11.8
DEFAULT CHARACTER SET = utf8mb4
DEFAULT COLLATION = utf8mb4_uca1400_ai_ci
```

#### Nouvelles collations UCA 14.0.0

| Collation | Sensibilité | Usage |
|-----------|-------------|-------|
| `utf8mb4_uca1400_ai_ci` | Accent-insensible, case-insensitive | Défaut |
| `utf8mb4_uca1400_as_cs` | Accent-sensible, case-sensitive | Données précises |
| `utf8mb4_uca1400_nopad_ai_ci` | NO PAD (espaces significatifs) | Comparaisons strictes |

#### Impact migration

```sql
-- Vérifier le charset actuel d'une base
SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'ma_base';

-- Convertir une base existante
ALTER DATABASE ma_base
    CHARACTER SET = utf8mb4
    COLLATE = utf8mb4_uca1400_ai_ci;

-- Convertir une table
ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

⚠️ **Attention** : La conversion peut nécessiter plus d'espace disque car `utf8mb4` utilise jusqu'à 4 octets par caractère (vs 3 pour `utf8mb3`).

💡 **Avantage** : Support natif complet de tous les emojis, symboles mathématiques, et caractères Unicode modernes 🚀 ✨ 🎉

### 11.12 🆕 Extension TIMESTAMP 2106 (résolution Y2038)

**Nouveauté MariaDB 11.8** : Extension de la plage TIMESTAMP jusqu'à **2106**, résolvant le problème **Y2038**.

#### Problème Y2038

Le type `TIMESTAMP` traditionnel utilisait un entier signé 32 bits pour stocker les secondes depuis l'epoch (1970-01-01), limitant la date maximale au **19 janvier 2038 03:14:07 UTC**.

#### Solution MariaDB 11.8

```sql
-- Ancienne limite (avant 11.8)
'1970-01-01 00:00:01' UTC à '2038-01-19 03:14:07' UTC

-- Nouvelle limite (depuis 11.8)
'1970-01-01 00:00:01' UTC à '2106-02-07 06:28:15' UTC
```

#### Format de stockage

MariaDB 11.8 utilise maintenant :
- **7 octets** au lieu de 4 pour les nouveaux champs TIMESTAMP
- Rétrocompatibilité complète avec les anciennes tables

```sql
-- Créer une table avec TIMESTAMP étendu
CREATE TABLE events (
    id INT PRIMARY KEY AUTO_INCREMENT,
    event_name VARCHAR(100),
    event_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    future_date TIMESTAMP  -- Peut stocker des dates jusqu'en 2106
);

-- Tester la nouvelle plage
INSERT INTO events (event_name, future_date)
VALUES ('Event futur', '2100-01-01 00:00:00');

-- Vérification
SELECT event_name, future_date
FROM events
WHERE future_date > '2038-01-19';
```

#### Migration des tables existantes

```sql
-- Les tables existantes continuent de fonctionner
-- Pas de migration automatique nécessaire

-- Pour bénéficier de l'extension, recréer le champ :
ALTER TABLE old_table
MODIFY timestamp_column TIMESTAMP;
```

💡 **Conseil** : Si vous utilisez des `System-Versioned Tables`, consultez le chapitre 19 (Migration) car le format de stockage des timestamps historiques a changé.

---

## Stratégie de configuration en production

### 1. Configuration initiale

```bash
# Créer une arborescence modulaire
/etc/mysql/
├── my.cnf                    # Configuration principale
├── conf.d/                   # Configurations modulaires
│   ├── 01-base.cnf          # Paramètres de base
│   ├── 02-innodb.cnf        # Configuration InnoDB
│   ├── 03-replication.cnf   # Réplication (si applicable)
│   └── 04-monitoring.cnf    # Logs et monitoring
└── mariadb.conf.d/          # Spécifique MariaDB
```

### 2. Surveillance continue

Métriques à surveiller quotidiennement :
- ✅ Espace disque (datadir, logs, tmp)
- ✅ Buffer pool hit rate (> 99%)
- ✅ Slow queries (tendance)
- ✅ Connexions refusées
- ✅ Réplication lag (si applicable)
- ✅ Deadlocks et lock waits

### 3. Maintenance régulière

| Tâche | Fréquence | Commande |
|-------|-----------|----------|
| Analyser les tables | Hebdomadaire | `ANALYZE TABLE` |
| Purger les binlogs | Quotidienne | `PURGE BINARY LOGS` |
| Vérifier les logs d'erreur | Quotidienne | `tail -f /var/log/mysql/error.log` |
| Optimiser les tables | Mensuelle | `OPTIMIZE TABLE` |
| Sauvegardes complètes | Quotidienne | `mariabackup` |

---

## Outils d'administration essentiels

### Ligne de commande

```bash
# Client MariaDB
mariadb -u root -p

# Administration
mariadb-admin ping
mariadb-admin processlist
mariadb-admin status
mariadb-admin variables

# Logs
mariadb-binlog /var/lib/mysql/mysql-bin.000001
mariadb-dumpslow /var/log/mysql/slow-query.log
```

### Scripts de monitoring

```sql
-- Script de vérification de santé
SELECT
    'Buffer Pool Hit Rate' AS metric,
    CONCAT(ROUND((1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100, 2), '%') AS value
FROM
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_reads FROM information_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') r,
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_read_requests FROM information_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') rr

UNION ALL

SELECT
    'Connexions actives',
    VARIABLE_VALUE
FROM information_schema.global_status
WHERE VARIABLE_NAME = 'Threads_connected'

UNION ALL

SELECT
    'Slow queries',
    VARIABLE_VALUE
FROM information_schema.global_status
WHERE VARIABLE_NAME = 'Slow_queries';
```

---

## ✅ Points clés à retenir

- **Configuration hiérarchique** : my.cnf permet une organisation modulaire et maintenable
- **Variables dynamiques** : Modifiables à chaud avec `SET GLOBAL`, mais nécessitent my.cnf pour la persistance
- **sql_mode** : Détermine le niveau de rigueur SQL et la compatibilité
- **4 types de logs** : Error (essentiel), Slow Query (diagnostic), General (debug uniquement), Binary (réplication/PITR)
- **Maintenance régulière** : ANALYZE (statistiques), OPTIMIZE (défragmentation), CHECK (intégrité)
- **Thread Pool** : Améliore les performances pour les charges haute concurrence
- 🆕 **Contrôle espace temporaire** : Nouvelles limites `max_tmp_space_usage` et `max_total_tmp_space_usage`
- 🆕 **utf8mb4 par défaut** : Unicode complet natif avec collations UCA 14.0.0
- 🆕 **TIMESTAMP 2106** : Résolution du problème Y2038, plage étendue jusqu'en 2106
- **Monitoring proactif** : Buffer pool, connexions, slow queries, espace disque
- **Sauvegarde before modify** : Toujours sauvegarder my.cnf avant modification

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Server System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [📖 Documentation officielle - sql_mode](https://mariadb.com/kb/en/sql-mode/)
- [📖 Documentation officielle - Binary Log](https://mariadb.com/kb/en/binary-log/)
- [📖 Documentation officielle - Thread Pool](https://mariadb.com/kb/en/thread-pool-in-mariadb/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
- [📖 Unicode Collation Algorithm 14.0.0](https://unicode.org/reports/tr10/)
- [🔧 Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit) - Outils d'administration
- [📊 PMM (Percona Monitoring and Management)](https://www.percona.com/software/database-tools/percona-monitoring-and-management)

---

## ➡️ Section suivante

**[11.1 Fichiers de configuration](./01-fichiers-configuration.md)** : Structure détaillée de my.cnf/my.ini, ordre de lecture, organisation en sections et configuration modulaire pour une administration professionnelle.

---

**💡 Conseil final** : L'administration MariaDB est un **processus continu**, pas un événement ponctuel. Établissez des routines de surveillance et de maintenance dès le départ, et documentez systématiquement vos modifications de configuration. Votre "vous" du futur vous remerciera ! 🚀

⏭️ [Fichiers de configuration](/11-administration-configuration/01-fichiers-configuration.md)

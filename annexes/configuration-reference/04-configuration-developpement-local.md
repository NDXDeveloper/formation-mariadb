🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.4 Configuration Développement Local (Dev, Testing, CI/CD)

> **Type** : Configuration de référence  
> **Cas d'usage** : Développement local, tests, CI/CD  
> **Caractéristiques** : Ressources minimales, debug-friendly, rapidité  
> **Public** : Développeurs, QA, DevOps

---

## 🎯 Profil du cas d'usage Développement

### Caractéristiques environnement de développement

**Développement Local** désigne les environnements de travail développeurs, testing et CI/CD :

#### Patterns d'accès typiques
- 💻 **Laptop/VM** : Ressources limitées (8-16GB RAM, 4-8 cores)
- 🚀 **Démarrage rapide** : < 5 secondes (redémarrages fréquents)
- 🔍 **Debug intensif** : Logs détaillés, slow queries, explain
- 🧪 **Tests** : Unitaires, intégration, E2E répétés
- 🔄 **Cycles courts** : Modification code → Test → Rebuild
- 📦 **Docker/Containers** : Environnements jetables, reproductibles

#### Exemples d'environnements
- 💻 **Dev Laptop** : MariaDB local sur macOS/Linux/Windows
- 🐳 **Docker Compose** : Stack complète (app + DB) pour dev
- 🧪 **CI/CD** : GitHub Actions, GitLab CI, Jenkins pipelines
- 🎭 **Staging** : Environnement pré-production pour QA
- 📚 **Documentation** : Exemples tutorials, demos

### Objectifs prioritaires DEV

| Objectif | Importance | Compromis production |
|----------|------------|---------------------|
| **Rapidité démarrage** | ⭐⭐⭐⭐⭐ | Durabilité |
| **Debugging facile** | ⭐⭐⭐⭐⭐ | Performance |
| **Ressources minimales** | ⭐⭐⭐⭐⭐ | Scalabilité |
| **Reproductibilité** | ⭐⭐⭐⭐ | Optimisations |
| **Compatibilité outils** | ⭐⭐⭐⭐ | Sécurité stricte |

### DEV vs PRODUCTION - Différences majeures

| Aspect | PRODUCTION | DÉVELOPPEMENT |
|--------|------------|---------------|
| **Durabilité** | ACID strict (=1) | Relaxed (=0 ou =2) |
| **RAM** | 60-80% serveur | 256M-2G max |
| **Logging** | Minimal (perf) | Verbose (debug) |
| **Sécurité** | Strict (TLS, auth) | Relaxed (localhost) |
| **Démarrage** | ~30s acceptable | < 5s impératif |
| **Crash** | Catastrophe | Rebuild 10s |

---

## 📝 Configuration my.cnf complète - Développement

### Configuration pour laptop/VM locale (8GB RAM)

```ini
# ============================================================================
# MARIADB 11.8 LTS - CONFIGURATION DÉVELOPPEMENT LOCAL
# ============================================================================
# Cas d'usage : Dev local, testing, CI/CD, apprentissage
# Matériel cible : Laptop/VM 8GB RAM, 4 cores
# Priorités : Rapidité, Debug, Ressources minimales
# Version : MariaDB 11.8 LTS
# Dernière mise à jour : Décembre 2025
# ============================================================================
# ⚠️ CETTE CONFIG N'EST PAS POUR PRODUCTION !
# ============================================================================

[client]
port                        = 3306
socket                      = /var/run/mysqld/mysqld.sock
default-character-set       = utf8mb4

[mysqld]

# ----------------------------------------------------------------------------
# IDENTITÉ SERVEUR
# ----------------------------------------------------------------------------
port                        = 3306
socket                      = /var/run/mysqld/mysqld.sock
pid-file                    = /var/run/mysqld/mysqld.pid
datadir                     = /var/lib/mysql

user                        = mysql

# Écoute localhost uniquement (sécurité dev)
bind-address                = 127.0.0.1
# Docker : utiliser 0.0.0.0 si accès depuis host

# ----------------------------------------------------------------------------
# CHARSET ET COLLATION (🆕 MariaDB 11.8)
# ----------------------------------------------------------------------------
character-set-server        = utf8mb4
collation-server            = utf8mb4_unicode_ci

# ----------------------------------------------------------------------------
# MOTEUR DE STOCKAGE
# ----------------------------------------------------------------------------
default-storage-engine      = InnoDB

# ----------------------------------------------------------------------------
# CONNEXIONS ET THREADS
# ----------------------------------------------------------------------------
max_connections             = 50
# Dev : 10-50 connexions largement suffisant
# Tests unitaires : ~5-10 connexions parallèles
# Économise RAM

max_connect_errors          = 1000000
# Éviter blocage pendant debug (tentatives répétées)

thread_cache_size           = 8
# Petit cache suffisant (peu de connexions)

# Pas de thread pool en dev (overhead inutile)
# thread_handling = one-thread-per-connection (défaut)

# ----------------------------------------------------------------------------
# TABLES ET CACHE
# ----------------------------------------------------------------------------
table_open_cache            = 400
# Dev : Peu de tables ouvertes simultanément

table_definition_cache      = 200

open_files_limit            = 1024
# Minimal pour économiser file descriptors

# ----------------------------------------------------------------------------
# MÉMOIRE GLOBALE - INNODB BUFFER POOL (🔥 MINIMAL)
# ----------------------------------------------------------------------------
innodb_buffer_pool_size     = 1G
# 🔥 DEV : 1GB suffisant pour petit dataset
# Laptop 8GB : 1G buffer pool OK
# Laptop 16GB : 2G possible
# CI/CD : 256M-512M minimal

innodb_buffer_pool_instances = 1
# 1 instance pour <2GB buffer pool

innodb_buffer_pool_chunk_size = 128M

# Pas de dump/load du buffer pool (démarrage plus rapide)
innodb_buffer_pool_load_at_startup = 0
innodb_buffer_pool_dump_at_shutdown = 0

# ----------------------------------------------------------------------------
# MÉMOIRE PAR CONNEXION (MINIMAL)
# ----------------------------------------------------------------------------
read_buffer_size            = 128K
# Minimal (vs 256K+ production)

read_rnd_buffer_size        = 256K

sort_buffer_size            = 256K
# Dev : Queries simples, datasets petits

join_buffer_size            = 128K

# Calcul mémoire dev :
# 128K + 256K + 256K + 128K ≈ 768K par connexion
# 50 connexions × 768K ≈ 38MB
# Très économe !

# ----------------------------------------------------------------------------
# TEMPORAIRES ET HEAP
# ----------------------------------------------------------------------------
tmp_table_size              = 32M
max_heap_table_size         = 32M
# Dev : Tables temporaires petites

# 🆕 MariaDB 11.8
max_tmp_space_usage         = 2G
# Limite espace temporaire (protection queries mal écrites)

# ----------------------------------------------------------------------------
# LOGS INNODB - REDO LOG (MINIMAL + RAPIDE)
# ----------------------------------------------------------------------------
innodb_log_file_size        = 256M
# Dev : Petit redo log = démarrage ultra rapide
# 256M × 2 = 512MB total (vs 2-4GB production)

innodb_log_files_in_group   = 2

innodb_log_buffer_size      = 8M
# Minimal (peu d'écritures simultanées)

innodb_flush_log_at_trx_commit = 0
# 🔥 DEV : PERFORMANCE > DURABILITÉ
# 0 = Flush toutes les secondes (pas à chaque commit)
# Crash = perte max 1s transactions
# OK dev : crash → docker-compose down/up (10s rebuild)

# ⚠️ JAMAIS utiliser = 0 en production !

# ----------------------------------------------------------------------------
# I/O ET DISQUES (MINIMAL)
# ----------------------------------------------------------------------------
innodb_io_capacity          = 200
# Dev : SSD laptop typique, valeur conservatrice

innodb_io_capacity_max      = 400

innodb_flush_method         = O_DIRECT
# Même en dev, évite double buffering

innodb_flush_neighbors      = 0
# SSD standard laptops

# 🆕 MariaDB 11.8
innodb_alter_copy_bulk      = ON

# Read-ahead : Défaut OK
innodb_read_ahead_threshold = 56

# ----------------------------------------------------------------------------
# UNDO LOG (MINIMAL)
# ----------------------------------------------------------------------------
innodb_undo_tablespaces     = 2
innodb_undo_log_truncate    = ON
innodb_max_undo_log_size    = 256M
# Petits undo logs dev

# ----------------------------------------------------------------------------
# BINARY LOGS (OPTIONNEL DEV)
# ----------------------------------------------------------------------------
# Désactivé par défaut (économie espace + perf)
# Activer uniquement si test réplication

# log_bin                   = /var/log/mysql/mysql-bin
# binlog_format             = ROW
# expire_logs_days          = 1  # Purge rapide
# max_binlog_size           = 100M

# Pour dev sans réplication :
skip-log-bin                = 1
# Gain : 10-20% perf writes, pas d'espace disque binlog

# ----------------------------------------------------------------------------
# SÉCURITÉ (RELAXED DEV)
# ----------------------------------------------------------------------------
# ⚠️ Développement localhost uniquement, sécurité relaxed

# Pas de TLS en dev local
# ssl = 0

# Authentication simple (dev)
# default_authentication_plugin = mysql_native_password

# Pas de validation password stricte
# validate_password.policy = LOW

# ----------------------------------------------------------------------------
# QUERY CACHE (⚠️ DÉPRÉCIÉ)
# ----------------------------------------------------------------------------
query_cache_type            = 0
query_cache_size            = 0

# ----------------------------------------------------------------------------
# OPTIMIZER (DÉFAUT OK DEV)
# ----------------------------------------------------------------------------
optimizer_search_depth      = 62
# Défaut OK, queries dev généralement simples

optimizer_switch             = 'mrr=on,mrr_cost_based=on,index_condition_pushdown=on'

innodb_stats_persistent     = ON
innodb_stats_auto_recalc    = ON
innodb_stats_persistent_sample_pages = 20

# ----------------------------------------------------------------------------
# MONITORING ET PERFORMANCE SCHEMA (🔥 ACTIVÉ POUR DEBUG)
# ----------------------------------------------------------------------------
performance_schema          = ON
# 🔥 ESSENTIEL DEV : Debug queries, profiling
# Overhead ~5% acceptable dev

# Instruments activés par défaut suffisants
# Ajuster si profiling spécifique nécessaire

# ----------------------------------------------------------------------------
# LOGS (🔥 VERBOSE POUR DEBUG)
# ----------------------------------------------------------------------------
log_error                   = /var/log/mysql/error.log

# 🔥 Slow query log : TRÈS PERMISSIF
slow_query_log              = ON
slow_query_log_file         = /var/log/mysql/slow-query.log
long_query_time             = 0.1
# Dev : Log queries > 100ms (vs 0.5-2s production)
# Identifier immédiatement problèmes perf

log_slow_verbosity          = query_plan,explain
# 🔥 Log plan d'exécution (EXPLAIN auto)
# Indispensable pour comprendre queries lentes

log_queries_not_using_indexes = ON
# 🔥 Log toute query sans index
# Aide identifier missing indexes

min_examined_row_limit      = 0
# Log même queries examinant peu de lignes

# 🔥 General log (optionnel, très verbose)
# Activer ponctuellement pour debug
general_log                 = OFF
general_log_file            = /var/log/mysql/general.log

# Pour activer temporairement :
# SET GLOBAL general_log = ON;
# ... debug ...
# SET GLOBAL general_log = OFF;

# ----------------------------------------------------------------------------
# TIMEOUTS (COURTS)
# ----------------------------------------------------------------------------
wait_timeout                = 300
# 5 minutes : Connexions dev courtes
# Ferme connexions idle (économise ressources)

interactive_timeout         = 300

connect_timeout             = 10

net_read_timeout            = 30
net_write_timeout           = 60

# ----------------------------------------------------------------------------
# AUTRES PARAMÈTRES
# ----------------------------------------------------------------------------
max_allowed_packet          = 64M
# Dev : Suffisant pour la plupart use cases

lower_case_table_names      = 0
# 0 = case-sensitive (Linux/macOS)
# 1 = case-insensitive (Windows)
# ⚠️ Fixer selon OS dev pour éviter surprises

sql_mode                    = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'
# Mode strict : Détecte erreurs tôt

# Optionnel dev : Mode encore plus strict
# sql_mode = 'TRADITIONAL'

# ----------------------------------------------------------------------------
# INNODB AVANCÉ (MINIMAL)
# ----------------------------------------------------------------------------
innodb_adaptive_hash_index  = OFF
# Dev : Datasets petits, pas utile

innodb_change_buffering     = none
# Dev : Peu d'écritures massives

innodb_old_blocks_time      = 0
# Dev : Pas de stratégie LRU complexe

innodb_print_all_deadlocks  = ON
# 🔥 DEBUG : Log tous deadlocks dans error log
# Aide debug contention

# ----------------------------------------------------------------------------
# DÉVELOPPEMENT - PARAMÈTRES SPÉCIFIQUES
# ----------------------------------------------------------------------------

# Auto-reconnect (utile IDE, scripts dev)
# Côté client, pas serveur
# [client]
# auto-reconnect = 1

# SQL Warnings (afficher warnings en plus d'errors)
# SET SESSION sql_warnings = ON;

# ----------------------------------------------------------------------------
[mysqldump]
quick
quote-names
max_allowed_packet          = 64M

# ----------------------------------------------------------------------------
[mysql]
# Client CLI : Améliorations UX dev
default-character-set       = utf8mb4

# Auto-completion commandes (pratique dev)
auto-rehash                 = 1

# Afficher warnings
show-warnings               = 1

# Prompt informatif
# prompt                    = '\\u@\\h [\\d]> '

# ----------------------------------------------------------------------------
# FIN DE CONFIGURATION
# ============================================================================
```

---

## 🐳 Configuration Docker / Docker Compose

### Dockerfile MariaDB dev

```dockerfile
# Dockerfile - MariaDB 11.8 optimisé développement
FROM mariadb:11.8

# Variables environnement
ENV MARIADB_ROOT_PASSWORD=devpassword
ENV MARIADB_DATABASE=myapp_dev
ENV MARIADB_USER=dev
ENV MARIADB_PASSWORD=devpass

# Copier config custom
COPY my-dev.cnf /etc/mysql/conf.d/

# Script initialisation (seeds, fixtures)
COPY init-db.sql /docker-entrypoint-initdb.d/

# Exposer port
EXPOSE 3306

# Health check
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD mariadb-admin ping -h localhost -u root -p${MARIADB_ROOT_PASSWORD} || exit 1
```

### docker-compose.yml - Stack développement

```yaml
# docker-compose.yml - Stack dev complète
version: '3.8'

services:
  mariadb:
    image: mariadb:11.8
    container_name: myapp_mariadb_dev
    
    # Config optimisée dev
    command: 
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --max-connections=50
      - --innodb-buffer-pool-size=1G
      - --innodb-flush-log-at-trx-commit=0  # Perf > Durabilité
      - --slow-query-log=ON
      - --long-query-time=0.1
      - --log-queries-not-using-indexes=ON
      - --skip-log-bin  # Pas de binlog dev
    
    environment:
      MARIADB_ROOT_PASSWORD: devroot
      MARIADB_DATABASE: myapp_dev
      MARIADB_USER: developer
      MARIADB_PASSWORD: devpass
    
    ports:
      - "3306:3306"
    
    volumes:
      # Données persistantes (optionnel, peut être volatile)
      - mariadb_data:/var/lib/mysql
      
      # Config custom
      - ./docker/mariadb/my-dev.cnf:/etc/mysql/conf.d/my-dev.cnf
      
      # Scripts init (seeds)
      - ./docker/mariadb/init:/docker-entrypoint-initdb.d
      
      # Logs accessibles depuis host
      - ./docker/mariadb/logs:/var/log/mysql
    
    # Limites ressources (économie laptop)
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          memory: 512M
    
    # Health check
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-h", "localhost", "-u", "root", "-pdevroot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    
    networks:
      - dev_network

  # Application (exemple)
  app:
    build: .
    container_name: myapp_app_dev
    depends_on:
      mariadb:
        condition: service_healthy
    environment:
      DB_HOST: mariadb
      DB_PORT: 3306
      DB_NAME: myapp_dev
      DB_USER: developer
      DB_PASSWORD: devpass
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    networks:
      - dev_network

volumes:
  mariadb_data:
    # Volume nommé (persistant)
    # Ou tmpfs pour volatilité totale (CI/CD)

networks:
  dev_network:
    driver: bridge
```

### Script init-db.sql (Seeds/Fixtures)

```sql
-- docker/mariadb/init/01-schema.sql
-- Exécuté automatiquement au premier démarrage

-- Schéma de base
CREATE TABLE IF NOT EXISTS users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS posts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB;

-- Données de test (fixtures)
INSERT INTO users (email, username) VALUES
    ('alice@example.com', 'alice'),
    ('bob@example.com', 'bob'),
    ('charlie@example.com', 'charlie');

INSERT INTO posts (user_id, title, content) VALUES
    (1, 'First Post', 'Hello World!'),
    (1, 'Second Post', 'Testing MariaDB 11.8'),
    (2, 'Bob Post', 'Docker is great');

-- User dev avec tous privilèges
GRANT ALL PRIVILEGES ON myapp_dev.* TO 'developer'@'%';
FLUSH PRIVILEGES;
```

---

## 🧪 Configuration pour Tests (CI/CD)

### GitHub Actions - Tests avec MariaDB

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mariadb:
        image: mariadb:11.8
        env:
          MARIADB_ROOT_PASSWORD: testroot
          MARIADB_DATABASE: test_db
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mariadb-admin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
          --character-set-server=utf8mb4
          --collation-server=utf8mb4_unicode_ci
          --max-connections=50
          --innodb-buffer-pool-size=256M
          --innodb-flush-log-at-trx-commit=0
          --skip-log-bin
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Wait for MariaDB
        run: |
          until mariadb -h 127.0.0.1 -u root -ptestroot -e "SELECT 1"; do
            echo "Waiting for MariaDB..."
            sleep 2
          done
      
      - name: Run migrations
        run: |
          mariadb -h 127.0.0.1 -u root -ptestroot test_db < migrations/schema.sql
      
      - name: Run tests
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_NAME: test_db
          DB_USER: root
          DB_PASSWORD: testroot
        run: |
          # Vos tests ici
          npm test
          # ou python -m pytest
          # ou go test ./...
```

### GitLab CI - Pipeline avec MariaDB

```yaml
# .gitlab-ci.yml
test:
  image: node:18
  
  services:
    - name: mariadb:11.8
      alias: mariadb
      command:
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
        - --max-connections=50
        - --innodb-buffer-pool-size=256M
        - --innodb-flush-log-at-trx-commit=0
        - --skip-log-bin
  
  variables:
    MARIADB_ROOT_PASSWORD: testpassword
    MARIADB_DATABASE: test_db
    DB_HOST: mariadb
    DB_PORT: 3306
  
  before_script:
    - apt-get update && apt-get install -y mariadb-client
    - |
      until mariadb -h mariadb -u root -p${MARIADB_ROOT_PASSWORD} -e "SELECT 1"; do
        echo "Waiting for MariaDB..."
        sleep 2
      done
    - mariadb -h mariadb -u root -p${MARIADB_ROOT_PASSWORD} ${MARIADB_DATABASE} < schema.sql
  
  script:
    - npm install
    - npm test
```

---

## 🔍 Debug et Profiling

### Activer logging détaillé temporairement

```sql
-- Session de debug : activer logs temporaires

-- 1. General log (toutes les queries)
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/debug-general.log';

-- Exécuter queries à debugger...
SELECT * FROM users WHERE email LIKE '%test%';

-- Désactiver
SET GLOBAL general_log = OFF;

-- 2. Profiling query spécifique
SET SESSION profiling = ON;

-- Query à profiler
SELECT u.*, COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id;

-- Voir profiling
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;

-- Détail CPU/block IO
SHOW PROFILE CPU, BLOCK IO FOR QUERY 1;

SET SESSION profiling = OFF;
```

### Explain queries avec Performance Schema

```sql
-- Analyser query lente avec détails

EXPLAIN ANALYZE
SELECT u.username, p.title
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY);

-- Résultat inclut :
-- - Plan d'exécution
-- - Temps réel par étape
-- - Nombre lignes traitées

-- Performance Schema : Stats récentes
SELECT 
    DIGEST_TEXT AS query,
    COUNT_STAR AS executions,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_sec,
    ROUND(MAX_TIMER_WAIT / 1000000000000, 3) AS max_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'myapp_dev'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### Identifier missing indexes

```sql
-- Queries sans index (depuis slow log)
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_NO_INDEX_USED,
    SUM_NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
   OR SUM_NO_GOOD_INDEX_USED > 0
ORDER BY SUM_NO_INDEX_USED DESC;

-- Index jamais utilisés (candidats suppression)
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NOT NULL
  AND COUNT_STAR = 0
  AND OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema')
ORDER BY OBJECT_SCHEMA, OBJECT_NAME;
```

---

## 💡 Outils et IDE Integration

### Visual Studio Code - Extensions

```json
// .vscode/settings.json
{
    "sqltools.connections": [
        {
            "name": "MariaDB Dev",
            "driver": "MariaDB",
            "server": "localhost",
            "port": 3306,
            "database": "myapp_dev",
            "username": "developer",
            "password": "devpass",
            "connectionTimeout": 30
        }
    ],
    
    // Auto-format SQL
    "sqltools.format": {
        "language": "sql",
        "indentSize": 2
    }
}
```

### DataGrip / DBeaver / HeidiSQL

**Configuration connexion :**
```
Type : MariaDB
Host : localhost
Port : 3306
Database : myapp_dev
User : developer
Password : devpass

Propriétés avancées :
- useSSL = false (dev local)
- allowPublicKeyRetrieval = true
- serverTimezone = UTC
```

### MySQL Workbench - Connexion dev

```
Connection Name : MariaDB 11.8 Dev
Hostname : localhost
Port : 3306
Username : developer

Advanced :
- Default Schema : myapp_dev
- SQL_MODE : (laisser défaut)
```

---

## 📊 Monitoring Dev (optionnel)

### Adminer - Interface web légère

```yaml
# docker-compose.yml - Ajouter Adminer
services:
  adminer:
    image: adminer:latest
    container_name: myapp_adminer
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: mariadb
    depends_on:
      - mariadb
    networks:
      - dev_network

# Accès : http://localhost:8080
# Serveur : mariadb
# Utilisateur : developer
# Mot de passe : devpass
# Base : myapp_dev
```

### phpMyAdmin (alternatif)

```yaml
services:
  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    container_name: myapp_phpmyadmin
    ports:
      - "8081:80"
    environment:
      PMA_HOST: mariadb
      PMA_PORT: 3306
      PMA_USER: developer
      PMA_PASSWORD: devpass
    depends_on:
      - mariadb
    networks:
      - dev_network
```

---

## 🔍 Explication des choix clés DEV

### 1. innodb_flush_log_at_trx_commit = 0 (Performance > Durabilité)

```ini
# PRODUCTION : innodb_flush_log_at_trx_commit = 1
# DEV : innodb_flush_log_at_trx_commit = 0
```

**Impact dev :**

| Valeur | Durabilité | Perf | Cas dev |
|--------|------------|------|---------|
| **0** | ❌ Perte 1s | ✅✅✅ +40% | **Recommandé dev** |
| **1** | ✅ ACID | ⚡ Baseline | Tests intégration |
| **2** | ⚠️ Perte si crash OS | ✅✅ +25% | Staging |

**Pourquoi = 0 en dev :**
```
Crash dev = docker-compose restart (10s)
→ Données perdues rechargées depuis seeds/fixtures
→ Aucun impact (environnement jetable)

Gain :
- Inserts : 40% plus rapide
- Démarrage : 2× plus rapide (redo log petit)
```

### 2. skip-log-bin (Pas de binary logs)

```ini
skip-log-bin = 1
```

**Bénéfices dev :**
- ✅ Writes 10-20% plus rapides
- ✅ Économie espace disque (binlog croît vite)
- ✅ Démarrage plus rapide (pas de recovery binlog)

**Activer uniquement si :**
- Test réplication
- Test PITR (Point-In-Time Recovery)
- Debug binlog format

### 3. Buffer Pool 1GB (vs 20-96GB production)

```ini
# Production 32GB serveur : 20GB buffer pool (62%)
# Dev 8GB laptop : 1GB buffer pool (12.5%)
```

**Dimensionnement dev :**

```
Laptop 8GB RAM :
├─ OS + Apps : 4GB
├─ IDE (VSCode) : 1GB
├─ Browser : 1GB
├─ MariaDB buffer pool : 1GB
├─ MariaDB autres : 256MB
└─ Marge : 750MB
───────────────
Total : 8GB

Dataset dev typique :
- 100K-1M lignes
- 100MB-1GB données
→ 1GB buffer pool largement suffisant
```

### 4. Logging verbose (Debug > Performance)

```ini
slow_query_log = ON
long_query_time = 0.1  # 100ms (vs 0.5-2s production)
log_queries_not_using_indexes = ON
innodb_print_all_deadlocks = ON
```

**Trade-off dev :**
- ✅ Identifier problèmes immédiatement
- ✅ Apprendre bonnes pratiques SQL
- ⚠️ Overhead 5-10% (acceptable dev)

**Exemple utilisation :**
```bash
# Surveiller slow query log en temps réel
tail -f /var/log/mysql/slow-query.log

# Queries > 100ms apparaissent immédiatement
# Avec EXPLAIN automatique !
```

### 5. Pas de thread pool (Simplicité)

```ini
# Production : thread_handling = pool-of-threads
# Dev : one-thread-per-connection (défaut)
```

**Raison :**
- Dev : < 10 connexions simultanées typique
- Thread pool overhead inutile
- Simplicité debug (1 thread = 1 query)

### 6. Sécurité relaxed (Localhost only)

```ini
bind-address = 127.0.0.1  # Localhost only
# Pas de TLS
# Authentication simple
```

**Justification :**
- Dev local : machine développeur trustée
- Pas d'exposition réseau externe
- Rapidité setup > Sécurité stricte

**⚠️ Staging/Pre-prod :**
```ini
# Utiliser config plus stricte :
bind-address = 0.0.0.0
require_secure_transport = ON  # TLS obligatoire
# Authentication moderne (ed25519)
```

---

## 🚀 Commandes utiles DEV

### Démarrage rapide

```bash
# Docker Compose : Stack complète
docker-compose up -d

# Logs temps réel
docker-compose logs -f mariadb

# Arrêt
docker-compose down

# Reset complet (données + volumes)
docker-compose down -v
docker-compose up -d

# Accès shell MariaDB
docker-compose exec mariadb mariadb -u developer -pdevpass myapp_dev
```

### Import/Export rapide

```bash
# Export structure + données
docker-compose exec mariadb mariadb-dump \
  -u developer -pdevpass \
  --single-transaction \
  myapp_dev > backup.sql

# Import
docker-compose exec -T mariadb mariadb \
  -u developer -pdevpass \
  myapp_dev < backup.sql

# Export seulement structure
docker-compose exec mariadb mariadb-dump \
  -u developer -pdevpass \
  --no-data \
  myapp_dev > schema.sql
```

### Reset base dev

```bash
# Script reset-db.sh
#!/bin/bash

echo "🗑️  Dropping database..."
docker-compose exec mariadb mariadb \
  -u root -pdevroot \
  -e "DROP DATABASE IF EXISTS myapp_dev; CREATE DATABASE myapp_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

echo "📝 Running migrations..."
docker-compose exec -T mariadb mariadb \
  -u root -pdevroot \
  myapp_dev < migrations/schema.sql

echo "🌱 Seeding data..."
docker-compose exec -T mariadb mariadb \
  -u root -pdevroot \
  myapp_dev < seeds/fixtures.sql

echo "✅ Database reset complete!"
```

---

## ⚠️ Points de vigilance DEV

### 1. Ne JAMAIS utiliser config dev en production

```bash
# ❌ DANGER
scp my-dev.cnf production-server:/etc/mysql/my.cnf
# → innodb_flush_log_at_trx_commit = 0
# → Perte données en production !

# ✅ Config séparées
my-dev.cnf       # Développement
my-staging.cnf   # Pre-production (config intermédiaire)
my-prod.cnf      # Production (D.1, D.2, D.3)
```

### 2. Volumes Docker : Persistance vs Volatilité

```yaml
# Approche 1 : Volume nommé (persistant)
volumes:
  - mariadb_data:/var/lib/mysql
# Données survivent docker-compose down
# Bon pour : Dev quotidien, conservation état

# Approche 2 : tmpfs (volatile, RAM)
volumes:
  - type: tmpfs
    target: /var/lib/mysql
# Données perdues à l'arrêt
# Bon pour : Tests CI/CD, environnement jetable
# Ultra rapide (RAM)
```

### 3. Charset cohérent dev/prod

```sql
-- ⚠️ PROBLÈME FRÉQUENT
-- Dev : utf8 (3 bytes)
-- Prod : utf8mb4 (4 bytes)
-- → Emojis 😀 fonctionnent dev, plantent prod

-- ✅ TOUJOURS utf8mb4 partout
CREATE DATABASE myapp_dev 
  CHARACTER SET utf8mb4 
  COLLATE utf8mb4_unicode_ci;
```

### 4. Taille dataset dev réaliste

```python
# ❌ Dataset dev trop petit
users_dev = 10  # 10 utilisateurs
# Queries rapides dev, lentes prod (1M users)

# ✅ Dataset représentatif
users_dev = 100_000  # 100K users
posts_dev = 1_000_000  # 1M posts
# Identifie problèmes perf dès dev
```

**Script seed volumétrie :**
```sql
-- Générer 100K users
DELIMITER $$
CREATE PROCEDURE generate_users(IN num_users INT)
BEGIN
    DECLARE i INT DEFAULT 0;
    WHILE i < num_users DO
        INSERT INTO users (email, username, created_at)
        VALUES (
            CONCAT('user', i, '@example.com'),
            CONCAT('user_', i),
            DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY)
        );
        SET i = i + 1;
    END WHILE;
END$$
DELIMITER ;

CALL generate_users(100000);
```

### 5. Tests isolation : Transactions rollback

```python
# Pattern test : Rollback automatique

import pytest
from myapp import db

@pytest.fixture(scope="function")
def db_session():
    """Fixture : Transaction rollback après chaque test"""
    connection = db.engine.connect()
    transaction = connection.begin()
    
    yield connection
    
    # Rollback : État initial restauré
    transaction.rollback()
    connection.close()

def test_create_user(db_session):
    # Créer user
    user = User(email='test@example.com')
    db_session.add(user)
    db_session.commit()
    
    # Assertions
    assert user.id is not None
    
    # Fin test : ROLLBACK automatique
    # User n'existe pas pour test suivant
```

### 6. Performance schema overhead

```sql
-- Performance Schema activé = overhead 5-10%
-- Acceptable dev, mais désactiver si :
-- - Tests benchmark performance
-- - CI/CD ultra rapide requis

-- Désactiver temporairement
SET GLOBAL performance_schema = OFF;
-- Redémarrage requis pour prise en compte
```

---

## ✅ Points clés à retenir

- 🚀 **Rapidité > Durabilité** : flush_log = 0, skip binlog
- 💾 **Ressources minimales** : 1GB buffer pool, 50 connexions max
- 🔍 **Logging verbose** : Slow queries 100ms, index missing loggés
- 🐳 **Docker recommended** : Reproductible, jetable, multi-versions
- 🧪 **CI/CD optimisé** : GitHub Actions, GitLab CI exemples
- 🛠️ **Debug tools** : EXPLAIN ANALYZE, profiling, Performance Schema
- ⚠️ **Jamais en prod** : Config dev = DANGER production
- 📊 **Dataset réaliste** : 100K lignes pour identifier problèmes perf
- 🔄 **Reset facile** : Scripts automatisés seeds/fixtures
- 🎯 **IDE integration** : VSCode, DataGrip, Adminer web

---

## 🔗 Ressources complémentaires

### Documentation MariaDB
- [Docker Official Images](https://hub.docker.com/_/mariadb)
- [Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [EXPLAIN](https://mariadb.com/kb/en/explain/)

### Outils Dev
- **Adminer** : https://www.adminer.org/
- **DBeaver** : https://dbeaver.io/
- **DataGrip** : https://www.jetbrains.com/datagrip/
- **MySQL Workbench** : https://www.mysql.com/products/workbench/

### Docker Compose
- [Compose File Reference](https://docs.docker.com/compose/compose-file/)
- [MariaDB Docker Docs](https://github.com/docker-library/docs/tree/master/mariadb)

### Autres annexes
- [D.1 - Configuration OLTP](./01-configuration-oltp.md)
- [D.2 - Configuration OLAP](./02-configuration-olap.md)
- [D.3 - Configuration Mixed](./03-configuration-mixed-workload.md)

---

**MariaDB** : Version 11.8 LTS

⏭️ [Checklist de Performance](/annexes/checklist-performance/README.md)

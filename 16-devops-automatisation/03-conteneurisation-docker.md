🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.3 Conteneurisation avec Docker

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : 
> - Compréhension des concepts Docker (images, conteneurs, volumes)
> - Connaissance de MariaDB (chapitres 1-11)
> - Expérience avec docker-compose
> - Notions de networking et stockage
> - Familiarité avec Linux et bash

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les spécificités de la conteneurisation pour bases de données stateful
- **Déployer** MariaDB dans des conteneurs Docker en production
- **Gérer** la persistance des données avec volumes et bind mounts
- **Configurer** MariaDB dans des conteneurs (variables d'environnement, fichiers de config)
- **Orchestrer** des stacks multi-conteneurs avec Docker Compose
- **Sécuriser** les conteneurs MariaDB (utilisateurs non-root, secrets, scanning)
- **Monitorer** les conteneurs MariaDB (logs, health checks, métriques)
- **Optimiser** les images Docker pour MariaDB (multi-stage builds, layers)

---

## Introduction

### Docker et les bases de données : Un défi unique

La conteneurisation des bases de données comme MariaDB présente des défis spécifiques par rapport aux applications stateless :

```
┌─────────────────────────────────────────────────────────────┐
│              Applications Stateless (Web, API)              │
├─────────────────────────────────────────────────────────────┤
│  ✅ Faciles à conteneuriser                                 │
│  ✅ Pas de données persistantes critiques                   │
│  ✅ Scale horizontal simple                                 │
│  ✅ Destruction/recréation sans risque                      │
│  ✅ Immutabilité totale                                     │
└─────────────────────────────────────────────────────────────┘

                           VS

┌─────────────────────────────────────────────────────────────┐
│            Bases de données (MariaDB, PostgreSQL)           │
├─────────────────────────────────────────────────────────────┤
│  ⚠️  Données critiques et précieuses                        │
│  ⚠️  État persistant obligatoire                            │
│  ⚠️  Performance I/O critique                               │
│  ⚠️  Backup et disaster recovery essentiels                 │
│  ⚠️  Configuration complexe (my.cnf, tuning)                │
│  ⚠️  Clustering et réplication délicats                     │
└─────────────────────────────────────────────────────────────┘
```

### Pourquoi conteneuriser MariaDB malgré tout ?

**Avantages majeurs** :

1. **Environnements reproductibles** :
   ```
   Dev → Staging → Production
   Même image, même comportement, zéro "ça marche sur ma machine"
   ```

2. **Isolation** :
   - Chaque base dans son propre conteneur
   - Versions différentes de MariaDB côte à côte (10.6, 11.4, 11.8)
   - Pas de conflits de dépendances

3. **Déploiements rapides** :
   ```bash
   # De 0 à MariaDB fonctionnel en secondes
   docker run -d mariadb:11.8
   ```

4. **CI/CD simplifié** :
   - Tests d'intégration avec vraie base
   - Environnements éphémères pour branches Git
   - Rollback instantané (retour à image précédente)

5. **Portabilité** :
   - Dev local, staging cloud, production on-premise
   - Même image partout

**Défis à surmonter** :

| Défi | Solution Docker |
|------|-----------------|
| Persistance données | **Volumes Docker** (gérés ou bind mounts) |
| Performance I/O | Volumes avec drivers optimisés, direct LVM |
| Configuration | Variables d'env + ConfigMaps + secrets |
| Backup | Scripts dans conteneurs, volumes snapshots |
| Monitoring | Logs structurés, health checks, exporters |
| Sécurité | Images scannées, non-root, secrets management |
| Networking | Docker networks, service discovery |

### Architecture Docker pour MariaDB

**Stack typique** :

```
┌──────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │   App 1    │  │   App 2    │  │   App 3    │              │
│  │ Container  │  │ Container  │  │ Container  │              │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘              │
│        │               │               │                     │
│        └───────────────┼───────────────┘                     │
│                        │                                     │
│                 ┌──────▼──────┐                              │
│                 │   Docker    │                              │
│                 │   Network   │                              │
│                 └──────┬──────┘                              │
│                        │                                     │
├────────────────────────┼─────────────────────────────────────┤
│                 Data Layer                                   │
│                        │                                     │
│       ┌────────────────▼─────────────────┐                   │
│       │      MariaDB Container           │                   │
│       │  ┌──────────────────────────┐    │                   │
│       │  │   MariaDB 11.8 Process   │    │                   │
│       │  └────────────┬─────────────┘    │                   │
│       │               │                  │                   │
│       │       ┌───────▼─────────┐        │                   │
│       │       │   /var/lib/     │        │                   │
│       │       │     mysql       │        │                   │
│       │       └───────┬─────────┘        │                   │
│       └───────────────┼──────────────────┘                   │
│                       │                                      │
│              ┌────────▼─────────┐                            │
│              │  Docker Volume   │                            │
│              │  (Persistent)    │                            │
│              └──────────────────┘                            │
│                       │                                      │
│              ┌────────▼─────────┐                            │
│              │   Host Storage   │                            │
│              │  (SSD, LVM, etc) │                            │
│              └──────────────────┘                            │
└──────────────────────────────────────────────────────────────┘
```

---

## Démarrage rapide : Premier conteneur MariaDB

### Commande basique

```bash
# La façon la PLUS SIMPLE (⚠️ NON recommandée pour production)
docker run --name my-mariadb -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mariadb:11.8

# Se connecter
docker exec -it my-mariadb mysql -p
```

**Problèmes de cette approche** :
- ❌ Données perdues si conteneur détruit
- ❌ Mot de passe en clair dans commande
- ❌ Pas de configuration personnalisée
- ❌ Pas de backup
- ❌ Port non exposé

### Approche production-ready

```bash
# 1. Créer un réseau dédié
docker network create mariadb-net

# 2. Créer un volume pour persistance
docker volume create mariadb-data

# 3. Créer un secret pour le mot de passe (Docker Swarm/Secrets)
echo "SuperSecurePassword123!" | docker secret create mariadb_root_password -

# 4. Lancer MariaDB avec configuration complète
docker run -d \
  --name mariadb-prod \
  --network mariadb-net \
  --restart unless-stopped \
  -v mariadb-data:/var/lib/mysql \
  -v $(pwd)/config/my.cnf:/etc/mysql/conf.d/custom.cnf:ro \
  -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mariadb_root_password \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=appuser \
  -e MYSQL_PASSWORD=apppassword \
  -p 3306:3306 \
  --health-cmd='mysqladmin ping --silent' \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=3 \
  mariadb:11.8 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci \
  --max-connections=500 \
  --innodb-buffer-pool-size=2G
```

**Explications** :

| Option | Rôle |
|--------|------|
| `--name` | Nom du conteneur (pour référence) |
| `--network` | Réseau Docker isolé |
| `--restart unless-stopped` | Redémarre automatiquement sauf arrêt manuel |
| `-v mariadb-data:/var/lib/mysql` | **Volume nommé** pour données |
| `-v .../my.cnf:...` | **Bind mount** pour config custom |
| `-e MYSQL_ROOT_PASSWORD_FILE` | Mot de passe depuis fichier (secret) |
| `-e MYSQL_DATABASE` | Créer base au démarrage |
| `-e MYSQL_USER` | Créer utilisateur applicatif |
| `-p 3306:3306` | Exposer port (0.0.0.0:3306) |
| `--health-cmd` | Commande de health check |
| Arguments finaux | Paramètres MariaDB (surcharge my.cnf) |

### Vérification

```bash
# Status du conteneur
docker ps

# Logs
docker logs mariadb-prod

# Logs en temps réel
docker logs -f mariadb-prod

# Health status
docker inspect --format='{{.State.Health.Status}}' mariadb-prod
# Output: healthy

# Stats ressources
docker stats mariadb-prod

# Se connecter
docker exec -it mariadb-prod mysql -uroot -p

# Depuis l'application
mysql -h localhost -P 3306 -uappuser -papppassword myapp
```

---

## Volumes et persistance des données

### Types de volumes Docker

**1. Volumes nommés (Docker-managed) - RECOMMANDÉ** :

```bash
# Créer volume
docker volume create mariadb-data

# Inspecter
docker volume inspect mariadb-data
```

```json
[
    {
        "CreatedAt": "2025-12-14T10:30:00Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/mariadb-data/_data",
        "Name": "mariadb-data",
        "Options": {},
        "Scope": "local"
    }
]
```

**Avantages** :
- ✅ Gérés par Docker (backups, migrations faciles)
- ✅ Indépendants du conteneur
- ✅ Portables entre hosts Docker
- ✅ Peuvent utiliser drivers spéciaux (NFS, cloud storage)

**Utilisation** :

```bash
docker run -d \
  -v mariadb-data:/var/lib/mysql \
  mariadb:11.8
```

**2. Bind mounts (Host paths)** :

```bash
# Créer répertoire sur host
mkdir -p /data/mariadb

# Changer propriétaire (UID 999 = mysql dans conteneur)
chown -R 999:999 /data/mariadb

# Lancer conteneur
docker run -d \
  -v /data/mariadb:/var/lib/mysql \
  mariadb:11.8
```

**Avantages** :
- ✅ Contrôle total sur l'emplacement
- ✅ Facile à backup avec outils système
- ✅ Performance potentiellement meilleure (accès direct)

**Inconvénients** :
- ❌ Path spécifique au host (moins portable)
- ❌ Permissions compliquées (UID/GID)
- ❌ Pas d'abstraction Docker

**3. tmpfs mounts (En mémoire - temporaire)** :

```bash
# Pour données non-critiques (tmp, logs temporaires)
docker run -d \
  -v mariadb-data:/var/lib/mysql \
  --tmpfs /tmp:rw,noexec,nosuid,size=1g \
  mariadb:11.8
```

**Usage** : `/tmp` en RAM pour meilleures performances sur données temporaires.

### Volume drivers avancés

**Local LVM (Direct access pour performance)** :

```bash
# Créer logical volume
lvcreate -L 100G -n mariadb-lv vg0

# Formater
mkfs.ext4 /dev/vg0/mariadb-lv

# Créer volume Docker pointant vers LV
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/dev/vg0/mariadb-lv \
  --opt o=bind \
  mariadb-lvm-data
```

**NFS (Pour partage entre hosts)** :

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/mariadb \
  mariadb-nfs-data
```

**Cloud storage (AWS EBS, Azure Disk)** :

```bash
# Avec plugin REX-Ray (AWS EBS)
docker volume create \
  --driver rexray/ebs \
  --opt size=100 \
  --opt volumetype=gp3 \
  mariadb-ebs-data
```

### Stratégies de backup des volumes

**1. Backup via docker volume** :

```bash
#!/bin/bash
# backup-mariadb-volume.sh

VOLUME_NAME="mariadb-data"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Créer conteneur temporaire pour backup
docker run --rm \
  -v ${VOLUME_NAME}:/data:ro \
  -v ${BACKUP_DIR}:/backup \
  alpine \
  tar czf /backup/mariadb-${DATE}.tar.gz -C /data .

echo "✅ Backup créé: ${BACKUP_DIR}/mariadb-${DATE}.tar.gz"
```

**2. Backup avec mysqldump** :

```bash
#!/bin/bash
# backup-mariadb-dump.sh

CONTAINER_NAME="mariadb-prod"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

docker exec ${CONTAINER_NAME} \
  mysqldump -uroot -p${MYSQL_ROOT_PASSWORD} \
  --all-databases \
  --single-transaction \
  --quick \
  --lock-tables=false \
  | gzip > ${BACKUP_DIR}/mariadb-dump-${DATE}.sql.gz

echo "✅ Dump créé: ${BACKUP_DIR}/mariadb-dump-${DATE}.sql.gz"
```

**3. Backup avec Mariabackup (HOT backup)** :

```bash
#!/bin/bash
# backup-mariadb-hot.sh

CONTAINER_NAME="mariadb-prod"
BACKUP_DIR="/backups/mariabackup"
DATE=$(date +%Y%m%d_%H%M%S)

# Créer répertoire de backup
mkdir -p ${BACKUP_DIR}/${DATE}

# Exécuter mariabackup dans le conteneur
docker exec ${CONTAINER_NAME} \
  mariabackup --backup \
  --target-dir=/tmp/backup \
  --user=root \
  --password=${MYSQL_ROOT_PASSWORD}

# Copier le backup hors du conteneur
docker cp ${CONTAINER_NAME}:/tmp/backup ${BACKUP_DIR}/${DATE}/

# Nettoyer dans le conteneur
docker exec ${CONTAINER_NAME} rm -rf /tmp/backup

echo "✅ Hot backup créé: ${BACKUP_DIR}/${DATE}/"
```

**4. Snapshot de volume (si supporté)** :

```bash
# Exemple avec LVM snapshot
lvcreate -L 10G -s -n mariadb-lv-snap /dev/vg0/mariadb-lv

# Monter le snapshot
mkdir -p /mnt/snapshot
mount /dev/vg0/mariadb-lv-snap /mnt/snapshot

# Backup
tar czf /backups/mariadb-snapshot.tar.gz -C /mnt/snapshot .

# Nettoyage
umount /mnt/snapshot
lvremove -y /dev/vg0/mariadb-lv-snap
```

---

## Configuration MariaDB dans Docker

### Variables d'environnement

**Variables supportées par l'image officielle** :

```bash
docker run -d \
  -e MYSQL_ROOT_PASSWORD=rootpassword \           # Obligatoire (ou _FILE ou _RANDOM)
  -e MYSQL_DATABASE=myapp \                       # Créer base au démarrage
  -e MYSQL_USER=appuser \                         # Créer utilisateur
  -e MYSQL_PASSWORD=apppassword \                 # Mot de passe utilisateur
  -e MYSQL_ROOT_HOST='%' \                        # Autoriser root depuis n'importe où
  -e MYSQL_INITDB_SKIP_TZINFO=yes \               # Skip timezone info
  -e MARIADB_AUTO_UPGRADE=1 \                     # 🆕 Auto-upgrade au démarrage
  mariadb:11.8
```

**Alternatives sécurisées pour mots de passe** :

```bash
# Option 1: Fichier secret (Docker Swarm)
echo "MySecretPassword" | docker secret create mysql_root_password -
docker service create \
  --secret mysql_root_password \
  -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_root_password \
  mariadb:11.8

# Option 2: Générer mot de passe aléatoire
docker run -d \
  -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
  mariadb:11.8

# Récupérer le mot de passe généré dans les logs
docker logs mariadb-container 2>&1 | grep "GENERATED ROOT PASSWORD"
# Output: GENERATED ROOT PASSWORD: <password-aléatoire>

# Option 3: Depuis fichier sur host
echo "MyPassword" > /secrets/mysql_root_password
docker run -d \
  -v /secrets:/secrets:ro \
  -e MYSQL_ROOT_PASSWORD_FILE=/secrets/mysql_root_password \
  mariadb:11.8
```

### Configuration via fichier my.cnf

**Structure recommandée** :

```
config/
├── my.cnf                    # Configuration principale
├── conf.d/
│   ├── 10-performance.cnf   # Tuning performance
│   ├── 20-charset.cnf       # Charset/collation
│   ├── 30-replication.cnf   # Réplication (si applicable)
│   └── 99-custom.cnf        # Overrides spécifiques
└── initdb.d/
    ├── 01-schema.sql        # Schéma initial
    ├── 02-data.sql          # Données de test
    └── 03-users.sql         # Utilisateurs supplémentaires
```

**Exemple de configuration** :

```ini
# config/my.cnf
[mysqld]
# Chemins
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock
pid-file = /var/run/mysqld/mysqld.pid

# Charset (🆕 MariaDB 11.8 default)
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Réseau
bind-address = 0.0.0.0
port = 3306
max_connections = 500
max_connect_errors = 1000000

# 🆕 MariaDB 11.8 - TLS par défaut
require_secure_transport = ON

# Logs
log_error = /var/log/mysql/error.log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = 1

# InnoDB
innodb_buffer_pool_size = 2G
innodb_buffer_pool_instances = 4
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 1
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1

# 🆕 MariaDB 11.8 - Optimisations SSD
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# 🆕 MariaDB 11.8 - Construction d'index efficace
innodb_alter_copy_bulk = ON

# Temporary tables
tmp_table_size = 256M
max_heap_table_size = 256M

# 🆕 MariaDB 11.8 - Contrôle espace temporaire
max_tmp_space_usage = 10G

# Query cache (deprecated but some still use)
query_cache_type = 0
query_cache_size = 0

# Thread pool
thread_handling = pool-of-threads
thread_pool_size = 16

[client]
port = 3306
socket = /var/run/mysqld/mysqld.sock
default-character-set = utf8mb4
```

**Monter la configuration** :

```bash
docker run -d \
  --name mariadb \
  -v $(pwd)/config/my.cnf:/etc/mysql/my.cnf:ro \
  -v $(pwd)/config/conf.d:/etc/mysql/conf.d:ro \
  -v mariadb-data:/var/lib/mysql \
  mariadb:11.8
```

### Scripts d'initialisation

**L'image officielle exécute automatiquement les scripts dans `/docker-entrypoint-initdb.d/`** :

```bash
# Structure
initdb.d/
├── 01-create-databases.sql
├── 02-create-tables.sql
├── 03-insert-data.sql
└── 04-setup-users.sh
```

**Exemple de scripts** :

```sql
-- initdb.d/01-create-databases.sql
CREATE DATABASE IF NOT EXISTS myapp;
CREATE DATABASE IF NOT EXISTS myapp_test;

GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'%';
GRANT ALL PRIVILEGES ON myapp_test.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

```sql
-- initdb.d/02-create-tables.sql
USE myapp;

CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

```bash
#!/bin/bash
# initdb.d/04-setup-users.sh

mysql -uroot -p${MYSQL_ROOT_PASSWORD} <<-EOSQL
    -- Créer utilisateur read-only
    CREATE USER 'readonly'@'%' IDENTIFIED BY 'readonlypass';
    GRANT SELECT ON myapp.* TO 'readonly'@'%';
    
    -- Créer utilisateur backup
    CREATE USER 'backup'@'localhost' IDENTIFIED BY 'backuppass';
    GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'backup'@'localhost';
    
    FLUSH PRIVILEGES;
EOSQL
```

**Monter les scripts** :

```bash
docker run -d \
  -v $(pwd)/initdb.d:/docker-entrypoint-initdb.d:ro \
  -v mariadb-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  mariadb:11.8
```

⚠️ **Important** : Les scripts ne s'exécutent que si le volume `/var/lib/mysql` est **vide** (première initialisation).

---

## Docker Compose pour MariaDB

### Stack simple : MariaDB + Application

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Base de données
  mariadb:
    image: mariadb:11.8
    container_name: mariadb-prod
    restart: unless-stopped
    
    # Environnement
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mariadb_root_password
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD_FILE: /run/secrets/mariadb_app_password
    
    # Secrets (Docker Swarm style)
    secrets:
      - mariadb_root_password
      - mariadb_app_password
    
    # Volumes
    volumes:
      - mariadb-data:/var/lib/mysql
      - ./config/my.cnf:/etc/mysql/my.cnf:ro
      - ./config/conf.d:/etc/mysql/conf.d:ro
      - ./initdb.d:/docker-entrypoint-initdb.d:ro
      - mariadb-logs:/var/log/mysql
    
    # Ports (exposition limitée)
    ports:
      - "127.0.0.1:3306:3306"  # Seulement localhost
    
    # Réseau
    networks:
      - backend
    
    # Health check
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "--silent"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    
    # Ressources (optionnel, recommandé en production)
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
  
  # Application exemple
  app:
    build: ./app
    container_name: myapp
    restart: unless-stopped
    
    environment:
      DB_HOST: mariadb
      DB_PORT: 3306
      DB_NAME: myapp
      DB_USER: appuser
      DB_PASSWORD_FILE: /run/secrets/mariadb_app_password
    
    secrets:
      - mariadb_app_password
    
    depends_on:
      mariadb:
        condition: service_healthy
    
    ports:
      - "8080:8080"
    
    networks:
      - backend

# Secrets
secrets:
  mariadb_root_password:
    file: ./secrets/mariadb_root_password.txt
  mariadb_app_password:
    file: ./secrets/mariadb_app_password.txt

# Volumes
volumes:
  mariadb-data:
    driver: local
  mariadb-logs:
    driver: local

# Réseaux
networks:
  backend:
    driver: bridge
```

**Structure des secrets** :

```bash
# secrets/mariadb_root_password.txt
SuperSecureRootPassword123!

# secrets/mariadb_app_password.txt
AppUserPassword456!
```

**Utilisation** :

```bash
# Démarrer la stack
docker-compose up -d

# Vérifier status
docker-compose ps

# Logs
docker-compose logs -f mariadb

# Arrêter
docker-compose down

# Arrêter ET supprimer volumes (⚠️ perte de données !)
docker-compose down -v
```

### Stack avancée : Réplication Primary-Replica

```yaml
# docker-compose-replication.yml
version: '3.8'

services:
  # Primary (Master)
  mariadb-primary:
    image: mariadb:11.8
    container_name: mariadb-primary
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    
    volumes:
      - mariadb-primary-data:/var/lib/mysql
      - ./config/primary/my.cnf:/etc/mysql/my.cnf:ro
    
    ports:
      - "3306:3306"
    
    networks:
      - mariadb-repl
    
    command: >
      --server-id=1
      --log-bin=mysql-bin
      --binlog-format=ROW
      --gtid-strict-mode=ON
      --log-slave-updates=ON
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "--silent"]
      interval: 10s
      timeout: 5s
      retries: 3
  
  # Replica 1
  mariadb-replica1:
    image: mariadb:11.8
    container_name: mariadb-replica1
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    
    volumes:
      - mariadb-replica1-data:/var/lib/mysql
      - ./config/replica/my.cnf:/etc/mysql/my.cnf:ro
      - ./scripts/setup-replication.sh:/docker-entrypoint-initdb.d/setup-replication.sh:ro
    
    ports:
      - "3307:3306"
    
    networks:
      - mariadb-repl
    
    depends_on:
      mariadb-primary:
        condition: service_healthy
    
    command: >
      --server-id=2
      --log-bin=mysql-bin
      --binlog-format=ROW
      --gtid-strict-mode=ON
      --read-only=ON
      --log-slave-updates=ON
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "--silent"]
      interval: 10s
      timeout: 5s
      retries: 3
  
  # Replica 2
  mariadb-replica2:
    image: mariadb:11.8
    container_name: mariadb-replica2
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    
    volumes:
      - mariadb-replica2-data:/var/lib/mysql
      - ./config/replica/my.cnf:/etc/mysql/my.cnf:ro
      - ./scripts/setup-replication.sh:/docker-entrypoint-initdb.d/setup-replication.sh:ro
    
    ports:
      - "3308:3306"
    
    networks:
      - mariadb-repl
    
    depends_on:
      mariadb-primary:
        condition: service_healthy
    
    command: >
      --server-id=3
      --log-bin=mysql-bin
      --binlog-format=ROW
      --gtid-strict-mode=ON
      --read-only=ON
      --log-slave-updates=ON

volumes:
  mariadb-primary-data:
  mariadb-replica1-data:
  mariadb-replica2-data:

networks:
  mariadb-repl:
    driver: bridge
```

**Script de setup réplication** :

```bash
#!/bin/bash
# scripts/setup-replication.sh

# Attendre que le primary soit prêt
until mysql -h mariadb-primary -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1" &> /dev/null
do
  echo "Waiting for primary..."
  sleep 5
done

# Configurer la réplication
mysql -uroot -p${MYSQL_ROOT_PASSWORD} <<-EOSQL
    STOP SLAVE;
    
    CHANGE MASTER TO
      MASTER_HOST='mariadb-primary',
      MASTER_USER='root',
      MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}',
      MASTER_USE_GTID=slave_pos;
    
    START SLAVE;
EOSQL

echo "✅ Replication configured"
```

### Stack complète : MariaDB + MaxScale + Monitoring

```yaml
# docker-compose-full-stack.yml
version: '3.8'

services:
  # MariaDB Primary
  mariadb-primary:
    image: mariadb:11.8
    # ... (comme ci-dessus)
  
  # MariaDB Replica
  mariadb-replica:
    image: mariadb:11.8
    # ... (comme ci-dessus)
  
  # MaxScale (Load Balancer)
  maxscale:
    image: mariadb/maxscale:latest
    container_name: maxscale
    restart: unless-stopped
    
    volumes:
      - ./maxscale/maxscale.cnf:/etc/maxscale.cnf:ro
    
    ports:
      - "4006:4006"  # Read-Write Split
      - "4008:4008"  # Read Connection Router
      - "8989:8989"  # MaxScale Admin
    
    networks:
      - mariadb-repl
    
    depends_on:
      - mariadb-primary
      - mariadb-replica
  
  # Prometheus Exporter pour MariaDB
  mysqld-exporter:
    image: prom/mysqld-exporter:latest
    container_name: mysqld-exporter
    restart: unless-stopped
    
    environment:
      DATA_SOURCE_NAME: "exporter:exporterpass@(mariadb-primary:3306)/"
    
    ports:
      - "9104:9104"
    
    networks:
      - mariadb-repl
      - monitoring
    
    depends_on:
      - mariadb-primary
  
  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    
    ports:
      - "9090:9090"
    
    networks:
      - monitoring
    
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
  
  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
    
    ports:
      - "3000:3000"
    
    networks:
      - monitoring
    
    depends_on:
      - prometheus

volumes:
  mariadb-primary-data:
  mariadb-replica-data:
  prometheus-data:
  grafana-data:

networks:
  mariadb-repl:
    driver: bridge
  monitoring:
    driver: bridge
```

---

## Sécurité des conteneurs MariaDB

### 1. Utilisateurs non-root

**L'image officielle MariaDB utilise l'utilisateur `mysql` (UID 999)** :

```dockerfile
# Vérifier l'utilisateur dans le conteneur
docker exec mariadb-prod whoami
# Output: mysql

docker exec mariadb-prod id
# Output: uid=999(mysql) gid=999(mysql) groups=999(mysql)
```

**Forcer l'utilisateur non-root** :

```bash
docker run -d \
  --user 999:999 \
  -v mariadb-data:/var/lib/mysql \
  mariadb:11.8
```

### 2. Read-only root filesystem

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run/mysqld \
  -v mariadb-data:/var/lib/mysql \
  mariadb:11.8
```

### 3. Capabilities (principe de moindre privilège)

```bash
docker run -d \
  --cap-drop=ALL \
  --cap-add=CHOWN \
  --cap-add=DAC_OVERRIDE \
  --cap-add=SETGID \
  --cap-add=SETUID \
  -v mariadb-data:/var/lib/mysql \
  mariadb:11.8
```

### 4. Scanning de vulnérabilités

**Avec Trivy** :

```bash
# Scanner l'image
trivy image mariadb:11.8

# Format JSON pour intégration CI/CD
trivy image --format json --output results.json mariadb:11.8

# Sortir en erreur si vulnérabilités critiques
trivy image --exit-code 1 --severity CRITICAL mariadb:11.8
```

**Avec Docker Scout** :

```bash
# Activer Docker Scout
docker scout quickview mariadb:11.8

# Analyse détaillée
docker scout cves mariadb:11.8

# Recommandations
docker scout recommendations mariadb:11.8
```

### 5. Secrets management

**Ne JAMAIS faire** :

```bash
# ❌ MAUVAIS - Secret en clair dans commande
docker run -d -e MYSQL_ROOT_PASSWORD=SuperSecret mariadb:11.8

# ❌ MAUVAIS - Secret en clair dans docker-compose.yml versionné
environment:
  MYSQL_ROOT_PASSWORD: SuperSecret
```

**Bonnes pratiques** :

```bash
# ✅ BON - Fichier secret non versionné
echo "SuperSecret" > /secrets/mysql_root_password
docker run -d \
  -v /secrets:/secrets:ro \
  -e MYSQL_ROOT_PASSWORD_FILE=/secrets/mysql_root_password \
  mariadb:11.8

# ✅ BON - Docker secret (Swarm)
echo "SuperSecret" | docker secret create mysql_root_password -

# ✅ BON - HashiCorp Vault
docker run -d \
  -e VAULT_ADDR=https://vault.example.com \
  -e VAULT_TOKEN=... \
  custom-mariadb-with-vault:11.8
```

### 6. Network isolation

```bash
# Créer réseau isolé
docker network create \
  --driver bridge \
  --subnet 172.25.0.0/16 \
  --ip-range 172.25.0.0/24 \
  --gateway 172.25.0.1 \
  mariadb-net

# Lancer MariaDB sur réseau isolé (pas d'accès internet)
docker run -d \
  --network mariadb-net \
  --network-alias mariadb \
  mariadb:11.8

# Applications se connectent via le réseau
docker run -d \
  --network mariadb-net \
  -e DB_HOST=mariadb \
  myapp:latest
```

---

## Performance et optimisation

### 1. Storage drivers

**Choisir le bon storage driver** :

```bash
# Vérifier le driver actuel
docker info | grep "Storage Driver"

# overlay2 est recommandé (moderne, performant)
# Pour production, considérer:
# - overlay2 avec XFS ou ext4
# - devicemapper en direct-lvm mode
# - ZFS pour snapshots avancés
```

**Configuration optimale pour MariaDB** :

```json
// /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

### 2. Volume drivers optimisés

```bash
# Local avec options de performance
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/dev/nvme0n1p1 \  # SSD NVMe direct
  --opt o=bind \
  mariadb-nvme-data
```

### 3. Resource limits

```yaml
# docker-compose.yml avec resource limits
services:
  mariadb:
    # ...
    deploy:
      resources:
        limits:
          cpus: '4.0'          # 4 CPU cores max
          memory: 8G           # 8GB RAM max
        reservations:
          cpus: '2.0'          # 2 cores garantis
          memory: 4G           # 4GB RAM garantis
    
    # Ou avec format ancien (docker-compose v2)
    cpus: 4.0
    mem_limit: 8g
    mem_reservation: 4g
```

### 4. Huge Pages (Linux)

**Pour InnoDB buffer pool > 4GB** :

```bash
# Configurer huge pages sur host
echo 1024 > /proc/sys/vm/nr_hugepages

# Lancer conteneur avec huge pages
docker run -d \
  --shm-size=2g \
  -v mariadb-data:/var/lib/mysql \
  mariadb:11.8 \
  --large-pages \
  --innodb-buffer-pool-size=8G
```

### 5. Benchmarking

```bash
# Sysbench dans conteneur
docker run --rm \
  --network container:mariadb-prod \
  severalnines/sysbench \
  sysbench /usr/share/sysbench/oltp_read_write.lua \
  --mysql-host=localhost \
  --mysql-user=root \
  --mysql-password=rootpass \
  --tables=10 \
  --table-size=10000 \
  prepare

docker run --rm \
  --network container:mariadb-prod \
  severalnines/sysbench \
  sysbench /usr/share/sysbench/oltp_read_write.lua \
  --mysql-host=localhost \
  --mysql-user=root \
  --mysql-password=rootpass \
  --tables=10 \
  --table-size=10000 \
  --threads=16 \
  --time=60 \
  run
```

---

## Logging et monitoring

### 1. Logs structurés

**Logs Docker** :

```bash
# Voir logs
docker logs mariadb-prod

# Logs temps réel
docker logs -f mariadb-prod

# Dernières 100 lignes
docker logs --tail 100 mariadb-prod

# Depuis timestamp
docker logs --since 2025-12-14T10:00:00 mariadb-prod
```

**Configuration logging driver** :

```yaml
# docker-compose.yml
services:
  mariadb:
    # ...
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "production,mariadb"
```

**Envoyer logs à service externe** :

```yaml
# Vers syslog
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.1.100:514"
    tag: "mariadb"

# Vers Fluentd
logging:
  driver: fluentd
  options:
    fluentd-address: "localhost:24224"
    tag: "docker.mariadb"
```

### 2. Health checks détaillés

```yaml
services:
  mariadb:
    # ...
    healthcheck:
      test: |
        bash -c '
          # Vérifier que MySQL répond
          mysqladmin ping -h localhost --silent || exit 1
          
          # Vérifier threads connectés
          threads=$(mysql -uroot -p$$MYSQL_ROOT_PASSWORD -e "SHOW STATUS LIKE \"Threads_connected\"" | grep Threads | awk "{print \$2}")
          if [ $$threads -gt 400 ]; then
            echo "Too many connections: $$threads"
            exit 1
          fi
          
          # Vérifier lag réplication (si replica)
          if [ -f /var/lib/mysql/replica.info ]; then
            lag=$(mysql -uroot -p$$MYSQL_ROOT_PASSWORD -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master | awk "{print \$2}")
            if [ "$$lag" != "0" ] && [ $$lag -gt 60 ]; then
              echo "Replication lag too high: $$lag seconds"
              exit 1
            fi
          fi
        '
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### 3. Métriques avec mysqld_exporter

**Voir section complète 16.9 pour Prometheus/Grafana détaillé.**

---

## ✅ Points clés à retenir

- **Volumes obligatoires** : Toujours utiliser des volumes Docker pour `/var/lib/mysql` (données persistantes)
- **Volumes nommés recommandés** : Préférer volumes Docker-managed plutôt que bind mounts
- **Configuration externalisée** : my.cnf via volumes, pas hardcodé dans image
- **Secrets sécurisés** : Utiliser `*_FILE` environment variables, jamais en clair
- **Health checks** : Toujours définir des health checks pour orchestration
- **Resource limits** : Définir CPU/RAM limits pour éviter contention
- **Logging structuré** : Configurer log drivers appropriés (json-file, syslog, fluentd)
- **Security scanning** : Scanner images régulièrement (Trivy, Docker Scout)
- **Non-root user** : Image officielle utilise déjà user mysql (UID 999)
- **Network isolation** : Utiliser Docker networks pour isoler MariaDB
- **Docker Compose** : Idéal pour dev/staging, production utiliser Kubernetes
- **Backups réguliers** : Automatiser backups des volumes et dumps SQL
- **Monitoring** : mysqld_exporter + Prometheus + Grafana

💡 **Best practice** : En production, préférer Kubernetes avec StatefulSets plutôt que Docker standalone.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 MariaDB Docker Official Image](https://hub.docker.com/_/mariadb)
- [📖 Docker Volumes](https://docs.docker.com/storage/volumes/)
- [📖 Docker Compose](https://docs.docker.com/compose/)
- [📖 Docker Security](https://docs.docker.com/engine/security/)

### Guides et best practices
- [📝 Docker for MariaDB Best Practices](https://mariadb.com/kb/en/installing-and-using-mariadb-via-docker/)
- [📝 Production-Ready Docker Images](https://www.docker.com/blog/intro-guide-to-dockerfile-best-practices/)
- [📝 Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver/)

### Outils
- [🔧 Trivy - Vulnerability Scanner](https://github.com/aquasecurity/trivy)
- [🔧 Docker Bench Security](https://github.com/docker/docker-bench-security)
- [🔧 Dive - Image Layer Explorer](https://github.com/wagoodman/dive)

---

## ➡️ Sections suivantes

**16.3.1 Images officielles MariaDB** : Nous explorerons en détail les différentes images officielles MariaDB (tags, versions, variantes), comment les personnaliser avec des Dockerfiles multi-stage, et comment créer vos propres images optimisées.

**16.3.2 Docker Compose pour développement** : Nous approfondirons l'utilisation de Docker Compose pour créer des environnements de développement complets avec MariaDB, incluant hot-reload, seed data, et intégration avec d'autres services.

**16.3.3 Volumes et persistance** : Nous détaillerons les stratégies avancées de gestion de volumes, backups automatisés, snapshots, et migration de données entre conteneurs.

---

**MariaDB** : Version 11.8 LTS

⏭️ [Images officielles MariaDB](/16-devops-automatisation/03.1-images-officielles.md)

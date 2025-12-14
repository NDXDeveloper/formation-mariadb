🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.7 Environnements de développement

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Notions de Docker et conteneurisation
> - Compréhension des variables d'environnement
> - Connaissance de Git
> - Bases des migrations (Section 17.5)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Configurer** un environnement de développement MariaDB avec Docker
- **Créer** des environnements reproductibles et isolés
- **Gérer** les différents environnements (dev, test, staging, production)
- **Implémenter** le seeding de données de développement
- **Automatiser** l'initialisation de la base de données
- **Synchroniser** les changements de schéma entre développeurs
- **Optimiser** le workflow de développement local

---

## Introduction

Un **environnement de développement** bien configuré est la base d'une productivité optimale. Il doit permettre aux développeurs de :

- ✅ **Démarrer rapidement** : Nouveau développeur opérationnel en minutes
- ✅ **Travailler en isolation** : Pas d'interférence entre projets
- ✅ **Reproduire la production** : Même version MariaDB, même config
- ✅ **Tester en local** : Données réalistes, migrations testées
- ✅ **Partager facilement** : Configuration versionnée dans Git

### 🎯 Les 12 facteurs (12-Factor App)

Principes pour applications modernes :

| Facteur | Application à MariaDB |
|---------|----------------------|
| **Codebase** | Configuration DB versionnée (docker-compose.yml) |
| **Dependencies** | Docker image fixe (mariadb:11.8) |
| **Config** | Variables d'environnement (.env) |
| **Backing services** | MariaDB comme service externe |
| **Dev/prod parity** | Même version MariaDB partout |

---

## Docker : La solution moderne

### 🐳 Pourquoi Docker pour MariaDB ?

**Avantages** :
- ✅ **Isolation** : Chaque projet = son propre conteneur
- ✅ **Reproductibilité** : Même environnement pour tous
- ✅ **Portabilité** : Fonctionne sur Linux, macOS, Windows
- ✅ **Versions multiples** : Plusieurs versions MariaDB en parallèle
- ✅ **Nettoyage facile** : `docker-compose down -v`

**vs Installation locale** :
- ❌ Conflits de versions
- ❌ Pollution du système
- ❌ Configuration spécifique à chaque développeur

### 📦 Installation Docker

```bash
# Linux (Ubuntu/Debian)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# macOS
brew install --cask docker

# Windows
# Télécharger Docker Desktop : https://www.docker.com/products/docker-desktop

# Vérifier installation
docker --version
docker-compose --version
```

---

## Configuration Docker Compose

### 🔧 Fichier de base

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  mariadb:
    image: mariadb:11.8
    container_name: myapp_mariadb
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-rootpassword}
      MYSQL_DATABASE: ${DB_NAME:-myapp_dev}
      MYSQL_USER: ${DB_USER:-myapp_user}
      MYSQL_PASSWORD: ${DB_PASSWORD:-myapp_password}
    
    ports:
      - "${DB_PORT:-3306}:3306"
    
    volumes:
      # Persistance des données
      - mariadb_data:/var/lib/mysql
      
      # Configuration personnalisée
      - ./docker/mariadb/conf.d:/etc/mysql/conf.d:ro
      
      # Scripts d'initialisation
      - ./docker/mariadb/init:/docker-entrypoint-initdb.d:ro
    
    networks:
      - app_network
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD:-rootpassword}"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

volumes:
  mariadb_data:
    driver: local

networks:
  app_network:
    driver: bridge
```

**Fichier .env** (à ne PAS commiter) :
```bash
# Database
DB_ROOT_PASSWORD=super_secret_root_password
DB_NAME=myapp_dev
DB_USER=myapp_user
DB_PASSWORD=myapp_dev_password
DB_PORT=3306

# Application
APP_ENV=development
DEBUG=true
```

**Fichier .env.example** (à commiter comme template) :
```bash
# Database
DB_ROOT_PASSWORD=change_me
DB_NAME=myapp_dev
DB_USER=myapp_user
DB_PASSWORD=change_me
DB_PORT=3306

# Application
APP_ENV=development
DEBUG=true
```

### 🚀 Démarrage

```bash
# Copier le template
cp .env.example .env
# Éditer .env avec vos valeurs

# Démarrer
docker-compose up -d

# Vérifier l'état
docker-compose ps

# Voir les logs
docker-compose logs -f mariadb

# Arrêter
docker-compose down

# Arrêter et supprimer les données
docker-compose down -v
```

---

## Configuration MariaDB personnalisée

### ⚙️ Fichier de configuration

**docker/mariadb/conf.d/custom.cnf** :
```ini
[mysqld]
# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# InnoDB settings (optimisé pour développement)
innodb_buffer_pool_size = 512M
innodb_log_file_size = 128M
innodb_flush_log_at_trx_commit = 2  # Performance (dev uniquement!)
innodb_flush_method = O_DIRECT

# Query cache (désactivé en dev pour voir vraies performances)
query_cache_type = 0
query_cache_size = 0

# Logs (utile pour debugging)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 1  # Log requêtes > 1s

# General log (ATTENTION : très verbeux)
general_log = 0
general_log_file = /var/log/mysql/general.log

# Timezone
default_time_zone = '+00:00'  # UTC

# Max connections (dev)
max_connections = 100

# SQL Mode strict (détecter les erreurs)
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```

💡 **Astuce** : Configuration dev != production (plus permissive, logs actifs).

---

## Scripts d'initialisation

### 🌱 Seeding automatique

**docker/mariadb/init/01-schema.sql** :
```sql
-- Ce script s'exécute automatiquement au premier démarrage

-- Créer les tables de base
CREATE TABLE IF NOT EXISTS users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS posts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Index
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);
```

**docker/mariadb/init/02-seed-data.sql** :
```sql
-- Données de développement (utilisateurs de test)
INSERT INTO users (name, email, password_hash) VALUES
    ('Alice Developer', 'alice@dev.local', '$2y$10$dummy_hash_for_dev_only'),
    ('Bob Tester', 'bob@dev.local', '$2y$10$dummy_hash_for_dev_only'),
    ('Charlie Admin', 'charlie@dev.local', '$2y$10$dummy_hash_for_dev_only');

-- Posts de test
INSERT INTO posts (user_id, title, content, published) VALUES
    (1, 'First Post', 'This is Alice''s first post', TRUE),
    (1, 'Draft Post', 'Work in progress...', FALSE),
    (2, 'Bob''s Tutorial', 'How to test applications', TRUE),
    (3, 'Admin Notice', 'Important announcement', TRUE);
```

**docker/mariadb/init/03-dev-user.sql** :
```sql
-- Créer un utilisateur avec privilèges complets (dev uniquement)
CREATE USER IF NOT EXISTS 'dev_admin'@'%' IDENTIFIED BY 'dev_admin_password';
GRANT ALL PRIVILEGES ON *.* TO 'dev_admin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

-- User read-only (pour tests de permissions)
CREATE USER IF NOT EXISTS 'readonly'@'%' IDENTIFIED BY 'readonly_password';
GRANT SELECT ON myapp_dev.* TO 'readonly'@'%';
FLUSH PRIVILEGES;
```

⚠️ **IMPORTANT** : Ces scripts ne s'exécutent qu'au **premier démarrage** (si volume vide).

### 🔄 Réinitialiser les données

```bash
# Supprimer le volume et redémarrer
docker-compose down -v
docker-compose up -d

# Ou via script
./scripts/reset-database.sh
```

**scripts/reset-database.sh** :
```bash
#!/bin/bash
set -e

echo "⚠️  This will DELETE all database data!"
read -p "Are you sure? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Aborted."
    exit 0
fi

echo "Stopping containers..."
docker-compose down

echo "Removing database volume..."
docker volume rm myapp_mariadb_data 2>/dev/null || true

echo "Starting fresh database..."
docker-compose up -d

echo "Waiting for MariaDB to be ready..."
until docker-compose exec -T mariadb mysqladmin ping -h localhost -u root -p${DB_ROOT_PASSWORD} --silent; do
    echo "  Waiting..."
    sleep 2
done

echo "✅ Database reset complete!"
```

---

## Multi-environnements

### 🎭 Environnements multiples

```
docker-compose.yml          # Development (défaut)
docker-compose.test.yml     # Tests (CI/CD)
docker-compose.staging.yml  # Staging (pré-production)
```

**docker-compose.test.yml** :
```yaml
version: '3.8'

services:
  mariadb-test:
    image: mariadb:11.8
    container_name: myapp_mariadb_test
    
    environment:
      MYSQL_ROOT_PASSWORD: test_root
      MYSQL_DATABASE: myapp_test
      MYSQL_USER: test_user
      MYSQL_PASSWORD: test_password
    
    ports:
      - "3307:3306"  # Port différent pour ne pas conflit
    
    # En RAM pour performance maximale
    tmpfs:
      - /var/lib/mysql
    
    volumes:
      - ./docker/mariadb/conf.d:/etc/mysql/conf.d:ro
      # Pas de seeding pour tests (données créées par tests)
```

**Utilisation** :
```bash
# Development
docker-compose up -d

# Tests
docker-compose -f docker-compose.test.yml up -d

# Les deux en parallèle (ports différents)
docker-compose up -d
docker-compose -f docker-compose.test.yml up -d
```

---

## Stack complète avec application

### 📦 Application multi-conteneurs

**docker-compose.yml complet** :
```yaml
version: '3.8'

services:
  # MariaDB
  mariadb:
    image: mariadb:11.8
    container_name: myapp_mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "${DB_PORT:-3306}:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./docker/mariadb/conf.d:/etc/mysql/conf.d:ro
      - ./docker/mariadb/init:/docker-entrypoint-initdb.d:ro
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3

  # Application backend (exemple Python/FastAPI)
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    container_name: myapp_api
    restart: unless-stopped
    environment:
      DB_HOST: mariadb
      DB_PORT: 3306
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      APP_ENV: development
    ports:
      - "8000:8000"
    volumes:
      # Hot reload : monter le code source
      - ./backend:/app
    depends_on:
      mariadb:
        condition: service_healthy
    networks:
      - app_network
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload

  # Frontend (exemple React)
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    container_name: myapp_frontend
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules  # Éviter de monter node_modules
    environment:
      REACT_APP_API_URL: http://localhost:8000
    networks:
      - app_network
    command: npm start

  # Adminer (GUI pour MariaDB)
  adminer:
    image: adminer:latest
    container_name: myapp_adminer
    restart: unless-stopped
    ports:
      - "8080:8080"
    networks:
      - app_network
    environment:
      ADMINER_DEFAULT_SERVER: mariadb

volumes:
  mariadb_data:

networks:
  app_network:
    driver: bridge
```

**Démarrage** :
```bash
docker-compose up -d

# Accès
# API: http://localhost:8000
# Frontend: http://localhost:3000
# Adminer: http://localhost:8080
# MariaDB: localhost:3306
```

---

## Makefile pour automatisation

### 🛠️ Commandes simplifiées

**Makefile** :
```makefile
.PHONY: help start stop restart logs reset test migrate seed shell

help: ## Afficher l'aide
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

start: ## Démarrer les conteneurs
	docker-compose up -d
	@echo "✅ Containers started"
	@echo "API: http://localhost:8000"
	@echo "Frontend: http://localhost:3000"
	@echo "Adminer: http://localhost:8080"

stop: ## Arrêter les conteneurs
	docker-compose down
	@echo "✅ Containers stopped"

restart: stop start ## Redémarrer les conteneurs

logs: ## Voir les logs
	docker-compose logs -f

logs-db: ## Voir les logs MariaDB
	docker-compose logs -f mariadb

reset: ## Réinitialiser la base de données
	@echo "⚠️  This will DELETE all database data!"
	@read -p "Continue? (y/n): " confirm && [ "$$confirm" = "y" ]
	docker-compose down -v
	docker-compose up -d
	@echo "✅ Database reset"

test: ## Lancer les tests
	docker-compose -f docker-compose.test.yml up -d
	docker-compose -f docker-compose.test.yml exec api pytest
	docker-compose -f docker-compose.test.yml down

migrate: ## Exécuter les migrations
	docker-compose exec api alembic upgrade head
	@echo "✅ Migrations applied"

seed: ## Charger les données de test
	docker-compose exec -T mariadb mysql -u root -p$$DB_ROOT_PASSWORD $$DB_NAME < ./seeds/dev-data.sql
	@echo "✅ Data seeded"

shell: ## Shell dans le conteneur MariaDB
	docker-compose exec mariadb bash

db-shell: ## Client MySQL
	docker-compose exec mariadb mysql -u root -p$$DB_ROOT_PASSWORD $$DB_NAME

backup: ## Backup de la base
	@mkdir -p backups
	docker-compose exec -T mariadb mysqldump -u root -p$$DB_ROOT_PASSWORD $$DB_NAME > backups/backup-$$(date +%Y%m%d-%H%M%S).sql
	@echo "✅ Backup created in backups/"

restore: ## Restaurer depuis backup (usage: make restore FILE=backup.sql)
	@test -n "$(FILE)" || (echo "❌ Usage: make restore FILE=backup.sql" && exit 1)
	docker-compose exec -T mariadb mysql -u root -p$$DB_ROOT_PASSWORD $$DB_NAME < $(FILE)
	@echo "✅ Database restored from $(FILE)"
```

**Utilisation** :
```bash
# Voir toutes les commandes
make help

# Démarrer l'environnement
make start

# Voir les logs
make logs

# Réinitialiser la DB
make reset

# Shell MariaDB
make db-shell

# Backup
make backup

# Restaurer
make restore FILE=backups/backup-20250115-120000.sql
```

---

## Seeding avancé

### 🌱 Seeders par langage

#### **Python (avec Faker)**

**seeds/seeder.py** :
```python
import os
import mysql.connector
from faker import Faker
from dotenv import load_dotenv

load_dotenv()

fake = Faker()

def get_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST', 'localhost'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        charset='utf8mb4'
    )

def seed_users(conn, count=100):
    """Créer N utilisateurs aléatoires"""
    cursor = conn.cursor()
    
    print(f"Seeding {count} users...")
    
    users = []
    for _ in range(count):
        users.append((
            fake.name(),
            fake.email(),
            fake.password(length=60)  # Simuler un hash bcrypt
        ))
    
    cursor.executemany(
        "INSERT INTO users (name, email, password_hash) VALUES (%s, %s, %s)",
        users
    )
    
    conn.commit()
    print(f"✅ Created {cursor.rowcount} users")

def seed_posts(conn, count=500):
    """Créer N posts aléatoires"""
    cursor = conn.cursor()
    
    # Récupérer les IDs utilisateurs
    cursor.execute("SELECT id FROM users")
    user_ids = [row[0] for row in cursor.fetchall()]
    
    if not user_ids:
        print("❌ No users found. Seed users first.")
        return
    
    print(f"Seeding {count} posts...")
    
    posts = []
    for _ in range(count):
        posts.append((
            fake.random_element(user_ids),
            fake.sentence(nb_words=6),
            fake.text(max_nb_chars=2000),
            fake.boolean(chance_of_getting_true=70)  # 70% publiés
        ))
    
    cursor.executemany(
        "INSERT INTO posts (user_id, title, content, published) VALUES (%s, %s, %s, %s)",
        posts
    )
    
    conn.commit()
    print(f"✅ Created {cursor.rowcount} posts")

def main():
    conn = get_connection()
    
    try:
        seed_users(conn, count=100)
        seed_posts(conn, count=500)
        
        print("\n✅ Seeding complete!")
    finally:
        conn.close()

if __name__ == '__main__':
    main()
```

**Exécution** :
```bash
pip install faker mysql-connector-python python-dotenv
python seeds/seeder.py
```

#### **Node.js (avec faker)**

**seeds/seeder.js** :
```javascript
const mysql = require('mysql2/promise');
const { faker } = require('@faker-js/faker');
require('dotenv').config();

async function seedUsers(connection, count = 100) {
    console.log(`Seeding ${count} users...`);
    
    const users = [];
    for (let i = 0; i < count; i++) {
        users.push([
            faker.person.fullName(),
            faker.internet.email(),
            faker.internet.password({ length: 60 })
        ]);
    }
    
    const [result] = await connection.query(
        'INSERT INTO users (name, email, password_hash) VALUES ?',
        [users]
    );
    
    console.log(`✅ Created ${result.affectedRows} users`);
}

async function seedPosts(connection, count = 500) {
    console.log(`Seeding ${count} posts...`);
    
    // Récupérer user IDs
    const [users] = await connection.query('SELECT id FROM users');
    if (users.length === 0) {
        console.log('❌ No users found. Seed users first.');
        return;
    }
    
    const userIds = users.map(u => u.id);
    
    const posts = [];
    for (let i = 0; i < count; i++) {
        posts.push([
            faker.helpers.arrayElement(userIds),
            faker.lorem.sentence(),
            faker.lorem.paragraphs(3),
            faker.datatype.boolean(0.7) // 70% publiés
        ]);
    }
    
    const [result] = await connection.query(
        'INSERT INTO posts (user_id, title, content, published) VALUES ?',
        [posts]
    );
    
    console.log(`✅ Created ${result.affectedRows} posts`);
}

async function main() {
    const connection = await mysql.createConnection({
        host: process.env.DB_HOST || 'localhost',
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME,
        charset: 'utf8mb4'
    });
    
    try {
        await seedUsers(connection, 100);
        await seedPosts(connection, 500);
        
        console.log('\n✅ Seeding complete!');
    } finally {
        await connection.end();
    }
}

main().catch(console.error);
```

**Exécution** :
```bash
npm install mysql2 @faker-js/faker dotenv
node seeds/seeder.js
```

---

## Hot reload et synchronisation

### 🔥 Développement avec hot reload

**Backend (FastAPI/Python)** :
```yaml
# Dans docker-compose.yml
api:
  volumes:
    - ./backend:/app  # Monter code source
  command: uvicorn main:app --host 0.0.0.0 --reload  # Auto-reload
```

**Frontend (React)** :
```yaml
frontend:
  volumes:
    - ./frontend:/app
    - /app/node_modules  # Exclure node_modules
  command: npm start  # React hot reload natif
```

### 🔄 Synchronisation des migrations

**Workflow équipe** :

1. **Développeur A** crée une migration
```bash
alembic revision -m "add user age column"
git add alembic/versions/abc123_add_user_age_column.py
git commit -m "Add user age migration"
git push
```

2. **Développeur B** récupère les changements
```bash
git pull
make migrate  # Applique nouvelles migrations
```

**Script d'update** (`scripts/update.sh`) :
```bash
#!/bin/bash
set -e

echo "🔄 Updating development environment..."

# Pull latest code
git pull

# Update dependencies
if [ -f "requirements.txt" ]; then
    pip install -r requirements.txt
fi

if [ -f "package.json" ]; then
    npm install
fi

# Run migrations
make migrate

echo "✅ Environment updated!"
```

---

## Outils de développement

### 🔧 GUI pour MariaDB

#### **1. Adminer (léger, dans Docker)**

```yaml
# Déjà dans docker-compose.yml
adminer:
  image: adminer:latest
  ports:
    - "8080:8080"
```

Accès : http://localhost:8080

#### **2. phpMyAdmin**

```yaml
phpmyadmin:
  image: phpmyadmin:latest
  container_name: myapp_phpmyadmin
  environment:
    PMA_HOST: mariadb
    PMA_PORT: 3306
    PMA_USER: root
    PMA_PASSWORD: ${DB_ROOT_PASSWORD}
  ports:
    - "8081:80"
  networks:
    - app_network
```

#### **3. DBeaver (desktop, recommandé)**

```bash
# Installation
# Linux
sudo snap install dbeaver-ce

# macOS
brew install --cask dbeaver-community

# Windows
# Télécharger : https://dbeaver.io/download/
```

**Configuration DBeaver** :
- Host: localhost
- Port: 3306
- Database: myapp_dev
- User: myapp_user
- Password: myapp_dev_password

#### **4. HeidiSQL (Windows)**

Télécharger : https://www.heidisql.com/

### 📊 Monitoring en développement

**Prometheus + Grafana** (optionnel) :

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  mysqld-exporter:
    image: prom/mysqld-exporter
    container_name: mysqld_exporter
    environment:
      DATA_SOURCE_NAME: "root:${DB_ROOT_PASSWORD}@(mariadb:3306)/"
    ports:
      - "9104:9104"
    networks:
      - app_network
  
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./docker/prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - app_network
  
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app_network

volumes:
  prometheus_data:
  grafana_data:

networks:
  app_network:
    external: true
```

---

## Bonnes pratiques

### ✅ Checklist environnement de développement

#### **Configuration**
- [ ] Docker Compose configuré et versionné
- [ ] Fichier .env.example fourni (template)
- [ ] .env dans .gitignore
- [ ] Documentation de setup dans README.md
- [ ] Healthchecks configurés

#### **Données**
- [ ] Scripts d'initialisation (schema + seed)
- [ ] Seeders avec données réalistes
- [ ] Script de reset de la base
- [ ] Backup/restore automatisé

#### **Workflow**
- [ ] Makefile ou scripts npm pour commandes courantes
- [ ] Hot reload configuré (backend + frontend)
- [ ] Migrations automatiques au démarrage
- [ ] Tests lancés facilement

#### **Documentation**
- [ ] README.md avec instructions de setup
- [ ] Variables d'environnement documentées
- [ ] Commandes courantes listées
- [ ] Troubleshooting section

### 📝 README.md template

```markdown
# MyApp - Development Environment

## Prerequisites

- Docker 20.10+
- Docker Compose 2.0+
- Git

## Quick Start

1. **Clone repository**
   ```bash
   git clone https://github.com/company/myapp.git
   cd myapp
   ```

2. **Configure environment**
   ```bash
   cp .env.example .env
   # Edit .env with your values
   ```

3. **Start services**
   ```bash
   make start
   # Or: docker-compose up -d
   ```

4. **Access application**
   - API: http://localhost:8000
   - Frontend: http://localhost:3000
   - Database GUI: http://localhost:8080

## Common Commands

```bash
make start      # Start all services
make stop       # Stop all services
make logs       # View logs
make reset      # Reset database
make test       # Run tests
make migrate    # Run migrations
make seed       # Load dev data
```

## Database Access

**CLI:**
```bash
make db-shell
```

**GUI:** http://localhost:8080
- Server: mariadb
- Username: myapp_user
- Password: (from .env)

## Troubleshooting

**Port already in use:**
```bash
# Change DB_PORT in .env
DB_PORT=3307
```

**Database not ready:**
```bash
docker-compose logs mariadb
```

**Reset everything:**
```bash
docker-compose down -v
make start
```
```

---

## Différences dev/staging/production

### 🎭 Configuration par environnement

| Paramètre | Dev | Staging | Production |
|-----------|-----|---------|------------|
| **Logging** | Verbeux | Modéré | Essentiel uniquement |
| **Debug mode** | ON | OFF | OFF |
| **SSL/TLS** | Optionnel | Requis | Requis |
| **Performance tuning** | Minimal | Optimisé | Maximum |
| **Data seeding** | Oui | Données test | Non |
| **Backup** | Non requis | Quotidien | Temps réel |

**Exemple de configuration conditionnelle** :

```python
# config.py
import os

class Config:
    ENV = os.getenv('APP_ENV', 'development')
    
    # Database
    DB_HOST = os.getenv('DB_HOST', 'localhost')
    DB_PORT = int(os.getenv('DB_PORT', '3306'))
    DB_NAME = os.getenv('DB_NAME')
    DB_USER = os.getenv('DB_USER')
    DB_PASSWORD = os.getenv('DB_PASSWORD')
    
    # Pooling (différent par env)
    if ENV == 'production':
        DB_POOL_SIZE = 20
        DB_POOL_MAX_OVERFLOW = 40
    elif ENV == 'staging':
        DB_POOL_SIZE = 10
        DB_POOL_MAX_OVERFLOW = 20
    else:  # development
        DB_POOL_SIZE = 5
        DB_POOL_MAX_OVERFLOW = 10
    
    # Logging
    LOG_LEVEL = 'DEBUG' if ENV == 'development' else 'INFO'
    
    # SSL
    DB_SSL = ENV in ['staging', 'production']
```

---

## Troubleshooting

### 🔍 Problèmes courants

#### **1. Port 3306 déjà utilisé**

**Symptôme** :
```
Error: port is already allocated
```

**Solution** :
```bash
# Option 1 : Changer le port
# .env
DB_PORT=3307

# Option 2 : Arrêter MariaDB local
sudo systemctl stop mariadb

# Option 3 : Identifier et arrêter le processus
lsof -i :3306
kill <PID>
```

#### **2. Base de données ne démarre pas**

**Diagnostic** :
```bash
docker-compose logs mariadb
```

**Solutions courantes** :
```bash
# Permissions volume
sudo chown -R 999:999 /path/to/volume

# Corruption du volume : reset
docker-compose down -v
docker-compose up -d

# Vérifier ressources Docker
docker stats
```

#### **3. Connexion refusée depuis l'application**

**Vérifications** :
```bash
# 1. MariaDB est démarré ?
docker-compose ps

# 2. Réseau correct ?
docker network ls
docker network inspect myapp_app_network

# 3. Healthcheck OK ?
docker-compose ps
# STATUS doit être "healthy"

# 4. Variables d'env correctes ?
docker-compose exec api env | grep DB_
```

#### **4. Migrations ne s'appliquent pas**

```bash
# Vérifier connexion
docker-compose exec api python -c "import mysql.connector; mysql.connector.connect(host='mariadb', user='myapp_user', password='myapp_password')"

# Logs de migration
docker-compose exec api alembic current
docker-compose exec api alembic history

# Forcer upgrade
docker-compose exec api alembic upgrade head
```

---

## ✅ Points clés à retenir

- 🐳 **Docker** : Solution standard pour environnements reproductibles
- 📦 **Docker Compose** : Orchestration multi-conteneurs (DB + app + outils)
- ⚙️ **Configuration** : Personnalisée par environnement, versionnée
- 🌱 **Seeding** : Scripts d'initialisation automatiques, seeders avec Faker
- 🔥 **Hot reload** : Volumes montés pour développement rapide
- 🔄 **Synchronisation** : Migrations versionnées, script d'update
- 🛠️ **Outils** : Adminer, phpMyAdmin, DBeaver, HeidiSQL
- 📝 **Documentation** : README.md complet, .env.example
- 🎭 **Multi-env** : dev, test, staging avec configs différentes
- ✅ **Automatisation** : Makefile, scripts bash, commandes simplifiées

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [Docker Documentation](https://docs.docker.com/)
- 📖 [Docker Compose](https://docs.docker.com/compose/)
- 📖 [MariaDB Docker Image](https://hub.docker.com/_/mariadb)

### **Outils**
- 🛠️ [Adminer](https://www.adminer.org/)
- 🛠️ [phpMyAdmin](https://www.phpmyadmin.net/)
- 🛠️ [DBeaver](https://dbeaver.io/)
- 🛠️ [HeidiSQL](https://www.heidisql.com/)

### **Seeders et Faker**
- 📖 [Faker (Python)](https://faker.readthedocs.io/)
- 📖 [Faker.js](https://fakerjs.dev/)
- 📖 [Java Faker](https://github.com/DiUS/java-faker)

### **12-Factor App**
- 📝 [The Twelve-Factor App](https://12factor.net/)

---

## ➡️ Sections suivantes

- **17.8** - Prévention des injections SQL : Comprendre, détecter, corriger
- **17.9** - Prepared Statements : Fonctionnement, sécurité, performances

---

**MariaDB** : Version 11.8 LTS

⏭️ [Prévention des injections SQL](/17-integration-developpement/08-prevention-injections-sql.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.5 Gestion des migrations de schéma

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Compréhension du SQL et DDL (CREATE, ALTER, DROP)
> - Connaissance des systèmes de version (Git)
> - Notions de CI/CD (recommandé)
> - Expérience avec les ORM (Section 17.3)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** pourquoi les migrations de schéma sont essentielles
- **Versionner** votre schéma de base de données comme votre code
- **Utiliser** les principaux outils de migration (Flyway, Liquibase, migrations ORM)
- **Créer** des migrations robustes et réversibles
- **Déployer** des changements de schéma en production sans downtime
- **Gérer** les migrations dans un workflow CI/CD
- **Résoudre** les conflits de migrations en équipe
- **Implémenter** des stratégies de rollback efficaces

---

## Introduction

Les **migrations de schéma** (ou migrations de base de données) sont des scripts versionnés qui modifient progressivement la structure de votre base de données. Elles permettent de faire évoluer votre schéma de manière **contrôlée, reproductible et auditable**.

### 🔄 Le problème sans migrations

**Scénario catastrophe** :

```
Développeur A : "J'ai ajouté une colonne 'age' à la table users"
Développeur B : "Bizarre, ça ne marche pas chez moi..."
Production : "ERROR: Unknown column 'age' in 'field list'"
DBA : "Quelle version du schéma est en production ??"
```

Sans migrations :
- ❌ Schéma différent entre environnements (dev, staging, prod)
- ❌ Pas d'historique des changements
- ❌ Déploiements manuels = erreurs
- ❌ Impossible de reproduire un environnement
- ❌ Rollback complexe voire impossible

### ✅ Avec migrations

```
Migration v001 : CREATE TABLE users
Migration v002 : ADD COLUMN age INT
Migration v003 : CREATE INDEX idx_users_email
...
```

Avantages :
- ✅ **Versioning** : Historique complet des changements
- ✅ **Reproductibilité** : Même schéma partout (dev, CI, prod)
- ✅ **Automatisation** : Déploiement automatique via CI/CD
- ✅ **Rollback** : Retour arrière possible
- ✅ **Collaboration** : Plusieurs développeurs sans conflit
- ✅ **Audit** : Qui a changé quoi et quand

---

## Principes fondamentaux

### 📋 Règles d'or des migrations

#### **1. Une migration = un fichier versionné**

```
migrations/
├── V001__create_users_table.sql
├── V002__add_email_index.sql
├── V003__add_posts_table.sql
└── V004__add_user_age_column.sql
```

Chaque fichier :
- **Numéro de version** : V001, V002, V003... (ordre d'exécution)
- **Description** : Nom explicite du changement
- **Extension** : .sql, .js, .py selon l'outil

#### **2. Les migrations sont immuables**

⚠️ **JAMAIS modifier une migration déjà appliquée** !

```bash
# ❌ MAUVAIS
# Modifier V001__create_users.sql après application

# ✅ BON
# Créer V005__modify_users_table.sql
```

**Pourquoi ?**
- Production a déjà V001 appliqué
- Modifier V001 → checksum différent → erreur
- Créer une nouvelle migration = historique préservé

#### **3. Les migrations sont séquentielles**

```
V001 → V002 → V003 → V004
  ✓      ✓      ✓      ✓
```

Chaque migration s'applique **une seule fois** dans l'ordre.

#### **4. Les migrations doivent être idempotentes (si possible)**

```sql
-- ❌ MAUVAIS : Échoue si la table existe déjà
CREATE TABLE users (id INT PRIMARY KEY);

-- ✅ BON : Idempotent
CREATE TABLE IF NOT EXISTS users (id INT PRIMARY KEY);
```

💡 Note : Certains outils (Flyway, Liquibase) gèrent automatiquement l'idempotence.

#### **5. Toujours fournir un rollback**

```sql
-- V002__add_email_column.sql (UP)
ALTER TABLE users ADD COLUMN email VARCHAR(255);

-- V002__add_email_column.rollback.sql (DOWN)
ALTER TABLE users DROP COLUMN email;
```

---

## Outils de migration

### 🛠️ Panorama des outils

| Outil | Langage | Type | Avantages | Inconvénients |
|-------|---------|------|-----------|---------------|
| **Flyway** | Java | SQL/Java | Simple, robuste, entreprise | Licence payante (Teams+) |
| **Liquibase** | Java | XML/JSON/SQL | Très flexible, multi-DB | Courbe apprentissage |
| **Alembic** | Python | Python | Intégration SQLAlchemy | Python uniquement |
| **EF Core Migrations** | C# | C# | Intégration .NET/EF Core | .NET uniquement |
| **Sequelize Migrations** | JavaScript | JavaScript | Intégration Sequelize | Node.js uniquement |
| **Prisma Migrate** | TypeScript | Prisma Schema | Type-safe, simple | Prisma uniquement |
| **golang-migrate** | Go | SQL | Léger, CLI puissant | Pas d'ORM intégré |
| **django.db.migrations** | Python | Python | Intégré Django | Django uniquement |

### 🎯 Choix de l'outil

**Question 1 : Utilisez-vous un ORM ?**
- ✅ Oui → Migrations ORM (Alembic, EF Core, Sequelize, Prisma)
- ❌ Non → Flyway ou Liquibase

**Question 2 : Avez-vous besoin de multi-DB ?**
- ✅ Oui (MySQL, PostgreSQL, Oracle...) → Liquibase
- ❌ Non (MariaDB uniquement) → Flyway ou migrations ORM

**Question 3 : Quel est votre budget ?**
- Gratuit → Flyway Community, Liquibase Community, migrations ORM
- Entreprise → Flyway Teams/Enterprise, Liquibase Pro

---

## Flyway : Le standard Java

### 📦 Installation et configuration

**Maven** :
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>10.4.1</version>
</dependency>

<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
    <version>10.4.1</version>
</dependency>
```

**Configuration** (`application.properties`) :
```properties
# Database
spring.datasource.url=jdbc:mariadb://localhost:3306/myapp
spring.datasource.username=flyway_user
spring.datasource.password=secret

# Flyway
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
spring.flyway.baseline-version=0
spring.flyway.validate-on-migrate=true
```

### 📝 Structure des migrations

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__add_email_index.sql
├── V3__create_posts_table.sql
└── V4__add_user_status.sql
```

**Convention de nommage** :
```
V<VERSION>__<DESCRIPTION>.sql

V : Prefix obligatoire (Versioned migration)
VERSION : Numéro de version (1, 2, 3... ou 1.0, 1.1, 2.0)
__ : Deux underscores (séparateur)
DESCRIPTION : Description en snake_case
.sql : Extension
```

### 🔧 Exemples de migrations Flyway

**V1__create_users_table.sql** :
```sql
-- Création de la table users
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Index pour recherche par email
CREATE INDEX idx_users_email ON users(email);
```

**V2__create_posts_table.sql** :
```sql
-- Table des posts
CREATE TABLE posts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Clé étrangère
    CONSTRAINT fk_posts_user_id 
        FOREIGN KEY (user_id) 
        REFERENCES users(id) 
        ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Index pour requêtes fréquentes
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);
```

**V3__add_user_age_column.sql** :
```sql
-- Ajout de la colonne age
ALTER TABLE users 
ADD COLUMN age INT UNSIGNED NULL AFTER email;

-- Contrainte de validation
ALTER TABLE users 
ADD CONSTRAINT chk_users_age 
CHECK (age IS NULL OR (age >= 0 AND age <= 150));
```

**V4__create_tags_and_post_tags.sql** :
```sql
-- Table des tags
CREATE TABLE tags (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Table de liaison many-to-many
CREATE TABLE post_tags (
    post_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (post_id, tag_id),
    
    CONSTRAINT fk_post_tags_post_id 
        FOREIGN KEY (post_id) 
        REFERENCES posts(id) 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_post_tags_tag_id 
        FOREIGN KEY (tag_id) 
        REFERENCES tags(id) 
        ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 🚀 Exécution des migrations

**Via Spring Boot** :
```java
// Application.java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        // Migrations exécutées automatiquement au démarrage
    }
}
```

**Via CLI** :
```bash
# Migrer vers la dernière version
flyway migrate

# Informations sur l'état
flyway info

# Valider les migrations
flyway validate

# Nettoyer la base (DEV UNIQUEMENT !)
flyway clean

# Réparer l'historique (cas spéciaux)
flyway repair
```

**Via Java API** :
```java
import org.flywaydb.core.Flyway;

public class MigrationRunner {
    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
            .dataSource("jdbc:mariadb://localhost:3306/myapp", "user", "password")
            .locations("classpath:db/migration")
            .baselineOnMigrate(true)
            .load();
        
        // Exécuter les migrations
        flyway.migrate();
        
        // Afficher l'état
        System.out.println("Applied migrations: " + flyway.info().all().length);
    }
}
```

### 📊 Table d'historique Flyway

Flyway crée automatiquement `flyway_schema_history` :

```sql
SELECT * FROM flyway_schema_history;
```

| installed_rank | version | description | type | script | checksum | installed_by | installed_on | execution_time | success |
|----------------|---------|-------------|------|--------|----------|--------------|--------------|----------------|---------|
| 1 | 1 | create users table | SQL | V1__create_users_table.sql | 123456789 | flyway_user | 2025-01-15 10:00:00 | 45 | 1 |
| 2 | 2 | create posts table | SQL | V2__create_posts_table.sql | 987654321 | flyway_user | 2025-01-15 10:00:01 | 38 | 1 |

⚠️ **Ne JAMAIS modifier cette table manuellement** (sauf cas très spéciaux avec `repair`).

---

## Liquibase : Flexibilité maximale

### 📦 Installation et configuration

**Maven** :
```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
    <version>4.25.0</version>
</dependency>
```

**Configuration** (`liquibase.properties`) :
```properties
changeLogFile=db/changelog/db.changelog-master.xml
url=jdbc:mariadb://localhost:3306/myapp
username=liquibase_user
password=secret
driver=org.mariadb.jdbc.Driver
```

### 📝 Structure des changelogs

**Master changelog** (`db.changelog-master.xml`) :
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.25.xsd">

    <include file="db/changelog/changes/v1.0-create-users-table.xml"/>
    <include file="db/changelog/changes/v1.1-create-posts-table.xml"/>
    <include file="db/changelog/changes/v1.2-add-user-age.xml"/>
</databaseChangeLog>
```

### 🔧 Exemples de changelogs Liquibase

**Format XML** (`v1.0-create-users-table.xml`) :
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.25.xsd">

    <changeSet id="1" author="john.doe">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="name" type="VARCHAR(100)">
                <constraints nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>
        
        <createIndex indexName="idx_users_email" tableName="users">
            <column name="email"/>
        </createIndex>
        
        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

**Format SQL** (`v1.1-create-posts-table.sql`) :
```sql
--liquibase formatted sql

--changeset john.doe:2
CREATE TABLE posts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT fk_posts_user_id FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

--rollback DROP TABLE posts;
```

**Format JSON** (`v1.2-add-user-age.json`) :
```json
{
  "databaseChangeLog": [
    {
      "changeSet": {
        "id": "3",
        "author": "jane.smith",
        "changes": [
          {
            "addColumn": {
              "tableName": "users",
              "columns": [
                {
                  "column": {
                    "name": "age",
                    "type": "INT",
                    "constraints": {
                      "nullable": true
                    }
                  }
                }
              ]
            }
          }
        ],
        "rollback": [
          {
            "dropColumn": {
              "tableName": "users",
              "columnName": "age"
            }
          }
        ]
      }
    }
  ]
}
```

### 🚀 Exécution Liquibase

```bash
# Appliquer les migrations
liquibase update

# Rollback N changements
liquibase rollback-count 2

# Rollback jusqu'à un tag
liquibase rollback v1.0

# Générer SQL sans exécuter
liquibase update-sql

# État actuel
liquibase status

# Valider
liquibase validate
```

---

## Migrations ORM

### 🐍 Alembic (Python / SQLAlchemy)

**Installation** :
```bash
pip install alembic
```

**Initialisation** :
```bash
alembic init alembic
```

**Configuration** (`alembic.ini`) :
```ini
[alembic]
script_location = alembic
sqlalchemy.url = mysql+mysqlconnector://user:password@localhost/myapp
```

**Générer une migration** :
```bash
# Migration vide
alembic revision -m "create users table"

# Auto-génération (compare modèles SQLAlchemy avec DB)
alembic revision --autogenerate -m "add user age column"
```

**Migration générée** (`alembic/versions/abc123_create_users_table.py`) :
```python
"""create users table

Revision ID: abc123
Revises: 
Create Date: 2025-01-15 10:00:00
"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = 'abc123'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    """Upgrade schema"""
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(length=100), nullable=False),
        sa.Column('email', sa.String(length=255), nullable=False),
        sa.Column('created_at', sa.DateTime(), server_default=sa.text('CURRENT_TIMESTAMP')),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email')
    )
    
    op.create_index('idx_users_email', 'users', ['email'])

def downgrade():
    """Downgrade schema"""
    op.drop_index('idx_users_email', table_name='users')
    op.drop_table('users')
```

**Exécution** :
```bash
# Migrer vers la dernière version
alembic upgrade head

# Migrer vers une version spécifique
alembic upgrade abc123

# Rollback d'une migration
alembic downgrade -1

# Rollback vers une version
alembic downgrade abc123

# État actuel
alembic current

# Historique
alembic history
```

### 🔷 Entity Framework Core (.NET)

**Installation** :
```bash
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet tool install --global dotnet-ef
```

**Créer une migration** :
```bash
dotnet ef migrations add CreateUsersTable
```

**Migration générée** :
```csharp
public partial class CreateUsersTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Users",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("MySql:ValueGenerationStrategy", 
                        MySqlValueGenerationStrategy.IdentityColumn),
                Name = table.Column<string>(maxLength: 100, nullable: false),
                Email = table.Column<string>(maxLength: 255, nullable: false),
                CreatedAt = table.Column<DateTime>(nullable: false)
                    .Annotation("MySql:ValueGenerationStrategy", 
                        MySqlValueGenerationStrategy.ComputedColumn)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Users", x => x.Id);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Users_Email",
            table: "Users",
            column: "Email",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Users");
    }
}
```

**Exécution** :
```bash
# Appliquer les migrations
dotnet ef database update

# Rollback vers une migration
dotnet ef database update CreateUsersTable

# Rollback complet
dotnet ef database update 0

# Générer SQL
dotnet ef migrations script

# Lister les migrations
dotnet ef migrations list
```

### 🟢 Sequelize (Node.js)

**Installation** :
```bash
npm install --save-dev sequelize-cli
npx sequelize-cli init
```

**Créer une migration** :
```bash
npx sequelize-cli migration:generate --name create-users-table
```

**Migration générée** (`migrations/20250115100000-create-users-table.js`) :
```javascript
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('Users', {
      id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true,
        allowNull: false
      },
      name: {
        type: Sequelize.STRING(100),
        allowNull: false
      },
      email: {
        type: Sequelize.STRING(255),
        allowNull: false,
        unique: true
      },
      createdAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP'),
        allowNull: false
      },
      updatedAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP'),
        allowNull: false
      }
    });

    await queryInterface.addIndex('Users', ['email'], {
      name: 'idx_users_email'
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable('Users');
  }
};
```

**Exécution** :
```bash
# Appliquer toutes les migrations
npx sequelize-cli db:migrate

# Rollback dernière migration
npx sequelize-cli db:migrate:undo

# Rollback toutes
npx sequelize-cli db:migrate:undo:all

# État
npx sequelize-cli db:migrate:status
```

### 🔷 Prisma Migrate (TypeScript)

**Schema Prisma** (`prisma/schema.prisma`) :
```prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int      @id @default(autoincrement())
  name      String   @db.VarChar(100)
  email     String   @unique @db.VarChar(255)
  age       Int?
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  
  posts Post[]
  
  @@index([email])
  @@map("users")
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String   @db.VarChar(255)
  content   String?  @db.Text
  published Boolean  @default(false)
  userId    Int      @map("user_id")
  createdAt DateTime @default(now()) @map("created_at")
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
  @@index([published])
  @@map("posts")
}
```

**Créer une migration** :
```bash
# Développement (crée migration + applique)
npx prisma migrate dev --name create-users-table

# Production (applique migrations existantes)
npx prisma migrate deploy

# Générer SQL sans appliquer
npx prisma migrate diff

# État
npx prisma migrate status
```

**Migration générée** (`prisma/migrations/20250115100000_create_users_table/migration.sql`) :
```sql
-- CreateTable
CREATE TABLE `users` (
    `id` INTEGER NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(100) NOT NULL,
    `email` VARCHAR(255) NOT NULL,
    `age` INTEGER NULL,
    `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    `updated_at` DATETIME(3) NOT NULL,

    UNIQUE INDEX `users_email_key`(`email`),
    INDEX `users_email_idx`(`email`),
    PRIMARY KEY (`id`)
) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## Stratégies avancées

### 🔄 Migrations zero-downtime

**Problème** : Modifier une colonne peut bloquer la table en production.

**Solution : Migration en plusieurs étapes**

#### **Exemple : Renommer une colonne**

**❌ Migration dangereuse** :
```sql
-- BLOQUE LA TABLE pendant l'ALTER !
ALTER TABLE users CHANGE COLUMN name full_name VARCHAR(100);
```

**✅ Migration zero-downtime** :

**Étape 1** : Ajouter nouvelle colonne
```sql
-- V010__add_full_name_column.sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(100) NULL AFTER name;
```

**Étape 2** : Copier les données (application)
```python
# Migration de données via script applicatif
def migrate_name_to_full_name():
    batch_size = 1000
    offset = 0
    
    while True:
        users = db.execute(
            "SELECT id, name FROM users WHERE full_name IS NULL LIMIT %s OFFSET %s",
            (batch_size, offset)
        )
        
        if not users:
            break
        
        for user in users:
            db.execute(
                "UPDATE users SET full_name = %s WHERE id = %s",
                (user['name'], user['id'])
            )
        
        offset += batch_size
        time.sleep(0.1)  # Pause pour ne pas surcharger
```

**Étape 3** : Rendre obligatoire
```sql
-- V011__make_full_name_not_null.sql
-- Attendre que toutes les données soient migrées
ALTER TABLE users MODIFY COLUMN full_name VARCHAR(100) NOT NULL;
```

**Étape 4** : Mettre à jour l'application (utiliser `full_name`)

**Étape 5** : Supprimer ancienne colonne
```sql
-- V012__drop_name_column.sql
-- Après plusieurs jours de monitoring
ALTER TABLE users DROP COLUMN name;
```

### 📊 Migration de données volumineuses

**Problème** : Migrer 100M lignes peut prendre des heures.

**Solution : Migration par batch**

```sql
-- V020__migrate_user_status.sql

-- 1. Ajouter colonne avec valeur par défaut temporaire
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'pending';

-- 2. Créer procédure de migration par batch
DELIMITER $$

CREATE PROCEDURE migrate_user_status()
BEGIN
    DECLARE batch_size INT DEFAULT 10000;
    DECLARE rows_affected INT;
    
    REPEAT
        -- Migrer un batch
        UPDATE users
        SET status = CASE
            WHEN email_verified = 1 THEN 'active'
            WHEN email_verified = 0 THEN 'pending'
            ELSE 'unknown'
        END
        WHERE status = 'pending'
        LIMIT batch_size;
        
        SET rows_affected = ROW_COUNT();
        
        -- Pause pour ne pas surcharger
        DO SLEEP(0.1);
        
    UNTIL rows_affected = 0 END REPEAT;
END$$

DELIMITER ;

-- 3. Exécuter (peut être fait en arrière-plan)
CALL migrate_user_status();

-- 4. Nettoyer
DROP PROCEDURE migrate_user_status;

-- 5. Rendre obligatoire (dans migration suivante, après vérification)
-- ALTER TABLE users MODIFY COLUMN status VARCHAR(20) NOT NULL;
```

### 🔀 Gestion des branches Git

**Problème** : Deux développeurs créent des migrations avec le même numéro.

```
branch main:     V010 → V011 → V012
branch feature:  V010 → V011 → V013 (conflit!)
```

**Solution 1 : Numérotation par timestamp** (Flyway, Sequelize)
```
V20250115100000__create_users.sql
V20250115103000__add_email_index.sql
```

**Solution 2 : Numérotation avec ID unique** (Liquibase)
```xml
<changeSet id="john.doe-20250115-1" author="john.doe">
```

**Solution 3 : Prisma (gère automatiquement)**
```
20250115100000_create_users/
20250115103000_add_email_index/
```

**Workflow Git** :
```bash
# 1. Créer migration dans feature branch
git checkout -b feature/add-user-age
alembic revision -m "add user age"

# 2. Merge main dans feature (résoudre conflits)
git checkout feature/add-user-age
git merge main

# 3. Si conflit de migration : renommer/réordonner
# Alembic : modifier down_revision
# Flyway : renommer avec nouveau timestamp

# 4. Tester migrations
alembic upgrade head

# 5. Merge dans main
git checkout main
git merge feature/add-user-age
```

---

## CI/CD et automatisation

### 🚀 Pipeline de migration

```yaml
# .github/workflows/database-migrations.yml
name: Database Migrations

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test-migrations:
    runs-on: ubuntu-latest
    
    services:
      mariadb:
        image: mariadb:11.8
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: test_db
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install alembic mysqlclient
      
      - name: Run migrations
        run: |
          alembic upgrade head
        env:
          DATABASE_URL: mysql://root:root@localhost:3306/test_db
      
      - name: Validate migrations
        run: |
          alembic current
          alembic history
      
      - name: Test rollback
        run: |
          alembic downgrade -1
          alembic upgrade head

  deploy-migrations:
    needs: test-migrations
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to production
        run: |
          # Se connecter au serveur de production
          # Exécuter migrations avec précautions
          alembic upgrade head
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
```

### 🛡️ Bonnes pratiques CI/CD

```python
# scripts/safe_migrate.py
import sys
import subprocess
from alembic import command
from alembic.config import Config

def safe_migrate():
    """Migration sécurisée avec vérifications"""
    config = Config("alembic.ini")
    
    # 1. Vérifier connexion DB
    try:
        from sqlalchemy import create_engine
        engine = create_engine(config.get_main_option("sqlalchemy.url"))
        conn = engine.connect()
        conn.close()
        print("✓ Database connection OK")
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        sys.exit(1)
    
    # 2. Backup avant migration (recommandé)
    print("Creating backup...")
    subprocess.run([
        "mysqldump",
        "-h", "localhost",
        "-u", "user",
        "--databases", "myapp",
        "--result-file", f"backup_pre_migration_{datetime.now():%Y%m%d_%H%M%S}.sql"
    ])
    print("✓ Backup created")
    
    # 3. Dry-run (générer SQL sans exécuter)
    print("Generating migration SQL...")
    command.upgrade(config, "head", sql=True)
    
    # 4. Demander confirmation
    confirm = input("Proceed with migration? (yes/no): ")
    if confirm.lower() != 'yes':
        print("Migration aborted")
        sys.exit(0)
    
    # 5. Exécuter migration
    try:
        print("Running migrations...")
        command.upgrade(config, "head")
        print("✓ Migrations applied successfully")
    except Exception as e:
        print(f"✗ Migration failed: {e}")
        print("Please restore from backup!")
        sys.exit(1)
    
    # 6. Vérifier état
    command.current(config)

if __name__ == "__main__":
    safe_migrate()
```

---

## Rollback et gestion d'erreurs

### ⏮️ Stratégies de rollback

#### **1. Rollback automatique (transaction)**

```python
# Alembic avec transaction
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Toutes ces opérations dans une transaction
    with op.batch_alter_table('users') as batch_op:
        batch_op.add_column(sa.Column('age', sa.Integer()))
        batch_op.create_index('idx_users_age', ['age'])
    
    # Si erreur → rollback automatique

def downgrade():
    with op.batch_alter_table('users') as batch_op:
        batch_op.drop_index('idx_users_age')
        batch_op.drop_column('age')
```

#### **2. Rollback manuel**

```bash
# Flyway : Rollback N versions (édition Teams+)
flyway undo

# Liquibase : Rollback par tag
liquibase rollback v1.0

# Alembic : Rollback d'une migration
alembic downgrade -1

# EF Core : Rollback vers migration
dotnet ef database update PreviousMigration

# Sequelize : Rollback
npx sequelize-cli db:migrate:undo
```

#### **3. Restauration depuis backup**

```bash
# Créer backup avant migration critique
mysqldump -u user -p myapp > backup_pre_migration.sql

# Si migration échoue : restaurer
mysql -u user -p myapp < backup_pre_migration.sql
```

### 🚨 Gestion des erreurs

**Migration qui échoue en production** :

```sql
-- V050__problematic_migration.sql
ALTER TABLE users ADD COLUMN problematic_column TEXT;
-- Erreur : Out of memory (table trop grande)
```

**Solution** :

1. **Identifier le problème**
```bash
# Logs Flyway/Liquibase/Alembic
tail -f /var/log/migrations.log
```

2. **Marquer comme échec**
```bash
# Flyway : Réparer l'historique
flyway repair

# Liquibase : Supprimer changeset échoué
liquibase clear-checksums
```

3. **Corriger la migration**
```sql
-- V050__fixed_migration.sql (nouveau fichier)
ALTER TABLE users ADD COLUMN problematic_column VARCHAR(1000);
-- Limite de taille pour éviter out of memory
```

4. **Retester en staging**
```bash
# Environnement de staging identique à production
alembic upgrade head
```

---

## ✅ Checklist des bonnes pratiques

### 📋 Avant de créer une migration

- [ ] Le changement est-il vraiment nécessaire ?
- [ ] La migration est-elle **idempotente** (si possible) ?
- [ ] Y a-t-il un **rollback** prévu ?
- [ ] La migration est-elle **testée** en local ?
- [ ] Les données existantes sont-elles **préservées** ?
- [ ] L'impact sur les **performances** est-il acceptable ?
- [ ] Le **downtime** est-il géré (zero-downtime si nécessaire) ?

### 📋 Pendant le développement

- [ ] Nommage **clair et explicite**
- [ ] Un changement = une migration (pas tout dans V001)
- [ ] Migrations **versionnées** avec Git
- [ ] **Tests** automatisés (apply + rollback)
- [ ] **Documentation** si migration complexe

### 📋 Avant le déploiement

- [ ] **Backup** de la base de production
- [ ] Migrations testées en **staging** (environnement identique)
- [ ] **Dry-run** (générer SQL, vérifier)
- [ ] **Plan de rollback** documenté
- [ ] **Monitoring** activé
- [ ] **Fenêtre de maintenance** si nécessaire

### 📋 Après le déploiement

- [ ] Vérifier que la migration s'est bien appliquée
- [ ] Vérifier les **logs** d'application
- [ ] Monitorer les **performances**
- [ ] Garder le **backup** quelques jours
- [ ] **Documenter** les incidents éventuels

---

## ✅ Points clés à retenir

- 📝 **Migrations = code** : Versionnées, testées, reviewées
- 🔄 **Immuabilité** : JAMAIS modifier une migration appliquée
- 📊 **Séquentialité** : Ordre strict d'exécution (V1 → V2 → V3)
- ↩️ **Réversibilité** : Toujours prévoir un rollback
- 🛠️ **Outils** : Flyway (Java), Liquibase (flexible), Alembic (Python), EF Core (.NET), Sequelize (Node.js), Prisma (TypeScript)
- 🚀 **CI/CD** : Migrations automatiques dans le pipeline
- 🔒 **Sécurité** : Backup avant migration critique
- 📈 **Zero-downtime** : Migration en plusieurs étapes pour gros changements
- 🐛 **Gestion d'erreurs** : Logs, repair, rollback plan
- 🎯 **Best practices** : Idempotence, tests, documentation

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [Flyway Documentation](https://flywaydb.org/documentation/)
- 📖 [Liquibase Documentation](https://docs.liquibase.com/)
- 📖 [Alembic Documentation](https://alembic.sqlalchemy.org/)
- 📖 [Entity Framework Core Migrations](https://learn.microsoft.com/ef/core/managing-schemas/migrations/)
- 📖 [Sequelize Migrations](https://sequelize.org/docs/v6/other-topics/migrations/)
- 📖 [Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate)

### **Guides et best practices**
- 📝 [Database Migrations Done Right](https://www.brunton-spall.co.uk/post/2014/05/06/database-migrations-done-right/)
- 📝 [Zero-Downtime Database Migrations](https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/)
- 📝 [Evolutionary Database Design](https://martinfowler.com/articles/evodb.html)

### **Outils**
- 🛠️ [golang-migrate](https://github.com/golang-migrate/migrate)
- 🛠️ [Flyway](https://flywaydb.org/)
- 🛠️ [Liquibase](https://www.liquibase.org/)

---

## ➡️ Sections suivantes

- **17.6** - Tests de bases de données : Stratégies, fixtures, mocking avancé
- **17.7** - Environnements de développement : Docker, seeding, reproductibilité
- **17.8** - Prévention des injections SQL : Comprendre, détecter, corriger
- **17.9** - Prepared Statements : Fonctionnement, sécurité, performances

---

**MariaDB** : Version 11.8 LTS

⏭️ [Tests de bases de données](/17-integration-developpement/06-tests-bases-donnees.md)

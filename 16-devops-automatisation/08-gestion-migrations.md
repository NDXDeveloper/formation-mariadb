🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.8 Gestion des migrations

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 7-8 heures  
> **Prérequis** : 
> - Section 16.7 CI/CD pour bases de données maîtrisée
> - Compréhension profonde de SQL DDL (ALTER TABLE, CREATE INDEX, etc.)
> - Expérience avec migrations de schéma en production
> - Connaissance des locks MariaDB (metadata locks, table locks)
> - Notions de performance tuning

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les défis des migrations de schéma en production
- **Choisir** l'outil approprié selon le contexte (Flyway, Liquibase, gh-ost, pt-osc)
- **Concevoir** des migrations backward-compatible et forward-only
- **Implémenter** des migrations zero-downtime sur tables volumineuses
- **Gérer** les rollbacks et recovery de migrations échouées
- **Automatiser** le processus de migration dans un pipeline CI/CD
- **Monitorer** l'exécution des migrations en temps réel
- **Appliquer** les patterns de migration éprouvés

---

## Introduction

### Qu'est-ce qu'une migration de schéma ?

Une **migration de schéma** (schema migration) est un **changement contrôlé et versionné** de la structure d'une base de données.

**Exemples de migrations** :

```sql
-- Migration V001: Créer table users
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Migration V002: Ajouter colonne name
ALTER TABLE users ADD COLUMN name VARCHAR(100);

-- Migration V003: Créer index sur email
CREATE INDEX idx_users_email ON users(email);

-- Migration V004: Ajouter contrainte
ALTER TABLE users 
    ADD CONSTRAINT chk_email_format 
    CHECK (email LIKE '%@%');
```

### Pourquoi la gestion des migrations est critique ?

```
┌──────────────────────────────────────────────────────────────┐
│         Problèmes sans gestion de migrations                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ Environnements désynchronisés                            │
│     Dev ≠ Staging ≠ Production                               │
│                                                              │
│  ❌ Scripts SQL éparpillés                                   │
│     "migration_final_v2_FINAL.sql"                           │
│                                                              │
│  ❌ État de la base inconnu                                  │
│     "Quelles migrations sont appliquées ?"                   │
│                                                              │
│  ❌ Rollback impossible                                      │
│     Changements destructifs irréversibles                    │
│                                                              │
│  ❌ Coordination app ↔ DB difficile                          │
│     Nouvelle app version attend nouvelle colonne             │
│                                                              │
│  ❌ Downtime prolongé en production                          │
│     ALTER TABLE sur table de 100GB = plusieurs heures        │
│                                                              │
└──────────────────────────────────────────────────────────────┘

                              VS

┌──────────────────────────────────────────────────────────────┐
│          Avec gestion de migrations                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Versioning clair                                         │
│     V001, V002, V003... ordre garanti                        │
│                                                              │
│  ✅ Traçabilité complète                                     │
│     Table schema_version track tout                          │
│                                                              │
│  ✅ Reproductibilité                                         │
│     Même migrations = même schéma                            │
│                                                              │
│  ✅ Tests automatisés                                        │
│     Migrations appliquées sur DB test dans CI                │
│                                                              │
│  ✅ Rollback strategy                                        │
│     Migrations backward-compatible                           │
│                                                              │
│  ✅ Zero-downtime possible                                   │
│     gh-ost / pt-online-schema-change                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Défis des migrations en production

### 1. Downtime

**Problème** : `ALTER TABLE` sur grande table = lock + temps long

```sql
-- Table de 100 millions de lignes
ALTER TABLE orders ADD COLUMN status VARCHAR(20);

-- Temps d'exécution:
-- - Petite table (<1M rows): quelques secondes
-- - Moyenne table (1-10M rows): minutes
-- - Grande table (>100M rows): HEURES ⚠️
-- 
-- Pendant ce temps:
-- ❌ Table LOCKED (metadata lock)
-- ❌ Pas de READ, pas de WRITE
-- ❌ Application DOWN
```

**Solutions** :
1. **gh-ost** : Online schema change (GitHub)
2. **pt-online-schema-change** : Online schema change (Percona)
3. **Migrations décomposées** : En plusieurs étapes
4. **Maintenance window** : Accepter downtime planifié

### 2. Coordination app ↔ database

**Problème** : Application v2 attend schéma v2

```
Timeline classique (PROBLEMATIQUE):

T0: Deploy database migration (ADD COLUMN status)
T1: Migration en cours (5 minutes)
    ↓ App v1 encore active
    ↓ Peut crash si attend colonne status
T2: Migration terminée
T3: Deploy app v2 (utilise column status)
    ↓ Mais si app v2 deployée AVANT migration terminée ?
    ↓ ERROR: Unknown column 'status'
```

**Solution : Backward-compatible migrations**

```
Phase 1: Migration (v1)
- ADD COLUMN status VARCHAR(20) NULL  -- Nullable!
- App v1 ignore cette colonne (backward-compatible)

Phase 2: Deploy app v2
- App v2 commence à utiliser column status
- Remplit progressivement les valeurs

Phase 3: Cleanup (v3)
- ALTER TABLE MODIFY status VARCHAR(20) NOT NULL
- Maintenant que toutes les lignes ont une valeur
```

### 3. Data migrations vs Schema migrations

**Schema migration** : Structure (DDL)
```sql
ALTER TABLE users ADD COLUMN age INT;
```

**Data migration** : Données (DML)
```sql
UPDATE users SET age = YEAR(CURDATE()) - YEAR(birth_date);
```

⚠️ **Piège** : Data migration sur grande table = très long

**Solution** : Background migration
```sql
-- Au lieu de:
UPDATE users SET age = ...;  -- Bloque tout

-- Faire:
UPDATE users SET age = ... WHERE id BETWEEN 1 AND 10000;
UPDATE users SET age = ... WHERE id BETWEEN 10001 AND 20000;
-- etc. (batch processing)
```

### 4. Rollback complexe

**Migrations forward-only** : Facile à appliquer, difficile à annuler

```sql
-- Migration V005: DROP COLUMN
ALTER TABLE users DROP COLUMN legacy_field;

-- Rollback = ❌ IMPOSSIBLE !
-- Données perdues définitivement
```

**Solution** : Expand-Contract pattern (voir plus bas)

---

## Comparaison des outils de migration

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                    Landscape des outils                      │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Schema Version Control                         │  │
│  │  (Track et apply migrations SQL)                       │  │
│  │                                                        │  │
│  │  ┌──────────────┐  ┌──────────────┐                    │  │
│  │  │   Flyway     │  │  Liquibase   │                    │  │
│  │  │              │  │              │                    │  │
│  │  │ - Simple     │  │ - Puissant   │                    │  │
│  │  │ - SQL pur    │  │ - XML/YAML   │                    │  │
│  │  │ - Gratuit    │  │ - Rollback   │                    │  │
│  │  └──────────────┘  └──────────────┘                    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Online Schema Change                           │  │
│  │  (Zero-downtime pour grandes tables)                   │  │
│  │                                                        │  │
│  │  ┌──────────────┐  ┌──────────────┐                    │  │
│  │  │   gh-ost     │  │    pt-osc    │                    │  │
│  │  │   (GitHub)   │  │  (Percona)   │                    │  │
│  │  │              │  │              │                    │  │
│  │  │ - Triggers   │  │ - Triggers   │                    │  │
│  │  │ - Safe       │  │ - Mature     │                    │  │
│  │  │ - Pausable   │  │ - Complex    │                    │  │
│  │  └──────────────┘  └──────────────┘                    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Tableau comparatif détaillé

| Critère | Flyway | Liquibase | gh-ost | pt-online-schema-change |
|---------|--------|-----------|--------|-------------------------|
| **Type** | Schema versioning | Schema versioning | Online schema change | Online schema change |
| **Développeur** | Redgate | Datical/Liquibase | GitHub | Percona |
| **License** | Apache 2.0 (OSS) + Commercial | Apache 2.0 (OSS) + Commercial | MIT (OSS) | GPL (OSS) |
| **Langage** | Java | Java | Go | Perl |
| **Format migrations** | SQL pur | SQL, XML, YAML, JSON | SQL (ALTER TABLE) | SQL (ALTER TABLE) |
| **Courbe apprentissage** | ⭐⭐ Facile | ⭐⭐⭐⭐ Difficile | ⭐⭐⭐ Moyenne | ⭐⭐⭐⭐ Difficile |
| **Use case principal** | CI/CD, versioning simple | Enterprise, multi-DB | Production zero-downtime | Production zero-downtime |
| **Rollback** | ❌ Manuel (scripts undo) | ✅ Automatique (rollback tags) | ❌ Non (online change) | ❌ Non (online change) |
| **Databases supportées** | MySQL, PostgreSQL, Oracle, SQL Server, etc. | MySQL, PostgreSQL, Oracle, SQL Server, MongoDB, etc. | MySQL, MariaDB | MySQL, MariaDB, Percona |
| **Zero-downtime** | ❌ Non (ALTER TABLE bloque) | ❌ Non (ALTER TABLE bloque) | ✅ Oui (copie table) | ✅ Oui (copie table) |
| **Grandes tables** | ⚠️  Problématique (lock) | ⚠️  Problématique (lock) | ✅ Conçu pour ça | ✅ Conçu pour ça |
| **Réplication support** | ⚠️  Via binlog standard | ⚠️  Via binlog standard | ✅ Supporte réplication | ✅ Supporte réplication |
| **Monitoring** | ⚠️  Via logs | ⚠️  Via logs | ✅ Throttling, pausable | ✅ Progress tracking |
| **Safety checks** | ✅ Checksums | ✅ Preconditions | ✅ Dry-run, postpone | ✅ Check replicas |
| **CI/CD integration** | ✅ Excellent | ✅ Excellent | ⚠️  Possible mais séparé | ⚠️  Possible mais séparé |
| **Community** | ⭐⭐⭐⭐⭐ Très large | ⭐⭐⭐⭐ Large | ⭐⭐⭐⭐ Large | ⭐⭐⭐⭐ Large |
| **Documentation** | ✅ Excellente | ✅ Excellente | ✅ Bonne | ✅ Bonne |
| **Performance** | Rapide (petites tables) | Rapide (petites tables) | Plus lent (copie table) | Plus lent (copie table) |
| **Idempotence** | ✅ Oui (checksums) | ✅ Oui (changelogs) | ⚠️  Dépend du script | ⚠️  Dépend du script |
| **État tracking** | Table `flyway_schema_history` | Table `DATABASECHANGELOG` | ❌ Non (one-shot tool) | ❌ Non (one-shot tool) |
| **Dry-run / Test mode** | ⚠️  Limité | ✅ Oui | ✅ Oui | ✅ Oui |
| **Coût** | Gratuit (Community) + Teams (€€) + Enterprise (€€€€) | Gratuit (OSS) + Pro (€€€) + Enterprise (€€€€€) | Gratuit (OSS) | Gratuit (OSS) |

### Quand utiliser quel outil ?

```
┌──────────────────────────────────────────────────────────────┐
│                  Arbre de décision                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Quel est votre besoin principal ?                        │
│                                                              │
│     A) Versioning + CI/CD + Multi-DB                         │
│        └─> Flyway (simple) ou Liquibase (avancé)             │
│                                                              │
│     B) ALTER TABLE sans downtime sur grande table            │
│        └─> gh-ost (moderne) ou pt-osc (mature)               │
│                                                              │
│  2. Quelle est la taille de vos tables ?                     │
│                                                              │
│     A) Petites (<10M rows)                                   │
│        └─> Flyway/Liquibase suffisent                        │
│                                                              │
│     B) Grandes (>100M rows)                                  │
│        └─> gh-ost/pt-osc obligatoires                        │
│                                                              │
│  3. Quel est votre budget downtime ?                         │
│                                                              │
│     A) Downtime acceptable (maintenance window)              │
│        └─> Flyway/Liquibase                                  │
│                                                              │
│     B) Zero-downtime requis (24/7)                           │
│        └─> gh-ost/pt-osc                                     │
│                                                              │
│  4. Quelle est votre stack ?                                 │
│                                                              │
│     A) Multi-database (MySQL + PostgreSQL + Oracle)          │
│        └─> Liquibase (support le plus large)                 │
│                                                              │
│     B) MySQL/MariaDB seulement                               │
│        └─> Flyway (plus simple) ou gh-ost/pt-osc             │
│                                                              │
│  5. Avez-vous besoin de rollback automatique ?               │
│                                                              │
│     A) Oui (compliance, audit)                               │
│        └─> Liquibase (seul à le supporter nativement)        │
│                                                              │
│     B) Non (rollback manuel acceptable)                      │
│        └─> Flyway                                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Recommandations par scénario** :

| Scénario | Outil recommandé | Pourquoi |
|----------|------------------|----------|
| Startup, petite base | Flyway | Simple, gratuit, facile à intégrer |
| Enterprise multi-DB | Liquibase | Support large, rollback, governance |
| Production 24/7, tables >100M rows | gh-ost | Zero-downtime, pausable, safe |
| Production avec réplication complexe | pt-osc | Mature, gère bien la réplication |
| CI/CD pipeline | Flyway | Excellente intégration, rapide |
| Conformité stricte (audit, rollback) | Liquibase | Changesets, rollback automatique |
| Migration one-shot grande table | gh-ost ou pt-osc | Minimise risque |

💡 **Best practice** : **Combiner** les outils !
```
Flyway (versioning quotidien)
    +
gh-ost (migrations lourdes ponctuelles)
```

---

## Stratégies de migration

### 1. Forward-only migrations

**Principe** : Migrations ne vont que dans un sens (avant), jamais en arrière.

```
V001 → V002 → V003 → V004
  ✅     ✅     ✅     ✅
  
Jamais:
V004 → V003 (rollback destructif)
  ❌
```

**Pourquoi ?**

- **Données perdues** : DROP COLUMN = perte irréversible
- **Complexité** : Rollback = écrire migration inverse (double travail)
- **Erreurs** : Migration inverse peut échouer aussi

**Solution : Backward-compatible changes**

### 2. Backward-compatible migrations

**Principe** : Nouvelle version DB compatible avec ancienne version app

```
┌──────────────────────────────────────────────────────────────┐
│              Backward-compatible Migration                   │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Phase 1: Migration DB                                  │ │
│  │  ALTER TABLE users ADD COLUMN status VARCHAR(20) NULL;  │ │
│  │                                                         │ │
│  │  ✅ App v1 peut continuer à fonctionner                 │ │
│  │     (ignore nouvelle colonne)                           │ │
│  └─────────────────────────────────────────────────────────┘ │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Phase 2: Deploy app v2                                 │ │
│  │  App v2 utilise colonne status                          │ │
│  │                                                         │ │
│  │  ✅ Migration déjà appliquée                            │ │
│  │  ✅ Pas de downtime                                     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Phase 3 (optionnel): Cleanup                           │ │
│  │  ALTER TABLE users MODIFY status VARCHAR(20) NOT NULL;  │ │
│  │                                                         │ │
│  │  ℹ️  Seulement quand app v1 n'existe plus               │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

**Règles pour backward-compatibility** :

| ❌ Breaking Change | ✅ Backward-compatible |
|-------------------|----------------------|
| DROP COLUMN | ADD COLUMN (nullable) |
| ALTER COLUMN NOT NULL | Laisser NULL temporairement |
| RENAME COLUMN | ADD nouvelle + garder ancienne |
| DROP TABLE | Laisser table vide |
| Change DATATYPE (incompatible) | Nouvelle colonne + migration graduelle |

### 3. Expand-Contract pattern

**Le pattern le plus sûr pour migrations complexes**

```
┌───────────────────────────────────────────────────────────────┐
│                  Expand-Contract Pattern                      │
│                                                               │
│  Exemple: Renommer colonne "name" en "full_name"              │
│                                                               │
│  ─────────────────────────────────────────────────────────    │
│                                                               │
│  EXPAND (Phase 1):                                            │
│  ├─ ALTER TABLE users ADD COLUMN full_name VARCHAR(255);      │
│  ├─ CREATE TRIGGER sync_name_to_full_name                     │
│  │    AFTER INSERT/UPDATE                                     │
│  │    SET NEW.full_name = NEW.name;                           │
│  └─ App v1 continue à utiliser "name"                         │
│     App v1 ne sait pas que "full_name" existe                 │
│                                                               │
│  ─────────────────────────────────────────────────────────    │
│                                                               │
│  MIGRATE DATA (Background):                                   │
│  └─ UPDATE users SET full_name = name WHERE full_name IS NULL;│
│                                                               │
│  ─────────────────────────────────────────────────────────    │
│                                                               │
│  DUAL WRITE (Phase 2):                                        │
│  ├─ App v2 écrit dans "name" ET "full_name"                   │
│  └─ Garantit synchronisation                                  │
│                                                               │
│  ─────────────────────────────────────────────────────────    │
│                                                               │
│  SWITCH (Phase 3):                                            │
│  ├─ App v3 lit "full_name" uniquement                         │
│  ├─ App v3 écrit dans "full_name" uniquement                  │
│  └─ "name" n'est plus utilisée                                │
│                                                               │
│  ─────────────────────────────────────────────────────────    │
│                                                               │
│  CONTRACT (Phase 4):                                          │
│  ├─ DROP TRIGGER sync_name_to_full_name;                      │
│  ├─ ALTER TABLE users DROP COLUMN name;                       │
│  └─ Cleanup terminé                                           │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Zero-downtime
- ✅ Rollback facile (revenir à phase précédente)
- ✅ Testable (chaque phase isolée)
- ✅ Safe (données jamais perdues)

**Inconvénients** :
- ⚠️  Complexe (4 phases)
- ⚠️  Long (plusieurs deployments)
- ⚠️  Storage temporaire doublé

### 4. Decomposed migrations

**Principe** : Décomposer grande migration en petites étapes

**Exemple : Ajouter index sur grande table**

```sql
-- ❌ Approche naïve (bloque table pendant 2h)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- ✅ Approche décomposée
-- Étape 1: Créer index CONCURRENTLY (MariaDB 10.6+)
ALTER TABLE orders 
    ADD INDEX idx_orders_customer_id (customer_id) 
    ALGORITHM=INPLACE, LOCK=NONE;

-- Ou avec gh-ost (si ALGORITHM=INPLACE pas dispo)
gh-ost \
    --alter="ADD INDEX idx_orders_customer_id (customer_id)" \
    --execute
```

---

## Patterns de migration courants

### Pattern 1: Ajouter colonne avec valeur par défaut

**❌ Problématique** :
```sql
-- Bloque table pour remplir toutes les lignes
ALTER TABLE users ADD COLUMN country VARCHAR(2) DEFAULT 'US';
```

**✅ Solution** :
```sql
-- Phase 1: Ajouter colonne NULL
ALTER TABLE users ADD COLUMN country VARCHAR(2) NULL;

-- Phase 2: Remplir en background (batches)
UPDATE users SET country = 'US' WHERE country IS NULL LIMIT 10000;
-- Répéter jusqu'à terminé

-- Phase 3: Rendre NOT NULL
ALTER TABLE users MODIFY COLUMN country VARCHAR(2) NOT NULL DEFAULT 'US';
```

### Pattern 2: Changer type de colonne

**❌ Problématique** :
```sql
-- Risqué: perte de données si incompatible
ALTER TABLE orders MODIFY COLUMN amount DECIMAL(12,2);
```

**✅ Solution (Expand-Contract)** :
```sql
-- Phase 1: Nouvelle colonne
ALTER TABLE orders ADD COLUMN amount_new DECIMAL(12,2);

-- Phase 2: Migrer données
UPDATE orders SET amount_new = CAST(amount AS DECIMAL(12,2));

-- Phase 3: App utilise amount_new

-- Phase 4: Drop ancienne colonne
ALTER TABLE orders DROP COLUMN amount;
ALTER TABLE orders CHANGE amount_new amount DECIMAL(12,2);
```

### Pattern 3: Ajouter contrainte

**❌ Problématique** :
```sql
-- Échoue si données existantes invalides
ALTER TABLE users ADD CONSTRAINT chk_email CHECK (email LIKE '%@%');
```

**✅ Solution** :
```sql
-- Phase 1: Vérifier données existantes
SELECT COUNT(*) FROM users WHERE email NOT LIKE '%@%';
-- Si > 0, nettoyer d'abord

-- Phase 2: Ajouter contrainte en mode NOT ENFORCED (MariaDB 10.2+)
ALTER TABLE users 
    ADD CONSTRAINT chk_email 
    CHECK (email LIKE '%@%') 
    NOT ENFORCED;

-- Phase 3: Vérifier nouvelles insertions pendant période de test

-- Phase 4: Enforcer
ALTER TABLE users ALTER CONSTRAINT chk_email ENFORCED;
```

### Pattern 4: Split table (normalisation)

**Scénario** : Déplacer colonnes vers nouvelle table

```sql
-- Avant: Table users avec adresse
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    address_street VARCHAR(255),
    address_city VARCHAR(100),
    address_country VARCHAR(2)
);

-- Après: Table addresses séparée
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    address_id BIGINT
);

CREATE TABLE addresses (
    id BIGINT PRIMARY KEY,
    street VARCHAR(255),
    city VARCHAR(100),
    country VARCHAR(2)
);
```

**Migration (Expand-Contract)** :

```sql
-- Phase 1: Créer nouvelle table
CREATE TABLE addresses (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    street VARCHAR(255),
    city VARCHAR(100),
    country VARCHAR(2)
);

-- Phase 2: Ajouter FK dans users
ALTER TABLE users ADD COLUMN address_id BIGINT NULL;

-- Phase 3: Migrer données
INSERT INTO addresses (street, city, country)
SELECT DISTINCT address_street, address_city, address_country 
FROM users 
WHERE address_street IS NOT NULL;

UPDATE users u
SET address_id = (
    SELECT a.id 
    FROM addresses a
    WHERE a.street = u.address_street
      AND a.city = u.address_city
      AND a.country = u.address_country
    LIMIT 1
);

-- Phase 4: App v2 utilise address_id

-- Phase 5: Drop anciennes colonnes
ALTER TABLE users 
    DROP COLUMN address_street,
    DROP COLUMN address_city,
    DROP COLUMN address_country;
```

### Pattern 5: Merge tables (dénormalisation)

**Inverse du pattern précédent** (pour performance)

```sql
-- Avant: Deux tables avec JOIN coûteux
SELECT u.*, a.*
FROM users u
JOIN addresses a ON u.address_id = a.id;
-- Lent si appelé 1000x/seconde

-- Après: Table dénormalisée (colonnes dupliquées)
SELECT * FROM users;
-- Rapide, pas de JOIN
```

**Migration** : Similaire au pattern 4 mais dans l'autre sens.

---

## Zero-downtime migrations

### Principe des outils online schema change

**gh-ost et pt-online-schema-change utilisent le même principe** :

```
┌──────────────────────────────────────────────────────────────┐
│           Online Schema Change - Fonctionnement              │
│                                                              │
│  1️⃣  Créer table fantôme (_new)                              │
│     CREATE TABLE orders_new LIKE orders;                     │
│     ALTER TABLE orders_new ADD COLUMN status VARCHAR(20);    │
│                                                              │
│  2️⃣  Copier données en petits batches                        │
│     INSERT INTO orders_new SELECT *, NULL FROM orders        │
│     WHERE id BETWEEN 1 AND 10000;                            │
│     -- Répéter par batch de 10k rows                         │
│                                                              │
│  3️⃣  Capturer changements pendant copie (triggers)           │
│     CREATE TRIGGER orders_insert                             │
│         AFTER INSERT ON orders                               │
│         INSERT INTO orders_new SELECT NEW.*;                 │
│                                                              │
│     CREATE TRIGGER orders_update ...                         │
│     CREATE TRIGGER orders_delete ...                         │
│                                                              │
│  4️⃣  Basculer atomiquement                                   │
│     RENAME TABLE                                             │
│         orders TO orders_old,                                │
│         orders_new TO orders;                                │
│     -- Atomique! Downtime = millisecondes                    │
│                                                              │
│  5️⃣  Cleanup                                                 │
│     DROP TABLE orders_old;                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Zero-downtime (sauf RENAME atomique)
- ✅ Pausable (si impacte performance)
- ✅ Rollback facile (garder ancienne table)
- ✅ Progress tracking en temps réel

**Inconvénients** :
- ⚠️  Plus lent que ALTER TABLE direct
- ⚠️  Consomme 2x le storage temporairement
- ⚠️  Triggers ajoutent overhead

### Comparaison gh-ost vs pt-online-schema-change

| Aspect | gh-ost | pt-online-schema-change |
|--------|--------|-------------------------|
| **Triggers** | ✅ Utilise binlog (pas triggers sur table) | ⚠️  Crée triggers (overhead) |
| **Safety** | ✅ Throttling automatique | ✅ Throttling configurable |
| **Réplication** | ✅ Utilise replica pour lire | ⚠️  Lit depuis primary |
| **Pause/Resume** | ✅ Oui | ✅ Oui |
| **Dry-run** | ✅ Oui | ✅ Oui |
| **Monitoring** | ✅ Metrics détaillées | ⚠️  Moins de metrics |
| **Langage** | Go (binary standalone) | Perl (dépendances) |
| **Performance** | Généralement plus rapide | Mature mais plus lent |
| **Support MariaDB** | ✅ Excellent | ✅ Excellent |

💡 **Recommandation** : **gh-ost** pour nouveaux projets (plus moderne, meilleure performance)

### Exemple complet gh-ost

```bash
#!/bin/bash
# migrate-add-status-column.sh

gh-ost \
  # Connexion
  --host=mariadb-primary.prod.svc.cluster.local \
  --port=3306 \
  --user=gh-ost \
  --password="${GHOST_PASSWORD}" \
  --database=myapp \
  --table=orders \
  
  # Migration
  --alter="ADD COLUMN status VARCHAR(20) NULL" \
  
  # Safety
  --allow-on-master \
  --assume-rbr \
  --concurrent-rowcount \
  
  # Performance
  --chunk-size=10000 \
  --max-load="Threads_running=50,Threads_connected=1000" \
  --critical-load="Threads_running=100,Threads_connected=2000" \
  --default-retries=120 \
  
  # Throttling
  --throttle-control-replicas="mariadb-replica-1:3306,mariadb-replica-2:3306" \
  --max-lag-millis=1500 \
  
  # Monitoring
  --serve-socket-file=/tmp/gh-ost.sock \
  --postpone-cut-over-flag-file=/tmp/gh-ost.postpone \
  
  # Execution
  --execute \
  --verbose \
  
  # Cleanup
  --initially-drop-ghost-table \
  --initially-drop-old-table
```

**Monitoring pendant exécution** :

```bash
# Voir le progrès en temps réel
echo "status" | nc -U /tmp/gh-ost.sock

# Throttle temporairement (si besoin)
echo "throttle" | nc -U /tmp/gh-ost.sock

# Reprendre
echo "no-throttle" | nc -U /tmp/gh-ost.sock

# Postpone cut-over (tester avant switch)
touch /tmp/gh-ost.postpone

# Exécuter cut-over (switch)
rm /tmp/gh-ost.postpone
```

---

## Rollback et recovery

### Stratégies de rollback

**1. Rollback via backup**

```bash
# Avant migration
mysqldump myapp > backup-pre-migration.sql

# Si problème
mysql myapp < backup-pre-migration.sql
```

⚠️  **Problème** : Perte des données créées après migration

**2. Rollback via migration inverse** (Liquibase)

```xml
<changeSet id="add-status-column" author="dba">
  <addColumn tableName="orders">
    <column name="status" type="varchar(20)"/>
  </addColumn>
  
  <rollback>
    <dropColumn tableName="orders" columnName="status"/>
  </rollback>
</changeSet>
```

```bash
liquibase rollback-count 1
```

**3. Rollback via Expand-Contract**

```
Si problème en Phase 3 (Switch):
→ Revenir Phase 2 (Dual Write)
→ Ou Phase 1 (Expand seulement)
→ Données jamais perdues
```

### Recovery de migration échouée

**Scénario** : Migration échoue au milieu

```sql
-- Migration V005_add_indexes.sql
CREATE INDEX idx_orders_customer ON orders(customer_id);  -- ✅ OK
CREATE INDEX idx_orders_date ON orders(created_at);       -- ❌ FAIL
CREATE INDEX idx_orders_status ON orders(status);         -- ❓ Pas exécuté
```

**Problèmes** :
1. Premier index créé
2. Migration marquée comme failed
3. Re-exécuter = erreur "index exists"

**Solution 1 : Idempotence**

```sql
-- V005_add_indexes.sql (idempotent)
CREATE INDEX IF NOT EXISTS idx_orders_customer ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_orders_date ON orders(created_at);
CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status);
```

**Solution 2 : Transactions (limité pour DDL)**

```sql
-- MariaDB: DDL auto-commit, pas de vrai transaction
-- Mais peut wrapper pour logging:

START TRANSACTION;

-- Log
INSERT INTO migration_log VALUES ('V005', 'start', NOW());

CREATE INDEX idx_orders_customer ON orders(customer_id);
INSERT INTO migration_log VALUES ('V005', 'idx_customer', NOW());

CREATE INDEX idx_orders_date ON orders(created_at);
INSERT INTO migration_log VALUES ('V005', 'idx_date', NOW());

COMMIT;
```

**Solution 3 : Flyway repair**

```bash
# Si migration failed
flyway repair

# Marque migration comme success manuellement
# Puis corriger et re-exécuter
```

---

## Best practices

### 1. Naming conventions

```sql
-- ✅ BON
V001__create_users_table.sql
V002__add_users_email_index.sql
V003__add_orders_status_column.sql

-- ❌ MAUVAIS
migration1.sql
fix.sql
new_changes_final_v2.sql
```

**Format recommandé** :
```
V{version}__{description}.sql

Version: 001, 002, 003... (padding zeros)
Description: snake_case, descriptive
```

### 2. Une migration = un changement logique

```sql
-- ✅ BON: Migration focused
-- V010__add_user_status.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20);

-- ❌ MAUVAIS: Trop de changements
-- V010__big_refactoring.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20);
CREATE TABLE logs (...);
ALTER TABLE orders ADD INDEX idx_status;
-- Difficile à rollback, difficile à tester
```

### 3. Tester migrations sur données réelles

```bash
# 1. Dump production (sample)
mysqldump production --where="id < 100000" > prod-sample.sql

# 2. Restore sur staging
mysql staging < prod-sample.sql

# 3. Tester migrations
flyway migrate -url=jdbc:mysql://staging

# 4. Vérifier résultats
mysql staging -e "SELECT COUNT(*) FROM users WHERE status IS NULL"
```

### 4. Documenter les migrations

```sql
-- V015__add_audit_columns.sql

/*
 * Migration: Add audit columns to all tables
 * 
 * Context: Compliance requirement for SOC2
 * 
 * Impact:
 * - Adds 3 columns to each table (created_at, updated_at, updated_by)
 * - Estimated time: 5 minutes per table
 * - Downtime: None (nullable columns)
 * 
 * Rollback:
 * - Keep columns (no harm)
 * - Or manual DROP COLUMN if needed
 * 
 * Author: DBA Team
 * Date: 2025-12-14
 * JIRA: INFRA-1234
 */

ALTER TABLE users 
    ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ADD COLUMN updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    ADD COLUMN updated_by VARCHAR(100);
```

### 5. Monitoring pendant migration

```bash
#!/bin/bash
# monitor-migration.sh

# Pendant migration, surveiller:

# 1. Threads connectés
watch -n 5 'mysql -e "SHOW STATUS LIKE \"Threads_connected\""'

# 2. Threads en cours
watch -n 5 'mysql -e "SHOW PROCESSLIST"'

# 3. Replication lag (si replicas)
watch -n 5 'mysql -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master'

# 4. Table locks
watch -n 5 'mysql -e "SHOW OPEN TABLES WHERE In_use > 0"'

# 5. InnoDB status
watch -n 5 'mysql -e "SHOW ENGINE INNODB STATUS\G" | grep "TRANSACTIONS"'
```

### 6. Checklist pré-migration production

```
☐ Migration testée sur staging avec données réalistes
☐ Impact analysé (temps, lock, downtime)
☐ Backup complet effectué
☐ Rollback plan documenté
☐ Équipe on-call notifiée
☐ Monitoring dashboard prêt
☐ Maintenance window communiquée
☐ Approbation(s) obtenue(s)
☐ Scripts de rollback testés
☐ gh-ost/pt-osc configuré si nécessaire
```

---

## ✅ Points clés à retenir

- **Outils complémentaires** : Flyway/Liquibase (versioning) + gh-ost/pt-osc (online changes)
- **Forward-only** : Migrations ne vont qu'en avant, jamais de rollback destructif
- **Backward-compatible** : Nouvelle DB fonctionne avec ancienne app
- **Expand-Contract** : Pattern le plus sûr pour migrations complexes
- **Zero-downtime** : gh-ost pour tables >100M rows
- **Idempotence** : Migrations safe à rejouer (CREATE IF NOT EXISTS)
- **Testing** : Tester sur données réalistes avant production
- **Monitoring** : Surveiller locks, connections, replication lag
- **Documentation** : Expliquer POURQUOI, pas seulement QUOI
- **Checklist** : Validation systématique avant production

💡 **Golden Rule** : Better safe than sorry. Préférer 3 petites migrations qu'une grosse.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Flyway](https://flywaydb.org/documentation/)
- [📖 Liquibase](https://docs.liquibase.com/)
- [📖 gh-ost](https://github.com/github/gh-ost)
- [📖 pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)

### Articles de référence
- [📝 GitHub: How we migrate databases](https://github.blog/2020-02-14-automating-mysql-schema-migrations-with-github-actions-and-more/)
- [📝 Stripe: Online migrations at scale](https://stripe.com/blog/online-migrations)
- [📝 Shopify: Frictionless data migrations](https://shopify.engineering/frictionless-data-migrations-at-shopify)

### Outils
- [🔧 SQLFluff (SQL Linter)](https://www.sqlfluff.com/)
- [🔧 Alembic (Python ORM migrations)](https://alembic.sqlalchemy.org/)

---

## ➡️ Sections suivantes

**16.8.1 Flyway** : Installation, configuration, utilisation détaillée, intégration CI/CD, patterns avancés.

**16.8.2 Liquibase** : XML/YAML changelogs, rollback automatique, preconditions, enterprise features.

**16.8.3 gh-ost** : Configuration production, monitoring, troubleshooting, cas d'usage avancés.

**16.8.4 pt-online-schema-change** : Configuration détaillée, réplication, comparaison avec gh-ost.

---

**MariaDB** : Version 11.8 LTS

⏭️ [Flyway](/16-devops-automatisation/08.1-flyway.md)

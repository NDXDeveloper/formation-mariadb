🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.1 Migration depuis MySQL

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Administration MySQL/MariaDB, connaissance des mécanismes de réplication, expérience en gestion de production

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Évaluer précisément le niveau de compatibilité entre votre version MySQL et MariaDB cible
- Identifier les fonctionnalités divergentes nécessitant une adaptation
- Choisir la stratégie de migration optimale selon votre contexte (dump, réplication, physique)
- Anticiper les pièges courants et préparer les solutions de contournement
- Planifier une migration MySQL → MariaDB avec un risque minimal

---

## Introduction

La migration de MySQL vers MariaDB est souvent présentée comme triviale : "c'est un drop-in replacement, il suffit de changer le binaire". Cette vision simpliste a conduit à de nombreuses migrations ratées, des applications cassées en production, et des rollbacks d'urgence le week-end.

La réalité est plus nuancée. MariaDB **est** hautement compatible avec MySQL — probablement le SGBD le plus compatible qui existe. Mais "hautement compatible" ne signifie pas "identique". Les deux projets ont divergé significativement depuis le fork de 2009, chacun développant ses propres fonctionnalités, optimisations, et parfois des comportements subtilement différents pour les mêmes commandes SQL.

Cette section pose les fondations d'une migration réussie en explorant l'histoire de cette compatibilité, ses limites actuelles, et la méthodologie pour naviguer sereinement entre les deux systèmes.

---

## Pourquoi migrer de MySQL vers MariaDB ?

Avant de plonger dans le "comment", examinons le "pourquoi". Les motivations d'une migration MySQL → MariaDB sont diverses et souvent cumulatives.

### Motivations stratégiques

**Indépendance vis-à-vis d'Oracle**

Depuis le rachat de Sun Microsystems (et donc MySQL) par Oracle en 2010, de nombreuses entreprises s'interrogent sur la pérennité du projet open source MySQL. Les préoccupations incluent :

- Évolution de la licence (MySQL Enterprise vs Community)
- Rythme de développement des fonctionnalités open source
- Support long terme des versions Community
- Coût croissant des licences Enterprise

MariaDB, gouverné par la MariaDB Foundation, offre une alternative avec une gouvernance communautaire transparente et un engagement fort envers l'open source.

**Fonctionnalités avancées**

MariaDB a développé des fonctionnalités absentes ou arrivées tardivement dans MySQL :

| Fonctionnalité | MariaDB | MySQL |
|----------------|---------|-------|
| System-Versioned Tables | 10.3+ (2018) | Non disponible |
| Sequences | 10.3+ (2018) | Non disponible |
| Window Functions | 10.2+ (2017) | 8.0+ (2018) |
| JSON natif | 10.2+ | 5.7+ |
| Colonnes invisibles | 10.3+ | 8.0.23+ |
| Oracle PL/SQL mode | 10.3+ | Non disponible |
| **MariaDB Vector** 🆕 | 11.7+ | Non disponible |
| Thread Pool (Community) | Toutes versions | Enterprise uniquement |

**Performance et optimisations**

MariaDB intègre des optimisations de l'optimiseur de requêtes souvent plus agressives :

```sql
-- Exemple : Optimisation des sous-requêtes
-- MariaDB transforme automatiquement certaines sous-requêtes en semi-joins
SELECT * FROM orders 
WHERE customer_id IN (SELECT id FROM customers WHERE country = 'FR');

-- MariaDB applique également des optimisations sur les dérivées tables
-- et propose le "condition pushdown" plus avancé
```

### Motivations techniques

**Moteurs de stockage additionnels**

MariaDB propose des moteurs absents de MySQL Community :

- **Aria** : Remplacement crash-safe de MyISAM
- **ColumnStore** : Analytique colonaire distribuée
- **Spider** : Sharding natif
- **S3** : Archivage sur object storage
- **CONNECT** : Accès à des sources externes (CSV, JSON, ODBC...)

**Réplication améliorée**

- GTID implémenté différemment (plus simple à gérer)
- Parallel replication plus mature
- Réplication multi-source native

---

## Historique de la compatibilité MySQL/MariaDB

Comprendre l'évolution de la relation entre MySQL et MariaDB aide à anticiper les zones de friction.

### Chronologie des divergences

```
2009    │ Fork initial - MariaDB 5.1 = MySQL 5.1
        │ Compatibilité : 100%
        │
2010    │ MariaDB 5.2/5.3 - Ajouts propriétaires (Virtual Columns, Aria)
        │ Compatibilité : ~99%
        │
2012    │ MariaDB 5.5 ≈ MySQL 5.5
        │ Compatibilité : ~98%
        │
2013    │ MariaDB 10.0 - Divergence majeure
        │ │ • GTID incompatible
        │ │ • Réplication différente
        │ Compatibilité : ~95%
        │
2015    │ MariaDB 10.1 vs MySQL 5.7
        │ │ • Encryption at rest (implémentation différente)
        │ │ • JSON (syntaxe compatible, stockage différent)
        │ Compatibilité : ~92%
        │
2018    │ MariaDB 10.3 vs MySQL 8.0
        │ │ • CTEs récursifs (syntaxe compatible)
        │ │ • Window Functions (compatible)
        │ │ • Instant DDL (implémentations différentes)
        │ Compatibilité : ~90%
        │
2023    │ MariaDB 11.x vs MySQL 8.x
        │ │ • Fonctionnalités exclusives des deux côtés
        │ │ • Optimiseur divergent
        │ Compatibilité SQL : ~88%
        │
2025    │ MariaDB 11.8 LTS 🆕
        │ │ • utf8mb4 par défaut
        │ │ • Vector search (exclusif)
        │ │ • Format TIMESTAMP étendu
        │ Compatibilité SQL : ~85-90%
```

### Ce qui reste compatible

Malgré les divergences, le cœur SQL reste hautement compatible :

✅ **Syntaxe DML standard** : SELECT, INSERT, UPDATE, DELETE  
✅ **Syntaxe DDL de base** : CREATE/ALTER/DROP TABLE, INDEX, VIEW  
✅ **Types de données courants** : INT, VARCHAR, TEXT, DATETIME, DECIMAL  
✅ **Contraintes** : PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK  
✅ **Jointures** : INNER, LEFT, RIGHT, CROSS JOIN  
✅ **Sous-requêtes** : Scalaires, IN, EXISTS, dérivées  
✅ **Fonctions SQL standard** : Chaînes, dates, mathématiques  
✅ **Procédures stockées** : Syntaxe de base compatible  
✅ **Triggers et Events** : Syntaxe identique  
✅ **Protocole client** : Connecteurs MySQL fonctionnels  

### Ce qui diverge

⚠️ **GTID** : Implémentation complètement différente  
⚠️ **JSON** : Fonctions compatibles, stockage interne différent  
⚠️ **Authentification** : Plugins par défaut différents (MySQL 8.0 : caching_sha2_password)  
⚠️ **Encryption** : Configuration et key management différents  
⚠️ **Group Replication** : Spécifique MySQL, équivalent MariaDB = Galera  
⚠️ **MySQL Shell** : Non compatible avec MariaDB  
⚠️ **Clone Plugin** : Spécifique MySQL  
⚠️ **Instant DDL** : Capacités et syntaxe légèrement différentes  

---

## Niveaux de migration

La migration MySQL → MariaDB peut s'envisager à différents niveaux de profondeur.

### Niveau 1 : Drop-in replacement

**Contexte** : Application simple, pas de fonctionnalités avancées MySQL, même version majeure.

```bash
# Arrêt MySQL
systemctl stop mysql

# Installation MariaDB (remplace MySQL)
apt install mariadb-server

# Démarrage - utilise les mêmes fichiers de données
systemctl start mariadb

# Upgrade des tables système
mariadb-upgrade
```

💡 **Conseil** : Ce niveau ne fonctionne que pour des migrations MySQL 5.7 → MariaDB 10.x sur des bases simples. Pour MySQL 8.0 ou des configurations avancées, préférez une migration logique.

### Niveau 2 : Migration logique

**Contexte** : Versions différentes, configurations spécifiques, besoin de validation.

```bash
# Export depuis MySQL
mysqldump --single-transaction --routines --triggers \
          --set-gtid-purged=OFF \
          --all-databases > full_backup.sql

# Pré-traitement si nécessaire
sed -i 's/utf8mb4_0900_ai_ci/utf8mb4_unicode_ci/g' full_backup.sql

# Import dans MariaDB
mariadb < full_backup.sql
```

### Niveau 3 : Migration avec réplication

**Contexte** : Zero-downtime requis, validation en production, rollback facilité.

```
┌─────────────┐         ┌─────────────┐
│   MySQL     │ ──────▶ │  MariaDB    │
│   (Source)  │  Repli- │  (Replica)  │
│             │  cation │             │
└─────────────┘         └─────────────┘
      │                       │
      │                       │
      ▼                       ▼
┌─────────────┐         ┌─────────────┐
│ Application │         │ Application │
│  (Active)   │         │  (Standby)  │
└─────────────┘         └─────────────┘

        Phase 1: Réplication MySQL → MariaDB
        Phase 2: Bascule applicative
        Phase 3: MariaDB devient source
```

---

## Matrice de compatibilité par version

Cette matrice détaille la compatibilité entre versions MySQL et MariaDB pour guider votre choix de version cible.

### MySQL 5.6 → MariaDB

| Version MariaDB cible | Compatibilité | Recommandation |
|-----------------------|---------------|----------------|
| 10.1 | ⭐⭐⭐⭐⭐ 98% | ✅ Recommandé |
| 10.2 | ⭐⭐⭐⭐⭐ 97% | ✅ Recommandé |
| 10.3 | ⭐⭐⭐⭐ 95% | ✅ Acceptable |
| 10.4+ | ⭐⭐⭐⭐ 93% | ⚠️ Tests approfondis |

### MySQL 5.7 → MariaDB

| Version MariaDB cible | Compatibilité | Recommandation |
|-----------------------|---------------|----------------|
| 10.2 | ⭐⭐⭐⭐⭐ 96% | ✅ Recommandé |
| 10.3 | ⭐⭐⭐⭐⭐ 95% | ✅ Recommandé |
| 10.4 | ⭐⭐⭐⭐ 94% | ✅ Recommandé |
| 10.5 | ⭐⭐⭐⭐ 93% | ✅ Acceptable |
| 10.6 LTS | ⭐⭐⭐⭐ 92% | ✅ Recommandé (LTS) |
| 11.4 LTS | ⭐⭐⭐ 88% | ⚠️ Tests approfondis |
| 11.8 LTS 🆕 | ⭐⭐⭐ 85% | ⚠️ Tests approfondis |

### MySQL 8.0 → MariaDB

| Version MariaDB cible | Compatibilité | Recommandation |
|-----------------------|---------------|----------------|
| 10.4 | ⭐⭐⭐ 82% | ⚠️ Nombreuses adaptations |
| 10.5 | ⭐⭐⭐ 83% | ⚠️ Nombreuses adaptations |
| 10.6 LTS | ⭐⭐⭐ 85% | ⚠️ Tests critiques |
| 11.4 LTS | ⭐⭐⭐ 86% | ⚠️ Tests critiques |
| 11.8 LTS 🆕 | ⭐⭐⭐ 87% | ⚠️ Tests critiques |

⚠️ **Attention MySQL 8.0** : La migration depuis MySQL 8.0 est plus complexe en raison de :
- Nouveau système d'authentification par défaut (`caching_sha2_password`)
- Collations `utf8mb4_0900_*` non supportées par MariaDB
- Fonctionnalités JSON étendues partiellement compatibles
- Expressions de table communes (CTE) avec syntaxe légèrement différente
- `INVISIBLE` indexes avec comportement différent

---

## Inventaire pré-migration

Avant toute migration, un inventaire exhaustif est indispensable.

### Script d'audit MySQL

```sql
-- ============================================
-- AUDIT PRÉ-MIGRATION MYSQL → MARIADB
-- ============================================

-- 1. Version et configuration
SELECT @@version AS mysql_version, 
       @@version_comment AS edition,
       @@datadir AS data_directory,
       @@character_set_server AS default_charset,
       @@collation_server AS default_collation;

-- 2. Taille totale des données
SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS total_size_gb,
    COUNT(DISTINCT table_schema) AS database_count,
    COUNT(*) AS table_count
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 3. Moteurs de stockage utilisés
SELECT 
    engine,
    COUNT(*) AS table_count,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY engine
ORDER BY size_mb DESC;

-- 4. Collations utilisées (attention aux _0900_)
SELECT 
    collation_name,
    COUNT(*) AS column_count
FROM information_schema.columns
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
  AND collation_name IS NOT NULL
GROUP BY collation_name
ORDER BY column_count DESC;

-- 5. Fonctionnalités MySQL 8.0 spécifiques
-- CHECK constraints (supportés par MariaDB mais syntaxe à vérifier)
SELECT 
    table_schema,
    table_name,
    constraint_name,
    check_clause
FROM information_schema.check_constraints
WHERE constraint_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 6. Colonnes JSON
SELECT 
    table_schema,
    table_name,
    column_name,
    column_type
FROM information_schema.columns
WHERE data_type = 'json'
  AND table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 7. Generated columns
SELECT 
    table_schema,
    table_name,
    column_name,
    generation_expression,
    extra
FROM information_schema.columns
WHERE generation_expression IS NOT NULL
  AND table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 8. Procédures stockées et fonctions
SELECT 
    routine_schema,
    routine_type,
    COUNT(*) AS count
FROM information_schema.routines
WHERE routine_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY routine_schema, routine_type;

-- 9. Triggers
SELECT 
    trigger_schema,
    COUNT(*) AS trigger_count
FROM information_schema.triggers
WHERE trigger_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY trigger_schema;

-- 10. Events
SELECT 
    event_schema,
    event_name,
    status
FROM information_schema.events
WHERE event_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 11. Plugins actifs (certains peuvent ne pas exister dans MariaDB)
SELECT plugin_name, plugin_status, plugin_type
FROM information_schema.plugins
WHERE plugin_status = 'ACTIVE';

-- 12. Utilisateurs et authentification
SELECT 
    user,
    host,
    plugin AS auth_plugin,
    password_expired,
    account_locked
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema');
```

### Checklist des points d'attention

Après l'audit, vérifiez ces points critiques :

| Élément | Risque si présent | Action |
|---------|-------------------|--------|
| Collation `*_0900_*` | 🔴 Élevé | Conversion requise |
| Auth `caching_sha2_password` | 🔴 Élevé | Changement plugin |
| `JSON_TABLE()` | 🟡 Moyen | Réécriture possible |
| `MEMBER OF()` | 🟡 Moyen | Réécriture requise |
| Invisible indexes | 🟢 Faible | Syntaxe compatible |
| Generated columns | 🟢 Faible | Généralement compatible |
| Group Replication | 🔴 Élevé | Migration vers Galera |
| Clone Plugin usage | 🔴 Élevé | Alternative Mariabackup |
| MySQL Shell scripts | 🟡 Moyen | Réécriture |

---

## Stratégies de migration

### Stratégie 1 : Big Bang (Offline)

**Quand l'utiliser** : Fenêtre de maintenance disponible, base < 100 GB, risque acceptable.

```
T-2h    : Annonce maintenance
T-1h    : Dernière sauvegarde MySQL
T0      : Arrêt applications
T+10min : Export mysqldump
T+1-3h  : Import MariaDB
T+3h    : Tests de validation
T+4h    : Démarrage applications
T+4h30  : Fin maintenance
```

**Avantages** : Simple, prévisible, rollback clair  
**Inconvénients** : Downtime significatif

### Stratégie 2 : Réplication puis bascule (Online)

**Quand l'utiliser** : Zero-downtime requis, base importante, besoin de validation en production.

```
Jour J-7  : Setup réplication MySQL → MariaDB
Jour J-7  : Synchronisation initiale
J-7 à J-1 : Réplication continue, tests sur replica
Jour J    : Bascule applicative (< 1 min downtime)
J+1 à J+7 : Surveillance, réplication inverse possible
Jour J+7  : Décommissionnement MySQL
```

```sql
-- Configuration MySQL (source)
-- my.cnf
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
gtid_mode = ON                    -- Si GTID MySQL utilisé
enforce_gtid_consistency = ON

-- Configuration MariaDB (replica)
-- Attention : GTID MariaDB != GTID MySQL
-- Utiliser position binlog classique pour la réplication cross-platform

CHANGE MASTER TO
    MASTER_HOST = 'mysql-source.example.com',
    MASTER_USER = 'replication_user',
    MASTER_PASSWORD = 'secure_password',
    MASTER_LOG_FILE = 'mysql-bin.000042',
    MASTER_LOG_POS = 12345678;

START SLAVE;
SHOW SLAVE STATUS\G
```

⚠️ **Attention GTID** : La réplication basée sur GTID entre MySQL et MariaDB ne fonctionne **pas** directement car les formats GTID sont incompatibles. Utilisez la réplication classique par position binlog.

### Stratégie 3 : Blue-Green avec double écriture

**Quand l'utiliser** : Applications critiques, validation fonctionnelle complète requise, rollback instantané nécessaire.

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│                    (ProxySQL/HAProxy)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
         ┌──────────────┴──────────────┐
         │                             │
         ▼                             ▼
┌─────────────────┐           ┌─────────────────┐
│   MySQL (Blue)  │           │ MariaDB (Green) │
│    ACTIVE       │◀─────────▶│    STANDBY      │
│                 │   Sync    │                 │
└─────────────────┘           └─────────────────┘

Phase 1: Double write vers les deux bases
Phase 2: Lectures progressives vers MariaDB
Phase 3: 100% trafic vers MariaDB
Phase 4: MySQL en standby puis décommissionné
```

---

## Outils de migration

### Outils natifs

#### mysqldump / mariadb-dump

L'outil classique, fiable et universel.

```bash
# Export optimisé pour migration
mysqldump \
    --single-transaction \           # Cohérence sans lock (InnoDB)
    --routines \                     # Inclut procédures/fonctions
    --triggers \                     # Inclut triggers
    --events \                       # Inclut events
    --set-gtid-purged=OFF \          # Évite problèmes GTID
    --column-statistics=0 \          # MySQL 8.0 : désactive stats
    --skip-lock-tables \             # Pas de LOCK TABLES
    --quick \                        # Streaming, moins de RAM
    --hex-blob \                     # Binaires en hexa (portable)
    --default-character-set=utf8mb4 \
    --databases db1 db2 db3 \
    | gzip > migration_$(date +%Y%m%d).sql.gz
```

#### mydumper / myloader

Export/import parallélisé, idéal pour les grosses bases.

```bash
# Installation
apt install mydumper

# Export parallèle
mydumper \
    --host=mysql-source \
    --user=backup_user \
    --password=secret \
    --outputdir=/backup/migration \
    --threads=8 \
    --compress \
    --triggers \
    --events \
    --routines \
    --rows=500000               # Split par chunks

# Import parallèle dans MariaDB
myloader \
    --host=mariadb-target \
    --user=root \
    --password=secret \
    --directory=/backup/migration \
    --threads=8 \
    --overwrite-tables
```

💡 **Conseil** : mydumper peut être 5-10x plus rapide que mysqldump sur les bases volumineuses grâce au parallélisme.

### Outils de synchronisation

#### pt-table-sync (Percona Toolkit)

Synchronisation fine des différences.

```bash
# Comparer les données entre MySQL et MariaDB
pt-table-checksum \
    --host=mysql-source \
    --databases=mydb \
    --replicate=percona.checksums

# Synchroniser les différences trouvées
pt-table-sync \
    --execute \
    --sync-to-master \
    h=mariadb-target \
    --databases=mydb
```

#### pt-archiver

Migration progressive de données historiques.

```bash
# Migrer les vieilles données par batch
pt-archiver \
    --source h=mysql-source,D=mydb,t=orders \
    --dest h=mariadb-target,D=mydb,t=orders \
    --where "created_at < '2024-01-01'" \
    --limit=10000 \
    --commit-each \
    --progress=10000
```

---

## Gestion des incompatibilités courantes

### Collations MySQL 8.0

MySQL 8.0 introduit les collations `utf8mb4_0900_*` non supportées par MariaDB.

```sql
-- Identifier les collations problématiques
SELECT DISTINCT collation_name 
FROM information_schema.columns 
WHERE collation_name LIKE '%0900%';

-- Mapping des collations
-- utf8mb4_0900_ai_ci → utf8mb4_unicode_ci (ou utf8mb4_uca1400_ai_ci en 11.8)
-- utf8mb4_0900_as_cs → utf8mb4_bin

-- Script de conversion avant import
sed -i 's/utf8mb4_0900_ai_ci/utf8mb4_unicode_ci/g' dump.sql
sed -i 's/utf8mb4_0900_as_cs/utf8mb4_bin/g' dump.sql
```

🆕 **MariaDB 11.8** : Supporte les collations UCA 14.0.0 (`utf8mb4_uca1400_*`) qui sont plus proches des collations MySQL 8.0 en termes de tri Unicode.

### Authentification MySQL 8.0

```sql
-- Vérifier le plugin d'authentification actuel
SELECT user, host, plugin FROM mysql.user;

-- Convertir vers mysql_native_password avant migration
ALTER USER 'app_user'@'%' 
IDENTIFIED WITH mysql_native_password BY 'password';

-- Ou dans MariaDB, utiliser ed25519 (plus sécurisé)
-- après migration :
ALTER USER 'app_user'@'%' 
IDENTIFIED VIA ed25519 USING PASSWORD('password');
```

### Fonctions JSON incompatibles

```sql
-- MySQL 8.0 : JSON_TABLE (non supporté MariaDB < 10.6)
SELECT * FROM JSON_TABLE(
    '[{"a":1},{"a":2}]',
    '$[*]' COLUMNS (a INT PATH '$.a')
) AS jt;

-- Alternative MariaDB : JSON_EXTRACT avec UNION ou procédure stockée
SELECT JSON_EXTRACT(json_col, '$[0].a') AS a FROM my_table
UNION ALL
SELECT JSON_EXTRACT(json_col, '$[1].a') AS a FROM my_table;

-- MySQL 8.0 : MEMBER OF()
SELECT * FROM t WHERE 'value' MEMBER OF(json_array_col);

-- Alternative MariaDB :
SELECT * FROM t WHERE JSON_CONTAINS(json_array_col, '"value"');
```

### Expression par défaut

```sql
-- MySQL 8.0 permet les expressions complexes en DEFAULT
CREATE TABLE t (
    id INT,
    data JSON DEFAULT (JSON_OBJECT())  -- OK MySQL 8.0
);

-- MariaDB : utiliser un trigger ou valeur NULL
CREATE TABLE t (
    id INT,
    data JSON DEFAULT NULL
);

DELIMITER //
CREATE TRIGGER t_before_insert BEFORE INSERT ON t
FOR EACH ROW
BEGIN
    IF NEW.data IS NULL THEN
        SET NEW.data = JSON_OBJECT();
    END IF;
END //
DELIMITER ;
```

---

## Validation post-migration

### Tests de cohérence des données

```sql
-- Compter les enregistrements par table
-- À exécuter sur MySQL et MariaDB, comparer les résultats

SELECT 
    table_schema,
    table_name,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY table_name;

-- Checksum des tables (attention : peut varier selon moteur/version)
CHECKSUM TABLE mydb.customers, mydb.orders, mydb.products;
```

### Tests fonctionnels

```bash
#!/bin/bash
# Script de validation post-migration

MYSQL_HOST="mysql-old.example.com"
MARIA_HOST="mariadb-new.example.com"
DB="production_db"

# Test 1 : Connexion
echo "Test connexion MariaDB..."
mariadb -h $MARIA_HOST -e "SELECT 1" && echo "✓ Connexion OK"

# Test 2 : Comparaison compte de lignes
for TABLE in customers orders products; do
    MYSQL_COUNT=$(mysql -h $MYSQL_HOST -N -e "SELECT COUNT(*) FROM $DB.$TABLE")
    MARIA_COUNT=$(mariadb -h $MARIA_HOST -N -e "SELECT COUNT(*) FROM $DB.$TABLE")
    
    if [ "$MYSQL_COUNT" -eq "$MARIA_COUNT" ]; then
        echo "✓ $TABLE : $MYSQL_COUNT lignes (identique)"
    else
        echo "✗ $TABLE : MySQL=$MYSQL_COUNT, MariaDB=$MARIA_COUNT"
    fi
done

# Test 3 : Procédures stockées
echo "Test procédures stockées..."
mariadb -h $MARIA_HOST -e "CALL mydb.process_order(12345)" && echo "✓ SP OK"

# Test 4 : Performance (requête critique)
echo "Test performance requête critique..."
time mariadb -h $MARIA_HOST -e "
    SELECT c.name, SUM(o.total) 
    FROM customers c 
    JOIN orders o ON c.id = o.customer_id 
    WHERE o.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
    GROUP BY c.id
    LIMIT 100
" > /dev/null
```

---

## Scénarios réels

### Scénario 1 : E-commerce MySQL 5.7 → MariaDB 10.6 LTS

**Contexte** : 
- Base de 200 GB, 500 tables
- 50 procédures stockées
- Application PHP/Laravel
- Fenêtre de maintenance : 4 heures max

**Approche retenue** : Migration logique avec mydumper

```bash
# Phase 1 : Préparation (J-7)
# Audit complet, tests en staging

# Phase 2 : Export (J-Day, T0)
mydumper -h mysql-prod -t 16 -o /backup/migration

# Phase 3 : Import (T+30min à T+2h30)
myloader -h mariadb-new -t 16 -d /backup/migration

# Phase 4 : Validation (T+2h30 à T+3h30)
./validate_migration.sh

# Phase 5 : Bascule DNS (T+3h30)
# TTL déjà réduit à 60s une semaine avant
```

**Résultat** : Migration réussie en 3h15, aucun rollback nécessaire.

### Scénario 2 : SaaS MySQL 8.0 → MariaDB 11.4 LTS (Zero-downtime)

**Contexte** :
- Base de 2 TB, haute criticité
- SLA 99.99%
- MySQL 8.0.32 avec caching_sha2_password
- Nombreuses collations utf8mb4_0900_ai_ci

**Approche retenue** : Réplication + Blue-Green

```bash
# Phase 1 : Préparation (J-14)
# - Changement auth plugin vers mysql_native_password
# - Création replica MariaDB avec réplication binlog

# Phase 2 : Synchronisation (J-14 à J-1)
# - Monitoring lag de réplication < 1s
# - Tests de charge sur replica MariaDB
# - Conversion collations dans les dumps de config

# Phase 3 : Pre-cutover (J-1)
# - Activation ProxySQL devant les deux backends
# - Configuration read/write split

# Phase 4 : Cutover progressif (J-Day)
# - 10% lectures vers MariaDB
# - 50% lectures vers MariaDB
# - 100% lectures vers MariaDB  
# - Writes vers MariaDB (< 30s downtime écritures)

# Phase 5 : Validation (J+1 à J+7)
# - MySQL maintenu en standby
# - Monitoring intensif
```

**Résultat** : Migration avec 23 secondes d'indisponibilité en écriture, rollback jamais activé.

---

## ✅ Points clés à retenir

- La migration MySQL → MariaDB est **hautement compatible** mais pas triviale : chaque version a ses spécificités
- **MySQL 5.7** migre plus facilement que **MySQL 8.0** vers MariaDB en raison des nouvelles fonctionnalités MySQL 8.0
- Les **collations `utf8mb4_0900_*`** de MySQL 8.0 nécessitent une conversion systématique
- Le **GTID MySQL** est incompatible avec le **GTID MariaDB** : utilisez la réplication par position binlog
- **mydumper/myloader** offre des performances 5-10x supérieures à mysqldump sur les grosses bases
- Un **inventaire exhaustif** pré-migration évite 90% des surprises
- Prévoyez toujours un **plan de rollback testé** avant toute migration production
- La stratégie **réplication + bascule** permet des migrations quasi zero-downtime

---

## 🔗 Ressources et références

- [📖 MariaDB KB : MariaDB vs MySQL Compatibility](https://mariadb.com/kb/en/mariadb-vs-mysql-compatibility/)
- [📖 MariaDB KB : Migrating to MariaDB from MySQL](https://mariadb.com/kb/en/migrating-to-mariadb-from-mysql/)
- [📖 MySQL to MariaDB Migration Guide](https://mariadb.com/resources/blog/how-to-migrate-from-mysql-to-mariadb/)
- [🔧 mydumper Documentation](https://github.com/mydumper/mydumper)
- [🔧 Percona Toolkit](https://docs.percona.com/percona-toolkit/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-release-notes/)

---

## ➡️ Section suivante

**[19.1.1 Compatibilité MySQL/MariaDB](./01.1-compatibilite-mysql-mariadb.md)** : Nous approfondirons les détails de compatibilité SQL entre MySQL et MariaDB, avec un focus sur les différences syntaxiques, les fonctions divergentes, et les comportements subtils qui peuvent impacter vos applications.

⏭️ [Compatibilité MySQL/MariaDB](/19-migration-compatibilite/01.1-compatibilite-mysql-mariadb.md)

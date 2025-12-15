🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.4 Stratégies de mise à jour et upgrade paths

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Administration MariaDB, gestion des sauvegardes (chapitre 12), compréhension des versions LTS/Rolling (section 19.3)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Choisir la stratégie de mise à jour adaptée à votre contexte (in-place vs logical)
- Maîtriser l'utilisation de `mariadb-upgrade` et ses options
- Planifier les chemins de migration entre versions majeures
- Gérer les mises à jour mineures et les patches de sécurité
- Automatiser les processus d'upgrade de manière sécurisée
- Anticiper et résoudre les problèmes courants lors des mises à jour

---

## Introduction

Mettre à jour une base de données en production est une opération délicate qui requiert préparation, méthode et une bonne dose de prudence. Une mise à jour mal préparée peut entraîner des corruptions de données, des incompatibilités applicatives, ou des temps d'arrêt prolongés.

MariaDB offre plusieurs stratégies de mise à jour, chacune avec ses avantages et contraintes. Le choix de la stratégie dépend de nombreux facteurs : criticité du système, fenêtre de maintenance disponible, écart de version, complexité du schéma, et tolérance au risque.

Cette section présente les différentes approches de mise à jour, leurs conditions d'application, et les bonnes pratiques pour réussir vos upgrades MariaDB.

---

## Taxonomie des mises à jour

### Types de mises à jour

Les mises à jour MariaDB se classifient selon leur amplitude :

```
Types de mises à jour MariaDB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PATCH UPDATE (11.8.1 → 11.8.2)
├── Contenu : Correctifs sécurité, bugs critiques
├── Risque : Très faible
├── Méthode : In-place, redémarrage simple
└── Downtime : Minutes

MINOR UPDATE (11.8.x → 11.8.y)
├── Contenu : Bug fixes, optimisations mineures
├── Risque : Faible
├── Méthode : In-place + mariadb-upgrade
└── Downtime : Minutes à dizaines de minutes

MAJOR UPDATE (11.4 → 11.8)
├── Contenu : Nouvelles fonctionnalités, changements comportement
├── Risque : Modéré à élevé
├── Méthode : In-place ou Logical selon contexte
└── Downtime : Dizaines de minutes à heures

CROSS-GENERATION UPDATE (10.x → 11.x)
├── Contenu : Changements architecturaux possibles
├── Risque : Élevé
├── Méthode : Logical recommandé
└── Downtime : Heures
```

### Matrice de risque par type de mise à jour

| Type | Exemple | Risque données | Risque compat. | Rollback | Temps estimé |
|------|---------|----------------|----------------|----------|--------------|
| **Patch** | 11.8.1→11.8.2 | 🟢 Négligeable | 🟢 Négligeable | Facile | < 5 min |
| **Minor** | 11.8.0→11.8.5 | 🟢 Très faible | 🟢 Très faible | Facile | 5-15 min |
| **Major LTS** | 11.4→11.8 | 🟡 Faible | 🟡 Modéré | Modéré | 30 min - 2h |
| **Cross-gen** | 10.6→11.8 | 🟡 Modéré | 🔴 Élevé | Complexe | 1-4h |
| **Legacy** | 10.3→11.8 | 🔴 Élevé | 🔴 Élevé | Très complexe | 2-8h |

---

## Stratégies de mise à jour

### Stratégie 1 : Mise à jour In-Place

La mise à jour in-place consiste à remplacer les binaires MariaDB sur le même serveur, en conservant les fichiers de données.

```
Mise à jour In-Place
━━━━━━━━━━━━━━━━━━━━

AVANT                              APRÈS
┌─────────────────────┐           ┌─────────────────────┐
│ MariaDB 11.4.2      │           │ MariaDB 11.8.1      │
│                     │           │                     │
│ ┌─────────────────┐ │           │ ┌─────────────────┐ │
│ │ Binaires 11.4   │ │  ──────▶  │ │ Binaires 11.8   │ │
│ └─────────────────┘ │  Upgrade  │ └─────────────────┘ │
│                     │           │                     │
│ ┌─────────────────┐ │           │ ┌─────────────────┐ │
│ │ Data files      │ │  ══════▶  │ │ Data files      │ │
│ │ (conservés)     │ │  (même)   │ │ (mis à jour)    │ │
│ └─────────────────┘ │           │ └─────────────────┘ │
└─────────────────────┘           └─────────────────────┘
```

**Processus type :**

```bash
#!/bin/bash
# Script de mise à jour in-place MariaDB

set -e

# Variables
OLD_VERSION="11.4"
NEW_VERSION="11.8"
BACKUP_DIR="/backup/pre-upgrade-$(date +%Y%m%d)"

echo "=== Mise à jour in-place MariaDB $OLD_VERSION → $NEW_VERSION ==="

# 1. Vérification pré-upgrade
echo "[1/7] Vérification pré-upgrade..."
mariadb -e "SELECT VERSION();"
mariadb -e "CHECK TABLE mysql.user, mysql.db, mysql.tables_priv;"

# 2. Sauvegarde complète
echo "[2/7] Sauvegarde complète..."
mkdir -p $BACKUP_DIR
mariadb-dump --all-databases --routines --triggers --events \
    --single-transaction > $BACKUP_DIR/full_backup.sql
mariabackup --backup --target-dir=$BACKUP_DIR/physical

# 3. Arrêt du service
echo "[3/7] Arrêt de MariaDB..."
systemctl stop mariadb

# 4. Mise à jour des paquets
echo "[4/7] Mise à jour des paquets..."
# Pour Debian/Ubuntu :
apt update
apt install mariadb-server mariadb-client -y
# Pour RHEL/CentOS :
# dnf update mariadb-server mariadb -y

# 5. Démarrage du service
echo "[5/7] Démarrage de MariaDB..."
systemctl start mariadb

# 6. Exécution de mariadb-upgrade
echo "[6/7] Exécution de mariadb-upgrade..."
mariadb-upgrade --force

# 7. Vérification post-upgrade
echo "[7/7] Vérification post-upgrade..."
mariadb -e "SELECT VERSION();"
mariadb -e "SHOW WARNINGS;"

echo "=== Mise à jour terminée avec succès ==="
```

**Avantages :**
- ✅ Rapide (pas de transfert de données)
- ✅ Simple à exécuter
- ✅ Downtime minimal
- ✅ Conserve les configurations

**Inconvénients :**
- ❌ Rollback complexe (nécessite restauration)
- ❌ Risque si fichiers de données corrompus
- ❌ Pas adapté aux grands sauts de version
- ❌ Même matériel (pas d'upgrade infra simultané)

**Quand utiliser :**
- Mises à jour patch et minor
- Mises à jour major entre versions LTS consécutives
- Environnements avec fenêtre de maintenance suffisante
- Bases de taille modérée (< 500 GB)

### Stratégie 2 : Migration Logique (Dump/Restore)

La migration logique exporte les données en SQL puis les réimporte dans une nouvelle instance.

```
Migration Logique
━━━━━━━━━━━━━━━━━

┌─────────────────────┐         ┌─────────────────────┐
│ MariaDB 10.6 LTS    │         │ MariaDB 11.8 LTS    │
│ (Source)            │         │ (Cible)             │
│                     │         │                     │
│ ┌─────────────────┐ │         │ ┌─────────────────┐ │
│ │ Data            │ │ ──────▶ │ │ Data            │ │
│ └─────────────────┘ │  Dump   │ └─────────────────┘ │
│                     │   SQL   │                     │
└─────────────────────┘         └─────────────────────┘
         │                               ▲
         │                               │
         ▼                               │
    ┌─────────┐                          │
    │  .sql   │ ─────────────────────────┘
    │  file   │        Import
    └─────────┘
```

**Processus type :**

```bash
#!/bin/bash
# Script de migration logique MariaDB

set -e

# Variables
SOURCE_HOST="mariadb-old.example.com"
TARGET_HOST="mariadb-new.example.com"
DUMP_FILE="/backup/migration_$(date +%Y%m%d).sql"

echo "=== Migration logique MariaDB ==="

# 1. Export depuis la source
echo "[1/5] Export depuis $SOURCE_HOST..."
mariadb-dump -h $SOURCE_HOST \
    --all-databases \
    --routines \
    --triggers \
    --events \
    --single-transaction \
    --quick \
    --set-gtid-purged=OFF \
    --default-character-set=utf8mb4 \
    | gzip > ${DUMP_FILE}.gz

# 2. Pré-traitement si nécessaire
echo "[2/5] Pré-traitement du dump..."
# Exemple : correction de collations pour 11.8
zcat ${DUMP_FILE}.gz | \
    sed 's/utf8mb3/utf8mb4/g' | \
    gzip > ${DUMP_FILE}_processed.gz

# 3. Création de la base cible (si nouvelle instance)
echo "[3/5] Préparation de la cible..."
# mariadb -h $TARGET_HOST -e "SET GLOBAL innodb_buffer_pool_size = 8G;"

# 4. Import dans la cible
echo "[4/5] Import vers $TARGET_HOST..."
zcat ${DUMP_FILE}_processed.gz | mariadb -h $TARGET_HOST

# 5. Vérification
echo "[5/5] Vérification..."
mariadb -h $TARGET_HOST -e "
    SELECT table_schema, COUNT(*) as tables 
    FROM information_schema.tables 
    WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
    GROUP BY table_schema;"

echo "=== Migration logique terminée ==="
```

**Avantages :**
- ✅ Nettoyage des données (défragmentation)
- ✅ Changement de matériel possible
- ✅ Rollback simple (ancienne instance intacte)
- ✅ Adapté aux grands sauts de version
- ✅ Possibilité de modifier le schéma pendant la migration

**Inconvénients :**
- ❌ Plus long (export + import)
- ❌ Nécessite espace disque supplémentaire
- ❌ Downtime plus important
- ❌ Deux instances à gérer temporairement

**Quand utiliser :**
- Migrations cross-génération (10.x → 11.x)
- Changement de matériel simultané
- Bases avec beaucoup de fragmentation
- Besoin d'un rollback simple
- Restructuration du schéma pendant la migration

### Stratégie 3 : Migration par Réplication

La réplication permet une migration avec un downtime minimal en synchronisant les données en temps réel.

```
Migration par Réplication
━━━━━━━━━━━━━━━━━━━━━━━━━

Phase 1 : Setup réplication
┌─────────────────────┐         ┌─────────────────────┐
│ MariaDB 11.4 LTS    │ ──────▶ │ MariaDB 11.8 LTS    │
│ (Primary)           │ Binlog  │ (Replica)           │
│                     │ Replic. │                     │
└─────────────────────┘         └─────────────────────┘

Phase 2 : Synchronisation
┌─────────────────────┐         ┌─────────────────────┐
│ MariaDB 11.4 LTS    │ ══════▶ │ MariaDB 11.8 LTS    │
│ (Primary)           │ Sync    │ (Replica)           │
│ Writes ─────────────│─────────│──────────────▶      │
└─────────────────────┘         └─────────────────────┘

Phase 3 : Bascule
┌─────────────────────┐         ┌─────────────────────┐
│ MariaDB 11.4 LTS    │         │ MariaDB 11.8 LTS    │
│ (Ancien Primary)    │         │ (Nouveau Primary)   │
│ [STOP]              │         │ Writes ────────▶    │
└─────────────────────┘         └─────────────────────┘
```

**Configuration de la réplication cross-version :**

```sql
-- Sur la source (11.4 LTS)
-- Vérifier que le binlog est activé
SHOW VARIABLES LIKE 'log_bin';
SHOW MASTER STATUS;

-- Créer l'utilisateur de réplication
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- Sur la cible (11.8 LTS)
-- Configurer la réplication
CHANGE MASTER TO
    MASTER_HOST = 'mariadb-114.example.com',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'secure_password',
    MASTER_LOG_FILE = 'mariadb-bin.000042',
    MASTER_LOG_POS = 12345678;

START SLAVE;

-- Vérifier le statut
SHOW SLAVE STATUS\G
```

**Avantages :**
- ✅ Downtime minimal (secondes)
- ✅ Validation en production avant bascule
- ✅ Rollback instantané (inverser la réplication)
- ✅ Possibilité de tester les performances

**Inconvénients :**
- ❌ Complexité de mise en œuvre
- ❌ Nécessite deux instances pendant la transition
- ❌ Certaines fonctionnalités peuvent bloquer la réplication
- ❌ Lag potentiel pendant la synchronisation initiale

**Quand utiliser :**
- SLA exigeant (< 1 min downtime)
- Bases volumineuses (> 1 TB)
- Besoin de validation en conditions réelles
- Migration vers nouvelle infrastructure

### Stratégie 4 : Blue-Green Deployment

Le déploiement blue-green maintient deux environnements complets, permettant une bascule instantanée.

```
Blue-Green Deployment
━━━━━━━━━━━━━━━━━━━━━

                    ┌─────────────────────────────┐
                    │        Load Balancer        │
                    │     (HAProxy/ProxySQL)      │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             │             ▼
           ┌──────────────┐       │      ┌──────────────┐
           │    BLUE      │       │      │    GREEN     │
           │  (Active)    │       │      │  (Standby)   │
           │              │       │      │              │
           │ MariaDB 11.4 │       │      │ MariaDB 11.8 │
           │              │◀──────┴─────▶│              │
           │              │    Sync      │              │
           └──────────────┘              └──────────────┘

Bascule : Changement de routage Load Balancer
          BLUE (Active) ──▶ GREEN (Active)
          GREEN (Standby) ◀── BLUE (Standby)
```

**Avantages :**
- ✅ Rollback instantané (rebascule)
- ✅ Tests complets en production shadow
- ✅ Zero downtime possible
- ✅ Isolation complète des environnements

**Inconvénients :**
- ❌ Coût (double infrastructure)
- ❌ Synchronisation bidirectionnelle complexe
- ❌ Gestion des données créées post-bascule si rollback

---

## Chemins de migration (Upgrade Paths)

### Chemins supportés

MariaDB définit des chemins de migration officiellement supportés :

```
Chemins de migration MariaDB supportés
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                         ┌─────────────────────────────────────┐
                         │           MariaDB 11.8 LTS 🆕       │
                         │         (Version cible actuelle)    │
                         └───────────────────┬─────────────────┘
                                             ▲
           ┌─────────────────────────────────┼─────────────────────────────────┐
           │                                 │                                 │
           │                                 │                                 │
    ┌──────┴──────┐                   ┌──────┴──────┐                   ┌──────┴──────┐
    │ 11.4 LTS    │                   │ 10.11 LTS   │                   │ 10.6 LTS    │
    │   Direct    │                   │   Direct    │                   │   Direct    │
    │     ✅      │                   │     ✅      │                   │     ✅      │
    └─────────────┘                   └──────┬──────┘                   └──────┬──────┘
                                             │                                 │
                                             │                                 │
                                      ┌──────┴──────┐                   ┌──────┴──────┐
                                      │ 10.5        │                   │ 10.4        │
                                      │ Via 10.6    │                   │ Via 10.6    │
                                      │     ⚠️      │                   │     ⚠️      │
                                      └─────────────┘                   └─────────────┘
                                             │
                                             │
                                      ┌──────┴──────┐
                                      │ 10.3/10.2   │
                                      │ Multi-step  │
                                      │     ⚠️      │
                                      └─────────────┘
```

### Matrice des chemins de migration

| Version source | Vers 11.8 LTS | Méthode recommandée | Notes |
|----------------|---------------|---------------------|-------|
| **11.4 LTS** | ✅ Direct | In-place | Chemin privilégié |
| **10.11 LTS** | ✅ Direct | In-place ou Logical | Test requis |
| **10.6 LTS** | ✅ Direct | Logical recommandé | Changements significatifs |
| **10.5** | ⚠️ Via 10.6 | Multi-step | 10.5 → 10.6 → 11.8 |
| **10.4** | ⚠️ Via 10.6 | Multi-step | 10.4 → 10.6 → 11.8 |
| **10.3** | ⚠️ Multi-step | Logical | 10.3 → 10.6 → 11.4 → 11.8 |
| **10.2 et moins** | 🔴 Complexe | Logical | Migration par étapes obligatoire |

### Migration multi-étapes

Pour les versions anciennes, une migration par étapes est nécessaire :

```bash
#!/bin/bash
# Migration multi-étapes : 10.3 → 11.8

echo "=== Migration 10.3 → 11.8 en plusieurs étapes ==="

# Étape 1 : 10.3 → 10.6 LTS
echo "[Étape 1] Migration 10.3 → 10.6 LTS"
# ... mise à jour vers 10.6 ...
mariadb-upgrade --force
mariadb -e "SELECT VERSION();"  # Vérifier 10.6.x

# Stabilisation et tests
sleep 60
mariadb -e "CHECK TABLE mysql.user;"

# Étape 2 : 10.6 → 11.4 LTS
echo "[Étape 2] Migration 10.6 → 11.4 LTS"
# ... mise à jour vers 11.4 ...
mariadb-upgrade --force
mariadb -e "SELECT VERSION();"  # Vérifier 11.4.x

# Stabilisation et tests
sleep 60
mariadb -e "CHECK TABLE mysql.user;"

# Étape 3 : 11.4 → 11.8 LTS
echo "[Étape 3] Migration 11.4 → 11.8 LTS"
# ... mise à jour vers 11.8 ...
mariadb-upgrade --force
mariadb -e "SELECT VERSION();"  # Vérifier 11.8.x

echo "=== Migration multi-étapes terminée ==="
```

💡 **Conseil** : Bien que les migrations directes soient souvent possibles techniquement, suivre les chemins recommandés réduit significativement les risques d'incompatibilités subtiles.

---

## L'outil mariadb-upgrade

### Rôle et fonctionnement

`mariadb-upgrade` est l'outil essentiel pour finaliser une mise à jour in-place. Il :

1. Met à jour les tables système (`mysql.*`)
2. Vérifie la compatibilité des tables utilisateur
3. Corrige les structures obsolètes
4. Met à jour les privilèges au nouveau format

```bash
# Utilisation basique
mariadb-upgrade

# Avec options courantes
mariadb-upgrade --force --verbose

# Vérification seule (sans modification)
mariadb-upgrade --check-only

# Sur un socket spécifique
mariadb-upgrade --socket=/var/run/mysqld/mysqld.sock

# Avec authentification
mariadb-upgrade --user=root --password='secret'
```

### Options importantes

| Option | Description | Quand utiliser |
|--------|-------------|----------------|
| `--force` | Ignore les erreurs, continue | Upgrade avec tables corrompues |
| `--verbose` | Affiche les détails | Debug, première migration |
| `--check-only` | Vérifie sans modifier | Pré-validation |
| `--skip-write-binlog` | N'écrit pas dans le binlog | Environnement répliqué |
| `--upgrade-system-tables` | Tables système uniquement | Upgrade rapide |

### Ce que fait mariadb-upgrade

```sql
-- mariadb-upgrade exécute essentiellement :

-- 1. Mise à jour des tables système
ALTER TABLE mysql.user ... ;
ALTER TABLE mysql.db ... ;
ALTER TABLE mysql.tables_priv ... ;
-- etc.

-- 2. Vérification des tables utilisateur
CHECK TABLE <chaque_table>;
REPAIR TABLE <si_nécessaire>;

-- 3. Mise à jour des métadonnées
UPDATE mysql.global_priv SET ... ;

-- 4. Marquage de la version
-- (stocké dans mysql.global_priv ou fichier .version)
```

---

## Spécificités MariaDB 11.8 LTS 🆕

### Changements impactant les upgrades

MariaDB 11.8 introduit plusieurs changements nécessitant une attention particulière lors des mises à jour :

| Changement | Impact upgrade | Action requise |
|------------|----------------|----------------|
| **utf8mb4 par défaut** | Nouvelles tables en utf8mb4 | Vérifier espace disque |
| **Collations UCA 14.0** | Nouveau tri par défaut | Tester comparaisons |
| **TLS obligatoire** | Connexions non-TLS rejetées | Configurer TLS ou désactiver |
| **TIMESTAMP étendu** | Format 2106 pour nouvelles tables | Rebuild tables temporelles |
| **System-Versioned format** | Nouveau format timestamp | Migration spécifique requise |

### Migration des System-Versioned Tables

⚠️ **Point critique** : MariaDB 11.8 modifie le format de stockage des timestamps dans les System-Versioned Tables pour supporter les dates au-delà de 2038.

```sql
-- Vérifier les tables system-versioned existantes
SELECT 
    table_schema,
    table_name,
    row_format,
    create_options
FROM information_schema.tables
WHERE create_options LIKE '%versioned%'
  AND table_schema NOT IN ('mysql', 'information_schema', 'performance_schema');

-- Exemple de sortie
+---------------+----------------+------------+---------------------------+
| table_schema  | table_name     | row_format | create_options            |
+---------------+----------------+------------+---------------------------+
| app_db        | contracts      | Dynamic    | WITH SYSTEM VERSIONING    |
| app_db        | audit_log      | Dynamic    | WITH SYSTEM VERSIONING    |
+---------------+----------------+------------+---------------------------+
```

**Procédure de migration des tables temporelles :**

```sql
-- Option 1 : Rebuild de la table (recommandé pour petites tables)
ALTER TABLE contracts ENGINE=InnoDB;

-- Option 2 : Pour les grandes tables, utiliser pt-online-schema-change
-- ou recréer avec dump/restore

-- Vérification post-migration
SELECT 
    table_name,
    column_name,
    column_type
FROM information_schema.columns
WHERE table_schema = 'app_db'
  AND column_name IN ('row_start', 'row_end');
```

La section 19.9 détaille complètement cette procédure de migration.

### Gestion du TLS par défaut

MariaDB 11.8 active TLS par défaut. Les clients non-TLS peuvent être rejetés :

```sql
-- Vérifier la configuration TLS actuelle
SHOW VARIABLES LIKE '%ssl%';
SHOW VARIABLES LIKE '%tls%';

-- Si nécessaire, autoriser temporairement les connexions non-TLS
-- (À utiliser uniquement pendant la transition)
SET GLOBAL require_secure_transport = OFF;
```

```ini
# my.cnf - Configuration TLS
[mariadbd]
# Activer TLS (par défaut en 11.8)
ssl = ON
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key = /etc/mysql/ssl/server-key.pem
ssl_ca = /etc/mysql/ssl/ca-cert.pem

# Pour désactiver temporairement (migration)
# require_secure_transport = OFF
```

---

## Planification d'un upgrade

### Checklist pré-upgrade

```markdown
## Checklist Pré-Upgrade MariaDB

### Préparation (J-7)
- [ ] Identifier la version source et cible
- [ ] Vérifier le chemin de migration supporté
- [ ] Lire les release notes de la version cible
- [ ] Identifier les breaking changes
- [ ] Planifier la fenêtre de maintenance

### Sauvegarde (J-1)
- [ ] Sauvegarde logique complète (mariadb-dump)
- [ ] Sauvegarde physique (mariabackup)
- [ ] Tester la restauration sur environnement de test
- [ ] Sauvegarder les fichiers de configuration

### Environnement de test (J-7 à J-1)
- [ ] Reproduire l'upgrade en staging
- [ ] Exécuter les tests de régression
- [ ] Valider les performances
- [ ] Documenter les problèmes rencontrés

### Jour J
- [ ] Communiquer le début de maintenance
- [ ] Arrêter les applications clientes
- [ ] Effectuer une dernière sauvegarde
- [ ] Exécuter l'upgrade
- [ ] Exécuter mariadb-upgrade
- [ ] Vérifier les logs d'erreur
- [ ] Tester les connexions
- [ ] Redémarrer les applications
- [ ] Communiquer la fin de maintenance

### Post-upgrade (J+1 à J+7)
- [ ] Monitoring intensif
- [ ] Vérifier les performances
- [ ] Surveiller les erreurs applicatives
- [ ] Documenter l'upgrade
```

### Timeline type d'un upgrade majeur

```
Timeline Upgrade Majeur (10.6 → 11.8)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

J-30        J-14        J-7         J-3       J          J+1        J+7
  │           │           │          │         │          │          │
  ▼           ▼           ▼          ▼         ▼          ▼          ▼
┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
│Plan │    │Test │    │Test │    │Dry  │    │PROD │    │Moni-│    │Vali-│
│     │    │Stag-│    │Stag-│    │Run  │    │UP-  │    │tor- │    │da-  │
│     │    │ing  │    │ing  │    │Prod │    │GRADE│    │ing  │    │tion │
│     │    │  1  │    │  2  │    │     │    │     │    │     │    │     │
└─────┘    └─────┘    └─────┘    └─────┘    └─────┘    └─────┘    └─────┘
  │           │          │          │          │
  │           │          │          │          │
  ▼           ▼          ▼          ▼          ▼
Lecture    Test       Correc-    Répéti-    Exécu-
Release    initial    tions      tion       tion
Notes                 issues     finale     réelle
```

---

## Résolution des problèmes courants

### Erreurs fréquentes et solutions

| Erreur | Cause probable | Solution |
|--------|----------------|----------|
| `Table 'mysql.user' doesn't exist` | Tables système corrompues | Restaurer mysql.* depuis backup |
| `Unknown collation` | Collation non supportée | Convertir les collations avant upgrade |
| `InnoDB: Table flags are X` | Format incompatible | Rebuild de la table |
| `Plugin 'xxx' is not loaded` | Plugin manquant | Installer le plugin ou désactiver |
| `Cannot start server` | Fichiers de données incompatibles | Migration logique nécessaire |

### Scripts de diagnostic

```sql
-- Vérification post-upgrade
SELECT 
    'Version' as check_type,
    VERSION() as result
UNION ALL
SELECT 
    'Uptime',
    CONCAT(@@uptime, ' seconds')
UNION ALL
SELECT 
    'Tables with errors',
    (SELECT COUNT(*) FROM information_schema.tables 
     WHERE table_comment LIKE '%error%')
UNION ALL
SELECT 
    'Charset server',
    @@character_set_server
UNION ALL
SELECT 
    'Collation server',
    @@collation_server;

-- Vérification des tables système
CHECK TABLE mysql.user;
CHECK TABLE mysql.db;
CHECK TABLE mysql.tables_priv;
CHECK TABLE mysql.columns_priv;
CHECK TABLE mysql.procs_priv;
CHECK TABLE mysql.global_priv;
```

```bash
#!/bin/bash
# Script de vérification post-upgrade

echo "=== Vérification Post-Upgrade ==="

# 1. Version
echo "[1] Version MariaDB:"
mariadb -N -e "SELECT VERSION();"

# 2. Erreurs dans les logs
echo "[2] Dernières erreurs (si présentes):"
tail -20 /var/log/mysql/error.log | grep -i "error\|warning" || echo "Aucune erreur récente"

# 3. Processus actifs
echo "[3] Processus actifs:"
mariadb -e "SHOW PROCESSLIST;"

# 4. Statut réplication (si applicable)
echo "[4] Statut réplication:"
mariadb -e "SHOW SLAVE STATUS\G" 2>/dev/null || echo "Réplication non configurée"

# 5. Variables critiques
echo "[5] Variables critiques:"
mariadb -e "SHOW VARIABLES WHERE Variable_name IN 
    ('version', 'innodb_buffer_pool_size', 'max_connections', 
     'character_set_server', 'sql_mode');"

echo "=== Fin vérification ==="
```

---

## Automatisation des upgrades

### Ansible playbook pour upgrade MariaDB

```yaml
# ansible/playbooks/mariadb-upgrade.yml
---
- name: MariaDB Upgrade Playbook
  hosts: mariadb_servers
  become: yes
  vars:
    target_version: "11.8"
    backup_dir: "/backup/pre-upgrade"
    
  pre_tasks:
    - name: Create backup directory
      file:
        path: "{{ backup_dir }}"
        state: directory
        mode: '0755'

    - name: Backup databases
      shell: |
        mariadb-dump --all-databases --routines --triggers \
          --single-transaction > {{ backup_dir }}/full_backup.sql
      
    - name: Verify backup
      stat:
        path: "{{ backup_dir }}/full_backup.sql"
      register: backup_file
      failed_when: backup_file.stat.size < 1000

  tasks:
    - name: Stop MariaDB
      service:
        name: mariadb
        state: stopped

    - name: Update MariaDB packages
      apt:
        name:
          - mariadb-server
          - mariadb-client
        state: latest
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Start MariaDB
      service:
        name: mariadb
        state: started

    - name: Run mariadb-upgrade
      command: mariadb-upgrade --force
      register: upgrade_result

    - name: Display upgrade result
      debug:
        var: upgrade_result.stdout_lines

  post_tasks:
    - name: Verify MariaDB version
      command: mariadb -N -e "SELECT VERSION();"
      register: new_version
      
    - name: Confirm upgrade
      debug:
        msg: "MariaDB upgraded to {{ new_version.stdout }}"
      failed_when: target_version not in new_version.stdout
```

---

## ✅ Points clés à retenir

- **Quatre stratégies principales** : In-place, Logical, Réplication, Blue-Green — choisissez selon votre contexte
- **mariadb-upgrade** est indispensable après toute mise à jour in-place pour mettre à jour les tables système
- Respectez les **chemins de migration supportés** pour éviter les incompatibilités
- Les migrations **cross-génération** (10.x → 11.x) nécessitent souvent plusieurs étapes
- MariaDB 11.8 introduit des **changements structurels** (TLS, utf8mb4, TIMESTAMP) à anticiper
- Les **System-Versioned Tables** nécessitent une attention particulière lors de la migration vers 11.8 🆕
- Toujours **tester en staging** avant la production
- Prévoyez un **plan de rollback testé** et documenté
- **Automatisez** les processus répétitifs avec Ansible ou scripts

---

## 🔗 Ressources et références

- [📖 MariaDB Upgrade Documentation](https://mariadb.com/kb/en/upgrading/)
- [📖 mariadb-upgrade Manual](https://mariadb.com/kb/en/mariadb-upgrade/)
- [📖 MariaDB 11.8 Upgrade Notes](https://mariadb.com/kb/en/upgrading-from-mariadb-114-to-mariadb-118/)
- [📖 What is mariadb-upgrade and when to use it](https://mariadb.com/kb/en/mariadb-upgrade/)
- [📖 System-Versioned Tables](https://mariadb.com/kb/en/system-versioned-tables/)

---

## 📚 Sections suivantes

Ce chapitre se poursuit avec des guides détaillés :

### 19.4.1 mariadb-upgrade
Utilisation avancée de l'outil, options détaillées, troubleshooting.

### 19.4.2 Upgrade in-place vs logical
Comparaison approfondie, critères de décision, scénarios d'application.

---

## ➡️ Section suivante

**[19.4.1 mariadb-upgrade](./04.1-mariadb-upgrade.md)** : Nous approfondirons l'utilisation de `mariadb-upgrade`, ses options avancées, les cas particuliers, et la résolution des problèmes courants lors de son exécution.

⏭️ [mariadb-upgrade](/19-migration-compatibilite/04.1-mariadb-upgrade.md)

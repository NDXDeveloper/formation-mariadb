🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2 Sauvegarde logique

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Section 12.1 (Stratégies de sauvegarde), Administration MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes et mécanismes des sauvegardes logiques
- **Distinguer** les avantages et limitations par rapport aux sauvegardes physiques
- **Choisir** l'outil approprié (mariadb-dump vs mydumper) selon le contexte
- **Optimiser** les performances des exports logiques en production
- **Concevoir** des stratégies de sauvegarde logique pour différents cas d'usage
- **Gérer** la compatibilité entre versions et plateformes

---

## Introduction

Les sauvegardes logiques constituent une approche fondamentalement différente des sauvegardes physiques : au lieu de copier directement les fichiers de données (`.ibd`, `.frm`, etc.), elles **exportent le contenu de la base sous forme de requêtes SQL** qui permettront de recréer l'état exact de la base.

### Qu'est-ce qu'une sauvegarde logique ?

Une sauvegarde logique génère un fichier texte contenant :

```sql
-- Exemple simplifié d'un export logique

-- Création de la structure
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(50) NOT NULL,
  `email` varchar(100) NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT current_timestamp(),
  PRIMARY KEY (`id`),
  UNIQUE KEY `username` (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Insertion des données
INSERT INTO `users` VALUES 
  (1,'john_doe','john@example.com','2025-01-15 10:30:00'),
  (2,'jane_smith','jane@example.com','2025-01-16 14:22:00'),
  (3,'bob_wilson','bob@example.com','2025-01-17 09:15:00');

-- Procédures, triggers, events, etc.
DELIMITER ;;
CREATE PROCEDURE `get_user_stats`() 
BEGIN
  SELECT COUNT(*) as total_users FROM users;
END;;
DELIMITER ;
```

Cette approche **textuelle et déclarative** présente des caractéristiques uniques qui la rendent indispensable dans certains scénarios, malgré ses limitations de performance.

### Philosophie : Portabilité vs Performance

```
┌─────────────────────────────────────────────────────┐
│         Trade-off Logique vs Physique               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Logique (SQL)          Physique (Fichiers)         │
│  ─────────────          ──────────────────          │
│                                                     │
│  + Portable             + Rapide                    │
│  + Sélectif             + Cohérent                  │
│  + Lisible              + Complet                   │
│  + Versionnable         + Production-ready          │
│                                                     │
│  - Lent                 - Dépend version            │
│  - Impact BDD           - Tout ou rien              │
│  - Pas temps réel       - Binaire                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Principes fondamentaux

### Comment fonctionne une sauvegarde logique ?

Le processus suit généralement ces étapes :

**1. Connexion au serveur MariaDB**
```bash
mariadb-dump --user=backup_user --password=***
```

**2. Verrouillage ou snapshot pour cohérence**
```sql
-- Option 1 : FLUSH TABLES WITH READ LOCK (bloquant)
FLUSH TABLES WITH READ LOCK;

-- Option 2 : START TRANSACTION WITH CONSISTENT SNAPSHOT (InnoDB)
START TRANSACTION WITH CONSISTENT SNAPSHOT;
```

**3. Extraction des métadonnées**
```sql
-- Récupération de la structure
SHOW CREATE TABLE users;
SHOW CREATE PROCEDURE get_user_stats;
SHOW CREATE TRIGGER audit_user_changes;
```

**4. Export des données**
```sql
-- Sélection de toutes les données
SELECT * FROM users;
SELECT * FROM orders;
-- ... pour chaque table
```

**5. Génération du fichier SQL**
```
Structure + Données → Fichier .sql
```

**6. Libération des verrous**
```sql
UNLOCK TABLES;  -- ou COMMIT;
```

### Cohérence transactionnelle

Un défi majeur des sauvegardes logiques est de garantir la **cohérence** des données exportées :

**Sans cohérence** :
```
10:00:00 - Export table orders (commande #1234 existe)
10:00:30 - Application : suppression commande #1234
10:00:45 - Export table order_items (référence commande #1234)

Résultat : Incohérence FK lors de la restauration !
```

**Avec cohérence (--single-transaction)** :
```sql
-- InnoDB utilise MVCC pour une vue cohérente
START TRANSACTION WITH CONSISTENT SNAPSHOT;

-- Vue figée au moment du START TRANSACTION
-- Les modifications concurrentes ne sont pas visibles
Export orders      → Snapshot au temps T
Export order_items → Même snapshot au temps T

COMMIT;

Résultat : Export cohérent ✅
```

⚠️ **Limitation** : `--single-transaction` ne fonctionne qu'avec les moteurs transactionnels (InnoDB, Aria avec transactions). MyISAM nécessite `FLUSH TABLES WITH READ LOCK`.

---

## Outils de sauvegarde logique pour MariaDB

### Vue d'ensemble

MariaDB propose deux approches majeures pour les sauvegardes logiques :

| Outil | Type | Parallélisme | Cas d'usage |
|-------|------|--------------|-------------|
| **mariadb-dump** | Single-threaded | Non | Petites bases, simplicité |
| **mydumper** | Multi-threaded | Oui | Moyennes/grandes bases, performance |

### mariadb-dump (mysqldump)

**Description** :
Outil officiel MariaDB/MySQL pour l'export logique. C'est le couteau suisse des sauvegardes logiques : simple, fiable, mais mono-thread.

```bash
# Syntaxe de base
mariadb-dump [options] database_name > backup.sql
```

**Caractéristiques** :
- ✅ Livré avec MariaDB (pas d'installation supplémentaire)
- ✅ Extrêmement stable et éprouvé (20+ ans de maturité)
- ✅ Options granulaires (tables, bases, routines, triggers)
- ✅ Compatible toutes versions MySQL/MariaDB
- ⚠️ Mono-thread (CPU/réseau sous-utilisés)
- ⚠️ Lent sur grosses bases (plusieurs heures pour > 100 Go)

**Quand l'utiliser** :
- Bases < 50 Go
- Export sélectif (quelques tables)
- Compatibilité maximale requise
- Migrations entre versions majeures
- Scripts d'automatisation simples

### mydumper / myloader

**Description** :
Alternative open-source multi-thread développée initialement par MySQL/MariaDB Community. Conçu pour la **performance** sur grandes bases.

```bash
# Syntaxe de base
mydumper --database=mydb --outputdir=/backups/mydumper
myloader --directory=/backups/mydumper --overwrite-tables
```

**Caractéristiques** :
- ✅ Export/import parallélisé (utilise tous les CPU)
- ✅ Jusqu'à 10x plus rapide que mariadb-dump
- ✅ Chunking intelligent des grandes tables
- ✅ Compression native (gzip/zstd)
- ✅ Support GTID et réplication
- ✅ Métadonnées détaillées (positions binlog, checksums)
- ⚠️ Installation séparée requise
- ⚠️ Syntaxe différente (courbe d'apprentissage)
- ⚠️ Génère de nombreux fichiers (un par table/chunk)

**Quand l'utiliser** :
- Bases > 50 Go
- Besoin de performance (fenêtre de backup limitée)
- Serveurs multi-CPU disponibles
- Exports réguliers en production
- Migrations avec volume important

### Comparaison détaillée

**Performance** :
```
Base de 500 Go, 200 tables

mariadb-dump (1 thread) :
├─ Export : 4h30
├─ Fichier : backup.sql (180 Go compressé)
└─ Import : 5h00

mydumper (8 threads) :
├─ Export : 35 minutes
├─ Fichiers : 200+ fichiers (175 Go compressé)
└─ Import : 45 minutes

Gain : ~6x plus rapide
```

**Granularité** :

```
mariadb-dump :
├─ Un seul fichier SQL
├─ Difficile de restaurer une seule table
└─ Séquentiel : structure puis données

mydumper :
├─ Fichiers séparés par table
├─ Facile de restaurer sélectivement
├─ Parallèle : structure + données simultanés
└─ Métadonnées riches (binlog pos, checksums)
```

**Gestion des grandes tables** :

```sql
-- mariadb-dump : Table entière dans une transaction
INSERT INTO huge_table VALUES (1,...), (2,...), ..., (1000000,...);

-- mydumper : Chunking automatique
-- Chunk 1 (rows 1-100000)
INSERT INTO huge_table VALUES (1,...), ..., (100000,...);
-- Chunk 2 (rows 100001-200000)
INSERT INTO huge_table VALUES (100001,...), ..., (200000,...);
```

💡 **Avantage chunking** : Réduit la taille des transactions, évite les timeouts, permet la reprise sur erreur.

---

## Avantages des sauvegardes logiques

### 1. Portabilité maximale

**Entre versions** :
```
MariaDB 10.6 ──[dump]──► SQL ──[restore]──► MariaDB 11.8
MySQL 5.7    ──[dump]──► SQL ──[restore]──► MariaDB 11.8
```

Les sauvegardes physiques (Mariabackup) nécessitent souvent des versions compatibles. Les backups logiques fonctionnent entre versions très différentes.

**Entre plateformes** :
```
Linux x86_64 ──[dump]──► SQL ──[restore]──► Windows ARM
Docker       ──[dump]──► SQL ──[restore]──► Bare metal
```

**Entre configurations** :
```
innodb_page_size=16K ──[dump]──► SQL ──[restore]──► innodb_page_size=4K
latin1               ──[dump]──► SQL ──[restore]──► utf8mb4
```

### 2. Sélectivité fine

Contrairement aux backups physiques (tout ou rien), les backups logiques permettent une granularité extrême :

```bash
# Une seule base
mariadb-dump myapp > myapp.sql

# Quelques tables spécifiques
mariadb-dump myapp users orders payments > critical_tables.sql

# Exclure certaines tables
mariadb-dump myapp --ignore-table=myapp.logs --ignore-table=myapp.sessions

# Seulement la structure (no data)
mariadb-dump --no-data myapp > schema_only.sql

# Seulement les données (no structure)
mariadb-dump --no-create-info myapp > data_only.sql

# Filtrage WHERE sur une table
mariadb-dump myapp users --where="created_at >= '2025-01-01'"
```

### 3. Lisibilité et inspection

Le format SQL texte permet :

```bash
# Inspecter le contenu sans restaurer
grep "INSERT INTO users" backup.sql | head -5

# Compter les lignes exportées
grep -c "INSERT INTO" backup.sql

# Extraire une table spécifique
sed -n '/CREATE TABLE `orders`/,/UNLOCK TABLES/p' backup.sql > orders_only.sql

# Diff entre deux backups
diff backup_v1.sql backup_v2.sql

# Versioning Git (pour petites bases)
git add schema.sql
git commit -m "Schema update: add column email_verified"
```

### 4. Compatibilité avec le versionning

Les dumps SQL peuvent être versionnés dans Git/SVN :

```bash
# Workflow typique pour schémas
mariadb-dump --no-data --routines myapp > schema.sql
git add schema.sql
git commit -m "Add index on users.email"
git push origin main

# Review dans PR
git diff schema.sql
```

💡 **Use case** : Équipes DevOps suivant Infrastructure as Code pour les schémas de base de données.

### 5. Migrations et réorganisations

Les backups logiques facilitent les transformations :

```sql
-- Pendant le dump : modifier les données
mariadb-dump myapp | sed 's/old_value/new_value/g' > modified.sql

-- Changer le nom de base
sed 's/USE `myapp`/USE `myapp_v2`/g' backup.sql > myapp_v2.sql

-- Modifier le moteur de stockage
sed 's/ENGINE=MyISAM/ENGINE=InnoDB/g' backup.sql > innodb_version.sql
```

---

## Limitations et inconvénients

### 1. Performance sur grandes bases

**Problème** : mariadb-dump est mono-thread et doit lire toutes les données via SQL.

```
Benchmark : Base 1 To, 500 tables

mariadb-dump :
├─ Temps : 8-10 heures
├─ CPU : 1 core à 100%, les autres idle
├─ I/O : Séquentiel, débit modéré
└─ Fenêtre acceptable : Non (trop long)

Mariabackup (physique) :
├─ Temps : 2-3 heures
├─ CPU : Faible (copie fichiers)
├─ I/O : Copie directe, débit maximal
└─ Fenêtre acceptable : Oui
```

**Solution partielle** : Utiliser mydumper pour améliorer significativement les performances.

### 2. Impact sur la base de données en production

**Charge CPU** :
```sql
-- mariadb-dump exécute de nombreux SELECT
SELECT * FROM users;           -- Scan complet
SELECT * FROM orders;          -- Scan complet
SELECT * FROM products;        -- Scan complet
-- ... pour chaque table

Impact : 20-40% CPU utilisé pour générer le SQL
```

**Charge I/O** :
```
Lecture séquentielle de toutes les tables
→ Peut saturer les disques
→ Impact sur les requêtes concurrentes (cache éviction)
```

**Locks potentiels** :
```sql
-- Sans --single-transaction : FLUSH TABLES WITH READ LOCK
-- Bloque TOUTES les écritures pendant l'export !

-- Avec --single-transaction : mieux, mais...
-- Long-running transaction → impact MVCC
-- Purge InnoDB retardé → ibdata grossit temporairement
```

### 3. Pas de backup incrémental natif

Contrairement à Mariabackup, mariadb-dump ne supporte pas les backups incrémentaux :

```
Mariabackup :
└─ Full + Incremental OK ✅

mariadb-dump :
└─ Full uniquement ❌
   (sauf techniques avancées avec binary logs)
```

### 4. Restauration lente

**Problème** : L'import SQL nécessite :
- Parser le SQL
- Exécuter les CREATE TABLE
- Construire les index pendant les INSERT
- Recalculer les statistiques

```
Restauration 100 Go :

Physical (Mariabackup) :
├─ Copie fichiers : 30 min
├─ Prepare (redo logs) : 15 min
└─ Total : ~45 min

Logical (mariadb-dump) :
├─ Parse SQL : 20 min
├─ CREATE tables : 5 min
├─ INSERT données : 90 min
├─ Build indexes : 60 min
└─ Total : ~2h45

Ratio : ~3.5x plus lent
```

### 5. Taille du fichier

Le format SQL texte est verbeux :

```sql
-- Données brutes : 1 Go
-- Format SQL non compressé : 2-3 Go (overhead syntaxe)
-- Format SQL compressé (gzip) : 400-600 Mo

-- Mariabackup compressé : 300-400 Mo (plus compact)
```

### 6. Gestion des transactions longues

Un export avec `--single-transaction` sur une base volumineuse crée une transaction de plusieurs heures :

**Impacts** :
```
Transaction longue (4h) :
├─ UNDO logs gonflent (espace ibdata)
├─ Purge InnoDB bloqué (vieilles versions accumulées)
├─ Risque dépassement max_tmp_space_usage
└─ Performance globale dégradée
```

⚠️ **Recommandation** : Sur très grosses bases (> 500 Go), préférer Mariabackup ou mydumper avec chunking.

---

## Comparaison Logique vs Physique

### Tableau synthétique

| Critère | Logique (SQL) | Physique (Mariabackup) |
|---------|---------------|------------------------|
| **Vitesse backup** | ⭐⭐ Lent | ⭐⭐⭐ Rapide |
| **Vitesse restauration** | ⭐ Très lent | ⭐⭐⭐ Rapide |
| **Portabilité** | ⭐⭐⭐ Maximale | ⭐ Limitée (version) |
| **Sélectivité** | ⭐⭐⭐ Granulaire | ⭐ Tout ou rien |
| **Impact production** | ⭐⭐ Moyen | ⭐⭐ Faible |
| **Taille fichier** | ⭐⭐ Correct | ⭐⭐⭐ Compact |
| **Incrémental** | ❌ Non natif | ✅ Oui |
| **Cohérence** | ⚠️ Dépend options | ✅ Garantie |
| **Complexité** | ⭐⭐⭐ Simple | ⭐⭐ Modérée |
| **Inspection** | ⭐⭐⭐ Texte lisible | ❌ Binaire |

### Matrice de décision

```
┌─────────────────────────────────────────────────────┐
│    Quand utiliser Logique vs Physique ?             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  LOGIQUE si :                                       │
│  ✓ Taille < 100 Go                                  │
│  ✓ Migration cross-version nécessaire               │
│  ✓ Export sélectif (tables spécifiques)             │
│  ✓ Besoin de lisibilité/inspection                  │
│  ✓ Versionning du schéma requis                     │
│  ✓ Environnement dev/test                           │
│                                                     │
│  PHYSIQUE si :                                      │
│  ✓ Taille > 100 Go                                  │
│  ✓ Performance critique (fenêtre limitée)           │
│  ✓ Production avec SLA stricts                      │
│  ✓ Backup incrémental requis                        │
│  ✓ Restauration rapide nécessaire (RTO court)       │
│  ✓ Hot backup sans impact                           │
│                                                     │
│  HYBRIDE (les deux) si :                            │
│  ✓ Besoin de flexibilité maximale                   │
│  ✓ Physique quotidien + Logique hebdo validation    │
│  ✓ Conformité réglementaire                         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Cas d'usage spécifiques des sauvegardes logiques

### 1. Migration entre versions majeures

**Scénario** : Migration MySQL 5.7 → MariaDB 11.8

```bash
# Sur MySQL 5.7
mysqldump --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events > mysql57_full.sql

# Vérification et nettoyage
sed -i 's/utf8mb3/utf8mb4/g' mysql57_full.sql  # Charset update
grep -v "sql_mode" mysql57_full.sql > clean.sql  # Remove incompatible modes

# Sur MariaDB 11.8
mariadb < clean.sql

# Validation
mariadb -e "SHOW DATABASES;"
mariadb myapp -e "SELECT COUNT(*) FROM users;"
```

### 2. Export de sous-ensembles de données

**Scénario** : Copier données de production vers environnement de test (avec anonymisation)

```bash
# Export sélectif + filtrage
mariadb-dump production_db users orders products \
  --where="created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)" \
  --single-transaction > last_30_days.sql

# Anonymisation (simple)
sed -i "s/\([a-zA-Z0-9._-]*\)@\([a-zA-Z0-9._-]*\)/anonymized@example.com/g" last_30_days.sql

# Import dans test
mariadb test_db < last_30_days.sql
```

### 3. Backup de schéma pour versioning

**Scénario** : Suivre l'évolution du schéma dans Git

```bash
#!/bin/bash
# schema_backup.sh

DATE=$(date +%Y%m%d)

# Export structure uniquement
mariadb-dump \
  --no-data \
  --routines \
  --triggers \
  --events \
  myapp > schema_${DATE}.sql

# Versioning Git
git add schema_${DATE}.sql
git commit -m "Schema snapshot ${DATE}"
git push origin main
```

### 4. Conformité et audit

**Scénario** : Archivage long terme avec signature

```bash
# Export avec métadonnées
mariadb-dump --all-databases > backup_$(date +%Y%m%d).sql

# Compression
gzip backup_$(date +%Y%m%d).sql

# Signature cryptographique (intégrité)
sha256sum backup_$(date +%Y%m%d).sql.gz > backup.sha256
gpg --sign backup.sha256

# Archivage sécurisé
aws s3 cp backup_$(date +%Y%m%d).sql.gz s3://audit-backups/mariadb/ \
  --storage-class GLACIER_IR \
  --server-side-encryption AES256
```

### 5. Synchronisation partielle entre environnements

**Scénario** : Sync configuration (tables de référence) entre datacenters

```bash
# Export tables de configuration uniquement
mariadb-dump myapp \
  countries \
  currencies \
  tax_rates \
  product_categories > config_tables.sql

# Transfert sécurisé
scp config_tables.sql user@remote-dc:/tmp/

# Import distant
ssh user@remote-dc "mariadb myapp < /tmp/config_tables.sql"
```

---

## Optimisations pour la production

### Réduire l'impact sur le serveur

**1. Utiliser un replica pour le backup**
```
Primary (Production) ──réplication──► Replica (Backup)
                                           │
                                           └──► mariadb-dump
                                                (0 impact Primary)
```

**2. Limiter les ressources**
```bash
# Throttling CPU
nice -n 19 mariadb-dump ... > backup.sql

# Throttling I/O (si supporté)
ionice -c2 -n7 mariadb-dump ... > backup.sql

# Parallélisme limité avec mydumper
mydumper --threads=4  # Ne pas saturer tous les CPU
```

**3. Utiliser la compression à la volée**
```bash
# Économiser I/O disque
mariadb-dump myapp | gzip -c > backup.sql.gz

# Avec mydumper (compression native)
mydumper --compress --compress-threads=2
```

### Accélérer les exports

**1. Désactiver certaines validations** (avec précaution)
```bash
mariadb-dump \
  --skip-lock-tables \      # Pas de lock (attention cohérence)
  --skip-add-locks \        # Pas de LOCK TABLES dans le dump
  --skip-extended-insert \  # Peut ralentir mais réduit mémoire
  --single-transaction      # Cohérence InnoDB
```

**2. Optimiser le réseau**
```bash
# Compression réseau si backup distant
mariadb-dump myapp | ssh user@backup-server "gzip > /backups/myapp.sql.gz"

# Parallélisme réseau avec mydumper
mydumper --compress --outputdir=/nfs/backups/
```

**3. Segmenter les exports**
```bash
# Séparer structure et données
mariadb-dump --no-data myapp > structure.sql
mariadb-dump --no-create-info myapp > data.sql

# Permet de versionner structure.sql sans les volumes de data.sql
```

---

## Stratégies de rétention pour backups logiques

### Rotation typique

```bash
#!/bin/bash
# logical_backup_rotation.sh

BACKUP_DIR="/backups/logical"
DATE=$(date +%Y%m%d)

# Backup quotidien
mariadb-dump --all-databases | gzip > "$BACKUP_DIR/daily/backup_${DATE}.sql.gz"

# Copie hebdomadaire (dimanche)
if [ $(date +%u) -eq 7 ]; then
  cp "$BACKUP_DIR/daily/backup_${DATE}.sql.gz" "$BACKUP_DIR/weekly/"
fi

# Copie mensuelle (1er du mois)
if [ $(date +%d) -eq 01 ]; then
  cp "$BACKUP_DIR/daily/backup_${DATE}.sql.gz" "$BACKUP_DIR/monthly/"
fi

# Purge
find "$BACKUP_DIR/daily" -mtime +7 -delete      # 7 jours
find "$BACKUP_DIR/weekly" -mtime +28 -delete    # 4 semaines
find "$BACKUP_DIR/monthly" -mtime +365 -delete  # 12 mois
```

### Versioning avec Delta

Pour les petites bases, on peut optimiser en stockant des deltas :

```bash
# Premier backup complet
mariadb-dump myapp > backup_full.sql

# Backups suivants : seulement les différences
mariadb-dump myapp > backup_current.sql
diff backup_full.sql backup_current.sql > backup_delta_$(date +%Y%m%d).diff
```

💡 **Attention** : Technique avancée, nécessite gestion rigoureuse pour reconstruction.

---

## Validation et tests

### Vérification d'intégrité

```bash
# Checksum du fichier
md5sum backup.sql.gz > backup.md5
md5sum -c backup.md5  # Vérification ultérieure

# Test de décompression
gzip -t backup.sql.gz && echo "✅ Archive OK"

# Test syntaxe SQL (rapide)
zcat backup.sql.gz | mariadb --help > /dev/null 2>&1 && echo "✅ SQL valide"
```

### Tests de restauration

```bash
#!/bin/bash
# test_restore.sh

# Environnement de test isolé
docker run -d --name restore-test \
  -e MARIADB_ROOT_PASSWORD=test \
  mariadb:11.8

# Attendre démarrage
sleep 10

# Restauration
zcat backup.sql.gz | docker exec -i restore-test mariadb

# Validation basique
docker exec restore-test mariadb -e "
  SELECT 
    COUNT(*) as total_databases 
  FROM information_schema.SCHEMATA 
  WHERE SCHEMA_NAME NOT IN ('mysql','information_schema','performance_schema');
"

# Nettoyage
docker rm -f restore-test
```

---

## ✅ Points clés à retenir

- **Sauvegardes logiques** : Export SQL textuel, portable mais plus lent que physique
- **mariadb-dump** : Simple, fiable, mono-thread — idéal < 50 Go
- **mydumper** : Multi-thread, performant — recommandé > 50 Go
- **Portabilité** : Principal avantage — migration entre versions/plateformes
- **Sélectivité** : Granularité fine (tables, conditions WHERE, structure seule)
- **Limitations** : Performance, impact production, pas d'incrémental natif
- **`--single-transaction`** : ESSENTIEL pour cohérence InnoDB sans blocage
- **Use cases** : Migrations, dev/test, exports partiels, versionning schémas
- **Production** : Utiliser replica pour backup, limiter ressources (nice/ionice)
- **Complémentaire** : Backup logique hebdo + physique quotidien = stratégie robuste

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 mariadb-dump - MariaDB Knowledge Base](https://mariadb.com/kb/en/mariadb-dump/)
- [📖 Backup and Restore Overview - MariaDB KB](https://mariadb.com/kb/en/backup-and-restore-overview/)
- [📖 Making Backups with mariadb-dump](https://mariadb.com/kb/en/making-backups-with-mariadb-dump/)

### Outils tiers

- [mydumper/myloader - GitHub Repository](https://github.com/mydumper/mydumper)
- [mydumper Documentation](https://github.com/mydumper/mydumper/blob/master/docs/mydumper_usage.rst)

### Articles et guides

- [Logical vs Physical Backups - Percona Blog](https://www.percona.com/blog/logical-vs-physical-mysql-backups/)
- [mysqldump Performance Optimization - MySQL Performance Blog](https://www.percona.com/blog/mysqldump-performance-considerations/)

---

## ➡️ Sections suivantes

Les sous-sections suivantes détailleront l'utilisation pratique de chaque outil :

**[12.2.1 - mysqldump / mariadb-dump](./02.1-mysqldump-mariadb-dump.md)** : Options essentielles, syntaxe complète, cas d'usage détaillés avec exemples.

**[12.2.2 - Options essentielles](./02.2-options-essentielles.md)** : Guide approfondi des options critiques (--single-transaction, --routines, --master-data, etc.).

**[12.2.3 - mydumper/myloader](./02.3-mydumper-myloader.md)** : Installation, configuration, utilisation avancée, parallélisation optimale.

---


⏭️ [mysqldump / mariadb-dump](/12-sauvegarde-restauration/02.1-mysqldump-mariadb-dump.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4 Sauvegarde incrémentale avec binary logs

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Sections 12.1-12.3, Réplication MariaDB (Chapitre 13), Binary logs

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle des binary logs dans la stratégie de sauvegarde
- **Configurer** les binary logs pour un backup incrémental continu
- **Choisir** le format approprié (STATEMENT, ROW, MIXED) selon le contexte
- **Gérer** la rotation, la purge et l'archivage des binary logs
- **Implémenter** un Point-in-Time Recovery (PITR) complet
- **Automatiser** la sauvegarde des binary logs vers le cloud (S3)
- **Calculer** le RPO effectif avec les binary logs

---

## Introduction

Les **binary logs** (binlogs) constituent un élément fondamental de toute stratégie de sauvegarde professionnelle. Ils enregistrent toutes les modifications de données sous forme de **journal de transactions**, permettant de rejouer l'historique complet des changements depuis un point de référence.

### Binary logs : Le chaînon manquant

Même avec des backups complets quotidiens, vous pouvez perdre jusqu'à 24 heures de données :

```
┌─────────────────────────────────────────────────────┐
│      Problème : RPO de 24h avec Full backup         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Dimanche 02:00    Lundi 14:37      Lundi 16:00     │
│  ┌──────────┐      │                │               │
│  │   FULL   │      ▼                ▼               │
│  │  BACKUP  │   INCIDENT        DÉTECTION           │
│  └──────────┘                                       │
│       │                                             │
│       └──────────────────►◄─────────────────►       │
│                           PERTE 14h37 DE DONNÉES    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Solution : Binary logs + Full backup** :

```
┌───────────────────────────────────────────────────────┐
│    Solution : RPO < 1 seconde avec Binary logs        │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Dimanche 02:00    Lundi 14:37      Lundi 16:00       │
│  ┌──────────┐      │                │                 │
│  │   FULL   │──────┼────────────────┼────►            │
│  │  BACKUP  │      ▼                ▼                 │
│  └──────────┘   INCIDENT        DÉTECTION             │
│       │                                               │
│       ├──► binlog.000042 (dim-lun)                    │
│       ├──► binlog.000043 (lun 00h-12h)                │
│       └──► binlog.000044 (lun 12h-14h37) ✅           │
│                                                       │
│  Restauration : Full + binlog.042 + .043 + .044       │
│  Perte : 0 secondes (jusqu'à la dernière transaction) │
│                                                       │
└───────────────────────────────────────────────────────┘
```

💡 **Principe clé** : Les binary logs transforment un backup quotidien en backup **continu**.

---

## Qu'est-ce que les binary logs ?

### Définition

Les binary logs sont des fichiers binaires qui enregistrent, dans l'ordre chronologique, **toutes les modifications de données** (INSERT, UPDATE, DELETE) et **changements de structure** (DDL).

```
Structure typique :
/var/log/mysql/
├── mariadb-bin.000001    (512 MB)
├── mariadb-bin.000002    (512 MB)
├── mariadb-bin.000003    (320 MB, fichier actif)
└── mariadb-bin.index     (index des fichiers)
```

### Contenu d'un binary log

```sql
-- Exemple de contenu (vue logique)

/* Position 0-120 : Métadonnées */
# at 4
# Server ver: 11.8.0-MariaDB, Binlog ver: 4
# Format_desc

/* Position 120-245 : BEGIN transaction */
# at 120
BEGIN

/* Position 245-389 : INSERT */
# at 245
INSERT INTO users (username, email) 
VALUES ('alice', 'alice@example.com')

/* Position 389-512 : UPDATE */
# at 389
UPDATE orders SET status='shipped' WHERE order_id=12345

/* Position 512-600 : COMMIT */
# at 512
COMMIT

/* Position 600-650 : DDL */
# at 600
ALTER TABLE products ADD COLUMN rating DECIMAL(3,2)
```

Chaque événement a une **position** (offset dans le fichier) qui permet de rejouer précisément les transactions.

### Utilisations des binary logs

Les binary logs servent à plusieurs usages critiques :

**1. Point-in-Time Recovery (PITR)** ⭐ :
```
Full backup + Binary logs = Restauration à n'importe quel instant
```

**2. Réplication** :
```
Primary envoie binary logs → Replica les rejoue
```

**3. Audit** :
```
Analyse forensique : Qui a fait quoi et quand ?
```

**4. Change Data Capture (CDC)** :
```
Streaming vers Kafka, Elasticsearch, Data Lake
```

---

## Configuration des binary logs

### Activation

**Dans my.cnf** :
```ini
# /etc/mysql/mariadb.conf.d/binlog.cnf

[mariadb]
# Activation des binary logs
log_bin = /var/log/mysql/mariadb-bin

# Format (ROW recommandé)
binlog_format = ROW

# Taille max par fichier (rotation automatique)
max_binlog_size = 512M

# Durée de rétention (jours)
expire_logs_days = 7

# Sync sur disque (0=perf, 1=sécurité max)
sync_binlog = 1

# Inclure les requêtes modifiant des tables sans PK
binlog_row_image = FULL
```

**Redémarrage requis** :
```bash
systemctl restart mariadb

# Vérification
mariadb -e "SHOW VARIABLES LIKE 'log_bin';"
# log_bin | ON
```

### Formats de binary logs

MariaDB propose trois formats d'enregistrement :

#### 1. STATEMENT (Basé sur les requêtes)

**Principe** : Enregistre les requêtes SQL telles quelles.

```sql
-- Requête exécutée
UPDATE users SET last_login = NOW() WHERE user_id = 123;

-- Enregistré dans le binlog
UPDATE users SET last_login = NOW() WHERE user_id = 123;
```

**Avantages** :
- ✅ Binlogs compacts (quelques octets par requête)
- ✅ Lisible et auditavle (format SQL)
- ✅ Moins d'espace disque

**Inconvénients** :
- ❌ Non-déterministe (NOW(), RAND(), UUID() peuvent varier)
- ❌ Réplication peut diverger
- ❌ Triggers/fonctions peuvent poser problème

**Cas d'usage** :
- Environnements où l'espace est critique
- Pas de fonctions non-déterministes
- Audit simplifié

#### 2. ROW (Basé sur les lignes modifiées) ✅ Recommandé

**Principe** : Enregistre les modifications au niveau des lignes.

```sql
-- Requête exécutée
UPDATE users SET last_login = '2025-12-13 14:30:00' WHERE user_id = 123;

-- Enregistré dans le binlog (format simplifié)
### UPDATE users
### WHERE
###   @1=123 /* user_id */
###   @2='alice' /* username */
###   @3='2025-12-10 10:00:00' /* old last_login */
### SET
###   @3='2025-12-13 14:30:00' /* new last_login */
```

**Avantages** :
- ✅ Déterministe (valeurs exactes enregistrées)
- ✅ Réplication fiable
- ✅ Compatible tous types de fonctions
- ✅ Audit précis (avant/après)

**Inconvénients** :
- ❌ Binlogs plus volumineux (surtout mass updates)
- ❌ Moins lisible

**Cas d'usage** :
- ✅ **Production (standard de facto)**
- Réplication critique
- Besoin de déterminisme absolu

#### 3. MIXED (Hybride)

**Principe** : MariaDB choisit automatiquement entre STATEMENT et ROW.

```
Requête déterministe     → STATEMENT (compact)
Requête non-déterministe → ROW (sûr)
```

**Avantages** :
- ✅ Compromis espace/sécurité
- ✅ Automatique

**Inconvénients** :
- ⚠️ Comportement moins prévisible
- ⚠️ Complexité debugging

**Cas d'usage** :
- Environnements mixtes (OLTP + batch)
- Transition STATEMENT → ROW

### Recommandation production

```ini
# Configuration recommandée pour production
[mariadb]
log_bin = /var/log/mysql/mariadb-bin
binlog_format = ROW              # ✅ Recommandé
sync_binlog = 1                  # Sécurité max (ACID)
expire_logs_days = 7             # Rétention 7 jours
max_binlog_size = 512M           # Rotation tous les 512 MB
binlog_row_image = FULL          # Inclure toutes les colonnes
```

---

## Rotation et gestion des binary logs

### Rotation automatique

MariaDB crée un nouveau fichier binlog lorsque :

**1. Taille maximale atteinte** :
```ini
max_binlog_size = 512M
# Dès que mariadb-bin.000042 atteint 512 MB
# → Création de mariadb-bin.000043
```

**2. Redémarrage du serveur** :
```bash
systemctl restart mariadb
# → Nouveau binlog créé
```

**3. Commande manuelle** :
```sql
FLUSH BINARY LOGS;
-- Ferme le binlog actuel et en crée un nouveau
```

### Purge des anciens binlogs

**Automatique (expire_logs_days)** :
```ini
[mariadb]
expire_logs_days = 7
# Supprime automatiquement les binlogs > 7 jours
```

**Manuelle** :
```sql
-- Supprimer tous les binlogs avant un fichier spécifique
PURGE BINARY LOGS TO 'mariadb-bin.000042';

-- Supprimer tous les binlogs avant une date
PURGE BINARY LOGS BEFORE '2025-12-01 00:00:00';

-- Supprimer tous les binlogs sauf les N derniers jours
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;
```

⚠️ **ATTENTION** : Ne jamais supprimer manuellement les fichiers binlog (via `rm`). Toujours utiliser `PURGE BINARY LOGS`.

### Vérification des binlogs existants

```sql
-- Liste de tous les binlogs
SHOW BINARY LOGS;

/*
+---------------------+-----------+-----------+
| Log_name            | File_size | Encrypted |
+---------------------+-----------+-----------+
| mariadb-bin.000040  | 536870912 | No        |
| mariadb-bin.000041  | 536870912 | No        |
| mariadb-bin.000042  | 320145678 | No        |
+---------------------+-----------+-----------+
*/

-- Binlog actuellement actif
SHOW MASTER STATUS;

/*
+---------------------+----------+--------------+------------------+
| File                | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+---------------------+----------+--------------+------------------+
| mariadb-bin.000042  | 320145678| myapp        |                  |
+---------------------+----------+--------------+------------------+
*/
```

### Dimensionnement et calculs

**Estimer la taille des binlogs** :

```
Taille quotidienne binlogs = Volume transactions × Taille moyenne transaction

Exemple :
- 10 000 transactions/heure
- Taille moyenne : 2 KB par transaction
- Heures d'activité : 16h/jour

Taille quotidienne = 10 000 × 2 KB × 16 = 320 MB/jour

Rétention 7 jours = 320 MB × 7 = 2.24 GB
```

**Monitoring de la croissance** :

```bash
#!/bin/bash
# binlog_growth_monitor.sh

BINLOG_DIR="/var/log/mysql"

# Taille actuelle
CURRENT_SIZE=$(du -sh $BINLOG_DIR | cut -f1)

# Taille il y a 24h (depuis logs)
YESTERDAY_SIZE=$(grep "binlog size" /var/log/backup.log | tail -2 | head -1 | awk '{print $4}')

echo "Binlog size: $CURRENT_SIZE (yesterday: $YESTERDAY_SIZE)"

# Alerte si croissance > 50% attendue
```

---

## Sauvegarde des binary logs

### Stratégie de sauvegarde

Les binary logs doivent être sauvegardés **séparément** des backups complets :

```
┌──────────────────────────────────────────────────┐
│        Stratégie complète de backup              │
├──────────────────────────────────────────────────┤
│                                                  │
│  Full backup      : Quotidien à 02:00            │
│  Binary logs      : Archivage toutes les 15 min  │
│  Rétention binlog : 7 jours local                │
│  Archivage S3     : 30 jours (compliance)        │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Archivage local avec rotation

```bash
#!/bin/bash
# binlog_archive.sh

BINLOG_DIR="/var/log/mysql"
ARCHIVE_DIR="/backups/binlogs/$(date +%Y%m%d)"
LOG_FILE="/var/log/binlog_archive.log"

mkdir -p "$ARCHIVE_DIR"

# Flush pour créer un nouveau binlog
mariadb -e "FLUSH BINARY LOGS;"

# Copier tous les binlogs sauf le dernier (actif)
CURRENT_BINLOG=$(mariadb -Nse "SHOW MASTER STATUS" | awk '{print $1}')

for binlog in $(ls $BINLOG_DIR/mariadb-bin.[0-9]* 2>/dev/null); do
  BINLOG_FILE=$(basename $binlog)
  
  # Ne pas archiver le binlog actif
  if [ "$BINLOG_FILE" != "$CURRENT_BINLOG" ]; then
    # Vérifier si pas déjà archivé
    if [ ! -f "$ARCHIVE_DIR/$BINLOG_FILE" ]; then
      cp "$binlog" "$ARCHIVE_DIR/"
      echo "$(date): Archived $BINLOG_FILE" >> "$LOG_FILE"
    fi
  fi
done

# Compression des archives de plus de 1 jour
find "$ARCHIVE_DIR/.." -type f -name "mariadb-bin.*" -mtime +1 ! -name "*.gz" -exec gzip {} \;

echo "$(date): Binlog archive completed" >> "$LOG_FILE"
```

### Sauvegarde vers S3 (cloud-native)

```bash
#!/bin/bash
# binlog_to_s3.sh

BINLOG_DIR="/var/log/mysql"
S3_BUCKET="s3://my-database-backups/binlogs"
DATE=$(date +%Y%m%d)

# Flush pour rotation
mariadb -e "FLUSH BINARY LOGS;"

# Upload vers S3 avec compression
for binlog in $(ls $BINLOG_DIR/mariadb-bin.[0-9]* 2>/dev/null); do
  BINLOG_FILE=$(basename $binlog)
  
  # Vérifier si déjà uploadé
  if ! aws s3 ls "$S3_BUCKET/$DATE/$BINLOG_FILE.gz" > /dev/null 2>&1; then
    # Compresser et uploader
    gzip -c "$binlog" | \
      aws s3 cp - "$S3_BUCKET/$DATE/$BINLOG_FILE.gz" \
        --storage-class INTELLIGENT_TIERING \
        --server-side-encryption AES256
    
    echo "Uploaded $BINLOG_FILE to S3"
  fi
done

# Lifecycle policy S3 (configuration une seule fois)
# - INTELLIGENT_TIERING : 30 jours
# - Glacier : 31-365 jours
# - Suppression : > 365 jours
```

**Configuration AWS CLI** :
```bash
aws configure
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: ...
# Default region: eu-west-1

# Test
aws s3 ls s3://my-database-backups/binlogs/
```

### Sauvegarde temps réel avec streaming

Pour des RPO ultra-courts (< 1 minute), streaming des binlogs en temps réel :

```bash
#!/bin/bash
# binlog_streaming.sh

BINLOG_DIR="/var/log/mysql"

# Utiliser mysqlbinlog en mode remote streaming
mysqlbinlog \
  --read-from-remote-server \
  --host=localhost \
  --user=replication_user \
  --password=secure_pass \
  --raw \
  --stop-never \
  --result-file=/backups/binlogs/stream- \
  mariadb-bin.000042

# Option --stop-never : Continue indéfiniment (daemon)
# Option --raw : Format binaire (pas de conversion SQL)
```

💡 **Use case** : Bases critiques (finance, santé) où même 15 minutes de perte sont inacceptables.

---

## Point-in-Time Recovery (PITR)

### Concept

Le PITR permet de restaurer la base de données **à n'importe quel instant précis** en combinant :
1. Un backup complet (point de départ)
2. Les binary logs (rejeu jusqu'à l'instant cible)

### Scénario typique

```
Timeline :
─────────────────────────────────────────────────

Dimanche 02:00     Lundi 14:37      Lundi 14:50
     │                 │                 │
     ▼                 ▼                 ▼
  FULL BACKUP    ERREUR APPLI      DÉTECTION
  (200 GB)       (DELETE sans       
                  WHERE)             

Objectif : Restaurer à Lundi 14:36 (juste avant l'erreur)
```

### Procédure de PITR

#### Étape 1 : Identifier la position/temps cible

```sql
-- Option 1 : Par horodatage (plus courant)
CIBLE_TIME = '2025-12-09 14:36:00'

-- Option 2 : Par position binlog (plus précis)
CIBLE_BINLOG = 'mariadb-bin.000043'
CIBLE_POSITION = 123456789
```

#### Étape 2 : Restaurer le backup complet

```bash
# Arrêt du serveur
systemctl stop mariadb

# Nettoyage datadir
rm -rf /var/lib/mysql/*

# Restauration Mariabackup
mariabackup --copy-back --target-dir=/backups/full/20251208

# Permissions
chown -R mysql:mysql /var/lib/mysql

# Redémarrage
systemctl start mariadb
```

**État actuel** : Base restaurée au dimanche 02:00.

#### Étape 3 : Identifier les binlogs nécessaires

```sql
-- Vérifier la position du backup
cat /backups/full/20251208/xtrabackup_binlog_info
# mariadb-bin.000040 position 1234

-- Binlogs requis :
# mariadb-bin.000040 (depuis position 1234)
# mariadb-bin.000041 (complet)
# mariadb-bin.000042 (complet)
# mariadb-bin.000043 (jusqu'à 14:36:00)
```

#### Étape 4 : Application des binlogs

**Méthode 1 : Par horodatage (recommandé)** :

```bash
# Extraire et appliquer les binlogs
mysqlbinlog \
  --start-position=1234 \
  /backups/binlogs/mariadb-bin.000040 \
  /backups/binlogs/mariadb-bin.000041 \
  /backups/binlogs/mariadb-bin.000042 \
  /backups/binlogs/mariadb-bin.000043 \
  --stop-datetime='2025-12-09 14:36:00' | \
  mariadb -u root -p

# Note : Application séquentielle dans l'ordre chronologique
```

**Méthode 2 : Par position (précision maximale)** :

```bash
mysqlbinlog \
  --start-position=1234 \
  --stop-position=123456789 \
  /backups/binlogs/mariadb-bin.000040 \
  ... | \
  mariadb -u root -p
```

**Méthode 3 : Exclure une transaction spécifique** :

```bash
# Scénario : Identifier la position de l'erreur
mysqlbinlog /backups/binlogs/mariadb-bin.000043 | \
  grep -A 5 "DELETE FROM users" | \
  head -1
# at 123456780  # Position de la mauvaise requête

# Appliquer jusqu'avant l'erreur
mysqlbinlog \
  /backups/binlogs/mariadb-bin.000043 \
  --stop-position=123456779 | \  # 1 position avant
  mariadb -u root -p

# Puis reprendre après l'erreur (si nécessaire)
mysqlbinlog \
  /backups/binlogs/mariadb-bin.000043 \
  --start-position=123456850 | \  # Après la mauvaise requête
  mariadb -u root -p
```

#### Étape 5 : Validation

```sql
-- Vérifier l'état final
SELECT COUNT(*) FROM users;
-- Doit correspondre à l'état attendu à 14:36

-- Vérifier les dernières transactions
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;

-- Audit trail
SELECT MAX(created_at) FROM orders;
-- Doit être <= '2025-12-09 14:36:00'
```

### Script complet de PITR

```bash
#!/bin/bash
# pitr_restore.sh

set -e  # Exit on error

BACKUP_DIR="/backups/full/20251208"
BINLOG_DIR="/backups/binlogs"
TARGET_TIME="2025-12-09 14:36:00"
MYSQL_ROOT_PASSWORD="secure_password"

echo "=== Point-in-Time Recovery to $TARGET_TIME ==="

# 1. Arrêt du serveur
echo "[1/5] Stopping MariaDB..."
systemctl stop mariadb

# 2. Sauvegarde de sécurité du datadir actuel
echo "[2/5] Backing up current datadir..."
mv /var/lib/mysql /var/lib/mysql.backup.$(date +%Y%m%d_%H%M%S)

# 3. Restauration du backup complet
echo "[3/5] Restoring full backup..."
mariabackup --copy-back --target-dir="$BACKUP_DIR"
chown -R mysql:mysql /var/lib/mysql

# 4. Démarrage pour application des binlogs
echo "[4/5] Starting MariaDB..."
systemctl start mariadb
sleep 10

# 5. Application des binlogs
echo "[5/5] Applying binary logs until $TARGET_TIME..."

# Récupérer la position de départ depuis le backup
START_BINLOG=$(cat $BACKUP_DIR/xtrabackup_binlog_info | awk '{print $1}')
START_POS=$(cat $BACKUP_DIR/xtrabackup_binlog_info | awk '{print $2}')

echo "Starting from $START_BINLOG position $START_POS"

# Trouver tous les binlogs depuis START_BINLOG
BINLOGS=$(ls $BINLOG_DIR/mariadb-bin.* | \
  awk -v start="$START_BINLOG" '$0 >= start' | \
  sort)

# Appliquer
mysqlbinlog \
  --start-position=$START_POS \
  --stop-datetime="$TARGET_TIME" \
  $BINLOGS | \
  mariadb -u root -p"$MYSQL_ROOT_PASSWORD"

echo "=== PITR Complete ==="
echo "Database restored to $TARGET_TIME"
echo "Please validate the data before resuming production"
```

---

## Automatisation et monitoring

### Cron pour archivage des binlogs

```cron
# /etc/cron.d/mariadb-binlog-archive

# Archivage binlogs toutes les 15 minutes
*/15 * * * * backup_user /scripts/binlog_archive.sh >> /var/log/binlog_archive.log 2>&1

# Upload S3 toutes les heures
0 * * * * backup_user /scripts/binlog_to_s3.sh >> /var/log/binlog_s3.log 2>&1

# Purge anciens binlogs (> 7 jours) quotidien
0 3 * * * backup_user mariadb -e "PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;"
```

### Monitoring avec Prometheus

**Exporter mysqld_exporter** :
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mariadb'
    static_configs:
      - targets: ['localhost:9104']
```

**Métriques clés** :
```promql
# Nombre de binlogs
mysql_global_status_binlog_cache_disk_use

# Taille totale des binlogs
sum(mysql_binlog_size_bytes)

# Taux de croissance
rate(mysql_binlog_size_bytes[5m])

# Alerte si > 10 GB/heure
rate(mysql_binlog_size_bytes[1h]) > 10737418240
```

### Alerting

```yaml
# alertmanager.yml
groups:
  - name: mariadb_binlog
    rules:
      # Croissance anormale
      - alert: BinlogHighGrowth
        expr: rate(mysql_binlog_size_bytes[1h]) > 10737418240
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Binlog growth > 10 GB/h"
          
      # Espace disque critique
      - alert: BinlogDiskSpaceLow
        expr: node_filesystem_avail_bytes{mountpoint="/var/log/mysql"} < 10737418240
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Binlog disk space < 10 GB"
          
      # Archivage échoué
      - alert: BinlogArchiveFailed
        expr: time() - binlog_last_archive_timestamp > 3600
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Binlog archive failed for > 1h"
```

---

## ✅ Points clés à retenir

- **Binary logs** : Journal continu des modifications, fondamental pour PITR
- **Formats** : ROW recommandé (déterministe), STATEMENT compact, MIXED hybride
- **Configuration** : `log_bin=ON`, `binlog_format=ROW`, `sync_binlog=1` en production
- **Rotation** : Automatique (max_binlog_size), manuelle (FLUSH BINARY LOGS)
- **Archivage** : Toutes les 15 min vers stockage distant (S3), rétention 7-30 jours
- **PITR** : Full backup + binlogs = restauration à n'importe quel instant
- **RPO** : Fréquence archivage = RPO (15 min archivage = RPO 15-45 min)
- **Sécurité** : Chiffrement binlogs (encrypt_binlog=ON), audit conformité
- **Monitoring** : Taille binlogs, taux croissance, lag réplication, échecs archivage
- **Purge** : `PURGE BINARY LOGS` uniquement (jamais `rm`), vérifier réplication avant

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Binary Log - MariaDB Knowledge Base](https://mariadb.com/kb/en/binary-log/)
- [📖 Binary Log Formats - MariaDB KB](https://mariadb.com/kb/en/binary-log-formats/)
- [📖 mysqlbinlog - MariaDB KB](https://mariadb.com/kb/en/mysqlbinlog/)
- [📖 PURGE BINARY LOGS - MariaDB KB](https://mariadb.com/kb/en/purge-binary-logs/)

---

## ➡️ Section suivante

**[12.5 - Restauration](./05-restauration.md)** : Procédures détaillées de restauration complète, Point-in-Time Recovery pratique, validation post-restauration.

---


⏭️ [Restauration](/12-sauvegarde-restauration/05-restauration.md)

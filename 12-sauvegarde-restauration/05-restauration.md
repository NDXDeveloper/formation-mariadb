🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5 Restauration

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Sections 12.1-12.4, Administration MariaDB, Compréhension des backups

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Maîtriser** les procédures de restauration complète et Point-in-Time Recovery
- **Valider** systématiquement l'intégrité des données après restauration
- **Gérer** les situations d'urgence avec méthodologie et sang-froid
- **Automatiser** les tests de restauration pour garantir la fiabilité des backups
- **Restaurer** depuis différentes sources (local, S3, Kubernetes VolumeSnapshots)
- **Optimiser** le RTO (Recovery Time Objective) en production
- **Documenter** et maintenir des procédures de restauration à jour

---

## Introduction

La restauration est **le moment de vérité** de toute stratégie de sauvegarde. C'est l'instant où l'on découvre si les backups sont réellement exploitables ou s'ils ne sont qu'une illusion de sécurité.

### Le paradoxe du backup

```
┌──────────────────────────────────────────────────────┐
│              Paradoxe du backup                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Temps passé sur les backups       : 95%             │
│  Temps passé sur les tests restore : 5%              │
│                                                      │
│  Mais...                                             │
│                                                      │
│  Valeur réelle du backup            : 0%             │
│  Valeur réelle de la restauration   : 100%           │
│                                                      │
│  → Un backup non testé = Pas de backup               │
│                                                      │
└──────────────────────────────────────────────────────┘
```

💡 **Principe fondamental** : *"Un backup que vous n'avez jamais restauré n'est pas un backup, c'est une hypothèse"*.

### Statistiques révélatrices

D'après les études de l'industrie (Veeam Data Protection Report 2024) :

```
Organisations ayant subi une perte de données en 2023 : 76%

Causes d'échec de restauration :
├─ Backup corrompu ou incomplet           : 34%
├─ Procédure incorrecte/obsolète          : 28%
├─ Infrastructure de restauration absente : 18%
├─ Manque de compétences/documentation    : 12%
└─ Dépendances non documentées            : 8%

Organisations testant leurs restaurations :
├─ Mensuellement  : 12%
├─ Trimestriellement : 23%
├─ Annuellement   : 31%
└─ Jamais         : 34% ⚠️
```

⚠️ **Alerte** : 34% des organisations ne testent **jamais** leurs procédures de restauration !

---

## Types de restauration

### 1. Restauration complète (Full Restore)

Rétablir l'intégralité de la base de données à un instant T précis.

```
Scénario typique :
┌────────────────────────────────────────────┐
│ Corruption généralisée de la base          │
│ Panne matérielle (disque détruit)          │
│ Ransomware chiffrant toutes les données    │
│ Migration vers nouveau serveur             │
└────────────────────────────────────────────┘

Workflow :
1. Arrêt du serveur MariaDB
2. Nettoyage du datadir
3. Restauration du backup complet
4. Application des binary logs (optionnel)
5. Redémarrage et validation
```

**Temps typique** : RTO de 1-4 heures selon taille base

### 2. Point-in-Time Recovery (PITR)

Restaurer la base à un instant précis (par exemple, juste avant une erreur).

```
Scénario typique :
┌────────────────────────────────────────────┐
│ DELETE/UPDATE sans WHERE clause            │
│ DROP TABLE accidentel                      │
│ Migration applicative défectueuse          │
│ Corruption de données progressive          │
└────────────────────────────────────────────┘

Workflow :
1. Restauration du backup complet
2. Application des binary logs jusqu'à T-1
3. Validation de l'état des données
4. Basculement en production
```

**Temps typique** : RTO de 2-6 heures selon volume binlogs

### 3. Restauration partielle (Selective Restore)

Restaurer uniquement certaines tables ou bases de données.

```
Scénario typique :
┌────────────────────────────────────────────┐
│ Corruption d'une seule table               │
│ Suppression accidentelle de données        │
│ Récupération de données archivées          │
└────────────────────────────────────────────┘

Workflow (logique) :
1. Extraction de la table depuis dump SQL
2. Import dans base temporaire
3. Copie sélective des données
4. Validation et merge avec production
```

**Temps typique** : RTO de 30 min - 2 heures

### 4. Restauration de disaster recovery (DR)

Bascule complète vers site de secours après incident majeur.

```
Scénario typique :
┌────────────────────────────────────────────┐
│ Datacenter détruit (incendie, inondation)  │
│ Panne généralisée (électrique, réseau)     │
│ Cyberattaque paralysant l'infrastructure   │
└────────────────────────────────────────────┘

Workflow :
1. Activation du plan DR
2. Provisionnement infrastructure secours
3. Restauration depuis backups distants
4. Reconfiguration DNS/VIP
5. Tests et bascule production
```

**Temps typique** : RTO de 4-12 heures selon plan DR

---

## Principes généraux de restauration

### La règle des 3C

Une restauration réussie repose sur trois piliers :

**1. Cohérence** : Les données restaurées doivent être transactionnellement cohérentes
```sql
-- Vérification cohérence référentielle
SELECT COUNT(*) FROM orders o 
LEFT JOIN users u ON o.user_id = u.id 
WHERE u.id IS NULL;
-- Résultat attendu : 0 (aucune commande orpheline)
```

**2. Complétude** : Toutes les données requises doivent être présentes
```bash
# Vérifier que tous les fichiers de backup existent
for file in $(cat backup_manifest.txt); do
  [ -f "$file" ] || echo "MISSING: $file"
done
```

**3. Chronologie** : L'ordre d'application des backups et binlogs est critique
```
Full backup (T0) → Inc1 (T1) → Inc2 (T2) → Binlogs (T2-T3)
                   ↓           ↓           ↓
               ORDRE STRICT OBLIGATOIRE
```

### Checklist pré-restauration

Avant toute restauration, valider systématiquement :

```
☐ Backup disponible et accessible
☐ Intégrité du backup vérifiée (checksums)
☐ Espace disque suffisant (1.5x taille backup)
☐ Infrastructure cible prête (serveur, réseau)
☐ Dépendances identifiées (binary logs, config)
☐ Fenêtre de maintenance confirmée
☐ Communication équipes (ops, dev, business)
☐ Procédure documentée et validée
☐ Backup de sécurité de l'état actuel (si possible)
☐ Rollback plan préparé
```

### Environnement de restauration

**Jamais directement en production** : Toujours restaurer d'abord dans un environnement isolé.

```
┌────────────────────────────────────────────────┐
│         Workflow de restauration sécurisé      │
├────────────────────────────────────────────────┤
│                                                │
│  1. Restauration dans environnement TEST       │
│     ├─ Validation cohérence                    │
│     ├─ Vérification données critiques          │
│     └─ Tests applicatifs                       │
│                                                │
│  2. Si validation OK → Restauration PROD       │
│     ├─ Fenêtre de maintenance                  │
│     ├─ Procédure identique à TEST              │
│     └─ Validation finale                       │
│                                                │
│  3. Si validation KO → Investigation           │
│     ├─ Analyse de l'échec                      │
│     ├─ Correction procédure                    │
│     └─ Nouveau test                            │
│                                                │
└────────────────────────────────────────────────┘
```

---

## Restauration depuis sauvegardes logiques

### Depuis mariadb-dump

**Restauration complète** :

```bash
#!/bin/bash
# restore_logical_full.sh

BACKUP_FILE="/backups/logical/full_backup_20251213.sql.gz"
LOG_FILE="/var/log/mariadb_restore.log"

echo "[$(date)] Starting logical restore" | tee -a $LOG_FILE

# 1. Vérification intégrité
echo "Checking backup integrity..." | tee -a $LOG_FILE
gunzip -t $BACKUP_FILE
if [ $? -ne 0 ]; then
  echo "ERROR: Backup file is corrupted" | tee -a $LOG_FILE
  exit 1
fi

# 2. Arrêt applications (éviter connexions pendant restore)
echo "Stopping applications..." | tee -a $LOG_FILE
systemctl stop nginx  # ou autre reverse proxy

# 3. Restauration
echo "Restoring database..." | tee -a $LOG_FILE
zcat $BACKUP_FILE | mariadb -u root -p 2>&1 | tee -a $LOG_FILE

# 4. Vérification
echo "Verifying restoration..." | tee -a $LOG_FILE
mariadb -e "SHOW DATABASES;" > /tmp/db_list.txt
diff /tmp/db_list.txt /backups/db_list_reference.txt

# 5. Redémarrage applications
echo "Restarting applications..." | tee -a $LOG_FILE
systemctl start nginx

echo "[$(date)] Logical restore completed" | tee -a $LOG_FILE
```

**Restauration partielle (une seule table)** :

```bash
#!/bin/bash
# restore_logical_single_table.sh

BACKUP_FILE="/backups/full_backup.sql.gz"
TABLE_NAME="users"
DATABASE="myapp"

# Extraire la table spécifique
zcat $BACKUP_FILE | \
  sed -n "/DROP TABLE.*\`${TABLE_NAME}\`/,/UNLOCK TABLES/p" > /tmp/${TABLE_NAME}.sql

# Restaurer dans base temporaire
mariadb -e "CREATE DATABASE IF NOT EXISTS restore_temp;"
mariadb restore_temp < /tmp/${TABLE_NAME}.sql

# Validation manuelle avant merge vers production
echo "Table restored in restore_temp.${TABLE_NAME}"
echo "Please validate before merging to production"
```

### Depuis mydumper/myloader

**Restauration complète** :

```bash
#!/bin/bash
# restore_mydumper.sh

BACKUP_DIR="/backups/mydumper/20251213"
TARGET_DB="myapp"

# Restauration parallélisée
myloader \
  --directory=$BACKUP_DIR \
  --database=$TARGET_DB \
  --overwrite-tables \
  --threads=8 \
  --verbose=3

# Vérification
mariadb $TARGET_DB -e "SELECT COUNT(*) FROM users;"
```

**Avantages myloader** :
- ✅ Parallélisation (8-16 threads)
- ✅ 3-5x plus rapide que mysql < dump.sql
- ✅ Reprise automatique sur erreur (par chunk)

---

## Restauration depuis sauvegardes physiques (Mariabackup)

### Restauration full backup

**Procédure standard** :

```bash
#!/bin/bash
# restore_mariabackup_full.sh

set -e  # Exit on error

BACKUP_DIR="/backups/mariabackup/full/20251213"
DATADIR="/var/lib/mysql"
LOG_FILE="/var/log/mariabackup_restore.log"

echo "[$(date)] Starting Mariabackup restore" | tee -a $LOG_FILE

# 1. Arrêt MariaDB
echo "Stopping MariaDB..." | tee -a $LOG_FILE
systemctl stop mariadb

# 2. Sauvegarde de sécurité du datadir actuel
echo "Backing up current datadir..." | tee -a $LOG_FILE
if [ -d "$DATADIR" ]; then
  mv $DATADIR ${DATADIR}.backup.$(date +%Y%m%d_%H%M%S)
fi

# 3. Préparation du backup (si pas déjà fait)
if [ ! -f "$BACKUP_DIR/xtrabackup_prepared" ]; then
  echo "Preparing backup..." | tee -a $LOG_FILE
  mariabackup --prepare --target-dir=$BACKUP_DIR 2>&1 | tee -a $LOG_FILE
fi

# 4. Restauration (copie des fichiers)
echo "Restoring files..." | tee -a $LOG_FILE
mariabackup --copy-back --target-dir=$BACKUP_DIR 2>&1 | tee -a $LOG_FILE

# 5. Permissions
echo "Fixing permissions..." | tee -a $LOG_FILE
chown -R mysql:mysql $DATADIR

# 6. Redémarrage
echo "Starting MariaDB..." | tee -a $LOG_FILE
systemctl start mariadb

# 7. Validation
sleep 10
mariadb -e "SELECT VERSION();" && \
  echo "[$(date)] Restore completed successfully" | tee -a $LOG_FILE || \
  echo "[$(date)] ERROR: Restore failed" | tee -a $LOG_FILE

```

### Restauration incrémentale

**Procédure avec chaîne d'incrémentaux** :

```bash
#!/bin/bash
# restore_mariabackup_incremental.sh

FULL_DIR="/backups/mariabackup/full/20251208"
INC1_DIR="/backups/mariabackup/inc/20251209"
INC2_DIR="/backups/mariabackup/inc/20251210"
INC3_DIR="/backups/mariabackup/inc/20251211"

# 1. Préparation du full (apply-log-only)
echo "Preparing full backup..."
mariabackup --prepare --apply-log-only --target-dir=$FULL_DIR

# 2. Application de chaque incrément (apply-log-only sauf dernier)
echo "Applying incremental 1..."
mariabackup --prepare --apply-log-only \
  --target-dir=$FULL_DIR \
  --incremental-dir=$INC1_DIR

echo "Applying incremental 2..."
mariabackup --prepare --apply-log-only \
  --target-dir=$FULL_DIR \
  --incremental-dir=$INC2_DIR

# 3. Dernier incrément (SANS apply-log-only)
echo "Applying incremental 3 (final)..."
mariabackup --prepare \
  --target-dir=$FULL_DIR \
  --incremental-dir=$INC3_DIR

# 4. Restauration
systemctl stop mariadb
rm -rf /var/lib/mysql/*
mariabackup --copy-back --target-dir=$FULL_DIR
chown -R mysql:mysql /var/lib/mysql
systemctl start mariadb

echo "Incremental restore completed"
```

⚠️ **CRITICAL** : L'option `--apply-log-only` est **obligatoire** pour tous les incrémentaux sauf le dernier.

---

## Point-in-Time Recovery (PITR)

### Workflow complet

Le PITR combine restauration physique/logique + application des binary logs :

```
┌───────────────────────────────────────────────────┐
│          Workflow PITR complet                    │
├───────────────────────────────────────────────────┤
│                                                   │
│  Phase 1 : Restauration du backup de référence    │
│  ├─ Full backup (ou Full + Incrementaux)          │
│  └─ État base au temps T0                         │
│                                                   │
│  Phase 2 : Identification du point cible          │
│  ├─ Timestamp : 2025-12-13 14:36:00               │
│  └─ ou Position binlog : 123456789                │
│                                                   │
│  Phase 3 : Application des binary logs            │
│  ├─ binlog.000042 (complet)                       │
│  ├─ binlog.000043 (complet)                       │
│  └─ binlog.000044 (jusqu'à 14:36:00)              │
│                                                   │
│  Phase 4 : Validation                             │
│  ├─ Cohérence référentielle                       │
│  ├─ Comptages de lignes                           │
│  └─ Tests applicatifs                             │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Exemple concret : Récupération après DELETE erroné

**Contexte** :
```
Lundi 14:37 : DELETE FROM users WHERE status='inactive';
              ↑ Erreur : Oubli du WHERE (tous les users supprimés!)
Lundi 14:50 : Détection de l'incident
Objectif    : Restaurer à 14:36:59 (1 sec avant l'erreur)
```

**Procédure** :

```bash
#!/bin/bash
# pitr_delete_recovery.sh

FULL_BACKUP="/backups/mariabackup/full/20251213"
BINLOG_DIR="/backups/binlogs"
TARGET_TIME="2025-12-13 14:36:59"

# 1. Restauration du full
echo "Step 1/4: Restoring full backup..."
systemctl stop mariadb
rm -rf /var/lib/mysql/*
mariabackup --copy-back --target-dir=$FULL_BACKUP
chown -R mysql:mysql /var/lib/mysql
systemctl start mariadb
sleep 10

# 2. Identification des binlogs requis
echo "Step 2/4: Identifying required binlogs..."
START_BINLOG=$(cat $FULL_BACKUP/xtrabackup_binlog_info | awk '{print $1}')
START_POS=$(cat $FULL_BACKUP/xtrabackup_binlog_info | awk '{print $2}')

echo "Starting from $START_BINLOG position $START_POS"

# 3. Application des binlogs jusqu'à 14:36:59
echo "Step 3/4: Applying binary logs until $TARGET_TIME..."
BINLOGS=$(ls $BINLOG_DIR/mariadb-bin.* | \
  awk -v start="$START_BINLOG" '$0 >= start' | sort)

mysqlbinlog \
  --start-position=$START_POS \
  --stop-datetime="$TARGET_TIME" \
  $BINLOGS | \
  mariadb -u root -p

# 4. Validation
echo "Step 4/4: Validating restoration..."
USER_COUNT=$(mariadb -Nse "SELECT COUNT(*) FROM myapp.users;")
echo "Users count: $USER_COUNT (expected: > 0)"

if [ $USER_COUNT -gt 0 ]; then
  echo "✅ PITR completed successfully"
  echo "Database restored to $TARGET_TIME"
else
  echo "❌ PITR failed - users table is empty"
  exit 1
fi
```

---

## Validation post-restauration

### Checklist de validation

Après toute restauration, valider systématiquement :

**1. Validation technique** :
```sql
-- Base de données accessibles
SHOW DATABASES;

-- Tables présentes
USE myapp;
SHOW TABLES;

-- Moteurs corrects
SELECT ENGINE, COUNT(*) 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA='myapp' 
GROUP BY ENGINE;

-- Pas de corruption
CHECK TABLE users;
CHECK TABLE orders;

-- Taille cohérente
SELECT 
  table_name,
  table_rows,
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE table_schema = 'myapp'
ORDER BY size_mb DESC;
```

**2. Validation fonctionnelle** :
```sql
-- Données critiques présentes
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM orders WHERE created_at >= '2025-12-01';

-- Cohérence référentielle
SELECT COUNT(*) FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;
-- Résultat attendu : 0

-- Timestamps cohérents avec le point de restauration
SELECT MAX(created_at) FROM orders;
-- Doit être <= point de restauration cible

-- Dernières transactions
SELECT * FROM audit_log ORDER BY timestamp DESC LIMIT 10;
```

**3. Validation applicative** :
```bash
#!/bin/bash
# app_validation.sh

# Test connexion application
curl -f http://localhost:8080/health || echo "❌ App health check failed"

# Test login utilisateur
curl -X POST http://localhost:8080/api/login \
  -d '{"username":"test","password":"test"}' || echo "❌ Login failed"

# Test requête métier
curl -f http://localhost:8080/api/orders/recent || echo "❌ Orders API failed"

echo "✅ Application validation completed"
```

### Matrice de validation

| Type de donnée | Méthode de validation | Criticité |
|----------------|----------------------|-----------|
| Tables système | SHOW TABLES | Critique |
| Procédures stockées | SHOW PROCEDURE STATUS | Haute |
| Triggers | SHOW TRIGGERS | Haute |
| Events | SHOW EVENTS | Moyenne |
| Utilisateurs/privilèges | SELECT * FROM mysql.user | Critique |
| Données métier | COUNT(*), SUM(), agrégations | Critique |
| Cohérence FK | LEFT JOIN NULL check | Critique |
| Performance | EXPLAIN SELECT sur requêtes clés | Haute |
| Connexion app | Health check endpoint | Critique |

---

## Tests de restauration

### Fréquence recommandée

```
┌──────────────────────────────────────────────┐
│     Fréquence des tests de restauration      │
├──────────────────────────────────────────────┤
│                                              │
│  Environnement          Fréquence minimale   │
│  ─────────────          ──────────────────   │
│  Production CRITIQUE    Hebdomadaire         │
│  Production STANDARD    Mensuelle            │
│  Développement          Trimestrielle        │
│  Archivage              Semestrielle         │
│                                              │
└──────────────────────────────────────────────┘
```

### Test automatisé de restauration

```bash
#!/bin/bash
# automated_restore_test.sh

set -e

TEST_DIR="/tmp/restore_test_$(date +%Y%m%d_%H%M%S)"
BACKUP_FILE="/backups/latest_full.sql.gz"
REPORT_FILE="/var/log/restore_tests/report_$(date +%Y%m%d).txt"

mkdir -p $TEST_DIR
mkdir -p /var/log/restore_tests

echo "=== Restore Test Report ===" > $REPORT_FILE
echo "Date: $(date)" >> $REPORT_FILE
echo "Backup: $BACKUP_FILE" >> $REPORT_FILE
echo "" >> $REPORT_FILE

# 1. Test d'intégrité du backup
echo "[TEST 1/5] Backup integrity..." | tee -a $REPORT_FILE
if gunzip -t $BACKUP_FILE 2>&1; then
  echo "✅ PASS: Backup file is valid" | tee -a $REPORT_FILE
else
  echo "❌ FAIL: Backup file is corrupted" | tee -a $REPORT_FILE
  exit 1
fi

# 2. Restauration dans conteneur Docker
echo "[TEST 2/5] Restoration in Docker..." | tee -a $REPORT_FILE
docker run -d --name restore_test_db \
  -e MARIADB_ROOT_PASSWORD=test \
  -v $TEST_DIR:/backup \
  mariadb:11.8

sleep 15

# Import du backup
zcat $BACKUP_FILE | docker exec -i restore_test_db mariadb -u root -ptest

# 3. Validation des bases
echo "[TEST 3/5] Database validation..." | tee -a $REPORT_FILE
DB_COUNT=$(docker exec restore_test_db mariadb -Nse "SELECT COUNT(*) FROM information_schema.SCHEMATA WHERE SCHEMA_NAME NOT IN ('mysql','information_schema','performance_schema');")
echo "Databases restored: $DB_COUNT" | tee -a $REPORT_FILE

# 4. Validation des tables
echo "[TEST 4/5] Table validation..." | tee -a $REPORT_FILE
TABLE_COUNT=$(docker exec restore_test_db mariadb -Nse "SELECT COUNT(*) FROM information_schema.TABLES WHERE TABLE_SCHEMA='myapp';")
echo "Tables restored: $TABLE_COUNT" | tee -a $REPORT_FILE

# 5. Validation des données
echo "[TEST 5/5] Data validation..." | tee -a $REPORT_FILE
USER_COUNT=$(docker exec restore_test_db mariadb -Nse "SELECT COUNT(*) FROM myapp.users;" 2>/dev/null || echo "0")
echo "Users count: $USER_COUNT" | tee -a $REPORT_FILE

# Nettoyage
docker rm -f restore_test_db
rm -rf $TEST_DIR

# Résumé
echo "" >> $REPORT_FILE
if [ $USER_COUNT -gt 0 ]; then
  echo "✅ RESTORE TEST PASSED" | tee -a $REPORT_FILE
  exit 0
else
  echo "❌ RESTORE TEST FAILED" | tee -a $REPORT_FILE
  # Envoyer alerte
  mail -s "ALERT: Restore test failed" dba@example.com < $REPORT_FILE
  exit 1
fi
```

**Automatisation avec cron** :
```cron
# Test de restauration hebdomadaire (dimanche 3h)
0 3 * * 0 /scripts/automated_restore_test.sh
```

---

## Restauration cloud-native

### Depuis AWS S3

```bash
#!/bin/bash
# restore_from_s3.sh

S3_BUCKET="s3://my-database-backups"
BACKUP_DATE="20251213"
LOCAL_DIR="/tmp/restore_from_s3"

mkdir -p $LOCAL_DIR

# 1. Téléchargement depuis S3
echo "Downloading from S3..."
aws s3 cp $S3_BUCKET/mariabackup/full/$BACKUP_DATE/ $LOCAL_DIR/ --recursive

# 2. Décompression si nécessaire
if ls $LOCAL_DIR/*.gz 1> /dev/null 2>&1; then
  echo "Decompressing..."
  gunzip $LOCAL_DIR/*.gz
fi

# 3. Préparation du backup
echo "Preparing backup..."
mariabackup --prepare --target-dir=$LOCAL_DIR

# 4. Restauration standard
systemctl stop mariadb
rm -rf /var/lib/mysql/*
mariabackup --copy-back --target-dir=$LOCAL_DIR
chown -R mysql:mysql /var/lib/mysql
systemctl start mariadb

echo "Restore from S3 completed"
```

### Depuis Kubernetes VolumeSnapshot

```yaml
# restore-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-restored-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: mariadb-snapshot-20251213
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-restored
spec:
  serviceName: mariadb-restored
  replicas: 1
  selector:
    matchLabels:
      app: mariadb-restored
  template:
    metadata:
      labels:
        app: mariadb-restored
    spec:
      containers:
      - name: mariadb
        image: mariadb:11.8
        env:
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      dataSource:
        name: mariadb-snapshot-20251213
        kind: VolumeSnapshot
        apiGroup: snapshot.storage.k8s.io
      resources:
        requests:
          storage: 100Gi
```

**Procédure** :
```bash
# 1. Lister les snapshots disponibles
kubectl get volumesnapshot

# 2. Appliquer le manifest de restauration
kubectl apply -f restore-from-snapshot.yaml

# 3. Vérifier le déploiement
kubectl get pods -l app=mariadb-restored

# 4. Validation
kubectl exec mariadb-restored-0 -- mariadb -e "SHOW DATABASES;"
```

---

## Optimisation du RTO

### Techniques de réduction du temps de restauration

**1. Pré-prepare des backups** :
```bash
# Préparer le backup immédiatement après sa création
mariabackup --backup --target-dir=/backups/full
mariabackup --prepare --target-dir=/backups/full  # Fait immédiatement

# Gain : -30% du temps de restauration
```

**2. Utiliser des disques SSD/NVMe** :
```
HDD : 150 MB/s  → 1 To restauré en ~2h
SSD : 500 MB/s  → 1 To restauré en ~35 min
NVMe: 3000 MB/s → 1 To restauré en ~6 min

Gain : Jusqu'à 95% du temps de restauration
```

**3. Restauration parallélisée** :
```bash
# mydumper/myloader avec 16 threads
myloader --threads=16 --directory=/backups/mydumper

# Gain : 3-5x plus rapide que import séquentiel
```

**4. Infrastructure pré-provisionnée** :
```
Serveur de restauration dédié :
├─ Toujours disponible
├─ Préchauffé (templates, scripts)
├─ Réseau dédié vers backup storage
└─ Gain : -50% temps de setup

Cloud : Instance pré-configurée avec AMI/Snapshot
```

---

## ✅ Points clés à retenir

- **Tests réguliers** : Un backup non testé = Pas de backup (tester mensuellement minimum)
- **Validation systématique** : Cohérence, complétude, chronologie (règle des 3C)
- **Environnement isolé** : Toujours restaurer dans TEST avant PROD
- **Documentation** : Procédures à jour, testées et accessibles 24/7
- **Automatisation** : Tests automatisés de restauration avec alertes
- **RTO optimisé** : Pré-prepare, SSD, parallélisation, infra pré-provisionnée
- **PITR** : Full backup + binlogs = restauration à n'importe quel instant
- **Cloud-native** : S3 et VolumeSnapshots simpllifient disaster recovery
- **Checklist** : Valider technique + fonctionnel + applicatif après chaque restore
- **Communication** : Plan de communication clair (équipes, management, clients)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Backup and Restore - MariaDB KB](https://mariadb.com/kb/en/backup-and-restore-overview/)
- [📖 Point-in-Time Recovery - MariaDB KB](https://mariadb.com/kb/en/point-in-time-recovery-using-the-binary-log/)
- [📖 Mariabackup Restore - MariaDB KB](https://mariadb.com/kb/en/full-backup-and-restore-with-mariabackup/)

### Articles et guides

- [Disaster Recovery Best Practices - MariaDB Corporation](https://mariadb.com/resources/blog/disaster-recovery-best-practices/)
- [Testing Your Backups - Percona](https://www.percona.com/blog/testing-mysql-backups/)

---

## ➡️ Sections suivantes

Les sous-sections suivantes détailleront chaque procédure :

**[12.5.1 - Restauration complète](./05.1-restauration-complete.md)** : Procédures détaillées pour full restore logique et physique, troubleshooting.

**[12.5.2 - Point-in-Time Recovery (PITR)](./05.2-pitr.md)** : Guide complet PITR avec cas d'usage, exclusion de transactions, validation.

---


⏭️ [Restauration complète](/12-sauvegarde-restauration/05.1-restauration-complete.md)

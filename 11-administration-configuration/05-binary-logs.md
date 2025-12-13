🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5 Binary logs et logs de transactions

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** :
> - Section 11.4 (Gestion des logs)
> - Compréhension des transactions ACID
> - Concepts de réplication
> - Connaissances système fichiers

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et l'architecture des binary logs
- **Distinguer** les trois formats de binlog (STATEMENT, ROW, MIXED)
- **Configurer** les binary logs pour la réplication et le PITR
- **Gérer** la rotation et la purge des binary logs
- **Exploiter** les binary logs pour la restauration Point-in-Time
- **Optimiser** l'impact performance des binary logs
- **Sécuriser** les binary logs en production
- **Analyser** les binary logs avec mysqlbinlog

---

## Introduction

Les **binary logs** (binlogs) constituent l'un des **mécanismes les plus critiques** de MariaDB. Ils enregistrent toutes les modifications de données sous forme binaire, permettant :

- 🔄 **Réplication** : Synchronisation master-slave, Galera
- ⏱️ **Point-in-Time Recovery (PITR)** : Restauration précise après incident
- 📊 **Audit** : Traçabilité des modifications
- 🔌 **Change Data Capture (CDC)** : Intégration avec Kafka, data warehouses
- 📈 **Analytics** : Analyse des patterns de modification

### Pourquoi les binary logs sont essentiels

```
Sans binary logs:
    ❌ Pas de réplication
    ❌ Pas de PITR (restauration = dernier backup complet)
    ❌ Perte de données entre backups
    ❌ RTO élevé (Recovery Time Objective)

Avec binary logs:
    ✅ Réplication haute disponibilité
    ✅ PITR à la seconde près
    ✅ RPO quasi-nul (Recovery Point Objective)
    ✅ Audit complet des modifications
```

💡 **Principe fondamental** : En production, les binary logs ne sont **pas optionnels** pour tout système nécessitant haute disponibilité ou restauration précise.

---

## Architecture des binary logs

### Structure générale

```
/var/lib/mysql/
├── mysql-bin.000001        # Binary log file 1
├── mysql-bin.000002        # Binary log file 2
├── mysql-bin.000003        # Binary log file 3 (actif)
├── mysql-bin.index         # Index des binlogs
├── relay-log.000001        # Relay logs (slave uniquement)
├── relay-log.000002
└── relay-log.index
```

### Composants clés

#### 1. Fichiers binlog (mysql-bin.NNNNNN)

Fichiers binaires contenant les **événements** de modification :

```
mysql-bin.000001:
    [BEGIN]
    [INSERT INTO users VALUES (1, 'Alice')]
    [UPDATE orders SET status='shipped' WHERE id=123]
    [COMMIT]
    [BEGIN]
    [DELETE FROM sessions WHERE expired < NOW()]
    [COMMIT]
    ...
```

**Caractéristiques** :
- Format **binaire** (non lisible directement)
- Taille limitée par `max_binlog_size` (défaut : 1 GB)
- Rotation automatique à la taille max ou au redémarrage
- Numérotation séquentielle (000001, 000002, etc.)

#### 2. Fichier index (mysql-bin.index)

Liste tous les fichiers binlog actifs :

```
./mysql-bin.000001
./mysql-bin.000002
./mysql-bin.000003
```

⚠️ **CRITIQUE** : Ne **JAMAIS** modifier ce fichier manuellement. MariaDB le gère automatiquement.

#### 3. Relay logs (slave uniquement)

Sur les serveurs **replica/slave**, les binary logs du master sont copiés dans des **relay logs** avant d'être appliqués.

```
Master (binlog) → Network → Slave (relay log) → Application locale
```

---

## Formats de binary logs

MariaDB supporte **trois formats** de binary logs, chacun avec ses avantages et inconvénients.

### 1. STATEMENT (Basé sur les requêtes)

**Principe** : Enregistre les **requêtes SQL** exécutées.

```sql
-- Requête exécutée
UPDATE products SET price = price * 1.1 WHERE category = 'electronics';

-- Binlog STATEMENT
BEGIN;
UPDATE products SET price = price * 1.1 WHERE category = 'electronics';
COMMIT;
```

**Avantages** :
- ✅ **Compact** : Une requête = une entrée
- ✅ **Lisible** : Format SQL standard
- ✅ **Efficient** : Moins d'I/O disque

**Inconvénients** :
- ❌ **Non-déterministe** : `NOW()`, `RAND()`, `UUID()` donnent des résultats différents
- ❌ **Risque d'inconsistance** en réplication
- ❌ **Problèmes avec triggers, stored procedures**

**Exemple de problème** :

```sql
-- Sur le master à 10:00:00
INSERT INTO events (id, created_at) VALUES (1, NOW());
-- created_at = '2025-12-13 10:00:00'

-- Sur le slave, replay à 10:05:00
INSERT INTO events (id, created_at) VALUES (1, NOW());
-- created_at = '2025-12-13 10:05:00' ❌ DIFFÉRENT !
```

### 2. ROW (Basé sur les lignes)

**Principe** : Enregistre les **changements de lignes** (valeurs avant/après).

```sql
-- Requête exécutée
UPDATE products SET price = price * 1.1 WHERE category = 'electronics';
-- Affecte 1000 produits

-- Binlog ROW (simplifié)
BEGIN;
UPDATE products SET price=110.00 WHERE id=1 AND price=100.00;
UPDATE products SET price=220.00 WHERE id=2 AND price=200.00;
UPDATE products SET price=330.00 WHERE id=3 AND price=300.00;
... (1000 lignes)
COMMIT;
```

**Avantages** :
- ✅ **Déterministe** : Valeurs exactes, pas de fonctions non-déterministes
- ✅ **Fiable** : Garantit la cohérence master-slave
- ✅ **Précis** : Enregistre exactement ce qui a changé
- ✅ **Optimal pour CDC** : Extraction facile des changements

**Inconvénients** :
- ❌ **Volumineux** : Une ligne modifiée = une entrée
- ❌ **I/O élevé** : Plus d'écritures disque
- ❌ **Moins lisible** : Format binaire plus complexe

**Quand utiliser ROW** :
- ✅ Réplication critique nécessitant cohérence absolue
- ✅ Utilisation de fonctions non-déterministes (NOW(), RAND())
- ✅ Change Data Capture (CDC)
- ✅ Triggers complexes

### 3. MIXED (Hybride intelligent)

**Principe** : **Automatiquement** choisit entre STATEMENT et ROW selon le contexte.

**Logique de décision** :

```
Requête déterministe (UPDATE simple sans fonctions)
    → Format STATEMENT (compact)

Requête non-déterministe (NOW(), RAND(), UUID())
    → Format ROW (fiable)

Requête avec trigger/stored procedure
    → Format ROW (sécurisé)
```

**Avantages** :
- ✅ **Meilleur des deux mondes** : Compact quand possible, fiable quand nécessaire
- ✅ **Automatique** : Pas de décision manuelle
- ✅ **Recommandé** pour la plupart des cas

**Inconvénients** :
- ❌ **Moins prédictible** : Format mixte
- ❌ **Complexité** : Analyse plus difficile

### Comparaison des formats

| Critère | STATEMENT | ROW | MIXED |
|---------|-----------|-----|-------|
| **Taille** | ⭐⭐⭐ Compact | ⭐ Volumineux | ⭐⭐ Variable |
| **Fiabilité** | ⭐ Risques | ⭐⭐⭐ Parfait | ⭐⭐⭐ Excellent |
| **Performance** | ⭐⭐⭐ Rapide | ⭐⭐ Moyen | ⭐⭐⭐ Optimal |
| **Lisibilité** | ⭐⭐⭐ SQL | ⭐ Binaire | ⭐⭐ Mixte |
| **Réplication** | ⭐⭐ OK | ⭐⭐⭐ Parfait | ⭐⭐⭐ Parfait |
| **CDC** | ⭐ Difficile | ⭐⭐⭐ Idéal | ⭐⭐ Bon |

### Recommandation par cas d'usage

```ini
# Cas général (production)
binlog_format = MIXED

# Réplication critique (finance, santé)
binlog_format = ROW

# Legacy / compatibilité ancienne
binlog_format = STATEMENT
```

---

## Événements binlog

Les binary logs contiennent différents **types d'événements** :

### Types d'événements principaux

| Type | Description | Exemple |
|------|-------------|---------|
| **QUERY_EVENT** | Requête SQL (STATEMENT) | `UPDATE users SET ...` |
| **WRITE_ROWS_EVENT** | INSERT (ROW) | Nouvelles lignes |
| **UPDATE_ROWS_EVENT** | UPDATE (ROW) | Lignes modifiées |
| **DELETE_ROWS_EVENT** | DELETE (ROW) | Lignes supprimées |
| **XID_EVENT** | Commit transaction | Fin de transaction |
| **ROTATE_EVENT** | Rotation binlog | Nouveau fichier binlog |
| **FORMAT_DESCRIPTION_EVENT** | Métadata | Version, format |

### Structure d'un événement

```
┌────────────────────────────────────┐
│  HEADER                            │
│  - Timestamp                       │
│  - Event type                      │
│  - Server ID                       │
│  - Event length                    │
│  - Next position                   │
│  - Flags                           │
├────────────────────────────────────┤
│  EVENT DATA                        │
│  - SQL query (STATEMENT)           │
│  - Row changes (ROW)               │
│  - Transaction info                │
├────────────────────────────────────┤
│  CHECKSUM (optionnel)              │
│  - CRC32 pour détection corruption │
└────────────────────────────────────┘
```

---

## Binary logs et réplication

### Architecture de réplication avec binlog

```
┌─────────────────────────────────────────┐
│           MASTER                        │
├─────────────────────────────────────────┤
│                                         │
│  1. Exécution transaction               │
│       ↓                                 │
│  2. Écriture binary log                 │
│       ↓                                 │
│  3. Commit                              │
│                                         │
└─────────────────────────────────────────┘
           ↓ (I/O Thread)
┌─────────────────────────────────────────┐
│           SLAVE / REPLICA               │
├─────────────────────────────────────────┤
│                                         │
│  4. I/O Thread : Copie vers relay log   │
│       ↓                                 │
│  5. SQL Thread : Applique relay log     │
│       ↓                                 │
│  6. Écriture dans tables                │
│                                         │
└─────────────────────────────────────────┘
```

### Workflow détaillé

1. **Master** : Transaction commitée → écriture dans binlog
2. **Slave I/O Thread** : Lit le binlog du master → écrit dans relay log
3. **Slave SQL Thread** : Lit le relay log → exécute les événements
4. **Slave** : Met à jour `Exec_Master_Log_Pos` (position courante)

### Variables critiques de réplication

```sql
-- Sur le MASTER
SHOW VARIABLES LIKE 'log_bin';                    -- ON
SHOW VARIABLES LIKE 'server_id';                  -- Unique (ex: 1)
SHOW VARIABLES LIKE 'binlog_format';              -- MIXED/ROW/STATEMENT

-- Sur le SLAVE
SHOW VARIABLES LIKE 'server_id';                  -- Unique (ex: 2)
SHOW VARIABLES LIKE 'relay_log';                  -- Activé
SHOW SLAVE STATUS\G                               -- État réplication
```

---

## Point-in-Time Recovery (PITR)

### Principe du PITR

Le **Point-in-Time Recovery** permet de restaurer une base de données à un **instant précis** en combinant :

1. **Backup complet** (snapshot à T0)
2. **Binary logs** (modifications de T0 à Tn)

```
Backup complet      Incident
    ↓                  ↓
    T0─────────────────Tn────→ Temps
    │                  │
    └──────binlogs─────┘
           ↓
    Restauration à Tn-1 (juste avant l'incident)
```

### Scénario PITR typique

**Situation** : Suppression accidentelle de données à 14:30.

```sql
-- 14:30 : ERREUR
DELETE FROM orders WHERE customer_id > 0;  -- Supprime TOUTES les commandes !
```

**Solution** :

```bash
# 1. Restaurer le backup complet (ex: 00:00)
mariabackup --copy-back --target-dir=/backup/full-2025-12-13

# 2. Appliquer les binlogs de 00:00 à 14:29:59
mysqlbinlog --start-datetime='2025-12-13 00:00:00' \
            --stop-datetime='2025-12-13 14:29:59' \
            /var/lib/mysql/mysql-bin.* | mariadb

# 3. Vérification
mariadb -e "SELECT COUNT(*) FROM orders"
# Données restaurées jusqu'à 14:29:59 ✅
```

### PITR par position binlog

Alternative plus précise que les timestamps :

```bash
# 1. Identifier la position du problème
mysqlbinlog /var/lib/mysql/mysql-bin.000123 | grep -B 10 "DELETE FROM orders"
# Position: 1234567

# 2. Restaurer jusqu'à cette position (exclue)
mysqlbinlog --start-position=0 \
            --stop-position=1234567 \
            /var/lib/mysql/mysql-bin.000123 | mariadb
```

💡 **Avantage position vs datetime** : Plus précis (évite les doublons si plusieurs transactions à la même seconde).

---

## Gestion et rotation des binary logs

### Rotation automatique

Les binary logs **rotent automatiquement** dans ces cas :

1. **Taille max atteinte** (`max_binlog_size`)
2. **Redémarrage** du serveur
3. **FLUSH LOGS** manuel
4. **Changement de format** binlog

```sql
-- Forcer une rotation
FLUSH BINARY LOGS;

-- Vérifier les binlogs
SHOW BINARY LOGS;
```

**Sortie exemple** :

```
+-------------------+-----------+
| Log_name          | File_size |
+-------------------+-----------+
| mysql-bin.000001  | 1073741824|  -- 1 GB
| mysql-bin.000002  | 536870912 |  -- 512 MB
| mysql-bin.000003  | 125829120 |  -- 120 MB (actif)
+-------------------+-----------+
```

### Purge des binary logs

⚠️ **DANGER** : Les binary logs **ne sont PAS purgés automatiquement** par défaut (sauf si `expire_logs_days` est configuré).

#### Purge automatique

```ini
# my.cnf - Purge auto après 7 jours
[mysqld]
expire_logs_days = 7
```

```sql
-- Vérifier la configuration
SHOW VARIABLES LIKE 'expire_logs_days';
```

#### Purge manuelle

```sql
-- Lister les binlogs
SHOW BINARY LOGS;

-- Purger avant une date
PURGE BINARY LOGS BEFORE '2025-12-01 00:00:00';

-- Purger jusqu'à un fichier spécifique
PURGE BINARY LOGS TO 'mysql-bin.000120';

-- Purger tous sauf les N derniers
-- (nécessite script externe)
```

**Script de purge intelligent** :

```bash
#!/bin/bash
# Garder uniquement les 7 derniers jours de binlogs

RETENTION_DAYS=7
PURGE_DATE=$(date -d "$RETENTION_DAYS days ago" '+%Y-%m-%d 00:00:00')

mariadb -e "PURGE BINARY LOGS BEFORE '$PURGE_DATE';"
```

⚠️ **ATTENTION** : Ne **JAMAIS** purger les binlogs nécessaires à la réplication !

```sql
-- Sur le MASTER, vérifier la position des SLAVES
SHOW SLAVE HOSTS;

-- Sur chaque SLAVE, vérifier quelle position est lue
SHOW SLAVE STATUS\G
-- Relay_Master_Log_File: mysql-bin.000115

-- Ne purger QUE les binlogs < 000115
```

### Suppression INTERDITE

```bash
# ❌ JAMAIS FAIRE ÇA
rm /var/lib/mysql/mysql-bin.000*

# Conséquences:
# - Corruption du fichier index
# - Réplication cassée
# - Impossible de faire PITR
```

**Toujours utiliser** `PURGE BINARY LOGS` !

---

## Impact performance et optimisation

### Overhead des binary logs

**Mesure de l'impact** :

| Configuration | TPS (Transactions/sec) | Impact |
|---------------|------------------------|--------|
| Binlog OFF | 10,000 | Baseline |
| Binlog ON + STATEMENT | 9,500 | -5% |
| Binlog ON + ROW | 8,500 | -15% |
| Binlog ON + ROW + sync_binlog=1 | 6,000 | -40% |

### Variables d'optimisation

#### 1. sync_binlog

Contrôle la **fréquence de synchronisation** sur disque.

```ini
# my.cnf
[mysqld]
# Sécurité MAXIMALE (par défaut depuis MariaDB 10.5)
sync_binlog = 1          # Flush à chaque commit
# Performance : Excellent pour durabilité
# Risque : Aucun (perte de données impossible)

# Compromis (haute performance)
sync_binlog = 100        # Flush tous les 100 commits
# Performance : Meilleure
# Risque : Perte max 100 transactions en cas de crash

# Performance MAXIMALE (dangereux)
sync_binlog = 0          # OS décide quand flusher
# Performance : Optimale
# Risque : Perte de données en crash système
```

**Recommandation** :

- **Production critique** : `sync_binlog = 1`
- **Haute performance acceptable** : `sync_binlog = 10-100`
- **Développement/test** : `sync_binlog = 0`

#### 2. binlog_cache_size

Cache pour les transactions **avant** écriture dans le binlog.

```ini
# my.cnf
[mysqld]
binlog_cache_size = 4M         # 4 MB par transaction
max_binlog_cache_size = 1G     # Max 1 GB par transaction
```

```sql
-- Surveiller l'utilisation du cache
SHOW STATUS LIKE 'Binlog_cache%';
```

**Sortie exemple** :

```
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Binlog_cache_disk_use      | 1250  |  -- Transactions trop grosses (spillées sur disque)
| Binlog_cache_use           | 125000|  -- Transactions utilisant le cache
+----------------------------+-------+
```

**Optimisation** :

```
Si Binlog_cache_disk_use > 1% de Binlog_cache_use
    → Augmenter binlog_cache_size
```

#### 3. Disque dédié pour binlogs

```ini
# my.cnf
[mysqld]
# Binlogs sur disque SSD dédié
log_bin = /mnt/binlogs/mysql-bin

# Datadir sur disque principal
datadir = /var/lib/mysql
```

**Avantages** :
- ✅ Réduit contention I/O
- ✅ Meilleure performance écriture
- ✅ Isolation des charges

---

## Sécurité des binary logs

### Chiffrement des binary logs

MariaDB 10.1.7+ supporte le **chiffrement des binary logs**.

```ini
# my.cnf
[mysqld]
# Activer chiffrement binlog
encrypt_binlog = ON

# Plugin de chiffrement (nécessaire)
plugin_load_add = file_key_management
file_key_management_filename = /etc/mysql/encryption/keyfile.enc
file_key_management_encryption_algorithm = AES_CBC
```

**Vérification** :

```sql
SHOW VARIABLES LIKE 'encrypt_binlog';
-- encrypt_binlog = ON
```

### Contrôle d'accès

```bash
# Permissions strictes sur les binlogs
chmod 640 /var/lib/mysql/mysql-bin.*
chown mysql:mysql /var/lib/mysql/mysql-bin.*

# Vérifier
ls -la /var/lib/mysql/mysql-bin.*
# -rw-r----- 1 mysql mysql
```

### Audit des accès

```sql
-- Activer l'audit des accès binlog (Server Audit Plugin)
INSTALL SONAME 'server_audit';

SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_events = 'QUERY_DDL,QUERY_DML';
```

---

## Outils d'analyse mysqlbinlog

### Installation

```bash
# Inclus par défaut avec MariaDB
which mysqlbinlog
# /usr/bin/mysqlbinlog
```

### Utilisation de base

```bash
# Afficher un binlog (format binaire → SQL)
mysqlbinlog /var/lib/mysql/mysql-bin.000123

# Filtrer par base de données
mysqlbinlog --database=ecommerce /var/lib/mysql/mysql-bin.000123

# Filtrer par période
mysqlbinlog --start-datetime='2025-12-13 10:00:00' \
            --stop-datetime='2025-12-13 12:00:00' \
            /var/lib/mysql/mysql-bin.*

# Filtrer par position
mysqlbinlog --start-position=1000 \
            --stop-position=5000 \
            /var/lib/mysql/mysql-bin.000123
```

### Options avancées

```bash
# Format ROW : Afficher les valeurs décodes
mysqlbinlog --base64-output=DECODE-ROWS \
            --verbose \
            /var/lib/mysql/mysql-bin.000123

# Extraire uniquement les requêtes (sans metadata)
mysqlbinlog --short-form /var/lib/mysql/mysql-bin.000123

# Désactiver Base64 (STATEMENT uniquement)
mysqlbinlog --base64-output=NEVER /var/lib/mysql/mysql-bin.000123
```

### Analyse de taille

```bash
# Taille totale des binlogs
du -sh /var/lib/mysql/mysql-bin.*

# Requête SQL pour taille détaillée
mariadb -e "
SELECT
    LOG_NAME,
    ROUND(FILE_SIZE / 1024 / 1024, 2) AS size_mb
FROM information_schema.BINARY_LOGS
ORDER BY LOG_NAME;
"
```

---

## Bonnes pratiques de production

### 1. Toujours activer les binary logs

```ini
# my.cnf - TOUJOURS en production
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
binlog_format = MIXED
expire_logs_days = 7
max_binlog_size = 1G
sync_binlog = 1  # Ou 10-100 selon besoin performance
```

### 2. Surveiller l'espace disque

```bash
# Script de monitoring quotidien
#!/bin/bash
BINLOG_DIR="/var/lib/mysql"
THRESHOLD=80  # Alerte si > 80%

USAGE=$(df -h $BINLOG_DIR | awk 'NR==2 {print $5}' | sed 's/%//')

if [ $USAGE -gt $THRESHOLD ]; then
    echo "ALERTE: Binlogs consomment ${USAGE}% du disque" | \
        mail -s "MariaDB Binlog Space Alert" dba@example.com
fi
```

### 3. Sauvegarder les binary logs

```bash
# Backup des binlogs vers stockage distant
#!/bin/bash
BINLOG_DIR="/var/lib/mysql"
BACKUP_DIR="/backup/binlogs/$(date +%Y-%m-%d)"

mkdir -p $BACKUP_DIR

# Copier tous les binlogs sauf le dernier (actif)
CURRENT_BINLOG=$(mariadb -sN -e "SHOW MASTER STATUS" | awk '{print $1}')

for binlog in $(ls -1 $BINLOG_DIR/mysql-bin.* | grep -v "$CURRENT_BINLOG" | grep -v ".index"); do
    cp $binlog $BACKUP_DIR/
done

# Compresser
gzip $BACKUP_DIR/*.bin
```

### 4. Tester la restauration PITR régulièrement

```bash
# Tous les mois : Test PITR sur environnement staging
# 1. Restaurer backup complet
# 2. Appliquer binlogs
# 3. Vérifier intégrité données
```

### 5. Documenter la stratégie de rétention

```ini
# my.cnf - Configuration documentée
[mysqld]
# Binary logs : 7 jours de rétention
# Justification :
#   - Backups complets quotidiens
#   - RPO : 24h max
#   - Espace disque : 200 GB disponibles
#   - Binlog moyen : 20 GB/jour
expire_logs_days = 7
```

---

## Troubleshooting

### Problème : Binlogs remplissent le disque

**Symptôme** : Erreur "No space left on device".

**Diagnostic** :

```bash
df -h /var/lib/mysql
du -sh /var/lib/mysql/mysql-bin.*
```

**Solutions** :

```sql
-- 1. Purger immédiatement les vieux binlogs
PURGE BINARY LOGS BEFORE DATE(NOW() - INTERVAL 3 DAY);

-- 2. Réduire la rétention
SET GLOBAL expire_logs_days = 3;

-- 3. Désactiver temporairement (DANGER)
-- Uniquement si urgence critique
SET GLOBAL sql_log_bin = 0;  -- Session uniquement
```

### Problème : Réplication cassée après purge

**Symptôme** : Slave en erreur "Could not find binlog".

**Cause** : Binlog purgé alors que slave n'avait pas encore lu.

**Solution** :

```sql
-- Sur le slave : Réinitialiser depuis nouvelle position
STOP SLAVE;
RESET SLAVE;

-- Obtenir position actuelle du master
-- (nécessite un nouveau backup ou GTID)
CHANGE MASTER TO
    MASTER_LOG_FILE='mysql-bin.000125',
    MASTER_LOG_POS=4;

START SLAVE;
```

💡 **Prévention** : Toujours vérifier les positions slaves avant purge.

### Problème : Corruption binlog

**Symptôme** : Erreur "Binlog has bad magic number".

**Diagnostic** :

```bash
# Vérifier le fichier
mysqlbinlog /var/lib/mysql/mysql-bin.000123
```

**Solution** :

```bash
# 1. Identifier le fichier corrompu
# 2. Supprimer uniquement ce fichier ET les suivants
# 3. Restaurer depuis backup + binlogs sains
```

---

## ✅ Points clés à retenir

- **Rôle critique** : Binary logs = réplication + PITR + audit
- **3 formats** : STATEMENT (compact), ROW (fiable), MIXED (recommandé)
- **Activation** : `log_bin = /var/lib/mysql/mysql-bin` dans my.cnf
- **Rotation** : Automatique à `max_binlog_size` (défaut 1 GB)
- **Purge** : TOUJOURS avec `PURGE BINARY LOGS`, JAMAIS avec `rm`
- **Rétention** : `expire_logs_days = 7` (adapter selon besoin)
- **Performance** : `sync_binlog = 1` (sécurité) vs `sync_binlog = 100` (performance)
- **PITR** : Backup complet + binlogs = restauration à la seconde
- **Réplication** : Binlogs copiés vers relay logs puis appliqués
- **Sécurité** : Chiffrement (`encrypt_binlog = ON`), permissions 640
- **Monitoring** : Surveiller espace disque et purger régulièrement
- **Outils** : `mysqlbinlog` pour analyse, `SHOW BINARY LOGS` pour liste
- **Production** : Binary logs NON OPTIONNELS pour HA et PITR

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Binary Log](https://mariadb.com/kb/en/binary-log/)
- [📖 Documentation officielle - mysqlbinlog](https://mariadb.com/kb/en/mysqlbinlog/)
- [📖 Documentation officielle - Replication](https://mariadb.com/kb/en/replication/)
- [📖 Documentation officielle - Point-in-Time Recovery](https://mariadb.com/kb/en/point-in-time-recovery-using-mariabackup/)
- [📖 Binary Log Formats](https://mariadb.com/kb/en/binary-log-formats/)
- [🔧 Percona Toolkit - pt-show-grants](https://www.percona.com/doc/percona-toolkit/)

---

## ➡️ Sections suivantes

- **11.5.1 Configuration binlog** : Paramètres détaillés, optimisations
- **11.5.2 Formats : STATEMENT, ROW, MIXED** : Comparaison approfondie
- **11.5.3 Purge et rotation** : Gestion du cycle de vie

---

**💡 Conseil final** : Les binary logs sont votre **filet de sécurité**. Ne les désactivez jamais en production, surveillez l'espace disque, testez régulièrement vos procédures PITR. La question n'est pas "si" vous en aurez besoin, mais "quand" ! 🛡️⏱️

⏭️ [Configuration binlog](/11-administration-configuration/05.1-configuration-binlog.md)

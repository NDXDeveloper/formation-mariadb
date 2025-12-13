🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4 Gestion des logs

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** :
> - Sections 11.1-11.3 (Configuration, variables, sql_mode)
> - Connaissance système Unix/Linux
> - Compréhension des systèmes de fichiers

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les différents types de logs MariaDB et leur rôle
- **Configurer** les logs de manière optimale pour la production
- **Analyser** les logs pour le diagnostic et l'optimisation
- **Gérer** la rotation et l'archivage des logs
- **Optimiser** l'impact performance des logs
- **Exploiter** les logs pour l'audit et la conformité
- **Utiliser** les outils d'analyse de logs

---

## Introduction

Les **logs** sont les yeux et les oreilles de votre serveur MariaDB. Ils constituent votre **première ligne de défense** pour :

- 🔍 **Diagnostic** : Identifier et résoudre les problèmes
- ⚡ **Performance** : Détecter les requêtes lentes
- 🔒 **Sécurité** : Tracer les activités suspectes
- 🔄 **Réplication** : Assurer la cohérence des données
- 📊 **Audit** : Conformité réglementaire (RGPD, SOC2, etc.)
- 🕐 **Point-in-Time Recovery** : Restauration précise après incident

### Architecture de logging MariaDB

```
┌─────────────────────────────────────────┐
│         SERVEUR MariaDB                 │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────┐  ┌─────────────────┐  │
│  │  Error Log   │  │  Slow Query Log │  │
│  │  (Erreurs)   │  │  (Performance)  │  │
│  └──────────────┘  └─────────────────┘  │
│                                         │
│  ┌──────────────┐  ┌─────────────────┐  │
│  │ General Log  │  │   Binary Log    │  │
│  │ (Debug)      │  │  (Réplication)  │  │
│  └──────────────┘  └─────────────────┘  │
│                                         │
└─────────────────────────────────────────┘
           ↓         ↓         ↓
    /var/log/mysql/*.log
```

---

## Les quatre types de logs principaux

### Vue d'ensemble comparative

| Type de log | Usage principal | Impact performance | Taille | Production |
|-------------|-----------------|-------------------|--------|------------|
| **Error Log** | Erreurs, warnings, démarrage/arrêt | ✅ Négligeable | Faible | **Toujours ON** |
| **Slow Query Log** | Optimisation requêtes lentes | ⚠️ Faible-Moyen | Moyen | **ON (recommandé)** |
| **General Log** | Debug toutes requêtes | ❌ Impact élevé | Très élevé | **OFF (sauf debug)** |
| **Binary Log** | Réplication, PITR | ⚠️ Moyen | Très élevé | **ON (HA/PITR)** |

### Error Log

**Rôle** : Enregistre les erreurs, warnings et événements système.

**Contenu typique** :
- Démarrage et arrêt du serveur
- Erreurs de connexion
- Problèmes InnoDB
- Warnings de configuration
- Crashs et corruptions

**Activation** : **Toujours activé** par défaut.

```ini
# Configuration my.cnf
[mysqld]
log_error = /var/log/mysql/error.log
```

**Exemple d'entrée** :

```
2025-12-13 10:30:15 0 [Note] Server socket created on IP: '0.0.0.0'.
2025-12-13 10:30:15 0 [Note] InnoDB: Buffer pool(s) load completed at 251213 10:30:15
2025-12-13 10:30:15 0 [Note] /usr/sbin/mariadbd: ready for connections.
2025-12-13 11:15:42 42 [Warning] Aborted connection 42 to db: 'unconnected' user: 'root' host: '192.168.1.100'
```

💡 **Bonne pratique** : Surveiller quotidiennement l'error log avec `tail -f` ou un système de monitoring.

### Slow Query Log

**Rôle** : Identifie les requêtes **dépassant un seuil de temps** configuré.

**Contenu typique** :
- Requêtes lentes (> `long_query_time`)
- Requêtes sans index (si `log_queries_not_using_indexes = ON`)
- Plan d'exécution (avec `log_slow_verbosity`)
- Temps d'exécution, lignes examinées/renvoyées

**Configuration** :

```ini
# my.cnf
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2                    # Seuil : 2 secondes
log_queries_not_using_indexes = 0      # OFF en production (trop verbeux)
log_slow_verbosity = query_plan,explain
```

**Exemple d'entrée** :

```
# Time: 2025-12-13T11:20:15.123456Z
# User@Host: app_user[app_user] @ app-server [10.0.1.50]
# Thread_id: 12345  Schema: ecommerce  QC_hit: No
# Query_time: 5.234567  Lock_time: 0.000234  Rows_sent: 1250  Rows_examined: 1500000
SET timestamp=1702468815;
SELECT * FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = 'pending'
ORDER BY o.created_at DESC;
```

**Cas d'usage** :
- ✅ Optimisation performance continue
- ✅ Détection de régressions
- ✅ Identification des requêtes nécessitant des index

⚠️ **Attention** : En environnement haute charge, le slow query log peut **devenir volumineux**. Dimensionner l'espace disque en conséquence.

### General Log

**Rôle** : Enregistre **TOUTES** les requêtes SQL exécutées (connexions, requêtes, déconnexions).

**Contenu typique** :
- Tentatives de connexion (succès/échec)
- Toutes les requêtes SELECT, INSERT, UPDATE, DELETE
- Commandes administratives
- Déconnexions

**Configuration** :

```ini
# my.cnf (DÉSACTIVÉ par défaut et recommandé)
[mysqld]
general_log = 0                        # OFF
general_log_file = /var/log/mysql/general.log
```

**Activation temporaire** (debug uniquement) :

```sql
-- Activer pour debug
SET GLOBAL general_log = ON;

-- Reproduire le problème...

-- Désactiver immédiatement
SET GLOBAL general_log = OFF;
```

**Exemple d'entrée** :

```
2025-12-13T11:25:30.123456Z    12345 Connect   app_user@10.0.1.50 on ecommerce
2025-12-13T11:25:30.234567Z    12345 Query     SELECT @@version_comment LIMIT 1
2025-12-13T11:25:30.345678Z    12345 Query     SELECT * FROM products WHERE category = 'electronics'
2025-12-13T11:25:30.456789Z    12345 Query     UPDATE cart SET quantity = 2 WHERE id = 1234
2025-12-13T11:25:30.567890Z    12345 Quit
```

❌ **DANGER** : Le General Log a un **impact performance majeur** et génère des **fichiers énormes**.

**Règle d'or** :
- ✅ Utiliser UNIQUEMENT pour debug ponctuel (15-30 min max)
- ❌ JAMAIS activer en production continue
- ⚠️ Contient des données sensibles (mots de passe en clair dans certaines requêtes)

### Binary Log (binlog)

**Rôle** : Enregistre **toutes les modifications** de données pour la réplication et le Point-in-Time Recovery.

**Contenu typique** :
- INSERT, UPDATE, DELETE (selon format)
- CREATE, ALTER, DROP
- Événements de transaction (BEGIN, COMMIT, ROLLBACK)
- Metadata (timestamps, server_id, etc.)

**Configuration** :

```ini
# my.cnf
[mysqld]
log_bin = /var/log/mysql/mysql-bin
binlog_format = MIXED                  # ROW, STATEMENT, ou MIXED
expire_logs_days = 7                   # Purge automatique après 7 jours
max_binlog_size = 1G                   # Rotation à 1 GB
```

**Formats disponibles** :

| Format | Avantages | Inconvénients |
|--------|-----------|---------------|
| **STATEMENT** | Compact, lisible | Non-déterministe (NOW(), RAND()) |
| **ROW** | Déterministe, précis | Taille importante |
| **MIXED** | Hybride intelligent | Complexité accrue |

**Cas d'usage** :
- ✅ Réplication Master-Slave / Galera
- ✅ Point-in-Time Recovery (PITR)
- ✅ Audit des modifications
- ✅ CDC (Change Data Capture) pour data warehousing

**Consultation** :

```bash
# Lister les binary logs
mariadb -e "SHOW BINARY LOGS;"

# Examiner le contenu (format lisible)
mysqlbinlog /var/log/mysql/mysql-bin.000123
```

---

## Configuration générale des logs

### Emplacement des fichiers

```ini
# my.cnf - Configuration centralisée
[mysqld]
# Répertoire des logs
log_error = /var/log/mysql/error.log
slow_query_log_file = /var/log/mysql/slow-query.log
general_log_file = /var/log/mysql/general.log
log_bin = /var/log/mysql/mysql-bin

# Alternative : Logging dans des tables système
# log_output = TABLE  # Logs dans mysql.general_log et mysql.slow_log
```

### Permissions et propriété

```bash
# Créer le répertoire des logs
sudo mkdir -p /var/log/mysql

# Permissions appropriées
sudo chown -R mysql:mysql /var/log/mysql
sudo chmod 750 /var/log/mysql

# Vérifier
ls -la /var/log/mysql
# drwxr-x--- 2 mysql mysql 4096 Dec 13 10:00 mysql
```

### Logging vers tables vs fichiers

MariaDB peut logger vers des **tables** (mysql.general_log, mysql.slow_log) ou des **fichiers**.

```sql
-- Vérifier la destination actuelle
SHOW VARIABLES LIKE 'log_output';

-- Changer vers tables
SET GLOBAL log_output = 'TABLE';

-- Changer vers fichiers (recommandé)
SET GLOBAL log_output = 'FILE';

-- Les deux simultanément
SET GLOBAL log_output = 'FILE,TABLE';
```

**Comparaison** :

| Destination | Avantages | Inconvénients |
|-------------|-----------|---------------|
| **FILE** | Performance, rotation simple | Nécessite outils externes (grep, awk) |
| **TABLE** | Requêtes SQL pour analyse | Impact performance, croissance illimitée |

💡 **Recommandation production** : Préférez **FILE** avec des outils d'analyse dédiés (pt-query-digest, logrotate).

---

## Bonnes pratiques de gestion des logs

### 1. Centraliser les logs

```bash
# Structure recommandée
/var/log/mysql/
├── error.log
├── slow-query.log
├── mysql-bin.000001
├── mysql-bin.000002
├── mysql-bin.index
└── archived/
    ├── error.log.2025-12-12.gz
    └── slow-query.log.2025-12-12.gz
```

### 2. Rotation automatique avec logrotate

```bash
# /etc/logrotate.d/mariadb
/var/log/mysql/*.log {
    daily                    # Rotation quotidienne
    rotate 7                 # Garder 7 jours
    compress                 # Compresser avec gzip
    delaycompress           # Compresser le jour suivant
    missingok               # Pas d'erreur si fichier absent
    notifempty              # Ne pas rotater si vide
    create 640 mysql mysql  # Permissions nouveaux fichiers
    sharedscripts
    postrotate
        # Forcer MariaDB à fermer et rouvrir les logs
        if test -x /usr/bin/mysqladmin && \
           /usr/bin/mysqladmin ping &>/dev/null
        then
           /usr/bin/mysqladmin flush-logs
        fi
    endscript
}
```

**Test de la configuration** :

```bash
# Tester sans exécuter
sudo logrotate -d /etc/logrotate.d/mariadb

# Forcer une rotation (test)
sudo logrotate -f /etc/logrotate.d/mariadb
```

### 3. Purge des binary logs

```sql
-- Lister les binary logs
SHOW BINARY LOGS;

-- Purger les logs avant une date
PURGE BINARY LOGS BEFORE '2025-12-01 00:00:00';

-- Purger tous les logs sauf les N derniers
PURGE BINARY LOGS TO 'mysql-bin.000120';

-- Vérification
SHOW BINARY LOGS;
```

⚠️ **DANGER** : Ne **JAMAIS** supprimer les binary logs manuellement (avec `rm`). Utiliser toujours `PURGE BINARY LOGS`.

**Raison** : MariaDB maintient un fichier index (`mysql-bin.index`). Supprimer manuellement crée une **désynchronisation**.

### 4. Surveillance de l'espace disque

```bash
# Script de surveillance quotidienne
#!/bin/bash
LOG_DIR="/var/log/mysql"
THRESHOLD=80  # Alerte si > 80%

USAGE=$(df -h $LOG_DIR | awk 'NR==2 {print $5}' | sed 's/%//')

if [ $USAGE -gt $THRESHOLD ]; then
    echo "ALERTE: Disque logs à ${USAGE}%" | mail -s "MariaDB Logs Full" dba@example.com
fi
```

### 5. Monitoring proactif

```sql
-- Taille des binary logs
SELECT
    CONCAT(ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2), ' GB') AS total_binlog_size,
    COUNT(*) AS nb_binlogs
FROM information_schema.binary_logs;

-- Ancienneté du plus vieux binlog
SELECT
    LOG_NAME,
    CREATED AS oldest_binlog_date,
    TIMESTAMPDIFF(DAY, CREATED, NOW()) AS days_old
FROM information_schema.binary_logs
ORDER BY CREATED ASC
LIMIT 1;
```

---

## Impact performance des logs

### Benchmarking de l'impact

| Configuration | Requêtes/sec | Impact |
|---------------|--------------|--------|
| Tous logs OFF | 10,000 | Baseline (100%) |
| Error log ON | 9,950 | -0.5% |
| + Slow query log | 9,800 | -2% |
| + Binary log (ROW) | 8,500 | -15% |
| + General log | 4,000 | **-60%** ❌ |

### Optimisations

#### 1. Binary log sur disque séparé

```ini
# my.cnf
[mysqld]
# Binlogs sur disque dédié (SSD recommandé)
log_bin = /mnt/binlogs/mysql-bin
```

**Avantage** : Réduit la contention I/O avec le datadir.

#### 2. Async binlog flushing

```ini
# my.cnf
[mysqld]
sync_binlog = 0      # Dangereux : perte possible en crash
# OU
sync_binlog = 100    # Compromis : flush tous les 100 commits
# OU
sync_binlog = 1      # Sécurisé : flush à chaque commit (par défaut)
```

**Trade-off** :
- `sync_binlog = 1` : **Sécurité maximale**, performance réduite
- `sync_binlog = 0` : **Performance maximale**, risque de perte de données
- `sync_binlog = 100` : **Compromis** raisonnable

#### 3. Slow query log sélectif

```sql
-- Ajuster le seuil selon la charge
SET GLOBAL long_query_time = 5;  -- 5 secondes au lieu de 2

-- Désactiver temporairement pendant batch
SET GLOBAL slow_query_log = OFF;
-- Exécuter batch...
SET GLOBAL slow_query_log = ON;
```

---

## Outils d'analyse de logs

### pt-query-digest (Percona Toolkit)

**Installation** :

```bash
# Ubuntu/Debian
sudo apt-get install percona-toolkit

# CentOS/RHEL
sudo yum install percona-toolkit
```

**Analyse du slow query log** :

```bash
# Analyse complète
pt-query-digest /var/log/mysql/slow-query.log

# Top 10 requêtes les plus lentes
pt-query-digest /var/log/mysql/slow-query.log --limit 10

# Exporter en HTML
pt-query-digest /var/log/mysql/slow-query.log --output html > report.html

# Analyser une période spécifique
pt-query-digest /var/log/mysql/slow-query.log \
    --since '2025-12-13 10:00:00' \
    --until '2025-12-13 12:00:00'
```

**Sortie exemple** :

```
# Overall: 1.5k total, 25 unique, 0.12 QPS, 0.05x concurrency
# Time range: 2025-12-13T10:00:00 to 2025-12-13T12:00:00
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           125s     1ms     45s    83ms   500ms     5s    20ms
# Lock time            12s     0us    500ms     8ms    50ms    25ms     5ms
# Rows sent        1.25M       0   500k    850   10.5k   5.25k     150
# Rows examine     125M       0    50M   85.3k     1M   500k   15.5k
```

### mysqldumpslow (Natif MariaDB)

```bash
# Top 10 requêtes par temps total
mysqldumpslow -s t -t 10 /var/log/mysql/slow-query.log

# Top 10 requêtes par nombre d'occurrences
mysqldumpslow -s c -t 10 /var/log/mysql/slow-query.log

# Filtrer par requêtes avec au moins 1000 lignes examinées
mysqldumpslow -s t -t 10 -g "Rows_examined: [1-9][0-9]{3,}" /var/log/mysql/slow-query.log
```

### mysqlbinlog (Binary logs)

```bash
# Lire un binary log
mysqlbinlog /var/log/mysql/mysql-bin.000123

# Format lisible (avec timestamps)
mysqlbinlog --base64-output=DECODE-ROWS -v /var/log/mysql/mysql-bin.000123

# Filtrer par base de données
mysqlbinlog --database=ecommerce /var/log/mysql/mysql-bin.000123

# Filtrer par période
mysqlbinlog --start-datetime='2025-12-13 10:00:00' \
            --stop-datetime='2025-12-13 12:00:00' \
            /var/log/mysql/mysql-bin.000123

# Extraire les requêtes pour replay
mysqlbinlog /var/log/mysql/mysql-bin.000123 > replay.sql
```

### Analyse en temps réel

```bash
# Surveiller l'error log en temps réel
tail -f /var/log/mysql/error.log

# Filtrer les erreurs seulement
tail -f /var/log/mysql/error.log | grep -i error

# Surveiller le slow query log
tail -f /var/log/mysql/slow-query.log

# Compter les erreurs en temps réel
tail -f /var/log/mysql/error.log | grep -i error | wc -l
```

---

## Sécurité et conformité

### Protection des logs sensibles

```bash
# Permissions restrictives
chmod 640 /var/log/mysql/*.log
chown mysql:mysql /var/log/mysql/*.log

# Audit des accès (avec auditd sur Linux)
auditctl -w /var/log/mysql/general.log -p rwa -k mysql_logs
```

### Anonymisation des données

```sql
-- Créer une vue anonymisée du slow log (si log_output=TABLE)
CREATE VIEW slow_log_anonymized AS
SELECT
    start_time,
    user_host,
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    db,
    REGEXP_REPLACE(sql_text, "'[^']*'", "'***'") AS sql_text_anonymized
FROM mysql.slow_log;
```

### Conformité RGPD

⚠️ **Attention** : Les logs peuvent contenir des **données personnelles** :
- Adresses IP dans les connexions
- Données sensibles dans les requêtes (emails, noms, etc.)

**Mesures** :
1. Définir une **politique de rétention** (ex: 30 jours max)
2. **Chiffrer** les logs archivés
3. **Restreindre** l'accès aux logs (principe du moindre privilège)
4. **Anonymiser** ou supprimer les données sensibles

```bash
# Chiffrer les archives
gzip /var/log/mysql/slow-query.log.2025-12-12
openssl enc -aes-256-cbc -salt \
    -in /var/log/mysql/slow-query.log.2025-12-12.gz \
    -out /var/log/mysql/slow-query.log.2025-12-12.gz.enc
rm /var/log/mysql/slow-query.log.2025-12-12.gz
```

---

## Stratégie de logging par environnement

### Production

```ini
# my.cnf - Production
[mysqld]
# Error log : TOUJOURS
log_error = /var/log/mysql/error.log

# Slow query : OUI (optimisation continue)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2
log_slow_verbosity = query_plan

# General log : NON (trop d'impact)
general_log = 0

# Binary log : OUI (réplication + PITR)
log_bin = /var/log/mysql/mysql-bin
binlog_format = MIXED
expire_logs_days = 7
max_binlog_size = 1G
```

### Développement

```ini
# my.cnf - Développement
[mysqld]
log_error = /var/log/mysql/error.log

# Slow query : très sensible (tout > 0.5s)
slow_query_log = 1
long_query_time = 0.5
log_queries_not_using_indexes = 1  # Détecter requêtes sans index

# General log : selon besoin
general_log = 0

# Binary log : optionnel (pas de réplication en dev)
# log_bin = /var/log/mysql/mysql-bin
```

### Staging

```ini
# my.cnf - Staging (réplique production)
[mysqld]
# Configuration identique à production
# pour détecter les problèmes avant déploiement
```

---

## Cas d'usage avancés

### 1. Audit de sécurité

Activer temporairement le general log pour tracer une activité suspecte :

```sql
-- Activer uniquement pour une session spécifique (impossible natif)
-- Alternative : Activer globalement brièvement

SET GLOBAL general_log = ON;
-- Attendre 5 minutes...
SET GLOBAL general_log = OFF;

-- Analyser les connexions suspectes
grep "Connect.*failed" /var/log/mysql/general.log
```

### 2. Debug d'une requête problématique

```sql
-- Forcer le logging d'une requête spécifique
SELECT SQL_NO_CACHE /* Force slow log */ * FROM huge_table;
```

### 3. Point-in-Time Recovery (PITR)

```bash
# 1. Restaurer un backup complet (ex: 2025-12-13 00:00)
mariabackup --copy-back --target-dir=/backup/full

# 2. Appliquer les binary logs jusqu'au point désiré
mysqlbinlog --start-datetime='2025-12-13 00:00:00' \
            --stop-datetime='2025-12-13 11:55:00' \
            /var/log/mysql/mysql-bin.* | mariadb
```

---

## Checklist de configuration logs

### Au démarrage d'un nouveau serveur

- [ ] Error log configuré et accessible
- [ ] Slow query log activé avec seuil adapté (2-5s)
- [ ] Binary log activé si réplication ou PITR nécessaire
- [ ] Permissions correctes sur /var/log/mysql (750, mysql:mysql)
- [ ] Logrotate configuré pour rotation quotidienne
- [ ] expire_logs_days configuré (7-14 jours)
- [ ] Surveillance espace disque active
- [ ] General log DÉSACTIVÉ
- [ ] Documentation de la stratégie de rétention

### Audit mensuel

- [ ] Vérifier la taille des logs (disk usage)
- [ ] Analyser le slow query log (pt-query-digest)
- [ ] Vérifier l'error log pour warnings récurrents
- [ ] Purger manuellement si expire_logs_days insuffisant
- [ ] Tester la rotation (logrotate -f)
- [ ] Vérifier les sauvegardes des binary logs

---

## ✅ Points clés à retenir

- **4 types de logs** : Error (essentiel), Slow Query (optimisation), General (debug uniquement), Binary (réplication/PITR)
- **Production** : Error + Slow Query + Binary (si HA/PITR), General OFF
- **Impact performance** : General log = -60%, Binary log = -15%, Slow query = -2%
- **Rotation** : Utiliser logrotate pour gérer la taille des fichiers
- **Binary logs** : JAMAIS supprimer avec `rm`, utiliser `PURGE BINARY LOGS`
- **Sécurité** : Permissions 640, chiffrement archives, rétention RGPD
- **Outils** : pt-query-digest (analyse slow log), mysqlbinlog (binary logs)
- **Monitoring** : Surveiller l'espace disque et analyser régulièrement
- **PITR** : Binary logs + backup complet = restauration précise
- **General log** : Réservé au debug ponctuel, impact massif
- **Destinations** : Préférer FILE vs TABLE pour la performance
- **Anonymisation** : Protéger les données sensibles dans les logs

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Server Logs](https://mariadb.com/kb/en/server-logs/)
- [📖 Documentation officielle - Error Log](https://mariadb.com/kb/en/error-log/)
- [📖 Documentation officielle - Slow Query Log](https://mariadb.com/kb/en/slow-query-log/)
- [📖 Documentation officielle - Binary Log](https://mariadb.com/kb/en/binary-log/)
- [🔧 Percona Toolkit - pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)
- [📖 logrotate Manual](https://linux.die.net/man/8/logrotate)
- [📖 mysqlbinlog Documentation](https://mariadb.com/kb/en/mysqlbinlog/)

---

## ➡️ Sections suivantes

- **11.4.1 Error log** : Configuration détaillée, niveaux de verbosité, analyse des erreurs
- **11.4.2 Slow query log** : Optimisation requêtes, pt-query-digest, stratégies d'indexation
- **11.4.3 General log** : Usage debug, sécurité, alternatives
- **11.4.4 Binary logs** : Formats, réplication, PITR, purge, mysqlbinlog

---

**💡 Conseil final** : Les logs sont comme les ceintures de sécurité : on ne réalise leur importance qu'au moment de l'accident. Configurez-les correctement **avant** d'en avoir besoin ! 🚨📊

⏭️ [Error log](/11-administration-configuration/04.1-error-log.md)

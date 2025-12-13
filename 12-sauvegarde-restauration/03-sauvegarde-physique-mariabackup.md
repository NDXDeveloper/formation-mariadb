🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3 Sauvegarde physique (Mariabackup)

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Section 12.1 (Stratégies), 12.2 (Sauvegardes logiques), Administration MariaDB, InnoDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes et mécanismes des sauvegardes physiques
- **Maîtriser** Mariabackup pour les backups hot (à chaud) sans interruption
- **Exploiter** 🆕 le support BACKUP STAGE (MariaDB 11.8) pour améliorer les performances
- **Distinguer** full backup, incremental backup et differential backup physiques
- **Concevoir** des stratégies de backup physique pour environnements critiques
- **Résoudre** les problématiques de backup sur bases volumineuses (multi-To)
- **Optimiser** les performances de backup et restauration en production

---

## Introduction

Les sauvegardes physiques constituent la **méthode privilégiée en production** pour les bases de données volumineuses nécessitant des objectifs RPO/RTO agressifs. Contrairement aux sauvegardes logiques qui exportent les données en SQL, les sauvegardes physiques **copient directement les fichiers de données** du système de fichiers.

### Qu'est-ce qu'une sauvegarde physique ?

Une sauvegarde physique copie les fichiers bruts du moteur de stockage :

```
Structure du répertoire MariaDB :
/var/lib/mysql/
├── ibdata1                    # InnoDB system tablespace
├── ib_logfile0                # InnoDB redo log
├── ib_logfile1                # InnoDB redo log
├── myapp/
│   ├── users.ibd              # Table InnoDB (file-per-table)
│   ├── orders.ibd
│   ├── products.ibd
│   └── db.opt
├── mysql/                     # Tables système
└── performance_schema/

Backup physique → Copie de TOUS ces fichiers
```

Cette approche **bas niveau** offre des performances maximales mais nécessite une gestion plus technique que les backups logiques.

### Pourquoi Mariabackup ?

**Mariabackup** est l'outil officiel de MariaDB pour les sauvegardes physiques. C'est un fork de Percona XtraBackup optimisé pour MariaDB, maintenu par MariaDB Corporation.

```
Évolution historique :
MySQL              → InnoDB Hot Backup (commercial)
                   ↓
Percona            → XtraBackup (open source)
                   ↓
MariaDB 10.1+      → Mariabackup (fork optimisé MariaDB)
                   ↓
MariaDB 11.8 🆕    → Support BACKUP STAGE
```

**Caractéristiques clés** :
- ✅ **Hot backup** : Copie à chaud sans arrêt du serveur
- ✅ **Non-bloquant** : Pas de verrouillage des tables (lecture/écriture continuent)
- ✅ **Incrémental natif** : Support full, incremental, differential
- ✅ **Compression** : Réduction de l'espace de stockage
- ✅ **Streaming** : Envoi direct vers destination distante
- ✅ **Tous moteurs** : InnoDB, Aria, MyISAM, RocksDB
- 🆕 **BACKUP STAGE** : Coordination améliorée avec le serveur (11.8)

---

## Principe du backup à chaud (Hot Backup)

### Comment copier une base en cours d'utilisation ?

Le défi fondamental : copier des fichiers qui changent constamment sans corrompre les données.

**Problème sans mécanisme spécial** :
```
T0: Copie page 1 du fichier users.ibd  → Valeur: Alice
T1: Application modifie page 1          → Valeur: Bob
T2: Application modifie page 2          → Valeur: Charlie
T3: Copie page 2 du fichier users.ibd  → Valeur: Charlie

Résultat : Incohérence ! (page 1 avant modif, page 2 après modif)
```

**Solution Mariabackup : Copie + Redo logs**

```
┌─────────────────────────────────────────────────────┐
│         Mécanisme Hot Backup de Mariabackup         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Phase 1 : Copie des fichiers de données            │
│  ┌─────────────────────────────────────┐            │
│  │ Copie users.ibd, orders.ibd, ...    │            │
│  │ (Les modifications continuent !)    │            │
│  └─────────────────────────────────────┘            │
│             │                                       │
│             │ En parallèle :                        │
│             ▼                                       │
│  ┌─────────────────────────────────────┐            │
│  │ Capture des redo logs (LSN)         │            │
│  │ ib_logfile0, ib_logfile1            │            │
│  │ (Toutes les modifs pendant copie)   │            │
│  └─────────────────────────────────────┘            │
│             │                                       │
│             ▼                                       │
│  Phase 2 : FLUSH TABLES (court instant)             │
│  ┌─────────────────────────────────────┐            │
│  │ Copie tables non-InnoDB (MyISAM)    │            │
│  │ Snapshot cohérent                   │            │
│  └─────────────────────────────────────┘            │
│             │                                       │
│             ▼                                       │
│  Résultat : Backup cohérent malgré activité         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### LSN (Log Sequence Number)

Concept central pour comprendre Mariabackup :

```sql
-- LSN = Compteur global des modifications InnoDB

État au démarrage du backup :
LSN_START = 12345678

Modifications pendant le backup :
  INSERT INTO users ... → LSN 12345679
  UPDATE orders ...     → LSN 12345680
  DELETE FROM logs ...  → LSN 12345681

État à la fin du backup :
LSN_END = 12345681

Mariabackup capture tout entre LSN_START et LSN_END
```

**Lors de la restauration** :
1. Copie des fichiers de données (état LSN_START)
2. Application des redo logs (LSN_START → LSN_END)
3. Base restaurée à l'état LSN_END ✅

---

## 🆕 Nouveauté MariaDB 11.8 : Support BACKUP STAGE

### Qu'est-ce que BACKUP STAGE ?

**BACKUP STAGE** est une nouvelle commande SQL introduite dans MariaDB 11.8 qui améliore la coordination entre Mariabackup et le serveur MariaDB pendant les backups.

```sql
-- Commandes BACKUP STAGE (MariaDB 11.8+)

BACKUP STAGE START;           -- Phase 1 : Début du backup
-- ... Mariabackup copie les données InnoDB ...

BACKUP STAGE BLOCK_COMMIT;    -- Phase 2 : Bloque nouveaux commits
-- ... Finalisation cohérente ...

BACKUP STAGE END;             -- Phase 3 : Libération
```

### Problème résolu

**Avant MariaDB 11.8** :
```
Mariabackup utilisait FLUSH TABLES WITH READ LOCK :
├─ Bloque TOUTES les écritures (global lock)
├─ Impact : Latence élevée pendant quelques secondes
├─ Problème : Peut bloquer des transactions importantes
└─ Sous-optimal pour bases volumineuses
```

**Avec MariaDB 11.8 et BACKUP STAGE** :
```
BACKUP STAGE approche plus granulaire :
├─ Verrouillage progressif et ciblé
├─ Impact minimisé (locks plus courts et localisés)
├─ Coordination explicite backup ↔ serveur
├─ Performance améliorée sur grosses bases
└─ Meilleure prévisibilité des temps de backup
```

### Architecture BACKUP STAGE

```
┌──────────────────────────────────────────────────────────┐
│     Workflow Mariabackup avec BACKUP STAGE (11.8)        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  T0 : Mariabackup démarre                                │
│  │                                                       │
│  ├─► BACKUP STAGE START                                  │
│  │    └─ Serveur prépare l'état pour backup              │
│  │                                                       │
│  ├─► Copie fichiers InnoDB (users.ibd, orders.ibd, ...)  │
│  │    └─ Les écritures continuent normalement            │
│  │    └─ Redo logs capturés en parallèle                 │
│  │                                                       │
│  ├─► BACKUP STAGE BLOCK_COMMIT                           │
│  │    └─ Nouveaux commits bloqués (court instant)        │
│  │    └─ Commits en cours peuvent terminer               │
│  │                                                       │
│  ├─► Copie tables non-InnoDB (MyISAM, Aria)              │
│  │    └─ Snapshot cohérent de toutes les tables          │
│  │                                                       │
│  └─► BACKUP STAGE END                                    │
│       └─ Libération de tous les verrous                  │
│       └─ Activité normale reprend                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Avantages mesurables

**Benchmark (base 500 Go, MariaDB 11.8)** :

| Métrique | Avant (FTWRL) | Avec BACKUP STAGE | Amélioration |
|----------|---------------|-------------------|--------------|
| Durée lock global | 8-12 sec | 2-3 sec | **-70%** |
| Transactions bloquées | 100-200 | 20-30 | **-80%** |
| Impact p95 latency | +500ms | +80ms | **-84%** |
| Prévisibilité | Variable | Stable | ✅ |

💡 **Impact production** : Sur une base à forte concurrence (5000 TPS), BACKUP STAGE réduit drastiquement le risque de timeout applicatif pendant le backup.

### Compatibilité

```
┌────────────────────────────────────────────────────┐
│  Support BACKUP STAGE selon versions               │
├────────────────────────────────────────────────────┤
│                                                    │
│  MariaDB < 11.8  : ❌ Non supporté                 │
│                    └─ Utilise FLUSH TABLES (legacy)│
│                                                    │
│  MariaDB 11.8+   : ✅ Support complet              │
│                    └─ Mariabackup l'utilise auto   │
│                                                    │
│  MySQL 8.x       : ❌ Non compatible               │
│                    └─ Syntaxe différente           │
│                                                    │
└────────────────────────────────────────────────────┘
```

⚠️ **Note** : Mariabackup 11.8 détecte automatiquement la version du serveur et utilise BACKUP STAGE si disponible. Aucune configuration manuelle nécessaire.

---

## Mariabackup vs Alternatives

### Comparaison avec Percona XtraBackup

| Critère | Mariabackup | XtraBackup |
|---------|-------------|------------|
| **Compatibilité MariaDB** | ✅ Natif | ⚠️ Limitée |
| **Support BACKUP STAGE** | ✅ 11.8+ | ❌ Non |
| **Encryption natives** | ✅ Oui | ✅ Oui |
| **Compression** | ✅ Multiple formats | ✅ Multiple formats |
| **Support Galera** | ✅ Optimal | ⚠️ Partiel |
| **Maintenu par** | MariaDB Corp | Percona |
| **Recommandation** | ✅ Pour MariaDB | ✅ Pour MySQL |

💡 **Conseil** : Sur MariaDB, toujours privilégier Mariabackup. Sur MySQL, utiliser XtraBackup.

### Comparaison avec mysqldump (logique)

| Aspect | Mariabackup (Physique) | mysqldump (Logique) |
|--------|------------------------|---------------------|
| **Vitesse backup** | ⭐⭐⭐ Très rapide | ⭐ Lent |
| **Vitesse restauration** | ⭐⭐⭐ Très rapide | ⭐ Très lent |
| **Taille base max** | ✅ Multi-To | ⚠️ < 100 Go |
| **Impact production** | ⭐⭐⭐ Minimal | ⭐⭐ Moyen |
| **Portabilité** | ⭐ Version identique | ⭐⭐⭐ Totale |
| **Incrémental** | ✅ Natif | ❌ Non |
| **Sélectivité** | ⚠️ Base entière | ✅ Tables/DB |
| **Complexité** | ⭐⭐ Modérée | ⭐⭐⭐ Simple |

**Règle de décision** :
```
Taille base > 100 Go         → Mariabackup
RPO/RTO stricts              → Mariabackup
Besoin incrémental           → Mariabackup
Migration cross-version      → mysqldump
Export sélectif (quelques tables) → mysqldump
```

---

## Architecture et types de backup

Mariabackup supporte trois types de backups, chacun avec ses caractéristiques :

### 1. Full Backup (Sauvegarde complète)

Copie l'intégralité de la base de données.

```
┌────────────────────────────────────────┐
│         Full Backup                    │
├────────────────────────────────────────┤
│  Copie complète :                      │
│  ├─ ibdata1 (100%)                     │
│  ├─ ib_logfile* (100%)                 │
│  ├─ Toutes tables .ibd (100%)          │
│  ├─ Tables système (100%)              │
│  └─ Métadonnées (100%)                 │
│                                        │
│  Résultat : Backup autonome            │
│  Restauration : 1 seul fichier requis  │
└────────────────────────────────────────┘
```

**Commande** :
```bash
mariabackup --backup \
  --target-dir=/backups/full/2025-12-13 \
  --user=backup_user \
  --password=secret
```

**Cas d'usage** :
- Base de référence hebdomadaire ou mensuelle
- Premier backup avant mise en place d'incrémentaux
- Restauration complète rapide

### 2. Incremental Backup (Sauvegarde incrémentale)

Copie uniquement les pages modifiées depuis le dernier backup (full ou incremental).

```
Timeline incrémentale :
────────────────────────────────────────

Dimanche     Lundi      Mardi      Mercredi
┌──────┐    ┌────┐     ┌────┐     ┌─────┐
│ FULL │───►│Inc1│────►│Inc2│────►│ Inc3│
│200 GB│    │8 GB│     │9 GB│     │ 7GB │
└──────┘    └────┘     └────┘     └─────┘
 LSN=100    LSN=105    LSN=110    LSN=115
            (base=100) (base=105) (base=110)

Restauration mercredi :
FULL → Inc1 → Inc2 → Inc3 (chaîne complète requise)
```

**Commande** :
```bash
mariabackup --backup \
  --target-dir=/backups/inc/2025-12-14 \
  --incremental-basedir=/backups/full/2025-12-13 \
  --user=backup_user \
  --password=secret
```

**Mécanisme** : Mariabackup compare les LSN (Log Sequence Numbers) des pages pour identifier celles modifiées.

**Cas d'usage** :
- Backup quotidien ou horaire avec RPO serré
- Économie d'espace sur bases volumineuses
- Stratégie full hebdo + inc quotidien

### 3. Differential Backup (Non natif, via scripting)

⚠️ **Note** : Mariabackup n'a pas de mode "differential" natif, mais on peut le simuler en gardant toujours le même `--incremental-basedir` (le full).

```bash
# Similaire à incremental mais toujours basé sur le FULL
mariabackup --backup \
  --target-dir=/backups/diff/2025-12-15 \
  --incremental-basedir=/backups/full/2025-12-13 \  # Toujours le même
  --user=backup_user \
  --password=secret
```

💡 **Astuce** : En pratique, on utilise surtout full + incremental en production. La différentielle est plus courante en logique.

---

## Phases d'un backup Mariabackup

### Phase 1 : Backup

```bash
mariabackup --backup --target-dir=/backups/full
```

**Actions** :
1. Connexion au serveur MariaDB
2. 🆕 Envoi `BACKUP STAGE START` (11.8+) ou `FLUSH TABLES WITH READ LOCK` (< 11.8)
3. Copie des fichiers InnoDB (.ibd, ibdata1)
4. Capture continue des redo logs
5. 🆕 `BACKUP STAGE BLOCK_COMMIT` (11.8+)
6. Copie des tables non-InnoDB (MyISAM, Aria)
7. Copie des métadonnées (xtrabackup_checkpoints, xtrabackup_info)
8. 🆕 `BACKUP STAGE END` (11.8+) ou `UNLOCK TABLES`

**Résultat** : Répertoire contenant copie brute des fichiers + redo logs capturés.

⚠️ **Important** : Le backup n'est PAS directement restaurable. Nécessite la phase "prepare".

### Phase 2 : Prepare

```bash
mariabackup --prepare --target-dir=/backups/full
```

**Actions** :
1. Application des redo logs capturés
2. Rollback des transactions non-committées
3. Mise en cohérence des fichiers de données
4. Génération d'un état "crash-safe"

**Résultat** : Backup prêt à être restauré (équivalent à un shutdown propre du serveur).

### Phase 3 : Restore

```bash
# Arrêt MariaDB
systemctl stop mariadb

# Nettoyage datadir
rm -rf /var/lib/mysql/*

# Copie du backup
mariabackup --copy-back --target-dir=/backups/full

# Permissions
chown -R mysql:mysql /var/lib/mysql

# Redémarrage
systemctl start mariadb
```

**Résultat** : Base de données restaurée et opérationnelle.

---

## Avantages des sauvegardes physiques

### 1. Performance maximale

**Vitesse de backup** :
```
Base 1 To :
├─ Mariabackup : 1h30 - 2h
├─ mysqldump : 8h - 12h
└─ Gain : 4-6x plus rapide
```

**Vitesse de restauration** :
```
Base 1 To :
├─ Mariabackup : 2h - 2h30 (copie + prepare)
├─ mysqldump : 10h - 14h (parse SQL + index rebuild)
└─ Gain : 4-6x plus rapide
```

### 2. Hot backup sans interruption

```
Application continue pendant le backup :
├─ Lectures : ✅ Normales
├─ Écritures : ✅ Normales (sauf ~3 sec BACKUP STAGE 🆕)
├─ Impact : < 5% charge supplémentaire (I/O)
└─ Downtime : 0 seconde
```

### 3. Support incrémental natif

```
Économie d'espace sur 1 mois (base 2 To, changement 2%/jour) :

Full quotidien :
└─ 30 × 800 Go = 24 To

Full hebdo + Inc quotidien :
├─ 4 Full : 4 × 800 Go = 3.2 To
├─ 26 Inc : 26 × 16 Go = 416 Go
└─ Total : 3.6 To (économie 85%)
```

### 4. Cohérence garantie

Contrairement aux copies de fichiers naïves (`cp -r /var/lib/mysql`), Mariabackup garantit la cohérence via :
- Capture des redo logs
- Application des transactions committées
- Rollback des transactions en cours

### 5. Support de tous les moteurs

```
InnoDB     : ✅ Hot backup (MVCC)
Aria       : ✅ Backup transactionnel
MyISAM     : ⚠️ Brève pause (FLUSH TABLES)
RocksDB    : ✅ Support natif
TokuDB     : ✅ Support natif (si compilé)
```

---

## Limitations et considérations

### 1. Dépendance version/configuration

⚠️ **Contrainte** : Le backup doit être restauré sur une version **compatible** :

```
Compatibilité :
├─ Même version majeure : ✅ OK (11.8 → 11.8)
├─ Version mineure proche : ✅ Souvent OK (11.4 → 11.8)
├─ Version très différente : ❌ Risqué (10.6 → 11.8)
└─ Cross-SGBD : ❌ Impossible (MariaDB → MySQL)
```

**Solution** : Pour migration majeure, utiliser mysqldump (logique).

### 2. Configuration innodb_page_size

```sql
-- Si backup créé avec innodb_page_size=16K
-- Restauration requiert innodb_page_size=16K identique

-- Vérification
SHOW VARIABLES LIKE 'innodb_page_size';
```

⚠️ **Important** : Le page size doit être identique entre source et destination.

### 3. Pas de sélectivité (tout ou rien)

Mariabackup copie toujours **toute la base** :

```
Impossible de :
├─ Backup d'une seule table
├─ Backup d'une seule base de données
├─ Filtrage WHERE sur les données
└─ Export structure sans données

Solution : Utiliser mysqldump pour ces cas
```

### 4. Gestion de l'espace pendant backup

**Espace requis** :
```
Backup non préparé (--backup) :
├─ Fichiers de données : 100%
├─ Redo logs capturés : +5-10%
└─ Total : ~110% de la taille de la base

Backup préparé (--prepare) :
└─ Taille finale : ~100% (redo logs appliqués et supprimés)
```

💡 **Recommandation** : Prévoir 1.5x la taille de la base pour les backups (marge de sécurité).

### 5. Impact I/O sur production

Bien que minimal, Mariabackup génère de l'I/O :

```
Impact mesuré (base 500 Go) :
├─ Reads : +100-150 MB/s (lecture fichiers)
├─ Writes : +50-80 MB/s (écriture backup)
├─ IOPS : +1000-2000 (dépend fragmentation)
└─ Recommandation : Planifier en heures creuses
```

**Optimisations** :
```bash
# Throttling I/O
mariabackup --backup --throttle=50  # Limite à 50 MB/s

# Utiliser un replica pour le backup (0 impact primary)
```

---

## Cas d'usage en production

### 1. Base de données critique (e-commerce)

**Contexte** :
- Base : 2 To
- Transactions : 5000 TPS
- RPO : 1 heure
- RTO : 2 heures

**Stratégie** :
```bash
# Full quotidien à 02:00
mariabackup --backup \
  --target-dir=/backups/full/$(date +%Y%m%d) \
  --compress \
  --compress-threads=4

# Incrémental toutes les heures
mariabackup --backup \
  --target-dir=/backups/inc/$(date +%Y%m%d_%H%M) \
  --incremental-basedir=/backups/full/$(date +%Y%m%d) \
  --compress
```

### 2. Data warehouse analytique

**Contexte** :
- Base : 10 To
- Charge : Batch nocturnes
- RPO : 24 heures
- RTO : 8 heures

**Stratégie** :
```bash
# Full hebdomadaire (dimanche)
mariabackup --backup \
  --target-dir=/backups/full/$(date +%Y%m%d) \
  --parallel=8 \
  --compress \
  --stream=xbstream | \
  aws s3 cp - s3://warehouse-backups/full_$(date +%Y%m%d).xbstream

# Incrémental quotidien
# (basé sur le full du dimanche précédent)
```

### 3. Environnement Kubernetes

**Contexte** :
- Déploiement : StatefulSet
- Stockage : PersistentVolume
- Besoin : Backup automatisé

**Stratégie** :
```yaml
# CronJob Kubernetes pour Mariabackup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mariadb-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mariabackup
            image: mariadb:11.8
            command:
            - /bin/bash
            - -c
            - |
              mariabackup --backup \
                --target-dir=/backup/full-$(date +%Y%m%d) \
                --host=mariadb-primary \
                --user=backup \
                --password=$BACKUP_PASSWORD
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

---

## Installation et prérequis

### Installation

**Debian/Ubuntu** :
```bash
# MariaDB 11.8 inclut Mariabackup
apt-get update
apt-get install mariadb-backup

# Vérification
mariabackup --version
# mariabackup based on MariaDB server 11.8.0 Linux (x86_64)
```

**RHEL/CentOS** :
```bash
yum install MariaDB-backup

mariabackup --version
```

**Depuis les sources** :
```bash
# Mariabackup fait partie du serveur MariaDB
cmake . -DWITH_EMBEDDED_SERVER=ON
make mariabackup
```

### Prérequis système

**Utilisateur backup** :
```sql
-- Créer un utilisateur dédié avec privilèges minimaux
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'secure_password';

GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT, PROCESS ON *.* 
  TO 'backup_user'@'localhost';

-- MariaDB 11.8 : Privilèges BACKUP STAGE 🆕
GRANT BACKUP_ADMIN ON *.* TO 'backup_user'@'localhost';

FLUSH PRIVILEGES;
```

**Espace disque** :
```bash
# Vérifier l'espace disponible
df -h /backups/

# Règle : Minimum 1.5x la taille de /var/lib/mysql
du -sh /var/lib/mysql
# 500G

# Espace requis : ~750G pour les backups
```

**Configuration MariaDB** :
```ini
# /etc/mysql/mariadb.conf.d/backup.cnf

[mariadb]
# Activer binary logs (optionnel mais recommandé pour PITR)
log_bin = /var/log/mysql/mariadb-bin
binlog_format = ROW

# Conserver assez de redo logs pour backup
innodb_log_file_size = 512M

# Buffer pool (aide performance backup)
innodb_buffer_pool_size = 16G
```

---

## Monitoring et métriques

### Informations pendant le backup

Mariabackup génère des logs verbeux :

```bash
mariabackup --backup --target-dir=/backups/test 2>&1 | tee backup.log
```

**Extrait de log** :
```
[00] 2025-12-13 14:30:15 Using server version 11.8.0-MariaDB
[00] 2025-12-13 14:30:15 Uses posix_fadvise().
[00] 2025-12-13 14:30:15 cd to /var/lib/mysql/
[00] 2025-12-13 14:30:15 open files limit = 65535
🆕 [00] 2025-12-13 14:30:15 mariabackup: using BACKUP STAGE
[01] 2025-12-13 14:30:16 Copying ibdata1 to /backups/test/ibdata1
[01] 2025-12-13 14:30:30 ...done
[01] 2025-12-13 14:30:30 Copying ./myapp/users.ibd to /backups/test/myapp/users.ibd
[01] 2025-12-13 14:31:05 ...done
```

### Fichiers de métadonnées

Chaque backup contient des fichiers d'information :

**xtrabackup_checkpoints** :
```
backup_type = full-backuped
from_lsn = 0
to_lsn = 123456789
last_lsn = 123456789
compact = 0
recover_binlog_info = 0
flushed_lsn = 123456789
```

**xtrabackup_info** :
```
uuid = 9a2c7e54-9a1b-11ef-b864-0242ac120002
name = 
tool_name = mariabackup
tool_command = --backup --target-dir=/backups/full
tool_version = 11.8.0
ibbackup_version = 11.8.0
server_version = 11.8.0-MariaDB
start_time = 2025-12-13 14:30:15
end_time = 2025-12-13 15:05:22
lock_time = 2
binlog_pos = filename 'mariadb-bin.000042', position '1234567'
innodb_from_lsn = 0
innodb_to_lsn = 123456789
```

💡 **Usage** : Ces fichiers sont critiques pour les backups incrémentaux et le troubleshooting.

---

## ✅ Points clés à retenir

- **Mariabackup** : Outil officiel MariaDB pour sauvegardes physiques, fork optimisé de XtraBackup
- **Hot backup** : Copie à chaud sans interruption via capture des redo logs et MVCC
- 🆕 **BACKUP STAGE** : Nouvelle commande MariaDB 11.8 réduisant l'impact des locks (-70% durée)
- **Performance** : 4-6x plus rapide que mysqldump en backup ET restauration
- **Types** : Full (complet), Incremental (delta), support natif des deux
- **Phases** : Backup → Prepare → Restore (3 étapes obligatoires)
- **LSN** : Log Sequence Number, mécanisme central pour cohérence et incrémentaux
- **Limitations** : Dépendance version, pas de sélectivité, même page_size requis
- **Production** : Stratégie full quotidien + inc horaire standard pour bases critiques
- **Prérequis** : Utilisateur avec privilège BACKUP_ADMIN (11.8+), espace 1.5x base

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Mariabackup Overview - MariaDB KB](https://mariadb.com/kb/en/mariabackup-overview/)
- [📖 Full Backup and Restore - MariaDB KB](https://mariadb.com/kb/en/full-backup-and-restore-with-mariabackup/)
- [📖 Incremental Backup - MariaDB KB](https://mariadb.com/kb/en/incremental-backup-and-restore-with-mariabackup/)
- [📖 BACKUP STAGE - MariaDB KB](https://mariadb.com/kb/en/backup-stage/) 🆕

### Articles techniques

- [Mariabackup Best Practices - MariaDB Corporation](https://mariadb.com/resources/blog/mariabackup-best-practices/)
- [Physical Backups with Mariabackup - Percona Blog](https://www.percona.com/blog/physical-backups-mariabackup/)

### Comparaisons

- [Mariabackup vs XtraBackup - MariaDB vs MySQL](https://mariadb.com/kb/en/mariabackup-vs-percona-xtrabackup/)

---

## ➡️ Sections suivantes

Les sous-sections suivantes détailleront chaque type de backup :

**[12.3.1 - Full backup](./03.1-full-backup.md)** : Procédure complète de backup full, options avancées, optimisations, scripting automatisé.

**[12.3.2 - Incremental backup](./03.2-incremental-backup.md)** : Stratégie incrémentale, chaînes de backups, restauration séquentielle, gestion des LSN.

**[12.3.3 - Support BACKUP STAGE](./03.3-backup-stage.md)** 🆕 : Deep dive sur BACKUP STAGE (MariaDB 11.8), configuration, monitoring, troubleshooting.

---


⏭️ [Full backup](/12-sauvegarde-restauration/03.1-full-backup.md)

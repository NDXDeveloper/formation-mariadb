🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1 Fichiers de configuration

> **Niveau** : Avancé  
> **Durée estimée** : 1-2 heures  
> **Prérequis** :
> - Connaissances système Linux/Unix et Windows
> - Bases de l'administration MariaDB
> - Compréhension des variables système

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Localiser** les fichiers de configuration MariaDB sur différents systèmes
- **Comprendre** l'ordre de lecture et la priorité des fichiers de configuration
- **Organiser** une configuration modulaire et maintenable
- **Appliquer** les bonnes pratiques de gestion de configuration
- **Sécuriser** les fichiers sensibles contenant des credentials
- **Diagnostiquer** les problèmes de configuration au démarrage

---

## Introduction

Les fichiers de configuration constituent le **point d'entrée principal** pour administrer et personnaliser le comportement de MariaDB. Une configuration bien structurée est essentielle pour :

- **Performance** : Ajuster les paramètres selon votre charge de travail
- **Sécurité** : Définir les règles de connexion et d'authentification
- **Maintenabilité** : Faciliter les modifications et le versionning
- **Reproductibilité** : Déployer des configurations identiques sur plusieurs serveurs

### Philosophie de configuration MariaDB

MariaDB adopte une approche **hiérarchique et modulaire** :

```
Configuration par défaut (compilée)
    ↓
Fichiers système (/etc/my.cnf)
    ↓
Fichiers utilisateur (~/.my.cnf)
    ↓
Variables de ligne de commande (--option)
    ↓
Variables dynamiques (SET GLOBAL)
```

💡 **Principe clé** : Chaque niveau peut **surcharger** les valeurs des niveaux précédents, offrant une grande flexibilité.

---

## Localisation des fichiers de configuration

MariaDB recherche les fichiers de configuration dans un **ordre précis** qui varie selon le système d'exploitation.

### Sur Linux/Unix

MariaDB lit les fichiers dans cet ordre (du plus général au plus spécifique) :

```bash
1. /etc/my.cnf                          # Configuration système globale
2. /etc/mysql/my.cnf                    # Alternative Debian/Ubuntu
3. /usr/local/mysql/etc/my.cnf         # Installation personnalisée
4. ~/.my.cnf                            # Configuration utilisateur
5. ~/.mylogin.cnf                       # Credentials (chiffré)
```

#### Distribution Debian/Ubuntu

```bash
/etc/mysql/
├── my.cnf                    # Fichier principal (souvent un wrapper)
├── mariadb.cnf              # Spécifique MariaDB
├── conf.d/                   # Configurations supplémentaires
│   └── mysql.cnf
└── mariadb.conf.d/          # Configurations spécifiques MariaDB
    ├── 50-server.cnf        # Configuration serveur
    ├── 50-client.cnf        # Configuration client
    └── 50-mysqld_safe.cnf   # Configuration mysqld_safe
```

**Exemple de `/etc/mysql/my.cnf` sur Ubuntu** :

```ini
# The MariaDB configuration file
#
# One can use all long options that the program supports.
# Run program with --help to get a list of available options

# This group is read both both by the client and the server
# use it for options that affect everything
[client-server]

# Import all .cnf files from configuration directory
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/
```

#### Distribution RedHat/CentOS/Rocky

```bash
/etc/
├── my.cnf                    # Configuration principale
└── my.cnf.d/                 # Configurations modulaires
    ├── client.cnf
    ├── server.cnf
    └── mariadb-server.cnf
```

### Sur Windows

MariaDB recherche les fichiers dans cet ordre :

```
1. C:\Windows\my.ini
2. C:\Windows\my.cnf
3. C:\my.ini
4. C:\my.cnf
5. INSTALLDIR\my.ini          # Répertoire d'installation MariaDB
6. INSTALLDIR\my.cnf
7. INSTALLDIR\data\my.ini     # Répertoire des données
8. INSTALLDIR\data\my.cnf
9. %MYSQL_HOME%\my.ini
10. %MYSQL_HOME%\my.cnf
```

💡 **Conseil Windows** : Privilégiez `my.ini` dans le répertoire d'installation MariaDB pour éviter les conflits avec d'autres logiciels.

### Vérifier les fichiers utilisés

Pour déterminer quels fichiers sont réellement lus par MariaDB :

```bash
# Afficher les fichiers de configuration consultés
mariadb --help --verbose | grep -A 1 "Default options"

# Ou avec mysqld
mysqld --verbose --help | grep -A 1 "Default options"
```

**Sortie exemple** :

```
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```

---

## Ordre de priorité et surcharge

### Principe de surcharge

Lorsque la même variable est définie dans plusieurs fichiers, **la dernière valeur lue l'emporte**.

**Exemple concret** :

```ini
# Dans /etc/my.cnf
[mysqld]
max_connections = 100

# Dans /etc/mysql/mariadb.conf.d/50-server.cnf
[mysqld]
max_connections = 500

# Dans ~/.my.cnf
[mysqld]
max_connections = 200
```

**Résultat** : `max_connections = 200` (le fichier utilisateur a priorité)

### Ligne de commande : priorité absolue

Les options passées en ligne de commande **surchargent tout** :

```bash
mysqld --max-connections=1000
```

Cette valeur remplacera toute configuration dans les fichiers, y compris `~/.my.cnf`.

### Vérification de la valeur effective

```sql
-- Connexion au serveur
mariadb -u root -p

-- Vérifier la valeur en cours
SHOW VARIABLES LIKE 'max_connections';

-- Vérifier toutes les variables
SHOW VARIABLES;
```

---

## Organisation modulaire avec `!includedir`

### Directive `!includedir`

La directive `!includedir` permet d'inclure **tous les fichiers `.cnf`** d'un répertoire.

```ini
# Fichier principal /etc/my.cnf
[client-server]

# Inclure tous les fichiers de configuration
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/
```

⚠️ **Important** :
- Les fichiers sont lus par **ordre alphabétique**
- Seuls les fichiers avec extension `.cnf` sont inclus
- Les fichiers de sauvegarde (`.bak`, `.old`, `~`) sont **ignorés**

### Structure recommandée pour la production

```bash
/etc/mysql/
├── my.cnf                           # Point d'entrée (minimal)
└── mariadb.conf.d/
    ├── 10-base.cnf                  # Configuration de base
    ├── 20-innodb.cnf                # Configuration InnoDB
    ├── 30-replication.cnf           # Réplication (si applicable)
    ├── 40-performance.cnf           # Optimisations performance
    ├── 50-security.cnf              # Paramètres de sécurité
    ├── 60-logging.cnf               # Configuration des logs
    └── 99-custom.cnf                # Personnalisations spécifiques
```

💡 **Avantage** : Cette organisation facilite :
- La **maintenance** (un aspect par fichier)
- Le **versionning** (Git, Ansible, etc.)
- Le **troubleshooting** (activer/désactiver un aspect)
- La **documentation** (commentaires dans chaque fichier)

### Exemple : Configuration modulaire complète

#### `/etc/mysql/my.cnf` (point d'entrée)

```ini
# MariaDB Configuration File
# /etc/mysql/my.cnf

[client-server]
# Default socket
socket = /var/run/mysqld/mysqld.sock

# Import all configuration files
!includedir /etc/mysql/mariadb.conf.d/
```

#### `/etc/mysql/mariadb.conf.d/10-base.cnf`

```ini
# Configuration de base MariaDB 11.8
# /etc/mysql/mariadb.conf.d/10-base.cnf

[mysqld]
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
port = 3306
datadir = /var/lib/mysql

# Nouveauté 11.8 : utf8mb4 par défaut
character-set-server = utf8mb4
collation-server = utf8mb4_uca1400_ai_ci

# Timezone
default_time_zone = '+00:00'
```

#### `/etc/mysql/mariadb.conf.d/20-innodb.cnf`

```ini
# Configuration InnoDB
# /etc/mysql/mariadb.conf.d/20-innodb.cnf

[mysqld]
# Buffer Pool (75% de la RAM pour serveur dédié)
innodb_buffer_pool_size = 24G
innodb_buffer_pool_instances = 8

# Log files
innodb_log_file_size = 2G
innodb_log_buffer_size = 16M

# I/O
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_flush_method = O_DIRECT
innodb_flush_log_at_trx_commit = 1

# Nouveauté 11.8 : Construction index efficace
innodb_alter_copy_bulk = ON
```

#### `/etc/mysql/mariadb.conf.d/60-logging.cnf`

```ini
# Configuration des logs
# /etc/mysql/mariadb.conf.d/60-logging.cnf

[mysqld]
# Error log
log_error = /var/log/mysql/error.log

# Slow Query Log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2
log_slow_verbosity = query_plan,explain

# Binary logs (réplication)
log_bin = /var/log/mysql/mysql-bin
binlog_format = MIXED
expire_logs_days = 7
max_binlog_size = 1G

# General log (DÉSACTIVÉ en production)
# general_log = 0
# general_log_file = /var/log/mysql/general.log
```

---

## Directive `!include` pour fichiers individuels

La directive `!include` permet d'inclure un **fichier spécifique** :

```ini
# Inclure un fichier de configuration spécifique
!include /etc/mysql/custom/replication.cnf

# Différence avec !includedir :
# - !include  = UN fichier spécifique
# - !includedir = TOUS les .cnf du répertoire
```

---

## Fichier de configuration utilisateur : `~/.my.cnf`

Le fichier `~/.my.cnf` permet de **personnaliser les paramètres client** pour un utilisateur Unix spécifique.

### Cas d'usage principal : Credentials

```ini
# ~/.my.cnf pour l'utilisateur 'admin'
[client]
user = admin
password = "SecurePassword123!"
socket = /var/run/mysqld/mysqld.sock

[mysql]
database = production
```

Avec cette configuration, l'utilisateur peut se connecter sans saisir les credentials :

```bash
# Connexion automatique
mariadb

# Équivalent à :
mariadb -u admin -p"SecurePassword123!" production
```

⚠️ **SÉCURITÉ CRITIQUE** : Ce fichier contient des **mots de passe en clair** !

### Sécurisation de `~/.my.cnf`

```bash
# Permissions strictes (lecture uniquement par le propriétaire)
chmod 600 ~/.my.cnf

# Vérifier les permissions
ls -la ~/.my.cnf
# -rw------- 1 admin admin 156 Dec 13 10:30 /home/admin/.my.cnf
```

💡 **Bonne pratique** : Pour les scripts automatisés, préférez `~/.mylogin.cnf` (chiffré) ou les variables d'environnement.

---

## Fichier `.mylogin.cnf` (credentials chiffrés)

### Avantage : Stockage sécurisé

`.mylogin.cnf` stocke les credentials de manière **chiffrée** (AES).

### Création avec `mysql_config_editor`

```bash
# Créer un profil de connexion nommé "production"
mysql_config_editor set \
    --login-path=production \
    --host=db.example.com \
    --user=admin \
    --password

# Saisir le mot de passe (prompt sécurisé)
Enter password: ******

# Lister les profils
mysql_config_editor print --all
```

### Utilisation

```bash
# Connexion avec le profil "production"
mariadb --login-path=production

# Dans un script
mysqldump --login-path=production ma_base > backup.sql
```

### Limitation

⚠️ Le chiffrement de `.mylogin.cnf` n'est **pas robuste** contre un attaquant ayant accès à la machine. C'est une **obfuscation**, pas une vraie sécurité cryptographique.

Pour une sécurité renforcée, utilisez :
- **HashiCorp Vault**
- **AWS Secrets Manager**
- **Azure Key Vault**
- **Variables d'environnement** avec gestion des secrets (Kubernetes Secrets, etc.)

---

## Variables système vs options de configuration

### Différence fondamentale

| Concept | Définition | Exemple |
|---------|------------|---------|
| **Option de configuration** | Paramètre dans `my.cnf` | `max_connections` |
| **Variable système** | Valeur en mémoire du serveur | `@@global.max_connections` |

```ini
# Dans my.cnf (option de configuration)
[mysqld]
max_connections = 500
```

```sql
-- En SQL (variable système)
SHOW VARIABLES LIKE 'max_connections';
-- Résultat: max_connections = 500

SET GLOBAL max_connections = 1000;
-- Résultat: max_connections = 1000 (mais seulement jusqu'au redémarrage)
```

### Persistance

- **my.cnf** : Persistant après redémarrage ✅
- **SET GLOBAL** : Temporaire (perdu au redémarrage) ❌

💡 **Workflow recommandé** :
1. Tester avec `SET GLOBAL` (effet immédiat)
2. Valider le comportement
3. Ajouter dans `my.cnf` (persistance)
4. Documenter le changement (commentaire dans my.cnf)

---

## Sections et groupes d'options

Les fichiers de configuration sont organisés en **sections** (ou groupes) identifiées par `[nom_section]`.

### Sections principales

| Section | Utilisée par | Usage |
|---------|--------------|-------|
| `[client]` | Tous les clients (mariadb, mysqldump, etc.) | Options client globales |
| `[mysql]` | Client `mariadb` uniquement | Options CLI interactif |
| `[mysqld]` | Serveur MariaDB | Configuration serveur |
| `[mysqld_safe]` | Script wrapper mysqld_safe | Options de démarrage |
| `[mariadb]` | Spécifique MariaDB | Options MariaDB (préféré) |
| `[mariadb-11.8]` | MariaDB 11.8 uniquement | Options version spécifique |

### Exemple complet

```ini
# Configuration multi-sections
# /etc/mysql/my.cnf

# Options pour TOUS les clients
[client]
port = 3306
socket = /var/run/mysqld/mysqld.sock
default-character-set = utf8mb4

# Options pour le client CLI uniquement
[mysql]
no-auto-rehash
show-warnings
prompt = "\\u@\\h [\\d]> "

# Options pour mysqldump
[mysqldump]
quick
quote-names
max_allowed_packet = 512M
single-transaction

# Options pour le SERVEUR
[mysqld]
user = mysql
datadir = /var/lib/mysql
port = 3306
bind-address = 0.0.0.0

# Mémoire
innodb_buffer_pool_size = 8G
max_connections = 500

# 🆕 Nouveautés MariaDB 11.8
character-set-server = utf8mb4
collation-server = utf8mb4_uca1400_ai_ci
max_tmp_space_usage = 10737418240  # 10 GB par session
max_total_tmp_space_usage = 107374182400  # 100 GB total

# Options spécifiques MariaDB (priorité sur [mysqld])
[mariadb]
# Thread Pool
thread_handling = pool-of-threads
thread_pool_size = 16

# Options spécifiques MariaDB 11.8
[mariadb-11.8]
# Extension TIMESTAMP 2106
# (activé automatiquement)
```

---

## Commentaires et documentation

### Syntaxe des commentaires

```ini
# Commentaire sur une ligne avec #

; Commentaire alternatif avec ;

[mysqld]
max_connections = 500  # Commentaire en fin de ligne
```

### Bonnes pratiques de documentation

```ini
# ============================================================
# CONFIGURATION INNODB - MariaDB 11.8 Production
# ============================================================
# Serveur: db-prod-01
# RAM: 32 GB
# Disque: SSD NVMe (7000 MB/s)
# Workload: OLTP haute concurrence
# Dernière modification: 2025-12-13 par admin@example.com
# ============================================================

[mysqld]

# Buffer Pool : 75% RAM (24 GB sur 32 GB)
# Justification: Serveur dédié MariaDB, workload OLTP
innodb_buffer_pool_size = 24G

# 8 instances pour réduire la contention
# Recommandation: 1 instance par 1-2 GB de buffer pool
innodb_buffer_pool_instances = 8

# Log files: 2 GB (permet ~1 heure de writes intensifs)
# ATTENTION: Modifier nécessite arrêt du serveur
innodb_log_file_size = 2G

# I/O Capacity: SSD NVMe moderne
# Benchmark: fio random write = 4000 IOPS
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
```

💡 **Conseil** : Documentez **pourquoi** vous avez choisi une valeur, pas seulement **quelle** valeur.

---

## Validation et test de configuration

### Vérifier la syntaxe avant démarrage

```bash
# Test de la configuration (sans démarrer le serveur)
mysqld --verbose --help > /dev/null

# Si la syntaxe est correcte, la commande ne retourne rien
# Si erreur, elle affiche le détail
```

### Démarrage en mode debug

```bash
# Démarrer en mode verbose pour diagnostiquer
mysqld --verbose --help 2>&1 | grep -A 5 "ERROR"

# Afficher toutes les options avec leurs valeurs
mysqld --verbose --help | less
```

### Sauvegarde avant modification

```bash
# TOUJOURS sauvegarder avant de modifier
cp /etc/mysql/my.cnf /etc/mysql/my.cnf.backup-$(date +%Y%m%d)

# Ou avec git (recommandé)
cd /etc/mysql
git add my.cnf mariadb.conf.d/
git commit -m "Backup avant modification max_connections"
```

---

## Gestion de configuration avec versionning

### Git pour la configuration

```bash
# Initialiser le dépôt
cd /etc/mysql
git init
git add my.cnf mariadb.conf.d/
git commit -m "Configuration initiale MariaDB 11.8"

# Modifier la configuration
vim mariadb.conf.d/20-innodb.cnf

# Commiter les changements
git add mariadb.conf.d/20-innodb.cnf
git commit -m "Augmentation innodb_buffer_pool_size de 16G à 24G"

# Voir l'historique
git log --oneline

# Rollback si nécessaire
git checkout HEAD~1 -- mariadb.conf.d/20-innodb.cnf
```

### Ansible pour le déploiement

```yaml
# playbook.yml
---
- name: Déployer configuration MariaDB
  hosts: db_servers
  become: yes
  tasks:
    - name: Copier la configuration InnoDB
      copy:
        src: files/20-innodb.cnf
        dest: /etc/mysql/mariadb.conf.d/20-innodb.cnf
        owner: root
        group: root
        mode: '0644'
      notify: Restart MariaDB

  handlers:
    - name: Restart MariaDB
      service:
        name: mariadb
        state: restarted
```

---

## Détection et résolution de problèmes

### Fichier de configuration ignoré

**Symptôme** : Modifications dans `my.cnf` n'ont aucun effet.

**Diagnostic** :

```bash
# Vérifier quels fichiers sont lus
mariadb --help --verbose | grep -A 1 "Default options"

# Vérifier la syntaxe
mysqld --verbose --help > /dev/null 2>&1
echo $?  # 0 = succès, >0 = erreur
```

**Solution** : Vérifier l'ordre de lecture et les permissions.

### Erreur de syntaxe

**Symptôme** : MariaDB refuse de démarrer après modification.

```bash
# Vérifier les logs d'erreur
tail -f /var/log/mysql/error.log

# Exemple d'erreur
# [ERROR] Unknown option 'max-connections'
# Correction: utiliser max_connections (underscore)
```

**Solution** : Respecter la syntaxe exacte (underscore vs dash).

### Conflit de valeurs

**Symptôme** : La valeur affichée ne correspond pas à `my.cnf`.

```sql
-- Dans my.cnf
max_connections = 500

-- Mais en SQL
SHOW VARIABLES LIKE 'max_connections';
-- Résultat: 1000 (différent !)
```

**Cause possible** :
1. Fichier utilisateur `~/.my.cnf` surcharge
2. Option ligne de commande `--max-connections=1000`
3. Variable définie par `SET GLOBAL` précédemment

**Diagnostic** :

```bash
# Afficher la commande complète de démarrage
ps aux | grep mysqld

# Vérifier tous les fichiers
mariadb --print-defaults
```

---

## ✅ Points clés à retenir

- **Hiérarchie** : Fichiers système → fichiers utilisateur → ligne de commande → SET GLOBAL
- **Localisation** : `/etc/my.cnf`, `/etc/mysql/my.cnf`, `~/.my.cnf` (Linux) ; `C:\my.ini` (Windows)
- **Modularité** : Utilisez `!includedir` pour organiser par thématique (10-base.cnf, 20-innodb.cnf, etc.)
- **Sections** : `[client]` (clients), `[mysqld]` (serveur), `[mariadb]` (spécifique MariaDB)
- **Priorité** : Dernier fichier lu = priorité ; ligne de commande = priorité absolue
- **Sécurité** : `chmod 600 ~/.my.cnf` ; préférer `.mylogin.cnf` ou secrets managers
- **Validation** : `mysqld --verbose --help` avant redémarrage
- **Sauvegarde** : TOUJOURS sauvegarder avant modification (`cp` ou Git)
- **Documentation** : Commenter le "pourquoi", pas seulement le "quoi"
- **Versionning** : Git + Ansible pour configuration as code
- **UTF-8** : 🆕 MariaDB 11.8 utilise `utf8mb4` par défaut
- **Nouveautés 11.8** : `max_tmp_space_usage`, `innodb_alter_copy_bulk`, extension TIMESTAMP

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Option Files](https://mariadb.com/kb/en/configuring-mariadb-with-option-files/)
- [📖 Documentation officielle - Server System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [📖 Documentation officielle - mysql_config_editor](https://mariadb.com/kb/en/mysql_config_editor/)
- [🔧 Configuration Wizard](https://tools.percona.com/wizard) - Générateur de configuration Percona
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)

---

## ➡️ Section suivante

**[11.1.1 my.cnf / my.ini : Structure et sections](./01.1-structure-mycnf.md)** : Détail approfondi de la structure des fichiers de configuration, syntaxe complète, et organisation des sections pour une configuration professionnelle.

---

**💡 Conseil final** : Traitez votre configuration MariaDB comme du **code source** : versionning, revue, tests, documentation. Une configuration bien gérée est la base d'un système stable et performant. 🚀

⏭️ [my.cnf / my.ini : Structure et sections](/11-administration-configuration/01.1-structure-mycnf.md)

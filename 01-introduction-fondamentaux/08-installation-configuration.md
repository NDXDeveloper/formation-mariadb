🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.8 Installation et configuration initiale 🔧

> **Niveau** : Débutant
> **Durée estimée** : 1 heure (pratique incluse)
> **Prérequis** : Sections 1.1 à 1.7 (connaissances théoriques)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Installer MariaDB sur Linux, Windows et macOS
- Choisir la bonne méthode d'installation selon votre contexte
- Effectuer la configuration initiale sécurisée
- Vous connecter à MariaDB pour la première fois
- Vérifier que l'installation fonctionne correctement
- Comprendre les fichiers de configuration essentiels
- Résoudre les problèmes d'installation courants

---

## Introduction

**Passons à la pratique !** 🚀

Vous avez appris la théorie, découvert l'écosystème MariaDB, compris les versions LTS vs Rolling. Il est maintenant temps d'**installer MariaDB** sur votre machine et de le configurer.

**Ce que nous allons faire** :
1. ✅ **Choisir** la version à installer (11.8 LTS recommandée)
2. ✅ **Installer** MariaDB selon votre OS
3. ✅ **Sécuriser** l'installation
4. ✅ **Se connecter** pour la première fois
5. ✅ **Vérifier** que tout fonctionne

**Important** : Cette section contient des commandes à exécuter. Prenez le temps de les comprendre avant de les lancer !

---

## Quelle version installer ?

### 🎯 Recommandation par profil

| Profil | Version recommandée | Raison |
|--------|---------------------|--------|
| 🎓 **Apprenant** | **11.8 LTS** | Stable, documentée, support long |
| 💼 **Production** | **11.8 LTS** ou 11.4 LTS | Stabilité maximale, support 3 ans |
| 🚀 **Développeur** | **11.8 LTS** | Features récentes + stabilité |
| 🧪 **Expérimentation** | 12.2 Rolling | Dernières features |
| 🏢 **Entreprise** | 11.4 LTS ou 10.11 LTS | Déjà testée en prod |

### 💡 Notre choix pour cette formation

**Nous installerons MariaDB 11.8 LTS** car :
- ✅ Version LTS la plus récente (Juin 2025)
- ✅ Toutes les nouvelles features (Vector, TLS, etc.)
- ✅ Support garanti jusqu'en Juin 2028
- ✅ Documentation complète
- ✅ Communauté active

---

## Préparation : Vérifications préalables

### 🔍 Vérifier si MariaDB/MySQL est déjà installé

Avant d'installer, vérifions qu'aucune version n'est déjà présente :

**Linux** :
```bash
# Vérifier si MariaDB est installé
which mariadb
# ou
which mysql

# Vérifier si un serveur tourne
sudo systemctl status mariadb
# ou
sudo systemctl status mysql
```

**Windows** :
```powershell
# Ouvrir PowerShell et taper :
Get-Service | Where-Object {$_.Name -like "*mysql*" -or $_.Name -like "*maria*"}
```

**macOS** :
```bash
# Terminal
which mariadb
which mysql

# Vérifier les services
brew services list | grep -i mariadb
```

**Si une version existe déjà** :
- Option 1 : La désinstaller avant (recommandé pour débutants)
- Option 2 : Installer sur un port différent (avancé)

### 💾 Espace disque requis

**Minimum requis** :
- 💿 **Installation** : 500 MB
- 📊 **Données** : 1 GB minimum (variable selon usage)
- 📚 **Recommandé** : 10+ GB disponibles

**Vérifier l'espace disponible** :

Linux :
```bash
df -h /var/lib/mysql
```

Windows :
```powershell
Get-PSDrive C | Select-Object Used,Free
```

macOS :
```bash
df -h /usr/local/mysql
```

---

## Installation sur Linux (Ubuntu/Debian)

### 📦 Méthode 1 : Dépôts officiels MariaDB (Recommandée)

**Avantages** :
- ✅ Version la plus récente (11.8)
- ✅ Mises à jour automatiques
- ✅ Support officiel

#### Étape 1 : Ajouter le dépôt MariaDB

```bash
# 1. Installer les prérequis
sudo apt update
sudo apt install -y software-properties-common gnupg2

# 2. Importer la clé GPG MariaDB
sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'

# 3. Ajouter le dépôt MariaDB 11.8
sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] https://mirror.23m.com/mariadb/repo/11.8/ubuntu focal main'

# Note : Remplacer 'focal' par votre version Ubuntu :
# - Ubuntu 20.04 : focal
# - Ubuntu 22.04 : jammy
# - Ubuntu 24.04 : noble
```

**Alternative : Script automatique** 🎯

```bash
# Script officiel MariaDB (méthode la plus simple)
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="11.8"
```

#### Étape 2 : Installer MariaDB

```bash
# Mettre à jour la liste des paquets
sudo apt update

# Installer MariaDB Server
sudo apt install -y mariadb-server

# Vérifier l'installation
mariadb --version
# Résultat attendu : mariadb Ver 15.1 Distrib 11.8.x-MariaDB
```

#### Étape 3 : Démarrer le service

```bash
# Démarrer MariaDB
sudo systemctl start mariadb

# Activer au démarrage
sudo systemctl enable mariadb

# Vérifier le statut
sudo systemctl status mariadb
```

**Résultat attendu** :
```
● mariadb.service - MariaDB 11.8.x database server
     Loaded: loaded (/lib/systemd/system/mariadb.service; enabled)
     Active: active (running) since ...
```

### 📦 Méthode 2 : Paquets distribution (Plus ancien)

**Ubuntu/Debian** ont MariaDB dans leurs dépôts, mais version plus ancienne.

```bash
# Installer depuis dépôts Ubuntu (exemple : 10.6 sur Ubuntu 22.04)
sudo apt update
sudo apt install -y mariadb-server

# Note : Version dépend de votre distribution
```

⚠️ **Limitation** : Version souvent plus ancienne (10.6, 10.11).

---

## Installation sur Linux (Red Hat/CentOS/Rocky)

### 📦 Installation via dépôts officiels

#### Étape 1 : Ajouter le dépôt MariaDB

```bash
# Créer le fichier de configuration du dépôt
sudo nano /etc/yum.repos.d/MariaDB.repo
```

**Contenu du fichier** (Rocky Linux 9 / RHEL 9) :
```ini
[mariadb]
name = MariaDB
baseurl = https://mirror.23m.com/mariadb/yum/11.8/rhel9-amd64
gpgkey = https://mirror.23m.com/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck = 1
```

**Pour d'autres versions** :
- RHEL 8 / Rocky 8 : `rhel8-amd64`
- CentOS 7 : `centos7-amd64`

#### Étape 2 : Installer MariaDB

```bash
# Installer
sudo dnf install -y MariaDB-server MariaDB-client

# ou sur CentOS 7 :
sudo yum install -y MariaDB-server MariaDB-client
```

#### Étape 3 : Démarrer le service

```bash
# Démarrer
sudo systemctl start mariadb

# Activer au démarrage
sudo systemctl enable mariadb

# Vérifier
sudo systemctl status mariadb
```

---

## Installation sur Windows

### 💻 Méthode MSI Installer (Recommandée)

#### Étape 1 : Télécharger l'installeur

1. Aller sur : https://mariadb.org/download/
2. Choisir :
   - **Version** : 11.8 (LTS)
   - **OS** : Windows
   - **Architecture** : x86_64 (64-bit)
3. Télécharger le fichier MSI (ex: `mariadb-11.8.1-winx64.msi`)

#### Étape 2 : Lancer l'installation

1. **Double-cliquer** sur le fichier MSI
2. **Assistant d'installation** :
   - Cliquer "Next"
   - Accepter la licence : "I accept"
   - Choisir les composants :
     ✅ **Database Instance** (Server)
     ✅ **Client Programs** (mariadb.exe)
     ✅ **Development Components** (pour devs)
     ⬜ HeidiSQL (optionnel, client GUI)

#### Étape 3 : Configuration pendant l'installation

**Écran "Database Configuration"** :

1. **Mot de passe root** :
   ```
   Choisir un mot de passe FORT :
   Exemple : MySecureP@ssw0rd!2025

   ⚠️ IMPORTANT : Noter ce mot de passe !
   ```

2. **Use UTF8 as default server's character set** : ✅ Cocher

3. **Enable networking** : ✅ Cocher
   - Port : `3306` (défaut)

4. **Install as service** : ✅ Cocher
   - Service Name : `MariaDB`
   - ✅ Enable on startup

5. **Feedback** : ⬜ Décocher (optionnel)

#### Étape 4 : Finaliser

1. Cliquer "Next" puis "Install"
2. Attendre l'installation (~2-3 minutes)
3. Cliquer "Finish"

#### Étape 5 : Vérifier l'installation

**Ouvrir PowerShell ou CMD** :

```powershell
# Vérifier le service
Get-Service MariaDB

# Se connecter (depuis cmd ou PowerShell)
cd "C:\Program Files\MariaDB 11.8\bin"
.\mariadb.exe -u root -p
# Entrer le mot de passe
```

**Résultat attendu** :
```
Welcome to the MariaDB monitor.
Commands end with ; or \g.
Your MariaDB connection id is 3
Server version: 11.8.1-MariaDB

MariaDB [(none)]>
```

---

## Installation sur macOS

### 🍺 Méthode Homebrew (Recommandée)

#### Étape 1 : Installer Homebrew (si pas déjà fait)

```bash
# Vérifier si Homebrew est installé
brew --version

# Si pas installé :
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### Étape 2 : Installer MariaDB

```bash
# Installer MariaDB (version la plus récente via Homebrew)
brew install mariadb

# Vérifier la version installée
mariadb --version
```

💡 **Note** : Homebrew installe généralement une version récente (11.4+ ou 11.8).

#### Étape 3 : Démarrer le service

```bash
# Démarrer MariaDB maintenant
brew services start mariadb

# Vérifier le statut
brew services list | grep mariadb
# Résultat : mariadb started ...
```

#### Étape 4 : Accéder à MariaDB

```bash
# Se connecter (pas de mot de passe par défaut sur macOS)
mariadb
```

### 📦 Méthode alternative : DMG Installer

1. Télécharger le DMG depuis mariadb.org
2. Ouvrir le fichier DMG
3. Glisser MariaDB dans Applications
4. Configurer via l'interface graphique

---

## Installation via Docker 🐳 (Multi-plateforme)

### Pourquoi Docker ?

**Avantages** :
- ✅ Installation ultra-rapide (1 commande)
- ✅ Isolement (pas d'impact sur le système)
- ✅ Facile à supprimer/recréer
- ✅ Idéal pour développement/apprentissage

**Idéal pour** :
- 🎓 Apprentissage
- 🧪 Tests
- 💻 Développement local

### Installation Docker MariaDB

#### Étape 1 : Installer Docker

**Linux** :
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Redémarrer session ou reboot
```

**Windows/macOS** :
- Télécharger Docker Desktop : https://www.docker.com/products/docker-desktop/

#### Étape 2 : Lancer MariaDB

**Commande de base** :
```bash
docker run -d \
  --name mariadb-11-8 \
  -e MARIADB_ROOT_PASSWORD=MySecurePassword123 \
  -p 3306:3306 \
  -v mariadb_data:/var/lib/mysql \
  mariadb:11.8
```

**Explication des paramètres** :
```
-d                         : Détaché (background)
--name mariadb-11-8        : Nom du container
-e MARIADB_ROOT_PASSWORD   : Mot de passe root
-p 3306:3306               : Port hôte:container
-v mariadb_data:/var/lib   : Volume persistant
mariadb:11.8               : Image MariaDB 11.8
```

#### Étape 3 : Se connecter

```bash
# Se connecter au container
docker exec -it mariadb-11-8 mariadb -u root -p
# Entrer le mot de passe : MySecurePassword123
```

#### Commandes Docker utiles

```bash
# Voir les containers en cours
docker ps

# Arrêter MariaDB
docker stop mariadb-11-8

# Démarrer MariaDB
docker start mariadb-11-8

# Voir les logs
docker logs mariadb-11-8

# Supprimer le container (données préservées dans le volume)
docker rm mariadb-11-8

# Supprimer aussi les données
docker volume rm mariadb_data
```

---

## Sécurisation de l'installation

### 🔒 Script de sécurisation : mysql_secure_installation

**Après installation**, il est **CRUCIAL** de sécuriser MariaDB.

#### Sur Linux / macOS

```bash
sudo mysql_secure_installation
```

#### Sur Windows

```powershell
cd "C:\Program Files\MariaDB 11.8\bin"
.\mysql_secure_installation.exe
```

### 📋 Déroulement du script

```
Enter current password for root (enter for none):
[Appuyer sur Entrée si pas de mot de passe, ou entrer le mot de passe]

Switch to unix_socket authentication [Y/n] n
[Répondre 'n' - Garder mot de passe]

Change the root password? [Y/n] y
New password: **********
Re-enter new password: **********
[Choisir un MOT DE PASSE FORT]

Remove anonymous users? [Y/n] y
[Répondre 'y' - Supprimer utilisateurs anonymes]

Disallow root login remotely? [Y/n] y
[Répondre 'y' - Root seulement en local]

Remove test database? [Y/n] y
[Répondre 'y' - Supprimer la base test]

Reload privilege tables now? [Y/n] y
[Répondre 'y' - Recharger les privilèges]
```

### ✅ Résultat de la sécurisation

Après le script :
- ✅ Mot de passe root défini
- ✅ Pas d'utilisateurs anonymes
- ✅ Root accessible uniquement en local
- ✅ Base de données test supprimée
- ✅ Privilèges rechargés

---

## Première connexion

### 💻 Se connecter en ligne de commande

#### Linux / macOS

```bash
# Se connecter en tant que root
mariadb -u root -p
# ou
mysql -u root -p

# Entrer le mot de passe
```

#### Windows

```powershell
cd "C:\Program Files\MariaDB 11.8\bin"
.\mariadb.exe -u root -p
```

#### Docker

```bash
docker exec -it mariadb-11-8 mariadb -u root -p
```

### 🎉 Premier contact avec MariaDB

**Vous devriez voir** :
```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8
Server version: 11.8.1-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

🎊 **Félicitations ! Vous êtes connecté à MariaDB !**

### 📝 Premières commandes

```sql
-- Afficher la version
SELECT VERSION();

-- Voir les bases de données
SHOW DATABASES;

-- Voir l'utilisateur actuel
SELECT USER();

-- Voir la date/heure du serveur
SELECT NOW();

-- Afficher les variables système
SHOW VARIABLES LIKE 'version%';

-- Sortir de MariaDB
EXIT;
-- ou
QUIT;
-- ou
\q
```

---

## Vérifications post-installation

### ✅ Checklist de vérification

#### 1️⃣ Service actif

**Linux** :
```bash
sudo systemctl status mariadb
# Doit afficher : active (running)
```

**Windows** :
```powershell
Get-Service MariaDB
# Status : Running
```

**macOS** :
```bash
brew services list | grep mariadb
# mariadb started
```

#### 2️⃣ Version correcte

```bash
mariadb --version
# Doit afficher : 11.8.x-MariaDB
```

#### 3️⃣ Connexion fonctionne

```bash
mariadb -u root -p
# Entrer mot de passe → Doit se connecter
```

#### 4️⃣ Port 3306 écoute

**Linux / macOS** :
```bash
sudo netstat -tlnp | grep 3306
# ou
sudo ss -tlnp | grep 3306
```

**Windows** :
```powershell
netstat -an | findstr 3306
```

**Résultat attendu** :
```
tcp  0  0  127.0.0.1:3306  0.0.0.0:*  LISTEN
```

#### 5️⃣ Fichiers de configuration présents

**Linux** :
```bash
ls -l /etc/mysql/
ls -l /etc/my.cnf
```

**Windows** :
```
C:\Program Files\MariaDB 11.8\data\my.ini
```

**macOS** :
```bash
ls -l /usr/local/etc/my.cnf
```

---

## Configuration de base

### 📝 Fichier de configuration : my.cnf / my.ini

**Emplacements selon OS** :

| OS | Emplacement principal |
|----|----------------------|
| **Linux (Ubuntu)** | `/etc/mysql/my.cnf` ou `/etc/my.cnf` |
| **Linux (RHEL)** | `/etc/my.cnf.d/server.cnf` |
| **Windows** | `C:\Program Files\MariaDB 11.8\data\my.ini` |
| **macOS** | `/usr/local/etc/my.cnf` |

### 🔧 Configuration minimale recommandée

**Éditer le fichier** :

Linux :
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
# ou
sudo nano /etc/my.cnf
```

**Ajouter/modifier dans la section [mysqld]** :

```ini
[mysqld]
# Bind address (127.0.0.1 = local uniquement, 0.0.0.0 = tout réseau)
bind-address = 127.0.0.1

# Port
port = 3306

# Character set (UTF-8)
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# InnoDB Buffer Pool (50-70% de la RAM disponible)
# Exemple pour 4GB RAM : 2GB
innodb_buffer_pool_size = 2G

# Log des requêtes lentes (utile pour debug)
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2

# Logs d'erreur
log_error = /var/log/mysql/error.log

# Max connections
max_connections = 150
```

**Après modification, redémarrer** :

Linux :
```bash
sudo systemctl restart mariadb
```

Windows :
```powershell
Restart-Service MariaDB
```

macOS :
```bash
brew services restart mariadb
```

### 🔍 Vérifier la configuration

```sql
-- Se connecter
mariadb -u root -p

-- Voir toutes les variables
SHOW VARIABLES;

-- Voir une variable spécifique
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

---

## Créer un utilisateur pour vos applications

### 👤 Ne JAMAIS utiliser root en production !

**Mauvaise pratique** ❌ :
```
Application → root@localhost
```

**Bonne pratique** ✅ :
```
Application → app_user@localhost (privilèges limités)
```

### 📝 Créer un utilisateur applicatif

```sql
-- Se connecter en root
mariadb -u root -p

-- Créer une base de données
CREATE DATABASE myapp_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Créer un utilisateur
CREATE USER 'myapp_user'@'localhost' IDENTIFIED BY 'SecurePassword123!';

-- Donner les droits sur la base uniquement
GRANT ALL PRIVILEGES ON myapp_db.* TO 'myapp_user'@'localhost';

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Vérifier
SHOW GRANTS FOR 'myapp_user'@'localhost';
```

**Test de connexion** :
```bash
mariadb -u myapp_user -p myapp_db
# Entrer : SecurePassword123!
```

### 🌍 Accès distant (si nécessaire)

**Par défaut, MariaDB accepte seulement les connexions locales**.

Pour autoriser l'accès distant :

1. **Modifier bind-address** dans my.cnf :
   ```ini
   bind-address = 0.0.0.0
   ```

2. **Créer utilisateur avec accès distant** :
   ```sql
   CREATE USER 'remote_user'@'%' IDENTIFIED BY 'StrongPass123!';
   GRANT ALL PRIVILEGES ON myapp_db.* TO 'remote_user'@'%';
   FLUSH PRIVILEGES;
   ```

3. **Ouvrir le firewall** :

   Linux (UFW) :
   ```bash
   sudo ufw allow 3306/tcp
   ```

   Linux (firewalld) :
   ```bash
   sudo firewall-cmd --permanent --add-port=3306/tcp
   sudo firewall-cmd --reload
   ```

⚠️ **Attention** : Accès distant = risque sécurité ! À utiliser avec précaution.

---

## Outils clients graphiques (optionnel)

### 🖥️ Clients GUI recommandés

#### 1️⃣ **HeidiSQL** (Windows, via Wine sur Linux)

**Installation Windows** :
- Télécharger : https://www.heidisql.com/
- Installer normalement
- Connexion :
  - Host : `localhost`
  - User : `root`
  - Password : votre mot de passe
  - Port : `3306`

#### 2️⃣ **DBeaver** (Multi-plateforme)

**Installation** :
- Télécharger : https://dbeaver.io/
- Disponible : Windows, macOS, Linux
- Gratuit et open source

**Connexion** :
1. New Connection → MariaDB
2. Host : `localhost`, Port : `3306`
3. Username : `root`, Password : votre mdp
4. Test Connection → Finish

#### 3️⃣ **phpMyAdmin** (Web, toutes plateformes)

**Installation via Docker** (le plus simple) :
```bash
docker run -d \
  --name phpmyadmin \
  -e PMA_HOST=host.docker.internal \
  -p 8080:80 \
  phpmyadmin/phpmyadmin

# Accéder : http://localhost:8080
# User : root
# Password : votre mot de passe
```

#### 4️⃣ **TablePlus** (macOS, Windows, Linux)

**Installation** :
- Télécharger : https://tableplus.com/
- Version gratuite limitée
- Interface moderne

#### 5️⃣ **DataGrip** (Multi-plateforme, payant)

**Par JetBrains** :
- Télécharger : https://www.jetbrains.com/datagrip/
- Version d'essai 30 jours
- Très complet, pour développeurs

---

## Dépannage : Problèmes courants

### ❌ Problème 1 : Service ne démarre pas

**Symptôme** :
```bash
sudo systemctl start mariadb
Job for mariadb.service failed
```

**Solutions** :

1. **Vérifier les logs** :
   ```bash
   sudo journalctl -u mariadb -n 50
   # ou
   sudo tail -f /var/log/mysql/error.log
   ```

2. **Permissions incorrectes** :
   ```bash
   sudo chown -R mysql:mysql /var/lib/mysql
   sudo chmod -R 755 /var/lib/mysql
   ```

3. **Port déjà utilisé** :
   ```bash
   sudo netstat -tlnp | grep 3306
   # Si autre processus → le stopper ou changer port MariaDB
   ```

### ❌ Problème 2 : "Access denied for user 'root'@'localhost'"

**Symptôme** :
```
ERROR 1045 (28000): Access denied for user 'root'@'localhost'
```

**Solutions** :

**Méthode 1 : Réinitialiser mot de passe root**

Linux :
```bash
# 1. Arrêter MariaDB
sudo systemctl stop mariadb

# 2. Démarrer en mode sans vérification
sudo mysqld_safe --skip-grant-tables &

# 3. Se connecter sans mot de passe
mariadb -u root

# 4. Réinitialiser le mot de passe
USE mysql;
UPDATE user SET password=PASSWORD('NewPassword123') WHERE User='root';
FLUSH PRIVILEGES;
EXIT;

# 5. Tuer le processus et redémarrer normalement
sudo killall mysqld
sudo systemctl start mariadb
```

**Méthode 2 : Utiliser sudo (Linux uniquement)**
```bash
# Sur Linux, root peut se connecter via sudo sans mot de passe
sudo mariadb
# Puis réinitialiser le mot de passe depuis là
```

### ❌ Problème 3 : "Can't connect to local MySQL server"

**Symptôme** :
```
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock'
```

**Cause** : Service pas démarré ou socket incorrect.

**Solutions** :

1. **Démarrer le service** :
   ```bash
   sudo systemctl start mariadb
   ```

2. **Vérifier le socket** :
   ```bash
   # Trouver le socket
   sudo find / -name mysqld.sock

   # Se connecter avec le bon socket
   mariadb -u root -p --socket=/var/run/mysqld/mysqld.sock
   ```

### ❌ Problème 4 : "Too many connections"

**Symptôme** :
```
ERROR 1040 (HY000): Too many connections
```

**Solution immédiate** :
```bash
# Augmenter max_connections temporairement
mariadb -u root -p
SET GLOBAL max_connections = 300;
```

**Solution permanente** : Modifier my.cnf
```ini
[mysqld]
max_connections = 300
```

### ❌ Problème 5 : Port 3306 déjà utilisé (Windows)

**Vérifier le processus** :
```powershell
netstat -ano | findstr 3306
```

**Changer le port de MariaDB** :

Modifier `my.ini` :
```ini
[mysqld]
port = 3307
```

Redémarrer le service.

---

## ✅ Points clés à retenir

- 📦 **Version recommandée** : MariaDB 11.8 LTS (stable, support 3 ans)
- 🐧 **Linux** : Installation via dépôts officiels ou Docker
- 🪟 **Windows** : MSI Installer (simple et guidé)
- 🍎 **macOS** : Homebrew (rapide et efficace)
- 🐳 **Docker** : Idéal pour dev/test, isolation complète
- 🔒 **Sécurisation obligatoire** : `mysql_secure_installation`
- 🔑 **Mot de passe fort** : Toujours définir un mot de passe root
- 👤 **Utilisateurs applicatifs** : Ne jamais utiliser root en prod
- 📝 **Fichier config** : my.cnf (Linux) ou my.ini (Windows)
- 🔧 **Configuration minimale** : UTF-8, InnoDB buffer pool, bind-address
- 🌐 **Accès distant** : Par défaut désactivé (sécurité)
- 🖥️ **Clients GUI** : HeidiSQL, DBeaver, phpMyAdmin, TablePlus
- ✅ **Vérifications** : Service actif, version, connexion, port 3306

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [MariaDB Downloads](https://mariadb.org/download/)
- [Installation Guide](https://mariadb.com/kb/en/getting-installing-and-upgrading-mariadb/)
- [Repository Configuration Tool](https://mariadb.org/download/?t=repo-config)

### 🐳 Docker
- [MariaDB Docker Hub](https://hub.docker.com/_/mariadb)
- [Docker Compose exemples](https://github.com/MariaDB/mariadb-docker)

### 🔧 Configuration
- [Server System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [my.cnf Configuration File](https://mariadb.com/kb/en/configuring-mariadb-with-option-files/)

### 🛡️ Sécurité
- [Security Guide](https://mariadb.com/kb/en/security/)
- [mysql_secure_installation](https://mariadb.com/kb/en/mysql_secure_installation/)

### 🖥️ Clients GUI
- [HeidiSQL](https://www.heidisql.com/)
- [DBeaver](https://dbeaver.io/)
- [phpMyAdmin](https://www.phpmyadmin.net/)
- [TablePlus](https://tableplus.com/)

---

## ➡️ Section suivante

**[1.9 - Outils d'administration et clients](./09-outils-administration.md)**

Maintenant que MariaDB est installé et configuré, explorons dans la section suivante les **outils d'administration** essentiels : ligne de commande avancée, clients graphiques professionnels, outils de backup, monitoring, et bien plus. Vous découvrirez comment gérer efficacement vos bases de données MariaDB au quotidien.

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.8*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Outils d'administration (CLI, HeidiSQL, DBeaver, phpMyAdmin)](/01-introduction-fondamentaux/09-outils-administration.md)

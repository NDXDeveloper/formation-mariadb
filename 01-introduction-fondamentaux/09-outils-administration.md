🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.9 Outils d'administration (CLI, HeidiSQL, DBeaver, phpMyAdmin) 🛠️

> **Niveau** : Débutant
> **Durée estimée** : 1 heure
> **Prérequis** : Section 1.8 (MariaDB installé et configuré)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser le client en ligne de commande (CLI) de MariaDB
- Utiliser HeidiSQL pour administration Windows
- Utiliser DBeaver comme client universel
- Installer et configurer phpMyAdmin
- Choisir le bon outil selon le contexte
- Effectuer les tâches d'administration courantes
- Comprendre les avantages et limites de chaque outil

---

## Introduction

Maintenant que MariaDB est installé, **comment l'administrer au quotidien ?** 🤔

Il existe de nombreux outils, chacun avec ses forces et faiblesses :
- 🖥️ **CLI** (Command Line Interface) : Rapide, scriptable, puissant
- 🪟 **HeidiSQL** : Simple, léger, parfait pour Windows
- 🌐 **DBeaver** : Universel, riche en fonctionnalités
- 🌍 **phpMyAdmin** : Accessible via navigateur, classique

**Dans cette section**, nous explorerons ces 4 outils principaux et comment choisir le bon selon vos besoins.

---

## Vue d'ensemble des outils

### 📊 Comparaison rapide

| Outil | Type | Plateformes | Niveau | Cas d'usage |
|-------|------|-------------|--------|-------------|
| **CLI (mariadb)** | Terminal | Toutes | Débutant→Expert | Scripts, serveur distant, automatisation |
| **HeidiSQL** | Desktop | Windows (+Wine) | Débutant | Admin Windows rapide |
| **DBeaver** | Desktop | Windows, Mac, Linux | Intermédiaire | Développement multi-DB |
| **phpMyAdmin** | Web | Toutes (serveur web) | Débutant | Accès web, hébergement |
| **TablePlus** | Desktop | Mac, Windows, Linux | Débutant | Interface moderne |
| **DataGrip** | Desktop (IDE) | Windows, Mac, Linux | Avancé | Dev professionnel |

### 🎯 Quel outil choisir ?

**Débutant complet** :
- 🪟 Windows → **HeidiSQL**
- 🍎 Mac → **TablePlus** ou **DBeaver**
- 🐧 Linux → **DBeaver**
- 🌐 Besoin web → **phpMyAdmin**

**Développeur** :
- Multi-DB → **DBeaver**
- Pro JetBrains → **DataGrip**
- Scripter → **CLI**

**Administrateur système** :
- **CLI** (obligatoire à maîtriser)
- + GUI au choix selon OS

---

## 1. Client CLI : mariadb (ligne de commande)

### 📚 Pourquoi maîtriser le CLI ?

**Avantages du CLI** :
- ✅ **Disponible partout** (serveurs sans GUI)
- ✅ **Rapide** (pas de surcharge graphique)
- ✅ **Scriptable** (automatisation)
- ✅ **Puissant** (toutes les fonctionnalités)
- ✅ **SSH-friendly** (administration distante)

💡 **Conseil** : Même si vous préférez les interfaces graphiques, apprenez le CLI ! C'est essentiel pour l'administration professionnelle.

### 🚀 Connexion de base

```bash
# Connexion locale en root
mariadb -u root -p

# Connexion à une base spécifique
mariadb -u root -p myapp_db

# Connexion à un serveur distant
mariadb -h 192.168.1.100 -u admin -p production_db

# Spécifier le port (si non-standard)
mariadb -h example.com -P 3307 -u user -p
```

**Syntaxe complète** :
```bash
mariadb [options] [database]

Options courantes :
-u, --user=name       Nom d'utilisateur
-p, --password        Demander le mot de passe (ne pas le mettre en clair !)
-h, --host=name       Hôte (localhost par défaut)
-P, --port=#          Port (3306 par défaut)
-D, --database=name   Base de données à utiliser
-e, --execute=query   Exécuter une requête et sortir
```

### 📝 Commandes essentielles

#### Commandes MySQL/MariaDB (dans le prompt)

```sql
-- Obtenir de l'aide
help;
\h

-- Voir les bases de données
SHOW DATABASES;

-- Utiliser une base
USE myapp_db;

-- Voir les tables
SHOW TABLES;

-- Décrire une table
DESCRIBE users;
-- ou
DESC users;

-- Voir la structure de création
SHOW CREATE TABLE users\G

-- Voir les utilisateurs
SELECT User, Host FROM mysql.user;

-- Voir les processus en cours
SHOW PROCESSLIST;

-- Voir les variables système
SHOW VARIABLES;
SHOW VARIABLES LIKE 'max_connections';

-- Voir les status
SHOW STATUS;
SHOW STATUS LIKE 'Threads_connected';

-- Afficher les warnings
SHOW WARNINGS;

-- Afficher les erreurs
SHOW ERRORS;
```

#### Commandes shell (caractères spéciaux)

```sql
-- Quitter
EXIT;
QUIT;
\q

-- Effacer l'écran
\! clear       -- Linux/Mac
\! cls         -- Windows

-- Exécuter une commande shell
\! ls -la      -- Linux/Mac
\! dir         -- Windows

-- Éditer dans un éditeur
\e

-- Afficher en vertical (G au lieu de ;)
SELECT * FROM users\G

-- Afficher sans la grille (moins d'espacement)
SELECT * FROM users\g

-- Changer de base de données
\u myapp_db

-- Afficher la base actuelle
SELECT DATABASE();

-- Recharger les privilèges
FLUSH PRIVILEGES;
```

### 🎨 Formatage de sortie

**Par défaut (table)** :
```sql
MariaDB [test]> SELECT id, name FROM users LIMIT 2;
+----+-------+
| id | name  |
+----+-------+
|  1 | Alice |
|  2 | Bob   |
+----+-------+
```

**Format vertical (\G)** :
```sql
MariaDB [test]> SELECT * FROM users LIMIT 1\G
*************************** 1. row ***************************
      id: 1
    name: Alice
   email: alice@example.com
created: 2025-01-15 10:30:00
```

**Format HTML (pour export web)** :
```bash
mariadb -u root -p --html -e "SELECT * FROM users" > output.html
```

**Format XML** :
```bash
mariadb -u root -p --xml -e "SELECT * FROM users" > output.xml
```

**Format CSV (pratique pour exports)** :
```bash
mariadb -u root -p -e "SELECT * FROM users" | sed 's/\t/,/g' > output.csv
```

### 📁 Exécuter des fichiers SQL

**Méthode 1 : Depuis le prompt MariaDB**
```sql
-- Se connecter
mariadb -u root -p myapp_db

-- Exécuter un fichier
SOURCE /path/to/script.sql;
-- ou
\. /path/to/script.sql
```

**Méthode 2 : En ligne de commande**
```bash
mariadb -u root -p myapp_db < script.sql

# Avec affichage des erreurs
mariadb -u root -p myapp_db < script.sql 2>&1 | tee import.log
```

**Méthode 3 : Exécution directe (-e)**
```bash
mariadb -u root -p -e "CREATE DATABASE test_db; SHOW DATABASES;"
```

### 🔧 Configuration du client (.my.cnf)

**Créer un fichier de configuration personnel** pour éviter de retaper les options :

Linux/Mac : `~/.my.cnf`
Windows : `%APPDATA%\MySQL\.my.cnf`

```ini
[client]
user=root
password=VotreMotDePasse
host=localhost
port=3306

# Prompt personnalisé
prompt="\u@\h [\d]> "

# Auto-completion
auto-rehash

# Pas de bip
no-beep

# Pager (pour grandes sorties)
pager=less
```

⚠️ **Sécurité** : Protéger ce fichier !
```bash
chmod 600 ~/.my.cnf
```

**Connexion simplifiée** :
```bash
# Avec .my.cnf configuré, plus besoin de -u et -p !
mariadb myapp_db
```

### 💡 Astuces CLI

**1. Historique des commandes**

Le CLI garde un historique dans `~/.mysql_history` :
```bash
# Naviguer : Flèche haut/bas
# Rechercher : Ctrl+R
```

**2. Auto-complétion**

Appuyez sur **Tab** pour auto-compléter :
```sql
USE mya[Tab]  → myapp_db
SELECT * FROM us[Tab]  → users
```

**3. Édition multiligne**

```sql
MariaDB> SELECT id,
      ->        name,
      ->        email
      -> FROM users
      -> WHERE active = 1;
```

**4. Annuler une commande**

Ctrl+C ou taper `\c`

**5. Rediriger la sortie**

```bash
# Sauvegarder dans un fichier
mariadb -u root -p -e "SELECT * FROM users" > users.txt

# Utiliser un pager
mariadb -u root -p --pager=less
```

---

## 2. HeidiSQL (Windows)

### 📖 Présentation

**HeidiSQL** est un client SQL gratuit et open source pour Windows, très populaire pour sa simplicité.

**Points forts** :
- ✅ Gratuit et léger
- ✅ Interface intuitive
- ✅ Support MariaDB, MySQL, PostgreSQL, SQL Server
- ✅ Export/Import facile
- ✅ Éditeur SQL avec coloration syntaxique

**Site officiel** : https://www.heidisql.com/

### 📦 Installation

1. Télécharger depuis https://www.heidisql.com/download.php
2. Version portable disponible (pas d'installation)
3. Lancer l'installeur
4. Suivre l'assistant (options par défaut OK)

### 🔌 Configuration connexion

**Première connexion** :

1. **Lancer HeidiSQL**
2. **Fenêtre "Session manager"** s'ouvre
3. **Cliquer "New"** (Nouvelle session)
4. **Configurer** :

```
Network type : MariaDB or MySQL (TCP/IP)
Hostname / IP : localhost (ou IP distante)
User : root
Password : [votre mot de passe]
Port : 3306
```

5. **Tester** : Cliquer "Open" ou "Test connection"
6. **Sauvegarder** : Donner un nom (ex: "MariaDB Local")

### 🎨 Interface utilisateur

```
┌─────────────────────────────────────────────────────────────┐
│ File  Edit  Tools  Help                            [- □ X]  │
├────────┬────────────────────────────────────────────────────┤
│        │  myapp_db                                          │
│ ┌──┐   ├───── Tables (5)                                    │
│ └──┘   │      ├── users                                     │
│ Local  │      ├── orders                                    │
│        │      ├── products                                  │
│ ┌──┐   │      ├── categories                                │
│ └──┘   │      └── settings                                  │
│ Prod   ├───── Views (0)                                     │
│        ├───── Routines (3)                                  │
│        ├───── Triggers (1)                                  │
│        └───── Events (0)                                    │
├────────┼────────────────────────────────────────────────────┤
│        │  Data  |  Structure  |  Query  |  Info             │
│  DB    ├────────────────────────────────────────────────────┤
│  Tree  │  id  │  name    │  email              │  created   │
│        ├──────┼──────────┼─────────────────────┼────────────┤
│        │  1   │  Alice   │  alice@example.com  │  2025-...  │
│        │  2   │  Bob     │  bob@example.com    │  2025-...  │
│        │  ...                                               │
└────────┴────────────────────────────────────────────────────┘
```

### 🔧 Fonctionnalités principales

#### 1️⃣ **Visualiser et éditer les données**

- Cliquer sur une table → Onglet "Data"
- **Éditer** : Double-cliquer sur une cellule
- **Ajouter** : Cliquer icône "+"
- **Supprimer** : Sélectionner ligne → icône "-"
- **Sauvegarder** : Cliquer icône "💾" ou F9

#### 2️⃣ **Créer une table**

```
1. Clic droit sur "Tables" → Create new → Table
2. Définir le nom : users_new
3. Ajouter colonnes :
   - id : INT, Auto-increment, Primary Key
   - name : VARCHAR(100), NOT NULL
   - email : VARCHAR(150), NOT NULL
4. Cliquer "Save"
```

#### 3️⃣ **Exécuter des requêtes SQL**

- Onglet **"Query"** (ou F9)
- Taper la requête :
```sql
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
ORDER BY order_count DESC;
```
- **Exécuter** : F9 ou bouton ▶️

#### 4️⃣ **Exporter des données**

```
1. Clic droit sur table → Export grid rows
2. Choisir format :
   - CSV
   - Excel
   - JSON
   - XML
   - SQL (INSERT statements)
3. Configurer options
4. Exporter
```

#### 5️⃣ **Importer des données**

```
1. Clic droit sur table → Import CSV file
2. Sélectionner fichier
3. Mapper les colonnes
4. Importer
```

#### 6️⃣ **Gérer les utilisateurs**

```
Tools → User Manager
- Voir tous les users
- Créer/Modifier/Supprimer
- Gérer les privilèges
```

### 💡 Raccourcis clavier HeidiSQL

| Raccourci | Action |
|-----------|--------|
| **F9** | Exécuter requête |
| **Ctrl+N** | Nouvelle connexion |
| **Ctrl+T** | Nouvel onglet |
| **Ctrl+W** | Fermer onglet |
| **Ctrl+Space** | Auto-complétion SQL |
| **F5** | Rafraîchir |
| **Ctrl+F** | Rechercher |

---

## 3. DBeaver (Multi-plateforme)

### 📖 Présentation

**DBeaver** est un client SQL universel, open source, qui supporte plus de 80 bases de données.

**Points forts** :
- ✅ Gratuit (Community Edition)
- ✅ Multi-plateforme (Windows, Mac, Linux)
- ✅ Support MariaDB, MySQL, PostgreSQL, Oracle, SQL Server, etc.
- ✅ Rich feature set (ERD, comparaison, mock data)
- ✅ Édition avancée avec auto-complétion intelligente

**Site officiel** : https://dbeaver.io/

### 📦 Installation

**Windows** :
1. Télécharger installeur depuis https://dbeaver.io/download/
2. Lancer `dbeaver-ce-latest-x86_64-setup.exe`
3. Suivre l'assistant

**macOS** :
```bash
brew install --cask dbeaver-community
```

**Linux (Ubuntu/Debian)** :
```bash
wget https://dbeaver.io/files/dbeaver-ce_latest_amd64.deb
sudo dpkg -i dbeaver-ce_latest_amd64.deb
sudo apt-get install -f
```

### 🔌 Configuration connexion

**Première connexion MariaDB** :

1. **Lancer DBeaver**
2. **Cliquer** : "New Database Connection" (icône prise électrique)
3. **Sélectionner** : MariaDB
4. **Cliquer** : Next
5. **Configurer** :

```
Host : localhost
Port : 3306
Database : (laisser vide pour voir toutes les DB)
Username : root
Password : [votre mot de passe]
☑ Save password
```

6. **Test Connection** : Vérifier que ça marche
   - Si erreur driver : Cliquer "Download" pour télécharger le driver MariaDB

7. **Finish**

### 🎨 Interface utilisateur

```
┌───────────────────────────────────────────────────────────────┐
│ File  Edit  Navigate  SQL Editor  Database  Window  Help      │
├────────┬──────────────────────────────────────────────────────┤
│        │  SQL Editor - myapp_db                               │
│ ┌──┐   ├──────────────────────────────────────────────────────┤
│ └──┘   │ SELECT * FROM users WHERE active = 1;                │
│MariaDB │                                                      │
│ Local  │                                                      │
│        │ ▶ Execute  📋 Format  💾 Save  ...                   │
│ ├─📁   ├──────────────────────────────────────────────────────┤
│ │myapp │  Results                                             │
│ │ ├─📊 │ ┌────┬──────────┬────────────────────┬────────────┐  │
│ │ │Tabl│ │ id │ name     │ email              │ created    │  │
│ │ │ └us│ ├────┼──────────┼────────────────────┼────────────┤  │
│ │ │ └or│ │ 1  │ Alice    │ alice@example.com  │ 2025-01... │  │
│ │ │ └pr│ │ 2  │ Bob      │ bob@example.com    │ 2025-01... │  │
│ │ └──  │ └────┴──────────┴────────────────────┴────────────┘  │
└────────┴──────────────────────────────────────────────────────┘
```

### 🔧 Fonctionnalités principales

#### 1️⃣ **Éditeur SQL avancé**

**Créer un script SQL** :
- SQL Editor → New SQL Script (Ctrl+N)
- Taper requêtes :

```sql
-- DBeaver supporte multi-requêtes
SELECT * FROM users LIMIT 5;

SELECT COUNT(*) FROM orders;

-- Auto-complétion : Ctrl+Space
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**Exécuter** :
- **Ctrl+Enter** : Exécuter requête sous curseur
- **Ctrl+Alt+X** : Exécuter tout le script
- **Ctrl+Shift+E** : Exécuter requête sélectionnée

#### 2️⃣ **Visualisation des données**

**Options d'affichage** :
- Grid (tableau)
- Record (formulaire, 1 ligne à la fois)
- JSON
- XML

**Filtres et tri** :
- Clic sur en-tête colonne → Trier
- Clic droit → Filters → Ajouter conditions

#### 3️⃣ **Diagramme ER (Entity-Relationship)**

```
1. Clic droit sur base de données → View Diagram
2. Glisser-déposer tables sur canvas
3. DBeaver affiche automatiquement les relations (Foreign Keys)
4. Exporter en PNG/SVG
```

**Très utile** pour comprendre la structure d'une base complexe !

#### 4️⃣ **Comparaison de données**

```
1. Sélectionner 2 tables (Ctrl+clic)
2. Clic droit → Compare Data
3. DBeaver montre les différences
4. Optionnel : Générer script pour synchroniser
```

#### 5️⃣ **Génération de données de test**

```
1. Clic droit sur table → Generate SQL → Mock Data
2. Choisir le nombre de lignes
3. DBeaver génère des données réalistes (noms, emails, etc.)
4. Exécuter le script
```

#### 6️⃣ **Export/Import**

**Export** :
```
1. Sélectionner table ou résultat requête
2. Clic droit → Export Data
3. Choisir format : CSV, JSON, XML, SQL, Excel, etc.
4. Configurer options
5. Exporter
```

**Import** :
```
1. Clic droit sur table → Import Data
2. Choisir fichier source (CSV, Excel, etc.)
3. Mapper colonnes
4. Importer
```

### 💡 Astuces DBeaver

**1. Thèmes sombres**
```
Window → Preferences → User Interface → Appearance
→ Theme : Dark
```

**2. Auto-complétion intelligente**
```
Preferences → Editors → SQL Editor → Code Completion
☑ Enable auto completion
☑ Insert single proposals automatically
```

**3. Formater SQL automatiquement**
```
Sélectionner requête → Ctrl+Shift+F (Format)

Avant :
select id,name,email from users where active=1;

Après :
SELECT
    id,
    name,
    email
FROM users
WHERE active = 1;
```

**4. Snippets de code**
```
Preferences → Editors → SQL Editor → Templates
→ Créer des templates réutilisables

Exemple : "selall" → SELECT * FROM ${table} WHERE ${condition};
```

---

## 4. phpMyAdmin (Interface Web)

### 📖 Présentation

**phpMyAdmin** est l'interface web classique pour administrer MySQL/MariaDB, écrite en PHP.

**Points forts** :
- ✅ Accessible via navigateur (aucune installation client)
- ✅ Parfait pour hébergement web
- ✅ Interface connue de tous
- ✅ Très complet (toutes les opérations possibles)

**Cas d'usage** :
- Hébergement mutualisé (fourni par hébergeur)
- Administration web d'un serveur
- Accès multi-utilisateur sans installation

**Site officiel** : https://www.phpmyadmin.net/

### 📦 Installation

#### Méthode 1 : Docker (Recommandée pour test local)

```bash
docker run -d \
  --name phpmyadmin \
  --link mariadb-container:db \
  -e PMA_HOST=db \
  -p 8080:80 \
  phpmyadmin/phpmyadmin

# Accéder : http://localhost:8080
```

**Connexion standalone** (si MariaDB sur hôte) :
```bash
docker run -d \
  --name phpmyadmin \
  -e PMA_HOST=host.docker.internal \
  -p 8080:80 \
  phpmyadmin/phpmyadmin
```

#### Méthode 2 : Installation manuelle (Linux)

**Ubuntu/Debian** :
```bash
# Installer Apache, PHP et phpMyAdmin
sudo apt update
sudo apt install -y apache2 php php-mysql phpmyadmin

# Pendant l'installation, choisir :
# - Serveur web : apache2
# - Configurer avec dbconfig-common : Yes
# - Mot de passe phpMyAdmin : [choisir un mot de passe]

# Activer la configuration
sudo ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin

# Redémarrer Apache
sudo systemctl restart apache2

# Accéder : http://localhost/phpmyadmin
```

#### Méthode 3 : Package tout-en-un (Windows/Mac)

**XAMPP** inclut phpMyAdmin :
- Windows/Mac/Linux : https://www.apachefriends.org/
- Lance Apache, PHP, MariaDB et phpMyAdmin d'un coup

### 🔐 Première connexion

**Ouvrir dans navigateur** :
```
http://localhost/phpmyadmin
ou
http://localhost:8080  (si Docker)
```

**Page de login** :
```
┌────────────────────────────────────┐
│         phpMyAdmin                 │
├────────────────────────────────────┤
│                                    │
│  Username : [root              ]   │
│  Password : [**********        ]   │
│                                    │
│  [ Go ]                            │
│                                    │
└────────────────────────────────────┘
```

**Se connecter** :
- Username : `root`
- Password : [votre mot de passe MariaDB]

### 🎨 Interface utilisateur

```
┌──────────────────────────────────────────────────────────────┐
│ phpMyAdmin                          root@localhost  [Logout] │
├─────────────┬────────────────────────────────────────────────┤
│             │  Server: localhost (MariaDB 11.8.1)            │
│ 📊 Databases├────────────────────────────────────────────────┤
│ 🔍 SQL      │                                                │
│ 👤 Users    │  Databases (5)                                 │
│ 📈 Status   │  ┌──────────────┬─────────┬────────┬──────┐    │
│ ⚙️ Settings │  │ Database     │ Tables  │ Size   │ ...  │    │
│             │  ├──────────────┼─────────┼────────┼──────┤    │
│ myapp_db ▼  │  │ myapp_db     │ 5       │ 2.1 MB │ ...  │    │
│  ├─ users   │  │ information_ │ 79      │ 2.4 MB │ ...  │    │
│  ├─ orders  │  │ mysql        │ 31      │ 4.8 MB │ ...  │    │
│  └─ ...     │  │ performance_ │ 110     │ 0 B    │ ...  │    │
│             │  └──────────────┴─────────┴────────┴──────┘    │
└─────────────┴────────────────────────────────────────────────┘
```

### 🔧 Fonctionnalités principales

#### 1️⃣ **Visualiser et éditer les données**

```
1. Cliquer sur base de données (ex: myapp_db)
2. Cliquer sur table (ex: users)
3. Onglet "Browse" → Voir les données
4. Cliquer "Edit" (crayon) → Modifier une ligne
5. Cliquer "Delete" (X) → Supprimer
6. Cliquer "Insert" → Ajouter ligne
```

#### 2️⃣ **Exécuter des requêtes SQL**

```
1. Onglet "SQL" en haut
2. Taper requête :

SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id
ORDER BY order_count DESC;

3. Cliquer "Go"
4. Résultats s'affichent en bas
```

#### 3️⃣ **Créer une table**

```
1. Sélectionner base de données
2. Onglet "Structure"
3. "Create table"
4. Nom : products
5. Nombre colonnes : 4
6. Cliquer "Go"
7. Définir chaque colonne :
   - id : INT, AUTO_INCREMENT, Primary
   - name : VARCHAR(100), NOT NULL
   - price : DECIMAL(10,2)
   - stock : INT, DEFAULT 0
8. Cliquer "Save"
```

#### 4️⃣ **Importer une base de données**

```
1. Sélectionner base de données
2. Onglet "Import"
3. "Choose File" → Sélectionner fichier .sql
4. Format : SQL
5. Cliquer "Go"
6. Attendre l'import
```

**Limites d'upload** :
- Par défaut : 2-8 MB (dépend config PHP)
- Pour fichiers plus gros : Augmenter `upload_max_filesize` dans php.ini

#### 5️⃣ **Exporter une base de données**

```
1. Sélectionner base de données
2. Onglet "Export"
3. Méthode :
   - Quick : Exporte tout en SQL
   - Custom : Options avancées (tables spécifiques, format, etc.)
4. Format : SQL (ou CSV, JSON, XML, etc.)
5. Cliquer "Go"
6. Fichier se télécharge
```

#### 6️⃣ **Gérer les utilisateurs**

```
1. Cliquer "User accounts" en haut
2. "Add user account"
3. Configurer :
   - User name : myapp_user
   - Host name : localhost (ou % pour distant)
   - Password : [choisir fort]
4. "Database for user account" :
   - ☑ Grant all privileges on database "myapp_db"
5. Cliquer "Go"
```

### 💡 Astuces phpMyAdmin

**1. Recherche dans toute la base**
```
Onglet "Search" → Rechercher une valeur dans toutes les tables
```

**2. Designer (diagramme ER)**
```
Onglet "Designer" → Vue graphique des relations entre tables
```

**3. Opérations sur tables**
```
Onglet "Operations" :
- Renommer table
- Copier/Déplacer table
- Optimiser table
- Réparer table
```

**4. Bookmark de requêtes**
```
Après exécution requête → "Bookmark this SQL query"
→ Réutiliser plus tard
```

---

## 5. Autres outils utiles

### 🔧 Outils en ligne de commande

#### mariadb-dump (sauvegarde)

```bash
# Sauvegarder une base
mariadb-dump -u root -p myapp_db > myapp_backup.sql

# Sauvegarder avec structure + données
mariadb-dump -u root -p --databases myapp_db production_db > backup.sql

# Sauvegarder tout le serveur
mariadb-dump -u root -p --all-databases > full_backup.sql

# Compresser la sauvegarde
mariadb-dump -u root -p myapp_db | gzip > myapp_backup.sql.gz
```

#### mariadb-backup (Mariabackup)

**Backup complet physique** :
```bash
# Installer si besoin
sudo apt install mariadb-backup

# Créer backup
sudo mariadb-backup --backup \
  --target-dir=/var/backups/mariadb/ \
  --user=root --password

# Préparer le backup (pour restauration)
sudo mariadb-backup --prepare \
  --target-dir=/var/backups/mariadb/
```

### 📊 Outils de monitoring

#### 1️⃣ **MariaDB Monitor (command-line)**

```bash
# Surveiller en temps réel
watch -n 1 'mariadb -u root -p -e "SHOW PROCESSLIST; SHOW STATUS LIKE \"Threads%\""'
```

#### 2️⃣ **mytop**

```bash
# Installer
sudo apt install mytop

# Lancer (comme 'top' pour MariaDB)
mytop -u root -p

# Affiche en temps réel :
# - Requêtes en cours
# - Threads connectés
# - QPS (queries per second)
# - Slow queries
```

#### 3️⃣ **Prometheus + Grafana**

**Stack de monitoring professionnel** :
- **mysqld_exporter** : Exporte métriques MariaDB
- **Prometheus** : Collecte et stocke
- **Grafana** : Visualisation dashboards

### 🛠️ Outils de développement

#### Adminer (Alternative légère à phpMyAdmin)

```bash
# Fichier PHP unique !
wget https://www.adminer.org/latest.php -O adminer.php

# Placer dans /var/www/html/
sudo mv adminer.php /var/www/html/

# Accéder : http://localhost/adminer.php
```

**Avantages** :
- 1 seul fichier PHP (500 KB vs ~10 MB phpMyAdmin)
- Support multi-DB (MySQL, PostgreSQL, SQLite, etc.)
- Interface moderne

#### TablePlus (GUI moderne)

**Mac/Windows/Linux** : https://tableplus.com/

**Points forts** :
- Interface ultra-moderne
- Multi-DB
- Version gratuite disponible
- Très rapide

---

## 6. Choisir le bon outil selon le contexte

### 🎯 Matrice de décision

| Situation | Outil recommandé | Raison |
|-----------|------------------|--------|
| **Serveur distant SSH** | CLI | Seul disponible |
| **Script automatisation** | CLI | Scriptable |
| **Admin rapide Windows** | HeidiSQL | Léger, simple |
| **Dev multi-DB** | DBeaver | Universel |
| **Hébergement web** | phpMyAdmin | Déjà installé |
| **Diagrams ER** | DBeaver | Meilleur outil |
| **Débutant absolu** | HeidiSQL ou phpMyAdmin | Interface simple |
| **Pro JetBrains** | DataGrip | Intégration IDE |
| **Mac moderne** | TablePlus | Interface native |

### 🔄 Workflow typique développeur

```
┌─────────────────────────────────────────────────────────┐
│         Workflow quotidien développeur                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Développement local                                 │
│     └──► GUI (DBeaver/HeidiSQL)                         │
│          - Créer tables, modifier structure             │
│          - Tester requêtes complexes                    │
│          - Visualiser diagrammes                        │
│                                                         │
│  2. Scripts automatisation                              │
│     └──► CLI (mariadb, mariadb-dump)                    │
│          - Backup quotidien                             │
│          - Import données de test                       │
│          - CI/CD pipelines                              │
│                                                         │
│  3. Administration serveur                              │
│     └──► CLI via SSH                                    │
│          - Monitoring                                   │
│          - Résolution problèmes                         │
│          - Maintenance                                  │
│                                                         │
│  4. Partage avec équipe non-technique                   │
│     └──► phpMyAdmin ou TablePlus                        │
│          - Accès web simple                             │
│          - Pas d'installation requise                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 7. Comparaison détaillée

### 📊 Tableau comparatif complet

| Critère | CLI | HeidiSQL | DBeaver | phpMyAdmin |
|---------|-----|----------|---------|------------|
| **Installation** | ✅ Incluse | 🟡 ~10 MB | 🟡 ~100 MB | 🟡 Serveur web requis |
| **Plateformes** | Toutes | Windows (+Wine) | Toutes | Web (toutes) |
| **Courbe apprentissage** | 🟡 Moyenne | ✅ Facile | 🟡 Moyenne | ✅ Facile |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Scriptable** | ✅ Oui | ❌ Non | 🟡 Limité | ❌ Non |
| **Autocomplétion SQL** | 🟡 Basique | ✅ Bonne | ⭐⭐⭐⭐⭐ | ✅ Bonne |
| **Diagrammes ER** | ❌ Non | 🟡 Basique | ⭐⭐⭐⭐⭐ | ✅ Oui |
| **Export/Import** | ✅ Puissant | ✅ Complet | ⭐⭐⭐⭐⭐ | ✅ Complet |
| **Multi-DB support** | MariaDB/MySQL | Multi | 80+ DB | MySQL/MariaDB |
| **Administration système** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Développement** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Prix** | 🆓 Gratuit | 🆓 Gratuit | 🆓 CE / 💰 Pro | 🆓 Gratuit |

### 🎓 Recommandations par niveau

**Débutant** :
1. **HeidiSQL** (Windows) ou **phpMyAdmin** (web)
2. Apprendre **CLI** progressivement

**Intermédiaire** :
1. **DBeaver** (feature-rich)
2. **CLI** pour scripts
3. **HeidiSQL/phpMyAdmin** pour tâches rapides

**Expert / SysAdmin** :
1. **CLI** (principal)
2. **DBeaver** pour visualisations complexes
3. **Monitoring tools** (mytop, Prometheus)

---

## ✅ Points clés à retenir

- 🖥️ **CLI (mariadb)** : Essentiel à maîtriser, disponible partout, scriptable
- 🪟 **HeidiSQL** : Meilleur pour Windows, léger, simple
- 🌐 **DBeaver** : Universel, riche, multi-DB, diagrammes ER excellents
- 🌍 **phpMyAdmin** : Classique web, idéal hébergement, aucune installation client
- 🎯 **Choisir selon contexte** : Serveur distant → CLI, Dev → GUI
- 📊 **Diagrammes ER** : DBeaver > phpMyAdmin > HeidiSQL
- 🔄 **Export/Import** : Tous très bons, formats multiples
- 💾 **Backup** : mariadb-dump (logique), mariadb-backup (physique)
- 📈 **Monitoring** : mytop, Prometheus+Grafana pour production
- 🎓 **Apprentissage** : Commencer GUI, apprendre CLI progressivement
- 🔧 **Automatisation** : CLI seule option scriptable
- 👥 **Multi-utilisateur** : phpMyAdmin ou TablePlus
- ⚡ **Performance** : CLI > HeidiSQL > DBeaver > phpMyAdmin

---

## 🔗 Ressources et références

### 📥 Téléchargements
- [HeidiSQL](https://www.heidisql.com/download.php)
- [DBeaver Community](https://dbeaver.io/download/)
- [phpMyAdmin](https://www.phpmyadmin.net/downloads/)
- [TablePlus](https://tableplus.com/)
- [DataGrip](https://www.jetbrains.com/datagrip/)
- [Adminer](https://www.adminer.org/)

### 📖 Documentation
- [MariaDB CLI Reference](https://mariadb.com/kb/en/mysql-command-line-client/)
- [mariadb-dump](https://mariadb.com/kb/en/mariadb-dump/)
- [mariadb-backup](https://mariadb.com/kb/en/mariadb-backup/)
- [HeidiSQL Documentation](https://www.heidisql.com/help.php)
- [DBeaver Wiki](https://github.com/dbeaver/dbeaver/wiki)
- [phpMyAdmin Documentation](https://docs.phpmyadmin.net/)

### 🎥 Tutoriels
- [MariaDB YouTube Channel](https://www.youtube.com/user/mariadbserver)
- [DBeaver Tutorials](https://dbeaver.com/docs/wiki/)

---

## 🎓 Conclusion du Chapitre 1

**Félicitations ! 🎉**

Vous avez terminé le **Chapitre 1 : Introduction et Fondamentaux** de la formation MariaDB.

### 📚 Ce que vous avez appris

Vous maîtrisez maintenant :

1. ✅ **Ce qu'est MariaDB** : SGBDR open source, fork MySQL
2. ✅ **L'histoire** : De MySQL à MariaDB, différences clés
3. ✅ **Les cas d'usage** : Web, IA, analytics, entreprise
4. ✅ **L'architecture** : Couches, moteurs, InnoDB
5. ✅ **Les versions** : LTS vs Rolling, cycle 3 ans
6. ✅ **La roadmap** : Série 12.x, innovations futures
7. ✅ **L'installation** : Linux, Windows, macOS, Docker
8. ✅ **Les outils** : CLI, HeidiSQL, DBeaver, phpMyAdmin

### 🎯 Vous êtes prêt pour

**Chapitre 2 : SQL de base**
- Langage SQL
- Requêtes SELECT
- Insertion, modification, suppression
- Jointures
- Agrégations

**Et bien plus !**

### 💪 Exercice pratique final

Pour consolider vos acquis :

```sql
-- 1. Créer une base de données
CREATE DATABASE bookstore CHARACTER SET utf8mb4;
USE bookstore;

-- 2. Créer des tables
CREATE TABLE authors (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    country VARCHAR(50)
);

CREATE TABLE books (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(200) NOT NULL,
    author_id INT,
    price DECIMAL(10,2),
    published_year INT,
    FOREIGN KEY (author_id) REFERENCES authors(id)
);

-- 3. Insérer des données
INSERT INTO authors (name, country) VALUES
    ('J.K. Rowling', 'UK'),
    ('George Orwell', 'UK'),
    ('Haruki Murakami', 'Japan');

INSERT INTO books (title, author_id, price, published_year) VALUES
    ('Harry Potter', 1, 19.99, 1997),
    ('1984', 2, 14.99, 1949),
    ('Norwegian Wood', 3, 16.99, 1987);

-- 4. Requêtes
SELECT * FROM books;
SELECT b.title, a.name, b.price
FROM books b
JOIN authors a ON b.author_id = a.id;

-- 5. Exporter avec mariadb-dump
-- (En ligne de commande)
mariadb-dump -u root -p bookstore > bookstore_backup.sql
```

**Essayez avec vos outils** :
- CLI pour créer
- HeidiSQL/DBeaver pour visualiser
- phpMyAdmin pour explorer

---

## ➡️ Suite de la formation

**Chapitre 2 : SQL de base et requêtes**

Dans le prochain chapitre, vous allez apprendre :
- Le langage SQL en profondeur
- Les requêtes SELECT avancées
- Les jointures (INNER, LEFT, RIGHT, CROSS)
- Les agrégations (COUNT, SUM, AVG, GROUP BY)
- Les sous-requêtes
- Et bien plus !

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.9*
*Licence : CC BY-NC-SA 4.0*

**🎊 Bravo pour avoir terminé le Chapitre 1 ! 🎊**

⏭️ [Bases du SQL](/02-bases-du-sql/README.md)

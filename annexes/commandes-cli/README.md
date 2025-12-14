🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B - Commandes mariadb CLI Essentielles

> **Niveau** : Tous niveaux (Référence)  
> **Durée estimée** : Consultation à la demande  
> **Type** : Guide de référence rapide

---

## 📖 Introduction

Cette annexe est un **guide de référence rapide** des commandes essentielles du client CLI MariaDB (`mariadb` / `mysql`). Elle couvre les opérations quotidiennes d'administration, de navigation et de gestion de données via la ligne de commande.

### 🎯 Objectifs de ce guide

- **Maîtriser** les commandes de base du client mariadb
- **Accélérer** les tâches d'administration courantes
- **Faciliter** la navigation et l'exploration de bases de données
- **Optimiser** le workflow en ligne de commande
- **Référencer** les commandes système et méta-commandes

---

## 🔧 Le Client MariaDB CLI

### Qu'est-ce que le CLI MariaDB ?

Le **client en ligne de commande MariaDB** est l'interface textuelle interactive pour se connecter et interagir avec un serveur MariaDB. Il permet d'exécuter des requêtes SQL, d'administrer le serveur et de gérer les données.

### Nomenclature

MariaDB fournit deux noms pour le même client :

```bash
mariadb    # Nom moderne recommandé (depuis MariaDB 10.5)
mysql      # Nom legacy pour compatibilité
```

💡 **Recommandation** : Utiliser `mariadb` pour clarté, mais `mysql` reste fonctionnel.

### Installation

Le client est généralement inclus avec le serveur MariaDB :

```bash
# Vérifier l'installation
mariadb --version
# ou
mysql --version

# Installation séparée (si nécessaire)
# Debian/Ubuntu
sudo apt install mariadb-client

# RedHat/CentOS/Rocky
sudo yum install mariadb

# macOS (Homebrew)
brew install mariadb
```

---

## 📋 Organisation de l'Annexe

Cette annexe est organisée en **trois sections** couvrant les aspects essentiels du CLI :

### B.1 - Connexion et Navigation
Commandes pour :
- Se connecter au serveur
- Naviguer entre bases de données
- Lister et explorer la structure
- Afficher les informations de base

**Commandes clés** : `\s`, `\u`, `USE`, `SHOW DATABASES`, `SHOW TABLES`, `DESCRIBE`

### B.2 - Informations Système
Commandes pour :
- Obtenir le statut du serveur
- Surveiller les processus actifs
- Examiner les moteurs de stockage
- Consulter les variables et statuts

**Commandes clés** : `STATUS`, `SHOW PROCESSLIST`, `SHOW ENGINE`, `SHOW VARIABLES`, `SHOW STATUS`

### B.3 - Export/Import
Commandes pour :
- Exporter des données et schémas
- Importer des fichiers SQL
- Sauvegarder et restaurer
- Enregistrer l'historique des commandes

**Commandes clés** : `SOURCE`, `TEE`, `mariadb-dump`, `LOAD DATA`, `SELECT INTO OUTFILE`

---

## 💻 Syntaxe de Base

### Structure d'une Commande

```bash
mariadb [OPTIONS] [database]
```

### Options Principales

| Option | Description | Exemple |
|--------|-------------|---------|
| `-h, --host` | Hôte du serveur | `mariadb -h localhost` |
| `-P, --port` | Port du serveur | `mariadb -P 3306` |
| `-u, --user` | Nom d'utilisateur | `mariadb -u root` |
| `-p, --password` | Demande mot de passe | `mariadb -u root -p` |
| `-D, --database` | Base de données | `mariadb -D mydb` |
| `-e, --execute` | Exécute commande et quitte | `mariadb -e "SHOW DATABASES"` |
| `-s, --silent` | Mode silencieux (pas de tableau) | `mariadb -s -e "SELECT 1"` |
| `-t, --table` | Force affichage tableau | `mariadb -t` |
| `-v, --verbose` | Mode verbeux | `mariadb -v` |
| `-A, --no-auto-rehash` | Désactive auto-complétion (plus rapide) | `mariadb -A` |
| `--ssl` | Active SSL/TLS | `mariadb --ssl` |
| `--ssl-ca` | Certificat CA SSL | `mariadb --ssl-ca=/path/to/ca.pem` |

### Exemples de Connexion

```bash
# Connexion locale en tant que root
mariadb -u root -p

# Connexion à une base spécifique
mariadb -u user -p mydb

# Connexion à un serveur distant
mariadb -h 192.168.1.100 -P 3306 -u user -p

# Connexion avec SSL/TLS
mariadb -h db.example.com -u user -p --ssl

# Exécution d'une commande unique
mariadb -u root -p -e "SELECT VERSION();"

# Exécution d'un fichier SQL
mariadb -u root -p mydb < script.sql
```

---

## 🎨 Types de Commandes

Le CLI MariaDB supporte **trois types de commandes** :

### 1. Commandes SQL Standard

Requêtes SQL normales terminées par `;` ou `\g`

```sql
SELECT * FROM users;
SHOW DATABASES;
CREATE TABLE test (id INT);
```

### 2. Méta-commandes (\)

Commandes spéciales du client commençant par `\`

```sql
\s          -- Affiche le statut
\u mydb     -- Change de base de données
\q          -- Quitte le client
\! ls -l    -- Exécute commande shell (Linux/macOS)
```

### 3. Commandes SHOW

Commandes d'information et d'introspection

```sql
SHOW DATABASES;
SHOW TABLES;
SHOW CREATE TABLE users;
SHOW PROCESSLIST;
SHOW VARIABLES;
```

---

## 📝 Conventions et Symboles

### Prompt du CLI

Le prompt change selon le contexte :

```
mariadb>         -- Prêt pour nouvelle commande
    ->          -- Continuation de commande multi-ligne
    '>          -- À l'intérieur d'une chaîne simple quote
    ">          -- À l'intérieur d'une chaîne double quote
    `>          -- À l'intérieur d'un identifiant backtick
    /*>         -- À l'intérieur d'un commentaire /* */
```

### Terminaisons de Commande

| Terminaison | Description | Exemple |
|-------------|-------------|---------|
| `;` ou `\g` | Exécute la commande | `SELECT * FROM users;` |
| `\G` | Affiche résultat verticalement | `SHOW VARIABLES\G` |
| `\c` | Annule la commande courante | `SELECT * FROM\c` |
| `\q` | Quitte le client | `\q` |

### Affichage Vertical vs Horizontal

```sql
-- Affichage horizontal (défaut)
SELECT * FROM users;
+----+-------+-------------------+
| id | name  | email             |
+----+-------+-------------------+
|  1 | Alice | alice@example.com |
+----+-------+-------------------+

-- Affichage vertical (\G)
SELECT * FROM users\G
*************************** 1. row ***************************
   id: 1
 name: Alice
email: alice@example.com
```

💡 **Astuce** : `\G` est très utile pour tables avec nombreuses colonnes.

---

## 🔑 Méta-commandes Essentielles

### Navigation et Information

| Commande | Description | Équivalent SQL |
|----------|-------------|----------------|
| `\s` | Affiche statut connexion | `STATUS;` |
| `\u dbname` | Change de base | `USE dbname;` |
| `\d` | Change délimiteur | N/A |
| `\c` | Annule commande | N/A |
| `\q` ou `exit` | Quitte le client | N/A |
| `\h` ou `help` | Aide | `HELP;` |
| `\?` | Liste méta-commandes | N/A |

### Exécution et Sortie

| Commande | Description |
|----------|-------------|
| `\. filename` ou `source filename` | Exécute fichier SQL |
| `\T filename` ou `tee filename` | Enregistre sortie dans fichier |
| `\t` ou `notee` | Arrête enregistrement |
| `\! command` | Exécute commande shell (Linux/macOS) |
| `\# comment` | Commentaire |
| `\g` | Envoie commande au serveur |
| `\G` | Envoie et affiche verticalement |

### Exemple d'Utilisation

```sql
-- Vérifier le statut
\s

-- Changer de base
\u production

-- Afficher les tables
SHOW TABLES;

-- Exécuter un script
\. /path/to/script.sql

-- Enregistrer la sortie
\T /tmp/output.log
SELECT * FROM large_table;
\t  -- Arrêter enregistrement
```

---

## 🎯 Bonnes Pratiques

### Sécurité

✅ **À FAIRE**
- Toujours utiliser `-p` sans spécifier le mot de passe sur la ligne de commande
- Utiliser SSL/TLS pour connexions distantes (`--ssl`)
- Limiter les privilèges utilisateurs au strict nécessaire
- Utiliser fichiers de configuration sécurisés (`~/.my.cnf`)

❌ **À ÉVITER**
```bash
# MAUVAIS : Mot de passe visible dans historique
mariadb -u root -pMotDePasse

# BON : Le mot de passe sera demandé de manière sécurisée
mariadb -u root -p
```

### Performance

```bash
# Désactiver auto-complétion pour grandes bases
mariadb -A -u user -p

# Mode batch pour scripts (pas de formatage)
mariadb -B -u user -p < script.sql

# Limiter l'affichage pour grandes tables
mariadb -u user -p -e "SELECT * FROM huge_table LIMIT 10"
```

### Productivité

```bash
# Exécution rapide de commandes
mariadb -u root -p -e "SHOW PROCESSLIST"

# Pipeline avec autres outils
mariadb -u root -p -B -N -e "SELECT email FROM users" | grep "@example.com"

# Formatage personnalisé
mariadb -u root -p --table -e "SELECT * FROM users"
```

---

## 📚 Configuration du Client

### Fichier de Configuration Personnel

Créer `~/.my.cnf` pour options par défaut :

```ini
[client]
user = myuser
password = secret
host = localhost
port = 3306

# Sécurité
ssl-ca = /path/to/ca.pem

# Interface
prompt = '\u@\h [\d]> '
pager = less -S
```

⚠️ **Sécurité** : Protéger le fichier
```bash
chmod 600 ~/.my.cnf
```

### Options d'Affichage

```ini
[client]
# Afficher toujours en tableau
table

# Pagination automatique
pager = less -S

# Prompt personnalisé
prompt = '\u@\h [\d]> '

# Historique
histfile = ~/.mariadb_history
```

### Prompt Personnalisé

Options de personnalisation du prompt :

| Variable | Description | Exemple |
|----------|-------------|---------|
| `\u` | Utilisateur | `root` |
| `\h` | Hostname | `localhost` |
| `\d` | Base courante | `mydb` |
| `\D` | Date complète | `Mon Jan 15 14:30:00 2025` |
| `\t` | Heure courante | `14:30:00` |
| `\n` | Nouvelle ligne | |
| `\v` | Version serveur | `11.8.0-MariaDB` |

```sql
-- Définir prompt en session
prompt \u@\h [\d]>\_

-- Résultat :
root@localhost [mydb]> _
```

---

## 🔍 Conseils de Dépannage

### Problèmes de Connexion

```bash
# Erreur : Can't connect to MySQL server
# Vérifier que le serveur est démarré
sudo systemctl status mariadb

# Vérifier le port
netstat -tlnp | grep 3306

# Tester avec télnet
telnet localhost 3306

# Vérifier les permissions firewall
sudo ufw status
```

### Problèmes d'Authentification

```bash
# Erreur : Access denied
# Vérifier utilisateur et privilèges
mariadb -u root -p -e "SELECT User, Host FROM mysql.user"

# Réinitialiser mot de passe root
sudo mariadb
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nouveau_password';
FLUSH PRIVILEGES;
```

### Problèmes de Performance

```bash
# Client lent au démarrage
# Désactiver auto-rehash
mariadb -A

# Limiter l'affichage
mariadb --max_allowed_packet=64M

# Mode batch pour grandes requêtes
mariadb -B < large_query.sql > output.txt
```

---

## 💡 Astuces Avancées

### Historique des Commandes

```bash
# Localisation historique
~/.mariadb_history
~/.mysql_history

# Rechercher dans l'historique (Ctrl+R dans terminal)
# Naviguer historique (flèches haut/bas)

# Désactiver historique (session temporaire)
mariadb --histignore="*PASSWORD*"
```

### Auto-complétion

```bash
# Active par défaut
# Complétion tables : TAB
# Complétion colonnes : TAB après SELECT

# Désactiver (plus rapide sur grandes bases)
mariadb -A

# Recharger complétion en session
rehash;
```

### Édition Multi-ligne

```sql
-- Commande SQL répartie sur plusieurs lignes
SELECT
  id,
  name,
  email
FROM
  users
WHERE
  status = 'active'
ORDER BY
  name;
```

### Exécution Conditionnelle

```bash
# Exécuter seulement si commande précédente réussit
mariadb -u root -p -e "CREATE DATABASE test" && \
mariadb -u root -p test < schema.sql
```

---

## 📊 Comparaison CLI vs GUI

| Aspect | CLI | GUI (HeidiSQL, DBeaver) |
|--------|-----|-------------------------|
| **Vitesse** | ⚡ Très rapide | 🐌 Plus lent |
| **Automatisation** | ✅ Excellent (scripts) | ❌ Limité |
| **Visualisation** | ⚠️ Textuel | ✅ Graphique, intuitif |
| **Courbe d'apprentissage** | 📈 Moyenne/élevée | 📉 Faible |
| **Ressources** | 💚 Minimales | 💛 Modérées à élevées |
| **Accès distant** | ✅ SSH facile | ⚠️ Nécessite config |
| **Édition données** | ⚠️ Manuel (SQL) | ✅ Interface visuelle |

💡 **Recommandation** : Maîtriser le CLI pour administration et automatisation, utiliser GUI pour exploration et édition visuelle.

---

## ✅ Checklist de Maîtrise du CLI

### Niveau Débutant
- [ ] Se connecter au serveur local
- [ ] Lister les bases de données
- [ ] Changer de base de données
- [ ] Lister les tables
- [ ] Exécuter une requête SELECT simple
- [ ] Quitter le client proprement

### Niveau Intermédiaire
- [ ] Se connecter à un serveur distant
- [ ] Utiliser les méta-commandes (\s, \u, \G)
- [ ] Exécuter un fichier SQL avec SOURCE
- [ ] Sauvegarder la sortie avec TEE
- [ ] Utiliser SHOW pour informations système
- [ ] Configurer ~/.my.cnf

### Niveau Avancé
- [ ] Automatiser avec scripts bash + mariadb
- [ ] Utiliser options SSL/TLS
- [ ] Personnaliser le prompt
- [ ] Utiliser mariadb en mode batch
- [ ] Combiner avec outils Unix (grep, awk, sed)
- [ ] Monitorer avec SHOW PROCESSLIST
- [ ] Gérer les exports/imports complexes

---

## 🎓 Ressources Complémentaires

### Documentation Officielle
- [MariaDB Client Documentation](https://mariadb.com/kb/en/mysql-command-line-client/)
- [mariadb Command Options](https://mariadb.com/kb/en/mariadb-command-line-client/)
- [Meta-Commands Reference](https://mariadb.com/kb/en/mysql-commands/)

### Tutoriels et Guides
- [MariaDB Administration Guide](https://mariadb.com/kb/en/getting-started-with-mariadb-administration/)
- [Shell Scripting with MariaDB](https://mariadb.com/kb/en/scripting-mariadb/)

### Outils Complémentaires
- **mycli** : Client CLI moderne avec auto-complétion avancée
- **pt-query-digest** : Analyse slow query log (Percona Toolkit)
- **mytop** : Monitoring temps réel type `top`

---

## 🔗 Sections de l'Annexe

### → [B.1 Connexion et Navigation](01-connexion-navigation.md)
Commandes de base pour se connecter, naviguer et explorer : `\s`, `\u`, `USE`, `SHOW DATABASES`, `SHOW TABLES`, `DESCRIBE`

### → [B.2 Informations Système](02-informations-systeme.md)
Commandes de monitoring et diagnostic : `STATUS`, `SHOW PROCESSLIST`, `SHOW ENGINE`, `SHOW VARIABLES`, `SHOW STATUS`

### → [B.3 Export/Import](03-export-import.md)
Commandes pour sauvegarder et restaurer : `SOURCE`, `TEE`, `mariadb-dump`, `LOAD DATA`, `SELECT INTO OUTFILE`

---

## 💬 Aide et Support

### Obtenir de l'Aide

```sql
-- Aide générale
help;
\h;

-- Aide sur commande SQL spécifique
help SELECT;
help CREATE TABLE;

-- Liste des méta-commandes
\?
```

### Ressources d'Aide

- **Documentation intégrée** : `help <command>`
- **Man pages** : `man mariadb`, `man mariadb-dump`
- **Forums** : [MariaDB Community](https://mariadb.org/community/)
- **Stack Overflow** : Tag `mariadb`

---

## 🎯 Objectifs d'Apprentissage

À l'issue de cette annexe, vous serez capable de :

✅ Vous connecter à n'importe quel serveur MariaDB  
✅ Naviguer efficacement entre bases et tables  
✅ Obtenir des informations système critiques  
✅ Exporter et importer des données  
✅ Automatiser des tâches avec des scripts  
✅ Personnaliser votre environnement CLI  
✅ Diagnostiquer et résoudre les problèmes courants  
✅ Travailler efficacement en production

---

**MariaDB** : 11.8 LTS

---

## ➡️ Section Suivante

**[B.1 Connexion et Navigation →](./01-connexion-navigation.md)**  
Découvrez les commandes essentielles pour vous connecter et naviguer dans vos bases de données

---

*💡 Astuce : Gardez cette référence à portée de main lors de vos sessions CLI pour une consultation rapide !*

⏭️ [Connexion et navigation (\s, \u, SHOW DATABASES, SHOW TABLES)](/annexes/commandes-cli/01-connexion-navigation.md)

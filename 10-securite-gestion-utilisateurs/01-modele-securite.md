🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.1 Modèle de sécurité MariaDB

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Connaissance des bases de données relationnelles, SQL

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture interne du système de sécurité MariaDB
- **Maîtriser** les tables système de privilèges (`mysql.user`, `mysql.db`, etc.)
- **Analyser** le processus d'authentification et d'autorisation étape par étape
- **Identifier** les différents niveaux de privilèges et leur hiérarchie
- **Diagnostiquer** les problèmes d'accès en examinant les tables de privilèges
- **Appliquer** les meilleures pratiques de sécurisation du serveur

---

## Introduction

Le modèle de sécurité de MariaDB repose sur un système sophistiqué à deux phases : **authentification** (qui êtes-vous ?) et **autorisation** (que pouvez-vous faire ?). Contrairement à de nombreux SGBD, MariaDB utilise un modèle unique combinant l'utilisateur ET l'hôte source pour identifier un compte.

Cette architecture, héritée de MySQL mais considérablement améliorée dans MariaDB, offre une granularité exceptionnelle tout en restant performante, même avec des milliers d'utilisateurs.

### Pourquoi comprendre le modèle de sécurité ?

En tant qu'administrateur, une compréhension profonde du modèle de sécurité vous permet de :

1. **Diagnostiquer rapidement** les problèmes d'accès refusé
2. **Concevoir des architectures sécurisées** dès le départ
3. **Optimiser les performances** en évitant les configurations inefficaces
4. **Automatiser** la gestion des utilisateurs dans des environnements DevOps
5. **Auditer** les privilèges existants et détecter les failles

---

## Architecture globale du système de sécurité

Le système de sécurité MariaDB est organisé en plusieurs couches successives :

```
┌─────────────────────────────────────────────────────────────────┐
│  1. CONNEXION CLIENT                                            │
│     ↓                                                           │
│  Tentative de connexion: mariadb -u user -p -h host             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. FILTRAGE RÉSEAU (bind-address, firewall)                    │
│     ↓                                                           │
│  ✓ IP autorisée ?                                               │
│  ✓ Port accessible ?                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. PHASE D'AUTHENTIFICATION                                    │
│     ↓                                                           │
│  Table: mysql.user                                              │
│  • Correspondance 'user'@'host'                                 │
│  • Vérification du mot de passe via plugin                      │
│  • Chiffrement SSL/TLS (🆕 par défaut en 11.8)                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. PHASE D'AUTORISATION (Privilèges)                           │
│     ↓                                                           │
│  Hiérarchie des tables de privilèges:                           │
│  1. mysql.user       → Privilèges globaux                       │
│  2. mysql.db         → Privilèges par base de données           │
│  3. mysql.tables_priv → Privilèges par table                    │
│  4. mysql.columns_priv → Privilèges par colonne                 │
│  5. mysql.procs_priv  → Privilèges sur routines                 │
│  6. mysql.proxies_priv → Privilèges de proxy                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. EXÉCUTION DE LA REQUÊTE                                     │
│     ↓                                                           │
│  • Vérification des privilèges pour chaque opération            │
│  • Cache des privilèges (performance)                           │
│  • Audit logging (si activé)                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Les tables système de privilèges

MariaDB stocke toutes les informations de sécurité dans la base de données système `mysql`. Voici les tables principales :

### Table `mysql.user` - Le cœur du système

Cette table contient **tous les comptes utilisateurs** et leurs privilèges globaux.

**Structure essentielle** :

```sql
-- Examiner la structure de mysql.user
DESCRIBE mysql.user;
```

**Colonnes clés** :

| Colonne | Type | Description |
|---------|------|-------------|
| `Host` | char(255) | Hôte source autorisé (supporte wildcards) |
| `User` | char(128) | Nom d'utilisateur |
| `plugin` | char(64) | Plugin d'authentification (mysql_native_password, ed25519, pam, etc.) |
| `authentication_string` | text | Hash du mot de passe ou données d'auth |
| `ssl_type` | enum | Type SSL requis (NONE, ANY, X509, SPECIFIED) |
| `ssl_cipher` | blob | Cipher SSL spécifique |
| `x509_issuer` | blob | Émetteur du certificat X.509 |
| `x509_subject` | blob | Sujet du certificat X.509 |
| `max_questions` | int | Limite de requêtes par heure |
| `max_updates` | int | Limite de mises à jour par heure |
| `max_connections` | int | Limite de connexions par heure |
| `max_user_connections` | int | Connexions simultanées max |
| `Select_priv` | enum('N','Y') | Privilège global SELECT |
| `Insert_priv` | enum('N','Y') | Privilège global INSERT |
| `Update_priv` | enum('N','Y') | Privilège global UPDATE |
| ... | ... | *33 colonnes de privilèges au total* |

**Exemple de requête** :

```sql
-- Voir tous les utilisateurs et leurs hôtes
SELECT User, Host, plugin, ssl_type
FROM mysql.user
ORDER BY User, Host;

-- Exemple de résultat
/*
+----------------+---------------+-------------------------+----------+
| User           | Host          | plugin                  | ssl_type |
+----------------+---------------+-------------------------+----------+
| admin          | 10.0.%        | ed25519                 | ANY      |
| app_user       | 192.168.1.10  | mysql_native_password   |          |
| backup_agent   | localhost     | ed25519                 |          |
| root           | localhost     | mysql_native_password   |          |
+----------------+---------------+-------------------------+----------+
*/
```

💡 **Conseil** : Ne jamais modifier directement `mysql.user` en production. Utiliser les commandes `CREATE USER`, `GRANT`, `REVOKE` qui gèrent automatiquement la cohérence.

### Table `mysql.db` - Privilèges par base de données

Contient les privilèges spécifiques à une base de données pour chaque utilisateur.

```sql
DESCRIBE mysql.db;

-- Colonnes principales
/*
Host        - Hôte source
Db          - Nom de la base (supporte wildcards: 'test\_%')
User        - Nom d'utilisateur
Select_priv - Privilège SELECT sur cette DB
Insert_priv - Privilège INSERT sur cette DB
... (22 privilèges par DB)
*/
```

**Exemple** :

```sql
-- Voir les privilèges par base de données
SELECT Host, Db, User, Select_priv, Insert_priv, Update_priv, Delete_priv
FROM mysql.db
WHERE User = 'app_user';

/*
+---------------+--------------+----------+-------------+-------------+-------------+-------------+
| Host          | Db           | User     | Select_priv | Insert_priv | Update_priv | Delete_priv |
+---------------+--------------+----------+-------------+-------------+-------------+-------------+
| 192.168.1.10  | production   | app_user | Y           | Y           | Y           | Y           |
| 192.168.1.10  | analytics    | app_user | Y           | N           | N           | N           |
+---------------+--------------+----------+-------------+-------------+-------------+-------------+
*/
```

### Table `mysql.tables_priv` - Privilèges par table

Granularité au niveau table. Utilisée pour des permissions spécifiques à certaines tables.

```sql
DESCRIBE mysql.tables_priv;

-- Colonnes clés
/*
Host        - Hôte source
Db          - Base de données
User        - Utilisateur
Table_name  - Nom de la table
Grantor     - Qui a accordé le privilège
Timestamp   - Date de création
Table_priv  - Privilèges table (SELECT, INSERT, UPDATE, DELETE, etc.)
Column_priv - Privilèges colonnes (si applicable)
*/
```

**Exemple** :

```sql
SELECT Host, Db, User, Table_name, Table_priv
FROM mysql.tables_priv
WHERE User = 'analyst';

/*
+-----------+-------------+----------+-------------+------------------+
| Host      | Db          | User     | Table_name  | Table_priv       |
+-----------+-------------+----------+-------------+------------------+
| 10.0.%.%  | production  | analyst  | orders      | Select           |
| 10.0.%.%  | production  | analyst  | customers   | Select           |
+-----------+-------------+----------+-------------+------------------+
*/
```

### Table `mysql.columns_priv` - Privilèges par colonne

Granularité maximale : privilèges sur des colonnes spécifiques.

```sql
DESCRIBE mysql.columns_priv;

-- Exemple d'utilisation
SELECT Host, Db, User, Table_name, Column_name, Column_priv
FROM mysql.columns_priv
WHERE User = 'auditor';

/*
+-----------+----------+----------+-------------+-------------+-------------+
| Host      | Db       | User     | Table_name  | Column_name | Column_priv |
+-----------+----------+----------+-------------+-------------+-------------+
| localhost | hr       | auditor  | employees   | salary      | Select      |
| localhost | hr       | auditor  | employees   | ssn         | Select      |
+-----------+----------+----------+-------------+-------------+-------------+
*/
```

⚠️ **Attention** : Les privilèges par colonne ont un **impact performance**. Préférer les vues si besoin de masquer des colonnes sensibles.

### Table `mysql.procs_priv` - Privilèges sur les routines

Gère les privilèges sur les procédures stockées et fonctions.

```sql
-- Privilèges sur les stored procedures/functions
SELECT Host, Db, User, Routine_name, Routine_type, Proc_priv
FROM mysql.procs_priv;

/*
+-----------+-------------+----------+-------------------+--------------+-----------------+
| Host      | Db          | User     | Routine_name      | Routine_type | Proc_priv       |
+-----------+-------------+----------+-------------------+--------------+-----------------+
| localhost | production  | app_user | calculate_total   | FUNCTION     | Execute         |
| localhost | production  | app_user | process_order     | PROCEDURE    | Execute         |
+-----------+-------------+----------+-------------------+--------------+-----------------+
*/
```

### Table `mysql.proxies_priv` - Privilèges de proxy

Permet à un utilisateur de se connecter en tant qu'un autre (delegation).

```sql
DESCRIBE mysql.proxies_priv;

-- Exemple: utilisateur 'proxy_app' peut se connecter comme 'app_user'
SELECT Host, User, Proxied_host, Proxied_user, With_grant
FROM mysql.proxies_priv;
```

### 🆕 Table `mysql.global_priv` (MariaDB 10.4+)

MariaDB 10.4 a introduit `mysql.global_priv` qui remplace progressivement `mysql.user` pour le stockage des informations d'authentification au format JSON.

```sql
-- Structure moderne avec JSON
SELECT Host, User, JSON_DETAILED(Priv)
FROM mysql.global_priv
LIMIT 1\G

/*
*************************** 1. row ***************************
                  Host: localhost
                  User: root
JSON_DETAILED(Priv): {
    "access": 18446744073709551615,
    "plugin": "mysql_native_password",
    "authentication_string": "*81F5E21E35407D884A6CD4A731AEBFB6AF209E1B",
    "ssl_cipher": "",
    "x509_issuer": "",
    "x509_subject": "",
    "max_questions": 0,
    "max_updates": 0,
    "max_connections": 0,
    "max_user_connections": 0,
    "max_statement_time": 0.000000
}
*/
```

💡 **Note** : `mysql.user` existe toujours comme **vue** pour la compatibilité, mais les données réelles sont dans `mysql.global_priv`.

---

## Processus d'authentification en détail

### Étape 1 : Correspondance User@Host

Lorsqu'un client tente de se connecter, MariaDB recherche une correspondance dans `mysql.user` (ou `mysql.global_priv`).

**Règles de correspondance** :

1. **Correspondance exacte prioritaire** : `'alice'@'192.168.1.100'` avant `'alice'@'%'`
2. **Wildcards supportés** :
   - `%` : N'importe quel caractère (0 à n)
   - `_` : Un caractère unique
3. **Ordre d'évaluation** : Le plus spécifique d'abord

**Algorithme de sélection** :

```sql
-- MariaDB trie les entrées mysql.user par:
-- 1. Host: Plus spécifique en premier (moins de wildcards)
-- 2. User: Plus spécifique en premier

-- Exemple de priorité:
SELECT Host, User FROM mysql.user ORDER BY Host DESC, User DESC;

/*
Priorité (du plus au moins spécifique):
1. 'app'@'192.168.1.100'        (IP exacte)
2. 'app'@'192.168.1.%'          (subnet)
3. 'app'@'%.example.com'        (domaine)
4. 'app'@'%'                    (partout)
5. ''@'192.168.1.100'           (utilisateur anonyme, IP exacte)
6. ''@'%'                       (utilisateur anonyme, partout)
*/
```

**Exemple concret** :

```sql
-- Configuration:
CREATE USER 'app'@'192.168.1.100' IDENTIFIED BY 'password1';
CREATE USER 'app'@'192.168.1.%' IDENTIFIED BY 'password2';
CREATE USER 'app'@'%' IDENTIFIED BY 'password3';

-- Connexion depuis 192.168.1.100:
-- → Utilisera 'app'@'192.168.1.100' avec 'password1' (plus spécifique)

-- Connexion depuis 192.168.1.50:
-- → Utilisera 'app'@'192.168.1.%' avec 'password2'

-- Connexion depuis 10.0.0.1:
-- → Utilisera 'app'@'%' avec 'password3'
```

💡 **Conseil production** : Éviter les doublons user@host avec wildcards. Privilégier la spécificité maximale.

### Étape 2 : Vérification du mot de passe

Une fois le compte identifié, MariaDB vérifie le mot de passe via le **plugin d'authentification** configuré.

**Plugins disponibles** :

| Plugin | Algorithme | Sécurité | Performance | Cas d'usage |
|--------|-----------|----------|-------------|-------------|
| `mysql_native_password` | SHA1 | 🟡 Moyenne | Rapide | Legacy, compatibilité |
| `ed25519` | EdDSA | 🟢 Haute | Très rapide | **Recommandé 2025** |
| `pam` | PAM modules | 🟢 Haute | Moyenne | SSO, LDAP, 2FA |
| `unix_socket` | UID/GID Unix | 🟢 Haute | Très rapide | Connexions locales |
| `gssapi` | Kerberos | 🟢 Haute | Moyenne | Active Directory |
| `🆕 parsec` | HSM | 🟢 Très haute | Moyenne | Compliance PCI/FIPS |

**Processus de vérification** :

```
Client envoie: username, password (hash), host
        ↓
MariaDB identifie: 'user'@'host' dans mysql.global_priv
        ↓
Lit: plugin = 'ed25519'
        ↓
Charge le plugin ed25519
        ↓
Plugin vérifie: password_hash == authentication_string
        ↓
Résultat: SUCCESS ou ACCESS DENIED
```

**Exemple avec ed25519** :

```sql
-- Création avec ed25519
CREATE USER 'secure_app'@'10.0.0.%'
  IDENTIFIED VIA ed25519 USING PASSWORD('StrongP@ssw0rd2025!');

-- Vérification du plugin utilisé
SELECT User, Host, plugin, authentication_string
FROM mysql.user
WHERE User = 'secure_app';

/*
+------------+-----------+---------+---------------------------------------------+
| User       | Host      | plugin  | authentication_string                       |
+------------+-----------+---------+---------------------------------------------+
| secure_app | 10.0.0.%  | ed25519 | ZGLm1RQ4JsZuZGLzYmE5OWY4ZmExNzNkODk4ZmMx... |
+------------+-----------+---------+---------------------------------------------+
*/
```

### Étape 3 : Exigences SSL/TLS

Si l'utilisateur a des exigences SSL/TLS, MariaDB les vérifie après l'authentification.

**Niveaux d'exigence SSL** :

```sql
-- Aucune exigence SSL (connexion en clair autorisée)
CREATE USER 'insecure'@'localhost'
  IDENTIFIED BY 'password';

-- SSL requis (n'importe quel certificat)
CREATE USER 'ssl_any'@'%'
  IDENTIFIED BY 'password'
  REQUIRE SSL;

-- Certificat X.509 requis
CREATE USER 'ssl_x509'@'%'
  IDENTIFIED BY 'password'
  REQUIRE X509;

-- Certificat avec émetteur spécifique
CREATE USER 'ssl_issuer'@'%'
  IDENTIFIED BY 'password'
  REQUIRE ISSUER '/C=US/O=Example/CN=CA';

-- Cipher spécifique (sécurité renforcée)
CREATE USER 'ssl_cipher'@'%'
  IDENTIFIED BY 'password'
  REQUIRE CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';
```

🆕 **MariaDB 11.8** : TLS est activé **par défaut** si des certificats sont présents dans le datadir.

```bash
# Vérification du statut SSL/TLS
mariadb -u root -p -e "SHOW VARIABLES LIKE 'have_ssl';"
# Résultat: have_ssl = YES (par défaut en 11.8)

# Vérifier les certificats auto-générés
ls -la /var/lib/mysql/*.pem
# server-cert.pem, server-key.pem, ca-cert.pem (générés automatiquement)
```

---

## Processus d'autorisation (Privilèges)

Une fois authentifié, l'utilisateur doit avoir les **privilèges** nécessaires pour exécuter des requêtes.

### Hiérarchie des privilèges

MariaDB évalue les privilèges dans un ordre spécifique :

```
1. PRIVILÈGES GLOBAUX (mysql.user / mysql.global_priv)
   ↓ Si non accordés globalement
2. PRIVILÈGES BASE DE DONNÉES (mysql.db)
   ↓ Si non accordés au niveau DB
3. PRIVILÈGES TABLE (mysql.tables_priv)
   ↓ Si non accordés au niveau table
4. PRIVILÈGES COLONNE (mysql.columns_priv)
   ↓ Si non accordés au niveau colonne
5. PRIVILÈGES ROUTINE (mysql.procs_priv)
```

**Règle d'or** : Dès qu'un privilège est trouvé à **n'importe quel niveau**, il est accordé. L'évaluation s'arrête.

### Exemple de résolution de privilèges

**Scénario** : L'utilisateur `'analyst'@'%'` tente d'exécuter `SELECT * FROM sales.orders;`

**Étape 1** : Vérifier `mysql.global_priv`

```sql
SELECT JSON_EXTRACT(Priv, '$.access') & (1 << 0) AS has_global_select
FROM mysql.global_priv
WHERE User = 'analyst' AND Host = '%';

-- Résultat: 0 (pas de SELECT global)
```

**Étape 2** : Vérifier `mysql.db`

```sql
SELECT Select_priv
FROM mysql.db
WHERE User = 'analyst' AND Db = 'sales';

-- Résultat: 'Y' → Privilège accordé ✓
```

**Conclusion** : L'utilisateur peut lire toutes les tables de `sales`.

### Privilèges globaux vs locaux

**Privilèges globaux** (dans `mysql.user`) :

```sql
-- Accorder SELECT sur TOUTES les bases de données
GRANT SELECT ON *.* TO 'readonly_admin'@'localhost';

-- Vérifie dans mysql.user
SELECT User, Host, Select_priv, Insert_priv
FROM mysql.user
WHERE User = 'readonly_admin';
/*
+-----------------+-----------+-------------+-------------+
| User            | Host      | Select_priv | Insert_priv |
+-----------------+-----------+-------------+-------------+
| readonly_admin  | localhost | Y           | N           |
+-----------------+-----------+-------------+-------------+
*/
```

**Privilèges par base** (dans `mysql.db`) :

```sql
-- Accorder INSERT uniquement sur la base 'logs'
GRANT INSERT ON logs.* TO 'log_writer'@'%';

-- Vérifie dans mysql.db
SELECT User, Db, Insert_priv, Update_priv
FROM mysql.db
WHERE User = 'log_writer';
/*
+------------+------+-------------+-------------+
| User       | Db   | Insert_priv | Update_priv |
+------------+------+-------------+-------------+
| log_writer | logs | Y           | N           |
+------------+------+-------------+-------------+
*/
```

### Types de privilèges

MariaDB distingue plusieurs catégories de privilèges :

#### 1. Privilèges de données (DML)

| Privilège | Description | Exemple |
|-----------|-------------|---------|
| `SELECT` | Lire les données | `SELECT * FROM table` |
| `INSERT` | Insérer des données | `INSERT INTO table VALUES (...)` |
| `UPDATE` | Modifier des données | `UPDATE table SET col = val` |
| `DELETE` | Supprimer des données | `DELETE FROM table WHERE ...` |

#### 2. Privilèges de structure (DDL)

| Privilège | Description | Exemple |
|-----------|-------------|---------|
| `CREATE` | Créer bases/tables | `CREATE TABLE ...` |
| `ALTER` | Modifier structure | `ALTER TABLE ... ADD COLUMN` |
| `DROP` | Supprimer objets | `DROP TABLE ...` |
| `INDEX` | Gérer les index | `CREATE INDEX ...` |
| `CREATE VIEW` | Créer des vues | `CREATE VIEW ...` |

#### 3. Privilèges d'administration

| Privilège | Description | Cas d'usage |
|-----------|-------------|-------------|
| `RELOAD` | Recharger configs | `FLUSH PRIVILEGES;` |
| `SHUTDOWN` | Arrêter le serveur | Maintenance |
| `PROCESS` | Voir les processus | `SHOW PROCESSLIST;` |
| `SUPER` | Opérations admin | Multiples (⚠️ puissant) |
| `REPLICATION SLAVE` | Réplication | Setup replica |
| `REPLICATION CLIENT` | Voir l'état réplication | Monitoring |

#### 4. 🆕 Privilèges granulaires (MariaDB 11.8)

MariaDB 11.8 découpe `SUPER` en privilèges plus fins :

| Privilège | Description | Remplace |
|-----------|-------------|----------|
| `BINLOG_ADMIN` | Administrer binlogs | Partie de SUPER |
| `BINLOG_REPLAY` | Rejouer binlogs (PITR) | Partie de SUPER |
| `CONNECTION_ADMIN` | Gérer connexions | Partie de SUPER |
| `SHOW_ROUTINE` | Voir routines sans EXECUTE | Nouveau |
| `BINLOG_MONITOR` | Lire les binlogs | Partie de SUPER |

**Exemple d'utilisation** :

```sql
-- Avant 11.8: Fallait donner SUPER (trop permissif)
GRANT SUPER ON *.* TO 'backup_user'@'localhost';

-- Depuis 11.8: Privilège précis
GRANT BINLOG_REPLAY ON *.* TO 'backup_user'@'localhost';
-- L'utilisateur peut rejouer les binlogs pour PITR, mais pas d'autres opérations SUPER
```

---

## Système de correspondance Host

Le système de correspondance d'hôtes de MariaDB est puissant mais peut être source de confusion.

### Wildcards et patterns

**Wildcards supportés** :

| Pattern | Signification | Exemple | Correspond à |
|---------|---------------|---------|--------------|
| `%` | N'importe quelle chaîne | `'%.example.com'` | `web1.example.com`, `db.example.com` |
| `_` | Un seul caractère | `'192.168.1._'` | `192.168.1.1` à `192.168.1.9` |
| IP/CIDR | ❌ Non supporté directement | - | Utiliser % à la place |

⚠️ **Attention** : MariaDB ne supporte **pas nativement** la notation CIDR (`192.168.1.0/24`). Il faut utiliser `%`.

**Exemples de patterns** :

```sql
-- Localhost uniquement
CREATE USER 'admin'@'localhost';

-- IP spécifique
CREATE USER 'app1'@'192.168.1.100';

-- Sous-réseau (wildcards)
CREATE USER 'app_cluster'@'192.168.1.%';  -- 192.168.1.0-255
CREATE USER 'app_cluster'@'10.0.%.%';     -- 10.0.0.0-255.255

-- Domaine
CREATE USER 'external'@'%.partner.com';

-- N'importe où (DANGEREUX en production)
CREATE USER 'test'@'%';
```

### Résolution DNS inverse

Par défaut, MariaDB effectue une **résolution DNS inverse** pour les connexions.

**Problème potentiel** :

```sql
-- Utilisateur configuré avec hostname
CREATE USER 'app'@'web-server.example.com' IDENTIFIED BY 'password';

-- Connexion depuis IP 192.168.1.100
-- MariaDB fait un reverse DNS: 192.168.1.100 → web-server.example.com
-- Si le DNS reverse échoue → Accès refusé
```

**Solution** : Désactiver le DNS reverse (recommandé en production)

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
skip-name-resolve
```

💡 **Impact** : Avec `skip-name-resolve`, seules les **adresses IP** sont utilisées pour les correspondances. Les hostnames ne fonctionneront plus.

```sql
-- Avec skip-name-resolve, ceci NE FONCTIONNERA PAS:
CREATE USER 'app'@'web-server.example.com';

-- Utiliser l'IP à la place:
CREATE USER 'app'@'192.168.1.100';
```

### Ordre de priorité complexe

Lorsque plusieurs entrées correspondent, MariaDB choisit selon ces règles :

1. **Host le plus spécifique**
2. **User le plus spécifique**
3. **Ordre lexicographique** en cas d'égalité

**Exemple complexe** :

```sql
-- Configuration
CREATE USER 'alice'@'192.168.1.100' IDENTIFIED BY 'pwd1';
CREATE USER 'alice'@'192.168.1.%' IDENTIFIED BY 'pwd2';
CREATE USER 'alice'@'%' IDENTIFIED BY 'pwd3';
CREATE USER ''@'192.168.1.100' IDENTIFIED BY 'pwd4';  -- Utilisateur anonyme

-- Connexion: alice depuis 192.168.1.100
-- Ordre de priorité:
-- 1. 'alice'@'192.168.1.100'  ← SÉLECTIONNÉ (match exact)
-- 2. 'alice'@'192.168.1.%'
-- 3. 'alice'@'%'
-- 4. ''@'192.168.1.100'
```

### Diagnostic des problèmes de connexion

**Commande utile pour diagnostiquer** :

```sql
-- Voir toutes les entrées qui pourraient matcher
SELECT User, Host, plugin, ssl_type
FROM mysql.user
ORDER BY
  -- Trier par spécificité (approximation)
  LENGTH(Host) - LENGTH(REPLACE(Host, '%', '')) ASC,  -- Moins de % = plus spécifique
  Host DESC,
  User DESC;
```

**Exemple de debugging** :

```bash
# Tentative de connexion échoue
mariadb -u app -h 192.168.1.50 -p
# ERROR 1045 (28000): Access denied for user 'app'@'192.168.1.50'

# Diagnostic côté serveur
mariadb -u root -p

# Vérifier les utilisateurs existants
SELECT User, Host FROM mysql.user WHERE User = 'app';
/*
+------+---------------+
| User | Host          |
+------+---------------+
| app  | 192.168.1.100 |  ← Ne matche pas 192.168.1.50
+------+---------------+
*/

# Solution: Ajouter le host manquant
CREATE USER 'app'@'192.168.1.%' IDENTIFIED BY 'password';
```

---

## Cache des privilèges et rechargement

Pour des raisons de **performance**, MariaDB met en cache les privilèges après la connexion.

### Comment fonctionne le cache

```
Connexion établie
    ↓
MariaDB lit les tables de privilèges (mysql.*)
    ↓
Privilèges chargés en MÉMOIRE (cache)
    ↓
Durée de la session: Cache utilisé pour toutes les requêtes
    ↓
Modifications de privilèges (GRANT/REVOKE) → Cache invalidé
```

### Quand le cache est-il mis à jour ?

1. **Automatiquement** : Lors des commandes `GRANT`, `REVOKE`, `CREATE USER`, etc.
2. **Manuellement** : Avec `FLUSH PRIVILEGES`

**Cas où `FLUSH PRIVILEGES` est nécessaire** :

```sql
-- Modification DIRECTE des tables système (DÉCONSEILLÉ)
UPDATE mysql.user SET Select_priv = 'Y' WHERE User = 'app';
-- ⚠️ Le cache n'est PAS mis à jour automatiquement

-- Solution: Forcer le rechargement
FLUSH PRIVILEGES;
```

💡 **Bonne pratique** : **Toujours** utiliser `GRANT`/`REVOKE` au lieu de modifier `mysql.user` directement. `FLUSH PRIVILEGES` ne sera pas nécessaire.

### Impact performance du cache

**Benchmark approximatif** :

| Opération | Avec cache | Sans cache (après FLUSH) |
|-----------|------------|--------------------------|
| SELECT (privilèges vérifiés) | 0.001 ms | 0.050 ms |
| Connexion initiale | 5 ms | 5 ms |

Le cache peut accélérer les vérifications de privilèges jusqu'à **50x**.

---

## Meilleures pratiques du modèle de sécurité

### 1. Principe du moindre privilège appliqué

```sql
-- ❌ Mauvais: Privilèges trop larges
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- ✅ Bon: Privilèges minimaux
GRANT SELECT, INSERT, UPDATE, DELETE ON production.orders TO 'app_user'@'app_server_ip';
GRANT SELECT ON production.products TO 'app_user'@'app_server_ip';
GRANT EXECUTE ON PROCEDURE production.calculate_shipping TO 'app_user'@'app_server_ip';
```

### 2. Éviter les utilisateurs anonymes

```sql
-- Vérifier la présence d'utilisateurs anonymes
SELECT User, Host FROM mysql.user WHERE User = '';

-- Supprimer les utilisateurs anonymes
DROP USER ''@'localhost';
DROP USER ''@'%';
```

### 3. Limiter les privilèges globaux

Les privilèges globaux (`GRANT ... ON *.*`) doivent être **exceptionnels**.

```sql
-- Privilèges globaux UNIQUEMENT pour les DBA
GRANT ALL ON *.* TO 'dba_admin'@'localhost' WITH GRANT OPTION;

-- Tous les autres: privilèges par base
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user'@'app_server';
```

### 4. Séparer les utilisateurs par environnement

```sql
-- Développement
CREATE USER 'dev_alice'@'dev_network_ip' IDENTIFIED BY 'dev_password';
GRANT ALL ON dev_database.* TO 'dev_alice'@'dev_network_ip';

-- Staging
CREATE USER 'staging_app'@'staging_server_ip' IDENTIFIED VIA ed25519 USING PASSWORD('staging_pass');
GRANT SELECT, INSERT, UPDATE, DELETE ON staging_database.* TO 'staging_app'@'staging_server_ip';

-- Production (utilisateur DIFFÉRENT, hôte DIFFÉRENT)
CREATE USER 'prod_app'@'prod_server_ip' IDENTIFIED VIA ed25519 USING PASSWORD('prod_strong_pass');
GRANT SELECT, INSERT, UPDATE, DELETE ON prod_database.* TO 'prod_app'@'prod_server_ip';
-- Pas de DROP, ALTER, CREATE en production pour l'app
```

### 5. Utiliser skip-name-resolve

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
skip-name-resolve
```

**Avantages** :
- ✅ Pas de latence DNS
- ✅ Pas de risque de DNS poisoning
- ✅ Comportement prévisible

**Inconvénient** :
- ⚠️ Obligation d'utiliser des IPs (pas de hostnames)

### 6. Auditer régulièrement les privilèges

```sql
-- Script d'audit mensuel
-- 1. Utilisateurs avec ALL PRIVILEGES global
SELECT User, Host
FROM mysql.user
WHERE
  Select_priv = 'Y' AND
  Insert_priv = 'Y' AND
  Update_priv = 'Y' AND
  Delete_priv = 'Y' AND
  Create_priv = 'Y' AND
  Drop_priv = 'Y';

-- 2. Utilisateurs avec accès depuis '%'
SELECT User, Host
FROM mysql.user
WHERE Host = '%';

-- 3. Utilisateurs avec SUPER
SELECT User, Host
FROM mysql.user
WHERE Super_priv = 'Y';

-- 4. Utilisateurs avec GRANT OPTION
SELECT User, Host
FROM mysql.user
WHERE Grant_priv = 'Y';
```

### 7. Documenter les utilisateurs

Utiliser des commentaires dans un système de gestion de configuration (GitOps) :

```sql
-- users.sql (versionné dans Git)
-- Application backend (Python Flask)
-- Owner: team-backend@example.com
-- Created: 2025-01-15
-- Last review: 2025-12-01
CREATE USER IF NOT EXISTS 'backend_api'@'10.0.1.%'
  IDENTIFIED VIA ed25519 USING PASSWORD('{{ vault_backend_password }}');
GRANT SELECT, INSERT, UPDATE ON production.orders TO 'backend_api'@'10.0.1.%';
GRANT SELECT ON production.products TO 'backend_api'@'10.0.1.%';
```

---

## Vérification et diagnostic du modèle de sécurité

### Commandes essentielles

```sql
-- 1. Voir tous les utilisateurs
SELECT User, Host, plugin, password_expired, account_locked
FROM mysql.user;

-- 2. Voir les privilèges d'un utilisateur spécifique
SHOW GRANTS FOR 'app_user'@'192.168.1.10';

-- 3. Voir les privilèges de l'utilisateur actuel
SHOW GRANTS;
-- Ou
SHOW GRANTS FOR CURRENT_USER();

-- 4. Voir tous les utilisateurs avec leurs privilèges globaux
SELECT User, Host,
  CONCAT(
    IF(Select_priv = 'Y', 'SELECT,', ''),
    IF(Insert_priv = 'Y', 'INSERT,', ''),
    IF(Update_priv = 'Y', 'UPDATE,', ''),
    IF(Delete_priv = 'Y', 'DELETE,', '')
  ) AS privileges
FROM mysql.user
WHERE User != '';

-- 5. Identifier les comptes inactifs (jamais connectés)
SELECT User, Host,
  IFNULL(JSON_EXTRACT(Priv, '$.last_login'), 'Never') AS last_login
FROM mysql.global_priv;
```

### Requêtes d'audit avancées

```sql
-- Utilisateurs avec privilèges de suppression sur toutes les bases
SELECT DISTINCT u.User, u.Host
FROM mysql.user u
WHERE u.Delete_priv = 'Y'
UNION
SELECT DISTINCT d.User, d.Host
FROM mysql.db d
WHERE d.Delete_priv = 'Y';

-- Tables accessibles par un utilisateur spécifique
SELECT DISTINCT Db, Table_name, Table_priv
FROM mysql.tables_priv
WHERE User = 'analyst'
ORDER BY Db, Table_name;

-- Privilèges sur les colonnes sensibles
SELECT User, Host, Db, Table_name, Column_name, Column_priv
FROM mysql.columns_priv
WHERE Column_name IN ('password', 'ssn', 'credit_card', 'salary');
```

### Détecter les problèmes de sécurité

```sql
-- 🔴 ALERTE: Utilisateurs avec mot de passe vide
SELECT User, Host
FROM mysql.user
WHERE authentication_string = '' OR authentication_string IS NULL;

-- 🔴 ALERTE: Utilisateurs avec mysql_native_password (obsolète)
SELECT User, Host, plugin
FROM mysql.user
WHERE plugin = 'mysql_native_password';

-- 🔴 ALERTE: Utilisateurs sans SSL requis mais avec accès distant
SELECT User, Host, ssl_type
FROM mysql.user
WHERE Host != 'localhost' AND ssl_type = '';

-- 🟡 AVERTISSEMENT: Utilisateurs avec accès global (%)
SELECT User, Host
FROM mysql.user
WHERE Host = '%';
```

---

## 🆕 Évolutions MariaDB 11.8

### 1. Transition vers mysql.global_priv

`mysql.global_priv` devient la source de vérité, `mysql.user` devient une vue.

**Requête pour voir les données JSON** :

```sql
SELECT User, Host,
  JSON_PRETTY(Priv) AS privileges_json
FROM mysql.global_priv
WHERE User = 'app_user'\G

/*
*************************** 1. row ***************************
           User: app_user
           Host: 10.0.1.%
privileges_json: {
  "access": 31,
  "plugin": "ed25519",
  "authentication_string": "ZGLm1RQ...",
  "password_last_changed": 1734038400,
  "password_lifetime": 90,
  "account_locked": false,
  "is_role": false
}
*/
```

### 2. Privilèges granulaires

Découpage de `SUPER` en privilèges spécialisés :

```sql
-- Ancien modèle (pré-11.8)
GRANT SUPER ON *.* TO 'ops_user'@'localhost';
-- Problème: Trop de pouvoirs (dangereux)

-- Nouveau modèle (11.8+)
GRANT BINLOG_ADMIN ON *.* TO 'ops_user'@'localhost';     -- Admin binlogs
GRANT CONNECTION_ADMIN ON *.* TO 'support_user'@'%';     -- Gérer connexions
GRANT BINLOG_REPLAY ON *.* TO 'backup_user'@'localhost'; -- Rejouer binlogs
```

**Vérifier les nouveaux privilèges** :

```sql
SHOW PRIVILEGES;
-- Rechercher: BINLOG_ADMIN, BINLOG_REPLAY, CONNECTION_ADMIN, SHOW_ROUTINE, etc.
```

### 3. TLS par défaut

```sql
-- Vérifier si TLS est actif
SHOW VARIABLES LIKE 'have_ssl';
-- Résultat en 11.8: YES (par défaut si certificats présents)

-- Vérifier les certificats auto-générés
SHOW VARIABLES LIKE 'ssl_%';
/*
+---------------------+--------------------------------+
| Variable_name       | Value                          |
+---------------------+--------------------------------+
| ssl_ca              | /var/lib/mysql/ca.pem          |
| ssl_cert            | /var/lib/mysql/server-cert.pem |
| ssl_key             | /var/lib/mysql/server-key.pem  |
+---------------------+--------------------------------+
*/
```

### 4. Plugin PARSEC

Nouveau plugin pour authentification via Hardware Security Module (HSM).

```sql
-- Installation du plugin
INSTALL SONAME 'auth_parsec';

-- Vérification
SHOW PLUGINS WHERE Name = 'parsec';

-- Création d'utilisateur avec PARSEC
CREATE USER 'hsm_user'@'localhost'
  IDENTIFIED VIA parsec USING 'parsec://provider/key_id';

-- Vérification
SELECT User, Host, plugin
FROM mysql.user
WHERE plugin = 'parsec';
```

---

## ✅ Points clés à retenir

- **MariaDB utilise un modèle 'User'@'Host'** unique qui permet une granularité exceptionnelle
- **L'authentification et l'autorisation sont deux phases distinctes** : qui êtes-vous ? que pouvez-vous faire ?
- **Les tables système `mysql.*` stockent tous les privilèges**, avec `mysql.global_priv` comme source moderne
- **La hiérarchie des privilèges** va du global (mysql.user) au très granulaire (mysql.columns_priv)
- **Le cache des privilèges améliore les performances** : modifications automatiques avec GRANT/REVOKE
- **Les wildcards (%, _) permettent des patterns flexibles** mais attention à la spécificité
- **🆕 MariaDB 11.8 introduit des privilèges granulaires** pour remplacer SUPER
- **🆕 TLS est activé par défaut en 11.8** si des certificats sont présents
- **🆕 Le plugin PARSEC permet l'intégration HSM** pour les environnements hautement sécurisés
- **Le principe du moindre privilège est fondamental** : ne donner que les droits nécessaires

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 User Account Management](https://mariadb.com/kb/en/user-account-management/)
- [📖 mysql.user Table](https://mariadb.com/kb/en/mysqluser-table/)
- [📖 mysql.global_priv Table](https://mariadb.com/kb/en/mysqlglobal_priv-table/)
- [📖 Privilege System](https://mariadb.com/kb/en/grant/)
- [📖 Authentication Plugins](https://mariadb.com/kb/en/authentication-plugins/)
- [📖 🆕 Granular Privileges](https://mariadb.com/kb/en/grant/#table-privileges)

### Outils et scripts

- [pt-show-grants](https://docs.percona.com/percona-toolkit/pt-show-grants.html) - Percona Toolkit pour exporter les privilèges
- [mysql_secure_installation](https://mariadb.com/kb/en/mysql_secure_installation/) - Sécurisation initiale

### Articles et guides

- [CIS MariaDB Benchmark](https://www.cisecurity.org/) - Standards de sécurité
- [OWASP Database Security](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)

---

## ➡️ Section suivante

**10.2 : Création et gestion des utilisateurs (CREATE/ALTER/DROP USER)** - Vous apprendrez à créer, modifier et supprimer des utilisateurs avec toutes les options avancées : plugins d'authentification, ressources limits, account locking, password policies, etc.

---


⏭️ [Création et gestion des utilisateurs (CREATE USER, ALTER USER, DROP USER)](/10-securite-gestion-utilisateurs/02-creation-gestion-utilisateurs.md)

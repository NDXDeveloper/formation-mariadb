🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2 Création et gestion des utilisateurs (CREATE USER, ALTER USER, DROP USER)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Section 10.1 (Modèle de sécurité MariaDB)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Créer** des utilisateurs avec toutes les options avancées (CREATE USER)
- **Modifier** des utilisateurs existants (ALTER USER)
- **Supprimer** des utilisateurs proprement (DROP USER)
- **Configurer** différents plugins d'authentification (native, ed25519, PAM, etc.)
- **Implémenter** des politiques de mots de passe robustes
- **Gérer** les limites de ressources par utilisateur
- **Automatiser** la gestion des utilisateurs dans des environnements DevOps
- **Appliquer** les meilleures pratiques de production

---

## Introduction

La gestion des utilisateurs est l'une des tâches les plus critiques en administration MariaDB. Une création d'utilisateur incorrecte peut exposer votre système à des failles de sécurité majeures, tandis qu'une configuration trop restrictive peut bloquer des opérations légitimes.

MariaDB offre un ensemble de commandes SQL puissantes pour gérer le cycle de vie complet des utilisateurs :

```
CREATE USER → ALTER USER → GRANT → REVOKE → DROP USER
     ↓            ↓           ↓        ↓         ↓
  Création   Modification  Attribution Révocation Suppression
```

### Privilèges requis

Pour gérer les utilisateurs, vous devez avoir le privilège **`CREATE USER`** :

```sql
-- Vérifier si vous avez le privilège
SHOW GRANTS FOR CURRENT_USER();

-- Accorder le privilège CREATE USER
GRANT CREATE USER ON *.* TO 'admin'@'localhost';
```

💡 **Note** : Le privilège `CREATE USER` permet de créer/modifier/supprimer des utilisateurs, mais **pas** d'accorder des privilèges. Pour cela, il faut `GRANT OPTION`.

---

## CREATE USER - Création d'utilisateurs

### Syntaxe complète

```sql
CREATE [OR REPLACE] USER [IF NOT EXISTS]
  user_specification [, user_specification] ...
  [REQUIRE {NONE | tls_option [[AND] tls_option] ...}]
  [WITH resource_option [resource_option] ...]
  [password_option | lock_option] ...;

user_specification:
  'user'@'host'
  [authentication_option]

authentication_option:
  IDENTIFIED BY 'password'
  | IDENTIFIED BY PASSWORD 'hash'
  | IDENTIFIED {VIA | WITH} auth_plugin [USING 'auth_string']
  | IDENTIFIED {VIA | WITH} auth_plugin [USING 'auth_string']
      [OR IDENTIFIED {VIA | WITH} auth_plugin [USING 'auth_string']] ...
```

### Création basique

**Exemple le plus simple** :

```sql
-- Utilisateur avec mot de passe (plugin par défaut)
CREATE USER 'john'@'localhost' IDENTIFIED BY 'StrongP@ssw0rd!';

-- Vérification
SELECT User, Host, plugin FROM mysql.user WHERE User = 'john';
/*
+------+-----------+-------------------------+
| User | Host      | plugin                  |
+------+-----------+-------------------------+
| john | localhost | mysql_native_password   |
+------+-----------+-------------------------+
*/
```

⚠️ **Attention** : Par défaut, MariaDB utilise `mysql_native_password` qui est **obsolète**. Préférer `ed25519`.

### Création avec plugin d'authentification moderne

```sql
-- 🟢 RECOMMANDÉ: Utiliser ed25519
CREATE USER 'alice'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('SecurePassword2025!');

-- Vérification
SELECT User, Host, plugin, authentication_string
FROM mysql.user
WHERE User = 'alice';
/*
+-------+------+---------+------------------------------------------------+
| User  | Host | plugin  | authentication_string                          |
+-------+------+---------+------------------------------------------------+
| alice | %    | ed25519 | ZGLm1RQ4JsZuZGLzYmE5OWY4ZmExNzNkODk4ZmMx...    |
+-------+------+---------+------------------------------------------------+
*/
```

💡 **Pourquoi ed25519 ?**
- ✅ Plus sécurisé (cryptographie moderne)
- ✅ Plus rapide que SHA256
- ✅ Résistant aux attaques par force brute
- ✅ Recommandé depuis MariaDB 10.4+

### CREATE OR REPLACE

Remplace un utilisateur existant (utile pour les scripts idempotents) :

```sql
-- Si l'utilisateur existe, le remplace, sinon le crée
CREATE OR REPLACE USER 'app_user'@'10.0.0.%'
  IDENTIFIED VIA ed25519 USING PASSWORD('NewPassword!');

-- Équivalent à:
DROP USER IF EXISTS 'app_user'@'10.0.0.%';
CREATE USER 'app_user'@'10.0.0.%'
  IDENTIFIED VIA ed25519 USING PASSWORD('NewPassword!');
```

⚠️ **Attention** : `CREATE OR REPLACE` **supprime tous les privilèges** de l'utilisateur existant. Utilisez `ALTER USER` pour modifier sans perdre les privilèges.

### CREATE IF NOT EXISTS

Crée uniquement si l'utilisateur n'existe pas :

```sql
-- Ne fait rien si l'utilisateur existe déjà
CREATE USER IF NOT EXISTS 'readonly'@'%'
  IDENTIFIED BY 'password';

-- Utile dans les scripts d'initialisation
CREATE USER IF NOT EXISTS 'backup_agent'@'localhost'
  IDENTIFIED VIA ed25519 USING PASSWORD('backup_secret');
```

### Création d'utilisateurs multiples

```sql
-- Créer plusieurs utilisateurs en une seule commande
CREATE USER
  'dev_alice'@'dev_network' IDENTIFIED BY 'dev_pass1',
  'dev_bob'@'dev_network' IDENTIFIED BY 'dev_pass2',
  'dev_charlie'@'dev_network' IDENTIFIED BY 'dev_pass3';
```

---

## Plugins d'authentification

MariaDB supporte plusieurs plugins d'authentification pour s'adapter à différents contextes de sécurité.

### 1. mysql_native_password (Legacy)

**Description** : Plugin historique utilisant SHA1 (obsolète).

```sql
-- Syntaxe explicite
CREATE USER 'legacy_user'@'localhost'
  IDENTIFIED VIA mysql_native_password USING PASSWORD('password123');

-- Syntaxe raccourcie (par défaut)
CREATE USER 'legacy_user'@'localhost'
  IDENTIFIED BY 'password123';
```

**⚠️ Problèmes** :
- SHA1 est cryptographiquement faible (collisions connues)
- Vulnérable aux attaques rainbow tables
- Non recommandé pour les nouveaux déploiements

**Cas d'usage** :
- Applications legacy qui ne supportent pas ed25519
- Compatibilité avec MySQL ancien (<5.7)

### 2. ed25519 (Recommandé)

**Description** : Authentification moderne basée sur EdDSA (Elliptic Curve Digital Signature Algorithm).

```sql
-- Création avec ed25519
CREATE USER 'secure_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('VeryStrongPassword!2025');

-- Avec hash pré-calculé (pour automatisation)
CREATE USER 'automated_user'@'localhost'
  IDENTIFIED VIA ed25519 USING 'ZGLm1RQ4JsZuZGLzYmE5OWY4ZmExNzNkODk4ZmMxNzkyZGQ5';
```

**✅ Avantages** :
- Sécurité maximale (256-bit)
- Performance excellente
- Résistant aux attaques par force brute
- Standard moderne (RFC 8032)

**Cas d'usage** :
- **Tous les nouveaux déploiements** (recommandation 2025)
- Applications critiques (finance, santé)
- Environnements cloud

**Génération de hash pour automatisation** :

```bash
# Générer un hash ed25519 hors ligne
echo -n 'MyPassword123!' | openssl dgst -sha256 -binary | base64

# Ou utiliser MariaDB directement
mariadb -u root -p -e "SELECT ED25519_PASSWORD('MyPassword123!');"
```

### 3. unix_socket (Connexions locales)

**Description** : Authentification basée sur l'UID/GID Unix. Pas de mot de passe requis.

```sql
-- Création avec unix_socket
CREATE USER 'backup_script'@'localhost'
  IDENTIFIED VIA unix_socket;

-- L'utilisateur Unix 'backup_script' peut se connecter sans mot de passe
```

**Fonctionnement** :

```bash
# En tant qu'utilisateur Unix 'backup_script'
sudo -u backup_script mariadb
# → Connexion automatique sans mot de passe
```

**✅ Avantages** :
- Pas de gestion de mots de passe
- Sécurité renforcée (basée sur l'OS)
- Idéal pour les scripts locaux

**⚠️ Limitations** :
- Fonctionne **uniquement** avec `@'localhost'` via socket Unix
- Ne marche pas avec TCP/IP

**Cas d'usage** :
- Scripts de backup locaux
- Monitoring agents (Zabbix, Nagios)
- Tâches cron

### 4. pam (Pluggable Authentication Modules)

**Description** : Délègue l'authentification aux modules PAM du système (LDAP, AD, 2FA, etc.).

**Installation** :

```sql
-- Installer le plugin PAM
INSTALL SONAME 'auth_pam';

-- Vérification
SHOW PLUGINS WHERE Name = 'pam';
```

**Création d'utilisateur PAM** :

```sql
-- Utilisateur qui s'authentifie via PAM
CREATE USER 'ldap_user'@'%'
  IDENTIFIED VIA pam USING 'mariadb';
-- 'mariadb' est le nom du service PAM dans /etc/pam.d/mariadb
```

**Configuration PAM** (`/etc/pam.d/mariadb`) :

```
# Authentification via LDAP
auth    required   pam_ldap.so
account required   pam_ldap.so

# Configuration LDAP dans /etc/ldap.conf
# host ldap.example.com
# base dc=example,dc=com
```

**Cas d'usage avancé : 2FA** :

```
# /etc/pam.d/mariadb-2fa
auth    required   pam_unix.so        # Mot de passe Unix
auth    required   pam_google_authenticator.so  # Google Authenticator (TOTP)
account required   pam_unix.so
```

```sql
CREATE USER 'admin_2fa'@'%'
  IDENTIFIED VIA pam USING 'mariadb-2fa';
```

**✅ Avantages** :
- Centralisation de l'authentification (SSO)
- Support 2FA/MFA
- Intégration Active Directory / OpenLDAP
- Politiques d'authentification complexes

**Cas d'usage** :
- Entreprises avec Active Directory
- Environnements nécessitant 2FA
- Conformité (SOC2, ISO 27001)

### 5. gssapi (Kerberos)

**Description** : Authentification Kerberos pour environnements Active Directory.

**Installation** :

```sql
INSTALL SONAME 'auth_gssapi';
```

**Configuration** :

```sql
-- Utilisateur Kerberos
CREATE USER 'alice@EXAMPLE.COM'@'%'
  IDENTIFIED VIA gssapi;

-- Avec mapping de principal
CREATE USER 'alice'@'%'
  IDENTIFIED VIA gssapi AS 'alice@EXAMPLE.COM';
```

**Configuration Kerberos** (`/etc/krb5.conf`) :

```ini
[libdefaults]
    default_realm = EXAMPLE.COM

[realms]
    EXAMPLE.COM = {
        kdc = kdc.example.com
        admin_server = admin.example.com
    }
```

**Connexion client** :

```bash
# Obtenir un ticket Kerberos
kinit alice@EXAMPLE.COM

# Connexion MariaDB (sans mot de passe)
mariadb --plugin-dir=/usr/lib64/mysql/plugin
```

**✅ Avantages** :
- SSO pour Windows/Linux
- Pas de transmission de mot de passe
- Intégration native avec AD

**Cas d'usage** :
- Environnements Windows/AD
- Grandes entreprises
- Environnements hautement sécurisés

### 6. 🆕 parsec (Hardware Security Module)

**Description** : Authentification via HSM (Hardware Security Module) pour conformité maximale.

**Installation** (MariaDB 11.8+) :

```sql
INSTALL SONAME 'auth_parsec';

SHOW PLUGINS WHERE Name = 'parsec';
```

**Configuration** :

```sql
-- Utilisateur avec clé stockée dans HSM
CREATE USER 'payment_processor'@'payment_gateway'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/key_id_12345';
```

**Configuration PARSEC** (`/etc/parsec/config.toml`) :

```toml
[provider.pkcs11]
library = "/usr/lib/libpkcs11.so"
slot_number = 0

[provider.tpm]
tcti = "device:/dev/tpm0"
```

**✅ Avantages** :
- Conformité PCI-DSS, FIPS 140-2/3
- Clés cryptographiques matérielles
- Impossibilité d'exfiltration des clés
- Audit trail matériel

**Cas d'usage** :
- Secteur bancaire/financier
- Traitement de paiements (PCI-DSS)
- Gouvernement et défense
- Cloud souverain

**Comparaison des coûts** :

| HSM | Type | Prix | Certifications |
|-----|------|------|----------------|
| Thales Luna | Matériel | 10 000-50 000 $ | FIPS 140-2 Level 3 |
| AWS CloudHSM | Cloud | ~1,50 $/h | FIPS 140-2 Level 3 |
| Azure Key Vault HSM | Cloud | ~1 $/h | FIPS 140-2 Level 2 |
| YubiKey 5 FIPS | USB | ~60 $ | FIPS 140-2 Level 1 |

### Authentification multi-plugin (Fallback)

MariaDB 10.4+ supporte plusieurs plugins pour un même utilisateur (fallback) :

```sql
-- Authentification ed25519 OU PAM (fallback)
CREATE USER 'flexible_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('primary_password')
  OR IDENTIFIED VIA pam USING 'mariadb';

-- Client peut se connecter avec:
-- 1. Mot de passe ed25519 (prioritaire)
-- 2. Authentification PAM (si ed25519 échoue)
```

**Cas d'usage** :
- Migration progressive d'un plugin à un autre
- Compatibilité avec clients anciens
- Redondance d'authentification

---

## ALTER USER - Modification d'utilisateurs

`ALTER USER` permet de modifier un utilisateur existant **sans perdre ses privilèges**.

### Syntaxe

```sql
ALTER USER [IF EXISTS]
  user_specification [, user_specification] ...
  [REQUIRE {NONE | tls_option [[AND] tls_option] ...}]
  [WITH resource_option [resource_option] ...]
  [password_option | lock_option] ...;
```

### Changement de mot de passe

```sql
-- Changer le mot de passe de l'utilisateur actuel
ALTER USER USER() IDENTIFIED BY 'NewPassword123!';

-- Changer le mot de passe d'un autre utilisateur
ALTER USER 'app_user'@'10.0.0.%'
  IDENTIFIED BY 'NewStrongPassword2025!';

-- Changer avec un plugin différent
ALTER USER 'legacy_user'@'localhost'
  IDENTIFIED VIA ed25519 USING PASSWORD('MigratedPassword!');
```

💡 **Note** : `ALTER USER` conserve tous les privilèges existants, contrairement à `CREATE OR REPLACE USER`.

### Changement de plugin d'authentification

**Migration de mysql_native_password vers ed25519** :

```sql
-- Vérifier le plugin actuel
SELECT User, Host, plugin FROM mysql.user WHERE User = 'app_user';
/*
+----------+---------+-------------------------+
| User     | Host    | plugin                  |
+----------+---------+-------------------------+
| app_user | 10.0.0.%| mysql_native_password   |
+----------+---------+-------------------------+
*/

-- Migrer vers ed25519
ALTER USER 'app_user'@'10.0.0.%'
  IDENTIFIED VIA ed25519 USING PASSWORD('SameOrNewPassword!');

-- Vérification
SELECT User, Host, plugin FROM mysql.user WHERE User = 'app_user';
/*
+----------+---------+---------+
| User     | Host    | plugin  |
+----------+---------+---------+
| app_user | 10.0.0.%| ed25519 |
+----------+---------+---------+
*/
```

### Modification des exigences SSL/TLS

```sql
-- Forcer SSL pour un utilisateur existant
ALTER USER 'external_user'@'%' REQUIRE SSL;

-- Exiger un certificat X.509
ALTER USER 'api_client'@'%' REQUIRE X509;

-- Exiger un émetteur spécifique
ALTER USER 'partner_api'@'%'
  REQUIRE ISSUER '/C=US/O=Example Corp/CN=CA';

-- Exiger un cipher spécifique (TLS 1.3)
ALTER USER 'secure_app'@'%'
  REQUIRE CIPHER 'TLS_AES_256_GCM_SHA384';

-- Combiner plusieurs exigences
ALTER USER 'banking_app'@'%'
  REQUIRE SSL
  AND CIPHER 'TLS_AES_256_GCM_SHA384'
  AND ISSUER '/C=US/O=Bank/CN=Internal CA';

-- Retirer toutes les exigences SSL
ALTER USER 'local_dev'@'localhost' REQUIRE NONE;
```

🆕 **MariaDB 11.8** : TLS est activé par défaut. `REQUIRE SSL` devient implicite pour les connexions non-localhost.

### Verrouillage de compte (Account Locking)

```sql
-- Verrouiller un compte (empêche toute connexion)
ALTER USER 'departed_employee'@'%' ACCOUNT LOCK;

-- Déverrouiller un compte
ALTER USER 'returning_contractor'@'%' ACCOUNT UNLOCK;

-- Vérification
SELECT User, Host, account_locked
FROM mysql.user
WHERE User IN ('departed_employee', 'returning_contractor');
/*
+---------------------+------+----------------+
| User                | Host | account_locked |
+---------------------+------+----------------+
| departed_employee   | %    | Y              |
| returning_contractor| %    | N              |
+---------------------+------+----------------+
*/
```

**Cas d'usage** :
- Suspension temporaire d'un employé
- Compte en attente d'activation
- Investigation de sécurité

💡 **Astuce** : Préférer `ACCOUNT LOCK` à `DROP USER` pour préserver l'historique.

### Expiration de mot de passe

```sql
-- Forcer le changement de mot de passe à la prochaine connexion
ALTER USER 'new_hire'@'%' PASSWORD EXPIRE;

-- Expiration après 90 jours
ALTER USER 'contractor'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Expiration à une date spécifique
ALTER USER 'temp_user'@'%' PASSWORD EXPIRE AT '2025-12-31';

-- Jamais expirer (par défaut)
ALTER USER 'service_account'@'localhost' PASSWORD EXPIRE NEVER;

-- Utiliser la politique par défaut du serveur
ALTER USER 'standard_user'@'%' PASSWORD EXPIRE DEFAULT;
```

**Configuration serveur** (`/etc/my.cnf.d/server.cnf`) :

```ini
[mysqld]
# Expiration par défaut : 180 jours
default_password_lifetime = 180
```

**Vérification** :

```sql
SELECT User, Host,
  password_expired,
  JSON_EXTRACT(Priv, '$.password_lifetime') AS lifetime_days
FROM mysql.global_priv
WHERE User = 'contractor';
```

### Historique des mots de passe

Empêcher la réutilisation de mots de passe récents :

```sql
-- Empêcher la réutilisation des 5 derniers mots de passe
ALTER USER 'security_conscious'@'%'
  PASSWORD HISTORY 5;

-- Empêcher la réutilisation pendant 365 jours
ALTER USER 'compliance_user'@'%'
  PASSWORD REUSE INTERVAL 365 DAY;

-- Combiner les deux
ALTER USER 'banking_user'@'%'
  PASSWORD HISTORY 10
  PASSWORD REUSE INTERVAL 180 DAY;

-- Désactiver l'historique
ALTER USER 'dev_user'@'localhost'
  PASSWORD HISTORY DEFAULT;
```

**Configuration serveur** :

```ini
[mysqld]
password_history = 5
password_reuse_interval = 180
```

---

## DROP USER - Suppression d'utilisateurs

### Syntaxe

```sql
DROP USER [IF EXISTS] user [, user] ...;
```

### Suppression simple

```sql
-- Supprimer un utilisateur
DROP USER 'old_user'@'localhost';

-- Supprimer plusieurs utilisateurs
DROP USER
  'temp1'@'%',
  'temp2'@'%',
  'temp3'@'%';

-- Supprimer uniquement s'il existe (pas d'erreur si absent)
DROP USER IF EXISTS 'maybe_exists'@'localhost';
```

### Vérifications avant suppression

```sql
-- 1. Vérifier les privilèges de l'utilisateur
SHOW GRANTS FOR 'user_to_delete'@'host';

-- 2. Vérifier les objets possédés (vues, routines)
SELECT TABLE_SCHEMA, TABLE_NAME, DEFINER
FROM information_schema.VIEWS
WHERE DEFINER = 'user_to_delete@host';

SELECT ROUTINE_SCHEMA, ROUTINE_NAME, DEFINER
FROM information_schema.ROUTINES
WHERE DEFINER = 'user_to_delete@host';

-- 3. Vérifier les événements
SELECT EVENT_SCHEMA, EVENT_NAME, DEFINER
FROM information_schema.EVENTS
WHERE DEFINER = 'user_to_delete@host';

-- 4. Vérifier les triggers
SELECT TRIGGER_SCHEMA, TRIGGER_NAME, DEFINER
FROM information_schema.TRIGGERS
WHERE DEFINER = 'user_to_delete@host';
```

⚠️ **Attention** : Les vues/routines/events/triggers ne sont **pas** supprimés avec `DROP USER`. Ils deviennent orphelins.

### Suppression propre avec nettoyage

```bash
#!/bin/bash
# Script de suppression propre d'un utilisateur

USER_TO_DELETE="old_user"
HOST_TO_DELETE="localhost"

# 1. Sauvegarder les privilèges
mariadb -u root -p -e "SHOW GRANTS FOR '${USER_TO_DELETE}'@'${HOST_TO_DELETE}';" > user_grants_backup.sql

# 2. Réassigner les objets (vues, routines)
mariadb -u root -p <<EOF
-- Changer le DEFINER des vues
UPDATE mysql.proc
SET definer = 'new_owner@localhost'
WHERE definer = '${USER_TO_DELETE}@${HOST_TO_DELETE}';

-- Flush privileges pour appliquer
FLUSH PRIVILEGES;
EOF

# 3. Supprimer l'utilisateur
mariadb -u root -p -e "DROP USER '${USER_TO_DELETE}'@'${HOST_TO_DELETE}';"

echo "Utilisateur ${USER_TO_DELETE}@${HOST_TO_DELETE} supprimé"
```

### Suppression avec audit

```sql
-- Créer une table d'audit avant suppression
CREATE TABLE IF NOT EXISTS audit_deleted_users (
  deleted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  username VARCHAR(128),
  hostname VARCHAR(255),
  privileges TEXT,
  deleted_by VARCHAR(128)
);

-- Procédure de suppression auditée
DELIMITER //
CREATE OR REPLACE PROCEDURE safe_drop_user(
  IN p_user VARCHAR(128),
  IN p_host VARCHAR(255)
)
BEGIN
  DECLARE v_grants TEXT;

  -- Récupérer les privilèges actuels
  SET v_grants = (
    SELECT GROUP_CONCAT(PRIVILEGE_TYPE)
    FROM information_schema.USER_PRIVILEGES
    WHERE GRANTEE = CONCAT("'", p_user, "'@'", p_host, "'")
  );

  -- Audit avant suppression
  INSERT INTO audit_deleted_users (username, hostname, privileges, deleted_by)
  VALUES (p_user, p_host, v_grants, USER());

  -- Suppression
  SET @drop_stmt = CONCAT('DROP USER IF EXISTS ''', p_user, '''@''', p_host, '''');
  PREPARE stmt FROM @drop_stmt;
  EXECUTE stmt;
  DEALLOCATE PREPARE stmt;

  SELECT CONCAT('User ', p_user, '@', p_host, ' deleted and audited') AS result;
END//
DELIMITER ;

-- Utilisation
CALL safe_drop_user('old_user', 'localhost');
```

---

## Gestion des limites de ressources

MariaDB permet de limiter l'utilisation des ressources par utilisateur pour éviter les abus.

### Types de limites

```sql
-- Création avec limites
CREATE USER 'limited_user'@'%'
  IDENTIFIED BY 'password'
  WITH
    MAX_QUERIES_PER_HOUR 1000          -- Max 1000 requêtes/heure
    MAX_UPDATES_PER_HOUR 500           -- Max 500 UPDATE/INSERT/DELETE/heure
    MAX_CONNECTIONS_PER_HOUR 100       -- Max 100 connexions/heure
    MAX_USER_CONNECTIONS 10;           -- Max 10 connexions simultanées

-- Modification des limites
ALTER USER 'limited_user'@'%'
  WITH
    MAX_QUERIES_PER_HOUR 2000
    MAX_USER_CONNECTIONS 20;

-- Retirer toutes les limites
ALTER USER 'unlimited_user'@'%'
  WITH
    MAX_QUERIES_PER_HOUR 0
    MAX_UPDATES_PER_HOUR 0
    MAX_CONNECTIONS_PER_HOUR 0
    MAX_USER_CONNECTIONS 0;
```

### Vérification des limites

```sql
-- Voir les limites configurées
SELECT User, Host,
  max_questions AS queries_per_hour,
  max_updates AS updates_per_hour,
  max_connections AS connections_per_hour,
  max_user_connections AS max_concurrent
FROM mysql.user
WHERE User = 'limited_user';

-- Voir l'utilisation actuelle
SHOW STATUS LIKE 'Questions';
SHOW STATUS LIKE 'Com_update';
SHOW STATUS LIKE 'Connections';
```

### Cas d'usage des limites

| Type d'utilisateur | MAX_QUERIES | MAX_UPDATES | MAX_CONNECTIONS | MAX_USER_CONNECTIONS |
|--------------------|-------------|-------------|-----------------|----------------------|
| **API publique** | 10 000 | 1 000 | 500 | 50 |
| **Application web** | 100 000 | 10 000 | 1 000 | 100 |
| **Batch job** | 0 (illimité) | 50 000 | 10 | 5 |
| **User reporting** | 1 000 | 0 | 50 | 5 |
| **Service account** | 0 | 0 | 0 | 20 |

**Exemple production - API rate limiting** :

```sql
-- API tier 1 (gratuit)
CREATE USER 'api_free_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('...')
  WITH
    MAX_QUERIES_PER_HOUR 1000
    MAX_USER_CONNECTIONS 5;

-- API tier 2 (payant)
CREATE USER 'api_premium_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('...')
  WITH
    MAX_QUERIES_PER_HOUR 100000
    MAX_USER_CONNECTIONS 50;

-- API tier 3 (entreprise)
CREATE USER 'api_enterprise_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('...')
  WITH
    MAX_QUERIES_PER_HOUR 0  -- Illimité
    MAX_USER_CONNECTIONS 500;
```

---

## Politiques de mots de passe

### Validation de la complexité

MariaDB supporte des plugins de validation de mots de passe.

**Installation de cracklib** :

```sql
-- Installer le plugin
INSTALL SONAME 'cracklib_password_check';

-- Vérification
SHOW PLUGINS WHERE Name = 'cracklib_password_check';
```

**Configuration** (`/etc/my.cnf.d/server.cnf`) :

```ini
[mysqld]
# Plugin de validation
plugin-load-add = cracklib_password_check.so

# Dictionnaire cracklib
cracklib_password_check_dictionary = /usr/share/cracklib/pw_dict
```

**Test** :

```sql
-- Mot de passe faible (sera rejeté)
CREATE USER 'test'@'localhost' IDENTIFIED BY 'password';
-- ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

-- Mot de passe fort (accepté)
CREATE USER 'test'@'localhost' IDENTIFIED BY 'C0mpl3x!P@ssw0rd#2025';
-- Query OK
```

### Plugin simple_password_check

Alternative plus simple que cracklib :

```sql
INSTALL SONAME 'simple_password_check';
```

**Configuration** :

```ini
[mysqld]
plugin-load-add = simple_password_check.so

# Longueur minimale
simple_password_check_minimal_length = 12

# Caractères requis
simple_password_check_digits = 2          # Au moins 2 chiffres
simple_password_check_letters_same_case = 3  # 3 lettres même casse
simple_password_check_other_characters = 2   # 2 caractères spéciaux
```

### Politique personnalisée avec trigger

```sql
-- Trigger pour valider les mots de passe complexes
DELIMITER //
CREATE OR REPLACE TRIGGER password_policy_check
BEFORE INSERT ON mysql.user
FOR EACH ROW
BEGIN
  -- Vérifier que le mot de passe n'est pas vide
  IF NEW.authentication_string = '' OR NEW.authentication_string IS NULL THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Empty passwords are not allowed';
  END IF;

  -- Autres validations personnalisées...
END//
DELIMITER ;
```

---

## Renommage d'utilisateurs

MariaDB 10.4+ supporte `RENAME USER` :

```sql
-- Renommer un utilisateur
RENAME USER 'old_name'@'localhost' TO 'new_name'@'localhost';

-- Renommer avec changement d'hôte
RENAME USER 'user'@'old_host' TO 'user'@'new_host';

-- Renommer plusieurs utilisateurs
RENAME USER
  'user1'@'%' TO 'newuser1'@'%',
  'user2'@'%' TO 'newuser2'@'%';
```

⚠️ **Important** : `RENAME USER` préserve **tous les privilèges**, contrairement à DROP + CREATE.

---

## Automatisation et scripts

### Script Bash de création d'utilisateur

```bash
#!/bin/bash
# create_user.sh - Script de création d'utilisateur MariaDB

set -e

# Variables
DB_USER="${1}"
DB_HOST="${2}"
DB_PLUGIN="${3:-ed25519}"  # Par défaut: ed25519
STRONG_PASSWORD=$(openssl rand -base64 32)

# Validation
if [ -z "$DB_USER" ] || [ -z "$DB_HOST" ]; then
  echo "Usage: $0 <username> <host> [plugin]"
  exit 1
fi

# Création utilisateur
mariadb -u root -p <<EOF
CREATE USER IF NOT EXISTS '${DB_USER}'@'${DB_HOST}'
  IDENTIFIED VIA ${DB_PLUGIN} USING PASSWORD('${STRONG_PASSWORD}');
EOF

# Stocker le mot de passe dans Vault (HashiCorp Vault)
vault kv put secret/mariadb/${DB_USER} password="${STRONG_PASSWORD}"

echo "User ${DB_USER}@${DB_HOST} created with plugin ${DB_PLUGIN}"
echo "Password stored in Vault: secret/mariadb/${DB_USER}"
```

### Script Python avec pymysql

```python
#!/usr/bin/env python3
import pymysql
import secrets
import string

def generate_strong_password(length=32):
    """Génère un mot de passe fort"""
    alphabet = string.ascii_letters + string.digits + string.punctuation
    password = ''.join(secrets.choice(alphabet) for _ in range(length))
    return password

def create_mariadb_user(username, host, plugin='ed25519'):
    """Crée un utilisateur MariaDB"""
    connection = pymysql.connect(
        host='localhost',
        user='root',
        password='root_password',
        database='mysql',
        cursorclass=pymysql.cursors.DictCursor
    )

    try:
        with connection.cursor() as cursor:
            password = generate_strong_password()

            # Création utilisateur
            sql = f"""
            CREATE USER IF NOT EXISTS '{username}'@'{host}'
              IDENTIFIED VIA {plugin} USING PASSWORD('{password}')
            """
            cursor.execute(sql)

            # Expiration du mot de passe après 90 jours
            sql = f"ALTER USER '{username}'@'{host}' PASSWORD EXPIRE INTERVAL 90 DAY"
            cursor.execute(sql)

            connection.commit()

            print(f"User {username}@{host} created successfully")
            print(f"Password: {password}")
            print("⚠️  Store this password securely!")

    finally:
        connection.close()

# Utilisation
if __name__ == '__main__':
    create_mariadb_user('app_user', '10.0.0.%', 'ed25519')
```

### Ansible Playbook

```yaml
---
# playbook.yml - Gestion utilisateurs MariaDB avec Ansible
- name: Manage MariaDB Users
  hosts: db_servers
  become: yes

  vars:
    mariadb_users:
      - name: app_user
        host: "10.0.0.%"
        password: "{{ vault_app_password }}"
        plugin: ed25519
        priv: "production.*:SELECT,INSERT,UPDATE,DELETE"

      - name: readonly_user
        host: "%"
        password: "{{ vault_readonly_password }}"
        plugin: ed25519
        priv: "production.*:SELECT"
        require_ssl: yes

      - name: backup_agent
        host: localhost
        plugin: unix_socket
        priv: "*.*:SELECT,LOCK TABLES,RELOAD,REPLICATION CLIENT"

  tasks:
    - name: Create MariaDB users
      mysql_user:
        name: "{{ item.name }}"
        host: "{{ item.host }}"
        password: "{{ item.password | default(omit) }}"
        plugin: "{{ item.plugin }}"
        priv: "{{ item.priv }}"
        tls_requires:
          SSL: "{{ item.require_ssl | default(false) }}"
        state: present
      loop: "{{ mariadb_users }}"
      no_log: true  # Ne pas logger les mots de passe
```

### Terraform (Infrastructure as Code)

```hcl
# main.tf - Gestion utilisateurs MariaDB avec Terraform

terraform {
  required_providers {
    mysql = {
      source  = "petoju/mysql"
      version = "~> 3.0"
    }
  }
}

provider "mysql" {
  endpoint = "mariadb.example.com:3306"
  username = "terraform"
  password = var.terraform_password
}

resource "random_password" "app_user" {
  length  = 32
  special = true
}

resource "mysql_user" "app_user" {
  user               = "app_user"
  host               = "10.0.0.%"
  plaintext_password = random_password.app_user.result
  auth_plugin        = "ed25519"

  tls_option = "SSL"
}

resource "mysql_grant" "app_user_grant" {
  user       = mysql_user.app_user.user
  host       = mysql_user.app_user.host
  database   = "production"
  privileges = ["SELECT", "INSERT", "UPDATE", "DELETE"]
}

# Stocker le mot de passe dans AWS Secrets Manager
resource "aws_secretsmanager_secret" "app_user_password" {
  name = "mariadb/app_user/password"
}

resource "aws_secretsmanager_secret_version" "app_user_password" {
  secret_id     = aws_secretsmanager_secret.app_user_password.id
  secret_string = random_password.app_user.result
}
```

---

## Bonnes pratiques de production

### 1. Politique de naming cohérente

```sql
-- Convention de nommage
-- Format: {env}_{app}_{role}@{host}

-- Développement
CREATE USER 'dev_webapp_rw'@'dev_network' ...;
CREATE USER 'dev_webapp_ro'@'dev_network' ...;

-- Staging
CREATE USER 'stg_webapp_rw'@'stg_network' ...;

-- Production
CREATE USER 'prd_webapp_rw'@'prd_app_server' ...;
CREATE USER 'prd_webapp_ro'@'prd_app_server' ...;

-- Service accounts
CREATE USER 'svc_backup'@'localhost' ...;
CREATE USER 'svc_monitoring'@'monitoring_server' ...;
```

### 2. Utiliser des secrets managers

**Ne jamais** stocker les mots de passe en clair. Utiliser :

- **HashiCorp Vault**
- **AWS Secrets Manager**
- **Azure Key Vault**
- **Google Secret Manager**
- **Kubernetes Secrets**

**Exemple avec Vault** :

```bash
# Stocker
vault kv put secret/mariadb/app_user password="StrongP@ss"

# Récupérer dans un script
DB_PASSWORD=$(vault kv get -field=password secret/mariadb/app_user)
mariadb -u app_user -p"${DB_PASSWORD}"
```

### 3. Rotation automatique des mots de passe

```bash
#!/bin/bash
# rotate_password.sh - Rotation mensuelle automatisée

USER="app_user"
HOST="10.0.0.%"

# Générer nouveau mot de passe
NEW_PASSWORD=$(openssl rand -base64 32)

# Changer dans MariaDB
mariadb -u root -p <<EOF
ALTER USER '${USER}'@'${HOST}'
  IDENTIFIED VIA ed25519 USING PASSWORD('${NEW_PASSWORD}');
EOF

# Mettre à jour Vault
vault kv put secret/mariadb/${USER} password="${NEW_PASSWORD}"

# Redémarrer l'application pour charger le nouveau mot de passe
kubectl rollout restart deployment/webapp

echo "Password rotated for ${USER}@${HOST}"
```

**Cron job** :

```cron
# Rotation mensuelle le 1er du mois à 2h du matin
0 2 1 * * /opt/scripts/rotate_password.sh app_user
```

### 4. Séparation des environnements

```sql
-- Utilisateurs DIFFÉRENTS par environnement
-- Jamais les mêmes credentials en dev/staging/prod

-- Dev: Privilèges larges, mot de passe simple
CREATE USER 'dev_user'@'%'
  IDENTIFIED BY 'dev_password';
GRANT ALL ON dev_db.* TO 'dev_user'@'%';

-- Production: Privilèges minimaux, authentification forte
CREATE USER 'prod_app'@'prod_app_server_ip'
  IDENTIFIED VIA ed25519 USING PASSWORD('ComplexProdPassword!')
  REQUIRE SSL;
GRANT SELECT, INSERT, UPDATE ON prod_db.* TO 'prod_app'@'prod_app_server_ip';
-- Pas de DELETE, DROP, ALTER en production
```

### 5. Audit trail des créations/modifications

```sql
-- Table d'audit
CREATE TABLE user_management_audit (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  action_type ENUM('CREATE', 'ALTER', 'DROP', 'GRANT', 'REVOKE'),
  target_user VARCHAR(128),
  target_host VARCHAR(255),
  performed_by VARCHAR(128),
  details TEXT,
  INDEX idx_action_time (action_time),
  INDEX idx_target (target_user, target_host)
);

-- Trigger d'audit (exemple simplifié)
DELIMITER //
CREATE TRIGGER audit_user_creation
AFTER INSERT ON mysql.user
FOR EACH ROW
BEGIN
  INSERT INTO user_management_audit (action_type, target_user, target_host, performed_by, details)
  VALUES ('CREATE', NEW.User, NEW.Host, USER(), CONCAT('Plugin: ', NEW.plugin));
END//
DELIMITER ;
```

### 6. Documentation automatique

```bash
#!/bin/bash
# document_users.sh - Génère documentation des utilisateurs

mariadb -u root -p -e "
SELECT
  User,
  Host,
  plugin AS 'Auth Plugin',
  CASE
    WHEN ssl_type = 'ANY' THEN 'SSL Required'
    WHEN ssl_type = 'X509' THEN 'X509 Required'
    WHEN ssl_type = 'SPECIFIED' THEN 'SSL Configured'
    ELSE 'No SSL'
  END AS 'SSL Status',
  max_questions AS 'Max Queries/h',
  max_user_connections AS 'Max Connections',
  account_locked AS 'Locked'
FROM mysql.user
WHERE User NOT IN ('root', 'mariadb.sys')
ORDER BY User, Host
" | pandoc -f markdown -t pdf -o users_documentation.pdf

echo "Documentation generated: users_documentation.pdf"
```

---

## Troubleshooting

### Problème 1 : "Access denied" malgré la création

**Symptôme** :

```bash
mariadb -u app_user -h 10.0.0.50 -p
# ERROR 1045 (28000): Access denied for user 'app_user'@'10.0.0.50'
```

**Diagnostic** :

```sql
-- 1. Vérifier que l'utilisateur existe
SELECT User, Host FROM mysql.user WHERE User = 'app_user';

-- 2. Vérifier la correspondance d'hôte
SELECT User, Host FROM mysql.user WHERE User = 'app_user' AND Host LIKE '10.0.%';

-- 3. Vérifier skip-name-resolve
SHOW VARIABLES LIKE 'skip_name_resolve';
```

**Solutions** :

```sql
-- Si l'hôte ne correspond pas
CREATE USER 'app_user'@'10.0.0.%' IDENTIFIED BY 'password';

-- Si besoin de FLUSH PRIVILEGES (rare)
FLUSH PRIVILEGES;
```

### Problème 2 : Plugin d'authentification non supporté

**Symptôme** :

```
ERROR 2059 (HY000): Authentication plugin 'ed25519' cannot be loaded
```

**Solution côté client** :

```bash
# Installer le plugin client ed25519
sudo apt-get install mariadb-plugin-auth-ed25519  # Debian/Ubuntu
sudo yum install MariaDB-client-plugin-ed25519    # RHEL/CentOS

# Ou spécifier le chemin du plugin
mariadb --plugin-dir=/usr/lib64/mysql/plugin -u user -p
```

### Problème 3 : Mot de passe rejeté par cracklib

**Symptôme** :

```
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
```

**Solution** :

```sql
-- Générer un mot de passe fort
SELECT RANDOM_BYTES(32);  -- En hexadécimal
SELECT SHA2(UUID(), 256); -- Alternative

-- Ou utiliser un générateur externe
```

```bash
# Générateur de mot de passe fort
openssl rand -base64 32
```

### Problème 4 : Trop de connexions

**Symptôme** :

```
ERROR 1203 (42000): User 'app_user' already has more than 'max_user_connections' active connections
```

**Diagnostic** :

```sql
-- Voir les connexions actuelles
SELECT User, Host, COUNT(*) AS connections
FROM information_schema.PROCESSLIST
GROUP BY User, Host;

-- Voir la limite
SELECT User, Host, max_user_connections
FROM mysql.user
WHERE User = 'app_user';
```

**Solution** :

```sql
-- Augmenter la limite
ALTER USER 'app_user'@'%' WITH MAX_USER_CONNECTIONS 100;

-- Ou supprimer la limite
ALTER USER 'app_user'@'%' WITH MAX_USER_CONNECTIONS 0;
```

---

## ✅ Points clés à retenir

- **CREATE USER crée des comptes** avec plugin d'authentification, SSL, limites de ressources
- **ALTER USER modifie sans perdre les privilèges**, contrairement à CREATE OR REPLACE
- **DROP USER supprime les comptes** mais pas les objets possédés (vues, routines)
- **ed25519 est le plugin recommandé** pour les nouveaux déploiements (sécurité + performance)
- **PAM permet SSO et 2FA** pour intégration enterprise
- **🆕 PARSEC (11.8) supporte les HSM** pour conformité PCI-DSS/FIPS
- **Account locking suspend temporairement** un compte sans suppression
- **Password expiration force le renouvellement** régulier des credentials
- **Resource limits évitent les abus** (rate limiting, connection pooling)
- **L'automatisation avec IaC** (Terraform, Ansible) garantit la reproductibilité

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 CREATE USER](https://mariadb.com/kb/en/create-user/)
- [📖 ALTER USER](https://mariadb.com/kb/en/alter-user/)
- [📖 DROP USER](https://mariadb.com/kb/en/drop-user/)
- [📖 Authentication Plugins](https://mariadb.com/kb/en/authentication-plugins/)
- [📖 🆕 PARSEC Plugin](https://mariadb.com/kb/en/parsec-authentication-plugin/)
- [📖 Password Validation Plugins](https://mariadb.com/kb/en/password-validation/)

### Outils

- [openssl](https://www.openssl.org/) - Génération de mots de passe forts
- [HashiCorp Vault](https://www.vaultproject.io/) - Secrets management
- [Ansible mysql_user module](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_user_module.html)

---

## ➡️ Section suivante

**10.3 : Système de privilèges (GRANT/REVOKE, niveaux de privilèges)** - Vous apprendrez à accorder et révoquer des privilèges à tous les niveaux (global, base, table, colonne), avec les bonnes pratiques du principe du moindre privilège.

---


⏭️ [Système de privilèges](/10-securite-gestion-utilisateurs/03-systeme-privileges.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.10 Password validation plugins et politiques

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 10.1-10.9, connaissances en sécurité des mots de passe

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'importance des politiques de mots de passe fortes
- **Configurer** les plugins de validation MariaDB (simple_password_check, cracklib)
- **Implémenter** des règles de complexité personnalisées
- **Gérer** l'expiration et la rotation des mots de passe
- **Forcer** le changement de mot de passe périodique
- **Intégrer** les politiques système (PAM) avec MariaDB
- **Auditer** la force des mots de passe existants
- **Répondre** aux exigences réglementaires (PCI-DSS, NIST, ISO)

---

## Introduction

Les **mots de passe faibles** sont l'une des principales causes de compromission des bases de données. Même avec un système d'authentification robuste (ed25519, TLS), un mot de passe comme `password123` ou `admin` rend toute la sécurité inefficace.

### Statistiques alarmantes

```
Top 10 des mots de passe les plus utilisés (2023):
┌─────────────────────────────────────────────────────────┐
│  1. 123456                 (23% des utilisateurs)       │
│  2. password               (7%)                         │
│  3. 123456789              (5%)                         │
│  4. 12345678               (4%)                         │
│  5. qwerty                 (3%)                         │
│  6. abc123                 (2%)                         │
│  7. password1              (2%)                         │
│  8. 1234567                (1.5%)                       │
│  9. admin                  (1.5%)                       │
│ 10. letmein                (1%)                         │
└─────────────────────────────────────────────────────────┘

Temps pour craquer (brute force):
- "password"     : <1 seconde
- "Password1"    : 3 minutes
- "P@ssw0rd1"    : 8 heures
- "P@ssw0rd123!" : 2 semaines
- "aK9#mL2$pQ5&vR3!" (16 chars) : 34 000 ans
```

### Pourquoi les politiques sont critiques

**Scénario sans politique** :

```
CREATE USER 'admin'@'%' IDENTIFIED BY 'admin';
                                        ^^^^^^
                                        ⚠️ Accepté!

CREATE USER 'backup'@'%' IDENTIFIED BY '123456';
                                        ^^^^^^
                                        ⚠️ Accepté!

CREATE USER 'app'@'%' IDENTIFIED BY 'password';
                                    ^^^^^^^^
                                    ⚠️ Accepté!

→ Base de données compromise en quelques minutes
→ Brute force trivial
→ Rainbow tables
→ Dictionnaires
```

**Avec politique stricte** :

```
CREATE USER 'admin'@'%' IDENTIFIED BY 'admin';
ERROR 1819 (HY000): Password does not satisfy policy requirements
- Minimum 12 caractères
- Au moins 1 majuscule
- Au moins 1 chiffre
- Au moins 1 caractère spécial
- Pas dans dictionnaire

CREATE USER 'admin'@'%' IDENTIFIED BY 'Adm!n2025Secure#DB';
Query OK, 0 rows affected
                           ✓ Accepté!
```

### Conformité réglementaire

| Réglementation | Exigences mots de passe | Plugin MariaDB |
|----------------|-------------------------|----------------|
| **PCI-DSS 4.0** | 12 caractères min, complexité, rotation 90 jours | ✅ simple_password_check + PASSWORD EXPIRE |
| **NIST SP 800-63B** | 8 caractères min, vérifier dictionnaires | ✅ cracklib_password_check |
| **ISO 27001** | Politique selon risque, rotation | ✅ Configurable |
| **RGPD** | Protection accès, rotation recommandée | ✅ Audit + rotation |
| **HIPAA** | Complexité, rotation, unicité | ✅ All plugins |

💡 **PCI-DSS 4.0** (Requirement 8.3.6) exige :
- Minimum **12 caractères**
- Complexité (majuscules, minuscules, chiffres, spéciaux)
- Rotation tous les **90 jours**
- Pas de réutilisation des 4 derniers mots de passe

---

## Plugins de validation disponibles

MariaDB propose **2 plugins natifs** pour valider les mots de passe.

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────┐
│              Plugins de validation MariaDB               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. simple_password_check                                │
│     - Règles basiques (longueur, caractères)             │
│     - Configuration simple                               │
│     - Performance excellente                             │
│     - Recommandé pour la plupart des cas                 │
│                                                          │
│  2. cracklib_password_check                              │
│     - Vérification contre dictionnaires                  │
│     - Détection mots communs                             │
│     - Basé sur cracklib (système)                        │
│     - Plus strict mais plus lourd                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Tableau comparatif

| Fonctionnalité | simple_password_check | cracklib_password_check |
|----------------|----------------------|------------------------|
| **Longueur minimale** | ✅ Configurable (1-1000) | ✅ Via cracklib |
| **Majuscules** | ✅ Configurable (0-1000) | ❌ |
| **Minuscules** | ✅ Configurable (0-1000) | ❌ |
| **Chiffres** | ✅ Configurable (0-1000) | ❌ |
| **Caractères spéciaux** | ✅ Configurable (0-1000) | ❌ |
| **Dictionnaire** | ❌ | ✅ cracklib dictionary |
| **Performance** | 🟢 Excellente | 🟡 Bonne |
| **Complexité config** | 🟢 Simple | 🟡 Moyenne |
| **Use case** | Production standard | Sécurité maximale |

**Recommandation** : **simple_password_check** pour la plupart des déploiements (balance sécurité/performance).

---

## Plugin simple_password_check

### Installation

```sql
-- Installer le plugin
INSTALL SONAME 'simple_password_check';

-- Vérifier l'installation
SHOW PLUGINS WHERE Name LIKE '%password%';
/*
+------------------------+--------+--------------------+---------------------------+---------+
| Name                   | Status | Type               | Library                   | License |
+------------------------+--------+--------------------+---------------------------+---------+
| simple_password_check  | ACTIVE | PASSWORD VALIDATION| simple_password_check.so  | GPL     |
+------------------------+--------+--------------------+---------------------------+---------+
*/

-- Vérifier les variables
SHOW VARIABLES LIKE 'simple_password_check%';
```

### Configuration de base

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
# Charger le plugin au démarrage
plugin-load-add = simple_password_check.so

# Configuration politique de base (PCI-DSS compliant)
simple_password_check_minimal_length = 12
simple_password_check_digits = 1
simple_password_check_letters_same_case = 1
simple_password_check_other_characters = 1
```

**Paramètres disponibles** :

```sql
-- Longueur minimale (1-1000, défaut: 8)
SET GLOBAL simple_password_check_minimal_length = 12;

-- Nombre minimum de chiffres (0-1000, défaut: 1)
SET GLOBAL simple_password_check_digits = 2;

-- Nombre minimum de lettres en minuscules (0-1000, défaut: 1)
SET GLOBAL simple_password_check_letters_same_case = 2;

-- Nombre minimum de caractères spéciaux (0-1000, défaut: 1)
SET GLOBAL simple_password_check_other_characters = 2;
```

### Niveaux de sécurité recommandés

#### Niveau 1 : Standard (dev/test)

```sql
-- Configuration minimale
SET GLOBAL simple_password_check_minimal_length = 8;
SET GLOBAL simple_password_check_digits = 1;
SET GLOBAL simple_password_check_letters_same_case = 1;
SET GLOBAL simple_password_check_other_characters = 0;

-- Exemples acceptés:
-- "Password1"  ✓
-- "MyApp123"   ✓
-- "test2025"   ✓
```

#### Niveau 2 : Production (recommandé)

```sql
-- Configuration PCI-DSS
SET GLOBAL simple_password_check_minimal_length = 12;
SET GLOBAL simple_password_check_digits = 1;
SET GLOBAL simple_password_check_letters_same_case = 2;  -- 2 min + 2 maj
SET GLOBAL simple_password_check_other_characters = 1;

-- Exemples acceptés:
-- "MySecure2025!"    ✓ (13 chars, 1 chiffre, 2 min, 2 maj, 1 spécial)
-- "Db@AdminPass99"   ✓ (14 chars)
-- "Production#2025X" ✓ (16 chars)

-- Exemples refusés:
-- "password123"      ✗ (pas de majuscule, pas de spécial)
-- "Password1"        ✗ (trop court, pas de spécial)
-- "Pass@123"         ✗ (trop court)
```

#### Niveau 3 : Haute sécurité (bancaire, santé)

```sql
-- Configuration maximale
SET GLOBAL simple_password_check_minimal_length = 16;
SET GLOBAL simple_password_check_digits = 2;
SET GLOBAL simple_password_check_letters_same_case = 3;
SET GLOBAL simple_password_check_other_characters = 2;

-- Exemples acceptés:
-- "MyBankApp#2025Secure!" ✓ (21 chars)
-- "Health$System99_Prod"  ✓ (20 chars)

-- Exemples refusés:
-- "MySecure2025!"         ✗ (trop court, pas assez de spéciaux)
```

### Tests de validation

```sql
-- Test 1: Mot de passe faible (doit échouer)
CREATE USER 'test1'@'localhost' IDENTIFIED BY 'password';
-- ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

-- Test 2: Mot de passe sans chiffre (doit échouer)
CREATE USER 'test2'@'localhost' IDENTIFIED BY 'Password!';
-- ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

-- Test 3: Mot de passe conforme (doit réussir)
CREATE USER 'test3'@'localhost' IDENTIFIED BY 'MySecure2025!';
-- Query OK, 0 rows affected

-- Test 4: Modification mot de passe
ALTER USER 'test3'@'localhost' IDENTIFIED BY 'weak';
-- ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

ALTER USER 'test3'@'localhost' IDENTIFIED BY 'NewSecure2025#';
-- Query OK, 0 rows affected
```

### Configuration permanente

```ini
# /etc/my.cnf.d/server.cnf - Configuration permanente
[mysqld]
# Plugin
plugin-load-add = simple_password_check.so

# Politique PCI-DSS
simple_password_check_minimal_length = 12
simple_password_check_digits = 1
simple_password_check_letters_same_case = 2
simple_password_check_other_characters = 1

# Redémarrer MariaDB
systemctl restart mariadb
```

---

## Plugin cracklib_password_check

### Qu'est-ce que cracklib ?

**Cracklib** est une bibliothèque système qui vérifie les mots de passe contre :
- Dictionnaires de mots communs (millions d'entrées)
- Variations simples (leet speak: p@ssw0rd)
- Palindromes
- Mots trop simples

### Installation

**Étape 1 : Installer cracklib système**

```bash
# RHEL/CentOS/Rocky
sudo dnf install cracklib cracklib-dicts

# Debian/Ubuntu
sudo apt install libcrack2 cracklib-runtime

# Vérifier dictionnaire
ls -lh /usr/share/cracklib/
# pw_dict.pwd  pw_dict.pwi  pw_dict.hwm
```

**Étape 2 : Installer plugin MariaDB**

```sql
-- Installer le plugin
INSTALL SONAME 'cracklib_password_check';

-- Vérifier
SHOW PLUGINS WHERE Name = 'cracklib_password_check';
/*
+---------------------------+--------+--------------------+-----------------------------+---------+
| Name                      | Status | Type               | Library                     | License |
+---------------------------+--------+--------------------+-----------------------------+---------+
| cracklib_password_check   | ACTIVE | PASSWORD VALIDATION| cracklib_password_check.so  | GPL     |
+---------------------------+--------+--------------------+-----------------------------+---------+
*/
```

### Configuration

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
plugin-load-add = cracklib_password_check.so

# Chemin vers dictionnaire cracklib (optionnel)
# cracklib_password_check_dictionary = /usr/share/cracklib/pw_dict
```

### Tests avec cracklib

```sql
-- Test 1: Mot commun (refusé)
CREATE USER 'user1'@'localhost' IDENTIFIED BY 'password';
-- ERROR 1819 (HY000): cracklib: it is based on a dictionary word

CREATE USER 'user2'@'localhost' IDENTIFIED BY 'welcome';
-- ERROR 1819 (HY000): cracklib: it is based on a dictionary word

-- Test 2: Variation simple (refusé)
CREATE USER 'user3'@'localhost' IDENTIFIED BY 'p@ssw0rd';
-- ERROR 1819 (HY000): cracklib: it is based on a dictionary word

-- Test 3: Trop court (refusé)
CREATE USER 'user4'@'localhost' IDENTIFIED BY 'Ab1!';
-- ERROR 1819 (HY000): cracklib: it is too short

-- Test 4: Mot de passe fort (accepté)
CREATE USER 'user5'@'localhost' IDENTIFIED BY 'MyUniquePass2025!#DB';
-- Query OK, 0 rows affected
```

### Combiner simple_password_check + cracklib

```ini
# /etc/my.cnf.d/server.cnf
# Utiliser LES DEUX plugins (sécurité maximale)
[mysqld]
plugin-load-add = simple_password_check.so
plugin-load-add = cracklib_password_check.so

# Politique stricte
simple_password_check_minimal_length = 16
simple_password_check_digits = 2
simple_password_check_letters_same_case = 3
simple_password_check_other_characters = 2

# Résultat:
# - Vérifie longueur, complexité (simple_password_check)
# - Vérifie dictionnaire (cracklib_password_check)
# → Sécurité maximale!
```

---

## Expiration et rotation des mots de passe

### PASSWORD EXPIRE (MariaDB 10.4.3+)

**Forcer le changement au prochain login** :

```sql
-- Expirer immédiatement
ALTER USER 'user'@'localhost' PASSWORD EXPIRE;

-- L'utilisateur doit changer son mot de passe
-- mariadb -u user -p
-- ERROR 1820 (HY000): You must SET PASSWORD before executing this statement

-- Changer le mot de passe
SET PASSWORD = PASSWORD('NewSecurePassword2025!');
-- Query OK, 0 rows affected
```

### PASSWORD EXPIRE INTERVAL (rotation périodique)

```sql
-- Expiration tous les 90 jours (PCI-DSS)
CREATE USER 'compliance_user'@'%'
  IDENTIFIED BY 'Initial2025#Pass'
  PASSWORD EXPIRE INTERVAL 90 DAY;

-- Vérifier la politique
SELECT User, Host, password_lifetime
FROM mysql.user
WHERE User = 'compliance_user';
/*
+-----------------+------+-------------------+
| User            | Host | password_lifetime |
+-----------------+------+-------------------+
| compliance_user | %    |                90 |
+-----------------+------+-------------------+
*/

-- Modifier un utilisateur existant
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 180 DAY;

-- Pas d'expiration (déconseillé)
ALTER USER 'service_account'@'localhost' PASSWORD EXPIRE NEVER;
```

### Politique globale d'expiration

```sql
-- Définir politique par défaut pour TOUS les nouveaux utilisateurs
SET GLOBAL default_password_lifetime = 90;  -- 90 jours

-- Configuration permanente
# /etc/my.cnf.d/server.cnf
[mysqld]
default_password_lifetime = 90

-- Après cette config, tous les nouveaux utilisateurs ont expiration 90 jours
CREATE USER 'new_user'@'%' IDENTIFIED BY 'Secure2025!';
-- password_lifetime = 90 automatiquement
```

### Vérifier les mots de passe expirés

```sql
-- Lister utilisateurs avec expiration
SELECT
    User,
    Host,
    password_lifetime,
    CASE
        WHEN password_lifetime IS NULL THEN 'Never'
        WHEN password_lifetime = 0 THEN 'Global policy'
        ELSE CONCAT(password_lifetime, ' days')
    END AS expiration_policy
FROM mysql.user
WHERE User NOT IN ('root', 'mariadb.sys')
ORDER BY password_lifetime DESC;

-- Identifier utilisateurs avec mot de passe expiré
SELECT User, Host, password_last_changed, password_lifetime
FROM mysql.user
WHERE password_last_changed + INTERVAL password_lifetime DAY < NOW()
  AND password_lifetime > 0;
```

### Script de notification expiration

```bash
#!/bin/bash
# check_password_expiration.sh
# Notifier les utilisateurs avec mot de passe expirant dans 7 jours

WARN_DAYS=7
EMAIL_ADMIN="admin@example.com"

mariadb -N -B -e "
SELECT
    CONCAT(User, '@', Host) AS user_host,
    DATEDIFF(password_last_changed + INTERVAL password_lifetime DAY, NOW()) AS days_left
FROM mysql.user
WHERE password_lifetime > 0
  AND DATEDIFF(password_last_changed + INTERVAL password_lifetime DAY, NOW()) <= $WARN_DAYS
  AND DATEDIFF(password_last_changed + INTERVAL password_lifetime DAY, NOW()) > 0
" | while IFS=$'\t' read -r user_host days_left; do
    echo "WARNING: Password for $user_host expires in $days_left days" | \
        mail -s "Password Expiration Warning" $EMAIL_ADMIN
done
```

---

## Historique et réutilisation de mots de passe

### PASSWORD HISTORY (MariaDB 10.10+)

**Empêcher la réutilisation des N derniers mots de passe** :

```sql
-- Empêcher réutilisation des 4 derniers mots de passe (PCI-DSS)
CREATE USER 'secure_user'@'%'
  IDENTIFIED BY 'Initial2025#Pass'
  PASSWORD HISTORY 4;

-- Tenter de réutiliser un ancien mot de passe
ALTER USER 'secure_user'@'%' IDENTIFIED BY 'Initial2025#Pass';
-- ERROR 3638 (HY000): Cannot use these credentials for 'secure_user'@'%'
-- because they contradict the password history policy

-- Utiliser un nouveau mot de passe
ALTER USER 'secure_user'@'%' IDENTIFIED BY 'NewSecure2025!';
-- Query OK, 0 rows affected

-- Vérifier l'historique
SELECT User, Host, password_reuse_history
FROM mysql.user
WHERE User = 'secure_user';
```

### PASSWORD REUSE INTERVAL

**Empêcher la réutilisation pendant N jours** :

```sql
-- Ne pas réutiliser mot de passe pendant 365 jours (1 an)
CREATE USER 'annual_user'@'%'
  IDENTIFIED BY 'Year2025#Pass'
  PASSWORD REUSE INTERVAL 365 DAY;

-- Combiner historique + intervalle
CREATE USER 'strict_user'@'%'
  IDENTIFIED BY 'Strict2025#Pass'
  PASSWORD HISTORY 5
  PASSWORD REUSE INTERVAL 180 DAY;
-- → Ne peut réutiliser les 5 derniers ET pas avant 180 jours
```

### Politique globale historique

```sql
-- Politique par défaut (tous nouveaux utilisateurs)
SET GLOBAL password_history = 4;
SET GLOBAL password_reuse_interval = 365;

-- Configuration permanente
# /etc/my.cnf.d/server.cnf
[mysqld]
password_history = 4
password_reuse_interval = 365
```

---

## Politiques complètes par cas d'usage

### Cas 1 : PCI-DSS Compliance

```sql
-- Configuration globale PCI-DSS
SET GLOBAL simple_password_check_minimal_length = 12;
SET GLOBAL simple_password_check_digits = 1;
SET GLOBAL simple_password_check_letters_same_case = 2;
SET GLOBAL simple_password_check_other_characters = 1;
SET GLOBAL default_password_lifetime = 90;
SET GLOBAL password_history = 4;

-- Utilisateur PCI-DSS
CREATE USER 'payment_app'@'payment_server'
  IDENTIFIED VIA ed25519 USING PASSWORD('PaymentSecure2025!#')
  PASSWORD EXPIRE INTERVAL 90 DAY
  PASSWORD HISTORY 4
  REQUIRE SSL;

-- Vérification
SHOW CREATE USER 'payment_app'@'payment_server'\G
```

### Cas 2 : NIST SP 800-63B

```sql
-- Configuration NIST (focus dictionnaire)
# /etc/my.cnf.d/server.cnf
[mysqld]
plugin-load-add = cracklib_password_check.so
simple_password_check_minimal_length = 8

-- Utilisateur NIST
CREATE USER 'nist_user'@'%'
  IDENTIFIED BY 'ComplexUnique2025!#'
  PASSWORD EXPIRE INTERVAL 365 DAY;  -- NIST recommande 1 an ou plus
```

### Cas 3 : Haute sécurité (bancaire)

```sql
-- Configuration maximale
SET GLOBAL simple_password_check_minimal_length = 16;
SET GLOBAL simple_password_check_digits = 2;
SET GLOBAL simple_password_check_letters_same_case = 3;
SET GLOBAL simple_password_check_other_characters = 2;
SET GLOBAL default_password_lifetime = 60;  -- 60 jours
SET GLOBAL password_history = 10;
SET GLOBAL password_reuse_interval = 730;  -- 2 ans

-- Plugin cracklib également
INSTALL SONAME 'cracklib_password_check';

-- Utilisateur bancaire
CREATE USER 'bank_admin'@'bank_network'
  IDENTIFIED VIA ed25519 USING PASSWORD('BankSecure2025!#Admin$')
  PASSWORD EXPIRE INTERVAL 60 DAY
  PASSWORD HISTORY 10
  PASSWORD REUSE INTERVAL 730 DAY
  REQUIRE X509;  -- Certificat client obligatoire
```

### Cas 4 : Comptes service (pas d'expiration)

```sql
-- Compte service automatisé (pas d'expiration)
CREATE USER 'monitoring_bot'@'monitoring_server'
  IDENTIFIED VIA unix_socket  -- Authentification OS
  PASSWORD EXPIRE NEVER;  -- Service automatisé

-- OU avec mot de passe (stocké dans Vault)
CREATE USER 'backup_service'@'backup_server'
  IDENTIFIED VIA ed25519 USING PASSWORD('BackupService2025!#LongTerm')
  PASSWORD EXPIRE NEVER;

-- Privilèges minimaux
GRANT SELECT, LOCK TABLES ON *.* TO 'backup_service'@'backup_server';
```

---

## Audit des mots de passe existants

### Identifier utilisateurs non-conformes

```sql
-- Utilisateurs sans politique d'expiration
SELECT User, Host, password_lifetime
FROM mysql.user
WHERE password_lifetime IS NULL
  AND User NOT IN ('root', 'mariadb.sys');

-- Utilisateurs avec politique faible
SELECT User, Host, password_lifetime
FROM mysql.user
WHERE password_lifetime > 365  -- Plus d'1 an
  OR password_lifetime = 0;    -- Politique globale seulement

-- Utilisateurs sans historique
SELECT User, Host, password_reuse_history
FROM mysql.user
WHERE password_reuse_history = 0
  AND User NOT IN ('root', 'mariadb.sys');
```

### Script d'audit complet

```bash
#!/bin/bash
# audit_password_policies.sh

echo "=== Password Policy Audit ==="
echo ""

# Vérifier plugins installés
echo "1. Password Validation Plugins:"
mariadb -N -B -e "SHOW PLUGINS WHERE Type = 'PASSWORD VALIDATION'"

echo ""
echo "2. Global Password Settings:"
mariadb -e "SHOW VARIABLES LIKE '%password%'"

echo ""
echo "3. Users without expiration policy:"
mariadb -e "
SELECT User, Host, 'No expiration' AS issue
FROM mysql.user
WHERE password_lifetime IS NULL
  AND User NOT IN ('root', 'mariadb.sys')
"

echo ""
echo "4. Users with weak expiration (>365 days):"
mariadb -e "
SELECT User, Host, password_lifetime, 'Weak expiration' AS issue
FROM mysql.user
WHERE password_lifetime > 365
"

echo ""
echo "5. Users without password history:"
mariadb -e "
SELECT User, Host, 'No history' AS issue
FROM mysql.user
WHERE (password_reuse_history IS NULL OR password_reuse_history = 0)
  AND User NOT IN ('root', 'mariadb.sys')
"

echo ""
echo "6. Passwords expiring soon (next 30 days):"
mariadb -e "
SELECT
    User,
    Host,
    DATEDIFF(password_last_changed + INTERVAL password_lifetime DAY, NOW()) AS days_left
FROM mysql.user
WHERE password_lifetime > 0
  AND DATEDIFF(password_last_changed + INTERVAL password_lifetime DAY, NOW()) BETWEEN 0 AND 30
ORDER BY days_left
"
```

---

## Intégration PAM pour politiques système

### Configuration PAM

**Avantage** : Utiliser les politiques de mot de passe **système** (pam_pwquality).

```bash
# /etc/pam.d/mariadb
auth        required      pam_unix.so
account     required      pam_unix.so

# Validation mot de passe via pam_pwquality
password    requisite     pam_pwquality.so retry=3
password    sufficient    pam_unix.so sha512 shadow
password    required      pam_deny.so
```

**Configuration pam_pwquality** :

```bash
# /etc/security/pwquality.conf
# Longueur minimum
minlen = 12

# Complexité
dcredit = -1     # Au moins 1 chiffre
ucredit = -1     # Au moins 1 majuscule
lcredit = -1     # Au moins 1 minuscule
ocredit = -1     # Au moins 1 spécial

# Dictionnaire
dictcheck = 1    # Vérifier dictionnaire

# Répétitions
maxrepeat = 2    # Max 2 caractères identiques consécutifs

# Séquences
maxsequence = 3  # Max 3 caractères en séquence (abc, 123)
```

**Utilisateur MariaDB avec PAM** :

```sql
-- Installer plugin PAM
INSTALL SONAME 'auth_pam';

-- Créer utilisateur PAM
CREATE USER 'pam_user'@'%'
  IDENTIFIED VIA pam USING 'mariadb';

-- Changement de mot de passe validé par PAM
-- SET PASSWORD = PASSWORD('weak');
-- → Rejeté par pam_pwquality si non-conforme
```

---

## Migration vers politiques strictes

### Stratégie de migration

**Étape 1 : Audit** (identifier non-conformités)

```sql
-- Lister tous les utilisateurs
SELECT User, Host, password_lifetime, password_reuse_history
FROM mysql.user
WHERE User NOT IN ('root', 'mariadb.sys');
```

**Étape 2 : Activer plugins** (sans forcer immédiatement)

```sql
-- Installer plugins
INSTALL SONAME 'simple_password_check';

-- Configuration modérée au début
SET GLOBAL simple_password_check_minimal_length = 8;
SET GLOBAL simple_password_check_digits = 1;
SET GLOBAL simple_password_check_letters_same_case = 1;
SET GLOBAL simple_password_check_other_characters = 0;
```

**Étape 3 : Notification** (prévenir les utilisateurs)

```bash
# Email à tous les utilisateurs
cat <<EOF | mail -s "Password Policy Update" users@example.com
La politique de mots de passe sera renforcée dans 30 jours.

Nouvelles exigences:
- Minimum 12 caractères
- Au moins 1 majuscule, 1 minuscule, 1 chiffre, 1 spécial
- Expiration tous les 90 jours
- Pas de réutilisation des 4 derniers mots de passe

Merci de préparer vos nouveaux mots de passe.
EOF
```

**Étape 4 : Renforcer progressivement**

```sql
-- Semaine 1: Augmenter longueur
SET GLOBAL simple_password_check_minimal_length = 10;

-- Semaine 2: Ajouter caractères spéciaux
SET GLOBAL simple_password_check_other_characters = 1;

-- Semaine 3: Augmenter complexité
SET GLOBAL simple_password_check_minimal_length = 12;
SET GLOBAL simple_password_check_letters_same_case = 2;

-- Semaine 4: Activer expiration
SET GLOBAL default_password_lifetime = 90;
```

**Étape 5 : Forcer changement** (utilisateurs existants)

```sql
-- Forcer tous les utilisateurs à changer leur mot de passe
UPDATE mysql.user
SET password_lifetime = 1  -- Expire dans 1 jour
WHERE User NOT IN ('root', 'mariadb.sys')
  AND (password_lifetime IS NULL OR password_lifetime = 0);

FLUSH PRIVILEGES;
```

---

## Bonnes pratiques

### ✅ À faire

1. **Activer validation pour TOUS les environnements** (dev, staging, prod)
```sql
# Même en dev, habituer les développeurs
plugin-load-add = simple_password_check.so
simple_password_check_minimal_length = 12
```

2. **Politique adaptée au risque**
```
Production/PCI-DSS: 12+ chars, complexité, 90 jours
Dev/Staging:       10+ chars, complexité, 180 jours
Services (bots):   16+ chars, pas d'expiration
```

3. **Combiner plugins pour sécurité maximale**
```sql
plugin-load-add = simple_password_check.so
plugin-load-add = cracklib_password_check.so
```

4. **Documenter les politiques**
```sql
-- Créer une table de documentation
CREATE TABLE security.password_policy_doc (
    environment ENUM('dev', 'staging', 'production'),
    min_length INT,
    complexity VARCHAR(255),
    expiration_days INT,
    history_count INT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO security.password_policy_doc VALUES
('production', 12, '1 digit, 2 uppercase, 2 lowercase, 1 special', 90, 4, NOW());
```

5. **Automatiser les audits**
```bash
# Cron job quotidien
0 9 * * * /usr/local/bin/audit_password_policies.sh | mail -s "Password Audit" admin@example.com
```

### ❌ À éviter

1. **Pas de validation** (vulnérabilité majeure)
```sql
# ❌ MAUVAIS
# Pas de plugin → mots de passe faibles acceptés
```

2. **Politique trop faible**
```sql
# ❌ MAUVAIS
simple_password_check_minimal_length = 6  # Trop court
default_password_lifetime = 0  # Jamais expiré
```

3. **Exceptions sans justification**
```sql
# ❌ MAUVAIS
ALTER USER 'admin'@'%' PASSWORD EXPIRE NEVER;
-- Même les admins doivent changer leur mot de passe!

# ✅ BON: Exception documentée
-- Seulement pour comptes service (bots)
ALTER USER 'monitoring_bot'@'localhost' PASSWORD EXPIRE NEVER;
-- Justification: Service automatisé, mot de passe dans Vault
```

4. **Forcer changement sans préavis**
```sql
# ❌ MAUVAIS
UPDATE mysql.user SET password_lifetime = 1;
-- Utilisateurs bloqués sans avertissement!

# ✅ BON: Notification + délai
-- Email 30 jours avant
-- Puis activer progressivement
```

---

## Troubleshooting

### Problème 1 : Plugin ne se charge pas

```bash
# Symptôme
INSTALL SONAME 'simple_password_check';
ERROR 1126 (HY000): Can't open shared library

# Diagnostic
ls -la /usr/lib64/mysql/plugin/simple_password_check.so
# -rwxr-xr-x 1 root root simple_password_check.so

# Solution: Vérifier permissions
sudo chmod 755 /usr/lib64/mysql/plugin/simple_password_check.so
sudo chown mysql:mysql /usr/lib64/mysql/plugin/simple_password_check.so

# Redémarrer MariaDB
sudo systemctl restart mariadb
```

### Problème 2 : Politique trop stricte (blocage utilisateurs)

```sql
-- Symptôme
CREATE USER 'test'@'localhost' IDENTIFIED BY 'MyPassword2025!';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

-- Diagnostic: Vérifier configuration
SHOW VARIABLES LIKE 'simple_password_check%';

-- Solution temporaire: Assouplir
SET GLOBAL simple_password_check_minimal_length = 10;

-- Puis tester
CREATE USER 'test'@'localhost' IDENTIFIED BY 'MyPass25!';
-- Query OK
```

### Problème 3 : Utilisateurs bloqués après expiration

```bash
# Symptôme
mariadb -u user -p
ERROR 1820 (HY000): You must SET PASSWORD before executing this statement

# Solution 1: Admin change le mot de passe
mariadb -u root -p
ALTER USER 'user'@'localhost' IDENTIFIED BY 'NewSecure2025!';

# Solution 2: Utilisateur change via SET PASSWORD
mariadb -u user -p
SET PASSWORD = PASSWORD('NewSecure2025!');
```

### Problème 4 : Cracklib dictionnaire manquant

```bash
# Symptôme
INSTALL SONAME 'cracklib_password_check';
# Plugin charge mais ne valide pas

# Diagnostic
ls /usr/share/cracklib/
# Vide ou manquant

# Solution
sudo dnf install cracklib-dicts  # RHEL
sudo apt install cracklib-runtime  # Debian

# Vérifier
ls -lh /usr/share/cracklib/
# pw_dict.*
```

---

## ✅ Points clés à retenir

- **Plugins natifs** : simple_password_check (recommandé) + cracklib_password_check
- **Validation obligatoire** : activer dans tous les environnements
- **PCI-DSS exige** : 12 chars min, complexité, rotation 90 jours, historique 4
- **PASSWORD EXPIRE** : forcer changement périodique (90-365 jours)
- **PASSWORD HISTORY** : empêcher réutilisation (4-10 derniers)
- **Combiner plugins** : simple + cracklib = sécurité maximale
- **Politique globale** : default_password_lifetime, password_history
- **Migration progressive** : notifier, assouplir puis renforcer
- **Audit régulier** : scripts automatisés pour détecter non-conformités
- **Comptes service** : exception PASSWORD EXPIRE NEVER (documentée)

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 Password Validation Plugins](https://mariadb.com/kb/en/password-validation/)
- [📖 simple_password_check](https://mariadb.com/kb/en/simple_password_check-plugin/)
- [📖 cracklib_password_check](https://mariadb.com/kb/en/cracklib_password_check-plugin/)
- [📖 Password Expiration](https://mariadb.com/kb/en/user-password-expiry/)

### Standards et conformité

- [PCI-DSS v4.0 Requirement 8.3](https://www.pcisecuritystandards.org/)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [ISO 27001:2022](https://www.iso.org/standard/27001)

### Outils

- [cracklib](https://github.com/cracklib/cracklib)
- [zxcvbn (password strength estimator)](https://github.com/dropbox/zxcvbn)

---

## ➡️ Conclusion du chapitre 10

Vous avez maintenant une maîtrise complète de la **sécurité et gestion des utilisateurs** dans MariaDB 11.8 :

- ✅ **Modèle de sécurité** : User@Host, privilèges hiérarchiques
- ✅ **Gestion utilisateurs** : CREATE/ALTER/DROP, authentication plugins
- ✅ **Système de privilèges** : GRANT/REVOKE, granularité table/colonne
- ✅ **Rôles (RBAC)** : Simplification gestion, hiérarchies
- ✅ **Plugins authentification** : ed25519, PAM, GSSAPI, 🆕 PARSEC (HSM)
- ✅ **SSL/TLS** : 🆕 Activé par défaut (11.8), TLS 1.3
- ✅ **Audit** : Server Audit Plugin, conformité PCI-DSS/RGPD
- ✅ **Sécurité applicative** : Prepared statements, secrets management
- ✅ **Politiques mots de passe** : Validation, expiration, historique

**Le chapitre suivant** abordera la **réplication et haute disponibilité**.

---


⏭️ [Privilèges granulaires (nouvelles options 11.8)](/10-securite-gestion-utilisateurs/11-privileges-granulaires.md)

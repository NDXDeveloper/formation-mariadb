🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.5 Authentification : Plugins

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 10.1-10.4 (Modèle de sécurité, Utilisateurs, Privilèges, Rôles)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture pluggable du système d'authentification MariaDB
- **Comparer** les différents plugins disponibles (native, ed25519, PAM, unix_socket, GSSAPI, PARSEC)
- **Choisir** le plugin approprié selon le contexte (sécurité, performance, intégration)
- **Migrer** entre différents plugins d'authentification
- **Configurer** l'authentification multi-plugin (fallback)
- **Appliquer** les meilleures pratiques de sécurité par plugin
- **Identifier** les nouveautés MariaDB 11.8 🆕 (PARSEC, amélioration ed25519)
- **Diagnostiquer** les problèmes d'authentification

---

## Introduction

L'**authentification** est la première barrière de sécurité de MariaDB : elle détermine **qui peut accéder au serveur**. Contrairement à de nombreux SGBD qui utilisent un seul mécanisme d'authentification, MariaDB propose une **architecture pluggable** qui permet de choisir parmi plusieurs plugins selon les besoins de sécurité, de performance et d'intégration.

### Pourquoi plusieurs plugins d'authentification ?

Différents contextes nécessitent différentes approches d'authentification :

```
┌────────────────────────────────────────────────────────────────┐
│  CONTEXTE                        PLUGIN APPROPRIÉ              │
├────────────────────────────────────────────────────────────────┤
│  Application legacy             mysql_native_password          │
│  Nouvelle application moderne   ed25519                        │
│  Intégration Active Directory   GSSAPI (Kerberos)              │
│  SSO entreprise                 PAM (LDAP/AD)                  │
│  Scripts locaux                 unix_socket                    │
│  Conformité PCI-DSS/FIPS        PARSEC (HSM) 🆕                │
│  Multi-factor authentication    PAM + Google Authenticator     │
└────────────────────────────────────────────────────────────────┘
```

**Exemple concret** : Dans une grande entreprise, vous pourriez avoir :
- **Employés** : Authentification via Active Directory (GSSAPI)
- **Applications** : Authentification forte moderne (ed25519)
- **Scripts système locaux** : Authentification OS (unix_socket)
- **Services de paiement** : Authentification matérielle (PARSEC HSM)

---

## Architecture du système d'authentification

### Fonctionnement général

L'authentification dans MariaDB suit un processus en plusieurs étapes :

```
┌────────────────────────────────────────────────────────────────┐
│  ÉTAPE 1: CONNEXION CLIENT                                     │
│  mariadb -u alice -p -h db.example.com                         │
└────────────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────────┐
│  ÉTAPE 2: IDENTIFICATION                                       │
│  → Correspondance 'user'@'host' dans mysql.user/global_priv    │
│  → Sélection du plugin d'authentification configuré            │
└────────────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────────┐
│  ÉTAPE 3: NÉGOCIATION PROTOCOLE                                │
│  → Client et serveur s'accordent sur le plugin                 │
│  → Échange des paramètres d'authentification                   │
└────────────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────────┐
│  ÉTAPE 4: VÉRIFICATION (selon le plugin)                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ mysql_native_password                                    │  │
│  │ → Hash SHA1 du mot de passe envoyé                       │  │
│  │ → Comparaison avec authentication_string                 │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ ed25519                                                  │  │
│  │ → Signature EdDSA vérifiée                               │  │
│  │ → Cryptographie à courbes elliptiques                    │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ PAM                                                      │  │
│  │ → Délégation à PAM (LDAP/AD/2FA)                         │  │
│  │ → Modules PAM système appelés                            │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ unix_socket                                              │  │
│  │ → Vérification UID/GID Unix du processus                 │  │
│  │ → Pas de mot de passe nécessaire                         │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ GSSAPI                                                   │  │
│  │ → Ticket Kerberos vérifié                                │  │
│  │ → Intégration Active Directory                           │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ PARSEC 🆕                                                │  │
│  │ → Vérification via HSM (Hardware Security Module)        │  │
│  │ → Clés cryptographiques matérielles                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────────┐
│  ÉTAPE 5: RÉSULTAT                                             │
│  ✓ Authentification réussie → Connexion établie                │
│  ✗ Authentification échouée → ERROR 1045 (Access denied)       │
└────────────────────────────────────────────────────────────────┘
```

### Architecture pluggable

MariaDB utilise une **architecture modulaire** pour l'authentification :

```
┌──────────────────────────────────────────────────────────────┐
│              MARIADB SERVER CORE                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Authentication Plugin API                      │  │
│  │  Interface standardisée pour tous les plugins          │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ↓                                   │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────┬──────┐  │
│  │ native  │ ed25519 │   PAM   │  unix   │ GSSAPI  │PARSEC│  │
│  │password │         │         │ socket  │         │ 🆕   │  │
│  └─────────┴─────────┴─────────┴─────────┴─────────┴──────┘  │
│       ↓         ↓         ↓         ↓         ↓         ↓    │
│  [SHA1]   [EdDSA]   [PAM lib]  [SO_PEERCRED] [Kerberos][HSM] │
└──────────────────────────────────────────────────────────────┘
```

**Avantages de cette architecture** :

1. ✅ **Flexibilité** : Choisir le meilleur plugin par contexte
2. ✅ **Évolutivité** : Nouveaux plugins sans modifier le core
3. ✅ **Compatibilité** : Support de plugins legacy et modernes
4. ✅ **Intégration** : S'adapter aux infrastructures existantes (AD, LDAP)
5. ✅ **Sécurité** : Plugins spécialisés pour contraintes réglementaires

---

## Vue d'ensemble des plugins disponibles

### Tableau comparatif complet

| Plugin | Sécurité | Performance | Complexité | Cas d'usage principal | Disponibilité |
|--------|----------|-------------|------------|----------------------|---------------|
| **mysql_native_password** | 🟡 Moyenne (SHA1) | 🟢 Rapide | 🟢 Simple | Applications legacy | ✅ Built-in |
| **ed25519** | 🟢 Haute (EdDSA) | 🟢 Très rapide | 🟢 Simple | **Recommandé 2025** | ✅ Built-in (10.1+) |
| **unix_socket** | 🟢 Haute | 🟢 Très rapide | 🟢 Simple | Scripts locaux | ✅ Built-in |
| **pam** | 🟢 Haute | 🟡 Moyenne | 🔴 Complexe | SSO, LDAP, 2FA | ⚙️ Plugin externe |
| **gssapi** | 🟢 Haute | 🟡 Moyenne | 🔴 Complexe | Active Directory | ⚙️ Plugin externe |
| **sha256_password** | 🟢 Haute | 🟡 Lent | 🟢 Simple | MySQL 5.7+ compat | ✅ Built-in |
| **🆕 parsec** | 🟢 Très haute | 🟡 Moyenne | 🔴 Très complexe | PCI-DSS, FIPS | ⚙️ Plugin (11.8+) |

**Légende** :
- 🟢 Excellent / Recommandé
- 🟡 Correct / Acceptable
- 🔴 Limité / Complexe

### Descriptions courtes

#### 1. mysql_native_password (Legacy)

**Description** : Plugin historique utilisant SHA1 (double hash).

**Caractéristiques** :
- Algorithme : `SHA1(SHA1(password))`
- Vulnérable aux attaques rainbow tables
- Rapide mais cryptographiquement faible
- Compatible avec tous les clients MySQL/MariaDB

**Verdict 2025** : ⚠️ **À éviter** pour les nouveaux déploiements. Migrer vers ed25519.

```sql
-- Utilisation (déconseillée)
CREATE USER 'legacy_app'@'%'
  IDENTIFIED BY 'password';
-- Par défaut: mysql_native_password
```

#### 2. ed25519 (Recommandé) ⭐

**Description** : Authentification moderne basée sur EdDSA (cryptographie à courbes elliptiques).

**Caractéristiques** :
- Algorithme : EdDSA (Ed25519)
- Sécurité cryptographique maximale (256 bits)
- Performance excellente (plus rapide que SHA256)
- Résistant aux attaques par force brute

**Verdict 2025** : ✅ **Recommandé** pour tous les nouveaux déploiements.

```sql
-- Utilisation (recommandée)
CREATE USER 'modern_app'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('StrongPassword123!');
```

#### 3. unix_socket (Connexions locales)

**Description** : Authentification basée sur l'identité Unix (UID/GID).

**Caractéristiques** :
- Vérification de l'UID/GID du processus client
- Pas de mot de passe requis
- Fonctionne **uniquement** via socket Unix (`@'localhost'`)
- Parfait pour scripts et cron jobs

**Verdict 2025** : ✅ **Idéal** pour automatisation locale.

```sql
-- Utilisation
CREATE USER 'backup_script'@'localhost'
  IDENTIFIED VIA unix_socket;

-- Connexion depuis l'utilisateur Unix 'backup_script'
sudo -u backup_script mariadb
-- → Connexion automatique sans mot de passe
```

#### 4. pam (Pluggable Authentication Modules)

**Description** : Délègue l'authentification aux modules PAM du système.

**Caractéristiques** :
- Intégration LDAP, Active Directory, Radius
- Support 2FA/MFA (Google Authenticator, YubiKey)
- Politiques d'authentification centralisées
- Complexité de configuration élevée

**Verdict 2025** : ✅ **Excellent** pour entreprises avec SSO.

```sql
-- Utilisation
CREATE USER 'employee'@'%'
  IDENTIFIED VIA pam USING 'mariadb';
-- 'mariadb' = service PAM dans /etc/pam.d/mariadb
```

#### 5. gssapi (Kerberos)

**Description** : Authentification Kerberos pour environnements Windows/AD.

**Caractéristiques** :
- SSO (Single Sign-On) natif Windows
- Tickets Kerberos (pas de transmission de mot de passe)
- Intégration Active Directory transparente
- Configuration Kerberos requise

**Verdict 2025** : ✅ **Parfait** pour environnements Windows.

```sql
-- Utilisation
CREATE USER 'alice@EXAMPLE.COM'@'%'
  IDENTIFIED VIA gssapi;

-- Connexion avec ticket Kerberos (après kinit)
mariadb --plugin-dir=/usr/lib64/mysql/plugin
-- → SSO, pas de mot de passe demandé
```

#### 6. 🆕 parsec (Hardware Security Module)

**Description** : Authentification via HSM pour conformité maximale.

**Caractéristiques** :
- Clés cryptographiques matérielles (HSM)
- Conformité PCI-DSS, FIPS 140-2/3
- Impossibilité d'exfiltration des clés
- Coût élevé (matériel HSM requis)

**Verdict 2025** : ✅ **Indispensable** pour secteurs régulés (finance, santé).

```sql
-- Utilisation (MariaDB 11.8+)
CREATE USER 'payment_processor'@'payment_gateway'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/key_id_12345';
```

---

## Choix du plugin selon le contexte

### Matrice de décision

| Critère | Plugin recommandé | Raison |
|---------|-------------------|--------|
| **Nouvelle application** | ed25519 | Sécurité moderne, performance, simplicité |
| **Application legacy** | mysql_native_password | Compatibilité (migrer vers ed25519 ASAP) |
| **Scripts locaux** | unix_socket | Pas de gestion de mots de passe |
| **SSO entreprise** | PAM ou GSSAPI | Intégration infrastructure existante |
| **Active Directory** | GSSAPI | SSO natif Windows |
| **2FA/MFA requis** | PAM | Support modules 2FA (TOTP, U2F) |
| **Conformité PCI-DSS** | PARSEC 🆕 | HSM, FIPS 140-2 |
| **Secteur bancaire** | PARSEC 🆕 | Sécurité maximale, audit matériel |
| **Cloud public** | ed25519 + TLS | Sécurité transport + authentification forte |
| **Environnement mixte** | Multi-plugin | Fallback pour migration progressive |

### Arbre de décision

```
Quelle est votre priorité ?
│
├─→ Compatibilité legacy
│   └─→ mysql_native_password (temporaire, migrer vers ed25519)
│
├─→ Sécurité maximale
│   ├─→ Conformité réglementaire (PCI-DSS, FIPS) ?
│   │   └─→ OUI → PARSEC (HSM) 🆕
│   │   └─→ NON → ed25519
│   │
│   └─→ 2FA/MFA requis ?
│       └─→ OUI → PAM + Google Authenticator / YubiKey
│       └─→ NON → ed25519
│
├─→ Intégration infrastructure
│   ├─→ Active Directory ?
│   │   └─→ OUI → GSSAPI (Kerberos)
│   │
│   ├─→ LDAP / OpenLDAP ?
│   │   └─→ OUI → PAM
│   │
│   └─→ SSO entreprise ?
│       └─→ OUI → PAM ou GSSAPI
│
├─→ Scripts / Automatisation locale
│   └─→ unix_socket (pas de mot de passe)
│
└─→ Performance critique
    └─→ ed25519 (plus rapide que SHA256, aussi sécurisé)
```

### Exemples de configurations par secteur

#### Startup / PME moderne

```sql
-- Simplicité + Sécurité
-- Tout en ed25519

-- Applications
CREATE USER 'webapp'@'app_servers'
  IDENTIFIED VIA ed25519 USING PASSWORD('strong_pass');

-- Scripts locaux
CREATE USER 'backup'@'localhost'
  IDENTIFIED VIA unix_socket;

-- DBA
CREATE USER 'dba'@'localhost'
  IDENTIFIED VIA ed25519 USING PASSWORD('dba_strong_pass')
  REQUIRE SSL;
```

#### Grande entreprise (Active Directory)

```sql
-- Intégration AD + Scripts locaux

-- Employés via AD
CREATE USER 'alice@CORP.COM'@'%'
  IDENTIFIED VIA gssapi;

CREATE USER 'bob@CORP.COM'@'%'
  IDENTIFIED VIA gssapi;

-- Applications (ed25519)
CREATE USER 'api_service'@'api_servers'
  IDENTIFIED VIA ed25519 USING PASSWORD('api_pass');

-- Scripts système
CREATE USER 'monitoring'@'localhost'
  IDENTIFIED VIA unix_socket;
```

#### Secteur financier / Banque

```sql
-- Conformité PCI-DSS + FIPS

-- Services de paiement (HSM)
CREATE USER 'payment_processor'@'payment_gateway'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/payment_key_001';

-- Trading systems (HSM)
CREATE USER 'trading_engine'@'trading_servers'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/trading_key_001';

-- Employés (2FA obligatoire)
CREATE USER 'trader_alice'@'%'
  IDENTIFIED VIA pam USING 'mariadb-2fa';
-- /etc/pam.d/mariadb-2fa configure Google Authenticator

-- Audit / Monitoring
CREATE USER 'auditor'@'audit_server'
  IDENTIFIED VIA ed25519 USING PASSWORD('audit_pass')
  REQUIRE X509;
```

#### SaaS / Multi-tenant

```sql
-- Isolation par tenant + Performance

-- Tenant 1
CREATE USER 'tenant_001_app'@'app_servers'
  IDENTIFIED VIA ed25519 USING PASSWORD('tenant001_pass');

-- Tenant 2
CREATE USER 'tenant_002_app'@'app_servers'
  IDENTIFIED VIA ed25519 USING PASSWORD('tenant002_pass');

-- Platform admin (2FA)
CREATE USER 'platform_admin'@'%'
  IDENTIFIED VIA pam USING 'platform-2fa';

-- Backup global
CREATE USER 'backup_all'@'localhost'
  IDENTIFIED VIA unix_socket;
```

---

## Authentification multi-plugin (Fallback)

MariaDB 10.4+ supporte **plusieurs plugins** pour un même utilisateur, permettant une migration progressive ou un fallback.

### Syntaxe

```sql
CREATE USER 'user'@'host'
  IDENTIFIED VIA plugin1 USING 'auth_string1'
  OR IDENTIFIED VIA plugin2 USING 'auth_string2'
  OR IDENTIFIED VIA plugin3 USING 'auth_string3';
```

### Cas d'usage 1 : Migration progressive

**Scénario** : Migrer de mysql_native_password vers ed25519 sans coupure.

```sql
-- Phase 1: Ajouter ed25519 en fallback
ALTER USER 'app_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('new_strong_password')
  OR IDENTIFIED VIA mysql_native_password USING PASSWORD('old_password');

-- L'utilisateur peut se connecter avec:
-- 1. Nouveau mot de passe ed25519 (prioritaire)
-- 2. Ancien mot de passe native (fallback)

-- Phase 2: Déployer application avec nouveau mot de passe

-- Phase 3: Retirer mysql_native_password
ALTER USER 'app_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('new_strong_password');
```

### Cas d'usage 2 : Flexibilité d'authentification

**Scénario** : Permettre authentification PAM (SSO) OU mot de passe direct.

```sql
-- Utilisateur peut s'authentifier de deux façons
CREATE USER 'flexible_user'@'%'
  IDENTIFIED VIA pam USING 'mariadb'
  OR IDENTIFIED VIA ed25519 USING PASSWORD('backup_password');

-- Utilisation:
-- 1. Depuis le bureau (SSO/PAM) → Authentification transparente
-- 2. Depuis un script → Mot de passe ed25519
```

### Ordre d'évaluation

Les plugins sont essayés **dans l'ordre de déclaration** jusqu'à ce qu'un réussisse.

```sql
CREATE USER 'test'@'%'
  IDENTIFIED VIA plugin_A USING 'auth_A'    -- Essayé en 1er
  OR IDENTIFIED VIA plugin_B USING 'auth_B' -- Essayé en 2ème si A échoue
  OR IDENTIFIED VIA plugin_C USING 'auth_C';-- Essayé en 3ème si B échoue

-- Connexion réussit dès que l'un des plugins accepte
```

---

## 🆕 Nouveautés MariaDB 11.8

### 1. Plugin PARSEC (Hardware Security Module)

**Nouveau plugin** pour authentification via HSM.

```sql
-- Installation
INSTALL SONAME 'auth_parsec';

-- Vérification
SHOW PLUGINS WHERE Name = 'parsec';

-- Utilisation
CREATE USER 'hsm_user'@'localhost'
  IDENTIFIED VIA parsec USING 'parsec://provider/key_name';
```

**Providers supportés** :
- **PKCS#11** : HSM matériels (Thales, Gemalto, AWS CloudHSM)
- **TPM** : Trusted Platform Module (TPM 2.0)
- **Mbed Crypto** : Software HSM (dev/test)

**Configuration** (`/etc/parsec/config.toml`) :

```toml
[provider.pkcs11]
library = "/usr/lib/libpkcs11.so"
slot_number = 0

[provider.tpm]
tcti = "device:/dev/tpm0"
owner_hierarchy_auth = ""
```

**Cas d'usage** :
- ✅ Conformité PCI-DSS Level 1
- ✅ FIPS 140-2 Level 3
- ✅ Secteur bancaire/financier
- ✅ Cloud souverain

### 2. Amélioration ed25519

**Optimisations** :
- Performance accrue (SIMD sur AVX2/AVX512)
- Support natif ARM NEON
- Réduction latence authentification (~20% plus rapide)

**Benchmark MariaDB 11.8 vs 10.11** :

| Opération | MariaDB 10.11 | MariaDB 11.8 | Gain |
|-----------|---------------|--------------|------|
| Authentification ed25519 | 1.2 ms | 0.95 ms | **~21%** |
| Génération hash | 0.8 ms | 0.65 ms | **~19%** |

### 3. TLS par défaut

**Depuis MariaDB 11.8**, TLS est activé par défaut si des certificats sont présents.

```bash
# Certificats auto-générés au démarrage
ls -la /var/lib/mysql/*.pem
# ca.pem, server-cert.pem, server-key.pem, client-cert.pem, client-key.pem

# Vérification
mariadb -u root -p -e "SHOW VARIABLES LIKE 'have_ssl';"
# have_ssl = YES (par défaut en 11.8)
```

**Impact sur les plugins** :
- **ed25519** : Bénéficie de TLS par défaut (transport chiffré)
- **PAM/GSSAPI** : TLS recommandé pour éviter interception tickets
- **PARSEC** : TLS obligatoire (protection clés HSM)

---

## Configuration générale des plugins

### Installation de plugins externes

```sql
-- Lister les plugins disponibles
SHOW PLUGINS;

-- Installer un plugin
INSTALL SONAME 'auth_pam';
INSTALL SONAME 'auth_gssapi';
INSTALL SONAME 'auth_parsec';  -- 11.8+

-- Vérifier l'installation
SELECT PLUGIN_NAME, PLUGIN_STATUS, PLUGIN_TYPE, PLUGIN_LIBRARY
FROM information_schema.PLUGINS
WHERE PLUGIN_TYPE = 'AUTHENTICATION';

-- Désinstaller un plugin
UNINSTALL SONAME 'auth_pam';
```

### Configuration permanente

**Charger plugins au démarrage** (`/etc/my.cnf.d/server.cnf`) :

```ini
[mysqld]
# Charger plugins d'authentification
plugin-load-add = auth_pam.so
plugin-load-add = auth_gssapi.so
plugin-load-add = auth_parsec.so  # 11.8+

# Configuration PAM
pam_use_cleartext_plugin = ON

# Configuration GSSAPI
gssapi_principal_name = mariadb/db.example.com@EXAMPLE.COM
gssapi_keytab_path = /etc/krb5.keytab
```

### Vérification de la disponibilité

```sql
-- Vérifier quels plugins d'authentification sont chargés
SELECT PLUGIN_NAME, PLUGIN_STATUS, PLUGIN_LIBRARY
FROM information_schema.PLUGINS
WHERE PLUGIN_TYPE = 'AUTHENTICATION'
ORDER BY PLUGIN_NAME;

/*
+-------------------------+---------------+--------------------+
| PLUGIN_NAME             | PLUGIN_STATUS | PLUGIN_LIBRARY     |
+-------------------------+---------------+--------------------+
| ed25519                 | ACTIVE        | NULL               |
| gssapi                  | ACTIVE        | auth_gssapi.so     |
| mysql_native_password   | ACTIVE        | NULL               |
| pam                     | ACTIVE        | auth_pam.so        |
| parsec                  | ACTIVE        | auth_parsec.so     |
| unix_socket             | ACTIVE        | NULL               |
+-------------------------+---------------+--------------------+
*/
```

---

## Migration entre plugins

### Stratégie de migration

**Étape 1 : Audit de l'existant**

```sql
-- Lister tous les utilisateurs avec leur plugin
SELECT User, Host, plugin
FROM mysql.user
WHERE User NOT IN ('root', 'mariadb.sys')
ORDER BY plugin, User;

-- Compter par plugin
SELECT plugin, COUNT(*) AS user_count
FROM mysql.user
WHERE User NOT IN ('root', 'mariadb.sys')
GROUP BY plugin
ORDER BY user_count DESC;
```

**Étape 2 : Test sur un utilisateur pilote**

```sql
-- Créer un utilisateur test
CREATE USER 'migration_test'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('test_password');

-- Tester la connexion
mariadb -u migration_test -p -h db.example.com
```

**Étape 3 : Migration avec multi-plugin (zero downtime)**

```sql
-- Ajouter ed25519 en gardant mysql_native_password
ALTER USER 'app_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('new_password')
  OR IDENTIFIED VIA mysql_native_password USING PASSWORD('old_password');

-- L'application peut utiliser l'un ou l'autre
-- Période de transition pour déployer le nouveau mot de passe
```

**Étape 4 : Retrait de l'ancien plugin**

```sql
-- Après vérification que tout fonctionne avec ed25519
ALTER USER 'app_user'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('new_password');
```

### Script de migration massif

```bash
#!/bin/bash
# migrate_to_ed25519.sh

DB_USER="root"
DB_PASS="password"
OLD_PLUGIN="mysql_native_password"
NEW_PLUGIN="ed25519"

# Liste des utilisateurs à migrer
mariadb -u $DB_USER -p$DB_PASS -N -B -e "
  SELECT CONCAT(\"'\", User, \"'@'\", Host, \"'\")
  FROM mysql.user
  WHERE plugin = '${OLD_PLUGIN}'
    AND User NOT IN ('root', 'mariadb.sys')
" | while read user; do

  echo "Migrating user: $user"

  # Générer un mot de passe fort
  NEW_PASSWORD=$(openssl rand -base64 24)

  # Ajouter ed25519 en gardant l'ancien (multi-plugin)
  mariadb -u $DB_USER -p$DB_PASS <<EOF
    ALTER USER $user
      IDENTIFIED VIA ${NEW_PLUGIN} USING PASSWORD('${NEW_PASSWORD}')
      OR IDENTIFIED VIA ${OLD_PLUGIN};
EOF

  # Stocker dans Vault
  echo "User: $user, New password: $NEW_PASSWORD" >> /tmp/migration_passwords.txt

done

echo "Migration phase 1 complete. Users can use old OR new password."
echo "Deploy new passwords, then run phase 2 to remove old plugin."
```

---

## Sécurité et bonnes pratiques

### 1. Éviter mysql_native_password

```sql
-- ❌ MAUVAIS: mysql_native_password pour nouveau déploiement
CREATE USER 'new_app'@'%' IDENTIFIED BY 'password';
-- Utilise mysql_native_password par défaut (SHA1, faible)

-- ✅ BON: ed25519 pour nouveau déploiement
CREATE USER 'new_app'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('StrongPassword123!');
```

### 2. Forcer TLS pour plugins vulnérables

```sql
-- PAM transmet des credentials en clair → Forcer TLS
CREATE USER 'pam_user'@'%'
  IDENTIFIED VIA pam USING 'mariadb'
  REQUIRE SSL;

-- GSSAPI peut être intercepté → TLS recommandé
CREATE USER 'kerberos_user'@'%'
  IDENTIFIED VIA gssapi
  REQUIRE SSL;
```

### 3. Restreindre les plugins autorisés

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
# Liste blanche des plugins autorisés
plugin-load-add = auth_ed25519.so
plugin-load-add = auth_unix_socket.so

# Refuser tous les autres
# (incluant mysql_native_password)
```

### 4. Audit des connexions par plugin

```sql
-- Activer audit plugin
INSTALL SONAME 'server_audit';

SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_events = 'CONNECT';

-- Vérifier dans /var/log/mysql/audit.log
-- Identifier les utilisateurs utilisant encore mysql_native_password
```

### 5. Rotation des credentials par plugin

| Plugin | Rotation recommandée | Méthode |
|--------|---------------------|---------|
| mysql_native_password | 90 jours | ALTER USER ... IDENTIFIED BY |
| ed25519 | 180 jours | ALTER USER ... IDENTIFIED VIA ed25519 USING PASSWORD() |
| unix_socket | N/A | Lié à l'utilisateur OS |
| PAM | Selon politique LDAP/AD | Géré par annuaire |
| GSSAPI | Selon politique Kerberos | Géré par KDC |
| PARSEC | N/A | Clés matérielles HSM |

---

## Tests et troubleshooting

### Tests de base

```sql
-- Test 1: Vérifier le plugin d'un utilisateur
SELECT User, Host, plugin, authentication_string
FROM mysql.user
WHERE User = 'test_user';

-- Test 2: Tester la connexion
mariadb -u test_user -p -h localhost

-- Test 3: Vérifier le plugin actif pendant la session
SELECT USER(), CURRENT_USER(), @@plugin_dir;
```

### Diagnostic des échecs d'authentification

```bash
# Activer le debug logging
mariadb --debug=d:t:i:o,/tmp/mariadb_debug.log -u user -p

# Analyser les logs
tail -f /var/log/mysql/error.log

# Rechercher les erreurs d'authentification
grep "Access denied" /var/log/mysql/error.log
grep "Plugin" /var/log/mysql/error.log
```

### Problèmes courants

**Problème 1 : Plugin non trouvé côté client**

```bash
# Symptôme
ERROR 2059 (HY000): Authentication plugin 'ed25519' cannot be loaded

# Solution
# Installer le plugin client
sudo apt-get install mariadb-plugin-auth-ed25519  # Debian/Ubuntu

# Ou spécifier le chemin
mariadb --plugin-dir=/usr/lib64/mysql/plugin -u user -p
```

**Problème 2 : PAM service non trouvé**

```bash
# Symptôme
ERROR 1045 (28000): Access denied for user 'user'@'host' (using password: YES)

# Diagnostic
ls /etc/pam.d/mariadb  # Vérifier que le fichier existe

# Solution: Créer le service PAM
sudo vi /etc/pam.d/mariadb
# auth    required   pam_unix.so
# account required   pam_unix.so
```

**Problème 3 : GSSAPI - Ticket Kerberos expiré**

```bash
# Symptôme
ERROR 1045 (28000): Access denied

# Diagnostic
klist  # Vérifier les tickets Kerberos

# Solution: Renouveler le ticket
kinit user@EXAMPLE.COM
```

---

## ✅ Points clés à retenir

- **MariaDB propose 6+ plugins d'authentification** pour s'adapter à tous les contextes
- **ed25519 est recommandé pour 2025** : sécurité maximale + performance
- **mysql_native_password est obsolète** (SHA1) : à éviter pour nouveaux déploiements
- **unix_socket est idéal pour scripts locaux** : pas de gestion de mots de passe
- **PAM permet SSO et 2FA** : intégration LDAP/AD/Google Authenticator
- **GSSAPI offre SSO Windows** : tickets Kerberos pour Active Directory
- **🆕 PARSEC (11.8) apporte HSM** : conformité PCI-DSS/FIPS pour secteurs régulés
- **Multi-plugin permet migration progressive** : fallback sans coupure
- **🆕 TLS est activé par défaut (11.8)** : protection transport automatique
- **Choisir le plugin selon le contexte** : sécurité, intégration, performance

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Authentication Plugins Overview](https://mariadb.com/kb/en/authentication-plugins/)
- [📖 Authentication from MariaDB 10.4](https://mariadb.com/kb/en/authentication-from-mariadb-104/)
- [📖 🆕 PARSEC Plugin](https://mariadb.com/kb/en/parsec-authentication-plugin/)
- [📖 Pluggable Authentication](https://mariadb.com/kb/en/pluggable-authentication-overview/)

### Standards et spécifications

- [RFC 8032 - Edwards-Curve Digital Signature Algorithm (EdDSA)](https://www.rfc-editor.org/rfc/rfc8032)
- [FIPS 140-2/3 Security Requirements](https://csrc.nist.gov/publications/detail/fips/140/2/final)
- [PCI-DSS v4.0](https://www.pcisecuritystandards.org/)

---

## ➡️ Sections suivantes

Les sous-sections détailleront chaque plugin avec configurations complètes :

- **10.5.1** : mysql_native_password (legacy, migration)
- **10.5.2** : ed25519 (recommandé, configuration avancée)
- **10.5.3** : PAM (LDAP, AD, 2FA)
- **10.5.4** : GSSAPI (Kerberos, Active Directory)
- **10.5.5** : 🆕 PARSEC (HSM, PCI-DSS, FIPS)

**La section suivante (10.5.1)** détaillera **mysql_native_password** : pourquoi il est obsolète, comment migrer, et quand il peut encore être utilisé temporairement.

---


⏭️ [mysql_native_password](/10-securite-gestion-utilisateurs/05.1-mysql-native-password.md)

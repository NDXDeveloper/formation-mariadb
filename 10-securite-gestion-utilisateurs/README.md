🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10. Sécurité et Gestion des Utilisateurs

> **Niveau** : Avancé
> **Durée estimée** : 8-10 heures
> **Prérequis** : Chapitres 1-2, notions d'administration système Unix/Linux

> **Public cible** : Administrateurs bases de données (DBA), DevOps, SRE, Architectes sécurité

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Comprendre** le modèle de sécurité multi-niveaux de MariaDB et son architecture
- **Créer et gérer** des utilisateurs avec les bonnes pratiques de production
- **Maîtriser** le système de privilèges (GRANT/REVOKE) à tous les niveaux (global, base, table, colonne)
- **Implémenter** des rôles pour simplifier la gestion des droits
- **Configurer** différents plugins d'authentification selon vos besoins (native, ed25519, PAM, LDAP, GSSAPI, PARSEC)
- **Sécuriser** les connexions avec SSL/TLS (par défaut en 11.8 🆕)
- **Déployer** l'audit et le logging pour la conformité
- **Appliquer** les privilèges granulaires introduits en MariaDB 11.8 🆕

---

## Introduction

La sécurité d'une base de données est un pilier fondamental de toute infrastructure informatique moderne. Dans un contexte où les violations de données peuvent coûter des millions et impacter la réputation d'une entreprise, MariaDB propose un modèle de sécurité robuste, flexible et conforme aux standards de l'industrie.

### Pourquoi la sécurité MariaDB est critique ?

Les bases de données stockent souvent les actifs les plus précieux d'une organisation :
- **Données clients** (informations personnelles, RGPD/GDPR)
- **Données financières** (transactions, paiements)
- **Propriété intellectuelle** (brevets, code source, secrets commerciaux)
- **Données médicales** (HIPAA compliance)

Un seul utilisateur mal configuré, un privilège excessif ou une connexion non chiffrée peut exposer l'ensemble du système à des risques critiques.

### Architecture de sécurité MariaDB

MariaDB implémente une approche de **sécurité en profondeur** (defense in depth) avec plusieurs couches :

```
┌─────────────────────────────────────────────────────────┐
│         Couche Application (Client)                     │
│  - Connection pooling                                   │
│  - Prepared statements                                  │
│  - Input validation                                     │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│         Couche Réseau                                   │
│  - SSL/TLS encryption 🆕 (par défaut en 11.8)           │
│  - Firewall (bind-address, port filtering)              │
│  - VPN / Private networking                             │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│         Couche Authentification                         │
│  - Plugins: native, ed25519, PAM, LDAP, GSSAPI          │
│  - PARSEC 🆕 (Hardware Security Module)                 │
│  - Multi-factor authentication (via PAM)                │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│         Couche Autorisation (Privilèges)                │
│  - Système de privilèges granulaires                    │
│  - Rôles (RBAC - Role-Based Access Control)             │
│  - Privilèges: Global → Database → Table → Column       │
│  - Nouveaux privilèges granulaires 🆕 (11.8)            │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│         Couche Données                                  │
│  - Encryption at rest (tablespace encryption)           │
│  - Column-level encryption                              │
│  - Masking / Redaction (via vues)                       │
└─────────────────────────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│         Couche Audit et Conformité                      │
│  - Server Audit Plugin                                  │
│  - Query logging                                        │
│  - Connection logging                                   │
│  - SIEM integration                                     │
└─────────────────────────────────────────────────────────┘
```

### Le modèle de sécurité "Host + User"

MariaDB utilise un modèle unique où l'identité d'un utilisateur est composée de **deux parties** :

```sql
'username'@'hostname'
```

Cela signifie que `'admin'@'localhost'` et `'admin'@'10.0.0.5'` sont **deux utilisateurs différents** avec potentiellement des privilèges différents. Cette approche permet une granularité fine dans la gestion des accès.

**Exemples :**
- `'app_user'@'192.168.1.%'` : Utilisateur applicatif depuis le réseau 192.168.1.0/24
- `'dba'@'localhost'` : Administrateur avec accès uniquement local
- `'readonly'@'%'` : Utilisateur en lecture seule depuis n'importe où (dangereux en production !)

### Évolution du modèle de sécurité en MariaDB 11.8

MariaDB 11.8 LTS introduit plusieurs améliorations majeures :

| Fonctionnalité | Description | Impact |
|----------------|-------------|--------|
| **🆕 TLS par défaut** | Les connexions chiffrées sont activées par défaut | Sécurité renforcée out-of-the-box |
| **🆕 Plugin PARSEC** | Intégration avec Hardware Security Modules (HSM) | Conformité PCI-DSS, FIPS 140-2 |
| **🆕 Privilèges granulaires** | Nouveaux privilèges plus fins (ex: `SHOW_ROUTINE`, `BINLOG_REPLAY`) | Principe du moindre privilège renforcé |
| **🆕 Amélioration ed25519** | Performances optimisées pour l'authentification moderne | Adoption facilitée des algorithmes modernes |

---

## Vue d'ensemble du chapitre

Ce chapitre est structuré en 11 sections progressives :

### 🔐 Fondamentaux (Sections 10.1-10.4)
1. **Modèle de sécurité MariaDB** : Architecture et principes
2. **Gestion des utilisateurs** : CREATE/ALTER/DROP USER, best practices
3. **Système de privilèges** : GRANT/REVOKE à tous les niveaux
4. **Rôles** : Simplification de la gestion des droits avec RBAC

### 🔑 Authentification (Sections 10.5-10.6)
5. **Plugins d'authentification** : mysql_native_password, ed25519, PAM, LDAP, GSSAPI
6. **🆕 Plugin PARSEC** : Intégration HSM pour sécurité maximale

### 🔒 Chiffrement et Transport (Section 10.7)
7. **SSL/TLS** : Configuration serveur, certificats, TLS par défaut en 11.8

### 📊 Audit et Conformité (Sections 10.8-10.11)
8. **Audit et logging** : Server Audit Plugin, traçabilité
9. **Sécurité applicative** : Bonnes pratiques développement
10. **Password validation** : Politiques de mots de passe robustes
11. **🆕 Privilèges granulaires** : Nouveautés MariaDB 11.8

---

## Principes directeurs de sécurité

### 1. Principe du moindre privilège (Least Privilege)

**Définition** : Chaque utilisateur ne doit avoir que les privilèges strictement nécessaires pour accomplir sa tâche.

**Exemple incorrect ❌** :
```sql
-- TROP DE PRIVILÈGES : L'utilisateur applicatif a tous les droits
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';
```

**Exemple correct ✅** :
```sql
-- Privilèges minimaux pour une application web en lecture/écriture
GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO 'app_user'@'app_server_ip';
```

### 2. Défense en profondeur (Defense in Depth)

Ne jamais se reposer sur une seule couche de sécurité. Combiner :
- Chiffrement réseau (TLS)
- Authentification forte (ed25519, 2FA via PAM)
- Isolation réseau (VPC, firewalls)
- Audit actif
- Encryption at rest

### 3. Authentification forte

💡 **Recommandation** : Abandonner progressivement `mysql_native_password` au profit de :
- **ed25519** : Authentification moderne, rapide, sécurisée
- **PAM** : Intégration avec SSO, LDAP/AD, 2FA
- **PARSEC** 🆕 : Pour environnements hautement sécurisés (finance, santé)

### 4. Séparation des environnements

Les utilisateurs de développement ne doivent **jamais** avoir accès à la production :

```sql
-- Environnement de développement
CREATE USER 'dev_alice'@'dev_network' IDENTIFIED BY 'dev_password';
GRANT ALL ON dev_db.* TO 'dev_alice'@'dev_network';

-- Environnement de production (utilisateur différent, hôte différent)
CREATE USER 'prod_app'@'prod_app_server' IDENTIFIED VIA ed25519 USING PASSWORD('strong_prod_password');
GRANT SELECT, INSERT, UPDATE ON prod_db.* TO 'prod_app'@'prod_app_server';
```

### 5. Rotation des credentials

Les mots de passe et certificats doivent être renouvelés régulièrement :

```sql
-- Rotation de mot de passe avec ALTER USER
ALTER USER 'app_user'@'localhost' IDENTIFIED BY 'new_strong_password_2025';

-- Expiration automatique des mots de passe (11.8+)
ALTER USER 'temp_contractor'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
```

---

## Cas d'usage typiques

### 🏢 Entreprise avec AD/LDAP

**Contexte** : Intégration avec Active Directory pour centraliser l'authentification.

**Solution** :
```sql
-- Installation du plugin PAM
INSTALL SONAME 'auth_pam';

-- Création d'utilisateurs qui s'authentifient via AD
CREATE USER 'john.doe'@'%' IDENTIFIED VIA pam USING 'mariadb';
GRANT SELECT ON analytics.* TO 'john.doe'@'%';
```

**Configuration PAM** (`/etc/pam.d/mariadb`) :
```
auth    required   pam_ldap.so
account required   pam_ldap.so
```

### 🏦 Secteur financier (Compliance PCI-DSS)

**Contexte** : Besoin de chiffrement matériel et d'audit strict.

**Solution** :
```sql
-- Activation du plugin PARSEC 🆕 pour HSM
INSTALL SONAME 'auth_parsec';

-- Utilisateur avec authentification HSM
CREATE USER 'payment_processor'@'payment_gateway'
  IDENTIFIED VIA parsec USING 'key_id_from_hsm';

-- Privilèges minimaux sur données de paiement
GRANT SELECT, INSERT ON payments.transactions TO 'payment_processor'@'payment_gateway';
GRANT UPDATE(status) ON payments.transactions TO 'payment_processor'@'payment_gateway';

-- Audit obligatoire
INSTALL SONAME 'server_audit';
SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_events = 'CONNECT,QUERY,TABLE';
```

### 🌐 Application SaaS multi-tenant

**Contexte** : Isolation des données entre tenants, accès différenciés.

**Solution avec rôles** :
```sql
-- Création de rôles par niveau d'accès
CREATE ROLE tenant_readonly;
CREATE ROLE tenant_readwrite;
CREATE ROLE tenant_admin;

GRANT SELECT ON tenant_db.* TO tenant_readonly;
GRANT SELECT, INSERT, UPDATE ON tenant_db.* TO tenant_readwrite;
GRANT ALL ON tenant_db.* TO tenant_admin;

-- Attribution des rôles aux utilisateurs
CREATE USER 'tenant1_user'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('...');
GRANT tenant_readwrite TO 'tenant1_user'@'%';
SET DEFAULT ROLE tenant_readwrite FOR 'tenant1_user'@'%';
```

### 🔬 Environnement DevOps/CI-CD

**Contexte** : Automatisation avec outils (Terraform, Ansible) nécessitant accès programmatique.

**Solution** :
```sql
-- Utilisateur pour Terraform (IaC)
CREATE USER 'terraform'@'ci_server' IDENTIFIED BY 'generated_secret_from_vault';
GRANT CREATE, ALTER, DROP ON infrastructure_db.* TO 'terraform'@'ci_server';
-- Pas de DELETE pour éviter suppressions accidentelles

-- Utilisateur pour backups automatisés
CREATE USER 'backup_agent'@'localhost' IDENTIFIED VIA ed25519 USING PASSWORD('...');
GRANT SELECT, LOCK TABLES, RELOAD, REPLICATION CLIENT ON *.* TO 'backup_agent'@'localhost';
```

---

## 🆕 Nouveautés MariaDB 11.8 LTS

### TLS par défaut

**Avant 11.8** : SSL/TLS devait être configuré manuellement.

**Depuis 11.8** : TLS est activé par défaut si des certificats sont présents.

```sql
-- Vérification du statut TLS
SHOW VARIABLES LIKE 'have_ssl';
-- Résultat: YES (par défaut si certificats auto-générés présents)

-- Forcer TLS pour un utilisateur
CREATE USER 'secure_user'@'%'
  IDENTIFIED BY 'password'
  REQUIRE SSL;
```

### Plugin PARSEC (Hardware Security Module)

Permet d'utiliser des clés cryptographiques stockées dans un HSM conforme FIPS 140-2.

```sql
INSTALL SONAME 'auth_parsec';

CREATE USER 'hsm_user'@'localhost'
  IDENTIFIED VIA parsec USING 'parsec://key_provider/key_name';
```

**Cas d'usage** :
- Conformité réglementaire (PCI-DSS, HIPAA)
- Secteur bancaire et financier
- Cloud souverain et gouvernemental

### Privilèges granulaires

MariaDB 11.8 introduit de nouveaux privilèges pour un contrôle plus fin :

| Privilège | Description | Cas d'usage |
|-----------|-------------|-------------|
| `SHOW_ROUTINE` | Voir les routines sans les exécuter | Audit, documentation |
| `BINLOG_REPLAY` | Rejouer des binlogs (PITR) | DBA backup/restore |
| `BINLOG_ADMIN` | Administration complète des binlogs | DBA senior |
| `CONNECTION_ADMIN` | Gérer les connexions (kill, etc.) | Support niveau 2 |

**Exemple** :
```sql
-- Utilisateur d'audit qui peut voir les procédures stockées
GRANT SHOW_ROUTINE ON production.* TO 'auditor'@'%';

-- DBA backup qui peut rejouer les binlogs
GRANT BINLOG_REPLAY ON *.* TO 'backup_dba'@'localhost';
```

---

## Pièges courants et erreurs à éviter

### ❌ Erreur 1 : Utilisateur avec '%' et mot de passe faible

```sql
-- DANGEREUX : Accessible depuis n'importe où avec mot de passe faible
CREATE USER 'admin'@'%' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%';
```

**Pourquoi c'est dangereux ?**
- Exposition sur Internet (si port 3306 ouvert)
- Force brute triviale
- Violation du principe du moindre privilège

**✅ Correction** :
```sql
-- Limiter l'accès réseau
CREATE USER 'admin'@'10.0.0.0/255.255.255.0'
  IDENTIFIED VIA ed25519 USING PASSWORD('$trong_P@ssw0rd_2025!')
  REQUIRE SSL;
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'10.0.0.0/255.255.255.0'
  WITH GRANT OPTION;
```

### ❌ Erreur 2 : Oublier de restreindre les utilisateurs anonymes

Certaines installations créent des utilisateurs anonymes par défaut :

```sql
-- Vérifier la présence d'utilisateurs anonymes
SELECT user, host FROM mysql.user WHERE user = '';

-- SUPPRIMER tous les utilisateurs anonymes
DROP USER ''@'localhost';
DROP USER ''@'hostname';
```

### ❌ Erreur 3 : Privilèges excessifs pour les applications

```sql
-- TROP PERMISSIF : L'application peut modifier la structure
GRANT ALL ON app_db.* TO 'app_user'@'app_server';
```

**✅ Correction** :
```sql
-- JUSTE NÉCESSAIRE : CRUD sans DDL
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'app_server';

-- Si besoin d'exécuter des stored procedures spécifiques
GRANT EXECUTE ON PROCEDURE app_db.calculate_price TO 'app_user'@'app_server';
```

### ❌ Erreur 4 : Ne pas activer l'audit en production

Sans audit, impossible de :
- Détecter une intrusion
- Investiguer un incident de sécurité
- Prouver la conformité (RGPD, SOC2)

**✅ Correction** :
```sql
INSTALL SONAME 'server_audit';
SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_file_path = '/var/log/mysql/audit.log';
SET GLOBAL server_audit_events = 'CONNECT,QUERY_DDL,QUERY_DML';
SET GLOBAL server_audit_incl_users = 'admin,app_user';
```

---

## Checklist de sécurité pour la production

### ✅ Avant le déploiement

- [ ] **Utilisateurs par défaut supprimés** (root, anonymous)
- [ ] **Mots de passe forts** pour tous les comptes (min. 16 caractères, complexité)
- [ ] **Plugin d'authentification moderne** (ed25519 ou PAM recommandé)
- [ ] **TLS activé** et **obligatoire** pour les connexions distantes
- [ ] **Bind-address** configuré (pas de 0.0.0.0 si possible)
- [ ] **Firewall** configuré (port 3306 fermé en externe)
- [ ] **Privilèges minimaux** appliqués (principe du moindre privilège)
- [ ] **Audit activé** avec rotation des logs

### ✅ Monitoring continu

- [ ] **Connexions échouées** surveillées (alert si > seuil)
- [ ] **Tentatives de privilege escalation** détectées
- [ ] **Requêtes DDL non autorisées** alertées
- [ ] **Rotation des credentials** planifiée (90-180 jours)
- [ ] **Revue trimestrielle** des privilèges utilisateurs

### ✅ Conformité

- [ ] **RGPD/GDPR** : Traçabilité des accès aux données personnelles
- [ ] **PCI-DSS** : Chiffrement, HSM (PARSEC), audit
- [ ] **HIPAA** : Encryption at rest/in transit, access logs
- [ ] **SOC2** : Contrôles d'accès documentés, audit trail

---

## 💡 Bonnes pratiques de production

### 1. Séparation des rôles (RBAC)

Créer des rôles par fonction métier, pas par personne :

```sql
CREATE ROLE app_backend_readonly;
CREATE ROLE app_backend_readwrite;
CREATE ROLE dba_operations;
CREATE ROLE dba_emergency;

-- Attribution granulaire
GRANT SELECT ON production.* TO app_backend_readonly;
GRANT SELECT, INSERT, UPDATE ON production.* TO app_backend_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER ON *.* TO dba_operations;
GRANT ALL ON *.* TO dba_emergency WITH GRANT OPTION;
```

### 2. Utilisation de secrets managers

Ne **jamais** stocker les mots de passe en clair dans le code ou la configuration.

**Intégrations recommandées** :
- **HashiCorp Vault** : Rotation automatique, secrets dynamiques
- **AWS Secrets Manager** : Intégration native avec RDS
- **Azure Key Vault** : Pour Azure Database for MariaDB
- **Kubernetes Secrets** : Pour environnements conteneurisés

### 3. Audit centralisé (SIEM)

Envoyer les logs d'audit vers un SIEM pour corrélation :

```bash
# Configuration rsyslog pour envoyer vers Splunk/ELK
# /etc/rsyslog.d/mariadb-audit.conf
module(load="imfile")
input(type="imfile"
      File="/var/log/mysql/audit.log"
      Tag="mariadb-audit"
      Severity="info"
      Facility="local3")

# Forward to SIEM
*.* @@siem-server:514
```

### 4. Principe de révocation proactive

Supprimer immédiatement les accès des employés qui quittent l'entreprise :

```sql
-- Désactivation immédiate (sans suppression)
ALTER USER 'departed_employee'@'%' ACCOUNT LOCK;

-- Revue des privilèges hérités
SHOW GRANTS FOR 'departed_employee'@'%';

-- Suppression après délai de rétention
DROP USER 'departed_employee'@'%';
```

---

## ✅ Points clés à retenir

- **MariaDB implémente un modèle de sécurité multi-couches** (réseau, authentification, autorisation, chiffrement, audit)
- **L'identité utilisateur est composée de `'user'@'host'`**, permettant une granularité fine
- **Le principe du moindre privilège est fondamental** : ne donner que les droits strictement nécessaires
- **MariaDB 11.8 active TLS par défaut** 🆕, renforçant la sécurité out-of-the-box
- **Les rôles (RBAC) simplifient la gestion** des privilèges dans les organisations complexes
- **L'authentification moderne (ed25519, PAM, PARSEC 🆕) doit remplacer mysql_native_password** progressivement
- **L'audit est obligatoire en production** pour la traçabilité et la conformité
- **La sécurité est un processus continu**, pas une configuration ponctuelle

---

## 🔗 Ressources et références

### Documentation officielle MariaDB 11.8

- [📖 Security Overview](https://mariadb.com/kb/en/security/)
- [📖 User Account Management](https://mariadb.com/kb/en/user-account-management/)
- [📖 Grant System](https://mariadb.com/kb/en/grant/)
- [📖 Authentication Plugins](https://mariadb.com/kb/en/authentication-plugins/)
- [📖 🆕 PARSEC Plugin](https://mariadb.com/kb/en/parsec-authentication-plugin/)
- [📖 SSL/TLS Configuration](https://mariadb.com/kb/en/secure-connections-overview/)
- [📖 Server Audit Plugin](https://mariadb.com/kb/en/mariadb-audit-plugin/)

### Standards et conformité

- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CIS MariaDB Benchmark](https://www.cisecurity.org/)
- [OWASP Database Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)

### Outils de sécurité

- [mysql_secure_installation](https://mariadb.com/kb/en/mysql_secure_installation/)
- [pt-show-grants (Percona Toolkit)](https://docs.percona.com/percona-toolkit/pt-show-grants.html)
- [HashiCorp Vault Database Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/databases/mysql-maria)

---

## ➡️ Sections suivantes

Les 11 sections de ce chapitre détailleront progressivement chaque aspect de la sécurité MariaDB :

- **10.1** : Modèle de sécurité MariaDB (architecture, tables système)
- **10.2** : Création et gestion des utilisateurs (CREATE/ALTER/DROP USER)
- **10.3** : Système de privilèges (GRANT/REVOKE, niveaux de privilèges)
- **10.4** : Rôles (RBAC, CREATE ROLE, DEFAULT ROLE)
- **10.5** : Plugins d'authentification (native, ed25519, PAM, LDAP, GSSAPI)
- **10.6** : 🆕 Plugin PARSEC (HSM, conformité)
- **10.7** : Chiffrement SSL/TLS (configuration, certificats, TLS par défaut 11.8)
- **10.8** : Audit et logging (Server Audit Plugin, compliance)
- **10.9** : Sécurité applicative (prepared statements, validation)
- **10.10** : Password validation (politiques, cracklib)
- **10.11** : 🆕 Privilèges granulaires (nouveautés 11.8)

**La section suivante (10.1)** entrera dans le détail du **modèle de sécurité MariaDB**, avec l'architecture des tables système (`mysql.user`, `mysql.db`, etc.) et les mécanismes d'authentification/autorisation.

---


⏭️ [Modèle de sécurité MariaDB](/10-securite-gestion-utilisateurs/01-modele-securite.md)

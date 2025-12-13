🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7 Chiffrement des connexions (SSL/TLS) 🔄

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 10.1-10.6, notions de PKI et cryptographie

> **Nouveauté** : TLS activé par défaut dans MariaDB 11.8 🆕

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'architecture SSL/TLS et son rôle dans la sécurité MariaDB
- **Maîtriser** les différences entre TLS 1.2, 1.3 et les versions antérieures
- **Configurer** le chiffrement des connexions serveur et client
- **Gérer** les certificats SSL (génération, renouvellement, révocation)
- **Appliquer** les nouveautés MariaDB 11.8 (TLS par défaut) 🆕
- **Choisir** les cipher suites appropriées selon le contexte
- **Forcer** le chiffrement pour utilisateurs spécifiques ou globalement
- **Diagnostiquer** les problèmes de connexions SSL/TLS
- **Implémenter** des architectures sécurisées en production

---

## Introduction

Le **chiffrement des connexions** est une couche de sécurité fondamentale qui protège les données **en transit** entre les clients et le serveur MariaDB. Sans chiffrement, toutes les informations échangées (requêtes SQL, résultats, mots de passe) circulent **en clair** sur le réseau et peuvent être interceptées.

### Pourquoi le chiffrement est critique ?

**Scénario sans SSL/TLS** :

```
Client (Application web)                    Serveur MariaDB
       │                                           │
       │  SELECT * FROM users                      │
       │  WHERE email='alice@example.com'          │
       │──────────────────────────────────────────>│
       │                                           │
       │  Résultats (données personnelles)         │
       │<──────────────────────────────────────────│

       ⚠️ PROBLÈME: Tout circule EN CLAIR sur le réseau

┌──────────────────────────────────────────────────────────┐
│  Attaquant sur le réseau (MITM - Man in the Middle)      │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Capture réseau (tcpdump, Wireshark)                │  │
│  │ → Voit toutes les requêtes SQL                     │  │
│  │ → Voit tous les résultats                          │  │
│  │ → Peut extraire les mots de passe                  │  │
│  │ → Peut modifier les requêtes (injection)           │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Avec SSL/TLS** :

```
Client (Application web)                    Serveur MariaDB
       │                                           │
       │  1. Handshake TLS (échange de clés)       │
       │<─────────────────────────────────────────>│
       │                                           │
       │  2. Canal chiffré établi                  │
       │━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│
       │                                           │
       │  3. Requêtes CHIFFRÉES                    │
       │══════════════════════════════════════════>│
       │                                           │
       │  4. Résultats CHIFFRÉS                    │
       │<══════════════════════════════════════════│

       ✓ Protection: Tout est chiffré (AES-256, ChaCha20)

┌──────────────────────────────────────────────────────────┐
│  Attaquant sur le réseau                                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Capture réseau                                     │  │
│  │ → Voit uniquement des données chiffrées            │  │
│  │ → Ne peut pas lire les requêtes                    │  │
│  │ → Ne peut pas lire les résultats                   │  │
│  │ → Ne peut pas modifier sans détection              │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Menaces protégées par SSL/TLS

| Menace | Sans SSL/TLS | Avec SSL/TLS |
|--------|-------------|--------------|
| **Écoute passive (sniffing)** | ❌ Données lisibles | ✅ Données chiffrées |
| **Vol de credentials** | ❌ Mots de passe en clair | ✅ Authentification sécurisée |
| **Man-in-the-Middle (MITM)** | ❌ Attaque triviale | ✅ Certificats valident l'identité |
| **Modification des données** | ❌ Injection possible | ✅ Intégrité garantie (HMAC) |
| **Rejeu d'attaques** | ❌ Possible | ✅ Nonces et timestamps |

### Conformité réglementaire

De nombreuses réglementations **exigent** le chiffrement des données en transit :

| Réglementation | Exigence | MariaDB avec SSL/TLS |
|----------------|----------|---------------------|
| **PCI-DSS 4.0** | Chiffrement obligatoire des données de carte bancaire | ✅ TLS 1.2+ requis |
| **RGPD/GDPR** | Protection des données personnelles | ✅ Chiffrement recommandé |
| **HIPAA** | Protection des données de santé | ✅ Chiffrement obligatoire |
| **SOC 2** | Contrôles de sécurité | ✅ TLS dans les contrôles |
| **ISO 27001** | Sécurité de l'information | ✅ Chiffrement dans les standards |

💡 **PCI-DSS 4.0** (depuis mars 2024) interdit explicitement SSL 3.0, TLS 1.0 et TLS 1.1. Seuls **TLS 1.2 et TLS 1.3** sont autorisés.

---

## Architecture SSL/TLS dans MariaDB

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                    Connexion client                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  ÉTAPE 1: TCP Handshake (couche transport)                   │
│  Client ←→ Serveur: Établissement connexion TCP              │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  ÉTAPE 2: TLS Handshake (négociation SSL/TLS)                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 1. ClientHello                                         │  │
│  │    → Versions TLS supportées (1.2, 1.3)                │  │
│  │    → Cipher suites proposées                           │  │
│  │    → Extensions (SNI, ALPN)                            │  │
│  │                                                        │  │
│  │ 2. ServerHello                                         │  │
│  │    ← Version TLS choisie                               │  │
│  │    ← Cipher suite sélectionnée                         │  │
│  │    ← Certificat serveur (X.509)                        │  │
│  │                                                        │  │
│  │ 3. Certificate Verify (optionnel)                      │  │
│  │    → Client vérifie le certificat serveur              │  │
│  │    → Chaîne de certification (CA)                      │  │
│  │                                                        │  │
│  │ 4. Client Certificate (optionnel)                      │  │
│  │    → Authentification mutuelle (mTLS)                  │  │
│  │                                                        │  │
│  │ 5. Key Exchange                                        │  │
│  │    ↔ Échange de clés (ECDHE, DHE)                      │  │
│  │    ↔ Génération clés de session                        │  │
│  │                                                        │  │
│  │ 6. Finished                                            │  │
│  │    ↔ Confirmation handshake réussi                     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  ÉTAPE 3: Canal chiffré établi                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Algorithme de chiffrement: AES-256-GCM, ChaCha20       │  │
│  │ Intégrité: HMAC-SHA256, Poly1305                       │  │
│  │ Perfect Forward Secrecy: ECDHE, DHE                    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  ÉTAPE 4: Communication applicative (protocole MariaDB)      │
│  Toutes les données sont chiffrées et authentifiées          │
└──────────────────────────────────────────────────────────────┘
```

### Composants SSL/TLS

**Côté serveur** :

1. **Certificat serveur** (`server-cert.pem`)
   - Identité du serveur
   - Clé publique du serveur
   - Signature par une autorité de certification (CA)

2. **Clé privée serveur** (`server-key.pem`)
   - Clé privée pour déchiffrer
   - **STRICTEMENT CONFIDENTIELLE**
   - Permissions 400 ou 600

3. **Certificat CA** (`ca-cert.pem`)
   - Autorité de certification racine
   - Utilisée pour valider les certificats clients (mTLS)

**Côté client** :

1. **Certificat CA** (`ca-cert.pem`)
   - Valide le certificat du serveur
   - Chaîne de confiance

2. **Certificat client** (`client-cert.pem`) - optionnel
   - Pour authentification mutuelle (mTLS)

3. **Clé privée client** (`client-key.pem`) - optionnel
   - Pour mTLS

---

## TLS 1.2 vs TLS 1.3

### Tableau comparatif

| Aspect | TLS 1.2 | TLS 1.3 |
|--------|---------|---------|
| **Année** | 2008 | 2018 (RFC 8446) |
| **Handshake** | 2 RTT (Round-Trip Time) | **1 RTT** ⚡ |
| **0-RTT** | Non | **Oui** (resumption) |
| **Cipher suites** | 37 suites (beaucoup obsolètes) | **5 suites modernes** |
| **Perfect Forward Secrecy** | Optionnel | **Obligatoire** |
| **Algorithmes faibles** | RSA key exchange (vulnérable) | **Supprimés** |
| **Performance** | Bonne | **Meilleure** (25-40% plus rapide) |
| **Sécurité** | Bonne (si bien configuré) | **Excellente** (par défaut) |
| **Support MariaDB** | 10.0+ | 10.4+ |

### TLS 1.2 Handshake (2 RTT)

```
Client                                 Serveur
  │                                       │
  │  1. ClientHello                       │
  │─────────────────────────────────────→ │  ← RTT 1
  │                                       │
  │  2. ServerHello, Certificate,         │
  │     ServerKeyExchange, ServerDone     │
  │ ←─────────────────────────────────────│
  │                                       │
  │  3. ClientKeyExchange, ChangeCipher,  │
  │     Finished                          │
  │─────────────────────────────────────→ │  ← RTT 2
  │                                       │
  │  4. ChangeCipher, Finished            │
  │ ←─────────────────────────────────────│
  │                                       │
  │  5. Application Data ✓                │
  │━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│

Temps total: ~100-200ms (selon latence réseau)
```

### TLS 1.3 Handshake (1 RTT) ⚡

```
Client                                 Serveur
  │                                       │
  │  1. ClientHello (+ Key Share)         │
  │─────────────────────────────────────→ │  ← RTT 1
  │                                       │
  │  2. ServerHello, Certificate,         │
  │     Finished (+ encrypted data!)      │
  │ ←─────────────────────────────────────│
  │                                       │
  │  3. Application Data ✓                │
  │━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│

Temps total: ~50-100ms (2x plus rapide!)
```

**Avantages TLS 1.3** :

1. ⚡ **50% plus rapide** : 1 RTT au lieu de 2
2. 🔐 **Plus sécurisé** : Suppression des algorithmes faibles
3. ✅ **PFS obligatoire** : Perfect Forward Secrecy par défaut
4. 🚀 **0-RTT resumption** : Reconnexion instantanée

💡 **Recommandation 2025** : Utiliser **TLS 1.3 uniquement** si possible (désactiver TLS 1.2 pour sécurité maximale).

### Cipher suites

#### TLS 1.2 - Cipher suites recommandées

```
Format:
TLS_<Key Exchange>_WITH_<Encryption>_<MAC>

Exemples:
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    └─ ECDHE: Échange de clés (Perfect Forward Secrecy)
       └─ RSA: Signature certificat
          └─ AES-256-GCM: Chiffrement
             └─ SHA384: Intégrité
```

**Liste blanche TLS 1.2** (PCI-DSS compliant) :

```
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
```

⚠️ **À éviter** (vulnérables ou obsolètes) :

```
❌ TLS_RSA_*                    (Pas de PFS)
❌ *_CBC_*                      (Vulnérable BEAST, Lucky13)
❌ *_RC4_*                      (RC4 cassé)
❌ *_3DES_*                     (Triple DES obsolète)
❌ *_MD5                        (MD5 cassé)
❌ *_EXPORT_*                   (Export grade, faible)
❌ *_NULL_*                     (Pas de chiffrement!)
```

#### TLS 1.3 - Cipher suites (simplifiées)

TLS 1.3 a **seulement 5 cipher suites** (toutes sécurisées) :

```
TLS_AES_256_GCM_SHA384           (Recommandé, AES matériel)
TLS_AES_128_GCM_SHA256           (Performance)
TLS_CHACHA20_POLY1305_SHA256     (Mobile, ARM)
TLS_AES_128_CCM_SHA256           (IoT, faible RAM)
TLS_AES_128_CCM_8_SHA256         (IoT, très faible RAM)
```

💡 **TLS 1.3** élimine la complexité : tous les ciphers sont sécurisés, le choix est basé sur la performance.

---

## 🆕 Nouveauté MariaDB 11.8 : TLS par défaut

### Avant MariaDB 11.8

**Configuration manuelle obligatoire** :

```ini
# /etc/my.cnf.d/server.cnf - AVANT 11.8
[mysqld]
# TLS désactivé par défaut
# Fallait configurer manuellement:
ssl-ca=/path/to/ca-cert.pem
ssl-cert=/path/to/server-cert.pem
ssl-key=/path/to/server-key.pem
```

**Problème** : Beaucoup d'installations sans TLS (configuration oubliée).

### Depuis MariaDB 11.8 🆕

**TLS activé automatiquement** si des certificats sont présents :

```
Installation MariaDB 11.8
        ↓
Génération automatique de certificats auto-signés
        ↓
TLS activé par défaut ✓
```

**Certificats auto-générés** :

```bash
# MariaDB 11.8 génère automatiquement:
ls -la /var/lib/mysql/*.pem

# -rw-r----- 1 mysql mysql ca-cert.pem          # CA root
# -rw-r----- 1 mysql mysql ca-key.pem           # CA private key
# -rw-r----- 1 mysql mysql server-cert.pem      # Server certificate
# -rw------- 1 mysql mysql server-key.pem       # Server private key
# -rw-r----- 1 mysql mysql client-cert.pem      # Client certificate
# -rw------- 1 mysql mysql client-key.pem       # Client private key
```

**Vérification** :

```sql
-- Connexion au serveur MariaDB 11.8
mariadb -u root -p

-- Vérifier que TLS est actif
SHOW VARIABLES LIKE 'have_ssl';
/*
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| have_ssl      | YES   |  ← TLS activé par défaut! 🆕
+---------------+-------+
*/

-- Voir les chemins des certificats
SHOW VARIABLES LIKE 'ssl%';
/*
+---------------------+------------------------------+
| Variable_name       | Value                        |
+---------------------+------------------------------+
| ssl_ca              | /var/lib/mysql/ca-cert.pem   |
| ssl_cert            | /var/lib/mysql/server-cert.pem|
| ssl_key             | /var/lib/mysql/server-key.pem|
+---------------------+------------------------------+
*/
```

### Impact sur les connexions

**Connexion client** :

```bash
# MariaDB 11.8 - TLS automatique
mariadb -u user -p -h db.example.com

# Vérifier que la connexion est chiffrée
MariaDB> SHOW STATUS LIKE 'Ssl_cipher';
/*
+---------------+--------------------+
| Variable_name | Value              |
+---------------+--------------------+
| Ssl_cipher    | TLS_AES_256_GCM... | ← Connexion chiffrée ✓
+---------------+--------------------+
*/
```

**Si pas de TLS souhaité** (dev local uniquement) :

```bash
# Forcer connexion non-chiffrée (déconseillé)
mariadb -u user -p --ssl=0

# Ou côté serveur (désactiver TLS)
# /etc/my.cnf.d/server.cnf
[mysqld]
skip-ssl
```

### Migration vers MariaDB 11.8

**Scénario 1 : Certificats custom déjà configurés**

```ini
# Avant (10.x, 11.x < 11.8)
[mysqld]
ssl-ca=/etc/pki/mariadb/ca-cert.pem
ssl-cert=/etc/pki/mariadb/server-cert.pem
ssl-key=/etc/pki/mariadb/server-key.pem

# Après (11.8+)
# Même configuration, fonctionne toujours
# Mais si retirée, TLS reste actif avec auto-signed
```

**Scénario 2 : Pas de certificats (avant 11.8)**

```
Avant upgrade:
  TLS désactivé (have_ssl = NO)

Après upgrade vers 11.8:
  Certificats auto-générés
  TLS activé (have_ssl = YES) ✓

⚠️ Attention: Applications legacy sans support TLS
→ Tester avant upgrade
```

---

## Types de certificats

### Certificats auto-signés (self-signed)

**Avantages** :
- ✅ Gratuit
- ✅ Génération instantanée
- ✅ Suffisant pour chiffrement

**Inconvénients** :
- ❌ Pas de validation d'identité
- ❌ Warning navigateur/client
- ❌ Vulnérable MITM sans validation manuelle

**Cas d'usage** :
- Développement local
- Environnements internes (VPN)
- Tests

**Génération** :

```bash
# OpenSSL - Certificat auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout server-key.pem \
  -out server-cert.pem \
  -subj "/C=FR/ST=Paris/L=Paris/O=MyCompany/CN=db.example.com"

# Générer CA + certificats serveur/client
mysql_ssl_rsa_setup --datadir=/var/lib/mysql
```

### Certificats signés par CA (Certificate Authority)

**Avantages** :
- ✅ Validation d'identité
- ✅ Confiance établie
- ✅ Pas de warning client
- ✅ Protection MITM

**CA publiques** (pour domaines publics) :
- **Let's Encrypt** : Gratuit, automatisé
- **DigiCert** : Payant, support entreprise
- **GlobalSign** : Payant, validation étendue

**CA privée** (pour domaines internes) :
- OpenSSL CA
- Microsoft AD CS
- HashiCorp Vault PKI

**Cas d'usage** :
- Production (toujours)
- Exposition Internet
- Conformité réglementaire

### Let's Encrypt (gratuit)

**Installation Certbot** :

```bash
# RHEL/CentOS
sudo dnf install certbot

# Debian/Ubuntu
sudo apt install certbot

# Obtenir un certificat
sudo certbot certonly --standalone -d db.example.com

# Certificats générés dans:
# /etc/letsencrypt/live/db.example.com/
#   - cert.pem        (certificat serveur)
#   - privkey.pem     (clé privée)
#   - chain.pem       (chaîne CA)
#   - fullchain.pem   (cert + chain)
```

**Configuration MariaDB** :

```ini
[mysqld]
ssl-ca=/etc/letsencrypt/live/db.example.com/chain.pem
ssl-cert=/etc/letsencrypt/live/db.example.com/cert.pem
ssl-key=/etc/letsencrypt/live/db.example.com/privkey.pem
```

**Renouvellement automatique** :

```bash
# Cron job (Let's Encrypt expire tous les 90 jours)
0 3 * * * certbot renew --post-hook "systemctl restart mariadb"
```

### CA privée (entreprise)

**Création d'une CA interne** :

```bash
#!/bin/bash
# create_mariadb_ca.sh

# 1. Générer CA root
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 3650 -key ca-key.pem -out ca-cert.pem \
  -subj "/C=FR/O=MyCompany/CN=MyCompany Root CA"

# 2. Générer clé serveur
openssl genrsa -out server-key.pem 4096

# 3. CSR serveur (Certificate Signing Request)
openssl req -new -key server-key.pem -out server-csr.pem \
  -subj "/C=FR/O=MyCompany/CN=db.example.com"

# 4. Signer le certificat serveur avec CA
openssl x509 -req -days 365 -in server-csr.pem \
  -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem

# 5. Vérifier
openssl verify -CAfile ca-cert.pem server-cert.pem
```

---

## Niveaux de sécurité TLS

MariaDB supporte **4 niveaux** de sécurité TLS pour les utilisateurs.

### Niveau 1 : SSL optionnel (par défaut)

```sql
-- Utilisateur peut se connecter avec ou sans SSL
CREATE USER 'app_user'@'%' IDENTIFIED BY 'password';
-- Pas de REQUIRE

-- Connexion sans SSL: ✓ Autorisée
-- Connexion avec SSL: ✓ Autorisée
```

**Cas d'usage** : Compatibilité maximale (dev, applications legacy).

### Niveau 2 : REQUIRE SSL

```sql
-- Utilisateur DOIT utiliser SSL (n'importe quel certificat)
CREATE USER 'secure_app'@'%'
  IDENTIFIED BY 'password'
  REQUIRE SSL;

-- Connexion sans SSL: ✗ Refusée
-- Connexion avec SSL: ✓ Autorisée (même auto-signé)
```

**Cas d'usage** : Production, protection minimale.

### Niveau 3 : REQUIRE X509

```sql
-- Utilisateur DOIT présenter un certificat client valide
CREATE USER 'mtls_user'@'%'
  IDENTIFIED BY 'password'
  REQUIRE X509;

-- Connexion sans SSL: ✗ Refusée
-- Connexion SSL sans cert client: ✗ Refusée
-- Connexion SSL avec cert client: ✓ Autorisée
```

**Cas d'usage** : Authentification mutuelle (mTLS), haute sécurité.

### Niveau 4 : REQUIRE ISSUER / SUBJECT (maximum)

```sql
-- Certificat client avec émetteur spécifique
CREATE USER 'bank_app'@'%'
  IDENTIFIED BY 'password'
  REQUIRE ISSUER '/C=FR/O=MyBank/CN=Internal CA'
    AND SUBJECT '/C=FR/O=MyBank/CN=payment-service';

-- Connexion uniquement si:
-- ✓ SSL activé
-- ✓ Certificat client valide
-- ✓ Issuer correspond
-- ✓ Subject correspond
```

**Cas d'usage** : Secteur bancaire, conformité stricte.

### Niveau 5 : REQUIRE CIPHER (très spécifique)

```sql
-- Forcer un cipher précis (rare)
CREATE USER 'gov_user'@'%'
  IDENTIFIED BY 'password'
  REQUIRE CIPHER 'TLS_AES_256_GCM_SHA384';

-- Connexion uniquement avec ce cipher exact
```

**Cas d'usage** : Gouvernement, défense (FIPS 140-2).

---

## Configuration des cipher suites

### Vérifier les ciphers disponibles

```sql
-- Voir les ciphers supportés par le serveur
SHOW VARIABLES LIKE 'tls_version';
/*
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| tls_version   | TLSv1.2,  |
|               | TLSv1.3   |
+---------------+-----------+
*/

-- Voir les cipher suites configurées
SHOW VARIABLES LIKE 'ssl_cipher';
```

### Configuration côté serveur

**TLS 1.3 uniquement (recommandé 2025)** :

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
# Désactiver TLS 1.0, 1.1, 1.2 (TLS 1.3 seulement)
tls_version = TLSv1.3

# Cipher suites TLS 1.3 (ordre de préférence)
ssl_cipher = TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
```

**TLS 1.2 + 1.3 (compatibilité)** :

```ini
[mysqld]
tls_version = TLSv1.2,TLSv1.3

# Ciphers TLS 1.2 (liste blanche PCI-DSS)
ssl_cipher = ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305

# Ciphers TLS 1.3 (automatique, pas besoin de spécifier)
```

**PCI-DSS compliant** :

```ini
[mysqld]
# PCI-DSS: TLS 1.2 minimum (TLS 1.3 recommandé)
tls_version = TLSv1.2,TLSv1.3

# Exclure les ciphers faibles
ssl_cipher = !aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!3DES:!CBC

# Liste blanche explicite (plus sûr)
ssl_cipher = ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
```

### Configuration côté client

```bash
# Forcer TLS 1.3 et cipher spécifique
mariadb -u user -p -h db.example.com \
  --ssl-mode=REQUIRED \
  --tls-version=TLSv1.3 \
  --ssl-cipher=TLS_AES_256_GCM_SHA384

# Connexion avec certificat client (mTLS)
mariadb -u user -p -h db.example.com \
  --ssl-ca=/path/to/ca-cert.pem \
  --ssl-cert=/path/to/client-cert.pem \
  --ssl-key=/path/to/client-key.pem
```

---

## Vérification et diagnostic

### Vérifier l'état SSL du serveur

```sql
-- Variables SSL
SHOW VARIABLES LIKE 'have_ssl';
SHOW VARIABLES LIKE 'have_openssl';
SHOW VARIABLES LIKE 'ssl%';

-- Exemple de sortie
/*
+---------------------+--------------------------------+
| Variable_name       | Value                          |
+---------------------+--------------------------------+
| have_openssl        | YES                            |
| have_ssl            | YES                            |
| ssl_ca              | /var/lib/mysql/ca-cert.pem     |
| ssl_cert            | /var/lib/mysql/server-cert.pem |
| ssl_cipher          | (liste des ciphers)            |
| ssl_key             | /var/lib/mysql/server-key.pem  |
+---------------------+--------------------------------+
*/
```

### Vérifier la connexion client

```sql
-- Informations SSL de la connexion actuelle
SHOW STATUS LIKE 'Ssl%';

-- Exemple de sortie pour connexion chiffrée
/*
+--------------------+----------------------------+
| Variable_name      | Value                      |
+--------------------+----------------------------+
| Ssl_cipher         | TLS_AES_256_GCM_SHA384     |
| Ssl_cipher_list    | ...                        |
| Ssl_version        | TLSv1.3                    |
| Ssl_server_not_... | ...                        |
+--------------------+----------------------------+
*/

-- Si Ssl_cipher est vide → Connexion NON chiffrée!
```

### Test de connexion SSL

```bash
# Test basique
mariadb -u user -p -h db.example.com --ssl

# Test avec vérification stricte
mariadb -u user -p -h db.example.com \
  --ssl-mode=VERIFY_CA \
  --ssl-ca=/path/to/ca-cert.pem

# Test avec vérification du hostname
mariadb -u user -p -h db.example.com \
  --ssl-mode=VERIFY_IDENTITY \
  --ssl-ca=/path/to/ca-cert.pem
```

### Modes SSL du client

| Mode | Description | Vérification |
|------|-------------|--------------|
| **DISABLED** | SSL désactivé | Aucune |
| **PREFERRED** | SSL si disponible (défaut) | Opportuniste |
| **REQUIRED** | SSL obligatoire | Chiffrement |
| **VERIFY_CA** | SSL + vérif CA | CA valide |
| **VERIFY_IDENTITY** | SSL + vérif hostname | CN/SAN match |

### Diagnostiquer les erreurs SSL

**Erreur : "SSL connection error"**

```bash
# Vérifier les certificats
openssl x509 -in /var/lib/mysql/server-cert.pem -text -noout

# Vérifier la chaîne de certification
openssl verify -CAfile ca-cert.pem server-cert.pem

# Vérifier les permissions
ls -la /var/lib/mysql/*.pem
# server-key.pem doit être 400 ou 600 (mysql:mysql)
```

**Erreur : "Certificate verify failed"**

```bash
# Client ne fait pas confiance au CA
# Solution: Copier ca-cert.pem côté client

mariadb -u user -p -h db.example.com \
  --ssl-ca=/path/to/ca-cert.pem
```

**Erreur : "SSL is required but not enabled"**

```sql
-- L'utilisateur a REQUIRE SSL mais le client ne l'utilise pas
-- Solution côté client:
mariadb -u user -p --ssl
```

---

## Bonnes pratiques production

### 1. Toujours utiliser TLS en production

```sql
-- ❌ MAUVAIS: SSL optionnel
CREATE USER 'prod_app'@'%' IDENTIFIED BY 'password';

-- ✅ BON: SSL obligatoire
CREATE USER 'prod_app'@'%'
  IDENTIFIED BY 'password'
  REQUIRE SSL;
```

### 2. TLS 1.3 uniquement (si possible)

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
tls_version = TLSv1.3

# Si compatibilité TLS 1.2 nécessaire
tls_version = TLSv1.2,TLSv1.3
```

### 3. Certificats signés par CA (Let's Encrypt minimum)

```bash
# Ne pas utiliser auto-signés en production
# Utiliser Let's Encrypt (gratuit) ou CA interne
```

### 4. Rotation des certificats

```bash
# Certificats expirent → Automatiser le renouvellement
# Let's Encrypt: 90 jours
# CA interne: 365-730 jours recommandés

# Cron: Renouveler et recharger
0 3 1 * * certbot renew --deploy-hook "systemctl reload mariadb"
```

### 5. Monitoring de l'expiration

```bash
#!/bin/bash
# check_cert_expiry.sh

CERT="/var/lib/mysql/server-cert.pem"
DAYS_WARNING=30

EXPIRY=$(openssl x509 -in $CERT -enddate -noout | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

if [ $DAYS_LEFT -lt $DAYS_WARNING ]; then
  echo "WARNING: Certificate expires in $DAYS_LEFT days"
  # Envoyer alerte
fi
```

### 6. Désactiver les protocoles faibles

```ini
[mysqld]
# NE PAS autoriser TLS 1.0 ou 1.1 (obsolètes, vulnérables)
tls_version = TLSv1.2,TLSv1.3

# NE PAS autoriser ciphers faibles
ssl_cipher = !aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!3DES
```

### 7. mTLS pour services critiques

```sql
-- Services de paiement, banking → Certificat client obligatoire
CREATE USER 'payment_service'@'payment_server'
  IDENTIFIED BY 'password'
  REQUIRE X509;
```

### 8. Séparation des certificats par environnement

```
DEV:    Certificats auto-signés (acceptable)
STAGING: Certificats Let's Encrypt ou CA interne
PROD:   Certificats CA publique ou CA interne validée
```

### 9. Permissions strictes sur les clés privées

```bash
# Clé privée serveur
chown mysql:mysql /var/lib/mysql/server-key.pem
chmod 400 /var/lib/mysql/server-key.pem

# Vérification
ls -la /var/lib/mysql/server-key.pem
# -r-------- 1 mysql mysql server-key.pem ✓
```

### 10. Audit des connexions non-chiffrées

```sql
-- Activer l'audit pour détecter connexions non-SSL
INSTALL SONAME 'server_audit';
SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_events = 'CONNECT';

-- Analyser les logs pour connexions sans SSL
-- (Ssl_cipher vide dans audit log)
```

---

## ✅ Points clés à retenir

- **SSL/TLS protège les données en transit** : requêtes, résultats, mots de passe chiffrés
- **🆕 MariaDB 11.8 active TLS par défaut** avec certificats auto-générés
- **TLS 1.3 est 2x plus rapide** que TLS 1.2 (1 RTT vs 2 RTT)
- **PCI-DSS exige TLS 1.2 minimum** (TLS 1.0/1.1 interdits)
- **REQUIRE SSL force le chiffrement** pour utilisateurs spécifiques
- **REQUIRE X509 active mTLS** (authentification mutuelle)
- **Let's Encrypt offre des certificats gratuits** (renouvellement automatique)
- **Les cipher suites doivent exclure les algorithmes faibles** (RC4, 3DES, MD5)
- **Perfect Forward Secrecy (PFS) est obligatoire** en TLS 1.3
- **Les certificats doivent être renouvelés** avant expiration (monitoring)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Secure Connections Overview](https://mariadb.com/kb/en/secure-connections-overview/)
- [📖 SSL/TLS Configuration](https://mariadb.com/kb/en/securing-connections-for-client-and-server/)
- [📖 🆕 TLS by Default (11.8)](https://mariadb.com/kb/en/mariadb-1184-release-notes/)
- [📖 REQUIRE Clause](https://mariadb.com/kb/en/create-user/#require-clause)

### Standards et RFC

- [RFC 8446 - TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 5246 - TLS 1.2](https://www.rfc-editor.org/rfc/rfc5246)
- [PCI-DSS v4.0](https://www.pcisecuritystandards.org/)

### Outils

- [OpenSSL](https://www.openssl.org/)
- [Let's Encrypt](https://letsencrypt.org/)
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)

---

## ➡️ Sections suivantes

Les sous-sections détailleront la configuration pratique de SSL/TLS :

- **10.7.1** : Configuration serveur SSL (génération certificats, my.cnf)
- **10.7.2** : Certificats et CA (Let's Encrypt, CA interne, rotation)
- **10.7.3** : 🆕 TLS par défaut depuis 11.8 (migration, certificats auto-générés)

**La section suivante (10.7.1)** entrera dans le détail de la **configuration serveur SSL** avec tous les paramètres my.cnf et la génération de certificats.

---


⏭️ [Configuration serveur SSL](/10-securite-gestion-utilisateurs/07.1-configuration-serveur-ssl.md)

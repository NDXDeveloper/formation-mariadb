🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6 Plugin d'authentification PARSEC 🆕

> **Niveau** : Expert
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 10.1-10.5, connaissances HSM et cryptographie

> **Nouveauté** : MariaDB 11.8 LTS (Juin 2025)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle des HSM (Hardware Security Modules) dans la sécurité
- **Maîtriser** l'architecture du plugin PARSEC et son fonctionnement
- **Installer** et configurer PARSEC avec différents providers
- **Intégrer** MariaDB avec des HSM matériels et cloud (AWS, Azure, Thales)
- **Implémenter** des solutions conformes PCI-DSS et FIPS 140-2/3
- **Déployer** PARSEC en production avec haute disponibilité
- **Auditer** et monitorer les authentifications HSM
- **Diagnostiquer** les problèmes de configuration PARSEC

---

## Introduction

Le **plugin PARSEC** (Platform AbstRaction for SECurity) est une innovation majeure introduite dans MariaDB 11.8 LTS. Il permet d'utiliser des **Hardware Security Modules (HSM)** pour l'authentification, offrant le plus haut niveau de sécurité possible pour les bases de données.

### Pourquoi PARSEC est révolutionnaire ?

Avant MariaDB 11.8, l'authentification reposait sur des **clés logicielles** stockées dans la base de données :

```
Authentification traditionnelle (ed25519, mysql_native_password):
┌────────────────────────────────────────┐
│  Base de données MariaDB               │
│  ┌──────────────────────────────────┐  │
│  │ mysql.global_priv                │  │
│  │ authentication_string (en RAM)   │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
         ↓
    ⚠️ Risque: Clés en mémoire
    ⚠️ Risque: Exfiltration possible
```

Avec PARSEC, les **clés cryptographiques ne quittent jamais le HSM** :

```
Authentification PARSEC (HSM):
┌────────────────────────────────────────┐
│  Base de données MariaDB               │
│  ┌──────────────────────────────────┐  │
│  │ mysql.global_priv                │  │
│  │ key_id: "parsec://hsm/key_001"   │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
         ↓ (référence uniquement)
┌────────────────────────────────────────┐
│  Hardware Security Module (HSM)        │
│  ┌──────────────────────────────────┐  │
│  │ Clés cryptographiques PHYSIQUES  │  │
│  │ ✓ Inextractibles                 │  │
│  │ ✓ Tamper-resistant               │  │
│  │ ✓ Audit matériel                 │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

### Cas d'usage critiques

PARSEC est **indispensable** pour :

1. **Conformité réglementaire**
   - PCI-DSS (Payment Card Industry)
   - FIPS 140-2/3 (Federal Information Processing Standards)
   - GDPR (données de santé)
   - SOC2 Type II

2. **Secteurs hautement régulés**
   - Banques et institutions financières
   - Traitement de paiements
   - Santé (HIPAA)
   - Gouvernement et défense
   - Cloud souverain

3. **Protection d'actifs critiques**
   - Clés de chiffrement maîtres
   - Signatures numériques
   - Certificats racine
   - Identités privilégiées

**Exemple réel** : Une banque traitant des millions de transactions doit garantir que les clés d'authentification de ses systèmes de paiement sont **inextractibles** et **auditables matériellement**. PARSEC avec un HSM certifié FIPS 140-2 Level 3 est la seule solution acceptable.

---

## Qu'est-ce qu'un HSM (Hardware Security Module) ?

### Définition

Un **Hardware Security Module** est un dispositif cryptographique **matériel** conçu pour :

1. **Générer** des clés cryptographiques avec un vrai générateur aléatoire matériel (TRNG)
2. **Stocker** les clés de manière **inextractible**
3. **Exécuter** les opérations cryptographiques (signature, déchiffrement) **sans jamais exposer les clés**
4. **Se détruire** en cas de tentative de violation physique (tamper-resistant)
5. **Auditer** toutes les opérations cryptographiques

### Architecture d'un HSM

```
┌─────────────────────────────────────────────────────────────┐
│                    HSM (Hardware)                           │
├─────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Secure Processor (cryptoprocessor dédié)             │  │
│  │  - ARM Cortex-M / Custom ASIC                         │  │
│  │  - Exécution isolée (secure enclave)                  │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Secure Memory (RAM chiffrée)                         │  │
│  │  - Clés en mémoire volatile                           │  │
│  │  - Effacement automatique si alimentation coupée      │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Secure Storage (stockage persistant chiffré)         │  │
│  │  - Clés privées (asymétriques)                        │  │
│  │  - Clés symétriques (AES, etc.)                       │  │
│  │  - Certificats                                        │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  TRNG (True Random Number Generator)                  │  │
│  │  - Générateur matériel (bruit quantique, etc.)        │  │
│  │  - Entropie cryptographique garantie                  │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Tamper Detection                                     │  │
│  │  - Détecteurs d'ouverture physique                    │  │
│  │  - Détecteurs de voltage/température                  │  │
│  │  - Auto-destruction si violation                      │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Audit Log (immuable)                                 │  │
│  │  - Log de toutes les opérations cryptographiques      │  │
│  │  - Horodatage sécurisé                                │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↓ Interface (PKCS#11, TPM, propriétaire)
┌─────────────────────────────────────────────────────────────┐
│              Application (MariaDB + PARSEC)                 │
└─────────────────────────────────────────────────────────────┘
```

### Niveaux de certification FIPS 140-2

| Level | Sécurité | Caractéristiques | Exemples |
|-------|----------|------------------|----------|
| **Level 1** | De base | Algorithmes validés, pas de sécurité physique | YubiKey FIPS, logiciels validés |
| **Level 2** | Moyenne | + Tamper evidence (détection violation) | AWS CloudHSM, Azure Key Vault HSM |
| **Level 3** | Haute | + Tamper resistance (résistance physique) | Thales Luna, Utimaco CryptoServer |
| **Level 4** | Très haute | + Détection environnement (temp, voltage) | Militaire, gouvernemental |

💡 **PCI-DSS** exige au minimum **FIPS 140-2 Level 2** pour stocker les clés de chiffrement de données de carte bancaire.

### Types de HSM

#### 1. HSM matériels (on-premise)

**Exemples** :
- **Thales Luna SA** : 10 000 - 50 000 € / unité (Level 3)
- **Entrust nShield** : 15 000 - 60 000 € / unité (Level 3)
- **Utimaco CryptoServer** : 20 000 - 70 000 € / unité (Level 3/4)

**Avantages** :
- ✅ Performance maximale (milliers d'ops/s)
- ✅ Contrôle total
- ✅ Certifications Level 3/4

**Inconvénients** :
- ❌ Coût élevé
- ❌ Maintenance complexe
- ❌ Pas de scalabilité élastique

#### 2. HSM cloud (HSM-as-a-Service)

**Exemples** :
- **AWS CloudHSM** : ~1,50 $/heure (Level 2)
- **Azure Dedicated HSM** : ~1,20 $/heure (Level 3)
- **Google Cloud HSM** : ~1,80 $/heure (Level 3)

**Avantages** :
- ✅ Pas d'investissement initial
- ✅ Scalabilité élastique
- ✅ Maintenance gérée

**Inconvénients** :
- ❌ Dépendance au cloud provider
- ❌ Coût récurrent
- ❌ Latence réseau

#### 3. TPM (Trusted Platform Module)

**Exemples** :
- **TPM 2.0** : Intégré dans les serveurs modernes (gratuit)
- **Intel vPro** : Avec TPM hardware

**Avantages** :
- ✅ Intégré (pas de coût)
- ✅ Disponible sur tous les serveurs récents
- ✅ Suffisant pour conformité de base

**Inconvénients** :
- ❌ Performance limitée
- ❌ Moins robuste que HSM dédié
- ❌ Certification variable

#### 4. HSM logiciels (dev/test uniquement)

**Exemples** :
- **SoftHSM** : HSM émulé (open source)
- **Mbed Crypto** : Bibliothèque crypto

**⚠️ Attention** : NON utilisables en production (pas de protection physique).

---

## Architecture du plugin PARSEC

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                   MariaDB Server                             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Plugin PARSEC (auth_parsec.so)                        │  │
│  │  - Interface MariaDB Authentication API                │  │
│  │  - Gestion lifecycle des clés                          │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  PARSEC Client Library (libparsec.so)                  │  │
│  │  - Protocole PARSEC (gRPC/Protobuf)                    │  │
│  │  - Sérialisation/Désérialisation                       │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                          ↓ Unix Socket / TCP
┌─────────────────────────────────────────────────────────────┐
│              PARSEC Service (parsec)                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Core Engine                                          │  │
│  │  - Authentification client                            │  │
│  │  - Autorisation (ACL)                                 │  │
│  │  - Dispatching vers providers                         │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌────────────┬─────────────┬──────────────┬─────────────┐  │
│  │ Provider   │ Provider    │ Provider     │ Provider    │  │
│  │ PKCS#11    │ TPM         │ Mbed Crypto  │ Trusted     │  │
│  │            │             │              │ Service     │  │
│  └────────────┴─────────────┴──────────────┴─────────────┘  │
└─────────────────────────────────────────────────────────────┘
        ↓              ↓             ↓              ↓
┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐
│ HSM matériel │ │   TPM    │ │ Software │ │  ARM TrustZone│
│ (Thales,etc) │ │   2.0    │ │  Crypto  │ │  / Intel SGX  │
└──────────────┘ └──────────┘ └──────────┘ └───────────────┘
```

### Providers supportés

#### 1. PKCS#11 Provider (HSM matériels et cloud)

**Description** : Interface standard pour HSM matériels.

**Supporte** :
- Thales Luna
- Entrust nShield
- Utimaco CryptoServer
- AWS CloudHSM (via client PKCS#11)
- Azure Dedicated HSM
- SoftHSM (dev/test)

**Configuration** :

```toml
# /etc/parsec/config.toml
[provider.pkcs11]
library = "/usr/lib/libpkcs11.so"  # Bibliothèque PKCS#11 du vendor
slot_number = 0                     # Slot HSM (partition logique)
user_pin = "1234"                   # PIN utilisateur (ou via env var)
```

#### 2. TPM Provider (Trusted Platform Module)

**Description** : Utilise le TPM 2.0 intégré au serveur.

**Supporte** :
- TPM 2.0 (hardware)
- TPM 2.0 (software emulator pour test)

**Configuration** :

```toml
[provider.tpm]
tcti = "device:/dev/tpmrm0"         # TPM Resource Manager
owner_hierarchy_auth = ""           # Mot de passe hiérarchie owner
endorsement_hierarchy_auth = ""     # Mot de passe hiérarchie endorsement
```

#### 3. Mbed Crypto Provider (Software, dev/test)

**Description** : Implémentation logicielle (NON pour production).

**Configuration** :

```toml
[provider.mbed-crypto]
# Pas de configuration HSM (tout en software)
```

⚠️ **Attention** : Mbed Crypto **NE doit PAS** être utilisé en production (pas de protection physique).

#### 4. Trusted Service Provider (ARM TrustZone / Intel SGX)

**Description** : Secure enclave pour ARM/Intel.

**Supporte** :
- ARM TrustZone (TEE - Trusted Execution Environment)
- Intel SGX (Software Guard Extensions)

**Configuration** :

```toml
[provider.trusted-service]
address = "unix:/run/parsec/trusted-service.sock"
```

---

## Installation et configuration

### Prérequis

**Système** :
- MariaDB 11.8+ (LTS ou supérieur)
- Linux x86_64 ou ARM64
- Kernel avec support HSM/TPM (selon provider)

**HSM** :
- HSM matériel configuré (PKCS#11)
- ou TPM 2.0 activé dans BIOS
- ou AWS CloudHSM / Azure Key Vault provisionné

### Étape 1 : Installation du plugin PARSEC

```bash
# Vérifier la disponibilité du plugin
ls /usr/lib64/mysql/plugin/auth_parsec.so

# Si absent, installer depuis les packages MariaDB
sudo dnf install MariaDB-plugin-auth-parsec  # RHEL/CentOS/Rocky
sudo apt install mariadb-plugin-auth-parsec  # Debian/Ubuntu
```

### Étape 2 : Installation du service PARSEC

```bash
# Installation depuis les repos
sudo dnf install parsec-service  # RHEL
sudo apt install parsec-service  # Debian

# Vérification
parsec --version
# parsec 1.3.0
```

### Étape 3 : Configuration du service PARSEC

**Configuration principale** (`/etc/parsec/config.toml`) :

```toml
# Configuration PARSEC pour MariaDB avec HSM

# Core settings
[core_settings]
log_level = "info"
log_timestamp = true
buffer_size_limit = 1048576  # 1MB

# Listener (Unix socket recommandé pour sécurité)
[listener]
listener_type = "UnixSocket"
socket_path = "/run/parsec/parsec.sock"

# Alternative: TCP (si MariaDB distant)
# [listener]
# listener_type = "TCP"
# bind_address = "127.0.0.1"
# port = 8081

# Provider PKCS#11 (HSM matériel)
[provider.pkcs11]
library = "/opt/thales/luna/libs/libCryptoki2_64.so"  # Thales Luna
slot_number = 0
user_pin = "${PARSEC_PKCS11_PIN}"  # Variable d'environnement
# user_pin = "1234"  # ⚠️ Ne pas hardcoder en production

# Provider TPM (fallback)
[provider.tpm]
tcti = "device:/dev/tpmrm0"
owner_hierarchy_auth = ""

# Authentification des clients
[authenticator.unix_peer_credentials]
# Vérifie l'UID/GID Unix du client
```

### Étape 4 : Configuration des permissions

```bash
# Créer le groupe parsec
sudo groupadd parsec

# Ajouter l'utilisateur MariaDB au groupe
sudo usermod -a -G parsec mysql

# Permissions socket
sudo mkdir -p /run/parsec
sudo chown root:parsec /run/parsec
sudo chmod 0770 /run/parsec

# Permissions configuration
sudo chown root:parsec /etc/parsec/config.toml
sudo chmod 0640 /etc/parsec/config.toml
```

### Étape 5 : Démarrage du service PARSEC

```bash
# Activer et démarrer
sudo systemctl enable parsec
sudo systemctl start parsec

# Vérifier le statut
sudo systemctl status parsec

# Vérifier les logs
sudo journalctl -u parsec -f

# Tester la connexion
parsec-tool list-providers
# PKCS#11 Provider (ID: 1)
# TPM Provider (ID: 2)
```

### Étape 6 : Installation du plugin dans MariaDB

```sql
-- Connexion en tant que root
mariadb -u root -p

-- Installer le plugin
INSTALL SONAME 'auth_parsec';

-- Vérifier l'installation
SHOW PLUGINS WHERE Name = 'parsec';
/*
+--------+--------+---------------+------------------+---------+
| Name   | Status | Type          | Library          | License |
+--------+--------+---------------+------------------+---------+
| parsec | ACTIVE | AUTHENTICATION| auth_parsec.so   | GPL     |
+--------+--------+---------------+------------------+---------+
*/

-- Vérifier la connexion PARSEC
SELECT @@parsec_socket_path;
-- /run/parsec/parsec.sock
```

---

## Configuration HSM spécifiques

### AWS CloudHSM

**Architecture** :

```
MariaDB → PARSEC → PKCS#11 Client → AWS CloudHSM
                    (libcloudhsm_pkcs11.so)
```

**Étape 1 : Provisionner CloudHSM**

```bash
# AWS Console ou CLI
aws cloudhsmv2 create-cluster \
  --subnet-ids subnet-xxxxx subnet-yyyyy \
  --hsm-type hsm1.medium

# Créer un HSM dans le cluster
aws cloudhsmv2 create-hsm \
  --cluster-id cluster-xxxxx \
  --availability-zone us-east-1a

# Initialiser le cluster (obtenir les certificats)
aws cloudhsmv2 describe-clusters --cluster-ids cluster-xxxxx
```

**Étape 2 : Installer le client CloudHSM**

```bash
# Télécharger et installer le client
wget https://s3.amazonaws.com/cloudhsmv2-software/CloudHsmClient/EL8/cloudhsm-client-latest.el8.x86_64.rpm
sudo rpm -ivh cloudhsm-client-latest.el8.x86_64.rpm

# Configurer le client
sudo /opt/cloudhsm/bin/configure -a <CLUSTER_IP>

# Vérifier la connexion
/opt/cloudhsm/bin/cloudhsm_mgmt_util
# loginHSM CU admin password
```

**Étape 3 : Configuration PARSEC**

```toml
# /etc/parsec/config.toml
[provider.pkcs11]
library = "/opt/cloudhsm/lib/libcloudhsm_pkcs11.so"
slot_number = 0
user_pin = "${CLOUDHSM_USER_PIN}"
```

**Étape 4 : Création d'utilisateur MariaDB**

```sql
-- Générer une clé dans CloudHSM d'abord
-- (via cloudhsm_mgmt_util ou key_mgmt_util)

-- Créer l'utilisateur MariaDB
CREATE USER 'payment_service'@'app_server'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/payment_key_001';

-- Tester la connexion
mariadb -u payment_service -p -h db.example.com
```

### Azure Dedicated HSM

**Architecture** :

```
MariaDB → PARSEC → PKCS#11 Client → Azure Dedicated HSM
                    (libCryptoki2_64.so)  (Thales Luna)
```

**Étape 1 : Provisionner Azure Dedicated HSM**

```bash
# Créer le HSM via Azure Portal ou CLI
az dedicated-hsm create \
  --resource-group myResourceGroup \
  --name myHSM \
  --location eastus \
  --sku SafeNet-Luna-Network-HSM-A790 \
  --stamp-id stamp1 \
  --subnet /subscriptions/.../subnets/hsm-subnet
```

**Étape 2 : Installer le client Thales Luna**

```bash
# Télécharger depuis le portail Thales
# Installer le package
sudo rpm -ivh LunaClient-10.x.x-xxx.x86_64.rpm

# Copier le certificat serveur depuis Azure
# Enregistrer le HSM
/usr/safenet/lunaclient/bin/vtl addServer -n <HSM_IP> -c server.pem

# Créer une partition (via LunaSH)
ssh admin@<HSM_IP>
lunash:> partition create -partition MariaDB
```

**Étape 3 : Configuration PARSEC**

```toml
[provider.pkcs11]
library = "/usr/safenet/lunaclient/lib/libCryptoki2_64.so"
slot_number = 1  # Numéro de partition
user_pin = "${AZURE_HSM_PIN}"
```

### Thales Luna on-premise

**Configuration complète** :

```toml
# /etc/parsec/config.toml
[core_settings]
log_level = "info"

[listener]
listener_type = "UnixSocket"
socket_path = "/run/parsec/parsec.sock"

[provider.pkcs11]
library = "/usr/safenet/lunaclient/lib/libCryptoki2_64.so"
slot_number = 0  # Ajuster selon votre partition

# PIN via variable d'environnement (recommandé)
user_pin = "${LUNA_PARTITION_PIN}"

# OU via fichier séparé
# user_pin_file = "/etc/parsec/hsm_pin"  # chmod 400

# Configuration avancée
[provider.pkcs11.advanced]
max_sessions = 100
session_timeout = 300  # 5 minutes
```

**Fichier systemd pour sécuriser le PIN** (`/etc/systemd/system/parsec.service.d/override.conf`) :

```ini
[Service]
# Charger le PIN depuis un fichier sécurisé
EnvironmentFile=/etc/parsec/hsm_credentials
# Contenu du fichier: LUNA_PARTITION_PIN=votre_pin_secret

# Permissions strictes
PermissionsStartOnly=true
ExecStartPre=/bin/chown root:parsec /run/parsec
ExecStartPre=/bin/chmod 0770 /run/parsec

# Sécurité renforcée
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
```

### TPM 2.0 (intégré au serveur)

**Étape 1 : Activer TPM dans le BIOS**

```bash
# Vérifier que TPM est disponible
ls /dev/tpm*
# /dev/tpm0  /dev/tpmrm0

# Vérifier la version
cat /sys/class/tpm/tpm0/tpm_version_major
# 2  (TPM 2.0)
```

**Étape 2 : Installer les outils TPM**

```bash
# RHEL/CentOS/Rocky
sudo dnf install tpm2-tools tpm2-tss

# Debian/Ubuntu
sudo apt install tpm2-tools libtss2-dev

# Vérifier TPM
tpm2_getrandom 8
# 0x1A2B3C4D5E6F7A8B  (random bytes)
```

**Étape 3 : Configuration PARSEC**

```toml
[provider.tpm]
tcti = "device:/dev/tpmrm0"  # TPM Resource Manager (recommandé)
# tcti = "device:/dev/tpm0"  # TPM direct (alternatif)

# Authentification hiérarchies (optionnel)
owner_hierarchy_auth = ""
endorsement_hierarchy_auth = ""
```

**Étape 4 : Créer un utilisateur MariaDB**

```sql
-- Générer une clé dans TPM
-- (via tpm2-tools ou PARSEC CLI)

CREATE USER 'app_tpm'@'localhost'
  IDENTIFIED VIA parsec USING 'parsec://tpm/app_key_001';

-- Tester
mariadb -u app_tpm -p
```

---

## Gestion des clés avec PARSEC

### Création de clés

**Via PARSEC CLI** :

```bash
# Installer parsec-tool
sudo dnf install parsec-tool

# Générer une clé RSA dans le HSM
parsec-tool generate-key \
  --provider pkcs11 \
  --key-name "mariadb_user_001" \
  --key-algorithm Rsa \
  --key-size 4096

# Générer une clé ECC (plus performant)
parsec-tool generate-key \
  --provider pkcs11 \
  --key-name "mariadb_user_002" \
  --key-algorithm EccP256

# Lister les clés
parsec-tool list-keys
/*
Key: mariadb_user_001 (RSA 4096 bits)
Provider: PKCS#11
Key: mariadb_user_002 (ECC P-256)
Provider: PKCS#11
*/
```

### Utilisation dans MariaDB

```sql
-- Créer un utilisateur avec clé PARSEC
CREATE USER 'secure_app'@'app_servers'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/mariadb_user_001';

-- Vérifier
SELECT User, Host, plugin, authentication_string
FROM mysql.user
WHERE User = 'secure_app';
/*
+------------+-------------+--------+----------------------------------+
| User       | Host        | plugin | authentication_string            |
+------------+-------------+--------+----------------------------------+
| secure_app | app_servers | parsec | parsec://pkcs11/mariadb_user_001 |
+------------+-------------+--------+----------------------------------+
*/
```

### Rotation de clés

```bash
#!/bin/bash
# rotate_parsec_key.sh - Rotation de clés HSM

USER_NAME="secure_app"
HOST="app_servers"
OLD_KEY="mariadb_user_001"
NEW_KEY="mariadb_user_001_v2"

# Générer nouvelle clé
parsec-tool generate-key \
  --provider pkcs11 \
  --key-name "$NEW_KEY" \
  --key-algorithm EccP256

# Mettre à jour l'utilisateur
mariadb -u root -p <<EOF
ALTER USER '$USER_NAME'@'$HOST'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/$NEW_KEY';
EOF

# Attendre période de transition (ex: 30 jours)
sleep $((30 * 24 * 3600))

# Supprimer l'ancienne clé
parsec-tool delete-key --key-name "$OLD_KEY"

echo "Key rotation complete for $USER_NAME@$HOST"
```

### Backup et récupération

⚠️ **Important** : Les clés HSM sont **non-extractibles** par design.

**Options de backup** :

1. **HSM clustering** (haute disponibilité)

```toml
# Configuration HA avec 2 HSM
[provider.pkcs11]
library = "/usr/safenet/lunaclient/lib/libCryptoki2_64.so"

# Partition répliquée automatiquement entre HSM
slot_number = 0  # Partition HA (synchronisée)
```

2. **Clés wrapped** (export chiffré)

```bash
# Créer une clé de wrapping master
parsec-tool generate-key \
  --key-name "master_wrap_key" \
  --key-algorithm Aes256

# Exporter une clé (chiffrée avec master)
hsm_export_wrapped_key \
  --key "mariadb_user_001" \
  --wrap-key "master_wrap_key" \
  --output /backup/mariadb_user_001.wrapped
```

3. **Disaster Recovery Plan**

```
Stratégie DR pour HSM:
1. HSM primaire (production)
2. HSM secondaire (réplication synchrone)
3. HSM tertiaire (site DR distant)
4. Backup des clés wrapped (offline, coffre)
```

---

## Conformité et audit

### PCI-DSS Compliance

**Exigences PCI-DSS 4.0** :

| Requirement | Description | PARSEC avec HSM |
|-------------|-------------|-----------------|
| **3.5** | Clés cryptographiques stockées de manière sécurisée | ✅ HSM FIPS 140-2 L2+ |
| **3.6** | Clés de chiffrement protégées | ✅ Inextractibles |
| **8.3** | MFA pour accès admin | ✅ Via HSM + PIN |
| **10.2** | Audit des accès aux données | ✅ Logs PARSEC + HSM |
| **12.3** | Documentation sécurité | ✅ Config PARSEC auditée |

**Configuration PCI-DSS** :

```toml
# /etc/parsec/config.toml - PCI-DSS compliant
[core_settings]
log_level = "debug"  # Audit complet
log_timestamp = true

[provider.pkcs11]
library = "/opt/thales/luna/libs/libCryptoki2_64.so"
slot_number = 0

# Audit obligatoire
[audit]
enabled = true
log_file = "/var/log/parsec/audit.log"
log_format = "json"
log_all_operations = true
```

### FIPS 140-2/3

**Validation FIPS** :

```bash
# Vérifier que le HSM est FIPS-validé
parsec-tool get-provider-info --provider pkcs11
/*
Provider: PKCS#11
Library: /opt/thales/luna/libs/libCryptoki2_64.so
FIPS Mode: Enabled
FIPS Level: 140-2 Level 3
Certificate: #xxxx (validé NIST)
*/

# Vérifier le mode FIPS de MariaDB
mariadb -u root -p -e "SHOW VARIABLES LIKE 'ssl_fips_mode';"
/*
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| ssl_fips_mode | ON    |
+---------------+-------+
*/
```

### Audit des opérations

**Activer l'audit PARSEC** :

```toml
[audit]
enabled = true
log_file = "/var/log/parsec/audit.log"
log_format = "json"
log_all_operations = true

# Rotation des logs
[audit.rotation]
max_file_size = "100MB"
max_files = 10
compress = true
```

**Exemple de log d'audit** :

```json
{
  "timestamp": "2025-12-13T10:15:30.123Z",
  "operation": "Sign",
  "key_name": "mariadb_user_001",
  "provider": "pkcs11",
  "client_uid": 999,
  "client_gid": 999,
  "client_process": "mariadbd",
  "status": "success",
  "duration_ms": 12
}
```

**Requêtes d'audit** :

```bash
# Nombre d'authentifications par utilisateur/jour
jq -r 'select(.operation == "Sign") | .key_name' /var/log/parsec/audit.log \
  | sort | uniq -c | sort -rn

# Détection d'anomalies (> 1000 auth/heure)
jq -r 'select(.operation == "Sign") | .timestamp' /var/log/parsec/audit.log \
  | awk '{print substr($0, 1, 13)}' | uniq -c | awk '$1 > 1000 {print}'
```

### Intégration SIEM

**Envoi vers Splunk/ELK** :

```bash
# Filebeat configuration
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/parsec/audit.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      source: parsec_hsm
      environment: production

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "parsec-audit-%{+yyyy.MM.dd}"
```

---

## Haute disponibilité

### Architecture HA avec HSM clustering

```
┌──────────────────────────────────────────────────────────┐
│                  MariaDB Galera Cluster                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Node 1       │  │ Node 2       │  │ Node 3       │    │
│  │ PARSEC       │  │ PARSEC       │  │ PARSEC       │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└──────────────────────────────────────────────────────────┘
         ↓                  ↓                  ↓
┌──────────────────────────────────────────────────────────┐
│              HSM Cluster (HA Partition)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ HSM Primary  │←→│ HSM Secondary│←→│ HSM Tertiary │    │
│  │ (Active)     │  │ (Sync)       │  │ (Sync)       │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│  Partition synchronized in real-time                     │
└──────────────────────────────────────────────────────────┘
```

**Configuration Thales Luna HA** :

```bash
# Sur le client Luna (chaque nœud MariaDB)
# Créer un groupe HA
/usr/safenet/lunaclient/bin/vtl createHAGroup -label MariaDB_HA

# Ajouter les membres du groupe
/usr/safenet/lunaclient/bin/vtl addHAMember \
  -group MariaDB_HA -serialNum 12345 -partition MariaDB_P1

/usr/safenet/lunaclient/bin/vtl addHAMember \
  -group MariaDB_HA -serialNum 67890 -partition MariaDB_P2

# Vérifier le groupe HA
/usr/safenet/lunaclient/bin/vtl haAdmin -show
/*
HA Group: MariaDB_HA
  Member 1: HSM Serial 12345, Partition MariaDB_P1 (PRIMARY)
  Member 2: HSM Serial 67890, Partition MariaDB_P2 (SECONDARY)
  Synchronization: ENABLED
  Auto-recovery: ENABLED
*/
```

**Configuration PARSEC pour HA** :

```toml
[provider.pkcs11]
library = "/usr/safenet/lunaclient/lib/libCryptoki2_64.so"
slot_number = 0  # Slot du groupe HA

# Timeout et retry pour HA
[provider.pkcs11.ha]
failover_timeout = 5000  # 5 secondes
retry_count = 3
```

### Monitoring HSM

**Script de monitoring** :

```bash
#!/bin/bash
# monitor_hsm.sh - Monitoring HSM health

# Vérifier PARSEC service
if ! systemctl is-active --quiet parsec; then
  echo "CRITICAL: PARSEC service is down"
  exit 2
fi

# Vérifier connexion HSM
if ! parsec-tool list-providers | grep -q "PKCS#11"; then
  echo "CRITICAL: HSM not accessible"
  exit 2
fi

# Vérifier latence
LATENCY=$(parsec-tool benchmark --provider pkcs11 --operations 10 | grep "Average" | awk '{print $3}')
if (( $(echo "$LATENCY > 100" | bc -l) )); then
  echo "WARNING: HSM latency high: ${LATENCY}ms"
  exit 1
fi

# Vérifier partition usage (si supporté par le vendor)
# ...

echo "OK: HSM operational, latency ${LATENCY}ms"
exit 0
```

**Intégration Prometheus** :

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'parsec_hsm'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'

# Exporter custom pour PARSEC
# /usr/local/bin/parsec_exporter.sh
```

---

## Troubleshooting

### Problème 1 : Service PARSEC ne démarre pas

```bash
# Symptôme
sudo systemctl status parsec
# Failed to start PARSEC service

# Diagnostic
sudo journalctl -u parsec -n 50

# Causes courantes:
# 1. Bibliothèque PKCS#11 introuvable
ls -la /opt/thales/luna/libs/libCryptoki2_64.so

# 2. Permissions socket
ls -la /run/parsec/
sudo chmod 0770 /run/parsec/

# 3. Configuration invalide
parsec --config /etc/parsec/config.toml --validate

# 4. HSM inaccessible
/usr/safenet/lunaclient/bin/vtl verify
```

### Problème 2 : Authentification échoue

```bash
# Symptôme
mariadb -u secure_app -p
# ERROR 1045 (28000): Access denied

# Diagnostic étape par étape

# 1. Vérifier plugin installé
mariadb -u root -p -e "SHOW PLUGINS WHERE Name = 'parsec';"

# 2. Vérifier socket PARSEC
ls -la /run/parsec/parsec.sock
sudo chmod 0770 /run/parsec/parsec.sock
sudo chown root:parsec /run/parsec/parsec.sock

# 3. Vérifier groupe MariaDB
groups mysql
# mysql : mysql parsec

# 4. Tester PARSEC directement
sudo -u mysql parsec-tool list-keys

# 5. Vérifier clé existe dans HSM
parsec-tool list-keys | grep mariadb_user_001

# 6. Logs MariaDB
sudo tail -f /var/log/mysql/error.log

# 7. Logs PARSEC
sudo journalctl -u parsec -f
```

### Problème 3 : Performance dégradée

```bash
# Symptôme
# Authentification très lente (> 1 seconde)

# Benchmark PARSEC
parsec-tool benchmark \
  --provider pkcs11 \
  --operations 100 \
  --key-algorithm EccP256

# Résultats normaux:
# Average: 10-50ms
# P95: < 100ms
# P99: < 150ms

# Si > 200ms:
# Causes possibles:

# 1. Latence réseau (HSM cloud)
ping -c 10 <HSM_IP>

# 2. HSM overload
# Vérifier charge HSM (vendor-specific)

# 3. Session pool épuisé
# Augmenter max_sessions dans config.toml

# 4. Timeout trop court
# Augmenter session_timeout
```

### Problème 4 : HSM clustering failover

```bash
# Symptôme
# Erreur après failover HSM primaire → secondaire

# Vérifier état cluster HA
/usr/safenet/lunaclient/bin/vtl haAdmin -show

# Si membre primaire down:
# 1. Vérifier auto-recovery
# 2. Forcer failover manuel si nécessaire
/usr/safenet/lunaclient/bin/vtl haAdmin -failover

# Vérifier que PARSEC a détecté le failover
sudo journalctl -u parsec | grep -i failover

# Redémarrer PARSEC si nécessaire
sudo systemctl restart parsec
```

---

## Bonnes pratiques production

### 1. Séparation des environnements

```toml
# DEV: Mbed Crypto (software)
[provider.mbed-crypto]

# STAGING: SoftHSM (émulé)
[provider.pkcs11]
library = "/usr/lib64/pkcs11/libsofthsm2.so"

# PRODUCTION: HSM matériel uniquement
[provider.pkcs11]
library = "/opt/thales/luna/libs/libCryptoki2_64.so"
```

### 2. Principe du moindre privilège

```sql
-- Utilisateur HSM avec privilèges minimaux
CREATE USER 'payment_ro'@'app_servers'
  IDENTIFIED VIA parsec USING 'parsec://pkcs11/payment_readonly_key';

GRANT SELECT ON payments.* TO 'payment_ro'@'app_servers';
-- Pas de INSERT/UPDATE/DELETE
```

### 3. Monitoring continu

```bash
# Cron job: Vérifier HSM toutes les 5 minutes
*/5 * * * * /usr/local/bin/monitor_hsm.sh || mail -s "HSM Alert" admin@example.com
```

### 4. Documentation

```markdown
# Documentation HSM - Obligatoire pour PCI-DSS

## Architecture
- HSM: Thales Luna SA 7.x
- Certification: FIPS 140-2 Level 3
- Partition: MariaDB_Production (slot 0)

## Clés
- payment_processor_001: Traitement paiements (ECC P-256)
- trading_engine_001: Trading (RSA 4096)

## Contacts
- Vendor Support: +1-xxx-xxx-xxxx
- HSM Admin: alice@example.com
- Backup: /backup/hsm/wrapped_keys/

## Procédures
- Rotation: Trimestrielle
- Backup: Hebdomadaire
- DR: Site distant (réplication synchrone)
```

### 5. Tests réguliers

```bash
# Test de failover HSM (mensuel)
# 1. Simuler panne HSM primaire
# 2. Vérifier failover automatique
# 3. Vérifier que MariaDB fonctionne normalement
# 4. Restaurer HSM primaire
# 5. Vérifier failback
```

---

## ✅ Points clés à retenir

- **PARSEC est un plugin MariaDB 11.8+ pour authentification via HSM** (Hardware Security Module)
- **Les clés cryptographiques restent dans le HSM**, inextractibles et protégées physiquement
- **Conformité PCI-DSS et FIPS 140-2/3** garantie avec HSM certifiés
- **4 providers supportés** : PKCS#11 (HSM), TPM (intégré), Mbed Crypto (dev), Trusted Service (TEE)
- **Cloud HSM supportés** : AWS CloudHSM, Azure Dedicated HSM, Google Cloud HSM
- **Configuration en 6 étapes** : plugin MariaDB + service PARSEC + HSM
- **HA avec clustering HSM** : réplication synchrone pour haute disponibilité
- **Audit complet** : logs PARSEC + logs HSM pour traçabilité totale
- **Coût variable** : gratuit (TPM) à 50 000 € (HSM Level 3) + cloud (1-2 $/h)
- **Indispensable pour secteurs régulés** : banque, finance, santé, gouvernement

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 PARSEC Plugin - MariaDB 11.8](https://mariadb.com/kb/en/parsec-authentication-plugin/)
- [📖 PARSEC Project](https://parsec.community/)
- [📖 PKCS#11 Specification](https://www.oasis-open.org/committees/pkcs11/)

### HSM Vendors

- [Thales Luna HSM](https://cpl.thalesgroup.com/encryption/hardware-security-modules/network-hsms)
- [Entrust nShield](https://www.entrust.com/digital-security/hsm)
- [Utimaco CryptoServer](https://utimaco.com/products/hardware-security-modules)

### Cloud HSM

- [AWS CloudHSM](https://aws.amazon.com/cloudhsm/)
- [Azure Dedicated HSM](https://azure.microsoft.com/en-us/products/azure-dedicated-hsm/)
- [Google Cloud HSM](https://cloud.google.com/kms/docs/hsm)

### Standards

- [FIPS 140-2 Standard](https://csrc.nist.gov/publications/detail/fips/140/2/final)
- [PCI-DSS v4.0](https://www.pcisecuritystandards.org/)

---

## ➡️ Section suivante

**10.7 : Chiffrement des connexions (SSL/TLS)** - Vous apprendrez à configurer le chiffrement des connexions réseau, gérer les certificats, et activer TLS par défaut (nouveauté 11.8).

---


⏭️ [Chiffrement des connexions (SSL/TLS)](/10-securite-gestion-utilisateurs/07-chiffrement-ssl-tls.md)

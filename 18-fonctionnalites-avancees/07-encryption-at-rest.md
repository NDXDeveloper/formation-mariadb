🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.7 Encryption at Rest

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Chapitre 10 (Sécurité), compréhension PKI et cryptographie de base

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre l'**architecture d'encryption at rest** dans MariaDB
- Configurer le **chiffrement des données** (tables, tablespaces, logs)
- Maîtriser les **plugins de gestion de clés** (file, AWS KMS, Vault, KMIP)
- Implémenter une **hiérarchie de clés** sécurisée
- Mesurer l'**impact sur les performances** du chiffrement
- Gérer la **rotation des clés** de chiffrement
- Combiner encryption et **compliance** (RGPD, PCI-DSS, HIPAA)
- Sécuriser les **backups chiffrés**

---

## Introduction

L'**encryption at rest** (chiffrement au repos) protège les données stockées sur disque contre l'accès non autorisé en cas de vol de disques, backup mal sécurisé, ou accès physique au stockage. MariaDB offre un système de chiffrement transparent qui protège les données sans modification du code applicatif.

### Qu'est-ce que l'Encryption at Rest ?

L'encryption at rest chiffre les données **avant qu'elles ne soient écrites sur disque** et les déchiffre **lors de la lecture**. Le processus est transparent pour l'application.

**Ce qui est chiffré** :
- ✅ **Tables InnoDB** (fichiers .ibd)
- ✅ **Tablespaces système et temporaires**
- ✅ **Redo logs** (transaction logs)
- ✅ **Undo logs**
- ✅ **Binary logs** (optionnel)
- ✅ **Relay logs** (réplication)

**Ce qui N'est PAS chiffré par défaut** :
- ❌ Données en mémoire (buffer pool)
- ❌ Données en transit réseau (utiliser SSL/TLS)
- ❌ Fichiers de configuration (my.cnf)
- ❌ Error logs et query logs

### Pourquoi Chiffrer les Données au Repos ?

**Menaces adressées** :

1. **🔒 Vol ou Perte de Disques**
   - Disques égarés lors de maintenance
   - Hardware volé dans datacenter
   - Disques mal détruits après décommissionnement

2. **📦 Backups Compromis**
   - Backups stockés sur médias non sécurisés
   - Copie de backup interceptée
   - Accès non autorisé à stockage backup

3. **💾 Snapshot Storage**
   - Snapshots cloud accessibles
   - Anciens snapshots non supprimés
   - Permissions mal configurées

4. **🔐 Compliance Réglementaire**
   - RGPD : Protection données personnelles
   - PCI-DSS : Données bancaires (carte crédit)
   - HIPAA : Dossiers médicaux
   - SOX : Données financières

**Protection offerte** :
- ✅ Données illisibles sans clés de chiffrement
- ✅ Conformité aux exigences réglementaires
- ✅ Défense en profondeur (defense in depth)
- ✅ Tranquillité d'esprit (data breaches moins catastrophiques)

**Limitations importantes** :
- ⚠️ **N'empêche PAS** les attaques applicatives (SQL injection)
- ⚠️ **N'empêche PAS** l'accès via connexion légitime
- ⚠️ **N'empêche PAS** l'exfiltration de données en mémoire
- 💡 **Complément de** SSL/TLS, firewall, authentification forte

---

## Architecture du Chiffrement MariaDB

### Hiérarchie de Clés à Trois Niveaux

MariaDB utilise une architecture à **trois niveaux de clés** pour sécurité et flexibilité :

```
┌─────────────────────────────────────────────────┐
│  NIVEAU 1 : Master Encryption Key (MEK)         │
│  Stockée en dehors de MariaDB                   │
│  - AWS KMS, HashiCorp Vault, fichier chiffré    │
│  - Rarement utilisée directement                │
│  - Rotation possible sans re-chiffrer données   │
└──────────────────┬──────────────────────────────┘
                   │ Déchiffre
                   ↓
┌─────────────────────────────────────────────────┐
│  NIVEAU 2 : Table Encryption Keys (TEK)         │
│  Une clé par table/tablespace                   │
│  - Générée aléatoirement (AES-256)              │
│  - Stockée chiffrée dans fichier .ibd           │
│  - Chiffrée par MEK                             │
└──────────────────┬──────────────────────────────┘
                   │ Chiffre/Déchiffre
                   ↓
┌─────────────────────────────────────────────────┐
│  NIVEAU 3 : Page Encryption                     │
│  Pages InnoDB chiffrées individuellement        │
│  - Chaque page 16KB chiffrée avec TEK           │
│  - Algorithme AES-256 (ou AES-128)              │
│  - IV unique par page (Initialization Vector)   │
└─────────────────────────────────────────────────┘
```

**Avantages de cette architecture** :
- ✅ **Rotation de MEK rapide** : Re-chiffre uniquement les TEK (quelques secondes)
- ✅ **Performance** : Clés de table en cache, pas d'appel externe à chaque page
- ✅ **Granularité** : Chiffrement par table possible
- ✅ **Sécurité** : Clé maître jamais stockée dans MariaDB

### Algorithmes de Chiffrement

```sql
-- Algorithmes supportés
-- AES (Advanced Encryption Standard)

-- AES-256 (défaut, recommandé)
-- Clé 256 bits, sécurité maximale
SET GLOBAL innodb_encrypt_algorithm = 'AES-256';

-- AES-192
SET GLOBAL innodb_encrypt_algorithm = 'AES-192';

-- AES-128 (plus rapide, sécurité suffisante pour la plupart)
SET GLOBAL innodb_encrypt_algorithm = 'AES-128';
```

**Comparaison** :

| Algorithme | Taille Clé | Performance | Sécurité | Usage |
|------------|-----------|-------------|----------|-------|
| **AES-128** | 128 bits | Rapide (+0%) | Élevée | Performance critique |
| **AES-192** | 192 bits | Moyen (+5%) | Très élevée | Compromis |
| **AES-256** | 256 bits | Lent (+10%) | Maximale | Compliance stricte |

💡 **Recommandation** : AES-256 par défaut, AES-128 si performance critique et compliance permise.

---

## Plugins de Gestion de Clés

MariaDB supporte plusieurs plugins pour gérer la Master Encryption Key (MEK) :

### 1. File Key Management Plugin (Développement/Test)

**Principe** : MEK stockée dans un fichier local chiffré avec mot de passe.

```ini
# my.cnf
[mysqld]
# Charger plugin
plugin-load-add=file_key_management

# Fichier contenant les clés
file-key-management-filename=/etc/mysql/encryption/keyfile.enc

# Méthode de chiffrement du fichier
file-key-management-encryption-algorithm=AES_CBC

# Fichier contenant le mot de passe pour déchiffrer keyfile.enc
file-key-management-filekey=/etc/mysql/encryption/keyfile.key
```

**Création du fichier de clés** :
```bash
# Générer clé de chiffrement (256 bits)
openssl rand -hex 32 > /etc/mysql/encryption/master.key

# Format du fichier keyfile (avant chiffrement)
# <key_id>;<hex_encoded_key>
# Exemple : 1 clé
echo "1;$(openssl rand -hex 32)" > /etc/mysql/encryption/keyfile

# Exemple : Multiple versions de clés (rotation)
cat > /etc/mysql/encryption/keyfile << EOF
1;$(openssl rand -hex 32)
2;$(openssl rand -hex 32)
3;$(openssl rand -hex 32)
EOF

# Chiffrer le keyfile avec mot de passe
# Mot de passe dans fichier séparé
openssl rand -hex 16 > /etc/mysql/encryption/keyfile.key

# Chiffrer keyfile avec clé
openssl enc -aes-256-cbc -md sha256 \
  -pass file:/etc/mysql/encryption/keyfile.key \
  -in /etc/mysql/encryption/keyfile \
  -out /etc/mysql/encryption/keyfile.enc

# Supprimer fichier non chiffré
rm /etc/mysql/encryption/keyfile

# Sécuriser permissions
chmod 600 /etc/mysql/encryption/keyfile.enc
chmod 600 /etc/mysql/encryption/keyfile.key
chown mysql:mysql /etc/mysql/encryption/keyfile.*
```

**Avantages** :
- ✅ Simple à mettre en place
- ✅ Pas de dépendance externe
- ✅ Gratuit

**Inconvénients** :
- ⚠️ Clé stockée localement (risque si serveur compromis)
- ⚠️ Gestion manuelle des clés
- ⚠️ Rotation complexe
- ❌ **Non recommandé pour production** (sauf contraintes spécifiques)

### 2. AWS Key Management Service (KMS) Plugin

**Principe** : MEK gérée par AWS KMS, appels API pour déchiffrement.

```ini
# my.cnf
[mysqld]
# Charger plugin AWS KMS
plugin-load-add=aws_key_management

# ARN de la clé KMS
aws-key-management-master-key-id=arn:aws:kms:eu-west-1:123456789012:key/abcd1234-5678-90ef-ghij-klmnopqrstuv

# Région AWS
aws-key-management-region=eu-west-1

# Rotation automatique des clés (optionnel)
aws-key-management-rotate-key=1

# Log niveau (optionnel)
aws-key-management-log-level=INFO
```

**Configuration IAM** :
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowMariaDBEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:eu-west-1:123456789012:key/abcd1234-*"
    }
  ]
}
```

**Authentification** :
```bash
# Option 1 : IAM Role (EC2, RDS)
# Automatique si instance EC2 avec rôle IAM approprié

# Option 2 : Credentials file
cat > ~/.aws/credentials << EOF
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF

# Option 3 : Variables d'environnement
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Avantages** :
- ✅ Clés gérées par AWS (jamais sur serveur MariaDB)
- ✅ Rotation automatique des clés
- ✅ Audit trail (CloudTrail)
- ✅ Intégration IAM
- ✅ Haute disponibilité AWS

**Inconvénients** :
- ⚠️ Coût AWS KMS (~$1/clé/mois + $0.03/10K requêtes)
- ⚠️ Latence réseau (appels API)
- ⚠️ Dépendance à AWS
- ⚠️ Nécessite connectivité internet/VPC endpoint

### 3. HashiCorp Vault Plugin

**Principe** : MEK stockée dans Vault, récupérée via API HTTP.

```ini
# my.cnf
[mysqld]
# Charger plugin Vault
plugin-load-add=hashicorp_key_management

# URL du serveur Vault
hashicorp-key-management-vault-url=https://vault.example.com:8200

# Token d'authentification (ou AppRole)
hashicorp-key-management-token=s.iyNUhq8Ov4hIAx6snw5mB2nL

# Chemin du secret dans Vault
hashicorp-key-management-vault-mount-point=secret
hashicorp-key-management-secret-name=mariadb/encryption-key

# Cache timeout (secondes)
hashicorp-key-management-cache-timeout=60

# Vérification certificat SSL
hashicorp-key-management-check-kv-version=true
```

**Configuration Vault** :
```bash
# Créer clé dans Vault
vault kv put secret/mariadb/encryption-key \
  key1=$(openssl rand -hex 32) \
  key2=$(openssl rand -hex 32)

# Créer policy pour MariaDB
cat > mariadb-encryption-policy.hcl << EOF
path "secret/data/mariadb/encryption-key" {
  capabilities = ["read"]
}
EOF

vault policy write mariadb-encryption mariadb-encryption-policy.hcl

# Créer token pour MariaDB
vault token create -policy=mariadb-encryption -ttl=720h
```

**Avantages** :
- ✅ Gestion centralisée des secrets
- ✅ Rotation automatique possible
- ✅ Audit détaillé
- ✅ Support multi-cloud
- ✅ Dynamic secrets

**Inconvénients** :
- ⚠️ Infrastructure Vault à maintenir
- ⚠️ Complexité configuration
- ⚠️ Latence réseau
- ⚠️ Dépendance à Vault

### 4. KMIP Plugin (Enterprise Key Management)

**Principe** : Protocole standard KMIP pour HSM et key managers enterprise.

```ini
# my.cnf
[mysqld]
plugin-load-add=kmip_key_management

# Serveur KMIP
kmip-key-management-server-addr=kmip.example.com
kmip-key-management-server-port=5696

# Certificats SSL client
kmip-key-management-client-certificate=/etc/mysql/kmip/client.pem
kmip-key-management-client-key=/etc/mysql/kmip/client-key.pem

# CA certificate
kmip-key-management-ca-certificate=/etc/mysql/kmip/ca.pem

# Identifier de la clé
kmip-key-management-key-id=mariadb-encryption-key-1
```

**Compatibilité** :
- ✅ Thales CipherTrust Manager
- ✅ Gemalto KeySecure
- ✅ IBM Security Key Lifecycle Manager
- ✅ Townsend Security Alliance Key Manager

**Avantages** :
- ✅ Standard industriel (OASIS)
- ✅ Support HSM (Hardware Security Modules)
- ✅ Haute sécurité (FIPS 140-2)
- ✅ Audit enterprise-grade

**Inconvénients** :
- ⚠️ Coût élevé (licenses HSM)
- ⚠️ Complexité infrastructure
- ⚠️ Expertise requise

---

## Configuration du Chiffrement

### Activer le Chiffrement Global

```ini
# my.cnf - Configuration recommandée production
[mysqld]
# --- Plugin de gestion de clés ---
plugin-load-add=aws_key_management
aws-key-management-master-key-id=arn:aws:kms:eu-west-1:123456:key/abc123
aws-key-management-region=eu-west-1

# --- Chiffrement InnoDB ---
# Chiffrer toutes les nouvelles tables par défaut
innodb-encrypt-tables=ON

# Chiffrer les logs (redo, undo)
innodb-encrypt-log=ON

# Chiffrer tablespaces temporaires
innodb-encrypt-temporary-tables=ON

# --- Algorithme ---
innodb-encryption-algorithm=AES-256

# --- Threads de chiffrement ---
# Pour re-chiffrer tables existantes en arrière-plan
innodb-encryption-threads=4

# --- Rotation des clés ---
# Âge maximum clé avant rotation (jours)
innodb-encryption-rotate-key-age=30

# --- Binary logs (optionnel) ---
encrypt-binlog=ON
```

### Chiffrer une Table Spécifique

```sql
-- Créer table chiffrée
CREATE TABLE sensitive_data (
  id INT PRIMARY KEY AUTO_INCREMENT,
  ssn VARCHAR(11),
  credit_card VARCHAR(19),
  medical_record TEXT
) ENCRYPTED=YES;

-- Chiffrer table existante
ALTER TABLE customers ENCRYPTED=YES;

-- Déchiffrer table (si autorisé par politique)
ALTER TABLE non_sensitive ENCRYPTED=NO;

-- Vérifier état chiffrement
SELECT 
  TABLE_SCHEMA,
  TABLE_NAME,
  CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_NAME = 'sensitive_data';
-- CREATE_OPTIONS: ENCRYPTED=YES
```

### Chiffrer Tablespace Système

```sql
-- Chiffrer tablespace système (nécessite redémarrage)
-- my.cnf:
-- innodb-encrypt-tables=FORCE
-- Toutes tables DOIVENT être chiffrées

-- Vérifier tablespaces chiffrés
SELECT 
  SPACE,
  NAME,
  ENCRYPTION
FROM information_schema.INNODB_TABLESPACES
WHERE ENCRYPTION = 'Y';
```

---

## Impact sur les Performances

### Benchmarks Typiques

**Configuration test** :
- Hardware : 8 vCPU, 32 GB RAM, SSD NVMe
- Dataset : 10M lignes, 5 GB de données
- Workload : Mix OLTP (70% SELECT, 20% INSERT, 10% UPDATE)

| Métrique | Non Chiffré | AES-128 | AES-256 | Overhead |
|----------|-------------|---------|---------|----------|
| **SELECT** | 1250 req/s | 1190 req/s | 1150 req/s | -8% |
| **INSERT** | 850 req/s | 780 req/s | 740 req/s | -13% |
| **UPDATE** | 620 req/s | 570 req/s | 540 req/s | -13% |
| **CPU Usage** | 35% | 42% | 48% | +37% |
| **IOPS** | 2400 | 2380 | 2370 | -1% |
| **Latency p95** | 12ms | 14ms | 15ms | +25% |

**Observations** :
- ⚠️ CPU overhead significatif (+30-40%)
- ✅ Impact I/O négligeable (chiffrement en mémoire)
- ⚠️ Latence augmentée (+20-30%)
- 💡 Hardware moderne avec AES-NI réduit overhead drastiquement

### Impact selon Hardware

```sql
-- Vérifier support AES-NI (accélération matérielle)
-- Linux :
cat /proc/cpuinfo | grep aes
-- Si "aes" présent → AES-NI supporté

-- Impact avec AES-NI :
```

| Hardware | Overhead AES-256 | Notes |
|----------|------------------|-------|
| **CPU moderne + AES-NI** | 5-10% | Intel depuis 2010, AMD Ryzen |
| **CPU sans AES-NI** | 30-50% | Ancien hardware |
| **ARM avec Crypto Extensions** | 8-12% | AWS Graviton, Apple Silicon |
| **VM sans pass-through** | 25-40% | AES-NI non exposé à VM |

💡 **Recommandation** : Vérifier support AES-NI avant activation encryption en production.

### Optimisations

```ini
# my.cnf - Optimiser performance avec encryption
[mysqld]
# Augmenter threads de chiffrement
innodb-encryption-threads=8  # = Nb CPU cores

# Buffer pool plus large (compensation CPU overhead)
innodb-buffer-pool-size=40G  # +25% vs sans encryption

# Thread pool pour gérer concurrence
thread-handling=pool-of-threads
thread-pool-size=8

# AES-128 si performance critique et compliance OK
innodb-encryption-algorithm=AES-128
```

---

## Cas d'Usage et Compliance

### 1. Conformité RGPD (Données Personnelles)

```sql
-- Table utilisateurs avec données personnelles
CREATE TABLE users (
  user_id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255),
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  phone VARCHAR(20),
  birth_date DATE,
  address TEXT,
  
  -- Métadonnées RGPD
  consent_marketing BOOLEAN,
  consent_date TIMESTAMP,
  data_retention_until DATE,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
  
) ENCRYPTED=YES
  COMMENT='Données personnelles RGPD - Chiffrement obligatoire';

-- Audit : Vérifier toutes tables PII sont chiffrées
SELECT 
  TABLE_SCHEMA,
  TABLE_NAME,
  TABLE_COMMENT,
  CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_COMMENT LIKE '%RGPD%'
  AND CREATE_OPTIONS NOT LIKE '%ENCRYPTED=YES%';
-- Alerte si résultat non vide !
```

### 2. PCI-DSS (Données Bancaires)

```sql
-- Exigence PCI-DSS 3.4 : Chiffrement données cartes
CREATE TABLE payment_methods (
  payment_id INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT,
  
  -- Données PCI (JAMAIS en clair même chiffrées - tokenisation recommandée)
  card_token VARCHAR(64),  -- Token du processeur de paiement
  card_last4 VARCHAR(4),   -- Derniers 4 chiffres (affichage)
  card_brand ENUM('VISA','MASTERCARD','AMEX','DISCOVER'),
  expiry_month TINYINT,
  expiry_year SMALLINT,
  
  -- Adresse facturation
  billing_address TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX idx_customer (customer_id)
  
) ENCRYPTED=YES
  COMMENT='PCI-DSS Level 1 - Chiffrement + Tokenisation obligatoire';

-- IMPORTANT : Même avec encryption, respecter PCI-DSS :
-- 1. Jamais stocker CVV/CVC
-- 2. Jamais stocker numéro complet (utiliser token)
-- 3. Chiffrement at rest + in transit (SSL/TLS)
-- 4. Logs audités
```

### 3. HIPAA (Dossiers Médicaux)

```sql
-- Dossiers patients - Conformité HIPAA
CREATE TABLE patient_records (
  record_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  patient_id INT,
  
  -- Identifiants protégés (PHI - Protected Health Information)
  ssn_encrypted VARBINARY(255),  -- SSN chiffré au niveau app aussi
  medical_record_number VARCHAR(50),
  
  -- Informations médicales
  diagnosis TEXT,
  treatment_plan TEXT,
  medications JSON,
  lab_results JSON,
  
  -- Dates importantes
  admission_date DATE,
  discharge_date DATE,
  
  -- Audit HIPAA
  accessed_by VARCHAR(100),
  accessed_at TIMESTAMP,
  access_reason VARCHAR(255),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX idx_patient (patient_id),
  INDEX idx_access_audit (accessed_at, accessed_by)
  
) ENCRYPTED=YES
  COMMENT='HIPAA Protected Health Information - Encryption at rest mandatory';

-- Trigger d'audit HIPAA
DELIMITER $$
CREATE TRIGGER patient_records_access_log
AFTER SELECT ON patient_records
FOR EACH ROW
BEGIN
  INSERT INTO hipaa_access_log (
    record_id, 
    user_id, 
    access_timestamp, 
    access_type
  ) VALUES (
    NEW.record_id,
    USER(),
    NOW(),
    'READ'
  );
END$$
DELIMITER ;
```

### 4. Backups Chiffrés

```bash
# Mariabackup avec encryption
mariabackup --backup \
  --target-dir=/backup/full \
  --user=backup_user \
  --password=SecurePass123 \
  # Backups automatiquement chiffrés si tables sources chiffrées
  
# Vérifier backup chiffré
mariabackup --prepare \
  --target-dir=/backup/full
  
# Les fichiers .ibd dans backup sont chiffrés
# Nécessite clés de chiffrement pour restore

# Alternative : Chiffrer backup au niveau filesystem
mariabackup --backup --stream=xbstream | \
  openssl enc -aes-256-cbc -salt -out backup.xbstream.enc

# Restore
openssl enc -aes-256-cbc -d -in backup.xbstream.enc | \
  mbstream -x -C /var/lib/mysql
```

---

## Rotation des Clés

### Rotation Manuelle de la Master Key

```bash
# Avec file_key_management

# 1. Générer nouvelle clé (key_id=2)
echo "2;$(openssl rand -hex 32)" >> /etc/mysql/encryption/keyfile

# 2. Re-chiffrer keyfile
openssl enc -aes-256-cbc -md sha256 \
  -pass file:/etc/mysql/encryption/keyfile.key \
  -in /etc/mysql/encryption/keyfile \
  -out /etc/mysql/encryption/keyfile.enc

# 3. Indiquer à MariaDB d'utiliser nouvelle clé
# my.cnf:
# file-key-management-encryption-key-id=2

# 4. Redémarrer MariaDB
systemctl restart mariadb

# 5. Re-chiffrer tables avec nouvelle clé (background)
# Automatique si innodb-encryption-threads > 0
# et innodb-encryption-rotate-key-age configuré
```

### Rotation Automatique (AWS KMS)

```sql
-- Avec AWS KMS, rotation gérée par AWS
-- Configuration dans AWS Console :
-- - Enable automatic key rotation (yearly)
-- - MariaDB utilise automatiquement nouvelle version

-- Vérifier dernière rotation
-- AWS CLI :
aws kms describe-key \
  --key-id arn:aws:kms:eu-west-1:123456:key/abc123 \
  --query 'KeyMetadata.{Created:CreationDate,Rotation:KeyRotationEnabled}'
```

### Surveillance Rotation

```sql
-- Requête : Tables nécessitant rotation
SELECT 
  SPACE,
  NAME,
  ENCRYPTION_SCHEME,
  CURRENT_KEY_VERSION
FROM information_schema.INNODB_TABLESPACES_ENCRYPTION
WHERE CURRENT_KEY_VERSION < (
  SELECT MAX(CURRENT_KEY_VERSION) 
  FROM information_schema.INNODB_TABLESPACES_ENCRYPTION
);

-- Si résultats → Tables avec anciennes clés
-- innodb-encryption-threads re-chiffrera automatiquement
```

---

## Sécurité et Best Practices

### 1. Séparation des Clés

```bash
# ✅ Stocker clés HORS du serveur database
# Option 1 : Vault/KMS externe
# Option 2 : Filesystem network monté read-only

# ❌ NE JAMAIS stocker clé non chiffrée dans /var/lib/mysql
# ❌ NE JAMAIS commiter clés dans Git
# ❌ NE JAMAIS logger clés dans error log
```

### 2. Principe du Moindre Privilège

```sql
-- Utilisateur backup : Peut lire données chiffrées
GRANT SELECT, RELOAD, LOCK TABLES ON *.* TO 'backup_user'@'localhost';

-- Utilisateur app : Peut lire/écrire mais pas gérer encryption
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'%';

-- DBA : Peut gérer encryption
GRANT SUPER, ENCRYPTION_KEY_ADMIN ON *.* TO 'dba_user'@'localhost';
```

### 3. Monitoring et Alertes

```sql
-- Dashboard encryption status
CREATE VIEW encryption_status AS
SELECT 
  TABLE_SCHEMA,
  TABLE_NAME,
  CASE 
    WHEN CREATE_OPTIONS LIKE '%ENCRYPTED=YES%' THEN 'ENCRYPTED'
    ELSE 'PLAINTEXT'
  END AS encryption_status,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
ORDER BY size_mb DESC;

-- Alerte : Tables sensibles non chiffrées
SELECT * FROM encryption_status 
WHERE encryption_status = 'PLAINTEXT'
  AND TABLE_NAME IN ('users', 'payments', 'patient_records');
```

### 4. Documentation et Procédures

```markdown
# Runbook : Perte de Clé de Chiffrement

## Prévention
- Backup clés dans coffre-fort sécurisé (physique)
- Backup clés dans second KMS (disaster recovery)
- Procédure testée trimestriellement

## Symptômes
- MariaDB refuse de démarrer
- Error log : "Encryption key not found"
- Tables .ibd inaccessibles

## Procédure de Recovery
1. Vérifier backup des clés existe
2. Restaurer keyfile.enc depuis backup
3. Vérifier permissions (600, mysql:mysql)
4. Redémarrer MariaDB
5. Vérifier tables accessibles
6. Post-mortem : Comment la clé a été perdue ?

## Escalation
- Si backup clé inexistant → DONNÉES PERDUES
- Contacter : security-team@company.com
- Notifier : CISO, Legal (RGPD breach possible)
```

---

## ✅ Points clés à retenir

### Architecture et Concepts
- ✅ **Encryption at rest** : Chiffre données sur disque, transparent pour app
- ✅ **3 niveaux de clés** : MEK (master) → TEK (table) → Page encryption
- ✅ **AES-256** : Algorithme recommandé (AES-128 si performance critique)
- ✅ **Ce qui est chiffré** : Tables, tablespaces, redo/undo logs, binary logs

### Plugins de Gestion de Clés
- ✅ **file_key_management** : Simple, dev/test uniquement
- ✅ **aws_key_management** : Production AWS, rotation automatique
- ✅ **hashicorp_vault** : Gestion centralisée, multi-cloud
- ✅ **kmip** : Enterprise, HSM, FIPS 140-2

### Configuration
- ✅ `innodb-encrypt-tables=ON` : Chiffrer nouvelles tables par défaut
- ✅ `innodb-encrypt-log=ON` : Chiffrer redo logs
- ✅ `innodb-encryption-threads=4` : Re-chiffrement arrière-plan
- ✅ `innodb-encryption-rotate-key-age=30` : Rotation clés (jours)

### Performance
- ⚠️ **CPU overhead** : +30-50% sans AES-NI, +5-10% avec AES-NI
- ✅ **I/O impact** : Négligeable (chiffrement en mémoire)
- ⚠️ **Latence** : +20-30% typique
- 💡 **Vérifier AES-NI** : `cat /proc/cpuinfo | grep aes`

### Compliance
- ✅ **RGPD** : Chiffrement données personnelles recommandé
- ✅ **PCI-DSS** : Chiffrement données carte obligatoire (+ tokenisation)
- ✅ **HIPAA** : Encryption at rest + audit trail
- ✅ **Backups** : Automatiquement chiffrés si source chiffrée

### Best Practices
- ✅ Stocker clés HORS du serveur database
- ✅ Rotation régulière (30-90 jours recommandé)
- ✅ Monitoring encryption status
- ✅ Tester procédure recovery clés
- ✅ Documenter (runbooks, DRP)
- ⚠️ Encryption ≠ solution miracle (défense en profondeur)

### Limitations
- ❌ N'empêche PAS SQL injection
- ❌ N'empêche PAS accès via credentials valides
- ❌ N'empêche PAS exfiltration mémoire
- 💡 Compléter avec SSL/TLS, firewall, MFA

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [Data at Rest Encryption](https://mariadb.com/kb/en/data-at-rest-encryption/) - Guide complet
- 📖 [File Key Management Plugin](https://mariadb.com/kb/en/file-key-management-encryption-plugin/)
- 📖 [AWS KMS Plugin](https://mariadb.com/kb/en/aws-key-management-encryption-plugin/)
- 📖 [HashiCorp Vault Plugin](https://mariadb.com/kb/en/hashicorp-key-management-plugin/)
- 📖 [Encryption Key Management](https://mariadb.com/kb/en/encryption-key-management/)

### Sécurité et Compliance
- 📝 [GDPR Compliance with MariaDB](https://mariadb.com/resources/blog/gdpr-compliance/)
- 📝 [PCI-DSS Database Encryption](https://www.pcisecuritystandards.org/)
- 📝 [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/)

### Performance et Tuning
- 📝 [Encryption Performance Impact](https://mariadb.com/resources/blog/encryption-performance/)
- 📝 [AES-NI Acceleration](https://mariadb.com/kb/en/aes-ni-support/)

### Cloud Providers
- 🔗 [AWS KMS](https://aws.amazon.com/kms/) - Key Management Service
- 🔗 [Azure Key Vault](https://azure.microsoft.com/services/key-vault/)
- 🔗 [GCP Cloud KMS](https://cloud.google.com/security-key-management)
- 🔗 [HashiCorp Vault](https://www.vaultproject.io/)

---

## ➡️ Sous-sections suivantes

### **18.7.1 Data at Rest Encryption**
Configuration détaillée du chiffrement des tables, tablespaces, et logs avec exemples de migration.

### **18.7.2 Key Management**
Gestion avancée des clés, rotation, HSM, et stratégies enterprise.

---


⏭️ [Data at Rest Encryption](/18-fonctionnalites-avancees/07.1-data-at-rest-encryption.md)

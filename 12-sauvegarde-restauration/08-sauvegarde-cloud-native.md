🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.8 Sauvegarde cloud-native

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Sections 12.1-12.7, Cloud computing (AWS/Azure/GCP), Kubernetes

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes des sauvegardes cloud-native
- **Comparer** les solutions Object Storage (S3, Azure Blob, GCS)
- **Concevoir** des architectures de backup multi-cloud et hybrides
- **Implémenter** des stratégies de sauvegarde dans Kubernetes
- **Optimiser** les coûts de stockage cloud (lifecycle policies, tiers)
- **Sécuriser** les backups cloud (chiffrement, IAM, compliance)
- **Gérer** la migration des backups vers le cloud
- **Évaluer** les performances réseau et temps de restauration

---

## Introduction

Les sauvegardes cloud-native transforment radicalement la gestion des backups en offrant **scalabilité illimitée**, **durabilité exceptionnelle** et **automatisation poussée**. Contrairement aux approches traditionnelles (NAS, SAN, bandes), le cloud apporte des avantages structurels majeurs.

### Évolution des infrastructures de backup

```
┌──────────────────────────────────────────────────────┐
│         Évolution des stratégies de backup           │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Génération 1 : On-premise traditionnel (1990-2010)  │
│  ├─ Bandes magnétiques (LTO)                         │
│  ├─ SAN/NAS dédié                                    │
│  ├─ Gestion manuelle                                 │
│  └─ Limitations : Capacité, coût, complexité         │
│                                                      │
│  Génération 2 : Virtualisation (2010-2015)           │
│  ├─ Snapshots VM (VMware, Hyper-V)                   │
│  ├─ Deduplication                                    │
│  ├─ Automatisation partielle                         │
│  └─ Limitations : Vendor lock-in, scalabilité        │
│                                                      │
│  Génération 3 : Cloud hybride (2015-2020)            │
│  ├─ Backup local + réplication cloud                 │
│  ├─ Object Storage (S3, Blob)                        │
│  ├─ Lifecycle management                             │
│  └─ Limitations : Complexité réseau, coûts transfert │
│                                                      │
│  Génération 4 : Cloud-native (2020+) ⭐              │
│  ├─ Kubernetes-native (VolumeSnapshots, Velero)      │
│  ├─ Multi-cloud par design                           │
│  ├─ Infrastructure as Code                           │
│  ├─ Observabilité intégrée                           │
│  └─ Automatisation complète                          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Qu'est-ce qu'une sauvegarde cloud-native ?

Une sauvegarde **cloud-native** respecte les principes suivants :

**1. API-First** : Gestion complète via API
```bash
# Exemple : Backup via AWS CLI
aws s3 sync /backups/mariadb s3://my-backups/mariadb/ \
  --storage-class INTELLIGENT_TIERING
```

**2. Infrastructure as Code** : Déclaratif (Terraform, CloudFormation)
```hcl
# Terraform : Bucket S3 avec lifecycle
resource "aws_s3_bucket" "backups" {
  bucket = "mariadb-backups"
  
  lifecycle_rule {
    enabled = true
    
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365
    }
  }
}
```

**3. Scalabilité élastique** : Capacité illimitée, pas de provisionnement
```
Traditional NAS : Acheter +10TB ? → Délai 2-4 semaines, CAPEX
Cloud Storage   : Besoin +10TB ? → Immédiat, OPEX
```

**4. Durabilité exceptionnelle** : 99.999999999% (11 9's)
```
Durabilité S3 Standard : 1 objet perdu tous les 10 000 ans (pour 10M objets)
vs
Durabilité RAID 6 : 1 perte possible tous les 5-10 ans
```

**5. Automatisation native** : Lifecycle, versioning, réplication
```
Lifecycle automatique :
├─ J+0 à J+30   : STANDARD (accès fréquent)
├─ J+30 à J+90  : INTELLIGENT_TIERING (auto-optimisation)
├─ J+90 à J+365 : GLACIER (archivage)
└─ J+365+       : Suppression automatique
```

---

## Avantages des sauvegardes cloud

### 1. Élimination du CAPEX

**Modèle traditionnel** :
```
Achat NAS 100 To : 50 000€ + maintenance 8 000€/an
Remplacement tous les 5 ans
Surcapacité de 40% (pour croissance)
Total 5 ans : ~90 000€
```

**Modèle cloud** :
```
S3 Standard : 0,023€/GB/mois
Utilisation réelle : 50 To (pas de surcapacité)
Total 5 ans : 50 000 × 0,023 × 12 × 5 = 69 000€
+ Pas de maintenance, pas de remplacement
+ Scalabilité instantanée
```

### 2. Durabilité et disponibilité

| Critère | On-premise RAID 6 | AWS S3 Standard | Azure Blob (GRS) | GCP Standard |
|---------|-------------------|-----------------|------------------|--------------|
| **Durabilité annuelle** | 99.9% - 99.99% | 99.999999999% | 99.99999999999999% | 99.999999999% |
| **Disponibilité** | 99.5% - 99.9% | 99.99% | 99.9% (RA-GRS: 99.99%) | 99.95% |
| **Réplication** | Locale | Multi-AZ | Multi-région | Multi-région |
| **Point de défaillance** | Datacenter | Aucun (région) | Aucun (géo) | Aucun (région) |

💡 **Implication** : Avec S3, probabilité de perdre un backup : 0,00000001% par an.

### 3. Scalabilité illimitée

```
┌────────────────────────────────────────────┐
│     Croissance base de données             │
├────────────────────────────────────────────┤
│                                            │
│  2020 : 500 GB                             │
│  2021 : 1.2 TB (+140%)                     │
│  2022 : 3.5 TB (+192%)                     │
│  2023 : 8 TB (+129%)                       │
│  2024 : 15 TB (+88%)                       │
│                                            │
│  On-premise : Achat NAS tous les 18 mois   │
│  Cloud : Aucune action, scaling automatique│
│                                            │
└────────────────────────────────────────────┘
```

### 4. Géo-redondance native

```
AWS S3 Replication (CRR) :
┌─────────────┐        Réplication       ┌─────────────┐
│  us-east-1  │ ────────────────────────►│  eu-west-1  │
│  (Primary)  │      automatique         │  (DR site)  │
└─────────────┘     < 15 minutes         └─────────────┘

Azure Blob Storage (GRS) :
┌─────────────┐     Réplication sync     ┌─────────────┐
│ West Europe │ ◄──────────────────────► │ North Europe│
│  (Primary)  │   + Async vers paired    │    (Pair)   │
└─────────────┘      region              └─────────────┘
```

### 5. Sécurité et conformité

**Chiffrement multi-couches** :
```
├─ At-rest : AES-256 (SSE-S3, SSE-KMS, SSE-C)
├─ In-transit : TLS 1.3
├─ Client-side : Chiffrement avant upload
└─ Key management : KMS, HSM, bring-your-own-key
```

**Conformité** :
- ✅ ISO 27001, SOC 2, PCI-DSS
- ✅ GDPR (data residency, right to erasure)
- ✅ HIPAA (healthcare data)
- ✅ Audit logs (CloudTrail, Azure Monitor, Cloud Audit Logs)

---

## Architectures cloud-native

### Architecture 1 : Cloud-first (100% cloud)

```
┌──────────────────────────────────────────────────────┐
│         Architecture Cloud-First                     │
├──────────────────────────────────────────────────────┤
│                                                      │
│   MariaDB (EC2/VM)                                   │
│         │                                            │
│         │ Daily Full + Hourly Inc                    │
│         ▼                                            │
│   ┌─────────────────┐                                │
│   │  Local staging  │  (NVMe temporaire)             │
│   │   /backups/     │                                │
│   └─────────────────┘                                │
│         │                                            │
│         │ Upload immédiat                            │
│         ▼                                            │
│   ┌─────────────────────────────────────┐            │
│   │    S3 / Azure Blob / GCS            │            │
│   │  ┌───────────────────────────┐      │            │
│   │  │ STANDARD (30 jours)       │      │            │
│   │  └───────────────────────────┘      │            │
│   │  ┌───────────────────────────┐      │            │
│   │  │ GLACIER (30-365 jours)    │      │            │
│   │  └───────────────────────────┘      │            │
│   │  ┌───────────────────────────┐      │            │
│   │  │ DEEP_ARCHIVE (365+ jours) │      │            │
│   │  └───────────────────────────┘      │            │
│   └─────────────────────────────────────┘            │
│         │                                            │
│         │ Réplication                                │
│         ▼                                            │
│   ┌─────────────────┐                                │
│   │  DR Site (autre │  (Cross-region replication)    │
│   │     région)     │                                │
│   └─────────────────┘                                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Pas d'infrastructure on-premise à gérer
- ✅ Scalabilité maximale
- ✅ Coûts prévisibles (OPEX)

**Inconvénients** :
- ⚠️ Dépendance réseau (bandwidth, latence)
- ⚠️ Coûts egress lors de restaurations volumineuses
- ⚠️ Vendor lock-in potentiel

### Architecture 2 : Hybride (On-premise + Cloud)

```
┌──────────────────────────────────────────────────────┐
│          Architecture Hybride                        │
├──────────────────────────────────────────────────────┤
│                                                      │
│   MariaDB (On-premise)                               │
│         │                                            │
│         │ Backup                                     │
│         ▼                                            │
│   ┌─────────────────┐                                │
│   │  NAS Local      │  ← Rétention 7 jours           │
│   │  (Backup 1°)    │     Restauration rapide        │
│   └─────────────────┘                                │
│         │                                            │
│         │ Réplication async                          │
│         ▼                                            │
│   ┌─────────────────────────────────────┐            │
│   │    S3 / Azure Blob                  │            │
│   │  (Backup 2° - DR)                   │            │
│   │  ┌───────────────────────────┐      │            │
│   │  │ Rétention 30+ jours       │      │            │
│   │  │ Protection ransomware     │      │            │
│   │  │ Object Lock (WORM)        │      │            │
│   │  └───────────────────────────┘      │            │
│   └─────────────────────────────────────┘            │
│                                                      │
│   Avantages :                                        │
│   ├─ Restauration rapide (local NAS)                 │
│   ├─ Protection hors-site (cloud)                    │
│   └─ Optimisation coûts (moins d'egress)             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Cas d'usage** :
- Bases critiques avec RTO < 2h (restauration locale)
- Compliance nécessitant copies hors-site
- Migration progressive vers cloud

### Architecture 3 : Kubernetes-native

```
┌──────────────────────────────────────────────────────┐
│       Architecture Kubernetes-native                 │
├──────────────────────────────────────────────────────┤
│                                                      │
│   MariaDB StatefulSet                                │
│   ├─ mariadb-0 (Primary)                             │
│   └─ mariadb-1 (Replica)                             │
│         │                                            │
│         │ PersistentVolumeClaim                      │
│         ▼                                            │
│   ┌─────────────────────────────┐                    │
│   │  CSI Driver (EBS/AzureDisk) │                    │
│   └─────────────────────────────┘                    │
│         │                                            │
│   ┌─────┴──────────────────────┐                     │
│   │                             │                    │
│   ▼                             ▼                    │
│ VolumeSnapshot            CronJob (Backup)           │
│ (point-in-time)          ├─ mariabackup              │
│ ├─ Hourly                │   └─► S3                  │
│ ├─ Retention 24h         └─ Binary logs → S3         │
│ └─ Fast restore                                      │
│                                                      │
│   Orchestration : Velero                             │
│   ├─ Backup namespaces complets                      │
│   ├─ Application-consistent                          │
│   └─ Multi-cloud restore                             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Technologies** :
- **CSI Snapshots** : Point-in-time natif Kubernetes
- **Velero** : Backup/restore Kubernetes resources + volumes
- **Stash/Kasten K10** : Solutions commerciales avancées

---

## Comparaison des fournisseurs cloud

### Object Storage : Fonctionnalités

| Fonctionnalité | AWS S3 | Azure Blob Storage | Google Cloud Storage | Compatibilité S3 |
|----------------|--------|-------------------|---------------------|------------------|
| **API** | S3 API (standard de facto) | Blob API + S3-compatible | GCS API + S3-compatible | ✅ / ⚠️ / ⚠️ |
| **Durabilité** | 99.999999999% (11 9's) | 99.99999999999999% (16 9's, GRS) | 99.999999999% (11 9's) | - |
| **Disponibilité** | 99.99% (Standard) | 99.9% (Hot), 99.99% (RA-GRS) | 99.95% (Standard) | - |
| **Tiers de stockage** | 6+ tiers | 3 tiers | 4 classes | ✅ |
| **Lifecycle management** | ✅ Avancé | ✅ Avancé | ✅ Avancé | ✅ |
| **Versioning** | ✅ | ✅ | ✅ | ✅ |
| **Object Lock (WORM)** | ✅ | ✅ (Immutable storage) | ✅ (Retention policies) | ✅ |
| **Chiffrement** | SSE-S3, SSE-KMS, SSE-C | SSE, CMK | CMEK, CSEK | ✅ |
| **Réplication cross-region** | ✅ CRR | ✅ Object replication | ✅ Turbo Replication | ✅ |
| **IAM granulaire** | ✅ | ✅ RBAC + SAS | ✅ IAM | ✅ |

### Tarification (Région EU, décembre 2024)

| Service | Stockage (€/GB/mois) | GET/PUT (€/1000 req) | Transfert sortie (€/GB) |
|---------|---------------------|---------------------|------------------------|
| **AWS S3 Standard** | 0,023 | 0,0004 / 0,005 | 0,09 (après 100 GB) |
| **S3 Intelligent-Tiering** | 0,023 - 0,0045 | 0,0004 / 0,005 | 0,09 |
| **S3 Glacier Instant** | 0,004 | 0,01 / 0,02 | 0,09 |
| **S3 Glacier Deep Archive** | 0,00099 | 0,02 / 0,05 | 0,09 |
| **Azure Blob Hot** | 0,0184 | 0,004 / 0,05 | 0,087 |
| **Azure Blob Cool** | 0,01 | 0,01 / 0,10 | 0,087 |
| **Azure Blob Archive** | 0,002 | 0,05 / 0,10 | 0,087 |
| **GCS Standard** | 0,020 | 0,004 / 0 | 0,12 |
| **GCS Nearline** | 0,010 | 0,01 / 0,01 | 0,12 |
| **GCS Coldline** | 0,004 | 0,05 / 0,05 | 0,12 |
| **GCS Archive** | 0,0012 | 0,05 / 0,05 | 0,12 |

💡 **Exemple de coût** : Backup 5 TB/mois pendant 1 an

```
AWS S3 Intelligent-Tiering :
├─ Stockage : 5000 GB × 0,015€ (moyenne) × 12 = 900€
├─ PUT (1 daily upload) : 30 × 0,005€ × 12 = 1,8€
├─ GET (1 test/mois) : 12 × 0,0004€ = 0,005€
└─ Total annuel : ~902€

Azure Blob Cool :
├─ Stockage : 5000 GB × 0,01€ × 12 = 600€
├─ PUT/GET : ~2€
└─ Total annuel : ~602€

Google Cloud Nearline :
├─ Stockage : 5000 GB × 0,01€ × 12 = 600€
├─ PUT : Gratuit
└─ Total annuel : ~600€
```

### Choix du fournisseur

**Critères de décision** :

```
┌─────────────────────────────────────────────┐
│      Matrice de décision cloud              │
├─────────────────────────────────────────────┤
│                                             │
│  Choisir AWS S3 si :                        │
│  ✅ Écosystème AWS existant                 │
│  ✅ Besoin d'intégrations AWS (EC2, RDS)    │
│  ✅ Lifecycle complexe (6+ tiers)           │
│  ✅ Standard de facto requis (S3 API)       │
│                                             │
│  Choisir Azure Blob si :                    │
│  ✅ Infrastructure Azure (VMs, SQL)         │
│  ✅ Microsoft Enterprise Agreement          │
│  ✅ Durabilité maximale requise (16 9's)    │
│  ✅ Intégration AD/Entra ID                 │
│                                             │
│  Choisir Google Cloud Storage si :          │
│  ✅ Infrastructure GCP (GKE, Cloud SQL)     │
│  ✅ Analytics (BigQuery integration)        │
│  ✅ Coûts PUT optimisés (gratuits)          │
│  ✅ Machine learning workloads              │
│                                             │
│  Stratégie multi-cloud si :                 │
│  ✅ Éviter vendor lock-in                   │
│  ✅ Redondance géographique maximale        │
│  ✅ Optimisation coûts (best of each)       │
│  ⚠️  Complexité gestion accrue              │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Kubernetes et backups cloud-native

### VolumeSnapshots (CSI)

Kubernetes 1.20+ intègre le support natif des snapshots de volumes :

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Retain
parameters:
  encrypted: "true"
---
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mariadb-snapshot-20251213
  namespace: databases
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: mariadb-data-pvc
```

**Avantages** :
- ✅ Point-in-time crash-consistent
- ✅ Rapide (snapshot storage-level)
- ✅ Multi-cloud (CSI abstraction)

**Limitations** :
- ⚠️ Pas application-consistent (nécessite quiesce)
- ⚠️ Snapshots locaux à la région

### Velero

Outil open-source de Disaster Recovery pour Kubernetes :

```yaml
# Schedule de backup Velero
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: mariadb-daily-backup
  namespace: velero
spec:
  # Cron schedule (daily 2 AM UTC)
  schedule: "0 2 * * *"
  
  template:
    # Inclure namespace databases
    includedNamespaces:
    - databases
    
    # Hooks pour cohérence applicative
    hooks:
      resources:
      - name: mariadb-backup-hook
        includedNamespaces:
        - databases
        labelSelector:
          matchLabels:
            app: mariadb
        pre:
        - exec:
            container: mariadb
            command:
            - /bin/bash
            - -c
            - "mariadb -e 'FLUSH TABLES WITH READ LOCK; SYSTEM sleep 10'"
        post:
        - exec:
            container: mariadb
            command:
            - /bin/bash
            - -c
            - "mariadb -e 'UNLOCK TABLES'"
    
    # Snapshot volumes
    snapshotVolumes: true
    
    # Rétention
    ttl: 720h  # 30 days
    
    # Destination S3
    storageLocation: aws-s3
```

**Fonctionnalités clés** :
- Backup Kubernetes resources (YAML manifests)
- Snapshot PersistentVolumes (via CSI)
- Application-consistent hooks
- Restore cross-cluster/cross-cloud
- Schedule automatisé

---

## Stratégies multi-cloud

### Architecture multi-cloud

```
┌──────────────────────────────────────────────────────┐
│          Stratégie Multi-Cloud                       │
├──────────────────────────────────────────────────────┤
│                                                      │
│   MariaDB (Primary)                                  │
│         │                                            │
│         │ Backup (Mariabackup)                       │
│         ▼                                            │
│   ┌─────────────────┐                                │
│   │  Local staging  │                                │
│   └─────────────────┘                                │
│         │                                            │
│         ├──────────────────┬─────────────────┐       │
│         │                  │                 │       │
│         ▼                  ▼                 ▼       │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐   │
│   │  AWS S3  │      │Azure Blob│      │   GCS    │   │
│   │          │      │          │      │          │   │
│   │ Primary  │      │Secondary │      │  Backup  │   │
│   │ (30j)    │      │  (90j)   │      │  (365j)  │   │
│   └──────────┘      └──────────┘      └──────────┘   │
│         │                  │                 │       │
│         └──────────────────┴─────────────────┘       │
│                            │                         │
│                            ▼                         │
│                   Restore possible                   │
│                   depuis n'importe                   │
│                   quelle source                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Pas de vendor lock-in
- ✅ Redondance maximale (multi-provider)
- ✅ Optimisation coûts (meilleur tarif par use case)
- ✅ Compliance (data residency multi-juridictions)

**Défis** :
- ⚠️ Complexité orchestration
- ⚠️ Coûts egress (sortie de chaque cloud)
- ⚠️ Gestion identités/credentials multiple
- ⚠️ Monitoring unifié nécessaire

### Outil : rclone (multi-cloud sync)

```bash
# Configuration rclone pour multi-cloud
cat ~/.config/rclone/rclone.conf

[aws]
type = s3
provider = AWS
access_key_id = AKIA...
secret_access_key = ...
region = eu-west-1

[azure]
type = azureblob
account = mystorageaccount
key = ...

[gcs]
type = google cloud storage
project_number = 123456789
service_account_file = /path/to/key.json
location = eu

# Synchronisation multi-cloud
rclone sync /backups/mariadb/full/latest \
  aws:my-backups/mariadb/full/latest

rclone copy /backups/mariadb/full/latest \
  azure:backups/mariadb/full/latest

rclone copy /backups/mariadb/full/latest \
  gcs:backups/mariadb/full/latest
```

---

## Sécurité et conformité

### Chiffrement

**Au repos (Server-Side Encryption)** :

```bash
# AWS S3 : SSE-S3 (géré par AWS)
aws s3 cp backup.tar.gz s3://my-backups/ \
  --server-side-encryption AES256

# AWS S3 : SSE-KMS (clés gérées KMS)
aws s3 cp backup.tar.gz s3://my-backups/ \
  --server-side-encryption aws:kms \
  --ssekms-key-id arn:aws:kms:eu-west-1:123456789:key/xxx

# Client-Side Encryption (avant upload)
openssl enc -aes-256-cbc -salt -pbkdf2 \
  -in backup.tar.gz -out backup.tar.gz.enc
aws s3 cp backup.tar.gz.enc s3://my-backups/
```

### Object Lock (WORM - Write Once Read Many)

Protection contre suppression/modification (ransomware, erreur humaine) :

```bash
# AWS S3 : Activer Object Lock
aws s3api put-object-lock-configuration \
  --bucket my-backups \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "GOVERNANCE",
        "Days": 90
      }
    }
  }'

# Uploader avec retention
aws s3api put-object \
  --bucket my-backups \
  --key backups/full/latest.tar.gz \
  --body latest.tar.gz \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2026-03-15T00:00:00Z
```

**Modes** :
- **GOVERNANCE** : Admin peut override
- **COMPLIANCE** : Immuable, même root ne peut supprimer

### IAM et accès minimal

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BackupUserWriteOnly",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::my-backups/mariadb/*"
    },
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::my-backups/*"
    }
  ]
}
```

💡 **Principe** : Utilisateur backup peut écrire mais pas supprimer.

---

## Optimisation des coûts

### Lifecycle policies intelligentes

**Exemple : Politique GFS (Grandfather-Father-Son)** :

```json
{
  "Rules": [
    {
      "Id": "DailyToIntelligentTiering",
      "Filter": {
        "Prefix": "mariadb/daily/"
      },
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 1,
          "StorageClass": "INTELLIGENT_TIERING"
        }
      ],
      "Expiration": {
        "Days": 7
      }
    },
    {
      "Id": "WeeklyToGlacier",
      "Filter": {
        "Prefix": "mariadb/weekly/"
      },
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 90
      }
    },
    {
      "Id": "MonthlyToDeepArchive",
      "Filter": {
        "Prefix": "mariadb/monthly/"
      },
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

**Économies** :

```
Base 5 TB, rétention 7 ans :

Sans lifecycle (STANDARD) :
├─ 5 TB × 12 mois × 7 ans × 0,023€ = 9 660€

Avec lifecycle (GFS intelligent) :
├─ Daily (7j, INTELLIGENT_TIERING) : 5 TB × 0,015€ = 75€/mois
├─ Weekly (90j, GLACIER) : 20 TB × 0,004€ = 80€/mois
├─ Monthly (7y, DEEP_ARCHIVE) : 420 TB × 0,00099€ = 416€/mois
└─ Total : ~571€/mois × 12 × 7 = 4 798€

Économie : 50%
```

### Compression et deduplication

```bash
# Compression avant upload (gain 60-70%)
mariabackup --backup --compress --compress-threads=4 \
  --target-dir=/backups/full/latest

# Taille : 2 TB → 600 GB compressé
# Économie stockage : 0,023€ × 1400 GB × 12 = 386€/an

# Deduplication (pour sauvegardes incrémentales)
# Mariabackup incremental : seules pages modifiées
# Gain : 95% sur incrementaux
```

---

## ✅ Points clés à retenir

- **Cloud-native = API-first** : Automatisation complète, Infrastructure as Code
- **Durabilité exceptionnelle** : 11-16 9's vs 2-3 9's on-premise
- **Scalabilité illimitée** : Pas de provisionnement, croissance instantanée
- **Multi-cloud** : AWS S3 (standard), Azure Blob (durabilité), GCS (analytics)
- **Kubernetes-native** : VolumeSnapshots (CSI) + Velero pour DR complet
- **Sécurité** : Chiffrement multi-couches, Object Lock (WORM), IAM granulaire
- **Coûts optimisés** : Lifecycle policies (50% économies), compression, intelligent tiering
- **Géo-redondance** : Cross-region replication, protection datacenter
- **Compliance** : GDPR, HIPAA, SOC 2, audit trails natifs
- **Restauration** : Attention coûts egress, latence réseau, bande passante

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 AWS S3 - Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/backup-best-practices.html)
- [📖 Azure Blob Storage - Disaster Recovery](https://docs.microsoft.com/azure/storage/common/storage-disaster-recovery-guidance)
- [📖 Google Cloud Storage - Backup and DR](https://cloud.google.com/architecture/dr-scenarios-planning-guide)
- [📖 Kubernetes CSI Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [📖 Velero Documentation](https://velero.io/docs/)

### Outils

- [Rclone - Multi-cloud sync](https://rclone.org/)
- [Restic - Cloud-native backup](https://restic.net/)
- [Velero - Kubernetes backup](https://velero.io/)

---

## ➡️ Sections suivantes

Les sous-sections suivantes détailleront chaque aspect :

**[12.8.1 - S3 et Object Storage](./08.1-s3-object-storage.md)** : Configuration S3, lifecycle policies, SDK, réplication cross-region, optimisations.

**[12.8.2 - Kubernetes VolumeSnapshots](./08.2-kubernetes-volumesnapshots.md)** : CSI drivers, VolumeSnapshotClass, restore, Velero, Stash.

---


⏭️ [S3 et Object Storage](/12-sauvegarde-restauration/08.1-s3-object-storage.md)

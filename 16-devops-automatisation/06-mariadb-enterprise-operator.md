🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.6 MariaDB Enterprise Operator 🆕

> **Niveau** : Expert  
> **Durée estimée** : 5-6 heures  
> **Prérequis** : 
> - Section 16.5 mariadb-operator (Community) maîtrisée
> - Compréhension des concepts Kubernetes Operators
> - Expérience avec StatefulSets et PersistentVolumes
> - Connaissance de MaxScale, ColumnStore (chapitres 7, 14)
> - Budget pour license enterprise

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les différences entre Community et Enterprise Operator
- **Évaluer** le ROI d'une solution enterprise pour votre organisation
- **Déployer** MariaDB avec MaxScale intégré via l'operator
- **Configurer** ColumnStore pour workloads analytiques
- **Implémenter** backup point-in-time recovery (PITR)
- **Gérer** multi-cloud et hybrid deployments
- **Utiliser** le support 24/7 et les SLAs commerciaux
- **Appliquer** les best practices enterprise-grade

---

## Introduction

### Qu'est-ce que le MariaDB Enterprise Operator ?

Le **MariaDB Enterprise Operator** est l'operator Kubernetes **officiel** développé et supporté par **MariaDB Corporation** pour gérer MariaDB Server, MaxScale, et ColumnStore sur Kubernetes.

**Caractéristiques principales** :

```
┌──────────────────────────────────────────────────────────────┐
│         MariaDB Enterprise Operator (Officiel)               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🏢 Développeur: MariaDB Corporation                         │
│  💰 License: Propriétaire (Subscription requise)             │
│  📅 GA: Q1 2024                                              │
│  🆕 Dernière version: 1.2.x (Décembre 2025)                  │
│  ☸️  Kubernetes: 1.25+                                       │
│  🐳 MariaDB: 10.6, 10.11, 11.4, 11.8 Enterprise              │
│                                                              │
│  Composants gérés:                                           │
│  ✅ MariaDB Server (toutes éditions Enterprise)              │
│  ✅ MaxScale 25.01 (intégré natif)                           │
│  ✅ ColumnStore (analytique)                                 │
│  ✅ Xpand (distributed SQL)                                  │
│  ✅ Backup/Restore enterprise                                │
│  ✅ Monitoring avancé                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Positionnement : Enterprise vs Community

**Tableau comparatif approfondi** :

| Aspect | Community Operator | Enterprise Operator |
|--------|-------------------|---------------------|
| **🏢 Organisation** | | |
| Développeur | Martin Montes (mmontes) | MariaDB Corporation |
| License | Apache 2.0 (OSS) | Propriétaire |
| Code source | ✅ Public (GitHub) | ❌ Fermé |
| Coût | Gratuit | Subscription ($$$$) |
| **📞 Support** | | |
| Support | GitHub issues (communauté) | 24/7 commercial + SLA |
| SLA disponible | ❌ Non | ✅ Oui (99.95%+) |
| Hotfixes prioritaires | ❌ Non | ✅ Oui |
| Support téléphonique | ❌ Non | ✅ Oui |
| Account manager dédié | ❌ Non | ✅ Oui (Enterprise+) |
| **🗄️ MariaDB Versions** | | |
| Community Server | ✅ 10.6, 10.11, 11.4, 11.8 | ✅ Compatible |
| Enterprise Server | ❌ Non supporté | ✅ 10.6, 10.11, 11.4, 11.8 |
| Extended lifecycle | ❌ Non | ✅ 10 ans (RHEL-like) |
| Security patches | Standard | Prioritaires + backports |
| **🔄 Haute Disponibilité** | | |
| Réplication standard | ✅ Primary-Replica | ✅ Primary-Replica avancée |
| Galera Cluster | ✅ 3+ nœuds | ✅ 3+ nœuds optimisé |
| MaxScale intégré | ❌ Non | ✅ Oui (25.01+) |
| ProxySQL | ✅ Oui | ❌ Non (MaxScale preferred) |
| Failover automatique | ⚠️ Basique | ✅ Avancé avec MaxScale |
| Multi-master writes | Via Galera | Via Galera + MaxScale routing |
| **💾 Backup & Restore** | | |
| Backup planifié | ✅ S3, GCS, PVC | ✅ S3, GCS, Azure, PVC |
| Backup on-demand | ✅ Oui | ✅ Oui |
| Point-in-Time Recovery | ❌ Non | ✅ Oui (PITR) |
| Incremental backup | ⚠️ Limité | ✅ Oui |
| Backup encryption | ⚠️ Manuel | ✅ Automatique (AES-256) |
| Backup compression | ✅ gzip | ✅ gzip, zstd, lz4 |
| Restore validation | ⚠️ Manuel | ✅ Automatique |
| **📊 Analytique** | | |
| ColumnStore OLAP | ❌ Non | ✅ Oui |
| Xpand distributed SQL | ❌ Non | ✅ Oui |
| Hybrid transactional/analytical | ❌ Non | ✅ Oui (HTAP) |
| **☁️ Multi-Cloud** | | |
| AWS | ✅ EKS | ✅ EKS + RDS optimization |
| GCP | ✅ GKE | ✅ GKE + CloudSQL optimization |
| Azure | ✅ AKS | ✅ AKS + Azure Database optimization |
| On-premise | ✅ Oui | ✅ Oui + hybrid |
| Multi-cloud management | ⚠️ Manuel | ✅ Unified control plane |
| **🔐 Sécurité** | | |
| Encryption at rest | ✅ Via K8s | ✅ MariaDB native encryption |
| Encryption in transit | ✅ TLS | ✅ TLS + certificate management |
| Audit logging | ⚠️ Manuel | ✅ Automatique |
| Compliance (GDPR, SOC2) | ⚠️ À implémenter | ✅ Built-in compliance features |
| Secrets management | External Secrets Operator | ✅ Intégré + ESO support |
| **📈 Monitoring** | | |
| Prometheus metrics | ✅ mysqld_exporter | ✅ Enhanced metrics |
| Grafana dashboards | ⚠️ Community templates | ✅ Official dashboards |
| Alerting | Via Prometheus | ✅ Integrated + custom |
| Performance Insights | ❌ Non | ✅ Oui |
| Query Analytics | ⚠️ Slow query log | ✅ Query Analytics UI |
| **🚀 Scalabilité** | | |
| Horizontal scaling | ⚠️ Manuel | ✅ Semi-automatique |
| Vertical scaling | ⚠️ Manuel (downtime) | ✅ Automatique (moins downtime) |
| Auto-scaling | ❌ Non | ✅ Oui (metric-based) |
| Storage expansion | ✅ Manual PVC resize | ✅ Automatique |
| **🔧 Opérations** | | |
| Rolling updates | ✅ Oui | ✅ Oui (zero-downtime) |
| Canary deployments | ⚠️ Manuel | ✅ Intégré |
| Blue-green deployments | ⚠️ Manuel | ✅ Intégré |
| Schema migration | ⚠️ Manuel (Flyway/Liquibase) | ✅ Intégré + validation |
| **💵 Coût** | | |
| License operator | Gratuit | Inclus dans subscription |
| MariaDB Server | Gratuit | Subscription MariaDB Enterprise |
| Support | Gratuit (best effort) | Inclus (24/7 SLA) |
| Formation | Communauté | ✅ Training inclus |
| **📚 Documentation** | | |
| Qualité | Bonne (GitHub) | Excellente (professionnelle) |
| Exemples | ✅ Nombreux | ✅ Très nombreux + templates |
| Training materials | Communauté | ✅ Official training |
| Best practices guides | ⚠️ Communauté | ✅ Official guides |

### Quand choisir Enterprise Operator ?

**Critères de décision** :

```
┌──────────────────────────────────────────────────────────────┐
│            Décision Community vs Enterprise                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Choisir COMMUNITY OPERATOR si:                              │
│  ✅ Startup / PME avec budget limité                         │
│  ✅ Environnements dev/test/staging                          │
│  ✅ Production non-critique (tolérance downtime)             │
│  ✅ Équipe DevOps expérimentée                               │
│  ✅ Pas de contraintes réglementaires strictes               │
│  ✅ Galera/Réplication suffisent (pas besoin MaxScale)       │
│  ✅ Backup basique suffisant (pas PITR)                      │
│  ✅ Mono-cloud ou on-premise simple                          │
│                                                              │
│  ──────────────────────────────────────────────────────────  │
│                                                              │
│  Choisir ENTERPRISE OPERATOR si:                             │
│  ✅ Grande entreprise / Corporation                          │
│  ✅ Production critique (SLA > 99.9%)                        │
│  ✅ Budget conséquent ($$$$)                                 │
│  ✅ Besoin support 24/7 avec SLA                             │
│  ✅ Compliance stricte (GDPR, SOC2, HIPAA, PCI-DSS)          │
│  ✅ MaxScale requis (workload routing, firewall)             │
│  ✅ ColumnStore pour analytique                              │
│  ✅ PITR (point-in-time recovery) obligatoire                │
│  ✅ Multi-cloud ou hybrid cloud                              │
│  ✅ Équipe limitée (besoin automation maximale)              │
│  ✅ Risk-averse (besoin garanties commerciales)              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Calcul ROI simplifié** :

```
Coût Community Operator:
- License operator: 0€
- MariaDB Server: 0€ (community edition)
- Support: 0€ (best effort communauté)
- Expertise DevOps: 80k€/an (1 senior DevOps)
- Downtime (4h/an): 50k€ (perte business)
─────────────────────────────────
Total annuel: ~130k€

Coût Enterprise Operator:
- Subscription MariaDB Enterprise: 50k€/an (exemple)
- Support 24/7: Inclus
- Expertise DevOps: 60k€/an (moins d'expertise requise)
- Downtime (30min/an): 6k€ (réduction 87%)
- Training: Inclus
─────────────────────────────────
Total annuel: ~116k€

ROI: 130k - 116k = 14k€ économisés/an
+ Réduction risque business
+ Meilleure time-to-market
+ Compliance garantie
```

💡 **Note** : Ce calcul est simplifié. Le ROI réel dépend de votre contexte spécifique (taille équipe, criticité business, coût downtime, etc.)

---

## Architecture Enterprise Operator

### Vue d'ensemble des composants

```
┌─────────────────────────────────────────────────────────────────┐
│              MariaDB Enterprise Operator Stack                  │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Control Plane                             │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │    Enterprise Operator Controller                    │  │ │
│  │  │                                                      │  │ │
│  │  │  - Reconciliation loops                              │  │ │
│  │  │  - Admission webhooks                                │  │ │
│  │  │  - License validation                                │  │ │
│  │  │  - Telemetry & analytics                             │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Custom Resources                         │ │
│  │                                                            │ │
│  │  - MariaDBServer (MariaDB instances)                       │ │
│  │  - MaxScale (proxy/router)                                 │ │
│  │  - ColumnStore (analytics)                                 │ │
│  │  - Backup/Restore (enterprise)                             │ │
│  │  - User/Database/Grant (declarative)                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 Managed Resources                          │ │
│  │                                                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │   MariaDB    │  │   MaxScale   │  │ ColumnStore  │      │ │
│  │  │ StatefulSet  │  │  Deployment  │  │ StatefulSet  │      │ │
│  │  │              │  │              │  │              │      │ │
│  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │ │
│  │  │  │Primary │  │  │  │MaxScale│  │  │  │CS Node │  │      │ │
│  │  │  │        │◄─┼──┼──┤Routing │  │  │  │        │  │      │ │
│  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │ │
│  │  │  ┌────────┐  │  │      │       │  │              │      │ │
│  │  │  │Replica │  │  │      │       │  │              │      │ │
│  │  │  │        │◄─┼──┼──────┘       │  │              │      │ │
│  │  │  └────────┘  │  │              │  │              │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Support & Monitoring Layer                    │ │
│  │                                                            │ │
│  │  - Enhanced Prometheus metrics                             │ │
│  │  - Grafana dashboards (official)                           │ │
│  │  - Query Analytics                                         │ │
│  │  - Performance Insights                                    │ │
│  │  - Telemetry to MariaDB Corp (opt-in)                      │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Composants clés

#### 1. MariaDB Server Enterprise

**Différences vs Community Edition** :

| Feature | Community | Enterprise |
|---------|-----------|------------|
| Core database | ✅ Même code | ✅ Même code |
| Plugins additionnels | Standard | ✅ Enterprise plugins |
| Thread pool | ✅ Oui | ✅ Optimisé |
| Audit plugin | ✅ Basique | ✅ Advanced (MariaDB Audit) |
| Encryption plugins | ✅ Standard | ✅ AWS KMS, HashiCorp Vault |
| PAM authentication | ✅ Oui | ✅ Oui + enhanced |
| Support lifecycle | 5 ans | ✅ 10 ans |
| Security patches | Standard | ✅ Prioritaires + backports |
| Bug fixes | Standard | ✅ Hotfixes rapides |

#### 2. MaxScale 25.01 🆕

**Intégration native dans l'operator** :

```yaml
apiVersion: mariadb.com/v1
kind: MaxScale
metadata:
  name: maxscale
spec:
  servers:
  - mariadb-0.mariadb.default.svc.cluster.local:3306
  - mariadb-1.mariadb.default.svc.cluster.local:3306
  - mariadb-2.mariadb.default.svc.cluster.local:3306
  
  # Nouveautés MaxScale 25.01
  services:
  - name: Read-Write-Service
    router: readwritesplit
    user: maxscale
    password:
      secretKeyRef:
        name: maxscale-secret
        key: password
    
    # 🆕 Workload Capture
    workloadCapture:
      enabled: true
      file: /var/lib/maxscale/capture.sql
    
    # 🆕 Workload Replay
    workloadReplay:
      enabled: false
      file: /var/lib/maxscale/capture.sql
  
  # 🆕 Diff Router (compare two databases)
  - name: Diff-Service
    router: diffreporter
    user: maxscale
    primaryServer: mariadb-0.mariadb
    secondaryServer: mariadb-1.mariadb
```

**Fonctionnalités MaxScale gérées par l'operator** :

- ✅ **Read-Write Split** : Route automatique R/W vers primary, R/O vers replicas
- ✅ **Database Firewall** : Filtrage requêtes SQL (SQL injection protection)
- ✅ **Query Routing** : Route par regex, utilisateur, database
- ✅ **Connection Pooling** : Multiplexing des connexions
- ✅ **Automatic Failover** : Bascule automatique si primary down
- ✅ **Health Checks** : Monitor automatique des backends
- 🆕 **Workload Capture/Replay** : Record et replay du trafic SQL
- 🆕 **Diff Router** : Compare deux bases (utile pour migration)

#### 3. ColumnStore

**Analytique OLAP sur Kubernetes** :

```yaml
apiVersion: mariadb.com/v1
kind: ColumnStore
metadata:
  name: analytics-cluster
spec:
  # ColumnStore version
  version: "11.8"
  
  # Topology
  primaryNode:
    replicas: 1
    storage:
      size: 500Gi
      storageClassName: fast-ssd
  
  performanceModule:
    replicas: 3
    storage:
      size: 1Ti
      storageClassName: fast-ssd
  
  # Configuration
  config: |
    [ColumnStore]
    PMInstanceCount = 3
    PMMemory = 32G
    
  # Integration with MariaDB Server (HTAP)
  mariadbRef:
    name: mariadb-oltp
```

**Use cases ColumnStore** :
- 📊 Data warehousing
- 📈 Business Intelligence (BI)
- 🔍 Log analytics
- 📉 Time-series analytics
- 🎯 HTAP (Hybrid Transactional/Analytical Processing)

---

## Installation et configuration

### Prérequis

**Licenses requises** :

```
1. MariaDB Enterprise Subscription
   └─> Contacter MariaDB Sales: https://mariadb.com/pricing/
   
2. Kubernetes cluster
   ├─> Version 1.25+
   ├─> RBAC enabled
   ├─> StorageClass avec dynamic provisioning
   └─> LoadBalancer support (optionnel)

3. Resources minimales
   ├─> 3+ worker nodes
   ├─> 16GB RAM par node (minimum)
   └─> SSD storage recommandé
```

### Installation via Helm

**1. Ajouter repository Helm** :

```bash
# Ajouter repo MariaDB Enterprise (nécessite credentials)
helm repo add mariadb-enterprise https://charts.mariadb.com/enterprise \
  --username <customer-id> \
  --password <license-token>

helm repo update
```

**2. Créer namespace** :

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mariadb-system
  labels:
    name: mariadb-system
    purpose: enterprise-operator
```

**3. Créer secret pour license** :

```bash
kubectl create secret generic mariadb-license \
  --namespace mariadb-system \
  --from-literal=customer-id='<your-customer-id>' \
  --from-literal=license-token='<your-license-token>'
```

**4. Installer l'operator** :

```bash
helm install mariadb-enterprise-operator mariadb-enterprise/mariadb-enterprise-operator \
  --namespace mariadb-system \
  --set license.secretName=mariadb-license \
  --set license.secretKeyCustomerId=customer-id \
  --set license.secretKeyLicenseToken=license-token \
  --set enableWebhooks=true \
  --set enableServiceMonitor=true
```

**5. Vérifier installation** :

```bash
# Vérifier pods
kubectl get pods -n mariadb-system

# Output:
# NAME                                        READY   STATUS    RESTARTS   AGE
# mariadb-enterprise-operator-xxxxx-xxxxx    1/1     Running   0          2m

# Vérifier CRDs
kubectl get crd | grep mariadb

# Output:
# mariadbservers.mariadb.com
# maxscales.mariadb.com
# columnstores.mariadb.com
# backups.mariadb.com
# restores.mariadb.com
```

---

## Custom Resources (CRDs) Enterprise

### 1. MariaDBServer

**Exemple production-ready** :

```yaml
apiVersion: mariadb.com/v1
kind: MariaDBServer
metadata:
  name: mariadb-prod
  namespace: production
spec:
  # Image enterprise
  image: "registry.mariadb.com/enterprise-server:11.8"
  imagePullSecrets:
  - name: mariadb-registry-credentials
  
  # Réplication Primary-Replica
  replication:
    enabled: true
    primary:
      replicas: 1
    replica:
      replicas: 2
  
  # Stockage
  storage:
    size: 1Ti
    storageClassName: premium-ssd
    # Politique de rétention
    persistentVolumeClaimRetentionPolicy:
      whenDeleted: Retain
      whenScaled: Retain
  
  # Resources
  resources:
    primary:
      requests:
        cpu: "4"
        memory: "16Gi"
      limits:
        cpu: "8"
        memory: "32Gi"
    replica:
      requests:
        cpu: "2"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "16Gi"
  
  # Configuration MariaDB
  config:
    myCnf: |
      [mysqld]
      # Charset (🆕 MariaDB 11.8)
      character-set-server = utf8mb4
      collation-server = utf8mb4_unicode_ci
      
      # InnoDB
      innodb_buffer_pool_size = 24G
      innodb_log_file_size = 2G
      innodb_flush_log_at_trx_commit = 1
      
      # 🆕 MariaDB 11.8 optimizations
      innodb_alter_copy_bulk = ON
      innodb_io_capacity = 4000
      
      # Replication
      log_bin = mysql-bin
      binlog_format = ROW
      gtid_strict_mode = ON
      
      # Security
      require_secure_transport = ON
  
  # Secrets
  auth:
    rootPasswordSecret:
      name: mariadb-root-password
      key: password
    replicationPasswordSecret:
      name: mariadb-replication-password
      key: password
  
  # Backup intégré
  backup:
    enabled: true
    schedule: "0 2 * * *"
    storage:
      s3:
        bucket: mariadb-backups-prod
        region: eu-west-1
        credentialsSecret:
          name: s3-credentials
          accessKeyIdKey: access-key-id
          secretAccessKeyKey: secret-access-key
    
    # 🆕 Enterprise features
    encryption:
      enabled: true
      algorithm: AES-256
    compression:
      enabled: true
      algorithm: zstd
      level: 3
    retention:
      full: 30d
      incremental: 7d
    
    # 🆕 Point-in-Time Recovery
    pitr:
      enabled: true
      binlogRetention: 7d
  
  # Monitoring
  monitoring:
    enabled: true
    serviceMonitor:
      enabled: true
      interval: 30s
    
    # 🆕 Query Analytics
    queryAnalytics:
      enabled: true
      slowQueryThreshold: 1s
  
  # Anti-affinity
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: mariadb-prod
        topologyKey: kubernetes.io/hostname
```

### 2. MaxScale (intégré)

**Configuration complète** :

```yaml
apiVersion: mariadb.com/v1
kind: MaxScale
metadata:
  name: maxscale-prod
  namespace: production
spec:
  # Version MaxScale
  version: "25.01"
  
  # Replicas (HA)
  replicas: 3
  
  # MariaDB servers backend
  mariadbServerRef:
    name: mariadb-prod
  
  # Credentials
  auth:
    adminPasswordSecret:
      name: maxscale-admin-password
      key: password
    clientPasswordSecret:
      name: maxscale-client-password
      key: password
  
  # Services configuration
  services:
  # 1. Read-Write Split
  - name: rw-service
    router: readwritesplit
    port: 3306
    
    # Routing rules
    routingRules:
    - name: regex-route
      type: regex
      match: "^SELECT.*FOR UPDATE"
      server: primary
    
    # Connection pooling
    connectionPooling:
      enabled: true
      maxPoolSize: 100
      minPoolSize: 10
    
    # 🆕 Workload Capture
    workloadCapture:
      enabled: true
      file: /var/lib/maxscale/workload-capture.sql
  
  # 2. Read-Only (Replicas)
  - name: read-service
    router: readconnroute
    port: 3307
    servers:
      - type: replica
  
  # 3. Database Firewall
  - name: firewall-service
    router: readwritesplit
    port: 3308
    
    firewall:
      enabled: true
      rules:
      - name: block-drop-table
        type: deny
        query: "DROP TABLE.*"
      - name: block-sql-injection
        type: deny
        query: ".*OR 1=1.*"
      - name: allow-select
        type: allow
        query: "SELECT.*"
  
  # 4. 🆕 Diff Router (migration testing)
  - name: diff-service
    router: diffreporter
    port: 3309
    primaryServer:
      ref: mariadb-prod-0
    secondaryServer:
      ref: mariadb-staging-0
    reportFile: /var/lib/maxscale/diff-report.txt
  
  # Resources
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  
  # Storage for captures/logs
  storage:
    size: 100Gi
    storageClassName: standard
  
  # Monitoring
  monitoring:
    enabled: true
    serviceMonitor:
      enabled: true
      interval: 15s
```

### 3. Backup Enterprise

**Point-in-Time Recovery** :

```yaml
apiVersion: mariadb.com/v1
kind: Backup
metadata:
  name: mariadb-pitr-backup
  namespace: production
spec:
  mariadbServerRef:
    name: mariadb-prod
  
  # Type de backup
  type: full  # ou incremental
  
  # Schedule
  schedule:
    cron: "0 */6 * * *"  # Toutes les 6h
    suspend: false
  
  # 🆕 Enterprise backup features
  engine: mariabackup
  
  # Compression
  compression:
    enabled: true
    algorithm: zstd  # Plus rapide que gzip
    level: 3  # Balance compression/speed
  
  # Encryption
  encryption:
    enabled: true
    algorithm: AES-256-CBC
    keySecret:
      name: backup-encryption-key
      key: encryption-key
  
  # Storage
  storage:
    s3:
      bucket: mariadb-backups-prod
      region: eu-west-1
      prefix: pitr/
      endpoint: s3.amazonaws.com
      
      # Credentials
      credentialsSecret:
        name: s3-backup-credentials
      
      # 🆕 Server-side encryption
      serverSideEncryption:
        enabled: true
        kmsKeyId: arn:aws:kms:eu-west-1:123456789:key/abc-def
  
  # Retention
  retention:
    full:
      count: 4  # Garder 4 full backups
      days: 30  # ou 30 jours
    incremental:
      count: 28  # 28 incremental backups
      days: 7
    binlog:
      days: 7  # Pour PITR
  
  # Validation
  validation:
    enabled: true
    checksum: true
  
  # Notifications
  notifications:
    onSuccess:
      webhookUrl: https://hooks.slack.com/...
    onFailure:
      webhookUrl: https://hooks.slack.com/...
      emailTo: dba-team@company.com
```

### 4. Restore avec PITR

**Restore point-in-time** :

```yaml
apiVersion: mariadb.com/v1
kind: Restore
metadata:
  name: pitr-restore-disaster
  namespace: production
spec:
  mariadbServerRef:
    name: mariadb-prod
  
  # Backup source
  backupRef:
    name: mariadb-pitr-backup-20251214-020000
  
  # 🆕 Point-in-Time Recovery
  pitr:
    enabled: true
    targetTime: "2025-12-14T09:30:00Z"  # Restaurer à ce moment exact
    
    # Ou utiliser position binlog
    # targetPosition:
    #   binlogFile: "mysql-bin.000123"
    #   binlogPosition: 456789
  
  # Restore to new instance (non-destructive)
  targetMariaDB:
    name: mariadb-restored
    namespace: recovery
  
  # Validation
  validation:
    enabled: true
    checksum: true
    
  # Notification
  notifications:
    onComplete:
      webhookUrl: https://hooks.slack.com/...
```

---

## Monitoring et observabilité enterprise

### Query Analytics 🆕

**Activation** :

```yaml
spec:
  monitoring:
    queryAnalytics:
      enabled: true
      
      # Seuils
      slowQueryThreshold: 1s
      longQueryTime: 2s
      
      # Sampling (pour hautes charges)
      samplingRate: 0.1  # 10% des requêtes
      
      # Retention
      retention: 30d
      
      # Export vers service externe
      export:
        enabled: true
        endpoint: https://analytics.company.com/api/queries
        apiKeySecret:
          name: analytics-api-key
          key: api-key
```

**Dashboards Grafana officiels** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-grafana-dashboards
  namespace: monitoring
data:
  mariadb-overview.json: |
    {
      "dashboard": {
        "title": "MariaDB Enterprise - Overview",
        "panels": [
          {
            "title": "QPS (Queries per Second)",
            "targets": [
              "rate(mysql_global_status_queries[5m])"
            ]
          },
          {
            "title": "Connections",
            "targets": [
              "mysql_global_status_threads_connected"
            ]
          },
          {
            "title": "InnoDB Buffer Pool",
            "targets": [
              "mysql_global_status_innodb_buffer_pool_pages_data",
              "mysql_global_status_innodb_buffer_pool_pages_total"
            ]
          },
          {
            "title": "🆕 Query Analytics - Top 10 Slow",
            "targets": [
              "topk(10, mysql_query_analytics_slow_queries)"
            ]
          }
        ]
      }
    }
```

### Performance Insights

**Metrics automatiques collectés** :

```yaml
# Enhanced metrics (au-delà de mysqld_exporter)
mariadb_performance_schema_queries_total
mariadb_performance_schema_table_io_waits_total
mariadb_performance_schema_table_lock_waits_total

# 🆕 Enterprise-specific
mariadb_enterprise_thread_pool_active_threads
mariadb_enterprise_columnstore_pm_memory_usage
mariadb_maxscale_connections_total
mariadb_maxscale_query_latency_seconds
```

---

## Cas d'usage enterprise

### 1. Multi-cloud déploiement

**Scenario** : Bases MariaDB sur AWS, GCP et Azure avec management centralisé

```yaml
# AWS cluster
apiVersion: mariadb.com/v1
kind: MariaDBServer
metadata:
  name: mariadb-aws
  namespace: production
  labels:
    cloud: aws
    region: eu-west-1
spec:
  image: registry.mariadb.com/enterprise-server:11.8
  storage:
    storageClassName: gp3  # AWS EBS gp3
  # ...

---
# GCP cluster
apiVersion: mariadb.com/v1
kind: MariaDBServer
metadata:
  name: mariadb-gcp
  namespace: production
  labels:
    cloud: gcp
    region: europe-west1
spec:
  image: registry.mariadb.com/enterprise-server:11.8
  storage:
    storageClassName: pd-ssd  # GCP Persistent Disk SSD
  # ...

---
# Azure cluster
apiVersion: mariadb.com/v1
kind: MariaDBServer
metadata:
  name: mariadb-azure
  namespace: production
  labels:
    cloud: azure
    region: westeurope
spec:
  image: registry.mariadb.com/enterprise-server:11.8
  storage:
    storageClassName: managed-premium  # Azure Premium SSD
  # ...
```

**Management centralisé** : Un seul operator peut gérer les 3 clusters via federation.

### 2. Compliance et audit

**Configuration GDPR/SOC2** :

```yaml
apiVersion: mariadb.com/v1
kind: MariaDBServer
metadata:
  name: mariadb-compliant
spec:
  # Encryption at rest (obligatoire GDPR)
  encryption:
    atRest:
      enabled: true
      plugin: file_key_management
      keyFile:
        secretKeyRef:
          name: encryption-key-file
          key: keyfile
  
  # Encryption in transit (TLS obligatoire)
  tls:
    enabled: true
    certificateSecret:
      name: mariadb-tls-cert
    requireSecureTransport: true
  
  # Audit logging (SOC2)
  audit:
    enabled: true
    plugin: server_audit
    config: |
      server_audit_events=CONNECT,QUERY_DDL,QUERY_DML
      server_audit_logging=ON
      server_audit_file_path=/var/log/mysql/audit.log
      server_audit_file_rotate_size=1000000
      server_audit_file_rotations=9
    
    # Export logs vers SIEM
    export:
      enabled: true
      endpoint: syslog://siem.company.com:514
  
  # Backup encryption (GDPR)
  backup:
    encryption:
      enabled: true
      algorithm: AES-256
    
    # Retention conforme
    retention:
      full: 90d  # 3 mois minimum
```

### 3. HTAP (Hybrid Transactional/Analytical)

**MariaDB Server + ColumnStore** :

```yaml
# OLTP (MariaDB Server)
apiVersion: mariadb.com/v1
kind: MariaDBServer
metadata:
  name: mariadb-oltp
spec:
  replicas: 3
  storage:
    size: 1Ti
    storageClassName: premium-ssd
  
  # Tables transactionnelles (InnoDB)
  initSQL: |
    CREATE DATABASE ecommerce;
    USE ecommerce;
    
    CREATE TABLE orders (
      id BIGINT AUTO_INCREMENT PRIMARY KEY,
      customer_id BIGINT NOT NULL,
      order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      total_amount DECIMAL(10,2)
    ) ENGINE=InnoDB;

---
# OLAP (ColumnStore)
apiVersion: mariadb.com/v1
kind: ColumnStore
metadata:
  name: columnstore-olap
spec:
  version: "11.8"
  
  primaryNode:
    replicas: 1
  
  performanceModule:
    replicas: 4
    storage:
      size: 5Ti
      storageClassName: premium-ssd
  
  # Integration avec MariaDB OLTP
  mariadbRef:
    name: mariadb-oltp
  
  # Tables analytiques (ColumnStore)
  initSQL: |
    USE ecommerce;
    
    CREATE TABLE orders_analytics (
      id BIGINT,
      customer_id BIGINT,
      order_date DATE,
      total_amount DECIMAL(10,2),
      INDEX(order_date)
    ) ENGINE=ColumnStore;
```

**Synchronisation OLTP → OLAP** :

```sql
-- CDC (Change Data Capture) avec trigger ou ETL
CREATE TRIGGER orders_to_analytics
AFTER INSERT ON orders
FOR EACH ROW
  INSERT INTO orders_analytics 
  SELECT * FROM orders WHERE id = NEW.id;
```

---

## Support et SLA

### Niveaux de support

**Support tiers** :

| Tier | Response Time | Channels | Best for |
|------|---------------|----------|----------|
| **Standard** | 4 business hours | Email, Web | Dev/Test |
| **Premium** | 1 hour (Sev 1) | Email, Web, Phone | Production |
| **Enterprise** | 30 min (Sev 1) | Email, Web, Phone, Slack | Mission-critical |

**Severity levels** :

```
Severity 1 (Critical):
- Production down
- Data loss/corruption
- Security breach
→ Response: 30min - 1h selon tier

Severity 2 (High):
- Degraded performance
- Feature not working
- Workaround available
→ Response: 4h - 8h selon tier

Severity 3 (Medium):
- Minor issues
- Questions
- Feature requests
→ Response: 1-2 business days

Severity 4 (Low):
- Documentation
- Best practices
- Training
→ Response: 3-5 business days
```

### Engagements SLA

**Uptime SLA** (avec Enterprise tier) :

```
99.95% uptime guarantee
= Maximum 4.38 heures downtime/an

Si non respecté:
- < 99.95%: 10% credit
- < 99.9%: 25% credit
- < 99.5%: 50% credit
- < 99%: 100% credit
```

**Support 24/7/365** :
- ✅ Équipes DBA expertes MariaDB
- ✅ Hotline téléphonique
- ✅ Slack channel dédié (Enterprise)
- ✅ Account manager assigné
- ✅ Quarterly Business Reviews (QBR)

---

## ✅ Points clés à retenir

- **Enterprise = Production critique** : SLA, support 24/7, garanties commerciales
- **MaxScale intégré** : Read-write split, failover, firewall natifs
- **PITR** : Point-in-Time Recovery avec binlog retention
- **ColumnStore** : Analytique OLAP sur même plateforme
- **Multi-cloud** : Unified management AWS, GCP, Azure
- **Compliance built-in** : Audit, encryption, GDPR-ready
- **Query Analytics** : Performance insights avancés
- **Support commercial** : 24/7 avec SLA 99.95%
- **Coût vs ROI** : Évaluer selon criticité, downtime cost, équipe
- **License requise** : Subscription MariaDB Enterprise obligatoire

💡 **Décision finale** : Enterprise si budget permet ET (production critique OU compliance stricte OU besoin MaxScale/ColumnStore)

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 MariaDB Enterprise Operator Docs](https://mariadb.com/docs/server/enterprise-operator/)
- [📖 MaxScale 25.01 Documentation](https://mariadb.com/kb/en/maxscale/)
- [📖 ColumnStore Documentation](https://mariadb.com/kb/en/mariadb-columnstore/)

### Pricing et licensing
- [💰 MariaDB Pricing](https://mariadb.com/pricing/)
- [📞 Contact Sales](https://mariadb.com/contact/)

### Support
- [🎫 Support Portal](https://support.mariadb.com/)
- [📚 Knowledge Base](https://mariadb.com/kb/)

### Training
- [🎓 MariaDB University](https://mariadb.com/university/)
- [🏆 MariaDB Certification](https://mariadb.com/certification/)

---

## ➡️ Prochaines sections

Dans les sections suivantes du chapitre DevOps, nous explorerons :

- **16.7 CI/CD pour bases de données** : Pipelines automatisés pour MariaDB
- **16.8 Gestion des migrations** : Flyway, Liquibase, gh-ost, pt-online-schema-change
- **16.9 Monitoring Prometheus/Grafana** : Métriques, dashboards, alerting
- **16.12 GitOps** : ArgoCD, FluxCD pour MariaDB

---

**MariaDB** : Version 11.8 LTS
**MariaDB Enterprise Operator** : v1.2.x

⏭️ [CI/CD pour bases de données](/16-devops-automatisation/07-cicd-bases-donnees.md)

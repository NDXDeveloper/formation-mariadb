🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 8 : DevOps, Cloud et Automatisation

> **Niveau** : Avancé à Expert — DevOps/SRE, Cloud Architects, DBA modernes  
> **Durée estimée** : 3-4 jours  
> **Prérequis** : Expertise DevOps (CI/CD, IaC), maîtrise conteneurs (Docker), orchestration (Kubernetes), administration MariaDB, expérience cloud (AWS/GCP/Azure)

---

## 🎯 L'ère du Database DevOps et du Cloud-Native

Le monde des bases de données connaît une **transformation radicale**. Pendant des années, les bases de données étaient gérées manuellement par des DBA qui configuraient des serveurs physiques, appliquaient des patches en heures creuses, et effectuaient des migrations avec des fenêtres de maintenance planifiées. Cette époque est révolue. Aujourd'hui, les bases de données doivent être **cloud-native, automatisées, et gérées comme du code**.

La convergence entre DevOps et databases — parfois appelée **"Database DevOps"** ou **"DBOps"** — apporte les mêmes principes qui ont révolutionné le développement logiciel : Infrastructure as Code, déploiements automatisés, rollback instantanés, monitoring en temps réel, et gestion déclarative via GitOps. Cette évolution n'est pas optionnelle : les entreprises qui déploient 100 fois par jour ne peuvent pas attendre une fenêtre de maintenance mensuelle pour une migration de schéma.

Les architectures cloud-native transforment également la manière dont nous pensons les bases de données. Au lieu de **pets** (serveurs soignés individuellement, nommés, irremplaçables), nous gérons du **cattle** (instances éphémères, interchangeables, auto-réparables). Kubernetes, avec ses operators et Custom Resource Definitions (CRDs), permet de gérer MariaDB comme n'importe quelle autre ressource : déclarative, versionnée, et réconciliée automatiquement.

L'objectif de cette partie est de vous donner les **compétences pour opérer MariaDB dans des environnements modernes** : déploiements conteneurisés, orchestration Kubernetes, automatisation complète via IaC (Ansible, Terraform), pipelines CI/CD pour migrations de schéma, monitoring Prometheus/Grafana, et gestion GitOps. Vous apprendrez à traiter les bases de données non plus comme des **snowflakes** (configurations uniques et fragiles) mais comme des **ressources reproductibles et automatisables**.

Ces compétences sont **essentielles pour tout DevOps ou SRE** en 2025 et au-delà. Le marché demande des professionnels capables de combiner expertise bases de données ET pratiques cloud-native modernes.

---

## 📚 Module unique : DevOps et Automatisation

### Module 16 : DevOps et Automatisation
**12 sections | Durée : 3-4 jours**

Ce module couvre l'intégralité du spectre DevOps appliqué à MariaDB, des fondamentaux aux architectures cloud les plus avancées :

#### 🏗️ Infrastructure as Code (IaC)
- **Principes fondamentaux** : Déclaratif vs impératif, idempotence, versioning
- **Benefits** : Reproductibilité, rollback, peer review, audit trail
- **Tools ecosystem** : Ansible, Terraform, Pulumi, CloudFormation
- **Best practices** : Modules réutilisables, environments séparés, secrets management

#### 🔧 Déploiement avec Ansible
- **Playbooks MariaDB** : Installation, configuration, déploiement 🔄
  - Roles structurés et réutilisables
  - Gestion des secrets avec Ansible Vault
  - Handlers pour restart services
- **Patterns avancés** : Rolling updates, blue-green deployments
- **Testing** : Molecule pour validation des playbooks

#### ☁️ Déploiement avec Terraform
- **Providers cloud** : AWS RDS (MariaDB), SkySQL (le cloud MariaDB) ; paysage managé contrasté pour MariaDB (GCP Cloud SQL **ne propose pas** MariaDB, Azure Database for MariaDB **retirée** en 2025) → l'auto-gestion reste souvent nécessaire 🔄
- **Modules MariaDB** : VPC, Security Groups, instances, backups
- **State management** : Remote backends (S3, GCS), state locking
- **Workspaces** : Gestion multi-environnements (dev, staging, prod)

#### 🐳 Conteneurisation avec Docker
- **Images officielles MariaDB** : Configuration et customization 🔄
- **Docker Compose pour développement** : Stack complète (MariaDB + app + cache)
- **Volumes et persistance** : Bind mounts vs named volumes vs tmpfs
- **Networking** : Bridge, host, overlay networks
- **Best practices** : Multi-stage builds, image optimization, security scanning

#### ☸️ Orchestration avec Kubernetes
- **StatefulSets pour MariaDB** : Identités stables, stockage persistant 🔄
  - Headless services pour découverte
  - Ordered deployment et scaling
  - Pod disruption budgets
- **PersistentVolumes** : StorageClasses, dynamic provisioning
  - CSI drivers (EBS, GCE PD, Azure Disk)
  - Snapshot et backup strategies
- **ConfigMaps et Secrets** : Configuration externalisée
- **Operators pattern** : Automation via custom controllers

#### 🎛️ mariadb-operator : Gestion déclarative 🔄
- **Installation et CRDs** : Custom Resource Definitions
  - MariaDB, Database, User, Grant resources
  - Connection, Backup, Restore resources
- **Déploiement Galera** : Cluster synchrone automatisé
  - Bootstrap et scaling automatiques
  - State transfer management
  - Split-brain prevention
- **Réplication** : Primary-replica automatique
  - GTID configuration
  - Failover handling
- **Backups automatisés** : Scheduled backups via CronJob
  - S3/GCS/Azure Blob integration
  - Point-in-time recovery

#### 🆕 MariaDB Enterprise Operator
- **Features avancées** : Enterprise-grade automation
- **Multi-tenancy** : Isolation et resource quotas
- **Advanced monitoring** : Intégration Prometheus native
- **Day 2 operations** : Upgrades, scaling, maintenance

#### 🔄 CI/CD pour bases de données
- **Pipeline challenges** : Stateful nature, data migrations, rollback complexes
- **Testing strategies** : Unit tests (queries), integration tests (schema)
- **Approval gates** : Manual validation pour production
- **Deployment strategies** : Blue-green, canary, progressive rollouts

#### 📊 Gestion des migrations de schéma
- **Flyway** : Version-based migrations 🔄
  - Versioned SQL scripts
  - Repeatable migrations
  - Callbacks et hooks
- **Liquibase** : Change-based migrations 🔄
  - XML/YAML/JSON/SQL changesets
  - Rollback support
  - Database-agnostic
- **gh-ost et pt-online-schema-change** : Zero-downtime migrations 🔄
  - Table fantôme + copie en ligne (triggers pour pt-osc, lecture du binlog pour gh-ost)
  - Throttling et pause/resume
  - Validation et rollback

#### 📈 Monitoring avec Prometheus/Grafana
- **mysqld_exporter** : Métriques MariaDB 🔄
  - Queries per second, connections, buffer pool
  - Replication lag, slow queries
  - InnoDB metrics (buffer pool, redo log)
- **Dashboards Grafana** : Visualisation temps réel 🔄
  - Templates communautaires
  - Alerting rules
  - Query insights

#### 🔭 Observabilité : Logs, Metrics, Traces
- **Logs centralisés** : ELK/EFK stack, Loki
  - Structured logging (JSON)
  - Log aggregation et correlation
- **Distributed tracing** : Jaeger, Zipkin
  - Query tracing application→database
- **Service mesh** : Istio, Linkerd pour database traffic
- **Golden signals** : Latency, Traffic, Errors, Saturation

#### 🚨 Alerting et incident response
- **Alert rules** : Prometheus AlertManager
  - Replication lag > threshold
  - Connections near max
  - Disk space < 20%
- **On-call workflows** : PagerDuty, Opsgenie
- **Runbooks automatisés** : Self-healing patterns
- **Post-mortems** : Blameless culture, learning from incidents

#### 🔀 GitOps pour les bases de données
- **Principes GitOps** : Git as single source of truth 🔄
  - Déclaratif, versionné, automatique
  - Reconciliation loops
  - Drift detection
- **Flux/ArgoCD** : Continuous deployment
  - Automated sync from Git
  - Health checks et rollback
- **Database migrations as code** : Schema versions in Git
- **Challenges** : State management, data vs code

💡 **Impact business** : L'automatisation DevOps réduit le temps de déploiement de jours/heures à quelques minutes, diminue les erreurs humaines de 80-90%, et permet de déployer 100x plus fréquemment tout en maintenant la stabilité.

---

## 🛠️ Outils modernes pour Database DevOps

### mariadb-operator : automation Kubernetes-native 🆕

Le **mariadb-operator** (open source, groupe d'API `k8s.mariadb.com`) gère MariaDB dans Kubernetes de façon entièrement déclarative via des CRDs. Son pendant commercial, le **MariaDB Enterprise Operator** (groupe d'API distinct `enterprise.mariadb.com`), en reprend le modèle et les ressources en y ajoutant support contractuel et fonctionnalités d'entreprise (détaillé en §16.6). Les exemples ci-dessous emploient l'opérateur communautaire.

#### Architecture et fonctionnement

```yaml
# Custom Resource Definition : MariaDB cluster
apiVersion: k8s.mariadb.com/v1alpha1  
kind: MariaDB  
metadata:  
  name: mariadb-galera
spec:
  # Image (la version vient de l'étiquette d'image) et replicas
  image: mariadb:12.3
  replicas: 3
  
  # Galera cluster configuration
  galera:
    enabled: true
    primary:
      podIndex: 0
    sst: mariabackup
    replicaThreads: 1
    
  # Storage
  storage:
    size: 100Gi
    storageClassName: fast-ssd
    
  # High availability
  podDisruptionBudget:
    maxUnavailable: 1
    
  # Monitoring (l'operator crée un exportateur sidecar + ServiceMonitor)
  metrics:
    enabled: true

  # Sauvegardes automatisées : déclarées via une ressource Backup distincte
  # (CronJob planifié vers S3/Azure Blob/PVC — voir l'exemple plus bas)
```

#### Fonctionnalités avancées

**1. Day-2 Operations automatisées**
```yaml
# Rolling upgrade sans downtime
kubectl patch mariadb mariadb-galera \
  --type merge \
  --patch '{"spec":{"image":"mariadb:12.3.2"}}'

# L'operator gère automatiquement :
# 1. Upgrade un nœud à la fois
# 2. Validation health check
# 3. Rollback si échec
```

**2. Backup et Restore déclaratifs**
```yaml
# Backup on-demand
apiVersion: k8s.mariadb.com/v1alpha1  
kind: Backup  
metadata:  
  name: mariadb-backup-20251215
spec:
  mariadbRef:
    name: mariadb-galera
  s3:
    bucket: mariadb-backups
    prefix: prod/

# Restore depuis backup
apiVersion: k8s.mariadb.com/v1alpha1  
kind: Restore  
metadata:  
  name: restore-from-20251215
spec:
  mariadbRef:
    name: mariadb-galera
  backupRef:
    name: mariadb-backup-20251215
```

**3. User et Database management as code**
```yaml
# Création utilisateur déclarative
apiVersion: k8s.mariadb.com/v1alpha1  
kind: User  
metadata:  
  name: app-user
spec:
  mariadbRef:
    name: mariadb-galera
  passwordSecretKeyRef:
    name: app-credentials
    key: password
  maxUserConnections: 100

# Création database
apiVersion: k8s.mariadb.com/v1alpha1  
kind: Database  
metadata:  
  name: production-db
spec:
  mariadbRef:
    name: mariadb-galera
  characterSet: utf8mb4
  collate: utf8mb4_unicode_ci

# Grant privileges
apiVersion: k8s.mariadb.com/v1alpha1  
kind: Grant  
metadata:  
  name: app-user-grants
spec:
  mariadbRef:
    name: mariadb-galera
  privileges:
    - "ALL"
  database: "production-db"
  table: "*"
  username: app-user
```

**Bénéfices** :
- ✅ Gestion 100% déclarative (GitOps-friendly)
- ✅ Reconciliation automatique (drift correction)
- ✅ Operations complexes automatisées (backup, upgrade, scaling)
- ✅ Self-healing (redémarrage automatique pods unhealthy)

---

### Prometheus/Grafana : Observabilité moderne

#### Stack complet de monitoring

**1. mysqld_exporter : Métriques MariaDB**
```yaml
# Déploiement exporter en sidecar
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: mariadb-with-exporter
spec:
  template:
    spec:
      containers:
      # Container MariaDB principal
      - name: mariadb
        image: mariadb:12.3
        ports:
        - containerPort: 3306
          
      # Sidecar : mysqld_exporter
      # (depuis l'exporter 0.15.0, DATA_SOURCE_NAME est supprimé :
      #  on passe par --mysqld.address/--mysqld.username + MYSQLD_EXPORTER_PASSWORD)
      - name: exporter
        image: prom/mysqld-exporter:latest
        ports:
        - containerPort: 9104
        env:
        - name: MYSQLD_EXPORTER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-exporter-secret
              key: password
        args:
        - --mysqld.address=localhost:3306
        - --mysqld.username=exporter
        - --collect.info_schema.tables
        - --collect.info_schema.innodb_metrics
        - --collect.global_status
        - --collect.slave_status
```

**2. ServiceMonitor : Scraping automatique**
```yaml
apiVersion: monitoring.coreos.com/v1  
kind: ServiceMonitor  
metadata:  
  name: mariadb-metrics
spec:
  selector:
    matchLabels:
      app: mariadb
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**3. Dashboards Grafana essentiels**
- **Overview Dashboard** : QPS, connections, latency
- **InnoDB Dashboard** : Buffer pool, redo log, locks
- **Replication Dashboard** : Lag, threads status, GTID
- **Query Analysis** : Slow queries, full table scans
- **Resource Usage** : CPU, memory, disk I/O

**4. Alert Rules Prometheus**
```yaml
groups:
- name: mariadb-alerts
  rules:
  # Replication lag critique
  - alert: MariaDBReplicationLag
    expr: mysql_slave_status_seconds_behind_master > 300
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Replication lag > 5 minutes"
      
  # Connexions near max
  - alert: MariaDBConnectionsHigh
    expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Connections at 80% capacity"
      
  # Buffer pool hit ratio faible
  - alert: MariaDBBufferPoolHitRatioLow
    expr: |
      (mysql_global_status_innodb_buffer_pool_read_requests - 
       mysql_global_status_innodb_buffer_pool_reads) / 
      mysql_global_status_innodb_buffer_pool_read_requests < 0.9
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Buffer pool hit ratio < 90%"
```

---

### Migrations automatisées : Zero-downtime schema changes

#### gh-ost : GitHub's Online Schema Migration Tool

**Principe** : Triggerless replication pour éviter overhead.

```bash
# Migration en ligne : Ajout d'index sans downtime
gh-ost \
  --user="ghost" \
  --password="***" \
  --host="mariadb-primary.prod.svc.cluster.local" \
  --database="production" \
  --table="orders" \
  --alter="ADD INDEX idx_customer_email (customer_id, email)" \
  --exact-rowcount \
  --concurrent-rowcount \
  --default-retries=120 \
  --chunk-size=1000 \
  --throttle-control-replicas="mariadb-replica-1,mariadb-replica-2" \
  --max-lag-millis=1500 \
  --initially-drop-ghost-table \
  --initially-drop-old-table \
  --ok-to-drop-table \
  --execute

# Features :
# - Throttling automatique si lag > 1.5s
# - Pause/resume manuel possible
# - Rollback avant cut-over final
# - Validation pré et post-migration
```

**Workflow gh-ost** :
1. Création table ghost (`_orders_gho`)
2. Copy data en chunks (throttled)
3. Capture changes via binlog parsing
4. Validation finale
5. Atomic swap (RENAME TABLE)

**Avantages** :
- ✅ Zero downtime (table accessible en R+W)
- ✅ Throttling intelligent (préserve réplication)
- ✅ Testable (dry-run mode)
- ✅ Reversible (avant cut-over)

#### Flyway : Database migrations as code

**Concept** : Versioned SQL scripts, immutable.

```bash
# Structure de projet Flyway
migrations/
├── V1__initial_schema.sql
├── V2__add_users_table.sql
├── V3__add_orders_table.sql
├── V4__add_indexes.sql
├── V5__add_audit_triggers.sql
└── R__refresh_materialized_views.sql  # Repeatable

# Configuration flyway.conf
flyway.url=jdbc:mariadb://mariadb-primary:3306/production  
flyway.user=flyway_user  
flyway.password=${FLYWAY_PASSWORD}  
flyway.schemas=production  
flyway.table=flyway_schema_history  
flyway.locations=filesystem:./migrations  

# Exécution dans CI/CD
flyway migrate

# Flyway track l'état dans table metadata
SELECT * FROM flyway_schema_history;
# installed_rank | version | description        | success | installed_on
# 1              | 1       | initial schema     | 1       | 2025-01-15 10:00:00
# 2              | 2       | add users table    | 1       | 2025-02-01 14:23:45
# 3              | 3       | add orders table   | 1       | 2025-03-15 09:12:33
```

**Intégration CI/CD** :
```yaml
# GitLab CI exemple
.deploy_migration:
  stage: deploy
  image: flyway/flyway:latest
  script:
    - flyway info
    - flyway validate
    - flyway migrate
  only:
    - main
    
deploy_staging:
  extends: .deploy_migration
  environment:
    name: staging
    
deploy_production:
  extends: .deploy_migration
  environment:
    name: production
  when: manual  # Requires approval
```

---

### GitOps : Git as single source of truth

#### Flux CD : Continuous deployment from Git

**Architecture** :
```plaintext
Git Repository (GitHub/GitLab)
    ↓
Flux controllers (in-cluster)
    ↓
Kubernetes resources
    ↓
MariaDB Operator reconciles
    ↓
MariaDB clusters running
```

**Configuration Flux** :
```yaml
# flux-system/gotk-sync.yaml
apiVersion: source.toolkit.fluxcd.io/v1  
kind: GitRepository  
metadata:  
  name: mariadb-configs
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/company/mariadb-infrastructure
  ref:
    branch: main
  secretRef:
    name: github-credentials
    
---
apiVersion: kustomize.toolkit.fluxcd.io/v1  
kind: Kustomization  
metadata:  
  name: mariadb-prod
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: mariadb-configs
  healthChecks:
    - apiVersion: k8s.mariadb.com/v1alpha1
      kind: MariaDB
      name: mariadb-galera
      namespace: production
```

**Workflow GitOps** :
1. Developer commits changes to Git
2. CI validates (linting, dry-run)
3. Merge to main branch
4. Flux detects change (polling/webhook)
5. Flux applies changes to cluster
6. Operator reconciles MariaDB state
7. Validation et health checks
8. Success ou automatic rollback

**Bénéfices** :
- ✅ **Audit trail** : Every change tracked in Git
- ✅ **Rollback** : `git revert` = infrastructure rollback
- ✅ **Review process** : Pull requests for infrastructure changes
- ✅ **Disaster recovery** : Cluster reconstruction from Git
- ✅ **Multi-cluster** : Same config, different environments

---

## ✅ Compétences acquises

À la fin de cette huitième partie, vous serez capable de :

### Infrastructure as Code
- ✅ **Écrire** des playbooks Ansible pour déploiement MariaDB
- ✅ **Créer** des modules Terraform pour infrastructure cloud
- ✅ **Gérer** les secrets avec Vault/Secret Manager
- ✅ **Versionner** l'infrastructure dans Git
- ✅ **Implémenter** des environments séparés (dev/staging/prod)

### Conteneurisation et orchestration
- ✅ **Construire** des images Docker optimisées pour MariaDB
- ✅ **Orchestrer** avec Docker Compose pour développement
- ✅ **Déployer** StatefulSets Kubernetes pour production
- ✅ **Gérer** le stockage persistant avec PV/PVC
- ✅ **Utiliser** les operators pour automation (mariadb-operator)
- ✅ **Implémenter** Galera clusters dans Kubernetes

### Pipeline CI/CD
- ✅ **Créer** des pipelines pour migrations de schéma
- ✅ **Automatiser** les tests de base de données
- ✅ **Implémenter** des approval gates pour production
- ✅ **Gérer** les migrations avec Flyway/Liquibase
- ✅ **Effectuer** des migrations zero-downtime avec gh-ost

### Observabilité moderne
- ✅ **Configurer** Prometheus pour métriques MariaDB
- ✅ **Créer** des dashboards Grafana informatifs
- ✅ **Définir** des alert rules pertinents
- ✅ **Implémenter** logs centralisés (ELK/Loki)
- ✅ **Tracer** les requêtes application→database
- ✅ **Monitorer** les golden signals (latency, traffic, errors, saturation)

### GitOps et gestion déclarative
- ✅ **Implémenter** GitOps avec Flux/ArgoCD
- ✅ **Gérer** databases as code
- ✅ **Versionner** les migrations de schéma
- ✅ **Automatiser** la reconciliation d'état
- ✅ **Détecter** et corriger la drift

### DevOps best practices
- ✅ **Appliquer** le principe d'immutabilité
- ✅ **Automatiser** tout ce qui est répétable
- ✅ **Documenter** as code (runbooks, playbooks)
- ✅ **Implémenter** self-healing patterns
- ✅ **Mesurer** et améliorer le MTTR (Mean Time To Recovery)

---

## 🎓 Parcours recommandés

Cette partie est **absolument essentielle** pour DevOps et Cloud, et devient critique pour DBA modernes.

| Parcours | Importance | Justification |
|----------|------------|---------------|
| ⚙️ **DevOps/SRE** | ⭐⭐⭐ ABSOLUMENT CRITIQUE | L'automatisation et le cloud-native sont au cœur du métier DevOps. Sans ces compétences, impossible d'opérer dans des environnements modernes. |
| ☁️ **Cloud Architect** | ⭐⭐⭐ ABSOLUMENT CRITIQUE | Conception d'architectures cloud nécessite maîtrise complète de IaC, Kubernetes, et patterns cloud-native pour databases. |
| 🔐 **DBA Moderne** | ⭐⭐⭐ ESSENTIEL | L'ère du DBA qui SSH sur serveurs est révolue. Les DBA modernes doivent maîtriser Kubernetes, IaC, et GitOps pour rester pertinents. |
| 🔧 **Tech Lead/Développeur Senior** | ⭐⭐ TRÈS UTILE | Comprendre le déploiement et l'orchestration aide à écrire du code database-friendly et facilite la collaboration avec DevOps. |

### Pourquoi cette partie est transformatrice ?

#### Pour les DevOps/SRE
Le métier DevOps consiste à **automatiser et fiabiliser** :
- Déploiements 100x par jour impossibles sans automation
- GitOps permet rollback en secondes vs heures manuelles
- Observabilité proactive évite 80% des incidents
- IaC élimine configuration drift et snowflakes

#### Pour les Cloud Architects
Architectures modernes sont **cloud-native by design** :
- Kubernetes est devenu le standard de facto
- Operators transforment les bases en ressources K8s natives
- Multi-cloud nécessite abstractions (Terraform)
- Costs optimization via automation (scale-to-zero)

#### Pour les DBA modernes
Évolution du rôle DBA **du manuel vers l'automation** :
- Configuration via code (Ansible, Terraform) vs SSH
- Gestion déclarative (operators) vs commandes impératives
- Monitoring proactif (Prometheus) vs réactif
- Migrations automatisées (CI/CD) vs scripts manuels

**Le DBA qui refuse d'évoluer devient obsolète. Le DBA qui adopte DevOps devient indispensable.**

---

## 🏗️ Architectures cloud-native et patterns d'orchestration

### Architecture cloud-native type

```plaintext
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │          Ingress / Load Balancer                   │     │
│  └────────────────────────────────────────────────────┘     │
│                          ↓                                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │        Application Pods (Stateless)                │     │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐            │     │
│  │  │ App1 │  │ App2 │  │ App3 │  │ App4 │            │     │
│  │  └──────┘  └──────┘  └──────┘  └──────┘            │     │
│  └────────────────────────────────────────────────────┘     │
│                          ↓                                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │          Service Mesh (Istio/Linkerd)              │     │
│  │  - Traffic management                              │     │
│  │  - Mutual TLS                                      │     │
│  │  - Observability                                   │     │
│  └────────────────────────────────────────────────────┘     │
│                          ↓                                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │        MariaDB StatefulSet (Galera)                │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │     │
│  │  │ Node 0   │  │ Node 1   │  │ Node 2   │          │     │
│  │  │ Primary  │  │ Replica  │  │ Replica  │          │     │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘          │     │
│  │       │             │             │                │     │
│  │  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐          │     │
│  │  │ PVC 0    │  │ PVC 1    │  │ PVC 2    │          │     │
│  │  │ 100Gi    │  │ 100Gi    │  │ 100Gi    │          │     │
│  │  └──────────┘  └──────────┘  └──────────┘          │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │            Observability Stack                     │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │     │
│  │  │Prometheus│  │ Grafana  │  │  Loki    │          │     │
│  │  └──────────┘  └──────────┘  └──────────┘          │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │            MariaDB Operator                        │     │
│  │  - Reconciliation loops                            │     │
│  │  - Backup/Restore automation                       │     │
│  │  - Scaling et upgrades                             │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
        ┌─────────────────────────────────────┐
        │      External Services              │
        │  - S3 (backups)                     │
        │  - Secret Manager (credentials)     │
        │  - DNS (service discovery)          │
        └─────────────────────────────────────┘
```

### Patterns d'orchestration essentiels

#### Pattern 1 : StatefulSet pour databases

**Problème** : Databases nécessitent identité stable et stockage persistant.

**Solution** : StatefulSet avec ordinal index.

```yaml
apiVersion: apps/v1  
kind: StatefulSet  
metadata:  
  name: mariadb
spec:
  serviceName: mariadb-headless
  replicas: 3
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:12.3
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

**Garanties StatefulSet** :
- 🔒 Identités stables : `mariadb-0`, `mariadb-1`, `mariadb-2`
- 💾 Volumes persistants liés à chaque pod
- 📋 Ordered deployment et termination
- 🌐 Headless service pour DNS stable

#### Pattern 2 : Operator pour automation

**Problème** : Opérations complexes (backup, failover, upgrade) nécessitent logique custom.

**Solution** : Kubernetes Operator avec custom controllers.

```go
// Pseudo-code operator reconciliation loop
func (r *MariaDBReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch MariaDB custom resource
    mariadb := &mariadbv1.MariaDB{}
    if err := r.Get(ctx, req.NamespacedName, mariadb); err != nil {
        return ctrl.Result{}, err
    }
    
    // 2. Ensure StatefulSet exists and matches desired spec
    if err := r.reconcileStatefulSet(ctx, mariadb); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. Ensure Services exist
    if err := r.reconcileServices(ctx, mariadb); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. Check health and perform actions
    if err := r.checkHealthAndRecover(ctx, mariadb); err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. Handle backups if scheduled
    if err := r.reconcileBackups(ctx, mariadb); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

**Bénéfices operator** :
- 🤖 Automation de logique complexe
- 🔄 Reconciliation continue (desired vs actual state)
- 🛠️ Day-2 operations automatisées
- 📚 Domain knowledge codifiée

#### Pattern 3 : Sidecar pour monitoring

**Problème** : Monitoring nécessite accès métriques sans modifier container principal.

**Solution** : Sidecar pattern avec exporter.

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: mariadb-with-monitoring
spec:
  containers:
  # Container principal
  - name: mariadb
    image: mariadb:12.3
    ports:
    - containerPort: 3306
      
  # Sidecar : mysqld_exporter (config via .my.cnf ou variables, cf. §16.9.1)
  - name: metrics-exporter
    image: prom/mysqld-exporter:latest
    ports:
    - containerPort: 9104
      name: metrics
    env:
    - name: MYSQLD_EXPORTER_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mariadb-exporter-secret
          key: password
    args:
    - --mysqld.address=localhost:3306
    - --mysqld.username=exporter
```

**Avantages sidecar** :
- ✅ Séparation des responsabilités
- ✅ Réutilisabilité (même exporter pour tous pods)
- ✅ Scaling indépendant
- ✅ Lifecycle lié (startup/shutdown ensemble)

#### Pattern 4 : Init containers pour bootstrap

**Problème** : Configuration ou initialisation nécessaire avant démarrage principal.

**Solution** : Init containers exécutés séquentiellement.

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: mariadb-initialized
spec:
  initContainers:
  # 1. Wait for storage to be ready
  - name: wait-for-storage
    image: busybox
    command: ['sh', '-c', 'until ls /var/lib/mysql; do sleep 1; done']
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
      
  # 2. Restore from backup if needed
  - name: restore-backup
    image: mariadb:12.3
    command: ['sh', '-c', 'if [ -f /backup/backup.sql ]; then mariadb < /backup/backup.sql; fi']
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
    - name: backup
      mountPath: /backup
      
  containers:
  - name: mariadb
    image: mariadb:12.3
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
      
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mariadb-data
  - name: backup
    emptyDir: {}
```

---

## 🔀 GitOps : Gestion déclarative et versionnée

### Principes fondamentaux GitOps

#### 1️⃣ Déclaratif over impératif

```yaml
# ✅ Déclaratif (GitOps)
# Fichier : mariadb-prod.yaml dans Git
apiVersion: k8s.mariadb.com/v1alpha1  
kind: MariaDB  
metadata:  
  name: production-db
spec:
  replicas: 3
  storage:
    size: 500Gi

# L'operator reconcile automatiquement l'état désiré
# Peu importe l'état actuel, il convergera vers spec

# ❌ Impératif (Old way)
$ kubectl scale statefulset mariadb --replicas=3
$ kubectl exec mariadb-0 -- mysql -e "CREATE DATABASE prod"
# État non tracé, non reproductible, drift inévitable
```

#### 2️⃣ Git as single source of truth

```plaintext
Git Repository (main branch)
    ├── clusters/
    │   ├── production/
    │   │   ├── mariadb-cluster.yaml
    │   │   ├── backups.yaml
    │   │   └── monitoring.yaml
    │   ├── staging/
    │   │   └── ...
    │   └── development/
    │       └── ...
    ├── migrations/
    │   ├── V1__initial_schema.sql
    │   ├── V2__add_users.sql
    │   └── ...
    └── infrastructure/
        ├── terraform/
        └── ansible/
```

**Workflow** :
1. All changes via Pull Request
2. CI validates (linting, dry-run, tests)
3. Peer review and approval
4. Merge to main
5. Flux/ArgoCD detects change
6. Automatic deployment
7. Health validation
8. Success or automatic rollback

#### 3️⃣ Continuous reconciliation

```plaintext
┌──────────────┐
│  Git Repo    │ ← Single source of truth
└───────┬──────┘
        │
        ↓ (Polling every 1min or webhook)
┌───────────────┐
│  Flux/ArgoCD  │ ← Continuous reconciliation
└───────┬───────┘
        │
        ↓ (Detect drift)
┌───────────────────────┐
│ Kubernetes Cluster    │
│  ┌─────────────────┐  │
│  │ Desired State   │  │ ← From Git
│  └─────────────────┘  │
│         vs            │
│  ┌─────────────────┐  │
│  │ Actual State    │  │ ← Running in cluster
│  └─────────────────┘  │
└───────────────────────┘
        │
        ↓ (If drift detected)
    Auto-correction
```

**Bénéfices** :
- 🔍 **Drift detection** : Alerting si manual changes
- 🔄 **Self-healing** : Automatic correction
- 📜 **Audit trail** : Every change in Git history
- 🔙 **Easy rollback** : `git revert` = infrastructure rollback

#### 4️⃣ Separation of concerns

```plaintext
├── Platform Team (maintains cluster & operators)
│   └── flux-system/
│       ├── gotk-components.yaml
│       └── gotk-sync.yaml
│
├── Database Team (manages MariaDB configs)
│   └── databases/
│       ├── mariadb-prod.yaml
│       └── backups.yaml
│
└── Application Teams (consume databases)
    └── apps/
        ├── api-service/
        │   └── database-connection.yaml
        └── web-frontend/
            └── ...
```

**Avantages** :
- ✅ Teams work independently in different directories
- ✅ RBAC enforcement at Git level
- ✅ Clear ownership and responsibilities
- ✅ Reduced blast radius of changes

---

### Challenges spécifiques databases avec GitOps

#### Challenge 1 : State management

**Problème** : Databases sont stateful, rollback n'est pas simple.

```yaml
# Git commit 1 : Database avec 1M lignes
spec:
  storage:
    size: 100Gi

# Git commit 2 : Scale up storage
spec:
  storage:
    size: 500Gi

# Données ajoutées : +10M lignes

# Git revert : Impossible de réduire storage avec données
# → Rollback de config ≠ rollback de données
```

**Solutions** :
- ✅ One-way operations (storage expansion only)
- ✅ Validation gates pour changements risqués
- ✅ Backup before destructive operations
- ✅ Separate data migrations from config changes

#### Challenge 2 : Schema migrations

**Problème** : Migrations de schéma ne sont pas idempotentes.

```sql
-- V1__add_email_column.sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);

-- Si rejouée : ERROR 1060 (42S21): Duplicate column name 'email'
```

**Solutions** :
- ✅ Version-based migrations (Flyway, Liquibase)
- ✅ Idempotent scripts (IF NOT EXISTS)
- ✅ Separate migration pipeline from config
- ✅ Manual approval for production migrations

#### Challenge 3 : Secrets management

**Problème** : Passwords ne doivent pas être dans Git.

```yaml
# ❌ NEVER do this
apiVersion: v1  
kind: Secret  
metadata:  
  name: mariadb-root-password
stringData:
  password: "SuperSecretPassword123"  # In Git = BAD
```

**Solutions** :
```yaml
# ✅ External Secrets Operator
apiVersion: external-secrets.io/v1beta1  
kind: ExternalSecret  
metadata:  
  name: mariadb-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: mariadb-secret
  data:
  - secretKey: password
    remoteRef:
      key: prod/mariadb/root-password

# Secret récupéré depuis AWS Secrets Manager
# Git contient seulement la référence, pas le secret
```

**Alternatives** :
- ✅ Sealed Secrets (encrypted secrets in Git)
- ✅ SOPS (encrypted YAML files)
- ✅ Vault Agent injector
- ✅ Cloud provider secret managers

---

## 📊 Métriques DevOps pour databases

### DORA Metrics appliquées aux databases

#### 1. Deployment Frequency
```
Avant automatisation : 1 deployment / mois (maintenance windows)  
Après automatisation : 10-20 deployments / jour (schema migrations via CI/CD)  

→ Amélioration : 200-600x
```

#### 2. Lead Time for Changes
```
Avant : 2-4 semaines (planning → approval → maintenance window)  
Après : 2-4 heures (PR → review → automated deployment)  

→ Amélioration : 100x
```

#### 3. Mean Time to Recovery (MTTR)
```
Avant : 2-8 heures (manual diagnosis + fix + deployment)  
Après : 5-15 minutes (automated detection + rollback/failover)  

→ Amélioration : 24-96x
```

#### 4. Change Failure Rate
```
Avant : 15-30% (manual errors, insufficient testing)  
Après : 0-5% (automated testing, validation gates)  

→ Amélioration : 3-6x
```

### Métriques spécifiques database automation

| Métrique | Cible | Mesure |
|----------|-------|--------|
| **Time to provision new DB** | <15 minutes | Time from request to ready |
| **Backup success rate** | >99.9% | Successful backups / total attempts |
| **Restore test frequency** | Weekly | Automated restore validation |
| **Schema migration success** | >95% | Successful migrations / total |
| **Replication lag** | <5 seconds | Average lag across all replicas |
| **Alert response time** | <5 minutes | Time from alert to acknowledgment |
| **Infrastructure drift** | 0 instances | Manual changes detected and corrected |

---

## 🚀 Prêt pour le Database DevOps ?

Cette partie vous transformera en **expert DevOps pour bases de données**, capable de :

- ✅ Automatiser 100% du cycle de vie database (provisioning → upgrade → backup → monitoring)
- ✅ Déployer et gérer MariaDB dans Kubernetes avec operators
- ✅ Implémenter GitOps pour infrastructure et migrations
- ✅ Créer des pipelines CI/CD pour schema changes
- ✅ Monitorer avec stack moderne (Prometheus/Grafana)
- ✅ Effectuer des migrations zero-downtime (gh-ost)
- ✅ Être reconnu comme spécialiste cloud-native databases

Les compétences de cette partie sont **parmi les plus demandées** en 2025. Les professionnels maîtrisant Kubernetes, IaC, et GitOps appliqués aux bases de données sont rares et peuvent prétendre à des salaires 40-60% supérieurs aux DBA traditionnels.

**Préparez-vous à révolutionner la manière dont vous gérez les bases de données.** ☁️

---

## ➡️ Prochaine étape

**Module 16 : DevOps et Automatisation** → Maîtrisez Infrastructure as Code avec Ansible et Terraform, orchestration Kubernetes, CI/CD pour databases, monitoring moderne, et GitOps.

Bienvenue dans l'ère du Database DevOps ! 🚀

---

**MariaDB** : Version 12.3 LTS  
**Kubernetes** : 1.28+  
**Flux CD** : 2.x  
**Terraform** : 1.6+  

⏭️ [DevOps et Automatisation](/16-devops-automatisation/README.md)

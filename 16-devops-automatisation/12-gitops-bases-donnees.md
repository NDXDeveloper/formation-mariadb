🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.12 GitOps pour les bases de données

> **Niveau** : Expert  
> **Durée estimée** : 8-9 heures  
> **Prérequis** : 
> - Sections 16.1-16.11 du chapitre DevOps maîtrisées
> - Expérience avec Git workflows (GitFlow, trunk-based)
> - Compréhension de Kubernetes operators
> - Notions de declarative infrastructure
> - Familiarité avec ArgoCD ou FluxCD

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes du GitOps et leur application aux bases de données
- **Déployer** MariaDB avec ArgoCD ou FluxCD de manière déclarative
- **Gérer** le schéma de base de données as code
- **Détecter** et corriger automatiquement la configuration drift
- **Sécuriser** les secrets dans un workflow GitOps
- **Implémenter** multi-environnements (dev, staging, production)
- **Automatiser** rollbacks et disaster recovery
- **Appliquer** les best practices GitOps pour databases stateful

---

## Introduction au GitOps

### Qu'est-ce que le GitOps ?

**GitOps** = Operational model where Git is the **single source of truth** for declarative infrastructure and applications.

```
┌──────────────────────────────────────────────────────────────┐
│                    GitOps Principles                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1️⃣  DECLARATIVE                                             │
│     État désiré décrit dans Git (YAML, pas scripts)          │
│     Exemple: "Je veux 3 replicas MariaDB" (pas "scale to 3") │
│                                                              │
│  2️⃣  VERSIONED & IMMUTABLE                                   │
│     Tout changement = Git commit                             │
│     History complète, rollback facile                        │
│                                                              │
│  3️⃣  PULLED AUTOMATICALLY                                    │
│     Agent (ArgoCD, FluxCD) pull depuis Git                   │
│     Pas de push vers cluster (sécurité)                      │
│                                                              │
│  4️⃣  CONTINUOUSLY RECONCILED                                 │
│     Agent vérifie cluster state vs Git state                 │
│     Auto-correction si drift détecté                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### GitOps vs Traditional CI/CD

```
┌─────────────────────────────────────────────────────────────────┐
│              Traditional CI/CD                                  │
│                                                                 │
│  Developer                                                      │
│      ↓ git push                                                 │
│  Git Repository                                                 │
│      ↓ webhook                                                  │
│  CI Server (Jenkins, GitHub Actions)                            │
│      ↓ kubectl apply (PUSH to cluster)                          │
│  Kubernetes Cluster                                             │
│                                                                 │
│  Problèmes:                                                     │
│  ❌ CI server needs cluster credentials                         │
│  ❌ No drift detection                                          │
│  ❌ Manual state management                                     │
└─────────────────────────────────────────────────────────────────┘

                           VS

┌─────────────────────────────────────────────────────────────────┐
│                    GitOps                                       │
│                                                                 │
│  Developer                                                      │
│      ↓ git push                                                 │
│  Git Repository (single source of truth)                        │
│      ↑ PULL (every 3min)                                        │
│  GitOps Agent in Cluster (ArgoCD, FluxCD)                       │
│      ↓ reconcile                                                │
│  Kubernetes Cluster                                             │
│                                                                 │
│  Avantages:                                                     │
│  ✅ No external cluster access needed                           │
│  ✅ Automatic drift detection & correction                      │
│  ✅ Declarative state management                                │
│  ✅ Audit trail (Git history)                                   │
│  ✅ Easy rollback (git revert)                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Défis GitOps pour bases de données

**Bases de données = état persistant ≠ applications stateless**

```
┌──────────────────────────────────────────────────────────────┐
│         Challenges: GitOps + Stateful Databases              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. DATA PERSISTENCE                                         │
│     App: Delete pod → New pod identical                      │
│     DB: Delete pod → PERTE DE DONNÉES ⚠️                     │
│     Solution: PersistentVolumes avec reclaimPolicy Retain    │
│                                                              │
│  2. SCHEMA EVOLUTION                                         │
│     App: Deploy new version → Replace old                    │
│     DB: Schema change → Migration forward-only               │
│     Solution: Schema as code (Flyway, Liquibase)             │
│                                                              │
│  3. SECRETS MANAGEMENT                                       │
│     Passwords, keys ne peuvent être dans Git en clair        │
│     Solution: Sealed Secrets, External Secrets Operator      │
│                                                              │
│  4. DRIFT DETECTION                                          │
│     App: Facile (image tag)                                  │
│     DB: Complexe (données, users, permissions)               │
│     Solution: Custom health checks + operators               │
│                                                              │
│  5. ROLLBACK                                                 │
│     App: Instant (previous image)                            │
│     DB: Délicat (data déjà modifiées)                        │
│     Solution: Backward-compatible changes + backups          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## ArgoCD pour MariaDB

### Architecture ArgoCD

```
┌─────────────────────────────────────────────────────────────────┐
│                    ArgoCD Architecture                          │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Git Repository                            │ │
│  │                                                            │ │
│  │  mariadb-gitops/                                           │ │
│  │  ├── base/                                                 │ │
│  │  │   ├── mariadb.yaml                                      │ │
│  │  │   ├── configmap.yaml                                    │ │
│  │  │   └── service.yaml                                      │ │
│  │  ├── overlays/                                             │ │
│  │  │   ├── dev/                                              │ │
│  │  │   ├── staging/                                          │ │
│  │  │   └── production/                                       │ │
│  │  └── argocd/                                               │ │
│  │      └── application.yaml                                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            ▲                                    │
│                            │ git pull (every 3min)              │
│  ┌─────────────────────────┴───────────────────────────────────┐│
│  │              ArgoCD in Kubernetes                           ││
│  │                                                             ││
│  │  ┌──────────────────────────────────────────────────────┐   ││
│  │  │  Application Controller                              │   ││
│  │  │  - Sync Git → Cluster                                │   ││
│  │  │  - Health checks                                     │   ││
│  │  │  - Auto-sync or manual                               │   ││
│  │  └──────────────────────────────────────────────────────┘   ││
│  │                            │                                ││
│  │                            ▼ kubectl apply                  ││
│  │  ┌──────────────────────────────────────────────────────┐   ││
│  │  │  Kubernetes Resources                                │   ││
│  │  │  - StatefulSet mariadb                               │   ││
│  │  │  - PVCs                                              │   ││
│  │  │  - Services                                          │   ││
│  │  │  - ConfigMaps                                        │   ││
│  │  └──────────────────────────────────────────────────────┘   ││
│  └─────────────────────────────────────────────────────────────┘│
│                            ▲                                    │
│                            │ status reporting                   │
│  ┌─────────────────────────┴───────────────────────────────────┐│
│  │              ArgoCD UI / CLI                                ││
│  │  - Visualization                                            ││
│  │  - Sync management                                          ││
│  │  - Rollback                                                 ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Installation ArgoCD

```bash
# 1. Installer ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Exposer ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 3. Obtenir mot de passe admin initial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 4. Login via CLI
argocd login localhost:8080
argocd account update-password
```

### Structure repository Git

```
mariadb-gitops/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── mariadb-statefulset.yaml
│   ├── mariadb-service.yaml
│   ├── mariadb-configmap.yaml
│   └── mariadb-pvc.yaml
│
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── mariadb-patch.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── mariadb-patch.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── mariadb-patch.yaml
│       └── sealed-secrets.yaml
│
└── argocd/
    ├── application-dev.yaml
    ├── application-staging.yaml
    └── application-production.yaml
```

### Base manifests

**base/kustomization.yaml** :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: databases

resources:
- namespace.yaml
- mariadb-statefulset.yaml
- mariadb-service.yaml
- mariadb-configmap.yaml

configMapGenerator:
- name: mariadb-config
  files:
  - my.cnf
```

**base/mariadb-statefulset.yaml** :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  serviceName: mariadb
  replicas: 1  # Override par overlays
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
        image: mariadb:11.8  # Base version
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard  # Override par overlays
      resources:
        requests:
          storage: 10Gi  # Override par overlays
```

### Overlays par environnement

**overlays/production/kustomization.yaml** :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
- ../../base

patchesStrategicMerge:
- mariadb-patch.yaml

resources:
- sealed-secrets.yaml

# Labels communs
commonLabels:
  environment: production
  managed-by: argocd

# Annotations
commonAnnotations:
  argocd.argoproj.io/sync-wave: "1"
```

**overlays/production/mariadb-patch.yaml** :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  replicas: 3  # Production: 3 replicas
  
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: mariadb
            topologyKey: kubernetes.io/hostname
      
      containers:
      - name: mariadb
        resources:
          requests:
            cpu: 4
            memory: 16Gi
          limits:
            cpu: 8
            memory: 32Gi
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: premium-ssd  # Production: SSD rapide
      resources:
        requests:
          storage: 500Gi  # Production: Large storage
```

### ArgoCD Application

**argocd/application-production.yaml** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mariadb-production
  namespace: argocd
  # Finalizer pour cleanup
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  # Git source
  source:
    repoURL: https://github.com/myorg/mariadb-gitops.git
    targetRevision: main
    path: overlays/production
  
  # Destination cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # Sync policy
  syncPolicy:
    automated:
      prune: false  # ⚠️ false pour databases (éviter suppression accidentelle)
      selfHeal: true  # Auto-correction si drift
      allowEmpty: false
    
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true  # Supprimer en dernier (si prune enabled)
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Health assessment
  ignoreDifferences:
  - group: apps
    kind: StatefulSet
    jsonPointers:
    - /spec/volumeClaimTemplates  # Ignore VolumeClaimTemplate changes after creation
  
  # Notifications
  revisionHistoryLimit: 10
```

**Déployer Application** :

```bash
# Via CLI
argocd app create -f argocd/application-production.yaml

# Ou via kubectl
kubectl apply -f argocd/application-production.yaml

# Vérifier status
argocd app get mariadb-production

# Sync manuel (si automated=false)
argocd app sync mariadb-production

# Voir diff
argocd app diff mariadb-production

# Rollback
argocd app rollback mariadb-production
```

### Health Checks personnalisés

**Lua script pour ArgoCD** :

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Custom health check pour StatefulSet MariaDB
  resource.customizations.health.apps_StatefulSet: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.readyReplicas ~= nil and obj.status.readyReplicas == obj.spec.replicas then
        hs.status = "Healthy"
        hs.message = "All replicas ready"
        return hs
      end
      if obj.status.currentReplicas ~= nil and obj.status.currentReplicas > 0 then
        hs.status = "Progressing"
        hs.message = "Waiting for replicas: " .. obj.status.readyReplicas .. "/" .. obj.spec.replicas
        return hs
      end
    end
    hs.status = "Degraded"
    hs.message = "No replicas ready"
    return hs
  
  # Custom health check pour MariaDB custom resource (operator)
  resource.customizations.health.mariadb.mmontes.io_MariaDB: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = condition.message
            return hs
          end
          if condition.type == "Ready" and condition.status == "False" then
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for MariaDB to be ready"
    return hs
```

---

## FluxCD Alternative

### FluxCD vs ArgoCD

| Aspect | ArgoCD | FluxCD |
|--------|--------|--------|
| **UI** | ✅ Web UI riche | ⚠️ CLI only (+ Weave GitOps UI payant) |
| **Architecture** | Controller centralisé | Controllers distribués (GitOps Toolkit) |
| **Multi-cluster** | ✅ Native | ✅ Via Flux multi-tenancy |
| **Helm support** | ✅ Native | ✅ HelmRelease CRD |
| **Kustomize** | ✅ Native | ✅ Kustomization CRD |
| **Image automation** | ⚠️ Via Image Updater | ✅ Native (Image Reflector) |
| **Notifications** | ✅ Slack, etc. | ✅ Notification Controller |
| **Learning curve** | ⭐⭐⭐ Moyenne | ⭐⭐⭐⭐ Difficile |
| **CNCF** | Graduated | Graduated |

**Recommandation** :
- **ArgoCD** : Si besoin UI, simplicité, multi-cluster
- **FluxCD** : Si GitOps natif, image automation, Helm avancé

### Installation FluxCD

```bash
# 1. Installer Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# 2. Vérifier prérequis
flux check --pre

# 3. Bootstrap Flux (connecte à Git)
flux bootstrap github \
  --owner=myorg \
  --repository=mariadb-gitops \
  --branch=main \
  --path=clusters/production \
  --personal
```

### HelmRelease pour MariaDB

**flux/mariadb-helmrelease.yaml** :

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami

---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mariadb
  namespace: databases
spec:
  interval: 5m
  chart:
    spec:
      chart: mariadb
      version: '>=11.8.0'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 1m
  
  # Values
  values:
    image:
      tag: 11.8-debian-11
    
    architecture: replication
    
    auth:
      rootPassword: ${MARIADB_ROOT_PASSWORD}
      replicationPassword: ${MARIADB_REPLICATION_PASSWORD}
    
    primary:
      persistence:
        enabled: true
        storageClass: premium-ssd
        size: 500Gi
      
      resources:
        requests:
          cpu: 4
          memory: 16Gi
        limits:
          cpu: 8
          memory: 32Gi
      
      configuration: |-
        [mysqld]
        character-set-server=utf8mb4
        collation-server=utf8mb4_unicode_ci
        max_connections=500
        innodb_buffer_pool_size=12G
        innodb_log_file_size=2G
    
    secondary:
      replicaCount: 2
      persistence:
        enabled: true
        size: 500Gi
      resources:
        requests:
          cpu: 2
          memory: 8Gi
  
  # Post-install/upgrade
  postRenderers:
  - kustomize:
      patchesStrategicMerge:
      - apiVersion: apps/v1
        kind: StatefulSet
        metadata:
          name: mariadb-primary
        spec:
          template:
            spec:
              affinity:
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchLabels:
                        app.kubernetes.io/name: mariadb
                    topologyKey: kubernetes.io/hostname
```

---

## Schema as Code

### Approche déclarative du schéma

**Problème** : Schema changes traditionnellement impératifs (migrations SQL)

**Solution** : Schema as code avec GitOps

```
┌──────────────────────────────────────────────────────────────┐
│              Schema Management Workflow                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Developer modifie schema.sql dans Git                    │
│                                                              │
│  2. Pull Request → Review → Merge                            │
│                                                              │
│  3. ArgoCD détecte changement                                │
│                                                              │
│  4. Pre-sync hook execute Flyway migration                   │
│                                                              │
│  5. ArgoCD sync reste des ressources                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Job Flyway via ArgoCD Hook

**migrations/flyway-job.yaml** :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mariadb-migration
  annotations:
    # ArgoCD hook: Exécuter AVANT sync
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: flyway
        image: flyway/flyway:10.4
        command:
        - flyway
        - migrate
        - -url=jdbc:mysql://mariadb.databases.svc.cluster.local:3306/myapp
        - -user=root
        - -password=${MYSQL_ROOT_PASSWORD}
        - -locations=filesystem:/flyway/sql
        volumeMounts:
        - name: migrations
          mountPath: /flyway/sql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
      volumes:
      - name: migrations
        configMap:
          name: mariadb-migrations

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-migrations
data:
  V001__initial_schema.sql: |
    CREATE TABLE IF NOT EXISTS users (
      id BIGINT AUTO_INCREMENT PRIMARY KEY,
      email VARCHAR(255) NOT NULL UNIQUE,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
  
  V002__add_users_name.sql: |
    ALTER TABLE users 
    ADD COLUMN IF NOT EXISTS name VARCHAR(100);
  
  V003__add_orders_table.sql: |
    CREATE TABLE IF NOT EXISTS orders (
      id BIGINT AUTO_INCREMENT PRIMARY KEY,
      user_id BIGINT NOT NULL,
      total_amount DECIMAL(10,2),
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (user_id) REFERENCES users(id)
    );
```

### Schema Operator (alternative)

**SchemaHero** : Kubernetes operator pour schema as code

```yaml
apiVersion: schemas.schemahero.io/v1alpha4
kind: Database
metadata:
  name: mariadb
spec:
  connection:
    mysql:
      uri:
        valueFrom:
          secretKeyRef:
            name: mariadb-secret
            key: uri

---
apiVersion: schemas.schemahero.io/v1alpha4
kind: Table
metadata:
  name: users
spec:
  database: mariadb
  name: users
  schema:
    mysql:
      primaryKey: [id]
      columns:
      - name: id
        type: bigint
        attributes:
          autoIncrement: true
      - name: email
        type: varchar(255)
        constraints:
          notNull: true
        attributes:
          unique: true
      - name: name
        type: varchar(100)
      - name: created_at
        type: timestamp
        default: CURRENT_TIMESTAMP
```

**Workflow** :
1. Developer modifie Table CRD
2. Commit → Git
3. ArgoCD sync
4. SchemaHero génère migration
5. Apply migration (après approbation optionnelle)

---

## Secrets Management

### Le problème des secrets dans Git

**❌ JAMAIS FAIRE** :

```yaml
# ❌ Secret en clair dans Git
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
stringData:
  root-password: "super-secret-password"  # ❌❌❌
```

### Solution 1 : Sealed Secrets

**Principe** : Chiffrer secrets côté client, déchiffrer dans cluster

```bash
# 1. Installer Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 2. Installer kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar xfz kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# 3. Créer secret (pas appliqué)
kubectl create secret generic mariadb-secret \
  --from-literal=root-password='super-secret' \
  --dry-run=client -o yaml > mariadb-secret.yaml

# 4. Sceller secret (chiffré avec clé publique du cluster)
kubeseal -f mariadb-secret.yaml -w mariadb-sealed-secret.yaml

# 5. Commit sealed secret dans Git (safe ✅)
git add mariadb-sealed-secret.yaml
git commit -m "Add MariaDB sealed secret"
```

**mariadb-sealed-secret.yaml** (versionnable) :

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mariadb-secret
  namespace: production
spec:
  encryptedData:
    root-password: AgByQRZ7L9... # Chiffré, safe dans Git ✅
  template:
    metadata:
      name: mariadb-secret
      namespace: production
```

### Solution 2 : External Secrets Operator

**Principe** : Synchro depuis vault externe (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)

```yaml
# 1. Installer External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace

# 2. SecretStore (connexion à AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
# 3. ExternalSecret (référence secret AWS)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mariadb-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: mariadb-secret  # Secret Kubernetes créé
    creationPolicy: Owner
  data:
  - secretKey: root-password
    remoteRef:
      key: prod/mariadb/root-password  # Chemin dans AWS
  - secretKey: replication-password
    remoteRef:
      key: prod/mariadb/replication-password
```

**Avantages** :
- ✅ Secrets jamais dans Git
- ✅ Rotation automatique
- ✅ Audit trail dans vault
- ✅ Fine-grained access control

---

## Drift Detection & Auto-Remediation

### Qu'est-ce que le drift ?

**Drift** = Différence entre état Git (desired) et état cluster (actual)

```
┌──────────────────────────────────────────────────────────────┐
│                    Configuration Drift                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Git (Source of Truth):                                      │
│  replicas: 3                                                 │
│                                                              │
│  Cluster (Actual):                                           │
│  replicas: 5  ⚠️ (Quelqu'un a fait kubectl scale ...)        │
│                                                              │
│  → DRIFT DETECTED                                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### ArgoCD Auto-Sync

**Avec selfHeal** :

```yaml
spec:
  syncPolicy:
    automated:
      selfHeal: true  # Auto-correction du drift
```

**Behavior** :
1. ArgoCD détecte drift (poll every 3min)
2. Auto-sync vers état Git
3. Logs action dans ArgoCD UI

**⚠️ Attention pour databases** :
- `prune: false` recommandé (éviter suppression PVC accidentelle)
- `selfHeal: true` OK si confiance dans Git

### Monitoring du drift

```yaml
# PrometheusRule pour alerter sur drift
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-drift-alerts
spec:
  groups:
  - name: argocd
    interval: 1m
    rules:
    - alert: ArgoCDOutOfSync
      expr: |
        argocd_app_info{sync_status!="Synced"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "ArgoCD app {{ $labels.name }} out of sync"
        description: "Application has been out of sync for >10min"
```

---

## Multi-Environments Workflow

### Structure Git avancée

```
mariadb-gitops/
├── apps/
│   └── mariadb/
│       ├── base/
│       │   └── ... (manifests communs)
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── production/
│
├── clusters/
│   ├── dev/
│   │   ├── flux-system/
│   │   └── mariadb.yaml  # Kustomization pointant vers apps/mariadb/overlays/dev
│   ├── staging/
│   │   ├── flux-system/
│   │   └── mariadb.yaml
│   └── production/
│       ├── flux-system/
│       └── mariadb.yaml
│
└── infrastructure/
    ├── sealed-secrets/
    ├── external-secrets/
    └── monitoring/
```

### Promotion entre environnements

**Workflow GitOps** :

```
┌─────────────────────────────────────────────────────────────────┐
│              Environment Promotion Workflow                     │
│                                                                 │
│  1️⃣  Developer commit changement → branch feature/new-index     │
│                                                                 │
│  2️⃣  Pull Request → Review → Merge to main                      │
│                                                                 │
│  3️⃣  ArgoCD/Flux sync DEV automatiquement                       │
│     (targetRevision: main)                                      │
│                                                                 │
│  4️⃣  Tests automatisés en DEV passent ✅                        │
│                                                                 │
│  5️⃣  Promotion STAGING:                                         │
│     Créer Git tag: v1.2.3                                       │
│     ArgoCD/Flux sync STAGING                                    │
│     (targetRevision: v1.2.3)                                    │
│                                                                 │
│  6️⃣  Tests staging + Approval manuel                            │
│                                                                 │
│  7️⃣  Promotion PRODUCTION:                                      │
│     Update production/kustomization.yaml                        │
│     Commit: "Promote v1.2.3 to production"                      │
│     ArgoCD/Flux sync PRODUCTION                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Kustomization par env** :

```yaml
# clusters/dev/mariadb.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mariadb
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/mariadb/overlays/dev
  prune: false  # Safety pour databases
  wait: true
  timeout: 10m

---
# clusters/production/mariadb.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mariadb
  namespace: flux-system
spec:
  interval: 10m  # Moins fréquent en production
  sourceRef:
    kind: GitRepository
    name: flux-system
    # Tag spécifique en production (pas main)
  path: ./apps/mariadb/overlays/production
  prune: false
  wait: true
  timeout: 30m
  
  # Health checks
  healthChecks:
  - apiVersion: apps/v1
    kind: StatefulSet
    name: mariadb
    namespace: production
```

---

## Disaster Recovery avec GitOps

### Backup as Code

```yaml
# backup-cronjob.yaml (versionné dans Git)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mariadb-backup
  namespace: production
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # Après MariaDB
spec:
  schedule: "0 2 * * *"  # 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: mariadb:11.8
            command:
            - /bin/bash
            - -c
            - |
              # Backup avec mariabackup
              mariabackup --backup \
                --target-dir=/backup/$(date +%Y%m%d_%H%M%S) \
                --user=root \
                --password=$MYSQL_ROOT_PASSWORD \
                --host=mariadb-0.mariadb.production.svc
              
              # Upload to S3
              tar czf - /backup/* | \
              aws s3 cp - s3://mariadb-backups/production/backup-$(date +%Y%m%d).tar.gz
            
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: root-password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            emptyDir: {}
```

### Disaster Recovery Procedure

**Scenario** : Cluster production détruit, recréer depuis Git

```bash
# 1. Nouveau cluster Kubernetes
# 2. Installer Flux
flux bootstrap github \
  --owner=myorg \
  --repository=mariadb-gitops \
  --branch=main \
  --path=clusters/production

# 3. Flux sync automatiquement:
#    - Infrastructure (sealed-secrets, external-secrets)
#    - MariaDB StatefulSet
#    - Services, ConfigMaps

# 4. Restore données depuis backup
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: mariadb-restore
  namespace: production
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: restore
        image: mariadb:11.8
        command:
        - /bin/bash
        - -c
        - |
          # Download from S3
          aws s3 cp s3://mariadb-backups/production/backup-20251214.tar.gz - | tar xzf - -C /backup
          
          # Stop MariaDB
          kubectl scale statefulset mariadb --replicas=0
          
          # Restore
          mariabackup --prepare --target-dir=/backup/20251214_020000
          mariabackup --copy-back --target-dir=/backup/20251214_020000
          
          # Start MariaDB
          kubectl scale statefulset mariadb --replicas=3
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-mariadb-0
EOF
```

**Recovery Time** : ~15-30 minutes (vs hours sans GitOps)

---

## Best Practices GitOps pour Databases

### 1. Separation of Concerns

```
mariadb-gitops/
├── infrastructure/      # Ops team
│   ├── namespaces/
│   ├── rbac/
│   └── operators/
│
├── platform/           # Platform team
│   ├── databases/
│   ├── monitoring/
│   └── networking/
│
└── applications/       # Dev teams
    ├── app1/
    └── app2/
```

**RBAC** : Dev teams = read-only sur databases

### 2. Immutable Infrastructure

```yaml
# ❌ Mutable: Edit in place
kubectl edit statefulset mariadb

# ✅ Immutable: Change Git, let GitOps sync
git commit -m "Increase MariaDB memory"
git push
# ArgoCD/Flux sync automatiquement
```

### 3. Progressive Rollouts

```yaml
# Canary avec Flagger (compatible GitOps)
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: mariadb
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: mariadb
  progressDeadlineSeconds: 600
  service:
    port: 3306
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
```

### 4. Audit Trail

**Tout changement tracé dans Git** :

```bash
# Qui a augmenté les replicas ?
git log --all --grep="replicas" -- overlays/production/

# Quand a-t-on changé la config ?
git log --follow -- base/mariadb-configmap.yaml

# Rollback facile
git revert abc123
git push  # ArgoCD sync automatiquement
```

### 5. Automated Testing

```yaml
# GitHub Actions: Test avant merge
name: Test MariaDB GitOps

on:
  pull_request:
    paths:
    - 'apps/mariadb/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Kubernetes (kind)
      uses: helm/kind-action@v1
    
    - name: Install ArgoCD
      run: |
        kubectl create namespace argocd
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    
    - name: Deploy MariaDB
      run: |
        kubectl apply -f argocd/application-dev.yaml
        kubectl wait --for=condition=Synced app/mariadb-dev -n argocd --timeout=300s
    
    - name: Verify MariaDB Health
      run: |
        kubectl wait --for=condition=Ready pod/mariadb-0 -n dev --timeout=300s
        kubectl exec mariadb-0 -n dev -- mysql -uroot -p$ROOT_PASSWORD -e "SELECT 1"
```

---

## ✅ Points clés à retenir

- **GitOps = Git as single source of truth** : Tout changement via Git
- **Pull model** : Agent dans cluster pull depuis Git (sécurité)
- **Declarative** : État désiré, pas étapes impératives
- **Auto-reconciliation** : Drift détecté et corrigé automatiquement
- **ArgoCD vs FluxCD** : ArgoCD (UI, simple), FluxCD (natif, image automation)
- **Secrets** : Sealed Secrets ou External Secrets (jamais clair dans Git)
- **Schema as code** : Migrations Flyway dans Git, Pre-sync hooks
- **Multi-env** : Overlays Kustomize, promotion via tags
- **Disaster Recovery** : Cluster recréé depuis Git en 15-30min
- **Immutability** : Éditer Git, jamais kubectl edit directement
- **Audit trail** : Git log = historique complet

💡 **Golden rule** : "If it's not in Git, it doesn't exist."

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 ArgoCD](https://argo-cd.readthedocs.io/)
- [📖 FluxCD](https://fluxcd.io/docs/)
- [📖 Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [📖 External Secrets Operator](https://external-secrets.io/)

### Guides
- [📝 GitOps Principles (CNCF)](https://opengitops.dev/)
- [📝 GitOps for Databases (Weave Works)](https://www.weave.works/blog/gitops-for-databases)

### Outils
- [🔧 SchemaHero](https://schemahero.io/)
- [🔧 Kustomize](https://kustomize.io/)

---

## 🎓 Conclusion du Chapitre 16

**Félicitations !** Vous avez terminé le chapitre **16. DevOps et Automatisation** 🎉

### Ce que vous avez appris

Au cours de ces 12 sections, vous avez maîtrisé :

1. **Infrastructure as Code** (Terraform, Ansible)
2. **Conteneurisation** (Docker, volumes, compose)
3. **Orchestration Kubernetes** (StatefulSets, PV/PVC)
4. **Operators** (Community et Enterprise)
5. **CI/CD** pour databases (pipelines complets)
6. **Migrations** (Flyway, Liquibase, gh-ost, pt-osc)
7. **Monitoring** (Prometheus, Grafana, mysqld_exporter)
8. **Observabilité** (Logs, Metrics, Traces)
9. **Alerting** (SLOs/SLIs, incident response)
10. **GitOps** (ArgoCD, FluxCD, declarative)

### Stack DevOps complète pour MariaDB

```
┌─────────────────────────────────────────────────────────────────┐
│                 Modern MariaDB DevOps Stack                     │
│                                                                 │
│  Infrastructure                 Deployment                      │
│  ├─ Terraform (AWS/GCP/Azure)   ├─ Docker (containers)          │
│  └─ Ansible (configuration)     ├─ Kubernetes (orchestration)   │
│                                  ├─ Operators (automation)      │
│                                  └─ GitOps (ArgoCD/Flux)        │
│                                                                 │
│  CI/CD                          Migrations                      │
│  ├─ GitHub Actions              ├─ Flyway (versioning)          │
│  ├─ GitLab CI                   ├─ Liquibase (rollback)         │
│  └─ Jenkins                     └─ gh-ost (zero-downtime)       │
│                                                                 │
│  Observability                  Incident Response               │
│  ├─ Prometheus (metrics)        ├─ Alertmanager (routing)       │
│  ├─ Loki (logs)                 ├─ PagerDuty (on-call)          │
│  ├─ Jaeger (traces)             ├─ Runbooks (procedures)        │
│  └─ Grafana (visualization)     └─ Post-mortems (learning)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Prochaines étapes

Vous êtes maintenant équipé pour :
- ✅ Déployer MariaDB en production avec confiance
- ✅ Automatiser complètement le cycle de vie
- ✅ Monitorer et observer en temps réel
- ✅ Répondre efficacement aux incidents
- ✅ Améliorer continuellement via GitOps

**Continuez vers** :
- Chapitre 17 : Sécurité avancée
- Chapitre 18 : Optimisation performance
- Ou : Mise en pratique sur projet réel !

---

**MariaDB** : Version 11.8 LTS
**ArgoCD** : v2.9+
**FluxCD** : v2.2+

⏭️ [Intégration et Développement](/17-integration-developpement/README.md)

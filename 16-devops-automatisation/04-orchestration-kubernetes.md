🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.4 Orchestration avec Kubernetes

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 5-6 heures  
> **Prérequis** : 
> - Compréhension solide de Kubernetes (pods, services, deployments)
> - Section 16.3 Docker maîtrisée
> - Expérience avec kubectl et manifests YAML
> - Notions de stockage Kubernetes (PV, PVC, StorageClasses)
> - Familiarité avec les concepts de StatefulSets

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les défis spécifiques du déploiement de MariaDB sur Kubernetes
- **Déployer** MariaDB avec StatefulSets et stockage persistant
- **Configurer** PersistentVolumes et PersistentVolumeClaims pour données critiques
- **Gérer** les services Kubernetes pour l'accès à MariaDB (headless, ClusterIP, LoadBalancer)
- **Implémenter** ConfigMaps et Secrets pour la configuration
- **Orchestrer** des clusters MariaDB (réplication, Galera) sur Kubernetes
- **Utiliser** des Operators pour automatiser la gestion du cycle de vie
- **Appliquer** les best practices de production (affinité, taints/tolerations, backup)

---

## Introduction

### Kubernetes vs Docker : Pourquoi orchestrer ?

Docker (section précédente) est excellent pour :
- ✅ Développement local
- ✅ Tests d'intégration
- ✅ Petites installations (< 5 conteneurs)

**Mais pour la production à grande échelle** :

```
┌──────────────────────────────────────────────────────────────┐
│                    Docker Compose                            │
│                   (Single Host)                              │
├──────────────────────────────────────────────────────────────┤
│  ❌ Un seul serveur (SPOF - Single Point of Failure)         │
│  ❌ Scaling manuel et limité                                 │
│  ❌ Pas de self-healing automatique                          │
│  ❌ Pas de rolling updates intelligents                      │
│  ❌ Pas de load balancing intégré                            │
│  ❌ Gestion manuelle des crashes                             │
└──────────────────────────────────────────────────────────────┘

                            VS

┌──────────────────────────────────────────────────────────────┐
│                      Kubernetes                              │
│                    (Cluster)                                 │
├──────────────────────────────────────────────────────────────┤
│  ✅ Multi-nœuds (haute disponibilité native)                 │
│  ✅ Auto-scaling horizontal et vertical                      │
│  ✅ Self-healing (redémarre pods automatiquement)            │
│  ✅ Rolling updates avec rollback automatique                │
│  ✅ Load balancing et service discovery natifs               │
│  ✅ Orchestration déclarative (desired state)                │
│  ✅ Gestion avancée du stockage (CSI drivers)                │
│  ✅ Network policies et sécurité granulaire                  │
│  ✅ Operators pour automation avancée                        │
└──────────────────────────────────────────────────────────────┘
```

### Défis de MariaDB sur Kubernetes

Les bases de données **stateful** comme MariaDB posent des défis uniques sur Kubernetes :

| Défi | Solution Kubernetes |
|------|---------------------|
| **Identité stable** | StatefulSets (noms prévisibles: mariadb-0, mariadb-1) |
| **Stockage persistant** | PersistentVolumes + PersistentVolumeClaims |
| **Ordre de démarrage** | StatefulSets avec podManagementPolicy |
| **Network identity** | Headless Services (DNS stable) |
| **Configuration unique** | ConfigMaps + init containers |
| **Secrets** | Kubernetes Secrets (chiffrés at-rest) |
| **Backup/Restore** | CronJobs + Volume Snapshots |
| **Haute disponibilité** | Multi-replica avec anti-affinity |
| **Monitoring** | ServiceMonitors (Prometheus Operator) |

### Architecture Kubernetes pour MariaDB

**Stack complète** :

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Control Plane                           │ │
│  │  (API Server, Scheduler, Controller Manager, etcd)         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Worker Nodes                            │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │              Namespace: databases                    │  │ │
│  │  │                                                      │  │ │
│  │  │  ┌────────────────────────────────────────────────┐  │  │ │
│  │  │  │         StatefulSet: mariadb                   │  │  │ │
│  │  │  │                                                │  │  │ │
│  │  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐   │  │  │ │
│  │  │  │  │mariadb-0  │  │mariadb-1  │  │mariadb-2  │   │  │  │ │
│  │  │  │  │  (Pod)    │  │  (Pod)    │  │  (Pod)    │   │  │  │ │
│  │  │  │  │           │  │           │  │           │   │  │  │ │
│  │  │  │  │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │   │  │  │ │
│  │  │  │  │ │MariaDB│ │  │ │MariaDB│ │  │ │MariaDB│ │   │  │  │ │
│  │  │  │  │ │11.8   │ │  │ │11.8   │ │  │ │11.8   │ │   │  │  │ │
│  │  │  │  │ └───┬───┘ │  │ └───┬───┘ │  │ └───┬───┘ │   │  │  │ │
│  │  │  │  └─────┼─────┘  └─────┼─────┘  └─────┼─────┘   │  │  │ │
│  │  │  │        │              │              │         │  │  │ │
│  │  │  │  ┌─────▼──────┐┌──────▼─────┐┌───────▼────┐    │  │  │ │
│  │  │  │  │    PVC     ││    PVC     ││    PVC     │    │  │  │ │
│  │  │  │  │ (100GB)    ││ (100GB)    ││ (100GB)    │    │  │  │ │
│  │  │  │  └────────────┘└────────────┘└────────────┘    │  │  │ │
│  │  │  └────────────────────────────────────────────────┘  │  │ │
│  │  │                                                      │  │ │
│  │  │  ┌────────────────────────────────────────────────┐  │  │ │
│  │  │  │         Service: mariadb (Headless)            │  │  │ │
│  │  │  │  DNS: mariadb-0.mariadb.databases.svc          │  │  │ │
│  │  │  │       mariadb-1.mariadb.databases.svc          │  │  │ │
│  │  │  │       mariadb-2.mariadb.databases.svc          │  │  │ │
│  │  │  └────────────────────────────────────────────────┘  │  │ │
│  │  │                                                      │  │ │
│  │  │  ┌────────────────────────────────────────────────┐  │  │ │
│  │  │  │      ConfigMap: mariadb-config                 │  │  │ │
│  │  │  │      Secret: mariadb-secret                    │  │  │ │
│  │  │  └────────────────────────────────────────────────┘  │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Persistent Storage Layer                      │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │ │
│  │  │   PV 1   │  │   PV 2   │  │   PV 3   │                  │ │
│  │  │  (SSD)   │  │  (SSD)   │  │  (SSD)   │                  │ │
│  │  └──────────┘  └──────────┘  └──────────┘                  │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Concepts Kubernetes fondamentaux pour MariaDB

### 1. StatefulSets

**Pourquoi pas Deployment ?**

```yaml
# ❌ Deployment - NE PAS utiliser pour MariaDB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
spec:
  replicas: 3
  # Problèmes:
  # - Pods avec noms aléatoires (mariadb-abc123, mariadb-def456)
  # - Pas d'ordre de création/suppression
  # - Pas de stockage stable par pod
  # - DNS non prévisible
```

```yaml
# ✅ StatefulSet - RECOMMANDÉ pour MariaDB
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  replicas: 3
  # Avantages:
  # - Pods avec noms ordonnés (mariadb-0, mariadb-1, mariadb-2)
  # - Création/suppression séquentielle
  # - PVC dédié par pod (mariadb-data-mariadb-0)
  # - DNS stable (mariadb-0.mariadb.default.svc.cluster.local)
```

**Caractéristiques clés des StatefulSets** :

| Caractéristique | Description | Importance pour MariaDB |
|-----------------|-------------|-------------------------|
| **Identité stable** | Noms prévisibles (0, 1, 2...) | Essentiel pour réplication |
| **Stockage stable** | PVC persiste même si pod recréé | Données ne sont jamais perdues |
| **Ordre déterministe** | Pods créés/supprimés dans l'ordre | Bootstrap cluster correct |
| **DNS stable** | `<pod>.<service>.<namespace>.svc` | Service discovery fiable |
| **Rolling update** | Un pod à la fois | Pas de downtime |

### 2. PersistentVolumes (PV) et PersistentVolumeClaims (PVC)

**Architecture stockage Kubernetes** :

```
┌──────────────────────────────────────────────────────────┐
│                    Storage Flow                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Admin crée StorageClass                              │
│     └─> Définit type de stockage (SSD, HDD, NFS...)      │
│                                                          │
│  2. StatefulSet demande du stockage via PVC              │
│     └─> "Je veux 100GB avec StorageClass 'fast-ssd'"     │
│                                                          │
│  3. Kubernetes provisionne automatiquement PV            │
│     └─> Crée volume réel (EBS, GCE PD, Azure Disk...)    │
│                                                          │
│  4. PVC se lie au PV (binding)                           │
│     └─> Association 1:1 permanente                       │
│                                                          │
│  5. Pod monte le PVC                                     │
│     └─> Accès au stockage via /var/lib/mysql             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Exemple de flux complet** :

```yaml
# 1. StorageClass (admin)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# 2. StatefulSet avec volumeClaimTemplates
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi

# Kubernetes crée automatiquement:
# - PVC: data-mariadb-0 (100GB)
# - PV: pv-abc123 (100GB AWS EBS gp3)
# - Binding: data-mariadb-0 <-> pv-abc123
```

### 3. Services Kubernetes

**Types de services pour MariaDB** :

```yaml
# 1. Headless Service (RECOMMANDÉ pour StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: mariadb
spec:
  clusterIP: None  # Headless = pas de ClusterIP
  ports:
  - port: 3306
  selector:
    app: mariadb

# DNS créé:
# - mariadb-0.mariadb.databases.svc.cluster.local -> IP du pod 0
# - mariadb-1.mariadb.databases.svc.cluster.local -> IP du pod 1
# - mariadb.databases.svc.cluster.local -> Tous les pods (round-robin)
```

```yaml
# 2. ClusterIP Service (pour accès applicatif)
apiVersion: v1
kind: Service
metadata:
  name: mariadb-read
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mariadb
    role: replica  # Seulement les replicas

# Accessible depuis pods du cluster:
# mysql -h mariadb-read.databases.svc.cluster.local -P 3306
```

```yaml
# 3. LoadBalancer (exposition externe - ⚠️ utiliser avec prudence)
apiVersion: v1
kind: Service
metadata:
  name: mariadb-external
spec:
  type: LoadBalancer
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mariadb
    role: primary
  # Cloud provider crée un Load Balancer avec IP publique
```

### 4. ConfigMaps et Secrets

**ConfigMap pour configuration MariaDB** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-config
  namespace: databases
data:
  my.cnf: |
    [mysqld]
    # Charset (🆕 MariaDB 11.8)
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
    
    # Network
    bind-address = 0.0.0.0
    port = 3306
    max_connections = 500
    
    # InnoDB
    innodb_buffer_pool_size = 4G
    innodb_buffer_pool_instances = 4
    innodb_log_file_size = 1G
    innodb_flush_log_at_trx_commit = 1
    innodb_flush_method = O_DIRECT
    
    # 🆕 MariaDB 11.8 optimizations
    innodb_alter_copy_bulk = ON
    innodb_io_capacity = 2000
    
    # Logs
    slow_query_log = 1
    slow_query_log_file = /var/log/mysql/slow.log
    long_query_time = 2
    
    # 🆕 MariaDB 11.8 - TLS
    require_secure_transport = ON
    
    [client]
    default-character-set = utf8mb4
```

**Secret pour mots de passe** :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
  namespace: databases
type: Opaque
stringData:  # Sera automatiquement base64 encodé
  root-password: "SuperSecureRootPassword123!"
  replication-password: "ReplicationPassword456!"
  
# Ou depuis fichier:
# kubectl create secret generic mariadb-secret \
#   --from-file=root-password=./secrets/root-password.txt \
#   --from-file=replication-password=./secrets/repl-password.txt
```

**Utilisation dans Pod** :

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: mariadb
    image: mariadb:11.8
    env:
    # Depuis Secret
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mariadb-secret
          key: root-password
    
    # Depuis ConfigMap
    - name: MAX_CONNECTIONS
      valueFrom:
        configMapKeyRef:
          name: mariadb-config
          key: max_connections
    
    volumeMounts:
    # Monter ConfigMap comme fichier
    - name: config
      mountPath: /etc/mysql/conf.d/my.cnf
      subPath: my.cnf
  
  volumes:
  - name: config
    configMap:
      name: mariadb-config
```

---

## Déploiement MariaDB sur Kubernetes : Approche progressive

### Étape 1 : Namespace dédié

```yaml
# 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: databases
  labels:
    name: databases
    purpose: stateful-services
```

```bash
kubectl apply -f 01-namespace.yaml
```

### Étape 2 : StorageClass

```yaml
# 02-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mariadb-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs  # Adapter selon cloud provider
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # Optimise placement
reclaimPolicy: Retain  # Ne pas supprimer PV si PVC supprimé
```

**Alternatives pour autres cloud providers** :

```yaml
# GCP
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd

# Azure
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed

# Local (pour tests)
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

### Étape 3 : Secret

```yaml
# 03-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
  namespace: databases
type: Opaque
stringData:
  root-password: "ChangeMe123!"
  user-password: "AppPassword456!"
```

```bash
# Créer depuis ligne de commande (plus sécurisé)
kubectl create secret generic mariadb-secret \
  --namespace=databases \
  --from-literal=root-password=$(openssl rand -base64 32) \
  --from-literal=user-password=$(openssl rand -base64 32)
```

### Étape 4 : ConfigMap

```yaml
# 04-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-config
  namespace: databases
data:
  my.cnf: |
    [mysqld]
    # Basic settings
    user = mysql
    pid-file = /var/run/mysqld/mysqld.pid
    socket = /var/run/mysqld/mysqld.sock
    port = 3306
    basedir = /usr
    datadir = /var/lib/mysql
    tmpdir = /tmp
    
    # Character set (🆕 MariaDB 11.8 default)
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
    
    # Networking
    bind-address = 0.0.0.0
    max_connections = 500
    max_connect_errors = 1000000
    
    # InnoDB Settings
    innodb_buffer_pool_size = 4G
    innodb_buffer_pool_instances = 4
    innodb_log_file_size = 1G
    innodb_log_buffer_size = 256M
    innodb_flush_log_at_trx_commit = 1
    innodb_flush_method = O_DIRECT
    innodb_file_per_table = 1
    
    # 🆕 MariaDB 11.8 performance
    innodb_alter_copy_bulk = ON
    innodb_io_capacity = 2000
    innodb_io_capacity_max = 4000
    
    # Query cache (disabled)
    query_cache_type = 0
    query_cache_size = 0
    
    # Temporary tables
    tmp_table_size = 256M
    max_heap_table_size = 256M
    
    # 🆕 MariaDB 11.8 - Temp space control
    max_tmp_space_usage = 10G
    
    # Logging
    log_error = /var/log/mysql/error.log
    slow_query_log = 1
    slow_query_log_file = /var/log/mysql/slow.log
    long_query_time = 2
    log_queries_not_using_indexes = 1
    
    # Binary logging (for replication)
    log_bin = /var/lib/mysql/mysql-bin
    binlog_format = ROW
    expire_logs_days = 7
    max_binlog_size = 100M
    
    # GTID
    gtid_strict_mode = ON
    
    [client]
    port = 3306
    socket = /var/run/mysqld/mysqld.sock
    default-character-set = utf8mb4
```

### Étape 5 : Headless Service

```yaml
# 05-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  namespace: databases
  labels:
    app: mariadb
spec:
  clusterIP: None  # Headless service
  ports:
  - port: 3306
    targetPort: 3306
    name: mysql
  selector:
    app: mariadb
  publishNotReadyAddresses: true  # Important pour StatefulSet
```

### Étape 6 : StatefulSet

```yaml
# 06-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  namespace: databases
  labels:
    app: mariadb
spec:
  serviceName: mariadb
  replicas: 3
  selector:
    matchLabels:
      app: mariadb
  
  # Gestion des pods
  podManagementPolicy: OrderedReady  # Créer pods dans l'ordre
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Mettre à jour tous les pods
  
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      # Anti-affinity: ne pas mettre 2 pods MariaDB sur même nœud
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - mariadb
            topologyKey: kubernetes.io/hostname
      
      # Init container pour setup permissions
      initContainers:
      - name: init-mariadb
        image: mariadb:11.8
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Générer server-id depuis nom du pod
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo "[mysqld]" > /mnt/conf.d/server-id.cnf
          echo "server-id=$((100 + $ordinal))" >> /mnt/conf.d/server-id.cnf
          
          # Copier configuration
          cp /mnt/config-map/my.cnf /mnt/conf.d/
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
      containers:
      - name: mariadb
        image: mariadb:11.8
        
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
        - name: MYSQL_DATABASE
          value: "myapp"
        - name: MYSQL_USER
          value: "appuser"
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: user-password
        
        ports:
        - containerPort: 3306
          name: mysql
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        - name: logs
          mountPath: /var/log/mysql
        
        # Resources
        resources:
          requests:
            cpu: "1"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
        
        # Liveness probe (pod vivant?)
        livenessProbe:
          exec:
            command:
            - bash
            - "-c"
            - mysqladmin ping -uroot -p$MYSQL_ROOT_PASSWORD
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe (pod prêt?)
        readinessProbe:
          exec:
            command:
            - bash
            - "-c"
            - |
              mysql -uroot -p$MYSQL_ROOT_PASSWORD -e "SELECT 1"
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mariadb-config
      - name: logs
        emptyDir: {}
  
  # PersistentVolumeClaim template (créé automatiquement par pod)
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mariadb-storage
      resources:
        requests:
          storage: 100Gi
```

### Déploiement complet

```bash
# 1. Créer namespace
kubectl apply -f 01-namespace.yaml

# 2. Créer StorageClass
kubectl apply -f 02-storageclass.yaml

# 3. Créer Secret
kubectl apply -f 03-secret.yaml

# 4. Créer ConfigMap
kubectl apply -f 04-configmap.yaml

# 5. Créer Service
kubectl apply -f 05-service.yaml

# 6. Créer StatefulSet
kubectl apply -f 06-statefulset.yaml

# Ou tout en une fois
kubectl apply -f .

# Vérifier status
kubectl get statefulset -n databases
kubectl get pods -n databases
kubectl get pvc -n databases
kubectl get pv

# Logs
kubectl logs -n databases mariadb-0 -f

# Se connecter
kubectl exec -it -n databases mariadb-0 -- mysql -uroot -p
```

**Vérification** :

```bash
# Status StatefulSet
kubectl get sts -n databases mariadb

# Output:
# NAME      READY   AGE
# mariadb   3/3     5m

# Pods
kubectl get pods -n databases -l app=mariadb

# Output:
# NAME        READY   STATUS    RESTARTS   AGE
# mariadb-0   1/1     Running   0          5m
# mariadb-1   1/1     Running   0          4m
# mariadb-2   1/1     Running   0          3m

# PVC
kubectl get pvc -n databases

# Output:
# NAME               STATUS   VOLUME     CAPACITY   STORAGECLASS
# data-mariadb-0     Bound    pv-abc     100Gi      mariadb-storage
# data-mariadb-1     Bound    pv-def     100Gi      mariadb-storage
# data-mariadb-2     Bound    pv-ghi     100Gi      mariadb-storage

# Tester connexion
kubectl run -it --rm mysql-client \
  --image=mariadb:11.8 \
  --restart=Never \
  --namespace=databases \
  -- mysql -h mariadb-0.mariadb.databases.svc.cluster.local \
          -uroot -p
```

---

## Patterns de déploiement avancés

### 1. Réplication Primary-Replica

**Architecture** :

```
mariadb-0 (Primary - R/W)
    ↓ binlog replication
    ├─→ mariadb-1 (Replica - R/O)
    └─→ mariadb-2 (Replica - R/O)
```

**Init container pour setup réplication** :

```yaml
initContainers:
- name: init-replication
  image: mariadb:11.8
  command:
  - bash
  - "-c"
  - |
    set -ex
    [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
    ordinal=${BASH_REMATCH[1]}
    
    # Si replica (ordinal > 0), configurer réplication
    if [ $ordinal -gt 0 ]; then
      cat > /mnt/conf.d/replication.cnf <<EOF
    [mysqld]
    read_only = 1
    log_slave_updates = 1
    relay_log = /var/lib/mysql/relay-bin
    EOF
      
      # Script pour setup replication (exécuté au démarrage)
      cat > /docker-entrypoint-initdb.d/setup-replication.sh <<'EOSCRIPT'
    #!/bin/bash
    until mysql -h mariadb-0.mariadb -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1" &>/dev/null; do
      echo "Waiting for primary..."
      sleep 5
    done
    
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} <<EOSQL
    STOP SLAVE;
    CHANGE MASTER TO
      MASTER_HOST='mariadb-0.mariadb.databases.svc.cluster.local',
      MASTER_USER='root',
      MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}',
      MASTER_USE_GTID=slave_pos;
    START SLAVE;
    EOSQL
    EOSCRIPT
      chmod +x /docker-entrypoint-initdb.d/setup-replication.sh
    fi
  volumeMounts:
  - name: conf
    mountPath: /mnt/conf.d
  - name: init-scripts
    mountPath: /docker-entrypoint-initdb.d
```

**Services séparés pour R/W et R/O** :

```yaml
---
# Service Primary (Read-Write)
apiVersion: v1
kind: Service
metadata:
  name: mariadb-primary
  namespace: databases
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mariadb
    statefulset.kubernetes.io/pod-name: mariadb-0

---
# Service Replicas (Read-Only)
apiVersion: v1
kind: Service
metadata:
  name: mariadb-read
  namespace: databases
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mariadb
  # Exclure le primary via label selector (nécessite labeling manuel)
```

### 2. Galera Cluster (Multi-Master)

**Configuration Galera** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-galera-config
  namespace: databases
data:
  galera.cnf: |
    [galera]
    wsrep_on = ON
    wsrep_provider = /usr/lib/galera/libgalera_smm.so
    wsrep_cluster_name = "k8s-galera-cluster"
    wsrep_cluster_address = "gcomm://mariadb-0.mariadb.databases.svc.cluster.local,mariadb-1.mariadb.databases.svc.cluster.local,mariadb-2.mariadb.databases.svc.cluster.local"
    wsrep_sst_method = mariabackup
    wsrep_sst_auth = "root:${MYSQL_ROOT_PASSWORD}"
    
    # Node specific (sera overridé par init container)
    wsrep_node_address = "$(POD_IP)"
    wsrep_node_name = "$(HOSTNAME)"
    
    binlog_format = ROW
    default_storage_engine = InnoDB
    innodb_autoinc_lock_mode = 2
    innodb_flush_log_at_trx_commit = 0
    
    # Performance
    wsrep_slave_threads = 4
```

**Init container pour Galera** :

```yaml
initContainers:
- name: init-galera
  image: mariadb:11.8
  env:
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: HOSTNAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  command:
  - bash
  - "-c"
  - |
    set -ex
    
    # Copier config Galera et remplacer variables
    sed "s/\$(POD_IP)/$POD_IP/g; s/\$(HOSTNAME)/$HOSTNAME/g" \
      /mnt/config-map/galera.cnf > /mnt/conf.d/galera.cnf
    
    # Bootstrap premier nœud
    [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
    ordinal=${BASH_REMATCH[1]}
    
    if [ $ordinal -eq 0 ]; then
      # Premier nœud: bootstrap cluster
      echo "[mysqld]" > /mnt/conf.d/bootstrap.cnf
      echo "wsrep_new_cluster = 1" >> /mnt/conf.d/bootstrap.cnf
    else
      # Autres nœuds: attendre que le premier soit prêt
      until mysql -h mariadb-0.mariadb -uroot -p${MYSQL_ROOT_PASSWORD} \
        -e "SHOW STATUS LIKE 'wsrep_cluster_size'" &>/dev/null; do
        echo "Waiting for cluster bootstrap..."
        sleep 5
      done
    fi
  volumeMounts:
  - name: conf
    mountPath: /mnt/conf.d
  - name: config-map
    mountPath: /mnt/config-map
```

💡 **Note** : Galera sur Kubernetes est complexe. Utiliser **mariadb-operator** (voir section 16.5) est fortement recommandé.

### 3. Backup automatisé avec CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mariadb-backup
  namespace: databases
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: mariadb:11.8
            command:
            - bash
            - "-c"
            - |
              set -e
              
              DATE=$(date +%Y%m%d_%H%M%S)
              BACKUP_FILE="/backups/mariadb-backup-${DATE}.sql.gz"
              
              # Dump avec mysqldump
              mysqldump -h mariadb-0.mariadb \
                -uroot -p${MYSQL_ROOT_PASSWORD} \
                --all-databases \
                --single-transaction \
                --quick \
                --lock-tables=false \
                --routines \
                --triggers \
                --events \
                | gzip > ${BACKUP_FILE}
              
              echo "✅ Backup created: ${BACKUP_FILE}"
              
              # Cleanup vieux backups (> 30 jours)
              find /backups -name "mariadb-backup-*.sql.gz" -mtime +30 -delete
            
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: root-password
            
            volumeMounts:
            - name: backups
              mountPath: /backups
          
          volumes:
          - name: backups
            persistentVolumeClaim:
              claimName: mariadb-backups
```

**PVC pour backups** :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-backups
  namespace: databases
spec:
  accessModes:
  - ReadWriteMany  # RWX si besoin accès depuis plusieurs nœuds
  storageClassName: standard
  resources:
    requests:
      storage: 500Gi
```

---

## Monitoring et observabilité

### ServiceMonitor (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mariadb
  namespace: databases
  labels:
    app: mariadb
spec:
  selector:
    matchLabels:
      app: mariadb-metrics
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**Sidecar mysqld_exporter** :

```yaml
# Ajouter dans StatefulSet template
containers:
- name: mysqld-exporter
  image: prom/mysqld-exporter:latest
  env:
  - name: DATA_SOURCE_NAME
    value: "exporter:exporterpass@(localhost:3306)/"
  ports:
  - containerPort: 9104
    name: metrics
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

---

## ✅ Points clés à retenir

- **StatefulSets sont essentiels** : Identité stable, stockage persistant, ordre déterministe
- **Headless Service** : DNS stable pour chaque pod (mariadb-0.mariadb.databases.svc)
- **PersistentVolumes** : StorageClass + volumeClaimTemplates pour stockage dynamique
- **Anti-affinity** : Ne jamais mettre 2 pods MariaDB sur même nœud
- **Init containers** : Setup configuration, permissions, réplication
- **ConfigMaps** : Configuration externalisée (my.cnf)
- **Secrets** : Mots de passe chiffrés (at-rest avec encryption)
- **Resource limits** : CPU/RAM pour éviter contention
- **Probes** : Liveness (redémarre si mort) + Readiness (retire du service si pas prêt)
- **Backup automatisé** : CronJob pour dumps réguliers
- **Monitoring** : mysqld_exporter + Prometheus + Grafana
- **Operators** : Automatisation niveau supérieur (voir 16.5)

💡 **Production tip** : Pour production, utiliser **mariadb-operator** plutôt que StatefulSet manuel pour automation complète.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [📖 Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)
- [📖 Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [📖 MariaDB on Kubernetes](https://mariadb.com/kb/en/running-mariadb-on-kubernetes/)

### Guides et best practices
- [📝 Running Databases on Kubernetes](https://www.kubernetes.io/blog/2021/05/14/running-mysql-on-kubernetes/)
- [📝 StatefulSet Best Practices](https://cloud.google.com/blog/products/databases/to-run-or-not-to-run-a-database-on-kubernetes-what-to-consider)

### Outils
- [🔧 Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
- [🔧 mysqld_exporter](https://github.com/prometheus/mysqld_exporter)

---

## ➡️ Sections suivantes

**16.4.1 StatefulSets pour MariaDB** : Nous approfondirons les StatefulSets avec des patterns avancés (rolling updates, scaling, failure scenarios), init containers complexes, et stratégies de migration.

**16.4.2 PersistentVolumes et StorageClasses** : Nous détaillerons la gestion du stockage Kubernetes, snapshots, expansion de volumes, migration de données, et performance tuning.

---

**MariaDB** : Version 11.8 LTS

⏭️ [StatefulSets pour MariaDB](/16-devops-automatisation/04.1-statefulsets.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.5 mariadb-operator pour Kubernetes

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : 
> - Section 16.4 Orchestration Kubernetes maîtrisée
> - Expérience avec StatefulSets et PersistentVolumes
> - Compréhension des Custom Resource Definitions (CRDs)
> - Familiarité avec Helm et kubectl
> - Connaissance de la réplication et clustering MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le pattern Kubernetes Operator et son intérêt pour MariaDB
- **Distinguer** mariadb-operator (communauté) et MariaDB Enterprise Operator
- **Déployer** MariaDB avec l'operator plutôt que des manifests manuels
- **Automatiser** la gestion du cycle de vie complet (installation, scaling, backup, restore)
- **Configurer** la réplication et Galera Cluster via CRDs déclaratives
- **Gérer** les backups et restores automatiquement
- **Monitorer** les ressources MariaDB via l'operator
- **Appliquer** les best practices de production avec operators

---

## Introduction

### Qu'est-ce qu'un Kubernetes Operator ?

Un **Operator** est un pattern Kubernetes qui **étend l'API Kubernetes** pour gérer des applications complexes de manière déclarative et automatisée.

**Définition officielle (CoreOS/Red Hat)** :
> "An Operator is a method of packaging, deploying, and managing a Kubernetes application. A Kubernetes application is an application that is both deployed on Kubernetes and managed using the Kubernetes APIs and kubectl tooling."

**En termes simples** :

```
┌──────────────────────────────────────────────────────────────┐
│                    Sans Operator                             │
├──────────────────────────────────────────────────────────────┤
│  Vous devez manuellement:                                    │
│  1. Écrire StatefulSet, Service, ConfigMap, Secret           │
│  2. Configurer réplication (init containers, scripts)        │
│  3. Gérer scaling (ajouter replicas, reconfigurer)           │
│  4. Planifier backups (CronJobs, scripts)                    │
│  5. Gérer failover (scripts, monitoring)                     │
│  6. Faire rolling updates (commandes kubectl)                │
│  7. Monitorer et alerter (setup Prometheus)                  │
│                                                              │
│  = Beaucoup de YAML, scripts bash, travail manuel            │
└──────────────────────────────────────────────────────────────┘

                              VS

┌──────────────────────────────────────────────────────────────┐
│                    Avec Operator                             │
├──────────────────────────────────────────────────────────────┤
│  Vous déclarez l'état désiré:                                │
│                                                              │
│  apiVersion: mariadb.mmontes.io/v1alpha1                     │
│  kind: MariaDB                                               │
│  spec:                                                       │
│    replicas: 3                                               │
│    galera:                                                   │
│      enabled: true                                           │
│    storage: 100Gi                                            │
│                                                              │
│  L'Operator gère automatiquement:                            │
│  ✅ Création StatefulSet, Services, ConfigMaps               │
│  ✅ Configuration Galera Cluster                             │
│  ✅ Scaling avec reconfiguration automatique                 │
│  ✅ Backups planifiés et on-demand                           │
│  ✅ Failover automatique                                     │
│  ✅ Rolling updates intelligents                             │
│  ✅ Monitoring intégré                                       │
│                                                              │
│  = Configuration déclarative simple, automation complète     │
└──────────────────────────────────────────────────────────────┘
```

### Comment fonctionne un Operator ?

**Architecture d'un Operator** :

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                         │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Control Plane                           │ │
│  │                                                            │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │              API Server                             │   │ │
│  │  │  ┌───────────────────────────────────────────────┐  │   │ │
│  │  │  │  Custom Resource Definitions (CRDs)           │  │   │ │
│  │  │  │  - MariaDB                                    │  │   │ │
│  │  │  │  - Backup                                     │  │   │ │
│  │  │  │  - Restore                                    │  │   │ │
│  │  │  │  - Connection                                 │  │   │ │
│  │  │  └───────────────────────────────────────────────┘  │   │ │
│  │  └─────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           ▲                                     │
│                           │ Watch CRDs                          │
│                           │                                     │
│  ┌────────────────────────┼───────────────────────────────────┐ │
│  │                        │                                   │ │
│  │  ┌─────────────────────▼─────────────────────────────────┐ │ │
│  │  │          mariadb-operator (Controller)                │ │ │
│  │  │                                                       │ │ │
│  │  │  ┌──────────────────────────────────────────────────┐ │ │ │
│  │  │  │  Reconciliation Loop                             │ │ │ │
│  │  │  │                                                  │ │ │ │
│  │  │  │  1. Watch: Observe MariaDB CRs                   │ │ │ │
│  │  │  │  2. Compare: Actual state vs Desired state       │ │ │ │
│  │  │  │  3. Reconcile: Create/Update/Delete resources    │ │ │ │
│  │  │  │  4. Status: Update CR status                     │ │ │ │
│  │  │  │  5. Repeat                                       │ │ │ │
│  │  │  └──────────────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────────────┘ │ │
│  │                            │                               │ │
│  │                            │ Creates/Manages               │ │
│  │                            ▼                               │ │
│  │  ┌───────────────────────────────────────────────────────┐ │ │
│  │  │          Kubernetes Resources                         │ │ │
│  │  │  - StatefulSet                                        │ │ │
│  │  │  - Services (headless, primary, replica)              │ │ │
│  │  │  - ConfigMaps (my.cnf, scripts)                       │ │ │
│  │  │  - Secrets (passwords)                                │ │ │
│  │  │  - PersistentVolumeClaims                             │ │ │
│  │  │  - Jobs (backups, restore)                            │ │ │
│  │  │  - ServiceMonitors (Prometheus)                       │ │ │
│  │  └───────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Reconciliation Loop (Boucle de réconciliation)** :

```python
# Pseudo-code simplifié d'un operator

while True:
    # 1. Watch events sur CRD MariaDB
    event = watch_mariadb_resources()
    
    if event.type == "ADDED":
        # Nouvelle ressource MariaDB créée
        desired_state = event.object.spec
        create_statefulset(desired_state)
        create_services(desired_state)
        create_configmaps(desired_state)
        if desired_state.galera.enabled:
            configure_galera_cluster()
        
    elif event.type == "MODIFIED":
        # Ressource MariaDB modifiée
        desired_state = event.object.spec
        actual_state = get_actual_state()
        
        if desired_state.replicas != actual_state.replicas:
            scale_statefulset(desired_state.replicas)
        
        if desired_state.storage != actual_state.storage:
            expand_pvc(desired_state.storage)
        
    elif event.type == "DELETED":
        # Ressource MariaDB supprimée
        delete_all_resources()
    
    # Mettre à jour le status
    update_status(event.object)
    
    sleep(reconciliation_interval)
```

### Pourquoi utiliser un Operator pour MariaDB ?

**Avantages vs StatefulSet manuel** :

| Aspect | StatefulSet Manuel | mariadb-operator |
|--------|-------------------|------------------|
| **Complexité** | 500+ lignes YAML + scripts bash | ~50 lignes YAML déclaratif |
| **Réplication** | Init containers complexes | `replication: enabled: true` |
| **Galera Cluster** | Setup manuel délicat | `galera: enabled: true` |
| **Backups** | CronJobs manuels | `backup: schedule: "0 2 * * *"` |
| **Scaling** | Reconfiguration manuelle | Ajuster `replicas`, operator reconfigure |
| **Failover** | Scripts custom | Automatique |
| **Monitoring** | Setup Prometheus manuel | ServiceMonitor automatique |
| **Upgrades** | Rolling update manuel | Version MariaDB dans spec |
| **Disaster recovery** | Scripts de restore | `Restore` CRD |
| **Maintenance** | Haute (expertise requise) | Faible (déclaratif) |

**Cas d'usage idéaux pour mariadb-operator** :

1. ✅ **Production multi-environnements** : Déployer identiquement dev/staging/prod
2. ✅ **Galera Cluster** : Setup complexe simplifié
3. ✅ **Backup automatisé** : S3, GCS, Azure Blob intégrés
4. ✅ **Disaster recovery** : Restore en une commande
5. ✅ **Multi-tenancy** : Plusieurs bases MariaDB gérées uniformément
6. ✅ **GitOps** : Configuration versionnée dans Git
7. ✅ **Self-service** : Developers peuvent déployer sans expertise K8s

**Quand NE PAS utiliser l'operator** :

- ❌ Environnement dev local simple (docker-compose suffit)
- ❌ Besoin de contrôle total sur chaque détail
- ❌ Infrastructure legacy incompatible avec CRDs
- ❌ Équipe sans compétences Kubernetes

---

## mariadb-operator : Vue d'ensemble

### Présentation

**mariadb-operator** (aussi appelé `mariadb-operator-enterprise` dans sa version communautaire) est un operator Kubernetes open-source pour MariaDB développé par **Martin Montes** (mmontes).

🔗 **Repository GitHub** : [github.com/mariadb-operator/mariadb-operator](https://github.com/mariadb-operator/mariadb-operator)

**Caractéristiques principales** :

- ✅ **Open source** (Apache License 2.0)
- ✅ **Production-ready** (utilisé en production par plusieurs entreprises)
- ✅ **Actif** : Développement continu, releases régulières
- ✅ **Communauté** : Support via GitHub issues, Slack
- ✅ **Documentation complète** : Exemples, guides, API reference

**Versions** :

| Version Operator | MariaDB Supportée | Kubernetes | Status |
|------------------|-------------------|------------|--------|
| v0.0.28 (latest) | 11.8, 11.4, 10.11 | 1.25+ | ✅ Stable |
| v0.0.27 | 11.4, 10.11, 10.6 | 1.24+ | ✅ Stable |
| v0.0.26 | 11.4, 10.11, 10.6 | 1.23+ | Deprecated |

🆕 **Nouveautés version récente** (v0.0.28+) :
- Support MariaDB 11.8 LTS
- Amélioration Galera Cluster (3+ nœuds stables)
- Backup S3 avec chiffrement
- Connection pooling avec ProxySQL
- Init jobs pour setup custom

### Custom Resource Definitions (CRDs)

L'operator introduit plusieurs **CRDs** pour gérer MariaDB de manière déclarative :

```
┌──────────────────────────────────────────────────────────────┐
│                  mariadb-operator CRDs                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. MariaDB                                                  │
│     └─> Définit une instance ou cluster MariaDB              │
│                                                              │
│  2. Backup                                                   │
│     └─> Définit une sauvegarde (one-time ou schedule)        │
│                                                              │
│  3. Restore                                                  │
│     └─> Restaure depuis un backup                            │
│                                                              │
│  4. Connection                                               │
│     └─> Crée utilisateur + secret de connexion               │
│                                                              │
│  5. Grant                                                    │
│     └─> Gère les permissions utilisateurs                    │
│                                                              │
│  6. Database                                                 │
│     └─> Crée une base de données                             │
│                                                              │
│  7. User                                                     │
│     └─> Crée un utilisateur MariaDB                          │
│                                                              │
│  8. SqlJob                                                   │
│     └─> Exécute scripts SQL (migrations, setup)              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Exemple simple de chaque CRD** :

```yaml
# 1. MariaDB - Instance standalone
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: mariadb
spec:
  rootPasswordSecretKeyRef:
    name: mariadb-root
    key: password
  
  storage:
    size: 100Gi
    storageClassName: fast-ssd
  
  myCnf: |
    [mysqld]
    max_connections=500
```

```yaml
# 2. Backup - Sauvegarde planifiée
apiVersion: mariadb.mmontes.io/v1alpha1
kind: Backup
metadata:
  name: mariadb-backup
spec:
  mariaDbRef:
    name: mariadb
  
  schedule:
    cron: "0 2 * * *"
  
  storage:
    s3:
      bucket: my-mariadb-backups
      endpoint: s3.amazonaws.com
      region: eu-west-1
```

```yaml
# 3. Restore - Restauration
apiVersion: mariadb.mmontes.io/v1alpha1
kind: Restore
metadata:
  name: mariadb-restore
spec:
  mariaDbRef:
    name: mariadb
  
  backupRef:
    name: mariadb-backup-20251214
```

```yaml
# 4. Connection - Utilisateur applicatif
apiVersion: mariadb.mmontes.io/v1alpha1
kind: Connection
metadata:
  name: app-connection
spec:
  mariaDbRef:
    name: mariadb
  
  username: appuser
  passwordSecretKeyRef:
    name: app-password
    key: password
  
  database: myapp
  
  # Crée automatiquement un Secret avec connection string
```

```yaml
# 5. Database - Créer une base
apiVersion: mariadb.mmontes.io/v1alpha1
kind: Database
metadata:
  name: myapp
spec:
  mariaDbRef:
    name: mariadb
  
  characterSet: utf8mb4
  collate: utf8mb4_unicode_ci
```

### Architecture déployée par l'operator

Quand vous créez une ressource `MariaDB`, l'operator **crée automatiquement** :

```
┌──────────────────────────────────────────────────────────────┐
│   kubectl apply -f mariadb.yaml (CR)                         │
│                                                              │
│   apiVersion: mariadb.mmontes.io/v1alpha1                    │
│   kind: MariaDB                                              │
│   metadata:                                                  │
│     name: mariadb                                            │
│   spec:                                                      │
│     replicas: 3                                              │
│     storage: 100Gi                                           │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   mariadb-operator observe et crée:   │
        └───────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ StatefulSet  │  │   Services   │  │  ConfigMaps  │
│              │  │              │  │              │
│ - mariadb-0  │  │ - mariadb    │  │ - my.cnf     │
│ - mariadb-1  │  │   (headless) │  │ - init       │
│ - mariadb-2  │  │ - mariadb-   │  │   scripts    │
│              │  │   primary    │  │              │
│              │  │ - mariadb-   │  │              │
│              │  │   secondary  │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
        │                   │                   │
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    PVCs      │  │   Secrets    │  │ Service      │
│              │  │              │  │  Monitors    │
│ - data-      │  │ - root-      │  │              │
│   mariadb-0  │  │   password   │  │ (Prometheus) │
│ - data-      │  │ - replica-   │  │              │
│   mariadb-1  │  │   password   │  │              │
│ - data-      │  │              │  │              │
│   mariadb-2  │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Ressources Kubernetes créées automatiquement** :

1. **StatefulSet** : Pods MariaDB avec noms stables
2. **Services** :
   - Headless service (DNS stable)
   - Primary service (R/W)
   - Secondary service (R/O) si réplication
3. **ConfigMaps** :
   - Configuration my.cnf
   - Scripts d'initialisation
   - Scripts de réplication/Galera
4. **Secrets** :
   - Root password
   - Replication user password
   - Application users passwords
5. **PersistentVolumeClaims** : Un par pod
6. **ServiceMonitor** : Monitoring Prometheus (si operator installé)
7. **Jobs** : Init jobs, backup jobs

---

## Comparaison : mariadb-operator vs MariaDB Enterprise Operator

Il existe **deux operators** pour MariaDB :

### 1. mariadb-operator (Communauté)

**Développeur** : Martin Montes (mmontes)  
**License** : Apache 2.0 (Open Source)  
**Repository** : github.com/mariadb-operator/mariadb-operator

**Caractéristiques** :

| Aspect | Support |
|--------|---------|
| Prix | ✅ Gratuit |
| Source | ✅ Open source |
| Réplication | ✅ Primary-Replica |
| Galera Cluster | ✅ Oui (3+ nœuds) |
| Backup S3 | ✅ Oui |
| Backup GCS | ✅ Oui |
| Restore | ✅ Oui |
| ProxySQL | ✅ Intégré |
| MaxScale | ❌ Non |
| Support commercial | ❌ Non (communauté GitHub) |
| Monitoring | ✅ Prometheus |
| Multi-master | ✅ Via Galera |

**Idéal pour** :
- ✅ Startups et PME
- ✅ Projets open source
- ✅ Environnements dev/staging
- ✅ Production sans SLA critique
- ✅ Équipes avec expertise Kubernetes

### 2. MariaDB Enterprise Operator 🆕

**Développeur** : MariaDB Corporation (officiel)  
**License** : Propriétaire (Enterprise)  
**Documentation** : mariadb.com/docs/server/enterprise-operator/

**Caractéristiques** :

| Aspect | Support |
|--------|---------|
| Prix | 💰 Payant (subscription) |
| Source | ⚠️ Closed source |
| Réplication | ✅ Primary-Replica avancée |
| Galera Cluster | ✅ Oui (optimisé) |
| Backup S3/GCS/Azure | ✅ Oui (tous clouds) |
| Restore | ✅ Oui (point-in-time) |
| ProxySQL | ❌ Non |
| MaxScale | ✅ Intégré natif |
| Support commercial | ✅ 24/7 SLA |
| Monitoring | ✅ Prometheus + custom |
| Multi-master | ✅ Via Galera + MaxScale |
| ColumnStore | ✅ Support analytique |
| Xpand | ✅ Distributed SQL |

🆕 **Nouveautés Enterprise Operator (2025)** :
- Support MariaDB 11.8 LTS
- MaxScale 25.01 intégré
- Backup point-in-time recovery
- Automated failover avancé
- Multi-cloud (AWS, GCP, Azure, on-premise)

**Idéal pour** :
- ✅ Grandes entreprises
- ✅ Production critique (SLA strict)
- ✅ Conformité réglementaire
- ✅ Support 24/7 requis
- ✅ MaxScale/ColumnStore/Xpand nécessaire

### Tableau comparatif complet

| Fonctionnalité | Community Operator | Enterprise Operator |
|----------------|-------------------|---------------------|
| **Prix** | Gratuit | Subscription |
| **License** | Apache 2.0 | Propriétaire |
| **Support** | GitHub issues | 24/7 commercial |
| **MariaDB versions** | 10.6, 10.11, 11.4, 11.8 | 10.6, 10.11, 11.4, 11.8 + Enterprise |
| **Réplication** | ✅ Primary-Replica | ✅ Primary-Replica avancé |
| **Galera Cluster** | ✅ 3+ nœuds | ✅ 3+ nœuds optimisé |
| **Backup** | S3, GCS, PVC | S3, GCS, Azure, PVC + PITR |
| **Restore** | ✅ Full restore | ✅ Full + Point-in-time |
| **MaxScale** | ❌ Non | ✅ Oui (intégré) |
| **ProxySQL** | ✅ Oui | ❌ Non (MaxScale à la place) |
| **Monitoring** | Prometheus | Prometheus + custom metrics |
| **Auto-scaling** | ⚠️ Manuel | ✅ Automatique |
| **Multi-cloud** | ✅ Oui | ✅ Oui + hybrid |
| **ColumnStore** | ❌ Non | ✅ Oui |
| **Xpand** | ❌ Non | ✅ Oui |
| **Maturité** | Production-ready | Enterprise-grade |
| **Documentation** | GitHub + exemples | Complète + training |

💡 **Recommandation** :

```
┌──────────────────────────────────────────────────────────┐
│  SI vous êtes...              ALORS utilisez...          │
├──────────────────────────────────────────────────────────┤
│  Startup, PME                 Community Operator         │
│  Projet open source           Community Operator         │
│  Dev/Staging                  Community Operator         │
│  Production non-critique      Community Operator         │
│  Grande entreprise            Enterprise Operator        │
│  Production critique (SLA)    Enterprise Operator        │
│  Besoin MaxScale              Enterprise Operator        │
│  Besoin ColumnStore/Xpand     Enterprise Operator        │
│  Support 24/7 requis          Enterprise Operator        │
└──────────────────────────────────────────────────────────┘
```

**Pour la suite de cette formation**, nous nous concentrerons sur **mariadb-operator (Community)** car :
- ✅ Gratuit et open source
- ✅ Accessible à tous
- ✅ Production-ready pour la majorité des cas d'usage
- ✅ Bien documenté avec exemples

---

## Concepts avancés de l'operator

### 1. Reconciliation et idempotence

L'operator fonctionne en **boucle de réconciliation** :

```
┌─────────────────────────────────────────────────────────────┐
│              Reconciliation Loop                            │
│                                                             │
│  1. User modifie MariaDB CR                                 │
│     spec:                                                   │
│       replicas: 3 → 5                                       │
│                                                             │
│  2. Operator détecte changement (Watch event)               │
│                                                             │
│  3. Operator compare:                                       │
│     - Desired state: 5 replicas                             │
│     - Actual state: 3 replicas (StatefulSet)                │
│                                                             │
│  4. Operator réconcilie:                                    │
│     - Scale StatefulSet de 3 → 5                            │
│     - Attend que pods soient Ready                          │
│     - Configure réplication sur nouveaux pods               │
│     - Met à jour Services                                   │
│                                                             │
│  5. Operator met à jour status:                             │
│     status:                                                 │
│       conditions:                                           │
│       - type: Ready                                         │
│         status: "True"                                      │
│       currentReplicas: 5                                    │
│                                                             │
│  6. Boucle recommence (watch next event)                    │
└─────────────────────────────────────────────────────────────┘
```

**Idempotence** : Appliquer la même configuration plusieurs fois donne le même résultat.

```bash
# Première application
kubectl apply -f mariadb.yaml
# Crée StatefulSet, Services, etc.

# Deuxième application (sans changement)
kubectl apply -f mariadb.yaml
# Aucun changement, state déjà désiré

# Troisième application (avec changement)
vim mariadb.yaml  # replicas: 3 → 5
kubectl apply -f mariadb.yaml
# Scale StatefulSet de 3 → 5
```

### 2. Finalizers et cleanup

Quand vous supprimez une ressource `MariaDB`, l'operator **nettoie proprement** :

```yaml
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: mariadb
  finalizers:
  - mariadb.mmontes.io/finalizer  # Empêche suppression immédiate
spec:
  # ...
```

**Workflow de suppression** :

```
1. kubectl delete mariadb mariadb
   ↓
2. Kubernetes marque CR comme "deletionTimestamp" set
   ↓
3. Operator détecte suppression (finalizer présent)
   ↓
4. Operator exécute cleanup:
   - Backup final (si configuré)
   - Supprime StatefulSet (graceful shutdown)
   - Supprime Services
   - Supprime ConfigMaps/Secrets
   - Option: Conserver ou supprimer PVCs
   ↓
5. Operator retire finalizer
   ↓
6. Kubernetes supprime CR
```

**Configuration du cleanup** :

```yaml
spec:
  storage:
    size: 100Gi
    # Politique de conservation des PVC
    persistentVolumeClaimRetentionPolicy:
      whenDeleted: Retain  # Ou Delete
      whenScaled: Retain   # Lors de scale down
```

### 3. Status et conditions

L'operator maintient le **status** de chaque ressource :

```yaml
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: mariadb
spec:
  replicas: 3
status:
  # État général
  phase: Running  # Pending, Running, Failed
  
  # Conditions (suivre évolution)
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2025-12-14T10:30:00Z"
    reason: AllReplicasReady
    message: "All 3 replicas are ready"
  
  - type: StorageReady
    status: "True"
    lastTransitionTime: "2025-12-14T10:25:00Z"
    reason: PVCsBound
  
  - type: Replication
    status: "True"
    lastTransitionTime: "2025-12-14T10:28:00Z"
    reason: ReplicationConfigured
  
  # Réplicas actuelles
  currentReplicas: 3
  readyReplicas: 3
  
  # Primary actuel
  currentPrimary: mariadb-0
```

**Vérifier le status** :

```bash
# Status complet
kubectl get mariadb mariadb -o yaml

# Status simplifié
kubectl get mariadb mariadb

# Conditions
kubectl get mariadb mariadb -o jsonpath='{.status.conditions[*].type}'

# Événements
kubectl describe mariadb mariadb
```

### 4. Webhooks de validation

L'operator utilise des **admission webhooks** pour valider les CRs :

```
User apply CR → Kubernetes API → Validation Webhook → Accept/Reject
```

**Exemple de validation** :

```yaml
# ❌ INVALIDE - rejeté par webhook
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
spec:
  replicas: 2  # Galera nécessite 3+ nœuds
  galera:
    enabled: true

# Error: admission webhook "mariadb.kb.io" denied the request: 
# Galera cluster requires at least 3 replicas
```

```yaml
# ✅ VALIDE
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
spec:
  replicas: 3
  galera:
    enabled: true
```

---

## Cas d'usage typiques

### 1. Application multi-tenant

**Scénario** : SaaS avec une base MariaDB par client

```yaml
# Tenant A
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: tenant-a-db
  namespace: tenant-a
spec:
  storage:
    size: 50Gi
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"

---
# Tenant B
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: tenant-b-db
  namespace: tenant-b
spec:
  storage:
    size: 100Gi
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
```

**Automatisation** :

```bash
#!/bin/bash
# create-tenant.sh

TENANT_NAME=$1

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-${TENANT_NAME}
---
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: ${TENANT_NAME}-db
  namespace: tenant-${TENANT_NAME}
spec:
  storage:
    size: 50Gi
  rootPasswordSecretKeyRef:
    name: ${TENANT_NAME}-root-password
    key: password
EOF
```

### 2. Dev/Staging/Production GitOps

**Structure Git** :

```
mariadb-gitops/
├── base/
│   └── mariadb.yaml          # Configuration commune
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
```

**base/mariadb.yaml** :

```yaml
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
metadata:
  name: mariadb
spec:
  image: mariadb:11.8
  
  rootPasswordSecretKeyRef:
    name: mariadb-root
    key: password
  
  myCnf: |
    [mysqld]
    character-set-server=utf8mb4
```

**overlays/production/kustomization.yaml** :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base

patches:
- patch: |-
    apiVersion: mariadb.mmontes.io/v1alpha1
    kind: MariaDB
    metadata:
      name: mariadb
    spec:
      replicas: 3
      storage:
        size: 500Gi
      resources:
        requests:
          cpu: "4"
          memory: "16Gi"
      galera:
        enabled: true
```

**Déploiement avec ArgoCD** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mariadb-production
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/mariadb-gitops
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3. Disaster Recovery

**Backup planifié** :

```yaml
apiVersion: mariadb.mmontes.io/v1alpha1
kind: Backup
metadata:
  name: mariadb-daily-backup
spec:
  mariaDbRef:
    name: mariadb
  
  # Planification
  schedule:
    cron: "0 2 * * *"
    suspend: false
  
  # Rétention
  maxRetention: 30d
  
  # Stockage S3
  storage:
    s3:
      bucket: mariadb-backups-prod
      endpoint: s3.amazonaws.com
      region: eu-west-1
      prefix: mariadb/daily
      
      # Credentials
      accessKeyIdSecretKeyRef:
        name: s3-credentials
        key: access-key-id
      secretAccessKeySecretKeyRef:
        name: s3-credentials
        key: secret-access-key
```

**Restore en cas de sinistre** :

```yaml
apiVersion: mariadb.mmontes.io/v1alpha1
kind: Restore
metadata:
  name: disaster-recovery-restore
spec:
  mariaDbRef:
    name: mariadb
  
  # Backup à restaurer
  backupRef:
    name: mariadb-daily-backup-20251214
  
  # Point-in-time (optionnel)
  targetRecoveryTime: "2025-12-14T09:30:00Z"
```

---

## Best practices avec mariadb-operator

### 1. Sizing et resources

**Définir requests et limits** :

```yaml
spec:
  resources:
    requests:
      cpu: "2"       # Garanti
      memory: "8Gi"  # Garanti
    limits:
      cpu: "4"       # Maximum
      memory: "16Gi" # Maximum
  
  # Buffer pool = 75% de memory request
  myCnf: |
    [mysqld]
    innodb_buffer_pool_size = 6G  # 75% de 8Gi
```

### 2. Anti-affinity

**Ne jamais mettre 2 replicas sur même nœud** :

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/instance
            operator: In
            values:
            - mariadb
        topologyKey: kubernetes.io/hostname
```

### 3. Update strategy

**Rolling updates intelligents** :

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    
  # Attendre que replica soit synchro avant next
  podManagementPolicy: OrderedReady
```

### 4. Monitoring

**Activer ServiceMonitor** :

```yaml
spec:
  metrics:
    enabled: true
    
    # Prometheus Operator
    serviceMonitor:
      enabled: true
      interval: 30s
      
    # mysqld_exporter
    exporter:
      image: prom/mysqld-exporter:latest
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
```

### 5. Secrets management

**Utiliser external secrets** :

```yaml
# External Secret (ESO)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mariadb-root
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: mariadb-root-password
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: secret/mariadb/root-password

---
# MariaDB utilise le secret
apiVersion: mariadb.mmontes.io/v1alpha1
kind: MariaDB
spec:
  rootPasswordSecretKeyRef:
    name: mariadb-root-password
    key: password
```

---

## ✅ Points clés à retenir

- **Operator = automation** : Gère cycle de vie complet MariaDB automatiquement
- **CRDs déclaratives** : Configuration simple en YAML (MariaDB, Backup, Restore, etc.)
- **Reconciliation loop** : Operator maintient état désiré en permanence
- **Community vs Enterprise** : Community gratuit et suffisant pour majorité des cas
- **Production-ready** : Galera, réplication, backups, monitoring intégrés
- **GitOps friendly** : Configuration versionnée, ArgoCD/FluxCD compatible
- **Best practices** : Resources, anti-affinity, monitoring, secrets externes
- **Simplification majeure** : ~50 lignes YAML vs 500+ lignes manifests manuels
- **Self-healing** : Redémarre pods, reconfigure réplication automatiquement
- **Multi-tenant** : Facilite gestion de multiples instances MariaDB

💡 **Recommandation** : Pour production Kubernetes, utiliser mariadb-operator plutôt que StatefulSets manuels.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 mariadb-operator GitHub](https://github.com/mariadb-operator/mariadb-operator)
- [📖 mariadb-operator Documentation](https://github.com/mariadb-operator/mariadb-operator/tree/main/docs)
- [📖 MariaDB Enterprise Operator](https://mariadb.com/docs/server/enterprise-operator/)

### Guides et exemples
- [📝 mariadb-operator Examples](https://github.com/mariadb-operator/mariadb-operator/tree/main/examples)
- [📝 Kubernetes Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [📝 Operator Framework](https://operatorframework.io/)

### Communauté
- [💬 mariadb-operator Discussions](https://github.com/mariadb-operator/mariadb-operator/discussions)
- [🐛 Issues](https://github.com/mariadb-operator/mariadb-operator/issues)

---

## ➡️ Sections suivantes

**16.5.1 Installation et CRDs** : Nous détaillerons l'installation de mariadb-operator via Helm, la configuration des CRDs, et le déploiement de votre premier cluster MariaDB avec l'operator.

**16.5.2 Déploiement Galera avec operator** : Nous approfondirons le déploiement de Galera Cluster multi-master avec l'operator, le bootstrap, la configuration avancée, et le troubleshooting.

**16.5.3 Réplication avec operator** : Nous configurerons la réplication Primary-Replica avec GTID, le failover automatique, et le monitoring du lag.

**16.5.4 Backups automatisés** : Nous mettrons en place des backups planifiés vers S3/GCS, des restores point-in-time, et des stratégies de disaster recovery.

---

**MariaDB** : Version 11.8 LTS
**mariadb-operator** : v0.0.28+

⏭️ [Installation et CRDs](/16-devops-automatisation/05.1-installation-crds.md)

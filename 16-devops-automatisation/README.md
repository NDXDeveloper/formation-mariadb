🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16. DevOps et Automatisation

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 12-15 heures  
> **Prérequis** : 
> - Connaissance approfondie de MariaDB (chapitres 1-15)
> - Expérience avec Linux/Unix et scripting bash
> - Familiarité avec Docker et concepts de conteneurisation
> - Compréhension basique de Kubernetes (pods, services, deployments)
> - Notions de CI/CD et versioning (Git)

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Automatiser** le déploiement de MariaDB avec Ansible et Terraform
- **Conteneuriser** MariaDB avec Docker en production avec haute disponibilité
- **Orchestrer** des clusters MariaDB sur Kubernetes avec StatefulSets et operators
- **Implémenter** des pipelines CI/CD pour gérer les migrations de schéma
- **Monitorer** MariaDB avec Prometheus/Grafana et mettre en place l'observabilité complète
- **Appliquer** les principes GitOps pour la gestion déclarative des bases de données
- **Concevoir** une architecture DevOps complète pour MariaDB en production

---

## Introduction

### Le DevOps appliqué aux bases de données : un défi unique

Dans le monde DevOps moderne, les bases de données représentent un défi particulier. Contrairement aux applications stateless qui peuvent être déployées et détruites à volonté, **les bases de données sont stateful par nature** : elles contiennent des données précieuses, souvent critiques, qui doivent être préservées, migrées et protégées avec le plus grand soin.

L'adoption des pratiques DevOps pour MariaDB nécessite donc une approche spécifique qui conjugue :

- **Automatisation** : Infrastructure as Code, déploiements reproductibles
- **Versioning** : Gestion des changements de schéma, migrations contrôlées
- **Observabilité** : Monitoring proactif, alerting, tracing
- **Résilience** : Haute disponibilité, disaster recovery, backups automatisés
- **Sécurité** : Secrets management, chiffrement, conformité

### L'évolution du paysage DevOps pour bases de données

**Traditionnel (avant 2015)** :
- Déploiements manuels ou scripts shell artisanaux
- Configuration serveurs "pets" (serveurs uniques, configurés à la main)
- Backups cron basiques
- Monitoring ad-hoc
- Migrations de schéma lancées manuellement en production

**Moderne (2025)** :
- Infrastructure as Code (Terraform, Ansible)
- Conteneurisation et orchestration (Docker, Kubernetes)
- Operators Kubernetes natifs pour MariaDB
- GitOps : configuration déclarative versionnée
- CI/CD avec tests automatisés de migrations
- Observabilité complète (métriques + logs + traces)
- Disaster recovery automatisé

### MariaDB dans l'écosystème cloud-native

MariaDB 11.8 LTS s'intègre parfaitement dans les architectures cloud-native modernes :

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│  (Microservices, Serverless, API Gateway)                   │
└─────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Data Plane (MariaDB)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Primary     │  │  Replica 1   │  │  Replica 2   │       │
│  │  (R/W)       │◄─┤  (Read)      │◄─┤  (Read)      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         ▲                                                   │
│         │         ┌─────────────────┐                       │
│         └─────────┤  MaxScale/      │                       │
│                   │  ProxySQL       │                       │
│                   └─────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Infrastructure & Orchestration                 │
│  ┌────────────┐ ┌─────────────┐ ┌──────────────┐            │
│  │ Kubernetes │ │  Terraform  │ │   Ansible    │            │
│  └────────────┘ └─────────────┘ └──────────────┘            │
└─────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                 Observability Stack                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Prometheus   │ │   Grafana    │ │  ELK/Loki    │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Nouveautés MariaDB 11.8 pertinentes pour le DevOps

🆕 **Améliorations pour l'automatisation** :
- **TLS par défaut** : Sécurité renforcée pour les déploiements automatisés
- **MariaDB Enterprise Operator** : Operator Kubernetes officiel nouvelle génération
- **Optimistic ALTER TABLE** : Réduction du lag de réplication lors des migrations
- **Support BACKUP STAGE** dans Mariabackup : Backups plus cohérents
- **Contrôle espace temporaire** : `max_tmp_space_usage` pour éviter saturation disque

### Structure de ce chapitre

Ce chapitre vous guidera à travers l'ensemble de l'écosystème DevOps pour MariaDB :

1. **Infrastructure as Code** (16.1) : Principes et approche déclarative
2. **Ansible & Terraform** (16.2) : Déploiement automatisé sur VMs et cloud
3. **Conteneurisation Docker** (16.3) : Images, volumes, Docker Compose
4. **Orchestration Kubernetes** (16.4) : StatefulSets et stockage persistant
5. **MariaDB Operator** (16.5) : Automation native Kubernetes
6. **Enterprise Operator** (16.6) 🆕 : Solution officielle production-grade
7. **CI/CD pour bases** (16.7) : Pipelines et testing
8. **Gestion migrations** (16.8) : Flyway, Liquibase, online schema change
9. **Monitoring Prometheus/Grafana** (16.9) : Métriques et dashboards
10. **Observabilité** (16.10) : Logs, metrics, traces
11. **Alerting** (16.11) : Détection proactive et response
12. **GitOps** (16.12) : Configuration déclarative versionnée

---

## Principes fondamentaux du DevOps pour bases de données

### 1. Infrastructure as Code (IaC)

**Principe** : Toute infrastructure doit être définie dans du code versionné, reproductible et testable.

**Pour MariaDB, cela signifie** :
- Configuration serveur dans Ansible/Terraform
- Schéma de base dans des migrations versionnées
- Paramètres MariaDB dans des fichiers de configuration versionnés
- Backups et restore procédures automatisés

**Exemple de workflow IaC** :

```yaml
# Fichier: mariadb-infrastructure.yml (conceptuel)
infrastructure:
  provider: aws
  region: eu-west-1
  
  database:
    engine: mariadb
    version: "11.8"
    instance_type: db.r6g.2xlarge
    storage: 500GB
    
  high_availability:
    multi_az: true
    replicas: 2
    backup_retention: 30
    
  security:
    encryption_at_rest: true
    tls_enforced: true
    network: private_subnet
```

### 2. Immutabilité et cattle vs pets

**Pets (ancien modèle)** :
- Serveurs nommés (db-prod-01, db-prod-02)
- Configuration manuelle et divergente
- Difficile à reproduire
- Peur de redémarrer/remplacer

**Cattle (modèle moderne)** :
- Instances interchangeables
- Configuration identique via automation
- Facile à remplacer
- Scale horizontal

**Pour MariaDB** : Approche hybride
- Les **données** restent précieuses (stateful)
- L'**infrastructure** doit être cattle (remplaçable)
- Solution : Séparation compute/storage, operators Kubernetes

### 3. Déclaratif vs impératif

**Impératif** : "Exécute ces commandes dans cet ordre"
```bash
# Impératif - fragile
apt-get update
apt-get install mariadb-server
systemctl start mariadb
mysql -e "CREATE DATABASE myapp"
```

**Déclaratif** : "Voici l'état désiré, atteins-le"
```yaml
# Déclaratif - robuste
- name: Ensure MariaDB is installed and running
  package:
    name: mariadb-server
    state: present
  
- name: Ensure MariaDB is started
  service:
    name: mariadb
    state: started
    
- name: Ensure database exists
  mysql_db:
    name: myapp
    state: present
```

💡 **Avantage** : Le code déclaratif est **idempotent** - on peut l'exécuter plusieurs fois sans effet de bord.

### 4. Versioning de schéma

Tout comme le code applicatif, le **schéma de base de données doit être versionné**.

**Principe** :
- Chaque changement de schéma = migration versionnée
- Migrations forward-only (pas de rollback dans le schéma)
- Ordre d'exécution strict (V1 → V2 → V3...)
- Traçabilité complète

**Exemple avec Flyway** :
```
migrations/
├── V1__initial_schema.sql
├── V2__add_users_table.sql
├── V3__add_email_index.sql
└── V4__add_created_at_column.sql
```

### 5. Observabilité : Trois piliers

**Metrics** (Prometheus) :
- Nombre de connexions
- Requêtes par seconde
- Utilisation buffer pool
- Lag de réplication

**Logs** (ELK/Loki) :
- Slow query log structuré
- Error log centralisé
- Audit log des accès

**Traces** (OpenTelemetry) :
- Temps d'exécution requêtes
- Chemins d'exécution distribués
- Identification bottlenecks

### 6. Sécurité dès la conception (DevSecOps)

🔒 **Principes de sécurité automatisés** :

- **Secrets management** : Vault, Kubernetes Secrets, AWS Secrets Manager
- **Least privilege** : Utilisateurs avec privilèges minimaux
- **Network isolation** : Private subnets, security groups
- **Encryption** : At-rest et in-transit par défaut
- **Audit** : Logging de tous les accès sensibles
- **Scanning** : Vulnérabilités dans les images Docker

**Exemple Kubernetes Secret** :
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-root-password
type: Opaque
data:
  password: <base64-encoded-password>
```

💡 **Conseil** : Ne JAMAIS committer de secrets en clair dans Git. Utiliser des outils comme `sealed-secrets` ou `SOPS`.

---

## Vue d'ensemble des technologies couvertes

### Infrastructure as Code

| Outil | Usage principal | Forces | Faiblesses |
|-------|----------------|--------|------------|
| **Ansible** | Configuration management, provisioning | Agentless, YAML simple, large écosystème | Peut devenir lent sur grands parcs |
| **Terraform** | Infrastructure provisioning (cloud) | Multi-cloud, plan/apply, state management | Courbe apprentissage, langage HCL |
| **Pulumi** | Infrastructure as real code (Python, TS) | Code véritable, fortement typé | Moins mature, communauté plus petite |

### Conteneurisation

**Docker** :
- Image officielle MariaDB : `mariadb:11.8`
- Multi-stage builds pour images optimisées
- Volumes pour persistance
- Networks pour isolation

**Podman** :
- Alternative Docker sans daemon
- Compatible avec images Docker
- Meilleure sécurité (rootless)

### Orchestration

**Kubernetes** :
- StatefulSets pour MariaDB
- PersistentVolumes/Claims pour stockage
- Services pour loadbalancing
- ConfigMaps/Secrets pour configuration

**Operators** :
- `mariadb-operator` (communauté)
- `mariadb-operator` (Enterprise) 🆕
- Automation niveau supérieur (Galera, backup, restore)

### CI/CD

**Pipelines** :
- GitHub Actions
- GitLab CI
- Jenkins
- ArgoCD (GitOps)

**Database migrations** :
- Flyway (Java)
- Liquibase (XML/YAML/JSON)
- gh-ost (GitHub online schema change)
- pt-online-schema-change (Percona)

### Monitoring & Observabilité

**Metrics** :
- Prometheus (time-series DB)
- mysqld_exporter (MariaDB metrics)
- Grafana (visualization)

**Logging** :
- Elasticsearch + Logstash + Kibana (ELK)
- Grafana Loki
- Fluentd/Fluent Bit (collectors)

**Tracing** :
- OpenTelemetry
- Jaeger
- Zipkin

---

## Architecture DevOps de référence pour MariaDB

Voici une architecture complète intégrant tous les composants DevOps :

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitOps Repository                       │
│  (Infrastructure code, configs, migrations versionnées)         │
└─────────────────────────────────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
        ┌──────────────────┐      ┌──────────────────┐
        │   Terraform      │      │     Ansible      │
        │   (Provision)    │      │   (Configure)    │
        └──────────────────┘      └──────────────────┘
                    │                         │
                    └────────────┬────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │   Kubernetes Cluster    │
                    │  ┌──────────────────┐   │
                    │  │ MariaDB Operator │   │
                    │  └──────────────────┘   │
                    │           │             │
                    │  ┌────────▼─────────┐   │
                    │  │   StatefulSet    │   │
                    │  │  ┌───┐ ┌───┐     │   │
                    │  │  │DB1│ │DB2│ ... │   │
                    │  │  └───┘ └───┘     │   │
                    │  └──────────────────┘   │
                    │           │             │
                    │  ┌────────▼─────────┐   │
                    │  │PersistentVolumes │   │
                    │  └──────────────────┘   │
                    └─────────────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
          ┌──────────────┐ ┌──────────┐ ┌──────────┐
          │  Prometheus  │ │   Loki   │ │  Jaeger  │
          │  (Metrics)   │ │  (Logs)  │ │ (Traces) │
          └──────────────┘ └──────────┘ └──────────┘
                    │            │            │
                    └────────────┼────────────┘
                                 ▼
                         ┌──────────────┐
                         │   Grafana    │
                         │ (Dashboards) │
                         └──────────────┘
                                 │
                                 ▼
                         ┌──────────────┐
                         │  Alertmanager│
                         │  PagerDuty   │
                         └──────────────┘
```

### Workflow type de déploiement

1. **Développement** :
   ```
   Developer → Git commit → Pull Request
   ```

2. **CI Pipeline** :
   ```
   GitHub Actions → Run tests → Run migrations (test DB) → Build Docker image
   ```

3. **CD Pipeline** :
   ```
   ArgoCD detect change → Apply to Kubernetes → Rolling update StatefulSet
   ```

4. **Monitoring** :
   ```
   mysqld_exporter → Prometheus → Grafana → Alert if threshold
   ```

5. **Incident Response** :
   ```
   Alert → PagerDuty → On-call engineer → Check Grafana → Fix → Git commit
   ```

---

## Exemple concret : Déploiement MariaDB complet

Voici un aperçu d'un déploiement DevOps complet (détaillé dans les sous-sections) :

### 1. Infrastructure (Terraform)

```hcl
# terraform/mariadb.tf
resource "aws_db_instance" "mariadb" {
  identifier = "myapp-mariadb-${var.environment}"
  
  engine         = "mariadb"
  engine_version = "11.8"
  instance_class = "db.r6g.xlarge"
  
  allocated_storage     = 500
  max_allocated_storage = 1000
  storage_encrypted     = true
  
  db_name  = "myapp"
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  
  vpc_security_group_ids = [aws_security_group.mariadb.id]
  db_subnet_group_name   = aws_db_subnet_group.mariadb.name
  
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  enabled_cloudwatch_logs_exports = ["error", "slowquery"]
  
  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### 2. Configuration (Ansible)

```yaml
# ansible/mariadb-config.yml
- name: Configure MariaDB
  hosts: mariadb_servers
  become: yes
  
  vars:
    mariadb_version: "11.8"
    innodb_buffer_pool_size: "24G"
    max_connections: 500
    
  tasks:
    - name: Install MariaDB repository
      apt_repository:
        repo: "deb [arch=amd64] https://downloads.mariadb.com/MariaDB/mariadb-{{ mariadb_version }}/repo/ubuntu {{ ansible_distribution_release }} main"
        
    - name: Install MariaDB server
      apt:
        name: mariadb-server
        state: present
        update_cache: yes
        
    - name: Configure my.cnf
      template:
        src: my.cnf.j2
        dest: /etc/mysql/my.cnf
      notify: restart mariadb
      
    - name: Ensure MariaDB is running
      service:
        name: mariadb
        state: started
        enabled: yes
```

### 3. Kubernetes (StatefulSet)

```yaml
# k8s/mariadb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  serviceName: mariadb
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
        image: mariadb:11.8
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi
```

### 4. Migration (Flyway)

```sql
-- migrations/V1__initial_schema.sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE sessions (
    id VARCHAR(128) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 5. Monitoring (Prometheus)

```yaml
# prometheus/mariadb-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mariadb
spec:
  selector:
    matchLabels:
      app: mariadb
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### 6. GitOps (ArgoCD Application)

```yaml
# argocd/mariadb-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mariadb
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/mariadb-infrastructure
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: databases
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## Best practices DevOps pour MariaDB

### 1. Séparation des environnements

Maintenir des environnements strictement séparés :

```
Development → Staging → Production
```

**Caractéristiques par environnement** :

| Aspect | Dev | Staging | Production |
|--------|-----|---------|------------|
| Données | Fake/anonymisées | Clone production récent | Réelles |
| Ressources | Minimales | ~Production | Optimales |
| HA | Non | Optionnel | Oui |
| Backups | Optionnel | Daily | Continuous |
| Monitoring | Basique | Complet | Complet + alerting |

### 2. Configuration externalisée

⚠️ **Ne JAMAIS hardcoder** :
- Credentials
- URLs de connexion
- Tailles de buffer pools
- Paramètres spécifiques à l'environnement

✅ **Utiliser** :
- Variables d'environnement
- ConfigMaps Kubernetes
- Secrets management (Vault, AWS Secrets Manager)
- Fichiers de configuration par environnement

### 3. Testing en continu

**Types de tests pour bases de données** :

1. **Unit tests** : Logique dans stored procedures/fonctions
2. **Integration tests** : Migrations sur copie de production
3. **Performance tests** : Benchmarks avant déploiement
4. **Disaster recovery tests** : Restore depuis backup

**Pipeline de test typique** :
```yaml
# .github/workflows/test.yml (simplifié)
test:
  runs-on: ubuntu-latest
  services:
    mariadb:
      image: mariadb:11.8
      env:
        MYSQL_ROOT_PASSWORD: test
  steps:
    - name: Run migrations
      run: flyway migrate
    - name: Run integration tests
      run: pytest tests/integration
    - name: Run performance tests
      run: sysbench oltp_read_write run
```

### 4. Immutabilité des images Docker

**Principe** : Une fois buildée et taguée, une image Docker ne doit JAMAIS être modifiée.

✅ **Bon** :
```dockerfile
# Dockerfile multi-stage
FROM mariadb:11.8 AS base
COPY custom.cnf /etc/mysql/conf.d/

FROM base AS production
# Production-specific optimizations
ENV MYSQL_INNODB_BUFFER_POOL_SIZE=16G

# Tag: myapp/mariadb:1.2.3-20250614
```

❌ **Mauvais** :
- Réutiliser le même tag pour différentes images
- Modifier l'image après push

### 5. Rollback strategy

**Migrations de schéma** :
- Toujours forward-only (pas de DOWN migrations en production)
- Migrations compatibles backward (expand-contract pattern)
- Validation en staging avant production

**Exemple expand-contract** :
```sql
-- V1: Expand - Ajouter nouvelle colonne (nullable)
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

-- Application déployée en dual-write (écrit dans ancienne ET nouvelle colonne)

-- V2: Contract - Rendre obligatoire + supprimer ancienne
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;
ALTER TABLE users DROP COLUMN old_phone_column;
```

### 6. Monitoring as Code

Définir alertes et dashboards dans du code versionné :

```yaml
# monitoring/alerts.yml (Prometheus AlertManager)
groups:
- name: mariadb
  rules:
  - alert: MariaDBDown
    expr: mysql_up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "MariaDB instance {{ $labels.instance }} down"
      
  - alert: HighConnectionUsage
    expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High connection usage on {{ $labels.instance }}"
```

### 7. Documentation as Code

**Principe** : Documenter dans le repository, pas dans un wiki séparé.

```
mariadb-infrastructure/
├── README.md                    # Vue d'ensemble
├── docs/
│   ├── architecture.md         # Diagrammes d'architecture
│   ├── runbooks/
│   │   ├── failover.md        # Procédure de failover
│   │   ├── restore.md         # Procédure de restore
│   │   └── scaling.md         # Procédure de scaling
│   └── ADR/                    # Architecture Decision Records
│       ├── 001-use-galera.md
│       └── 002-kubernetes-operator.md
├── terraform/
├── ansible/
└── k8s/
```

---

## Checklist DevOps pour MariaDB

Avant de considérer votre setup DevOps comme "production-ready" :

### Infrastructure
- [ ] Infrastructure définie en code (Terraform/Ansible)
- [ ] Tous les environnements reproductibles automatiquement
- [ ] Secrets gérés via solution dédiée (Vault, Secrets Manager)
- [ ] Network isolation (private subnets, security groups)
- [ ] Backup automatisé avec tests de restore réguliers

### Conteneurisation & Orchestration
- [ ] Images Docker optimisées et scannées (vulnérabilités)
- [ ] StatefulSets Kubernetes avec PersistentVolumes
- [ ] Resource limits et requests définis
- [ ] Health checks (liveness, readiness probes)
- [ ] Operator Kubernetes déployé (si applicable)

### CI/CD
- [ ] Pipelines automatisés pour tests de migrations
- [ ] Validation schéma en environnement de test
- [ ] Rollback procedure documentée et testée
- [ ] Déploiements sans downtime (blue-green ou rolling)
- [ ] GitOps implémenté (ArgoCD, FluxCD)

### Monitoring & Observabilité
- [ ] Métriques collectées (Prometheus + mysqld_exporter)
- [ ] Dashboards Grafana pour toutes les métriques clés
- [ ] Logs centralisés et structurés (ELK/Loki)
- [ ] Alerting configuré avec seuils pertinents
- [ ] Runbooks documentés pour chaque alerte

### Sécurité
- [ ] TLS activé pour toutes les connexions
- [ ] Principe de moindre privilège appliqué
- [ ] Audit logging activé
- [ ] Chiffrement at-rest configuré
- [ ] Scans de vulnérabilité réguliers

### Documentation
- [ ] Architecture documentée (diagrammes à jour)
- [ ] Runbooks à jour pour incidents courants
- [ ] ADRs (Architecture Decision Records) maintenus
- [ ] Procédures de disaster recovery testées
- [ ] Onboarding documentation pour nouveaux membres

---

## Cas d'usage : Migration d'une installation legacy vers cloud-native

**Situation initiale** :
- MariaDB 10.6 sur VM unique
- Configuration manuelle
- Backups cron artisanaux
- Pas de monitoring centralisé
- Déploiements manuels

**Situation cible** :
- MariaDB 11.8 sur Kubernetes
- Galera Cluster 3 nœuds via operator
- Infrastructure as Code complète
- CI/CD avec tests automatisés
- Monitoring Prometheus/Grafana

**Plan de migration (6 étapes)** :

### Phase 1 : Préparation (2 semaines)
1. Audit de la configuration actuelle
2. Définition architecture cible
3. Setup environnement de test Kubernetes
4. Formation équipe sur outils DevOps

### Phase 2 : Infrastructure as Code (3 semaines)
1. Codifier infrastructure Terraform
2. Créer playbooks Ansible
3. Setup environnement staging identique à prod
4. Tests de déploiement automatisé

### Phase 3 : Conteneurisation (2 semaines)
1. Créer images Docker optimisées
2. Tester StatefulSets Kubernetes
3. Valider persistance données
4. Benchmark performances

### Phase 4 : Monitoring & Observabilité (2 semaines)
1. Déployer stack Prometheus/Grafana
2. Configurer mysqld_exporter
3. Créer dashboards et alertes
4. Setup logging centralisé

### Phase 5 : CI/CD (3 semaines)
1. Setup pipelines GitHub Actions
2. Implémenter Flyway pour migrations
3. Tests automatisés (unit + integration)
4. Déploiement GitOps avec ArgoCD

### Phase 6 : Migration Production (1 semaine + validation)
1. Backup complet legacy
2. Réplication legacy → nouveau cluster
3. Tests de charge sur nouveau cluster
4. Basculement progressif du trafic
5. Surveillance intensive 72h
6. Décommissionnement legacy après validation

**Résultats attendus** :
- ✅ Déploiements de 2h → 15min
- ✅ Recovery time de 4h → 30min
- ✅ Incidents détectés avant impact utilisateurs
- ✅ Configuration reproductible à 100%
- ✅ Coût infra réduit de 30% (auto-scaling)

---

## ✅ Points clés à retenir

- **DevOps pour bases de données requiert une approche spécifique** : stateful vs stateless
- **Infrastructure as Code** : Terraform pour provisioning, Ansible pour configuration
- **Conteneurisation** : Docker avec volumes persistants, images optimisées
- **Kubernetes** : StatefulSets + operators pour automation niveau supérieur
- **GitOps** : Configuration déclarative versionnée, déploiements automatiques
- **CI/CD** : Pipelines avec tests de migrations, déploiements sans downtime
- **Migrations** : Flyway/Liquibase pour versioning, gh-ost pour online schema change
- **Monitoring** : Prometheus/Grafana pour métriques, observabilité complète (logs + traces)
- **Sécurité** : Secrets management, least privilege, encryption everywhere
- **Testing** : Tests automatisés à chaque étape, disaster recovery tests réguliers
- **Documentation** : Documentation as Code, runbooks versionnés
- **Reproductibilité** : Tout doit pouvoir être reconstruit depuis Git

💡 **La règle d'or** : Si ce n'est pas dans Git et automatisé, ça n'existe pas (ou ça ne devrait pas).

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 MariaDB on Kubernetes](https://mariadb.com/kb/en/running-mariadb-on-kubernetes/)
- [📖 MariaDB Operator Documentation](https://github.com/mariadb-operator/mariadb-operator)
- [📖 Galera Cluster on Kubernetes](https://mariadb.com/kb/en/galera-on-kubernetes/)

### Outils
- [🔧 Terraform MariaDB Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance)
- [🔧 Ansible MariaDB Collection](https://galaxy.ansible.com/community/mysql)
- [🔧 Prometheus mysqld_exporter](https://github.com/prometheus/mysqld_exporter)
- [🔧 Flyway](https://flywaydb.org/)
- [🔧 Liquibase](https://www.liquibase.org/)

### Guides et articles
- [📝 Database DevOps Best Practices](https://www.redgate.com/hub/database-devops)
- [📝 Kubernetes StatefulSets Guide](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [📝 GitOps Principles](https://www.gitops.tech/)

### Communauté
- [MariaDB Community Slack](https://mariadb.org/community/)
- [CNCF Kubernetes Slack #sig-storage](https://kubernetes.slack.com/)

---

## ➡️ Section suivante

**16.1 Infrastructure as Code pour MariaDB** : Nous plongerons dans les principes fondamentaux de l'IaC, l'approche déclarative vs impérative, et comment définir toute votre infrastructure MariaDB dans du code versionné et testable.

Vous apprendrez à :
- Comprendre les principes IaC (idempotence, immutabilité, versioning)
- Choisir entre Terraform et Ansible selon vos besoins
- Structurer vos repositories IaC
- Gérer les secrets et configuration sensitive
- Tester et valider votre infrastructure code

---

**MariaDB** : Version 11.8 LTS

⏭️ [Infrastructure as Code pour MariaDB](/16-devops-automatisation/01-infrastructure-as-code.md)

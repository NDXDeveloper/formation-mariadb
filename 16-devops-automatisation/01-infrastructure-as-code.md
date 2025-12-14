🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.1 Infrastructure as Code pour MariaDB

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Compréhension des bases MariaDB (chapitres 1-11)
> - Expérience avec Git et versioning
> - Familiarité avec YAML et JSON
> - Connaissances Linux système (bash, networking)
> - Notions de cloud computing (AWS, GCP ou Azure)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes fondamentaux de l'Infrastructure as Code appliqués aux bases de données
- **Distinguer** les approches déclaratives et impératives et choisir la bonne pour votre contexte
- **Structurer** un repository IaC pour MariaDB de manière maintenable et scalable
- **Gérer** les secrets et configurations sensibles de façon sécurisée
- **Implémenter** l'idempotence et l'immutabilité dans vos déploiements
- **Tester** et valider votre infrastructure avant déploiement en production

---

## Introduction

### Qu'est-ce que l'Infrastructure as Code ?

**Infrastructure as Code (IaC)** est une pratique DevOps qui consiste à **définir, provisionner et gérer l'infrastructure informatique via du code** plutôt que par des processus manuels ou des interfaces graphiques.

Pour MariaDB, cela signifie :

```
┌─────────────────────────────────────────────────────────────┐
│          Approche Traditionnelle (Manuelle)                 │
├─────────────────────────────────────────────────────────────┤
│  1. Se connecter en SSH au serveur                          │
│  2. Installer MariaDB manuellement (apt-get install...)     │
│  3. Éditer /etc/mysql/my.cnf à la main                      │
│  4. Créer utilisateurs avec des commandes SQL               │
│  5. Configurer firewall manuellement                        │
│  6. Espérer ne pas avoir fait d'erreurs...                  │
│  7. Documentation (si elle existe) dans un wiki/doc         │
└─────────────────────────────────────────────────────────────┘
                              ❌
            - Non reproductible
            - Erreurs humaines fréquentes
            - Configuration drift (divergence)
            - Pas de versioning
            - Difficile à auditer

                              ▼

┌─────────────────────────────────────────────────────────────┐
│       Approche Moderne (Infrastructure as Code)             │
├─────────────────────────────────────────────────────────────┤
│  1. Définir infrastructure dans fichiers .tf ou .yml        │
│  2. Versioner dans Git                                      │
│  3. Tester en environnement de test                         │
│  4. Code review + validation par équipe                     │
│  5. Exécuter: terraform apply ou ansible-playbook           │
│  6. Infrastructure déployée exactement comme définie        │
│  7. Documentation = le code lui-même                        │
└─────────────────────────────────────────────────────────────┘
                              ✅
            - Reproductible à 100%
            - Versioning complet
            - Rollback facile (git revert)
            - Audit trail automatique
            - Self-documented
```

### Pourquoi l'IaC est crucial pour MariaDB

Les bases de données posent des défis uniques pour l'IaC :

**1. État persistant** :
- Contrairement aux applications stateless, MariaDB stocke des données précieuses
- Destruction/recréation n'est pas une option viable
- L'IaC doit gérer à la fois l'infrastructure ET l'état des données

**2. Configuration complexe** :
- Des dizaines de paramètres impactant les performances
- Configurations différentes par environnement (dev/staging/prod)
- Besoin de cohérence entre serveurs d'un cluster

**3. Sécurité critique** :
- Credentials, certificats SSL, encryption keys
- Accès réseau strictement contrôlé
- Conformité réglementaire (GDPR, PCI-DSS)

**4. Haute disponibilité** :
- Réplication, clustering (Galera)
- Failover automatique
- Backup et disaster recovery

💡 **L'IaC résout ces défis** en fournissant une définition unique, versionnée et testable de toute l'infrastructure MariaDB.

---

## Principes fondamentaux de l'IaC

### 1. Déclaratif vs Impératif

**Approche impérative** : Définir **comment** atteindre l'état désiré (séquence d'actions)

```bash
# Impératif - Script shell
#!/bin/bash
# ❌ Problème: pas idempotent, peut échouer si déjà exécuté

apt-get update
apt-get install -y mariadb-server
systemctl start mariadb
mysql -e "CREATE DATABASE myapp;"
mysql -e "CREATE USER 'appuser'@'%' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL ON myapp.* TO 'appuser'@'%';"
```

**Problèmes** :
- Si exécuté 2 fois, erreurs (base/utilisateur déjà existants)
- Ordre d'exécution critique
- Difficile à maintenir
- Pas de gestion d'état

**Approche déclarative** : Définir **quel** état final on veut (le système détermine comment y arriver)

```yaml
# Déclaratif - Ansible
# ✅ Idempotent, peut être exécuté plusieurs fois

- name: Ensure MariaDB is installed
  package:
    name: mariadb-server
    state: present

- name: Ensure MariaDB is running
  service:
    name: mariadb
    state: started
    enabled: yes

- name: Ensure database exists
  mysql_db:
    name: myapp
    state: present

- name: Ensure user exists
  mysql_user:
    name: appuser
    password: "{{ vault_db_password }}"
    priv: 'myapp.*:ALL'
    state: present
```

**Avantages** :
- Idempotent : même résultat peu importe le nombre d'exécutions
- Lisible : description de l'état désiré
- Résilient : gère les erreurs et états partiels
- Maintenable : facile à modifier

💡 **Règle d'or** : Privilégiez toujours l'approche déclarative pour l'IaC des bases de données.

### 2. Idempotence

**Définition** : Une opération est idempotente si elle peut être appliquée plusieurs fois sans changer le résultat au-delà de l'application initiale.

**Mathématiquement** : `f(f(x)) = f(x)`

**En pratique pour MariaDB** :

```yaml
# ✅ Idempotent
- name: Ensure innodb_buffer_pool_size is set
  lineinfile:
    path: /etc/mysql/my.cnf
    regexp: '^innodb_buffer_pool_size'
    line: 'innodb_buffer_pool_size = 8G'
    state: present
  notify: restart mariadb

# Si exécuté 1 fois → ligne ajoutée
# Si exécuté 10 fois → même résultat (ligne présente une seule fois)
```

```yaml
# ❌ Non idempotent
- name: Add buffer pool size (MAUVAIS EXEMPLE)
  shell: echo "innodb_buffer_pool_size = 8G" >> /etc/mysql/my.cnf
  
# Si exécuté 10 fois → 10 lignes identiques dans le fichier !
```

**Test d'idempotence** :

```bash
# Test simple
ansible-playbook mariadb.yml  # Première exécution
ansible-playbook mariadb.yml  # Deuxième exécution

# Résultat attendu sur la 2e exécution:
# changed=0  ok=X  unreachable=0  failed=0
```

### 3. Immutabilité

**Principe** : Au lieu de modifier l'infrastructure existante, créer une nouvelle version et basculer.

**Pour les VMs/conteneurs MariaDB** :

```
Approche mutable (ancienne):
┌──────────┐
│ Server 1 │  → Update in-place → Modifications appliquées
└──────────┘     (risqué)          Configuration drift possible

Approche immutable (moderne):
┌──────────┐                     ┌──────────┐
│ Server 1 │                     │ Server 2 │
│ (v1.0)   │  → Créer nouveau →  │ (v1.1)   │  → Switch trafic
└──────────┘     serveur         └──────────┘     + Détruire v1.0
```

**Application à MariaDB** :

```hcl
# Terraform - Infrastructure immutable
resource "aws_db_instance" "mariadb" {
  identifier = "myapp-db-${var.version}"  # Nouveau nom à chaque version
  
  engine         = "mariadb"
  engine_version = "11.8"
  instance_class = "db.r6g.xlarge"
  
  # Configuration as code
  parameter_group_name = aws_db_parameter_group.mariadb.name
  
  # Immutabilité
  apply_immediately     = false  # Appliqué lors maintenance window
  skip_final_snapshot   = false  # Toujours créer snapshot avant destroy
  final_snapshot_identifier = "myapp-db-final-${var.version}"
  
  lifecycle {
    create_before_destroy = true  # Créer nouveau avant détruire ancien
  }
}
```

**Avantages pour MariaDB** :
- Rollback instantané (pointer vers ancienne version)
- Tests sur nouveau système avant basculement
- Zero-downtime deployments
- Configuration garantie identique

⚠️ **Attention** : Immutabilité s'applique à l'infrastructure, pas aux données. Les données doivent être migrées ou répliquées.

### 4. Versioning et Single Source of Truth

**Tout dans Git** :

```
mariadb-infrastructure/
├── .git/                       # Historique complet des changements
├── README.md                   # Documentation principale
├── terraform/
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── terraform.tfvars
│   │   ├── staging/
│   │   └── production/
│   └── modules/
│       └── mariadb/
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
├── ansible/
│   ├── inventories/
│   │   ├── dev.yml
│   │   ├── staging.yml
│   │   └── production.yml
│   ├── playbooks/
│   │   ├── mariadb-install.yml
│   │   ├── mariadb-configure.yml
│   │   └── mariadb-galera-cluster.yml
│   └── roles/
│       └── mariadb/
│           ├── defaults/
│           ├── tasks/
│           ├── templates/
│           └── handlers/
└── docs/
    ├── architecture.md
    └── runbooks/
```

**Workflow Git standard** :

```bash
# 1. Créer branche pour changement
git checkout -b feature/increase-buffer-pool

# 2. Modifier configuration
vim terraform/environments/production/terraform.tfvars
# innodb_buffer_pool_size: 16G → 32G

# 3. Tester en local ou staging
terraform plan

# 4. Commit avec message descriptif
git add .
git commit -m "feat(mariadb): increase buffer pool to 32G for better performance

Refs: JIRA-1234"

# 5. Push et créer Pull Request
git push origin feature/increase-buffer-pool

# 6. Code Review par équipe
# 7. Merge dans main après approbation
# 8. CI/CD applique automatiquement
```

**Avantages** :
- 📜 **Audit trail complet** : Qui a changé quoi et quand
- 🔄 **Rollback facile** : `git revert` pour annuler
- 👥 **Collaboration** : Code review, approbations
- 🧪 **Testing** : Branches pour tester sans impact
- 📖 **Documentation** : Historique Git = documentation des décisions

### 5. DRY (Don't Repeat Yourself)

**Problème** : Duplication de configuration entre environnements

```yaml
# ❌ Duplication (dev.yml et prod.yml presque identiques)
# dev.yml
mariadb_version: "11.8"
mariadb_buffer_pool: "2G"
mariadb_max_connections: 100

# prod.yml
mariadb_version: "11.8"          # ← Dupliqué
mariadb_buffer_pool: "32G"       # ← Différent
mariadb_max_connections: 1000    # ← Différent
```

**Solution** : Variables et templating

```yaml
# ✅ Configuration DRY avec variables
# group_vars/all.yml (configuration commune)
mariadb_version: "11.8"
mariadb_charset: "utf8mb4"
mariadb_collation: "utf8mb4_unicode_ci"

# group_vars/dev.yml (spécifique dev)
mariadb_buffer_pool: "2G"
mariadb_max_connections: 100
environment: "development"

# group_vars/production.yml (spécifique prod)
mariadb_buffer_pool: "32G"
mariadb_max_connections: 1000
environment: "production"

# Template my.cnf.j2 (utilisé partout)
[mysqld]
# Common settings
character-set-server = {{ mariadb_charset }}
collation-server = {{ mariadb_collation }}

# Environment-specific
innodb_buffer_pool_size = {{ mariadb_buffer_pool }}
max_connections = {{ mariadb_max_connections }}

# Conditional based on environment
{% if environment == "production" %}
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
{% else %}
innodb_flush_log_at_trx_commit = 2
sync_binlog = 0
{% endif %}
```

**Modules réutilisables (Terraform)** :

```hcl
# modules/mariadb/main.tf
variable "environment" {}
variable "instance_class" {}
variable "allocated_storage" {}

resource "aws_db_instance" "this" {
  identifier = "mariadb-${var.environment}"
  instance_class = var.instance_class
  allocated_storage = var.allocated_storage
  # ... configuration commune
}

# Utilisation
module "mariadb_production" {
  source = "./modules/mariadb"
  
  environment = "production"
  instance_class = "db.r6g.4xlarge"
  allocated_storage = 1000
}

module "mariadb_staging" {
  source = "./modules/mariadb"
  
  environment = "staging"
  instance_class = "db.r6g.xlarge"
  allocated_storage = 200
}
```

---

## Déclaratif vs Impératif : Comparaison approfondie

### Tableau comparatif

| Aspect | Impératif | Déclaratif |
|--------|-----------|------------|
| **Focus** | Comment faire | Quoi obtenir |
| **Idempotence** | ❌ Non par défaut | ✅ Oui par conception |
| **Complexité** | 📈 Augmente avec taille | 📊 Reste gérable |
| **Gestion d'état** | Manuelle | Automatique |
| **Rollback** | Complexe | Simple (état précédent) |
| **Ordre d'exécution** | Critique | Géré par l'outil |
| **Exemple d'outil** | Bash scripts, custom Python | Terraform, Ansible, CloudFormation |

### Exemple concret : Installation MariaDB en cluster

**Approche impérative (Bash)** :

```bash
#!/bin/bash
# deploy-mariadb-cluster.sh
# ❌ Impératif - fragile et non idempotent

set -e

# Variables
NODES=("node1" "node2" "node3")
CLUSTER_NAME="galera_cluster"
CLUSTER_ADDRESS="gcomm://node1,node2,node3"

# Installation sur chaque nœud
for node in "${NODES[@]}"; do
  echo "Installing on $node..."
  ssh $node "apt-get update && apt-get install -y mariadb-server galera-4"
  
  # Configuration
  ssh $node "cat > /etc/mysql/mariadb.conf.d/99-galera.cnf <<EOF
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_name=$CLUSTER_NAME
wsrep_cluster_address=$CLUSTER_ADDRESS
wsrep_node_address=$(hostname -I | cut -d' ' -f1)
wsrep_node_name=$node
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
EOF"
  
  # Redémarrage
  ssh $node "systemctl restart mariadb"
done

# Bootstrap premier nœud
ssh ${NODES[0]} "galera_new_cluster"

# Attendre que les autres rejoignent
sleep 30

echo "Cluster deployed!"
```

**Problèmes** :
1. ❌ Non idempotent : échoue si exécuté 2 fois
2. ❌ Pas de gestion d'état : ne sait pas ce qui existe déjà
3. ❌ Erreurs de timing (sleep 30 arbitraire)
4. ❌ Pas de rollback possible
5. ❌ Difficile à maintenir
6. ❌ Pas de validation de configuration

**Approche déclarative (Ansible)** :

```yaml
# playbooks/galera-cluster.yml
# ✅ Déclaratif - robuste et idempotent

---
- name: Deploy MariaDB Galera Cluster
  hosts: galera_nodes
  become: yes
  
  vars:
    mariadb_version: "11.8"
    cluster_name: "galera_cluster"
    
  tasks:
    - name: Install MariaDB repository
      apt_repository:
        repo: "deb [arch=amd64] https://downloads.mariadb.com/MariaDB/mariadb-{{ mariadb_version }}/repo/ubuntu {{ ansible_distribution_release }} main"
        state: present
        
    - name: Install MariaDB and Galera
      apt:
        name:
          - mariadb-server
          - galera-4
        state: present
        update_cache: yes
        
    - name: Configure Galera
      template:
        src: galera.cnf.j2
        dest: /etc/mysql/mariadb.conf.d/99-galera.cnf
        owner: root
        group: root
        mode: '0644'
      notify: restart mariadb
      
    - name: Ensure MariaDB is started
      service:
        name: mariadb
        state: started
        enabled: yes

- name: Bootstrap Galera cluster
  hosts: galera_nodes[0]
  become: yes
  tasks:
    - name: Check if cluster is already bootstrapped
      shell: mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';" | grep wsrep_cluster_size | awk '{print $2}'
      register: cluster_size
      changed_when: false
      failed_when: false
      
    - name: Bootstrap first node if needed
      command: galera_new_cluster
      when: cluster_size.stdout == "" or cluster_size.stdout == "0"
      
- name: Wait for cluster to be ready
  hosts: galera_nodes
  become: yes
  tasks:
    - name: Wait for all nodes to join
      wait_for:
        timeout: 300
        sleep: 10
      until: cluster_ready.stdout == "3"
      retries: 30
      delay: 10
      vars:
        cluster_ready: "{{ lookup('pipe', 'mysql -N -e \"SHOW STATUS LIKE \\'wsrep_cluster_size\\';\" | awk \\'{print $2}\\'') }}"

  handlers:
    - name: restart mariadb
      service:
        name: mariadb
        state: restarted
```

**Avantages** :
1. ✅ Idempotent : peut être exécuté plusieurs fois
2. ✅ Gestion d'état : vérifie ce qui existe
3. ✅ Gestion intelligente des timings (wait_for)
4. ✅ Rollback via Git
5. ✅ Maintenable et lisible
6. ✅ Validation intégrée

### Cas d'usage pour chaque approche

**Utiliser l'approche impérative quand** :
- ⚡ Tâches ponctuelles, one-off (migration de données unique)
- 🔧 Debugging/troubleshooting rapide
- 📊 Scripts de reporting/analyse
- 🧪 Prototypage rapide

**Utiliser l'approche déclarative quand** :
- 🏗️ Infrastructure persistante (toujours pour MariaDB production)
- 🔄 Configuration à maintenir dans le temps
- 👥 Collaboration en équipe
- 🧪 Besoin de tests et validation
- 📜 Conformité et audit

💡 **Recommandation** : Pour MariaDB en production, **toujours utiliser l'approche déclarative** (Terraform + Ansible).

---

## Structure d'un repository IaC pour MariaDB

### Architecture recommandée

```
mariadb-infrastructure/
│
├── README.md                          # Documentation principale
├── .gitignore                         # Ignorer secrets, .terraform/, etc.
├── .pre-commit-config.yaml           # Hooks pre-commit (linting, secrets scan)
│
├── docs/                              # Documentation détaillée
│   ├── architecture.md                # Diagrammes d'architecture
│   ├── disaster-recovery.md          # Procédures DR
│   ├── runbooks/
│   │   ├── failover.md
│   │   ├── backup-restore.md
│   │   └── scaling.md
│   └── ADR/                          # Architecture Decision Records
│       ├── 001-use-terraform.md
│       ├── 002-galera-vs-replication.md
│       └── 003-backup-strategy.md
│
├── terraform/                         # Provisioning infrastructure
│   ├── .terraform-version            # Version Terraform à utiliser
│   ├── backend.tf                    # Configuration remote state (S3, etc.)
│   ├── provider.tf                   # Providers (AWS, GCP, Azure)
│   │
│   ├── modules/                      # Modules réutilisables
│   │   ├── mariadb-instance/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── README.md
│   │   ├── mariadb-cluster/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── mariadb-backup/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   │
│   └── environments/                 # Configuration par environnement
│       ├── dev/
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   ├── terraform.tfvars     # Non versionné (secrets)
│       │   └── terraform.tfvars.example
│       ├── staging/
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   └── terraform.tfvars.example
│       └── production/
│           ├── main.tf
│           ├── variables.tf
│           └── terraform.tfvars.example
│
├── ansible/                           # Configuration management
│   ├── ansible.cfg                   # Configuration Ansible
│   ├── requirements.yml              # Collections et roles externes
│   │
│   ├── inventories/                  # Inventaires par environnement
│   │   ├── dev/
│   │   │   ├── hosts.yml
│   │   │   └── group_vars/
│   │   │       ├── all.yml
│   │   │       └── mariadb.yml
│   │   ├── staging/
│   │   │   ├── hosts.yml
│   │   │   └── group_vars/
│   │   └── production/
│   │       ├── hosts.yml
│   │       └── group_vars/
│   │           ├── all.yml
│   │           └── mariadb.yml
│   │
│   ├── playbooks/                    # Playbooks principaux
│   │   ├── site.yml                 # Playbook principal
│   │   ├── mariadb-install.yml
│   │   ├── mariadb-configure.yml
│   │   ├── mariadb-galera.yml
│   │   ├── mariadb-replication.yml
│   │   └── mariadb-backup.yml
│   │
│   ├── roles/                        # Roles Ansible
│   │   ├── mariadb/
│   │   │   ├── defaults/
│   │   │   │   └── main.yml
│   │   │   ├── tasks/
│   │   │   │   ├── main.yml
│   │   │   │   ├── install.yml
│   │   │   │   ├── configure.yml
│   │   │   │   └── secure.yml
│   │   │   ├── templates/
│   │   │   │   ├── my.cnf.j2
│   │   │   │   └── galera.cnf.j2
│   │   │   ├── handlers/
│   │   │   │   └── main.yml
│   │   │   └── README.md
│   │   ├── mariadb-backup/
│   │   └── maxscale/
│   │
│   └── group_vars/
│       └── vault.yml                 # Secrets chiffrés avec ansible-vault
│
├── kubernetes/                        # Manifests K8s (si applicable)
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   └── pvc.yaml
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── production/
│
├── scripts/                           # Scripts utilitaires
│   ├── validate.sh                   # Validation pre-commit
│   ├── backup.sh                     # Scripts backup
│   └── test-connectivity.sh
│
└── tests/                            # Tests infrastructure
    ├── terraform/
    │   └── validate_test.go         # Tests Terratest
    └── ansible/
        └── test_mariadb.py          # Tests avec pytest + testinfra
```

### Fichiers clés et leurs rôles

#### `.gitignore`

```gitignore
# Terraform
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfvars          # Secrets ne doivent PAS être versionnés
crash.log
override.tf
override.tf.json
.terraformrc
terraform.rc

# Ansible
*.retry
*.vault_pass
.vault_password

# Secrets généraux
secrets/
*.pem
*.key
credentials.json

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db
```

#### `README.md` - Template

```markdown
# MariaDB Infrastructure

Infrastructure as Code pour nos déploiements MariaDB.

## 📋 Table des matières

- [Architecture](#architecture)
- [Prérequis](#prérequis)
- [Quick Start](#quick-start)
- [Environnements](#environnements)
- [Déploiement](#déploiement)
- [Maintenance](#maintenance)
- [Disaster Recovery](#disaster-recovery)

## Architecture

```
[Insérer diagramme architecture]
```

## Prérequis

- Terraform >= 1.6
- Ansible >= 2.15
- AWS CLI configuré (ou autre cloud provider)
- Accès aux secrets (Vault/1Password/AWS Secrets Manager)

## Quick Start

### 1. Provisioning infrastructure (Terraform)

```bash
cd terraform/environments/dev
terraform init
terraform plan
terraform apply
```

### 2. Configuration serveurs (Ansible)

```bash
cd ansible
ansible-playbook -i inventories/dev playbooks/site.yml
```

## Environnements

| Env | Description | Terraform workspace | Ansible inventory |
|-----|-------------|---------------------|-------------------|
| dev | Développement local | dev | inventories/dev |
| staging | Pré-production | staging | inventories/staging |
| production | Production | production | inventories/production |

## Déploiement

Voir [docs/deployment.md](docs/deployment.md)

## Maintenance

- [Backup/Restore](docs/runbooks/backup-restore.md)
- [Scaling](docs/runbooks/scaling.md)
- [Failover](docs/runbooks/failover.md)

## Contacts

- Équipe DBA: dba@example.com
- On-call: +33 X XX XX XX XX
```

---

## Gestion des secrets

### Le problème des secrets

**Secrets à gérer pour MariaDB** :
- 🔐 Mots de passe root et utilisateurs applicatifs
- 🔑 Certificats SSL/TLS
- 🗝️ Clés de chiffrement (encryption at rest)
- 📝 Credentials cloud provider (AWS, GCP, Azure)
- 🔒 Tokens API (backup, monitoring)

⚠️ **RÈGLE ABSOLUE** : **Ne JAMAIS committer de secrets en clair dans Git**

```yaml
# ❌ DANGEREUX - Ne jamais faire ça !
mariadb_root_password: "SuperSecretP@ssw0rd123"  # En clair dans Git !
```

### Solutions de gestion des secrets

#### 1. Ansible Vault

**Pour** : Chiffrer des fichiers ou variables dans les playbooks Ansible

```bash
# Créer un fichier de secrets chiffré
ansible-vault create group_vars/production/vault.yml

# Contenu (sera chiffré)
vault_mariadb_root_password: "SuperSecretP@ssw0rd123"
vault_mariadb_app_password: "AppUserP@ssw0rd456"
vault_backup_s3_access_key: "AKIAIOSFODNN7EXAMPLE"
vault_backup_s3_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**Utilisation dans playbooks** :

```yaml
# playbooks/mariadb-configure.yml
- name: Configure MariaDB
  hosts: mariadb_servers
  vars:
    mariadb_root_password: "{{ vault_mariadb_root_password }}"  # Référence au vault
    
  tasks:
    - name: Set root password
      mysql_user:
        name: root
        password: "{{ mariadb_root_password }}"
        host_all: yes
        state: present
```

**Exécution** :

```bash
# Avec mot de passe en prompt
ansible-playbook -i inventories/production playbooks/site.yml --ask-vault-pass

# Avec fichier de mot de passe (à ne pas versionner !)
ansible-playbook -i inventories/production playbooks/site.yml --vault-password-file ~/.vault_pass

# Avec variable d'environnement
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook -i inventories/production playbooks/site.yml
```

**Avantages** :
- ✅ Intégré nativement à Ansible
- ✅ Simple à utiliser
- ✅ Fichiers versionnés (chiffrés)

**Inconvénients** :
- ❌ Partage du mot de passe vault entre équipe (mot de passe commun)
- ❌ Rotation des secrets complexe
- ❌ Pas d'audit trail des accès

#### 2. HashiCorp Vault

**Pour** : Gestion centralisée des secrets avec contrôle d'accès granulaire

**Architecture** :

```
┌─────────────────────────────────────────────┐
│           HashiCorp Vault Server            │
│  ┌───────────────────────────────────────┐  │
│  │   Database Secrets Engine             │  │
│  │   - Dynamic credentials               │  │
│  │   - Rotation automatique              │  │
│  │   - Lease management                  │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │   KV Secrets Engine                   │  │
│  │   - Static passwords                  │  │
│  │   - Certificates                      │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
                   ▲
                   │ API calls
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Terraform│  │ Ansible │  │   App   │
└─────────┘  └─────────┘  └─────────┘
```

**Configuration Vault** :

```bash
# Activer le secrets engine pour databases
vault secrets enable database

# Configurer la connexion MariaDB
vault write database/config/mariadb \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mariadb.example.com:3306)/" \
    allowed_roles="app-role,readonly-role" \
    username="vault" \
    password="vault-password"

# Créer un role avec credentials dynamiques
vault write database/roles/app-role \
    db_name=mariadb \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```

**Utilisation avec Ansible** :

```yaml
# playbooks/deploy-app.yml
- name: Deploy application with dynamic MariaDB credentials
  hosts: app_servers
  
  tasks:
    - name: Get MariaDB credentials from Vault
      set_fact:
        db_creds: "{{ lookup('hashi_vault', 'secret=database/creds/app-role') }}"
    
    - name: Configure application with dynamic credentials
      template:
        src: app-config.j2
        dest: /etc/app/config.yml
      vars:
        db_username: "{{ db_creds.username }}"
        db_password: "{{ db_creds.password }}"
```

**Avantages** :
- ✅ Credentials dynamiques (créés à la demande, expiration automatique)
- ✅ Rotation automatique des secrets
- ✅ Audit trail complet (qui a accédé à quoi et quand)
- ✅ Contrôle d'accès granulaire (policies)
- ✅ Révocation immédiate en cas de compromission

**Inconvénients** :
- ❌ Infrastructure supplémentaire à maintenir
- ❌ Complexité accrue
- ❌ Coût (pour version Enterprise)

#### 3. Cloud Provider Secrets Managers

**AWS Secrets Manager** :

```hcl
# terraform/environments/production/secrets.tf

# Créer secret pour mot de passe root MariaDB
resource "aws_secretsmanager_secret" "mariadb_root_password" {
  name = "mariadb/production/root-password"
  description = "MariaDB root password for production"
  
  recovery_window_in_days = 30  # Période de récupération si suppression accidentelle
}

resource "aws_secretsmanager_secret_version" "mariadb_root_password" {
  secret_id = aws_secretsmanager_secret.mariadb_root_password.id
  secret_string = random_password.mariadb_root.result
}

# Générer mot de passe aléatoire
resource "random_password" "mariadb_root" {
  length  = 32
  special = true
}

# Utiliser dans RDS instance
resource "aws_db_instance" "mariadb" {
  identifier = "production-mariadb"
  
  engine         = "mariadb"
  engine_version = "11.8"
  
  username = "admin"
  password = random_password.mariadb_root.result
  
  # ... reste de la config
}

# Output ARN du secret (pas le secret lui-même !)
output "mariadb_root_password_arn" {
  value = aws_secretsmanager_secret.mariadb_root_password.arn
  description = "ARN of MariaDB root password in Secrets Manager"
}
```

**Récupération du secret avec Ansible** :

```yaml
# playbooks/configure-app.yml
- name: Configure application
  hosts: app_servers
  
  tasks:
    - name: Get MariaDB password from AWS Secrets Manager
      set_fact:
        mariadb_password: "{{ lookup('aws_secret', 'mariadb/production/root-password', region='eu-west-1') }}"
    
    - name: Configure database connection
      template:
        src: db-config.j2
        dest: /etc/app/database.conf
      vars:
        db_host: "{{ rds_endpoint }}"
        db_password: "{{ mariadb_password }}"
```

**Rotation automatique** :

```hcl
# Rotation automatique tous les 30 jours
resource "aws_secretsmanager_secret_rotation" "mariadb_root" {
  secret_id           = aws_secretsmanager_secret.mariadb_root_password.id
  rotation_lambda_arn = aws_lambda_function.rotate_mariadb_password.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

**Avantages** :
- ✅ Intégration native avec cloud provider
- ✅ Rotation automatique
- ✅ Audit trail (CloudTrail)
- ✅ Chiffrement géré par KMS
- ✅ Haute disponibilité

**Inconvénients** :
- ❌ Lock-in cloud provider
- ❌ Coût (facturé par secret)

#### 4. Secrets dans Kubernetes

```yaml
# k8s/base/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-credentials
  namespace: databases
type: Opaque
stringData:  # Sera automatiquement base64 encodé
  root-password: "SuperSecretPassword"  # ⚠️ Ne pas versionner ce fichier !
  app-password: "AppUserPassword"
```

**Meilleures pratiques Kubernetes Secrets** :

1. **Sealed Secrets** (chiffrement côté client) :

```bash
# Installer Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Créer secret et le sceller
echo -n "SuperSecretPassword" | kubectl create secret generic mariadb-root-password \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# sealed-secret.yaml peut être versionné en toute sécurité dans Git
```

2. **External Secrets Operator** (synchronisation depuis Vault/AWS Secrets Manager) :

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mariadb-credentials
  namespace: databases
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: mariadb-credentials
    creationPolicy: Owner
  data:
  - secretKey: root-password
    remoteRef:
      key: secret/mariadb/production
      property: root-password
```

### Comparaison des solutions

| Solution | Complexité | Coût | Rotation auto | Audit | Multi-cloud | Recommandé pour |
|----------|------------|------|---------------|-------|-------------|-----------------|
| **Ansible Vault** | 🟢 Faible | Gratuit | ❌ Non | ❌ Non | ✅ Oui | Petites équipes, simple |
| **HashiCorp Vault** | 🟡 Moyenne | Gratuit (OSS) | ✅ Oui | ✅ Oui | ✅ Oui | Production enterprise |
| **AWS Secrets Manager** | 🟢 Faible | $0.40/secret/mois | ✅ Oui | ✅ Oui | ❌ Non | AWS-centric |
| **GCP Secret Manager** | 🟢 Faible | $0.06/secret/mois | ✅ Oui | ✅ Oui | ❌ Non | GCP-centric |
| **Azure Key Vault** | 🟢 Faible | ~$0.03/10k ops | ✅ Oui | ✅ Oui | ❌ Non | Azure-centric |
| **Sealed Secrets (K8s)** | 🟡 Moyenne | Gratuit | ❌ Non | ❌ Non | ✅ Oui | Kubernetes-native |

💡 **Recommandation** :
- **Petites infras** : Ansible Vault
- **Production multi-cloud** : HashiCorp Vault
- **Cloud-native (AWS/GCP/Azure)** : Secrets Manager du provider
- **Kubernetes** : External Secrets Operator + Vault ou cloud provider

---

## Testing et validation

### 1. Validation syntaxique

**Terraform** :

```bash
# Validation syntaxe HCL
terraform fmt -check -recursive

# Validation configuration
terraform validate

# Plan sans appliquer (dry-run)
terraform plan
```

**Ansible** :

```bash
# Validation syntaxe YAML
ansible-playbook --syntax-check playbooks/site.yml

# Dry-run (check mode)
ansible-playbook -i inventories/dev playbooks/site.yml --check

# Lint avec ansible-lint
ansible-lint playbooks/site.yml
```

### 2. Tests automatisés

**Terratest (Go)** :

```go
// tests/terraform/mariadb_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestMariaDBInstance(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../../terraform/environments/dev",
        Vars: map[string]interface{}{
            "instance_class": "db.t3.micro",
        },
    }
    
    defer terraform.Destroy(t, terraformOptions)
    
    // Apply
    terraform.InitAndApply(t, terraformOptions)
    
    // Validate outputs
    dbEndpoint := terraform.Output(t, terraformOptions, "db_endpoint")
    assert.NotEmpty(t, dbEndpoint, "DB endpoint should not be empty")
    
    instanceClass := terraform.Output(t, terraformOptions, "instance_class")
    assert.Equal(t, "db.t3.micro", instanceClass)
}
```

**Testinfra (Python - pour Ansible)** :

```python
# tests/ansible/test_mariadb.py
import pytest

def test_mariadb_installed(host):
    mariadb = host.package("mariadb-server")
    assert mariadb.is_installed
    assert mariadb.version.startswith("11.8")

def test_mariadb_running(host):
    mariadb = host.service("mariadb")
    assert mariadb.is_running
    assert mariadb.is_enabled

def test_mariadb_listening(host):
    socket = host.socket("tcp://0.0.0.0:3306")
    assert socket.is_listening

def test_mariadb_config(host):
    config = host.file("/etc/mysql/my.cnf")
    assert config.exists
    assert config.contains("innodb_buffer_pool_size")

def test_database_exists(host):
    cmd = host.run("mysql -e 'SHOW DATABASES LIKE \"myapp\"'")
    assert cmd.rc == 0
    assert "myapp" in cmd.stdout
```

### 3. Pre-commit hooks

**`.pre-commit-config.yaml`** :

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key  # Détecte les clés privées
      
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.6
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      
  - repo: https://github.com/ansible/ansible-lint
    rev: v6.22.0
    hooks:
      - id: ansible-lint
        files: \.(yaml|yml)$
        
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets  # Scan des secrets dans le code
        args: ['--baseline', '.secrets.baseline']
```

**Installation** :

```bash
# Installer pre-commit
pip install pre-commit

# Installer hooks dans repo Git
cd mariadb-infrastructure
pre-commit install

# Tester manuellement
pre-commit run --all-files
```

### 4. CI/CD Pipeline validation

**GitHub Actions** :

```yaml
# .github/workflows/validate.yml
name: Validate Infrastructure

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  terraform-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0
          
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: terraform/
        
      - name: Terraform Init
        run: terraform init
        working-directory: terraform/environments/dev
        
      - name: Terraform Validate
        run: terraform validate
        working-directory: terraform/environments/dev
        
      - name: Terraform Plan
        run: terraform plan
        working-directory: terraform/environments/dev
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  
  ansible-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Install Ansible
        run: pip install ansible ansible-lint
        
      - name: Ansible Lint
        run: ansible-lint playbooks/
        working-directory: ansible/
        
      - name: Ansible Syntax Check
        run: ansible-playbook --syntax-check playbooks/site.yml
        working-directory: ansible/
  
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          
      - name: Detect secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
```

---

## ✅ Points clés à retenir

- **Infrastructure as Code** : Définir toute l'infrastructure dans du code versionné, reproductible et testable
- **Déclaratif > Impératif** : Privilégier l'approche déclarative (Terraform, Ansible) pour l'idempotence et la maintenabilité
- **Idempotence** : Les opérations doivent pouvoir être répétées sans effets de bord
- **Immutabilité** : Créer nouvelle infrastructure plutôt que modifier l'existante (pour compute, pas pour données)
- **Versioning** : Git comme single source of truth, tout doit être versionné
- **DRY** : Éviter duplication avec variables, templates et modules
- **Secrets** : Ne JAMAIS versionner de secrets en clair, utiliser Vault/Secrets Manager
- **Structure** : Organiser repository de façon claire (environnements, modules, roles)
- **Testing** : Valider syntaxe, linter, tests automatisés, pre-commit hooks
- **Documentation** : Le code doit être auto-documenté, compléter avec ADRs et runbooks

💡 **La règle d'or** : Si ce n'est pas dans Git, ça n'existe pas. Si c'est un secret, ne le mettez pas en clair dans Git.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Terraform Documentation](https://www.terraform.io/docs)
- [📖 Ansible Documentation](https://docs.ansible.com/)
- [📖 HashiCorp Vault](https://www.vaultproject.io/docs)
- [📖 AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)

### Guides et best practices
- [📝 Terraform Best Practices](https://www.terraform-best-practices.com/)
- [📝 Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [📝 Infrastructure as Code Patterns](https://www.oreilly.com/library/view/infrastructure-as-code/9781098114664/)

### Outils
- [🔧 Terratest](https://terratest.gruntwork.io/) - Testing Terraform
- [🔧 Testinfra](https://testinfra.readthedocs.io/) - Testing Ansible
- [🔧 Pre-commit](https://pre-commit.com/) - Git hooks
- [🔧 Ansible Lint](https://ansible-lint.readthedocs.io/)
- [🔧 TFLint](https://github.com/terraform-linters/tflint)

### Articles de référence
- [Infrastructure as Code Best Practices](https://cloud.google.com/architecture/devops/devops-tech-infrastructure-as-code)
- [Managing Secrets in IaC](https://blog.gruntwork.io/a-comprehensive-guide-to-managing-secrets-in-your-terraform-code-1d586955ace1)

---

## ➡️ Section suivante

**16.2 Déploiement avec Ansible et Terraform** : Nous approfondirons l'utilisation pratique de Terraform pour provisionner l'infrastructure cloud et Ansible pour configurer les serveurs MariaDB. Vous découvrirez des playbooks complets, des modules Terraform réutilisables et des patterns de déploiement production-ready.

Vous apprendrez à :
- Créer des modules Terraform pour MariaDB (RDS, EC2, networking)
- Écrire des playbooks Ansible pour installation, configuration et clustering
- Gérer multi-environnements (dev/staging/production)
- Orchestrer Terraform + Ansible ensemble
- Implémenter des déploiements blue-green et rolling updates

---

**MariaDB** : Version 11.8 LTS

⏭️ [Déploiement avec Ansible/Terraform](/16-devops-automatisation/02-deploiement-ansible-terraform.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.2 Déploiement avec Ansible/Terraform

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : 
> - Section 16.1 Infrastructure as Code maîtrisée
> - Connaissance de base de Terraform (HCL)
> - Connaissance de base d'Ansible (YAML)
> - Expérience avec au moins un cloud provider (AWS/GCP/Azure)
> - Compréhension des réseaux (VPC, subnets, security groups)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Distinguer** les rôles complémentaires de Terraform et Ansible dans le déploiement MariaDB
- **Provisionner** une infrastructure MariaDB complète sur AWS, GCP ou Azure avec Terraform
- **Configurer** des serveurs MariaDB avec Ansible (installation, tuning, clustering)
- **Orchestrer** Terraform et Ansible ensemble dans un workflow cohérent
- **Implémenter** des déploiements multi-environnements (dev/staging/production)
- **Déployer** des architectures HA (réplication, Galera) de manière automatisée
- **Gérer** le state Terraform et l'inventaire Ansible dynamique

---

## Introduction

### Terraform vs Ansible : Rôles complémentaires

Terraform et Ansible sont souvent perçus comme des outils concurrents, mais pour le déploiement de MariaDB, **ils sont hautement complémentaires** :

```
┌─────────────────────────────────────────────────────────────────┐
│                        Stack DevOps MariaDB                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1️⃣  TERRAFORM (Infrastructure Provisioning)                    │
│      "Créer les ressources cloud"                               │
│                                                                 │
│      ┌──────────────────────────────────────────────┐           │
│      │ • VPC, Subnets, Security Groups              │           │
│      │ • EC2 Instances / VMs                        │           │
│      │ • RDS MariaDB (managed)                      │           │
│      │ • EBS Volumes / Persistent Disks             │           │
│      │ • Load Balancers                             │           │
│      │ • IAM Roles & Policies                       │           │
│      └──────────────────────────────────────────────┘           │
│                            │                                    │
│                            ▼                                    │
│  2️⃣  ANSIBLE (Configuration Management)                         │
│      "Configurer les serveurs"                                  │
│                                                                 │
│      ┌──────────────────────────────────────────────┐           │
│      │ • Installation MariaDB                       │           │
│      │ • Configuration my.cnf (tuning)              │           │
│      │ • Setup utilisateurs et bases                │           │
│      │ • Configuration Galera Cluster               │           │
│      │ • Setup réplication                          │           │
│      │ • Installation MaxScale / ProxySQL           │           │
│      │ • Configuration monitoring agents            │           │
│      └──────────────────────────────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Quand utiliser Terraform vs Ansible ?

| Tâche | Terraform | Ansible | Recommandation |
|-------|-----------|---------|----------------|
| Créer VPC et subnets | ✅ Parfait | ❌ Possible mais mal adapté | **Terraform** |
| Provisionner instances EC2 | ✅ Parfait | ⚠️ Possible (module ec2) | **Terraform** |
| Créer RDS MariaDB | ✅ Parfait | ❌ Non supporté | **Terraform** |
| Installer packages OS | ⚠️ Via user-data | ✅ Parfait | **Ansible** |
| Configurer my.cnf | ❌ Non adapté | ✅ Parfait | **Ansible** |
| Setup Galera Cluster | ❌ Non adapté | ✅ Parfait | **Ansible** |
| Gérer utilisateurs MariaDB | ⚠️ Via provider SQL | ✅ Parfait | **Ansible** |
| Rolling updates | ❌ Limité | ✅ Excellent | **Ansible** |
| Immutable infrastructure | ✅ Parfait | ⚠️ Possible | **Terraform** |
| Configuration drift detection | ✅ Oui (plan) | ⚠️ Limité | **Terraform** |

💡 **Règle d'or** : 
- **Terraform** → "**Quoi**" (quelle infrastructure)
- **Ansible** → "**Comment**" (comment configurer)

### Architecture de déploiement recommandée

```
┌───────────────────────────────────────────────────────────────┐
│                        Workflow Type                          │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Developer fait un changement                              │
│     ├── Modifie terraform/*.tf                                │
│     └── Modifie ansible/playbooks/*.yml                       │
│                                                               │
│  2. Git commit + Push                                         │
│     └── Déclenche CI/CD pipeline                              │
│                                                               │
│  3. CI/CD Pipeline                                            │
│     ├── Validation (terraform validate, ansible-lint)         │
│     ├── Tests (terraform plan, ansible --check)               │
│     ├── Security scan (tfsec, ansible-lint)                   │
│     └── Attente approbation (PR review)                       │
│                                                               │
│  4. Déploiement (après merge)                                 │
│     ├── Phase 1: Terraform Apply                              │
│     │   ├── Crée/modifie infrastructure cloud                 │
│     │   ├── Output: IPs des instances                         │
│     │   └── Génère inventaire dynamique Ansible               │
│     │                                                         │
│     └── Phase 2: Ansible Playbook                             │
│         ├── Utilise inventaire dynamique                      │
│         ├── Configure serveurs MariaDB                        │
│         ├── Setup cluster/réplication                         │
│         └── Valide état final                                 │
│                                                               │
│  5. Monitoring & Alerting                                     │
│     └── Prometheus/Grafana détecte anomalies                  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## Terraform pour MariaDB

### Cas d'usage 1 : RDS MariaDB sur AWS (Managed)

**Architecture cible** :

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Account                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                      VPC (10.0.0.0/16)                │  │
│  │                                                       │  │
│  │  ┌─────────────────┐         ┌─────────────────┐      │  │
│  │  │ Private Subnet  │         │ Private Subnet  │      │  │
│  │  │ AZ-1 (10.0.1.0) │         │ AZ-2 (10.0.2.0) │      │  │
│  │  │                 │         │                 │      │  │
│  │  │  ┌───────────┐  │         │  ┌───────────┐  │      │  │
│  │  │  │ RDS       │  │◄────────┤  │ RDS       │  │      │  │
│  │  │  │ Primary   │  │ Replica │  │ Standby   │  │      │  │
│  │  │  └───────────┘  │         │  └───────────┘  │      │  │
│  │  └─────────────────┘         └─────────────────┘      │  │
│  │           ▲                           ▲               │  │
│  │           │                           │               │  │
│  │  ┌────────┴───────────────────────────┴──────────┐    │  │
│  │  │          DB Subnet Group                      │    │  │
│  │  └───────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────┘  │
│                           ▲                                 │
│                           │                                 │
│                  ┌────────┴─────────┐                       │
│                  │  Security Group  │                       │
│                  │  Port 3306       │                       │
│                  └──────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

**Code Terraform complet** :

```hcl
# terraform/modules/rds-mariadb/variables.tf

variable "environment" {
  description = "Environment name (dev, staging, production)"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID where RDS will be deployed"
  type        = string
}

variable "private_subnet_ids" {
  description = "List of private subnet IDs for RDS"
  type        = list(string)
}

variable "mariadb_version" {
  description = "MariaDB engine version"
  type        = string
  default     = "11.8"
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.r6g.xlarge"
}

variable "allocated_storage" {
  description = "Allocated storage in GB"
  type        = number
  default     = 500
}

variable "max_allocated_storage" {
  description = "Maximum storage for autoscaling"
  type        = number
  default     = 1000
}

variable "backup_retention_period" {
  description = "Backup retention in days"
  type        = number
  default     = 30
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = true
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to connect"
  type        = list(string)
  default     = []
}

variable "parameter_family" {
  description = "DB parameter group family"
  type        = string
  default     = "mariadb11.8"
}
```

```hcl
# terraform/modules/rds-mariadb/main.tf

# DB Subnet Group
resource "aws_db_subnet_group" "mariadb" {
  name       = "${var.environment}-mariadb-subnet-group"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name        = "${var.environment}-mariadb-subnet-group"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Security Group
resource "aws_security_group" "mariadb" {
  name        = "${var.environment}-mariadb-sg"
  description = "Security group for MariaDB RDS instance"
  vpc_id      = var.vpc_id

  # Ingress rule - MariaDB port
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
    description = "MariaDB access from allowed CIDR blocks"
  }

  # Egress - Allow all (for updates, etc.)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }

  tags = {
    Name        = "${var.environment}-mariadb-sg"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# DB Parameter Group
resource "aws_db_parameter_group" "mariadb" {
  name   = "${var.environment}-mariadb-params"
  family = var.parameter_family

  # MariaDB 11.8 specific optimizations
  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  parameter {
    name  = "max_connections"
    value = "1000"
  }

  parameter {
    name  = "innodb_buffer_pool_size"
    value = "{DBInstanceClassMemory*3/4}"  # 75% of instance memory
  }

  parameter {
    name  = "innodb_log_file_size"
    value = "2147483648"  # 2GB
  }

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "2"
  }

  parameter {
    name  = "log_queries_not_using_indexes"
    value = "1"
  }

  # 🆕 MariaDB 11.8 - TLS by default
  parameter {
    name  = "require_secure_transport"
    value = "1"
  }

  tags = {
    Name        = "${var.environment}-mariadb-params"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Random password for initial setup
resource "random_password" "master" {
  length  = 32
  special = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# Store password in AWS Secrets Manager
resource "aws_secretsmanager_secret" "mariadb_master_password" {
  name        = "${var.environment}/mariadb/master-password"
  description = "MariaDB master password for ${var.environment}"

  recovery_window_in_days = 30

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "mariadb_master_password" {
  secret_id     = aws_secretsmanager_secret.mariadb_master_password.id
  secret_string = random_password.master.result
}

# RDS Instance
resource "aws_db_instance" "mariadb" {
  identifier = "${var.environment}-mariadb"

  # Engine configuration
  engine               = "mariadb"
  engine_version       = var.mariadb_version
  instance_class       = var.instance_class
  allocated_storage    = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type         = "gp3"
  storage_encrypted    = true

  # Database configuration
  db_name  = replace(var.environment, "-", "_")
  username = "admin"
  password = random_password.master.result
  port     = 3306

  # Network configuration
  db_subnet_group_name   = aws_db_subnet_group.mariadb.name
  vpc_security_group_ids = [aws_security_group.mariadb.id]
  publicly_accessible    = false
  multi_az              = var.multi_az

  # Parameter and option groups
  parameter_group_name = aws_db_parameter_group.mariadb.name

  # Backup configuration
  backup_retention_period = var.backup_retention_period
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  # Enhanced monitoring
  enabled_cloudwatch_logs_exports = ["error", "slowquery", "general"]
  monitoring_interval            = 60
  monitoring_role_arn           = aws_iam_role.rds_monitoring.arn

  # Performance Insights
  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  # Deletion protection
  deletion_protection = var.environment == "production" ? true : false
  skip_final_snapshot = var.environment == "production" ? false : true
  final_snapshot_identifier = var.environment == "production" ? "${var.environment}-mariadb-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}" : null

  # Apply changes immediately for non-production
  apply_immediately = var.environment != "production"

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  tags = {
    Name        = "${var.environment}-mariadb"
    Environment = var.environment
    ManagedBy   = "terraform"
  }

  lifecycle {
    # Prevent accidental deletion
    prevent_destroy = false  # Set to true for production

    # Ignore password changes (managed externally after initial creation)
    ignore_changes = [password]
  }
}

# Read Replica (optional, for production)
resource "aws_db_instance" "mariadb_replica" {
  count = var.environment == "production" ? 1 : 0

  identifier = "${var.environment}-mariadb-replica"

  replicate_source_db = aws_db_instance.mariadb.identifier

  instance_class    = var.instance_class
  publicly_accessible = false

  # Use same parameter group
  parameter_group_name = aws_db_parameter_group.mariadb.name

  # Monitoring
  enabled_cloudwatch_logs_exports = ["error", "slowquery"]
  monitoring_interval            = 60
  monitoring_role_arn           = aws_iam_role.rds_monitoring.arn

  performance_insights_enabled = true

  auto_minor_version_upgrade = true

  tags = {
    Name        = "${var.environment}-mariadb-replica"
    Environment = var.environment
    Role        = "read-replica"
    ManagedBy   = "terraform"
  }
}

# IAM Role for Enhanced Monitoring
resource "aws_iam_role" "rds_monitoring" {
  name = "${var.environment}-rds-monitoring-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "monitoring.rds.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  role       = aws_iam_role.rds_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}
```

```hcl
# terraform/modules/rds-mariadb/outputs.tf

output "db_instance_endpoint" {
  description = "Connection endpoint for the RDS instance"
  value       = aws_db_instance.mariadb.endpoint
}

output "db_instance_address" {
  description = "Address of the RDS instance"
  value       = aws_db_instance.mariadb.address
}

output "db_instance_port" {
  description = "Port of the RDS instance"
  value       = aws_db_instance.mariadb.port
}

output "db_instance_id" {
  description = "RDS instance ID"
  value       = aws_db_instance.mariadb.id
}

output "db_instance_arn" {
  description = "ARN of the RDS instance"
  value       = aws_db_instance.mariadb.arn
}

output "db_replica_endpoint" {
  description = "Connection endpoint for the read replica"
  value       = var.environment == "production" ? aws_db_instance.mariadb_replica[0].endpoint : null
}

output "db_name" {
  description = "Database name"
  value       = aws_db_instance.mariadb.db_name
}

output "master_username" {
  description = "Master username"
  value       = aws_db_instance.mariadb.username
  sensitive   = true
}

output "master_password_secret_arn" {
  description = "ARN of the secret containing master password"
  value       = aws_secretsmanager_secret.mariadb_master_password.arn
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.mariadb.id
}
```

**Utilisation du module** :

```hcl
# terraform/environments/production/main.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }

  # Remote state dans S3
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "mariadb/production/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "eu-west-1"

  default_tags {
    tags = {
      Project     = "MyApp"
      Environment = "production"
      ManagedBy   = "terraform"
      Owner       = "platform-team"
    }
  }
}

# Import existing VPC (ou créer un nouveau)
data "aws_vpc" "main" {
  id = "vpc-xxxxxxxxx"
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  tags = {
    Tier = "private"
  }
}

# Déployer MariaDB
module "mariadb" {
  source = "../../modules/rds-mariadb"

  environment         = "production"
  vpc_id             = data.aws_vpc.main.id
  private_subnet_ids = data.aws_subnets.private.ids

  mariadb_version = "11.8"
  instance_class  = "db.r6g.4xlarge"  # Production-grade instance

  allocated_storage     = 1000
  max_allocated_storage = 5000

  backup_retention_period = 35  # 35 jours pour production
  multi_az               = true  # HA obligatoire en production

  allowed_cidr_blocks = [
    "10.0.0.0/8",  # Internal VPC
  ]
}

# Outputs
output "mariadb_endpoint" {
  description = "MariaDB connection endpoint"
  value       = module.mariadb.db_instance_endpoint
}

output "mariadb_replica_endpoint" {
  description = "MariaDB read replica endpoint"
  value       = module.mariadb.db_replica_endpoint
}

output "password_secret_arn" {
  description = "ARN of the secret containing the database password"
  value       = module.mariadb.master_password_secret_arn
}
```

**Déploiement** :

```bash
# Initialiser Terraform
cd terraform/environments/production
terraform init

# Valider la configuration
terraform validate

# Voir le plan d'exécution
terraform plan -out=tfplan

# Appliquer (après revue)
terraform apply tfplan

# Outputs
terraform output mariadb_endpoint
# Résultat: production-mariadb.xxxxxxxxxx.eu-west-1.rds.amazonaws.com:3306
```

### Cas d'usage 2 : EC2 avec MariaDB auto-géré (Self-Managed)

**Architecture cible** :

```
┌──────────────────────────────────────────────────────────┐
│                    Galera Cluster (3 nodes)              │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐      │
│  │   Node 1   │◄──►│   Node 2   │◄──►│   Node 3   │      │
│  │  Primary   │    │   Sync     │    │   Sync     │      │
│  │  AZ-1      │    │   AZ-2     │    │   AZ-3     │      │
│  └────────────┘    └────────────┘    └────────────┘      │
│        ▲                 ▲                 ▲             │
│        │                 │                 │             │
│        └─────────────────┴─────────────────┘             │
│                          │                               │
│                   ┌──────┴───────┐                       │
│                   │ Load Balancer│                       │
│                   │  (MaxScale)  │                       │
│                   └──────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

```hcl
# terraform/modules/ec2-mariadb-galera/main.tf

# Data source pour l'AMI Ubuntu la plus récente
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-*-server-*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Security Group pour Galera
resource "aws_security_group" "galera" {
  name        = "${var.environment}-galera-sg"
  description = "Security group for Galera cluster nodes"
  vpc_id      = var.vpc_id

  # MariaDB port
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
    description = "MariaDB client connections"
  }

  # Galera replication traffic (inter-node)
  ingress {
    from_port = 4567
    to_port   = 4567
    protocol  = "tcp"
    self      = true
    description = "Galera replication traffic"
  }

  # Galera IST (Incremental State Transfer)
  ingress {
    from_port = 4568
    to_port   = 4568
    protocol  = "tcp"
    self      = true
    description = "Galera IST"
  }

  # Galera SST (State Snapshot Transfer)
  ingress {
    from_port = 4444
    to_port   = 4444
    protocol  = "tcp"
    self      = true
    description = "Galera SST"
  }

  # SSH (restreint au bastion)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.bastion_cidr_blocks
    description = "SSH from bastion"
  }

  # All outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-galera-sg"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# IAM Role pour les instances EC2
resource "aws_iam_role" "galera_instance" {
  name = "${var.environment}-galera-instance-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Policy pour accéder aux secrets
resource "aws_iam_role_policy" "galera_secrets" {
  name = "${var.environment}-galera-secrets-policy"
  role = aws_iam_role.galera_instance.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = aws_secretsmanager_secret.galera_passwords.arn
      }
    ]
  })
}

# CloudWatch logs policy
resource "aws_iam_role_policy_attachment" "galera_cloudwatch" {
  role       = aws_iam_role.galera_instance.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# Instance profile
resource "aws_iam_instance_profile" "galera" {
  name = "${var.environment}-galera-instance-profile"
  role = aws_iam_role.galera_instance.name
}

# EBS Volumes pour données MariaDB
resource "aws_ebs_volume" "galera_data" {
  count = var.cluster_size

  availability_zone = element(var.availability_zones, count.index)
  size              = var.data_volume_size
  type              = "gp3"
  iops              = var.data_volume_iops
  throughput        = var.data_volume_throughput
  encrypted         = true

  tags = {
    Name        = "${var.environment}-galera-data-${count.index + 1}"
    Environment = var.environment
    NodeIndex   = count.index + 1
    ManagedBy   = "terraform"
  }
}

# EC2 Instances pour Galera Cluster
resource "aws_instance" "galera_node" {
  count = var.cluster_size

  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  subnet_id              = element(var.private_subnet_ids, count.index)
  vpc_security_group_ids = [aws_security_group.galera.id]
  iam_instance_profile   = aws_iam_instance_profile.galera.name

  key_name = var.ssh_key_name

  # User data pour configuration initiale minimale
  user_data = templatefile("${path.module}/templates/user_data.sh.tpl", {
    node_name     = "galera-${count.index + 1}"
    node_index    = count.index + 1
    environment   = var.environment
    aws_region    = var.aws_region
  })

  # Monitoring détaillé
  monitoring = true

  # Metadata options (IMDSv2)
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  # Root volume
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 50
    encrypted             = true
    delete_on_termination = true
  }

  tags = {
    Name        = "${var.environment}-galera-node-${count.index + 1}"
    Environment = var.environment
    Role        = "galera-node"
    NodeIndex   = count.index + 1
    ManagedBy   = "terraform"
  }

  lifecycle {
    # Éviter recréation si AMI change
    ignore_changes = [ami]
  }
}

# Attach EBS volumes
resource "aws_volume_attachment" "galera_data" {
  count = var.cluster_size

  device_name = "/dev/xvdf"
  volume_id   = aws_ebs_volume.galera_data[count.index].id
  instance_id = aws_instance.galera_node[count.index].id

  # Ne pas forcer le détachement (peut causer perte de données)
  force_detach = false
}

# Secrets pour les passwords
resource "random_password" "galera_root" {
  length  = 32
  special = true
}

resource "random_password" "galera_sst" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "galera_passwords" {
  name        = "${var.environment}/galera/passwords"
  description = "Galera cluster passwords"

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "galera_passwords" {
  secret_id = aws_secretsmanager_secret.galera_passwords.id
  secret_string = jsonencode({
    root_password = random_password.galera_root.result
    sst_password  = random_password.galera_sst.result
  })
}

# DNS interne (Route53 private zone)
resource "aws_route53_record" "galera_nodes" {
  count = var.cluster_size

  zone_id = var.route53_zone_id
  name    = "galera-${count.index + 1}.${var.environment}.internal"
  type    = "A"
  ttl     = 60

  records = [aws_instance.galera_node[count.index].private_ip]
}

# DNS pour le cluster (round-robin sur tous les nœuds)
resource "aws_route53_record" "galera_cluster" {
  zone_id = var.route53_zone_id
  name    = "galera.${var.environment}.internal"
  type    = "A"
  ttl     = 60

  # Multi-value answer (round-robin)
  records = aws_instance.galera_node[*].private_ip
}
```

```hcl
# terraform/modules/ec2-mariadb-galera/outputs.tf

output "node_ids" {
  description = "IDs of Galera cluster nodes"
  value       = aws_instance.galera_node[*].id
}

output "node_private_ips" {
  description = "Private IPs of Galera nodes"
  value       = aws_instance.galera_node[*].private_ip
}

output "node_dns_names" {
  description = "DNS names of Galera nodes"
  value       = aws_route53_record.galera_nodes[*].fqdn
}

output "cluster_dns_name" {
  description = "DNS name for the cluster (round-robin)"
  value       = aws_route53_record.galera_cluster.fqdn
}

output "data_volume_ids" {
  description = "IDs of EBS data volumes"
  value       = aws_ebs_volume.galera_data[*].id
}

output "security_group_id" {
  description = "Security group ID for Galera cluster"
  value       = aws_security_group.galera.id
}

output "passwords_secret_arn" {
  description = "ARN of secrets containing passwords"
  value       = aws_secretsmanager_secret.galera_passwords.arn
}

# Output pour inventaire Ansible dynamique
output "ansible_inventory" {
  description = "Ansible inventory in YAML format"
  value = yamlencode({
    galera_nodes = {
      hosts = {
        for idx, instance in aws_instance.galera_node : 
        "galera-${idx + 1}" => {
          ansible_host = instance.private_ip
          node_index   = idx + 1
          is_primary   = idx == 0
        }
      }
      vars = {
        ansible_user                 = "ubuntu"
        ansible_ssh_private_key_file = "~/.ssh/${var.ssh_key_name}.pem"
        environment                  = var.environment
        cluster_name                 = "${var.environment}-galera"
        data_device                  = "/dev/xvdf"
      }
    }
  })
}
```

**Template user_data** :

```bash
# terraform/modules/ec2-mariadb-galera/templates/user_data.sh.tpl
#!/bin/bash
set -e

# Configuration hostname
hostnamectl set-hostname ${node_name}
echo "127.0.0.1 ${node_name}" >> /etc/hosts

# Update système
apt-get update
apt-get upgrade -y

# Installation CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
dpkg -i -E ./amazon-cloudwatch-agent.deb

# Installation AWS CLI
apt-get install -y awscli

# Installation packages de base
apt-get install -y \
    python3-pip \
    jq \
    nvme-cli

# Tag pour identification
aws ec2 create-tags \
    --resources $(ec2-metadata --instance-id | cut -d " " -f 2) \
    --tags Key=Provisioned,Value=true \
    --region ${aws_region}

# Signal à Terraform que l'instance est prête
echo "Instance ${node_name} provisioned at $(date)" > /var/log/provision-complete.log
```

### Inventaire Ansible dynamique depuis Terraform

**Script pour générer l'inventaire** :

```bash
#!/bin/bash
# scripts/generate-ansible-inventory.sh

cd terraform/environments/production

# Récupérer l'inventaire Ansible généré par Terraform
terraform output -json ansible_inventory | jq -r '.' > ../../ansible/inventories/production/terraform-generated.yml

echo "✅ Ansible inventory generated from Terraform outputs"
```

**Intégration dans le workflow** :

```bash
# 1. Provisionner l'infrastructure
terraform apply

# 2. Générer l'inventaire Ansible
./scripts/generate-ansible-inventory.sh

# 3. Configurer les serveurs avec Ansible
cd ansible
ansible-playbook -i inventories/production/terraform-generated.yml playbooks/galera-cluster.yml
```

---

## Orchestration Terraform + Ansible

### Workflow automatisé complet

```bash
#!/bin/bash
# scripts/deploy-mariadb.sh
# Script de déploiement complet : Terraform + Ansible

set -e

ENVIRONMENT=${1:-dev}
TERRAFORM_DIR="terraform/environments/${ENVIRONMENT}"
ANSIBLE_DIR="ansible"

echo "🚀 Starting MariaDB deployment for environment: ${ENVIRONMENT}"

# Étape 1: Validation Terraform
echo "📋 Step 1/5: Validating Terraform configuration..."
cd ${TERRAFORM_DIR}
terraform init -upgrade
terraform validate
terraform fmt -check

# Étape 2: Plan Terraform
echo "📋 Step 2/5: Planning infrastructure changes..."
terraform plan -out=tfplan

# Demander confirmation
read -p "Apply Terraform plan? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "❌ Deployment cancelled"
    exit 1
fi

# Étape 3: Apply Terraform
echo "🏗️  Step 3/5: Provisioning infrastructure..."
terraform apply tfplan

# Étape 4: Générer inventaire Ansible
echo "📝 Step 4/5: Generating Ansible inventory from Terraform outputs..."
cd -
terraform -chdir=${TERRAFORM_DIR} output -json ansible_inventory | \
    jq -r '.' > ${ANSIBLE_DIR}/inventories/${ENVIRONMENT}/terraform-generated.yml

# Attendre que les instances soient prêtes
echo "⏳ Waiting for instances to be ready..."
sleep 60

# Étape 5: Configuration Ansible
echo "⚙️  Step 5/5: Configuring MariaDB with Ansible..."
cd ${ANSIBLE_DIR}

# Validation syntaxe
ansible-playbook --syntax-check \
    -i inventories/${ENVIRONMENT}/terraform-generated.yml \
    playbooks/galera-cluster.yml

# Dry-run
ansible-playbook --check \
    -i inventories/${ENVIRONMENT}/terraform-generated.yml \
    playbooks/galera-cluster.yml

# Demander confirmation
read -p "Execute Ansible playbook? (yes/no): " confirm_ansible
if [ "$confirm_ansible" != "yes" ]; then
    echo "❌ Ansible execution cancelled"
    exit 1
fi

# Exécution réelle
ansible-playbook \
    -i inventories/${ENVIRONMENT}/terraform-generated.yml \
    playbooks/galera-cluster.yml

echo "✅ Deployment completed successfully!"
echo ""
echo "🔗 Cluster endpoint: $(terraform -chdir=../../${TERRAFORM_DIR} output -raw cluster_dns_name)"
```

### Makefile pour simplifier

```makefile
# Makefile

.PHONY: help plan apply destroy ansible validate test

ENVIRONMENT ?= dev
TERRAFORM_DIR = terraform/environments/$(ENVIRONMENT)
ANSIBLE_DIR = ansible

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

validate: ## Validate Terraform and Ansible
	@echo "Validating Terraform..."
	cd $(TERRAFORM_DIR) && terraform validate
	@echo "Validating Ansible..."
	cd $(ANSIBLE_DIR) && ansible-playbook --syntax-check playbooks/galera-cluster.yml

plan: ## Run Terraform plan
	cd $(TERRAFORM_DIR) && terraform plan -out=tfplan

apply: ## Apply Terraform and run Ansible
	@./scripts/deploy-mariadb.sh $(ENVIRONMENT)

destroy: ## Destroy infrastructure
	@echo "⚠️  WARNING: This will destroy all resources!"
	@read -p "Type '$(ENVIRONMENT)' to confirm: " confirm && [ "$$confirm" = "$(ENVIRONMENT)" ]
	cd $(TERRAFORM_DIR) && terraform destroy

ansible-only: ## Run only Ansible configuration
	cd $(ANSIBLE_DIR) && ansible-playbook \
		-i inventories/$(ENVIRONMENT)/terraform-generated.yml \
		playbooks/galera-cluster.yml

test: ## Run tests
	@echo "Running Terratest..."
	cd tests/terraform && go test -v -timeout 30m
	@echo "Running Ansible tests..."
	cd tests/ansible && pytest -v

clean: ## Clean temporary files
	find . -name "*.tfstate*" -delete
	find . -name ".terraform" -type d -exec rm -rf {} +
	find . -name "tfplan" -delete
```

**Utilisation** :

```bash
# Déploiement complet
make apply ENVIRONMENT=production

# Seulement Ansible
make ansible-only ENVIRONMENT=staging

# Tests
make test

# Nettoyage
make clean
```

---

## Patterns de déploiement

### 1. Blue-Green Deployment

**Concept** : Maintenir deux environnements identiques (Blue et Green), basculer le trafic instantanément.

```hcl
# terraform/blue-green/main.tf

variable "active_environment" {
  description = "Active environment (blue or green)"
  type        = string
  default     = "blue"
  
  validation {
    condition     = contains(["blue", "green"], var.active_environment)
    error_message = "Active environment must be 'blue' or 'green'"
  }
}

# Blue environment
module "mariadb_blue" {
  source = "../modules/rds-mariadb"
  
  environment         = "production-blue"
  vpc_id             = var.vpc_id
  private_subnet_ids = var.private_subnet_ids
  
  # ... reste de la configuration
}

# Green environment
module "mariadb_green" {
  source = "../modules/rds-mariadb"
  
  environment         = "production-green"
  vpc_id             = var.vpc_id
  private_subnet_ids = var.private_subnet_ids
  
  # ... reste de la configuration
}

# DNS pointant vers l'environnement actif
resource "aws_route53_record" "mariadb_active" {
  zone_id = var.route53_zone_id
  name    = "mariadb.production.internal"
  type    = "CNAME"
  ttl     = 60
  
  # Switch entre Blue et Green
  records = [
    var.active_environment == "blue" 
      ? module.mariadb_blue.db_instance_address 
      : module.mariadb_green.db_instance_address
  ]
}
```

**Workflow de déploiement** :

```bash
# Actuellement en Blue, déployer sur Green
terraform apply -var="active_environment=blue"

# Tester Green
# ... tests de validation ...

# Basculer vers Green
terraform apply -var="active_environment=green"

# Si problème, rollback immédiat vers Blue
terraform apply -var="active_environment=blue"

# Quand Green est stable, détruire Blue et le recréer pour le prochain cycle
```

### 2. Canary Deployment

**Concept** : Router progressivement le trafic vers la nouvelle version.

```hcl
# Utiliser weighted routing policy dans Route53
resource "aws_route53_record" "mariadb_weighted" {
  count = 2
  
  zone_id = var.route53_zone_id
  name    = "mariadb.production.internal"
  type    = "CNAME"
  ttl     = 60
  
  weighted_routing_policy {
    weight = count.index == 0 ? var.blue_weight : var.green_weight
  }
  
  set_identifier = count.index == 0 ? "blue" : "green"
  records        = [
    count.index == 0 
      ? module.mariadb_blue.db_instance_address 
      : module.mariadb_green.db_instance_address
  ]
}

# Variables pour contrôler les weights
variable "blue_weight" {
  description = "Traffic weight for blue (0-100)"
  type        = number
  default     = 90
}

variable "green_weight" {
  description = "Traffic weight for green (0-100)"
  type        = number
  default     = 10
}
```

**Progression** :

```bash
# Étape 1: 90% Blue, 10% Green (canary)
terraform apply -var="blue_weight=90" -var="green_weight=10"

# Étape 2: Si OK, 50/50
terraform apply -var="blue_weight=50" -var="green_weight=50"

# Étape 3: 100% Green
terraform apply -var="blue_weight=0" -var="green_weight=100"
```

### 3. Rolling Update (avec Ansible)

**Pour cluster Galera, mise à jour nœud par nœud** :

```yaml
# ansible/playbooks/rolling-update.yml
---
- name: Rolling update of Galera cluster
  hosts: galera_nodes
  serial: 1  # Un nœud à la fois
  max_fail_percentage: 0  # Arrêter si un nœud échoue
  
  pre_tasks:
    - name: Check cluster size before update
      shell: mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';" | grep wsrep_cluster_size | awk '{print $2}'
      register: cluster_size
      failed_when: cluster_size.stdout | int < 3
      
  tasks:
    - name: Stop MariaDB on current node
      service:
        name: mariadb
        state: stopped
        
    - name: Perform upgrade
      apt:
        name: mariadb-server
        state: latest
        update_cache: yes
        
    - name: Start MariaDB
      service:
        name: mariadb
        state: started
        
    - name: Wait for node to rejoin cluster
      shell: mysql -e "SHOW STATUS LIKE 'wsrep_local_state_comment';" | grep wsrep_local_state_comment | awk '{print $2}'
      register: node_state
      until: node_state.stdout == "Synced"
      retries: 30
      delay: 10
      
    - name: Verify cluster size after node rejoin
      shell: mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';" | grep wsrep_cluster_size | awk '{print $2}'
      register: cluster_size_after
      failed_when: cluster_size_after.stdout | int < 3
      
  post_tasks:
    - name: Final cluster health check
      shell: mysql -e "SHOW STATUS LIKE 'wsrep_ready';" | grep wsrep_ready | awk '{print $2}'
      register: wsrep_ready
      failed_when: wsrep_ready.stdout != "ON"
```

---

## ✅ Points clés à retenir

- **Terraform et Ansible sont complémentaires** : Terraform pour provisionner l'infrastructure, Ansible pour configurer
- **Terraform gère** : VPC, EC2/RDS, volumes, security groups, IAM, DNS
- **Ansible gère** : Installation packages, configuration MariaDB, clustering, utilisateurs
- **Inventaire dynamique** : Générer l'inventaire Ansible depuis les outputs Terraform
- **Modules réutilisables** : Créer des modules Terraform pour éviter la duplication
- **Remote state** : Toujours utiliser un backend distant (S3, GCS, Azure Storage)
- **Secrets management** : Ne jamais hardcoder de secrets, utiliser Secrets Manager/Vault
- **Multi-environnements** : Structure claire dev/staging/production
- **Blue-Green** : Pattern pour déploiements sans downtime
- **Rolling updates** : Mise à jour progressive des clusters avec Ansible
- **Automation** : Scripts et Makefiles pour orchestrer Terraform + Ansible
- **Testing** : Valider avec `terraform plan`, `ansible --check`, tests automatisés

💡 **Best practice** : Toujours exécuter `terraform plan` et `ansible-playbook --check` avant d'appliquer en production.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [📖 Terraform Modules](https://www.terraform.io/docs/language/modules/index.html)
- [📖 Ansible AWS Modules](https://docs.ansible.com/ansible/latest/collections/amazon/aws/)
- [📖 Ansible Dynamic Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html)

### Guides et best practices
- [📝 Terraform Best Practices for AWS](https://aws.amazon.com/blogs/apn/terraform-beyond-the-basics-with-aws/)
- [📝 Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [📝 Blue-Green Deployments with Terraform](https://www.hashicorp.com/blog/blue-green-deployments-with-terraform)

### Outils complémentaires
- [🔧 Terragrunt](https://terragrunt.gruntwork.io/) - DRY Terraform configurations
- [🔧 Atlantis](https://www.runatlantis.io/) - Terraform pull request automation
- [🔧 Ansible Tower/AWX](https://www.ansible.com/products/tower) - Ansible automation platform

---

## ➡️ Sections suivantes

**16.2.1 Ansible : Playbooks MariaDB** : Nous approfondirons la création de playbooks Ansible complets pour installer, configurer et maintenir MariaDB. Vous découvrirez des rôles réutilisables, le templating avancé et l'automatisation de tâches complexes.

**16.2.2 Terraform : Providers cloud** : Nous explorerons en détail les providers Terraform pour différents clouds (GCP, Azure) et comment gérer du multi-cloud pour MariaDB.

---

**MariaDB** : Version 11.8 LTS

⏭️ [Ansible : Playbooks MariaDB](/16-devops-automatisation/02.1-ansible-playbooks.md)

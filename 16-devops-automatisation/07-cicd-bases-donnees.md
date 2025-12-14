🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.7 CI/CD pour bases de données

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 6-7 heures  
> **Prérequis** : 
> - Sections 16.1-16.6 maîtrisées (IaC, Docker, Kubernetes, Operators)
> - Compréhension du CI/CD pour applications
> - Expérience avec Git workflows (GitFlow, trunk-based)
> - Connaissance d'au moins un outil CI/CD (GitHub Actions, GitLab CI, Jenkins)
> - Notions de testing automatisé

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les défis spécifiques du CI/CD pour bases de données stateful
- **Concevoir** un pipeline CI/CD complet pour MariaDB
- **Implémenter** des tests automatisés (unit, integration, performance)
- **Valider** les changements de schéma automatiquement
- **Déployer** des migrations de manière sûre et reproductible
- **Automatiser** les rollbacks en cas d'échec
- **Intégrer** monitoring et alerting dans le pipeline
- **Appliquer** les stratégies de déploiement (blue-green, canary)

---

## Introduction

### CI/CD pour applications vs bases de données

Le CI/CD pour bases de données présente des **défis uniques** comparé aux applications stateless :

```
┌──────────────────────────────────────────────────────────────┐
│           Applications Stateless (API, Web)                  │
├──────────────────────────────────────────────────────────────┤
│  ✅ Code immutable (nouveau deployment = nouveau code)       │
│  ✅ Rollback simple (retour à image précédente)              │
│  ✅ Tests isolés (pas d'état persistant)                     │
│  ✅ Deployment rapide (secondes)                             │
│  ✅ Environnements identiques faciles                        │
│  ✅ Blue-green trivial                                       │
└──────────────────────────────────────────────────────────────┘

                           VS

┌──────────────────────────────────────────────────────────────┐
│              Bases de données (MariaDB)                      │
├──────────────────────────────────────────────────────────────┤
│  ⚠️  État persistant (données critiques)                     │
│  ⚠️  Schéma évolutif (migrations forward-only)               │
│  ⚠️  Rollback complexe (données déjà modifiées)              │
│  ⚠️  Tests nécessitent données réalistes                     │
│  ⚠️  Deployment lent (migrations longues)                    │
│  ⚠️  Downtime potentiel (ALTER TABLE)                        │
│  ⚠️  Blue-green difficile (données dupliquées)               │
│  ⚠️  Interdépendances (app ↔ schéma)                         │
└──────────────────────────────────────────────────────────────┘
```

### Principes du database CI/CD

**5 principes fondamentaux** :

1. **Version Control Everything** 📝
   - Schéma DDL (CREATE TABLE, ALTER TABLE)
   - Migrations (scripts SQL versionnés)
   - Configuration (my.cnf)
   - Seed data (données de test)
   - Scripts admin (backup, restore)

2. **Automate Testing** 🤖
   - Unit tests (triggers, stored procedures)
   - Integration tests (avec vraie base)
   - Performance tests (slow queries, indexes)
   - Schema validation (lint, constraints)

3. **Incremental Migrations** 📊
   - Jamais modifier migration déjà appliquée
   - Forward-only (pas de rollback destructif)
   - Idempotent (safe à rejouer)
   - Versioned (V1, V2, V3...)

4. **Safe Deployments** 🛡️
   - Online schema changes (gh-ost, pt-osc)
   - Canary releases (test sur subset)
   - Rollback strategy (backward-compatible schema)
   - Monitoring during deployment

5. **Infrastructure as Code** 🏗️
   - Bases de données provisionnées automatiquement
   - Environnements éphémères pour tests
   - Configuration déclarative
   - Reproducible builds

---

## Architecture d'un pipeline CI/CD pour MariaDB

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────┐
│                    Database CI/CD Pipeline                      │
│                                                                 │
│  1️⃣  CODE CHANGES                                               │
│     Developer → Git commit → Push to branch                     │
│                                                                 │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  2️⃣  CONTINUOUS INTEGRATION (CI)                                │
│                                                                 │
│     ┌────────────────────────────────────────────────────────┐  │
│     │  Trigger: Pull Request or Commit                       │  │
│     └────────────────────────────────────────────────────────┘  │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Phase 1: Validation                                    │ │
│     │  ├── Lint SQL scripts                                   │ │
│     │  ├── Validate migration syntax                          │ │
│     │  ├── Check naming conventions                           │ │
│     │  └── Detect dangerous operations (DROP, TRUNCATE)       │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Phase 2: Build Test Database                           │ │
│     │  ├── Spin up MariaDB container (Testcontainers)         │ │
│     │  ├── Apply all migrations from scratch                  │ │
│     │  ├── Load seed data                                     │ │
│     │  └── Verify schema integrity                            │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Phase 3: Automated Tests                               │ │
│     │  ├── Unit tests (stored procedures, triggers)           │ │
│     │  ├── Integration tests (application queries)            │ │
│     │  ├── Performance tests (EXPLAIN, slow queries)          │ │
│     │  └── Data quality tests (constraints, referential)      │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Phase 4: Security Scanning                             │ │
│     │  ├── SQL injection patterns                             │ │
│     │  ├── Privilege escalation risks                         │ │
│     │  ├── Sensitive data exposure                            │ │
│     │  └── Compliance checks (GDPR, PCI-DSS)                  │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Phase 5: Generate Report                               │ │
│     │  ├── Migration plan (what will be applied)              │ │
│     │  ├── Impact analysis (tables affected, downtime)        │ │
│     │  ├── Rollback plan                                      │ │
│     │  └── Performance predictions                            │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│                     ✅ CI Passed                                │
│                     ❌ CI Failed → Block merge                  │
│                                                                 │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  3️⃣  CONTINUOUS DEPLOYMENT (CD)                                 │
│                                                                 │
│     ┌────────────────────────────────────────────────────────┐  │
│     │  Trigger: Merge to main branch                         │  │
│     └────────────────────────────────────────────────────────┘  │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Stage 1: Deploy to DEV                                 │ │
│     │  ├── Apply migrations automatically                     │ │
│     │  ├── Run smoke tests                                    │ │
│     │  └── Verify health                                      │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Stage 2: Deploy to STAGING                             │ │
│     │  ├── Backup before migration                            │ │
│     │  ├── Apply migrations (online if possible)              │ │
│     │  ├── Run integration tests                              │ │
│     │  ├── Run performance tests                              │ │
│     │  └── Manual approval gate (optional)                    │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│     ┌──────────────────────▼──────────────────────────────────┐ │
│     │  Stage 3: Deploy to PRODUCTION                          │ │
│     │  ├── Maintenance window notification                    │ │
│     │  ├── Full backup                                        │ │
│     │  ├── Apply migrations (gh-ost for zero-downtime)        │ │
│     │  ├── Monitoring intensified                             │ │
│     │  ├── Smoke tests on production                          │ │
│     │  └── Rollback if issues detected                        │ │
│     └─────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│     ┌──────────────────────────────────────────────────────────┐│
│     │  Post-Deployment                                         ││
│     │  ├── Update documentation                                ││
│     │  ├── Notify stakeholders                                 ││
│     │  ├── Monitor for 24-48h                                  ││
│     │  └── Archive migration artifacts                         ││
│     └──────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implémentation : GitHub Actions

### Structure du projet

```
mariadb-app/
├── .github/
│   └── workflows/
│       ├── ci-database.yml           # CI pipeline
│       ├── cd-database-staging.yml   # CD staging
│       └── cd-database-production.yml # CD production
│
├── database/
│   ├── migrations/
│   │   ├── V001__initial_schema.sql
│   │   ├── V002__add_users_table.sql
│   │   ├── V003__add_orders_index.sql
│   │   └── V004__add_audit_trigger.sql
│   │
│   ├── seeds/
│   │   ├── dev/
│   │   │   └── sample_data.sql
│   │   └── test/
│   │       └── test_data.sql
│   │
│   ├── procedures/
│   │   ├── create_order.sql
│   │   └── calculate_total.sql
│   │
│   ├── triggers/
│   │   └── audit_log_trigger.sql
│   │
│   └── tests/
│       ├── unit/
│       │   ├── test_create_order.sql
│       │   └── test_calculate_total.sql
│       ├── integration/
│       │   └── test_order_workflow.py
│       └── performance/
│           └── test_query_performance.py
│
├── config/
│   ├── my.cnf.dev
│   ├── my.cnf.staging
│   └── my.cnf.production
│
└── docker-compose.test.yml
```

### Pipeline CI complet

```yaml
# .github/workflows/ci-database.yml
name: Database CI

on:
  pull_request:
    branches: [main, develop]
    paths:
      - 'database/**'
      - 'config/**'
      - '.github/workflows/ci-database.yml'

env:
  MARIADB_VERSION: "11.8"
  MARIADB_ROOT_PASSWORD: "test-root-password"
  MARIADB_DATABASE: "testdb"

jobs:
  # Job 1: Validation statique
  validate:
    name: Validate SQL Scripts
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install SQLFluff (SQL Linter)
      run: |
        pip install sqlfluff
    
    - name: Lint SQL scripts
      run: |
        sqlfluff lint database/migrations/*.sql \
          --dialect mysql \
          --rules L001,L002,L003,L010,L019,L031
    
    - name: Check for dangerous operations
      run: |
        # Detect DROP, TRUNCATE without safeguards
        if grep -rE "DROP TABLE|TRUNCATE TABLE" database/migrations/ | grep -v "IF EXISTS"; then
          echo "❌ Dangerous DROP/TRUNCATE without IF EXISTS found"
          exit 1
        fi
        echo "✅ No dangerous operations found"
    
    - name: Validate migration naming
      run: |
        # Check V{version}__{description}.sql pattern
        for file in database/migrations/*.sql; do
          basename=$(basename "$file")
          if ! [[ $basename =~ ^V[0-9]{3}__.+\.sql$ ]]; then
            echo "❌ Invalid migration name: $basename"
            echo "Expected: V001__description.sql"
            exit 1
          fi
        done
        echo "✅ All migrations follow naming convention"

  # Job 2: Build et tests
  test:
    name: Build and Test Database
    runs-on: ubuntu-latest
    needs: validate
    
    services:
      mariadb:
        image: mariadb:11.8
        env:
          MYSQL_ROOT_PASSWORD: ${{ env.MARIADB_ROOT_PASSWORD }}
          MYSQL_DATABASE: ${{ env.MARIADB_DATABASE }}
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping --silent"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Wait for MariaDB to be ready
      run: |
        while ! mysqladmin ping -h127.0.0.1 -uroot -p${{ env.MARIADB_ROOT_PASSWORD }} --silent; do
          echo "Waiting for MariaDB..."
          sleep 2
        done
        echo "✅ MariaDB is ready"
    
    - name: Install Flyway
      run: |
        wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.4.1/flyway-commandline-10.4.1-linux-x64.tar.gz | tar xvz
        sudo ln -s $(pwd)/flyway-10.4.1/flyway /usr/local/bin
    
    - name: Apply migrations
      run: |
        flyway migrate \
          -url=jdbc:mysql://127.0.0.1:3306/${{ env.MARIADB_DATABASE }} \
          -user=root \
          -password=${{ env.MARIADB_ROOT_PASSWORD }} \
          -locations=filesystem:./database/migrations
    
    - name: Verify schema
      run: |
        # Check that all expected tables exist
        mysql -h127.0.0.1 -uroot -p${{ env.MARIADB_ROOT_PASSWORD }} ${{ env.MARIADB_DATABASE }} \
          -e "SHOW TABLES;"
    
    - name: Load test data
      run: |
        mysql -h127.0.0.1 -uroot -p${{ env.MARIADB_ROOT_PASSWORD }} ${{ env.MARIADB_DATABASE }} \
          < database/seeds/test/test_data.sql
    
    - name: Set up Python for tests
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install test dependencies
      run: |
        pip install pytest pymysql pytest-cov
    
    - name: Run unit tests
      run: |
        # Tests SQL (stored procedures, triggers)
        for test in database/tests/unit/*.sql; do
          echo "Running $test"
          mysql -h127.0.0.1 -uroot -p${{ env.MARIADB_ROOT_PASSWORD }} \
            ${{ env.MARIADB_DATABASE }} < "$test"
        done
    
    - name: Run integration tests
      env:
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_USER: root
        DB_PASSWORD: ${{ env.MARIADB_ROOT_PASSWORD }}
        DB_NAME: ${{ env.MARIADB_DATABASE }}
      run: |
        pytest database/tests/integration/ -v --cov
    
    - name: Run performance tests
      env:
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_USER: root
        DB_PASSWORD: ${{ env.MARIADB_ROOT_PASSWORD }}
        DB_NAME: ${{ env.MARIADB_DATABASE }}
      run: |
        pytest database/tests/performance/ -v
    
    - name: Generate migration report
      run: |
        echo "## Migration Impact Report" > migration-report.md
        echo "" >> migration-report.md
        echo "### Migrations to be applied:" >> migration-report.md
        flyway info \
          -url=jdbc:mysql://127.0.0.1:3306/${{ env.MARIADB_DATABASE }} \
          -user=root \
          -password=${{ env.MARIADB_ROOT_PASSWORD }} \
          -locations=filesystem:./database/migrations \
          >> migration-report.md
    
    - name: Upload report as artifact
      uses: actions/upload-artifact@v3
      with:
        name: migration-report
        path: migration-report.md
    
    - name: Comment PR with report
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const fs = require('fs');
          const report = fs.readFileSync('migration-report.md', 'utf8');
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: report
          });

  # Job 3: Security scanning
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: validate
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Run SQLMap patterns check
      run: |
        # Check for common SQL injection patterns
        if grep -rE "(\$_GET|\$_POST|\$_REQUEST|request\.args|request\.form)" database/; then
          echo "⚠️  Warning: Potential SQL injection risk detected"
          # Not failing, just warning
        fi
    
    - name: Check for hardcoded credentials
      run: |
        if grep -rEi "(password|passwd|pwd)\s*=\s*['\"][^'\"]+['\"]" database/; then
          echo "❌ Hardcoded credentials found"
          exit 1
        fi
        echo "✅ No hardcoded credentials"
    
    - name: Scan for sensitive data
      run: |
        # Check for SSN, credit card patterns
        if grep -rE "\b[0-9]{3}-[0-9]{2}-[0-9]{4}\b" database/seeds/; then
          echo "⚠️  Warning: Potential SSN found in seed data"
        fi
```

### Pipeline CD Staging

```yaml
# .github/workflows/cd-database-staging.yml
name: Database CD - Staging

on:
  push:
    branches: [develop]
    paths:
      - 'database/**'

env:
  MARIADB_VERSION: "11.8"

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Set up kubeconfig
      run: |
        echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Create backup before migration
      run: |
        kubectl exec -n staging mariadb-0 -- \
          mysqldump -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD_STAGING }} \
          --all-databases --single-transaction \
          | gzip > backup-pre-migration-$(date +%Y%m%d_%H%M%S).sql.gz
    
    - name: Upload backup to S3
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: eu-west-1
    
    - run: |
        aws s3 cp backup-pre-migration-*.sql.gz \
          s3://mariadb-backups/staging/pre-migration/
    
    - name: Install Flyway
      run: |
        wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.4.1/flyway-commandline-10.4.1-linux-x64.tar.gz | tar xvz
        sudo ln -s $(pwd)/flyway-10.4.1/flyway /usr/local/bin
    
    - name: Port-forward to staging MariaDB
      run: |
        kubectl port-forward -n staging svc/mariadb 3306:3306 &
        sleep 5
    
    - name: Apply migrations
      run: |
        flyway migrate \
          -url=jdbc:mysql://127.0.0.1:3306/myapp \
          -user=root \
          -password=${{ secrets.MARIADB_ROOT_PASSWORD_STAGING }} \
          -locations=filesystem:./database/migrations \
          -outOfOrder=false \
          -validateOnMigrate=true
    
    - name: Run smoke tests
      run: |
        mysql -h127.0.0.1 -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD_STAGING }} \
          -e "SELECT 'Smoke test passed' AS status;"
    
    - name: Verify schema integrity
      run: |
        # Check constraints
        mysql -h127.0.0.1 -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD_STAGING }} \
          -e "SELECT COUNT(*) FROM information_schema.TABLE_CONSTRAINTS;"
    
    - name: Notify Slack
      if: always()
      uses: slackapi/slack-github-action@v1
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
        payload: |
          {
            "text": "Staging database deployment: ${{ job.status }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Staging Database Deployment*\nStatus: ${{ job.status }}\nCommit: ${{ github.sha }}"
                }
              }
            ]
          }
```

### Pipeline CD Production

```yaml
# .github/workflows/cd-database-production.yml
name: Database CD - Production

on:
  push:
    branches: [main]
    paths:
      - 'database/**'

env:
  MARIADB_VERSION: "11.8"

jobs:
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: 
      name: production
      url: https://mariadb.company.com
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Manual approval gate
      uses: trstringer/manual-approval@v1
      with:
        secret: ${{ secrets.GITHUB_TOKEN }}
        approvers: dba-team,platform-team
        minimum-approvals: 2
        issue-title: "Production Database Migration Approval"
        issue-body: |
          **Production database migration requires approval**
          
          Commit: ${{ github.sha }}
          Author: ${{ github.actor }}
          
          Please review the migration plan and approve if safe to proceed.
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Set up kubeconfig
      run: |
        echo "${{ secrets.KUBE_CONFIG_PRODUCTION }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Notify start of maintenance
      uses: slackapi/slack-github-action@v1
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
        payload: |
          {
            "text": "🚨 Production database migration starting in 5 minutes",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Production Database Migration*\n⏱ Starting in 5 minutes\n📝 Commit: ${{ github.sha }}"
                }
              }
            ]
          }
    
    - name: Wait for maintenance window
      run: sleep 300
    
    - name: Create full backup (Mariabackup)
      run: |
        kubectl exec -n production mariadb-0 -- bash -c '
          mariabackup --backup \
            --user=root \
            --password=$MYSQL_ROOT_PASSWORD \
            --target-dir=/tmp/backup-$(date +%Y%m%d_%H%M%S)
        '
    
    - name: Upload backup to S3
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: eu-west-1
    
    - run: |
        kubectl exec -n production mariadb-0 -- \
          tar czf - /tmp/backup-* | \
        aws s3 cp - s3://mariadb-backups/production/pre-migration/backup-$(date +%Y%m%d_%H%M%S).tar.gz
    
    - name: Install gh-ost (online schema migration)
      run: |
        wget https://github.com/github/gh-ost/releases/download/v1.1.5/gh-ost-binary-linux-amd64-20230614094501.tar.gz
        tar -xzf gh-ost-binary-linux-amd64-20230614094501.tar.gz
        sudo mv gh-ost /usr/local/bin/
        chmod +x /usr/local/bin/gh-ost
    
    - name: Apply migrations with gh-ost
      run: |
        # For ALTER TABLE statements, use gh-ost for zero-downtime
        # For other migrations, use Flyway
        
        # Extract ALTER TABLE migrations
        for migration in database/migrations/*.sql; do
          if grep -q "ALTER TABLE" "$migration"; then
            echo "Applying $migration with gh-ost..."
            # Parse and apply with gh-ost
            # (simplified, real implementation more complex)
          else
            echo "Applying $migration with Flyway..."
            flyway migrate -url=... -locations="filesystem:$migration"
          fi
        done
    
    - name: Monitor during migration
      run: |
        # Check replication lag
        kubectl exec -n production mariadb-1 -- \
          mysql -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD }} \
          -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
        
        # Check connections
        kubectl exec -n production mariadb-0 -- \
          mysql -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD }} \
          -e "SHOW STATUS LIKE 'Threads_connected';"
    
    - name: Run smoke tests
      run: |
        kubectl port-forward -n production svc/mariadb 3306:3306 &
        sleep 5
        
        # Critical queries
        mysql -h127.0.0.1 -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD }} \
          -e "SELECT COUNT(*) FROM users;" \
          -e "SELECT COUNT(*) FROM orders WHERE created_at > NOW() - INTERVAL 1 DAY;"
    
    - name: Rollback if tests fail
      if: failure()
      run: |
        echo "🚨 Tests failed, initiating rollback..."
        
        # Restore from backup
        kubectl exec -n production mariadb-0 -- bash -c '
          # Stop MariaDB
          systemctl stop mariadb
          
          # Restore from backup
          mariabackup --prepare --target-dir=/tmp/backup-*
          mariabackup --copy-back --target-dir=/tmp/backup-*
          
          # Start MariaDB
          systemctl start mariadb
        '
    
    - name: Update documentation
      if: success()
      run: |
        # Auto-generate schema documentation
        kubectl exec -n production mariadb-0 -- \
          mysqldump -uroot -p${{ secrets.MARIADB_ROOT_PASSWORD }} \
          --no-data --skip-comments myapp \
          > docs/schema-$(date +%Y%m%d).sql
        
        git add docs/schema-*.sql
        git commit -m "docs: update schema documentation [skip ci]"
        git push
    
    - name: Notify completion
      if: always()
      uses: slackapi/slack-github-action@v1
      with:
        webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
        payload: |
          {
            "text": "Production database migration: ${{ job.status }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Production Database Migration Complete*\n✅ Status: ${{ job.status }}\n📝 Commit: ${{ github.sha }}\n👤 Author: ${{ github.actor }}"
                }
              }
            ]
          }
```

---

## Tests automatisés

### Tests unitaires (stored procedures)

```sql
-- database/tests/unit/test_create_order.sql

-- Test: create_order procedure
USE testdb;

-- Setup
DROP TABLE IF EXISTS test_orders;
CREATE TABLE test_orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Test case 1: Valid order creation
CALL create_order(123, 99.99, @order_id);
SELECT @order_id AS created_order_id;

SELECT 
    CASE 
        WHEN @order_id > 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS test_result,
    'Valid order creation' AS test_name;

-- Test case 2: Negative amount (should fail)
CALL create_order(123, -10.00, @order_id);

SELECT 
    CASE 
        WHEN @order_id IS NULL THEN 'PASS'
        ELSE 'FAIL'
    END AS test_result,
    'Negative amount rejection' AS test_name;

-- Cleanup
DROP TABLE test_orders;
```

### Tests d'intégration (Python)

```python
# database/tests/integration/test_order_workflow.py

import pytest
import pymysql
import os

@pytest.fixture
def db_connection():
    """Create database connection for tests"""
    conn = pymysql.connect(
        host=os.getenv('DB_HOST', 'localhost'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER', 'root'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME', 'testdb'),
        cursorclass=pymysql.cursors.DictCursor
    )
    yield conn
    conn.close()

def test_order_creation_workflow(db_connection):
    """Test complete order creation workflow"""
    cursor = db_connection.cursor()
    
    # Create customer
    cursor.execute("""
        INSERT INTO customers (email, name) 
        VALUES ('test@example.com', 'Test User')
    """)
    customer_id = cursor.lastrowid
    
    # Create order
    cursor.execute("""
        INSERT INTO orders (customer_id, total_amount) 
        VALUES (%s, %s)
    """, (customer_id, 99.99))
    order_id = cursor.lastrowid
    
    # Verify order
    cursor.execute("""
        SELECT o.*, c.email 
        FROM orders o
        JOIN customers c ON o.customer_id = c.id
        WHERE o.id = %s
    """, (order_id,))
    
    result = cursor.fetchone()
    
    assert result is not None
    assert result['customer_id'] == customer_id
    assert float(result['total_amount']) == 99.99
    assert result['email'] == 'test@example.com'
    
    db_connection.rollback()  # Cleanup

def test_referential_integrity(db_connection):
    """Test foreign key constraints"""
    cursor = db_connection.cursor()
    
    # Try to create order with non-existent customer
    with pytest.raises(pymysql.IntegrityError):
        cursor.execute("""
            INSERT INTO orders (customer_id, total_amount) 
            VALUES (999999, 50.00)
        """)
    
    db_connection.rollback()

def test_concurrent_order_creation(db_connection):
    """Test concurrent order creation (isolation)"""
    import threading
    
    results = []
    
    def create_order(customer_id, amount):
        conn = pymysql.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO orders (customer_id, total_amount) VALUES (%s, %s)",
            (customer_id, amount)
        )
        conn.commit()
        results.append(cursor.lastrowid)
        conn.close()
    
    # Create customer first
    cursor = db_connection.cursor()
    cursor.execute("INSERT INTO customers (email, name) VALUES ('test@example.com', 'Test')")
    customer_id = cursor.lastrowid
    db_connection.commit()
    
    # Create 10 orders concurrently
    threads = []
    for i in range(10):
        t = threading.Thread(target=create_order, args=(customer_id, 10.0 * i))
        threads.append(t)
        t.start()
    
    for t in threads:
        t.join()
    
    # Verify all orders created
    assert len(results) == 10
    assert len(set(results)) == 10  # All unique IDs
    
    db_connection.rollback()
```

### Tests de performance

```python
# database/tests/performance/test_query_performance.py

import pytest
import pymysql
import time
import os

@pytest.fixture
def db_connection():
    conn = pymysql.connect(
        host=os.getenv('DB_HOST', 'localhost'),
        user=os.getenv('DB_USER', 'root'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME', 'testdb')
    )
    yield conn
    conn.close()

def test_index_usage(db_connection):
    """Verify queries use indexes (EXPLAIN)"""
    cursor = db_connection.cursor()
    
    # Query that should use index
    cursor.execute("""
        EXPLAIN SELECT * FROM orders 
        WHERE customer_id = 123
    """)
    
    explain_result = cursor.fetchone()
    
    # Check that index is used (not full table scan)
    assert 'index' in str(explain_result).lower()
    assert 'ALL' not in explain_result  # Not full table scan

def test_query_performance(db_connection):
    """Test query execution time"""
    cursor = db_connection.cursor()
    
    # Create test data
    cursor.execute("""
        INSERT INTO orders (customer_id, total_amount)
        SELECT 
            FLOOR(1 + RAND() * 1000) as customer_id,
            ROUND(RAND() * 1000, 2) as total_amount
        FROM 
            (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) t1,
            (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) t2,
            (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) t3
    """)
    db_connection.commit()
    
    # Test query performance
    start_time = time.time()
    
    cursor.execute("""
        SELECT customer_id, COUNT(*) as order_count, SUM(total_amount) as total
        FROM orders
        GROUP BY customer_id
        HAVING order_count > 5
    """)
    
    results = cursor.fetchall()
    execution_time = time.time() - start_time
    
    # Assert query completes within threshold
    assert execution_time < 0.5, f"Query took {execution_time}s (threshold: 0.5s)"
    
    db_connection.rollback()

def test_slow_query_detection(db_connection):
    """Detect slow queries via performance_schema"""
    cursor = db_connection.cursor()
    
    # Enable performance schema
    cursor.execute("SET GLOBAL performance_schema = ON")
    
    # Run potentially slow query
    cursor.execute("""
        SELECT * FROM orders o
        JOIN customers c ON o.customer_id = c.id
        WHERE o.created_at > DATE_SUB(NOW(), INTERVAL 1 YEAR)
    """)
    
    # Check slow query log
    cursor.execute("""
        SELECT query_time, sql_text
        FROM performance_schema.events_statements_history
        WHERE sql_text LIKE '%orders%'
        ORDER BY query_time DESC
        LIMIT 1
    """)
    
    result = cursor.fetchone()
    
    if result and result[0] > 1.0:  # Slower than 1 second
        pytest.fail(f"Slow query detected: {result[1]} ({result[0]}s)")
```

---

## Stratégies de déploiement

### 1. Blue-Green pour bases de données

**Concept** : Maintenir deux bases identiques, basculer le trafic instantanément.

```
┌─────────────────────────────────────────────────────────────┐
│              Blue-Green Database Deployment                 │
│                                                             │
│  Phase 1: État initial                                      │
│  ┌────────────────┐                                         │
│  │  Application   │                                         │
│  └───────┬────────┘                                         │
│          │                                                  │
│          ▼                                                  │
│  ┌────────────────┐                                         │
│  │  Blue DB       │ ← ACTIVE (production)                   │
│  │  (v1 schema)   │                                         │
│  └────────────────┘                                         │
│  ┌────────────────┐                                         │
│  │  Green DB      │ ← IDLE                                  │
│  │  (empty)       │                                         │
│  └────────────────┘                                         │
│                                                             │
│  ────────────────────────────────────────────────────────── │
│                                                             │
│  Phase 2: Synchroniser données Blue → Green                 │
│  ┌────────────────┐                                         │
│  │  Blue DB       │ ← ACTIVE                                │
│  │  (v1 schema)   │                                         │
│  └───────┬────────┘                                         │
│          │ replicate                                        │
│          ▼                                                  │
│  ┌────────────────┐                                         │
│  │  Green DB      │                                         │
│  │  (v1 schema)   │                                         │
│  └────────────────┘                                         │
│                                                             │
│  ────────────────────────────────────────────────────────── │
│                                                             │
│  Phase 3: Appliquer migrations sur Green                    │
│  ┌────────────────┐                                         │
│  │  Blue DB       │ ← ACTIVE                                │
│  │  (v1 schema)   │                                         │
│  └────────────────┘                                         │
│  ┌────────────────┐                                         │
│  │  Green DB      │ ← Migrations v2 applied                 │
│  │  (v2 schema)   │                                         │
│  └────────────────┘                                         │
│                                                             │
│  ────────────────────────────────────────────────────────── │
│                                                             │
│  Phase 4: Basculer application vers Green                   │
│  ┌────────────────┐                                         │
│  │  Application   │                                         │
│  └───────┬────────┘                                         │
│          │ switch                                           │
│          ▼                                                  │
│  ┌────────────────┐                                         │
│  │  Green DB      │ ← ACTIVE                                │
│  │  (v2 schema)   │                                         │
│  └────────────────┘                                         │
│  ┌────────────────┐                                         │
│  │  Blue DB       │ ← IDLE (ready for rollback)             │
│  │  (v1 schema)   │                                         │
│  └────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

**Implémentation Kubernetes** :

```yaml
# Service switchable Blue/Green
apiVersion: v1
kind: Service
metadata:
  name: mariadb-active
spec:
  selector:
    app: mariadb
    version: blue  # Changer à 'green' pour basculer
  ports:
  - port: 3306

---
# Blue database
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-blue
spec:
  selector:
    matchLabels:
      app: mariadb
      version: blue
  # ...

---
# Green database
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-green
spec:
  selector:
    matchLabels:
      app: mariadb
      version: green
  # ...
```

**Script de basculement** :

```bash
#!/bin/bash
# blue-green-switch.sh

CURRENT=$(kubectl get service mariadb-active -o jsonpath='{.spec.selector.version}')

if [ "$CURRENT" = "blue" ]; then
  NEW="green"
else
  NEW="blue"
fi

echo "Switching from $CURRENT to $NEW..."

# Update service selector
kubectl patch service mariadb-active -p "{\"spec\":{\"selector\":{\"version\":\"$NEW\"}}}"

echo "✅ Switched to $NEW"
```

### 2. Canary releases pour schéma

**Concept** : Tester changements de schéma sur subset d'utilisateurs.

```yaml
# Canary deployment avec Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: mariadb-canary
spec:
  replicas: 5
  
  strategy:
    canary:
      steps:
      - setWeight: 20  # 20% vers canary
      - pause: {duration: 10m}
      - setWeight: 40
      - pause: {duration: 10m}
      - setWeight: 60
      - pause: {duration: 10m}
      - setWeight: 80
      - pause: {duration: 10m}
      
      # Rollback si métrique dépasse seuil
      analysis:
        templates:
        - templateName: error-rate-analysis
        startingStep: 2
        args:
        - name: service-name
          value: mariadb-canary
```

---

## ✅ Points clés à retenir

- **Database CI/CD ≠ Application CI/CD** : État persistant, migrations forward-only, rollback complexe
- **Version control tout** : Schéma, migrations, config, seeds, tests
- **Tests automatisés essentiels** : Unit, integration, performance, security
- **Migrations incrémentales** : V1, V2, V3... jamais modifier existant
- **Backups systématiques** : Avant chaque deployment production
- **Online schema changes** : gh-ost, pt-online-schema-change pour zero-downtime
- **Monitoring intensif** : Pendant et après deployment
- **Rollback strategy** : Backup + backward-compatible schema
- **Multi-stage pipeline** : Dev → Staging → Production
- **Approval gates** : Humain valide production

💡 **Best practice** : Database changes sont plus risqués que code changes → plus de validations, plus de tests, plus de prudence.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 GitHub Actions](https://docs.github.com/en/actions)
- [📖 GitLab CI/CD](https://docs.gitlab.com/ee/ci/)
- [📖 Flyway](https://flywaydb.org/documentation/)

### Outils
- [🔧 Testcontainers](https://www.testcontainers.org/)
- [🔧 SQLFluff (SQL Linter)](https://www.sqlfluff.com/)
- [🔧 gh-ost](https://github.com/github/gh-ost)

---

## ➡️ Section suivante

**16.8 Gestion des migrations** : Nous approfondirons les outils de migration (Flyway, Liquibase, gh-ost, pt-online-schema-change) avec exemples détaillés et comparatifs.

---

**MariaDB** : Version 11.8 LTS

⏭️ [Gestion des migrations](/16-devops-automatisation/08-gestion-migrations.md)

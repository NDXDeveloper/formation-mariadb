🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6 Automatisation des sauvegardes

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Sections 12.1-12.5, Linux scripting, Cron, Systemd, Kubernetes (optionnel)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Concevoir** une infrastructure de backup entièrement automatisée
- **Implémenter** des scripts robustes avec gestion d'erreurs et retry logic
- **Orchestrer** les sauvegardes avec cron, systemd timers ou Kubernetes CronJobs
- **Monitorer** l'état des backups avec Prometheus/Grafana
- **Notifier** automatiquement les équipes en cas de succès/échec
- **Gérer** la rétention et la purge automatique selon les politiques
- **Valider** automatiquement l'intégrité des backups
- **Documenter** les processus pour garantir la maintenabilité

---

## Introduction

L'automatisation des sauvegardes est **non négociable** en environnement de production. Une sauvegarde manuelle est :
- Sujette à l'oubli humain
- Incohérente dans son exécution
- Non vérifiable systématiquement
- Impossible à scaler

### Le coût de l'absence d'automatisation

```
┌──────────────────────────────────────────────────┐
│     Impact d'une sauvegarde manuelle ratée       │
├──────────────────────────────────────────────────┤
│                                                  │
│  Scénario : DBA oublie le backup du vendredi     │
│                                                  │
│  Vendredi 23h : Backup non effectué              │
│  Samedi 14h  : Panne disque serveur production   │
│  Samedi 15h  : Tentative restauration            │
│  Samedi 15h30: Découverte backup manquant ⚠️     │
│                                                  │
│  Résultat :                                      │
│  ├─ Perte de données : 48 heures                 │
│  ├─ Downtime : 12+ heures                        │
│  ├─ Coût business : 500K€+                       │
│  └─ Réputation : Dégradée                        │
│                                                  │
└──────────────────────────────────────────────────┘
```

💡 **Principe clé** : *"Les backups ne sont fiables que s'ils sont automatisés, monitorés et testés régulièrement"*.

### Statistiques de l'industrie

D'après Veeam 2024 Data Protection Report :

```
Organisations avec backups automatisés : 87%
  ├─ Monitoring actif des backups : 62%
  ├─ Validation automatique : 41%
  └─ Tests réguliers : 23%

Causes d'échec de backup automatisé :
  ├─ Espace disque insuffisant : 38%
  ├─ Erreur script non détectée : 27%
  ├─ Timeout/performance : 18%
  └─ Permissions/accès : 17%
```

---

## Architecture d'automatisation complète

### Vue d'ensemble

```
┌───────────────────────────────────────────────────────┐
│         Infrastructure de Backup Automatisée          │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Orchestration                                        │
│  ├─ Cron / Systemd Timers / K8s CronJobs              │
│  └─ Scheduling intelligent (fenêtres de maintenance)  │
│                                                       │
│  Exécution                                            │
│  ├─ Scripts bash avec gestion erreurs                 │
│  ├─ Retry logic (3 tentatives)                        │
│  └─ Logging structuré                                 │
│                                                       │
│  Stockage                                             │
│  ├─ Local : Backup primaire                           │
│  ├─ S3/Cloud : Backup secondaire                      │
│  └─ Tape/Glacier : Archivage long terme               │
│                                                       │
│  Validation                                           │
│  ├─ Checksums (MD5/SHA256)                            │
│  ├─ Test restauration automatique                     │
│  └─ Cohérence données                                 │
│                                                       │
│  Rétention                                            │
│  ├─ Purge automatique (GFS)                           │
│  ├─ Rotation intelligente                             │
│  └─ Archivage compliance                              │
│                                                       │
│  Monitoring                                           │
│  ├─ Prometheus metrics                                │
│  ├─ Grafana dashboards                                │
│  └─ Alerting (PagerDuty, Slack, Email)                │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

## Scripts d'automatisation robustes

### Script de backup complet production-ready

```bash
#!/bin/bash
# mariadb_backup_automated.sh
# Version: 2.0
# Description: Production-grade automated backup with error handling

set -euo pipefail  # Exit on error, undefined var, pipe failure
IFS=$'\n\t'        # Safe Internal Field Separator

#─────────────────────────────────────────────────────────
# Configuration
#─────────────────────────────────────────────────────────
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
readonly TIMESTAMP=$(date +%Y%m%d_%H%M%S)
readonly DATE=$(date +%Y%m%d)

# Backup configuration
readonly BACKUP_TYPE="${1:-full}"  # full, incremental
readonly BACKUP_BASE_DIR="/backups/mariadb"
readonly BACKUP_DIR="${BACKUP_BASE_DIR}/${BACKUP_TYPE}/${DATE}"
readonly LOG_DIR="/var/log/mariadb_backup"
readonly LOG_FILE="${LOG_DIR}/backup_${TIMESTAMP}.log"

# Database configuration
readonly DB_USER="backup_user"
readonly DB_PASSWORD_FILE="/etc/mysql/backup.password"
readonly DB_HOST="localhost"
readonly DB_PORT="3306"

# Retention policy (days)
readonly RETENTION_FULL=30
readonly RETENTION_INCREMENTAL=7

# S3 configuration
readonly S3_BUCKET="s3://my-database-backups/mariadb"
readonly S3_ENABLED=true

# Notification configuration
readonly SLACK_WEBHOOK_URL="${SLACK_WEBHOOK_URL:-}"
readonly EMAIL_RECIPIENT="dba@example.com"
readonly ALERT_ON_SUCCESS=false
readonly ALERT_ON_FAILURE=true

# Validation
readonly VALIDATE_BACKUP=true
readonly CHECKSUM_ALGO="sha256sum"

#─────────────────────────────────────────────────────────
# Functions
#─────────────────────────────────────────────────────────

log() {
  local level="$1"
  shift
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

log_info() { log "INFO" "$@"; }
log_warn() { log "WARN" "$@"; }
log_error() { log "ERROR" "$@"; }
log_success() { log "SUCCESS" "$@"; }

check_prerequisites() {
  log_info "Checking prerequisites..."
  
  # Check commands
  local required_commands=("mariabackup" "gzip" "aws")
  for cmd in "${required_commands[@]}"; do
    if ! command -v "$cmd" &>/dev/null; then
      log_error "Required command not found: $cmd"
      return 1
    fi
  done
  
  # Check password file
  if [[ ! -f "$DB_PASSWORD_FILE" ]]; then
    log_error "Password file not found: $DB_PASSWORD_FILE"
    return 1
  fi
  
  # Check disk space (require 1.5x DB size)
  local db_size=$(mariadb -h"$DB_HOST" -u"$DB_USER" \
    -p"$(cat $DB_PASSWORD_FILE)" -Nse \
    "SELECT SUM(data_length + index_length) 
     FROM information_schema.TABLES;" 2>/dev/null)
  
  local required_space=$((db_size * 3 / 2))
  local available_space=$(df -B1 "$BACKUP_BASE_DIR" | awk 'NR==2 {print $4}')
  
  if ((available_space < required_space)); then
    log_error "Insufficient disk space. Required: $required_space, Available: $available_space"
    return 1
  fi
  
  log_info "Prerequisites check passed"
  return 0
}

perform_backup() {
  log_info "Starting $BACKUP_TYPE backup to $BACKUP_DIR"
  
  mkdir -p "$BACKUP_DIR"
  
  local backup_cmd="mariabackup --backup \
    --host=$DB_HOST \
    --port=$DB_PORT \
    --user=$DB_USER \
    --password=$(cat $DB_PASSWORD_FILE) \
    --target-dir=$BACKUP_DIR \
    --compress \
    --compress-threads=4 \
    --parallel=4"
  
  # Add incremental options if needed
  if [[ "$BACKUP_TYPE" == "incremental" ]]; then
    local last_full=$(find "$BACKUP_BASE_DIR/full" -maxdepth 1 -type d | sort -r | head -2 | tail -1)
    if [[ -z "$last_full" ]]; then
      log_error "No full backup found for incremental backup"
      return 1
    fi
    backup_cmd="$backup_cmd --incremental-basedir=$last_full"
    log_info "Incremental backup based on: $last_full"
  fi
  
  # Execute backup with timeout (4 hours)
  if timeout 14400 $backup_cmd &>> "$LOG_FILE"; then
    log_success "Backup completed successfully"
    return 0
  else
    local exit_code=$?
    log_error "Backup failed with exit code: $exit_code"
    return 1
  fi
}

validate_backup() {
  if [[ "$VALIDATE_BACKUP" != "true" ]]; then
    log_info "Validation skipped (disabled)"
    return 0
  fi
  
  log_info "Validating backup integrity..."
  
  # Check backup metadata
  if [[ ! -f "$BACKUP_DIR/xtrabackup_checkpoints" ]]; then
    log_error "Backup metadata missing: xtrabackup_checkpoints"
    return 1
  fi
  
  # Generate checksum
  local checksum_file="${BACKUP_DIR}.${CHECKSUM_ALGO%sum}"
  find "$BACKUP_DIR" -type f -exec $CHECKSUM_ALGO {} \; > "$checksum_file"
  log_info "Checksum file created: $checksum_file"
  
  # Verify compressed files
  local compressed_files=$(find "$BACKUP_DIR" -name "*.qp" -o -name "*.gz")
  if [[ -n "$compressed_files" ]]; then
    while IFS= read -r file; do
      if ! gzip -t "$file" 2>/dev/null && ! qpress -t "$file" 2>/dev/null; then
        log_error "Corrupted compressed file: $file"
        return 1
      fi
    done <<< "$compressed_files"
  fi
  
  log_success "Backup validation passed"
  return 0
}

upload_to_s3() {
  if [[ "$S3_ENABLED" != "true" ]]; then
    log_info "S3 upload skipped (disabled)"
    return 0
  fi
  
  log_info "Uploading backup to S3..."
  
  local s3_path="${S3_BUCKET}/${BACKUP_TYPE}/${DATE}/"
  
  # Upload with retry logic
  local max_retries=3
  local retry_count=0
  
  while ((retry_count < max_retries)); do
    if aws s3 sync "$BACKUP_DIR" "$s3_path" \
        --storage-class INTELLIGENT_TIERING \
        --sse AES256 \
        --only-show-errors 2>> "$LOG_FILE"; then
      log_success "Backup uploaded to S3: $s3_path"
      return 0
    else
      ((retry_count++))
      log_warn "S3 upload attempt $retry_count failed, retrying..."
      sleep $((retry_count * 10))
    fi
  done
  
  log_error "S3 upload failed after $max_retries attempts"
  return 1
}

cleanup_old_backups() {
  log_info "Cleaning up old backups..."
  
  # Full backups older than RETENTION_FULL days
  find "$BACKUP_BASE_DIR/full" -maxdepth 1 -type d -mtime +$RETENTION_FULL -exec rm -rf {} \;
  log_info "Removed full backups older than $RETENTION_FULL days"
  
  # Incremental backups older than RETENTION_INCREMENTAL days
  find "$BACKUP_BASE_DIR/incremental" -maxdepth 1 -type d -mtime +$RETENTION_INCREMENTAL -exec rm -rf {} \;
  log_info "Removed incremental backups older than $RETENTION_INCREMENTAL days"
  
  # S3 cleanup (use lifecycle policies instead)
  log_info "S3 cleanup handled by lifecycle policies"
}

send_notification() {
  local status="$1"
  local message="$2"
  
  # Determine if we should send notification
  if [[ "$status" == "success" && "$ALERT_ON_SUCCESS" != "true" ]]; then
    return 0
  fi
  
  if [[ "$status" == "failure" && "$ALERT_ON_FAILURE" != "true" ]]; then
    return 0
  fi
  
  # Email notification
  if command -v mail &>/dev/null; then
    echo "$message" | mail -s "MariaDB Backup $status: $BACKUP_TYPE" "$EMAIL_RECIPIENT"
  fi
  
  # Slack notification
  if [[ -n "$SLACK_WEBHOOK_URL" ]]; then
    local color="good"
    [[ "$status" == "failure" ]] && color="danger"
    
    curl -X POST "$SLACK_WEBHOOK_URL" \
      -H 'Content-Type: application/json' \
      -d "{
        \"attachments\": [{
          \"color\": \"$color\",
          \"title\": \"MariaDB Backup $status\",
          \"text\": \"$message\",
          \"fields\": [
            {\"title\": \"Type\", \"value\": \"$BACKUP_TYPE\", \"short\": true},
            {\"title\": \"Date\", \"value\": \"$DATE\", \"short\": true},
            {\"title\": \"Host\", \"value\": \"$(hostname)\", \"short\": true}
          ],
          \"footer\": \"MariaDB Backup System\",
          \"ts\": $(date +%s)
        }]
      }" 2>/dev/null
  fi
  
  log_info "Notification sent: $status"
}

generate_metrics() {
  log_info "Generating metrics..."
  
  local metrics_file="/var/lib/node_exporter/textfile_collector/mariadb_backup.prom"
  mkdir -p "$(dirname "$metrics_file")"
  
  local backup_size=$(du -sb "$BACKUP_DIR" | awk '{print $1}')
  local backup_duration=$SECONDS
  local backup_success=1
  
  cat > "$metrics_file" << EOF
# HELP mariadb_backup_last_success_timestamp Unix timestamp of last successful backup
# TYPE mariadb_backup_last_success_timestamp gauge
mariadb_backup_last_success_timestamp{type="$BACKUP_TYPE"} $(date +%s)

# HELP mariadb_backup_size_bytes Size of the backup in bytes
# TYPE mariadb_backup_size_bytes gauge
mariadb_backup_size_bytes{type="$BACKUP_TYPE"} $backup_size

# HELP mariadb_backup_duration_seconds Duration of backup in seconds
# TYPE mariadb_backup_duration_seconds gauge
mariadb_backup_duration_seconds{type="$BACKUP_TYPE"} $backup_duration

# HELP mariadb_backup_success Boolean indicator of backup success
# TYPE mariadb_backup_success gauge
mariadb_backup_success{type="$BACKUP_TYPE"} $backup_success
EOF
  
  log_info "Metrics written to $metrics_file"
}

#─────────────────────────────────────────────────────────
# Main
#─────────────────────────────────────────────────────────

main() {
  mkdir -p "$LOG_DIR"
  
  log_info "========================================="
  log_info "MariaDB Backup Started"
  log_info "Type: $BACKUP_TYPE"
  log_info "Date: $DATE"
  log_info "========================================="
  
  local overall_status="success"
  local error_message=""
  
  # Execute backup workflow
  if ! check_prerequisites; then
    overall_status="failure"
    error_message="Prerequisites check failed"
  elif ! perform_backup; then
    overall_status="failure"
    error_message="Backup execution failed"
  elif ! validate_backup; then
    overall_status="failure"
    error_message="Backup validation failed"
  elif ! upload_to_s3; then
    overall_status="failure"
    error_message="S3 upload failed"
  else
    cleanup_old_backups
    generate_metrics
  fi
  
  # Send notification
  if [[ "$overall_status" == "success" ]]; then
    local size=$(du -sh "$BACKUP_DIR" | cut -f1)
    send_notification "success" "Backup completed successfully. Size: $size"
    log_success "========================================="
    log_success "Backup workflow completed successfully"
    log_success "========================================="
  else
    send_notification "failure" "Backup failed: $error_message"
    log_error "========================================="
    log_error "Backup workflow failed: $error_message"
    log_error "========================================="
    exit 1
  fi
}

# Trap errors
trap 'log_error "Script interrupted"; send_notification "failure" "Backup interrupted"' INT TERM

# Execute
main "$@"
```

### Script pour binary logs

```bash
#!/bin/bash
# mariadb_binlog_archive.sh
# Archives binary logs with rotation and S3 upload

set -euo pipefail

#─────────────────────────────────────────────────────────
# Configuration
#─────────────────────────────────────────────────────────
readonly BINLOG_DIR="/var/log/mysql"
readonly ARCHIVE_DIR="/backups/binlogs/$(date +%Y%m%d)"
readonly S3_BUCKET="s3://my-database-backups/binlogs"
readonly LOG_FILE="/var/log/binlog_archive.log"
readonly RETENTION_DAYS=7

#─────────────────────────────────────────────────────────
# Functions
#─────────────────────────────────────────────────────────

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

archive_binlogs() {
  log "Starting binlog archival..."
  
  mkdir -p "$ARCHIVE_DIR"
  
  # Flush to create new binlog
  mariadb -e "FLUSH BINARY LOGS;"
  
  # Get current binlog
  local current_binlog=$(mariadb -Nse "SHOW MASTER STATUS;" | awk '{print $1}')
  
  # Archive all except current
  for binlog in "$BINLOG_DIR"/mariadb-bin.[0-9]*; do
    local binlog_name=$(basename "$binlog")
    
    # Skip current binlog
    [[ "$binlog_name" == "$current_binlog" ]] && continue
    
    # Skip if already archived
    [[ -f "$ARCHIVE_DIR/$binlog_name.gz" ]] && continue
    
    # Compress and archive
    gzip -c "$binlog" > "$ARCHIVE_DIR/$binlog_name.gz"
    log "Archived: $binlog_name"
  done
}

upload_to_s3() {
  log "Uploading to S3..."
  
  aws s3 sync "$ARCHIVE_DIR" "$S3_BUCKET/$(date +%Y%m%d)/" \
    --storage-class INTELLIGENT_TIERING \
    --sse AES256 \
    --only-show-errors
  
  log "S3 upload completed"
}

cleanup_old_archives() {
  log "Cleaning up old archives..."
  
  # Local cleanup
  find "$(dirname "$ARCHIVE_DIR")" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
  
  log "Cleanup completed"
}

main() {
  archive_binlogs
  upload_to_s3
  cleanup_old_archives
  
  log "Binlog archival completed successfully"
}

main "$@"
```

---

## Orchestration avec Cron

### Configuration cron production

```cron
# /etc/cron.d/mariadb-backups

# Environment
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=dba@example.com

# Full backup daily at 2:00 AM
0 2 * * * backup_user /usr/local/bin/mariadb_backup_automated.sh full >> /var/log/cron_backup.log 2>&1

# Incremental backup every 6 hours (except during full backup)
0 8,14,20 * * * backup_user /usr/local/bin/mariadb_backup_automated.sh incremental >> /var/log/cron_backup.log 2>&1

# Binlog archival every 15 minutes
*/15 * * * * backup_user /usr/local/bin/mariadb_binlog_archive.sh >> /var/log/cron_binlog.log 2>&1

# Test restore monthly (1st Sunday at 3 AM)
0 3 * * 0 backup_user [ $(date +\%d) -le 7 ] && /usr/local/bin/automated_restore_test.sh >> /var/log/cron_restore_test.log 2>&1

# Cleanup old backups daily at 4 AM
0 4 * * * backup_user /usr/local/bin/cleanup_old_backups.sh >> /var/log/cron_cleanup.log 2>&1
```

### Validation de la configuration cron

```bash
#!/bin/bash
# validate_cron_config.sh

echo "Checking cron configuration..."

# Check syntax
if ! crontab -l -u backup_user &>/dev/null; then
  echo "❌ Crontab syntax error"
  exit 1
fi

# Check script permissions
for script in /usr/local/bin/mariadb_backup_automated.sh \
              /usr/local/bin/mariadb_binlog_archive.sh \
              /usr/local/bin/automated_restore_test.sh; do
  if [[ ! -x "$script" ]]; then
    echo "❌ Script not executable: $script"
    exit 1
  fi
done

# Check log directories
for dir in /var/log/mariadb_backup /backups/mariadb; do
  if [[ ! -d "$dir" ]] || [[ ! -w "$dir" ]]; then
    echo "❌ Directory not writable: $dir"
    exit 1
  fi
done

echo "✅ Cron configuration is valid"
```

---

## Orchestration avec Systemd Timers

Alternative moderne à cron, plus robuste et intégré à systemd.

### Service de backup

```ini
# /etc/systemd/system/mariadb-backup@.service

[Unit]
Description=MariaDB %i Backup
After=mariadb.service
Requires=mariadb.service

[Service]
Type=oneshot
User=backup_user
Group=backup_user

# Environment
Environment="BACKUP_TYPE=%i"

# Execute backup script
ExecStart=/usr/local/bin/mariadb_backup_automated.sh %i

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=mariadb-backup

# Resource limits
CPUQuota=50%
IOWeight=50

# Timeouts
TimeoutStartSec=4h

[Install]
WantedBy=multi-user.target
```

### Timers

```ini
# /etc/systemd/system/mariadb-backup-full.timer

[Unit]
Description=Daily MariaDB Full Backup
Requires=mariadb-backup@full.service

[Timer]
# Daily at 2:00 AM
OnCalendar=daily
OnCalendar=*-*-* 02:00:00

# Randomize start time by 30 minutes to avoid spike
RandomizedDelaySec=30min

# Persistent (run on boot if missed)
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/mariadb-backup-incremental.timer

[Unit]
Description=MariaDB Incremental Backup (every 6h)
Requires=mariadb-backup@incremental.service

[Timer]
# Every 6 hours
OnCalendar=*-*-* 08,14,20:00:00

RandomizedDelaySec=10min
Persistent=true

[Install]
WantedBy=timers.target
```

### Activation

```bash
# Reload systemd
systemctl daemon-reload

# Enable and start timers
systemctl enable mariadb-backup-full.timer
systemctl enable mariadb-backup-incremental.timer

systemctl start mariadb-backup-full.timer
systemctl start mariadb-backup-incremental.timer

# Check status
systemctl status mariadb-backup-full.timer
systemctl list-timers mariadb-backup*

# View logs
journalctl -u mariadb-backup@full.service -f
```

---

## Orchestration Kubernetes

### CronJob pour backup

```yaml
# mariadb-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mariadb-full-backup
  namespace: databases
spec:
  # Daily at 2:00 AM UTC
  schedule: "0 2 * * *"
  
  # Keep last 3 successful jobs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  
  # Don't run if previous job still running
  concurrencyPolicy: Forbid
  
  jobTemplate:
    spec:
      # Retry on failure
      backoffLimit: 3
      
      # Timeout after 4 hours
      activeDeadlineSeconds: 14400
      
      template:
        metadata:
          labels:
            app: mariadb-backup
            type: full
        spec:
          restartPolicy: OnFailure
          
          # Service account with S3 access (IRSA)
          serviceAccountName: mariadb-backup
          
          containers:
          - name: backup
            image: mariadb:11.8
            
            command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              set -euo pipefail
              
              BACKUP_DIR="/backup/full-$(date +%Y%m%d_%H%M%S)"
              S3_BUCKET="s3://my-database-backups/k8s/mariadb"
              
              echo "Starting backup to $BACKUP_DIR"
              
              # Perform backup
              mariabackup --backup \
                --host=mariadb-primary.databases.svc.cluster.local \
                --port=3306 \
                --user=$BACKUP_USER \
                --password=$BACKUP_PASSWORD \
                --target-dir=$BACKUP_DIR \
                --compress \
                --compress-threads=4
              
              # Upload to S3
              echo "Uploading to S3..."
              aws s3 sync $BACKUP_DIR $S3_BUCKET/full/$(date +%Y%m%d)/ \
                --storage-class INTELLIGENT_TIERING \
                --sse AES256
              
              echo "Backup completed successfully"
            
            env:
            - name: BACKUP_USER
              valueFrom:
                secretKeyRef:
                  name: mariadb-backup-credentials
                  key: username
            - name: BACKUP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-backup-credentials
                  key: password
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            
            resources:
              requests:
                cpu: 500m
                memory: 2Gi
              limits:
                cpu: 2000m
                memory: 4Gi
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mariadb-backup-pvc
---
# Incremental backup every 6 hours
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mariadb-incremental-backup
  namespace: databases
spec:
  schedule: "0 */6 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 7200
      
      template:
        metadata:
          labels:
            app: mariadb-backup
            type: incremental
        spec:
          restartPolicy: OnFailure
          serviceAccountName: mariadb-backup
          
          containers:
          - name: backup
            image: mariadb:11.8
            
            command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              set -euo pipefail
              
              BACKUP_DIR="/backup/inc-$(date +%Y%m%d_%H%M%S)"
              S3_BUCKET="s3://my-database-backups/k8s/mariadb"
              
              # Find last full backup
              LAST_FULL=$(aws s3 ls $S3_BUCKET/full/ | tail -1 | awk '{print $2}')
              
              if [ -z "$LAST_FULL" ]; then
                echo "No full backup found, skipping incremental"
                exit 0
              fi
              
              # Download last full to use as base
              aws s3 sync $S3_BUCKET/full/$LAST_FULL /backup/last_full/
              
              # Perform incremental backup
              mariabackup --backup \
                --host=mariadb-primary.databases.svc.cluster.local \
                --user=$BACKUP_USER \
                --password=$BACKUP_PASSWORD \
                --target-dir=$BACKUP_DIR \
                --incremental-basedir=/backup/last_full \
                --compress
              
              # Upload to S3
              aws s3 sync $BACKUP_DIR $S3_BUCKET/incremental/$(date +%Y%m%d_%H%M)/ \
                --storage-class INTELLIGENT_TIERING \
                --sse AES256
              
              echo "Incremental backup completed"
            
            env:
            - name: BACKUP_USER
              valueFrom:
                secretKeyRef:
                  name: mariadb-backup-credentials
                  key: username
            - name: BACKUP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-backup-credentials
                  key: password
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            
            resources:
              requests:
                cpu: 250m
                memory: 1Gi
              limits:
                cpu: 1000m
                memory: 2Gi
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mariadb-backup-pvc
```

### Monitoring des CronJobs

```bash
# Lister les CronJobs
kubectl get cronjobs -n databases

# Vérifier l'historique des jobs
kubectl get jobs -n databases -l app=mariadb-backup

# Voir les logs du dernier job
LAST_JOB=$(kubectl get jobs -n databases -l app=mariadb-backup --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
kubectl logs job/$LAST_JOB -n databases

# Déclencher manuellement un job
kubectl create job --from=cronjob/mariadb-full-backup manual-backup-$(date +%s) -n databases
```

---

## Monitoring et métriques

### Prometheus metrics

```bash
# /var/lib/node_exporter/textfile_collector/mariadb_backup.prom
# Generated by backup scripts

# HELP mariadb_backup_last_success_timestamp Unix timestamp of last successful backup
# TYPE mariadb_backup_last_success_timestamp gauge
mariadb_backup_last_success_timestamp{type="full"} 1702476000
mariadb_backup_last_success_timestamp{type="incremental"} 1702497600

# HELP mariadb_backup_size_bytes Size of backup in bytes
# TYPE mariadb_backup_size_bytes gauge
mariadb_backup_size_bytes{type="full"} 214748364800
mariadb_backup_size_bytes{type="incremental"} 5368709120

# HELP mariadb_backup_duration_seconds Duration of backup in seconds
# TYPE mariadb_backup_duration_seconds gauge
mariadb_backup_duration_seconds{type="full"} 3600
mariadb_backup_duration_seconds{type="incremental"} 180

# HELP mariadb_backup_success Boolean indicator of backup success (1=success, 0=failure)
# TYPE mariadb_backup_success gauge
mariadb_backup_success{type="full"} 1
mariadb_backup_success{type="incremental"} 1

# HELP mariadb_backup_age_seconds Time since last successful backup
# TYPE mariadb_backup_age_seconds gauge
mariadb_backup_age_seconds{type="full"} 21600
mariadb_backup_age_seconds{type="incremental"} 300
```

### Requêtes PromQL

```promql
# Alerte si backup échoué
mariadb_backup_success{type="full"} == 0

# Alerte si pas de backup depuis 36h
time() - mariadb_backup_last_success_timestamp{type="full"} > 129600

# Taille moyenne des backups (7 derniers jours)
avg_over_time(mariadb_backup_size_bytes{type="full"}[7d])

# Taux de croissance des backups
rate(mariadb_backup_size_bytes{type="full"}[7d])

# Durée moyenne des backups
avg_over_time(mariadb_backup_duration_seconds{type="full"}[7d])
```

### Alertes Prometheus

```yaml
# prometheus-rules.yaml
groups:
  - name: mariadb_backup_alerts
    rules:
      # Backup failed
      - alert: MariaDBBackupFailed
        expr: mariadb_backup_success == 0
        for: 5m
        labels:
          severity: critical
          component: backup
        annotations:
          summary: "MariaDB backup failed"
          description: "Backup type {{ $labels.type }} failed on {{ $labels.instance }}"
      
      # No backup in 36 hours
      - alert: MariaDBBackupMissing
        expr: time() - mariadb_backup_last_success_timestamp{type="full"} > 129600
        for: 10m
        labels:
          severity: critical
          component: backup
        annotations:
          summary: "MariaDB full backup missing for 36h"
          description: "Last successful backup was {{ $value | humanizeDuration }} ago"
      
      # Backup size growing abnormally
      - alert: MariaDBBackupSizeGrowthHigh
        expr: |
          (
            mariadb_backup_size_bytes{type="full"} - 
            mariadb_backup_size_bytes{type="full"} offset 7d
          ) / mariadb_backup_size_bytes{type="full"} offset 7d > 0.5
        for: 1h
        labels:
          severity: warning
          component: backup
        annotations:
          summary: "MariaDB backup size increased by >50% in 7 days"
          description: "Current size: {{ $value | humanize1024 }}"
      
      # Backup duration too long
      - alert: MariaDBBackupDurationHigh
        expr: mariadb_backup_duration_seconds{type="full"} > 7200
        for: 10m
        labels:
          severity: warning
          component: backup
        annotations:
          summary: "MariaDB backup taking too long"
          description: "Backup duration: {{ $value | humanizeDuration }}"
```

### Dashboard Grafana

```json
{
  "dashboard": {
    "title": "MariaDB Backups",
    "panels": [
      {
        "title": "Backup Success Rate",
        "targets": [
          {
            "expr": "avg_over_time(mariadb_backup_success{type=\"full\"}[24h])"
          }
        ],
        "type": "stat"
      },
      {
        "title": "Time Since Last Backup",
        "targets": [
          {
            "expr": "time() - mariadb_backup_last_success_timestamp"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Backup Size Trend",
        "targets": [
          {
            "expr": "mariadb_backup_size_bytes{type=\"full\"}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Backup Duration",
        "targets": [
          {
            "expr": "mariadb_backup_duration_seconds"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

---

## ✅ Points clés à retenir

- **Automatisation obligatoire** : Backups manuels = risque inacceptable en production
- **Scripts robustes** : Gestion erreurs, retry logic, logging structuré, notifications
- **Orchestration** : Cron (simple), Systemd timers (moderne), K8s CronJobs (cloud-native)
- **Validation** : Checksums automatiques, tests de restauration mensuels minimum
- **Monitoring** : Prometheus metrics, alertes Grafana, dashboards temps réel
- **Notifications** : Email, Slack, PagerDuty en cas succès/échec
- **Rétention** : Politique GFS automatisée (quotidien, hebdo, mensuel)
- **Cloud** : Upload S3 automatique, lifecycle policies, chiffrement
- **Tests** : Automatiser les tests de restauration (mensuels minimum)
- **Documentation** : Procédures à jour, logs structurés, runbooks accessibles

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 Systemd Timers - Arch Wiki](https://wiki.archlinux.org/title/Systemd/Timers)
- [📖 Kubernetes CronJobs - K8s Docs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [📖 Prometheus Alerting - Prometheus Docs](https://prometheus.io/docs/alerting/latest/overview/)

---

## ➡️ Section suivante

**[12.7 - Tests de restauration et PRA](./07-tests-restauration-pra.md)** : Méthodologie de tests, scénarios d'incident, documentation du Plan de Reprise d'Activité.

---


⏭️ [Tests de restauration et plan de reprise (PRA)](/12-sauvegarde-restauration/07-tests-restauration-pra.md)

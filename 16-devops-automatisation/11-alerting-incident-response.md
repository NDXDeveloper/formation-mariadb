🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.11 Alerting et Incident Response

> **Niveau** : Expert  
> **Durée estimée** : 7-8 heures  
> **Prérequis** : 
> - Sections 16.9-16.10 Monitoring et Observabilité maîtrisées
> - Expérience avec gestion d'incidents en production
> - Compréhension de Prometheus Alertmanager
> - Notions de SRE (Site Reliability Engineering)
> - Familiarité avec on-call et PagerDuty

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Concevoir** une stratégie d'alerting efficace (éviter alert fatigue)
- **Définir** SLOs/SLIs pertinents pour MariaDB
- **Configurer** Alertmanager pour routing et escalation intelligents
- **Implémenter** un processus d'incident response structuré
- **Gérer** on-call rotations et escalations
- **Rédiger** post-mortems constructifs
- **Créer** runbooks actionnables
- **Intégrer** ChatOps (Slack, PagerDuty, MS Teams)
- **Mesurer** et améliorer MTTR/MTTD

---

## Introduction

### Le problème de l'alert fatigue

```
┌──────────────────────────────────────────────────────────────┐
│                   Alert Fatigue Cycle                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1️⃣  Trop d'alertes → 📧📧📧 Inbox pleine                    │
│                                                              │
│  2️⃣  Majorité false positives ou bruit                       │
│                                                              │
│  3️⃣  Équipe ignore alertes (alert fatigue)                   │
│                                                              │
│  4️⃣  Alerte critique noyée dans le bruit                     │
│                                                              │
│  5️⃣  Incident critique manqué → OUTAGE 🔥                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Statistiques réelles** (source: Google SRE Book) :
- 📊 Équipe typique : **95% des alertes sont non-actionnables**
- ⏱️ MTTD (Mean Time To Detect) augmente avec alert fatigue
- 😰 Burnout on-call corrélé avec nombre d'alertes

### Alerting efficace : Principes

```
┌──────────────────────────────────────────────────────────────┐
│              Good Alerting Principles                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. ACTIONABLE                                               │
│     Alerte = Action immédiate requise                        │
│     ❌ "CPU high" (et alors?)                                │
│     ✅ "Database down, users impacted"                       │
│                                                              │
│  2. SYMPTOM-BASED (pas cause-based)                          │
│     Alerter sur impact utilisateur, pas métrique technique   │
│     ❌ "Buffer pool 90% full"                                │
│     ✅ "Query latency p95 > 2s (SLO breach)"                 │
│                                                              │
│  3. PROPORTIONAL                                             │
│     Severité proportionnelle à l'impact                      │
│     Critical = Revenue loss / Data loss                      │
│     Warning = Degraded but functional                        │
│                                                              │
│  4. CONTEXTUALIZED                                           │
│     Alerte avec context (what, when, who, impact)            │
│     Lien vers runbook, dashboard, logs                       │
│                                                              │
│  5. TUNED                                                    │
│     Faux positifs < 5%                                       │
│     Révision régulière (post-mortems)                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Pyramide d'alerting

### Modèle en pyramide inversée

```
┌──────────────────────────────────────────────────────────────────┐
│                    Alerting Pyramid                              │
│                                                                  │
│                         🔴 CRITICAL                              │
│                    ╱────────────────╲                            │
│                   ╱ Page immédiate   ╲                           │
│                  ╱  (PagerDuty)       ╲                          │
│                 ╱   Ex: DB down        ╲                         │
│                ╱    Impact users        ╲                        │
│               ╱                          ╲                       │
│              ╱─────────────────────────────╲                     │
│                                                                  │
│                    🟠 WARNING                                    │
│              ╱────────────────────────────────╲                  │
│             ╱ Notification Slack/Email         ╲                 │
│            ╱  (pas de page)                     ╲                │
│           ╱   Ex: Réplication lag >60s           ╲               │
│          ╱    Pas d'impact immédiat               ╲              │
│         ╱                                          ╲             │
│        ╱────────────────────────────────────────────╲            │
│                                                                  │
│                    🟡 INFO                                       │
│          ╱──────────────────────────────────────────────╲        │
│         ╱ Dashboard seulement (pas de notification)      ╲       │
│        ╱  Ex: Connexions 60% utilisées                    ╲      │
│       ╱   Monitoring, pas d'action requise                 ╲     │
│      ╱                                                      ╲    │
│     ╱────────────────────────────────────────────────────────╲   │
│                                                                  │
│                    🟢 DEBUG                                      │
│      ╱────────────────────────────────────────────────────────╲  │
│     ╱ Logs seulement (queryable on-demand)                    ╲  │
│    ╱  Ex: Query execution time détails                         ╲ │
│   ╱                                                             ╱│
│  ╱─────────────────────────────────────────────────────────────╱ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Règle: Plus on monte, MOINS il y a d'alertes
       Critical: < 5 alertes/semaine
       Warning: < 20 alertes/semaine
```

### Classification des alertes MariaDB

| Alerte | Severité | Action | Destination |
|--------|----------|--------|-------------|
| **Database down** | 🔴 Critical | Page immédiate | PagerDuty |
| **Replication stopped** | 🔴 Critical | Page immédiate | PagerDuty |
| **Galera cluster < 3 nodes** | 🔴 Critical | Page immédiate | PagerDuty |
| **Disk space < 5%** | 🔴 Critical | Page immédiate | PagerDuty |
| **Connection pool exhausted** | 🔴 Critical | Page immédiate | PagerDuty |
| **Replication lag > 300s** | 🟠 Warning | Slack notification | Slack #dba-team |
| **Slow queries > 20/min** | 🟠 Warning | Slack notification | Slack #dba-team |
| **Buffer pool hit ratio < 95%** | 🟠 Warning | Slack notification | Slack #dba-team |
| **Connections > 80%** | 🟡 Info | Dashboard only | Grafana |
| **Query execution time p95** | 🟡 Info | Dashboard only | Grafana |

---

## SLOs, SLIs, SLAs

### Définitions

```
┌──────────────────────────────────────────────────────────────┐
│                 SLO / SLI / SLA Explained                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SLI (Service Level Indicator)                               │
│  ───────────────────────────────                             │
│  Métrique quantitative de niveau de service                  │
│  Exemples:                                                   │
│  - Query latency p95                                         │
│  - Availability (uptime %)                                   │
│  - Error rate                                                │
│                                                              │
│  SLO (Service Level Objective)                               │
│  ───────────────────────────────                             │
│  Target pour un SLI                                          │
│  Exemples:                                                   │
│  - Query latency p95 < 100ms (99.9% du temps)                │
│  - Availability > 99.9% (43min downtime/mois max)            │
│  - Error rate < 0.1%                                         │
│                                                              │
│  SLA (Service Level Agreement)                               │
│  ───────────────────────────────                             │
│  Contrat business avec conséquences si non respecté          │
│  Exemples:                                                   │
│  - Uptime 99.9% garanti, sinon 10% credit                    │
│  - SLA externe (clients) vs SLO interne (équipe)             │
│                                                              │
│  ──────────────────────────────────────────────────────────  │
│                                                              │
│  Relation:                                                   │
│  SLI (mesure) → SLO (objectif) → SLA (contrat)               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### SLIs pour MariaDB

**4 Golden Signals (Google SRE)** adaptés à MariaDB :

```yaml
# 1. LATENCY (Latence)
slis:
  query_latency_p50:
    description: "Median query execution time"
    query: "histogram_quantile(0.50, rate(mysql_global_status_queries[5m]))"
    unit: "seconds"
  
  query_latency_p95:
    description: "95th percentile query execution time"
    query: "histogram_quantile(0.95, rate(mysql_global_status_queries[5m]))"
    unit: "seconds"
  
  query_latency_p99:
    description: "99th percentile query execution time"
    query: "histogram_quantile(0.99, rate(mysql_global_status_queries[5m]))"
    unit: "seconds"

# 2. TRAFFIC (Trafic)
slis:
  qps:
    description: "Queries per second"
    query: "rate(mysql_global_status_queries[5m])"
    unit: "queries/second"
  
  connections:
    description: "Active connections"
    query: "mysql_global_status_threads_connected"
    unit: "count"

# 3. ERRORS (Erreurs)
slis:
  error_rate:
    description: "Rate of failed queries"
    query: "rate(mysql_global_status_aborted_connects[5m])"
    unit: "errors/second"
  
  replication_errors:
    description: "Replication errors"
    query: "mysql_slave_status_last_errno"
    unit: "count"

# 4. SATURATION (Saturation)
slis:
  connection_usage:
    description: "Connection pool utilization"
    query: "(mysql_global_status_threads_connected / mysql_global_variables_max_connections) * 100"
    unit: "percentage"
  
  buffer_pool_usage:
    description: "InnoDB buffer pool utilization"
    query: "(mysql_global_status_innodb_buffer_pool_bytes_data / mysql_global_variables_innodb_buffer_pool_size) * 100"
    unit: "percentage"
```

### SLOs pour MariaDB

**Exemple de SLOs réalistes** :

```yaml
slos:
  # Availability
  - name: "Database Availability"
    sli: "mysql_up"
    target: 99.95  # 99.95% uptime
    window: "30d"
    # = 21.6 minutes downtime max par mois
  
  # Latency
  - name: "Query Latency P95"
    sli: "query_latency_p95"
    target: 100  # < 100ms
    percentile: 99.9  # 99.9% des queries
    window: "30d"
  
  - name: "Query Latency P99"
    sli: "query_latency_p99"
    target: 500  # < 500ms
    percentile: 99.0
    window: "30d"
  
  # Error rate
  - name: "Query Success Rate"
    sli: "1 - (error_rate / qps)"
    target: 99.9  # < 0.1% errors
    window: "30d"
  
  # Replication
  - name: "Replication Lag"
    sli: "mysql_slave_status_seconds_behind_master"
    target: 60  # < 60 seconds
    percentile: 99.0
    window: "30d"
```

### Error Budget

**Concept** : Si SLO = 99.9%, alors error budget = 0.1%

```
┌──────────────────────────────────────────────────────────────┐
│                    Error Budget Example                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SLO: 99.9% availability (monthly)                           │
│                                                              │
│  Error budget = 100% - 99.9% = 0.1%                          │
│                = 43.2 minutes de downtime autorisé / mois    │
│                                                              │
│  Utilisation error budget:                                   │
│  ────────────────────────────                                │
│  Semaine 1: Incident 15 min    →  35% budget utilisé         │
│  Semaine 2: Incident 10 min    →  58% budget utilisé         │
│  Semaine 3: Incident 20 min    →  104% budget dépassé ⚠️     │
│                                                              │
│  Actions quand budget dépassé:                               │
│  - Freeze deployments (sauf fixes)                           │
│  - Focus 100% sur stability                                  │
│  - Root cause analysis obligatoire                           │
│  - Amélioration avant nouveaux features                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Recording rule pour tracking** :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mariadb-slo-recording
spec:
  groups:
  - name: mariadb-slo
    interval: 1m
    rules:
    # Availability SLI
    - record: slo:mariadb_availability:ratio_30d
      expr: |
        avg_over_time(mysql_up[30d])
    
    # Error budget (inverse availability)
    - record: slo:mariadb_error_budget:ratio_30d
      expr: |
        1 - slo:mariadb_availability:ratio_30d
    
    # Error budget burn rate (combien vite on consomme budget)
    - record: slo:mariadb_error_budget_burn_rate:1h
      expr: |
        (
          1 - avg_over_time(mysql_up[1h])
        ) / (1 - 0.999)  # 0.999 = SLO target
```

---

## Configuration Alertmanager avancée

### Routing complexe

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      
      # Intégrations
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
      pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'
    
    # Templates pour messages
    templates:
    - '/etc/alertmanager/templates/*.tmpl'
    
    # Routing principal
    route:
      receiver: 'default'
      group_by: ['alertname', 'cluster', 'namespace']
      group_wait: 10s        # Attendre 10s avant d'envoyer groupe
      group_interval: 5m     # Intervalle entre groupes
      repeat_interval: 12h   # Re-notifier si toujours actif
      
      routes:
      # ════════════════════════════════════════════════════════
      # CRITICAL: Database Down
      # ════════════════════════════════════════════════════════
      - match:
          alertname: MariaDBDown
          severity: critical
        receiver: 'dba-pagerduty'
        continue: true  # Aussi envoyer vers autres routes
        
        # Escalation après 15min si pas résolu
        routes:
        - match:
            alertname: MariaDBDown
          receiver: 'platform-lead-pagerduty'
          group_wait: 15m
      
      # ════════════════════════════════════════════════════════
      # CRITICAL: Autres alertes critiques
      # ════════════════════════════════════════════════════════
      - match:
          severity: critical
          component: mariadb
        receiver: 'dba-pagerduty'
        continue: true
      
      # ════════════════════════════════════════════════════════
      # WARNING: Notifications Slack uniquement
      # ════════════════════════════════════════════════════════
      - match:
          severity: warning
          component: mariadb
        receiver: 'dba-slack'
        # Throttling: max 1 notification/5min pour même alerte
        group_interval: 5m
      
      # ════════════════════════════════════════════════════════
      # Production vs Non-Production
      # ════════════════════════════════════════════════════════
      - match:
          environment: production
          severity: critical
        receiver: 'dba-pagerduty'
      
      - match:
          environment: staging
          severity: critical
        receiver: 'dba-slack'  # Staging = pas de page, juste Slack
      
      # ════════════════════════════════════════════════════════
      # Heures ouvrables vs On-call
      # ════════════════════════════════════════════════════════
      - match_re:
          severity: warning
        receiver: 'dba-slack'
        # Mute warnings en dehors heures ouvrables
        mute_time_intervals:
        - weekends
        - nights
    
    # ════════════════════════════════════════════════════════
    # Inhibition Rules (suppress alertes dérivées)
    # ════════════════════════════════════════════════════════
    inhibit_rules:
    # Si DB down, supprimer toutes les autres alertes MariaDB
    - source_match:
        alertname: MariaDBDown
      target_match_re:
        alertname: MariaDB.*
      equal: ['namespace', 'pod']
    
    # Si cluster Galera en perte de quorum, supprimer node alerts
    - source_match:
        alertname: GaleraClusterSizeReduced
        severity: critical
      target_match:
        alertname: GaleraNodeNotSynced
      equal: ['namespace']
    
    # Si réplication stopped, supprimer replication lag alerts
    - source_match:
        alertname: MariaDBReplicationStopped
      target_match:
        alertname: MariaDBReplicationLag
      equal: ['namespace', 'pod']
    
    # ════════════════════════════════════════════════════════
    # Receivers (destinations)
    # ════════════════════════════════════════════════════════
    receivers:
    # Default (fallback)
    - name: 'default'
      slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
    
    # DBA PagerDuty (critical alerts)
    - name: 'dba-pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: |
          {{ .GroupLabels.alertname }}
          {{ range .Alerts }}
          {{ .Annotations.summary }}
          {{ end }}
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
          num_firing: '{{ .Alerts.Firing | len }}'
          num_resolved: '{{ .Alerts.Resolved | len }}'
        client: 'Prometheus'
        client_url: 'http://prometheus.monitoring.svc:9090'
    
    # DBA Slack (warnings)
    - name: 'dba-slack'
      slack_configs:
      - channel: '#dba-team'
        send_resolved: true
        title: |
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Namespace:* {{ .Labels.namespace }}
          *Pod:* {{ .Labels.pod }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
        actions:
        - type: button
          text: 'View Dashboard :grafana:'
          url: 'http://grafana.monitoring.svc/d/mariadb-overview'
        - type: button
          text: 'Silence :no_bell:'
          url: 'http://alertmanager.monitoring.svc/#/silences/new'
    
    # Platform Lead (escalation)
    - name: 'platform-lead-pagerduty'
      pagerduty_configs:
      - service_key: 'PLATFORM_LEAD_SERVICE_KEY'
        description: |
          🚨 ESCALATION: {{ .GroupLabels.alertname }}
          Unresolved after 15 minutes
    
    # ════════════════════════════════════════════════════════
    # Mute Time Intervals (silences programmées)
    # ════════════════════════════════════════════════════════
    mute_time_intervals:
    - name: weekends
      time_intervals:
      - weekdays: ['saturday', 'sunday']
    
    - name: nights
      time_intervals:
      - times:
        - start_time: '22:00'
          end_time: '08:00'
```

### Templates de messages

```yaml
# /etc/alertmanager/templates/mariadb.tmpl
{{ define "slack.mariadb.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] 
🗄️ MariaDB: {{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.mariadb.text" }}
{{ range .Alerts }}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
*Summary:* {{ .Annotations.summary }}
*Description:* {{ .Annotations.description }}

📊 *Details:*
• Severity: {{ .Labels.severity }}
• Namespace: {{ .Labels.namespace }}
• Pod: {{ .Labels.pod }}
• Environment: {{ .Labels.environment }}

⏰ *Timing:*
• Started: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
{{ if .EndsAt }}• Ended: {{ .EndsAt.Format "2006-01-02 15:04:05" }}{{ end }}

📖 *Runbook:* {{ .Annotations.runbook_url }}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{{ end }}
{{ end }}
```

---

## Incident Response Process

### Incident Severity Levels

```
┌──────────────────────────────────────────────────────────────┐
│                 Incident Severity Definitions                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🔴 SEV-1 (Critical)                                         │
│  ────────────────────                                        │
│  Impact: Service down pour TOUS les utilisateurs             │
│  Exemples:                                                   │
│  - Database complètement down                                │
│  - Data corruption                                           │
│  - Security breach                                           │
│                                                              │
│  Response:                                                   │
│  - Page immédiate (< 5 min)                                  │
│  - War room avec Incident Commander                          │
│  - Communication clients toutes les 30 min                   │
│  - Post-mortem obligatoire                                   │
│                                                              │
│  ──────────────────────────────────────────────────────────  │
│                                                              │
│  🟠 SEV-2 (High)                                             │
│  ────────────────                                            │
│  Impact: Service dégradé pour majorité utilisateurs          │
│  Exemples:                                                   │
│  - Réplication stopped (risque data loss)                    │
│  - Performance très dégradée (p95 > 5s)                      │
│  - 1 node Galera cluster down                                │
│                                                              │
│  Response:                                                   │
│  - Page (< 15 min)                                           │
│  - Incident Response Team mobilisée                          │
│  - Communication interne toutes les heures                   │
│  - Post-mortem recommandé                                    │
│                                                              │
│  ──────────────────────────────────────────────────────────  │
│                                                              │
│  🟡 SEV-3 (Medium)                                           │
│  ──────────────────                                          │
│  Impact: Subset utilisateurs ou degradation mineure          │
│  Exemples:                                                   │
│  - Slow queries spike                                        │
│  - Connection pool 95% utilisé                               │
│  - Replication lag 5 minutes                                 │
│                                                              │
│  Response:                                                   │
│  - Notification (pas de page)                                │
│  - Investigation pendant heures ouvrables                    │
│  - Fix dans next sprint                                      │
│                                                              │
│  ──────────────────────────────────────────────────────────  │
│                                                              │
│  🟢 SEV-4 (Low)                                              │
│  ─────────────────                                           │
│  Impact: Aucun impact utilisateur                            │
│  Exemples:                                                   │
│  - Métrique tendance préoccupante                            │
│  - Warning dans logs                                         │
│                                                              │
│  Response:                                                   │
│  - Ticket backlog                                            │
│  - Investigation opportuniste                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Incident Response Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│              Incident Response Lifecycle                        │
│                                                                 │
│  1️⃣  DETECTION                                                  │
│     ─────────────────────────────────────────────────────────   │
│     • Alerte Prometheus/Alertmanager                            │
│     • Page PagerDuty                                            │
│     • User report                                               │
│     ↓                                                           │
│     ⏱️ MTTD (Mean Time To Detect) starts                        │
│                                                                 │
│  2️⃣  RESPONSE                                                   │
│     ─────────────────────────────────────────────────────────   │
│     • On-call engineer acknowledge (< 5min for SEV-1)           │
│     • Créer incident dans système (Jira, PagerDuty)             │
│     • Déterminer severity                                       │
│     ↓                                                           │
│     ⏱️ MTTA (Mean Time To Acknowledge)                          │
│                                                                 │
│  3️⃣  TRIAGE                                                     │
│     ─────────────────────────────────────────────────────────   │
│     • Incident Commander assigné (si SEV-1/SEV-2)               │
│     • Créer war room (Slack channel, Zoom)                      │
│     • Mobiliser équipe (DBA, Platform, App)                     │
│     • Assess impact (combien d'utilisateurs?)                   │
│     ↓                                                           │
│                                                                 │
│  4️⃣  INVESTIGATION                                              │
│     ─────────────────────────────────────────────────────────   │
│     • Consulter dashboard (Grafana)                             │
│     • Analyser logs (Loki)                                      │
│     • Check traces (Jaeger)                                     │
│     • Runbook consultation                                      │
│     • Hypothèses → Tests                                        │
│     ↓                                                           │
│                                                                 │
│  5️⃣  MITIGATION                                                 │
│     ─────────────────────────────────────────────────────────   │
│     • Action immédiate pour restaurer service                   │
│     • Peut être workaround temporaire                           │
│     • Exemples:                                                 │
│       - Restart MariaDB                                         │
│       - Rollback deployment                                     │
│       - Scale up resources                                      │
│       - Disable feature flag                                    │
│     ↓                                                           │
│     ⏱️ MTTR (Mean Time To Repair)                               │
│                                                                 │
│  6️⃣  RECOVERY                                                   │
│     ─────────────────────────────────────────────────────────   │
│     • Service restauré                                          │
│     • Vérifier SLIs revenus à normal                            │
│     • Monitoring intensifié 24-48h                              │
│     ↓                                                           │
│                                                                 │
│  7️⃣  POST-INCIDENT                                              │
│     ─────────────────────────────────────────────────────────   │
│     • Post-mortem réunion (dans 24-48h)                         │
│     • Documenter timeline                                       │
│     • Identifier root cause(s)                                  │
│     • Action items pour prévenir récurrence                     │
│     • Mettre à jour runbooks                                    │
│     • Améliorer alerting si false positive                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Incident Command System

**Rôles pendant incident SEV-1/SEV-2** :

```yaml
roles:
  incident_commander:
    responsibilities:
    - Coordonner réponse
    - Décisions go/no-go
    - Communication externe
    - Assign tasks
    person: "DBA Lead ou Platform Lead"
  
  technical_lead:
    responsibilities:
    - Investigation technique
    - Hypothèses et tests
    - Implémentation fix
    person: "Senior DBA"
  
  communications_lead:
    responsibilities:
    - Status updates (Slack, email)
    - Customer communication
    - Documentation timeline
    person: "Product Manager ou Support Lead"
  
  scribe:
    responsibilities:
    - Document timeline (actions, décisions)
    - Logs chat war room
    - Préparer post-mortem doc
    person: "Junior engineer ou PM"
```

---

## Runbooks

### Structure de runbook

**Template runbook MariaDB** :

```markdown
# Runbook: MariaDB Down

## Metadata
- **Alert Name**: MariaDBDown
- **Severity**: SEV-1 (Critical)
- **Owner**: DBA Team
- **Last Updated**: 2025-12-14
- **Reviewed By**: Platform Team

## Summary
MariaDB instance is down and not accepting connections.
**Impact**: Complete service outage for all users.

## Detection
- Prometheus alert: `mysql_up == 0`
- Users cannot access application
- PagerDuty page sent to on-call DBA

## Triage Questions
1. Is this a single pod or all pods?
   ```bash
   kubectl get pods -n databases -l app=mariadb
   ```

2. Are there recent deployments or changes?
   ```bash
   kubectl rollout history statefulset/mariadb -n databases
   ```

3. Are there disk space issues?
   ```bash
   kubectl exec -n databases mariadb-0 -- df -h
   ```

4. Are there OOM kills?
   ```bash
   kubectl describe pod -n databases mariadb-0 | grep -i oom
   ```

## Investigation Steps

### Step 1: Check Pod Status
```bash
kubectl get pods -n databases -l app=mariadb
kubectl describe pod -n databases mariadb-0
kubectl logs -n databases mariadb-0 --tail=100
```

**Expected Output**: Pod status, recent logs, error messages

### Step 2: Check MariaDB Error Log
```bash
kubectl exec -n databases mariadb-0 -- tail -100 /var/log/mysql/error.log
```

**Look for**:
- `InnoDB: Out of memory`
- `Can't create/write to file`
- `Disk is full`
- `Crash recovery`

### Step 3: Check Resources
```bash
# CPU/Memory
kubectl top pod -n databases mariadb-0

# Disk
kubectl exec -n databases mariadb-0 -- df -h /var/lib/mysql
```

### Step 4: Check Prometheus Metrics
Open Grafana dashboard: http://grafana/d/mariadb-overview
- Check QPS before crash
- Check slow queries
- Check connections spike

## Common Causes & Fixes

### Cause 1: Out of Memory (OOM)
**Symptoms**: Pod restarted, dmesg shows OOM kill
**Fix**:
```bash
# Increase memory limit
kubectl patch statefulset mariadb -n databases -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "mariadb",
          "resources": {
            "limits": {"memory": "16Gi"}
          }
        }]
      }
    }
  }
}'
```

### Cause 2: Disk Full
**Symptoms**: Error log shows "Disk is full"
**Fix**:
```bash
# Emergency: Purge old binlogs
kubectl exec -n databases mariadb-0 -- mysql -uroot -p${ROOT_PASSWORD} \
  -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 1 DAY);"

# Long-term: Increase PVC size
kubectl patch pvc data-mariadb-0 -n databases -p '
{
  "spec": {
    "resources": {
      "requests": {"storage": "200Gi"}
    }
  }
}'
```

### Cause 3: Crash During Startup
**Symptoms**: Pod CrashLoopBackOff, error log shows "InnoDB: Database corruption"
**Fix**:
```bash
# Try crash recovery
kubectl exec -n databases mariadb-0 -- \
  mysqld --innodb-force-recovery=1

# If still fails, restore from backup (see Backup Runbook)
```

### Cause 4: Configuration Error
**Symptoms**: Error log shows "unknown variable"
**Fix**:
```bash
# Check my.cnf syntax
kubectl exec -n databases mariadb-0 -- \
  mysqld --help --verbose | grep error

# Rollback ConfigMap
kubectl rollout undo configmap/mariadb-config -n databases
kubectl rollout restart statefulset/mariadb -n databases
```

## Escalation Path

1. **Try fixes above** (15 minutes)
2. If not resolved → **Escalate to DBA Lead**
   - Slack: @dba-lead
   - Phone: +33 6 XX XX XX XX

3. If still not resolved (30 minutes) → **Escalate to Platform Lead**
   - Slack: @platform-lead
   - Phone: +33 6 YY YY YY YY

4. If critical (1 hour) → **Page CTO**

## Communication Templates

### Slack Announcement (Initial)
```
🚨 INCIDENT: MariaDB Down (SEV-1)
Status: Investigating
Impact: Complete service outage
ETA: Unknown (investigating)
War Room: #incident-2025-12-14-mariadb
Updates: Every 15 minutes
```

### Customer Status Page
```
We are currently experiencing database connectivity issues.
Our team is actively working on a resolution.
Next update: 14:30 UTC
```

## Post-Incident Actions

- [ ] Write post-mortem (within 48h)
- [ ] Update this runbook with learnings
- [ ] Improve alerting if false positive
- [ ] Add to knowledge base
- [ ] Schedule preventive actions

## Related Runbooks
- [Runbook: MariaDB Performance Degradation](runbook-performance.md)
- [Runbook: MariaDB Backup & Restore](runbook-backup.md)
- [Runbook: Galera Cluster Recovery](runbook-galera.md)

## References
- [MariaDB Documentation](https://mariadb.com/kb/en/)
- [Grafana Dashboard](http://grafana/d/mariadb-overview)
- [On-call Schedule](https://pagerduty.com/schedules/mariadb)
```

---

## Post-Mortems

### Template de post-mortem

```markdown
# Post-Mortem: MariaDB Outage - 2025-12-14

## Metadata
- **Date**: 2025-12-14
- **Duration**: 45 minutes (14:15 - 15:00 UTC)
- **Severity**: SEV-1
- **Incident Commander**: Alice (DBA Lead)
- **Authors**: Alice, Bob (Platform), Carol (App Team)

## Executive Summary
MariaDB primary database went down due to disk space exhaustion,
causing complete service outage for 45 minutes. Approximately
10,000 users were impacted. Service was restored by purging old
binary logs. Revenue impact estimated at $5,000.

## Impact
- **Users Affected**: ~10,000 (100% of active users)
- **Duration**: 45 minutes
- **Revenue Loss**: ~$5,000
- **SLA Breach**: Yes (99.9% → 99.87% for month)
- **Error Budget**: 32% consumed in single incident

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 14:15:00 | 🔴 Prometheus alert: MariaDBDown fired |
| 14:15:30 | Alice (on-call) paged via PagerDuty |
| 14:16:00 | Alice acknowledges, starts investigation |
| 14:18:00 | War room created (#incident-2025-12-14) |
| 14:20:00 | Bob joins war room |
| 14:22:00 | Identified: Disk 100% full |
| 14:25:00 | Root cause: Binary logs not purged (auto-purge disabled) |
| 14:27:00 | Decision: Manual purge of binlogs >7 days |
| 14:30:00 | Binlogs purged, 50GB freed |
| 14:32:00 | MariaDB restarted successfully |
| 14:35:00 | Service health checks passing |
| 14:40:00 | Traffic restored to 100% |
| 15:00:00 | Incident closed, monitoring continues |

## Root Cause Analysis

### What Happened
1. Binary logging enabled for replication
2. Auto-purge disabled in my.cnf (expire_logs_days = 0)
3. Binlogs accumulated over 3 months → 150GB
4. Disk 200GB → reached 100% capacity
5. MariaDB unable to write, crashed

### Why It Happened
- **Immediate cause**: Disk full
- **Contributing factors**:
  1. Binary log retention not configured properly
  2. No monitoring on disk usage growth rate
  3. No alert for disk >90% (only >95%)
  4. Binlog rotation not tested in staging

### Five Whys
1. **Why did MariaDB crash?**
   → Disk was full

2. **Why was disk full?**
   → Binary logs accumulated to 150GB

3. **Why did binlogs accumulate?**
   → Auto-purge was disabled (expire_logs_days = 0)

4. **Why was auto-purge disabled?**
   → Initial config mistake, never caught in code review

5. **Why wasn't it caught?**
   → No monitoring on binlog disk usage, no tests for this scenario

## What Went Well
- ✅ Alert fired immediately (MTTD < 1 minute)
- ✅ On-call responded quickly (MTTA < 2 minutes)
- ✅ War room established quickly
- ✅ Root cause identified in 7 minutes
- ✅ Mitigation effective (purge)
- ✅ Communication clear and frequent
- ✅ No data loss

## What Went Wrong
- ❌ Config error not caught in review
- ❌ No staging test for binlog accumulation
- ❌ Disk alert threshold too high (95% vs 85%)
- ❌ No growth rate monitoring
- ❌ Runbook didn't cover this scenario

## Action Items

| Priority | Action | Owner | Due Date | Status |
|----------|--------|-------|----------|--------|
| 🔴 P0 | Enable binlog auto-purge (expire_logs_days=7) | Alice | 2025-12-15 | ✅ Done |
| 🔴 P0 | Add disk usage alert at 85% | Bob | 2025-12-15 | ✅ Done |
| 🟠 P1 | Add binlog size monitoring | Alice | 2025-12-18 | 🟡 In Progress |
| 🟠 P1 | Update runbook with binlog scenario | Carol | 2025-12-20 | ⏳ Todo |
| 🟠 P1 | Add staging test for binlog accumulation | Bob | 2025-12-22 | ⏳ Todo |
| 🟡 P2 | Review all my.cnf configs for similar issues | Alice | 2025-12-30 | ⏳ Todo |
| 🟡 P2 | Increase PVC size to 500GB | Bob | 2026-01-15 | ⏳ Todo |

## Lessons Learned

### Technical
1. **Config reviews must include operational impact**
   - Adding checklist: "How will this config behave after 3 months?"

2. **Disk monitoring insufficient**
   - Need both absolute threshold (85%) AND growth rate

3. **Binlog management critical**
   - Documented best practice: expire_logs_days = 7-14

### Process
1. **Staging must mirror production config**
   - Binlogs disabled in staging → missed this issue

2. **Runbooks need regular review**
   - Last update: 6 months ago (too old)

3. **Error budget tracking needed**
   - 32% consumed in one incident → need visibility

## Follow-up Review
- **1 week**: Review action items progress
- **1 month**: Verify no recurrence
- **3 months**: Incorporate learnings into training

## Attachments
- [Grafana Dashboard Screenshot](link)
- [Error Log Excerpt](link)
- [War Room Chat Log](link)
```

### Post-Mortem Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│              Blameless Post-Mortem Culture                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ DO:                                                      │
│  - Focus on WHAT happened, not WHO                           │
│  - Assume good intent                                        │
│  - Look for systemic issues                                  │
│  - Learn and improve                                         │
│  - Share publicly (transparency)                             │
│                                                              │
│  ❌ DON'T:                                                   │
│  - Blame individuals                                         │
│  - Punish mistakes                                           │
│  - Hide incidents                                            │
│  - Skip action items                                         │
│                                                              │
│  Mantra: "Bad systems create bad outcomes"                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## ChatOps Integration

### Slack Commands

```python
# Slack bot pour incident management
# /incident create <severity> <description>
@slack_app.command("/incident")
def incident_create(ack, command, say):
    ack()
    
    # Parse command
    severity = command['text'].split()[0]  # SEV-1, SEV-2, etc.
    description = ' '.join(command['text'].split()[1:])
    
    # Create incident in PagerDuty
    incident = pagerduty.create_incident(
        title=f"{severity}: {description}",
        service_id="MARIADB_SERVICE_ID",
        urgency="high" if severity in ["SEV-1", "SEV-2"] else "low"
    )
    
    # Create war room channel
    channel_name = f"incident-{incident['id']}"
    channel = slack_client.conversations_create(name=channel_name)
    
    # Post to channel
    say(
        channel=channel['id'],
        text=f"""
🚨 **INCIDENT CREATED**
Severity: {severity}
Description: {description}
PagerDuty: {incident['html_url']}
War Room: #{channel_name}

@here - Incident Response Team, please join!
        """
    )
    
    # Start incident timeline
    timeline = IncidentTimeline(incident['id'])
    timeline.add_event("Incident created", command['user_name'])

# /incident status
@slack_app.command("/incident-status")
def incident_status(ack, command, say):
    ack()
    
    # Get active incidents from PagerDuty
    incidents = pagerduty.list_incidents(status="triggered")
    
    if not incidents:
        say("✅ No active incidents")
    else:
        message = "🚨 **ACTIVE INCIDENTS**\n"
        for inc in incidents:
            message += f"• {inc['title']} - {inc['status']}\n"
        say(message)

# /runbook <alert-name>
@slack_app.command("/runbook")
def get_runbook(ack, command, say):
    ack()
    
    alert_name = command['text']
    runbook_url = f"https://wiki.company.com/runbooks/{alert_name}"
    
    say(f"📖 Runbook: {runbook_url}")
```

---

## ✅ Points clés à retenir

- **Alert fatigue = ennemi #1** : Trop d'alertes → ignorer alertes critiques
- **Pyramide alerting** : Critical (page) < Warning (Slack) < Info (dashboard)
- **SLOs/SLIs** : Mesurer ce qui compte pour users (latency, availability, errors)
- **Error budget** : SLO breach = freeze features, focus stability
- **Symptom-based alerting** : Alerter sur impact user, pas métrique technique
- **Routing intelligent** : Severity, environment, time-of-day
- **Inhibition rules** : Supprimer alertes dérivées (DB down → mute other alerts)
- **Incident severity** : SEV-1 (total outage) vs SEV-4 (monitoring only)
- **Runbooks essentiels** : Investigation steps, fixes, escalation
- **Post-mortems blameless** : Focus on systems, not people
- **ChatOps** : Incident management dans Slack

💡 **Golden rule** : "Hope is not a strategy. Monitor, alert, respond, learn, improve."

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [📖 Google SRE Workbook - Alerting](https://sre.google/workbook/alerting-on-slos/)
- [📖 Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)

### Articles de référence
- [📝 The Art of SLOs (Alex Hidalgo)](https://www.alex-hidalgo.com/the-art-of-slos)
- [📝 Incident Response (PagerDuty)](https://response.pagerduty.com/)

### Outils
- [🔧 PagerDuty](https://www.pagerduty.com/)
- [🔧 Opsgenie](https://www.atlassian.com/software/opsgenie)
- [🔧 OnCall (Grafana)](https://grafana.com/products/oncall/)

---

## ➡️ Prochaine section

**16.12 GitOps pour bases de données** : ArgoCD, FluxCD, declarative database management, drift detection, automated reconciliation.

---

**MariaDB** : Version 11.8 LTS

⏭️ [GitOps pour les bases de données](/16-devops-automatisation/12-gitops-bases-donnees.md)

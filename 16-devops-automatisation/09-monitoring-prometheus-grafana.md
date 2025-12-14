🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.9 Monitoring avec Prometheus/Grafana

> **Niveau** : Avancé à Expert  
> **Durée estimée** : 6-7 heures  
> **Prérequis** : 
> - Sections 16.4-16.6 Kubernetes maîtrisées
> - Compréhension des métriques systèmes (CPU, RAM, I/O)
> - Notions de time-series databases
> - Familiarité avec MariaDB performance tuning
> - Expérience avec dashboards (Grafana ou équivalent)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les principes de l'observabilité pour bases de données
- **Déployer** une stack Prometheus/Grafana pour MariaDB sur Kubernetes
- **Configurer** mysqld_exporter pour exposer métriques MariaDB
- **Créer** des dashboards Grafana production-ready
- **Définir** des alertes pertinentes sur métriques critiques
- **Monitorer** réplication, Galera Cluster, et performance queries
- **Diagnostiquer** problèmes de performance via métriques
- **Intégrer** monitoring dans le workflow DevOps

---

## Introduction

### Qu'est-ce que l'observabilité ?

L'**observabilité** (observability) est la capacité à comprendre l'état interne d'un système en observant ses outputs externes.

**Les 3 piliers de l'observabilité** :

```
┌──────────────────────────────────────────────────────────────┐
│                 Three Pillars of Observability               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1️⃣  METRICS (Métriques)                                     │
│     ┌────────────────────────────────────────────────────┐   │
│     │  Agrégations numériques dans le temps              │   │
│     │  Exemples:                                         │   │
│     │  - QPS (Queries per Second): 5,234                 │   │
│     │  - Connections actives: 127                        │   │
│     │  - InnoDB buffer pool usage: 87%                   │   │
│     │  - Slow queries: 12 /minute                        │   │
│     │                                                    │   │
│     │  Stack: Prometheus + mysqld_exporter               │   │
│     └────────────────────────────────────────────────────┘   │
│                                                              │
│  2️⃣  LOGS (Journaux)                                         │
│     ┌────────────────────────────────────────────────────┐   │
│     │  Événements discrets horodatés                     │   │
│     │  Exemples:                                         │   │
│     │  - Error log: "Table doesn't exist"                │   │
│     │  - Slow query log: SELECT took 3.2s                │   │
│     │  - Audit log: User 'admin' connected               │   │
│     │  - Binary log: Transaction committed               │   │
│     │                                                    │   │
│     │  Stack: ELK (Elasticsearch/Logstash/Kibana)        │   │
│     │         ou Loki + Promtail                         │   │
│     └────────────────────────────────────────────────────┘   │
│                                                              │
│  3️⃣  TRACES (Traces distribuées)                             │
│     ┌────────────────────────────────────────────────────┐   │
│     │  Suivi de requêtes à travers systèmes              │   │
│     │  Exemples:                                         │   │
│     │  - API request → Load balancer → App → MariaDB     │   │
│     │  - Latency par composant                           │   │
│     │  - Spans et context propagation                    │   │
│     │                                                    │   │
│     │  Stack: Jaeger, Zipkin, OpenTelemetry              │   │
│     └────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Dans cette section** : Focus sur **Metrics** (Prometheus/Grafana)

### Pourquoi monitorer MariaDB ?

```
┌──────────────────────────────────────────────────────────────┐
│          Sans monitoring                                     │
├──────────────────────────────────────────────────────────────┤
│  ❌ Découvrir problèmes APRÈS les utilisateurs               │
│  ❌ Pas de visibility sur performance                        │
│  ❌ Debugging = deviner                                      │
│  ❌ Capacity planning impossible                             │
│  ❌ Incidents prolongés                                      │
│  ❌ MTTR (Mean Time To Repair) élevé                         │
└──────────────────────────────────────────────────────────────┘

                           VS

┌──────────────────────────────────────────────────────────────┐
│          Avec monitoring                                     │
├──────────────────────────────────────────────────────────────┤
│  ✅ Détection proactive (alertes avant incident)             │
│  ✅ Performance visible en temps réel                        │
│  ✅ Debugging basé sur données (pas devinettes)              │
│  ✅ Capacity planning data-driven                            │
│  ✅ Post-mortem avec metrics historiques                     │
│  ✅ MTTR réduit (troubleshooting rapide)                     │
│  ✅ SLA mesurable et auditable                               │
└──────────────────────────────────────────────────────────────┘
```

**Bénéfices business** :
- 💰 Réduction downtime = économies
- 🚀 Performance optimisée = meilleure UX
- 📊 Capacity planning = coûts maîtrisés
- 🛡️ Sécurité (détection anomalies)
- 📈 Amélioration continue (trends)

---

## Architecture de monitoring pour MariaDB

### Stack complète Prometheus/Grafana

```
┌─────────────────────────────────────────────────────────────────┐
│              Monitoring Stack Architecture                      │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    MariaDB Layer                           │ │
│  │                                                            │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │ │
│  │  │  mariadb-0   │   │  mariadb-1   │   │  mariadb-2   │    │ │
│  │  │  (Primary)   │   │  (Replica)   │   │  (Replica)   │    │ │
│  │  │              │   │              │   │              │    │ │
│  │  │  Port 3306   │   │  Port 3306   │   │  Port 3306   │    │ │
│  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘    │ │
│  │         │                  │                  │            │ │
│  │         │ expose metrics   │                  │            │ │
│  │         ▼                  ▼                  ▼            │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │ │
│  │  │mysqld_export │   │mysqld_export │   │mysqld_export │    │ │
│  │  │   :9104      │   │   :9104      │   │   :9104      │    │ │
│  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘    │ │
│  └─────────┼──────────────────┼──────────────────┼────────────┘ │
│            │                  │                  │              │
│            │ scrape           │                  │              │
│            ▼                  ▼                  ▼              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Prometheus                              │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Time-Series Database                                │  │ │
│  │  │  - Metrics storage                                   │  │ │
│  │  │  - Retention: 30 days                                │  │ │
│  │  │  - Scrape interval: 15s                              │  │ │
│  │  │                                                      │  │ │
│  │  │  ServiceMonitors (auto-discovery)                    │  │ │
│  │  │  └─> mariadb-servicemonitor                          │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Alertmanager                                        │  │ │
│  │  │  - Alert rules evaluation                            │  │ │
│  │  │  - Routing (Slack, PagerDuty, email)                 │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────┬───────────────────────────────────────────┘ │
│                   │                                             │
│                   │ query                                       │
│                   ▼                                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Grafana                                 │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Dashboards                                          │  │ │
│  │  │  - MariaDB Overview                                  │  │ │
│  │  │  - Replication Status                                │  │ │
│  │  │  - Galera Cluster                                    │  │ │
│  │  │  - Query Performance                                 │  │ │
│  │  │  - InnoDB Metrics                                    │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Alerting                                            │  │ │
│  │  │  - Visual alerts on dashboards                       │  │ │
│  │  │  - Integration with Prometheus alerts                │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Composants clés

#### 1. mysqld_exporter

**Rôle** : Exporte métriques MariaDB au format Prometheus

```
MariaDB (port 3306)
    ↓
    ├─> SHOW GLOBAL STATUS
    ├─> SHOW GLOBAL VARIABLES
    ├─> SHOW SLAVE STATUS
    ├─> SELECT * FROM performance_schema.*
    ↓
mysqld_exporter (port 9104)
    ↓
    └─> /metrics (format Prometheus)
```

**Métriques exposées** : 200+ métriques

#### 2. Prometheus

**Rôle** : Collecte, stocke et interroge métriques time-series

**Concepts clés** :

```
┌──────────────────────────────────────────────────────────────┐
│                  Prometheus Concepts                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Scraping:                                                   │
│  - Prometheus pull metrics depuis exporters                  │
│  - Intervalle: toutes les 15-30s (configurable)              │
│                                                              │
│  Storage:                                                    │
│  - TSDB (Time-Series Database) local                         │
│  - Compression efficace                                      │
│  - Retention: 15-30 jours typiquement                        │
│                                                              │
│  Querying:                                                   │
│  - PromQL (Prometheus Query Language)                        │
│  - Agrégations, fonctions, math                              │
│                                                              │
│  Service Discovery:                                          │
│  - Kubernetes: ServiceMonitor, PodMonitor                    │
│  - Auto-discovery des targets                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### 3. Alertmanager

**Rôle** : Gère alertes (routing, grouping, silencing)

**Flow** :
```
Prometheus rules
    ↓ (alert triggered)
Alertmanager
    ├─> Grouping (multiple alerts → une notification)
    ├─> Throttling (éviter spam)
    ├─> Routing (équipe, canal)
    └─> Notification
        ├─> Slack
        ├─> PagerDuty
        ├─> Email
        └─> Webhook
```

#### 4. Grafana

**Rôle** : Visualisation et dashboards

**Features** :
- Dashboards interactifs
- Variables dynamiques
- Alerting visuel
- Templating
- Annotations
- Snapshots

---

## Métriques MariaDB essentielles

### Catégories de métriques

```
┌──────────────────────────────────────────────────────────────┐
│              MariaDB Metrics Categories                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. THROUGHPUT (Débit)                                       │
│     - Queries/second (QPS)                                   │
│     - Transactions/second (TPS)                              │
│     - Read vs Write ratio                                    │
│                                                              │
│  2. LATENCY (Latence)                                        │
│     - Query execution time                                   │
│     - Lock wait time                                         │
│     - Network latency                                        │
│                                                              │
│  3. ERRORS (Erreurs)                                         │
│     - Connection errors                                      │
│     - Aborted connections                                    │
│     - Table lock errors                                      │
│                                                              │
│  4. SATURATION (Saturation)                                  │
│     - Connections (used/max)                                 │
│     - Buffer pool (used/total)                               │
│     - Thread cache                                           │
│                                                              │
│  5. AVAILABILITY (Disponibilité)                             │
│     - Uptime                                                 │
│     - Replication status                                     │
│     - Galera cluster size                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Top 20 métriques critiques

| Métrique | Description | Seuil alerte | Priorité |
|----------|-------------|--------------|----------|
| **mysql_up** | MariaDB est-il up? | 0 (down) | 🔴 P0 |
| **mysql_global_status_threads_connected** | Connexions actives | >80% max_connections | 🔴 P0 |
| **mysql_global_status_queries** | Queries/second | Baisse soudaine >50% | 🟠 P1 |
| **mysql_global_status_slow_queries** | Slow queries | >10/minute | 🟠 P1 |
| **mysql_global_status_aborted_connects** | Connexions avortées | >100/minute | 🟠 P1 |
| **mysql_global_status_innodb_buffer_pool_read_requests** | InnoDB reads | Monitoring | 🟡 P2 |
| **mysql_global_status_innodb_buffer_pool_reads** | InnoDB disk reads | >1000/s | 🟠 P1 |
| **mysql_global_status_innodb_row_lock_waits** | Row lock waits | >100/s | 🟠 P1 |
| **mysql_global_status_table_locks_waited** | Table lock waits | >10/s | 🟡 P2 |
| **mysql_global_status_select** | SELECT queries | Monitoring | 🟢 P3 |
| **mysql_global_status_insert** | INSERT queries | Monitoring | 🟢 P3 |
| **mysql_global_status_update** | UPDATE queries | Monitoring | 🟢 P3 |
| **mysql_global_status_delete** | DELETE queries | Monitoring | 🟢 P3 |
| **mysql_slave_status_seconds_behind_master** | Replication lag | >60s | 🔴 P0 |
| **mysql_slave_status_slave_io_running** | Replication IO thread | 0 (stopped) | 🔴 P0 |
| **mysql_slave_status_slave_sql_running** | Replication SQL thread | 0 (stopped) | 🔴 P0 |
| **mysql_global_status_wsrep_cluster_size** | Galera cluster size | <3 (quorum lost) | 🔴 P0 |
| **mysql_global_status_wsrep_local_state** | Galera node state | ≠4 (not Synced) | 🟠 P1 |
| **mysql_global_variables_max_connections** | Max connections config | Monitoring | 🟢 P3 |
| **mysql_global_variables_innodb_buffer_pool_size** | Buffer pool size | Monitoring | 🟢 P3 |

### Métriques dérivées (calculs)

**Calculs importants avec PromQL** :

```promql
# 1. QPS (Queries Per Second)
rate(mysql_global_status_queries[5m])

# 2. Buffer pool hit ratio (devrait être >99%)
(
  mysql_global_status_innodb_buffer_pool_read_requests
  - 
  mysql_global_status_innodb_buffer_pool_reads
) 
/ 
mysql_global_status_innodb_buffer_pool_read_requests * 100

# 3. Connection utilization (%)
mysql_global_status_threads_connected 
/ 
mysql_global_variables_max_connections * 100

# 4. Slow query rate
rate(mysql_global_status_slow_queries[5m])

# 5. Read/Write ratio
sum(rate(mysql_global_status_select[5m]))
/
sum(rate(mysql_global_status_insert[5m]) + rate(mysql_global_status_update[5m]) + rate(mysql_global_status_delete[5m]))

# 6. Table cache hit ratio
(
  mysql_global_status_open_tables 
  / 
  mysql_global_status_opened_tables
) * 100

# 7. Thread cache hit ratio
(
  1 - 
  (
    mysql_global_status_threads_created 
    / 
    mysql_global_status_connections
  )
) * 100
```

---

## Déploiement sur Kubernetes

### 1. Installer Prometheus Operator

**Via Helm (kube-prometheus-stack)** :

```bash
# Ajouter repo Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Installer stack complète (Prometheus + Grafana + Alertmanager)
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set grafana.adminPassword='admin-secure-password'
```

**Vérifier installation** :

```bash
kubectl get pods -n monitoring

# Output:
# NAME                                                     READY   STATUS
# prometheus-kube-prometheus-stack-prometheus-0            2/2     Running
# kube-prometheus-stack-grafana-xxxxx                      3/3     Running
# kube-prometheus-stack-operator-xxxxx                     1/1     Running
# alertmanager-kube-prometheus-stack-alertmanager-0        2/2     Running
```

### 2. Déployer mysqld_exporter

**Sidecar pattern (recommandé)** :

```yaml
# mariadb-with-exporter.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  namespace: databases
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
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9104"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      # MariaDB container
      - name: mariadb
        image: mariadb:11.8
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
      
      # mysqld_exporter sidecar
      - name: mysqld-exporter
        image: prom/mysqld-exporter:v0.15.1
        ports:
        - containerPort: 9104
          name: metrics
        env:
        - name: DATA_SOURCE_NAME
          value: "exporter:$(EXPORTER_PASSWORD)@(localhost:3306)/"
        - name: EXPORTER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-exporter-secret
              key: password
        
        # Configuration exporter
        args:
        - --collect.global_status
        - --collect.global_variables
        - --collect.slave_status
        - --collect.info_schema.innodb_metrics
        - --collect.info_schema.processlist
        - --collect.info_schema.tables
        - --collect.perf_schema.tableiowaits
        - --collect.perf_schema.tablelocks
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        
        livenessProbe:
          httpGet:
            path: /
            port: 9104
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 9104
          initialDelaySeconds: 10
          periodSeconds: 5
  
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

**Créer utilisateur exporter dans MariaDB** :

```sql
-- Se connecter à MariaDB
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'exporter-password';

-- Permissions minimales requises
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'localhost';
GRANT SELECT ON performance_schema.* TO 'exporter'@'localhost';

-- 🆕 MariaDB 11.8: Performance schema extended
GRANT SELECT ON mysql.* TO 'exporter'@'localhost';

FLUSH PRIVILEGES;
```

**Secret pour exporter** :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-exporter-secret
  namespace: databases
type: Opaque
stringData:
  password: "exporter-password"
```

### 3. ServiceMonitor (auto-discovery)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mariadb-metrics
  namespace: databases
  labels:
    app: mariadb
    release: kube-prometheus-stack  # Important: label pour discovery
spec:
  selector:
    matchLabels:
      app: mariadb
  
  endpoints:
  - port: metrics
    interval: 30s
    scrapeTimeout: 10s
    path: /metrics
    
    # Relabeling pour ajouter labels utiles
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: namespace
    - sourceLabels: [__meta_kubernetes_pod_label_statefulset_kubernetes_io_pod_name]
      regex: mariadb-0
      targetLabel: role
      replacement: primary
    - sourceLabels: [__meta_kubernetes_pod_label_statefulset_kubernetes_io_pod_name]
      regex: mariadb-[1-9]
      targetLabel: role
      replacement: replica
```

**Service pour métriques** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb-metrics
  namespace: databases
  labels:
    app: mariadb
spec:
  clusterIP: None  # Headless
  ports:
  - name: metrics
    port: 9104
    targetPort: 9104
  selector:
    app: mariadb
```

### 4. Vérifier collecte des métriques

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Ouvrir browser: http://localhost:9090
# Tester query: mysql_up
# Devrait retourner: mysql_up{pod="mariadb-0",...} 1
```

---

## Dashboards Grafana

### Dashboard MariaDB Overview

**Import via Grafana UI** :

1. Dashboard ID communautaire : **7362** (MySQL Overview by Percona)
2. Ou créer custom dashboard :

```json
{
  "dashboard": {
    "title": "MariaDB Overview",
    "tags": ["mariadb", "database"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "MariaDB Status",
        "type": "stat",
        "targets": [
          {
            "expr": "mysql_up",
            "legendFormat": "{{ pod }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {
                "type": "value",
                "value": "0",
                "text": "DOWN",
                "color": "red"
              },
              {
                "type": "value",
                "value": "1",
                "text": "UP",
                "color": "green"
              }
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "Queries Per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mysql_global_status_queries[5m])",
            "legendFormat": "{{ pod }}"
          }
        ]
      },
      {
        "id": 3,
        "title": "Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "mysql_global_status_threads_connected",
            "legendFormat": "Connected - {{ pod }}"
          },
          {
            "expr": "mysql_global_variables_max_connections",
            "legendFormat": "Max connections"
          }
        ]
      },
      {
        "id": 4,
        "title": "Slow Queries Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mysql_global_status_slow_queries[5m])",
            "legendFormat": "{{ pod }}"
          }
        ]
      },
      {
        "id": 5,
        "title": "InnoDB Buffer Pool Hit Ratio",
        "type": "gauge",
        "targets": [
          {
            "expr": "(mysql_global_status_innodb_buffer_pool_read_requests - mysql_global_status_innodb_buffer_pool_reads) / mysql_global_status_innodb_buffer_pool_read_requests * 100",
            "legendFormat": "{{ pod }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                { "value": 0, "color": "red" },
                { "value": 95, "color": "yellow" },
                { "value": 99, "color": "green" }
              ]
            }
          }
        }
      },
      {
        "id": 6,
        "title": "Replication Lag",
        "type": "graph",
        "targets": [
          {
            "expr": "mysql_slave_status_seconds_behind_master",
            "legendFormat": "{{ pod }}"
          }
        ]
      }
    ]
  }
}
```

### Dashboard variables (pour multi-instance)

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(mysql_up, namespace)",
        "current": {
          "value": "databases"
        }
      },
      {
        "name": "instance",
        "type": "query",
        "query": "label_values(mysql_up{namespace=\"$namespace\"}, pod)",
        "current": {
          "value": "All"
        },
        "multi": true,
        "includeAll": true
      }
    ]
  }
}
```

**Utilisation dans panels** :

```promql
# Au lieu de:
mysql_global_status_threads_connected

# Utiliser:
mysql_global_status_threads_connected{namespace="$namespace", pod=~"$instance"}
```

---

## Alerting

### PrometheusRule (Kubernetes)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mariadb-alerts
  namespace: databases
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: mariadb
    interval: 30s
    rules:
    
    # Alert 1: MariaDB Down
    - alert: MariaDBDown
      expr: mysql_up == 0
      for: 1m
      labels:
        severity: critical
        component: mariadb
      annotations:
        summary: "MariaDB instance down"
        description: "MariaDB instance {{ $labels.pod }} in namespace {{ $labels.namespace }} is down for more than 1 minute."
    
    # Alert 2: High connection usage
    - alert: MariaDBHighConnectionUsage
      expr: |
        (
          mysql_global_status_threads_connected 
          / 
          mysql_global_variables_max_connections
        ) * 100 > 80
      for: 5m
      labels:
        severity: warning
        component: mariadb
      annotations:
        summary: "High connection usage"
        description: "MariaDB instance {{ $labels.pod }} is using {{ $value }}% of max connections."
    
    # Alert 3: Too many slow queries
    - alert: MariaDBTooManySlowQueries
      expr: rate(mysql_global_status_slow_queries[5m]) > 10
      for: 5m
      labels:
        severity: warning
        component: mariadb
      annotations:
        summary: "Too many slow queries"
        description: "MariaDB instance {{ $labels.pod }} has {{ $value }} slow queries per second."
    
    # Alert 4: Replication lag
    - alert: MariaDBReplicationLag
      expr: mysql_slave_status_seconds_behind_master > 60
      for: 5m
      labels:
        severity: warning
        component: mariadb
      annotations:
        summary: "Replication lag detected"
        description: "MariaDB replica {{ $labels.pod }} is {{ $value }} seconds behind master."
    
    # Alert 5: Replication stopped
    - alert: MariaDBReplicationStopped
      expr: |
        mysql_slave_status_slave_io_running == 0
        or
        mysql_slave_status_slave_sql_running == 0
      for: 1m
      labels:
        severity: critical
        component: mariadb
      annotations:
        summary: "Replication stopped"
        description: "MariaDB replication on {{ $labels.pod }} has stopped."
    
    # Alert 6: Galera cluster size
    - alert: GaleraClusterSizeReduced
      expr: mysql_global_status_wsrep_cluster_size < 3
      for: 5m
      labels:
        severity: critical
        component: mariadb-galera
      annotations:
        summary: "Galera cluster size reduced"
        description: "Galera cluster has only {{ $value }} nodes (expected 3+). Quorum may be at risk."
    
    # Alert 7: Galera node not synced
    - alert: GaleraNodeNotSynced
      expr: mysql_global_status_wsrep_local_state != 4
      for: 5m
      labels:
        severity: warning
        component: mariadb-galera
      annotations:
        summary: "Galera node not synced"
        description: "Galera node {{ $labels.pod }} is not in Synced state (state={{ $value }})."
    
    # Alert 8: InnoDB buffer pool low hit ratio
    - alert: InnoDBBufferPoolLowHitRatio
      expr: |
        (
          (
            mysql_global_status_innodb_buffer_pool_read_requests
            - 
            mysql_global_status_innodb_buffer_pool_reads
          ) 
          / 
          mysql_global_status_innodb_buffer_pool_read_requests
        ) * 100 < 95
      for: 15m
      labels:
        severity: warning
        component: mariadb
      annotations:
        summary: "Low InnoDB buffer pool hit ratio"
        description: "InnoDB buffer pool hit ratio on {{ $labels.pod }} is {{ $value }}% (should be >99%)."
    
    # Alert 9: Aborted connections
    - alert: MariaDBAbortedConnections
      expr: rate(mysql_global_status_aborted_connects[5m]) > 10
      for: 5m
      labels:
        severity: warning
        component: mariadb
      annotations:
        summary: "High rate of aborted connections"
        description: "MariaDB instance {{ $labels.pod }} has {{ $value }} aborted connections per second."
    
    # Alert 10: Disk space low (requires node_exporter)
    - alert: MariaDBDiskSpaceLow
      expr: |
        (
          node_filesystem_avail_bytes{mountpoint="/var/lib/mysql"} 
          / 
          node_filesystem_size_bytes{mountpoint="/var/lib/mysql"}
        ) * 100 < 10
      for: 5m
      labels:
        severity: critical
        component: mariadb
      annotations:
        summary: "Low disk space for MariaDB"
        description: "MariaDB data directory has less than 10% free space."
```

### Alertmanager configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    route:
      receiver: 'default'
      group_by: ['alertname', 'cluster', 'namespace']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      
      routes:
      # Critical alerts → PagerDuty
      - match:
          severity: critical
        receiver: 'pagerduty'
        continue: true
      
      # MariaDB alerts → DBA team Slack
      - match:
          component: mariadb
        receiver: 'dba-slack'
      
      # Galera alerts → Platform team
      - match:
          component: mariadb-galera
        receiver: 'platform-slack'
    
    receivers:
    - name: 'default'
      slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'dba-slack'
      slack_configs:
      - channel: '#dba-team'
        title: '🗄️ MariaDB Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Severity:* {{ .Labels.severity }}
          *Summary:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          {{ end }}
    
    - name: 'platform-slack'
      slack_configs:
      - channel: '#platform-team'
        title: '☸️ Galera Cluster Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR-PAGERDUTY-SERVICE-KEY'
        description: '{{ .GroupLabels.alertname }}'
```

---

## Monitoring multi-environnements

### Labels pour différencier environnements

```yaml
# Production
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  namespace: production
  labels:
    app: mariadb
    environment: production
    tier: database

---
# Staging
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  namespace: staging
  labels:
    app: mariadb
    environment: staging
    tier: database
```

**Queries PromQL par environnement** :

```promql
# Toutes les instances production
mysql_up{environment="production"}

# QPS production vs staging
sum(rate(mysql_global_status_queries{environment="production"}[5m]))
sum(rate(mysql_global_status_queries{environment="staging"}[5m]))

# Alertes seulement sur production
mysql_global_status_threads_connected{environment="production"} 
/ 
mysql_global_variables_max_connections{environment="production"} > 0.8
```

### Federation (multi-cluster)

**Scénario** : Prometheus par cluster, Grafana centralisé

```
┌─────────────────────────────────────────────────────────┐
│                  Multi-Cluster Monitoring               │
│                                                         │
│  ┌──────────────────┐  ┌──────────────────┐             │
│  │   Cluster AWS    │  │   Cluster GCP    │             │
│  │                  │  │                  │             │
│  │  Prometheus-AWS  │  │  Prometheus-GCP  │             │
│  └────────┬─────────┘  └────────┬─────────┘             │
│           │                     │                       │
│           │ federate            │ federate              │
│           ▼                     ▼                       │
│  ┌──────────────────────────────────────────┐           │
│  │     Prometheus Central (Federation)      │           │
│  └──────────────────┬───────────────────────┘           │
│                     │                                   │
│                     ▼                                   │
│  ┌──────────────────────────────────────────┐           │
│  │          Grafana (Centralisé)            │           │
│  └──────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

**Configuration federation** :

```yaml
# prometheus-central.yaml
scrape_configs:
- job_name: 'federate-aws'
  scrape_interval: 30s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
    - '{job="mariadb"}'
  static_configs:
  - targets:
    - 'prometheus-aws.monitoring.svc.cluster.local:9090'

- job_name: 'federate-gcp'
  scrape_interval: 30s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
    - '{job="mariadb"}'
  static_configs:
  - targets:
    - 'prometheus-gcp.monitoring.svc.cluster.local:9090'
```

---

## Best practices

### 1. Rétention des métriques

```yaml
# Prometheus retention policy
prometheus:
  prometheusSpec:
    retention: 30d  # 30 jours de métriques
    retentionSize: 50GB  # Ou limite par taille
    
    # Pour long-term storage, utiliser Thanos ou Cortex
```

**Long-term storage avec Thanos** :

```
Prometheus (retention 7d)
    ↓
Thanos Sidecar
    ↓
S3 / GCS (retention 1 an)
    ↓
Thanos Query (unified view)
    ↓
Grafana
```

### 2. Scrape intervals

```yaml
# Recommandations
global:
  scrape_interval: 30s  # Standard
  scrape_timeout: 10s

# Ajuster selon besoins
scrape_configs:
- job_name: 'mariadb-metrics'
  scrape_interval: 15s  # Production critique
  
- job_name: 'mariadb-dev'
  scrape_interval: 60s  # Dev (moins critique)
```

⚠️ **Trade-off** : Intervalle court = plus précis MAIS plus de données + charge

### 3. High cardinality attention

**Éviter** :

```promql
# ❌ MAUVAIS: label avec valeur unique par query
mysql_query_duration{query_id="12345"}  # Des millions de series

# ❌ MAUVAIS: user_id comme label
mysql_connections{user_id="user123"}
```

**Préférer** :

```promql
# ✅ BON: labels avec cardinalité limitée
mysql_query_duration{database="myapp", query_type="select"}

# ✅ BON: agrégation
sum(mysql_connections) by (database)
```

### 4. Recording rules (pré-calcul)

**Pour queries complexes fréquentes** :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mariadb-recording-rules
spec:
  groups:
  - name: mariadb-precomputed
    interval: 30s
    rules:
    
    # Pré-calculer buffer pool hit ratio
    - record: mariadb:buffer_pool_hit_ratio:percentage
      expr: |
        (
          mysql_global_status_innodb_buffer_pool_read_requests
          - 
          mysql_global_status_innodb_buffer_pool_reads
        ) 
        / 
        mysql_global_status_innodb_buffer_pool_read_requests * 100
    
    # Pré-calculer QPS par instance
    - record: mariadb:queries_per_second:rate5m
      expr: rate(mysql_global_status_queries[5m])
```

**Utilisation** :

```promql
# Au lieu de query complexe
# Utiliser recording rule pré-calculée
mariadb:buffer_pool_hit_ratio:percentage
```

### 5. Annotations dans Grafana

**Marquer événements importants** :

```yaml
# Annotation via API lors d'un deployment
curl -X POST \
  http://grafana:3000/api/annotations \
  -H 'Content-Type: application/json' \
  -d '{
    "dashboardId": 1,
    "time": '$(date +%s000)',
    "tags": ["deployment", "mariadb"],
    "text": "MariaDB migration V042 deployed"
  }'
```

Visible sur dashboards pour corréler changements ↔ métriques

---

## ✅ Points clés à retenir

- **Observabilité = Metrics + Logs + Traces** : Focus ici sur metrics (Prometheus)
- **mysqld_exporter** : Exporte 200+ métriques MariaDB au format Prometheus
- **ServiceMonitor** : Auto-discovery des targets dans Kubernetes
- **Top métriques** : Up, Connections, QPS, Slow queries, Replication lag, Galera status
- **Dashboards** : Variables pour multi-instance, panels essentiels
- **Alerting** : PrometheusRule pour Kubernetes, routing Alertmanager
- **Multi-env** : Labels (environment, tier) pour différencier
- **Best practices** : Retention 30d, scrape 15-30s, éviter high cardinality, recording rules
- **Long-term** : Thanos/Cortex pour storage >1 an
- **Integration** : Annotations pour corréler deployments ↔ metrics

💡 **Golden rule** : Monitor what matters. Trop de métriques = bruit, trop peu = blind spots.

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Prometheus](https://prometheus.io/docs/)
- [📖 Grafana](https://grafana.com/docs/)
- [📖 mysqld_exporter](https://github.com/prometheus/mysqld_exporter)
- [📖 Prometheus Operator](https://prometheus-operator.dev/)

### Dashboards communautaires
- [📊 MySQL Overview (7362)](https://grafana.com/grafana/dashboards/7362)
- [📊 MariaDB Galera Cluster](https://grafana.com/grafana/dashboards/13106)
- [📊 MySQL Replication](https://grafana.com/grafana/dashboards/7371)

### Articles de référence
- [📝 Monitoring MySQL with Prometheus](https://www.percona.com/blog/2020/07/30/monitoring-mysql-with-prometheus/)
- [📝 SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

---

## ➡️ Sections suivantes

**16.9.1 mysqld_exporter** : Configuration détaillée, collectors disponibles, customisation, troubleshooting.

**16.9.2 Dashboards Grafana avancés** : Création custom, variables, templating, panels complexes.

**16.9.3 Alerting avancé** : Routing complexe, inhibition, silencing, escalation.

**16.10 Observabilité complète** : Logs (Loki), traces (Jaeger), correlation logs↔metrics↔traces.

---

**MariaDB** : Version 11.8 LTS
**Prometheus** : v2.48+
**Grafana** : v10.2+

⏭️ [mysqld_exporter](/16-devops-automatisation/09.1-mysqld-exporter.md)

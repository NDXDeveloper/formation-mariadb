🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.10 Observabilité : Logs, Metrics, Traces

> **Niveau** : Expert  
> **Durée estimée** : 8-9 heures  
> **Prérequis** : 
> - Section 16.9 Monitoring Prometheus/Grafana maîtrisée
> - Compréhension des concepts d'observabilité
> - Expérience avec analyse de logs et debugging
> - Notions de distributed tracing
> - Familiarité avec ELK stack ou Loki

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les 3 piliers de l'observabilité et leur complémentarité
- **Collecter** et centraliser tous les logs MariaDB (error, slow query, audit)
- **Analyser** les logs avec Loki ou ELK stack
- **Corréler** logs et metrics dans Grafana
- **Implémenter** distributed tracing pour requêtes database
- **Diagnostiquer** incidents complexes via observabilité complète
- **Créer** des dashboards unifiés (logs + metrics + traces)
- **Appliquer** observability-driven development

---

## Introduction

### Les 3 piliers de l'observabilité revisités

Dans la section 16.9, nous avons vu **Metrics** (Prometheus). Maintenant, explorons **Logs** et **Traces**, puis leur **corrélation**.

```
┌─────────────────────────────────────────────────────────────────┐
│           Observability: Complete Picture                       │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  1️⃣  METRICS (What is happening?)                          │ │
│  │  ────────────────────────────────────────────────────────  │ │
│  │  Agrégations numériques dans le temps                      │ │
│  │                                                            │ │
│  │  Exemple: CPU usage = 87% (at timestamp T)                 │ │
│  │                                                            │ │
│  │  Questions répondues:                                      │ │
│  │  - Système en bonne santé?                                 │ │
│  │  - Performance dégradée?                                   │ │
│  │  - Tendances (scaling needed?)                             │ │
│  │                                                            │ │
│  │  Limites:                                                  │ │
│  │  ❌ Ne dit pas POURQUOI problème                           │ │
│  │  ❌ Pas de contexte détaillé                               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  2️⃣  LOGS (Why is it happening?)                           │ │
│  │  ────────────────────────────────────────────────────────  │ │
│  │  Événements discrets avec contexte riche                   │ │
│  │                                                            │ │
│  │  Exemple:                                                  │ │
│  │  [ERROR] Table 'users' doesn't exist                       │ │
│  │  [WARN] Aborted connection (Got timeout reading)           │ │
│  │                                                            │ │
│  │  Questions répondues:                                      │ │
│  │  - Quel est le message d'erreur exact?                     │ │
│  │  - Quand est-ce arrivé (timestamp précis)?                 │ │
│  │  - Quel utilisateur/query?                                 │ │
│  │                                                            │ │
│  │  Limites:                                                  │ │
│  │  ❌ Verbose (beaucoup de bruit)                            │ │
│  │  ❌ Difficile de voir tendances                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  3️⃣  TRACES (How did it flow through system?)              │ │
│  │  ────────────────────────────────────────────────────────  │ │
│  │  Suivi d'une requête à travers composants distribués       │ │
│  │                                                            │ │
│  │  Exemple:                                                  │ │
│  │  API (50ms) → App (120ms) → MariaDB (800ms) → Total: 970ms │ │
│  │                                                            │ │
│  │  Questions répondues:                                      │ │
│  │  - Quel composant est lent?                                │ │
│  │  - Quel est le chemin critique?                            │ │
│  │  - Dépendances entre services?                             │ │
│  │                                                            │ │
│  │  Limites:                                                  │ │
│  │  ❌ Complexité instrumentation                             │ │
│  │  ❌ Overhead performance                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  🔗 CORRELATION = SUPER POWER                              │ │
│  │  ────────────────────────────────────────────────────────  │ │
│  │  Metrics → Alerte "Slow queries spike"                     │ │
│  │      ↓                                                     │ │
│  │  Logs → "Table lock timeout on orders table"               │ │
│  │      ↓                                                     │ │
│  │  Traces → "Request from API v2.1 taking 5s on checkout"    │ │
│  │      ↓                                                     │ │
│  │  Root cause: New query in API v2.1 missing index           │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Observabilité vs Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│           Monitoring (traditionnel)                          │
├──────────────────────────────────────────────────────────────┤
│  Focus: "Système up ou down?"                                │
│  Approche: Prédéfinir métriques et alertes                   │
│  Questions: Connues à l'avance                               │
│  Exemple: "CPU >80%" → alerte                                │
│                                                              │
│  Limitation: Ne peut détecter que ce qu'on a prévu           │
└──────────────────────────────────────────────────────────────┘

                           VS

┌──────────────────────────────────────────────────────────────┐
│           Observability (moderne)                            │
├──────────────────────────────────────────────────────────────┤
│  Focus: "Pourquoi comportement inattendu?"                   │
│  Approche: Collecter TOUT, interroger dynamiquement          │
│  Questions: Posées APRÈS incident (inconnues au départ)      │
│  Exemple: "Pourquoi latence spike à 14h23 sur user_id=123?"  │
│          → Drill down logs + traces + metrics                │
│                                                              │
│  Avantage: Peut investiguer l'imprévisible                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Pilier 1 : Logs MariaDB

### Types de logs MariaDB

```
┌──────────────────────────────────────────────────────────────┐
│                    MariaDB Log Types                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. ERROR LOG                                                │
│     Localisation: /var/log/mysql/error.log                   │
│     Contenu: Erreurs serveur, warnings, startup/shutdown     │
│     Utilité: Debugging crashes, config errors                │
│                                                              │
│  2. SLOW QUERY LOG                                           │
│     Localisation: /var/log/mysql/slow.log                    │
│     Contenu: Requêtes dépassant long_query_time              │
│     Utilité: Performance optimization                        │
│                                                              │
│  3. GENERAL QUERY LOG                                        │
│     Localisation: /var/log/mysql/general.log                 │
│     Contenu: TOUTES les requêtes (⚠️ très verbose)           │
│     Utilité: Debugging, audit (dev only)                     │
│                                                              │
│  4. BINARY LOG                                               │
│     Localisation: /var/lib/mysql/mysql-bin.000001            │
│     Contenu: Changements de données (DML)                    │
│     Utilité: Réplication, point-in-time recovery             │
│                                                              │
│  5. AUDIT LOG (🆕 Plugin MariaDB Audit)                      │
│     Localisation: /var/log/mysql/audit.log                   │
│     Contenu: Connexions, queries (compliance)                │
│     Utilité: Sécurité, compliance (GDPR, SOC2)               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Configuration des logs

**my.cnf configuration** :

```ini
[mysqld]
# 1. Error log
log_error = /var/log/mysql/error.log
log_warnings = 2  # 0=off, 1=important, 2=all

# 2. Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2  # secondes
log_queries_not_using_indexes = 1
min_examined_row_limit = 1000  # Éviter bruit

# 3. General query log (⚠️ DEV ONLY)
general_log = 0  # OFF en production (trop verbose)
general_log_file = /var/log/mysql/general.log

# 4. Binary log
log_bin = /var/lib/mysql/mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 100M

# 🆕 MariaDB 11.8 - Binary log compression
binlog_row_image = MINIMAL  # Réduit taille binlog
```

**Audit plugin (Enterprise)** :

```sql
-- Installer plugin
INSTALL SONAME 'server_audit';

-- Configuration
SET GLOBAL server_audit_logging = ON;
SET GLOBAL server_audit_events = 'CONNECT,QUERY_DDL,QUERY_DML';
SET GLOBAL server_audit_file_path = '/var/log/mysql/audit.log';
SET GLOBAL server_audit_file_rotate_size = 1000000;  # 1MB
SET GLOBAL server_audit_file_rotations = 9;

-- Exclure utilisateurs (monitoring)
SET GLOBAL server_audit_excl_users = 'exporter,grafana';
```

### Exemple de logs

**Error log** :

```
2025-12-14 10:23:45 0 [Note] Server socket created on IP: '0.0.0.0'.
2025-12-14 10:23:45 0 [Note] /usr/sbin/mysqld: ready for connections.
2025-12-14 11:15:32 123 [Warning] Aborted connection 123 to db: 'myapp' user: 'appuser' host: '10.0.1.5' (Got timeout reading communication packets)
2025-12-14 11:30:12 456 [ERROR] Table 'myapp.orders_old' doesn't exist
```

**Slow query log** :

```
# Time: 2025-12-14T11:45:32.123456Z
# User@Host: appuser[appuser] @ app-pod-5 [10.0.1.10]
# Thread_id: 789  Schema: myapp  QC_hit: No
# Query_time: 5.234567  Lock_time: 0.000123  Rows_sent: 1234  Rows_examined: 567890
SET timestamp=1734177932;
SELECT o.*, u.email, u.name 
FROM orders o 
JOIN users u ON o.user_id = u.id 
WHERE o.created_at > '2024-01-01' 
ORDER BY o.created_at DESC;
```

**Audit log** :

```json
{
  "timestamp": "2025-12-14T12:00:00.123Z",
  "event": "CONNECT",
  "user": "admin",
  "host": "10.0.2.5",
  "database": "myapp",
  "status": 0
}
{
  "timestamp": "2025-12-14T12:00:05.456Z",
  "event": "QUERY",
  "user": "admin",
  "host": "10.0.2.5",
  "database": "myapp",
  "query": "DROP TABLE old_data",
  "status": 0
}
```

---

## Stack de logging : Loki vs ELK

### Comparaison

| Aspect | Loki (Grafana Labs) | ELK (Elasticsearch, Logstash, Kibana) |
|--------|---------------------|--------------------------------------|
| **Philosophie** | Logs comme metrics (labels) | Full-text indexing |
| **Storage** | Objet storage (S3, GCS) | Elasticsearch indices |
| **Query language** | LogQL (like PromQL) | Lucene query syntax |
| **Resource usage** | ⭐⭐⭐⭐⭐ Léger | ⭐⭐ Lourd (RAM hungry) |
| **Cost** | 💰 Faible | 💰💰💰 Élevé |
| **Learning curve** | ⭐⭐ Facile (si connaît PromQL) | ⭐⭐⭐⭐ Difficile |
| **Full-text search** | ⚠️ Limité | ✅ Excellent |
| **Grafana integration** | ✅ Native | ⚠️ Via plugin |
| **Scalability** | ✅ Horizontal | ✅ Horizontal (complexe) |
| **Use case** | Cloud-native, Kubernetes | Enterprise, compliance stricte |

**Recommandation** :
- **Loki** : Kubernetes, intégration Grafana, budget limité
- **ELK** : Compliance stricte, full-text search crucial, infrastructure existante

### Architecture Loki

```
┌─────────────────────────────────────────────────────────────────┐
│                    Loki Architecture                            │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  MariaDB Pods                              │ │
│  │                                                            │ │
│  │  /var/log/mysql/error.log                                  │ │
│  │  /var/log/mysql/slow.log                                   │ │
│  │  /var/log/mysql/audit.log                                  │ │
│  └───────────────────┬────────────────────────────────────────┘ │
│                      │                                          │
│                      │ tail logs                                │
│                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │               Promtail (agent)                             │ │
│  │  - Scrape logs from pods                                   │ │
│  │  - Parse and label                                         │ │
│  │  - Push to Loki                                            │ │
│  └───────────────────┬────────────────────────────────────────┘ │
│                      │                                          │
│                      │ HTTP push                                │
│                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Loki                                    │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Distributor (receive logs)                          │  │ │
│  │  └──────────────┬───────────────────────────────────────┘  │ │
│  │                 │                                          │ │
│  │  ┌──────────────▼───────────────────────────────────────┐  │ │
│  │  │  Ingester (index + chunk logs)                       │  │ │
│  │  └──────────────┬───────────────────────────────────────┘  │ │
│  │                 │                                          │ │
│  │  ┌──────────────▼───────────────────────────────────────┐  │ │
│  │  │  Storage (S3, GCS, local)                            │  │ │
│  │  │  - Chunks (compressed logs)                          │  │ │
│  │  │  - Index (time + labels)                             │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Querier (query logs)                                │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └───────────────────┬────────────────────────────────────────┘ │
│                      │                                          │
│                      │ LogQL queries                            │
│                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Grafana                                  │ │
│  │  - Explore logs                                            │ │
│  │  - Dashboards (logs + metrics)                             │ │
│  │  - Alerting on log patterns                                │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Déploiement Loki sur Kubernetes

**1. Installer Loki stack via Helm** :

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=100Gi \
  --set promtail.enabled=true \
  --set grafana.enabled=false  # Déjà installé via kube-prometheus-stack
```

**2. Configuration Promtail pour MariaDB** :

```yaml
# promtail-mariadb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-mariadb-config
  namespace: databases
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
      grpc_listen_port: 0
    
    positions:
      filename: /tmp/positions.yaml
    
    clients:
    - url: http://loki.logging.svc.cluster.local:3100/loki/api/v1/push
    
    scrape_configs:
    # Error log
    - job_name: mariadb-error
      static_configs:
      - targets:
          - localhost
        labels:
          job: mariadb
          log_type: error
          __path__: /var/log/mysql/error.log
      
      # Pipeline pour parser error log
      pipeline_stages:
      - regex:
          expression: '^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) (?P<thread_id>\d+) \[(?P<level>\w+)\] (?P<message>.*)'
      - labels:
          level:
          thread_id:
      - timestamp:
          source: timestamp
          format: '2006-01-02 15:04:05'
    
    # Slow query log
    - job_name: mariadb-slow
      static_configs:
      - targets:
          - localhost
        labels:
          job: mariadb
          log_type: slow_query
          __path__: /var/log/mysql/slow.log
      
      # Pipeline pour parser slow query log
      pipeline_stages:
      - multiline:
          firstline: '^\# Time:'
          max_wait_time: 3s
      - regex:
          expression: '# Time: (?P<timestamp>.*)\n# User@Host: (?P<user>\S+)\[(?P<user2>\S+)\] @ (?P<host>\S+) \[(?P<ip>[\d\.]+)\]\n# Thread_id: (?P<thread_id>\d+)  Schema: (?P<schema>\S+).*\n# Query_time: (?P<query_time>[\d\.]+)  Lock_time: (?P<lock_time>[\d\.]+)  Rows_sent: (?P<rows_sent>\d+)  Rows_examined: (?P<rows_examined>\d+)'
      - labels:
          user:
          schema:
      - metrics:
          query_time:
            type: Histogram
            description: "Query execution time"
            source: query_time
            config:
              buckets: [0.1, 0.5, 1, 2, 5, 10]
    
    # Audit log (JSON format)
    - job_name: mariadb-audit
      static_configs:
      - targets:
          - localhost
        labels:
          job: mariadb
          log_type: audit
          __path__: /var/log/mysql/audit.log
      
      pipeline_stages:
      - json:
          expressions:
            timestamp: timestamp
            event: event
            user: user
            host: host
            database: database
            query: query
      - labels:
          event:
          user:
          database:
      - timestamp:
          source: timestamp
          format: RFC3339
```

**3. DaemonSet Promtail** :

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail-mariadb
  namespace: databases
spec:
  selector:
    matchLabels:
      app: promtail-mariadb
  template:
    metadata:
      labels:
        app: promtail-mariadb
    spec:
      serviceAccountName: promtail
      containers:
      - name: promtail
        image: grafana/promtail:2.9.3
        args:
        - -config.file=/etc/promtail/promtail.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: mariadb-logs
          mountPath: /var/log/mysql
          readOnly: true
        - name: positions
          mountPath: /tmp
        ports:
        - containerPort: 9080
          name: http-metrics
      volumes:
      - name: config
        configMap:
          name: promtail-mariadb-config
      - name: mariadb-logs
        hostPath:
          path: /var/log/mysql  # Ou volume partagé avec MariaDB
      - name: positions
        emptyDir: {}
```

**4. Queries LogQL dans Grafana** :

```logql
# Tous les logs MariaDB
{job="mariadb"}

# Erreurs seulement
{job="mariadb", level="ERROR"}

# Slow queries >5s
{job="mariadb", log_type="slow_query"} | json | query_time > 5

# Audit: Connexions d'un utilisateur spécifique
{job="mariadb", log_type="audit", event="CONNECT", user="admin"}

# Rate d'erreurs
sum(rate({job="mariadb", level="ERROR"}[5m]))

# Top 10 slow queries
topk(10, 
  sum by (query) (
    rate({job="mariadb", log_type="slow_query"}[5m])
  )
)
```

---

## Pilier 2 : Metrics (rappel)

**Déjà couvert en section 16.9**, mais intégration avec logs :

```
┌──────────────────────────────────────────────────────────────┐
│          Metrics → Logs Correlation                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Scenario: Alerte "Slow queries spike"                       │
│                                                              │
│  1. Metric alerte:                                           │
│     rate(mysql_global_status_slow_queries[5m]) > 10          │
│                                                              │
│  2. Cliquer sur alerte dans Grafana                          │
│     → Ouvre dashboard avec timestamp de l'alerte             │
│                                                              │
│  3. Panel logs (Loki) avec même timestamp:                   │
│     {job="mariadb", log_type="slow_query"}                   │
│     | json                                                   │
│     | query_time > 2                                         │
│                                                              │
│  4. Voir requêtes lentes exactes pendant spike               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Dashboard Grafana unifié (Metrics + Logs)

```json
{
  "dashboard": {
    "title": "MariaDB Observability - Unified",
    "panels": [
      {
        "id": 1,
        "title": "QPS (Metrics)",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(mysql_global_status_queries[5m])"
          }
        ]
      },
      {
        "id": 2,
        "title": "Slow Queries Count (Metrics)",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(mysql_global_status_slow_queries[5m])"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [10],
                "type": "gt"
              }
            }
          ]
        }
      },
      {
        "id": 3,
        "title": "Slow Query Log (Logs)",
        "type": "logs",
        "datasource": "Loki",
        "targets": [
          {
            "expr": "{job=\"mariadb\", log_type=\"slow_query\"} | json | query_time > 2"
          }
        ],
        "options": {
          "showTime": true,
          "wrapLogMessage": true
        }
      },
      {
        "id": 4,
        "title": "Error Logs (Logs)",
        "type": "logs",
        "datasource": "Loki",
        "targets": [
          {
            "expr": "{job=\"mariadb\", level=\"ERROR\"}"
          }
        ]
      },
      {
        "id": 5,
        "title": "Query Time Histogram (Logs-derived metric)",
        "type": "heatmap",
        "datasource": "Loki",
        "targets": [
          {
            "expr": "sum by (le) (rate({job=\"mariadb\", log_type=\"slow_query\"} | json | unwrap query_time | __error__=\"\" [5m]))"
          }
        ]
      }
    ],
    "templating": {
      "list": [
        {
          "name": "namespace",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(mysql_up, namespace)"
        },
        {
          "name": "instance",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(mysql_up{namespace=\"$namespace\"}, pod)"
        }
      ]
    }
  }
}
```

---

## Pilier 3 : Traces distribuées

### Concept de distributed tracing

```
┌─────────────────────────────────────────────────────────────────┐
│              Distributed Trace Example                          │
│                                                                 │
│  User Request: GET /api/orders/123                              │
│                                                                 │
│  Trace ID: abc-def-ghi-123                                      │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  Span 1: API Gateway                                            │
│  ├─ Start: T+0ms                                                │
│  ├─ End: T+1200ms                                               │
│  └─ Tags: service=api-gateway, http.method=GET                  │
│                                                                 │
│     Span 2: Authentication Service                              │
│     ├─ Start: T+10ms                                            │
│     ├─ End: T+150ms                                             │
│     ├─ Parent: Span 1                                           │
│     └─ Tags: service=auth, user_id=456                          │
│                                                                 │
│     Span 3: Application Logic                                   │
│     ├─ Start: T+160ms                                           │
│     ├─ End: T+1180ms                                            │
│     ├─ Parent: Span 1                                           │
│     └─ Tags: service=app, endpoint=/orders                      │
│                                                                 │
│        Span 4: MariaDB Query 🗄️                                 
│        ├─ Start: T+200ms                                        │
│        ├─ End: T+1150ms                                         │
│        ├─ Parent: Span 3                                        │
│        ├─ Tags:                                                 │
│        │   - db.system=mariadb                                  │
│        │   - db.name=myapp                                      │
│        │   - db.statement=SELECT * FROM orders WHERE id=?       │
│        │   - db.user=appuser                                    │
│        │   - net.peer.name=mariadb-0.mariadb.svc                │
│        │   - net.peer.port=3306                                 │
│        └─ Duration: 950ms ⚠️ SLOW                               │
│                                                                 │
│  Total Duration: 1200ms                                         │
│  Slowest Span: MariaDB Query (950ms = 79% of total)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Insight** : MariaDB query est le goulot d'étranglement (79% du temps total)

### OpenTelemetry pour MariaDB

**OpenTelemetry** = Standard pour instrumentation (metrics + logs + traces)

```
┌──────────────────────────────────────────────────────────────┐
│           OpenTelemetry Architecture                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Application (Python, Java, Go, Node.js...)                  │
│      ↓                                                       │
│  OpenTelemetry SDK                                           │
│      ├─> Auto-instrumentation (databases, HTTP, etc.)        │
│      └─> Manual instrumentation (custom spans)               │
│      ↓                                                       │
│  OpenTelemetry Collector                                     │
│      ├─> Receive traces                                      │
│      ├─> Process (sampling, filtering)                       │
│      └─> Export to backend                                   │
│      ↓                                                       │
│  Backend (Jaeger, Zipkin, Tempo)                             │
│      └─> Store and query traces                              │
│      ↓                                                       │
│  Grafana                                                     │
│      └─> Visualize traces + correlate with metrics/logs      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Exemple : Application Python avec tracing MariaDB

```python
# app.py
from flask import Flask
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.pymysql import PyMySQLInstrumentor
import pymysql

# Setup OpenTelemetry
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Export to OpenTelemetry Collector
otlp_exporter = OTLPSpanExporter(
    endpoint="http://otel-collector.observability.svc.cluster.local:4317",
    insecure=True
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)

# Auto-instrument Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

# Auto-instrument PyMySQL (MariaDB)
PyMySQLInstrumentor().instrument()

# Database connection
def get_db():
    return pymysql.connect(
        host='mariadb.databases.svc.cluster.local',
        port=3306,
        user='appuser',
        password='password',
        database='myapp'
    )

@app.route('/orders/<int:order_id>')
def get_order(order_id):
    # Auto-instrumented: Creates span for this request
    
    with tracer.start_as_current_span("fetch_order_from_db") as span:
        # Custom span attributes
        span.set_attribute("order.id", order_id)
        
        db = get_db()
        cursor = db.cursor()
        
        # Auto-instrumented: Creates span for MariaDB query
        cursor.execute(
            "SELECT * FROM orders WHERE id = %s",
            (order_id,)
        )
        result = cursor.fetchone()
        
        cursor.close()
        db.close()
        
        if result:
            span.set_attribute("order.found", True)
            return {"order": result}
        else:
            span.set_attribute("order.found", False)
            return {"error": "Order not found"}, 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**Ce code génère automatiquement** :
- Span pour requête HTTP Flask
- Span pour query MariaDB (avec statement SQL, duration, host, etc.)
- Propagation trace ID entre services

### Déployer Jaeger sur Kubernetes

```bash
# Via Jaeger Operator
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.51.0/jaeger-operator.yaml -n observability

# Créer instance Jaeger
kubectl apply -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch.observability.svc:9200
  ingress:
    enabled: true
  query:
    replicas: 2
  collector:
    replicas: 3
    resources:
      limits:
        cpu: 1
        memory: 1Gi
EOF
```

### OpenTelemetry Collector

```yaml
# otel-collector-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      
      # Sampling: Garder seulement 10% des traces normales, 100% des erreurs
      probabilistic_sampler:
        sampling_percentage: 10
      
      # Ajouter attributs
      resource:
        attributes:
        - key: cluster
          value: production
          action: insert
    
    exporters:
      # Export vers Jaeger
      jaeger:
        endpoint: jaeger-collector.observability.svc.cluster.local:14250
        tls:
          insecure: true
      
      # Export vers Tempo (alternative Grafana)
      otlp:
        endpoint: tempo.observability.svc.cluster.local:4317
        tls:
          insecure: true
      
      # Logging (debug)
      logging:
        loglevel: info
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, probabilistic_sampler, resource]
          exporters: [jaeger, otlp, logging]
```

---

## Corrélation complète : Logs + Metrics + Traces

### Scenario de debugging complet

**Problème** : Utilisateurs reportent lenteurs sur page checkout à 14h30

**Investigation avec observabilité complète** :

```
┌─────────────────────────────────────────────────────────────────┐
│                  Debugging Workflow                             │
│                                                                 │
│  1️⃣  METRICS (Grafana dashboard)                                │
│     ────────────────────────────────────────────────────────    │
│     Graph: QPS spike à 14h30                                    │
│     Graph: Slow queries spike (10 → 150/min)                    │
│     Graph: Connection usage spike (50% → 95%)                   │
│                                                                 │
│     ✅ Confirmation: Problème à 14h30, lié slow queries         │
│                                                                 │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  2️⃣  LOGS (Loki via Grafana Explore)                            │
│     ────────────────────────────────────────────────────────    │
│     Query LogQL:                                                │
│     {job="mariadb", log_type="slow_query"}                      │
│     | json                                                      │
│     | query_time > 5                                            │
│     | line_format "{{.query}}"                                  │
│                                                                 │
│     Résultat: Query récurrente:                                 │
│     SELECT p.*, i.stock                                         │
│     FROM products p                                             │
│     JOIN inventory i ON p.id = i.product_id                     │
│     WHERE p.category = 'electronics'                            │
│     ORDER BY p.created_at DESC                                  │
│                                                                 │
│     ✅ Query lente identifiée, mais pourquoi soudain lente?     │
│                                                                 │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  3️⃣  TRACES (Jaeger via Grafana)                                │
│     ────────────────────────────────────────────────────────    │
│     Search traces:                                              │
│     - Service: checkout-api                                     │
│     - Time: 14h30-14h35                                         │
│     - Min duration: >5s                                         │
│                                                                 │
│     Résultat: Trace ID abc-123                                  │
│     ├─ API Gateway: 50ms                                        │
│     ├─ Auth: 120ms                                              │
│     ├─ Checkout Service: 8500ms                                 │
│     │  └─ MariaDB query: 8200ms ⚠️                              │
│     │     Statement: [Query ci-dessus]                          │
│     │     Tags:                                                 │
│     │       - db.rows_examined: 5,234,567 ⚠️⚠️⚠️                
│     │       - db.rows_sent: 250                                 │
│                                                                 │
│     ✅ Root cause: Full table scan (5M rows examinées!)         │
│                                                                 │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  4️⃣  SOLUTION                                                   │
│     ────────────────────────────────────────────────────────    │
│     EXPLAIN query:                                              │
│     -> Pas d'index sur products.category                        │
│                                                                 │
│     Fix:                                                        │
│     CREATE INDEX idx_products_category                          │
│       ON products(category, created_at);                        │
│                                                                 │
│     Déployer via migration (Flyway)                             │
│                                                                 │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  5️⃣  VERIFICATION (post-deployment)                             │
│     ────────────────────────────────────────────────────────    │
│     Metrics: Slow queries retombent à 5/min                     │
│     Logs: Query time < 100ms                                    │
│     Traces: MariaDB span < 150ms                                │
│                                                                 │
│     ✅ Problème résolu!                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Temps de résolution** : 15 minutes (vs plusieurs heures sans observabilité)

### Grafana unified dashboard

**Panels corrélés** :

```json
{
  "panels": [
    {
      "id": 1,
      "title": "Timeline - Metrics + Logs + Traces",
      "type": "timeseries",
      "datasource": "-- Mixed --",
      "targets": [
        {
          "datasource": "Prometheus",
          "expr": "rate(mysql_global_status_slow_queries[1m])",
          "legendFormat": "Slow queries/min (metrics)"
        },
        {
          "datasource": "Loki",
          "expr": "sum(count_over_time({job=\"mariadb\", log_type=\"slow_query\"}[1m]))",
          "legendFormat": "Slow query logs (logs)"
        },
        {
          "datasource": "Tempo",
          "query": {
            "queryType": "metrics",
            "serviceName": "mariadb",
            "spanName": "SELECT",
            "metric": "duration"
          },
          "legendFormat": "Query duration p95 (traces)"
        }
      ]
    },
    {
      "id": 2,
      "title": "Trace → Logs Correlation",
      "type": "trace-to-logs",
      "datasource": "Tempo",
      "options": {
        "datasourceUid": "loki-uid",
        "tags": [
          {
            "key": "trace_id",
            "value": "traceID"
          }
        ],
        "query": "{job=\"mariadb\"} |= \"$${__trace.traceId}\""
      }
    }
  ]
}
```

**Workflow utilisateur dans Grafana** :

1. Voir spike dans metrics
2. Cliquer sur point dans graph
3. "View related logs" → Ouvre Loki avec même timerange
4. Cliquer sur log line
5. "View related trace" → Ouvre Jaeger avec trace ID
6. Voir détails complets requête DB

---

## Best practices observabilité

### 1. Structured logging

**❌ Mauvais** (logs non structurés) :

```
2025-12-14 10:30:15 User admin connected from 10.0.1.5
```

**✅ Bon** (logs structurés JSON) :

```json
{
  "timestamp": "2025-12-14T10:30:15.123Z",
  "level": "INFO",
  "event": "user_connected",
  "user": "admin",
  "ip": "10.0.1.5",
  "session_id": "abc-123"
}
```

**Avantages** :
- Facile à parser (Loki, ELK)
- Query précises (field-based)
- Corrélation via IDs

### 2. Correlation IDs

**Propager ID unique à travers système** :

```
Request ID: req-abc-123

API Gateway
  └─> Set request_id: req-abc-123
      └─> App logs: {"request_id": "req-abc-123", "message": "Processing order"}
          └─> MariaDB audit log: {"request_id": "req-abc-123", "query": "INSERT..."}
              └─> Trace: trace_id = req-abc-123
```

**Permet** :
```logql
# Voir TOUS les logs d'une requête spécifique
{job=~".*"} |= "req-abc-123"
```

### 3. Sampling intelligent

**Problème** : Tracer 100% requêtes = overhead énorme

**Solution** : Sampling adaptatif

```yaml
# OpenTelemetry Collector
processors:
  probabilistic_sampler:
    # 10% des requêtes normales
    sampling_percentage: 10
  
  tail_sampling:
    policies:
    # 100% des requêtes avec erreurs
    - name: errors
      type: status_code
      status_code:
        status_codes: [ERROR]
    
    # 100% des requêtes lentes (>1s)
    - name: slow
      type: latency
      latency:
        threshold_ms: 1000
    
    # 50% des requêtes de l'endpoint checkout
    - name: checkout
      type: string_attribute
      string_attribute:
        key: http.url
        values: ["/checkout"]
        enabled_regex_matching: true
      sampling_percentage: 50
```

### 4. Rétention différenciée

```yaml
# Loki
limits_config:
  retention_period: 30d  # Par défaut
  
  # Rétention personnalisée par tenant
  per_tenant_override_config: /etc/loki/overrides.yaml

# overrides.yaml
overrides:
  "production":
    retention_period: 90d  # Compliance
  "dev":
    retention_period: 7d   # Économies
```

### 5. Alerting sur logs

**PrometheusRule basée sur logs** :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mariadb-log-alerts
spec:
  groups:
  - name: mariadb-logs
    interval: 1m
    rules:
    # Alert sur rate d'erreurs dans logs
    - alert: HighErrorRate
      expr: |
        sum(rate({job="mariadb", level="ERROR"}[5m])) > 10
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate in MariaDB logs"
    
    # Alert sur pattern spécifique
    - alert: TableDoesNotExist
      expr: |
        sum(count_over_time({job="mariadb"} |= "doesn't exist" [5m])) > 0
      labels:
        severity: critical
      annotations:
        summary: "Table doesn't exist errors detected"
```

---

## ✅ Points clés à retenir

- **3 piliers complémentaires** : Metrics (what), Logs (why), Traces (how)
- **Loki vs ELK** : Loki pour Kubernetes/Grafana, ELK pour full-text search
- **Logs MariaDB** : Error, Slow query, Audit (compliance), Binary (replication)
- **Promtail** : Agent pour scraper logs et push vers Loki
- **OpenTelemetry** : Standard pour instrumentation (auto + manual)
- **Jaeger/Tempo** : Backend pour distributed tracing
- **Corrélation** : Request ID partout, trace ID → logs, metrics → logs
- **Sampling** : 100% erreurs/slow, 10% normal (réduire overhead)
- **Structured logs** : JSON pour parsing facile
- **Unified dashboard** : Metrics + Logs + Traces dans même vue

💡 **Golden rule** : L'observabilité n'est pas un but, c'est un moyen. But = Réduire MTTR (Mean Time To Repair).

---

## 🔗 Ressources et références

### Documentation officielle
- [📖 Grafana Loki](https://grafana.com/docs/loki/latest/)
- [📖 OpenTelemetry](https://opentelemetry.io/docs/)
- [📖 Jaeger](https://www.jaegertracing.io/docs/)
- [📖 Tempo](https://grafana.com/docs/tempo/latest/)

### Guides
- [📝 Observability Engineering (O'Reilly)](https://www.oreilly.com/library/view/observability-engineering/9781492076438/)
- [📝 Distributed Tracing in Practice](https://www.oreilly.com/library/view/distributed-tracing-in/9781492056621/)

### Outils
- [🔧 LogQL Cheat Sheet](https://grafana.com/docs/loki/latest/logql/)
- [🔧 OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)

---

## ➡️ Prochaines sections

**16.11 Alerting et Incident Response** : Stratégies d'alerting, on-call, post-mortems, SLOs/SLIs.

**16.12 GitOps pour bases de données** : ArgoCD, FluxCD, declarative database management.

---

**MariaDB** : Version 11.8 LTS
**Loki** : v2.9+
**OpenTelemetry** : v1.21+
**Jaeger** : v1.51+

⏭️ [Alerting et incident response](/16-devops-automatisation/11-alerting-incident-response.md)

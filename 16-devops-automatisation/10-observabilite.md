🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.10 — Observabilité : Logs, Metrics, Traces

> **Positionnement.** La section [16.9](09-monitoring-prometheus-grafana.md) a posé le pilier *Metrics* (Prometheus, Grafana, `mysqld_exporter`). Cette section prend de la hauteur : elle replace les métriques dans le cadre plus large de l'**observabilité**, dont elles ne sont qu'un tiers, et montre comment les trois piliers — **Logs, Metrics, Traces** — se combinent pour superviser MariaDB dans une architecture moderne et distribuée.

## Monitoring n'est pas observabilité

Ces deux termes sont souvent confondus, mais ils ne répondent pas à la même question.

Le **monitoring** consiste à surveiller des indicateurs *connus à l'avance* pour détecter des modes de défaillance *anticipés* : « le serveur répond-il ? », « la réplication est-elle en retard ? ». On construit pour cela des dashboards et des alertes figés. Le monitoring excelle face aux problèmes que l'on a déjà su nommer.

L'**observabilité** est une propriété plus ambitieuse : c'est la capacité à comprendre l'état interne d'un système *uniquement à partir de ses sorties externes*, y compris pour des questions que l'on n'avait pas prévues — sans avoir à déployer du nouveau code pour investiguer. Elle vise les *unknown unknowns* : « pourquoi cette requête précise, déclenchée par ce service précis, est-elle lente uniquement le mardi à 14 h ? ». Le monitoring dit *qu'*un problème existe ; l'observabilité permet de comprendre *pourquoi*.

Pour atteindre cette propriété, on s'appuie sur trois sources de données complémentaires, traditionnellement appelées les **trois piliers de l'observabilité**.

## Les trois piliers en un coup d'œil

| Pilier | Question à laquelle il répond | Nature de la donnée | Source côté MariaDB |
|--------|-------------------------------|---------------------|----------------------|
| **Metrics** | *Quoi* et *quand* ? (santé agrégée) | Séries temporelles numériques | Variables de statut → `mysqld_exporter` |
| **Logs** | *Pourquoi* ? (événements unitaires) | Enregistrements horodatés, textuels | Error log, slow query log, audit log… |
| **Traces** | *Où* dans la requête distribuée ? | Arbres de spans corrélés par requête | Instrumentation **applicative** (OpenTelemetry) |

Aucun pilier ne se suffit à lui-même. Les métriques signalent une anomalie mais n'en donnent pas la cause ; les logs détaillent un événement mais ne le situent pas dans une requête de bout en bout ; les traces montrent le chemin d'une requête à travers les services mais sans le détail interne de chaque base. La valeur réelle de l'observabilité naît de leur **corrélation**, traitée plus loin.

## Pilier 1 — Metrics

Les métriques sont des mesures numériques échantillonnées dans le temps : nombre de connexions, débit de requêtes, taux de hits du Buffer Pool, lag de réplication. Elles sont **peu coûteuses à stocker** (une valeur par point de temps), agrégeables, et idéales pour les tendances, les seuils et les alertes.

Ce pilier a été traité en détail en [16.9](09-monitoring-prometheus-grafana.md) : `mysqld_exporter` traduit les variables de statut de MariaDB en métriques, Prometheus les scrape selon un modèle *pull*, et Grafana les visualise. On retiendra ici simplement leur **rôle dans l'observabilité** : les métriques constituent le point d'entrée du diagnostic. C'est une métrique (un pic de latence, une chute du hit ratio) qui déclenche l'investigation — laquelle se poursuit ensuite dans les logs et les traces.

Leur limite est intrinsèque : une métrique est **agrégée**. `slow_queries` augmente, mais la métrique seule ne dit pas *quelle* requête est lente, *qui* l'a lancée, ni *pourquoi*. Pour cela, il faut descendre d'un cran.

## Pilier 2 — Logs

Les logs sont des enregistrements d'**événements unitaires et datés**. Là où une métrique dit « il y a eu 12 requêtes lentes », un log dit « *cette* requête, à *cet* instant, a duré 4,2 s avec *ce* plan d'exécution ». MariaDB est riche en journaux, chacun avec une finalité distincte.

### Les journaux de MariaDB pertinents pour l'observabilité

| Journal | Variable de contrôle | Contenu | Usage en observabilité |
|---------|----------------------|---------|------------------------|
| **Error log** | `log_error`, `log_warnings` | Démarrages/arrêts, erreurs, warnings, crashs | Diagnostic d'incidents, santé du serveur |
| **Slow query log** | `slow_query_log`, `long_query_time` | Requêtes dépassant un seuil de durée | Analyse de performance (cœur du sujet) |
| **General query log** | `general_log` | *Toutes* les requêtes reçues | Débogage ponctuel — **jamais en prod** (volume) |
| **Audit log** | `server_audit_*` (plugin) | Connexions, requêtes, accès — traçabilité | Sécurité, conformité, SIEM (voir [§10.8](../10-securite-gestion-utilisateurs/08-audit-logging.md)) |
| **Binary log** | `log_bin` | Modifications de données (réplication) | Source de CDC plus qu'observabilité (voir [§20.8](../20-cas-usage-architectures/08-architectures-event-driven.md)) |

Pour l'observabilité de performance, le **slow query log** est le journal central (détaillé en [§11.4.2](../11-administration-configuration/04.2-slow-query-log.md)). MariaDB permet d'en enrichir considérablement le contenu via `log_slow_verbosity`, qui peut y joindre le plan d'exécution et des statistiques InnoDB :

```ini
[mariadb]
slow_query_log        = ON  
slow_query_log_file   = /var/log/mysql/mariadb-slow.log  
long_query_time       = 0.5          # seuil en secondes  
log_slow_verbosity    = query_plan,innodb,explain  
log_queries_not_using_indexes = ON  
log_output            = FILE         # voir ci-dessous  
```

### FILE ou TABLE : un choix structurant

La variable `log_output` détermine où atterrissent le slow log et le general log : dans un **fichier** (`FILE`), dans une **table système** (`TABLE` → `mysql.slow_log`, `mysql.general_log`), ou les deux.

Le mode `TABLE` rend les logs immédiatement interrogeables en SQL, ce qui est pratique pour une analyse ponctuelle. Mais en contexte d'observabilité centralisée, on privilégie **`FILE`** : un agent de collecte tail le fichier et l'expédie vers un système d'agrégation, sans imposer à MariaDB le surcoût d'écriture dans une table à chaque requête lente.

### Centraliser les logs

Sur un parc de plusieurs instances, consulter les logs serveur par serveur est ingérable. Le principe de l'observabilité moderne est d'**agréger tous les logs dans un système centralisé et indexé**. La chaîne type comporte un **agent de collecte** sur chaque hôte (Promtail ou Grafana Alloy, Fluent Bit, Vector…) qui lit les fichiers de log, les parse, les enrichit d'étiquettes (hôte, environnement, type de journal) et les pousse vers un backend.

```yaml
# Exemple : collecte vers Loki (extrait de configuration d'agent)
scrape_configs:
  - job_name: mariadb-logs
    static_configs:
      - targets: [localhost]
        labels:
          host: db-prod-01
          log: mariadb-error
          __path__: /var/log/mysql/error.log
      - targets: [localhost]
        labels:
          host: db-prod-01
          log: mariadb-slow
          __path__: /var/log/mysql/mariadb-slow.log
```

Deux familles de backends dominent : **Loki** (écosystème Grafana, indexation par étiquettes, recherche LogQL) et la **suite Elastic** (Elasticsearch + Kibana, indexation plein texte). Le choix relève de l'architecture d'entreprise ; les deux remplissent la même fonction : rendre l'ensemble des logs du parc consultables, filtrables et corrélables depuis un point unique.

> 🆕 **Tie-in MariaDB 12.x.** À fort volume, l'écriture de l'audit log peut devenir un goulot. Le plugin d'audit gère désormais une **écriture bufferisée** (`server_audit_file_buffer_size`, voir [§10.8.3](../10-securite-gestion-utilisateurs/08.3-audit-bufferise.md)), ce qui réduit l'impact de la collecte de traçabilité sur les performances — un point appréciable lorsque l'audit log alimente un pipeline d'observabilité ou un SIEM.

## Pilier 3 — Traces

Le *tracing distribué* suit une requête utilisateur **de bout en bout** à travers tous les services qu'elle traverse, sous forme d'un arbre de **spans** (un span = une unité de travail, avec durée, parent et attributs). Dans une architecture microservices, une seule requête HTTP peut solliciter une passerelle, plusieurs services et plusieurs bases. La trace répond à la question : *où* part le temps, et *quelle* dépendance est responsable de la latence ?

### Le défi pour une base de données

C'est ici qu'il faut être lucide : **MariaDB n'émet pas nativement de traces OpenTelemetry**. Il n'existe pas, dans le serveur, de mécanisme produisant des spans pour chaque requête. Le tracing d'un appel base de données se fait donc **au niveau du connecteur ou de l'application**.

Concrètement, les bibliothèques d'instrumentation OpenTelemetry des connecteurs (JDBC pour Java, les connecteurs Python/SQLAlchemy, `mysql2` pour Node.js, etc. — voir [§17.1](../17-integration-developpement/01-connexion-langages.md)) entourent chaque exécution de requête d'un **span enfant**. Ce span porte, selon les conventions sémantiques OpenTelemetry, des attributs comme `db.system`, `db.statement` ou `db.operation`. Du point de vue de la trace, l'appel à MariaDB apparaît comme un span dont on connaît la durée et le contexte applicatif — mais c'est l'application qui le crée, pas la base.

### Corréler une trace à une requête : le contexte dans le commentaire SQL

Reste un fossé à combler : la trace sait *qu'*un appel à la base a duré 4 s, mais comment relier ce span au slow query log de MariaDB, qui, lui, connaît le plan d'exécution réel ? La technique la plus largement adoptée est l'**injection du contexte de trace dans un commentaire SQL** (approche *SQLCommenter*, intégrée à l'écosystème OpenTelemetry).

L'application ajoute le `traceparent` (identifiant de trace W3C) en commentaire de la requête :

```sql
SELECT * FROM commandes WHERE client_id = 42
/*traceparent='00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01'*/;
```

Le commentaire est ignoré par l'exécution SQL, mais il **apparaît tel quel dans le slow query log et dans PERFORMANCE_SCHEMA**. On peut alors joindre, par l'identifiant de trace, le span applicatif (côté APM) à la requête exacte enregistrée côté base — fermant la boucle entre les trois piliers.

### PERFORMANCE_SCHEMA : l'introspection interne

Au-delà des logs, MariaDB offre une source d'introspection puissante via **PERFORMANCE_SCHEMA** et le schéma `sys` (voir [§15.8](../15-performance-tuning/08-performance-schema-sys.md) et [§9.7.2](../09-vues-et-donnees-virtuelles/07.2-performance-schema.md)). La table `events_statements_summary_by_digest` agrège les requêtes par *digest* (forme normalisée), ce qui identifie les requêtes les plus coûteuses sans dépendre du slow log :

```sql
SELECT digest_text,
       count_star,
       ROUND(sum_timer_wait / 1e12, 2) AS total_s,
       ROUND(avg_timer_wait / 1e9, 2)  AS avg_ms
FROM performance_schema.events_statements_summary_by_digest  
ORDER BY sum_timer_wait DESC  
LIMIT 10;  
```

Ce n'est pas du tracing au sens distribué, mais cela fournit une vue d'introspection fine que les pipelines d'observabilité peuvent collecter périodiquement (certains exporters dédiés exposent ces données comme métriques).

## Corréler les trois piliers : là où réside la valeur

L'objectif final n'est pas trois silos d'outils, mais une **boucle de diagnostic fluide**. Un parcours d'incident type illustre la complémentarité :

1. **Metrics** — une alerte se déclenche : la latence p99 d'un service grimpe, et en parallèle le taux de requêtes lentes de MariaDB augmente. *On sait qu'il y a un problème, et quand.*
2. **Traces** — on ouvre une trace lente de la période concernée : la majorité du temps est consommée dans un span « base de données », pointant vers une requête précise. *On sait où.*
3. **Logs** — via le `traceparent` injecté, on retrouve cette requête exacte dans le slow query log, avec son plan d'exécution : un *full scan* sur une table récemment grossie. *On sait pourquoi.*

Le ciment de cette corrélation, ce sont des **identifiants communs** : un identifiant de trace partagé, des étiquettes cohérentes (hôte, service, environnement) et une **synchronisation temporelle** (NTP) rigoureuse entre tous les composants. Sans horloges alignées, corréler un pic de métrique à un log devient un exercice approximatif.

## La stack d'observabilité moderne

Deux approches structurent le marché, articulées autour d'un standard pivot, **OpenTelemetry (OTel)**. OTel fournit un format et un protocole *neutres* (les conventions sémantiques, le protocole OTLP) ainsi qu'un **Collector** qui reçoit, transforme et route les trois types de signaux vers les backends de son choix — ce qui évite l'enfermement propriétaire.

| Pilier | Stack Grafana (« LGTM ») | Suite Elastic |
|--------|--------------------------|----------------|
| Metrics | Prometheus / Mimir | Elasticsearch (+ Metricbeat) |
| Logs | **L**oki | **E**lasticsearch (+ Filebeat) |
| Traces | **T**empo | Elastic APM |
| Visualisation | **G**rafana | **K**ibana |

La stack **Grafana** (Loki pour les logs, Tempo pour les traces, Mimir/Prometheus pour les métriques, le tout unifié dans **Grafana**) a l'avantage de fédérer les trois piliers dans une seule interface, avec navigation directe d'un dashboard de métrique vers les logs puis les traces correspondants. La **suite Elastic** offre une indexation plein texte particulièrement puissante pour la recherche dans les logs. Dans les deux cas, **OpenTelemetry** s'intercale idéalement comme couche de collecte standardisée en amont.

## Bonnes pratiques d'observabilité pour MariaDB

- **Choisir ses journaux avec discernement.** Slow query log et error log centralisés : oui. General log en production : non (volume ingérable, surcoût). Audit log : selon les besoins de conformité, idéalement en mode bufferisé.
- **Logguer en `FILE`, pas en `TABLE`**, dès que la collecte est centralisée, pour ne pas faire payer à MariaDB le coût d'écriture en table.
- **Enrichir le slow log** via `log_slow_verbosity` (plan d'exécution, InnoDB) : un slow log sans plan d'exécution oblige à rejouer la requête pour comprendre.
- **Propager le contexte de trace dans les commentaires SQL** côté application : c'est le maillon qui transforme trois silos en une chaîne de diagnostic.
- **Synchroniser les horloges (NTP)** sur tous les composants : sans cela, la corrélation temporelle entre piliers est illusoire.
- **Gérer le rapport signal/bruit.** Plus de données n'égale pas plus d'observabilité. Un `long_query_time` trop bas noie les vraies anomalies ; des étiquettes incohérentes rendent la corrélation impossible. Définir la rétention et le niveau de détail en fonction du coût et de l'usage réel.

## Points clés à retenir

L'observabilité dépasse le monitoring : elle vise la compréhension du *pourquoi*, y compris pour des questions non anticipées, à partir de trois piliers complémentaires. Les **métriques** ([16.9](09-monitoring-prometheus-grafana.md)) signalent et alertent mais restent agrégées ; les **logs** de MariaDB (slow query log en tête, à centraliser via Loki ou Elastic) détaillent les événements unitaires ; les **traces**, qui ne sont *pas* produites par MariaDB mais par l'instrumentation applicative OpenTelemetry, situent l'appel base de données dans la requête distribuée. La valeur émerge de leur **corrélation** — rendue possible par l'injection du contexte de trace dans les commentaires SQL, des étiquettes cohérentes et une horloge commune. En pratique, on s'appuie sur OpenTelemetry comme standard de collecte et sur une stack unifiée (Grafana LGTM ou suite Elastic), en gardant à l'esprit que la pertinence prime sur le volume.

---

↩️ [Section précédente : 16.9 — Monitoring avec Prometheus/Grafana](09-monitoring-prometheus-grafana.md)  
➡️ **Section suivante :** [16.11 — Alerting et incident response](11-alerting-incident-response.md)

⏭️ [Alerting et incident response](/16-devops-automatisation/11-alerting-incident-response.md)

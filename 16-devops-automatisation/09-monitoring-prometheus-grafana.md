🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.9 — Monitoring avec Prometheus/Grafana

> Chapitre 16 — DevOps et Automatisation · Vue d'ensemble de la section, avant l'étude de l'exportateur et des tableaux de bord.

---

## Introduction

Après la gestion des migrations, le chapitre ouvre son **volet observabilité**, dont cette section est la première étape : la **supervision** (*monitoring*) de MariaDB au moyen de la pile **Prometheus/Grafana**. Cette page d'introduction pose les concepts et l'architecture de cette pile, avant que les sous-sections n'en détaillent les composants — l'**exportateur** mysqld_exporter (§16.9.1) et les **tableaux de bord** Grafana (§16.9.2).

## Pourquoi superviser une base de données

Une base de données de production exige une **visibilité continue** : est-elle saine, performante, disponible ? La supervision répond à plusieurs besoins convergents. Elle permet la **détection proactive** — repérer un problème avant que les utilisateurs ne le subissent —, le **dimensionnement** (planification de capacité), l'**optimisation des performances** (chapitre 15), la **réponse aux incidents** (§16.11) et le suivi des **objectifs de niveau de service**. Pour un DBA ou un ingénieur DevOps, le principe est simple : *on ne pilote bien que ce que l'on mesure*.

Pour MariaDB, les grandeurs à surveiller sont nombreuses : connexions, débit et latence des requêtes, *buffer pool*, **retard de réplication** (chapitre 13), verrous, espace disque, fils d'exécution. Elles rejoignent les métriques importantes évoquées en §11.9 et l'analyse de performance du chapitre 15.

## La pile Prometheus/Grafana

La pile **Prometheus/Grafana** est devenue le standard de fait de la supervision *cloud native*. Elle s'articule autour de quelques composants.

**Prometheus** est à la fois une **base de données de séries temporelles** et un système de supervision : il **collecte** (par *scraping*) des métriques auprès de cibles à intervalles réguliers, les stocke sous forme de séries temporelles, et les interroge au moyen de son langage de requête, **PromQL**.

**Grafana** est l'outil de **visualisation** : il interroge Prometheus (et d'autres sources) pour afficher des **tableaux de bord**, graphiques et indicateurs.

Les **exportateurs** (*exporters*) font le pont : Prometheus ne comprend pas MariaDB directement, et un exportateur — **mysqld_exporter** (§16.9.1) — **traduit** les métriques internes de MariaDB dans le format que Prometheus sait collecter.

Enfin, l'**Alertmanager**, compagnon de Prometheus, **achemine les alertes** déclenchées par des règles — sujet du volet alerting (§16.11).

## Comment cela fonctionne : le pipeline

L'enchaînement de ces composants forme un pipeline clair. MariaDB **expose ses métriques internes** — via `SHOW STATUS`, `INFORMATION_SCHEMA` et `PERFORMANCE_SCHEMA` (§9.7, §15.8). L'**exportateur** s'y connecte, lit ces métriques et les **expose sur un point d'accès HTTP** au format Prometheus. **Prometheus collecte** ce point d'accès à intervalle régulier et **stocke** les séries temporelles. **Grafana interroge** Prometheus en PromQL et **affiche** les tableaux de bord. L'**Alertmanager**, alimenté par les règles d'alerte de Prometheus, **route** les notifications.

Le flux se résume ainsi : **MariaDB → exportateur → Prometheus → Grafana**, l'Alertmanager se greffant sur Prometheus pour les alertes. Chaque composant a un rôle distinct et bien délimité.

## Le modèle pull

Une caractéristique de Prometheus mérite d'être soulignée : il **collecte** les métriques en allant les **chercher** (*pull*) auprès des cibles, plutôt que de les recevoir (*push*). Ce modèle présente des avantages : Prometheus **maîtrise la fréquence** de collecte, l'**indisponibilité d'une cible** devient en elle-même un signal (une cible injoignable est détectée), et l'intégration avec la **découverte de services** facilite le suivi de cibles dynamiques. Pour les tâches éphémères qui ne vivent pas assez longtemps pour être collectées, une passerelle de poussée (*Pushgateway*) existe, mais le *pull* reste le modèle par défaut.

## Les types de métriques

Les métriques se rangent en quelques types. Les **compteurs** (*counters*) ne font que croître (par exemple le nombre total de requêtes), les **jauges** (*gauges*) montent et descendent (par exemple le nombre de connexions actives), et les **histogrammes** décrivent des distributions (par exemple la latence des requêtes). PromQL opère sur ces types — la fonction `rate()`, par exemple, calcule un taux à partir d'un compteur. Une compréhension de ces types est utile pour interpréter les tableaux de bord.

## Le patron de l'exportateur

Prometheus est **indépendant de la technologie supervisée** ; ce sont les exportateurs qui relient un système spécifique à Prometheus. Pour MariaDB, l'exportateur de référence est **mysqld_exporter**, qui fonctionne avec MariaDB : il s'exécute aux côtés de la base, s'y connecte avec un compte de supervision et expose les métriques sur un point d'accès. Son fonctionnement et sa configuration font l'objet de la sous-section §16.9.1.

## Tableaux de bord et visualisation

Une fois les métriques collectées, **Grafana** les transforme en **tableaux de bord** — graphiques, jauges, tableaux, cartes de chaleur. On peut s'appuyer sur des **tableaux de bord communautaires** prêts à l'emploi pour MariaDB/MySQL, ou en **concevoir** sur mesure. Ce sujet est détaillé en §16.9.2.

## Sur Kubernetes : l'intégration prometheus-operator

Sur Kubernetes, la pile se gère elle-même de façon déclarative. Le **prometheus-operator** administre Prometheus et l'Alertmanager via des ressources personnalisées (notamment `ServiceMonitor` pour déclarer les cibles à collecter, et `PrometheusRule` pour les règles). Surtout, le **mariadb-operator** (§16.5) et son pendant Enterprise (§16.6) **intègrent nativement** cette supervision : ils exécutent des **exportateurs en *sidecar*** (mysqld-exporter, maxscale-exporter) et **créent automatiquement les ressources `ServiceMonitor`** — de sorte que la supervision est câblée pour l'utilisateur, sans configuration manuelle. C'est là un bénéfice concret de l'approche par opérateur présentée plus haut.

## Place dans le volet observabilité

Cette section s'inscrit dans un ensemble de trois. La présente section (§16.9) traite de la **supervision par métriques**, fondée sur Prometheus/Grafana. La suivante (§16.10) **élargit à l'observabilité** au sens large — les trois piliers que sont les journaux, les métriques et les traces. La dernière (§16.11) porte sur l'**alerting et la réponse aux incidents**, c'est-à-dire l'**action** sur les signaux collectés. Les métriques de cette section en sont le socle.

## Spécificités MariaDB 12.3 LTS

Les métriques supervisées proviennent de l'état et des schémas système de MariaDB (`SHOW STATUS`, `INFORMATION_SCHEMA`, `PERFORMANCE_SCHEMA`), pour l'essentiel **indépendants de la version** ; l'exportateur fonctionne avec la 12.3 comme avec les versions antérieures. Les évolutions de la 12.3 — par exemple autour du binlog (§11.5.4) — peuvent s'observer au travers de ces mêmes métriques, sans changer l'architecture de supervision.

## Ce que couvrent les sous-sections

À partir de cette vue d'ensemble, les sous-sections entrent dans le concret. La première (§16.9.1) détaille **mysqld_exporter** : déploiement, configuration et métriques exposées. La seconde (§16.9.2) présente les **tableaux de bord Grafana** pour MariaDB.

---

↩️ [Section précédente : 16.8 — Gestion des migrations](08-gestion-migrations.md)  
➡️ **Sous-section suivante :** [16.9.1 — mysqld_exporter](09.1-mysqld-exporter.md)

⏭️ [mysqld_exporter](/16-devops-automatisation/09.1-mysqld-exporter.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.11 — Alerting et incident response

> **Positionnement.** Les sections [16.9](09-monitoring-prometheus-grafana.md) et [16.10](10-observabilite.md) ont permis de *voir* l'état de MariaDB. Mais une instance n'est pas supervisée en pratique parce qu'on la *regarde* : elle l'est parce que le système *réagit* en notre absence. Cette section traite ce passage à l'action en deux temps : l'**alerting** (comment le système signale automatiquement une anomalie) et l'**incident response** (comment l'humain et l'organisation y répondent).

## De la donnée à l'action

L'observabilité produit des données ; elle ne produit pas, à elle seule, de réaction. Personne ne contemple un dashboard à 3 h du matin. La supervision réelle repose donc sur une chaîne : une condition anormale est **détectée** automatiquement (alerting), une **notification** est routée vers la bonne personne, qui suit une **procédure** pour rétablir le service (incident response), puis l'organisation **apprend** de l'événement (postmortem). Une alerte sans processus de réponse est inutile ; un processus de réponse sans alertes fiables est aveugle.

---

# Partie A — Alerting

## Où définir les alertes : Prometheus/Alertmanager ou Grafana ?

Comme annoncé en [16.10](10-observabilite.md), deux approches coexistent, et le choix structure l'architecture.

**Grafana Alerting** évalue des règles directement à partir des panneaux et des sources de données. C'est l'option la plus simple à mettre en place : alerte et visualisation vivent au même endroit, et l'on peut alerter aussi bien sur des métriques que sur des logs (Loki). C'est un bon choix pour des environnements modestes ou pour démarrer.

**Prometheus + Alertmanager** sépare les responsabilités. Prometheus *évalue* les règles d'alerte (en PromQL, sur la base de ses propres données) et délègue tout le *routage* des notifications à un composant dédié, **Alertmanager**. C'est l'approche privilégiée en production exigeante, pour trois raisons : l'évaluation des alertes est découplée de la couche de visualisation (Grafana peut tomber sans empêcher les alertes de partir) ; Alertmanager peut être déployé en **cluster haute disponibilité** ; et il offre une logique de routage, de regroupement et d'inhibition bien plus riche.

> Ces deux mondes ne s'excluent pas : Grafana peut être configuré pour **router ses alertes via Alertmanager**, ce qui combine la facilité de création de Grafana et la puissance de routage d'Alertmanager.

La suite de cette section décrit l'approche Prometheus/Alertmanager, qui est la référence ; les principes de conception s'appliquent quel que soit l'outil.

## Anatomie d'une règle d'alerte Prometheus

Une règle d'alerte est une expression PromQL associée à une durée de persistance et à des métadonnées. Lorsque l'expression renvoie un résultat pendant toute la durée `for`, l'alerte passe de l'état *pending* à l'état *firing*, et Prometheus la transmet à Alertmanager.

```yaml
# /etc/prometheus/rules/mariadb.yml
groups:
  - name: mariadb.rules
    rules:
      - alert: MariaDBDown
        expr: mysql_up == 0
        for: 1m                         # tolère un scrape manqué isolé
        labels:
          severity: critical
        annotations:
          summary: "Instance MariaDB injoignable ({{ $labels.instance }})"
          description: "mysqld_exporter n'interroge plus l'instance depuis 1 min."
          runbook_url: "https://runbooks.exemple.com/mariadb/down"

      - alert: MariaDBReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Lag de réplication élevé sur {{ $labels.instance }}"
          description: "Retard de plus de 300 s depuis 5 minutes."
```

Trois éléments méritent attention. La clause **`for`** est un filtre anti-bruit essentiel : elle évite de déclencher sur un pic transitoire d'une seconde. Le label **`severity`** servira de critère de routage côté Alertmanager. Et l'annotation **`runbook_url`** relie l'alerte à sa procédure de remédiation — un détail décisif pour l'incident response (voir Partie B).

> 💡 Les expressions coûteuses ou réutilisées (comme le taux de hits du Buffer Pool calculé en [16.9.2](09.2-dashboards-grafana.md)) gagnent à être précalculées via des **recording rules** Prometheus, puis référencées dans les règles d'alerte. On évite ainsi de recalculer la même formule à chaque évaluation.

## Alertmanager : router, regrouper, faire taire

Alertmanager reçoit les alertes émises par Prometheus et décide *quoi en faire*. Ses quatre fonctions essentielles sont :

- **Le routage** : un arbre de routes dirige chaque alerte vers le bon destinataire selon ses labels (par exemple, `severity = critical` vers l'astreinte, le reste vers un canal d'équipe).
- **Le regroupement** (*grouping*) : plutôt que d'envoyer 50 notifications quand 50 instances tombent simultanément, Alertmanager les agrège en une seule notification. Indispensable contre la fatigue d'alertes.
- **La déduplication** : si plusieurs Prometheus en HA émettent la même alerte, l'opérateur n'est notifié qu'une fois.
- **L'inhibition et les silences** : une alerte peut en **inhiber** d'autres (si tout le datacenter est down, inutile d'alerter sur chaque base), et un opérateur peut poser un **silence** temporaire pendant une maintenance planifiée.

```yaml
# alertmanager.yml (extrait)
route:
  receiver: 'dba-slack'
  group_by: ['alertname', 'instance']
  group_wait: 30s          # attend avant la 1re notification (regroupement)
  group_interval: 5m       # délai avant d'ajouter de nouvelles alertes au groupe
  repeat_interval: 4h      # ré-émission si non résolu
  routes:
    - matchers: [ 'severity = critical' ]
      receiver: 'astreinte-pagerduty'

receivers:
  - name: 'dba-slack'
    slack_configs:
      - channel: '#alertes-db'
  - name: 'astreinte-pagerduty'
    pagerduty_configs:
      - routing_key: '<clé>'
```

Les destinataires (*receivers*) couvrent les canaux usuels : e-mail, Slack/Teams, et surtout les outils d'astreinte comme **PagerDuty** ou **Opsgenie**, qui gèrent les rotations et l'escalade. Le récepteur générique **webhook** permet par ailleurs d'intégrer n'importe quel système maison ou d'automatiser une remédiation.

## Bien concevoir ses alertes : alerter sur les symptômes

La qualité d'un système d'alerte ne se mesure pas au nombre de règles, mais à sa **pertinence**. Trop d'alertes tue l'alerte : un opérateur noyé sous les notifications finit par toutes les ignorer — y compris la seule qui comptait. C'est le syndrome de la **fatigue d'alertes**, première cause de dégradation d'une astreinte.

Le principe directeur, hérité de la culture SRE, est d'**alerter sur les symptômes, pas sur les causes**. Un symptôme affecte l'utilisateur ou le service (« les requêtes échouent », « la latence explose ») ; une cause est un mécanisme interne (« le Buffer Pool est plein »). Alerter sur chaque cause potentielle produit du bruit, car une cause n'entraîne pas toujours de conséquence visible. On réserve donc les notifications urgentes (*page*) aux symptômes qui exigent une action humaine immédiate, et l'on relègue les causes en *warnings* non urgents (un ticket) servant au diagnostic une fois le symptôme constaté.

D'où une hiérarchie de sévérité claire :

| Sévérité | Signification | Canal | Exemple MariaDB |
|----------|---------------|-------|-----------------|
| **Critical** (*page*) | Action humaine immédiate requise | Astreinte (réveille) | Instance down, réplication arrêtée |
| **Warning** (*ticket*) | À traiter, mais non urgent | Canal d'équipe | Lag modéré, hit ratio en baisse |
| **Info** | Contexte, pas d'action | Tableau de bord / log | Pic de connexions ponctuel |

La règle d'or : **toute alerte critique doit être actionnable**. Si l'opérateur réveillé ne peut rien faire d'utile, l'alerte ne devrait pas le réveiller.

## SLI, SLO et error budget

Au-delà des seuils statiques, l'approche SRE moderne formalise des objectifs de service. Un **SLI** (*Service Level Indicator*) est une mesure de qualité du service (taux de disponibilité, latence sous un seuil). Un **SLO** (*Service Level Objective*) est la cible visée pour ce SLI (par exemple « 99,9 % de disponibilité sur 30 jours »). La marge entre 100 % et le SLO constitue l'**error budget** : le volume d'erreurs *tolérable* avant de manquer l'objectif.

L'intérêt pour l'alerting est qu'on n'alerte plus sur un seuil brut, mais sur la **vitesse de consommation de ce budget** (*burn rate*). Une consommation lente n'a rien d'urgent ; une consommation rapide qui épuiserait le budget en quelques heures justifie de réveiller quelqu'un. Cette approche **multi-fenêtre / multi-burn-rate** réduit drastiquement le bruit tout en restant réactive aux dégradations graves — elle est plus difficile à mettre en place, mais c'est l'état de l'art pour des services critiques s'appuyant sur MariaDB.

## Alertes essentielles pour MariaDB

Le tableau suivant constitue un socle d'alerting pour une instance MariaDB, en réutilisant les métriques des sections précédentes. Les seuils sont indicatifs et à calibrer selon le contexte.

| Condition | Expression PromQL | Sévérité |
|-----------|-------------------|----------|
| Instance injoignable | `mysql_up == 0` | Critical |
| Réplication SQL arrêtée | `mysql_slave_status_slave_sql_running == 0` | Critical |
| Réplication I/O arrêtée | `mysql_slave_status_slave_io_running == 0` | Critical |
| Lag de réplication élevé | `mysql_slave_status_seconds_behind_master > 300` | Warning |
| Saturation des connexions | `mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.85` | Warning |
| Nœud Galera non prêt | `mysql_global_status_wsrep_ready == 0` | Critical |
| Cluster Galera réduit | `mysql_global_status_wsrep_cluster_size < 3` | Critical |
| Connexions avortées (spike) | `rate(mysql_global_status_aborted_connects[5m]) > 1` | Warning |
| Espace disque faible | `node_filesystem_avail_bytes{mountpoint="/var/lib/mysql"} / node_filesystem_size_bytes < 0.10` | Critical |

> ⚠️ La dernière ligne provient du **`node_exporter`**, pas de `mysqld_exporter` : l'espace disque est une métrique système. C'est pourtant l'une des alertes les plus critiques pour une base de données — un disque plein arrête brutalement les écritures et peut corrompre l'instance. Ne jamais l'omettre.

---

# Partie B — Incident Response

L'alerte n'est que le commencement. Ce qui se passe *après* qu'elle a sonné — la réponse à incident — détermine la durée et la gravité réelle de l'indisponibilité.

## Le cycle de vie d'un incident

Un incident bien géré suit des phases identifiables :

1. **Détection** — l'alerte se déclenche et notifie l'astreinte.
2. **Acquittement** (*acknowledge*) — un opérateur prend en charge, signalant qu'il s'en occupe (et stoppant l'escalade).
3. **Triage** — évaluation de l'impact et de la sévérité : combien d'utilisateurs ? service dégradé ou totalement interrompu ?
4. **Mitigation** — rétablir le service *avant* de comprendre la cause racine. Promouvoir un réplica, basculer le trafic, libérer de l'espace : l'objectif immédiat est de **stopper l'hémorragie**, pas de diagnostiquer.
5. **Résolution** — le service est rétabli et stable.
6. **Postmortem** — l'analyse a posteriori, traitée plus bas.

Le point le plus contre-intuitif pour un débutant est la priorité de la **phase 4 sur la cause racine** : en pleine indisponibilité, restaurer le service prime sur la compréhension. Le diagnostic approfondi vient après, à froid.

## Sévérité de l'incident et astreinte

À ne pas confondre avec la sévérité d'*alerte* : la sévérité d'*incident* qualifie l'impact métier global. Une échelle courante distingue un **SEV1** (interruption totale, mobilisation maximale), un **SEV2** (dégradation majeure d'une fonctionnalité importante) et un **SEV3** (impact mineur, traitable en heures ouvrées). Cette classification conditionne qui est mobilisé et à quelle vitesse.

L'**astreinte** (*on-call*) organise la disponibilité humaine : une rotation équitable, une politique d'**escalade** (si le premier niveau n'acquitte pas en N minutes, on remonte au suivant), et des outils dédiés (PagerDuty, Opsgenie) pour gérer le tout. Une astreinte saine repose sur des alertes pertinentes (cf. Partie A) : c'est la condition pour qu'elle reste tenable dans la durée.

## Runbooks : relier l'alerte au remède

Un opérateur réveillé en pleine nuit ne doit pas réinventer la procédure. Un **runbook** est une procédure documentée, pas-à-pas, associée à un type d'incident : comment diagnostiquer, quelles commandes exécuter, quand escalader. L'annotation `runbook_url` de la règle d'alerte (vue en Partie A) crée le lien direct : la notification contient l'URL de la marche à suivre.

Pour MariaDB, les runbooks s'appuient largement sur les chapitres existants de cette formation : la procédure de **failover** renvoie au [chapitre 14](../14-haute-disponibilite/06-failover-automatique.md), la **restauration** au [chapitre 12](../12-sauvegarde-restauration/05-restauration.md), le **diagnostic de réplication** à [§13.7.3](../13-replication/07.3-erreurs-courantes.md). Un bon runbook ne duplique pas ces contenus : il les **orchestre** sous forme de checklist actionnable propre au contexte de production.

## Rôles en incident majeur

Pour un SEV1, la coordination devient un enjeu en soi. On distingue alors des rôles : l'**Incident Commander** (qui pilote, décide et n'a pas les mains dans le clavier), des **intervenants techniques** (qui exécutent la remédiation), et un responsable de la **communication** (qui informe les parties prenantes et tient à jour une *status page*). Séparer la coordination de l'exécution évite le chaos où tout le monde diagnostique et personne ne décide.

## Apprendre : le postmortem *blameless*

Une fois l'incident clos, le **postmortem** (ou *retour d'expérience*) en extrait les enseignements. Son principe fondateur est d'être **blameless** (sans recherche de coupable) : on part du postulat que les personnes ont agi rationnellement avec l'information dont elles disposaient, et l'on cherche les **causes systémiques** — un manque d'alerte, une procédure absente, une configuration fragile — plutôt qu'une faute individuelle. Chercher un coupable détruit la transparence ; chercher une cause systémique améliore le système.

Un postmortem utile reconstitue une **chronologie** factuelle, identifie la cause racine, et surtout débouche sur des **actions correctives** assignées et suivies (« ajouter une alerte sur la taille du cluster Galera », « automatiser la rotation des binlogs »). Sans actions de suivi, le postmortem n'est qu'un compte rendu.

## Playbooks d'incident spécifiques à MariaDB

| Incident | Première action (mitigation) | Référence |
|----------|------------------------------|-----------|
| Réplica arrêté | Diagnostiquer l'erreur, sauter/corriger l'événement fautif | [§13.7.3](../13-replication/07.3-erreurs-courantes.md) |
| Primaire en panne | Promouvoir un réplica / déclencher le failover | [§13.8](../13-replication/08-failover-switchover.md), [§14.6](../14-haute-disponibilite/06-failover-automatique.md) |
| Split-brain Galera | Rétablir le quorum, désigner le composant primaire | [§14.3](../14-haute-disponibilite/03-split-brain-quorum.md) |
| Disque plein | Libérer de l'espace (binlogs, temp), purger | [§11.5.3](../11-administration-configuration/05.3-purge-rotation.md) |
| Requête folle (*runaway*) | Identifier puis `KILL` la requête bloquante | [annexe C.1](../annexes/c-requetes-sql-reference/01-requetes-administration.md) |
| Pic de deadlocks | Analyser les transactions concurrentes | [§6.5](../06-transactions-et-concurrence/05-deadlocks-resolution.md) |
| Corruption / perte de données | Restauration + recovery point-in-time | [chapitre 12](../12-sauvegarde-restauration/05.2-pitr.md) |

> 🆕 **Note 12.x sur la récupération.** La récupération point-in-time (PITR) rejoue les *binary logs* pour ramener l'instance à un instant précis ([§12.5.2](../12-sauvegarde-restauration/05.2-pitr.md)). En série 12.x, le binlog a été **réécrit et intégré à InnoDB** ([§11.5.4](../11-administration-configuration/05.4-binlog-innodb-performance.md)) : les principes de la PITR restent identiques, mais ce changement de fondations mérite d'être pris en compte lors de la validation des procédures de restauration sur une instance migrée.

## Mesurer le processus : MTTA et MTTR

Le processus de réponse se mesure et s'améliore. Deux indicateurs font référence : le **MTTA** (*Mean Time To Acknowledge*), délai moyen entre l'alerte et sa prise en charge — il reflète la santé de l'astreinte et la pertinence des alertes ; et le **MTTR** (*Mean Time To Recovery*), délai moyen jusqu'au rétablissement du service — il reflète l'efficacité de la remédiation, donc la qualité des runbooks et de l'automatisation. Un MTTA qui se dégrade trahit souvent une fatigue d'alertes ; un MTTR élevé pointe des procédures manquantes ou trop manuelles.

## Points clés à retenir

L'alerting transforme l'observabilité passive en réaction automatique : on évalue des conditions (en PromQL via Prometheus, routées par **Alertmanager** en production, ou via Grafana pour les cas simples), puis on notifie. La qualité tient à la **pertinence**, pas au volume : alerter sur les **symptômes** actionnables, hiérarchiser par sévérité (*page* vs *ticket*), et lutter contre la fatigue d'alertes — l'approche SLO/error budget poussant cette logique le plus loin. Côté MariaDB, le socle d'alertes critiques couvre l'indisponibilité, l'arrêt de réplication, la santé Galera et — via le `node_exporter` — l'espace disque. L'**incident response** prend le relais : détecter, acquitter, **mitiger avant de diagnostiquer**, résoudre, puis tirer les leçons dans un **postmortem blameless** orienté causes systémiques. Les **runbooks** (reliés aux alertes par `runbook_url`) orchestrent les procédures de failover, de restauration et de diagnostic décrites ailleurs dans cette formation, et les indicateurs **MTTA/MTTR** permettent d'améliorer le processus en continu.

---

↩️ [Section précédente : 16.10 — Observabilité : Logs, Metrics, Traces](10-observabilite.md)  
➡️ **Section suivante :** [16.12 — GitOps pour les bases de données](12-gitops-bases-donnees.md)

⏭️ [GitOps pour les bases de données](/16-devops-automatisation/12-gitops-bases-donnees.md)

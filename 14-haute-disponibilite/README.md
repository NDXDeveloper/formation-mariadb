🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 14 — Haute Disponibilité

> **Partie 6 : Réplication et Haute Disponibilité (DBA/DevOps)**  
> Version de référence : **MariaDB 12.3 LTS** · LTS précédente de comparaison : 11.8

La haute disponibilité (*High Availability*, HA) regroupe l'ensemble des techniques, architectures et outils qui permettent à un service de base de données de **rester accessible malgré les pannes** — qu'elles soient matérielles, logicielles, réseau ou humaines. Là où le chapitre 13 a posé les fondations de la *réplication* (comment propager les données d'un serveur à un autre), ce chapitre s'intéresse à la question complémentaire : **comment transformer cette redondance de données en disponibilité continue du service**, automatiser la bascule en cas d'incident, et présenter aux applications un point d'accès stable.

---

## À qui s'adresse ce chapitre ?

Ce chapitre fait partie des parcours **Administrateur/DBA** et **DevOps/Cloud**. Il s'adresse à toute personne responsable de la disponibilité d'une base MariaDB en production : DBA, ingénieur SRE/DevOps, architecte d'infrastructure.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- distinguer les grandes **architectures de haute disponibilité** et leurs compromis (synchrone vs asynchrone, actif/actif vs actif/passif) ;
- déployer et exploiter un **cluster MariaDB Galera** (réplication synchrone multi-maître) et comprendre la *certification-based replication* ;
- diagnostiquer et prévenir les situations de **split-brain** grâce aux mécanismes de **quorum** ;
- mettre en place **MaxScale** comme proxy intelligent (load balancing, *read/write split*, routage, pare-feu) et exploiter ses fonctionnalités récentes (*Workload Capture/Replay*, *Diff Router*) ;
- concevoir une stratégie de **failover automatique** et de récupération après incident ;
- assurer un point d'entrée stable via **Virtual IP / keepalived** ou des alternatives comme **ProxySQL** et **HAProxy** ;
- offrir une expérience transparente au client grâce au **Transaction Replay** et à la **Connection Migration**.

---

## Prérequis

Ce chapitre suppose une bonne maîtrise des notions abordées précédemment :

| Prérequis | Pourquoi c'est nécessaire |
|-----------|---------------------------|
| **Chapitre 13 — Réplication** | La HA s'appuie sur la réplication (asynchrone, semi-synchrone, GTID). |
| **Chapitre 6 — Transactions et Concurrence** | Comprendre ACID, l'isolation et les verrous est indispensable pour raisonner sur la cohérence en cluster. |
| **Chapitre 11 — Administration et Configuration** | La configuration du serveur (binlog, variables système) conditionne la HA. |
| **Chapitre 12 — Sauvegarde et Restauration** | La HA ne remplace pas les sauvegardes ; elle les complète. |

---

## Pourquoi la haute disponibilité ?

Une indisponibilité de base de données a un coût direct (perte de chiffre d'affaires, pénalités contractuelles via les SLA) et indirect (perte de confiance, dette opérationnelle). L'objectif de la HA est de **réduire ce coût** en agissant sur deux indicateurs fondamentaux :

- **RTO — *Recovery Time Objective*** : durée maximale d'interruption tolérée. *« En combien de temps le service doit-il redevenir opérationnel ? »*
- **RPO — *Recovery Point Objective*** : quantité maximale de données que l'on accepte de perdre. *« Jusqu'à quel point dans le passé peut-on remonter ? »*

Une réplication **synchrone** (Galera) vise un RPO proche de zéro mais impose un surcoût à chaque écriture ; une réplication **asynchrone** offre de meilleures performances en écriture au prix d'un RPO non nul en cas de bascule.

### Les « neuf » de la disponibilité

La disponibilité s'exprime souvent en pourcentage de temps de fonctionnement. Voici l'indisponibilité annuelle correspondante :

| Disponibilité | Surnom | Indisponibilité / an |
|--------------|--------|----------------------|
| 99 % | *two nines* | ≈ 3,65 jours |
| 99,9 % | *three nines* | ≈ 8,77 heures |
| 99,99 % | *four nines* | ≈ 52,6 minutes |
| 99,999 % | *five nines* | ≈ 5,26 minutes |
| 99,9999 % | *six nines* | ≈ 31,5 secondes |

Chaque « neuf » supplémentaire augmente significativement la complexité et le coût de l'architecture. L'enjeu n'est pas de viser systématiquement le maximum, mais de **dimensionner la HA en fonction des besoins réels** du service.

---

## Vocabulaire clé du chapitre

| Terme | Définition courte |
|-------|-------------------|
| **SPOF** (*Single Point of Failure*) | Composant unique dont la panne entraîne l'arrêt de tout le service. La HA vise à les éliminer. |
| **Failover** | Bascule **automatique** vers un nœud sain après détection d'une panne. |
| **Switchover** | Bascule **planifiée et contrôlée** (maintenance, mise à jour), sans incident. |
| **Quorum** | Majorité de nœuds nécessaire pour qu'un cluster prenne des décisions et reste opérationnel. |
| **Split-brain** | Situation où une partition réseau fait croire à plusieurs sous-groupes qu'ils sont seuls maîtres, menaçant la cohérence. |
| **SST / IST** | *State Snapshot Transfer* (copie complète) et *Incremental State Transfer* (différentiel) : méthodes de synchronisation d'un nœud Galera rejoignant le cluster. |
| **wsrep** | *Write Set Replication* : l'API et le protocole de réplication synchrone utilisés par Galera. |
| **VIP** (*Virtual IP*) | Adresse IP flottante basculée d'un serveur à l'autre pour offrir un point d'accès stable. |

---

## Panorama des approches de HA sous MariaDB

Il n'existe pas une seule « bonne » architecture : le choix dépend du profil de charge, de la tolérance à la perte de données et de la répartition géographique.

| Approche | Topologie | RPO | Points forts | Points de vigilance |
|----------|-----------|-----|--------------|---------------------|
| **Galera Cluster** | Synchrone, multi-maître (actif/actif) | ≈ 0 | Pas de perte de données, adhésion automatique, lectures réparties | Écritures limitées par le nœud le plus lent et la latence réseau ; sensible aux gros transactions et aux points chauds |
| **Réplication + proxy** | Asynchrone / semi-synchrone (actif/passif) | > 0 | Excellentes performances en écriture, souplesse géographique | Bascule à orchestrer, *replication lag*, perte possible en asynchrone |
| **Proxy intelligent (MaxScale)** | Au-dessus des deux précédents | — | *Read/write split*, routage, failover automatique, pare-feu | Composant à rendre lui-même hautement disponible |

Ces approches se combinent fréquemment : par exemple un cluster Galera placé derrière MaxScale, ou une réplication asynchrone entre deux clusters Galera géo-distribués.

---

## Positionnement MariaDB 12.3 LTS

La 12.3 LTS apporte plusieurs évolutions qui touchent directement la haute disponibilité :

- **Galera packagé séparément** : le support Galera est désormais fourni par un paquet dédié, `mariadb-server-galera`, et n'est plus une dépendance du serveur lui-même (voir 14.2.5). Cela a un impact concret sur les images Docker officielles et le déploiement via l'operator Kubernetes.
- **Retry des *write sets*** : la variable `wsrep_applier_retry_count` permet de rejouer automatiquement les *write sets* en cas de conflit transitoire (voir 14.2.6).
- **Réplication parallèle entre clusters Galera** : amélioration de `slave_parallel_threads` pour réduire le *lag* entre clusters (voir 13.11).

Ces points sont signalés par le marqueur 🆕 dans le plan ci-dessous et détaillés dans leurs sections respectives.

---

## Plan du chapitre

Le chapitre progresse des **concepts** vers les **outils**, puis vers les **stratégies opérationnelles**.

1. **[14.1 — Architectures haute disponibilité : Concepts](01-architectures-ha-concepts.md)**
   Fondations : SPOF, RTO/RPO, redondance, actif/actif vs actif/passif.

2. **[14.2 — MariaDB Galera Cluster](02-galera-cluster.md)**
   Le cœur de la HA synchrone sous MariaDB.
   - [14.2.1 — Architecture synchrone multi-master](02.1-architecture-synchrone.md)
   - [14.2.2 — Certification-based replication](02.2-certification-based.md)
   - [14.2.3 — Configuration et déploiement](02.3-configuration-deploiement.md)
   - [14.2.4 — State transfers (SST, IST)](02.4-state-transfers.md)
   - [14.2.5 — Packaging Galera en 12.3 : paquet `mariadb-server-galera` séparé](02.5-packaging-galera.md) 🆕
   - [14.2.6 — Retry des write sets (`wsrep_applier_retry_count`)](02.6-retry-write-sets.md) 🆕

3. **[14.3 — Split-brain et quorum](03-split-brain-quorum.md)**
   Garantir la cohérence lors des partitions réseau.

4. **[14.4 — MaxScale](04-maxscale.md)**
   Le proxy de base de données de MariaDB.
   - [14.4.1 — Load Balancing](04.1-load-balancing.md)
   - [14.4.2 — Read/Write Split](04.2-read-write-split.md)
   - [14.4.3 — Query Routing](04.3-query-routing.md)
   - [14.4.4 — Database Firewall](04.4-database-firewall.md)

5. **[14.5 — MaxScale : fonctionnalités récentes](05-maxscale-nouveautes.md)**
   - [14.5.1 — Workload Capture](05.1-workload-capture.md)
   - [14.5.2 — Workload Replay](05.2-workload-replay.md)
   - [14.5.3 — Diff Router](05.3-diff-router.md)

6. **[14.6 — Solutions de failover automatique](06-failover-automatique.md)**
   Détection de panne et bascule sans intervention humaine.

7. **[14.7 — Virtual IP et keepalived](07-virtual-ip-keepalived.md)**
   Offrir un point d'accès stable via une IP flottante (VRRP).

8. **[14.8 — Stratégies de récupération après incident](08-strategies-recuperation.md)**
   Procédures et plans de reprise.

9. **[14.9 — Alternatives : ProxySQL et HAProxy](09-proxysql-haproxy.md)**
   Comparaison avec d'autres couches de routage et d'équilibrage.

10. **[14.10 — Transaction Replay et Connection Migration](10-transaction-replay-connection-migration.md)**
    Rendre la bascule transparente pour le client.

---

## À retenir avant de commencer

- La **haute disponibilité n'est pas une sauvegarde** : un cluster réplique aussi les erreurs (un `DROP TABLE` accidentel se propage à tous les nœuds). HA et sauvegarde (chapitre 12) sont **complémentaires**, pas substituables.
- Tout composant introduit pour la HA (proxy, VIP) peut devenir à son tour un **SPOF** : il doit lui-même être rendu redondant.
- Le bon dimensionnement se fait **à partir des objectifs métier** (RTO/RPO, SLA), pas à partir de la technologie disponible.

---

⬅️ [Chapitre 13 — Réplication](../13-replication/README.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.1 — Concepts ➡️](01-architectures-ha-concepts.md)

⏭️ [Architectures haute disponibilité : Concepts](/14-haute-disponibilite/01-architectures-ha-concepts.md)

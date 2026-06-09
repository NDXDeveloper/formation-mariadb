🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.8 MariaDB dans les architectures Event-Driven

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Plusieurs sections précédentes ont fait allusion à la propagation des données **par événements** : pour synchroniser les vues entre microservices (§20.2), alimenter un entrepôt analytique (§20.3), ou décorréler les composants d'un système distribué (§20.7). Cette section traite ce sujet de front : comment MariaDB s'intègre dans une **architecture orientée événements** (*event-driven architecture*, EDA).

L'idée centrale est de transformer les **changements de données** survenant dans MariaDB en un **flux d'événements** que d'autres systèmes peuvent consommer en temps réel. Cette section pose les concepts ; les sous-sections suivantes en détaillent la mise en œuvre concrète — la capture de changements, le transport via Kafka, et le connecteur Debezium.

---

## Qu'est-ce qu'une architecture orientée événements ?

Dans une architecture orientée événements, les composants ne s'appellent pas directement les uns les autres : ils communiquent en **produisant** et en **consommant** des **événements**. Un événement est l'enregistrement **immuable** d'un fait survenu — « une commande a été passée », « un paiement a été reçu », « un stock a été mis à jour ».

Trois rôles structurent ce modèle :

- le **producteur** émet un événement lorsqu'un fait se produit ;
- le **consommateur** réagit à cet événement ;
- un **courtier d'événements** (*broker*) ou une **plateforme de streaming** (par exemple Kafka) transporte les événements entre les deux.

Ce fonctionnement est **asynchrone** et **découplé** : le producteur n'attend pas le consommateur et ignore même qui le consomme. Il s'oppose au modèle classique **requête-réponse**, synchrone, où l'appelant attend que l'appelé réponde.

---

## Pourquoi l'événementiel ?

L'EDA apporte des bénéfices qui prolongent ceux recherchés par les microservices (§20.2) :

- **Découplage** : producteurs et consommateurs évoluent indépendamment. On peut ajouter un nouveau consommateur (un service d'indexation, un cache, un entrepôt) **sans toucher** au producteur.
- **Passage à l'échelle** : le traitement asynchrone permet aux consommateurs de monter en charge à leur propre rythme.
- **Résilience** : un consommateur lent ou en panne ne bloque pas le producteur ; les événements peuvent être **tamponnés** et **rejoués** une fois le consommateur rétabli.
- **Réactivité en temps réel** : les systèmes réagissent aux changements au moment où ils surviennent.
- **Intégration** : l'événement devient un langage commun pour relier des systèmes hétérogènes.

---

## Le rôle de la base de données

Une base de données peut participer à une architecture événementielle de deux manières :

- comme **source d'événements** : lorsqu'une donnée change, un événement est émis. C'est le rôle le plus structurant, et celui sur lequel se concentre cette section ;
- comme **consommateur** : en projetant les événements dans un état interrogeable (vues de lecture, séparation lecture/écriture **CQRS**), évoquée en §20.2.

La question clé est donc : comment faire de MariaDB une **source fiable d'événements** ? La réponse n'est pas aussi évidente qu'il y paraît.

---

## Le problème de la double écriture

L'approche naïve consiste à faire **écrire à l'application dans la base** *et* **publier l'événement** dans le courtier. Mais ce sont **deux opérations distinctes**, sans transaction commune : si l'une réussit et l'autre échoue (panne réseau, redémarrage), le système se retrouve **incohérent** — une commande enregistrée sans événement émis, ou un événement émis pour une commande jamais validée. C'est le **problème de la double écriture** (*dual-write problem*).

Deux solutions s'imposent :

- **Le pattern « outbox » transactionnel** : l'application écrit l'événement dans une table « boîte d'envoi » (*outbox*) **dans la même transaction** que la donnée métier. Un processus séparé lit ensuite cette table et publie les événements. La cohérence est garantie parce que donnée et événement sont validés atomiquement.
- **La capture de changements (CDC)** : plutôt que de publier explicitement, on **dérive les événements du journal des modifications** de la base elle-même. Le **commit** de la transaction devient alors la source unique de vérité, et il n'y a plus de double écriture du tout.

La CDC est l'approche la plus propre, car elle ne demande aucune publication explicite par l'application : tout changement validé dans MariaDB devient automatiquement un événement. C'est précisément l'objet de la section §20.8.1.

---

## MariaDB comme source d'événements : le journal binaire

Le mécanisme qui rend la CDC possible existe déjà au cœur de MariaDB : le **journal binaire** (*binlog*, chapitre 11.5). Le binlog enregistre **chaque modification** de données — c'est lui qui sert déjà à la réplication. Un outil de CDC se contente de **lire ce journal** pour en extraire un flux d'événements de changement, **sans aucune modification de l'application**.

Quelques points techniques conditionnent cet usage :

- **Le format ROW** (§11.5.2) est requis : la CDC a besoin des changements **au niveau ligne** (les valeurs avant/après), et non des seules instructions SQL.

  ```ini
  [mariadb]
  log_bin       = ON
  binlog_format = ROW      # format requis pour la capture de changements
  ```

- **Les GTID** (§13.4) permettent à l'outil de CDC de suivre sa **position** de façon fiable et de reprendre sans perte après une interruption.
- **Le nouveau binlog intégré à InnoDB** de la série 12.x (§11.5.4), nettement plus performant en écriture, profite directement aux pipelines de CDC qui s'appuient sur ce journal.

Ainsi, sans rien changer au code applicatif, MariaDB devient une source d'événements alimentée par son propre flux de transactions.

---

## Les patterns rendus possibles

Faire de MariaDB une source d'événements ouvre la voie à de nombreux usages, détaillés ailleurs dans la formation :

- **Alimentation d'un entrepôt analytique** : streamer les changements opérationnels vers ColumnStore (§20.3) pour une analytique quasi temps réel.
- **Propagation entre microservices** : diffuser les changements d'un service vers les vues, caches ou bases d'autres services (§20.2), dans le respect de leur autonomie.
- **Vues de lecture (CQRS)** : construire des projections optimisées pour la lecture à partir du flux d'événements.
- **Synchronisation de systèmes annexes** : maintenir à jour un index de recherche, un cache, ou un autre magasin de données.
- **Pipelines de données temps réel** : alimenter en continu des chaînes de traitement analytiques ou décisionnelles.

---

## Ce que couvrent les sections suivantes

Les trois sous-sections construisent, pas à pas, un pipeline événementiel concret à partir de MariaDB :

- **§20.8.1 — CDC (Change Data Capture)** : la **technique** de capture des changements à partir du journal binaire.
- **§20.8.2 — Intégration avec Kafka** : la **plateforme de streaming** qui transporte et conserve les événements à grande échelle.
- **§20.8.3 — Debezium connector** : le **connecteur** qui lit le binlog de MariaDB et publie les changements dans Kafka, reliant concrètement CDC et Kafka.

Ensemble, ces trois briques transforment MariaDB en source d'un flux d'événements fiable, sur lequel reposent les architectures temps réel modernes. La section suivante (§20.8.1) commence par le fondement de tout l'édifice : la capture de changements.

---

## Navigation

⬅️ Section précédente : [20.7 Scaling vertical vs horizontal](07-scaling-vertical-horizontal.md)  
➡️ Section suivante : [20.8.1 CDC (Change Data Capture)](08.1-cdc.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [CDC (Change Data Capture)](/20-cas-usage-architectures/08.1-cdc.md)

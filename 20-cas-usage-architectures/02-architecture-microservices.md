🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.2 Architecture microservices

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

L'architecture **microservices** consiste à concevoir une application non plus comme un bloc unique, mais comme un ensemble de petits services indépendants, chacun responsable d'une capacité métier précise (la gestion des commandes, le catalogue produits, la facturation, les notifications…). Ces services sont développés, déployés et mis à l'échelle séparément, et communiquent entre eux via des interfaces bien définies — API REST, gRPC ou messages asynchrones.

Ce style architectural est devenu une référence pour les systèmes à grande échelle, mais il déplace la complexité plutôt qu'il ne la supprime : ce qui était auparavant un appel de fonction interne devient un appel réseau, et ce qui était une transaction unique sur une seule base devient une coordination entre plusieurs bases distinctes. C'est précisément sur ce terrain — la **gestion des données** — que se jouent les décisions les plus structurantes, et c'est ce qui motive ce chapitre.

Cette section pose le cadre général : elle explique pourquoi la donnée est le point névralgique des microservices, introduit le principe d'**autonomie des données**, et présente les deux grandes approches de stockage qui seront détaillées dans les sous-sections suivantes.

---

## Du monolithe aux microservices

Dans une application **monolithique** classique, l'ensemble du code partage une seule base de données. Cette unicité présente des avantages réels : les jointures entre toutes les tables sont triviales, et une transaction ACID unique peut garantir la cohérence d'une opération qui touche plusieurs domaines métier à la fois. En contrepartie, le monolithe se fait rigide à mesure qu'il grossit : une modification de schéma peut avoir des répercussions imprévues, les équipes se gênent mutuellement, et la mise à l'échelle se fait en bloc plutôt que là où le besoin se trouve.

Le passage aux microservices répond à ces limites en **découpant** l'application selon les frontières du métier. Chaque service gagne en autonomie : on peut le faire évoluer, le déployer et le dimensionner sans toucher aux autres. Mais ce découpage ne vaut que s'il s'accompagne d'un découpage équivalent des données — sans quoi les services resteraient enchaînés les uns aux autres par une base commune, et l'on n'aurait qu'un monolithe distribué, cumulant les inconvénients des deux modèles.

---

## La donnée, point névralgique des microservices

Le principal défi des microservices n'est pas le code, mais la **donnée**. Trois questions reviennent systématiquement :

- **Comment garantir la cohérence** d'une opération qui s'étend sur plusieurs services, alors qu'on ne dispose plus d'une transaction ACID unique pour les englober ?
- **Comment répondre à une requête** qui a besoin de croiser des données appartenant à plusieurs services, alors qu'une jointure SQL classique entre leurs bases n'est plus possible ?
- **Comment maintenir un couplage faible**, c'est-à-dire éviter que les services ne deviennent dépendants des détails internes de stockage les uns des autres ?

La réponse à ces questions détermine toute la robustesse de l'architecture. Mal résolue, la gestion des données transforme l'élégance théorique des microservices en source de fragilité opérationnelle.

---

## Le principe d'autonomie des données

Le principe fondateur de la gestion des données en microservices est la **décentralisation** : chaque service est le **propriétaire exclusif** de ses données. Concrètement, cela signifie qu'un service ne doit jamais accéder directement à la base d'un autre service. S'il a besoin d'une information détenue par un autre service, il la demande **via l'API** de ce service, jamais en interrogeant sa base par-dessus son épaule.

Cette règle a une conséquence majeure : tant que le contrat d'API reste stable, chaque service peut faire évoluer librement son schéma interne, changer de modèle de données, voire changer de moteur de stockage, sans que les autres en soient affectés. C'est cette indépendance qui rend le couplage faible et autorise l'évolution autonome — la promesse même des microservices.

Ce principe ouvre aussi la voie à la **persistance polyglotte** (*polyglot persistence*) : chaque service choisit le type de stockage le plus adapté à son besoin. Un service transactionnel s'appuiera sur MariaDB avec InnoDB ; un service de recherche sémantique pourra exploiter la recherche vectorielle (Vector) de MariaDB (voir §20.9) ; un autre encore pourra recourir à un store documentaire ou à un cache clé-valeur. Dans une telle architecture, MariaDB est souvent **l'un des magasins de données parmi plusieurs**, choisi là où ses forces relationnelles et transactionnelles font la différence.

---

## Deux approches, un compromis

Le respect de l'autonomie des données peut se décliner selon plusieurs niveaux de séparation, qui forment un continuum entre **isolation** et **mutualisation**. Les deux sous-sections suivantes en présentent les deux extrémités :

| Approche | Section | Principe | Couplage |
|----------|---------|----------|----------|
| **Base par service** | §20.2.1 | Chaque service possède sa propre base, totalement séparée | Faible (recommandé) |
| **Base partagée** | §20.2.2 | Plusieurs services se partagent une même base | Fort (à manier avec précaution) |

La **base par service** est l'approche canonique de la littérature microservices : elle respecte pleinement l'autonomie des données et maximise le découplage. La **base partagée** est une approche plus pragmatique, parfois justifiée (migration progressive depuis un monolithe, périmètre limité, contraintes opérationnelles), mais qui réintroduit un couplage entre services et doit être adoptée en connaissance de cause. Le choix entre les deux est un **compromis** entre découplage et simplicité — un thème récurrent de tout ce chapitre.

---

## Les défis transversaux

Quel que soit le niveau de séparation retenu, plusieurs défis sont inhérents à la répartition des données entre services. Ils ne sont pas spécifiques à MariaDB, mais leur compréhension est indispensable pour bien situer les patterns de stockage.

- **La cohérence sans transaction globale** : on ne peut généralement pas envelopper une opération inter-services dans une seule transaction ACID. On bascule alors vers la **cohérence à terme** (*eventual consistency*), souvent orchestrée par le pattern **Saga** : une suite de transactions locales, chacune confirmée dans son propre service, avec des actions de compensation en cas d'échec. (Les transactions distribuées XA, vues au chapitre 6, existent mais sont rarement le bon outil entre microservices, car elles recréent un couplage fort.)
- **Les requêtes inter-services** : sans jointure possible entre bases distinctes, on recourt à des stratégies comme la **composition par API** (chaque service répond pour sa part, l'appelant agrège) ou la séparation lecture/écriture (**CQRS**), où une vue de lecture dédiée, alimentée par événements, regroupe les données nécessaires.
- **La propagation des changements** : pour maintenir des vues cohérentes ou répliquer certaines données, les services s'échangent des **événements**. La capture de changements (CDC) à partir du journal binaire de MariaDB, associée à Kafka et Debezium, est un mécanisme privilégié à cet effet (voir §20.8).

Ces défis expliquent pourquoi le choix de l'approche de stockage n'est jamais purement technique : il engage toute la manière dont les services collaboreront.

---

## MariaDB dans une architecture microservices

MariaDB se prête bien à ce style architectural, pour plusieurs raisons :

- **Légèreté et conteneurisation** : une instance MariaDB démarre rapidement et s'intègre naturellement dans un conteneur. On peut ainsi dédier une instance à chaque service, ou regrouper plusieurs bases logiques sur une instance mutualisée, selon le degré d'isolation souhaité.
- **Déploiement automatisé** : sur Kubernetes, le **mariadb-operator** (voir §16.5) facilite le provisionnement, la réplication et les sauvegardes par service, ce qui rend l'approche « une base par service » réaliste à grande échelle.
- **Gestion des connexions** : multiplier les services multiplie les connexions à la base. Le **connection pooling** (chapitre 17), éventuellement renforcé par un proxy comme ProxySQL ou MaxScale, devient un élément clé pour éviter l'épuisement des connexions.
- **Polyvalence fonctionnelle** : du transactionnel pur (InnoDB) à la recherche vectorielle (Vector) en passant par l'analytique (ColumnStore), MariaDB peut couvrir des besoins variés au sein d'un même écosystème de services, tout en restant un outil familier des équipes.

L'objectif n'est pas d'imposer MariaDB partout, mais de l'employer là où un stockage relationnel fiable et transactionnel apporte le plus de valeur, dans le respect de l'autonomie de chaque service.

---

## Ce que couvrent les sections suivantes

Les deux sous-sections approfondissent les deux approches introduites ici :

- **§20.2.1 — Database per service** : le modèle d'isolation maximale, ses bénéfices, ses contraintes et sa mise en œuvre avec MariaDB.
- **§20.2.2 — Shared database pattern** : l'approche mutualisée, ses cas d'usage légitimes, ses risques de couplage et les précautions à prendre.

À l'issue de ces sections, le lecteur disposera des éléments nécessaires pour arbitrer entre découplage et simplicité selon son contexte, avant d'aborder le cas voisin de l'**architecture multi-tenant** (§20.4), qui pose des questions de cloisonnement de nature comparable.

---

## Navigation

⬅️ Section précédente : [20.1 OLTP vs OLAP](01-oltp-vs-olap.md)  
➡️ Section suivante : [20.2.1 Database per service](02.1-database-per-service.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Database per service](/20-cas-usage-architectures/02.1-database-per-service.md)

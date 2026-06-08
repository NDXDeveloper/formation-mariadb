🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.5 — mariadb-operator pour Kubernetes

> Chapitre 16 — DevOps et Automatisation · Vue d'ensemble de la section, avant l'installation, les déploiements Galera et réplication, et les sauvegardes.

---

## Introduction

Tout au long de la section 16.4, un fil conducteur est revenu : les primitives Kubernetes — StatefulSet, volumes persistants, Service — permettent d'**exécuter des pods MariaDB**, mais ne prennent en charge **aucune des opérations du jour 2** (amorçage de cluster, bascule, sauvegardes, réplication, montées de version). §16.4.1 le concluait sans détour : un StatefulSet n'est pas un cluster, et la gestion manuelle de manifestes bruts n'est pas viable en production. C'est précisément le vide que comble un **opérateur**.

Cette section présente le **mariadb-operator**, l'opérateur **open source** pour MariaDB sur Kubernetes. La section suivante (§16.6) traitera de l'**opérateur Enterprise**, son pendant commercial. La présente page pose les concepts ; les sous-sections détailleront l'installation et les CRDs, les déploiements Galera et réplication, et les sauvegardes automatisées.

## Qu'est-ce qu'un opérateur

Le **patron de l'opérateur** (*operator pattern*) étend Kubernetes selon une mécanique précise. D'une part, des **définitions de ressources personnalisées** (CRD, *Custom Resource Definitions*) ajoutent de **nouveaux types d'objets déclaratifs** au cluster — par exemple un objet « MariaDB ». D'autre part, un **contrôleur** observe ces objets et **réconcilie** sans cesse l'état réel avec l'état déclaré, exactement comme le plan de contrôle le fait pour les objets natifs.

L'apport décisif d'un opérateur est d'**encoder le savoir-faire opérationnel** d'une application donnée — ici, l'expertise qu'un DBA appliquerait à MariaDB. Plutôt que d'assembler et de coordonner soi-même les primitives, on **déclare une intention** — « je veux un cluster Galera de trois nœuds, avec telles sauvegardes » — et l'opérateur se charge du reste : provisionnement, amorçage, bascule, sauvegardes. C'est la réponse directe au vide identifié en §16.4.1.

## Pourquoi un opérateur pour MariaDB

L'intérêt d'un opérateur pour une base de données tient à ce qu'il **automatise ce que les primitives laissent à la charge de l'humain**. Il prend en main l'**amorçage** d'un cluster, la **bascule** et la reprise après panne, la **configuration de la réplication**, les **sauvegardes et restaurations**, les **montées de version** et la **mise à l'échelle** — autant d'opérations délicates et sujettes à erreurs lorsqu'elles sont conduites à la main.

À cette automatisation s'ajoutent trois bénéfices. L'approche est **déclarative**, donc naturellement compatible avec une démarche GitOps (§16.12). Elle **réduit la charge d'exploitation et le risque d'erreur humaine**. Et elle **encapsule les bonnes pratiques**, l'opérateur appliquant des choix éprouvés là où une mise en œuvre artisanale multiplierait les écueils. En somme, c'est l'opérateur qui rend **praticable en production** ce que §16.4.1 déconseillait de faire à la main.

## Le mariadb-operator

Le **mariadb-operator** est un opérateur **open source** qui gère le **cycle de vie complet** de MariaDB sur Kubernetes, de façon déclarative et via des CRDs. Son objet central est la ressource **`MariaDB`**, qui décrit une instance ou un cluster ; d'autres ressources couvrent les sauvegardes, les bases, les comptes et les privilèges (la liste exhaustive et à jour est l'objet de §16.5.1).

Un point essentiel pour bien le situer : l'opérateur **ne remplace pas** les primitives de la section 16.4, il les **emploie**. Lorsqu'on déclare une ressource `MariaDB`, l'opérateur **génère** sous le capot un StatefulSet (§16.4.1), des demandes de volume résolues via la StorageClass choisie (§16.4.2), un Service sans tête, une ConfigMap et un Secret. La compréhension acquise dans la section précédente éclaire donc directement ce que l'opérateur réalise.

Côté topologies, le mariadb-operator prend en charge à la fois la **mise en cluster Galera** (multi-maître, §16.5.2) et la **réplication** primaire/réplicas (§16.5.3), c'est-à-dire les deux grandes approches de haute disponibilité du chapitre 14.

## Ce que gère l'opérateur

À grands traits, les capacités de l'opérateur se répartissent ainsi — chacune étant approfondie dans une sous-section ou dans §16.5.1.

- **Provisionner** une instance ou un cluster MariaDB, via la ressource `MariaDB`.
- Mettre en place des **topologies de haute disponibilité** : Galera (§16.5.2) et réplication (§16.5.3).
- Automatiser et **planifier les sauvegardes et restaurations** (§16.5.4).
- Gérer de façon **déclarative les bases, les utilisateurs et les privilèges**, au moyen de ressources dédiées.
- Administrer les **connexions et les secrets** associés.
- Selon les versions, **s'intégrer à MaxScale** pour le routage et la haute disponibilité (chapitre 14).

L'inventaire précis des CRDs et de leurs champs, susceptible d'évoluer d'une version de l'opérateur à l'autre, relève de §16.5.1 et gagne à être vérifié dans la documentation de la version employée.

## Opérateur communautaire et opérateur Enterprise

Deux opérateurs coexistent pour MariaDB sur Kubernetes, et il est utile de les distinguer dès maintenant. Cette section traite du **mariadb-operator communautaire**, open source et largement adopté. La section 16.6 présentera l'**opérateur Enterprise** de MariaDB, offre **commerciale** assortie d'un support et de fonctionnalités orientées entreprise, souvent associée à MariaDB Enterprise Server.

Le choix entre les deux se pose en toute objectivité, selon le **besoin de support**, les **fonctionnalités** requises et le **modèle de licence**. L'opérateur communautaire est pleinement capable et convient à de nombreux contextes ; l'opérateur Enterprise répond aux organisations qui privilégient un support contractuel. Les éléments de décision seront précisés en §16.6.

## Le packaging Galera de la 12.3, pris en charge

Le thème récurrent de ce chapitre trouve ici sa résolution la plus confortable. La contrainte de la 12.3 — disposer du paquet **`mariadb-server-galera`**, désormais distinct, pour un déploiement en cluster (§16.3.1, §14.2.5) — fait partie des détails que l'opérateur **abstrait pour l'utilisateur** : on déclare une topologie Galera, et l'opérateur s'assure des prérequis. Le déploiement Galera assisté par l'opérateur est détaillé en §16.5.2.

## Une approche déclarative, alignée sur GitOps

Parce que l'opérateur se pilote **entièrement par des objets déclaratifs** rédigés en YAML, il s'inscrit naturellement dans une démarche **GitOps** (§16.12) : l'état désiré de la base — topologie, sauvegardes, comptes — réside dans Git, et se trouve réconcilié à la fois par l'outil GitOps et par l'opérateur lui-même. Cette double boucle de réconciliation est l'aboutissement de la logique d'Infrastructure as Code posée en ouverture du chapitre (§16.1).

## Ce que couvrent les sous-sections

À partir de cette vue d'ensemble, les sous-sections entrent dans le concret. La première (§16.5.1) traite de l'**installation de l'opérateur et de ses CRDs**. La deuxième (§16.5.2) détaille le **déploiement d'un cluster Galera** avec l'opérateur, packaging `mariadb-server-galera` compris. La troisième (§16.5.3) aborde la **réplication** pilotée par l'opérateur. La quatrième (§16.5.4) présente les **sauvegardes automatisées**.

---

↩️ [Section précédente : 16.4 — Orchestration avec Kubernetes](04-orchestration-kubernetes.md)  
➡️ **Sous-section suivante :** [16.5.1 — Installation et CRDs](05.1-installation-crds.md)

⏭️ [Installation et CRDs](/16-devops-automatisation/05.1-installation-crds.md)

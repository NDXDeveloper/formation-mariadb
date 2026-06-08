🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.2 — Déploiement avec Ansible/Terraform

> Chapitre 16 — DevOps et Automatisation · Vue d'ensemble de la section, avant l'étude détaillée de chaque outil.

---

## Introduction

La section précédente (§16.1) a posé un principe : l'automatisation d'une plateforme MariaDB s'articule autour de deux disciplines complémentaires, le **provisionnement** de l'infrastructure et la **gestion de configuration** des systèmes. Cette section donne un visage concret à ces deux disciplines en présentant les deux outils qui les incarnent le plus souvent dans l'écosystème : **Terraform** pour le provisionnement, **Ansible** pour la configuration.

L'objectif ici n'est pas encore d'écrire des playbooks ou des fichiers de provisionnement — ce sera l'objet des deux sous-sections suivantes — mais de comprendre ce que fait chaque outil, en quoi ils diffèrent, et surtout **comment ils s'enchaînent** pour déployer MariaDB de bout en bout. C'est cette articulation, que ni l'une ni l'autre des sous-sections ne traite isolément, qui constitue le cœur de la présente section.

## Deux outils, deux rôles

Le partage des responsabilités entre les deux outils découle directement du modèle en couches présenté en §16.1. **Terraform agit sur la couche infrastructure** : il fait exister les machines, les volumes, les réseaux et les ressources cloud. **Ansible agit sur la couche système et configuration** : une fois les machines disponibles, il installe MariaDB, dépose la configuration, gère le service et crée les comptes.

Ce n'est pas une règle absolue — chacun des deux outils est techniquement capable d'empiéter sur le domaine de l'autre — mais c'est l'usage idiomatique, et celui qui produit les architectures les plus lisibles et les plus maintenables.

## Terraform : provisionner l'infrastructure

**Terraform** est un outil de provisionnement **déclaratif**. On y décrit, dans un langage dédié appelé **HCL** (*HashiCorp Configuration Language*), l'ensemble des ressources d'infrastructure souhaitées ; Terraform se charge de les créer, de les modifier ou de les détruire pour faire correspondre la réalité à cette description.

Son fonctionnement repose sur quelques mécanismes caractéristiques. Terraform dialogue avec les API des fournisseurs au travers de **providers** (greffons), ce qui lui permet de gérer aussi bien des instances chez un fournisseur cloud que des ressources réseau ou des services managés. Il construit un **graphe de dépendances** entre les ressources, afin de les créer dans le bon ordre. Surtout, il maintient un **fichier d'état** (*state*) — déjà évoqué en §16.1 — qui lui sert de mémoire de ce qui existe réellement, et il propose un flux en deux temps : `plan`, qui présente les changements envisagés, puis `apply`, qui les exécute. Ce *plan* préalable est une sécurité précieuse pour une infrastructure de base de données, où une suppression accidentelle de volume serait catastrophique.

Pour MariaDB, le domaine de Terraform couvre typiquement les **instances de calcul**, les **volumes de stockage persistants** (élément critique pour un composant stateful), les **réseaux et sous-réseaux**, les **groupes de sécurité** ouvrant le port de la base aux seuls clients autorisés, et le cas échéant les **équilibreurs de charge** placés devant un cluster.

> **Note — Terraform et OpenTofu.** Depuis le changement de licence de Terraform (passage à la *Business Source License* en 2023), une bifurcation open source, **OpenTofu**, est maintenue sous une licence libre et reste largement compatible avec HCL. Les concepts présentés dans cette section s'appliquent indifféremment à l'un ou à l'autre.

## Ansible : configurer le système et MariaDB

**Ansible** est un outil de **gestion de configuration** (et d'orchestration légère) **sans agent**. On y décrit, dans des fichiers YAML appelés **playbooks**, une suite de tâches à appliquer à un ensemble de machines décrites dans un **inventaire**. Ansible se connecte à ces machines en **push**, généralement via SSH, et y exécute les tâches sans qu'aucun logiciel n'ait besoin d'y être préinstallé.

Son modèle est orienté **tâches** : chaque tâche fait appel à un **module** (installer un paquet, déposer un fichier, gérer un service, créer un utilisateur de base de données…). Ces modules sont conçus pour être **idempotents** : ils inspectent l'état réel de la machine et n'agissent que si une modification est nécessaire. Contrairement à Terraform, Ansible ne conserve pas de fichier d'état persistant ; il réévalue la situation à chaque exécution.

Pour MariaDB, le domaine d'Ansible couvre l'**installation des paquets** — y compris, en 12.3, le paquet `mariadb-server-galera` désormais distinct (§16.1, §14.2.5) —, la **gestion des fichiers de configuration** (`my.cnf` et la configuration modulaire de §11.1.3), le **service systemd**, la **création des utilisateurs, rôles et privilèges**, le déploiement des **certificats SSL/TLS**, et plus largement toute opération de configuration récurrente. Sa capacité d'**orchestration** — exécuter des tâches dans un ordre précis, en série ou sur un nœud unique d'un groupe — le rend par ailleurs précieux pour les opérations multi-nœuds, comme l'amorçage d'un cluster.

## Comparaison

Le tableau suivant résume les différences structurantes entre les deux outils.

| Critère | Terraform | Ansible |
|---|---|---|
| **Objectif principal** | Provisionnement d'infrastructure | Configuration et orchestration |
| **Paradigme** | Déclaratif | Procédural orienté tâches, à modules idempotents |
| **Langage** | HCL | YAML |
| **Modèle d'exécution** | `plan` puis `apply`, réconciliation par l'état | Exécution de tâches en push (SSH) |
| **Agent** | Sans agent (API des fournisseurs) | Sans agent (SSH) |
| **État persistant** | Oui (fichier *state*) | Non (inspection à chaque exécution) |
| **Domaine MariaDB** | Instances, volumes, réseau, sécurité, équilibreurs | Paquets, `my.cnf`, service, comptes, certificats, amorçage de cluster |
| **Point fort** | Création et cycle de vie des ressources | Convergence d'un système existant vers un état voulu |

La lecture essentielle de ce tableau tient en une phrase : Terraform excelle à **faire exister** une infrastructure et à en gérer le cycle de vie, tandis qu'Ansible excelle à **amener un système existant** dans l'état de configuration voulu. Ce sont deux moments distincts du déploiement, non deux solutions concurrentes au même problème.

## Des outils complémentaires, non concurrents

Il existe bien un recouvrement entre les deux. Terraform dispose de mécanismes d'exécution de commandes sur les machines provisionnées (*provisioners*), et Ansible possède des modules capables de créer des ressources cloud. On pourrait donc, en théorie, tout faire avec l'un ou avec l'autre.

En pratique, cet usage croisé est déconseillé. Les *provisioners* de Terraform sont eux-mêmes présentés par ses concepteurs comme un recours de dernière intention : ils s'intègrent mal au modèle d'état et de plan. Quant à la gestion d'infrastructure par Ansible, elle ne bénéficie pas du suivi d'état ni du graphe de dépendances qui font la force de Terraform. La règle de bon sens est donc d'**employer chaque outil sur la couche pour laquelle il est conçu** : provisionner avec Terraform, configurer avec Ansible.

## Le flux de déploiement combiné

C'est dans leur enchaînement que les deux outils déploient toute leur valeur. Le schéma canonique d'un déploiement MariaDB sur cloud se déroule en trois temps.

1. **Terraform provisionne l'infrastructure.** Il crée les instances (par exemple les trois nœuds d'un futur cluster Galera), attache les volumes persistants, met en place le réseau, les groupes de sécurité et, le cas échéant, l'équilibreur de charge. À l'issue de cette phase, des machines vierges existent et sont accessibles.
2. **La main passe de Terraform à Ansible.** Ce relais — point délicat de l'enchaînement — peut s'opérer de plusieurs manières. La plus courante et la plus recommandée est l'**inventaire dynamique** : Ansible construit automatiquement la liste des hôtes à configurer en interrogeant l'API du fournisseur, ou en lisant les *sorties* (*outputs*) de Terraform. Une alternative consiste à confier une amorce minimale à **cloud-init** au premier démarrage des instances. On évite en revanche, pour les raisons évoquées plus haut, de lancer Ansible directement comme *provisioner* de Terraform.
3. **Ansible configure MariaDB.** Sur les machines provisionnées, il installe les paquets, dépose la configuration, gère le service, crée les comptes et — pour un cluster — orchestre l'**amorçage** : démarrage du premier nœud, puis rattachement ordonné des suivants.

Cette séparation nette présente un avantage majeur : on peut détruire et recréer l'infrastructure (couche Terraform) indépendamment de sa configuration (couche Ansible), et inversement reconfigurer les serveurs sans toucher aux ressources sous-jacentes.

## Cas particuliers : n'employer qu'un seul outil

Le tandem n'est pas toujours nécessaire. Deux situations justifient l'usage d'un seul des deux outils.

Lorsqu'on adopte une approche d'**images immuables**, on construit en amont une image système contenant déjà MariaDB et sa configuration (à l'aide d'un outil de fabrication d'images), puis Terraform se contente d'**instancier cette image**, la configuration résiduelle étant assurée par une amorce légère. Le besoin d'Ansible au déploiement s'efface alors largement.

À l'inverse, sur un parc de **serveurs existants** — typiquement en environnement sur site (*on-premises*), sans provisionnement cloud à réaliser —, Terraform n'a rien à provisionner : **Ansible suffit** à configurer et à maintenir les machines déjà en place.

## Déploiements multi-nœuds en MariaDB 12.3

Provisionner plusieurs nœuds est trivial pour Terraform ; la subtilité d'un déploiement MariaDB en cluster réside dans la **coordination de leur configuration**. Pour un cluster **Galera**, l'ordre des opérations importe : il faut amorcer le premier nœud puis y rattacher les autres, ce qui relève de l'orchestration et où la capacité d'Ansible à séquencer les tâches sur un inventaire prend tout son sens (ce point rejoint le chapitre 14, §14.2). Pour une **réplication**, on provisionne un primaire et ses réplicas, puis on configure le binlog côté primaire et le rattachement côté réplicas (chapitre 13).

Le changement de *packaging* de la 12.3 a ici une incidence directe : puisque le support Galera fait désormais l'objet du paquet distinct `mariadb-server-galera`, l'étape d'installation pilotée par Ansible doit l'**inclure explicitement** pour un déploiement en cluster. Sur Kubernetes, cette même coordination multi-nœuds est prise en charge différemment, par un **opérateur** (§16.5), qui constitue une alternative déclarative au tandem Terraform/Ansible.

## Alternatives

Le couple Terraform/Ansible n'est pas la seule option. Côté gestion de configuration, **Puppet**, **Chef** et **Salt** répondent à des besoins comparables à ceux d'Ansible, avec des modèles parfois fondés sur un agent. Côté provisionnement, **Pulumi** permet de décrire l'infrastructure dans des langages de programmation généralistes, **CloudFormation** est l'outil natif d'un fournisseur particulier, et **OpenTofu** constitue la bifurcation libre de Terraform déjà mentionnée. Le tandem Terraform/Ansible demeure néanmoins l'association la plus répandue, et c'est elle que développent les sous-sections suivantes.

## Ce que couvrent les sous-sections

À partir de cette vue d'ensemble, les deux sous-sections entrent dans le concret. La première (§16.2.1) détaille les **playbooks Ansible** pour MariaDB : structure, modules pertinents, gestion de la configuration et des comptes. La seconde (§16.2.2) traite des **providers Terraform** pour le cloud : provisionnement des instances, des volumes et du réseau qui accueilleront la base.

---

↩️ [Section précédente : 16.1 — Infrastructure as Code](01-infrastructure-as-code.md)  
➡️ **Sous-section suivante :** [16.2.1 — Ansible : Playbooks MariaDB](02.1-ansible-playbooks.md)

⏭️ [Ansible : Playbooks MariaDB](/16-devops-automatisation/02.1-ansible-playbooks.md)

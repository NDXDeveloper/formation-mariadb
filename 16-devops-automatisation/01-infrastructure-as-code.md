🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.1 — Infrastructure as Code pour MariaDB

> Chapitre 16 — DevOps et Automatisation · Section de fondation conceptuelle.

---

## Qu'est-ce que l'Infrastructure as Code ?

L'**Infrastructure as Code** (IaC) désigne la pratique consistant à décrire et à gérer une infrastructure — serveurs, réseaux, stockage, configuration logicielle — au moyen de **fichiers de définition lisibles par la machine**, plutôt que par des opérations manuelles successives. Au lieu de se connecter à un serveur pour y taper des commandes, on écrit un fichier qui exprime l'**état désiré** du système, puis on laisse un outil se charger de le réaliser et de le maintenir.

L'idée centrale est de traiter l'infrastructure exactement comme du code applicatif : elle est versionnée dans Git, relue, testée, et déployée par une chaîne automatisée. Une infrastructure devient ainsi un artefact reproductible, et non plus le fruit d'une accumulation d'interventions dont personne ne se souvient précisément.

Pour une base de données comme MariaDB, cette approche présente une particularité que ce chapitre n'aura de cesse de rappeler : la base est un composant **stateful**, porteur d'un état durable. L'IaC s'y applique donc avec discernement, en distinguant soigneusement ce qui peut être recréé à volonté de ce qui doit être préservé.

## Du provisionnement manuel au code

L'administration traditionnelle d'une base de données repose souvent sur une succession d'actions manuelles : installer le paquet serveur, éditer `my.cnf`, créer les utilisateurs et leurs privilèges, générer les certificats, démarrer le service, ajuster quelques variables au fil des incidents. Cette approche, parfois appelée *ClickOps* lorsqu'elle passe par des interfaces graphiques, souffre de plusieurs faiblesses bien identifiées.

Le serveur ainsi obtenu est un **« serveur de compagnie »** (*snowflake server*) : unique, façonné à la main, impossible à reproduire à l'identique. La documentation qui était censée décrire sa configuration diverge inévitablement de la réalité, parce qu'une modification urgente faite à 3 h du matin n'est jamais reportée dans le wiki. Lorsqu'il faut monter un second environnement — préproduction, plan de reprise, nouvelle région — on recommence le travail à la main, avec le risque de petites différences invisibles qui se manifesteront au pire moment.

L'IaC répond à ces problèmes en faisant du **code la seule source de vérité**. Le fichier de définition n'est pas une documentation *à côté* de l'infrastructure : il *est* l'infrastructure. Toute modification passe par lui, ce qui rétablit l'alignement entre ce qui est décrit et ce qui tourne réellement.

## Les principes fondamentaux

Plusieurs principes structurent toute démarche d'IaC, indépendamment des outils employés.

**Déclaratif plutôt qu'impératif.** Une approche impérative décrit *comment* atteindre un résultat, étape par étape. Une approche déclarative décrit *l'état final souhaité* et laisse l'outil déterminer les opérations nécessaires pour y parvenir. La différence se voit clairement sur un exemple conceptuel :

```bash
# Approche impérative (suite d'instructions)
apt-get install -y mariadb-server  
sed -i 's/^bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/my.cnf  
systemctl enable mariadb  
systemctl start mariadb  
```

```yaml
# Approche déclarative (état désiré, pseudo-code)
mariadb:
  paquet: mariadb-server
  version: "12.3"
  configuration:
    bind-address: 0.0.0.0
  service:
    actif: true
    au-démarrage: true
```

Dans le second cas, on ne se préoccupe pas de savoir si le paquet est déjà installé ou si le service tourne déjà : on exprime ce que l'on veut, et l'outil converge vers cet état. C'est cette logique qui rend l'automatisation robuste et rejouable.

**Idempotence.** Une opération est *idempotente* lorsqu'on peut l'exécuter plusieurs fois sans changer le résultat au-delà de la première application réussie. Appliquer deux fois une définition déclarative ne réinstalle pas le paquet ni ne redémarre inutilement le service : si l'état réel correspond déjà à l'état désiré, rien ne se passe. L'idempotence est la propriété qui permet de rejouer le code en toute sécurité, par exemple pour réconcilier un serveur qui aurait été modifié à la main.

**Versionnement.** Parce que l'infrastructure est du code, elle vit dans un dépôt Git. On obtient gratuitement l'historique des changements, la possibilité de revenir à une version antérieure, la relecture par les pairs (*pull requests*) et la traçabilité complète : qui a changé quoi, quand et pourquoi. L'audit de l'infrastructure devient une simple lecture de l'historique Git.

**Reproductibilité.** À partir du même code, on doit pouvoir reconstruire un environnement strictement équivalent, autant de fois que nécessaire. C'est ce qui rend possibles les environnements jetables de test, la création rapide d'un environnement de préproduction fidèle à la production, ou encore la reconstruction d'une plateforme dans le cadre d'un plan de reprise d'activité.

## L'IaC appliquée à une base de données : un modèle en quatre couches

C'est ici que l'IaC pour une base de données se distingue de l'IaC applicative classique. Tout n'est pas « du code » de la même manière, et confondre les couches est l'une des erreurs les plus fréquentes. On peut distinguer quatre couches, chacune relevant d'un outillage propre.

La **couche infrastructure** regroupe les ressources sous-jacentes : machines (ou instances cloud), réseaux, groupes de sécurité, volumes de stockage. Elle se prête parfaitement au provisionnement déclaratif et constitue le terrain naturel d'un outil comme Terraform.

La **couche système et configuration MariaDB** couvre l'installation des paquets, la gestion des fichiers de configuration (`my.cnf`), le service systemd, ainsi que — avec quelques nuances — la création des utilisateurs, des rôles et des privilèges. Elle relève de la gestion de configuration, typiquement Ansible.

La **couche schéma** correspond à la structure des données : tables, colonnes, index, contraintes. Elle ne doit **pas** être gérée comme de l'IaC classique. Une modification de schéma sur des données existantes n'est généralement pas réversible et ne se « ré-applique » pas de façon idempotente comme un fichier de configuration. Elle relève d'outils de **migration versionnée** (Flyway, Liquibase, ou des modifications en ligne via gh-ost et pt-online-schema-change), traités plus loin dans le chapitre (§16.8).

La **couche données** enfin — les lignes elles-mêmes, qu'il s'agisse de données de référence (*seed*) ou de données de production — relève de la sauvegarde et de la restauration (chapitre 12) et de mécanismes d'amorçage, jamais d'un outil de provisionnement.

L'enseignement essentiel est le suivant : **l'IaC traite remarquablement bien les couches infrastructure et configuration, mais le schéma et les données exigent un outillage dédié**. Une plateforme MariaDB mature combine donc plusieurs disciplines complémentaires, là où une approche naïve tenterait — à tort — de tout faire avec un seul outil. La frontière la plus délicate est celle des utilisateurs et des privilèges : leur structure (qui a accès à quoi) peut souvent être gérée de façon déclarative par la gestion de configuration, mais certaines équipes choisissent de la versionner via les migrations, au plus près de l'évolution du schéma.

## Provisionnement et gestion de configuration

Deux familles d'outils, complémentaires, se partagent les couches que l'IaC sait automatiser.

Le **provisionnement** consiste à créer les ressources qui n'existent pas encore : instances de calcul, volumes, réseaux, équilibreurs de charge. C'est le domaine d'un outil comme **Terraform**, qui dialogue avec les API des fournisseurs cloud pour faire exister l'infrastructure.

La **gestion de configuration** intervient une fois les machines en place : installer MariaDB, déposer la bonne configuration, gérer le service, créer les comptes. C'est le domaine d'un outil comme **Ansible**, orienté vers la configuration d'un système existant.

Les deux approches se complètent : on provisionne d'abord, puis on configure. La transition entre les deux peut être assurée par un mécanisme d'amorçage (*cloud-init*), par l'appel d'Ansible depuis Terraform, ou par une chaîne d'orchestration. La section suivante (§16.2) entre dans le détail concret de ces deux outils.

## La dérive de configuration

Avec le temps, l'état réel d'un serveur tend à s'éloigner de l'état décrit : c'est la **dérive de configuration** (*configuration drift*). Elle naît des interventions manuelles non reportées dans le code — un paramètre ajusté en urgence, un correctif appliqué à la main, une variable modifiée le temps d'un diagnostic et jamais remise en état.

L'IaC combat cette dérive de deux manières. D'abord en faisant du code la référence unique : toute modification *devrait* passer par lui. Ensuite en permettant une **réconciliation périodique** : parce que les définitions sont idempotentes, on peut les rejouer régulièrement pour ramener le serveur à son état déclaré, neutralisant ainsi les modifications sauvages. Cette logique de réconciliation continue trouve son aboutissement dans le modèle GitOps (§16.12).

## La gestion de l'état dans les outils IaC

Certains outils de provisionnement, Terraform au premier rang, maintiennent un **fichier d'état** (*state*) qui décrit la correspondance entre les ressources déclarées dans le code et les ressources réellement existantes chez le fournisseur. Cet état est ce qui permet à l'outil de savoir s'il doit créer, modifier ou détruire une ressource lors de la prochaine application.

Ce fichier d'état appelle deux précautions importantes. Il peut contenir des **informations sensibles** (adresses, identifiants, parfois des secrets) et doit donc être stocké de façon sécurisée, dans un *backend distant* chiffré plutôt que sur un poste de travail. Il doit en outre être protégé par un **verrouillage** (*locking*) lorsque plusieurs personnes ou plusieurs exécutions automatisées travaillent en parallèle, faute de quoi deux applications simultanées pourraient corrompre la représentation de l'infrastructure.

## La gestion des secrets

Une base de données implique nécessairement des identifiants : mot de passe `root`, comptes applicatifs, clés de chiffrement, certificats. La règle absolue est de **ne jamais inscrire ces secrets en clair dans le code d'infrastructure**, puisque ce code est versionné dans Git et potentiellement partagé.

Les secrets sont confiés à des mécanismes dédiés : coffres-forts comme HashiCorp Vault, chiffrement intégré (`ansible-vault`), ou gestionnaires de secrets des fournisseurs cloud et de Kubernetes. Le code d'infrastructure ne contient alors qu'une **référence** vers le secret, résolue au moment de l'exécution. Ce sujet rejoint directement les chapitres consacrés à la sécurité (chapitre 10) et à la gestion des clés de chiffrement (§18.7.2).

## Spécificités MariaDB 12.3 LTS

Deux éléments propres à MariaDB méritent une attention particulière lorsqu'on décrit son infrastructure sous forme de code.

D'abord, depuis la série 12.x, le support **Galera** n'est plus une dépendance intégrée au paquet serveur : il fait l'objet d'un paquet distinct, `mariadb-server-galera`. En IaC, cela signifie que la déclaration d'un cluster Galera doit désormais **inclure explicitement** ce paquet, là où une version antérieure l'aurait embarqué automatiquement. Ce changement de *packaging* a des répercussions concrètes sur les définitions de provisionnement et sur la construction des images de conteneurs, détaillées plus loin (§16.3.1, §16.5.2).

Ensuite, la **configuration modulaire** de MariaDB — la possibilité de découper la configuration en plusieurs fichiers chargés depuis un répertoire d'inclusion (§11.1.3) — s'accorde particulièrement bien avec l'IaC : chaque préoccupation (réglages réseau, mémoire, réplication, journalisation) peut faire l'objet d'un fichier déposé indépendamment, plutôt que d'un unique `my.cnf` monolithique difficile à composer par morceaux.

## Bénéfices et limites

Les apports de l'IaC pour une plateforme MariaDB sont substantiels : reproductibilité d'un environnement à l'autre, rapidité de mise en place, auditabilité complète via Git, fiabilisation du plan de reprise d'activité (la plateforme peut être reconstruite à partir du code, à charge de restaurer les données par ailleurs), et collaboration facilitée entre développeurs, DBA et équipes d'exploitation.

Les limites tiennent essentiellement à la nature stateful de la base. L'« infrastructure immuable », principe selon lequel on remplace un composant plutôt que de le modifier, ne s'applique pas tel quel à une base de données : on ne jette pas un serveur portant des données. Pour la base, l'immuabilité se conçoit plutôt comme une **séparation entre un calcul remplaçable et un stockage persistant**, ou via des stratégies de bascule s'appuyant sur la réplication. De même, vouloir gérer le schéma et les données avec les outils d'IaC est un contresens : ces couches relèvent respectivement des migrations et des sauvegardes.

## Vers le GitOps

L'IaC constitue la fondation d'une démarche plus large. Lorsqu'on combine des définitions versionnées dans Git, une chaîne automatisée qui les applique, et une boucle de réconciliation continue qui ramène en permanence l'infrastructure vers son état déclaré, on obtient le modèle **GitOps**, où Git devient la source unique de vérité de l'état désiré de l'ensemble du système. Ce modèle, particulièrement adapté aux environnements Kubernetes, fait l'objet de la dernière section du chapitre (§16.12). Les sections intermédiaires en posent les briques : provisionnement et configuration (§16.2), conteneurisation (§16.3), orchestration et opérateurs (§16.4 à §16.6), puis intégration continue et migrations (§16.7 et §16.8).

---

↩️ [Retour à l'introduction du chapitre](README.md)  
➡️ **Section suivante :** [16.2 — Déploiement avec Ansible/Terraform](02-deploiement-ansible-terraform.md)

⏭️ [Déploiement avec Ansible/Terraform](/16-devops-automatisation/02-deploiement-ansible-terraform.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.3 — Conteneurisation avec Docker

> Chapitre 16 — DevOps et Automatisation · Vue d'ensemble de la section, avant l'étude détaillée des images, de Compose et de la persistance.

---

## Introduction

Les sections précédentes ont déployé MariaDB sur des machines — physiques ou virtuelles — provisionnées avec Terraform et configurées avec Ansible (§16.2). Cette section change d'unité de déploiement : la base ne s'exécute plus sur un serveur que l'on installe et configure, mais dans un **conteneur**, instancié à partir d'une **image** prête à l'emploi.

L'objet de cette page d'introduction n'est pas d'écrire des fichiers Dockerfile ou Compose — ce sera celui des sous-sections — mais de poser les notions et, surtout, de cerner la **tension de fond** entre la philosophie des conteneurs et la nature d'une base de données. Cette tension est le fil conducteur de toute la section.

## Du serveur à l'image : qu'est-ce qu'un conteneur ?

Un **conteneur** est un environnement d'exécution isolé, qui empaquette une application et ses dépendances tout en partageant le noyau du système hôte. Il se distingue d'une machine virtuelle par sa **légèreté** : là où une VM embarque un système d'exploitation complet, un conteneur n'embarque que ce dont l'application a besoin, ce qui le rend rapide à démarrer et économe en ressources.

Deux notions doivent être distinguées d'emblée. L'**image** est un modèle en lecture seule, construit par empilement de couches, qui contient les binaires de MariaDB et sa configuration par défaut. Le **conteneur** est une instance en cours d'exécution de cette image. Le point essentiel — et toute la difficulté pour une base de données — tient à ceci : l'image est **immuable**, et le conteneur est, par conception, **éphémère**. Tout ce qu'un conteneur écrit dans sa couche modifiable disparaît lorsqu'il est supprimé.

## La tension fondamentale : conteneurs sans état, base avec état

Les conteneurs ont été conçus pour des charges **sans état** (*stateless*), interchangeables et jetables — le fameux contraste entre le « bétail » que l'on remplace sans état d'âme et l'« animal de compagnie » que l'on soigne. Une base de données est précisément l'inverse : elle est le **gardien de l'état durable** du système. Y faire tourner MariaDB consiste donc à exécuter une charge profondément *stateful* dans une technologie pensée pour le *stateless*.

Cette contradiction n'interdit nullement de conteneuriser MariaDB — c'est une pratique courante et parfaitement maîtrisée — mais elle impose une discipline : **l'état doit être explicitement externalisé** hors du conteneur, dans un stockage persistant qui lui survit. C'est le sens de la troisième sous-section (§16.3.3), et le réflexe à acquérir avant toute autre considération : *un conteneur MariaDB sans volume persistant est une base de données qui perdra ses données*.

## Pourquoi conteneuriser MariaDB

Malgré cette tension, les bénéfices sont substantiels et expliquent l'adoption massive du modèle.

- **Parité des environnements** : la même image s'exécute en développement, en intégration et en production, ce qui élimine la classe d'incidents du « cela fonctionnait sur ma machine ».
- **Bases jetables et rapides** : un développeur obtient une instance MariaDB en quelques secondes, qu'il peut détruire et recréer à volonté.
- **Verrouillage de version par les étiquettes** : il suffit de référencer une étiquette d'image pour exécuter exactement la version voulue — ici **MariaDB 12.3**.
- **Isolation et portabilité** : plusieurs bases cohabitent sur un même hôte sans interférer, et l'image se déplace d'un environnement à l'autre sans réinstallation.
- **Onboarding simplifié** : la mise en route d'un nouvel arrivant ne suppose plus une procédure d'installation, mais l'exécution d'une image.
- **Fondation pour l'orchestration** : la conteneurisation est le préalable à Kubernetes et aux opérateurs, traités dans la suite du chapitre.

## Où les conteneurs conviennent — et où être prudent

Tous les usages ne se valent pas, et il est utile de distinguer les contextes.

En **développement** et en **intégration continue**, les conteneurs sont un choix sans réserve : leur nature éphémère, leur rapidité et leur reproductibilité y sont des atouts directs. Une base jetée à la fin d'une suite de tests, recréée à l'identique à la suivante, illustre parfaitement le modèle.

En **production**, la conteneurisation d'une base de données est aujourd'hui légitime et largement pratiquée, mais elle exige une **rigueur supplémentaire** : stockage persistant fiable, sauvegardes, réseau, limites de ressources, et de préférence un orchestrateur capable d'assurer la disponibilité — non un simple `docker run` isolé. Il faut noter, en toute objectivité, que certaines organisations choisissent délibérément de **conserver leurs bases de production sur des machines virtuelles ou des services managés**, jugeant le surcroît de complexité opérationnelle supérieur au bénéfice. Le choix dépend du contexte, de la maturité de l'équipe et des exigences de disponibilité ; cette section donne les moyens de le faire en connaissance de cause.

## Image, conteneur, volume : trois concepts à distinguer

La compréhension de la section repose sur la distinction nette de trois objets, que les sous-sections détaillent tour à tour. L'**image** est le modèle immuable d'où tout part (§16.3.1). Le **conteneur** est l'instance qui exécute MariaDB, éphémère par défaut. Le **volume** est le stockage persistant où résident réellement les données, et qui survit à la destruction du conteneur (§16.3.3). Garder ces trois notions séparées dans son esprit, c'est déjà éviter l'écueil principal du sujet.

## Les préoccupations opérationnelles d'une base conteneurisée

Exécuter MariaDB en conteneur soulève un ensemble de questions, dont chacune est approfondie ailleurs dans ce chapitre ou cette section.

La **persistance** est la première : les données doivent vivre dans un volume, jamais dans la couche modifiable du conteneur (§16.3.3). La **configuration** vient ensuite : l'image se paramètre par des variables d'environnement (le mot de passe `root`, par exemple) et par l'injection d'un fichier de configuration (§16.3.1). La **gestion des secrets** impose de ne jamais inscrire d'identifiants en clair dans une image ni de les transmettre de façon non sécurisée. Le **réseau** détermine comment la base est exposée et jointe par les applications. Les **limites de ressources** méritent une vigilance particulière : la taille du *buffer pool* InnoDB doit rester cohérente avec la limite mémoire allouée au conteneur, sous peine de voir ce dernier interrompu par le mécanisme de protection mémoire de l'hôte (ce point rejoint les chapitres 11 et 15). L'**initialisation** au premier démarrage et la **journalisation** — que les conteneurs dirigent par convention vers la sortie standard — complètent ce tableau.

## L'impact du packaging Galera en 12.3

La nouveauté de la série 12.x qui touche le plus directement la conteneurisation est le **changement de packaging de Galera**. Le support Galera relevant désormais d'un paquet distinct, `mariadb-server-galera` (§14.2.5), le **contenu des images officielles** et la **façon de construire un déploiement en cluster** s'en trouvent modifiés. C'est précisément l'objet de la première sous-section (§16.3.1), où ce point est traité en détail ; il suffit ici de retenir qu'il s'agit de la considération phare de la 12.3 pour qui conteneurise MariaDB.

## Du conteneur unique à l'orchestration

Docker est excellent pour exécuter **un nœud**, en développement comme pour des besoins simples. Mais la conteneurisation seule n'apporte ni la mise en cluster automatique, ni l'auto-réparation, ni la mise à l'échelle d'une base de production. **Docker Compose** (§16.3.2) occupe une position intermédiaire : il orchestre plusieurs conteneurs sur une même machine et se prête remarquablement aux environnements de développement multi-services, mais il **ne constitue pas une solution de haute disponibilité** pour la production. Ces besoins relèvent d'un orchestrateur — **Kubernetes** (§16.4) — et des **opérateurs** (§16.5), qui prennent en charge la nature stateful de la base à grande échelle. La présente section pose les fondations sur lesquelles ces outils s'appuieront.

## Ce que couvrent les sous-sections

À partir de cette vue d'ensemble, les trois sous-sections entrent dans le concret. La première (§16.3.1) traite des **images officielles MariaDB** et de l'impact du packaging Galera de la 12.3 sur leur usage. La deuxième (§16.3.2) présente **Docker Compose** comme outil d'environnement de développement. La troisième (§16.3.3) aborde le sujet décisif des **volumes et de la persistance**, qui résout la tension exposée plus haut.

---

↩️ [Section précédente : 16.2 — Déploiement avec Ansible/Terraform](02-deploiement-ansible-terraform.md)  
➡️ **Sous-section suivante :** [16.3.1 — Images officielles MariaDB](03.1-images-officielles.md)

⏭️ [Images officielles MariaDB (impact du packaging Galera 12.3)](/16-devops-automatisation/03.1-images-officielles.md)

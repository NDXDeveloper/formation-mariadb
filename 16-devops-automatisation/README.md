🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 16 — DevOps et Automatisation

> **Partie 8 : DevOps, Cloud et Automatisation**  
> Niveau : intermédiaire à avancé · Public principal : DevOps / Cloud, mais aussi DBA souhaitant moderniser ses pratiques.

---

## Introduction

Pendant longtemps, la base de données a été le composant que l'on automatisait en dernier — et souvent le moins. Tandis que les serveurs applicatifs devenaient interchangeables, scriptés et jetables, la base de données restait le « serveur de compagnie » que l'on installait à la main, que l'on configurait avec soin et que l'on hésitait à toucher une fois en production. Cette asymétrie a une cause profonde : **la base de données est *stateful***. Elle porte l'état durable du système, ce qui la rend bien plus délicate à traiter comme du code que des composants sans état.

Ce chapitre a précisément pour objet de combler ce fossé. Il montre comment appliquer à MariaDB les principes du DevOps — reproductibilité, automatisation, versionnement, observabilité — sans sacrifier les exigences propres aux données : durabilité, cohérence, et continuité de service. L'enjeu n'est pas de transformer la base en simple conteneur jetable, mais de rendre son cycle de vie *déclaratif, traçable et automatisable*, tout en respectant la nature critique de l'état qu'elle héberge.

Concrètement, vous y verrez comment décrire une infrastructure MariaDB sous forme de code, comment l'empaqueter et l'exécuter dans des conteneurs, comment l'orchestrer et l'exploiter à grande échelle sur Kubernetes, comment intégrer les évolutions de schéma dans une chaîne d'intégration et de livraison continues, et enfin comment superviser l'ensemble grâce à une véritable démarche d'observabilité. Le chapitre se conclut sur le modèle **GitOps**, qui pousse cette logique jusqu'au bout en faisant d'un dépôt Git la source unique de vérité de l'état désiré.

## Pourquoi le DevOps pour les bases de données ?

Automatiser une base de données ne se réduit pas à transposer les pratiques connues du monde applicatif. Trois caractéristiques imposent une approche spécifique.

D'abord, **la persistance de l'état** : on ne peut pas « recréer à neuf » une base de production sans en restaurer les données. Toute automatisation doit donc composer avec le stockage persistant, les sauvegardes et la récupération.

Ensuite, **les contraintes de disponibilité** : une montée de version, une modification de configuration ou un changement de schéma doivent s'effectuer sans interrompre le service, ce qui suppose des stratégies de déploiement progressif, de réplication et de bascule maîtrisées.

Enfin, **les évolutions de schéma irréversibles** : contrairement à un déploiement applicatif que l'on peut annuler en redéployant la version précédente, une migration de données est difficilement réversible. La gestion des migrations devient ainsi une discipline à part entière, au cœur de toute chaîne CI/CD orientée base de données.

Tenir compte de ces trois contraintes transforme la manière dont on outille MariaDB : c'est le fil conducteur de l'ensemble du chapitre.

## Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez en mesure de :

- comprendre les principes de l'**Infrastructure as Code** (IaC) et leur application à une base de données stateful ;
- automatiser le provisionnement et la configuration de MariaDB avec **Ansible** et **Terraform** ;
- conteneuriser MariaDB avec **Docker** et organiser un environnement de développement reproductible via **Docker Compose** ;
- déployer et exploiter MariaDB sur **Kubernetes**, en maîtrisant les objets adaptés au stockage persistant (StatefulSets, PersistentVolumes, StorageClasses) ;
- tirer parti du **mariadb-operator** (et connaître l'alternative **MariaDB Enterprise Operator**) pour gérer de façon déclarative des clusters Galera, la réplication et les sauvegardes ;
- intégrer les **migrations de schéma** dans une chaîne CI/CD à l'aide d'outils tels que Flyway, Liquibase, gh-ost et pt-online-schema-change ;
- mettre en place une supervision avec **Prometheus** et **Grafana**, et adopter une démarche d'**observabilité** couvrant logs, métriques et traces ;
- concevoir une stratégie d'**alerting** et de réponse aux incidents ;
- appliquer les principes du **GitOps** à la gestion d'une base de données.

## Plan du chapitre

Le chapitre suit une progression qui va de la description de l'infrastructure jusqu'à son exploitation continue.

**De l'infrastructure déclarative au provisionnement automatisé (16.1 – 16.2).** On pose d'abord les fondations de l'Infrastructure as Code appliquée à MariaDB, avant de la mettre en pratique avec deux outils complémentaires : Ansible, pour la configuration et l'orchestration des opérations, et Terraform, pour le provisionnement de ressources auprès des fournisseurs cloud.

**Conteneurisation et orchestration (16.3 – 16.6).** On passe ensuite à Docker — images officielles, Compose et persistance des volumes — puis à Kubernetes, où la nature stateful de la base impose des objets spécifiques. La gestion à grande échelle est confiée aux opérateurs : le mariadb-operator open source, qui prend en charge déploiement Galera, réplication et sauvegardes, ainsi que le MariaDB Enterprise Operator pour les environnements d'entreprise.

**Chaîne de livraison et migrations (16.7 – 16.8).** Le chapitre aborde ensuite l'intégration et la livraison continues appliquées aux bases de données, puis la gestion des migrations de schéma — la pierre angulaire d'un déploiement de base de données fiable — avec Flyway, Liquibase et les outils de modification de schéma en ligne.

**Observabilité et exploitation (16.9 – 16.11).** Vient la supervision avec Prometheus et Grafana (dont le `mysqld_exporter`), élargie à une démarche complète d'observabilité — logs, métriques, traces — et prolongée par l'alerting et la réponse aux incidents.

**Le modèle GitOps (16.12).** Enfin, le chapitre synthétise l'ensemble autour du GitOps, où l'état désiré de la base est décrit dans Git et réconcilié automatiquement.

## Spécificités MariaDB 12.3 LTS

Une évolution de la **série 12.x** a un impact direct sur les pratiques DevOps abordées ici : le support **Galera** n'est plus une dépendance intégrée au serveur, mais fait désormais l'objet d'un paquet distinct, `mariadb-server-galera`. Cette séparation du *packaging* change la façon dont on construit les images de conteneurs (la couche Galera n'étant plus systématiquement embarquée) et la façon dont les opérateurs provisionnent un cluster Galera. Ce point sera développé là où il importe — notamment pour les images officielles (§16.3.1) et le déploiement Galera via l'operator (§16.5.2) — et il est également couvert dans le chapitre consacré à la haute disponibilité.

Plus largement, la formation prend pour socle **MariaDB 12.3 LTS** (support jusqu'en juin 2029), ce qui guide les choix d'images, de versions et de configurations utilisées tout au long du chapitre.

## Prérequis

Ce chapitre s'appuie sur plusieurs notions vues précédemment. Il est recommandé de maîtriser :

- l'**administration et la configuration** de MariaDB (chapitre 11) : fichiers `my.cnf`, variables système, modes SQL — indispensables dès que l'on automatise la configuration ;
- la **sauvegarde et la restauration** (chapitre 12), socle de l'automatisation des backups ;
- la **réplication** (chapitre 13) et la **haute disponibilité** (chapitre 14, notamment Galera et MaxScale), nécessaires pour comprendre ce que les opérateurs orchestrent.

Côté écosystème, des bases en ligne de commande Linux, en conteneurs (Docker), en Kubernetes et en Git sont supposées acquises. Une familiarité avec les concepts d'intégration et de livraison continues facilitera la lecture des sections CI/CD et GitOps.

## Place dans le parcours

Ce chapitre constitue le cœur du **parcours DevOps / Cloud** (modules 1, 11-12, 14, **16**, 20). Il intéressera également les **DBA** désireux de moderniser leurs pratiques d'exploitation, ainsi que les architectes amenés à concevoir des plateformes de données automatisées et reproductibles.

---

➡️ **Section suivante :** [16.1 — Infrastructure as Code pour MariaDB](01-infrastructure-as-code.md)  
↩️ [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Infrastructure as Code pour MariaDB](/16-devops-automatisation/01-infrastructure-as-code.md)

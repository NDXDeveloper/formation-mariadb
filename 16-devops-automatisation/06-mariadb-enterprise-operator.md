🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.6 — MariaDB Enterprise Operator

> Chapitre 16 — DevOps et Automatisation · L'opérateur commercial de MariaDB, et comment choisir entre les deux éditions.

---

## Introduction

La section 16.5 a présenté le **mariadb-operator communautaire**, open source. Cette section traite de son pendant **commercial**, l'**opérateur Kubernetes Enterprise** de MariaDB, et fournit les éléments permettant de **choisir** entre les deux — comme annoncé en §16.5. On verra que l'essentiel de ce qui a été appris en 16.5 se transpose directement, l'édition Enterprise se distinguant par le **serveur qu'elle déploie**, par un **support commercial**, et par quelques **apports de production**.

## Un opérateur commercial, de même lignée

L'opérateur Enterprise n'est pas un produit sans rapport avec le communautaire : c'est sa **version commercialement supportée, dotée de fonctionnalités d'entreprise supplémentaires**. La documentation du projet communautaire renvoie d'ailleurs explicitement vers lui pour les usages d'entreprise. Les deux partagent le **même modèle de ressources** et la même architecture — opérateur, webhook, gestion de certificats —, de sorte que la connaissance des CRDs acquise en §16.5.1 reste valable, à quelques différences de **groupe d'API** et de **registre d'images** près.

## Ce qu'il déploie : MariaDB Enterprise Server

La différence la plus structurante tient au **serveur déployé**. Là où l'opérateur communautaire exécute MariaDB **communautaire**, l'opérateur Enterprise provisionne **MariaDB Enterprise Server** — une version **durcie** de MariaDB Community Server, conçue pour la production : soumise à une assurance qualité étendue, **configurée par défaut pour la production**, et enrichie de fonctionnalités d'entreprise (sécurité renforcée, conformité, sauvegarde, ainsi qu'une **compatibilité Oracle PL/SQL** dont la migration tire parti — chapitre 19). Il déploie également **MaxScale** pour le routage et la répartition lecture/écriture. On parle ainsi de gérer l'ensemble de la **plateforme MariaDB Enterprise** dans Kubernetes.

## Un modèle déclaratif identique

L'expérience d'utilisation est, pour l'essentiel, **identique** à celle de l'opérateur communautaire. On retrouve les **mêmes types de ressources** (dans le groupe `enterprise.mariadb.com` plutôt que `k8s.mariadb.com`) : `MariaDB`, `MaxScale`, `Backup`, `PhysicalBackup`, `Restore`, `PointInTimeRecovery`, `Database`, `User`, `Grant`, `Connection`, `SqlJob`, `ExternalMariaDB`. On retrouve les **mêmes topologies** de haute disponibilité — Galera (§16.5.2) et réplication primaire/réplicas (§16.5.3) —, le **même dispositif de sauvegarde et de restauration** (§16.5.4), et la **même intégration MaxScale**. Tout ce qui a été présenté en 16.5 s'applique donc, ce qui rend la transition entre les deux éditions naturelle.

## Les apports de l'édition Enterprise

Au-delà du serveur, plusieurs éléments distinguent l'édition Enterprise et justifient son positionnement de production.

Le premier est le **support commercial** : l'opérateur est fourni dans le cadre d'un **abonnement** à la plateforme MariaDB Enterprise, avec l'accompagnement et les garanties associés.

Le deuxième concerne les **sauvegardes physiques non bloquantes**. L'édition Enterprise enrichit les capacités de sauvegarde du communautaire en s'appuyant sur la fonctionnalité **`BACKUP STAGE`** de MariaDB Enterprise Server : les sauvegardes physiques s'effectuent **sans verrou de lecture prolongé ni interruption de service**, ce qui procure des sauvegardes cohérentes et de qualité production avec un impact minimal sur la charge (à rapprocher de §12.3.3).

Le troisième tient aux **certifications et plateformes** : l'opérateur est **certifié par Red Hat** (notamment pour OpenShift), prend en charge l'architecture **ppc64le** (systèmes IBM Power), et est supporté sur les **trois dernières versions de Kubernetes** parmi les distributions certifiées CNCF.

S'y ajoutent une **récupération à un instant donné (PITR) granulaire** pour les topologies primaire/réplicas (combinant sauvegardes physiques et archivage des journaux binaires) et l'**export natif de métriques Prometheus** pour le serveur comme pour MaxScale, exploitables dans Grafana avec alerting (§16.9).

## Installation et abonnement

L'installation suit le même schéma que le communautaire — deux chartes Helm, l'une pour les CRDs (`mariadb-enterprise-operator-crds`), l'autre pour l'opérateur (`mariadb-enterprise-operator`) — avec une différence notable : les **images proviennent d'un registre privé**, ce qui impose de configurer des **identifiants client** et de référencer un `imagePullSecrets`.

```yaml
spec:
  imagePullSecrets:
    - name: mariadb-enterprise-registry   # identifiants client pour le registre privé
```

L'opérateur Enterprise suit par ailleurs un versionnement **calendaire** avec des versions **LTS** (par exemple une série `25.10`), testées contre des versions précises de Kubernetes.

## La question des versions

Un point mérite l'attention, dans le prolongement de la remarque faite sur les services managés (§16.2.2). L'opérateur Enterprise déploie **MariaDB Enterprise Server**, dont la **lignée de versions** est **distincte** de celle de MariaDB communautaire et **suit cette dernière avec un décalage**. Concrètement, si l'on a besoin d'exécuter **précisément MariaDB communautaire 12.3**, c'est l'**opérateur communautaire** (§16.5) qui constitue la voie ; l'opérateur Enterprise s'inscrit, lui, dans le calendrier de versions d'Enterprise Server. Ce facteur de version entre donc dans la décision, au même titre que le besoin de support.

## Communautaire ou Enterprise ?

Le choix entre les deux éditions se pose en toute objectivité, et le tableau suivant en résume les axes.

| Critère | mariadb-operator (communautaire, §16.5) | MariaDB Enterprise Operator (§16.6) |
|---|---|---|
| **Modèle** | Open source, gratuit | Commercial (abonnement) |
| **Serveur déployé** | MariaDB communautaire | MariaDB Enterprise Server (durci, QA) |
| **Support** | Communautaire | Support commercial MariaDB |
| **Groupe d'API** | `k8s.mariadb.com` | `enterprise.mariadb.com` |
| **Images** | Registre public | Registre privé (identifiants client) |
| **Sauvegardes physiques** | `mariadb-backup` / VolumeSnapshot | + **non bloquantes** via `BACKUP STAGE` |
| **Certifications / plateformes** | — | Red Hat / OpenShift, ppc64le (IBM Power) |
| **CRDs, topologies, MaxScale** | Identiques | Identiques |

En synthèse, l'opérateur **communautaire** convient à de nombreux contextes : capable (Galera, réplication, sauvegardes, MaxScale), gratuit, mais **auto-supporté** et fondé sur le serveur communautaire. L'opérateur **Enterprise** s'adresse aux organisations qui privilégient un **support contractuel**, qui ont besoin des **fonctionnalités d'Enterprise Server** (sécurité, conformité, compatibilité Oracle), de **sauvegardes non bloquantes de production**, ou de **plateformes certifiées** (OpenShift, Power). Les facteurs décisifs sont donc le **besoin de support**, les **fonctionnalités serveur** requises, les **certifications de plateforme** et la **contrainte de version** — sans oublier le **budget**.

## Place dans le chapitre

Quelle que soit l'édition retenue, le raisonnement reste celui des sections précédentes. Les deux opérateurs incarnent le **patron de l'opérateur** présenté en §16.5, **s'appuient sur les primitives** de la section 16.4 (StatefulSets, volumes persistants, services), s'inscrivent dans une logique d'**Infrastructure as Code** (§16.1) et se prêtent au **GitOps** (§16.12). Le choix de l'édition relève de l'organisation et de ses contraintes ; les principes d'exploitation, eux, sont communs.

## Ce que couvre la section suivante

Après l'orchestration et les opérateurs, la section §16.7 aborde l'**intégration et la livraison continues (CI/CD)** appliquées aux bases de données.

---

↩️ [Section précédente : 16.5 — mariadb-operator pour Kubernetes](05-mariadb-operator.md)  
➡️ **Section suivante :** [16.7 — CI/CD pour bases de données](07-cicd-bases-donnees.md)

⏭️ [CI/CD pour bases de données](/16-devops-automatisation/07-cicd-bases-donnees.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.8 Sauvegarde cloud-native

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Les déploiements modernes de MariaDB s'exécutent de plus en plus dans le **cloud**, en **conteneurs** et sur **Kubernetes**. Dans ces environnements, l'approche traditionnelle — sauvegarder dans un répertoire local puis copier le fichier vers un stockage distant — s'intègre mal : les serveurs sont éphémères, le stockage est découplé du calcul, et l'orchestration est déclarative.

La **sauvegarde cloud-native** consiste à tirer parti des **capacités natives de la plateforme** plutôt qu'à plaquer des habitudes héritées du monde « machine physique ». Elle s'articule autour de deux piliers complémentaires, qui font l'objet des sous-sections suivantes : le **stockage objet** comme cible de sauvegarde ([12.8.1](08.1-s3-object-storage.md)), et les **snapshots** au niveau du stockage, notamment via les **VolumeSnapshots** de Kubernetes ([12.8.2](08.2-kubernetes-volumesnapshots.md)). Cette page présente la philosophie d'ensemble et le point d'attention majeur de ces approches : la **cohérence**.

---

## Qu'est-ce qu'une sauvegarde « cloud-native » ?

L'idée directrice est de cesser de considérer le cloud comme un simple « disque distant » pour exploiter ses **services dédiés**. Deux angles se dégagent, qui correspondent à deux questions distinctes :

- **Où** stocker les sauvegardes ? → le **stockage objet** (la *cible*) ;
- **Comment** réaliser la sauvegarde ? → les **snapshots** au niveau du stockage (le *mécanisme*).

Ces deux dimensions ne s'opposent pas : on peut combiner un mécanisme (un snapshot, ou une sauvegarde Mariabackup) avec une cible cloud (le stockage objet).

---

## Le stockage objet comme cible de sauvegarde

Les services de **stockage objet** — Amazon S3, MinIO, Google Cloud Storage, Azure Blob Storage — constituent une cible idéale pour les sauvegardes. Plusieurs propriétés expliquent leur succès dans ce rôle : une **durabilité** très élevée (réplication interne sur plusieurs équipements), un caractère **hors site par nature** (qui satisfait directement la règle 3-2-1 vue en [section 12.1](01-strategies-sauvegarde.md)), une **capacité quasi illimitée**, un **coût maîtrisé** grâce aux classes de stockage (chaud, froid, archive), et un accès **par API**, totalement **découplé** du serveur de base de données.

Côté MariaDB, l'intégration est directe : **Mariabackup** sait **diffuser** son flux de sauvegarde vers un stockage objet ([12.3.1](03.1-full-backup.md) a introduit le *streaming*), les dumps logiques peuvent y être téléversés, et le **moteur S3** (chapitre [7.6](../07-moteurs-de-stockage/06-moteur-s3.md)) permet même d'archiver directement des données froides sur S3 ou MinIO. Le détail de ces approches est traité en [section 12.8.1](08.1-s3-object-storage.md).

---

## Les snapshots au niveau du stockage

La seconde approche déplace la sauvegarde vers la **couche de stockage** elle-même. Un **snapshot** est une copie instantanée d'un volume à un point précis dans le temps, réalisée par le système de stockage (snapshots de disques cloud, ou **VolumeSnapshots** via l'interface CSI sur Kubernetes).

Son atout est la **rapidité** : grâce à des techniques de copie sur écriture (*copy-on-write*) au niveau du stockage, un snapshot est quasi instantané et **indépendant de la taille** de la base — un avantage décisif sur les très gros volumes, là où une sauvegarde logique serait interminable.

Mais cet atout s'accompagne d'un **point d'attention déterminant**, abordé ci-dessous : la cohérence.

---

## Cohérence : le point d'attention majeur

Quel que soit le mécanisme retenu, une sauvegarde n'a de valeur que si elle est **cohérente**. Les outils dédiés — `mariadb-dump`, `mydumper`, Mariabackup — gèrent nativement cette cohérence (instantané transactionnel, `BACKUP STAGE`… voir [12.2](02-sauvegarde-logique.md) et [12.3](03-sauvegarde-physique-mariabackup.md)). Les **snapshots de volume bruts**, eux, ne l'offrent pas automatiquement.

Il faut ici distinguer deux niveaux de cohérence :

- un snapshot pris **sans coordination** avec le serveur capture un état dit **« cohérent en cas de crash »** (*crash-consistent*) : c'est l'équivalent d'une coupure de courant. InnoDB sait s'en remettre par sa **reprise après incident**, mais l'opération comporte un risque et n'inclut pas les tables non transactionnelles dans un état fiable ;
- un snapshot **« cohérent au niveau applicatif »** (*application-consistent*) suppose de **figer brièvement** la base avant la capture — en bloquant les commits (via `BACKUP STAGE BLOCK_COMMIT`, ou l'ancien `FLUSH TABLES WITH READ LOCK`, voir [12.3.3](03.3-backup-stage.md)) — puis de relâcher aussitôt après.

C'est le **principe central** des approches par snapshot : pour une sauvegarde réellement fiable, le snapshot doit être **coordonné** avec MariaDB. Ce point est développé pour Kubernetes en [section 12.8.2](08.2-kubernetes-volumesnapshots.md).

---

## Le contexte Kubernetes

Kubernetes est devenu un terrain privilégié pour MariaDB, déployée via des **StatefulSets** et des **PersistentVolumes** (chapitre [16.4](../16-devops-automatisation/04-orchestration-kubernetes.md)). La sauvegarde cloud-native y prend deux formes : les **VolumeSnapshots** (au niveau du stockage, via CSI), et l'**opérateur MariaDB** (`mariadb-operator`), qui propose des **sauvegardes planifiées déclaratives** (chapitre [16.5](../16-devops-automatisation/05-mariadb-operator.md)). L'opérateur peut orchestrer des sauvegardes vers un stockage objet comme via des snapshots, en intégrant la coordination nécessaire à la cohérence.

---

## Cloud-native et stratégie d'ensemble

Les approches cloud-native ne **remplacent pas** les principes de ce chapitre : elles les **mettent en œuvre** dans une infrastructure moderne. La règle 3-2-1, l'automatisation ([12.6](06-automatisation-sauvegardes.md)) et surtout les **tests de restauration** ([12.7](07-tests-restauration-pra.md)) restent tout aussi indispensables — un snapshot non testé n'est pas plus fiable qu'une sauvegarde classique non éprouvée.

Notons enfin que les **bases de données managées** (services cloud type *Database-as-a-Service*) intègrent souvent leurs propres mécanismes de sauvegarde automatique. Même dans ce cas, comprendre les mécanismes sous-jacents — cohérence, PITR, rétention — demeure essentiel pour valider que la protection répond réellement aux objectifs RPO/RTO.

---

## Structure de la section

- [**12.8.1 — S3 et Object Storage**](08.1-s3-object-storage.md) : utiliser le stockage objet comme cible de sauvegarde (streaming Mariabackup, dumps, moteur S3).
- [**12.8.2 — Kubernetes VolumeSnapshots**](08.2-kubernetes-volumesnapshots.md) : réaliser des sauvegardes par snapshot de volume dans un cluster Kubernetes, et garantir leur cohérence.

---

## À retenir

- La **sauvegarde cloud-native** exploite les capacités natives de la plateforme (stockage objet, snapshots, primitives Kubernetes) plutôt que de transposer les méthodes héritées.
- Deux angles complémentaires : le **stockage objet** comme **cible** (durable, hors site, satisfait la règle 3-2-1) et les **snapshots** comme **mécanisme** (rapides, indépendants de la taille).
- Mariabackup sait **diffuser vers le stockage objet** ; le **moteur S3** archive directement les données froides.
- **Point central des snapshots** : un snapshot brut n'est que *crash-consistent* ; pour une cohérence applicative, il faut **coordonner** la capture avec MariaDB (`BACKUP STAGE BLOCK_COMMIT`).
- Sur **Kubernetes**, la sauvegarde s'appuie sur les **VolumeSnapshots** et l'**opérateur MariaDB** (sauvegardes planifiées déclaratives).
- Les approches cloud-native **n'exemptent pas** des principes fondamentaux : 3-2-1, automatisation et **tests de restauration** restent obligatoires.

---

⏮️ [12.7 — Tests de restauration et plan de reprise (PRA)](07-tests-restauration-pra.md) · ⏭️ [12.8.1 — S3 et Object Storage](08.1-s3-object-storage.md)

⏭️ [S3 et Object Storage](/12-sauvegarde-restauration/08.1-s3-object-storage.md)

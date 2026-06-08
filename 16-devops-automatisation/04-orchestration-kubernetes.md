🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.4 — Orchestration avec Kubernetes

> Chapitre 16 — DevOps et Automatisation · Vue d'ensemble de la section, avant l'étude des StatefulSets et du stockage persistant.

---

## Introduction

La section précédente s'arrêtait à un constat : Docker et Compose conviennent au développement et à un hôte unique, mais ne procurent ni tolérance aux pannes multi-nœuds, ni auto-réparation, ni mise à l'échelle (§16.3). C'est précisément le domaine de **Kubernetes**, l'orchestrateur qui exécute des conteneurs sur un **ensemble de machines**. La transition de §16.3.3 l'annonçait : la préoccupation de persistance ne disparaît pas, elle change d'abstraction.

Cette page d'introduction pose les notions et le **défi central** de l'exécution d'une base de données sur Kubernetes. Elle reste au niveau des concepts : les **primitives** (StatefulSets, stockage persistant) font l'objet des deux sous-sections, et les **opérateurs**, qui automatisent l'ensemble, des sections 16.5 et 16.6. Comprendre cette progression — des briques de base vers les opérateurs — est la clé de toute la fin du chapitre.

## Qu'est-ce que Kubernetes

Kubernetes est un **orchestrateur de conteneurs** : il planifie l'exécution de conteneurs, regroupés en **pods**, à travers un cluster de machines (les *nœuds*). Son fonctionnement est **déclaratif** : on décrit l'état désiré dans des manifestes, et le plan de contrôle s'emploie en permanence à faire **converger l'état réel vers cet état désiré**, au moyen d'une boucle de réconciliation.

De ce modèle découlent ses garanties caractéristiques : l'**auto-réparation** (un pod défaillant est relancé, un nœud perdu voit ses pods reprogrammés ailleurs), la **mise à l'échelle** déclarative, les **mises à jour progressives**, et la **découverte de services** avec répartition de charge. Ce sont ces garanties, absentes de Compose, qui motivent le passage à l'orchestration.

## Le défi central : Kubernetes a été pensé pour le stateless

On retrouve ici, à un niveau supérieur, la tension exposée pour Docker (§16.3). Les abstractions par défaut de Kubernetes — les **Deployments** et les ReplicaSets — supposent des pods **interchangeables et sans état**, librement remplacés, reprogrammés et multipliés. Une base de données contredit chacune de ces hypothèses : elle exige une **identité stable**, un **stockage stable**, des **opérations ordonnées** et un traitement soigneux de son état.

Kubernetes a donc introduit des primitives **spécifiques aux charges stateful** — les StatefulSets et les volumes persistants, objets des sous-sections — pour rendre l'exercice possible. Mais cet exercice demeure suffisamment **complexe** pour qu'une couche supplémentaire ait vu le jour : les **opérateurs** (§16.5), qui encapsulent le savoir-faire opérationnel d'exploitation d'une base. La présente section enseigne les fondations ; les sections suivantes en montrent l'automatisation.

## Pourquoi orchestrer MariaDB avec Kubernetes

Par rapport à Docker et Compose, les apports sont considérables.

- **Tolérance aux pannes multi-nœuds** : la défaillance d'une machine n'emporte plus toute la pile ; les pods sont reprogrammés sur d'autres nœuds.
- **Auto-réparation** : un pod en échec est automatiquement relancé.
- **Mise à l'échelle et mises à jour progressives** déclaratives.
- **Découverte de services et réseau stable**, indispensables à la mise en cluster.
- **Plateforme unifiée** : la base cohabite avec les applications, sous le même outillage et la même démarche GitOps (§16.12).

Il faut toutefois énoncer la contrepartie en toute objectivité : Kubernetes est **complexe**, et pour une base de données, ses bénéfices doivent être mis en balance avec la charge d'exploitation qu'il introduit. Nombre d'organisations choisissent, en toute légitimité, d'exécuter leurs **applications** sur Kubernetes tout en confiant leur **base** à un service managé (§16.2.2) auquel elles se connectent. Le présent chapitre donne les moyens de trancher en connaissance de cause, et — lorsque le choix se porte sur Kubernetes — de le faire avec les bons outils.

## Les objets Kubernetes utiles à MariaDB

Avant d'entrer dans le détail, il est utile de disposer d'une carte des objets mobilisés. Chacun est approfondi dans une sous-section ou dans la section consacrée aux opérateurs.

| Objet Kubernetes | Rôle pour MariaDB |
|---|---|
| **Pod** | Unité d'exécution — le conteneur MariaDB y tourne |
| **StatefulSet** | Gère des répliques à identité et stockage stables (§16.4.1) |
| **PersistentVolume / PVC / StorageClass** | Abstraction et provisionnement du stockage persistant (§16.4.2) |
| **Service (sans tête)** | Identité réseau stable et résolution DNS, par pod |
| **ConfigMap** | Configuration (`my.cnf` via `conf.d`) |
| **Secret** | Identifiants (mot de passe `root`, comptes applicatifs) |

## Pourquoi pas un Deployment, mais un StatefulSet

Le réflexe le plus courant sur Kubernetes — déployer une application via un **Deployment** — est **inadapté** à une base de données, et il importe de comprendre pourquoi. Un Deployment traite ses pods comme **fongibles** : ils reçoivent des noms aléatoires, ne disposent pas d'un stockage propre et stable, et sont créés ou détruits sans ordre garanti. Pour une base, cela signifierait des répliques sans identité fixe, susceptibles de partager ou de perdre leur stockage — un non-sens.

Le **StatefulSet** répond exactement à ce besoin : il confère à chaque réplique un **nom stable et ordonné** (`mariadb-0`, `mariadb-1`…), un **stockage qui lui est propre et persiste**, et des opérations de déploiement et de mise à l'échelle **ordonnées**. C'est la primitive centrale pour MariaDB sur Kubernetes, détaillée en §16.4.1.

## Le stockage, point névralgique

S'il fallait désigner la difficulté majeure d'une charge stateful sur Kubernetes, ce serait le **stockage**. La réponse repose sur un triptyque : le **PersistentVolume** représente une ressource de stockage du cluster, le **PersistentVolumeClaim** en exprime la demande, et la **StorageClass** permet le provisionnement **dynamique** de ce stockage. Au sein d'un StatefulSet, chaque réplique obtient ainsi **son propre volume**. Ce mécanisme prolonge directement ce qui a été vu pour Docker (§16.3.3) — externaliser l'état hors d'un conteneur éphémère — en l'adaptant à l'échelle d'un cluster. Il fait l'objet de la sous-section §16.4.2.

## Configuration, secrets et réseau

Trois autres objets complètent l'édifice. La **ConfigMap** porte la configuration de MariaDB, montée sous forme de fichiers dans le répertoire d'inclusion (`conf.d`), dans le prolongement de la configuration modulaire (§11.1.3). Le **Secret** porte les identifiants, consommés via la convention `_FILE` de l'image (§16.3.1) ou montés comme fichiers. Enfin, le **Service sans tête** (*headless*) attribue à chaque pod d'un StatefulSet un **nom DNS stable** — condition indispensable à la mise en cluster, où chaque nœud Galera ou chaque réplica doit pouvoir joindre ses pairs à une adresse prévisible.

## Des primitives aux opérateurs

Voici le point qui structure toute la suite du chapitre. Assembler un StatefulSet, des volumes persistants, un Service, une ConfigMap et un Secret permet d'obtenir des **pods MariaDB qui s'exécutent**. Mais cet assemblage **ne fournit pas** pour autant les opérations qui font le quotidien d'une base en production : l'**amorçage automatique d'un cluster**, la **bascule** en cas de panne, la **configuration de la réplication**, les **sauvegardes**, les **montées de version**. Ces opérations dites « du jour 2 » supposent d'**encoder un savoir-faire opérationnel** — et c'est exactement la fonction d'un **opérateur** (§16.5).

La conclusion qui en découle oriente la lecture des sections suivantes : il est précieux de **comprendre les primitives**, car les opérateurs sont bâtis sur elles et l'on doit savoir ce qu'ils orchestrent ; mais en **production**, on privilégiera un opérateur plutôt que la gestion manuelle de manifestes bruts pour une base de données. La présente section et ses sous-sections donnent les fondations ; les sections 16.5 et 16.6 montrent comment s'en affranchir au profit d'une approche déclarative de plus haut niveau.

## Spécificités MariaDB 12.3 LTS

La nature stateful étant l'enjeu, l'essentiel des primitives est **indépendant de la version**. Le seul prolongement notable de la 12.3 concerne, là encore, le **packaging de Galera** : l'image employée dans un StatefulSet doit **inclure le paquet `mariadb-server-galera`** pour un déploiement en cluster (§16.3.1, §14.2.5) — contrainte que les opérateurs (§16.5) prennent en charge pour l'utilisateur. Pour le reste, stockage et orchestration s'appliquent à la 12.3 comme aux versions précédentes.

## Ce que couvrent les sous-sections

À partir de cette vue d'ensemble, les deux sous-sections approfondissent les primitives essentielles. La première (§16.4.1) détaille les **StatefulSets** : identité stable, stockage par réplique, opérations ordonnées. La seconde (§16.4.2) traite des **PersistentVolumes et des StorageClasses** : comment le stockage est provisionné et rattaché aux répliques. La section suivante (§16.5) montrera ensuite comment un **opérateur** automatise l'ensemble.

---

↩️ [Section précédente : 16.3 — Conteneurisation avec Docker](03-conteneurisation-docker.md)  
➡️ **Sous-section suivante :** [16.4.1 — StatefulSets pour MariaDB](04.1-statefulsets.md)

⏭️ [StatefulSets pour MariaDB](/16-devops-automatisation/04.1-statefulsets.md)

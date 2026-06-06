🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12. Sauvegarde et Restauration

> **Partie 5 — Sécurité et Administration (DBA)**  
> Version de référence : **MariaDB 12.3 LTS** · Niveau : Administrateur / DBA

---

## Introduction

Pour la plupart des organisations, les données stockées dans une base sont l'actif le plus précieux et, surtout, le moins remplaçable du système d'information. Le code applicatif peut être réécrit, un serveur peut être réinstallé, une configuration peut être reconstruite — mais des données de production perdues sont fréquemment irrécupérables. C'est ce qui fait de la sauvegarde, non pas une tâche d'administration parmi d'autres, mais la dernière ligne de défense d'un système face à un éventail de scénarios de défaillance.

Ces scénarios sont nombreux et de natures très différentes : panne matérielle (disque défaillant, contrôleur RAID corrompu), bug logiciel, erreur humaine (un `DROP TABLE` exécuté sur la production, un `UPDATE` lancé sans clause `WHERE`), acte malveillant (rançongiciel, sabotage interne), corruption silencieuse de fichiers, ou encore sinistre majeur affectant un centre de données entier ou une région cloud. Aucune réplication ni aucune architecture haute disponibilité ne protège contre toutes ces situations : une suppression accidentelle ou une corruption logique se propage instantanément vers les répliques. La sauvegarde joue donc un rôle que la redondance ne peut pas assurer.

Il faut enfin insister sur un point que le titre de ce chapitre rend explicite en associant les deux termes : ce qui compte n'est pas de *sauvegarder*, mais de pouvoir *restaurer*. Une sauvegarde dont la restauration n'a jamais été testée n'est pas une sauvegarde, c'est une espérance. La capacité à remettre un système en service dans un délai maîtrisé, avec une perte de données acceptable, est l'unique critère qui valide une stratégie.

---

## Deux métriques qui structurent toute stratégie

Avant de choisir des outils ou des fréquences, une stratégie de sauvegarde se définit à partir de deux indicateurs métier qui guideront ensuite toutes les décisions techniques :

- Le **RPO** (*Recovery Point Objective*) répond à la question : « quelle quantité de données peut-on accepter de perdre ? », exprimée en temps. Un RPO de 24 heures autorise la perte d'une journée de transactions ; un RPO de quelques secondes impose une approche en continu, typiquement adossée aux *binary logs*.
- Le **RTO** (*Recovery Time Objective*) répond à la question : « combien de temps peut-on rester indisponible ? ». Il conditionne le choix entre une restauration logique (souvent plus lente) et une restauration physique, ainsi que le niveau d'automatisation requis.

Ces deux exigences, presque toujours en tension avec le coût et la complexité, déterminent l'ensemble des arbitrages présentés dans ce chapitre.

---

## Les grands axes du chapitre

Sauvegarder une base MariaDB revient à se positionner sur plusieurs dimensions complémentaires, que le chapitre aborde successivement :

La **portée et la fréquence** d'abord, avec la distinction entre sauvegardes complète, incrémentale et différentielle, qui détermine le compromis entre espace de stockage, durée de sauvegarde et complexité de restauration.

La **méthode** ensuite, qui oppose la sauvegarde *logique* (un export des données et de leur structure sous forme d'instructions SQL, portable et lisible) à la sauvegarde *physique* (une copie des fichiers du moteur, plus rapide et mieux adaptée aux gros volumes).

La **continuité** enfin, assurée par les *binary logs*, qui permettent de rejouer les transactions survenues après la dernière sauvegarde et constituent le socle de la restauration à un instant précis (*Point-in-Time Recovery*).

À ces fondations s'ajoutent les dimensions opérationnelles indispensables en production : l'**automatisation** des sauvegardes, la **vérification régulière des restaurations** et la construction d'un plan de reprise d'activité (PRA), puis l'ouverture vers les approches **cloud-native** (stockage objet S3, *snapshots* de volumes Kubernetes) qui prennent une place croissante dans les architectures modernes.

---

## Le paysage des outils MariaDB

MariaDB propose un écosystème d'outils dédiés, chacun couvrant un besoin particulier. Ce chapitre les présentera en détail ; en voici la vue d'ensemble :

| Outil | Type | Usage principal |
|-------|------|-----------------|
| `mariadb-dump` (`mysqldump`) | Logique | Export SQL portable, idéal pour volumes modérés et migrations |
| `mydumper` / `myloader` | Logique | Export et import logiques **parallélisés**, pour de meilleures performances |
| **Mariabackup** | Physique | Sauvegarde « à chaud » des fichiers, complète ou incrémentale, avec support `BACKUP STAGE` |
| *Binary logs* | Journalisation | Base de la restauration à un instant précis (PITR) |
| Stockage objet / `VolumeSnapshots` | Cloud-native | Externalisation des sauvegardes (S3, Kubernetes) |

L'outillage logique a récemment gagné en souplesse — `mariadb-dump` prend en charge les *wildcards* (motifs avec jokers) via l'option `-L` depuis la **12.1** (et donc dans la 12.3 LTS) —, tandis que les approches physiques et cloud restent les piliers des environnements à fort volume ou conteneurisés. Le détail de ces fonctionnalités est traité dans les sections concernées.

---

## Un principe directeur : la règle 3-2-1

Au-delà des outils, une bonne pratique fait consensus et sert de fil conducteur à ce chapitre : la règle **3-2-1**. Elle recommande de conserver au moins **3** copies des données, sur **2** supports de nature différente, dont **1** copie hors site. Appliquée à MariaDB, cette règle invite à ne jamais se reposer sur une sauvegarde unique stockée sur le même serveur que la base de production, et à toujours prévoir une copie externalisée — un objectif que les approches cloud-native facilitent grandement.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- distinguer les stratégies complète, incrémentale et différentielle, et choisir celle qui correspond à vos contraintes de RPO/RTO ;
- arbitrer entre sauvegarde logique et sauvegarde physique selon le volume et le type de base ;
- mettre en œuvre `mariadb-dump`, `mydumper`/`myloader` et Mariabackup ;
- exploiter les *binary logs* pour réaliser une restauration à un instant précis (PITR) ;
- automatiser et superviser vos sauvegardes ;
- valider votre dispositif par des tests de restauration et formaliser un plan de reprise d'activité (PRA) ;
- concevoir une stratégie de sauvegarde adaptée aux environnements cloud et Kubernetes.

---

## Prérequis

Pour tirer pleinement parti de ce chapitre, il est recommandé d'avoir assimilé :

- les **transactions et les propriétés ACID** (chapitre 6), pour comprendre la notion de cohérence d'une sauvegarde ;
- le fonctionnement des **moteurs de stockage**, en particulier **InnoDB** (chapitre 7) ;
- les bases de l'**administration**, notamment les *binary logs* abordés au chapitre 11 (section 11.5) ;
- une aisance minimale avec la **ligne de commande Linux** et la planification de tâches (`cron`, *systemd timers*).

---

## Structure du chapitre

- 12.1 — [Stratégies de sauvegarde : Full, Incrémentale, Différentielle](01-strategies-sauvegarde.md)
- 12.2 — [Sauvegarde logique](02-sauvegarde-logique.md)
- 12.3 — [Sauvegarde physique (Mariabackup)](03-sauvegarde-physique-mariabackup.md)
- 12.4 — [Sauvegarde incrémentale avec binary logs](04-sauvegarde-incrementale-binlog.md)
- 12.5 — [Restauration](05-restauration.md)
- 12.6 — [Automatisation des sauvegardes](06-automatisation-sauvegardes.md)
- 12.7 — [Tests de restauration et plan de reprise (PRA)](07-tests-restauration-pra.md)
- 12.8 — [Sauvegarde cloud-native](08-sauvegarde-cloud-native.md)

---

> **À retenir avant de commencer.** Une stratégie de sauvegarde ne se mesure pas à la régularité de ses sauvegardes, mais à sa capacité prouvée à restaurer les données dans les délais et avec la perte acceptable définis par l'organisation. Gardez cette idée en tête tout au long du chapitre : chaque technique présentée n'a de valeur qu'au regard de la restauration qu'elle rend possible.

---

⏮️ [Partie 5 — Sécurité et Administration](../partie-05-securite-administration.md) · ⏭️ [12.1 — Stratégies de sauvegarde](01-strategies-sauvegarde.md)

⏭️ [Stratégies de sauvegarde : Full, Incrémentale, Différentielle](/12-sauvegarde-restauration/01-strategies-sauvegarde.md)

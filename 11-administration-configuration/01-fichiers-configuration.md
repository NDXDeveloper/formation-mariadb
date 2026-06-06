🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1 — Fichiers de configuration

> Section d'ouverture du chapitre 11. Elle pose le cadre avant d'entrer dans le détail de la structure des fichiers, de leur ordre de lecture et de la configuration modulaire.

---

## Introduction

Un serveur MariaDB se comporte de la manière dont on le configure. Taille des caches, nombre de connexions autorisées, emplacement des journaux, niveau de rigueur du moteur SQL : tous ces réglages doivent être définis quelque part, et surtout être conservés d'un redémarrage à l'autre. C'est précisément le rôle des **fichiers de configuration**.

Ces fichiers constituent la source de vérité statique du serveur : ce que MariaDB lit au démarrage pour savoir comment fonctionner. Comprendre où ils se trouvent, comment ils s'articulent et dans quel ordre ils sont lus est la toute première compétence d'administration, car c'est le point d'entrée de presque tous les réglages que nous verrons dans la suite du chapitre.

---

## Pourquoi des fichiers de configuration ?

On pourrait imaginer passer toutes les options au serveur sur la ligne de commande à chaque lancement. En pratique, ce n'est ni tenable ni souhaitable. Les fichiers de configuration répondent à trois besoins fondamentaux.

D'abord, la **persistance**. Une option écrite dans un fichier survit aux redémarrages du serveur, contrairement à un réglage saisi à la volée. C'est la garantie qu'une instance redémarrée — après une mise à jour, un incident ou une simple maintenance — repartira avec la configuration attendue.

Ensuite, la **reproductibilité**. Un fichier de configuration est un artefact texte, versionnable et copiable. Il peut être placé sous gestion de version, déployé par un outil d'*Infrastructure as Code* (Ansible, Terraform — voir le chapitre 16) et appliqué à l'identique sur plusieurs serveurs. C'est la condition d'une infrastructure homogène et auditable.

Enfin, la **lisibilité**. Regrouper les réglages dans des fichiers structurés, commentés et organisés en sections rend la configuration compréhensible pour toute l'équipe, là où une longue ligne de commande deviendrait vite illisible et fragile.

---

## Configuration statique ou dynamique : deux mondes complémentaires

Il est essentiel, dès le départ, de distinguer deux manières d'agir sur le comportement de MariaDB.

La **configuration par fichier** est appliquée au démarrage du serveur. C'est le sujet de cette section 11.1. Elle est persistante et concerne aussi bien des paramètres modifiables à chaud que des paramètres dits *statiques*, qui ne peuvent être changés qu'ici, fichier en main, suivi d'un redémarrage.

La **configuration dynamique**, à l'inverse, agit sur le serveur en cours d'exécution sans le redémarrer, via les variables système et de session. C'est l'objet de la section 11.2. Elle est précieuse pour ajuster un réglage en production, mais une modification dynamique seule n'est pas persistante : si l'instance redémarre, elle est perdue tant qu'elle n'a pas été reportée dans un fichier.

Ces deux mondes sont complémentaires et leur articulation est au cœur du travail quotidien : on règle souvent à chaud pour tester, puis on pérennise dans un fichier. Retenez dès maintenant cette règle pratique — un paramètre n'est vraiment acquis que lorsqu'il figure dans la configuration sur disque.

---

## `my.cnf` et `my.ini` : une affaire de plateforme

Les fichiers de configuration de MariaDB portent par convention deux noms selon le système d'exploitation. Sur les systèmes Unix et Linux, on parle de **`my.cnf`**. Sur Windows, le nom usuel est **`my.ini`** (le serveur sait aussi y lire un `my.cnf`).

Au-delà du nom, le principe est identique : il s'agit d'un fichier texte au format de type *INI*, organisé en sections délimitées par des intitulés entre crochets, chaque section adressant une partie de l'écosystème MariaDB (le serveur, les clients, certains outils). La structure interne de ces fichiers — quelles sections existent, à qui elles s'adressent et comment y écrire des options — fait l'objet de la sous-section suivante.

Un point distingue d'emblée MariaDB d'une configuration centralisée dans un fichier unique : le serveur ne lit pas *un* fichier, mais **une série de fichiers** répartis à des emplacements standards, et il sait composer une configuration finale à partir de cet ensemble. Cette mécanique — l'ordre dans lequel les fichiers sont lus et la façon dont les valeurs successives se complètent ou se remplacent — est déterminante pour comprendre pourquoi un réglage « ne prend pas » comme on l'attendait.

---

## Ce que couvre cette section

Cette section décompose la gestion des fichiers de configuration en trois temps :

- **11.1.1 — Structure et sections** : l'anatomie d'un fichier `my.cnf` / `my.ini`, ses sections et la manière d'y déclarer des options.
- **11.1.2 — Ordre de lecture des fichiers** : les emplacements standards, la séquence de lecture et les règles de priorité qui en découlent, y compris vis-à-vis de la ligne de commande.
- **11.1.3 — Include et configuration modulaire** : comment éclater la configuration en plusieurs fichiers pour la rendre plus maintenable, notamment dans les déploiements modernes et conteneurisés.

À l'issue de cette section, vous saurez où placer un réglage, dans quel fichier et dans quelle section, et vous comprendrez pourquoi MariaDB retient telle valeur plutôt qu'une autre lorsque plusieurs fichiers entrent en jeu.

⏭️ [my.cnf / my.ini : Structure et sections](/11-administration-configuration/01.1-structure-mycnf.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3 Sauvegarde physique (Mariabackup)

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Là où la sauvegarde **logique** ([section 12.2](02-sauvegarde-logique.md)) reconstitue la base sous forme d'instructions SQL, la sauvegarde **physique** procède tout autrement : elle copie directement les **fichiers de données** du moteur de stockage tels qu'ils existent sur le disque. On ne reconstruit pas la base, on en duplique l'image binaire.

L'outil de référence pour cela dans MariaDB est **Mariabackup**. Livré avec le serveur (parfois dans un paquet dédié `mariadb-backup`), il réalise des sauvegardes **physiques « à chaud »** : il peut copier les fichiers d'une instance **en cours d'exécution**, sans l'arrêter et, pour les tables InnoDB, sans bloquer les écritures. Cette section présente l'outil, le principe de la sauvegarde à chaud et le workflow caractéristique en trois phases. Les procédures détaillées — sauvegarde complète, incrémentale et le mécanisme `BACKUP STAGE` — sont traitées dans les sous-sections suivantes.

---

## Pourquoi la sauvegarde physique ?

La distinction logique / physique a été détaillée en [section 12.2](02-sauvegarde-logique.md) ; rappelons ici l'essentiel du point de vue de Mariabackup. L'approche physique brille sur les **gros volumes**, pour une raison simple : copier des fichiers est bien plus rapide que de parcourir les données, les sérialiser en SQL, puis — à la restauration — réexécuter chaque insertion et reconstruire tous les index.

Le gain le plus déterminant se situe d'ailleurs à la **restauration** : remettre des fichiers en place est sans commune mesure avec le rejeu d'un dump logique. Pour une base de plusieurs centaines de gigaoctets, l'écart se compte souvent en heures. En contrepartie, la sauvegarde physique est **moins portable** (liée à la version de MariaDB, à l'architecture et au format des fichiers), **non lisible**, et de granularité plus grossière. Les deux approches sont donc **complémentaires** plutôt que concurrentes.

---

## Qu'est-ce que Mariabackup ?

Mariabackup est un **fork de Percona XtraBackup**, adapté aux spécificités de MariaDB. Historiquement, XtraBackup ne prenait pas en charge certaines fonctionnalités propres à MariaDB (notamment ses versions d'InnoDB et son chiffrement des données) ; MariaDB a donc développé Mariabackup pour disposer d'un outil de sauvegarde physique pleinement compatible avec son serveur.

Concrètement, c'est un utilitaire en ligne de commande (binaire `mariabackup`) qui :

- copie les fichiers de l'instance **pendant que le serveur tourne** ;
- gère nativement les fonctionnalités MariaDB comme le **chiffrement au repos** et la **compression** ;
- prend en charge les sauvegardes **complètes** et **incrémentales** ;
- s'appuie sur le mécanisme de verrouillage moderne **`BACKUP STAGE`** de MariaDB.

Contrairement à `mydumper`, il **fait partie de la distribution MariaDB** : aucune installation tierce n'est requise.

---

## Le principe : une sauvegarde « à chaud » d'InnoDB

L'idée centrale de Mariabackup repose sur le fonctionnement même d'InnoDB. Comme le serveur continue d'écrire pendant la copie, les fichiers de données copiés sont **incohérents** au moment de la copie : certaines pages reflètent un état antérieur, d'autres un état postérieur à des transactions en cours. On parle d'image « floue » (*fuzzy*).

Pour rétablir la cohérence, Mariabackup exploite le **journal de rétablissement** (*redo log*) d'InnoDB : pendant qu'il copie les fichiers, il enregistre **en parallèle** toutes les modifications inscrites dans le redo log. Il dispose ainsi, à la fin de la copie, à la fois d'une image floue des données **et** de la liste des changements survenus durant la copie. Il suffira ensuite de **rejouer ce redo log** sur l'image floue pour la rendre cohérente — exactement comme InnoDB le ferait lors d'une reprise après un arrêt brutal (*crash recovery*).

Les tables des moteurs **non transactionnels** (MyISAM, Aria) ne disposent pas de ce mécanisme : Mariabackup les copie en posant un **bref verrou** sur celles-ci, le temps de garantir leur cohérence.

---

## Le workflow en trois phases : *backup*, *prepare*, *restore*

C'est la particularité la plus importante à comprendre — et la source d'erreurs la plus fréquente chez les débutants. Une sauvegarde Mariabackup n'est **pas immédiatement restaurable** : elle doit d'abord être *préparée*. Le cycle complet comporte trois phases distinctes :

| Phase | Option principale | Rôle |
|-------|-------------------|------|
| **1. Sauvegarde** (*backup*) | `--backup` | Copie les fichiers de données et le redo log vers un répertoire cible. Le serveur tourne ; le résultat est une image *floue*. |
| **2. Préparation** (*prepare*) | `--prepare` | Applique le redo log à l'image copiée pour la rendre **cohérente**. Opération hors-ligne, sur le répertoire de sauvegarde. |
| **3. Restauration** (*restore*) | `--copy-back` ou `--move-back` | Replace les fichiers préparés dans le répertoire de données (`datadir`). Le serveur doit être **arrêté** et le `datadir` vide. |

Retenez la règle d'or : **on ne restaure jamais une sauvegarde qui n'a pas été préparée**. La phase *prepare* est ce qui transforme une copie brute et incohérente en une sauvegarde exploitable. La phase *restore*, quant à elle, exige l'arrêt du serveur, puisqu'on écrit directement dans son répertoire de données.

---

## Sauvegarde complète et incrémentale

Mariabackup prend en charge deux des stratégies vues en [section 12.1](01-strategies-sauvegarde.md) :

- la **sauvegarde complète** (*full*), qui copie l'intégralité des fichiers — détaillée en [12.3.1](03.1-full-backup.md) ;
- la **sauvegarde incrémentale**, qui ne copie que les **pages modifiées** depuis une sauvegarde de référence — détaillée en [12.3.2](03.2-incremental-backup.md).

Le mécanisme d'incrément repose sur le **LSN** (*Log Sequence Number*) d'InnoDB, déjà évoqué : chaque page porte un numéro de séquence permettant à Mariabackup d'identifier celles qui ont changé depuis la sauvegarde de base. C'est aussi par ce biais que l'on met en œuvre, en pratique, une stratégie **différentielle** (en prenant toujours la sauvegarde complète comme base de référence).

---

## `BACKUP STAGE` : le verrouillage moderne

Pour garantir la cohérence des tables non transactionnelles et relever proprement les coordonnées du journal binaire, les anciens outils s'appuyaient sur un verrou global brutal (`FLUSH TABLES WITH READ LOCK`), qui bloquait l'ensemble des écritures. MariaDB a introduit un mécanisme bien plus fin, **`BACKUP STAGE`** : une série d'étapes SQL permettant de n'immobiliser que ce qui est strictement nécessaire, et seulement au dernier moment. Mariabackup l'utilise en interne pour minimiser l'impact de la sauvegarde sur la production. Ce mécanisme est détaillé en [section 12.3.3](03.3-backup-stage.md).

---

## Fonctionnalités et prérequis

Au-delà de la sauvegarde de base, Mariabackup propose le **streaming** (envoi du flux vers la sortie standard ou sur le réseau, pour archiver à distance ou compresser à la volée), la **compression**, le **chiffrement**, ainsi que des **sauvegardes partielles** (limitées à certaines bases ou tables).

Côté contraintes, deux points sont à connaître dès maintenant. D'abord, Mariabackup doit **s'exécuter sur la même machine que le serveur**, puisqu'il accède directement aux fichiers de données. Ensuite, l'utilisateur qui lance la sauvegarde doit disposer des **privilèges appropriés** pour copier les données et interagir avec le verrouillage de sauvegarde. La configuration précise de ces privilèges accompagne la procédure de sauvegarde complète en [12.3.1](03.1-full-backup.md).

---

## Mariabackup ou sauvegarde logique : comment choisir ?

| Critère | Mariabackup (physique) | `mariadb-dump` / `mydumper` (logique) |
|---------|------------------------|----------------------------------------|
| Vitesse (gros volumes) | Très élevée | Modérée à faible |
| Vitesse de restauration | Très élevée | Lente (rejeu SQL + index) |
| Portabilité du livrable | Faible (liée à la version/architecture) | Élevée (SQL portable) |
| Granularité | Grossière (instance, partielle) | Fine (table, base, sélection) |
| Étape de préparation | Oui (*prepare* obligatoire) | Non |
| Migrations / changements de version | Peu adapté | Idéal |
| Cas d'usage idéal | **Grandes bases**, restauration rapide, faible interruption | Bases modérées, migrations, restauration sélective |

En résumé, Mariabackup est l'outil de choix pour les **bases volumineuses de production** où la rapidité de sauvegarde et surtout de restauration prime, tandis que l'approche logique reste préférable pour la portabilité, les migrations et la restauration ciblée d'objets.

---

## Structure de la section

- [**12.3.1 — Full backup**](03.1-full-backup.md) : réaliser une sauvegarde complète, sa préparation et sa restauration.
- [**12.3.2 — Incremental backup**](03.2-incremental-backup.md) : sauvegardes incrémentales fondées sur le LSN.
- [**12.3.3 — Support `BACKUP STAGE`**](03.3-backup-stage.md) : le mécanisme de verrouillage progressif utilisé par Mariabackup.

---

## À retenir

- La sauvegarde **physique** copie les **fichiers de données** du moteur, à l'inverse de l'approche logique qui produit du SQL.
- **Mariabackup**, livré avec MariaDB et issu de Percona XtraBackup, réalise des sauvegardes **« à chaud »** d'un serveur en activité, sans bloquer les écritures InnoDB.
- Le principe : copier une image *floue* des fichiers tout en enregistrant le **redo log**, puis rejouer ce dernier pour rétablir la cohérence (comme une reprise après incident).
- Le workflow comporte **trois phases** : *backup* (copie), *prepare* (mise en cohérence) et *restore* (remise en place, serveur arrêté). **Une sauvegarde non préparée n'est pas restaurable.**
- Mariabackup gère les sauvegardes **complètes** et **incrémentales** (via le **LSN**) et s'appuie sur **`BACKUP STAGE`** pour minimiser son impact.
- Choisissez Mariabackup pour les **grandes bases** et la rapidité de restauration ; l'approche logique pour la **portabilité** et la granularité.

---

⏮️ [12.2.3 — mydumper / myloader](02.3-mydumper-myloader.md) · ⏭️ [12.3.1 — Full backup](03.1-full-backup.md)

⏭️ [Full backup](/12-sauvegarde-restauration/03.1-full-backup.md)

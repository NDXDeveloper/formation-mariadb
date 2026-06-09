🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.4 Stratégies de mise à jour et upgrade paths

> **Chapitre 19 — Migration et Compatibilité** · Section 19.4  
> Niveau : DBA / Architecte

---

## Mettre à jour MariaDB, et non plus migrer depuis un autre système

Les sections précédentes traitaient la migration **depuis** d'autres SGBD. Cette section change de sujet : il s'agit de **mettre à jour MariaDB d'une version à une autre**. C'est l'opération que l'on réalise pour suivre le canal choisi en 19.3 — sauter de LTS en LTS, ou suivre le rythme trimestriel des versions Rolling.

Une mise à jour reste une opération **sensible** : elle peut toucher au format des données sur disque, aux tables système et introduire des **changements de comportement**. Elle se prépare donc avec une stratégie, et ne s'improvise pas. Cette section pose les concepts ; les deux sous-sections détaillent ensuite l'outil `mariadb-upgrade` (19.4.1) et le choix entre mise à jour en place et logique (19.4.2).

---

## Types de mises à jour

On distingue deux natures de mise à jour, d'exigence très différente :

- **Mise à jour mineure** (correctif au sein d'une série), par exemple `12.3.2 → 12.3.x`. Le format de stockage n'étant pas modifié, l'opération est généralement simple. Elle ne concerne que les LTS, puisque les versions Rolling ne reçoivent pas de correctifs après leur GA (voir 19.3).
- **Mise à jour majeure**, par exemple `11.8 → 12.3`. Elle demande davantage de précautions : exécution de `mariadb-upgrade`, revue des changements de comportement et tests préalables.

---

## Les chemins de mise à jour (upgrade paths)

La question des *upgrade paths* est celle de savoir **quels enchaînements de versions sont supportés**. La réponse a évolué :

- **Historiquement**, il fallait passer par **chaque version majeure successivement**.
- **Pour les versions modernes**, on peut désormais, dans la plupart des cas, mettre à jour **directement vers la version cible**. Pour un **serveur autonome**, le **saut de versions majeures ou LTS intermédiaires est pleinement pris en charge et testé**.

Il y a toutefois une condition impérative : même en sautant des versions, il faut **passer en revue la section « Incompatible Changes » de chaque version majeure** située entre la version de départ et la version cible. C'est ce qui garantit que l'application ne dépend pas d'une fonctionnalité, d'une configuration ou d'un mot réservé modifié entre-temps.

Le cas des **clusters Galera** est différent : lors d'une **mise à jour continue** (*rolling upgrade*, nœud par nœud, sans interruption de service), il faut procéder **une version majeure/LTS à la fois** — le saut n'est pas supporté en raison de la compatibilité de communication entre nœuds. En revanche, si une **interruption complète du cluster** est possible, on peut arrêter tous les nœuds, les mettre à jour vers la cible et redémarrer, auquel cas le saut direct redevient possible, comme pour un serveur autonome.

En lien avec 19.3, le chemin typique en production consiste à **sauter de LTS en LTS** (par exemple `11.8 → 12.3`), tandis que le canal Rolling impose des mises à jour trimestrielles.

---

## Deux approches techniques

Il existe deux grandes manières de réaliser une mise à jour majeure, détaillées en 19.4.2 :

- la **mise à jour en place** (*in-place*) installe les nouveaux binaires sur le répertoire de données existant, puis exécute `mariadb-upgrade`. C'est l'approche la plus **rapide**, mais aussi la plus **risquée** ;
- la **mise à jour logique** exporte les données avec l'ancien serveur, puis les réimporte dans une **installation neuve** de la nouvelle version. C'est l'approche la plus **fiable**, en particulier pour les **sauts de versions majeures**, car le nouveau serveur recrée les tables et index avec ses formats et valeurs par défaut les plus récents ; en contrepartie, elle implique une **interruption plus longue** pour les gros volumes.

Dans les deux cas, c'est l'outil **`mariadb-upgrade`** qui finalise une mise à jour en place, en mettant à jour les tables système et en vérifiant les tables (voir 19.4.1).

---

## Le downgrade : à éviter, d'où l'importance des sauvegardes

Un point structurant pour toute stratégie de mise à jour : **le retour à une version majeure antérieure (downgrade) n'est pas officiellement pris en charge**. MariaDB n'est pas conçu pour être rétrogradé entre versions majeures, pour plusieurs raisons : les **tables de privilèges et de statut** du schéma `mysql` évoluent à presque chaque version majeure, le **format des données sur disque** peut changer, et certaines évolutions internes des moteurs de stockage (comme le journal *redo* d'InnoDB) ne sont pas rétrocompatibles.

Un downgrade en place n'est en théorie possible que **tant que `mariadb-upgrade` n'a pas été exécuté** sur la nouvelle version — or son exécution est précisément recommandée après une mise à jour. La conséquence pratique est claire : on ne planifie **pas** son retour arrière sur un downgrade, mais sur une **sauvegarde** (restauration) ou sur un **réplica resté en version inférieure**. Cela rend la **sauvegarde préalable indispensable** (le mécanisme de retour arrière est traité en 19.7).

---

## Bonnes pratiques de mise à jour

Quelques réflexes réduisent fortement le risque :

- **Lire les notes de version** et la liste des changements incompatibles de la version cible — et de **toutes les versions majeures intermédiaires** en cas de saut. Le passage spécifique `11.8 → 12.3`, avec ses changements de comportement, est traité en 19.10.
- **Sauvegarder avant la mise à jour**, comme assurance de retour arrière, puisque le downgrade n'est pas une option fiable.
- **Tester** la mise à jour sur une copie ou un environnement de pré-production représentatif.
- En **réplication**, mettre à jour les **réplicas avant le primaire** : un réplica doit exécuter une version supérieure ou égale à celle du primaire (la réplication d'une version ancienne vers une plus récente est prise en charge). Une mise à jour fondée sur un réplica minimise par ailleurs l'interruption (voir aussi 19.8).
- Assurer un **arrêt propre** du serveur avant l'opération.
- **Exécuter `mariadb-upgrade`** après une mise à jour en place.
- **Choisir l'approche adaptée** : en place pour la rapidité, logique pour la sécurité et les grands sauts de versions.

---

## Organisation de la section

Les sous-sections suivantes approfondissent ces deux aspects :

- **19.4.1 — `mariadb-upgrade`** : l'outil à exécuter après une mise à jour en place, et ce qu'il fait concrètement.
- **19.4.2 — Mise à jour en place vs logique** : la comparaison détaillée des deux approches et leurs cas d'usage.

---

## En résumé

Mettre à jour MariaDB suppose de **planifier le chemin**, de **choisir l'approche** et de **sécuriser le retour arrière** :

- les **chemins** : pour un serveur autonome, le **saut de versions intermédiaires est supporté** (à condition de revoir les changements incompatibles de chaque version majeure traversée) ; pour un **cluster Galera en mise à jour continue**, on procède **une version à la fois** ;
- les **approches** : **en place** (rapide) ou **logique** (fiable, recommandée pour les grands sauts), `mariadb-upgrade` finalisant l'opération en place ;
- le **retour arrière** : le **downgrade n'étant pas supporté**, on s'appuie sur une **sauvegarde** ou un **réplica** en version inférieure — d'où l'importance de sauvegarder avant toute mise à jour.

La sous-section suivante, **19.4.1 (`mariadb-upgrade`)**, détaille l'outil central de la mise à jour en place.

⏭️ [mariadb-upgrade](/19-migration-compatibilite/04.1-mariadb-upgrade.md)

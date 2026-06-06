🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1 Stratégies de sauvegarde : Full, Incrémentale, Différentielle

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Avant de choisir un outil ou d'écrire la moindre ligne de configuration, une stratégie de sauvegarde se définit par une question simple : **que copie-t-on, et à quelle fréquence ?** C'est sur cette question que se distinguent les trois stratégies fondamentales — complète, incrémentale et différentielle. Toutes trois reposent sur la même idée de départ : il est rarement nécessaire de tout recopier à chaque sauvegarde, car entre deux instants la majorité des données n'a pas changé. Ce qui les sépare, c'est la façon dont elles définissent ce qui « a changé » et le point de référence par rapport auquel ce changement est mesuré.

Ce choix n'est pas purement théorique : il détermine directement l'espace de stockage consommé, la durée et la charge des sauvegardes, mais aussi — et c'est le critère décisif — la **durée et la complexité de la restauration**. Comme souvent en matière de sauvegarde, ce qui est gagné d'un côté (des sauvegardes plus rapides et plus légères) se paie de l'autre (une restauration plus longue et plus fragile). Cette section présente chacune des trois stratégies, leurs compromis, et la manière dont elles s'incarnent concrètement dans l'écosystème MariaDB.

Une précision de vocabulaire s'impose d'emblée : dans le monde MariaDB, la distinction native se fait entre sauvegarde **complète** et sauvegarde **incrémentale** (notamment avec Mariabackup). La sauvegarde **différentielle** n'est pas un mode distinct, mais un *usage particulier* du mécanisme incrémental, comme nous le verrons plus bas. Cette nuance, propre aux bases de données, est essentielle pour bien interpréter les trois concepts.

---

## La sauvegarde complète (*Full backup*)

La sauvegarde complète est la plus simple à concevoir : elle produit une copie intégrale et autonome de l'ensemble des données, indépendamment de toute sauvegarde antérieure. Chaque sauvegarde complète constitue un point de restauration à elle seule, sans dépendre d'aucun autre fichier.

Dans MariaDB, elle prend deux formes principales. La voie *logique*, avec `mariadb-dump`, génère un fichier contenant les instructions SQL nécessaires pour reconstruire intégralement la structure et le contenu de la base : un export `mariadb-dump` est, par nature, toujours une sauvegarde complète. La voie *physique*, avec **Mariabackup**, copie directement les fichiers de données du moteur (typiquement InnoDB) pour produire une image complète exploitable.

Son principal atout est la **simplicité de restauration** : disposer d'une seule sauvegarde complète suffit à remettre l'intégralité des données en service, sans assemblage ni chaîne de dépendances. C'est aussi la stratégie la plus robuste, puisqu'aucun maillon manquant ne peut compromettre la restauration.

En contrepartie, elle est la plus coûteuse à produire. Recopier l'intégralité des données à chaque cycle consomme un volume de stockage élevé, mobilise une fenêtre de sauvegarde longue et impose une charge d'entrées/sorties importante sur le serveur. Sur une base volumineuse qui évolue peu entre deux sauvegardes, répéter des sauvegardes complètes revient surtout à recopier en boucle des données identiques — un gaspillage que les deux autres stratégies cherchent précisément à éviter.

---

## La sauvegarde incrémentale (*Incremental backup*)

La sauvegarde incrémentale ne capture que les données **modifiées depuis la dernière sauvegarde**, qu'il s'agisse de la sauvegarde complète initiale ou de la sauvegarde incrémentale précédente. On constitue ainsi une *chaîne* : une sauvegarde complète de base, suivie d'une série de sauvegardes incrémentales qui ne contiennent chacune que le « delta » accumulé depuis la sauvegarde immédiatement antérieure.

Techniquement, Mariabackup s'appuie sur le **LSN** (*Log Sequence Number*) d'InnoDB pour déterminer quelles pages de données ont été modifiées depuis le point de référence : seule la différence est copiée. Les *binary logs* offrent un autre mécanisme de capture continue des modifications, de logique très proche, qui sera détaillé dans la section [12.4](04-sauvegarde-incrementale-binlog.md).

L'avantage est net du côté de la sauvegarde : chaque incrément étant minimal, il occupe peu d'espace, se réalise rapidement et sollicite peu le serveur. Cette légèreté autorise des sauvegardes **fréquentes** — toutes les heures, par exemple — ce qui améliore directement le RPO (la perte de données potentielle est réduite à l'intervalle entre deux incréments).

Le revers se manifeste lors de la restauration. Pour reconstruire l'état des données à un instant donné, il faut repartir de la sauvegarde complète puis **rejouer dans l'ordre exact toutes les sauvegardes incrémentales** intermédiaires. La restauration est donc plus lente et plus complexe. Surtout, la chaîne est **fragile** : si un seul incrément est corrompu, manquant ou appliqué dans le mauvais ordre, toutes les sauvegardes situées après lui dans la chaîne deviennent inexploitables. Plus la chaîne est longue, plus ce risque augmente.

Le schéma suivant illustre une rotation hebdomadaire en mode incrémental (sauvegarde complète le lundi) :

| Jour | Type de sauvegarde | Contenu capturé | Base de référence |
|------|--------------------|------------------|--------------------|
| Lundi | Complète | Toutes les données | — |
| Mardi | Incrémentale | Modifications depuis **lundi** | Complète (lundi) |
| Mercredi | Incrémentale | Modifications depuis **mardi** | Incrémentale (mardi) |
| Jeudi | Incrémentale | Modifications depuis **mercredi** | Incrémentale (mercredi) |
| Vendredi | Incrémentale | Modifications depuis **jeudi** | Incrémentale (jeudi) |

En cas d'incident le jeudi soir, la restauration nécessite quatre artefacts à appliquer dans l'ordre : la complète du lundi, puis les incrémentales du mardi, du mercredi et du jeudi.

---

## La sauvegarde différentielle (*Differential backup*)

La sauvegarde différentielle capture toutes les modifications survenues **depuis la dernière sauvegarde complète**, et non depuis la dernière sauvegarde quelle qu'elle soit. Chaque différentielle est donc *cumulative* : elle englobe toutes les précédentes différentielles du même cycle, et ne dépend que de la sauvegarde complète de base — jamais d'une autre différentielle.

C'est sur ce point que se joue la différence avec MariaDB. Mariabackup ne propose pas de mode « différentiel » distinct : on obtient ce comportement en réalisant des sauvegardes incrémentales dont la **base de référence pointe systématiquement vers la sauvegarde complète** (et non vers la sauvegarde incrémentale précédente). Autrement dit, une sauvegarde différentielle est une sauvegarde incrémentale dont le point de comparaison reste figé sur la sauvegarde complète. La mise en œuvre concrète de cette option est traitée dans la section [12.3](03-sauvegarde-physique-mariabackup.md).

L'intérêt principal réside dans la **restauration**, qui ne requiert que deux artefacts : la sauvegarde complète et la **dernière** différentielle en date. Plus rapide et nettement plus simple qu'une restauration incrémentale, cette approche est aussi plus robuste, car elle élimine la longue chaîne de dépendances : la perte d'une différentielle ancienne n'a aucune incidence sur la capacité à restaurer à partir de la plus récente.

Son inconvénient est l'effet inverse de l'incrémental : chaque différentielle étant cumulative, elle **grossit au fil du cycle**. En milieu ou en fin de semaine, une différentielle peut devenir presque aussi volumineuse qu'une sauvegarde complète, consommant davantage d'espace et de temps qu'une incrémentale équivalente.

Le même cycle hebdomadaire, en mode différentiel, donne :

| Jour | Type de sauvegarde | Contenu capturé | Base de référence |
|------|--------------------|------------------|--------------------|
| Lundi | Complète | Toutes les données | — |
| Mardi | Différentielle | Modifications depuis **lundi** | Complète (lundi) |
| Mercredi | Différentielle | Modifications depuis **lundi** | Complète (lundi) |
| Jeudi | Différentielle | Modifications depuis **lundi** | Complète (lundi) |
| Vendredi | Différentielle | Modifications depuis **lundi** | Complète (lundi) |

En cas d'incident le jeudi soir, la restauration ne mobilise que deux artefacts : la complète du lundi et la différentielle du jeudi.

---

## Comparaison des trois stratégies

Le tableau ci-dessous synthétise les compromis caractéristiques de chaque approche. La logique d'ensemble est constante : on échange systématiquement de la **rapidité de sauvegarde** contre de la **rapidité de restauration**.

| Critère | Complète | Incrémentale | Différentielle |
|---------|----------|--------------|----------------|
| Espace par sauvegarde | Élevé | Faible | Croissant (moyen) |
| Durée de la sauvegarde | Longue | Courte | Croissante |
| Charge I/O sur le serveur | Élevée | Faible | Moyenne |
| Durée de la restauration | Courte | Longue | Moyenne |
| Complexité de la restauration | Minimale | Élevée | Faible |
| Artefacts nécessaires pour restaurer | 1 | Complète + **toutes** les incrémentales | Complète + **1** différentielle |
| Robustesse de la chaîne | Autonome (sans dépendance) | Fragile (chaîne longue) | Solide (aucune chaîne entre différentielles) |

La sauvegarde complète privilégie la sécurité et la simplicité au prix du coût. L'incrémentale optimise le coût et la fenêtre de sauvegarde au prix d'une restauration lente et fragile. La différentielle occupe une position intermédiaire, en cherchant un équilibre entre les deux.

---

## Comment choisir : les facteurs de décision

Aucune des trois stratégies n'est universellement supérieure ; le choix dépend du contexte et, plus précisément, de l'arbitrage entre les objectifs définis au niveau du chapitre (RPO et RTO) et les ressources disponibles. Les principaux facteurs à pondérer sont les suivants.

Le **volume de la base** et son **taux de modification** : une grande base qui change peu se prête bien aux stratégies incrémentale ou différentielle, là où une petite base peut se contenter de sauvegardes complètes répétées. La **fenêtre de sauvegarde** disponible, c'est-à-dire la durée pendant laquelle on peut sauvegarder sans pénaliser la production, pèse en faveur des approches légères lorsqu'elle est courte. Le **RTO**, qui exige une restauration rapide, oriente vers la sauvegarde complète ou différentielle plutôt qu'incrémentale. Le **RPO**, qui réclame une faible perte de données, plaide pour des sauvegardes très fréquentes — donc incrémentales, souvent complétées par les *binary logs*. Enfin, le **budget de stockage** et la durée de **rétention** souhaitée tranchent souvent l'équation économique.

---

## Stratégies combinées en pratique

En production, on n'oppose presque jamais les trois stratégies : on les **combine** au sein d'une rotation. Deux schémas reviennent fréquemment.

Le premier associe une **sauvegarde complète hebdomadaire** à des **sauvegardes incrémentales quotidiennes** (voire horaires), elles-mêmes complétées par l'archivage continu des *binary logs*. Cette combinaison minimise l'espace et la fenêtre de sauvegarde et offre un excellent RPO, au prix d'une restauration plus longue à orchestrer.

Le second remplace les incrémentales par des **différentielles quotidiennes** : on conserve une sauvegarde complète hebdomadaire et l'on accumule chaque jour une différentielle. La restauration redevient simple (complète + dernière différentielle) tout en limitant le coût des sauvegardes quotidiennes, ce qui constitue un compromis apprécié pour de nombreuses charges de travail.

Ces rotations s'inscrivent souvent dans un schéma de conservation structuré, dont le plus répandu est le **GFS** (*Grandfather-Father-Son*, ou Grand-père–Père–Fils) : les sauvegardes quotidiennes (les « fils ») sont conservées quelques jours, les sauvegardes hebdomadaires (les « pères ») quelques semaines, et les sauvegardes mensuelles (les « grands-pères ») plusieurs mois. Ce type de politique permet de concilier un historique de restauration suffisant avec une consommation de stockage maîtrisée.

---

## La place des binary logs

Les trois stratégies décrites ici fonctionnent par instantanés, à des moments discrets dans le temps. Entre deux sauvegardes, les modifications les plus récentes ne sont pas encore protégées. C'est précisément le rôle des **binary logs** : en journalisant chaque transaction au fil de l'eau, ils permettent de rejouer les modifications survenues *après* la dernière sauvegarde et constituent le socle de la restauration à un instant précis (*Point-in-Time Recovery*). Ils complètent ainsi naturellement n'importe laquelle des trois stratégies, en comblant l'intervalle qui les sépare. Leur configuration et leur exploitation sont détaillées dans la section [12.4](04-sauvegarde-incrementale-binlog.md).

---

## À retenir

- La **sauvegarde complète** copie l'intégralité des données ; elle est autonome et offre la restauration la plus simple, mais c'est la plus coûteuse en espace et en temps.
- La **sauvegarde incrémentale** ne capture que les modifications depuis la *dernière sauvegarde* ; légère et rapide, elle impose une restauration longue et une chaîne fragile.
- La **sauvegarde différentielle** capture les modifications depuis la *dernière sauvegarde complète* ; cumulative, elle simplifie la restauration (deux artefacts) au prix d'une taille croissante.
- Dans MariaDB, **complète** et **incrémentale** sont natives (Mariabackup, via le LSN d'InnoDB) ; la **différentielle** s'obtient en gardant la sauvegarde complète comme base de référence fixe.
- Le bon choix résulte d'un arbitrage entre **RPO**, **RTO**, volume et budget — et passe le plus souvent par une **combinaison** des stratégies, complétée par les **binary logs**.

---

⏮️ [Chapitre 12 — Introduction](README.md) · ⏭️ [12.2 — Sauvegarde logique](02-sauvegarde-logique.md)

⏭️ [Sauvegarde logique](/12-sauvegarde-restauration/02-sauvegarde-logique.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.13 Adaptive Hash Index et optimisations du buffer pool

Cette section traite de deux mécanismes internes d'InnoDB qui influencent les performances en lecture : l'**Adaptive Hash Index**, un index de hachage en mémoire construit automatiquement au-dessus des pages d'index les plus sollicitées, et les **optimisations du buffer pool** au-delà de son simple dimensionnement (vu aux §15.2.1 et §15.2.2) — son algorithme de remplacement, son réchauffement par sauvegarde/restauration, et son redimensionnement en ligne. Ces mécanismes sont en grande partie automatiques, mais les comprendre permet au DBA de les régler — ou de les désactiver — à bon escient.

## L'Adaptive Hash Index (AHI)

### Principe

InnoDB observe en permanence les motifs de recherche dans les index. Lorsqu'il constate que certaines pages d'index sont consultées fréquemment selon un même motif de préfixe de clé, il construit en mémoire un **index de hachage** qui associe directement ces préfixes aux enregistrements. Une recherche qui exigerait normalement la traversée d'un arbre B (coût logarithmique) se ramène alors à un accès quasi direct par hachage, pour les données les plus chaudes.

Cet index est bâti **à la demande**, uniquement pour les pages fréquemment accédées, et il est entièrement automatique et auto-régulé : on ne le crée ni ne le maintient manuellement. Il réside dans le buffer pool, dont il consomme une partie de la mémoire. Quand le jeu de travail tient largement en RAM et que les accès sont répétitifs, l'AHI permet à InnoDB de se comporter davantage comme une base en mémoire.

### Un mécanisme désactivé par défaut depuis MariaDB 10.5

C'est le point essentiel de cette section. L'AHI est piloté par la variable `innodb_adaptive_hash_index` (`ON`/`OFF`). Il était **activé par défaut jusqu'à MariaDB 10.5**, mais il est **désactivé par défaut depuis** cette version — et donc en 12.3.

Ce changement de défaut s'explique par plusieurs constats. D'une part, l'AHI a été à l'origine de bugs et d'un risque de corruption de données dans certaines conditions. D'autre part, il consomme de la mémoire du buffer pool, ce qui peut dégrader les performances précisément lorsque le jeu de travail tiendrait presque entièrement dans ce buffer pool — l'espace soustrait par l'AHI provoquant alors davantage d'évictions. À cela s'ajoute la contention sur le verrou de l'AHI dans les charges à forte concurrence. Pour ces raisons, MariaDB a fait le choix conservateur de le désactiver par défaut (une décision qu'a également suivie MySQL dans ses versions récentes).

La variable est dynamique : on peut activer ou désactiver l'AHI à chaud, sans redémarrage.

```sql
-- Activer l'AHI à chaud (il est désactivé par défaut depuis MariaDB 10.5)
SET GLOBAL innodb_adaptive_hash_index = ON;
```

### Partitionnement et contention

Lorsqu'il est actif, l'AHI est **partitionné en interne** — chaque partition étant protégée par un verrou distinct — afin de réduire la contention historiquement associée à son verrou unique, qui constituait un goulet d'étranglement sur les charges fortement concurrentes ; le nombre de partitions est paramétrable. Cette contention se surveille dans la section `SEMAPHORES` de `SHOW ENGINE INNODB STATUS` : de nombreux threads en attente sur les verrous de l'AHI signalent qu'il faut augmenter le nombre de partitions ou, plus radicalement, désactiver l'AHI.

### Faut-il l'activer ?

Il n'existe pas de réponse universelle : la seule démarche valable est de **mesurer**, en exécutant un benchmark représentatif avec l'AHI activé puis désactivé (au sens du §15.12). L'AHI peut aider sur les charges majoritairement en lecture, dominées par des recherches d'égalité répétitives sur un ensemble stable d'index chauds, lorsque les données tiennent largement en mémoire. Il peut au contraire nuire sur les charges mixtes à forte concurrence, sur celles aux motifs d'accès variés, en présence de nombreuses opérations DDL (les `DROP` et `TRUNCATE` doivent purger l'AHI), ou lorsque le jeu de travail remplit presque le buffer pool.

Le défaut conservateur (désactivé) traduit le fait que, pour beaucoup de charges modernes, l'AHI coûte plus qu'il ne rapporte. On ne l'activera donc qu'après avoir mesuré un gain réel.

### Surveiller l'AHI

La section `INSERT BUFFER AND ADAPTIVE HASH INDEX` de `SHOW ENGINE INNODB STATUS` rapporte le nombre de recherches servies par hachage par seconde face aux recherches non hachées : ce rapport indique dans quelle mesure l'AHI est réellement utilisé. La taille de la table de hachage y révèle par ailleurs son empreinte mémoire.

## Les optimisations du buffer pool

Le buffer pool est le cache mémoire d'InnoDB pour les pages de données et d'index ; son dimensionnement (§15.2.1) et son découpage en instances (§15.2.2) en constituent les leviers de premier ordre. Cette section aborde les mécanismes internes qui régissent ce que le buffer pool conserve, la façon dont il se réchauffe et celle dont il se redimensionne.

### L'algorithme LRU à point d'insertion médian

InnoDB n'emploie pas un simple algorithme LRU. Il utilise une variante à **insertion médiane**, dans laquelle la liste des pages est scindée en deux sous-listes : la sous-liste « jeune » (pages récemment et fréquemment utilisées) et la sous-liste « ancienne ». Une page nouvellement lue n'entre pas en tête de la liste entière, mais au **point médian**, c'est-à-dire en tête de la sous-liste ancienne. Elle n'est promue vers la sous-liste jeune que si elle est consultée à nouveau **après** un délai de `innodb_old_blocks_time` millisecondes (1000 par défaut). La taille de la sous-liste ancienne est fixée par `innodb_old_blocks_pct` (environ 37 % par défaut).

L'objectif est la **résistance aux scans**. Un balayage ponctuel et massif — une lecture de table complète, une sauvegarde — remplit la sous-liste ancienne de pages qui en sont rapidement évincées, sans déloger les pages chaudes installées dans la sous-liste jeune. Sans ce mécanisme, un seul gros balayage suffirait à polluer l'intégralité du cache et à en chasser les données réellement utiles. Pour des charges très marquées par les balayages, on peut renforcer cette protection en réduisant `innodb_old_blocks_pct` ou en augmentant `innodb_old_blocks_time`.

### Réchauffement : sauvegarde et restauration du buffer pool

Après un redémarrage, le buffer pool est vide : les premières requêtes sont lentes car servies depuis le disque, jusqu'à ce que le cache se remplisse de nouveau — c'est le problème du cache froid évoqué au §15.12. InnoDB le résout en **sauvegardant la liste des pages** présentes dans le buffer pool à l'arrêt, puis en la **rechargeant** au démarrage :

- `innodb_buffer_pool_dump_at_shutdown` (activé par défaut) déclenche la sauvegarde à l'arrêt ;
- `innodb_buffer_pool_load_at_startup` (activé par défaut) déclenche le rechargement au démarrage ;
- `innodb_buffer_pool_dump_pct` (25 % par défaut) fixe la proportion des pages les plus récemment utilisées à sauvegarder.

La sauvegarde ne porte que sur les **identifiants** des pages — un fichier de petite taille —, et non sur les données elles-mêmes, qui sont relues depuis le disque lors du chargement. Une commande manuelle reste possible à tout moment :

```sql
-- Déclencher manuellement la sauvegarde puis le rechargement du buffer pool
SET GLOBAL innodb_buffer_pool_dump_now = ON;  
SET GLOBAL innodb_buffer_pool_load_now = ON;  
```

Ce dispositif raccourcit considérablement la montée en température après un redémarrage planifié ou une mise à niveau.

### Redimensionnement en ligne

Le buffer pool peut être redimensionné **sans redémarrage**, en modifiant dynamiquement sa taille :

```sql
-- Redimensionner le buffer pool à 8 Gio en ligne
SET GLOBAL innodb_buffer_pool_size = 8589934592;
```

Sur MariaDB 12.3, ce redimensionnement est **fin** (par incréments souples) et plafonné par `innodb_buffer_pool_size_max` fixé au démarrage ; l'ancien découpage en *chunks* (`innodb_buffer_pool_chunk_size`) est désormais **déprécié et sans effet**, comme l'explique le §15.2.1. Fait notable qui relie les deux thèmes de cette section, InnoDB **désactive temporairement l'AHI** pendant le redimensionnement, puis le reconstruit une fois l'opération terminée, en même temps qu'il redimensionne ses tables de hachage internes. Le redimensionnement en ligne permet d'adapter la mémoire allouée à une charge évolutive sans interruption de service.

### Le vidage des pages modifiées

Le buffer pool contient des pages « sales » (modifiées) qui doivent être périodiquement écrites sur disque. Le rythme de ce vidage est gouverné par un ensemble de paramètres traités ailleurs dans ce chapitre : `innodb_io_capacity` et `innodb_io_capacity_max` (§15.4.1), `innodb_flush_method` (§15.4.2), ainsi que le vidage adaptatif (`innodb_adaptive_flushing`), le seuil de pages modifiées (`innodb_max_dirty_pages_pct`) et le vidage des pages voisines (`innodb_flush_neighbors`). On retiendra ici simplement que ce vidage est l'opération qui relie le buffer pool au sous-système d'entrées/sorties, et que son réglage conditionne la régularité des écritures.

### Surveiller le buffer pool

L'indicateur fondamental est le **taux de succès** du buffer pool, que l'on calcule en rapprochant `Innodb_buffer_pool_read_requests` (lectures logiques) de `Innodb_buffer_pool_reads` (lectures ayant dû atteindre le disque). Un taux proche de 100 % signifie que la quasi-totalité des lectures sont servies depuis la mémoire. La section `BUFFER POOL AND MEMORY` de `SHOW ENGINE INNODB STATUS`, ainsi que la table `INFORMATION_SCHEMA.INNODB_BUFFER_POOL_STATS`, en fournissent l'état détaillé. Un taux de succès durablement faible indique que le buffer pool est sous-dimensionné par rapport au jeu de travail — un signal qui renvoie au dimensionnement du §15.2.1.

## Évolutions de la série 12.x

La série 12.x a poursuivi le travail d'amélioration de la concurrence et de la gestion mémoire d'InnoDB. Conjuguées à la réécriture du binlog intégré à InnoDB (§11.5.4), ces évolutions contribuent aux gains observés sur le chemin d'écriture de cette série. Pour le comportement spécifique d'une version donnée, on se reportera aux notes de version correspondantes.

## En conclusion

L'Adaptive Hash Index et les mécanismes internes du buffer pool sont en grande partie automatiques, mais ils ne dispensent pas le DBA de trois décisions : dimensionner correctement le buffer pool (§15.2.1), déterminer **par la mesure** si l'AHI apporte un gain sur sa charge — sachant qu'il est désormais désactivé par défaut —, et tirer parti du réchauffement et du redimensionnement en ligne pour limiter l'impact des redémarrages et des variations de charge. La section suivante (§15.14) se tourne vers l'optimiseur lui-même, et notamment l'amélioration de son modèle de coût pour tenir compte des disques SSD.

⏭️ [Cost-based optimizer amélioré (prise en compte SSD)](/15-performance-tuning/14-cost-based-optimizer-ssd.md)

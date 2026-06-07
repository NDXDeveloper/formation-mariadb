🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.14 L'optimiseur basé sur les coûts amélioré (prise en compte SSD)

L'optimiseur de requêtes de MariaDB choisit le plan d'exécution en estimant le **coût** de chaque alternative possible. La qualité d'un plan dépend donc directement de la justesse de ces estimations. En MariaDB 11.0, ce modèle de coût a été profondément réécrit : passage d'un modèle en partie fondé sur des règles à un modèle **purement basé sur les coûts**, recalibré pour le matériel moderne — au premier rang duquel les disques SSD. C'est ce modèle qui est en vigueur en 12.3.

Cette section explique la refonte, ses hypothèses relatives au SSD, les variables de coût ajustables et la façon de les inspecter — tout en insistant sur un point : les valeurs par défaut sont bonnes, et leur ajustement relève d'une démarche d'expert, en dernier recours.

## Le rôle de l'optimiseur basé sur les coûts

Pour une même requête, plusieurs plans d'exécution sont généralement envisageables : quel index utiliser, dans quel ordre joindre les tables, faut-il un balayage de table ou une recherche par index, recourir à une fusion d'index, etc. L'optimiseur attribue un coût estimé à chacun de ces plans et retient le moins coûteux. Si l'estimation est juste, le plan choisi est efficace ; si elle est faussée, l'optimiseur peut retenir un plan sous-optimal.

C'est précisément pour corriger des estimations imparfaites que le modèle a été refondu. Ce mécanisme s'articule avec l'analyse des plans d'exécution (`EXPLAIN`, §5.7) et l'optimisation des requêtes (§5.8) ; lorsqu'il se trompe malgré tout, les Optimizer Hints (§15.15) permettent de forcer un choix précis.

## La refonte du modèle de coût en MariaDB 11.0

Avant la version 11.0, l'optimiseur reposait sur un « coût de base » de 1 pour un accès disque, la lecture d'une clé, la lecture d'une ligne à partir d'un rowid, et quelques autres petits coûts. Ces valeurs étaient raisonnables pour choisir le bon index, mais bien moins adaptées pour estimer le coût d'un balayage de table, d'un parcours d'index ou d'une recherche par intervalle. Ces coûts, quelque peu arbitraires, supposaient en outre des caractéristiques héritées de l'ère des disques durs, et la communauté avait rapporté de nombreux cas de plans inadaptés.

MariaDB 11.0 a opéré le passage d'un modèle fondé sur des règles à un modèle **purement basé sur les coûts**, en recalibrant ces derniers pour qu'ils reflètent un **temps réel**. La référence retenue est qu'une opération du moteur de stockage correspond à l'ordre de grandeur d'une milliseconde, de sorte que le coût total estimé d'une requête approche le temps effectivement passé dans le moteur de stockage, le cache de jointure et les tris. Pour rester lisibles, les coûts élémentaires sont exprimés en **microsecondes**. L'ensemble des coûts du moteur a par ailleurs été décomposé en composantes fines — recherche de clé, recherche de ligne, copie de clé ou de ligne, comparaisons, copie de bloc d'index, initialisation de balayage — afin d'en améliorer la précision. C'est ce modèle qui est resté en place de la 11.0 jusqu'à la 12.3.

## La prise en compte du SSD

Le point qui donne son titre à cette section est le suivant : par défaut, les coûts d'accès disque supposent désormais un **SSD moderne** — un SSD de milieu de gamme délivrant de l'ordre de 400 Mo/s — et non plus un disque dur à plateaux. Le coût par défaut de lecture d'un bloc, `optimizer_disk_read_cost`, vaut ainsi environ **10,24 microsecondes**.

La conséquence pratique est importante : l'optimiseur valorise correctement les accès aléatoires comme étant peu coûteux, puisqu'un SSD ne souffre pas de la pénalité de positionnement (seek) d'un disque mécanique. Cela modifie certains choix de plan — l'optimiseur est par exemple plus enclin à recourir à des recherches par index et à des accès aléatoires là où un modèle hérité du disque dur aurait privilégié un balayage séquentiel. Pour le cas, devenu plus rare, d'un stockage sur disque dur, l'utilisateur peut au contraire relever ce coût (le moteur Archive sur disque dur, par exemple, où un positionnement prend 8 à 10 ms).

## Les variables de coût ajustables

L'ensemble des coûts est exposé sous forme de variables préfixées `optimizer_`, exprimées en microsecondes. Les deux plus structurantes sont les suivantes.

`optimizer_disk_read_cost` (environ 10,24 par défaut) est le coût de lecture d'un bloc depuis le disque, calibré pour un SSD.

`optimizer_disk_read_ratio` (environ 0,02 par défaut) modélise la part des lectures qui atteignent réellement le disque par opposition à celles servies par le cache : une valeur de 0,02 traduit un taux de succès du cache d'environ 98 %. C'est ainsi que le modèle relie le coût à la **réalité du cache** — un ratio faible reflète un cache chaud, un ratio plus élevé un accès davantage tributaire du disque. Pour un buffer pool InnoDB dont le taux de succès n'est que de 80 %, on porterait par exemple ce ratio à 0,20. Cette variable fait ainsi directement écho au taux de succès du buffer pool évoqué au §15.13.

À ces deux variables s'ajoutent les composantes fines déjà mentionnées (coûts de recherche et de copie de clé et de ligne, comparaisons, initialisation de balayage, coût d'évaluation de la clause `WHERE`, etc.). Toutes peuvent être définies via le fichier de configuration, un paramètre de ligne de commande, ou en SQL au moyen de `SET GLOBAL` ou `SET SESSION`.

## Des coûts définis par moteur de stockage

Une particularité notable du modèle est que les coûts peuvent être définis **par moteur de stockage**, car les moteurs présentent des caractéristiques différentes : InnoDB possède un index clusterisé, MyISAM et Aria n'en ont pas, Archive peut résider sur disque dur, et ainsi de suite. Les coûts par défaut de l'« engine » nommé `default` conviennent aux moteurs sans index clusterisé, tandis qu'InnoDB dispose de ses propres coûts — ses coûts de recherche de clé et de ligne sont plus élevés, reflet de la traversée vers la clé primaire qu'impose son index clusterisé.

Les coûts spécifiques à un moteur sont de portée **globale** (et non de session, contrairement aux autres coûts qui peuvent l'être). Pour des raisons de performance, ils sont stockés dans le cache de définition des tables (`TABLE_SHARE`) ; en conséquence, modifier le coût d'un moteur ne s'applique qu'aux tables nouvellement mises en cache, et il faut recourir à `FLUSH TABLES` pour forcer la prise en compte immédiate. La configuration suivante illustre une cohabitation SSD/disque dur :

```ini
[mariadbd]
# Archive sur disque dur (temps de positionnement ~8 à 10 ms)
archive.optimizer_disk_read_cost = 8000
# Les autres moteurs sur SSD : valeur par défaut
optimizer_disk_read_cost = 10.24
```

L'intégralité des coûts par moteur est consultable dans la table `information_schema.optimizer_costs`.

## Inspecter et ajuster, avec prudence

L'inspection des coûts en vigueur se fait directement par requête :

```sql
SELECT * FROM information_schema.optimizer_costs  
WHERE engine = 'default'\G  
```

dont la sortie liste **toutes** les composantes de coût propres au moteur, avec leurs valeurs par défaut :

```
*************************** 1. row ***************************
                         ENGINE: default
       OPTIMIZER_DISK_READ_COST: 10.240000
OPTIMIZER_INDEX_BLOCK_COPY_COST: 0.035600
     OPTIMIZER_KEY_COMPARE_COST: 0.011361
        OPTIMIZER_KEY_COPY_COST: 0.015685
      OPTIMIZER_KEY_LOOKUP_COST: 0.435777
   OPTIMIZER_KEY_NEXT_FIND_COST: 0.082347
      OPTIMIZER_DISK_READ_RATIO: 0.020000
        OPTIMIZER_ROW_COPY_COST: 0.060866
      OPTIMIZER_ROW_LOOKUP_COST: 0.130839
   OPTIMIZER_ROW_NEXT_FIND_COST: 0.045916
   OPTIMIZER_ROWID_COMPARE_COST: 0.002653
      OPTIMIZER_ROWID_COPY_COST: 0.002653
```

Quelques coûts du modèle demeurent **globaux** plutôt que définis par moteur — en particulier l'initialisation de balayage (`optimizer_scan_setup_cost`, 10 par défaut) et l'évaluation de la clause `WHERE` (`optimizer_where_cost`) : ils n'apparaissent donc pas dans cette table par moteur, et se consultent via `SHOW VARIABLES LIKE 'optimizer_%cost%'`.

L'ajustement, lui, appelle la plus grande prudence. La visibilité de ces coûts **n'est pas destinée à un réglage courant** par les utilisateurs : il s'agit avant tout d'un outil permettant aux équipes de support et d'ingénierie de corriger un problème de plan sans avoir à reconstruire un binaire. Les valeurs par défaut sont conçues pour être adaptées dans la grande majorité des cas. On ne modifiera ces coûts qu'avec une raison claire et mesurée — un stockage sur disque dur, un coût manifestement mal estimé — et l'on validera toute modification par un benchmark (§15.12) et par l'Optimizer Trace.

Un dernier avertissement : modifier ces variables globalement affecte **toutes** les requêtes, ce qui est risqué. Pour une expérimentation, on préférera la portée `SESSION` ; et pour corriger une seule requête, on recourra plutôt aux Optimizer Hints (§15.15) qu'à un réglage global du modèle de coût.

```sql
-- Informer l'optimiseur d'un taux de succès de 80 % du buffer pool InnoDB
SET GLOBAL innodb.optimizer_disk_read_ratio = 0.20;
```

## Autres améliorations de l'optimiseur en 11.x

La refonte du modèle de coût s'est accompagnée d'améliorations connexes : la prise en compte du coût de tous les appels de lecture de bas niveau des moteurs, un meilleur choix de l'index de parcours dans le cadre des optimisations `GROUP BY`/`ORDER BY`, la création d'une clé distincte sur les tables dérivées et les unions (apportant une élimination automatique des doublons), et un Optimizer Trace nettement plus détaillé. Dans le même esprit de modernisation pour le matériel actuel, le **change buffering d'InnoDB a été retiré** en 11.0 : déjà désactivé par défaut auparavant, il représentait sur les disques modernes davantage une surcharge qu'un gain.

## Impact sur la migration

Comme ce modèle a changé en 11.0, une mise à niveau depuis une version antérieure (par exemple une 10.6 ou une 10.11) vers la série 11.x ou 12.x fait basculer l'optimiseur vers le nouveau modèle. Les plans d'exécution peuvent alors changer, et quelques requêtes pourraient même régresser — c'est d'ailleurs pour signaler ce risque que MariaDB avait incrémenté le numéro de version majeure. Après une telle mise à niveau, il convient de tester les requêtes critiques et de surveiller les changements de plan ; le nouveau modèle constitue généralement une amélioration, mais cela se vérifie (§19.4). En revanche, une migration entre la 11.8 et la 12.3 n'introduit pas ce changement, puisque les deux versions emploient déjà le nouveau modèle : cette précaution concerne donc les montées depuis les lignes LTS plus anciennes.

## En conclusion

Le nouveau modèle de coût, purement basé sur les coûts et calibré pour le SSD depuis la 11.0, conduit l'optimiseur à de meilleurs choix de plan sur le matériel moderne, sans intervention. Ses coûts fins, définissables par moteur et inspectables, sont puissants mais relèvent de l'expertise. Pour piloter le plan d'une requête au quotidien, on préférera les Optimizer Hints — fonctionnalité majeure de la série 12.x que présente la section suivante (§15.15).

⏭️ [Optimizer Hints : contrôle fin du plan d'exécution](/15-performance-tuning/15-optimizer-hints.md)

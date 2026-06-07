🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.9 Partitionnement de tables

Le partitionnement consiste à découper une table volumineuse en plusieurs morceaux physiques distincts — les *partitions* — tout en continuant à la manipuler comme une seule table logique. Pour l'application, le code SQL et les utilisateurs, rien ne change : on lit et on écrit dans `commandes`, on ne voit jamais directement `commandes#p2024` ou `commandes#p2025`. C'est le serveur qui se charge de router chaque ligne vers la bonne partition, en fonction d'une *expression de partitionnement* définie à la création de la table.

Cette section pose les fondations du partitionnement dans MariaDB : sa raison d'être, son fonctionnement interne, les types disponibles, ainsi que les prérequis et limitations qu'il faut absolument connaître avant de partitionner une table de production. Les sous-sections suivantes détailleront chaque méthode de partitionnement (RANGE, LIST, HASH/KEY) puis le mécanisme d'élagage de partitions (*partition pruning*) qui en constitue le principal levier de performance.

## Partitionnement horizontal, vertical et sharding

Le mot « partitionnement » recouvre plusieurs notions qu'il convient de distinguer.

Le **partitionnement horizontal** est celui dont il est question ici : on répartit les *lignes* d'une table entre plusieurs partitions selon la valeur d'une ou plusieurs colonnes. Chaque partition contient la totalité des colonnes, mais seulement une fraction des lignes. C'est la seule forme prise en charge nativement par la clause `PARTITION BY` de MariaDB.

Le **partitionnement vertical** sépare au contraire les *colonnes* : on éclate une table large en plusieurs tables reliées par une clé commune, par exemple pour isoler des colonnes peu consultées ou volumineuses (BLOB, TEXT). MariaDB ne propose pas de syntaxe dédiée pour cela ; c'est une décision de conception de schéma que l'on met en œuvre à la main.

Le **sharding** (ou distribution horizontale) pousse la logique du partitionnement horizontal jusqu'à répartir les données sur *plusieurs serveurs* physiques distincts. Le partitionnement natif, lui, reste cantonné à un seul serveur : toutes les partitions vivent dans la même instance MariaDB. Pour distribuer réellement les données entre plusieurs nœuds, on s'oriente vers le moteur Spider, ColumnStore ou une architecture applicative dédiée — sujets traités au §15.11 (Sharding et distribution horizontale).

## Pourquoi partitionner ?

Le partitionnement répond à trois grandes catégories de besoins, qu'il est utile de garder à l'esprit pour ne pas l'employer à mauvais escient.

Le bénéfice le plus recherché est la **performance des requêtes** grâce à l'élagage de partitions. Lorsqu'une requête comporte un filtre sur la colonne de partitionnement, l'optimiseur peut déterminer que seules certaines partitions sont susceptibles de contenir des résultats et ignorer purement et simplement les autres. Sur une table de plusieurs centaines de millions de lignes partitionnée par année, une requête ne portant que sur l'année courante ne lira qu'une seule partition au lieu de balayer l'ensemble. Ce mécanisme est détaillé au §15.9.4.

Vient ensuite la **facilité de maintenance et d'archivage**. Supprimer une année entière de données historiques au moyen d'un `DELETE` classique sur une table monolithique est une opération coûteuse, longue et génératrice de journaux. Sur une table partitionnée, `ALTER TABLE … DROP PARTITION` retire la partition correspondante de façon quasi instantanée, sans parcourir les lignes une à une. La même logique vaut pour le déplacement de données froides ou l'application d'une politique de rétention : on raisonne par partition, pas par ligne.

Enfin, le partitionnement améliore la **gérabilité des très grandes tables**. Les opérations de maintenance (`OPTIMIZE`, `ANALYZE`, `CHECK`, `REPAIR`, reconstruction d'index) peuvent être ciblées sur une seule partition, ce qui réduit la durée et l'impact des fenêtres de maintenance. Les statistiques sont par ailleurs collectées au niveau de chaque partition, ce qui peut affiner les choix de l'optimiseur.

## Quand le partitionnement n'est pas la bonne réponse

Le partitionnement n'est pas une optimisation universelle, et il est fréquent de le voir appliqué là où il dégrade les performances au lieu de les améliorer. Quelques principes méritent d'être posés d'emblée.

Le partitionnement n'accélère que les requêtes qui *filtrent sur la clé de partitionnement*. Une table partitionnée par date n'apportera aucun gain — voire une légère pénalité — pour des requêtes qui recherchent par identifiant client sans contrainte de date, puisque le serveur devra alors interroger toutes les partitions. Avant de partitionner, il faut donc connaître les schémas d'accès dominants de l'application.

Un partitionnement ne remplace pas une bonne stratégie d'indexation. Sur une table de taille modeste (quelques millions de lignes), un index B-Tree bien conçu est presque toujours préférable et suffisant ; le partitionnement n'a de sens qu'à partir d'un volume où la maintenance et l'élagage deviennent réellement décisifs. Multiplier les partitions sur une petite table ajoute de la complexité sans contrepartie.

Un excès de partitions est par ailleurs contre-productif. Chaque partition consomme des ressources (descripteurs de fichiers, métadonnées, lignes de statistiques), et au-delà de quelques centaines de partitions ouvertes simultanément, les coûts d'ouverture et de planification finissent par l'emporter sur les gains. « Plus de partitions » n'est jamais synonyme de « plus de performance ».

## Comment MariaDB implémente le partitionnement

Le partitionnement repose sur une couche logicielle intermédiaire — le *gestionnaire de partitionnement* (partition handler) — qui s'intercale entre l'analyseur SQL et le moteur de stockage sous-jacent. Concrètement, une table partitionnée se comporte comme un objet logique unique au niveau du dictionnaire de données, mais chaque partition est stockée et gérée par le moteur de stockage choisi (InnoDB dans l'immense majorité des cas).

Trois conséquences importantes en découlent. D'abord, **le moteur de stockage doit prendre en charge le partitionnement** : c'est le cas d'InnoDB, d'Aria, de MyISAM ou d'Archive, mais pas de tous les moteurs spécialisés. Ensuite, **toutes les partitions d'une même table doivent utiliser le même moteur** ; on ne peut pas mélanger InnoDB et Aria au sein d'une table partitionnée. Enfin, l'expression de partitionnement est évaluée à chaque insertion ou modification pour déterminer la partition cible, ce qui suppose qu'elle soit déterministe et peu coûteuse.

Dans les versions modernes de MariaDB, le support du partitionnement est intégré au serveur et activé par défaut ; il n'y a plus de variable `have_partitioning` ni de plugin à charger explicitement. On peut vérifier la présence du composant via `SHOW PLUGINS`, où `partition` apparaît comme moteur de stockage actif.

## Les types de partitionnement

MariaDB propose plusieurs méthodes de partitionnement, qui se distinguent par la façon dont une ligne est affectée à une partition. Le tableau suivant en donne une vue d'ensemble ; chaque méthode est ensuite approfondie dans les sous-sections dédiées.

| Méthode | Principe d'affectation | Cas d'usage typique |
|---------|------------------------|---------------------|
| `RANGE` | La ligne tombe dans la partition dont l'intervalle de valeurs la contient | Données chronologiques (par mois, par année), rétention par tranches |
| `LIST` | La ligne est affectée selon une liste explicite de valeurs discrètes | Catégories fermées : régions, statuts, codes pays |
| `HASH` | Une fonction de hachage appliquée à une expression répartit uniformément les lignes | Répartition équilibrée sans logique métier particulière |
| `KEY` | Comme `HASH`, mais MariaDB choisit lui-même la fonction de hachage interne | Répartition équilibrée sur une ou plusieurs colonnes, y compris non entières |

À ces quatre méthodes s'ajoutent plusieurs variantes utiles. Les formes **`RANGE COLUMNS`** et **`LIST COLUMNS`** permettent de partitionner directement sur des colonnes de types variés (chaînes, dates, plusieurs colonnes combinées) sans exiger une expression renvoyant un entier. Les formes **`LINEAR HASH`** et **`LINEAR KEY`** emploient un algorithme de hachage linéaire, qui accélère certaines opérations d'ajout ou de fusion de partitions au prix d'une répartition parfois moins uniforme. Enfin, le partitionnement **`SYSTEM_TIME`** est spécifiquement destiné aux tables temporelles à versionnement système, abordées au §18.2.

> **Note sur `PARTITION BY KEY` en MariaDB 12.x** : la série 12.x introduit de nouveaux algorithmes de hachage pour le partitionnement par clé. Ce point, ainsi que ses implications, sont traités au §15.9.3 (HASH partitioning).

## Le sous-partitionnement (partitionnement composite)

MariaDB autorise le *sous-partitionnement*, c'est-à-dire la subdivision de chaque partition en sous-partitions. Seules les partitions définies par `RANGE` ou `LIST` peuvent être sous-partitionnées, et la sous-partition doit obligatoirement utiliser `HASH` ou `KEY`. On obtient ainsi un partitionnement composite, par exemple « par année, puis réparti uniformément en quatre par hachage », qui combine la lisibilité du découpage chronologique et l'équilibrage du hachage. Le sous-partitionnement reste une fonctionnalité de niche, à réserver aux très grands volumes où le partitionnement à un seul niveau ne suffit plus à maîtriser la taille des partitions.

## Prérequis et règles fondamentales

Une règle structurante gouverne tout partitionnement et constitue la principale source d'erreurs : **toutes les colonnes utilisées dans l'expression de partitionnement doivent faire partie de chaque clé primaire et de chaque clé unique de la table**. Cette contrainte découle du fait que l'unicité doit pouvoir être garantie partition par partition, sans recherche globale entre toutes les partitions.

En pratique, cela oblige souvent à intégrer la colonne de partitionnement dans une clé primaire composite. L'exemple suivant illustre l'erreur la plus courante et sa correction :

```sql
-- ÉCHEC : la clé primaire (id) ne contient pas la colonne
-- de partitionnement (created_at)
CREATE TABLE commandes (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    created_at  DATE NOT NULL,
    montant     DECIMAL(10,2)
) ENGINE = InnoDB
PARTITION BY RANGE ( YEAR(created_at) ) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- CORRECT : created_at est intégrée à la clé primaire composite
CREATE TABLE commandes (
    id          BIGINT AUTO_INCREMENT,
    created_at  DATE NOT NULL,
    montant     DECIMAL(10,2),
    PRIMARY KEY (id, created_at)
) ENGINE = InnoDB
PARTITION BY RANGE ( YEAR(created_at) ) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

Au-delà de cette règle, l'expression de partitionnement doit renvoyer un entier dans le cas des méthodes `RANGE`, `LIST` et `HASH` classiques. Les variantes `COLUMNS` lèvent cette contrainte en acceptant directement des colonnes de types non entiers. L'expression doit également être déterministe : elle ne peut pas dépendre de fonctions dont le résultat varie d'un appel à l'autre.

## Limitations à connaître

Plusieurs limitations doivent être prises en compte avant de partitionner une table.

La plus structurante concerne les **clés étrangères** : une table partitionnée ne peut ni contenir de contrainte de clé étrangère, ni être référencée par une clé étrangère d'une autre table. Cette restriction s'applique aux tables InnoDB et constitue souvent le point bloquant principal dans un schéma fortement relationnel. Il faut alors arbitrer entre intégrité référentielle assurée par le SGBD et bénéfices du partitionnement, ou reporter la vérification d'intégrité au niveau applicatif.

Une table peut comporter au maximum **8192 partitions**, sous-partitions comprises (cette limite était de 1024 jusqu'à MariaDB 10.0.3). En pratique, on reste très en deçà de ce plafond, le bon sens en matière de performance imposant des ordres de grandeur bien plus modestes.

Les requêtes portant sur une table partitionnée **ne sont jamais parallélisées** entre partitions : même lorsqu'une requête concerne plusieurs partitions, MariaDB les traite séquentiellement. Le partitionnement réduit le volume de données à parcourir, mais n'introduit pas de parallélisme d'exécution.

Les **index FULLTEXT** ne sont pas pris en charge sur les tables partitionnées, et il n'est pas possible de partitionner une **table temporaire**. Concernant les colonnes de type géométrique (`GEOMETRY` et types spatiaux dérivés), la situation a évolué : depuis MariaDB 11.3.2 — et donc en 12.3 — il est désormais possible de partitionner des tables contenant de telles colonnes, ce qui n'était pas le cas dans les versions antérieures.

Enfin, le cache de requêtes ne tient pas compte du partitionnement : la modification d'une partition invalide les entrées de cache relatives à l'ensemble de la table. Ce point reste largement théorique puisque le query cache est déprécié (voir §15.3). On notera par ailleurs qu'une mise à jour peut s'avérer plus lente sur une table partitionnée lorsque `binlog_format = ROW`, en comparaison de la même opération sur une table non partitionnée.

## Syntaxe générale

La déclaration d'un partitionnement s'ajoute à l'instruction `CREATE TABLE` au moyen de la clause `PARTITION BY`, placée après la définition des colonnes et le choix du moteur. Le squelette général prend la forme suivante :

```sql
CREATE TABLE nom_table (
    -- définition des colonnes et des contraintes
    ...
) ENGINE = InnoDB
PARTITION BY { RANGE | LIST | HASH | KEY } ( expression_ou_colonne )
[ PARTITIONS n ]
(
    PARTITION p0 ... ,
    PARTITION p1 ... ,
    ...
);
```

La clause `PARTITIONS n` précise le nombre de partitions pour les méthodes `HASH` et `KEY`, où il n'est pas nécessaire de nommer chaque partition individuellement. À l'inverse, `RANGE` et `LIST` exigent la définition explicite de chaque partition avec son intervalle ou sa liste de valeurs. Les paramètres propres à chaque méthode seront présentés dans les sous-sections correspondantes.

Il est également possible de partitionner une table existante après coup, ou de modifier son schéma de partitionnement, au moyen de `ALTER TABLE … PARTITION BY …`. Cette opération réécrit la table entière et peut donc être coûteuse sur de gros volumes ; elle relève des stratégies de maintenance avancées abordées plus loin.

## Inspecter et gérer les partitions

Plusieurs outils permettent d'examiner l'état d'une table partitionnée. L'instruction `SHOW CREATE TABLE` restitue la définition complète, clause de partitionnement comprise, ce qui constitue souvent le moyen le plus direct de vérifier la structure en place :

```sql
SHOW CREATE TABLE commandes\G
```

Pour une vue plus analytique, la table `information_schema.PARTITIONS` expose une ligne par partition, avec sa méthode, son expression, le nombre estimé de lignes et l'espace occupé :

```sql
SELECT partition_name,
       partition_method,
       partition_expression,
       table_rows
FROM   information_schema.PARTITIONS  
WHERE  table_schema = DATABASE()  
  AND  table_name   = 'commandes';
```

Pour vérifier que l'élagage de partitions fonctionne réellement sur une requête donnée, on utilise `EXPLAIN PARTITIONS` (ou la colonne dédiée du plan d'exécution), qui indique quelles partitions seront effectivement consultées. Ce diagnostic est central et fait l'objet d'un traitement approfondi au §15.9.4.

Côté maintenance, `ALTER TABLE` offre un ensemble d'opérations dédiées : ajout (`ADD PARTITION`), suppression (`DROP PARTITION`), fusion ou redéfinition (`REORGANIZE PARTITION`), réduction du nombre de partitions de hachage (`COALESCE PARTITION`), vidage rapide d'une partition (`TRUNCATE PARTITION`), ainsi que les variantes ciblées des commandes de maintenance (`ANALYZE`, `CHECK`, `OPTIMIZE`, `REPAIR`, `REBUILD PARTITION`). L'échange de données entre une partition et une table autonome (`EXCHANGE PARTITION`) et la conversion entre partition et table relèvent quant à eux de la gestion avancée des partitions, présentée au §15.10.

## En résumé

Le partitionnement est un découpage horizontal d'une table en partitions physiques gérées comme une table logique unique, dont les bénéfices — élagage à la lecture, maintenance et archivage par bloc, gérabilité des très grandes tables — ne se concrétisent qu'à condition d'aligner le schéma de partitionnement sur les schémas d'accès réels de l'application. Sa mise en œuvre est encadrée par des règles strictes, au premier rang desquelles l'inclusion obligatoire de la clé de partitionnement dans toute clé unique, et par des limitations significatives, notamment l'incompatibilité avec les clés étrangères.

Les sous-sections suivantes détaillent désormais chaque méthode de partitionnement, en commençant par le partitionnement par intervalles (`RANGE`), de loin le plus répandu pour les données chronologiques.

⏭️ [RANGE partitioning](/15-performance-tuning/09.1-range-partitioning.md)

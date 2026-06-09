🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15. Performance et Tuning

> **Partie 7 — Performance et Tuning (profil DBA avancé).** Ce chapitre suppose une bonne maîtrise des [index](../05-index-et-performance/README.md), des [transactions](../06-transactions-et-concurrence/README.md), des [moteurs de stockage](../07-moteurs-de-stockage/README.md) et de l'[administration](../11-administration-configuration/README.md). Version de référence : **MariaDB 12.3 LTS**, avec rappels comparatifs vers la 11.8 lorsque c'est utile.

La performance n'est pas une fonctionnalité que l'on active : c'est le résultat d'un ensemble de décisions cohérentes prises à chaque couche de la pile. Une même requête lente peut trouver son origine dans un index manquant, un buffer pool sous-dimensionné, un disque saturé, un plan d'exécution sous-optimal retenu par l'optimiseur, ou simplement un schéma mal adapté à la volumétrie. L'enjeu de ce chapitre est d'outiller le DBA pour identifier *où* se situe réellement le goulot d'étranglement, puis pour agir avec la bonne technique, au bon endroit.

Le fil conducteur tient en une règle simple mais exigeante : **mesurer avant d'optimiser**. Modifier une variable de configuration « parce qu'on l'a lue sur un blog » est l'erreur la plus répandue et la plus coûteuse. Chaque réglage doit partir d'une observation chiffrée — issue de `EXPLAIN` / `ANALYZE` (vus au chapitre 5), du slow query log ou de Performance Schema — et son effet doit être vérifié de la même manière. L'optimisation est un cycle : observer, formuler une hypothèse, agir sur un seul paramètre à la fois, mesurer de nouveau.

Il est utile de garder à l'esprit que le tuning se joue sur plusieurs couches successives : le matériel et le système d'exploitation, la configuration du serveur, la conception du schéma et l'indexation, l'écriture des requêtes, et enfin le comportement de l'application. Ce chapitre se concentre sur les couches que le DBA contrôle directement côté serveur — configuration des ressources, scalabilité du schéma et réglage de l'optimiseur — en s'appuyant sur les fondamentaux d'indexation déjà couverts au chapitre 5.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez en mesure de :

- structurer une démarche d'optimisation méthodique, fondée sur la mesure plutôt que sur l'intuition ;
- dimensionner les ressources critiques (buffer pool InnoDB, mémoire, E/S disque) selon le profil de charge ;
- diagnostiquer et hiérarchiser les requêtes lentes à l'aide du slow query log, de Performance Schema et d'outils dédiés ;
- mettre en œuvre le partitionnement et les stratégies de distribution horizontale pour les grandes volumétries ;
- influencer finement les choix de l'optimiseur grâce au nouveau moteur d'**Optimizer Hints** introduit dans la série 12.x ;
- évaluer objectivement l'effet d'un réglage par le benchmarking.

## Vue d'ensemble du chapitre

Le chapitre s'ouvre sur une [méthodologie d'optimisation](01-methodologie-optimisation.md) (§15.1) qui pose le cadre de travail et les réflexes à acquérir avant toute intervention.

Vient ensuite le réglage des ressources. La [configuration mémoire](02-configuration-memoire.md) (§15.2) traite le dimensionnement du buffer pool InnoDB, ses instances et le key buffer hérité de MyISAM. On y rattache une mise au point sur le [Query Cache et les raisons de sa dépréciation](03-query-cache-deprecie.md) (§15.3), la [configuration des I/O et des disques](04-configuration-io-disques.md) (§15.4) — `innodb_io_capacity`, `innodb_flush_method`, optimisations SSD —, l'[optimisation du moteur InnoDB](05-optimisation-innodb.md) (§15.5), la construction d'index accélérée via [`innodb_alter_copy_bulk`](06-innodb-alter-copy-bulk.md) (§15.6) et l'[Adaptive Hash Index](13-adaptive-hash-index.md) (§15.13).

Le diagnostic occupe une place centrale : [analyse des requêtes lentes](07-analyse-requetes-lentes.md) (§15.7) avec le slow query log et `pt-query-digest`, exploration via [Performance Schema et le sys schema](08-performance-schema-sys.md) (§15.8), et mesure contrôlée par le [benchmarking](12-benchmarking.md) (§15.12) à l'aide de sysbench et mysqlslap.

Pour les fortes volumétries, le chapitre aborde le [partitionnement de tables](09-partitionnement-tables.md) (§15.9) — RANGE, LIST, HASH (avec les nouveaux algorithmes `PARTITION BY KEY`) et partition pruning —, la [gestion avancée des partitions](10-gestion-avancee-partitions.md) (§15.10) et le [sharding et la distribution horizontale](11-sharding-distribution.md) (§15.11).

Enfin, le réglage de l'optimiseur lui-même : le [cost-based optimizer](14-cost-based-optimizer-ssd.md) (§15.14), désormais sensible aux caractéristiques des SSD, et surtout le nouveau moteur d'[Optimizer Hints](15-optimizer-hints.md) (§15.15) 🆕, qui permet un contrôle fin et par requête du plan d'exécution. Le chapitre se clôt sur les [optimisations de la recherche vectorielle](16-vector-optimisations.md) (§15.16) 🆕 apportées par la 12.3.

## Nouveautés MariaDB 12.3 abordées dans ce chapitre 🆕

- **Optimizer Hints (§15.15)** — la fonctionnalité phare. MariaDB introduit une syntaxe d'indications normalisée, placée sous forme de commentaire dans la requête (`/*+ ... */`), pour orienter l'optimiseur sans toucher aux variables globales : nommage des blocs de requête (`QB_NAME`), hints de table et d'index, contrôle de l'ordre de jointure, traitement des sous-requêtes, et limitation du temps d'exécution avec `MAX_EXECUTION_TIME`.
- **Nouveaux algorithmes pour `PARTITION BY KEY` (§15.9.3)** — une répartition des lignes améliorée pour le partitionnement par clé.
- **Optimisations de la recherche vectorielle (§15.16)** — estimation de distance par extrapolation et calcul rapproché du moteur de stockage, pour des recherches HNSW plus efficaces.

> **À noter.** Aucun réglage n'est universel : les valeurs « idéales » dépendent du matériel, du volume de données et du profil de charge (OLTP, OLAP, mixte). Les configurations de référence par cas d'usage sont rassemblées en [Annexe D](../annexes/d-configuration-reference/README.md), et une checklist de performance en [Annexe E](../annexes/e-checklist-performance/README.md).

⏭️ [Méthodologie d'optimisation](/15-performance-tuning/01-methodologie-optimisation.md)

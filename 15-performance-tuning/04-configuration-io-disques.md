🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.4 Configuration I/O et disques

Les entrées/sorties disque (E/S, ou *I/O*) constituent le goulot d'étranglement le plus fréquent d'une base de données. Tandis que la puissance des processeurs et la capacité mémoire ont explosé, l'accès au stockage est resté, comparativement, l'opération la plus lente du système — même à l'ère des SSD. Une base de données passe l'essentiel de son temps à déplacer des données entre le disque et la mémoire ; configurer correctement le sous-système d'E/S est donc déterminant pour la performance globale.

Cette section pose les fondamentaux nécessaires pour comprendre *comment* MariaDB sollicite le stockage, *quelles* métriques observer, et *pourquoi* certains réglages existent. Les paramètres de réglage proprement dits sont ensuite détaillés dans les sous-sections : `innodb_io_capacity` (§15.4.1), `innodb_flush_method` (§15.4.2) et les optimisations spécifiques aux disques flash (§15.4.3).

---

## Pourquoi les E/S disque conditionnent la performance

La hiérarchie mémoire d'un serveur s'étale sur plusieurs ordres de grandeur de latence : un accès à la RAM se compte en nanosecondes, un accès à un SSD NVMe en dizaines de microsecondes, et un accès à un disque mécanique en millisecondes. Autrement dit, lire une donnée sur un disque dur peut être des dizaines de milliers de fois plus lent que la lire en mémoire.

Le rôle d'un SGBD est précisément de masquer cet écart en maintenant en mémoire (dans l'**InnoDB Buffer Pool**, cf. §7.2.2 et §15.2.1) les données les plus sollicitées. Tant que l'ensemble de travail (*working set*) tient en RAM, les E/S disque restent marginales. Mais dès que ce volume dépasse la mémoire disponible, le serveur doit lire et écrire sur le stockage, et c'est alors la performance du disque qui dicte celle de la base.

Deux objectifs complémentaires guident donc la configuration des E/S :

- **réduire le volume d'E/S** inutiles (bon dimensionnement du Buffer Pool, indexation pertinente, requêtes optimisées) ;
- **rendre les E/S inévitables aussi efficaces que possible**, en alignant la configuration de MariaDB sur les capacités réelles du matériel.

C'est ce second objectif qui fait l'objet de cette section.

---

## Comment InnoDB génère des E/S

### Les lectures

Lorsqu'une requête a besoin d'une page de données absente du Buffer Pool — un défaut de cache (*cache miss*) —, InnoDB doit la lire depuis le disque. Ces lectures sont le plus souvent **aléatoires** : elles ciblent des pages dispersées dans les fichiers de données. Le Buffer Pool constitue ainsi la première ligne de défense contre les E/S : plus son taux de succès est élevé, moins le disque est sollicité en lecture.

### Les écritures : le principe du Write-Ahead Logging

Le chemin d'écriture d'InnoDB repose sur la journalisation à écriture anticipée (*Write-Ahead Logging*, WAL), un mécanisme essentiel à comprendre car il explique la plupart des réglages d'E/S. Lorsqu'une transaction modifie des données :

1. La page concernée est modifiée **en mémoire**, dans le Buffer Pool ; elle devient une page « sale » (*dirty page*) qui diffère désormais de sa version sur disque.
2. La modification est consignée dans le **redo log** (journal de reprise), par une écriture **séquentielle**. Au `COMMIT`, et avec le réglage par défaut `innodb_flush_log_at_trx_commit = 1`, le redo log est synchronisé sur disque (`fsync`) : c'est cette opération qui garantit la **durabilité** (le « D » de ACID, cf. chapitre 6 et §7.2.3).
3. Les pages sales, elles, ne sont **pas** réécrites dans les fichiers de données au moment du commit. Elles le sont **plus tard**, de façon asynchrone, par le **flush d'arrière-plan** (*background flushing*). Ce sont ces écritures-là qui sont aléatoires et coûteuses.

Ce découplage est l'idée centrale : la durabilité est assurée par une écriture **séquentielle et rapide** dans le redo log, tandis que les écritures **aléatoires et coûteuses** vers les fichiers de données sont différées et regroupées en arrière-plan. C'est précisément la raison pour laquelle la latence du périphérique hébergeant le redo log, ainsi que le réglage du rythme de flush d'arrière-plan, ont un impact si important sur la performance — ce dernier point motivant l'existence du paramètre `innodb_io_capacity` (§15.4.1).

Deux mécanismes complètent ce chemin d'écriture :

- le **doublewrite buffer**, qui écrit d'abord les pages dans une zone contiguë avant de les placer à leur emplacement définitif, afin de se prémunir contre les écritures partielles (*torn pages*) en cas de coupure ; il accroît le volume écrit mais protège l'intégrité des données ;
- le **checkpointing**, qui s'assure périodiquement que les pages sales antérieures à une certaine position du journal ont bien été écrites, libérant ainsi l'espace du redo log pour qu'il puisse être réutilisé de façon cyclique.

À noter : depuis MariaDB 10.5, il n'existe qu'un seul fichier de redo log (`ib_logfile0`), dont la taille est fixée par `innodb_log_file_size` et qui peut être redimensionné dynamiquement depuis la 10.9. Le dimensionnement de ce journal redevient un sujet de premier plan avec le nouveau binlog 12.3 (voir plus bas).

### Les différents flux d'E/S

Au-delà des fichiers de données et du redo log, plusieurs flux d'écriture coexistent et se disputent le sous-système de stockage :

- les **fichiers de données** (`.ibd` avec `innodb_file_per_table`), cibles des écritures aléatoires de flush ;
- le **redo log**, écriture séquentielle critique pour la durabilité ;
- les **undo logs**, nécessaires au MVCC et aux rollbacks ;
- le **doublewrite buffer**, décrit ci-dessus ;
- le **binary log**, utilisé pour la réplication et la restauration à un instant donné (PITR), cf. §11.5 ;
- les **fichiers et tables temporaires**, produits notamment par les tris et les tables dérivées (dont la consommation peut être bornée par `max_tmp_total_space_usage`, cf. §11.8) ;
- les **journaux** d'erreur, de requêtes lentes et général.

Comprendre que ces flux partagent le même budget d'E/S aide à choisir leur répartition sur les périphériques et à dimensionner la configuration.

---

## E/S séquentielles vs aléatoires

La distinction entre E/S séquentielles et aléatoires est fondamentale, car elle conditionne le comportement du matériel.

Les **E/S séquentielles** (redo log, binary log, lecture de grandes plages contiguës) accèdent à des emplacements adjacents : même un disque mécanique les traite efficacement, car la tête de lecture n'a pas à se repositionner. Les **E/S aléatoires** (flush des pages sales vers les fichiers de données, lectures sur défaut de cache) ciblent des emplacements dispersés et imposent, sur un disque dur, de coûteux déplacements mécaniques.

C'est là que le type de stockage change tout : un disque mécanique est lourdement pénalisé par les E/S aléatoires, tandis qu'un SSD, dépourvu de pièces mobiles, efface presque entièrement l'écart entre accès séquentiel et accès aléatoire. Cette différence explique pourquoi la migration vers le stockage flash transforme radicalement le profil de performance d'une base de données.

---

## Le matériel de stockage : quelques repères

Les ordres de grandeur suivants permettent de situer les capacités d'un périphérique de stockage. Ce sont des repères indicatifs, à confirmer par mesure sur le matériel réel (le réglage fin propre aux SSD est traité au §15.4.3).

| Type de stockage | E/S aléatoires (IOPS) | Latence typique | Débit séquentiel |
|---|---|---|---|
| Disque mécanique (7200 tr/min) | ~75 – 200 | ~5 – 10 ms | ~100 – 200 Mo/s |
| SSD SATA | ~10 000 – 100 000 | ~0,1 – 0,5 ms | ~500 Mo/s |
| SSD NVMe | ~200 000 – 1 000 000+ | ~10 – 100 µs | plusieurs Go/s |

Au-delà du périphérique lui-même, deux points d'architecture méritent attention :

- **RAID** : pour une base de données, le RAID 10 est généralement préféré pour son équilibre performance/redondance. Les niveaux RAID 5 et 6 souffrent d'une pénalité d'écriture (lecture-modification-écriture de la parité) qui pénalise les charges transactionnelles. Un contrôleur doté d'un cache protégé par batterie (*BBU*) améliore nettement les écritures.
- **Stockage réseau et cloud** (par exemple les volumes type EBS ou Premium SSD) : la performance dépend du niveau d'IOPS provisionné, et le cache d'écriture du système d'exploitation ou de l'hyperviseur peut fausser les mesures. Un *benchmark* d'E/S sur un disque cloud partagé donne souvent des résultats trompeurs : il convient de tester sur un volume dédié et représentatif de la production.

---

## L'impact du nouveau binlog InnoDB (MariaDB 12.3)

Une évolution architecturale majeure de MariaDB 12.3 modifie sensiblement le chemin d'écriture. Traditionnellement, le binary log et le moteur InnoDB étaient deux composants distincts, dont la cohérence exigeait un coûteux **commit en deux phases** (*2PC*) et une **double synchronisation** sur disque à chaque commit — une pour les données InnoDB, une pour les événements de réplication.

En 12.3, le binary log peut être **stocké directement dans des tablespaces gérés par InnoDB** (fichiers `binlog-NNNNNN.ibb`). Cette intégration **élimine le 2PC** et réduit d'environ moitié le nombre de synchronisations au commit, d'où un gain d'écriture important. MariaDB annonce jusqu'à **~4× en écriture** sur les charges fortement transactionnelles ; des évaluations indépendantes mesurent plutôt un gain de **2,4 à 3,3× en TPS** à durabilité maximale, avec une latence p99 deux fois moindre sous forte concurrence. Le binlog devient en outre intrinsèquement *crash-safe*, même avec un `innodb_flush_log_at_trx_commit` relâché (0 ou 2).

La contrepartie côté E/S est une **amplification du redo log** (environ 2×) : il faut prévoir un `innodb_log_file_size` plus généreux. Cette fonctionnalité est optionnelle et s'active par configuration (`binlog_storage_engine = innodb`). Le détail de son fonctionnement, de ses prérequis (notamment GTID) et de ses implications pour la réplication est traité au §11.5.4.

Pour la configuration des E/S, la conséquence pratique est double : le coût du commit s'allège nettement, mais le **dimensionnement du redo log gagne en importance** dès lors que cette option est activée.

---

## Système de fichiers et organisation des données

Le système de fichiers sous-jacent influe également sur les performances. **XFS** et **ext4** sont les choix recommandés sous Linux pour MariaDB. L'option de montage `noatime` évite des écritures de métadonnées inutiles à chaque lecture de fichier.

Sur du matériel à disques distincts, **répartir les flux d'E/S** sur plusieurs périphériques (par exemple isoler le redo log, le binary log ou les fichiers temporaires des fichiers de données) peut réduire la contention. Sur du stockage NVMe rapide et unifié, ce cloisonnement perd cependant beaucoup de son intérêt : la capacité d'E/S y est suffisamment élevée pour absorber tous les flux. L'usage de `innodb_file_per_table` (un fichier `.ibd` par table) reste par ailleurs la norme, pour la souplesse de gestion de l'espace qu'il procure.

---

## Méthodologie : mesurer avant de régler

Comme pour toute optimisation (cf. §15.1), un réglage d'E/S ne doit jamais se faire à l'aveugle : il faut d'abord **mesurer** le comportement réel du système.

Du côté du système d'exploitation, les outils standards permettent d'observer la pression sur le stockage : `iostat` (avec notamment le taux d'utilisation `%util`, le temps d'attente `await`, les IOPS et le débit), `vmstat` et `pidstat`. Une file d'attente longue et un `%util` proche de 100 % signalent un disque saturé.

Du côté de MariaDB, plusieurs sources de métriques renseignent sur l'activité d'E/S : la commande `SHOW ENGINE INNODB STATUS`, les variables d'état `Innodb_data_reads`, `Innodb_data_writes`, `Innodb_os_log_written`, ainsi que les tables de `performance_schema` et du schéma `sys`. Le suivi des attentes liées au Buffer Pool et au flush oriente les décisions de réglage.

La règle d'or est de comparer la **capacité mesurée du matériel** aux paramètres de MariaDB, afin de les aligner. C'est exactement l'objet du premier paramètre étudié dans la suite, `innodb_io_capacity`.

---

## Feuille de route de la section

Les sous-sections suivantes déclinent ces principes en réglages concrets :

- **§15.4.1 — `innodb_io_capacity`** : indiquer à InnoDB la capacité d'E/S du stockage, afin de calibrer le rythme du flush d'arrière-plan.
- **§15.4.2 — `innodb_flush_method`** : choisir la manière dont InnoDB ouvre et synchronise ses fichiers, et gérer l'interaction avec le cache du système d'exploitation.
- **§15.4.3 — Optimisations SSD modernes** : tirer parti des spécificités du stockage flash (NVMe, parallélisme, absence de pénalité sur les accès aléatoires).

⏭️ [innodb_io_capacity](/15-performance-tuning/04.1-innodb-io-capacity.md)

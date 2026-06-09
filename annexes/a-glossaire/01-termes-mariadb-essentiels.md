🔝 Retour au [Sommaire](/SOMMAIRE.md)

# A.1 — Termes MariaDB essentiels

> 🔤 Définitions des concepts de fond de la formation, classées par ordre alphabétique.

Cette section définit les termes conceptuels que l'on rencontre tout au long de la formation. Les sigles « purs » (FK, PK, CTE, GTID, SST, IST…) sont regroupés à part dans la section [A.2 — Acronymes courants](02-acronymes-courants.md). Chaque entrée renvoie au chapitre où la notion est approfondie, et le marqueur 🆕 signale un terme ou un comportement introduit dans les versions récentes de MariaDB (de la 11.6 à la 12.x) et présent dans la **12.3 LTS** de référence.

---

## A

**ACID** — Ensemble des quatre propriétés qui garantissent la fiabilité d'une transaction : **A**tomicité (tout ou rien), **C**ohérence (la base passe d'un état valide à un autre), **I**solation (les transactions concurrentes n'interfèrent pas) et **D**urabilité (une transaction validée survit aux pannes). InnoDB est le moteur ACID de référence de MariaDB. *Voir Ch. 6.*

**Aria** — Moteur de stockage non transactionnel développé par MariaDB comme successeur de MyISAM. Il en partage la vocation (rapidité de lecture, simplicité) tout en ajoutant la résistance aux crashs (*crash-safe*). Il est notamment utilisé pour les tables système et les tables temporaires internes. 🆕 La série 12.x ajoute un cache de clés segmenté (`aria_pagecache_segments`). *Voir Ch. 7.*

## B

**B-Tree (arbre B équilibré)** — Structure de données arborescente et équilibrée qui sert de base à la grande majorité des index MariaDB. Elle maintient les clés triées et garantit un coût logarithmique en recherche, insertion et suppression, ce qui la rend efficace aussi bien pour les égalités que pour les plages de valeurs et les tris. *Voir Ch. 5.*

**Binary log (journal binaire, ou binlog)** — Journal qui enregistre, sous forme d'événements, toutes les modifications de données et de schéma. Il est au cœur de la réplication et du *point-in-time recovery*. Trois formats existent : STATEMENT, ROW et MIXED. 🆕 La série 12.x ajoute une nouvelle implémentation **optionnelle** intégrant le binlog à InnoDB (à activer via `binlog_storage_engine=innodb`) ; la suppression de la synchronisation entre journaux améliore alors le débit en écriture (de l'ordre de 4×). *Voir Ch. 11.*

**Buffer Pool (InnoDB Buffer Pool)** — Zone mémoire dans laquelle InnoDB met en cache les pages de données et d'index. C'est le levier de performance le plus important du moteur : plus le jeu de données « chaud » tient en mémoire, moins MariaDB sollicite le disque. Dimensionné via `innodb_buffer_pool_size`. *Voir Ch. 7 et Ch. 15.*

## C

**Certification-based replication** — Mécanisme de réplication employé par Galera Cluster : au moment du COMMIT, chaque nœud certifie la transaction en vérifiant l'absence de conflit avec les *write sets* concurrents avant de l'appliquer sur l'ensemble du cluster. C'est ce qui rend la réplication synchrone et virtuellement sans perte. *Voir Ch. 14.*

**Charset & Collation (jeu de caractères et collation)** — Le jeu de caractères définit l'encodage des chaînes — `utf8mb4` par défaut depuis la 11.8, couvrant l'intégralité d'Unicode, emojis compris — tandis que la collation définit les règles de tri et de comparaison associées (collations UCA 14.0.0). *Voir Ch. 11.*

**ColumnStore** — Moteur de stockage orienté colonnes destiné aux charges analytiques (OLAP) et au *data warehousing*. En stockant les données par colonne plutôt que par ligne, il accélère massivement les agrégations sur de grands volumes, au prix d'écritures unitaires plus coûteuses. *Voir Ch. 7 et Ch. 20.*

**Curseur (cursor)** — Objet permettant de parcourir ligne à ligne le résultat d'une requête au sein d'une procédure ou d'une fonction stockée. 🆕 La série 12.x ajoute les curseurs sur *prepared statements* et le type SYS_REFCURSOR (compatibilité Oracle). *Voir Ch. 8.*

## D

**Deadlock (interblocage)** — Situation où deux transactions (ou plus) s'attendent mutuellement pour libérer des verrous, formant un cycle d'attente que rien ne peut dénouer. InnoDB détecte automatiquement les deadlocks et annule l'une des transactions (la « victime ») pour débloquer les autres. *Voir Ch. 6.*

## E

**Event (événement planifié)** — Tâche SQL exécutée automatiquement par le serveur selon une planification ponctuelle ou récurrente, à la manière d'un *cron* interne à la base. Sa prise en charge dépend de l'Event Scheduler. *Voir Ch. 8.*

**EXPLAIN (plan d'exécution)** — Commande qui révèle la stratégie retenue par l'optimiseur pour exécuter une requête : ordre des tables, index utilisés, type d'accès, estimation du nombre de lignes. En MariaDB, l'instruction **`ANALYZE`** (et non `EXPLAIN ANALYZE`, propre à MySQL) exécute la requête et y ajoute les mesures réelles d'exécution. C'est l'outil de base du diagnostic de performance. *Voir Ch. 5.*

## F

**Failover & Switchover (bascule)** — Procédures de promotion d'un réplica au rang de primaire. Le *failover* est subi (le primaire est tombé) ; le *switchover* est planifié (bascule maîtrisée, par exemple pour une maintenance). Le recours au GTID en facilite grandement l'automatisation. *Voir Ch. 13 et Ch. 14.*

**Fonction stockée (stored function)** — Routine SQL nommée qui retourne une valeur et peut être appelée directement dans une expression (`SELECT`, `WHERE`, etc.). À la différence d'une procédure, elle est conçue pour produire un résultat plutôt que pour enchaîner une suite d'actions. *Voir Ch. 8.*

**Full-text (index plein texte)** — Type d'index spécialisé dans la recherche de mots et d'expressions à l'intérieur de colonnes textuelles, avec gestion de la pertinence, par opposition à la recherche par préfixe qu'offre un index B-Tree. *Voir Ch. 5.*

## G

**Galera Cluster** — Solution de haute disponibilité offrant une réplication synchrone multi-maître : tous les nœuds acceptent les écritures et partagent un état cohérent grâce à la certification des transactions. Elle élimine le décalage de réplication et le risque de perte de données, au prix de contraintes sur la latence réseau et la gestion des conflits d'écriture. 🆕 En 12.3, Galera est livré dans un paquet distinct (`mariadb-server-galera`), la dépendance ayant été retirée du serveur. *Voir Ch. 14.*

## I

**Index** — Structure auxiliaire accélérant la localisation des lignes correspondant à un critère, à la manière de l'index d'un livre. Il accélère les lectures mais alourdit légèrement les écritures et occupe de l'espace disque. *Voir Ch. 5.*

**Index couvrant (covering index)** — Index contenant toutes les colonnes nécessaires à une requête, ce qui permet d'y répondre sans accéder à la table elle-même (*index-only scan*). *Voir Ch. 5.*

**InnoDB** — Moteur de stockage par défaut de MariaDB. Transactionnel et conforme ACID, il gère le verrouillage au niveau de la ligne (*row-level locking*), les clés étrangères et le MVCC. C'est le choix de référence pour les charges transactionnelles (OLTP). *Voir Ch. 7.*

**Isolation (niveau d')** — Degré auquel une transaction est protégée des effets des transactions concurrentes. MariaDB propose quatre niveaux normalisés — READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ (défaut d'InnoDB) et SERIALIZABLE — qui arbitrent entre cohérence et concurrence. 🆕 `innodb_snapshot_isolation` est activé par défaut **depuis la 11.6.2** (donc déjà en 11.8), ce qui modifie le comportement de REPEATABLE READ. *Voir Ch. 6.*

## J

**JSON** — Type de données et ensemble de fonctions permettant de stocker et de manipuler des documents semi-structurés directement en SQL. 🆕 La série 12.x ajoute notamment le prédicat standard `IS JSON` et lève la limite de profondeur. *Voir Ch. 4.*

## M

**Mariabackup** — Outil de sauvegarde physique « à chaud » (*hot backup*) de MariaDB, dérivé de Percona XtraBackup. Il copie les fichiers de données sans interrompre le service et prend en charge les sauvegardes complètes comme incrémentales. *Voir Ch. 12.*

**MaxScale** — Proxy de base de données intelligent développé par MariaDB. Il assure la répartition de charge, la séparation lecture/écriture (*read/write split*), le routage de requêtes et le failover automatique en s'intercalant entre les applications et le cluster. *Voir Ch. 14.*

**Moteur de stockage (storage engine)** — Composant enfichable (*pluggable*) responsable de la façon dont les données d'une table sont physiquement stockées, indexées et verrouillées. MariaDB permet de choisir le moteur table par table : InnoDB (par défaut), Aria, MyISAM, ColumnStore, Memory, Spider, etc. (La recherche vectorielle n'est pas un moteur, mais un type `VECTOR` doté d'un index HNSW au sein des moteurs existants — *voir l'entrée « VECTOR »*.) *Voir Ch. 7.*

**MVCC (Multi-Version Concurrency Control)** — Technique de contrôle de concurrence par laquelle InnoDB conserve plusieurs versions d'une même ligne. Les lecteurs voient une version cohérente sans bloquer les écrivains, et inversement, ce qui maximise la concurrence. Le mécanisme s'appuie sur les *undo logs*. *Voir Ch. 6.*

**MyISAM** — Ancien moteur de stockage par défaut de MySQL/MariaDB, non transactionnel, dépourvu de clés étrangères et verrouillant au niveau de la table. Considéré comme *legacy*, il subsiste surtout par compatibilité ; Aria et InnoDB lui sont préférés. *Voir Ch. 7.*

## O

**Optimiseur (optimizer)** — Composant du serveur qui transforme une requête SQL en un plan d'exécution efficace, en estimant le coût des différentes stratégies (choix des index, ordre des jointures, etc.). MariaDB utilise un optimiseur basé sur les coûts (*cost-based*), désormais sensible aux spécificités des SSD. *Voir Ch. 5 et Ch. 15.*

**Optimizer Hints** 🆕 — Directives insérées dans une requête sous forme de commentaires `/*+ ... */` pour influencer finement les choix de l'optimiseur : forcer ou interdire un index, fixer l'ordre des jointures, limiter le temps d'exécution (`MAX_EXECUTION_TIME`), etc. Il s'agit de l'une des fonctionnalités phares de la série 12.x. *Voir Ch. 15.*

## P

**Partitionnement (partitioning)** — Découpage d'une grande table en sous-ensembles physiques (partitions) selon une clé (RANGE, LIST, HASH, KEY), tout en la présentant comme une seule table logique. Il autorise l'élagage des partitions (*partition pruning*) et facilite la gestion des données historiques. *Voir Ch. 15.*

**Procédure stockée (stored procedure)** — Routine SQL nommée regroupant une suite d'instructions, invoquée par `CALL`. Elle accepte des paramètres IN, OUT et INOUT et permet de déporter de la logique côté serveur. *Voir Ch. 8.*

## Q

**Quorum** — Majorité de nœuds qu'un cluster Galera doit réunir pour rester opérationnel et accepter les écritures. C'est le rempart contre le *split-brain* : une partition minoritaire se met d'elle-même hors service. *Voir Ch. 14.*

## R

**Redo log** — Journal de reprise d'InnoDB enregistrant les modifications physiques des pages avant leur écriture définitive sur disque. Il garantit la durabilité (le « D » d'ACID) : après un crash, InnoDB rejoue le *redo log* pour retrouver un état cohérent. *Voir Ch. 7.*

**Réplication** — Mécanisme de copie des données d'un serveur (primaire/source) vers un ou plusieurs autres (réplicas). Elle peut être asynchrone (par défaut), semi-synchrone ou synchrone (Galera), et sert la haute disponibilité, la répartition de charge en lecture et la sauvegarde. *Voir Ch. 13.*

**Rôle (role)** — Ensemble nommé de privilèges que l'on peut attribuer en bloc à un ou plusieurs utilisateurs, ce qui simplifie la gestion des droits. *Voir Ch. 10.*

## S

**Schéma (schema)** — En MariaDB, synonyme de base de données : un conteneur logique regroupant tables, vues, routines et autres objets. *Voir Ch. 2.*

**Sequence** — Objet générant des suites de valeurs numériques uniques (`CREATE SEQUENCE`), indépendant des tables et plus souple que l'AUTO_INCREMENT (pas d'incrément paramétrable, bornes minimale et maximale, cycle). *Voir Ch. 18.*

**Snapshot isolation** 🆕 — Isolation par instantané : une transaction travaille sur une vue figée des données prise à son démarrage. `innodb_snapshot_isolation` est activé par défaut **depuis la 11.6.2** (donc déjà en 11.8), ce qui durcit REPEATABLE READ en signalant les conflits d'écriture (erreur `1020`) plutôt que de les ignorer silencieusement. *Voir Ch. 6.*

**Split-brain** — Défaillance d'un cluster où une partition réseau scinde les nœuds en groupes qui se croient chacun seuls actifs, au risque d'écritures divergentes. Le quorum est le mécanisme qui le prévient. *Voir Ch. 14.*

## T

**Tables temporelles (system-versioned tables)** — Tables qui conservent automatiquement l'historique des versions de chaque ligne, ce qui permet d'interroger l'état de la base à un instant passé (`AS OF`). *Voir Ch. 18.*

**Thread Pool** — Mécanisme qui mutualise un nombre limité de threads pour servir de nombreuses connexions, par opposition au modèle « un thread par connexion ». Il améliore la stabilité sous forte concurrence. *Voir Ch. 11 et Ch. 18.*

**Transaction** — Unité logique de travail regroupant une ou plusieurs instructions SQL, exécutée selon le principe du tout ou rien et délimitée par `START TRANSACTION` / `COMMIT` / `ROLLBACK`. C'est le support des propriétés ACID. *Voir Ch. 6.*

**Trigger (déclencheur)** — Routine exécutée automatiquement en réaction à un événement de modification (INSERT, UPDATE, DELETE) sur une table, avant (BEFORE) ou après (AFTER) celui-ci. 🆕 La série 12.x autorise un trigger unique couvrant plusieurs événements. *Voir Ch. 8.*

## U

**Undo log** — Journal d'annulation d'InnoDB conservant les versions précédentes des lignes modifiées. Il sert à la fois au ROLLBACK et au MVCC, en fournissant aux lecteurs des versions cohérentes. *Voir Ch. 7.*

## V

**VECTOR & MariaDB Vector** — Type de données stockant des vecteurs (*embeddings*) et fonctionnalité de recherche par similarité associée, reposant sur un index HNSW et des fonctions de distance (euclidienne, cosinus). C'est le socle des cas d'usage d'IA : RAG, recherche sémantique, recommandation. *Voir Ch. 18 et Ch. 20.*

**Verrou (lock) & verrouillage** — Mécanisme garantissant qu'une ressource (ligne, table) n'est pas modifiée de façon incohérente par des transactions concurrentes. InnoDB privilégie le verrouillage fin au niveau de la ligne ; on distingue les verrous partagés (lecture) des verrous exclusifs (écriture). *Voir Ch. 6.*

**Vue (view)** — Table virtuelle définie par une requête `SELECT` mémorisée. Elle ne stocke pas de données propres mais restitue, à la demande, le résultat de sa requête sous-jacente — utile pour simplifier, réutiliser ou sécuriser l'accès aux données. *Voir Ch. 9.*

**Vue matérialisée (materialized view)** — Vue dont le résultat est physiquement stocké et rafraîchi périodiquement. MariaDB ne la propose pas en natif ; on l'émule par des tables alimentées via triggers ou events. *Voir Ch. 9.*

## W

**Window function (fonction de fenêtrage)** — Fonction qui calcule une valeur sur un ensemble de lignes liées à la ligne courante (la « fenêtre ») sans les regrouper, contrairement à `GROUP BY`. Elle permet rangs, moyennes mobiles et cumuls (ROW_NUMBER, RANK, LAG, LEAD, etc.). *Voir Ch. 4.*

---

⬅️ [Annexe A — Glossaire (présentation)](README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [A.2 — Acronymes courants](02-acronymes-courants.md)

⏭️ [Acronymes courants (FK, PK, CTE, GTID, SST, IST, etc.)](/annexes/a-glossaire/02-acronymes-courants.md)

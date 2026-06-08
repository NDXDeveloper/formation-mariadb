🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.8 Thread Pool avancé

Cette section approfondit le **thread pool** de MariaDB, dont les fondamentaux sont posés en §11.10. On y revient brièvement sur le principe, avant de se concentrer sur les **leviers de réglage** et les considérations opérationnelles propres aux fortes concurrences.

## Du « un thread par connexion » au pool de threads

Par défaut, MariaDB fonctionne en mode **`one-thread-per-connection`** : chaque connexion cliente se voit attribuer un thread système dédié pendant toute sa durée de vie. Ce modèle est efficace jusqu'à quelques centaines de connexions, mais se dégrade au-delà : des milliers de threads engendrent des changements de contexte coûteux, une consommation mémoire importante (une pile par thread) et de la contention. Le débit finit par *chuter* à mesure que la concurrence augmente.

Le **thread pool** découple les connexions des threads. Un nombre restreint de threads de travail dessert un grand nombre de connexions : un thread exécute une requête, puis revient dans le pool pour en servir une autre. Le nombre de threads *réellement actifs* reste ainsi proche du nombre de cœurs, ce qui limite les changements de contexte et permet d'absorber des dizaines de milliers de connexions. Le bénéfice est maximal pour les charges **OLTP faites de nombreuses requêtes courtes** issues de connexions intermittentes — le profil typique des applications web adossées à un pool de connexions.

Le thread pool de MariaDB est **intégré au serveur communautaire** (il s'appuie sur un mécanisme événementiel sous Unix et sur le pool natif du système sous Windows).

> **À ne pas confondre** avec le *connection pooling* applicatif (§17.2) : ce dernier réutilise les connexions *côté client*, tandis que le thread pool gère, *côté serveur*, la façon dont ces connexions se partagent les threads d'exécution. Les deux sont complémentaires.

## Activer le thread pool

Le mode de gestion des connexions se choisit au démarrage :

```ini
[mariadb]
thread_handling = pool-of-threads
```

## Les leviers de réglage

C'est ici que se situe la dimension « avancée ». Quelques variables gouvernent le comportement du pool.

**`thread_pool_size`** fixe le nombre de **groupes de threads**, qui détermine grosso modo le niveau de parallélisme. Sa valeur par défaut est le nombre de cœurs du serveur — un bon point de départ que l'on ajuste rarement à la baisse.

**`thread_pool_stall_limit`** est le délai (en millisecondes, ~500 par défaut) au-delà duquel un groupe considère que le thread en cours d'exécution est **bloqué** (sur une E/S ou un verrou) et réveille ou crée un thread supplémentaire pour ne pas affamer les autres connexions. Une valeur basse réagit vite aux blocages mais multiplie les threads ; une valeur haute économise les threads au risque d'allonger la latence en cas de blocage.

**`thread_pool_oversubscribe`** contrôle le degré de **sur-souscription** par groupe — combien de threads peuvent s'exécuter au-delà de la cible nominale (3 par défaut). On l'augmente quand les requêtes font beaucoup d'E/S (les threads passent du temps à attendre), on le réduit pour rester au plus près d'un thread par cœur.

**`thread_pool_max_threads`** plafonne le **nombre total** de threads que le pool peut créer. Élevé par défaut, il sert de garde-fou contre une explosion du nombre de threads lorsque beaucoup d'entre eux se retrouvent bloqués.

**`thread_pool_idle_timeout`** est la durée (60 s par défaut) après laquelle un thread de travail inactif est retiré, ce qui permet au pool de **se rétracter** une fois le pic de charge passé.

Une configuration typique pour un serveur OLTP très connecté ressemble à :

```ini
[mariadb]
thread_handling           = pool-of-threads
thread_pool_size          = 16     # ≈ nombre de cœurs
thread_pool_max_threads   = 2000   # plafond de sécurité
thread_pool_stall_limit   = 500    # ms avant de réagir à un blocage
thread_pool_oversubscribe = 3
thread_pool_idle_timeout  = 60     # s avant de retirer un thread inactif
```

## Prioriser les connexions

Le thread pool sait accorder un traitement préférentiel à certaines connexions via `thread_pool_priority`, qui accepte `auto`, `high` ou `low`. Une connexion prioritaire est ordonnancée plus favorablement au sein de son groupe :

```sql
SET SESSION thread_pool_priority = 'high';
```

Cela permet, par exemple, de favoriser des transactions critiques par rapport à des tâches de fond.

## Garder une porte d'entrée : `extra_port`

Un risque, lorsque le pool est saturé ou que de nombreux threads sont bloqués, est de ne plus pouvoir s'y connecter — y compris pour diagnostiquer le problème. MariaDB prévoit pour cela un **port d'administration de secours**, doté de son propre mode « un thread par connexion » et de son propre quota de connexions :

```ini
[mariadb]
extra_port            = 3307
extra_max_connections = 5
```

Un administrateur conserve ainsi un accès garanti pour observer et débloquer l'instance, même quand le pool principal est sous pression.

## Quand le thread pool aide, et quand être prudent

Le thread pool **brille** face à un très grand nombre de connexions exécutant des requêtes courtes : il évite l'effondrement de débit du mode « un thread par connexion » et stabilise les performances sous forte concurrence.

Il apporte en revanche **peu de bénéfice — voire complique les choses — dans certains cas**. Pour un faible nombre de connexions, le mode par défaut reste plus simple et parfois marginalement plus rapide. Surtout, une charge dominée par des **requêtes longues** ou de **longs blocages** (verrous tenus longtemps) demande un réglage attentif de `thread_pool_stall_limit`, `thread_pool_oversubscribe` et `thread_pool_max_threads`, faute de quoi le pool peut sérialiser les traitements de façon inattendue. La règle est de **mesurer** sur une charge représentative avant de généraliser.

## Surveiller le thread pool

MariaDB expose l'état interne du pool via plusieurs vues d'`INFORMATION_SCHEMA` — `THREAD_POOL_GROUPS`, `THREAD_POOL_QUEUES`, `THREAD_POOL_STATS` et `THREAD_POOL_WAITS` — qui renseignent sur les groupes, les connexions en file d'attente et les causes d'attente :

```sql
SELECT * FROM information_schema.THREAD_POOL_GROUPS;
```

Les variables de statut de la famille `Threadpool%` (dont `Threadpool_threads`, le nombre de threads actuellement dans le pool, et `Threadpool_idle_threads`) complètent ce suivi :

```sql
SHOW STATUS LIKE 'Threadpool%';
```

## Points clés à retenir

- Le thread pool **découple connexions et threads** : un petit nombre de threads sert de nombreuses connexions, ce qui stabilise le débit sous forte concurrence (s'active par `thread_handling = pool-of-threads`).
- Principaux réglages : `thread_pool_size` (≈ cœurs), `thread_pool_stall_limit` (réaction aux blocages), `thread_pool_oversubscribe` (sur-souscription), `thread_pool_max_threads` (plafond), `thread_pool_idle_timeout` (rétractation).
- `thread_pool_priority` priorise des connexions ; `extra_port` garantit un **accès d'administration de secours** quand le pool est saturé.
- Idéal pour **beaucoup de connexions à requêtes courtes** ; à régler avec soin (ou à éviter) pour des requêtes **longues/bloquantes** ou un faible nombre de connexions.
- À distinguer du **connection pooling applicatif** (§17.2), qui agit côté client.
- Surveiller via les vues `INFORMATION_SCHEMA.THREAD_POOL_*` et les statuts `Threadpool%`.

⏭️ [Dynamic columns](/18-fonctionnalites-avancees/09-dynamic-columns.md)

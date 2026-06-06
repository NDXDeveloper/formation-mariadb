🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.10 — Thread Pool et gestion de la concurrence

La gestion de la concurrence concerne la manière dont le serveur traite de nombreuses connexions simultanées. Le modèle par défaut — un thread du système d'exploitation par connexion — est simple et efficace sous charge modérée, mais montre ses limites lorsque le nombre de connexions actives devient très élevé. Le **Thread Pool** de MariaDB répond à ce besoin en découplant le nombre de connexions du nombre de threads d'exécution. Cette section présente les deux modèles, l'architecture du Thread Pool sur Unix et Windows, sa configuration, les situations où il apporte un gain (et celles où il en apporte peu), et sa relation avec le *connection pooling* applicatif (voir [17.2](../17-integration-developpement/02-connection-pooling.md)).

---

## Le modèle par défaut : un thread par connexion

Par défaut sur Unix, MariaDB fonctionne en mode `one-thread-per-connection` : chaque connexion cliente se voit attribuer un thread dédié du système d'exploitation, qui l'accompagne pendant toute sa durée de vie. Ce modèle offre une **faible latence par requête** et une bonne isolation entre connexions ; il convient parfaitement à un nombre de connexions faible à modéré.

Son coût apparaît à grande échelle. Avec plusieurs centaines de connexions actives, le système doit gérer autant de threads : le **surcoût de commutation de contexte** augmente, l'ordonnanceur du noyau est sollicité, et chaque thread consomme de la mémoire (pile d'exécution et buffers par session). Au-delà d'un certain seuil, le débit cesse de croître puis se dégrade, le serveur passant plus de temps à arbitrer entre threads qu'à exécuter des requêtes. Le cache de threads (`thread_cache_size`) atténue le coût d'ouverture/fermeture des connexions, mais ne réduit pas le nombre de threads **actifs** simultanément, qui est la véritable source du problème.

---

## Le Thread Pool : principe

Le mode `pool-of-threads` substitue à ce modèle un **ensemble borné de threads de travail** qui servent un nombre bien plus grand de connexions. Le nombre de threads n'est plus indexé sur le nombre de connexions : il reste maîtrisé, ce qui maintient une utilisation du CPU stable même lorsque les connexions affluent.

La contrepartie est que les requêtes peuvent être **mises en file d'attente** : lorsque tous les threads disponibles sont occupés, une nouvelle requête attend qu'un thread se libère. C'est un changement de comportement important — on y revient plus bas. Le Thread Pool s'active via la variable `thread_handling`, dont les valeurs possibles sont :

- `one-thread-per-connection` — le modèle par défaut sur Unix ;
- `pool-of-threads` — le Thread Pool ;
- `no-threads` — un thread unique pour toutes les connexions, réservé au débogage.

> 🔔 À noter : **sur Windows, le Thread Pool est le mode par défaut** ; c'est sur Unix qu'il faut explicitement l'activer. Sur les très anciennes versions de Windows où il n'est pas implémenté, le serveur bascule silencieusement en `one-thread-per-connection`.

---

## Architecture sur Unix : les groupes de threads

L'implémentation Unix repose sur des **groupes de threads** (*thread groups*) : les connexions sont réparties en ensembles indépendants de threads. La variable maîtresse est `thread_pool_size`, qui définit **le nombre de groupes** et détermine donc combien d'instructions peuvent s'exécuter **simultanément**. Sa valeur par défaut est le **nombre de cœurs CPU** de la machine, ce qui constitue un bon point de départ.

Au sein de chaque groupe, on distingue :

- un **thread d'écoute** (*listener*), qui surveille les événements d'I/O et distribue le travail ;
- un ou plusieurs **threads de travail** (*workers*), dont, en règle générale, un seul est actif à la fois par groupe.

Un mécanisme essentiel garantit la robustesse de l'ensemble : la **détection de blocage** (*stall detection*). Un thread minuteur surveille les groupes ; s'il détecte qu'un groupe est bloqué — typiquement parce qu'un worker exécute une requête longue ou attend un verrou — il réveille ou crée un thread supplémentaire pour que ce blocage ne fige pas le groupe entier. La sensibilité de ce mécanisme est réglée par `thread_pool_stall_limit` (exprimé en millisecondes, de l'ordre d'une demi-seconde par défaut). La variable `thread_pool_oversubscribe` autorise par ailleurs quelques threads actifs concurrents par groupe.

---

## Architecture sur Windows

Sur Windows, le Thread Pool s'appuie sur l'API native `CreateThreadpool` du système. Le dimensionnement se fait par `thread_pool_min_threads` (nombre minimal de threads, par défaut 1, utile pour les charges très irrégulières) et `thread_pool_max_threads` (plafond). Comme indiqué plus haut, c'est le mode par défaut sur cette plateforme.

---

## Les variables de configuration

Les principales variables de réglage du Thread Pool sont les suivantes :

| Variable | Rôle | Remarques |
|---|---|---|
| `thread_handling` | Sélectionne le modèle (`pool-of-threads`, `one-thread-per-connection`, `no-threads`) | À définir **au démarrage** dans le fichier de configuration ; non modifiable à chaud |
| `thread_pool_size` | Nombre de groupes de threads (Unix) = parallélisme | Défaut : nombre de cœurs CPU ; modifiable à chaud (s'applique aux nouvelles connexions, non persistant) |
| `thread_pool_max_threads` | Plafond du nombre total de threads | Garde-fou contre une création incontrôlée de threads lors de blocages ; le total peut légèrement dépasser ce plafond, chaque groupe nécessitant au moins deux threads |
| `thread_pool_stall_limit` | Délai de détection de blocage d'un groupe | En millisecondes |
| `thread_pool_idle_timeout` | Délai avant qu'un worker inactif ne se termine | Défaut : 60 s ; à abaisser pour des charges brèves ou en rafales |
| `thread_pool_oversubscribe` | Threads actifs concurrents tolérés par groupe | Unix |
| `thread_pool_min_threads` | Nombre minimal de threads | Windows |
| `thread_pool_priority`, `thread_pool_prio_kickup_timer` | Ordonnancement par priorité des connexions | Voir le Thread Pool avancé ([18.8](../18-fonctionnalites-avancees/08-thread-pool-avance.md)) |

Une configuration typique sur un serveur Unix à 16 cœurs :

```ini
[mariadb]
thread_handling          = pool-of-threads
thread_pool_size         = 16
thread_pool_max_threads  = 500
thread_pool_idle_timeout = 60
```

`thread_pool_size` peut être ajusté dynamiquement, le changement ne s'appliquant qu'aux nouvelles connexions :

```sql
SET GLOBAL thread_pool_size = 24;
```

> ⚠️ Lorsque le Thread Pool est actif, la variable `thread_cache_size` n'est **pas utilisée** et la variable d'état `Threads_cached` reste à 0. Le cache de threads est une notion propre au modèle un-thread-par-connexion.

---

## Le port d'administration (`extra_port`)

Le Thread Pool souffre d'un risque opérationnel historique : si tous les threads de travail sont bloqués par des requêtes longues ou non coopératives, plus aucune nouvelle connexion ne peut être établie — y compris celle de l'administrateur qui voudrait diagnostiquer et tuer les requêtes fautives. La détection de blocage atténue ce risque, mais ne l'élimine pas dans les cas extrêmes.

La parade consiste à configurer un **port d'administration distinct** (`extra_port`, avec `extra_max_connections`). Même lorsque le pool principal est entièrement saturé par des requêtes non coopératives, on peut s'y connecter pour se connecter au serveur et exécuter `KILL` sur les requêtes bloquantes. Configurer ce port est une **bonne pratique systématique** dès lors que l'on active le Thread Pool en production.

---

## Quand utiliser le Thread Pool — et quand s'en abstenir

Le Thread Pool apporte un gain mesurable lorsque :

- le nombre de connexions est **élevé** (de plusieurs centaines à plusieurs milliers) ;
- la charge est dominée par des **requêtes courtes** (profil OLTP), avec peu de verrous de table ou de ligne bloquants ;
- la dégradation observée provient précisément de la multiplication des threads (commutation de contexte, pression mémoire).

Il faut en revanche se montrer prudent, voire l'éviter, lorsque :

- la charge est dominée par **quelques requêtes longues ou analytiques** : une requête lente monopolise un groupe, et la mise en file d'attente pénalise alors les autres ;
- l'application **exige** que les requêtes triviales se terminent toujours instantanément quelle que soit la charge : la mise en file d'attente rompt cette garantie, et même un `SELECT 1` peut subir un délai sous forte charge ;
- un *pooler* de connexions externe **plafonne déjà strictement** la concurrence à un petit nombre fixe : le Thread Pool n'apporte alors que peu.

Sur le plan conceptuel, le Thread Pool optimise le **débit agrégé** du serveur, parfois au prix de la latence d'une requête individuelle en période de saturation. C'est un compromis à assumer en connaissance de cause.

---

## Thread Pool et connection pooling : deux mécanismes distincts

Une confusion fréquente consiste à assimiler le Thread Pool au *connection pooling*. Ce sont deux mécanismes complémentaires qui agissent à des niveaux différents :

- le **connection pooling** (côté application, ou via ProxySQL — voir [17.2](../17-integration-developpement/02-connection-pooling.md) et [14.9](../14-haute-disponibilite/09-proxysql-haproxy.md)) réutilise un ensemble limité de **connexions**, évitant le coût d'ouverture/fermeture répété et bornant le nombre de connexions qui atteignent le serveur ;
- le **Thread Pool** gère, lui, le nombre de threads qui **s'exécutent** à l'intérieur du serveur, indépendamment du nombre de connexions.

Les deux ne s'excluent pas. En amont, un *pooler* réduit le va-et-vient des connexions ; en aval, le Thread Pool maîtrise la concurrence d'exécution. Lorsqu'un *pooler* externe limite déjà rigoureusement la concurrence à un petit nombre, l'apport du Thread Pool diminue. À l'inverse, dans les environnements où les clients sont nombreux et indépendants — microservices, fonctions serverless — et où le nombre de connexions est élevé et difficile à borner, le Thread Pool côté serveur est précisément l'outil adapté.

---

## Supervision

L'état du Thread Pool se suit via les variables d'état dédiées, notamment `Threadpool_threads` (nombre de threads dans le pool) et `Threadpool_idle_threads` (threads inactifs) :

```sql
SHOW GLOBAL STATUS LIKE 'Threadpool%';
```

En complément, `Threads_running` (voir [11.9](09-monitoring-metriques.md)) reste la jauge de référence de la concurrence réelle. Les signes d'un pool mal dimensionné sont une latence de mise en file d'attente persistante (requêtes simples anormalement lentes sous charge) ou un nombre de threads proche de `thread_pool_max_threads`. Le réglage fin, l'ordonnancement par priorité et les fonctionnalités avancées sont développés dans la section [18.8](../18-fonctionnalites-avancees/08-thread-pool-avance.md).

---

En définitive, la gestion de la concurrence consiste à **accorder le modèle de threads à la nature de la charge** : le modèle un-thread-par-connexion pour les charges modérées ou dominées par des requêtes longues, le Thread Pool pour les charges à forte concurrence et requêtes courtes — l'un comme l'autre s'articulant avec une stratégie de *connection pooling* adaptée en amont ([17.2](../17-integration-developpement/02-connection-pooling.md)).

⏭️ [Charset par défaut : utf8mb4 avec collations UCA 14.0.0 (depuis 11.8)](/11-administration-configuration/11-charset-utf8mb4-uca14.md)

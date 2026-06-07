🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.2 Configuration mémoire

Après le matériel, la mémoire est le levier le plus puissant à la disposition du DBA. La RAM est plusieurs ordres de grandeur plus rapide que le disque, même sur SSD : une donnée déjà présente en mémoire est servie sans la moindre E/S. Régler la mémoire de MariaDB revient donc, pour l'essentiel, à décider *ce qui mérite d'y rester* — et à le faire sans jamais dépasser la mémoire physique de la machine. Cette section pose le cadre général ; le dimensionnement précis du buffer pool, sa répartition en instances et le cas particulier de MyISAM sont traités dans les sous-sections qui suivent.

## Mémoire globale et mémoire par connexion

La première chose à comprendre est qu'il existe deux familles de zones mémoire, dont la logique de dimensionnement diffère radicalement.

Les **buffers globaux** sont alloués une seule fois au démarrage et partagés par toutes les connexions : c'est le cas du buffer pool InnoDB, du key buffer de MyISAM ou du log buffer InnoDB. Leur taille est fixe (ou redimensionnable à chaud) et indépendante du nombre de clients connectés.

Les **buffers par connexion** (ou par session), au contraire, sont alloués pour chaque thread qui en a besoin : `sort_buffer_size`, `join_buffer_size`, `read_buffer_size`, `read_rnd_buffer_size`, ainsi que la mémoire des tables temporaires. Leur danger tient à la multiplication : un buffer de quelques mégaoctets, multiplié par des centaines de connexions simultanées, peut consommer plusieurs gigaoctets. C'est une cause classique d'épuisement mémoire (OOM) en production.

Cette distinction commande une règle de prudence : on peut être généreux avec les buffers globaux, qui ne sont comptés qu'une fois, mais il faut rester mesuré avec les buffers par connexion. Augmenter `sort_buffer_size` à une valeur élevée « pour aller plus vite » est souvent contre-productif, car la mémoire peut être allouée intégralement à chaque usage.

## Le budget mémoire

La mémoire totale réellement consommée par le serveur s'estime ainsi :

> mémoire totale ≈ buffers globaux + (buffers par connexion × connexions simultanées) + marge pour le cache du système d'exploitation + surcoût du serveur

La règle cardinale qui en découle ne souffre aucune exception : **un serveur de base de données ne doit jamais recourir au swap**. Le moindre accès à de la mémoire « paginée » sur disque ruine les bénéfices de la mise en cache et fait s'effondrer les temps de réponse. Il faut donc laisser une marge confortable et ne pas chercher à allouer la totalité de la RAM. Côté système, ce dimensionnement prudent se complète en réduisant la propension du noyau Linux à paginer : une valeur basse de `vm.swappiness` (typiquement `1` plutôt que `0`, afin de garder le swap comme ultime filet de sécurité contre l'*OOM killer*) dissuade le noyau de déplacer vers le disque des pages pourtant utiles, sans pour autant interdire le swap en cas d'urgence.

Cette marge profite par ailleurs au **cache de fichiers du système d'exploitation**, qui constitue un second niveau de cache. Selon le moteur et le mode d'écriture des fichiers (voir [§15.4](04-configuration-io-disques.md)), InnoDB peut contourner ce cache pour ses fichiers de données, tandis que MyISAM s'y appuie entièrement pour les données de tables. Le dimensionnement « idéal » dépend donc aussi du moteur dominant et du profil de charge — les configurations de référence sont rassemblées en [Annexe D](../annexes/d-configuration-reference/README.md).

## Les principales zones mémoire

Au-delà de cette dichotomie, il est utile de cartographier les zones que l'on rencontre le plus souvent :

- **Buffer pool InnoDB** (`innodb_buffer_pool_size`) — de loin la zone la plus importante pour le moteur par défaut : il met en cache les pages de données et d'index. Son dimensionnement et sa répartition font l'objet des [§15.2.1](02.1-innodb-buffer-pool.md) et [§15.2.2](02.2-buffer-pool-instances.md).
- **Log buffer InnoDB** (`innodb_log_buffer_size`) — tampon des écritures du redo log avant leur transfert sur disque ; utile pour absorber les rafales d'écritures et les grosses transactions.
- **Key buffer MyISAM** (`key_buffer_size`) — cache des blocs d'index des tables MyISAM uniquement (les données passent par le cache de l'OS) ; pertinent surtout pour d'éventuelles tables au format MyISAM ([§15.2.3](02.3-key-buffer.md)). Le moteur Aria dispose, lui, de son propre cache de pages (voir le [chapitre 7](../07-moteurs-de-stockage/04-aria.md)).
- **Buffers par session** (`sort_buffer_size`, `join_buffer_size`, `read_buffer_size`, `read_rnd_buffer_size`) — espaces de travail pour les tris, jointures et lectures, alloués par connexion.
- **Tables temporaires** (`tmp_table_size`, `max_heap_table_size`) — taille maximale des tables temporaires conservées en RAM ; au-delà, elles débordent sur disque, ce qui dégrade les performances.
- **Caches d'objets** (`table_open_cache`, `table_definition_cache`, `thread_cache_size`) — mémorisent des descripteurs de tables et des threads réutilisables ; ils évitent des ouvertures et créations répétées, mais consomment peu de mémoire de données.
- **Query cache** — historiquement un cache de résultats de requêtes, aujourd'hui déprécié et désactivé par défaut ; les raisons en sont détaillées en [§15.3](03-query-cache-deprecie.md).

## Observer l'utilisation de la mémoire

On ne règle pas la mémoire à l'aveugle. `SHOW ENGINE INNODB STATUS` expose l'état du buffer pool (pages totales, libres, modifiées, taux de succès). L'instrumentation mémoire de **Performance Schema** et les vues `memory_*` du sys schema détaillent la consommation par composant interne ([§15.8](08-performance-schema-sys.md)). Les compteurs de `SHOW GLOBAL STATUS` — notamment ceux du buffer pool et des tables temporaires créées sur disque — complètent le tableau, et les outils système (`free`, `ps`, `/proc/meminfo`) donnent la vue d'ensemble côté machine, indispensable pour vérifier l'absence de swap.

## Ce que couvrent les sous-sections

La suite entre dans le détail des zones qui pèsent le plus dans un réglage mémoire : le [dimensionnement du buffer pool InnoDB](02.1-innodb-buffer-pool.md) (§15.2.1), sa [répartition en instances](02.2-buffer-pool-instances.md) (§15.2.2) pour limiter la contention, et enfin le [key buffer de MyISAM](02.3-key-buffer.md) (§15.2.3), un héritage à connaître même lorsque l'on travaille exclusivement avec InnoDB.

⏭️ [InnoDB Buffer Pool : Dimensionnement](/15-performance-tuning/02.1-innodb-buffer-pool.md)

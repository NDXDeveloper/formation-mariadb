🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.1 Méthodologie d'optimisation

La méthodologie est ce qui sépare l'optimisation efficace du bricolage. Avant de toucher la moindre variable de configuration ou de réécrire la moindre requête, il faut savoir vers quel objectif on tend, comment on mesure le chemin parcouru, et où agir pour un effet maximal. Cette section décrit une démarche reproductible — applicable quelle que soit la version de MariaDB — dont les sections suivantes du chapitre fourniront les outils détaillés.

## Définir l'objectif avant d'optimiser

« Optimiser » n'a de sens que par rapport à une cible. Avant tout, il faut formuler ce que « performant » signifie pour l'application concernée : une **cible de latence** (par exemple, 95 % des requêtes du panier sous 50 ms), une **cible de débit** (par exemple, 5 000 transactions par seconde en heure de pointe), ou des **contraintes de ressources** (tenir la charge sur le matériel disponible, sans dépasser un budget mémoire ou un coût cloud).

Deux distinctions sont importantes. D'abord, latence et débit ne varient pas toujours dans le même sens : optimiser l'un peut dégrader l'autre, et il faut savoir lequel prime. Ensuite, il vaut mieux raisonner en **percentiles** qu'en moyennes : la moyenne masque la traîne, et c'est souvent la queue de distribution (p95, p99) qui dégrade l'expérience perçue.

Enfin, l'optimisation a une fin. Une fois la cible atteinte, on s'arrête : chercher à gagner quelques millisecondes sur une requête déjà sous l'objectif est rarement rentable. C'est le revers de l'**optimisation prématurée**, contre laquelle Knuth mettait en garde.

## Le cycle d'optimisation

L'optimisation est un processus itératif et discipliné, pas une série d'ajustements au hasard. Chaque itération suit le même cycle :

1. **Établir une référence (baseline).** Mesurer l'état actuel dans des conditions reproductibles : temps de réponse, débit, métriques système. Sans point de départ chiffré, impossible de prouver une amélioration.
2. **Localiser le goulot d'étranglement.** Déterminer *où* le temps est consommé : CPU, E/S disque, attente de verrous, plan d'exécution inefficace, latence réseau ? On ne devine pas, on mesure.
3. **Formuler une hypothèse.** Proposer une cause plausible et l'action correspondante (« cet index manquant force un balayage complet de la table »).
4. **Agir sur une seule variable à la fois.** Un seul changement par itération. Modifier plusieurs paramètres simultanément rend impossible d'attribuer l'effet — bon ou mauvais — à l'un d'eux.
5. **Mesurer de nouveau.** Reproduire exactement la mesure de référence et comparer.
6. **Conserver, annuler, documenter.** Garder le changement s'il rapproche de la cible, l'annuler sinon. Dans tous les cas, consigner ce qui a été testé et le résultat obtenu.

On recommence ensuite sur le prochain goulot d'étranglement, jusqu'à atteindre l'objectif fixé.

## Raisonner par couches

Un problème de performance peut naître à n'importe quel niveau de la pile. Identifier la bonne couche évite de « réparer » au mauvais endroit :

- **Application** — gestion des connexions (pooling), requêtes en rafale (problème N+1), absence de mise en cache, traitements ligne à ligne au lieu de traitements ensemblistes.
- **Conception du schéma et indexation** — types de données inadaptés, absence d'index utile, modèle mal normalisé (ou trop normalisé pour le besoin réel).
- **Écriture des requêtes** — prédicats non sargables, sous-requêtes mal placées, tris ou regroupements évitables.
- **Configuration du serveur** — buffer pool, paramètres d'E/S, journalisation.
- **Matériel et système d'exploitation** — CPU, RAM, type de disque, réseau, réglages noyau.

Règle empirique essentielle : **les plus gros gains se situent généralement dans le haut de cette liste**. Un index manquant peut diviser un temps de réponse par mille ; ajuster une variable serveur dépasse rarement quelques dizaines de pour cent. On commence donc par traquer les requêtes et le schéma, et l'on ne touche au matériel et à la configuration qu'ensuite — d'autant qu'ajouter du matériel ne fait souvent que repousser un problème de conception. Les fondamentaux d'indexation et la lecture des plans d'exécution sont traités au [chapitre 5](../05-index-et-performance/README.md).

## Caractériser la charge de travail

Il n'existe pas de configuration universellement « optimale » : la bonne configuration dépend du profil de charge. Avant de régler quoi que ce soit, il faut le caractériser :

- le type de charge : **OLTP** (nombreuses petites transactions, faible latence) vs **OLAP** (requêtes analytiques lourdes, fort débit de lecture) vs mixte ;
- le ratio lectures/écritures ;
- la taille du *working set* (les données réellement consultées) par rapport à la RAM disponible ;
- le profil temporel : charge régulière ou en pics, présence de traitements batch.

Ces caractéristiques orientent tous les réglages ultérieurs — du dimensionnement du buffer pool au choix d'une stratégie de partitionnement. Des configurations de référence par cas d'usage sont rassemblées en [Annexe D](../annexes/d-configuration-reference/README.md).

## Cibler ce qui compte vraiment

En pratique, une petite fraction des requêtes est responsable de l'essentiel de la charge : c'est la règle des 80/20. L'effort d'optimisation doit se concentrer là.

Le critère pertinent n'est pas le temps d'une requête prise isolément, mais son **temps cumulé** : `fréquence × temps par exécution`. Une requête de 5 ms exécutée des millions de fois pèse souvent bien plus lourd qu'une requête de 2 s lancée une fois par jour. Les outils d'agrégation du slow query log, comme `pt-query-digest`, classent précisément les requêtes par temps total consommé (voir [§15.7](07-analyse-requetes-lentes.md)) : c'est le bon point de départ pour décider quoi optimiser.

## Le panorama des instruments

La mesure repose sur une boîte à outils que les sections suivantes détaillent. À ce stade, il suffit d'en connaître le rôle :

- **`EXPLAIN` (plan estimé) et l'instruction `ANALYZE` (plan réel — l'équivalent MariaDB de l'« `EXPLAIN ANALYZE` » de MySQL)** — révèlent le chemin d'exécution choisi et, pour `ANALYZE`, confrontent les estimations de l'optimiseur aux lignes réellement traitées. Le format `FORMAT=JSON` fournit le détail le plus complet (voir [§5.7](../05-index-et-performance/07-analyse-plans-execution.md)).
- **`optimizer_trace`** — explique *pourquoi* l'optimiseur a retenu tel plan, en exposant son raisonnement de coût.
- **Slow query log** — capture les requêtes dépassant un seuil de durée (`long_query_time`) ou n'utilisant pas d'index ; c'est la base de l'analyse des requêtes lentes ([§15.7](07-analyse-requetes-lentes.md)).
- **Performance Schema et sys schema** — instrumentation fine de l'activité interne : attentes, verrous, E/S, mémoire ([§15.8](08-performance-schema-sys.md)).
- **`SHOW [GLOBAL] STATUS`, `SHOW ENGINE INNODB STATUS`, `SHOW PROCESSLIST`** — clichés rapides de l'état du serveur, des compteurs InnoDB et de l'activité en cours.
- **Outils système** — `top`, `iostat`, `vmstat`, etc., pour distinguer une saturation CPU d'une saturation disque ou réseau.
- **Benchmarking** — `sysbench`, `mysqlslap` : reproduire une charge représentative pour mesurer l'effet d'un changement dans un environnement contrôlé ([§15.12](12-benchmarking.md)).

Sur le plan de la méthode, deux cadres complémentaires aident à structurer le diagnostic : la **méthode USE** (Utilisation, Saturation, Erreurs), orientée ressources système, et l'approche centrée sur le **temps de réponse**, qui consiste à profiler où le temps est réellement passé plutôt qu'à surveiller des taux d'utilisation isolés.

## Les pièges classiques

La plupart des échecs d'optimisation découlent de quelques erreurs récurrentes :

- **Optimiser sans mesurer**, en se fiant à l'intuition ou à une croyance.
- **Copier-coller une configuration** trouvée en ligne, sans rapport avec son matériel ni sa charge.
- **Changer plusieurs paramètres à la fois**, ce qui empêche d'isoler l'effet de chacun.
- **Se fier aux moyennes** au lieu des percentiles, et passer à côté de la traîne.
- **Optimiser le mauvais composant** (ou trop tôt) : soigner une requête marginale pendant que le vrai goulot est ailleurs.
- **Benchmarker sur des données ou un matériel non représentatifs**, et en tirer des conclusions invalides en production.
- **Ignorer la couche applicative**, alors qu'un problème N+1 ou un pooling défaillant annule tout réglage serveur.
- **Confondre symptôme et cause** : un CPU à 100 % est un symptôme, pas un diagnostic.

## Tracer et documenter

Une optimisation non documentée est à moitié perdue : on oublie ce qui a été essayé et l'on ne peut pas revenir en arrière proprement. Il faut tenir un journal des changements (paramètre modifié, valeur avant/après, effet mesuré), versionner les fichiers de configuration et privilégier des modifications réversibles. Gérer la configuration comme du code (*Infrastructure as Code*) facilite ce suivi et la reproductibilité entre environnements — voir le [chapitre 16](../16-devops-automatisation/README.md).

## En résumé

Optimiser, c'est suivre un cycle discipliné : définir une cible, mesurer une référence, localiser le goulot d'étranglement, agir sur une seule variable, mesurer de nouveau, puis recommencer. On raisonne par couches en privilégiant les plus gros leviers (schéma, index et requêtes avant configuration et matériel), on concentre l'effort sur les requêtes au temps cumulé le plus élevé, et l'on documente chaque pas. Les sections suivantes appliquent cette méthode, en commençant par la [configuration mémoire](02-configuration-memoire.md). La [checklist de performance](../annexes/e-checklist-performance/README.md) de l'Annexe E en propose une version condensée et actionnable.

⏭️ [Configuration mémoire](/15-performance-tuning/02-configuration-memoire.md)

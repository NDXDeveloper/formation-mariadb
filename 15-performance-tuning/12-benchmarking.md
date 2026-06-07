🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.12 Benchmarking

Tout au long de ce chapitre, chaque optimisation — réglage mémoire, indexation, partitionnement, choix d'architecture — a été présentée comme un levier de performance. Mais un levier ne vaut que par l'effet qu'il produit réellement, et cet effet ne se présume pas : il se mesure. Le benchmarking est la discipline qui consiste à **mesurer systématiquement les performances** d'une base de données afin de répondre à une question précise : tel changement améliore-t-il vraiment les performances pour telle charge de travail, et de combien ?

C'est le contrepoint indispensable de l'optimisation. Optimiser à l'intuition, sans mesure, conduit aussi souvent à dégrader qu'à améliorer ; le benchmarking transforme une conviction en fait vérifié. Cette section pose les principes et la méthodologie du benchmarking ; les sous-sections suivantes présenteront les deux outils de référence, sysbench (§15.12.1) et mysqlslap (§15.12.2).

## Pourquoi benchmarker ?

Le benchmarking répond à plusieurs besoins concrets. Le premier est la **validation d'une optimisation** : avant et après un changement de configuration, l'ajout d'un index ou une montée de version, on mesure pour confirmer le gain — c'est ainsi que l'on vérifierait, par exemple, l'amélioration annoncée du nouveau binlog intégré à InnoDB en 12.x (§11.5.4) plutôt que de la tenir pour acquise. Le deuxième est le **dimensionnement** (capacity planning) : déterminer la charge qu'un système peut absorber avant saturation. Le troisième est la **détection de régressions** : s'assurer qu'une mise à jour ou une modification n'a pas ralenti le système. Le dernier est la **comparaison** : opposer deux moteurs, deux configurations, deux générations de matériel ou deux versions de MariaDB sur une base objective.

## Types de benchmarks

On distingue d'abord les benchmarks **synthétiques** des benchmarks **applicatifs**. Les premiers, produits par des outils comme sysbench ou mysqlslap, exécutent une charge standardisée : ils sont reproductibles et comparables d'un système à l'autre, mais ne reflètent pas nécessairement le profil réel d'une application donnée. Les seconds rejouent une charge réelle — par exemple au moyen de la capture et du rejeu de charge de MaxScale (Workload Capture/Replay, §14.5), ou à partir des requêtes réelles analysées par pt-query-digest (§15.7.2). Ils collent à la réalité, au prix d'une préparation et d'un contrôle plus délicats.

On distingue ensuite les **micro-benchmarks**, qui isolent une seule opération — une requête précise, un index particulier — des **macro-benchmarks**, qui éprouvent le système entier sous une charge mixte réaliste. Les deux ont leur usage : le micro-benchmark diagnostique un point précis, le macro-benchmark évalue le comportement d'ensemble.

Le profil de charge importe enfin autant que l'outil. Une charge **OLTP** (§20.1) — nombreuses transactions courtes et rapides — ne se mesure pas comme une charge **OLAP** — quelques requêtes analytiques lourdes. Le benchmark doit reproduire le profil que l'on cherche à caractériser, sous peine de mesurer une réalité sans rapport avec la production visée.

## Les métriques clés

Quatre familles de métriques structurent toute analyse de performance. Le **débit** (throughput) mesure la quantité de travail par unité de temps, exprimée en transactions par seconde (TPS) ou en requêtes par seconde (QPS) ; c'est l'indicateur de capacité. La **latence** (ou temps de réponse) mesure le temps consacré à chaque requête ou transaction ; elle ne doit jamais être réduite à sa seule moyenne, car la moyenne masque la traîne. Ce sont les **percentiles** — médiane (p50), p95, p99 — qui révèlent l'expérience réelle : un p99 élevé signifie qu'un utilisateur sur cent subit une lenteur marquée, fait qu'une moyenne flatteuse dissimulerait.

La **concurrence** — le nombre de clients ou de threads simultanés — est la troisième dimension : les performances doivent être mesurées à un niveau de concurrence réaliste, et l'on observe comment débit et latence évoluent à mesure qu'elle augmente. Typiquement, le débit croît avec la concurrence jusqu'à un point de saturation, au-delà duquel il plafonne tandis que la latence s'envole. L'**utilisation des ressources**, enfin — processeur, entrées/sorties, mémoire, réseau —, permet d'identifier le goulet d'étranglement responsable de cette saturation.

## Méthodologie d'un benchmark rigoureux

Un benchmark utile commence par une **question claire** : que cherche-t-on à mesurer, et pour quelle décision ? Sans objectif défini, les chiffres produits n'éclairent rien.

L'exigence centrale est ensuite de **reproduire les conditions de production** aussi fidèlement que possible. Cela suppose un matériel comparable, la même version et la même configuration de MariaDB, mais surtout un **volume de données réaliste** : un benchmark sur dix mille lignes, qui tiennent intégralement dans le buffer pool, ne dit rien du comportement d'une table de cent millions de lignes sur laquelle les entrées/sorties disque deviennent déterminantes. La distribution des données doit elle aussi être représentative.

Vient l'**isolation des variables** : on ne modifie qu'un seul paramètre à la fois, tout le reste demeurant constant, faute de quoi on ne saura pas attribuer un écart de performance à sa cause. Avant de mesurer, on laisse le système **monter en température** (warm-up) afin que les caches — buffer pool, cache du système d'exploitation — se remplissent : mesurer à froid revient à mesurer le disque plutôt que le régime établi. La mesure proprement dite porte sur une **période soutenue** en régime stable, et non sur un pic instantané. Surtout, on **répète** : un tirage unique n'est que du bruit ; plusieurs exécutions, dont on examine la variance, sont nécessaires pour distinguer le signal.

Tout cela suppose un **environnement maîtrisé** — aucune autre charge concurrente sur la machine, sources de variabilité neutralisées — et une **documentation complète** : matériel, configuration, jeu de données, version de l'outil et paramètres employés. Sans cette traçabilité, un benchmark n'est pas reproductible, et donc difficilement exploitable.

## Les pièges courants

Plusieurs erreurs reviennent suffisamment souvent pour mériter d'être nommées. Benchmarker sur un matériel ou une configuration **différents de la production** produit des résultats qui ne se transposent pas. Négliger la **montée en température** revient à mesurer des entrées/sorties à froid au lieu du régime établi. Travailler sur un **jeu de données trop petit**, qui tient entièrement en mémoire alors que la production déborde, donne des chiffres irréalistes de rapidité.

Se contenter d'une **exécution unique** fait prendre le bruit pour un résultat. Ne regarder que la **moyenne** en ignorant le p99 masque la douleur réellement ressentie par les utilisateurs. Tester à une **concurrence irréaliste** — un seul client quand la production en compte plusieurs centaines — dépeint un système qui n'existe pas. L'**effet de l'observateur** est plus insidieux : l'outil de benchmark et la supervision consomment eux-mêmes des ressources et peuvent fausser la mesure.

Le piège le plus coûteux reste cependant de **benchmarker le mauvais élément** : optimiser et mesurer une requête qui n'est pas le véritable goulet d'étranglement. C'est pourquoi le benchmarking s'articule avec la méthodologie d'optimisation (§15.1) et l'analyse des requêtes lentes (§15.7) — on mesure d'abord pour localiser le vrai problème avant de l'attaquer. Enfin, une saine méfiance s'impose à l'égard des benchmarks « marketing » publiés par des fournisseurs et taillés pour flatter un produit : la seule mesure qui engage est celle que l'on conduit soi-même, sur sa propre charge.

## Le processus de benchmarking

En pratique, un benchmark suit un déroulé reproductible. On définit d'abord l'objectif et la métrique à observer, puis on prépare l'environnement et le jeu de données représentatif. On établit ensuite une **mesure de référence** (baseline) : c'est le point de comparaison sans lequel aucun « gain » ne peut être quantifié. On exécute alors le benchmark en régime stable, on relève les métriques, on les analyse, puis on modifie **une seule variable** avant de relancer et de comparer au point de référence. La conclusion ne porte que sur l'écart ainsi mesuré, dans les conditions documentées.

L'établissement de la baseline mérite d'être souligné : sans référence initiale, on ne mesure pas une amélioration mais une valeur isolée, dépourvue de sens comparatif. C'est la première mesure que l'on conserve, et celle à laquelle toutes les suivantes se rapportent.

## Les outils

L'écosystème offre plusieurs outils, à choisir selon l'objectif. **sysbench** (§15.12.1) est le benchmark synthétique OLTP de référence, conçu pour produire une charge transactionnelle paramétrable et comparable. **mysqlslap** (§15.12.2), livré avec MariaDB, est un outil plus simple d'émulation de charge, pratique pour un test rapide. Pour valider sur une charge réaliste, on s'appuiera plutôt sur le rejeu de charge de MaxScale (§14.5) ou sur l'analyse des requêtes réelles via pt-query-digest (§15.7.2). D'autres outils de l'écosystème, tels HammerDB ou les utilitaires Percona, complètent cette panoplie. La règle est constante : l'outil doit correspondre à la question — comparaison synthétique avec sysbench, test de charge rapide avec mysqlslap, validation réaliste par rejeu de charge.

Les deux sous-sections qui suivent détaillent les deux principaux outils de cette panoplie : sysbench (§15.12.1), puis mysqlslap (§15.12.2).

⏭️ [sysbench](/15-performance-tuning/12.1-sysbench.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2 — Variables système et de session

> Deuxième section du chapitre 11. Après la configuration *statique* par fichiers (section 11.1), on aborde ici l'autre versant : la configuration *dynamique* du serveur en cours d'exécution, à travers les variables système et de session.

---

## Introduction

La section précédente a montré *où* MariaDB lit ses réglages au démarrage. Mais une fois le serveur lancé, ces réglages prennent vie sous une autre forme : les **variables système**. Ce sont les paramètres du serveur en fonctionnement — taille des caches, nombre de connexions autorisées, comportement du moteur SQL, délais d'attente, et des centaines d'autres. Les consulter, c'est lire le tableau de bord du serveur ; les modifier, c'est agir sur son comportement, parfois sans même avoir à le redémarrer.

Cette section pose les bases du travail quotidien sur ces variables : comment les lire et les écrire, lesquelles peuvent changer à chaud, et lesquelles ont disparu avec la série 12.x.

---

## Des fichiers aux variables : deux faces d'un même réglage

Variables et fichiers de configuration ne sont pas deux mondes séparés, mais deux vues du même réglage. Une option écrite dans un fichier sous le groupe `[mariadbd]` — par exemple `max_connections = 200` — devient, au démarrage, la valeur de la variable système `max_connections`. Le fichier fixe la valeur *initiale* ; la variable la *porte* pendant toute la vie du serveur et peut, pour beaucoup d'entre elles, être ajustée en cours de route.

D'où une règle qu'il faut intégrer dès maintenant : **une modification de variable effectuée à chaud n'est pas persistante**. Elle agit immédiatement sur le serveur en marche, mais sera perdue au prochain redémarrage tant qu'elle n'a pas été reportée dans un fichier de configuration (section 11.1). Le réflexe professionnel est donc souvent en deux temps : on ajuste la variable à chaud pour observer l'effet, puis on pérennise le réglage retenu dans un fichier.

---

## Portée globale et portée de session : le cœur de la section

Le titre de cette section — variables « système *et* de session » — renvoie à la notion la plus structurante du sujet : la **portée** (*scope*) d'une variable. Une même variable peut exister à deux niveaux.

La **portée globale** concerne le serveur dans son ensemble. Une variable globale a une valeur unique, partagée par toutes les connexions, et régit le fonctionnement général de l'instance. La taille du *buffer pool* InnoDB (`innodb_buffer_pool_size`) ou le nombre maximal de connexions (`max_connections`) en sont des exemples : il n'y aurait aucun sens à les définir connexion par connexion, ce sont des paramètres de toute l'instance.

La **portée de session** concerne une seule connexion. Chaque client dispose de sa propre copie de ces variables, qui n'affecte que lui. À l'ouverture d'une connexion, ses variables de session sont **initialisées à partir des valeurs globales** du moment ; le client peut ensuite ajuster les siennes sans gêner les autres. Le tampon de tri (`sort_buffer_size`) illustre bien l'intérêt : un travail d'import ou de reporting particulièrement gourmand peut augmenter son tampon pour sa seule session, le temps de son exécution, sans imposer cette consommation mémoire à l'ensemble du serveur.

Toutes les variables n'existent pas dans les deux portées : certaines sont purement globales, d'autres purement de session, beaucoup sont disponibles dans les deux. Un point subtil mais important en découle : modifier une variable **globale** n'altère pas la valeur de session des connexions *déjà établies* — cela ne change que la valeur de départ des connexions *futures*. Pour agir sur une session en cours, il faut modifier sa variable de session.

Cette distinction est le fondement d'une administration fine : on règle globalement ce qui relève de l'instance, et l'on ajuste localement, par session, ce qui relève d'un traitement particulier.

---

## Ce que couvre cette section

Les trois sous-sections suivantes déclinent ce sujet :

- **11.2.1 — `SHOW VARIABLES` et `SET`** : les commandes pour consulter et modifier les variables, à la fois dans leur portée globale et de session.
- **11.2.2 — Variables dynamiques vs statiques** : la frontière essentielle entre les variables modifiables à chaud et celles qui exigent un redémarrage du serveur pour prendre effet.
- **11.2.3 — Variables retirées en 12.x** : l'inventaire des variables disparues dans la série 12.x (`big_tables`, `large_page_size`, `storage_engine`), un point de contrôle clé lors d'une migration depuis la 11.8.

---

## En résumé

Les variables système sont l'expression vivante de la configuration d'un serveur en fonctionnement, prolongeant les fichiers étudiés en 11.1 — avec la réserve qu'un changement à chaud n'est pas persistant. La clé de lecture du sujet est la **portée** : globale pour ce qui concerne toute l'instance, de session pour ce qui est propre à une connexion. Cette dualité ouvre la voie à un réglage à deux niveaux, que les sous-sections suivantes outillent concrètement, en commençant par les commandes de consultation et de modification, **11.2.1 — `SHOW VARIABLES` et `SET`**.

⏭️ [SHOW VARIABLES et SET](/11-administration-configuration/02.1-show-variables-set.md)

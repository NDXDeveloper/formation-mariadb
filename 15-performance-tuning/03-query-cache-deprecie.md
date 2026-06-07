🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.3 Query Cache : Pourquoi il est déprécié

Le *Query Cache* a longtemps été présenté comme une optimisation « gratuite » : mémoriser le résultat d'une requête pour le resservir instantanément lors de la prochaine exécution identique. En pratique, ce mécanisme — introduit dès 2001 avec MySQL 4.0.1 — est aujourd'hui considéré comme une fonctionnalité héritée (*legacy*), voire un anti-pattern de performance sur les serveurs modernes.

La situation diffère cependant selon le SGBD, et il est important de bien la comprendre avant d'aller plus loin : **MySQL a purement et simplement retiré le Query Cache**, tandis que **MariaDB le conserve mais le désactive par défaut** et en déconseille l'usage. Cette section détaille son fonctionnement, clarifie ce que « déprécié » signifie concrètement, puis expose les raisons architecturales qui ont conduit à son abandon de fait.

---

## Rappel : qu'est-ce que le Query Cache ?

Le Query Cache conserve en mémoire le **texte d'une requête `SELECT`** associé à son **jeu de résultats** complet. Lorsqu'une requête strictement identique est reçue par la suite, le serveur renvoie directement le résultat mémorisé, sans repasser par l'analyse syntaxique (*parsing*), l'optimisation, ni l'exécution.

Un point d'architecture est déterminant pour comprendre la suite : la consultation du cache intervient **après la réception de la requête mais avant l'analyseur SQL**. Le serveur compare donc la requête entrante au texte brut des requêtes déjà mémorisées. Cette comparaison est volontairement très stricte :

- elle est **sensible à la casse** : `SELECT * FROM t` et `SELECT * from t` sont considérées comme deux requêtes différentes ;
- elle prend en compte les **commentaires** : `/* v1 */ SELECT ...` diffère de `/* v2 */ SELECT ...` ;
- deux requêtes ne sont identiques que si elles partagent la **même base de données**, la **même version de protocole** et le **même jeu de caractères** par défaut.

Lorsqu'une donnée change dans une table, tous les résultats du cache concernant cette table sont immédiatement vidés : il n'est jamais possible d'obtenir une donnée périmée (*stale*) depuis le Query Cache.

---

## « Déprécié » : que faut-il comprendre exactement ?

Le titre de cette section reprend le terme courant de « dépréciation », mais le statut technique mérite une précision, car il n'est pas identique d'un SGBD à l'autre.

**Du côté de MySQL**, la dépréciation a été formelle puis définitive : le Query Cache a été déprécié en **MySQL 5.7.20** (2017), puis **entièrement supprimé du moteur en MySQL 8.0** (2018). Les variables `query_cache_*` ainsi que les indicateurs `SQL_CACHE` / `SQL_NO_CACHE` n'existent tout simplement plus. Toute configuration en faisant référence empêche le démarrage du serveur après migration.

**Du côté de MariaDB**, le Query Cache n'a *pas* été retiré. Il reste compilé dans le serveur par défaut, ce que l'on peut vérifier ainsi :

```sql
SHOW VARIABLES LIKE 'have_query_cache';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| have_query_cache | YES   |
+------------------+-------+
```

En revanche, il est **désactivé par défaut**. La documentation officielle est explicite : le cache « ne passe pas bien à l'échelle dans les environnements à fort débit sur machines multi-cœurs, c'est pourquoi il est désactivé par défaut ». Concrètement, `query_cache_type` vaut `OFF` au démarrage et le cache reste inactif tant qu'on ne l'active pas manuellement.

Pour MariaDB, il est donc plus exact de parler d'une fonctionnalité **désactivée par défaut et fortement déconseillée** que d'une fonctionnalité « supprimée ». Les raisons techniques de cette mise à l'écart sont néanmoins exactement les mêmes que celles qui ont motivé la suppression chez MySQL.

| | MySQL | MariaDB (12.3 LTS) |
|---|---|---|
| Présent dans le moteur | Non (retiré en 8.0) | Oui (`have_query_cache = YES`) |
| Statut | Déprécié en 5.7.20, supprimé en 8.0 | Conservé mais désactivé par défaut |
| Activé par défaut | — | Non (`query_cache_type = OFF`) |
| Variables `query_cache_*` | Supprimées | Toujours disponibles |
| Recommandation | Inutilisable | À laisser désactivé |

---

## Pourquoi il pose problème

Six raisons principales expliquent l'abandon du Query Cache. La première est de loin la plus importante.

### 1. Le verrou global : le défaut de conception majeur

Le Query Cache est protégé par un **unique verrou global** (le mutex `structure_guard_mutex`). Toute opération sur le cache — recherche d'une requête, insertion d'un résultat, invalidation après une écriture — doit d'abord acquérir ce verrou. Le cache constitue donc un **point de sérialisation** par lequel tous les threads doivent passer, à tour de rôle.

Sur un serveur multi-cœurs traitant de nombreuses connexions simultanées, les threads s'accumulent en attente de ce verrou. Pour limiter les blocages, MariaDB n'attend la prise du verrou que pendant un délai borné (50 ms, codé en dur) lors d'une recherche : au-delà, la requête contourne purement et simplement le cache et s'exécute normalement. De même, lorsque deux processus exécutent simultanément la même requête, seul le dernier stocke réellement le résultat ; tous les autres incrémentent le compteur `Qcache_not_cached`.

Il en résulte un **paradoxe** : plus le serveur est chargé — c'est-à-dire précisément la situation où l'on attend le plus d'un cache — plus le verrou global devient un goulot d'étranglement. Sur une charge fortement concurrente, le Query Cache peut rendre le serveur **plus lent** que s'il était totalement désactivé.

### 2. L'invalidation massive à chaque écriture

Chaque modification d'une table (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `ALTER`...) purge **immédiatement tous les résultats mémorisés qui font référence à cette table**, sans distinction. Une simple mise à jour d'une seule ligne dans une grande table suffit à effacer l'intégralité des requêtes en cache portant sur cette table, même celles dont le résultat n'était pas affecté par la ligne modifiée.

La conséquence est directe : dès qu'une charge comporte un taux d'écriture non négligeable, les entrées du cache sont détruites presque aussi vite qu'elles sont créées. À l'intérieur d'une transaction, le comportement est encore plus pénalisant : une écriture invalide le cache pour la table concernée et le désactive jusqu'au `COMMIT` ou au `ROLLBACK`, afin de préserver la cohérence et le verrouillage au niveau ligne.

C'est la raison pour laquelle le Query Cache n'a jamais réellement été bénéfique que pour les charges **à forte lecture et faible écriture** (typiquement certains sites web statiques d'autrefois) — un profil que la plupart des applications modernes ne présentent plus.

### 3. La fragmentation mémoire

Le cache stocke ses résultats dans des blocs de taille variable. Au fil du temps, l'allocation et la libération de ces blocs **fragmentent** la mémoire du cache. Une valeur élevée de `Qcache_free_blocks` par rapport à `Qcache_total_blocks` est un symptôme typique de fragmentation.

Récupérer cet espace impose de défragmenter explicitement le cache avec `FLUSH QUERY CACHE` — opération qui prend elle-même le verrou global évoqué plus haut. Le réglage de `query_cache_min_res_unit` n'offre qu'un compromis : une petite valeur augmente les verrous et la fragmentation mais gaspille moins de mémoire pour les petits résultats, tandis qu'une grande valeur réduit les verrous au prix d'un gaspillage mémoire accru.

### 4. De très nombreuses requêtes ne sont pas « cachables »

Une part importante du trafic d'une application réelle est, par nature, exclue du cache. Ne sont notamment **jamais mises en cache** les requêtes qui :

- utilisent une fonction non déterministe : `NOW()`, `CURDATE()`, `CURRENT_TIMESTAMP()`, `RAND()`, `UUID()`, `USER()`, `CONNECTION_ID()`, `SLEEP()`, `LAST_INSERT_ID()`, etc. ;
- contiennent `SELECT ... FOR UPDATE`, `LOCK IN SHARE MODE`, `INTO OUTFILE` ou `INTO DUMPFILE` ;
- interrogent une table d'`INFORMATION_SCHEMA`, de `mysql` ou de `performance_schema` ;
- font appel à une fonction stockée, une fonction définie par l'utilisateur (UDF), ou utilisent des variables utilisateur/locales ;
- portent sur une table `TEMPORARY`, ou n'utilisent aucune table ;
- génèrent un avertissement, ou s'exécutent dans une transaction en niveau d'isolation `SERIALIZABLE`.

À cela s'ajoute un point décisif pour le développement moderne : les **requêtes préparées** (*prepared statements*) sont toujours traitées comme distinctes des requêtes non préparées. Or l'usage des requêtes préparées est aujourd'hui la norme — pour la sécurité (prévention des injections SQL, cf. §17.8 et §17.9) comme pour la performance. Une large fraction du trafic applicatif est donc, de fait, inéligible au cache.

### 5. Une correspondance au texte trop fragile

Au-delà de la sensibilité à la casse et aux commentaires déjà mentionnée, de nombreuses **variables de session** différencient les entrées du cache : jeu de caractères du client et des résultats, collation de connexion, `sql_mode`, `time_zone`, `sql_select_limit`, `group_concat_max_len`, etc. Deux requêtes textuellement identiques exécutées avec des paramètres de session différents occupent deux entrées distinctes.

En pratique, les ORM (Hibernate, SQLAlchemy, Prisma, Entity Framework...) génèrent un SQL légèrement variable, et les sessions applicatives n'ont pas toujours des réglages homogènes. Le cache se fragmente alors en multiples variantes peu réutilisées, et le **taux de succès s'effondre** : un espace ou un commentaire différent suffit à transformer un succès en échec de cache.

### 6. Le matériel a profondément changé

Le Query Cache a été conçu pour un matériel mono- ou bicœur associé à des disques mécaniques lents. L'arbitrage qu'il opérait avait alors du sens : éviter une réexécution et des lectures disque coûteuses justifiait le coût d'un verrou.

Sur le matériel actuel — processeurs à plusieurs dizaines de cœurs, SSD/NVMe, grande quantité de RAM et **InnoDB Buffer Pool** très efficace — cet arbitrage s'est inversé. Le coût que le cache évitait (réexécution et accès disque) s'est effondré, tandis que le coût qu'il introduit (la contention sur le verrou global) est devenu dominant à mesure que le nombre de cœurs et de connexions augmentait. Le mécanisme combat aujourd'hui un problème devenu marginal au prix d'un problème devenu majeur.

---

## Diagnostiquer un Query Cache contre-productif

Sur un serveur hérité où le cache aurait été activé, plusieurs variables d'état permettent d'évaluer son efficacité :

```sql
SHOW STATUS LIKE 'Qcache%';
```

Les indicateurs à observer sont les suivants :

- `Qcache_hits` : nombre de requêtes servies depuis le cache ;
- `Qcache_inserts` : nombre de requêtes ajoutées au cache ;
- `Qcache_lowmem_prunes` : nombre de requêtes évincées faute de mémoire ;
- `Qcache_not_cached` : nombre de requêtes non mises en cache ;
- `Qcache_free_blocks` / `Qcache_total_blocks` : indicateurs de fragmentation ;
- `Qcache_queries_in_cache` : nombre de requêtes actuellement présentes.

Le signal d'alarme le plus parlant est le suivant : lorsque `Qcache_inserts` **et** `Qcache_lowmem_prunes` dépassent nettement `Qcache_hits`, cela signifie que des entrées sont ajoutées puis évincées plus vite qu'elles ne sont réellement réutilisées — autrement dit, le cache coûte plus qu'il ne rapporte.

On peut estimer le taux de succès par le rapport `Qcache_hits / (Qcache_hits + Qcache_inserts + Qcache_not_cached)`. Un taux faible accompagné d'un nombre élevé d'évictions est une indication claire qu'il faut désactiver le cache. Pour une analyse plus fine du contenu, le plugin `QUERY_CACHE_INFO` expose une table éponyme dans `INFORMATION_SCHEMA`.

À noter : les résultats renvoyés par le Query Cache sont comptabilisés dans `Com_select`, ce qui peut fausser certaines lectures de métriques si l'on ne tient pas compte de cette particularité.

---

## Les alternatives modernes

L'abandon du Query Cache ne laisse aucun vide : les mécanismes qui le remplacent sont à la fois plus performants et mieux adaptés au matériel actuel.

- **L'InnoDB Buffer Pool** est le véritable cache du serveur. Il conserve en mémoire les pages de données *et* d'index, bénéficie aussi bien aux lectures qu'aux écritures, passe à l'échelle sur les cœurs et ne souffre d'aucun verrou d'invalidation global. Son dimensionnement (typiquement 70 à 80 % de la RAM sur un serveur dédié) est l'optimisation la plus rentable. Voir §15.2.1.
- **L'indexation et l'optimisation des requêtes** : un index pertinent surpasse n'importe quel cache de résultats, car il réduit le coût de la requête elle-même plutôt que de masquer ce coût. Voir le chapitre 5 et §15.7.
- **Les requêtes préparées** (*prepared statements*) : l'analyse est réalisée une seule fois, le plan étant réutilisé ensuite. Voir §17.9.
- **Le cache applicatif** (Redis, Memcached) : la mise en cache au niveau de l'application offre un contrôle total sur la durée de vie (TTL) et l'invalidation, sans imposer de verrou au SGBD. C'est la meilleure réponse pour des résultats réellement « chauds » et rarement modifiés.
- **Le cache au niveau du proxy** (ProxySQL) : il permet de mettre en cache des résultats avec un TTL en dehors du serveur, en évitant entièrement le mutex côté moteur. Voir §14.9 et §17.2.2.
- **Les réplicas de lecture** : ils répartissent la charge de lecture horizontalement sur plusieurs serveurs. Voir le chapitre 13.

L'optimiseur de MariaDB, par ailleurs conscient des SSD dans son modèle de coûts (cf. §15.14), contribue lui aussi à réduire le coût de réexécution que le Query Cache cherchait historiquement à éviter.

---

## Désactiver proprement le Query Cache

Le cache étant désactivé par défaut sur les versions récentes de MariaDB, cette étape concerne surtout la **migration d'une configuration héritée** de MySQL 5.x ou d'anciennes versions de MariaDB.

On vérifie d'abord l'état courant :

```sql
SHOW VARIABLES LIKE 'query_cache_type';  
SHOW VARIABLES LIKE 'query_cache_size';  
```

Pour désactiver totalement le cache et libérer au maximum les ressources, on positionne **les deux variables à zéro** :

```sql
SET GLOBAL query_cache_type = 0;  
SET GLOBAL query_cache_size = 0;  
```

Dans le fichier `my.cnf` / `my.ini`, on supprime (ou on commente) les lignes `query_cache_*` héritées :

```ini
[mysqld]
# Lignes héritées à supprimer après migration :
# query_cache_type = 1
# query_cache_size = 64M

# Concentrer la mémoire sur le Buffer Pool InnoDB :
innodb_buffer_pool_size = 4G
```

Un piège classique mérite d'être signalé : `query_cache_type` est automatiquement basculé à `ON` si le serveur démarre avec un `query_cache_size` non nul et différent de la valeur par défaut. Une ancienne ligne `query_cache_size = 64M` oubliée dans la configuration **réactive donc silencieusement le cache** au redémarrage. C'est une erreur fréquente lors des migrations.

Enfin, `have_query_cache` ne change pas selon que le cache est activé ou non : il indique uniquement si le code du cache est compilé dans le serveur (`YES` par défaut). Il n'y a aucune raison de reconstruire MariaDB sans le cache : il suffit de le laisser désactivé.

---

## Points clés à retenir

- Le Query Cache mémorise le **texte d'un `SELECT`** et son **résultat**, avec une correspondance au texte exacte et sensible à la casse, vérifiée avant l'analyseur SQL.
- **MySQL** l'a déprécié en 5.7.20 puis **supprimé en 8.0** ; **MariaDB** le conserve mais le **désactive par défaut** (`query_cache_type = OFF`) et en déconseille l'usage.
- Le **défaut majeur** est le **verrou global** : tout accès au cache se sérialise, et la contention augmente avec le nombre de cœurs et de connexions — le cache peut ralentir un serveur chargé.
- L'**invalidation totale à chaque écriture**, la **fragmentation mémoire** et la **part élevée de requêtes non cachables** (fonctions non déterministes, requêtes préparées, etc.) limitent fortement son intérêt réel.
- Le **matériel moderne** (multi-cœurs, SSD/NVMe, Buffer Pool efficace) a **inversé l'arbitrage** qui justifiait son existence.
- Les remplaçants recommandés sont l'**InnoDB Buffer Pool**, une **indexation soignée**, les **requêtes préparées**, le **cache applicatif (Redis/Memcached)**, le **cache de proxy (ProxySQL)** et les **réplicas de lecture**.

⏭️ [Configuration I/O et disques](/15-performance-tuning/04-configuration-io-disques.md)

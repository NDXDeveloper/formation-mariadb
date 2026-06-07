🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.5 Optimisation du moteur InnoDB

InnoDB est le moteur de stockage par défaut de MariaDB (cf. §7.2) et celui qui sous-tend la quasi-totalité des charges transactionnelles. Son optimisation est donc au cœur du *tuning* d'un serveur. Cette section adopte une vision d'ensemble : elle relie les réglages déjà traités dans leurs sections dédiées, approfondit les leviers InnoDB qui n'ont pas la leur, et signale ce que MariaDB moderne a automatisé ou retiré — un point essentiel pour ne pas appliquer des conseils devenus obsolètes.

---

## Une approche méthodique

Le principe directeur de l'optimisation d'InnoDB tient en une phrase : **un petit nombre de paramètres apporte l'essentiel des gains**, et l'immense majorité des valeurs par défaut sont pertinentes. Plutôt que de modifier des dizaines de variables obscures — démarche risquée et souvent contre-productive —, il faut se concentrer sur trois piliers (mémoire, journalisation, E/S), mesurer avant d'agir (cf. §15.1), puis ajuster au matériel et à la charge.

---

## Les trois piliers, et où les traiter

Deux de ces piliers font l'objet de sections dédiées ; il suffit ici de les situer :

- **La mémoire** est le facteur numéro un : le dimensionnement de l'**InnoDB Buffer Pool** détermine la part des données servies sans accès disque. Il est traité au §15.2.1. On notera qu'en MariaDB moderne (depuis 11.8.2), le Buffer Pool peut être **redimensionné dynamiquement** par incréments de 1 Mo jusqu'à `innodb_buffer_pool_size_max` (à fixer au démarrage), et réduit jusqu'à `innodb_buffer_pool_size_auto_min` en cas de pression mémoire ; `innodb_buffer_pool_chunk_size` est désormais déprécié et ignoré.
- **Les E/S** sont traitées en §15.4 : capacité d'E/S au §15.4.1 (`innodb_io_capacity`), méthode de flush au §15.4.2 (`innodb_flush_method`), optimisations SSD au §15.4.3.
- **La journalisation** (redo log) n'a pas de section dédiée et fait l'objet du point suivant.

---

## Le redo log : dimensionnement

Le redo log (journal de reprise) est au cœur de la durabilité d'InnoDB : selon le principe du *Write-Ahead Logging* (§15.4), toute modification y est consignée avant que les pages correspondantes ne soient écrites dans les fichiers de données, ce qui permet une reprise cohérente après un crash. Depuis MariaDB 10.5, il s'agit d'un **fichier unique** (`ib_logfile0`).

Sa taille est fixée par **`innodb_log_file_size`** — à ne pas confondre avec `innodb_redo_log_capacity`, qui est une variable propre à MySQL (8.0.30+) et **n'existe pas sous MariaDB**. Depuis MariaDB 10.9, cette variable est **dynamique** : elle peut être modifiée sans redémarrage via `SET GLOBAL` (l'opération de redimensionnement s'effectue de façon asynchrone en arrière-plan, et doit aussi être inscrite dans le fichier de configuration pour survivre aux redémarrages). La variable `innodb_log_files_in_group`, héritée de l'ancien modèle multi-fichiers, est devenue sans objet.

Le dimensionnement obéit à un compromis :

- un **redo log plus grand** espace les checkpoints, lisse les E/S et améliore le débit en écriture ;
- mais il **allonge la durée de reprise** après un crash, ralentit l'arrêt avec `innodb_fast_shutdown=0`, et consomme davantage d'espace disque.

La valeur par défaut est modeste (de l'ordre de la centaine de mégaoctets) et se révèle généralement **trop petite pour une charge de production en écriture**. Le bon indicateur est l'**occupation du redo log** (*redo log occupancy*) : si l'âge du checkpoint approche régulièrement la capacité du journal (rapport proche de 1,0), c'est que celui-ci est sous-dimensionné. Un point spécifique à MariaDB 12.3 : l'activation du nouveau **binlog intégré à InnoDB** (§11.5.4) augmente d'environ 2× l'amplification du redo, ce qui impose alors un `innodb_log_file_size` nettement plus généreux.

Enfin, une mise en garde : il ne faut **jamais déplacer ou éditer manuellement** `ib_logfile0`, sous peine de corruption. Le tampon d'écriture du journal, `innodb_log_buffer_size`, a par ailleurs une valeur minimale de 2 Mo depuis la 10.8 ; il est rarement nécessaire de l'augmenter au-delà de quelques dizaines de mégaoctets pour les charges à grosses transactions.

---

## La durabilité au commit

Le paramètre **`innodb_flush_log_at_trx_commit`** règle le compromis fondamental entre durabilité et performance, en déterminant ce qui se passe à chaque `COMMIT` :

| Valeur | Écriture du redo au commit | Synchronisation (`fsync`) | Risque en cas de crash |
|---|---|---|---|
| `1` (défaut) | oui | à chaque commit | aucune perte — durabilité ACID complète |
| `2` | oui (vers le cache de l'OS) | environ une fois par seconde | perte ≤ ~1 s **uniquement** en cas de crash de l'OS ou coupure d'alimentation |
| `0` | environ une fois par seconde | environ une fois par seconde | perte ≤ ~1 s même en cas de crash de `mariadbd` |

L'intervalle de synchronisation pour les valeurs `0` et `2` est gouverné par `innodb_flush_log_at_timeout` (1 seconde par défaut). InnoDB regroupe par ailleurs les synchronisations de commits concurrents (*group commit*), ce qui amortit leur coût.

En pratique : on conserve **`1`** pour toute donnée critique (aucune perte tolérée) ; **`2`** est un compromis acceptable lorsqu'une perte de l'ordre de la seconde est admissible et que l'on cherche du débit ; **`0`** ne se justifie que très rarement. Un point propre à la 12.3 modifie ce calcul pour les architectures répliquées : le binlog intégré à InnoDB rend le binary log *crash-safe* même avec un réglage relâché (§11.5.4), ce qui sécurise les configurations qui choisissent `2` pour la performance.

---

## Purge, undo et MVCC

Pour assurer le contrôle de concurrence multi-version (MVCC, §6.6), InnoDB conserve les **anciennes versions des lignes** dans les *undo logs*. Ces versions doivent ensuite être nettoyées : c'est le rôle de la **purge**. Lorsque la purge ne suit pas — typiquement à cause de **transactions longues ou laissées ouverte** qui « épinglent » les anciennes versions —, la *history list* s'allonge, les *undo tablespaces* gonflent et les performances se dégradent.

Plusieurs paramètres encadrent ce mécanisme : `innodb_purge_threads` (nombre de threads de purge parallèles) et `innodb_purge_batch_size` règlent l'agressivité de la purge ; `innodb_undo_tablespaces` permet d'isoler les *undo logs* dans des tablespaces dédiés ; `innodb_undo_log_truncate`, conjugué à `innodb_max_undo_log_size`, autorise la **récupération de l'espace** en tronquant les *undo tablespaces* devenus trop volumineux.

Le bon réflexe de surveillance est de suivre la longueur de la *history list* (via `SHOW ENGINE INNODB STATUS` ou la variable `Innodb_history_list_length`). Mais le levier le plus efficace reste organisationnel : **éviter les transactions de longue durée** et veiller à ce qu'aucune connexion ne laisse une transaction ouverte inutilement.

---

## Concurrence et threads d'E/S

Le paramètre `innodb_thread_concurrency` (qui limitait le nombre de threads pouvant entrer simultanément dans InnoDB), ainsi que les autres réglages d'étranglement de concurrence d'InnoDB, ont été **dépréciés en 10.5.5 puis retirés en MariaDB 10.6.0** (MDEV-23397) : ils **n'existent plus** en 12.3. Les conseils anciens recommandant de plafonner la concurrence d'InnoDB sont donc caducs — l'InnoDB moderne a supprimé ce mécanisme d'étranglement, devenu contre-productif, et la concurrence se gère désormais au niveau du **Thread Pool** (§11.10).

Les threads d'E/S asynchrones — `innodb_read_io_threads` et `innodb_write_io_threads` — déterminent le parallélisme des lectures et écritures de pages (cf. §15.4.3). Leurs valeurs par défaut conviennent à la plupart des serveurs ; on ne les augmente que pour un stockage à très haut débit capable d'absorber davantage d'E/S parallèles. Enfin, `innodb_deadlock_detect` (activé par défaut) peut, sur des charges à concurrence extrême, être désactivé au profit du seul `innodb_lock_wait_timeout` — un réglage de niche, à n'envisager qu'avec précaution.

---

## Taille de page

`innodb_page_size` vaut **16 Ko par défaut** et ne peut être fixé qu'à l'**initialisation** de l'instance (valeurs possibles : 4K, 8K, 16K, 32K, 64K). Des pages plus petites peuvent marginalement aider certaines charges à accès très aléatoires ou à lignes courtes, des pages plus grandes des charges à grandes lignes ou à balayages massifs. En pratique, **16 Ko convient à la quasi-totalité des cas** et il est très rare d'avoir une raison solide de changer cette valeur.

---

## Préchauffage du Buffer Pool

Pour éviter un cache « froid » après un redémarrage, InnoDB peut enregistrer la liste des pages présentes dans le Buffer Pool à l'arrêt et les recharger au démarrage. Les variables `innodb_buffer_pool_dump_at_shutdown` et `innodb_buffer_pool_load_at_startup` (activées par défaut) assurent ce mécanisme, qui accélère sensiblement la montée en régime après un redémarrage (voir aussi §15.2).

---

## Évolutions à connaître et pièges à éviter

L'InnoDB de MariaDB 12.3 a beaucoup évolué, et plusieurs réglages couramment cités en ligne sont devenus inutiles, voire dangereux.

Le changement le plus notable est la **suppression du change buffer**. Ce mécanisme, qui différait l'application des modifications aux index secondaires, a été **désactivé par défaut dès 10.5.15**, déprécié en 10.9, puis **retiré en MariaDB 11.0** (MDEV-29694). Il a été abandonné parce qu'il constituait une source majeure de bugs de corruption très difficiles à reproduire, pour un gain de performance au mieux marginal, et qu'il pouvait faire croître le *system tablespace* de façon incontrôlée. **En 12.3, il n'existe plus** : il ne faut donc pas chercher à régler `innodb_change_buffering` ni `innodb_change_buffer_max_size`. Une précaution s'impose lors d'une migration franchissant la frontière de la 11.0 : comme désactiver le change buffer ne le vidait pas, il faut **fusionner les changements en attente avec `innodb_fast_shutdown=0`** avant la mise à niveau, sous peine d'un échec au démarrage (cf. §19.10).

D'autres évolutions sont utiles à connaître : le **redimensionnement dynamique du redo log** (depuis 10.9) et la **refonte du redimensionnement du Buffer Pool** (modèle souple par incréments fins plafonné par `innodb_buffer_pool_size_max`, depuis 11.8.2 — le redimensionnement à chaud lui-même existant, lui, depuis 10.2.2 ; voir §15.2.1), ainsi que l'activation par défaut, en 12.x, de **`innodb_snapshot_isolation`**, qui modifie le comportement du niveau d'isolation REPEATABLE READ (cf. §6.9 et §19.10).

Enfin, plusieurs paramètres encore présents dans d'anciens guides ont été **retirés ou rendus inopérants** et doivent être ignorés : `innodb_buffer_pool_instances` (supprimé en 10.5, Buffer Pool désormais en instance unique), `innodb_page_cleaners` (supprimé en 10.6), `innodb_buffer_pool_chunk_size` (déprécié et sans effet), `innodb_log_files_in_group` (sans objet avec le fichier unique) et `innodb_additional_mem_pool_size` (supprimé en 10.2). Quant à `innodb_log_write_ahead_size`, retiré en 10.8, il a été **réintroduit dans la série 11.x** et existe toujours (défaut 512) : sa taille est auto-détectée et il n'est à régler qu'en cas de mauvaise détection (cf. §15.4.3).

---

## Points clés à retenir

- L'optimisation d'InnoDB repose sur **peu de paramètres** : la mémoire (Buffer Pool, §15.2.1), la journalisation (redo log, ci-dessus) et les E/S (§15.4) ; les défauts restants sont en général pertinents.
- Le **redo log** se dimensionne avec **`innodb_log_file_size`** (et non `innodb_redo_log_capacity`, propre à MySQL), dynamique depuis la 10.9 ; un journal plus grand lisse les E/S mais allonge la reprise. Le binlog InnoDB de la 12.3 impose de le surdimensionner (§11.5.4).
- **`innodb_flush_log_at_trx_commit`** arbitre durabilité/performance : **`1`** (défaut, aucune perte), `2` (perte ≤ ~1 s sur crash OS), `0` (perte ≤ ~1 s même sur crash serveur).
- La **purge/undo** doit suivre le rythme des écritures ; le risque principal vient des **transactions longues** qui font gonfler la *history list*.
- Ne plus chercher à brider la concurrence d'InnoDB : **`innodb_thread_concurrency` a été retiré en 10.6** (avec les autres paramètres d'étranglement) ; la concurrence se gère via le **Thread Pool** (§11.10). Conserver le **préchauffage du Buffer Pool** activé.
- Le **change buffer a été supprimé en 11.0** : ne pas le régler, et fusionner ses changements avant une migration franchissant la 11.0 (§19.10).
- **Se méfier des guides obsolètes** : `innodb_buffer_pool_instances`, `innodb_page_cleaners`, `innodb_change_buffering`, `innodb_additional_mem_pool_size`, etc., n'existent plus (`innodb_log_write_ahead_size`, lui, a été retiré en 10.8 puis réintroduit en 11.x — cf. §15.4.3).

⏭️ [innodb_alter_copy_bulk : Construction d'index efficace](/15-performance-tuning/06-innodb-alter-copy-bulk.md)

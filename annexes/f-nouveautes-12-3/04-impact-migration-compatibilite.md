🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.4 — Impact sur migration et compatibilité (changements de comportement)

La plupart des nouveautés de la série 12.x sont *additives* : de nouvelles fonctionnalités, souvent optionnelles, qui n'altèrent pas le comportement existant. Une poignée d'évolutions modifient toutefois des comportements en place et méritent une attention particulière lors d'une montée de version depuis la 11.8. Cette section les passe en revue, des plus susceptibles d'affecter une application ou un déploiement existant aux plus ciblées. La procédure de migration proprement dite est traitée au [§19.10](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md), les chemins de mise à jour au §19.4 et le repli au §19.7.

## Les changements de comportement à anticiper

### 1. Idée reçue à dissiper : l'isolation par instantané n'est *pas* un changement 11.8 → 12.3 (§6.9)

On lit souvent que l'activation par défaut de `innodb_snapshot_isolation` serait *le* grand changement de comportement du passage à la 12.3. **C'est inexact pour une migration depuis la 11.8.** Le défaut de ce paramètre est passé à `ON` dès la **11.6.2** : il est donc déjà actif en 11.8, et le passage 11.8 → 12.3 n'y change rien.

Le mécanisme reste utile à connaître : sous `REPEATABLE READ`, une transaction qui tente de modifier une ligne qu'une autre a validée *après* la prise de son instantané reçoit une **erreur de conflit** (`1020`) au lieu d'opérer silencieusement sur la version la plus récente. Une application transactionnelle doit donc capter cette erreur et rejouer la transaction — mais une application qui tournait déjà sur 11.8 le fait (ou aurait dû le faire) **avant** la migration.

**Quand est-ce réellement un changement ?** Uniquement pour les migrations depuis une version **antérieure à 11.6.2** (MariaDB 10.6 / 10.11 / 11.4) ou depuis **MySQL** : là, le code passe d'une modification silencieuse à une erreur, et l'adaptation (logique de *retry*) devient nécessaire. Le comportement antérieur peut alors être rétabli à titre transitoire via `innodb_snapshot_isolation = OFF` (§6.9).

### 2. Variables système retirées (§11.2.3)

Les variables `big_tables`, `large_page_size` et `storage_engine`, dépréciées de longue date, sont **supprimées** en 12.x : elles n'existent plus comme variables système (`SET GLOBAL` ou `@@` renvoient « *Unknown system variable* »). Contrairement à une idée répandue, leur présence dans un `my.cnf` n'**empêche pas** le démarrage : MariaDB les tolère encore par compatibilité, démarre normalement, journalise un avertissement — « *'big-tables' was removed. It does nothing now and exists only for compatibility with old my.cnf files* » — et **ignore le réglage**. Le vrai risque n'est donc pas un refus de démarrage, mais un paramètre que l'on croit appliqué et qui reste sans effet, ainsi que tout code applicatif qui lit ou positionne ces variables via SQL et qui, lui, échouera. (À distinguer d'une variable réellement *inconnue* — une faute de frappe, par exemple : celle-ci provoque bien l'arrêt au démarrage avec `[ERROR] mariadbd: unknown variable`.)

**À vérifier.** Passer en revue les fichiers de configuration et le code applicatif pour retirer ces variables ; `storage_engine` se remplace par `default_storage_engine`. C'est un correctif simple, à appliquer de préférence avant la montée de version, pour éviter avertissements au journal et réglages silencieusement ignorés.

### 3. Packaging de Galera séparé (§14.2.5)

Le fournisseur Galera est désormais distribué dans un **paquet distinct, `mariadb-server-galera`**, et la dépendance a été retirée du paquet serveur principal. Installer le seul paquet serveur n'apporte donc plus Galera : chaque nœud de cluster doit installer explicitement `mariadb-server-galera`. Cela concerne les installations par paquets, les images Docker (§16.3.1) et le déploiement Galera via l'opérateur Kubernetes (§16.5.2).

**À vérifier.** Mettre à jour les mécanismes de provisionnement (rôles Ansible, Dockerfiles, manifestes de l'opérateur) pour installer le paquet Galera sur les nœuds de cluster. Les serveurs autonomes ne sont pas concernés.

### 4. Binlog intégré à InnoDB : optionnel (§11.5.4)

Le journal binaire réécrit et intégré à InnoDB, présenté en [F.2](02-features-phares.md), est **optionnel**. La mise à jour ne change donc pas le comportement du binlog par défaut. En revanche, son activation modifie l'architecture du journal : il convient de valider au préalable la compatibilité des outils qui consomment le binlog — réplicas externes, pipelines de *Change Data Capture* comme Debezium (§20.8.3), chaînes de sauvegarde et de restauration *point-in-time*.

**À vérifier.** Procéder à la mise à jour en conservant d'abord le binlog classique, valider la chaîne outillée, puis activer le binlog InnoDB une fois les tests concluants (paramétrage au §11.5.4).

### 5. Noms de contraintes de clés étrangères relâchés (§18.12)

La règle d'unicité des noms de contraintes de clés étrangères est **assouplie** : un nom doit désormais être unique au sein de sa table seulement, et non plus à l'échelle de la base. Il s'agit d'un relâchement de contrainte : les schémas valides en 11.8 le restent après la mise à jour, et l'on peut désormais réutiliser un même nom de contrainte dans des tables différentes — ce qui simplifie l'import de schémas en provenance d'Oracle ou d'autres SGBD.

**À vérifier.** Le point d'attention est ici inverse de celui des autres changements : il concerne l'outillage ou les requêtes qui supposaient l'unicité d'un nom de contrainte à l'échelle de la base (par exemple une interrogation d'`INFORMATION_SCHEMA` indexée sur le seul nom de contrainte), et qui peuvent désormais remonter plusieurs lignes ; il faut alors préciser la table.

### 6. DROP USER en mode Oracle (§10.2.1)

En **mode Oracle** (`sql_mode = ORACLE`), `DROP USER` émet un avertissement lorsque l'utilisateur visé a des sessions actives et **échoue** dans ce mode. Le comportement, déjà évoqué en [F.3](03-compatibilite-oracle-mysql.md) au titre de la compatibilité, constitue aussi un changement à connaître : un script de nettoyage qui supprime un utilisateur encore connecté ne se déroulera plus de la même manière.

**À vérifier.** Sous `sql_mode = ORACLE`, fermer au préalable les sessions de l'utilisateur (ou traiter l'échec dans le script). Hors mode Oracle, seul l'avertissement s'applique.

## Points de contrôle pour la migration

En synthèse, les vérifications à mener autour d'une montée de version 11.8 → 12.3 :

- Isolation par instantané : **pas un changement 11.8 → 12.3** (déjà active en 11.8) ; n'ajouter une logique de *retry* que pour une migration depuis une version antérieure à 11.6.2 ou depuis MySQL.
- Nettoyer les fichiers de configuration des variables retirées : le serveur les tolère mais les ignore (avertissement au journal), et tout accès SQL à ces variables échoue.
- Sur les clusters, installer explicitement le paquet `mariadb-server-galera`.
- Valider les outils consommant le binlog avant d'activer le binlog InnoDB.
- Repérer l'outillage et les requêtes supposant l'unicité des noms de contraintes FK à l'échelle de la base.
- En mode Oracle, adapter les scripts `DROP USER`.

Pour la marche à suivre détaillée, voir le [§19.10](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md) (migration dédiée 11.8 → 12.3), le [§19.4](../../19-migration-compatibilite/04-strategies-mise-a-jour.md) (chemins de mise à jour) et le [§19.7](../../19-migration-compatibilite/07-rollback-contingence.md) (repli et contingence). Le panorama des nouveautés figure en [F.1](01-tableau-recapitulatif.md), et les recommandations d'adoption en [F.5](05-recommandations-adoption.md).

⏭️ [Recommandations d'adoption](/annexes/f-nouveautes-12-3/05-recommandations-adoption.md)

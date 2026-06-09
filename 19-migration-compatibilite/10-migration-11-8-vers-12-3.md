🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.10 Migration 11.8 → 12.3 : changements de comportement 🆕

Cette section est la **synthèse opérationnelle du chapitre** pour le chemin de migration central de la formation : le passage de la LTS précédente (11.8, juin 2025) à la LTS de référence (12.3, mai 2026). Là où les sections précédentes ont décrit des *méthodes* transposables à toute migration — analyse de compatibilité (§19.5), tests (§19.6), rollback (§19.7), bascule sans interruption (§19.8), tables temporelles (§19.9) —, celle-ci les applique au cas concret 11.8 → 12.3 et recense les **changements de comportement** précis à anticiper.

L'enjeu d'une telle section est de distinguer nettement deux catégories souvent confondues : les nouveautés qui **s'ajoutent sans rien casser** (des opportunités, adoptables à loisir) et les changements qui **modifient le comportement d'un code ou d'une configuration existants** (des risques, à traiter avant la bascule). Une migration réussie consiste à neutraliser la seconde catégorie ; la première peut attendre.

---

## 19.10.1 Une migration LTS → LTS relativement contenue

La 12.3 **consolide la série rolling 12.0 → 12.2** : ce n'est pas une rupture, mais l'aboutissement stabilisé d'un cycle. Par rapport à la 11.8, l'essentiel des nouveautés 12.x sont des **ajouts** — compatibilité Oracle/MySQL renforcée, SQL standard, fonctions de sécurité — qui n'affectent pas le comportement du code existant. Un nombre réduit de changements modifient en revanche des comportements établis et exigent une action.

**Trois** changements de comportement structurent réellement cette migration et méritent un traitement individuel. Un quatrième point — `innodb_snapshot_isolation` — est presque toujours cité comme *le* changement phare de la 12.3 : c'est une **idée reçue**, dissipée d'emblée (§19.10.2), car ce paramètre est en réalité **déjà actif par défaut en 11.8**.

| # | Changement | Nature | Section dédiée |
|---|------------|--------|----------------|
| 1 | Variables système retirées (`big_tables`, `large_page_size`, `storage_engine`) | Configuration / démarrage | §11.2.3 |
| 2 | Noms de contraintes FK uniques par table seulement | Schéma / outillage | §18.12 |
| 3 | Packaging Galera séparé (`mariadb-server-galera`) | Déploiement / provisionnement | §14.2.5 |
| — | `innodb_snapshot_isolation = ON` (**déjà le cas en 11.8**) | *Idée reçue* — pas un delta 11.8 → 12.3 | §19.10.2 |

La section 19.10.2 dissipe l'idée reçue sur l'isolation par instantané ; les sections 19.10.3 à 19.10.5 traitent les trois changements réels ; la 19.10.6 recense les changements de second rang ; la 19.10.7 rappelle les ajouts non risqués ; les 19.10.8 et 19.10.9 proposent une démarche et une checklist consolidées.

---

## 19.10.2 Idée reçue à dissiper : l'isolation par instantané n'est pas un changement 11.8 → 12.3

On lit souvent que l'activation par défaut de `innodb_snapshot_isolation` serait *le* grand changement de comportement de la 12.3. **C'est inexact pour une migration 11.8 → 12.3.** Le défaut de ce paramètre est passé à `ON` dès **MariaDB 11.6.2** (MDEV-35124) — il est donc déjà à `ON` dans la 11.8. Les deux versions se comportent ici à l'identique, et **ce point n'introduit aucun changement sur le chemin 11.8 → 12.3**.

Le mécanisme reste utile à connaître. Sous `REPEATABLE READ` (le défaut d'InnoDB), lorsqu'une transaction tente de modifier une ligne qu'une autre a modifiée *et validée* après l'établissement de son instantané, MariaDB lève l'erreur `1020` :

```
ERROR 1020 (HY000): Record has changed since last read in table '...'; try restarting transaction
```

Une application transactionnelle doit donc **capturer cette erreur et rejouer la transaction**. Mais — point essentiel — une application qui tournait déjà sur 11.8 **gère déjà ce cas** (ou aurait dû le faire). Ce paramètre ne devient un *vrai* changement de comportement que pour les migrations depuis une version **antérieure à 11.6.2** (MariaDB 10.6 / 10.11 / 11.4) ou depuis **MySQL** : là, le code passe d'une modification silencieuse à une erreur, et l'adaptation devient nécessaire (au besoin, `innodb_snapshot_isolation = OFF` rétablit transitoirement l'ancien comportement). Ce point a été analysé sous l'angle de la compatibilité applicative (§19.5.4) et fait l'objet de tests dédiés (§19.6.9).

---

## 19.10.3 Changement majeur 1 : variables système retirées

La série 12.x a retiré plusieurs variables système devenues obsolètes : `big_tables`, `large_page_size` et `storage_engine` (§11.2.3). Ces variables avaient perdu leur utilité — `storage_engine` était remplacée de longue date par `default_storage_engine`, et `big_tables` correspond à un comportement désormais automatique.

Le risque est direct et potentiellement bloquant : tout **fichier de configuration** (`my.cnf` / `my.ini`) ou **script de démarrage** qui mentionne l'une de ces variables provoquera une erreur, pouvant empêcher le serveur de démarrer. De même, toute instruction `SET` (globale ou de session) émise par l'application et référençant ces variables échouera.

L'action est simple mais ne doit pas être omise : **rechercher ces trois noms** dans l'ensemble des fichiers de configuration *et* dans le code applicatif susceptible de positionner des variables de session, puis les supprimer ou les remplacer avant la migration. C'est un cas d'école de vérification par *smoke test* de démarrage avec la configuration cible (§19.6.9).

---

## 19.10.4 Changement majeur 2 : portée des noms de contraintes de clés étrangères

Jusqu'à la 11.8, le nom d'une contrainte de clé étrangère devait être unique **à l'échelle de la base de données**. En 12.3, cette unicité n'est plus exigée qu'**à l'échelle de la table** (§18.12). Le même nom de contrainte peut donc désormais être réutilisé dans des tables différentes.

Ce changement est globalement **assouplissant** : il lève une contrainte plutôt qu'il n'en ajoute, et la plupart des applications n'en seront pas affectées. Les points de vigilance concernent l'**outillage** plutôt que le SQL applicatif courant. Un outil de gestion de migrations de schéma (Flyway, Liquibase — §16.8), un ORM qui génère et nomme automatiquement les contraintes, ou un script qui manipule les contraintes par leur nom en supposant une unicité à l'échelle de la base, peut produire un résultat différent ou s'appuyer sur une hypothèse devenue caduque.

L'action consiste à **passer en revue l'outillage de gestion de schéma** et le DDL généré, et à vérifier tout code qui référence des contraintes de clés étrangères par leur nom. En pratique, ce point est rarement bloquant, mais il mérite d'être validé plutôt que présumé.

---

## 19.10.5 Changement majeur 3 : packaging Galera séparé

En 12.3, la dépendance Galera est **retirée du paquet serveur de base** et isolée dans un paquet distinct, `mariadb-server-galera` (§14.2.5). Installer ou mettre à niveau `mariadb-server` seul ne tire donc plus automatiquement la bibliothèque Galera.

L'impact dépend du type de déploiement. Un serveur **autonome** (non Galera) n'est pas affecté — il est même légèrement allégé. Un **nœud de cluster Galera**, en revanche, requiert désormais l'installation explicite du paquet `mariadb-server-galera`. Ce changement se répercute sur l'ensemble de la chaîne de provisionnement : scripts d'installation, playbooks Ansible (§16.2.1), images Docker officielles (§16.3.1) et déploiements via l'operator Kubernetes (§16.5.2).

L'action est d'**ajouter l'installation explicite de `mariadb-server-galera`** au provisionnement des nœuds de cluster, et de vérifier que les images conteneurisées et les manifestes d'operator intègrent ce paquet. C'est un point purement opérationnel, mais son oubli empêcherait un nœud de rejoindre le cluster.

---

## 19.10.6 Autres changements à anticiper

Au-delà de ces trois changements majeurs, trois évolutions de second rang méritent attention.

**Nouveau binlog intégré à InnoDB** (§11.5.4), **optionnel** : il s'active via `binlog_storage_engine=innodb` et reste désactivé par défaut. Tant qu'on ne l'active pas, rien ne change. Une fois activé, c'est avant tout un gain — la suppression de la synchronisation entre binlog et moteur apporte un débit en écriture nettement supérieur. Mais le changement d'architecture du journal a des implications : il rend fragile la **réplication de la 12.3 vers une version antérieure** (le filet de rollback par réplication inverse, §19.7.5), impose de **vérifier les consommateurs externes du binlog** (CDC, Debezium — §20.8) et la chaîne de PITR (§12.5.2). À noter que la réplication *dans le sens de la migration* (11.8 → 12.3) n'est pas affectée, la 12.3 consommant un binlog classique (§19.8.4).

**Plans d'exécution potentiellement différents.** L'optimiseur 12.x a évolué : Optimizer Hints (§15.15), optimisations sur scans inversés (§5.8.1), nouveaux algorithmes pour `PARTITION BY KEY` (§15.9.3), prise en compte du SSD dans le calcul des coûts (§15.14). Pour une même requête, la 12.3 peut donc choisir un plan différent — généralement meilleur, parfois en régression sur un cas particulier. La comparaison des plans `EXPLAIN` sur les requêtes critiques (§19.6.8) est l'action recommandée ; les Optimizer Hints permettent de figer un plan si nécessaire. Les nouveaux algorithmes de partitionnement par clé peuvent par ailleurs modifier la distribution des lignes en cas de reconstruction, à vérifier du point de vue du *partition pruning* (§15.9.4).

**`DROP USER` avec sessions actives** (§10.2.1). La suppression d'un utilisateur disposant de sessions actives émet désormais un avertissement — et échoue en mode Oracle. Les scripts d'administration ou d'automatisation qui suppriment des comptes doivent en tenir compte.

---

## 19.10.7 Ce qui s'ajoute sans rien casser

La majorité des nouveautés 12.x sont des **ajouts sans incidence sur l'existant** : ils n'introduisent aucun risque de migration et constituent des opportunités à adopter après la bascule, à son rythme. Les recenser ici permet de les écarter mentalement de l'analyse de risque.

Côté **compatibilité**, `caching_sha2_password` (§10.5.5) facilite l'accueil d'applications conçues pour MySQL 8 (§19.5.5), et un ensemble de fonctionnalités lissent la migration depuis Oracle : `TO_DATE`/`TO_NUMBER`/`TRUNC` (§3.7.1), jointures externes `( + )` (§3.3.5), tableaux associatifs (§8.1.4), `SYS_REFCURSOR` (§8.5), type `XMLTYPE` (§2.2.6) — l'ensemble est cartographié en §19.2.1. Côté **SQL standard**, on trouve le prédicat `IS JSON` (§4.7.4), `SET PATH` (§19.2.1) et `UPDATE`/`DELETE` lisant une CTE (§4.4.1), ainsi que les triggers multi-événements (§8.3.4). Côté **sécurité**, `SET SESSION AUTHORIZATION` (§10.12), les clés SSL protégées par passphrase (§10.7.4) et le logging d'audit bufferisé (§10.8.3). Côté **Galera**, le retry des write sets (§14.2.6) et la réplication parallèle inter-clusters (§13.11), elle-même utile à la migration de cluster à cluster (§19.8.7).

Aucun de ces éléments n'exige d'action avant la migration : ils enrichissent la plateforme une fois la 12.3 en place.

---

## 19.10.8 Démarche de migration recommandée

La conduite d'une migration 11.8 → 12.3 mobilise l'ensemble du chapitre dans un enchaînement logique :

1. **Inventaire et analyse de compatibilité** (§19.5). Recenser connecteurs, ORM, méthodes d'authentification et `sql_mode`, puis cibler en priorité les trois changements majeurs (§19.10.3 à 19.10.5) et le risque de plans d'exécution différents.
2. **Tests** (§19.6). Exécuter la suite fonctionnelle contre une 12.3 configurée comme la cible, en y intégrant les scénarios spécifiques : conflit d'isolation par instantané, *smoke test* des variables retirées, comparaison des plans, et validation du provisionnement Galera. La comparaison automatisée des résultats (`pt-upgrade`, Diff Router de MaxScale) débusque les écarts silencieux.
3. **Choix de la méthode et bascule** (§19.4, §19.8). Privilégier la migration par réplication 11.8 → 12.3 — la direction sûre — et une bascule sans interruption orchestrée par la couche de routage. Rappel : un cluster Galera se migre de cluster à cluster (§19.8.7), les versions majeures ne cohabitant pas.
4. **Filet de rollback** (§19.7). Disposer d'une sauvegarde vérifiée comme base ; tenir compte de la fragilité de la réplication inverse 12.3 → 11.8 due au nouveau binlog (§11.5.4). Pour les tables à versionnement système, le format 2106 partagé par les deux versions fait que ce point *n'est pas* un obstacle au rollback 11.8 ↔ 12.3 (§19.9.6).
5. **Décision go/no-go et exécution** (§19.6.11). Franchir la bascule sur la base de critères objectifs, le plan de contingence prêt.

---

## 19.10.9 Checklist de migration 11.8 → 12.3

Cette checklist consolide les actions des sections précédentes. Elle constitue un support de référence pour préparer et exécuter la migration.

**Préparation et inventaire**
- [ ] Recenser les applications, connecteurs et versions d'ORM (§19.5.2, §19.5.7)
- [ ] Comparer le `sql_mode` source et cible (§11.3)
- [ ] Identifier les tables à versionnement système et estimer leur volume (§19.9)
- [ ] Vérifier l'environnement matériel (64 bits requis pour la plage TIMESTAMP étendue, §19.9.7)

**Configuration**
- [ ] Rechercher et retirer `big_tables`, `large_page_size`, `storage_engine` des fichiers de configuration et du code (§19.10.3)
- [ ] Noter que `innodb_snapshot_isolation` est **déjà à `ON` en 11.8** : aucune décision propre au passage 11.8 → 12.3 ; ne (re)considérer sa valeur que si la source est antérieure à 11.6.2 ou MySQL (§19.10.2)
- [ ] Pour un cluster : intégrer `mariadb-server-galera` au provisionnement (§19.10.5)

**Adaptation applicative**
- [ ] Vérifier la logique de rejeu des conflits d'isolation par instantané — déjà nécessaire sous 11.8, à *ajouter* seulement si la source précède 11.6.2 ou provient de MySQL (§19.10.2)
- [ ] Vérifier l'outillage de gestion de schéma vis-à-vis du scope des noms de contraintes FK (§19.10.4)
- [ ] Valider la compatibilité des consommateurs de binlog / CDC avec le nouveau format (§11.5.4)

**Tests**
- [ ] Exécuter la suite fonctionnelle contre une 12.3 configurée comme la cible (§19.6.4)
- [ ] Comparer les résultats des requêtes (`pt-upgrade`, Diff Router) (§19.6.6, §19.6.7)
- [ ] Comparer les plans d'exécution des requêtes critiques (§19.6.8)
- [ ] Valider la connectivité par connecteur et plugin d'authentification (§19.5.5)

**Bascule**
- [ ] Mettre en place la réplication 11.8 → 12.3 et atteindre un retard nul (§19.8.4)
- [ ] Vérifier la sauvegarde antérieure et le plan de rollback (§19.7)
- [ ] Définir les critères go/no-go (§19.6.11)
- [ ] Exécuter la séquence de bascule (lecture seule, rattrapage, promotion, repointage) (§19.8.5)

**Post-migration**
- [ ] Vérifier le format des tables à versionnement système le cas échéant (§19.9.7)
- [ ] Surveiller les journaux d'erreurs et le slow query log (§19.6.10)
- [ ] Maintenir le filet de rollback jusqu'au point de non-retour décidé (§19.7.6)
- [ ] Planifier l'adoption des nouveautés non bloquantes (§19.10.7)

---

## Points clés à retenir

- La migration 11.8 → 12.3 est une montée **LTS → LTS relativement contenue** : la 12.3 consolide la série rolling 12.x, et la plupart des nouveautés sont des *ajouts* sans risque.
- **Trois changements de comportement** concentrent le risque : variables système retirées (§11.2.3), portée des noms de contraintes FK (§18.12) et packaging Galera séparé (§14.2.5). L'isolation par instantané (`innodb_snapshot_isolation = ON`) est souvent ajoutée à cette liste **par erreur** : elle est déjà active par défaut en 11.8 (depuis la 11.6.2) et n'est donc *pas* un delta 11.8 → 12.3 (§19.10.2).
- Pour les migrations venant d'un socle **antérieur à 11.6.2 ou de MySQL** (et non depuis 11.8), l'isolation par instantané devient en revanche un changement majeur : elle peut faire échouer des transactions auparavant tolérées et appelle une logique de rejeu côté application.
- Les **variables retirées** sont un risque de démarrage bloquant, facilement neutralisé par une recherche dans la configuration et le code.
- Le **nouveau binlog InnoDB** (§11.5.4) est un gain de performance, mais affecte la réplication inverse, le CDC et le PITR — donc le rollback.
- L'optimiseur 12.x pouvant choisir des **plans différents**, la comparaison `EXPLAIN` des requêtes critiques fait partie de la validation.
- La migration mobilise tout le chapitre : compatibilité (§19.5), tests (§19.6), rollback (§19.7), bascule sans interruption (§19.8) et tables temporelles (§19.9), réunis par la démarche et la checklist ci-dessus.

> 🔗 **Pour aller plus loin** : §6.9 *Snapshot Isolation*, §11.2.3 *Variables retirées*, §18.12 *Contraintes FK*, §14.2.5 *Packaging Galera*, §11.5.4 *Binlog intégré à InnoDB*, et l'ensemble du chapitre 19 dont cette section constitue la synthèse. Pour une vue d'ensemble des nouveautés 12.3, voir l'**Annexe F** *Nouveautés MariaDB 12.3 LTS en un Coup d'Œil*.

⏭️ [Cas d'Usage et Architectures](/20-cas-usage-architectures/README.md)

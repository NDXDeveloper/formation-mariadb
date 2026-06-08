🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.5 Gestion des migrations de schéma

Le schéma d'une base **évolue avec le code** : nouvelle table, colonne ajoutée, index créé, contrainte modifiée. Gérer ces évolutions à la main — en appliquant des `ALTER TABLE` directement sur chaque environnement — ne passe pas à l'échelle : ce n'est ni reproductible, ni traçable, et cela produit fatalement des écarts entre développement, test et production. Les **migrations de schéma** apportent la discipline qui résout ce problème.

Cette section adopte le point de vue du **développeur** : principes, compatibilité, bonnes pratiques. L'outillage dédié (Flyway, Liquibase, gh-ost, pt-online-schema-change) est détaillé au chap. 16.8 ; les migrations intégrées aux ORM ont été vues dans les sections 17.3.x (Alembic, Prisma Migrate, migrations EF Core, sequelize-cli).

---

## Pourquoi des migrations ?

Modifier le schéma à la main pose plusieurs problèmes : aucune **trace** de ce qui a été appliqué et où, une **dérive** progressive entre les environnements, l'impossibilité de **rejouer** l'historique sur une base neuve, et un risque d'erreur élevé. Les migrations répondent à ces besoins en transformant chaque évolution du schéma en un **artefact versionné, ordonné et reproductible**.

---

## Les principes

Quel que soit l'outil, une bonne gestion des migrations repose sur les mêmes fondations :

- **Des scripts versionnés et ordonnés** — chaque changement est une migration numérotée/horodatée, appliquée dans un ordre déterministe.
- **Dans le contrôle de version, avec le code** — les migrations vivent dans le dépôt, évoluent et sont relues comme du code.
- **Appliquées de façon cohérente** sur tous les environnements (dev, test, *staging*, prod), par le même mécanisme.
- **Un historique des migrations appliquées** — l'outil tient une table de suivi (par exemple `flyway_schema_history`) qui sait ce qui a déjà été exécuté, et n'applique que les migrations en attente.
- **Une migration = un changement** focalisé, et **jamais de modification d'une migration déjà appliquée** : on crée une nouvelle migration pour corriger.

---

## Forward-only ou réversible (up/down) ?

Deux philosophies coexistent. Certaines équipes écrivent des migrations **réversibles** (un volet *up* et un volet *down* permettant de revenir en arrière) ; EF Core et Alembic le permettent. D'autres adoptent une approche **forward-only** : pour annuler un changement, on écrit une **nouvelle migration** qui le défait — c'est notamment le modèle de Prisma (§17.3.4). En pratique, le *rollback* automatique est souvent illusoire pour les changements destructeurs (une colonne supprimée ne se « dé-supprime » pas) : beaucoup privilégient le forward-only, en s'appuyant sur les sauvegardes (chap. 12) pour les cas extrêmes.

---

## Le paysage des outils

On distingue trois familles, complémentaires :

- **Intégrés aux ORM** — Alembic (§17.3.2), Prisma Migrate (§17.3.4), migrations EF Core (§17.3.5), sequelize-cli (§17.3.3). Souvent capables de **dériver** la migration des changements de modèle (sauf Sequelize). Pratiques quand l'application est centrée sur un ORM.
- **Autonomes, centrés base** — **Flyway** (§16.8.1) et **Liquibase** (§16.8.2), qui appliquent des scripts SQL (ou XML/YAML) indépendamment du langage. Adaptés aux équipes polyglottes ou à une gouvernance du schéma découplée du code applicatif.
- **Changement de schéma en ligne** — **gh-ost** et **pt-online-schema-change** (§16.8.3), pour modifier de **très grandes tables sans les bloquer**.

---

## Le défi : modifier sans interruption

Sur une table volumineuse en production, un `ALTER TABLE` naïf peut verrouiller la table et provoquer une indisponibilité. Plusieurs leviers l'évitent :

- **L'Online Schema Change de MariaDB** (chap. 18.11) — les `ALTER TABLE` peuvent s'exécuter avec `ALGORITHM=INPLACE` ou `INSTANT` et `LOCK=NONE`, certaines opérations (ajout de colonne en fin de table, par exemple) étant **instantanées**.
- **Les outils externes** (gh-ost, pt-osc, §16.8.3) — lorsque l'opération n'est pas réalisable en ligne ou que la table est trop grosse, ils créent une table fantôme, recopient les données, puis basculent.
- **Le motif *expand/contract*** (changement parallèle) — pour un déploiement sans interruption, on découpe le changement en étapes compatibles (voir ci-dessous), en lien avec les migrations *zero-downtime* (§19.8).

---

## Compatibilité ascendante et descendante

Point crucial souvent négligé : lors d'un déploiement progressif (*rolling deploy*), **l'ancien et le nouveau code tournent simultanément** quelques instants. Le changement de schéma doit donc rester **compatible avec les deux versions** du code. Une conséquence directe : une opération en apparence simple comme **renommer une colonne n'est pas une migration unique sûre**. On procède par étapes (motif *expand/contract*) :

1. **Ajouter** la nouvelle colonne sans toucher à l'ancienne — compatible avec l'ancien comme le nouveau code.
2. **Écrire dans les deux** (double écriture) et **recopier** l'existant (*backfill*).
3. **Basculer les lectures** vers la nouvelle colonne (déploiement suivant).
4. **Cesser d'écrire** l'ancienne, puis la **supprimer** (déploiement final).

Ce qui ressemblait à un `RENAME` devient ainsi plusieurs migrations réparties sur plusieurs déploiements — c'est le prix de l'absence d'interruption.

---

## Spécificités MariaDB

- **DDL non transactionnel** — contrairement à PostgreSQL, le DDL de MariaDB/MySQL provoque un **commit implicite** et **ne s'annule pas** dans une transaction comme le DML. Une migration enchaînant plusieurs DDL et échouant en cours de route peut donc laisser le schéma dans un **état partiel** ; l'historique de l'outil aide à reprendre, mais il faut concevoir les migrations en gardant cette absence d'atomicité à l'esprit.
- **Online Schema Change** (chap. 18.11) et opérations **INSTANT** — à exploiter pour les ALTER sur grandes tables.
- Les fonctionnalités spécifiques (séquences, tables temporelles, VECTOR — chap. 18) se déclarent en SQL natif dans les migrations.

---

## Bonnes pratiques

- **Petites migrations focalisées** — un changement par migration, faciles à relire et à appliquer.
- **Relire les migrations autogénérées** — Alembic, Prisma et EF Core proposent une migration dérivée du modèle ; elle doit être **vérifiée** avant application (§17.3.x).
- **Tester les migrations** (application, et *rollback* le cas échéant) sur un environnement non productif avant la prod (§17.6, §17.7).
- **Séparer migrations de schéma et migrations de données** quand c'est possible, pour limiter les risques et le temps de verrou.
- **Jamais d'auto-synchronisation en production** — `ddl-auto=update`/`create` (Hibernate, §17.3.1), `sync({ force/alter })` (Sequelize, §17.3.3), `EnsureCreated()` (EF Core, §17.3.5) sont à proscrire : on passe par des migrations versionnées.
- **Coordonner les changements destructeurs avec les déploiements** — ne jamais supprimer une colonne ou une table dont du code en cours d'exécution a encore besoin.

---

## Ce qu'il faut retenir

- Le schéma évolue avec le code : gérer ses changements par des **migrations versionnées, ordonnées, dans le dépôt** et appliquées de façon cohérente partout, avec un **historique** des migrations.
- Trois familles d'outils : **intégrés aux ORM** (17.3.x), **autonomes** (Flyway/Liquibase, §16.8.1/.2), **en ligne** (gh-ost/pt-osc, §16.8.3).
- Sur grandes tables, éviter le verrou via l'**Online Schema Change** (chap. 18.11) ou les outils externes ; viser le **zéro interruption** (§19.8).
- Penser **compatibilité ascendante/descendante** : un renommage se fait par **expand/contract** sur plusieurs déploiements.
- Spécificité MariaDB : **DDL non transactionnel** (commit implicite, pas de *rollback*) — attention à l'état partiel.
- **Petites migrations relues**, testées hors prod, **jamais d'auto-sync en production** (§17.3.x), changements destructeurs coordonnés.

⏭️ [Tests de bases de données](/17-integration-developpement/06-tests-bases-donnees.md)

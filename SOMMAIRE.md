# 📚 Sommaire - Formation MariaDB 11.8 LTS
**Un guide progressif pour découvrir et approfondir MariaDB — Développeurs & DevOps**

---

## 🎯 Parcours de Formation

| Parcours | Modules | Durée estimée |
|----------|---------|---------------|
| **Développeur** | 1-6, 8-9, 17-18 | 5-7 jours |
| **Administrateur/DBA** | 1, 7, 10-15, 19 | 7-10 jours |
| **DevOps/Cloud** | 1, 11-12, 14, 16, 20 | 4-6 jours |
| **IA/ML** | 1, 4 (JSON/Vector), 17-18, 20 | 3-4 jours |
| **Formation complète** | 1-20 | 15-20 jours |

---

## **[Partie 1 : Introduction et Fondamentaux (Débutant)](/partie-01-introduction-fondamentaux.md)**

### 1. [Introduction et Fondamentaux](01-introduction-fondamentaux/README.md)
- 1.1 [Qu'est-ce que MariaDB ?](01-introduction-fondamentaux/01-quest-ce-que-mariadb.md)
- 1.2 [Histoire et différences avec MySQL](01-introduction-fondamentaux/02-histoire-et-differences-mysql.md)
- 1.3 [Cas d'usage et écosystème](01-introduction-fondamentaux/03-cas-usage-et-ecosysteme.md)
- 1.4 [Architecture générale d'un SGBD relationnel](01-introduction-fondamentaux/04-architecture-generale-sgbd.md)
- 1.5 [Politique de versions : LTS (11.4, 11.8) vs Rolling releases](01-introduction-fondamentaux/05-politique-versions-lts-rolling.md) 🔄
- **1.6 [Cycle de support : 3 ans LTS (depuis 11.4), rolling trimestriel](01-introduction-fondamentaux/06-cycle-support-lts.md)** 🆕
- **1.7 [Roadmap : série 12.x (12.0→12.2 rolling, 12.3 LTS prévu Q2 2026)](01-introduction-fondamentaux/07-roadmap-serie-12.md)** 🆕
- 1.8 [Installation et configuration initiale](01-introduction-fondamentaux/08-installation-configuration.md)
- 1.9 [Outils d'administration (CLI, HeidiSQL, DBeaver, phpMyAdmin)](01-introduction-fondamentaux/09-outils-administration.md)

### 2. [Bases du SQL](02-bases-du-sql/README.md)
- 2.1 [Introduction au langage SQL](02-bases-du-sql/01-introduction-langage-sql.md)
- 2.2 [Types de données MariaDB](02-bases-du-sql/02-types-de-donnees.md)
    - 2.2.1 [Numériques (INT, BIGINT, DECIMAL, FLOAT, DOUBLE)](02-bases-du-sql/02.1-types-numeriques.md)
    - 2.2.2 [Texte (VARCHAR, TEXT, CHAR, ENUM, SET)](02-bases-du-sql/02.2-types-texte.md)
    - 2.2.3 [Temporels (DATE, DATETIME, TIMESTAMP, TIME, YEAR)](02-bases-du-sql/02.3-types-temporels.md)
    - 2.2.4 [Binaires (BLOB, BINARY, VARBINARY)](02-bases-du-sql/02.4-types-binaires.md)
    - 2.2.5 [Spécifiques MariaDB (JSON, UUID, INET6)](02-bases-du-sql/02.5-types-specifiques-mariadb.md)
- 2.3 [Création et gestion des bases de données](02-bases-du-sql/03-creation-gestion-bases.md)
- 2.4 [Création et modification de tables (CREATE, ALTER, DROP)](02-bases-du-sql/04-creation-modification-tables.md)
- 2.5 [Contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT)](02-bases-du-sql/05-contraintes.md)
- 2.6 [Insertion de données (INSERT, INSERT INTO SELECT, LOAD DATA)](02-bases-du-sql/06-insertion-donnees.md)
- 2.7 [Requêtes de sélection simples (SELECT, WHERE, ORDER BY, LIMIT)](02-bases-du-sql/07-requetes-selection-simples.md)
- 2.8 [Mise à jour et suppression de données (UPDATE, DELETE, TRUNCATE)](02-bases-du-sql/08-mise-a-jour-suppression.md)

---

## **[Partie 2 : Requêtes SQL Intermédiaires et Avancées (Intermédiaire)](/partie-02-requetes-sql-intermediaires-avancees.md)**

### 3. [Requêtes SQL Intermédiaires](03-requetes-sql-intermediaires/README.md)
- 3.1 [Fonctions d'agrégation (COUNT, SUM, AVG, MIN, MAX)](03-requetes-sql-intermediaires/01-fonctions-agregation.md)
- 3.2 [Regroupement de données (GROUP BY, HAVING)](03-requetes-sql-intermediaires/02-regroupement-donnees.md)
- 3.3 [Jointures](03-requetes-sql-intermediaires/03-jointures.md)
    - 3.3.1 [INNER JOIN : Intersection](03-requetes-sql-intermediaires/03.1-inner-join.md)
    - 3.3.2 [LEFT/RIGHT JOIN : Jointures externes](03-requetes-sql-intermediaires/03.2-left-right-join.md)
    - 3.3.3 [CROSS JOIN : Produit cartésien](03-requetes-sql-intermediaires/03.3-cross-join.md)
    - 3.3.4 [Self-Join : Joindre une table à elle-même](03-requetes-sql-intermediaires/03.4-self-join.md)
- 3.4 [Sous-requêtes et requêtes imbriquées](03-requetes-sql-intermediaires/04-sous-requetes.md)
- 3.5 [Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)](03-requetes-sql-intermediaires/05-operateurs-ensemblistes.md)
- 3.6 [Fonctions de chaînes de caractères](03-requetes-sql-intermediaires/06-fonctions-chaines.md)
- 3.7 [Fonctions de dates et heures](03-requetes-sql-intermediaires/07-fonctions-dates-heures.md)
- 3.8 [Expressions conditionnelles (CASE, IF, IFNULL, COALESCE, NULLIF)](03-requetes-sql-intermediaires/08-expressions-conditionnelles.md)

### 4. [Concepts Avancés SQL](04-concepts-avances-sql/README.md)
- 4.1 [Requêtes récursives (WITH RECURSIVE)](04-concepts-avances-sql/01-requetes-recursives.md)
- 4.2 [Window Functions](04-concepts-avances-sql/02-window-functions.md)
    - 4.2.1 [Fonctions de rang (ROW_NUMBER, RANK, DENSE_RANK, NTILE)](04-concepts-avances-sql/02.1-fonctions-rang.md)
    - 4.2.2 [Fonctions de valeur (LAG, LEAD, FIRST_VALUE, LAST_VALUE)](04-concepts-avances-sql/02.2-fonctions-valeur.md)
    - 4.2.3 [Frames de fenêtre (ROWS, RANGE, GROUPS)](04-concepts-avances-sql/02.3-frames-fenetre.md)
    - 4.2.4 [Cas d'usage : Top N, moyenne mobile, cumuls](04-concepts-avances-sql/02.4-cas-usage-window.md)
- 4.3 [Requêtes pivotées et transformations](04-concepts-avances-sql/03-requetes-pivotees.md)
- 4.4 [Expressions de table communes (CTE)](04-concepts-avances-sql/04-expressions-table-communes.md)
- 4.5 [Requêtes complexes multi-tables](04-concepts-avances-sql/05-requetes-complexes-multi-tables.md)
- 4.6 [Gestion des valeurs NULL : Logique ternaire](04-concepts-avances-sql/06-gestion-valeurs-null.md)
- 4.7 [JSON dans MariaDB](04-concepts-avances-sql/07-json-mariadb.md)
    - 4.7.1 [Stockage et type de données JSON](04-concepts-avances-sql/07.1-stockage-type-json.md)
    - 4.7.2 [Fonctions JSON (JSON_EXTRACT, JSON_SET, JSON_ARRAY, etc.)](04-concepts-avances-sql/07.2-fonctions-json.md)
    - 4.7.3 [Opérateur raccourci (->>)](04-concepts-avances-sql/07.3-operateur-raccourci-json.md)
- **4.8 [JSON Path Expressions avancées](04-concepts-avances-sql/08-json-path-expressions.md)** 🆕
- **4.9 [JSON Schema Validation](04-concepts-avances-sql/09-json-schema-validation.md)** 🆕
- 4.10 [Indexation de colonnes virtuelles extraites du JSON](04-concepts-avances-sql/10-indexation-colonnes-virtuelles-json.md)
- 4.11 [Expressions régulières (REGEXP, REGEXP_REPLACE, REGEXP_SUBSTR)](04-concepts-avances-sql/11-expressions-regulieres.md)

---

## **[Partie 3 : Index, Transactions et Performance (Intermédiaire)](/partie-03-index-transactions-performance.md)**

### 5. [Index et Performance](05-index-et-performance/README.md)
- 5.1 [Fonctionnement des index : Structure B-Tree](05-index-et-performance/01-fonctionnement-index.md)
- 5.2 [Types d'index](05-index-et-performance/02-types-index.md)
    - 5.2.1 [B-Tree : Le standard](05-index-et-performance/02.1-btree.md)
    - 5.2.2 [Hash : Égalité stricte](05-index-et-performance/02.2-hash.md)
    - 5.2.3 [Full-Text : Recherche textuelle](05-index-et-performance/02.3-full-text.md)
    - 5.2.4 [Spatial : Données géographiques](05-index-et-performance/02.4-spatial.md)
- **5.3 [Index VECTOR (HNSW) pour la recherche vectorielle](05-index-et-performance/03-index-vector-hnsw.md)** 🆕
- 5.4 [Création et gestion des index](05-index-et-performance/04-creation-gestion-index.md)
- 5.5 [Stratégies d'indexation](05-index-et-performance/05-strategies-indexation.md)
    - 5.5.1 [Index sur colonnes fréquemment filtrées](05-index-et-performance/05.1-index-colonnes-filtrees.md)
    - 5.5.2 [Index sur clés étrangères](05-index-et-performance/05.2-index-cles-etrangeres.md)
    - 5.5.3 [Index pour ORDER BY et GROUP BY](05-index-et-performance/05.3-index-order-group.md)
- 5.6 [Index composites et ordre des colonnes](05-index-et-performance/06-index-composites.md)
- 5.7 [Analyse des plans d'exécution (EXPLAIN, EXPLAIN ANALYZE)](05-index-et-performance/07-analyse-plans-execution.md)
- 5.8 [Optimisation des requêtes](05-index-et-performance/08-optimisation-requetes.md)
- 5.9 [Index covering et index-only scans](05-index-et-performance/09-index-covering.md)
- 5.10 [Invisible indexes et Progressive indexes](05-index-et-performance/10-invisible-progressive-indexes.md)

### 6. [Transactions et Concurrence](06-transactions-et-concurrence/README.md)
- 6.1 [Concept de transaction et propriétés ACID](06-transactions-et-concurrence/01-concept-transaction-acid.md)
- 6.2 [Gestion des transactions (START TRANSACTION, BEGIN, COMMIT, ROLLBACK)](06-transactions-et-concurrence/02-gestion-transactions.md)
- 6.3 [Niveaux d'isolation](06-transactions-et-concurrence/03-niveaux-isolation.md)
    - 6.3.1 [READ UNCOMMITTED : Dirty reads possibles](06-transactions-et-concurrence/03.1-read-uncommitted.md)
    - 6.3.2 [READ COMMITTED : Lectures cohérentes](06-transactions-et-concurrence/03.2-read-committed.md)
    - 6.3.3 [REPEATABLE READ : Default InnoDB](06-transactions-et-concurrence/03.3-repeatable-read.md)
    - 6.3.4 [SERIALIZABLE : Isolation maximale](06-transactions-et-concurrence/03.4-serializable.md)
- 6.4 [Verrous (LOCK TABLES, SELECT FOR UPDATE, SELECT FOR SHARE)](06-transactions-et-concurrence/04-verrous.md)
- 6.5 [Deadlocks : Détection et résolution](06-transactions-et-concurrence/05-deadlocks-resolution.md)
- 6.6 [MVCC (Multi-Version Concurrency Control)](06-transactions-et-concurrence/06-mvcc.md)
- 6.7 [Savepoints : Points de sauvegarde](06-transactions-et-concurrence/07-savepoints.md)
- 6.8 [Transactions distribuées (XA)](06-transactions-et-concurrence/08-transactions-distribuees-xa.md)

---

## **[Partie 4 : Moteurs de Stockage et Programmation Serveur (Avancé)](/partie-04-moteurs-stockage-programmation.md)**

### 7. [Moteurs de Stockage](07-moteurs-de-stockage/README.md)
- 7.1 [Vue d'ensemble : Architecture Pluggable Storage Engine](07-moteurs-de-stockage/01-vue-ensemble-architecture.md)
- 7.2 [InnoDB : Le moteur par défaut](07-moteurs-de-stockage/02-innodb.md)
    - 7.2.1 [Caractéristiques : ACID, FK, Row-level locking](07-moteurs-de-stockage/02.1-innodb-caracteristiques.md)
    - 7.2.2 [Buffer Pool et gestion mémoire](07-moteurs-de-stockage/02.2-innodb-buffer-pool.md)
    - 7.2.3 [Redo Log et Undo Log](07-moteurs-de-stockage/02.3-innodb-redo-undo-log.md)
    - 7.2.4 [Configuration avancée](07-moteurs-de-stockage/02.4-innodb-configuration.md)
- 7.3 [MyISAM : Moteur legacy](07-moteurs-de-stockage/03-myisam.md)
- 7.4 [Aria : Le successeur de MyISAM](07-moteurs-de-stockage/04-aria.md)
- 7.5 [ColumnStore : Analytique et OLAP](07-moteurs-de-stockage/05-columnstore.md)
- 7.6 [Moteur S3 : Archivage données froides sur AWS S3/MinIO](07-moteurs-de-stockage/06-moteur-s3.md)
- **7.7 [Moteur Vector/HNSW : Recherche vectorielle pour IA](07-moteurs-de-stockage/07-moteur-vector-hnsw.md)** 🆕
- 7.8 [Comparaison et choix du moteur approprié](07-moteurs-de-stockage/08-comparaison-choix-moteur.md)
- 7.9 [Conversion entre moteurs (ALTER TABLE ENGINE)](07-moteurs-de-stockage/09-conversion-entre-moteurs.md)
- 7.10 [Moteurs spécialisés](07-moteurs-de-stockage/10-moteurs-specialises.md)
    - 7.10.1 [Memory : Tables en RAM](07-moteurs-de-stockage/10.1-memory.md)
    - 7.10.2 [Archive : Compression maximale](07-moteurs-de-stockage/10.2-archive.md)
    - 7.10.3 [Spider : Sharding distribué](07-moteurs-de-stockage/10.3-spider.md)
    - 7.10.4 [CONNECT : Accès données externes](07-moteurs-de-stockage/10.4-connect.md)

### 8. [Programmation Côté Serveur](08-programmation-cote-serveur/README.md)
- 8.1 [Procédures stockées](08-programmation-cote-serveur/01-procedures-stockees.md)
    - 8.1.1 [Syntaxe CREATE PROCEDURE](08-programmation-cote-serveur/01.1-syntaxe-create-procedure.md)
    - 8.1.2 [Paramètres IN, OUT, INOUT](08-programmation-cote-serveur/01.2-parametres-in-out-inout.md)
    - 8.1.3 [Appel avec CALL](08-programmation-cote-serveur/01.3-appel-call.md)
- 8.2 [Fonctions stockées](08-programmation-cote-serveur/02-fonctions-stockees.md)
    - 8.2.1 [Syntaxe CREATE FUNCTION](08-programmation-cote-serveur/02.1-syntaxe-create-function.md)
    - 8.2.2 [Caractéristiques (DETERMINISTIC, NO SQL, etc.)](08-programmation-cote-serveur/02.2-caracteristiques-fonction.md)
- 8.3 [Triggers (déclencheurs)](08-programmation-cote-serveur/03-triggers.md)
    - 8.3.1 [BEFORE et AFTER](08-programmation-cote-serveur/03.1-before-after.md)
    - 8.3.2 [INSERT, UPDATE, DELETE triggers](08-programmation-cote-serveur/03.2-insert-update-delete-triggers.md)
    - 8.3.3 [Variables OLD et NEW](08-programmation-cote-serveur/03.3-variables-old-new.md)
- 8.4 [Events (tâches planifiées)](08-programmation-cote-serveur/04-events.md)
    - 8.4.1 [CREATE EVENT et planification](08-programmation-cote-serveur/04.1-create-event.md)
    - 8.4.2 [Event Scheduler](08-programmation-cote-serveur/04.2-event-scheduler.md)
- 8.5 [Curseurs](08-programmation-cote-serveur/05-curseurs.md)
- 8.6 [Gestion des erreurs et exceptions (DECLARE HANDLER)](08-programmation-cote-serveur/06-gestion-erreurs-exceptions.md)
- 8.7 [Variables et flow control (IF, CASE, LOOP, WHILE, REPEAT)](08-programmation-cote-serveur/07-variables-flow-control.md)
- 8.8 [Bonnes pratiques de programmation](08-programmation-cote-serveur/08-bonnes-pratiques.md)

### 9. [Vues et Données Virtuelles](09-vues-et-donnees-virtuelles/README.md)
- 9.1 [Création et gestion des vues (CREATE VIEW, ALTER VIEW)](09-vues-et-donnees-virtuelles/01-creation-gestion-vues.md)
- 9.2 [Vues matérialisées : Alternatives et workarounds](09-vues-et-donnees-virtuelles/02-vues-materialisees.md)
- 9.3 [Vues updatable : Conditions et limitations](09-vues-et-donnees-virtuelles/03-vues-updatable.md)
- 9.4 [WITH CHECK OPTION](09-vues-et-donnees-virtuelles/04-with-check-option.md)
- 9.5 [Sécurité et vues : Masquage de données](09-vues-et-donnees-virtuelles/05-securite-et-vues.md)
- 9.6 [Performance des vues : MERGE vs TEMPTABLE](09-vues-et-donnees-virtuelles/06-performance-vues.md)
- 9.7 [Vues système](09-vues-et-donnees-virtuelles/07-vues-systeme.md)
    - 9.7.1 [INFORMATION_SCHEMA](09-vues-et-donnees-virtuelles/07.1-information-schema.md)
    - 9.7.2 [PERFORMANCE_SCHEMA](09-vues-et-donnees-virtuelles/07.2-performance-schema.md)
    - 9.7.3 [mysql system tables](09-vues-et-donnees-virtuelles/07.3-mysql-system-tables.md)

---

## **[Partie 5 : Sécurité et Administration (DBA)](/partie-05-securite-administration.md)**

### 10. [Sécurité et Gestion des Utilisateurs](10-securite-gestion-utilisateurs/README.md)
- 10.1 [Modèle de sécurité MariaDB](10-securite-gestion-utilisateurs/01-modele-securite.md)
- 10.2 [Création et gestion des utilisateurs (CREATE USER, ALTER USER, DROP USER)](10-securite-gestion-utilisateurs/02-creation-gestion-utilisateurs.md)
- 10.3 [Système de privilèges](10-securite-gestion-utilisateurs/03-systeme-privileges.md)
    - 10.3.1 [GRANT : Attribution de privilèges](10-securite-gestion-utilisateurs/03.1-grant.md)
    - 10.3.2 [REVOKE : Révocation de privilèges](10-securite-gestion-utilisateurs/03.2-revoke.md)
    - 10.3.3 [Privilèges globaux, base, table, colonne](10-securite-gestion-utilisateurs/03.3-niveaux-privileges.md)
- 10.4 [Rôles (CREATE ROLE, SET ROLE, DEFAULT ROLE)](10-securite-gestion-utilisateurs/04-roles.md)
- 10.5 [Authentification : Plugins](10-securite-gestion-utilisateurs/05-authentification-plugins.md)
    - 10.5.1 [mysql_native_password](10-securite-gestion-utilisateurs/05.1-mysql-native-password.md)
    - 10.5.2 [ed25519 : Authentification moderne](10-securite-gestion-utilisateurs/05.2-ed25519.md)
    - 10.5.3 [PAM et LDAP](10-securite-gestion-utilisateurs/05.3-pam-ldap.md)
    - 10.5.4 [GSSAPI/Kerberos](10-securite-gestion-utilisateurs/05.4-gssapi-kerberos.md)
- **10.6 [Plugin d'authentification PARSEC](10-securite-gestion-utilisateurs/06-plugin-parsec.md)** 🆕
- 10.7 [Chiffrement des connexions (SSL/TLS)](10-securite-gestion-utilisateurs/07-chiffrement-ssl-tls.md) 🔄
    - 10.7.1 [Configuration serveur SSL](10-securite-gestion-utilisateurs/07.1-configuration-serveur-ssl.md)
    - 10.7.2 [Certificats et CA](10-securite-gestion-utilisateurs/07.2-certificats-ca.md)
    - **10.7.3 [TLS par défaut depuis 11.8](10-securite-gestion-utilisateurs/07.3-tls-defaut-11-8.md)** 🆕
- 10.8 [Audit et logging](10-securite-gestion-utilisateurs/08-audit-logging.md)
    - 10.8.1 [Server Audit Plugin](10-securite-gestion-utilisateurs/08.1-server-audit-plugin.md)
    - 10.8.2 [Audit de connexions et requêtes](10-securite-gestion-utilisateurs/08.2-audit-connexions-requetes.md)
- 10.9 [Sécurité au niveau application](10-securite-gestion-utilisateurs/09-securite-niveau-application.md)
- 10.10 [Password validation plugins et politiques](10-securite-gestion-utilisateurs/10-password-validation.md)
- **10.11 [Privilèges granulaires (nouvelles options 11.8)](10-securite-gestion-utilisateurs/11-privileges-granulaires.md)** 🆕

### 11. [Administration et Configuration](11-administration-configuration/README.md)
- 11.1 [Fichiers de configuration](11-administration-configuration/01-fichiers-configuration.md)
    - 11.1.1 [my.cnf / my.ini : Structure et sections](11-administration-configuration/01.1-structure-mycnf.md)
    - 11.1.2 [Ordre de lecture des fichiers](11-administration-configuration/01.2-ordre-lecture.md)
    - 11.1.3 [Include et configuration modulaire](11-administration-configuration/01.3-include-modulaire.md)
- 11.2 [Variables système et de session](11-administration-configuration/02-variables-systeme-session.md)
    - 11.2.1 [SHOW VARIABLES et SET](11-administration-configuration/02.1-show-variables-set.md)
    - 11.2.2 [Variables dynamiques vs statiques](11-administration-configuration/02.2-dynamiques-statiques.md)
- 11.3 [Modes SQL (sql_mode)](11-administration-configuration/03-modes-sql.md)
- 11.4 [Gestion des logs](11-administration-configuration/04-gestion-logs.md)
    - 11.4.1 [Error log](11-administration-configuration/04.1-error-log.md)
    - 11.4.2 [Slow query log](11-administration-configuration/04.2-slow-query-log.md)
    - 11.4.3 [General log](11-administration-configuration/04.3-general-log.md)
- 11.5 [Binary logs et logs de transactions](11-administration-configuration/05-binary-logs.md)
    - 11.5.1 [Configuration binlog](11-administration-configuration/05.1-configuration-binlog.md)
    - 11.5.2 [Formats : STATEMENT, ROW, MIXED](11-administration-configuration/05.2-formats-binlog.md)
    - 11.5.3 [Purge et rotation](11-administration-configuration/05.3-purge-rotation.md)
- 11.6 [Maintenance des tables](11-administration-configuration/06-maintenance-tables.md)
    - 11.6.1 [OPTIMIZE TABLE](11-administration-configuration/06.1-optimize-table.md)
    - 11.6.2 [ANALYZE TABLE](11-administration-configuration/06.2-analyze-table.md)
    - 11.6.3 [CHECK TABLE et REPAIR TABLE](11-administration-configuration/06.3-check-repair-table.md)
- 11.7 [Gestion de l'espace disque](11-administration-configuration/07-gestion-espace-disque.md)
- **11.8 [Contrôle espace temporaire (max_tmp_space_usage, max_total_tmp_space_usage)](11-administration-configuration/08-controle-espace-temporaire.md)** 🆕
- 11.9 [Monitoring et métriques importantes](11-administration-configuration/09-monitoring-metriques.md)
- 11.10 [Thread Pool et gestion de la concurrence](11-administration-configuration/10-thread-pool.md)
- **11.11 [Charset par défaut : utf8mb4 avec collations UCA 14.0.0 (depuis 11.8)](11-administration-configuration/11-charset-utf8mb4-uca14.md)** 🆕
- **11.12 [Extension TIMESTAMP 2038→2106 (problème Y2038 résolu)](11-administration-configuration/12-extension-timestamp-2106.md)** 🆕

### 12. [Sauvegarde et Restauration](12-sauvegarde-restauration/README.md)
- 12.1 [Stratégies de sauvegarde : Full, Incrémentale, Différentielle](12-sauvegarde-restauration/01-strategies-sauvegarde.md)
- 12.2 [Sauvegarde logique](12-sauvegarde-restauration/02-sauvegarde-logique.md)
    - 12.2.1 [mysqldump / mariadb-dump](12-sauvegarde-restauration/02.1-mysqldump-mariadb-dump.md)
    - 12.2.2 [Options essentielles (--single-transaction, --routines, etc.)](12-sauvegarde-restauration/02.2-options-essentielles.md)
    - 12.2.3 [mydumper/myloader : Parallélisme](12-sauvegarde-restauration/02.3-mydumper-myloader.md)
- 12.3 [Sauvegarde physique (Mariabackup)](12-sauvegarde-restauration/03-sauvegarde-physique-mariabackup.md) 🔄
    - 12.3.1 [Full backup](12-sauvegarde-restauration/03.1-full-backup.md)
    - 12.3.2 [Incremental backup](12-sauvegarde-restauration/03.2-incremental-backup.md)
    - **12.3.3 [Support BACKUP STAGE](12-sauvegarde-restauration/03.3-backup-stage.md)** 🆕
- 12.4 [Sauvegarde incrémentale avec binary logs](12-sauvegarde-restauration/04-sauvegarde-incrementale-binlog.md)
- 12.5 [Restauration](12-sauvegarde-restauration/05-restauration.md)
    - 12.5.1 [Restauration complète](12-sauvegarde-restauration/05.1-restauration-complete.md)
    - 12.5.2 [Point-in-time recovery (PITR)](12-sauvegarde-restauration/05.2-pitr.md)
- 12.6 [Automatisation des sauvegardes](12-sauvegarde-restauration/06-automatisation-sauvegardes.md)
- 12.7 [Tests de restauration et plan de reprise (PRA)](12-sauvegarde-restauration/07-tests-restauration-pra.md)
- 12.8 [Sauvegarde cloud-native](12-sauvegarde-restauration/08-sauvegarde-cloud-native.md)
    - 12.8.1 [S3 et Object Storage](12-sauvegarde-restauration/08.1-s3-object-storage.md)
    - 12.8.2 [Kubernetes VolumeSnapshots](12-sauvegarde-restauration/08.2-kubernetes-volumesnapshots.md)

---

## **[Partie 6 : Réplication et Haute Disponibilité (DBA/DevOps)](/partie-06-replication-haute-disponibilite.md)**

### 13. [Réplication](13-replication/README.md)
- 13.1 [Concepts de réplication : Asynchrone vs Semi-synchrone](13-replication/01-concepts-replication.md)
- 13.2 [Réplication Master-Slave (Source-Replica)](13-replication/02-replication-master-slave.md)
    - 13.2.1 [Configuration du Primary (binlog)](13-replication/02.1-configuration-primary.md)
    - 13.2.2 [Configuration du Replica](13-replication/02.2-configuration-replica.md)
    - 13.2.3 [CHANGE MASTER TO / CHANGE REPLICATION SOURCE](13-replication/02.3-change-master-to.md)
- 13.3 [Réplication basée sur les positions (binlog coordinates)](13-replication/03-replication-positions.md)
- 13.4 [GTID (Global Transaction Identifier)](13-replication/04-gtid.md)
    - 13.4.1 [Configuration GTID](13-replication/04.1-configuration-gtid.md)
    - 13.4.2 [Avantages pour failover](13-replication/04.2-avantages-failover.md)
- 13.5 [Réplication multi-source](13-replication/05-replication-multi-source.md)
- 13.6 [Réplication en cascade](13-replication/06-replication-cascade.md)
- 13.7 [Monitoring et troubleshooting](13-replication/07-monitoring-troubleshooting.md)
    - 13.7.1 [SHOW SLAVE STATUS / SHOW REPLICA STATUS](13-replication/07.1-show-slave-status.md)
    - 13.7.2 [Seconds_Behind_Master et lag](13-replication/07.2-seconds-behind-master.md)
    - 13.7.3 [Erreurs courantes et résolution](13-replication/07.3-erreurs-courantes.md)
- 13.8 [Failover et switchover](13-replication/08-failover-switchover.md)
- 13.9 [Réplication semi-synchrone](13-replication/09-replication-semi-synchrone.md)
- **13.10 [Optimistic ALTER TABLE pour réduction du lag](13-replication/10-optimistic-alter-table.md)** 🆕

### 14. [Haute Disponibilité](14-haute-disponibilite/README.md)
- 14.1 [Architectures haute disponibilité : Concepts](14-haute-disponibilite/01-architectures-ha-concepts.md)
- 14.2 [MariaDB Galera Cluster](14-haute-disponibilite/02-galera-cluster.md)
    - 14.2.1 [Architecture synchrone multi-master](14-haute-disponibilite/02.1-architecture-synchrone.md)
    - 14.2.2 [Certification-based replication](14-haute-disponibilite/02.2-certification-based.md)
    - 14.2.3 [Configuration et déploiement](14-haute-disponibilite/02.3-configuration-deploiement.md)
    - 14.2.4 [State transfers (SST, IST)](14-haute-disponibilite/02.4-state-transfers.md)
- 14.3 [Split-brain et quorum](14-haute-disponibilite/03-split-brain-quorum.md)
- 14.4 [MaxScale](14-haute-disponibilite/04-maxscale.md) 🔄
    - 14.4.1 [Load Balancing](14-haute-disponibilite/04.1-load-balancing.md)
    - 14.4.2 [Read/Write Split](14-haute-disponibilite/04.2-read-write-split.md)
    - 14.4.3 [Query Routing](14-haute-disponibilite/04.3-query-routing.md)
    - 14.4.4 [Database Firewall](14-haute-disponibilite/04.4-database-firewall.md)
- **14.5 [MaxScale 25.01 : Nouvelles fonctionnalités](14-haute-disponibilite/05-maxscale-25-nouveautes.md)** 🆕
    - 14.5.1 [Workload Capture](14-haute-disponibilite/05.1-workload-capture.md)
    - 14.5.2 [Workload Replay](14-haute-disponibilite/05.2-workload-replay.md)
    - 14.5.3 [Diff Router](14-haute-disponibilite/05.3-diff-router.md)
- 14.6 [Solutions de failover automatique](14-haute-disponibilite/06-failover-automatique.md)
- 14.7 [Virtual IP et keepalived](14-haute-disponibilite/07-virtual-ip-keepalived.md)
- 14.8 [Stratégies de récupération après incident](14-haute-disponibilite/08-strategies-recuperation.md)
- 14.9 [Alternatives : ProxySQL et HAProxy](14-haute-disponibilite/09-proxysql-haproxy.md)
- **14.10 [Transaction Replay et Connection Migration](14-haute-disponibilite/10-transaction-replay-connection-migration.md)** 🆕

---

## **[Partie 7 : Performance et Tuning (DBA Avancé)](/partie-07-performance-tuning.md)**

### 15. [Performance et Tuning](15-performance-tuning/README.md)
- 15.1 [Méthodologie d'optimisation](15-performance-tuning/01-methodologie-optimisation.md)
- 15.2 [Configuration mémoire](15-performance-tuning/02-configuration-memoire.md)
    - 15.2.1 [InnoDB Buffer Pool : Dimensionnement](15-performance-tuning/02.1-innodb-buffer-pool.md)
    - 15.2.2 [Buffer Pool instances](15-performance-tuning/02.2-buffer-pool-instances.md)
    - 15.2.3 [Key buffer (MyISAM)](15-performance-tuning/02.3-key-buffer.md)
- 15.3 [Query Cache : Pourquoi il est déprécié](15-performance-tuning/03-query-cache-deprecie.md)
- 15.4 [Configuration I/O et disques](15-performance-tuning/04-configuration-io-disques.md) 🔄
    - 15.4.1 [innodb_io_capacity](15-performance-tuning/04.1-innodb-io-capacity.md)
    - 15.4.2 [innodb_flush_method](15-performance-tuning/04.2-innodb-flush-method.md)
    - 15.4.3 [Optimisations SSD modernes](15-performance-tuning/04.3-optimisations-ssd.md)
- 15.5 [Optimisation du moteur InnoDB](15-performance-tuning/05-optimisation-innodb.md)
- **15.6 [innodb_alter_copy_bulk : Construction d'index efficace](15-performance-tuning/06-innodb-alter-copy-bulk.md)** 🆕
- 15.7 [Analyse des requêtes lentes](15-performance-tuning/07-analyse-requetes-lentes.md)
    - 15.7.1 [Slow query log](15-performance-tuning/07.1-slow-query-log.md)
    - 15.7.2 [pt-query-digest (Percona Toolkit)](15-performance-tuning/07.2-pt-query-digest.md)
- 15.8 [Performance Schema et sys schema](15-performance-tuning/08-performance-schema-sys.md)
- 15.9 [Partitionnement de tables](15-performance-tuning/09-partitionnement-tables.md)
    - 15.9.1 [RANGE partitioning](15-performance-tuning/09.1-range-partitioning.md)
    - 15.9.2 [LIST partitioning](15-performance-tuning/09.2-list-partitioning.md)
    - 15.9.3 [HASH partitioning](15-performance-tuning/09.3-hash-partitioning.md)
    - 15.9.4 [Partition pruning](15-performance-tuning/09.4-partition-pruning.md)
- **15.10 [Gestion avancée des partitions (conversion partition↔table)](15-performance-tuning/10-gestion-avancee-partitions.md)** 🆕
- 15.11 [Sharding et distribution horizontale](15-performance-tuning/11-sharding-distribution.md)
- 15.12 [Benchmarking](15-performance-tuning/12-benchmarking.md)
    - 15.12.1 [sysbench](15-performance-tuning/12.1-sysbench.md)
    - 15.12.2 [mysqlslap](15-performance-tuning/12.2-mysqlslap.md)
- 15.13 [Adaptive Hash Index et Buffer Pool optimizations](15-performance-tuning/13-adaptive-hash-index.md)
- **15.14 [Cost-based optimizer amélioré (prise en compte SSD)](15-performance-tuning/14-cost-based-optimizer-ssd.md)** 🆕

---

## **[Partie 8 : DevOps, Cloud et Automatisation](/partie-08-devops-cloud-automatisation.md)**

### 16. [DevOps et Automatisation](16-devops-automatisation/README.md)
- 16.1 [Infrastructure as Code pour MariaDB](16-devops-automatisation/01-infrastructure-as-code.md)
- 16.2 [Déploiement avec Ansible/Terraform](16-devops-automatisation/02-deploiement-ansible-terraform.md)
    - 16.2.1 [Ansible : Playbooks MariaDB](16-devops-automatisation/02.1-ansible-playbooks.md)
    - 16.2.2 [Terraform : Providers cloud](16-devops-automatisation/02.2-terraform-providers.md)
- 16.3 [Conteneurisation avec Docker](16-devops-automatisation/03-conteneurisation-docker.md)
    - 16.3.1 [Images officielles MariaDB](16-devops-automatisation/03.1-images-officielles.md)
    - 16.3.2 [Docker Compose pour développement](16-devops-automatisation/03.2-docker-compose.md)
    - 16.3.3 [Volumes et persistance](16-devops-automatisation/03.3-volumes-persistance.md)
- 16.4 [Orchestration avec Kubernetes](16-devops-automatisation/04-orchestration-kubernetes.md)
    - 16.4.1 [StatefulSets pour MariaDB](16-devops-automatisation/04.1-statefulsets.md)
    - 16.4.2 [PersistentVolumes et StorageClasses](16-devops-automatisation/04.2-persistentvolumes.md)
- 16.5 [mariadb-operator pour Kubernetes](16-devops-automatisation/05-mariadb-operator.md) 🔄
    - 16.5.1 [Installation et CRDs](16-devops-automatisation/05.1-installation-crds.md)
    - 16.5.2 [Déploiement Galera avec operator](16-devops-automatisation/05.2-deploiement-galera.md)
    - 16.5.3 [Réplication avec operator](16-devops-automatisation/05.3-replication-operator.md)
    - 16.5.4 [Backups automatisés](16-devops-automatisation/05.4-backups-automatises.md)
- **16.6 [MariaDB Enterprise Operator](16-devops-automatisation/06-mariadb-enterprise-operator.md)** 🆕
- 16.7 [CI/CD pour bases de données](16-devops-automatisation/07-cicd-bases-donnees.md)
- 16.8 [Gestion des migrations](16-devops-automatisation/08-gestion-migrations.md)
    - 16.8.1 [Flyway](16-devops-automatisation/08.1-flyway.md)
    - 16.8.2 [Liquibase](16-devops-automatisation/08.2-liquibase.md)
    - 16.8.3 [gh-ost et pt-online-schema-change](16-devops-automatisation/08.3-gh-ost-pt-osc.md)
- 16.9 [Monitoring avec Prometheus/Grafana](16-devops-automatisation/09-monitoring-prometheus-grafana.md)
    - 16.9.1 [mysqld_exporter](16-devops-automatisation/09.1-mysqld-exporter.md)
    - 16.9.2 [Dashboards Grafana](16-devops-automatisation/09.2-dashboards-grafana.md)
- 16.10 [Observabilité : Logs, Metrics, Traces](16-devops-automatisation/10-observabilite.md)
- 16.11 [Alerting et incident response](16-devops-automatisation/11-alerting-incident-response.md)
- 16.12 [GitOps pour les bases de données](16-devops-automatisation/12-gitops-bases-donnees.md)

---

## **[Partie 9 : Intégration, Développement et Fonctionnalités Avancées](/partie-09-integration-fonctionnalites-avancees.md)**

### 17. [Intégration et Développement](17-integration-developpement/README.md)
- 17.1 [Connexion depuis différents langages](17-integration-developpement/01-connexion-langages.md)
    - 17.1.1 [PHP : mysqli et PDO](17-integration-developpement/01.1-php-mysqli-pdo.md)
    - 17.1.2 [Python : mysql-connector, PyMySQL, SQLAlchemy](17-integration-developpement/01.2-python-connectors.md)
    - 17.1.3 [Java : JDBC, MariaDB Connector/J](17-integration-developpement/01.3-java-jdbc.md)
    - 17.1.4 [Node.js : mysql2, mariadb](17-integration-developpement/01.4-nodejs-connectors.md)
    - 17.1.5 [Go : go-sql-driver/mysql](17-integration-developpement/01.5-go-connector.md)
    - 17.1.6 [.NET : MySqlConnector, MariaDB.Data, ADO.NET](17-integration-developpement/01.6-dotnet-connectors.md)
- 17.2 [Connection pooling](17-integration-developpement/02-connection-pooling.md)
    - 17.2.1 [Pool côté application](17-integration-developpement/02.1-pool-application.md)
    - 17.2.2 [ProxySQL comme pooler](17-integration-developpement/02.2-proxysql-pooler.md)
- 17.3 [ORM et frameworks](17-integration-developpement/03-orm-frameworks.md)
    - 17.3.1 [Hibernate (Java)](17-integration-developpement/03.1-hibernate.md)
    - 17.3.2 [SQLAlchemy (Python)](17-integration-developpement/03.2-sqlalchemy.md)
    - 17.3.3 [Sequelize (Node.js)](17-integration-developpement/03.3-sequelize.md)
    - 17.3.4 [Prisma](17-integration-developpement/03.4-prisma.md)
    - 17.3.5 [Entity Framework Core (.NET)](17-integration-developpement/03.5-entity-framework-core.md)
- 17.4 [Bonnes pratiques de développement](17-integration-developpement/04-bonnes-pratiques.md)
- 17.5 [Gestion des migrations de schéma](17-integration-developpement/05-gestion-migrations-schema.md)
- 17.6 [Tests de bases de données](17-integration-developpement/06-tests-bases-donnees.md)
- 17.7 [Environnements de développement](17-integration-developpement/07-environnements-developpement.md)
- 17.8 [Prévention des injections SQL](17-integration-developpement/08-prevention-injections-sql.md)
- 17.9 [Prepared statements et parameterized queries](17-integration-developpement/09-prepared-statements.md)

### 18. [Fonctionnalités Avancées](18-fonctionnalites-avancees/README.md)
- 18.1 [Sequences (CREATE SEQUENCE)](18-fonctionnalites-avancees/01-sequences.md)
- 18.2 [Tables temporelles (System-Versioned Tables)](18-fonctionnalites-avancees/02-system-versioned-tables.md) 🔄
    - 18.2.1 [Création et configuration](18-fonctionnalites-avancees/02.1-creation-configuration.md)
    - 18.2.2 [Requêtes temporelles (AS OF, BETWEEN, FROM...TO)](18-fonctionnalites-avancees/02.2-requetes-temporelles.md)
    - 18.2.3 [Partitionnement des données historiques](18-fonctionnalites-avancees/02.3-partitionnement-historique.md)
- **18.3 [Application Time Period Tables](18-fonctionnalites-avancees/03-application-time-period-tables.md)** 🆕
- 18.4 [Colonnes virtuelles et générées](18-fonctionnalites-avancees/04-virtual-generated-columns.md)
    - 18.4.1 [VIRTUAL vs STORED](18-fonctionnalites-avancees/04.1-virtual-vs-stored.md)
    - 18.4.2 [Indexation de colonnes générées](18-fonctionnalites-avancees/04.2-indexation-colonnes-generees.md)
- 18.5 [Invisible columns](18-fonctionnalites-avancees/05-invisible-columns.md)
- 18.6 [Compression de tables](18-fonctionnalites-avancees/06-compression-tables.md)
- 18.7 [Encryption at rest](18-fonctionnalites-avancees/07-encryption-at-rest.md)
    - 18.7.1 [Data at Rest Encryption](18-fonctionnalites-avancees/07.1-data-at-rest-encryption.md)
    - 18.7.2 [Key management](18-fonctionnalites-avancees/07.2-key-management.md)
- 18.8 [Thread Pool avancé](18-fonctionnalites-avancees/08-thread-pool-avance.md)
- 18.9 [Dynamic columns](18-fonctionnalites-avancees/09-dynamic-columns.md)
- **18.10 [MariaDB Vector : Recherche vectorielle pour l'IA/RAG](18-fonctionnalites-avancees/10-mariadb-vector.md)** 🆕
    - 18.10.1 [Type de données VECTOR](18-fonctionnalites-avancees/10.1-type-donnees-vector.md)
    - 18.10.2 [Index HNSW (Hierarchical Navigable Small Worlds)](18-fonctionnalites-avancees/10.2-index-hnsw.md)
    - 18.10.3 [Fonctions de distance (VEC_DISTANCE_EUCLIDEAN, VEC_DISTANCE_COSINE)](18-fonctionnalites-avancees/10.3-fonctions-distance.md)
    - 18.10.4 [Fonctions de conversion (VEC_FromText, VEC_ToText)](18-fonctionnalites-avancees/10.4-fonctions-conversion.md)
    - 18.10.5 [Optimisations SIMD (AVX2, AVX512, ARM, IBM Power10)](18-fonctionnalites-avancees/10.5-optimisations-simd.md)
    - 18.10.6 [Intégration avec LLMs (OpenAI, Claude, LLaMA)](18-fonctionnalites-avancees/10.6-integration-llms.md)
- **18.11 [Online Schema Change (ALTER TABLE non-bloquant)](18-fonctionnalites-avancees/11-online-schema-change.md)** 🆕

---

## **[Partie 10 : Migration, Compatibilité et Architectures](/partie-10-migration-architectures.md)**

### 19. [Migration et Compatibilité](19-migration-compatibilite/README.md)
- 19.1 [Migration depuis MySQL](19-migration-compatibilite/01-migration-depuis-mysql.md)
    - 19.1.1 [Compatibilité MySQL/MariaDB](19-migration-compatibilite/01.1-compatibilite-mysql-mariadb.md)
    - 19.1.2 [Points d'attention et différences](19-migration-compatibilite/01.2-points-attention.md)
    - 19.1.3 [Outils de migration](19-migration-compatibilite/01.3-outils-migration.md)
- 19.2 [Migration depuis d'autres SGBD](19-migration-compatibilite/02-migration-autres-sgbd.md)
    - 19.2.1 [Depuis Oracle](19-migration-compatibilite/02.1-depuis-oracle.md)
    - 19.2.2 [Depuis SQL Server](19-migration-compatibilite/02.2-depuis-sql-server.md)
    - 19.2.3 [Depuis PostgreSQL](19-migration-compatibilite/02.3-depuis-postgresql.md)
- 19.3 [Gestion des versions : Stratégie LTS vs Rolling](19-migration-compatibilite/03-gestion-versions-lts-rolling.md) 🔄
- 19.4 [Stratégies de mise à jour et upgrade paths](19-migration-compatibilite/04-strategies-mise-a-jour.md)
    - 19.4.1 [mariadb-upgrade](19-migration-compatibilite/04.1-mariadb-upgrade.md)
    - 19.4.2 [Upgrade in-place vs logical](19-migration-compatibilite/04.2-upgrade-inplace-logical.md)
- 19.5 [Compatibilité des applications](19-migration-compatibilite/05-compatibilite-applications.md)
- 19.6 [Tests de compatibilité](19-migration-compatibilite/06-tests-compatibilite.md)
- 19.7 [Rollback et contingence](19-migration-compatibilite/07-rollback-contingence.md)
- 19.8 [Zero-downtime migrations](19-migration-compatibilite/08-zero-downtime-migrations.md)
- **19.9 [Migration System-Versioned Tables (changement format timestamp 11.8)](19-migration-compatibilite/09-migration-system-versioned-tables.md)** 🆕

### 20. [Cas d'Usage et Architectures](20-cas-usage-architectures/README.md)
- 20.1 [OLTP vs OLAP](20-cas-usage-architectures/01-oltp-vs-olap.md)
- 20.2 [Architecture microservices](20-cas-usage-architectures/02-architecture-microservices.md)
    - 20.2.1 [Database per service](20-cas-usage-architectures/02.1-database-per-service.md)
    - 20.2.2 [Shared database pattern](20-cas-usage-architectures/02.2-shared-database.md)
- 20.3 [Data warehousing avec ColumnStore](20-cas-usage-architectures/03-data-warehousing-columnstore.md)
- 20.4 [Architecture multi-tenant](20-cas-usage-architectures/04-architecture-multi-tenant.md)
    - 20.4.1 [Database per tenant](20-cas-usage-architectures/04.1-database-per-tenant.md)
    - 20.4.2 [Schema per tenant](20-cas-usage-architectures/04.2-schema-per-tenant.md)
    - 20.4.3 [Shared schema avec discriminateur](20-cas-usage-architectures/04.3-shared-schema.md)
- 20.5 [Géo-distribution](20-cas-usage-architectures/05-geo-distribution.md)
- 20.6 [Hybrid cloud et multi-cloud](20-cas-usage-architectures/06-hybrid-multi-cloud.md)
- 20.7 [Scaling vertical vs horizontal](20-cas-usage-architectures/07-scaling-vertical-horizontal.md)
- 20.8 [MariaDB dans les architectures Event-Driven](20-cas-usage-architectures/08-architectures-event-driven.md)
    - 20.8.1 [CDC (Change Data Capture)](20-cas-usage-architectures/08.1-cdc.md)
    - 20.8.2 [Integration avec Kafka](20-cas-usage-architectures/08.2-integration-kafka.md)
    - 20.8.3 [Debezium connector](20-cas-usage-architectures/08.3-debezium.md)
- **20.9 [Use cases IA : RAG et recherche vectorielle](20-cas-usage-architectures/09-use-cases-ia-rag.md)** 🆕
    - 20.9.1 [Semantic Search](20-cas-usage-architectures/09.1-semantic-search.md)
    - 20.9.2 [Recommendation Engines](20-cas-usage-architectures/09.2-recommendation-engines.md)
    - 20.9.3 [Anomaly Detection](20-cas-usage-architectures/09.3-anomaly-detection.md)
    - 20.9.4 [Hybrid Search (vecteurs + SQL relationnel)](20-cas-usage-architectures/09.4-hybrid-search.md)
- **20.10 [MariaDB MCP Server pour intégration IA](20-cas-usage-architectures/10-mcp-server-integration-ia.md)** 🆕
- **20.11 [Intégrations frameworks IA (LangChain, LlamaIndex, etc.)](20-cas-usage-architectures/11-integrations-frameworks-ia.md)** 🆕
- 20.12 [Études de cas réels](20-cas-usage-architectures/12-etudes-cas-reels.md)

---

## **Annexes**

### A. [Glossaire des Termes Techniques](annexes/glossaire/README.md)
- A.1 [Termes MariaDB essentiels (ACID, MVCC, InnoDB, Galera, etc.)](annexes/glossaire/01-termes-mariadb-essentiels.md)
- A.2 [Acronymes courants (FK, PK, CTE, GTID, SST, IST, etc.)](annexes/glossaire/02-acronymes-courants.md)

### B. [Commandes mariadb CLI Essentielles](annexes/commandes-cli/README.md)
- B.1 [Connexion et navigation (\s, \u, SHOW DATABASES, SHOW TABLES)](annexes/commandes-cli/01-connexion-navigation.md)
- B.2 [Informations système (STATUS, SHOW PROCESSLIST, SHOW ENGINE)](annexes/commandes-cli/02-informations-systeme.md)
- B.3 [Export/Import (SOURCE, TEE, mariadb-dump)](annexes/commandes-cli/03-export-import.md)

### C. [Requêtes SQL de Référence](annexes/requetes-sql-reference/README.md)
- C.1 [Requêtes d'administration (locks, processlist, table sizes)](annexes/requetes-sql-reference/01-requetes-administration.md)
- C.2 [Requêtes de monitoring (buffer pool, slow queries, replication)](annexes/requetes-sql-reference/02-requetes-monitoring.md)
- C.3 [Requêtes d'analyse (statistiques, index usage)](annexes/requetes-sql-reference/03-requetes-analyse.md)

### D. [Configuration de Référence par Cas d'Usage](annexes/configuration-reference/README.md)
- D.1 [OLTP (High concurrency, low latency)](annexes/configuration-reference/01-configuration-oltp.md)
- D.2 [OLAP (Data warehouse, analytics)](annexes/configuration-reference/02-configuration-olap.md)
- D.3 [Mixed workload](annexes/configuration-reference/03-configuration-mixed-workload.md)
- D.4 [Développement local](annexes/configuration-reference/04-configuration-developpement-local.md)

### E. [Checklist de Performance](annexes/checklist-performance/README.md)
- E.1 [Audit de configuration](annexes/checklist-performance/01-audit-configuration.md)
- E.2 [Audit d'indexation](annexes/checklist-performance/02-audit-indexation.md)
- E.3 [Audit de requêtes](annexes/checklist-performance/03-audit-requetes.md)
- E.4 [Audit de schéma](annexes/checklist-performance/04-audit-schema.md)

### F. [Nouveautés MariaDB 11.8 LTS en un Coup d'Œil](annexes/nouveautes-11-8/README.md) 🆕
- F.1 [Tableau récapitulatif des features majeures](annexes/nouveautes-11-8/01-tableau-recapitulatif.md)
- F.2 [MariaDB Vector : La fonctionnalité phare](annexes/nouveautes-11-8/02-mariadb-vector.md)
- F.3 [Impact sur migration et compatibilité](annexes/nouveautes-11-8/03-impact-migration-compatibilite.md)
- F.4 [Recommandations d'adoption](annexes/nouveautes-11-8/04-recommandations-adoption.md)

### G. [Versions de Référence](annexes/versions-reference/README.md)

| Version | Type | Date GA | Support jusqu'à |
|---------|------|---------|-----------------|
| **11.8** | LTS | Juin 2025 | 2028 |
| **11.4** | LTS | Mai 2024 | 2027 |
| 10.11 | LTS | Fév 2023 | 2028 |
| 10.6 | LTS | Juil 2021 | 2026 |

### H. [Ressources et Documentation](annexes/ressources-documentation/README.md)
- H.1 [Documentation officielle](annexes/ressources-documentation/01-documentation-officielle.md)
- H.2 [Communautés et forums](annexes/ressources-documentation/02-communautes-forums.md)
- H.3 [Blogs techniques recommandés](annexes/ressources-documentation/03-blogs-techniques.md)
- H.4 [Conférences et événements](annexes/ressources-documentation/04-conferences-evenements.md)

### I. [Changelog de la Formation](annexes/changelog/README.md)

**Novembre 2025 :**
- Ajout MariaDB 11.8 LTS comme version de référence principale
- Intégration complète de MariaDB Vector (18.10, 20.9-20.11)
- Nouveau parcours IA/ML
- Mise à jour politique de versions (3 ans LTS, roadmap 12.x)
- Ajout fonctionnalités JSON avancées (4.8, 4.9)
- Nouveautés sécurité (PARSEC, TLS par défaut, privilèges granulaires)
- Ajout charset utf8mb4 par défaut et extension TIMESTAMP 2106
- Mise à jour MaxScale 25.01 avec Workload Capture/Replay
- Ajout MCP Server et intégrations frameworks IA

---

## **Points Forts de Cette Formation**

- ✅ **Progressive** : Parcours structuré du débutant à l'expert
- ✅ **Complète** : 20 chapitres couvrant tous les aspects
- ✅ **À jour** : Intégration complète des nouveautés MariaDB 11.8 LTS 🆕
- ✅ **Professionnelle** : Production, monitoring, HA, architectures modernes
- ✅ **Moderne** : Cloud, Kubernetes, IA, architectures distribuées
- ✅ **Pratique** : Annexes et références techniques détaillées

**Durée estimée** : 35-50 heures de formation théorique
**Public** : Développeurs, DevOps, DBAs débutants à avancés
**Prérequis** : Bases en SQL et systèmes Unix/Linux

---

**Version** : 1.0 - Novembre 2025
**MariaDB** : Version 11.8 LTS (Juin 2025)
**Licence** : Creative Commons BY-NC-SA 4.0

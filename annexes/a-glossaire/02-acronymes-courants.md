🔝 Retour au [Sommaire](/SOMMAIRE.md)

# A.2 — Acronymes courants

> 🔠 Le « décodeur » de la formation : sigle → forme développée → courte mise en contexte.

Cette section développe les sigles qui parsèment la documentation et les chapitres. Le format est volontairement compact. Les acronymes correspondant à un concept détaillé dans la section [A.1 — Termes MariaDB essentiels](01-termes-mariadb-essentiels.md) y figurent en version brève, accompagnés d'un renvoi vers la définition complète. Les entrées sont classées par ordre alphabétique ; le marqueur 🆕 signale un sigle lié à une nouveauté de la série 12.x (depuis la 11.8).

---

## A

**ACID** — *Atomicity, Consistency, Isolation, Durability.* Les quatre propriétés d'une transaction fiable. *Voir A.1 et Ch. 6.*

**ADO.NET** — *ActiveX Data Objects for .NET.* Couche d'accès aux données de l'écosystème .NET. *Voir Ch. 17.*

**API** — *Application Programming Interface* (interface de programmation applicative).

**AVX** — *Advanced Vector Extensions.* Jeux d'instructions SIMD d'Intel (AVX2, AVX512) accélérant les calculs de distance vectorielle. *Voir Ch. 18.*

## B

**BKA** — *Batched Key Access.* Stratégie de jointure par accès groupé aux clés, pilotable par hint d'optimiseur (🆕 12.x). *Voir Ch. 15.*

**BNL** — *Block Nested Loop.* Algorithme de jointure par blocs, pilotable par hint d'optimiseur (🆕 12.x). *Voir Ch. 15.*

## C

**CA** — *Certificate Authority* (autorité de certification). Émet et valide les certificats TLS. *Voir Ch. 10.*

**CDC** — *Change Data Capture* (capture des changements de données). Diffusion des modifications vers des systèmes externes (Kafka, Debezium). *Voir Ch. 20.*

**CI/CD** — *Continuous Integration / Continuous Delivery* (ou *Deployment*). Intégration et livraison/déploiement continus, appliqués aussi aux schémas de base. *Voir Ch. 16.*

**CLI** — *Command Line Interface* (interface en ligne de commande). Mode d'utilisation du client `mariadb` et de ses outils compagnons (`mariadb-dump`, `mariadb-admin`…). *Voir Annexe B.*

**CRD** — *Custom Resource Definition.* Ressource Kubernetes personnalisée, utilisée par le `mariadb-operator`. *Voir Ch. 16.*

**CTE** — *Common Table Expression* (expression de table commune). Sous-requête nommée introduite par `WITH`. *Voir Ch. 4.*

## D

**DBA** — *Database Administrator* (administrateur de base de données).

**DBMS** — *Database Management System.* Équivalent anglais de SGBD. *Voir SGBD.*

**DCL** — *Data Control Language.* Sous-ensemble SQL de gestion des droits (`GRANT`, `REVOKE`). *Voir Ch. 10.*

**DDL** — *Data Definition Language.* Sous-ensemble SQL de définition des objets (`CREATE`, `ALTER`, `DROP`). *Voir Ch. 2.*

**DML** — *Data Manipulation Language.* Sous-ensemble SQL de manipulation des données (`INSERT`, `UPDATE`, `DELETE`, `SELECT`). *Voir Ch. 2.*

## E

**EITS** — *Engine-Independent Table Statistics* (statistiques moteur-indépendantes). Statistiques de distribution des données, collectées par `ANALYZE TABLE … PERSISTENT FOR ALL` et stockées dans `mysql.table_stats` / `column_stats` / `index_stats`, indépendamment du moteur de stockage. *Voir Ch. 11.6 et Annexe C.3.*

## F

**FK** — *Foreign Key* (clé étrangère). Contrainte référentielle reliant une colonne à la clé d'une autre table, pour garantir l'intégrité des références. En 12.3, le nom d'une contrainte FK ne doit être unique qu'au sein de sa table (et non plus de toute la base). *Voir Ch. 2 et Ch. 18.*

## G

**GA** — *General Availability* (disponibilité générale). Statut d'une version stable destinée à la production ; la 12.3 est GA depuis fin mai 2026. *Voir Annexe G.*

**GIS** — *Geographic Information System* (système d'information géographique). Données et fonctions spatiales (géométries, fonctions `ST_*`, index `SPATIAL`) ; la série 12.x s'aligne sur de nouvelles fonctions GIS de MySQL 8. *Voir Ch. 19.*

**GSSAPI** — *Generic Security Services Application Program Interface.* API d'authentification utilisée notamment avec Kerberos. *Voir Ch. 10.*

**GTID** — *Global Transaction Identifier* (identifiant de transaction global). Identifie de façon unique chaque transaction et simplifie grandement le failover. *Voir Ch. 13.*

## H

**HA** — *High Availability* (haute disponibilité). Capacité à rester opérationnel malgré les pannes. *Voir Ch. 14.*

**HNSW** — *Hierarchical Navigable Small Worlds.* Algorithme d'index pour la recherche vectorielle par similarité (approximate nearest neighbor). *Voir Ch. 5 et Ch. 18.*

## I

**IaC** — *Infrastructure as Code* (infrastructure en tant que code). Gestion déclarative de l'infrastructure (Ansible, Terraform). *Voir Ch. 16.*

**ICP** — *Index Condition Pushdown.* Optimisation poussant l'évaluation des conditions au niveau de l'index ; (dés)activable par hint en 12.x (🆕). *Voir Ch. 5.*

**I/O** — *Input/Output* (entrées/sorties). Désigne les accès disque, souvent facteur limitant de performance. *Voir Ch. 15.*

**IST** — *Incremental State Transfer.* Resynchronisation incrémentale d'un nœud Galera de retour dans le cluster. *Voir Ch. 14.*

## J

**JDBC** — *Java Database Connectivity.* API standard d'accès aux bases depuis Java (Connector/J). *Voir Ch. 17.*

**JSON** — *JavaScript Object Notation.* Format de données semi-structurées, géré nativement par MariaDB. *Voir A.1 et Ch. 4.*

## L

**LDAP** — *Lightweight Directory Access Protocol.* Annuaire centralisé exploitable pour l'authentification. *Voir Ch. 10.*

**LLM** — *Large Language Model* (grand modèle de langage). Modèle d'IA générative interrogé via embeddings et RAG. *Voir Ch. 18.*

**LTS** — *Long Term Support* (support à long terme). Versions à support long (ex. 10.11, 11.4, 11.8, 12.3) ; maintenance communautaire de **3 ans depuis la 11.8** (5 ans pour les LTS antérieures, 10.11 et 11.4), par opposition aux versions rolling. *Voir Ch. 1 et Annexe G.*

## M

**MCP** — *Model Context Protocol.* Protocole standard (introduit par Anthropic) par lequel un assistant IA dialogue avec des outils et des sources de données ; MariaDB fournit un *MCP Server* (lecture seule par défaut) pour interroger une base. *Voir Ch. 20.*

**ML** — *Machine Learning* (apprentissage automatique). Souvent associé à « IA » (AI/ML).

**MRR** — *Multi-Range Read.* Optimisation de lecture par plages réduisant les accès aléatoires ; pilotable par hint d'optimiseur (🆕 12.x). *Voir Ch. 15.*

**MVCC** — *Multi-Version Concurrency Control* (contrôle de concurrence multiversion). *Voir A.1 et Ch. 6.*

## O

**OLAP** — *Online Analytical Processing.* Charges analytiques sur de gros volumes (ColumnStore). *Voir Ch. 20.*

**OLTP** — *Online Transaction Processing.* Charges transactionnelles à forte concurrence et faible latence (InnoDB). *Voir Ch. 20.*

**ORM** — *Object-Relational Mapping* (mapping objet-relationnel). Pont entre objets applicatifs et tables (Hibernate, SQLAlchemy, Sequelize, Prisma, EF Core). *Voir Ch. 17.*

## P

**PAM** — *Pluggable Authentication Modules.* Cadre d'authentification système réutilisable par MariaDB. *Voir Ch. 10.*

**PARSEC** — Plugin d'authentification moderne de MariaDB reposant sur la cryptographie à courbes elliptiques (signature d'un défi). *Voir Ch. 10.*

**PDO** — *PHP Data Objects.* Couche d'accès aux bases en PHP. *Voir Ch. 17.*

**PITR** — *Point-In-Time Recovery* (restauration à un instant précis). Rejoue les binlogs jusqu'à un point donné. *Voir Ch. 12.*

**PK** — *Primary Key* (clé primaire). Identifiant unique d'une ligne. *Voir Ch. 2.*

**PRA** — Plan de Reprise d'Activité (≈ *Disaster Recovery Plan*). Procédures de remise en service après incident majeur. *Voir Ch. 12.*

## Q

**QB** — *Query Block* (bloc de requête). Unité ciblée par les Optimizer Hints, nommable via `QB_NAME` (🆕 12.x). *Voir Ch. 15.*

## R

**RAG** — *Retrieval-Augmented Generation* (génération augmentée par récupération). Cas d'usage IA combinant recherche vectorielle et LLM. *Voir Ch. 20.*

**RAM** — *Random Access Memory* (mémoire vive). Support des tables du moteur Memory. *Voir Ch. 7.*

**RC** — *Release Candidate* (version candidate). Jalon précédant la GA ; par exemple la 12.3.1. *Voir Annexe G.*

**RPO** — *Recovery Point Objective.* Perte de données maximale tolérée, exprimée en temps. *Voir Ch. 12.*

**RTO** — *Recovery Time Objective.* Durée d'indisponibilité maximale tolérée. *Voir Ch. 12.*

## S

**S3** — *Simple Storage Service.* Stockage objet d'AWS (et compatibles comme MinIO), employé par le moteur S3 et les sauvegardes cloud. *Voir Ch. 7 et Ch. 12.*

**SGBD** — Système de Gestion de Base de Données (équivalent français de DBMS). *Voir Ch. 1.*

**SIMD** — *Single Instruction, Multiple Data.* Parallélisme de données exploité pour accélérer les calculs vectoriels. *Voir Ch. 18.*

**SQL** — *Structured Query Language* (langage de requête structuré). *Voir Ch. 2.*

**SSD** — *Solid-State Drive* (disque à mémoire flash). Désormais pris en compte par l'optimiseur basé sur les coûts. *Voir Ch. 15.*

**SSL** — *Secure Sockets Layer.* Ancien protocole de chiffrement, souvent employé comme synonyme de TLS. *Voir Ch. 10.*

**SST** — *State Snapshot Transfer.* Resynchronisation complète d'un nœud Galera. *Voir Ch. 14.*

## T

**TLS** — *Transport Layer Security.* Protocole de chiffrement des connexions, successeur de SSL (TLS zéro-configuration depuis la 11.8). *Voir Ch. 10.*

## U

**UCA** — *Unicode Collation Algorithm.* Algorithme de tri Unicode (collations UCA 14.0.0). *Voir Ch. 11.*

**UUID** — *Universally Unique Identifier* (identifiant unique universel). Type de données dédié. *Voir Ch. 2.*

## X

**XA** — *eXtended Architecture.* Standard des transactions distribuées (validation en deux phases). *Voir Ch. 6.*

**XML** — *eXtensible Markup Language.* 🆕 Type `XMLTYPE` basique ajouté en 12.3 pour la compatibilité Oracle. *Voir Ch. 2.*

---

⬅️ [A.1 — Termes MariaDB essentiels](01-termes-mariadb-essentiels.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [Annexe B — Commandes mariadb CLI](../b-commandes-cli/README.md)

⏭️ [Commandes mariadb CLI Essentielles](/annexes/b-commandes-cli/README.md)

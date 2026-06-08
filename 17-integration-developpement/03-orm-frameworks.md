🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3 ORM et frameworks

Les sections précédentes ont montré comment dialoguer avec MariaDB « au plus près du SQL » : connecteurs (§17.1) et *pooling* (§17.2). Beaucoup d'applications, pourtant, n'écrivent pas de SQL à la main : elles passent par un **ORM** (*Object-Relational Mapping*). C'est ici que se retrouvent les usages d'ORM renvoyés depuis chaque langage — SQLAlchemy, Sequelize, Prisma, Entity Framework Core, Hibernate.

Point capital à garder en tête dès le départ : **un ORM ne remplace ni le connecteur ni le pool**. Il se construit **au-dessus** d'eux. Hibernate s'appuie sur JDBC, SQLAlchemy sur un pilote DB-API, Sequelize sur mysql2 ou le connecteur `mariadb`, Entity Framework Core sur MySqlConnector. Tout ce qui a été dit sur les connexions, les transactions et le *pooling* reste donc valable.

Cette section pose les concepts communs ; chaque sous-section décline ensuite un ORM particulier.

---

## Le problème : l'« impedance mismatch »

Le monde objet et le monde relationnel ne se ressemblent pas. D'un côté, des **classes** avec héritage, des **références** entre objets, des **collections**, une identité par référence en mémoire. De l'autre, des **tables**, des **lignes**, des **clés étrangères** (§2.5), aucune notion d'héritage, une identité par clé primaire. Faire correspondre ces deux univers à la main est répétitif et source d'erreurs : c'est le « décalage d'impédance » (*object-relational impedance mismatch*).

Un ORM a précisément pour rôle de **combler ce fossé** : il fait correspondre les classes aux tables, les instances aux lignes, les attributs aux colonnes, et traduit les manipulations d'objets en requêtes SQL.

---

## Les concepts communs

Au-delà des syntaxes, tous les ORM partagent un même vocabulaire :

- **Entité / modèle** — une classe associée à une table ; ses instances représentent des lignes.
- **Mapping** — la déclaration des correspondances classe↔table et attribut↔colonne (par annotations, décorateurs, fichier de schéma ou configuration *fluent*).
- **Identité et clé primaire** — l'ORM tient souvent une *identity map* : un seul objet par ligne au sein d'une même unité de travail.
- **Associations** — les relations *un-à-plusieurs*, *plusieurs-à-un*, *plusieurs-à-plusieurs*, traduites en clés étrangères et tables de jointure.
- **Stratégies de chargement** — chargement **paresseux** (*lazy* : la relation n'est lue qu'au premier accès) ou **anticipé** (*eager* : lue d'emblée). C'est là que se niche le problème N+1 (voir plus bas).
- **Unité de travail / session / contexte** — un objet (`Session` Hibernate, `Session` SQLAlchemy, `DbContext` EF Core) qui suit les modifications et les écrit dans une transaction.
- **Approches de requête** — un langage ou une API propre à l'ORM (HQL/JPQL, l'API de requête SQLAlchemy, les *finders* Sequelize, le client Prisma, LINQ pour EF Core), **plus** une porte de sortie vers le SQL brut.

---

## Deux familles : Active Record et Data Mapper

Les ORM se rattachent classiquement à deux patrons d'architecture :

- **Active Record** — l'objet modèle sait se sauvegarder et se charger lui-même (`objet.save()`). Dans cette section, **Sequelize** (§17.3.3) en relève.
- **Data Mapper** — une couche distincte gère la persistance, les objets du domaine en restant ignorants. **Hibernate** (§17.3.1), **SQLAlchemy** (§17.3.2) et **Entity Framework Core** (§17.3.5) s'en rapprochent, autour d'une *unité de travail*.

**Prisma** (§17.3.4) se range mal dans cette dichotomie : ce n'est pas un ORM à modèles instanciables, mais un **client typé généré à partir d'un schéma déclaratif** — on appelle des méthodes sur un client (`prisma.produit.findMany()`) plutôt que sur des objets entités. On le présente parfois comme un « ORM nouvelle génération ».

---

## Bénéfices et coûts

Un ORM bien employé apporte beaucoup, mais n'est pas gratuit. Il faut peser les deux plateaux.

**Bénéfices :** moins de code répétitif (plus de mapping manuel ligne↔objet) ; productivité élevée sur les opérations CRUD ; sécurité (les requêtes sont **paramétrées par défaut**, ce qui protège des injections, §17.8) ; support de l'outillage et, dans les langages typés, vérifications à la compilation ; une certaine portabilité entre dialectes SQL ; des commodités comme le suivi des modifications et le chargement paresseux.

**Coûts et risques :** c'est une **abstraction « qui fuit »** — il faut toujours comprendre le SQL généré ; les **requêtes analytiques complexes** sont souvent plus claires et plus rapides en SQL brut ; le SQL produit peut être **sous-optimal** ; et l'ORM a sa propre **courbe d'apprentissage**. Surtout, il expose au piège emblématique ci-dessous.

---

## Le piège emblématique : le problème N+1

C'est, de loin, le problème de performance le plus fréquent avec un ORM. Il survient lorsqu'on charge une liste de N entités, puis qu'on accède à une relation **chargée paresseusement** sur chacune, dans une boucle : l'ORM émet **1 requête** pour la liste, puis **N requêtes** (une par entité) pour les relations — soit **N+1 requêtes** là où une ou deux suffiraient.

Par exemple, charger 100 commandes puis lire `commande.client` pour chacune déclenche 1 + 100 = **101 requêtes**. La parade consiste à demander un **chargement anticipé** (jointure : `JOIN FETCH`, `include`, `joinedload`, `Include`…) pour rapatrier les données liées en une seule fois. Ce piège, déjà évoqué au §17.1 et approfondi côté bonnes pratiques (§17.4), justifie à lui seul de toujours surveiller le SQL réellement exécuté.

---

## ORM, SQL brut ou query builder ?

Il ne faut pas opposer ces approches : elles forment un continuum, et les combiner est la posture la plus saine.

- **ORM** — idéal pour le CRUD, la logique métier, l'essentiel de l'accès aux données d'une application.
- **SQL brut** — préférable pour les requêtes analytiques, les traitements en masse, les chemins critiques en performance, le *reporting*.
- **Query builder** — un intermédiaire qui construit du SQL de façon programmatique sans aller jusqu'au mapping objet (SQLAlchemy Core en est un exemple, §17.1.2).

La quasi-totalité des ORM offrent une **porte de sortie vers le SQL brut** : on utilise l'ORM pour le quotidien et l'on descend au SQL là où c'est justifié.

---

## Lien avec les connecteurs, les pools et les migrations

Trois rappels pour situer l'ORM dans l'ensemble du chapitre :

- **Connecteur (§17.1)** — l'ORM s'appuie sur un pilote ; il faut lui indiquer le **dialecte MariaDB** (ou MySQL) et le bon pilote. Les considérations d'authentification (`caching_sha2_password`, `ed25519`…) y restent pertinentes.
- **Pool (§17.2)** — l'ORM gère ou délègue le *pooling* ; tout ce qui précède sur le dimensionnement et la durée de vie des connexions s'applique encore.
- **Migrations (§17.5)** — l'évolution du schéma est souvent prise en charge par l'ORM ou un outil dédié (Flyway, Liquibase… chap. 16.8).

Enfin, certaines **fonctionnalités propres à MariaDB** (séquences, tables temporelles, JSON, VECTOR — chap. 18) ne sont pas toujours exposées par l'ORM et peuvent nécessiter du SQL brut ou une extension.

---

## Les frameworks couverts

| Sous-section | ORM | Langage | Pilote sous-jacent (§17.1) |
|--------------|-----|---------|----------------------------|
| [17.3.1](03.1-hibernate.md) | **Hibernate** (JPA) | Java | JDBC / Connector/J |
| [17.3.2](03.2-sqlalchemy.md) | **SQLAlchemy** | Python | PyMySQL, mysqlclient… |
| [17.3.3](03.3-sequelize.md) | **Sequelize** | Node.js | mysql2 / mariadb |
| [17.3.4](03.4-prisma.md) | **Prisma** | Node.js / TypeScript | moteur Prisma |
| [17.3.5](03.5-entity-framework-core.md) | **Entity Framework Core** | .NET | MySqlConnector (via Pomelo) |

---

## Ce qu'il faut retenir

- Un ORM comble le **décalage d'impédance** entre objets et tables ; il se construit **au-dessus** du connecteur (§17.1) et du pool (§17.2), qu'il ne remplace pas.
- Concepts communs : entité, mapping, identité, associations, **chargement paresseux/anticipé**, unité de travail, langage de requête + **SQL brut** en secours.
- Deux familles — **Active Record** (Sequelize) et **Data Mapper** (Hibernate, SQLAlchemy, EF Core) — **Prisma** étant un client typé généré, à part.
- Bénéfices (productivité, sécurité, moins de code) **vs** coûts (abstraction qui fuit, SQL généré parfois sous-optimal).
- Le piège **N+1** est le problème de performance n°1 : y remédier par le **chargement anticipé** et surveiller le SQL exécuté (§17.4).
- Combiner ORM (CRUD) et **SQL brut** (analytique, performance) ; penser au **dialecte MariaDB** et aux **migrations** (§17.5).

⏭️ [Hibernate (Java)](/17-integration-developpement/03.1-hibernate.md)

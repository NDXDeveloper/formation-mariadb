🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.1 — Qu'est-ce que MariaDB ?

> 🧭 Première section du chapitre. Objectif : définir précisément MariaDB et en cerner la nature, avant d'explorer son histoire (§1.2), son écosystème (§1.3) et son architecture (§1.4).

## Une définition

**MariaDB** est un **système de gestion de bases de données relationnelles** (SGBDR) **open source**. Concrètement, c'est un logiciel serveur qui stocke des données sous forme de *tables* — des lignes et des colonnes — et qui permet de les interroger et de les manipuler à l'aide du langage **SQL** (*Structured Query Language*).

Par exemple, une table `client` se présente comme un tableau à deux dimensions :

| id | nom    | ville  |
|----|--------|--------|
| 1  | Dupont | Lyon   |
| 2  | Martin | Nantes |

Chaque **ligne** correspond à un client, chaque **colonne** (`id`, `nom`, `ville`) à un attribut, et la colonne `id` sert d'**identifiant unique**. C'est cette organisation en tables — et la possibilité de les **relier** entre elles — qui caractérise le modèle relationnel, détaillé en [§1.4](04-architecture-generale-sgbd.md).

Issu du code de MySQL et développé par ses créateurs d'origine, MariaDB est aujourd'hui l'une des bases de données relationnelles libres les plus répandues au monde, présente aussi bien sur des postes de développement que dans de grandes infrastructures de production.

## Un logiciel libre et open source

Le serveur MariaDB est distribué sous licence **GPL v2**, une licence libre qui garantit que le code source est ouvert, consultable et modifiable. Son utilisation est donc gratuite, sans coût de licence ; son développement se fait de manière transparente, au sein d'une communauté ; et aucune entreprise ne peut « refermer » le cœur du projet à elle seule. Cette ouverture est l'une des raisons d'être historiques de MariaDB et le distingue de bases de données propriétaires comme Oracle Database ou Microsoft SQL Server.

Cette licence concerne le **serveur**. Les **bibliothèques clientes** (les connecteurs par lesquels les applications se connectent) sont, elles, publiées sous **LGPL**, plus permissive : on peut donc intégrer MariaDB dans un logiciel propriétaire sans que celui-ci hérite des obligations de la GPL (voir le chapitre 17).

## Une origine MySQL

MariaDB est né en 2009 comme un **fork** (une branche dérivée) de MySQL, lancé par Michael « Monty » Widenius — le créateur originel de MySQL — entouré de plusieurs développeurs historiques du projet. Le nom rend hommage à l'une de ses filles, Maria (MySQL ayant été baptisé d'après l'autre, My).

Pendant longtemps, MariaDB a été conçu comme un **remplacement direct** (*drop-in replacement*) de MySQL : mêmes formats de fichiers, mêmes protocoles, mêmes commandes. Avec le temps, les deux projets ont toutefois divergé et possèdent désormais chacun des fonctionnalités propres. *Cette histoire et ces différences sont détaillées dans la section [1.2](02-histoire-et-differences-mysql.md).*

## Les caractéristiques essentielles

Au-delà de la définition, quelques propriétés résument bien la nature de MariaDB :

- **Relationnel et conforme à SQL** : les données sont organisées en tables reliées entre elles, et manipulées via SQL (proche du standard, avec des extensions propres à MariaDB).
- **Transactionnel et ACID** : grâce à son moteur par défaut **InnoDB**, MariaDB préserve l'intégrité des données même en cas d'incident, via des transactions atomiques, cohérentes, isolées et durables.
- **Architecture client-serveur** : un processus serveur gère les données ; les applications s'y connectent en tant que clients, en local ou via le réseau. *(Voir [1.4](04-architecture-generale-sgbd.md).)*
- **Moteurs de stockage enfichables** : MariaDB peut s'appuyer sur différents moteurs selon le besoin (InnoDB pour le transactionnel, ColumnStore pour l'analytique, Aria, etc.), une particularité héritée de MySQL.
- **Multiplateforme** : il fonctionne sous Linux, Windows et macOS, ainsi que dans des conteneurs (Docker, Kubernetes).
- **Moderne** : les versions récentes intègrent des fonctionnalités avancées comme le type JSON, les tables temporelles ou la recherche vectorielle pour l'IA, abordées plus loin dans la formation.

## Qui développe et maintient MariaDB ?

Le projet repose sur un modèle à deux entités complémentaires. La **MariaDB Foundation**, organisation à but non lucratif, veille à ce que le serveur reste libre, ouvert et accessible à la communauté. En parallèle, une **société commerciale** (MariaDB Corporation / MariaDB plc) propose des produits, du support et des services destinés aux entreprises : offres « Enterprise », outils additionnels, accompagnement. Cette double structure permet à MariaDB d'être à la fois un projet communautaire ouvert et une solution soutenue professionnellement.

## « MariaDB », un nom pour plusieurs réalités

Dans cette formation, **MariaDB** désigne avant tout le **serveur de base de données** — le cœur du système, objet de la quasi-totalité des chapitres. Mais le terme recouvre aussi une **famille de produits et d'outils** gravitant autour de ce serveur : le proxy MaxScale, le moteur analytique ColumnStore, les connecteurs pour différents langages, etc. *Cet écosystème est présenté dans la section [1.3](03-cas-usage-et-ecosysteme.md).*

## Ce que MariaDB n'est pas

Pour bien cerner l'outil, il est utile de préciser ce qu'il n'est pas :

- ce n'est **pas une base NoSQL** au sens premier : son modèle reste relationnel, même s'il intègre des capacités JSON et vectorielles ;
- ce n'est **pas un simple clone figé de MySQL** : les deux projets évoluent désormais de façon indépendante ;
- ce n'est **pas réservé aux grandes entreprises** : il s'installe et s'utilise tout aussi bien sur un poste de développement local.

## À retenir

MariaDB est un **SGBDR open source**, issu de MySQL et développé par ses créateurs d'origine, distribué sous licence libre GPL v2. Il stocke les données en tables relationnelles interrogées en SQL, assure l'intégrité transactionnelle via InnoDB, repose sur une architecture client-serveur à moteurs de stockage enfichables, et fonctionne sur la plupart des plateformes. La version de référence de cette formation est la **12.3 LTS**.

---

**Navigation** : [⬆️ Chapitre 1 — Introduction et Fondamentaux](README.md) · Section suivante → [1.2 Histoire et différences avec MySQL](02-histoire-et-differences-mysql.md)

⏭️ [Histoire et différences avec MySQL](/01-introduction-fondamentaux/02-histoire-et-differences-mysql.md)

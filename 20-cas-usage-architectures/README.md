🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20. Cas d'Usage et Architectures

> **Partie 10 — Migration, Compatibilité et Architectures**  
> Niveau : Avancé · Public : Développeurs, DevOps, DBAs  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Ce chapitre clôt la formation et en constitue, à bien des égards, la **synthèse**. Les chapitres précédents ont présenté MariaDB « brique par brique » : le langage SQL, l'indexation, les transactions, les moteurs de stockage, la programmation côté serveur, la sécurité, l'administration, la sauvegarde, la réplication, la haute disponibilité, le tuning, le DevOps et l'intégration applicative. L'objectif est désormais d'assembler ces briques pour répondre à une question concrète et récurrente sur le terrain : **quelle architecture mettre en place pour tel besoin ?**

Car il n'existe pas d'architecture MariaDB « universelle ». Le bon choix dépend toujours d'un faisceau de contraintes : la nature de la charge (transactionnelle ou analytique), le volume de données, le niveau de disponibilité exigé, la répartition géographique des utilisateurs, le modèle de cloisonnement des clients (multi-tenant), et les besoins d'intégration (flux d'événements, intelligence artificielle). Ce chapitre passe en revue les grands cas d'usage rencontrés en production et montre comment les fonctionnalités de MariaDB 12.3 — d'InnoDB à ColumnStore, de Galera à MaxScale, de Spider à la recherche vectorielle (Vector) — se combinent pour y répondre.

L'approche est volontairement orientée **décision** : pour chaque cas d'usage, l'enjeu est de comprendre les compromis (isolation contre densité, cohérence contre latence, simplicité contre élasticité) afin de faire un choix éclairé plutôt que d'appliquer une recette toute faite.

---

## Pourquoi ce chapitre est important

Maîtriser une fonctionnalité isolée (par exemple savoir configurer la réplication ou créer un index vectoriel) ne suffit pas à concevoir un système robuste. La valeur d'un architecte de données se situe dans sa capacité à **articuler** ces fonctionnalités au service d'objectifs métier, en arbitrant entre des exigences souvent contradictoires.

Ce chapitre vise précisément à développer cette compétence : passer de la connaissance des outils à la conception de systèmes. Il s'appuie sur tout ce qui précède et illustre, à travers des patterns architecturaux éprouvés et des études de cas, comment ces patterns se déclinent concrètement avec MariaDB.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- **Distinguer les charges OLTP et OLAP** et choisir le moteur de stockage et la configuration adaptés à chacune.
- **Concevoir l'architecture de données d'applications distribuées**, qu'il s'agisse de microservices (base par service ou base partagée) ou d'applications multi-tenant (base par locataire, schéma par locataire, ou schéma partagé avec discriminateur).
- **Mettre en œuvre un entrepôt de données analytique** avec le moteur ColumnStore.
- **Planifier des déploiements géo-distribués, hybrides et multi-cloud**, en comprenant leurs implications sur la cohérence et la latence.
- **Arbitrer entre scaling vertical et scaling horizontal** selon le contexte et les contraintes de coût.
- **Intégrer MariaDB dans des architectures orientées événements** via la capture de changements (CDC), Kafka et Debezium.
- **Construire des cas d'usage d'intelligence artificielle** (recherche sémantique, RAG, moteurs de recommandation, détection d'anomalies, recherche hybride) en exploitant MariaDB Vector, le serveur MCP et les frameworks d'IA.
- **Vous appuyer sur des études de cas réels** pour ancrer ces concepts dans des situations concrètes.

---

## Prérequis

Ce chapitre étant un chapitre de synthèse, il mobilise des notions vues précédemment. Il est recommandé d'être à l'aise avec :

| Domaine | Chapitre(s) de référence |
|---------|--------------------------|
| Moteurs de stockage (InnoDB, ColumnStore, Spider, S3) | Chapitre 7 |
| Réplication (asynchrone, GTID, multi-source) | Chapitre 13 |
| Haute disponibilité (Galera, MaxScale) | Chapitre 14 |
| Performance, partitionnement et sharding | Chapitre 15 |
| DevOps, conteneurs et cloud | Chapitre 16 |
| Intégration applicative et connecteurs | Chapitre 17 |
| MariaDB Vector et recherche vectorielle | Chapitre 18 (§18.10) |

Aucune nouvelle syntaxe SQL fondamentale n'est introduite ici : l'accent porte sur l'**assemblage** et le **choix architectural**.

---

## Vue d'ensemble du chapitre

Les douze sections de ce chapitre peuvent se regrouper en six grands axes :

| Axe | Sections | Objet |
|-----|----------|-------|
| **Nature de la charge** | 20.1 | Différencier traitement transactionnel et analytique |
| **Patterns applicatifs distribués** | 20.2, 20.4 | Microservices et multi-tenant |
| **Analytique et entreposage** | 20.3 | Data warehousing avec ColumnStore |
| **Distribution et mise à l'échelle** | 20.5, 20.6, 20.7 | Géo-distribution, cloud hybride/multi-cloud, scaling |
| **Architectures événementielles** | 20.8 | CDC, Kafka, Debezium |
| **Intelligence artificielle** | 20.9, 20.10, 20.11 | RAG, recherche vectorielle, MCP Server, frameworks |
| **Mise en pratique** | 20.12 | Études de cas réels |

---

## Les grands axes abordés

### Nature de la charge : OLTP et OLAP

Toute réflexion architecturale commence par la **caractérisation de la charge**. Une application transactionnelle (OLTP) traite de nombreuses opérations courtes et concurrentes sur un schéma normalisé : elle s'appuie naturellement sur InnoDB, son verrouillage au niveau ligne et son modèle ACID. Une charge analytique (OLAP) exécute au contraire un faible nombre de requêtes complexes balayant de grands volumes : elle tire parti d'un stockage orienté colonnes comme ColumnStore. Cette distinction fondamentale conditionne le choix du moteur, du schéma et de l'infrastructure, et structure l'ensemble du chapitre.

### Patterns applicatifs distribués

Les architectures modernes éclatent souvent une application monolithique en composants indépendants. Le chapitre examine deux familles de patterns. Les **microservices** posent la question de la propriété des données : faut-il une base par service (couplage faible, autonomie forte) ou une base partagée (simplicité, mais couplage accru) ? Le **multi-tenant** soulève la question du cloisonnement entre clients, sur un continuum allant de l'isolation maximale (une base par locataire) à la densité maximale (un schéma partagé distingué par un identifiant de locataire), en passant par des solutions intermédiaires. Chaque approche présente un arbitrage distinct entre isolation, coût d'exploitation et facilité de maintenance.

### Analytique et entreposage de données

Au-delà du simple OLAP, la construction d'un véritable entrepôt de données avec ColumnStore est traitée comme un cas d'usage à part entière : modélisation en étoile, compression colonnaire, et complémentarité avec une base InnoDB transactionnelle en amont.

### Distribution géographique et mise à l'échelle

Lorsque les utilisateurs sont répartis sur plusieurs continents, ou lorsque le système doit survivre à la perte d'un site, la **géo-distribution** et les déploiements **hybrides ou multi-cloud** deviennent incontournables — avec leurs implications sur la cohérence, la latence et la résilience. Cet axe est indissociable de la question du **scaling** : faut-il une machine plus puissante (vertical) ou davantage de machines (horizontal, via réplicas de lecture, Galera ou sharding avec Spider) ? Les sections correspondantes mettent en regard les solutions de MariaDB (Galera, MaxScale, moteur S3 pour l'archivage de données froides) avec ces contraintes.

### Architectures orientées événements

MariaDB ne vit pas en isolation. Dans un système moderne, ses changements de données doivent souvent alimenter d'autres composants en temps réel. Le chapitre montre comment exploiter la **capture de changements (CDC)** à partir du journal binaire, et comment l'intégrer à un bus d'événements via **Kafka** et le connecteur **Debezium**, ouvrant la voie aux architectures pilotées par les événements.

### Cas d'usage en intelligence artificielle

C'est l'un des apports les plus marquants des versions récentes : grâce au type `VECTOR` et à l'index HNSW — en préversion dès la 11.6 et **GA dans la 11.8 LTS**, donc pleinement présents en 12.3 —, MariaDB devient une base de données apte à porter des charges d'IA. Le chapitre détaille la **recherche sémantique**, le pattern **RAG** (Retrieval-Augmented Generation), les **moteurs de recommandation**, la **détection d'anomalies** et la **recherche hybride** combinant vecteurs et filtres SQL relationnels. Il aborde également le **serveur MCP** de MariaDB et l'intégration avec les frameworks d'IA tels que LangChain ou LlamaIndex, permettant d'utiliser MariaDB comme socle de données pour des applications fondées sur les grands modèles de langage.

### Études de cas réels

Enfin, le chapitre se conclut par des études de cas qui rassemblent ces différents axes dans des situations concrètes, afin de montrer comment les compromis se tranchent dans la pratique.

---

## Une démarche guidée par les compromis

Le fil conducteur de ce chapitre est la notion de **compromis** (*trade-off*). Aucune architecture n'optimise simultanément l'isolation, la cohérence, la latence, le coût et la simplicité d'exploitation : améliorer un critère se fait généralement au détriment d'un autre. L'objectif n'est donc pas de mémoriser une architecture « idéale », mais d'acquérir une grille de lecture permettant, face à un besoin donné, d'identifier les contraintes dominantes et de choisir la combinaison de fonctionnalités MariaDB la plus pertinente.

---

## Navigation

➡️ Section suivante : [20.1 OLTP vs OLAP](01-oltp-vs-olap.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [OLTP vs OLAP](/20-cas-usage-architectures/01-oltp-vs-olap.md)

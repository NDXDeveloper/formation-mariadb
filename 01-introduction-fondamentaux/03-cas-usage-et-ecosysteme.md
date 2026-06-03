🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.3 — Cas d'usage et écosystème

> 🧭 Cette section dresse un panorama des domaines où MariaDB est utilisé et présente les outils qui l'entourent. Les architectures et cas d'usage avancés sont approfondis au chapitre 20 ; les composants cités ici renvoient à leurs chapitres respectifs.

## Un SGBD polyvalent

MariaDB est une base de données **généraliste** : elle convient aussi bien à un petit site personnel qu'à des systèmes critiques traitant des milliers de transactions par seconde. Cette polyvalence, héritée de MySQL et enrichie au fil des versions, explique sa présence dans des contextes très variés — du poste de développement jusqu'au cloud à grande échelle. On la retrouve d'ailleurs dans des organisations de premier plan : **Wikipédia** (fondation Wikimedia, qui a migré de MySQL vers MariaDB dès 2013), **ServiceNow**, **Google** ou **Mozilla** l'exploitent en production, à très grande échelle.

## Cas d'usage typiques

Voici les grandes familles d'usage où MariaDB est couramment déployé :

- **Sites web et CMS** : c'est l'usage historique. MariaDB est le « M » des piles *LAMP* et *LEMP*, et la base par défaut de nombreux CMS et plateformes e-commerce (WordPress, Drupal, Joomla, Magento, PrestaShop…).
- **Applications transactionnelles (OLTP)** : applications métier, ERP, systèmes de réservation ou bancaires, caractérisés par une forte concurrence et une faible latence. *(Voir §20.1.)*
- **Analytique et entrepôts de données (OLAP)** : grâce au moteur **ColumnStore**, MariaDB adresse aussi les charges analytiques sur de gros volumes. *(Voir §7.5 et §20.3.)*
- **Architectures microservices et cloud-native** : déploiement en conteneurs, orchestration Kubernetes, base par service. *(Voir chapitre 16 et §20.2.)*
- **Applications multi-tenant / SaaS** : hébergement de nombreux clients sur une infrastructure mutualisée. *(Voir §20.4.)*
- **IA, RAG et recherche vectorielle** : avec le type **VECTOR** et l'index HNSW, MariaDB stocke et interroge des *embeddings* pour des applications d'intelligence artificielle. *(Voir §18.10 et §20.9.)*
- **Remplacement de MySQL** : MariaDB reste un choix naturel pour les organisations souhaitant migrer depuis MySQL tout en restant dans un écosystème ouvert. *(Voir chapitre 19.)*

Sa légèreté lui permet par ailleurs de tourner sur du matériel modeste (instances réduites, *edge*), ce qui élargit encore son champ d'application.

## L'écosystème MariaDB

Le serveur n'est que le cœur d'un ensemble plus vaste d'outils et de composants. Les principaux sont :

| Composant | Rôle | Détaillé dans |
|-----------|------|---------------|
| **Serveur MariaDB** | Le SGBD lui-même, avec ses moteurs de stockage enfichables (InnoDB, Aria, ColumnStore, Spider, S3, Vector…) | Chapitre 7 |
| **MaxScale** | Proxy de base de données avancé : répartition de charge, *read/write split*, routage et pare-feu de requêtes | §14.4–14.5 |
| **Galera Cluster** | Réplication synchrone multi-maître pour la haute disponibilité | §14.2 |
| **Connecteurs / drivers** | Bibliothèques de connexion pour PHP, Python, Java, Node.js, Go, .NET, C/C++… | §17.1 |
| **Outils CLI** | Client `mariadb`, `mariadb-dump`, `mariadb-admin`, sauvegarde `Mariabackup`… | §1.9, chapitre 12, annexe B |
| **Outils graphiques tiers** | HeidiSQL, DBeaver, phpMyAdmin | §1.9 |
| **mariadb-operator** | Opérateur Kubernetes pour déployer et gérer MariaDB | §16.5 |
| **MCP Server** | Passerelle d'intégration avec les outils d'IA | §20.10 |
| **Supervision** | Export de métriques (`mysqld_exporter`) vers Prometheus / Grafana | §16.9 |

Cet écosystème est **en grande partie open source**, à quelques exceptions près. La plus notable est le proxy **MaxScale**, devenu **propriétaire** à partir de sa version **25.01** (janvier 2025) — il était auparavant distribué sous *Business Source License* (BSL, code « source ouverte » mais à usage restreint, gratuit jusqu'à deux serveurs en arrière-plan). Le serveur, les connecteurs, l'opérateur Kubernetes et la supervision restant libres, on peut néanmoins bâtir des architectures complètes en s'appuyant largement sur des briques ouvertes — avec, pour le proxy, des **alternatives libres** comme ProxySQL ou HAProxy (§14.9).

## Édition communautaire et offre Enterprise

Comme évoqué précédemment, MariaDB se décline en deux grandes formes. La **Community Server**, libre et gratuite, est la version utilisée tout au long de cette formation. À côté, une **offre Enterprise** commerciale ajoute des outils, des optimisations et un support professionnel destinés aux environnements de production exigeants (par exemple l'Enterprise Operator pour Kubernetes, §16.6). Pour l'apprentissage et la grande majorité des projets, l'édition communautaire est pleinement suffisante.

## Une communauté active

Au-delà des outils, MariaDB s'appuie sur une **communauté mondiale** de contributeurs, d'utilisateurs et d'entreprises. Celle-ci alimente une documentation officielle fournie, des forums d'entraide et des événements réguliers. Les ressources utiles pour aller plus loin sont rassemblées dans l'**annexe H**.

## À retenir

MariaDB est un SGBD **polyvalent**, présent du site web personnel aux architectures cloud distribuées, en passant par l'analytique (ColumnStore) et, plus récemment, l'IA (recherche vectorielle). Il s'inscrit dans un **écosystème riche, en grande partie open source** — MaxScale, Galera, connecteurs multi-langages, opérateur Kubernetes, outils CLI et graphiques — couvrant l'ensemble du cycle de vie d'une base de données. L'édition **communautaire**, libre, suffit à la plupart des usages, l'offre **Enterprise** ciblant les besoins de production avancés.

---

**Navigation** : [⬆️ Chapitre 1 — Introduction et Fondamentaux](README.md) · Section précédente : [1.2 Histoire et différences avec MySQL](02-histoire-et-differences-mysql.md) · Section suivante → [1.4 Architecture générale d'un SGBD relationnel](04-architecture-generale-sgbd.md)

⏭️ [Architecture générale d'un SGBD relationnel](/01-introduction-fondamentaux/04-architecture-generale-sgbd.md)

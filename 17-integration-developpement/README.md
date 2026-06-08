🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17. Intégration et Développement

> **Partie 9 — Intégration, Développement et Fonctionnalités Avancées**  
> Parcours concernés : **Développeur** · IA/ML  
> Prérequis : Bases du SQL (chap. 2-4), Transactions (chap. 6), Sécurité et utilisateurs (chap. 10)  
> Version de référence : **MariaDB 12.3 LTS**

---

## Pourquoi ce chapitre ?

Jusqu'ici, la formation a exploré MariaDB « de l'intérieur » : le langage SQL, les moteurs de stockage, les transactions, la sécurité, l'administration, la réplication et le tuning. Mais une base de données ne vit presque jamais seule : elle se trouve au bout d'une chaîne applicative. Site web, API REST, microservice, traitement batch, pipeline de données ou agent IA — tous finissent par ouvrir une connexion vers le serveur et lui envoyer des requêtes.

Ce chapitre traite précisément de cette **frontière entre le code applicatif et le serveur MariaDB**. C'est une zone à la fois critique et souvent négligée. Une part importante des incidents en production n'a pas pour origine le moteur lui-même, mais la façon dont l'application l'utilise :

- connexions ouvertes et jamais refermées (fuites de connexions, saturation de `max_connections`) ;
- requêtes construites par concaténation de chaînes, donc vulnérables aux **injections SQL** ;
- explosion du nombre de requêtes générées par un ORM mal maîtrisé (problème dit « N+1 ») ;
- migrations de schéma appliquées à la main, non reproductibles, voire jouées dans le désordre entre environnements ;
- code base de données (procédures, requêtes complexes) déployé sans aucun test automatisé.

Bien intégrer MariaDB dans une application, ce n'est donc pas seulement « savoir se connecter ». C'est concevoir une couche d'accès aux données **fiable, sécurisée, performante et maintenable**.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- vous connecter à MariaDB depuis les principaux langages de l'écosystème (PHP, Python, Java, Node.js, Go, .NET) en utilisant les pilotes adaptés ;
- mettre en place et dimensionner un **pool de connexions**, et comprendre où le placer (application ou proxy) ;
- utiliser un **ORM** de façon éclairée, en connaissant ses apports comme ses limites ;
- appliquer les **bonnes pratiques** de développement avec une base relationnelle ;
- gérer les **migrations de schéma** de manière versionnée et reproductible ;
- mettre en place des **tests** automatisés ciblant la base de données ;
- construire des **environnements de développement** cohérents et jetables ;
- prévenir les **injections SQL** grâce aux requêtes paramétrées et aux *prepared statements*.

---

## Plan du chapitre

Le chapitre suit une progression logique, du plus bas niveau (la connexion brute) vers les pratiques d'ingénierie qui encadrent l'usage de la base au quotidien.

| Section | Sujet | En bref |
|---------|-------|---------|
| **17.1** | [Connexion depuis différents langages](01-connexion-langages.md) | Le socle : ouvrir une connexion fiable depuis PHP, Python, Java, Node.js, Go et .NET. |
| **17.2** | [Connection pooling](02-connection-pooling.md) | Réutiliser les connexions pour tenir la charge ; pool applicatif vs proxy. |
| **17.3** | [ORM et frameworks](03-orm-frameworks.md) | Hibernate, SQLAlchemy, Sequelize, Prisma, EF Core : abstraction, productivité et pièges. |
| **17.4** | [Bonnes pratiques de développement](04-bonnes-pratiques.md) | Gestion des connexions, des erreurs, des transactions côté application. |
| **17.5** | [Gestion des migrations de schéma](05-gestion-migrations-schema.md) | Faire évoluer le schéma de façon versionnée et reproductible. |
| **17.6** | [Tests de bases de données](06-tests-bases-donnees.md) | Tester le code qui touche aux données, sans fragiliser la suite de tests. |
| **17.7** | [Environnements de développement](07-environnements-developpement.md) | Des environnements locaux cohérents avec la production. |
| **17.8** | [Prévention des injections SQL](08-prevention-injections-sql.md) | La vulnérabilité applicative n°1 et comment l'éliminer par construction. |
| **17.9** | [Prepared statements et parameterized queries](09-prepared-statements.md) | Le mécanisme qui sécurise *et* accélère ; inclut les curseurs sur *prepared statements*. |

Les premières sections (17.1 à 17.3) répondent à la question « **comment** parler à MariaDB » ; les suivantes (17.4 à 17.9) à la question « comment le faire **bien** ».

---

## Positionnement dans l'écosystème MariaDB 12.3

Deux points de contexte propres à la version de référence méritent d'être gardés à l'esprit dès le départ :

- **Les connecteurs évoluent indépendamment du serveur.** Connector/J, Connector/Python, Connector/Node.js, Connector/C, etc. suivent leur propre cycle de versions. Une bonne pratique consiste à toujours associer une version de connecteur récente à votre serveur 12.3, plutôt que de raisonner uniquement en termes de version serveur.

- **La compatibilité d'authentification avec MySQL est renforcée.** MariaDB 12.3 prend en charge le plugin `caching_sha2_password` (voir §10.5.5), ce qui simplifie la connexion d'outils et de pilotes initialement conçus pour MySQL 8. Ce détail, en apparence mineur, évite bien des blocages au moment de configurer un connecteur.

Par ailleurs, la section 17.9 met en lumière une nouveauté de la série 12.x : la possibilité d'utiliser des **curseurs reposant sur des *prepared statements*** (détaillée au §8.5.1), utile notamment lors de migrations depuis Oracle et pour certains patterns d'accès dynamiques.

---

## Pour qui ?

Ce chapitre s'adresse en priorité aux **développeurs** et développeuses qui intègrent MariaDB dans une application, ainsi qu'aux profils **IA/ML** appelés à interroger la base (y compris en recherche vectorielle, voir chap. 18 et 20). Les DBA y trouveront un éclairage utile sur la manière dont les applications sollicitent le serveur — connaissance précieuse pour diagnostiquer un comportement côté production.

Aucune connaissance d'un langage de programmation particulier n'est exigée pour suivre les concepts ; les exemples couvrent toutefois plusieurs langages afin que chacun puisse se rattacher à son environnement habituel.

⏭️ [Connexion depuis différents langages](/17-integration-developpement/01-connexion-langages.md)

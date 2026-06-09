🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.10 MariaDB MCP Server pour intégration IA

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La section §20.9 a montré que MariaDB peut porter des charges d'IA grâce à la recherche vectorielle. Reste à **intégrer** la base à l'écosystème des outils d'IA — assistants, agents, environnements de développement. Le **serveur MCP de MariaDB** répond précisément à ce besoin : il expose la base aux applications d'IA selon un protocole standardisé, le **Model Context Protocol** (MCP).

Cette section présente le protocole MCP, le serveur MCP de MariaDB, les outils qu'il met à disposition, et — point essentiel — les enjeux de sécurité liés au fait de donner à un agent d'IA l'accès à une base de données.

---

## Qu'est-ce que le protocole MCP ?

Le **Model Context Protocol** (MCP) est un **standard ouvert**, publié par Anthropic en novembre 2024, qui définit une manière normalisée de connecter les modèles de langage et les systèmes d'IA à des **outils** et **sources de données** externes. Il repose sur une architecture **client-serveur** : un **serveur MCP** expose des capacités (outils à appeler, ressources à lire), qu'un **client MCP** — l'application d'IA — peut découvrir et utiliser.

Le problème que MCP résout est concret : sans standard, chaque éditeur d'outil ou de base devrait développer des intégrations spécifiques pour chaque application d'IA, et réciproquement. MCP offre un **langage commun** et **agnostique du modèle**, qui standardise ces intégrations tout en gérant la sécurité. Comme il est ouvert et extensible, de nouveaux schémas d'outils peuvent être définis selon les besoins. On le compare souvent à un « connecteur universel » pour les LLM, à l'image de ce qu'a apporté un port standard pour les périphériques.

---

## Le serveur MCP de MariaDB

Le **serveur MCP de MariaDB** est une implémentation de ce protocole, dédiée à MariaDB. Son objectif est de permettre à un assistant ou à un agent d'IA d'interagir avec une base MariaDB de façon standardisée — explorer le schéma, exécuter des requêtes, et effectuer de la **recherche vectorielle**. Il prend ainsi en charge à la fois les **opérations relationnelles classiques** et les **capacités vectorielles** devenues essentielles aux applications d'IA modernes.

Ce projet, **open source** et agnostique du modèle, est né d'une contribution communautaire (lauréate du hackathon RAG de MariaDB) que MariaDB a ensuite reprise pour l'enrichir. Il en existe aussi une déclinaison **Enterprise**, plus durcie, conçue comme l'interface principale et sécurisée entre les assistants d'IA et les écosystèmes de données MariaDB, intégrant notamment un service de RAG et des mécanismes d'authentification avancés.

Concrètement, le serveur fait office de **pont** : des outils comme un assistant conversationnel ou un environnement de développement assisté par IA s'y connectent et traduisent des **interactions en langage naturel** en opérations sur la base.

---

## Les outils exposés

Le serveur MCP de MariaDB expose un ensemble d'**outils** que l'agent d'IA peut appeler via le protocole. On y trouve typiquement :

- **lister les bases** accessibles et les **tables** d'une base ;
- **décrire le schéma** d'une table (colonnes, types, clés), y compris ses **relations de clés étrangères** ;
- **exécuter une requête SQL en lecture seule** (`SELECT`, `SHOW`, `DESCRIBE`) ;
- gérer des **magasins de vecteurs** : en créer, en supprimer, en lister ;
- **insérer par lots** des documents (avec d'éventuelles métadonnées) dans un magasin de vecteurs ;
- effectuer une **recherche sémantique** : retrouver les documents similaires à partir d'une requête en langage naturel.

Deux points méritent d'être soulignés. D'abord, l'exécution de requêtes est **en lecture seule** par défaut, ce qui constitue une garde-fou important : l'agent peut interroger, mais pas modifier les données par ce canal. Ensuite, les capacités vectorielles sont **optionnelles** : on peut les activer en configurant un fournisseur d'embeddings, ou les désactiver si seules les opérations relationnelles sont nécessaires.

---

## Embeddings et fournisseurs

Le serveur MCP ajoute une dimension par rapport à l'usage en SQL « brut » du §20.9. Là où l'application devait elle-même produire les embeddings avant de les insérer, le serveur MCP peut **orchestrer le calcul des embeddings** en s'appuyant sur un **fournisseur** configurable — modèles d'OpenAI ou modèles HuggingFace (par exemple des modèles multilingues comme *e5* ou *bge-m3*). Lors d'une recherche sémantique, il encode la requête en langage naturel avec ce fournisseur, puis effectue la recherche vectorielle dans MariaDB.

Le serveur fournit ainsi un **flux RAG intégré** : ingestion de documents, génération d'embeddings et recherche sémantique, le tout exposé comme des outils à l'agent d'IA, sans que celui-ci ait à manipuler directement le SQL vectoriel.

---

## Sécurité et déploiement

Donner à un agent d'IA l'accès à une base de données est une décision **sensible**, qui exige des garde-fous rigoureux. Le serveur MCP de MariaDB et son écosystème prévoient plusieurs niveaux de protection :

- **Lecture seule par défaut** pour l'exécution de requêtes, limitant le risque de modification non désirée.
- **Authentification** : prise en charge de mécanismes standards (par exemple OAuth) pour contrôler quels clients peuvent se connecter.
- **Identifiants gérés par configuration** (variables d'environnement), séparés des clients d'IA.
- En version Enterprise, un **point d'accès durci** avec validation des jetons sur plusieurs niveaux, un service de **RAG** dédié, et un enregistrement **adaptatif** des outils permettant d'activer ou de désactiver dynamiquement les capacités.

À ces dispositifs propres au serveur, il faut **ajouter** les bonnes pratiques du chapitre 10 : faire se connecter le serveur MCP avec un **utilisateur dédié** aux privilèges minimaux (en lecture seule, restreint aux bases pertinentes), et encadrer l'accès réseau. Le principe du moindre privilège est ici fondamental : un agent d'IA ne doit disposer que des droits strictement nécessaires.

Côté déploiement, le serveur se lance typiquement via une **image Docker** ou depuis ses sources, l'ensemble des paramètres (connexion à la base, clés du fournisseur d'embeddings) étant fourni par des **variables d'environnement** :

```bash
# Configuration du serveur MCP MariaDB (illustratif, variables d'environnement)
DB_HOST=mariadb
DB_USER=mcp_lecture_seule        # utilisateur dédié, privilèges minimaux
DB_PASSWORD=...
EMBEDDING_PROVIDER=openai        # ou un modèle HuggingFace
OPENAI_API_KEY=...
```

> **Note.** Les noms exacts des variables, des outils et des options dépendent de l'implémentation et de la version ; il convient de se reporter à la documentation et au dépôt du serveur MCP de MariaDB pour les détails précis.

---

## Cas d'usage

Le serveur MCP ouvre plusieurs usages concrets :

- **Exploration et interrogation en langage naturel** : depuis un assistant ou un environnement de développement (par exemple un assistant conversationnel ou un IDE assisté par IA), un utilisateur interroge la base **sans écrire de SQL**, l'agent traduisant la demande en requêtes via les outils MCP.
- **Backend de RAG** : le serveur sert de socle de récupération pour des applications de RAG (§20.9), en gérant l'ingestion, les embeddings et la recherche sémantique.
- **Flux agentiques** : un agent autonome peut enchaîner exploration de schéma, requêtes et recherche vectorielle pour accomplir une tâche de bout en bout.
- **Démocratisation de l'accès aux données** : des utilisateurs non experts en SQL peuvent obtenir des réponses à partir des données, dans le cadre des permissions accordées.

Ces usages combinent naturellement les acquis du chapitre : la recherche vectorielle de MariaDB (§20.9), le respect du cloisonnement et des permissions (au sens de §20.4 et du chapitre 10), exposés à l'écosystème d'IA.

---

## Synthèse

Le serveur MCP de MariaDB est le **pont standardisé** qui expose la base — opérations relationnelles **et** recherche vectorielle — aux assistants et agents d'IA, via le protocole ouvert MCP. Il peut même orchestrer le flux RAG complet (embeddings et recherche sémantique). Sa mise en œuvre n'a de sens qu'**assortie de garde-fous de sécurité stricts** : lecture seule, authentification, et surtout un utilisateur aux privilèges minimaux, selon le principe du moindre privilège.

Le serveur MCP constitue une **voie d'intégration** parmi d'autres : il s'adresse principalement aux assistants et agents conversationnels. Une autre voie, complémentaire, consiste à intégrer MariaDB **dans le code** d'applications d'IA à l'aide de frameworks dédiés. C'est l'objet de la section suivante, consacrée aux intégrations avec des frameworks tels que **LangChain** ou **LlamaIndex** (§20.11).

---

## Navigation

⬅️ Section précédente : [20.9.4 Hybrid Search (vecteurs + SQL relationnel)](09.4-hybrid-search.md)  
➡️ Section suivante : [20.11 Intégrations frameworks IA (LangChain, LlamaIndex, etc.)](11-integrations-frameworks-ia.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Intégrations frameworks IA (LangChain, LlamaIndex, etc.)](/20-cas-usage-architectures/11-integrations-frameworks-ia.md)

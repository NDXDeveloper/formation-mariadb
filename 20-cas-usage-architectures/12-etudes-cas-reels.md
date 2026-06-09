🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.12 Études de cas réels

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Cette dernière section confronte l'ensemble des concepts du chapitre — du cloisonnement à l'intelligence artificielle — à des situations concrètes. MariaDB est, rappelons-le, une base **mature et largement déployée** en production à travers de nombreux secteurs ; elle est issue d'une scission de MySQL menée par ses créateurs d'origine, et fournie par défaut dans de grandes distributions Linux. Les architectures réelles qui s'appuient sur elle combinent presque toujours **plusieurs** des patterns étudiés ici.

Les études qui suivent sont des **cas représentatifs** — des scénarios types, inspirés de situations courantes — plutôt que des descriptions d'architectures d'organisations nommées. Leur objectif est pédagogique : montrer comment les patterns s'**assemblent** et quels **compromis** ils impliquent. Chacune indique les sections du chapitre qu'elle mobilise.

---

## Étude de cas 1 — Plateforme SaaS B2B multi-tenant

**Contexte.** Un éditeur de logiciel en mode SaaS sert des centaines d'entreprises clientes : une majorité de petites structures, et quelques grands comptes soumis à des exigences réglementaires fortes (résidence des données, isolation).

**Architecture.** La plateforme adopte un modèle multi-tenant **hybride** (§20.4) : les petits clients partagent un **schéma commun** distingué par un discriminateur (§20.4.3) pour maximiser la densité, tandis que les grands comptes bénéficient d'une **base dédiée**, voire d'une instance dédiée (§20.4.1), pour l'isolation et la conformité. La charge transactionnelle repose sur InnoDB (§20.1), avec un cluster **Galera** pour la haute disponibilité et des **réplicas de lecture** pour absorber la croissance des lectures (§20.7). Pour les clients régulés, certaines instances sont placées dans la région géographique imposée (§20.5).

**Compromis.** Le cœur de la décision est l'arbitrage **isolation vs densité** : le modèle par paliers optimise le coût global tout en répondant aux exigences des plus gros clients. L'isolation des petits locataires reposant sur l'application (pas de *row-level security* native, §20.4.3), un cantonnement centralisé par `tenant_id` est imposé dans la couche d'accès aux données.

> *Sections mobilisées : §20.1, §20.4, §20.5, §20.7.*

---

## Étude de cas 2 — Plateforme e-commerce avec recommandations IA

**Contexte.** Une boutique en ligne doit traiter les commandes et le stock en temps réel, produire des analyses de vente, et recommander des produits pertinents.

**Architecture.** Le **transactionnel** (commandes, stock) s'appuie sur InnoDB (§20.1). L'**analytique** est confiée à un moteur **ColumnStore** (§20.3), alimenté en continu depuis la base transactionnelle par un pipeline de **capture de changements** (§20.8 — CDC, Kafka, Debezium), qui sert aussi à mettre à jour un index de recherche et un cache. Les **recommandations** exploitent la recherche vectorielle (§20.9.2) : on recommande des produits **similaires**, mais la requête est **hybride** (§20.9.4) — « similaires, **en stock** et **dans le budget** ». Des **réplicas de lecture** servent le catalogue à grande échelle (§20.7).

**Compromis.** L'architecture sépare les charges **OLTP et OLAP** (HTAP) pour ne pas pénaliser la production, au prix d'une synchronisation des données entre moteurs. La recommandation étant **intrinsèquement hybride**, héberger embeddings, stock et historique dans la **même base** simplifie radicalement les requêtes par rapport à un magasin vectoriel externe.

> *Sections mobilisées : §20.1, §20.3, §20.7, §20.8, §20.9.*

---

## Étude de cas 3 — Base de connaissances d'entreprise (assistant RAG)

**Contexte.** Une organisation déploie un assistant interne capable de répondre aux questions des collaborateurs à partir de sa documentation privée, dans le strict respect des **droits d'accès** de chacun.

**Architecture.** Les documents sont découpés, encodés en embeddings et stockés dans MariaDB (§20.9). L'assistant met en œuvre un pipeline de **RAG** : la question est encodée, une **recherche sémantique** récupère les fragments pertinents, qui sont injectés dans l'invite d'un LLM. La récupération est **hybride** (§20.9.4) : elle filtre les fragments selon les **permissions** et le **périmètre** de l'utilisateur (§20.4), de sorte qu'aucun contenu non autorisé ne parvienne au modèle. L'intégration s'appuie sur un **framework** (§20.11) côté application, ou sur le **serveur MCP** (§20.10) pour un assistant conversationnel.

**Compromis.** L'enjeu central est le **contrôle d'accès au moment de la récupération** : la recherche hybride permet de conjuguer pertinence sémantique et permissions en une seule requête. Stocker embeddings et métadonnées d'accès dans la **même base** garantit cette cohérence, là où un magasin vectoriel séparé compliquerait l'application des droits.

> *Sections mobilisées : §20.4, §20.9, §20.10, §20.11.*

---

## Étude de cas 4 — Application mondiale multi-région

**Contexte.** Une application sert des utilisateurs répartis sur plusieurs continents, doit respecter des exigences de **résidence des données**, et survivre à la perte d'une région entière.

**Architecture.** La **géo-distribution** (§20.5) repose sur un **cluster Galera par région** (cohérence forte et faible latence en local), les régions étant reliées par une **réplication asynchrone** (§13.11) pour une cohérence à terme. Les lectures sont servies localement et les écritures **partitionnées par région** (chaque région propriétaire de ses données) pour éviter les conflits. L'infrastructure est **multi-cloud** (§20.6) : la portabilité de MariaDB permet de répartir les régions entre fournisseurs, d'assurer une reprise inter-cloud et d'éviter l'enfermement propriétaire. La mise à l'échelle combine montée en puissance des nœuds et réplicas de lecture (§20.7).

**Compromis.** Le compromis fondamental est **latence vs cohérence** : l'asynchrone inter-régions privilégie la latence au prix d'une cohérence à terme, ce qui fixe le **RPO** (§20.5). Les **coûts de sortie de données** entre régions et entre clouds (§20.6) doivent être anticipés, et le partitionnement des écritures par région évite la complexité de la résolution de conflits.

> *Sections mobilisées : §20.5, §20.6, §20.7.*

---

## Les enseignements transversaux

Ces quatre cas, qui touchent ensemble à presque toutes les sections du chapitre, font ressortir plusieurs constantes :

- **Les patterns se combinent.** Aucune architecture réelle ne se réduit à un seul pattern : multi-tenant et haute disponibilité, transactionnel et analytique, vectoriel et relationnel s'assemblent au service d'un besoin.
- **Le compromis est partout.** Isolation contre densité, cohérence contre latence, simplicité contre capacité, indépendance contre coût : chaque décision tranche un arbitrage. C'est le fil rouge du chapitre tout entier.
- **Partir du besoin, non de la technologie.** La bonne démarche consiste à caractériser la charge (transactionnelle ou analytique), les exigences d'isolation, de latence, de conformité et de mise à l'échelle, **avant** de choisir les mécanismes — et non l'inverse.
- **La polyvalence de MariaDB.** Un même moteur couvre l'OLTP (InnoDB), l'OLAP (ColumnStore), le vectoriel (HNSW), la distribution (Galera, réplication, Spider) et l'intégration à l'IA (MCP, frameworks). Cette polyvalence permet de répondre à des besoins variés **sans multiplier les systèmes**, ce qui réduit la complexité, le coût et les problèmes de cohérence.

---

## Conclusion du chapitre

Ce chapitre a opéré un changement de perspective. Les chapitres précédents présentaient MariaDB **brique par brique** — SQL, index, transactions, moteurs, sécurité, administration, réplication, haute disponibilité, performance, DevOps, intégration, fonctionnalités avancées. Le chapitre 20 a montré comment **assembler** ces briques pour répondre à des problèmes concrets : cloisonner des services et des clients, distribuer et mettre à l'échelle, intégrer des flux d'événements, et bâtir des applications d'intelligence artificielle.

Le rôle de l'architecte de données ne consiste pas à connaître une « bonne » architecture universelle — il n'en existe pas — mais à **arbitrer des compromis** en fonction d'un contexte. C'est cette grille de lecture, plus que toute recette, que ce chapitre a cherché à transmettre. MariaDB, par sa maturité et sa polyvalence, offre un socle capable d'accompagner la plupart de ces choix, du système transactionnel le plus classique jusqu'aux architectures d'IA les plus récentes.

Au terme de cette formation, le lecteur dispose ainsi non seulement de la maîtrise des fonctionnalités de MariaDB, mais aussi de la capacité à les **mettre au service d'une architecture** — la compétence qui distingue l'usage de l'outil de la conception de systèmes.

---

## Navigation

⬅️ Section précédente : [20.11 Intégrations frameworks IA (LangChain, LlamaIndex, etc.)](11-integrations-frameworks-ia.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

*Fin du chapitre 20 — Cas d'Usage et Architectures.*

⏭️ [Glossaire des Termes Techniques](/annexes/a-glossaire/README.md)

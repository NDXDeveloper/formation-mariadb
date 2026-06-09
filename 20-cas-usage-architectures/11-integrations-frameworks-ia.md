🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.11 Intégrations frameworks IA (LangChain, LlamaIndex, etc.)

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La section §20.10 a présenté le serveur MCP, qui expose MariaDB aux **assistants et agents** d'IA. Cette section aborde la voie **complémentaire** : intégrer MariaDB **dans le code** des applications d'IA, à l'aide des **frameworks** dédiés tels que **LangChain** et **LlamaIndex**.

Là où le serveur MCP s'adresse à un agent conversationnel, les frameworks s'adressent au **développeur** qui construit une application de RAG ou d'IA générative. Dans les deux cas, MariaDB joue le même rôle de fond — **magasin de vecteurs** — mais l'accès se fait par une API de programmation plutôt que par un protocole d'agent.

---

## Les frameworks d'IA et l'abstraction « vector store »

**LangChain** et **LlamaIndex** comptent parmi les frameworks de référence pour bâtir des applications fondées sur les LLM, en particulier des pipelines de **RAG**. Ils fournissent des composants prêts à l'emploi pour l'ingestion de documents, la gestion des invites, les agents et — central ici — la **recherche par similarité**.

L'abstraction clé est le **vector store** (magasin de vecteurs) : un composant qui stocke des embeddings et effectue la recherche des plus proches voisins. Les frameworks définissent une **interface commune** de vector store, puis proposent des **intégrations** pour de nombreux moteurs. MariaDB, grâce à son type `VECTOR` et à son index HNSW (§20.9), constitue l'un de ces moteurs.

---

## L'apport des frameworks : abstraire le SQL vectoriel

L'intérêt majeur de ces frameworks est qu'ils **dispensent d'écrire le SQL vectoriel** directement. Là où le §20.9 manipulait `VEC_DISTANCE_COSINE`, `VEC_FromText` et `VECTOR INDEX`, le développeur se contente ici d'appeler l'**API de plus haut niveau** du framework : on ajoute des documents, on lance une recherche, et le framework se charge de générer les embeddings, de les stocker dans MariaDB et d'exécuter la recherche.

C'est une troisième manière d'exploiter les capacités vectorielles de MariaDB, complémentaire du **SQL brut** (§20.9) et du **serveur MCP** (§20.10) : le SQL pour le contrôle fin, le MCP pour les assistants, les frameworks pour le développement d'applications.

---

## Les intégrations MariaDB

MariaDB met à disposition un ensemble d'intégrations **open source**, couvrant les écosystèmes Python et Java :

- **LlamaIndex** (Python) — *MariaDB Vector Store* ;
- **LangChain** (Python) — paquet `langchain-mariadb` ;
- **Spring AI** (Java) — *MariaDB Vector Store* ;
- **LangChain4j** (Java) — *MariaDB Embedding Store*.

Ces intégrations s'appuient sur les capacités vectorielles de MariaDB (préversion dès la 11.6, **GA en 11.8 LTS**, donc pleinement disponibles en 12.3). Dans tous les cas, MariaDB sert de **magasin de vecteurs** au framework, qui l'utilise comme socle de stockage et de recherche pour ses pipelines.

---

## LlamaIndex avec MariaDB

L'intégration LlamaIndex expose une classe `MariaDBVectorStore`, que l'on instancie à partir des paramètres de connexion et de quelques réglages vectoriels :

```python
from llama_index.vector_stores.mariadb import MariaDBVectorStore

vector_store = MariaDBVectorStore.from_params(
    host="localhost",
    port=3306,
    user="llamaindex",
    password="...",
    database="vectordb",
    table_name="llama_index_vectorstore",
    embed_dim=1536,      # dimension du modèle d'embedding
    default_m=6,         # paramètre de l'index HNSW
    ef_search=20,        # paramètre de l'index HNSW
)
```

On reconnaît, parmi les paramètres, des réglages propres à l'index **HNSW** (`default_m`, `ef_search`), que le framework configure pour le compte du développeur. Ce magasin sert ensuite à construire un index interrogeable (un `VectorStoreIndex`) au sein de LlamaIndex.

---

## LangChain avec MariaDB

Du côté de LangChain, l'intégration `langchain-mariadb` fournit une classe `MariaDBStore`, à laquelle on associe un modèle d'embedding et une chaîne de connexion :

```python
from langchain_mariadb import MariaDBStore
from langchain_openai import OpenAIEmbeddings

url = "mariadb+mariadbconnector://utilisateur:motdepasse@localhost/langchain"

vectorstore = MariaDBStore(
    embeddings=OpenAIEmbeddings(),
    embedding_length=1536,
    datasource=url,
    collection_name="mes_documents",
)

# Recherche par similarité avec filtre sur les métadonnées
resultats = vectorstore.similarity_search(
    "chaton", k=10, filter={"topic": "animals"}
)
```

On ajoute des documents avec `add_documents`, on recherche avec `similarity_search`, et — point notable — on peut **filtrer sur des métadonnées** lors de la recherche.

---

## Filtrage par métadonnées : la recherche hybride au niveau du framework

Ce filtrage par métadonnées est précisément la **recherche hybride** (§20.9.4), exposée au niveau du framework. On combine la similarité vectorielle avec des conditions structurées (`filter={...}`, avec des combinaisons logiques), sans rédiger soi-même la requête SQL mêlant `ORDER BY VEC_DISTANCE` et clauses `WHERE`.

La nature **relationnelle** de MariaDB rend ce filtrage naturel : les métadonnées sont stockées aux côtés des vecteurs, et le framework traduit le filtre en conditions appliquées à la recherche. On retrouve ainsi, derrière une API simple, l'atout hybride détaillé en §20.9.4.

---

## Pourquoi MariaDB comme magasin de vecteurs d'un framework

Au-delà de la commodité, utiliser MariaDB comme vector store présente les mêmes avantages structurels que ceux exposés en §20.9 :

- **Cohérence des données** : embeddings et données relationnelles dans une seule base, sans synchronisation avec un magasin externe ;
- **Charge d'exploitation réduite** : un seul système à administrer, sauvegarder et sécuriser ;
- **Coût souvent inférieur** à celui d'une base vectorielle dédiée externe ;
- **Latence moindre** et **gestion de la sécurité simplifiée** ;
- **Performance** : les capacités vectorielles de MariaDB sont conçues pour être efficaces, comme en témoignent les bancs d'essai ANN.

L'application bénéficie ainsi de la simplicité d'une API de framework **et** de la robustesse d'un SGBD relationnel mature, sans multiplier les systèmes.

---

## MCP et frameworks : deux voies complémentaires

Il est utile de distinguer clairement les deux modes d'intégration à l'IA présentés dans ce chapitre :

- le **serveur MCP** (§20.10) **expose** MariaDB à des **assistants et agents** d'IA, via un protocole standardisé, pour des interactions conversationnelles ou agentiques ;
- les **frameworks** (§20.11) **intègrent** MariaDB **dans le code** d'une application d'IA construite par un développeur.

Ces deux voies ne s'excluent pas : on choisit l'une, l'autre, ou les deux, selon que l'on **connecte un agent** ou que l'on **développe une application**. Ensemble, elles font de MariaDB un participant à part entière de l'écosystème d'IA.

---

## Synthèse

Les frameworks tels que LangChain, LlamaIndex, Spring AI ou LangChain4j permettent au développeur d'employer MariaDB comme **magasin de vecteurs** dans ses applications de RAG et d'IA, via une **API de haut niveau** qui abstrait le SQL vectoriel — y compris le **filtrage par métadonnées**, qui est la recherche hybride (§20.9.4) sous une forme programmatique. Combinés au SQL vectoriel (§20.9) et au serveur MCP (§20.10), ils offrent un éventail complet de voies d'intégration, toutes adossées aux mêmes atouts : une seule base, cohérence transactionnelle et maturité opérationnelle.

Avec cette section s'achève le panorama des cas d'usage d'IA. Le chapitre se conclut en confrontant l'ensemble des concepts étudiés — du cloisonnement à l'IA — à des situations concrètes, à travers des **études de cas réels** (§20.12).

---

## Navigation

⬅️ Section précédente : [20.10 MariaDB MCP Server pour intégration IA](10-mcp-server-integration-ia.md)  
➡️ Section suivante : [20.12 Études de cas réels](12-etudes-cas-reels.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Études de cas réels](/20-cas-usage-architectures/12-etudes-cas-reels.md)

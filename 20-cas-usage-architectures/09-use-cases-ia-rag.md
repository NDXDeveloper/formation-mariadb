🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.9 Use cases IA : RAG et recherche vectorielle

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Après le cloisonnement, la distribution, la mise à l'échelle et l'intégration événementielle, le chapitre aborde un domaine d'usage en pleine expansion, où MariaDB joue un rôle nouveau : l'**intelligence artificielle**. Grâce au type de données `VECTOR` et à l'index **HNSW** — introduits en préversion dès la 11.6 et passés en GA avec la 11.8 LTS, donc présents en 12.3 (chapitre 18, §18.10) —, MariaDB n'est plus seulement une base relationnelle : elle devient capable de stocker des **embeddings** et d'effectuer de la **recherche vectorielle**, deux fondations des applications d'IA modernes.

Cette section pose les concepts — embeddings, recherche par similarité, RAG — et explique pourquoi il peut être pertinent de réaliser ces traitements **dans** MariaDB plutôt que dans une base de données vectorielle dédiée. Les sous-sections suivantes en déclinent les cas d'usage concrets.

---

## Les fondations : embeddings et vecteurs

Un **embedding** est une représentation **numérique** d'une donnée — un texte, une image, un son — sous la forme d'un **vecteur** de nombres qui en capture le **sens**. Ces vecteurs sont produits par des **modèles d'apprentissage** spécialisés (modèles d'embedding) : deux contenus sémantiquement proches sont transformés en deux vecteurs **proches** dans l'espace vectoriel.

La conséquence est fondamentale : la **distance** entre deux vecteurs (cosinus, euclidienne) mesure leur **similarité sémantique**. On passe ainsi d'une recherche par **mots-clés** (correspondance littérale) à une recherche par **sens** (proximité de vecteurs). Une requête sur « véhicule électrique » peut alors retrouver un document parlant de « voiture à batterie », sans partage de mot-clé.

---

## La recherche vectorielle : les plus proches voisins

La **recherche vectorielle** consiste, à partir d'un vecteur de requête, à retrouver les vecteurs stockés qui en sont les **plus proches** — les *k* plus proches voisins (*k-NN*).

À grande échelle, comparer la requête à **tous** les vecteurs stockés serait trop coûteux. On recourt donc à des algorithmes de **plus proches voisins approchés** (*ANN*, *Approximate Nearest Neighbors*), qui sacrifient une exactitude marginale pour une rapidité considérable. MariaDB s'appuie pour cela sur l'index **HNSW** (*Hierarchical Navigable Small Worlds*, §18.10.2), qui structure les vecteurs en un graphe navigable permettant de retrouver les voisins en temps quasi logarithmique.

---

## MariaDB Vector : le socle technique

MariaDB fournit l'ensemble des briques nécessaires (chapitre 18, §18.10) :

- le **type `VECTOR(n)`** (§18.10.1) pour stocker des embeddings de dimension *n* ;
- l'**index vectoriel HNSW** (§18.10.2) pour la recherche rapide des plus proches voisins ;
- les **fonctions de distance** `VEC_DISTANCE_EUCLIDEAN` et `VEC_DISTANCE_COSINE` (§18.10.3) ;
- les **fonctions de conversion** `VEC_FromText` / `VEC_ToText` (§18.10.4) ;
- des **optimisations SIMD** (§18.10.5) et, en 12.3, des optimisations supplémentaires — calcul de distance par extrapolation et calcul au niveau du moteur de stockage (§18.10.7, §15.16).

Concrètement, on stocke les vecteurs dans une colonne `VECTOR`, indexée en HNSW, et l'on interroge par similarité en SQL :

```sql
-- Documents et leur embedding, avec index vectoriel HNSW
CREATE TABLE documents (
    doc_id    INT PRIMARY KEY,
    contenu   TEXT,
    embedding VECTOR(1536) NOT NULL,   -- dimension du modèle d'embedding
    VECTOR INDEX (embedding)
) ENGINE=InnoDB;
```

```sql
-- Les 5 documents les plus proches d'un vecteur de requête
SELECT doc_id, contenu
FROM documents
ORDER BY VEC_DISTANCE_COSINE(embedding, VEC_FromText('[0.12, 0.04, ...]'))
LIMIT 5;
```

> **Répartition des rôles.** MariaDB **stocke et recherche** les vecteurs, mais ne les **génère** pas : les embeddings sont produits par un modèle externe (service d'embedding, modèle local) que l'application appelle, puis insère dans MariaDB. L'intégration avec ces modèles et les LLM est traitée au §18.10.6.

---

## RAG (Retrieval-Augmented Generation)

Le cas d'usage phare de la recherche vectorielle est le **RAG** (*Retrieval-Augmented Generation*). Il répond à une limite intrinsèque des grands modèles de langage (LLM) : leur connaissance s'arrête à une date, ils peuvent **inventer** des réponses (*hallucinations*), et ils ignorent les données **privées** d'une organisation.

Le RAG consiste à **récupérer un contexte pertinent** dans une base de connaissances, puis à l'**injecter** dans l'invite du LLM, afin que celui-ci réponde en s'**appuyant sur des faits** plutôt que sur sa seule mémoire. Le pipeline se déroule en quatre temps :

1. **Ingestion** : les documents sont découpés en fragments (*chunks*), chaque fragment est transformé en embedding, puis **stocké dans MariaDB** (colonne `VECTOR`).
2. **Recherche** : la question de l'utilisateur est elle-même convertie en embedding, puis une **recherche vectorielle** dans MariaDB renvoie les *k* fragments les plus pertinents.
3. **Augmentation** : ces fragments sont **insérés dans l'invite** envoyée au LLM.
4. **Génération** : le LLM produit une réponse **ancrée** dans le contexte récupéré.

Dans cette chaîne, MariaDB tient le rôle de **magasin de vecteurs** et de **moteur de récupération**. Les bénéfices sont des réponses **fondées et à jour** (y compris sur des données privées), une **réduction des hallucinations**, et la possibilité de **citer** les sources.

---

## Pourquoi MariaDB pour l'IA

De nombreuses applications d'IA recourent à une **base vectorielle dédiée**. Réaliser la recherche vectorielle dans MariaDB présente pourtant des avantages architecturaux décisifs :

- **Une seule base de données** : les embeddings cohabitent avec les **données relationnelles** qu'ils décrivent ; pas de base vectorielle séparée à exploiter, sécuriser et synchroniser.
- **Requêtes hybrides** : on combine, dans **une même requête SQL**, la similarité vectorielle et des **filtres relationnels** classiques (`WHERE`). On peut ainsi demander « les produits sémantiquement proches de X, **en stock** et **à moins de 50 €** » — ce que les bases vectorielles pures peinent à faire. C'est l'objet du §20.9.4.
- **Cohérence transactionnelle** : les vecteurs restent **synchronisés** avec les données qu'ils accompagnent, sans le problème de double écriture (§20.8) entre une base relationnelle et un magasin vectoriel externe.
- **Maturité opérationnelle** : sécurité, sauvegarde, haute disponibilité, supervision — tout l'outillage éprouvé de MariaDB s'applique aux vecteurs.
- **Pas de déplacement de données** : on évite la complexité et le coût de la synchronisation vers un magasin externe.

En d'autres termes, MariaDB permet d'ajouter des capacités d'IA **là où vivent déjà les données**, sans multiplier les systèmes.

---

## Ce que couvrent les sections suivantes

Les sous-sections déclinent quatre familles d'usage de la recherche vectorielle :

- **§20.9.1 — Semantic Search** : la recherche par le sens plutôt que par mots-clés.
- **§20.9.2 — Recommendation Engines** : la recommandation d'éléments similaires par proximité de vecteurs.
- **§20.9.3 — Anomaly Detection** : la détection d'anomalies comme points éloignés des regroupements habituels.
- **§20.9.4 — Hybrid Search** : la combinaison de la recherche vectorielle et des filtres SQL relationnels — l'atout distinctif de MariaDB.

Au-delà, le chapitre abordera l'intégration de MariaDB à l'écosystème IA via le **serveur MCP** (§20.10) et les **frameworks d'IA** tels que LangChain ou LlamaIndex (§20.11). La section suivante (§20.9.1) commence par l'application la plus directe de la recherche vectorielle : la recherche sémantique.

---

## Navigation

⬅️ Section précédente : [20.8.3 Debezium connector](08.3-debezium.md)  
➡️ Section suivante : [20.9.1 Semantic Search](09.1-semantic-search.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Semantic Search](/20-cas-usage-architectures/09.1-semantic-search.md)

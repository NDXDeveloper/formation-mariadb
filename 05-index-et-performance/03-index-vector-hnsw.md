🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 Index VECTOR (HNSW) pour la recherche vectorielle 🆕

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 3-4 heures

> **Prérequis** :
> - Section 5.1 - Fonctionnement des index B-Tree
> - Notions de machine learning et embeddings (recommandé)
> - Compréhension des vecteurs et distances mathématiques
> - Familiarité avec les concepts d'IA générative (optionnel)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le type de données VECTOR et son utilité pour l'IA/ML
- Maîtriser l'algorithme HNSW (Hierarchical Navigable Small Worlds)
- Créer et configurer des index VECTOR avec les bons paramètres
- Utiliser les fonctions de distance vectorielle (cosine, euclidean, dot product)
- Intégrer MariaDB avec des modèles d'IA (OpenAI, LangChain, LlamaIndex)
- Implémenter des cas d'usage IA : semantic search, RAG, recommandations
- Optimiser les performances avec les paramètres HNSW et SIMD
- Comparer MariaDB Vector avec les alternatives (pgvector, Pinecone, Weaviate)

---

## Introduction : L'IA rencontre les bases de données

### La révolution des embeddings

L'émergence des **Large Language Models** (LLM) comme GPT-4, Claude, LLaMA a créé un besoin crucial : stocker et rechercher efficacement des **embeddings vectoriels**.

```
Qu'est-ce qu'un embedding ?

Texte original : "MariaDB est une base de données relationnelle"
                          ↓
        Modèle d'IA (ex: OpenAI text-embedding-3-small)
                          ↓
Vecteur de 1536 dimensions : [0.023, -0.154, 0.891, ..., 0.432]

Ce vecteur capture le SENS SÉMANTIQUE du texte
→ Textes similaires ont des vecteurs proches
→ Permet la recherche par similarité sémantique
```

### Pourquoi MariaDB Vector ? 🆕

**MariaDB 11.8 LTS** (juin 2025) introduit le support natif des vecteurs, permettant :

```sql
-- Avant 11.8 : Stocker dans JSON (inefficace)
CREATE TABLE docs (
    id INT PRIMARY KEY,
    embedding JSON  -- [0.1, 0.2, ..., 0.9]
);
-- Problèmes :
-- ❌ Pas d'index optimisé
-- ❌ Calculs de distance lents
-- ❌ Pas d'optimisations SIMD

-- Avec 11.8 : Type VECTOR natif + index HNSW
CREATE TABLE docs (
    id INT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536) NOT NULL  -- Type natif !
) ENGINE=InnoDB;

CREATE INDEX idx_embedding ON docs(embedding)
USING HNSW
WITH (
    m = 16,
    ef_construction = 200,
    metric = 'cosine'
);
-- Avantages :
-- ✅ Index HNSW ultra-rapide
-- ✅ Optimisations SIMD (AVX2/AVX512)
-- ✅ Recherche en O(log n)
-- ✅ Intégration native avec SQL
```

### Applications de MariaDB Vector

| Cas d'usage | Description | Impact business |
|-------------|-------------|-----------------|
| 🔍 **Semantic Search** | Recherche par sens, pas par mots-clés | Meilleure UX recherche |
| 🤖 **Chatbots RAG** | Retrieval-Augmented Generation | Réponses contextuelles |
| 🎯 **Recommandations** | Produits/contenus similaires | ↑ Conversions |
| 🔒 **Détection d'anomalies** | Fraude, intrusions | ↓ Risques |
| 🎨 **Recherche d'images** | Par contenu visuel | UX innovante |
| 📄 **Duplicate detection** | Documents/tickets similaires | Efficacité |

---

## Type de données VECTOR

### Déclaration et caractéristiques

```sql
-- Syntaxe
VECTOR(dimensions)

-- Exemples selon les modèles d'IA
embedding VECTOR(384)    -- Sentence-BERT (sentence-transformers)
embedding VECTOR(768)    -- BERT-base
embedding VECTOR(1536)   -- OpenAI text-embedding-3-small
embedding VECTOR(3072)   -- OpenAI text-embedding-3-large
embedding VECTOR(1024)   -- Cohere embed-english-v3.0
embedding VECTOR(4096)   -- Voyage AI voyage-large-2

-- Contraintes
-- Minimum : 1 dimension
-- Maximum : 16 000 dimensions
-- Type : FLOAT (4 bytes par dimension)
```

**Calcul de la taille** :

```
Taille en mémoire = dimensions × 4 bytes

Exemples :
- VECTOR(384)  : 384 × 4 = 1,536 bytes  (~1.5 KB)
- VECTOR(1536) : 1536 × 4 = 6,144 bytes (~6 KB)
- VECTOR(3072) : 3072 × 4 = 12,288 bytes (~12 KB)

Pour 1 million de vecteurs (1536 dimensions) :
→ ~6 GB minimum en mémoire/disque
```

### Création de tables avec VECTOR

```sql
-- Table pour documents avec embeddings
CREATE TABLE documents (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    category VARCHAR(100),
    embedding VECTOR(1536) NOT NULL,  -- OpenAI embeddings
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_category (category),
    INDEX idx_embedding (embedding)
        USING HNSW
        WITH (m=16, ef_construction=200, metric='cosine')
) ENGINE=InnoDB;

-- Table pour produits avec features vectoriels
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    price DECIMAL(10,2),
    features_vector VECTOR(512) NOT NULL,  -- Caractéristiques produit

    INDEX idx_features (features_vector)
        USING HNSW
        WITH (m=24, ef_construction=300, metric='dot')
) ENGINE=InnoDB;

-- Table pour images avec embeddings visuels
CREATE TABLE images (
    image_id INT PRIMARY KEY,
    filename VARCHAR(255),
    visual_embedding VECTOR(2048) NOT NULL,  -- CLIP, ResNet embeddings

    INDEX idx_visual (visual_embedding)
        USING HNSW
        WITH (m=32, ef_construction=400, metric='euclidean')
) ENGINE=InnoDB;
```

### Insertion de vecteurs

```sql
-- Méthode 1 : VEC_FromText (depuis texte)
INSERT INTO documents (title, content, embedding) VALUES
('MariaDB Performance', 'Guide to optimize...',
 VEC_FromText('[0.023, -0.154, 0.891, ..., 0.432]'));  -- 1536 valeurs

-- Méthode 2 : Depuis application (Python)
-- L'embedding est généré côté application puis inséré
import openai
import mariadb

# Générer embedding avec OpenAI
response = openai.Embedding.create(
    model="text-embedding-3-small",
    input="MariaDB Performance Guide"
)
embedding = response['data'][0]['embedding']  # Liste de 1536 floats

# Convertir en format MariaDB
embedding_str = '[' + ','.join(map(str, embedding)) + ']'

# Insérer
cursor.execute("""
    INSERT INTO documents (title, content, embedding)
    VALUES (%s, %s, VEC_FromText(%s))
""", (title, content, embedding_str))
```

---

## Algorithme HNSW : Le cœur de la performance

### Hierarchical Navigable Small Worlds

**HNSW** est un algorithme de graphe pour la recherche approximative des plus proches voisins (Approximate Nearest Neighbor - ANN).

### Structure multi-niveaux

```
Architecture HNSW :

Niveau 3 (top, sparse)        [V42] ←─────→ [V1337]
                                ↓              ↓
Niveau 2                      [V42]─→[V108]─→[V1337]←─[V2001]
                                ↓      ↓       ↓         ↓
Niveau 1                 [V7]→[V42]─→[V108]─→[V1337]←─[V2001]→[V3456]
                          ↓     ↓      ↓       ↓         ↓        ↓
Niveau 0 (base, dense)   [V1][V7][V23][V42][V87][V108]...[V3456]...[VN]
                          ↔    ↔   ↔    ↔    ↔    ↔         ↔        ↔

Chaque nœud a ~m connexions vers ses voisins les plus proches
Plus on monte dans les niveaux, moins il y a de nœuds
```

**Principes clés** :

1. **Hiérarchique** : Plusieurs niveaux de graphe
2. **Navigable** : Connexions vers voisins proches
3. **Small Worlds** : Peu de sauts pour atteindre n'importe quel nœud

### Algorithme de recherche

```
Recherche des k plus proches voisins d'un vecteur query :

1. Commencer au niveau le plus haut (sparse)
2. Trouver le nœud le plus proche au niveau courant
3. Descendre au niveau inférieur
4. Répéter jusqu'au niveau 0 (base)
5. Au niveau 0 : explorer le voisinage local
6. Retourner les k vecteurs les plus proches

Complexité : O(log n) en moyenne
Comparé à recherche exhaustive O(n)

Exemple pour 1 million de vecteurs :
- Recherche exhaustive : 1 000 000 comparaisons
- HNSW : ~20-50 comparaisons (20 000x plus rapide !)
```

### Paramètres de configuration HNSW

#### 1. m (nombre de connexions par nœud)

```sql
-- m : Nombre de connexions bidirectionnelles par nœud

CREATE INDEX idx_low_m ON table(embedding)
USING HNSW WITH (m=8, metric='cosine');
-- Avantages : Construction rapide, moins de mémoire
-- Inconvénients : Précision moindre

CREATE INDEX idx_default_m ON table(embedding)
USING HNSW WITH (m=16, metric='cosine');
-- Équilibre optimal pour la plupart des cas

CREATE INDEX idx_high_m ON table(embedding)
USING HNSW WITH (m=48, metric='cosine');
-- Avantages : Meilleure précision (recall)
-- Inconvénients : Construction lente, plus de mémoire
```

**Recommandations** :

| m | Usage | Recall | Construction | Mémoire |
|---|-------|--------|--------------|---------|
| 8-12 | Prototypage rapide | ~85-90% | Rapide | Faible |
| 16-24 | Production standard | ~92-96% | Moyen | Moyen |
| 32-48 | Haute précision | ~97-99% | Lent | Élevé |

#### 2. ef_construction (qualité de construction)

```sql
-- ef_construction : Taille de la liste de candidats lors de la construction

CREATE INDEX idx_low_ef ON table(embedding)
USING HNSW WITH (m=16, ef_construction=100);
-- Construction rapide, qualité d'index moindre

CREATE INDEX idx_default_ef ON table(embedding)
USING HNSW WITH (m=16, ef_construction=200);
-- Équilibre optimal

CREATE INDEX idx_high_ef ON table(embedding)
USING HNSW WITH (m=16, ef_construction=500);
-- Construction lente, index de haute qualité
```

**Règle générale** : `ef_construction >= 2 × m`

#### 3. ef_search (qualité de recherche)

```sql
-- ef_search : Taille de la liste lors de la recherche (runtime)
-- Peut être ajusté dynamiquement par session

-- Recherche rapide (moins précise)
SET @@session.hnsw_ef_search = 50;

-- Recherche standard
SET @@session.hnsw_ef_search = 100;  -- Défaut

-- Recherche précise (plus lente)
SET @@session.hnsw_ef_search = 200;

-- Recherche ultra-précise
SET @@session.hnsw_ef_search = 500;

-- Impact sur une requête
SET @@session.hnsw_ef_search = 200;
SELECT id, title,
       VEC_DISTANCE_COSINE(embedding, @query_vector) AS similarity
FROM documents
ORDER BY similarity ASC
LIMIT 10;
```

**Trade-off ef_search** :

```
ef_search = 50  : 5 ms,   recall ~88%
ef_search = 100 : 10 ms,  recall ~94%  ← Défaut
ef_search = 200 : 20 ms,  recall ~97%
ef_search = 500 : 50 ms,  recall ~99%
```

#### 4. metric (métrique de distance)

```sql
-- Cosine : Angle entre vecteurs (texte, sémantique)
CREATE INDEX idx_cosine ON docs(embedding)
USING HNSW WITH (m=16, ef_construction=200, metric='cosine');

-- Euclidean : Distance géométrique classique
CREATE INDEX idx_euclidean ON docs(embedding)
USING HNSW WITH (m=16, ef_construction=200, metric='euclidean');

-- Dot Product : Produit scalaire (recommandations)
CREATE INDEX idx_dot ON docs(embedding)
USING HNSW WITH (m=16, ef_construction=200, metric='dot');
```

---

## Métriques de distance vectorielle

### 1. Cosine Similarity (Distance cosinus)

Mesure l'**angle** entre deux vecteurs (indépendant de la magnitude).

```sql
-- VEC_DISTANCE_COSINE : Retourne 1 - cosine_similarity
-- Résultat entre 0 (identiques) et 2 (opposés)
SELECT
    id,
    title,
    VEC_DISTANCE_COSINE(embedding, @query_vector) AS distance
FROM documents
ORDER BY distance ASC  -- Plus petit = plus similaire
LIMIT 10;

-- Formule mathématique :
-- cosine_distance = 1 - (A · B) / (||A|| × ||B||)
-- Où A · B = produit scalaire
-- ||A|| = norme euclidienne de A
```

**Quand utiliser Cosine ?**

✅ **Excellent pour** :
- Texte et embeddings sémantiques
- Quand la direction importe plus que la magnitude
- OpenAI, Cohere, Sentence-BERT embeddings
- Recherche de documents similaires

```sql
-- Exemple : Recherche sémantique
-- "database performance" devrait trouver "DB optimization"
SET @query = VEC_FromText('[0.12, -0.45, 0.89, ...]');  -- Embedding de "database performance"

SELECT title, content,
       VEC_DISTANCE_COSINE(embedding, @query) AS similarity
FROM articles
WHERE VEC_DISTANCE_COSINE(embedding, @query) < 0.3  -- Seuil de similarité
ORDER BY similarity ASC
LIMIT 5;
```

### 2. Euclidean Distance (Distance euclidienne)

Mesure la **distance géométrique directe** entre deux points.

```sql
-- VEC_DISTANCE_EUCLIDEAN : Distance L2
SELECT
    id,
    name,
    VEC_DISTANCE_EUCLIDEAN(features, @query_vector) AS distance
FROM products
ORDER BY distance ASC
LIMIT 10;

-- Formule mathématique :
-- euclidean_distance = √(Σ(Ai - Bi)²)
```

**Quand utiliser Euclidean ?**

✅ **Excellent pour** :
- Vecteurs de caractéristiques normalisées
- Embeddings d'images (CLIP, ResNet)
- Quand la magnitude a du sens
- Détection d'anomalies

```sql
-- Exemple : Détection d'anomalies dans transactions
-- Transactions avec profil comportemental inhabituel
SELECT
    transaction_id,
    amount,
    VEC_DISTANCE_EUCLIDEAN(behavior_vector, @normal_profile) AS anomaly_score
FROM transactions
WHERE transaction_date = CURRENT_DATE
AND VEC_DISTANCE_EUCLIDEAN(behavior_vector, @normal_profile) > 2.0  -- Seuil
ORDER BY anomaly_score DESC;
```

### 3. Dot Product (Produit scalaire)

Mesure la **projection** d'un vecteur sur un autre.

```sql
-- VEC_DISTANCE_DOT : -1 × (A · B)
-- Résultat : plus négatif = plus similaire
SELECT
    product_id,
    name,
    VEC_DISTANCE_DOT(features, @user_preferences) AS score
FROM products
ORDER BY score ASC  -- Plus négatif = meilleure correspondance
LIMIT 20;

-- Formule mathématique :
-- dot_product = Σ(Ai × Bi)
-- VEC_DISTANCE_DOT = -dot_product (pour cohérence ASC)
```

**Quand utiliser Dot Product ?**

✅ **Excellent pour** :
- Systèmes de recommandation
- Scoring de pertinence
- Quand les vecteurs sont déjà normalisés
- Collaborative filtering

```sql
-- Exemple : Recommandations produits
-- Profil utilisateur × caractéristiques produits
SET @user_profile = VEC_FromText('[0.8, 0.2, -0.5, ...]');  -- Préférences utilisateur

SELECT
    p.product_id,
    p.name,
    p.price,
    -VEC_DISTANCE_DOT(p.features, @user_profile) AS recommendation_score
FROM products p
WHERE p.in_stock = TRUE
AND p.price BETWEEN 50 AND 500
ORDER BY recommendation_score DESC
LIMIT 10;
```

### Comparaison des métriques

| Métrique | Formule | Range | Normalisation | Usage principal |
|----------|---------|-------|---------------|-----------------|
| **Cosine** | 1 - (A·B)/(‖A‖‖B‖) | [0, 2] | Automatique | Texte, sémantique |
| **Euclidean** | √Σ(Ai-Bi)² | [0, ∞] | Recommandée | Images, features |
| **Dot** | -(A·B) | [-∞, ∞] | Requise | Recommandations |

---

## Fonctions vectorielles

### Création et conversion

```sql
-- VEC_FromText : Texte → VECTOR
SET @vec1 = VEC_FromText('[0.1, 0.2, 0.3]');

-- VEC_ToText : VECTOR → Texte
SELECT VEC_ToText(embedding) FROM documents WHERE id = 1;
-- Résultat : '[0.023, -0.154, 0.891, ...]'

-- VEC_Dimension : Obtenir le nombre de dimensions
SELECT VEC_Dimension(embedding) FROM documents WHERE id = 1;
-- Résultat : 1536
```

### Distances

```sql
-- Les trois fonctions de distance
SELECT
    VEC_DISTANCE_COSINE(v1, v2) AS cosine_dist,
    VEC_DISTANCE_EUCLIDEAN(v1, v2) AS euclidean_dist,
    VEC_DISTANCE_DOT(v1, v2) AS dot_dist
FROM vectors;
```

### Opérations vectorielles

```sql
-- Normalisation L2 (pour dot product)
-- Côté application avant insertion
def normalize_vector(vec):
    norm = np.linalg.norm(vec)
    return vec / norm if norm > 0 else vec

-- Vérification de validité
-- MariaDB valide automatiquement :
-- - Nombre de dimensions correct
-- - Valeurs numériques valides
-- - Pas de NaN ou Inf
```

---

## Optimisations SIMD 🆕

### Accélération matérielle

MariaDB 11.8 exploite les **instructions SIMD** (Single Instruction Multiple Data) pour paralléliser les calculs vectoriels au niveau CPU.

```
Support SIMD par processeur :

Intel/AMD x86-64 :
- SSE4.2 : Base (2008+)
- AVX2 : 4-8x plus rapide (2013+)
- AVX-512 : 8-16x plus rapide (2017+)

ARM :
- NEON : Standard ARM (Raspberry Pi, mobiles, AWS Graviton)
- SVE : ARM nouvelle génération

IBM Power :
- VSX (Power10+) : Serveurs haute performance
```

### Vérifier le support SIMD

```sql
-- Variables système SIMD
SHOW VARIABLES LIKE 'have_vector_%';

-- Exemples de sortie :
-- have_vector_avx2   : ON   ← AVX2 disponible et activé
-- have_vector_avx512 : ON   ← AVX-512 disponible
-- have_vector_neon   : OFF  ← Pas ARM
```

### Impact sur les performances

**Benchmarks comparatifs** (1 million de vecteurs, 1536 dimensions) :

| Opération | Sans SIMD | AVX2 | AVX-512 | Speedup |
|-----------|-----------|------|---------|---------|
| Cosine distance | 250 ms | 35 ms | 18 ms | 14x |
| Euclidean distance | 180 ms | 28 ms | 15 ms | 12x |
| Dot product | 120 ms | 18 ms | 10 ms | 12x |

```sql
-- Désactiver SIMD pour benchmark (non recommandé en production)
SET GLOBAL vector_use_simd = OFF;

-- Réactiver
SET GLOBAL vector_use_simd = ON;  -- Défaut
```

💡 **En production** : Laissez SIMD activé. MariaDB choisit automatiquement la meilleure instruction selon le CPU.

---

## Cas d'usage IA : Implémentations détaillées

### 1. Semantic Search (Recherche sémantique) 🔍

Rechercher par **sens** plutôt que par mots-clés exacts.

```sql
-- Structure de la base de connaissances
CREATE TABLE knowledge_base (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    category VARCHAR(100),
    url VARCHAR(500),
    embedding VECTOR(1536) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_category (category),
    FULLTEXT INDEX idx_content (title, content),
    INDEX idx_embedding (embedding)
        USING HNSW
        WITH (m=16, ef_construction=200, metric='cosine')
) ENGINE=InnoDB;

-- Requête de recherche sémantique
-- L'embedding de la requête est généré côté application
SET @query_embedding = VEC_FromText('[...]');  -- Généré par OpenAI

SELECT
    id,
    title,
    category,
    SUBSTRING(content, 1, 200) AS excerpt,
    VEC_DISTANCE_COSINE(embedding, @query_embedding) AS similarity_score
FROM knowledge_base
WHERE VEC_DISTANCE_COSINE(embedding, @query_embedding) < 0.4  -- Seuil de pertinence
ORDER BY similarity_score ASC
LIMIT 10;
```

**Workflow complet (Python + OpenAI)** :

```python
import openai
import mariadb

# 1. Générer embedding de la requête utilisateur
query = "How do I optimize MariaDB indexes?"
response = openai.Embedding.create(
    model="text-embedding-3-small",
    input=query
)
query_embedding = response['data'][0]['embedding']

# 2. Recherche dans MariaDB
embedding_str = '[' + ','.join(map(str, query_embedding)) + ']'
cursor.execute("""
    SELECT
        id, title, content,
        VEC_DISTANCE_COSINE(embedding, VEC_FromText(%s)) AS score
    FROM knowledge_base
    WHERE VEC_DISTANCE_COSINE(embedding, VEC_FromText(%s)) < 0.4
    ORDER BY score ASC
    LIMIT 5
""", (embedding_str, embedding_str))

results = cursor.fetchall()

# 3. Afficher les résultats pertinents
for row in results:
    print(f"Score: {row['score']:.3f} - {row['title']}")
    print(f"  {row['content'][:200]}...\n")
```

### 2. RAG (Retrieval-Augmented Generation) 🤖

Enrichir les réponses d'un LLM avec du contexte tiré de votre base de données.

```sql
-- Table pour chunks de documentation
CREATE TABLE doc_chunks (
    chunk_id INT PRIMARY KEY AUTO_INCREMENT,
    document_id INT,
    chunk_index INT,
    chunk_text TEXT,  -- 500-1000 tokens par chunk
    embedding VECTOR(1536) NOT NULL,
    metadata JSON,  -- {source, page, section, ...}

    INDEX idx_document (document_id),
    INDEX idx_embedding (embedding)
        USING HNSW
        WITH (m=20, ef_construction=250, metric='cosine')
) ENGINE=InnoDB;

-- Workflow RAG :
-- 1. User pose une question
-- 2. Générer embedding de la question
-- 3. Trouver les chunks les plus pertinents
-- 4. Envoyer chunks + question au LLM
-- 5. LLM génère une réponse contextualisée

-- Requête pour récupérer le contexte
SET @question_embedding = VEC_FromText('[...]');

SELECT
    chunk_text,
    metadata,
    VEC_DISTANCE_COSINE(embedding, @question_embedding) AS relevance
FROM doc_chunks
ORDER BY relevance ASC
LIMIT 3;  -- Top 3 chunks les plus pertinents

-- Résultats envoyés au LLM avec la question
```

**Implémentation Python avec LangChain** :

```python
from langchain.vectorstores import MariaDBVector
from langchain.embeddings import OpenAIEmbeddings
from langchain.chat_models import ChatOpenAI
from langchain.chains import RetrievalQA

# 1. Configuration
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = MariaDBVector(
    connection_string="mariadb://user:pass@localhost/mydb",
    embedding_function=embeddings,
    table_name="doc_chunks",
    dimension=1536
)

# 2. Créer la chaîne RAG
llm = ChatOpenAI(model="gpt-4", temperature=0)
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3})
)

# 3. Poser une question
question = "How do I configure replication in MariaDB?"
answer = qa_chain.run(question)
print(answer)

# Le système :
# - Génère l'embedding de la question
# - Recherche les 3 chunks les plus pertinents dans MariaDB
# - Envoie chunks + question à GPT-4
# - Retourne une réponse contextualisée
```

### 3. Système de recommandation 🎯

Recommander des produits/contenus similaires basés sur les préférences utilisateur.

```sql
-- Table produits avec features vectoriels
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2),
    features_vector VECTOR(512) NOT NULL,  -- Caractéristiques produit

    INDEX idx_category (category),
    INDEX idx_price (price),
    INDEX idx_features (features_vector)
        USING HNSW
        WITH (m=24, ef_construction=300, metric='dot')
) ENGINE=InnoDB;

-- Table interactions utilisateur
CREATE TABLE user_interactions (
    user_id INT,
    product_id INT,
    interaction_type ENUM('view', 'click', 'purchase'),
    timestamp TIMESTAMP,
    PRIMARY KEY (user_id, product_id, timestamp)
);

-- Calculer le profil utilisateur (côté application)
-- Profil = moyenne pondérée des features des produits achetés/aimés
```

**Recommandations basées sur le profil** :

```sql
-- Produits similaires à ce que l'utilisateur aime
SET @user_profile = VEC_FromText('[...]');  -- Calculé côté application

SELECT
    p.product_id,
    p.name,
    p.price,
    p.category,
    -VEC_DISTANCE_DOT(p.features_vector, @user_profile) AS recommendation_score
FROM products p
WHERE p.product_id NOT IN (
    -- Exclure produits déjà achetés
    SELECT product_id FROM user_interactions
    WHERE user_id = 123 AND interaction_type = 'purchase'
)
AND -VEC_DISTANCE_DOT(p.features_vector, @user_profile) > 0.5  -- Seuil
ORDER BY recommendation_score DESC
LIMIT 20;
```

**Produits similaires (collaborative filtering)** :

```sql
-- "Clients qui ont acheté X ont aussi acheté Y"
SET @product_features = (
    SELECT features_vector FROM products WHERE product_id = 456
);

SELECT
    p.product_id,
    p.name,
    p.price,
    VEC_DISTANCE_COSINE(p.features_vector, @product_features) AS similarity
FROM products p
WHERE p.product_id != 456
ORDER BY similarity ASC
LIMIT 10;
```

### 4. Détection d'anomalies 🔒

Identifier des comportements inhabituels (fraude, intrusions, etc.).

```sql
-- Table transactions avec profil comportemental
CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    user_id INT,
    amount DECIMAL(10,2),
    merchant_category VARCHAR(100),
    location_lat DECIMAL(9,6),
    location_lon DECIMAL(9,6),
    behavior_vector VECTOR(256) NOT NULL,  -- Profil comportemental
    timestamp DATETIME,
    is_flagged BOOLEAN DEFAULT FALSE,

    INDEX idx_user (user_id),
    INDEX idx_timestamp (timestamp),
    INDEX idx_behavior (behavior_vector)
        USING HNSW
        WITH (m=16, ef_construction=200, metric='euclidean')
) ENGINE=InnoDB;

-- Détecter transactions anormales
-- Comparer avec le profil normal de l'utilisateur
SET @user_normal_profile = (
    -- Calculer le profil moyen des 100 dernières transactions normales
    SELECT AVG(behavior_vector)
    FROM transactions
    WHERE user_id = 123
    AND is_flagged = FALSE
    ORDER BY timestamp DESC
    LIMIT 100
);

SELECT
    t.transaction_id,
    t.amount,
    t.merchant_category,
    VEC_DISTANCE_EUCLIDEAN(t.behavior_vector, @user_normal_profile) AS anomaly_score
FROM transactions t
WHERE t.user_id = 123
AND t.timestamp > DATE_SUB(NOW(), INTERVAL 1 HOUR)
AND VEC_DISTANCE_EUCLIDEAN(t.behavior_vector, @user_normal_profile) > 2.5  -- Seuil
ORDER BY anomaly_score DESC;

-- Transactions avec score élevé = potentiellement frauduleuses
-- Déclencher alerte ou demander vérification supplémentaire
```

### 5. Recherche d'images par contenu 🎨

Trouver des images similaires basées sur leur contenu visuel.

```sql
-- Table images avec embeddings visuels (CLIP, ResNet)
CREATE TABLE image_library (
    image_id INT PRIMARY KEY AUTO_INCREMENT,
    filename VARCHAR(255),
    url VARCHAR(500),
    alt_text TEXT,
    visual_embedding VECTOR(512) NOT NULL,  -- CLIP embeddings
    tags VARCHAR(500),

    INDEX idx_embedding (visual_embedding)
        USING HNSW
        WITH (m=32, ef_construction=400, metric='cosine')
) ENGINE=InnoDB;

-- Recherche d'images similaires
-- Embedding de l'image requête généré par CLIP
SET @query_image_embedding = VEC_FromText('[...]');

SELECT
    image_id,
    filename,
    url,
    alt_text,
    VEC_DISTANCE_COSINE(visual_embedding, @query_image_embedding) AS similarity
FROM image_library
WHERE VEC_DISTANCE_COSINE(visual_embedding, @query_image_embedding) < 0.3
ORDER BY similarity ASC
LIMIT 20;

-- Recherche texte → image (CLIP multimodal)
-- CLIP génère des embeddings dans le même espace vectoriel
-- pour le texte ET les images
SET @text_query_embedding = VEC_FromText('[...]');  -- "cat playing with yarn"

SELECT
    image_id,
    filename,
    alt_text,
    VEC_DISTANCE_COSINE(visual_embedding, @text_query_embedding) AS relevance
FROM image_library
ORDER BY relevance ASC
LIMIT 10;
```

---

## Performance et benchmarks

### Métriques de performance

**Test standard** : 1 million de vecteurs, 1536 dimensions

| Opération | Temps | Throughput |
|-----------|-------|------------|
| Insertion (sans index) | 25 min | 667 vec/s |
| Construction index HNSW (m=16) | 45 min | - |
| Recherche top-10 (ef=100) | 5-8 ms | 125-200 queries/s |
| Recherche top-10 (ef=200) | 12-18 ms | 55-80 queries/s |
| Recherche top-100 (ef=200) | 25-35 ms | 28-40 queries/s |

**Configuration test** :
- CPU : Intel Xeon (AVX2)
- RAM : 32 GB
- Disque : NVMe SSD
- MariaDB 11.8, InnoDB Buffer Pool : 16 GB

### Recall vs Performance

**Recall** = % de vrais plus proches voisins trouvés

```
Trade-off m × ef :

Configuration  │ Recall │ Query time │ Index size
───────────────┼────────┼────────────┼───────────
m=8,  ef=50    │  85%   │   3 ms     │   +15%
m=16, ef=100   │  94%   │   7 ms     │   +30%  ← Défaut
m=24, ef=200   │  97%   │  15 ms     │   +45%
m=48, ef=400   │  99%   │  35 ms     │   +90%
```

💡 **Recommandation** : Commencez avec m=16, ef_construction=200. Ajustez si nécessaire après benchmarks.

### Optimiser les performances

**1. Dimensionnement du Buffer Pool**

```sql
-- Allouer suffisamment de RAM pour index + données
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Recommandation : 60-70% de la RAM serveur
SET GLOBAL innodb_buffer_pool_size = 17179869184;  -- 16 GB

-- Vérifier le hit ratio
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
-- Objectif : > 99%
```

**2. Configuration HNSW spécifique**

```sql
-- Augmenter le cache pour construction d'index
SET GLOBAL innodb_vector_cache_size = 2147483648;  -- 2 GB

-- Threads parallèles pour construction
SET GLOBAL innodb_vector_threads = 8;  -- Selon nb de cores
```

**3. Batch insertions**

```python
# Insérer par lots pour meilleures performances
batch_size = 1000
for i in range(0, len(embeddings), batch_size):
    batch = embeddings[i:i+batch_size]
    cursor.executemany("""
        INSERT INTO documents (title, content, embedding)
        VALUES (%s, %s, VEC_FromText(%s))
    """, batch)
    conn.commit()
```

**4. Filtrage hybride**

```sql
-- Combiner index vectoriel + index B-Tree
-- Filtrer d'abord avec B-Tree, puis recherche vectorielle
SELECT
    id, title,
    VEC_DISTANCE_COSINE(embedding, @query_vec) AS score
FROM documents
WHERE category = 'tutorial'  -- Index B-Tree: filtre rapide
AND published = TRUE
AND VEC_DISTANCE_COSINE(embedding, @query_vec) < 0.4
ORDER BY score ASC
LIMIT 10;
```

---

## Comparaison avec les alternatives

### MariaDB Vector vs pgvector (PostgreSQL)

| Aspect | MariaDB Vector (11.8) | pgvector |
|--------|----------------------|----------|
| **Algorithme** | HNSW | IVFFlat, HNSW |
| **SIMD** | AVX2, AVX-512, NEON, Power10 | AVX-512 (partiel) |
| **Dimensions max** | 16 000 | 16 000 |
| **Métriques** | Cosine, Euclidean, Dot | Cosine, Euclidean, Inner |
| **Maturité** | Nouveau (2025) | Plus mature (2021) |
| **Écosystème** | Intégration SQL native | Large écosystème Python |
| **Performance** | Comparable | Comparable |
| **License** | GPL v2 (open-source) | PostgreSQL License |

### MariaDB Vector vs bases vectorielles dédiées

**Pinecone, Weaviate, Qdrant, Milvus**

✅ **Avantages MariaDB** :
- Tout dans une seule base (pas de synchro)
- Transactions ACID
- Requêtes SQL hybrides (vecteurs + relationnel)
- Coût réduit (pas de service supplémentaire)
- Connaissance SQL existante

❌ **Inconvénients MariaDB** :
- Moins de features spécialisées IA
- Scaling horizontal plus complexe
- Pas de features comme filtrage pré/post natif
- Écosystème IA moins riche

**Quand choisir quoi ?**

```
MariaDB Vector :
- Application existante sur MariaDB
- Données relationnelles + vecteurs
- Budget limité
- < 10M vecteurs
- Transactions critiques

Base vectorielle dédiée :
- > 100M vecteurs
- Scaling horizontal nécessaire
- Features IA avancées
- Équipe data science dédiée
```

---

## Configuration en production

### Checklist de déploiement

```sql
-- 1. Vérifier support SIMD
SHOW VARIABLES LIKE 'have_vector_%';

-- 2. Dimensionner le Buffer Pool
-- Formule : (nb_vecteurs × dimensions × 4 bytes × 1.5) + données
-- Pour 1M vecteurs 1536 dims : ~9 GB minimum
SET GLOBAL innodb_buffer_pool_size = 17179869184;

-- 3. Configuration HNSW
-- Production standard
CREATE INDEX idx_vector ON table(embedding)
USING HNSW
WITH (
    m = 16,
    ef_construction = 200,
    metric = 'cosine'
);

-- 4. Ajuster ef_search selon besoin
-- Session utilisateur standard
SET @@session.hnsw_ef_search = 100;

-- 5. Monitoring
SELECT
    table_name,
    index_name,
    stat_value
FROM mysql.innodb_index_stats
WHERE table_name = 'documents'
AND index_name LIKE 'idx_vector%';
```

### Monitoring des performances

```sql
-- 1. Taille de l'index
SELECT
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb'
AND table_name = 'documents'
AND index_name = 'idx_vector'
AND stat_name = 'size';

-- 2. Performance Schema
SELECT
    object_name,
    index_name,
    count_star AS total_accesses,
    sum_timer_wait / 1000000000 AS total_time_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'mydb'
AND object_name = 'documents'
AND index_name = 'idx_vector';

-- 3. Slow Query Log
-- Activer pour requêtes vectorielles lentes
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.1;  -- 100ms
```

---

## ✅ Points clés à retenir

- 🆕 **Nouveauté 11.8** : Type VECTOR natif + index HNSW pour IA/ML
- 📊 **HNSW** : Algorithme de graphe hiérarchique, recherche en O(log n)
- 🎯 **3 métriques** : Cosine (texte), Euclidean (images), Dot (recommandations)
- ⚙️ **Paramètres clés** : m (connexions), ef_construction (qualité), ef_search (runtime)
- 🚀 **SIMD** : Accélération AVX2/AVX-512/NEON (4-16x plus rapide)
- 🤖 **Cas d'usage** : Semantic search, RAG, recommandations, anomalies, images
- 📏 **Dimensions** : 1 à 16 000, 4 bytes/dimension
- 💡 **m=16, ef=200** : Configuration par défaut optimale pour la plupart des cas
- 🔍 **ef_search** : Ajustable à la volée (trade-off vitesse/précision)
- 🎪 **Intégrations** : OpenAI, LangChain, LlamaIndex support natif
- ⚖️ **Trade-off** : Recall 94% @ 7ms vs 99% @ 35ms
- 🏗️ **Production** : Buffer Pool suffisant + monitoring performance

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 VECTOR Data Type](https://mariadb.com/kb/en/vector-data-type/)
- [📖 HNSW Vector Indexes](https://mariadb.com/kb/en/vector-indexes/)
- [📖 Vector Functions](https://mariadb.com/kb/en/vector-functions/)
- [📖 Vector Performance Tuning](https://mariadb.com/kb/en/vector-performance/)

### Algorithme HNSW

- [📄 HNSW Paper](https://arxiv.org/abs/1603.09320) - Publication originale (2016)
- [Efficient and robust approximate nearest neighbor search](https://arxiv.org/abs/1603.09320)

### Intégrations IA

- [OpenAI Embeddings API](https://platform.openai.com/docs/guides/embeddings)
- [LangChain MariaDB Integration](https://python.langchain.com/docs/integrations/vectorstores/mariadb)
- [LlamaIndex MariaDB](https://docs.llamaindex.ai/en/stable/examples/vector_stores/MariaDBDemo.html)

### Comparaisons et benchmarks

- [MariaDB Vector vs pgvector](https://mariadb.org/vector-search-benchmark/)
- [HNSW vs IVFFlat Performance](https://arxiv.org/abs/1603.09320)
- [Vector Database Comparison](https://benchmark.vectorview.ai/)

### Articles techniques

- [Introducing MariaDB Vector Search](https://mariadb.org/vector-search-announcement/)
- [Building RAG Applications with MariaDB](https://mariadb.org/rag-tutorial/)
- [SIMD Optimizations in Vector Search](https://mariadb.org/simd-vectors/)

---

## ➡️ Section suivante

**[5.4 Création et gestion des index](./04-creation-gestion-index.md)**

Approfondissez la création et gestion pratique des index dans MariaDB : syntaxes CREATE INDEX et ALTER TABLE, options d'indexation, Online DDL pour créer des index sans bloquer la production, suppression et modification d'index, et stratégies de maintenance pour tous les types d'index (B-Tree, Hash, Full-Text, Spatial, VECTOR).

---


⏭️ [Création et gestion des index](/05-index-et-performance/04-creation-gestion-index.md)

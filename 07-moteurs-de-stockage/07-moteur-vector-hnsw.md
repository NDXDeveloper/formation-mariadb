🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.7 Moteur Vector/HNSW : Recherche vectorielle pour IA 🆕

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 7.1-7.2 (Architecture, InnoDB), concepts d'IA/ML, embeddings, algèbre linéaire
> **Public cible** : DBA, Architectes IA, Data Scientists, MLOps Engineers

> **Nouveauté** : MariaDB 11.8 LTS

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre les concepts de recherche vectorielle et d'embeddings
- Maîtriser l'architecture du moteur Vector et l'index HNSW
- Implémenter des cas d'usage IA : RAG, recherche sémantique, recommandations
- Optimiser les performances de recherche vectorielle à grande échelle
- Intégrer MariaDB Vector avec des modèles d'IA (OpenAI, LLaMA, etc.)
- Concevoir des architectures hybrides (SQL + Vector)
- Choisir les bons paramètres d'index selon les cas d'usage
- Monitorer et tuner les performances Vector

---

## Introduction

Le **moteur Vector/HNSW** (introduit dans MariaDB 11.8 LTS) apporte la **recherche vectorielle native** dans MariaDB, permettant de stocker et interroger efficacement des **embeddings** pour les applications d'**IA générative** et de **machine learning**.

### Contexte : L'essor de l'IA générative

```
Évolution des bases de données :
┌────────────────────────────────────────────────────────┐
│ 1970-2000 : Données structurées (SQL)                  │
│  • Nombres, textes, dates                              │
│  • Recherche exacte (WHERE id = 42)                    │
│  • Index B-Tree                                        │
├────────────────────────────────────────────────────────┤
│ 2000-2020 : Données semi-structurées (NoSQL)           │
│  • JSON, documents                                     │
│  • Recherche Full-Text (MATCH AGAINST)                 │
│  • Index inversés                                      │
├────────────────────────────────────────────────────────┤
│ 2020+ : IA et Embeddings (Vector)                      │
│  • Vecteurs de nombres (512-1536 dimensions)           │
│  • Recherche par similarité sémantique                 │
│  • Index HNSW, FAISS                                   │
│  • Cas d'usage : RAG, recherche sémantique, IA         │
└────────────────────────────────────────────────────────┘
```

### Qu'est-ce qu'un embedding ?

Un **embedding** est une représentation numérique d'un objet (texte, image, audio) sous forme de vecteur dense dans un espace vectoriel de haute dimension.

```
Exemple : Embeddings de textes
┌────────────────────────────────────────────────────────┐
│ Texte original → Modèle IA → Vecteur (embedding)       │
│                                                        │
│ "chat"         → OpenAI    → [0.2, -0.5, 0.8, ...]     │
│                  text-embedding-3-small                │
│                  (1536 dimensions)                     │
│                                                        │
│ "chien"        → OpenAI    → [0.3, -0.4, 0.7, ...]     │
│                                                        │
│ "automobile"   → OpenAI    → [-0.8, 0.6, -0.2, ...]    │
└────────────────────────────────────────────────────────┘

Propriété clé : Similarité sémantique
• Distance("chat", "chien") = faible (concepts proches)
• Distance("chat", "automobile") = élevée (concepts éloignés)
```

**Visualisation 2D simplifiée** (en réalité 512-1536D) :

```
        Y
        │
   chien ●                   Animaux
        │ ● chat
        │   ● cheval
    ────┼──────────── X
        │
        │     ● voiture    Véhicules
        │   ● camion
        │
```

### Problème : Recherche dans les bases traditionnelles

```sql
-- Recherche traditionnelle (exact match)
SELECT * FROM products
WHERE name = 'iPhone 15';  -- Trouve seulement "iPhone 15"

-- Full-Text (mots-clés)
SELECT * FROM products
WHERE MATCH(description) AGAINST('smartphone apple');
-- Trouve si mots présents, mais pas de compréhension sémantique

-- Limitations :
-- ❌ "iPhone" vs "téléphone Apple" → Pas de correspondance
-- ❌ "iPhone 15" vs "dernier iPhone" → Pas de correspondance
-- ❌ Pas de recherche par concept/signification
```

### Solution : Recherche vectorielle

```sql
-- Recherche vectorielle (similarité sémantique)
SELECT
    product_name,
    description,
    VEC_DISTANCE_EUCLIDEAN(embedding, ?) AS distance
FROM products
ORDER BY distance
LIMIT 10;

-- Requête : "téléphone dernière génération"
-- Résultats :
-- 1. iPhone 15 Pro Max (distance: 0.12)
-- 2. Samsung Galaxy S24 Ultra (distance: 0.15)
-- 3. Google Pixel 8 Pro (distance: 0.18)
-- → Compréhension sémantique, pas juste mots-clés !
```

---

## Architecture du moteur Vector

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────┐
│                   MariaDB Server                        │
│  ┌────────────────────────────────────────────────────┐ │
│  │           SQL Layer (Parser, Optimizer)            │ │
│  │  • Nouvelles fonctions : VEC_DISTANCE_*()          │ │
│  │  • Support type de données VECTOR                  │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                         ↓ Handler API
┌─────────────────────────────────────────────────────────┐
│              InnoDB Storage Engine                      │
│  (Vector est une extension d'InnoDB, pas un moteur      │
│   séparé - il s'agit d'un nouveau type d'index)         │
│                                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │         HNSW Index (Hierarchical Navigable         │ │
│  │         Small World Graph)                         │ │
│  │                                                    │ │
│  │  Layer 0 (tous les vecteurs)                       │ │
│  │    ┌─────────┐    ┌─────────┐                      │ │
│  │    │ Vector1 │────│ Vector2 │                      │ │
│  │    └─────────┘    └─────────┘                      │ │
│  │         │              │                           │ │
│  │  Layer 1 (échantillon)                             │ │
│  │    ┌─────────┐                                     │ │
│  │    │ VectorA │                                     │ │
│  │    └─────────┘                                     │ │
│  │         │                                          │ │
│  │  Layer N (point d'entrée)                          │ │
│  │    ┌─────────┐                                     │ │
│  │    │  Entry  │                                     │ │
│  │    └─────────┘                                     │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │      Optimisations SIMD                            │ │
│  │  • AVX2, AVX-512 (Intel/AMD)                       │ │
│  │  • ARM NEON                                        │ │
│  │  • Calcul distance vectorisé                       │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│           Stockage InnoDB (Buffer Pool, Disque)         │
│  • Vecteurs stockés comme BLOB                          │
│  • Index HNSW structure graphe                          │
└─────────────────────────────────────────────────────────┘
```

### Type de données VECTOR

```sql
-- Déclaration d'une colonne vector
CREATE TABLE documents (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    embedding VECTOR(1536)  -- Vecteur de 1536 dimensions
);

-- Dimensions supportées : 1-16384
-- Taille en mémoire : dimensions × 4 bytes (float32)
-- Exemple : VECTOR(1536) = 1536 × 4 = 6144 bytes (~6 KB par vecteur)
```

**Formats d'embeddings courants** :

| Modèle | Provider | Dimensions | Taille | Use Case |
|--------|----------|------------|--------|----------|
| text-embedding-3-small | OpenAI | 1536 | 6 KB | Usage général, économique |
| text-embedding-3-large | OpenAI | 3072 | 12 KB | Haute précision |
| text-embedding-ada-002 | OpenAI | 1536 | 6 KB | Legacy (deprecated) |
| all-MiniLM-L6-v2 | HuggingFace | 384 | 1.5 KB | Local, rapide |
| LLaMA-2-7b embeddings | Meta | 4096 | 16 KB | Open-source |
| CLIP | OpenAI | 512 | 2 KB | Images + Textes |

### Index HNSW (Hierarchical Navigable Small World)

HNSW est l'algorithme d'indexation state-of-the-art pour la recherche vectorielle. Il construit un **graphe navigable multi-couches**.

```
Structure HNSW à 3 couches :
┌────────────────────────────────────────────────────────┐
│ Layer 2 (point d'entrée, très peu de nœuds)            │
│                                                        │
│         Entry Point ●                                  │
│                     ↓                                  │
└────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────┐
│ Layer 1 (échantillon, distances longues)               │
│                                                        │
│      ●───────────●───────────●                         │
│      │           │           │                         │
│      ●───────────●───────────●                         │
│                                                        │
└────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────┐
│ Layer 0 (tous les vecteurs, distances courtes)         │
│                                                        │
│  ●──●──●──●──●──●──●──●──●──●──●──●──●──●              │
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │              │
│  ●──●──●──●──●──●──●──●──●──●──●──●──●──●              │
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │              │
│  ●──●──●──●──●──●──●──●──●──●──●──●──●──●              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Algorithme de recherche HNSW** :

```
1. Partir du Entry Point (Layer max)
2. Descendre les layers en trouvant les voisins les plus proches
3. Arriver au Layer 0 avec une bonne approximation
4. Affiner localement dans Layer 0
5. Retourner les K plus proches voisins

Complexité :
• Construction : O(N × log N)
• Recherche : O(log N)  (vs O(N) pour scan linéaire)
• Précision : ~99% vs recherche exacte
```

**Paramètres HNSW** :

```sql
-- Créer un index HNSW
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (
    M = 16,              -- Connexions par nœud (défaut: 16)
    ef_construction = 200, -- Précision construction (défaut: 200)
    metric = 'euclidean'  -- euclidean, cosine, inner_product
);

-- M (max connections) :
-- • Plus grand = Plus de connexions = Meilleure précision, mais index plus gros
-- • Recommandé : 12-48 (défaut 16 bon pour la plupart)

-- ef_construction (exploration factor) :
-- • Plus grand = Construction plus lente, mais meilleure qualité
-- • Recommandé : 100-500 (défaut 200)
```

### Métriques de distance

MariaDB Vector supporte **3 métriques de distance** :

#### 1. Distance Euclidienne (L2)

```
Distance Euclidienne entre A et B :
d(A, B) = √(Σ(Aᵢ - Bᵢ)²)

Exemple 2D :
A = [3, 4]
B = [0, 0]
d = √((3-0)² + (4-0)²) = √(9 + 16) = √25 = 5

SQL :
SELECT VEC_DISTANCE_EUCLIDEAN(embedding, '[0.2, 0.5, 0.8]') AS distance
FROM documents;
```

**Usage** : Mesure de distance "géométrique" classique. Bon pour la plupart des cas.

#### 2. Similarité cosinus (Cosine Similarity)

```
Similarité cosinus entre A et B :
sim(A, B) = (A · B) / (||A|| × ||B||)
distance(A, B) = 1 - sim(A, B)

Exemple 2D :
A = [3, 4]  (magnitude = 5)
B = [6, 8]  (magnitude = 10)
sim = (3×6 + 4×8) / (5 × 10) = 50/50 = 1.0 (colinéaires)

SQL :
SELECT VEC_DISTANCE_COSINE(embedding, '[0.2, 0.5, 0.8]') AS distance
FROM documents;
```

**Usage** : Ignore la magnitude, mesure l'angle. **Idéal pour NLP** (embeddings normalisés).

#### 3. Produit scalaire (Inner Product)

```
Produit scalaire (dot product) :
A · B = Σ(Aᵢ × Bᵢ)
distance = -A · B  (négatif car plus grand = plus similaire)

Exemple 2D :
A = [3, 4]
B = [1, 2]
A · B = 3×1 + 4×2 = 3 + 8 = 11

SQL :
SELECT VEC_DISTANCE_INNER_PRODUCT(embedding, '[0.2, 0.5, 0.8]') AS distance
FROM documents;
```

**Usage** : Rapide (pas de racine carrée), bon pour embeddings déjà normalisés.

**Tableau comparatif** :

| Métrique | Complexité | Usage principal | Normalisé ? |
|----------|-----------|-----------------|-------------|
| Euclidienne | O(n) + sqrt | Distance absolue | Non |
| Cosine | O(n) + 2×sqrt | NLP, textes, normalisés | Oui |
| Inner Product | O(n) | Maximum similarity score | Peut être |

💡 **Recommandation** : Pour NLP (OpenAI, LLaMA, etc.) → **Cosine**. Pour images, audio → **Euclidienne**.

---

## Configuration et utilisation

### Installation et activation

```sql
-- Vérifier support Vector (MariaDB 11.8+)
SELECT VERSION();
-- 11.8.0-MariaDB

-- Le support Vector est intégré, pas de plugin séparé
-- Créer une table avec colonne VECTOR
CREATE TABLE test_vector (
    id INT PRIMARY KEY,
    vec VECTOR(128)
);

-- Si erreur → Vérifier version MariaDB
```

### Création de table

```sql
-- Table pour recherche sémantique de documents
CREATE TABLE knowledge_base (
    doc_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Embedding généré par OpenAI text-embedding-3-small
    embedding VECTOR(1536),

    -- Index traditionnel pour filtrage
    INDEX idx_created (created_at),
    INDEX idx_title (title)
) ENGINE=InnoDB;

-- Créer l'index HNSW
CREATE INDEX idx_embedding ON knowledge_base(embedding)
USING HNSW
WITH (
    M = 16,
    ef_construction = 200,
    metric = 'cosine'  -- Pour NLP
);
```

### Insertion de données

**Méthode 1 : Avec embeddings pré-calculés** :

```sql
-- Embeddings générés via API OpenAI (Python, Node.js, etc.)
INSERT INTO knowledge_base (title, content, embedding) VALUES
(
    'Introduction à MariaDB',
    'MariaDB est un système de gestion de base de données...',
    '[0.023, -0.045, 0.789, ..., 0.234]'  -- 1536 valeurs
);

-- Format : Tableau JSON de floats
-- Dimension doit correspondre à la déclaration VECTOR(1536)
```

**Méthode 2 : Via application (recommandé)** :

```python
# Python avec OpenAI SDK
import openai
import mariadb

# Générer embedding
response = openai.embeddings.create(
    model="text-embedding-3-small",
    input="MariaDB est un système de gestion de base de données..."
)
embedding = response.data[0].embedding  # Liste de 1536 floats

# Insérer dans MariaDB
conn = mariadb.connect(
    host="localhost",
    user="root",
    password="password",
    database="mydb"
)
cursor = conn.cursor()

cursor.execute(
    "INSERT INTO knowledge_base (title, content, embedding) VALUES (?, ?, ?)",
    (
        "Introduction à MariaDB",
        "MariaDB est un système...",
        str(embedding)  # Convertir liste en string JSON
    )
)
conn.commit()
```

### Recherche vectorielle

```sql
-- Recherche des documents les plus similaires
-- 1. Générer l'embedding de la requête (via API OpenAI)
-- Requête utilisateur : "Comment optimiser les performances de ma base de données ?"
-- Embedding : [0.123, -0.456, 0.789, ...]

-- 2. Recherche dans MariaDB
SELECT
    doc_id,
    title,
    content,
    VEC_DISTANCE_COSINE(
        embedding,
        '[0.123, -0.456, 0.789, ..., 0.321]'  -- Embedding de la requête
    ) AS similarity_score
FROM knowledge_base
ORDER BY similarity_score ASC  -- Plus petit score = plus similaire
LIMIT 10;

-- Résultats :
-- ┌────────┬─────────────────────────────┬─────────────┐
-- │ doc_id │ title                       │ score       │
-- ├────────┼─────────────────────────────┼─────────────┤
-- │ 42     │ Optimisation Buffer Pool    │ 0.08        │
-- │ 17     │ Configuration InnoDB        │ 0.12        │
-- │ 89     │ Tuning des index            │ 0.15        │
-- │ ...    │ ...                         │ ...         │
-- └────────┴─────────────────────────────┴─────────────┘
```

**Avec filtres hybrides** :

```sql
-- Recherche vectorielle + filtres SQL traditionnels
SELECT
    doc_id,
    title,
    created_at,
    VEC_DISTANCE_COSINE(embedding, ?) AS score
FROM knowledge_base
WHERE
    created_at >= '2024-01-01'  -- Filtrage temporel
    AND title LIKE '%MariaDB%'   -- Filtrage textuel
ORDER BY score ASC
LIMIT 10;

-- Avantage : Combiner recherche sémantique + filtres précis
```

---

## Cas d'usage : RAG (Retrieval Augmented Generation)

**RAG** est le pattern dominant pour les applications IA avec LLM (ChatGPT, Claude, LLaMA).

### Architecture RAG complète

```
┌──────────────────────────────────────────────────────────┐
│ 1. INDEXATION (offline)                                  │
│                                                          │
│  Documents → Chunking → Embeddings → MariaDB Vector      │
│  (corpus)    (512 tok)  (OpenAI)     (stockage)          │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│ 2. RETRIEVAL (runtime)                                   │
│                                                          │
│  Question → Embedding → Vector Search → Top K docs       │
│  user       (OpenAI)    (MariaDB)       (contexte)       │
│                                                          │
│  "Comment configurer InnoDB ?"                           │
│       ↓                                                  │
│  [0.12, -0.45, ...]  (embedding)                         │
│       ↓                                                  │
│  SELECT ... VEC_DISTANCE_COSINE(...)                     │
│       ↓                                                  │
│  [Doc1: "Configuration InnoDB Buffer Pool...",           │
│   Doc2: "Optimiser innodb_log_file_size...",             │
│   Doc3: "Tuning InnoDB pour SSD..."]                     │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│ 3. GENERATION (LLM)                                      │
│                                                          │
│  Prompt = Question + Contexte (Top K) → LLM → Réponse    │
│                                                          │
│  Prompt:                                                 │
│  """                                                     │
│  Contexte:                                               │
│  - Doc1: Configuration InnoDB Buffer Pool...             │
│  - Doc2: Optimiser innodb_log_file_size...               │
│  - Doc3: Tuning InnoDB pour SSD...                       │
│                                                          │
│  Question: Comment configurer InnoDB ?                   │
│                                                          │
│  Réponds en te basant uniquement sur le contexte.        │
│  """                                                     │
│       ↓                                                  │
│  GPT-4 / Claude / LLaMA                                  │
│       ↓                                                  │
│  "Pour configurer InnoDB optimalement, voici             │
│   les paramètres clés : innodb_buffer_pool_size          │
│   (70-80% RAM), innodb_log_file_size (1-2GB), ..."       │
└──────────────────────────────────────────────────────────┘
```

### Implémentation RAG complète

```python
# app.py - Système RAG avec MariaDB Vector
import openai
import mariadb
from typing import List

class RAGSystem:
    def __init__(self):
        self.conn = mariadb.connect(
            host="localhost",
            user="root",
            password="password",
            database="rag_db"
        )
        self.cursor = self.conn.cursor()
        openai.api_key = "sk-..."

    def chunk_document(self, content: str, chunk_size: int = 512) -> List[str]:
        """Découper document en chunks de 512 tokens"""
        # Simplification : découpe par paragraphes
        paragraphs = content.split('\n\n')
        chunks = []
        current_chunk = ""

        for para in paragraphs:
            if len(current_chunk) + len(para) < chunk_size * 4:  # ~4 chars/token
                current_chunk += para + "\n\n"
            else:
                if current_chunk:
                    chunks.append(current_chunk.strip())
                current_chunk = para + "\n\n"

        if current_chunk:
            chunks.append(current_chunk.strip())

        return chunks

    def generate_embedding(self, text: str) -> List[float]:
        """Générer embedding via OpenAI"""
        response = openai.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return response.data[0].embedding

    def index_document(self, title: str, content: str):
        """Indexer un document (chunking + embeddings)"""
        chunks = self.chunk_document(content)

        for i, chunk in enumerate(chunks):
            embedding = self.generate_embedding(chunk)

            self.cursor.execute(
                """
                INSERT INTO knowledge_base
                (title, content, chunk_index, embedding)
                VALUES (?, ?, ?, ?)
                """,
                (f"{title} (part {i+1})", chunk, i, str(embedding))
            )

        self.conn.commit()
        print(f"Indexed {len(chunks)} chunks for '{title}'")

    def retrieve(self, query: str, top_k: int = 5) -> List[dict]:
        """Retrieval : Recherche vectorielle"""
        # 1. Générer embedding de la requête
        query_embedding = self.generate_embedding(query)

        # 2. Recherche dans MariaDB
        self.cursor.execute(
            """
            SELECT
                doc_id,
                title,
                content,
                VEC_DISTANCE_COSINE(embedding, ?) AS score
            FROM knowledge_base
            ORDER BY score ASC
            LIMIT ?
            """,
            (str(query_embedding), top_k)
        )

        results = []
        for row in self.cursor:
            results.append({
                'doc_id': row[0],
                'title': row[1],
                'content': row[2],
                'score': row[3]
            })

        return results

    def generate(self, query: str, context: List[dict]) -> str:
        """Generation : LLM avec contexte"""
        # Construire le prompt avec contexte
        context_str = "\n\n".join([
            f"Document {i+1}: {doc['title']}\n{doc['content']}"
            for i, doc in enumerate(context)
        ])

        prompt = f"""Contexte (documentation technique) :
{context_str}

Question de l'utilisateur : {query}

Instructions :
- Réponds uniquement en te basant sur le contexte fourni
- Si l'information n'est pas dans le contexte, dis-le clairement
- Sois concis et technique

Réponse :"""

        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "Tu es un assistant technique expert en bases de données."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3  # Basse température pour réponses factuelles
        )

        return response.choices[0].message.content

    def query(self, user_question: str) -> dict:
        """Pipeline RAG complet"""
        # 1. Retrieval
        relevant_docs = self.retrieve(user_question, top_k=5)

        # 2. Generation
        answer = self.generate(user_question, relevant_docs)

        return {
            'question': user_question,
            'answer': answer,
            'sources': relevant_docs
        }

# Utilisation
rag = RAGSystem()

# Indexation (une fois)
rag.index_document(
    "Configuration InnoDB",
    """Le Buffer Pool est le composant le plus critique d'InnoDB.
    Il doit être configuré à 70-80% de la RAM disponible via
    innodb_buffer_pool_size. Pour un serveur de 64 GB, utilisez
    innodb_buffer_pool_size=48G..."""
)

# Requête
result = rag.query("Comment dimensionner le Buffer Pool InnoDB ?")
print("Question:", result['question'])
print("Réponse:", result['answer'])
print("Sources utilisées:", len(result['sources']))
```

**Schéma de la base** :

```sql
CREATE TABLE knowledge_base (
    doc_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    chunk_index INT,           -- Position du chunk dans le document
    source_url VARCHAR(1000),  -- URL d'origine (optionnel)
    metadata JSON,             -- Métadonnées additionnelles
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    embedding VECTOR(1536),

    INDEX idx_embedding ON knowledge_base(embedding)
    USING HNSW
    WITH (M=16, ef_construction=200, metric='cosine')
);
```

---

## Cas d'usage : Recherche sémantique

### E-commerce : Recherche produits par description

```sql
-- Scénario : Boutique en ligne avec 1 million de produits

-- Table produits
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(500),
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2),

    -- Embedding de la description
    description_embedding VECTOR(1536),

    INDEX idx_category (category),
    INDEX idx_price (price),
    INDEX idx_embedding ON products(description_embedding)
    USING HNSW WITH (M=24, metric='cosine')
);

-- Recherche utilisateur : "chaussures de running confortables pour marathon"
-- (embedding généré par l'application)

SELECT
    product_id,
    name,
    description,
    price,
    VEC_DISTANCE_COSINE(
        description_embedding,
        ?  -- Embedding de la requête
    ) AS relevance_score
FROM products
WHERE
    category = 'Sports'  -- Filtrage pré-sélection
    AND price BETWEEN 50 AND 200
ORDER BY relevance_score ASC
LIMIT 20;

-- Résultats pertinents même si mots différents :
-- • "Nike Air Zoom Pegasus - running shoes with cushioning..."
-- • "Adidas Ultraboost - marathon ready sneakers..."
-- • "Asics Gel-Kayano - comfortable long distance trainers..."
```

### Support client : FAQ intelligente

```sql
-- Table FAQ avec embeddings
CREATE TABLE faq (
    faq_id INT PRIMARY KEY AUTO_INCREMENT,
    question TEXT,
    answer TEXT,
    category VARCHAR(100),
    question_embedding VECTOR(1536),

    INDEX idx_embedding ON faq(question_embedding)
    USING HNSW WITH (metric='cosine')
);

-- Exemple de questions indexées :
-- "Comment réinitialiser mon mot de passe ?"
-- "Quels sont les délais de livraison ?"
-- "Comment retourner un produit ?"

-- Requête client : "J'ai oublié mon code d'accès"
-- → Trouve automatiquement "Comment réinitialiser mon mot de passe ?"

SELECT
    question,
    answer,
    VEC_DISTANCE_COSINE(question_embedding, ?) AS similarity
FROM faq
ORDER BY similarity ASC
LIMIT 3;
```

---

## Cas d'usage : Système de recommandation

### Recommandation de contenu

```sql
-- Table articles avec embeddings de contenu
CREATE TABLE articles (
    article_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    author VARCHAR(200),
    content_embedding VECTOR(1536),

    INDEX idx_embedding ON articles(content_embedding)
    USING HNSW
);

-- Recommandation basée sur un article lu
-- Article ID 42 : "Introduction au Machine Learning"

SELECT
    a.article_id,
    a.title,
    VEC_DISTANCE_COSINE(
        a.content_embedding,
        (SELECT content_embedding FROM articles WHERE article_id = 42)
    ) AS similarity
FROM articles a
WHERE a.article_id != 42  -- Exclure l'article source
ORDER BY similarity ASC
LIMIT 10;

-- Résultats similaires :
-- • "Deep Learning pour débutants"
-- • "Réseaux de neurones expliqués"
-- • "IA et apprentissage supervisé"
```

### Détection de duplications

```sql
-- Détecter articles dupliqués ou très similaires
SELECT
    a1.article_id AS id1,
    a2.article_id AS id2,
    a1.title AS title1,
    a2.title AS title2,
    VEC_DISTANCE_COSINE(a1.content_embedding, a2.content_embedding) AS similarity
FROM articles a1
JOIN articles a2 ON a1.article_id < a2.article_id
WHERE VEC_DISTANCE_COSINE(a1.content_embedding, a2.content_embedding) < 0.05  -- Seuil
ORDER BY similarity ASC
LIMIT 100;

-- Trouve les paires d'articles quasi-identiques
-- (plagiat, réutilisation de contenu, etc.)
```

---

## Performance et optimisations

### Dimensionnement de l'index

**Taille de l'index HNSW** :

```
Taille index HNSW ≈ N × d × 4 bytes × (M + 1)

Où :
• N = nombre de vecteurs
• d = dimensions (ex: 1536)
• M = connexions par nœud (défaut 16)
• 4 bytes = float32

Exemple :
• 1 million de vecteurs
• 1536 dimensions
• M = 16
• Taille = 1M × 1536 × 4 × 17 = 104 GB

Recommandations :
• RAM Buffer Pool > taille index pour performance optimale
• Si index > RAM : Performance dégradée (I/O disque)
```

**Optimisation Buffer Pool** :

```ini
# my.cnf
# Pour 1M vecteurs (104 GB index)
innodb_buffer_pool_size = 120G  # Index + working set

# Multiples instances pour concurrence
innodb_buffer_pool_instances = 16
```

### Paramètres d'index selon cas d'usage

```sql
-- CAS 1 : Haute précision (recherche scientifique)
CREATE INDEX idx_high_precision ON documents(embedding)
USING HNSW WITH (
    M = 48,                -- Plus de connexions
    ef_construction = 500, -- Construction précise
    metric = 'euclidean'
);
-- Avantage : Précision ~99.5%
-- Inconvénient : Index 3× plus gros, construction 5× plus lente

-- CAS 2 : Haute performance (e-commerce)
CREATE INDEX idx_high_performance ON products(embedding)
USING HNSW WITH (
    M = 12,                -- Moins de connexions
    ef_construction = 100, -- Construction rapide
    metric = 'cosine'
);
-- Avantage : Recherche 2× plus rapide, index compact
-- Inconvénient : Précision ~95%

-- CAS 3 : Équilibré (recommandé)
CREATE INDEX idx_balanced ON knowledge_base(embedding)
USING HNSW WITH (
    M = 16,                -- Défaut
    ef_construction = 200, -- Défaut
    metric = 'cosine'
);
-- Bon compromis pour la plupart des cas
```

### Optimisation des requêtes

```sql
-- ✅ BON : Filtrage avant recherche vectorielle
SELECT ...
FROM products
WHERE
    category = 'Electronics'  -- Index B-Tree, filtrage rapide
    AND price > 100
    AND VEC_DISTANCE_COSINE(embedding, ?) < 0.3  -- Recherche sur subset
ORDER BY VEC_DISTANCE_COSINE(embedding, ?) ASC
LIMIT 10;

-- ❌ ÉVITER : Recherche vectorielle sur toute la table si possible
SELECT ...
FROM products  -- 10M lignes
WHERE VEC_DISTANCE_COSINE(embedding, ?) < 0.1  -- Scan 10M vecteurs
LIMIT 10;

-- MIEUX : Partitionnement
CREATE TABLE products_electronics (...) ENGINE=InnoDB;
CREATE TABLE products_clothing (...) ENGINE=InnoDB;
-- Recherche vectorielle sur partition pertinente uniquement
```

### Parallélisation des recherches

```sql
-- MariaDB Vector utilise automatiquement le parallélisme InnoDB

-- Configuration
SET GLOBAL innodb_read_io_threads = 16;
SET GLOBAL innodb_write_io_threads = 16;

-- Requêtes vectorielles bénéficient du thread pool
-- Pas de configuration spéciale nécessaire
```

### Optimisations SIMD

MariaDB Vector utilise automatiquement les instructions SIMD du CPU :

```
CPU Support :
┌────────────────────────────────────────────────────┐
│ Intel/AMD :                                        │
│  • AVX2 (256-bit) : 2× plus rapide que scalaire    │
│  • AVX-512 (512-bit) : 4× plus rapide              │
│                                                    │
│ ARM :                                              │
│  • NEON (128-bit) : 2× plus rapide                 │
│                                                    │
│ Apple Silicon :                                    │
│  • M1/M2/M3 NEON : Optimisé                        │
└────────────────────────────────────────────────────┘

Vérifier support :
shell> lscpu | grep -E "avx|sse"
Flags: ... avx avx2 avx512f ...

MariaDB détecte automatiquement et utilise les meilleures instructions
```

---

## Monitoring et diagnostics

### Métriques de performance

```sql
-- Statistiques index HNSW
SELECT
    TABLE_NAME,
    INDEX_NAME,
    CARDINALITY,
    INDEX_LENGTH / 1024 / 1024 / 1024 AS size_gb
FROM information_schema.STATISTICS
WHERE INDEX_TYPE = 'HNSW';

-- Performance des requêtes vectorielles
SHOW STATUS LIKE 'vec_%';
-- +---------------------------+----------+
-- | Variable_name             | Value    |
-- +---------------------------+----------+
-- | vec_distance_calls        | 1000000  |
-- | vec_distance_avg_time_ms  | 0.05     |
-- | vec_hnsw_navigations      | 850000   |
-- +---------------------------+----------+

-- Analyser une requête
EXPLAIN
SELECT ...
FROM knowledge_base
ORDER BY VEC_DISTANCE_COSINE(embedding, '[...]') ASC
LIMIT 10;
-- Vérifie que l'index HNSW est utilisé
```

### Benchmarking

```sql
-- Test de performance
SET @query_embedding = '[0.1, -0.2, 0.3, ..., 0.5]';

-- 1000 recherches
SELECT BENCHMARK(1000, (
    SELECT doc_id
    FROM knowledge_base
    ORDER BY VEC_DISTANCE_COSINE(embedding, @query_embedding) ASC
    LIMIT 10
));

-- Résultat typique :
-- 1000 queries : 2.5 secondes
-- → 2.5 ms par recherche (400 req/sec)
```

---

## Limitations et considérations

### Limitations techniques

| Limitation | Détails | Workaround |
|------------|---------|------------|
| **Dimensions max** | 16384 | Suffisant (modèles actuels ≤ 4096) |
| **Précision HNSW** | ~95-99% vs exact | Acceptable pour la plupart des cas |
| **Taille index** | Peut être massive | Dimensionner RAM en conséquence |
| **UPDATE lent** | Reconstruction partielle index | Batch updates |
| **Pas de partitionnement natif** | Index par partition | Créer tables séparées |

### Cas où Vector n'est PAS approprié

```sql
-- ❌ Mauvais cas 1 : Recherche exacte
SELECT * FROM products WHERE name = 'iPhone 15';
-- Utilisez index B-Tree classique

-- ❌ Mauvais cas 2 : Petites tables (< 10 000 lignes)
-- Overhead index HNSW non justifié
-- Scan linéaire plus rapide

-- ❌ Mauvais cas 3 : Embeddings non pertinents
-- Si embeddings de mauvaise qualité → Résultats aléatoires
-- Qualité embedding > Qualité index

-- ❌ Mauvais cas 4 : Temps réel critique (< 1ms)
-- Latence recherche vectorielle : 5-50 ms
-- Utilisez cache applicatif si nécessaire
```

### Coûts embeddings

```
Coûts API OpenAI (text-embedding-3-small) :
┌────────────────────────────────────────────────────┐
│ Prix : $0.02 / 1M tokens                           │
│                                                    │
│ Exemples :                                         │
│ • 1 document (500 tokens) : $0.00001               │
│ • 100 000 documents : $1                           │
│ • 1 million de documents : $10                     │
│                                                    │
│ Alternatives :                                     │
│ • Modèles open-source (HuggingFace) : Gratuit      │
│ • Self-hosted (LLaMA) : Coût infra uniquement      │
└────────────────────────────────────────────────────┘
```

---

## Comparaison avec autres solutions

| Solution | Avantages | Inconvénients | Usage |
|----------|-----------|---------------|-------|
| **MariaDB Vector** | • Intégré SQL<br>• ACID transactions<br>• Filtres hybrides | • Moins mature que spécialisés | Données SQL + Vector |
| **Pinecone** | • Spécialisé<br>• Très rapide<br>• Managed | • Coût élevé<br>• Vendor lock-in | Pure vector search |
| **Weaviate** | • Open-source<br>• Performant | • Complexité setup<br>• Pas de SQL | Microservice vector |
| **pgvector** | • PostgreSQL<br>• Mature | • Performance limitée grande échelle | PostgreSQL users |
| **Elasticsearch** | • Mature<br>• Full ecosystem | • Complexe<br>• Ressources++ | Search engine |
| **Milvus** | • Très performant<br>• Scalable | • Complexité<br>• Pas de SQL | Grandes échelles |

💡 **Recommandation** : MariaDB Vector si données déjà dans MariaDB + besoin de requêtes hybrides SQL+Vector.

---

## ✅ Points clés à retenir

1. **Embeddings** : Représentation vectorielle de contenu (texte, image, audio) pour capture de signification sémantique.

2. **HNSW** : Index state-of-the-art, recherche en O(log N) avec précision ~99% (vs scan linéaire O(N)).

3. **3 métriques** : Euclidienne (distance géométrique), Cosine (NLP recommandé), Inner Product (rapide).

4. **RAG pattern** : Retrieval (Vector Search) + Augmentation (Contexte) + Generation (LLM) = Applications IA modernes.

5. **Configuration** : M=16, ef_construction=200, metric='cosine' bon pour la plupart des cas.

6. **Dimensions** : Supporte 1-16384 dimensions (OpenAI=1536, LLaMA=4096).

7. **Requêtes hybrides** : Combiner filtres SQL traditionnels + recherche vectorielle = puissance unique.

8. **Performance** : 5-50 ms par recherche, 400-1000 req/sec sur matériel moderne.

9. **Taille index** : ~N × d × 4 × (M+1) bytes - Dimensionner Buffer Pool en conséquence.

10. **SIMD** : Utilise automatiquement AVX2/AVX-512/NEON pour calculs vectoriels optimisés.

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Vector Data Type](https://mariadb.com/kb/en/vector/)
- [📖 HNSW Index](https://mariadb.com/kb/en/hnsw-index/)
- [📖 Vector Functions](https://mariadb.com/kb/en/vector-functions/)
- [📖 Vector Performance Tuning](https://mariadb.com/kb/en/vector-performance/)

### Articles et guides

- [MariaDB 11.8: Native Vector Search](https://mariadb.org/vector-search-11-8/)
- [Building RAG Applications with MariaDB](https://mariadb.com/resources/blog/rag-mariadb/)
- [Vector Search Benchmarks](https://mariadb.com/kb/en/vector-benchmarks/)

### Modèles d'embeddings

- [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [HuggingFace Sentence Transformers](https://huggingface.co/sentence-transformers)
- [Cohere Embeddings](https://cohere.com/embed)

### Algorithmes et recherche

- [HNSW Paper (Malkov & Yashunin)](https://arxiv.org/abs/1603.09320)
- [Vector Search Explained](https://www.pinecone.io/learn/vector-search/)
- [ANN Benchmarks](http://ann-benchmarks.com/)

---

## ➡️ Prochaines sections

**[7.8 Comparaison et choix du moteur approprié](/07-moteurs-de-stockage/08-comparaison-choix-moteur.md)** : Tableau comparatif complet de tous les moteurs, critères de décision, arbres de décision pour choisir le bon moteur.

Puis nous terminerons avec :
- **7.9** : Conversion entre moteurs (stratégies, exemples, migration)

---

**📌 Mémo Architecte** : "Vector = recherche sémantique. Pour RAG, NLP, recommandations. HNSW index avec M=16, metric=cosine. Dimensionner Buffer Pool = taille index. Qualité embeddings > qualité technique."

**🎯 Cas d'usage typique** : Application IA avec ChatGPT → Générer embeddings (OpenAI) → Stocker dans MariaDB Vector → Retrieval pour RAG → Contexte pour LLM. Simple, efficace, production-ready.

**🆕 Innovation 11.8** : Premier SGBD relationnel majeur avec recherche vectorielle native. Pas besoin de base séparée (Pinecone, Weaviate). SQL + Vector = One database for everything.

⏭️ [Comparaison et choix du moteur approprié](/07-moteurs-de-stockage/08-comparaison-choix-moteur.md)

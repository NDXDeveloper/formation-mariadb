🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.10 MariaDB Vector : Recherche Vectorielle pour l'IA/RAG 🆕

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Compréhension IA/ML de base, embeddings, concept de similarité vectorielle

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les **embeddings vectoriels** et la recherche sémantique
- Utiliser le **type VECTOR** pour stocker des vecteurs
- Créer des **index HNSW** pour recherche rapide
- Maîtriser les **fonctions de distance** (cosine, euclidean, dot product)
- Implémenter un système **RAG** (Retrieval Augmented Generation)
- Intégrer MariaDB avec **OpenAI, Hugging Face, LLMs**
- Optimiser les **performances** (SIMD, index tuning)
- Combiner recherche **vectorielle + SQL** classique
- Comparer MariaDB Vector aux **bases vectorielles spécialisées**

---

## Introduction

**MariaDB Vector** (introduit dans MariaDB 11.7, stable 11.8 LTS) apporte le support natif de la **recherche vectorielle** dans MariaDB, permettant de stocker et rechercher des **embeddings** générés par des modèles d'IA pour des cas d'usage comme la recherche sémantique, les systèmes RAG, et la recommandation.

### Qu'est-ce que la Recherche Vectorielle ?

**Définition** : Technique permettant de trouver des éléments similaires en comparant leurs représentations numériques (vecteurs) dans un espace multidimensionnel.

**Exemple concret** :
```python
# Texte → Vecteur (embedding)
"MariaDB est une base de données SQL"
  ↓ [Modèle AI : OpenAI text-embedding-3-small]
  ↓
[0.023, -0.145, 0.892, ..., 0.234]  # Vecteur 1536 dimensions

"MySQL est un système de gestion de base de données"
  ↓ [Modèle AI]
  ↓
[0.019, -0.139, 0.881, ..., 0.227]  # Vecteur similaire !

# Similarité cosine : 0.94 (très similaire)
# → Ces deux phrases ont un sens proche
```

**Différence avec recherche textuelle classique** :

| Recherche Classique (FULLTEXT) | Recherche Vectorielle |
|--------------------------------|------------------------|
| Correspondance de **mots-clés** | Correspondance de **sens** |
| "base de données" ≠ "SGBD" | "base de données" ≈ "SGBD" |
| "chat" (animal) = "chat" (discussion) | "chat" (contexte) distingué |
| Exact/proximité lexicale | Similarité sémantique |
| Rapide pour mots exacts | Rapide avec index HNSW |

**Exemple limitation recherche textuelle** :
```sql
-- Recherche FULLTEXT classique
SELECT * FROM documents
WHERE MATCH(content) AGAINST('base de données');
-- ✅ Trouve : "base de données SQL"
-- ❌ Ne trouve PAS : "SGBD relationnel"
-- ❌ Ne trouve PAS : "système de gestion de données"

-- Recherche Vectorielle
SELECT * FROM documents
ORDER BY VEC_DISTANCE_COSINE(embedding, query_vector)
LIMIT 10;
-- ✅ Trouve : "base de données SQL"
-- ✅ Trouve : "SGBD relationnel" (sens similaire)
-- ✅ Trouve : "système de gestion de données" (synonyme)
```

### Pourquoi MariaDB Vector ?

**Problème** : Avant MariaDB Vector, stack typique pour recherche vectorielle :

```
Application
    ↓
MariaDB (données métier) + Pinecone/Weaviate/Qdrant (vecteurs)
    ↓
Synchronisation complexe, 2 bases à maintenir
```

**Solution** : MariaDB Vector unifie tout dans une seule base :

```
Application
    ↓
MariaDB (données métier + vecteurs + recherche hybride)
    ↓
Une seule base, transactions ACID, jointures SQL+Vector
```

**Avantages MariaDB Vector** :

1. **🔄 Intégration Transparente**
   - Vecteurs dans même base que données métier
   - Transactions ACID sur données + vecteurs
   - Jointures SQL classique + recherche vectorielle

2. **💰 Coût Réduit**
   - Pas de base vectorielle séparée (Pinecone = $$)
   - Infrastructure simplifiée
   - Moins de synchronisation

3. **🚀 Performance**
   - Index HNSW optimisé
   - Support SIMD (AVX2, AVX-512)
   - Calcul vectoriel au niveau moteur

4. **🔒 Sécurité et Fiabilité**
   - Encryption at rest native
   - Backup/restore standard MariaDB
   - Réplication maître-esclave

5. **🛠️ Flexibilité SQL**
   - Requêtes hybrides (filtre SQL + similarité)
   - Agrégations, GROUP BY sur résultats
   - CTEs, window functions

---

## Architecture MariaDB Vector

### Composants Principaux

```
┌────────────────────────────────────────────────────┐
│              Application / LLM                     │
│  (OpenAI, Hugging Face, Local Model)               │
└──────────────────┬─────────────────────────────────┘
                   │ Embeddings (vecteurs)
                   ↓
┌────────────────────────────────────────────────────┐
│              MariaDB Vector Storage                │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │  Table avec colonne VECTOR                 │    │
│  │  CREATE TABLE docs (                       │    │
│  │    id INT,                                 │    │
│  │    content TEXT,                           │    │
│  │    embedding VECTOR(1536)  ← Type natif    │    │
│  │  )                                         │    │
│  └────────────────────────────────────────────┘    │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │  Index HNSW (Approximate Nearest Neighbor) │    │
│  │  CREATE INDEX idx_emb ON docs(embedding)   │    │
│  │    USING HNSW;                             │    │
│  └────────────────────────────────────────────┘    │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │  Fonctions de Distance                     │    │
│  │  - VEC_DISTANCE_COSINE(v1, v2)             │    │
│  │  - VEC_DISTANCE_EUCLIDEAN(v1, v2)          │    │
│  │  - VEC_DISTANCE_DOT(v1, v2)                │    │
│  └────────────────────────────────────────────┘    │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │  Optimisations SIMD                        │    │
│  │  - AVX2 (256-bit)                          │    │
│  │  - AVX-512 (512-bit)                       │    │
│  │  - Calculs parallèles CPU                  │    │
│  └────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────┘
```

### Type VECTOR

**Déclaration** :
```sql
-- VECTOR(dimensions)
-- dimensions : 1 à 65535 (limite pratique ~10000)

CREATE TABLE embeddings (
  id INT PRIMARY KEY AUTO_INCREMENT,
  text_content TEXT,
  
  -- Vecteur 1536 dimensions (OpenAI text-embedding-3-small)
  embedding VECTOR(1536),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Autres dimensions courantes
embedding_small VECTOR(384)   -- sentence-transformers/all-MiniLM-L6-v2
embedding_medium VECTOR(768)  -- BERT base
embedding_large VECTOR(1536)  -- OpenAI text-embedding-3-small
embedding_huge VECTOR(3072)   -- OpenAI text-embedding-3-large
```

**Stockage** :
```sql
-- Format : [val1, val2, val3, ..., valN]
-- Valeurs : FLOAT (32-bit)

-- Insérer vecteur
INSERT INTO embeddings (text_content, embedding) VALUES (
  'MariaDB supporte maintenant les vecteurs',
  '[0.023, -0.145, 0.892, ..., 0.234]'
  -- Chaîne JSON convertie automatiquement en VECTOR
);

-- Depuis variable/programme
SET @vec = '[0.1, 0.2, 0.3, 0.4]';
INSERT INTO embeddings (text_content, embedding) VALUES (
  'Test',
  @vec
);
```

### Index HNSW

**HNSW** (Hierarchical Navigable Small World) : Algorithme d'indexation pour recherche rapide de voisins les plus proches (k-NN).

**Principe** :
```
Structure multi-niveau hiérarchique

Niveau 2 (sparse) :  o─────o─────o
                     │     │     │
Niveau 1 (medium) :  o──o──o──o──o──o
                     │  │  │  │  │  │
Niveau 0 (dense)  :  o─o─o─o─o─o─o─o─o

Recherche : Commence en haut, descend progressivement
Complexité : O(log n) au lieu de O(n) pour scan complet
```

**Création** :
```sql
-- Créer index HNSW sur colonne VECTOR
CREATE INDEX idx_embedding ON embeddings(embedding)
  USING HNSW;

-- Paramètres optionnels (tuning avancé)
CREATE INDEX idx_embedding ON embeddings(embedding)
  USING HNSW
  WITH (
    M = 16,              -- Nombre de connexions par nœud
    ef_construction = 200, -- Qualité construction index
    distance = 'cosine'   -- Métrique de distance
  );
```

**Paramètres HNSW** :

| Paramètre | Description | Défaut | Impact |
|-----------|-------------|--------|--------|
| **M** | Connexions par nœud | 16 | ↑ M = meilleure précision, + mémoire |
| **ef_construction** | Qualité construction | 200 | ↑ ef = meilleur index, construction + lente |
| **distance** | Métrique | cosine | cosine, euclidean, dot |

**Trade-offs** :
- M=8 : Rapide, moins précis, compact
- M=16 : Équilibré (recommandé)
- M=32 : Très précis, plus lent, volumineux

### Fonctions de Distance

**3 fonctions principales** :

#### 1. VEC_DISTANCE_COSINE (Recommandé)

**Formule** : 
```
distance = 1 - (A·B) / (||A|| * ||B||)
où A·B = produit scalaire
||A|| = norme euclidienne de A
```

**Plage** : 0 (identique) à 2 (opposé)  
**Usage** : Similarité sémantique (texte, images)

```sql
-- Recherche documents similaires
SELECT 
  id,
  text_content,
  VEC_DISTANCE_COSINE(embedding, @query_vector) AS distance
FROM embeddings
ORDER BY distance ASC
LIMIT 10;
-- distance proche de 0 = très similaire
```

**Avantage** : Insensible à la magnitude, compare direction dans l'espace.

#### 2. VEC_DISTANCE_EUCLIDEAN

**Formule** : 
```
distance = √(Σ(Ai - Bi)²)
```

**Plage** : 0 (identique) à ∞  
**Usage** : Distance géométrique absolue

```sql
SELECT 
  id,
  VEC_DISTANCE_EUCLIDEAN(embedding, @query_vector) AS distance
FROM embeddings
ORDER BY distance ASC;
```

**Avantage** : Métrique classique, intuitive.  
**Inconvénient** : Sensible à la magnitude des vecteurs.

#### 3. VEC_DISTANCE_DOT

**Formule** : 
```
distance = -(A·B)
Produit scalaire négatif (pour ORDER BY ASC)
```

**Plage** : -∞ à ∞ (négatif de produit scalaire)  
**Usage** : Vecteurs normalisés, calcul rapide

```sql
SELECT 
  id,
  VEC_DISTANCE_DOT(embedding, @query_vector) AS distance
FROM embeddings
ORDER BY distance ASC;
```

**Avantage** : Calcul le plus rapide.  
**Note** : Équivalent à cosine si vecteurs normalisés.

**Comparaison** :

| Distance | Vitesse | Précision Texte | Normalisé Requis |
|----------|---------|-----------------|------------------|
| **Cosine** | Moyen | ✅ Excellent | ❌ Non |
| **Euclidean** | Moyen | ⚠️ Bon | ❌ Non |
| **Dot Product** | ✅ Rapide | ✅ Excellent | ✅ Oui |

**Recommandation** : **Cosine** pour texte/sémantique, **Dot** si vecteurs normalisés (gain performance).

---

## Workflow RAG Complet

### Architecture RAG (Retrieval Augmented Generation)

```
1. Indexation (offline)
   Document → Chunking → Embeddings → MariaDB Vector

2. Recherche (online)
   Question → Embedding → Similarité → Top-K Documents

3. Génération (online)
   Question + Documents → LLM → Réponse augmentée
```

### Exemple Complet : Base de Connaissances Technique

#### Étape 1 : Création Table

```sql
-- Table pour base de connaissances
CREATE TABLE knowledge_base (
  doc_id INT PRIMARY KEY AUTO_INCREMENT,
  
  -- Métadonnées
  title VARCHAR(255),
  category VARCHAR(100),
  url VARCHAR(500),
  source VARCHAR(100),
  
  -- Contenu
  chunk_text TEXT,  -- Fragment de texte (500-1000 tokens)
  chunk_index INT,  -- Numéro du fragment dans document
  
  -- Embedding vectoriel
  embedding VECTOR(1536),  -- OpenAI text-embedding-3-small
  
  -- Audit
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  -- Index
  INDEX idx_category (category),
  INDEX idx_source (source),
  FULLTEXT INDEX ft_chunk (chunk_text),  -- Recherche hybride
  INDEX idx_embedding ON (embedding) USING HNSW  -- ← Index vectoriel
);
```

#### Étape 2 : Insertion Documents (avec Python)

```python
import mariadb
import openai
from typing import List

# Configuration
openai.api_key = "sk-..."
conn = mariadb.connect(
    user="user",
    password="pass",
    host="localhost",
    database="rag_db"
)
cursor = conn.cursor()

def chunk_text(text: str, chunk_size: int = 500) -> List[str]:
    """Découper texte en fragments de ~chunk_size mots"""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size):
        chunk = ' '.join(words[i:i+chunk_size])
        chunks.append(chunk)
    return chunks

def get_embedding(text: str) -> List[float]:
    """Générer embedding via OpenAI"""
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def index_document(title: str, content: str, category: str, url: str):
    """Indexer un document dans la base vectorielle"""
    chunks = chunk_text(content, chunk_size=500)
    
    for idx, chunk in enumerate(chunks):
        # Générer embedding
        embedding = get_embedding(chunk)
        embedding_str = '[' + ','.join(map(str, embedding)) + ']'
        
        # Insérer dans MariaDB
        cursor.execute("""
            INSERT INTO knowledge_base 
            (title, category, url, source, chunk_text, chunk_index, embedding)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            title,
            category,
            url,
            'documentation',
            chunk,
            idx,
            embedding_str
        ))
    
    conn.commit()
    print(f"Indexed: {title} ({len(chunks)} chunks)")

# Exemple : Indexer documentation MariaDB
index_document(
    title="MariaDB Vector Documentation",
    content="""
    MariaDB Vector enables semantic search and AI applications.
    The VECTOR data type stores embeddings generated by ML models.
    HNSW index provides fast approximate nearest neighbor search.
    Vector search can be combined with traditional SQL queries.
    [... contenu complet ...]
    """,
    category="Database",
    url="https://mariadb.com/kb/en/vector/"
)

# Indexer plusieurs documents
documents = [
    {"title": "Python Best Practices", "content": "...", "category": "Programming"},
    {"title": "SQL Optimization Guide", "content": "...", "category": "Database"},
    # ...
]

for doc in documents:
    index_document(**doc)
```

#### Étape 3 : Recherche Sémantique

```python
def semantic_search(query: str, limit: int = 5, category: str = None):
    """Recherche sémantique dans la base de connaissances"""
    
    # 1. Générer embedding de la question
    query_embedding = get_embedding(query)
    query_embedding_str = '[' + ','.join(map(str, query_embedding)) + ']'
    
    # 2. Recherche vectorielle
    sql = """
        SELECT 
            doc_id,
            title,
            category,
            chunk_text,
            url,
            VEC_DISTANCE_COSINE(embedding, ?) AS similarity
        FROM knowledge_base
    """
    
    # Filtrage optionnel par catégorie (recherche hybride)
    params = [query_embedding_str]
    if category:
        sql += " WHERE category = ?"
        params.append(category)
    
    sql += """
        ORDER BY similarity ASC
        LIMIT ?
    """
    params.append(limit)
    
    cursor.execute(sql, params)
    results = cursor.fetchall()
    
    return [{
        'doc_id': row[0],
        'title': row[1],
        'category': row[2],
        'text': row[3],
        'url': row[4],
        'similarity': row[5]
    } for row in results]

# Exemple recherche
results = semantic_search(
    query="Comment optimiser les requêtes SQL lentes ?",
    limit=5,
    category="Database"
)

for i, result in enumerate(results, 1):
    print(f"\n{i}. {result['title']} (similarity: {result['similarity']:.4f})")
    print(f"   Category: {result['category']}")
    print(f"   Text preview: {result['text'][:200]}...")
    print(f"   URL: {result['url']}")

# Résultat typique :
# 1. SQL Optimization Guide (similarity: 0.1234)
#    Category: Database
#    Text preview: To optimize slow queries, start by using EXPLAIN...
#    URL: https://...
```

#### Étape 4 : Génération RAG (Question + Contexte → Réponse)

```python
def rag_query(question: str, context_limit: int = 3):
    """RAG : Retrieval + Augmented Generation"""
    
    # 1. Retrieval : Rechercher contexte pertinent
    context_docs = semantic_search(question, limit=context_limit)
    
    # 2. Construire prompt avec contexte
    context_text = "\n\n".join([
        f"Document {i+1} ({doc['title']}):\n{doc['text']}"
        for i, doc in enumerate(context_docs)
    ])
    
    prompt = f"""
    Tu es un assistant technique expert. Utilise UNIQUEMENT les documents fournis ci-dessous pour répondre à la question.
    Si la réponse n'est pas dans les documents, dis-le clairement.
    
    DOCUMENTS DE RÉFÉRENCE:
    {context_text}
    
    QUESTION: {question}
    
    RÉPONSE:
    """
    
    # 3. Génération avec LLM
    response = openai.chat.completions.create(
        model="gpt-4-turbo",
        messages=[
            {"role": "system", "content": "Tu es un assistant technique précis."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.3  # Réponse factuelle, peu créative
    )
    
    answer = response.choices[0].message.content
    
    return {
        'question': question,
        'answer': answer,
        'sources': [
            {'title': doc['title'], 'url': doc['url']}
            for doc in context_docs
        ]
    }

# Exemple utilisation
result = rag_query("Comment créer un index HNSW dans MariaDB ?")

print(f"Question: {result['question']}\n")
print(f"Réponse: {result['answer']}\n")
print("Sources:")
for source in result['sources']:
    print(f"  - {source['title']}: {source['url']}")

# Résultat typique :
# Question: Comment créer un index HNSW dans MariaDB ?
# 
# Réponse: Pour créer un index HNSW dans MariaDB, utilisez la syntaxe :
# CREATE INDEX idx_name ON table(vector_column) USING HNSW;
# Vous pouvez également spécifier des paramètres comme M et ef_construction...
# 
# Sources:
#   - MariaDB Vector Documentation: https://mariadb.com/kb/en/vector/
#   - HNSW Index Guide: https://...
```

#### Étape 5 : Recherche Hybride (SQL + Vector)

```sql
-- Combiner filtres SQL classiques + similarité vectorielle

-- Recherche dans catégorie spécifique
SELECT 
  doc_id,
  title,
  chunk_text,
  VEC_DISTANCE_COSINE(embedding, @query_vec) AS similarity
FROM knowledge_base
WHERE category = 'Database'  -- Filtre SQL
  AND created_at >= DATE_SUB(NOW(), INTERVAL 6 MONTH)  -- Documents récents
ORDER BY similarity ASC
LIMIT 10;

-- Recherche multi-critères
SELECT 
  doc_id,
  title,
  category,
  chunk_text,
  VEC_DISTANCE_COSINE(embedding, @query_vec) AS vec_similarity,
  MATCH(chunk_text) AGAINST('optimization' IN BOOLEAN MODE) AS text_score
FROM knowledge_base
WHERE 
  category IN ('Database', 'Performance')
  AND (
    -- Texte contient mot-clé OU sémantiquement similaire
    MATCH(chunk_text) AGAINST('optimization index' IN BOOLEAN MODE)
    OR VEC_DISTANCE_COSINE(embedding, @query_vec) < 0.3
  )
ORDER BY 
  vec_similarity ASC,
  text_score DESC
LIMIT 20;

-- Agrégation sur résultats
SELECT 
  category,
  COUNT(*) AS relevant_docs,
  AVG(VEC_DISTANCE_COSINE(embedding, @query_vec)) AS avg_similarity
FROM knowledge_base
GROUP BY category
HAVING avg_similarity < 0.5
ORDER BY avg_similarity ASC;
```

---

## Cas d'Usage Concrets

### 1. Chatbot Support Client

**Architecture** :
```
Client Question → Embedding → MariaDB Vector → Top-3 FAQ similaires → LLM → Réponse
```

**Implémentation** :
```sql
CREATE TABLE faq (
  faq_id INT PRIMARY KEY AUTO_INCREMENT,
  question TEXT,
  answer TEXT,
  category VARCHAR(50),
  language VARCHAR(5) DEFAULT 'fr',
  view_count INT DEFAULT 0,
  helpful_count INT DEFAULT 0,
  embedding VECTOR(1536),
  
  INDEX idx_category (category),
  INDEX idx_embedding ON (embedding) USING HNSW
);

-- Indexer FAQ
INSERT INTO faq (question, answer, category, embedding) VALUES (
  'Comment réinitialiser mon mot de passe ?',
  'Pour réinitialiser votre mot de passe : 1. Cliquez sur "Mot de passe oublié"...',
  'Account',
  '[0.12, -0.34, ...]'  -- Embedding de la question
);

-- Recherche question similaire
SET @user_question_vec = get_embedding('J''ai oublié mon mot de passe');

SELECT 
  faq_id,
  question,
  answer,
  VEC_DISTANCE_COSINE(embedding, @user_question_vec) AS similarity
FROM faq
WHERE language = 'fr'
ORDER BY similarity ASC
LIMIT 3;

-- Mettre à jour statistiques
UPDATE faq 
SET view_count = view_count + 1 
WHERE faq_id = 123;
```

### 2. Recommandation de Produits

**Principe** : Similarité entre embeddings de descriptions produits.

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(255),
  description TEXT,
  category VARCHAR(100),
  price DECIMAL(10,2),
  
  -- Embedding de la description
  description_embedding VECTOR(1536),
  
  INDEX idx_category (category),
  INDEX idx_price (price),
  INDEX idx_emb ON (description_embedding) USING HNSW
);

-- Recommandation : Produits similaires
SELECT 
  p2.product_id,
  p2.name,
  p2.price,
  VEC_DISTANCE_COSINE(
    p2.description_embedding,
    p1.description_embedding
  ) AS similarity
FROM products p1
CROSS JOIN products p2
WHERE p1.product_id = 42  -- Produit de référence
  AND p2.product_id != p1.product_id
  AND p2.category = p1.category  -- Même catégorie
  AND ABS(p2.price - p1.price) < 100  -- Prix similaire
ORDER BY similarity ASC
LIMIT 5;
```

### 3. Recherche d'Images par Similarité

**Principe** : Embeddings d'images (CLIP, ResNet).

```sql
CREATE TABLE images (
  image_id INT PRIMARY KEY AUTO_INCREMENT,
  filename VARCHAR(255),
  url VARCHAR(500),
  description TEXT,
  
  -- Embedding visuel (CLIP ViT-L/14)
  visual_embedding VECTOR(768),
  
  -- Tags
  tags JSON,
  
  uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX idx_emb ON (visual_embedding) USING HNSW
);

-- Recherche images visuellement similaires
SET @query_image_vec = get_clip_embedding('path/to/query.jpg');

SELECT 
  image_id,
  filename,
  description,
  VEC_DISTANCE_COSINE(visual_embedding, @query_image_vec) AS similarity
FROM images
ORDER BY similarity ASC
LIMIT 10;

-- Recherche cross-modale : Texte → Images similaires
SET @text_query_vec = get_clip_text_embedding('sunset over mountains');

SELECT 
  image_id,
  filename,
  VEC_DISTANCE_COSINE(visual_embedding, @text_query_vec) AS similarity
FROM images
ORDER BY similarity ASC
LIMIT 10;
```

### 4. Détection de Contenu Dupliqué

**Principe** : Documents avec embeddings très proches = potentiellement dupliqués.

```sql
-- Trouver documents similaires (potentiel plagiat)
SELECT 
  d1.doc_id AS doc1_id,
  d1.title AS doc1_title,
  d2.doc_id AS doc2_id,
  d2.title AS doc2_title,
  VEC_DISTANCE_COSINE(d1.embedding, d2.embedding) AS similarity
FROM documents d1
CROSS JOIN documents d2
WHERE d1.doc_id < d2.doc_id  -- Éviter doublons (A,B) = (B,A)
  AND VEC_DISTANCE_COSINE(d1.embedding, d2.embedding) < 0.05  -- Très similaire
ORDER BY similarity ASC;

-- Résultat :
-- doc1_id | doc1_title        | doc2_id | doc2_title         | similarity
--    123  | "Article X"       |   456   | "Article X (copy)" | 0.001
--    789  | "Tutorial Python" |   234   | "Python Guide"     | 0.042
```

---

## Performance et Optimisations

### Optimisations SIMD

**SIMD** (Single Instruction Multiple Data) : Instructions CPU parallèles pour calculs vectoriels.

**Support MariaDB Vector** :
- ✅ **AVX2** (256-bit) : CPU Intel/AMD depuis 2013
- ✅ **AVX-512** (512-bit) : CPU Intel Skylake-X+, AMD Zen 4+
- ✅ **ARM NEON** : Processeurs ARM

**Vérification support** :
```bash
# Linux : Vérifier flags CPU
cat /proc/cpuinfo | grep -E 'avx2|avx512'
# Si présent → SIMD activé automatiquement

# MariaDB : Vérifier optimisations
SHOW VARIABLES LIKE '%vector%';
-- vector_simd_enabled: ON
```

**Impact performance** :

| Opération | Sans SIMD | AVX2 | AVX-512 | Gain |
|-----------|-----------|------|---------|------|
| Distance cosine (1536d) | 12 μs | 3 μs | 1.5 μs | **8x** |
| Distance euclidean | 15 μs | 4 μs | 2 μs | **7.5x** |
| Dot product | 8 μs | 2 μs | 1 μs | **8x** |

### Tuning Index HNSW

**Paramètres à ajuster** :

```sql
-- M : Nombre de connexions par nœud
-- Trade-off : Précision vs Mémoire

-- M=8 : Rapide, compact (recommandé si < 100K vecteurs)
CREATE INDEX idx_emb ON docs(embedding) USING HNSW WITH (M = 8);

-- M=16 : Équilibré (défaut, recommandé général)
CREATE INDEX idx_emb ON docs(embedding) USING HNSW WITH (M = 16);

-- M=32 : Haute précision (recommandé si > 1M vecteurs)
CREATE INDEX idx_emb ON docs(embedding) USING HNSW WITH (M = 32);

-- ef_construction : Qualité construction index
-- Plus élevé = meilleur index, mais construction plus lente

-- ef_construction=100 : Rapide (dev/test)
CREATE INDEX idx_emb ON docs(embedding) USING HNSW 
  WITH (ef_construction = 100);

-- ef_construction=200 : Équilibré (défaut, prod)
CREATE INDEX idx_emb ON docs(embedding) USING HNSW 
  WITH (ef_construction = 200);

-- ef_construction=500 : Haute qualité (critique)
CREATE INDEX idx_emb ON docs(embedding) USING HNSW 
  WITH (ef_construction = 500);
```

**Recommandations** :

| Taille Dataset | M | ef_construction | Taille Index | Recall@10 |
|----------------|---|-----------------|--------------|-----------|
| < 10K | 8 | 100 | Petit | 95% |
| 10K - 100K | 16 | 200 | Moyen | 98% |
| 100K - 1M | 16 | 300 | Grand | 99% |
| > 1M | 32 | 400 | Très grand | 99.5% |

**Recall@10** : Probabilité que les 10 meilleurs vrais voisins soient dans les 10 résultats de l'index.

### Benchmark Comparatif

**Configuration test** :
- Dataset : 1M vecteurs 1536 dimensions
- Hardware : 16 cores, 64 GB RAM, SSD NVMe
- Modèle : OpenAI text-embedding-3-small

**Résultats** :

| Méthode | Temps Recherche | Recall@10 | Mémoire Index |
|---------|----------------|-----------|---------------|
| **Scan Complet** | 2,500 ms | 100% | 0 MB (pas d'index) |
| **HNSW M=8** | 12 ms | 95% | 450 MB |
| **HNSW M=16** | 18 ms | 98% | 850 MB |
| **HNSW M=32** | 28 ms | 99.5% | 1,650 MB |

**Observations** :
- HNSW M=16 : **140x plus rapide** que scan complet
- Trade-off acceptable : -2% recall pour 850 MB mémoire
- Pour applications critiques : M=32 (-0.5% recall)

---

## Comparaison avec Bases Vectorielles Spécialisées

### MariaDB Vector vs Pinecone/Weaviate/Qdrant

| Aspect | MariaDB Vector | Pinecone | Weaviate | Qdrant |
|--------|----------------|----------|----------|--------|
| **Type** | SQL + Vector | Vector pure | GraphQL + Vector | REST + Vector |
| **Déploiement** | Self-hosted | Cloud (SaaS) | Self-hosted | Self-hosted |
| **Coût** | ✅ Gratuit | $$$ (usage-based) | Gratuit/Entreprise | Gratuit |
| **Indexation** | HNSW | Propriétaire | HNSW | HNSW |
| **Dimensions max** | 65,535 | 20,000 | 65,536 | 65,536 |
| **Transactions** | ✅ ACID | ❌ Non | ⚠️ Limité | ❌ Non |
| **SQL natif** | ✅ Complet | ❌ Non | ❌ Non | ❌ Non |
| **Joins** | ✅ Oui | ❌ Non | ⚠️ GraphQL | ❌ Non |
| **Filtrage** | ✅ SQL WHERE | ⚠️ Metadata | ✅ GraphQL | ✅ Payload |
| **Performance** | Très bon | Excellent | Excellent | Excellent |
| **Scalabilité** | Horizontale | Auto | Manuelle | Manuelle |
| **Backup** | ✅ Standard SQL | ✅ Automatique | ⚠️ Manuel | ⚠️ Manuel |
| **Encryption** | ✅ At-rest | ✅ Transit | ✅ Both | ✅ Both |

### Quand Choisir MariaDB Vector ?

**✅ Utiliser MariaDB Vector si** :
- Données métier + vecteurs dans même système
- Besoin de transactions ACID
- Requêtes hybrides SQL + similarité
- Infrastructure existante MariaDB
- Budget limité (pas de SaaS)
- < 10M vecteurs (performant jusqu'à cette échelle)

**❌ Utiliser base vectorielle spécialisée si** :
- > 100M vecteurs (scalabilité extrême)
- Latence < 1 ms critique (edge cases)
- Équipe dédiée ML/Vector ops
- Budget permet SaaS géré (Pinecone)
- Pas de données métier relationnelles

**🔀 Approche hybride** :
```
MariaDB (données métier + vecteurs récents/actifs)
    +
Pinecone/Weaviate (archive vecteurs anciens/massifs)
```

---

## Best Practices

### 1. Design de Schéma

```sql
-- ✅ Bonnes pratiques

CREATE TABLE documents (
  -- ID numérique (performant)
  doc_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  
  -- Métadonnées filtrables (index classiques)
  category VARCHAR(100),
  language VARCHAR(5),
  created_at TIMESTAMP,
  INDEX idx_cat_lang (category, language),
  INDEX idx_created (created_at),
  
  -- Contenu textuel (FULLTEXT pour recherche hybride)
  title VARCHAR(500),
  content TEXT,
  FULLTEXT INDEX ft_content (title, content),
  
  -- Vecteur (dimensions fixes selon modèle)
  embedding VECTOR(1536) NOT NULL,
  INDEX idx_emb ON (embedding) USING HNSW WITH (M = 16),
  
  -- Normalisation : Séparer texte long si > 64KB
  -- Utiliser table liée si nécessaire
  
  -- Version embedding (pour re-indexation)
  embedding_model VARCHAR(50) DEFAULT 'text-embedding-3-small',
  embedding_version INT DEFAULT 1
);
```

### 2. Chunking Optimal

```python
def smart_chunk(text: str, max_tokens: int = 500, overlap: int = 50):
    """
    Découpage intelligent avec overlap
    
    Overlap : Contexte partagé entre chunks adjacents
    Améliore résultats pour phrases à cheval sur 2 chunks
    """
    import tiktoken
    
    enc = tiktoken.get_encoding("cl100k_base")
    tokens = enc.encode(text)
    
    chunks = []
    start = 0
    
    while start < len(tokens):
        end = start + max_tokens
        chunk_tokens = tokens[start:end]
        chunk_text = enc.decode(chunk_tokens)
        
        chunks.append({
            'text': chunk_text,
            'start_token': start,
            'end_token': end
        })
        
        start = end - overlap  # Overlap
    
    return chunks

# Longueur optimale par use case :
# - FAQ/Support : 200-500 tokens (questions courtes)
# - Documentation : 500-1000 tokens (paragraphes complets)
# - Articles : 1000-2000 tokens (sections logiques)
```

### 3. Gestion des Versions d'Embeddings

```sql
-- Problème : Modèle d'embedding évolue (v1 → v2)
-- Solution : Versioning + re-indexation progressive

ALTER TABLE documents
ADD COLUMN embedding_v2 VECTOR(1536),
ADD INDEX idx_emb_v2 ON (embedding_v2) USING HNSW;

-- Re-indexer progressivement (batch)
-- Script Python exécuté en background
UPDATE documents
SET 
  embedding_v2 = get_embedding_v2(content),
  embedding_version = 2
WHERE embedding_version = 1
  AND doc_id BETWEEN ? AND ?
LIMIT 1000;

-- Une fois complète, swap colonnes
ALTER TABLE documents
DROP COLUMN embedding,
CHANGE COLUMN embedding_v2 embedding VECTOR(1536);
```

### 4. Monitoring et Alertes

```sql
-- Métriques à surveiller

-- 1. Taille index HNSW
SELECT 
  table_name,
  index_name,
  ROUND(stat_value / 1024 / 1024, 2) AS index_size_mb
FROM information_schema.INNODB_SYS_TABLESTATS
WHERE index_name LIKE '%embedding%';

-- 2. Distribution similarité
SELECT 
  CASE 
    WHEN similarity < 0.1 THEN 'Très similaire'
    WHEN similarity < 0.3 THEN 'Similaire'
    WHEN similarity < 0.5 THEN 'Moyennement similaire'
    ELSE 'Peu similaire'
  END AS similarity_range,
  COUNT(*) AS count
FROM (
  SELECT VEC_DISTANCE_COSINE(embedding, @query_vec) AS similarity
  FROM documents
  LIMIT 10000
) AS distances
GROUP BY similarity_range;

-- 3. Performance requêtes
-- Utiliser Performance Schema
SELECT 
  DIGEST_TEXT,
  COUNT_STAR,
  AVG_TIMER_WAIT / 1000000000 AS avg_time_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%VEC_DISTANCE%'
ORDER BY avg_time_ms DESC
LIMIT 10;
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **Recherche vectorielle** : Similarité sémantique vs correspondance lexicale
- ✅ **Embeddings** : Représentation numérique texte/image (vecteurs haute dimension)
- ✅ **Type VECTOR** : Stockage natif vecteurs (1 à 65535 dimensions)
- ✅ **Index HNSW** : Recherche approximative k-NN rapide (O(log n))

### Fonctions de Distance
- ✅ **VEC_DISTANCE_COSINE** : Recommandé pour texte/sémantique (0-2)
- ✅ **VEC_DISTANCE_EUCLIDEAN** : Distance géométrique (0-∞)
- ✅ **VEC_DISTANCE_DOT** : Plus rapide si vecteurs normalisés
- 💡 **Choix** : Cosine par défaut, Dot si normalisé

### RAG (Retrieval Augmented Generation)
- ✅ **Workflow** : Document → Chunking → Embedding → Index → Recherche → LLM
- ✅ **Chunking** : 500-1000 tokens avec overlap 50-100
- ✅ **Recherche hybride** : SQL WHERE + similarité vectorielle
- ✅ **Top-K** : Récupérer 3-5 documents les plus pertinents

### Performance
- ✅ **SIMD** : AVX2/AVX-512 = 8x plus rapide
- ✅ **HNSW tuning** : M=16 (défaut), M=32 (haute précision)
- ✅ **Benchmark** : 18ms recherche sur 1M vecteurs (140x vs scan)
- ✅ **Recall@10** : 98% avec M=16

### Intégration
- ✅ **OpenAI** : text-embedding-3-small (1536d), text-embedding-3-large (3072d)
- ✅ **Hugging Face** : sentence-transformers (384-768d)
- ✅ **CLIP** : Multimodal texte-image (768d)
- ✅ **Local** : Ollama, llama.cpp (open source)

### Cas d'Usage
- ✅ **Chatbot/FAQ** : Recherche questions similaires
- ✅ **Base connaissances** : Documentation technique searchable
- ✅ **Recommandation** : Produits/contenus similaires
- ✅ **Détection duplicatas** : Plagiat, contenu similaire

### vs Bases Vectorielles Spécialisées
- ✅ **Avantages MariaDB** : ACID, SQL, Joins, Gratuit, Une seule base
- ⚠️ **Limites** : Scalabilité < Pinecone (performant jusqu'à 10M vecteurs)
- 💡 **Sweet spot** : Applications avec données métier + vecteurs

### Best Practices
- ✅ Versioning embeddings (embedding_version)
- ✅ Chunking intelligent avec overlap
- ✅ Index HNSW sur colonnes vectorielles
- ✅ Recherche hybride (SQL + similarité)
- ✅ Monitoring taille index et performance

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [MariaDB Vector](https://mariadb.com/kb/en/vector/) - Documentation complète
- 📖 [VECTOR Data Type](https://mariadb.com/kb/en/vector-data-type/)
- 📖 [HNSW Index](https://mariadb.com/kb/en/vector-indexes/)
- 📖 [Vector Functions](https://mariadb.com/kb/en/vector-functions/)

### Embeddings et Modèles
- 🤖 [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings) - API officielle
- 🤗 [Hugging Face Models](https://huggingface.co/models?pipeline_tag=sentence-similarity)
- 📝 [Sentence Transformers](https://www.sbert.net/) - Modèles open source

### RAG et Applications
- 📚 [RAG Architecture](https://arxiv.org/abs/2005.11401) - Paper original
- 🔧 [LangChain](https://python.langchain.com/) - Framework RAG
- 🦜 [LlamaIndex](https://www.llamaindex.ai/) - Alternative RAG

### Algorithmes et Performance
- 📄 [HNSW Paper](https://arxiv.org/abs/1603.09320) - Algorithme original
- 📊 [Vector Search Benchmarks](https://github.com/erikbern/ann-benchmarks)

### Comparaisons
- 🔄 [Pinecone](https://www.pinecone.io/) - Base vectorielle SaaS
- 🔄 [Weaviate](https://weaviate.io/) - Base vectorielle open source
- 🔄 [Qdrant](https://qdrant.tech/) - Alternative Rust

---

## ➡️ Sous-sections suivantes

### **18.10.1 Type de Données VECTOR**
Détails du type VECTOR, stockage interne, limites de dimensions, et conversions.

### **18.10.2 Index HNSW**
Architecture détaillée HNSW, paramètres avancés (M, ef_construction, ef_search), tuning performance.

### **18.10.3 Fonctions de Distance**
Formules mathématiques, cas d'usage par fonction, normalisation, optimisations SIMD.

### **18.10.4 Intégration avec LLMs**
Exemples complets OpenAI, Hugging Face, Ollama, gestion tokens, coûts, rate limits.

---


⏭️ [Type de données VECTOR](/18-fonctionnalites-avancees/10.1-type-donnees-vector.md)

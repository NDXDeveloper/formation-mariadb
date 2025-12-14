🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.2 MariaDB Vector : La Fonctionnalité Phare 🆕

> **Niveau** : Tous niveaux (Veille technologique)  
> **Durée estimée** : 25-30 minutes  
> **Prérequis** : Compréhension basique de l'IA et des embeddings (recommandé mais pas obligatoire)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre pourquoi MariaDB Vector est une innovation majeure
- Identifier les cas d'usage concrets de la recherche vectorielle
- Évaluer les avantages de MariaDB Vector vs solutions concurrentes
- Concevoir une architecture IA unifiée avec MariaDB
- Prendre une décision éclairée sur l'adoption de MariaDB Vector
- Planifier l'intégration dans vos applications existantes

---

## Introduction

**MariaDB Vector** est **LA** fonctionnalité révolutionnaire de MariaDB 11.8 LTS. Elle transforme MariaDB d'un simple SGBD relationnel en une **plateforme complète pour l'IA moderne**, capable de gérer simultanément :

- 📊 **Données structurées** (tables relationnelles classiques)
- 🧠 **Embeddings vectoriels** (représentations numériques de concepts)
- 🔍 **Recherche sémantique** (similarité de sens, pas seulement de mots-clés)
- 🤖 **Applications RAG** (Retrieval-Augmented Generation pour LLM)

Cette innovation arrive au **moment parfait** : l'explosion de l'IA générative (ChatGPT, Claude, LLaMA) crée un besoin massif de bases de données vectorielles performantes.

---

## 🌟 Pourquoi MariaDB Vector est Révolutionnaire

### Le contexte : L'explosion de l'IA générative

Depuis 2023, l'IA générative a transformé le paysage technologique :

```
Évolution du marché IA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2020 │                    ▄
     │               ▄▄▄▄▀
2021 │          ▄▄▄▀▀
     │      ▄▄▀▀
2022 │  ▄▄▀▀              ChatGPT
     │▄▀                  lance la révolution
2023 │████████████▄
     │            ▀▀▀▄▄▄
2024 │                  ▀▀▄▄▄
     │                      ▀▀▄▄   
2025 │                          ▀▀▄▄  MariaDB Vector
     └──────────────────────────────────────────
     Applications IA : 12M → 180M+ (×15)
```

**Le problème** : Les LLM (Large Language Models) ont une limite fondamentale :
- ❌ Connaissances figées au moment de l'entraînement
- ❌ Pas d'accès aux données internes de l'entreprise
- ❌ Hallucinations sur des faits spécifiques
- ❌ Coût prohibitif pour ré-entraîner constamment

**La solution** : **RAG** (Retrieval-Augmented Generation)

```
┌─────────────────────────────────────────────────────────┐
│                    Architecture RAG                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Question utilisateur                                │
│     "Quels sont nos produits bio en stock ?"            │
│           │                                             │
│           ▼                                             │
│  2. Transformation en embedding (vecteur)               │
│     [0.234, -0.891, 0.456, ..., 0.123]  (1536 dims)     │
│           │                                             │
│           ▼                                             │
│  3. Recherche vectorielle dans MariaDB                  │
│     SELECT * FROM products                              │
│     ORDER BY VEC_DISTANCE_COSINE(embedding, @query)     │
│     LIMIT 5;                                            │
│           │                                             │
│           ▼                                             │
│  4. Contexte récupéré                                   │
│     - Produit A : Lait bio 1L, stock: 245               │
│     - Produit B : Yaourt bio, stock: 89                 │
│     - Produit C : Fromage bio, stock: 34                │
│           │                                             │
│           ▼                                             │
│  5. Augmentation du prompt LLM                          │
│     "Contexte: [produits bio...] Question: ..."         │
│           │                                             │
│           ▼                                             │
│  6. Réponse LLM enrichie et factuelle                   │
│     "Nous avons 3 produits bio en stock: ..."           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Avant MariaDB 11.8 : Architecture complexe et coûteuse

```
┌─────────────────────────────────────────────────────────┐
│        Architecture classique (Sans MariaDB Vector)     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐                                       │
│  │ Application  │                                       │
│  └──────┬───────┘                                       │
│         │                                               │
│    ┌────┴────┐                                          │
│    │         │                                          │
│    ▼         ▼                                          │
│ ┌─────┐  ┌──────────┐        Problèmes:                 │
│ │MySQL│  │ Pinecone │        - 2 systèmes à maintenir   │
│ │     │  │   ou     │        - Cohérence difficile      │
│ │Rela │  │Weaviate  │        - Coûts doublés            │
│ │tions│  │   ou     │        - Latency réseau           │
│ │     │  │Qdrant    │        - Sécurité complexe        │
│ └─────┘  └──────────┘        - 2 backups à gérer        │
│                                                         │
│  Données      Vecteurs                                  │
│ structurées   embeddings                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Avec MariaDB 11.8 : Architecture unifiée et simple

```
┌─────────────────────────────────────────────────────────┐
│       Architecture moderne (Avec MariaDB Vector)        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐                                       │
│  │ Application  │                                       │
│  └──────┬───────┘                                       │
│         │                                               │
│         ▼                                               │
│  ┌─────────────────────┐    Avantages:                  │
│  │   MariaDB 11.8      │    ✅ 1 seul système           │
│  │   ┌─────┬────────┐  │    ✅ Cohérence ACID           │
│  │   │Rela │ Vector │  │    ✅ Coûts réduits            │
│  │   │tions│ HNSW   │  │    ✅ Latency minimale         │
│  │   │     │ Index  │  │    ✅ Sécurité unifiée         │
│  │   └─────┴────────┘  │    ✅ 1 backup                 │
│  └─────────────────────┘    ✅ SQL standard             │
│                                                         │
│  Données structurées + Vecteurs dans la même base       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

💡 **Impact business** : Réduction de 40-60% des coûts d'infrastructure IA, simplification de 70% de l'architecture.

---

## 🧩 Les Composants de MariaDB Vector

MariaDB Vector n'est pas une simple fonctionnalité, c'est un **écosystème complet** comprenant :

### 1. Type de données VECTOR 📦

```sql
-- Déclaration d'une colonne vectorielle
CREATE TABLE documents (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    content TEXT,
    embedding VECTOR(1536)  -- Vecteur de 1536 dimensions
);

-- Dimensions supportées : 1 à 65,535
-- Exemples courants:
-- - OpenAI text-embedding-ada-002 : 1536 dimensions
-- - OpenAI text-embedding-3-small : 1536 dimensions
-- - OpenAI text-embedding-3-large : 3072 dimensions
-- - Cohere embed-multilingual-v3 : 1024 dimensions
-- - Google PaLM embeddings : 768 dimensions
```

**Caractéristiques** :
- ✅ Stockage efficace (format binaire compact)
- ✅ Support jusqu'à 65,535 dimensions
- ✅ Validation automatique de la dimensionnalité
- ✅ Compatibilité avec tous les moteurs de stockage InnoDB/Aria

### 2. Index HNSW (Hierarchical Navigable Small Worlds) 🔍

L'algorithme **HNSW** est l'état de l'art pour la recherche de voisins les plus proches (ANN - Approximate Nearest Neighbors).

```sql
-- Création d'un index HNSW
CREATE INDEX idx_embedding ON documents(embedding) 
    USING HNSW 
    WITH (
        M = 16,                  -- Connexions par nœud (4-64, défaut: 16)
        ef_construction = 200,   -- Précision construction (100-1000, défaut: 200)
        metric = 'cosine'        -- Métrique: cosine, euclidean, dotproduct
    );
```

**Principe de fonctionnement** :

```
Index HNSW - Structure multi-niveaux
═════════════════════════════════════

Niveau 3 (sparse)    ○────────────○
                     │            │
Niveau 2             ○──○────○────○──○
                     │  │    │    │  │
Niveau 1         ○───○──○──○─○─○──○──○───○
                 │   │  │  │ │ │  │  │   │
Niveau 0     ○───○─○─○─○○─○○○○○○─○○─○─○─○─○
(complet)    └────────────────────────────────┘
             Vecteurs stockés

Recherche:
1. Entrer au niveau le plus haut
2. Naviguer vers le voisin le plus proche
3. Descendre au niveau suivant
4. Répéter jusqu'au niveau 0
5. Affiner localement

Complexité: O(log N) au lieu de O(N) (scan complet)
```

**Paramètres de tuning** :

| Paramètre | Valeur | Impact | Recommandation |
|-----------|--------|--------|----------------|
| **M** | 4-64 | Connexions/nœud | 16 (défaut), 32 pour haute précision |
| **ef_construction** | 100-1000 | Précision index | 200 (défaut), 400+ pour datasets larges |
| **metric** | cosine/euclidean/dot | Calcul distance | cosine (recommandé pour texte) |

### 3. Fonctions de Distance 📐

MariaDB Vector fournit **3 métriques de distance** :

```sql
-- 1. Distance Cosinus (la plus courante pour le texte)
-- Mesure l'angle entre vecteurs (insensible à la magnitude)
-- Plage : 0 (identique) à 2 (opposé)
SELECT VEC_DISTANCE_COSINE(
    embedding, 
    '[0.1, 0.2, 0.3, ..., 0.5]'::VECTOR
) AS cosine_similarity
FROM documents;

-- 2. Distance Euclidienne (géométrie classique)
-- Mesure la distance "à vol d'oiseau"
-- Plage : 0 (identique) à +∞
SELECT VEC_DISTANCE_EUCLIDEAN(
    embedding, 
    @query_vector
) AS euclidean_distance
FROM documents;

-- 3. Produit Scalaire (Dot Product)
-- Mesure l'alignement directionnel
-- Plage : -∞ à +∞
SELECT VEC_DISTANCE_DOT(
    embedding,
    @query_vector
) AS dot_product
FROM documents;
```

**Quelle métrique choisir ?**

| Métrique | Cas d'usage | Avantages | Inconvénients |
|----------|-------------|-----------|---------------|
| **Cosine** | Texte, NLP, sémantique | Insensible magnitude, normalisé | Calcul légèrement plus coûteux |
| **Euclidean** | Images, géométrie, clustering | Intuitive, rapide | Sensible à la magnitude |
| **Dot Product** | Recommendations, scoring | Très rapide | Requiert vecteurs normalisés |

💡 **Recommandation** : **Cosine** pour 95% des cas d'usage textuels/sémantiques.

### 4. Fonctions de Conversion 🔄

```sql
-- Conversion texte → vecteur
SELECT VEC_FromText('[0.234, -0.891, 0.456, 0.123]') AS vec;

-- Conversion vecteur → texte
SELECT VEC_ToText(embedding) FROM documents WHERE id = 1;

-- Dimension d'un vecteur
SELECT VEC_Dimensions(embedding) FROM documents WHERE id = 1;
-- Résultat: 1536

-- Normalisation (utile pour dot product)
SELECT VEC_Normalize(embedding) FROM documents;
```

### 5. Optimisations SIMD ⚡

MariaDB Vector exploite les **instructions SIMD** (Single Instruction, Multiple Data) du CPU pour des calculs vectoriels ultra-rapides :

| Architecture CPU | Instructions SIMD | Performance | Gain vs Scalar |
|------------------|-------------------|-------------|----------------|
| Intel/AMD x86-64 | **AVX2** | Baseline | 4-8x |
| Intel/AMD moderne | **AVX-512** | Haute | 8-16x |
| ARM (Apple M-series, AWS Graviton) | **NEON** | Haute | 4-8x |
| IBM Power10 | **MMA** | Très haute | 10-20x |

```
Benchmark: Calcul distance cosinus sur 1M vecteurs (1536D)
═══════════════════════════════════════════════════════════

Sans SIMD (scalar):    ████████████████████ 2450 ms

AVX2 (Intel/AMD):      ████ 320 ms  (×7.6 faster)

AVX-512 (Intel):       ██ 155 ms    (×15.8 faster)

ARM NEON (Apple M2):   ███ 280 ms   (×8.8 faster)
```

💡 **Conseil** : MariaDB détecte automatiquement les instructions disponibles. Aucune configuration manuelle requise.

---

## 🎯 Cas d'Usage Concrets

### 1. Recherche Sémantique (Semantic Search) 🔍

**Problème** : Recherche par mots-clés classique manque les synonymes et le contexte.

```sql
-- Recherche classique (limitée)
SELECT * FROM articles 
WHERE title LIKE '%intelligence%' 
   OR content LIKE '%intelligence%';
-- ❌ Ne trouve pas "IA", "smart", "cognitif", etc.

-- Recherche sémantique (puissante)
SELECT title, 
       VEC_DISTANCE_COSINE(embedding, @query_embedding) AS relevance
FROM articles
ORDER BY relevance
LIMIT 10;
-- ✅ Trouve tous les articles sur le concept d'intelligence
--    même sans le mot exact
```

**Exemple réel : Support client**

```sql
-- Base de connaissances vectorisée
CREATE TABLE kb_articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    category VARCHAR(100),
    embedding VECTOR(1536),
    INDEX idx_emb (embedding) USING HNSW
);

-- Requête utilisateur: "Mon mot de passe ne fonctionne pas"
-- Embedding généré par OpenAI API
SET @user_query_emb = '[0.234, -0.891, ..., 0.123]'::VECTOR;

-- Recherche des articles les plus pertinents
SELECT 
    title,
    category,
    VEC_DISTANCE_COSINE(embedding, @user_query_emb) AS relevance
FROM kb_articles
ORDER BY relevance
LIMIT 5;

-- Résultats:
-- 1. "Réinitialiser votre mot de passe" (relevance: 0.02)
-- 2. "Problèmes de connexion" (relevance: 0.15)
-- 3. "Sécuriser votre compte" (relevance: 0.23)
-- 4. "Authentification à deux facteurs" (relevance: 0.28)
-- 5. "Gérer vos identifiants" (relevance: 0.31)
```

### 2. RAG (Retrieval-Augmented Generation) 🤖

**Architecture complète RAG avec MariaDB** :

```sql
-- 1. Base de documents d'entreprise
CREATE TABLE company_docs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    doc_type ENUM('policy', 'procedure', 'manual', 'faq'),
    title VARCHAR(500),
    content TEXT,
    department VARCHAR(100),
    last_updated TIMESTAMP,
    embedding VECTOR(1536),
    INDEX idx_embedding (embedding) USING HNSW
);

-- 2. Fonction de recherche contextuelle
DELIMITER //
CREATE PROCEDURE get_rag_context(
    IN query_text VARCHAR(1000),
    IN query_embedding VECTOR(1536),
    IN top_k INT
)
BEGIN
    -- Recherche hybride : vecteur + filtres SQL
    SELECT 
        id,
        title,
        content,
        doc_type,
        department,
        VEC_DISTANCE_COSINE(embedding, query_embedding) AS relevance
    FROM company_docs
    WHERE last_updated >= DATE_SUB(NOW(), INTERVAL 2 YEAR)  -- Docs récents
      AND (
          -- Recherche vectorielle
          embedding IS NOT NULL
      )
    ORDER BY relevance
    LIMIT top_k;
END //
DELIMITER ;

-- 3. Application Python avec LangChain
```

```python
from langchain.llms import OpenAI
from langchain.embeddings import OpenAIEmbeddings
import mariadb

# Connexion MariaDB
conn = mariadb.connect(
    host="localhost",
    user="app",
    password="secure_pwd",
    database="rag_db"
)
cursor = conn.cursor()

# Initialisation embeddings
embeddings = OpenAIEmbeddings(model="text-embedding-ada-002")
llm = OpenAI(model="gpt-4")

def rag_query(user_question: str) -> str:
    # 1. Générer embedding de la question
    query_emb = embeddings.embed_query(user_question)
    
    # 2. Récupérer contexte depuis MariaDB
    cursor.execute("""
        CALL get_rag_context(%s, %s, 5)
    """, (user_question, str(query_emb)))
    
    docs = cursor.fetchall()
    context = "\n\n".join([doc[2] for doc in docs])  # content
    
    # 3. Construire prompt enrichi
    prompt = f"""
    Contexte: {context}
    
    Question: {user_question}
    
    Réponds en te basant uniquement sur le contexte fourni.
    """
    
    # 4. Générer réponse avec LLM
    response = llm(prompt)
    return response

# Exemple d'utilisation
answer = rag_query("Quelle est notre politique de télétravail ?")
print(answer)
# "D'après notre politique RH mise à jour en 2024, 
#  le télétravail est autorisé jusqu'à 3 jours par semaine..."
```

### 3. Système de Recommandation 🎁

```sql
-- E-commerce : Recommandations produits
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2),
    image_url VARCHAR(500),
    embedding VECTOR(1536),  -- Embedding de description + attributs
    INDEX idx_emb (embedding) USING HNSW
);

-- Historique utilisateur
CREATE TABLE user_interactions (
    user_id INT,
    product_id INT,
    interaction_type ENUM('view', 'cart', 'purchase'),
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, product_id, created_at)
);

-- Génération du profil utilisateur (vecteur moyen)
CREATE VIEW user_profile_vector AS
SELECT 
    ui.user_id,
    -- Moyenne pondérée des embeddings des produits interagis
    AVG(p.embedding) AS profile_embedding
FROM user_interactions ui
JOIN products p ON ui.product_id = p.id
WHERE ui.interaction_type IN ('cart', 'purchase')
  AND ui.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY ui.user_id;

-- Recommandations personnalisées
SELECT 
    p.name,
    p.category,
    p.price,
    VEC_DISTANCE_COSINE(p.embedding, upv.profile_embedding) AS relevance
FROM products p
CROSS JOIN user_profile_vector upv
WHERE upv.user_id = 12345
  AND p.id NOT IN (
      -- Exclure produits déjà achetés
      SELECT product_id FROM user_interactions
      WHERE user_id = 12345 AND interaction_type = 'purchase'
  )
ORDER BY relevance
LIMIT 20;
```

### 4. Détection d'Anomalies 🚨

```sql
-- Monitoring de logs applicatifs
CREATE TABLE app_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    service_name VARCHAR(100),
    log_level ENUM('INFO', 'WARN', 'ERROR'),
    message TEXT,
    embedding VECTOR(768),  -- Embedding du message
    is_anomaly BOOLEAN DEFAULT FALSE,
    INDEX idx_emb (embedding) USING HNSW
);

-- Détection d'anomalies (messages éloignés du pattern normal)
WITH normal_pattern AS (
    SELECT AVG(embedding) AS avg_embedding
    FROM app_logs
    WHERE log_level = 'INFO'
      AND timestamp >= DATE_SUB(NOW(), INTERVAL 7 DAY)
)
SELECT 
    al.timestamp,
    al.service_name,
    al.message,
    VEC_DISTANCE_COSINE(al.embedding, np.avg_embedding) AS anomaly_score
FROM app_logs al
CROSS JOIN normal_pattern np
WHERE al.timestamp >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
HAVING anomaly_score > 0.5  -- Seuil d'anomalie
ORDER BY anomaly_score DESC;
```

### 5. Recherche Visuelle (Image Similarity) 🖼️

```sql
-- Catalogue de produits avec embeddings d'images
CREATE TABLE product_images (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT,
    image_url VARCHAR(500),
    image_embedding VECTOR(512),  -- Embedding vision (CLIP, ResNet)
    INDEX idx_img_emb (image_embedding) USING HNSW
);

-- Recherche "Trouver des produits similaires visuellement"
SELECT 
    pi.product_id,
    pi.image_url,
    VEC_DISTANCE_COSINE(pi.image_embedding, @query_image_emb) AS visual_similarity
FROM product_images pi
ORDER BY visual_similarity
LIMIT 12;

-- Cas d'usage : "Upload une photo, trouve des produits similaires"
```

---

## 📊 Comparaison avec les Alternatives

### MariaDB Vector vs Bases Vectorielles Spécialisées

| Critère | MariaDB Vector | Pinecone | Weaviate | Qdrant | Milvus |
|---------|----------------|----------|----------|--------|--------|
| **Type** | Hybride SQL+Vector | Vector pur (SaaS) | Vector pur | Vector pur | Vector pur |
| **Recherche vectorielle** | ✅ HNSW | ✅ Propriétaire | ✅ HNSW | ✅ HNSW | ✅ Multiple |
| **Recherche SQL** | ✅ Full SQL | ❌ Limitée | 🟡 GraphQL | ❌ JSON | ❌ JSON |
| **Transactions ACID** | ✅ Full | ❌ | ❌ | ❌ | ❌ |
| **Jointures SQL** | ✅ Oui | ❌ | ❌ | ❌ | ❌ |
| **Coût (1M vecteurs)** | ~$50/mois (self-hosted) | ~$200/mois | ~$100/mois | ~$75/mois | ~$60/mois |
| **Latency P99** | 12ms | 8ms | 18ms | 10ms | 15ms |
| **Complexité infra** | 🟢 Faible | 🟢 Très faible (SaaS) | 🟡 Moyenne | 🟡 Moyenne | 🔴 Élevée |
| **Vendor lock-in** | ✅ Aucun | 🔴 Élevé | 🟡 Moyen | 🟢 Faible | 🟢 Faible |
| **Communauté** | ⭐⭐⭐⭐⭐ Énorme | ⭐⭐⭐ Moyenne | ⭐⭐⭐⭐ Grande | ⭐⭐⭐ Moyenne | ⭐⭐⭐⭐ Grande |
| **Maturité** | 🆕 Nouvelle (2025) | ⭐⭐⭐⭐ Mature | ⭐⭐⭐⭐ Mature | ⭐⭐⭐ Bonne | ⭐⭐⭐⭐ Mature |

### MariaDB Vector vs PostgreSQL pgvector

| Critère | MariaDB Vector | PostgreSQL pgvector |
|---------|----------------|---------------------|
| **Algorithme** | HNSW | IVFFlat + HNSW (depuis pg14) |
| **Performance** | ⚡⚡⚡⚡ Très bonne | ⚡⚡⚡ Bonne |
| **Optimisations SIMD** | AVX2/512, NEON, Power10 | AVX2 uniquement |
| **Index** | HNSW natif | IVFFlat, HNSW (extension) |
| **Dimensions max** | 65,535 | 16,000 |
| **Setup** | Built-in (11.8+) | Extension à installer |
| **Écosystème** | MariaDB/MySQL | PostgreSQL |
| **Licence** | GPL v2 | PostgreSQL License |

💡 **Verdict** : 
- **MariaDB Vector** : Meilleur choix si vous êtes déjà sur MySQL/MariaDB
- **PostgreSQL pgvector** : Meilleur choix si vous êtes sur PostgreSQL
- **Pinecone/Weaviate** : Meilleur si vous voulez SaaS managé sans infra

---

## 🏗️ Architecture d'Intégration

### Pattern 1 : Hybrid Search (Vecteur + SQL)

```sql
-- Recherche hybride : Similarité vectorielle + Filtres métier
SELECT 
    p.id,
    p.name,
    p.category,
    p.price,
    p.stock_quantity,
    VEC_DISTANCE_COSINE(p.embedding, @query_emb) AS relevance
FROM products p
WHERE 
    -- Filtres SQL classiques
    p.price BETWEEN 10 AND 100
    AND p.stock_quantity > 0
    AND p.category IN ('Electronics', 'Home')
    AND p.created_at >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
    -- Pré-filtrage pour performance
ORDER BY relevance
LIMIT 20;

-- Impossible avec bases vectorielles pures !
```

### Pattern 2 : Enrichissement Progressif

```sql
-- Table existante
CREATE TABLE articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    author VARCHAR(100),
    published_at TIMESTAMP
);

-- Ajout colonne vectorielle (non-bloquant)
ALTER TABLE articles 
ADD COLUMN embedding VECTOR(1536) DEFAULT NULL;

-- Ajout index HNSW
CREATE INDEX idx_emb ON articles(embedding) USING HNSW;

-- Population progressive via batch
UPDATE articles 
SET embedding = generate_embedding(content)  -- Fonction custom
WHERE embedding IS NULL
LIMIT 1000;

-- Application fonctionne en mode dégradé pendant migration
SELECT * FROM articles
WHERE embedding IS NOT NULL  -- Seulement articles vectorisés
ORDER BY VEC_DISTANCE_COSINE(embedding, @query_emb);
```

### Pattern 3 : Multi-Modal Search

```sql
-- Table unifiée texte + image
CREATE TABLE media_library (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    description TEXT,
    media_type ENUM('image', 'video', 'document'),
    file_url VARCHAR(500),
    
    -- Embeddings multi-modaux
    text_embedding VECTOR(1536),    -- OpenAI ada-002
    image_embedding VECTOR(512),    -- CLIP ViT-B/32
    
    -- Index séparés
    INDEX idx_text_emb (text_embedding) USING HNSW,
    INDEX idx_img_emb (image_embedding) USING HNSW
);

-- Recherche par texte
SELECT * FROM media_library
ORDER BY VEC_DISTANCE_COSINE(text_embedding, @text_query_emb)
LIMIT 10;

-- Recherche par image
SELECT * FROM media_library
ORDER BY VEC_DISTANCE_COSINE(image_embedding, @image_query_emb)
LIMIT 10;

-- Recherche combinée (pondérée)
SELECT 
    id,
    title,
    (
        0.6 * VEC_DISTANCE_COSINE(text_embedding, @text_emb) +
        0.4 * VEC_DISTANCE_COSINE(image_embedding, @img_emb)
    ) AS combined_score
FROM media_library
ORDER BY combined_score
LIMIT 10;
```

---

## 🚀 Performance et Scalabilité

### Benchmarks Officiels

**Dataset** : 1 million vecteurs (1536 dimensions, OpenAI ada-002)  
**Hardware** : AWS c6i.4xlarge (16 vCPU, 32GB RAM)  
**Métrique** : Distance Cosinus

| Opération | MariaDB 11.8 HNSW | PostgreSQL pgvector | Pinecone (Serveless) |
|-----------|-------------------|---------------------|----------------------|
| **Indexation 1M vecteurs** | 45 min | 75 min | 12 min (managed) |
| **Query P50 latency** | 8ms | 25ms | 5ms |
| **Query P95 latency** | 15ms | 60ms | 12ms |
| **Query P99 latency** | 28ms | 120ms | 18ms |
| **Throughput (QPS)** | 2,500 | 800 | 5,000 |
| **Recall@10** | 0.975 | 0.95 | 0.98 |
| **Taille index** | 12 GB | 15 GB | N/A (managed) |

### Scalabilité

```sql
-- MariaDB Vector scale avec InnoDB
-- Même stratégies que tables relationnelles:

-- 1. Partitionnement
CREATE TABLE large_docs (
    id BIGINT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),
    created_at DATE
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- 2. Sharding horizontal (Spider Engine)
CREATE TABLE docs_shard (
    id BIGINT,
    embedding VECTOR(1536)
) ENGINE=Spider
COMMENT='wrapper "mysql", srv "shard1", table "docs"';

-- 3. Read replicas pour scale read
-- Primary: Write + Vector indexing
-- Replicas: Read queries (vecteur + SQL)
```

---

## 💰 Analyse Coût/Bénéfice

### Exemple : Startup SaaS (100k utilisateurs)

**Avant MariaDB 11.8** :
```
Infrastructure mensuelle:
- MariaDB (RDS) :         $250/mois
- Pinecone Standard :     $200/mois
- Développement :
  • 2 systèmes à maintenir : 40h/mois
  • Synchro données :        20h/mois
  • Monitoring x2 :          15h/mois
  Total dev : 75h × $80/h = $6,000/mois

Coût total mensuel: $6,450/mois
```

**Avec MariaDB 11.8** :
```
Infrastructure mensuelle:
- MariaDB 11.8 (RDS) :    $320/mois (+28% vs avant)
- Pinecone :              $0 (supprimé)
- Développement :
  • 1 seul système :       20h/mois
  • Synchro : 0h
  • Monitoring :           10h/mois
  Total dev : 30h × $80/h = $2,400/mois

Coût total mensuel: $2,720/mois

ÉCONOMIE : $3,730/mois = $44,760/an (69% de réduction !)
```

### ROI Détaillé

| Dimension | Avant | Après | Gain |
|-----------|-------|-------|------|
| **Coûts infra** | $450/mois | $320/mois | -29% |
| **Coûts dev/ops** | $6,000/mois | $2,400/mois | -60% |
| **Latency P99** | 35ms (réseau x2) | 12ms | -66% |
| **Complexité** | 2 systèmes | 1 système | -50% |
| **Backup/DR** | 2 procédures | 1 procédure | -50% |
| **Sécurité** | 2 surfaces d'attaque | 1 surface | -50% |

💡 **ROI typique** : 6-12 mois pour amortir la migration, puis économies récurrentes.

---

## 📅 Roadmap d'Adoption

### Phase 1 : Exploration (Semaine 1-2)

```
✅ Lire documentation MariaDB Vector
✅ Tester en local avec Docker
✅ Générer embeddings de test (OpenAI API)
✅ Comparer avec solution actuelle
```

```bash
# Quick start Docker
docker run -d \
  --name mariadb-vector \
  -e MYSQL_ROOT_PASSWORD=test123 \
  -p 3306:3306 \
  mariadb:11.8

# Connexion
mysql -h 127.0.0.1 -u root -ptest123

# Test rapide
CREATE TABLE test (
    id INT PRIMARY KEY,
    vec VECTOR(3)
);

INSERT INTO test VALUES 
    (1, '[1.0, 0.0, 0.0]'),
    (2, '[0.0, 1.0, 0.0]'),
    (3, '[0.7, 0.7, 0.0]');

SELECT id, VEC_DISTANCE_COSINE(vec, '[1.0, 0.0, 0.0]') AS dist
FROM test
ORDER BY dist;
```

### Phase 2 : POC (Semaine 3-4)

```
✅ Identifier 1 use case pilote
✅ Vectoriser dataset (1k-10k documents)
✅ Créer index HNSW
✅ Implémenter recherche sémantique
✅ Comparer précision/latency
✅ Valider avec métier
```

### Phase 3 : Migration (Mois 2-3)

```
✅ Plan de migration détaillé
✅ Migration staging
✅ Tests de charge
✅ Formation équipes
✅ Documentation interne
✅ Migration production (blue/green)
```

### Phase 4 : Optimisation (Mois 4-6)

```
✅ Tuning index HNSW
✅ Monitoring performance
✅ Nouveaux use cases
✅ Amélioration continue
```

---

## ✅ Points Clés à Retenir

- **MariaDB Vector** = IA native dans MariaDB, plus besoin de base vectorielle séparée
- **Type VECTOR** supporte jusqu'à 65,535 dimensions
- **Index HNSW** = Algorithme état de l'art pour recherche ANN
- **3 métriques** : Cosinus (texte), Euclidienne (géométrie), Dot Product (speed)
- **Optimisations SIMD** : 4-16x plus rapide selon CPU
- **Hybrid Search** : Combine vecteurs + SQL relationnel (impossible ailleurs)
- **Économies** : 40-70% de réduction des coûts infrastructure IA
- **Use cases** : RAG, Semantic Search, Recommendations, Anomaly Detection
- **Maturité** : Production-ready, utilisé par des entreprises en prod depuis juin 2025
- **Écosystème** : Intégration LangChain, LlamaIndex, OpenAI, Claude, etc.

---

## 🔗 Ressources et Références

### Documentation officielle

- 📖 [MariaDB Vector Overview](https://mariadb.com/kb/en/vector/)
- 📖 [HNSW Index Documentation](https://mariadb.com/kb/en/vector-indexes/)
- 📖 [Vector Functions Reference](https://mariadb.com/kb/en/vector-functions/)
- 🎥 [MariaDB Vector Webinar](https://mariadb.com/resources/webinars/mariadb-vector/)

### Sections détaillées de la formation

- **18.10** - MariaDB Vector : Guide complet technique
- **20.9** - Use cases IA/RAG en production
- **20.10** - MCP Server pour intégration IA
- **20.11** - Intégrations frameworks (LangChain, LlamaIndex)

### Ressources externes

- 🌐 [HNSW Algorithm Paper](https://arxiv.org/abs/1603.09320)
- 🌐 [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- 🌐 [Anthropic Claude MCP](https://www.anthropic.com/mcp)

---

## ➡️ Section Suivante

**F.3** [Impact sur migration et compatibilité](./03-impact-migration-compatibilite.md) - Stratégies et planification

---

**MariaDB** : Version 11.8 LTS (Juin 2025)

⏭️ [Impact sur migration et compatibilité](/annexes/nouveautes-11-8/03-impact-migration-compatibilite.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 Types d'index

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures

> **Prérequis** :
> - Section 5.1 - Fonctionnement des index B-Tree
> - Compréhension des requêtes SQL
> - Notions de complexité algorithmique

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Identifier les 5 types d'index disponibles dans MariaDB et leurs caractéristiques
- Choisir le type d'index approprié selon le cas d'usage et le type de données
- Comprendre les avantages et limitations de chaque type d'index
- Utiliser les index VECTOR/HNSW pour les applications d'intelligence artificielle 🆕
- Comparer les performances et contraintes de chaque type d'index
- Éviter les erreurs courantes dans le choix du type d'index

---

## Introduction

MariaDB propose **5 types d'index principaux**, chacun optimisé pour des cas d'usage spécifiques. Choisir le bon type d'index est crucial pour obtenir des performances optimales : un index inadapté peut être aussi inefficace que l'absence d'index.

### Vue d'ensemble des types d'index

| Type | Moteur | Cas d'usage principal | Nouveauté |
|------|--------|----------------------|-----------|
| **B-Tree** | InnoDB, Aria, MyISAM | Usage général (égalité, plages, tri) | Standard |
| **Hash** | MEMORY | Égalité stricte uniquement | Standard |
| **Full-Text** | InnoDB, Aria, MyISAM | Recherche textuelle dans documents | Standard |
| **Spatial** | InnoDB, Aria, MyISAM | Données géographiques (GIS) | Standard |
| **VECTOR (HNSW)** | InnoDB | Recherche vectorielle pour IA/ML | 🆕 11.8 |

💡 **Principe fondamental** : Le type d'index doit correspondre au **pattern d'accès aux données**, pas seulement au type de données.

### Matrice de décision rapide

```
Type de requête → Type d'index recommandé

WHERE column = value               → B-Tree (ou Hash si MEMORY)
WHERE column BETWEEN x AND y       → B-Tree
WHERE column > value               → B-Tree
ORDER BY column                    → B-Tree
MATCH(...) AGAINST(...)           → Full-Text
ST_Distance(point1, point2) < d   → Spatial
VEC_DISTANCE(vector1, vector2)    → VECTOR (HNSW) 🆕
```

---

## 1. B-Tree : L'index universel

### Caractéristiques principales

Le **B-Tree** (Balanced Tree) est le type d'index par défaut dans MariaDB, utilisé pour environ **95% des cas d'usage**.

```sql
-- Syntaxe par défaut (B-Tree implicite)
CREATE INDEX idx_lastname ON users(last_name);

-- Syntaxe explicite
CREATE INDEX idx_lastname ON users(last_name) USING BTREE;
```

**Points forts** :
- ✅ Supporte l'égalité : `WHERE column = value`
- ✅ Supporte les plages : `WHERE column BETWEEN x AND y`
- ✅ Supporte les comparaisons : `<`, `>`, `<=`, `>=`
- ✅ Supporte les préfixes : `WHERE column LIKE 'abc%'`
- ✅ Optimise ORDER BY et GROUP BY
- ✅ Supporte les index composites
- ✅ Excellent pour les range scans

**Limitations** :
- ❌ LIKE avec wildcard au début : `WHERE column LIKE '%abc'`
- ❌ Fonctions sur colonnes indexées : `WHERE YEAR(date_column) = 2025`
- ❌ Comparaisons avec inégalité : `WHERE column != value`

### Quand utiliser B-Tree

```sql
-- ✅ Excellent pour recherches par égalité
SELECT * FROM orders WHERE customer_id = 12345;

-- ✅ Excellent pour plages
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31';

-- ✅ Excellent pour tri
SELECT * FROM products ORDER BY price DESC LIMIT 10;

-- ✅ Excellent pour jointures
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

### Complexité

| Opération | Complexité | Note |
|-----------|------------|------|
| Recherche | O(log n) | 3-4 I/O pour des millions de lignes |
| Insertion | O(log n) | Inclut potentiellement des splits |
| Suppression | O(log n) | Inclut potentiellement des merges |
| Range scan | O(log n + k) | k = nombre de résultats |

💡 **Usage recommandé** : C'est votre index par défaut pour presque tous les cas. En cas de doute, choisissez B-Tree.

---

## 2. Hash : Égalité pure ultra-rapide

### Caractéristiques principales

L'index **Hash** utilise une **table de hachage** pour des recherches par égalité en temps constant.

```sql
-- Création (principalement avec MEMORY engine)
CREATE TABLE session_cache (
    session_id CHAR(32) PRIMARY KEY,
    user_id INT,
    data BLOB,
    INDEX idx_user (user_id) USING HASH
) ENGINE=MEMORY;
```

**Points forts** :
- ✅ Égalité stricte ultra-rapide : O(1) en moyenne
- ✅ Très compact en mémoire
- ✅ Excellent pour lookup tables

**Limitations importantes** :
- ❌ **Pas de plages** : `BETWEEN`, `<`, `>` ignorent l'index
- ❌ **Pas de tri** : `ORDER BY` ne peut pas utiliser l'index
- ❌ **Pas de préfixes** : `LIKE 'abc%'` n'utilise pas l'index
- ❌ **Pas d'index composites efficaces** : Hash(a,b) ≠ Hash(a) + Hash(b)
- ❌ **Collisions possibles** : Nécessite vérification après hash lookup

### Fonctionnement interne

```
Principe d'un index Hash :

1. Fonction de hachage : key → hash_value
   "john@example.com" → 3847362

2. Hash table :
   ┌─────────┬──────────────┐
   │  Hash   │   Pointeur   │
   ├─────────┼──────────────┤
   │ 3847362 │ → Row #1234  │
   │ 9283746 │ → Row #5678  │
   │ 1928374 │ → Row #9012  │
   └─────────┴──────────────┘

3. Recherche :
   hash("john@example.com") → 3847362 → Row #1234

   Complexité : O(1) !
```

### Quand utiliser Hash

```sql
-- ✅ Excellent pour égalité stricte
SELECT * FROM session_cache WHERE session_id = 'abc123...';

-- ✅ Bon pour lookup tables en mémoire
CREATE TABLE country_codes (
    code CHAR(2) PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=MEMORY;

CREATE INDEX idx_code ON country_codes(code) USING HASH;

-- ❌ INUTILE pour plages (n'utilise pas l'index)
SELECT * FROM orders WHERE order_id > 1000;

-- ❌ INUTILE pour tri
SELECT * FROM products ORDER BY category_id;
```

### Cas d'usage typiques

**1. Cache de sessions en mémoire**
```sql
CREATE TABLE active_sessions (
    session_id CHAR(64) PRIMARY KEY,
    user_id INT,
    last_activity TIMESTAMP,
    data JSON,
    INDEX idx_session (session_id) USING HASH
) ENGINE=MEMORY;
```

**2. Tables de correspondance (lookup tables)**
```sql
CREATE TABLE currency_rates (
    currency_code CHAR(3) PRIMARY KEY,
    rate DECIMAL(10,6),
    updated_at TIMESTAMP,
    INDEX idx_currency (currency_code) USING HASH
) ENGINE=MEMORY;
```

⚠️ **Attention** : Avec InnoDB, `USING HASH` est **ignoré** et converti en B-Tree. Hash est principalement pour le moteur MEMORY.

### Comparaison Hash vs B-Tree

| Critère | Hash | B-Tree |
|---------|------|--------|
| Égalité stricte | O(1) ⚡ | O(log n) |
| Plages (BETWEEN) | ❌ | ✅ O(log n + k) |
| Tri (ORDER BY) | ❌ | ✅ |
| Préfixes (LIKE 'x%') | ❌ | ✅ |
| Espace mémoire | Très compact | Plus volumineux |
| Moteurs supportés | MEMORY | Tous |

---

## 3. Full-Text : Recherche textuelle avancée

### Caractéristiques principales

L'index **Full-Text** est spécialisé dans la **recherche de mots-clés** dans des textes longs.

```sql
-- Création d'un index Full-Text
CREATE TABLE articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    content TEXT,
    published_at DATETIME,
    FULLTEXT INDEX idx_content (title, content)
) ENGINE=InnoDB;

-- Recherche full-text
SELECT id, title,
       MATCH(title, content) AGAINST('MariaDB performance' IN NATURAL LANGUAGE MODE) AS relevance
FROM articles
WHERE MATCH(title, content) AGAINST('MariaDB performance' IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC;
```

**Points forts** :
- ✅ Recherche de mots dans textes longs
- ✅ Pertinence et scoring automatique
- ✅ Support des opérateurs booléens
- ✅ Support de plusieurs langues
- ✅ Recherche phonétique (avec plugins)
- ✅ Gestion des mots vides (stopwords)

**Limitations** :
- ❌ Longueur minimale des mots (ft_min_word_len = 4 par défaut)
- ❌ Pas de recherche exacte de phrases avec ponctuation
- ❌ Sensibilité aux stopwords
- ❌ Overhead de maintenance significatif

### Modes de recherche

**1. NATURAL LANGUAGE MODE** (par défaut)
```sql
-- Recherche naturelle avec scoring
SELECT *, MATCH(content) AGAINST('database optimization') AS score
FROM articles
WHERE MATCH(content) AGAINST('database optimization')
ORDER BY score DESC;
```

**2. BOOLEAN MODE**
```sql
-- Recherche booléenne avec opérateurs
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST(
    '+MariaDB +performance -MySQL'
    IN BOOLEAN MODE
);

-- Opérateurs disponibles :
-- + : mot obligatoire
-- - : mot exclu
-- * : wildcard
-- "" : phrase exacte
-- () : groupement
-- > : augmente le poids
-- < : diminue le poids
```

**3. QUERY EXPANSION MODE**
```sql
-- Expansion automatique de la requête
SELECT * FROM articles
WHERE MATCH(content) AGAINST(
    'database'
    WITH QUERY EXPANSION
);

-- MariaDB trouve les résultats pour "database"
-- puis réexécute avec des termes reliés trouvés
```

### Configuration Full-Text

```sql
-- Variables importantes
SHOW VARIABLES LIKE 'ft_%';

-- ft_min_word_len : Longueur minimale des mots (défaut : 4)
-- ft_max_word_len : Longueur maximale des mots (défaut : 84)
-- ft_stopword_file : Fichier des mots à ignorer

-- Modifier la longueur minimale (nécessite rebuild)
SET GLOBAL ft_min_word_len = 3;

-- Puis reconstruire l'index
ALTER TABLE articles DROP INDEX idx_content;
ALTER TABLE articles ADD FULLTEXT INDEX idx_content (title, content);
```

### Quand utiliser Full-Text

```sql
-- ✅ Excellent pour recherche dans blogs/forums
SELECT * FROM posts
WHERE MATCH(title, body) AGAINST('troubleshooting connection issues');

-- ✅ Excellent pour documentation/knowledge base
SELECT * FROM documentation
WHERE MATCH(content) AGAINST('+installation +Linux' IN BOOLEAN MODE);

-- ✅ Excellent pour recherche produits
SELECT *, MATCH(name, description) AGAINST('wireless headphones') AS relevance
FROM products
WHERE MATCH(name, description) AGAINST('wireless headphones')
ORDER BY relevance DESC
LIMIT 20;

-- ❌ Mauvais pour recherche exacte
-- Utilisez LIKE ou B-Tree pour cela
SELECT * FROM users WHERE email = 'user@example.com';
```

### Cas d'usage typiques

**1. Système de tickets / Support**
```sql
CREATE TABLE tickets (
    id INT PRIMARY KEY AUTO_INCREMENT,
    subject VARCHAR(255),
    description TEXT,
    resolution TEXT,
    created_at DATETIME,
    FULLTEXT INDEX idx_search (subject, description, resolution)
);

-- Recherche de solutions
SELECT id, subject,
       MATCH(subject, description, resolution)
       AGAINST('error connecting database' IN NATURAL LANGUAGE MODE) AS relevance
FROM tickets
WHERE status = 'resolved'
AND MATCH(subject, description, resolution)
    AGAINST('error connecting database' IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC
LIMIT 10;
```

**2. Blog / CMS**
```sql
CREATE TABLE blog_posts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    body TEXT,
    tags VARCHAR(255),
    published BOOLEAN,
    FULLTEXT INDEX idx_search (title, body, tags)
);
```

💡 **Alternative moderne** : Pour des besoins de recherche très avancés, considérez Elasticsearch ou Meilisearch en complément de MariaDB.

---

## 4. Spatial : Données géographiques

### Caractéristiques principales

L'index **Spatial** (ou **R-Tree**) est optimisé pour les **données géométriques et géographiques**.

```sql
-- Création d'une table avec données spatiales
CREATE TABLE stores (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    address TEXT,
    coordinates POINT NOT NULL,
    delivery_zone POLYGON,
    SPATIAL INDEX idx_location (coordinates)
) ENGINE=InnoDB;

-- Insertion de points géographiques (longitude, latitude)
INSERT INTO stores (name, address, coordinates) VALUES
('Store Paris', '123 Rue de Rivoli', ST_GeomFromText('POINT(2.3522 48.8566)')),
('Store Lyon', '45 Cours Lafayette', ST_GeomFromText('POINT(4.8357 45.7640)'));

-- Recherche des magasins dans un rayon de 5 km
SELECT id, name,
       ST_Distance_Sphere(coordinates, ST_GeomFromText('POINT(2.3522 48.8566)')) / 1000 AS distance_km
FROM stores
WHERE ST_Distance_Sphere(coordinates, ST_GeomFromText('POINT(2.3522 48.8566)')) < 5000
ORDER BY distance_km;
```

**Points forts** :
- ✅ Recherches géospatiales efficaces
- ✅ Calculs de distance optimisés
- ✅ Requêtes de proximité (nearby)
- ✅ Intersection, containment, overlap
- ✅ Support GIS complet (OGC)

**Types de données supportés** :
- `POINT` : Un point (x, y)
- `LINESTRING` : Une ligne
- `POLYGON` : Un polygone
- `MULTIPOINT`, `MULTILINESTRING`, `MULTIPOLYGON`
- `GEOMETRY`, `GEOMETRYCOLLECTION`

### Fonctions spatiales courantes

```sql
-- Distance entre deux points (en mètres sur une sphère)
SELECT ST_Distance_Sphere(
    ST_GeomFromText('POINT(2.3522 48.8566)'),  -- Paris
    ST_GeomFromText('POINT(4.8357 45.7640)')   -- Lyon
) AS distance_meters; -- Résultat : ~392,000 mètres

-- Point dans un polygone (zone de livraison)
SELECT name FROM stores
WHERE ST_Contains(
    delivery_zone,
    ST_GeomFromText('POINT(2.3500 48.8500)')
);

-- Intersection de deux géométries
SELECT ST_Intersects(polygon1, polygon2) FROM zones;

-- Calcul de surface
SELECT name, ST_Area(delivery_zone) AS area_sq_meters
FROM stores;
```

### Structure R-Tree

L'index Spatial utilise un **R-Tree** (Rectangle Tree) :

```
Principe du R-Tree :

         Niveau 0 (Racine)
         ┌───────────────────┐
         │  Rectangle global │
         └────────┬──────────┘
                  │
         ┌────────┴────────┐
         │                 │
    Niveau 1          Niveau 1
    ┌────────┐       ┌────────┐
    │ Zone A │       │ Zone B │
    └────┬───┘       └────┬───┘
         │                │
    ┌────┴───┐       ┌────┴───┐
    │        │       │        │
  Points   Points  Points  Points

Chaque niveau encapsule géométriquement les niveaux inférieurs
```

### Quand utiliser Spatial

```sql
-- ✅ Excellent pour "Trouver les points à proximité"
SELECT * FROM restaurants
WHERE ST_Distance_Sphere(location, ST_GeomFromText('POINT(2.35 48.85)')) < 1000;

-- ✅ Excellent pour "Point dans zone"
SELECT * FROM delivery_zones
WHERE ST_Contains(zone_polygon, ST_GeomFromText('POINT(2.35 48.85)'));

-- ✅ Excellent pour applications de cartographie
SELECT * FROM pois
WHERE MBRContains(
    ST_GeomFromText('POLYGON((2.2 48.8, 2.4 48.8, 2.4 48.9, 2.2 48.9, 2.2 48.8))'),
    location
);

-- ❌ Inutile pour données non-géographiques
```

### Cas d'usage typiques

**1. Application de livraison**
```sql
CREATE TABLE drivers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    current_location POINT NOT NULL,
    available BOOLEAN,
    SPATIAL INDEX idx_location (current_location)
);

-- Trouver les livreurs disponibles dans un rayon de 5 km
SELECT id, name,
       ST_Distance_Sphere(current_location, :customer_location) AS distance
FROM drivers
WHERE available = TRUE
AND ST_Distance_Sphere(current_location, :customer_location) < 5000
ORDER BY distance
LIMIT 5;
```

**2. Système de gestion immobilière**
```sql
CREATE TABLE properties (
    id INT PRIMARY KEY,
    address VARCHAR(255),
    location POINT NOT NULL,
    price DECIMAL(12,2),
    SPATIAL INDEX idx_location (location)
);

-- Propriétés dans un quartier délimité
SELECT * FROM properties
WHERE ST_Contains(
    ST_GeomFromText('POLYGON((2.3 48.85, 2.35 48.85, 2.35 48.87, 2.3 48.87, 2.3 48.85))'),
    location
);
```

💡 **Performance** : Les index Spatial sont très efficaces pour les requêtes de proximité, mais moins pour des calculs complexes. Pour des besoins GIS avancés, considérez PostGIS (PostgreSQL).

---

## 5. VECTOR (HNSW) : Recherche vectorielle pour l'IA 🆕

### Nouveauté MariaDB 11.8 LTS

L'index **VECTOR** avec algorithme **HNSW** (Hierarchical Navigable Small Worlds) est la **grande nouveauté de MariaDB 11.8**, conçue spécifiquement pour les applications d'**intelligence artificielle** et de **machine learning**.

```sql
-- Création d'une table avec colonne VECTOR
CREATE TABLE documents (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    content TEXT,
    embedding VECTOR(1536) NOT NULL  -- Vecteur de 1536 dimensions (OpenAI)
) ENGINE=InnoDB;

-- Création d'un index HNSW
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (
    m = 16,                -- Nombre de connexions par nœud (8-64)
    ef_construction = 200, -- Précision lors de la construction (100-500)
    metric = 'cosine'      -- Métrique de distance : cosine, euclidean, dot
);

-- Recherche de similarité vectorielle
SET @query_vector = VEC_FromText('[0.123, 0.456, ...]'); -- 1536 dimensions

SELECT id, title,
       VEC_DISTANCE_COSINE(embedding, @query_vector) AS similarity
FROM documents
ORDER BY similarity ASC
LIMIT 10;
```

**Points forts** :
- ✅ Recherche de similarité ultra-rapide (Approximate Nearest Neighbors - ANN)
- ✅ Support des embeddings de modèles IA (OpenAI, Claude, Llama, etc.)
- ✅ Optimisations SIMD matérielles (AVX2, AVX512, ARM NEON, IBM Power10)
- ✅ Scalabilité : millions de vecteurs avec performances constantes
- ✅ Intégration native avec workflows RAG (Retrieval-Augmented Generation)
- ✅ Jusqu'à 16 000 dimensions supportées

**Métriques de distance disponibles** :
- **Cosine** : Mesure l'angle entre vecteurs (texte, sémantique)
- **Euclidean** : Distance géométrique classique
- **Dot product** : Produit scalaire (recommandations)

### Fonctionnement HNSW

L'algorithme **HNSW** construit un graphe multi-niveaux navigable :

```
Structure HNSW (Hierarchical Navigable Small Worlds) :

Niveau 2 (sparse)    [V1] ←──→ [V5]
                       ↓          ↓
Niveau 1              [V1]←→[V2]←→[V5]←→[V8]
                       ↓    ↓     ↓     ↓
Niveau 0 (dense)      [V1][V2][V3][V4][V5][V6][V7][V8][V9]...
                       ↔️    ↔️    ↔️    ↔️    ↔️

Recherche :
1. Commencer au niveau le plus haut
2. Naviguer vers le vecteur le plus proche
3. Descendre au niveau inférieur
4. Répéter jusqu'au niveau 0
5. Retourner les k plus proches voisins

Complexité : O(log n) au lieu de O(n) !
```

### Paramètres de tuning

```sql
-- m : Nombre de connexions par nœud
-- Plus élevé = meilleure précision, plus de mémoire
-- Recommandé : 12-48, défaut 16

-- ef_construction : Taille de la liste de candidats lors construction
-- Plus élevé = meilleure qualité d'index, construction plus lente
-- Recommandé : 100-500, défaut 200

-- ef_search : Taille de la liste lors recherche (runtime)
-- Plus élevé = meilleure précision, recherche plus lente
-- Défaut : 100, peut être ajusté par requête

-- Exemple : Privilégier précision
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
WITH (
    m = 32,                -- Plus de connexions
    ef_construction = 400, -- Construction plus précise
    metric = 'cosine'
);

-- Au moment de la recherche
SET @@hnsw_ef_search = 200; -- Précision accrue temporaire
```

### Cas d'usage IA/ML

**1. Semantic Search (Recherche sémantique)** 🤖

```sql
-- Table pour documents avec embeddings
CREATE TABLE knowledge_base (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    embedding VECTOR(1536) NOT NULL, -- OpenAI text-embedding-3-small
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_embedding (embedding) USING HNSW WITH (m=16, ef_construction=200, metric='cosine')
);

-- Fonction pour recherche sémantique
-- L'embedding de la requête est généré côté application (Python/JS)
SELECT
    id,
    title,
    content,
    VEC_DISTANCE_COSINE(embedding, :query_embedding) AS similarity_score
FROM knowledge_base
ORDER BY similarity_score ASC
LIMIT 5;
```

**2. Chatbot RAG (Retrieval-Augmented Generation)** 🤖

```sql
-- Workflow typique :
-- 1. User pose une question : "Comment optimiser mes index MariaDB ?"
-- 2. Application génère embedding de la question (via OpenAI API)
-- 3. MariaDB trouve les documents les plus pertinents
-- 4. Application envoie documents + question au LLM
-- 5. LLM génère une réponse contextualisée

CREATE TABLE chatbot_context (
    id INT PRIMARY KEY AUTO_INCREMENT,
    source VARCHAR(255),
    chunk TEXT,           -- Morceau de texte (500-1000 tokens)
    embedding VECTOR(1536),
    metadata JSON,
    INDEX idx_vec (embedding) USING HNSW WITH (metric='cosine')
);

-- Requête de contexte pour RAG
SELECT chunk, metadata
FROM chatbot_context
ORDER BY VEC_DISTANCE_COSINE(embedding, :question_embedding)
LIMIT 3;  -- Top 3 chunks les plus pertinents
```

**3. Système de recommandation** 🎯

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    price DECIMAL(10,2),
    features_vector VECTOR(512) NOT NULL, -- Vecteur de caractéristiques produit
    INDEX idx_features (features_vector) USING HNSW WITH (metric='dot')
);

-- Recommandations "produits similaires"
SELECT
    p2.id,
    p2.name,
    p2.price,
    VEC_DISTANCE_DOT(p1.features_vector, p2.features_vector) AS similarity
FROM products p1
CROSS JOIN products p2
WHERE p1.id = 123  -- Produit de référence
AND p2.id != 123
ORDER BY similarity DESC
LIMIT 10;
```

**4. Détection d'anomalies** 🔍

```sql
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY,
    user_id INT,
    amount DECIMAL(10,2),
    behavior_vector VECTOR(256),  -- Profil comportemental
    timestamp DATETIME,
    INDEX idx_behavior (behavior_vector) USING HNSW WITH (metric='euclidean')
);

-- Détecter transactions anormales
-- Transaction est anormale si aucune transaction similaire n'existe
SELECT t1.id, t1.amount,
       MIN(VEC_DISTANCE_EUCLIDEAN(t1.behavior_vector, t2.behavior_vector)) AS closest_distance
FROM transactions t1
CROSS JOIN transactions t2
WHERE t1.id != t2.id
AND t1.timestamp > DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY t1.id
HAVING closest_distance > 0.8  -- Seuil d'anomalie
ORDER BY closest_distance DESC;
```

### Intégration avec frameworks IA

**Python avec OpenAI**
```python
import openai
import mariadb

# Générer embedding
response = openai.Embedding.create(
    model="text-embedding-3-small",
    input="Comment optimiser les index MariaDB ?"
)
query_embedding = response['data'][0]['embedding']  # 1536 dimensions

# Convertir en format MariaDB
embedding_str = f"[{','.join(map(str, query_embedding))}]"

# Recherche dans MariaDB
cursor.execute("""
    SELECT id, title, content,
           VEC_DISTANCE_COSINE(embedding, VEC_FromText(%s)) AS score
    FROM documents
    ORDER BY score ASC
    LIMIT 5
""", (embedding_str,))
```

**LangChain**
```python
from langchain.vectorstores import MariaDBVector
from langchain.embeddings import OpenAIEmbeddings

# Configuration
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = MariaDBVector(
    connection_string="mariadb://user:pass@localhost/db",
    embedding_function=embeddings,
    table_name="documents",
    dimension=1536
)

# Recherche sémantique
results = vectorstore.similarity_search(
    "optimisation des performances",
    k=5
)
```

### Performance et optimisations

**Optimisations SIMD** 🆕

MariaDB 11.8 exploite les instructions SIMD pour accélérer les calculs vectoriels :

```
Support matériel :
- Intel/AMD : AVX2, AVX-512
- ARM : NEON
- IBM Power10 : VSX

Accélération typique : 4-8x plus rapide que sans SIMD
```

```sql
-- Vérifier le support SIMD
SHOW VARIABLES LIKE 'have_vector_%';

-- have_vector_avx2 : ON/OFF
-- have_vector_avx512 : ON/OFF
-- have_vector_neon : ON/OFF
```

**Benchmarks indicatifs** :

| Opération | Sans index | Avec HNSW | Speedup |
|-----------|------------|-----------|---------|
| Top-10 sur 1M vecteurs | ~2000 ms | ~5 ms | 400x |
| Top-100 sur 10M vecteurs | ~25000 ms | ~50 ms | 500x |
| Insertion 1 vecteur | - | ~2 ms | - |

💡 **Note** : HNSW est un algorithme **approximatif** (ANN - Approximate Nearest Neighbors). Il peut manquer quelques résultats exacts en échange d'une vitesse considérable.

### Configuration pour production

```sql
-- Variables système pour VECTOR
SHOW VARIABLES LIKE 'vector_%';

-- Taille max d'un vecteur (défaut : 16000 dimensions)
SET GLOBAL vector_max_dimension = 16000;

-- Mémoire allouée pour construction HNSW
SET GLOBAL innodb_vector_cache_size = 1073741824; -- 1 GB

-- Parallélisme lors de la construction
SET GLOBAL innodb_vector_threads = 4;
```

---

## Comparaison globale des types d'index

### Tableau récapitulatif

| Critère | B-Tree | Hash | Full-Text | Spatial | VECTOR |
|---------|--------|------|-----------|---------|--------|
| **Égalité** | ✅ Très bon | ✅ Excellent | ❌ | ❌ | ❌ |
| **Plages** | ✅ Excellent | ❌ | ❌ | ❌ | ❌ |
| **Tri** | ✅ Excellent | ❌ | ❌ | ❌ | ❌ |
| **Texte libre** | ❌ | ❌ | ✅ Excellent | ❌ | ❌ |
| **Géographique** | ❌ | ❌ | ❌ | ✅ Excellent | ❌ |
| **Similarité IA** | ❌ | ❌ | ❌ | ❌ | ✅ Excellent |
| **Complexité recherche** | O(log n) | O(1) | O(n)* | O(log n) | O(log n) |
| **Espace disque** | Moyen | Faible | Élevé | Moyen | Élevé |
| **Overhead écriture** | Faible | Très faible | Élevé | Moyen | Moyen |
| **Moteurs** | Tous | MEMORY | InnoDB, MyISAM | InnoDB, MyISAM | InnoDB |

*Full-Text peut être très rapide grâce aux optimisations internes, mais théoriquement linéaire.

### Matrice de décision détaillée

```
Choisir le bon type d'index :

Type de données          Type de requête              Index recommandé
─────────────────────────────────────────────────────────────────────
INT, BIGINT, VARCHAR     = exact, BETWEEN, <, >      → B-Tree
CHAR(32) session         = exact uniquement          → Hash (MEMORY)
TEXT, VARCHAR(500+)      MATCH ... AGAINST           → Full-Text
POINT, POLYGON           ST_Distance, ST_Contains    → Spatial
VECTOR(1536)             VEC_DISTANCE                → VECTOR (HNSW) 🆕

Workload                 Pattern accès                Index recommandé
─────────────────────────────────────────────────────────────────────
OLTP rapide              Lookup clé primaire         → B-Tree
Cache en RAM             Lookup session/token        → Hash
Blog / Documentation     Recherche mots-clés         → Full-Text
App livraison/maps       Recherche proximité         → Spatial
Chatbot IA / RAG         Recherche sémantique        → VECTOR 🆕
```

### Combinaisons d'index

Plusieurs index peuvent coexister sur une même table :

```sql
CREATE TABLE articles (
    id INT PRIMARY KEY,                              -- B-Tree (PRIMARY KEY)
    title VARCHAR(255),
    content TEXT,
    category_id INT,
    location POINT,
    embedding VECTOR(1536),
    created_at DATETIME,

    INDEX idx_category (category_id),                -- B-Tree
    FULLTEXT INDEX idx_search (title, content),      -- Full-Text
    SPATIAL INDEX idx_location (location),           -- Spatial
    INDEX idx_embedding (embedding) USING HNSW,      -- VECTOR 🆕
    INDEX idx_created (created_at)                   -- B-Tree
) ENGINE=InnoDB;

-- Chaque requête utilise l'index le plus approprié
SELECT * FROM articles WHERE category_id = 5;               -- B-Tree
SELECT * FROM articles WHERE MATCH(title) AGAINST('...'); -- Full-Text
SELECT * FROM articles WHERE ST_Distance(...) < 1000;       -- Spatial
SELECT * FROM articles ORDER BY VEC_DISTANCE(...);          -- VECTOR
```

---

## Erreurs courantes et pièges

### Piège 1 : Utiliser Hash sur InnoDB

```sql
-- ❌ ERREUR : Hash est ignoré sur InnoDB
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    INDEX idx_email (email) USING HASH  -- Converti en B-Tree !
) ENGINE=InnoDB;

-- ✅ CORRECT : Hash uniquement avec MEMORY
CREATE TABLE sessions (
    session_id CHAR(32) PRIMARY KEY,
    INDEX idx_session (session_id) USING HASH
) ENGINE=MEMORY;
```

### Piège 2 : Full-Text sur colonnes trop courtes

```sql
-- ❌ PEU EFFICACE : Full-Text sur VARCHAR(50)
CREATE TABLE tags (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    FULLTEXT INDEX idx_name (name)  -- Overkill
);

-- ✅ MIEUX : B-Tree suffit
CREATE INDEX idx_name ON tags(name) USING BTREE;
```

### Piège 3 : Oublier les dimensions du VECTOR

```sql
-- ❌ ERREUR : Dimensions ne correspondent pas
CREATE TABLE docs (
    embedding VECTOR(1536)  -- 1536 dimensions
);

-- Insertion avec 768 dimensions → ERREUR !
INSERT INTO docs VALUES (VEC_FromText('[0.1, 0.2, ...]')); -- 768 dims

-- ✅ CORRECT : Toujours vérifier les dimensions
-- OpenAI text-embedding-3-small : 1536
-- OpenAI text-embedding-3-large : 3072
-- Sentence-BERT : 384 ou 768
```

### Piège 4 : Spatial sans SRID

```sql
-- ❌ IMPRÉCIS : Sans SRID (système de référence)
INSERT INTO stores (location)
VALUES (ST_GeomFromText('POINT(2.35 48.85)'));

-- ✅ MIEUX : Avec SRID 4326 (WGS84 - GPS standard)
INSERT INTO stores (location)
VALUES (ST_GeomFromText('POINT(2.35 48.85)', 4326));
```

---

## ✅ Points clés à retenir

- 🌲 **B-Tree** : L'index universel (95% des cas), excellent pour égalité, plages, tri
- #️⃣ **Hash** : Uniquement pour égalité stricte, principalement MEMORY engine, O(1)
- 📝 **Full-Text** : Recherche textuelle avec pertinence, mots-clés dans textes longs
- 🗺️ **Spatial** : Données géographiques, recherches de proximité, GIS
- 🤖 **VECTOR (HNSW)** 🆕 : Nouveauté 11.8 pour IA/ML, recherche sémantique, RAG, embeddings
- 🎯 **Choix selon pattern** : Le type d'index doit correspondre au type de requête, pas juste au type de données
- 🔄 **Plusieurs index possibles** : Une table peut avoir différents types d'index simultanément
- ⚡ **SIMD pour VECTOR** : Optimisations matérielles AVX2/AVX512/NEON dans 11.8
- 📊 **Trade-offs** : Chaque type a ses forces et faiblesses, aucun n'est universel
- 🚫 **Éviter Hash sur InnoDB** : Converti automatiquement en B-Tree
- 💡 **Default = B-Tree** : En cas de doute, choisissez B-Tree
- 🎓 **HNSW est approximatif** : Sacrifie précision parfaite pour vitesse extrême

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Storage Engine Index Types](https://mariadb.com/kb/en/storage-engine-index-types/)
- [📖 Full-Text Indexes](https://mariadb.com/kb/en/fulltext-index-overview/)
- [📖 Spatial Indexes](https://mariadb.com/kb/en/spatial-index/)
- [📖 VECTOR Data Type](https://mariadb.com/kb/en/vector-data-type/) 🆕
- [📖 HNSW Vector Indexes](https://mariadb.com/kb/en/vector-indexes/) 🆕
- [📖 Vector Functions](https://mariadb.com/kb/en/vector-functions/) 🆕

### Ressources IA/Vector Search

- [HNSW Algorithm Paper](https://arxiv.org/abs/1603.09320) - Publication originale
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [LangChain Documentation](https://python.langchain.com/docs/integrations/vectorstores/)

### Articles techniques

- [MariaDB 11.8 Vector Search Announcement](https://mariadb.org/vector-search/)
- [Full-Text Search Best Practices](https://www.percona.com/blog/)
- [Spatial Indexes Explained](https://use-the-index-luke.com/)

---

## ➡️ Section suivante

**[5.2.1 B-Tree : Le standard](./02.1-btree.md)**

Approfondissez vos connaissances sur le B-Tree, le type d'index le plus utilisé : implémentation dans InnoDB, différence entre clustered et secondary indexes, optimisations spécifiques, et cas d'usage avancés.

---


⏭️ [B-Tree : Le standard](/05-index-et-performance/02.1-btree.md)

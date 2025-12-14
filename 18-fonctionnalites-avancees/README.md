🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18. Fonctionnalités Avancées

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 8-10 heures  
> **Prérequis** : Chapitres 1-7, bonne maîtrise SQL et administration MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- Utiliser les **Sequences** pour générer des identifiants indépendants des tables
- Implémenter des **tables temporelles** (System-Versioned) pour l'audit et l'historisation automatique
- Maîtriser les **Application Time Period Tables** pour gérer des périodes temporelles applicatives
- Créer et optimiser des **colonnes virtuelles et générées** (VIRTUAL vs STORED)
- Utiliser les **invisible columns** pour des migrations et évolutions de schéma sans impact
- Mettre en œuvre la **compression de tables** pour optimiser le stockage
- Configurer l'**encryption at rest** pour sécuriser les données sensibles
- **Exploiter MariaDB Vector** pour la recherche vectorielle IA/RAG (nouveauté majeure 11.8) 🆕
- Réaliser des **modifications de schéma non-bloquantes** (Online Schema Change)

---

## Introduction

MariaDB 11.8 LTS introduit un ensemble de fonctionnalités avancées qui positionnent le SGBD au-delà d'une simple base relationnelle classique. Ces fonctionnalités répondent à des **cas d'usage spécialisés** en production moderne :

- **Conformité réglementaire** : Audit trail, historisation, versioning
- **Optimisation des performances** : Colonnes calculées, compression, indexation avancée
- **Sécurité** : Chiffrement, masquage de données
- **Intelligence Artificielle** : Recherche vectorielle pour RAG, semantic search, recommendations
- **Disponibilité** : Modifications de schéma sans downtime

Ce chapitre explore ces fonctionnalités avancées avec un focus particulier sur **MariaDB Vector**, la fonctionnalité phare de la version 11.8 qui ouvre MariaDB au monde de l'IA générative.

💡 **Conseil** : Ces fonctionnalités sont puissantes mais peuvent avoir un impact significatif sur les performances et la complexité. Évaluez toujours leur pertinence pour votre cas d'usage avant adoption en production.

---

## Vue d'ensemble des fonctionnalités avancées

### 1. Sequences (CREATE SEQUENCE)

Les **sequences** permettent de générer des nombres séquentiels **indépendamment des tables**, offrant plus de flexibilité que les colonnes AUTO_INCREMENT.

**Cas d'usage principaux** :
- Génération d'identifiants multi-tables
- Numérotation de factures, commandes, tickets
- Partage de séquences entre plusieurs applications
- Éviter les gaps d'AUTO_INCREMENT après ROLLBACK

```sql
-- Création d'une séquence pour numérotation de factures
CREATE SEQUENCE invoice_seq
  START WITH 1000
  INCREMENT BY 1
  MINVALUE 1000
  MAXVALUE 999999
  CACHE 50
  CYCLE;

-- Utilisation dans INSERT
INSERT INTO invoices (invoice_number, customer_id, amount)
VALUES (NEXT VALUE FOR invoice_seq, 123, 1500.00);

-- Obtenir la valeur courante
SELECT LASTVAL(invoice_seq);
```

**Avantages vs AUTO_INCREMENT** :
- ✅ Partage entre tables
- ✅ Contrôle précis (min/max/cycle)
- ✅ Pas de lock de table
- ✅ Reset indépendant des données

⚠️ **Attention** : CACHE améliore les performances mais peut créer des gaps en cas de crash.

---

### 2. System-Versioned Tables (Tables Temporelles)

Les **tables temporelles** (ou system-versioned) conservent automatiquement l'**historique complet des modifications** avec des timestamps gérés par le système.

**Cas d'usage** :
- Audit trail et conformité (RGPD, SOX, HIPAA)
- Analyse forensique
- Revert vers un état antérieur
- Audit des changements utilisateurs

```sql
-- Création d'une table temporelle
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  name VARCHAR(100),
  salary DECIMAL(10,2),
  department VARCHAR(50)
) WITH SYSTEM VERSIONING;

-- MariaDB ajoute automatiquement deux colonnes cachées :
-- row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START
-- row_end TIMESTAMP(6) GENERATED ALWAYS AS ROW END

-- Modifications normales
INSERT INTO employees VALUES (1, 'Alice', 50000, 'IT');
UPDATE employees SET salary = 55000 WHERE employee_id = 1;
UPDATE employees SET department = 'Engineering' WHERE employee_id = 1;

-- Requête à un instant T (AS OF)
SELECT * FROM employees 
FOR SYSTEM_TIME AS OF TIMESTAMP '2025-06-01 10:00:00'
WHERE employee_id = 1;

-- Historique complet d'un employé
SELECT employee_id, name, salary, department, 
       row_start, row_end
FROM employees 
FOR SYSTEM_TIME ALL
WHERE employee_id = 1
ORDER BY row_start;

-- Requête sur période
SELECT * FROM employees
FOR SYSTEM_TIME BETWEEN 
  TIMESTAMP '2025-06-01' AND TIMESTAMP '2025-06-30'
WHERE department = 'IT';
```

**Architecture** :
- Table courante : données actuelles
- Table historique : versions précédentes (automatiquement créée)
- Requêtes temporelles transparentes via FOR SYSTEM_TIME

🆕 **MariaDB 11.8** : Extension TIMESTAMP 2106 (résout le problème Y2038) - les tables temporelles peuvent maintenant gérer des dates jusqu'en 2106.

💡 **Conseil** : Partitionnez la table historique par date pour optimiser les requêtes et faciliter l'archivage.

---

### 3. Application Time Period Tables 🆕

Nouveauté **MariaDB 11.8**, les **Application Time Period Tables** permettent de gérer des **périodes temporelles applicatives** avec contraintes d'intégrité.

**Différence avec System-Versioned** :
- System-Versioned : timestamps système automatiques (audit)
- Application Time Period : périodes métier définies par l'application

**Cas d'usage** :
- Réservations (chambres d'hôtel, salles de réunion)
- Contrats avec dates de validité
- Tarifications périodiques
- Planification de ressources

```sql
-- Table de réservations avec période applicative
CREATE TABLE room_bookings (
  booking_id INT PRIMARY KEY AUTO_INCREMENT,
  room_number INT,
  customer_name VARCHAR(100),
  booking_start DATE,
  booking_end DATE,
  PERIOD FOR booking_period (booking_start, booking_end),
  -- Empêche les réservations qui se chevauchent
  UNIQUE (room_number, booking_period WITHOUT OVERLAPS)
);

-- Cette insertion réussit
INSERT INTO room_bookings (room_number, customer_name, booking_start, booking_end)
VALUES (101, 'Alice', '2025-06-01', '2025-06-05');

-- Cette insertion ÉCHOUE car période se chevauche
INSERT INTO room_bookings (room_number, customer_name, booking_start, booking_end)
VALUES (101, 'Bob', '2025-06-03', '2025-06-07');
-- ERROR: Duplicate entry '101-2025-06-03-2025-06-07' for key 'room_number'

-- Cette insertion RÉUSSIT (pas de chevauchement)
INSERT INTO room_bookings (room_number, customer_name, booking_start, booking_end)
VALUES (101, 'Bob', '2025-06-05', '2025-06-10');

-- Requête : qui occupe la chambre 101 le 15 juin ?
SELECT * FROM room_bookings
WHERE room_number = 101
  AND booking_period OVERLAPS PERIOD('2025-06-15', '2025-06-15');
```

**Avantages** :
- ✅ Contraintes d'intégrité au niveau base de données
- ✅ Pas de logique applicative complexe pour gérer les chevauchements
- ✅ Performance optimale avec index appropriés
- ✅ Sémantique SQL standard (SQL:2011)

---

### 4. Colonnes Virtuelles et Générées

Les **colonnes générées** sont calculées automatiquement à partir d'autres colonnes. Deux types :

#### VIRTUAL (Calculée à la volée)
```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  price DECIMAL(10,2),
  tax_rate DECIMAL(4,2),
  -- Colonne virtuelle : calculée lors de la lecture
  price_with_tax DECIMAL(10,2) AS (price * (1 + tax_rate)) VIRTUAL
);

-- La valeur de price_with_tax n'est PAS stockée
-- Elle est calculée à chaque SELECT
```

#### STORED (Calculée et stockée)
```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  price DECIMAL(10,2),
  tax_rate DECIMAL(4,2),
  -- Colonne stockée : calculée lors de INSERT/UPDATE et stockée
  price_with_tax DECIMAL(10,2) AS (price * (1 + tax_rate)) STORED
);

-- La valeur est stockée physiquement
-- Mise à jour automatique lors de changements de price ou tax_rate
```

**Comparaison** :

| Aspect | VIRTUAL | STORED |
|--------|---------|--------|
| **Stockage** | Aucun | Espace disque utilisé |
| **Calcul** | À chaque lecture | À l'INSERT/UPDATE |
| **Performance lecture** | Plus lent (calcul) | Rapide (déjà calculé) |
| **Performance écriture** | Rapide | Plus lent (calcul + stockage) |
| **Indexation** | ✅ Possible (important !) | ✅ Possible |
| **Usage** | Colonnes rarement lues | Colonnes fréquemment lues |

**Cas d'usage avancé : Indexation de colonnes JSON** :

```sql
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  profile JSON,
  -- Extraction d'attribut JSON en colonne virtuelle
  email VARCHAR(255) AS (JSON_UNQUOTE(JSON_EXTRACT(profile, '$.email'))) VIRTUAL,
  -- Index sur colonne virtuelle !
  INDEX idx_email (email)
);

-- Requête optimisée par l'index
SELECT * FROM users WHERE email = 'alice@example.com';
-- Utilise idx_email même si email est virtuelle
```

💡 **Best practice** : 
- VIRTUAL pour colonnes rarement utilisées ou très volumineuses
- STORED pour colonnes fréquemment filtrées/triées
- Toujours créer un INDEX sur colonnes générées utilisées en WHERE/ORDER BY

---

### 5. Invisible Columns

Les **colonnes invisibles** sont présentes dans la table mais **non retournées par SELECT *** sauf mention explicite.

**Cas d'usage** :
- Migration progressive de schéma
- Ajout de colonnes sans casser les applications existantes
- Colonnes de métadonnées (audit, versioning)
- Colonnes techniques non exposées aux applications

```sql
-- Création avec colonne invisible
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customer_id INT,
  total DECIMAL(10,2),
  -- Colonne invisible pour audit
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP INVISIBLE,
  created_by VARCHAR(50) INVISIBLE
);

-- SELECT * ne retourne PAS les colonnes invisibles
SELECT * FROM orders WHERE order_id = 1;
-- Retourne : order_id, customer_id, total

-- Mention explicite pour voir les colonnes invisibles
SELECT order_id, customer_id, total, created_at, created_by 
FROM orders WHERE order_id = 1;

-- Rendre une colonne visible/invisible
ALTER TABLE orders ALTER COLUMN created_at SET VISIBLE;
ALTER TABLE orders ALTER COLUMN created_at SET INVISIBLE;
```

**Stratégie de migration sans downtime** :

```sql
-- Phase 1 : Ajouter nouvelle colonne en invisible
ALTER TABLE customers 
  ADD COLUMN email_normalized VARCHAR(255) 
  AS (LOWER(TRIM(email))) STORED INVISIBLE;

-- Phase 2 : Créer index (application non impactée)
CREATE INDEX idx_email_norm ON customers(email_normalized);

-- Phase 3 : Modifier application pour utiliser email_normalized

-- Phase 4 : Rendre visible et supprimer ancienne colonne
ALTER TABLE customers ALTER COLUMN email_normalized SET VISIBLE;
ALTER TABLE customers DROP COLUMN email; -- si applicable
```

---

### 6. Compression de Tables

MariaDB InnoDB supporte **plusieurs niveaux de compression** pour réduire l'espace disque et optimiser les I/O.

**Méthodes de compression** :

#### 6.1 ROW_FORMAT=COMPRESSED
```sql
-- Compression page-level (1K, 2K, 4K, 8K)
CREATE TABLE logs (
  log_id BIGINT PRIMARY KEY,
  message TEXT,
  timestamp TIMESTAMP
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;

-- KEY_BLOCK_SIZE : taille de page compressée
-- Plus petit = meilleure compression, mais plus de CPU
```

#### 6.2 PAGE_COMPRESSED (depuis MariaDB 10.1)
```sql
-- Compression transparente (punch hole)
CREATE TABLE archives (
  archive_id BIGINT PRIMARY KEY,
  content LONGTEXT,
  metadata JSON
) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=9;

-- LEVEL 1-9 (défaut 6)
-- Nécessite filesystem supportant punch hole (XFS, ext4)
```

**Compression vs Performance** :

| Métrique | Sans compression | COMPRESSED | PAGE_COMPRESSED |
|----------|------------------|------------|-----------------|
| **Espace disque** | 100% | 30-50% | 20-40% |
| **CPU utilisation** | Baseline | +15-25% | +10-20% |
| **I/O lectures** | Baseline | -40-60% | -50-70% |
| **I/O écritures** | Baseline | Variable | -30-50% |

**Cas d'usage** :
- ✅ Tables volumineuses peu modifiées (logs, archives)
- ✅ Données textuelles (JSON, XML, logs)
- ✅ Optimisation coûts cloud (stockage)
- ❌ Tables OLTP à forte écriture
- ❌ Données déjà compressées (images, vidéos)

```sql
-- Exemple : Table de logs avec compression
CREATE TABLE application_logs (
  log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  app_name VARCHAR(50),
  level ENUM('DEBUG','INFO','WARN','ERROR'),
  message TEXT,
  context JSON,
  created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
  INDEX idx_app_time (app_name, created_at)
) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=6;

-- Économie typique : 60-70% d'espace
-- Impact performance lectures : négligeable avec SSD
```

💡 **Best practice** : Testez la compression sur un environnement de staging avec charge réaliste avant production.

---

### 7. Encryption at Rest

L'**encryption at rest** protège les données stockées sur disque contre l'accès physique non autorisé.

**Architecture MariaDB Encryption** :
1. **Data Encryption** : Tables, tablespaces, logs
2. **Key Management** : Plugin de gestion des clés
3. **Encryption Keys** : Hiérarchie de clés (master, table, page)

#### 7.1 Configuration avec File Key Management Plugin

```ini
# my.cnf
[mysqld]
# Activer encryption
plugin-load-add=file_key_management
file-key-management-filename=/etc/mysql/encryption/keyfile.enc

# Chiffrer InnoDB
innodb-encrypt-tables=ON
innodb-encrypt-log=ON
innodb-encrypt-temporary-tables=ON

# Rotation automatique des clés
innodb-encryption-threads=4
innodb-encryption-rotate-key-age=7
```

#### 7.2 Création du fichier de clés

```bash
# Générer clé de chiffrement
openssl rand -hex 32 > /etc/mysql/encryption/keyfile

# Format du fichier :
# 1;<hex_key>
# 2;<hex_key>
echo "1;$(openssl rand -hex 32)" > /etc/mysql/encryption/keyfile.enc

# Chiffrer le keyfile avec passphrase
openssl enc -aes-256-cbc -md sha256 \
  -in /etc/mysql/encryption/keyfile \
  -out /etc/mysql/encryption/keyfile.enc

chmod 600 /etc/mysql/encryption/keyfile.enc
chown mysql:mysql /etc/mysql/encryption/keyfile.enc
```

#### 7.3 Encryption au niveau table

```sql
-- Table chiffrée
CREATE TABLE sensitive_data (
  id INT PRIMARY KEY,
  ssn VARCHAR(11),
  credit_card VARCHAR(19),
  medical_record TEXT
) ENCRYPTED=YES;

-- Vérifier le statut de chiffrement
SELECT TABLE_SCHEMA, TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_NAME = 'sensitive_data';

-- Chiffrer une table existante
ALTER TABLE customers ENCRYPTED=YES;
```

#### 7.4 Key Management Plugins

**Options disponibles** :
1. **file_key_management** : Fichier local (développement)
2. **aws_key_management** : AWS KMS (production cloud)
3. **hashicorp_key_management** : Vault (enterprise)
4. **kmip** : Key Management Interoperability Protocol

**Exemple AWS KMS** :
```ini
[mysqld]
plugin-load-add=aws_key_management
aws-key-management-master-key-id=arn:aws:kms:region:account:key/key-id
aws-key-management-region=us-east-1
```

**Impact sur performance** :
- CPU : +5-15% (chiffrement/déchiffrement)
- I/O : Impact minimal avec hardware moderne (AES-NI)
- Mémoire : Buffer pool stocke données déchiffrées

⚠️ **Sécurité** :
- Encryption at rest ≠ encryption in transit (utilisez SSL/TLS)
- Protège contre vol de disques, mais pas contre attaques logiques
- Sauvegardez les clés de chiffrement séparément !

```sql
-- Vérifier que encryption fonctionne
SHOW VARIABLES LIKE '%encrypt%';

-- Tables chiffrées
SELECT TABLE_SCHEMA, TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE CREATE_OPTIONS LIKE '%ENCRYPTED=YES%';
```

---

### 8. MariaDB Vector - Recherche Vectorielle pour l'IA 🆕

**MariaDB Vector** est LA fonctionnalité phare de MariaDB 11.8 qui positionne le SGBD dans l'écosystème de l'**Intelligence Artificielle générative**.

#### 8.1 Concept : Embeddings et Recherche Vectorielle

Les **embeddings** sont des représentations numériques (vecteurs) de données non structurées :
- Texte → Vecteur de 768, 1536, ou 3072 dimensions
- Images → Vecteur de 512, 2048 dimensions
- Audio → Vecteur de dimensions variables

**Recherche vectorielle** = Trouver les vecteurs les plus similaires dans un espace multidimensionnel.

**Applications** :
- 🔍 **Semantic Search** : Recherche par sens, pas par mots-clés
- 💬 **RAG (Retrieval Augmented Generation)** : Contexte pour LLMs
- 🎯 **Recommendation Systems** : Produits, contenus similaires
- 🚨 **Anomaly Detection** : Détecter comportements inhabituels
- 🎨 **Image Similarity** : Recherche d'images similaires

#### 8.2 Type de données VECTOR

```sql
-- Création table avec colonnes VECTOR
CREATE TABLE documents (
  doc_id INT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(255),
  content TEXT,
  -- Embedding OpenAI text-embedding-3-small (1536 dimensions)
  embedding VECTOR(1536),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertion avec vecteur
INSERT INTO documents (title, content, embedding)
VALUES (
  'Introduction to AI',
  'Artificial Intelligence is transforming...',
  VEC_FromText('[0.123, -0.456, 0.789, ..., 0.234]')  -- 1536 valeurs
);

-- Format du vecteur : tableau de FLOAT
-- Dimensions supportées : 1 à 65535
-- Optimisé en mémoire (4 bytes par dimension)
```

#### 8.3 Index HNSW (Hierarchical Navigable Small Worlds)

L'index **HNSW** permet une recherche vectorielle **ultra-rapide** avec approximation.

```sql
-- Créer index HNSW pour recherche vectorielle
CREATE INDEX idx_embedding ON documents(embedding)
  USING HNSW
  WITH (
    metric = 'COSINE',      -- Métrique de distance
    m = 16,                 -- Nombre de connexions (défaut 16)
    ef_construction = 200   -- Qualité construction (défaut 200)
  );

-- Paramètres HNSW :
-- - metric : COSINE, EUCLIDEAN, DOTPRODUCT
-- - m : Plus grand = meilleure précision, plus de mémoire (12-48)
-- - ef_construction : Plus grand = meilleure qualité, construction plus lente
```

**Caractéristiques HNSW** :
- ✅ Recherche approximative (99%+ précision)
- ✅ Requêtes en millisecondes (vs secondes scan complet)
- ✅ Scalable : millions de vecteurs
- ⚠️ Consomme mémoire (graph structure)
- ⚠️ Construction d'index peut être longue

#### 8.4 Fonctions de Distance

MariaDB 11.8 fournit plusieurs fonctions de calcul de similarité :

```sql
-- 1. COSINE Similarity (le plus courant pour texte)
-- Valeur : -1 (opposés) à 1 (identiques)
SELECT 
  doc_id, 
  title,
  VEC_DISTANCE_COSINE(embedding, @query_vector) AS similarity
FROM documents
ORDER BY similarity DESC
LIMIT 10;

-- 2. EUCLIDEAN Distance (distance L2)
-- Valeur : 0 (identiques) à +∞ (très différents)
SELECT 
  doc_id, 
  title,
  VEC_DISTANCE_EUCLIDEAN(embedding, @query_vector) AS distance
FROM documents
ORDER BY distance ASC
LIMIT 10;

-- 3. DOT PRODUCT (produit scalaire)
-- Pour vecteurs normalisés
SELECT 
  doc_id, 
  title,
  VEC_DISTANCE_DOTPRODUCT(embedding, @query_vector) AS score
FROM documents
ORDER BY score DESC
LIMIT 10;
```

**Quelle métrique choisir ?** :

| Métrique | Usage | Normalisation requise |
|----------|-------|----------------------|
| **COSINE** | Texte, NLP, semantic search | Non (gère magnitude) |
| **EUCLIDEAN** | Images, données spatiales | Recommandé |
| **DOTPRODUCT** | Performance max | Oui (vecteurs normalisés) |

#### 8.5 Fonctions de Conversion

```sql
-- VEC_FromText : Convertir texte JSON en VECTOR
SET @vec = VEC_FromText('[0.1, 0.2, 0.3, 0.4]');

-- VEC_ToText : Convertir VECTOR en texte JSON
SELECT VEC_ToText(embedding) FROM documents WHERE doc_id = 1;
-- Retourne : "[0.123, -0.456, 0.789, ...]"

-- Dimensions du vecteur
SELECT VEC_Dimensions(embedding) FROM documents WHERE doc_id = 1;
-- Retourne : 1536
```

#### 8.6 Optimisations SIMD

MariaDB Vector utilise **instructions SIMD** (Single Instruction Multiple Data) pour accélérer les calculs vectoriels :

**Support matériel** :
- ✅ **x86-64** : AVX2, AVX-512 (Intel, AMD)
- ✅ **ARM** : NEON, SVE (Apple Silicon, AWS Graviton)
- ✅ **IBM Power10** : VSX

**Gains de performance** :
- AVX2 : **4-8x** plus rapide que calcul scalaire
- AVX-512 : **8-16x** plus rapide
- ARM NEON : **4-6x** plus rapide

```sql
-- Vérifier support SIMD
SHOW VARIABLES LIKE 'vector_simd%';

-- Résultat typique sur Intel moderne :
-- vector_simd_support = AVX2,AVX512
```

💡 **Best practice** : Déployez MariaDB sur hardware moderne avec AVX2 minimum pour performances optimales.

#### 8.7 Exemple Complet : Système RAG (Retrieval Augmented Generation)

**Architecture RAG** :
1. Indexer documents dans MariaDB avec embeddings
2. Requête utilisateur → Générer embedding
3. Recherche vectorielle → Top-K documents pertinents
4. Contexte + Requête → LLM → Réponse enrichie

```sql
-- 1. Créer table pour knowledge base
CREATE TABLE knowledge_base (
  kb_id INT PRIMARY KEY AUTO_INCREMENT,
  source VARCHAR(255),
  chunk TEXT,              -- Segment de document (max 500 tokens)
  embedding VECTOR(1536),  -- OpenAI text-embedding-3-small
  metadata JSON,           -- {page, section, timestamp, ...}
  INDEX idx_vec (embedding) USING HNSW WITH (metric='COSINE')
);

-- 2. Procédure pour recherche RAG
DELIMITER $$
CREATE PROCEDURE search_knowledge(
  IN query_embedding VECTOR(1536),
  IN top_k INT
)
BEGIN
  SELECT 
    kb_id,
    source,
    chunk,
    metadata,
    VEC_DISTANCE_COSINE(embedding, query_embedding) AS relevance_score
  FROM knowledge_base
  ORDER BY relevance_score DESC
  LIMIT top_k;
END$$
DELIMITER ;

-- 3. Utilisation depuis application (pseudo-code Python)
-- from openai import OpenAI
-- client = OpenAI()
-- 
-- # Générer embedding de la requête
-- query = "How does photosynthesis work?"
-- response = client.embeddings.create(
--   model="text-embedding-3-small",
--   input=query
-- )
-- query_vec = response.data[0].embedding  # 1536 dimensions
--
-- # Recherche vectorielle dans MariaDB
-- cursor.execute("CALL search_knowledge(%s, 5)", (str(query_vec),))
-- results = cursor.fetchall()
--
-- # Construire contexte pour LLM
-- context = "\n\n".join([r['chunk'] for r in results])
--
-- # Requête LLM avec contexte
-- completion = client.chat.completions.create(
--   model="gpt-4",
--   messages=[
--     {"role": "system", "content": "Answer based on context provided."},
--     {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {query}"}
--   ]
-- )
```

#### 8.8 Intégration avec LLMs et Frameworks IA

**Frameworks supportés** :

**1. LangChain** (Python)
```python
from langchain.vectorstores import MariaDB
from langchain.embeddings import OpenAIEmbeddings

# Configuration
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector_store = MariaDB(
    connection_string="mysql://user:pass@host/db",
    embedding_function=embeddings,
    table_name="knowledge_base"
)

# Ajout de documents
vector_store.add_texts(["Doc 1", "Doc 2", "Doc 3"])

# Recherche de similarité
results = vector_store.similarity_search("query text", k=5)
```

**2. LlamaIndex**
```python
from llama_index import MariaDBVectorStore
from llama_index.embeddings import OpenAIEmbedding

embed_model = OpenAIEmbedding(model="text-embedding-3-small")
vector_store = MariaDBVectorStore(
    connection_string="mysql://user:pass@host/db",
    embed_dim=1536
)

# Indexation
index = VectorStoreIndex.from_documents(
    documents, 
    vector_store=vector_store,
    embed_model=embed_model
)

# Query
query_engine = index.as_query_engine()
response = query_engine.query("What is AI?")
```

**3. Haystack** (Deepset)
```python
from haystack.document_stores import MariaDBDocumentStore

document_store = MariaDBDocumentStore(
    host="localhost",
    username="root",
    password="password",
    database="haystack",
    embedding_dim=1536
)
```

#### 8.9 Cas d'Usage Avancés

**1. Hybrid Search (SQL + Vector)** :
```sql
-- Combiner filtres SQL classiques + recherche vectorielle
SELECT 
  doc_id,
  title,
  category,
  VEC_DISTANCE_COSINE(embedding, @query_vec) AS score
FROM documents
WHERE 
  created_at >= '2025-01-01'           -- Filtre temporel
  AND category IN ('Tech', 'Science')  -- Filtre catégoriel
  AND VEC_DISTANCE_COSINE(embedding, @query_vec) > 0.7  -- Seuil similarité
ORDER BY score DESC
LIMIT 10;
```

**2. Multi-Modal Search** :
```sql
-- Table multi-modale (texte + image)
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(255),
  description TEXT,
  image_path VARCHAR(500),
  text_embedding VECTOR(1536),    -- CLIP text embedding
  image_embedding VECTOR(512),    -- CLIP image embedding
  INDEX idx_text (text_embedding) USING HNSW,
  INDEX idx_image (image_embedding) USING HNSW
);

-- Recherche par texte OU image
SELECT product_id, name,
  GREATEST(
    VEC_DISTANCE_COSINE(text_embedding, @query_text_vec),
    VEC_DISTANCE_COSINE(image_embedding, @query_image_vec)
  ) AS max_similarity
FROM products
ORDER BY max_similarity DESC
LIMIT 20;
```

**3. Clustering et Segmentation** :
```sql
-- Identifier clusters de documents similaires
WITH document_pairs AS (
  SELECT 
    d1.doc_id AS doc1_id,
    d2.doc_id AS doc2_id,
    VEC_DISTANCE_COSINE(d1.embedding, d2.embedding) AS similarity
  FROM documents d1
  CROSS JOIN documents d2
  WHERE d1.doc_id < d2.doc_id
    AND VEC_DISTANCE_COSINE(d1.embedding, d2.embedding) > 0.8
)
SELECT doc1_id, doc2_id, similarity
FROM document_pairs
ORDER BY similarity DESC;
```

#### 8.10 Performance et Tuning

**Dimensionnement** :
```sql
-- Calcul espace mémoire pour index HNSW
-- Formule approximative : nb_vectors * dimensions * 4 bytes * factor
-- Factor ≈ 1.5-2x (overhead HNSW graph)

-- Exemple : 1M documents, 1536 dimensions
-- 1,000,000 * 1536 * 4 * 1.5 = ~9 GB de RAM pour l'index
```

**Optimisation requêtes** :
```sql
-- Paramètre ef_search (qualité vs vitesse)
-- Plus élevé = meilleure précision, plus lent
SET SESSION hnsw_ef_search = 100;  -- Défaut : 10

-- Recherche avec différents ef_search
SELECT doc_id, title,
  VEC_DISTANCE_COSINE(embedding, @query_vec) AS score
FROM documents
ORDER BY score DESC
LIMIT 10;  -- Utilise ef_search=100
```

**Benchmarks typiques** :

| Dataset | Vecteurs | Dimensions | ef_search | Latence p95 | Rappel@10 |
|---------|----------|------------|-----------|-------------|-----------|
| Small | 100K | 768 | 10 | 5ms | 95% |
| Medium | 1M | 1536 | 50 | 15ms | 98% |
| Large | 10M | 1536 | 100 | 80ms | 99% |

💡 **Best practices** :
- Commencez avec `ef_search=10`, augmentez si précision insuffisante
- Monitoring régulier de la mémoire (index HNSW réside en RAM)
- Batch les insertions (plus efficace que row-by-row)
- Partitionnez si >50M vecteurs

---

### 9. Online Schema Change (ALTER TABLE non-bloquant) 🆕

Les **modifications de schéma online** permettent d'altérer les tables sans bloquer les opérations DML en production.

**Problème des ALTER classiques** :
- Bloquent les écritures (voire lectures)
- Reconstruction complète de table
- Downtime sur tables volumineuses

**Solutions MariaDB 11.8** :

#### 9.1 ALGORITHM=INSTANT
```sql
-- Modifications instantanées (metadata only)
-- Support : Ajout colonnes, modification DEFAULT, INVISIBLE, etc.

-- Ajout colonne en fin de table (INSTANT)
ALTER TABLE users 
  ADD COLUMN registration_date DATE DEFAULT '2025-01-01',
  ALGORITHM=INSTANT;

-- Modification DEFAULT (INSTANT)
ALTER TABLE users 
  ALTER COLUMN status SET DEFAULT 'active',
  ALGORITHM=INSTANT;

-- Colonne INVISIBLE (INSTANT)
ALTER TABLE users 
  ALTER COLUMN internal_id SET INVISIBLE,
  ALGORITHM=INSTANT;
```

**Opérations INSTANT** :
- ✅ ADD COLUMN (fin de table, avec DEFAULT)
- ✅ ALTER DEFAULT
- ✅ SET/DROP INVISIBLE
- ✅ RENAME COLUMN
- ❌ Modification type de données
- ❌ Ajout index (sauf cas particuliers)

#### 9.2 ALGORITHM=INPLACE
```sql
-- Modifications sans copie de table complète
-- Écritures concurrentes permises durant l'opération

-- Ajout index (INPLACE)
ALTER TABLE orders 
  ADD INDEX idx_customer (customer_id),
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Modification longueur VARCHAR (extension)
ALTER TABLE users 
  MODIFY COLUMN email VARCHAR(320),  -- était VARCHAR(255)
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- LOCK options :
-- - NONE : Lectures ET écritures concurrentes
-- - SHARED : Lectures concurrentes, écritures bloquées
-- - EXCLUSIVE : Tout bloqué
```

🆕 **MariaDB 11.8 : Optimistic ALTER TABLE** :
- Permet réplication asynchrone de continuer durant ALTER
- Réduit le lag de réplication (secondes vs minutes)

#### 9.3 ALGORITHM=COPY (à éviter)
```sql
-- Copie complète de table (ancien comportement)
-- À éviter en production sur tables volumineuses
ALTER TABLE large_table 
  ADD COLUMN new_col INT,
  ALGORITHM=COPY;

-- Bloque toutes les écritures
-- Durée : minutes à heures selon taille
```

#### 9.4 Outils Externes pour Online DDL

**gh-ost** (GitHub Online Schema Transformation) :
```bash
# ALTER sans downtime, même sur très grandes tables
gh-ost \
  --host=localhost \
  --database=mydb \
  --table=orders \
  --alter="ADD COLUMN processed BOOLEAN DEFAULT FALSE" \
  --execute

# Fonctionnement :
# 1. Crée table ghost (orders_gho)
# 2. Copie données progressivement
# 3. Replay binlog en temps réel
# 4. Swap atomique à la fin
```

**pt-online-schema-change** (Percona Toolkit) :
```bash
# Alternative mature et éprouvée
pt-online-schema-change \
  --alter="ADD COLUMN discount DECIMAL(5,2)" \
  D=mydb,t=products \
  --execute

# Avantages :
# - Contrôle fin de la charge (--max-load)
# - Pause/Resume
# - Rollback si problème
```

#### 9.5 Stratégie de Migration Online

**Étapes recommandées** :

```sql
-- 1. Tester sur environnement de staging
ALTER TABLE test_users ADD COLUMN phone VARCHAR(20), ALGORITHM=INSTANT;
-- Vérifier : Query OK, 0 rows affected (0.001 sec)

-- 2. Vérifier capacité INSTANT/INPLACE
ALTER TABLE users ADD COLUMN phone VARCHAR(20), ALGORITHM=INSTANT;
-- Si erreur : utiliser INPLACE ou gh-ost

-- 3. Monitoring durant opération
-- Terminal 1 : Lancer ALTER
ALTER TABLE users ADD INDEX idx_created (created_at), ALGORITHM=INPLACE, LOCK=NONE;

-- Terminal 2 : Monitoring
SHOW PROCESSLIST;
SELECT * FROM information_schema.INNODB_TRX;

-- 4. Validation post-migration
SHOW CREATE TABLE users;
SELECT COUNT(*) FROM users;  -- Vérifier intégrité
```

**Limites et précautions** :
- ⚠️ INSTANT limité à certaines opérations
- ⚠️ INPLACE consomme CPU et I/O (impact performance)
- ⚠️ Espace disque : 2x taille table pour COPY/gh-ost
- ⚠️ Réplication : lag possible sur replicas

💡 **Conseil** : Préférez toujours ALGORITHM=INSTANT > INPLACE > gh-ost > COPY.

---

## ✅ Points clés à retenir

### Sequences
- ✅ Génération d'identifiants indépendants des tables
- ✅ Contrôle précis (min/max/cycle/cache)
- ✅ Partageable entre tables et applications

### Tables Temporelles
- ✅ **System-Versioned** : Audit automatique avec timestamps système
- ✅ **Application Time Period** : Périodes métier avec contraintes WITHOUT OVERLAPS
- ✅ Requêtes temporelles : FOR SYSTEM_TIME AS OF / BETWEEN
- 🆕 Extension TIMESTAMP 2106 (résolution Y2038)

### Colonnes Générées
- ✅ **VIRTUAL** : Calcul à la lecture, économie stockage
- ✅ **STORED** : Calcul à l'écriture, performance lecture
- ✅ Indexables (crucial pour performance)
- ✅ Cas d'usage : Extraction JSON, calculs dérivés

### Invisible Columns
- ✅ Migration progressive sans casser applications
- ✅ Non retournées par SELECT *
- ✅ Stratégie pour évolutions de schéma

### Compression
- ✅ **ROW_FORMAT=COMPRESSED** : Compression page-level
- ✅ **PAGE_COMPRESSED** : Compression transparente (punch hole)
- ✅ Économie 40-70% stockage
- ⚠️ Impact CPU, tester avant production

### Encryption at Rest
- ✅ Protection données sur disque
- ✅ Plusieurs plugins : file, AWS KMS, Vault
- ✅ Transparent pour applications
- ⚠️ Sauvegarder les clés séparément !

### MariaDB Vector 🆕 (Fonctionnalité phare 11.8)
- ✅ Type **VECTOR(dimensions)** natif
- ✅ Index **HNSW** ultra-rapide (ms vs secondes)
- ✅ Fonctions distance : COSINE, EUCLIDEAN, DOTPRODUCT
- ✅ Optimisations **SIMD** (AVX2, AVX-512, ARM NEON)
- ✅ Intégration **LLMs** : OpenAI, Claude, LLaMA
- ✅ Frameworks IA : **LangChain, LlamaIndex, Haystack**
- ✅ Cas d'usage : **RAG, Semantic Search, Recommendations**

### Online Schema Change
- ✅ **ALGORITHM=INSTANT** : Métadonnées uniquement (préféré)
- ✅ **ALGORITHM=INPLACE** : Sans copie complète
- ✅ **gh-ost / pt-osc** : Pour très grandes tables
- 🆕 **Optimistic ALTER** : Réplication asynchrone continue
- ⚠️ Toujours tester sur staging !

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB 11.8
- 📖 [Sequences](https://mariadb.com/kb/en/sequences/)
- 📖 [System-Versioned Tables](https://mariadb.com/kb/en/system-versioned-tables/)
- 📖 [Application Time Period Tables](https://mariadb.com/kb/en/application-time-period-tables/) 🆕
- 📖 [Generated Columns](https://mariadb.com/kb/en/generated-columns/)
- 📖 [Invisible Columns](https://mariadb.com/kb/en/invisible-columns/)
- 📖 [InnoDB Compression](https://mariadb.com/kb/en/innodb-compression/)
- 📖 [Data at Rest Encryption](https://mariadb.com/kb/en/data-at-rest-encryption/)
- 📖 [**MariaDB Vector**](https://mariadb.com/kb/en/vector-overview/) 🆕
- 📖 [HNSW Index](https://mariadb.com/kb/en/hnsw-index/) 🆕
- 📖 [ALTER TABLE Online Operations](https://mariadb.com/kb/en/alter-table/)

### Blogs et Articles
- 📝 [MariaDB Vector: AI-Powered Database](https://mariadb.com/resources/blog/mariadb-vector-ai-powered-database/) (2025)
- 📝 [Implementing RAG with MariaDB Vector](https://mariadb.com/resources/blog/rag-mariadb-vector/) (2025)
- 📝 [HNSW vs Traditional Indexes](https://mariadb.com/resources/blog/hnsw-vs-traditional-indexes/)
- 📝 [Best Practices: Online Schema Changes](https://mariadb.com/kb/en/alter-table-best-practices/)

### Outils
- 🛠️ [gh-ost](https://github.com/github/gh-ost) - GitHub Online Schema Transformation
- 🛠️ [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit) - pt-online-schema-change
- 🛠️ [LangChain](https://www.langchain.com/) - Framework LLM avec support MariaDB
- 🛠️ [LlamaIndex](https://www.llamaindex.ai/) - Data framework pour LLM applications

### Frameworks IA compatibles
- 🤖 [OpenAI API](https://platform.openai.com/docs/guides/embeddings) - Embeddings GPT
- 🤖 [Anthropic Claude](https://www.anthropic.com/api) - LLM et embeddings
- 🤖 [Haystack](https://haystack.deepset.ai/) - NLP framework

---

## ➡️ Chapitres suivants

### **Chapitre 19 : Migration et Compatibilité**
Apprenez à migrer depuis MySQL ou d'autres SGBD, gérer les versions, et assurer la compatibilité de vos applications.

### **Chapitre 20 : Cas d'Usage et Architectures**
Explorez les architectures modernes (microservices, event-driven), le data warehousing avec ColumnStore, et les **use cases IA avancés** (RAG, semantic search, hybrid search) utilisant MariaDB Vector.

---


⏭️ [Sequences (CREATE SEQUENCE)](/18-fonctionnalites-avancees/01-sequences.md)

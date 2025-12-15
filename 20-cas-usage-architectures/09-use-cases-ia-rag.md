🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.9 Use Cases IA : RAG et Recherche Vectorielle 🆕

> **Niveau** : Avancé  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : Section 12.3 (Recherche Full-Text), notions de Machine Learning et embeddings, connaissance de Python

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les architectures RAG (Retrieval-Augmented Generation) et leur intérêt
- Implémenter le stockage et la recherche de vecteurs avec MariaDB
- Concevoir des systèmes de recherche sémantique combinant SQL et embeddings
- Intégrer MariaDB avec les frameworks IA modernes (LangChain, LlamaIndex)
- Utiliser MariaDB MCP Server pour connecter Claude aux bases de données
- Construire des moteurs de recommandation basés sur la similarité vectorielle
- Combiner recherche lexicale (Full-Text) et sémantique (Hybrid Search)

---

## Introduction

L'essor des **Large Language Models (LLM)** comme Claude, GPT-4 et Llama a révolutionné l'interaction avec les données. Cependant, ces modèles ont des limitations : connaissances figées à leur date d'entraînement, hallucinations possibles, et incapacité à accéder aux données privées d'une entreprise.

**RAG (Retrieval-Augmented Generation)** résout ces problèmes en combinant la puissance des LLM avec la précision des bases de données. MariaDB, avec ses capacités Full-Text, JSON et ses nouvelles fonctionnalités vectorielles, devient un composant clé de ces architectures IA modernes.

### Pourquoi MariaDB pour l'IA ?

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    MARIADB DANS L'ÉCOSYSTÈME IA                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Avantages de MariaDB pour les workloads IA :                              │
│  ════════════════════════════════════════════                              │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. DONNÉES STRUCTURÉES + VECTEURS                                   │   │
│  │    • Stocker embeddings aux côtés des métadonnées SQL               │   │
│  │    • Jointures entre vecteurs et données relationnelles             │   │
│  │    • Filtrage SQL avant recherche vectorielle                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. RECHERCHE FULL-TEXT MATURE                                       │   │
│  │    • Recherche lexicale haute performance                           │   │
│  │    • Combinaison avec recherche sémantique (Hybrid Search)          │   │
│  │    • Support natif du français et autres langues                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. TRANSACTIONS ACID                                                │   │
│  │    • Cohérence des données (vecteurs + métadonnées)                 │   │
│  │    • Mises à jour atomiques                                         │   │
│  │    • Intégrité référentielle                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 4. ÉCOSYSTÈME EXISTANT                                              │   │
│  │    • Données déjà dans MariaDB → pas de migration                   │   │
│  │    • Outils familiers (backup, monitoring, réplication)             │   │
│  │    • Compétences équipes existantes                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 5. COÛT                                                             │   │
│  │    • Open source, pas de licence par vecteur                        │   │
│  │    • Infrastructure mutualisée (pas de DB vectorielle séparée)      │   │
│  │    • Réduction de la complexité opérationnelle                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Comparaison avec les bases vectorielles spécialisées :                    │
│  ══════════════════════════════════════════════════════                    │
│                                                                            │
│  │ Critère            │ MariaDB      │ Pinecone/Weaviate │ pgvector      │ │
│  │────────────────────│──────────────│───────────────────│───────────────│ │
│  │ Données SQL        │ ✅ Natif     │ ❌ Séparé         │ ✅ Natif      │ │
│  │ Recherche Full-Text│ ✅ Mature    │ ⚠️ Limité         │ ✅ Bon        │ │
│  │ Perf vectorielle   │ ⚠️ Modérée   │ ✅ Optimisée      │ ✅ Bonne      │ │
│  │ Index ANN          │ ⚠️ Manuel    │ ✅ Natif          │ ✅ HNSW/IVF   │ │
│  │ Coût               │ ✅ Gratuit   │ $$$ Cloud         │ ✅ Gratuit    │ │
│  │ Écosystème         │ ✅ Large     │ ⚠️ Spécialisé     │ ✅ Large      │ │
│  │                                                                       │ │
│  │ Verdict : MariaDB excellent pour < 10M vecteurs et use cases          │ │
│  │           hybrides (SQL + vecteurs + Full-Text)                       │ │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture RAG (Retrieval-Augmented Generation)

### Principe fondamental

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE RAG AVEC MARIADB                           │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  PROBLÈME DES LLM SEULS :                                                  │
│  ═══════════════════════                                                   │
│                                                                            │
│  User: "Quel est le chiffre d'affaires de notre filiale allemande ?"       │
│  LLM:  "Je n'ai pas accès à vos données internes." ❌                      │
│        ou pire : invente un chiffre (hallucination) ⚠️                     │
│                                                                            │
│  SOLUTION RAG :                                                            │
│  ══════════════                                                            │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         USER QUERY                                  │   │
│  │  "Quel est le chiffre d'affaires de notre filiale allemande ?"      │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      1. EMBEDDING MODEL                             │   │
│  │                   (OpenAI, Voyage, Sentence-BERT)                   │   │
│  │                                                                     │   │
│  │  Query → [0.023, -0.156, 0.892, ..., 0.034]  (1536 dimensions)      │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      2. RETRIEVAL (MariaDB)                         │   │
│  │                                                                     │   │
│  │  SELECT content, metadata                                           │   │
│  │  FROM documents                                                     │   │
│  │  WHERE category = 'finance'  -- Filtrage SQL                        │   │
│  │  ORDER BY cosine_similarity(embedding, query_vector) DESC           │   │
│  │  LIMIT 5;                                                           │   │
│  │                                                                     │   │
│  │  Résultats :                                                        │   │
│  │  • "Rapport Q3 2024 : CA Allemagne = 45M€..."                       │   │
│  │  • "Prévisions 2025 filiale DE : croissance 12%..."                 │   │
│  │  • "Comparatif filiales Europe : DE leader avec..."                 │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      3. AUGMENTED PROMPT                            │   │
│  │                                                                     │   │
│  │  System: "Tu es un assistant financier. Utilise UNIQUEMENT les      │   │
│  │           informations fournies pour répondre."                     │   │
│  │                                                                     │   │
│  │  Context:                                                           │   │
│  │  """                                                                │   │
│  │  [Document 1] Rapport Q3 2024 : CA Allemagne = 45M€...              │   │
│  │  [Document 2] Prévisions 2025 filiale DE : croissance 12%...        │   │
│  │  [Document 3] Comparatif filiales Europe...                         │   │
│  │  """                                                                │   │
│  │                                                                     │   │
│  │  Question: "Quel est le chiffre d'affaires de notre filiale         │   │
│  │            allemande ?"                                             │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         4. LLM (Claude)                             │   │
│  │                                                                     │   │
│  │  "D'après le rapport Q3 2024, le chiffre d'affaires de votre        │   │
│  │   filiale allemande est de 45 millions d'euros. Les prévisions      │   │
│  │   pour 2025 indiquent une croissance de 12%." ✅                    │   │
│  │                                                                     │   │
│  │  [Sources: Rapport Q3 2024, Prévisions 2025]                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Composants clés

| Composant | Rôle | Options |
|-----------|------|---------|
| **Embedding Model** | Convertit le texte en vecteurs | OpenAI `text-embedding-3-small`, Voyage AI, Sentence-BERT, Cohere |
| **Vector Store** | Stocke et recherche les embeddings | MariaDB (cette section), pgvector, Pinecone, Weaviate |
| **Retriever** | Trouve les documents pertinents | Similarity search, Hybrid (BM25 + vecteurs) |
| **LLM** | Génère la réponse finale | Claude, GPT-4, Llama, Mistral |
| **Orchestrator** | Coordonne le pipeline | LangChain, LlamaIndex, custom |

---

## Stockage des vecteurs dans MariaDB

### Schéma de base

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SCHÉMA POUR STOCKAGE DE VECTEURS DANS MARIADB
-- ═══════════════════════════════════════════════════════════════════════════

CREATE DATABASE vector_store CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE vector_store;

-- Table principale des documents
CREATE TABLE documents (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    -- Contenu
    content TEXT NOT NULL,
    content_hash CHAR(64) AS (SHA2(content, 256)) STORED,  -- Déduplication
    
    -- Métadonnées structurées
    title VARCHAR(500),
    source VARCHAR(255),          -- Fichier, URL, etc.
    doc_type VARCHAR(50),         -- 'pdf', 'html', 'markdown', etc.
    category VARCHAR(100),
    language CHAR(2) DEFAULT 'fr',
    
    -- Métadonnées JSON flexibles
    metadata JSON,
    
    -- Chunking info
    chunk_index INT DEFAULT 0,    -- Position du chunk dans le document
    parent_doc_id BIGINT,         -- Document parent si chunked
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    indexed_at TIMESTAMP,         -- Quand le vecteur a été généré
    
    -- Index
    UNIQUE KEY uk_content_hash (content_hash),
    INDEX idx_category (category),
    INDEX idx_source (source),
    INDEX idx_type (doc_type),
    INDEX idx_parent (parent_doc_id),
    FULLTEXT INDEX ft_content (content, title)  -- Pour hybrid search
) ENGINE=InnoDB;

-- Table des embeddings (séparée pour flexibilité)
CREATE TABLE embeddings (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    document_id BIGINT NOT NULL,
    
    -- Vecteur (stocké en BLOB pour flexibilité)
    embedding BLOB NOT NULL,        -- Vecteur sérialisé (float32 array)
    
    -- Métadonnées du modèle
    model_name VARCHAR(100) NOT NULL,  -- 'text-embedding-3-small', etc.
    model_version VARCHAR(50),
    dimensions INT NOT NULL,        -- 1536, 768, etc.
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Contraintes
    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE,
    UNIQUE KEY uk_doc_model (document_id, model_name),
    INDEX idx_model (model_name)
) ENGINE=InnoDB;

-- Vue pratique pour les requêtes
CREATE VIEW v_documents_with_embeddings AS
SELECT 
    d.id,
    d.content,
    d.title,
    d.source,
    d.category,
    d.metadata,
    e.embedding,
    e.model_name,
    e.dimensions
FROM documents d
JOIN embeddings e ON d.id = e.document_id;

-- ═══════════════════════════════════════════════════════════════════════════
-- ALTERNATIVE : Vecteur dans la même table (plus simple)
-- ═══════════════════════════════════════════════════════════════════════════

CREATE TABLE documents_simple (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    content TEXT NOT NULL,
    embedding BLOB,                 -- Vecteur directement dans la table
    embedding_model VARCHAR(100),
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FULLTEXT INDEX ft_content (content)
) ENGINE=InnoDB;
```

### Fonctions de similarité vectorielle

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- FONCTIONS POUR CALCUL DE SIMILARITÉ
-- ═══════════════════════════════════════════════════════════════════════════

DELIMITER //

-- Cosine Similarity (la plus utilisée pour les embeddings normalisés)
-- Formule : cos(θ) = (A·B) / (||A|| × ||B||)
-- Pour embeddings normalisés : cos(θ) = A·B (produit scalaire)
CREATE FUNCTION cosine_similarity(vec1 BLOB, vec2 BLOB)
RETURNS DOUBLE
DETERMINISTIC
NO SQL
BEGIN
    DECLARE dot_product DOUBLE DEFAULT 0;
    DECLARE norm1 DOUBLE DEFAULT 0;
    DECLARE norm2 DOUBLE DEFAULT 0;
    DECLARE i INT DEFAULT 0;
    DECLARE dim INT;
    DECLARE v1 DOUBLE;
    DECLARE v2 DOUBLE;
    
    -- Dimensions (assume float32 = 4 bytes)
    SET dim = LENGTH(vec1) / 4;
    
    -- Calcul itératif (lent en SQL pur, voir alternative Python)
    WHILE i < dim DO
        -- Extraction des floats (little-endian)
        SET v1 = CAST(CONV(HEX(REVERSE(SUBSTRING(vec1, i*4+1, 4))), 16, 10) AS DOUBLE);
        SET v2 = CAST(CONV(HEX(REVERSE(SUBSTRING(vec2, i*4+1, 4))), 16, 10) AS DOUBLE);
        
        SET dot_product = dot_product + (v1 * v2);
        SET norm1 = norm1 + (v1 * v1);
        SET norm2 = norm2 + (v2 * v2);
        SET i = i + 1;
    END WHILE;
    
    IF norm1 = 0 OR norm2 = 0 THEN
        RETURN 0;
    END IF;
    
    RETURN dot_product / (SQRT(norm1) * SQRT(norm2));
END //

-- Euclidean Distance (L2)
CREATE FUNCTION euclidean_distance(vec1 BLOB, vec2 BLOB)
RETURNS DOUBLE
DETERMINISTIC
NO SQL
BEGIN
    DECLARE sum_sq DOUBLE DEFAULT 0;
    DECLARE i INT DEFAULT 0;
    DECLARE dim INT;
    DECLARE v1 DOUBLE;
    DECLARE v2 DOUBLE;
    
    SET dim = LENGTH(vec1) / 4;
    
    WHILE i < dim DO
        SET v1 = CAST(CONV(HEX(REVERSE(SUBSTRING(vec1, i*4+1, 4))), 16, 10) AS DOUBLE);
        SET v2 = CAST(CONV(HEX(REVERSE(SUBSTRING(vec2, i*4+1, 4))), 16, 10) AS DOUBLE);
        
        SET sum_sq = sum_sq + POW(v1 - v2, 2);
        SET i = i + 1;
    END WHILE;
    
    RETURN SQRT(sum_sq);
END //

DELIMITER ;

-- ═══════════════════════════════════════════════════════════════════════════
-- NOTE IMPORTANTE : Performance
-- ═══════════════════════════════════════════════════════════════════════════
-- 
-- Les fonctions SQL ci-dessus sont LENTES pour de gros volumes.
-- Pour la production, utiliser :
-- 1. UDF C/C++ pour calcul vectoriel optimisé
-- 2. Calcul côté application (Python/NumPy)
-- 3. Table de distances pré-calculées (pour petits datasets)
-- 4. Index ANN (Approximate Nearest Neighbors) via extensions
--
-- Voir la section "Optimisation" plus bas pour les solutions performantes.
```

### Implémentation Python performante

```python
#!/usr/bin/env python3
"""
vector_store.py
Stockage et recherche de vecteurs avec MariaDB
"""

import mariadb
import numpy as np
from typing import List, Dict, Any, Optional, Tuple
import json
import hashlib
from dataclasses import dataclass
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@dataclass
class Document:
    """Représente un document avec son embedding"""
    id: Optional[int]
    content: str
    title: Optional[str] = None
    source: Optional[str] = None
    category: Optional[str] = None
    metadata: Optional[Dict] = None
    embedding: Optional[np.ndarray] = None


class MariaDBVectorStore:
    """
    Vector Store implémenté sur MariaDB.
    Optimisé pour < 10M vecteurs avec filtrage SQL.
    """
    
    def __init__(self, connection_config: Dict[str, Any], 
                 embedding_model: str = "text-embedding-3-small",
                 dimensions: int = 1536):
        """
        Initialise le vector store.
        
        Args:
            connection_config: Configuration de connexion MariaDB
            embedding_model: Nom du modèle d'embedding
            dimensions: Dimensions des vecteurs
        """
        self.connection_config = connection_config
        self.embedding_model = embedding_model
        self.dimensions = dimensions
        self._conn = None
    
    @property
    def conn(self):
        if self._conn is None or not self._conn.open:
            self._conn = mariadb.connect(**self.connection_config)
        return self._conn
    
    def _serialize_vector(self, vector: np.ndarray) -> bytes:
        """Sérialise un vecteur numpy en bytes (float32)"""
        return vector.astype(np.float32).tobytes()
    
    def _deserialize_vector(self, blob: bytes) -> np.ndarray:
        """Désérialise un blob en vecteur numpy"""
        return np.frombuffer(blob, dtype=np.float32)
    
    def add_documents(self, documents: List[Document]) -> List[int]:
        """
        Ajoute des documents avec leurs embeddings.
        
        Args:
            documents: Liste de documents avec embeddings
            
        Returns:
            Liste des IDs insérés
        """
        cursor = self.conn.cursor()
        inserted_ids = []
        
        try:
            for doc in documents:
                # Insérer le document
                cursor.execute("""
                    INSERT INTO documents 
                    (content, title, source, category, metadata, indexed_at)
                    VALUES (%s, %s, %s, %s, %s, NOW())
                    ON DUPLICATE KEY UPDATE
                        title = VALUES(title),
                        source = VALUES(source),
                        category = VALUES(category),
                        metadata = VALUES(metadata),
                        updated_at = NOW()
                """, (
                    doc.content,
                    doc.title,
                    doc.source,
                    doc.category,
                    json.dumps(doc.metadata) if doc.metadata else None
                ))
                
                doc_id = cursor.lastrowid
                
                # Insérer l'embedding
                if doc.embedding is not None:
                    embedding_blob = self._serialize_vector(doc.embedding)
                    cursor.execute("""
                        INSERT INTO embeddings 
                        (document_id, embedding, model_name, dimensions)
                        VALUES (%s, %s, %s, %s)
                        ON DUPLICATE KEY UPDATE
                            embedding = VALUES(embedding),
                            created_at = NOW()
                    """, (
                        doc_id,
                        embedding_blob,
                        self.embedding_model,
                        self.dimensions
                    ))
                
                inserted_ids.append(doc_id)
            
            self.conn.commit()
            logger.info(f"Inserted {len(inserted_ids)} documents")
            return inserted_ids
            
        except Exception as e:
            self.conn.rollback()
            logger.error(f"Error inserting documents: {e}")
            raise
        finally:
            cursor.close()
    
    def similarity_search(
        self,
        query_vector: np.ndarray,
        k: int = 5,
        category: Optional[str] = None,
        source: Optional[str] = None,
        min_similarity: float = 0.0,
        metadata_filter: Optional[Dict] = None
    ) -> List[Tuple[Document, float]]:
        """
        Recherche les documents les plus similaires.
        
        Args:
            query_vector: Vecteur de requête
            k: Nombre de résultats
            category: Filtrer par catégorie
            source: Filtrer par source
            min_similarity: Similarité minimum
            metadata_filter: Filtre JSON sur metadata
            
        Returns:
            Liste de (document, score) triés par similarité décroissante
        """
        cursor = self.conn.cursor(dictionary=True)
        
        # Construction de la requête avec filtres
        where_clauses = ["1=1"]
        params = []
        
        if category:
            where_clauses.append("d.category = %s")
            params.append(category)
        
        if source:
            where_clauses.append("d.source = %s")
            params.append(source)
        
        if metadata_filter:
            for key, value in metadata_filter.items():
                where_clauses.append(f"JSON_EXTRACT(d.metadata, '$.{key}') = %s")
                params.append(json.dumps(value))
        
        where_clause = " AND ".join(where_clauses)
        
        # Récupérer tous les candidats (avec filtres SQL)
        cursor.execute(f"""
            SELECT 
                d.id, d.content, d.title, d.source, d.category, d.metadata,
                e.embedding
            FROM documents d
            JOIN embeddings e ON d.id = e.document_id
            WHERE e.model_name = %s AND {where_clause}
        """, [self.embedding_model] + params)
        
        results = cursor.fetchall()
        cursor.close()
        
        if not results:
            return []
        
        # Calcul des similarités côté Python (NumPy optimisé)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        scored_results = []
        for row in results:
            doc_vector = self._deserialize_vector(row['embedding'])
            doc_norm = doc_vector / np.linalg.norm(doc_vector)
            
            # Cosine similarity = dot product des vecteurs normalisés
            similarity = float(np.dot(query_norm, doc_norm))
            
            if similarity >= min_similarity:
                doc = Document(
                    id=row['id'],
                    content=row['content'],
                    title=row['title'],
                    source=row['source'],
                    category=row['category'],
                    metadata=json.loads(row['metadata']) if row['metadata'] else None,
                    embedding=doc_vector
                )
                scored_results.append((doc, similarity))
        
        # Trier par similarité décroissante
        scored_results.sort(key=lambda x: x[1], reverse=True)
        
        return scored_results[:k]
    
    def hybrid_search(
        self,
        query_text: str,
        query_vector: np.ndarray,
        k: int = 5,
        vector_weight: float = 0.7,
        fulltext_weight: float = 0.3,
        **filters
    ) -> List[Tuple[Document, float]]:
        """
        Recherche hybride combinant Full-Text et similarité vectorielle.
        
        Args:
            query_text: Texte de recherche pour Full-Text
            query_vector: Vecteur de requête
            k: Nombre de résultats
            vector_weight: Poids de la recherche vectorielle (0-1)
            fulltext_weight: Poids de la recherche Full-Text (0-1)
            **filters: Filtres SQL additionnels
            
        Returns:
            Liste de (document, score) avec score hybride
        """
        cursor = self.conn.cursor(dictionary=True)
        
        # Construire les filtres
        where_clauses = ["1=1"]
        params = [query_text]  # Pour MATCH AGAINST
        
        for key, value in filters.items():
            if value is not None:
                where_clauses.append(f"d.{key} = %s")
                params.append(value)
        
        where_clause = " AND ".join(where_clauses)
        
        # Récupérer avec score Full-Text
        cursor.execute(f"""
            SELECT 
                d.id, d.content, d.title, d.source, d.category, d.metadata,
                e.embedding,
                MATCH(d.content, d.title) AGAINST(%s IN NATURAL LANGUAGE MODE) AS ft_score
            FROM documents d
            JOIN embeddings e ON d.id = e.document_id
            WHERE e.model_name = %s AND {where_clause}
            HAVING ft_score > 0 OR 1=1  -- Inclure même sans match FT
        """, params + [self.embedding_model] + params[1:])
        
        results = cursor.fetchall()
        cursor.close()
        
        if not results:
            return []
        
        # Calcul des scores hybrides
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        # Normaliser les scores Full-Text
        ft_scores = [row['ft_score'] for row in results]
        max_ft = max(ft_scores) if ft_scores and max(ft_scores) > 0 else 1
        
        scored_results = []
        for row in results:
            # Score vectoriel
            doc_vector = self._deserialize_vector(row['embedding'])
            doc_norm = doc_vector / np.linalg.norm(doc_vector)
            vector_score = float(np.dot(query_norm, doc_norm))
            
            # Score Full-Text normalisé
            ft_score = row['ft_score'] / max_ft if max_ft > 0 else 0
            
            # Score hybride pondéré
            hybrid_score = (vector_weight * vector_score) + (fulltext_weight * ft_score)
            
            doc = Document(
                id=row['id'],
                content=row['content'],
                title=row['title'],
                source=row['source'],
                category=row['category'],
                metadata=json.loads(row['metadata']) if row['metadata'] else None
            )
            scored_results.append((doc, hybrid_score))
        
        scored_results.sort(key=lambda x: x[1], reverse=True)
        return scored_results[:k]
    
    def delete_documents(self, document_ids: List[int]):
        """Supprime des documents et leurs embeddings"""
        cursor = self.conn.cursor()
        try:
            # CASCADE supprime automatiquement les embeddings
            cursor.execute(
                "DELETE FROM documents WHERE id IN (%s)" % 
                ','.join(['%s'] * len(document_ids)),
                document_ids
            )
            self.conn.commit()
            logger.info(f"Deleted {cursor.rowcount} documents")
        finally:
            cursor.close()
    
    def get_stats(self) -> Dict[str, Any]:
        """Retourne des statistiques sur le vector store"""
        cursor = self.conn.cursor(dictionary=True)
        
        cursor.execute("""
            SELECT 
                COUNT(DISTINCT d.id) AS document_count,
                COUNT(e.id) AS embedding_count,
                COUNT(DISTINCT d.category) AS category_count,
                COUNT(DISTINCT d.source) AS source_count,
                AVG(LENGTH(d.content)) AS avg_content_length,
                MIN(d.created_at) AS oldest_document,
                MAX(d.indexed_at) AS latest_indexing
            FROM documents d
            LEFT JOIN embeddings e ON d.id = e.document_id
        """)
        
        stats = cursor.fetchone()
        cursor.close()
        
        return stats


# ═══════════════════════════════════════════════════════════════════════════
# EXEMPLE D'UTILISATION
# ═══════════════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    import openai
    
    # Configuration
    db_config = {
        'host': 'localhost',
        'port': 3306,
        'user': 'vector_user',
        'password': 'secure_password',
        'database': 'vector_store'
    }
    
    # Initialiser le vector store
    store = MariaDBVectorStore(db_config)
    
    # Fonction pour générer des embeddings
    def get_embedding(text: str) -> np.ndarray:
        response = openai.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return np.array(response.data[0].embedding)
    
    # Ajouter des documents
    docs = [
        Document(
            content="MariaDB est un système de gestion de base de données relationnelle.",
            title="Introduction à MariaDB",
            category="documentation",
            metadata={"lang": "fr", "version": "11.8"}
        ),
        Document(
            content="Les embeddings permettent de représenter le texte sous forme vectorielle.",
            title="Embeddings et IA",
            category="tutoriel",
            metadata={"lang": "fr", "topic": "ml"}
        )
    ]
    
    # Générer les embeddings
    for doc in docs:
        doc.embedding = get_embedding(doc.content)
    
    # Insérer
    ids = store.add_documents(docs)
    print(f"Documents insérés : {ids}")
    
    # Rechercher
    query = "Comment fonctionne une base de données ?"
    query_vec = get_embedding(query)
    
    results = store.similarity_search(query_vec, k=3)
    for doc, score in results:
        print(f"[{score:.4f}] {doc.title}: {doc.content[:100]}...")
```

---

## Intégration avec LangChain

### Custom Vector Store LangChain

```python
#!/usr/bin/env python3
"""
langchain_mariadb.py
Intégration de MariaDB comme Vector Store pour LangChain
"""

from typing import List, Optional, Any, Tuple, Dict, Type
from langchain_core.documents import Document
from langchain_core.embeddings import Embeddings
from langchain_core.vectorstores import VectorStore
import numpy as np
import mariadb
import json


class MariaDBVectorStore(VectorStore):
    """
    LangChain Vector Store backed by MariaDB.
    
    Exemple d'utilisation:
        embeddings = OpenAIEmbeddings()
        vectorstore = MariaDBVectorStore(
            connection_config={"host": "localhost", ...},
            embedding_function=embeddings,
            table_name="documents"
        )
        
        # Ajouter des documents
        vectorstore.add_texts(["Hello world", "Goodbye world"])
        
        # Rechercher
        results = vectorstore.similarity_search("Hello", k=3)
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        embedding_function: Embeddings,
        table_name: str = "documents",
        content_column: str = "content",
        embedding_column: str = "embedding",
        metadata_column: str = "metadata",
    ):
        self.connection_config = connection_config
        self._embedding_function = embedding_function
        self.table_name = table_name
        self.content_column = content_column
        self.embedding_column = embedding_column
        self.metadata_column = metadata_column
        self._conn = None
    
    @property
    def embeddings(self) -> Embeddings:
        return self._embedding_function
    
    @property
    def conn(self):
        if self._conn is None or not self._conn.open:
            self._conn = mariadb.connect(**self.connection_config)
        return self._conn
    
    def add_texts(
        self,
        texts: List[str],
        metadatas: Optional[List[dict]] = None,
        **kwargs: Any,
    ) -> List[str]:
        """Ajoute des textes avec leurs embeddings"""
        
        # Générer les embeddings
        embeddings = self._embedding_function.embed_documents(texts)
        
        cursor = self.conn.cursor()
        ids = []
        
        try:
            for i, (text, embedding) in enumerate(zip(texts, embeddings)):
                metadata = metadatas[i] if metadatas else {}
                
                # Sérialiser l'embedding
                embedding_blob = np.array(embedding, dtype=np.float32).tobytes()
                
                cursor.execute(f"""
                    INSERT INTO {self.table_name} 
                    ({self.content_column}, {self.embedding_column}, {self.metadata_column})
                    VALUES (%s, %s, %s)
                """, (text, embedding_blob, json.dumps(metadata)))
                
                ids.append(str(cursor.lastrowid))
            
            self.conn.commit()
            return ids
            
        finally:
            cursor.close()
    
    def similarity_search(
        self,
        query: str,
        k: int = 4,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> List[Document]:
        """Recherche par similarité"""
        
        results = self.similarity_search_with_score(query, k, filter, **kwargs)
        return [doc for doc, _ in results]
    
    def similarity_search_with_score(
        self,
        query: str,
        k: int = 4,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> List[Tuple[Document, float]]:
        """Recherche par similarité avec scores"""
        
        # Générer l'embedding de la requête
        query_embedding = self._embedding_function.embed_query(query)
        query_vector = np.array(query_embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        cursor = self.conn.cursor(dictionary=True)
        
        # Construire la requête avec filtres
        where_clause = "1=1"
        params = []
        
        if filter:
            for key, value in filter.items():
                where_clause += f" AND JSON_EXTRACT({self.metadata_column}, '$.{key}') = %s"
                params.append(json.dumps(value))
        
        cursor.execute(f"""
            SELECT id, {self.content_column}, {self.embedding_column}, {self.metadata_column}
            FROM {self.table_name}
            WHERE {where_clause}
        """, params)
        
        rows = cursor.fetchall()
        cursor.close()
        
        # Calculer les similarités
        results = []
        for row in rows:
            doc_vector = np.frombuffer(row[self.embedding_column], dtype=np.float32)
            doc_norm = doc_vector / np.linalg.norm(doc_vector)
            similarity = float(np.dot(query_norm, doc_norm))
            
            metadata = json.loads(row[self.metadata_column]) if row[self.metadata_column] else {}
            doc = Document(
                page_content=row[self.content_column],
                metadata={**metadata, "id": row["id"]}
            )
            results.append((doc, similarity))
        
        # Trier et limiter
        results.sort(key=lambda x: x[1], reverse=True)
        return results[:k]
    
    @classmethod
    def from_texts(
        cls,
        texts: List[str],
        embedding: Embeddings,
        metadatas: Optional[List[dict]] = None,
        connection_config: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> "MariaDBVectorStore":
        """Crée un vector store à partir de textes"""
        
        if connection_config is None:
            raise ValueError("connection_config is required")
        
        store = cls(
            connection_config=connection_config,
            embedding_function=embedding,
            **kwargs
        )
        store.add_texts(texts, metadatas)
        return store
    
    def delete(self, ids: Optional[List[str]] = None, **kwargs: Any) -> Optional[bool]:
        """Supprime des documents par ID"""
        if not ids:
            return True
        
        cursor = self.conn.cursor()
        try:
            placeholders = ','.join(['%s'] * len(ids))
            cursor.execute(f"DELETE FROM {self.table_name} WHERE id IN ({placeholders})", ids)
            self.conn.commit()
            return True
        finally:
            cursor.close()


# ═══════════════════════════════════════════════════════════════════════════
# PIPELINE RAG COMPLET AVEC LANGCHAIN
# ═══════════════════════════════════════════════════════════════════════════

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough


def create_rag_chain(vectorstore: MariaDBVectorStore, model_name: str = "gpt-4"):
    """
    Crée une chaîne RAG complète.
    
    Args:
        vectorstore: Vector store MariaDB
        model_name: Modèle LLM à utiliser
        
    Returns:
        Chaîne LangChain pour RAG
    """
    
    # Retriever
    retriever = vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={"k": 5}
    )
    
    # Prompt RAG
    template = """Tu es un assistant expert qui répond aux questions en utilisant 
uniquement le contexte fourni. Si tu ne trouves pas la réponse dans le contexte, 
dis-le clairement.

Contexte:
{context}

Question: {question}

Réponds de manière précise et concise, en citant les sources quand c'est pertinent."""

    prompt = ChatPromptTemplate.from_template(template)
    
    # LLM
    llm = ChatOpenAI(model=model_name, temperature=0)
    
    # Formater les documents
    def format_docs(docs):
        return "\n\n".join([
            f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
            for doc in docs
        ])
    
    # Chaîne RAG
    rag_chain = (
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | prompt
        | llm
        | StrOutputParser()
    )
    
    return rag_chain


# Exemple d'utilisation
if __name__ == "__main__":
    # Configuration
    db_config = {
        'host': 'localhost',
        'port': 3306,
        'user': 'rag_user',
        'password': 'secure_password',
        'database': 'rag_db'
    }
    
    # Créer le vector store
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    vectorstore = MariaDBVectorStore(
        connection_config=db_config,
        embedding_function=embeddings
    )
    
    # Ajouter des documents
    docs = [
        "MariaDB 11.8 LTS introduit de nouvelles fonctionnalités pour l'IA.",
        "Les embeddings sont stockés sous forme de BLOB dans MariaDB.",
        "La recherche vectorielle permet de trouver des documents sémantiquement similaires."
    ]
    vectorstore.add_texts(docs, metadatas=[{"source": "doc_mariadb"} for _ in docs])
    
    # Créer la chaîne RAG
    rag_chain = create_rag_chain(vectorstore)
    
    # Poser une question
    response = rag_chain.invoke("Comment stocker des embeddings dans MariaDB ?")
    print(response)
```

---

## Intégration avec LlamaIndex

```python
#!/usr/bin/env python3
"""
llamaindex_mariadb.py
Intégration de MariaDB avec LlamaIndex
"""

from llama_index.core import (
    VectorStoreIndex,
    SimpleDirectoryReader,
    StorageContext,
    ServiceContext,
    Document as LlamaDocument
)
from llama_index.core.vector_stores.types import (
    VectorStore,
    VectorStoreQuery,
    VectorStoreQueryResult,
)
from llama_index.core.schema import TextNode, NodeWithScore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
from typing import List, Any, Optional, Dict
import numpy as np
import mariadb
import json


class MariaDBVectorStore(VectorStore):
    """
    LlamaIndex Vector Store backed by MariaDB.
    """
    
    stores_text: bool = True
    flat_metadata: bool = False
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        table_name: str = "llama_nodes",
        embed_dim: int = 1536,
    ):
        self.connection_config = connection_config
        self.table_name = table_name
        self.embed_dim = embed_dim
        self._conn = None
        self._ensure_table()
    
    @property
    def conn(self):
        if self._conn is None or not self._conn.open:
            self._conn = mariadb.connect(**self.connection_config)
        return self._conn
    
    def _ensure_table(self):
        """Crée la table si elle n'existe pas"""
        cursor = self.conn.cursor()
        cursor.execute(f"""
            CREATE TABLE IF NOT EXISTS {self.table_name} (
                id VARCHAR(255) PRIMARY KEY,
                text TEXT NOT NULL,
                embedding BLOB,
                metadata JSON,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FULLTEXT INDEX ft_text (text)
            ) ENGINE=InnoDB
        """)
        self.conn.commit()
        cursor.close()
    
    def add(self, nodes: List[TextNode], **kwargs: Any) -> List[str]:
        """Ajoute des nodes au vector store"""
        cursor = self.conn.cursor()
        ids = []
        
        try:
            for node in nodes:
                embedding_blob = None
                if node.embedding is not None:
                    embedding_blob = np.array(node.embedding, dtype=np.float32).tobytes()
                
                cursor.execute(f"""
                    INSERT INTO {self.table_name} (id, text, embedding, metadata)
                    VALUES (%s, %s, %s, %s)
                    ON DUPLICATE KEY UPDATE
                        text = VALUES(text),
                        embedding = VALUES(embedding),
                        metadata = VALUES(metadata)
                """, (
                    node.node_id,
                    node.get_content(),
                    embedding_blob,
                    json.dumps(node.metadata) if node.metadata else None
                ))
                ids.append(node.node_id)
            
            self.conn.commit()
            return ids
            
        finally:
            cursor.close()
    
    def delete(self, ref_doc_id: str, **kwargs: Any) -> None:
        """Supprime les nodes d'un document"""
        cursor = self.conn.cursor()
        cursor.execute(f"""
            DELETE FROM {self.table_name} 
            WHERE JSON_EXTRACT(metadata, '$.ref_doc_id') = %s
        """, (json.dumps(ref_doc_id),))
        self.conn.commit()
        cursor.close()
    
    def query(self, query: VectorStoreQuery, **kwargs: Any) -> VectorStoreQueryResult:
        """Recherche par similarité"""
        
        if query.query_embedding is None:
            return VectorStoreQueryResult(nodes=[], similarities=[], ids=[])
        
        query_vector = np.array(query.query_embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        cursor = self.conn.cursor(dictionary=True)
        
        # Construire les filtres
        where_clause = "embedding IS NOT NULL"
        params = []
        
        if query.filters is not None:
            for f in query.filters.filters:
                where_clause += f" AND JSON_EXTRACT(metadata, '$.{f.key}') = %s"
                params.append(json.dumps(f.value))
        
        cursor.execute(f"""
            SELECT id, text, embedding, metadata
            FROM {self.table_name}
            WHERE {where_clause}
        """, params)
        
        rows = cursor.fetchall()
        cursor.close()
        
        # Calculer les similarités
        results = []
        for row in rows:
            doc_vector = np.frombuffer(row['embedding'], dtype=np.float32)
            doc_norm = doc_vector / np.linalg.norm(doc_vector)
            similarity = float(np.dot(query_norm, doc_norm))
            
            node = TextNode(
                id_=row['id'],
                text=row['text'],
                metadata=json.loads(row['metadata']) if row['metadata'] else {}
            )
            results.append((node, similarity, row['id']))
        
        # Trier et limiter
        results.sort(key=lambda x: x[1], reverse=True)
        k = query.similarity_top_k or 10
        results = results[:k]
        
        nodes = [NodeWithScore(node=r[0], score=r[1]) for r in results]
        
        return VectorStoreQueryResult(
            nodes=nodes,
            similarities=[r[1] for r in results],
            ids=[r[2] for r in results]
        )


# ═══════════════════════════════════════════════════════════════════════════
# EXEMPLE : INDEXATION DE DOCUMENTS PDF
# ═══════════════════════════════════════════════════════════════════════════

def index_documents(
    documents_path: str,
    db_config: Dict[str, Any],
    chunk_size: int = 512,
    chunk_overlap: int = 50
):
    """
    Indexe des documents dans MariaDB avec LlamaIndex.
    
    Args:
        documents_path: Chemin vers les documents
        db_config: Configuration MariaDB
        chunk_size: Taille des chunks
        chunk_overlap: Overlap entre chunks
    """
    from llama_index.core.node_parser import SentenceSplitter
    
    # Charger les documents
    documents = SimpleDirectoryReader(documents_path).load_data()
    print(f"Loaded {len(documents)} documents")
    
    # Parser en chunks
    parser = SentenceSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap
    )
    nodes = parser.get_nodes_from_documents(documents)
    print(f"Created {len(nodes)} chunks")
    
    # Embedding model
    embed_model = OpenAIEmbedding(model="text-embedding-3-small")
    
    # Vector store MariaDB
    vector_store = MariaDBVectorStore(
        connection_config=db_config,
        table_name="document_chunks"
    )
    
    # Créer le storage context
    storage_context = StorageContext.from_defaults(vector_store=vector_store)
    
    # Créer l'index
    index = VectorStoreIndex(
        nodes=nodes,
        storage_context=storage_context,
        embed_model=embed_model,
        show_progress=True
    )
    
    print("Indexing complete!")
    return index


def query_index(db_config: Dict[str, Any], question: str):
    """
    Interroge l'index avec une question.
    """
    # Recréer le vector store et l'index
    vector_store = MariaDBVectorStore(
        connection_config=db_config,
        table_name="document_chunks"
    )
    
    embed_model = OpenAIEmbedding(model="text-embedding-3-small")
    llm = OpenAI(model="gpt-4", temperature=0)
    
    index = VectorStoreIndex.from_vector_store(
        vector_store=vector_store,
        embed_model=embed_model
    )
    
    # Créer le query engine
    query_engine = index.as_query_engine(
        llm=llm,
        similarity_top_k=5
    )
    
    # Poser la question
    response = query_engine.query(question)
    
    print(f"Question: {question}")
    print(f"Response: {response}")
    print(f"\nSources:")
    for node in response.source_nodes:
        print(f"  - [{node.score:.4f}] {node.text[:100]}...")
    
    return response


if __name__ == "__main__":
    db_config = {
        'host': 'localhost',
        'port': 3306,
        'user': 'llama_user',
        'password': 'secure_password',
        'database': 'llama_db'
    }
    
    # Indexer des documents
    # index = index_documents("./documents", db_config)
    
    # Interroger
    response = query_index(db_config, "Quelles sont les nouvelles fonctionnalités de MariaDB 11.8 ?")
```

---

## MariaDB MCP Server 🆕

### Introduction au Model Context Protocol

Le **Model Context Protocol (MCP)** est un protocole standardisé permettant aux LLM (comme Claude) d'interagir avec des sources de données externes. MariaDB MCP Server permet à Claude d'exécuter des requêtes SQL directement.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    MARIADB MCP SERVER ARCHITECTURE                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          Claude Desktop                             │   │
│  │                                                                     │   │
│  │  User: "Quels sont les 10 clients avec le plus de commandes ?"      │   │
│  │                                                                     │   │
│  │  Claude: Je vais interroger la base de données...                   │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  │ MCP Protocol                            │
│                                  │                                         │
│  ┌───────────────────────────────▼─────────────────────────────────────┐   │
│  │                       MariaDB MCP Server                            │   │
│  │                                                                     │   │
│  │  Tools exposés :                                                    │   │
│  │  • query(sql) - Exécuter une requête SELECT                         │   │
│  │  • describe_table(table) - Décrire une table                        │   │
│  │  • list_tables() - Lister les tables                                │   │
│  │  • get_schema() - Obtenir le schéma complet                         │   │
│  │                                                                     │   │
│  │  Resources exposées :                                               │   │
│  │  • mariadb://schema - Schéma de la base                             │   │
│  │  • mariadb://tables/{name} - Définition d'une table                 │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  │ SQL (read-only)                         │
│                                  │                                         │
│  ┌───────────────────────────────▼─────────────────────────────────────┐   │
│  │                          MariaDB Server                             │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  ecommerce                                                  │    │   │
│  │  │  ├── customers (10K rows)                                   │    │   │
│  │  │  ├── orders (100K rows)                                     │    │   │
│  │  │  ├── products (5K rows)                                     │    │   │
│  │  │  └── ...                                                    │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Sécurité :                                                                │
│  • Utilisateur read-only dédié                                             │
│  • Whitelist de requêtes (optionnel)                                       │
│  • Rate limiting                                                           │
│  • Audit logging                                                           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Implémentation MCP Server

```python
#!/usr/bin/env python3
"""
mariadb_mcp_server.py
Serveur MCP pour MariaDB - Permet à Claude d'interroger des bases de données
"""

import asyncio
import json
import logging
from typing import Any, Dict, List, Optional
import mariadb
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    TextContent,
    Resource,
    ResourceContents,
)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class MariaDBMCPServer:
    """
    Serveur MCP exposant MariaDB à Claude.
    """
    
    def __init__(self, connection_config: Dict[str, Any]):
        self.connection_config = connection_config
        self.server = Server("mariadb-mcp")
        self._conn = None
        self._setup_handlers()
    
    @property
    def conn(self):
        if self._conn is None or not self._conn.open:
            self._conn = mariadb.connect(**self.connection_config)
        return self._conn
    
    def _setup_handlers(self):
        """Configure les handlers MCP"""
        
        # Handler pour lister les tools
        @self.server.list_tools()
        async def list_tools() -> List[Tool]:
            return [
                Tool(
                    name="query",
                    description="Execute a read-only SQL query on the MariaDB database. Only SELECT statements are allowed.",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "sql": {
                                "type": "string",
                                "description": "The SQL SELECT query to execute"
                            },
                            "limit": {
                                "type": "integer",
                                "description": "Maximum number of rows to return (default: 100)",
                                "default": 100
                            }
                        },
                        "required": ["sql"]
                    }
                ),
                Tool(
                    name="describe_table",
                    description="Get the structure of a specific table",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "table_name": {
                                "type": "string",
                                "description": "Name of the table to describe"
                            }
                        },
                        "required": ["table_name"]
                    }
                ),
                Tool(
                    name="list_tables",
                    description="List all tables in the database",
                    inputSchema={
                        "type": "object",
                        "properties": {}
                    }
                ),
                Tool(
                    name="get_sample_data",
                    description="Get sample rows from a table",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "table_name": {
                                "type": "string",
                                "description": "Name of the table"
                            },
                            "limit": {
                                "type": "integer",
                                "description": "Number of rows to return",
                                "default": 5
                            }
                        },
                        "required": ["table_name"]
                    }
                )
            ]
        
        # Handler pour exécuter les tools
        @self.server.call_tool()
        async def call_tool(name: str, arguments: Dict[str, Any]) -> List[TextContent]:
            try:
                if name == "query":
                    result = self._execute_query(
                        arguments["sql"],
                        arguments.get("limit", 100)
                    )
                elif name == "describe_table":
                    result = self._describe_table(arguments["table_name"])
                elif name == "list_tables":
                    result = self._list_tables()
                elif name == "get_sample_data":
                    result = self._get_sample_data(
                        arguments["table_name"],
                        arguments.get("limit", 5)
                    )
                else:
                    result = f"Unknown tool: {name}"
                
                return [TextContent(type="text", text=result)]
                
            except Exception as e:
                logger.error(f"Error executing tool {name}: {e}")
                return [TextContent(type="text", text=f"Error: {str(e)}")]
        
        # Handler pour les resources
        @self.server.list_resources()
        async def list_resources() -> List[Resource]:
            return [
                Resource(
                    uri="mariadb://schema",
                    name="Database Schema",
                    description="Complete schema of the database",
                    mimeType="application/json"
                )
            ]
        
        @self.server.read_resource()
        async def read_resource(uri: str) -> ResourceContents:
            if uri == "mariadb://schema":
                schema = self._get_full_schema()
                return ResourceContents(
                    uri=uri,
                    mimeType="application/json",
                    text=json.dumps(schema, indent=2)
                )
            raise ValueError(f"Unknown resource: {uri}")
    
    def _execute_query(self, sql: str, limit: int) -> str:
        """Exécute une requête SELECT (read-only)"""
        
        # Validation de sécurité
        sql_upper = sql.strip().upper()
        if not sql_upper.startswith("SELECT"):
            return "Error: Only SELECT queries are allowed"
        
        # Mots-clés dangereux
        dangerous_keywords = ["INSERT", "UPDATE", "DELETE", "DROP", "CREATE", 
                            "ALTER", "TRUNCATE", "GRANT", "REVOKE"]
        for keyword in dangerous_keywords:
            if keyword in sql_upper:
                return f"Error: {keyword} is not allowed"
        
        # Ajouter LIMIT si absent
        if "LIMIT" not in sql_upper:
            sql = f"{sql.rstrip(';')} LIMIT {limit}"
        
        cursor = self.conn.cursor(dictionary=True)
        try:
            cursor.execute(sql)
            rows = cursor.fetchall()
            
            # Formater le résultat
            if not rows:
                return "Query returned no results"
            
            # Convertir en format lisible
            result = f"Returned {len(rows)} rows:\n\n"
            result += json.dumps(rows, indent=2, default=str)
            
            return result
            
        finally:
            cursor.close()
    
    def _describe_table(self, table_name: str) -> str:
        """Décrit la structure d'une table"""
        cursor = self.conn.cursor(dictionary=True)
        try:
            cursor.execute(f"DESCRIBE `{table_name}`")
            columns = cursor.fetchall()
            
            cursor.execute(f"SHOW INDEX FROM `{table_name}`")
            indexes = cursor.fetchall()
            
            result = f"Table: {table_name}\n\n"
            result += "Columns:\n"
            for col in columns:
                result += f"  - {col['Field']}: {col['Type']}"
                if col['Null'] == 'NO':
                    result += " NOT NULL"
                if col['Key'] == 'PRI':
                    result += " (PRIMARY KEY)"
                elif col['Key'] == 'UNI':
                    result += " (UNIQUE)"
                result += "\n"
            
            if indexes:
                result += "\nIndexes:\n"
                for idx in indexes:
                    result += f"  - {idx['Key_name']}: {idx['Column_name']}\n"
            
            return result
            
        finally:
            cursor.close()
    
    def _list_tables(self) -> str:
        """Liste toutes les tables"""
        cursor = self.conn.cursor()
        try:
            cursor.execute("SHOW TABLES")
            tables = [row[0] for row in cursor.fetchall()]
            
            result = f"Found {len(tables)} tables:\n\n"
            for table in tables:
                cursor.execute(f"SELECT COUNT(*) FROM `{table}`")
                count = cursor.fetchone()[0]
                result += f"  - {table} ({count:,} rows)\n"
            
            return result
            
        finally:
            cursor.close()
    
    def _get_sample_data(self, table_name: str, limit: int) -> str:
        """Récupère des données d'exemple"""
        cursor = self.conn.cursor(dictionary=True)
        try:
            cursor.execute(f"SELECT * FROM `{table_name}` LIMIT {limit}")
            rows = cursor.fetchall()
            
            if not rows:
                return f"Table {table_name} is empty"
            
            return json.dumps(rows, indent=2, default=str)
            
        finally:
            cursor.close()
    
    def _get_full_schema(self) -> Dict[str, Any]:
        """Récupère le schéma complet"""
        cursor = self.conn.cursor(dictionary=True)
        schema = {"tables": {}}
        
        try:
            cursor.execute("SHOW TABLES")
            tables = [row[list(row.keys())[0]] for row in cursor.fetchall()]
            
            for table in tables:
                cursor.execute(f"DESCRIBE `{table}`")
                columns = cursor.fetchall()
                
                schema["tables"][table] = {
                    "columns": [
                        {
                            "name": col["Field"],
                            "type": col["Type"],
                            "nullable": col["Null"] == "YES",
                            "key": col["Key"],
                            "default": col["Default"]
                        }
                        for col in columns
                    ]
                }
            
            return schema
            
        finally:
            cursor.close()
    
    async def run(self):
        """Démarre le serveur MCP"""
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream,
                write_stream,
                self.server.create_initialization_options()
            )


# Configuration pour Claude Desktop
MCP_CONFIG = """
{
  "mcpServers": {
    "mariadb": {
      "command": "python",
      "args": ["path/to/mariadb_mcp_server.py"],
      "env": {
        "MARIADB_HOST": "localhost",
        "MARIADB_PORT": "3306",
        "MARIADB_USER": "mcp_readonly",
        "MARIADB_PASSWORD": "secure_password",
        "MARIADB_DATABASE": "ecommerce"
      }
    }
  }
}
"""


if __name__ == "__main__":
    import os
    
    config = {
        'host': os.environ.get('MARIADB_HOST', 'localhost'),
        'port': int(os.environ.get('MARIADB_PORT', 3306)),
        'user': os.environ.get('MARIADB_USER', 'mcp_readonly'),
        'password': os.environ.get('MARIADB_PASSWORD', ''),
        'database': os.environ.get('MARIADB_DATABASE', 'test')
    }
    
    server = MariaDBMCPServer(config)
    asyncio.run(server.run())
```

### Configuration sécurisée pour MCP

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- UTILISATEUR MARIADB POUR MCP (READ-ONLY)
-- ═══════════════════════════════════════════════════════════════════════════

-- Créer un utilisateur avec permissions minimales
CREATE USER 'mcp_readonly'@'localhost' IDENTIFIED BY 'secure_mcp_password';

-- Permissions lecture seule
GRANT SELECT ON ecommerce.* TO 'mcp_readonly'@'localhost';

-- Permettre de voir le schéma
GRANT SHOW VIEW ON ecommerce.* TO 'mcp_readonly'@'localhost';

-- Pas de permissions sensibles
-- NO GRANT for: INSERT, UPDATE, DELETE, DROP, CREATE, ALTER

-- Limites de ressources
ALTER USER 'mcp_readonly'@'localhost' 
    WITH MAX_QUERIES_PER_HOUR 1000
         MAX_CONNECTIONS_PER_HOUR 100
         MAX_USER_CONNECTIONS 5;

FLUSH PRIVILEGES;
```

---

## Moteurs de recommandation

### Architecture de recommandation basée sur les embeddings

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- SYSTÈME DE RECOMMANDATION PRODUITS
-- ═══════════════════════════════════════════════════════════════════════════

CREATE DATABASE recommendations;
USE recommendations;

-- Produits avec leurs embeddings
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2),
    
    -- Embedding de la description
    description_embedding BLOB,
    
    -- Embedding combiné (description + catégorie + attributs)
    combined_embedding BLOB,
    
    -- Attributs pour filtrage
    attributes JSON,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_category (category),
    FULLTEXT INDEX ft_search (name, description)
) ENGINE=InnoDB;

-- Historique des vues/achats
CREATE TABLE user_interactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    interaction_type ENUM('view', 'cart', 'purchase', 'like', 'review'),
    interaction_score DECIMAL(3,2),  -- 0.1 pour view, 1.0 pour purchase
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user (user_id, created_at),
    INDEX idx_product (product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
) ENGINE=InnoDB;

-- Embeddings utilisateurs (basés sur leurs interactions)
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    
    -- Embedding moyen des produits consultés/achetés (pondéré)
    preference_embedding BLOB,
    
    -- Catégories préférées
    category_preferences JSON,
    
    -- Dernière mise à jour
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Cache des recommandations (optionnel, pour performance)
CREATE TABLE recommendation_cache (
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    score DECIMAL(5,4),
    reason VARCHAR(100),  -- 'similar_to_viewed', 'popular_in_category', etc.
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    
    PRIMARY KEY (user_id, product_id),
    INDEX idx_user_score (user_id, score DESC),
    INDEX idx_expires (expires_at)
) ENGINE=InnoDB;
```

### Implémentation Python du moteur de recommandation

```python
#!/usr/bin/env python3
"""
recommendation_engine.py
Moteur de recommandation basé sur MariaDB et embeddings
"""

import numpy as np
import mariadb
from typing import List, Dict, Tuple, Optional
import json
from dataclasses import dataclass
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@dataclass
class Product:
    id: int
    name: str
    description: str
    category: str
    price: float
    embedding: Optional[np.ndarray] = None


@dataclass  
class Recommendation:
    product: Product
    score: float
    reason: str


class RecommendationEngine:
    """
    Moteur de recommandation hybride combinant:
    - Similarité vectorielle (content-based)
    - Collaborative filtering (basé sur les interactions)
    - Filtrage par attributs (SQL)
    """
    
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self._conn = None
    
    @property
    def conn(self):
        if self._conn is None or not self._conn.open:
            self._conn = mariadb.connect(**self.db_config)
        return self._conn
    
    def get_similar_products(
        self,
        product_id: int,
        k: int = 10,
        category_filter: Optional[str] = None,
        price_range: Optional[Tuple[float, float]] = None
    ) -> List[Recommendation]:
        """
        Trouve des produits similaires à un produit donné.
        
        Args:
            product_id: ID du produit de référence
            k: Nombre de recommandations
            category_filter: Filtrer par catégorie
            price_range: (min, max) prix
            
        Returns:
            Liste de recommandations triées par similarité
        """
        cursor = self.conn.cursor(dictionary=True)
        
        # Récupérer l'embedding du produit source
        cursor.execute(
            "SELECT combined_embedding FROM products WHERE product_id = %s",
            (product_id,)
        )
        result = cursor.fetchone()
        if not result or not result['combined_embedding']:
            return []
        
        source_embedding = np.frombuffer(result['combined_embedding'], dtype=np.float32)
        source_norm = source_embedding / np.linalg.norm(source_embedding)
        
        # Construire la requête avec filtres
        where_clauses = ["product_id != %s", "combined_embedding IS NOT NULL"]
        params = [product_id]
        
        if category_filter:
            where_clauses.append("category = %s")
            params.append(category_filter)
        
        if price_range:
            where_clauses.append("price BETWEEN %s AND %s")
            params.extend(price_range)
        
        where_clause = " AND ".join(where_clauses)
        
        cursor.execute(f"""
            SELECT product_id, name, description, category, price, combined_embedding
            FROM products
            WHERE {where_clause}
        """, params)
        
        candidates = cursor.fetchall()
        cursor.close()
        
        # Calculer les similarités
        recommendations = []
        for row in candidates:
            target_embedding = np.frombuffer(row['combined_embedding'], dtype=np.float32)
            target_norm = target_embedding / np.linalg.norm(target_embedding)
            similarity = float(np.dot(source_norm, target_norm))
            
            product = Product(
                id=row['product_id'],
                name=row['name'],
                description=row['description'],
                category=row['category'],
                price=float(row['price'])
            )
            recommendations.append(Recommendation(
                product=product,
                score=similarity,
                reason="similar_product"
            ))
        
        # Trier par score
        recommendations.sort(key=lambda x: x.score, reverse=True)
        return recommendations[:k]
    
    def get_personalized_recommendations(
        self,
        user_id: int,
        k: int = 10,
        diversity_factor: float = 0.3
    ) -> List[Recommendation]:
        """
        Génère des recommandations personnalisées pour un utilisateur.
        
        Combine:
        - Similarité avec le profil utilisateur
        - Produits populaires dans les catégories préférées
        - Diversification pour éviter la répétition
        
        Args:
            user_id: ID de l'utilisateur
            k: Nombre de recommandations
            diversity_factor: 0-1, favorise la diversité vs pertinence
            
        Returns:
            Liste de recommandations personnalisées
        """
        cursor = self.conn.cursor(dictionary=True)
        
        # Récupérer le profil utilisateur
        cursor.execute(
            "SELECT preference_embedding, category_preferences FROM user_profiles WHERE user_id = %s",
            (user_id,)
        )
        profile = cursor.fetchone()
        
        if not profile or not profile['preference_embedding']:
            # Fallback : recommandations populaires
            return self._get_popular_products(k)
        
        user_embedding = np.frombuffer(profile['preference_embedding'], dtype=np.float32)
        user_norm = user_embedding / np.linalg.norm(user_embedding)
        
        category_prefs = json.loads(profile['category_preferences']) if profile['category_preferences'] else {}
        
        # Récupérer les produits déjà vus
        cursor.execute("""
            SELECT DISTINCT product_id FROM user_interactions
            WHERE user_id = %s AND created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
        """, (user_id,))
        seen_products = {row['product_id'] for row in cursor.fetchall()}
        
        # Récupérer tous les produits candidats
        cursor.execute("""
            SELECT product_id, name, description, category, price, combined_embedding
            FROM products
            WHERE combined_embedding IS NOT NULL
        """)
        candidates = cursor.fetchall()
        cursor.close()
        
        # Calculer les scores combinés
        recommendations = []
        for row in candidates:
            if row['product_id'] in seen_products:
                continue
            
            target_embedding = np.frombuffer(row['combined_embedding'], dtype=np.float32)
            target_norm = target_embedding / np.linalg.norm(target_embedding)
            
            # Score de similarité
            similarity = float(np.dot(user_norm, target_norm))
            
            # Bonus catégorie préférée
            category_bonus = category_prefs.get(row['category'], 0) * 0.2
            
            # Score final
            final_score = similarity + category_bonus
            
            product = Product(
                id=row['product_id'],
                name=row['name'],
                description=row['description'],
                category=row['category'],
                price=float(row['price'])
            )
            recommendations.append(Recommendation(
                product=product,
                score=final_score,
                reason="personalized"
            ))
        
        # Trier par score
        recommendations.sort(key=lambda x: x.score, reverse=True)
        
        # Diversification : ne pas retourner trop de produits de la même catégorie
        if diversity_factor > 0:
            recommendations = self._diversify(recommendations, k, diversity_factor)
        
        return recommendations[:k]
    
    def _diversify(
        self,
        recommendations: List[Recommendation],
        k: int,
        diversity_factor: float
    ) -> List[Recommendation]:
        """Diversifie les recommandations par catégorie"""
        
        result = []
        category_counts = {}
        max_per_category = max(1, int(k * (1 - diversity_factor)))
        
        for rec in recommendations:
            cat = rec.product.category
            if category_counts.get(cat, 0) < max_per_category:
                result.append(rec)
                category_counts[cat] = category_counts.get(cat, 0) + 1
            
            if len(result) >= k:
                break
        
        return result
    
    def _get_popular_products(self, k: int) -> List[Recommendation]:
        """Fallback : produits populaires"""
        cursor = self.conn.cursor(dictionary=True)
        
        cursor.execute("""
            SELECT p.product_id, p.name, p.description, p.category, p.price,
                   COUNT(i.id) as interaction_count
            FROM products p
            LEFT JOIN user_interactions i ON p.product_id = i.product_id
            WHERE i.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
            GROUP BY p.product_id
            ORDER BY interaction_count DESC
            LIMIT %s
        """, (k,))
        
        recommendations = []
        for row in cursor.fetchall():
            product = Product(
                id=row['product_id'],
                name=row['name'],
                description=row['description'],
                category=row['category'],
                price=float(row['price'])
            )
            recommendations.append(Recommendation(
                product=product,
                score=row['interaction_count'] / 100,  # Normaliser
                reason="popular"
            ))
        
        cursor.close()
        return recommendations
    
    def update_user_profile(self, user_id: int):
        """
        Met à jour le profil utilisateur basé sur ses interactions récentes.
        Calcule un embedding moyen pondéré par le type d'interaction.
        """
        cursor = self.conn.cursor(dictionary=True)
        
        # Récupérer les interactions avec embeddings
        cursor.execute("""
            SELECT p.combined_embedding, i.interaction_type, i.interaction_score
            FROM user_interactions i
            JOIN products p ON i.product_id = p.product_id
            WHERE i.user_id = %s 
              AND p.combined_embedding IS NOT NULL
              AND i.created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)
            ORDER BY i.created_at DESC
            LIMIT 100
        """, (user_id,))
        
        interactions = cursor.fetchall()
        
        if not interactions:
            cursor.close()
            return
        
        # Calculer l'embedding moyen pondéré
        weighted_sum = None
        total_weight = 0
        
        weights = {
            'purchase': 1.0,
            'cart': 0.5,
            'like': 0.4,
            'review': 0.3,
            'view': 0.1
        }
        
        category_counts = {}
        
        for row in interactions:
            embedding = np.frombuffer(row['combined_embedding'], dtype=np.float32)
            weight = weights.get(row['interaction_type'], 0.1)
            
            if weighted_sum is None:
                weighted_sum = embedding * weight
            else:
                weighted_sum += embedding * weight
            total_weight += weight
        
        if total_weight > 0:
            preference_embedding = weighted_sum / total_weight
            embedding_blob = preference_embedding.astype(np.float32).tobytes()
            
            # Calculer les préférences de catégorie
            cursor.execute("""
                SELECT p.category, SUM(i.interaction_score) as score
                FROM user_interactions i
                JOIN products p ON i.product_id = p.product_id
                WHERE i.user_id = %s
                GROUP BY p.category
                ORDER BY score DESC
            """, (user_id,))
            
            categories = {row['category']: float(row['score']) for row in cursor.fetchall()}
            
            # Normaliser
            max_score = max(categories.values()) if categories else 1
            categories = {k: v/max_score for k, v in categories.items()}
            
            # Upsert le profil
            cursor.execute("""
                INSERT INTO user_profiles (user_id, preference_embedding, category_preferences)
                VALUES (%s, %s, %s)
                ON DUPLICATE KEY UPDATE
                    preference_embedding = VALUES(preference_embedding),
                    category_preferences = VALUES(category_preferences),
                    updated_at = NOW()
            """, (user_id, embedding_blob, json.dumps(categories)))
            
            self.conn.commit()
            logger.info(f"Updated profile for user {user_id}")
        
        cursor.close()
```

---

## ✅ Points clés à retenir

- **RAG (Retrieval-Augmented Generation)** combine la puissance des LLM avec la précision des bases de données — MariaDB stocke le contexte, le LLM génère la réponse
- **Les embeddings** sont stockés en BLOB dans MariaDB — le calcul de similarité est fait côté application (Python/NumPy) pour les performances
- **Hybrid Search** combine Full-Text (lexical) et similarité vectorielle (sémantique) pour de meilleurs résultats
- **LangChain et LlamaIndex** s'intègrent facilement avec MariaDB via des custom Vector Stores
- **MariaDB MCP Server** permet à Claude d'interroger directement les bases de données — configuration read-only obligatoire
- **Les moteurs de recommandation** utilisent les embeddings produits + profils utilisateurs pour des suggestions personnalisées
- **Pour < 10M vecteurs**, MariaDB est une excellente option combinant SQL + vecteurs + Full-Text dans une seule base
- **Pour > 10M vecteurs** avec recherche temps réel, considérer une base vectorielle spécialisée (Pinecone, Weaviate) en complément

---

## 🔗 Ressources et références

- [📖 LangChain Documentation](https://python.langchain.com/docs/)
- [📖 LlamaIndex Documentation](https://docs.llamaindex.ai/)
- [📖 Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [📖 OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [📖 RAG Best Practices](https://www.anthropic.com/research/retrieval-augmented-generation)
- [📖 MariaDB Full-Text Search](https://mariadb.com/kb/en/full-text-indexes/)

---

## ➡️ Section suivante

Cette section se décline en une sous-section détaillée :

| Section | Titre | Focus |
|---------|-------|-------|
| 20.9.1 | [Semantic Search](./09.1-semantic-search.md) | Implémentation avancée, optimisation, benchmarks |

⏭️ [Semantic Search](/20-cas-usage-architectures/09.1-semantic-search.md)

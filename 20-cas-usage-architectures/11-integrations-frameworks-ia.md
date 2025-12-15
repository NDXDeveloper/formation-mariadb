🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.11 Intégrations Frameworks IA (LangChain, LlamaIndex, etc.) 🆕

> **Niveau** : Avancé  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : Section 20.9 (RAG), Section 20.10 (MCP Server), Python avancé, notions de LLM

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Intégrer MariaDB comme vector store dans LangChain
- Construire des pipelines RAG avec LlamaIndex et MariaDB
- Implémenter des agents SQL intelligents avec accès à MariaDB
- Créer des chaînes conversationnelles avec mémoire persistée
- Utiliser Haystack et Semantic Kernel avec MariaDB
- Optimiser les performances des applications IA en production
- Déployer des solutions IA/MariaDB scalables

---

## Introduction

Les frameworks d'orchestration IA comme **LangChain**, **LlamaIndex**, **Haystack** et **Semantic Kernel** simplifient considérablement le développement d'applications basées sur les LLM. L'intégration de MariaDB avec ces frameworks permet de combiner la puissance des modèles de langage avec la fiabilité et les performances d'une base de données relationnelle éprouvée.

### Vue d'ensemble des frameworks

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    FRAMEWORKS IA ET MARIADB                                │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         APPLICATION IA                              │   │
│  │                                                                     │   │
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │   │
│  │  │  Chatbots     │ │  Agents SQL   │ │  RAG Apps     │              │   │
│  │  │  Q&A Systems  │ │  Data Analysts│ │  Search       │              │   │
│  │  └───────────────┘ └───────────────┘ └───────────────┘              │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    FRAMEWORKS D'ORCHESTRATION                       │   │
│  │                                                                     │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │
│  │  │ LangChain   │ │ LlamaIndex  │ │  Haystack   │ │  Semantic   │    │   │
│  │  │             │ │             │ │             │ │  Kernel     │    │   │
│  │  │ • Chains    │ │ • Indexing  │ │ • Pipelines │ │ • Plugins   │    │   │
│  │  │ • Agents    │ │ • Querying  │ │ • Nodes     │ │ • Planners  │    │   │
│  │  │ • Memory    │ │ • Storage   │ │ • Retrievers│ │ • Memory    │    │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│         ┌────────────────────────┼────────────────────────┐                │
│         │                        │                        │                │
│         ▼                        ▼                        ▼                │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐     │
│  │  Vector Store   │      │   SQL Store     │      │  Memory Store   │     │
│  │  (Embeddings)   │      │  (Structured)   │      │ (Conversations) │     │
│  └────────┬────────┘      └────────┬────────┘      └────────┬────────┘     │
│           │                        │                        │              │
│           └────────────────────────┼────────────────────────┘              │
│                                    │                                       │
│                                    ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                           MariaDB                                   │   │
│  │                                                                     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │  documents  │  │  embeddings │  │  chat_hist  │  │  business   │ │   │
│  │  │  (content)  │  │  (vectors)  │  │  (memory)   │  │  (data)     │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Comparaison des frameworks

| Framework | Forces | Use Cases | Intégration MariaDB |
|-----------|--------|-----------|---------------------|
| **LangChain** | Flexibilité, agents, chains composables | Chatbots, agents autonomes, RAG | Vector Store, SQL Agent, Memory |
| **LlamaIndex** | Indexation optimisée, query engines | RAG avancé, knowledge bases | Vector Store, SQL integration |
| **Haystack** | Pipelines modulaires, production-ready | Search, Q&A, NER | Document Store, Retriever |
| **Semantic Kernel** | Intégration Microsoft, planification | Enterprise apps, automation | Memory, plugins custom |

---

## LangChain : Intégration complète

### Architecture LangChain + MariaDB

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    LANGCHAIN + MARIADB ARCHITECTURE                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      LangChain Components                           │   │
│  │                                                                     │   │
│  │  ┌──────────────────────────────────────────────────────────────┐   │   │
│  │  │                      CHAINS                                  │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │   │
│  │  │  │   LLMChain  │  │   RAG Chain │  │ SQL Chain   │           │   │   │
│  │  │  │             │  │             │  │             │           │   │   │
│  │  │  │ Prompt +    │  │ Retrieval + │  │ NL → SQL    │           │   │   │
│  │  │  │ LLM         │  │ Generation  │  │ Execution   │           │   │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │   │
│  │  └──────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  ┌──────────────────────────────────────────────────────────────┐   │   │
│  │  │                      AGENTS                                  │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │   │
│  │  │  │  SQL Agent  │  │  ReAct      │  │  Tool-using │           │   │   │
│  │  │  │             │  │  Agent      │  │  Agent      │           │   │   │
│  │  │  │ Query DB    │  │ Reasoning   │  │ Multi-tool  │           │   │   │
│  │  │  │ Autonomously│  │ + Acting    │  │ Orchestr.   │           │   │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │   │
│  │  └──────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  ┌──────────────────────────────────────────────────────────────┐   │   │
│  │  │                    MEMORY                                    │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │   │
│  │  │  │  Buffer     │  │  Summary    │  │  Entity     │           │   │   │
│  │  │  │  Memory     │  │  Memory     │  │  Memory     │           │   │   │
│  │  │  │             │  │             │  │             │           │   │   │
│  │  │  │ Last N msgs │  │ Compressed  │  │ Key facts   │           │   │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │   │
│  │  └──────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    MariaDB Integrations                             │   │
│  │                                                                     │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │   │
│  │  │ MariaDBVector   │  │ SQLDatabase     │  │ MariaDBChat     │      │   │
│  │  │ Store           │  │ + SQLChain      │  │ MessageHistory  │      │   │
│  │  │                 │  │                 │  │                 │      │   │
│  │  │ - embeddings    │  │ - schema info   │  │ - session_id    │      │   │
│  │  │ - similarity    │  │ - query exec    │  │ - messages      │      │   │
│  │  │ - metadata      │  │ - results       │  │ - timestamps    │      │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Vector Store LangChain complet

```python
#!/usr/bin/env python3
"""
langchain_mariadb_vectorstore.py
Vector Store MariaDB complet pour LangChain
"""

from typing import Any, Dict, Iterable, List, Optional, Tuple, Type
from langchain_core.documents import Document
from langchain_core.embeddings import Embeddings
from langchain_core.vectorstores import VectorStore
import numpy as np
import mariadb
import json
import hashlib
from datetime import datetime
import logging

logger = logging.getLogger(__name__)


class MariaDBVectorStore(VectorStore):
    """
    LangChain Vector Store backed by MariaDB.
    
    Features:
    - Cosine similarity search
    - Metadata filtering
    - Hybrid search (Full-Text + Vector)
    - Batch operations
    - Connection pooling
    
    Example:
        from langchain_openai import OpenAIEmbeddings
        
        vectorstore = MariaDBVectorStore(
            connection_config={
                "host": "localhost",
                "port": 3306,
                "user": "langchain",
                "password": "password",
                "database": "vectordb"
            },
            embedding=OpenAIEmbeddings(),
            table_name="documents"
        )
        
        # Add documents
        vectorstore.add_texts(["Hello world", "Goodbye world"])
        
        # Search
        results = vectorstore.similarity_search("Hello", k=3)
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        embedding: Embeddings,
        table_name: str = "langchain_documents",
        content_column: str = "content",
        embedding_column: str = "embedding",
        metadata_column: str = "metadata",
        create_table: bool = True,
        pool_size: int = 5,
    ):
        """
        Initialize the MariaDB Vector Store.
        
        Args:
            connection_config: MariaDB connection parameters
            embedding: Embedding function
            table_name: Name of the table to store documents
            content_column: Column name for document content
            embedding_column: Column name for embeddings
            metadata_column: Column name for metadata (JSON)
            create_table: Whether to create the table if it doesn't exist
            pool_size: Connection pool size
        """
        self.connection_config = connection_config
        self._embedding = embedding
        self.table_name = table_name
        self.content_column = content_column
        self.embedding_column = embedding_column
        self.metadata_column = metadata_column
        
        # Initialize connection pool
        self._pool = mariadb.ConnectionPool(
            pool_name="langchain_pool",
            pool_size=pool_size,
            **connection_config
        )
        
        if create_table:
            self._create_table()
    
    @property
    def embeddings(self) -> Embeddings:
        return self._embedding
    
    def _get_connection(self) -> mariadb.Connection:
        """Get a connection from the pool."""
        return self._pool.get_connection()
    
    def _create_table(self):
        """Create the documents table if it doesn't exist."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute(f"""
                CREATE TABLE IF NOT EXISTS {self.table_name} (
                    id BIGINT PRIMARY KEY AUTO_INCREMENT,
                    {self.content_column} TEXT NOT NULL,
                    {self.embedding_column} BLOB,
                    {self.metadata_column} JSON,
                    content_hash CHAR(64) AS (SHA2({self.content_column}, 256)) STORED,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                    
                    UNIQUE KEY uk_content_hash (content_hash),
                    FULLTEXT INDEX ft_content ({self.content_column})
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
            """)
            conn.commit()
            logger.info(f"Table {self.table_name} created or already exists")
        finally:
            cursor.close()
            conn.close()
    
    def _serialize_embedding(self, embedding: List[float]) -> bytes:
        """Serialize embedding to bytes."""
        return np.array(embedding, dtype=np.float32).tobytes()
    
    def _deserialize_embedding(self, blob: bytes) -> np.ndarray:
        """Deserialize bytes to numpy array."""
        return np.frombuffer(blob, dtype=np.float32)
    
    def add_texts(
        self,
        texts: Iterable[str],
        metadatas: Optional[List[dict]] = None,
        ids: Optional[List[str]] = None,
        batch_size: int = 100,
        **kwargs: Any,
    ) -> List[str]:
        """
        Add texts to the vector store.
        
        Args:
            texts: Texts to add
            metadatas: Optional metadata for each text
            ids: Optional IDs (ignored, auto-generated)
            batch_size: Batch size for embedding generation
            
        Returns:
            List of document IDs
        """
        texts_list = list(texts)
        
        if not texts_list:
            return []
        
        # Generate embeddings in batches
        all_embeddings = []
        for i in range(0, len(texts_list), batch_size):
            batch = texts_list[i:i + batch_size]
            batch_embeddings = self._embedding.embed_documents(batch)
            all_embeddings.extend(batch_embeddings)
        
        conn = self._get_connection()
        cursor = conn.cursor()
        inserted_ids = []
        
        try:
            for i, (text, embedding) in enumerate(zip(texts_list, all_embeddings)):
                metadata = metadatas[i] if metadatas else {}
                embedding_blob = self._serialize_embedding(embedding)
                
                cursor.execute(f"""
                    INSERT INTO {self.table_name} 
                    ({self.content_column}, {self.embedding_column}, {self.metadata_column})
                    VALUES (%s, %s, %s)
                    ON DUPLICATE KEY UPDATE
                        {self.embedding_column} = VALUES({self.embedding_column}),
                        {self.metadata_column} = VALUES({self.metadata_column}),
                        updated_at = NOW()
                """, (text, embedding_blob, json.dumps(metadata)))
                
                inserted_ids.append(str(cursor.lastrowid))
            
            conn.commit()
            logger.info(f"Added {len(inserted_ids)} documents to {self.table_name}")
            return inserted_ids
            
        except Exception as e:
            conn.rollback()
            logger.error(f"Error adding documents: {e}")
            raise
        finally:
            cursor.close()
            conn.close()
    
    def add_documents(
        self,
        documents: List[Document],
        **kwargs: Any,
    ) -> List[str]:
        """Add documents to the vector store."""
        texts = [doc.page_content for doc in documents]
        metadatas = [doc.metadata for doc in documents]
        return self.add_texts(texts, metadatas, **kwargs)
    
    def similarity_search(
        self,
        query: str,
        k: int = 4,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> List[Document]:
        """Search for similar documents."""
        results = self.similarity_search_with_score(query, k, filter, **kwargs)
        return [doc for doc, _ in results]
    
    def similarity_search_with_score(
        self,
        query: str,
        k: int = 4,
        filter: Optional[Dict[str, Any]] = None,
        score_threshold: Optional[float] = None,
        **kwargs: Any,
    ) -> List[Tuple[Document, float]]:
        """
        Search for similar documents with similarity scores.
        
        Args:
            query: Query text
            k: Number of results
            filter: Metadata filter (e.g., {"category": "tech"})
            score_threshold: Minimum similarity score (0-1)
            
        Returns:
            List of (Document, score) tuples
        """
        # Generate query embedding
        query_embedding = self._embedding.embed_query(query)
        query_vector = np.array(query_embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            # Build filter clause
            where_clauses = [f"{self.embedding_column} IS NOT NULL"]
            params = []
            
            if filter:
                for key, value in filter.items():
                    where_clauses.append(
                        f"JSON_EXTRACT({self.metadata_column}, '$.{key}') = %s"
                    )
                    params.append(json.dumps(value))
            
            where_clause = " AND ".join(where_clauses)
            
            cursor.execute(f"""
                SELECT id, {self.content_column}, {self.embedding_column}, {self.metadata_column}
                FROM {self.table_name}
                WHERE {where_clause}
            """, params)
            
            rows = cursor.fetchall()
            
            # Calculate similarities
            results = []
            for row in rows:
                doc_vector = self._deserialize_embedding(row[self.embedding_column])
                doc_norm = doc_vector / np.linalg.norm(doc_vector)
                similarity = float(np.dot(query_norm, doc_norm))
                
                if score_threshold is None or similarity >= score_threshold:
                    metadata = json.loads(row[self.metadata_column]) if row[self.metadata_column] else {}
                    metadata["id"] = row["id"]
                    
                    doc = Document(
                        page_content=row[self.content_column],
                        metadata=metadata
                    )
                    results.append((doc, similarity))
            
            # Sort by similarity (descending) and return top k
            results.sort(key=lambda x: x[1], reverse=True)
            return results[:k]
            
        finally:
            cursor.close()
            conn.close()
    
    def similarity_search_by_vector(
        self,
        embedding: List[float],
        k: int = 4,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> List[Document]:
        """Search by embedding vector directly."""
        query_vector = np.array(embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            where_clauses = [f"{self.embedding_column} IS NOT NULL"]
            params = []
            
            if filter:
                for key, value in filter.items():
                    where_clauses.append(
                        f"JSON_EXTRACT({self.metadata_column}, '$.{key}') = %s"
                    )
                    params.append(json.dumps(value))
            
            where_clause = " AND ".join(where_clauses)
            
            cursor.execute(f"""
                SELECT id, {self.content_column}, {self.embedding_column}, {self.metadata_column}
                FROM {self.table_name}
                WHERE {where_clause}
            """, params)
            
            rows = cursor.fetchall()
            
            results = []
            for row in rows:
                doc_vector = self._deserialize_embedding(row[self.embedding_column])
                doc_norm = doc_vector / np.linalg.norm(doc_vector)
                similarity = float(np.dot(query_norm, doc_norm))
                
                metadata = json.loads(row[self.metadata_column]) if row[self.metadata_column] else {}
                metadata["id"] = row["id"]
                metadata["score"] = similarity
                
                doc = Document(
                    page_content=row[self.content_column],
                    metadata=metadata
                )
                results.append((doc, similarity))
            
            results.sort(key=lambda x: x[1], reverse=True)
            return [doc for doc, _ in results[:k]]
            
        finally:
            cursor.close()
            conn.close()
    
    def hybrid_search(
        self,
        query: str,
        k: int = 4,
        vector_weight: float = 0.7,
        fulltext_weight: float = 0.3,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> List[Document]:
        """
        Hybrid search combining vector similarity and Full-Text search.
        
        Args:
            query: Query text
            k: Number of results
            vector_weight: Weight for vector similarity (0-1)
            fulltext_weight: Weight for Full-Text score (0-1)
            filter: Metadata filter
            
        Returns:
            List of documents ranked by hybrid score
        """
        query_embedding = self._embedding.embed_query(query)
        query_vector = np.array(query_embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            where_clauses = [f"{self.embedding_column} IS NOT NULL"]
            params = [query]  # For MATCH AGAINST
            
            if filter:
                for key, value in filter.items():
                    where_clauses.append(
                        f"JSON_EXTRACT({self.metadata_column}, '$.{key}') = %s"
                    )
                    params.append(json.dumps(value))
            
            where_clause = " AND ".join(where_clauses)
            
            cursor.execute(f"""
                SELECT 
                    id, 
                    {self.content_column}, 
                    {self.embedding_column}, 
                    {self.metadata_column},
                    MATCH({self.content_column}) AGAINST(%s IN NATURAL LANGUAGE MODE) AS ft_score
                FROM {self.table_name}
                WHERE {where_clause}
            """, params)
            
            rows = cursor.fetchall()
            
            if not rows:
                return []
            
            # Normalize Full-Text scores
            ft_scores = [row['ft_score'] for row in rows]
            max_ft = max(ft_scores) if max(ft_scores) > 0 else 1
            
            results = []
            for row in rows:
                # Vector similarity
                doc_vector = self._deserialize_embedding(row[self.embedding_column])
                doc_norm = doc_vector / np.linalg.norm(doc_vector)
                vector_score = float(np.dot(query_norm, doc_norm))
                
                # Normalized Full-Text score
                ft_score = row['ft_score'] / max_ft if max_ft > 0 else 0
                
                # Hybrid score
                hybrid_score = (vector_weight * vector_score) + (fulltext_weight * ft_score)
                
                metadata = json.loads(row[self.metadata_column]) if row[self.metadata_column] else {}
                metadata["id"] = row["id"]
                metadata["vector_score"] = vector_score
                metadata["fulltext_score"] = ft_score
                metadata["hybrid_score"] = hybrid_score
                
                doc = Document(
                    page_content=row[self.content_column],
                    metadata=metadata
                )
                results.append((doc, hybrid_score))
            
            results.sort(key=lambda x: x[1], reverse=True)
            return [doc for doc, _ in results[:k]]
            
        finally:
            cursor.close()
            conn.close()
    
    def max_marginal_relevance_search(
        self,
        query: str,
        k: int = 4,
        fetch_k: int = 20,
        lambda_mult: float = 0.5,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> List[Document]:
        """
        Maximal Marginal Relevance search for diversity.
        
        Args:
            query: Query text
            k: Number of results
            fetch_k: Number of candidates to fetch
            lambda_mult: Balance between relevance and diversity (0-1)
            filter: Metadata filter
            
        Returns:
            Diverse list of relevant documents
        """
        query_embedding = self._embedding.embed_query(query)
        
        # Get initial candidates
        candidates = self.similarity_search_with_score(
            query, k=fetch_k, filter=filter
        )
        
        if not candidates:
            return []
        
        # MMR selection
        query_vector = np.array(query_embedding, dtype=np.float32)
        
        selected = []
        selected_embeddings = []
        remaining = list(candidates)
        
        while len(selected) < k and remaining:
            best_score = -float('inf')
            best_idx = 0
            
            for i, (doc, score) in enumerate(remaining):
                # Get document embedding
                conn = self._get_connection()
                cursor = conn.cursor(dictionary=True)
                cursor.execute(f"""
                    SELECT {self.embedding_column} FROM {self.table_name}
                    WHERE id = %s
                """, (doc.metadata.get("id"),))
                row = cursor.fetchone()
                cursor.close()
                conn.close()
                
                if not row:
                    continue
                
                doc_embedding = self._deserialize_embedding(row[self.embedding_column])
                
                # Calculate MMR score
                relevance = score
                
                diversity = 0
                if selected_embeddings:
                    similarities = [
                        np.dot(doc_embedding, sel_emb) / 
                        (np.linalg.norm(doc_embedding) * np.linalg.norm(sel_emb))
                        for sel_emb in selected_embeddings
                    ]
                    diversity = max(similarities)
                
                mmr_score = lambda_mult * relevance - (1 - lambda_mult) * diversity
                
                if mmr_score > best_score:
                    best_score = mmr_score
                    best_idx = i
            
            # Add best candidate to selected
            best_doc, _ = remaining.pop(best_idx)
            selected.append(best_doc)
            
            # Store embedding for diversity calculation
            conn = self._get_connection()
            cursor = conn.cursor(dictionary=True)
            cursor.execute(f"""
                SELECT {self.embedding_column} FROM {self.table_name}
                WHERE id = %s
            """, (best_doc.metadata.get("id"),))
            row = cursor.fetchone()
            cursor.close()
            conn.close()
            
            if row:
                selected_embeddings.append(
                    self._deserialize_embedding(row[self.embedding_column])
                )
        
        return selected
    
    def delete(
        self,
        ids: Optional[List[str]] = None,
        filter: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> Optional[bool]:
        """Delete documents by ID or filter."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        try:
            if ids:
                placeholders = ','.join(['%s'] * len(ids))
                cursor.execute(f"""
                    DELETE FROM {self.table_name} WHERE id IN ({placeholders})
                """, ids)
            elif filter:
                where_clauses = []
                params = []
                for key, value in filter.items():
                    where_clauses.append(
                        f"JSON_EXTRACT({self.metadata_column}, '$.{key}') = %s"
                    )
                    params.append(json.dumps(value))
                
                cursor.execute(f"""
                    DELETE FROM {self.table_name} WHERE {' AND '.join(where_clauses)}
                """, params)
            else:
                return False
            
            conn.commit()
            logger.info(f"Deleted {cursor.rowcount} documents")
            return True
            
        finally:
            cursor.close()
            conn.close()
    
    @classmethod
    def from_texts(
        cls,
        texts: List[str],
        embedding: Embeddings,
        metadatas: Optional[List[dict]] = None,
        connection_config: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> "MariaDBVectorStore":
        """Create a vector store from texts."""
        if connection_config is None:
            raise ValueError("connection_config is required")
        
        store = cls(
            connection_config=connection_config,
            embedding=embedding,
            **kwargs
        )
        store.add_texts(texts, metadatas)
        return store
    
    @classmethod
    def from_documents(
        cls,
        documents: List[Document],
        embedding: Embeddings,
        connection_config: Optional[Dict[str, Any]] = None,
        **kwargs: Any,
    ) -> "MariaDBVectorStore":
        """Create a vector store from documents."""
        texts = [doc.page_content for doc in documents]
        metadatas = [doc.metadata for doc in documents]
        return cls.from_texts(
            texts, embedding, metadatas, connection_config, **kwargs
        )
    
    def as_retriever(self, **kwargs: Any):
        """Return a retriever for this vector store."""
        from langchain_core.vectorstores import VectorStoreRetriever
        
        search_type = kwargs.pop("search_type", "similarity")
        search_kwargs = kwargs.pop("search_kwargs", {})
        
        if search_type == "hybrid":
            # Custom retriever for hybrid search
            return HybridRetriever(vectorstore=self, **search_kwargs)
        
        return VectorStoreRetriever(
            vectorstore=self,
            search_type=search_type,
            search_kwargs=search_kwargs
        )


class HybridRetriever:
    """Custom retriever for hybrid search."""
    
    def __init__(
        self, 
        vectorstore: MariaDBVectorStore,
        k: int = 4,
        vector_weight: float = 0.7,
        fulltext_weight: float = 0.3,
        filter: Optional[Dict] = None
    ):
        self.vectorstore = vectorstore
        self.k = k
        self.vector_weight = vector_weight
        self.fulltext_weight = fulltext_weight
        self.filter = filter
    
    def get_relevant_documents(self, query: str) -> List[Document]:
        return self.vectorstore.hybrid_search(
            query=query,
            k=self.k,
            vector_weight=self.vector_weight,
            fulltext_weight=self.fulltext_weight,
            filter=self.filter
        )
    
    async def aget_relevant_documents(self, query: str) -> List[Document]:
        return self.get_relevant_documents(query)
```

### SQL Agent LangChain

```python
#!/usr/bin/env python3
"""
langchain_sql_agent.py
Agent SQL intelligent avec LangChain et MariaDB
"""

from langchain_community.utilities import SQLDatabase
from langchain_community.agent_toolkits import SQLDatabaseToolkit
from langchain_openai import ChatOpenAI
from langchain.agents import create_sql_agent, AgentType
from langchain_core.prompts import PromptTemplate
from langchain.agents.agent_toolkits import create_retriever_tool
from langchain.tools import Tool
from typing import Optional, List, Dict, Any
import logging

logger = logging.getLogger(__name__)


class MariaDBSQLAgent:
    """
    Agent SQL intelligent capable de :
    - Comprendre les questions en langage naturel
    - Explorer le schéma de la base
    - Générer et exécuter des requêtes SQL
    - Interpréter les résultats
    """
    
    def __init__(
        self,
        connection_string: str,
        model_name: str = "gpt-4",
        temperature: float = 0,
        include_tables: Optional[List[str]] = None,
        sample_rows_in_table_info: int = 3,
        verbose: bool = False
    ):
        """
        Initialize the SQL Agent.
        
        Args:
            connection_string: MariaDB connection string
                Format: "mariadb+mariadbconnector://user:pass@host:port/db"
            model_name: LLM model to use
            temperature: Model temperature
            include_tables: List of tables to include (None = all)
            sample_rows_in_table_info: Number of sample rows to show
            verbose: Enable verbose logging
        """
        # Initialize database connection
        self.db = SQLDatabase.from_uri(
            connection_string,
            include_tables=include_tables,
            sample_rows_in_table_info=sample_rows_in_table_info
        )
        
        # Initialize LLM
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=temperature
        )
        
        # Create toolkit
        self.toolkit = SQLDatabaseToolkit(
            db=self.db,
            llm=self.llm
        )
        
        # Custom prompt for better SQL generation
        self.custom_prompt = """You are an expert SQL analyst working with a MariaDB database.

Given an input question, create a syntactically correct MariaDB query to run, then look at the results and return the answer.

Unless the user specifies a specific number of examples they wish to obtain, always limit your query to at most 10 results.
You can order the results by a relevant column to return the most interesting examples in the database.

Never query for all the columns from a specific table, only ask for the relevant columns given the question.

Pay attention to use only the column names that you can see in the schema description. 
Be careful to not query for columns that do not exist. Also, pay attention to which column is in which table.

Use the following format:

Question: Question here
SQLQuery: SQL Query to run
SQLResult: Result of the SQLQuery
Answer: Final answer here

Only use the following tables:
{table_info}

Question: {input}"""
        
        # Create agent
        self.agent = create_sql_agent(
            llm=self.llm,
            toolkit=self.toolkit,
            agent_type=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
            verbose=verbose,
            handle_parsing_errors=True,
            max_iterations=10
        )
    
    def query(self, question: str) -> str:
        """
        Ask a question in natural language.
        
        Args:
            question: The question to answer
            
        Returns:
            Natural language answer
        """
        try:
            result = self.agent.invoke({"input": question})
            return result.get("output", "No answer generated")
        except Exception as e:
            logger.error(f"Error executing query: {e}")
            return f"Error: {str(e)}"
    
    def get_schema(self) -> str:
        """Get the database schema."""
        return self.db.get_table_info()
    
    def run_sql(self, sql: str) -> str:
        """Execute a raw SQL query."""
        return self.db.run(sql)
    
    def get_sample_queries(self, table_name: str) -> List[str]:
        """Generate sample queries for a table."""
        prompt = f"""Given the table '{table_name}' with schema:
{self.db.get_table_info([table_name])}

Generate 5 useful example SQL queries that a business user might want to run.
Return only the SQL queries, one per line."""
        
        response = self.llm.invoke(prompt)
        queries = response.content.strip().split('\n')
        return [q.strip() for q in queries if q.strip()]


class ConversationalSQLAgent:
    """
    Agent SQL conversationnel avec mémoire.
    """
    
    def __init__(
        self,
        connection_string: str,
        model_name: str = "gpt-4",
        memory_connection_config: Optional[Dict] = None
    ):
        from langchain.memory import ConversationBufferMemory
        from langchain.chains import ConversationChain
        
        self.db = SQLDatabase.from_uri(connection_string)
        self.llm = ChatOpenAI(model=model_name, temperature=0)
        
        # Memory (optionally backed by MariaDB)
        if memory_connection_config:
            self.memory = MariaDBChatMemory(
                connection_config=memory_connection_config,
                session_id="sql_agent_session"
            )
        else:
            self.memory = ConversationBufferMemory(
                memory_key="chat_history",
                return_messages=True
            )
        
        # Tools
        self.tools = [
            Tool(
                name="query_database",
                description="Execute a SQL query on the database. Input should be a valid SQL query.",
                func=self.db.run
            ),
            Tool(
                name="get_schema",
                description="Get the schema of the database including table names and columns.",
                func=lambda _: self.db.get_table_info()
            ),
            Tool(
                name="get_table_info",
                description="Get detailed info about a specific table. Input should be the table name.",
                func=lambda table: self.db.get_table_info([table])
            )
        ]
        
        # Agent with memory
        from langchain.agents import initialize_agent, AgentType
        
        self.agent = initialize_agent(
            tools=self.tools,
            llm=self.llm,
            agent=AgentType.CONVERSATIONAL_REACT_DESCRIPTION,
            memory=self.memory,
            verbose=True,
            handle_parsing_errors=True
        )
    
    def chat(self, message: str) -> str:
        """Send a message and get a response."""
        try:
            result = self.agent.invoke({"input": message})
            return result.get("output", "No response")
        except Exception as e:
            logger.error(f"Error in chat: {e}")
            return f"Error: {str(e)}"
    
    def clear_memory(self):
        """Clear the conversation memory."""
        self.memory.clear()


# ═══════════════════════════════════════════════════════════════════════════
# MEMORY BACKED BY MARIADB
# ═══════════════════════════════════════════════════════════════════════════

class MariaDBChatMemory:
    """
    Chat memory stored in MariaDB.
    Persists conversation history across sessions.
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        session_id: str,
        table_name: str = "chat_history",
        max_messages: int = 100
    ):
        self.connection_config = connection_config
        self.session_id = session_id
        self.table_name = table_name
        self.max_messages = max_messages
        self._ensure_table()
    
    def _get_connection(self):
        return mariadb.connect(**self.connection_config)
    
    def _ensure_table(self):
        conn = self._get_connection()
        cursor = conn.cursor()
        cursor.execute(f"""
            CREATE TABLE IF NOT EXISTS {self.table_name} (
                id BIGINT PRIMARY KEY AUTO_INCREMENT,
                session_id VARCHAR(255) NOT NULL,
                role ENUM('human', 'ai', 'system') NOT NULL,
                content TEXT NOT NULL,
                metadata JSON,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                
                INDEX idx_session (session_id, created_at)
            ) ENGINE=InnoDB
        """)
        conn.commit()
        cursor.close()
        conn.close()
    
    @property
    def memory_variables(self) -> List[str]:
        return ["chat_history"]
    
    def load_memory_variables(self, inputs: Dict[str, Any]) -> Dict[str, Any]:
        """Load conversation history."""
        from langchain_core.messages import HumanMessage, AIMessage
        
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute(f"""
            SELECT role, content FROM {self.table_name}
            WHERE session_id = %s
            ORDER BY created_at DESC
            LIMIT %s
        """, (self.session_id, self.max_messages))
        
        rows = cursor.fetchall()
        cursor.close()
        conn.close()
        
        messages = []
        for row in reversed(rows):
            if row['role'] == 'human':
                messages.append(HumanMessage(content=row['content']))
            elif row['role'] == 'ai':
                messages.append(AIMessage(content=row['content']))
        
        return {"chat_history": messages}
    
    def save_context(self, inputs: Dict[str, Any], outputs: Dict[str, str]):
        """Save a turn of conversation."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        # Save human message
        human_input = inputs.get("input", inputs.get("question", ""))
        cursor.execute(f"""
            INSERT INTO {self.table_name} (session_id, role, content)
            VALUES (%s, 'human', %s)
        """, (self.session_id, human_input))
        
        # Save AI message
        ai_output = outputs.get("output", outputs.get("response", ""))
        cursor.execute(f"""
            INSERT INTO {self.table_name} (session_id, role, content)
            VALUES (%s, 'ai', %s)
        """, (self.session_id, ai_output))
        
        conn.commit()
        cursor.close()
        conn.close()
    
    def clear(self):
        """Clear the session history."""
        conn = self._get_connection()
        cursor = conn.cursor()
        cursor.execute(f"""
            DELETE FROM {self.table_name} WHERE session_id = %s
        """, (self.session_id,))
        conn.commit()
        cursor.close()
        conn.close()


# ═══════════════════════════════════════════════════════════════════════════
# EXEMPLE D'UTILISATION
# ═══════════════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    # Connection string for MariaDB
    conn_string = "mariadb+mariadbconnector://user:password@localhost:3306/ecommerce"
    
    # Create agent
    agent = MariaDBSQLAgent(
        connection_string=conn_string,
        model_name="gpt-4",
        include_tables=["customers", "orders", "products"],
        verbose=True
    )
    
    # Ask questions
    questions = [
        "Combien de clients avons-nous ?",
        "Quel est le chiffre d'affaires total du mois dernier ?",
        "Quels sont les 5 produits les plus vendus ?",
        "Quel est le panier moyen par client ?"
    ]
    
    for q in questions:
        print(f"\nQuestion: {q}")
        answer = agent.query(q)
        print(f"Answer: {answer}")
```

### RAG Chain complète

```python
#!/usr/bin/env python3
"""
langchain_rag_chain.py
Pipeline RAG complet avec LangChain et MariaDB
"""

from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from langchain_core.messages import HumanMessage, AIMessage
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import (
    PyPDFLoader,
    TextLoader,
    DirectoryLoader,
    UnstructuredHTMLLoader
)
from typing import List, Dict, Any, Optional
import logging

# Import our MariaDB Vector Store
from langchain_mariadb_vectorstore import MariaDBVectorStore, MariaDBChatMemory

logger = logging.getLogger(__name__)


class MariaDBRAGPipeline:
    """
    Pipeline RAG complet avec :
    - Ingestion de documents (PDF, TXT, HTML)
    - Chunking intelligent
    - Stockage dans MariaDB
    - Recherche hybride (vector + full-text)
    - Génération avec mémoire conversationnelle
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        model_name: str = "gpt-4",
        embedding_model: str = "text-embedding-3-small",
        chunk_size: int = 1000,
        chunk_overlap: int = 200,
        table_name: str = "rag_documents"
    ):
        self.connection_config = connection_config
        
        # Initialize embeddings
        self.embeddings = OpenAIEmbeddings(model=embedding_model)
        
        # Initialize LLM
        self.llm = ChatOpenAI(model=model_name, temperature=0)
        
        # Initialize vector store
        self.vectorstore = MariaDBVectorStore(
            connection_config=connection_config,
            embedding=self.embeddings,
            table_name=table_name
        )
        
        # Text splitter
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap,
            separators=["\n\n", "\n", ". ", " ", ""]
        )
        
        # RAG prompt
        self.rag_prompt = ChatPromptTemplate.from_messages([
            ("system", """Tu es un assistant expert qui répond aux questions en utilisant 
le contexte fourni. Si tu ne trouves pas l'information dans le contexte, dis-le clairement.

Règles:
- Réponds de manière précise et concise
- Cite les sources quand c'est pertinent
- Si tu n'es pas sûr, exprime ton incertitude
- Utilise le français pour répondre

Contexte:
{context}"""),
            MessagesPlaceholder(variable_name="chat_history"),
            ("human", "{question}")
        ])
        
        # Build the chain
        self._build_chain()
    
    def _build_chain(self):
        """Build the RAG chain."""
        
        def format_docs(docs):
            return "\n\n".join([
                f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
                for doc in docs
            ])
        
        # Retriever
        retriever = self.vectorstore.as_retriever(
            search_type="hybrid",
            search_kwargs={
                "k": 5,
                "vector_weight": 0.7,
                "fulltext_weight": 0.3
            }
        )
        
        # Chain
        self.chain = (
            {
                "context": retriever | format_docs,
                "question": RunnablePassthrough(),
                "chat_history": RunnableLambda(lambda x: x.get("chat_history", []))
            }
            | self.rag_prompt
            | self.llm
            | StrOutputParser()
        )
    
    def ingest_documents(
        self,
        source: str,
        source_type: str = "auto",
        metadata: Optional[Dict] = None
    ) -> int:
        """
        Ingest documents into the vector store.
        
        Args:
            source: File path, directory path, or URL
            source_type: "pdf", "txt", "html", "directory", or "auto"
            metadata: Additional metadata to attach
            
        Returns:
            Number of chunks ingested
        """
        # Load documents based on type
        if source_type == "auto":
            if source.endswith(".pdf"):
                source_type = "pdf"
            elif source.endswith(".txt") or source.endswith(".md"):
                source_type = "txt"
            elif source.endswith(".html"):
                source_type = "html"
            elif "." not in source.split("/")[-1]:
                source_type = "directory"
        
        if source_type == "pdf":
            loader = PyPDFLoader(source)
        elif source_type == "txt":
            loader = TextLoader(source)
        elif source_type == "html":
            loader = UnstructuredHTMLLoader(source)
        elif source_type == "directory":
            loader = DirectoryLoader(
                source,
                glob="**/*.{txt,pdf,md}",
                show_progress=True
            )
        else:
            raise ValueError(f"Unknown source type: {source_type}")
        
        documents = loader.load()
        logger.info(f"Loaded {len(documents)} documents from {source}")
        
        # Split into chunks
        chunks = self.text_splitter.split_documents(documents)
        logger.info(f"Split into {len(chunks)} chunks")
        
        # Add metadata
        if metadata:
            for chunk in chunks:
                chunk.metadata.update(metadata)
        
        # Add to vector store
        self.vectorstore.add_documents(chunks)
        
        return len(chunks)
    
    def query(
        self,
        question: str,
        chat_history: Optional[List] = None,
        filter: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """
        Query the RAG system.
        
        Args:
            question: The question to answer
            chat_history: Previous conversation messages
            filter: Metadata filter for retrieval
            
        Returns:
            Dict with answer and sources
        """
        chat_history = chat_history or []
        
        # Get relevant documents
        if filter:
            docs = self.vectorstore.hybrid_search(
                question, k=5, filter=filter
            )
        else:
            docs = self.vectorstore.hybrid_search(question, k=5)
        
        # Generate answer
        answer = self.chain.invoke({
            "question": question,
            "chat_history": chat_history
        })
        
        # Extract sources
        sources = [
            {
                "content": doc.page_content[:200] + "...",
                "source": doc.metadata.get("source", "unknown"),
                "score": doc.metadata.get("hybrid_score", 0)
            }
            for doc in docs
        ]
        
        return {
            "answer": answer,
            "sources": sources,
            "num_sources": len(docs)
        }
    
    def chat(
        self,
        message: str,
        session_id: str = "default"
    ) -> str:
        """
        Chat with memory.
        
        Args:
            message: User message
            session_id: Session identifier for memory
            
        Returns:
            Assistant response
        """
        # Load chat history from MariaDB
        memory = MariaDBChatMemory(
            connection_config=self.connection_config,
            session_id=session_id
        )
        
        history_vars = memory.load_memory_variables({})
        chat_history = history_vars.get("chat_history", [])
        
        # Query RAG
        result = self.query(message, chat_history=chat_history)
        answer = result["answer"]
        
        # Save to memory
        memory.save_context(
            {"input": message},
            {"output": answer}
        )
        
        return answer


class MultiSourceRAG:
    """
    RAG avec multiple sources de données :
    - Documents (Vector Store)
    - Base de données SQL
    - APIs externes
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        sql_connection_string: str,
        model_name: str = "gpt-4"
    ):
        self.llm = ChatOpenAI(model=model_name, temperature=0)
        
        # Document RAG
        self.doc_rag = MariaDBRAGPipeline(
            connection_config=connection_config,
            model_name=model_name
        )
        
        # SQL Agent
        self.sql_agent = MariaDBSQLAgent(
            connection_string=sql_connection_string,
            model_name=model_name
        )
        
        # Router prompt
        self.router_prompt = ChatPromptTemplate.from_template("""
Analyze the following question and determine the best source to answer it:

Question: {question}

Available sources:
1. DOCUMENTS - Company policies, procedures, manuals, reports
2. DATABASE - Transactional data, customers, orders, products, sales figures
3. BOTH - Questions requiring information from both sources

Respond with only one word: DOCUMENTS, DATABASE, or BOTH
""")
    
    def _route_question(self, question: str) -> str:
        """Determine which source to use."""
        chain = self.router_prompt | self.llm | StrOutputParser()
        route = chain.invoke({"question": question}).strip().upper()
        
        if route not in ["DOCUMENTS", "DATABASE", "BOTH"]:
            route = "BOTH"
        
        return route
    
    def query(self, question: str) -> Dict[str, Any]:
        """
        Route and answer a question using the appropriate source.
        """
        route = self._route_question(question)
        logger.info(f"Routing question to: {route}")
        
        if route == "DOCUMENTS":
            result = self.doc_rag.query(question)
            return {
                "answer": result["answer"],
                "source": "documents",
                "details": result["sources"]
            }
        
        elif route == "DATABASE":
            answer = self.sql_agent.query(question)
            return {
                "answer": answer,
                "source": "database",
                "details": None
            }
        
        else:  # BOTH
            # Query both sources
            doc_result = self.doc_rag.query(question)
            sql_result = self.sql_agent.query(question)
            
            # Combine answers
            combine_prompt = ChatPromptTemplate.from_template("""
Combine the following information to provide a comprehensive answer:

Question: {question}

Information from documents:
{doc_answer}

Information from database:
{sql_answer}

Provide a unified, coherent answer that synthesizes both sources.
""")
            
            chain = combine_prompt | self.llm | StrOutputParser()
            combined = chain.invoke({
                "question": question,
                "doc_answer": doc_result["answer"],
                "sql_answer": sql_result
            })
            
            return {
                "answer": combined,
                "source": "both",
                "details": {
                    "documents": doc_result["sources"],
                    "database": sql_result
                }
            }


# ═══════════════════════════════════════════════════════════════════════════
# EXEMPLE D'UTILISATION
# ═══════════════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    config = {
        "host": "localhost",
        "port": 3306,
        "user": "rag_user",
        "password": "password",
        "database": "rag_db"
    }
    
    # Create RAG pipeline
    rag = MariaDBRAGPipeline(connection_config=config)
    
    # Ingest documents
    rag.ingest_documents(
        "./documents/policies/",
        source_type="directory",
        metadata={"category": "policies"}
    )
    
    # Query
    result = rag.query("Quelle est la politique de remboursement ?")
    print(f"Answer: {result['answer']}")
    print(f"Sources: {result['sources']}")
    
    # Chat with memory
    response = rag.chat("Bonjour, j'ai une question sur les congés", session_id="user123")
    print(response)
    
    response = rag.chat("Combien de jours ai-je droit ?", session_id="user123")
    print(response)
```

---

## LlamaIndex : Intégration avancée

### Vector Store LlamaIndex

```python
#!/usr/bin/env python3
"""
llamaindex_mariadb.py
Intégration complète de LlamaIndex avec MariaDB
"""

from llama_index.core import (
    VectorStoreIndex,
    SimpleDirectoryReader,
    StorageContext,
    Settings,
    Document as LlamaDocument,
    QueryBundle
)
from llama_index.core.vector_stores.types import (
    VectorStore,
    VectorStoreQuery,
    VectorStoreQueryResult,
    VectorStoreQueryMode
)
from llama_index.core.schema import TextNode, NodeWithScore, BaseNode
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
from typing import Any, Dict, List, Optional
import numpy as np
import mariadb
import json
import logging

logger = logging.getLogger(__name__)


class MariaDBVectorStore(VectorStore):
    """
    LlamaIndex Vector Store implementation for MariaDB.
    
    Features:
    - Full vector store functionality
    - Metadata filtering
    - Hybrid search support
    - Batch operations
    """
    
    stores_text: bool = True
    flat_metadata: bool = False
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        table_name: str = "llama_vectors",
        embed_dim: int = 1536,
        hybrid_search: bool = False
    ):
        """
        Initialize MariaDB Vector Store.
        
        Args:
            connection_config: MariaDB connection parameters
            table_name: Table name for storing vectors
            embed_dim: Embedding dimension
            hybrid_search: Enable hybrid search
        """
        self.connection_config = connection_config
        self.table_name = table_name
        self.embed_dim = embed_dim
        self.hybrid_search = hybrid_search
        self._pool = None
        self._ensure_table()
    
    def _get_pool(self):
        if self._pool is None:
            self._pool = mariadb.ConnectionPool(
                pool_name="llama_pool",
                pool_size=5,
                **self.connection_config
            )
        return self._pool
    
    def _get_connection(self):
        return self._get_pool().get_connection()
    
    def _ensure_table(self):
        """Create the table if it doesn't exist."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        cursor.execute(f"""
            CREATE TABLE IF NOT EXISTS {self.table_name} (
                id VARCHAR(255) PRIMARY KEY,
                text TEXT NOT NULL,
                embedding BLOB,
                metadata JSON,
                node_info JSON,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                
                FULLTEXT INDEX ft_text (text)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        """)
        
        conn.commit()
        cursor.close()
        conn.close()
    
    @property
    def client(self):
        return self._get_connection()
    
    def add(
        self,
        nodes: List[BaseNode],
        **kwargs: Any
    ) -> List[str]:
        """Add nodes to the vector store."""
        conn = self._get_connection()
        cursor = conn.cursor()
        ids = []
        
        try:
            for node in nodes:
                node_id = node.node_id
                text = node.get_content()
                embedding = node.embedding
                
                embedding_blob = None
                if embedding is not None:
                    embedding_blob = np.array(embedding, dtype=np.float32).tobytes()
                
                # Node info for reconstruction
                node_info = {
                    "class_name": node.class_name(),
                    "relationships": {
                        k.name: v.node_id for k, v in node.relationships.items()
                    } if hasattr(node, 'relationships') else {}
                }
                
                cursor.execute(f"""
                    INSERT INTO {self.table_name} (id, text, embedding, metadata, node_info)
                    VALUES (%s, %s, %s, %s, %s)
                    ON DUPLICATE KEY UPDATE
                        text = VALUES(text),
                        embedding = VALUES(embedding),
                        metadata = VALUES(metadata),
                        node_info = VALUES(node_info)
                """, (
                    node_id,
                    text,
                    embedding_blob,
                    json.dumps(node.metadata) if node.metadata else None,
                    json.dumps(node_info)
                ))
                
                ids.append(node_id)
            
            conn.commit()
            logger.info(f"Added {len(ids)} nodes to {self.table_name}")
            return ids
            
        finally:
            cursor.close()
            conn.close()
    
    def delete(self, ref_doc_id: str, **kwargs: Any) -> None:
        """Delete nodes by reference document ID."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        cursor.execute(f"""
            DELETE FROM {self.table_name}
            WHERE JSON_EXTRACT(metadata, '$.ref_doc_id') = %s
        """, (json.dumps(ref_doc_id),))
        
        conn.commit()
        cursor.close()
        conn.close()
    
    def query(
        self,
        query: VectorStoreQuery,
        **kwargs: Any
    ) -> VectorStoreQueryResult:
        """Query the vector store."""
        
        if query.query_embedding is None:
            return VectorStoreQueryResult(nodes=[], similarities=[], ids=[])
        
        query_vector = np.array(query.query_embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            # Build query with filters
            where_clauses = ["embedding IS NOT NULL"]
            params = []
            
            if query.filters is not None:
                for f in query.filters.legacy_filters():
                    key = f.key
                    value = f.value
                    where_clauses.append(
                        f"JSON_EXTRACT(metadata, '$.{key}') = %s"
                    )
                    params.append(json.dumps(value))
            
            where_clause = " AND ".join(where_clauses)
            
            # Hybrid search
            if self.hybrid_search and query.query_str:
                cursor.execute(f"""
                    SELECT id, text, embedding, metadata, node_info,
                           MATCH(text) AGAINST(%s IN NATURAL LANGUAGE MODE) AS ft_score
                    FROM {self.table_name}
                    WHERE {where_clause}
                """, [query.query_str] + params)
            else:
                cursor.execute(f"""
                    SELECT id, text, embedding, metadata, node_info
                    FROM {self.table_name}
                    WHERE {where_clause}
                """, params)
            
            rows = cursor.fetchall()
            
            if not rows:
                return VectorStoreQueryResult(nodes=[], similarities=[], ids=[])
            
            # Calculate similarities
            results = []
            max_ft_score = 1
            if self.hybrid_search and rows:
                ft_scores = [r.get('ft_score', 0) for r in rows]
                max_ft_score = max(ft_scores) if max(ft_scores) > 0 else 1
            
            for row in rows:
                doc_vector = np.frombuffer(row['embedding'], dtype=np.float32)
                doc_norm = doc_vector / np.linalg.norm(doc_vector)
                vector_sim = float(np.dot(query_norm, doc_norm))
                
                # Hybrid score
                if self.hybrid_search:
                    ft_score = row.get('ft_score', 0) / max_ft_score
                    similarity = 0.7 * vector_sim + 0.3 * ft_score
                else:
                    similarity = vector_sim
                
                # Reconstruct node
                metadata = json.loads(row['metadata']) if row['metadata'] else {}
                
                node = TextNode(
                    id_=row['id'],
                    text=row['text'],
                    metadata=metadata,
                    embedding=doc_vector.tolist()
                )
                
                results.append((node, similarity, row['id']))
            
            # Sort and limit
            results.sort(key=lambda x: x[1], reverse=True)
            k = query.similarity_top_k or 10
            results = results[:k]
            
            nodes_with_scores = [
                NodeWithScore(node=r[0], score=r[1])
                for r in results
            ]
            
            return VectorStoreQueryResult(
                nodes=nodes_with_scores,
                similarities=[r[1] for r in results],
                ids=[r[2] for r in results]
            )
            
        finally:
            cursor.close()
            conn.close()


class MariaDBIndexer:
    """
    High-level indexer for documents using LlamaIndex and MariaDB.
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        table_name: str = "llama_vectors",
        model_name: str = "gpt-4",
        embedding_model: str = "text-embedding-3-small",
        chunk_size: int = 512,
        chunk_overlap: int = 50
    ):
        self.connection_config = connection_config
        
        # Configure LlamaIndex settings
        Settings.llm = OpenAI(model=model_name, temperature=0)
        Settings.embed_model = OpenAIEmbedding(model=embedding_model)
        Settings.node_parser = SentenceSplitter(
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap
        )
        
        # Create vector store
        self.vector_store = MariaDBVectorStore(
            connection_config=connection_config,
            table_name=table_name,
            hybrid_search=True
        )
        
        # Storage context
        self.storage_context = StorageContext.from_defaults(
            vector_store=self.vector_store
        )
        
        self._index = None
    
    def index_documents(self, path: str) -> VectorStoreIndex:
        """
        Index documents from a path.
        
        Args:
            path: File or directory path
            
        Returns:
            VectorStoreIndex
        """
        # Load documents
        documents = SimpleDirectoryReader(path).load_data()
        logger.info(f"Loaded {len(documents)} documents")
        
        # Create index
        self._index = VectorStoreIndex.from_documents(
            documents,
            storage_context=self.storage_context,
            show_progress=True
        )
        
        return self._index
    
    def index_texts(
        self,
        texts: List[str],
        metadatas: Optional[List[Dict]] = None
    ) -> VectorStoreIndex:
        """
        Index raw texts.
        
        Args:
            texts: List of text strings
            metadatas: Optional metadata for each text
            
        Returns:
            VectorStoreIndex
        """
        documents = []
        for i, text in enumerate(texts):
            metadata = metadatas[i] if metadatas else {}
            doc = LlamaDocument(text=text, metadata=metadata)
            documents.append(doc)
        
        self._index = VectorStoreIndex.from_documents(
            documents,
            storage_context=self.storage_context,
            show_progress=True
        )
        
        return self._index
    
    def get_index(self) -> VectorStoreIndex:
        """Get or create index from existing data."""
        if self._index is None:
            self._index = VectorStoreIndex.from_vector_store(
                self.vector_store,
                storage_context=self.storage_context
            )
        return self._index
    
    def query(
        self,
        question: str,
        similarity_top_k: int = 5,
        response_mode: str = "compact"
    ) -> Dict[str, Any]:
        """
        Query the index.
        
        Args:
            question: Query string
            similarity_top_k: Number of similar documents to retrieve
            response_mode: Response synthesis mode
            
        Returns:
            Dict with response and source nodes
        """
        index = self.get_index()
        
        query_engine = index.as_query_engine(
            similarity_top_k=similarity_top_k,
            response_mode=response_mode
        )
        
        response = query_engine.query(question)
        
        return {
            "response": str(response),
            "source_nodes": [
                {
                    "text": node.node.text[:200] + "...",
                    "score": node.score,
                    "metadata": node.node.metadata
                }
                for node in response.source_nodes
            ]
        }
    
    def chat(
        self,
        message: str,
        chat_history: Optional[List] = None
    ) -> str:
        """
        Chat with the index (with memory).
        """
        index = self.get_index()
        
        chat_engine = index.as_chat_engine(
            chat_mode="context",
            verbose=True
        )
        
        if chat_history:
            for msg in chat_history:
                if msg["role"] == "user":
                    chat_engine.chat_history.append(
                        {"role": "user", "content": msg["content"]}
                    )
                else:
                    chat_engine.chat_history.append(
                        {"role": "assistant", "content": msg["content"]}
                    )
        
        response = chat_engine.chat(message)
        return str(response)


# ═══════════════════════════════════════════════════════════════════════════
# SQL QUERY ENGINE
# ═══════════════════════════════════════════════════════════════════════════

from llama_index.core.query_engine import NLSQLTableQueryEngine
from llama_index.core import SQLDatabase
from sqlalchemy import create_engine, text


class MariaDBSQLQueryEngine:
    """
    Natural language to SQL query engine using LlamaIndex.
    """
    
    def __init__(
        self,
        connection_string: str,
        tables: Optional[List[str]] = None,
        model_name: str = "gpt-4"
    ):
        """
        Initialize SQL Query Engine.
        
        Args:
            connection_string: SQLAlchemy connection string
            tables: List of tables to include
            model_name: LLM model name
        """
        # Create SQLAlchemy engine
        engine = create_engine(connection_string)
        
        # Create SQL Database
        self.sql_database = SQLDatabase(
            engine,
            include_tables=tables
        )
        
        # Configure LLM
        Settings.llm = OpenAI(model=model_name, temperature=0)
        
        # Create query engine
        self.query_engine = NLSQLTableQueryEngine(
            sql_database=self.sql_database,
            tables=tables
        )
    
    def query(self, question: str) -> Dict[str, Any]:
        """
        Query using natural language.
        
        Args:
            question: Natural language question
            
        Returns:
            Dict with response and SQL query
        """
        response = self.query_engine.query(question)
        
        return {
            "response": str(response),
            "sql_query": response.metadata.get("sql_query", ""),
            "result": response.metadata.get("result", "")
        }
    
    def get_schema(self) -> str:
        """Get database schema."""
        return self.sql_database.get_table_info()


# ═══════════════════════════════════════════════════════════════════════════
# EXEMPLE D'UTILISATION
# ═══════════════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    config = {
        "host": "localhost",
        "port": 3306,
        "user": "llama_user",
        "password": "password",
        "database": "llama_db"
    }
    
    # Create indexer
    indexer = MariaDBIndexer(connection_config=config)
    
    # Index documents
    indexer.index_documents("./documents/")
    
    # Query
    result = indexer.query("What is the return policy?")
    print(f"Response: {result['response']}")
    
    # Chat
    response = indexer.chat("Hello, I have a question about shipping")
    print(response)
    
    # SQL queries
    sql_engine = MariaDBSQLQueryEngine(
        connection_string="mariadb+mariadbconnector://user:pass@localhost/db",
        tables=["orders", "customers", "products"]
    )
    
    result = sql_engine.query("How many orders were placed last month?")
    print(f"Response: {result['response']}")
    print(f"SQL: {result['sql_query']}")
```

---

## Autres frameworks

### Haystack

```python
#!/usr/bin/env python3
"""
haystack_mariadb.py
Intégration de Haystack avec MariaDB
"""

from haystack import Pipeline, Document
from haystack.components.retrievers import InMemoryBM25Retriever
from haystack.components.generators import OpenAIGenerator
from haystack.components.builders import PromptBuilder
from haystack.document_stores.types import DocumentStore, DuplicatePolicy
from typing import Any, Dict, List, Optional
import numpy as np
import mariadb
import json


class MariaDBDocumentStore(DocumentStore):
    """
    Haystack DocumentStore implementation for MariaDB.
    """
    
    def __init__(
        self,
        connection_config: Dict[str, Any],
        table_name: str = "haystack_documents",
        embedding_dim: int = 1536
    ):
        self.connection_config = connection_config
        self.table_name = table_name
        self.embedding_dim = embedding_dim
        self._ensure_table()
    
    def _get_connection(self):
        return mariadb.connect(**self.connection_config)
    
    def _ensure_table(self):
        conn = self._get_connection()
        cursor = conn.cursor()
        cursor.execute(f"""
            CREATE TABLE IF NOT EXISTS {self.table_name} (
                id VARCHAR(255) PRIMARY KEY,
                content TEXT NOT NULL,
                embedding BLOB,
                meta JSON,
                score FLOAT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FULLTEXT INDEX ft_content (content)
            ) ENGINE=InnoDB
        """)
        conn.commit()
        cursor.close()
        conn.close()
    
    def count_documents(self) -> int:
        conn = self._get_connection()
        cursor = conn.cursor()
        cursor.execute(f"SELECT COUNT(*) FROM {self.table_name}")
        count = cursor.fetchone()[0]
        cursor.close()
        conn.close()
        return count
    
    def filter_documents(
        self,
        filters: Optional[Dict[str, Any]] = None
    ) -> List[Document]:
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        where_clause = "1=1"
        params = []
        
        if filters:
            for key, value in filters.items():
                where_clause += f" AND JSON_EXTRACT(meta, '$.{key}') = %s"
                params.append(json.dumps(value))
        
        cursor.execute(f"""
            SELECT id, content, embedding, meta, score
            FROM {self.table_name}
            WHERE {where_clause}
        """, params)
        
        documents = []
        for row in cursor.fetchall():
            embedding = None
            if row['embedding']:
                embedding = np.frombuffer(row['embedding'], dtype=np.float32).tolist()
            
            doc = Document(
                id=row['id'],
                content=row['content'],
                embedding=embedding,
                meta=json.loads(row['meta']) if row['meta'] else {},
                score=row['score']
            )
            documents.append(doc)
        
        cursor.close()
        conn.close()
        return documents
    
    def write_documents(
        self,
        documents: List[Document],
        policy: DuplicatePolicy = DuplicatePolicy.NONE
    ) -> int:
        conn = self._get_connection()
        cursor = conn.cursor()
        written = 0
        
        for doc in documents:
            embedding_blob = None
            if doc.embedding:
                embedding_blob = np.array(doc.embedding, dtype=np.float32).tobytes()
            
            try:
                if policy == DuplicatePolicy.OVERWRITE:
                    cursor.execute(f"""
                        INSERT INTO {self.table_name} (id, content, embedding, meta, score)
                        VALUES (%s, %s, %s, %s, %s)
                        ON DUPLICATE KEY UPDATE
                            content = VALUES(content),
                            embedding = VALUES(embedding),
                            meta = VALUES(meta),
                            score = VALUES(score)
                    """, (
                        doc.id,
                        doc.content,
                        embedding_blob,
                        json.dumps(doc.meta) if doc.meta else None,
                        doc.score
                    ))
                else:
                    cursor.execute(f"""
                        INSERT IGNORE INTO {self.table_name} (id, content, embedding, meta, score)
                        VALUES (%s, %s, %s, %s, %s)
                    """, (
                        doc.id,
                        doc.content,
                        embedding_blob,
                        json.dumps(doc.meta) if doc.meta else None,
                        doc.score
                    ))
                written += cursor.rowcount
            except Exception as e:
                if policy == DuplicatePolicy.FAIL:
                    raise e
        
        conn.commit()
        cursor.close()
        conn.close()
        return written
    
    def delete_documents(self, document_ids: List[str]) -> None:
        conn = self._get_connection()
        cursor = conn.cursor()
        placeholders = ','.join(['%s'] * len(document_ids))
        cursor.execute(f"""
            DELETE FROM {self.table_name} WHERE id IN ({placeholders})
        """, document_ids)
        conn.commit()
        cursor.close()
        conn.close()


class MariaDBEmbeddingRetriever:
    """
    Embedding-based retriever for Haystack using MariaDB.
    """
    
    def __init__(
        self,
        document_store: MariaDBDocumentStore,
        top_k: int = 10
    ):
        self.document_store = document_store
        self.top_k = top_k
    
    def run(
        self,
        query_embedding: List[float],
        filters: Optional[Dict] = None,
        top_k: Optional[int] = None
    ) -> Dict[str, List[Document]]:
        """Retrieve documents by embedding similarity."""
        
        k = top_k or self.top_k
        query_vector = np.array(query_embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        # Get all documents (with optional filters)
        all_docs = self.document_store.filter_documents(filters)
        
        # Calculate similarities
        results = []
        for doc in all_docs:
            if doc.embedding:
                doc_vector = np.array(doc.embedding, dtype=np.float32)
                doc_norm = doc_vector / np.linalg.norm(doc_vector)
                similarity = float(np.dot(query_norm, doc_norm))
                doc.score = similarity
                results.append(doc)
        
        # Sort and return top k
        results.sort(key=lambda x: x.score or 0, reverse=True)
        return {"documents": results[:k]}


def create_haystack_rag_pipeline(
    document_store: MariaDBDocumentStore,
    model: str = "gpt-4"
) -> Pipeline:
    """
    Create a Haystack RAG pipeline with MariaDB.
    """
    
    # Components
    retriever = MariaDBEmbeddingRetriever(document_store=document_store)
    
    prompt_builder = PromptBuilder(template="""
Given the following documents, answer the question.
If the answer is not in the documents, say so.

Documents:
{% for doc in documents %}
{{ doc.content }}
{% endfor %}

Question: {{ question }}
Answer:
""")
    
    generator = OpenAIGenerator(model=model)
    
    # Build pipeline
    pipeline = Pipeline()
    pipeline.add_component("retriever", retriever)
    pipeline.add_component("prompt_builder", prompt_builder)
    pipeline.add_component("generator", generator)
    
    pipeline.connect("retriever.documents", "prompt_builder.documents")
    pipeline.connect("prompt_builder", "generator")
    
    return pipeline
```

### Semantic Kernel (Microsoft)

```python
#!/usr/bin/env python3
"""
semantic_kernel_mariadb.py
Intégration de Semantic Kernel avec MariaDB
"""

import semantic_kernel as sk
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion
from semantic_kernel.memory.semantic_text_memory import SemanticTextMemory
from semantic_kernel.connectors.memory.memory_store_base import MemoryStoreBase
from semantic_kernel.memory.memory_record import MemoryRecord
from typing import List, Optional, Dict, Any
import numpy as np
import mariadb
import json
from datetime import datetime


class MariaDBMemoryStore(MemoryStoreBase):
    """
    Semantic Kernel Memory Store implementation for MariaDB.
    """
    
    def __init__(self, connection_config: Dict[str, Any]):
        self.connection_config = connection_config
        self._collections = set()
    
    def _get_connection(self):
        return mariadb.connect(**self.connection_config)
    
    async def create_collection_async(self, collection_name: str) -> None:
        """Create a new collection (table)."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        table_name = f"sk_memory_{collection_name}"
        cursor.execute(f"""
            CREATE TABLE IF NOT EXISTS `{table_name}` (
                id VARCHAR(255) PRIMARY KEY,
                text TEXT NOT NULL,
                description TEXT,
                embedding BLOB,
                metadata JSON,
                timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                
                FULLTEXT INDEX ft_text (text, description)
            ) ENGINE=InnoDB
        """)
        
        conn.commit()
        cursor.close()
        conn.close()
        
        self._collections.add(collection_name)
    
    async def get_collections_async(self) -> List[str]:
        """Get all collections."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        cursor.execute("""
            SELECT table_name FROM information_schema.tables
            WHERE table_name LIKE 'sk_memory_%'
        """)
        
        collections = [row[0].replace('sk_memory_', '') for row in cursor.fetchall()]
        
        cursor.close()
        conn.close()
        
        return collections
    
    async def delete_collection_async(self, collection_name: str) -> None:
        """Delete a collection."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        table_name = f"sk_memory_{collection_name}"
        cursor.execute(f"DROP TABLE IF EXISTS `{table_name}`")
        
        conn.commit()
        cursor.close()
        conn.close()
        
        self._collections.discard(collection_name)
    
    async def does_collection_exist_async(self, collection_name: str) -> bool:
        """Check if collection exists."""
        collections = await self.get_collections_async()
        return collection_name in collections
    
    async def upsert_async(
        self,
        collection_name: str,
        record: MemoryRecord
    ) -> str:
        """Insert or update a memory record."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        table_name = f"sk_memory_{collection_name}"
        embedding_blob = None
        if record.embedding is not None:
            embedding_blob = np.array(record.embedding, dtype=np.float32).tobytes()
        
        cursor.execute(f"""
            INSERT INTO `{table_name}` (id, text, description, embedding, metadata)
            VALUES (%s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
                text = VALUES(text),
                description = VALUES(description),
                embedding = VALUES(embedding),
                metadata = VALUES(metadata),
                timestamp = NOW()
        """, (
            record.id,
            record.text,
            record.description,
            embedding_blob,
            json.dumps(record.additional_metadata) if record.additional_metadata else None
        ))
        
        conn.commit()
        cursor.close()
        conn.close()
        
        return record.id
    
    async def upsert_batch_async(
        self,
        collection_name: str,
        records: List[MemoryRecord]
    ) -> List[str]:
        """Batch upsert."""
        ids = []
        for record in records:
            id = await self.upsert_async(collection_name, record)
            ids.append(id)
        return ids
    
    async def get_async(
        self,
        collection_name: str,
        key: str,
        with_embedding: bool = False
    ) -> Optional[MemoryRecord]:
        """Get a record by key."""
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        table_name = f"sk_memory_{collection_name}"
        cursor.execute(f"""
            SELECT id, text, description, embedding, metadata, timestamp
            FROM `{table_name}`
            WHERE id = %s
        """, (key,))
        
        row = cursor.fetchone()
        cursor.close()
        conn.close()
        
        if not row:
            return None
        
        embedding = None
        if with_embedding and row['embedding']:
            embedding = np.frombuffer(row['embedding'], dtype=np.float32).tolist()
        
        return MemoryRecord(
            id=row['id'],
            text=row['text'],
            description=row['description'],
            embedding=embedding,
            additional_metadata=json.loads(row['metadata']) if row['metadata'] else {},
            timestamp=row['timestamp']
        )
    
    async def get_batch_async(
        self,
        collection_name: str,
        keys: List[str],
        with_embeddings: bool = False
    ) -> List[MemoryRecord]:
        """Batch get."""
        records = []
        for key in keys:
            record = await self.get_async(collection_name, key, with_embeddings)
            if record:
                records.append(record)
        return records
    
    async def remove_async(self, collection_name: str, key: str) -> None:
        """Remove a record."""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        table_name = f"sk_memory_{collection_name}"
        cursor.execute(f"DELETE FROM `{table_name}` WHERE id = %s", (key,))
        
        conn.commit()
        cursor.close()
        conn.close()
    
    async def remove_batch_async(
        self,
        collection_name: str,
        keys: List[str]
    ) -> None:
        """Batch remove."""
        for key in keys:
            await self.remove_async(collection_name, key)
    
    async def get_nearest_matches_async(
        self,
        collection_name: str,
        embedding: List[float],
        limit: int = 10,
        min_relevance_score: float = 0.0,
        with_embeddings: bool = False
    ) -> List[tuple]:
        """Find nearest matches by embedding similarity."""
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        table_name = f"sk_memory_{collection_name}"
        cursor.execute(f"""
            SELECT id, text, description, embedding, metadata, timestamp
            FROM `{table_name}`
            WHERE embedding IS NOT NULL
        """)
        
        rows = cursor.fetchall()
        cursor.close()
        conn.close()
        
        query_vector = np.array(embedding, dtype=np.float32)
        query_norm = query_vector / np.linalg.norm(query_vector)
        
        results = []
        for row in rows:
            doc_vector = np.frombuffer(row['embedding'], dtype=np.float32)
            doc_norm = doc_vector / np.linalg.norm(doc_vector)
            similarity = float(np.dot(query_norm, doc_norm))
            
            if similarity >= min_relevance_score:
                emb = doc_vector.tolist() if with_embeddings else None
                record = MemoryRecord(
                    id=row['id'],
                    text=row['text'],
                    description=row['description'],
                    embedding=emb,
                    additional_metadata=json.loads(row['metadata']) if row['metadata'] else {},
                    timestamp=row['timestamp']
                )
                results.append((record, similarity))
        
        results.sort(key=lambda x: x[1], reverse=True)
        return results[:limit]
    
    async def get_nearest_match_async(
        self,
        collection_name: str,
        embedding: List[float],
        min_relevance_score: float = 0.0,
        with_embedding: bool = False
    ) -> Optional[tuple]:
        """Get single nearest match."""
        results = await self.get_nearest_matches_async(
            collection_name, embedding, limit=1,
            min_relevance_score=min_relevance_score,
            with_embeddings=with_embedding
        )
        return results[0] if results else None


async def create_semantic_kernel_app():
    """
    Create a Semantic Kernel application with MariaDB memory.
    """
    
    # Initialize kernel
    kernel = sk.Kernel()
    
    # Add AI service
    kernel.add_service(OpenAIChatCompletion(
        service_id="chat",
        ai_model_id="gpt-4"
    ))
    
    # Add MariaDB memory
    memory_store = MariaDBMemoryStore({
        "host": "localhost",
        "port": 3306,
        "user": "sk_user",
        "password": "password",
        "database": "sk_db"
    })
    
    # Create collection
    await memory_store.create_collection_async("knowledge")
    
    # Add some memories
    # (In practice, you'd use an embedding service)
    
    return kernel, memory_store
```

---

## Bonnes pratiques et optimisation

### Performance tips

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTIMISATION DES PERFORMANCES                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. BATCH EMBEDDING GENERATION                                              │
│  ══════════════════════════════                                             │
│                                                                             │
│  ❌ Mauvais : Un appel API par document                                     │
│  ```python                                                                  │
│  for doc in documents:                                                      │
│      embedding = embeddings.embed_query(doc)  # N appels API                │
│  ```                                                                        │
│                                                                             │
│  ✅ Bon : Batch embedding                                                   │
│  ```python                                                                  │
│  embeddings = embeddings.embed_documents(documents)  # 1 appel API          │
│  ```                                                                        │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  2. CONNECTION POOLING                                                      │
│  ═════════════════════                                                      │
│                                                                             │
│  ```python                                                                  │
│  # Initialiser le pool une fois                                             │
│  pool = mariadb.ConnectionPool(                                             │
│      pool_name="ai_pool",                                                   │
│      pool_size=10,  # Ajuster selon la charge                               │
│      **connection_config                                                    │
│  )                                                                          │
│                                                                             │
│  # Utiliser pour chaque requête                                             │
│  conn = pool.get_connection()                                               │
│  try:                                                                       │
│      # ... work ...                                                         │
│  finally:                                                                   │
│      conn.close()  # Retourne au pool                                       │
│  ```                                                                        │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  3. CACHING DES EMBEDDINGS                                                  │
│  ═════════════════════════                                                  │
│                                                                             │
│  ```python                                                                  │
│  from functools import lru_cache                                            │
│  import hashlib                                                             │
│                                                                             │
│  @lru_cache(maxsize=10000)                                                  │
│  def get_cached_embedding(text_hash: str) -> List[float]:                   │
│      return embeddings.embed_query(text_from_hash(text_hash))               │
│                                                                             │
│  # Ou utiliser Redis pour le cache distribué                                │
│  ```                                                                        │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  4. INDEX OPTIMAUX                                                          │
│  ═════════════════                                                          │
│                                                                             │
│  ```sql                                                                     │
│  -- Index pour les filtres metadata fréquents                               │
│  CREATE INDEX idx_category ON documents(                                    │
│      (CAST(JSON_EXTRACT(metadata, '$.category') AS CHAR(100)))              │
│  );                                                                         │
│                                                                             │
│  -- Index Full-Text pour hybrid search                                      │
│  CREATE FULLTEXT INDEX ft_content ON documents(content);                    │
│  ```                                                                        │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  5. PRÉ-FILTRAGE SQL AVANT CALCUL VECTORIEL                                 │
│  ═════════════════════════════════════════                                  │
│                                                                             │
│  ```python                                                                  │
│  # Filtrer par SQL d'abord, puis calculer les similarités                   │
│  cursor.execute("""                                                         │
│      SELECT * FROM documents                                                │
│      WHERE category = %s                                                    │
│        AND created_at > %s                                                  │
│      LIMIT 1000  -- Limiter les candidats                                   │
│  """, (category, date))                                                     │
│                                                                             │
│  candidates = cursor.fetchall()                                             │
│  # Calcul vectoriel sur les candidats seulement                             │
│  ```                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architecture de production

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE PRODUCTION                                │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                         Load Balancer                              │   │
│  └───────────────────────────────┬────────────────────────────────────┘   │
│                                  │                                        │
│              ┌───────────────────┼───────────────────┐                    │
│              │                   │                   │                    │
│              ▼                   ▼                   ▼                    │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐        │
│  │   API Server 1    │ │   API Server 2    │ │   API Server 3    │        │
│  │                   │ │                   │ │                   │        │
│  │  FastAPI +        │ │  FastAPI +        │ │  FastAPI +        │        │
│  │  LangChain        │ │  LangChain        │ │  LangChain        │        │
│  └─────────┬─────────┘ └─────────┬─────────┘ └─────────┬─────────┘        │
│            │                     │                     │                  │
│            └─────────────────────┼─────────────────────┘                  │
│                                  │                                        │
│              ┌───────────────────┼───────────────────┐                    │
│              │                   │                   │                    │
│              ▼                   ▼                   ▼                    │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐        │
│  │    Redis Cache    │ │   MariaDB         │ │   Embedding       │        │
│  │                   │ │   Cluster         │ │   Service         │        │
│  │  - Query cache    │ │   (Galera)        │ │   (GPU)           │        │
│  │  - Session cache  │ │                   │ │                   │        │
│  │  - Rate limiting  │ │  ┌─────────────┐  │ │  OpenAI /         │        │
│  │                   │ │  │   Primary   │  │ │  Local Model      │        │
│  └───────────────────┘ │  └─────────────┘  │ └───────────────────┘        │
│                        │  ┌─────────────┐  │                              │
│                        │  │  Replica 1  │  │                              │
│                        │  └─────────────┘  │                              │
│                        │  ┌─────────────┐  │                              │
│                        │  │  Replica 2  │  │                              │
│                        │  └─────────────┘  │                              │
│                        └───────────────────┘                              │
│                                                                           │
│  Composants additionnels :                                                │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐        │
│  │   Prometheus      │ │   Grafana         │ │   Celery          │        │
│  │   (Métriques)     │ │   (Dashboards)    │ │   (Background)    │        │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘        │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Points clés à retenir

- **LangChain** offre la plus grande flexibilité avec agents, chains et memory — idéal pour des applications complexes
- **LlamaIndex** excelle dans l'indexation et le querying de documents — parfait pour RAG avancé
- **Haystack** est production-ready avec des pipelines modulaires — bon pour le déploiement enterprise
- **Semantic Kernel** s'intègre bien dans l'écosystème Microsoft — idéal pour les environnements Azure
- **Le batch embedding** réduit drastiquement les coûts API (jusqu'à 10x moins d'appels)
- **Le connection pooling** est essentiel pour les performances en production
- **La recherche hybride** (Full-Text + Vector) améliore significativement la pertinence
- **MariaDB** peut servir à la fois de vector store, SQL store et memory store — simplifiant l'architecture

---

## 🔗 Ressources et références

- [📖 LangChain Documentation](https://python.langchain.com/docs/)
- [📖 LlamaIndex Documentation](https://docs.llamaindex.ai/)
- [📖 Haystack Documentation](https://docs.haystack.deepset.ai/)
- [📖 Semantic Kernel Documentation](https://learn.microsoft.com/semantic-kernel/)
- [📖 MariaDB Connector/Python](https://mariadb.com/docs/connect/programming-languages/python/)
- [📖 OpenAI Embeddings Best Practices](https://platform.openai.com/docs/guides/embeddings/use-cases)

---


⏭️ [Études de cas réels](/20-cas-usage-architectures/12-etudes-cas-reels.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.10 MariaDB MCP Server pour Intégration IA 🆕

> **Niveau** : Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Section 20.9 (Use Cases IA), notions de protocoles client-serveur, Python asyncio

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le Model Context Protocol (MCP) et son rôle dans l'écosystème IA
- Implémenter un serveur MCP complet pour MariaDB
- Configurer l'intégration avec Claude Desktop et d'autres clients MCP
- Sécuriser l'accès aux données via MCP
- Concevoir des architectures avancées combinant MCP, RAG et agents IA
- Optimiser les performances des requêtes initiées par IA

---

## Introduction

Le **Model Context Protocol (MCP)** est un protocole ouvert développé par Anthropic qui standardise la communication entre les Large Language Models (LLM) et les sources de données externes. Il permet aux assistants IA comme Claude d'accéder à des bases de données, APIs, et fichiers de manière sécurisée et contrôlée.

MariaDB MCP Server transforme votre base de données en une ressource accessible par les agents IA, permettant des interactions naturelles en langage humain avec vos données d'entreprise.

### Pourquoi MCP ?

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    LE PROBLÈME SANS MCP                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Avant MCP : Intégrations ad-hoc, fragiles et non-standardisées            │
│  ════════════════════════════════════════════════════════════              │
│                                                                            │
│  ┌─────────────┐                                                           │
│  │   Claude    │                                                           │
│  │    LLM      │                                                           │
│  └──────┬──────┘                                                           │
│         │                                                                  │
│         │ "Comment accéder à la base de données ?"                         │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Options actuelles :                                                │   │
│  │                                                                     │   │
│  │  1. Copier-coller manuel les données dans le prompt                 │   │
│  │     → Fastidieux, non-scalable, données obsolètes                   │   │
│  │                                                                     │   │
│  │  2. Développer une API custom + function calling                    │   │
│  │     → Complexe, maintenance lourde, pas réutilisable                │   │
│  │                                                                     │   │
│  │  3. Plugins propriétaires (ChatGPT plugins, etc.)                   │   │
│  │     → Vendor lock-in, standards incompatibles                       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                    LA SOLUTION MCP                                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Avec MCP : Un protocole standardisé pour toutes les intégrations          │
│  ═══════════════════════════════════════════════════════════════           │
│                                                                            │
│  ┌─────────────┐                                                           │
│  │   Claude    │                                                           │
│  │    LLM      │                                                           │
│  └──────┬──────┘                                                           │
│         │                                                                  │
│         │ MCP Protocol (standardisé)                                       │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      MCP Client (Claude Desktop)                    │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│         ┌────────────────────────┼────────────────────────┐                │
│         │                        │                        │                │
│         ▼                        ▼                        ▼                │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐         │
│  │  MariaDB    │          │   GitHub    │          │   Slack     │         │
│  │ MCP Server  │          │ MCP Server  │          │ MCP Server  │         │
│  └─────────────┘          └─────────────┘          └─────────────┘         │
│         │                        │                        │                │
│         ▼                        ▼                        ▼                │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐         │
│  │  MariaDB    │          │   GitHub    │          │   Slack     │         │
│  │  Database   │          │    API      │          │    API      │         │
│  └─────────────┘          └─────────────┘          └─────────────┘         │
│                                                                            │
│  Avantages :                                                               │
│  ✅ Standard ouvert (pas de vendor lock-in)                                │
│  ✅ Réutilisable entre différents LLM                                      │
│  ✅ Sécurité intégrée (permissions, audit)                                 │
│  ✅ Découverte automatique des capabilities                                │
│  ✅ Écosystème croissant de serveurs MCP                                   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture MCP

### Concepts fondamentaux

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE MCP                                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         MCP HOST                                    │   │
│  │                   (Claude Desktop, IDE, etc.)                       │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │                      MCP CLIENT                             │    │   │
│  │  │                                                             │    │   │
│  │  │  • Maintient les connexions aux serveurs                    │    │   │
│  │  │  • Route les requêtes du LLM vers les bons serveurs         │    │   │
│  │  │  • Agrège les réponses                                      │    │   │
│  │  │  • Gère l'authentification                                  │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                      │   │
│  └──────────────────────────────┼──────────────────────────────────────┘   │
│                                 │                                          │
│                    JSON-RPC over stdio / HTTP / WebSocket                  │
│                                 │                                          │
│  ┌──────────────────────────────┼──────────────────────────────────────┐   │
│  │                              ▼                                      │   │
│  │                         MCP SERVER                                  │   │
│  │                    (MariaDB MCP Server)                             │   │
│  │                                                                     │   │
│  │  Expose 3 types de primitives :                                     │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  1. TOOLS (Outils)                                          │    │   │
│  │  │     Actions que le LLM peut exécuter                        │    │   │
│  │  │                                                             │    │   │
│  │  │     • query(sql) → Exécuter une requête                     │    │   │
│  │  │     • describe_table(name) → Décrire une table              │    │   │
│  │  │     • list_tables() → Lister les tables                     │    │   │
│  │  │     • explain_query(sql) → Analyser un plan d'exécution     │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  2. RESOURCES (Ressources)                                  │    │   │
│  │  │     Données que le LLM peut lire                            │    │   │
│  │  │                                                             │    │   │
│  │  │     • mariadb://schema → Schéma complet                     │    │   │
│  │  │     • mariadb://tables/{name} → Définition d'une table      │    │   │
│  │  │     • mariadb://stats → Statistiques de la base             │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  3. PROMPTS (Modèles)                                       │    │   │
│  │  │     Templates de prompts réutilisables                      │    │   │
│  │  │                                                             │    │   │
│  │  │     • analyze_table → Analyse complète d'une table          │    │   │
│  │  │     • optimize_query → Suggestions d'optimisation           │    │   │
│  │  │     • generate_report → Génération de rapport               │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Flux de communication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FLUX DE COMMUNICATION MCP                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. INITIALISATION                                                          │
│  ══════════════════                                                         │
│                                                                             │
│  Client                              Server                                 │
│     │                                   │                                   │
│     │──── initialize ──────────────────►│                                   │
│     │     {protocolVersion, ...}        │                                   │
│     │                                   │                                   │
│     │◄─── initialize_result ────────────│                                   │
│     │     {capabilities, serverInfo}    │                                   │
│     │                                   │                                   │
│     │──── initialized ─────────────────►│                                   │
│     │                                   │                                   │
│                                                                             │
│  2. DÉCOUVERTE DES CAPABILITIES                                             │
│  ══════════════════════════════                                             │
│                                                                             │
│     │──── tools/list ──────────────────►│                                   │
│     │◄─── tools: [{name, schema},...] ──│                                   │
│     │                                   │                                   │
│     │──── resources/list ──────────────►│                                   │
│     │◄─── resources: [{uri, name},...] ─│                                   │
│     │                                   │                                   │
│                                                                             │
│  3. EXÉCUTION D'UN TOOL                                                     │
│  ══════════════════════                                                     │
│                                                                             │
│  User: "Montre-moi les 10 meilleurs clients"                                │
│                                                                             │
│  Claude analyse → décide d'utiliser le tool "query"                         │
│                                                                             │
│     │──── tools/call ──────────────────►│                                   │
│     │     {name: "query",               │                                   │
│     │      arguments: {                 │                                   │
│     │        sql: "SELECT * FROM        │                                   │
│     │              customers ORDER BY   │                                   │
│     │              total_orders DESC    │                                   │
│     │              LIMIT 10"            │                                   │
│     │      }}                           │                                   │
│     │                                   │                                   │
│     │◄─── result ───────────────────────│                                   │
│     │     {content: [{type: "text",     │                                   │
│     │                 text: "..."}]}    │                                   │
│     │                                   │                                   │
│                                                                             │
│  Claude génère une réponse formatée pour l'utilisateur                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implémentation complète du serveur MCP

### Structure du projet

```
mariadb-mcp-server/
├── pyproject.toml
├── README.md
├── src/
│   └── mariadb_mcp/
│       ├── __init__.py
│       ├── server.py          # Serveur MCP principal
│       ├── database.py        # Connexion MariaDB
│       ├── tools.py           # Définition des tools
│       ├── resources.py       # Définition des resources
│       ├── prompts.py         # Templates de prompts
│       ├── security.py        # Validation et sécurité
│       └── config.py          # Configuration
├── tests/
│   └── test_server.py
└── docker/
    ├── Dockerfile
    └── docker-compose.yml
```

### Configuration (config.py)

```python
#!/usr/bin/env python3
"""
config.py
Configuration du serveur MCP MariaDB
"""

from pydantic import BaseSettings, Field
from typing import List, Optional
import os


class MCPConfig(BaseSettings):
    """Configuration du serveur MCP"""
    
    # Connexion MariaDB
    db_host: str = Field(default="localhost", env="MARIADB_HOST")
    db_port: int = Field(default=3306, env="MARIADB_PORT")
    db_user: str = Field(default="mcp_user", env="MARIADB_USER")
    db_password: str = Field(default="", env="MARIADB_PASSWORD")
    db_name: str = Field(default="", env="MARIADB_DATABASE")
    
    # Sécurité
    read_only: bool = Field(default=True, env="MCP_READ_ONLY")
    max_rows: int = Field(default=1000, env="MCP_MAX_ROWS")
    query_timeout: int = Field(default=30, env="MCP_QUERY_TIMEOUT")
    allowed_tables: Optional[List[str]] = Field(default=None, env="MCP_ALLOWED_TABLES")
    blocked_tables: List[str] = Field(
        default=["mysql.user", "information_schema.user_privileges"],
        env="MCP_BLOCKED_TABLES"
    )
    
    # Rate limiting
    max_queries_per_minute: int = Field(default=60, env="MCP_RATE_LIMIT")
    
    # Logging
    log_level: str = Field(default="INFO", env="MCP_LOG_LEVEL")
    log_queries: bool = Field(default=True, env="MCP_LOG_QUERIES")
    
    # Features
    enable_explain: bool = Field(default=True, env="MCP_ENABLE_EXPLAIN")
    enable_schema_discovery: bool = Field(default=True, env="MCP_ENABLE_SCHEMA")
    enable_sample_data: bool = Field(default=True, env="MCP_ENABLE_SAMPLE")
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


def get_config() -> MCPConfig:
    """Retourne la configuration"""
    return MCPConfig()
```

### Sécurité (security.py)

```python
#!/usr/bin/env python3
"""
security.py
Validation et sécurité des requêtes SQL
"""

import re
import time
from typing import Optional, Tuple, List
from collections import defaultdict
import logging
from .config import MCPConfig

logger = logging.getLogger(__name__)


class SQLSecurityValidator:
    """
    Valide les requêtes SQL pour la sécurité.
    Implémente plusieurs couches de protection.
    """
    
    # Patterns dangereux
    DANGEROUS_PATTERNS = [
        r'\bINSERT\b',
        r'\bUPDATE\b',
        r'\bDELETE\b',
        r'\bDROP\b',
        r'\bCREATE\b',
        r'\bALTER\b',
        r'\bTRUNCATE\b',
        r'\bGRANT\b',
        r'\bREVOKE\b',
        r'\bEXEC\b',
        r'\bEXECUTE\b',
        r'\bCALL\b',
        r'\bLOAD\b',
        r'\bOUTFILE\b',
        r'\bINFILE\b',
        r'\bINTO\s+DUMPFILE\b',
        r'\bINTO\s+OUTFILE\b',
        r'\bBENCHMARK\b',
        r'\bSLEEP\b',
        r';\s*--',  # SQL injection pattern
        r'/\*.*\*/',  # Comments qui peuvent cacher du code
    ]
    
    # Tables système sensibles
    SENSITIVE_TABLES = [
        'mysql.user',
        'mysql.db',
        'mysql.tables_priv',
        'mysql.columns_priv',
        'mysql.procs_priv',
        'information_schema.user_privileges',
        'information_schema.schema_privileges',
        'performance_schema.users',
    ]
    
    def __init__(self, config: MCPConfig):
        self.config = config
        self.compiled_patterns = [
            re.compile(p, re.IGNORECASE) for p in self.DANGEROUS_PATTERNS
        ]
        
        # Rate limiting
        self.query_timestamps = defaultdict(list)
    
    def validate_query(self, sql: str, client_id: str = "default") -> Tuple[bool, Optional[str]]:
        """
        Valide une requête SQL.
        
        Args:
            sql: La requête à valider
            client_id: Identifiant du client pour rate limiting
            
        Returns:
            (is_valid, error_message)
        """
        sql = sql.strip()
        
        # 1. Vérifier le rate limiting
        if not self._check_rate_limit(client_id):
            return False, "Rate limit exceeded. Please wait before sending more queries."
        
        # 2. Vérifier que c'est un SELECT (si read_only)
        if self.config.read_only:
            if not sql.upper().startswith("SELECT") and not sql.upper().startswith("SHOW") and not sql.upper().startswith("DESCRIBE") and not sql.upper().startswith("EXPLAIN"):
                return False, "Only SELECT, SHOW, DESCRIBE, and EXPLAIN queries are allowed in read-only mode."
        
        # 3. Vérifier les patterns dangereux
        for pattern in self.compiled_patterns:
            if pattern.search(sql):
                logger.warning(f"Dangerous pattern detected in query: {sql[:100]}...")
                return False, f"Query contains forbidden pattern."
        
        # 4. Vérifier les tables sensibles
        sql_upper = sql.upper()
        for table in self.SENSITIVE_TABLES + self.config.blocked_tables:
            if table.upper() in sql_upper:
                return False, f"Access to table '{table}' is not allowed."
        
        # 5. Vérifier les tables autorisées (si configuré)
        if self.config.allowed_tables:
            if not self._check_allowed_tables(sql):
                return False, "Query references tables not in the allowed list."
        
        # 6. Vérifier la longueur
        if len(sql) > 10000:
            return False, "Query too long (max 10000 characters)."
        
        # 7. Vérifier les sous-requêtes imbriquées (limite)
        subquery_count = sql.upper().count("SELECT")
        if subquery_count > 5:
            return False, "Too many nested subqueries (max 5)."
        
        return True, None
    
    def _check_rate_limit(self, client_id: str) -> bool:
        """Vérifie le rate limit"""
        now = time.time()
        minute_ago = now - 60
        
        # Nettoyer les anciennes entrées
        self.query_timestamps[client_id] = [
            ts for ts in self.query_timestamps[client_id] if ts > minute_ago
        ]
        
        # Vérifier la limite
        if len(self.query_timestamps[client_id]) >= self.config.max_queries_per_minute:
            return False
        
        self.query_timestamps[client_id].append(now)
        return True
    
    def _check_allowed_tables(self, sql: str) -> bool:
        """Vérifie que seules les tables autorisées sont accédées"""
        # Extraction simplifiée des noms de tables
        # En production, utiliser un parser SQL comme sqlparse
        
        allowed_upper = [t.upper() for t in self.config.allowed_tables]
        
        # Pattern pour trouver les tables après FROM, JOIN, etc.
        table_pattern = r'(?:FROM|JOIN|INTO|UPDATE)\s+[`"]?(\w+)[`"]?'
        matches = re.findall(table_pattern, sql, re.IGNORECASE)
        
        for table in matches:
            if table.upper() not in allowed_upper:
                return False
        
        return True
    
    def sanitize_identifier(self, identifier: str) -> str:
        """Sanitize un identifiant SQL (nom de table, colonne)"""
        # Autoriser uniquement les caractères alphanumériques et underscore
        if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', identifier):
            raise ValueError(f"Invalid identifier: {identifier}")
        return identifier
    
    def add_limit_if_missing(self, sql: str, max_rows: int) -> str:
        """Ajoute une clause LIMIT si absente"""
        sql_upper = sql.upper().strip()
        
        if sql_upper.startswith("SELECT") and "LIMIT" not in sql_upper:
            sql = sql.rstrip(";") + f" LIMIT {max_rows}"
        
        return sql


class AuditLogger:
    """Logger d'audit pour les requêtes"""
    
    def __init__(self, config: MCPConfig):
        self.config = config
        self.audit_logger = logging.getLogger("mcp.audit")
    
    def log_query(
        self, 
        client_id: str, 
        query: str, 
        success: bool, 
        execution_time_ms: float,
        row_count: int = 0,
        error: Optional[str] = None
    ):
        """Log une requête pour audit"""
        if not self.config.log_queries:
            return
        
        log_entry = {
            "client_id": client_id,
            "query": query[:500],  # Tronquer les longues requêtes
            "success": success,
            "execution_time_ms": execution_time_ms,
            "row_count": row_count,
            "error": error,
            "timestamp": time.time()
        }
        
        if success:
            self.audit_logger.info(f"Query executed: {log_entry}")
        else:
            self.audit_logger.warning(f"Query failed: {log_entry}")
```

### Database (database.py)

```python
#!/usr/bin/env python3
"""
database.py
Gestion de la connexion MariaDB
"""

import mariadb
from typing import List, Dict, Any, Optional, Tuple
from contextlib import contextmanager
import logging
import time
from .config import MCPConfig
from .security import SQLSecurityValidator, AuditLogger

logger = logging.getLogger(__name__)


class MariaDBConnection:
    """
    Gestionnaire de connexion MariaDB avec pooling et sécurité.
    """
    
    def __init__(self, config: MCPConfig):
        self.config = config
        self.security = SQLSecurityValidator(config)
        self.audit = AuditLogger(config)
        self._pool = None
    
    def _get_connection(self) -> mariadb.Connection:
        """Obtient une connexion depuis le pool"""
        if self._pool is None:
            self._pool = mariadb.ConnectionPool(
                host=self.config.db_host,
                port=self.config.db_port,
                user=self.config.db_user,
                password=self.config.db_password,
                database=self.config.db_name,
                pool_name="mcp_pool",
                pool_size=5,
                connect_timeout=10
            )
        return self._pool.get_connection()
    
    @contextmanager
    def get_cursor(self, dictionary: bool = True):
        """Context manager pour obtenir un curseur"""
        conn = self._get_connection()
        try:
            cursor = conn.cursor(dictionary=dictionary)
            yield cursor
        finally:
            cursor.close()
            conn.close()
    
    def execute_query(
        self, 
        sql: str, 
        client_id: str = "default"
    ) -> Tuple[bool, Any, Optional[str]]:
        """
        Exécute une requête SQL de manière sécurisée.
        
        Args:
            sql: La requête SQL
            client_id: Identifiant du client
            
        Returns:
            (success, result, error_message)
        """
        start_time = time.time()
        
        # Validation de sécurité
        is_valid, error = self.security.validate_query(sql, client_id)
        if not is_valid:
            self.audit.log_query(client_id, sql, False, 0, error=error)
            return False, None, error
        
        # Ajouter LIMIT si nécessaire
        sql = self.security.add_limit_if_missing(sql, self.config.max_rows)
        
        try:
            with self.get_cursor() as cursor:
                # Timeout
                cursor.execute(f"SET SESSION max_statement_time = {self.config.query_timeout}")
                
                # Exécuter
                cursor.execute(sql)
                
                # Récupérer les résultats
                if cursor.description:  # SELECT query
                    columns = [desc[0] for desc in cursor.description]
                    rows = cursor.fetchall()
                    result = {
                        "columns": columns,
                        "rows": rows,
                        "row_count": len(rows)
                    }
                else:
                    result = {"affected_rows": cursor.rowcount}
                
                execution_time = (time.time() - start_time) * 1000
                self.audit.log_query(
                    client_id, sql, True, execution_time, 
                    row_count=result.get("row_count", 0)
                )
                
                return True, result, None
                
        except mariadb.Error as e:
            execution_time = (time.time() - start_time) * 1000
            error_msg = str(e)
            self.audit.log_query(client_id, sql, False, execution_time, error=error_msg)
            logger.error(f"Query error: {error_msg}")
            return False, None, f"Database error: {error_msg}"
    
    def get_schema(self) -> Dict[str, Any]:
        """Récupère le schéma complet de la base de données"""
        schema = {"database": self.config.db_name, "tables": {}}
        
        with self.get_cursor() as cursor:
            # Lister les tables
            cursor.execute("SHOW TABLES")
            tables = [row[f"Tables_in_{self.config.db_name}"] for row in cursor.fetchall()]
            
            for table in tables:
                # Vérifier si la table est autorisée
                if self.config.allowed_tables and table not in self.config.allowed_tables:
                    continue
                if table in self.config.blocked_tables:
                    continue
                
                table_info = {"columns": [], "indexes": [], "row_count": 0}
                
                # Colonnes
                cursor.execute(f"DESCRIBE `{table}`")
                for col in cursor.fetchall():
                    table_info["columns"].append({
                        "name": col["Field"],
                        "type": col["Type"],
                        "nullable": col["Null"] == "YES",
                        "key": col["Key"],
                        "default": col["Default"],
                        "extra": col["Extra"]
                    })
                
                # Index
                cursor.execute(f"SHOW INDEX FROM `{table}`")
                indexes = {}
                for idx in cursor.fetchall():
                    idx_name = idx["Key_name"]
                    if idx_name not in indexes:
                        indexes[idx_name] = {
                            "name": idx_name,
                            "unique": not idx["Non_unique"],
                            "columns": []
                        }
                    indexes[idx_name]["columns"].append(idx["Column_name"])
                table_info["indexes"] = list(indexes.values())
                
                # Nombre de lignes (approximatif)
                cursor.execute(f"SELECT COUNT(*) as cnt FROM `{table}`")
                table_info["row_count"] = cursor.fetchone()["cnt"]
                
                schema["tables"][table] = table_info
        
        return schema
    
    def get_table_info(self, table_name: str) -> Dict[str, Any]:
        """Récupère les informations détaillées d'une table"""
        # Sanitize le nom de table
        table_name = self.security.sanitize_identifier(table_name)
        
        with self.get_cursor() as cursor:
            info = {
                "name": table_name,
                "columns": [],
                "indexes": [],
                "foreign_keys": [],
                "create_statement": "",
                "row_count": 0,
                "sample_data": []
            }
            
            # Colonnes
            cursor.execute(f"DESCRIBE `{table_name}`")
            info["columns"] = cursor.fetchall()
            
            # CREATE statement
            cursor.execute(f"SHOW CREATE TABLE `{table_name}`")
            result = cursor.fetchone()
            info["create_statement"] = result.get("Create Table", "")
            
            # Row count
            cursor.execute(f"SELECT COUNT(*) as cnt FROM `{table_name}`")
            info["row_count"] = cursor.fetchone()["cnt"]
            
            # Sample data
            if self.config.enable_sample_data:
                cursor.execute(f"SELECT * FROM `{table_name}` LIMIT 5")
                info["sample_data"] = cursor.fetchall()
            
            return info
    
    def explain_query(self, sql: str) -> Dict[str, Any]:
        """Exécute EXPLAIN sur une requête"""
        if not self.config.enable_explain:
            return {"error": "EXPLAIN is disabled"}
        
        # Validation basique
        is_valid, error = self.security.validate_query(sql)
        if not is_valid:
            return {"error": error}
        
        with self.get_cursor() as cursor:
            cursor.execute(f"EXPLAIN {sql}")
            explain_result = cursor.fetchall()
            
            cursor.execute(f"EXPLAIN FORMAT=JSON {sql}")
            explain_json = cursor.fetchone()
            
            return {
                "explain": explain_result,
                "explain_json": explain_json
            }
    
    def list_tables(self) -> List[Dict[str, Any]]:
        """Liste toutes les tables avec leurs statistiques basiques"""
        tables = []
        
        with self.get_cursor() as cursor:
            cursor.execute("SHOW TABLE STATUS")
            for row in cursor.fetchall():
                table_name = row["Name"]
                
                # Filtrage
                if self.config.allowed_tables and table_name not in self.config.allowed_tables:
                    continue
                if table_name in self.config.blocked_tables:
                    continue
                
                tables.append({
                    "name": table_name,
                    "engine": row["Engine"],
                    "rows": row["Rows"],
                    "data_length": row["Data_length"],
                    "index_length": row["Index_length"],
                    "created": str(row["Create_time"]) if row["Create_time"] else None,
                    "updated": str(row["Update_time"]) if row["Update_time"] else None
                })
        
        return tables
```

### Serveur MCP principal (server.py)

```python
#!/usr/bin/env python3
"""
server.py
Serveur MCP pour MariaDB
"""

import asyncio
import json
import logging
from typing import Any, List, Dict, Optional
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    TextContent,
    Resource,
    ResourceContents,
    Prompt,
    PromptMessage,
    PromptArgument,
    GetPromptResult,
)

from .config import get_config, MCPConfig
from .database import MariaDBConnection

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class MariaDBMCPServer:
    """
    Serveur MCP complet pour MariaDB.
    
    Expose:
    - Tools: query, describe_table, list_tables, explain_query, get_sample
    - Resources: schema, table definitions, statistics
    - Prompts: analyze_table, optimize_query, generate_report
    """
    
    def __init__(self, config: Optional[MCPConfig] = None):
        self.config = config or get_config()
        self.db = MariaDBConnection(self.config)
        self.server = Server("mariadb-mcp")
        self._setup_handlers()
    
    def _setup_handlers(self):
        """Configure tous les handlers MCP"""
        self._setup_tools()
        self._setup_resources()
        self._setup_prompts()
    
    # ═══════════════════════════════════════════════════════════════════════
    # TOOLS
    # ═══════════════════════════════════════════════════════════════════════
    
    def _setup_tools(self):
        @self.server.list_tools()
        async def list_tools() -> List[Tool]:
            tools = [
                Tool(
                    name="query",
                    description="""Execute a SQL query on the MariaDB database.
                    
Only SELECT, SHOW, DESCRIBE, and EXPLAIN queries are allowed.
Results are limited to {max_rows} rows.
Use this to retrieve data, explore tables, and answer questions about the database.""".format(
                        max_rows=self.config.max_rows
                    ),
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "sql": {
                                "type": "string",
                                "description": "The SQL query to execute. Must be a SELECT, SHOW, DESCRIBE, or EXPLAIN statement."
                            }
                        },
                        "required": ["sql"]
                    }
                ),
                Tool(
                    name="list_tables",
                    description="List all available tables in the database with their basic statistics (row count, size, engine).",
                    inputSchema={
                        "type": "object",
                        "properties": {}
                    }
                ),
                Tool(
                    name="describe_table",
                    description="Get detailed information about a specific table including columns, indexes, and sample data.",
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
                    name="get_sample",
                    description="Get sample rows from a table to understand its data structure.",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "table_name": {
                                "type": "string",
                                "description": "Name of the table"
                            },
                            "limit": {
                                "type": "integer",
                                "description": "Number of rows to return (default: 5, max: 20)",
                                "default": 5
                            }
                        },
                        "required": ["table_name"]
                    }
                ),
            ]
            
            if self.config.enable_explain:
                tools.append(Tool(
                    name="explain_query",
                    description="Analyze a query's execution plan to understand performance characteristics.",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "sql": {
                                "type": "string",
                                "description": "The SQL query to analyze"
                            }
                        },
                        "required": ["sql"]
                    }
                ))
            
            return tools
        
        @self.server.call_tool()
        async def call_tool(name: str, arguments: Dict[str, Any]) -> List[TextContent]:
            try:
                if name == "query":
                    result = self._handle_query(arguments["sql"])
                elif name == "list_tables":
                    result = self._handle_list_tables()
                elif name == "describe_table":
                    result = self._handle_describe_table(arguments["table_name"])
                elif name == "get_sample":
                    result = self._handle_get_sample(
                        arguments["table_name"],
                        arguments.get("limit", 5)
                    )
                elif name == "explain_query":
                    result = self._handle_explain(arguments["sql"])
                else:
                    result = f"Unknown tool: {name}"
                
                return [TextContent(type="text", text=result)]
                
            except Exception as e:
                logger.exception(f"Error in tool {name}")
                return [TextContent(type="text", text=f"Error: {str(e)}")]
    
    def _handle_query(self, sql: str) -> str:
        """Exécute une requête SQL"""
        success, result, error = self.db.execute_query(sql)
        
        if not success:
            return f"Query failed: {error}"
        
        # Formater le résultat
        if "rows" in result:
            output = f"Query returned {result['row_count']} rows:\n\n"
            
            if result['row_count'] == 0:
                return "Query returned no results."
            
            # Format as markdown table for better readability
            columns = result["columns"]
            rows = result["rows"]
            
            # Header
            output += "| " + " | ".join(str(c) for c in columns) + " |\n"
            output += "| " + " | ".join("---" for _ in columns) + " |\n"
            
            # Rows (limit display to 50 for readability)
            for row in rows[:50]:
                values = [str(row.get(c, "NULL"))[:50] for c in columns]
                output += "| " + " | ".join(values) + " |\n"
            
            if len(rows) > 50:
                output += f"\n... and {len(rows) - 50} more rows"
            
            return output
        else:
            return f"Query executed. Affected rows: {result.get('affected_rows', 0)}"
    
    def _handle_list_tables(self) -> str:
        """Liste les tables"""
        tables = self.db.list_tables()
        
        if not tables:
            return "No tables found in the database."
        
        output = f"Found {len(tables)} tables:\n\n"
        output += "| Table | Engine | Rows | Data Size | Index Size |\n"
        output += "| --- | --- | --- | --- | --- |\n"
        
        for t in tables:
            data_size = self._format_size(t["data_length"])
            index_size = self._format_size(t["index_length"])
            output += f"| {t['name']} | {t['engine']} | {t['rows']:,} | {data_size} | {index_size} |\n"
        
        return output
    
    def _handle_describe_table(self, table_name: str) -> str:
        """Décrit une table"""
        try:
            info = self.db.get_table_info(table_name)
        except ValueError as e:
            return f"Error: {e}"
        
        output = f"## Table: {table_name}\n\n"
        output += f"**Row count:** {info['row_count']:,}\n\n"
        
        output += "### Columns\n\n"
        output += "| Column | Type | Nullable | Key | Default | Extra |\n"
        output += "| --- | --- | --- | --- | --- | --- |\n"
        
        for col in info["columns"]:
            output += f"| {col['Field']} | {col['Type']} | {col['Null']} | {col['Key'] or ''} | {col['Default'] or ''} | {col['Extra'] or ''} |\n"
        
        if info.get("sample_data"):
            output += "\n### Sample Data\n\n"
            output += "```json\n"
            output += json.dumps(info["sample_data"][:3], indent=2, default=str)
            output += "\n```\n"
        
        return output
    
    def _handle_get_sample(self, table_name: str, limit: int) -> str:
        """Récupère des données d'exemple"""
        limit = min(limit, 20)  # Cap at 20
        
        try:
            sql = f"SELECT * FROM `{self.db.security.sanitize_identifier(table_name)}` LIMIT {limit}"
            success, result, error = self.db.execute_query(sql)
            
            if not success:
                return f"Error: {error}"
            
            if result["row_count"] == 0:
                return f"Table '{table_name}' is empty."
            
            return f"Sample data from '{table_name}':\n\n```json\n{json.dumps(result['rows'], indent=2, default=str)}\n```"
            
        except ValueError as e:
            return f"Error: {e}"
    
    def _handle_explain(self, sql: str) -> str:
        """Analyse un plan d'exécution"""
        result = self.db.explain_query(sql)
        
        if "error" in result:
            return f"Error: {result['error']}"
        
        output = "## Query Execution Plan\n\n"
        output += "```\n"
        
        for row in result["explain"]:
            output += f"id: {row.get('id')}, "
            output += f"select_type: {row.get('select_type')}, "
            output += f"table: {row.get('table')}, "
            output += f"type: {row.get('type')}, "
            output += f"key: {row.get('key')}, "
            output += f"rows: {row.get('rows')}, "
            output += f"Extra: {row.get('Extra')}\n"
        
        output += "```\n"
        
        return output
    
    def _format_size(self, size_bytes: int) -> str:
        """Formate une taille en bytes"""
        if size_bytes is None:
            return "N/A"
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size_bytes < 1024:
                return f"{size_bytes:.1f} {unit}"
            size_bytes /= 1024
        return f"{size_bytes:.1f} TB"
    
    # ═══════════════════════════════════════════════════════════════════════
    # RESOURCES
    # ═══════════════════════════════════════════════════════════════════════
    
    def _setup_resources(self):
        @self.server.list_resources()
        async def list_resources() -> List[Resource]:
            resources = [
                Resource(
                    uri="mariadb://schema",
                    name="Database Schema",
                    description="Complete schema of the database including all tables, columns, and indexes",
                    mimeType="application/json"
                ),
                Resource(
                    uri="mariadb://stats",
                    name="Database Statistics",
                    description="Database statistics and health metrics",
                    mimeType="application/json"
                )
            ]
            
            # Ajouter une resource par table
            if self.config.enable_schema_discovery:
                tables = self.db.list_tables()
                for table in tables:
                    resources.append(Resource(
                        uri=f"mariadb://tables/{table['name']}",
                        name=f"Table: {table['name']}",
                        description=f"Schema and sample data for table {table['name']}",
                        mimeType="application/json"
                    ))
            
            return resources
        
        @self.server.read_resource()
        async def read_resource(uri: str) -> ResourceContents:
            if uri == "mariadb://schema":
                schema = self.db.get_schema()
                return ResourceContents(
                    uri=uri,
                    mimeType="application/json",
                    text=json.dumps(schema, indent=2, default=str)
                )
            
            elif uri == "mariadb://stats":
                stats = self._get_stats()
                return ResourceContents(
                    uri=uri,
                    mimeType="application/json",
                    text=json.dumps(stats, indent=2, default=str)
                )
            
            elif uri.startswith("mariadb://tables/"):
                table_name = uri.split("/")[-1]
                info = self.db.get_table_info(table_name)
                return ResourceContents(
                    uri=uri,
                    mimeType="application/json",
                    text=json.dumps(info, indent=2, default=str)
                )
            
            raise ValueError(f"Unknown resource: {uri}")
    
    def _get_stats(self) -> Dict[str, Any]:
        """Récupère les statistiques de la base"""
        with self.db.get_cursor() as cursor:
            stats = {}
            
            # Variables globales
            cursor.execute("SHOW GLOBAL STATUS LIKE 'Questions'")
            stats["total_queries"] = int(cursor.fetchone()["Value"])
            
            cursor.execute("SHOW GLOBAL STATUS LIKE 'Uptime'")
            stats["uptime_seconds"] = int(cursor.fetchone()["Value"])
            
            cursor.execute("SHOW GLOBAL STATUS LIKE 'Threads_connected'")
            stats["connections"] = int(cursor.fetchone()["Value"])
            
            # Taille de la base
            cursor.execute("""
                SELECT 
                    SUM(data_length + index_length) as total_size,
                    COUNT(*) as table_count
                FROM information_schema.tables 
                WHERE table_schema = %s
            """, (self.config.db_name,))
            result = cursor.fetchone()
            stats["total_size_bytes"] = result["total_size"]
            stats["table_count"] = result["table_count"]
            
            return stats
    
    # ═══════════════════════════════════════════════════════════════════════
    # PROMPTS
    # ═══════════════════════════════════════════════════════════════════════
    
    def _setup_prompts(self):
        @self.server.list_prompts()
        async def list_prompts() -> List[Prompt]:
            return [
                Prompt(
                    name="analyze_table",
                    description="Perform a comprehensive analysis of a database table",
                    arguments=[
                        PromptArgument(
                            name="table_name",
                            description="Name of the table to analyze",
                            required=True
                        )
                    ]
                ),
                Prompt(
                    name="optimize_query",
                    description="Get suggestions to optimize a SQL query",
                    arguments=[
                        PromptArgument(
                            name="query",
                            description="The SQL query to optimize",
                            required=True
                        )
                    ]
                ),
                Prompt(
                    name="generate_report",
                    description="Generate a report based on database data",
                    arguments=[
                        PromptArgument(
                            name="topic",
                            description="What the report should be about",
                            required=True
                        )
                    ]
                )
            ]
        
        @self.server.get_prompt()
        async def get_prompt(name: str, arguments: Optional[Dict[str, str]]) -> GetPromptResult:
            if name == "analyze_table":
                table_name = arguments.get("table_name", "")
                return GetPromptResult(
                    description=f"Analyze table {table_name}",
                    messages=[
                        PromptMessage(
                            role="user",
                            content=TextContent(
                                type="text",
                                text=f"""Please perform a comprehensive analysis of the '{table_name}' table.

Use the available tools to:
1. First, describe the table structure (columns, types, indexes)
2. Get sample data to understand the content
3. Run aggregate queries to understand data distribution
4. Identify potential issues (missing indexes, data quality)

Provide your analysis in a clear, structured format with:
- Table Overview
- Column Analysis
- Data Distribution
- Recommendations for optimization
"""
                            )
                        )
                    ]
                )
            
            elif name == "optimize_query":
                query = arguments.get("query", "")
                return GetPromptResult(
                    description="Optimize SQL query",
                    messages=[
                        PromptMessage(
                            role="user",
                            content=TextContent(
                                type="text",
                                text=f"""Please analyze and optimize this SQL query:

```sql
{query}
```

Use the explain_query tool to understand the execution plan, then:
1. Identify performance bottlenecks
2. Suggest index improvements
3. Propose query rewrites if beneficial
4. Estimate the improvement potential
"""
                            )
                        )
                    ]
                )
            
            elif name == "generate_report":
                topic = arguments.get("topic", "")
                return GetPromptResult(
                    description=f"Generate report about {topic}",
                    messages=[
                        PromptMessage(
                            role="user",
                            content=TextContent(
                                type="text",
                                text=f"""Generate a detailed report about: {topic}

Use the available database tools to:
1. Identify relevant tables
2. Query the necessary data
3. Perform calculations and aggregations
4. Present findings with data visualizations described in text

Format the report with:
- Executive Summary
- Key Metrics
- Detailed Analysis
- Conclusions and Recommendations
"""
                            )
                        )
                    ]
                )
            
            raise ValueError(f"Unknown prompt: {name}")
    
    # ═══════════════════════════════════════════════════════════════════════
    # RUN
    # ═══════════════════════════════════════════════════════════════════════
    
    async def run(self):
        """Démarre le serveur MCP"""
        logger.info(f"Starting MariaDB MCP Server for database: {self.config.db_name}")
        
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream,
                write_stream,
                self.server.create_initialization_options()
            )


def main():
    """Point d'entrée principal"""
    import sys
    
    config = get_config()
    
    # Validation de la configuration
    if not config.db_name:
        print("Error: MARIADB_DATABASE environment variable is required", file=sys.stderr)
        sys.exit(1)
    
    server = MariaDBMCPServer(config)
    asyncio.run(server.run())


if __name__ == "__main__":
    main()
```

---

## Configuration de Claude Desktop

### Fichier de configuration

```json
// macOS: ~/Library/Application Support/Claude/claude_desktop_config.json
// Windows: %APPDATA%\Claude\claude_desktop_config.json
// Linux: ~/.config/Claude/claude_desktop_config.json

{
  "mcpServers": {
    "mariadb-ecommerce": {
      "command": "python",
      "args": ["-m", "mariadb_mcp"],
      "env": {
        "MARIADB_HOST": "localhost",
        "MARIADB_PORT": "3306",
        "MARIADB_USER": "mcp_readonly",
        "MARIADB_PASSWORD": "your_secure_password",
        "MARIADB_DATABASE": "ecommerce",
        "MCP_READ_ONLY": "true",
        "MCP_MAX_ROWS": "1000",
        "MCP_LOG_QUERIES": "true"
      }
    },
    "mariadb-analytics": {
      "command": "python",
      "args": ["-m", "mariadb_mcp"],
      "env": {
        "MARIADB_HOST": "analytics-db.internal",
        "MARIADB_PORT": "3306",
        "MARIADB_USER": "mcp_analytics",
        "MARIADB_PASSWORD": "analytics_password",
        "MARIADB_DATABASE": "analytics_dw",
        "MCP_READ_ONLY": "true",
        "MCP_MAX_ROWS": "5000",
        "MCP_QUERY_TIMEOUT": "60"
      }
    }
  }
}
```

### Utilisateur MariaDB dédié

```sql
-- ═══════════════════════════════════════════════════════════════════════════
-- CRÉATION D'UN UTILISATEUR MCP SÉCURISÉ
-- ═══════════════════════════════════════════════════════════════════════════

-- Utilisateur read-only avec permissions minimales
CREATE USER 'mcp_readonly'@'localhost' 
    IDENTIFIED BY 'your_secure_password_here'
    PASSWORD EXPIRE NEVER;

-- Permissions SELECT uniquement sur les tables métier
GRANT SELECT ON ecommerce.customers TO 'mcp_readonly'@'localhost';
GRANT SELECT ON ecommerce.orders TO 'mcp_readonly'@'localhost';
GRANT SELECT ON ecommerce.products TO 'mcp_readonly'@'localhost';
GRANT SELECT ON ecommerce.order_items TO 'mcp_readonly'@'localhost';

-- Permettre de voir le schéma
GRANT SELECT ON information_schema.TABLES TO 'mcp_readonly'@'localhost';
GRANT SELECT ON information_schema.COLUMNS TO 'mcp_readonly'@'localhost';
GRANT SELECT ON information_schema.STATISTICS TO 'mcp_readonly'@'localhost';

-- INTERDIT : Pas d'accès aux tables système sensibles
-- NO GRANT sur mysql.*, performance_schema.*

-- Limites de ressources
ALTER USER 'mcp_readonly'@'localhost'
    WITH MAX_QUERIES_PER_HOUR 1000
         MAX_CONNECTIONS_PER_HOUR 100
         MAX_USER_CONNECTIONS 5
         MAX_STATEMENT_TIME 30;  -- 30 secondes max par requête

FLUSH PRIVILEGES;

-- Vérification
SHOW GRANTS FOR 'mcp_readonly'@'localhost';
```

---

## Architectures avancées

### Multi-Database MCP

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE MULTI-DATABASE                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Claude Desktop                               │   │
│  │                                                                     │   │
│  │  User: "Compare les ventes du trimestre entre notre ERP et le CRM"  │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  │ MCP                                     │
│                                  │                                         │
│         ┌────────────────────────┼────────────────────────┐                │
│         │                        │                        │                │
│         ▼                        ▼                        ▼                │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐     │
│  │  MCP Server     │      │  MCP Server     │      │  MCP Server     │     │
│  │  "erp"          │      │  "crm"          │      │  "analytics"    │     │
│  └────────┬────────┘      └────────┬────────┘      └────────┬────────┘     │
│           │                        │                        │              │
│           ▼                        ▼                        ▼              │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐     │
│  │    MariaDB      │      │    MariaDB      │      │  ColumnStore    │     │
│  │    ERP DB       │      │    CRM DB       │      │  Analytics DW   │     │
│  └─────────────────┘      └─────────────────┘      └─────────────────┘     │
│                                                                            │
│  Claude peut :                                                             │
│  • Interroger chaque base séparément                                       │
│  • Combiner les résultats de plusieurs sources                             │
│  • Comparer les données entre systèmes                                     │
│  • Identifier les incohérences                                             │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### MCP + RAG Hybrid

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE MCP + RAG HYBRIDE                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  User Query: "Quels clients n'ont pas commandé depuis 3 mois et            │
│               correspondent au profil décrit dans notre politique de       │
│               réactivation ?"                                              │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                           Claude                                    │   │
│  │                                                                     │   │
│  │  1. Recherche RAG : Politique de réactivation → critères            │   │
│  │  2. Query MCP : Clients inactifs > 3 mois avec critères             │   │
│  │  3. Synthèse : Liste qualifiée + recommandations                    │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│              ┌───────────────────┴───────────────────┐                     │
│              │                                       │                     │
│              ▼                                       ▼                     │
│  ┌─────────────────────────┐           ┌─────────────────────────┐         │
│  │    RAG MCP Server       │           │   MariaDB MCP Server    │         │
│  │                         │           │                         │         │
│  │  Tools:                 │           │  Tools:                 │         │
│  │  • search_documents()   │           │  • query()              │         │
│  │  • get_context()        │           │  • describe_table()     │         │
│  │                         │           │                         │         │
│  └───────────┬─────────────┘           └───────────┬─────────────┘         │
│              │                                     │                       │
│              ▼                                     ▼                       │
│  ┌─────────────────────────┐           ┌─────────────────────────┐         │
│  │   Vector Store          │           │      MariaDB            │         │
│  │   (Embeddings docs)     │           │   (Données clients)     │         │
│  │                         │           │                         │         │
│  │  • Politiques           │           │  • customers            │         │
│  │  • Procédures           │           │  • orders               │         │
│  │  • Documentation        │           │  • interactions         │         │
│  └─────────────────────────┘           └─────────────────────────┘         │
│                                                                            │
│  Avantages :                                                               │
│  • Combiner données structurées (SQL) et non-structurées (docs)            │
│  • Réponses contextualisées par les politiques de l'entreprise             │
│  • Single point of access pour Claude                                      │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Agent IA autonome avec MCP

```python
#!/usr/bin/env python3
"""
autonomous_agent.py
Agent IA autonome utilisant MCP pour analyser les données
"""

import anthropic
from typing import List, Dict, Any
import json


class DataAnalysisAgent:
    """
    Agent autonome qui utilise Claude + MCP pour analyser des données
    et prendre des décisions.
    """
    
    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.conversation_history = []
    
    def analyze(self, objective: str, max_iterations: int = 10) -> str:
        """
        Exécute une analyse autonome avec un objectif donné.
        
        L'agent va:
        1. Planifier les étapes nécessaires
        2. Exécuter les requêtes via MCP
        3. Analyser les résultats
        4. Itérer jusqu'à atteindre l'objectif
        """
        
        system_prompt = """Tu es un analyste de données expert avec accès à une base de données MariaDB via MCP.

Tu dois atteindre l'objectif donné en:
1. Explorant d'abord le schéma de la base
2. Formulant des hypothèses
3. Exécutant des requêtes pour valider/invalider tes hypothèses
4. Synthétisant tes découvertes

À chaque étape, explique ton raisonnement avant d'agir.
Quand tu as terminé ton analyse, commence ta réponse finale par "ANALYSE TERMINÉE:".
"""
        
        self.conversation_history = [
            {"role": "user", "content": f"Objectif: {objective}\n\nCommence par explorer la base de données."}
        ]
        
        for iteration in range(max_iterations):
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                system=system_prompt,
                messages=self.conversation_history
            )
            
            assistant_message = response.content[0].text
            self.conversation_history.append({
                "role": "assistant",
                "content": assistant_message
            })
            
            # Vérifier si l'analyse est terminée
            if "ANALYSE TERMINÉE:" in assistant_message:
                return assistant_message
            
            # Vérifier si Claude veut utiliser un tool
            if response.stop_reason == "tool_use":
                # Les tools MCP sont gérés par Claude Desktop
                # Ici on simule la continuation
                self.conversation_history.append({
                    "role": "user",
                    "content": "Continue ton analyse avec les résultats obtenus."
                })
        
        return "Analyse non terminée après le nombre maximum d'itérations."
    
    def generate_report(self, topic: str) -> str:
        """Génère un rapport automatique sur un sujet"""
        
        prompt = f"""Génère un rapport complet sur: {topic}

Utilise les tools MCP disponibles pour:
1. Identifier les tables pertinentes (list_tables)
2. Comprendre leur structure (describe_table)
3. Extraire les données nécessaires (query)
4. Calculer les métriques clés

Format du rapport:
# Rapport: {topic}
## Résumé exécutif
## Données analysées
## Métriques clés
## Visualisations (décrites)
## Conclusions
## Recommandations
"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=8192,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return response.content[0].text


# Exemple d'utilisation avec Claude Desktop (MCP activé)
if __name__ == "__main__":
    # Note: En pratique, Claude Desktop gère la connexion MCP
    # Ce code montre la logique d'un agent autonome
    
    agent = DataAnalysisAgent(api_key="your-api-key")
    
    # Analyse autonome
    result = agent.analyze(
        objective="Identifier les produits avec le meilleur ratio marge/volume "
                  "et suggérer une stratégie de pricing"
    )
    print(result)
```

---

## Bonnes pratiques de sécurité

### Checklist de sécurité

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CHECKLIST SÉCURITÉ MCP + MARIADB                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ☐ AUTHENTIFICATION                                                         │
│    ☐ Utilisateur MariaDB dédié pour MCP                                     │
│    ☐ Mot de passe fort (32+ caractères)                                     │
│    ☐ Rotation régulière des credentials                                     │
│    ☐ Pas de credentials en clair dans le code                               │
│                                                                             │
│  ☐ AUTORISATION                                                             │
│    ☐ Permissions SELECT uniquement (read-only)                              │
│    ☐ Whitelist explicite des tables autorisées                              │
│    ☐ Blacklist des tables système (mysql.*, etc.)                           │
│    ☐ Pas d'accès aux procédures stockées                                    │
│                                                                             │
│  ☐ VALIDATION DES REQUÊTES                                                  │
│    ☐ Parser SQL pour détecter les injections                                │
│    ☐ Bloquer les mots-clés dangereux (DROP, DELETE, etc.)                   │
│    ☐ Limiter la complexité (sous-requêtes, JOINs)                           │
│    ☐ Timeout sur les requêtes longues                                       │
│                                                                             │
│  ☐ RATE LIMITING                                                            │
│    ☐ Limite de requêtes par minute                                          │
│    ☐ Limite de connexions simultanées                                       │
│    ☐ Limite de données retournées (MAX_ROWS)                                │
│                                                                             │
│  ☐ AUDIT                                                                    │
│    ☐ Logger toutes les requêtes                                             │
│    ☐ Inclure: timestamp, user, query, duration, row_count                   │
│    ☐ Alertes sur patterns suspects                                          │
│    ☐ Rétention des logs appropriée                                          │
│                                                                             │
│  ☐ RÉSEAU                                                                   │
│    ☐ MCP Server sur localhost ou réseau privé                               │
│    ☐ Pas d'exposition directe sur Internet                                  │
│    ☐ TLS pour les connexions distantes                                      │
│    ☐ Firewall configuré                                                     │
│                                                                             │
│  ☐ DONNÉES SENSIBLES                                                        │
│    ☐ Identifier les colonnes PII                                            │
│    ☐ Masquer/exclure les données sensibles                                  │
│    ☐ Anonymisation si nécessaire                                            │
│    ☐ Conformité RGPD/HIPAA vérifiée                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Masquage des données sensibles

```python
# Extension pour masquer les données sensibles

class SensitiveDataMasker:
    """Masque les données sensibles dans les résultats"""
    
    # Patterns de colonnes sensibles
    SENSITIVE_PATTERNS = [
        r'password',
        r'pwd',
        r'secret',
        r'token',
        r'api_key',
        r'credit_card',
        r'card_number',
        r'ssn',
        r'social_security',
        r'email',  # Optionnel selon le use case
        r'phone',
    ]
    
    def __init__(self, mask_char: str = "*", visible_chars: int = 4):
        self.mask_char = mask_char
        self.visible_chars = visible_chars
        self.patterns = [re.compile(p, re.IGNORECASE) for p in self.SENSITIVE_PATTERNS]
    
    def is_sensitive_column(self, column_name: str) -> bool:
        """Vérifie si une colonne contient des données sensibles"""
        for pattern in self.patterns:
            if pattern.search(column_name):
                return True
        return False
    
    def mask_value(self, value: Any) -> str:
        """Masque une valeur sensible"""
        if value is None:
            return None
        
        str_value = str(value)
        if len(str_value) <= self.visible_chars:
            return self.mask_char * len(str_value)
        
        return str_value[:self.visible_chars] + self.mask_char * (len(str_value) - self.visible_chars)
    
    def mask_result(self, columns: List[str], rows: List[Dict]) -> List[Dict]:
        """Masque les colonnes sensibles dans un résultat"""
        sensitive_columns = [col for col in columns if self.is_sensitive_column(col)]
        
        if not sensitive_columns:
            return rows
        
        masked_rows = []
        for row in rows:
            masked_row = row.copy()
            for col in sensitive_columns:
                if col in masked_row:
                    masked_row[col] = self.mask_value(masked_row[col])
            masked_rows.append(masked_row)
        
        return masked_rows
```

---

## Étude de cas : Assistant Business Intelligence

```
┌────────────────────────────────────────────────────────────────────────────┐
│               ÉTUDE DE CAS : ASSISTANT BI AVEC MCP                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Contexte :                                                                │
│  • PME avec 50 employés, pas de data analyst dédié                         │
│  • Base MariaDB avec données ERP (ventes, stocks, clients)                 │
│  • Besoin : permettre aux managers d'interroger les données en français    │
│                                                                            │
│  Solution déployée :                                                       │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     Interface Utilisateur                           │   │
│  │                                                                     │   │
│  │  Manager: "Quel a été notre chiffre d'affaires par région ce        │   │
│  │            trimestre comparé au trimestre précédent ?"              │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Claude Desktop + MCP                             │   │
│  │                                                                     │   │
│  │  1. Comprend la question en français                                │   │
│  │  2. Explore le schéma (list_tables, describe_table)                 │   │
│  │  3. Génère la requête SQL appropriée                                │   │
│  │  4. Exécute via MCP (query)                                         │   │
│  │  5. Formate les résultats en tableau + analyse                      │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    MariaDB MCP Server                               │   │
│  │                                                                     │   │
│  │  Configuration :                                                    │   │
│  │  • read_only: true                                                  │   │
│  │  • max_rows: 10000                                                  │   │
│  │  • allowed_tables: [customers, orders, products, regions]           │   │
│  │  • blocked_tables: [users, payments_raw]                            │   │
│  │  • query_timeout: 30s                                               │   │
│  │                                                                     │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
│                                  ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        MariaDB                                      │   │
│  │                                                                     │   │
│  │  Tables :                                                           │   │
│  │  • orders (500K rows)                                               │   │
│  │  • customers (50K rows)                                             │   │
│  │  • products (5K rows)                                               │   │
│  │  • regions (20 rows)                                                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Exemple de dialogue :                                                     │
│  ═════════════════════                                                     │
│                                                                            │
│  Manager: "Quel a été notre CA par région ce trimestre vs le précédent ?"  │
│                                                                            │
│  Claude: "Je vais analyser les données de ventes par région.               │
│                                                                            │
│           [Utilise list_tables pour voir les tables disponibles]           │
│           [Utilise describe_table('orders') pour comprendre la structure]  │
│           [Exécute la requête SQL de comparaison]                          │
│                                                                            │
│           Voici l'analyse du chiffre d'affaires par région :               │
│                                                                            │
│           | Région     | Q4 2024   | Q3 2024   | Évolution |               │
│           |------------|-----------|-----------|-----------|               │
│           | Île-de-Fr. | 1,245,000 | 1,180,000 | +5.5%     |               │
│           | Rhône-Alpes| 890,000   | 920,000   | -3.3%     |               │
│           | PACA       | 756,000   | 710,000   | +6.5%     |               │
│           | ...        | ...       | ...       | ...       |               │
│                                                                            │
│           Points clés :                                                    │
│           • La région PACA montre la plus forte croissance (+6.5%)         │
│           • Attention : Rhône-Alpes en recul (-3.3%)                       │
│           • CA total : +3.2% vs trimestre précédent                        │
│                                                                            │
│           Souhaitez-vous approfondir une région en particulier ?"          │
│                                                                            │
│  Résultats observés :                                                      │
│  ═══════════════════                                                       │
│  • Temps de réponse moyen : 5-10 secondes                                  │
│  • Précision des requêtes : 95%+ (après feedback initial)                  │
│  • Adoption : 80% des managers utilisent l'outil quotidiennement           │
│  • ROI : Économie de 15h/semaine de travail data analyst                   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Points clés à retenir

- **MCP standardise** l'accès des LLM aux sources de données — plus de vendor lock-in
- **Trois primitives** : Tools (actions), Resources (données), Prompts (templates)
- **Sécurité multicouche** : utilisateur read-only, validation SQL, rate limiting, audit
- **Configuration Claude Desktop** simple via JSON avec variables d'environnement
- **Multi-database** possible en configurant plusieurs serveurs MCP
- **Combinaison MCP + RAG** pour répondre à des questions complexes nécessitant contexte documentaire + données SQL
- **Masquage des données sensibles** indispensable pour les colonnes PII
- **Logs d'audit** essentiels pour traçabilité et debugging

---

## 🔗 Ressources et références

- [📖 Model Context Protocol Specification](https://modelcontextprotocol.io/specification)
- [📖 MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [📖 Claude Desktop MCP Configuration](https://docs.anthropic.com/claude/docs/claude-desktop)
- [📖 MCP Servers Repository](https://github.com/modelcontextprotocol/servers)
- [📖 MariaDB Python Connector](https://mariadb.com/docs/connect/programming-languages/python/)
- [📖 SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

---


⏭️ [Intégrations frameworks IA (LangChain, LlamaIndex, etc.)](/20-cas-usage-architectures/11-integrations-frameworks-ia.md)

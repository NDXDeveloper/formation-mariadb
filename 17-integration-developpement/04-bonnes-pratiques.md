🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.4 Bonnes pratiques de développement

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Maîtrise d'un langage de programmation
> - Compréhension des connexions et pooling (Sections 17.1-17.2)
> - Bases de l'architecture logicielle
> - Connaissance des principes SOLID (recommandé)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Structurer** votre code d'accès aux données de manière maintenable
- **Implémenter** une gestion d'erreurs robuste et informative
- **Sécuriser** vos applications contre les vulnérabilités courantes
- **Gérer** correctement les transactions et la cohérence des données
- **Logger** efficacement pour faciliter le debugging et le monitoring
- **Tester** vos interactions avec la base de données
- **Optimiser** les performances sans sacrifier la lisibilité
- **Documenter** votre code pour faciliter la maintenance

---

## Introduction

Les **bonnes pratiques de développement** transforment du code fonctionnel en code **professionnel, maintenable et évolutif**. Dans le contexte des bases de données, ces pratiques sont d'autant plus critiques qu'elles impactent directement la **sécurité**, les **performances** et la **fiabilité** de votre application.

### 📊 Impact des bonnes pratiques

| Aspect | Sans bonnes pratiques | Avec bonnes pratiques |
|--------|----------------------|----------------------|
| **Maintenabilité** | Code spaghetti, difficile à modifier | Structure claire, facile à faire évoluer |
| **Sécurité** | Vulnérable (injections SQL, etc.) | Robuste, principe de moindre privilège |
| **Performance** | Requêtes non optimisées, N+1 | Requêtes efficaces, indexation |
| **Fiabilité** | Bugs en production, perte de données | Tests, transactions, gestion d'erreurs |
| **Collaboration** | Chacun son style, conflits | Standards communs, code review |
| **Debugging** | Logs absents ou inutiles | Logging structuré, traçabilité |

💡 **Principe fondamental** : "Il est plus facile de faire les choses correctement dès le début que de corriger plus tard."

---

## 1. Architecture et séparation des responsabilités

### 🏗️ Repository Pattern

Le **Repository Pattern** isole la logique d'accès aux données du reste de l'application.

**Principe** :
```
Controller/Service → Repository → Database
      ↓                  ↓            ↓
   Business Logic   Data Access   Storage
```

#### **Exemple Python (sans ORM)**

```python
# repositories/user_repository.py
from typing import Optional, List
from contextlib import contextmanager
import mysql.connector
from mysql.connector.pooling import MySQLConnectionPool

class UserRepository:
    """Repository pour la gestion des utilisateurs"""
    
    def __init__(self, pool: MySQLConnectionPool):
        self.pool = pool
    
    @contextmanager
    def _get_connection(self):
        """Context manager pour connexions poolées"""
        conn = self.pool.get_connection()
        try:
            yield conn
        finally:
            conn.close()
    
    def find_by_id(self, user_id: int) -> Optional[dict]:
        """Récupère un utilisateur par ID"""
        with self._get_connection() as conn:
            cursor = conn.cursor(dictionary=True)
            cursor.execute(
                "SELECT id, name, email, created_at FROM users WHERE id = %s",
                (user_id,)
            )
            return cursor.fetchone()
    
    def find_by_email(self, email: str) -> Optional[dict]:
        """Récupère un utilisateur par email"""
        with self._get_connection() as conn:
            cursor = conn.cursor(dictionary=True)
            cursor.execute(
                "SELECT id, name, email, created_at FROM users WHERE email = %s",
                (email,)
            )
            return cursor.fetchone()
    
    def find_all(self, limit: int = 100, offset: int = 0) -> List[dict]:
        """Récupère tous les utilisateurs (paginé)"""
        with self._get_connection() as conn:
            cursor = conn.cursor(dictionary=True)
            cursor.execute(
                """
                SELECT id, name, email, created_at 
                FROM users 
                ORDER BY id 
                LIMIT %s OFFSET %s
                """,
                (limit, offset)
            )
            return cursor.fetchall()
    
    def create(self, name: str, email: str) -> int:
        """Crée un utilisateur et retourne son ID"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO users (name, email) VALUES (%s, %s)",
                (name, email)
            )
            conn.commit()
            return cursor.lastrowid
    
    def update(self, user_id: int, name: str = None, email: str = None) -> bool:
        """Met à jour un utilisateur"""
        updates = []
        params = []
        
        if name is not None:
            updates.append("name = %s")
            params.append(name)
        
        if email is not None:
            updates.append("email = %s")
            params.append(email)
        
        if not updates:
            return False
        
        params.append(user_id)
        
        with self._get_connection() as conn:
            cursor = conn.cursor()
            query = f"UPDATE users SET {', '.join(updates)} WHERE id = %s"
            cursor.execute(query, params)
            conn.commit()
            return cursor.rowcount > 0
    
    def delete(self, user_id: int) -> bool:
        """Supprime un utilisateur"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("DELETE FROM users WHERE id = %s", (user_id,))
            conn.commit()
            return cursor.rowcount > 0

# Utilisation dans un service
class UserService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo
    
    def get_user_profile(self, user_id: int) -> dict:
        """Business logic : récupérer profil utilisateur"""
        user = self.user_repo.find_by_id(user_id)
        if not user:
            raise ValueError(f"User {user_id} not found")
        
        # Business logic additionnelle
        user['display_name'] = user['name'].title()
        
        return user
```

#### **Exemple Java (JDBC)**

```java
// repositories/UserRepository.java
package com.example.repositories;

import com.example.models.User;
import javax.sql.DataSource;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class UserRepository {
    private final DataSource dataSource;
    
    public UserRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    public Optional<User> findById(Long id) {
        String sql = "SELECT id, name, email, created_at FROM users WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setLong(1, id);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapToUser(rs));
                }
            }
        } catch (SQLException e) {
            throw new RepositoryException("Error finding user by id", e);
        }
        
        return Optional.empty();
    }
    
    public List<User> findAll(int limit, int offset) {
        String sql = "SELECT id, name, email, created_at FROM users " +
                     "ORDER BY id LIMIT ? OFFSET ?";
        List<User> users = new ArrayList<>();
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setInt(1, limit);
            stmt.setInt(2, offset);
            
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    users.add(mapToUser(rs));
                }
            }
        } catch (SQLException e) {
            throw new RepositoryException("Error finding all users", e);
        }
        
        return users;
    }
    
    public Long create(User user) {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql, 
                 Statement.RETURN_GENERATED_KEYS)) {
            
            stmt.setString(1, user.getName());
            stmt.setString(2, user.getEmail());
            stmt.executeUpdate();
            
            try (ResultSet generatedKeys = stmt.getGeneratedKeys()) {
                if (generatedKeys.next()) {
                    return generatedKeys.getLong(1);
                }
            }
        } catch (SQLException e) {
            throw new RepositoryException("Error creating user", e);
        }
        
        throw new RepositoryException("Creating user failed, no ID obtained");
    }
    
    public boolean update(User user) {
        String sql = "UPDATE users SET name = ?, email = ? WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, user.getName());
            stmt.setString(2, user.getEmail());
            stmt.setLong(3, user.getId());
            
            return stmt.executeUpdate() > 0;
        } catch (SQLException e) {
            throw new RepositoryException("Error updating user", e);
        }
    }
    
    public boolean delete(Long id) {
        String sql = "DELETE FROM users WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setLong(1, id);
            return stmt.executeUpdate() > 0;
        } catch (SQLException e) {
            throw new RepositoryException("Error deleting user", e);
        }
    }
    
    private User mapToUser(ResultSet rs) throws SQLException {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setEmail(rs.getString("email"));
        user.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
        return user;
    }
}
```

#### **Exemple Node.js (TypeScript)**

```typescript
// repositories/UserRepository.ts
import { Pool, RowDataPacket, ResultSetHeader } from 'mysql2/promise';

export interface User {
    id: number;
    name: string;
    email: string;
    createdAt: Date;
}

export class UserRepository {
    constructor(private pool: Pool) {}
    
    async findById(id: number): Promise<User | null> {
        const [rows] = await this.pool.query<RowDataPacket[]>(
            'SELECT id, name, email, created_at FROM users WHERE id = ?',
            [id]
        );
        
        return rows.length > 0 ? this.mapToUser(rows[0]) : null;
    }
    
    async findByEmail(email: string): Promise<User | null> {
        const [rows] = await this.pool.query<RowDataPacket[]>(
            'SELECT id, name, email, created_at FROM users WHERE email = ?',
            [email]
        );
        
        return rows.length > 0 ? this.mapToUser(rows[0]) : null;
    }
    
    async findAll(limit: number = 100, offset: number = 0): Promise<User[]> {
        const [rows] = await this.pool.query<RowDataPacket[]>(
            'SELECT id, name, email, created_at FROM users ORDER BY id LIMIT ? OFFSET ?',
            [limit, offset]
        );
        
        return rows.map(row => this.mapToUser(row));
    }
    
    async create(name: string, email: string): Promise<number> {
        const [result] = await this.pool.query<ResultSetHeader>(
            'INSERT INTO users (name, email) VALUES (?, ?)',
            [name, email]
        );
        
        return result.insertId;
    }
    
    async update(id: number, updates: Partial<Pick<User, 'name' | 'email'>>): Promise<boolean> {
        const fields: string[] = [];
        const values: any[] = [];
        
        if (updates.name !== undefined) {
            fields.push('name = ?');
            values.push(updates.name);
        }
        
        if (updates.email !== undefined) {
            fields.push('email = ?');
            values.push(updates.email);
        }
        
        if (fields.length === 0) {
            return false;
        }
        
        values.push(id);
        
        const [result] = await this.pool.query<ResultSetHeader>(
            `UPDATE users SET ${fields.join(', ')} WHERE id = ?`,
            values
        );
        
        return result.affectedRows > 0;
    }
    
    async delete(id: number): Promise<boolean> {
        const [result] = await this.pool.query<ResultSetHeader>(
            'DELETE FROM users WHERE id = ?',
            [id]
        );
        
        return result.affectedRows > 0;
    }
    
    private mapToUser(row: RowDataPacket): User {
        return {
            id: row.id,
            name: row.name,
            email: row.email,
            createdAt: new Date(row.created_at)
        };
    }
}

// Service utilisant le repository
export class UserService {
    constructor(private userRepo: UserRepository) {}
    
    async getUserProfile(userId: number): Promise<User> {
        const user = await this.userRepo.findById(userId);
        
        if (!user) {
            throw new Error(`User ${userId} not found`);
        }
        
        return user;
    }
    
    async isEmailTaken(email: string): Promise<boolean> {
        const user = await this.userRepo.findByEmail(email);
        return user !== null;
    }
}
```

#### **Exemple .NET (C#)**

```csharp
// Repositories/UserRepository.cs
using System;
using System.Collections.Generic;
using System.Data;
using System.Threading.Tasks;
using MySqlConnector;

namespace MyApp.Repositories
{
    public interface IUserRepository
    {
        Task<User?> FindByIdAsync(int id);
        Task<User?> FindByEmailAsync(string email);
        Task<IEnumerable<User>> FindAllAsync(int limit = 100, int offset = 0);
        Task<int> CreateAsync(User user);
        Task<bool> UpdateAsync(User user);
        Task<bool> DeleteAsync(int id);
    }
    
    public class UserRepository : IUserRepository
    {
        private readonly string _connectionString;
        
        public UserRepository(string connectionString)
        {
            _connectionString = connectionString;
        }
        
        public async Task<User?> FindByIdAsync(int id)
        {
            using var connection = new MySqlConnection(_connectionString);
            await connection.OpenAsync();
            
            using var command = new MySqlCommand(
                "SELECT id, name, email, created_at FROM users WHERE id = @id",
                connection
            );
            command.Parameters.AddWithValue("@id", id);
            
            using var reader = await command.ExecuteReaderAsync();
            if (await reader.ReadAsync())
            {
                return MapToUser(reader);
            }
            
            return null;
        }
        
        public async Task<User?> FindByEmailAsync(string email)
        {
            using var connection = new MySqlConnection(_connectionString);
            await connection.OpenAsync();
            
            using var command = new MySqlCommand(
                "SELECT id, name, email, created_at FROM users WHERE email = @email",
                connection
            );
            command.Parameters.AddWithValue("@email", email);
            
            using var reader = await command.ExecuteReaderAsync();
            if (await reader.ReadAsync())
            {
                return MapToUser(reader);
            }
            
            return null;
        }
        
        public async Task<IEnumerable<User>> FindAllAsync(int limit = 100, int offset = 0)
        {
            var users = new List<User>();
            
            using var connection = new MySqlConnection(_connectionString);
            await connection.OpenAsync();
            
            using var command = new MySqlCommand(
                "SELECT id, name, email, created_at FROM users ORDER BY id LIMIT @limit OFFSET @offset",
                connection
            );
            command.Parameters.AddWithValue("@limit", limit);
            command.Parameters.AddWithValue("@offset", offset);
            
            using var reader = await command.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                users.Add(MapToUser(reader));
            }
            
            return users;
        }
        
        public async Task<int> CreateAsync(User user)
        {
            using var connection = new MySqlConnection(_connectionString);
            await connection.OpenAsync();
            
            using var command = new MySqlCommand(
                "INSERT INTO users (name, email) VALUES (@name, @email)",
                connection
            );
            command.Parameters.AddWithValue("@name", user.Name);
            command.Parameters.AddWithValue("@email", user.Email);
            
            await command.ExecuteNonQueryAsync();
            return (int)command.LastInsertedId;
        }
        
        public async Task<bool> UpdateAsync(User user)
        {
            using var connection = new MySqlConnection(_connectionString);
            await connection.OpenAsync();
            
            using var command = new MySqlCommand(
                "UPDATE users SET name = @name, email = @email WHERE id = @id",
                connection
            );
            command.Parameters.AddWithValue("@name", user.Name);
            command.Parameters.AddWithValue("@email", user.Email);
            command.Parameters.AddWithValue("@id", user.Id);
            
            var rowsAffected = await command.ExecuteNonQueryAsync();
            return rowsAffected > 0;
        }
        
        public async Task<bool> DeleteAsync(int id)
        {
            using var connection = new MySqlConnection(_connectionString);
            await connection.OpenAsync();
            
            using var command = new MySqlCommand(
                "DELETE FROM users WHERE id = @id",
                connection
            );
            command.Parameters.AddWithValue("@id", id);
            
            var rowsAffected = await command.ExecuteNonQueryAsync();
            return rowsAffected > 0;
        }
        
        private User MapToUser(MySqlDataReader reader)
        {
            return new User
            {
                Id = reader.GetInt32("id"),
                Name = reader.GetString("name"),
                Email = reader.GetString("email"),
                CreatedAt = reader.GetDateTime("created_at")
            };
        }
    }
}
```

✅ **Avantages du Repository Pattern** :
- Centralisation de la logique d'accès aux données
- Facilite les tests (mock du repository)
- Changement de base de données facilité
- Code métier découplé de la persistance

---

## 2. Gestion d'erreurs robuste

### 🛡️ Hiérarchie d'exceptions personnalisées

**Principe** : Créer des exceptions spécifiques pour distinguer les types d'erreurs.

```python
# exceptions.py
class DatabaseError(Exception):
    """Erreur de base pour toutes les erreurs DB"""
    pass

class ConnectionError(DatabaseError):
    """Erreur de connexion à la base"""
    pass

class QueryError(DatabaseError):
    """Erreur d'exécution de requête"""
    pass

class NotFoundError(DatabaseError):
    """Ressource non trouvée"""
    def __init__(self, resource_type: str, resource_id):
        super().__init__(f"{resource_type} {resource_id} not found")
        self.resource_type = resource_type
        self.resource_id = resource_id

class DuplicateError(DatabaseError):
    """Violation de contrainte d'unicité"""
    def __init__(self, field: str, value):
        super().__init__(f"Duplicate value '{value}' for field '{field}'")
        self.field = field
        self.value = value
```

**Utilisation** :
```python
import mysql.connector
from mysql.connector import Error as MySQLError

def create_user(name: str, email: str) -> int:
    try:
        with get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO users (name, email) VALUES (%s, %s)",
                (name, email)
            )
            conn.commit()
            return cursor.lastrowid
            
    except mysql.connector.errors.IntegrityError as e:
        # Erreur 1062 = Duplicate entry
        if e.errno == 1062:
            raise DuplicateError('email', email) from e
        raise QueryError(f"Integrity constraint violation: {e}") from e
    
    except mysql.connector.errors.DatabaseError as e:
        raise QueryError(f"Database error: {e}") from e
    
    except Exception as e:
        raise DatabaseError(f"Unexpected error: {e}") from e
```

### 📝 Logging structuré

```python
import logging
import json
from datetime import datetime

# Configuration du logger
logging.basicConfig(
    level=logging.INFO,
    format='%(message)s'
)
logger = logging.getLogger(__name__)

class StructuredLogger:
    """Logger avec sortie JSON structurée"""
    
    @staticmethod
    def log_query(query: str, params: tuple, duration_ms: float, rows_affected: int = None):
        """Log une requête SQL avec métriques"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'event': 'database_query',
            'query': query,
            'params': params,
            'duration_ms': duration_ms,
            'rows_affected': rows_affected
        }
        logger.info(json.dumps(log_entry))
    
    @staticmethod
    def log_error(error: Exception, context: dict = None):
        """Log une erreur avec contexte"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'event': 'database_error',
            'error_type': type(error).__name__,
            'error_message': str(error),
            'context': context or {}
        }
        logger.error(json.dumps(log_entry))

# Utilisation
import time

def find_user_by_id(user_id: int) -> dict:
    query = "SELECT * FROM users WHERE id = %s"
    start = time.time()
    
    try:
        with get_connection() as conn:
            cursor = conn.cursor(dictionary=True)
            cursor.execute(query, (user_id,))
            result = cursor.fetchone()
            
            duration_ms = (time.time() - start) * 1000
            StructuredLogger.log_query(query, (user_id,), duration_ms)
            
            if not result:
                raise NotFoundError('User', user_id)
            
            return result
            
    except Exception as e:
        StructuredLogger.log_error(e, {'user_id': user_id, 'query': query})
        raise
```

---

## 3. Configuration externalisée

### ⚙️ Variables d'environnement et secrets

**Principe** : Ne **jamais** hardcoder les credentials.

#### **Python avec python-dotenv**

```python
# config.py
import os
from dotenv import load_dotenv

# Charger variables .env
load_dotenv()

class Config:
    """Configuration centralisée"""
    
    # Database
    DB_HOST = os.getenv('DB_HOST', 'localhost')
    DB_PORT = int(os.getenv('DB_PORT', '3306'))
    DB_NAME = os.getenv('DB_NAME', 'mydb')
    DB_USER = os.getenv('DB_USER', 'root')
    DB_PASSWORD = os.getenv('DB_PASSWORD', '')
    
    # Pool
    DB_POOL_SIZE = int(os.getenv('DB_POOL_SIZE', '10'))
    DB_POOL_MAX_OVERFLOW = int(os.getenv('DB_POOL_MAX_OVERFLOW', '20'))
    
    # Application
    ENV = os.getenv('ENV', 'development')
    DEBUG = os.getenv('DEBUG', 'False').lower() == 'true'
    
    @classmethod
    def get_db_config(cls) -> dict:
        """Retourne la config DB pour mysql.connector"""
        return {
            'host': cls.DB_HOST,
            'port': cls.DB_PORT,
            'database': cls.DB_NAME,
            'user': cls.DB_USER,
            'password': cls.DB_PASSWORD,
            'charset': 'utf8mb4',
            'use_unicode': True,
        }
    
    @classmethod
    def validate(cls):
        """Valide la configuration au démarrage"""
        required = ['DB_HOST', 'DB_NAME', 'DB_USER', 'DB_PASSWORD']
        missing = [key for key in required if not getattr(cls, key)]
        
        if missing:
            raise ValueError(f"Missing required config: {', '.join(missing)}")
```

```bash
# .env (JAMAIS commité dans Git !)
DB_HOST=localhost
DB_PORT=3306
DB_NAME=myapp_production
DB_USER=app_user
DB_PASSWORD=super_secret_password_here

DB_POOL_SIZE=20
DB_POOL_MAX_OVERFLOW=40

ENV=production
DEBUG=false
```

```python
# .gitignore
.env
.env.*
*.env
```

#### **Node.js avec dotenv**

```typescript
// config/database.ts
import dotenv from 'dotenv';

dotenv.config();

export interface DatabaseConfig {
    host: string;
    port: number;
    database: string;
    user: string;
    password: string;
    connectionLimit: number;
    charset: string;
}

export const databaseConfig: DatabaseConfig = {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '3306'),
    database: process.env.DB_NAME!,
    user: process.env.DB_USER!,
    password: process.env.DB_PASSWORD!,
    connectionLimit: parseInt(process.env.DB_POOL_SIZE || '10'),
    charset: 'utf8mb4'
};

// Validation au démarrage
export function validateConfig(): void {
    const required = ['DB_NAME', 'DB_USER', 'DB_PASSWORD'];
    const missing = required.filter(key => !process.env[key]);
    
    if (missing.length > 0) {
        throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
    }
}
```

#### **Java avec Spring Boot**

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mariadb://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    driver-class-name: org.mariadb.jdbc.Driver
    hikari:
      minimum-idle: ${DB_POOL_MIN:10}
      maximum-pool-size: ${DB_POOL_MAX:20}
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

```java
// Configuration bean
@Configuration
public class DatabaseConfig {
    
    @Value("${spring.datasource.url}")
    private String url;
    
    @Value("${spring.datasource.username}")
    private String username;
    
    @Value("${spring.datasource.password}")
    private String password;
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        config.setPassword(password);
        config.setDriverClassName("org.mariadb.jdbc.Driver");
        
        return new HikariDataSource(config);
    }
}
```

#### **.NET avec appsettings.json**

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=${DB_HOST};Database=${DB_NAME};User=${DB_USER};Password=${DB_PASSWORD};CharSet=utf8mb4"
  },
  "Database": {
    "PoolSize": 20,
    "ConnectionTimeout": 30
  }
}
```

```csharp
// Startup.cs ou Program.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        var connectionString = Configuration.GetConnectionString("DefaultConnection");
        
        // Remplacer les variables d'environnement
        connectionString = connectionString
            .Replace("${DB_HOST}", Environment.GetEnvironmentVariable("DB_HOST") ?? "localhost")
            .Replace("${DB_NAME}", Environment.GetEnvironmentVariable("DB_NAME"))
            .Replace("${DB_USER}", Environment.GetEnvironmentVariable("DB_USER"))
            .Replace("${DB_PASSWORD}", Environment.GetEnvironmentVariable("DB_PASSWORD"));
        
        services.AddScoped<IUserRepository>(provider => 
            new UserRepository(connectionString));
    }
}
```

---

## 4. Transactions et cohérence des données

### 🔄 Gestion appropriée des transactions

**Principe ACID** : Les transactions garantissent l'atomicité, la cohérence, l'isolation et la durabilité.

#### **Pattern de transaction (Python)**

```python
from contextlib import contextmanager
from typing import Generator
import mysql.connector

@contextmanager
def transaction(conn: mysql.connector.MySQLConnection) -> Generator:
    """Context manager pour transactions"""
    conn.start_transaction()
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise

# Utilisation
def transfer_money(from_account_id: int, to_account_id: int, amount: float):
    """Transfert d'argent entre comptes (atomique)"""
    if amount <= 0:
        raise ValueError("Amount must be positive")
    
    with get_connection() as conn:
        with transaction(conn):
            cursor = conn.cursor()
            
            # Vérifier solde suffisant
            cursor.execute(
                "SELECT balance FROM accounts WHERE id = %s FOR UPDATE",
                (from_account_id,)
            )
            from_balance = cursor.fetchone()[0]
            
            if from_balance < amount:
                raise ValueError("Insufficient balance")
            
            # Débiter
            cursor.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_account_id)
            )
            
            # Créditer
            cursor.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_account_id)
            )
            
            # Enregistrer l'opération
            cursor.execute(
                """
                INSERT INTO transactions (from_account_id, to_account_id, amount, type)
                VALUES (%s, %s, %s, 'transfer')
                """,
                (from_account_id, to_account_id, amount)
            )
            
            # Commit automatique si pas d'exception
```

#### **Transactions avec Java**

```java
public class AccountService {
    private final DataSource dataSource;
    
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) 
            throws InsufficientBalanceException, SQLException {
        
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            
            try {
                // Vérifier solde
                BigDecimal fromBalance = getBalanceForUpdate(conn, fromAccountId);
                if (fromBalance.compareTo(amount) < 0) {
                    throw new InsufficientBalanceException("Insufficient balance");
                }
                
                // Débiter
                updateBalance(conn, fromAccountId, amount.negate());
                
                // Créditer
                updateBalance(conn, toAccountId, amount);
                
                // Logger transaction
                logTransaction(conn, fromAccountId, toAccountId, amount);
                
                conn.commit();
            } catch (Exception e) {
                conn.rollback();
                throw e;
            }
        }
    }
    
    private BigDecimal getBalanceForUpdate(Connection conn, Long accountId) 
            throws SQLException {
        String sql = "SELECT balance FROM accounts WHERE id = ? FOR UPDATE";
        
        try (PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setLong(1, accountId);
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return rs.getBigDecimal("balance");
                }
                throw new AccountNotFoundException(accountId);
            }
        }
    }
    
    private void updateBalance(Connection conn, Long accountId, BigDecimal delta) 
            throws SQLException {
        String sql = "UPDATE accounts SET balance = balance + ? WHERE id = ?";
        
        try (PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setBigDecimal(1, delta);
            stmt.setLong(2, accountId);
            stmt.executeUpdate();
        }
    }
}
```

#### **Transactions avec Node.js**

```typescript
async function transferMoney(
    fromAccountId: number,
    toAccountId: number,
    amount: number
): Promise<void> {
    if (amount <= 0) {
        throw new Error('Amount must be positive');
    }
    
    const connection = await pool.getConnection();
    
    try {
        await connection.beginTransaction();
        
        // Vérifier solde avec verrou
        const [rows] = await connection.query<RowDataPacket[]>(
            'SELECT balance FROM accounts WHERE id = ? FOR UPDATE',
            [fromAccountId]
        );
        
        if (rows.length === 0) {
            throw new Error('Account not found');
        }
        
        const fromBalance = rows[0].balance;
        if (fromBalance < amount) {
            throw new Error('Insufficient balance');
        }
        
        // Débiter
        await connection.query(
            'UPDATE accounts SET balance = balance - ? WHERE id = ?',
            [amount, fromAccountId]
        );
        
        // Créditer
        await connection.query(
            'UPDATE accounts SET balance = balance + ? WHERE id = ?',
            [amount, toAccountId]
        );
        
        // Logger
        await connection.query(
            'INSERT INTO transactions (from_account_id, to_account_id, amount, type) VALUES (?, ?, ?, ?)',
            [fromAccountId, toAccountId, amount, 'transfer']
        );
        
        await connection.commit();
    } catch (error) {
        await connection.rollback();
        throw error;
    } finally {
        connection.release();
    }
}
```

---

## 5. Tests de bases de données

### 🧪 Stratégies de test

#### **Test avec base de données de test (Python + pytest)**

```python
# conftest.py
import pytest
import mysql.connector
from mysql.connector.pooling import MySQLConnectionPool

@pytest.fixture(scope='session')
def db_pool():
    """Pool de connexions pour tests"""
    pool = MySQLConnectionPool(
        pool_name='test_pool',
        pool_size=5,
        host='localhost',
        database='test_db',
        user='test_user',
        password='test_password',
        charset='utf8mb4'
    )
    
    yield pool
    
    # Cleanup
    pool._remove_connections()

@pytest.fixture(scope='function')
def db_connection(db_pool):
    """Connexion avec rollback automatique"""
    conn = db_pool.get_connection()
    conn.start_transaction()
    
    yield conn
    
    conn.rollback()
    conn.close()

@pytest.fixture(scope='function', autouse=True)
def clean_database(db_connection):
    """Nettoie la base avant chaque test"""
    cursor = db_connection.cursor()
    cursor.execute("DELETE FROM users")
    cursor.execute("DELETE FROM posts")
    db_connection.commit()

# test_user_repository.py
def test_create_user(db_connection):
    """Test création utilisateur"""
    repo = UserRepository(db_connection)
    
    user_id = repo.create('Alice', 'alice@example.com')
    
    assert user_id > 0
    
    user = repo.find_by_id(user_id)
    assert user['name'] == 'Alice'
    assert user['email'] == 'alice@example.com'

def test_find_by_email(db_connection):
    """Test recherche par email"""
    repo = UserRepository(db_connection)
    
    repo.create('Bob', 'bob@example.com')
    
    user = repo.find_by_email('bob@example.com')
    assert user is not None
    assert user['name'] == 'Bob'
    
    not_found = repo.find_by_email('nonexistent@example.com')
    assert not_found is None

def test_update_user(db_connection):
    """Test mise à jour"""
    repo = UserRepository(db_connection)
    
    user_id = repo.create('Charlie', 'charlie@example.com')
    
    success = repo.update(user_id, name='Charles')
    assert success is True
    
    user = repo.find_by_id(user_id)
    assert user['name'] == 'Charles'
    assert user['email'] == 'charlie@example.com'  # Inchangé
```

#### **Test avec mocking (Python)**

```python
from unittest.mock import Mock, patch
import pytest

def test_user_service_get_profile():
    """Test du service avec mock du repository"""
    # Mock du repository
    mock_repo = Mock(spec=UserRepository)
    mock_repo.find_by_id.return_value = {
        'id': 1,
        'name': 'alice',
        'email': 'alice@example.com'
    }
    
    service = UserService(mock_repo)
    
    # Appel
    profile = service.get_user_profile(1)
    
    # Assertions
    assert profile['display_name'] == 'Alice'  # Business logic
    mock_repo.find_by_id.assert_called_once_with(1)

def test_user_service_user_not_found():
    """Test gestion utilisateur introuvable"""
    mock_repo = Mock(spec=UserRepository)
    mock_repo.find_by_id.return_value = None
    
    service = UserService(mock_repo)
    
    with pytest.raises(ValueError, match="User 999 not found"):
        service.get_user_profile(999)
```

---

## 6. Sécurité : Aperçu

### 🔒 Principes de sécurité

**Note** : La sécurité sera détaillée en **Section 17.8** (Injections SQL) et **17.9** (Prepared Statements).

#### **Règles d'or**

1. ✅ **TOUJOURS** utiliser des prepared statements
2. ✅ **JAMAIS** de concaténation de SQL avec input utilisateur
3. ✅ **Valider** toutes les entrées utilisateur
4. ✅ **Principe du moindre privilège** (utilisateurs DB)
5. ✅ **Chiffrer** les communications (SSL/TLS)

#### **Exemple de validation (TypeScript)**

```typescript
import Joi from 'joi';

// Schéma de validation
const createUserSchema = Joi.object({
    name: Joi.string().min(2).max(100).required(),
    email: Joi.string().email().required(),
    age: Joi.number().integer().min(0).max(150).optional()
});

async function createUser(input: any): Promise<number> {
    // Validation AVANT accès DB
    const { error, value } = createUserSchema.validate(input);
    
    if (error) {
        throw new ValidationError(error.message);
    }
    
    // Utiliser prepared statement (détails en 17.9)
    const [result] = await pool.query<ResultSetHeader>(
        'INSERT INTO users (name, email, age) VALUES (?, ?, ?)',
        [value.name, value.email, value.age]
    );
    
    return result.insertId;
}
```

---

## 7. Performance et optimisation

### ⚡ Bonnes pratiques de performance

#### **1. Sélectionner uniquement les colonnes nécessaires**

```python
# ❌ MAUVAIS : SELECT *
def get_users_bad():
    cursor.execute("SELECT * FROM users")
    return cursor.fetchall()

# ✅ BON : Colonnes explicites
def get_users_good():
    cursor.execute("SELECT id, name, email FROM users")
    return cursor.fetchall()
```

#### **2. Utiliser LIMIT et OFFSET pour la pagination**

```typescript
// ✅ BON : Toujours paginer
async function getUsers(page: number = 1, pageSize: number = 20): Promise<User[]> {
    const offset = (page - 1) * pageSize;
    
    const [rows] = await pool.query<RowDataPacket[]>(
        'SELECT id, name, email FROM users ORDER BY id LIMIT ? OFFSET ?',
        [pageSize, offset]
    );
    
    return rows.map(row => mapToUser(row));
}
```

#### **3. Utiliser les index appropriés**

```python
# Vérifier qu'un index existe pour les colonnes filtrées
def check_indexes():
    cursor.execute("""
        SELECT DISTINCT INDEX_NAME, COLUMN_NAME
        FROM information_schema.STATISTICS
        WHERE TABLE_SCHEMA = DATABASE()
        AND TABLE_NAME = 'users'
        ORDER BY INDEX_NAME, SEQ_IN_INDEX
    """)
    
    for index_name, column in cursor.fetchall():
        print(f"Index {index_name} sur colonne {column}")
```

#### **4. Batch operations pour insertions multiples**

```java
// ✅ BON : Batch insert
public void createUsersBatch(List<User> users) throws SQLException {
    String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
    
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement(sql)) {
        
        conn.setAutoCommit(false);
        
        for (User user : users) {
            stmt.setString(1, user.getName());
            stmt.setString(2, user.getEmail());
            stmt.addBatch();
            
            // Exécuter par lots de 1000
            if (users.indexOf(user) % 1000 == 0) {
                stmt.executeBatch();
            }
        }
        
        stmt.executeBatch();  // Dernier lot
        conn.commit();
    }
}
```

#### **5. Utiliser EXPLAIN pour analyser les requêtes**

```python
def analyze_query(query: str, params: tuple):
    """Analyse une requête avec EXPLAIN"""
    cursor.execute(f"EXPLAIN {query}", params)
    
    print("Query Plan:")
    for row in cursor.fetchall():
        print(row)
    
    # Vérifier les warnings
    cursor.execute("SHOW WARNINGS")
    warnings = cursor.fetchall()
    if warnings:
        print("Warnings:")
        for warning in warnings:
            print(warning)
```

---

## 8. Documentation et maintenabilité

### 📚 Documentation du code

```python
def find_user_by_email(email: str) -> Optional[dict]:
    """
    Récupère un utilisateur par son adresse email.
    
    Args:
        email (str): Adresse email de l'utilisateur (doit être valide)
    
    Returns:
        Optional[dict]: Dictionnaire avec les données utilisateur si trouvé,
                       None sinon. Structure:
                       {
                           'id': int,
                           'name': str,
                           'email': str,
                           'created_at': datetime
                       }
    
    Raises:
        ValueError: Si l'email est invalide
        DatabaseError: En cas d'erreur de connexion ou requête
    
    Example:
        >>> user = find_user_by_email('alice@example.com')
        >>> print(user['name'])
        Alice
    
    Note:
        Cette fonction utilise un prepared statement pour prévenir
        les injections SQL.
    """
    if not email or '@' not in email:
        raise ValueError("Invalid email format")
    
    with get_connection() as conn:
        cursor = conn.cursor(dictionary=True)
        cursor.execute(
            "SELECT id, name, email, created_at FROM users WHERE email = %s",
            (email,)
        )
        return cursor.fetchone()
```

### 📝 README pour le repository

```markdown
# User Repository

## Description
Gestion des utilisateurs dans la base de données MariaDB.

## Méthodes

### `find_by_id(user_id: int) -> Optional[dict]`
Récupère un utilisateur par son ID.

**Paramètres:**
- `user_id`: ID de l'utilisateur

**Retourne:**
- Dictionnaire utilisateur ou None

**Exceptions:**
- `DatabaseError`: Erreur de connexion/requête

### `create(name: str, email: str) -> int`
Crée un nouvel utilisateur.

**Paramètres:**
- `name`: Nom complet
- `email`: Adresse email (unique)

**Retourne:**
- ID du nouvel utilisateur

**Exceptions:**
- `DuplicateError`: Email déjà existant
- `DatabaseError`: Erreur de création

## Exemples d'utilisation

```python
# Créer un repository
repo = UserRepository(connection_pool)

# Récupérer un utilisateur
user = repo.find_by_id(1)
print(user['name'])

# Créer un utilisateur
user_id = repo.create('Alice', 'alice@example.com')
```

## Configuration requise
- MariaDB 11.8+
- Python 3.9+
- mysql-connector-python

## Tests
```bash
pytest tests/test_user_repository.py -v
```
```

---

## ✅ Checklist des bonnes pratiques

### 🏗️ **Architecture**
- [ ] Utiliser le Repository Pattern ou équivalent
- [ ] Séparer la logique métier de l'accès données
- [ ] Créer des interfaces/abstractions pour faciliter les tests
- [ ] Centraliser la configuration de connexion

### 🛡️ **Sécurité**
- [ ] Toujours utiliser des prepared statements
- [ ] Valider toutes les entrées utilisateur
- [ ] Ne jamais exposer les détails techniques dans les erreurs
- [ ] Utiliser SSL/TLS en production
- [ ] Appliquer le principe du moindre privilège

### 🔄 **Transactions**
- [ ] Utiliser des transactions pour opérations multiples
- [ ] Implémenter un rollback automatique en cas d'erreur
- [ ] Utiliser des verrous (FOR UPDATE) si nécessaire
- [ ] Définir le niveau d'isolation approprié

### 📝 **Logging**
- [ ] Logger toutes les erreurs avec contexte
- [ ] Utiliser un format structuré (JSON)
- [ ] Ne jamais logger les mots de passe
- [ ] Configurer différents niveaux (DEBUG, INFO, ERROR)

### ⚡ **Performance**
- [ ] Sélectionner uniquement les colonnes nécessaires
- [ ] Toujours paginer les résultats (LIMIT/OFFSET)
- [ ] Utiliser les index appropriés
- [ ] Batch les insertions/updates multiples
- [ ] Analyser les requêtes lentes avec EXPLAIN

### 🧪 **Tests**
- [ ] Tester chaque méthode du repository
- [ ] Utiliser une base de test dédiée
- [ ] Implémenter des rollbacks automatiques dans les tests
- [ ] Tester les cas d'erreur
- [ ] Mesurer la couverture de code

### 📚 **Documentation**
- [ ] Documenter toutes les méthodes publiques
- [ ] Créer un README pour chaque module
- [ ] Documenter les exceptions possibles
- [ ] Fournir des exemples d'utilisation

### ⚙️ **Configuration**
- [ ] Externaliser toute la configuration
- [ ] Ne jamais commiter les secrets
- [ ] Utiliser des variables d'environnement
- [ ] Valider la configuration au démarrage

---

## ✅ Points clés à retenir

- 🏗️ **Architecture** : Repository Pattern pour séparer logique métier et accès données
- 🛡️ **Gestion d'erreurs** : Exceptions spécifiques, logging structuré, contexte
- ⚙️ **Configuration** : Externalisation via variables d'environnement, validation
- 🔄 **Transactions** : Context managers, rollback automatique, atomicité
- 🧪 **Tests** : Base de test dédiée, rollbacks auto, mocking quand approprié
- 🔒 **Sécurité** : Prepared statements TOUJOURS, validation entrées (détails 17.8-17.9)
- ⚡ **Performance** : SELECT explicite, pagination, batch operations, EXPLAIN
- 📚 **Documentation** : Docstrings, README, exemples d'utilisation
- 💡 **Principe DRY** : Ne pas répéter le code, centraliser la logique
- 🎯 **SOLID** : Single Responsibility, dépendances injectées

---

## 🔗 Ressources et références

### **Patterns et Architecture**
- 📖 [Repository Pattern - Martin Fowler](https://martinfowler.com/eaaCatalog/repository.html)
- 📖 [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- 📝 [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)

### **Tests**
- 📖 [pytest Documentation](https://docs.pytest.org/)
- 📖 [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- 📖 [Jest Testing Framework](https://jestjs.io/docs/getting-started)

### **Logging**
- 📖 [Python logging](https://docs.python.org/3/library/logging.html)
- 📖 [SLF4J (Java)](http://www.slf4j.org/)
- 📖 [Winston (Node.js)](https://github.com/winstonjs/winston)

### **Configuration**
- 📖 [The Twelve-Factor App - Config](https://12factor.net/config)
- 📖 [dotenv](https://github.com/motdotla/dotenv)

### **Sécurité**
- 🔒 [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- 🔒 [SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

---

## ➡️ Sections suivantes

- **17.5** - Gestion des migrations de schéma : Versioning, rollback, automatisation
- **17.6** - Tests de bases de données : Stratégies avancées, fixtures, mocking
- **17.7** - Environnements de développement : Docker, seeding, reproductibilité
- **17.8** - Prévention des injections SQL : Comprendre, identifier, mitiger
- **17.9** - Prepared Statements : Fonctionnement, implémentation, performances

---

**MariaDB** : Version 11.8 LTS

⏭️ [Gestion des migrations de schéma](/17-integration-developpement/05-gestion-migrations-schema.md)

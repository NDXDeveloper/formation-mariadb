🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.6 Tests de bases de données

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Maîtrise des tests unitaires et d'intégration
> - Compréhension des connexions et transactions (Sections 17.1-17.2)
> - Connaissance d'un framework de test (pytest, JUnit, Jest, xUnit)
> - Architecture avec Repository Pattern (Section 17.4)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Distinguer** les différents types de tests impliquant une base de données
- **Configurer** une base de données de test isolée et reproductible
- **Implémenter** des fixtures et des seeders pour les données de test
- **Utiliser** les transactions pour isoler les tests
- **Mocker** les interactions avec la base quand approprié
- **Mesurer** la couverture de code incluant les repositories
- **Optimiser** la vitesse d'exécution des tests
- **Intégrer** les tests dans un pipeline CI/CD

---

## Introduction

Tester du code qui interagit avec une base de données est **l'un des défis majeurs** du développement logiciel. Les tests doivent être :

- ✅ **Fiables** : Résultats reproductibles
- ✅ **Rapides** : Feedback instantané
- ✅ **Isolés** : Pas de dépendances entre tests
- ✅ **Maintenables** : Faciles à comprendre et modifier

### 🎭 Le dilemme du test de base de données

**Option 1 : Tests avec vraie base de données**
- ✅ Teste le comportement réel
- ✅ Détecte les bugs SQL
- ❌ Plus lent
- ❌ Nécessite une base de test

**Option 2 : Mocking complet**
- ✅ Très rapide
- ✅ Pas de dépendance DB
- ❌ Ne teste pas le SQL réel
- ❌ Peut masquer des bugs

**💡 Solution : Approche hybride**
```
Tests unitaires (90%) : Mock des repositories
Tests d'intégration (10%) : Vraie base de données de test
```

---

## Types de tests

### 🔬 Taxonomie des tests

```
┌─────────────────────────────────────────────┐
│         Pyramide des tests                  │
│                                             │
│              ▲                              │
│             ╱ ╲  E2E Tests                  │
│            ╱   ╲ (Lents, peu nombreux)      │
│           ╱─────╲                           │
│          ╱       ╲ Integration Tests        │
│         ╱         ╲ (Moyens, modérés)       │
│        ╱───────────╲                        │
│       ╱             ╲ Unit Tests            │
│      ╱               ╲ (Rapides, nombreux)  │
│     ╱─────────────────╲                     │
│                                             │
└─────────────────────────────────────────────┘
```

#### **1. Tests unitaires (sans DB réelle)**

**Objectif** : Tester la logique métier isolément

```python
# test_user_service.py
from unittest.mock import Mock
import pytest

def test_get_user_profile():
    """Test du service avec repository mocké"""
    # Arrange
    mock_repo = Mock()
    mock_repo.find_by_id.return_value = {
        'id': 1,
        'name': 'alice',
        'email': 'alice@example.com'
    }
    
    service = UserService(mock_repo)
    
    # Act
    profile = service.get_user_profile(1)
    
    # Assert
    assert profile['display_name'] == 'Alice'  # Business logic
    mock_repo.find_by_id.assert_called_once_with(1)

def test_user_not_found():
    """Test gestion utilisateur introuvable"""
    mock_repo = Mock()
    mock_repo.find_by_id.return_value = None
    
    service = UserService(mock_repo)
    
    with pytest.raises(UserNotFoundError):
        service.get_user_profile(999)
```

#### **2. Tests d'intégration (avec DB de test)**

**Objectif** : Tester l'interaction réelle avec MariaDB

```python
# test_user_repository_integration.py
import pytest

def test_create_and_find_user(db_connection):
    """Test création et lecture d'un utilisateur (DB réelle)"""
    repo = UserRepository(db_connection)
    
    # Create
    user_id = repo.create('Bob', 'bob@example.com')
    assert user_id > 0
    
    # Read
    user = repo.find_by_id(user_id)
    assert user['name'] == 'Bob'
    assert user['email'] == 'bob@example.com'

def test_unique_email_constraint(db_connection):
    """Test contrainte d'unicité sur email"""
    repo = UserRepository(db_connection)
    
    repo.create('Alice', 'alice@example.com')
    
    # Dupliquer l'email doit échouer
    with pytest.raises(DuplicateError):
        repo.create('Alicia', 'alice@example.com')
```

#### **3. Tests end-to-end (E2E)**

**Objectif** : Tester le flux complet (API → Service → Repository → DB)

```python
# test_api_e2e.py
import pytest
from fastapi.testclient import TestClient

def test_create_user_api(client: TestClient, db):
    """Test API complète de création utilisateur"""
    response = client.post('/users', json={
        'name': 'Charlie',
        'email': 'charlie@example.com'
    })
    
    assert response.status_code == 201
    data = response.json()
    assert data['id'] > 0
    assert data['name'] == 'Charlie'
    
    # Vérifier en DB
    user = db.query("SELECT * FROM users WHERE id = ?", data['id'])
    assert user['email'] == 'charlie@example.com'
```

---

## Configuration de l'environnement de test

### 🗄️ Base de données de test dédiée

**Principe** : **JAMAIS** tester sur la base de développement ou production !

```
Development DB : myapp_dev
Test DB        : myapp_test
Production DB  : myapp_prod
```

#### **Option 1 : Base locale permanente**

```bash
# Créer base de test
mysql -u root -p -e "CREATE DATABASE myapp_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Créer utilisateur de test
mysql -u root -p -e "CREATE USER 'test_user'@'localhost' IDENTIFIED BY 'test_password';"
mysql -u root -p -e "GRANT ALL PRIVILEGES ON myapp_test.* TO 'test_user'@'localhost';"
```

#### **Option 2 : Docker pour isolation complète**

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  mariadb-test:
    image: mariadb:11.8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: test_db
      MYSQL_USER: test_user
      MYSQL_PASSWORD: test_password
    ports:
      - "3307:3306"  # Port différent pour ne pas conflit avec DB dev
    tmpfs:
      - /var/lib/mysql  # En RAM pour performance max
```

```bash
# Démarrer pour tests
docker-compose -f docker-compose.test.yml up -d

# Tests
pytest

# Arrêter
docker-compose -f docker-compose.test.yml down
```

#### **Option 3 : Base en mémoire (SQLite pour tests rapides)**

⚠️ **Attention** : SQLite != MariaDB (dialecte SQL différent)

```python
# Uniquement pour tests très simples
# Pas recommandé pour tester des fonctionnalités MariaDB spécifiques
```

---

## Fixtures et données de test

### 🎭 Fixtures avec pytest (Python)

**Structure** :
```
tests/
├── conftest.py           # Fixtures globales
├── fixtures/
│   └── data.sql          # Données de test
├── test_user_repo.py
└── test_post_repo.py
```

**conftest.py** :
```python
import pytest
import mysql.connector
from mysql.connector.pooling import MySQLConnectionPool

@pytest.fixture(scope='session')
def db_pool():
    """Pool de connexions pour toute la session de tests"""
    pool = MySQLConnectionPool(
        pool_name='test_pool',
        pool_size=5,
        host='localhost',
        port=3307,  # Docker test DB
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
    """Connexion avec rollback automatique (isolation des tests)"""
    conn = db_pool.get_connection()
    conn.start_transaction()
    
    yield conn
    
    # ROLLBACK automatique : aucun test ne modifie réellement la DB
    conn.rollback()
    conn.close()

@pytest.fixture(scope='function', autouse=True)
def setup_schema(db_pool):
    """Recréer le schéma avant chaque test"""
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # Drop et recréer tables
    cursor.execute("DROP TABLE IF EXISTS posts")
    cursor.execute("DROP TABLE IF EXISTS users")
    
    cursor.execute("""
        CREATE TABLE users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            email VARCHAR(255) UNIQUE NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    """)
    
    cursor.execute("""
        CREATE TABLE posts (
            id INT AUTO_INCREMENT PRIMARY KEY,
            user_id INT NOT NULL,
            title VARCHAR(255) NOT NULL,
            content TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    """)
    
    conn.commit()
    conn.close()

@pytest.fixture
def sample_users(db_connection):
    """Fixture de données : créer des utilisateurs de test"""
    cursor = db_connection.cursor()
    
    users = [
        ('Alice', 'alice@example.com'),
        ('Bob', 'bob@example.com'),
        ('Charlie', 'charlie@example.com')
    ]
    
    cursor.executemany(
        "INSERT INTO users (name, email) VALUES (%s, %s)",
        users
    )
    
    return users

# Utilisation dans les tests
def test_find_all_users(db_connection, sample_users):
    """Test avec données pré-créées"""
    repo = UserRepository(db_connection)
    
    users = repo.find_all()
    
    assert len(users) == 3
    assert users[0]['name'] == 'Alice'
```

### 🧪 Setup/Teardown avec JUnit (Java)

```java
// UserRepositoryTest.java
import org.junit.jupiter.api.*;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.*;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class UserRepositoryTest {
    
    private HikariDataSource dataSource;
    private Connection connection;
    
    @BeforeAll
    void setupDataSource() {
        // Configuration du pool de test
        dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:mariadb://localhost:3307/test_db");
        dataSource.setUsername("test_user");
        dataSource.setPassword("test_password");
        dataSource.setMaximumPoolSize(5);
    }
    
    @BeforeEach
    void setupDatabase() throws SQLException {
        connection = dataSource.getConnection();
        connection.setAutoCommit(false);
        
        // Recréer schéma
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("DROP TABLE IF EXISTS posts");
            stmt.execute("DROP TABLE IF EXISTS users");
            
            stmt.execute("""
                CREATE TABLE users (
                    id BIGINT AUTO_INCREMENT PRIMARY KEY,
                    name VARCHAR(100) NOT NULL,
                    email VARCHAR(255) UNIQUE NOT NULL,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            """);
        }
    }
    
    @AfterEach
    void cleanupDatabase() throws SQLException {
        // Rollback pour isolation
        connection.rollback();
        connection.close();
    }
    
    @AfterAll
    void shutdownDataSource() {
        dataSource.close();
    }
    
    @Test
    void testCreateUser() throws SQLException {
        UserRepository repo = new UserRepository(dataSource);
        
        Long userId = repo.create(new User("Alice", "alice@example.com"));
        
        Assertions.assertTrue(userId > 0);
        
        User user = repo.findById(userId).orElseThrow();
        Assertions.assertEquals("Alice", user.getName());
        Assertions.assertEquals("alice@example.com", user.getEmail());
    }
    
    @Test
    void testDuplicateEmail() {
        UserRepository repo = new UserRepository(dataSource);
        
        repo.create(new User("Alice", "alice@example.com"));
        
        Assertions.assertThrows(DuplicateEmailException.class, () -> {
            repo.create(new User("Alicia", "alice@example.com"));
        });
    }
}
```

### 🟢 Hooks avec Jest (Node.js/TypeScript)

```typescript
// user.repository.test.ts
import { Pool } from 'mysql2/promise';
import mysql from 'mysql2/promise';
import { UserRepository } from '../repositories/UserRepository';

describe('UserRepository', () => {
    let pool: Pool;
    let repo: UserRepository;
    
    // Setup global : une fois pour tous les tests
    beforeAll(async () => {
        pool = mysql.createPool({
            host: 'localhost',
            port: 3307,
            user: 'test_user',
            password: 'test_password',
            database: 'test_db',
            charset: 'utf8mb4',
            connectionLimit: 5
        });
        
        repo = new UserRepository(pool);
    });
    
    // Setup avant chaque test
    beforeEach(async () => {
        const conn = await pool.getConnection();
        
        try {
            // Recréer schéma
            await conn.query('DROP TABLE IF EXISTS posts');
            await conn.query('DROP TABLE IF EXISTS users');
            
            await conn.query(`
                CREATE TABLE users (
                    id INT AUTO_INCREMENT PRIMARY KEY,
                    name VARCHAR(100) NOT NULL,
                    email VARCHAR(255) UNIQUE NOT NULL,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            `);
        } finally {
            conn.release();
        }
    });
    
    // Cleanup après tous les tests
    afterAll(async () => {
        await pool.end();
    });
    
    test('should create a user', async () => {
        const userId = await repo.create('Alice', 'alice@example.com');
        
        expect(userId).toBeGreaterThan(0);
        
        const user = await repo.findById(userId);
        expect(user).not.toBeNull();
        expect(user!.name).toBe('Alice');
        expect(user!.email).toBe('alice@example.com');
    });
    
    test('should throw on duplicate email', async () => {
        await repo.create('Alice', 'alice@example.com');
        
        await expect(
            repo.create('Alicia', 'alice@example.com')
        ).rejects.toThrow('Duplicate entry');
    });
    
    test('should find user by email', async () => {
        await repo.create('Bob', 'bob@example.com');
        
        const user = await repo.findByEmail('bob@example.com');
        
        expect(user).not.toBeNull();
        expect(user!.name).toBe('Bob');
    });
});
```

### 🔷 Fixtures avec xUnit (.NET)

```csharp
// UserRepositoryTests.cs
using System;
using System.Threading.Tasks;
using Xunit;
using MySqlConnector;

public class DatabaseFixture : IAsyncLifetime
{
    public string ConnectionString { get; private set; }
    
    public async Task InitializeAsync()
    {
        ConnectionString = "Server=localhost;Port=3307;Database=test_db;User=test_user;Password=test_password;CharSet=utf8mb4";
        
        // Recréer schéma
        using var connection = new MySqlConnection(ConnectionString);
        await connection.OpenAsync();
        
        using var command = connection.CreateCommand();
        command.CommandText = @"
            DROP TABLE IF EXISTS posts;
            DROP TABLE IF EXISTS users;
            
            CREATE TABLE users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        await command.ExecuteNonQueryAsync();
    }
    
    public Task DisposeAsync()
    {
        return Task.CompletedTask;
    }
}

[Collection("Database")]
public class UserRepositoryTests : IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;
    private MySqlConnection _connection;
    private MySqlTransaction _transaction;
    
    public UserRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }
    
    public async Task InitializeAsync()
    {
        _connection = new MySqlConnection(_fixture.ConnectionString);
        await _connection.OpenAsync();
        _transaction = await _connection.BeginTransactionAsync();
    }
    
    public async Task DisposeAsync()
    {
        // Rollback pour isolation
        await _transaction.RollbackAsync();
        await _connection.DisposeAsync();
    }
    
    [Fact]
    public async Task CreateUser_ShouldReturnValidId()
    {
        var repo = new UserRepository(_fixture.ConnectionString);
        
        var userId = await repo.CreateAsync(new User 
        { 
            Name = "Alice", 
            Email = "alice@example.com" 
        });
        
        Assert.True(userId > 0);
        
        var user = await repo.FindByIdAsync(userId);
        Assert.NotNull(user);
        Assert.Equal("Alice", user.Name);
    }
    
    [Fact]
    public async Task CreateUser_DuplicateEmail_ShouldThrow()
    {
        var repo = new UserRepository(_fixture.ConnectionString);
        
        await repo.CreateAsync(new User { Name = "Alice", Email = "alice@example.com" });
        
        await Assert.ThrowsAsync<MySqlException>(async () =>
            await repo.CreateAsync(new User { Name = "Alicia", Email = "alice@example.com" })
        );
    }
}

[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture>
{
    // Cette classe n'a pas de code, elle sert juste à définir la collection
}
```

---

## Stratégies d'isolation des tests

### 🔄 Transaction rollback (recommandé)

**Avantage** : Rapide, pas besoin de nettoyer la DB

```python
@pytest.fixture
def db_connection(db_pool):
    conn = db_pool.get_connection()
    conn.start_transaction()
    
    yield conn
    
    conn.rollback()  # Annule TOUS les changements
    conn.close()
```

✅ **Avantages** :
- Très rapide
- Isolation parfaite
- Pas de cleanup manuel

⚠️ **Limitations** :
- Ne teste pas le COMMIT
- Ne fonctionne pas avec DDL (CREATE TABLE, etc.)

### 🧹 Truncate tables

**Alternative** : Nettoyer les tables après chaque test

```python
@pytest.fixture(autouse=True)
def cleanup_database(db_connection):
    yield  # Test s'exécute
    
    # Cleanup après le test
    cursor = db_connection.cursor()
    cursor.execute("SET FOREIGN_KEY_CHECKS = 0")
    cursor.execute("TRUNCATE TABLE posts")
    cursor.execute("TRUNCATE TABLE users")
    cursor.execute("SET FOREIGN_KEY_CHECKS = 1")
    db_connection.commit()
```

✅ **Avantages** :
- Teste le COMMIT réel
- Fonctionne avec DDL

❌ **Inconvénients** :
- Plus lent
- Nécessite gestion des FK

### 🔄 Recréer le schéma

**Alternative** : DROP/CREATE avant chaque test

```typescript
beforeEach(async () => {
    await pool.query('DROP DATABASE IF EXISTS test_db');
    await pool.query('CREATE DATABASE test_db');
    await pool.query('USE test_db');
    
    // Créer schéma
    await pool.query(schemaSQL);
});
```

✅ **Avantages** :
- État complètement propre
- Teste les migrations

❌ **Inconvénients** :
- Très lent
- Recommandé uniquement si nécessaire

---

## Mocking vs vraie base de données

### 🎭 Quand mocker ?

**Tests unitaires de services** :
```python
# test_user_service.py (mock)
def test_register_user_success(mock_user_repo, mock_email_service):
    """Test logique métier sans DB"""
    mock_user_repo.find_by_email.return_value = None
    mock_user_repo.create.return_value = 123
    
    service = UserService(mock_user_repo, mock_email_service)
    user_id = service.register_user('Alice', 'alice@example.com')
    
    assert user_id == 123
    mock_email_service.send_welcome_email.assert_called_once()
```

✅ **Cas d'usage du mocking** :
- Tester la logique métier isolément
- Tests très rapides (millisecondes)
- Pas de dépendance DB

### 🗄️ Quand utiliser la vraie DB ?

**Tests d'intégration des repositories** :
```python
# test_user_repository.py (vraie DB)
def test_complex_query(db_connection):
    """Test requête SQL complexe"""
    repo = UserRepository(db_connection)
    
    # Créer données de test
    for i in range(10):
        repo.create(f'User{i}', f'user{i}@example.com')
    
    # Tester requête avec JOIN, GROUP BY, etc.
    stats = repo.get_user_statistics()
    
    assert stats['total_users'] == 10
```

✅ **Cas d'usage vraie DB** :
- Tester requêtes SQL complexes
- Tester contraintes (FK, UNIQUE)
- Tester types de données MariaDB spécifiques
- Tester transactions et niveaux d'isolation

---

## Patterns de test avancés

### 🏭 Factory pattern pour données de test

```python
# factories.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class UserFactory:
    """Factory pour créer des utilisateurs de test"""
    name: str = "Test User"
    email: str = "test@example.com"
    age: Optional[int] = None
    
    _counter = 0
    
    @classmethod
    def create(cls, **kwargs):
        """Créer un utilisateur avec données par défaut + overrides"""
        cls._counter += 1
        
        defaults = {
            'name': f'User{cls._counter}',
            'email': f'user{cls._counter}@example.com'
        }
        defaults.update(kwargs)
        
        return defaults
    
    @classmethod
    def create_batch(cls, count: int, **kwargs):
        """Créer plusieurs utilisateurs"""
        return [cls.create(**kwargs) for _ in range(count)]

# Utilisation dans tests
def test_pagination(db_connection):
    repo = UserRepository(db_connection)
    
    # Créer 50 utilisateurs rapidement
    users = UserFactory.create_batch(50)
    for user in users:
        repo.create(**user)
    
    # Tester pagination
    page1 = repo.find_all(limit=20, offset=0)
    page2 = repo.find_all(limit=20, offset=20)
    
    assert len(page1) == 20
    assert len(page2) == 20
    assert page1[0]['id'] != page2[0]['id']
```

### 📦 Builder pattern pour objets complexes

```typescript
// UserBuilder.ts
export class UserBuilder {
    private user: Partial<User> = {};
    
    withName(name: string): this {
        this.user.name = name;
        return this;
    }
    
    withEmail(email: string): this {
        this.user.email = email;
        return this;
    }
    
    withAge(age: number): this {
        this.user.age = age;
        return this;
    }
    
    build(): User {
        return {
            name: this.user.name || 'Test User',
            email: this.user.email || 'test@example.com',
            age: this.user.age
        } as User;
    }
}

// Utilisation
test('should create adult user', async () => {
    const user = new UserBuilder()
        .withName('Alice')
        .withAge(25)
        .build();
    
    const userId = await repo.create(user);
    
    const found = await repo.findById(userId);
    expect(found!.age).toBe(25);
});
```

### 🎯 Parameterized tests

**Python (pytest)** :
```python
@pytest.mark.parametrize('name,email,expected_valid', [
    ('Alice', 'alice@example.com', True),
    ('Bob', 'bob@example.com', True),
    ('', 'invalid@example.com', False),  # Nom vide
    ('Charlie', 'invalid-email', False),  # Email invalide
    ('A' * 101, 'long@example.com', False),  # Nom trop long
])
def test_user_validation(name, email, expected_valid):
    """Test validation avec différents cas"""
    try:
        user = User(name=name, email=email)
        user.validate()
        is_valid = True
    except ValidationError:
        is_valid = False
    
    assert is_valid == expected_valid
```

**Java (JUnit 5)** :
```java
@ParameterizedTest
@CsvSource({
    "Alice, alice@example.com, true",
    "Bob, bob@example.com, true",
    "'', invalid@example.com, false",
    "Charlie, invalid-email, false"
})
void testUserValidation(String name, String email, boolean expectedValid) {
    User user = new User(name, email);
    
    if (expectedValid) {
        assertDoesNotThrow(() -> user.validate());
    } else {
        assertThrows(ValidationException.class, () -> user.validate());
    }
}
```

---

## Performance et optimisation

### ⚡ Accélérer les tests

#### **1. Utiliser tmpfs (RAM) pour MariaDB**

```yaml
# docker-compose.test.yml
services:
  mariadb-test:
    image: mariadb:11.8
    tmpfs:
      - /var/lib/mysql  # DB en RAM = 10x plus rapide
```

#### **2. Réutiliser le schéma**

```python
# Slow : Recréer schéma à chaque test
@pytest.fixture(autouse=True)
def setup_schema():
    drop_all_tables()
    create_schema()  # 100ms par test !

# Fast : Schéma créé une fois, TRUNCATE entre tests
@pytest.fixture(scope='session', autouse=True)
def setup_schema_once():
    create_schema()  # 100ms une seule fois

@pytest.fixture(autouse=True)
def cleanup():
    yield
    truncate_all_tables()  # 5ms par test
```

#### **3. Paralléliser les tests**

```bash
# pytest avec xdist
pip install pytest-xdist
pytest -n 4  # 4 workers en parallèle

# Jest
npm test -- --maxWorkers=4
```

⚠️ **Attention** : Nécessite isolation parfaite (base par worker ou transactions).

#### **4. Minimiser les données de test**

```python
# ❌ Slow : Trop de données
def test_find_user(db):
    create_1000_users()  # Inutile !
    user = repo.find_by_id(1)
    assert user is not None

# ✅ Fast : Données minimales
def test_find_user(db):
    user_id = repo.create('Alice', 'alice@example.com')
    user = repo.find_by_id(user_id)
    assert user is not None
```

---

## Couverture de code

### 📊 Mesurer la couverture

**Python (coverage.py)** :
```bash
pip install coverage pytest-cov

# Exécuter tests avec couverture
pytest --cov=src --cov-report=html --cov-report=term

# Voir rapport
open htmlcov/index.html
```

**Java (JaCoCo)** :
```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```bash
mvn clean test
# Rapport : target/site/jacoco/index.html
```

**JavaScript/TypeScript (Jest)** :
```json
// package.json
{
  "scripts": {
    "test:coverage": "jest --coverage"
  },
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.{js,ts}",
      "!src/**/*.test.{js,ts}"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

### 🎯 Objectifs de couverture

| Composant | Couverture cible | Justification |
|-----------|-----------------|---------------|
| **Repositories** | 90-100% | Code critique, facile à tester |
| **Services** | 85-95% | Logique métier importante |
| **Controllers/Routes** | 70-85% | Tests E2E suffisent souvent |
| **Utilities** | 95-100% | Fonctions pures, tests simples |

💡 **Priorité** : Couvrir le code critique (sécurité, paiements, données sensibles) à 100%.

---

## Intégration CI/CD

### 🚀 GitHub Actions

```yaml
# .github/workflows/tests.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mariadb:
        image: mariadb:11.8
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: test_db
          MYSQL_USER: test_user
          MYSQL_PASSWORD: test_password
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Wait for MariaDB
        run: |
          while ! mysqladmin ping -h127.0.0.1 -P3306 -uroot -proot --silent; do
            echo "Waiting for MariaDB..."
            sleep 1
          done
      
      - name: Run migrations
        run: |
          python manage.py migrate
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_NAME: test_db
          DB_USER: test_user
          DB_PASSWORD: test_password
      
      - name: Run tests
        run: |
          pytest --cov=src --cov-report=xml --cov-report=term
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_NAME: test_db
          DB_USER: test_user
          DB_PASSWORD: test_password
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          fail_ci_if_error: true
```

### 🔵 GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test

test:
  stage: test
  image: python:3.11
  
  services:
    - name: mariadb:11.8
      alias: mariadb
      variables:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: test_db
        MYSQL_USER: test_user
        MYSQL_PASSWORD: test_password
  
  variables:
    DB_HOST: mariadb
    DB_PORT: "3306"
    DB_NAME: test_db
    DB_USER: test_user
    DB_PASSWORD: test_password
  
  before_script:
    - pip install -r requirements.txt
    - pip install pytest pytest-cov
    - apt-get update && apt-get install -y default-mysql-client
    - |
      until mysqladmin ping -hmariadb -uroot -proot; do
        echo "Waiting for MariaDB..."
        sleep 1
      done
  
  script:
    - pytest --cov=src --cov-report=term --cov-report=html
  
  coverage: '/TOTAL.*\s+(\d+%)$/'
  
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - htmlcov/
    expire_in: 1 week
```

---

## Bonnes pratiques

### ✅ Règles d'or

#### **1. AAA Pattern (Arrange-Act-Assert)**

```python
def test_create_user():
    # ARRANGE : Préparer les données
    repo = UserRepository(db_connection)
    name = 'Alice'
    email = 'alice@example.com'
    
    # ACT : Exécuter l'action
    user_id = repo.create(name, email)
    
    # ASSERT : Vérifier le résultat
    assert user_id > 0
    user = repo.find_by_id(user_id)
    assert user['name'] == name
```

#### **2. Un test = un concept**

```python
# ❌ MAUVAIS : Teste trop de choses
def test_user_crud():
    user_id = repo.create('Alice', 'alice@example.com')
    user = repo.find_by_id(user_id)
    repo.update(user_id, name='Alicia')
    repo.delete(user_id)
    # Difficile à débugger si ça échoue !

# ✅ BON : Tests séparés
def test_create_user():
    user_id = repo.create('Alice', 'alice@example.com')
    assert user_id > 0

def test_find_user_by_id():
    user_id = repo.create('Alice', 'alice@example.com')
    user = repo.find_by_id(user_id)
    assert user['name'] == 'Alice'

def test_update_user():
    user_id = repo.create('Alice', 'alice@example.com')
    repo.update(user_id, name='Alicia')
    user = repo.find_by_id(user_id)
    assert user['name'] == 'Alicia'
```

#### **3. Noms de test descriptifs**

```python
# ❌ MAUVAIS
def test_user_1():
    pass

# ✅ BON
def test_create_user_with_valid_data_should_return_id():
    pass

def test_create_user_with_duplicate_email_should_raise_error():
    pass

def test_find_user_by_nonexistent_id_should_return_none():
    pass
```

#### **4. Tests indépendants**

```python
# ❌ MAUVAIS : Tests dépendants
class TestUsers:
    user_id = None
    
    def test_1_create(self):
        self.user_id = repo.create('Alice', 'alice@example.com')
    
    def test_2_find(self):
        user = repo.find_by_id(self.user_id)  # Dépend de test_1 !
        assert user is not None

# ✅ BON : Tests indépendants
def test_create_user():
    user_id = repo.create('Alice', 'alice@example.com')
    assert user_id > 0

def test_find_user():
    # Créer ses propres données
    user_id = repo.create('Alice', 'alice@example.com')
    user = repo.find_by_id(user_id)
    assert user is not None
```

#### **5. Tester les cas limites**

```python
def test_edge_cases():
    # Nom minimum
    repo.create('A', 'a@example.com')
    
    # Nom maximum (100 chars)
    repo.create('A' * 100, 'long@example.com')
    
    # Unicode
    repo.create('François', 'françois@example.com')
    
    # Emojis (UTF8MB4)
    repo.create('Alice 😊', 'alice@example.com')
```

### ⚠️ Anti-patterns à éviter

#### **❌ Tests qui dépendent de l'ordre d'exécution**

```python
# MAUVAIS : Ordre d'exécution important
def test_a_create():
    global user_id
    user_id = repo.create('Alice', 'alice@example.com')

def test_b_update():
    repo.update(user_id, name='Alicia')  # Crash si test_a pas exécuté !
```

#### **❌ Tests qui modifient la base de développement**

```python
# MAUVAIS : Utilise DB de dev
def test_create():
    # Pollue votre base de développement !
    repo.create('Test User', 'test@example.com')
```

#### **❌ Tests sans assertions**

```python
# MAUVAIS : Pas d'assertion
def test_create():
    repo.create('Alice', 'alice@example.com')
    # Aucune vérification !

# BON
def test_create():
    user_id = repo.create('Alice', 'alice@example.com')
    assert user_id > 0
```

---

## ✅ Points clés à retenir

- 🧪 **Types de tests** : Unitaires (mocks), intégration (DB test), E2E (flux complet)
- 🗄️ **Base de test** : Dédiée, isolée, reproductible (Docker recommandé)
- 🎭 **Fixtures** : Données de test réutilisables, factories, builders
- 🔄 **Isolation** : Transaction rollback (rapide) ou TRUNCATE (complet)
- 🎯 **Mocking** : Pour logique métier, vraie DB pour requêtes SQL
- ⚡ **Performance** : tmpfs, schéma réutilisé, tests parallèles
- 📊 **Couverture** : 90%+ pour repositories, 85%+ pour services
- 🚀 **CI/CD** : Tests automatiques avec MariaDB en service
- ✅ **Bonnes pratiques** : AAA pattern, tests indépendants, noms descriptifs
- 🆕 **MariaDB 11.8** : Tester nouvelles fonctionnalités (JSON, Vector, UTF8MB4)

---

## 🔗 Ressources et références

### **Documentation des frameworks de test**
- 📖 [pytest](https://docs.pytest.org/)
- 📖 [JUnit 5](https://junit.org/junit5/docs/current/user-guide/)
- 📖 [Jest](https://jestjs.io/docs/getting-started)
- 📖 [xUnit (.NET)](https://xunit.net/)
- 📖 [Mocha](https://mochajs.org/)

### **Couverture de code**
- 📖 [coverage.py](https://coverage.readthedocs.io/)
- 📖 [JaCoCo](https://www.jacoco.org/jacoco/)
- 📖 [Istanbul/nyc](https://istanbul.js.org/)

### **Mocking**
- 📖 [unittest.mock (Python)](https://docs.python.org/3/library/unittest.mock.html)
- 📖 [Mockito (Java)](https://site.mockito.org/)
- 📖 [Jest Mocking](https://jestjs.io/docs/mock-functions)

### **Best practices**
- 📝 [Testing Best Practices](https://martinfowler.com/articles/practical-test-pyramid.html)
- 📝 [Database Testing Strategies](https://www.kenst.com/2015/03/testing-with-databases/)

---

## ➡️ Sections suivantes

- **17.7** - Environnements de développement : Docker, seeding, reproductibilité
- **17.8** - Prévention des injections SQL : Comprendre, détecter, corriger
- **17.9** - Prepared Statements : Fonctionnement, sécurité, performances

---

**MariaDB** : Version 11.8 LTS

⏭️ [Environnements de développement](/17-integration-developpement/07-environnements-developpement.md)

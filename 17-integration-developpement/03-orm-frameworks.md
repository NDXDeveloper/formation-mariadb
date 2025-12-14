🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3 ORM et Frameworks

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Maîtrise du SQL (Chapitres 2-4)
> - Programmation orientée objet (classes, héritage, composition)
> - Compréhension des connexions et du pooling (Sections 17.1-17.2)
> - Connaissance d'au moins un langage de programmation moderne

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** ce qu'est un ORM et comment il fonctionne
- **Évaluer** quand utiliser un ORM vs du SQL natif
- **Choisir** l'ORM approprié pour votre stack technologique
- **Configurer** les principaux ORM avec MariaDB
- **Mapper** vos modèles objets vers des tables relationnelles
- **Gérer** les relations (one-to-one, one-to-many, many-to-many)
- **Implémenter** des migrations de schéma avec les ORM
- **Optimiser** les requêtes générées par les ORM
- **Éviter** les pièges courants (N+1 queries, lazy loading, etc.)

---

## Introduction

Les **ORM (Object-Relational Mapping)** sont des bibliothèques qui permettent de manipuler les données de votre base relationnelle comme des **objets** dans votre langage de programmation. Ils éliminent le besoin d'écrire du SQL pour les opérations courantes (CRUD) et offrent une abstraction puissante.

### 🔄 Le problème de l'impedance mismatch

**Paradigme relationnel** (SQL) vs **Paradigme objet** (POO) :

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   Monde Relationnel     │         │      Monde Objet        │
│                         │         │                         │
│  • Tables               │         │  • Classes              │
│  • Lignes               │         │  • Objets/Instances     │
│  • Colonnes             │         │  • Attributs            │
│  • Clés étrangères      │  ←→     │  • Références           │
│  • Jointures            │         │  • Collections          │
│  • NULL                 │         │  • null/None/nil        │
│  • Types SQL            │         │  • Types natifs         │
│                         │         │                         │
└─────────────────────────┘         └─────────────────────────┘
           ▲                                    ▲
           │                                    │
           └────────────── ORM ─────────────────┘
              (Traduction bidirectionnelle)
```

**L'ORM comme traducteur** :
- **Code → SQL** : Traduit les opérations objets en requêtes SQL
- **SQL → Code** : Hydrate les résultats SQL en objets du langage
- **Abstraction** : Cache la complexité du mapping

### 💡 Exemple concret

**Sans ORM** (SQL manuel) :
```python
# Récupérer un utilisateur
cursor.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
row = cursor.fetchone()
user = {'id': row[0], 'name': row[1], 'email': row[2]}

# Récupérer ses posts
cursor.execute("SELECT id, title, content FROM posts WHERE user_id = %s", (user_id,))
posts = [{'id': r[0], 'title': r[1], 'content': r[2]} for r in cursor.fetchall()]
```

**Avec ORM** (SQLAlchemy) :
```python
# Récupérer un utilisateur avec ses posts (en une ligne !)
user = session.query(User).filter_by(id=user_id).first()
posts = user.posts  # Relation automatique
```

🎯 **Bénéfice** : Code plus lisible, maintenable, et type-safe.

---

## Pourquoi utiliser un ORM ?

### ✅ Avantages des ORM

#### **1. Productivité accrue**
```python
# CRUD en quelques lignes (SQLAlchemy)
user = User(name="Alice", email="alice@example.com")
session.add(user)        # INSERT
session.commit()

user.name = "Alicia"     # UPDATE
session.commit()

session.delete(user)     # DELETE
session.commit()
```

Sans ORM, chaque opération nécessite d'écrire du SQL manuel.

#### **2. Abstraction de la base de données**
```javascript
// Même code Sequelize pour MySQL, PostgreSQL, SQLite
const users = await User.findAll({
    where: { status: 'active' }
});
```

Changer de SGBD = changer la connection string (pas le code métier).

#### **3. Type safety et auto-complétion**
```csharp
// Entity Framework Core (.NET)
var users = await context.Users
    .Where(u => u.Age > 18)  // Typage fort, IntelliSense
    .OrderBy(u => u.Name)
    .ToListAsync();

// Erreur de compilation si propriété inexistante
var error = users.FirstOrDefault().NonExistentField;  // ❌ Erreur
```

#### **4. Gestion automatique des relations**
```java
// Hibernate - Récupération d'un utilisateur avec ses posts
User user = session.get(User.class, 1L);
List<Post> posts = user.getPosts();  // JOIN automatique
```

#### **5. Migrations de schéma intégrées**
```bash
# Entity Framework Core
dotnet ef migrations add AddUserTable
dotnet ef database update

# Django ORM
python manage.py makemigrations
python manage.py migrate
```

#### **6. Protection contre les injections SQL**
```python
# ORM : Requêtes paramétrées automatiquement
users = User.query.filter_by(email=user_input).all()
# Généré : SELECT * FROM users WHERE email = %s
```

Impossible d'injecter du SQL via `user_input`.

#### **7. Lazy loading et eager loading**
```ruby
# Active Record (Ruby on Rails)
user = User.find(1)
posts = user.posts  # Lazy loading : requête seulement si accédé

# Eager loading : une seule requête avec JOIN
user = User.includes(:posts).find(1)
```

### ❌ Inconvénients des ORM

#### **1. Performance**
```python
# N+1 Query Problem (piège classique)
users = User.query.all()  # 1 requête
for user in users:
    print(user.posts)  # N requêtes (une par user) !

# Solution : Eager loading
users = User.query.options(joinedload(User.posts)).all()  # 1 requête
```

#### **2. Requêtes complexes limitées**
```sql
-- SQL analytique complexe
SELECT 
    DATE_FORMAT(created_at, '%Y-%m') as month,
    COUNT(*) as total,
    AVG(price) OVER (ORDER BY created_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg
FROM orders
WHERE status = 'completed'
GROUP BY month
HAVING total > 100;
```

Impossible ou très complexe à exprimer avec un ORM → utiliser SQL natif.

#### **3. Courbe d'apprentissage**
- Maîtriser l'ORM **ET** SQL sous-jacent
- Debugging : comprendre les requêtes générées
- Performance tuning : savoir quand bypasser l'ORM

#### **4. Overhead mémoire**
```python
# Charger 100,000 objets en mémoire
users = User.query.all()  # Peut consommer plusieurs GB !

# Solution : Pagination ou streaming
users = User.query.yield_per(1000)  # Itération par batch
```

#### **5. Abstractions qui fuient**
```javascript
// Sequelize - Sous-requêtes complexes
const users = await User.findAll({
    attributes: [
        'id',
        [sequelize.literal('(SELECT COUNT(*) FROM posts WHERE posts.user_id = User.id)'), 'postCount']
    ]
});
```

On finit par écrire du SQL dans l'ORM...

---

## ORM vs SQL natif : La décision

### 🎯 Matrice de décision

| Cas d'usage | Recommandation | Justification |
|-------------|----------------|---------------|
| **CRUD simple** | ✅ ORM | Productivité maximale |
| **Relations 1-N, N-N** | ✅ ORM | Gestion automatique |
| **Requêtes complexes** | ❌ SQL natif | Performance, expressivité |
| **Analytics/Reporting** | ❌ SQL natif | Window functions, agrégations |
| **Bulk operations** | ❌ SQL natif | Performance (bulk insert) |
| **Migrations schéma** | ✅ ORM | Versioning, rollback |
| **Prototypage rapide** | ✅ ORM | Scaffolding, auto-génération |
| **Performance critique** | ❌ SQL natif | Contrôle total, optimisation |

### 💡 Approche hybride (recommandée)

**80% ORM + 20% SQL natif** :

```python
# ORM pour CRUD et relations
user = User.query.filter_by(email='alice@example.com').first()
user.posts.append(Post(title='Hello', content='World'))
session.commit()

# SQL natif pour analytics
result = session.execute(text("""
    SELECT DATE(created_at) as date, COUNT(*) as count
    FROM posts
    WHERE user_id = :user_id
    GROUP BY date
    ORDER BY date DESC
    LIMIT 30
"""), {'user_id': user.id})
stats = [dict(row) for row in result]
```

---

## Vue d'ensemble des principaux ORM

### 🏆 ORM par écosystème

| Écosystème | ORM principal | Alternatives | Popularité |
|------------|---------------|--------------|------------|
| **Java/JVM** | Hibernate | jOOQ, MyBatis, EclipseLink | ⭐⭐⭐⭐⭐ |
| **Python** | SQLAlchemy | Django ORM, Peewee, Tortoise ORM | ⭐⭐⭐⭐⭐ |
| **Node.js** | Sequelize | TypeORM, Prisma, MikroORM | ⭐⭐⭐⭐⭐ |
| **TypeScript** | Prisma | TypeORM, MikroORM, Drizzle | ⭐⭐⭐⭐⭐ |
| **.NET/C#** | Entity Framework Core | Dapper, NHibernate | ⭐⭐⭐⭐⭐ |
| **PHP** | Doctrine | Eloquent (Laravel), Propel | ⭐⭐⭐⭐ |
| **Ruby** | Active Record | Sequel | ⭐⭐⭐⭐⭐ |
| **Go** | GORM | ent, sqlx (query builder) | ⭐⭐⭐⭐ |

### 📊 Comparaison détaillée

| ORM | Langage | Type | Lazy Loading | Migrations | Type Safety | Performance |
|-----|---------|------|--------------|------------|-------------|-------------|
| **Hibernate** | Java | Full ORM | ✅ | ✅ | ✅ (fort) | ⭐⭐⭐⭐ |
| **SQLAlchemy** | Python | Full ORM + Core | ✅ | ✅ (Alembic) | ⚠️ (duck typing) | ⭐⭐⭐⭐⭐ |
| **Sequelize** | JavaScript | Full ORM | ✅ | ✅ | ❌ | ⭐⭐⭐ |
| **Prisma** | TypeScript | Schema-first | ❌ | ✅ | ✅ (excellent) | ⭐⭐⭐⭐⭐ |
| **EF Core** | C# | Full ORM | ✅ | ✅ | ✅ (fort) | ⭐⭐⭐⭐ |
| **Doctrine** | PHP | Full ORM | ✅ | ✅ | ⚠️ (annotations) | ⭐⭐⭐ |
| **Active Record** | Ruby | Active Record | ✅ | ✅ | ❌ | ⭐⭐⭐⭐ |
| **GORM** | Go | Full ORM | ✅ | ✅ | ✅ | ⭐⭐⭐⭐ |

---

## Concepts fondamentaux des ORM

### 🗺️ Mapping Objet-Relationnel

#### **1. Entités et Tables**

```python
# SQLAlchemy
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'  # Nom de la table
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, nullable=False)
```

SQL généré :
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

#### **2. Relations : One-to-Many**

```javascript
// Sequelize (Node.js)
const User = sequelize.define('User', {
    name: DataTypes.STRING,
    email: DataTypes.STRING
});

const Post = sequelize.define('Post', {
    title: DataTypes.STRING,
    content: DataTypes.TEXT
});

// Relation 1-N
User.hasMany(Post, { foreignKey: 'user_id' });
Post.belongsTo(User, { foreignKey: 'user_id' });
```

Usage :
```javascript
const user = await User.findByPk(1, {
    include: [Post]  // Eager loading
});
console.log(user.Posts);  // Array de posts
```

#### **3. Relations : Many-to-Many**

```csharp
// Entity Framework Core
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Course> Courses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }
    public ICollection<Student> Students { get; set; }
}

// Configuration dans DbContext
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity(j => j.ToTable("StudentCourses"));
```

SQL généré :
```sql
CREATE TABLE StudentCourses (
    StudentsId INT,
    CoursesId INT,
    PRIMARY KEY (StudentsId, CoursesId),
    FOREIGN KEY (StudentsId) REFERENCES Students(Id),
    FOREIGN KEY (CoursesId) REFERENCES Courses(Id)
);
```

### 🔄 Stratégies de chargement

#### **Lazy Loading** (chargement à la demande)

```java
// Hibernate
User user = session.get(User.class, 1L);
// Requête : SELECT * FROM users WHERE id = 1

List<Post> posts = user.getPosts();
// Requête : SELECT * FROM posts WHERE user_id = 1 (seulement maintenant)
```

⚠️ **Piège** : N+1 queries !

#### **Eager Loading** (chargement immédiat)

```python
# SQLAlchemy
from sqlalchemy.orm import joinedload

user = session.query(User).options(
    joinedload(User.posts)
).filter_by(id=1).first()

# Requête : SELECT * FROM users 
#           LEFT JOIN posts ON users.id = posts.user_id 
#           WHERE users.id = 1
```

✅ **Une seule requête** avec JOIN.

#### **Explicit Loading** (chargement explicite)

```csharp
// Entity Framework Core
var user = await context.Users.FindAsync(1);

// Charger explicitement les posts
await context.Entry(user)
    .Collection(u => u.Posts)
    .LoadAsync();
```

---

## Migrations de schéma

### 🔄 Workflow typique

```
1. Modifier le modèle (entité/classe)
2. Générer la migration (diff auto)
3. Réviser la migration SQL
4. Appliquer la migration
5. Version control (Git)
```

### 📝 Exemples par ORM

#### **Entity Framework Core (.NET)**

```bash
# Créer une migration
dotnet ef migrations add AddUserAgeColumn

# Voir le SQL généré
dotnet ef migrations script

# Appliquer
dotnet ef database update

# Rollback
dotnet ef database update PreviousMigration
```

```csharp
// Migration générée automatiquement
public partial class AddUserAgeColumn : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<int>(
            name: "Age",
            table: "Users",
            nullable: false,
            defaultValue: 0);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "Age",
            table: "Users");
    }
}
```

#### **SQLAlchemy + Alembic (Python)**

```bash
# Initialiser Alembic
alembic init alembic

# Générer migration auto
alembic revision --autogenerate -m "add user age column"

# Appliquer
alembic upgrade head

# Rollback
alembic downgrade -1
```

```python
# Migration générée (alembic/versions/xxx_add_user_age.py)
def upgrade():
    op.add_column('users', sa.Column('age', sa.Integer(), nullable=True))

def downgrade():
    op.drop_column('users', 'age')
```

#### **Sequelize (Node.js)**

```bash
# Créer migration vide
npx sequelize-cli migration:generate --name add-user-age

# Éditer la migration manuellement
# Appliquer
npx sequelize-cli db:migrate

# Rollback
npx sequelize-cli db:migrate:undo
```

```javascript
// migrations/xxx-add-user-age.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.addColumn('Users', 'age', {
      type: Sequelize.INTEGER,
      allowNull: true
    });
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.removeColumn('Users', 'age');
  }
};
```

#### **Prisma (TypeScript)**

```bash
# Modifier schema.prisma
# Créer migration
npx prisma migrate dev --name add-user-age

# Appliquer en production
npx prisma migrate deploy
```

```prisma
// schema.prisma
model User {
  id    Int    @id @default(autoincrement())
  name  String
  email String @unique
  age   Int?   // Nouvelle colonne
}
```

---

## Exemples rapides par ORM

### ☕ **Hibernate (Java)**

**Configuration** :
```xml
<!-- hibernate.cfg.xml -->
<hibernate-configuration>
    <session-factory>
        <property name="connection.driver_class">org.mariadb.jdbc.Driver</property>
        <property name="connection.url">jdbc:mariadb://localhost:3306/mydb</property>
        <property name="connection.username">user</property>
        <property name="connection.password">password</property>
        <property name="dialect">org.hibernate.dialect.MariaDB103Dialect</property>
        <property name="show_sql">true</property>
        <property name="hbm2ddl.auto">update</property>
    </session-factory>
</hibernate-configuration>
```

**Entité** :
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Post> posts = new ArrayList<>();
    
    // Getters, setters, constructors
}
```

**CRUD** :
```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

// Create
User user = new User("Alice", "alice@example.com");
session.save(user);

// Read
User found = session.get(User.class, 1L);

// Update
found.setName("Alicia");
session.update(found);

// Delete
session.delete(found);

tx.commit();
session.close();
```

---

### 🐍 **SQLAlchemy (Python)**

**Configuration** :
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    'mysql+mysqlconnector://user:password@localhost/mydb',
    echo=True,  # Log SQL
    pool_size=10,
    max_overflow=20
)

Session = sessionmaker(bind=engine)
session = Session()
```

**Modèles** :
```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    
    posts = relationship('Post', back_populates='user')

class Post(Base):
    __tablename__ = 'posts'
    
    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(Text)
    user_id = Column(Integer, ForeignKey('users.id'))
    
    user = relationship('User', back_populates='posts')
```

**CRUD** :
```python
# Create
user = User(name='Alice', email='alice@example.com')
session.add(user)
session.commit()

# Read
user = session.query(User).filter_by(email='alice@example.com').first()

# Update
user.name = 'Alicia'
session.commit()

# Delete
session.delete(user)
session.commit()

# Relations
post = Post(title='Hello', content='World', user=user)
session.add(post)
session.commit()

# Query avec relations
users_with_posts = session.query(User).join(Post).all()
```

---

### 🟢 **Sequelize (Node.js)**

**Configuration** :
```javascript
const { Sequelize, DataTypes } = require('sequelize');

const sequelize = new Sequelize('mydb', 'user', 'password', {
    host: 'localhost',
    dialect: 'mariadb',
    pool: {
        max: 10,
        min: 0,
        acquire: 30000,
        idle: 10000
    },
    logging: console.log  // Log SQL
});
```

**Modèles** :
```javascript
const User = sequelize.define('User', {
    name: {
        type: DataTypes.STRING(100),
        allowNull: false
    },
    email: {
        type: DataTypes.STRING(255),
        unique: true,
        allowNull: false
    }
});

const Post = sequelize.define('Post', {
    title: {
        type: DataTypes.STRING(200),
        allowNull: false
    },
    content: {
        type: DataTypes.TEXT
    }
});

// Relations
User.hasMany(Post);
Post.belongsTo(User);

// Sync (création tables)
await sequelize.sync({ alter: true });
```

**CRUD** :
```javascript
// Create
const user = await User.create({
    name: 'Alice',
    email: 'alice@example.com'
});

// Read
const found = await User.findOne({
    where: { email: 'alice@example.com' }
});

// Update
await user.update({ name: 'Alicia' });

// Delete
await user.destroy();

// Relations
const post = await Post.create({
    title: 'Hello',
    content: 'World',
    UserId: user.id
});

// Query avec relations
const usersWithPosts = await User.findAll({
    include: [Post]
});
```

---

### 🔷 **Prisma (TypeScript)**

**Schema** :
```prisma
// schema.prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id    Int     @id @default(autoincrement())
  name  String  @db.VarChar(100)
  email String  @unique @db.VarChar(255)
  posts Post[]
}

model Post {
  id      Int    @id @default(autoincrement())
  title   String @db.VarChar(200)
  content String @db.Text
  userId  Int
  user    User   @relation(fields: [userId], references: [id])
}
```

**Génération client** :
```bash
# .env
DATABASE_URL="mysql://user:password@localhost:3306/mydb"

# Générer client TypeScript
npx prisma generate
```

**CRUD** :
```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// Create
const user = await prisma.user.create({
    data: {
        name: 'Alice',
        email: 'alice@example.com'
    }
});

// Read
const found = await prisma.user.findUnique({
    where: { email: 'alice@example.com' }
});

// Update
const updated = await prisma.user.update({
    where: { id: user.id },
    data: { name: 'Alicia' }
});

// Delete
await prisma.user.delete({
    where: { id: user.id }
});

// Relations
const post = await prisma.post.create({
    data: {
        title: 'Hello',
        content: 'World',
        user: {
            connect: { id: user.id }
        }
    }
});

// Query avec relations (type-safe !)
const usersWithPosts = await prisma.user.findMany({
    include: {
        posts: true
    }
});
```

---

### 🔷 **Entity Framework Core (.NET)**

**Configuration** :
```csharp
public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Post> Posts { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseMySql(
            "Server=localhost;Database=mydb;User=user;Password=password",
            ServerVersion.AutoDetect("Server=localhost;Database=mydb;User=user;Password=password")
        );
    }
}
```

**Modèles** :
```csharp
public class User
{
    public int Id { get; set; }
    
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }
    
    [Required]
    [MaxLength(255)]
    public string Email { get; set; }
    
    public ICollection<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    
    [Required]
    [MaxLength(200)]
    public string Title { get; set; }
    
    public string Content { get; set; }
    
    public int UserId { get; set; }
    public User User { get; set; }
}
```

**CRUD** :
```csharp
using var context = new AppDbContext();

// Create
var user = new User { Name = "Alice", Email = "alice@example.com" };
context.Users.Add(user);
await context.SaveChangesAsync();

// Read
var found = await context.Users
    .FirstOrDefaultAsync(u => u.Email == "alice@example.com");

// Update
found.Name = "Alicia";
await context.SaveChangesAsync();

// Delete
context.Users.Remove(found);
await context.SaveChangesAsync();

// Relations
var post = new Post { Title = "Hello", Content = "World", UserId = user.Id };
context.Posts.Add(post);
await context.SaveChangesAsync();

// Query avec relations (LINQ)
var usersWithPosts = await context.Users
    .Include(u => u.Posts)
    .ToListAsync();
```

---

## 🆕 Compatibilité MariaDB 11.8

### **Support des nouveautés**

Tous les ORM modernes supportent MariaDB 11.8 :

#### **1. JSON amélioré**

```python
# SQLAlchemy avec type JSON
from sqlalchemy import JSON

class Product(Base):
    __tablename__ = 'products'
    id = Column(Integer, primary_key=True)
    metadata = Column(JSON)  # Support JSON natif

# Utilisation
product = Product(metadata={'color': 'red', 'size': 'L'})
session.add(product)
session.commit()

# Query JSON
products = session.query(Product).filter(
    Product.metadata['color'].astext == 'red'
).all()
```

#### **2. Type VECTOR (nouveau !)**

```typescript
// Prisma - type Vector pour IA/RAG
model Document {
  id        Int      @id @default(autoincrement())
  content   String   @db.Text
  embedding Bytes    // Stockage binaire du vecteur
}

// Utilisation (conversion manuelle pour l'instant)
const embedding = new Float32Array([0.1, 0.2, 0.3, ...]);
const buffer = Buffer.from(embedding.buffer);

await prisma.document.create({
  data: {
    content: 'Hello world',
    embedding: buffer
  }
});
```

⚠️ **Note** : Support natif du type VECTOR en cours d'ajout dans les ORM majeurs.

#### **3. UTF8MB4 par défaut**

```javascript
// Sequelize - Plus besoin de spécifier
const User = sequelize.define('User', {
    name: DataTypes.STRING  // UTF8MB4 par défaut en 11.8
});
```

#### **4. Collations UCA 14.0.0**

```csharp
// Entity Framework Core
modelBuilder.Entity<User>()
    .Property(u => u.Name)
    .UseCollation("utf8mb4_unicode_ci");  // UCA 14.0.0
```

---

## ⚠️ Pièges courants et solutions

### 🐛 **Problème 1 : N+1 Queries**

**Symptôme** :
```python
users = User.query.all()  # 1 requête
for user in users:
    print(len(user.posts))  # N requêtes !
```

**Diagnostic** :
```python
# Activer logging SQL
import logging
logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Résultat : 1 + N requêtes
# SELECT * FROM users
# SELECT * FROM posts WHERE user_id = 1
# SELECT * FROM posts WHERE user_id = 2
# ...
```

**Solution** : Eager loading
```python
users = User.query.options(joinedload(User.posts)).all()
# SELECT * FROM users LEFT JOIN posts ON users.id = posts.user_id
```

### 🐛 **Problème 2 : Modification détachée**

**Symptôme** :
```java
// Hibernate
User user = session.get(User.class, 1L);
session.close();

user.setName("Alice");  // Objet détaché !
// Erreur : No Session
```

**Solution** :
```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);
user.setName("Alice");  // Modification dans la session

tx.commit();
session.close();
```

### 🐛 **Problème 3 : Type mismatch**

**Symptôme** :
```csharp
// .NET - DateTime vs TIMESTAMP
public class Event {
    public DateTime OccurredAt { get; set; }
}

// Erreur : Timezone perdue lors du round-trip
```

**Solution** :
```csharp
// Spécifier UTC explicitement
public class Event {
    private DateTime _occurredAt;
    
    public DateTime OccurredAt {
        get => _occurredAt;
        set => _occurredAt = DateTime.SpecifyKind(value, DateTimeKind.Utc);
    }
}
```

### 🐛 **Problème 4 : Transactions oubliées**

**Symptôme** :
```python
# Pas de transaction explicite
user = User.query.get(1)
user.balance -= 100  # Débit
session.commit()  # Si crash ici...

other_user = User.query.get(2)
other_user.balance += 100  # Crédit jamais effectué !
session.commit()
```

**Solution** :
```python
# Transaction explicite
try:
    user1 = User.query.get(1)
    user2 = User.query.get(2)
    
    user1.balance -= 100
    user2.balance += 100
    
    session.commit()  # Atomique
except Exception as e:
    session.rollback()
    raise
```

---

## 💡 Bonnes pratiques

### ✅ À faire systématiquement

**1. Utiliser les transactions**
```python
# Context manager automatique
from contextlib import contextmanager

@contextmanager
def transaction_scope():
    session = Session()
    try:
        yield session
        session.commit()
    except:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
with transaction_scope() as session:
    user = User(name='Alice')
    session.add(user)
```

**2. Limiter le nombre de résultats**
```javascript
// Sequelize - Toujours paginer
const users = await User.findAll({
    limit: 100,
    offset: 0,
    order: [['id', 'ASC']]
});
```

**3. Projeter uniquement les colonnes nécessaires**
```csharp
// EF Core - Select seulement ce dont on a besoin
var userNames = await context.Users
    .Select(u => new { u.Id, u.Name })
    .ToListAsync();
// SELECT id, name FROM users (pas SELECT *)
```

**4. Indexer les colonnes recherchées**
```python
# SQLAlchemy
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    email = Column(String(255), index=True)  # Index pour recherche
```

**5. Utiliser les bulk operations**
```java
// Hibernate - Bulk insert
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

for (int i = 0; i < 10000; i++) {
    session.save(new User("User" + i, "user" + i + "@example.com"));
    
    if (i % 50 == 0) {
        session.flush();
        session.clear();  // Libérer mémoire
    }
}

tx.commit();
session.close();
```

### ❌ À éviter

1. ❌ Charger tous les résultats sans limite
2. ❌ Lazy loading dans les boucles (N+1)
3. ❌ Modifier sans transaction
4. ❌ Ignorer les migrations (modification manuelle schéma)
5. ❌ Utiliser l'ORM pour tout (analytics complexes)

---

## ✅ Points clés à retenir

- 🎯 **ORM = abstraction** : Objets ↔ Tables, productivité accrue
- ⚖️ **Trade-off** : Productivité vs Performance, abstraction vs contrôle
- 🔄 **Approche hybride** : ORM (CRUD, relations) + SQL natif (analytics)
- 🏆 **Leaders** : Hibernate (Java), SQLAlchemy (Python), EF Core (.NET), Prisma (TypeScript)
- 📊 **Migrations** : Versioning du schéma, rollback, automatisation
- ⚡ **Performance** : Attention N+1 queries, eager loading, pagination
- 🆕 **MariaDB 11.8** : Support JSON, Vector (en cours), UTF8MB4 par défaut
- 💡 **Bonnes pratiques** : Transactions, limit/offset, projections, bulk ops
- 🐛 **Pièges** : N+1, objets détachés, type mismatch, transactions oubliées

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [Hibernate ORM](https://hibernate.org/orm/documentation/)
- 📖 [SQLAlchemy](https://docs.sqlalchemy.org/)
- 📖 [Sequelize](https://sequelize.org/docs/)
- 📖 [Prisma](https://www.prisma.io/docs/)
- 📖 [Entity Framework Core](https://docs.microsoft.com/ef/core/)

### **Guides de performance**
- 📝 [N+1 Queries Explained](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem)
- 📝 [Hibernate Performance Best Practices](https://vladmihalcea.com/tutorials/hibernate/)
- 📝 [SQLAlchemy Performance](https://docs.sqlalchemy.org/en/14/faq/performance.html)

### **Comparaisons**
- 🔗 [ORM vs Query Builder vs Raw SQL](https://blog.logrocket.com/node-js-orms-why-shouldnt-use/)
- 🔗 [TypeORM vs Prisma vs Sequelize](https://www.prisma.io/docs/concepts/more/comparisons)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent chaque ORM en profondeur :

- **17.3.1** - Hibernate (Java) : Configuration, mappings, HQL, Criteria API
- **17.3.2** - SQLAlchemy (Python) : ORM + Core, Alembic migrations
- **17.3.3** - Sequelize (Node.js) : Modèles, associations, hooks
- **17.3.4** - Prisma (TypeScript) : Schema Prisma, type-safety, migrations
- **17.3.5** - Entity Framework Core (.NET) : Code-First, LINQ, migrations

---

**MariaDB** : Version 11.8 LTS

⏭️ [Hibernate (Java)](/17-integration-developpement/03.1-hibernate.md)

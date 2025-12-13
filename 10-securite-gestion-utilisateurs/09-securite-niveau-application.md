🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.9 Sécurité au niveau application

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 10.1-10.8, connaissances en développement applicatif

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Prévenir** les injections SQL avec prepared statements et validation
- **Gérer** les credentials de manière sécurisée (sans hardcoding)
- **Implémenter** le principe du moindre privilège au niveau applicatif
- **Configurer** des connection pools sécurisés
- **Utiliser** des gestionnaires de secrets (Vault, AWS Secrets Manager)
- **Chiffrer** les données sensibles au repos (application-level encryption)
- **Appliquer** les bonnes pratiques OWASP Top 10
- **Détecter** et corriger les erreurs courantes de sécurité
- **Auditer** le code applicatif pour les vulnérabilités SQL

---

## Introduction

La **sécurité au niveau application** est la **première ligne de défense** contre les attaques ciblant les bases de données. Même avec un MariaDB parfaitement configuré (TLS, audit, RBAC), une application mal codée peut compromettre toute la sécurité.

### Pyramide de sécurité

```
┌─────────────────────────────────────────────────────────────┐
│  NIVEAU 1: APPLICATION (Première défense) 🎯                │
│  - Validation des entrées                                   │
│  - Prepared statements                                      │
│  - Gestion sécurisée credentials                            │
│  - Principe moindre privilège                               │
├─────────────────────────────────────────────────────────────┤
│  NIVEAU 2: BASE DE DONNÉES                                  │
│  - Authentification (ed25519, PAM, PARSEC)                  │
│  - Autorisation (GRANT/REVOKE, RBAC)                        │
│  - Chiffrement (TLS, data-at-rest)                          │
│  - Audit (Server Audit Plugin)                              │
├─────────────────────────────────────────────────────────────┤
│  NIVEAU 3: INFRASTRUCTURE                                   │
│  - Firewall (iptables, AWS Security Groups)                 │
│  - Segmentation réseau (VLAN, VPC)                          │
│  - IDS/IPS (Snort, Suricata)                                │
│  - Monitoring (Datadog, Prometheus)                         │
└─────────────────────────────────────────────────────────────┘
```

**Principe clé** : La sécurité est une **chaîne** - le maillon le plus faible détermine la résistance totale.

### Statistiques alarmantes

```
OWASP Top 10 (2021):
┌─────────────────────────────────────────────────────────────┐
│  #1: Broken Access Control (↑ depuis #5 en 2017)            │
│  #3: Injection SQL (toujours dans le top 3)                 │
│  #7: Identification and Authentication Failures             │
│  #8: Software and Data Integrity Failures                   │
└─────────────────────────────────────────────────────────────┘

Verizon DBIR 2023:
- 43% des violations impliquent des applications web
- 74% des violations impliquent l'élément humain
- Injection SQL responsable de 19% des violations
```

---

## 1. Prévention des injections SQL

### Qu'est-ce qu'une injection SQL ?

**Définition** : Insertion de code SQL malveillant dans une requête via les entrées utilisateur.

**Exemple classique (vulnérable)** :

```python
# ❌ CODE VULNÉRABLE - NE JAMAIS FAIRE CECI
def login(username, password):
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    cursor.execute(query)
    return cursor.fetchone()

# Attaque:
username = "admin' --"
password = "n'importe quoi"

# Requête générée:
# SELECT * FROM users WHERE username = 'admin' --' AND password = 'n'importe quoi'
#                                              ^^^^
#                                              Tout après est commenté!
# → Connexion en tant qu'admin sans mot de passe! ⚠️
```

**Autre exemple (vol de données)** :

```php
// ❌ VULNÉRABLE
$id = $_GET['id'];
$query = "SELECT * FROM products WHERE id = $id";

// Attaque:
// URL: /product.php?id=1 UNION SELECT username,password,NULL FROM users --
// → Exfiltration de tous les utilisateurs et mots de passe! ⚠️
```

### Solution 1 : Prepared Statements (Requêtes préparées) ✅

**Principe** : Séparer le code SQL des données.

```
Non préparé (vulnérable):
┌─────────────────────────────────────────────┐
│ "SELECT * FROM users WHERE id = " + user_id │
│  [SQL]           [SQL]         [DONNÉE]     │
│  → Tout est interprété comme SQL            │
└─────────────────────────────────────────────┘

Préparé (sécurisé):
┌─────────────────────────────────────────────┐
│ "SELECT * FROM users WHERE id = ?"          │
│  [SQL]           [SQL]         [PLACEHOLDER]│
│                                             │
│ Paramètres: [user_id]                       │
│             [DONNÉE PURE]                   │
│  → Données ne peuvent JAMAIS être du SQL    │
└─────────────────────────────────────────────┘
```

#### Python (PyMySQL, mysqlclient)

```python
import pymysql

# ✅ BON: Prepared statement
def get_user(user_id):
    connection = pymysql.connect(
        host='db.example.com',
        user='app_user',
        password='secure_password',
        database='production',
        charset='utf8mb4'
    )

    try:
        with connection.cursor() as cursor:
            # Utiliser %s comme placeholder (PyMySQL)
            sql = "SELECT * FROM users WHERE id = %s"
            cursor.execute(sql, (user_id,))  # Tuple de paramètres
            result = cursor.fetchone()
            return result
    finally:
        connection.close()

# ✅ BON: Multiples paramètres
def authenticate(username, password):
    sql = "SELECT * FROM users WHERE username = %s AND password = SHA2(%s, 256)"
    cursor.execute(sql, (username, password))
    return cursor.fetchone()

# ✅ BON: INSERT avec prepared statement
def create_user(username, email, password):
    sql = "INSERT INTO users (username, email, password) VALUES (%s, %s, SHA2(%s, 256))"
    cursor.execute(sql, (username, email, password))
    connection.commit()
```

#### PHP (PDO)

```php
<?php
// ✅ BON: PDO avec prepared statements
$pdo = new PDO(
    'mysql:host=db.example.com;dbname=production;charset=utf8mb4',
    'app_user',
    'secure_password',
    [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_EMULATE_PREPARES => false,  // IMPORTANT: vraies prepared statements
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
    ]
);

// SELECT avec paramètre
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$user_id]);
$user = $stmt->fetch();

// INSERT avec paramètres nommés
$stmt = $pdo->prepare("INSERT INTO users (username, email) VALUES (:username, :email)");
$stmt->execute([
    'username' => $username,
    'email' => $email
]);

// UPDATE
$stmt = $pdo->prepare("UPDATE users SET last_login = NOW() WHERE id = ?");
$stmt->execute([$user_id]);
?>
```

#### Java (JDBC)

```java
import java.sql.*;

public class UserDAO {
    // ✅ BON: PreparedStatement
    public User getUser(int userId) throws SQLException {
        String sql = "SELECT * FROM users WHERE id = ?";

        try (Connection conn = DriverManager.getConnection(
                "jdbc:mariadb://db.example.com:3306/production",
                "app_user",
                "secure_password");
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setInt(1, userId);  // Paramètre sécurisé

            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return new User(
                        rs.getInt("id"),
                        rs.getString("username"),
                        rs.getString("email")
                    );
                }
            }
        }
        return null;
    }

    // ✅ BON: INSERT avec PreparedStatement
    public void createUser(String username, String email, String password)
            throws SQLException {
        String sql = "INSERT INTO users (username, email, password) " +
                     "VALUES (?, ?, SHA2(?, 256))";

        try (Connection conn = getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, username);
            stmt.setString(2, email);
            stmt.setString(3, password);
            stmt.executeUpdate();
        }
    }
}
```

#### Node.js (mysql2)

```javascript
const mysql = require('mysql2/promise');

// ✅ BON: Prepared statements avec mysql2
async function getUser(userId) {
    const connection = await mysql.createConnection({
        host: 'db.example.com',
        user: 'app_user',
        password: 'secure_password',
        database: 'production'
    });

    try {
        // Utiliser ? comme placeholder
        const [rows] = await connection.execute(
            'SELECT * FROM users WHERE id = ?',
            [userId]
        );
        return rows[0];
    } finally {
        await connection.end();
    }
}

// ✅ BON: INSERT
async function createUser(username, email, password) {
    const connection = await getConnection();

    const [result] = await connection.execute(
        'INSERT INTO users (username, email, password) VALUES (?, ?, SHA2(?, 256))',
        [username, email, password]
    );

    return result.insertId;
}

// ✅ BON: Avec named placeholders (mysql2)
async function authenticate(username, password) {
    const [rows] = await connection.execute(
        'SELECT * FROM users WHERE username = :username AND password = SHA2(:password, 256)',
        { username, password }
    );
    return rows[0];
}
```

### Solution 2 : Validation et sanitization des entrées

**Principe** : Valider **toutes** les entrées utilisateur.

```python
import re
from typing import Optional

# ✅ BON: Validation stricte
def validate_username(username: str) -> bool:
    """
    Username: 3-20 caractères alphanumériques + underscore
    """
    if not username or len(username) < 3 or len(username) > 20:
        return False
    return re.match(r'^[a-zA-Z0-9_]+$', username) is not None

def validate_email(email: str) -> bool:
    """
    Email: format standard
    """
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def validate_id(id_value: str) -> Optional[int]:
    """
    ID: entier positif uniquement
    """
    try:
        id_int = int(id_value)
        return id_int if id_int > 0 else None
    except ValueError:
        return None

# Utilisation
def get_user_safe(user_id_input: str):
    user_id = validate_id(user_id_input)
    if user_id is None:
        raise ValueError("Invalid user ID")

    # Maintenant safe pour la requête
    sql = "SELECT * FROM users WHERE id = %s"
    cursor.execute(sql, (user_id,))
    return cursor.fetchone()
```

**Validation côté application (exemples)** :

```python
# Validation selon le type de donnée
VALIDATORS = {
    'user_id': lambda x: isinstance(x, int) and x > 0,
    'username': lambda x: 3 <= len(x) <= 20 and x.isalnum(),
    'email': lambda x: '@' in x and len(x) <= 255,
    'age': lambda x: isinstance(x, int) and 0 <= x <= 150,
    'country_code': lambda x: len(x) == 2 and x.isalpha(),
    'phone': lambda x: re.match(r'^\+?[0-9]{10,15}$', x),
}

def validate_input(field: str, value) -> bool:
    validator = VALIDATORS.get(field)
    if validator is None:
        raise ValueError(f"No validator for field: {field}")
    return validator(value)
```

### Solution 3 : Whitelist (liste blanche)

```python
# ✅ BON: Whitelist pour colonnes et tables
ALLOWED_SORT_COLUMNS = {'id', 'username', 'created_at', 'email'}
ALLOWED_TABLES = {'users', 'products', 'orders'}

def get_users_sorted(sort_by: str):
    # Vérifier que la colonne est dans la whitelist
    if sort_by not in ALLOWED_SORT_COLUMNS:
        raise ValueError("Invalid sort column")

    # Safe d'utiliser directement (pas d'injection possible)
    sql = f"SELECT * FROM users ORDER BY {sort_by}"
    cursor.execute(sql)
    return cursor.fetchall()

# ❌ MAUVAIS: Accepter n'importe quelle colonne
def get_users_sorted_bad(sort_by: str):
    sql = f"SELECT * FROM users ORDER BY {sort_by}"  # ⚠️ Injection possible!
    # Attaque: sort_by = "id; DROP TABLE users; --"
```

### Erreurs courantes à éviter

```python
# ❌ ERREUR 1: Échapper manuellement les quotes
username = username.replace("'", "\\'")  # NE PAS FAIRE
sql = f"SELECT * FROM users WHERE username = '{username}'"  # Toujours vulnérable

# ❌ ERREUR 2: Construire des requêtes complexes avec concaténation
filters = []
if username:
    filters.append(f"username = '{username}'")  # Vulnérable
if email:
    filters.append(f"email = '{email}'")  # Vulnérable
where_clause = " AND ".join(filters)
sql = f"SELECT * FROM users WHERE {where_clause}"

# ✅ CORRECTION: Prepared statements dynamiques
conditions = []
params = []
if username:
    conditions.append("username = %s")
    params.append(username)
if email:
    conditions.append("email = %s")
    params.append(email)

where_clause = " AND ".join(conditions) if conditions else "1=1"
sql = f"SELECT * FROM users WHERE {where_clause}"
cursor.execute(sql, tuple(params))
```

---

## 2. Gestion sécurisée des credentials

### Problème : Hardcoded credentials ❌

```python
# ❌ ERREUR CRITIQUE: Mots de passe en dur dans le code
import pymysql

connection = pymysql.connect(
    host='db.example.com',
    user='app_user',
    password='SuperSecret123!',  # ⚠️ EN CLAIR DANS LE CODE SOURCE
    database='production'
)

# Risques:
# 1. Code source dans Git → mot de passe visible
# 2. Décompilation → mot de passe exposé
# 3. Logs d'erreur → mot de passe dans les traces
# 4. Rotation impossible sans redéploiement
```

### Solution 1 : Variables d'environnement

```python
# ✅ BON: Variables d'environnement
import os
import pymysql

connection = pymysql.connect(
    host=os.environ['DB_HOST'],
    user=os.environ['DB_USER'],
    password=os.environ['DB_PASSWORD'],
    database=os.environ['DB_NAME']
)
```

**Configuration Docker** :

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      DB_HOST: db.example.com
      DB_USER: app_user
      DB_PASSWORD: ${DB_PASSWORD}  # Depuis fichier .env
      DB_NAME: production
    env_file:
      - .env  # Fichier non versionné
```

```bash
# .env (NON versionné dans Git - .gitignore)
DB_PASSWORD=SuperSecret123!
```

**Configuration Kubernetes** :

```yaml
# kubernetes-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-credentials
type: Opaque
stringData:
  DB_HOST: db.example.com
  DB_USER: app_user
  DB_PASSWORD: SuperSecret123!
  DB_NAME: production

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        envFrom:
        - secretRef:
            name: mariadb-credentials
```

### Solution 2 : Gestionnaires de secrets

#### HashiCorp Vault

```python
# ✅ BON: HashiCorp Vault
import hvac
import pymysql

# Connexion à Vault
vault_client = hvac.Client(
    url='https://vault.example.com',
    token=os.environ['VAULT_TOKEN']  # Token depuis env ou K8s service account
)

# Récupérer les credentials depuis Vault
secret = vault_client.secrets.kv.v2.read_secret_version(
    path='database/production',
    mount_point='secret'
)

db_config = secret['data']['data']

# Connexion à MariaDB avec credentials Vault
connection = pymysql.connect(
    host=db_config['host'],
    user=db_config['username'],
    password=db_config['password'],
    database=db_config['database']
)

# Avantages:
# ✓ Credentials centralisés
# ✓ Rotation automatique
# ✓ Audit des accès
# ✓ Révocation immédiate possible
```

**Configuration Vault** :

```bash
# Activer le secret engine database
vault secrets enable database

# Configurer la connexion MariaDB
vault write database/config/production \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(db.example.com:3306)/" \
    allowed_roles="app-role" \
    username="vault_admin" \
    password="vault_admin_password"

# Créer un rôle avec credentials dynamiques
vault write database/roles/app-role \
    db_name=production \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; \
                         GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

# L'application demande des credentials (TTL 1h)
vault read database/creds/app-role
# Key                Value
# username           v-token-app-role-x7fj2k3l
# password           A1a-2b3c4d5e6f7g8h9i
```

#### AWS Secrets Manager

```python
# ✅ BON: AWS Secrets Manager
import boto3
import json
import pymysql

def get_secret():
    secret_name = "production/mariadb"
    region_name = "eu-west-1"

    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )

    get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    secret = json.loads(get_secret_value_response['SecretString'])
    return secret

# Utilisation
db_config = get_secret()
connection = pymysql.connect(
    host=db_config['host'],
    user=db_config['username'],
    password=db_config['password'],
    database=db_config['dbname']
)
```

**Création du secret AWS** :

```bash
aws secretsmanager create-secret \
    --name production/mariadb \
    --description "MariaDB production credentials" \
    --secret-string '{
        "host": "db.example.com",
        "username": "app_user",
        "password": "SuperSecret123!",
        "dbname": "production"
    }'

# Rotation automatique (90 jours)
aws secretsmanager rotate-secret \
    --secret-id production/mariadb \
    --rotation-lambda-arn arn:aws:lambda:eu-west-1:123456789:function:MariaDBRotation \
    --rotation-rules AutomaticallyAfterDays=90
```

#### Azure Key Vault

```python
# ✅ BON: Azure Key Vault
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import pymysql

# Connexion à Key Vault
credential = DefaultAzureCredential()
vault_url = "https://mykeyvault.vault.azure.net"
client = SecretClient(vault_url=vault_url, credential=credential)

# Récupérer les secrets
db_password = client.get_secret("mariadb-password").value
db_user = client.get_secret("mariadb-username").value

# Connexion MariaDB
connection = pymysql.connect(
    host='db.example.com',
    user=db_user,
    password=db_password,
    database='production'
)
```

### Solution 3 : IAM Authentication (Cloud)

#### AWS RDS IAM Authentication

```python
# ✅ BON: AWS RDS IAM (pas de mot de passe!)
import boto3
import pymysql

def get_iam_auth_token():
    rds_client = boto3.client('rds', region_name='eu-west-1')

    token = rds_client.generate_db_auth_token(
        DBHostname='db.example.com',
        Port=3306,
        DBUsername='app_user',
        Region='eu-west-1'
    )
    return token

# Connexion avec IAM token (15 min de validité)
connection = pymysql.connect(
    host='db.example.com',
    user='app_user',
    password=get_iam_auth_token(),  # Token IAM au lieu d'un mot de passe
    database='production',
    ssl={'ssl': True}  # SSL obligatoire pour IAM auth
)

# Avantages:
# ✓ Pas de mot de passe stocké
# ✓ Rotation automatique (token 15 min)
# ✓ Audit via AWS CloudTrail
# ✓ Gestion via IAM policies
```

**Configuration côté MariaDB** :

```sql
-- Créer utilisateur IAM
CREATE USER 'app_user'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO 'app_user'@'%';
```

---

## 3. Principe du moindre privilège

### Utilisateurs applicatifs dédiés

```sql
-- ❌ MAUVAIS: Application utilise le compte root
-- connection = pymysql.connect(user='root', password='...')

-- ✅ BON: Utilisateur dédié par application/service
CREATE USER 'webapp_readonly'@'app_servers'
  IDENTIFIED VIA ed25519 USING PASSWORD('...');

CREATE USER 'webapp_readwrite'@'app_servers'
  IDENTIFIED VIA ed25519 USING PASSWORD('...');

CREATE USER 'batch_jobs'@'batch_server'
  IDENTIFIED VIA ed25519 USING PASSWORD('...');

-- Privilèges minimaux
GRANT SELECT ON production.* TO 'webapp_readonly'@'app_servers';

GRANT SELECT, INSERT, UPDATE ON production.users TO 'webapp_readwrite'@'app_servers';
GRANT SELECT, INSERT, UPDATE ON production.orders TO 'webapp_readwrite'@'app_servers';
-- Pas de DELETE, pas de DDL

GRANT SELECT, INSERT, UPDATE, DELETE ON production.* TO 'batch_jobs'@'batch_server';
-- Batch a besoin de DELETE pour purges
```

### Séparation par fonctionnalité

```python
# ✅ BON: Connexions différentes selon l'opération
class DatabaseManager:
    def __init__(self):
        # Connexion lecture seule (SELECT)
        self.read_pool = self._create_pool(
            user='webapp_readonly',
            password=get_secret('webapp_readonly'),
            max_connections=50
        )

        # Connexion lecture/écriture (SELECT, INSERT, UPDATE)
        self.write_pool = self._create_pool(
            user='webapp_readwrite',
            password=get_secret('webapp_readwrite'),
            max_connections=10
        )

    def get_user(self, user_id):
        # Utiliser la connexion readonly
        with self.read_pool.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            return cursor.fetchone()

    def create_order(self, user_id, product_id):
        # Utiliser la connexion readwrite
        with self.write_pool.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO orders (user_id, product_id) VALUES (%s, %s)",
                (user_id, product_id)
            )
            conn.commit()
```

### Privilèges par table/colonne

```sql
-- ✅ BON: Privilèges granulaires par table
CREATE USER 'customer_service'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('...');

-- Lecture complète clients
GRANT SELECT ON production.customers TO 'customer_service'@'%';

-- Modification limitée (seulement certaines colonnes)
GRANT UPDATE (phone, email, address) ON production.customers TO 'customer_service'@'%';
-- Pas de UPDATE sur credit_card_number, password_hash

-- Lecture limitée commandes (pas de montants)
GRANT SELECT (id, customer_id, status, created_at) ON production.orders TO 'customer_service'@'%';
-- Pas de SELECT sur total_amount
```

---

## 4. Connection Pooling sécurisé

### Pourquoi le connection pooling ?

```
Sans pooling (mauvais):
┌────────────────────────────────────────────┐
│ Requête 1 → Connexion → Requête → Fermer   │
│ Requête 2 → Connexion → Requête → Fermer   │
│ Requête 3 → Connexion → Requête → Fermer   │
│ ...                                        │
│ Coût: 100-500ms par connexion              │
│ Overhead: TLS handshake, auth à chaque fois│
└────────────────────────────────────────────┘

Avec pooling (bon):
┌────────────────────────────────────────────┐
│ Pool: [Conn1] [Conn2] [Conn3] ... [Conn10] │
│                                            │
│ Requête 1 → Emprunter Conn1 → Rendre       │
│ Requête 2 → Emprunter Conn1 → Rendre       │
│ Requête 3 → Emprunter Conn2 → Rendre       │
│ ...                                        │
│ Coût: 0ms (connexions réutilisées)         │
│ Performance: +90%                          │
└────────────────────────────────────────────┘
```

### Python (DBUtils)

```python
# ✅ BON: Connection pooling avec DBUtils
from dbutils.pooled_db import PooledDB
import pymysql

# Créer un pool de connexions
db_pool = PooledDB(
    creator=pymysql,
    maxconnections=50,      # Max connexions dans le pool
    mincached=5,            # Min connexions en cache
    maxcached=10,           # Max connexions en cache
    maxshared=0,            # 0 = pas de partage (thread-safe)
    blocking=True,          # Bloquer si pool plein
    maxusage=1000,          # Recycler connexion après 1000 usages
    setsession=['SET AUTOCOMMIT = 1'],
    ping=1,                 # Vérifier connexion avant usage (0=jamais, 1=défaut, 2=toujours)
    host='db.example.com',
    user='app_user',
    password=get_secret('db_password'),
    database='production',
    charset='utf8mb4',
    ssl={'ssl': True}
)

# Utilisation
def get_user(user_id):
    conn = db_pool.connection()  # Emprunter du pool
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        return cursor.fetchone()
    finally:
        conn.close()  # Rendre au pool (pas vraiment fermé)
```

### Java (HikariCP) ⭐

```java
// ✅ BON: HikariCP (pool le plus performant)
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class DatabasePool {
    private static HikariDataSource dataSource;

    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mariadb://db.example.com:3306/production");
        config.setUsername("app_user");
        config.setPassword(getSecretFromVault("db_password"));

        // Configuration pool
        config.setMaximumPoolSize(50);           // Max connexions
        config.setMinimumIdle(5);                // Min connexions idle
        config.setConnectionTimeout(30000);      // Timeout 30s
        config.setIdleTimeout(600000);           // Idle timeout 10 min
        config.setMaxLifetime(1800000);          // Max lifetime 30 min
        config.setLeakDetectionThreshold(60000); // Détection fuites 1 min

        // Sécurité
        config.addDataSourceProperty("useSSL", "true");
        config.addDataSourceProperty("requireSSL", "true");
        config.addDataSourceProperty("verifyServerCertificate", "true");

        // Performance
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");

        dataSource = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}

// Utilisation
try (Connection conn = DatabasePool.getConnection();
     PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
    stmt.setInt(1, userId);
    try (ResultSet rs = stmt.executeQuery()) {
        // ...
    }
}
```

### Node.js (mysql2 pool)

```javascript
// ✅ BON: mysql2 connection pool
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
    host: 'db.example.com',
    user: 'app_user',
    password: await getSecretFromVault('db_password'),
    database: 'production',
    waitForConnections: true,
    connectionLimit: 50,        // Max connexions
    maxIdle: 10,                // Max connexions idle
    idleTimeout: 60000,         // Idle timeout 60s
    queueLimit: 0,              // Illimité
    enableKeepAlive: true,
    keepAliveInitialDelay: 0,
    ssl: {
        rejectUnauthorized: true,
        ca: fs.readFileSync('/path/to/ca-cert.pem')
    }
});

// Utilisation
async function getUser(userId) {
    const [rows] = await pool.execute(
        'SELECT * FROM users WHERE id = ?',
        [userId]
    );
    return rows[0];
}

// Graceful shutdown
process.on('SIGTERM', async () => {
    await pool.end();
    process.exit(0);
});
```

### PHP (PDO persistent)

```php
<?php
// ✅ BON: PDO avec connexions persistantes
class Database {
    private static $pdo = null;

    public static function getConnection() {
        if (self::$pdo === null) {
            $dsn = 'mysql:host=db.example.com;dbname=production;charset=utf8mb4';
            $options = [
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_PERSISTENT => true,  // Connexions persistantes (pooling PHP)
                PDO::ATTR_EMULATE_PREPARES => false,
                PDO::MYSQL_ATTR_SSL_CA => '/path/to/ca-cert.pem',
                PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT => true,
            ];

            self::$pdo = new PDO($dsn, 'app_user', getSecretFromVault('db_password'), $options);
        }
        return self::$pdo;
    }
}

// Utilisation
$pdo = Database::getConnection();
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$userId]);
?>
```

### Bonnes pratiques pooling

```python
# ✅ BON: Configuration sécurisée
pool_config = {
    # Taille
    'maxconnections': 50,           # Adapter selon charge (CPU cores × 2-4)
    'mincached': 5,                 # Toujours prêtes

    # Timeouts
    'connection_timeout': 30,       # 30s max pour obtenir connexion
    'idle_timeout': 600,            # 10 min idle → recycler
    'max_lifetime': 1800,           # 30 min max lifetime

    # Sécurité
    'ping': 1,                      # Vérifier connexion avant usage
    'reset_session': True,          # Reset variables session

    # SSL/TLS
    'ssl': {
        'ssl': True,
        'ssl_verify_cert': True,
        'ssl_ca': '/path/to/ca-cert.pem'
    }
}

# ❌ MAUVAIS: Pool trop grand
# maxconnections=1000  → Épuise ressources serveur
# maxconnections=1     → Goulot d'étranglement
```

---

## 5. Chiffrement des données sensibles

### Chiffrement au repos (application-level)

**Principe** : Chiffrer les données sensibles **avant** de les stocker dans MariaDB.

```python
# ✅ BON: Chiffrement AES-256-GCM au niveau application
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os
import base64

class DataEncryption:
    def __init__(self, master_key: bytes):
        """
        master_key: Clé 256 bits (32 bytes) depuis Vault/KMS
        """
        self.cipher = AESGCM(master_key)

    def encrypt(self, plaintext: str) -> str:
        """
        Chiffrer données avec AES-256-GCM
        Retourne: base64(nonce + ciphertext + tag)
        """
        plaintext_bytes = plaintext.encode('utf-8')
        nonce = os.urandom(12)  # 96 bits nonce

        # Chiffrement + authentification (GCM)
        ciphertext = self.cipher.encrypt(nonce, plaintext_bytes, None)

        # Concaténer nonce + ciphertext pour stockage
        encrypted = nonce + ciphertext
        return base64.b64encode(encrypted).decode('utf-8')

    def decrypt(self, encrypted_b64: str) -> str:
        """
        Déchiffrer données
        """
        encrypted = base64.b64decode(encrypted_b64)

        # Extraire nonce (12 premiers bytes)
        nonce = encrypted[:12]
        ciphertext = encrypted[12:]

        # Déchiffrement + vérification intégrité
        plaintext_bytes = self.cipher.decrypt(nonce, ciphertext, None)
        return plaintext_bytes.decode('utf-8')

# Utilisation
# Clé master depuis KMS/Vault (JAMAIS hardcodée)
master_key = get_encryption_key_from_kms()
encryptor = DataEncryption(master_key)

# Insertion
credit_card = "4532-1234-5678-9010"
encrypted_cc = encryptor.encrypt(credit_card)

cursor.execute(
    "INSERT INTO payments (user_id, credit_card_encrypted) VALUES (%s, %s)",
    (user_id, encrypted_cc)
)

# Lecture
cursor.execute("SELECT credit_card_encrypted FROM payments WHERE id = %s", (payment_id,))
encrypted_cc = cursor.fetchone()[0]
credit_card = encryptor.decrypt(encrypted_cc)
```

**Stockage dans MariaDB** :

```sql
-- Table avec colonnes chiffrées
CREATE TABLE payments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,

    -- Données en clair (cherchables)
    amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'completed', 'failed'),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Données chiffrées (application-level encryption)
    credit_card_encrypted VARCHAR(255) NOT NULL,  -- Base64(nonce + ciphertext)
    cvv_encrypted VARCHAR(100) NOT NULL,

    INDEX idx_user (user_id),
    INDEX idx_status (status)
);

-- Avantages:
-- ✓ Données chiffrées même si DB compromise
-- ✓ Clés de chiffrement dans KMS (pas dans DB)
-- ✓ Conformité PCI-DSS

-- Inconvénients:
-- ✗ Pas de recherche sur colonnes chiffrées
-- ✗ Overhead application (chiffrement/déchiffrement)
```

### Chiffrement sélectif vs complet

```python
# ✅ BON: Chiffrer uniquement les données sensibles (PII/PCI)
ENCRYPTED_FIELDS = {
    'users': ['ssn', 'credit_card', 'password'],  # PII
    'medical_records': ['diagnosis', 'treatment'],  # PHI (HIPAA)
    'payments': ['card_number', 'cvv'],  # PCI
}

# Données non sensibles en clair (performance)
NON_ENCRYPTED = {
    'users': ['username', 'email', 'created_at'],  # Cherchables
    'products': ['name', 'price', 'category'],  # Publiques
}
```

### Gestion des clés de chiffrement

```python
# ✅ BON: Clés de chiffrement depuis KMS
import boto3

def get_encryption_key_from_kms():
    """
    Récupérer clé de chiffrement depuis AWS KMS
    """
    kms_client = boto3.client('kms', region_name='eu-west-1')

    # Générer une data key (AES-256)
    response = kms_client.generate_data_key(
        KeyId='arn:aws:kms:eu-west-1:123456789:key/12345678-1234',
        KeySpec='AES_256'
    )

    # Plaintext key (32 bytes) - utilisée pour chiffrement
    plaintext_key = response['Plaintext']

    # Encrypted key - stockée avec les données (pour rotation)
    encrypted_key = response['CiphertextBlob']

    return plaintext_key

# Rotation automatique (90 jours)
# AWS KMS gère la rotation de la master key
# Application re-chiffre les data keys périodiquement
```

---

## 6. Logging et monitoring applicatif

### Logging sécurisé

```python
import logging

# ✅ BON: Logger les événements sécurité
logger = logging.getLogger('security')
logger.setLevel(logging.INFO)

handler = logging.FileHandler('/var/log/app/security.log')
formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
handler.setFormatter(formatter)
logger.addHandler(handler)

# Logger connexions
def login(username, password):
    try:
        user = authenticate(username, password)
        logger.info(f"Successful login: user={username}, ip={request.remote_addr}")
        return user
    except AuthenticationError:
        logger.warning(f"Failed login attempt: user={username}, ip={request.remote_addr}")
        raise

# Logger accès données sensibles
def get_credit_card(user_id, card_id):
    logger.info(f"Credit card access: user_id={user_id}, card_id={card_id}, ip={request.remote_addr}")
    # ...

# ❌ MAUVAIS: Logger des données sensibles
logger.info(f"Credit card: {credit_card_number}")  # ⚠️ Fuite de données!
logger.debug(f"Password: {password}")  # ⚠️ JAMAIS logger les mots de passe!
```

### Détection d'anomalies

```python
# ✅ BON: Détecter comportements anormaux
from datetime import datetime, timedelta
from collections import defaultdict

class AnomalyDetector:
    def __init__(self):
        self.failed_logins = defaultdict(list)
        self.query_counts = defaultdict(int)

    def check_brute_force(self, username, ip):
        """
        Détecter brute force: >5 échecs en 5 min
        """
        now = datetime.now()
        cutoff = now - timedelta(minutes=5)

        # Nettoyer anciennes tentatives
        self.failed_logins[ip] = [
            ts for ts in self.failed_logins[ip] if ts > cutoff
        ]

        # Ajouter nouvelle tentative
        self.failed_logins[ip].append(now)

        # Bloquer si > 5 tentatives
        if len(self.failed_logins[ip]) > 5:
            logger.critical(f"BRUTE FORCE DETECTED: ip={ip}, username={username}")
            # Bloquer IP, notifier SOC
            return True

        return False

    def check_sql_injection_pattern(self, query_string):
        """
        Détecter patterns d'injection SQL
        """
        sql_injection_patterns = [
            "' OR '1'='1",
            "'; DROP TABLE",
            "UNION SELECT",
            "' --",
            "'; EXEC",
        ]

        for pattern in sql_injection_patterns:
            if pattern.lower() in query_string.lower():
                logger.critical(f"SQL INJECTION ATTEMPT: query={query_string}")
                # Bloquer requête, alerter
                return True

        return False

# Utilisation
detector = AnomalyDetector()

def login(username, password):
    if detector.check_brute_force(username, request.remote_addr):
        raise SecurityException("Too many failed attempts")
    # ...
```

---

## 7. Gestion des erreurs sécurisée

### Messages d'erreur

```python
# ❌ MAUVAIS: Exposer détails techniques
try:
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
except Exception as e:
    return {"error": str(e)}  # ⚠️ Expose structure DB, tables, etc.

# Erreur retournée à l'utilisateur:
# "Table 'production.users' doesn't exist"
# → Révèle nom base, tables

# ✅ BON: Messages génériques + logging détaillé
try:
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
except Exception as e:
    # Logger détails (serveur)
    logger.error(f"Database error: {e}", exc_info=True)

    # Retourner message générique (client)
    return {"error": "An error occurred. Please contact support."}
```

### Gestion des exceptions

```python
# ✅ BON: Hiérarchie d'exceptions
class DatabaseError(Exception):
    """Base exception pour erreurs DB"""
    pass

class ConnectionError(DatabaseError):
    """Erreur de connexion"""
    pass

class QueryError(DatabaseError):
    """Erreur de requête"""
    pass

class DataNotFoundError(DatabaseError):
    """Données introuvables"""
    pass

# Utilisation
def get_user(user_id):
    try:
        conn = get_connection()
    except Exception as e:
        logger.error(f"Connection failed: {e}")
        raise ConnectionError("Unable to connect to database")

    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        result = cursor.fetchone()

        if result is None:
            raise DataNotFoundError(f"User {user_id} not found")

        return result
    except pymysql.Error as e:
        logger.error(f"Query failed: {e}")
        raise QueryError("Database query failed")
    finally:
        conn.close()
```

---

## ✅ Points clés à retenir

- **Toujours utiliser prepared statements** : protège contre injection SQL
- **Jamais hardcoder les credentials** : variables d'environnement, Vault, KMS
- **Principe du moindre privilège** : utilisateurs dédiés par app/service
- **Connection pooling obligatoire** : performance + gestion ressources
- **Validation stricte des entrées** : whitelist, regex, type checking
- **Chiffrement application-level** : données sensibles (PCI, PII, PHI)
- **Logging sécurisé** : événements sans données sensibles
- **Messages d'erreur génériques** : pas de détails techniques aux clients
- **Détection d'anomalies** : brute force, injection SQL, accès suspects
- **Secrets management centralisé** : Vault, AWS Secrets Manager, Azure Key Vault

---

## 🔗 Ressources et références

### Sécurité applicative

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)

### Bibliothèques et frameworks

- [HikariCP (Java)](https://github.com/brettwooldridge/HikariCP)
- [DBUtils (Python)](https://webwareforpython.github.io/DBUtils/)
- [mysql2 (Node.js)](https://github.com/sidorares/node-mysql2)

### Secrets management

- [HashiCorp Vault](https://www.vaultproject.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [Azure Key Vault](https://azure.microsoft.com/en-us/products/key-vault/)

---

## ➡️ Section suivante

**10.10 : Sécurité du serveur et durcissement** - Vous apprendrez à sécuriser le serveur MariaDB au niveau OS et réseau (firewall, SELinux, AppArmor, configuration système).

---


⏭️ [Password validation plugins et politiques](/10-securite-gestion-utilisateurs/10-password-validation.md)

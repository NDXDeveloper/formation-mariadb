🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.4 Architecture générale d'un SGBD relationnel

> **Niveau** : Débutant
> **Durée estimée** : 1 heure
> **Prérequis** : Sections 1.1, 1.2, 1.3

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture en couches d'un SGBDR
- Identifier les composants principaux de MariaDB Server
- Suivre le parcours d'une requête SQL de bout en bout
- Comprendre comment les données sont stockées physiquement
- Connaître les mécanismes de gestion de la mémoire
- Distinguer les différents processus et threads
- Comprendre l'architecture pluggable des moteurs de stockage

---

## Introduction

Vous savez maintenant **ce qu'est** MariaDB et **pourquoi l'utiliser**. Mais comment fonctionne-t-il réellement sous le capot ? Comment une simple requête SQL comme `SELECT * FROM users` se transforme-t-elle en opérations sur le disque dur ?

Comprendre l'architecture de MariaDB vous permettra de :
- 🎯 **Optimiser** vos requêtes en sachant comment elles sont traitées
- 🔍 **Diagnostiquer** les problèmes de performance
- ⚙️ **Configurer** le serveur de manière éclairée
- 🏗️ **Concevoir** de meilleures bases de données

Dans cette section, nous allons explorer l'architecture de MariaDB **couche par couche**, du client jusqu'au stockage physique sur disque.

---

## Vue d'ensemble : Architecture en couches

Un SGBDR comme MariaDB est organisé en **couches empilées**, chacune ayant un rôle spécifique.

### 🏗️ Les 5 couches principales

```
┌─────────────────────────────────────────────────────────────┐
│                    COUCHE 1 : CLIENTS                       │
│  Applications, CLI, GUI, API (PHP, Python, Java, Node.js)   │
└──────────────────────────┬──────────────────────────────────┘
                           │ Protocole MySQL/MariaDB
                           │ (Port 3306 par défaut)
                           ↓
┌─────────────────────────────────────────────────────────────┐
│              COUCHE 2 : CONNEXION ET SÉCURITÉ               │
│  - Gestion connexions (threads)                             │
│  - Authentification (user/password, certificats)            │
│  - Autorisation (privileges, permissions)                   │
│  - SSL/TLS encryption                                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│               COUCHE 3 : TRAITEMENT SQL                     │
│  - Parser SQL (analyse syntaxe)                             │
│  - Optimizer (choix meilleur plan d'exécution)              │
│  - Query Cache (deprecated mais présent)                    │
│  - Préparation de l'exécution                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│             COUCHE 4 : MOTEURS DE STOCKAGE                  │
│  InnoDB | Aria | MyISAM | ColumnStore | Memory | Vector     │
│  (Pluggable Storage Engine Architecture)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│              COUCHE 5 : STOCKAGE PHYSIQUE                   │
│  - Fichiers de données (.ibd, .frm, .MYD)                   │
│  - Fichiers de log (redo, undo, binary log)                 │
│  - Système de fichiers (ext4, XFS, NTFS)                    │
│  - Disques (HDD, SSD, NVMe)                                 │
└─────────────────────────────────────────────────────────────┘
```

### 📝 Analogie : Une bibliothèque

Pour mieux comprendre, comparons MariaDB à une **bibliothèque** :

| Couche MariaDB | Analogie bibliothèque |
|----------------|----------------------|
| **Clients** | Lecteurs qui viennent consulter des livres |
| **Connexion** | Bureau d'accueil : carte de bibliothèque, vérification identité |
| **Traitement SQL** | Bibliothécaire qui comprend votre demande et cherche le meilleur chemin |
| **Moteurs de stockage** | Différentes sections (romans, BD, archives, magazines) |
| **Stockage physique** | Étagères physiques avec les livres |

---

## Couche 1 : Les clients

### 🖥️ Types de clients

Les **clients** sont les programmes qui se connectent à MariaDB pour envoyer des requêtes SQL.

**Clients courants** :
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  CLI Tool   │  │   Web App   │  │  Mobile App │  │   Desktop   │
│  (mariadb)  │  │ (PHP/Node)  │  │  (API REST) │  │  (DBeaver)  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       └────────────────┴────────────────┴────────────────┘
                               │
                               ↓ TCP/IP (Port 3306)
                    ┌──────────────────────┐
                    │   MariaDB Server     │
                    └──────────────────────┘
```

### 🔌 Protocole de communication

Les clients communiquent avec MariaDB via le **protocole MySQL/MariaDB** :

1. **Connexion TCP/IP** (port 3306 par défaut)
2. **Handshake** : Échange de paramètres
3. **Authentification** : Vérification credentials
4. **Requêtes/Réponses** : Échange SQL
5. **Déconnexion** : Fermeture propre

**Exemple : Flux de connexion**
```
Client                          MariaDB Server
  │                                    │
  ├─── Connection Request ────────────►│
  │                                    │
  │◄─── Server Greeting ───────────────┤
  │    (version, capabilities)         │
  │                                    │
  ├─── Authentication ────────────────►│
  │    (user, password, database)      │
  │                                    │
  │◄─── OK / Error ────────────────────┤
  │                                    │
  ├─── SQL Query ─────────────────────►│
  │    SELECT * FROM users;            │
  │                                    │
  │◄─── Result Set ────────────────────┤
  │    (rows, columns, data)           │
  │                                    │
  ├─── Close ─────────────────────────►│
  │                                    │
```

---

## Couche 2 : Connexion et sécurité

### 🔐 Gestion des connexions

Lorsqu'un client se connecte, MariaDB doit :
1. **Accepter** la connexion réseau
2. **Créer** un thread dédié (ou le récupérer du thread pool)
3. **Authentifier** l'utilisateur
4. **Vérifier** les permissions
5. **Maintenir** la session active

#### Thread per connection (modèle classique)

```
Client 1 ──────► Thread 1 ──────► Requêtes SQL
Client 2 ──────► Thread 2 ──────► Requêtes SQL
Client 3 ──────► Thread 3 ──────► Requêtes SQL
Client 4 ──────► Thread 4 ──────► Requêtes SQL
   ...              ...               ...
Client N ──────► Thread N ──────► Requêtes SQL
```

💡 **Limite** : Chaque thread consomme de la mémoire (256 KB - 4 MB). Avec 10 000 connexions = plusieurs GB de RAM !

#### Thread Pool (amélioration) 🆕

```
Client 1 ──┐
Client 2 ──┤
Client 3 ──┼───► Thread Pool ───► Requêtes SQL
Client 4 ──┤     (ex: 16 threads)
   ...     ┤
Client N ──┘
```

💡 **Avantage** : Gère des milliers de connexions avec peu de threads. **Gratuit dans MariaDB** (payant dans MySQL Enterprise).

### 🔒 Authentification

MariaDB vérifie l'identité via plusieurs méthodes :

| Méthode | Description | Usage |
|---------|-------------|-------|
| **mysql_native_password** | Hash SHA1 du mot de passe | Standard MySQL/MariaDB |
| **ed25519** | Cryptographie moderne (courbes elliptiques) | Recommandé MariaDB 🆕 |
| **caching_sha2_password** | SHA256 avec cache | MySQL 8.0+ |
| **unix_socket** | Auth via user Unix/Linux | Local uniquement |
| **PAM** | Pluggable Authentication Modules | LDAP, Kerberos, etc. |
| **PARSEC** 🆕 | Plugin de sécurité avancé | MariaDB 11.8 |

**Exemple : Vérification des droits**
```sql
-- L'utilisateur 'webapp' essaie de se connecter
-- MariaDB vérifie dans la table mysql.user

SELECT Host, User, authentication_string
FROM mysql.user
WHERE User = 'webapp';

-- Si authentification OK, vérifie les permissions
SHOW GRANTS FOR 'webapp'@'localhost';
```

### 🔐 SSL/TLS Encryption 🆕

**Depuis MariaDB 11.8, TLS est activé par défaut !**

```
Client                          MariaDB Server
  │                                    │
  ├─── Connection Request ────────────►│
  │                                    │
  │◄─── TLS Handshake ─────────────────┤
  │    (certificats, chiffrement)      │
  │                                    │
  ├─── Encrypted SQL ─────────────────►│
  │    🔒🔒🔒🔒🔒🔒🔒🔒🔒             │
  │                                    │
  │◄─── Encrypted Results ─────────────┤
  │    🔒🔒🔒🔒🔒🔒🔒🔒🔒             │
```

---

## Couche 3 : Traitement SQL

C'est le **cerveau** de MariaDB. Cette couche transforme votre SQL en opérations concrètes.

### 📝 Le parcours d'une requête SQL

Suivons une requête de sa réception à son exécution :

```sql
SELECT u.name, u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.country = 'France'
GROUP BY u.id
HAVING order_count > 5
ORDER BY order_count DESC
LIMIT 10;
```

#### Étape 1 : Parser SQL (Analyseur syntaxique)

Le **parser** transforme le texte SQL en structure de données compréhensible.

```
Texte SQL ──► Lexical Analysis ──► Tokens ──► Syntax Analysis ──► Parse Tree
```

**Vérifications** :
- ✅ Syntaxe correcte ?
- ✅ Tables existent ?
- ✅ Colonnes existent ?
- ✅ Types compatibles ?

**Sortie** : Un **arbre syntaxique** (parse tree)

```
                    SELECT
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    Columns        FROM          WHERE
   (name,email)   (JOIN)      (country='France')
                    │
              ┌─────┴─────┐
           users       orders
```

#### Étape 2 : Optimizer (Optimiseur de requêtes)

L'**optimiseur** détermine le **meilleur plan d'exécution** pour obtenir les résultats le plus rapidement possible.

**Questions de l'optimiseur** :
- 🤔 Dans quel ordre lire les tables ?
- 🤔 Utiliser quel index ?
- 🤔 Utiliser un index covering ?
- 🤔 Faire un index scan ou un table scan ?
- 🤔 Quelle méthode de jointure (nested loop, hash join) ?

**Exemple : Choix d'index**
```sql
-- La table users a 2 index :
-- 1. PRIMARY KEY (id)
-- 2. INDEX idx_country (country)

-- Pour WHERE country = 'France', l'optimiseur choisit idx_country
EXPLAIN SELECT * FROM users WHERE country = 'France';

+------+-------+---------------+
| type | key   | rows          |
+------+-------+---------------+
| ref  | idx_country | ~50000 |  ← INDEX UTILISÉ
+------+-------+---------------+
```

**Types de scans (du meilleur au pire)** :
1. 🚀 **const** : Lecture d'une seule ligne (PRIMARY KEY)
2. ⚡ **eq_ref** : Une ligne par jointure (clé unique)
3. ✅ **ref** : Plusieurs lignes via index (non unique)
4. 🔍 **range** : Scan d'une plage d'index (BETWEEN, >, <)
5. 🐌 **index** : Scan complet d'index
6. ❌ **ALL** : Full table scan (à éviter !)

💡 **L'optimiseur utilise des statistiques** sur les tables (nombre de lignes, distribution des valeurs) pour décider.

#### Étape 3 : Query Cache (déprécié)

⚠️ **Note** : Le query cache est **déprécié** dans MariaDB et MySQL 8.0 (problèmes de performance et de scalabilité).

Anciennement : Cache les résultats des SELECT pour éviter de les recalculer.

#### Étape 4 : Exécution

Le plan d'exécution est envoyé aux **moteurs de stockage** pour récupérer les données.

```
┌──────────────────────────────────────────────┐
│          Execution Engine                    │
├──────────────────────────────────────────────┤
│  1. Scan users with idx_country              │
│  2. For each user, lookup orders (JOIN)      │
│  3. Count orders (GROUP BY)                  │
│  4. Filter (HAVING order_count > 5)          │
│  5. Sort by order_count DESC                 │
│  6. Return top 10 (LIMIT)                    │
└────────────┬─────────────────────────────────┘
             │
             ↓ Appels API Storage Engine
┌─────────────────────────────────────────────┐
│        InnoDB Storage Engine                │
│  - Lecture pages disque                     │
│  - Utilisation buffer pool (cache mémoire)  │
│  - Gestion locks et transactions            │
└─────────────────────────────────────────────┘
```

---

## Couche 4 : Moteurs de stockage (Storage Engines)

### 🎛️ Architecture Pluggable

MariaDB utilise une **architecture modulaire** : différents moteurs de stockage peuvent être utilisés selon les besoins.

```
┌─────────────────────────────────────────────────────────┐
│              SQL Layer (commun)                         │
│  Parser, Optimizer, Cache, Query Execution              │
└────────────────────────┬────────────────────────────────┘
                         │ API Storage Engine
        ┌────────────────┼────────────────┐
        │                │                │
┌───────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│   InnoDB     │  │    Aria     │  │ ColumnStore │
│ (Défaut)     │  │  (Crash-    │  │  (Analytics)│
│ ACID, FK     │  │   safe)     │  │   OLAP      │
└──────────────┘  └─────────────┘  └─────────────┘
        │                │                │
┌───────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│   MyISAM     │  │   Memory    │  │  S3 🆕      │
│  (Legacy)    │  │   (RAM)     │  │ (Archive)   │
└──────────────┘  └─────────────┘  └─────────────┘
        │
┌───────▼──────┐
│  Vector 🆕   │
│  (HNSW AI)   │
└──────────────┘
```

### 🔧 Comparaison des moteurs principaux

| Moteur | Usage | Transactions | Clés étrangères | Performance |
|--------|-------|--------------|-----------------|-------------|
| **InnoDB** | OLTP, défaut | ✅ ACID | ✅ Oui | ⚡⚡⚡⚡ |
| **Aria** | Tables système | ✅ Crash-safe | ❌ Non | ⚡⚡⚡ |
| **MyISAM** | Legacy | ❌ Non | ❌ Non | ⚡⚡ |
| **ColumnStore** | OLAP, BI | ✅ Oui | ❌ Non | ⚡⚡⚡⚡⚡ (agrégations) |
| **Memory** | Cache, temp | ❌ Non | ❌ Non | ⚡⚡⚡⚡⚡ (RAM) |
| **S3** 🆕 | Archivage | ❌ Non | ❌ Non | 🐌 (lectures) |
| **Vector** 🆕 | IA, RAG | ✅ Oui | ❌ Non | ⚡⚡⚡⚡ (similarité) |

**Exemple : Choisir le moteur**
```sql
-- Table transactionnelle (défaut)
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    amount DECIMAL(10,2),
    created_at TIMESTAMP
) ENGINE=InnoDB;  -- ACID, transactions

-- Table analytics (agrégations massives)
CREATE TABLE sales_facts (
    date DATE,
    product_id INT,
    quantity INT,
    revenue DECIMAL(12,2)
) ENGINE=ColumnStore;  -- Optimisé pour SUM, COUNT, AVG

-- Table cache temporaire (RAM)
CREATE TABLE session_cache (
    session_id VARCHAR(64) PRIMARY KEY,
    data TEXT,
    expires_at TIMESTAMP
) ENGINE=Memory;  -- Ultra rapide, volatil

-- Table vecteurs IA 🆕
CREATE TABLE document_embeddings (
    id INT PRIMARY KEY AUTO_INCREMENT,
    content TEXT,
    embedding VECTOR(1536),
    VECTOR INDEX idx_emb (embedding) USING HNSW
) ENGINE=InnoDB;  -- Recherche vectorielle
```

### 🔍 InnoDB en détail (moteur par défaut)

**InnoDB** est le moteur le plus important à comprendre.

#### Architecture InnoDB

```
┌──────────────────────────────────────────────────────────┐
│                     InnoDB                               │
├──────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────┐  │
│  │          Buffer Pool (RAM)                         │  │
│  │  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │ Data Pages   │  │ Index Pages  │                │  │
│  │  │ (tables)     │  │ (B-Tree)     │                │  │
│  │  └──────────────┘  └──────────────┘                │  │
│  │  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │ Change Buffer│  │ Adaptive Hash│                │  │
│  │  │ (async write)│  │ Index        │                │  │
│  │  └──────────────┘  └──────────────┘                │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Log Files                             │  │
│  │  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │  Redo Log    │  │  Undo Log    │                │  │
│  │  │ (durability) │  │ (rollback)   │                │  │
│  │  └──────────────┘  └──────────────┘                │  │
│  └────────────────────────────────────────────────────┘  │
│                             │                            │
│                             ↓                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │         Tablespaces (fichiers .ibd)                │  │
│  │  - ibdata1 (system tablespace)                     │  │
│  │  - table1.ibd (file per table)                     │  │
│  │  - table2.ibd                                      │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Composants clés** :
1. **Buffer Pool** : Cache mémoire des pages les plus utilisées
2. **Redo Log** : Garantit la durabilité (ACID)
3. **Undo Log** : Permet le rollback et MVCC
4. **Tablespaces** : Fichiers physiques sur disque

---

## Couche 5 : Stockage physique

### 💾 Comment les données sont stockées sur disque

#### Structure des fichiers InnoDB

**Exemple de répertoire de données** :
```bash
/var/lib/mysql/
├── ibdata1                    # System tablespace
├── ib_logfile0                # Redo log
├── ib_logfile1                # Redo log (rotation)
├── mysql/                     # Base système
│   ├── user.frm
│   ├── user.ibd
│   └── ...
└── myapp/                     # Votre base de données
    ├── users.frm              # Définition table (structure)
    ├── users.ibd              # Données + index InnoDB
    ├── orders.frm
    ├── orders.ibd
    └── products.ibd
```

**Extensions de fichiers** :
- `.frm` : **For**mat (structure de la table)
- `.ibd` : **I**nno**D**B **D**ata (données et index)
- `.MYD` : **My**ISAM **D**ata (pour MyISAM)
- `.MYI` : **My**ISAM **I**ndex (pour MyISAM)

#### Pages InnoDB

InnoDB organise les données en **pages de 16 KB** (par défaut).

```
Fichier .ibd
┌────────────────────────────────────────────────────┐
│ Page 0 (16 KB)  │ Page 1 (16 KB)  │ Page 2 ...     │
├────────────────────────────────────────────────────┤
│ - Header        │ - Header        │                │
│ - Rows data     │ - Rows data     │                │
│ - Footer        │ - Footer        │                │
└────────────────────────────────────────────────────┘
```

**Une page contient** :
- **Header** : Métadonnées (page number, checksums)
- **Rows** : Données des lignes
- **Footer** : Informations de consistance

**Exemple : Calcul du nombre de lignes par page**
```
Taille page      = 16 384 bytes
- Header/Footer  =   ~200 bytes
- Espace dispo   = 16 000 bytes

Si une ligne = 200 bytes
→ ~80 lignes par page

Table de 1 million de lignes
→ ~12 500 pages
→ ~195 MB sur disque
```

### 📊 Index B-Tree

Les **index** sont organisés en **arbres B-Tree** (Balanced Tree).

```
                     [Page Root]
                    /     |     \
                   /      |      \
              [Page]   [Page]   [Page]
              /  \      /  \      /  \
         [Leaf] [Leaf][Leaf][Leaf][Leaf][Leaf]
          │      │      │      │      │      │
        rows   rows   rows   rows   rows   rows
```

**Propriétés B-Tree** :
- ✅ **Équilibré** : Toutes les feuilles à la même profondeur
- ✅ **Ordonné** : Recherche rapide par dichotomie
- ✅ **Compact** : Peu de niveaux (3-4 pour millions de lignes)

**Exemple : Recherche dans un index**
```sql
-- Table users avec index sur email
SELECT * FROM users WHERE email = 'john@example.com';

Étapes :
1. Lire la page root de l'index idx_email
2. Suivre le pointeur vers la page intermédiaire
3. Suivre le pointeur vers la page feuille
4. Trouver la ligne correspondante
5. Utiliser le pointeur pour lire la page de données

→ 3-4 lectures de pages (index) + 1 lecture (data)
→ Très rapide même avec millions de lignes !
```

---

## Gestion de la mémoire

### 🧠 Buffer Pool (cache mémoire principal)

Le **Buffer Pool** est le cache mémoire le plus important de MariaDB.

**Rôle** : Mettre en cache les pages de données et d'index les plus utilisées pour éviter les lectures disque.

```
                Applications
                     ↓
           ┌─────────────────┐
           │   SQL Layer     │
           └────────┬────────┘
                    ↓
        ┌───────────────────────────┐
        │      Buffer Pool          │
        │    (Configurable size)    │
        ├───────────────────────────┤
        │  ✓ Pages en cache (RAM)   │ ← Lecture rapide !
        │  ✗ Pages non cachées      │
        └───────────┬───────────────┘
                    │
        ┌───────────▼───────────────┐
        │      Disque (SSD/HDD)     │ ← Lecture lente
        └───────────────────────────┘
```

**Configuration** :
```ini
[mysqld]
# Règle : 50-80% de la RAM serveur
innodb_buffer_pool_size = 8G   # Ex: serveur 16 GB RAM

# Multiple instances pour réduire contention (8+ GB)
innodb_buffer_pool_instances = 8
```

**Métriques importantes** :
```sql
-- Voir l'efficacité du buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Taux de cache hit (doit être > 99%)
Buffer Pool Hit Rate = (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
```

### 📝 Autres caches

| Cache | Rôle | Taille |
|-------|------|--------|
| **Buffer Pool** | Pages données + index (InnoDB) | Large (GB) |
| **Key Buffer** | Index MyISAM uniquement | Moyenne (MB) |
| **Table Cache** | Descripteurs de tables ouvertes | Petite (count) |
| **Thread Cache** | Réutilisation threads connexions | Petite (count) |
| **Query Cache** | ⚠️ Déprécié | - |

---

## Processus et threads

### 🔄 Architecture des processus

**MariaDB fonctionne avec 1 processus principal et plusieurs threads.**

```bash
$ ps aux | grep mariadb
mysql    1234  ... /usr/sbin/mariadbd
         └─► Processus principal (mysqld/mariadbd)
                 │
                 ├─► Thread 1: Connection handler
                 ├─► Thread 2: Query execution
                 ├─► Thread 3: InnoDB background
                 ├─► Thread 4: InnoDB I/O
                 ├─► Thread 5: InnoDB purge
                 └─► Thread N: ...
```

**Types de threads** :

#### 1️⃣ **Threads de connexion**
- Un thread par connexion client (modèle classique)
- Ou thread pool (gère N connexions avec M threads)

#### 2️⃣ **Threads InnoDB background**
```
┌────────────────────────────────────────┐
│      InnoDB Background Threads         │
├────────────────────────────────────────┤
│ • Master Thread                        │
│   - Coordination générale              │
│                                        │
│ • I/O Read Threads (4 par défaut)      │
│   - Lectures asynchrones               │
│                                        │
│ • I/O Write Threads (4 par défaut)     │
│   - Écritures asynchrones              │
│                                        │
│ • Log Thread                           │
│   - Flush redo log sur disque          │
│                                        │
│ • Purge Threads (1-4)                  │
│   - Nettoyage undo logs                │
│                                        │
│ • Page Cleaner Thread                  │
│   - Flush dirty pages buffer pool      │
└────────────────────────────────────────┘
```

**Configuration** :
```ini
[mysqld]
# Threads I/O (augmenter pour SSD/NVMe)
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# Threads purge (nettoyage)
innodb_purge_threads = 4
```

---

## Le parcours complet d'une requête (récapitulatif)

Reprenons tout depuis le début avec un exemple concret :

```sql
SELECT name, email FROM users WHERE id = 12345;
```

### 🚀 Étapes détaillées

```
[1] CLIENT
    │ Application envoie la requête SQL
    ↓ TCP/IP (port 3306)

[2] CONNEXION
    │ Thread dédié reçoit la requête
    │ Vérifie permissions (SELECT sur users ?)
    ↓

[3] PARSER
    │ Analyse syntaxe SQL
    │ Valide tables et colonnes existent
    ↓ Parse Tree

[4] OPTIMIZER
    │ Analyse : WHERE id = 12345
    │ → id est PRIMARY KEY
    │ → Plan : Index scan sur PRIMARY (ultra rapide)
    ↓ Execution Plan

[5] EXECUTION ENGINE
    │ Appelle InnoDB : "Give me row with id=12345"
    ↓

[6] InnoDB STORAGE ENGINE
    │ 1. Cherche page dans Buffer Pool (cache)
    │    ├─ Trouvée ? ✓ Return immédiatement
    │    └─ Pas trouvée ? ↓ Lit depuis disque
    │
    │ 2. Lecture disque si nécessaire
    │    ├─ Cherche dans index B-Tree PRIMARY
    │    ├─ Trouve la page contenant id=12345
    │    └─ Charge page dans Buffer Pool
    │
    │ 3. Extrait la ligne (name, email)
    ↓ Row data

[7] RESULT SET
    │ Formate résultat (protocole MySQL)
    ↓ TCP/IP

[8] CLIENT
    │ Reçoit résultat
    └─ Affiche : John Doe | john@example.com
```

**Temps total** : ~1-10 ms (si en cache) ou ~50-100 ms (lecture disque)

---

## Optimisations architecture

### ⚡ Points clés de performance

#### 1️⃣ **Buffer Pool dimensionnement**
```ini
# Règle d'or : 50-80% RAM serveur
innodb_buffer_pool_size = 12G  # Pour serveur 16 GB
```

#### 2️⃣ **Thread Pool vs Thread per Connection**
```ini
# Activer Thread Pool (recommandé production)
thread_handling = pool-of-threads
thread_pool_size = 16  # Nombre de CPU cores
```

#### 3️⃣ **I/O Configuration**
```ini
# Augmenter pour SSD/NVMe
innodb_read_io_threads = 8
innodb_write_io_threads = 8
innodb_io_capacity = 2000      # IOPS disque
innodb_io_capacity_max = 4000
```

#### 4️⃣ **Logs et durabilité**
```ini
# Équilibre performance / sécurité
innodb_flush_log_at_trx_commit = 1  # Max sécurité (défaut)
                                 = 2  # Flush chaque seconde (compromis)
                                 = 0  # Max perf (risque perte 1s)
```

### 🔍 Monitoring architecture

**Requêtes utiles** :
```sql
-- Voir les threads actifs
SHOW PROCESSLIST;

-- Statistiques InnoDB
SHOW ENGINE INNODB STATUS\G

-- Taille Buffer Pool
SELECT
    ROUND(@@innodb_buffer_pool_size / 1024 / 1024 / 1024, 2) AS buffer_pool_gb;

-- Utilisation Buffer Pool
SELECT
    (SELECT COUNT(*) FROM information_schema.INNODB_BUFFER_PAGE) AS total_pages,
    (SELECT COUNT(*) FROM information_schema.INNODB_BUFFER_PAGE
     WHERE DATA_SIZE > 0) AS pages_with_data;
```

---

## ✅ Points clés à retenir

- 🏗️ **Architecture en 5 couches** : Clients → Connexion → SQL Processing → Storage Engines → Physical Storage
- 🔌 **Clients** communiquent via protocole MySQL/MariaDB (port 3306)
- 🔐 **Connexion** : Thread per connection ou Thread Pool (préféré)
- 🧠 **Parser** : Analyse syntaxe et valide la requête SQL
- 🎯 **Optimizer** : Choisit le meilleur plan d'exécution (index, jointures)
- 🎛️ **Architecture Pluggable** : Multiple storage engines (InnoDB, Aria, ColumnStore, Vector)
- 💾 **InnoDB** : Moteur par défaut, ACID, transactions, clés étrangères
- 🧠 **Buffer Pool** : Cache mémoire crucial (50-80% RAM serveur)
- 📊 **Pages 16 KB** : Unité de stockage InnoDB
- 🌳 **B-Tree Index** : Structure équilibrée pour recherches rapides
- 📝 **Redo/Undo Logs** : Garantissent durabilité et rollback
- 🔄 **Threads background** : I/O, purge, log, page cleaner
- ⚡ **Performance** : Dimensionner Buffer Pool, activer Thread Pool, optimiser I/O

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [InnoDB Storage Engine](https://mariadb.com/kb/en/innodb/)
- [MariaDB Architecture](https://mariadb.com/kb/en/mariadb-server-architecture/)
- [Storage Engines](https://mariadb.com/kb/en/storage-engines/)

### 📚 Guides techniques
- [InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)
- [Thread Pool](https://mariadb.com/kb/en/thread-pool/)
- [InnoDB Page Structure](https://mariadb.com/kb/en/innodb-page-structure/)

### 🎥 Vidéos
- [MariaDB Internals](https://www.youtube.com/results?search_query=mariadb+internals)
- [InnoDB Architecture Deep Dive](https://www.youtube.com/results?search_query=innodb+architecture)

### 📰 Articles recommandés
- [How InnoDB Works](https://blog.mariadb.org/tag/innodb/)
- [Understanding Query Optimization](https://mariadb.com/kb/en/optimization/)

---

## ➡️ Section suivante

**[1.5 - Politique de versions : LTS vs Rolling releases](./05-politique-versions-lts-rolling.md)**

Maintenant que vous comprenez l'architecture interne de MariaDB, découvrons dans la section suivante la **politique de versions** de MariaDB : comment sont organisées les releases, qu'est-ce qu'une version **LTS (Long Term Support)**, et comment choisir la bonne version pour votre projet. Cette compréhension est essentielle pour planifier vos déploiements et mises à jour !

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.4*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Politique de versions : LTS (11.4, 11.8) vs Rolling releases](/01-introduction-fondamentaux/05-politique-versions-lts-rolling.md) 🔄

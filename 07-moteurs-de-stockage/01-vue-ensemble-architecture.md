🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1 Vue d'ensemble : Architecture Pluggable Storage Engine

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Architecture SGBD relationnels, concepts de systèmes d'exploitation, C/C++ (notions)

> **Public cible** : DBA, Architectes de bases de données, Développeurs de moteurs

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture modulaire en couches de MariaDB
- Maîtriser le concept de Pluggable Storage Engine et ses avantages
- Analyser le cycle de vie complet d'une requête SQL à travers les couches
- Identifier les responsabilités du SQL Layer vs Storage Engine Layer
- Comprendre l'interface Handler API et ses méthodes principales
- Évaluer les compromis architecturaux de cette approche
- Comparer avec d'autres architectures SGBD (PostgreSQL, Oracle, SQL Server)

---

## Introduction

L'architecture **Pluggable Storage Engine** est une caractéristique unique et fondamentale de MariaDB (héritée de MySQL). Elle représente un choix architectural majeur qui distingue MariaDB de la plupart des autres SGBD relationnels.

### Qu'est-ce qu'un Storage Engine ?

Un **Storage Engine** (moteur de stockage) est le composant logiciel responsable de :

1. **Gestion physique des données**
   - Organisation des fichiers sur disque
   - Format de stockage des enregistrements
   - Structures d'index (B-Tree, Hash, etc.)
   - Gestion de l'espace disque

2. **Gestion des accès concurrents**
   - Stratégie de verrouillage (table-level, row-level)
   - Isolation des transactions
   - Gestion des deadlocks

3. **Fiabilité et récupération**
   - Journalisation (WAL, logs)
   - Crash recovery
   - Mécanismes de sauvegarde

4. **Optimisations de performance**
   - Cache et buffers mémoire
   - Compression
   - Stratégies de lecture/écriture

### Principe du "Pluggable"

```
Architecture traditionnelle (PostgreSQL, Oracle) :
┌─────────────────────────────────────┐
│         SQL Layer                   │
│  (Parser, Optimizer, Executor)      │
└─────────────────────────────────────┘
               ↓
┌─────────────────────────────────────┐
│    Storage Layer (monolithique)     │
│    ✗ Un seul moteur pour toutes     │
│      les tables                     │
└─────────────────────────────────────┘

Architecture MariaDB (Pluggable) :
┌─────────────────────────────────────┐
│         SQL Layer                   │
│  (Parser, Optimizer, Executor)      │
└─────────────────────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      Handler API (interface)        │
└─────────────────────────────────────┘
        ↓       ↓       ↓       ↓
   ┌────────┬────────┬────────┬────────┐
   │InnoDB  │Aria    │Column- │Vector  │
   │        │        │Store   │/HNSW   │
   └────────┴────────┴────────┴────────┘
   ✓ Moteur différent par table
```

**Avantage clé** : Chaque table peut utiliser le moteur le plus adapté à ses besoins spécifiques, sans compromis global.

---

## Architecture en couches de MariaDB

### Vue d'ensemble des couches

MariaDB est structuré en quatre couches principales :

```
┌──────────────────────────────────────────────────────────────┐
│                    COUCHE 1 : CLIENT                         │
│  • Applications (PHP, Python, Java, etc.)                    │
│  • Protocoles : MySQL Protocol, JDBC, ODBC                   │
│  • Connection pooling, load balancing                        │
└──────────────────────────────────────────────────────────────┘
                            ↕ TCP/IP, Unix Socket
┌──────────────────────────────────────────────────────────────┐
│              COUCHE 2 : SQL LAYER (Server Layer)             │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Connection Manager                                    │   │
│  │  • Authentification (PAM, ed25519, LDAP, etc.)        │   │
│  │  • Gestion des privilèges                             │   │
│  │  • Thread per connection / Thread Pool                │   │
│  └───────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Query Parser                                          │   │
│  │  • Analyse lexicale et syntaxique                     │   │
│  │  • Validation SQL                                     │   │
│  │  • Construction de l'arbre syntaxique                 │   │
│  └───────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Query Optimizer                                       │   │
│  │  • Réécriture de requêtes                             │   │
│  │  • Calcul des coûts (Cost-based optimization)         │   │
│  │  • Choix du plan d'exécution optimal                  │   │
│  │  • Statistiques des tables et index                   │   │
│  └───────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Query Executor                                        │   │
│  │  • Exécution du plan                                  │   │
│  │  • Gestion des jointures                              │   │
│  │  • Agrégations                                        │   │
│  │  • Sorting, grouping                                  │   │
│  └───────────────────────────────────────────────────────┘   │
│                         ↓                                     │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Caches et Buffers                                     │   │
│  │  • Table cache                                        │   │
│  │  • Table definition cache                             │   │
│  │  • Thread cache                                       │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                            ↕ Handler API
┌──────────────────────────────────────────────────────────────┐
│            COUCHE 3 : STORAGE ENGINE API (Handler)           │
│                                                              │
│  Interface standardisée avec méthodes :                      │
│  • open(), close()                                           │
│  • index_read(), index_next()                                │
│  • rnd_init(), rnd_next() (scan séquentiel)                  │
│  • write_row(), update_row(), delete_row()                   │
│  • start_stmt(), external_lock() (transactions)              │
│  • info() (statistiques)                                     │
└──────────────────────────────────────────────────────────────┘
                            ↕
┌──────────────────────────────────────────────────────────────┐
│           COUCHE 4 : STORAGE ENGINES (Implémentations)       │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ InnoDB   │  │ MyISAM   │  │ Aria     │  │ Memory   │      │
│  │          │  │          │  │          │  │          │      │
│  │ •ACID    │  │ •No trx  │  │ •Crash   │  │ •RAM     │      │
│  │ •MVCC    │  │ •Table   │  │  safe    │  │ •Hash    │      │
│  │ •FK      │  │  lock    │  │ •No FK   │  │ •Temp    │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ColumnS-  │  │ S3       │  │ Vector/  │  │ Spider   │      │
│  │ tore     │  │          │  │ HNSW     │  │          │      │
│  │          │  │          │  │ 🆕       │  │          │      │
│  │ •Columnar│  │ •Cloud   │  │ •AI/RAG  │  │ •Sharding│      │
│  │ •OLAP    │  │ •Archive │  │ •ANN     │  │ •Proxy   │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
└──────────────────────────────────────────────────────────────┘
                            ↕
┌──────────────────────────────────────────────────────────────┐
│                  SYSTÈME DE FICHIERS / DISQUE                │
│  • .ibd files (InnoDB)                                       │
│  • .MYI, .MYD files (MyISAM)                                 │
│  • .MAI, .MAD files (Aria)                                   │
│  • Binary logs, Redo logs, Undo logs                         │
└──────────────────────────────────────────────────────────────┘
```

### Responsabilités par couche

#### Couche 1 : Client Layer
- **Rôle** : Établir et gérer les connexions
- **Responsabilités** :
  - Implémentation du protocole MySQL
  - Gestion du pool de connexions
  - Retry logic, failover
  - Parsing des result sets

#### Couche 2 : SQL Layer
- **Rôle** : Intelligence SQL et orchestration
- **Responsabilités** :
  - ✅ Parsing et validation SQL
  - ✅ Optimisation des requêtes (indépendante du moteur)
  - ✅ Gestion des privilèges et sécurité
  - ✅ Gestion des transactions (coordination)
  - ✅ Exécution des jointures, tris, agrégations
  - ✅ Gestion des vues, procédures stockées, triggers
  - ❌ **N'accède JAMAIS directement aux données**

#### Couche 3 : Storage Engine API
- **Rôle** : Interface de découplage
- **Responsabilités** :
  - Définition des méthodes standardisées
  - Gestion des appels polymorphiques
  - Abstraction des spécificités moteur
  - Enregistrement et découverte des moteurs

#### Couche 4 : Storage Engines
- **Rôle** : Gestion physique des données
- **Responsabilités** :
  - ✅ Lecture/écriture physique des données
  - ✅ Gestion des index et structures de données
  - ✅ Verrouillage et concurrence
  - ✅ Transactions ACID (si supporté)
  - ✅ Cache et buffers propres au moteur
  - ✅ Crash recovery
  - ❌ **Ne comprend PAS le SQL**

💡 **Principe clé** : Le SQL Layer ne sait rien de la façon dont les données sont stockées physiquement. Le Storage Engine ne sait rien du SQL.

---

## L'interface Handler API

### Concept et design

L'**Handler API** est une abstraction C++ qui définit un contrat entre le SQL Layer et les Storage Engines. Tous les moteurs implémentent cette interface.

```cpp
// Simplifié pour illustration
class handler {
public:
    // Lifecycle
    virtual int open(const char *name, int mode, int test_if_locked) = 0;
    virtual int close(void) = 0;

    // Lecture séquentielle (table scan)
    virtual int rnd_init(bool scan) = 0;
    virtual int rnd_next(uchar *buf) = 0;
    virtual int rnd_end() = 0;

    // Lecture via index
    virtual int index_init(uint index_number, bool sorted) = 0;
    virtual int index_read(uchar *buf, const uchar *key,
                          uint key_len, enum ha_rkey_function find_flag) = 0;
    virtual int index_next(uchar *buf) = 0;
    virtual int index_prev(uchar *buf) = 0;
    virtual int index_end() = 0;

    // Écriture
    virtual int write_row(uchar *buf) = 0;
    virtual int update_row(const uchar *old_data, uchar *new_data) = 0;
    virtual int delete_row(const uchar *buf) = 0;

    // Transactions
    virtual int start_stmt(THD *thd, thr_lock_type lock_type) = 0;
    virtual int external_lock(THD *thd, int lock_type) = 0;
    virtual int commit(THD *thd, bool all) = 0;
    virtual int rollback(THD *thd, bool all) = 0;

    // Informations
    virtual int info(uint flag) = 0;
    virtual ha_rows records_in_range(uint inx, key_range *min_key,
                                     key_range *max_key) = 0;

    // Capacités du moteur
    virtual ulong table_flags() const = 0;
    virtual ulong index_flags(uint idx, uint part, bool all_parts) const = 0;
};
```

### Principales méthodes de l'API

#### 1. Méthodes de lifecycle

```cpp
// Ouvrir une table
int open(const char *name, int mode, int test_if_locked);
// Paramètres :
//   - name : Nom de la table (ex: "mydb/users")
//   - mode : O_RDONLY, O_RDWR
//   - test_if_locked : Vérifier si table verrouillée
// Retour : 0 = succès, != 0 = erreur

// Fermer une table
int close(void);
```

**Exemple d'utilisation** :
```sql
-- Quand on exécute :
SELECT * FROM users;

-- Le SQL Layer appelle :
handler->open("mydb/users", O_RDONLY, 0);
-- ... lecture des données ...
handler->close();
```

#### 2. Méthodes de lecture séquentielle (Table Scan)

```cpp
// Initialiser un scan complet de table
int rnd_init(bool scan);

// Lire la prochaine ligne (ordre physique)
int rnd_next(uchar *buf);
// Retour :
//   0 = succès (ligne dans buf)
//   HA_ERR_END_OF_FILE = fin de table
//   autre = erreur

// Terminer le scan
int rnd_end();
```

**Exemple** :
```sql
-- SELECT * FROM users; (sans WHERE, sans index)

handler->rnd_init(true);
while (handler->rnd_next(buffer) == 0) {
    // Traiter la ligne
}
handler->rnd_end();
```

#### 3. Méthodes de lecture via index

```cpp
// Initialiser la lecture par index
int index_init(uint index_number, bool sorted);

// Lire une ligne via index (seek)
int index_read(uchar *buf, const uchar *key,
               uint key_len, enum ha_rkey_function find_flag);
// find_flag :
//   HA_READ_KEY_EXACT : Recherche exacte (=)
//   HA_READ_KEY_OR_NEXT : >= (range)
//   HA_READ_PREFIX : LIKE 'abc%'

// Lire la ligne suivante (index order)
int index_next(uchar *buf);

// Lire la ligne précédente
int index_prev(uchar *buf);

// Terminer la lecture par index
int index_end();
```

**Exemple** :
```sql
-- SELECT * FROM users WHERE id = 42;

handler->index_init(0, false);  // Index primaire (id)
handler->index_read(buffer, &key_42, sizeof(key_42), HA_READ_KEY_EXACT);
if (success) {
    // Ligne trouvée dans buffer
}
handler->index_end();
```

#### 4. Méthodes d'écriture

```cpp
// Insérer une ligne
int write_row(uchar *buf);
// buf contient la ligne à insérer (format interne)

// Mettre à jour une ligne
int update_row(const uchar *old_data, uchar *new_data);
// old_data : Données actuelles
// new_data : Nouvelles données

// Supprimer une ligne
int delete_row(const uchar *buf);
// buf : Ligne à supprimer
```

**Exemple** :
```sql
-- INSERT INTO users (id, name) VALUES (1, 'Alice');

// SQL Layer prépare le buffer avec id=1, name='Alice'
handler->start_stmt(thd, TL_WRITE);
handler->write_row(buffer);
handler->commit(thd, false);
```

#### 5. Méthodes transactionnelles

```cpp
// Début de statement (transaction implicite ou explicite)
int start_stmt(THD *thd, thr_lock_type lock_type);

// Verrous externes (table-level)
int external_lock(THD *thd, int lock_type);
// lock_type : F_UNLCK, F_RDLCK, F_WRLCK

// Valider la transaction
int commit(THD *thd, bool all);
// all : true = commit complet, false = savepoint

// Annuler la transaction
int rollback(THD *thd, bool all);
```

**Exemple** :
```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Séquence d'appels :
handler1->start_stmt(thd, TL_WRITE);
handler1->external_lock(thd, F_WRLCK);
handler1->index_read(...);  // Trouver id=1
handler1->update_row(old_buf, new_buf);

handler2->start_stmt(thd, TL_WRITE);
handler2->external_lock(thd, F_WRLCK);
handler2->index_read(...);  // Trouver id=2
handler2->update_row(old_buf, new_buf);

// COMMIT déclenche :
handler1->commit(thd, true);
handler2->commit(thd, true);
handler1->external_lock(thd, F_UNLCK);
handler2->external_lock(thd, F_UNLCK);
```

#### 6. Méthodes d'information

```cpp
// Obtenir des statistiques sur la table
int info(uint flag);
// flag peut demander :
//   HA_STATUS_CONST : Nombre de lignes, taille
//   HA_STATUS_VARIABLE : Statistiques temps réel
//   HA_STATUS_ERRKEY : Dernière erreur de clé
//   HA_STATUS_AUTO : Valeur AUTO_INCREMENT

// Estimation du nombre de lignes dans un range
ha_rows records_in_range(uint index_number,
                         key_range *min_key,
                         key_range *max_key);
```

**Utilisation par l'optimiseur** :
```sql
-- EXPLAIN SELECT * FROM users WHERE age BETWEEN 20 AND 30;

// Optimiseur demande :
ha_rows estimate = handler->records_in_range(
    idx_age,  // Index sur 'age'
    &key_20,  // min_key
    &key_30   // max_key
);

// Utilise 'estimate' pour choisir :
// - Index scan vs Table scan
// - Ordre des jointures
```

### Flags de capacités

Chaque moteur déclare ses capacités via des flags :

```cpp
virtual ulong table_flags() const {
    return (
        HA_BINLOG_ROW_CAPABLE |      // Supporte binlog ROW format
        HA_BINLOG_STMT_CAPABLE |     // Supporte binlog STATEMENT format
        HA_CAN_INDEX_BLOBS |         // Peut indexer BLOB/TEXT
        HA_CAN_SQL_HANDLER |         // Supporte HANDLER commands
        HA_DUPLICATE_POS |           // Gère les doublons
        HA_NULL_IN_KEY |             // Supporte NULL dans les clés
        HA_REQUIRE_PRIMARY_KEY |     // Nécessite une clé primaire
        HA_STATS_RECORDS_IS_EXACT |  // Statistiques exactes
        HA_TABLE_SCAN_ON_INDEX       // Scan via index primaire
    );
}
```

**Exemple de différences** :

| Flag | InnoDB | MyISAM | Aria | ColumnStore |
|------|--------|--------|------|-------------|
| `HA_REQUIRE_PRIMARY_KEY` | ❌ Non | ❌ Non | ❌ Non | ❌ Non |
| `HA_CAN_INDEX_BLOBS` | ✅ Oui | ✅ Oui | ✅ Oui | ❌ Non |
| `HA_STATS_RECORDS_IS_EXACT` | ❌ Non | ✅ Oui | ✅ Oui | ❌ Non |
| `HA_NULL_IN_KEY` | ✅ Oui | ✅ Oui | ✅ Oui | ✅ Oui |

💡 **Impact** : Ces flags permettent à l'optimiseur de générer des plans adaptés aux capacités réelles de chaque moteur.

---

## Cycle de vie d'une requête SQL

Analysons en détail le cheminement d'une requête `UPDATE` à travers toutes les couches.

### Requête exemple

```sql
UPDATE accounts
SET balance = balance - 100
WHERE account_id = 42;
```

### Étape 1 : Réception et authentification

```
Client (application PHP)
    │
    └─> TCP connection vers MariaDB:3306
            │
            ▼
    ┌─────────────────────────────────┐
    │  Connection Thread              │
    │  • Vérification authentification│
    │  • Validation privilèges        │
    │  • Allocation mémoire thread    │
    └─────────────────────────────────┘
```

**Vérifications** :
- Utilisateur autorisé à se connecter ?
- Privilège `UPDATE` sur `accounts` ?
- Ressources disponibles (max_connections) ?

### Étape 2 : Parsing SQL

```
┌─────────────────────────────────────────────┐
│         SQL Parser                          │
│                                             │
│  1. Analyse lexicale (tokenization)         │
│     "UPDATE" → TOKEN_UPDATE                 │
│     "accounts" → TOKEN_IDENTIFIER           │
│     "SET" → TOKEN_SET                       │
│     ...                                     │
│                                             │
│  2. Analyse syntaxique                      │
│     Construction de l'AST (Abstract         │
│     Syntax Tree)                            │
│                                             │
│  3. Validation sémantique                   │
│     • Table 'accounts' existe ?             │
│     • Colonne 'balance' existe ?            │
│     • Type de données compatible ?          │
│     • Expressions valides ?                 │
└─────────────────────────────────────────────┘
                  ↓
        Abstract Syntax Tree (AST)
```

**Résultat** : Arbre syntaxique représentant la structure logique de la requête.

### Étape 3 : Optimisation

```
┌─────────────────────────────────────────────┐
│       Query Optimizer                       │
│                                             │
│  1. Détermination du plan d'accès           │
│     • WHERE account_id = 42                 │
│     • Index disponibles ?                   │
│       → PRIMARY KEY (account_id) ✓          │
│                                             │
│  2. Statistiques du moteur                  │
│     handler->info(HA_STATUS_CONST);         │
│     → Estimation : 1 ligne trouvée          │
│                                             │
│  3. Calcul du coût                          │
│     • Index seek : Coût = 1 (log n)         │
│     • Table scan : Coût = 1000000 (n)       │
│     → Choix : Index seek                    │
│                                             │
│  4. Plan d'exécution optimal                │
│     1. Index read on PRIMARY (id=42)        │
│     2. Update row                           │
│     3. Commit                               │
└─────────────────────────────────────────────┘
```

💡 **Indépendance du moteur** : L'optimiseur utilise l'API (`info()`, `records_in_range()`) pour obtenir des statistiques, mais ne connaît pas les détails d'implémentation.

### Étape 4 : Exécution (SQL Layer)

```
┌─────────────────────────────────────────────┐
│        Query Executor                       │
│                                             │
│  1. Obtenir le handler de la table          │
│     handler = get_handler("accounts");      │
│     → Instance de ha_innobase (InnoDB)      │
│                                             │
│  2. Initialiser la transaction              │
│     handler->start_stmt(thd, TL_WRITE);     │
│                                             │
│  3. Acquérir les verrous                    │
│     handler->external_lock(thd, F_WRLCK);   │
│                                             │
│  4. Exécuter le plan                        │
│     → Délégation au Storage Engine          │
└─────────────────────────────────────────────┘
                  ↓
          Handler API call
```

### Étape 5 : Lecture des données (Storage Engine)

```
┌─────────────────────────────────────────────┐
│         InnoDB Handler                      │
│                                             │
│  1. Initialiser l'index                     │
│     handler->index_init(PRIMARY_KEY, false);│
│                                             │
│  2. Chercher la ligne (index seek)          │
│     handler->index_read(                    │
│         buffer,                             │
│         &key_42,                            │
│         sizeof(key_42),                     │
│         HA_READ_KEY_EXACT                   │
│     );                                      │
│                                             │
│     InnoDB internal:                        │
│     ├─> Recherche dans Buffer Pool          │
│     │   Page en cache ? → Read from RAM     │
│     │   Page absente ? → Read from disk     │
│     │                                       │
│     ├─> Traversée B-Tree index              │
│     │   Root → Branch → Leaf                │
│     │                                       │
│     └─> Placement Row Lock (X-lock)         │
│         Verrou exclusif sur la ligne        │
│                                             │
│  3. Lire la ligne dans buffer               │
│     account_id=42, balance=1000             │
└─────────────────────────────────────────────┘
```

### Étape 6 : Modification (SQL Layer + Storage Engine)

```
┌─────────────────────────────────────────────┐
│     SQL Executor (évaluation expression)    │
│                                             │
│  1. Calculer la nouvelle valeur             │
│     new_balance = old_balance - 100         │
│     new_balance = 1000 - 100 = 900          │
│                                             │
│  2. Préparer le buffer avec nouvelles       │
│     données                                 │
│     new_buffer: {account_id=42, balance=900}│
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│         InnoDB Handler                      │
│                                             │
│  3. Appel update_row()                      │
│     handler->update_row(old_buffer,         │
│                        new_buffer);         │
│                                             │
│     InnoDB internal:                        │
│     ├─> Écriture dans Undo Log              │
│     │   Sauvegarder ancienne valeur (1000)  │
│     │   Pour ROLLBACK et MVCC               │
│     │                                       │
│     ├─> Modification dans Buffer Pool       │
│     │   balance = 900                       │
│     │   Page marquée "dirty"                │
│     │                                       │
│     └─> Écriture dans Redo Log              │
│         Enregistrement : UPDATE accounts    │
│         SET balance=900 WHERE id=42         │
│         (Write-Ahead Logging)               │
└─────────────────────────────────────────────┘
```

### Étape 7 : Commit

```
┌─────────────────────────────────────────────┐
│         SQL Layer                           │
│                                             │
│  handler->commit(thd, true);                │
│                                             │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│         InnoDB Handler                      │
│                                             │
│  1. Flush Redo Log vers disque              │
│     fsync(ib_logfile0);                     │
│     → Transaction DURABLE                   │
│                                             │
│  2. Libérer les verrous                     │
│     Release X-lock sur row(account_id=42)   │
│                                             │
│  3. (Asynchrone) Flush dirty pages          │
│     Buffer Pool → Disque (.ibd file)        │
│     Peut être différé                       │
└─────────────────────────────────────────────┘
```

### Étape 8 : Réponse au client

```
┌─────────────────────────────────────────────┐
│         SQL Layer                           │
│                                             │
│  1. Construire le result set                │
│     Query OK, 1 row affected (0.01 sec)     │
│                                             │
│  2. Envoyer via MySQL Protocol              │
│     → Client PHP                            │
│                                             │
│  3. Nettoyer les ressources                 │
│     handler->index_end();                   │
│     handler->external_lock(thd, F_UNLCK);   │
│     handler->close();                       │
└─────────────────────────────────────────────┘
```

### Diagramme de séquence complet

```
Client        SQL Layer       Handler API      InnoDB          Disk
  │               │                 │             │              │
  ├─ UPDATE ─────>│                 │             │              │
  │               │                 │             │              │
  │               ├─ Parse          │             │              │
  │               ├─ Optimize       │             │              │
  │               │                 │             │              │
  │               ├─ start_stmt() ─>│             │              │
  │               │                 ├────────────>│              │
  │               │                 │             ├─ Begin trx   │
  │               │                 │<────────────┤              │
  │               │                 │             │              │
  │               ├─ index_read() ─>│             │              │
  │               │                 ├────────────>│              │
  │               │                 │             ├─ Buffer      │
  │               │                 │             │   Pool?      │
  │               │                 │             ├──────────>   │
  │               │                 │             │ Read page    │
  │               │                 │             │<─────────────┤
  │               │                 │<────────────┤              │
  │               │<────────────────┤             │              │
  │               │                 │             │              │
  │               ├─ update_row() ─>│             │              │
  │               │                 ├────────────>│              │
  │               │                 │             ├─ Undo Log    │
  │               │                 │             ├─ Modify buf  │
  │               │                 │             ├─ Redo Log    │
  │               │                 │             ├─────────────>│
  │               │                 │             │ Write WAL    │
  │               │                 │<────────────┤              │
  │               │                 │             │              │
  │               ├─ commit() ─────>│             │              │
  │               │                 ├────────────>│              │
  │               │                 │             ├─────────────>│
  │               │                 │             │ fsync log    │
  │               │                 │             │<─────────────┤
  │               │                 │<────────────┤              │
  │               │<────────────────┤             │              │
  │               │                 │             │              │
  │<─ OK (1 row) ─┤                 │             │              │
  │               │                 │             │              │
```

💡 **Observation clé** : Le SQL Layer ne sait JAMAIS si les données sont dans le Buffer Pool, sur disque, compressées, chiffrées, etc. Il appelle simplement les méthodes de l'API et le moteur gère les détails.

---

## Avantages de l'architecture Pluggable

### 1. Flexibilité par table

```sql
-- OLTP : Transactions critiques
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    total DECIMAL(10,2)
) ENGINE=InnoDB;  -- ACID, FK, row-level locking

-- OLAP : Analytics intensifs
CREATE TABLE sales_analytics (
    date DATE,
    product_id INT,
    revenue DECIMAL(10,2)
) ENGINE=ColumnStore;  -- Columnar, compression

-- Archive : Données froides
CREATE TABLE old_logs (
    id INT,
    timestamp DATETIME,
    message TEXT
) ENGINE=S3;  -- Cloud storage, read-only

-- IA/RAG : Recherche vectorielle
CREATE TABLE documents (
    id INT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536)
) ENGINE=InnoDB;  -- Vector/HNSW index support
```

**Bénéfice** : Chaque table utilise le moteur optimal pour son cas d'usage, sans compromis.

### 2. Innovation et compétition

L'architecture ouverte permet :
- **Développement de nouveaux moteurs** sans modifier le core
- **Expérimentation** (ColumnStore, Spider, CONNECT)
- **Contribution communautaire** (Aria par MariaDB Foundation)
- 🆕 **Nouveaux paradigmes** (Vector/HNSW pour IA en 11.8)

**Exemples historiques** :
- 2005 : InnoDB remplace MyISAM comme défaut
- 2010 : Aria améliore MyISAM avec crash recovery
- 2015 : ColumnStore pour analytics (acquisition InfiniDB)
- 2025 : Vector/HNSW pour recherche vectorielle IA

### 3. Migration progressive

```sql
-- Migration par étapes sans downtime
-- Phase 1 : Table mixte (partitions avec moteurs différents)
CREATE TABLE orders_migration (
    id INT,
    order_date DATE,
    data TEXT
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2023 VALUES LESS THAN (2024) ENGINE=MyISAM,  -- Legacy
    PARTITION p2024 VALUES LESS THAN (2025) ENGINE=InnoDB,  -- Migré
    PARTITION p2025 VALUES LESS THAN (2026) ENGINE=InnoDB   -- Nouveau
);

-- Phase 2 : Conversion partition par partition
ALTER TABLE orders_migration
    REORGANIZE PARTITION p2023 INTO (
        PARTITION p2023 VALUES LESS THAN (2024) ENGINE=InnoDB
    );
```

### 4. Isolation des risques

Si un moteur a un bug :
- ✅ Les autres moteurs continuent de fonctionner
- ✅ Impact limité aux tables utilisant ce moteur
- ✅ Rollback possible (ALTER TABLE ENGINE)

---

## Inconvénients et limitations

### 1. Complexité de l'optimiseur

L'optimiseur doit être **moteur-agnostique** :

```sql
-- Requête cross-engine
SELECT o.*, u.name
FROM orders o              -- InnoDB
JOIN users u ON o.user_id = u.id  -- ColumnStore
WHERE o.date > '2025-01-01';

-- Problèmes :
-- • Statistiques différentes par moteur
-- • Capacités différentes (index, locks)
-- • Pas de pushdown optimal
```

**Conséquence** : L'optimiseur ne peut pas utiliser les optimisations spécifiques à un moteur (ex: index pushdown dans ColumnStore).

### 2. Inconsistance des comportements

```sql
-- AUTO_INCREMENT avec InnoDB
INSERT INTO innodb_table (id, name) VALUES (NULL, 'Test');
-- id = 1 (continu même après ROLLBACK)

-- AUTO_INCREMENT avec MyISAM
INSERT INTO myisam_table (id, name) VALUES (NULL, 'Test');
-- id peut avoir des trous après crash
```

**Impact** : Les applications doivent être conscientes des différences.

### 3. Overhead de l'abstraction

Chaque appel Handler a un coût :
```
Direct call (monolithique) : ~10 ns
Handler API call (virtual) : ~15-20 ns (overhead 50%)
```

Pour des milliards d'appels par seconde, cela compte (mais reste négligeable vs I/O).

### 4. Limitations transactionnelles cross-engine

```sql
START TRANSACTION;
UPDATE innodb_table SET value = 10;   -- Transactionnel
UPDATE myisam_table SET value = 20;   -- NON transactionnel
ROLLBACK;
-- innodb_table : ROLLBACK OK
-- myisam_table : AUCUN EFFET (modification déjà commitée)
```

⚠️ **Piège** : Les transactions mixtes peuvent entraîner des états incohérents.

---

## Comparaison avec d'autres SGBD

| Aspect | MariaDB (Pluggable) | PostgreSQL | Oracle | SQL Server |
|--------|---------------------|------------|--------|------------|
| **Architecture** | Multi-moteurs par table | Monolithique | Monolithique | Monolithique |
| **Flexibilité** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐ |
| **Optimisation** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Complexité** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Innovation** | ⭐⭐⭐⭐⭐ (communautaire) | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

### PostgreSQL : Extensions au lieu de moteurs

PostgreSQL utilise un modèle d'**extensions** :
```sql
-- Exemple : Stockage columnar via extension
CREATE EXTENSION cstore_fdw;

-- Table avec Foreign Data Wrapper
CREATE FOREIGN TABLE analytics (
    date DATE,
    revenue DECIMAL
) SERVER cstore_server OPTIONS (compression 'pglz');
```

**Différence** : Les extensions n'ont pas accès au niveau de contrôle bas-niveau des Storage Engines MariaDB.

### Oracle : Tablespaces et stockage hiérarchique

```sql
-- Oracle : Un moteur, plusieurs tablespaces
CREATE TABLESPACE fast_data
    DATAFILE '/ssd/data01.dbf' SIZE 10G;

CREATE TABLE orders
    TABLESPACE fast_data
    COMPRESS FOR OLTP;
```

**Différence** : Optimisations au niveau stockage, mais pas de changement de moteur transactionnel.

---

## Enregistrement et découverte des moteurs

### Mécanisme de plugin

```cpp
// Déclaration d'un moteur (exemple InnoDB)
struct st_mysql_storage_engine innodb_storage_engine = {
    MYSQL_HANDLERTON_INTERFACE_VERSION
};

mysql_declare_plugin(innobase)
{
    MYSQL_STORAGE_ENGINE_PLUGIN,
    &innodb_storage_engine,
    "InnoDB",
    "Oracle Corporation",
    "Supports transactions, row-level locking, foreign keys and MVCC",
    PLUGIN_LICENSE_GPL,
    innodb_init,          // Initialisation
    NULL,                 // Dé-initialisation
    0x0101,              // Version 1.1
    innodb_status_variables,
    innodb_system_variables,
    NULL,
    0,
}
mysql_declare_plugin_end;
```

### Installation dynamique

```sql
-- Installer un moteur au runtime
INSTALL SONAME 'ha_columnstore.so';

-- Vérifier les moteurs disponibles
SHOW ENGINES;

-- Désinstaller
UNINSTALL SONAME 'ha_columnstore.so';
```

### Découverte des capacités

```sql
-- Interroger les capacités d'un moteur
SELECT
    ENGINE,
    SUPPORT,
    TRANSACTIONS,
    XA,
    SAVEPOINTS
FROM information_schema.ENGINES;

+--------------------+---------+--------------+------+-----------+
| ENGINE             | SUPPORT | TRANSACTIONS | XA   | SAVEPOINTS|
+--------------------+---------+--------------+------+-----------+
| InnoDB             | DEFAULT | YES          | YES  | YES       |
| Aria               | YES     | NO           | NO   | NO        |
| MyISAM             | YES     | NO           | NO   | NO        |
| ColumnStore        | YES     | YES          | NO   | NO        |
| MEMORY             | YES     | NO           | NO   | NO        |
+--------------------+---------+--------------+------+-----------+
```

---

## Implémentation d'un moteur custom (aperçu)

Pour implémenter un nouveau moteur, il faut :

### 1. Dériver de la classe handler

```cpp
class ha_myengine : public handler {
private:
    // Structures de données internes
    MyEngine_share *share;
    off_t current_position;

public:
    ha_myengine(handlerton *hton, TABLE_SHARE *table_arg);
    ~ha_myengine();

    // Implémenter toutes les méthodes virtuelles pures
    int open(const char *name, int mode, int test_if_locked) override;
    int close(void) override;
    int rnd_init(bool scan) override;
    int rnd_next(uchar *buf) override;
    int write_row(uchar *buf) override;
    // ... etc

    // Déclarer les capacités
    ulong table_flags() const override {
        return HA_CAN_INDEX_BLOBS | HA_NULL_IN_KEY;
    }
};
```

### 2. Créer un handlerton (descriptor)

```cpp
handlerton *myengine_hton;

int myengine_init(void *p) {
    myengine_hton = (handlerton *)p;
    myengine_hton->state = SHOW_OPTION_YES;
    myengine_hton->db_type = DB_TYPE_MYENGINE;
    myengine_hton->create = myengine_create_handler;
    myengine_hton->flags = HTON_CAN_RECREATE;
    myengine_hton->commit = myengine_commit;
    myengine_hton->rollback = myengine_rollback;

    return 0;
}
```

### 3. Enregistrer le plugin

```cpp
mysql_declare_plugin(myengine)
{
    MYSQL_STORAGE_ENGINE_PLUGIN,
    &myengine_storage_engine,
    "MYENGINE",
    "Your Name",
    "Custom storage engine for specific use case",
    PLUGIN_LICENSE_GPL,
    myengine_init,
    NULL,
    0x0001,
    NULL,
    NULL,
    NULL,
    0,
}
mysql_declare_plugin_end;
```

💡 **Exemples réels** : CONNECT (accès données externes), Spider (sharding), S3 (cloud storage).

---

## ✅ Points clés à retenir

1. **Architecture en couches** : MariaDB sépare strictement le SQL Layer (logique) du Storage Engine Layer (physique).

2. **Handler API** : Interface standardisée définissant un contrat entre les couches. Tous les moteurs l'implémentent.

3. **Flexibilité par table** : Chaque table peut utiliser un moteur différent selon ses besoins (OLTP, OLAP, archive, IA).

4. **Indépendance du SQL** : Le SQL Layer ne connaît pas les détails d'implémentation des moteurs. Il appelle des méthodes abstraites.

5. **Cycle de vie d'une requête** : Parse → Optimize → Execute → Handler API calls → Storage Engine → Disk.

6. **Write-Ahead Logging** : Principe clé implémenté par InnoDB dans le Redo Log (durabilité sans pénalité performance).

7. **Avantages** : Innovation, migration progressive, isolation des risques, optimisation par cas d'usage.

8. **Inconvénients** : Complexité optimiseur, inconsistances possibles, overhead d'abstraction.

9. **Comparaison SGBD** : Approche unique à MariaDB/MySQL. PostgreSQL utilise les extensions, Oracle/SQL Server sont monolithiques.

10. **Extensibilité** : Possibilité de créer des moteurs custom via l'API plugin.

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Storage Engines](https://mariadb.com/kb/en/storage-engines/)
- [📖 Storage Engine API](https://mariadb.com/kb/en/storage-engine-api/)
- [📖 Writing a Custom Storage Engine](https://mariadb.com/kb/en/writing-a-storage-engine/)
- [📖 Plugin API](https://mariadb.com/kb/en/plugin-api/)

### Code source et exemples

- [GitHub MariaDB Server - Storage Engines](https://github.com/MariaDB/server/tree/11.8/storage)
- [InnoDB Handler Implementation](https://github.com/MariaDB/server/blob/11.8/storage/innobase/handler/ha_innodb.cc)
- [EXAMPLE Storage Engine](https://github.com/MariaDB/server/tree/11.8/storage/example) - Template simple

### Articles techniques

- [MySQL Internals Manual - Storage Engines](https://dev.mysql.com/doc/internals/en/storage-engines.html)
- [Understanding MySQL Handler API](https://www.percona.com/blog/understanding-mysql-handler-api/)
- [MariaDB Architecture Deep Dive](https://mariadb.org/architecture-overview/)

### Ressources académiques

- [Architecture of MySQL Storage Engines (PDF)](https://www.researchgate.net/publication/mysql-storage-engines)
- [Comparative Analysis of Database Architectures (IEEE)](https://ieeexplore.ieee.org/)

---

## ➡️ Section suivante

**[7.2 InnoDB : Le moteur par défaut](/07-moteurs-de-stockage/02-innodb.md)** : Deep dive dans InnoDB, le moteur OLTP de référence avec Buffer Pool, Redo/Undo Log, MVCC, et configuration avancée.

Puis nous explorerons :
- **7.3** : MyISAM (legacy) et migration vers InnoDB
- **7.4** : Aria (crash-safe alternative)
- **7.5** : ColumnStore (analytics OLAP)
- **7.6** : S3 (archivage cloud)
- **7.7** : Vector/HNSW (IA/RAG) 🆕

---

**📌 Mémo Architecte** : "Le SQL Layer pense en ensembles (relationnel). Le Storage Engine pense en pages et lignes (physique). L'Handler API est le pont entre ces deux mondes."

⏭️ [InnoDB : Le moteur par défaut](/07-moteurs-de-stockage/02-innodb.md)

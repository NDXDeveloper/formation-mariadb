🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5. Index et Performance

> **Niveau** : Intermédiaire
> **Durée estimée** : 8-10 heures

> **Prérequis** :
> - Compréhension des requêtes SQL (Chapitre 2 et 3)
> - Notions de base sur les structures de données
> - Expérience avec des bases de données en production (recommandé)

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :
- Comprendre le fonctionnement interne des index et leur impact sur les performances
- Choisir le type d'index approprié selon le cas d'usage (B-Tree, Hash, Full-Text, Spatial, VECTOR)
- Concevoir des stratégies d'indexation efficaces pour optimiser vos requêtes
- Analyser les plans d'exécution avec EXPLAIN et EXPLAIN ANALYZE
- Identifier et résoudre les problèmes de performance liés à l'indexation
- Exploiter les index VECTOR/HNSW pour la recherche vectorielle et l'IA 🆕

---

## Introduction

Les **index** sont l'un des mécanismes les plus puissants pour optimiser les performances d'une base de données. Un index bien conçu peut transformer une requête qui prend plusieurs secondes en une requête qui s'exécute en quelques millisecondes. À l'inverse, une mauvaise stratégie d'indexation peut dégrader les performances d'insertion et de mise à jour, tout en consommant inutilement de l'espace disque.

### Qu'est-ce qu'un index ?

Un **index de base de données** est une structure de données auxiliaire qui permet d'accélérer la recherche et la récupération de données dans une table. On peut le comparer à l'index d'un livre : plutôt que de parcourir toutes les pages pour trouver un sujet, vous consultez l'index qui vous indique directement les pages pertinentes.

```sql
-- Sans index : MariaDB doit scanner toute la table
SELECT * FROM users WHERE email = 'john@example.com';
-- Temps : 2.5 secondes sur 1 million de lignes

-- Avec index sur email : accès quasi instantané
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE email = 'john@example.com';
-- Temps : 0.003 secondes
```

### Le compromis performance

L'indexation implique toujours un **compromis** :

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Accélération drastique des SELECT | ❌ Ralentissement des INSERT/UPDATE/DELETE |
| ✅ Amélioration des JOIN et ORDER BY | ❌ Consommation d'espace disque supplémentaire |
| ✅ Optimisation des recherches | ❌ Maintenance overhead |
| ✅ Réduction de la charge CPU | ❌ Complexité de gestion |

💡 **Règle d'or** : N'indexez pas tout ! Un index doit être justifié par un gain de performance mesurable sur des requêtes fréquentes.

---

## Pourquoi l'indexation est critique

### Impact sur les performances

Sans index, MariaDB effectue un **full table scan** (parcours complet de la table), ce qui signifie qu'il doit lire chaque ligne pour trouver les résultats. La complexité est alors **O(n)** où n est le nombre de lignes.

Avec un index B-Tree (le type par défaut), la complexité devient **O(log n)**, ce qui représente une amélioration exponentielle :

| Nombre de lignes | Sans index (O(n)) | Avec index B-Tree (O(log n)) |
|------------------|-------------------|------------------------------|
| 1 000 | 1 000 lectures | ~10 lectures |
| 1 000 000 | 1 000 000 lectures | ~20 lectures |
| 100 000 000 | 100 000 000 lectures | ~27 lectures |

### Exemples concrets d'impact

**Scénario 1 : Application e-commerce**
```sql
-- Table products avec 500 000 produits
-- Recherche par catégorie sans index
SELECT * FROM products WHERE category_id = 42;
-- Résultat : 3.2 secondes (full table scan)

-- Après création d'index
CREATE INDEX idx_category ON products(category_id);
-- Résultat : 0.008 secondes (amélioration de 400x)
```

**Scénario 2 : Système de gestion d'utilisateurs**
```sql
-- Table users avec 2 millions d'utilisateurs
-- Authentification par email
SELECT id, password_hash FROM users WHERE email = 'user@domain.com';
-- Sans index : 5 secondes (inacceptable pour un login)
-- Avec index UNIQUE : 0.002 secondes
```

⚠️ **Attention** : Ces gains ne sont possibles que si l'index est **utilisé correctement** par l'optimiseur de requêtes.

---

## Vue d'ensemble des types d'index

MariaDB 11.8 LTS propose plusieurs types d'index, chacun optimisé pour des cas d'usage spécifiques :

### 1. **B-Tree** : L'index universel

Le type d'index par défaut, adapté à la majorité des cas d'usage.

```sql
CREATE INDEX idx_name ON users(last_name);
```

**Points forts** :
- Égalité : `WHERE column = value`
- Plages : `WHERE column BETWEEN x AND y`
- Préfixes : `WHERE column LIKE 'abc%'`
- Tri : `ORDER BY column`

**Structure** : Arbre équilibré permettant des recherches en O(log n).

### 2. **Hash** : Égalité pure

Optimisé pour les recherches par égalité stricte uniquement.

```sql
CREATE INDEX idx_hash ON sessions(session_id) USING HASH;
```

**Points forts** :
- Recherches par égalité ultra-rapides : O(1)
- Consommation mémoire réduite

**Limitations** :
- ❌ Pas de plages (BETWEEN, <, >)
- ❌ Pas de LIKE avec wildcards
- ❌ Pas de tri (ORDER BY)

💡 **Usage** : Principalement avec le moteur MEMORY pour des tables en RAM.

### 3. **Full-Text** : Recherche textuelle

Spécialisé pour la recherche dans du texte libre.

```sql
CREATE FULLTEXT INDEX idx_content ON articles(title, body);

-- Recherche full-text
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('MariaDB performance' IN NATURAL LANGUAGE MODE);
```

**Points forts** :
- Recherche de mots-clés dans de longs textes
- Pertinence et scoring
- Support de plusieurs langues
- Opérateurs booléens

**Cas d'usage** : Blogs, documentation, systèmes de tickets, recherche de contenu.

### 4. **Spatial** : Données géographiques

Optimisé pour les données géométriques et géographiques.

```sql
CREATE SPATIAL INDEX idx_location ON stores(coordinates);

-- Recherche de magasins dans un rayon
SELECT * FROM stores
WHERE ST_Distance_Sphere(coordinates, POINT(2.3522, 48.8566)) < 5000;
```

**Points forts** :
- Recherches géospatiales efficaces
- Support GIS complet
- Fonctions géométriques optimisées

**Cas d'usage** : Applications de cartographie, localisation de points d'intérêt, livraison.

### 5. **VECTOR (HNSW)** : Recherche vectorielle pour l'IA 🆕

**Nouveauté MariaDB 11.8 LTS** : Index pour la recherche vectorielle et les applications d'intelligence artificielle.

```sql
-- Création d'une table avec colonne VECTOR
CREATE TABLE documents (
    id INT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536) -- Vecteur 1536 dimensions (OpenAI)
);

-- Index HNSW pour recherche vectorielle rapide
CREATE INDEX idx_embedding ON documents(embedding)
USING HNSW
ALGORITHM HNSW
WITH (
    m = 16,              -- Nombre de connexions par nœud
    ef_construction = 200 -- Précision lors de la construction
);

-- Recherche de similarité
SELECT id, content,
       VEC_DISTANCE_COSINE(embedding, :query_vector) AS similarity
FROM documents
ORDER BY similarity
LIMIT 10;
```

**Points forts** :
- Recherche de similarité ultra-rapide (Approximate Nearest Neighbors)
- Support des embeddings de modèles IA (OpenAI, Claude, LLaMA)
- Optimisations SIMD (AVX2, AVX512, ARM NEON, IBM Power10)
- Intégration native avec les workflows RAG (Retrieval-Augmented Generation)

**Cas d'usage** :
- 🤖 Recherche sémantique et chatbots IA
- 🔍 Moteurs de recommandation
- 📄 Recherche de documents similaires
- 🎨 Recherche d'images par contenu
- 🔒 Détection d'anomalies et de fraude

⚠️ **Note** : Nécessite MariaDB 11.8+ compilé avec support vectoriel.

---

## Stratégies d'indexation : Les fondamentaux

### Principes directeurs

**1. Indexer les colonnes utilisées dans WHERE, JOIN, ORDER BY, GROUP BY**

```sql
-- Cette requête bénéficie d'index sur status et created_at
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY created_at DESC;

-- Index composites recommandés
CREATE INDEX idx_status_created ON orders(status, created_at);
```

**2. L'ordre des colonnes dans un index composite est crucial**

```sql
-- Index (a, b, c) peut être utilisé pour :
-- ✅ WHERE a = ?
-- ✅ WHERE a = ? AND b = ?
-- ✅ WHERE a = ? AND b = ? AND c = ?
-- ❌ WHERE b = ? (ne peut pas utiliser l'index)
-- ❌ WHERE c = ? (ne peut pas utiliser l'index)
```

**3. Privilégier la sélectivité**

La **sélectivité** d'un index mesure sa capacité à filtrer les données :

```sql
-- Mauvaise sélectivité (colonne binaire)
CREATE INDEX idx_is_active ON users(is_active);
-- Seulement 2 valeurs possibles : 50% des lignes en moyenne

-- Bonne sélectivité (colonne unique)
CREATE INDEX idx_email ON users(email);
-- Des millions de valeurs uniques : très sélectif
```

💡 **Formule de sélectivité** :
```
Sélectivité = COUNT(DISTINCT column) / COUNT(column)
```
Plus la sélectivité est proche de 1, plus l'index est efficace.

**4. Index covering pour éviter les accès table**

Un **covering index** contient toutes les colonnes nécessaires à une requête :

```sql
-- Requête qui récupère 3 colonnes
SELECT user_id, order_date, total
FROM orders
WHERE status = 'completed';

-- Index covering incluant toutes les colonnes
CREATE INDEX idx_covering ON orders(status, user_id, order_date, total);
-- MariaDB peut répondre uniquement depuis l'index sans accéder à la table
```

### Quand NE PAS créer d'index

- ❌ Tables de petite taille (< 1000 lignes) : le scan complet est plus rapide
- ❌ Colonnes avec très faible sélectivité (ex: booléens, enums à 2-3 valeurs)
- ❌ Tables avec beaucoup d'INSERT/UPDATE et peu de SELECT
- ❌ Colonnes rarement utilisées dans les requêtes
- ❌ Duplication d'index existants

⚠️ **Surindexation** : Avoir trop d'index peut dégrader les performances des écritures et consommer de l'espace disque inutilement.

---

## Analyse des plans d'exécution

### EXPLAIN : Votre meilleur ami

La commande **EXPLAIN** est l'outil principal pour comprendre comment MariaDB exécute une requête et si vos index sont utilisés.

```sql
EXPLAIN SELECT * FROM orders
WHERE customer_id = 123
AND status = 'pending';
```

**Sortie typique** :

| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
|----|-------------|-------|------|---------------|-----|---------|-----|------|-------|
| 1 | SIMPLE | orders | ref | idx_customer,idx_status | idx_customer | 4 | const | 52 | Using where |

**Colonnes clés à analyser** :

- **type** : Type d'accès (const > eq_ref > ref > range > index > ALL)
  - `ALL` ⚠️ : Full table scan (à éviter)
  - `index` : Parcours complet d'index
  - `range` : Recherche dans une plage
  - `ref` : Recherche par égalité non-unique
  - `eq_ref` : Recherche par égalité unique (JOIN)
  - `const` : Recherche sur PRIMARY KEY ou UNIQUE (optimal)

- **possible_keys** : Index candidats
- **key** : Index réellement utilisé (NULL = pas d'index !)
- **rows** : Nombre estimé de lignes examinées
- **Extra** : Informations supplémentaires importantes

### EXPLAIN ANALYZE : Exécution réelle 🆕

**Nouveauté** disponible depuis MariaDB 10.6, améliorée en 11.8 :

```sql
EXPLAIN ANALYZE SELECT * FROM orders
WHERE customer_id = 123
AND status = 'pending';
```

**Avantages sur EXPLAIN simple** :
- Exécute réellement la requête
- Affiche les temps d'exécution réels
- Montre le nombre exact de lignes traitées (pas une estimation)
- Identifie les goulets d'étranglement

```
-> Filter: (orders.status = 'pending')  (cost=10.5 rows=52) (actual time=0.089..2.145 rows=48 loops=1)
    -> Index lookup on orders using idx_customer (customer_id=123)  (cost=10.5 rows=52) (actual time=0.082..2.089 rows=145 loops=1)
```

💡 **Interpréter les résultats** :
- `cost` : Estimation de l'optimiseur
- `rows` : Estimation du nombre de lignes
- `actual time` : Temps réel en millisecondes
- Comparer `rows` estimé vs `actual rows` pour identifier les problèmes d'estimation

---

## L'optimiseur de requêtes MariaDB

### Comment fonctionne l'optimiseur

L'**optimiseur de requêtes** est le cerveau de MariaDB. Il analyse votre requête SQL et détermine le plan d'exécution le plus efficace :

**Processus de décision** :
1. **Analyse syntaxique** : Vérification de la validité de la requête
2. **Réécriture** : Simplification et optimisation logique
3. **Estimation des coûts** : Calcul du coût de chaque plan possible
4. **Sélection du plan** : Choix du plan avec le coût le plus faible
5. **Exécution** : Application du plan sélectionné

```sql
-- L'optimiseur peut réécrire cette requête
SELECT * FROM orders WHERE order_id > 100 AND order_id < 200;

-- En requête optimisée équivalente
SELECT * FROM orders WHERE order_id BETWEEN 101 AND 199;
```

### Statistiques et estimation

L'optimiseur s'appuie sur des **statistiques** pour estimer les coûts :

```sql
-- Mettre à jour les statistiques
ANALYZE TABLE orders;

-- Voir les statistiques d'une table
SHOW INDEX FROM orders;
```

⚠️ **Statistiques obsolètes** : Un problème courant de performance. Si vos données changent significativement, les statistiques doivent être mises à jour pour que l'optimiseur prenne les bonnes décisions.

### Cost-based optimizer amélioré (11.8) 🆕

MariaDB 11.8 améliore le **cost-based optimizer** avec :

- **Prise en compte des SSD modernes** : Ajustement des coûts I/O pour refléter les performances réelles des SSD NVMe
- **Meilleure estimation des jointures** : Calculs plus précis pour les requêtes multi-tables complexes
- **Support des index vectoriels** : Intégration du coût des recherches HNSW dans les plans d'exécution

```sql
-- Variables de configuration de l'optimiseur
SET optimizer_switch = 'index_merge=on,index_merge_union=on';

-- Coûts ajustables pour SSD (11.8+)
SET GLOBAL optimizer_disk_read_cost = 0.5;  -- Plus faible pour SSD
SET GLOBAL optimizer_index_block_copy_cost = 0.05;
```

---

## Performance des index : Bonnes pratiques

### 1. Monitoring et maintenance

```sql
-- Identifier les index inutilisés (Performance Schema)
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
AND index_name != 'PRIMARY'
AND count_star = 0;

-- Vérifier la fragmentation
SHOW TABLE STATUS LIKE 'orders';

-- Défragmenter si nécessaire
OPTIMIZE TABLE orders;
```

### 2. Index progressifs et invisibles

```sql
-- Index invisible : testé avant activation
CREATE INDEX idx_test ON users(registration_date) INVISIBLE;

-- Tester l'impact
SET optimizer_switch='use_invisible_indexes=on';
-- Exécuter des tests de performance

-- Rendre visible si bénéfique
ALTER TABLE users ALTER INDEX idx_test VISIBLE;
```

### 3. Taille et ordre des index

```sql
-- Préférer les colonnes plus courtes en premier
CREATE INDEX idx_good ON users(country_code, email);  -- Mieux
CREATE INDEX idx_bad ON users(email, country_code);   -- Moins efficace

-- Index partiels pour économiser de l'espace
CREATE INDEX idx_partial ON logs(created_at)
WHERE status = 'error';  -- Indexe uniquement les erreurs
```

### 4. Monitoring continu

**Métriques clés à surveiller** :
- Ratio de hit du buffer pool
- Temps d'exécution des requêtes lentes
- Nombre de full table scans
- Utilisation CPU et I/O
- Taille des index vs taille des tables

```sql
-- Buffer pool hit ratio (doit être > 95%)
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- Requêtes sans index
SHOW STATUS LIKE 'Select_scan';
SHOW STATUS LIKE 'Select_full_join';
```

---

## Structure de ce chapitre

Ce chapitre est organisé en **10 sections** pour couvrir tous les aspects de l'indexation et de la performance :

1. **[5.1 Fonctionnement des index : Structure B-Tree](./01-fonctionnement-index.md)**
   - Architecture interne d'un B-Tree
   - Opérations de recherche, insertion, suppression
   - Pages et fill factor

2. **[5.2 Types d'index](02-types-index.md)**
   - B-Tree : Le standard (5.2.1)
   - Hash : Égalité stricte (5.2.2)
   - Full-Text : Recherche textuelle (5.2.3)
   - Spatial : Données géographiques (5.2.4)

3. **[5.3 Index VECTOR (HNSW) pour la recherche vectorielle](./03-index-vector-hnsw.md)** 🆕
   - Architecture HNSW (Hierarchical Navigable Small Worlds)
   - Configuration et paramètres (m, ef_construction, ef_search)
   - Intégration avec les modèles d'IA
   - Performances et cas d'usage

4. **[5.4 Création et gestion des index](./04-creation-gestion-index.md)**
   - Syntaxe CREATE INDEX, ALTER TABLE
   - Online DDL et opérations non-bloquantes
   - Suppression et modification d'index

5. **[5.5 Stratégies d'indexation](./05-strategies-indexation.md)**
   - Index sur colonnes filtrées (5.5.1)
   - Index sur clés étrangères (5.5.2)
   - Index pour ORDER BY et GROUP BY (5.5.3)

6. **[5.6 Index composites et ordre des colonnes](./06-index-composites.md)**
   - Règles de construction
   - Leftmost prefix rule
   - Cas d'usage optimaux

7. **[5.7 Analyse des plans d'exécution (EXPLAIN, EXPLAIN ANALYZE)](./07-analyse-plans-execution.md)**
   - Interprétation détaillée d'EXPLAIN
   - EXPLAIN ANALYZE pour l'analyse en temps réel
   - Identification des problèmes de performance

8. **[5.8 Optimisation des requêtes](./08-optimisation-requetes.md)**
   - Réécriture de requêtes
   - Éviter les anti-patterns
   - Utilisation des hints

9. **[5.9 Index covering et index-only scans](./09-index-covering.md)**
   - Concept et avantages
   - Design d'index covering
   - Impact sur la performance

10. **[5.10 Invisible indexes et Progressive indexes](./10-invisible-progressive-indexes.md)**
    - Tester des index sans impacter la production
    - Stratégies de déploiement d'index
    - Rollback sécurisé

---

## ✅ Points clés à retenir

- 🎯 **Les index sont essentiels** pour les performances, mais impliquent un compromis entre vitesse de lecture et coût d'écriture
- 🌲 **B-Tree est l'index universel** : adapté à la majorité des cas (égalité, plages, tri)
- 🔢 **Hash** : uniquement pour égalité stricte, principalement avec MEMORY engine
- 📝 **Full-Text** : spécialisé pour recherche dans du texte libre avec pertinence
- 🗺️ **Spatial** : optimisé pour les données géographiques et géométriques
- 🤖 **VECTOR (HNSW)** 🆕 : nouveau dans 11.8 pour la recherche vectorielle et l'IA
- 📊 **Sélectivité** : un index est efficace si la colonne a beaucoup de valeurs distinctes
- 🔍 **EXPLAIN/EXPLAIN ANALYZE** : outils indispensables pour comprendre et optimiser vos requêtes
- 💰 **Cost-based optimizer** : amélioré en 11.8 avec support SSD et index vectoriels
- ⚖️ **Éviter la surindexation** : trop d'index ralentit les écritures et consomme de l'espace
- 🔧 **Maintenance régulière** : ANALYZE TABLE pour mettre à jour les statistiques
- 📈 **Monitoring continu** : identifier les index inutilisés et les requêtes lentes

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Storage Engine Index Types](https://mariadb.com/kb/en/storage-engine-index-types/)
- [📖 EXPLAIN](https://mariadb.com/kb/en/explain/)
- [📖 EXPLAIN ANALYZE](https://mariadb.com/kb/en/explain-analyze/)
- [📖 B-Tree Index Structures](https://mariadb.com/kb/en/innodb-indexes/)
- [📖 Full-Text Indexes](https://mariadb.com/kb/en/fulltext-index-overview/)
- [📖 VECTOR Data Type](https://mariadb.com/kb/en/vector-data-type/) 🆕
- [📖 HNSW Index](https://mariadb.com/kb/en/vector-indexes/) 🆕
- [📖 Cost-Based Optimizer](https://mariadb.com/kb/en/optimizer/)

### Articles et guides

- [Use The Index, Luke!](https://use-the-index-luke.com/) - Guide complet sur l'indexation SQL
- [MariaDB Foundation - Performance Blog](https://mariadb.org/blog/)
- [Percona Database Performance Blog](https://www.percona.com/blog/)

### Outils recommandés

- **pt-query-digest** : Analyse des slow query logs
- **pt-index-usage** : Détection des index inutilisés
- **mysqldumpslow** : Analyse des requêtes lentes
- **Performance Schema** : Monitoring natif MariaDB

---

## ➡️ Section suivante

**[5.1 Fonctionnement des index : Structure B-Tree](./01-fonctionnement-index.md)**

Plongez dans l'architecture interne d'un index B-Tree : comment MariaDB organise les données pour des recherches ultra-rapides, comment les pages sont structurées, et pourquoi cette structure est si efficace. Comprendre le fonctionnement interne vous permettra de concevoir de meilleurs index et d'anticiper leur comportement en production.

---


⏭️ [Fonctionnement des index : Structure B-Tree](/05-index-et-performance/01-fonctionnement-index.md)

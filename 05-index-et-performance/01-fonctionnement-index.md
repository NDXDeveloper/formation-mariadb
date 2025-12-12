🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.1 Fonctionnement des index : Structure B-Tree

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures

> **Prérequis** :
> - Compréhension des structures de données de base (arbres, listes)
> - Notions de complexité algorithmique (O notation)
> - Bases SQL (CREATE TABLE, SELECT)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'architecture interne d'un index B-Tree et son fonctionnement
- Expliquer pourquoi le B-Tree est la structure de données privilégiée pour les bases de données
- Analyser comment les opérations de recherche, insertion et suppression fonctionnent dans un B-Tree
- Comprendre la notion de pages, de facteur de remplissage et de fragmentation
- Anticiper le comportement d'un index B-Tree en production et optimiser son utilisation

---

## Introduction

Le **B-Tree** (Balanced Tree - Arbre Équilibré) est la structure de données fondamentale utilisée par MariaDB pour organiser les index. Inventé en 1972 par Rudolf Bayer et Edward McCreight, le B-Tree a été spécifiquement conçu pour optimiser les accès disque dans les systèmes de bases de données.

### Pourquoi le B-Tree ?

Contrairement aux structures de données classiques comme les arbres binaires de recherche (BST) ou les arbres AVL, le B-Tree possède des caractéristiques uniques qui le rendent idéal pour les bases de données :

| Caractéristique | B-Tree | Arbre Binaire | Impact |
|-----------------|--------|---------------|--------|
| **Nombre d'enfants** | Plusieurs centaines | 2 maximum | Moins de niveaux = moins d'I/O |
| **Équilibrage** | Toujours équilibré | Peut dégénérer | Performances prévisibles |
| **Remplissage** | Haute densité | Faible densité | Meilleure utilisation mémoire |
| **Localité** | Excellente | Faible | Cache CPU efficace |
| **I/O** | Optimisé pour le disque | Optimisé pour la RAM | Adapté aux contraintes réelles |

💡 **L'insight clé** : En base de données, l'opération la plus coûteuse est la lecture sur disque. Le B-Tree minimise le nombre d'accès disque en regroupant beaucoup de clés par nœud, réduisant ainsi la hauteur de l'arbre.

```
Exemple concret :
- Arbre binaire pour 1 million de clés : ~20 niveaux = 20 I/O
- B-Tree (200 clés/nœud) pour 1 million de clés : 3-4 niveaux = 3-4 I/O

Gain : 5x moins d'accès disque !
```

---

## Architecture d'un B-Tree

### Structure générale

Un B-Tree est un **arbre équilibré** où :
- Tous les chemins de la racine aux feuilles ont la même longueur
- Chaque nœud contient plusieurs clés (pas seulement une)
- Les clés sont triées dans l'ordre croissant
- Chaque nœud a entre ⌈m/2⌉ et m enfants (sauf la racine)

```
Représentation visuelle d'un B-Tree d'ordre 5 :

                    [50|100]
                   /    |    \
                  /     |     \
        [10|20|30]  [60|75]  [120|150|180]
        /  |  |  \   /   \    /   |   |   \
     [5] [15][25][40][55][80][110][130][160][200]
```

### Propriétés fondamentales

Un B-Tree d'**ordre m** respecte ces règles :

1. **Chaque nœud contient au maximum m-1 clés**
2. **Chaque nœud (sauf la racine) contient au minimum ⌈m/2⌉-1 clés**
3. **Tous les nœuds feuilles sont au même niveau**
4. **Les clés dans un nœud sont triées**
5. **Un nœud avec k clés a k+1 enfants**

⚠️ **Note** : MariaDB utilise en réalité une variante appelée **B+Tree** où seules les feuilles contiennent les données complètes, et les nœuds internes ne contiennent que des clés de routage.

### Pages et blocs

Dans MariaDB/InnoDB, un B-Tree est organisé en **pages** :

```sql
-- Taille de page par défaut : 16 KB
SHOW VARIABLES LIKE 'innodb_page_size';
```

**Caractéristiques d'une page InnoDB** :
- Taille : 16 KB par défaut (configurable : 4KB, 8KB, 16KB, 32KB, 64KB)
- Contient : En-tête (38 bytes) + enregistrements + espace libre
- Capacité : ~100-500 enregistrements selon la taille des clés

```
Structure d'une page InnoDB :

┌─────────────────────────────────────────┐
│ FIL Header (38 bytes)                   │  Métadonnées de la page
├─────────────────────────────────────────┤
│ Index Header (36 bytes)                 │  Informations sur l'index
├─────────────────────────────────────────┤
│ FSEG Header (20 bytes)                  │  Gestion des segments
├─────────────────────────────────────────┤
│                                         │
│ Enregistrements (Records)               │  Les clés et pointeurs
│                                         │
├─────────────────────────────────────────┤
│ Espace libre                            │  Pour nouvelles insertions
├─────────────────────────────────────────┤
│ Page Directory                          │  Index interne à la page
├─────────────────────────────────────────┤
│ FIL Trailer (8 bytes)                   │  Checksum
└─────────────────────────────────────────┘
     Total : 16 KB (16384 bytes)
```

💡 **Implication pratique** : Une page entière est lue lors d'un accès disque, même pour récupérer une seule clé. C'est pourquoi regrouper plusieurs clés par page est si efficace.

---

## B-Tree vs B+Tree : La variante de MariaDB

### Différences fondamentales

MariaDB (via InnoDB) utilise un **B+Tree**, une variante optimisée du B-Tree :

| Aspect | B-Tree classique | B+Tree (MariaDB) |
|--------|------------------|------------------|
| **Données dans nœuds internes** | Oui | Non (uniquement clés) |
| **Données dans feuilles** | Oui | Oui (toutes les données) |
| **Lien entre feuilles** | Non | Oui (doubly-linked list) |
| **Parcours séquentiel** | Nécessite traversée | Direct via liste chaînée |
| **Espace nœuds internes** | Moins de clés | Plus de clés (plus efficace) |

```
B+Tree utilisé par InnoDB :

         Nœuds internes          [50|100|150]
         (clés uniquement)      /    |    |    \
                               /     |    |     \
         Feuilles            [10-45]→[50-95]→[100-145]→[150-200]
         (clés + données)    ←──────doubly-linked list──────→
```

### Avantages du B+Tree pour les bases de données

**1. Parcours séquentiel optimisé**

```sql
-- Requête de plage : très efficace avec B+Tree
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';

-- Le moteur :
-- 1. Trouve la première clé (2025-01-01) via l'arbre
-- 2. Parcourt séquentiellement les feuilles chaînées
-- 3. S'arrête à la dernière clé (2025-01-31)
```

Avec un B+Tree, le **range scan** devient une opération de liste chaînée O(k) où k est le nombre de résultats, au lieu de k recherches individuelles O(log n).

**2. Nœuds internes plus compacts**

Sans données, les nœuds internes peuvent contenir plus de clés :

```
Exemple avec clé INT (4 bytes) + pointeur (6 bytes) :
- Page 16 KB
- En-têtes : ~100 bytes
- Espace utile : ~16 000 bytes

B-Tree classique avec données (50 bytes/enregistrement) :
→ ~300 entrées par page

B+Tree sans données dans nœuds internes :
→ ~1600 entrées par page

Résultat : Arbre moins profond = moins d'I/O !
```

**3. Cachabilité améliorée**

Les nœuds internes (sans données) tiennent plus facilement en mémoire (InnoDB Buffer Pool), réduisant les I/O :

```sql
-- Vérifier le ratio de hit du buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- Innodb_buffer_pool_reads : Lectures depuis disque
-- Innodb_buffer_pool_read_requests : Total de lectures

-- Ratio optimal : > 99% (presque toutes les lectures en cache)
```

---

## Opérations sur un B-Tree

### Recherche (SELECT)

La recherche dans un B-Tree suit un chemin de la racine aux feuilles :

```sql
-- Recherche simple par égalité
SELECT * FROM users WHERE user_id = 12345;
```

**Algorithme de recherche** :

```
1. Commencer à la racine
2. Tant que pas dans une feuille :
   a. Trouver la clé k[i] où valeur_recherchée < k[i]
   b. Suivre le pointeur vers l'enfant i
3. Dans la feuille, recherche binaire pour trouver la valeur exacte
4. Retourner les données associées
```

**Exemple pas à pas pour user_id = 12345** :

```
Niveau 0 (Racine) : [10000|20000|30000]
                     → 12345 < 20000 → Branche 2

Niveau 1 :          [11000|13000|15000]
                     → 12345 > 11000 ET < 13000 → Branche 2

Niveau 2 (Feuille) : [12100|12345|12580|12890]
                     → Recherche binaire → Trouvé à position 2
                     → Retourner les données
```

**Complexité** : O(log_m n) où m est l'ordre de l'arbre et n le nombre de clés.

Avec m=200 et n=1 million :
```
log₂₀₀(1 000 000) ≈ 2.6 → 3 accès disque maximum !
```

### Insertion (INSERT)

L'insertion maintient l'équilibre de l'arbre en divisant les nœuds pleins :

```sql
-- Insertion simple
INSERT INTO products (product_id, name, price)
VALUES (5025, 'Widget Pro', 29.99);
```

**Algorithme d'insertion** :

```
1. Rechercher la feuille appropriée pour la nouvelle clé
2. Si la feuille a de l'espace :
   → Insérer la clé à la position triée
3. Si la feuille est pleine (m-1 clés) :
   → Split : diviser en 2 feuilles
   → Promouvoir la clé médiane au parent
   → Récursivement, remonter les splits si nécessaire
```

**Exemple de split** :

```
Avant insertion de 55 (ordre 5, max 4 clés) :

Parent :           [50]
                    |
Feuille :      [40|45|50|60]  ← Pleine !

Après insertion et split :

Parent :           [50|55]       ← Clé médiane promue
                   /    \
Feuilles :    [40|45] [55|60]   ← Divisée en 2
```

💡 **Fill Factor** : Pour limiter les splits fréquents, InnoDB ne remplit pas complètement les pages lors de la création initiale d'index (typiquement 93%).

### Suppression (DELETE)

La suppression peut nécessiter des fusions de nœuds pour maintenir l'équilibre :

```sql
-- Suppression
DELETE FROM products WHERE product_id = 5025;
```

**Algorithme de suppression** :

```
1. Rechercher et supprimer la clé dans la feuille
2. Si la feuille a assez de clés (≥ ⌈m/2⌉-1) :
   → Terminé
3. Si la feuille a trop peu de clés :
   → Emprunter une clé du frère adjacent (redistribution)
   → OU fusionner avec un frère (merge)
   → Récursivement, remonter les merges si nécessaire
```

**Exemple de fusion** :

```
Avant suppression de 45 :

Parent :           [50|60]
                   /   |   \
Feuilles :    [40|45] [55] [65|70]

Après suppression et fusion (trop peu de clés dans feuille 1) :

Parent :           [60]
                   /    \
Feuilles :    [40|55]  [65|70]  ← Fusion des 2 premières
```

⚠️ **Fragmentation** : Des suppressions répétées peuvent créer des pages sous-remplies, dégradant les performances. Solution : `OPTIMIZE TABLE`.

---

## Facteur de remplissage (Fill Factor)

### Concept et impact

Le **fill factor** détermine combien d'espace d'une page est utilisé lors de la création ou reconstruction d'un index :

```sql
-- Exemple : Fill factor implicite lors de la création
CREATE INDEX idx_created_at ON orders(created_at);

-- InnoDB laisse ~7% d'espace libre par défaut
-- Page 16 KB → ~15 KB utilisés, ~1 KB libre
```

**Pourquoi ne pas remplir à 100% ?**

| Fill Factor | Avantages | Inconvénients |
|-------------|-----------|---------------|
| **100%** | Compacité maximale | Splits fréquents sur INSERT |
| **93% (défaut)** | Équilibre optimal | Espace gaspillé minimal |
| **70%** | Peu de splits | Beaucoup d'espace gaspillé |

### Cas d'usage du Fill Factor

**1. Index sur données append-only (logs, séries temporelles)**

```sql
-- Données toujours insérées à la fin (ordre chronologique)
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME,
    message TEXT,
    INDEX idx_timestamp (timestamp)
);

-- Ici, fill factor élevé est optimal (peu de splits)
-- Car les insertions se font toujours "à droite" de l'arbre
```

**2. Index sur données aléatoires (UUIDs, hashs)**

```sql
-- UUIDs : insertions partout dans l'arbre
CREATE TABLE sessions (
    session_id CHAR(36) PRIMARY KEY, -- UUID
    user_id INT,
    data JSON
);

-- Ici, fill factor plus faible (85-90%) pour anticiper les splits
```

💡 **Configuration InnoDB** :

```sql
-- Fill factor lors de la construction d'index (11.8+)
-- innodb_fill_factor n'est pas directement configurable
-- Mais innodb_max_purge_lag influence le comportement
SHOW VARIABLES LIKE 'innodb_fill_factor';
```

---

## Fragmentation et maintenance

### Types de fragmentation

**1. Fragmentation interne** : Espace inutilisé à l'intérieur des pages

```
Page partiellement remplie après suppressions :

┌─────────────────────────────────────┐
│ [10] [25] [__] [50] [__] [75] [__]  │  ← Trous dans la page
└─────────────────────────────────────┘
```

**2. Fragmentation externe** : Pages logiquement consécutives physiquement dispersées

```
Ordre logique :  Page1 → Page2 → Page3 → Page4
Ordre physique:  Page1 → Page4 → Page2 → Page3

→ I/O aléatoires au lieu de séquentiels
```

### Détecter la fragmentation

```sql
-- Vérifier l'état des tables
SHOW TABLE STATUS LIKE 'orders'\G

-- Indicateurs clés :
-- Data_free : Espace inutilisé (fragmentation interne)
-- Data_length : Taille totale des données
```

**Calcul du taux de fragmentation** :

```sql
SELECT
    table_name,
    data_length,
    data_free,
    ROUND(data_free / (data_length + data_free) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE table_schema = 'mydb'
AND data_length > 0
ORDER BY fragmentation_pct DESC;
```

**Interprétation** :
- < 5% : Excellent, pas d'action nécessaire
- 5-15% : Acceptable, surveiller
- 15-30% : Optimisation recommandée
- > 30% : Optimisation urgente

### Défragmentation

```sql
-- Option 1 : OPTIMIZE TABLE (reconstruction complète)
OPTIMIZE TABLE orders;

-- Équivalent à :
ALTER TABLE orders ENGINE=InnoDB;

-- Option 2 : ALTER TABLE avec ALGORITHM=INPLACE (11.8+)
ALTER TABLE orders FORCE, ALGORITHM=INPLACE;
```

⚠️ **Précautions** :
- `OPTIMIZE TABLE` verrouille la table (peut être long)
- Nécessite de l'espace disque libre (2x la taille de la table)
- À faire en maintenance programmée pour grandes tables

💡 **Alternative pour grandes tables** :

```sql
-- gh-ost ou pt-online-schema-change pour défragmentation sans downtime
pt-online-schema-change \
  --alter "ENGINE=InnoDB" \
  --execute \
  h=localhost,D=mydb,t=orders
```

---

## Pages et gestion de la mémoire

### Buffer Pool et pages

L'**InnoDB Buffer Pool** cache les pages d'index en RAM :

```sql
-- Taille du buffer pool (à configurer selon RAM disponible)
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Recommandation : 60-80% de la RAM serveur
SET GLOBAL innodb_buffer_pool_size = 8589934592; -- 8 GB
```

**Gestion LRU (Least Recently Used)** :

```
Buffer Pool divisé en 2 zones :

┌──────────────────────────────────┐
│ Young Zone (37%)                 │  Pages récemment accédées
│ ─────────────────────────────────│
│ Old Zone (63%)                   │  Pages moins accédées
└──────────────────────────────────┘

→ Les nouvelles pages entrent dans Old Zone
→ Si réaccédées rapidement → promues vers Young Zone
→ Évite de polluer le cache avec des full scans
```

### Pages et I/O

```sql
-- Statistiques I/O du buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Métriques clés :
-- Innodb_buffer_pool_reads : Lectures depuis disque (cold reads)
-- Innodb_buffer_pool_read_requests : Lectures totales (hot + cold)
-- Innodb_buffer_pool_read_ahead : Lectures anticipées
-- Innodb_buffer_pool_pages_dirty : Pages modifiées non encore écrites
```

**Calcul du hit ratio** :

```sql
SELECT
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS hit_ratio
FROM (
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_reads
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) reads,
(
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) requests;

-- Objectif : > 99%
```

### Read-Ahead : Préchargement intelligent

InnoDB précharge des pages en anticipation :

```sql
-- Configuration du read-ahead
SHOW VARIABLES LIKE 'innodb_read_ahead_threshold';

-- Valeur par défaut : 56 (sur 64 pages d'une extent)
-- Si 56 pages d'une extent sont lues séquentiellement,
-- InnoDB précharge automatiquement l'extent suivante
```

**Types de read-ahead** :

1. **Linear read-ahead** : Détecte les lectures séquentielles
2. **Random read-ahead** : Désactivé par défaut (peu efficace)

```sql
-- Désactiver random read-ahead (recommandé)
SET GLOBAL innodb_random_read_ahead = OFF;
```

---

## Hauteur de l'arbre et performance

### Calcul de la hauteur

La **hauteur d'un B-Tree** détermine directement le nombre d'I/O nécessaires pour une recherche.

**Formule** : hauteur ≈ log_m(n) où m = nombre de clés par page, n = nombre total de clés

```sql
-- Exemple concret : Table avec 10 millions de lignes
-- Clé primaire INT (4 bytes)
-- Pointeur vers ligne (6 bytes)
-- Page 16 KB

-- Calcul du nombre de clés par page :
-- Espace utile : 16 KB - 100 bytes (headers) ≈ 16 000 bytes
-- Taille par entrée : 4 + 6 = 10 bytes
-- Clés par page : 16 000 / 10 = 1600

-- Hauteur de l'arbre :
-- log₁₆₀₀(10 000 000) ≈ 2.14 → 3 niveaux

-- Donc : Maximum 3 I/O pour n'importe quelle recherche !
```

### Évolution avec la taille

| Nombre de lignes | Clés par page | Hauteur | I/O max |
|------------------|---------------|---------|---------|
| 1 000 | 1600 | 1 | 1 |
| 100 000 | 1600 | 2 | 2 |
| 10 000 000 | 1600 | 3 | 3 |
| 1 000 000 000 | 1600 | 4 | 4 |

💡 **Insight** : Même avec 1 milliard de lignes, seulement 4 I/O sont nécessaires ! C'est la puissance du B-Tree.

### Impact de la taille de clé

Plus la clé est grande, moins on peut mettre de clés par page :

```sql
-- Comparaison : INT vs VARCHAR(255)

-- INT (4 bytes) : ~1600 clés/page → 3 niveaux pour 10M lignes
-- VARCHAR(255) (255 bytes) : ~60 clés/page → 4 niveaux pour 10M lignes

-- Conclusion : Préférer des clés compactes !
```

**Recommandations** :
- ✅ Utiliser INT/BIGINT pour clés primaires
- ✅ Éviter VARCHAR(255) comme clé primaire si possible
- ✅ Envisager des clés composites intelligemment ordonnées
- ❌ Éviter TEXT/BLOB comme clé

---

## Comparaison de complexité

### Complexité algorithmique des opérations

| Opération | B-Tree | Table non indexée | Gain |
|-----------|--------|-------------------|------|
| **Recherche égalité** | O(log n) | O(n) | Exponentiel |
| **Recherche plage** | O(log n + k) | O(n) | Significatif |
| **Insertion** | O(log n) | O(1) | Perte acceptable |
| **Suppression** | O(log n) | O(n) | Gain important |
| **Parcours trié** | O(n) | O(n log n) | Évite le tri |

*n = nombre total de lignes, k = nombre de résultats*

### Exemples de performance réelle

```sql
-- Table : 5 millions de lignes

-- 1. Recherche sans index
SELECT * FROM orders WHERE customer_id = 12345;
-- Temps : 4.2 secondes (full scan)
-- I/O : 5 000 000 lectures

-- 2. Recherche avec index B-Tree
CREATE INDEX idx_customer ON orders(customer_id);
SELECT * FROM orders WHERE customer_id = 12345;
-- Temps : 0.003 secondes
-- I/O : 3 lectures (hauteur de l'arbre) + récupération données

-- 3. Range scan avec index
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';
-- Sans index : 4.5 secondes (full scan)
-- Avec index : 0.12 secondes (3 I/O arbre + scan séquentiel feuilles)
```

---

## Limitations et considérations

### Quand le B-Tree est moins efficace

**1. Faible sélectivité**

```sql
-- Colonne avec peu de valeurs distinctes
SELECT * FROM users WHERE gender = 'M';
-- Si 50% des utilisateurs sont 'M', l'index n'aide pas beaucoup
-- L'optimiseur peut préférer un full scan !
```

**2. Recherches avec wildcards au début**

```sql
-- Index inutilisable
SELECT * FROM products WHERE name LIKE '%widget%';

-- Index utilisable
SELECT * FROM products WHERE name LIKE 'widget%';
```

**3. Fonctions sur colonnes indexées**

```sql
-- Index non utilisé
SELECT * FROM orders WHERE YEAR(order_date) = 2025;

-- Index utilisé
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31';
```

### Overhead de maintenance

**Impact sur les écritures** :

```sql
-- Sans index
INSERT INTO logs (message) VALUES ('Test'); -- 1 écriture

-- Avec 3 index
INSERT INTO logs (timestamp, level, message) VALUES (NOW(), 'INFO', 'Test');
-- 1 écriture table + 3 écritures index = 4 écritures !

-- Chaque index ralentit les INSERT/UPDATE/DELETE
```

**Recommandation** : Maximum 3-5 index par table pour les tables OLTP avec beaucoup d'écritures.

---

## Cas pratiques et exemples

### Exemple 1 : Index sur clé primaire auto-incrémentée

```sql
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    order_date DATETIME,
    total DECIMAL(10,2)
);

-- Le PRIMARY KEY crée automatiquement un B-Tree clustered index
-- Les insertions sont toujours à la fin → pas de splits (optimal)
-- L'arbre grandit "par la droite"
```

**Structure après 1 million d'insertions** :

```
Nœuds internes "à gauche" : [1-500k|500k-1M]
                                    |
Feuille "active" :              [999001|999002|...|1000000]
                                 ↑ Toutes les insertions ici
```

### Exemple 2 : Index sur UUID (contre-exemple)

```sql
CREATE TABLE sessions (
    session_id CHAR(36) PRIMARY KEY, -- UUID v4 (aléatoire)
    user_id INT,
    created_at DATETIME
);

-- Problème : UUID aléatoires → insertions partout dans l'arbre
-- → Splits fréquents, fragmentation, performances dégradées

-- Solution : UUID v7 (timebased) ou BIGINT auto-incrementé
```

### Exemple 3 : Index composite optimisé

```sql
-- Requête fréquente
SELECT * FROM orders
WHERE customer_id = ?
AND status = 'pending'
ORDER BY order_date DESC
LIMIT 10;

-- Index optimal (ordre important !)
CREATE INDEX idx_optimized
ON orders(customer_id, status, order_date);

-- Pourquoi cet ordre ?
-- 1. customer_id : plus sélectif, filtre d'abord
-- 2. status : filtre secondaire
-- 3. order_date : permet le tri sans opération supplémentaire
```

---

## Monitoring et diagnostic

### Analyser l'utilisation des index

```sql
-- Statistiques d'utilisation (Performance Schema)
SELECT
    object_schema AS db,
    object_name AS table_name,
    index_name,
    count_star AS accesses
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'mydb'
AND index_name IS NOT NULL
ORDER BY count_star DESC;
```

### Visualiser la structure d'un index

```sql
-- Informations sur l'index
SHOW INDEX FROM orders;

-- Colonnes importantes :
-- - Cardinality : Nombre de valeurs uniques estimées
-- - Sub_part : Longueur de préfixe indexé (si applicable)
-- - Index_type : BTREE, HASH, FULLTEXT...
```

### Estimer la taille de l'index

```sql
SELECT
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb'
AND table_name = 'orders'
AND stat_name = 'size'
ORDER BY stat_value DESC;
```

---

## ✅ Points clés à retenir

- 🌲 **Le B-Tree est la structure optimale** pour les bases de données grâce à sa faible hauteur et son équilibrage automatique
- 📄 **MariaDB utilise un B+Tree** où les données sont uniquement dans les feuilles, chaînées pour les range scans
- 📦 **Les pages de 16 KB** sont l'unité de base : une page entière est lue lors d'un accès disque
- 🎯 **Complexité O(log n)** : Même avec des milliards de lignes, seulement 3-4 I/O maximum
- ⚖️ **Fill factor** : InnoDB laisse ~7% d'espace libre pour anticiper les insertions futures
- 🔧 **Fragmentation** : Surveillez avec `SHOW TABLE STATUS` et défragmentez avec `OPTIMIZE TABLE`
- 💾 **Buffer Pool** : Cache intelligent LRU qui maintient les pages chaudes en mémoire (hit ratio > 99%)
- 📏 **Taille de clé** : Plus la clé est compacte, plus de clés par page, moins de niveaux, moins d'I/O
- 🚀 **Auto-increment** : Optimal pour clé primaire car insertions toujours "à droite" (pas de splits)
- ⚠️ **UUID aléatoires** : Dégradent les performances (splits partout) → préférer UUID v7 ou BIGINT
- 📊 **Hauteur = Performance** : 3 niveaux pour 10M lignes, 4 niveaux pour 1 milliard
- 🎪 **Trade-off** : B-Tree accélère les lectures mais ralentit légèrement les écritures

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 InnoDB Indexes](https://mariadb.com/kb/en/innodb-indexes/)
- [📖 Storage Engine Index Types](https://mariadb.com/kb/en/storage-engine-index-types/)
- [📖 InnoDB Page Structure](https://mariadb.com/kb/en/innodb-page-structure/)
- [📖 OPTIMIZE TABLE](https://mariadb.com/kb/en/optimize-table/)
- [📖 InnoDB Buffer Pool](https://mariadb.com/kb/en/innodb-buffer-pool/)

### Ressources externes

- [Use The Index, Luke! - Anatomy of an Index](https://use-the-index-luke.com/sql/anatomy)
- [MySQL Internals Manual - InnoDB](https://dev.mysql.com/doc/internals/en/innodb.html)
- [B-tree vs B+tree (Stack Overflow)](https://stackoverflow.com/questions/870218/b-trees-b-trees-difference)

### Articles techniques

- [Understanding InnoDB clustered indexes](https://www.percona.com/blog/understanding-innodb-clustered-indexes/)
- [How InnoDB Lost its Advantage](https://www.xaprb.com/blog/2010/01/09/how-innodb-lost-its-advantage/)

---

## ➡️ Section suivante

**[5.2 Types d'index](./02-types-index.md)**

Maintenant que vous comprenez le fonctionnement interne du B-Tree, explorons les autres types d'index disponibles dans MariaDB : Hash pour l'égalité stricte, Full-Text pour la recherche textuelle, Spatial pour les données géographiques, et découvrez les index VECTOR/HNSW pour la recherche vectorielle et l'IA (nouveauté MariaDB 11.8 LTS 🆕).

---


⏭️ [Types d'index](/05-index-et-performance/02-types-index.md)

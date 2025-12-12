🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.8 Comparaison et choix du moteur approprié

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 7.1-7.7 (tous les moteurs)
> **Public cible** : DBA, Architectes de bases de données, Décideurs techniques

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comparer objectivement tous les moteurs de stockage MariaDB
- Choisir le moteur optimal selon les critères techniques et business
- Utiliser des arbres de décision pour guider le choix
- Identifier les anti-patterns et erreurs courantes
- Concevoir des architectures hybrides multi-moteurs
- Évaluer les compromis (performance, coût, maintenance)
- Planifier des migrations entre moteurs
- Prendre des décisions architecturales justifiées

---

## Introduction

Le choix du moteur de stockage est une **décision architecturale critique** qui impacte :
- **Performance** : Latence, throughput, scalabilité
- **Fiabilité** : Durabilité des données, crash recovery
- **Coût** : Stockage, calcul, opérations
- **Maintenance** : Complexité, expertise requise

> "Choisir le mauvais moteur peut dégrader les performances de 10-100× et multiplier les coûts par 5-10."

**Principe fondamental** : Il n'existe pas de moteur "meilleur" dans l'absolu. Chaque moteur excelle dans son domaine spécifique.

---

## Tableau comparatif complet

### Vue d'ensemble des 6 moteurs

| Moteur | Type | Statut | Cas d'usage principal | Performance | Complexité |
|--------|------|--------|----------------------|-------------|------------|
| **InnoDB** | Transactionnel OLTP | ✅ Production (défaut) | Applications transactionnelles | ⭐⭐⭐⭐⭐ | Moyenne |
| **MyISAM** | Non-transactionnel | ⚠️ Legacy (déprécié) | Aucun (migration vers InnoDB) | ⭐⭐⭐ | Faible |
| **Aria** | Crash-safe simple | ✅ Production (tables système) | Tables système, temporaires | ⭐⭐⭐⭐ | Faible |
| **ColumnStore** | Analytique columnar | ✅ Production (OLAP) | Data warehouse, analytics | ⭐⭐⭐⭐⭐ (OLAP) | Élevée |
| **S3** | Archivage objet | ✅ Production (cold data) | Archivage données froides | ⭐⭐ (latence élevée) | Moyenne |
| **Vector/HNSW** | Recherche vectorielle | 🆕 Production (11.8+) | IA, RAG, recherche sémantique | ⭐⭐⭐⭐ | Moyenne |

### Comparaison détaillée par critères

#### 1. Transactions et intégrité

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **Transactions ACID** | ✅ Complet | ❌ Non | ❌ Non | ⚠️ Limité | ❌ N/A | ✅ Complet (InnoDB) |
| **MVCC** | ✅ Oui | ❌ Non | ❌ Non | ❌ Non | ❌ N/A | ✅ Oui (InnoDB) |
| **Rollback** | ✅ Oui | ❌ Non | ❌ Non | ⚠️ Limité | ❌ N/A | ✅ Oui |
| **Foreign Keys** | ✅ Oui | ❌ Non | ❌ Non | ❌ Non | ❌ N/A | ✅ Oui |
| **Crash Recovery** | ✅ Auto | ❌ Corruption | ✅ Auto (WAL) | ✅ Auto | ✅ Immuable | ✅ Auto |
| **Checksums** | ⚠️ Optionnel | ❌ Non | ✅ Oui | ✅ Oui | ✅ Oui | ✅ Oui |

#### 2. Concurrence et verrouillage

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **Locking** | Row-level | Table-level | Table-level | Extent-level | Read-only | Row-level |
| **Readers bloquent writers** | ❌ Non (MVCC) | ✅ Oui | ✅ Oui | ⚠️ Partiel | ❌ N/A | ❌ Non |
| **Haute concurrence** | ✅ Excellente | ❌ Faible | ❌ Faible | ⚠️ Moyenne | ❌ N/A | ✅ Bonne |
| **Deadlocks possibles** | ✅ Oui (détectés) | ❌ N/A | ❌ N/A | ❌ Rare | ❌ N/A | ✅ Oui |

#### 3. Performance

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **SELECT point (1 row)** | ⭐⭐⭐⭐⭐<br>0.1 ms | ⭐⭐⭐⭐<br>0.2 ms | ⭐⭐⭐⭐<br>0.2 ms | ⭐⭐<br>50 ms | ⭐<br>50-100 ms | ⭐⭐⭐⭐<br>5-50 ms |
| **SELECT scan (1M rows)** | ⭐⭐⭐<br>2 sec | ⭐⭐⭐<br>1.5 sec | ⭐⭐⭐<br>1.5 sec | ⭐⭐⭐⭐⭐<br>0.5 sec | ⭐⭐<br>20-60 sec | ⭐⭐⭐<br>3 sec |
| **INSERT (1 row)** | ⭐⭐⭐⭐⭐<br>0.1 ms | ⭐⭐⭐⭐<br>0.05 ms | ⭐⭐⭐⭐<br>0.05 ms | ⭐⭐<br>1 ms | ❌ Read-only | ⭐⭐⭐⭐<br>0.2 ms |
| **INSERT batch (1M rows)** | ⭐⭐⭐⭐<br>30 sec | ⭐⭐⭐⭐⭐<br>20 sec | ⭐⭐⭐⭐⭐<br>20 sec | ⭐⭐⭐⭐⭐<br>10 sec | ❌ Read-only | ⭐⭐⭐⭐<br>40 sec |
| **UPDATE** | ⭐⭐⭐⭐⭐<br>0.2 ms | ⭐⭐⭐<br>0.5 ms | ⭐⭐⭐<br>0.5 ms | ⭐<br>Très lent | ❌ Read-only | ⭐⭐⭐⭐<br>0.3 ms |
| **DELETE** | ⭐⭐⭐⭐⭐<br>0.2 ms | ⭐⭐⭐<br>0.5 ms | ⭐⭐⭐<br>0.5 ms | ⭐<br>Très lent | ❌ Read-only | ⭐⭐⭐⭐<br>0.3 ms |
| **Agrégations (SUM, AVG)** | ⭐⭐⭐<br>5 sec | ⭐⭐⭐<br>4 sec | ⭐⭐⭐<br>4 sec | ⭐⭐⭐⭐⭐<br>0.5 sec | ⭐⭐<br>30 sec | ⭐⭐⭐<br>8 sec |

#### 4. Stockage et compression

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **Overhead stockage** | +30-50% | +10-20% | +10-20% | Négatif (compression) | Négatif (compression) | +20-40% (index HNSW) |
| **Compression** | ⚠️ ROW_FORMAT=COMPRESSED<br>2-3× | ✅ myisampack<br>5-10× | ✅ aria_pack<br>5-10× | ✅ Automatique<br>10-50× | ✅ Automatique<br>3-5× | ❌ Vecteurs non compressibles |
| **Fragmentation** | ⚠️ Possible | ✅ Fréquente | ⚠️ Possible | ❌ Rare | ❌ Aucune | ⚠️ Possible |
| **Défragmentation** | OPTIMIZE TABLE | OPTIMIZE TABLE | OPTIMIZE TABLE | Pas nécessaire | Pas nécessaire | OPTIMIZE TABLE |

#### 5. Index et recherche

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **B-Tree** | ✅ Oui | ✅ Oui | ✅ Oui | ❌ Non | ❌ Non | ✅ Oui |
| **Hash** | ⚠️ Adaptatif | ❌ Non | ❌ Non | ❌ Non | ❌ Non | ❌ Non |
| **Full-Text** | ✅ Oui (depuis 10.0) | ✅ Oui | ✅ Oui | ⚠️ Limité | ❌ Non | ❌ Non |
| **Spatial (GIS)** | ✅ Oui | ✅ Oui | ✅ Oui | ❌ Non | ❌ Non | ❌ Non |
| **HNSW (Vector)** | ❌ Non | ❌ Non | ❌ Non | ❌ Non | ❌ Non | ✅ Oui |
| **Clustering** | ✅ PK clustering | ❌ Non | ❌ Non | ⚠️ Implicite | ❌ Non | ✅ PK clustering |

#### 6. Maintenance et opérations

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **Backup à chaud** | ✅ Oui (mariabackup) | ⚠️ FLUSH TABLES | ⚠️ FLUSH TABLES | ✅ Oui | ✅ Trivial (S3) | ✅ Oui |
| **Online DDL** | ✅ Excellente | ❌ Lock table | ❌ Lock table | ⚠️ Limitée | ❌ Read-only | ✅ Bonne |
| **Réparation nécessaire** | ❌ Rare | ✅ Fréquente | ⚠️ Rare | ❌ Rare | ❌ Jamais | ❌ Rare |
| **Complexité tuning** | ⭐⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Expertise requise** | Moyenne-Élevée | Faible | Faible | Élevée | Moyenne | Moyenne-Élevée |

#### 7. Coût et scalabilité

| Critère | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|--------|--------|------|-------------|----|----|
| **Coût stockage** | 💰💰💰<br>SSD requis | 💰💰<br>HDD OK | 💰💰<br>HDD OK | 💰💰💰<br>SSD recommandé | 💰<br>Très économique | 💰💰💰💰<br>RAM + SSD |
| **Coût CPU** | 💰💰 | 💰 | 💰 | 💰💰💰 | 💰 | 💰💰💰💰 |
| **Scalabilité verticale** | ✅ Bonne | ⚠️ Limitée | ⚠️ Limitée | ✅ Excellente | ✅ Excellente | ✅ Bonne |
| **Scalabilité horizontale** | ⚠️ Complexe | ❌ Difficile | ❌ Difficile | ✅ Native (MPP) | ✅ Illimitée | ⚠️ Complexe |

---

## Arbres de décision

### Arbre de décision principal

```
                        Nouvelle table ?
                              │
                ┌─────────────┴─────────────┐
                │                           │
              OUI                          NON
                │                           │
                ↓                      Migration existante
        Quel type de charge ?              (voir section 7.9)
                │
    ┌───────────┼───────────┬───────────┬──────────┐
    │           │           │           │          │
  OLTP      OLAP/DW    Archivage     IA/ML    Read-only
    │           │           │           │       référence
    ↓           ↓           ↓           ↓           ↓
┌─────────┐ ┌─────────┐ ┌─────┐   ┌────────┐  ┌─────┐
│ InnoDB  │ │ColumnS- │ │ S3  │   │ Vector │  │ Aria│
│         │ │ tore    │ │     │   │+InnoDB │  │  ou │
│         │ └─────────┘ └─────┘   └────────┘  │ S3  │
│         │                                    └─────┘
└─────────┘
    │
    └─> Besoins spécifiques ?
           │
    ┌──────┴──────┬─────────────┬───────────┐
    │             │             │           │
Vectoriel    Full-Text    Spatial GIS   Standard
    │             │             │           │
    ↓             ↓             ↓           ↓
Vector+     InnoDB FTS    InnoDB R-Tree  InnoDB
InnoDB
```

### Arbre de décision OLTP

```
              Transaction ACID requise ?
                      │
          ┌───────────┴───────────┐
          │                       │
        OUI                      NON
          │                       │
          ↓                       ↓
     InnoDB                  Vraiment ?
                             Réfléchir !
                                 │
                        ┌────────┴────────┐
                        │                 │
                    Tables            Tables
                    système         temporaires
                        │                 │
                        ↓                 ↓
                      Aria            Aria/InnoDB

Haute concurrence requise ?
          │
        OUI → InnoDB (row-level locking)
          │
        NON → Aria acceptable (mais InnoDB recommandé)
```

### Arbre de décision Analytics

```
        Taille du dataset ?
                │
    ┌───────────┼───────────┬────────────┐
    │           │           │            │
  < 1 GB    1-100 GB   100 GB-10 TB   > 10 TB
    │           │           │            │
    ↓           ↓           ↓            ↓
InnoDB    ColumnStore  ColumnStore  ColumnStore
          (single)     (MPP 3-5      (MPP 10+
                        nœuds)        nœuds)

Fréquence d'accès ?
        │
  ┌─────┴─────┬──────────┐
  │           │          │
Quotidien  Hebdo    Rare (>1 mois)
  │           │          │
  ↓           ↓          ↓
ColumnS-  ColumnS-     S3
tore      tore sur   (archivage)
(SSD)     HDD ou
          MinIO
```

### Arbre de décision Archivage

```
        Fréquence de consultation ?
                    │
        ┌───────────┼───────────┬──────────┐
        │           │           │          │
    Jamais     Rare (>6 mois) Occasionnel Fréquent
        │           │           │          │
        ↓           ↓           ↓          ↓
    DELETE      S3 AWS      S3 MinIO   InnoDB
    (purge)     Glacier     local      ou Aria

Modification nécessaire ?
        │
  ┌─────┴─────┐
  │           │
OUI          NON
  │           │
  ↓           ↓
InnoDB       S3
(pas        (read-only
S3)          OK)

Compliance/Audit requis ?
        │
      OUI → S3 (immuable, versioning)
        │
      NON → InnoDB/Aria (modifiable)
```

### Arbre de décision IA/ML

```
        Type de données ?
                │
    ┌───────────┼───────────┐
    │           │           │
Embeddings   Données      Mix
vectoriels   structurées  (hybrid)
    │           │           │
    ↓           ↓           ↓
Vector/     InnoDB      Vector
HNSW                    +InnoDB
                        (même table)

Volume d'embeddings ?
        │
  ┌─────┴─────┬──────────┐
  │           │          │
< 100K    100K-10M    > 10M
  │           │          │
  ↓           ↓          ↓
Vector    Vector     Vector
(M=12)    (M=16)     (M=24)
simple    standard   haute perf

Latence critique ?
        │
  ┌─────┴─────┐
  │           │
< 10 ms    < 100 ms
  │           │
  ↓           ↓
Cache     Vector
appli +   pur
Vector
```

---

## Matrice de décision par cas d'usage

### Applications web (CRUD standard)

| Critère | Poids | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|-------|--------|--------|------|-------------|----|----|
| Transactions | ⭐⭐⭐⭐⭐ | 10 | 0 | 0 | 2 | 0 | 10 |
| Concurrence | ⭐⭐⭐⭐⭐ | 10 | 2 | 2 | 5 | 0 | 9 |
| Performance OLTP | ⭐⭐⭐⭐⭐ | 10 | 8 | 8 | 2 | 1 | 9 |
| Simplicité | ⭐⭐⭐ | 8 | 9 | 9 | 4 | 6 | 7 |
| Coût stockage | ⭐⭐ | 6 | 7 | 7 | 6 | 10 | 5 |
| **Score pondéré** | | **9.2** | 4.5 | 4.5 | 3.5 | 2.3 | 8.7 |

**Recommandation** : InnoDB (défaut optimal)

### Data Warehouse / BI

| Critère | Poids | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|-------|--------|--------|------|-------------|----|----|
| Performance OLAP | ⭐⭐⭐⭐⭐ | 3 | 3 | 3 | 10 | 4 | 2 |
| Compression | ⭐⭐⭐⭐ | 3 | 7 | 7 | 10 | 9 | 2 |
| Scalabilité | ⭐⭐⭐⭐⭐ | 5 | 3 | 3 | 10 | 10 | 4 |
| Agrégations | ⭐⭐⭐⭐⭐ | 4 | 4 | 4 | 10 | 4 | 3 |
| Coût | ⭐⭐⭐⭐ | 5 | 6 | 6 | 6 | 10 | 4 |
| **Score pondéré** | | **4.1** | 4.7 | 4.7 | **9.8** | 7.5 | 3.0 |

**Recommandation** : ColumnStore (optimal pour analytics)

### Archivage compliance

| Critère | Poids | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|-------|--------|--------|------|-------------|----|----|
| Coût stockage | ⭐⭐⭐⭐⭐ | 4 | 5 | 5 | 5 | 10 | 3 |
| Immuabilité | ⭐⭐⭐⭐⭐ | 3 | 3 | 3 | 5 | 10 | 3 |
| Compression | ⭐⭐⭐⭐ | 3 | 7 | 7 | 9 | 8 | 2 |
| Simplicité | ⭐⭐⭐ | 8 | 7 | 7 | 5 | 9 | 7 |
| Durabilité | ⭐⭐⭐⭐⭐ | 9 | 3 | 8 | 9 | 10 | 9 |
| **Score pondéré** | | **5.3** | 5.0 | 5.9 | 6.7 | **9.6** | 4.8 |

**Recommandation** : S3 (optimal pour archivage long terme)

### Application IA (RAG, chatbot)

| Critère | Poids | InnoDB | MyISAM | Aria | ColumnStore | S3 | Vector |
|---------|-------|--------|--------|------|-------------|----|----|
| Recherche vectorielle | ⭐⭐⭐⭐⭐ | 0 | 0 | 0 | 0 | 0 | 10 |
| Requêtes hybrides | ⭐⭐⭐⭐⭐ | 8 | 2 | 2 | 3 | 1 | 10 |
| Performance latence | ⭐⭐⭐⭐ | 10 | 8 | 8 | 3 | 2 | 8 |
| Scalabilité | ⭐⭐⭐⭐ | 7 | 3 | 3 | 9 | 9 | 7 |
| Complexité | ⭐⭐⭐ | 8 | 9 | 9 | 4 | 6 | 6 |
| **Score pondéré** | | 6.4 | 3.8 | 3.8 | 3.5 | 3.1 | **8.8** |

**Recommandation** : Vector/HNSW (seul moteur adapté)

---

## Anti-patterns : Quand NE PAS utiliser un moteur

### ❌ InnoDB - Anti-patterns

```sql
-- Anti-pattern 1 : Table de logs append-only massive
CREATE TABLE application_logs (
    log_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    timestamp DATETIME,
    message TEXT
) ENGINE=InnoDB;
-- Problème : Overhead transactionnel inutile
-- Solution : ColumnStore pour analytics, S3 pour archivage

-- Anti-pattern 2 : Table immuable historique
CREATE TABLE orders_2020 (
    order_id INT PRIMARY KEY,
    ...
) ENGINE=InnoDB;
-- Problème : Gaspillage RAM/SSD pour données jamais modifiées
-- Solution : S3 (read-only, coût 10× inférieur)

-- Anti-pattern 3 : Analytics sur milliards de lignes
SELECT region, AVG(amount) FROM huge_table GROUP BY region;
-- Problème : Scan row-based inefficace
-- Solution : ColumnStore (10-100× plus rapide)
```

### ❌ MyISAM - Anti-patterns

```sql
-- Anti-pattern 1 : TOUTE nouvelle table
CREATE TABLE * ENGINE=MyISAM;
-- Problème : Moteur déprécié, pas de transactions, crash-prone
-- Solution : N'utilisez JAMAIS MyISAM en 2025

-- Anti-pattern 2 : Application transactionnelle
CREATE TABLE bank_accounts ENGINE=MyISAM;
-- Problème : Perte de données garantie en cas de crash
-- Solution : InnoDB OBLIGATOIRE

-- Anti-pattern 3 : Haute concurrence
-- Problème : Table-level locking = goulot d'étranglement
-- Solution : InnoDB (row-level locking)
```

💡 **Règle absolue** : Ne créez AUCUNE nouvelle table MyISAM. Migrez les existantes vers InnoDB.

### ❌ Aria - Anti-patterns

```sql
-- Anti-pattern 1 : Application critique nécessitant ACID
CREATE TABLE financial_transactions ENGINE=Aria;
-- Problème : Pas de rollback, pas de transactions complètes
-- Solution : InnoDB

-- Anti-pattern 2 : Haute concurrence en écriture
-- Problème : Table-level locking comme MyISAM
-- Solution : InnoDB

-- Anti-pattern 3 : Tables applicatives générales
CREATE TABLE users ENGINE=Aria;
-- Problème : InnoDB est supérieur dans 99% des cas
-- Solution : InnoDB
```

💡 **Usage légitime Aria** : Tables système MariaDB, quelques tables temporaires. C'est tout.

### ❌ ColumnStore - Anti-patterns

```sql
-- Anti-pattern 1 : OLTP (transactions courtes)
CREATE TABLE orders ENGINE=ColumnStore;
INSERT INTO orders VALUES (...);  -- Lent
UPDATE orders SET status = 'shipped' WHERE id = 42;  -- Très lent
-- Problème : Optimisé pour scans, pas point-lookup
-- Solution : InnoDB

-- Anti-pattern 2 : Petites tables (< 1 GB)
CREATE TABLE countries (id INT, name VARCHAR(100)) ENGINE=ColumnStore;
-- Problème : Overhead non justifié
-- Solution : InnoDB

-- Anti-pattern 3 : Modifications fréquentes (UPDATE/DELETE)
-- Problème : ColumnStore reconstruit blocs à chaque modification
-- Solution : InnoDB
```

💡 **Règle** : ColumnStore = OLAP uniquement. Pour OLTP → InnoDB.

### ❌ S3 - Anti-patterns

```sql
-- Anti-pattern 1 : Données actives consultées quotidiennement
CREATE TABLE current_orders ENGINE=S3;
-- Problème : Latence 50-100 ms inacceptable pour données chaudes
-- Solution : InnoDB

-- Anti-pattern 2 : Modifications nécessaires
-- Problème : S3 est READ-ONLY
-- Solution : InnoDB/Aria intermédiaire puis conversion S3

-- Anti-pattern 3 : Petites tables (< 100 MB)
CREATE TABLE config (key VARCHAR(50), value TEXT) ENGINE=S3;
-- Problème : Overhead S3 non justifié, coût équivalent
-- Solution : InnoDB
```

💡 **Règle** : S3 = Données froides (> 6 mois sans accès) uniquement.

### ❌ Vector/HNSW - Anti-patterns

```sql
-- Anti-pattern 1 : Recherche exacte
SELECT * FROM products WHERE name = 'iPhone 15';
-- Problème : Vector = similarité, pas égalité
-- Solution : Index B-Tree classique

-- Anti-pattern 2 : Petits datasets (< 10 000 vecteurs)
-- Problème : Overhead index HNSW non justifié
-- Solution : Scan linéaire plus rapide

-- Anti-pattern 3 : Embeddings de mauvaise qualité
-- Problème : Garbage in, garbage out
-- Solution : Améliorer modèle embeddings AVANT d'optimiser BD
```

💡 **Règle** : Vector = Recherche sémantique sur embeddings de qualité uniquement.

---

## Architectures hybrides multi-moteurs

### Architecture 1 : E-commerce OLTP + Analytics

```
┌────────────────────────────────────────────────────────┐
│              Application E-commerce                    │
└────────────────────────────────────────────────────────┘
                        ↓
┌────────────────────────────────────────────────────────┐
│                  MariaDB Server                        │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  OLTP (Transactionnel) - InnoDB                  │  │
│  │  ─────────────────────────────────────────────   │  │
│  │  • orders (actif)           10M lignes           │  │
│  │  • customers                5M lignes            │  │
│  │  • products                 100K lignes          │  │
│  │  • cart_items               2M lignes            │  │
│  │                                                  │  │
│  │  → Accès temps réel (< 10 ms)                    │  │
│  │  → Transactions ACID                             │  │
│  │  → Haute concurrence                             │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Analytics (BI/Reporting) - ColumnStore          │  │
│  │  ────────────────────────────────────────────    │  │
│  │  • orders_fact              500M lignes          │  │
│  │  • customer_dimension       5M lignes            │  │
│  │  • product_dimension        100K lignes          │  │
│  │  • sales_aggregates         1M lignes            │  │
│  │                                                  │  │
│  │  → ETL quotidien depuis InnoDB                   │  │
│  │  → Requêtes analytics (GROUP BY, SUM, AVG)       │  │
│  │  → Compression 20×                               │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Archivage (Historique) - S3                     │  │
│  │  ───────────────────────────────────────────     │  │
│  │  • orders_2020              50M lignes           │  │
│  │  • orders_2021              60M lignes           │  │
│  │  • orders_2022              80M lignes           │  │
│  │                                                  │  │
│  │  → Archives > 2 ans                              │  │
│  │  → Read-only                                     │  │
│  │  → Coût 95% inférieur à InnoDB                   │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘

ETL Pipeline :
InnoDB (daily) → ColumnStore (aggregation) → S3 (yearly archival)
```

**Implémentation** :

```sql
-- Tables OLTP (InnoDB)
CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    order_date DATE,
    customer_id INT,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    INDEX idx_date (order_date),
    INDEX idx_customer (customer_id)
) ENGINE=InnoDB;

-- Tables Analytics (ColumnStore)
CREATE TABLE orders_fact (
    order_date DATE,
    customer_country VARCHAR(50),
    product_category VARCHAR(100),
    total_amount DECIMAL(10,2),
    quantity INT
) ENGINE=ColumnStore;

-- ETL quotidien
INSERT INTO orders_fact
SELECT
    o.order_date,
    c.country,
    p.category,
    o.total_amount,
    oi.quantity
FROM orders o  -- InnoDB
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date = CURDATE() - INTERVAL 1 DAY;

-- Archives (S3)
-- Chaque année : orders_YYYY (InnoDB) → orders_YYYY (S3)
```

### Architecture 2 : Application IA/ML avec RAG

```
┌────────────────────────────────────────────────────────┐
│             Application IA (Chatbot, Assistant)        │
└────────────────────────────────────────────────────────┘
                        ↓
┌────────────────────────────────────────────────────────┐
│                  MariaDB Server                        │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Données structurées - InnoDB                    │  │
│  │  ─────────────────────────────────────────────   │  │
│  │  • users                    1M lignes            │  │
│  │  • sessions                 10M lignes           │  │
│  │  • conversations            50M lignes           │  │
│  │                                                  │  │
│  │  → Gestion utilisateurs, sessions                │  │
│  │  → Transactions ACID                             │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Knowledge Base - Vector/HNSW + InnoDB           │  │
│  │  ────────────────────────────────────────────    │  │
│  │  • documents                                     │  │
│  │    - doc_id (PK)                                 │  │
│  │    - title, content (InnoDB)                     │  │
│  │    - embedding VECTOR(1536) (HNSW index)         │  │
│  │                                                  │  │
│  │  → Recherche sémantique (RAG)                    │  │
│  │  → Filtres hybrides (SQL + Vector)               │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Logs & Analytics - ColumnStore                  │  │
│  │  ───────────────────────────────────────────     │  │
│  │  • query_logs               1B lignes            │  │
│  │  • feedback                 10M lignes           │  │
│  │                                                  │  │
│  │  → Analytics usage, performance                  │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Implémentation** :

```sql
-- Table hybride InnoDB + Vector
CREATE TABLE knowledge_base (
    doc_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(500),
    content TEXT,
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Embedding vectoriel
    embedding VECTOR(1536),

    -- Index traditionnels
    INDEX idx_category (category),
    INDEX idx_created (created_at),

    -- Index HNSW
    INDEX idx_embedding ON knowledge_base(embedding)
    USING HNSW WITH (M=16, metric='cosine')
) ENGINE=InnoDB;

-- Requête hybride : Filtres SQL + Similarité vectorielle
SELECT
    doc_id,
    title,
    VEC_DISTANCE_COSINE(embedding, ?) AS relevance
FROM knowledge_base
WHERE
    category = 'technical_docs'  -- Filtre SQL
    AND created_at >= '2024-01-01'
ORDER BY relevance ASC
LIMIT 10;
```

### Architecture 3 : Data Lake multi-tiers

```
┌────────────────────────────────────────────────────────┐
│              Applications & Data Sources               │
└────────────────────────────────────────────────────────┘
                        ↓
┌────────────────────────────────────────────────────────┐
│                  MariaDB Data Lake                     │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Tier 1 : Hot Data (7 jours) - InnoDB SSD        │  │
│  │  ──────────────────────────────────────────────  │  │
│  │  • events_current           100M lignes          │  │
│  │  • metrics_current          500M lignes          │  │
│  │                                                  │  │
│  │  → Latence < 10 ms                               │  │
│  │  → Requêtes temps réel                           │  │
│  │  → Coût : $$$                                    │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓ (après 7 jours)               │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Tier 2 : Warm Data (1 an) - ColumnStore         │  │
│  │  ──────────────────────────────────────────────  │  │
│  │  • events_2024_Q4           2B lignes            │  │
│  │  • metrics_2024             10B lignes           │  │
│  │                                                  │  │
│  │  → Latence < 1 sec                               │  │
│  │  → Analytics, dashboards                         │  │
│  │  → Compression 20×                               │  │
│  │  → Coût : $$                                     │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓ (après 1 an)                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Tier 3 : Cold Data (5 ans) - S3                 │  │
│  │  ───────────────────────────────────────────     │  │
│  │  • events_2020              5B lignes            │  │
│  │  • events_2021              6B lignes            │  │
│  │  • events_2022              7B lignes            │  │
│  │  • events_2023              8B lignes            │  │
│  │                                                  │  │
│  │  → Latence < 10 sec                              │  │
│  │  → Compliance, audit                             │  │
│  │  → Coût : $                                      │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓ (après 5 ans)                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Tier 4 : Glacier (>5 ans) - AWS S3 Glacier      │  │
│  │  ───────────────────────────────────────────     │  │
│  │  • events_2015-2019         20B lignes           │  │
│  │                                                  │  │
│  │  → Retrieval : heures/jours                      │  │
│  │  → Archivage long terme                          │  │
│  │  → Coût : ¢                                      │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘

Coûts comparatifs pour 1 TB :
• Tier 1 (InnoDB SSD) : $100/mois
• Tier 2 (ColumnStore) : $20/mois (compression 5×)
• Tier 3 (S3) : $2/mois
• Tier 4 (Glacier) : $0.40/mois
```

---

## Checklist de décision

### Étape 1 : Caractérisation de la charge

```
□ Type de charge :
  □ OLTP (transactions courtes, haute concurrence)
  □ OLAP (analytics, agrégations, reporting)
  □ Hybride OLTP/OLAP
  □ Archivage (données historiques, rarement accédées)
  □ IA/ML (embeddings, recherche vectorielle)

□ Pattern d'accès :
  □ Point-lookup (SELECT WHERE id = X)
  □ Range scan (SELECT WHERE date BETWEEN X AND Y)
  □ Full table scan (SELECT COUNT(*), GROUP BY)
  □ Recherche sémantique (similarité vectorielle)

□ Fréquence d'opérations :
  □ Lecture : _____ req/sec
  □ Écriture : _____ req/sec
  □ Ratio Lecture/Écriture : _____

□ Volume de données :
  □ Actuel : _____ GB/TB
  □ Croissance : _____ GB/mois
  □ Prévision 2 ans : _____ TB
```

### Étape 2 : Exigences fonctionnelles

```
□ Transactions ACID :
  □ Obligatoire (banking, finance, commandes)
  □ Souhaitable (la plupart des applications)
  □ Pas nécessaire (logs, metrics, read-only)

□ Concurrence :
  □ Très haute (1000+ connexions simultanées)
  □ Moyenne (100-1000 connexions)
  □ Faible (< 100 connexions)

□ Durabilité :
  □ Critique (perte inacceptable)
  □ Importante (backup récent acceptable)
  □ Non critique (reconstructible)

□ Modifications :
  □ Fréquentes (UPDATE/DELETE quotidiens)
  □ Occasionnelles (corrections mensuelles)
  □ Jamais (immutable après création)
```

### Étape 3 : Contraintes techniques

```
□ Latence :
  □ < 1 ms (ultra critique)
  □ < 10 ms (temps réel)
  □ < 100 ms (interactif)
  □ < 1 sec (acceptable)
  □ > 1 sec (batch/analytics OK)

□ Infrastructure :
  □ SSD NVMe (haute performance)
  □ SSD SATA (performance moyenne)
  □ HDD (économique)
  □ Cloud S3 (archivage)

□ RAM disponible :
  □ < 8 GB (très limité)
  □ 8-32 GB (standard)
  □ 32-128 GB (confortable)
  □ > 128 GB (data warehouse)

□ Expertise équipe :
  □ DBA expert (tuning avancé OK)
  □ DBA junior (simplicité préférée)
  □ Dev sans DBA (moteur simple requis)
```

### Étape 4 : Contraintes business

```
□ Budget :
  □ Coût stockage : _____ $/mois
  □ Coût CPU/RAM : _____ $/mois
  □ Budget total : _____ $/mois

□ SLA/Disponibilité :
  □ 99.999% (5 nine - critique)
  □ 99.99% (4 nine - important)
  □ 99.9% (3 nine - standard)
  □ 99% (acceptable)

□ Compliance/Réglementation :
  □ RGPD (données personnelles UE)
  □ HIPAA (santé USA)
  □ SOX (finance USA)
  □ Autre : _____________
```

### Étape 5 : Décision finale

```
Basé sur les réponses ci-dessus :

Moteur recommandé : _____________

Justification :
□ _________________________________
□ _________________________________
□ _________________________________

Moteurs alternatifs considérés :
1. _____________ (raison rejet : _________)
2. _____________ (raison rejet : _________)

Plan de migration (si existant) :
□ Moteur actuel : _____________
□ Stratégie : _____________
□ Durée estimée : _____________
□ Risques identifiés : _____________
```

---

## Cas d'usage réels et recommandations

### Startup MVP (< 1 an, < 100 GB)

**Recommandation** : InnoDB pour tout
- ✅ Simplicité : Un seul moteur à gérer
- ✅ Flexibilité : Change facilement avec la croissance
- ✅ Performance : Suffisante pour petits volumes
- ⚠️ À réévaluer à 100 GB+ ou besoins analytics

**Configuration** :
```sql
-- Tout en InnoDB
CREATE TABLE users ENGINE=InnoDB;
CREATE TABLE orders ENGINE=InnoDB;
CREATE TABLE products ENGINE=InnoDB;
-- etc.
```

### SaaS B2B mature (5+ ans, 1-10 TB)

**Recommandation** : Architecture hybride
- InnoDB : Données actives (< 1 an), transactions
- ColumnStore : Analytics, reporting BI
- S3 : Archives (> 2 ans), compliance

**Configuration** :
```sql
-- OLTP (InnoDB)
CREATE TABLE current_data ENGINE=InnoDB;

-- OLAP (ColumnStore)
CREATE TABLE analytics_fact ENGINE=ColumnStore;

-- Archives (S3)
CREATE TABLE historical_2020 ENGINE=S3;
```

### Marketplace e-commerce (haute échelle)

**Recommandation** : Multi-moteurs optimisés
- InnoDB : Catalogue, commandes, utilisateurs
- ColumnStore : Analytics vendeurs, KPIs
- Vector/HNSW : Recherche produits sémantique
- S3 : Historique commandes > 2 ans

**Architecture** :
```sql
-- Catalogue (InnoDB avec Vector)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(500),
    description TEXT,
    price DECIMAL(10,2),

    -- Recherche sémantique
    description_embedding VECTOR(1536),
    INDEX idx_embedding USING HNSW(description_embedding)
) ENGINE=InnoDB;

-- Analytics (ColumnStore)
CREATE TABLE sales_analytics (
    sale_date DATE,
    seller_id INT,
    revenue DECIMAL(15,2)
) ENGINE=ColumnStore;
```

### Application IA générative (chatbot, assistant)

**Recommandation** : InnoDB + Vector/HNSW
- InnoDB : Users, sessions, conversations
- Vector : Knowledge base, RAG
- ColumnStore : Analytics usage (optionnel)

**Configuration** :
```sql
-- Knowledge base (Vector + InnoDB)
CREATE TABLE documents (
    doc_id INT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),
    INDEX idx_emb USING HNSW(embedding)
) ENGINE=InnoDB;
```

---

## ✅ Points clés à retenir

1. **Pas de moteur universel** : Chaque moteur excelle dans son domaine spécifique.

2. **InnoDB = défaut** : 90% des cas d'usage → InnoDB est le bon choix.

3. **Spécialisation** : ColumnStore (OLAP), S3 (archivage), Vector (IA) pour cas spécifiques.

4. **MyISAM déprécié** : Ne jamais utiliser en 2025, migrer vers InnoDB.

5. **Architecture hybride** : Combiner plusieurs moteurs = optimal pour applications complexes.

6. **Tiering des données** : Hot (InnoDB) → Warm (ColumnStore) → Cold (S3) optimise coûts.

7. **Critères de décision** : Type de charge > Volume > Latence > Coût > Complexité.

8. **Anti-patterns** : Éviter InnoDB pour OLAP pur, ColumnStore pour OLTP, S3 pour données chaudes.

9. **Compromis** : Performance vs Coût, Simplicité vs Optimisation, Flexibilité vs Spécialisation.

10. **Réévaluation** : Revoir le choix du moteur tous les 6-12 mois (croissance, nouveaux besoins).

---

## 🔗 Ressources et références

### Documentation MariaDB

- [📖 Storage Engines Overview](https://mariadb.com/kb/en/storage-engines/)
- [📖 Choosing the Right Engine](https://mariadb.com/kb/en/choosing-the-right-storage-engine/)
- [📖 ALTER TABLE ENGINE](https://mariadb.com/kb/en/alter-table/)

### Guides de décision

- [MariaDB Storage Engine Decision Tree](https://mariadb.org/storage-engine-decision/)
- [InnoDB vs ColumnStore: When to Use Each](https://mariadb.com/resources/blog/innodb-vs-columnstore/)
- [Data Tiering Best Practices](https://mariadb.com/kb/en/data-tiering/)

### Comparaisons et benchmarks

- [Storage Engine Benchmarks](https://mariadb.com/kb/en/storage-engine-benchmarks/)
- [TPC-H ColumnStore Results](https://mariadb.com/resources/datasheets/columnstore-tpch/)

---

## ➡️ Section suivante

**[7.9 Conversion entre moteurs](/07-moteurs-de-stockage/09-conversion-entre-moteurs.md)** : Stratégies détaillées, procédures, outils et exemples pour migrer d'un moteur à l'autre (InnoDB ↔ ColumnStore, InnoDB → S3, MyISAM → InnoDB, etc.).

---

**📌 Mémo DBA** : "90% des cas → InnoDB. Analytics massif → ColumnStore. Archives > 6 mois → S3. IA/RAG → Vector. MyISAM → JAMAIS. Aria → Tables système uniquement."

**🎯 Règle d'or de décision** :
1. Si OLTP → InnoDB
2. Si OLAP (> 100 GB) → ColumnStore
3. Si Archivage → S3
4. Si IA/Embeddings → Vector
5. Si doute → InnoDB (migration facile ensuite)

**💡 Architecture moderne typique** :
```
Application → InnoDB (OLTP hot data)
           ↓
    ETL quotidien
           ↓
    ColumnStore (OLAP warm data)
           ↓
    Archivage annuel
           ↓
    S3 (cold data)
```

⏭️ [Conversion entre moteurs (ALTER TABLE ENGINE)](/07-moteurs-de-stockage/09-conversion-entre-moteurs.md)

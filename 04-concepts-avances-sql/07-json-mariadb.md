🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.7 JSON dans MariaDB

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Maîtrise du SQL avancé, compréhension du format JSON

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre les avantages et limitations du JSON dans une base relationnelle
- Identifier les cas d'usage appropriés pour le stockage JSON
- Connaître les fonctionnalités JSON de MariaDB et leurs évolutions
- Maîtriser les nouveautés MariaDB 11.8 : JSON Path Expressions avancées et JSON Schema Validation
- Choisir entre modèle relationnel pur et approche hybride avec JSON
- Évaluer l'impact performance du JSON par rapport aux colonnes traditionnelles

---

## Introduction

Le support du **JSON (JavaScript Object Notation)** dans MariaDB permet de combiner la puissance du modèle relationnel avec la flexibilité des documents semi-structurés. Cette capacité, en constante évolution, offre une alternative aux bases de données NoSQL tout en conservant les garanties ACID et les jointures SQL.

### Qu'est-ce que JSON ?

JSON est un format d'échange de données léger, lisible par l'humain et facilement parsable par les machines :

```json
{
  "id": 12345,
  "nom": "Smartphone XZ",
  "prix": 599.99,
  "specifications": {
    "ecran": "6.5 pouces",
    "stockage": "128 GB",
    "couleurs": ["noir", "blanc", "bleu"]
  },
  "en_stock": true,
  "tags": ["5G", "OLED", "NFC"]
}
```

**Caractéristiques** :
- Format texte structuré
- Types de données : string, number, boolean, null, array, object
- Hiérarchique (objets imbriqués)
- Sans schéma strict (flexible)

---

## Pourquoi utiliser JSON dans une base relationnelle ?

### Avantages du JSON

**1. Flexibilité du schéma**
```sql
-- Différents produits avec des attributs variables
INSERT INTO produits (nom, details_json) VALUES
  ('Laptop', '{"processeur": "Intel i7", "ram": "16GB", "ssd": "512GB"}'),
  ('T-shirt', '{"taille": "M", "couleur": "bleu", "matiere": "coton"}'),
  ('Livre', '{"auteur": "Victor Hugo", "pages": 523, "isbn": "978-2-07-036692-4"}');
-- Pas besoin de créer des colonnes pour chaque attribut spécifique
```

**2. Évolution sans ALTER TABLE**
```sql
-- Ajouter de nouveaux attributs sans modification de schéma
UPDATE produits
SET details_json = JSON_SET(details_json, '$.garantie_ans', 2)
WHERE categorie = 'Electronique';
-- Aucun ALTER TABLE nécessaire
```

**3. Stockage de données complexes**
```sql
-- Historique d'événements, logs, métadonnées
CREATE TABLE audit_logs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    utilisateur VARCHAR(100),
    action VARCHAR(50),
    details JSON  -- Contexte variable selon l'action
);

INSERT INTO audit_logs (utilisateur, action, details) VALUES
  ('alice', 'login', '{"ip": "192.168.1.10", "user_agent": "Chrome/120.0"}'),
  ('bob', 'update_profile', '{"champs_modifies": ["email", "telephone"], "ancienne_valeur_email": "old@example.com"}'),
  ('alice', 'achat', '{"produit_id": 42, "montant": 99.99, "mode_paiement": "carte"}');
```

**4. Intégration avec APIs et microservices**
```sql
-- Stocker directement les réponses d'API externes
CREATE TABLE cache_api (
    endpoint VARCHAR(255) PRIMARY KEY,
    reponse JSON,
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Évite la transformation constante entre JSON (API) et colonnes relationnelles
```

**5. Données polymorphiques**
```sql
-- Différents types de notifications avec structures variables
CREATE TABLE notifications (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    type ENUM('email', 'sms', 'push', 'webhook'),
    payload JSON,  -- Structure dépend du type
    statut ENUM('pending', 'sent', 'failed'),
    INDEX(user_id, statut)
);
```

### Limitations et compromis

**❌ Performance des requêtes complexes**
```sql
-- ❌ LENT : Filtrer sur des valeurs JSON profondément imbriquées
SELECT * FROM produits
WHERE JSON_EXTRACT(details_json, '$.specifications.camera.megapixels') > 12;
-- Pas d'index possible sur cette expression complexe

-- ✅ RAPIDE : Colonne indexée traditionnelle
SELECT * FROM produits
WHERE megapixels > 12;  -- Index standard utilisé
```

**❌ Validation de schéma limitée (avant 11.8)**
```sql
-- Rien n'empêche d'insérer des données incohérentes
INSERT INTO produits (details_json) VALUES
  ('{"prix": "pas un nombre"}'),  -- Type incorrect
  ('{"poids": -50}'),              -- Valeur absurde
  ('{invalid json');                -- JSON invalide (rejeté par MariaDB)
```

**❌ Consommation d'espace**
```sql
-- JSON stocké comme texte = plus volumineux que types natifs
-- int : 4 bytes
-- JSON '{"id": 42}' : ~11 bytes
```

**❌ Complexité des jointures**
```sql
-- Difficile de joindre sur des valeurs JSON
-- (nécessite extraction + conversion)
SELECT p.*, c.nom AS categorie
FROM produits p
JOIN categories c ON c.id = CAST(JSON_EXTRACT(p.details_json, '$.categorie_id') AS UNSIGNED);
-- Moins performant qu'une colonne categorie_id standard
```

---

## Évolution du support JSON dans MariaDB

### Chronologie des fonctionnalités

| Version | Nouveautés JSON |
|---------|----------------|
| **10.2** (2017) | Alias JSON pour LONGTEXT, fonctions de base (JSON_EXTRACT, JSON_ARRAY, JSON_OBJECT) |
| **10.3** (2018) | JSON_TABLE (extraction vers lignes), amélioration performances |
| **10.4** (2019) | Fonctions supplémentaires (JSON_ARRAYAGG, JSON_OBJECTAGG), opérateur raccourci `->` |
| **10.5** (2020) | JSON_OVERLAPS, JSON_NORMALIZE, optimisations |
| **10.6** (2021) | JSON Schema validation (CHECK constraints), améliorations performances |
| **10.11 LTS** (2023) | Optimisations mémoire, index sur colonnes virtuelles JSON |
| **11.4 LTS** (2024) | JSON Path Expressions améliorées, meilleures performances |
| **11.8 LTS** (2025) 🆕 | **JSON Path Expressions avancées**, **JSON Schema Validation native**, optimisations SIMD |

### MariaDB 11.8 : Nouveautés JSON majeures 🆕

#### 1. JSON Path Expressions avancées

MariaDB 11.8 introduit une syntaxe JSONPath plus riche et conforme aux standards :

```sql
-- ✅ NOUVEAU : Filtres dans les chemins JSON
SELECT JSON_EXTRACT(catalogue, '$..produits[?(@.prix < 100)].nom')
FROM boutiques;
-- Extrait les noms de tous les produits avec prix < 100

-- ✅ NOUVEAU : Wildcard récursif
SELECT JSON_EXTRACT(config, '$..*.email')
FROM utilisateurs;
-- Trouve tous les champs "email" à n'importe quel niveau

-- ✅ NOUVEAU : Opérateurs de comparaison dans les filtres
SELECT JSON_EXTRACT(inventaire, '$.produits[?(@.stock > 10 && @.prix < 50)]')
FROM entrepots;
-- Produits en stock > 10 ET prix < 50
```

**Opérateurs supportés** :
- Comparaison : `==`, `!=`, `<`, `>`, `<=`, `>=`
- Logiques : `&&` (AND), `||` (OR), `!` (NOT)
- Appartenance : `in` (liste de valeurs)
- Regex : `=~` (correspondance pattern)

#### 2. JSON Schema Validation native 🆕

MariaDB 11.8 permet de valider automatiquement les documents JSON lors de l'insertion/modification :

```sql
-- Définir un schéma JSON pour validation
CREATE TABLE produits_valides (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100),
    details JSON,
    CONSTRAINT check_json_schema CHECK (
        JSON_SCHEMA_VALID(
            '{
                "type": "object",
                "required": ["prix", "stock"],
                "properties": {
                    "prix": {"type": "number", "minimum": 0},
                    "stock": {"type": "integer", "minimum": 0},
                    "description": {"type": "string", "maxLength": 500}
                },
                "additionalProperties": false
            }',
            details
        )
    )
);

-- ✅ Insertion valide
INSERT INTO produits_valides (nom, details) VALUES
  ('Produit A', '{"prix": 99.99, "stock": 50, "description": "Super produit"}');

-- ❌ Insertion rejetée (prix négatif)
INSERT INTO produits_valides (nom, details) VALUES
  ('Produit B', '{"prix": -10, "stock": 10}');
-- Erreur: Check constraint 'check_json_schema' is violated

-- ❌ Insertion rejetée (champ requis manquant)
INSERT INTO produits_valides (nom, details) VALUES
  ('Produit C', '{"stock": 10}');  -- 'prix' manquant
-- Erreur: Check constraint 'check_json_schema' is violated
```

**Avantages** :
- ✅ Garantit la cohérence des données JSON
- ✅ Validation côté base de données (pas besoin de valider dans l'application)
- ✅ Documentation du schéma intégrée
- ✅ Prévient les erreurs de typage

#### 3. Optimisations performances

```sql
-- ✅ OPTIMISÉ : Extraction JSON avec SIMD (AVX2/AVX512)
-- MariaDB 11.8 utilise les instructions SIMD pour parser le JSON
SELECT JSON_EXTRACT(data, '$.user.profile.settings')
FROM large_json_table;
-- Jusqu'à 2-3x plus rapide sur données volumineuses
```

---

## Types de stockage JSON dans MariaDB

### Type JSON (alias de LONGTEXT)

```sql
-- Ces trois déclarations sont équivalentes
CREATE TABLE t1 (data JSON);
CREATE TABLE t2 (data LONGTEXT);
CREATE TABLE t3 (data TEXT);  -- Si < 65KB
```

💡 **Important** : Contrairement à MySQL 5.7+, MariaDB ne dispose pas d'un type binaire JSON natif. `JSON` est un **alias** pour `LONGTEXT` avec validation automatique du format.

### Validation automatique

```sql
-- ✅ JSON valide : accepté
INSERT INTO t1 (data) VALUES ('{"nom": "Alice", "age": 30}');

-- ❌ JSON invalide : rejeté
INSERT INTO t1 (data) VALUES ('{invalid}');
-- Erreur: Invalid JSON text in argument 1 to function json_valid

-- ⚠️ Peut être contourné en insérant via LONGTEXT
ALTER TABLE t1 MODIFY data LONGTEXT;  -- Retire la validation
INSERT INTO t1 (data) VALUES ('{invalid}');  -- Accepté !
-- Puis JSON_VALID() retournera 0 pour cette ligne
```

**Recommandation** : Toujours utiliser le type `JSON` (pas `LONGTEXT`) pour bénéficier de la validation automatique.

---

## Quand utiliser JSON vs colonnes relationnelles ?

### Arbre de décision

```
Avez-vous besoin de recherches/jointures fréquentes sur ces données ?
│
├─ OUI → Utiliser des colonnes relationnelles
│         Exemple : user_id, email, created_at
│
└─ NON → Le schéma est-il stable et connu à l'avance ?
          │
          ├─ OUI → Utiliser des colonnes relationnelles
          │         Exemple : prix, quantite, nom_produit
          │
          └─ NON → Le schéma varie-t-il selon les enregistrements ?
                    │
                    ├─ OUI → Utiliser JSON
                    │         Exemple : attributs produits, métadonnées, configurations
                    │
                    └─ NON → Les données sont-elles complexes/hiérarchiques ?
                              │
                              ├─ OUI → Utiliser JSON
                              │         Exemple : historique, logs, réponses API
                              │
                              └─ NON → Utiliser colonnes relationnelles (défaut)
```

### Matrice de décision

| Critère | Colonnes relationnelles | JSON |
|---------|------------------------|------|
| **Schéma stable** | ✅ Idéal | ❌ Surdimensionné |
| **Recherches fréquentes** | ✅ Performant (index) | ⚠️ Lent (scan) |
| **Jointures complexes** | ✅ Natif SQL | ❌ Difficile |
| **Schéma variable** | ❌ ALTER TABLE fréquents | ✅ Flexible |
| **Données hiérarchiques** | ⚠️ Multiples tables | ✅ Structure naturelle |
| **Intégration API** | ⚠️ Mapping nécessaire | ✅ Direct |
| **Validation typage** | ✅ Natif (types SQL) | ⚠️ Via JSON Schema (11.8+) |
| **Consommation espace** | ✅ Optimisé | ⚠️ Plus volumineux |
| **Agrégations** | ✅ Rapide (SUM, AVG) | ❌ Extraction + calcul |

### Exemples de cas d'usage

#### ✅ Bon usage de JSON

```sql
-- 1. Attributs produits variables selon les catégories
CREATE TABLE produits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    categorie VARCHAR(50) NOT NULL,
    prix DECIMAL(10,2) NOT NULL,  -- Colonne : recherche fréquente
    stock INT NOT NULL,            -- Colonne : agrégations (SUM, AVG)
    attributs JSON,                -- JSON : varie selon catégorie
    INDEX(categorie, prix)
);

-- Électronique
INSERT INTO produits (nom, categorie, prix, stock, attributs) VALUES
  ('Laptop Pro', 'Electronique', 1299, 15,
   '{"processeur": "Intel i7", "ram_gb": 16, "ssd_gb": 512, "ecran_pouces": 15.6}');

-- Vêtements
INSERT INTO produits (nom, categorie, prix, stock, attributs) VALUES
  ('T-Shirt Premium', 'Vetements', 29, 150,
   '{"taille": "M", "couleur": "bleu", "matiere": "coton bio", "coupe": "slim"}');

-- Livres
INSERT INTO produits (nom, categorie, prix, stock, attributs) VALUES
  ('Les Misérables', 'Livres', 12, 45,
   '{"auteur": "Victor Hugo", "pages": 1488, "editeur": "Folio", "isbn": "978-2-07-036692-4"}');
```

```sql
-- 2. Logs et événements
CREATE TABLE audit_trail (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    timestamp DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    user_id INT NOT NULL,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50),
    entity_id INT,
    details JSON,  -- Contexte variable selon l'action
    INDEX(user_id, timestamp),
    INDEX(action, timestamp)
);

-- Différentes actions = différentes structures de details
INSERT INTO audit_trail (user_id, action, entity_type, entity_id, details) VALUES
  (101, 'login', NULL, NULL,
   '{"ip": "192.168.1.50", "user_agent": "Chrome 120", "session_id": "abc123"}'),
  (101, 'update', 'commande', 5042,
   '{"champs": ["statut", "adresse_livraison"], "ancien_statut": "en_preparation", "nouveau_statut": "expedie"}'),
  (102, 'delete', 'produit', 888,
   '{"nom_produit": "Produit obsolète", "raison": "discontinué", "stock_restant": 3}');
```

```sql
-- 3. Configuration et préférences utilisateur
CREATE TABLE user_preferences (
    user_id INT PRIMARY KEY,
    preferences JSON,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Chaque utilisateur peut avoir des préférences différentes
INSERT INTO user_preferences (user_id, preferences) VALUES
  (1, '{"theme": "dark", "langue": "fr", "notifications": {"email": true, "push": false}, "sidebar_collapsed": true}'),
  (2, '{"theme": "light", "langue": "en", "timezone": "America/New_York", "weekly_digest": true}');
```

#### ❌ Mauvais usage de JSON

```sql
-- ❌ ERREUR : Données structurées stables en JSON
CREATE TABLE commandes_bad (
    id INT PRIMARY KEY,
    data JSON  -- Tout en JSON !
);

INSERT INTO commandes_bad VALUES
  (1, '{"client_id": 42, "date": "2024-12-10", "montant": 99.99, "statut": "livree"}');

-- Problèmes :
-- 1. Impossible d'indexer client_id, date, statut
-- 2. Jointures complexes : JSON_EXTRACT + CAST
-- 3. Agrégations lentes : SUM(CAST(JSON_EXTRACT(...) AS DECIMAL))
-- 4. Pas de validation (client_id pourrait être une string)

-- ✅ CORRECT : Colonnes pour données structurées
CREATE TABLE commandes_good (
    id INT PRIMARY KEY,
    client_id INT NOT NULL,
    date_commande DATE NOT NULL,
    montant DECIMAL(10,2) NOT NULL,
    statut ENUM('en_attente', 'expedie', 'livree', 'annulee') NOT NULL,
    metadata JSON,  -- Optionnel : infos supplémentaires variables
    INDEX(client_id, date_commande),
    INDEX(statut)
);
```

---

## Approche hybride : Le meilleur des deux mondes

### Pattern : Colonnes clés + JSON pour détails

```sql
-- ✅ RECOMMANDÉ : Modèle hybride
CREATE TABLE commandes (
    -- Colonnes relationnelles : données critiques, recherchées, agrégées
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    date_commande DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    montant_total DECIMAL(10,2) NOT NULL,
    statut ENUM('en_attente', 'paiement_recu', 'expedie', 'livree', 'annulee') NOT NULL DEFAULT 'en_attente',

    -- JSON : métadonnées, contexte, données variables
    metadata JSON COMMENT 'Informations supplémentaires variables',

    -- Index sur colonnes relationnelles
    INDEX(client_id, date_commande),
    INDEX(statut),
    INDEX(montant_total),

    -- Contrainte : garantir structure JSON (11.8+)
    CONSTRAINT check_metadata_schema CHECK (
        JSON_SCHEMA_VALID(
            '{
                "type": "object",
                "properties": {
                    "source": {"type": "string", "enum": ["web", "mobile", "api"]},
                    "promo_code": {"type": "string"},
                    "gift_message": {"type": "string", "maxLength": 200},
                    "shipping_notes": {"type": "string"}
                }
            }',
            metadata
        )
    )
);

-- Insertion avec données hybrides
INSERT INTO commandes (client_id, montant_total, statut, metadata) VALUES
  (1001, 149.99, 'paiement_recu',
   '{"source": "mobile", "promo_code": "NOEL2024", "gift_message": "Joyeux Noël!", "shipping_notes": "Livraison express demandée"}');

-- Requêtes performantes sur colonnes relationnelles
SELECT
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
WHERE statut = 'livree'
  AND date_commande >= '2024-01-01';

-- Extraction JSON quand nécessaire
SELECT
    id,
    client_id,
    montant_total,
    JSON_EXTRACT(metadata, '$.promo_code') AS code_promo,
    JSON_EXTRACT(metadata, '$.source') AS canal_vente
FROM commandes
WHERE date_commande >= CURRENT_DATE - INTERVAL 7 DAY;
```

### Pattern : Colonnes virtuelles indexées

```sql
-- Extraire des valeurs JSON fréquemment recherchées vers des colonnes virtuelles indexées
CREATE TABLE produits_hybride (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    categorie VARCHAR(50) NOT NULL,
    details JSON,

    -- Colonnes virtuelles extraites du JSON
    prix DECIMAL(10,2) AS (CAST(JSON_EXTRACT(details, '$.prix') AS DECIMAL(10,2))) STORED,
    marque VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(details, '$.marque'))) VIRTUAL,

    -- Index sur colonnes virtuelles pour performance
    INDEX(prix),
    INDEX(marque)
);

-- Insertion : seulement le JSON
INSERT INTO produits_hybride (nom, categorie, details) VALUES
  ('Smartphone XZ', 'Electronique',
   '{"prix": 599.99, "marque": "TechCorp", "couleurs": ["noir", "blanc"], "garantie_ans": 2}');

-- Requête performante grâce aux index sur colonnes virtuelles
SELECT nom, prix, marque
FROM produits_hybride
WHERE prix BETWEEN 500 AND 700
  AND marque = 'TechCorp';
-- Utilise les index sur prix et marque
```

---

## Vue d'ensemble des fonctionnalités JSON

### Catégories de fonctions

MariaDB offre plus de 40 fonctions JSON réparties en catégories :

| Catégorie | Fonctions clés | Usage |
|-----------|---------------|-------|
| **Création** | JSON_ARRAY, JSON_OBJECT, JSON_QUOTE | Construire des documents JSON |
| **Extraction** | JSON_EXTRACT, ->, ->>, JSON_UNQUOTE | Lire des valeurs JSON |
| **Modification** | JSON_SET, JSON_INSERT, JSON_REPLACE, JSON_REMOVE | Mettre à jour JSON |
| **Recherche** | JSON_CONTAINS, JSON_SEARCH, JSON_OVERLAPS | Vérifier présence de valeurs |
| **Validation** | JSON_VALID, JSON_SCHEMA_VALID 🆕 | Valider structure JSON |
| **Agrégation** | JSON_ARRAYAGG, JSON_OBJECTAGG | Construire JSON depuis lignes |
| **Transformation** | JSON_TABLE, JSON_KEYS, JSON_LENGTH | Convertir JSON ↔ relationnel |
| **Utilitaires** | JSON_TYPE, JSON_DEPTH, JSON_PRETTY | Inspecter documents JSON |

### Aperçu rapide des fonctions essentielles

```sql
-- Création
SELECT JSON_OBJECT('nom', 'Alice', 'age', 30) AS utilisateur;
-- {"nom": "Alice", "age": 30}

SELECT JSON_ARRAY('pomme', 'banane', 'orange') AS fruits;
-- ["pomme", "banane", "orange"]

-- Extraction
SELECT JSON_EXTRACT('{"nom": "Bob", "age": 25}', '$.nom') AS nom;
-- "Bob"

SELECT '{"nom": "Bob", "age": 25}'->>'$.nom' AS nom;  -- Opérateur raccourci (11.8+)
-- Bob (sans guillemets)

-- Modification
SELECT JSON_SET('{"x": 1}', '$.y', 2) AS resultat;
-- {"x": 1, "y": 2}

-- Validation
SELECT JSON_VALID('{"valide": true}') AS est_valide;
-- 1

SELECT JSON_VALID('{invalide}') AS est_valide;
-- 0

-- Validation schéma (11.8+) 🆕
SELECT JSON_SCHEMA_VALID(
    '{"type": "object", "required": ["nom"]}',
    '{"nom": "Alice", "age": 30}'
) AS conforme;
-- 1

-- Recherche
SELECT JSON_CONTAINS('{"tags": ["json", "sql"]}', '"json"', '$.tags') AS contient;
-- 1

-- Transformation : JSON → Table
SELECT * FROM JSON_TABLE(
    '[{"nom": "Alice", "age": 30}, {"nom": "Bob", "age": 25}]',
    '$[*]' COLUMNS(
        nom VARCHAR(50) PATH '$.nom',
        age INT PATH '$.age'
    )
) AS jt;
-- nom   | age
-- Alice | 30
-- Bob   | 25
```

---

## Performance : Considérations importantes

### ⚠️ Limitations de performance

```sql
-- ❌ LENT : Pas d'index direct sur valeurs JSON
SELECT * FROM produits
WHERE JSON_EXTRACT(details, '$.prix') < 100;
-- Full table scan

-- ✅ RAPIDE : Index sur colonne virtuelle
ALTER TABLE produits
ADD COLUMN prix_extracted DECIMAL(10,2) AS (CAST(JSON_EXTRACT(details, '$.prix') AS DECIMAL(10,2))) STORED,
ADD INDEX(prix_extracted);

SELECT * FROM produits
WHERE prix_extracted < 100;
-- Utilise l'index
```

### 💡 Bonnes pratiques de performance

1. **Extraire les champs fréquemment recherchés vers des colonnes (virtuelles ou réelles)**
   ```sql
   -- Colonne virtuelle STORED = matérialisée, indexable
   ADD COLUMN prix AS (JSON_EXTRACT(data, '$.prix')) STORED;
   ```

2. **Limiter la profondeur d'imbrication**
   ```sql
   -- ✅ BON : Structure plate
   {"nom": "Alice", "email": "alice@example.com", "age": 30}

   -- ⚠️ MOINS BON : Imbrication profonde
   {"user": {"profile": {"contact": {"primary": {"email": "alice@example.com"}}}}}
   ```

3. **Utiliser JSON pour données peu recherchées**
   ```sql
   -- Données affichées mais rarement filtrées
   -- Ex : préférences UI, métadonnées, contexte
   ```

4. **Surveiller la taille des documents JSON**
   ```sql
   -- Vérifier la taille moyenne
   SELECT
       AVG(LENGTH(details)) AS avg_size_bytes,
       MAX(LENGTH(details)) AS max_size_bytes
   FROM produits;

   -- Si > 64KB, risque de fragmentation
   ```

---

## Compatibilité et migration

### Différences MariaDB vs MySQL

| Aspect | MariaDB | MySQL 5.7+ |
|--------|---------|-----------|
| **Type de stockage** | LONGTEXT (texte) | Binaire optimisé |
| **Validation** | À l'insertion | À l'insertion + structure interne |
| **Performance** | Parsing à chaque requête | Plus rapide (format binaire) |
| **Fonctions** | ~40 fonctions | ~40 fonctions |
| **JSON_TABLE** | ✅ Supporté | ✅ Supporté |
| **JSON Schema** | ✅ 11.8+ 🆕 | ❌ Via CHECK (limité) |
| **JSONPath avancé** | ✅ 11.8+ 🆕 | ⚠️ Limité |

### Migration MySQL → MariaDB

```sql
-- Les fonctions JSON standard sont compatibles
-- Requêtes portables entre MySQL et MariaDB :
SELECT
    JSON_EXTRACT(data, '$.user.name') AS nom,
    JSON_EXTRACT(data, '$.user.email') AS email
FROM users;

-- ⚠️ Attention : Performance peut différer
-- MariaDB parse le JSON à chaque requête (texte)
-- MySQL utilise un format binaire (plus rapide)
```

---

## Nouveautés MariaDB 11.8 en détail 🆕

### 1. JSON Path Expressions : Syntaxe avancée

```sql
-- Filtre avec condition
SELECT JSON_EXTRACT(
    '{"produits": [
        {"nom": "A", "prix": 50, "stock": 10},
        {"nom": "B", "prix": 150, "stock": 5},
        {"nom": "C", "prix": 30, "stock": 20}
    ]}',
    '$.produits[?(@.prix < 100 && @.stock > 5)].nom'
) AS produits_disponibles_abordables;
-- ["A", "C"]

-- Wildcard récursif
SELECT JSON_EXTRACT(
    '{"dept": {
        "it": {"email": "it@company.com"},
        "sales": {"email": "sales@company.com"},
        "hr": {"contact": {"email": "hr@company.com"}}
    }}',
    '$..email'
) AS tous_les_emails;
-- ["it@company.com", "sales@company.com", "hr@company.com"]

-- Opérateurs logiques complexes
SELECT JSON_EXTRACT(
    '{"items": [
        {"type": "A", "value": 10},
        {"type": "B", "value": 20},
        {"type": "A", "value": 30}
    ]}',
    '$.items[?(@.type == "A" || @.value > 25)]'
) AS resultats_filtres;
-- [{"type": "A", "value": 10}, {"type": "B", "value": 20}, {"type": "A", "value": 30}]
```

### 2. JSON Schema Validation : Contraintes robustes

```sql
-- Schéma complexe avec validations multiples
CREATE TABLE utilisateurs_valides (
    id INT PRIMARY KEY AUTO_INCREMENT,
    profile JSON,
    CONSTRAINT check_profile_schema CHECK (
        JSON_SCHEMA_VALID(
            '{
                "type": "object",
                "required": ["nom", "email", "age"],
                "properties": {
                    "nom": {
                        "type": "string",
                        "minLength": 2,
                        "maxLength": 100
                    },
                    "email": {
                        "type": "string",
                        "format": "email",
                        "pattern": "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
                    },
                    "age": {
                        "type": "integer",
                        "minimum": 18,
                        "maximum": 120
                    },
                    "telephone": {
                        "type": "string",
                        "pattern": "^\\+?[0-9]{10,15}$"
                    },
                    "adresse": {
                        "type": "object",
                        "properties": {
                            "rue": {"type": "string"},
                            "ville": {"type": "string"},
                            "code_postal": {"type": "string", "pattern": "^[0-9]{5}$"}
                        }
                    }
                },
                "additionalProperties": false
            }',
            profile
        )
    )
);

-- ✅ Valide
INSERT INTO utilisateurs_valides (profile) VALUES
  ('{"nom": "Alice Dupont", "email": "alice@example.com", "age": 30, "telephone": "+33612345678"}');

-- ❌ Invalide : email incorrect
INSERT INTO utilisateurs_valides (profile) VALUES
  ('{"nom": "Bob", "email": "pas-un-email", "age": 25}');
-- Erreur : Check constraint violated

-- ❌ Invalide : age hors limites
INSERT INTO utilisateurs_valides (profile) VALUES
  ('{"nom": "Charlie", "email": "charlie@example.com", "age": 15}');
-- Erreur : Check constraint violated (age < 18)
```

---

## ✅ Points clés à retenir

- **JSON dans MariaDB** = LONGTEXT avec validation + fonctions spécialisées (pas de format binaire comme MySQL)
- **Cas d'usage idéaux** : Attributs variables, logs, métadonnées, intégration API, données polymorphiques
- **Éviter JSON pour** : Données structurées stables, recherches fréquentes, jointures critiques, agrégations
- **Approche hybride recommandée** : Colonnes relationnelles pour champs clés + JSON pour flexibilité
- **MariaDB 11.8 🆕** : JSON Path Expressions avancées (filtres, wildcards) + JSON Schema Validation native
- **Performance** : Utiliser colonnes virtuelles indexées pour valeurs JSON fréquemment recherchées
- **Validation** : JSON Schema (11.8+) garantit la cohérence des données JSON
- **Compatibilité** : Fonctions JSON standard portables entre MySQL et MariaDB
- **Best practice** : Ne pas tout mettre en JSON - combiner avec le modèle relationnel

---

## 🔗 Ressources et références

- [📖 Documentation officielle MariaDB - JSON Functions](https://mariadb.com/kb/en/json-functions/)
- [📖 JSON Data Type](https://mariadb.com/kb/en/json-data-type/)
- [📖 MariaDB 11.8 Release Notes - JSON improvements](https://mariadb.com/kb/en/mariadb-11-8-0-release-notes/) 🆕
- [📖 JSON_SCHEMA_VALID](https://mariadb.com/kb/en/json_schema_valid/) 🆕
- [📖 JSON Path Expressions](https://mariadb.com/kb/en/json-path-expressions/) 🆕

**Standards** :
- [JSON Schema Specification](https://json-schema.org/)
- [JSONPath Specification](https://goessner.net/articles/JsonPath/)
- [RFC 7159 - JSON Format](https://tools.ietf.org/html/rfc7159)

---

## ➡️ Section suivante

**[4.7.1 Stockage et type de données JSON](/04-concepts-avances-sql/07.1-stockage-type-json.md)** : Détails techniques sur le type JSON, comparaison avec LONGTEXT, comportement de validation, et optimisations de stockage.

---

**Note importante** : Le support JSON de MariaDB évolue rapidement. La version 11.8 marque un tournant majeur avec les JSON Path Expressions avancées et la validation de schéma native. Ces fonctionnalités rapprochent MariaDB des capacités des bases NoSQL tout en conservant les avantages du relationnel. Utilisez JSON avec discernement : c'est un outil puissant qui complète le SQL, mais ne le remplace pas ! 🎯

⏭️ [Stockage et type de données JSON](/04-concepts-avances-sql/07.1-stockage-type-json.md)

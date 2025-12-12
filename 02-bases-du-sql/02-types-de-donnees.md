🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.2 Types de données MariaDB

> **Niveau** : Débutant
> **Durée estimée** : 1 heure
> **Prérequis** : Section 2.1 (Introduction au langage SQL)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre ce qu'est un type de données et pourquoi il est important
- Identifier les différentes catégories de types disponibles dans MariaDB
- Connaître les critères de choix d'un type de données approprié
- Anticiper l'impact du choix du type sur les performances et le stockage
- Comprendre les nouveautés de MariaDB 11.8 concernant les types de données

---

## Introduction

Les **types de données** définissent la nature des valeurs qui peuvent être stockées dans chaque colonne d'une table. C'est l'une des décisions les plus importantes lors de la conception d'une base de données.

### Qu'est-ce qu'un type de données ?

Un type de données spécifie :
- **Quelle sorte de valeur** peut être stockée (nombre, texte, date, etc.)
- **Combien d'espace** la valeur occupera en mémoire et sur disque
- **Quelles opérations** peuvent être effectuées sur cette valeur
- **Quelles sont les limites** (valeur minimale/maximale, longueur, etc.)

```sql
-- Exemple simple : Une table avec différents types
CREATE TABLE exemple_types (
    -- Entier pour identifiant
    id INT PRIMARY KEY AUTO_INCREMENT,

    -- Texte pour nom
    nom VARCHAR(100),

    -- Nombre décimal pour prix
    prix DECIMAL(10,2),

    -- Date pour date de naissance
    date_naissance DATE,

    -- Booléen (stocké comme TINYINT)
    actif BOOLEAN
);

-- Chaque colonne a un type adapté à son usage
```

---

## Pourquoi les types de données sont-ils importants ?

### 1. Intégrité des données

Le type de données assure que seules des valeurs valides sont stockées :

```sql
-- Avec le bon type, MariaDB valide automatiquement
CREATE TABLE produits (
    produit_id INT,
    nom VARCHAR(100),
    prix DECIMAL(8,2),                          -- Seulement des nombres avec 2 décimales
    date_ajout DATE                             -- Seulement des dates valides
);

-- ✅ Insertion valide
INSERT INTO produits VALUES (1, 'Livre', 19.99, '2025-12-12');

-- ❌ Insertion invalide - Rejetée automatiquement
INSERT INTO produits VALUES (2, 'Stylo', 'quinze euros', '32/13/2025');
-- ERROR: Incorrect decimal value, incorrect date value
```

### 2. Optimisation de l'espace de stockage

Choisir le bon type économise de l'espace :

```sql
-- Comparaison d'espace de stockage
CREATE TABLE demo_stockage (
    -- TINYINT : 1 octet (0-255)
    age TINYINT UNSIGNED,                       -- Économique pour l'âge

    -- INT : 4 octets (-2B à 2B)
    population INT,                             -- Adapté aux grandes valeurs

    -- BIGINT : 8 octets (très grandes valeurs)
    compteur_total BIGINT,                      -- Pour compteurs massifs

    -- VARCHAR(50) : Variable (jusqu'à 50 caractères + 1 octet)
    nom VARCHAR(50),                            -- Économique si texte court

    -- TEXT : Variable (jusqu'à 64KB + 2 octets)
    description TEXT                            -- Pour texte long
);

-- Pour 1 million de lignes :
-- age (TINYINT) : 1 MB
-- age (INT) : 4 MB
-- Économie : 3 MB par million de lignes !
```

### 3. Performances des requêtes

Le type influence la vitesse des opérations :

```sql
-- Les entiers sont plus rapides que les chaînes
CREATE TABLE comparaison_perf (
    id_entier INT PRIMARY KEY,                  -- Comparaison très rapide
    id_texte VARCHAR(20) UNIQUE                 -- Comparaison plus lente
);

-- Recherche sur entier : Rapide
SELECT * FROM comparaison_perf WHERE id_entier = 12345;

-- Recherche sur texte : Plus lent
SELECT * FROM comparaison_perf WHERE id_texte = '12345';

-- Les index sur entiers sont aussi plus compacts et rapides
```

### 4. Compatibilité avec les applications

Le type doit correspondre aux besoins de l'application :

```sql
-- Types qui correspondent aux types des langages de programmation
CREATE TABLE mapping_types (
    -- INT → int (Python, Java, C++)
    quantite INT,

    -- DECIMAL → Decimal (Python), BigDecimal (Java)
    prix DECIMAL(10,2),

    -- VARCHAR → String
    nom VARCHAR(100),

    -- DATETIME → datetime (Python), LocalDateTime (Java)
    timestamp_event DATETIME,

    -- JSON → dict (Python), Object (JavaScript)
    metadata JSON
);
```

---

## Vue d'ensemble des types de données

MariaDB propose **5 catégories principales** de types de données :

### 1. Types numériques

Pour stocker des nombres (entiers ou décimaux).

| Type | Taille | Plage | Usage typique |
|------|--------|-------|---------------|
| **TINYINT** | 1 octet | -128 à 127 | Âge, statut, pourcentage |
| **SMALLINT** | 2 octets | -32,768 à 32,767 | Année, quantité |
| **INT** | 4 octets | -2.1B à 2.1B | Identifiants, compteurs |
| **BIGINT** | 8 octets | -9.2E18 à 9.2E18 | Grands compteurs, timestamps |
| **DECIMAL(M,D)** | Variable | Précis | **Prix, argent** (précision exacte) |
| **FLOAT** | 4 octets | Approximatif | Calculs scientifiques |
| **DOUBLE** | 8 octets | Approximatif | Calculs scientifiques précis |

```sql
-- Exemple de types numériques
CREATE TABLE exemple_numeriques (
    id INT PRIMARY KEY AUTO_INCREMENT,
    age TINYINT UNSIGNED,                       -- 0 à 255
    annee_naissance SMALLINT,                   -- 1901 à 2155
    population_ville INT,                       -- Millions d'habitants
    nombre_visites BIGINT,                      -- Compteur très grand
    prix_euros DECIMAL(10,2),                   -- Argent (TOUJOURS DECIMAL)
    temperature_celsius FLOAT,                  -- Mesure scientifique
    coordonnee_gps DOUBLE                       -- Latitude/Longitude précise
);
```

💡 **Règle d'or** : Pour l'argent, **TOUJOURS utiliser DECIMAL** (jamais FLOAT ou DOUBLE).

---

### 2. Types texte

Pour stocker des chaînes de caractères.

| Type | Longueur max | Préfixe | Usage typique |
|------|--------------|---------|---------------|
| **CHAR(M)** | 255 caractères | - | Codes fixes (ISO, postal) |
| **VARCHAR(M)** | 65,535 octets | 1-2 octets | Noms, emails, descriptions |
| **TEXT** | 65,535 octets | 2 octets | Articles, commentaires |
| **MEDIUMTEXT** | 16 MB | 3 octets | Documents longs |
| **LONGTEXT** | 4 GB | 4 octets | Livres, gros contenus |
| **ENUM** | 65,535 valeurs | 1-2 octets | Liste fermée (statuts) |
| **SET** | 64 membres | 1-8 octets | Sélections multiples |

```sql
-- Exemple de types texte
CREATE TABLE exemple_texte (
    id INT PRIMARY KEY AUTO_INCREMENT,
    code_pays CHAR(2),                          -- 'FR', 'US' (longueur fixe)
    nom VARCHAR(100) NOT NULL,                  -- Nom variable
    email VARCHAR(255) UNIQUE,                  -- Email (standard 255)
    description TEXT,                           -- Description longue
    contenu_article MEDIUMTEXT,                 -- Article complet
    statut ENUM('actif', 'inactif', 'suspendu'), -- Une valeur parmi liste
    tags SET('urgent', 'important', 'public')   -- Plusieurs valeurs possibles
);
```

🆕 **MariaDB 11.8** : Le charset par défaut est **utf8mb4** avec la collation **uca_1400_ai_ci**, offrant un meilleur support Unicode (emojis inclus 😊).

---

### 3. Types temporels

Pour stocker des dates et des heures.

| Type | Taille | Plage | Usage typique |
|------|--------|-------|---------------|
| **DATE** | 3 octets | 1000-01-01 à 9999-12-31 | Dates seules |
| **TIME** | 3 octets | -838:59:59 à 838:59:59 | Heures, durées |
| **YEAR** | 1 octet | 1901 à 2155 | Années seules |
| **DATETIME** | 5 octets | 1000 à 9999 | Date+heure sans TZ |
| **TIMESTAMP** | 4 octets | 1970 à **2106** 🆕 | Date+heure avec TZ |

```sql
-- Exemple de types temporels
CREATE TABLE exemple_temporel (
    id INT PRIMARY KEY AUTO_INCREMENT,
    date_naissance DATE,                        -- Date seule
    heure_ouverture TIME,                       -- Heure seule
    annee_fabrication YEAR,                     -- Année seule
    date_evenement DATETIME,                    -- Date et heure (fixe)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, -- Avec fuseau horaire
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                        ON UPDATE CURRENT_TIMESTAMP -- Auto-update
);
```

🆕 **MariaDB 11.8** : Le type **TIMESTAMP** supporte maintenant les dates jusqu'en **2106** (au lieu de 2038), résolvant le problème Y2038.

---

### 4. Types binaires

Pour stocker des données brutes (images, fichiers).

| Type | Longueur max | Usage typique |
|------|--------------|---------------|
| **BINARY(M)** | 255 octets | Hash, signatures fixes |
| **VARBINARY(M)** | 65,535 octets | Petits fichiers |
| **BLOB** | 65,535 octets | Images moyennes |
| **MEDIUMBLOB** | 16 MB | Documents, PDF |
| **LONGBLOB** | 4 GB | Vidéos (déconseillé) |

```sql
-- Exemple de types binaires
CREATE TABLE exemple_binaire (
    id INT PRIMARY KEY AUTO_INCREMENT,
    hash_password BINARY(32),                   -- SHA-256 (32 octets)
    token_session VARBINARY(255),               -- Token variable
    avatar BLOB,                                -- Image d'avatar
    document_pdf MEDIUMBLOB                     -- Document PDF
);
```

⚠️ **Attention** : Pour les fichiers > 1 MB, il est souvent préférable de les stocker sur le filesystem et de garder seulement le chemin en base.

---

### 5. Types spécifiques MariaDB

Types modernes pour cas d'usage avancés.

| Type | Taille | Nouveauté | Usage typique |
|------|--------|-----------|---------------|
| **JSON** | Variable | ✅ | Données semi-structurées, APIs |
| **UUID** | 36 ou 16 octets | ✅ | Identifiants uniques universels |
| **INET6** | 4-16 octets | ✅ | Adresses IPv4 et IPv6 |
| **VECTOR(N)** | Variable | 🆕 11.8 | Embeddings IA/ML |

```sql
-- Exemple de types spécifiques
CREATE TABLE exemple_specifique (
    id INT PRIMARY KEY AUTO_INCREMENT,

    -- JSON pour données flexibles
    preferences JSON,                           -- {'theme': 'dark', 'langue': 'fr'}

    -- UUID pour identifiants distribués
    session_uuid BINARY(16) DEFAULT (UUID_TO_BIN(UUID())),

    -- INET6 pour adresses IP
    ip_address INET6,                           -- IPv4 et IPv6

    -- VECTOR pour IA (aperçu)
    embedding VECTOR(1536)                      -- Embeddings OpenAI 🆕
);
```

🆕 **MariaDB 11.8** : Introduction du type **VECTOR** pour stocker des embeddings et effectuer des recherches vectorielles natives (IA/ML).

---

## Principes de choix d'un type de données

### Règle 1 : Choisir le type le plus petit qui convient

```sql
-- ❌ MAUVAIS : Type trop grand
CREATE TABLE mauvais_choix (
    age BIGINT,                                 -- 8 octets pour 0-120 !
    actif VARCHAR(50)                           -- 50 caractères pour 'oui'/'non' !
);

-- ✅ BON : Type adapté
CREATE TABLE bon_choix (
    age TINYINT UNSIGNED,                       -- 1 octet pour 0-255
    actif BOOLEAN                               -- 1 octet (alias de TINYINT(1))
);

-- Pour 1 million de lignes :
-- Économie : (8-1) + (50-1) = 56 octets × 1M = 56 MB
```

### Règle 2 : Privilégier la précision pour l'argent

```sql
-- ❌ MAUVAIS : FLOAT pour l'argent
CREATE TABLE prix_float (
    prix FLOAT                                  -- Imprécis !
);

INSERT INTO prix_float VALUES (19.99);
SELECT prix * 3 FROM prix_float;                -- 59.97000122... (erreur !)

-- ✅ BON : DECIMAL pour l'argent
CREATE TABLE prix_decimal (
    prix DECIMAL(10,2)                          -- Précision exacte
);

INSERT INTO prix_decimal VALUES (19.99);
SELECT prix * 3 FROM prix_decimal;              -- 59.97 (exact)
```

### Règle 3 : Utiliser VARCHAR plutôt que CHAR (sauf cas spécifiques)

```sql
-- CHAR : Longueur fixe (padding avec espaces)
CREATE TABLE demo_char (
    code CHAR(10)                               -- Toujours 10 octets
);

INSERT INTO demo_char VALUES ('ABC');           -- Stocké comme 'ABC       '

-- VARCHAR : Longueur variable
CREATE TABLE demo_varchar (
    code VARCHAR(10)                            -- 3 octets + 1 octet préfixe
);

INSERT INTO demo_varchar VALUES ('ABC');        -- Stocké comme 'ABC' (3 octets)

-- ✅ Utilisez CHAR uniquement pour longueur vraiment fixe :
-- - Codes pays (FR, US, DE)
-- - Hash MD5 (32 caractères)
-- - UUID textuel (36 caractères)
```

### Règle 4 : DATETIME vs TIMESTAMP selon le besoin

```sql
-- DATETIME : Date/heure "fixe" (pas de conversion fuseau horaire)
CREATE TABLE reservations (
    date_rdv DATETIME                           -- 2025-12-12 15:00:00 reste fixe
);

-- TIMESTAMP : Date/heure "mobile" (converti selon fuseau horaire session)
CREATE TABLE logs (
    timestamp_log TIMESTAMP                     -- Stocké en UTC, converti
);

-- ✅ Règle simple :
-- - Rendez-vous, événements → DATETIME
-- - Logs, audit, tracking → TIMESTAMP
```

### Règle 5 : Éviter ENUM/SET pour données qui évoluent

```sql
-- ❌ PROBLÉMATIQUE : Ajouter une valeur nécessite ALTER TABLE
CREATE TABLE commandes_enum (
    statut ENUM('attente', 'validee', 'expediee')
);

-- Plus tard : besoin d'ajouter 'annulee'
ALTER TABLE commandes_enum
MODIFY statut ENUM('attente', 'validee', 'expediee', 'annulee');
-- ALTER TABLE = blocage de table, lent sur grandes tables !

-- ✅ FLEXIBLE : Table de référence
CREATE TABLE statuts (
    statut_id TINYINT PRIMARY KEY,
    libelle VARCHAR(50)
);

CREATE TABLE commandes (
    commande_id INT PRIMARY KEY,
    statut_id TINYINT,
    FOREIGN KEY (statut_id) REFERENCES statuts(statut_id)
);

-- Ajouter un statut = simple INSERT (pas d'ALTER TABLE)
```

---

## Impact du type sur les index

Les types de données influencent directement les performances des index :

```sql
-- Comparaison de taille d'index
CREATE TABLE demo_index (
    -- Index sur INT : Compact et rapide
    id INT PRIMARY KEY,                         -- 4 octets par entrée

    -- Index sur BIGINT : Plus grand
    user_id BIGINT,                             -- 8 octets par entrée
    INDEX idx_user (user_id),

    -- Index sur VARCHAR : Variable, plus lent
    email VARCHAR(255),                         -- Jusqu'à 255 caractères
    INDEX idx_email (email),

    -- Index sur TEXT : Impossible sans préfixe
    description TEXT,
    INDEX idx_desc (description(100))           -- Préfixe requis
);

-- Pour 1 million de lignes :
-- idx_user (BIGINT) : ~8 MB
-- idx_email (VARCHAR) : 50-200 MB (selon longueur moyenne)

-- ✅ Recommandation : Entiers pour clés primaires et étrangères
```

---

## Tableau récapitulatif par usage

| Usage | Type recommandé | Exemple |
|-------|-----------------|---------|
| **Identifiant (PK/FK)** | INT ou BIGINT | `id INT PRIMARY KEY AUTO_INCREMENT` |
| **Prix, argent** | DECIMAL(M,D) | `prix DECIMAL(10,2)` |
| **Âge** | TINYINT UNSIGNED | `age TINYINT UNSIGNED` |
| **Nom, prénom** | VARCHAR(50-100) | `nom VARCHAR(100)` |
| **Email** | VARCHAR(255) | `email VARCHAR(255)` |
| **Description courte** | VARCHAR(500) | `description VARCHAR(500)` |
| **Article, texte long** | TEXT | `contenu TEXT` |
| **Date de naissance** | DATE | `date_naissance DATE` |
| **Date création** | DATETIME | `created_at DATETIME` |
| **Audit log** | TIMESTAMP | `timestamp TIMESTAMP` |
| **Année** | YEAR ou SMALLINT | `annee YEAR` |
| **Booléen** | BOOLEAN (TINYINT) | `actif BOOLEAN` |
| **Hash password** | BINARY(32) | `hash BINARY(32)` |
| **Statut (liste fixe)** | ENUM ou FK | `statut ENUM('actif', 'inactif')` |
| **Tags multiples** | JSON ou table M:N | `tags JSON` |
| **Adresse IP** | INET6 | `ip INET6` |
| **UUID** | BINARY(16) | `uuid BINARY(16)` |
| **Données flexibles** | JSON | `metadata JSON` |

---

## Conversions et compatibilité

### Conversion automatique (CAST implicite)

MariaDB convertit automatiquement certains types :

```sql
-- Conversion automatique
SELECT
    10 + '5' AS addition,                       -- '5' converti en 5 → 15
    CONCAT(123, ' items') AS concatenation,     -- 123 converti en '123'
    '2025-12-12' + INTERVAL 1 DAY AS date_calc; -- String → DATE
```

### Conversion explicite (CAST)

```sql
-- CAST : Conversion explicite
SELECT
    CAST('123' AS SIGNED) AS chaine_vers_entier,        -- 123
    CAST(123.456 AS DECIMAL(10,2)) AS arrondi,          -- 123.46
    CAST('2025-12-12' AS DATE) AS texte_vers_date,      -- DATE
    CAST(NOW() AS CHAR) AS datetime_vers_texte;         -- String

-- CONVERT : Syntaxe alternative
SELECT
    CONVERT('123', SIGNED) AS conversion1,
    CONVERT(123.456, DECIMAL(10,2)) AS conversion2;
```

### Modification de type (ALTER TABLE)

```sql
-- Changer le type d'une colonne existante
CREATE TABLE demo_alter (
    id INT PRIMARY KEY,
    code VARCHAR(10)
);

-- Élargir la taille
ALTER TABLE demo_alter MODIFY code VARCHAR(20);         -- OK : pas de perte

-- Réduire la taille (attention !)
ALTER TABLE demo_alter MODIFY code VARCHAR(5);          -- RISQUE : troncature !

-- Changer complètement le type
ALTER TABLE demo_alter MODIFY code INT;                 -- Conversion si possible

-- ⚠️ ALTER TABLE peut être long et bloquant sur grandes tables
```

---

## Nouveautés MariaDB 11.8

🆕 **Principales améliorations des types de données** :

### 1. TIMESTAMP étendu jusqu'en 2106

```sql
-- ✅ Maintenant possible (résout problème Y2038)
CREATE TABLE events_futur (
    event_date TIMESTAMP
);

INSERT INTO events_futur VALUES ('2050-01-01 00:00:00');  -- OK !
INSERT INTO events_futur VALUES ('2100-12-31 23:59:59');  -- OK !
```

### 2. utf8mb4 par défaut

```sql
-- Dans MariaDB 11.8, utf8mb4 est le charset par défaut
CREATE DATABASE ma_base;  -- Automatiquement en utf8mb4

-- Support complet des emojis
CREATE TABLE messages (
    message TEXT
);

INSERT INTO messages VALUES ('Hello 👋 Comment ça va ? 😊');  -- OK !
```

### 3. Type VECTOR pour IA/ML

```sql
-- Nouveau type pour embeddings (intelligence artificielle)
CREATE TABLE documents_ia (
    doc_id INT PRIMARY KEY,
    contenu TEXT,
    embedding VECTOR(1536),                     -- Vecteur de 1536 dimensions
    INDEX idx_vector (embedding) USING HNSW    -- Index de recherche vectorielle
);

-- Permet recherche sémantique native dans MariaDB
-- Détails complets : voir section 18.10
```

### 4. Optimisations de performance

- Meilleures performances pour BIGINT dans calculs
- Optimisations du cost optimizer pour SSD
- Amélioration des fonctions JSON

---

## ✅ Points clés à retenir

- Le **type de données** définit la nature, la taille et les limites des valeurs stockées
- Choix du type impacte : **intégrité**, **espace disque**, **performances**, **index**
- **5 catégories** : Numériques, Texte, Temporels, Binaires, Spécifiques
- Pour l'argent : **TOUJOURS DECIMAL** (jamais FLOAT/DOUBLE)
- Pour identifiants : **INT** ou **BIGINT** (pas VARCHAR)
- Pour texte variable : **VARCHAR** (pas CHAR sauf codes fixes)
- 🆕 **MariaDB 11.8** : TIMESTAMP 2106, utf8mb4 défaut, type VECTOR
- Toujours choisir le **type le plus petit** qui convient
- Les entiers sont **plus rapides** que les chaînes pour index
- **Colonnes virtuelles** pour indexer JSON

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Data Types Overview](https://mariadb.com/kb/en/data-types/)
- [📖 Choosing the Right Data Type](https://mariadb.com/kb/en/data-type-storage-requirements/)
- [📖 Data Type Storage Requirements](https://mariadb.com/kb/en/data-type-storage-requirements/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-0-release-notes/)

### Bonnes pratiques
- [Database Design Best Practices](https://www.vertabelo.com/blog/database-design-best-practices/)
- [SQL Data Types Performance](https://use-the-index-luke.com/)

---

## ➡️ Sections suivantes

Les prochaines sections détaillent chaque catégorie de types :

**[2.2.1 Types numériques (INT, BIGINT, DECIMAL, FLOAT, DOUBLE)](/02-bases-du-sql/02.1-types-numeriques.md)**
Découvrez en détail les types entiers (TINYINT à BIGINT), les attributs UNSIGNED et AUTO_INCREMENT, et les types décimaux (DECIMAL, FLOAT, DOUBLE) avec leurs cas d'usage.

**[2.2.2 Types de texte (VARCHAR, TEXT, CHAR, ENUM, SET)](/02-bases-du-sql/02.2-types-texte.md)**
Maîtrisez les différents types texte, les charsets et collations (utf8mb4 🆕), et apprenez quand utiliser ENUM vs table de référence.

**[2.2.3 Types temporels (DATE, DATETIME, TIMESTAMP, TIME, YEAR)](/02-bases-du-sql/02.3-types-temporels.md)**
Comprenez les différences entre DATETIME et TIMESTAMP, l'extension 2106 🆕, et les fonctions de manipulation de dates.

**[2.2.4 Types binaires (BLOB, BINARY, VARBINARY)](/02-bases-du-sql/02.4-types-binaires.md)**
Apprenez à stocker des données binaires, les avantages/inconvénients du stockage de fichiers en base, et les alternatives filesystem.

**[2.2.5 Types spécifiques MariaDB (JSON, UUID, INET6)](/02-bases-du-sql/02.5-types-specifiques-mariadb.md)**
Explorez les types modernes : JSON pour données flexibles, UUID pour identifiants distribués, INET6 pour IP, et introduction au type VECTOR 🆕.

---


⏭️ [Numériques (INT, BIGINT, DECIMAL, FLOAT, DOUBLE)](/02-bases-du-sql/02.1-types-numeriques.md)

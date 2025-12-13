🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.11 Charset par défaut : utf8mb4 avec collations UCA 14.0.0 🆕

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** :
> - Section 11.1 (Fichiers de configuration)
> - Section 11.2 (Variables système)
> - Compréhension d'Unicode
> - Expérience en internationalisation (i18n)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** la différence entre utf8 et utf8mb4
- **Exploiter** utf8mb4 comme charset par défaut dans MariaDB 11.8
- **Utiliser** les collations UCA 14.0.0 modernes
- **Migrer** depuis utf8 ou latin1 vers utf8mb4
- **Gérer** les emoji, caractères spéciaux et langues complexes
- **Configurer** les charsets et collations optimalement
- **Diagnostiquer** les problèmes d'encodage
- **Optimiser** les performances avec utf8mb4

---

## Introduction

### 🆕 Nouveauté MariaDB 11.8

MariaDB 11.8 LTS marque un **tournant historique** en adoptant **utf8mb4** comme charset par défaut (au lieu de utf8/latin1) avec des **collations UCA 14.0.0** modernes.

**Changements clés** :
- ✅ **character_set_server = utf8mb4** (défaut)
- ✅ **collation_server = utf8mb4_uca1400_ai_ci** (défaut)
- ✅ Support complet Unicode 14.0 (septembre 2021)
- ✅ Emoji, symboles, langues rares nativement supportés

### Pourquoi utf8mb4 par défaut ?

```
Ancienne limitation utf8 (MySQL/MariaDB):
    "utf8" = Alias pour utf8mb3
    UTF-8 3 octets maximum par caractère
    ❌ Pas d'emoji (😀, 💖, 🎉)
    ❌ Caractères rares (𝕳𝖊𝖑𝖑𝖔, 𐐀𐐨)
    ❌ Caractères hors BMP (Basic Multilingual Plane)

Solution utf8mb4:
    UTF-8 4 octets maximum par caractère
    ✅ Emoji complets
    ✅ Tous les caractères Unicode
    ✅ Conformité standard UTF-8
```

💡 **Rappel historique** : "utf8" dans MySQL/MariaDB est en réalité **utf8mb3** (maximum 3 octets), ce qui est **incorrect** selon le standard UTF-8. **utf8mb4** est le vrai UTF-8.

---

## Histoire des charsets dans MariaDB

### Évolution chronologique

| Version | Charset défaut | Collation défaut | Support Unicode |
|---------|---------------|------------------|-----------------|
| **MySQL 4.0** | latin1 | latin1_swedish_ci | Limité (latin) |
| **MySQL 4.1** | latin1 | latin1_swedish_ci | utf8 disponible |
| **MySQL 5.5** | latin1 | latin1_swedish_ci | utf8mb4 introduit |
| **MySQL 8.0** | utf8mb4 | utf8mb4_0900_ai_ci | UCA 9.0.0 |
| **MariaDB ≤ 10.5** | latin1 | latin1_swedish_ci | utf8mb4 disponible |
| **MariaDB 10.6** | utf8mb4 | utf8mb4_uca1400_ai_ci | UCA 14.0.0 |
| **MariaDB 11.8** 🆕 | **utf8mb4** | **utf8mb4_uca1400_ai_ci** | **UCA 14.0.0** |

### Problème de l'alias "utf8"

```sql
-- ⚠️ PIÈGE HISTORIQUE
CREATE TABLE users (
    name VARCHAR(100) CHARACTER SET utf8
);

-- En réalité :
-- CHARACTER SET utf8 = CHARACTER SET utf8mb3 (3 octets max)
-- ❌ Ne stockera PAS les emoji !

-- Correct :
CREATE TABLE users (
    name VARCHAR(100) CHARACTER SET utf8mb4
);
```

---

## Unicode et UCA (Unicode Collation Algorithm)

### Qu'est-ce qu'Unicode ?

**Unicode** est un standard universel pour représenter **tous** les caractères de toutes les langues.

**Versions Unicode** :
- Unicode 1.0 (1991) : 7,161 caractères
- Unicode 6.0 (2010) : 109,449 caractères
- Unicode 14.0 (2021) : **144,697 caractères** ← MariaDB 11.8

**Caractères supportés** :
- Alphabets latins, grecs, cyrilliques
- Idéogrammes chinois, japonais, coréens (CJK)
- Arabe, hébreu, hindi, thaï, etc.
- Emoji 😀🎉💖
- Symboles mathématiques 𝕳𝖊𝖑𝖑𝖔
- Caractères historiques (hiéroglyphes, cunéiforme)

### Qu'est-ce qu'une collation ?

Une **collation** définit comment **comparer et trier** les caractères.

```sql
-- Exemple : Tri avec différentes collations
INSERT INTO words VALUES ('Café'), ('cafe'), ('CAFÉ');

-- Collation case-sensitive (cs)
SELECT * FROM words ORDER BY name COLLATE utf8mb4_bin;
-- CAFÉ, Café, cafe

-- Collation case-insensitive (ci)
SELECT * FROM words ORDER BY name COLLATE utf8mb4_uca1400_ai_ci;
-- cafe, Café, CAFÉ (ordre alphabétique, casse ignorée)
```

### UCA (Unicode Collation Algorithm)

**UCA** est l'algorithme officiel Unicode pour le tri multilingue.

**Versions UCA dans MariaDB** :

| Collation | UCA Version | Unicode | Disponible depuis |
|-----------|-------------|---------|-------------------|
| utf8mb4_unicode_ci | UCA 4.0.0 | Unicode 5.0 | MariaDB 5.5 |
| utf8mb4_uca1400_ai_ci | **UCA 14.0.0** | **Unicode 14.0** | **MariaDB 10.10** 🆕 |

**Suffixes de collation** :

| Suffixe | Signification | Exemple |
|---------|---------------|---------|
| **ai** | Accent-Insensitive | 'e' = 'é' = 'ê' |
| **as** | Accent-Sensitive | 'e' ≠ 'é' ≠ 'ê' |
| **ci** | Case-Insensitive | 'A' = 'a' |
| **cs** | Case-Sensitive | 'A' ≠ 'a' |
| **bin** | Binary (octet par octet) | Strictement binaire |

**Exemples** :
- `utf8mb4_uca1400_ai_ci` : Accent-Insensitive, Case-Insensitive
- `utf8mb4_uca1400_as_cs` : Accent-Sensitive, Case-Sensitive
- `utf8mb4_bin` : Binary (pas d'UCA, comparaison octet par octet)

---

## Configuration par défaut MariaDB 11.8

### Variables système

```sql
-- Vérifier les charsets par défaut
SHOW VARIABLES LIKE 'character_set%';
```

**Sortie MariaDB 11.8** :

```
+-------------------------------+----------------------------+
| Variable_name                 | Value                      |
+-------------------------------+----------------------------+
| character_set_client          | utf8mb4                    |
| character_set_connection      | utf8mb4                    |
| character_set_database        | utf8mb4                    |
| character_set_filesystem      | binary                     |
| character_set_results         | utf8mb4                    |
| character_set_server          | utf8mb4                    | ← Défaut serveur
| character_set_system          | utf8mb3                    |
| character_sets_dir            | /usr/share/mysql/charsets/ |
+-------------------------------+----------------------------+
```

```sql
-- Vérifier les collations par défaut
SHOW VARIABLES LIKE 'collation%';
```

**Sortie MariaDB 11.8** :

```
+-------------------------------+------------------------+
| Variable_name                 | Value                  |
+-------------------------------+------------------------+
| collation_connection          | utf8mb4_uca1400_ai_ci  |
| collation_database            | utf8mb4_uca1400_ai_ci  |
| collation_server              | utf8mb4_uca1400_ai_ci  | ← Défaut serveur
+-------------------------------+------------------------+
```

### Configuration my.cnf

```ini
# my.cnf - MariaDB 11.8 (valeurs par défaut)
[mysqld]
# Charset serveur
character-set-server = utf8mb4

# Collation serveur
collation-server = utf8mb4_uca1400_ai_ci

[client]
# Charset client
default-character-set = utf8mb4
```

---

## Collations disponibles utf8mb4

### Liste des collations principales

```sql
-- Toutes les collations utf8mb4
SHOW COLLATION WHERE Charset = 'utf8mb4';
-- Plus de 200 collations !

-- Collations UCA 14.0.0 (recommandées)
SHOW COLLATION WHERE Charset = 'utf8mb4' AND Collation LIKE '%uca1400%';
```

**Collations UCA 14.0.0 courantes** :

| Collation | Usage | Exemple |
|-----------|-------|---------|
| `utf8mb4_uca1400_ai_ci` | **Défaut 11.8**, général | 'Café' = 'cafe' |
| `utf8mb4_uca1400_as_ci` | Accent-sensitive | 'Café' ≠ 'Cafe' |
| `utf8mb4_uca1400_as_cs` | Accent+Case-sensitive | 'Café' ≠ 'café' ≠ 'cafe' |
| `utf8mb4_unicode_520_ci` | UCA 5.2.0, legacy | Ancienne version |
| `utf8mb4_bin` | Binary, strict | 0xC3A9 ≠ 0x65 |

**Collations spécifiques par langue** :

```sql
-- Français
utf8mb4_uca1400_french_ai_ci

-- Allemand
utf8mb4_uca1400_german2_ai_ci

-- Espagnol
utf8mb4_uca1400_spanish_ai_ci

-- Polonais
utf8mb4_uca1400_polish_ai_ci

-- Chinois (pinyin)
utf8mb4_zh_0900_as_cs

-- Japonais
utf8mb4_ja_0900_as_cs
```

### Choix de la collation

**Recommandations** :

| Cas d'usage | Collation recommandée |
|-------------|-----------------------|
| **Application multilingue** | `utf8mb4_uca1400_ai_ci` ✅ |
| **Recherche sensible accents** | `utf8mb4_uca1400_as_ci` |
| **Comparaison stricte** | `utf8mb4_uca1400_as_cs` |
| **Performance maximale** | `utf8mb4_bin` (pas de tri intelligent) |
| **Application française** | `utf8mb4_uca1400_french_ai_ci` |
| **Legacy (compatibilité)** | `utf8mb4_unicode_ci` |

---

## Création de bases et tables avec utf8mb4

### Base de données

```sql
-- Créer une base utf8mb4 (défaut 11.8)
CREATE DATABASE myapp;
-- Équivalent à :
CREATE DATABASE myapp
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_ai_ci;

-- Spécifier une collation différente
CREATE DATABASE myapp_fr
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_french_ai_ci;

-- Vérifier
SELECT
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'myapp';
```

### Tables

```sql
-- Table avec charset par défaut (utf8mb4)
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);

-- Table avec collation spécifique
CREATE TABLE products_fr (
    id INT PRIMARY KEY,
    name VARCHAR(200) COLLATE utf8mb4_uca1400_french_ai_ci,
    description TEXT COLLATE utf8mb4_uca1400_french_ai_ci
);

-- Colonne binaire (comparaison stricte)
CREATE TABLE passwords (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    password_hash VARCHAR(255) COLLATE utf8mb4_bin  -- Binaire !
);

-- Vérifier
SHOW CREATE TABLE users\G
```

### Colonnes individuelles

```sql
-- Différentes collations par colonne
CREATE TABLE content (
    id INT PRIMARY KEY,
    title_en VARCHAR(200) COLLATE utf8mb4_uca1400_ai_ci,
    title_fr VARCHAR(200) COLLATE utf8mb4_uca1400_french_ai_ci,
    title_de VARCHAR(200) COLLATE utf8mb4_uca1400_german2_ai_ci,
    content TEXT COLLATE utf8mb4_uca1400_ai_ci,
    tags VARCHAR(500) COLLATE utf8mb4_bin  -- Tags sensibles à la casse
);
```

---

## Migration vers utf8mb4

### Évaluation pré-migration

```sql
-- Identifier les bases/tables NON utf8mb4
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_COLLATION,
    ENGINE
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND TABLE_COLLATION NOT LIKE 'utf8mb4%'
ORDER BY TABLE_SCHEMA, TABLE_NAME;

-- Identifier les colonnes NON utf8mb4
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    COLUMN_NAME,
    CHARACTER_SET_NAME,
    COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND CHARACTER_SET_NAME IS NOT NULL
    AND CHARACTER_SET_NAME != 'utf8mb4'
ORDER BY TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME;
```

### Migration base de données

```sql
-- Modifier le charset par défaut d'une base
ALTER DATABASE myapp
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_ai_ci;

-- ⚠️ Cela affecte UNIQUEMENT les nouvelles tables !
-- Les tables existantes ne sont PAS modifiées
```

### Migration table par table

```sql
-- Méthode 1 : ALTER TABLE (simple mais locks)
ALTER TABLE users
    CONVERT TO CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_ai_ci;

-- Méthode 2 : ALTER TABLE colonne par colonne (contrôle fin)
ALTER TABLE users
    MODIFY COLUMN name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci,
    MODIFY COLUMN email VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- Vérifier
SHOW CREATE TABLE users\G
```

### Migration sans downtime (pt-online-schema-change)

```bash
# Percona Toolkit pour migration sans lock
pt-online-schema-change \
    --alter "CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci" \
    --execute \
    h=localhost,D=myapp,t=users
```

### Script de migration complet

```bash
#!/bin/bash
# migrate-to-utf8mb4.sh

DB_NAME="myapp"
LOG_FILE="/var/log/mysql/migration-utf8mb4-$(date +%Y%m%d).log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Migration vers utf8mb4 ==="

# 1. Backup complet
log "Backup de la base $DB_NAME"
mysqldump --single-transaction "$DB_NAME" > "/backup/$DB_NAME-pre-utf8mb4.sql"

# 2. Modifier base de données
log "Modification du charset de la base"
mariadb -e "ALTER DATABASE $DB_NAME CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci"

# 3. Lister les tables à migrer
TABLES=$(mariadb -sN -e "
    SELECT TABLE_NAME
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '$DB_NAME'
    AND TABLE_COLLATION NOT LIKE 'utf8mb4%'
")

# 4. Migrer chaque table
for table in $TABLES; do
    log "Migration de $table"
    mariadb -e "
        ALTER TABLE $DB_NAME.$table
        CONVERT TO CHARACTER SET utf8mb4
        COLLATE utf8mb4_uca1400_ai_ci
    " 2>&1 | tee -a "$LOG_FILE"
done

log "=== Migration terminée ==="

# 5. Rapport
log "Rapport post-migration :"
mariadb -e "
    SELECT
        TABLE_NAME,
        TABLE_COLLATION
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '$DB_NAME'
    ORDER BY TABLE_NAME
" | tee -a "$LOG_FILE"
```

---

## Gestion des emoji et caractères spéciaux

### Support natif des emoji 🆕

```sql
-- Avec utf8mb4 : Emoji nativement supportés
INSERT INTO posts (content) VALUES
    ('Hello 👋 World 🌍'),
    ('I love coding 💻🚀'),
    ('Great job! 🎉👏');

-- Recherche
SELECT * FROM posts WHERE content LIKE '%🎉%';

-- Tri (emoji triés par code Unicode)
SELECT content FROM posts ORDER BY content;
```

### Caractères rares

```sql
-- Mathématiques
INSERT INTO formulas (formula) VALUES
    ('E = mc²'),
    ('∫f(x)dx'),
    ('∑i=1→n');

-- Alphabets anciens
INSERT INTO historical_texts (text) VALUES
    ('𐐀𐐨 (Deseret)'),
    ('𓀀𓀁 (Hieroglyphes)');

-- Symboles monétaires rares
INSERT INTO currencies (symbol) VALUES
    ('₿'),  -- Bitcoin
    ('₽'),  -- Rouble
    ('₹');  -- Roupie
```

### Longueurs et tailles

```sql
-- Longueur en caractères vs octets
SELECT
    'Café' AS text,
    CHAR_LENGTH('Café') AS char_length,  -- 4 caractères
    LENGTH('Café') AS byte_length;       -- 5 octets (é = 2 octets)

SELECT
    '👋🌍' AS text,
    CHAR_LENGTH('👋🌍') AS char_length,  -- 2 caractères
    LENGTH('👋🌍') AS byte_length;       -- 8 octets (chaque emoji = 4 octets)
```

**Impact sur VARCHAR** :

```sql
-- VARCHAR(100) avec utf8mb4
-- = Maximum 100 CARACTÈRES
-- = Maximum 400 OCTETS (100 * 4)

CREATE TABLE messages (
    content VARCHAR(100) CHARACTER SET utf8mb4  -- Max 100 chars, 400 bytes
);

-- OK : 100 caractères ASCII (100 octets)
INSERT INTO messages VALUES (REPEAT('a', 100));

-- OK : 100 emoji (400 octets)
INSERT INTO messages VALUES (REPEAT('🎉', 100));

-- ❌ ERREUR : 101 caractères
INSERT INTO messages VALUES (REPEAT('🎉', 101));
```

---

## Impact sur les performances

### Taille des index

**Limite de longueur d'index InnoDB** : 767 octets (défaut) ou 3072 octets (avec `innodb_large_prefix`).

```sql
-- Avec utf8mb3 (3 octets max)
-- VARCHAR(255) = 255 * 3 = 765 octets ✅ OK

-- Avec utf8mb4 (4 octets max)
-- VARCHAR(255) = 255 * 4 = 1020 octets ❌ DÉPASSE 767 octets !

-- Solution 1 : Réduire la longueur
CREATE TABLE users (
    email VARCHAR(191) CHARACTER SET utf8mb4,  -- 191 * 4 = 764 octets ✅
    INDEX idx_email (email)
);

-- Solution 2 : Index partiel
CREATE TABLE users (
    email VARCHAR(255) CHARACTER SET utf8mb4,
    INDEX idx_email (email(191))  -- Index sur 191 premiers caractères
);

-- Solution 3 : Activer innodb_large_prefix (défaut depuis MariaDB 10.2)
-- Permet index jusqu'à 3072 octets
SET GLOBAL innodb_large_prefix = ON;
SET GLOBAL innodb_file_format = Barracuda;
SET GLOBAL innodb_file_per_table = ON;

CREATE TABLE users (
    email VARCHAR(255) CHARACTER SET utf8mb4,
    INDEX idx_email (email)
) ROW_FORMAT=DYNAMIC;  -- ou COMPRESSED
```

### Performance de tri et comparaison

```sql
-- Benchmark : Tri 1M lignes
-- utf8mb4_bin : 1.2 secondes (comparaison binaire simple)
-- utf8mb4_uca1400_ai_ci : 2.5 secondes (algorithme UCA complexe)

-- Pour performance maximale sur colonnes de tri fréquent
CREATE TABLE logs (
    id INT PRIMARY KEY,
    timestamp DATETIME,
    level VARCHAR(20) COLLATE utf8mb4_bin,  -- Binaire pour performance
    message TEXT COLLATE utf8mb4_uca1400_ai_ci
);
```

### Taille en mémoire et disque

```sql
-- Comparaison taille stockage (1M lignes)
-- latin1 : Nom VARCHAR(100) = 100 MB
-- utf8mb3 : Nom VARCHAR(100) = 300 MB (+200%)
-- utf8mb4 : Nom VARCHAR(100) = 400 MB (+300%)

-- Mais en pratique (texte occidental) :
-- utf8mb4 : ~100-150 MB (peu de caractères multi-octets)
```

---

## Problèmes courants et solutions

### Problème : Caractères mal affichés (mojibake)

**Symptômes** :
```
Attendu : "Café"
Affiché : "CafÃ©"
```

**Causes** :
1. Client utilise un charset différent
2. Connexion mal configurée
3. Données mal encodées initialement

**Diagnostic** :

```sql
-- Vérifier les charsets de connexion
SHOW VARIABLES LIKE 'character_set%';

-- Afficher en hexadécimal
SELECT HEX(name) FROM users WHERE name LIKE '%Café%';
-- Correct : 436166C3A9 (UTF-8)
-- Incorrect : 43616665 + C383C2A9 (double encodage)
```

**Solution** :

```sql
-- Côté client : Forcer utf8mb4
SET NAMES utf8mb4;

-- Ou dans l'application (PHP)
mysqli_set_charset($conn, 'utf8mb4');

-- Ou dans la chaîne de connexion
jdbc:mysql://localhost/myapp?characterEncoding=UTF-8
```

### Problème : "Illegal mix of collations"

**Symptôme** :

```sql
SELECT * FROM users u
JOIN posts p ON u.name = p.author
WHERE u.name = 'Jean';

-- ERROR 1267 (HY000): Illegal mix of collations
-- (utf8mb4_uca1400_ai_ci,IMPLICIT) and (utf8mb4_unicode_ci,IMPLICIT)
```

**Cause** : Colonnes avec collations différentes comparées.

**Solution** :

```sql
-- Option 1 : Forcer la collation dans la requête
SELECT * FROM users u
JOIN posts p ON u.name COLLATE utf8mb4_uca1400_ai_ci = p.author
WHERE u.name = 'Jean';

-- Option 2 : Modifier la table pour harmoniser
ALTER TABLE posts
    MODIFY COLUMN author VARCHAR(100) COLLATE utf8mb4_uca1400_ai_ci;
```

### Problème : Emoji tronqués

**Symptôme** :

```sql
INSERT INTO posts (content) VALUES ('Hello 👋');
SELECT content FROM posts;
-- Résultat : "Hello ?" ou "Hello "
```

**Cause** : Colonne en utf8mb3 (3 octets max).

**Solution** :

```sql
-- Convertir en utf8mb4
ALTER TABLE posts
    MODIFY COLUMN content TEXT CHARACTER SET utf8mb4;
```

---

## Bonnes pratiques

### ✅ À FAIRE

1. **Toujours utf8mb4** pour nouvelles applications (défaut 11.8)
2. **utf8mb4_uca1400_ai_ci** comme collation par défaut
3. **VARCHAR(191)** max pour colonnes indexées (sauf large_prefix actif)
4. **SET NAMES utf8mb4** côté client
5. **Tester avec emoji** (👋🌍) lors du développement
6. **Harmoniser collations** entre tables jointes
7. **utf8mb4_bin** pour colonnes sensibles (mots de passe hash, tokens)
8. **Documenter** le choix de collation par langue

### ❌ À ÉVITER

1. **"utf8" sans mb4** : Limité à 3 octets
2. **Mélanger collations** dans les jointures
3. **Oublier SET NAMES** côté application
4. **VARCHAR(255)** pour colonnes indexées sans large_prefix
5. **latin1** pour nouvelles applications (obsolète)
6. **Supposer 1 caractère = 1 octet** (faux en UTF-8)
7. **Ignorer les emoji** dans les tests
8. **Migrer sans backup** complet

---

## Checklist migration vers utf8mb4

### Pré-migration

- [ ] Backup complet de toutes les bases
- [ ] Inventaire des bases/tables/colonnes non-utf8mb4
- [ ] Identification des colonnes indexées > 191 caractères
- [ ] Test sur environnement staging
- [ ] Vérification version MariaDB ≥ 10.6 (UCA 14.0.0)

### Migration

- [ ] `ALTER DATABASE ... CHARACTER SET utf8mb4`
- [ ] `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4` (toutes tables)
- [ ] Vérification intégrité données (SELECT après migration)
- [ ] Test complet application

### Post-migration

- [ ] `SET NAMES utf8mb4` dans applications
- [ ] Configuration my.cnf (character-set-server = utf8mb4)
- [ ] Surveillance performances (index, tris)
- [ ] Documentation mise à jour
- [ ] Formation équipes (emoji supportés !)

---

## Exemples par cas d'usage

### Application web multilingue

```sql
CREATE DATABASE webapp
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_ai_ci;

USE webapp;

CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(191) UNIQUE,  -- Index possible
    username VARCHAR(50),
    display_name VARCHAR(100),
    bio TEXT,
    created_at DATETIME
) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

CREATE TABLE posts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    title VARCHAR(200),
    content TEXT,
    reactions VARCHAR(500),  -- Emoji stockés ici : 👍😀❤️
    created_at DATETIME,
    FOREIGN KEY (user_id) REFERENCES users(id)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

### Site e-commerce français

```sql
CREATE DATABASE shop_fr
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_uca1400_french_ai_ci;

USE shop_fr;

CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200),
    description TEXT,
    price DECIMAL(10,2),
    -- Collation française pour tri alphabétique correct
    -- (é trié après e, etc.)
    INDEX idx_name (name)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_french_ai_ci;

-- Recherche : "café" trouve "Café", "CAFÉ", "cafe"
SELECT * FROM products WHERE name LIKE '%café%';
```

### Application avec tokens/hash

```sql
CREATE TABLE api_tokens (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    token VARCHAR(255) COLLATE utf8mb4_bin,  -- Binaire, sensible casse
    created_at DATETIME,
    expires_at DATETIME,
    UNIQUE KEY idx_token (token)
);

-- Comparaison stricte (sensible casse)
SELECT * FROM api_tokens WHERE token = 'AbC123XyZ';
-- Trouve UNIQUEMENT 'AbC123XyZ', pas 'abc123xyz'
```

---

## ✅ Points clés à retenir

- 🆕 **MariaDB 11.8** : utf8mb4 et utf8mb4_uca1400_ai_ci par défaut
- **utf8mb4** : Vrai UTF-8 (4 octets), supporte emoji et tous caractères Unicode
- **utf8mb3** : Ancien "utf8", 3 octets max, ❌ pas d'emoji
- **UCA 14.0.0** : Unicode Collation Algorithm, version septembre 2021
- **Collations** : ai (accent-insensitive), ci (case-insensitive)
- **VARCHAR(191)** : Max pour index sans innodb_large_prefix
- **SET NAMES utf8mb4** : Obligatoire côté client
- **Migration** : ALTER TABLE CONVERT TO CHARACTER SET utf8mb4
- **Performance** : utf8mb4_bin pour comparaisons simples
- **Emoji** : Nativement supportés avec utf8mb4
- **Langues** : Collations spécifiques (french, german, spanish, etc.)
- **Backup** : Toujours avant migration charset

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Character Sets and Collations](https://mariadb.com/kb/en/character-sets-and-collations/)
- [📖 Documentation officielle - Unicode Support](https://mariadb.com/kb/en/unicode/)
- [📖 Unicode Standard 14.0](https://www.unicode.org/versions/Unicode14.0.0/)
- [📖 Unicode Collation Algorithm (UCA)](https://www.unicode.org/reports/tr10/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
- [🔧 Emoji List](https://unicode.org/emoji/charts/full-emoji-list.html)

---

## ➡️ Section suivante

**[11.12 Extension TIMESTAMP 2106](./12-extension-timestamp-2106.md)** 🆕 : Résolution du problème Y2038 avec support jusqu'en 2106.

---

**💡 Conseil final** : utf8mb4 n'est pas juste un charset, c'est une **posture moderne** face à l'internationalisation. Avec MariaDB 11.8, vous êtes prêt pour le monde entier, des emoji japonais 🎌 aux hiéroglyphes égyptiens 𓀀. Adoptez-le sans hésitation ! 🌍🚀

⏭️ [Extension TIMESTAMP 2038→2106 (problème Y2038 résolu)](/11-administration-configuration/12-extension-timestamp-2106.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.11 Expressions régulières (REGEXP, REGEXP_REPLACE, REGEXP_SUBSTR)

> **Niveau** : Avancé
> **Durée estimée** : 2-3 heures
> **Prérequis** : Connaissance de base des regex, fonctions de chaînes SQL

> **Version** : MariaDB 10.0+ (améliorations 11.x)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Maîtriser la **syntaxe des regex** dans MariaDB
- Utiliser **REGEXP/RLIKE** pour filtrer et valider des données
- Appliquer **REGEXP_REPLACE** pour nettoyer et transformer des chaînes
- Extraire des données avec **REGEXP_SUBSTR** et **REGEXP_INSTR**
- Construire des **patterns complexes** (groupes, classes, quantificateurs)
- Valider des **formats** (email, téléphone, URL, codes postaux)
- Optimiser les **performances** des regex
- Éviter les **pièges courants** et erreurs de sécurité

---

## Introduction

### Qu'est-ce qu'une expression régulière ?

Une **expression régulière (regex)** est un **pattern de recherche** qui permet de trouver, valider ou extraire des chaînes de caractères selon des règles complexes.

**Exemples simples** :
- `^[0-9]{5}$` : Exactement 5 chiffres (code postal)
- `@[a-z]+\.[a-z]{2,}` : Domaine email basique
- `[A-Z]{2}-[0-9]{6}` : Format SKU (AB-123456)

### Pourquoi utiliser les regex en SQL ?

✅ **Avantages** :
- **Validation** : Vérifier formats de données (email, téléphone, IBAN)
- **Nettoyage** : Supprimer ou remplacer patterns indésirables
- **Extraction** : Isoler des informations dans du texte non structuré
- **Recherche avancée** : Filtres complexes impossibles avec LIKE
- **Transformation** : Reformater des données en masse

**Comparaison LIKE vs REGEXP** :

```sql
-- LIKE : Patterns simples avec % et _
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- REGEXP : Patterns complexes
SELECT * FROM users WHERE email REGEXP '^[a-z0-9._%+-]+@gmail\.com$';
```

💡 **LIKE** = Simple, rapide pour patterns basiques
💡 **REGEXP** = Puissant, nécessaire pour patterns complexes

---

## Syntaxe de base

### Opérateur REGEXP / RLIKE

```sql
-- Syntaxe : column REGEXP 'pattern'
-- Retourne 1 (TRUE) si match, 0 (FALSE) sinon

SELECT 'hello' REGEXP 'h';           -- 1 (match)
SELECT 'hello' REGEXP '^h';          -- 1 (commence par h)
SELECT 'hello' REGEXP 'o$';          -- 1 (finit par o)
SELECT 'hello' REGEXP 'world';       -- 0 (pas de match)

-- RLIKE est un alias de REGEXP
SELECT 'hello' RLIKE 'h';            -- 1 (identique à REGEXP)
```

### NOT REGEXP

```sql
-- Négation
SELECT 'hello' NOT REGEXP '[0-9]';   -- 1 (pas de chiffre)
SELECT 'hello123' NOT REGEXP '[0-9]'; -- 0 (contient des chiffres)
```

### Sensibilité à la casse

```sql
-- Par défaut : insensible à la casse
SELECT 'Hello' REGEXP 'hello';       -- 1 (match)

-- Forcer la casse : REGEXP BINARY
SELECT 'Hello' REGEXP BINARY 'hello'; -- 0 (pas de match)
SELECT 'Hello' REGEXP BINARY 'Hello'; -- 1 (match exact)
```

---

## Métacaractères et quantificateurs

### Ancres

| Symbole | Signification | Exemple | Matches |
|---------|---------------|---------|---------|
| `^` | Début de chaîne | `^hello` | "hello world" ✅ |
| `$` | Fin de chaîne | `world$` | "hello world" ✅ |
| `^...$` | Chaîne exacte | `^hello$` | Seulement "hello" |

```sql
-- Commence par "user"
SELECT username FROM users WHERE username REGEXP '^user';

-- Finit par ".com"
SELECT email FROM users WHERE email REGEXP '\.com$';

-- Exactement 5 caractères
SELECT code FROM products WHERE code REGEXP '^.....$';
-- Ou mieux : '^.{5}$'
```

### Classes de caractères

| Classe | Signification | Équivalent | Exemple |
|--------|---------------|------------|---------|
| `[abc]` | a, b ou c | - | `[aeiou]` = voyelle |
| `[^abc]` | Tout sauf a, b, c | - | `[^0-9]` = non-chiffre |
| `[a-z]` | Lettres minuscules | - | `[a-zA-Z]` = lettre |
| `[0-9]` | Chiffres | `\d` (non standard) | `[0-9]{3}` = 3 chiffres |
| `.` | N'importe quel caractère | - | `a.c` = "abc", "a1c" |

```sql
-- Email avec domaine .com, .fr ou .org
SELECT email FROM users
WHERE email REGEXP '@[a-z]+\.(com|fr|org)$';

-- Code commençant par une lettre majuscule puis 5 chiffres
SELECT code FROM products
WHERE code REGEXP '^[A-Z][0-9]{5}$';

-- Téléphone avec séparateurs optionnels
SELECT phone FROM contacts
WHERE phone REGEXP '^[0-9]{2}[- ]?[0-9]{2}[- ]?[0-9]{2}[- ]?[0-9]{2}[- ]?[0-9]{2}$';
```

### Quantificateurs

| Quantificateur | Signification | Exemple | Matches |
|----------------|---------------|---------|---------|
| `*` | 0 ou plus | `ab*c` | "ac", "abc", "abbc" |
| `+` | 1 ou plus | `ab+c` | "abc", "abbc" (pas "ac") |
| `?` | 0 ou 1 | `ab?c` | "ac", "abc" (pas "abbc") |
| `{n}` | Exactement n | `[0-9]{3}` | "123" (3 chiffres) |
| `{n,}` | Au moins n | `[0-9]{3,}` | "123", "1234", "12345" |
| `{n,m}` | Entre n et m | `[0-9]{3,5}` | "123", "1234", "12345" |

```sql
-- Code postal français : 5 chiffres
SELECT * FROM addresses WHERE zipcode REGEXP '^[0-9]{5}$';

-- Nom de 3 à 20 lettres
SELECT * FROM users WHERE name REGEXP '^[a-zA-Z]{3,20}$';

-- URL http:// ou https:// (s optionnel)
SELECT * FROM links WHERE url REGEXP '^https?://';

-- Téléphone international : +33 puis 9 chiffres
SELECT * FROM contacts WHERE phone REGEXP '^\+33[0-9]{9}$';
```

### Alternation et groupes

```sql
-- Alternation avec |
SELECT * FROM products
WHERE category REGEXP '^(electronics|books|clothing)$';

-- Groupes avec ()
SELECT * FROM users
WHERE email REGEXP '^[a-z]+@(gmail|yahoo|hotmail)\.com$';

-- Groupes optionnels
SELECT * FROM addresses
WHERE street REGEXP '^[0-9]+[a-z]? (rue|avenue|boulevard) ';
-- Matches : "10 rue", "10a rue", "15 avenue"
```

---

## REGEXP_REPLACE : Remplacement avec regex

### Syntaxe

```sql
REGEXP_REPLACE(string, pattern, replacement [, position [, occurrence [, match_type]]])
```

**Paramètres** :
- `string` : Chaîne source
- `pattern` : Regex à matcher
- `replacement` : Chaîne de remplacement
- `position` : Position de départ (défaut: 1)
- `occurrence` : Quelle occurrence remplacer (0 = toutes, défaut: 0)
- `match_type` : Options (i = insensible casse, c = sensible, m = multiline)

### Exemples de base

```sql
-- Supprimer tous les chiffres
SELECT REGEXP_REPLACE('hello123world456', '[0-9]+', '');
-- Résultat : 'helloworld'

-- Remplacer espaces multiples par un seul
SELECT REGEXP_REPLACE('hello    world', ' +', ' ');
-- Résultat : 'hello world'

-- Masquer les chiffres d'une carte de crédit
SELECT REGEXP_REPLACE('1234-5678-9012-3456', '[0-9]', '*');
-- Résultat : '****-****-****-****'
```

### Nettoyage de données

```sql
-- Supprimer caractères non-alphanumériques
UPDATE users
SET username = REGEXP_REPLACE(username, '[^a-zA-Z0-9]', '')
WHERE username REGEXP '[^a-zA-Z0-9]';

-- Normaliser numéros de téléphone (garder seulement chiffres)
UPDATE contacts
SET phone = REGEXP_REPLACE(phone, '[^0-9]', '')
WHERE phone IS NOT NULL;

-- Supprimer espaces en début/fin et multiples
UPDATE products
SET name = REGEXP_REPLACE(TRIM(name), ' {2,}', ' ');

-- Supprimer tags HTML
UPDATE articles
SET content_text = REGEXP_REPLACE(content_html, '<[^>]+>', '');
```

### Reformatage de données

```sql
-- Téléphone : 0612345678 → 06 12 34 56 78
SELECT REGEXP_REPLACE('0612345678',
    '([0-9]{2})([0-9]{2})([0-9]{2})([0-9]{2})([0-9]{2})',
    '\\1 \\2 \\3 \\4 \\5'
);
-- Résultat : '06 12 34 56 78'

-- Date : 20251215 → 2025-12-15
SELECT REGEXP_REPLACE('20251215',
    '([0-9]{4})([0-9]{2})([0-9]{2})',
    '\\1-\\2-\\3'
);
-- Résultat : '2025-12-15'

-- SKU : ABCD1234 → AB-CD-1234
SELECT REGEXP_REPLACE('ABCD1234',
    '([A-Z]{2})([A-Z]{2})([0-9]{4})',
    '\\1-\\2-\\3'
);
-- Résultat : 'AB-CD-1234'
```

### Groupes de capture et backreferences

```sql
-- Inverser prénom nom : "Alice Martin" → "Martin, Alice"
SELECT REGEXP_REPLACE('Alice Martin',
    '^([A-Z][a-z]+) ([A-Z][a-z]+)$',
    '\\2, \\1'
);
-- Résultat : 'Martin, Alice'

-- Extraire domaine d'email : user@example.com → example.com
SELECT REGEXP_REPLACE('user@example.com',
    '^[^@]+@(.+)$',
    '\\1'
);
-- Résultat : 'example.com'
```

---

## REGEXP_SUBSTR : Extraction avec regex

### Syntaxe

```sql
REGEXP_SUBSTR(string, pattern [, position [, occurrence [, match_type]]])
```

**Retourne** : La première sous-chaîne qui match le pattern.

### Exemples d'extraction

```sql
-- Extraire le premier mot
SELECT REGEXP_SUBSTR('hello world test', '[a-z]+');
-- Résultat : 'hello'

-- Extraire un nombre
SELECT REGEXP_SUBSTR('Prix: 1299.99 euros', '[0-9]+\.[0-9]+');
-- Résultat : '1299.99'

-- Extraire email
SELECT REGEXP_SUBSTR('Contact: john@example.com pour info',
    '[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}'
);
-- Résultat : 'john@example.com'

-- Extraire code postal français
SELECT REGEXP_SUBSTR('Adresse: 75001 Paris', '[0-9]{5}');
-- Résultat : '75001'
```

### Extraire occurrence spécifique

```sql
-- Deuxième occurrence
SELECT REGEXP_SUBSTR('hello world test', '[a-z]+', 1, 2);
-- Résultat : 'world'

-- Troisième occurrence
SELECT REGEXP_SUBSTR('hello world test', '[a-z]+', 1, 3);
-- Résultat : 'test'
```

### Extraction dans une requête

```sql
CREATE TABLE logs (
    id INT PRIMARY KEY,
    message TEXT
);

INSERT INTO logs VALUES
(1, 'User alice logged in from 192.168.1.100'),
(2, 'Failed login attempt from 10.0.0.50'),
(3, 'User bob logged in from 172.16.0.25');

-- Extraire adresses IP
SELECT
    id,
    message,
    REGEXP_SUBSTR(message, '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}') AS ip_address
FROM logs;
```

**Résultat** :
```
+----+----------------------------------------------+---------------+
| id | message                                      | ip_address    |
+----+----------------------------------------------+---------------+
|  1 | User alice logged in from 192.168.1.100      | 192.168.1.100 |
|  2 | Failed login attempt from 10.0.0.50          | 10.0.0.50     |
|  3 | User bob logged in from 172.16.0.25          | 172.16.0.25   |
+----+----------------------------------------------+---------------+
```

---

## REGEXP_INSTR : Trouver la position

### Syntaxe

```sql
REGEXP_INSTR(string, pattern [, position [, occurrence [, return_option [, match_type]]]])
```

**Retourne** : Position du match (ou 0 si pas de match).

### Exemples

```sql
-- Position du premier chiffre
SELECT REGEXP_INSTR('hello123world', '[0-9]');
-- Résultat : 6

-- Position du premier espace
SELECT REGEXP_INSTR('hello world test', ' ');
-- Résultat : 6

-- Position du deuxième mot
SELECT REGEXP_INSTR('hello world test', '[a-z]+', 1, 2);
-- Résultat : 7 (position de 'world')
```

---

## Cas d'usage pratiques

### Exemple 1 : Validation de formats

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20),
    postal_code VARCHAR(10),

    -- Contraintes de validation avec REGEXP
    CONSTRAINT chk_username CHECK (
        username REGEXP '^[a-zA-Z0-9_]{3,20}$'
    ),
    CONSTRAINT chk_email CHECK (
        email REGEXP '^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$'
    ),
    CONSTRAINT chk_phone CHECK (
        phone REGEXP '^(\\+33|0)[0-9]{9}$'
    ),
    CONSTRAINT chk_postal CHECK (
        postal_code REGEXP '^[0-9]{5}$'
    )
);

-- ✅ INSERT valide
INSERT INTO users (username, email, phone, postal_code) VALUES
('alice_92', 'alice@example.com', '0612345678', '75001');

-- ❌ INSERT invalide : email incorrect
INSERT INTO users (username, email, phone, postal_code) VALUES
('bob', 'invalid-email', '0612345678', '75001');
-- ERROR: Check constraint 'chk_email' violated
```

### Exemple 2 : Nettoyage de données importées

```sql
-- Table de données brutes importées
CREATE TABLE raw_contacts (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    phone VARCHAR(50),
    email VARCHAR(100)
);

-- Données sales
INSERT INTO raw_contacts VALUES
(1, '  Alice   Martin  ', '+33 6 12 34 56 78', 'ALICE@EXAMPLE.COM  '),
(2, 'Bob<script>alert(1)</script>', '06-12-34-56-78', '  bob@test.com'),
(3, 'Carol123', '(+33)612345678', 'carol.jones@example.com');

-- Nettoyage avec REGEXP_REPLACE
CREATE TABLE clean_contacts AS
SELECT
    id,
    -- Nom : trim + espaces multiples → simple
    REGEXP_REPLACE(TRIM(name), ' {2,}', ' ') AS clean_name,
    -- Téléphone : garder seulement chiffres et +
    REGEXP_REPLACE(phone, '[^0-9+]', '') AS clean_phone,
    -- Email : trim + lowercase
    LOWER(TRIM(email)) AS clean_email
FROM raw_contacts
-- Supprimer injections HTML/JS
WHERE name NOT REGEXP '<script|<iframe|javascript:';
```

**Résultat** :
```
+----+---------------+--------------+--------------------------+
| id | clean_name    | clean_phone  | clean_email              |
+----+---------------+--------------+--------------------------+
|  1 | Alice Martin  | +33612345678 | alice@example.com        |
|  2 | Bob           | 0612345678   | bob@test.com             |
|  3 | Carol123      | +33612345678 | carol.jones@example.com  |
+----+---------------+--------------+--------------------------+
```

### Exemple 3 : Parsing de logs

```sql
CREATE TABLE server_logs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    log_line TEXT,
    parsed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO server_logs (log_line) VALUES
('[2025-12-12 14:23:45] ERROR: Connection timeout to 192.168.1.100:3306'),
('[2025-12-12 14:25:12] INFO: User admin logged in from 10.0.0.50'),
('[2025-12-12 14:30:01] WARNING: High memory usage: 85%');

-- Extraire les composants du log
SELECT
    id,
    -- Date
    REGEXP_SUBSTR(log_line, '[0-9]{4}-[0-9]{2}-[0-9]{2}') AS log_date,
    -- Heure
    REGEXP_SUBSTR(log_line, '[0-9]{2}:[0-9]{2}:[0-9]{2}') AS log_time,
    -- Niveau (ERROR, INFO, WARNING)
    REGEXP_SUBSTR(log_line, '(ERROR|INFO|WARNING)') AS log_level,
    -- IP si présente
    REGEXP_SUBSTR(log_line, '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}') AS ip_address,
    -- Message (après le niveau)
    REGEXP_REPLACE(log_line, '^\\[[^\\]]+\\] (ERROR|INFO|WARNING): ', '') AS message
FROM server_logs;
```

**Résultat** :
```
+----+------------+----------+-----------+---------------+--------------------------------------+
| id | log_date   | log_time | log_level | ip_address    | message                              |
+----+------------+----------+-----------+---------------+--------------------------------------+
|  1 | 2025-12-12 | 14:23:45 | ERROR     | 192.168.1.100 | Connection timeout to 192.168...     |
|  2 | 2025-12-12 | 14:25:12 | INFO      | 10.0.0.50     | User admin logged in from 10.0.0.50  |
|  3 | 2025-12-12 | 14:30:01 | WARNING   | NULL          | High memory usage: 85%               |
+----+------------+----------+-----------+---------------+--------------------------------------+
```

### Exemple 4 : Validation et reformatage d'identifiants

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    raw_sku VARCHAR(50),
    normalized_sku VARCHAR(20) AS (
        CASE
            -- Format: AB-123456 (déjà OK)
            WHEN raw_sku REGEXP '^[A-Z]{2}-[0-9]{6}$' THEN raw_sku
            -- Format: AB123456 (ajouter tiret)
            WHEN raw_sku REGEXP '^[A-Z]{2}[0-9]{6}$' THEN
                REGEXP_REPLACE(raw_sku, '^([A-Z]{2})([0-9]{6})$', '\\1-\\2')
            -- Format: ab-123456 (uppercase)
            WHEN raw_sku REGEXP '^[a-z]{2}-[0-9]{6}$' THEN
                UPPER(raw_sku)
            -- Invalide
            ELSE NULL
        END
    ) STORED,
    is_valid BOOLEAN AS (
        raw_sku REGEXP '^[A-Za-z]{2}-?[0-9]{6}$'
    ) STORED
);

INSERT INTO products (raw_sku) VALUES
('AB-123456'),  -- OK
('CD987654'),   -- À normaliser
('ef-555555'),  -- À uppercase
('INVALID123'); -- Invalide

SELECT raw_sku, normalized_sku, is_valid FROM products;
```

**Résultat** :
```
+-------------+-----------------+----------+
| raw_sku     | normalized_sku  | is_valid |
+-------------+-----------------+----------+
| AB-123456   | AB-123456       |        1 |
| CD987654    | CD-987654       |        1 |
| ef-555555   | EF-555555       |        1 |
| INVALID123  | NULL            |        0 |
+-------------+-----------------+----------+
```

---

## Patterns courants

### Validation email

```sql
-- Basic (simple mais pas RFC-compliant)
email REGEXP '^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$'

-- Avec domaines spécifiques
email REGEXP '^[a-z0-9._%+-]+@(gmail|yahoo|hotmail)\.com$'

-- Plus strict
email REGEXP '^[a-z0-9][a-z0-9._%+-]*@[a-z0-9][a-z0-9.-]*\.[a-z]{2,}$'
```

### Validation téléphone

```sql
-- France : 0X XX XX XX XX
phone REGEXP '^0[1-9][0-9]{8}$'

-- International : +33 X XX XX XX XX
phone REGEXP '^\\+33[1-9][0-9]{8}$'

-- Flexible avec séparateurs
phone REGEXP '^(\\+33|0)[1-9]([0-9]{2}[- ]?){4}$'
```

### Validation URL

```sql
-- HTTP(S) basique
url REGEXP '^https?://[a-z0-9.-]+\\.[a-z]{2,}'

-- Avec chemin et paramètres
url REGEXP '^https?://[a-z0-9.-]+\\.[a-z]{2,}(/[^\\s]*)?$'
```

### Validation code postal

```sql
-- France : 5 chiffres
postal_code REGEXP '^[0-9]{5}$'

-- UK : AA9A 9AA
postal_code REGEXP '^[A-Z]{1,2}[0-9][A-Z0-9]? ?[0-9][A-Z]{2}$'

-- USA : 12345 ou 12345-6789
postal_code REGEXP '^[0-9]{5}(-[0-9]{4})?$'
```

### Validation carte de crédit

```sql
-- Visa : commence par 4, 13-16 chiffres
card REGEXP '^4[0-9]{12}([0-9]{3})?$'

-- MasterCard : commence par 51-55 ou 2221-2720
card REGEXP '^(5[1-5][0-9]{14}|2(22[1-9]|2[3-9][0-9]|[3-6][0-9]{2}|7[01][0-9]|720)[0-9]{12})$'
```

---

## Performance et optimisations

### Impact sur les performances

Les regex sont **plus lentes** que les opérations simples :

```sql
-- ⚡ RAPIDE : LIKE avec index
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- 🐌 LENT : REGEXP sans index (full scan)
SELECT * FROM users WHERE email REGEXP '@gmail\.com$';

-- ⚡ OPTIMAL : Colonne virtuelle + index
ALTER TABLE users
ADD COLUMN email_domain VARCHAR(100)
    AS (REGEXP_REPLACE(email, '^[^@]+@(.+)$', '\\1')) STORED;
CREATE INDEX idx_email_domain ON users(email_domain);

SELECT * FROM users WHERE email_domain = 'gmail.com';
```

### Optimisations

#### 1. Limiter le scope

```sql
-- ❌ LENT : Regex sur toute la table
SELECT * FROM logs
WHERE message REGEXP 'error|warning|critical';

-- ✅ RAPIDE : Filtrer d'abord avec index
SELECT * FROM logs
WHERE log_level IN ('ERROR', 'WARNING', 'CRITICAL')
  AND message REGEXP 'database|connection';
```

#### 2. Ancrer les patterns

```sql
-- 🐌 LENT : Cherche partout dans la chaîne
WHERE code REGEXP '[A-Z]{2}[0-9]{6}'

-- ⚡ RAPIDE : Ancré au début
WHERE code REGEXP '^[A-Z]{2}[0-9]{6}'
```

#### 3. Utiliser des colonnes virtuelles

```sql
-- Pour validation fréquente
ALTER TABLE users
ADD COLUMN email_valid BOOLEAN
    AS (email REGEXP '^[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}$') STORED;

CREATE INDEX idx_email_valid ON users(email_valid);

-- Requête rapide
SELECT COUNT(*) FROM users WHERE email_valid = 0;
```

### Benchmark

```sql
-- Test : 100,000 lignes
CREATE TABLE test_users (
    id INT PRIMARY KEY,
    email VARCHAR(100)
);

-- Test 1 : LIKE
SELECT COUNT(*) FROM test_users WHERE email LIKE '%@gmail.com';
-- Temps : ~50ms

-- Test 2 : REGEXP
SELECT COUNT(*) FROM test_users WHERE email REGEXP '@gmail\.com$';
-- Temps : ~200ms (4x plus lent)

-- Test 3 : Colonne virtuelle indexée
ALTER TABLE test_users
ADD COLUMN is_gmail BOOLEAN
    AS (email REGEXP '@gmail\.com$') STORED;
CREATE INDEX idx_gmail ON test_users(is_gmail);

SELECT COUNT(*) FROM test_users WHERE is_gmail = 1;
-- Temps : ~2ms (100x plus rapide que REGEXP direct)
```

---

## Pièges courants et solutions

### Piège 1 : Oublier d'échapper les caractères spéciaux

```sql
-- ❌ INCORRECT : . signifie "n'importe quel caractère"
WHERE email REGEXP '@example.com$'
-- Matches : alice@exampleXcom ⚠️

-- ✅ CORRECT : Échapper le .
WHERE email REGEXP '@example\\.com$'
-- Matches seulement : alice@example.com
```

**Caractères à échapper** : `. * + ? ^ $ { } [ ] ( ) | \`

### Piège 2 : Regex trop gourmands (catastrophic backtracking)

```sql
-- ❌ DANGER : Peut être très lent sur certaines chaînes
WHERE text REGEXP '(a+)+b'

-- ✅ SOLUTION : Simplifier le pattern
WHERE text REGEXP 'a+b'
```

### Piège 3 : Oublier les ancres

```sql
-- ❌ PROBLÈME : Match partiel
WHERE code REGEXP '[0-9]{5}'
-- Matches : "ABC12345XYZ" (pas exactement 5 chiffres)

-- ✅ CORRECT : Ancrer pour match exact
WHERE code REGEXP '^[0-9]{5}$'
-- Matches seulement : "12345"
```

### Piège 4 : Sensibilité à la casse inattendue

```sql
-- Par défaut : insensible à la casse
SELECT 'Hello' REGEXP 'hello';  -- 1

-- Pour forcer la casse
SELECT 'Hello' REGEXP BINARY 'hello';  -- 0
SELECT 'Hello' REGEXP BINARY 'Hello';  -- 1
```

### Piège 5 : NULL retourne NULL

```sql
SELECT NULL REGEXP '[0-9]+';  -- NULL (pas 0 !)

-- ✅ Gérer NULL explicitement
SELECT COALESCE(email REGEXP '@gmail\\.com$', 0) FROM users;
```

---

## Bonnes pratiques

### 1. Documenter les regex complexes

```sql
-- ❌ Illisible
ALTER TABLE users ADD CONSTRAINT chk CHECK (
    phone REGEXP '^(\\+33|0)[1-9]([0-9]{2}){4}$'
);

-- ✅ Documenté
ALTER TABLE users ADD CONSTRAINT chk_phone CHECK (
    -- Format téléphone français:
    -- +33 ou 0, puis chiffre 1-9, puis 8 chiffres
    -- Exemples: 0612345678, +33612345678
    phone REGEXP '^(\\+33|0)[1-9]([0-9]{2}){4}$'
);
```

### 2. Tester les regex avant déploiement

```sql
-- Test unitaire
SELECT
    '0612345678' REGEXP '^(\\+33|0)[1-9]([0-9]{2}){4}$' AS test1, -- 1
    '+33612345678' REGEXP '^(\\+33|0)[1-9]([0-9]{2}){4}$' AS test2, -- 1
    '0012345678' REGEXP '^(\\+33|0)[1-9]([0-9]{2}){4}$' AS test3, -- 0 (commence par 00)
    '061234567' REGEXP '^(\\+33|0)[1-9]([0-9]{2}){4}$' AS test4; -- 0 (trop court)
```

### 3. Utiliser des constantes pour patterns réutilisés

```sql
-- Dans l'application, définir des constantes
-- const REGEX_EMAIL = '^[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}$';
-- const REGEX_PHONE_FR = '^(\\+33|0)[1-9]([0-9]{2}){4}$';

-- Facilite maintenance et cohérence
```

### 4. Préférer la simplicité

```sql
-- ❌ Sur-complexe
WHERE name REGEXP '^[A-Za-z]+([ \'-][A-Za-z]+)*$'

-- ✅ Plus simple et suffisant
WHERE name REGEXP '^[A-Za-z \'-]+$'
  AND LENGTH(name) BETWEEN 2 AND 50
```

### 5. Valider côté application aussi

```sql
-- Ne pas compter UNIQUEMENT sur la base
-- Validation en couches :
-- 1. Frontend (UX rapide)
-- 2. Backend (sécurité)
-- 3. Base de données (intégrité finale)
```

---

## Alternatives aux regex

### Quand NE PAS utiliser regex

❌ **Éviter regex pour** :
- Recherche simple : Utiliser `LIKE`
- Extraction simple : Utiliser `SUBSTRING`, `LEFT`, `RIGHT`
- Validation simple : Utiliser `LENGTH`, `IN`, comparaisons

```sql
-- ❌ OVER-ENGINEERING
WHERE name REGEXP '^.{3,20}$'

-- ✅ PLUS SIMPLE
WHERE LENGTH(name) BETWEEN 3 AND 20

-- ❌ OVER-ENGINEERING
WHERE status REGEXP '^(active|inactive|pending)$'

-- ✅ PLUS SIMPLE
WHERE status IN ('active', 'inactive', 'pending')
```

### Fonctions alternatives

```sql
-- Au lieu de REGEXP_SUBSTR pour extraire domaine email
-- Option 1 : REGEXP
SELECT REGEXP_REPLACE(email, '^[^@]+@', '');

-- Option 2 : SUBSTRING_INDEX (plus rapide)
SELECT SUBSTRING_INDEX(email, '@', -1);
```

---

## ✅ Points clés à retenir

- 🔍 **REGEXP/RLIKE** : Filtrage avec patterns complexes
- 🔄 **REGEXP_REPLACE** : Nettoyage et transformation de chaînes
- 📤 **REGEXP_SUBSTR** : Extraction de sous-chaînes
- 📍 **REGEXP_INSTR** : Position d'un match
- ⚓ **Ancres** : `^` début, `$` fin pour validation stricte
- 🔢 **Quantificateurs** : `*`, `+`, `?`, `{n,m}` pour répétitions
- 🎯 **Classes** : `[a-z]`, `[0-9]`, `[^...]` pour ensembles
- ⚡ **Performance** : Colonnes virtuelles indexées pour requêtes fréquentes
- 🐌 **Lenteur** : Regex 4-10x plus lent que LIKE sans index
- 📝 **Documentation** : Toujours commenter regex complexes
- ✅ **Validation** : Email, téléphone, URL, codes postaux
- 🧹 **Nettoyage** : Normalisation, suppression HTML, parsing logs

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Regular Expressions Overview](https://mariadb.com/kb/en/regular-expressions-overview/)
- [📖 REGEXP](https://mariadb.com/kb/en/regexp/)
- [📖 REGEXP_REPLACE](https://mariadb.com/kb/en/regexp_replace/)
- [📖 REGEXP_SUBSTR](https://mariadb.com/kb/en/regexp_substr/)
- [📖 REGEXP_INSTR](https://mariadb.com/kb/en/regexp_instr/)

### Syntaxe regex
- [Regular-Expressions.info](https://www.regular-expressions.info/) - Tutoriels complets
- [Regex101](https://regex101.com/) - Testeur en ligne avec explications
- [RegExr](https://regexr.com/) - Visualiseur et testeur

### Patterns courants
- [HTML5 Pattern Attribute](https://html5pattern.com/) - Patterns validation formulaires
- [Regex Library](http://regexlib.com/) - Bibliothèque de patterns

---

## 🎉 Fin du chapitre 4 - Concepts Avancés SQL

Félicitations ! Vous avez maintenant une **maîtrise complète** des concepts SQL avancés dans MariaDB 11.8 :

- ✅ **4.1** : Requêtes récursives (WITH RECURSIVE)
- ✅ **4.2** : Window Functions
- ✅ **4.3** : Requêtes pivotées
- ✅ **4.4** : CTE (Common Table Expressions)
- ✅ **4.5** : Requêtes complexes multi-tables
- ✅ **4.6** : Gestion des valeurs NULL
- ✅ **4.7** : JSON dans MariaDB
- ✅ **4.8** : JSON Path Expressions 🆕
- ✅ **4.9** : JSON Schema Validation 🆕
- ✅ **4.10** : Indexation colonnes virtuelles JSON
- ✅ **4.11** : Expressions régulières

**Prochaine étape** : Chapitre 5 - Index et Performance

---


⏭️ [Index et Performance](/05-index-et-performance/README.md)

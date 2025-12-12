🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.7 Requêtes de sélection simples (SELECT, WHERE, ORDER BY, LIMIT)

> **Niveau** : Débutant
> **Durée estimée** : 2 heures
> **Prérequis** : Section 2.6 (Insertion de données)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Extraire des données avec SELECT (toutes colonnes ou spécifiques)
- Filtrer les résultats avec WHERE et les opérateurs de comparaison
- Utiliser les opérateurs logiques (AND, OR, NOT)
- Rechercher avec LIKE et les patterns
- Trier les résultats avec ORDER BY
- Limiter le nombre de résultats avec LIMIT
- Paginer les résultats avec OFFSET
- Éliminer les doublons avec DISTINCT
- Combiner plusieurs clauses dans une requête complexe

---

## Introduction

La requête **SELECT** est la commande la plus utilisée en SQL. Elle permet d'interroger la base de données pour extraire, filtrer, trier et limiter les données selon vos besoins.

### Anatomie d'une requête SELECT

```sql
SELECT colonne1, colonne2          -- Quelles colonnes ?
FROM nom_table                     -- Quelle table ?
WHERE condition                    -- Quels filtres ?
ORDER BY colonne                   -- Quel ordre ?
LIMIT nombre;                      -- Combien de lignes ?
```

---

## Données de démonstration

Pour illustrer cette section, créons une base de données de librairie :

```sql
-- Création de la base
CREATE DATABASE librairie CHARACTER SET utf8mb4;
USE librairie;

-- Table auteurs
CREATE TABLE auteurs (
    auteur_id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    pays VARCHAR(50),
    date_naissance DATE
);

-- Table livres
CREATE TABLE livres (
    livre_id INT PRIMARY KEY AUTO_INCREMENT,
    titre VARCHAR(200) NOT NULL,
    auteur_id INT NOT NULL,
    prix DECIMAL(10,2) NOT NULL,
    stock INT DEFAULT 0,
    annee_publication YEAR,
    genre VARCHAR(50),
    pages INT,
    FOREIGN KEY (auteur_id) REFERENCES auteurs(auteur_id)
);

-- Insertion des auteurs
INSERT INTO auteurs (nom, prenom, pays, date_naissance) VALUES
    ('Hugo', 'Victor', 'France', '1802-02-26'),
    ('Camus', 'Albert', 'France', '1913-11-07'),
    ('Tolkien', 'J.R.R.', 'Royaume-Uni', '1892-01-03'),
    ('Rowling', 'J.K.', 'Royaume-Uni', '1965-07-31'),
    ('Hemingway', 'Ernest', 'États-Unis', '1899-07-21'),
    ('García Márquez', 'Gabriel', 'Colombie', '1927-03-06'),
    ('Murakami', 'Haruki', 'Japon', '1949-01-12');

-- Insertion des livres
INSERT INTO livres (titre, auteur_id, prix, stock, annee_publication, genre, pages) VALUES
    ('Les Misérables', 1, 25.99, 15, 1862, 'Roman', 1488),
    ('Notre-Dame de Paris', 1, 19.99, 8, 1831, 'Roman', 512),
    ('L''Étranger', 2, 12.50, 20, 1942, 'Roman', 123),
    ('La Peste', 2, 14.99, 12, 1947, 'Roman', 279),
    ('Le Seigneur des Anneaux', 3, 34.99, 25, 1954, 'Fantasy', 1216),
    ('Le Hobbit', 3, 18.99, 30, 1937, 'Fantasy', 310),
    ('Harry Potter à l''école des sorciers', 4, 22.99, 40, 1997, 'Fantasy', 320),
    ('Harry Potter et la Chambre des secrets', 4, 23.99, 35, 1998, 'Fantasy', 368),
    ('Le Vieil Homme et la Mer', 5, 11.99, 18, 1952, 'Roman', 127),
    ('Cent ans de solitude', 6, 21.99, 10, 1967, 'Roman', 417),
    ('Chronique d''une mort annoncée', 6, 15.99, 14, 1981, 'Roman', 120),
    ('Kafka sur le rivage', 7, 24.99, 22, 2002, 'Roman', 505),
    ('1Q84', 7, 29.99, 8, 2009, 'Roman', 925);
```

---

## SELECT - Sélection de colonnes

### SELECT * - Toutes les colonnes

```sql
-- Sélectionner toutes les colonnes
SELECT * FROM auteurs;

-- Résultat :
-- +----------+----------------+----------+--------------+----------------+
-- | auteur_id| nom            | prenom   | pays         | date_naissance |
-- +----------+----------------+----------+--------------+----------------+
-- | 1        | Hugo           | Victor   | France       | 1802-02-26     |
-- | 2        | Camus          | Albert   | France       | 1913-11-07     |
-- | 3        | Tolkien        | J.R.R.   | Royaume-Uni  | 1892-01-03     |
-- | ...      | ...            | ...      | ...          | ...            |
-- +----------+----------------+----------+--------------+----------------+
```

⚠️ **Attention** : `SELECT *` est pratique en développement mais déconseillé en production :
- Performance : Charge toutes les colonnes, même celles non nécessaires
- Réseau : Transfère plus de données qu'utile
- Maintenance : Cache les dépendances du code

### SELECT colonnes spécifiques

```sql
-- Sélectionner seulement certaines colonnes
SELECT titre, prix FROM livres;

-- Résultat :
-- +------------------------------------------+-------+
-- | titre                                    | prix  |
-- +------------------------------------------+-------+
-- | Les Misérables                           | 25.99 |
-- | Notre-Dame de Paris                      | 19.99 |
-- | L'Étranger                               | 12.50 |
-- | ...                                      | ...   |
-- +------------------------------------------+-------+

-- Plusieurs colonnes dans un ordre spécifique
SELECT titre, genre, annee_publication, prix, stock
FROM livres;
```

### Alias de colonnes (AS)

```sql
-- Renommer les colonnes dans le résultat
SELECT
    titre AS 'Titre du livre',
    prix AS 'Prix (€)',
    stock AS 'En stock'
FROM livres;

-- Résultat :
-- +------------------------------------------+----------+----------+
-- | Titre du livre                           | Prix (€) | En stock |
-- +------------------------------------------+----------+----------+
-- | Les Misérables                           | 25.99    | 15       |
-- | ...                                      | ...      | ...      |
-- +------------------------------------------+----------+----------+

-- AS est optionnel (mais recommandé pour lisibilité)
SELECT titre 'Titre', prix 'Prix' FROM livres;

-- Alias sans espaces (pas besoin de guillemets)
SELECT titre AS titre_livre, prix AS prix_euros FROM livres;
```

### Expressions et calculs

```sql
-- Calculs dans SELECT
SELECT
    titre,
    prix AS prix_original,
    prix * 0.8 AS prix_promotion,
    prix - (prix * 0.8) AS economie
FROM livres;

-- Résultat :
-- +-------------------------+---------------+-----------------+----------+
-- | titre                   | prix_original | prix_promotion  | economie |
-- +-------------------------+---------------+-----------------+----------+
-- | Les Misérables          | 25.99         | 20.79           | 5.20     |
-- | L'Étranger              | 12.50         | 10.00           | 2.50     |
-- +-------------------------+---------------+-----------------+----------+

-- Concaténation de chaînes
SELECT
    CONCAT(prenom, ' ', nom) AS nom_complet,
    pays
FROM auteurs;

-- Résultat :
-- +---------------------+--------------+
-- | nom_complet         | pays         |
-- +---------------------+--------------+
-- | Victor Hugo         | France       |
-- | Albert Camus        | France       |
-- | J.R.R. Tolkien      | Royaume-Uni  |
-- +---------------------+--------------+

-- Conditions dans SELECT
SELECT
    titre,
    stock,
    IF(stock > 20, 'Bien fourni', 'Stock faible') AS statut_stock
FROM livres;
```

### DISTINCT - Éliminer les doublons

```sql
-- Lister tous les genres (avec doublons)
SELECT genre FROM livres;
-- Résultat : Roman, Roman, Roman, Roman, Fantasy, Fantasy, Fantasy...

-- Lister les genres uniques
SELECT DISTINCT genre FROM livres;
-- Résultat :
-- +----------+
-- | genre    |
-- +----------+
-- | Roman    |
-- | Fantasy  |
-- +----------+

-- DISTINCT sur plusieurs colonnes
SELECT DISTINCT genre, annee_publication FROM livres;

-- Compter les valeurs distinctes
SELECT COUNT(DISTINCT genre) AS nb_genres FROM livres;
-- Résultat : 2

SELECT COUNT(DISTINCT pays) AS nb_pays FROM auteurs;
-- Résultat : 5
```

---

## WHERE - Filtrage des données

### Opérateurs de comparaison

```sql
-- Égalité (=)
SELECT titre, prix FROM livres WHERE prix = 25.99;

-- Différent (!= ou <>)
SELECT titre, genre FROM livres WHERE genre != 'Fantasy';
SELECT titre, genre FROM livres WHERE genre <> 'Fantasy';

-- Plus grand que (>)
SELECT titre, prix FROM livres WHERE prix > 20;

-- Plus grand ou égal (>=)
SELECT titre, prix FROM livres WHERE prix >= 20;

-- Plus petit que (<)
SELECT titre, pages FROM livres WHERE pages < 300;

-- Plus petit ou égal (<=)
SELECT titre, pages FROM livres WHERE pages <= 300;
```

### Exemples pratiques de filtrage

```sql
-- Livres à moins de 15€
SELECT titre, prix
FROM livres
WHERE prix < 15;

-- Résultat :
-- +-------------------------+-------+
-- | titre                   | prix  |
-- +-------------------------+-------+
-- | L'Étranger              | 12.50 |
-- | La Peste                | 14.99 |
-- | Le Vieil Homme et la Mer| 11.99 |
-- +-------------------------+-------+

-- Livres avec beaucoup de pages
SELECT titre, pages
FROM livres
WHERE pages > 500;

-- Livres publiés après 2000
SELECT titre, annee_publication
FROM livres
WHERE annee_publication > 2000;

-- Auteurs français
SELECT nom, prenom, pays
FROM auteurs
WHERE pays = 'France';
```

---

## Opérateurs logiques (AND, OR, NOT)

### AND - Toutes les conditions doivent être vraies

```sql
-- Livres Fantasy ET à moins de 25€
SELECT titre, genre, prix
FROM livres
WHERE genre = 'Fantasy' AND prix < 25;

-- Résultat :
-- +------------------------------------------+---------+-------+
-- | titre                                    | genre   | prix  |
-- +------------------------------------------+---------+-------+
-- | Le Hobbit                                | Fantasy | 18.99 |
-- | Harry Potter à l'école des sorciers      | Fantasy | 22.99 |
-- | Harry Potter et la Chambre des secrets   | Fantasy | 23.99 |
-- +------------------------------------------+---------+-------+

-- Livres publiés entre 1940 et 1960 avec stock > 10
SELECT titre, annee_publication, stock
FROM livres
WHERE annee_publication >= 1940
  AND annee_publication <= 1960
  AND stock > 10;

-- Trois conditions avec AND
SELECT titre, prix, stock, genre
FROM livres
WHERE prix < 20
  AND stock > 15
  AND genre = 'Roman';
```

### OR - Au moins une condition doit être vraie

```sql
-- Livres Fantasy OU à plus de 25€
SELECT titre, genre, prix
FROM livres
WHERE genre = 'Fantasy' OR prix > 25;

-- Auteurs français OU britanniques
SELECT nom, prenom, pays
FROM auteurs
WHERE pays = 'France' OR pays = 'Royaume-Uni';

-- Résultat :
-- +----------+----------+--------------+
-- | nom      | prenom   | pays         |
-- +----------+----------+--------------+
-- | Hugo     | Victor   | France       |
-- | Camus    | Albert   | France       |
-- | Tolkien  | J.R.R.   | Royaume-Uni  |
-- | Rowling  | J.K.     | Royaume-Uni  |
-- +----------+----------+--------------+
```

### Combinaison AND et OR (avec parenthèses)

```sql
-- Sans parenthèses (peut être ambigu)
SELECT titre, genre, prix, stock
FROM livres
WHERE genre = 'Fantasy' AND prix < 25 OR stock > 30;
-- Évalué comme : (genre = 'Fantasy' AND prix < 25) OR stock > 30

-- Avec parenthèses (recommandé)
SELECT titre, genre, prix, stock
FROM livres
WHERE genre = 'Fantasy' AND (prix < 25 OR stock > 30);
-- Évalué comme : genre = 'Fantasy' AND (prix < 25 OR stock > 30)

-- Autre exemple avec parenthèses
SELECT titre, genre, prix
FROM livres
WHERE (genre = 'Roman' OR genre = 'Fantasy') AND prix < 20;
-- Livres Roman ou Fantasy, avec prix < 20
```

💡 **Bonne pratique** : Utilisez toujours des parenthèses pour clarifier la priorité des opérateurs.

### NOT - Négation

```sql
-- Livres qui ne sont PAS du genre Fantasy
SELECT titre, genre
FROM livres
WHERE NOT genre = 'Fantasy';
-- Équivalent à : WHERE genre != 'Fantasy'

-- Auteurs qui ne sont PAS français
SELECT nom, prenom, pays
FROM auteurs
WHERE NOT pays = 'France';

-- Livres avec prix pas entre 15 et 25
SELECT titre, prix
FROM livres
WHERE NOT (prix >= 15 AND prix <= 25);
-- Équivalent à : WHERE prix < 15 OR prix > 25
```

---

## BETWEEN - Intervalle de valeurs

### BETWEEN pour les nombres

```sql
-- Prix entre 15 et 25 euros (inclusif)
SELECT titre, prix
FROM livres
WHERE prix BETWEEN 15 AND 25;
-- Équivalent à : WHERE prix >= 15 AND prix <= 25

-- Résultat :
-- +------------------------------------------+-------+
-- | titre                                    | prix  |
-- +------------------------------------------+-------+
-- | Notre-Dame de Paris                      | 19.99 |
-- | Le Hobbit                                | 18.99 |
-- | Harry Potter à l'école des sorciers      | 22.99 |
-- | Harry Potter et la Chambre des secrets   | 23.99 |
-- | Cent ans de solitude                     | 21.99 |
-- | Kafka sur le rivage                      | 24.99 |
-- +------------------------------------------+-------+

-- Pages entre 100 et 300
SELECT titre, pages
FROM livres
WHERE pages BETWEEN 100 AND 300;
```

### BETWEEN pour les dates

```sql
-- Auteurs nés entre 1890 et 1920
SELECT nom, prenom, date_naissance
FROM auteurs
WHERE date_naissance BETWEEN '1890-01-01' AND '1920-12-31';

-- Résultat :
-- +----------+---------+----------------+
-- | nom      | prenom  | date_naissance |
-- +----------+---------+----------------+
-- | Tolkien  | J.R.R.  | 1892-01-03     |
-- | Camus    | Albert  | 1913-11-07     |
-- | Hemingway| Ernest  | 1899-07-21     |
-- +----------+---------+----------------+

-- Livres publiés dans les années 50-60
SELECT titre, annee_publication
FROM livres
WHERE annee_publication BETWEEN 1950 AND 1969;
```

### NOT BETWEEN

```sql
-- Livres dont le prix n'est PAS entre 15 et 25
SELECT titre, prix
FROM livres
WHERE prix NOT BETWEEN 15 AND 25;

-- Équivalent à :
SELECT titre, prix
FROM livres
WHERE prix < 15 OR prix > 25;
```

---

## IN - Liste de valeurs

### IN pour remplacer plusieurs OR

```sql
-- Auteurs de plusieurs pays spécifiques
SELECT nom, prenom, pays
FROM auteurs
WHERE pays IN ('France', 'Royaume-Uni', 'Japon');

-- Équivalent à (mais plus lisible) :
SELECT nom, prenom, pays
FROM auteurs
WHERE pays = 'France'
   OR pays = 'Royaume-Uni'
   OR pays = 'Japon';

-- Résultat :
-- +----------+---------+--------------+
-- | nom      | prenom  | pays         |
-- +----------+---------+--------------+
-- | Hugo     | Victor  | France       |
-- | Camus    | Albert  | France       |
-- | Tolkien  | J.R.R.  | Royaume-Uni  |
-- | Rowling  | J.K.    | Royaume-Uni  |
-- | Murakami | Haruki  | Japon        |
-- +----------+---------+--------------+

-- Livres publiés certaines années
SELECT titre, annee_publication
FROM livres
WHERE annee_publication IN (1942, 1954, 1997, 2002);

-- Livres de genres spécifiques
SELECT titre, genre, prix
FROM livres
WHERE genre IN ('Fantasy', 'Science-Fiction', 'Thriller');
```

### NOT IN

```sql
-- Auteurs qui ne sont ni français ni britanniques
SELECT nom, prenom, pays
FROM auteurs
WHERE pays NOT IN ('France', 'Royaume-Uni');

-- Résultat :
-- +----------------+---------+--------------+
-- | nom            | prenom  | pays         |
-- +----------------+---------+--------------+
-- | Hemingway      | Ernest  | États-Unis   |
-- | García Márquez | Gabriel | Colombie     |
-- | Murakami       | Haruki  | Japon        |
-- +----------------+---------+--------------+
```

### IN avec sous-requête (aperçu)

```sql
-- Livres écrits par des auteurs français
SELECT titre, auteur_id
FROM livres
WHERE auteur_id IN (
    SELECT auteur_id
    FROM auteurs
    WHERE pays = 'France'
);
-- Nous verrons les sous-requêtes en détail dans une section ultérieure
```

---

## LIKE - Recherche de patterns

### Opérateurs wildcard

| Symbole | Signification | Exemple |
|---------|---------------|---------|
| **%** | N'importe quelle séquence de caractères (0 ou plus) | `'H%'` = commence par H |
| **_** | Exactement un caractère | `'_arry'` = 5 lettres finissant par arry |

### LIKE avec %

```sql
-- Titres commençant par "Le"
SELECT titre FROM livres WHERE titre LIKE 'Le%';
-- Résultat :
-- Les Misérables
-- Le Seigneur des Anneaux
-- Le Hobbit
-- Le Vieil Homme et la Mer

-- Titres contenant "Harry"
SELECT titre FROM livres WHERE titre LIKE '%Harry%';
-- Résultat :
-- Harry Potter à l'école des sorciers
-- Harry Potter et la Chambre des secrets

-- Titres finissant par "secrets"
SELECT titre FROM livres WHERE titre LIKE '%secrets';
-- Résultat :
-- Harry Potter et la Chambre des secrets

-- Auteurs dont le nom contient "arqu"
SELECT nom, prenom FROM auteurs WHERE nom LIKE '%arqu%';
-- Résultat : García Márquez
```

### LIKE avec _

```sql
-- Prénoms de 5 lettres exactement
SELECT prenom FROM auteurs WHERE prenom LIKE '_____';
-- Résultat : Victor, Ernest, Haruki

-- Prénoms commençant par J suivi de 3 caractères
SELECT nom, prenom FROM auteurs WHERE prenom LIKE 'J___';
-- Résultat : J.R.R. (4 caractères)

-- Combiner % et _
SELECT titre FROM livres WHERE titre LIKE '1Q__';
-- Résultat : 1Q84 (1Q suivi de 2 caractères)
```

### LIKE sensible à la casse

```sql
-- Par défaut, LIKE est case-insensitive (selon collation)
SELECT titre FROM livres WHERE titre LIKE '%harry%';
-- Trouve "Harry Potter..."

-- Forcer la sensibilité à la casse avec BINARY
SELECT titre FROM livres WHERE titre LIKE BINARY '%Harry%';
-- Trouve "Harry Potter..." mais pas "harry potter..."

-- Ou utiliser COLLATE
SELECT titre FROM livres
WHERE titre LIKE '%Harry%' COLLATE utf8mb4_bin;
```

### NOT LIKE

```sql
-- Livres dont le titre ne contient pas "Harry"
SELECT titre FROM livres WHERE titre NOT LIKE '%Harry%';

-- Auteurs dont le nom ne commence pas par "H"
SELECT nom, prenom FROM auteurs WHERE nom NOT LIKE 'H%';
```

### Échapper les caractères spéciaux

```sql
-- Rechercher un caractère % ou _ littéral
CREATE TABLE fichiers (nom VARCHAR(100));
INSERT INTO fichiers VALUES ('rapport_2025.pdf'), ('image_10%.jpg');

-- Trouver les fichiers avec un underscore
SELECT nom FROM fichiers WHERE nom LIKE '%\_%';
-- Résultat : rapport_2025.pdf, image_10%.jpg

-- Trouver les fichiers avec %
SELECT nom FROM fichiers WHERE nom LIKE '%\%%';
-- Résultat : image_10%.jpg

-- Ou spécifier un caractère d'échappement personnalisé
SELECT nom FROM fichiers WHERE nom LIKE '%!_%' ESCAPE '!';
```

---

## IS NULL / IS NOT NULL - Valeurs nulles

### Rappel : NULL vs valeurs vides

```sql
-- NULL n'est pas la même chose que '' ou 0
SELECT
    NULL = NULL AS test1,           -- NULL (pas TRUE)
    NULL IS NULL AS test2,          -- TRUE
    '' = NULL AS test3,             -- NULL
    '' IS NULL AS test4;            -- FALSE
```

### IS NULL

```sql
-- Créer une table avec valeurs NULL
CREATE TABLE test_null (
    id INT PRIMARY KEY AUTO_INCREMENT,
    valeur VARCHAR(50)
);

INSERT INTO test_null (valeur) VALUES ('ABC'), (NULL), (''), (NULL), ('DEF');

-- ❌ INCORRECT : Utiliser = NULL
SELECT * FROM test_null WHERE valeur = NULL;
-- Résultat : 0 lignes (ne fonctionne jamais)

-- ✅ CORRECT : Utiliser IS NULL
SELECT * FROM test_null WHERE valeur IS NULL;
-- Résultat :
-- +----+--------+
-- | id | valeur |
-- +----+--------+
-- | 2  | NULL   |
-- | 4  | NULL   |
-- +----+--------+

-- Exemple réel : Livres sans genre spécifié
SELECT titre, genre FROM livres WHERE genre IS NULL;
```

### IS NOT NULL

```sql
-- Lignes avec valeur non-NULL
SELECT * FROM test_null WHERE valeur IS NOT NULL;
-- Résultat :
-- +----+--------+
-- | id | valeur |
-- +----+--------+
-- | 1  | ABC    |
-- | 3  |        |  -- Chaîne vide, pas NULL
-- | 5  | DEF    |
-- +----+--------+

-- Exemple réel : Auteurs avec date de naissance connue
SELECT nom, prenom, date_naissance
FROM auteurs
WHERE date_naissance IS NOT NULL;
```

### COALESCE - Remplacer NULL par une valeur

```sql
-- Remplacer NULL par une valeur par défaut
SELECT
    titre,
    COALESCE(genre, 'Non classé') AS genre
FROM livres;

-- Si genre est NULL, affiche 'Non classé'
-- Sinon, affiche la valeur du genre

-- Avec plusieurs alternatives
SELECT
    COALESCE(NULL, NULL, 'Valeur par défaut', 'Autre') AS resultat;
-- Résultat : 'Valeur par défaut' (première valeur non-NULL)
```

---

## ORDER BY - Tri des résultats

### Tri ascendant (ASC)

```sql
-- Tri par prix croissant (du moins cher au plus cher)
SELECT titre, prix
FROM livres
ORDER BY prix ASC;

-- ASC est le défaut, peut être omis
SELECT titre, prix
FROM livres
ORDER BY prix;

-- Résultat :
-- +------------------------------------------+-------+
-- | titre                                    | prix  |
-- +------------------------------------------+-------+
-- | Le Vieil Homme et la Mer                 | 11.99 |
-- | L'Étranger                               | 12.50 |
-- | La Peste                                 | 14.99 |
-- | Chronique d'une mort annoncée            | 15.99 |
-- | Le Hobbit                                | 18.99 |
-- | ...                                      | ...   |
-- +------------------------------------------+-------+

-- Tri alphabétique
SELECT titre FROM livres ORDER BY titre;
-- Résultat : 1Q84, Cent ans de solitude, Chronique..., Harry Potter...
```

### Tri descendant (DESC)

```sql
-- Tri par prix décroissant (du plus cher au moins cher)
SELECT titre, prix
FROM livres
ORDER BY prix DESC;

-- Résultat :
-- +------------------------------------------+-------+
-- | titre                                    | prix  |
-- +------------------------------------------+-------+
-- | Le Seigneur des Anneaux                  | 34.99 |
-- | 1Q84                                     | 29.99 |
-- | Les Misérables                           | 25.99 |
-- | Kafka sur le rivage                      | 24.99 |
-- | Harry Potter et la Chambre des secrets   | 23.99 |
-- | ...                                      | ...   |
-- +------------------------------------------+-------+

-- Livres les plus récents d'abord
SELECT titre, annee_publication
FROM livres
ORDER BY annee_publication DESC;
```

### Tri sur plusieurs colonnes

```sql
-- Tri par genre, puis par prix
SELECT titre, genre, prix
FROM livres
ORDER BY genre, prix;
-- Tri principal : genre (alphabétique)
-- Tri secondaire : prix (croissant) pour chaque genre

-- Résultat :
-- +------------------------------------------+---------+-------+
-- | titre                                    | genre   | prix  |
-- +------------------------------------------+---------+-------+
-- | Le Hobbit                                | Fantasy | 18.99 |
-- | Harry Potter à l'école des sorciers      | Fantasy | 22.99 |
-- | Harry Potter et la Chambre des secrets   | Fantasy | 23.99 |
-- | Le Seigneur des Anneaux                  | Fantasy | 34.99 |
-- | Le Vieil Homme et la Mer                 | Roman   | 11.99 |
-- | L'Étranger                               | Roman   | 12.50 |
-- | ...                                      | ...     | ...   |
-- +------------------------------------------+---------+-------+

-- Combiner ASC et DESC
SELECT titre, genre, prix
FROM livres
ORDER BY genre ASC, prix DESC;
-- Genre alphabétique, mais prix décroissant dans chaque genre

-- Tri sur 3 colonnes
SELECT titre, genre, annee_publication, prix
FROM livres
ORDER BY genre, annee_publication DESC, prix;
```

### ORDER BY avec expressions

```sql
-- Tri par calcul
SELECT
    titre,
    prix,
    stock,
    prix * stock AS valeur_stock
FROM livres
ORDER BY valeur_stock DESC;

-- Tri par longueur de titre
SELECT titre, LENGTH(titre) AS longueur
FROM livres
ORDER BY LENGTH(titre) DESC;

-- Tri par fonction
SELECT nom, prenom
FROM auteurs
ORDER BY CONCAT(prenom, ' ', nom);
```

### ORDER BY avec position de colonne

```sql
-- Référencer les colonnes par leur position (1, 2, 3...)
SELECT titre, prix, stock
FROM livres
ORDER BY 2;  -- Trier par la 2e colonne (prix)

-- Équivalent à :
SELECT titre, prix, stock
FROM livres
ORDER BY prix;

-- ⚠️ Déconseillé : Peu lisible et fragile (si l'ordre des colonnes change)
```

---

## LIMIT - Limiter le nombre de résultats

### LIMIT simple

```sql
-- Les 5 premiers livres
SELECT titre, prix FROM livres LIMIT 5;

-- Résultat : 5 lignes seulement

-- Les 3 livres les moins chers
SELECT titre, prix
FROM livres
ORDER BY prix
LIMIT 3;

-- Résultat :
-- +---------------------------+-------+
-- | titre                     | prix  |
-- +---------------------------+-------+
-- | Le Vieil Homme et la Mer  | 11.99 |
-- | L'Étranger                | 12.50 |
-- | La Peste                  | 14.99 |
-- +---------------------------+-------+

-- Les 5 livres les plus chers
SELECT titre, prix
FROM livres
ORDER BY prix DESC
LIMIT 5;
```

### LIMIT avec OFFSET - Pagination

```sql
-- Syntaxe : LIMIT nombre OFFSET decalage

-- Page 1 : Les 5 premiers livres (0-4)
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 5 OFFSET 0;

-- Page 2 : Les 5 suivants (5-9)
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 5 OFFSET 5;

-- Page 3 : Les 5 suivants (10-14)
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 5 OFFSET 10;

-- Formule générale pour la pagination :
-- Page N avec X éléments par page
-- LIMIT X OFFSET (N-1) * X
```

### Syntaxe alternative LIMIT

```sql
-- Syntaxe MySQL : LIMIT offset, count
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 5, 5;  -- Équivalent à LIMIT 5 OFFSET 5

-- Page 1 : LIMIT 0, 10
-- Page 2 : LIMIT 10, 10
-- Page 3 : LIMIT 20, 10

-- ⚠️ Attention à l'ordre : LIMIT offset, count (inversé)
```

### LIMIT sans ORDER BY

```sql
-- LIMIT sans ORDER BY : Ordre imprévisible
SELECT titre FROM livres LIMIT 3;
-- Résultat : 3 lignes, mais lesquelles ? Ordre non garanti

-- ✅ TOUJOURS utiliser ORDER BY avec LIMIT
SELECT titre FROM livres ORDER BY livre_id LIMIT 3;
```

### Exemples pratiques de pagination

```sql
-- Système de pagination : 10 livres par page

-- Compter le total de livres
SELECT COUNT(*) AS total FROM livres;
-- Résultat : 13 livres → 2 pages complètes + 1 partielle

-- Page 1
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 10 OFFSET 0;

-- Page 2
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 10 OFFSET 10;

-- Page 3 (partielle)
SELECT titre, prix
FROM livres
ORDER BY titre
LIMIT 10 OFFSET 20;
-- Résultat : 3 lignes seulement
```

---

## Requêtes combinées

### Exemple 1 : Recherche avec plusieurs critères

```sql
-- Livres Fantasy, prix < 25€, plus de 300 pages, triés par prix
SELECT
    titre,
    genre,
    prix,
    pages
FROM livres
WHERE genre = 'Fantasy'
  AND prix < 25
  AND pages > 300
ORDER BY prix
LIMIT 5;
```

### Exemple 2 : Top 5 des auteurs

```sql
-- Les 5 auteurs les plus jeunes (nés le plus récemment)
SELECT
    CONCAT(prenom, ' ', nom) AS nom_complet,
    pays,
    date_naissance,
    YEAR(CURDATE()) - YEAR(date_naissance) AS age
FROM auteurs
WHERE date_naissance IS NOT NULL
ORDER BY date_naissance DESC
LIMIT 5;
```

### Exemple 3 : Recherche et pagination

```sql
-- Recherche de livres contenant "Harry", page 1
SELECT
    titre,
    genre,
    prix,
    stock
FROM livres
WHERE titre LIKE '%Harry%'
ORDER BY titre
LIMIT 5 OFFSET 0;

-- Si plusieurs pages de résultats, page 2
SELECT
    titre,
    genre,
    prix,
    stock
FROM livres
WHERE titre LIKE '%Harry%'
ORDER BY titre
LIMIT 5 OFFSET 5;
```

### Exemple 4 : Catalogue de produits

```sql
-- Livres disponibles (stock > 0), 10-20€, triés par popularité (stock décroissant)
SELECT
    titre,
    prix,
    stock,
    prix * stock AS valeur_inventaire
FROM livres
WHERE stock > 0
  AND prix BETWEEN 10 AND 20
ORDER BY stock DESC, titre
LIMIT 20;
```

### Exemple 5 : Recherche multicritères

```sql
-- Livres français ou britanniques, publiés après 1950, moins de 30€
SELECT
    l.titre,
    CONCAT(a.prenom, ' ', a.nom) AS auteur,
    a.pays,
    l.annee_publication,
    l.prix
FROM livres l
JOIN auteurs a ON l.auteur_id = a.auteur_id
WHERE a.pays IN ('France', 'Royaume-Uni')
  AND l.annee_publication > 1950
  AND l.prix < 30
ORDER BY l.annee_publication DESC, l.prix;
```

---

## Fonctions d'agrégation (aperçu)

### COUNT - Compter

```sql
-- Nombre total de livres
SELECT COUNT(*) AS total_livres FROM livres;
-- Résultat : 13

-- Nombre de livres par genre
SELECT
    genre,
    COUNT(*) AS nombre
FROM livres
GROUP BY genre;

-- Résultat :
-- +---------+--------+
-- | genre   | nombre |
-- +---------+--------+
-- | Fantasy | 4      |
-- | Roman   | 9      |
-- +---------+--------+

-- Compter les valeurs non-NULL
SELECT COUNT(genre) FROM livres;
```

### SUM - Somme

```sql
-- Valeur totale de l'inventaire
SELECT
    SUM(prix * stock) AS valeur_totale
FROM livres;

-- Nombre total de livres en stock
SELECT
    SUM(stock) AS total_stock
FROM livres;
```

### AVG - Moyenne

```sql
-- Prix moyen des livres
SELECT
    AVG(prix) AS prix_moyen
FROM livres;
-- Résultat : 21.68

-- Prix moyen par genre
SELECT
    genre,
    AVG(prix) AS prix_moyen
FROM livres
GROUP BY genre;
```

### MIN et MAX

```sql
-- Livre le moins cher et le plus cher
SELECT
    MIN(prix) AS moins_cher,
    MAX(prix) AS plus_cher
FROM livres;

-- Résultat :
-- +------------+-----------+
-- | moins_cher | plus_cher |
-- +------------+-----------+
-- | 11.99      | 34.99     |
-- +------------+-----------+

-- Année de publication la plus ancienne et la plus récente
SELECT
    MIN(annee_publication) AS plus_ancien,
    MAX(annee_publication) AS plus_recent
FROM livres;
```

💡 **Note** : Les fonctions d'agrégation et GROUP BY seront détaillées dans une section ultérieure.

---

## Bonnes pratiques

### 1. Toujours spécifier les colonnes dans SELECT

```sql
-- ❌ MAUVAIS : SELECT *
SELECT * FROM livres WHERE genre = 'Fantasy';

-- ✅ BON : Colonnes explicites
SELECT titre, prix, stock FROM livres WHERE genre = 'Fantasy';
```

**Avantages** :
- Performance (moins de données transférées)
- Lisibilité (on sait ce qu'on récupère)
- Maintenance (indépendant de la structure de la table)

### 2. Utiliser ORDER BY avec LIMIT

```sql
-- ❌ MAUVAIS : Ordre imprévisible
SELECT titre FROM livres LIMIT 5;

-- ✅ BON : Ordre déterministe
SELECT titre FROM livres ORDER BY titre LIMIT 5;
```

### 3. Utiliser des alias pour la lisibilité

```sql
-- ❌ MOINS LISIBLE
SELECT titre, prix * 0.8 FROM livres;

-- ✅ PLUS LISIBLE
SELECT
    titre,
    prix AS prix_original,
    prix * 0.8 AS prix_promotion
FROM livres;
```

### 4. Utiliser des parenthèses avec AND/OR

```sql
-- ❌ AMBIGU
SELECT * FROM livres
WHERE genre = 'Roman' AND prix < 15 OR stock > 20;

-- ✅ CLAIR
SELECT * FROM livres
WHERE genre = 'Roman' AND (prix < 15 OR stock > 20);
```

### 5. Utiliser IN plutôt que multiples OR

```sql
-- ❌ VERBEUX
SELECT * FROM auteurs
WHERE pays = 'France'
   OR pays = 'Japon'
   OR pays = 'Colombie';

-- ✅ CONCIS
SELECT * FROM auteurs
WHERE pays IN ('France', 'Japon', 'Colombie');
```

### 6. Utiliser BETWEEN pour les intervalles

```sql
-- ❌ VERBEUX
SELECT * FROM livres
WHERE prix >= 15 AND prix <= 25;

-- ✅ CONCIS
SELECT * FROM livres
WHERE prix BETWEEN 15 AND 25;
```

### 7. Toujours utiliser IS NULL, jamais = NULL

```sql
-- ❌ NE FONCTIONNE JAMAIS
SELECT * FROM livres WHERE genre = NULL;

-- ✅ CORRECT
SELECT * FROM livres WHERE genre IS NULL;
```

---

## Pièges courants à éviter

### ❌ Piège 1 : WHERE colonne = NULL

```sql
-- ❌ NE RETOURNE JAMAIS DE LIGNES
SELECT * FROM livres WHERE genre = NULL;

-- ✅ CORRECT
SELECT * FROM livres WHERE genre IS NULL;
```

### ❌ Piège 2 : Oublier les guillemets pour les chaînes

```sql
-- ❌ ERREUR : Colonne inconnue 'Fantasy'
SELECT * FROM livres WHERE genre = Fantasy;

-- ✅ CORRECT
SELECT * FROM livres WHERE genre = 'Fantasy';
```

### ❌ Piège 3 : Confondre = et LIKE

```sql
-- = : Égalité exacte
SELECT * FROM livres WHERE titre = 'Harry';
-- Résultat : 0 lignes (pas de titre exactement "Harry")

-- LIKE : Pattern matching
SELECT * FROM livres WHERE titre LIKE '%Harry%';
-- Résultat : 2 livres contenant "Harry"
```

### ❌ Piège 4 : Ordre des opérations avec LIMIT OFFSET

```sql
-- ❌ ERREUR : Syntaxe incorrecte
SELECT * FROM livres OFFSET 5 LIMIT 10;

-- ✅ CORRECT : LIMIT avant OFFSET
SELECT * FROM livres LIMIT 10 OFFSET 5;

-- OU syntaxe MySQL
SELECT * FROM livres LIMIT 5, 10;  -- offset, count
```

### ❌ Piège 5 : Case sensitivity avec LIKE

```sql
-- Selon la collation, LIKE peut être case-insensitive
SELECT * FROM livres WHERE titre LIKE '%harry%';
-- Peut trouver "Harry Potter" selon la collation

-- Pour forcer case-sensitive
SELECT * FROM livres WHERE titre LIKE BINARY '%Harry%';
```

### ❌ Piège 6 : DISTINCT sur mauvaises colonnes

```sql
-- ❌ DISTINCT sur toutes les colonnes (peu utile)
SELECT DISTINCT * FROM livres;

-- ✅ DISTINCT sur la colonne pertinente
SELECT DISTINCT genre FROM livres;
```

### ❌ Piège 7 : Pagination sans ORDER BY stable

```sql
-- ❌ INSTABLE : Les résultats peuvent changer entre pages
SELECT * FROM livres LIMIT 10 OFFSET 0;
SELECT * FROM livres LIMIT 10 OFFSET 10;
-- Si des insertions/suppressions ont lieu, les pages se chevauchent

-- ✅ STABLE : Ordre déterministe
SELECT * FROM livres ORDER BY livre_id LIMIT 10 OFFSET 0;
SELECT * FROM livres ORDER BY livre_id LIMIT 10 OFFSET 10;
```

---

## Optimisation et performance

### Index et performances WHERE

```sql
-- Créer un index pour accélérer les recherches
CREATE INDEX idx_genre ON livres(genre);
CREATE INDEX idx_prix ON livres(prix);
CREATE INDEX idx_auteur ON livres(auteur_id);

-- Maintenant les requêtes avec WHERE sur ces colonnes sont plus rapides
SELECT * FROM livres WHERE genre = 'Fantasy';
SELECT * FROM livres WHERE prix < 20;
```

### EXPLAIN - Analyser les requêtes

```sql
-- Voir le plan d'exécution
EXPLAIN SELECT * FROM livres WHERE genre = 'Fantasy';

-- Résultat montre :
-- - Si un index est utilisé
-- - Combien de lignes sont scannées
-- - Le type de recherche (ALL, range, ref, etc.)
```

### LIMIT pour éviter de charger trop de données

```sql
-- ❌ INEFFICACE : Charge potentiellement des millions de lignes
SELECT * FROM livres WHERE prix > 10;

-- ✅ EFFICACE : Limite si on a juste besoin d'un échantillon
SELECT * FROM livres WHERE prix > 10 LIMIT 100;
```

---

## Exemples pratiques complets

### Exemple 1 : Catalogue de librairie en ligne

```sql
-- Page d'accueil : Top 10 livres en stock
SELECT
    titre,
    CONCAT(a.prenom, ' ', a.nom) AS auteur,
    l.prix,
    l.stock
FROM livres l
JOIN auteurs a ON l.auteur_id = a.auteur_id
WHERE l.stock > 0
ORDER BY l.stock DESC
LIMIT 10;
```

### Exemple 2 : Recherche avancée

```sql
-- Recherche : "roman français moins de 20€"
SELECT
    l.titre,
    CONCAT(a.prenom, ' ', a.nom) AS auteur,
    l.prix,
    l.annee_publication
FROM livres l
JOIN auteurs a ON l.auteur_id = a.auteur_id
WHERE l.genre = 'Roman'
  AND a.pays = 'France'
  AND l.prix < 20
ORDER BY l.annee_publication DESC, l.prix;
```

### Exemple 3 : Recommandations "Vous aimerez aussi"

```sql
-- Livres similaires (même genre, prix proche, auteur différent)
SET @livre_id = 5;  -- Le Seigneur des Anneaux
SET @genre = (SELECT genre FROM livres WHERE livre_id = @livre_id);
SET @prix = (SELECT prix FROM livres WHERE livre_id = @livre_id);
SET @auteur = (SELECT auteur_id FROM livres WHERE livre_id = @livre_id);

SELECT
    titre,
    prix,
    ABS(prix - @prix) AS difference_prix
FROM livres
WHERE livre_id != @livre_id
  AND genre = @genre
  AND auteur_id != @auteur
  AND prix BETWEEN @prix * 0.7 AND @prix * 1.3
ORDER BY ABS(prix - @prix)
LIMIT 5;
```

### Exemple 4 : Statistiques du catalogue

```sql
-- Vue d'ensemble du catalogue
SELECT
    'Total livres' AS statistique,
    COUNT(*) AS valeur
FROM livres
UNION ALL
SELECT
    'Prix moyen',
    ROUND(AVG(prix), 2)
FROM livres
UNION ALL
SELECT
    'Stock total',
    SUM(stock)
FROM livres
UNION ALL
SELECT
    'Valeur inventaire',
    ROUND(SUM(prix * stock), 2)
FROM livres;

-- Résultat :
-- +-------------------+--------+
-- | statistique       | valeur |
-- +-------------------+--------+
-- | Total livres      | 13     |
-- | Prix moyen        | 21.68  |
-- | Stock total       | 274    |
-- | Valeur inventaire | 5940.26|
-- +-------------------+--------+
```

---

## ✅ Points clés à retenir

- **SELECT** : Spécifier les colonnes (éviter SELECT *)
- **FROM** : Table source des données
- **WHERE** : Filtrage avec opérateurs =, !=, <, >, <=, >=
- **AND/OR/NOT** : Combiner conditions (utiliser parenthèses)
- **BETWEEN** : Intervalles de valeurs (inclusif)
- **IN** : Liste de valeurs (remplace plusieurs OR)
- **LIKE** : Recherche de patterns (% et _)
- **IS NULL / IS NOT NULL** : Tester valeurs NULL (jamais = NULL)
- **ORDER BY** : Tri ASC (croissant) ou DESC (décroissant)
- **LIMIT** : Limiter nombre de résultats
- **OFFSET** : Pagination (sauter N lignes)
- **DISTINCT** : Éliminer doublons
- **Alias (AS)** : Renommer colonnes dans résultats
- **Toujours ORDER BY avec LIMIT** pour ordre déterministe
- **Index** : Accélère les recherches WHERE
- **Parenthèses** : Clarifier priorité AND/OR

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 SELECT Statement](https://mariadb.com/kb/en/select/)
- [📖 WHERE Clause](https://mariadb.com/kb/en/where/)
- [📖 ORDER BY Clause](https://mariadb.com/kb/en/order-by/)
- [📖 LIMIT and OFFSET](https://mariadb.com/kb/en/limit/)
- [📖 LIKE Operator](https://mariadb.com/kb/en/like/)
- [📖 Pattern Matching](https://mariadb.com/kb/en/string-comparison-functions/)

### Lectures complémentaires
- [SQL SELECT Tutorial](https://www.sqltutorial.org/sql-select/)
- [Query Optimization](https://mariadb.com/kb/en/query-optimization/)

---

## ➡️ Section suivante

**2.8 Mise à jour et suppression (UPDATE, DELETE, TRUNCATE)**

Maintenant que vous savez interroger les données, apprenez à les modifier et les supprimer : UPDATE pour mettre à jour, DELETE pour supprimer des lignes spécifiques, et TRUNCATE pour vider une table rapidement.

---


⏭️ [Mise à jour et suppression de données (UPDATE, DELETE, TRUNCATE)](/02-bases-du-sql/08-mise-a-jour-suppression.md)

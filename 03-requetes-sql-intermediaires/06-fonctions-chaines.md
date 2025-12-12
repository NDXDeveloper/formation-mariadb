🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6 Fonctions de chaînes de caractères

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 3.1 à 3.5, manipulation de base SELECT

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser les fonctions de concaténation et formatage
- Transformer la casse des textes (majuscules, minuscules)
- Extraire des sous-chaînes avec précision
- Rechercher et remplacer du texte
- Nettoyer et normaliser des données textuelles
- Appliquer des fonctions de remplissage et alignement
- Résoudre des problèmes métier de traitement de texte
- Optimiser les requêtes utilisant des fonctions de chaînes

---

## Introduction

Les **fonctions de chaînes de caractères** permettent de **manipuler, transformer et analyser du texte** directement dans vos requêtes SQL. Elles sont essentielles pour :

### Cas d'usage courants

| Cas d'usage | Exemple |
|-------------|---------|
| **Formatage** | Noms complets, adresses formatées, libellés |
| **Nettoyage** | Suppression espaces, normalisation |
| **Validation** | Formats email, codes postaux, téléphones |
| **Recherche** | Filtrage, correspondances partielles |
| **Transformation** | Casse, extraction, remplacement |
| **Exports** | CSV, rapports, étiquettes |

### Catégories de fonctions

```
📝 CONCATÉNATION       ✂️ EXTRACTION          🔍 RECHERCHE
   CONCAT()               SUBSTRING()            LOCATE()
   CONCAT_WS()            LEFT()                 POSITION()
   GROUP_CONCAT()         RIGHT()                INSTR()

🔤 CASSE               🧹 NETTOYAGE           📏 MESURE
   UPPER()                TRIM()                 LENGTH()
   LOWER()                LTRIM()                CHAR_LENGTH()
   INITCAP()              RTRIM()                BIT_LENGTH()

🔄 TRANSFORMATION      ⚖️ COMPARAISON         📊 FORMATAGE
   REPLACE()              STRCMP()               LPAD()
   REVERSE()              LIKE                   RPAD()
   REPEAT()               REGEXP                 FORMAT()
```

---

## Concaténation : Assembler des chaînes

### CONCAT() : Concaténation simple

**CONCAT()** assemble plusieurs chaînes en une seule.

```sql
CONCAT(chaine1, chaine2, ..., chaineN)
```

#### Exemple 1 : Nom complet

**Question métier** : *Afficher les noms complets des clients*

```sql
SELECT
    id_client,
    CONCAT(prenom, ' ', nom) AS nom_complet,
    email
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+------------+----------------+-------------------+
| id_client  | nom_complet    | email             |
+------------+----------------+-------------------+
|       1847 | Alice Martin   | alice@email.com   |
|       2934 | Bob Dupont     | bob@email.com     |
|       3621 | Charlie Durand | charlie@email.com |
|       4582 | Diana Lopez    | diana@email.com   |
|       5193 | Éric Moreau    | eric@email.com    |
+------------+----------------+-------------------+
```

**Points clés** :
- Espace `' '` ajouté manuellement entre prénom et nom
- Si une valeur est NULL, CONCAT() retourne NULL

#### Exemple 2 : Adresse complète

**Question métier** : *Formatter une adresse postale sur une ligne*

```sql
SELECT
    id_client,
    nom,
    CONCAT(
        numero_rue, ' ',
        rue, ', ',
        code_postal, ' ',
        ville, ', ',
        pays
    ) AS adresse_complete
FROM clients
WHERE pays = 'France'
LIMIT 3;
```

**Résultat attendu** :
```
+------------+----------------+--------------------------------------------+
| id_client  | nom            | adresse_complete                           |
+------------+----------------+--------------------------------------------+
|       1847 | Alice Martin   | 15 Rue de la Paix, 75002 Paris, France    |
|       2934 | Bob Dupont     | 42 Avenue Victor Hugo, 69003 Lyon, France |
|       3621 | Charlie Durand | 8 Boulevard Carnot, 13001 Marseille, France|
+------------+----------------+--------------------------------------------+
```

⚠️ **Piège NULL** :

```sql
-- ❌ Si une colonne est NULL, tout le résultat est NULL
SELECT CONCAT('Bonjour ', NULL, ' !');  -- Résultat : NULL

-- ✅ Solution : COALESCE pour gérer les NULL
SELECT CONCAT('Bonjour ', COALESCE(prenom, 'Inconnu'), ' !');
```

---

### CONCAT_WS() : Concaténation avec séparateur

**CONCAT_WS()** (With Separator) ajoute automatiquement un séparateur entre les valeurs et **ignore les NULL**.

```sql
CONCAT_WS(separateur, chaine1, chaine2, ..., chaineN)
```

#### Exemple 3 : Liste de tags

**Question métier** : *Créer une liste de catégories de produits*

```sql
SELECT
    id_produit,
    nom_produit,
    CONCAT_WS(' | ', categorie, sous_categorie, marque) AS tags
FROM produits
LIMIT 5;
```

**Résultat attendu** :
```
+------------+----------------------+-----------------------------------+
| id_produit | nom_produit          | tags                              |
+------------+----------------------+-----------------------------------+
|        101 | Laptop Dell XPS 15   | Informatique | Ordinateurs | Dell |
|        102 | iPhone 15 Pro        | Téléphones   | Smartphones | Apple|
|        103 | Chaise Bureau Ergo   | Mobilier     | Bureautique | NULL |
+------------+----------------------+-----------------------------------+
```

**Avantages CONCAT_WS** :
- ✅ Séparateur automatique
- ✅ Ignore les valeurs NULL (vs CONCAT qui retourne NULL)
- ✅ Plus lisible pour listes

#### Exemple 4 : Export CSV

**Question métier** : *Générer un export CSV des commandes*

```sql
SELECT
    CONCAT_WS(
        ';',                           -- Séparateur CSV
        id_commande,
        DATE_FORMAT(date_commande, '%Y-%m-%d'),
        id_client,
        REPLACE(montant_total, '.', ','),  -- Virgule décimale FR
        statut
    ) AS ligne_csv
FROM commandes
LIMIT 5;
```

**Résultat attendu** :
```
+------------------------------------+
| ligne_csv                          |
+------------------------------------+
| 1847;2025-12-10;2934;289,95;Payée  |
| 1848;2025-12-10;3621;149,99;Payée  |
| 1849;2025-12-11;1847;89,50;Envoyée |
+------------------------------------+
```

---

### GROUP_CONCAT() : Agrégation de chaînes

**GROUP_CONCAT()** agrège plusieurs valeurs en une seule chaîne (équivalent string de SUM/AVG).

```sql
GROUP_CONCAT([DISTINCT] colonne [ORDER BY colonne] [SEPARATOR 'sep'])
```

#### Exemple 5 : Liste de produits par commande

**Question métier** : *Afficher tous les produits d'une commande sur une ligne*

```sql
SELECT
    cmd.id_commande,
    cmd.id_client,
    GROUP_CONCAT(
        p.nom_produit
        ORDER BY p.nom_produit
        SEPARATOR ', '
    ) AS liste_produits,
    SUM(dc.quantite) AS total_articles
FROM commandes cmd
INNER JOIN details_commande dc ON cmd.id_commande = dc.id_commande
INNER JOIN produits p ON dc.id_produit = p.id_produit
GROUP BY cmd.id_commande, cmd.id_client
LIMIT 3;
```

**Résultat attendu** :
```
+-------------+-----------+----------------------------------------+----------------+
| id_commande | id_client | liste_produits                         | total_articles |
+-------------+-----------+----------------------------------------+----------------+
|        1847 |      2934 | Clavier, Souris Logitech, Écran 27"    |              3 |
|        1848 |      3621 | Casque Bluetooth, Webcam HD            |              2 |
|        1849 |      1847 | Câble USB-C, Chargeur MacBook          |              2 |
+-------------+-----------+----------------------------------------+----------------+
```

**Options GROUP_CONCAT** :
- `DISTINCT` : Élimine les doublons
- `ORDER BY` : Trie les valeurs
- `SEPARATOR` : Définit le séparateur (défaut : virgule)

#### Exemple 6 : Tags uniques

**Question métier** : *Liste des catégories de produits achetés par client*

```sql
SELECT
    c.id_client,
    c.nom,
    GROUP_CONCAT(
        DISTINCT p.categorie
        ORDER BY p.categorie
        SEPARATOR ' | '
    ) AS categories_achetees
FROM clients c
INNER JOIN commandes cmd ON c.id_client = cmd.id_client
INNER JOIN details_commande dc ON cmd.id_commande = dc.id_commande
INNER JOIN produits p ON dc.id_produit = p.id_produit
GROUP BY c.id_client, c.nom
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+----------------+-----------------------------------+
| id_client | nom            | categories_achetees               |
+-----------+----------------+-----------------------------------+
|      1847 | Alice Martin   | Électronique | Informatique       |
|      2934 | Bob Dupont     | Électronique | Mobilier | Sport   |
|      3621 | Charlie Durand | Informatique                      |
+-----------+----------------+-----------------------------------+
```

⚠️ **Limite GROUP_CONCAT** :

```sql
-- Limite par défaut : 1024 caractères
-- Augmenter avec :
SET SESSION group_concat_max_len = 10000;
```

---

## Transformation de casse

### UPPER() et LOWER() : Majuscules / Minuscules

```sql
UPPER(chaine)  -- Tout en MAJUSCULES
LOWER(chaine)  -- tout en minuscules
```

#### Exemple 7 : Normalisation pour comparaisons

**Question métier** : *Recherche insensible à la casse*

```sql
-- ❌ Sensible à la casse (peut manquer des résultats)
SELECT nom, email
FROM clients
WHERE email = 'alice@email.com';

-- ✅ Insensible à la casse
SELECT nom, email
FROM clients
WHERE LOWER(email) = LOWER('Alice@EMAIL.com');
```

**Cas d'usage** :
- Recherches utilisateur
- Dédoublonnage
- Normalisation de données

#### Exemple 8 : Formatage pour exports

**Question métier** : *Codes postaux en majuscules, noms en minuscules*

```sql
SELECT
    id_client,
    LOWER(prenom) AS prenom_format,
    UPPER(nom) AS nom_format,
    UPPER(code_postal) AS code_postal_format,
    ville
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+---------------+-------------+---------------------+-----------+
| id_client | prenom_format | nom_format  | code_postal_format  | ville     |
+-----------+---------------+-------------+---------------------+-----------+
|      1847 | alice         | MARTIN      | 75002               | Paris     |
|      2934 | bob           | DUPONT      | 69003               | Lyon      |
|      3621 | charlie       | DURAND      | 13001               | Marseille |
+-----------+---------------+-------------+---------------------+-----------+
```

---

## Extraction de sous-chaînes

### SUBSTRING() : Extraire une portion

```sql
SUBSTRING(chaine, position, longueur)
-- ou
SUBSTRING(chaine FROM position FOR longueur)
```

**Position** : Commence à 1 (pas 0 comme en programmation)

#### Exemple 9 : Extraire domaine email

**Question métier** : *Analyser les domaines d'email des clients*

```sql
SELECT
    email,
    SUBSTRING(
        email,
        LOCATE('@', email) + 1        -- Position après '@'
    ) AS domaine,
    COUNT(*) AS nb_clients
FROM clients
GROUP BY domaine
ORDER BY nb_clients DESC
LIMIT 5;
```

**Résultat attendu** :
```
+-------------------+--------------+-------------+
| email             | domaine      | nb_clients  |
+-------------------+--------------+-------------+
| alice@gmail.com   | gmail.com    |        1847 |
| bob@yahoo.fr      | yahoo.fr     |         934 |
| charlie@hotmail.fr| hotmail.fr   |         672 |
| diana@outlook.com | outlook.com  |         521 |
+-------------------+--------------+-------------+
```

#### Exemple 10 : Code postal - département

**Question métier** : *Extraire le département du code postal*

```sql
SELECT
    code_postal,
    ville,
    SUBSTRING(code_postal, 1, 2) AS departement
FROM clients
WHERE pays = 'France'
LIMIT 5;
```

**Résultat attendu** :
```
+-------------+-----------+--------------+
| code_postal | ville     | departement  |
+-------------+-----------+--------------+
| 75002       | Paris     | 75           |
| 69003       | Lyon      | 69           |
| 13001       | Marseille | 13           |
| 44000       | Nantes    | 44           |
| 33000       | Bordeaux  | 33           |
+-------------+-----------+--------------+
```

### LEFT() et RIGHT() : Début et fin de chaîne

```sql
LEFT(chaine, longueur)   -- N premiers caractères
RIGHT(chaine, longueur)  -- N derniers caractères
```

#### Exemple 11 : Initiales

**Question métier** : *Générer des initiales pour affichage compact*

```sql
SELECT
    nom,
    prenom,
    CONCAT(
        LEFT(prenom, 1),
        '. ',
        LEFT(nom, 1),
        '.'
    ) AS initiales
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+----------------+----------+------------+
| nom            | prenom   | initiales  |
+----------------+----------+------------+
| Martin         | Alice    | A. M.      |
| Dupont         | Bob      | B. D.      |
| Durand         | Charlie  | C. D.      |
| Lopez          | Diana    | D. L.      |
| Moreau         | Éric     | É. M.      |
+----------------+----------+------------+
```

#### Exemple 12 : Masquage partiel (RGPD)

**Question métier** : *Masquer une partie de l'email pour confidentialité*

```sql
SELECT
    id_client,
    email AS email_complet,
    CONCAT(
        LEFT(email, 3),                       -- 3 premiers caractères
        '***',                                -- Masquage
        RIGHT(email, LENGTH(email) - LOCATE('@', email) + 1)  -- Domaine complet
    ) AS email_masque
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+-------------------+-----------------+
| id_client | email_complet     | email_masque    |
+-----------+-------------------+-----------------+
|      1847 | alice@email.com   | ali***@email.com|
|      2934 | bob@email.com     | bob***@email.com|
|      3621 | charlie@email.com | cha***@email.com|
+-----------+-------------------+-----------------+
```

---

## Recherche dans les chaînes

### LOCATE() et POSITION() : Position d'une sous-chaîne

```sql
LOCATE(sous_chaine, chaine)           -- Retourne position (0 si absent)
POSITION(sous_chaine IN chaine)       -- SQL standard, équivalent
```

#### Exemple 13 : Validation format email

**Question métier** : *Trouver les emails invalides (sans @)*

```sql
SELECT
    id_client,
    nom,
    email,
    CASE
        WHEN LOCATE('@', email) = 0 THEN '❌ Invalide'
        WHEN LOCATE('.', email, LOCATE('@', email)) = 0 THEN '❌ Invalide'
        ELSE '✅ Valide'
    END AS validite_email
FROM clients
WHERE LOCATE('@', email) = 0
   OR LOCATE('.', email, LOCATE('@', email)) = 0;
```

**Explication validation** :
1. `LOCATE('@', email) = 0` : Pas d'arobase → invalide
2. `LOCATE('.', email, LOCATE('@', email)) = 0` : Pas de point après @ → invalide

#### Exemple 14 : Filtrage par contenu

**Question métier** : *Produits contenant "Pro" dans le nom*

```sql
SELECT
    id_produit,
    nom_produit,
    LOCATE('Pro', nom_produit) AS position_pro
FROM produits
WHERE LOCATE('Pro', nom_produit) > 0
ORDER BY nom_produit
LIMIT 5;
```

**Résultat attendu** :
```
+------------+---------------------+--------------+
| id_produit | nom_produit         | position_pro |
+------------+---------------------+--------------+
|        847 | iPhone 15 Pro       |          11  |
|       1024 | MacBook Pro 16"     |          9   |
|       2038 | Surface Pro 9       |          9   |
|       3192 | Logitech Pro X      |          10  |
+------------+---------------------+--------------+
```

### INSTR() : Variante de LOCATE()

```sql
INSTR(chaine, sous_chaine)  -- Équivalent à LOCATE mais arguments inversés
```

#### Exemple 15 : Extraction nom de fichier

**Question métier** : *Extraire le nom sans extension*

```sql
SELECT
    fichier,
    SUBSTRING(
        fichier,
        1,
        INSTR(fichier, '.') - 1
    ) AS nom_sans_extension,
    SUBSTRING(
        fichier,
        INSTR(fichier, '.') + 1
    ) AS extension
FROM documents
LIMIT 5;
```

**Résultat attendu** :
```
+---------------------+---------------------+-----------+
| fichier             | nom_sans_extension  | extension |
+---------------------+---------------------+-----------+
| rapport_2025.pdf    | rapport_2025        | pdf       |
| facture_1847.docx   | facture_1847        | docx      |
| photo_produit.jpg   | photo_produit       | jpg       |
+---------------------+---------------------+-----------+
```

---

## Nettoyage de chaînes

### TRIM(), LTRIM(), RTRIM() : Supprimer espaces

```sql
TRIM(chaine)        -- Supprime espaces début ET fin
LTRIM(chaine)       -- Supprime espaces à gauche (Left)
RTRIM(chaine)       -- Supprime espaces à droite (Right)
TRIM(caractere FROM chaine)  -- Supprime caractère spécifique
```

#### Exemple 16 : Nettoyage données importées

**Question métier** : *Nettoyer les noms avec espaces parasites*

```sql
-- Avant nettoyage
SELECT
    id_client,
    CONCAT('|', nom, '|') AS nom_brut,
    LENGTH(nom) AS longueur_brut
FROM clients_import
LIMIT 3;

-- Résultat :
-- |  Martin  | (longueur 10)
-- | Dupont   | (longueur 9)

-- Après nettoyage
SELECT
    id_client,
    CONCAT('|', TRIM(nom), '|') AS nom_nettoye,
    LENGTH(TRIM(nom)) AS longueur_nettoye
FROM clients_import
LIMIT 3;

-- Résultat :
-- |Martin| (longueur 6)
-- |Dupont| (longueur 6)
```

#### Exemple 17 : Normalisation téléphones

**Question métier** : *Retirer espaces et tirets des numéros*

```sql
SELECT
    telephone AS telephone_brut,
    REPLACE(
        REPLACE(
            TRIM(telephone),
            ' ', ''           -- Supprime espaces
        ),
        '-', ''               -- Supprime tirets
    ) AS telephone_normalise
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+-----------------+----------------------+
| telephone_brut  | telephone_normalise  |
+-----------------+----------------------+
| 06 01 02 03 04  | 0601020304           |
| 06-05-06-07-08  | 0605060708           |
| 06 12 34 56 78  | 0612345678           |
+-----------------+----------------------+
```

---

### REPLACE() : Remplacer du texte

```sql
REPLACE(chaine, ancien, nouveau)
```

#### Exemple 18 : Anonymisation

**Question métier** : *Remplacer voyelles par étoiles pour anonymisation*

```sql
SELECT
    nom AS nom_original,
    REPLACE(
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(nom, 'a', '*'),
                'e', '*'),
            'i', '*'),
        'o', '*'),
    'u', '*') AS nom_anonymise
FROM clients
LIMIT 5;
```

**Résultat attendu** :
```
+----------------+----------------+
| nom_original   | nom_anonymise  |
+----------------+----------------+
| Martin         | M*rt*n         |
| Dupont         | D*p*nt         |
| Durand         | D*r*nd         |
| Lopez          | L*p*z          |
| Moreau         | M*r***         |
+----------------+----------------+
```

#### Exemple 19 : Correction données

**Question métier** : *Corriger fautes de frappe courantes*

```sql
UPDATE produits
SET nom_produit = REPLACE(
    REPLACE(
        REPLACE(nom_produit, 'Iphone', 'iPhone'),
    'Macbook', 'MacBook'),
'Airpods', 'AirPods')
WHERE nom_produit LIKE '%Iphone%'
   OR nom_produit LIKE '%Macbook%'
   OR nom_produit LIKE '%Airpods%';
```

---

## Remplissage et formatage

### LPAD() et RPAD() : Remplir avec caractères

```sql
LPAD(chaine, longueur, caractere)  -- Remplit à gauche (Left Pad)
RPAD(chaine, longueur, caractere)  -- Remplit à droite (Right Pad)
```

#### Exemple 20 : Codes produits avec zéros

**Question métier** : *Formater les ID produits sur 8 caractères*

```sql
SELECT
    id_produit,
    LPAD(id_produit, 8, '0') AS code_produit,
    nom_produit
FROM produits
LIMIT 5;
```

**Résultat attendu** :
```
+------------+--------------+----------------------+
| id_produit | code_produit | nom_produit          |
+------------+--------------+----------------------+
|        101 | 00000101     | Laptop Dell XPS 15   |
|       1024 | 00001024     | MacBook Pro 16"      |
|      12847 | 00012847     | iPhone 15 Pro        |
+------------+--------------+----------------------+
```

**Application** : Codes-barres, références, exports formatés.

#### Exemple 21 : Alignement dans rapports texte

**Question métier** : *Créer un rapport aligné en texte*

```sql
SELECT
    RPAD(nom_produit, 30, ' ') AS produit,
    LPAD(FORMAT(prix_unitaire, 2), 10, ' ') AS prix,
    LPAD(stock, 5, ' ') AS stock
FROM produits
ORDER BY nom_produit
LIMIT 5;
```

**Résultat attendu** :
```
+--------------------------------+------------+-------+
| produit                        | prix       | stock |
+--------------------------------+------------+-------+
| AirPods Pro                    |     249.99 |   147 |
| Clavier Logitech MX Keys       |      99.99 |    89 |
| iPhone 15 Pro                  |    1229.00 |    23 |
| Laptop Dell XPS 15             |    1899.99 |    12 |
| Souris Logitech MX Master      |      89.99 |   234 |
+--------------------------------+------------+-------+
```

---

## Mesure de longueur

### LENGTH() vs CHAR_LENGTH()

```sql
LENGTH(chaine)       -- Longueur en OCTETS
CHAR_LENGTH(chaine)  -- Longueur en CARACTÈRES
```

⚠️ **Différence importante avec UTF-8** :

```sql
SELECT
    'Été' AS texte,
    LENGTH('Été') AS octets,        -- 4 (É = 2 octets en UTF-8)
    CHAR_LENGTH('Été') AS caracteres;  -- 3
```

#### Exemple 22 : Validation longueur champs

**Question métier** : *Vérifier les descriptions trop courtes ou trop longues*

```sql
SELECT
    id_produit,
    nom_produit,
    CHAR_LENGTH(description) AS longueur_desc,
    CASE
        WHEN CHAR_LENGTH(description) < 50 THEN '⚠️ Trop courte'
        WHEN CHAR_LENGTH(description) > 500 THEN '⚠️ Trop longue'
        ELSE '✅ OK'
    END AS validation
FROM produits
WHERE CHAR_LENGTH(description) < 50
   OR CHAR_LENGTH(description) > 500;
```

#### Exemple 23 : Statistiques de contenu

**Question métier** : *Longueur moyenne des commentaires clients*

```sql
SELECT
    ROUND(AVG(CHAR_LENGTH(commentaire)), 2) AS longueur_moy,
    MIN(CHAR_LENGTH(commentaire)) AS longueur_min,
    MAX(CHAR_LENGTH(commentaire)) AS longueur_max,
    COUNT(*) AS nb_commentaires
FROM avis_clients
WHERE commentaire IS NOT NULL;
```

**Résultat attendu** :
```
+--------------+--------------+--------------+------------------+
| longueur_moy | longueur_min | longueur_max | nb_commentaires  |
+--------------+--------------+--------------+------------------+
|       147.32 |           15 |          847 |             2847 |
+--------------+--------------+--------------+------------------+
```

---

## Autres fonctions utiles

### REVERSE() : Inverser une chaîne

```sql
SELECT
    mot,
    REVERSE(mot) AS mot_inverse
FROM (
    SELECT 'radar' AS mot
    UNION SELECT 'kayak'
    UNION SELECT 'level'
) AS palindromes;
```

**Résultat attendu** :
```
+-------+-------------+
| mot   | mot_inverse |
+-------+-------------+
| radar | radar       |
| kayak | kayak       |
| level | level       |
+-------+-------------+
```

**Application** : Détection palindromes, algorithmes spécifiques.

### REPEAT() : Répéter une chaîne

```sql
SELECT
    niveau,
    CONCAT(REPEAT('⭐', niveau), ' (', niveau, '/5)') AS etoiles
FROM (
    SELECT 1 AS niveau UNION SELECT 2 UNION SELECT 3
    UNION SELECT 4 UNION SELECT 5
) AS niveaux;
```

**Résultat attendu** :
```
+--------+-------------+
| niveau | etoiles     |
+--------+-------------+
|      1 | ⭐ (1/5)    |
|      2 | ⭐⭐ (2/5)  |
|      3 | ⭐⭐⭐ (3/5)|
|      4 | ⭐⭐⭐⭐ (4/5)|
|      5 | ⭐⭐⭐⭐⭐ (5/5)|
+--------+-------------+
```

### FORMAT() : Formatage numérique

```sql
FORMAT(nombre, decimales)  -- Ajoute séparateurs de milliers
```

#### Exemple 24 : Prix formatés

**Question métier** : *Afficher prix avec séparateurs de milliers*

```sql
SELECT
    nom_produit,
    prix_unitaire,
    CONCAT(FORMAT(prix_unitaire, 2), ' €') AS prix_format
FROM produits
WHERE prix_unitaire > 1000
ORDER BY prix_unitaire DESC
LIMIT 5;
```

**Résultat attendu** :
```
+----------------------+---------------+--------------+
| nom_produit          | prix_unitaire | prix_format  |
+----------------------+---------------+--------------+
| MacBook Pro 16"      |       2499.99 | 2,499.99 €   |
| Laptop Dell XPS 15   |       1899.99 | 1,899.99 €   |
| iPhone 15 Pro Max    |       1429.00 | 1,429.00 €   |
| TV Samsung QLED 65"  |       1299.00 | 1,299.00 €   |
+----------------------+---------------+--------------+
```

---

## Cas d'usage métier avancés

### Cas 1 : Génération de slugs URL

**Question métier** : *Créer des URL SEO-friendly pour produits*

```sql
SELECT
    id_produit,
    nom_produit,
    LOWER(
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(
                            REPLACE(nom_produit, ' ', '-'),
                        'é', 'e'),
                    'è', 'e'),
                'à', 'a'),
            'ç', 'c'),
        '''', '')
    ) AS slug_url
FROM produits
LIMIT 5;
```

**Résultat attendu** :
```
+------------+----------------------+-------------------------+
| id_produit | nom_produit          | slug_url                |
+------------+----------------------+-------------------------+
|        101 | Laptop Dell XPS 15   | laptop-dell-xps-15      |
|        102 | iPhone 15 Pro        | iphone-15-pro           |
|        103 | Écran QLED 27"       | ecran-qled-27           |
+------------+----------------------+-------------------------+
```

### Cas 2 : Détection doublons avec normalisation

**Question métier** : *Trouver clients en double malgré variations*

```sql
WITH clients_normalises AS (
    SELECT
        id_client,
        nom,
        email,
        LOWER(TRIM(REPLACE(nom, ' ', ''))) AS nom_normalise,
        LOWER(TRIM(email)) AS email_normalise
    FROM clients
)
SELECT
    cn1.id_client AS id_client1,
    cn1.nom AS nom1,
    cn2.id_client AS id_client2,
    cn2.nom AS nom2,
    cn1.email
FROM clients_normalises cn1
INNER JOIN clients_normalises cn2
    ON cn1.nom_normalise = cn2.nom_normalise
    AND cn1.email_normalise = cn2.email_normalise
    AND cn1.id_client < cn2.id_client
LIMIT 5;
```

**Application** : Nettoyage de base, fusion de comptes, dédoublonnage.

### Cas 3 : Validation format téléphone

**Question métier** : *Identifier les téléphones au mauvais format*

```sql
SELECT
    id_client,
    nom,
    telephone,
    CASE
        -- Format attendu : 10 chiffres (avec ou sans espaces/tirets)
        WHEN CHAR_LENGTH(REPLACE(REPLACE(telephone, ' ', ''), '-', '')) <> 10
            THEN '❌ Mauvaise longueur'
        -- Commence par 0
        WHEN LEFT(REPLACE(REPLACE(telephone, ' ', ''), '-', ''), 1) <> '0'
            THEN '❌ Ne commence pas par 0'
        -- Que des chiffres (après nettoyage)
        WHEN REPLACE(REPLACE(telephone, ' ', ''), '-', '') NOT REGEXP '^[0-9]+$'
            THEN '❌ Contient caractères invalides'
        ELSE '✅ Valide'
    END AS validation_telephone
FROM clients
WHERE telephone IS NOT NULL
HAVING validation_telephone <> '✅ Valide';
```

### Cas 4 : Extraction données structurées

**Question métier** : *Parser un champ "Nom Prénom (Email)" en colonnes séparées*

```sql
-- Format : "Martin Alice (alice@email.com)"
SELECT
    contact_complet,
    SUBSTRING_INDEX(contact_complet, ' ', 1) AS nom,
    SUBSTRING_INDEX(
        SUBSTRING_INDEX(contact_complet, '(', 1),
        ' ',
        -1
    ) AS prenom,
    TRIM(
        SUBSTRING_INDEX(
            SUBSTRING_INDEX(contact_complet, '(', -1),
            ')',
            1
        )
    ) AS email
FROM contacts_import
LIMIT 5;
```

**Résultat attendu** :
```
+-------------------------------+--------+--------+-------------------+
| contact_complet               | nom    | prenom | email             |
+-------------------------------+--------+--------+-------------------+
| Martin Alice (alice@email.com)| Martin | Alice  | alice@email.com   |
| Dupont Bob (bob@email.com)    | Dupont | Bob    | bob@email.com     |
+-------------------------------+--------+--------+-------------------+
```

---

## Performance et optimisation

### Impact des fonctions sur les index

⚠️ **Règle critique** : Les fonctions appliquées sur colonnes **empêchent l'utilisation des index**.

```sql
-- ❌ LENT : Index non utilisé
SELECT * FROM clients
WHERE LOWER(email) = 'alice@email.com';

-- ✅ RAPIDE : Index utilisé
SELECT * FROM clients
WHERE email = 'alice@email.com';

-- Solution : Colonne calculée avec index
ALTER TABLE clients ADD COLUMN email_lower VARCHAR(255)
    GENERATED ALWAYS AS (LOWER(email)) STORED;
CREATE INDEX idx_email_lower ON clients(email_lower);
```

### Colonnes calculées (Generated Columns)

```sql
-- Créer une colonne calculée stockée
ALTER TABLE clients
ADD COLUMN nom_complet VARCHAR(200)
    GENERATED ALWAYS AS (CONCAT(prenom, ' ', nom)) STORED;

-- Index sur colonne calculée
CREATE INDEX idx_nom_complet ON clients(nom_complet);

-- Requête optimisée
SELECT * FROM clients
WHERE nom_complet LIKE 'Alice%';  -- Utilise l'index !
```

### Exemple 25 : Optimisation recherche

```sql
-- Table avec colonnes normalisées
CREATE TABLE clients_optimise (
    id_client INT PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    -- Colonnes calculées pour recherche
    nom_normalise VARCHAR(100)
        GENERATED ALWAYS AS (LOWER(TRIM(nom))) STORED,
    email_normalise VARCHAR(255)
        GENERATED ALWAYS AS (LOWER(TRIM(email))) STORED,
    INDEX idx_nom_norm (nom_normalise),
    INDEX idx_email_norm (email_normalise)
);

-- Recherche ultra-rapide
SELECT * FROM clients_optimise
WHERE nom_normalise = 'martin';  -- Index utilisé !
```

---

## Pièges courants et solutions

### Piège 1 : NULL avec CONCAT()

```sql
-- ❌ Résultat NULL si une partie est NULL
SELECT CONCAT('Bonjour ', NULL, ' !');  -- NULL

-- ✅ Utiliser CONCAT_WS ou COALESCE
SELECT CONCAT('Bonjour ', COALESCE(prenom, 'Inconnu'), ' !');
SELECT CONCAT_WS(' ', 'Bonjour', prenom, '!');  -- Ignore NULL
```

### Piège 2 : Position commence à 1

```sql
-- ❌ ERREUR : Position 0 n'existe pas
SELECT SUBSTRING(nom, 0, 3);  -- Retourne chaîne vide

-- ✅ Position commence à 1
SELECT SUBSTRING(nom, 1, 3);  -- Correct
```

### Piège 3 : LOCATE retourne 0 si absent

```sql
-- ❌ Faux : LOCATE ne retourne pas NULL
SELECT * FROM clients
WHERE LOCATE('@', email) IS NULL;  -- Aucun résultat

-- ✅ Correct : Tester avec = 0
SELECT * FROM clients
WHERE LOCATE('@', email) = 0;
```

### Piège 4 : Sensibilité à la casse

```sql
-- ⚠️ Dépend du COLLATION de la colonne
SELECT * FROM clients
WHERE nom = 'martin';  -- Peut ou non trouver 'Martin'

-- ✅ Explicite : Forcer insensibilité
SELECT * FROM clients
WHERE LOWER(nom) = LOWER('martin');

-- Ou définir COLLATION :
WHERE nom = 'martin' COLLATE utf8mb4_general_ci;
```

### Piège 5 : LENGTH vs CHAR_LENGTH

```sql
-- ❌ Erreur avec caractères multi-octets
SELECT * FROM produits
WHERE LENGTH(nom_produit) > 100;  -- Compte OCTETS, pas caractères

-- ✅ Correct : Compter caractères
SELECT * FROM produits
WHERE CHAR_LENGTH(nom_produit) > 100;
```

---

## Bonnes pratiques

### ✅ DO : Recommandations

1. **CONCAT_WS pour listes** : Gère automatiquement les NULL
2. **TRIM systématique** : Nettoyer données saisies par utilisateurs
3. **CHAR_LENGTH pour validations** : Compter caractères, pas octets
4. **Colonnes calculées** : Pour recherches fréquentes sur fonctions
5. **LOWER/UPPER pour comparaisons** : Recherches insensibles à la casse
6. **COALESCE avec CONCAT** : Éviter résultats NULL inattendus
7. **Valider format** : Emails, téléphones, codes postaux avant insertion

### ❌ DON'T : À éviter

1. **Fonctions dans WHERE avec gros volumes** : Empêche usage index
2. **CONCAT sans gérer NULL** : Résultat entier devient NULL
3. **Négliger espaces parasites** : TRIM systématique sur imports
4. **Position 0** : Positions commencent à 1 en SQL
5. **Tester LOCATE() avec IS NULL** : Utiliser = 0
6. **LENGTH pour UTF-8** : Utiliser CHAR_LENGTH
7. **REPLACE imbriqués excessifs** : Envisager REGEXP_REPLACE

---

## Tableau récapitulatif des fonctions

| Catégorie | Fonction | Usage | Exemple |
|-----------|----------|-------|---------|
| **Concaténation** | CONCAT() | Assembler | `CONCAT(prenom, ' ', nom)` |
| | CONCAT_WS() | Avec séparateur | `CONCAT_WS(', ', ville, pays)` |
| | GROUP_CONCAT() | Agrégation | `GROUP_CONCAT(produit ORDER BY nom)` |
| **Casse** | UPPER() | Majuscules | `UPPER(email)` |
| | LOWER() | Minuscules | `LOWER(nom)` |
| **Extraction** | SUBSTRING() | Sous-chaîne | `SUBSTRING(code, 1, 5)` |
| | LEFT() | N premiers | `LEFT(nom, 1)` |
| | RIGHT() | N derniers | `RIGHT(tel, 4)` |
| **Recherche** | LOCATE() | Position | `LOCATE('@', email)` |
| | INSTR() | Position (inv.) | `INSTR(email, '@')` |
| **Nettoyage** | TRIM() | Espaces | `TRIM(nom)` |
| | REPLACE() | Remplacer | `REPLACE(tel, ' ', '')` |
| **Mesure** | LENGTH() | Octets | `LENGTH(texte)` |
| | CHAR_LENGTH() | Caractères | `CHAR_LENGTH(texte)` |
| **Formatage** | LPAD() | Remplit gauche | `LPAD(id, 8, '0')` |
| | RPAD() | Remplit droite | `RPAD(nom, 20, ' ')` |
| | FORMAT() | Nombre | `FORMAT(prix, 2)` |
| **Autres** | REVERSE() | Inverser | `REVERSE(mot)` |
| | REPEAT() | Répéter | `REPEAT('*', 5)` |

---

## ✅ Points clés à retenir

1. **CONCAT vs CONCAT_WS** : WS ignore NULL et ajoute séparateur automatiquement

2. **GROUP_CONCAT pour agrégations** : Assemble plusieurs valeurs en une chaîne (avec DISTINCT, ORDER BY, SEPARATOR)

3. **UPPER/LOWER pour recherches** : Comparaisons insensibles à la casse

4. **SUBSTRING, LEFT, RIGHT** : Extraction de portions (position commence à 1)

5. **LOCATE retourne 0 si absent** : Pas NULL, tester avec `= 0`

6. **TRIM essentiel pour nettoyage** : Supprimer espaces parasites (LTRIM/RTRIM)

7. **REPLACE pour transformations** : Remplacer caractères/sous-chaînes

8. **CHAR_LENGTH vs LENGTH** : Caractères vs octets (important UTF-8)

9. **Fonctions empêchent index** : Utiliser colonnes calculées pour optimiser

10. **LPAD/RPAD pour formatage** : Codes avec zéros, alignement rapports

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 String Functions](https://mariadb.com/kb/en/string-functions/) – Documentation complète
- [📖 Regular Expressions](https://mariadb.com/kb/en/regular-expressions-overview/) – REGEXP avancé
- [📖 Generated Columns](https://mariadb.com/kb/en/generated-columns/) – Optimisation

### Articles approfondis
- [String Manipulation](https://modern-sql.com/feature/string-functions) – Guide moderne
- [Performance Tips](https://use-the-index-luke.com/sql/where-clause/functions) – Impact index

---

## ➡️ Prochaine section

**3.7 Fonctions de dates et heures**

Prêt à manipuler les dates ? La prochaine section couvre :
- Fonctions DATE, TIME, DATETIME
- Calculs de durées et intervalles
- Formatage de dates
- Fuseaux horaires
- Séries temporelles

Continuons la maîtrise de SQL ! 🚀

---


⏭️ [Fonctions de dates et heures](/03-requetes-sql-intermediaires/07-fonctions-dates-heures.md)

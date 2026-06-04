🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6 Fonctions de chaînes de caractères

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.6  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Les fonctions de chaînes transforment, découpent, recherchent et assemblent du texte directement dans les requêtes. Elles sont omniprésentes : normaliser une saisie, extraire un fragment, construire un libellé d'affichage, nettoyer des données importées. Cette section présente les plus utiles, regroupées par usage.

Un préalable conditionne plusieurs d'entre elles : **l'encodage et la collation**.

---

## Préalable : encodage et collation

Depuis la 11.8, le jeu de caractères par défaut de MariaDB est **`utf8mb4`** (voir [section 11.11](../11-administration-configuration/11-charset-utf8mb4-uca14.md)). Deux conséquences directes pour les fonctions de chaînes :

- **Octets ≠ caractères.** En `utf8mb4`, un caractère accentué ou un emoji occupe **plusieurs octets**. La longueur en octets et la longueur en caractères diffèrent donc — d'où la distinction `LENGTH` / `CHAR_LENGTH` ci-dessous.
- **Sensibilité à la casse selon la collation.** Avec une collation *insensible à la casse* (suffixe `_ci`, le défaut courant), `'abc' = 'ABC'` est vrai ; avec une collation binaire ou `_cs`, non. La casse des comparaisons et de `LIKE` dépend donc de la collation de la colonne, pas de la fonction.

---

## Longueur d'une chaîne

| Fonction | Renvoie |
|----------|---------|
| `CHAR_LENGTH(str)` | la longueur en **caractères** |
| `LENGTH(str)` | la longueur en **octets** |

```sql
SELECT LENGTH('Café') AS octets, CHAR_LENGTH('Café') AS caracteres;
```

| octets | caracteres |
|--------|------------|
| 5 | 4 |

`'Café'` compte 4 caractères, mais le `é` occupant 2 octets en `utf8mb4`, sa taille en octets est 5. **Pour compter des caractères, utilisez toujours `CHAR_LENGTH`** ; `LENGTH` ne convient que pour des données binaires ou un calcul de stockage.

---

## Changer la casse

`UPPER(str)` (alias `UCASE`) et `LOWER(str)` (alias `LCASE`) convertissent la casse, en tenant compte des caractères accentués grâce à `utf8mb4` :

```sql
SELECT UPPER('café') AS maj, LOWER('PARIS') AS min;
-- 'CAFÉ', 'paris'
```

---

## Concaténer

`CONCAT(a, b, …)` assemble plusieurs valeurs. Attention : **si l'un des arguments est `NULL`, le résultat entier est `NULL`**. `CONCAT_WS(séparateur, a, b, …)` (« with separator ») insère un séparateur entre les éléments et, surtout, **ignore les arguments `NULL`** au lieu de tout annuler :

```sql
SELECT
    CONCAT('Paris', NULL, 'France')          AS avec_concat,
    CONCAT_WS(', ', 'Paris', NULL, 'France') AS avec_concat_ws;
```

| avec_concat | avec_concat_ws |
|-------------|----------------|
| *NULL* | Paris, France |

`CONCAT_WS` est donc le bon choix pour assembler des champs potentiellement nuls (prénom, nom, complément d'adresse…).

> ℹ️ **Migration.** MariaDB n'utilise **pas** `||` pour la concaténation par défaut (`||` y est l'opérateur logique `OR`). En mode `ORACLE` ou `PIPES_AS_CONCAT`, `||` concatène comme en Oracle/PostgreSQL — un point à connaître lors d'une migration (voir [section 11.3](../11-administration-configuration/03-modes-sql.md)).

---

## Extraire une sous-chaîne

| Fonction | Effet |
|----------|-------|
| `SUBSTRING(str, pos[, len])` | sous-chaîne à partir de `pos` (alias `SUBSTR`, `MID`) |
| `LEFT(str, len)` | les `len` premiers caractères |
| `RIGHT(str, len)` | les `len` derniers caractères |
| `SUBSTRING_INDEX(str, délim, n)` | la portion avant/après la n-ième occurrence d'un délimiteur |

Les positions sont **indexées à partir de 1** ; une position négative compte depuis la fin. `SUBSTRING_INDEX` est particulièrement pratique pour découper une valeur structurée — par exemple isoler le domaine d'une adresse e-mail (tout ce qui suit le dernier `@`) :

```sql
SELECT
    email,
    SUBSTRING_INDEX(email, '@', -1) AS domaine,
    SUBSTRING_INDEX(email, '@',  1) AS identifiant
FROM client_web;
```

| email | domaine | identifiant |
|-------|---------|-------------|
| alice@ex.fr | ex.fr | alice |
| bruno@ex.fr | ex.fr | bruno |
| chloe@ex.fr | ex.fr | chloe |

Un `n` positif renvoie la partie **avant** la n-ième occurrence, un `n` négatif la partie **après**.

---

## Localiser une sous-chaîne

`LOCATE(sous_chaîne, str[, pos])` renvoie la position (à partir de 1) de la première occurrence, ou `0` si absente :

```sql
SELECT LOCATE('@', 'alice@ex.fr') AS position;   -- 6
```

Variantes : `INSTR(str, sous_chaîne)` (mêmes résultats mais **ordre des arguments inversé**) et `POSITION(sous_chaîne IN str)` (syntaxe standard).

---

## Remplacer et nettoyer

- **`REPLACE(str, ancien, nouveau)`** remplace **toutes** les occurrences d'une sous-chaîne :

```sql
SELECT REPLACE('2026-06-03', '-', '/') AS date_fr;   -- '2026/06/03'
```

- **`TRIM(str)`** supprime les espaces de début et de fin ; `LTRIM`/`RTRIM` n'agissent que d'un côté. La forme complète retire un caractère donné :

```sql
SELECT TRIM(BOTH '*' FROM '***promo***') AS nettoye;   -- 'promo'
```

`TRIM` accepte `BOTH`, `LEADING` ou `TRAILING` pour cibler le début, la fin ou les deux extrémités.

---

## Remplir, répéter, inverser

| Fonction | Effet |
|----------|-------|
| `LPAD(str, len, pad)` / `RPAD(...)` | complète à gauche / à droite jusqu'à `len` |
| `REPEAT(str, n)` | répète `n` fois |
| `SPACE(n)` | renvoie `n` espaces |
| `REVERSE(str)` | inverse les caractères |

`LPAD` est l'idiome classique pour formater un identifiant à largeur fixe :

```sql
SELECT LPAD(42, 5, '0') AS reference;   -- '00042'
```

---

## Formater un nombre en chaîne

`FORMAT(nombre, décimales)` produit une **chaîne** avec séparateurs de milliers et nombre de décimales fixé (utile pour l'affichage, mais inadapté aux calculs ultérieurs puisque le résultat n'est plus numérique) :

```sql
SELECT FORMAT(1234567.891, 2);   -- '1,234,567.89'
```

---

## Recherche par motif : LIKE

L'opérateur `LIKE` filtre selon un motif simple, avec deux jokers : **`%`** (toute séquence, y compris vide) et **`_`** (un caractère unique) :

```sql
SELECT nom FROM client WHERE nom LIKE 'D%';   -- noms commençant par D
```

La sensibilité à la casse de `LIKE` suit la **collation** de la colonne (cf. préalable). Pour rechercher un `%` ou `_` littéral, on définit un caractère d'échappement avec `ESCAPE`. Pour des motifs plus puissants (classes de caractères, quantificateurs, ancres), on passe aux **expressions régulières** `REGEXP`, traitées en [section 4.11](../04-concepts-avances-sql/11-expressions-regulieres.md).

---

## Valeurs NULL et fonctions de chaînes

La plupart des fonctions de chaînes **renvoient `NULL` si leur argument l'est** (`UPPER(NULL)`, `SUBSTRING(NULL, …)`, et `CONCAT` dès qu'un argument est nul). Les exceptions notables sont `CONCAT_WS`, qui ignore les `NULL`, et les comparaisons. Pour neutraliser un `NULL` avant traitement, enveloppez la valeur dans `COALESCE(col, '')` (voir [section 3.8](08-expressions-conditionnelles.md)).

---

## Tableau récapitulatif

| Catégorie | Fonctions principales |
|-----------|------------------------|
| Longueur | `CHAR_LENGTH`, `LENGTH` |
| Casse | `UPPER`/`UCASE`, `LOWER`/`LCASE` |
| Concaténation | `CONCAT`, `CONCAT_WS` |
| Extraction | `SUBSTRING`/`SUBSTR`/`MID`, `LEFT`, `RIGHT`, `SUBSTRING_INDEX` |
| Localisation | `LOCATE`, `INSTR`, `POSITION` |
| Nettoyage | `REPLACE`, `TRIM`, `LTRIM`, `RTRIM` |
| Remplissage | `LPAD`, `RPAD`, `REPEAT`, `SPACE`, `REVERSE` |
| Formatage | `FORMAT` |
| Motif | `LIKE` (jokers `%`, `_`) |

---

## À retenir

- En `utf8mb4`, distinguez **`CHAR_LENGTH`** (caractères) de **`LENGTH`** (octets) ; utilisez `CHAR_LENGTH` pour compter des caractères.
- `CONCAT` renvoie `NULL` si un argument est nul ; **`CONCAT_WS` ignore les `NULL`** et insère un séparateur.
- `||` ne concatène **pas** par défaut sous MariaDB (sauf mode `ORACLE`/`PIPES_AS_CONCAT`).
- `SUBSTRING_INDEX` découpe autour d'un délimiteur (n positif = avant, n négatif = après) ; `LEFT`/`RIGHT`/`SUBSTRING` pour extraire par position (indexée à 1).
- `LOCATE`/`INSTR`/`POSITION` localisent ; `REPLACE` remplace toutes les occurrences ; `TRIM` (`BOTH`/`LEADING`/`TRAILING`) nettoie ; `LPAD`/`RPAD` formatent à largeur fixe.
- La sensibilité à la casse de `LIKE` et des comparaisons dépend de la **collation** ; pour des motifs avancés, voir `REGEXP` ([section 4.11](../04-concepts-avances-sql/11-expressions-regulieres.md)).

---

## Navigation

- ⬅️ Section précédente : [3.5 — Opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)](05-operateurs-ensemblistes.md)
- ➡️ Section suivante : [3.7 — Fonctions de dates et heures](07-fonctions-dates-heures.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Fonctions de dates et heures](/03-requetes-sql-intermediaires/07-fonctions-dates-heures.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.11 Expressions régulières (`REGEXP`, `REGEXP_REPLACE`, `REGEXP_SUBSTR`)

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Moteur PCRE2 depuis MariaDB 10.5.1 (PCRE dès 10.0.5) · Référence : **MariaDB 12.3 LTS**

Les **expressions régulières** permettent de rechercher, d'extraire et de remplacer du texte selon des **motifs**, bien au-delà de ce que permet `LIKE`. MariaDB s'appuie pour cela sur le moteur **PCRE2** (*Perl Compatible Regular Expressions*), qui offre une syntaxe riche et standard. Cette section présente l'opérateur de correspondance et les principales fonctions, la syntaxe essentielle des motifs, et les points d'attention liés à la casse et à la performance.

## 🎯 Objectif de la section

Maîtriser l'opérateur `REGEXP` et les fonctions `REGEXP_SUBSTR`, `REGEXP_REPLACE` et `REGEXP_INSTR`, connaître la syntaxe PCRE de base, gérer la sensibilité à la casse et comprendre l'incidence des expressions régulières sur la performance.

## Le moteur : PCRE2

MariaDB utilise la bibliothèque **PCRE** depuis la 10.0.5, passée à **PCRE2** depuis la 10.5.1 — c'est donc PCRE2 qui équipe la 12.3. Ce moteur apporte la syntaxe complète des expressions régulières de type Perl : classes de caractères, ancres, quantificateurs, alternatives, groupes, références arrière, etc. C'est un point de **compatibilité** à connaître : MySQL 8, lui, s'appuie sur le moteur **ICU**, dont la syntaxe diffère sur certains détails. Un motif élaboré peut donc ne pas se comporter strictement à l'identique entre les deux SGBD.

> ⚠️ **Échappement dans les littéraux.** Par défaut, l'antislash est un caractère d'échappement dans les chaînes SQL. Pour transmettre une séquence comme `\d`, `\.` ou `\b` au moteur d'expressions régulières, il faut **doubler l'antislash** dans le littéral : `'\\d'`, `'\\.'`, `'\\b'` (sauf si le mode SQL `NO_BACKSLASH_ESCAPES` est actif). Les exemples ci-dessous suivent cette règle.

## L'opérateur `REGEXP` (alias `RLIKE`)

`expr REGEXP motif` renvoie `1` si l'expression correspond au motif, `0` sinon ; `RLIKE` en est un synonyme, et `NOT REGEXP` la négation. Il s'emploie typiquement dans un `WHERE` :

```sql
SELECT 'MariaDB 12.3' REGEXP '[0-9]+\\.[0-9]+';     -- 1 (contient un numéro de version)

SELECT email FROM clients
WHERE email REGEXP '@example\\.(com|org)$';          -- adresses se terminant par @example.com/.org

SELECT nom FROM produits
WHERE nom NOT REGEXP '^[A-Z]';                        -- initiale hors A–Z (la casse suit la collation : voir plus bas)
```

## `REGEXP_SUBSTR` : extraire

`REGEXP_SUBSTR(sujet, motif)` renvoie la **première sous-chaîne** correspondant au motif (ou `NULL` si aucune) :

```sql
SELECT REGEXP_SUBSTR('Commande #4567 du 2026-06-04', '[0-9]+');   -- 4567 (premier nombre)
SELECT REGEXP_SUBSTR('contact : jean@example.com', '[\\w.]+@[\\w.]+'); -- jean@example.com
```

## `REGEXP_REPLACE` : remplacer

`REGEXP_REPLACE(sujet, motif, remplacement)` remplace **toutes** les occurrences du motif. Le remplacement peut réutiliser les groupes capturés au moyen de **références arrière** `\1`, `\2`, etc. :

```sql
-- Réordonner une date AAAA-MM-JJ → JJ/MM/AAAA grâce aux groupes capturés
SELECT REGEXP_REPLACE('2026-06-04',
                      '([0-9]{4})-([0-9]{2})-([0-9]{2})',
                      '\\3/\\2/\\1');                  -- 04/06/2026

-- Normaliser les espaces multiples en un seul
SELECT REGEXP_REPLACE('trop    d''espaces', ' +', ' ');  -- "trop d'espaces"
```

## `REGEXP_INSTR` : localiser

`REGEXP_INSTR(sujet, motif)` renvoie la **position** (à partir de 1) de la première correspondance, ou `0` si aucune :

```sql
SELECT REGEXP_INSTR('abc123def', '[0-9]+');   -- 4 (position du premier chiffre)
```

## Syntaxe PCRE : l'essentiel

Les éléments les plus courants des motifs :

| Élément              | Signification                          | Exemple        |
|----------------------|----------------------------------------|----------------|
| `.`                  | n'importe quel caractère               | `a.c`          |
| `\d` `\w` `\s`       | chiffre / caractère de mot / espace    | `\d+`          |
| `[abc]` `[^abc]` `[a-z]` | classe / négation / intervalle     | `[A-Za-z]`     |
| `^` `$`              | début / fin de chaîne                  | `^abc$`        |
| `\b`                 | frontière de mot                       | `\bmot\b`      |
| `*` `+` `?`          | 0 ou plus / 1 ou plus / 0 ou 1         | `a+`           |
| `{n}` `{n,m}`        | nombre de répétitions                  | `\d{4}`        |
| `\|`                 | alternative                            | `chat\|chien`  |
| `( )`                | groupe de capture                      | `(ab)+`        |
| `\1`                 | référence arrière (dans le remplacement)| `\1`           |

(Rappel : ces antislashs doivent être doublés dans les chaînes SQL.)

## Sensibilité à la casse

La sensibilité à la casse de `REGEXP` dépend par défaut de la **collation** de l'expression : une collation insensible (`utf8mb4_uca1400_ai_ci`, `..._general_ci`) rend la correspondance insensible à la casse, tandis qu'une collation sensible (`..._cs`) ou binaire (`..._bin`) la rend sensible. On peut forcer le comportement au moyen des indicateurs PCRE en ligne `(?i)` (insensible) et `(?-i)` (sensible) :

```sql
SELECT 'HELLO' REGEXP '(?i)hello';   -- 1 (forcé insensible)
SELECT 'HELLO' REGEXP '(?-i)hello';  -- 0 (forcé sensible)
```

La variable système `default_regex_flags` permet par ailleurs de définir des indicateurs PCRE par défaut (par exemple `DOTALL`, `MULTILINE`, `EXTENDED`, `UNGREEDY`).

## Performance et bonnes pratiques

Un filtre `REGEXP` dans un `WHERE` **ne peut pas exploiter un index** B-Tree classique : le moteur évalue le motif ligne par ligne, ce qui revient à un balayage. Pour une simple correspondance de préfixe, `LIKE 'préfixe%'` est *sargable* (utilisable par un index) et donc préférable ; les expressions régulières sont à réserver aux motifs réellement complexes. Pour de la recherche textuelle par mots à grande échelle, un **index FULLTEXT** (§ 5.2.3) est bien plus performant que `REGEXP`. En résumé : puissantes mais coûteuses, les expressions régulières gagnent à filtrer un ensemble déjà réduit par d'autres conditions indexées.

## Points clés à retenir

MariaDB s'appuie sur le moteur **PCRE2** (depuis 10.5.1 ; PCRE auparavant, dès 10.0.5) — syntaxe riche de type Perl, distincte du moteur ICU de MySQL 8. L'opérateur **`REGEXP`** (alias `RLIKE`) teste une correspondance ; **`REGEXP_SUBSTR`** extrait la première sous-chaîne correspondante, **`REGEXP_REPLACE`** remplace les occurrences (avec références arrière `\1`, `\2`…), et **`REGEXP_INSTR`** renvoie la position d'une correspondance. Dans les littéraux SQL, les antislashs des motifs doivent être **doublés**. La **casse** suit la collation, ajustable par `(?i)`/`(?-i)` ou `default_regex_flags`. Enfin, `REGEXP` **n'utilise pas d'index** : on lui préfère `LIKE 'x%'` pour un préfixe et un index `FULLTEXT` pour la recherche de mots, en réservant les expressions régulières aux motifs complexes sur des ensembles déjà filtrés.

---

**Section précédente :** [4.10 — Indexation de colonnes virtuelles extraites du JSON](10-indexation-colonnes-virtuelles-json.md)  
**Fin du chapitre 4.** Retour au [sommaire du chapitre](README.md) · Chapitre suivant : [5 — Index et Performance](../05-index-et-performance/README.md)  

⏭️ [Partie 3 : Index, Transactions et Performance (Intermédiaire)](/partie-03-index-transactions-performance.md)

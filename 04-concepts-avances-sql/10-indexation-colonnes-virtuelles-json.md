🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.10 Indexation de colonnes virtuelles extraites du JSON

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Colonnes virtuelles indexables depuis 10.2 · Reconnaissance de l'expression par l'optimiseur depuis 11.8 · Référence : **MariaDB 12.3 LTS**

Nous l'avons souligné à plusieurs reprises (§ 4.7.1, § 4.7.2) : comme le JSON est stocké en texte, MariaDB **analyse le document entier** à chaque accès et **n'indexe pas directement** les chemins JSON. Filtrer fréquemment sur un champ enfoui devient alors coûteux. La solution standard, objet de cette section, consiste à **extraire le champ dans une colonne virtuelle**, puis à **indexer cette colonne** — conciliant la souplesse du JSON et la performance d'un index.

## 🎯 Objectif de la section

Savoir indexer une valeur JSON via une colonne virtuelle, exploiter la reconnaissance de l'expression par l'optimiseur (11.8+), choisir entre colonne `VIRTUAL` et `PERSISTENT`, et comprendre la différence avec les index fonctionnels de MySQL.

## Le problème : pas d'index direct sur le JSON

Un index sur la colonne JSON elle-même (un `LONGTEXT`) n'aurait aucun intérêt pour interroger un champ interne : il indexerait la chaîne entière. Et, contrairement à MySQL, **MariaDB ne propose pas d'index fonctionnel** — c'est-à-dire un index posé directement sur une expression. Il faut donc matérialiser l'expression d'extraction dans une **colonne dédiée**, puis l'indexer.

## La solution : colonne virtuelle + index

La démarche se fait en deux temps : ajouter une **colonne virtuelle** (colonne générée) qui extrait et **type** la valeur voulue, puis créer un **index** sur cette colonne.

```sql
CREATE TABLE produits (
    id        INT PRIMARY KEY,
    attributs JSON
);

ALTER TABLE produits
    ADD COLUMN stock INT
        AS (CAST(JSON_VALUE(attributs, '$.stock') AS INTEGER)),
    ADD INDEX idx_stock (stock);
```

Deux points méritent attention. D'abord, le **typage** : `JSON_VALUE` renvoie par défaut une chaîne ; le `CAST(... AS INTEGER)` garantit que la colonne — et donc l'index — porte sur un entier, indispensable pour des comparaisons et des tris numériques corrects. Ensuite, pour une valeur **textuelle**, la **collation** compte : il est recommandé de l'aligner entre la colonne et l'expression d'extraction.

```sql
ALTER TABLE produits
    ADD COLUMN couleur VARCHAR(50) COLLATE utf8mb4_uca1400_ai_ci
        AS (JSON_VALUE(attributs, '$.couleur') COLLATE utf8mb4_uca1400_ai_ci),
    ADD INDEX idx_couleur (couleur);
```

## Interroger : la colonne ou l'expression (11.8+)

Une fois l'index en place, l'optimiseur l'exploite pour construire des accès par intervalle ou par égalité. La façon d'écrire la requête a toutefois évolué :

```sql
-- Fonctionne depuis toujours : on référence la colonne virtuelle
SELECT * FROM produits WHERE stock < 10;

-- Depuis MariaDB 11.8 : l'expression JSON elle-même utilise aussi l'index
SELECT * FROM produits
WHERE CAST(JSON_VALUE(attributs, '$.stock') AS INTEGER) < 10;
```

Avant la 11.8, seule la première forme — référencer le nom de la colonne virtuelle — déclenchait l'usage de l'index. Depuis la **11.8** (donc en 12.3), l'optimiseur **reconnaît l'expression d'extraction** dans le `WHERE` et la rattache à la colonne virtuelle indexée correspondante : les deux écritures bénéficient de l'index. Il reste prudent de **vérifier le plan** avec `EXPLAIN` (§ 5.7) pour confirmer que l'index est bien utilisé.

## `VIRTUAL` ou `PERSISTENT` ?

Une colonne générée peut être **`VIRTUAL`** (calculée à la lecture, sans stockage dans la ligne — le comportement par défaut) ou **`PERSISTENT`** (calculée à l'écriture et stockée). MariaDB sait **indexer les deux** ; et, depuis la 10.2, la colonne n'a plus besoin d'être persistante pour être indexée.

Pour l'indexation du JSON, la variante **`VIRTUAL`** est généralement préférable : la valeur n'occupe pas d'espace dans la ligne, et c'est l'index qui matérialise les valeurs nécessaires aux recherches. La variante `PERSISTENT` se justifie surtout lorsque l'expression est coûteuse et la colonne fréquemment lue **hors index**. Le choix `VIRTUAL` vs `PERSISTENT` et l'indexation des colonnes générées sont traités plus largement au § 18.4.

## Pas d'index fonctionnel : la différence avec MySQL

MySQL permet, depuis la version 8.0.13, de créer un **index fonctionnel** directement sur une expression — `CREATE INDEX ... ((expression))` — qui crée en coulisses une colonne générée masquée. **MariaDB ne dispose pas de cette possibilité** : la colonne virtuelle doit être créée **explicitement**, comme ci-dessus. L'amélioration de la 11.8 (reconnaissance de l'expression par l'optimiseur) réduit néanmoins l'écart d'usage : on crée toujours la colonne, mais on peut écrire la requête avec l'expression JSON naturelle. Ce point est à considérer lors d'une migration depuis MySQL (chapitre 19), où un index fonctionnel devra être converti en colonne virtuelle indexée.

## Points clés à retenir

Le JSON n'étant pas indexable directement et MariaDB n'offrant pas d'index fonctionnel, on indexe un champ JSON en **deux temps** : une **colonne virtuelle** qui l'extrait et le **type** (`CAST(JSON_VALUE(col, '$.chemin') AS …)`), puis un **index** sur cette colonne. Le typage et, pour les chaînes, la **collation** conditionnent la justesse des comparaisons. Depuis la **11.8**, l'optimiseur reconnaît l'expression d'extraction dans le `WHERE`, en plus du nom de la colonne. La variante **`VIRTUAL`** (indexable depuis 10.2) est généralement préférée à `PERSISTENT`. Enfin, l'absence d'**index fonctionnel** (présent dans MySQL) impose la colonne virtuelle explicite — un point d'attention en migration.

---

**Section précédente :** [4.9 — JSON Schema Validation](09-json-schema-validation.md)  
**Section suivante :** [4.11 — Expressions régulières (`REGEXP`, `REGEXP_REPLACE`, `REGEXP_SUBSTR`)](11-expressions-regulieres.md)  

⏭️ [Expressions régulières (REGEXP, REGEXP_REPLACE, REGEXP_SUBSTR)](/04-concepts-avances-sql/11-expressions-regulieres.md)

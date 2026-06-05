🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.7 Variables et contrôle de flux (`IF`, `CASE`, `LOOP`, `WHILE`, `REPEAT`)

Les exemples des sections précédentes ont employé, sans les détailler, des variables et des structures comme `IF`, `LOOP` ou `LEAVE`. Ce sont les éléments *procéduraux* du langage — ceux qui, par-dessus le SQL déclaratif, permettent de mémoriser un état et de diriger le déroulement d'un traitement. Cette section les rassemble : les variables locales d'abord, puis les structures de choix et de boucle.

## Les variables locales

Une variable locale se déclare en tête d'un bloc `BEGIN … END` avec `DECLARE`, en précisant son type et, éventuellement, une valeur initiale (`DEFAULT`) ; sans valeur par défaut, elle vaut `NULL`. Plusieurs variables d'un même type peuvent être déclarées ensemble.

```sql
DECLARE v_compteur INT DEFAULT 0;
DECLARE v_nom VARCHAR(100);
DECLARE v_prix, v_remise DECIMAL(10,2);
```

On lui affecte une valeur avec `SET`, ou avec `SELECT … INTO` pour récupérer le résultat d'une requête mono-ligne :

```sql
SET v_compteur = v_compteur + 1;
SELECT nom INTO v_nom FROM clients WHERE id = 1;
```

Une variable locale est limitée au bloc où elle est déclarée (et à ses blocs imbriqués), et sa déclaration doit précéder celle des curseurs et des gestionnaires (voir 8.1.1). Il ne faut pas la confondre avec une **variable de session**, préfixée par `@`, qui n'est pas déclarée, persiste pour toute la connexion et n'est pas typée. En mode Oracle, l'affectation s'écrit aussi avec l'opérateur `:=`.

## Choix : `IF` et `CASE`

L'instruction `IF` enchaîne des conditions ; le mot-clé de branche intermédiaire est `ELSEIF`, en un seul mot.

```sql
IF v_prix < 50 THEN
    SET v_remise = 0;
ELSEIF v_prix < 200 THEN
    SET v_remise = v_prix * 0.05;
ELSE
    SET v_remise = v_prix * 0.10;
END IF;
```

L'instruction `CASE` exprime plus lisiblement un choix multiple. Elle existe sous deux formes : *simple* (comparaison d'une valeur à des constantes) et *recherchée* (suite de conditions). Voici la forme recherchée :

```sql
CASE
    WHEN v_prix < 50  THEN SET v_categorie = 'Économique';
    WHEN v_prix < 200 THEN SET v_categorie = 'Standard';
    ELSE                   SET v_categorie = 'Premium';
END CASE;
```

À ne pas confondre avec l'*expression* `CASE` utilisée dans les requêtes (qui se termine par `END`) : il s'agit ici de l'*instruction* `CASE`, qui se termine par `END CASE` et exécute des instructions. Si aucune branche ne correspond et qu'il n'y a pas de `ELSE`, l'instruction `CASE` lève une erreur — là où l'expression, elle, renverrait `NULL`.

## Boucles : `LOOP`, `WHILE`, `REPEAT`

Trois boucles sont disponibles, qui diffèrent par le moment où leur condition d'arrêt est évaluée :

| Boucle | Test de la condition | Exécution minimale | Sortie |
|---|---|---|---|
| `LOOP` | aucun (inconditionnel) | infinie sans `LEAVE` | `LEAVE` obligatoire |
| `WHILE … DO` | avant chaque tour | 0 fois possible | quand la condition devient fausse |
| `REPEAT … UNTIL` | après chaque tour | au moins 1 fois | quand la condition `UNTIL` devient vraie |

`LOOP` est une boucle inconditionnelle : elle tourne indéfiniment jusqu'à ce qu'on en sorte explicitement avec `LEAVE`. On l'étiquette (ici `compteur:`) pour pouvoir la cibler, et `ITERATE` relance le tour suivant sans exécuter le reste du corps.

```sql
DECLARE i INT DEFAULT 0;

compteur: LOOP
    SET i = i + 1;
    IF i > 10 THEN
        LEAVE compteur;          -- sortie de la boucle
    END IF;
    IF MOD(i, 2) = 0 THEN
        ITERATE compteur;        -- passe au tour suivant (ignore les pairs)
    END IF;
    -- traitement des nombres impairs
END LOOP compteur;
```

`WHILE … DO` teste sa condition *avant* chaque tour : le corps peut donc ne jamais s'exécuter.

```sql
WHILE v_compteur < 10 DO
    SET v_compteur = v_compteur + 1;
END WHILE;
```

`REPEAT … UNTIL` teste sa condition *après* chaque tour : le corps s'exécute au moins une fois, et la boucle se répète jusqu'à ce que la condition `UNTIL` devienne vraie.

```sql
REPEAT
    SET v_compteur = v_compteur + 1;
UNTIL v_compteur >= 10
END REPEAT;
```

`LEAVE` et `ITERATE` s'appliquent aux trois boucles via leur étiquette. `LEAVE` peut d'ailleurs aussi quitter un bloc `BEGIN … END` étiqueté, offrant une sortie anticipée — l'équivalent, pour une procédure, du `RETURN` réservé aux fonctions (voir 8.2.1).

## Compléments : mode Oracle et boucle `FOR`

En mode Oracle, outre l'affectation par `:=`, l'écriture des structures de contrôle suit les conventions de PL/SQL. MariaDB propose par ailleurs une boucle `FOR`, commode pour itérer sans gérer manuellement un curseur. Elle parcourt soit un **intervalle d'entiers** (`FOR i IN 1..10 DO …`), soit directement les **lignes d'une requête** — condensant alors en une seule construction tout le cérémonial d'un curseur (8.5) :

```sql
-- parcourt chaque ligne du résultat, sans DECLARE/OPEN/FETCH/CLOSE ni gestionnaire NOT FOUND
FOR rec IN (SELECT id, nom FROM clients) DO
    -- traitement de rec.id, rec.nom …
END FOR;
```

Ce n'est qu'un complément : les cinq structures précédentes suffisent à la plupart des traitements.

---

Ces briques procédurales — variables, choix, boucles — complètent l'arsenal de la programmation côté serveur. La dernière section du chapitre prend de la hauteur avec les **[bonnes pratiques de programmation](08-bonnes-pratiques.md)**.

⏭️ [Bonnes pratiques de programmation](/08-programmation-cote-serveur/08-bonnes-pratiques.md)

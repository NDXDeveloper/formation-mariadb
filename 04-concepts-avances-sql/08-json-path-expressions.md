🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.8 JSON Path Expressions avancées

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Indice négatif / `last` / intervalle depuis 10.9 · Référence : **MariaDB 12.3 LTS**

Les fonctions JSON (§ 4.7.2) reposent toutes sur des **expressions de chemin** (*JSONPath*) pour désigner l'élément à atteindre — `'$.nom'`, `'$[0]'`, etc. Maîtriser cette syntaxe, et en particulier ses sélecteurs avancés (caractères génériques, recherche récursive, accès par la fin, intervalles), permet de cibler précisément n'importe quel élément d'un document, même profondément imbriqué.

## 🎯 Objectif de la section

Connaître l'anatomie d'un chemin JSON, maîtriser les sélecteurs de membres et d'éléments, exploiter le pas récursif `**` et les accès avancés aux tableaux, et connaître le mode et les limites de l'implémentation MariaDB.

## Anatomie d'un chemin

Un chemin JSON commence par le symbole **`$`**, qui représente le **point de départ** (le document courant), suivi de zéro, un ou plusieurs **pas** sélectionnant des éléments.

Le chemin peut être précédé d'un **mode** optionnel. MariaDB ne prend en charge que le mode **`lax`** (« souple »), qui est aussi celui appliqué par défaut : un chemin qui ne correspond à rien renvoie simplement `NULL` ou aucune ligne, sans lever d'erreur. Le mode `strict` n'existe pas en MariaDB.

## Sélecteurs de membres d'objet

Pour atteindre un membre d'un objet :

- **`.membre`** sélectionne la valeur du membre nommé `membre` ;
- **`."nom avec espace"`** fait de même pour un nom qui n'est pas un identifiant valide (espace, point, caractères spéciaux) ;
- **`.*`** sélectionne **toutes les valeurs** des membres de l'objet (caractère générique d'objet). Si l'élément courant est un tableau, rien n'est sélectionné.

```sql
SET @j = '{"nom":"Monty", "adresse":{"ville":"Helsinki", "pays":"Finlande"}}';
SELECT JSON_EXTRACT(@j, '$.nom');             -- "Monty"
SELECT JSON_EXTRACT(@j, '$.adresse.ville');    -- "Helsinki"
SELECT JSON_EXTRACT(@j, '$.adresse.*');        -- ["Helsinki", "Finlande"]
```

## Sélecteurs d'éléments de tableau

Pour atteindre les éléments d'un tableau (indexés à partir de **zéro**) :

- **`[N]`** sélectionne l'élément numéro N ;
- **`[*]`** sélectionne **tous** les éléments du tableau.

Depuis **MariaDB 10.9** (donc disponible en 12.3), trois accès supplémentaires facilitent le travail par la fin et par plages :

- **`[-N]`** : indice négatif, sélectionne le Nème élément **depuis la fin** (`[-1]` étant le dernier) ;
- **`[last]`** et **`[last-N]`** : le **dernier** élément, et le Nème avant le dernier ;
- **`[M to N]`** : un **intervalle** d'éléments, de l'indice M à N (mot-clé `to`).

```sql
SET @a = '[10, 20, 30, 40, 50]';
SELECT JSON_EXTRACT(@a, '$[0]');       -- 10
SELECT JSON_EXTRACT(@a, '$[*]');        -- [10, 20, 30, 40, 50]
SELECT JSON_EXTRACT(@a, '$[-1]');       -- 50  (dernier, indice négatif)
SELECT JSON_EXTRACT(@a, '$[last]');     -- 50
SELECT JSON_EXTRACT(@a, '$[last-1]');   -- 40  (avant-dernier)
SELECT JSON_EXTRACT(@a, '$[1 to 3]');   -- [20, 30, 40]  (intervalle)
```

> 💡 **Intervalle impossible.** Si l'intervalle est incohérent (M supérieur à N), il est traité comme vide et renvoie `NULL` : `JSON_EXTRACT(@a, '$[4 to 2]')` donne `NULL`. Avant la 10.9, l'accès par la fin imposait de connaître la longueur du tableau ; ces ajouts comblent ce manque.

## Le pas récursif `**` (recherche en profondeur)

Le pas générique **`**`** sélectionne **récursivement tous les éléments descendants** de l'élément courant — membres d'objets comme éléments de tableaux, à tous les niveaux. C'est l'outil pour retrouver une donnée **où qu'elle se trouve** dans le document, sans connaître sa position exacte.

```sql
SET @doc = '[{"nom":"Wag","type":"chien"}, {"nom":"Meow","type":"chat"}]';
SELECT JSON_EXTRACT(@doc, '$**.nom');   -- ["Wag", "Meow"]
```

Deux règles encadrent son usage : `**` **ne peut pas être le dernier pas** du chemin, et doit être **suivi d'un sélecteur** de membre ou d'élément. Ainsi, `$**.prix` sélectionne tous les membres `prix` du document, à n'importe quelle profondeur, tandis que `$**` seul est invalide. Ce pas est une extension non standard, de même signification qu'en MySQL.

## Plusieurs correspondances : l'auto-enveloppement

Lorsqu'un chemin peut désigner **plusieurs** éléments — par un caractère générique (`.*`, `[*]`, `**`), un intervalle (`[M to N]`), ou plusieurs chemins passés à la même fonction —, le résultat est **enveloppé dans un tableau JSON**. À l'inverse, un chemin unique sans générique ni intervalle renvoie une valeur simple. C'est pourquoi `$[*]` ou `$**.nom` produisent un tableau de résultats.

## Mode et limites

L'implémentation MariaDB des chemins JSON est **proche de celle de MySQL**, à une différence près : MariaDB autorise à préciser explicitement le mode (même si seul `lax` existe), là où MySQL ne le permet pas. Deux limites méritent d'être signalées :

- seul le **mode `lax`** est disponible (pas de mode `strict`) ;
- les **filtres** de la syntaxe JSONPath (expressions conditionnelles du type `?(@.couleur=="noir")`, arithmétique, fonctions) **ne sont pas pris en charge** — contrairement à PostgreSQL. Le filtrage se fait donc côté SQL, dans le `WHERE`, à partir des valeurs extraites.

## Récapitulatif des sélecteurs

| Sélecteur            | Cible                                   | Exemple        |
|----------------------|-----------------------------------------|----------------|
| `.membre`            | valeur d'un membre d'objet              | `$.nom`        |
| `."nom avec espace"` | membre au nom non standard              | `$."date achat"` |
| `.*`                 | toutes les valeurs d'un objet           | `$.adresse.*`  |
| `[N]`                | élément N d'un tableau (base 0)         | `$[0]`         |
| `[*]`                | tous les éléments d'un tableau          | `$[*]`         |
| `[-N]`               | Nème élément depuis la fin              | `$[-1]`        |
| `[last]` / `[last-N]`| dernier / Nème avant la fin             | `$[last]`      |
| `[M to N]`           | intervalle d'éléments                   | `$[1 to 3]`    |
| `**`                 | tous les descendants (récursif)         | `$**.prix`     |

## Points clés à retenir

Un chemin JSON débute par `$` et enchaîne des pas ; MariaDB n'applique que le mode **`lax`**. Les membres d'objet se ciblent par `.membre`, `."nom"` ou `.*` ; les éléments de tableau par `[N]` ou `[*]`, complétés depuis la 10.9 par l'**indice négatif `[-N]`**, le **mot-clé `last`** et l'**intervalle `[M to N]`** (un intervalle impossible renvoyant `NULL`). Le pas récursif **`**`** retrouve un élément à n'importe quelle profondeur, à condition de ne pas être le dernier pas. Tout chemin à correspondances multiples (générique, intervalle, chemins multiples) **enveloppe son résultat dans un tableau**. Enfin, MariaDB ne propose ni mode `strict` ni **filtres** JSONPath : le filtrage conditionnel se fait dans le `WHERE`.

---

**Section précédente :** [4.7.4 — Prédicat `IS JSON` et suppression de la limite de profondeur](07.4-is-json-predicate.md)  
**Section suivante :** [4.9 — JSON Schema Validation](09-json-schema-validation.md)  

⏭️ [JSON Schema Validation](/04-concepts-avances-sql/09-json-schema-validation.md)

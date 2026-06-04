🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.9 JSON Schema Validation

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> `JSON_SCHEMA_VALID` disponible depuis MariaDB 11.1 · Référence : **MariaDB 12.3 LTS**

Vérifier qu'un document est du **JSON bien formé** (§ 4.7.1) ne dit rien de sa **structure** : un objet vide `{}` est syntaxiquement valide, mais ne contient peut-être pas les champs attendus. La **validation par schéma** comble ce manque : elle vérifie qu'un document **conforme à une structure attendue** — champs obligatoires, types, plages de valeurs, motifs, etc. En MariaDB, c'est le rôle de la fonction `JSON_SCHEMA_VALID`, qui apporte une discipline structurelle à des données par nature souples.

## 🎯 Objectif de la section

Comprendre la place de la validation par schéma, savoir utiliser `JSON_SCHEMA_VALID`, écrire un schéma avec les mots-clés courants, imposer un schéma via une contrainte `CHECK`, et connaître les limites de l'implémentation MariaDB.

## Du « bien formé » au « conforme »

Trois niveaux de contrôle se complètent, du plus permissif au plus strict :

- **`JSON_VALID`** (§ 4.7.1, 4.7.2) : le document est-il du JSON syntaxiquement correct ?
- **`IS JSON`** (§ 4.7.4) : idem, avec en option une contrainte de **type** (objet, tableau, scalaire) et d'**unicité des clés** ;
- **`JSON_SCHEMA_VALID`** (cette section) : le document **respecte-t-il une structure complète** — quels champs, de quels types, dans quelles bornes ?

C'est ce dernier qui permet de garantir, par exemple, qu'un document contient bien un champ `nom` de type chaîne et un `age` numérique positif.

## `JSON_SCHEMA_VALID`

La fonction prend un **schéma** et un **document**, et renvoie `1` si le document est valide au regard du schéma, `0` sinon :

```text
JSON_SCHEMA_VALID(schema, document)
```

Reprenons l'exemple de la documentation : un schéma exigeant un `lastname` de type chaîne et un `age` numérique d'au moins 18.

```sql
SET @schema = '{
  "properties": {
    "lastname": { "type": "string" },
    "age":      { "type": "number", "minimum": 18 }
  },
  "required": ["lastname"]
}';

SELECT JSON_SCHEMA_VALID(@schema, '{"age":25, "lastname":"Miller"}');  -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"age":10, "lastname":"Miller"}');  -- 0 (age < 18)
SELECT JSON_SCHEMA_VALID(@schema, '{"age":25}');                       -- 0 (lastname manquant)
```

Le résultat est strictement **booléen** : en cas d'échec, la fonction ne dit **pas** quel mot-clé a échoué — elle renvoie seulement `0` (voir les limites plus bas).

## Écrire un schéma : les mots-clés courants

MariaDB suit le **JSON Schema Draft 2020**. Un schéma est lui-même un document JSON qui décrit la forme attendue. Les mots-clés les plus utiles, par catégorie :

| Catégorie    | Mots-clés                                                        |
|--------------|------------------------------------------------------------------|
| Structure    | `type`, `properties`, `required`, `additionalProperties`         |
| Nombres      | `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, `multipleOf` |
| Chaînes      | `minLength`, `maxLength`, `pattern`                              |
| Tableaux     | `items`, `prefixItems`, `minItems`, `maxItems`, `uniqueItems`    |
| Valeurs      | `enum`, `const`                                                  |
| Combinaisons | `anyOf`, `allOf`, `oneOf`, `not`                                 |

Un schéma plus riche pourrait par exemple s'écrire ainsi :

```sql
SET @schema = '{
  "type": "object",
  "properties": {
    "nom":    { "type": "string", "minLength": 1 },
    "age":    { "type": "number", "minimum": 0, "maximum": 130 },
    "statut": { "enum": ["actif", "inactif", "suspendu"] }
  },
  "required": ["nom", "age"]
}';
```

## Imposer un schéma : la contrainte `CHECK`

L'usage le plus puissant consiste à placer `JSON_SCHEMA_VALID` dans une contrainte `CHECK`, afin que **seuls les documents conformes** puissent être stockés dans une colonne :

```sql
CREATE TABLE personnes (
    id      INT PRIMARY KEY,
    donnees JSON,
    CONSTRAINT verifie_schema CHECK (
        JSON_SCHEMA_VALID(
            '{"type":"object",
              "properties":{"nom":{"type":"string"},
                            "age":{"type":"number","minimum":0}},
              "required":["nom"]}',
            donnees
        )
    )
);
```

Toute insertion ou mise à jour d'un document non conforme (sans `nom`, ou avec un `age` négatif) est alors rejetée. Cette contrainte de schéma **coexiste** avec la contrainte `JSON_VALID` ajoutée automatiquement (§ 4.7.1) : la première garantit la conformité structurelle, la seconde le simple fait d'être du JSON valide. On apporte ainsi à une colonne JSON une rigueur proche de celle d'un schéma relationnel, tout en conservant la souplesse du format.

## Limites en MariaDB

Le support du Draft 2020 comporte quelques **exceptions** importantes à connaître :

- les **ressources externes** ne sont pas prises en charge (pas de `$ref` vers des schémas distants) ;
- les mots-clés **hyper-schema** ne sont pas pris en charge ;
- les **`format`** (comme `date`, `email`, `uri`…) sont traités comme de **simples annotations**, et non comme des règles de validation. Autrement dit, déclarer `"format":"email"` ne vérifie **pas** que la valeur est un courriel valide :

```sql
-- "format":"email" n'est pas appliqué : la valeur passe quand même
SELECT JSON_SCHEMA_VALID('{"type":"string","format":"email"}', '"pas-un-email"');  -- 1
```

Pour contraindre réellement un tel format, on utilise un `pattern` (expression régulière), par exemple, plutôt que `format`.

Enfin, le **résultat est binaire** : `JSON_SCHEMA_VALID` ne fournit pas de rapport détaillé indiquant quel mot-clé a échoué (MariaDB ne propose pas d'équivalent à la fonction de rapport de MySQL). Diagnostiquer pourquoi un document est rejeté demande donc une inspection manuelle du schéma et du document.

## Points clés à retenir

`JSON_SCHEMA_VALID(schema, document)` (depuis MariaDB 11.1) valide un document contre un **JSON Schema Draft 2020** et renvoie un booléen. Elle complète `JSON_VALID` (bien formé) et `IS JSON` (type/unicité) en vérifiant une **structure complète** : champs requis, types, bornes, motifs, énumérations. Son emploi le plus utile est en **contrainte `CHECK`**, pour n'autoriser que des documents conformes dans une colonne JSON. Trois limites : pas de ressources externes ni de hyper-schema, les **`format` ne sont que des annotations** (utiliser `pattern` pour contraindre), et un **résultat purement binaire** sans détail sur la cause d'un échec.

---

**Section précédente :** [4.8 — JSON Path Expressions avancées](08-json-path-expressions.md)  
**Section suivante :** [4.10 — Indexation de colonnes virtuelles extraites du JSON](10-indexation-colonnes-virtuelles-json.md)  

⏭️ [Indexation de colonnes virtuelles extraites du JSON](/04-concepts-avances-sql/10-indexation-colonnes-virtuelles-json.md)

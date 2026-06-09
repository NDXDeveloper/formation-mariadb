🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6 — Index composites et ordre des colonnes

> **Chapitre 5 — Index et Performance** · Section 5.6  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Toutes les sections précédentes y renvoyaient : voici le **principe du préfixe gauche** (*left-most prefix*), développé en détail. Un index **composite** — un index portant sur plusieurs colonnes — est l'un des outils les plus puissants de l'indexation, car un seul index bien conçu peut servir le filtrage, la jointure, le tri et le regroupement à la fois. Mais toute sa valeur dépend d'une décision unique et déterminante : **l'ordre des colonnes**.

---

## Qu'est-ce qu'un index composite ?

Un index composite indexe **plusieurs colonnes ensemble**, dans un ordre donné :

```sql
CREATE INDEX idx_client_date ON commandes (client_id, date_commande);
```

Sa structure se comprend par analogie avec un **annuaire téléphonique** trié par *(nom, prénom)*. Les entrées sont d'abord classées par nom ; puis, pour un même nom, par prénom. De même, l'index ci-dessus trie d'abord par `client_id`, puis — pour un même `client_id` — par `date_commande` :

```
Index (client_id, date_commande), trié :
 client_id │ date_commande
    7      │ 2026-01-03
    7      │ 2026-02-11     ← à client_id égal, les dates sont triées entre elles
    7      │ 2026-05-20
   42      │ 2026-01-09
   42      │ 2026-03-15
   108     │ 2026-02-01
```

Cet ordre hiérarchique — colonne 1, puis colonne 2 à l'intérieur, etc. — est la clé de tout ce qui suit.

---

## Le principe du préfixe gauche

La règle fondamentale : un index composite ne peut être utilisé que pour les requêtes qui s'appuient sur un **préfixe gauche** de ses colonnes — c'est-à-dire les premières colonnes, dans l'ordre, sans en sauter.

Pour un index sur `(client_id, statut, date_commande)` :

```sql
CREATE INDEX idx_abc ON commandes (client_id, statut, date_commande);
```

| Requête (clause `WHERE`) | Préfixe | Index utilisable ? |
|--------------------------|---------|:------------------:|
| `client_id = 42` | (a) | ✅ |
| `client_id = 42 AND statut = 'expediee'` | (a, b) | ✅ |
| `client_id = 42 AND statut = 'expediee' AND date_commande = …` | (a, b, c) | ✅ |
| `statut = 'expediee'` | (b) seul | ❌ |
| `date_commande = …` | (c) seul | ❌ |
| `statut = 'expediee' AND date_commande = …` | (b, c) sans a | ❌ |

L'analogie de l'annuaire l'explique intuitivement : on peut y retrouver tous les « Dupont », ou « Dupont Marie » — mais **pas** tous les « Marie », car l'annuaire n'est pas trié par prénom. De même, l'index `idx_abc` est inutilisable pour une recherche portant sur `statut` ou `date_commande` **sans** `client_id`.

> **Conséquence directe :** l'ordre des colonnes doit refléter la façon dont les requêtes interrogent les données. Mettre en tête une colonne rarement filtrée seule rend l'index inutilisable pour de nombreux cas.

---

## Jusqu'où le préfixe est-il utilisable ? Le rôle des plages

Le préfixe « utilisable » ne s'arrête pas seulement quand une colonne manque : il s'arrête aussi à la **première colonne soumise à une plage**. La distinction est capitale :

- une **égalité** (`=`) sur une colonne **prolonge** l'usage du préfixe vers la colonne suivante ;
- une **plage** (`>`, `<`, `BETWEEN`, `LIKE 'x%'`) **interrompt** le préfixe : l'index sert jusqu'à cette colonne incluse, mais **pas au-delà**.

```sql
-- Index (client_id, date_commande)
WHERE client_id = 42 AND date_commande > '2026-01-01';
-- ✅ égalité puis plage : les DEUX colonnes sont exploitées

WHERE client_id > 10 AND date_commande = '2026-01-01';
-- ⚠️ plage sur client_id en tête : SEULE client_id est exploitée ;
--    date_commande = … ne peut plus restreindre le balayage
```

C'est la même mécanique que le piège « la plage casse l'ordre » vu pour le tri en [section 5.5.3](05.3-index-order-group.md) : une fois une plage rencontrée, les colonnes suivantes ne sont plus globalement ordonnées dans l'index.

---

## Comment ordonner les colonnes : la règle ESR

De ces principes découle une règle d'ordonnancement éprouvée, le **mnémonique ESR — Égalité, Tri (*Sort*), Plage (*Range*)** :

> **1. Égalité d'abord — 2. Tri ensuite — 3. Plage en dernier.**

Voici le raisonnement, sur une requête combinant les trois :

```sql
SELECT * FROM commandes
WHERE statut = 'expediee'        -- (E) égalité
  AND montant > 100              -- (R) plage
ORDER BY date_commande;          -- (S) tri
```

L'index recommandé est donc `(statut, date_commande, montant)` :

1. **Égalité (`statut`) en tête.** Les colonnes filtrées par égalité positionnent précisément le point de départ dans l'index, et toute colonne d'égalité prolonge le préfixe.
2. **Tri (`date_commande`) ensuite.** Une fois `statut` fixé, l'index restitue `date_commande` **dans l'ordre** : l'`ORDER BY` est satisfait sans *filesort*.
3. **Plage (`montant`) en dernier.** Placée à la fin, elle filtre les lignes restantes sans rompre l'ordre de tri. Si on la plaçait avant `date_commande`, on récupérerait certes une borne de balayage, mais on **perdrait l'ordre** et l'on retomberait sur un *filesort*.

ESR est une **heuristique** qui arbitre une tension : faut-il utiliser la plage pour borner le balayage, ou préserver l'ordre de l'index pour éviter le tri ? L'ordre ESR privilégie l'évitement du tri — souvent le plus coûteux, surtout avec un `LIMIT` qui permet alors de s'arrêter tôt.

Parmi plusieurs colonnes d'**égalité**, une heuristique secondaire consiste à placer la **plus sélective** en premier ; mais l'essentiel reste de coller aux **formes de requêtes réelles**, pas de classer mécaniquement par sélectivité.

---

## Un composite ou plusieurs index mono-colonne ?

Pour une requête filtrant sur plusieurs colonnes en `AND`, **un index composite est presque toujours préférable** à plusieurs index séparés :

```sql
-- Pour : WHERE client_id = 42 AND statut = 'expediee'
CREATE INDEX idx_client_statut ON commandes (client_id, statut);   -- ✅ un seul composite
```

Avec deux index distincts `(client_id)` et `(statut)`, l'optimiseur ne peut au mieux que les **combiner** (technique de *index merge*), généralement moins efficace qu'un composite bien conçu.

Le composite apporte aussi un bénéfice de **mutualisation** ([section 5.5](05-strategies-indexation.md)) : grâce au préfixe gauche, `idx_abc (client_id, statut, date_commande)` sert **également** les requêtes sur `(client_id)` seul et sur `(client_id, statut)`. En revanche, il ne sert **pas** une requête sur `statut` seul ou `date_commande` seul — pour celles-là, il faudrait un autre index.

---

## Éviter la redondance entre composites

Le préfixe gauche permet de repérer les index **redondants**. Si `idx_abc (client_id, statut, date_commande)` existe :

- un index `(client_id)` est **redondant** (c'est un préfixe de `idx_abc`) ;
- un index `(client_id, statut)` est **largement redondant** lui aussi.

On évite donc de créer ces index plus courts en présence du composite. La seule nuance : un index plus court est **plus petit**, donc parfois marginalement préféré par l'optimiseur pour une requête ne touchant que son préfixe — mais cela ne justifie qu'exceptionnellement de le conserver. La règle pratique : **peu d'index composites, bien ordonnés**, plutôt qu'une accumulation d'index qui se chevauchent.

---

## Lien avec les index couvrants

Un index composite qui contient **toutes** les colonnes dont une requête a besoin (celles du `SELECT` comprises) devient un **index couvrant** : MariaDB répond alors entièrement depuis l'index, sans accéder à la table — supprimant le « retour à la table » vu en [section 5.1](01-fonctionnement-index.md). C'est l'objet de la [section 5.9](09-index-covering.md).

---

## Combien de colonnes ?

Il n'y a pas de nombre « idéal », mais des bornes de bon sens : chaque colonne ajoutée alourdit l'index (taille, coût d'écriture). MariaDB impose par ailleurs des **limites techniques** (nombre de colonnes par index et longueur totale de la clé), rarement atteintes en pratique. Le bon réflexe n'est pas de viser large, mais de construire des index **focalisés** sur les requêtes à servir.

---

## Vérifier l'usage du préfixe

`EXPLAIN` ([section 5.7](07-analyse-plans-execution.md)) indique quel index est choisi, et surtout la colonne **`key_len`** : le nombre d'octets de l'index réellement utilisés. Cette valeur révèle **combien de colonnes** du composite sont effectivement exploitées — un `key_len` plus court qu'attendu signale qu'une partie du composite n'est pas utilisée (souvent à cause d'une plage en amont ou d'un préfixe rompu).

---

> ### 📝 À retenir  
>  
> - Un index composite trie les lignes par sa **première colonne**, puis par la deuxième à l'intérieur, etc. (analogie de l'annuaire `(nom, prénom)`).  
> - **Préfixe gauche** : l'index ne sert que les requêtes s'appuyant sur ses **premières colonnes dans l'ordre** ; `(a, b, c)` couvre `(a)`, `(a, b)`, `(a, b, c)` — mais ni `(b)`, ni `(c)`, ni `(b, c)`.  
> - Une **égalité** prolonge le préfixe ; une **plage** l'**interrompt** (les colonnes suivantes ne servent plus à borner le balayage).  
> - Ordonner les colonnes selon **ESR** : **Égalité**, puis **Tri** (pour éviter le *filesort*), puis **Plage** en dernier.  
> - Un **composite** vaut mieux que plusieurs index mono-colonne pour les filtres `AND` ; il **mutualise** via ses préfixes et rend les index plus courts **redondants**.  
> - Vérifier l'usage réel du préfixe avec **`EXPLAIN`** et la colonne **`key_len`**.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.5.3 Index pour ORDER BY et GROUP BY](05.3-index-order-group.md)
- ➡️ Section suivante : [5.7 Analyse des plans d'exécution (EXPLAIN et ANALYZE)](07-analyse-plans-execution.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Analyse des plans d'exécution (EXPLAIN et ANALYZE)](/05-index-et-performance/07-analyse-plans-execution.md)

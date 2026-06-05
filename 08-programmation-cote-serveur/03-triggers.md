🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3 Triggers (déclencheurs)

Un *trigger* (déclencheur) est une routine attachée à une table, que le serveur exécute **automatiquement** lorsqu'une modification de données — `INSERT`, `UPDATE` ou `DELETE` — y survient. Contrairement aux procédures et aux fonctions, on ne l'invoque jamais explicitement : il se déclenche de lui-même, en réaction à l'événement auquel il est associé.

C'est cette automaticité qui fait des triggers un outil puissant — pour garantir une règle métier, journaliser une modification ou maintenir une donnée dérivée — mais aussi un outil à manier avec discernement, précisément parce que leur logique s'exécute en coulisses.

## Un déclenchement automatique

La différence fondamentale avec les autres routines tient au mode d'invocation. Une procédure attend un `CALL`, une fonction est évaluée dans une expression ; un trigger, lui, est déclenché par le moteur au moment où une instruction modifie la table. Son code réagit donc à un événement plutôt qu'à un appel, ce qui le rend idéal pour imposer un comportement systématique : quelle que soit l'application ou la requête à l'origine de la modification, le trigger s'exécute.

## Les dimensions d'un trigger

Un trigger se définit par la combinaison de plusieurs dimensions, qui structurent les sous-sections à venir :

- son **moment** (timing) : `BEFORE` ou `AFTER` l'événement (8.3.1) ;
- son **événement** : `INSERT`, `UPDATE` ou `DELETE` (8.3.2) ;
- sa **granularité** : dans MariaDB, un trigger est toujours *au niveau ligne* (`FOR EACH ROW`) ; il s'exécute une fois par ligne affectée, et non une fois par instruction — il n'existe pas de trigger « au niveau instruction » ;
- l'accès aux **valeurs concernées** via les pseudo-enregistrements `OLD` et `NEW`, qui exposent la ligne avant et après la modification (8.3.3).

## À quoi servent les triggers ?

Les cas d'usage les plus fréquents tournent autour de l'automatisation d'actions liées aux données. L'**audit et la journalisation** en sont l'exemple type : un trigger `AFTER` peut consigner dans une table d'historique chaque modification, avec son horodatage et son auteur. Les triggers servent aussi à **faire respecter des règles d'intégrité** que les contraintes classiques n'expriment pas (validations conditionnelles, cohérence entre colonnes), à **maintenir des données dérivées** (mise à jour d'un total, d'un compteur ou d'un champ calculé à chaque changement), ou encore à **renseigner automatiquement des colonnes** comme une date de dernière modification.

## Intégration transactionnelle et précautions

Un trigger s'exécute dans la **même transaction** que l'instruction qui l'a déclenché. S'il échoue — erreur, contrainte violée, `SIGNAL` explicite (voir 8.6) — l'instruction tout entière est annulée : le trigger peut ainsi bloquer une modification invalide. Cette intégration est précieuse pour l'intégrité, mais elle a des contreparties.

D'abord, la logique d'un trigger est *invisible* depuis la requête : un développeur qui en ignore l'existence peut être surpris par des effets de bord (lignes insérées ailleurs, valeurs modifiées). Ensuite, un trigger s'exécute pour **chaque ligne affectée** : sur une instruction touchant des millions de lignes, un trigger coûteux dégrade fortement les performances. Enfin, les triggers compliquent le test, le versionnage et le débogage, et peuvent provoquer des effets en cascade difficiles à suivre. La règle de prudence est donc de les garder **simples, rapides et bien documentés**, et de réserver la logique lourde à des procédures appelées explicitement.

## Création, gestion et sécurité

Un trigger appartient à une base et est attaché à une table. On le crée avec `CREATE TRIGGER` et on le supprime avec `DROP TRIGGER` ; la syntaxe précise est introduite dès 8.3.1. Les triggers existants se consultent avec `SHOW TRIGGERS` ou via la vue `INFORMATION_SCHEMA.TRIGGERS`. Leur création requiert le privilège `TRIGGER` sur la table concernée et, comme les autres routines, un trigger possède un définisseur (`DEFINER`) qui détermine le contexte de privilèges de son exécution. Une même table peut porter plusieurs triggers — par exemple un `BEFORE INSERT` et un `AFTER UPDATE`.

## Nouveauté 12.x : le trigger multi-événements 🆕

Historiquement, chaque trigger ne couvrait qu'un seul événement : il fallait écrire des déclencheurs distincts pour `INSERT`, `UPDATE` et `DELETE`, même lorsqu'ils partageaient la même logique. MariaDB 12.x introduit les **triggers multi-événements** : un seul trigger peut désormais réagir à plusieurs événements à la fois, dans l'esprit de la syntaxe Oracle. Cette nouveauté, qui réduit la duplication, fait l'objet de la section 8.3.4.

## Plan de la section

- **8.3.1 — `BEFORE` et `AFTER`** : le choix du moment de déclenchement, avant ou après l'événement, et ses conséquences.
- **8.3.2 — Triggers `INSERT`, `UPDATE`, `DELETE`** : les trois événements déclencheurs et leurs spécificités.
- **8.3.3 — Variables `OLD` et `NEW`** : l'accès aux valeurs de la ligne avant et après la modification.
- **8.3.4 — Triggers multi-événements** 🆕 : un seul trigger pour plusieurs événements.

---

Commençons par la première dimension d'un trigger, le moment de son déclenchement : **[`BEFORE` et `AFTER`](03.1-before-after.md)**.

⏭️ [BEFORE et AFTER](/08-programmation-cote-serveur/03.1-before-after.md)

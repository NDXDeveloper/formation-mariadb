🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.3 — Audit de requêtes

> 🔎 Le SQL lui-même : lire les plans, repérer les anti-patterns, et n'utiliser les optimizer hints qu'à bon escient.

Après la configuration (E.1) et les index (E.2), cet audit examine les requêtes : ce qu'elles demandent au serveur, et comment le planificateur les exécute. Les outils sont `EXPLAIN` et le slow query log (Annexe C.2). Comme toujours : mesurer, ne changer qu'une chose, re-mesurer (voir la [présentation de l'annexe E](README.md)).

---

## Lire les plans d'exécution

- [ ] **`EXPLAIN` / `ANALYZE` lus sur les requêtes critiques et lentes** — repérées dans le slow query log (Annexe C.2) ; en MariaDB, c'est l'instruction `ANALYZE` (et non `EXPLAIN ANALYZE`, propre à MySQL) qui exécute la requête et annote le plan avec les lignes réellement traitées.
- [ ] **Pas de `type: ALL` injustifié** — un parcours complet de table sur une requête sélective signale un index manquant ou inutilisable.
- [ ] **Ni `Using filesort` ni `Using temporary` inattendus** sur les requêtes chaudes.
- [ ] **Lignes estimées ≈ lignes réelles** — un écart important trahit des statistiques périmées (lancer `ANALYZE TABLE`).
- [ ] **L'index retenu est celui attendu** — sinon, vérifier statistiques et conception avant d'envisager un hint.

## Repérer les anti-patterns SQL

- [ ] **Prédicats sargables** — pas de fonction sur la colonne indexée (`WHERE YEAR(d) = 2024` empêche l'usage de l'index sur `d` ; préférer une plage de dates), pas de joker en tête (`LIKE '%x'`), pas de conversion de type implicite (comparer une colonne texte à un nombre).
- [ ] **`SELECT *` évité** — ne récupérer que les colonnes utiles réduit les E/S et autorise les index couvrants.
- [ ] **Pas de N+1** — une jointure plutôt qu'une multitude de petites requêtes dans une boucle applicative.
- [ ] **`LIMIT` sur les gros résultats** — et pagination par `OFFSET` profond reconsidérée au profit d'une pagination par clé (*keyset*).
- [ ] **Sous-requêtes examinées** — les sous-requêtes corrélées qui gagneraient à devenir des jointures ; laisser l'optimiseur convertir quand c'est possible.
- [ ] **`DISTINCT` / `GROUP BY` justifiés** — ils imposent souvent une table temporaire ou un tri.

## Recours pertinent aux optimizer hints (nouveauté 12.x)

La série 12.x introduit des optimizer hints « nouvelle génération » : des commentaires `/*+ ... */` insérés dans la requête, de syntaxe compatible MySQL mais au comportement propre à l'optimiseur MariaDB. Contrairement à `optimizer_switch` (global), ils agissent sur une requête précise. C'est un outil puissant — donc à manier avec discernement.

- [ ] **Le hint reste un dernier recours ciblé** — on vérifie d'abord que les statistiques sont fraîches et l'indexation correcte ; un hint qui masque un problème de statistiques est un mauvais signe.
- [ ] **Chaque hint est documenté** — la raison de sa présence (le mauvais choix de l'optimiseur qu'il corrige) est notée, pour pouvoir le réévaluer.
- [ ] **Les hints sont revus après une mise à niveau ou une forte croissance des données** — un hint qui aidait hier peut nuire demain.
- [ ] **`MAX_EXECUTION_TIME` employé comme garde-fou** — plafonner en millisecondes les requêtes de lecture susceptibles de s'emballer.
- [ ] **Les familles de hints sont connues** — ordre de jointure (`JOIN_FIXED_ORDER`, `JOIN_ORDER`, `JOIN_PREFIX`, `JOIN_SUFFIX`), méthode d'accès (`INDEX`/`NO_INDEX`, `NO_ICP`, `MRR`, `BKA`, `BNL`), sous-requête (`SEMIJOIN`, `SUBQUERY`) ; les anciens `USE`/`FORCE`/`IGNORE INDEX` restent par ailleurs disponibles.

Exemple combinant un garde-fou de temps et un forçage d'ordre de jointure :

```sql
SELECT /*+ MAX_EXECUTION_TIME(5000) JOIN_ORDER(c, l) */ c.*
FROM   commandes c
JOIN   lignes    l ON l.commande_id = c.id
WHERE  c.statut = 'EXPEDIEE';
```

Ici, `MAX_EXECUTION_TIME(5000)` interrompt la requête au-delà de 5 secondes, et `JOIN_ORDER(c, l)` impose de joindre `commandes` avant `lignes`. *Le détail des hints figure au Ch. 15.15 et ses sous-sections.*

## Pour aller plus loin

L'analyse des requêtes lentes (slow query log, `pt-query-digest`) est traitée au [Ch. 15.7](../../15-performance-tuning/README.md), les optimizer hints au [Ch. 15.15](../../15-performance-tuning/README.md), et la lecture des plans au [Ch. 5.7](../../05-index-et-performance/README.md). Les requêtes de constat (requêtes lentes, requêtes coûteuses agrégées) figurent à l'[Annexe C.2](../c-requetes-sql-reference/02-requetes-monitoring.md), et l'entretien des statistiques au [Ch. 11.6](../../11-administration-configuration/README.md).

---

⬅️ [E.2 — Audit d'indexation](02-audit-indexation.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [E.4 — Audit de schéma](04-audit-schema.md)

⏭️ [Audit de schéma](/annexes/e-checklist-performance/04-audit-schema.md)

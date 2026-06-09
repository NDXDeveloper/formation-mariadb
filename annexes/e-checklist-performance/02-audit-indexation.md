🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.2 — Audit d'indexation

> 🗂️ L'audit au plus fort retour sur investissement : l'indexation est de loin la première cause de lenteur.

Cette checklist vérifie que les index sont présents là où ils servent, absents là où ils nuisent, et bien conçus. Elle s'appuie sur les requêtes de l'[Annexe C](../c-requetes-sql-reference/README.md) (cardinalité, index inutilisés) et sur la lecture des plans (`EXPLAIN`). Comme toujours : mesurer, ne changer qu'une chose, re-mesurer (voir la [présentation de l'annexe E](README.md)).

---

## Index manquants (le cas le plus fréquent)

- [ ] **Colonnes filtrées indexées** — les colonnes fréquemment utilisées dans les clauses `WHERE` disposent d'un index adapté (cf. Ch. 5.5.1).
- [ ] **Clés étrangères indexées** — toute colonne de clé étrangère porte un index, indispensable aux jointures et aux vérifications d'intégrité (cf. Ch. 5.5.2).
- [ ] **Tris et regroupements soutenus par un index** — les colonnes de `ORDER BY` et `GROUP BY` sont indexées, dans un ordre cohérent avec la requête (cf. Ch. 5.5.3).
- [ ] **Pas de parcours complet sur les requêtes chaudes** — `EXPLAIN` ne montre pas de `type: ALL` injustifié ; les tables accumulant beaucoup d'opérations sans index se repèrent via l'Annexe C.3 (`INDEX_NAME IS NULL`).
- [ ] **Requêtes fréquentes couvertes** — les requêtes les plus sollicitées (slow log, Annexe C.2) s'appuient sur des index pertinents.

## Index inutiles ou redondants

- [ ] **Index inutilisés repérés** — ceux qui ne sont jamais lus (Annexe C.3) ne coûtent que de la maintenance en écriture.
- [ ] **Index redondants supprimés** — un index sur `(a)` est redondant si un index sur `(a, b)` existe (préfixe le plus à gauche) ; idem pour les définitions en double.
- [ ] **Index peu sélectifs réexaminés** — un index non unique à faible cardinalité, ou à `avg_frequency` élevé (Annexe C.3), n'apporte souvent rien : une colonne à peu de valeurs distinctes profite rarement d'un B-Tree.
- [ ] **Suppression prudente** — avant de retirer un index, le vérifier avec `EXPLAIN`, puis le passer en **`IGNORED`** (MariaDB 10.6+) comme étape réversible avant suppression définitive (cf. Ch. 5.10).

## Conception des index composites

- [ ] **Ordre des colonnes respectant le préfixe gauche** — colonnes d'égalité d'abord, colonnes de plage et de tri ensuite ; les plus sélectives en tête (cf. Ch. 5.6).
- [ ] **Index composites au service de requêtes réelles** — ils répondent à des motifs d'accès effectifs, et non à des combinaisons spéculatives.
- [ ] **Index couvrants pour les requêtes de lecture chaudes** — ils incluent toutes les colonnes nécessaires, permettant un *index-only scan* (cf. Ch. 5.9).

## Qualité et entretien

- [ ] **Statistiques à jour** — `ANALYZE TABLE` exécuté après des changements de données importants ; statistiques EITS collectées si elles sont utilisées (Annexe C.3, Ch. 11.6).
- [ ] **Cardinalités plausibles** — les estimations d'`information_schema.STATISTICS` sont cohérentes avec le nombre réel de lignes (Annexe C.1 et C.3).
- [ ] **Types d'index appropriés** — B-Tree par défaut ; `FULLTEXT` pour la recherche textuelle, `SPATIAL` pour le géographique, `VECTOR`/HNSW pour la similarité (cf. Ch. 5.2-5.3).
- [ ] **Index très larges ou très volumineux justifiés** — leur coût en écriture et en disque est mis en regard du bénéfice en lecture.

## Vérification par les plans d'exécution

- [ ] **`EXPLAIN` / `ANALYZE` lus sur les requêtes critiques** — confirmer l'index retenu, le type d'accès, et l'écart entre lignes estimées et lignes réelles (en MariaDB, c'est l'instruction `ANALYZE` — et non `EXPLAIN ANALYZE`, propre à MySQL — qui exécute la requête et annote le plan des mesures réelles ; cf. Ch. 5.7).
- [ ] **Ni `Using filesort` ni `Using temporary` inattendus** sur les requêtes chaudes.
- [ ] **Optimisations 12.x prises en compte** — l'optimiseur de la série 12.x ajoute des gains sur les scans inversés (Index Condition Pushdown, Rowid Filtering, Loose Index Scan sur clés `DESC`) ; voir Ch. 5.8.1.

## Pour aller plus loin

Les stratégies d'indexation sont développées au [Ch. 5.5](../../05-index-et-performance/README.md), les index composites au [Ch. 5.6](../../05-index-et-performance/README.md), l'analyse des plans au [Ch. 5.7](../../05-index-et-performance/README.md), les index couvrants au [Ch. 5.9](../../05-index-et-performance/README.md) et les index invisibles/ignorés au [Ch. 5.10](../../05-index-et-performance/README.md). Les requêtes sur l'usage et la cardinalité des index figurent à l'[Annexe C.3](../c-requetes-sql-reference/03-requetes-analyse.md), et l'entretien des statistiques au [Ch. 11.6](../../11-administration-configuration/README.md).

---

⬅️ [E.1 — Audit de configuration](01-audit-configuration.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [E.3 — Audit de requêtes](03-audit-requetes.md)

⏭️ [Audit de requêtes (usage des optimizer hints)](/annexes/e-checklist-performance/03-audit-requetes.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 · `WITH CHECK OPTION`

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

La section précédente (§9.3) s'est achevée sur une question laissée en suspens : lorsqu'on écrit à travers une vue **filtrée** (dotée d'une clause `WHERE`), que se passe-t-il si l'opération produit une ligne qui *ne satisfait pas* cette clause ? La réponse par défaut surprend souvent — et c'est précisément le problème que résout `WITH CHECK OPTION`.

## Le problème : des lignes qui s'échappent de la vue

Considérons une vue modifiable qui n'expose que les employés d'un département donné :

```sql
CREATE VIEW v_dept_ventes AS
SELECT id, nom, prenom, salaire, dept_id
FROM employes
WHERE dept_id = 3;
```

Cette vue est pleinement modifiable (une seule table, colonnes simples, toutes les colonnes obligatoires présentes — cf. §9.3). Rien n'empêche pourtant d'y insérer une ligne… qui n'appartient pas au département 3 :

```sql
INSERT INTO v_dept_ventes (id, nom, prenom, salaire, dept_id)
VALUES (200, 'Petit', 'Luc', 40000, 5);
```

Par défaut, **cet `INSERT` réussit**. La ligne est bel et bien créée dans `employes`, mais avec `dept_id = 5` : elle ne satisfait donc pas la condition `dept_id = 3` de la vue et devient aussitôt **invisible** à travers `v_dept_ventes`. Elle s'est « échappée ». Le même phénomène se produit avec un `UPDATE` qui ferait basculer une ligne hors du filtre :

```sql
UPDATE v_dept_ventes SET dept_id = 5 WHERE id = 200;
```

Ce comportement est rarement souhaitable : une vue censée représenter « les employés du département 3 » ne devrait pas servir à créer ou déplacer des employés *ailleurs*.

## La solution : `WITH CHECK OPTION`

La clause **`WITH CHECK OPTION`**, ajoutée à la définition d'une vue modifiable, impose que toute ligne écrite à travers la vue **satisfasse sa clause `WHERE`**. Dans le cas contraire, l'opération est rejetée.

```sql
CREATE OR REPLACE VIEW v_dept_ventes AS
SELECT id, nom, prenom, salaire, dept_id
FROM employes
WHERE dept_id = 3
WITH CHECK OPTION;
```

Avec cette clause, les écritures hors périmètre échouent désormais explicitement :

```sql
-- Refusé : dept_id = 5 ne satisfait pas la condition de la vue
INSERT INTO v_dept_ventes (id, nom, prenom, salaire, dept_id)
VALUES (200, 'Petit', 'Luc', 40000, 5);
-- ERROR 1369 (44000): CHECK OPTION failed `ma_base`.`v_dept_ventes`

-- Accepté : dept_id = 3 respecte la condition
INSERT INTO v_dept_ventes (id, nom, prenom, salaire, dept_id)
VALUES (200, 'Petit', 'Luc', 40000, 3);

-- Refusé : déplacer l'employé hors du département 3 le ferait sortir de la vue
UPDATE v_dept_ventes SET dept_id = 5 WHERE id = 200;
-- ERROR 1369 (44000): CHECK OPTION failed `ma_base`.`v_dept_ventes`
```

L'erreur `1369` (`CHECK OPTION failed`) signale qu'une écriture aurait produit une ligne invisible à travers la vue. La vue devient ainsi une **frontière étanche** : ce qu'on y écrit reste visible dans la vue.

## Ce que la clause vérifie — et ce qu'elle ne vérifie pas

Quelques précisions importantes sur la portée de `WITH CHECK OPTION` :

- elle s'applique aux opérations **`INSERT` et `UPDATE`** réalisées **à travers la vue** ;
- elle ne concerne **pas le `DELETE`** : une suppression ne peut de toute façon porter que sur des lignes déjà visibles dans la vue, il n'y a donc rien à contrôler ;
- elle suppose une **vue modifiable** : tenter de poser `WITH CHECK OPTION` sur une vue non modifiable (agrégation, `GROUP BY`, etc. — cf. §9.3) provoque une erreur dès la création de la vue ;
- elle n'a d'effet que si la vue possède une **clause `WHERE`** : sans condition de filtrage, tout est trivialement « dans la vue », et la clause est sans portée pratique ;
- surtout, elle ne gouverne que les écritures **passant par la vue**. Une modification directe de la **table de base** échappe entièrement à ce contrôle : `WITH CHECK OPTION` n'est donc **pas** un substitut aux véritables contraintes d'intégrité (`CHECK`, `FOREIGN KEY`, `NOT NULL`…), qui, elles, s'imposent quel que soit le chemin d'écriture.

## Un piège : les colonnes filtrées non exposées

Une subtilité mérite l'attention lorsque la condition `WHERE` porte sur une colonne que la vue **n'expose pas**. Soit une vue des employés les plus anciens, filtrée sur `date_embauche` sans inclure cette colonne :

```sql
CREATE VIEW v_seniors AS
SELECT id, nom, prenom
FROM employes
WHERE date_embauche < '2015-01-01'
WITH CHECK OPTION;
```

Structurellement, cette vue est insertable (une table, colonnes simples, colonnes obligatoires `id`/`nom`/`prenom` présentes). Mais à l'insertion, on **ne peut pas renseigner `date_embauche`** puisque la vue ne l'expose pas : la colonne prend donc sa valeur par défaut (`NULL` ici). Or `NULL < '2015-01-01'` ne vaut pas *vrai* — le `CHECK OPTION` échoue, et **toute insertion est rejetée**. Autrement dit, une vue qui filtre sur une colonne qu'elle n'expose pas, assortie de `WITH CHECK OPTION`, peut devenir **impossible à alimenter**. C'est un comportement logique, mais déroutant si l'on n'y prend garde.

## `CASCADED` contre `LOCAL`

Lorsqu'une vue est construite **sur une autre vue**, `WITH CHECK OPTION` accepte un qualificatif qui détermine *jusqu'où* remonte le contrôle :

```sql
... WITH [CASCADED | LOCAL] CHECK OPTION
```

- **`CASCADED`** (valeur **par défaut** si l'on écrit simplement `WITH CHECK OPTION`) : la ligne écrite doit satisfaire la condition de la vue **et celles de toutes les vues sous-jacentes**, *même si ces dernières n'ont pas leur propre `CHECK OPTION`*.
- **`LOCAL`** : la ligne ne doit satisfaire que la condition **de la vue courante**. Les conditions des vues sous-jacentes ne sont vérifiées que si **elles-mêmes** portent une `CHECK OPTION`.

Illustrons avec deux niveaux. Une vue de base filtre sur un salaire minimal, **sans** `CHECK OPTION` :

```sql
CREATE VIEW v_salaire_min AS
SELECT id, nom, prenom, salaire, dept_id
FROM employes
WHERE salaire >= 30000;
```

Puis une vue bâtie dessus, qui ajoute un plafond. Comparons les deux variantes :

```sql
-- Variante CASCADED (équivalente à WITH CHECK OPTION tout court)
CREATE VIEW v_salaire_cascaded AS
SELECT id, nom, prenom, salaire, dept_id
FROM v_salaire_min
WHERE salaire <= 60000
WITH CASCADED CHECK OPTION;

-- Variante LOCAL
CREATE VIEW v_salaire_local AS
SELECT id, nom, prenom, salaire, dept_id
FROM v_salaire_min
WHERE salaire <= 60000
WITH LOCAL CHECK OPTION;
```

Le tableau suivant montre le sort d'un `INSERT` selon le salaire, pour chacune des deux variantes :

| Salaire inséré | `salaire >= 30000` (vue sous-jacente) | `salaire <= 60000` (vue externe) | `CASCADED` | `LOCAL` |
|----------------|:---:|:---:|------------|---------|
| 45 000 | ✔ | ✔ | Accepté | Accepté |
| 70 000 | ✔ | ✘ | Refusé | Refusé |
| 20 000 | ✘ | ✔ | Refusé | **Accepté** (ligne invisible !) |

La ligne décisive est la dernière. Avec **`CASCADED`**, insérer un salaire de 20 000 est refusé, car la condition de la vue sous-jacente (`>= 30000`) est elle aussi contrôlée. Avec **`LOCAL`**, la même insertion **réussit** — seule la condition locale (`<= 60000`) est vérifiée —, mais la ligne créée, avec un salaire de 20 000, ne réapparaît dans **aucune** des deux vues, puisqu'elle est filtrée par `v_salaire_min`. On retombe alors exactement sur le problème de la ligne « échappée » que la clause était censée éliminer.

C'est pourquoi **`CASCADED` est la valeur par défaut** et, dans la grande majorité des cas, le choix le plus sûr : `LOCAL` ne garantit l'étanchéité qu'au niveau de la vue elle-même, pas de l'empilement complet.

## Quand utiliser `WITH CHECK OPTION`

Cette clause prend tout son sens dès qu'une vue matérialise une **frontière logique** que les écritures ne doivent pas franchir :

- des vues de type « enregistrements actifs », « commandes de l'année en cours », « employés d'un service » — où l'on veut garantir que ce qui est inséré ou modifié **reste** dans la catégorie ;
- des vues de **cloisonnement par tenant** dans une architecture multi-tenant (§20.4) : `WITH CHECK OPTION` empêche un utilisateur d'insérer, via sa vue, des lignes appartenant à un autre tenant. Combinée au système de privilèges, la vue devient alors un mécanisme de **sécurité** à part entière — sujet de la section suivante.

En règle générale, on conservera le mode **`CASCADED`** (le défaut), et l'on ne recourra à `LOCAL` qu'avec une raison précise et en pleine connaissance de sa portée limitée.

## En résumé

`WITH CHECK OPTION` garantit qu'aucune ligne écrite à travers une vue filtrée ne puisse échapper à sa clause `WHERE` : les `INSERT` et `UPDATE` produisant une ligne hors périmètre sont rejetés (erreur `1369`). Elle ne s'applique qu'aux **vues modifiables**, ignore les `DELETE`, et ne contrôle que les écritures *passant par la vue* — ce n'est donc pas une contrainte d'intégrité au sens strict. Le qualificatif **`CASCADED`** (par défaut) étend le contrôle à toutes les vues sous-jacentes, là où **`LOCAL`** se limite à la vue courante, au risque de laisser passer des lignes invisibles.

En verrouillant ainsi le périmètre d'écriture d'une vue, `WITH CHECK OPTION` ouvre la voie à un usage plus large des vues comme outil de **sécurité et de confidentialité** — masquage de colonnes, restriction de lignes, contexte d'exécution —, qui fait l'objet de la **section 9.5 — Sécurité et vues : masquage de données**.

⏭️ [Sécurité et vues : Masquage de données](/09-vues-et-donnees-virtuelles/05-securite-et-vues.md)

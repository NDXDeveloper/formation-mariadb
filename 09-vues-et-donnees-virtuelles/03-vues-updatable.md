🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3 · Vues updatable : conditions et limitations

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

Une vue n'est pas qu'une fenêtre de lecture : sous certaines conditions, on peut **écrire à travers elle** — c'est-à-dire exécuter `INSERT`, `UPDATE` ou `DELETE` sur la vue, MariaDB répercutant alors l'opération sur la ou les tables sous-jacentes. On parle de **vue modifiable** (*updatable view*). Mais toutes les vues ne s'y prêtent pas, et celles qui s'y prêtent ne le font pas forcément pour les trois opérations. Cette section précise quand et comment.

## Le principe : une correspondance ligne à ligne

Une vue est modifiable lorsqu'il existe une **relation un-à-un** entre les lignes de la vue et les lignes de la table sous-jacente. Autrement dit, MariaDB doit être capable, pour chaque ligne visible dans la vue, de remonter sans ambiguïté à **une ligne précise** de la table de base sur laquelle appliquer la modification.

Dès que cette correspondance est rompue — parce que la vue agrège, regroupe, dédoublonne ou combine plusieurs lignes en une seule — la vue cesse d'être modifiable : il n'existe plus de cible unique vers laquelle propager l'écriture.

## Conditions générales : ce qui rend une vue non modifiable

Une vue **n'est pas modifiable** si sa requête de définition contient l'un des éléments suivants, qui tous brisent la correspondance ligne à ligne :

- une **fonction d'agrégation** (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`…) ;
- une clause **`GROUP BY`** ou **`HAVING`** ;
- le mot-clé **`DISTINCT`** ;
- une clause **`LIMIT`** ;
- un opérateur **`UNION`** ou **`UNION ALL`** ;
- une **fonction de fenêtrage** (*window function*) ;
- une **sous-requête dans la liste du `SELECT`** ;
- plusieurs **références à la même colonne** d'une table de base (par exemple `SELECT col, col`) ;
- une référence à une **autre vue non modifiable** dans la clause `FROM` ;
- l'absence de toute table sous-jacente (la vue ne renvoie que des **valeurs littérales**) ;
- l'algorithme **`TEMPTABLE`** (§9.6), qui matérialise le résultat et coupe donc le lien avec les lignes d'origine.

## Trois opérations, trois niveaux d'exigence

Le point souvent mal compris est que la modifiabilité **n'est pas tout ou rien** : `UPDATE`, `DELETE` et `INSERT` obéissent à des exigences **croissantes**. Une vue peut très bien accepter l'une et refuser l'autre.

| Opération | Conditions, en plus des conditions générales ci-dessus |
|-----------|--------------------------------------------------------|
| **`UPDATE`** | Les colonnes modifiées doivent être de **simples références** à des colonnes (pas des expressions), appartenant à **une seule** table. |
| **`DELETE`** | La vue doit référencer **exactement une** table de base. |
| **`INSERT`** | La vue doit référencer **une seule** table, n'avoir **aucun doublon** de nom de colonne, et l'insertion doit fournir **toutes les colonnes** `NOT NULL` sans valeur par défaut de la table de base. L'insertion ne peut viser que des colonnes **simples** : une colonne dérivée ne peut recevoir de valeur — mais, **depuis MariaDB 10.11.7** (MDEV-29587), sa seule présence n'empêche plus d'insérer dans les autres colonnes (auparavant, comme dans MySQL, elle rendait la vue entièrement non insertable). |

L'`INSERT` est donc le plus contraignant : il faut non seulement insérer dans des colonnes **simples** d'une seule table, mais aussi exposer **assez de colonnes** pour qu'une ligne complète et valide puisse être créée dans la table de base.

## Exemples

Les exemples reprennent le schéma `employes` / `departements` des sections précédentes. Pour mémoire, dans `employes` les colonnes `id`, `nom` et `prenom` sont `NOT NULL` sans valeur par défaut ; `salaire`, `date_embauche` et `dept_id` acceptent `NULL`.

### Une vue pleinement modifiable

Une simple projection de colonnes, sur une seule table, exposant toutes les colonnes obligatoires :

```sql
CREATE VIEW v_employes_contact AS
SELECT id, nom, prenom, dept_id
FROM employes;
```

Les trois opérations fonctionnent, car la correspondance ligne à ligne est parfaite et toutes les colonnes requises sont présentes :

```sql
UPDATE v_employes_contact SET prenom = 'Marie' WHERE id = 5;

INSERT INTO v_employes_contact (id, nom, prenom, dept_id)
VALUES (101, 'Durand', 'Paul', 2);

DELETE FROM v_employes_contact WHERE id = 101;
```

### Une vue non modifiable (agrégation)

Dès qu'on agrège, plus aucune écriture n'est possible :

```sql
CREATE VIEW v_effectifs AS
SELECT dept_id, COUNT(*) AS nb_employes
FROM employes
GROUP BY dept_id;
```

Toute tentative d'`UPDATE`, d'`INSERT` ou de `DELETE` sur `v_effectifs` échoue : une ligne de la vue résume *plusieurs* lignes de `employes`, il n'existe pas de cible unique.

### Une vue partiellement modifiable (colonne dérivée)

Le cas le plus instructif : une vue qui mêle colonnes simples et colonne calculée.

```sql
CREATE VIEW v_employes_calc AS
SELECT id, nom, prenom, salaire * 1.1 AS salaire_augmente
FROM employes;
```

Le comportement diverge selon l'opération :

- **`UPDATE` d'une colonne simple** — autorisé : `UPDATE v_employes_calc SET nom = 'Martin' WHERE id = 5;` met bien à jour `employes`.
- **`UPDATE` de la colonne dérivée** — refusé : `salaire_augmente` est une expression (`salaire * 1.1`), elle ne correspond à aucune colonne réelle ; MariaDB ne sait pas quelle valeur de `salaire` en déduire.
- **`DELETE`** — autorisé : la vue ne référence qu'une table et la correspondance ligne à ligne tient. `DELETE FROM v_employes_calc WHERE id = 5;` supprime la ligne sous-jacente.
- **`INSERT`** — autorisé **sur les colonnes simples** : `INSERT INTO v_employes_calc (id, nom, prenom) VALUES (…)` insère bien une ligne dans `employes`. Seule une insertion **visant la colonne dérivée** (`… (…, salaire_augmente) …`) échoue (*not insertable-into*). Depuis MariaDB **10.11.7** (MDEV-29587), la présence d'une colonne dérivée ne disqualifie plus la vue pour l'insertion : elle interdit seulement d'écrire *dans cette colonne* (comportement antérieur, encore celui de MySQL : la vue était entièrement non insertable).

Cette vue est donc *modifiable* (pour `UPDATE` sur ses colonnes simples), *deletable*, et *insertable sur ses colonnes simples* — une bonne illustration du fait que chaque opération, et même chaque colonne, est jugée séparément.

### Une vue sur jointure

Une vue joignant plusieurs tables peut rester modifiable, mais avec d'importantes restrictions :

```sql
CREATE VIEW v_emp_dept AS
SELECT e.id, e.nom, e.salaire, d.nom AS departement, d.ville
FROM employes e
JOIN departements d ON d.id = e.dept_id;
```

- **`UPDATE` ne touchant qu'une seule table** — autorisé. `UPDATE v_emp_dept SET salaire = 50000 WHERE id = 5;` modifie `employes`.
- **`UPDATE` touchant deux tables dans la même instruction** — refusé : une seule table sous-jacente peut être modifiée par instruction.
- **`DELETE`** — refusé : la suppression exige une **table unique**, ce qu'une vue de jointure ne satisfait pas.
- **`INSERT`** — possible uniquement si toutes les colonnes insérées appartiennent à **une seule** des tables ; en pratique, c'est restrictif et fragile, donc déconseillé.

> **Attention aux effets de bord.** Sur une vue de jointure, modifier une colonne du « côté » `departements` (par exemple `UPDATE v_emp_dept SET ville = 'Lyon' WHERE id = 5;`) met à jour la **ligne partagée** du département concerné, donc *tous les employés* de ce département, et pas seulement l'employé n° 5. C'est rarement l'effet recherché.

## Vérifier l'updatabilité d'une vue

La vue système `INFORMATION_SCHEMA.VIEWS` expose une colonne **`IS_UPDATABLE`** qui indique si une vue est modifiable :

```sql
SELECT TABLE_NAME, IS_UPDATABLE
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE();
```

Trois précautions à garder en tête. D'une part, `IS_UPDATABLE` renseigne sur la modifiabilité au sens général (`UPDATE`/`DELETE`) mais **ne distingue pas l'insertabilité** : il n'existe pas d'indicateur séparé pour `INSERT`, dont l'éligibilité n'est tranchée qu'au moment de l'opération. D'autre part, ce drapeau caractérise la vue *dans son ensemble* ; il ne dit pas quelles **colonnes** sont individuellement modifiables (rappelons qu'une colonne dérivée d'une vue par ailleurs modifiable ne l'est pas). Enfin, le drapeau peut **se tromper dans le sens permissif** : une vue bâtie sur une vue non modifiable (`SELECT … FROM v_agregat`) est parfois rapportée `IS_UPDATABLE = YES` alors que tout `UPDATE` à travers elle échoue (`ERROR 1288`). Calculé à partir de la seule forme de la vue, `IS_UPDATABLE` ne fait donc pas autorité — seules les règles énoncées plus haut, ou l'exécution réelle de l'opération, tranchent.

## Colonnes générées

Les **colonnes générées** d'une table (`VIRTUAL` ou `STORED`, §18.4) ne peuvent jamais recevoir de valeur, que ce soit directement ou via une vue : leur contenu est calculé par le moteur. Une vue ne permet donc pas de contourner cette règle — toute tentative d'écriture sur une colonne générée échoue.

## Interaction avec `WITH CHECK OPTION`

Écrire à travers une vue dotée d'une clause `WHERE` soulève une question : que se passe-t-il si l'on insère ou modifie une ligne de telle sorte qu'elle **sort du périmètre** de la vue (elle ne satisfait plus la condition `WHERE`) ? Par défaut, l'opération est acceptée, mais la ligne devient alors invisible à travers la vue. La clause **`WITH CHECK OPTION`** permet précisément d'interdire ce comportement. Elle fait l'objet de la **section 9.4**.

## Limitations et bonnes pratiques

En synthèse, quelques principes guident un usage sain des vues en écriture :

- privilégier, pour écrire, des vues **sur une seule table** et à **colonnes simples** : elles sont les seules à supporter les trois opérations sans surprise ;
- considérer les vues **multi-tables et les colonnes calculées** comme des vues de **lecture** ; les écritures à travers elles sont restreintes, parfois interdites, et porteuses d'effets de bord ;
- ne pas se fier au seul `IS_UPDATABLE` : raisonner explicitement sur l'opération (`INSERT` vs `UPDATE` vs `DELETE`) et sur les colonnes concernées ;
- documenter clairement, pour chaque vue destinée à l'écriture, les colonnes que l'on peut effectivement renseigner.

## En résumé

Une vue est modifiable lorsqu'elle maintient une **correspondance ligne à ligne** avec une table de base. Les trois opérations obéissent à des exigences croissantes : `UPDATE` (colonnes simples d'une table), `DELETE` (table unique), `INSERT` (table unique + colonnes simples + toutes les colonnes obligatoires présentes). Agrégations, `GROUP BY`, `DISTINCT`, `UNION`, fonctions de fenêtrage et algorithme `TEMPTABLE` rendent une vue non modifiable. `IS_UPDATABLE` aide au diagnostic, sans tout dire.

Reste un garde-fou indispensable lorsqu'on écrit à travers une vue filtrée : empêcher la création de lignes qui échapperaient ensuite à la vue. C'est l'objet de la **section 9.4 — `WITH CHECK OPTION`**.

⏭️ [WITH CHECK OPTION](/09-vues-et-donnees-virtuelles/04-with-check-option.md)

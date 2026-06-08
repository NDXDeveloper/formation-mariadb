🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.11 Online Schema Change (ALTER TABLE non-bloquant)

> **Version** : fonctionnalité standard d'InnoDB, élargie au fil des séries 10.x/11.x et pleinement disponible en 12.3 LTS.

Historiquement, modifier le schéma d'une table (`ALTER TABLE`) imposait souvent de la **reconstruire** en la verrouillant — bloquant lectures et écritures le temps de l'opération. Sur une grosse table de production, cela signifie une **indisponibilité** parfois longue. MariaDB propose des mécanismes d'*online DDL* qui permettent, pour de nombreuses opérations, de modifier le schéma **sans bloquer** l'activité, grâce à deux clauses de `ALTER TABLE` : `ALGORITHM` (comment l'opération est réalisée) et `LOCK` (ce qui reste permis pendant l'opération).

## La clause ALGORITHM : quatre niveaux d'impact

`ALGORITHM` indique *comment* MariaDB applique la modification. Du moins au plus coûteux :

| Algorithme | Ce qu'il fait | Coût |
|------------|---------------|------|
| `INSTANT` | modifie uniquement les **métadonnées**, sans toucher aux fichiers de données | quasi instantané |
| `NOCOPY` | opère **sur place**, sans recopier la table | faible |
| `INPLACE` | opère sur place mais peut **reconstruire** la table (réorganisation des données, reconstruction des index) | parfois élevé |
| `COPY` | **recopie intégralement** la table (ancien comportement) | élevé |

Ces niveaux sont **emboîtés** : `INSTANT` ne couvre qu'un sous-ensemble des opérations de `NOCOPY`, lui-même un sous-ensemble d'`INPLACE`, etc. La valeur par défaut, `DEFAULT`, laisse MariaDB choisir **l'algorithme le moins coûteux possible** pour l'opération demandée.

## La clause LOCK : ce qui reste permis pendant l'opération

`LOCK` contrôle la concurrence autorisée pendant le changement :

| Verrou | Concurrence autorisée |
|--------|------------------------|
| `NONE` | toute la DML — lectures **et** écritures |
| `SHARED` | lectures seules ; écritures bloquées |
| `EXCLUSIVE` | aucune ; lectures et écritures bloquées |

`LOCK=NONE` est la clé d'un changement véritablement « non bloquant » : l'application continue de lire **et** d'écrire pendant l'opération. `DEFAULT` choisit le verrou le moins restrictif compatible avec l'opération.

## Le principe essentiel : échouer vite plutôt que ralentir en silence

C'est le comportement le plus important à comprendre. Si l'on demande explicitement un algorithme (ou un verrou) que l'opération **ne prend pas en charge**, MariaDB **lève une erreur** au lieu de retomber silencieusement sur une méthode plus coûteuse :

```sql
ALTER TABLE commandes ADD PRIMARY KEY (id), ALGORITHM=INSTANT;
-- ERROR 1845 (0A000): ALGORITHM=INSTANT is not supported for this operation.
--   Try ALGORITHM=INPLACE
```

```sql
ALTER TABLE commandes ADD INDEX idx_client (client_id), ALGORITHM=INSTANT;
-- ERROR 1846 (0A000): ALGORITHM=INSTANT is not supported. Reason: ADD INDEX.
--   Try ALGORITHM=NOCOPY
```

Ce choix est délibéré : mieux vaut une erreur explicite — qui indique d'ailleurs l'algorithme à essayer — qu'une opération qui reconstruit la table et verrouille la production de manière inattendue. C'est ce qui rend l'online DDL **sûr en production** : écrire `ALTER TABLE … ALGORITHM=INPLACE, LOCK=NONE` garantit que l'opération s'exécutera en ligne *ou* échouera immédiatement, sans jamais bloquer la table par surprise.

Ce garde-fou s'obtient en exprimant l'exigence **directement dans le `ALTER TABLE`** (clauses `ALGORITHM=` et `LOCK=`). L'ancienne variable de session `alter_algorithm`, qui fournissait jadis cette valeur par défaut, est **dépréciée et ignorée depuis MariaDB 11.5** (MDEV-33655) : toute affectation déclenche désormais un avertissement (`The setting 'alter_algorithm' is ignored…`) et reste sans effet. Il faut donc préciser l'algorithme voulu dans chaque instruction, et non plus compter sur ce réglage global.

## Quelle opération avec quel algorithme ?

À titre indicatif, pour des tables InnoDB (le détail exact dépend de la version et de la structure de la table) :

| Opération | Algorithme minimal | Bloquant ? |
|-----------|--------------------|------------|
| Ajouter / supprimer une colonne | `INSTANT` | non |
| Renommer une colonne, changer sa valeur `DEFAULT` | `INSTANT` | non |
| Supprimer une clé étrangère | `INSTANT` | non |
| Ajouter ou supprimer un index secondaire | `NOCOPY` | non (`LOCK=NONE`) |
| Ajouter une clé étrangère | `NOCOPY` / `INPLACE` | non |
| Ajouter une clé primaire, réordonner les colonnes | `INPLACE` (reconstruction) | non, mais coûteux |
| **Changer le type d'une colonne** | `COPY` (reconstruction complète) | oui |

Règle pratique : tenter `ALGORITHM=INSTANT` d'abord ; en cas d'erreur, escalader vers `NOCOPY`, puis `INPLACE` ; conserver `LOCK=NONE` tant que l'opération l'autorise.

## Sous le capot et limites

Lorsqu'une opération `INPLACE` s'exécute avec `LOCK=NONE`, les modifications concurrentes (DML) sont enregistrées dans un **journal en ligne** puis rejouées sur la table reconstruite à la fin. De brefs verrous de métadonnées (exclusifs) sont tout de même posés en **début et en fin** d'opération, mais leur durée est négligeable.

Quelques limites à connaître :

- Une reconstruction `INPLACE` peut rester **lente** (elle réorganise les données) même si elle n'est pas bloquante : c'est précisément ce coût qu'`INSTANT` évite quand il s'applique.
- Une table comportant un index **FULLTEXT** ou **SPATIAL** ne peut pas être reconstruite avec `LOCK=NONE`.
- Les **colonnes générées** (§18.4) ne supportent pas l'online DDL pour toutes les opérations applicables aux colonnes ordinaires.
- Le changement de **type** de colonne reste une opération `COPY` bloquante.

## Quand l'online DDL natif ne suffit pas

Pour les opérations qui imposent encore une recopie bloquante, ou lorsqu'on souhaite un contrôle plus fin (régulation de charge, reprise…), on se tourne vers des outils de *schema change* externes — **gh-ost** et **pt-online-schema-change** — qui réalisent la modification par copie progressive et bascule, sans bloquer les écritures (§16.8.3). Côté réplication, l'**Optimistic ALTER TABLE** (§13.10) réduit en outre le retard induit par les DDL sur les réplicas.

## À retenir

- `ALTER TABLE` accepte deux clauses d'online DDL : **`ALGORITHM`** (INSTANT < NOCOPY < INPLACE < COPY) et **`LOCK`** (NONE / SHARED / EXCLUSIVE).
- `ALGORITHM=INSTANT` ne touche qu'aux **métadonnées** (ajout/suppression de colonne, etc.) ; `LOCK=NONE` autorise lectures **et** écritures concurrentes.
- MariaDB **échoue avec une erreur** (et suggère l'algorithme adéquat) plutôt que de retomber en silence sur une méthode coûteuse — d'où la sûreté en production.
- Le **changement de type** de colonne et quelques autres opérations restent en `COPY` (bloquant) ; FULLTEXT/SPATIAL et colonnes générées posent des limites.
- Pour le reste, **gh-ost** / **pt-online-schema-change** (§16.8.3) et l'**Optimistic ALTER TABLE** (§13.10) complètent la panoplie.

⏭️ [Contraintes FK : noms uniques par table seulement (plus par base)](/18-fonctionnalites-avancees/12-fk-constraint-names.md)

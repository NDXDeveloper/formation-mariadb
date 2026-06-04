🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Requêtes récursives (`WITH RECURSIVE`)

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Disponible depuis MariaDB 10.2.2 · Référence : **MariaDB 12.3 LTS**

Certaines données ne se prêtent pas à une requête « plate ». Un organigramme, une arborescence de catégories, une nomenclature de produits (un assemblage composé de sous-assemblages eux-mêmes composés de pièces) ou un réseau de dépendances forment des structures *hiérarchiques* ou *graphiques* dont la profondeur n'est pas connue à l'avance. Interroger ce type de données impliquait autrefois une boucle dans le code applicatif, une requête par niveau, ou une procédure stockée. Les **requêtes récursives** permettent d'exprimer ce parcours directement en SQL, dans une seule instruction déclarative.

Une requête récursive est une forme particulière de **CTE** (*Common Table Expression*, étudiée plus en détail en § 4.4). Là où une CTE classique nomme un résultat intermédiaire pour le réutiliser, une CTE récursive se référence **elle-même** afin de répéter un calcul jusqu'à épuisement des données.

## 🎯 Objectif de la section

Comprendre l'anatomie et le modèle d'exécution d'une requête récursive, savoir l'appliquer aux deux grands cas d'usage (génération de séries et parcours de hiérarchies), gérer les cycles en l'absence de clause dédiée, et connaître les garde-fous et limitations propres à MariaDB.

## Anatomie d'une CTE récursive

Une CTE récursive se compose toujours de deux parties, réunies par un opérateur ensembliste :

```sql
WITH RECURSIVE nom_cte AS (
    -- 1. Membre d'ancrage (non récursif) : le point de départ
    SELECT ...

    UNION ALL          -- ou UNION

    -- 2. Membre récursif : il référence nom_cte
    SELECT ...
    FROM nom_cte
    WHERE ...          -- condition d'arrêt
)
SELECT * FROM nom_cte;
```

Le **membre d'ancrage** (*anchor*) ne fait jamais référence à la CTE : il produit l'ensemble initial de lignes, la « graine » du calcul. Le **membre récursif** référence le nom de la CTE et s'appuie sur les lignes produites à l'itération précédente pour en générer de nouvelles. La condition placée dans le `WHERE` du membre récursif est essentielle : c'est elle qui finit par ne plus produire de lignes et qui met fin à la récursion.

Le mot-clé `RECURSIVE` est **obligatoire** dès qu'une CTE de la clause `WITH` est récursive. S'il y a plusieurs CTE et qu'au moins l'une d'elles est récursive, `RECURSIVE` se place une seule fois, juste après `WITH`.

## Le modèle d'exécution

Le moteur procède par itérations successives :

1. Il évalue le **membre d'ancrage**. Le résultat constitue la première version de la table de travail et alimente le résultat final.
2. Il évalue le **membre récursif** en utilisant comme source les lignes produites à l'étape précédente. Les nouvelles lignes obtenues sont ajoutées au résultat et deviennent la source de l'itération suivante.
3. L'étape 2 se répète tant qu'elle produit de nouvelles lignes.
4. Lorsqu'une itération ne renvoie plus aucune ligne, la récursion s'arrête et le résultat accumulé est transmis à la requête principale.

Avec `UNION ALL`, toutes les lignes sont conservées (comportement le plus courant et le plus performant). Avec `UNION`, les doublons stricts sont éliminés à chaque étape ; cela ralentit le calcul et ne suffit pas, à soi seul, à prévenir les cycles (voir plus loin).

## Premier exemple : générer une suite de nombres

Le cas le plus simple ne touche aucune table : il fabrique une série de valeurs.

```sql
WITH RECURSIVE compteur AS (
    SELECT 1 AS n                      -- ancre : on part de 1
    UNION ALL
    SELECT n + 1 FROM compteur         -- on incrémente
    WHERE n < 10                       -- jusqu'à 10
)
SELECT n FROM compteur;
```

Le déroulement se lit aisément :

| Itération | Lignes ajoutées | Table de travail |
|-----------|-----------------|------------------|
| Ancrage   | `1`             | `1`              |
| 1         | `2`             | `1, 2`           |
| 2         | `3`             | `1, 2, 3`        |
| …         | …               | …                |
| 9         | `10`            | `1 … 10`         |
| 10        | *(aucune)*      | arrêt            |

Ce motif est utile pour produire des bornes, des numéros de ligne synthétiques ou des grilles de référence.

## Deuxième exemple : générer une série de dates

Même principe, appliqué à des dates — pratique pour combler les jours manquants d'un rapport, par exemple en jointure externe avec une table de ventes.

```sql
WITH RECURSIVE calendrier AS (
    SELECT DATE '2026-01-01' AS jour
    UNION ALL
    SELECT jour + INTERVAL 1 DAY
    FROM calendrier
    WHERE jour < '2026-01-31'
)
SELECT jour FROM calendrier;
```

## Troisième exemple : parcourir une hiérarchie

C'est l'usage emblématique. Considérons une table d'employés en **liste d'adjacence** (chaque ligne pointe vers son responsable) :

```sql
CREATE TABLE employes (
    id          INT PRIMARY KEY,
    nom         VARCHAR(50),
    manager_id  INT,
    CONSTRAINT fk_manager FOREIGN KEY (manager_id) REFERENCES employes(id)
);
```

Peuplons-la d'un petit organigramme — Alice dirige Bruno et Chloé, et Bruno encadre David et Emma :

| id | nom | manager_id |
|----|-----|------------|
| 1 | Alice | *NULL* |
| 2 | Bruno | 1 |
| 3 | Chloé | 1 |
| 4 | David | 2 |
| 5 | Emma | 2 |

La requête suivante part du sommet de la hiérarchie (l'employé sans responsable) et descend de proche en proche, en calculant au passage le **niveau** de profondeur :

```sql
WITH RECURSIVE hierarchie AS (
    -- Ancre : la racine (pas de manager)
    SELECT id, nom, manager_id, 0 AS niveau
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    -- Membre récursif : les subordonnés directs des lignes déjà trouvées
    SELECT e.id, e.nom, e.manager_id, h.niveau + 1
    FROM employes AS e
    JOIN hierarchie AS h ON e.manager_id = h.id
)
SELECT id, nom, niveau
FROM hierarchie
ORDER BY niveau, nom;
```

| id | nom | niveau |
|----|-----|--------|
| 1 | Alice | 0 |
| 2 | Bruno | 1 |
| 3 | Chloé | 1 |
| 4 | David | 2 |
| 5 | Emma | 2 |

À chaque itération, la jointure entre la table `employes` et la CTE relie les employés à leurs responsables déjà présents dans le résultat : on découvre ainsi un niveau supplémentaire de l'organigramme. La colonne `niveau`, incrémentée à chaque étape, indique la profondeur.

Pour ne traiter que la sous-arborescence d'un responsable donné, il suffit d'adapter l'ancrage, par exemple `WHERE id = 42`.

## Construire un chemin (et un piège à connaître)

On souhaite souvent reconstituer le chemin complet depuis la racine (« fil d'Ariane »). On accumule alors une chaîne de caractères au fil de la descente :

```sql
WITH RECURSIVE chemin AS (
    SELECT id, nom, manager_id,
           CAST(nom AS CHAR(1000)) AS fil_ariane   -- voir l'avertissement
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.nom, e.manager_id,
           CONCAT(c.fil_ariane, ' > ', e.nom)
    FROM employes AS e
    JOIN chemin AS c ON e.manager_id = c.id
)
SELECT id, nom, fil_ariane
FROM chemin;
```

| id | nom | fil_ariane |
|----|-----|------------|
| 1 | Alice | Alice |
| 2 | Bruno | Alice > Bruno |
| 3 | Chloé | Alice > Chloé |
| 4 | David | Alice > Bruno > David |
| 5 | Emma | Alice > Bruno > Emma |

> ⚠️ **Avertissement — type des colonnes.** Dans une CTE récursive, le type et la **longueur** de chaque colonne sont déterminés par le **membre d'ancrage**. Si l'ancre renvoyait simplement `nom` (un `VARCHAR(50)`), les concaténations successives seraient **tronquées** à 50 caractères. D'où le `CAST(nom AS CHAR(1000))` dans l'ancrage : il réserve dès le départ une largeur suffisante pour le chemin complet.

## Gérer les cycles

Une hiérarchie d'employés est par nature acyclique, mais beaucoup de structures réelles (réseaux, dépendances, relations « ami de ») peuvent contenir des **cycles**. Sans précaution, le membre récursif tournerait indéfiniment — jusqu'à buter sur `max_recursive_iterations` (voir plus bas), qui renvoie alors un résultat *incomplet*. MariaDB offre deux moyens d'y remédier proprement.

### La clause `CYCLE … RESTRICT` (depuis 10.5.2)

Depuis MariaDB 10.5.2, une clause **`CYCLE`** peut être placée **après** la définition de la CTE : MariaDB cesse d'étendre un chemin dès qu'il reviendrait sur une combinaison de valeurs déjà rencontrée pour les colonnes indiquées.

```sql
WITH RECURSIVE parcours AS (
    SELECT id, nom, manager_id
    FROM employes
    WHERE id = 1

    UNION ALL

    SELECT e.id, e.nom, e.manager_id
    FROM employes AS e
    JOIN parcours AS p ON e.manager_id = p.id
)
CYCLE id RESTRICT          -- stoppe le parcours dès qu'un id réapparaît
SELECT id, nom FROM parcours;
```

> ℹ️ MariaDB n'en implémente qu'une forme **assouplie**, non conforme au standard : `CYCLE <colonnes> RESTRICT`, sans les sous-clauses `SET … TO … DEFAULT … USING` qu'exige le standard SQL. La clause `SEARCH` du standard (qui impose un ordre de parcours) n'est, elle, **pas** prise en charge.

### La parade manuelle (portable)

Avant la 10.5.2 — ou pour rester portable, ou encore lorsqu'on veut *conserver* le chemin parcouru —, on **mémorise les nœuds déjà visités** dans une colonne, puis on les exclut dans le membre récursif :

```sql
WITH RECURSIVE parcours AS (
    SELECT id, nom, manager_id,
           CAST(CONCAT(',', id, ',') AS CHAR(1000)) AS visites
    FROM employes
    WHERE id = 1

    UNION ALL

    SELECT e.id, e.nom, e.manager_id,
           CONCAT(p.visites, e.id, ',')
    FROM employes AS e
    JOIN parcours AS p ON e.manager_id = p.id
    WHERE p.visites NOT LIKE CONCAT('%,', e.id, ',%')  -- nœud non encore visité
)
SELECT id, nom FROM parcours;
```

La colonne `visites` accumule les identifiants rencontrés, encadrés de virgules ; le `NOT LIKE` empêche de réintégrer un nœud déjà parcouru, ce qui brise les cycles. Plus verbeuse que `CYCLE … RESTRICT`, cette technique a l'avantage d'être portable et d'exposer, en prime, le chemin suivi.

## Le garde-fou `max_recursive_iterations`

Pour éviter qu'une récursion mal conçue ne tourne sans fin, MariaDB plafonne le nombre d'itérations via la variable système `max_recursive_iterations`. Depuis MariaDB 10.6, sa valeur par défaut est **`1000`** (elle valait `4294967295`, soit « quasi illimité », jusqu'à la 10.5) : c'est donc, par défaut, une **limite réellement active**.

Point crucial : lorsque ce plafond est atteint, MariaDB **n'émet pas d'erreur** mais un simple **avertissement** — `Query execution was interrupted. The query exceeded max_recursive_iterations = 1000. The query result may be incomplete` — et renvoie le **résultat partiel** accumulé jusque-là. Une récursion qui bouclerait sans fin ne « plante » donc pas : elle renvoie silencieusement des données **incomplètes**. C'est une raison de plus de ne jamais compter sur ce garde-fou pour terminer une récursion — la condition d'arrêt (`WHERE`) et la gestion des cycles restent la seule façon correcte de la borner.

En pratique, 1000 niveaux suffisent très largement à la plupart des hiérarchies. Si une récursion *légitime* doit aller plus loin (génération d'une longue série, par exemple), on relève la limite pour la session :

```sql
SET max_recursive_iterations = 100000;
```

À l'inverse, pendant la mise au point, on peut l'abaisser (par exemple à `100`) pour repérer plus vite une boucle involontaire.

## Limitations du membre récursif

Le membre récursif obéit à des restrictions, héritées du standard SQL. Les principales :

- il ne peut référencer la CTE récursive **qu'une seule fois** ;
- il n'admet pas de fonctions d'agrégation (`SUM`, `COUNT`…), ni `GROUP BY` / `HAVING` ;
- il n'admet pas `DISTINCT` (en combinaison avec `UNION`) ;
- la table récursive ne peut pas figurer du côté « optionnel » d'un `LEFT`/`RIGHT JOIN`, ni à l'intérieur d'une sous-requête qui la référencerait.

Le membre d'**ancrage**, lui, ne subit pas ces contraintes : il peut agréger, joindre et filtrer librement, puisqu'il ne participe pas à la boucle.

## Performance et modélisation

La récursion repose ici sur le modèle de **liste d'adjacence** (`manager_id`). Pour qu'elle soit efficace, la colonne servant à la jointure récursive doit être **indexée** : sans index sur `manager_id`, chaque itération provoquerait un balayage complet de la table.

Quelques repères pratiques :

- Les résultats intermédiaires sont matérialisés ; sur des arbres très larges, surveillez la taille de la table de travail.
- Privilégiez `UNION ALL` à `UNION` lorsque les doublons sont impossibles : la déduplication a un coût.
- Pour des hiérarchies **lues très fréquemment** mais peu modifiées, d'autres modèles peuvent surpasser la récursion à la lecture : *closure table* (table de fermeture), *nested set* ou chemin matérialisé. Le choix relève d'un arbitrage entre coût d'écriture et coût de lecture, abordé plus loin dans la formation.

> 💡 **Note de migration (Oracle).** MariaDB n'implémente pas la syntaxe propriétaire `CONNECT BY ... PRIOR` d'Oracle. La requête récursive `WITH RECURSIVE`, conforme au standard SQL et portable, en est l'équivalent direct lors d'une migration. Les autres aspects de compatibilité Oracle sont traités en § 19.2.1.

## Points clés à retenir

Une requête récursive associe un **membre d'ancrage** (le point de départ) et un **membre récursif** (qui se référence lui-même), réunis par `UNION ALL` ou `UNION`, le mot-clé `RECURSIVE` étant obligatoire. Le moteur itère jusqu'à ce qu'aucune nouvelle ligne ne soit produite. Deux familles d'usage dominent : la **génération de séries** (nombres, dates) et le **parcours de hiérarchies ou de graphes**. Les cycles se gèrent soit par la clause **`CYCLE … RESTRICT`** (depuis 10.5.2), soit **manuellement** par une colonne de suivi des nœuds visités (portable) ; la clause `SEARCH` du standard, en revanche, n'existe pas dans MariaDB. Enfin, pensez à élargir le type des colonnes accumulées dès l'ancrage, à indexer la colonne de jointure récursive, et rappelez-vous que `max_recursive_iterations` (1000 par défaut depuis la 10.6) interrompt une récursion trop longue par un simple **avertissement**, en renvoyant un résultat incomplet.

---

**Section précédente :** [4. Concepts Avancés SQL](README.md)  
**Section suivante :** [4.2 — Window Functions](02-window-functions.md)  

⏭️ [Window Functions](/04-concepts-avances-sql/02-window-functions.md)

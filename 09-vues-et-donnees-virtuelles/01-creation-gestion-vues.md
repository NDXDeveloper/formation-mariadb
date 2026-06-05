🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 · Création et gestion des vues (`CREATE VIEW`, `ALTER VIEW`)

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

Comme l'a introduit ce chapitre, une vue est une **requête `SELECT` nommée et enregistrée** dans le serveur, qui se manipule comme une table virtuelle. Cette première section couvre l'ensemble du **cycle de vie** d'une vue : la créer, en consulter la définition, la modifier et la supprimer. Les aspects plus spécialisés — algorithme d'exécution (§9.6), sécurité et contexte d'exécution (§9.5), contrôle des écritures (§9.4) et conditions de modifiabilité (§9.3) — sont abordés dans les sections suivantes ; ils ne sont évoqués ici que dans la mesure où ils apparaissent dans la syntaxe.

## Schéma d'exemple

Les exemples de cette section s'appuient sur le schéma minimal suivant, représentant des employés rattachés à des départements :

```sql
CREATE TABLE departements (
    id      INT PRIMARY KEY,
    nom     VARCHAR(50) NOT NULL,
    ville   VARCHAR(50)
);

CREATE TABLE employes (
    id            INT PRIMARY KEY,
    nom           VARCHAR(50) NOT NULL,
    prenom        VARCHAR(50) NOT NULL,
    salaire       DECIMAL(10,2),
    date_embauche DATE,
    dept_id       INT,
    FOREIGN KEY (dept_id) REFERENCES departements(id)
);
```

## Créer une vue avec `CREATE VIEW`

Dans sa forme la plus simple, une vue se déclare en associant un nom à une requête `SELECT` :

```sql
CREATE VIEW v_employes_resume AS
SELECT id, nom, prenom, date_embauche
FROM employes;
```

La vue se requête ensuite exactement comme une table :

```sql
SELECT * FROM v_employes_resume
WHERE date_embauche >= '2024-01-01';
```

Aucune donnée n'est dupliquée : à chaque interrogation, MariaDB exécute la requête sous-jacente sur la table `employes`. La vue reflète donc toujours l'état courant des données.

> **Convention de nommage.** Préfixer les vues (par exemple `v_` ou `vw_`) est une pratique courante qui permet de les distinguer immédiatement des tables de base dans le schéma. Ce n'est pas une obligation du langage, mais cela facilite grandement la lecture et la maintenance.

### Syntaxe complète

La forme complète de `CREATE VIEW` accepte plusieurs clauses optionnelles :

```sql
CREATE
    [OR REPLACE]
    [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    [DEFINER = { utilisateur | CURRENT_USER | rôle | CURRENT_ROLE }]
    [SQL SECURITY { DEFINER | INVOKER }]
    VIEW [IF NOT EXISTS] nom_vue [(liste_colonnes)]
    AS instruction_select
    [WITH [CASCADED | LOCAL] CHECK OPTION]
```

Le rôle de chaque clause :

- **`OR REPLACE`** : remplace la vue si elle existe déjà (voir plus bas).
- **`ALGORITHM`** : impose la stratégie d'exécution (`MERGE`, `TEMPTABLE` ou `UNDEFINED`, valeur par défaut où le serveur choisit). Ce point est déterminant pour les performances et fait l'objet de la **section 9.6**.
- **`DEFINER`** et **`SQL SECURITY`** : déterminent au nom de quel utilisateur la vue s'exécute et avec quels privilèges. Détaillé dans la **section 9.5**.
- **`IF NOT EXISTS`** : ne crée la vue que si elle n'existe pas, sans erreur ni remplacement.
- **`liste_colonnes`** : noms explicites des colonnes de la vue (voir ci-dessous).
- **`WITH ... CHECK OPTION`** : contrôle l'intégrité des lignes écrites à travers la vue. Couvert par la **section 9.4**.

## Nommer les colonnes d'une vue

Par défaut, les noms des colonnes de la vue sont **hérités** de la requête `SELECT`. Lorsque le `SELECT` renvoie une expression, un appel de fonction ou un calcul, il est nécessaire de fournir un nom — soit par un alias dans la requête, soit via la liste de colonnes après le nom de la vue.

Les deux écritures suivantes produisent une vue identique. La première utilise des alias :

```sql
CREATE VIEW v_salaires_annuels AS
SELECT
    id,
    CONCAT(prenom, ' ', nom) AS employe,
    salaire * 12              AS salaire_annuel
FROM employes;
```

La seconde déclare les noms dans la liste de colonnes de la vue :

```sql
CREATE VIEW v_salaires_annuels (id, employe, salaire_annuel) AS
SELECT id, CONCAT(prenom, ' ', nom), salaire * 12
FROM employes;
```

Un nom explicite (par l'une ou l'autre méthode) est **obligatoire** dans trois cas : lorsque la colonne provient d'une expression ou d'une fonction, lorsque deux colonnes issues de tables différentes porteraient le même nom (par exemple deux colonnes `id` dans une jointure), et plus généralement chaque fois qu'on souhaite exposer un nom métier plus clair. Si une liste de colonnes est fournie, **son nombre d'éléments doit correspondre exactement** au nombre de colonnes renvoyées par le `SELECT`, faute de quoi la création échoue.

## `CREATE OR REPLACE` et `IF NOT EXISTS`

Deux clauses gèrent le cas où une vue du même nom existe déjà, mais avec des comportements opposés :

- **`CREATE OR REPLACE VIEW`** redéfinit la vue si elle existe, ou la crée sinon. C'est l'écriture idéale pour des scripts **idempotents** (migrations, déploiements) que l'on peut rejouer sans erreur.
- **`CREATE VIEW IF NOT EXISTS`** crée la vue uniquement si elle n'existe pas ; si elle existe, l'instruction est ignorée silencieusement et la définition en place est **conservée intacte**.

Ces deux clauses sont **mutuellement exclusives** : on ne peut pas combiner `OR REPLACE` et `IF NOT EXISTS` dans la même instruction.

```sql
-- Redéfinit systématiquement la vue (recommandé pour les migrations)
CREATE OR REPLACE VIEW v_employes_resume AS
SELECT id, nom, prenom, date_embauche, dept_id
FROM employes;
```

## Consulter les vues et leur définition

Les vues partagent le même espace de noms que les tables : on ne peut pas avoir une table et une vue portant le même nom dans la même base. Pour lister uniquement les vues d'une base, on filtre sur le type d'objet :

```sql
SHOW FULL TABLES WHERE Table_type = 'VIEW';
```

Pour afficher la définition exacte d'une vue, `SHOW CREATE VIEW` reconstitue l'instruction `CREATE` (l'option `\G` de la console `mariadb` donne un affichage vertical, plus lisible) :

```sql
SHOW CREATE VIEW v_salaires_annuels\G
```

Sur la vue `v_salaires_annuels` créée plus haut, la sortie révèle la **forme canonique** réellement enregistrée par le serveur (valeur `Create View` reformatée ici pour la lisibilité) :

```text
*************************** 1. row ***************************
                View: v_salaires_annuels
         Create View: CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost`
                      SQL SECURITY DEFINER VIEW `v_salaires_annuels` AS
                      select `employes`.`id` AS `id`,
                             concat(`employes`.`prenom`,' ',`employes`.`nom`) AS `employe`,
                             `employes`.`salaire` * 12 AS `salaire_annuel`
                      from `employes`
character_set_client: utf8mb4
collation_connection: utf8mb4_uca1400_ai_ci
```

Bien qu'on n'ait saisi qu'un simple `CREATE VIEW … AS SELECT …`, MariaDB a **ajouté** les clauses implicites (`ALGORITHM`, `DEFINER`, `SQL SECURITY` — détaillées en §9.5) et **qualifié** chaque colonne par sa table. C'est par cette même réécriture que le `SELECT *` se retrouve figé (voir plus bas) : l'étoile est remplacée, une fois pour toutes, par la liste explicite des colonnes présentes à la création.

Enfin, la vue système `INFORMATION_SCHEMA.VIEWS` permet d'interroger les métadonnées de toutes les vues de manière programmatique — y compris l'algorithme retenu et le caractère modifiable de la vue (`IS_UPDATABLE`, voir §9.3) :

```sql
SELECT TABLE_NAME, IS_UPDATABLE, ALGORITHM, VIEW_DEFINITION
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE();
```

> **À noter.** Le serveur ne conserve pas le texte original de la requête : il en stocke une forme **canonique réécrite** (noms d'objets et de colonnes qualifiés, `*` développé, commentaires et mise en forme supprimés). La définition restituée par `SHOW CREATE VIEW` peut donc différer visuellement de ce que vous aviez saisi, tout en étant sémantiquement équivalente.

## Modifier une vue : `ALTER VIEW`

`ALTER VIEW` redéfinit une vue existante. Sa syntaxe reprend celle de `CREATE VIEW`, sans la clause `OR REPLACE` :

```sql
ALTER
    [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    [DEFINER = { utilisateur | CURRENT_USER | rôle | CURRENT_ROLE }]
    [SQL SECURITY { DEFINER | INVOKER }]
    VIEW nom_vue [(liste_colonnes)]
    AS instruction_select
    [WITH [CASCADED | LOCAL] CHECK OPTION]
```

Exemple — restreindre la vue aux employés effectivement embauchés :

```sql
ALTER VIEW v_employes_resume AS
SELECT id, nom, prenom, date_embauche, dept_id
FROM employes
WHERE date_embauche IS NOT NULL;
```

Il faut bien comprendre qu'`ALTER VIEW` **ne modifie pas partiellement** une vue : on fournit l'intégralité de la nouvelle définition, qui se substitue à l'ancienne. La différence avec `CREATE OR REPLACE VIEW` est donc surtout sémantique : `ALTER VIEW` **exige que la vue existe** (et échoue sinon), tandis que `CREATE OR REPLACE` la crée si elle est absente. Dans des scripts automatisés, `CREATE OR REPLACE` est souvent préféré pour sa robustesse.

## Supprimer une vue : `DROP VIEW`

`DROP VIEW` retire une ou plusieurs vues. La clause `IF EXISTS` évite une erreur si une vue n'existe pas (elle produit alors un simple avertissement) :

```sql
DROP VIEW v_salaires_annuels;

DROP VIEW IF EXISTS v_obsolete, v_temporaire;
```

Supprimer une vue n'affecte **ni les tables sous-jacentes ni leurs données** : seule la définition de la vue disparaît. Les mots-clés `RESTRICT` et `CASCADE`, prévus par le standard SQL, sont acceptés par l'analyseur syntaxique mais **sans effet** dans MariaDB.

## Le piège du `SELECT *` : une définition figée

Un point important et souvent négligé concerne l'usage de `SELECT *` dans une vue. Au moment de la création, l'étoile est **développée et figée** : MariaDB enregistre la liste explicite des colonnes existant à cet instant. Par conséquent, si vous ajoutez ultérieurement une colonne à la table de base, **elle n'apparaîtra pas** dans la vue.

```sql
CREATE VIEW v_tous_employes AS
SELECT * FROM employes;          -- développé en (id, nom, prenom, salaire, …)

ALTER TABLE employes ADD COLUMN email VARCHAR(100);

SELECT * FROM v_tous_employes;   -- la colonne « email » est absente de la vue
```

Pour intégrer la nouvelle colonne, il faut redéfinir la vue (`CREATE OR REPLACE` ou `ALTER VIEW`). En pratique, mieux vaut **lister explicitement les colonnes** dans la définition d'une vue plutôt que de recourir à `*` : on évite ainsi les surprises et l'on documente précisément l'interface exposée.

## Vues construites sur d'autres vues

Une vue peut interroger une autre vue, ce qui permet de composer des couches d'abstraction successives :

```sql
CREATE VIEW v_employes_paris AS
SELECT e.id, e.nom, e.prenom
FROM v_employes_resume AS e
JOIN departements AS d ON d.id = e.dept_id
WHERE d.ville = 'Paris';
```

Cette possibilité est utile, mais à manier avec mesure : des chaînes de vues trop profondes compliquent la maintenance, rendent les diagnostics de performance plus difficiles et peuvent contraindre l'algorithme d'exécution (voir §9.6).

## Restrictions principales

La requête qui définit une vue est soumise à plusieurs contraintes qu'il faut connaître :

- elle ne peut référencer ni **variable système** ni **variable utilisateur**, ni **paramètre** de requête préparée ;
- elle ne peut pas faire référence à une **table temporaire**, et l'on ne peut pas créer de **vue temporaire** ;
- toutes les tables et vues citées dans la définition **doivent exister** au moment de la création ;
- on ne peut **pas associer de déclencheur** à une vue ;
- un `ORDER BY` présent dans la définition de la vue est **ignoré** si la requête qui interroge la vue possède son propre `ORDER BY` ;
- les fonctions sont évaluées **au moment de l'interrogation** de la vue, et non à sa création : une vue contenant `NOW()`, par exemple, renvoie l'heure de chaque requête.

## Privilèges requis

La création d'une vue nécessite le privilège **`CREATE VIEW`** sur la base concernée, ainsi que le privilège **`SELECT`** sur chacune des colonnes référencées par la requête. La modification via `ALTER VIEW` requiert en outre le privilège `DROP`. La gestion fine des droits et le contexte d'exécution (`DEFINER` / `INVOKER`) sont traités dans la **section 9.5**, et le système de privilèges dans son ensemble fait l'objet du **chapitre 10**.

## En résumé

`CREATE VIEW` enregistre une requête sous forme de table virtuelle ; `CREATE OR REPLACE` et `IF NOT EXISTS` en gèrent la (re)création de façon contrôlée ; `ALTER VIEW` la redéfinit ; `DROP VIEW` la supprime sans toucher aux données. `SHOW CREATE VIEW` et `INFORMATION_SCHEMA.VIEWS` permettent d'en inspecter la définition. Deux réflexes à retenir : **nommer explicitement les colonnes** plutôt que d'utiliser `*`, et **préfixer les vues** pour les repérer dans le schéma.

Toutes les vues rencontrées jusqu'ici sont recalculées à chaque interrogation. Mais que faire lorsqu'on souhaite, au contraire, **matérialiser** un résultat coûteux pour le réutiliser sans le recalculer ? C'est précisément la question abordée à la **section 9.2 — Vues matérialisées : alternatives et workarounds**.

⏭️ [Vues matérialisées : Alternatives et workarounds](/09-vues-et-donnees-virtuelles/02-vues-materialisees.md)

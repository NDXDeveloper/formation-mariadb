🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 · Vues matérialisées : alternatives et workarounds

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Vue standard et vue matérialisée : deux philosophies

Les vues étudiées jusqu'ici (§9.1) sont des vues **standard** : leur définition est une requête `SELECT` recalculée *à chaque interrogation*. C'est leur force — les données sont toujours à jour — mais aussi leur faiblesse : si la requête sous-jacente est coûteuse (agrégations lourdes, jointures multiples, gros volumes), ce coût est payé **à chaque appel**.

Une **vue matérialisée** (*materialized view*) répond à ce problème en adoptant la philosophie inverse : le résultat de la requête est **calculé une fois, puis stocké physiquement** sur disque. Les interrogations lisent directement ce résultat pré-calculé, sans réexécuter la requête d'origine. Le compromis s'inverse également : on gagne énormément en vitesse de lecture, mais les données peuvent être **périmées** (elles ne reflètent l'état des tables sources qu'au dernier rafraîchissement), et l'on consomme de l'espace de stockage supplémentaire.

| Critère | Vue standard | Vue matérialisée |
|---------|--------------|------------------|
| Stockage des données | Aucun (virtuel) | Réel (sur disque) |
| Fraîcheur | Toujours à jour | Périmée entre deux rafraîchissements |
| Coût en lecture | Recalcul à chaque requête | Lecture directe (rapide) |
| Indexation propre | Non | Oui |

## L'absence de vues matérialisées natives dans MariaDB

Le point essentiel de cette section est le suivant : **MariaDB ne propose pas d'instruction `CREATE MATERIALIZED VIEW`**. Là où PostgreSQL offre `CREATE MATERIALIZED VIEW` / `REFRESH MATERIALIZED VIEW`, et où Oracle dispose de vues matérialisées avec rafraîchissement automatique, MariaDB n'a pas d'équivalent intégré — et la série 12.x n'introduit rien de nouveau sur ce point. La matérialisation doit donc être **mise en œuvre manuellement**, à l'aide des briques que le SGBD met par ailleurs à disposition : tables, procédures, événements et déclencheurs.

> **À ne pas confondre.** L'optimiseur de MariaDB *matérialise* bel et bien certains objets en interne — tables dérivées, CTE, sous-requêtes — dans des tables temporaires, le temps d'exécuter une requête (c'est l'algorithme `TEMPTABLE`, central pour les performances des vues, voir §9.6). Mais cette matérialisation est **transitoire et propre à chaque requête** : elle n'a rien à voir avec une vue matérialisée *persistante*, qui survit entre les requêtes et que l'on rafraîchit explicitement.

## Le principe du contournement : une table réelle

Le contournement standard consiste à remplacer la vue matérialisée par une **table ordinaire**, que l'on remplit avec le résultat d'une requête puis que l'on rafraîchit périodiquement. Cette approche présente des avantages concrets : la table est persistante, on peut y créer **ses propres index** (impossible sur une vue standard), et les lectures sont aussi rapides que sur n'importe quelle table.

En contrepartie, trois responsabilités incombent désormais au concepteur : assurer le **rafraîchissement** des données, gérer le **coût de stockage**, et accepter (ou maîtriser) le **décalage de fraîcheur**.

### Créer la table matérialisée

Reprenons le schéma `employes` / `departements` de la section 9.1. Un bon candidat à la matérialisation est une **agrégation coûteuse fréquemment lue**, par exemple des statistiques par département :

```sql
CREATE TABLE mv_stats_departement (
    dept_id         INT PRIMARY KEY,
    departement     VARCHAR(50),
    nb_employes     INT,
    masse_salariale DECIMAL(14,2),
    salaire_moyen   DECIMAL(10,2),
    rafraichi_le    DATETIME
);
```

La colonne `rafraichi_le` n'est pas indispensable, mais il est vivement recommandé d'**horodater chaque rafraîchissement** : c'est ce qui permet aux applications (et aux administrateurs) de connaître l'ancienneté des données servies.

> **Convention.** De même que les vues sont souvent préfixées `v_` (§9.1), préfixer ces tables `mv_` (*materialized view*) signale clairement qu'il s'agit de données dérivées, rafraîchies, et non d'une table de référence.

## Stratégies de rafraîchissement

Deux grandes familles s'opposent : le rafraîchissement **complet** (on reconstruit tout) et le rafraîchissement **incrémental** (on ne met à jour que ce qui a changé).

### Rafraîchissement complet (full refresh)

C'est l'approche la plus simple : on vide la table et on la repeuple intégralement. On l'encapsule typiquement dans une procédure stockée :

```sql
DELIMITER //
CREATE PROCEDURE rafraichir_mv_stats_departement()
BEGIN
    TRUNCATE TABLE mv_stats_departement;

    INSERT INTO mv_stats_departement
    SELECT d.id, d.nom,
           COUNT(e.id),
           COALESCE(SUM(e.salaire), 0),
           AVG(e.salaire),
           NOW()
    FROM departements d
    LEFT JOIN employes e ON e.dept_id = d.id
    GROUP BY d.id, d.nom;
END //
DELIMITER ;
```

Cette méthode est facile à écrire et à raisonner, mais elle souffre d'un **défaut de concurrence important**. Entre le `TRUNCATE` et la fin de l'`INSERT`, la table est vide ou partielle, et toute lecture concurrente verra des données incomplètes. De plus, `TRUNCATE` provoque une **validation implicite** (*implicit commit*) et ne peut pas être annulé : l'envelopper dans une transaction ne procure donc aucune atomicité. Pour les tables réellement consultées en continu, on lui préfère le motif d'échange atomique ci-dessous.

### Le motif d'échange atomique (build-and-swap)

L'idée est de **construire la nouvelle version dans une table à part**, puis de l'échanger avec la table en service par un `RENAME TABLE`. Le renommage de plusieurs tables au sein d'une même instruction est **atomique** : aucune session concurrente ne voit d'état intermédiaire.

```sql
-- 1. Construire la nouvelle version (structure et index identiques grâce à LIKE)
CREATE TABLE mv_stats_departement_new LIKE mv_stats_departement;

INSERT INTO mv_stats_departement_new
SELECT d.id, d.nom,
       COUNT(e.id),
       COALESCE(SUM(e.salaire), 0),
       AVG(e.salaire),
       NOW()
FROM departements d
LEFT JOIN employes e ON e.dept_id = d.id
GROUP BY d.id, d.nom;

-- 2. Échange atomique : les lecteurs basculent instantanément sur la nouvelle table
RENAME TABLE
    mv_stats_departement     TO mv_stats_departement_old,
    mv_stats_departement_new TO mv_stats_departement;

-- 3. Nettoyage de l'ancienne version
DROP TABLE mv_stats_departement_old;
```

Pendant toute la durée de la reconstruction, les lectures continuent d'être servies par l'ancienne table, complète et cohérente. C'est l'approche recommandée dès qu'on ne peut tolérer aucune fenêtre de données partielles.

### Rafraîchissement incrémental (incremental / delta refresh)

Plutôt que de tout recalculer, on ne met à jour que les lignes **affectées par les changements** survenus depuis le dernier rafraîchissement. Sur de gros volumes, le gain est considérable, mais la complexité l'est aussi : il faut être capable d'**identifier les changements**, par exemple via une colonne d'horodatage `modifie_le` sur les tables sources, via des déclencheurs, ou via une capture des changements (*CDC*, voir §20.8).

Une astuce de conception facilite grandement la maintenance incrémentale des agrégats : **stocker les composants additifs plutôt que les valeurs dérivées**. Ici, conserver `nb_employes` (un `COUNT`) et `masse_salariale` (un `SUM`) suffit, car la moyenne s'en déduit (`salaire_moyen = masse_salariale / nb_employes`). Les sommes et les comptages se mettent à jour simplement (`+1`, `-1`, `+ montant`…), alors qu'une moyenne, non additive, ne peut pas être ajustée directement.

## Automatiser le rafraîchissement

Reste à déclencher le rafraîchissement. Trois mécanismes sont disponibles, selon le niveau de fraîcheur attendu.

### Avec l'Event Scheduler (rafraîchissement périodique)

Un **événement** (§8.4) appelle la procédure de rafraîchissement à intervalle régulier. C'est le choix idoine pour des besoins de type tableau de bord ou reporting, où une fraîcheur « à l'heure près » ou « au quart d'heure près » est acceptable.

```sql
SET GLOBAL event_scheduler = ON;

CREATE EVENT ev_rafraichir_stats
ON SCHEDULE EVERY 1 HOUR
DO CALL rafraichir_mv_stats_departement();
```

Les données sont alors périmées au maximum pendant la durée de l'intervalle choisi.

### Avec des déclencheurs (quasi temps réel)

Des **triggers** (§8.3) posés sur les tables sources maintiennent la table matérialisée à chaque `INSERT`, `UPDATE` ou `DELETE`. L'avantage est une fraîcheur permanente ; l'inconvénient est un **surcoût sur chaque écriture** de la table source et une logique délicate à écrire correctement. Cette approche convient surtout aux agrégats sur des tables à **faible taux d'écriture et forte lecture**.

En s'appuyant sur le principe additif évoqué plus haut, un déclencheur d'insertion reste simple :

```sql
DELIMITER //
CREATE TRIGGER trg_employes_ai
AFTER INSERT ON employes
FOR EACH ROW
BEGIN
    UPDATE mv_stats_departement
    SET salaire_moyen   = (masse_salariale + COALESCE(NEW.salaire, 0))
                          / NULLIF(nb_employes + 1, 0),   -- AVANT de muter nb/masse
        nb_employes     = nb_employes + 1,
        masse_salariale = masse_salariale + COALESCE(NEW.salaire, 0),
        rafraichi_le    = NOW()
    WHERE dept_id = NEW.dept_id;
END //
DELIMITER ;
```

> **L'ordre des affectations compte.** MariaDB évalue les affectations d'un `UPDATE` **de gauche à droite**, chaque expression voyant les valeurs **déjà modifiées** des colonnes précédentes. `salaire_moyen` doit donc être calculé **avant** que `nb_employes` et `masse_salariale` ne soient mis à jour — sans quoi la formule additionnerait une seconde fois le nouveau salaire et diviserait par un effectif déjà incrémenté, faussant la moyenne.

Le cas de l'`UPDATE` est plus subtil et doit être traité avec soin : une modification peut porter sur le **salaire** (il faut ajuster la masse salariale) *et/ou* sur le **`dept_id`** (l'employé bascule d'un département à l'autre, ce qui impacte deux lignes de la table matérialisée). Le `NULLIF(..., 0)` protège par ailleurs contre une division par zéro lorsqu'un département se vide. La fonctionnalité de **triggers multi-événements** (§8.3.4) peut aider à regrouper cette logique en un seul déclencheur.

### Avec un ordonnanceur externe (cron)

Enfin, on peut piloter le rafraîchissement depuis l'extérieur du serveur, via `cron` (ou tout planificateur équivalent) appelant le client `mariadb` ou un script qui exécute la procédure. Cette option est pertinente lorsqu'on souhaite **journaliser et orchestrer** le rafraîchissement hors de la base, ou le coordonner avec un pipeline ETL plus large.

## Pour mémoire : FlexViews

Historiquement, l'outil tiers **FlexViews** proposait de véritables vues matérialisées *incrémentales* pour MySQL/MariaDB, en lisant les journaux binaires pour détecter les changements. Il est aujourd'hui largement non maintenu et n'est mentionné ici que par souci d'exhaustivité. Pour une approche moderne de la matérialisation incrémentale à grande échelle, on s'orientera plutôt vers les outils de capture de changements (Debezium, etc.) abordés au **§20.8**.

## Choisir une stratégie

Le tableau suivant résume les arbitrages, du plus simple au plus exigeant :

| Stratégie | Fraîcheur | Surcoût en écriture | Complexité | Cas d'usage typique |
|-----------|-----------|---------------------|------------|---------------------|
| Full refresh + Event | Périodique | Nul (sur les tables sources) | Faible | Reporting, tableaux de bord |
| Build-and-swap + Event | Périodique | Nul | Moyenne | Tables lues en continu |
| Incrémental + Event | Périodique | Faible | Élevée | Gros volumes |
| Déclencheurs | Quasi temps réel | Élevé | Élevée | Agrégats très lus, peu écrits |
| Ordonnanceur externe | Périodique | Nul | Moyenne | Coordination avec ETL |

Les facteurs déterminants sont donc : le **niveau de fraîcheur** réellement nécessaire, le **volume de données** à recalculer, la **charge en écriture** sur les tables sources, et la **complexité** que l'on est prêt à maintenir.

## Points d'attention

Quelques précautions valables pour toutes les approches : la table matérialisée est une **table réelle** — elle est sauvegardée, répliquée et occupe de l'espace disque comme n'importe quelle autre. Il faut **l'indexer** en fonction des requêtes de lecture (c'est tout l'intérêt par rapport à une vue standard). Il convient d'**exposer ou documenter la fraîcheur** des données (d'où la colonne `rafraichi_le`). Enfin, pour éviter que des lecteurs ne tombent sur des données partielles, on privilégiera le **motif d'échange atomique** lors des rafraîchissements complets.

## En résumé

MariaDB n'offre pas de vues matérialisées natives : on les simule par une **table réelle**, alimentée selon une stratégie de **rafraîchissement** (complet, par échange atomique, ou incrémental) et **automatisée** via l'Event Scheduler, des déclencheurs ou un ordonnanceur externe. Le choix résulte d'un compromis entre fraîcheur, volume, charge en écriture et complexité.

Après cette parenthèse sur la matérialisation, la section suivante revient aux vues standard pour examiner une question essentielle : peut-on **écrire** à travers une vue (`INSERT`, `UPDATE`, `DELETE`) ? C'est l'objet du **§9.3 — Vues updatable : conditions et limitations**.

⏭️ [Vues updatable : Conditions et limitations](/09-vues-et-donnees-virtuelles/03-vues-updatable.md)

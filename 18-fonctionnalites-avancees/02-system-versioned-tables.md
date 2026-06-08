🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.2 Tables temporelles (System-Versioned Tables)

## Le problème : conserver l'histoire des données

Une base de données relationnelle classique ne retient, pour chaque ligne, que sa **valeur courante**. Lorsqu'un `UPDATE` modifie un salaire, un tarif ou une adresse, l'ancienne valeur disparaît purement et simplement ; un `DELETE` efface toute trace de l'enregistrement. Or de nombreux contextes exigent au contraire de savoir *ce que contenait la base à un instant donné* : audit et conformité réglementaire, traçabilité financière, reconstitution d'un état passé après une erreur de manipulation, analyse a posteriori d'un incident.

Historiquement, on répondait à ce besoin par des moyens artisanaux : tables d'audit dupliquant la structure des tables suivies, triggers recopiant l'ancienne ligne à chaque modification, ou logique applicative dédiée. Ces approches fonctionnent, mais elles sont verbeuses, fragiles, faciles à contourner par mégarde et coûteuses à maintenir. Le moindre changement de schéma oblige à répercuter la modification à plusieurs endroits.

Les **tables temporelles à versionnement système** (*system-versioned tables*) répondent à ce besoin nativement : le serveur conserve automatiquement et de façon transparente l'historique complet des lignes, sans trigger ni table annexe à entretenir.

## Le principe du versionnement système

L'idée fondatrice est d'associer à chaque ligne une **période de validité**, délimitée par deux bornes temporelles : un instant de début et un instant de fin. Ces deux bornes sont portées par deux colonnes de période, généralement nommées `row_start` et `row_end`, de type horodaté à précision microseconde. Elles forment ce que le standard appelle la *période de temps système*.

À partir de là, le comportement du serveur change subtilement :

- Une ligne **courante** est une ligne dont la borne de fin vaut une valeur maximale tenant lieu d'« infini » : elle est valide « jusqu'à nouvel ordre ».
- Lorsqu'un `UPDATE` modifie cette ligne, l'ancienne version n'est pas écrasée. Sa borne de fin est fixée à l'instant de la transaction — elle devient une ligne **historique** —, et une nouvelle ligne courante est créée avec les valeurs modifiées et une borne de début à ce même instant.
- Un `DELETE` ne supprime pas physiquement la ligne : il referme simplement sa période en positionnant la borne de fin, et la conserve comme version historique.

Tout cela reste **transparent pour les requêtes courantes**. Un `SELECT`, un `UPDATE` ou un `DELETE` ordinaires ne voient que les lignes courantes, exactement comme sur une table classique ; les colonnes de période peuvent d'ailleurs être masquées de sorte qu'un `SELECT *` n'en soit pas affecté. L'historique n'est consulté qu'à la demande, au moyen des clauses temporelles dédiées (étudiées en §18.2.2).

### Illustration

Supposons une table d'employés versionnée. Le 10 janvier à 9 h, on y enregistre Alice avec un salaire de 3000 €. Le 1ᵉʳ mars à 14 h, son salaire est porté à 3300 €. La table contient alors physiquement **deux** versions de la ligne d'Alice :

| nom | salaire | row_start | row_end | état |
|---|---|---|---|---|
| Alice | 3000 | 10/01 09:00 | 01/03 14:00 | historique |
| Alice | 3300 | 01/03 14:00 | *(infini)* | courante |

Une requête ordinaire ne renvoie que la ligne courante (3300 €) : pour l'application, rien n'a changé. La version à 3000 € demeure néanmoins disponible et interrogeable. Si Alice quittait l'entreprise, un `DELETE` refermerait à son tour la période de la ligne courante, qui rejoindrait l'historique sans qu'aucune information ne soit perdue.

## Temps système et temps applicatif

Il est essentiel de comprendre ce que mesure exactement le versionnement système : il enregistre **le temps pendant lequel une donnée a physiquement existé dans la base** (le temps de transaction). Il répond à la question « *qu'est-ce que la base affirmait, et quand ?* ».

C'est une notion distincte du **temps applicatif** (ou temps métier), qui décrit la période pendant laquelle un fait est vrai *dans le monde réel* — la durée de validité d'un contrat, d'un tarif, d'une affectation — indépendamment de la date à laquelle l'information a été saisie. Ce second axe fait l'objet des **Application Time Period Tables** (§18.3). MariaDB sait gérer les deux séparément, et même les combiner sur une même table : on parle alors de modèle **bitemporel**, qui répond simultanément aux questions « depuis quand le savions-nous ? » et « depuis quand est-ce vrai ? ».

## Un standard largement répandu

Le versionnement système n'est pas une particularité isolée de MariaDB : il s'agit d'une fonctionnalité du **standard SQL:2011**, disponible dans MariaDB depuis la version 10.3. On retrouve des mécanismes équivalents dans d'autres systèmes — *temporal tables* de SQL Server, support temporel de Db2, *Flashback* d'Oracle — ce qui en fait un atout de portabilité conceptuelle entre SGBD. S'appuyer sur une fonctionnalité normalisée plutôt que sur une mécanique de triggers maison facilite aussi la compréhension et la reprise du schéma par d'autres équipes.

## Cas d'usage

Le versionnement système trouve naturellement sa place dès que l'**historique fait partie des exigences** :

- **Audit et conformité.** Conserver une trace inaltérable de l'évolution des données répond à de nombreuses obligations réglementaires et facilite les contrôles.
- **« Voyage dans le temps ».** Reconstituer l'état exact d'une table — voire d'un jeu de tables — à une date passée, par exemple pour produire un état comptable rétroactif ou comprendre une décision prise sur la base des données de l'époque.
- **Récupération après erreur.** Retrouver la valeur d'avant une modification ou une suppression accidentelle, sans recourir à une restauration de sauvegarde complète.
- **Dimensions à évolution lente** (*slowly changing dimensions*) dans un contexte décisionnel, où l'on souhaite conserver l'historique des attributs.
- **Analyse et investigation.** Étudier la chronologie des changements lors d'un incident ou d'une enquête interne.

## Points d'attention

Cette puissance a une contrepartie qu'il faut anticiper dès la conception.

La principale est la **croissance du volume** : puisque chaque modification et chaque suppression conservent l'ancienne version, une table versionnée soumise à de nombreuses écritures grossit continûment. Une **stratégie de rétention** est donc indispensable — purge de l'historique au-delà d'une certaine ancienneté, ou organisation des données historiques en partitions que l'on peut détacher et supprimer en bloc. Cet aspect est précisément l'objet du partitionnement des données historiques (§18.2.3).

Par ailleurs, la borne de fin des lignes courantes s'appuie sur la valeur maximale du type horodaté. L'**extension du type `TIMESTAMP` introduite en 11.8** (§11.12) a repoussé l'ancienne limite de 2038 bien au-delà, ce qui sécurise durablement cette borne « infinie ». Ce format d'horodatage est aussi un point de vigilance lors des migrations (§19.9).

Enfin, certaines opérations de **DDL** sur une table versionnée (ajout, suppression ou modification de colonnes) obéissent à des règles particulières afin de préserver la cohérence de l'historique ; elles seront évoquées avec la création et la configuration.

## Organisation de cette section

La suite décline le versionnement système en trois temps :

1. **[Création et configuration](02.1-creation-configuration.md)** (§18.2.1) — comment déclarer une table versionnée, gérer les colonnes de période et activer le versionnement sur une table existante.
2. **[Requêtes temporelles](02.2-requetes-temporelles.md)** (§18.2.2) — interroger l'historique avec les clauses `AS OF`, `BETWEEN` et `FROM … TO`.
3. **[Partitionnement des données historiques](02.3-partitionnement-historique.md)** (§18.2.3) — séparer lignes courantes et lignes historiques pour maîtriser le volume et les performances.

⏭️ [Création et configuration](/18-fonctionnalites-avancees/02.1-creation-configuration.md)

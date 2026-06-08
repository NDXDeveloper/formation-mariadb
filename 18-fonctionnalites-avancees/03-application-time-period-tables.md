🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.3 Application Time Period Tables

## Temps applicatif et temps système

La section précédente (§18.2) traitait du **temps système** : le moment où une donnée a *physiquement existé dans la base*. Les *Application Time Period Tables* — tables à période de temps applicatif — répondent à une tout autre question : pendant quelle période une information est-elle **vraie dans le monde réel** ?

C'est le *temps métier* (ou *valid time*) : la durée de validité d'un tarif, d'un contrat, d'un abonnement, d'une affectation, d'une réservation. Cette période est définie et maîtrisée par l'application — c'est vous qui la renseignez —, contrairement au temps système alimenté automatiquement par le serveur. Disponible depuis MariaDB 10.4 et issue du standard SQL:2011, cette fonctionnalité partage avec le versionnement système le vocabulaire des *périodes*, mais sert un objectif différent.

## Définir une période applicative

Une période applicative repose sur **deux colonnes temporelles que vous gérez vous-même**, associées par la déclaration `PERIOD FOR`, dont le nom est libre :

```sql
CREATE TABLE tarif (
  produit_id INT           NOT NULL,
  prix       DECIMAL(10,2) NOT NULL,
  debut      DATE          NOT NULL,
  fin        DATE          NOT NULL,
  PERIOD FOR validite (debut, fin)
);
```

Quelques règles encadrent cette déclaration :

- les deux colonnes doivent être du **même type temporel** (`DATE`, `DATETIME` ou `TIMESTAMP`) et **`NOT NULL`** ;
- la période est interprétée comme un intervalle **fermé à gauche, ouvert à droite** `[debut, fin)` : une validité `[2026-01-01, 2026-04-01)` court du 1ᵉʳ janvier *inclus* au 1ᵉʳ avril *exclu* ;
- une contrainte implicite impose `debut < fin` : une période de durée nulle ou négative est rejetée.

À la différence du versionnement système, ces colonnes sont des colonnes ordinaires : on les renseigne dans les `INSERT` et on peut les lire directement.

```sql
INSERT INTO tarif (produit_id, prix, debut, fin)
VALUES (1, 80.00, '2026-01-01', '2026-04-01');
```

On peut aussi ajouter ou retirer une période sur une table existante avec `ALTER TABLE … ADD PERIOD FOR …` et `ALTER TABLE … DROP PERIOD FOR …`.

## Empêcher les chevauchements : `WITHOUT OVERLAPS`

L'apport le plus distinctif des périodes applicatives est la contrainte d'unicité temporelle `WITHOUT OVERLAPS`. Placée en dernier élément d'une clé `PRIMARY` ou `UNIQUE`, elle garantit que, pour une même valeur des autres colonnes de la clé, **deux lignes ne peuvent pas avoir des périodes qui se chevauchent** :

```sql
ALTER TABLE tarif
  ADD PRIMARY KEY (produit_id, validite WITHOUT OVERLAPS);
```

Ici, un même produit ne peut avoir deux tarifs valides simultanément. Toute tentative d'insérer une période en conflit est refusée par le serveur :

```sql
INSERT INTO tarif (produit_id, prix, debut, fin)
VALUES (1, 90.00, '2026-03-01', '2026-06-01');
-- ERREUR : la période chevauche le tarif [2026-01-01, 2026-04-01)
```

C'est exactement le mécanisme attendu pour interdire une double réservation d'une ressource, un double tarif, ou deux affectations concurrentes d'une même personne — garanti au niveau de la base plutôt que par du code applicatif faillible.

## Modifier une portion de période : `FOR PORTION OF`

Que se passe-t-il lorsqu'on ne veut modifier qu'une **partie** de la durée de validité d'une ligne ? La réponse de SQL:2011, implémentée par MariaDB, est la clause `FOR PORTION OF`, qui **découpe automatiquement** la ligne pour préserver la cohérence temporelle.

### `UPDATE … FOR PORTION OF`

Partons de notre tarif valide `[2026-01-01, 2026-04-01)` à 80 €, et appliquons une promotion de 99 € sur le seul mois de février :

```sql
UPDATE tarif FOR PORTION OF validite
  FROM '2026-02-01' TO '2026-03-01'
  SET prix = 99.00
  WHERE produit_id = 1;
```

La ligne unique est automatiquement scindée en trois, seule la portion ciblée recevant la nouvelle valeur :

| produit_id | prix | debut | fin |
|---|---|---|---|
| 1 | 80.00 | 2026-01-01 | 2026-02-01 |
| 1 | 99.00 | 2026-02-01 | 2026-03-01 |
| 1 | 80.00 | 2026-03-01 | 2026-04-01 |

### `DELETE … FOR PORTION OF`

De même, supprimer la validité sur une sous-période ne fait pas disparaître toute la ligne : elle est découpée et seules les portions concernées sont retirées. En repartant du tarif initial `[2026-01-01, 2026-04-01)` :

```sql
DELETE FROM tarif FOR PORTION OF validite
  FROM '2026-02-01' TO '2026-03-01'
  WHERE produit_id = 1;
```

Il reste deux lignes, `[2026-01-01, 2026-02-01)` et `[2026-03-01, 2026-04-01)`, la tranche de février ayant été supprimée. Lorsque la sous-période recouvre intégralement la validité d'une ligne, celle-ci est entièrement mise à jour (ou supprimée) sans découpe. C'est cette gestion automatique des découpages qui fait tout l'intérêt de `FOR PORTION OF` : elle évite la manipulation manuelle, source d'erreurs, de lignes adjacentes.

## Interroger les périodes applicatives

Contrairement au versionnement système et à sa clause `FOR SYSTEM_TIME` (§18.2.2), les périodes applicatives **ne disposent pas de syntaxe d'interrogation dédiée**. Les colonnes de période étant ordinaires, on les filtre avec des conditions `WHERE` classiques, en respectant la convention d'intervalle fermé-ouvert :

```sql
SELECT prix FROM tarif
WHERE produit_id = 1
  AND '2026-02-15' >= debut
  AND '2026-02-15' <  fin;
```

Pour obtenir l'ensemble des tarifs ayant été en vigueur sur une fenêtre donnée, on compare de la même façon les bornes de la période à celles de la fenêtre recherchée.

## Tables bitemporelles : combiner les deux périodes

Les deux mécanismes ne s'excluent pas : on peut déclarer sur une même table **une période applicative et le versionnement système**. On obtient alors une table **bitemporelle**, qui répond simultanément à deux questions distinctes — « *depuis quand est-ce vrai ?* » (temps applicatif) et « *depuis quand le savions-nous ?* » (temps système).

```sql
CREATE TABLE contrat (
  client_id INT           NOT NULL,
  montant   DECIMAL(12,2) NOT NULL,
  debut     DATE          NOT NULL,
  fin       DATE          NOT NULL,
  PERIOD FOR validite (debut, fin),
  PRIMARY KEY (client_id, validite WITHOUT OVERLAPS),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START INVISIBLE,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END   INVISIBLE,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;
```

Concrètement, corriger rétroactivement le montant d'un contrat avec un `UPDATE … FOR PORTION OF` modifie la *réalité métier* (la période de validité concernée), tandis que le versionnement système enregistre que cette correction a été saisie *aujourd'hui* — de sorte que l'on peut toujours reconstituer ce que la base affirmait avant la correction. C'est le modèle de référence pour l'audit financier et réglementaire le plus exigeant.

## Période applicative ou versionnement système ?

Les deux fonctionnalités sont complémentaires ; le tableau ci-dessous résume ce qui les distingue.

| Critère | System-Versioned (§18.2) | Application Time Period (§18.3) |
|---|---|---|
| Temps mesuré | Transaction (présence en base) | Métier (validité dans le réel) |
| Colonnes de période | Gérées par le **serveur** (`GENERATED`) | Gérées par l'**application** |
| Objectif | Audit, historique, « voyage dans le temps » | Périodes de validité effectives |
| Conservation du passé | Automatique | À la charge de l'application |
| Modification ciblée | `UPDATE`/`DELETE` ordinaires (historisés) | `FOR PORTION OF` (découpe automatique) |
| Unicité temporelle | — | `WITHOUT OVERLAPS` |
| Interrogation | `FOR SYSTEM_TIME AS OF`, etc. | `WHERE` classique sur les colonnes |

## Cas d'usage

Les périodes applicatives s'imposent dès qu'une donnée est intrinsèquement assortie d'une **durée de validité métier** : grilles tarifaires effectives sur une plage de dates, contrats et abonnements, polices d'assurance, affectations de personnel, plannings, et surtout réservations de ressources où `WITHOUT OVERLAPS` interdit nativement les conflits (salles, véhicules, chambres). Dans un contexte décisionnel, elles modélisent proprement des dimensions à validité datée. Et combinées au versionnement système, elles constituent le socle des systèmes bitemporels exigés par certaines obligations d'audit.

## Points clés à retenir

- Les périodes applicatives modélisent le **temps métier** (validité dans le réel), avec des colonnes de période **gérées par l'application**, là où le versionnement système gère le temps de transaction.
- `PERIOD FOR nom (debut, fin)` définit une période sur deux colonnes `NOT NULL` de même type ; l'intervalle est **fermé-ouvert** `[debut, fin)` et `debut < fin` est imposé.
- `WITHOUT OVERLAPS`, dans une clé, **interdit les chevauchements** de période pour une même clé — idéal contre les doubles réservations.
- `UPDATE`/`DELETE … FOR PORTION OF` modifient une sous-période en **découpant automatiquement** la ligne.
- L'interrogation se fait par `WHERE` ordinaire (pas de clause dédiée), en respectant la borne haute exclue.
- Combinées au versionnement système, elles forment une table **bitemporelle** (« depuis quand est-ce vrai ? » + « depuis quand le savions-nous ? »).

⏭️ [Colonnes virtuelles et générées](/18-fonctionnalites-avancees/04-virtual-generated-columns.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.4 — Audit de schéma

> 🏛️ Les fondations : un schéma sain — types justes, clés bien conçues, structure cohérente — évite quantité de problèmes de performance en aval.

Cet audit examine la structure des données elle-même. Des choix de conception solides allègent les requêtes, les index et le stockage avant même toute optimisation fine. Comme pour les autres audits, la démarche reste guidée par la mesure (voir la [présentation de l'annexe E](README.md)).

---

## Types de données

- [ ] **Types numériques au plus juste** — `INT` plutôt que `BIGINT` quand la plage suffit ; `DECIMAL` pour les montants monétaires, jamais `FLOAT`/`DOUBLE`.
- [ ] **Dates et heures stockées comme telles** — types temporels, pas des chaînes ; `DATETIME` pour une plage large et absolue, `TIMESTAMP` (dont la borne haute a été étendue à 2106 depuis la 11.8) lorsqu'on a besoin de conversion de fuseau.
- [ ] **Types natifs MariaDB exploités** — `UUID`, `INET6`, `JSON`, `VECTOR` au lieu de `VARCHAR`/`BLOB` génériques (cf. Ch. 2.2.5).
- [ ] **`NOT NULL` partout où c'est possible** — les valeurs `NULL` ajoutent de la complexité et un léger surcoût.
- [ ] **`TEXT`/`BLOB` utilisés avec parcimonie** — stockés hors page, ils ne devraient pas figurer dans un `SELECT *` par habitude.
- [ ] **Jeu de caractères `utf8mb4`** (défaut depuis la 11.8) avec une collation adaptée, homogène dans tout le schéma.

## Clés et contraintes (spécifique InnoDB)

- [ ] **Chaque table a une clé primaire** — InnoDB organise physiquement la table autour d'elle (index *clustered*) ; sans clé déclarée, il en crée une cachée.
- [ ] **Clé primaire courte et de préférence monotone** — une clé aléatoire (UUID v4 brut) fragmente l'index *clustered* par éclatement de pages ; et comme chaque index secondaire embarque la clé primaire, une clé volumineuse les alourdit tous.
- [ ] **Clés étrangères là où l'intégrité l'exige** — avec les colonnes de clé étrangère indexées (cf. E.2).
- [ ] **Contraintes d'intégrité posées au niveau base** — `UNIQUE`, `CHECK`, `DEFAULT` appropriés.
- [ ] **Note 12.x** — les noms de contraintes de clé étrangère ne doivent désormais être uniques que **par table**, et non plus par base ; à connaître lors de la migration de schémas multi-tables (cf. Ch. 18.12).

## Normalisation et structure

- [ ] **Normalisation appropriée** — la redondance source d'anomalies de mise à jour est évitée ; toute dénormalisation délibérée (pour la lecture) est documentée avec son compromis.
- [ ] **Pas de tables démesurément larges** — les colonnes rarement lues qui alourdissent les lignes peuvent être isolées dans une table dédiée.
- [ ] **Anti-pattern EAV évité** — lorsqu'un modèle relationnel propre ou une colonne `JSON` rendrait mieux service qu'un entité-attribut-valeur.
- [ ] **Colonnes générées pour les valeurs dérivées** — calculées (et indexables) plutôt que stockées en double (cf. Ch. 18.4).

## Grandes tables et données froides

- [ ] **Très grandes tables partitionnées** — typiquement par plage de dates, pour l'élagage et la maintenance (cf. Ch. 15.9).
- [ ] **Tables temporelles seulement là où l'historique est nécessaire** — leur croissance est surveillée (cf. Ch. 18.2).
- [ ] **Stratégie d'archivage des données froides** — suppression de partitions, ou bascule vers un moteur Archive/S3.

## Cohérence

- [ ] **Conventions de nommage homogènes** dans tout le schéma.
- [ ] **Jeu de caractères et collation cohérents, surtout sur les colonnes jointes** — un écart force une conversion et peut empêcher l'usage des index.
- [ ] **Moteur de stockage cohérent** — InnoDB par défaut ; tout mélange de moteurs est un choix assumé et compris.

## Pour aller plus loin

Les types de données sont traités au [Ch. 2.2](../../02-bases-du-sql/README.md), les contraintes au [Ch. 2.5](../../02-bases-du-sql/README.md), le partitionnement au [Ch. 15.9](../../15-performance-tuning/README.md), les colonnes générées au [Ch. 18.4](../../18-fonctionnalites-avancees/README.md), les tables temporelles au [Ch. 18.2](../../18-fonctionnalites-avancees/README.md) et le jeu de caractères par défaut au [Ch. 11.11](../../11-administration-configuration/README.md).

---

⬅️ [E.3 — Audit de requêtes](03-audit-requetes.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [Annexe F — Nouveautés MariaDB 12.3 LTS](../f-nouveautes-12-3/README.md)

⏭️ [Nouveautés MariaDB 12.3 LTS en un Coup d'Œil](/annexes/f-nouveautes-12-3/README.md)

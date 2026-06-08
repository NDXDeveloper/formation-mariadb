🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.12 Contraintes FK : noms uniques par table seulement (plus par base) 🆕

> **Version** : nouveauté de la série 12.x (MDEV-28933, dès la **12.1**), intégrée au socle **12.3 LTS**.

Depuis MariaDB 12.1 — et donc dans le socle 12.3 LTS — le **nom d'une contrainte de clé étrangère** ne doit plus être unique à l'échelle de la **base de données**, mais seulement à l'échelle de la **table** (MDEV-28933). Deux tables d'une même base peuvent désormais porter une contrainte FK du même nom, ce qui était jusqu'alors interdit.

## L'ancien comportement et sa bizarrerie

Pour des raisons historiques, MariaDB traitait les noms de contraintes FK de façon particulière. Contrairement aux contraintes `CHECK` (vérifiées au niveau de l'exécution des requêtes), les contraintes FK sont gérées par le moteur InnoDB, qui **préfixait en interne le nom de la base** à celui de la contrainte. Conséquence : le nom devait être **unique dans toute la base**, et pas seulement dans la table. Réutiliser un nom dans une autre table de la même base échouait, avec une erreur du type « *a foreign key constraint of name `ma_base`.`fk_client` already exists* ».

Cette règle était plus stricte que ce à quoi on s'attend pour un identifiant d'objet : un **index**, par exemple, n'a jamais eu besoin d'être unique qu'au sein de sa propre table.

## Le nouveau comportement (12.1+)

À partir de la 12.1, **l'unicité du nom de contrainte FK s'évalue par table**. Le `symbol` (le nom de la contrainte, dans la clause `CONSTRAINT`) ne peut donc plus entrer en collision qu'avec une autre contrainte **de la même table**. En interne, MariaDB inclut désormais le nom de la table dans l'identité de la contrainte. Le changement est par ailleurs **rétrocompatible** : tout schéma valide auparavant le reste — on gagne simplement la possibilité de réutiliser des noms entre tables.

```sql
CREATE TABLE commandes (
    id        INT PRIMARY KEY,
    client_id INT,
    CONSTRAINT fk_client FOREIGN KEY (client_id) REFERENCES clients(id)
);

CREATE TABLE factures (
    id        INT PRIMARY KEY,
    client_id INT,
    CONSTRAINT fk_client FOREIGN KEY (client_id) REFERENCES clients(id)  -- même nom
);
-- Jusqu'à la 12.0 : ERREUR — le nom « fk_client » existe déjà dans la base.
-- À partir de la 12.1 : OK — l'unicité s'évalue désormais par table.
```

Pour mémoire, le nom de contrainte est facultatif : si on l'omet, MariaDB en génère un (de la forme `nomtable_ibfk_N`). On retrouve ce nom dans les messages d'erreur et dans la vue `INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS`.

## Pourquoi c'est utile

Ce changement lève une source classique de frictions :

- **ORM et outils** : beaucoup génèrent des noms de contraintes par convention de table (par exemple un `fk_created_by` répété dans plusieurs tables). Sous l'ancienne règle, ces noms entraient en collision dès qu'ils cohabitaient dans une même base ; ils fonctionnent désormais sans renommage.
- **Consolidation et migration** : en regroupant plusieurs bases ou schémas dans une seule base MariaDB — ou en migrant depuis un système qui scope les noms de contraintes par table — on ne se heurte plus à des collisions de noms de FK imposant un renommage (§19.2, §19.10).
- **Cohérence** : les noms de contraintes FK suivent enfin la même logique que les index — uniques au sein de leur table.

## À retenir

- Depuis **MariaDB 12.1** (socle **12.3 LTS**, MDEV-28933), un nom de contrainte FK doit être **unique par table**, et non plus par base.
- Deux tables d'une même base peuvent donc **réutiliser le même nom** de contrainte FK.
- Le changement est **rétrocompatible** : rien de ce qui était valide ne casse ; c'est une **relaxation** de la règle.
- Il simplifie l'usage des **ORM/outils** et la **migration/consolidation** (§19.2), et figure parmi les changements de comportement de la 12.x à connaître (§19.10).

⏭️ [Partie 10 : Migration, Compatibilité et Architectures](/partie-10-migration-architectures.md)

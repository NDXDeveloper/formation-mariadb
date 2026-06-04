🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7 Savepoints : points de sauvegarde

Un **savepoint** (point de sauvegarde) est un **repère nommé posé à l'intérieur d'une transaction**. Il permet d'**annuler une partie seulement** des opérations — revenir jusqu'à ce repère — **sans abandonner toute la transaction**. C'est le mécanisme d'**annulation partielle** de MariaDB.

Il répond directement à une limite vue en [§6.2](02-gestion-transactions.md) : il n'existe pas de **transactions imbriquées** au sens strict ; les savepoints offrent une granularité plus fine à l'intérieur d'une **unique** transaction.

## Les trois instructions

```sql
SAVEPOINT identifiant;                      -- pose un point de sauvegarde nommé
ROLLBACK [WORK] TO [SAVEPOINT] identifiant; -- annule jusqu'à ce point (transaction maintenue)
RELEASE SAVEPOINT identifiant;              -- supprime le point (sans rien annuler)
```

- **`SAVEPOINT`** crée un repère nommé à l'endroit courant de la transaction.
- **`ROLLBACK TO SAVEPOINT`** **annule toutes les modifications effectuées après** le repère, mais **laisse la transaction ouverte** : on peut poursuivre, poser d'autres savepoints, puis valider ou annuler complètement.
- **`RELEASE SAVEPOINT`** **supprime** un repère devenu inutile, **sans annuler** quoi que ce soit (les modifications restent dans la transaction).

## Exemple : annulation partielle

```sql
START TRANSACTION;
  INSERT INTO commandes (client_id, montant) VALUES (42, 250);   -- étape 1
  SAVEPOINT apres_commande;

  INSERT INTO lignes (commande_id, produit_id, qte)
         VALUES (LAST_INSERT_ID(), 7, 3);                        -- étape 2
  -- … un problème est détecté sur l'étape 2 …
  ROLLBACK TO SAVEPOINT apres_commande;   -- l'étape 2 est annulée, l'étape 1 conservée

  INSERT INTO lignes (commande_id, produit_id, qte)
         VALUES (LAST_INSERT_ID(), 9, 1);                        -- étape 2 bis
COMMIT;   -- valide l'étape 1 et l'étape 2 bis
```

Au `ROLLBACK TO SAVEPOINT`, seule l'insertion de l'étape 2 disparaît ; la commande de l'étape 1 demeure et la transaction se poursuit normalement.

## Règles de comportement

- **Portée limitée à la transaction** : tous les savepoints sont **détruits** par un `COMMIT` ou par un `ROLLBACK` complet (sans `TO`).
- **Repère conservé après usage** : `ROLLBACK TO SAVEPOINT sp` ne supprime pas `sp` — on peut y revenir **plusieurs fois**. En revanche, **tous les savepoints créés après `sp` sont supprimés**.
- **Réutilisation d'un nom** : redéfinir un savepoint portant un nom déjà utilisé crée un **nouveau** repère et **supprime** le précédent de ce nom.
- **Repère inexistant** : `ROLLBACK TO SAVEPOINT` ou `RELEASE SAVEPOINT` sur un nom inconnu provoque une **erreur** (*SAVEPOINT … does not exist*).
- **Validations implicites** : les instructions qui valident implicitement la transaction (DDL, etc. — cf. [§6.2](02-gestion-transactions.md)) effacent aussi les savepoints.

## ⚠️ Point d'attention : les verrous ne sont pas libérés

C'est la subtilité majeure des savepoints sous InnoDB :

> **`ROLLBACK TO SAVEPOINT` annule les modifications de données postérieures au repère, mais ne libère pas les verrous de ligne acquis après ce repère.** Ces verrous restent détenus jusqu'à la fin de la transaction (`COMMIT` / `ROLLBACK` complet).

Autrement dit, revenir à un savepoint **ne réduit pas** l'empreinte de verrouillage de la transaction. Il ne faut donc pas compter sur l'annulation partielle pour relâcher la contention ([§6.4](04-verrous.md)) : seule la fin de la transaction le fera.

## Cas d'usage

- **Transactions complexes en plusieurs étapes** : isoler des étapes « à risque » derrière un savepoint, pour rejouer ou contourner une étape échouée sans perdre le travail déjà accompli.
- **Émulation de transactions imbriquées** : faute de vraie imbrication ([§6.2](02-gestion-transactions.md)), de nombreux **ORM et frameworks** (Hibernate/JPA, SQLAlchemy, etc.) implémentent leurs « transactions imbriquées » au moyen de savepoints ([§17.3](../17-integration-developpement/03-orm-frameworks.md)).
- **Gestion d'erreurs dans les routines stockées** : poser un savepoint, tenter une opération, et y revenir dans un *handler* en cas d'échec (voir [§8.6 — Gestion des erreurs et exceptions](../08-programmation-cote-serveur/06-gestion-erreurs-exceptions.md)).
- **Traitements par lots** : dans une boucle, un savepoint par élément permet d'annuler le seul élément fautif et de poursuivre le lot.

## À retenir

- Un **savepoint** = repère nommé permettant l'**annulation partielle** d'une transaction.
- Trois instructions : **`SAVEPOINT`**, **`ROLLBACK TO SAVEPOINT`** (annule jusqu'au repère, transaction maintenue), **`RELEASE SAVEPOINT`** (supprime le repère).
- `ROLLBACK TO` **conserve** le repère visé mais **supprime** les savepoints plus récents ; `COMMIT` et `ROLLBACK` complet effacent **tous** les savepoints.
- ⚠️ Revenir à un savepoint **ne libère pas** les verrous acquis après lui — ils tiennent jusqu'à la fin de la transaction.
- Usages typiques : étapes à risque, émulation d'imbrication par les ORM, gestion d'erreurs en routines stockées, traitements par lots.

---

> **Section suivante** : [6.8 — Transactions distribuées (XA)](08-transactions-distribuees-xa.md), pour coordonner une transaction s'étendant sur plusieurs ressources.

⏭️ [Transactions distribuées (XA)](/06-transactions-et-concurrence/08-transactions-distribuees-xa.md)

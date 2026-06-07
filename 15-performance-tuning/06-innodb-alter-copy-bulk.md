🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.6 `innodb_alter_copy_bulk` : Construction d'index efficace

Lorsqu'une instruction `ALTER TABLE` doit reconstruire une table avec l'algorithme `COPY` — le mode le plus universel, choisi lorsqu'une modification ne peut être réalisée « en place » (INPLACE/INSTANT/NOCOPY) —, la phase la plus coûteuse est souvent la **reconstruction des index**. Le paramètre `innodb_alter_copy_bulk` active un chemin de chargement en masse (*bulk insert*) qui rend cette construction nettement plus rapide et produit des index plus compacts.

Ce paramètre n'est pas une nouveauté propre à la 12.x : il a été introduit dans plusieurs branches stables (10.11.9, 11.1.6, 11.2.5, 11.4.3, 11.5.2, 11.6.1) et est bien sûr présent en 12.3, où il est **activé par défaut**.

---

## Le problème : l'ancienne construction d'index par `ALGORITHM=COPY`

L'algorithme `COPY` reconstruit une table selon un schéma classique : création d'une nouvelle table avec la structure cible, copie des lignes depuis l'ancienne, puis bascule (voir aussi §18.11). Historiquement, la construction des index pendant cette copie était particulièrement inefficace :

- les lignes étaient insérées dans **chaque index une par une** (ligne à ligne), comme autant d'insertions individuelles ;
- **aucun tampon de tri** n'était utilisé côté InnoDB ;
- l'opération générait une **grande quantité de journalisation** undo et redo pour la seconde copie de la table (au point qu'une astuce de « commit toutes les 10 000 lignes » avait longtemps été nécessaire pour accélérer la reprise après crash).

Les conséquences étaient une reconstruction lente, une forte pression sur le redo log, et des index bâtis par insertions désordonnées — donc **moins compacts et plus fragmentés**. À l'inverse, la « *fast index creation* » dédiée (un simple `CREATE INDEX` sur un index secondaire) bénéficiait déjà d'une construction efficace par tri puis chargement en masse ; mais une reconstruction complète de table via `COPY` n'en profitait pas.

---

## La solution : le chemin *bulk insert* pour `ALGORITHM=COPY`

`innodb_alter_copy_bulk` étend aux reconstructions par `ALGORITHM=COPY` l'optimisation de chargement en masse qu'InnoDB appliquait déjà aux insertions dans une table vide. Le principe est le suivant :

- les lignes sont chargées **en masse** dans la nouvelle table (vide), plutôt qu'insérées une à une ;
- les index sont construits **pré-triés, page par page**, exactement comme le fait `CREATE INDEX`, en s'appuyant sur les tampons de tri d'InnoDB (`innodb_sort_buffer_size`) ;
- la journalisation est **minimisée** : au lieu d'enregistrer chaque ligne dans les journaux undo/redo, la reprise s'appuie sur un unique enregistrement undo de type « tronquer en cas de rollback » — l'intégralité de la nouvelle table étant simplement abandonnée si l'opération échoue.

Les bénéfices sont triples : une reconstruction et une construction d'index **beaucoup plus rapides**, des **index plus compacts et moins fragmentés** (bâtis triés, avec un taux de remplissage maîtrisé, ce qui améliore l'occupation disque et les performances de lecture), et une **journalisation undo/redo fortement réduite**, donc moins d'E/S et moins de pression sur le redo log (cf. §15.4 et §15.5).

---

## Activation et valeur par défaut

`innodb_alter_copy_bulk` est **activé par défaut** (`ON`) dans les versions qui l'embarquent, dont MariaDB 12.3. Le désactiver (`OFF`) restaure l'ancien comportement d'insertion ligne à ligne. C'est une variable globale dynamique :

```sql
-- Vérifier l'état courant
SHOW VARIABLES LIKE 'innodb_alter_copy_bulk';

-- Désactivation éventuelle (diagnostic uniquement)
SET GLOBAL innodb_alter_copy_bulk = OFF;
```

```ini
[mariadb]
innodb_alter_copy_bulk = ON
```

En pratique, il n'y a **aucune raison de le désactiver** : on le laisse activé. Sa désactivation ne se justifie que pour isoler un comportement lors d'un diagnostic.

---

## Conditions et limitations

Le chemin *bulk* s'applique aux reconstructions par `ALGORITHM=COPY`, c'est-à-dire lorsque la modification demandée ne peut être traitée par un algorithme plus léger (INSTANT, NOCOPY, INPLACE) ; MariaDB choisit automatiquement l'algorithme le plus efficace pour chaque opération (§18.11). L'optimisation suppose que la nouvelle table est **construite à neuf** (initialement vide), ce qui est précisément le cas d'une reconstruction.

Sa principale limite tient à la **vérification ligne à ligne des contraintes**. Le chargement en masse construit les index sans traitement individuel des lignes ; or certaines contraintes, en particulier les **clés étrangères**, doivent être évaluées pour chaque ligne. Dans ces situations, InnoDB retombe généralement sur le chemin classique. Par ailleurs, quelques cas particuliers (par exemple des valeurs BLOB très volumineuses) ont fait l'objet de corrections dans les versions de maintenance qui ont suivi l'introduction du paramètre.

---

## Variables liées : régler la construction d'index

Deux variables InnoDB encadrent la construction d'index par tri et chargement en masse, et méritent d'être connues conjointement avec `innodb_alter_copy_bulk` :

- **`innodb_sort_buffer_size`** : taille de chacun des **trois** tampons qu'InnoDB alloue pendant la phase de tri d'une construction d'index. Sa valeur par défaut est de 1 Mo. L'augmenter accélère la création d'index sur de grandes tables, en réduisant le nombre de passes de fusion lors du tri.
- **`innodb_fill_factor`** : pourcentage de remplissage de chaque page de l'arbre B lors d'une construction d'index triée (donc en masse). Il s'agit d'une indication plutôt que d'une valeur absolue. Par défaut proche de 100 % (remplissage maximal, index le plus compact) ; une valeur de 70, par exemple, réserve 30 % d'espace libre sur chaque page pour absorber les insertions futures et limiter les éclatements de page (*page splits*). C'est un compromis entre compacité immédiate et marge de croissance.

Ensemble, ces deux paramètres déterminent la **vitesse** de la construction (via le tri) et la **compacité** des index produits par le chemin *bulk*.

---

## Relation avec les algorithmes d'`ALTER TABLE`

`ALTER TABLE` dispose de plusieurs algorithmes : `INSTANT` (modification de métadonnées quasi instantanée), `NOCOPY` et `INPLACE` (sans copie complète de la table), et `COPY` (reconstruction complète). Ce dernier est le plus universel — il prend en charge des modifications arbitraires — mais aussi le plus coûteux, puisqu'il recopie l'intégralité de la table et de ses index. C'est précisément ce chemin de repli que `innodb_alter_copy_bulk` rend beaucoup moins onéreux.

Pour les modifications portant sur de très grandes tables, on combinera utilement cette optimisation avec les mécanismes de DDL en ligne et de réduction du lag de réplication (cf. §18.11 sur l'Online Schema Change, et §13.10 sur l'*optimistic ALTER TABLE*).

---

## Points clés à retenir

- `innodb_alter_copy_bulk` fait construire les index d'une reconstruction par `ALGORITHM=COPY` en **masse et pré-triés, page par page** (comme `CREATE INDEX`), au lieu de l'ancienne insertion **ligne à ligne**.
- Résultat : reconstruction **plus rapide**, index **plus compacts et moins fragmentés**, et journalisation undo/redo **fortement réduite**.
- Il est **activé par défaut** (présent depuis 10.11.9 / 11.4.3 / etc., et en 12.3) ; on le laisse activé, sa désactivation n'étant utile qu'en diagnostic.
- Il s'applique aux reconstructions `COPY` d'une table construite à neuf ; il **retombe sur le chemin classique** lorsque des contraintes doivent être vérifiées ligne à ligne (typiquement les **clés étrangères**).
- Les variables `innodb_sort_buffer_size` (vitesse du tri, 1 Mo par défaut) et `innodb_fill_factor` (compacité des pages, proche de 100 % par défaut) complètent ce réglage.

⏭️ [Analyse des requêtes lentes](/15-performance-tuning/07-analyse-requetes-lentes.md)

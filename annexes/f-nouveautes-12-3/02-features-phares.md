🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.2 — Les features phares : binlog réécrit (~4× en écriture) et Optimizer Hints

Parmi les nouveautés de la série 12.x recensées en [F.1](01-tableau-recapitulatif.md), deux se distinguent par leur portée structurante. La première agit au niveau du moteur et relève le plafond d'écriture de l'instance ; la seconde donne aux développeurs et aux DBA un contrôle fin du plan d'exécution, requête par requête. Cette section les présente en détail ; la procédure d'adoption et l'impact sur une migration sont traités en [F.4](04-impact-migration-compatibilite.md).

## 1. Le journal binaire réécrit et intégré à InnoDB

### Le problème : deux journaux durables à coordonner

Dans l'architecture historique, MariaDB maintient deux journaux durables distincts. Le **journal binaire** (binlog), situé dans la couche serveur, enregistre les modifications pour la réplication et la restauration *point-in-time*. Le **redo log** d'InnoDB, propre au moteur de stockage, garantit la durabilité et la reprise après incident.

Pour qu'une transaction soit, après un crash, présente *dans les deux journaux ou dans aucun*, le serveur les coordonne via une forme de *two-phase commit* interne. En configuration durable — `innodb_flush_log_at_trx_commit = 1` et `sync_binlog = 1` —, chaque validation entraîne plusieurs opérations de synchronisation sur disque (`fsync`). Le *group commit* regroupe ces écritures pour en amortir le coût, mais la coordination entre deux journaux indépendants reste un goulet d'étranglement bien connu pour les charges fortement transactionnelles.

### La solution en 12.x

La série 12.x réécrit le journal binaire et l'**intègre directement à InnoDB** (§11.5.4). En faisant du binlog une composante du mécanisme transactionnel et durable du moteur, la synchronisation redondante entre deux journaux séparés devient inutile : un seul chemin d'écriture durable suffit à garantir la cohérence. Le gain annoncé sur cette base est d'environ **4× en écriture**.

### Ce que cela change concrètement

- **Performance** — le bénéfice est maximal sur les charges OLTP intensives en écriture configurées pour une durabilité stricte, là où la double synchronisation pesait le plus.
- **Garanties préservées** — le contenu du binlog continue d'être produit : la réplication et la restauration *point-in-time* fonctionnent comme auparavant.
- **Adoption optionnelle** — d'après le §19.10, le binlog InnoDB est *optionnel* : on peut l'activer ou conserver le binlog classique. L'activation et le paramétrage relèvent du §11.5.4, en complément des bases du binlog (configuration, formats, purge) couvertes aux §11.5.1 à §11.5.3.
- **Point de vigilance migration** — il s'agit d'un changement d'architecture du journal ; ses implications sont reprises en [F.4](04-impact-migration-compatibilite.md) et au [§19.10](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md).

## 2. Le moteur d'Optimizer Hints

### Pourquoi des indices d'optimisation

L'optimiseur basé sur les coûts choisit en général de bons plans, mais il peut se tromper : statistiques obsolètes, distribution de données très déséquilibrée, jointures complexes. Les *optimizer hints* permettent de **guider ou de forcer une décision précise de l'optimiseur pour une requête donnée**, sans modifier la configuration globale ni réécrire la logique de la requête.

MariaDB offrait déjà des leviers (drapeaux `optimizer_switch`, `FORCE INDEX`, etc.). La série 12.x ajoute un véritable moteur d'indices sous forme de commentaires, dans la lignée de la syntaxe popularisée par Oracle et MySQL 8 (§15.15).

### Forme générale

Les hints se placent dans un commentaire spécial, introduit par `/*+ … */`, juste après le mot-clé de l'instruction :

```sql
SELECT /*+ HINT_A(...) HINT_B(...) */ col1, col2
FROM ...
```

Plusieurs hints peuvent être combinés. Ils ciblent des tables ou des index et peuvent, dans les requêtes complexes, viser un bloc de requête nommé.

### Les familles de hints

**Nommage des blocs de requête (§15.15.1).** Le hint `QB_NAME` attribue un nom à un bloc (sous-requête, table dérivée…) afin que d'autres hints puissent le cibler sans ambiguïté ; MariaDB attribue par ailleurs des noms implicites aux blocs. C'est indispensable dès qu'une requête comporte plusieurs niveaux.

**Hints de table et d'index (§15.15.2).** Ils pilotent les méthodes d'accès et l'usage des index : `[NO_]INDEX` pour forcer ou interdire un index, `JOIN_INDEX` / `GROUP_INDEX` / `ORDER_INDEX` pour choisir l'index servant respectivement la jointure, le `GROUP BY` ou l'`ORDER BY`, et `NO_ICP`, `MRR`, `BKA`, `BNL` pour activer ou désactiver des stratégies d'accès (*Index Condition Pushdown*, *Multi-Range Read*, *Batched Key Access*, *Block Nested-Loop*).

**Hints d'ordre de jointure (§15.15.3).** Ils contraignent l'ordre des tables : `JOIN_FIXED_ORDER` impose l'ordre textuel, `JOIN_ORDER` un ordre précis, `JOIN_PREFIX` et `JOIN_SUFFIX` épinglant respectivement les premières et les dernières tables de la séquence.

**Hints de sous-requête (§15.15.4).** Ils contrôlent le traitement des sous-requêtes et tables dérivées : `SEMIJOIN` pour les stratégies de semi-jointure (`IN` / `EXISTS`), `SUBQUERY` pour le choix matérialisation / *in-to-exists*, `MERGE` pour fusionner une dérivée plutôt que la matérialiser, `SPLIT_MATERIALIZED` pour l'optimisation par découpage de la matérialisation, et `DERIVED_CONDITION_PUSHDOWN` pour pousser des conditions dans une table dérivée.

**Limitation du temps d'exécution (§15.15.5).** Le hint `MAX_EXECUTION_TIME` plafonne la durée d'une requête (en millisecondes — voir §15.15.5), garde-fou utile contre les requêtes qui s'emballent.

### Illustration

À titre d'exemple de la forme — la syntaxe et les options exactes étant détaillées au §15.15 :

```sql
SELECT /*+ JOIN_ORDER(c, o) NO_INDEX(o idx_date) MAX_EXECUTION_TIME(2000) */
       c.nom, COUNT(*)
FROM clients c
JOIN commandes o ON o.client_id = c.id
GROUP BY c.nom;
```

Ici, l'ordre de jointure est imposé, un index est écarté sur `commandes`, et la requête est interrompue au-delà de deux secondes.

### Bénéfices et précautions

L'intérêt est d'obtenir un contrôle **local et ciblé**, sans effet de bord global : plans reproductibles, comportement prévisible, garde-fous explicites. En contrepartie, un hint *fige* une décision : un choix pertinent aujourd'hui peut devenir contre-productif à mesure que les données et les statistiques évoluent. La bonne pratique consiste à les employer avec parcimonie, à documenter la raison de chaque hint, à privilégier d'abord la correction des statistiques ou de l'indexation, puis à re-tester régulièrement. L'usage des optimizer hints fait d'ailleurs partie de l'audit de requêtes (Annexe E.3).

## Pourquoi ces deux features sont « phares »

Les deux apports se complètent. Le binlog intégré à InnoDB relève le **plafond d'écriture** au niveau du moteur, de façon transparente une fois activé ; les Optimizer Hints offrent un **contrôle chirurgical** au niveau de chaque requête. Ensemble, ils donnent à la série 12.x une orientation nettement tournée vers la performance et la maîtrise du comportement. Le panorama complet des nouveautés figure en [F.1](01-tableau-recapitulatif.md) ; l'impact migration du changement de binlog est détaillé en [F.4](04-impact-migration-compatibilite.md), et le détail technique dans les chapitres §11.5.4 (binlog) et §15.15 (hints).

⏭️ [Compatibilité renforcée Oracle & MySQL](/annexes/f-nouveautes-12-3/03-compatibilite-oracle-mysql.md)

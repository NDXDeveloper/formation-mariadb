🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.12 — Tables temporaires en réplication : prévisibilité (`create_tmp_table_binlog_formats`) 🆕

> **Chapitre 13 — Réplication** · Version de référence : **MariaDB 12.3 LTS**

---

> **Nouveauté MariaDB 12.3 LTS** (MDEV-36099). Cette section explique pourquoi les tables temporaires ont longtemps été une source d'imprévisibilité en réplication, en quoi consistent les nouvelles règles déterministes de journalisation introduites en 12.3, et comment la variable `create_tmp_table_binlog_formats` permet de contrôler ce comportement — y compris pour préserver la compatibilité avec la 11.8.

## Le contexte : une zone historiquement délicate

Les tables temporaires (`CREATE TEMPORARY TABLE`) sont **locales à la session** : elles ne sont visibles que de la connexion qui les a créées et disparaissent à la fin de celle-ci. Cette nature transitoire entre en tension directe avec la réplication, dont l'objectif est de rejouer fidèlement, sur un serveur distinct, ce qui s'est produit sur le primaire.

Pendant des années, le sort d'une table temporaire dans le binlog a dépendu du format de journalisation actif **au moment** des différentes opérations, et de la séquence de commandes qui les avait précédées. Le résultat était difficile à prévoir : selon le contexte, une même suite d'instructions pouvait être journalisée en `STATEMENT` ou en `ROW`, et une table temporaire pouvait être recréée ou non sur le réplica. Cette imprévisibilité provoquait des incohérences subtiles, des erreurs de réplication, et compliquait l'usage de la réplication parallèle. La 12.3 remplace ces comportements implicites par un jeu de règles explicites et stables.

## Rappel : tables temporaires et formats de binlog

Le comportement des tables temporaires dépend du format de journalisation binaire (`binlog_format`). Le tableau suivant en résume le principe, valeur par défaut en 12.3 incluse.

| `binlog_format` | Table temporaire journalisée (recréée sur le réplica) ? | Effet par défaut en 12.3 |
|---|---|---|
| `STATEMENT` | Oui | Inchangé : la table temporaire est recréée sur le réplica |
| `MIXED` | **Non** (par défaut) | Les instructions modifiant une **table permanente** à partir d'une table temporaire basculent en `ROW` |
| `ROW` | Non (jamais) | Inchangé : seules les modifications des tables permanentes sont journalisées |

Le point essentiel à mémoriser : en format **`ROW`**, les tables temporaires ne sont **jamais** transmises au réplica. Seuls les changements qu'elles induisent sur des tables permanentes (lignes insérées, mises à jour, supprimées) le sont. C'est le mode le plus propre vis-à-vis des tables temporaires, et la nouvelle valeur par défaut s'aligne sur cette philosophie pour le mode `MIXED`.

## Les nouvelles règles déterministes de la 12.3

À partir de la 12.3, la journalisation des tables temporaires suit des règles fixes et documentées :

- Une table temporaire n'est créée sur le réplica **que** si le primaire journalise en format `STATEMENT`.
- Les **modifications** d'une table temporaire ne sont journalisées **que si** son `CREATE` l'a été lui-même.
- De façon symétrique, le `DROP TEMPORARY TABLE` n'est journalisé **que si** le `CREATE` correspondant l'avait été.
- En format `ROW`, aucune opération sur les tables temporaires n'est journalisée.

La conséquence directe en mode `MIXED` est importante : par défaut, les tables temporaires n'y sont **plus** recréées sur le réplica. Toute instruction qui écrit dans une table **permanente** en lisant une table temporaire doit donc être journalisée en format `ROW`, puisque la table temporaire référencée n'existe pas côté réplica. On journalise alors le résultat concret (les lignes), et non l'instruction qui s'appuierait sur un objet absent.

Cette nouveauté corrige au passage plusieurs cas où le mode `MIXED` basculait en `ROW` de façon inattendue alors que `STATEMENT` aurait été le choix attendu, ce qui était l'une des principales sources de confusion.

## La variable `create_tmp_table_binlog_formats`

C'est le levier de configuration introduit par cette fonctionnalité. Elle énumère les formats de binlog pour lesquels la création (et donc les changements ultérieurs) des tables temporaires est journalisée.

| Valeur | Comportement |
|---|---|
| `STATEMENT` *(défaut en 12.3)* | `CREATE TEMPORARY TABLE` journalisé uniquement lorsque `binlog_format = STATEMENT`. En mode `MIXED`, les tables temporaires ne sont pas recréées sur le réplica. |
| `MIXED,STATEMENT` | Restaure le comportement antérieur (11.8 et versions précédentes) : la création est aussi journalisée en format `STATEMENT` lorsqu'on est en mode `MIXED`. |

### Inspection et modification

La variable est consultable et modifiable comme les autres variables système :

```sql
-- Vérifier la valeur effective (STATEMENT par défaut en 12.3)
SELECT @@global.create_tmp_table_binlog_formats;

-- Rétablir le comportement « legacy » : journaliser les tables
-- temporaires également en mode MIXED
SET GLOBAL create_tmp_table_binlog_formats = 'MIXED,STATEMENT';
```

Ou de façon persistante dans le fichier de configuration :

```ini
[mariadb]
binlog_format = MIXED

# Conserver le comportement historique en mode MIXED
create_tmp_table_binlog_formats = MIXED,STATEMENT
```

### Réglage côté réplica

Pour obtenir exactement les mêmes résultats qu'avec les anciennes versions de MariaDB, il faut aligner la valeur de la variable sur le réplica comme sur le primaire. Le réplica fonctionne même sans cet alignement, mais en mode `MIXED` ses propres tables temporaires ne seraient alors pas journalisées dans **son** binlog (ce qui compte dès lors qu'il sert lui-même de source dans une topologie en cascade). Après modification, les threads de réplication doivent être relancés pour prendre en compte la nouvelle valeur :

```sql
STOP REPLICA;
START REPLICA;
```

## Conséquences pratiques

### Réplication parallèle

Le bénéfice le plus structurant concerne la réplication parallèle. Une table temporaire journalisée en `STATEMENT` ne peut pas être appliquée en parallèle sur le réplica, car le rejeu dépend d'un état de session reconstitué séquentiellement. En basculant par défaut vers une journalisation en `ROW` des effets des tables temporaires, la 12.3 lève cet obstacle et améliore le parallélisme d'application — un gain direct sur le délai de réplication (lag) des charges qui font un usage intensif de tables intermédiaires.

### Taille du binlog et trafic réseau

Le revers du compromis mérite d'être anticipé. Considérons un traitement de type ETL qui matérialise un gros volume dans une table temporaire avant d'alimenter une table permanente :

```sql
BEGIN;
CREATE TEMPORARY TABLE tmp AS
    SELECT a, b, c
    FROM grande_table1
    JOIN grande_table2 ON ...;

INSERT INTO autre_table
    SELECT b, c FROM tmp WHERE a < 100;

DROP TEMPORARY TABLE tmp;
COMMIT;
```

Avec l'ancien comportement en mode `MIXED`, ce bloc se résumait à quelques centaines d'octets d'événements `Query` (instructions). Avec le nouveau comportement par défaut, l'`INSERT ... SELECT FROM tmp` est journalisé en `ROW` : il peut alors représenter plusieurs **mégaoctets** d'événements de lignes. Pour la grande majorité des charges, ce surcoût est acceptable et compensé par la robustesse et le parallélisme gagnés. Mais sur des workloads manipulant de très gros volumes via des tables temporaires, il convient de surveiller la croissance du binlog et la bande passante de réplication, et d'envisager le réglage `MIXED,STATEMENT` si le profil le justifie.

### Cohérence entre primaire et réplica

Les règles déterministes réduisent un risque connu : celui d'une table temporaire alimentée par une fonction **non déterministe**.

```sql
-- En journalisation STATEMENT d'une table temporaire,
-- une valeur générée ainsi diverge entre primaire et réplica
INSERT INTO tmp VALUES (UUID());
```

Une valeur produite par `UUID()` (ou une autre fonction non reproductible) n'est par nature pas identique d'un serveur à l'autre. Si la modification de la table temporaire n'est pas transmise telle quelle — ce qui est précisément le comportement par défaut en 12.3 — on évite la classe d'incohérences où primaire et réplica finissent avec des contenus différents.

## Compatibilité ascendante et migration 11.8 → 12.3

Ce changement fait partie des **changements de comportement** à intégrer lors d'une montée de version depuis la 11.8 (voir [§19.10](../19-migration-compatibilite/10-migration-11-8-vers-12-3.md)). Trois réflexes :

1. **Identifier les charges en mode `MIXED` qui reposent sur des tables temporaires.** C'est là que le comportement diffère ; en `STATEMENT` pur comme en `ROW` pur, aucune surprise.
2. **Choisir explicitement la valeur de `create_tmp_table_binlog_formats`.** Conserver le défaut `STATEMENT` (recommandé pour bénéficier du parallélisme et de la prévisibilité) ou poser `MIXED,STATEMENT` pour reproduire à l'identique le comportement de la 11.8 pendant une phase de transition.
3. **Mesurer l'impact sur le volume de binlog** sur un environnement de pré-production représentatif avant de basculer en production.

## Points de vigilance

- **Rejeu de binlogs et PITR (`mariadb-binlog`).** Lorsque l'on rejoue plusieurs fichiers de binlog, il faut le faire **dans une seule connexion**. Les tables temporaires sont détruites à la déconnexion du client : si un fichier crée une table temporaire et qu'un fichier suivant y fait référence via une connexion distincte, le serveur renvoie une erreur « table inconnue ». On passe donc tous les fichiers en une seule commande, ou on les concatène avant exécution. Ce point reste vrai indépendamment de la 12.3 et concerne directement la restauration jusqu'à un instant donné (point-in-time recovery).
- **Fonctions non déterministes.** Comme illustré plus haut, alimenter une table temporaire avec `UUID()`, `RAND()` ou des valeurs dépendant de l'environnement reste à proscrire dès qu'une journalisation en `STATEMENT` est en jeu.
- **Format `ROW`.** Si votre déploiement est déjà entièrement en `ROW`, le comportement vis-à-vis des tables temporaires ne change pas : elles n'étaient pas journalisées, elles ne le sont toujours pas. La variable n'a alors pas d'effet pratique sur la création des tables temporaires.

## Idées clés à retenir

La 12.3 transforme la journalisation des tables temporaires, d'un comportement implicite et difficile à prévoir, en un jeu de règles explicites pilotées par `create_tmp_table_binlog_formats`. Le défaut (`STATEMENT`) privilégie la **prévisibilité** et la **réplication parallèle** au prix d'un binlog potentiellement plus volumineux pour les charges riches en tables temporaires ; la valeur `MIXED,STATEMENT` reste disponible pour qui doit retrouver le comportement de la 11.8. Le bon réflexe est de **choisir cette valeur consciemment** plutôt que de la subir, en s'appuyant sur le profil réel de la charge et sur une mesure de l'impact côté binlog.

⏭️ [Haute Disponibilité](/14-haute-disponibilite/README.md)

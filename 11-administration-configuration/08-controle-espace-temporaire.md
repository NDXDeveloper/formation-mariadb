🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.8 — Contrôle de l'espace temporaire (`max_tmp_space_usage`, `max_total_tmp_space_usage`)

La section précédente ([11.7](07-gestion-espace-disque.md)) a montré *où* l'espace disque est consommé et *comment* le récupérer. Le tablespace temporaire (`ibtmp1`) et les fichiers temporaires créés dans `tmpdir` y figuraient comme des sources de croissance pouvant gonfler de plusieurs gigaoctets sous l'effet d'une seule requête mal écrite. Cette section présente le mécanisme introduit en **MariaDB 11.5** (et donc disponible en 12.3) qui permet de **borner explicitement** l'espace disque temporaire, par session et globalement : un garde-fou destiné à empêcher qu'une requête isolée ne sature le disque et n'interrompe le service pour l'ensemble des utilisateurs.

---

## Le problème : la requête qui sature le disque

Considérons une jointure cartésienne involontaire, un `GROUP BY` sur une colonne non indexée portant sur des centaines de millions de lignes, ou un tri (`ORDER BY`) qui dépasse la mémoire allouée. Dans tous ces cas, MariaDB matérialise des données intermédiaires sur disque. Une seule requête de ce type peut écrire des dizaines de gigaoctets de fichiers temporaires en quelques minutes.

La conséquence dépasse de loin la requête fautive : lorsque le volume hébergeant `tmpdir` ou `ibtmp1` est saturé, **toutes** les requêtes nécessitant de l'espace temporaire échouent, et le serveur peut basculer en lecture seule ou s'interrompre (voir [11.7](07-gestion-espace-disque.md)). Un utilisateur unique, par une requête malheureuse, provoque une indisponibilité globale.

Avant la 11.5, on ne disposait que d'instruments indirects et grossiers pour limiter ce risque : isoler `tmpdir` sur un volume dédié, plafonner indirectement via `max_join_size` ou `sql_select_limit`, ou surveiller a posteriori. Les variables `max_tmp_session_space_usage` et `max_tmp_total_space_usage` apportent désormais une limite **directe et précise** sur l'espace disque temporaire réellement consommé.

---

## Le modèle en couches de l'espace temporaire

Pour positionner correctement ces deux variables, il faut comprendre les trois niveaux qui interviennent dans la gestion d'une table temporaire :

1. **En mémoire** : une table temporaire interne est d'abord créée en RAM. Sa taille est plafonnée par `tmp_memory_table_size` (et par le plus petit de `tmp_table_size` / `max_heap_table_size` pour les tables internes).
2. **Débordement sur disque** : lorsque ce plafond mémoire est dépassé, MariaDB convertit la table en table **sur disque**. Pour les tables temporaires internes, MariaDB utilise par défaut le moteur **Aria** (et non InnoDB) ; les tables temporaires explicites créées en `ENGINE=InnoDB` vont, elles, dans `ibtmp1`. Les fichiers de tri (`filesort`) sont également écrits dans `tmpdir`.
3. **Plafond disque** : **c'est ici qu'interviennent `max_tmp_session_space_usage` et `max_tmp_total_space_usage`**, qui bornent l'espace disque temporaire consommé une fois le débordement sur disque survenu.

Au-delà de ces couches logiques, le dimensionnement et la réduction physique du fichier `ibtmp1` relèvent de la section [11.7](07-gestion-espace-disque.md). On peut résumer la chaîne ainsi : **limite mémoire** (`tmp_memory_table_size`) → **quota disque** (cette section) → **fichier physique** (`ibtmp1`, dimensionné en 11.7). Les trois leviers sont complémentaires.

---

## Les deux variables et leur périmètre

### Point de vigilance : le nommage a évolué

Le titre de cette section reprend les noms d'origine, mais ceux-ci ont changé peu après l'introduction de la fonctionnalité. Il est essentiel de le savoir pour ne pas se tromper de variable en 12.3 :

| Rôle | Nom en 11.5.0 (initial) | Nom canonique à partir de 11.5.1, **utilisé en 12.3** |
|---|---|---|
| Limite **par session** | `max_tmp_space_usage` | **`max_tmp_session_space_usage`** |
| Limite **globale** (toutes connexions) | `max_total_tmp_space_usage` | **`max_tmp_total_space_usage`** |

En MariaDB 12.3, on utilise donc **`max_tmp_session_space_usage`** et **`max_tmp_total_space_usage`**. Les noms d'origine (`max_tmp_space_usage`, `max_total_tmp_space_usage`) désignent la même fonctionnalité mais n'ont été en vigueur que dans la toute première version. La suite de cette section emploie les noms canoniques.

### `max_tmp_session_space_usage` — la limite par session

Cette variable plafonne l'espace disque temporaire consommé par **une seule session**. C'est le garde-fou contre la requête individuelle qui dérape : la connexion fautive est arrêtée, mais les autres sessions continuent de fonctionner normalement.

### `max_tmp_total_space_usage` — la limite globale

Cette variable plafonne la **somme** de l'espace disque temporaire utilisé par **l'ensemble des connexions**. Elle protège contre l'effet cumulatif de nombreuses sessions consommant chacune une part raisonnable, mais qui, additionnées, satureraient le disque.

### Sémantique commune

Les deux variables partagent les mêmes règles :

- la valeur est exprimée **en octets**, mais arrondie à des blocs de **65536 octets** (les valeurs non multiples de 65536 sont arrondies à l'inférieur) ;
- la valeur **`0` désactive** la limite correspondante ; la valeur **par défaut n'est toutefois pas `0` mais 1 Tio** (`1099511627776` octets, vérifié sur 12.3.2) — un plafond si élevé qu'il est en pratique inopérant sur la plupart des serveurs, et qu'il convient donc d'**abaisser** à une valeur adaptée au volume réel pour que le garde-fou serve à quelque chose ;
- lorsqu'une requête ferait dépasser la limite, **elle est arrêtée avec une erreur** (et non mise en file d'attente).

---

## Ce qui est compté — et ce qui ne l'est pas

Le périmètre exact de ces quotas est une source fréquente de malentendus. Tout l'espace temporaire n'est pas pris en compte.

| ✅ Compté dans le quota | ❌ Non compté dans le quota |
|---|---|
| Fichiers temporaires de niveau SQL : `filesort`, espace temporaire de transaction, `ANALYZE`, `binlog_stmt_cache`, etc. | Fichiers temporaires **internes au moteur** utilisés pour `REPAIR`, `ALTER TABLE`, le pré-tri d'index, etc. |
| Tables temporaires **sur disque** créées pour résoudre un `SELECT`, un `UPDATE` multi-sources, etc. | |

La conséquence pratique est importante : **les fichiers de tri générés par un gros `ALTER TABLE` ne sont pas bornés par ces quotas.** Un administrateur qui s'attendrait à ce que `max_tmp_total_space_usage` protège contre un `ALTER TABLE` volumineux serait surpris. Pour le dimensionnement de ces opérations, on se reporte à [11.7](07-gestion-espace-disque.md) (marge nécessaire pour les reconstructions) et à [15.6](../15-performance-tuning/06-innodb-alter-copy-bulk.md) (`innodb_alter_copy_bulk`).

---

## Configuration

Les deux variables sont dynamiques et peuvent être définies dans le fichier de configuration ou à chaud.

Dans `my.cnf` (les suffixes `K`, `M`, `G` sont acceptés) :

```ini
[mariadb]
# Limite par session : aucune session ne peut dépasser 5 Go de temporaire disque
max_tmp_session_space_usage = 5G

# Limite globale : la somme sur toutes les sessions ne dépasse pas 50 Go
max_tmp_total_space_usage   = 50G
```

À chaud, la limite globale se définit au niveau `GLOBAL` ; la limite par session possède une valeur globale (qui sert de défaut aux nouvelles connexions) et peut être ajustée au niveau `SESSION` :

```sql
-- Limite globale (toutes connexions)
SET GLOBAL max_tmp_total_space_usage = 50 * 1024 * 1024 * 1024;

-- Valeur par défaut appliquée aux nouvelles sessions
SET GLOBAL max_tmp_session_space_usage = 5 * 1024 * 1024 * 1024;

-- Ajustement ponctuel pour la session courante (par exemple un batch légitime)
SET SESSION max_tmp_session_space_usage = 20 * 1024 * 1024 * 1024;
```

---

## Supervision

Deux variables d'état permettent de suivre la consommation et de dimensionner les limites :

```sql
-- Espace temporaire disque actuellement utilisé (toutes sessions)
SHOW GLOBAL STATUS LIKE 'tmp_space_used';

-- Pic d'espace temporaire observé
SHOW GLOBAL STATUS LIKE 'max_tmp_space_used';
```

Pour identifier précisément la session responsable d'une consommation anormale, la table `INFORMATION_SCHEMA.PROCESSLIST` expose une colonne `TMP_SPACE_USED` :

```sql
SELECT
    id          AS session_id,
    user        AS utilisateur,
    host        AS hote,
    command     AS commande,
    time        AS duree_s,
    tmp_space_used AS espace_temp_octets,
    LEFT(info, 80) AS requete
FROM information_schema.processlist
ORDER BY tmp_space_used DESC;
```

Cette requête est l'outil de diagnostic de référence : elle pointe immédiatement la connexion à l'origine d'une saturation imminente, ce qui permet de la cibler (`KILL`) ou d'identifier la requête à optimiser.

---

## Comportement à la limite et cas particuliers

### L'erreur retournée

Lorsqu'une requête atteint la limite, elle échoue avec une erreur indiquant que l'espace temporaire local est saturé — concrètement `ERROR 201 (HY000): Local temporary space limit reached` (en interne, `HA_ERR_LOCAL_TMP_SPACE_FULL` / `EE_LOCAL_TMP_SPACE_FULL`). Pour une table transactionnelle, l'instruction échoue proprement et peut être annulée sans laisser de données incohérentes.

### Tolérance au moment du commit

Une exception est prévue pour éviter des erreurs au moment de la validation : **le dernier vidage de `binlog_stmt_cache` lors d'un `COMMIT` ne déclenche pas d'erreur**, même si la limite est dépassée. Concrètement, une session peut temporairement dépasser sa limite d'un montant pouvant atteindre la taille de `binlog_stmt_cache_size`. Il faut en tenir compte dans le dimensionnement : la limite effective n'est pas strictement absolue à l'instant du commit.

### ⚠️ Interaction critique avec le journal binaire et les tables non transactionnelles

C'est le point le plus délicat de cette fonctionnalité, et il doit être compris avant tout déploiement en production avec réplication.

Si une entrée de journal binaire pour une requête est **plus grande que `binlog_stmt_cache_size`** et que la limite d'espace temporaire est atteinte **au moment de vider cette entrée sur disque**, alors :

- la requête est **arrêtée** ;
- le journal binaire **ne contient pas les dernières modifications** apportées à la table ;
- en conséquence, **la réplication s'arrête** sur les réplicas.

Ce risque concerne tout particulièrement les **tables non transactionnelles**, notamment les tables **Aria**, qui ne savent pas faire de rollback (sauf en cas de crash) : les modifications déjà appliquées ne peuvent pas être annulées, d'où la divergence avec le journal binaire.

Deux précautions permettent de s'en prémunir :

- pour les requêtes modifiant un grand nombre de lignes, utiliser `@@binlog_format=statement` (voir [11.5.2](05.2-formats-binlog.md)), afin que l'entrée de binlog reste compacte ;
- ne pas fixer de limites trop basses sur des serveurs combinant réplication et tables non transactionnelles. En dernier recours, positionner les deux variables à `0` désactive complètement le mécanisme de quota.

Ce comportement est à garder à l'esprit conjointement avec la configuration du journal binaire ([11.5](05-binary-logs.md)) et le diagnostic des erreurs de réplication ([13.7.3](../13-replication/07.3-erreurs-courantes.md)).

---

## Recommandations de dimensionnement

En synthèse, quelques principes guident un usage sain de ces quotas :

- **Activer les deux limites comme filet de sécurité**, même avec des valeurs généreuses : il est presque toujours préférable d'arrêter une requête fautive que de risquer une indisponibilité globale par saturation du disque.
- **Dimensionner la limite par session nettement en dessous de la limite globale**, de sorte qu'une session unique ne puisse pas monopoliser tout l'espace, tout en laissant passer les requêtes analytiques légitimes.
- **Intégrer la tolérance du commit** (`binlog_stmt_cache_size`) et la marge physique du volume dans le calcul : la limite globale doit rester confortablement inférieure à la capacité réelle du volume hébergeant `tmpdir` / `ibtmp1`.
- **Renforcer le quota par l'isolation physique** : héberger `tmpdir` et `ibtmp1` sur un volume dédié ([11.7](07-gestion-espace-disque.md)) fait que le quota logique et la limite physique se confortent mutuellement.
- **Surveiller `max_tmp_space_used`** dans la durée pour ajuster les valeurs au plus près de la réalité de la charge.
- **Être prudent sur les charges Aria répliquées** : combiner réplication et tables non transactionnelles impose des limites conservatrices et, le cas échéant, le format `statement` pour les écritures massives.
- **Ne pas confondre garde-fou et correctif** : ces quotas empêchent l'incident, mais ne remplacent pas l'optimisation des requêtes (voir [Partie 3](../partie-03-index-transactions-performance.md) et le chapitre [5](../05-index-et-performance/README.md)). Une requête arrêtée par dépassement de quota signale un problème à corriger en amont.

En combinant le **contrôle logique** apporté par ces deux variables et le **dimensionnement physique** détaillé en [11.7](07-gestion-espace-disque.md), on met en place une véritable défense en profondeur contre l'épuisement du disque par l'espace temporaire — l'une des causes les plus courantes, et les plus évitables, d'indisponibilité d'un serveur MariaDB.

⏭️ [Monitoring et métriques importantes](/11-administration-configuration/09-monitoring-metriques.md)

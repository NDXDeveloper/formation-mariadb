🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.12 — Extension TIMESTAMP 2038→2106 (problème Y2038 résolu, depuis 11.8)

Le **problème de l'an 2038** — surnommé *Y2038* ou *Epochalypse* — est l'équivalent du bug de l'an 2000 pour les systèmes qui stockent le temps en secondes depuis l'epoch Unix dans un entier 32 bits signé. MariaDB l'a neutralisé pour le type `TIMESTAMP` en repoussant sa borne supérieure de **2038 à 2106**, à partir de la LTS **11.8** (et donc en 12.3). Cette section explique en quoi consiste le problème, pourquoi `TIMESTAMP` était concerné là où `DATETIME` ne l'était pas, la solution retenue par MariaDB, ses limites, le cas particulier des tables temporelles, et l'impact sur la migration. Elle clôt le chapitre 11 consacré à l'administration et à la configuration.

---

## Le problème de l'an 2038 (Y2038)

Le type `TIMESTAMP` a longtemps été stocké comme un entier **32 bits signé** représentant le nombre de secondes écoulées depuis l'epoch Unix, fixé au **1ᵉʳ janvier 1970 à 00:00:00 UTC**. Or, la plus grande valeur représentable par un entier 32 bits signé est 2 147 483 647 ; ce compteur atteint son maximum le **19 janvier 2038 à 03:14:07 UTC**. Au-delà, l'ajout d'une seconde provoque un dépassement de capacité : la valeur « déborde » et les dates deviennent incohérentes.

Le risque n'est pas théorique ni lointain. Tout système déployé aujourd'hui mais destiné à fonctionner au-delà de 2038 (politiques de rétention de documents, suivi de prescriptions, planification) a besoin d'une solution. Surtout, le problème se manifeste **dès maintenant** dès qu'on manipule une date **postérieure à 2038** — une échéance fixée en 2040, par exemple, dépasse déjà l'ancienne borne. Comme `TIMESTAMP` était de fait limité à la plage 1970–2038 (les valeurs négatives étant rejetées, l'usage réel n'était que de 31 bits), cette limite pouvait être atteinte bien avant l'année 2038 elle-même.

---

## `TIMESTAMP` et `DATETIME` : pourquoi `TIMESTAMP` était concerné

MariaDB propose deux types principaux pour stocker une date et une heure, dont les caractéristiques diffèrent fondamentalement (voir aussi [2.2.3](../02-bases-du-sql/02.3-types-temporels.md)) :

| | `TIMESTAMP` | `DATETIME` |
|---|---|---|
| Stockage | secondes depuis l'epoch (UTC), 4 octets | valeur calendaire littérale, ~8 octets |
| Fuseau horaire | **converti** selon le fuseau de la session (gestion correcte de l'heure d'été) | aucune conversion |
| Plage | 1970 → **2106** (depuis l'extension) | 1000 → 9999 |
| Concerné par Y2038 | oui — **résolu jusqu'en 2106** | non (jamais concerné) |

C'est précisément le mode de stockage de `TIMESTAMP` — des secondes depuis l'epoch — qui en faisait la cible du problème Y2038. En contrepartie, ce stockage en UTC lui confère sa qualité la plus précieuse : il est **conscient du fuseau horaire** et reste correct au fil des changements d'heure. `DATETIME`, qui stocke une valeur calendaire brute sans conversion, n'a jamais été affecté par Y2038, mais ne gère pas les fuseaux horaires.

Avant l'extension, échapper à 2038 imposait donc un compromis insatisfaisant : basculer vers `DATETIME` (et perdre la justesse vis-à-vis des fuseaux), ou stocker un entier `BIGINT` d'epoch converti dans le code applicatif (au prix d'une complexité et d'un risque d'erreur accrus). L'extension de MariaDB élimine ce dilemme : on conserve la justesse de `TIMESTAMP` **tout en dépassant 2038**.

---

## La solution de MariaDB : la plage 32 bits non signée

La solution retenue (MDEV-32188) est élégante : réinterpréter la valeur 32 bits existante comme un entier **non signé** plutôt que signé. Puisque les valeurs négatives étaient déjà rejetées, passer en non signé **double la plage utile** sans changer la taille du champ. Concrètement :

- la **nouvelle borne haute** devient **2106-02-07 06:28:15 UTC** (contre 2038-01-19 03:14:07 UTC) ;
- le **format de stockage n'est pas modifié** : `TIMESTAMP` reste sur 4 octets. Les nouvelles tables restent lisibles par d'anciens serveurs MariaDB **tant que les valeurs restent dans l'ancienne plage** (≤ 2038) ;
- les **fonctions associées** (`UNIX_TIMESTAMP()`, `FROM_UNIXTIME()`) prennent en charge la plage étendue.

À titre d'illustration, les deux bornes s'observent ainsi (la valeur affichée dépend du fuseau de session, propriété intrinsèque de `TIMESTAMP`) :

```sql
-- Ancienne borne haute (32 bits signés)
SELECT FROM_UNIXTIME(2147483647);   -- 2038-01-19 03:14:07 (en UTC)

-- Nouvelle borne haute (32 bits non signés, plateformes 64 bits)
SELECT FROM_UNIXTIME(4294967295);   -- 2106-02-07 06:28:15 (en UTC)
```

L'intérêt de cette approche est qu'elle ne requiert **ni changement de schéma, ni coût de stockage supplémentaire** : il s'agit simplement d'une interprétation plus large de la même donnée.

---

## Les limites de l'extension

Deux limites importantes doivent être connues :

1. **La borne basse reste inchangée.** L'extension ne concerne que la borne **supérieure** ; la borne inférieure demeure le 1ᵉʳ janvier 1970 (l'epoch). La plage ne peut pas être étendue vers le passé, la valeur d'epoch 0 ayant une signification particulière. Pour des dates antérieures à 1970, ou pour une plage beaucoup plus large, on utilise `DATETIME`.
2. **Plateformes 64 bits uniquement.** L'extension est prise en charge sur les systèmes 64 bits ; elle ne s'applique pas aux plateformes 32 bits, aujourd'hui devenues rares.

Sur la **datation** : à l'image du changement de jeu de caractères par défaut ([11.11](11-charset-utf8mb4-uca14.md)), cette extension a été introduite dans la série rolling avec la **11.5.1** (2024), puis consolidée dans la **LTS 11.8** (2025) — c'est donc à partir de cette LTS qu'elle s'impose sur le canal LTS, et elle est présente en 12.3.

---

## Cas particulier : les tables temporelles (system-versioned)

C'est le point de vigilance majeur de cette évolution. Les **tables temporelles** versionnées par le système (voir [18.2](../18-fonctionnalites-avancees/02-system-versioned-tables.md)) utilisent une colonne `row_end` dont la valeur « fin des temps » — celle qui marque une ligne comme toujours en vigueur — correspondait historiquement à la valeur maximale de `TIMESTAMP`, c'est-à-dire la borne 2038. Avec l'extension, ce marqueur de fin des temps doit passer à la **valeur fondée sur 2106**.

La conséquence pratique est directe sur les mises à niveau : une mise à jour de MariaDB se déroule généralement en quelques secondes, **sauf lorsque des tables temporelles sont présentes**, car leurs horodatages internes `row_end` doivent être migrés de la base 2038 vers la base 2106. Cette opération a connu des cas particuliers documentés (mises à niveau « à chaud » comme par export/rechargement). Il faut donc **planifier spécifiquement** la mise à niveau des tables temporelles ; ce sujet est traité dans la section de migration dédiée [19.9](../19-migration-compatibilite/09-migration-system-versioned-tables.md), et le fonctionnement de ces tables en [18.2](../18-fonctionnalites-avancees/02-system-versioned-tables.md).

---

## Migration et compatibilité

Au-delà des tables temporelles, l'extension a des implications de compatibilité à anticiper (voir le chapitre [19](../19-migration-compatibilite/README.md)).

**Réplication vers des serveurs plus anciens.** Une valeur postérieure à 2038 écrite sur un primaire en 12.3 ne peut pas être représentée par un réplica MariaDB (ou MySQL) plus ancien : la réplication se rompt. Les nouvelles tables ne restent compatibles avec d'anciens serveurs que tant que leurs valeurs demeurent dans l'ancienne plage. Il existe par ailleurs une interaction avec le format temporel du journal binaire (`mysql56_temporal_format`) susceptible d'altérer l'affichage des horodatages supérieurs à 2³¹ (voir [13](../13-replication/README.md) et [11.5](05-binary-logs.md)).

**Interopérabilité avec MySQL.** MySQL n'a **pas** corrigé ce problème : son type `TIMESTAMP` reste plafonné à 2038. MySQL 8.0.28 et ultérieurs ont étendu certaines **fonctions** (`FROM_UNIXTIME()`, `UNIX_TIMESTAMP()`) au 64 bits, mais pas le type lui-même. Il en résulte un avantage net pour MariaDB et une considération de migration concrète : une migration **depuis MySQL vers MariaDB gagne** la plage jusqu'en 2106, mais une réplication **de MariaDB vers MySQL** se heurterait au plafond 2038 de MySQL pour toute valeur le dépassant (voir [19.1](../19-migration-compatibilite/01-migration-depuis-mysql.md)).

**Outillage.** Certains outils et interfaces plus anciens continuent d'indiquer que `TIMESTAMP` s'arrête en 2038 et peuvent ne pas interpréter correctement les valeurs étendues. Il convient de vérifier le support de la chaîne d'outils utilisée.

En synthèse : en 12.3, `TIMESTAMP` est sûr jusqu'en 2106 tout en conservant sa justesse vis-à-vis des fuseaux horaires ; pour des horizons au-delà de 2106 ou des dates antérieures à 1970, on recourt à `DATETIME` (ou à un `BIGINT` d'epoch) ; on traite explicitement les tables temporelles lors des mises à niveau ; et l'on ne présume pas qu'un réplica MySQL partage cette extension.

---

Avec cette extension, MariaDB neutralise l'« Epochalypse » pour le type `TIMESTAMP` à coût de stockage nul, permettant aux applications de conserver des horodatages conscients du fuseau horaire bien au-delà de 2038 — une correction discrète mais lourde de conséquences, introduite dans la série rolling 11.5.1, consolidée dans la LTS 11.8 et reprise en 12.3. Les principaux points d'attention demeurent la **mise à niveau des tables temporelles** et la **compatibilité inter-moteurs** avec MySQL. Pour le détail des types temporels, on se reportera à [2.2.3](../02-bases-du-sql/02.3-types-temporels.md), et pour les tables temporelles versionnées à [18.2](../18-fonctionnalites-avancees/02-system-versioned-tables.md).

⏭️ [Sauvegarde et Restauration](/12-sauvegarde-restauration/README.md)

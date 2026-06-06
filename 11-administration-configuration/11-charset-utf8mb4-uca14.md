🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.11 — Charset par défaut : utf8mb4 avec collations UCA 14.0.0 (depuis 11.8)

Pendant des années, MariaDB a utilisé par défaut le jeu de caractères `latin1` et la collation `latin1_swedish_ci` — des choix hérités, adaptés aux langues d'Europe de l'Ouest mais inaptes à représenter l'ensemble des écritures du monde ou les emojis. À partir de la LTS **11.8** (et donc en 12.3), les valeurs par défaut deviennent le jeu de caractères **`utf8mb4`** et la collation **`utf8mb4_uca1400_ai_ci`**, fondée sur l'algorithme de collation Unicode 14.0.0. Cette section explique la distinction entre jeu de caractères et collation, ce qui a changé et quand, la famille de collations `uca1400`, le piège de l'alias `utf8`, les implications de `utf8mb4` sur le stockage et les index, la hiérarchie de configuration, et l'impact sur la migration.

---

## Jeu de caractères et collation : deux notions distinctes

Ces deux concepts sont souvent confondus, alors qu'ils répondent à des questions différentes :

- le **jeu de caractères** (*character set*, ou *charset*) définit comment les caractères sont **encodés** en octets. C'est la table de correspondance entre les caractères et leur représentation binaire.
- la **collation** définit les règles de **comparaison et de tri** des caractères **au sein d'un jeu de caractères donné**. Elle détermine, par exemple, si `a` est égal à `A`, si `é` est égal à `e`, et l'ordre relatif des caractères.

Chaque jeu de caractères possède une ou plusieurs collations, dont l'une est sa collation par défaut. À l'inverse, une collation appartient à **un seul** jeu de caractères : c'est pourquoi son nom commence toujours par le nom du charset (`utf8mb4_uca1400_ai_ci` est une collation du charset `utf8mb4`).

---

## `utf8mb4` face au `utf8` historique (`utf8mb3`)

Le jeu de caractères `utf8mb4` est l'implémentation **complète** d'UTF-8 : il encode chaque caractère sur 1 à 4 octets et couvre l'intégralité des points de code Unicode, y compris les caractères supplémentaires (plans astraux) et donc les **emojis** ainsi que de nombreuses extensions CJK.

À l'opposé, l'ancien jeu `utf8` de MariaDB — désormais nommé **`utf8mb3`** — est limité à 3 octets par caractère et ne couvre que le plan multilingue de base (BMP) : ni emojis, ni de nombreux caractères d'écritures asiatiques.

> ⚠️ **Le piège à connaître absolument** : depuis MariaDB 10.6.1, l'alias **`utf8` désigne `utf8mb3`**, et non `utf8mb4` (comportement contrôlé par le drapeau `UTF8_IS_UTF8MB3` de la variable `old_mode`). Autrement dit, même si le jeu de caractères par défaut — c'est-à-dire celui appliqué *en l'absence de spécification* — est désormais `utf8mb4`, écrire explicitement `CHARSET=utf8` vous donne toujours le `utf8mb3` à 3 octets, **sans** support des emojis. La règle pratique est donc d'écrire **toujours `utf8mb4` explicitement**, jamais `utf8`.

---

## Le changement de défaut : de `latin1` à `utf8mb4`

Le tableau ci-dessous résume la bascule :

| | Avant | À partir de la LTS 11.8 (et en 12.3) |
|---|---|---|
| Jeu de caractères par défaut | `latin1` | `utf8mb4` |
| Collation par défaut | `latin1_swedish_ci` | `utf8mb4_uca1400_ai_ci` |

Ce nouveau défaut apporte trois bénéfices majeurs : une **compatibilité globale** sans configuration supplémentaire (le défaut `latin1` ne convenait qu'aux langues d'Europe de l'Ouest), la **prise en charge des caractères supplémentaires** comme les emojis, et un **tri Unicode correct** — par exemple, le caractère allemand « ß » est correctement comparé comme égal à « ss ».

Sur la **datation** : ce changement a en réalité été introduit dans la série rolling avec la **11.6**, puis consolidé dans la **LTS 11.8** — c'est à partir de cette LTS qu'il s'impose sur le canal LTS, et il est naturellement présent en 12.3. Techniquement, la bascule s'appuie sur la variable système `character_set_collations` (introduite en 11.2), qui définit la collation par défaut associée à un jeu de caractères.

Point essentiel de **portée** : ce changement n'affecte que les **nouveaux objets** (bases, tables, colonnes) créés **sans spécification explicite** de jeu de caractères. Les données et schémas existants ne sont **pas** modifiés par une mise à jour. Un serveur migré peut donc cohabiter avec des tables `latin1` héritées et des tables `utf8mb4` récentes — voir la section sur la migration plus bas.

---

## Les collations UCA 14.0.0 (famille `uca1400`)

**UCA** désigne l'*Unicode Collation Algorithm*, l'algorithme standard de tri d'Unicode. La famille de collations `uca1400` est fondée sur **Unicode 14.0.0** et a été ajoutée à MariaDB dès la **10.10.1** (184 collations). Elle se reconnaît à l'**infixe `_uca1400_`** dans les noms de collations, et s'applique à tous les jeux de caractères Unicode (`utf8mb3`, `utf8mb4`, `utf16`, `utf32`).

Sa structure est régulière : une collation **neutre** (« racine ») accompagnée d'une vingtaine de collations **adaptées** à des langues spécifiques ; pour chaque variante, huit combinaisons sont disponibles, croisant accent (`ai`/`as`), casse (`ci`/`cs`) et gestion des espaces (`pad`/`nopad`).

L'apport décisif de cette famille tient à ses **contractions intégrées** (939, définies dans la *Default Unicode Collation Element Table*, ou DUCET), absentes des anciennes collations fondées sur UCA 4.0.0 ou 5.2.0. Concrètement, certaines langues (comme le thaï) n'ont plus besoin de collation spécifique et fonctionnent correctement avec la collation racine. Le tri s'effectue sur trois niveaux de poids : **primaire** (la lettre de base), **secondaire** (les accents) et **tertiaire** (la casse).

### Décoder les suffixes d'une collation

Le nom d'une collation `uca1400` suit le schéma `<charset>_uca1400_[langue_]<ai|as>_<ci|cs>[_nopad]`. Les suffixes se lisent ainsi :

| Suffixe | Signification |
|---|---|
| `ai` | **A**ccent-**i**nsensible : `a` est égal à `á` |
| `as` | **A**ccent-**s**ensible : `a` est différent de `á` |
| `ci` | **C**asse-**i**nsensible : `a` est égal à `A` |
| `cs` | **C**asse-**s**ensible : `a` est différent de `A` |
| `nopad` | Les espaces de fin sont **significatifs** (à défaut : *PAD*, espaces de fin ignorés à la comparaison) |
| *(langue)* | Présence d'un code langue = collation adaptée à cette langue ; absence = collation racine neutre |

La collation par défaut `utf8mb4_uca1400_ai_ci` est donc **accent-insensible, casse-insensible, de type PAD et neutre** — un comportement raisonnable pour la plupart des applications. Pour des besoins différents, on peut choisir `utf8mb4_uca1400_as_cs` (sensible aux accents et à la casse) ou encore la collation binaire `utf8mb4_bin`, qui compare octet par octet — non pertinente pour le texte humain, mais utile pour du JSON ou des données machine.

---

## Implications de `utf8mb4` sur le stockage et les index

Puisque `utf8mb4` peut utiliser jusqu'à **4 octets par caractère** (contre 1 pour `latin1`), quelques effets méritent attention :

- **Longueur des préfixes d'index** : sur les formats de ligne modernes (`DYNAMIC`, qui est le format par défaut), la longueur maximale d'un préfixe d'index InnoDB est de **3072 octets**, ce qui permet d'indexer par exemple un `VARCHAR(768)` en `utf8mb4` (768 × 4 = 3072). L'ancienne erreur « *Specified key was too long; max key length is 767 bytes* » ne concerne que les anciens formats `COMPACT`/`REDUNDANT` ; elle ne se rencontre donc plus guère, sauf en migrant des schémas très anciens.
- **Capacité des `VARCHAR`** : pour un budget d'octets donné (lié à la taille de ligne), le nombre de caractères stockables est mécaniquement réduit par rapport à `latin1`.
- **Mémoire des opérations temporaires** : les tables temporaires internes et les buffers de tri se dimensionnent sur la longueur maximale possible en octets, ce qui peut légèrement accroître la mémoire consommée par certaines opérations (à rapprocher de la gestion de l'espace temporaire, [11.8](08-controle-espace-temporaire.md)).

Ces points restent, dans l'immense majorité des cas, sans conséquence pratique sur une installation moderne ; il est néanmoins utile de les connaître.

---

## La hiérarchie de configuration

Les jeux de caractères et collations se définissent à plusieurs niveaux, chacun héritant du niveau supérieur sauf surcharge explicite : du **serveur** (`character_set_server`, `collation_server`) à la **base de données**, puis à la **table**, jusqu'à la **colonne**. S'y ajoute la configuration de la **connexion** client-serveur (`character_set_client`, `character_set_connection`, `character_set_results`), que l'on règle généralement d'un seul geste avec `SET NAMES`.

Configuration au niveau serveur :

```ini
[mariadb]
character_set_server     = utf8mb4
collation_server         = utf8mb4_uca1400_ai_ci
# Optionnel : collation par défaut associée à un charset
character_set_collations = utf8mb4=uca1400_ai_ci
```

Spécification explicite à la création d'objets (recommandée en contexte de migration) :

```sql
CREATE DATABASE boutique
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_uca1400_ai_ci;

CREATE TABLE produits (
  libelle VARCHAR(255)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

Réglage de la connexion et inspection de l'existant :

```sql
-- Fixe character_set_client, _connection et _results en une instruction
SET NAMES utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- Vérifier les défauts du serveur et lister les collations
SELECT @@character_set_server, @@collation_server;
SHOW COLLATION WHERE Charset = 'utf8mb4' AND Collation LIKE '%uca1400%';
```

---

## Impact sur la migration et la compatibilité

Ce changement de défaut a des conséquences directes sur les scénarios de migration et de réplication, traités plus largement au chapitre [19](../19-migration-compatibilite/README.md).

**Cohabitation après mise à jour.** Comme indiqué, la mise à jour ne convertit pas les données existantes : le nouveau défaut ne s'applique qu'aux objets créés sans spécification explicite. Un serveur post-migration peut donc mêler des tables `latin1` anciennes et des tables `utf8mb4` nouvelles. Pour éviter toute ambiguïté, il est conseillé d'être **explicite** sur le charset et la collation lors de la création d'objets pendant une phase de migration.

**Réplication vers des serveurs plus anciens.** Une collation comme `utf8mb4_uca1400_ai_ci` n'est pas connue des versions antérieures de MariaDB (ni de MySQL), ni de certains outils plus anciens — d'où des erreurs du type « *Unknown collation: 'utf8mb4_uca1400_ai_ci'* ». Lorsqu'un serveur primaire en 12.3 réplique du DDL vers un réplica plus ancien, cela peut **rompre la réplication**. Les parades consistent à spécifier une collation comprise des deux côtés, ou à mettre à niveau les réplicas en premier (voir [13](../13-replication/README.md)).

**Interopérabilité avec MySQL.** MySQL 8.0 utilise par défaut `utf8mb4` mais avec la collation `utf8mb4_0900_ai_ci` (fondée sur UCA 9.0.0). Les deux écosystèmes convergent donc sur `utf8mb4`, mais leurs collations par défaut diffèrent. Pour faciliter la réplication et la migration depuis MySQL 8, MariaDB a ajouté (à partir de la 11.4.5) des **collations de compatibilité UCA 9.0.0** mappées sur les collations `uca1400`. Ce point est développé en [19.1](../19-migration-compatibilite/01-migration-depuis-mysql.md) et [19.1.1](../19-migration-compatibilite/01.1-compatibilite-mysql-mariadb.md).

**Cas particulier.** Les colonnes d'`INFORMATION_SCHEMA` conservent la collation `utf8mb3_general_ci` et ne sont pas concernées par ce changement.

En synthèse : sur un déploiement neuf en 12.3, on adopte sans réserve les nouveaux défauts ; en contexte de migration, on reste **explicite** sur le charset et la collation et l'on **vérifie la compatibilité** des réplicas et des outils.

---

Avec `utf8mb4` et la collation UCA 14.0.0 par défaut, MariaDB se met au niveau des applications modernes, internationales et compatibles emojis, tout en offrant un tri Unicode correct dès l'installation — une évolution attendue de longue date, consolidée dans la LTS 11.8 et reprise en 12.3. Les deux principaux points de vigilance restent le **piège de l'alias `utf8`** (qui désigne toujours `utf8mb3`) et la **compatibilité inter-versions et inter-moteurs** lors des migrations. La gestion de la nouvelle borne temporelle de `TIMESTAMP`, autre évolution de cette même génération, est traitée dans la section suivante ([11.12](12-extension-timestamp-2106.md)).

⏭️ [Extension TIMESTAMP 2038→2106 (problème Y2038 résolu, depuis 11.8)](/11-administration-configuration/12-extension-timestamp-2106.md)

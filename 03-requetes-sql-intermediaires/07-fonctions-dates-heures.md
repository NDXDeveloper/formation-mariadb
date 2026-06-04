🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.7 Fonctions de dates et heures

> **Chapitre 3 : Requêtes SQL Intermédiaires** · Section 3.7 (introduction)  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Le temps est partout dans les bases de données : dates de commande, horodatages d'événements, échéances, durées. MariaDB offre un riche ensemble de fonctions pour **obtenir** la date courante, **extraire** des composants, **calculer** avec des intervalles, et **formater** ou **analyser** des chaînes temporelles. Cette section présente les fonctions **natives** de MariaDB.

> 🆕 MariaDB 12.x **enrichit** par ailleurs la **compatibilité Oracle** des conversions — `TO_DATE` (nouveau en 12.3), `TO_NUMBER`, `TRUNC`, et le modificateur `FM` de `TO_CHAR` —, qui font l'objet de la [section 3.7.1](07.1-fonctions-oracle.md). La présente introduction se concentre sur les fonctions natives ; les équivalents Oracle sont signalés au passage et détaillés ensuite.

---

## Rappel : les types temporels

Les fonctions de cette section s'appliquent aux types vus en [section 2.2.3](../02-bases-du-sql/02.3-types-temporels.md) : `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `YEAR`. Deux distinctions méritent un rappel :

- **`DATETIME` vs `TIMESTAMP`** : `TIMESTAMP` est sensible au fuseau horaire (stocké en UTC, converti selon `time_zone` à la lecture), tandis que `DATETIME` est stocké tel quel. Choisissez `TIMESTAMP` pour un instant absolu, `DATETIME` pour une date « murale ».
- Depuis la 11.8, la plage de `TIMESTAMP` s'étend **au-delà de 2038, jusqu'en 2106** (problème Y2038 résolu, voir [section 11.12](../11-administration-configuration/12-extension-timestamp-2106.md)).

---

## Date et heure courantes

| Fonction | Renvoie |
|----------|---------|
| `NOW()` / `CURRENT_TIMESTAMP` | date et heure courantes (`DATETIME`) |
| `CURDATE()` / `CURRENT_DATE` | date courante (`DATE`) |
| `CURTIME()` / `CURRENT_TIME` | heure courante (`TIME`) |
| `UTC_TIMESTAMP()`, `UTC_DATE()`, `UTC_TIME()` | équivalents en UTC |
| `UNIX_TIMESTAMP()` / `FROM_UNIXTIME(n)` | conversion vers/depuis l'epoch Unix |

```sql
SELECT NOW() AS maintenant, CURDATE() AS aujourd_hui, CURTIME() AS heure;
```

> ℹ️ **`NOW()` vs `SYSDATE()`.** `NOW()` renvoie l'heure du **début de l'instruction** : elle est donc constante au sein d'une même requête. `SYSDATE()`, elle, renvoie l'heure de **son exécution effective**, et peut varier d'une ligne à l'autre dans une longue requête. Préférez `NOW()` dans la quasi-totalité des cas.

---

## Extraire des composants

De nombreuses fonctions isolent une partie d'une date :

| Fonction | Exemple de résultat pour `2026-06-03` |
|----------|----------------------------------------|
| `YEAR()`, `MONTH()`, `DAY()` | 2026, 6, 3 |
| `HOUR()`, `MINUTE()`, `SECOND()` | (sur la partie heure) |
| `QUARTER()`, `WEEK()`, `DAYOFYEAR()` | 2, … , 154 |
| `DAYOFWEEK()`, `WEEKDAY()` | indices du jour de semaine |
| `MONTHNAME()`, `DAYNAME()` | `June`, `Wednesday` |
| `LAST_DAY()` | `2026-06-30` (dernier jour du mois) |

```sql
SELECT
    id,
    date_cmd,
    YEAR(date_cmd)      AS annee,
    MONTH(date_cmd)     AS mois,
    MONTHNAME(date_cmd) AS nom_mois
FROM commande;
```

La syntaxe standard **`EXTRACT(unité FROM date)`** offre une alternative portable :

```sql
SELECT EXTRACT(YEAR FROM '2026-06-03') AS annee;   -- 2026
```

> ℹ️ **Langue des noms.** `MONTHNAME` et `DAYNAME` renvoient les noms dans la langue définie par la variable `lc_time_names` (`en_US` par défaut → `June`, `Wednesday`). Pour des noms français, on positionne `SET lc_time_names = 'fr_FR';` (voir les [variables de session, section 11.2](../11-administration-configuration/02-variables-systeme-session.md)).

---

## Calculer avec les dates

### Ajouter ou retrancher un intervalle

`DATE_ADD(date, INTERVAL n unité)` et `DATE_SUB(...)` décalent une date. La syntaxe `date + INTERVAL n unité` est équivalente et plus concise. Les unités vont de `SECOND` à `YEAR` (`DAY`, `WEEK`, `MONTH`, `QUARTER`…).

```sql
SELECT
    date_cmd,
    DATE_ADD(date_cmd, INTERVAL 30 DAY) AS echeance,
    date_cmd + INTERVAL 1 MONTH         AS dans_un_mois
FROM commande;
```

Pour la commande du `2026-01-05`, l'échéance à 30 jours tombe le `2026-02-04`, et un mois plus tard le `2026-02-05`.

### Mesurer un écart entre deux dates

| Fonction | Renvoie | Ordre des arguments |
|----------|---------|---------------------|
| `DATEDIFF(d1, d2)` | l'écart en **jours** (`d1 − d2`) | fin, début |
| `TIMESTAMPDIFF(unité, d1, d2)` | l'écart dans l'**unité choisie** (unités complètes) | unité, **début**, fin |

```sql
SELECT
    DATEDIFF('2026-03-01', '2026-01-05')              AS jours,
    TIMESTAMPDIFF(MONTH, '2026-01-05', '2026-03-01')  AS mois;
```

| jours | mois |
|-------|------|
| 55 | 1 |

> ⚠️ **Attention à l'ordre des arguments**, source d'erreurs fréquente : `DATEDIFF` attend **(fin, début)** et renvoie `d1 − d2`, tandis que `TIMESTAMPDIFF` attend **(unité, début, fin)**. Les deux conventions sont inversées. Notez aussi que `TIMESTAMPDIFF` compte les unités **complètes** : du 5 janvier au 1ᵉʳ mars, il n'y a qu'**un** mois entier révolu.

---

## Formater et analyser

C'est ici qu'intervient la distinction la plus importante avec Oracle.

### DATE_FORMAT : date → chaîne

`DATE_FORMAT(date, format)` produit une chaîne à partir de spécificateurs `%` :

```sql
SELECT DATE_FORMAT('2026-06-03', '%d/%m/%Y') AS date_fr;   -- '03/06/2026'
```

Principaux spécificateurs :

| Code | Signification | Code | Signification |
|------|---------------|------|---------------|
| `%Y` | année sur 4 chiffres | `%y` | année sur 2 chiffres |
| `%m` | mois `01`–`12` | `%c` | mois `1`–`12` |
| `%d` | jour `01`–`31` | `%e` | jour `1`–`31` |
| `%H` | heure `00`–`23` | `%h` | heure `01`–`12` |
| `%i` | minutes | `%s` | secondes |
| `%p` | `AM`/`PM` | `%j` | jour de l'année |
| `%W` | nom du jour | `%M` | nom du mois |
| `%a` | jour abrégé | `%b` | mois abrégé |

```sql
SELECT DATE_FORMAT(NOW(), '%W %e %M %Y');   -- ex. 'Wednesday 3 June 2026'
```

(Les codes `%W` et `%M` suivent également `lc_time_names` pour la langue.)

### STR_TO_DATE : chaîne → date

`STR_TO_DATE(chaîne, format)` est l'opération **inverse** : elle analyse une chaîne selon un format pour produire une date — indispensable à l'import de données :

```sql
SELECT STR_TO_DATE('03/06/2026', '%d/%m/%Y') AS d;   -- 2026-06-03
```

> 🆕 **Équivalents Oracle.** Sous Oracle, ces deux opérations s'écrivent `TO_CHAR` (date → chaîne) et `TO_DATE` (chaîne → date), avec une **syntaxe de format différente** (par exemple `'DD/MM/YYYY'` au lieu de `'%d/%m/%Y'`). MariaDB 12.x les prend en charge en mode de compatibilité : c'est l'objet de la [section 3.7.1](07.1-fonctions-oracle.md).

---

## Application : regrouper par période

Combinées au `GROUP BY` ([section 3.2](02-regroupement-donnees.md)), ces fonctions permettent d'agréger par période. Pour un nombre de commandes par mois :

```sql
SELECT
    DATE_FORMAT(date_cmd, '%Y-%m') AS mois,
    COUNT(*)                       AS nb
FROM commande
GROUP BY DATE_FORMAT(date_cmd, '%Y-%m')
ORDER BY mois;
```

| mois | nb |
|------|----|
| 2026-01 | 2 |
| 2026-02 | 2 |
| 2026-03 | 1 |

Regrouper sur `DATE_FORMAT(date_cmd, '%Y-%m')` (l'année-mois) plutôt que sur la date complète rassemble toutes les commandes d'un même mois.

---

## À retenir

- `NOW()`/`CURRENT_TIMESTAMP`, `CURDATE()`, `CURTIME()` donnent l'instant courant ; `NOW()` est figée pour toute l'instruction (contrairement à `SYSDATE()`).
- Extraire : `YEAR`/`MONTH`/`DAY`…, `EXTRACT(unité FROM …)`, `LAST_DAY` ; `MONTHNAME`/`DAYNAME` dépendent de `lc_time_names` pour la langue.
- Calculer : `DATE_ADD`/`DATE_SUB` ou `+ INTERVAL n unité` ; `DATEDIFF` (en jours, **fin, début**) et `TIMESTAMPDIFF` (unité au choix, **unité, début, fin**) — ordres d'arguments inversés.
- Formater/analyser : **`DATE_FORMAT`** (date → chaîne, spécificateurs `%`) et **`STR_TO_DATE`** (chaîne → date).
- `TIMESTAMP` est sensible au fuseau et désormais valide jusqu'en 2106 ([section 11.12](../11-administration-configuration/12-extension-timestamp-2106.md)) ; `DATETIME` ne l'est pas.
- Les équivalents Oracle `TO_CHAR`/`TO_DATE`/`TRUNC`/`TO_NUMBER` sont traités en [section 3.7.1](07.1-fonctions-oracle.md) 🆕.

---

## Navigation

- ⬅️ Section précédente : [3.6 — Fonctions de chaînes de caractères](06-fonctions-chaines.md)
- ➡️ Section suivante : [3.7.1 — Fonctions de compatibilité Oracle (TO_DATE, TRUNC, TO_NUMBER, TO_CHAR avec format FM)](07.1-fonctions-oracle.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Fonctions de compatibilité Oracle (TO_DATE, TRUNC, TO_NUMBER, TO_CHAR avec format FM)](/03-requetes-sql-intermediaires/07.1-fonctions-oracle.md)

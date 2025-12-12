🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.7 Fonctions de dates et heures

> **Niveau** : Intermédiaire
> **Durée estimée** : 2-3 heures
> **Prérequis** : Sections 3.1 à 3.6, compréhension des types temporels

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Maîtriser les types de données temporelles (DATE, TIME, DATETIME, TIMESTAMP)
- Extraire des composants de dates (année, mois, jour, heure)
- Effectuer des calculs et comparaisons de dates
- Formater des dates pour l'affichage
- Convertir des chaînes en dates
- Gérer les fuseaux horaires
- Calculer des durées et intervalles
- Résoudre des problèmes métier liés au temps
- Optimiser les requêtes avec filtres temporels

---

## Introduction

Les **fonctions de dates et heures** sont essentielles pour gérer les données temporelles. Elles permettent de manipuler, calculer, formater et analyser des informations liées au temps.

### Types de données temporelles en MariaDB

| Type | Format | Précision | Plage | Usage |
|------|--------|-----------|-------|-------|
| **DATE** | YYYY-MM-DD | Jour | 1000-01-01 à 9999-12-31 | Dates de naissance, échéances |
| **TIME** | HH:MM:SS | Seconde | -838:59:59 à 838:59:59 | Durées, horaires |
| **DATETIME** | YYYY-MM-DD HH:MM:SS | Seconde | 1000-01-01 à 9999-12-31 | Événements horodatés |
| **TIMESTAMP** | YYYY-MM-DD HH:MM:SS | Seconde | 1970-01-01 à 2038-01-19 | Avec fuseau horaire |
| **YEAR** | YYYY | Année | 1901 à 2155 | Années uniquement |

### Cas d'usage courants

```
📅 ANALYSES TEMPORELLES    ⏱️ DURÉES & INTERVALLES
   Ventes par mois            Temps de traitement
   Tendances annuelles        Âges, anciennetés
   Saisonnalité               Délais de livraison

📊 RAPPORTS                 🔔 ALERTES & RAPPELS
   Périodes comptables        Échéances proches
   KPI hebdomadaires          Abonnements expirés
   Comparaisons N vs N-1      Contrats à renouveler

🗓️ PLANIFICATION           🌍 INTERNATIONAL
   Calendriers                Fuseaux horaires
   Horaires de travail        Conversions UTC
   Disponibilités             Formats locaux
```

---

## Obtenir la date et l'heure actuelles

### NOW(), CURDATE(), CURTIME()

```sql
NOW()        -- Date et heure actuelles (DATETIME)
CURDATE()    -- Date actuelle uniquement (DATE)
CURTIME()    -- Heure actuelle uniquement (TIME)
CURRENT_TIMESTAMP()  -- Équivalent à NOW()
```

#### Exemple 1 : Fonctions de base

```sql
SELECT
    NOW() AS date_heure_actuelle,
    CURDATE() AS date_actuelle,
    CURTIME() AS heure_actuelle,
    UNIX_TIMESTAMP() AS timestamp_unix;
```

**Résultat attendu** :
```
+---------------------+----------------+----------------+----------------+
| date_heure_actuelle | date_actuelle  | heure_actuelle | timestamp_unix |
+---------------------+----------------+----------------+----------------+
| 2025-12-12 14:30:45 | 2025-12-12     | 14:30:45       |     1734012645 |
+---------------------+----------------+----------------+----------------+
```

#### Exemple 2 : Horodatage d'insertion

**Question métier** : *Enregistrer la date de création d'une commande*

```sql
INSERT INTO commandes (id_client, date_commande, statut, montant_total)
VALUES (1847, NOW(), 'En attente', 289.95);

-- Ou avec valeur par défaut dans la table
CREATE TABLE commandes (
    id_commande INT PRIMARY KEY AUTO_INCREMENT,
    date_commande DATETIME DEFAULT NOW(),
    date_modification TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                               ON UPDATE CURRENT_TIMESTAMP
);
```

---

## Extraction de composants

### YEAR(), MONTH(), DAY(), etc.

Extraire des parties spécifiques d'une date.

```sql
YEAR(date)       -- Année (2025)
MONTH(date)      -- Mois (1-12)
DAY(date)        -- Jour du mois (1-31)
HOUR(time)       -- Heure (0-23)
MINUTE(time)     -- Minute (0-59)
SECOND(time)     -- Seconde (0-59)
```

#### Exemple 3 : Analyse des ventes par mois

**Question métier** : *Chiffre d'affaires mensuel 2025*

```sql
SELECT
    YEAR(date_commande) AS annee,
    MONTH(date_commande) AS mois,
    MONTHNAME(date_commande) AS nom_mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
WHERE YEAR(date_commande) = 2025
GROUP BY annee, mois, nom_mois
ORDER BY mois;
```

**Résultat attendu** :
```
+--------+------+-----------+--------------+-----------+
| annee  | mois | nom_mois  | nb_commandes | ca_total  |
+--------+------+-----------+--------------+-----------+
|   2025 |    1 | January   |         1847 | 289473.45 |
|   2025 |    2 | February  |         1923 | 312847.92 |
|   2025 |    3 | March     |         2104 | 347293.18 |
|   2025 |    4 | April     |         1998 | 328492.73 |
+--------+------+-----------+--------------+-----------+
```

#### Exemple 4 : Jour de la semaine

**Question métier** : *Quel jour de la semaine génère le plus de ventes ?*

```sql
SELECT
    DAYNAME(date_commande) AS jour_semaine,
    DAYOFWEEK(date_commande) AS numero_jour,  -- 1=Dimanche, 7=Samedi
    COUNT(*) AS nb_commandes,
    ROUND(AVG(montant_total), 2) AS panier_moyen
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
GROUP BY jour_semaine, numero_jour
ORDER BY numero_jour;
```

**Résultat attendu** :
```
+--------------+-------------+--------------+---------------+
| jour_semaine | numero_jour | nb_commandes | panier_moyen  |
+--------------+-------------+--------------+---------------+
| Sunday       |           1 |          234 |        147.82 |
| Monday       |           2 |          389 |        156.34 |
| Tuesday      |           3 |          412 |        163.29 |
| Wednesday    |           4 |          398 |        159.47 |
| Thursday     |           5 |          421 |        168.92 |
| Friday       |           6 |          503 |        182.34 |  ← Plus fort
| Saturday     |           7 |          487 |        175.28 |
+--------------+-------------+--------------+---------------+
```

**Fonctions complémentaires** :

```sql
DAYOFMONTH(date)    -- Jour du mois (1-31), équivalent à DAY()
DAYOFYEAR(date)     -- Jour de l'année (1-366)
WEEK(date)          -- Numéro de semaine (0-53)
QUARTER(date)       -- Trimestre (1-4)
WEEKDAY(date)       -- 0=Lundi, 6=Dimanche
```

---

## Calculs de dates : Ajout et soustraction

### DATE_ADD() et DATE_SUB()

```sql
DATE_ADD(date, INTERVAL valeur unite)
DATE_SUB(date, INTERVAL valeur unite)

-- Unités : DAY, WEEK, MONTH, YEAR, HOUR, MINUTE, SECOND
```

#### Exemple 5 : Date d'échéance

**Question métier** : *Calculer la date de livraison prévue (7 jours après commande)*

```sql
SELECT
    id_commande,
    date_commande,
    DATE_ADD(date_commande, INTERVAL 7 DAY) AS date_livraison_prevue,
    DATE_ADD(date_commande, INTERVAL 1 WEEK) AS equivalent_1_semaine
FROM commandes
WHERE statut = 'En cours'
LIMIT 5;
```

**Résultat attendu** :
```
+-------------+---------------------+-----------------------+-----------------------+
| id_commande | date_commande       | date_livraison_prevue | equivalent_1_semaine  |
+-------------+---------------------+-----------------------+-----------------------+
|        1847 | 2025-12-10 14:23:00 | 2025-12-17 14:23:00   | 2025-12-17 14:23:00   |
|        1848 | 2025-12-11 09:45:00 | 2025-12-18 09:45:00   | 2025-12-18 09:45:00   |
|        1849 | 2025-12-12 16:30:00 | 2025-12-19 16:30:00   | 2025-12-19 16:30:00   |
+-------------+---------------------+-----------------------+-----------------------+
```

#### Exemple 6 : Périodes d'analyse

**Question métier** : *Comparer les ventes des 30 derniers jours*

```sql
SELECT
    DATE_SUB(CURDATE(), INTERVAL 30 DAY) AS debut_periode,
    CURDATE() AS fin_periode,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
WHERE date_commande BETWEEN DATE_SUB(CURDATE(), INTERVAL 30 DAY)
                        AND CURDATE();
```

**Résultat attendu** :
```
+---------------+-------------+--------------+-----------+
| debut_periode | fin_periode | nb_commandes | ca_total  |
+---------------+-------------+--------------+-----------+
| 2025-11-12    | 2025-12-12  |         2847 | 472938.45 |
+---------------+-------------+--------------+-----------+
```

#### Exemple 7 : Intervalles multiples

```sql
SELECT
    NOW() AS maintenant,
    DATE_ADD(NOW(), INTERVAL 3 MONTH) AS dans_3_mois,
    DATE_ADD(NOW(), INTERVAL 1 YEAR) AS dans_1_an,
    DATE_SUB(NOW(), INTERVAL 6 MONTH) AS il_y_a_6_mois,
    -- Combinaison d'intervalles
    DATE_ADD(DATE_ADD(NOW(), INTERVAL 1 YEAR), INTERVAL 3 MONTH) AS dans_15_mois;
```

**Unités disponibles** :
```
MICROSECOND, SECOND, MINUTE, HOUR
DAY, WEEK, MONTH, QUARTER, YEAR
SECOND_MICROSECOND, MINUTE_MICROSECOND, HOUR_MICROSECOND
DAY_MICROSECOND, MINUTE_SECOND, HOUR_SECOND
DAY_SECOND, HOUR_MINUTE, DAY_MINUTE, DAY_HOUR
YEAR_MONTH
```

---

## Calculs de différences

### DATEDIFF() : Différence en jours

```sql
DATEDIFF(date1, date2)  -- Retourne date1 - date2 en jours
```

#### Exemple 8 : Délai de traitement

**Question métier** : *Combien de jours entre commande et livraison ?*

```sql
SELECT
    id_commande,
    date_commande,
    date_livraison,
    DATEDIFF(date_livraison, date_commande) AS delai_jours,
    CASE
        WHEN DATEDIFF(date_livraison, date_commande) <= 3 THEN '🟢 Rapide'
        WHEN DATEDIFF(date_livraison, date_commande) <= 7 THEN '🟡 Normal'
        ELSE '🔴 Lent'
    END AS performance
FROM commandes
WHERE date_livraison IS NOT NULL
ORDER BY delai_jours DESC
LIMIT 5;
```

**Résultat attendu** :
```
+-------------+---------------------+---------------------+-------------+-------------+
| id_commande | date_commande       | date_livraison      | delai_jours | performance |
+-------------+---------------------+---------------------+-------------+-------------+
|        1723 | 2025-11-15 10:30:00 | 2025-11-27 14:20:00 |          12 | 🔴 Lent     |
|        1782 | 2025-11-20 08:15:00 | 2025-11-28 16:45:00 |           8 | 🔴 Lent     |
|        1847 | 2025-12-10 14:23:00 | 2025-12-16 11:30:00 |           6 | 🟡 Normal   |
|        1892 | 2025-12-08 09:00:00 | 2025-12-10 15:20:00 |           2 | 🟢 Rapide   |
+-------------+---------------------+---------------------+-------------+-------------+
```

#### Exemple 9 : Âge des clients

**Question métier** : *Calculer l'âge des clients*

```sql
SELECT
    id_client,
    nom,
    date_naissance,
    DATEDIFF(CURDATE(), date_naissance) AS age_jours,
    FLOOR(DATEDIFF(CURDATE(), date_naissance) / 365.25) AS age_annees
FROM clients
WHERE date_naissance IS NOT NULL
ORDER BY age_annees DESC
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+----------------+----------------+-----------+-------------+
| id_client | nom            | date_naissance | age_jours | age_annees  |
+-----------+----------------+----------------+-----------+-------------+
|      1847 | Alice Martin   | 1975-03-15     |     18534 |          50 |
|      2934 | Bob Dupont     | 1982-07-22     |     15898 |          43 |
|      3621 | Charlie Durand | 1990-11-08     |     12817 |          35 |
+-----------+----------------+----------------+-----------+-------------+
```

⚠️ **Note** : DATEDIFF compte les jours, division par 365.25 pour années approximatives.

### TIMESTAMPDIFF() : Différence en unités variées

```sql
TIMESTAMPDIFF(unite, date1, date2)  -- Retourne date2 - date1 dans l'unité
-- Unités : MICROSECOND, SECOND, MINUTE, HOUR, DAY, WEEK, MONTH, QUARTER, YEAR
```

#### Exemple 10 : Âge précis en années

**Question métier** : *Âge exact en années complètes*

```sql
SELECT
    id_client,
    nom,
    date_naissance,
    TIMESTAMPDIFF(YEAR, date_naissance, CURDATE()) AS age_exact,
    TIMESTAMPDIFF(MONTH, date_naissance, CURDATE()) AS age_mois
FROM clients
WHERE date_naissance IS NOT NULL
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+----------------+----------------+-----------+-----------+
| id_client | nom            | date_naissance | age_exact | age_mois  |
+-----------+----------------+----------------+-----------+-----------+
|      1847 | Alice Martin   | 1975-03-15     |        50 |       608 |
|      2934 | Bob Dupont     | 1982-07-22     |        43 |       520 |
|      3621 | Charlie Durand | 1990-11-08     |        35 |       421 |
+-----------+----------------+----------------+-----------+-----------+
```

#### Exemple 11 : Durée en heures et minutes

**Question métier** : *Temps de traitement des tickets support*

```sql
SELECT
    id_ticket,
    date_ouverture,
    date_cloture,
    TIMESTAMPDIFF(HOUR, date_ouverture, date_cloture) AS duree_heures,
    TIMESTAMPDIFF(MINUTE, date_ouverture, date_cloture) % 60 AS duree_minutes,
    CONCAT(
        TIMESTAMPDIFF(HOUR, date_ouverture, date_cloture), 'h ',
        TIMESTAMPDIFF(MINUTE, date_ouverture, date_cloture) % 60, 'min'
    ) AS duree_format
FROM tickets_support
WHERE date_cloture IS NOT NULL
ORDER BY duree_heures DESC
LIMIT 5;
```

**Résultat attendu** :
```
+-----------+---------------------+---------------------+--------------+---------------+--------------+
| id_ticket | date_ouverture      | date_cloture        | duree_heures | duree_minutes | duree_format |
+-----------+---------------------+---------------------+--------------+---------------+--------------+
|       847 | 2025-12-08 09:00:00 | 2025-12-11 16:45:00 |           79 |            45 | 79h 45min    |
|       892 | 2025-12-09 14:30:00 | 2025-12-11 11:20:00 |           44 |            50 | 44h 50min    |
|       923 | 2025-12-10 08:15:00 | 2025-12-11 15:30:00 |           31 |            15 | 31h 15min    |
+-----------+---------------------+---------------------+--------------+---------------+--------------+
```

---

## Formatage de dates

### DATE_FORMAT() : Personnaliser l'affichage

```sql
DATE_FORMAT(date, format)
```

#### Principaux spécificateurs

| Spécificateur | Description | Exemple |
|---------------|-------------|---------|
| **%Y** | Année 4 chiffres | 2025 |
| **%y** | Année 2 chiffres | 25 |
| **%m** | Mois numérique (01-12) | 12 |
| **%c** | Mois numérique (1-12) | 12 |
| **%M** | Nom du mois | December |
| **%b** | Nom court du mois | Dec |
| **%d** | Jour du mois (01-31) | 12 |
| **%e** | Jour du mois (1-31) | 12 |
| **%D** | Jour avec suffixe | 12th |
| **%W** | Nom du jour | Friday |
| **%a** | Nom court du jour | Fri |
| **%H** | Heure 24h (00-23) | 14 |
| **%h** / **%I** | Heure 12h (01-12) | 02 |
| **%i** | Minutes (00-59) | 30 |
| **%s** / **%S** | Secondes (00-59) | 45 |
| **%p** | AM/PM | PM |
| **%w** | Jour de semaine (0-6) | 5 |

#### Exemple 12 : Formats variés

**Question métier** : *Afficher dates dans différents formats*

```sql
SELECT
    date_commande,
    DATE_FORMAT(date_commande, '%d/%m/%Y') AS format_fr,
    DATE_FORMAT(date_commande, '%Y-%m-%d') AS format_iso,
    DATE_FORMAT(date_commande, '%M %D, %Y') AS format_us,
    DATE_FORMAT(date_commande, '%W %d %M %Y') AS format_long,
    DATE_FORMAT(date_commande, '%H:%i:%s') AS heure_seule,
    DATE_FORMAT(date_commande, '%d/%m/%Y %H:%i') AS format_complet
FROM commandes
LIMIT 3;
```

**Résultat attendu** :
```
+---------------------+------------+------------+---------------------+----------------------------+-------------+------------------+
| date_commande       | format_fr  | format_iso | format_us           | format_long                | heure_seule | format_complet   |
+---------------------+------------+------------+---------------------+----------------------------+-------------+------------------+
| 2025-12-12 14:30:45 | 12/12/2025 | 2025-12-12 | December 12th, 2025 | Friday 12 December 2025    | 14:30:45    | 12/12/2025 14:30 |
| 2025-12-10 09:15:22 | 10/12/2025 | 2025-12-10 | December 10th, 2025 | Wednesday 10 December 2025 | 09:15:22    | 10/12/2025 09:15 |
+---------------------+------------+------------+---------------------+----------------------------+-------------+------------------+
```

#### Exemple 13 : Rapport de ventes formaté

**Question métier** : *Export avec dates lisibles*

```sql
SELECT
    DATE_FORMAT(date_commande, '%d %M %Y') AS date_lisible,
    COUNT(*) AS nb_commandes,
    FORMAT(SUM(montant_total), 2) AS ca_total
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY DATE(date_commande)
ORDER BY date_commande DESC;
```

**Résultat attendu** :
```
+-------------------+--------------+-------------+
| date_lisible      | nb_commandes | ca_total    |
+-------------------+--------------+-------------+
| 12 December 2025  |          234 | 38,492.73   |
| 11 December 2025  |          198 | 32,847.92   |
| 10 December 2025  |          212 | 35,293.18   |
+-------------------+--------------+-------------+
```

---

## Conversion de chaînes en dates

### STR_TO_DATE() : Parser des dates

```sql
STR_TO_DATE(chaine, format)
```

#### Exemple 14 : Import de données

**Question métier** : *Convertir dates texte en format DATE*

```sql
-- Données import avec formats variés
SELECT
    date_texte,
    STR_TO_DATE(date_texte, '%d/%m/%Y') AS date_fr,
    STR_TO_DATE(date_texte, '%m-%d-%Y') AS date_us,
    STR_TO_DATE(date_texte, '%Y%m%d') AS date_iso_compact
FROM (
    SELECT '12/12/2025' AS date_texte
    UNION SELECT '25-03-2025'
    UNION SELECT '20251215'
) AS imports;
```

**Résultat attendu** :
```
+------------+------------+------------+------------------+
| date_texte | date_fr    | date_us    | date_iso_compact |
+------------+------------+------------+------------------+
| 12/12/2025 | 2025-12-12 | NULL       | NULL             |
| 25-03-2025 | NULL       | NULL       | NULL             |
| 20251215   | NULL       | NULL       | 2025-12-15       |
+------------+------------+------------+------------------+
```

⚠️ **Important** : Le format doit correspondre exactement au format de la chaîne.

#### Exemple 15 : Nettoyage et conversion

**Question métier** : *Importer dates avec formats incohérents*

```sql
UPDATE clients_import
SET date_naissance_propre =
    CASE
        -- Format JJ/MM/AAAA
        WHEN date_naissance_texte LIKE '%/%'
            THEN STR_TO_DATE(date_naissance_texte, '%d/%m/%Y')
        -- Format JJ-MM-AAAA
        WHEN date_naissance_texte LIKE '%-%'
            THEN STR_TO_DATE(date_naissance_texte, '%d-%m-%Y')
        -- Format AAAAMMJJ
        WHEN LENGTH(date_naissance_texte) = 8
            THEN STR_TO_DATE(date_naissance_texte, '%Y%m%d')
        ELSE NULL
    END;
```

---

## Comparaisons et filtres temporels

### BETWEEN pour périodes

#### Exemple 16 : Commandes d'une période

**Question métier** : *Commandes du dernier trimestre*

```sql
SELECT
    id_commande,
    id_client,
    DATE_FORMAT(date_commande, '%d/%m/%Y') AS date,
    montant_total
FROM commandes
WHERE date_commande BETWEEN '2025-10-01' AND '2025-12-31 23:59:59'
ORDER BY date_commande DESC
LIMIT 5;
```

#### Exemple 17 : Comparaison de périodes

**Question métier** : *Comparer CA décembre 2024 vs décembre 2025*

```sql
SELECT
    'Décembre 2024' AS periode,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
WHERE date_commande BETWEEN '2024-12-01' AND '2024-12-31 23:59:59'

UNION ALL

SELECT
    'Décembre 2025' AS periode,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total
FROM commandes
WHERE date_commande BETWEEN '2025-12-01' AND '2025-12-31 23:59:59';
```

**Résultat attendu** :
```
+----------------+--------------+-----------+
| periode        | nb_commandes | ca_total  |
+----------------+--------------+-----------+
| Décembre 2024  |         1847 | 289473.45 |
| Décembre 2025  |         2104 | 347293.18 | ← +19.9% vs N-1
+----------------+--------------+-----------+
```

### Optimisation : DATE() vs comparaisons

```sql
-- ❌ MOINS OPTIMAL : Fonction sur colonne empêche usage index
SELECT * FROM commandes
WHERE DATE(date_commande) = '2025-12-12';

-- ✅ PLUS OPTIMAL : Comparaison directe utilise l'index
SELECT * FROM commandes
WHERE date_commande >= '2025-12-12 00:00:00'
  AND date_commande < '2025-12-13 00:00:00';

-- Ou avec BETWEEN
SELECT * FROM commandes
WHERE date_commande BETWEEN '2025-12-12 00:00:00'
                        AND '2025-12-12 23:59:59';
```

---

## Cas d'usage métier avancés

### Cas 1 : Analyse de cohort (rétention client)

**Question métier** : *Combien de clients reviennent après leur 1ère commande ?*

```sql
WITH premiere_commande AS (
    SELECT
        id_client,
        MIN(date_commande) AS date_premiere_commande
    FROM commandes
    GROUP BY id_client
),
commandes_suivantes AS (
    SELECT
        c.id_client,
        pc.date_premiere_commande,
        c.date_commande,
        TIMESTAMPDIFF(DAY, pc.date_premiere_commande, c.date_commande) AS jours_depuis_premiere
    FROM commandes c
    INNER JOIN premiere_commande pc ON c.id_client = pc.id_client
    WHERE c.date_commande > pc.date_premiere_commande
)
SELECT
    CASE
        WHEN jours_depuis_premiere <= 7 THEN '0-7 jours'
        WHEN jours_depuis_premiere <= 30 THEN '8-30 jours'
        WHEN jours_depuis_premiere <= 90 THEN '31-90 jours'
        ELSE '90+ jours'
    END AS periode_retour,
    COUNT(DISTINCT id_client) AS nb_clients_revenus,
    COUNT(*) AS nb_commandes_suivantes
FROM commandes_suivantes
GROUP BY periode_retour
ORDER BY
    FIELD(periode_retour, '0-7 jours', '8-30 jours', '31-90 jours', '90+ jours');
```

**Résultat attendu** :
```
+---------------+---------------------+---------------------------+
| periode_retour| nb_clients_revenus  | nb_commandes_suivantes    |
+---------------+---------------------+---------------------------+
| 0-7 jours     |                 847 |                      1023 |
| 8-30 jours    |                1204 |                      1892 |
| 31-90 jours   |                 934 |                      1456 |
| 90+ jours     |                2847 |                      8394 |
+---------------+---------------------+---------------------------+
```

### Cas 2 : Calcul d'ancienneté

**Question métier** : *Années d'ancienneté des employés*

```sql
SELECT
    id_employe,
    nom,
    date_embauche,
    TIMESTAMPDIFF(YEAR, date_embauche, CURDATE()) AS annees_anciennete,
    TIMESTAMPDIFF(MONTH, date_embauche, CURDATE()) % 12 AS mois_supplementaires,
    CONCAT(
        TIMESTAMPDIFF(YEAR, date_embauche, CURDATE()), ' ans ',
        TIMESTAMPDIFF(MONTH, date_embauche, CURDATE()) % 12, ' mois'
    ) AS anciennete_complete,
    CASE
        WHEN TIMESTAMPDIFF(YEAR, date_embauche, CURDATE()) >= 10 THEN '🏅 Senior'
        WHEN TIMESTAMPDIFF(YEAR, date_embauche, CURDATE()) >= 5 THEN '⭐ Confirmé'
        ELSE '🌱 Junior'
    END AS niveau
FROM employes
ORDER BY date_embauche;
```

### Cas 3 : Calendrier de livraisons

**Question métier** : *Planning des livraisons cette semaine*

```sql
SELECT
    DAYNAME(date_livraison_prevue) AS jour,
    DATE_FORMAT(date_livraison_prevue, '%d/%m/%Y') AS date,
    COUNT(*) AS nb_livraisons,
    GROUP_CONCAT(
        CONCAT('#', id_commande)
        ORDER BY date_livraison_prevue
        SEPARATOR ', '
    ) AS commandes
FROM commandes
WHERE date_livraison_prevue BETWEEN
      DATE_ADD(CURDATE(), INTERVAL - WEEKDAY(CURDATE()) DAY)  -- Lundi
  AND DATE_ADD(CURDATE(), INTERVAL 6 - WEEKDAY(CURDATE()) DAY)  -- Dimanche
GROUP BY jour, date
ORDER BY date_livraison_prevue;
```

**Résultat attendu** :
```
+-----------+------------+----------------+---------------------------+
| jour      | date       | nb_livraisons  | commandes                 |
+-----------+------------+----------------+---------------------------+
| Monday    | 09/12/2025 |             23 | #1847, #1852, #1859, ...  |
| Tuesday   | 10/12/2025 |             31 | #1723, #1734, #1741, ...  |
| Wednesday | 11/12/2025 |             28 | #1892, #1903, #1912, ...  |
| Thursday  | 12/12/2025 |             34 | #1801, #1814, #1823, ...  |
| Friday    | 13/12/2025 |             42 | #1765, #1778, #1789, ...  |
+-----------+------------+----------------+---------------------------+
```

### Cas 4 : Détection tendances saisonnières

**Question métier** : *Mois avec le plus de ventes*

```sql
SELECT
    MONTHNAME(date_commande) AS mois,
    YEAR(date_commande) AS annee,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS ca_total,
    ROUND(AVG(montant_total), 2) AS panier_moyen
FROM commandes
WHERE date_commande >= DATE_SUB(CURDATE(), INTERVAL 24 MONTH)
GROUP BY MONTH(date_commande), YEAR(date_commande), mois, annee
ORDER BY MONTH(date_commande), annee;
```

### Cas 5 : Rappels d'échéances

**Question métier** : *Abonnements arrivant à expiration dans 30 jours*

```sql
SELECT
    id_abonnement,
    id_client,
    nom_client,
    date_fin_abonnement,
    DATEDIFF(date_fin_abonnement, CURDATE()) AS jours_restants,
    CASE
        WHEN DATEDIFF(date_fin_abonnement, CURDATE()) <= 7 THEN '🔴 Urgent'
        WHEN DATEDIFF(date_fin_abonnement, CURDATE()) <= 15 THEN '🟡 Bientôt'
        ELSE '🟢 OK'
    END AS urgence
FROM abonnements
WHERE date_fin_abonnement BETWEEN CURDATE()
                              AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
  AND statut = 'Actif'
ORDER BY jours_restants;
```

**Résultat attendu** :
```
+----------------+-----------+----------------+----------------------+----------------+-----------+
| id_abonnement  | id_client | nom_client     | date_fin_abonnement  | jours_restants | urgence   |
+----------------+-----------+----------------+----------------------+----------------+-----------+
|           1847 |      2934 | Alice Martin   | 2025-12-15           |              3 | 🔴 Urgent |
|           1923 |      3621 | Bob Dupont     | 2025-12-20           |              8 | 🟡 Bientôt|
|           2048 |      1847 | Charlie Durand | 2025-12-28           |             16 | 🟢 OK     |
+----------------+-----------+----------------+----------------------+----------------+-----------+
```

---

## Fuseaux horaires et TIMESTAMP

### CONVERT_TZ() : Conversion de fuseau

```sql
CONVERT_TZ(datetime, from_tz, to_tz)
```

#### Exemple 18 : Conversion UTC vers local

**Question métier** : *Afficher heures dans différents fuseaux*

```sql
SELECT
    date_commande AS heure_utc,
    CONVERT_TZ(date_commande, '+00:00', '+01:00') AS heure_paris,
    CONVERT_TZ(date_commande, '+00:00', '-05:00') AS heure_new_york,
    CONVERT_TZ(date_commande, '+00:00', '+09:00') AS heure_tokyo
FROM commandes
WHERE id_commande = 1847;
```

**Résultat attendu** :
```
+---------------------+---------------------+---------------------+---------------------+
| heure_utc           | heure_paris         | heure_new_york      | heure_tokyo         |
+---------------------+---------------------+---------------------+---------------------+
| 2025-12-12 13:30:00 | 2025-12-12 14:30:00 | 2025-12-12 08:30:00 | 2025-12-12 22:30:00 |
+---------------------+---------------------+---------------------+---------------------+
```

### Configuration fuseau horaire

```sql
-- Voir fuseau actuel
SELECT @@global.time_zone, @@session.time_zone;

-- Définir fuseau session
SET time_zone = '+01:00';  -- Europe/Paris
SET time_zone = 'UTC';
```

---

## Extraction avec EXTRACT()

```sql
EXTRACT(unite FROM date)
-- Unités : YEAR, MONTH, DAY, HOUR, MINUTE, SECOND, etc.
```

#### Exemple 19 : Alternative aux fonctions individuelles

```sql
SELECT
    date_commande,
    EXTRACT(YEAR FROM date_commande) AS annee,
    EXTRACT(MONTH FROM date_commande) AS mois,
    EXTRACT(DAY FROM date_commande) AS jour,
    EXTRACT(HOUR FROM date_commande) AS heure
FROM commandes
LIMIT 3;
```

**Équivalent à** :
```sql
SELECT
    date_commande,
    YEAR(date_commande) AS annee,
    MONTH(date_commande) AS mois,
    DAY(date_commande) AS jour,
    HOUR(date_commande) AS heure
FROM commandes
LIMIT 3;
```

---

## Fonctions spéciales

### LAST_DAY() : Dernier jour du mois

```sql
SELECT
    CURDATE() AS aujourd_hui,
    LAST_DAY(CURDATE()) AS dernier_jour_mois,
    DAY(LAST_DAY(CURDATE())) AS nb_jours_mois;
```

**Résultat** :
```
+--------------+-------------------+----------------+
| aujourd_hui  | dernier_jour_mois | nb_jours_mois  |
+--------------+-------------------+----------------+
| 2025-12-12   | 2025-12-31        |             31 |
+--------------+-------------------+----------------+
```

### MAKEDATE() : Créer date depuis année/jour

```sql
SELECT
    MAKEDATE(2025, 1) AS premier_jour_annee,
    MAKEDATE(2025, 365) AS dernier_jour_annee;
```

### FROM_UNIXTIME() : Timestamp Unix vers DATE

```sql
SELECT
    1734012645 AS timestamp_unix,
    FROM_UNIXTIME(1734012645) AS date_normale;
```

**Résultat** :
```
+----------------+---------------------+
| timestamp_unix | date_normale        |
+----------------+---------------------+
|     1734012645 | 2025-12-12 14:30:45 |
+----------------+---------------------+
```

---

## Performance et optimisation

### Index sur colonnes de dates

```sql
-- ✅ Index sur date pour filtres fréquents
CREATE INDEX idx_date_commande ON commandes(date_commande);

-- ✅ Index composite pour analyses par période
CREATE INDEX idx_annee_mois ON commandes(
    YEAR(date_commande),
    MONTH(date_commande)
);  -- Mais attention, fonctions empêchent usage optimal

-- ✅ MIEUX : Colonnes calculées
ALTER TABLE commandes
ADD COLUMN annee INT GENERATED ALWAYS AS (YEAR(date_commande)) STORED,
ADD COLUMN mois INT GENERATED ALWAYS AS (MONTH(date_commande)) STORED;

CREATE INDEX idx_annee_mois ON commandes(annee, mois);
```

### Éviter fonctions dans WHERE

```sql
-- ❌ LENT : Fonction empêche usage index
SELECT * FROM commandes
WHERE YEAR(date_commande) = 2025;

-- ✅ RAPIDE : Comparaison directe
SELECT * FROM commandes
WHERE date_commande >= '2025-01-01'
  AND date_commande < '2026-01-01';
```

### Partitionnement par date

```sql
-- Pour très grandes tables, partitionner par période
CREATE TABLE commandes_partitioned (
    id_commande INT,
    date_commande DATETIME,
    ...
) PARTITION BY RANGE (YEAR(date_commande)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

## Pièges courants et solutions

### Piège 1 : DATEDIFF vs TIMESTAMPDIFF ordre

```sql
-- ❌ Ordre inversé !
DATEDIFF(date1, date2)        -- date1 - date2
TIMESTAMPDIFF(unit, date1, date2)  -- date2 - date1

-- Exemple :
SELECT
    DATEDIFF('2025-12-15', '2025-12-10') AS datediff_resultat,      -- 5
    TIMESTAMPDIFF(DAY, '2025-12-10', '2025-12-15') AS timestampdiff_resultat;  -- 5
```

### Piège 2 : BETWEEN inclusif des deux côtés

```sql
-- ⚠️ BETWEEN inclut les bornes
SELECT * FROM commandes
WHERE date_commande BETWEEN '2025-12-01' AND '2025-12-31';
-- Inclut 2025-12-31 00:00:00 mais pas 2025-12-31 23:59:59 !

-- ✅ Solution : Jour suivant avec <
SELECT * FROM commandes
WHERE date_commande >= '2025-12-01'
  AND date_commande < '2026-01-01';
```

### Piège 3 : Comparaison DATE vs DATETIME

```sql
-- ❌ Peut manquer des résultats
SELECT * FROM commandes
WHERE date_commande = '2025-12-12';
-- Ne trouve que 2025-12-12 00:00:00

-- ✅ Utiliser plage
SELECT * FROM commandes
WHERE DATE(date_commande) = '2025-12-12';
-- Ou mieux (pour index) :
WHERE date_commande >= '2025-12-12 00:00:00'
  AND date_commande < '2025-12-13 00:00:00';
```

### Piège 4 : NULL dans calculs

```sql
-- ❌ NULL + calcul = NULL
SELECT DATEDIFF(NULL, '2025-12-12');  -- NULL

-- ✅ Utiliser COALESCE
SELECT DATEDIFF(COALESCE(date_livraison, CURDATE()), date_commande);
```

### Piège 5 : Années bissextiles

```sql
-- ❌ Division simpliste
SELECT DATEDIFF(CURDATE(), date_naissance) / 365 AS age;

-- ✅ Utiliser TIMESTAMPDIFF
SELECT TIMESTAMPDIFF(YEAR, date_naissance, CURDATE()) AS age;
```

### Piège 6 : Fuseaux horaires avec TIMESTAMP

```sql
-- ⚠️ TIMESTAMP stocke en UTC, affiche selon session
-- DATE/DATETIME ne changent pas selon fuseau

-- Voir différence :
SET time_zone = '+00:00';
SELECT date_commande FROM commandes WHERE id_commande = 1;
-- 2025-12-12 14:30:00

SET time_zone = '+01:00';
SELECT date_commande FROM commandes WHERE id_commande = 1;
-- 2025-12-12 15:30:00 (si TIMESTAMP), ou 14:30:00 (si DATETIME)
```

---

## Bonnes pratiques

### ✅ DO : Recommandations

1. **TIMESTAMPDIFF pour différences précises** : Plus fiable que DATEDIFF / 365
2. **Plages avec >= et < pour index** : Éviter fonctions dans WHERE
3. **DATETIME pour horodatage** : Pas affecté par fuseau horaire (vs TIMESTAMP)
4. **DATE_FORMAT pour affichage** : Jamais pour stockage
5. **Colonnes calculées pour analyses fréquentes** : YEAR, MONTH stockés
6. **BETWEEN avec attention** : Ou préférer >= et <
7. **Valider les dates** : STR_TO_DATE peut retourner NULL
8. **Index sur colonnes temporelles** : Pour filtres fréquents

### ❌ DON'T : À éviter

1. **Fonctions dans WHERE avec gros volumes** : Empêche usage index
2. **Stocker dates en VARCHAR** : Toujours utiliser DATE/DATETIME
3. **Comparer DATE = DATETIME directement** : Utiliser DATE() ou plages
4. **Oublier NULL** : Toujours gérer avec COALESCE/IS NULL
5. **DATE_FORMAT pour calculs** : Uniquement pour affichage
6. **Division par 365** : Utiliser TIMESTAMPDIFF(YEAR, ...)
7. **BETWEEN pour journée entière** : Utiliser >= et <
8. **Ignorer fuseaux horaires** : Documenter si UTC ou local

---

## Tableau récapitulatif des fonctions

| Catégorie | Fonction | Usage | Exemple |
|-----------|----------|-------|---------|
| **Obtenir date/heure** | NOW() | Date et heure | `NOW()` |
| | CURDATE() | Date uniquement | `CURDATE()` |
| | CURTIME() | Heure uniquement | `CURTIME()` |
| **Extraction** | YEAR() | Année | `YEAR(date_commande)` |
| | MONTH() | Mois | `MONTH(date_commande)` |
| | DAY() | Jour | `DAY(date_commande)` |
| | DAYNAME() | Nom du jour | `DAYNAME(date_commande)` |
| | HOUR(), MINUTE(), SECOND() | Composants heure | `HOUR(NOW())` |
| **Calculs** | DATE_ADD() | Ajouter intervalle | `DATE_ADD(NOW(), INTERVAL 7 DAY)` |
| | DATE_SUB() | Soustraire intervalle | `DATE_SUB(NOW(), INTERVAL 1 MONTH)` |
| | DATEDIFF() | Différence en jours | `DATEDIFF(date1, date2)` |
| | TIMESTAMPDIFF() | Différence en unités | `TIMESTAMPDIFF(YEAR, naissance, NOW())` |
| **Formatage** | DATE_FORMAT() | Formater affichage | `DATE_FORMAT(date, '%d/%m/%Y')` |
| | TIME_FORMAT() | Formater heure | `TIME_FORMAT(heure, '%H:%i')` |
| **Conversion** | STR_TO_DATE() | Texte vers date | `STR_TO_DATE('12/12/2025', '%d/%m/%Y')` |
| | FROM_UNIXTIME() | Unix vers date | `FROM_UNIXTIME(1734012645)` |
| | UNIX_TIMESTAMP() | Date vers Unix | `UNIX_TIMESTAMP(NOW())` |
| **Fuseau horaire** | CONVERT_TZ() | Changer fuseau | `CONVERT_TZ(date, '+00:00', '+01:00')` |
| **Spéciales** | LAST_DAY() | Dernier jour mois | `LAST_DAY(CURDATE())` |
| | MAKEDATE() | Créer depuis année/jour | `MAKEDATE(2025, 100)` |
| | EXTRACT() | Extraire composant | `EXTRACT(YEAR FROM date)` |

---

## ✅ Points clés à retenir

1. **Types DATE, DATETIME, TIMESTAMP** : Choisir selon besoin (fuseau horaire ?)

2. **Extraction : YEAR, MONTH, DAY, etc.** : Composants individuels

3. **DATE_ADD/DATE_SUB avec INTERVAL** : Calculs sur dates (7 DAY, 1 MONTH)

4. **DATEDIFF pour jours** : Différence simple en jours

5. **TIMESTAMPDIFF pour précision** : Différence dans unité spécifique (YEAR, MONTH, HOUR)

6. **DATE_FORMAT pour affichage** : Personnaliser présentation (%d/%m/%Y)

7. **STR_TO_DATE pour conversion** : Parser texte en date

8. **Éviter fonctions dans WHERE** : Empêchent usage index (utiliser plages)

9. **BETWEEN inclusif** : Attention aux bornes (préférer >= et <)

10. **Colonnes calculées pour performance** : YEAR, MONTH stockés avec index

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Date and Time Functions](https://mariadb.com/kb/en/date-time-functions/) – Documentation complète
- [📖 DATE_FORMAT](https://mariadb.com/kb/en/date_format/) – Tous les spécificateurs
- [📖 Time Zones](https://mariadb.com/kb/en/time-zones/) – Gestion fuseaux horaires

### Articles approfondis
- [Date Time Best Practices](https://use-the-index-luke.com/sql/where-clause/obfuscation/dates) – Performance
- [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) – Standard international

---

## ➡️ Prochaine section

**3.8 Expressions conditionnelles (CASE, IF, COALESCE)**

Prêt pour la logique conditionnelle ? La prochaine section couvre :
- Expression CASE (simple et recherchée)
- Fonction IF()
- COALESCE et NULLIF
- IFNULL et ISNULL
- Logique complexe dans les requêtes

Continuons vers la maîtrise complète ! 🚀

---


⏭️ [Expressions conditionnelles (CASE, IF, IFNULL, COALESCE, NULLIF)](/03-requetes-sql-intermediaires/08-expressions-conditionnelles.md)

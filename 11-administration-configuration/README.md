🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 11 — Administration et Configuration

> **Parcours concernés** : Administrateur/DBA (cœur de parcours) · DevOps/Cloud (sélection de sections)  
> **Version de référence** : MariaDB **12.3 LTS** — comparaisons ponctuelles avec la **11.8 LTS** lorsque c'est pertinent pour la migration  
> **Prérequis** : chapitre 1 (fondamentaux et installation), avec une bonne connaissance du chapitre 6 (transactions) et du chapitre 7 (moteurs de stockage)

---

## Vue d'ensemble

Les chapitres précédents vous ont appris à *parler* à MariaDB : écrire des requêtes, concevoir un schéma, programmer côté serveur. Ce chapitre change de perspective. Il ne s'agit plus d'utiliser la base, mais de la *faire vivre* dans la durée : la configurer correctement, comprendre les leviers qui régissent son comportement, la surveiller et la maintenir.

C'est le cœur du métier d'administrateur de bases de données (DBA). Une instance MariaDB livrée avec sa configuration par défaut fonctionne, mais elle n'est ni optimisée pour votre matériel, ni dimensionnée pour votre charge, ni instrumentée pour être diagnostiquée le jour où un incident survient. L'administration consiste précisément à transformer une installation générique en un service fiable, performant et prévisible, adapté à un contexte de production donné.

Ce chapitre adopte une logique « du fichier au comportement observé ». Nous partons de l'endroit où MariaDB lit ses réglages (les fichiers de configuration), nous remontons vers les variables qui pilotent le serveur à chaud, puis nous abordons les mécanismes qui produisent de la matière exploitable au quotidien : les journaux, la maintenance des objets, la gestion de l'espace disque et temporaire, et enfin la supervision et le réglage de la concurrence. Le chapitre se referme sur deux acquis structurants hérités de la 11.8 et désormais standards : le jeu de caractères `utf8mb4` par défaut et l'extension de la portée temporelle du type `TIMESTAMP`.

L'objectif n'est pas d'apprendre par cœur des centaines de variables, mais d'acquérir une grille de lecture : savoir *où* chercher un réglage, *comment* le modifier sans risque, et *quel signal* observer pour vérifier son effet.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- localiser, structurer et faire lire à MariaDB ses fichiers de configuration, en comprenant l'ordre de priorité entre eux ;
- distinguer variables système et variables de session, et modifier un réglage à chaud ou de façon persistante en sachant lesquelles sont dynamiques ;
- maîtriser le rôle de `sql_mode` dans le comportement du serveur (rigueur des contrôles, compatibilité) ;
- mettre en place et exploiter les différents journaux (erreurs, requêtes lentes, journal général, journaux binaires) ;
- assurer la maintenance courante des tables (`OPTIMIZE`, `ANALYZE`, `CHECK`, `REPAIR`) et gérer l'espace disque comme l'espace temporaire ;
- définir les métriques de supervision essentielles et comprendre le rôle du Thread Pool dans la gestion de la concurrence ;
- expliquer les choix par défaut de la 12.3 en matière de jeu de caractères et de gestion des dates.

---

## Plan du chapitre

Le chapitre suit une progression qui va de la configuration statique vers l'observation dynamique du serveur.

- **11.1 — Fichiers de configuration** : où et comment MariaDB lit ses réglages (`my.cnf` / `my.ini`), ordre de lecture des fichiers et configuration modulaire par inclusion.
- **11.2 — Variables système et de session** : consultation et modification des variables, distinction dynamique/statique, et inventaire des **variables retirées dans la série 12.x**.
- **11.3 — Modes SQL (`sql_mode`)** : l'interrupteur qui durcit ou assouplit le comportement du serveur, avec ses implications sur la compatibilité.
- **11.4 — Gestion des logs** : journal d'erreurs, journal des requêtes lentes et journal général.
- **11.5 — Journaux binaires et journaux de transactions** : configuration, formats (`STATEMENT`, `ROW`, `MIXED`), purge et rotation — ainsi que le **nouveau binlog intégré à InnoDB** de la 12.3.
- **11.6 — Maintenance des tables** : `OPTIMIZE`, `ANALYZE`, `CHECK` et `REPAIR`.
- **11.7 — Gestion de l'espace disque** : suivi et maîtrise de l'occupation des fichiers de données.
- **11.8 — Contrôle de l'espace temporaire** : encadrement de l'usage du temporaire avec `max_tmp_space_usage` et `max_total_tmp_space_usage`.
- **11.9 — Monitoring et métriques importantes** : les indicateurs à surveiller en priorité.
- **11.10 — Thread Pool et gestion de la concurrence** : adapter le serveur à un grand nombre de connexions simultanées.
- **11.11 — Jeu de caractères par défaut** : `utf8mb4` avec collations UCA 14.0.0 (acquis depuis la 11.8).
- **11.12 — Extension `TIMESTAMP` 2038 → 2106** : le problème de l'an 2038 (Y2038) résolu (acquis depuis la 11.8).

---

## Nouveautés et points d'attention 12.3 LTS

Deux évolutions de la série 12.x touchent directement l'administration et méritent une attention particulière dans ce chapitre :

| Section | Évolution | Pourquoi c'est important |
|---------|-----------|--------------------------|
| **11.2.3** | Retrait des variables `big_tables`, `large_page_size` et `storage_engine` | Des configurations ou scripts hérités de la 11.8 peuvent faire échouer le démarrage ou être ignorés : c'est un point de vérification clé lors d'une migration. |
| **11.5.4** | Journal binaire réécrit et intégré à InnoDB | La synchronisation entre binlog et moteur disparaît, ce qui se traduit par un gain d'écriture de l'ordre de **4×**. C'est l'une des features phares de la série 12.x. |

Par ailleurs, plusieurs fonctionnalités introduites par la 11.8 sont désormais considérées comme **du contenu standard** dans cette formation. Elles restent abordées ici parce qu'elles structurent le comportement par défaut du serveur : le jeu de caractères `utf8mb4` et les collations UCA 14.0.0 (section 11.11), l'extension de `TIMESTAMP` jusqu'en 2106 (section 11.12) et le contrôle de l'espace temporaire (section 11.8).

---

## Comment aborder ce chapitre

Ce chapitre est conçu pour être lu de façon linéaire, car chaque section s'appuie sur la précédente : on ne peut raisonner sur les variables (§11.2) sans savoir d'où elles sont lues (§11.1), ni superviser utilement (§11.9) sans avoir mis en place les journaux (§11.4 et §11.5).

Les DBA y trouveront le socle indispensable à l'exploitation quotidienne. Les profils DevOps peuvent se concentrer sur la configuration (11.1, 11.2), les journaux binaires (11.5, essentiels pour la réplication abordée au chapitre 13) et la supervision (§11.9), qui prolongent les chapitres consacrés à l'automatisation et au cloud.

Si vous préparez une montée de version depuis la 11.8, gardez les sections 11.2.3 et 11.5.4 en tête : elles sont reprises et approfondies dans la section dédiée à la migration 11.8 → 12.3 (§19.10).

⏭️ [Fichiers de configuration](/11-administration-configuration/01-fichiers-configuration.md)

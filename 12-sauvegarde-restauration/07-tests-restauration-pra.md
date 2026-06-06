🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.7 Tests de restauration et plan de reprise (PRA)

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Cette section est l'aboutissement du chapitre. Tout ce qui précède — les stratégies de sauvegarde, les procédures de restauration, l'automatisation — ne sert qu'un seul objectif : **être capable de remettre le service en route** lorsqu'un incident survient. Or cette capacité ne se décrète pas, elle se **vérifie** et se **planifie**.

Deux piliers structurent donc cette démarche. D'une part, **tester régulièrement les restaurations**, pour s'assurer que les sauvegardes sont réellement exploitables et mesurer le temps de remise en service. D'autre part, formaliser un **plan de reprise d'activité (PRA)**, qui documente la marche à suivre face à un sinistre. C'est ici que le fil conducteur du chapitre — « une sauvegarde non éprouvée n'est qu'une espérance » — devient une pratique concrète.

---

## Pourquoi tester les restaurations

Une sauvegarde est dans un état comparable au chat de Schrödinger : tant qu'on ne l'a pas restaurée, on **ignore si elle est valide**. Une sauvegarde peut en effet échouer de mille façons **invisibles** au quotidien : fichier corrompu, dump incomplet, objet oublié (procédures, événements — voir [12.2.2](02.2-options-essentielles.md)), chaîne incrémentale rompue, mauvaise configuration, sauvegarde jamais préparée… Aucun de ces défauts ne se manifeste **avant** la tentative de restauration — c'est-à-dire, sans test, au pire moment possible : en pleine crise.

Seule une **restauration réussie prouve** qu'une sauvegarde est exploitable. Tester présente en outre un bénéfice essentiel : cela révèle le **RTO réel**. Le temps de restauration est presque toujours plus long que ce que l'on imagine, et le découvrir lors d'un incident, plutôt qu'à l'avance, peut être dramatique.

---

## Que tester

Un test de restauration complet couvre plusieurs dimensions :

- l'**intégrité** de la sauvegarde (est-elle lisible et complète ?) ;
- la **restauration complète** (les données reviennent-elles correctement ? voir [12.5.1](05.1-restauration-complete.md)) ;
- la **restauration à un instant précis** (le PITR fonctionne-t-il ? voir [12.5.2](05.2-pitr.md)) ;
- le **temps réel** de restauration (mesure du RTO) ;
- la **cohérence des données** après restauration ;
- l'ensemble de la **chaîne**, y compris les copies hors site.

---

## Comment tester

Le premier principe est intangible : on restaure **toujours sur un environnement isolé**, **jamais sur la production**. Un test raté ne doit en aucun cas aggraver la situation.

La vérification doit ensuite être **systématique**, et non un simple « ça a l'air d'avoir marché ». Quelques contrôles concrets : comparer le **nombre de lignes** des tables clés entre la source et la base restaurée, calculer des **sommes de contrôle** (`CHECKSUM TABLE`, ou un outil comme `pt-table-checksum` pour comparer deux serveurs), et exécuter des **tests fonctionnels** de l'application (connexion, requêtes critiques).

Dans la mesure du possible, ces tests gagnent à être **automatisés** : une tâche planifiée qui restaure la dernière sauvegarde sur une instance de test, lance les vérifications, mesure la durée et **alerte en cas d'anomalie**. C'est le prolongement naturel de l'automatisation des sauvegardes ([12.6](06-automatisation-sauvegardes.md)). Enfin, chaque test doit être **documenté** : résultat, RTO mesuré, anomalies rencontrées.

---

## À quelle fréquence

Les tests de restauration doivent être **réguliers et planifiés** (par exemple un test de restauration complète mensuel, plus fréquent pour les systèmes critiques). Ils doivent surtout être **renouvelés après tout changement** susceptible d'affecter la chaîne : modification du processus de sauvegarde, évolution du schéma, montée de version de MariaDB, changement d'infrastructure. Le tout **premier** test d'une nouvelle configuration est le plus important : c'est lui qui valide que l'ensemble fonctionne réellement.

---

## Le plan de reprise d'activité (PRA)

Un **PRA** (*Plan de Reprise d'Activité*) est un document décrivant **comment rétablir le service** après un sinistre. Il ne faut pas le confondre avec le **PCA** (*Plan de Continuité d'Activité*), plus large : le PCA vise à **maintenir** l'activité pendant l'incident, là où le PRA organise la **reprise** après une interruption. Le PRA traduit en procédures concrètes les objectifs métier que sont le **RPO** et le **RTO** (introduits en [section 12.1](01-strategies-sauvegarde.md)).

### Les composants d'un PRA

Un PRA opérationnel comporte au minimum :

- les **objectifs RPO et RTO** (l'exigence métier qui dimensionne tout le reste) ;
- un **inventaire** de ce qui doit être restauré : bases de données, fichiers de configuration, journaux binaires, identifiants, certificats… ;
- des **procédures de restauration documentées** (*runbooks*), suffisamment précises pour être exécutées **sous stress, par une personne qui ne les a pas écrites** ;
- les **rôles et responsabilités** : qui fait quoi en cas d'incident ;
- les **contacts et l'escalade** (équipes, prestataires, astreinte) ;
- l'**emplacement des sauvegardes** et les **accès** nécessaires ;
- les **dépendances** (DNS, réseau, serveurs applicatifs, supervision) ;
- un **plan de communication** vers les parties prenantes et les utilisateurs.

### Les scénarios à couvrir

Un bon PRA envisage plusieurs types de sinistres, qui n'appellent pas tous la même réponse :

- **panne matérielle** (disque, serveur) ;
- **corruption de données** ;
- **acte malveillant** (rançongiciel, sabotage) ;
- **erreur humaine** (un `DROP`/`DELETE` accidentel → PITR, [12.5.2](05.2-pitr.md)) ;
- **sinistre majeur** (perte d'un centre de données ou d'une région cloud).

---

## PRA et haute disponibilité : complémentaires, pas interchangeables

Une confusion fréquente consiste à penser que la **haute disponibilité** (réplication, Galera, *failover* — chapitre [14](../14-haute-disponibilite/README.md)) rend les sauvegardes superflues. C'est une erreur dangereuse. La HA réduit l'indisponibilité face aux **pannes d'infrastructure**, mais elle ne protège **pas** contre la corruption logique, le rançongiciel ou l'erreur humaine : une suppression accidentelle ou une corruption **se propage instantanément** vers toutes les répliques.

Les sauvegardes et le PRA couvrent précisément ce que la HA ne peut pas couvrir. Une stratégie complète **combine les deux** : la haute disponibilité pour la résilience aux pannes, les sauvegardes et le PRA pour les scénarios de corruption, d'erreur et de sinistre.

---

## Tester le PRA : les exercices de reprise

De même qu'une sauvegarde non testée n'est qu'une espérance, **un PRA jamais exercé est tout aussi peu fiable**. Le plan doit être **éprouvé** périodiquement par des **exercices de reprise**. On distingue généralement les exercices « sur table » (*tabletop*, où l'on déroule le plan théoriquement, étape par étape) des exercices « réels » (où l'on **exécute effectivement** une restauration de bout en bout, idéalement sur un environnement dédié).

Ces exercices ont un double objectif : **mesurer le RPO et le RTO réellement atteints** au regard des cibles fixées, et **identifier les lacunes** du plan (procédure obsolète, accès manquant, dépendance oubliée). Le PRA doit ensuite être **mis à jour** à la lumière de ces enseignements — c'est un document vivant, pas une archive.

---

## Bonnes pratiques

- **Testez les restaurations régulièrement** et **automatisez la vérification** (restauration + contrôles + alerte).
- **Restaurez sur un environnement isolé**, jamais sur la production.
- **Vérifiez réellement** : volumes, sommes de contrôle, tests applicatifs — pas seulement l'absence d'erreur.
- **Documentez des *runbooks*** exécutables sous stress par une tierce personne.
- **Définissez et suivez** vos objectifs **RPO/RTO**, et mesurez-les lors des exercices.
- **Couvrez plusieurs scénarios** de sinistre, y compris ceux que la HA n'adresse pas.
- **Combinez** haute disponibilité et sauvegardes/PRA — ils sont complémentaires.
- **Exercez le PRA**, ne vous contentez pas de l'écrire, et **tenez-le à jour**.

---

## À retenir

- Une sauvegarde est de **statut inconnu tant qu'elle n'a pas été restaurée** ; seule une restauration réussie en prouve la validité, et le test révèle le **RTO réel**.
- Testez **sur un environnement isolé**, **vérifiez systématiquement** (volumes, sommes de contrôle, contrôles applicatifs) et **automatisez** ces tests.
- Renouvelez les tests **régulièrement** et **après tout changement** (processus, schéma, version, infrastructure).
- Le **PRA** documente la reprise après sinistre : objectifs RPO/RTO, inventaire, *runbooks*, rôles, contacts, dépendances, communication.
- La **haute disponibilité ne remplace pas** les sauvegardes : elle ne protège ni de la corruption, ni du rançongiciel, ni de l'erreur humaine. Les deux sont **complémentaires**.
- Un PRA doit être **exercé** périodiquement (exercices sur table ou réels) et **mis à jour** en conséquence.

---

⏮️ [12.6 — Automatisation des sauvegardes](06-automatisation-sauvegardes.md) · ⏭️ [12.8 — Sauvegarde cloud-native](08-sauvegarde-cloud-native.md)

⏭️ [Sauvegarde cloud-native](/12-sauvegarde-restauration/08-sauvegarde-cloud-native.md)

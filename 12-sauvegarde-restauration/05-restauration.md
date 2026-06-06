🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5 Restauration

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Tout ce chapitre converge vers ce moment : la **restauration**. C'est elle, et elle seule, qui donne une valeur à une sauvegarde. Comme nous l'avons posé en introduction du chapitre, une sauvegarde dont la restauration n'a jamais été éprouvée n'est pas une sauvegarde, c'est une espérance. Une stratégie ne se mesure donc jamais à la régularité de ses sauvegardes, mais à sa capacité **prouvée** à remettre les données en service.

Restaurer n'est pas un simple « copier-coller à l'envers ». L'opération se déroule presque toujours dans des conditions difficiles — souvent en situation de crise, production interrompue, sous la pression du temps. La méthode dépend par ailleurs étroitement du **type de sauvegarde** réalisé, et le résultat visé n'est pas toujours le même : remettre la base dans l'état du dernier instantané, ou la ramener à un instant **précis** d'avant un incident. Cette section organise la restauration autour de ces deux scénarios.

---

## La restauration dépend du type de sauvegarde

Chaque approche de sauvegarde possède sa propre voie de restauration, déjà esquissée dans les sections correspondantes :

| Type de sauvegarde | Outil | Méthode de restauration |
|--------------------|-------|--------------------------|
| Logique (fichier unique) | `mariadb-dump` | Rejeu du script SQL : `mariadb base < dump.sql` ([12.2.1](02.1-mysqldump-mariadb-dump.md)) |
| Logique (parallèle) | `mydumper` | Rejeu parallèle avec `myloader` ([12.2.3](02.3-mydumper-myloader.md)) |
| Physique | Mariabackup | **`--prepare`** puis **`--copy-back`** / `--move-back`, serveur arrêté ([12.3.1](03.1-full-backup.md)) |
| Continue (journaux) | binlogs / `mariadb-binlog` | Rejeu des journaux après une sauvegarde de base ([12.4](04-sauvegarde-incrementale-binlog.md)) |

Deux différences structurantes en découlent. La restauration **logique** réexécute des instructions SQL et reconstruit les index : elle est simple mais **lente** sur de gros volumes. La restauration **physique** remet des fichiers en place : elle est **rapide**, mais exige un serveur arrêté, un répertoire de données vide et la correction des permissions. Ce contraste, déjà détaillé en [12.2](02-sauvegarde-logique.md) et [12.3](03-sauvegarde-physique-mariabackup.md), pèse directement sur le temps de remise en service.

---

## Deux scénarios de restauration

Cette section distingue deux situations, traitées dans les sous-sections suivantes.

La **restauration complète** ([12.5.1](05.1-restauration-complete.md)) ramène la base dans l'état exact d'une sauvegarde donnée. C'est le scénario de la reprise après sinistre majeur (perte d'un serveur), de la reconstruction d'une instance, ou du clonage d'un environnement (production → recette). On restaure « jusqu'au dernier bon instantané ».

La **restauration à un instant précis** (*Point-in-Time Recovery*, [12.5.2](05.2-pitr.md)) vise un moment **précis** dans le temps, en combinant une sauvegarde de base et le **rejeu des journaux binaires** ([12.4](04-sauvegarde-incrementale-binlog.md)). C'est le scénario chirurgical : remonter à l'instant qui précède immédiatement une fausse manœuvre (un `DROP TABLE` ou un `DELETE` accidentel), pour récupérer tout sauf l'erreur.

En résumé : la restauration complète répond à « revenir au dernier état sauvegardé », le PITR à « revenir à un moment exact, souvent juste avant une erreur ».

---

## Le facteur temps : le RTO

La restauration se déroule presque toujours sous contrainte de temps. Le **RTO** (*Recovery Time Objective*), introduit au début du chapitre, fixe la durée d'indisponibilité acceptable — et c'est lui qui guide les choix techniques. Une exigence de remise en service rapide oriente vers la sauvegarde **physique** (restauration par copie de fichiers) plutôt que logique (rejeu de SQL). De même, la **parallélisation** (`myloader`) peut réduire fortement le temps d'une restauration logique sur les grandes bases.

Mais le facteur temps ne se joue pas qu'au moment de l'incident : il se prépare **en amont**, par des procédures documentées, des sauvegardes vérifiées et un stockage accessible.

---

## Préparer la restauration *avant* d'en avoir besoin

Le pire moment pour découvrir qu'une sauvegarde est inexploitable est en plein incident de production. Plusieurs préparations réduisent ce risque :

- **Tester régulièrement les restaurations** sur un environnement distinct, pour vérifier que les sauvegardes sont valides et que la procédure fonctionne réellement.
- **Documenter un mode opératoire** (*runbook*) clair, exécutable sous stress, par une personne qui ne l'a pas écrit.
- **Vérifier l'intégrité** des sauvegardes et **conserver les coordonnées** nécessaires (par exemple `mariadb_backup_binlog_info` pour un PITR).
- **Garantir l'accessibilité** du stockage de sauvegarde (et sa séparation du serveur, règle 3-2-1).

Ces aspects, ainsi que la formalisation d'un plan de reprise d'activité (PRA), sont développés en [section 12.7](07-tests-restauration-pra.md).

---

## Points d'attention communs

Quelques précautions valent pour toute restauration, quel que soit le scénario :

- **Restaurer d'abord sur une cible sûre** lorsque c'est possible : une instance temporaire ou de recette, pour valider la sauvegarde **avant** d'écraser quoi que ce soit en production. On évite ainsi d'aggraver un incident par une restauration ratée.
- **Respecter les contraintes de la méthode** : serveur arrêté et `datadir` vide pour une restauration physique ; serveur démarré pour un rejeu logique.
- **Corriger les permissions** après une restauration physique (`chown -R mysql:mysql`), Mariabackup écrivant les fichiers sous l'identité de l'utilisateur qui restaure.
- **Vérifier après coup** : cohérence des données, volumes attendus, contrôles applicatifs — une restauration n'est terminée qu'une fois validée.
- **Veiller à la compatibilité** de version et d'architecture, particulièrement pour une restauration physique (peu portable, contrairement à une sauvegarde logique).

---

## Structure de la section

- [**12.5.1 — Restauration complète**](05.1-restauration-complete.md) : ramener la base dans l'état d'une sauvegarde (logique ou physique).
- [**12.5.2 — Point-in-time recovery (PITR)**](05.2-pitr.md) : restaurer à un instant précis en combinant une sauvegarde de base et le rejeu des journaux binaires.

---

## À retenir

- La **restauration** est ce qui valide une sauvegarde ; une stratégie se mesure à sa capacité **prouvée** à restaurer, pas à la fréquence de ses sauvegardes.
- La méthode dépend du **type de sauvegarde** : rejeu SQL (logique), `--prepare` + `--copy-back` (physique), rejeu de binlogs (continue).
- Deux scénarios : la **restauration complète** (revenir au dernier état sauvegardé) et le **PITR** (revenir à un instant précis, souvent juste avant une erreur).
- Le **RTO** guide le choix de la méthode : le physique restaure plus vite que le logique.
- La restauration se **prépare en amont** : procédures testées, sauvegardes vérifiées, stockage accessible (voir [12.7](07-tests-restauration-pra.md)).
- **Restaurez d'abord sur une cible sûre**, respectez les contraintes de la méthode, corrigez les permissions et **vérifiez** le résultat.

---

⏮️ [12.4 — Sauvegarde incrémentale avec binary logs](04-sauvegarde-incrementale-binlog.md) · ⏭️ [12.5.1 — Restauration complète](05.1-restauration-complete.md)

⏭️ [Restauration complète](/12-sauvegarde-restauration/05.1-restauration-complete.md)

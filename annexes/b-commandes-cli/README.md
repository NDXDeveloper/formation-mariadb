🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B — Commandes mariadb CLI Essentielles

> ⌨️ Aide-mémoire du client en ligne de commande `mariadb`, dans le contexte de MariaDB 12.3 LTS.

Cette annexe rassemble les commandes du client en ligne de commande livré avec MariaDB. Elle se consulte comme un aide-mémoire : on y revient pour retrouver la syntaxe d'une commande, non pour apprendre l'outil de façon linéaire.

Un point de vocabulaire utile dès le départ : le client historiquement appelé `mysql` se nomme désormais `mariadb`, l'ancien nom étant conservé comme alias de compatibilité. Il en va de même pour les outils compagnons (`mariadb-dump` pour `mysqldump`, `mariadb-admin`, `mariadb-import`, etc.). Les exemples de cette formation privilégient la nomenclature `mariadb*`.

## Distinction essentielle : commandes client et instructions SQL

Au sein d'une session, le client mêle deux natures d'instructions qu'il faut savoir distinguer. Les **commandes client** sont des raccourcis interprétés localement par `mariadb` (par exemple `\s`, `\u`, `\q`) ; elles ne sont pas envoyées au serveur et ne se terminent pas par un point-virgule. Les **instructions SQL** (par exemple `SHOW DATABASES`, `STATUS`) sont, elles, transmises au serveur et se terminent par `;` ou par `\G`. Cette annexe couvre les deux.

## Objectif

L'aide-mémoire vise à fixer les commandes du quotidien — se connecter, naviguer entre bases, inspecter l'état du serveur, importer ou exporter des données — et à servir de point d'entrée vers les chapitres qui développent chaque sujet en profondeur. Les profils débutants y trouvent les gestes de base ; les profils confirmés, un rappel rapide de syntaxe.

## Comment utiliser cet aide-mémoire

Les commandes sont réparties en trois familles, à consulter selon le besoin : la connexion et la navigation pour entrer dans le serveur et s'y repérer (B.1), les commandes d'inspection pour observer son état (B.2), et l'export/import pour faire entrer ou sortir des données (B.3).

## Contenu de l'annexe

### B.1 — [Connexion et navigation](01-connexion-navigation.md)

Se connecter au serveur, changer de base courante et lister les objets : commandes client comme `\s` (statut de la session) et `\u` (changement de base), et instructions SQL d'exploration comme `SHOW DATABASES` et `SHOW TABLES`.

### B.2 — [Informations système](02-informations-systeme.md)

Inspecter l'état du serveur et des sessions : `STATUS`, `SHOW PROCESSLIST` (connexions et requêtes en cours) et `SHOW ENGINE` (état interne des moteurs, dont InnoDB). Ce sont les premiers réflexes du diagnostic en production.

### B.3 — [Export/Import](03-export-import.md)

Faire entrer et sortir des données et des scripts : exécution d'un fichier SQL avec `SOURCE` (et l'option `--script-dir`), journalisation de la session avec `TEE`, et sauvegarde logique avec `mariadb-dump` (prise en charge des wildcards via `-L`).

## Pour aller plus loin

L'annexe reste centrée sur l'usage du client. Les sujets sous-jacents sont traités en détail ailleurs dans la formation.

| Pour approfondir… | Voir |
|-------------------|------|
| Configuration et administration du serveur | [Ch. 11 — Administration et Configuration](../../11-administration-configuration/README.md) |
| Sauvegarde logique et physique, restauration | [Ch. 12 — Sauvegarde et Restauration](../../12-sauvegarde-restauration/README.md) |
| Monitoring et état de la réplication | [Ch. 13 — Réplication](../../13-replication/README.md) |
| Requêtes d'administration et de monitoring prêtes à l'emploi | [Annexe C — Requêtes SQL de Référence](../c-requetes-sql-reference/README.md) |
| Outils graphiques (HeidiSQL, DBeaver, phpMyAdmin) | [Ch. 1.9 — Outils d'administration](../../01-introduction-fondamentaux/README.md) |

## Conventions

Les commandes, options et noms d'objets sont notés en `police à chasse fixe`. Les paramètres à remplacer apparaissent entre chevrons, par exemple `<base>` ou `<utilisateur>`. Les commandes client sont précédées d'un antislash (`\s`, `\u`) et ne prennent pas de point-virgule final, contrairement aux instructions SQL.

---

⬅️ [Annexe A — Glossaire des Termes Techniques](../a-glossaire/README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [B.1 — Connexion et navigation](01-connexion-navigation.md)

⏭️ [Connexion et navigation (\s, \u, SHOW DATABASES, SHOW TABLES)](/annexes/b-commandes-cli/01-connexion-navigation.md)

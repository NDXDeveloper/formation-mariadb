🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.3 — Export/Import

> 📤📥 Faire entrer et sortir scripts et données depuis le client et la ligne de commande.

Cette section regroupe les gestes d'import/export accessibles depuis le client `mariadb` et son écosystème : exécuter un script SQL, journaliser une session, et produire une sauvegarde logique avec `mariadb-dump`. Il s'agit ici du niveau « aide-mémoire » ; la stratégie de sauvegarde et de restauration dans son ensemble (sauvegarde physique, parallélisme, PITR) est traitée au Ch. 12.

---

## Exécuter un script SQL : `SOURCE` / `\.`

Depuis une session ouverte, la commande client `SOURCE` (ou sa forme courte `\.`) exécute le contenu d'un fichier SQL. Le client lit le fichier **localement** puis envoie les instructions au serveur.

```sql
SOURCE /chemin/vers/schema.sql;
\. /chemin/vers/donnees.sql
```

Sans chemin, le fichier est recherché dans le répertoire de travail du client ; il est donc plus sûr de fournir un **chemin absolu**. En dehors d'une session, la redirection d'entrée du shell offre une alternative équivalente, pratique pour le mode batch :

```bash
mariadb -u root -p boutique < schema.sql
```

### Scripts imbriqués et `--script-dir`

Un script peut en appeler d'autres via `source`. Lorsque ces sous-scripts sont référencés par leur seul nom (sans chemin), le client les cherche par défaut dans son répertoire de travail. Depuis **MariaDB 12.0**, l'option de démarrage `--script-dir=<répertoire>` permet d'indiquer où ces scripts doivent être recherchés, ce qui facilite l'organisation d'une arborescence de scripts modulaires.

```bash
mariadb -u root -p --script-dir=/opt/sql/modules boutique < install.sql
```

## Journaliser une session : `TEE` / `\T`

La commande client `TEE` (forme courte `\T`) recopie l'intégralité de la session — instructions saisies et résultats — dans un fichier, en ajout. C'est commode pour conserver une trace ou un compte rendu d'une session interactive. `\t` (`notee`) interrompt la journalisation.

```sql
\T /tmp/session.log     -- démarre la journalisation
-- ... vos instructions ...
\t                      -- arrête la journalisation
```

L'option de démarrage `--tee=<fichier>` active la journalisation dès le lancement du client.

## Sécurité : exécution de fichiers et mode `--sandbox`

Les commandes qui touchent au système de fichiers (`source`, `tee`, `system` via `\!`, `pager` avec argument) peuvent être un risque lorsqu'on exécute un script d'origine incertaine. Le client propose un mode bac à sable : l'option de démarrage `--sandbox`, ou la commande `\-` au sein d'une session, désactive ces commandes. Toute tentative d'utilisation devient alors une erreur (que l'on peut forcer à ignorer avec `--force`). Activé par `\-`, le mode reste en vigueur jusqu'à la fin du fichier ou de la session.

## Sauvegarde logique : `mariadb-dump`

`mariadb-dump` (l'ancien `mysqldump`) produit une **sauvegarde logique** : un fichier de texte contenant les instructions SQL nécessaires pour recréer schéma et données.

```bash
# Une base
mariadb-dump -u root -p boutique > boutique.sql

# Plusieurs bases, ou toutes
mariadb-dump -u root -p --databases boutique facturation > deux_bases.sql
mariadb-dump -u root -p --all-databases > complet.sql

# Tables précises d'une base
mariadb-dump -u root -p boutique clients commandes > tables.sql
```

Les options les plus courantes :

| Option | Rôle |
|--------|------|
| `--single-transaction` | Dump cohérent sans verrouiller les tables (recommandé pour InnoDB) |
| `--all-databases`, `-A` | Sauvegarde toutes les bases |
| `--databases`, `-B` | Sauvegarde plusieurs bases nommées (ajoute les `CREATE DATABASE` / `USE`) |
| `--routines` | Inclut les procédures et fonctions stockées |
| `--events` | Inclut les événements planifiés |
| `--no-data`, `-d` | Structure seule (aucune donnée) |
| `--no-create-info`, `-t` | Données seules (aucun `CREATE TABLE`) |
| `--ignore-table=base.table` | Exclut une table (répétable) |
| `--ignore-database=base` | Exclut une base (avec `--all-databases`) |
| `--result-file=fichier` | Écrit dans un fichier (préférable à `>`, notamment sous Windows) |

Deux points d'attention sur le contenu : les triggers sont inclus par défaut, alors que `--routines` et `--events` doivent être ajoutés explicitement pour ne pas perdre la logique côté serveur. Par ailleurs, depuis **MariaDB 12.1**, `mariadb-dump` accepte les wildcards (`%`, `_`) dans les noms, à condition d'activer l'option `-L` / `--wildcards` : employée seule, elle ne reconnaît les motifs que dans les noms de **tables** ; combinée à `--databases`, elle les étend aux noms de **bases**. Sans cette option, un argument contenant `%` est interprété littéralement (le dump échoue alors sur « *Couldn't find table* »). À ne pas confondre avec `-l` (minuscule), qui est `--lock-tables`.

### Restaurer un dump

Une sauvegarde produite par `mariadb-dump` est un simple script SQL : on la restaure donc comme tel, par redirection ou via `SOURCE`.

```bash
# Restauration d'une base précise
mariadb -u root -p boutique < boutique.sql
```

```sql
-- Ou depuis une session
USE boutique;
SOURCE boutique.sql;
```

Pour un dump créé avec `--databases` ou `--all-databases`, il ne faut pas préciser de base cible : le fichier contient déjà les instructions de création et de sélection des bases.

## Pour aller plus loin

Cet aide-mémoire couvre l'usage direct du client. La sauvegarde logique en profondeur (cohérence avec `--single-transaction`, parallélisme avec `mydumper`/`myloader`), la sauvegarde physique avec `Mariabackup` et la restauration à un instant précis (PITR) sont traitées au [Ch. 12 — Sauvegarde et Restauration](../../12-sauvegarde-restauration/README.md).

---

⬅️ [B.2 — Informations système](02-informations-systeme.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [Annexe C — Requêtes SQL de Référence](../c-requetes-sql-reference/README.md)

⏭️ [Requêtes SQL de Référence](/annexes/c-requetes-sql-reference/README.md)

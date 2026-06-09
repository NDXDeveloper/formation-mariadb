🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.1 — Connexion et navigation

> 🔌 Entrer dans le serveur depuis le terminal, contrôler sa session et explorer bases et tables.

Cette section couvre les premiers gestes d'une session `mariadb` : se connecter, savoir où l'on se trouve, changer de base courante et lister les objets. On y retrouve la distinction posée dans la présentation de l'annexe — commandes client (préfixées d'un antislash, sans point-virgule) et instructions SQL (terminées par `;` ou `\G`).

---

## Se connecter au serveur

La connexion s'établit en appelant `mariadb` avec les options de connexion. Les plus courantes :

| Option | Forme longue | Rôle |
|--------|--------------|------|
| `-u` | `--user=` | Nom d'utilisateur |
| `-p` | `--password[=...]` | Mot de passe (sans valeur : invite interactive) |
| `-h` | `--host=` | Hôte du serveur (défaut : `localhost`) |
| `-P` | `--port=` | Port TCP (défaut : `3306`) |
| `-S` | `--socket=` | Chemin du socket Unix |
| `-D` | `--database=` | Base sélectionnée dès la connexion |
| `-e` | `--execute=` | Exécute une instruction puis quitte |
| — | `--protocol=` | Protocole forcé : `TCP`, `SOCKET`, `PIPE`, `MEMORY` |
| `-A` | `--no-auto-rehash` | Démarrage plus rapide (désactive la complétion des noms) |

Quelques exemples :

```bash
# Connexion locale en tant que root (le mot de passe est demandé)
mariadb -u root -p

# Connexion distante sur une base précise
mariadb -u app_user -p -h db.example.com -P 3306 boutique

# Connexion via un socket Unix explicite (chemin variable selon la distribution)
mariadb -u root -p --socket=/run/mysqld/mysqld.sock

# Exécuter une requête sans entrer en session interactive
mariadb -u root -p -e "SHOW DATABASES;"
```

Sur un système Unix, une connexion à `localhost` passe par défaut par le socket plutôt que par TCP. Le nom de base peut être fourni en argument positionnel (comme `boutique` ci-dessus) ou via `-D`.

> 🔐 Évitez de coller le mot de passe sur la ligne de commande (`-pMonSecret`, sans espace) : il apparaîtrait dans l'historique du shell et dans la liste des processus, et le client affiche d'ailleurs un avertissement. Préférez l'invite interactive (`-p` seul) ou un fichier d'options contenant une section `[client]`. *Voir Ch. 11 (fichiers de configuration) et Ch. 10 (sécurité).*

## Commandes client utiles

Une fois en session, ces raccourcis pilotent le client lui-même. Ils s'utilisent sans point-virgule.

| Commande | Équivalent | Rôle |
|----------|-----------|------|
| `\s` | `status` | Affiche l'état de la session et du serveur |
| `\u <base>` | `use <base>` | Change la base de données courante |
| `\r [base [hôte]]` | `connect` | Se reconnecte, éventuellement à une autre base ou un autre hôte |
| `\c` | — | Annule l'instruction en cours de saisie |
| `\G` | — | Termine l'instruction et affiche le résultat en vertical |
| `\g` | — | Termine l'instruction (équivaut à `;`) |
| `\d <délimiteur>` | `delimiter` | Change le délimiteur d'instruction (utile pour les routines, *voir Ch. 8*) |
| `\h` | `help` | Liste les commandes client |
| `\q` | `quit` / `exit` | Quitte le client |

## Identifier sa session

Avant toute action, il est prudent de savoir qui l'on est et où l'on se trouve.

```sql
STATUS;                  -- ou \s : récapitulatif session + serveur
SELECT VERSION();        -- version du serveur
SELECT DATABASE();       -- base courante (NULL si aucune n'est sélectionnée)
SELECT USER();           -- compte/hôte tel que demandé à la connexion
SELECT CURRENT_USER();   -- compte réellement authentifié par le serveur
```

`USER()` et `CURRENT_USER()` peuvent différer : le premier reflète ce que vous avez demandé, le second la ligne de compte effectivement retenue (par exemple en cas de connexion anonyme ou de correspondance sur un hôte générique).

## Lister et explorer les objets

La navigation proprement dite repose sur quelques instructions `SHOW` et sur `USE`.

```sql
-- Lister les bases
SHOW DATABASES;
SHOW DATABASES LIKE 'app%';     -- filtrage par motif

-- Choisir la base courante
USE boutique;

-- Lister les tables de la base courante
SHOW TABLES;
SHOW FULL TABLES;               -- ajoute le type : BASE TABLE ou VIEW
SHOW TABLES FROM boutique LIKE 'cmd_%';

-- Inspecter une table
DESCRIBE clients;               -- ou DESC clients;
SHOW COLUMNS FROM clients;
SHOW CREATE TABLE clients\G     -- définition complète, affichée en vertical
SHOW CREATE DATABASE boutique;
```

Les instructions `SHOW DATABASES` et `SHOW TABLES` ne révèlent que les objets sur lesquels le compte courant dispose de privilèges. *Voir Ch. 2 pour la création et la structure des objets, et Ch. 10 pour les privilèges.*

## Lire confortablement un résultat large

Lorsqu'une ligne comporte de nombreuses colonnes — typiquement la sortie de `STATUS` ou de `SHOW CREATE TABLE` —, l'affichage tabulaire devient illisible. Terminer l'instruction par `\G` au lieu de `;` présente alors chaque ligne sous forme d'un bloc vertical (une colonne par ligne, sous la forme `colonne: valeur`), nettement plus lisible.

## Quitter la session

Pour sortir du client, utilisez la commande `\q` (ou `quit`, `exit`), ou la combinaison `Ctrl-D`.

---

⬅️ [Annexe B — Commandes mariadb CLI (présentation)](README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [B.2 — Informations système](02-informations-systeme.md)

⏭️ [Informations système (STATUS, SHOW PROCESSLIST, SHOW ENGINE)](/annexes/b-commandes-cli/02-informations-systeme.md)

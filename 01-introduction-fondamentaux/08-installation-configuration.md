🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.8 — Installation et configuration initiale

> 🧭 Cette section couvre l'installation de MariaDB 12.3 LTS sur les principales plateformes et les tout premiers réglages. La configuration approfondie est traitée au **chapitre 11**, la sécurité au **chapitre 10**, et les outils d'administration graphiques à la **section suivante (§1.9)**.

> ⚠️ Les commandes ci-dessous sont **représentatives** : les URL de dépôt et les noms de paquets évoluent. Référez-vous toujours à la page officielle **[mariadb.org/download](https://mariadb.org/download/)** pour la commande exacte correspondant à votre système et à la 12.3.

## Choisir sa méthode d'installation

Plusieurs voies mènent à une installation de MariaDB :

- **Dépôts officiels MariaDB** : la méthode recommandée pour installer précisément la **12.3 LTS**. MariaDB fournit un script qui configure le dépôt pour la version voulue.
- **Dépôts de la distribution** : les plus simples (`apt`, `dnf`…), mais ils embarquent souvent une version **plus ancienne** que la dernière LTS.
- **Installateur graphique** : sous Windows, via un fichier MSI.
- **Homebrew** : sous macOS.
- **Conteneur Docker** : pour le développement et les déploiements cloud-native.

## Sur Debian / Ubuntu

Pour obtenir la 12.3 depuis le dépôt officiel :

```bash
# Configurer le dépôt officiel MariaDB pour la 12.3
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-12.3"

sudo apt update
sudo apt install mariadb-server
```

*(À défaut, `sudo apt install mariadb-server` installe la version fournie par la distribution, qui n'est pas nécessairement la 12.3.)*

## Sur RHEL / Rocky / AlmaLinux / Fedora

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-12.3"

sudo dnf install MariaDB-server MariaDB-client
```

> 💡 Dans les paquets **officiels** au format RPM, les noms commencent par une majuscule (`MariaDB-server`), contrairement aux paquets de la distribution (`mariadb-server`).

## Sur Windows

Téléchargez l'installateur **MSI** depuis [mariadb.org/download](https://mariadb.org/download/) et suivez l'assistant. Celui-ci permet, en quelques écrans, de définir le **mot de passe `root`**, le **port** (3306 par défaut), l'encodage **UTF-8**, et d'enregistrer MariaDB comme **service Windows** démarrant automatiquement.

## Sur macOS

Avec **Homebrew** :

```bash
brew install mariadb
brew services start mariadb   # démarre le service et l'active au démarrage
```

## Via Docker

Pour lancer rapidement une instance 12.3 :

```bash
docker run --name mariadb \
  -e MARIADB_ROOT_PASSWORD=motdepasse_solide \
  -p 3306:3306 \
  -d mariadb:12.3
```

La variable `MARIADB_ROOT_PASSWORD` définit le mot de passe `root` à la création. Le conteneur, la persistance des données (volumes) et l'orchestration sont approfondis au **chapitre 16**.

## Note 12.3 : le packaging de Galera

Un changement de packaging propre à la **12.3** mérite d'être signalé dès l'installation : la prise en charge du clustering **Galera** n'est plus intégrée au paquet du serveur. Elle est désormais fournie par un **paquet séparé**, `mariadb-server-galera`, qu'il faut installer explicitement pour monter un cluster. Ce changement a aussi un impact sur les **images Docker officielles**. Le détail est traité en §14.2.5 (et §16.3.1 pour les images).

## Après l'installation : démarrer et sécuriser

Sous Linux, activez et démarrez le service, puis lancez l'assistant de sécurisation :

```bash
sudo systemctl enable --now mariadb     # démarre maintenant + au prochain boot
systemctl status mariadb                # doit afficher « active (running) »
sudo mariadb-secure-installation
```

L'outil **`mariadb-secure-installation`** guide à travers plusieurs étapes de durcissement : définir (ou confirmer) le mot de passe `root`, supprimer les comptes anonymes, désactiver la connexion `root` à distance et retirer la base de test. C'est une étape **fortement recommandée** sur tout serveur destiné à être utilisé.

> 💡 Sur une installation Linux fraîche, le compte `root` de MariaDB s'authentifie par défaut via le *plugin* `unix_socket` : on s'y connecte donc avec `sudo mariadb`, sans mot de passe applicatif. La sécurité et l'authentification sont détaillées au chapitre 10.

## Vérifier l'installation

Quelques contrôles simples confirment que tout fonctionne :

```bash
mariadb --version          # version du client installé
```

Puis, en se connectant au serveur :

```bash
sudo mariadb
```

```sql
SELECT VERSION();
```

La requête doit renvoyer un numéro de version commençant par **`12.3`**. (La prise en main du client en ligne de commande et des outils graphiques comme HeidiSQL, DBeaver ou phpMyAdmin fait l'objet de la **section §1.9**.)

Dans la foulée, un premier coup d'œil aux **moteurs de stockage** disponibles (notion introduite en §1.4 et approfondie au chapitre 7) :

```sql
SHOW ENGINES;
```

La colonne `Support` y signale le moteur par défaut — `DEFAULT` en regard d'**InnoDB** — et les autres moteurs activés (`YES`).

## Premiers réglages de configuration

MariaDB lit ses paramètres dans des fichiers de configuration (`my.cnf` / `my.ini`), dont l'emplacement varie selon le système (par exemple `/etc/mysql/mariadb.conf.d/` sous Debian/Ubuntu, `/etc/my.cnf.d/` sous RHEL). Pour un premier serveur, quelques réglages suffisent généralement :

- **`bind-address`** : interface réseau d'écoute (l'accès distant est souvent restreint par défaut, pour des raisons de sécurité) ;
- **`port`** : 3306 par défaut ;
- **`datadir`** : répertoire des fichiers de données ;
- **`innodb_buffer_pool_size`** : mémoire allouée au cache d'InnoDB, paramètre clé de performance.

Bonne nouvelle : depuis la 11.8, le jeu de caractères par défaut est **`utf8mb4`** (§11.11), ce qui évite d'avoir à configurer l'encodage pour gérer correctement l'Unicode (émojis compris). L'ensemble des fichiers, sections et variables est détaillé au **chapitre 11**.

## À retenir

Pour installer la **12.3 LTS**, la voie recommandée est le **dépôt officiel MariaDB** (les dépôts de distribution étant souvent en retard) ; Windows passe par un **MSI**, macOS par **Homebrew**, et le développement par **Docker** (`mariadb:12.3`). Après l'installation sous Linux, on **démarre le service** (`systemctl enable --now mariadb`) puis on exécute **`mariadb-secure-installation`**. On vérifie avec `SELECT VERSION();`. Point d'attention propre à la 12.3 : **Galera** est désormais dans un **paquet séparé** (`mariadb-server-galera`). Les premiers réglages se font dans `my.cnf`, sachant que **`utf8mb4`** est déjà l'encodage par défaut.

---

**Navigation** : [⬆️ Chapitre 1 — Introduction et Fondamentaux](README.md) · Section précédente : [1.7 Roadmap : 12.3 LTS → série 13.x](07-roadmap-serie-13.md) · Section suivante → [1.9 Outils d'administration](09-outils-administration.md)

⏭️ [Outils d'administration (CLI, HeidiSQL, DBeaver, phpMyAdmin)](/01-introduction-fondamentaux/09-outils-administration.md)

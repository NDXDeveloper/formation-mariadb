🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.9 — Outils d'administration (CLI, HeidiSQL, DBeaver, phpMyAdmin)

> 🧭 Dernière section du chapitre. Elle présente les principaux outils pour se connecter à MariaDB et l'administrer. Les commandes du client en ligne de commande sont détaillées en [annexe B](../annexes/b-commandes-cli/README.md), et les outils de sauvegarde au chapitre 12.

## Un point commun : les paramètres de connexion

Quel que soit l'outil choisi, se connecter à un serveur MariaDB demande **toujours les mêmes informations** :

- l'**hôte** (adresse du serveur ; `localhost` en local) ;
- le **port** (3306 par défaut) ;
- un **nom d'utilisateur** et son **mot de passe** ;
- éventuellement une **base de données** par défaut.

Les outils ne se distinguent donc pas par *ce* à quoi ils se connectent, mais par *la manière* dont on interagit ensuite avec la base : en ligne de commande, via une interface graphique de bureau, ou depuis un navigateur.

## Le client en ligne de commande (CLI)

Le **client `mariadb`** (anciennement `mysql`) est l'outil **universel** : il est installé avec MariaDB et présent sur tout serveur. On s'y connecte ainsi :

```bash
mariadb -h <hôte> -P 3306 -u <utilisateur> -p <base>
```

L'option `-p` (sans valeur) invite à saisir le mot de passe de façon masquée. En local, `sudo mariadb` suffit souvent (voir §1.8).

Le client fait partie d'une **famille d'outils en ligne de commande** : `mariadb-admin` (tâches d'administration), `mariadb-dump` (sauvegardes), `mariadb-import`, `mariadb-show`, etc. Leur usage est détaillé en [annexe B](../annexes/b-commandes-cli/README.md) et au chapitre 12.

Ses atouts : il est **léger**, fonctionne parfaitement à distance via **SSH**, et surtout il est **scriptable** — donc indispensable pour l'**automatisation** et l'administration de serveurs sans interface graphique. Sa contrepartie : tout passe par le texte, ce qui demande un peu plus d'habitude qu'une interface visuelle.

## HeidiSQL

**HeidiSQL** est une interface graphique **gratuite et open source**, **native sous Windows** (elle fonctionne aussi sous Linux/macOS via Wine). Légère et rapide, elle est très populaire dans l'univers MySQL/MariaDB.

Elle permet de parcourir et modifier les bases et tables, d'exécuter des requêtes, de gérer les utilisateurs et d'exporter des données, le tout dans une interface compacte. C'est un excellent choix pour un **poste de travail Windows** et pour une **prise en main rapide**.

## DBeaver

**DBeaver** est une interface graphique **multiplateforme** (Windows, macOS, Linux), écrite en Java. Son édition **Community** est gratuite et open source.

Sa particularité est d'être un outil **universel** : il se connecte à de très nombreux SGBD (MariaDB, MySQL, PostgreSQL, Oracle, SQL Server…) via une interface unique, et propose des fonctionnalités riches (éditeur de données, diagrammes entité-association, éditeur SQL avancé). Il s'impose lorsqu'on travaille avec **plusieurs systèmes de bases de données** ou que l'on a besoin d'un outil **identique sur tous les OS**.

## phpMyAdmin

**phpMyAdmin** est une application **web** gratuite et open source, écrite en PHP. Elle ne s'installe pas sur le poste de travail mais sur un **serveur web** : on l'utilise ensuite depuis un simple **navigateur**.

C'est l'outil emblématique des environnements **LAMP** et des **hébergements mutualisés**, où il est souvent préinstallé. Il couvre l'essentiel de l'administration courante (bases, tables, utilisateurs, import/export, exécution de requêtes SQL) sans rien installer côté client — pratique pour gérer une base **à distance via le web**.

## Comparatif

| Outil | Type | Plateforme | Idéal pour |
|-------|------|-----------|------------|
| **Client `mariadb`** | Ligne de commande | Linux, Windows, macOS | Automatisation, scripts, administration distante (SSH) |
| **HeidiSQL** | Interface graphique | Windows (Linux/macOS via Wine) | Poste Windows, prise en main rapide |
| **DBeaver** | Interface graphique | Windows, macOS, Linux | Travail multi-SGBD, fonctionnalités visuelles |
| **phpMyAdmin** | Application web | Navigateur (via serveur web) | Hébergement mutualisé, gestion via navigateur |

## Et aussi…

D'autres outils gravitent autour de MariaDB : **Adminer** (alternative web ultra-légère à phpMyAdmin, en un seul fichier), **MySQL Workbench** (orienté MySQL, compatibilité limitée avec MariaDB), ou divers clients commerciaux. Le **client `mariadb`** reste toutefois le **dénominateur commun**, présent partout.

## Comment choisir ?

Il n'y a pas de « meilleur » outil dans l'absolu — le bon choix dépend du contexte, et l'on en combine souvent plusieurs :

- pour **scripter, automatiser ou administrer un serveur distant** : le **client en ligne de commande** ;
- pour un **travail interactif confortable** sur un poste de bureau : **HeidiSQL** (Windows) ou **DBeaver** (multiplateforme, multi-SGBD) ;
- pour une **gestion via navigateur**, typiquement en hébergement web : **phpMyAdmin**.

Dans cette formation, les exemples s'appuient principalement sur le **client en ligne de commande**, car il est universel et indépendant de la plateforme.

## À retenir

Tous les outils d'administration se connectent à MariaDB avec les **mêmes paramètres** (hôte, port, utilisateur, mot de passe) ; ils diffèrent par le **mode d'interaction**. Le **client `mariadb`** (CLI) est universel, léger et scriptable — idéal pour l'automatisation et l'administration distante. Côté interfaces graphiques, **HeidiSQL** brille sur Windows par sa simplicité, tandis que **DBeaver** est multiplateforme et multi-SGBD. **phpMyAdmin**, enfin, offre une administration **web** prisée des hébergements mutualisés. Le choix dépend du contexte, et l'on utilise souvent **plusieurs outils** selon les besoins.

---

**Navigation** : [⬆️ Chapitre 1 — Introduction et Fondamentaux](README.md) · Section précédente : [1.8 Installation et configuration initiale](08-installation-configuration.md) · Chapitre suivant → [Chapitre 2 — Bases du SQL](../02-bases-du-sql/README.md)

⏭️ [Bases du SQL](/02-bases-du-sql/README.md)

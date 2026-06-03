🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 1 — Introduction et Fondamentaux

**Poser les bases avant d'écrire la première requête SQL**

> **Niveau** : Débutant · **Durée estimée** : ~1 jour · **Prérequis** : aucun · **Version de référence** : MariaDB 12.3 LTS

---

Bienvenue dans le premier chapitre de cette formation **MariaDB 12.3 LTS**. Avant d'aborder le langage SQL et les fonctionnalités avancées, il est essentiel de poser des fondations solides : comprendre *ce qu'est* MariaDB, *d'où* il vient, *où* il s'utilise et *comment* il s'organise. Ce chapitre fondateur installe le vocabulaire, le contexte historique et les concepts d'architecture qui serviront de socle à l'ensemble du parcours.

C'est également ici que vous mettrez en place votre environnement de travail : installation du serveur, configuration initiale et choix d'un outil d'administration. À l'issue du chapitre, vous disposerez d'une instance MariaDB fonctionnelle et d'une vision claire de l'écosystème dans lequel vous allez évoluer.

## Objectifs pédagogiques

À la fin de ce chapitre, vous serez en mesure de :

- définir MariaDB et le situer parmi les systèmes de gestion de bases de données relationnelles (SGBDR) ;
- expliquer l'origine du projet et les principales divergences avec MySQL ;
- identifier les cas d'usage typiques et les composants clés de l'écosystème ;
- décrire l'architecture générale d'un SGBDR (modèle client-serveur, moteurs de stockage) ;
- distinguer les versions **LTS** des versions **rolling** et comprendre leur cycle de support ;
- situer la version **12.3 LTS** dans la roadmap du projet ;
- installer et configurer une première instance MariaDB ;
- choisir et prendre en main un outil d'administration adapté à votre contexte.

## À qui s'adresse ce chapitre

Ce chapitre constitue le point de départ commun à **tous les parcours** de la formation (Développeur, DBA, DevOps/Cloud, IA/ML). Quel que soit votre profil, il est recommandé de le parcourir avant d'attaquer les chapitres spécialisés : il fixe les repères partagés par le reste du contenu.

## Prérequis

Ce premier chapitre ne suppose aucune connaissance préalable de MariaDB. Une familiarité de base avec la ligne de commande sous Unix/Linux et quelques notions générales de bases de données faciliteront la lecture, sans être indispensables. Les concepts SQL proprement dits seront introduits progressivement à partir du chapitre 2.

## Contenu du chapitre

Le chapitre est organisé en neuf sections, de la définition générale jusqu'à la mise en place concrète de l'environnement :

| Section | Thème | En bref |
|---------|-------|---------|
| [1.1](01-quest-ce-que-mariadb.md) | Qu'est-ce que MariaDB ? | Définition, nature open source et positionnement comme SGBDR. |
| [1.2](02-histoire-et-differences-mysql.md) | Histoire et différences avec MySQL | Naissance du fork, gouvernance et divergences techniques. |
| [1.3](03-cas-usage-et-ecosysteme.md) | Cas d'usage et écosystème | Domaines d'application et composants gravitant autour du serveur. |
| [1.4](04-architecture-generale-sgbd.md) | Architecture générale d'un SGBD relationnel | Modèle relationnel, architecture client-serveur, moteurs de stockage. |
| [1.5](05-politique-versions-lts-rolling.md) | Politique de versions : LTS vs Rolling | Deux cadences de publication et leurs usages respectifs. |
| [1.6](06-cycle-support-lts.md) | Cycle de support | Durée de maintenance des LTS (3 ans) et des versions rolling. |
| [1.7](07-roadmap-serie-13.md) | Roadmap | De la 12.3 LTS vers la série 13.x. |
| [1.8](08-installation-configuration.md) | Installation et configuration initiale | Mise en place du serveur et premiers réglages. |
| [1.9](09-outils-administration.md) | Outils d'administration | CLI, HeidiSQL, DBeaver, phpMyAdmin. |

## À propos de la version

Cette formation prend pour référence **MariaDB 12.3 LTS** (*Long Term Support*), dont la version GA (12.3.2, la 12.3.1 ayant été une *release candidate*) est sortie fin mai 2026 et sera maintenue jusqu'en **juin 2029**. La 12.3 consolide les nouveautés introduites par la série rolling 12.0 → 12.2.

La précédente LTS, la **11.8** (juin 2025, supportée jusqu'en 2028), reste très largement déployée : elle sert de point de comparaison tout au long du parcours, en particulier pour les questions de migration. Les fonctionnalités spécifiques à la série 12.x sont signalées par le marqueur 🆕 dans le reste de la formation.

> 💡 Si vous découvrez MariaDB, retenez simplement pour l'instant qu'une version **LTS** est conçue pour un déploiement durable en production, tandis que les versions **rolling** permettent de suivre les nouveautés au fil de l'eau. La section 1.5 détaille cette distinction.

---

**Navigation** : [📚 Sommaire](../SOMMAIRE.md) · Section suivante → [1.1 Qu'est-ce que MariaDB ?](01-quest-ce-que-mariadb.md)

⏭️ [Qu'est-ce que MariaDB ?](/01-introduction-fondamentaux/01-quest-ce-que-mariadb.md)

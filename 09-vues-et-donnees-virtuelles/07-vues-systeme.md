🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.7 · Vues système

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introspection : interroger le serveur en SQL

Tout au long de ce chapitre, nous avons inspecté nos propres vues en interrogeant `INFORMATION_SCHEMA.VIEWS`. Ce n'était pas un hasard : MariaDB **expose sa propre structure et son propre état** sous la forme d'objets que l'on consulte avec de simples requêtes `SELECT`. Plutôt que de mémoriser une multitude de commandes propriétaires, l'administrateur ou le développeur peut **introspecter le serveur** avec le langage qu'il connaît déjà.

C'est l'aboutissement de la notion de **donnée virtuelle** qui donne son titre au chapitre : ces objets présentent des informations — la liste des tables, l'état des verrous, les comptes utilisateurs — comme s'il s'agissait de tables ordinaires, alors qu'elles ne stockent (pour la plupart) aucune donnée applicative et sont souvent calculées à la volée.

## Trois espaces système

MariaDB regroupe ces informations dans trois espaces distincts, qui répondent à trois besoins différents :

- **`INFORMATION_SCHEMA`** décrit la **structure** de l'instance : bases, tables, colonnes, vues, index, contraintes, privilèges, jeux de caractères, moteurs… C'est l'outil d'introspection des *métadonnées*.
- **`PERFORMANCE_SCHEMA`** instrumente l'**exécution** du serveur en temps réel : attentes, requêtes, étapes d'exécution, verrous, consommation mémoire… C'est l'outil de *supervision* fine.
- Les **tables système `mysql`** contiennent les **données opérationnelles** du serveur lui-même : comptes et privilèges, fuseaux horaires, tables d'aide, métadonnées des routines et des événements… C'est la *source de vérité* de la configuration.

Le tableau suivant résume leurs différences fondamentales :

| Espace | Rôle | Nature | Standard SQL ? | Modifiable ? |
|--------|------|--------|----------------|--------------|
| `INFORMATION_SCHEMA` | Métadonnées de **structure** | Virtuel, généré à la demande | **Oui** (ANSI/ISO) | Non — lecture seule |
| `PERFORMANCE_SCHEMA` | Métriques d'**exécution** | Tables d'instrumentation en mémoire | Non — spécifique MariaDB/MySQL | Non — lecture seule en pratique |
| Base `mysql` | **Données opérationnelles** (privilèges, config…) | Tables réelles **persistantes** | Non | Oui, mais via les commandes dédiées (`GRANT`, `CREATE USER`…), pas en SQL direct |

> **À noter — le schéma `sys`.** Un quatrième espace, le schéma **`sys`**, complète ce paysage : il rassemble un ensemble de **vues** (et de procédures) bâties par-dessus `PERFORMANCE_SCHEMA` et `INFORMATION_SCHEMA` pour en présenter les données sous une forme directement lisible. Il est étudié avec `PERFORMANCE_SCHEMA` au **§15.8**.

## Sont-ce vraiment des « vues » ?

Le terme « vues système » est employé au sens large. À la lumière de tout ce que le chapitre a établi, il est utile de préciser la nature exacte de chacun de ces objets :

- les tables d'`INFORMATION_SCHEMA` sont **virtuelles** : elles n'existent pas physiquement et sont produites à la demande à partir du dictionnaire de données. Elles se comportent comme des vues en lecture seule — c'en est l'illustration la plus pure ;
- les tables de `PERFORMANCE_SCHEMA` sont, au sens strict, de véritables **tables** (servies par un moteur de stockage dédié, en mémoire) que l'instrumentation alimente ; ce ne sont pas des vues, mais on les interroge de la même manière ;
- la base `mysql` contient des **tables réelles et persistantes** — et, fait notable pour ce chapitre, certaines y sont en réalité des **vues de compatibilité** (point abordé au §9.7.3) ;
- le schéma `sys`, enfin, est constitué de **vraies vues SQL**.

Ce qui les réunit n'est donc pas leur nature technique, mais leur **fonction commune** : offrir, via `SELECT`, une fenêtre virtuelle sur le serveur.

## Caractéristiques communes

Quelle que soit leur implémentation, ces espaces partagent plusieurs propriétés :

- ils s'interrogent avec des **requêtes `SELECT` standard** — jointures, filtres et agrégats compris —, ce qui en fait un terrain d'application direct de tout ce qui précède dans la formation ;
- ils sont **filtrés par les privilèges** : un utilisateur n'y voit que les objets et informations auxquels ses droits lui donnent accès ;
- ils sont, à l'usage, **essentiellement en lecture** (les tables `mysql` faisant exception, mais se gérant par les commandes dédiées) ;
- ils sont la **fondation de l'outillage** : clients graphiques (HeidiSQL, DBeaver, phpMyAdmin), supervision (Prometheus, Grafana — §16.9) et scripts d'automatisation s'appuient tous sur ces espaces pour découvrir et surveiller la base.

## Plan de la section

| Sous-section | Espace | En bref |
|--------------|--------|---------|
| **9.7.1** | `INFORMATION_SCHEMA` | Les métadonnées de structure, portables et standard |
| **9.7.2** | `PERFORMANCE_SCHEMA` | L'instrumentation et la supervision de l'exécution |
| **9.7.3** | Tables système `mysql` | Les tables opérationnelles du serveur (privilèges, configuration) |

## En résumé

MariaDB se laisse **interroger en SQL** : il expose sa structure, son état d'exécution et sa configuration à travers des espaces système consultables par de simples `SELECT`. `INFORMATION_SCHEMA` (virtuel et standard) décrit la structure, `PERFORMANCE_SCHEMA` (instrumentation en mémoire) mesure l'exécution, et la base `mysql` (tables réelles) conserve les données opérationnelles — le schéma `sys` apportant par-dessus une couche de lisibilité. Tous se distinguent par leur nature mais convergent vers un même rôle d'**introspection**, filtré par les privilèges et fondement de l'outillage d'administration.

Commençons par le plus universel d'entre eux, celui que nous avons déjà mis à contribution dans ce chapitre : le **§9.7.1 — `INFORMATION_SCHEMA`**.

⏭️ [INFORMATION_SCHEMA](/09-vues-et-donnees-virtuelles/07.1-information-schema.md)

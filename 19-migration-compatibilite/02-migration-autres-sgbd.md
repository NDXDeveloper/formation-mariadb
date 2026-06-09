🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.2 Migration depuis d'autres SGBD

> **Chapitre 19 — Migration et Compatibilité** · Section 19.2  
> Niveau : DBA / Architecte

---

## D'une migration homogène à une migration hétérogène

La section précédente traitait la migration depuis MySQL, un cas **homogène** : grâce à l'origine commune des deux moteurs, l'essentiel du travail se résumait à un transfert de données et à la vérification d'un nombre limité d'écarts. Migrer depuis **Oracle, SQL Server ou PostgreSQL** relève d'une tout autre nature. Il s'agit d'une migration **hétérogène**, où le système source et MariaDB ne partagent ni le même dialecte SQL, ni le même langage procédural, ni les mêmes types de données ou comportements transactionnels.

La conséquence est qu'une migration hétérogène est un véritable **projet de portage**, et non un simple déplacement de données. L'effort se concentre rarement sur le transfert lui-même — relativement mécanique — mais sur la **conversion du schéma et du code procédural**, qui constitue le cœur de la difficulté.

---

## Ce qui rend une migration hétérogène difficile

Plusieurs sources de divergence se cumulent :

- **Le dialecte SQL.** Chaque système a ses particularités syntaxiques : Oracle emploie des identifiants entre guillemets doubles et des types comme `VARCHAR2` ou `NUMBER`, que MariaDB ne reconnaît pas par défaut. SQL Server et PostgreSQL ont leurs propres conventions.
- **Le langage procédural.** Oracle utilise **PL/SQL**, SQL Server **T-SQL**, PostgreSQL **PL/pgSQL**, tandis que MariaDB repose nativement sur **SQL/PSM** (le langage procédural du standard SQL). Procédures stockées, fonctions, déclencheurs et — côté Oracle — paquets (*packages*) doivent être convertis. C'est le poste de travail le plus lourd.
- **Les types de données.** Chaque type source doit être transposé vers un équivalent MariaDB (par exemple `NUMBER` vers `DECIMAL` ou un type entier, `VARCHAR2` vers `VARCHAR`).
- **Les comportements et la sémantique.** Gestion des transactions et de l'isolation, traitement des valeurs `NULL`, sensibilité à la casse des identifiants, séquences, fonctions intégrées : autant de différences qui peuvent modifier le comportement d'une application.

On distingue donc en pratique **deux chantiers** : la conversion du schéma et du code (la partie difficile, spécifique à chaque SGBD), et le transfert des données (plus mécanique, mais à fiabiliser).

---

## Les atouts de compatibilité de MariaDB

MariaDB a investi dans des fonctionnalités destinées à réduire l'effort de migration hétérogène.

### Le mode Oracle : `SQL_MODE=ORACLE`

C'est l'atout phare. Depuis **MariaDB 10.3**, le réglage `SQL_MODE=ORACLE` active une **compatibilité étendue avec le dialecte SQL et le PL/SQL d'Oracle** : syntaxe Oracle, types de données, séquences et constructions procédurales sont reconnus dans une large mesure, ce qui réduit nettement la quantité de code à réécrire manuellement.

Ce mode est **optionnel** et s'active simplement, pour la session courante, globalement, ou de façon permanente dans un fichier de configuration :

```sql
SET SESSION sql_mode = 'ORACLE';
```

Deux nuances importantes : d'une part, le mode Oracle ne couvre pas **toutes** les fonctionnalités d'Oracle — certaines constructions doivent encore être converties vers le SQL natif de MariaDB ; d'autre part, MariaDB **enrichit régulièrement** cette compatibilité au fil des versions. La série **12.x** y a apporté de nouveaux éléments (fonctions de conversion, syntaxes de jointure et constructions procédurales supplémentaires), détaillés dans la sous-section 19.2.1.

### Des modes pour SQL Server et PostgreSQL

MariaDB propose également des valeurs de `sql_mode` pour **SQL Server** et **PostgreSQL** (ainsi que pour d'anciennes versions de MySQL). Il faut toutefois en mesurer la portée : ces modes sont **nettement plus limités** que le mode Oracle. Ils agissent surtout sur la syntaxe et le traitement des identifiants, et n'offrent pas une émulation de dialecte aussi poussée. La migration depuis SQL Server ou PostgreSQL repose donc davantage sur l'outillage de conversion et la réécriture manuelle.

À noter : le PL/pgSQL de PostgreSQL étant proche du PL/SQL d'Oracle, il peut être avantageux d'essayer une migration PostgreSQL avec `SQL_MODE=ORACLE` activé.

### Le moteur CONNECT pour le transfert de données

Pour le transfert des données proprement dit, le **moteur de stockage CONNECT** (voir §7.10.4) offre une voie élégante. Il permet de créer dans MariaDB des tables qui pointent vers des tables **externes** via **ODBC ou JDBC**, puis de rapatrier les données directement par une instruction `INSERT ... SELECT` vers une table native InnoDB. On peut ainsi lire une source Oracle, SQL Server ou PostgreSQL et alimenter MariaDB sans passer par des fichiers intermédiaires.

---

## Outils et ressources

Au-delà des fonctionnalités intégrées, plusieurs ressources facilitent la migration :

- **La documentation officielle de MariaDB** propose des guides dédiés à la migration depuis Oracle, SQL Server et PostgreSQL : ce sont les points de départ recommandés.
- **Les outils de conversion** automatisent une partie du travail de schéma et de code : convertisseurs de DDL, de vues, de procédures et de scripts (par exemple la famille d'outils SQLines), ou encore l'assistant de migration de MySQL Workbench utilisé avec des pilotes ODBC.

Ces outils accélèrent la conversion mais ne la garantissent pas à 100 % : le mode Oracle comme les convertisseurs ne couvrent pas l'intégralité des fonctionnalités sources. Une **relecture et une validation** restent indispensables.

---

## Pourquoi migrer vers MariaDB

Les motivations varient selon le système d'origine, mais certaines reviennent fréquemment — particulièrement dans le cas d'Oracle, où la **réduction des coûts et la simplification du modèle de licence** constituent souvent le premier moteur de décision. S'y ajoutent la **flexibilité de déploiement** (du matériel dédié au cloud, en conteneurs ou sur Kubernetes), la **gouvernance ouverte** et les performances. Ces éléments sont présentés ici à titre de contexte ; le choix relève toujours d'une analyse propre à chaque organisation.

---

## Démarche générale

Une migration hétérogène suit une trame structurée, précisée pour chaque SGBD dans les sous-sections suivantes :

1. **Audit** : inventaire du schéma, du code procédural, des types utilisés et des fonctionnalités propres au système source.
2. **Conversion du schéma** : transposition des définitions d'objets et des types, en s'appuyant sur le `sql_mode` adapté et les outils de conversion.
3. **Conversion du code procédural** : portage des procédures, fonctions et déclencheurs (PL/SQL, T-SQL ou PL/pgSQL) vers le mode Oracle ou le SQL/PSM natif — le poste le plus exigeant.
4. **Transfert des données** : via le moteur CONNECT, un outil d'ETL ou un export/import.
5. **Adaptation applicative et tests** : validation fonctionnelle et de non-régression (voir 19.6).
6. **Bascule et plan de repli** : exécution de la migration, avec retour arrière (voir 19.7) et, le cas échéant, sans interruption (voir 19.8).

---

## Organisation de la section

Les sous-sections suivantes traitent chaque système d'origine selon son niveau de compatibilité et ses spécificités :

- **19.2.1 — Depuis Oracle** : le cas le mieux pris en charge, grâce à `SQL_MODE=ORACLE` et aux apports de la série 12.x (fonctions `TO_DATE`/`TO_NUMBER`/`TRUNC`, syntaxe de jointure externe `( + )`, tableaux associatifs, `SYS_REFCURSOR`, `SET PATH`).
- **19.2.2 — Depuis SQL Server** : conversion du T-SQL, mode dédié plus limité, et donc une part de réécriture manuelle plus importante.
- **19.2.3 — Depuis PostgreSQL** : portage du PL/pgSQL, recours au moteur CONNECT, et possibilité d'exploiter le mode Oracle en raison de la proximité des langages procéduraux.

⏭️ [Depuis Oracle (TO_DATE/TO_NUMBER/TRUNC, ( + ), tableaux associatifs, SYS_REFCURSOR, SET PATH)](/19-migration-compatibilite/02.1-depuis-oracle.md)
